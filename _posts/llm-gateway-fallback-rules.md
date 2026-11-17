---
layout: post
title: "Your LLM fallback is probably making outages worse"
date: 2026-10-11
tags: [llm, ai-gateway, litellm, vertex-ai, bedrock]
---

Everybody wants "if Vertex is down, fail over to Bedrock." Sounds easy. Add a fallback list, done.

Then a single team blows through its quota, gets a wall of 429s, and the gateway happily reroutes all of it to the backup provider. Now two providers are getting hammered instead of one, and the team that caused it never even noticed because their calls "worked."

I run the LLM gateway at my day job. 150+ production agents go through it, Vertex AI primary, Bedrock as fallback, LiteLLM in the middle. Most of what I've learned is about when *not* to fall back.

## the basic setup

Same model family on both sides. That part's non-negotiable. If your fallback is a different model, every outage turns into a quality incident too. Agents that worked fine start doing weird stuff with tool calls and nobody can tell if it's the outage or the model.

```yaml
model_list:
  - model_name: claude-sonnet              # what agents ask for
    litellm_params:
      model: vertex_ai/claude-sonnet-4-5@20250929
      vertex_project: os.environ/VERTEX_PROJECT
      vertex_location: us-east5

  - model_name: claude-sonnet-bedrock      # never called directly
    litellm_params:
      model: bedrock/us.anthropic.claude-sonnet-4-5-20250929-v1:0
      aws_region_name: us-east-1

router_settings:
  num_retries: 2          # retry the primary first
  timeout: 60
  allowed_fails: 3        # then cool the deployment down
  cooldown_time: 60
  fallbacks:
    - claude-sonnet: ["claude-sonnet-bedrock"]

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL   # needed for spend tracking
```

Double check the exact model IDs for your regions. They change, and Bedrock wants the inference profile ID in a lot of regions, not the bare model ID.

Agents only ever see `claude-sonnet`. They don't know there are two providers. That's the point.

## rules I actually care about

**1. Retry before you fall back.** Most provider errors are blips. A couple of retries with backoff clears them. Jumping straight to the other provider on the first 503 means you're splitting traffic across two clouds for no reason, and your cost and latency numbers get noisy.

**2. Not every error is a fallback error.** A 400 is a bad request. Sending it to Bedrock gets you another 400, just slower. Context too long is its own thing (LiteLLM has a separate `context_window_fallbacks` for that, different problem). Fallback is for the provider being sick: timeouts, 5xx, capacity errors.

**3. Your own quota isn't an outage.** This is the one from the top. If a team hits *their* limit, that's the system working. They should get the 429. Don't let the gateway turn one team's runaway loop into load on the backup provider that everybody else is counting on.

**4. Streaming can't fail over mid-response.** Once tokens are going out to the client, you're committed. If the stream dies halfway, the agent sees a broken response, not a seamless switch. Fallback only really helps before the first token. Agents need their own retry for this. The gateway can't fix it.

**5. Prompt caching doesn't come with you.** Cache lives at the provider. Fail over and your first few calls on the backup are full price and slower. Fine for an outage. Bad if you're flapping between providers every few minutes, because now you're paying for cache misses on both sides. That's what the cooldown is for.

## per-team keys or you're flying blind

Every team gets its own key. Not one shared key per environment. The gateway tags every request with the team, so spend and token counts roll up per team without anyone filling out a spreadsheet.

```bash
# one team, one budget, one model allowlist
curl -s -X POST "$PROXY/team/new" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "team_alias": "support-agents",
    "models": ["claude-sonnet"],
    "max_budget": 500,
    "budget_duration": "30d",
    "tpm_limit": 200000
  }'

# key scoped to that team
curl -s -X POST "$PROXY/key/generate" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"team_id": "<team_id from above>", "key_alias": "support-agents-prod"}'
```

The numbers there are placeholders. Set them off what the team actually uses, not a guess.

This is also what makes rule 3 possible. When a limit trips, you know whose limit it is, and it stops at that team.

The other thing it buys you: when finance asks why the AI bill jumped, you answer with a team name and a date instead of "uh, agents."

## what I watch

- fallback count over time. Should be close to zero. If it's always a little above zero, something in the primary path is flaky and fallback is hiding it.
- 429s by team. Tells you who's about to come ask for more budget.
- p95 latency per provider. Bedrock and Vertex won't match, and agents with tight timeouts will notice the difference during an incident.

Basically, fallback is insurance for when a provider's actually down. Treat it like that. If it fires on a normal Tuesday, that's a bug you haven't found yet.
