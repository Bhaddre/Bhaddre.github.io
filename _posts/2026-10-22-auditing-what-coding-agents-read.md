---
layout: post
title: "Which files did the agent read? You probably can't say"
date: 2026-10-22
tags: [ai-agents, claude-code, opentelemetry, security, platform-engineering]
---

Roll out Claude Code or Cursor across an engineering org and sooner or later someone from security asks a very reasonable question. On this laptop, last week, which files did the agent open? Did it touch the `.env`? Did it read the terraform state somebody left lying around in a repo?

Out of the box you can't answer that. You've got usage dashboards. Token counts, sessions, cost. None of it says which files.

I did the rollout for both tools at my day job, with org-wide privacy and data retention settings plus OpenTelemetry auditing of what the agents read or modify. The retention part is mostly contracts and admin toggles. The auditing part is the one people underestimate, so that's this post.

## deny rules aren't a sandbox

First thing everyone does is write deny rules. Claude Code reads managed settings from a system path that users can't override (`/etc/claude-code/managed-settings.json` on Linux, `/Library/Application Support/ClaudeCode/` on macOS, check the docs for your version):

```json
{
  "permissions": {
    "deny": [
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Read(**/*.tfstate)",
      "Read(**/secrets/**)"
    ]
  }
}
```

Do this. It's cheap and it stops the obvious stuff. But know what it is. It's a rule on the `Read` tool. The agent also has a shell. `cat .env` is a Bash call, not a Read call, and a deny on `Read(**/.env)` doesn't cover it. You can add Bash deny patterns too, and you should for the obvious ones, but shell commands have infinite spellings. `head`, `less`, `python -c`, `grep -r` across the whole repo. You won't win that by pattern matching.

So deny rules lower the odds. They don't give you an answer to "what happened." For that you need a record.

## hooks are where the file paths are

Claude Code has built-in OpenTelemetry export for metrics and events, and you should turn it on. But last I checked the default events are about the tool call (which tool, success, duration), not the arguments. File paths aren't in there unless you opt into more detail, and the knobs for that have changed between versions.

Hooks are the reliable spot. A `PostToolUse` hook gets the full tool input as JSON on stdin, including `file_path` for Read/Edit/Write and `command` for Bash. Ship that to your collector and you have the audit trail.

Put the hook in the same managed settings file so nobody can turn it off:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Read|Edit|MultiEdit|Write|Grep|Glob|Bash",
        "hooks": [
          { "type": "command", "command": "/opt/agent-audit/audit_hook.py", "timeout": 5 }
        ]
      }
    ]
  }
}
```

And the hook. Stdlib only, on purpose:

```python
#!/usr/bin/env python3
# PostToolUse hook: ship which file/command the agent touched to OTLP. never block.
import getpass
import json
import os
import socket
import sys
import time
import urllib.request

ENDPOINT = os.environ.get("AGENT_AUDIT_OTLP", "https://otel.example.com/v1/logs")


def attr(k, v):
    return {"key": k, "value": {"stringValue": str(v)}}


def main():
    ev = json.load(sys.stdin)
    tool = ev.get("tool_name", "")
    inp = ev.get("tool_input", {}) or {}
    # paths and commands only. never file contents, never tool output.
    target = inp.get("file_path") or inp.get("path") or inp.get("command") or ""

    record = {
        "timeUnixNano": str(time.time_ns()),
        "severityText": "INFO",
        "body": {"stringValue": f"{tool} {target}"[:2000]},
        "attributes": [
            attr("agent.tool", tool),
            attr("agent.target", target[:2000]),
            attr("agent.session_id", ev.get("session_id", "")),
            attr("agent.cwd", ev.get("cwd", "")),
            attr("user.name", getpass.getuser()),
            attr("host.name", socket.gethostname()),
        ],
    }
    body = {"resourceLogs": [{
        "resource": {"attributes": [attr("service.name", "agent-audit")]},
        "scopeLogs": [{"scope": {"name": "agent-audit"}, "logRecords": [record]}],
    }]}

    req = urllib.request.Request(
        ENDPOINT,
        data=json.dumps(body).encode(),
        headers={"Content-Type": "application/json"},
    )
    urllib.request.urlopen(req, timeout=1)


if __name__ == "__main__":
    try:
        main()
    except Exception:
        pass  # audit is best effort. a broken collector must not break someone's editor.
    sys.exit(0)
```

That's OTLP/HTTP with a JSON body straight to `/v1/logs`. No SDK. From there it's whatever your collector already does with logs.

## stuff I'd tell you before you ship it

**Stdlib only.** The hook runs on every matching tool call. Importing the OTel SDK costs real time on every call. Engineers will feel a laggy agent long before they file a ticket about it, and then they'll find a way to turn your stuff off.

**Always exit 0.** For hooks, exit code 2 has meaning (it blocks or feeds stderr back to the model depending on the event). You don't want a timeout on your collector showing up as an agent behavior change. Fail open on audit, fail closed on the deny rules. Different jobs.

**Paths, not contents.** The whole point of the privacy side of the rollout is that code doesn't leave where it shouldn't. An audit pipeline that copies file contents into your log store just built a second copy of the problem. Path, command, user, host, session. That's enough to answer the question.

**Log Bash commands, accept the noise.** Since Bash is the hole in the deny rules, it's the most important thing to record. Yes it's noisy. Yes some commands are long. Truncate and move on.

**Grep and Glob matter.** People forget these. A recursive grep reads a lot of files without a single Read call. Record the path and pattern.

**Cursor is a different shape.** Cursor added its own hooks more recently with different event names and payloads. Same idea, separate adapter. Don't try to force one script to parse both, write two small ones that emit the same attributes so the dashboard doesn't care which tool it was.

## what you get

One query. Which sessions touched anything matching `.env` or `tfstate` in the last 7 days, grouped by user. That's the question security asks, and now it's a saved search instead of a shrug.

It also changes the rollout conversation. "We have deny rules" gets you a nod. "Here's every file an agent opened in that repo this month" gets you approval for the next team.

If you can't say what an agent read, you didn't roll out an agent, you rolled out an unaudited contractor with shell access.
