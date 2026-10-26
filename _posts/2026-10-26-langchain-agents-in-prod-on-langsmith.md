---
layout: post
title: "The LangChain agent worked on my laptop. Slack had other plans."
date: 2026-10-26
tags: [langchain, langgraph, langsmith, ai-agents, slack]
---

Manager asks the bot a question in Slack. Gets the answer. Thirty seconds later gets the same answer again. Then a third time. Each one burned a full agent run, tool calls and all.

Nothing was wrong with the agent. Slack retries any event you don't ack within 3 seconds, and an agent run takes way longer than 3 seconds. So Slack decided we were dead and sent it again. Twice.

This is the Slack agent at my day job. It's a LangChain agent that pulls status from the systems where the work actually lives and answers questions about it. It replaced a weekly status deck that around 60 managers used to read (or skim, or not open). Making it behave in prod took way more work than building it, and almost none of that went into the prompt.

Here's what I'd tell someone starting one now.

## the agent itself is the easy part

With LangChain 1.0 the core is small. `create_agent`, a model, some tools, a system prompt. The interesting bits are the middleware.

```python
# agent.py
import os

from langchain.agents import create_agent
from langchain.agents.middleware import ModelCallLimitMiddleware, ToolCallLimitMiddleware
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

# talk to the gateway, not the provider. no raw API keys in the agent.
model = ChatOpenAI(
    model="claude-sonnet",
    base_url=os.environ["LLM_GATEWAY_URL"],
    api_key=os.environ["LLM_GATEWAY_KEY"],
    timeout=60,
)

MAX_CHARS = 4000


@tool
def project_status(project: str) -> str:
    """Current status, owner and open blockers for one project."""
    try:
        data = fetch_status(project)  # your source of truth
    except Exception as e:
        # hand the error back to the model. a raised exception kills the run.
        return f"lookup failed for {project}: {e}. say so, don't guess."
    out = render(data)
    return out[:MAX_CHARS] + ("\n[truncated]" if len(out) > MAX_CHARS else "")


agent = create_agent(
    model=model,
    tools=[project_status],
    system_prompt=open("prompts/status.md").read(),
    middleware=[
        ModelCallLimitMiddleware(run_limit=10, exit_behavior="end"),
        ToolCallLimitMiddleware(run_limit=15),
    ],
)
```

Heads up, middleware names and params have moved between releases. Check the reference for the version you pinned.

Stuff in there that came from getting burned:

**Cap the loop.** Without limits an agent that can't find something will keep calling the same tool with slightly different arguments forever. It's not being dumb, it's trying. It's also burning tokens on your gateway bill. A hard ceiling per run turns "spun for four minutes" into "gave up and said so."

**Errors go back to the model as text.** A tool that throws kills the run and the user gets nothing. A tool that returns "lookup failed, say so" gets you an answer that admits it doesn't know. That's the behavior you want for a status bot. Making stuff up about a project's status in front of 60 managers is the fastest way to get the bot uninstalled.

**Truncate tool output.** Ticket lists get long. Dump one raw into context and the next model call is slow, expensive, and dumber. Tools should return what the model needs to answer, not everything they can find.

## hosting it on LangSmith

I didn't want to run another service for this, so it's on LangSmith Deployment (what used to be called LangGraph Platform). You give it a `langgraph.json` and it builds and runs an Agent Server with threads, runs and persistence already wired up.

```json
{
  "dependencies": ["."],
  "graphs": {
    "status_agent": "./agent.py:agent"
  },
  "env": ".env"
}
```

The `env` file is for local dev only. In the deployment, set env vars in the deployment config. Don't ship a `.env` in the image.

The big win is that persistence is somebody else's problem. Each Slack thread maps to an agent thread, so follow-up questions have the earlier conversation without me running a Postgres for checkpoints.

## the slack side is where it actually breaks

The fix for the triple answers: ack immediately, kick off a background run, post the result when it's done. Never run the agent inside the Slack request.

```python
# slack_events.py
import os
import uuid

from fastapi import FastAPI, Request, Response
from langgraph_sdk import get_client

app = FastAPI()
lg = get_client(url=os.environ["LANGSMITH_DEPLOYMENT_URL"], api_key=os.environ["LANGSMITH_API_KEY"])
NS = uuid.UUID(os.environ["THREAD_NAMESPACE"])


@app.post("/slack/events")
async def events(req: Request):
    if req.headers.get("x-slack-retry-num"):
        return Response(status_code=200)  # we already got it. don't start another run.

    body = await req.json()  # verify the slack signature before this. skipped here.
    if body.get("type") == "url_verification":
        return {"challenge": body["challenge"]}

    ev = body["event"]
    if ev.get("bot_id"):
        return Response(status_code=200)  # don't answer ourselves

    # same slack thread -> same agent thread, every time
    thread_ts = ev.get("thread_ts", ev["ts"])
    thread_id = str(uuid.uuid5(NS, f"{ev['channel']}:{thread_ts}"))
    await lg.threads.create(thread_id=thread_id, if_exists="do_nothing")

    await lg.runs.create(
        thread_id,
        "status_agent",
        input={"messages": [{"role": "user", "content": ev["text"]}]},
        metadata={"slack_user": ev["user"], "channel": ev["channel"]},
        webhook=os.environ["RUN_DONE_WEBHOOK"],  # posts the answer back to the thread
        multitask_strategy="enqueue",  # two quick questions in one thread run in order
    )
    return Response(status_code=200)
```

A couple of notes. Deterministic thread IDs from channel plus `thread_ts` means no lookup table. And `multitask_strategy` matters more than you'd think. People fire off a second question before the first answer lands. Without it you get two runs fighting over the same thread state.

Dropping retries by header is the blunt version. If you want to be careful, dedupe on `event_id` too.

## traces or it didn't happen

When a manager says "the bot told me X was green and it isn't," you need that exact run. That's what the metadata on `runs.create` is for. Slack user and channel go on every run, so finding the trace in LangSmith is a filter, not an archaeology project. You see every tool call, what came back, and what the model did with it. Most of the time the agent reported exactly what the source said and the source was wrong.

Before any prompt change I run it against a dataset of real questions people asked, pulled from those same traces. Same habit I have with RAG evals. Prompt changes that look harmless break stuff all the time, and you only find out if you check.

Most "agent in prod" problems aren't model problems. They're timeouts, retries and state, same as any other distributed system, and the agent just makes each one cost tokens.
