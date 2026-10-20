---
layout: post
title: "The service was up. The data was an hour old."
date: 2026-10-20
tags: [sre, slo, prometheus, observability]
---

Every availability dashboard was green. The API was answering in milliseconds. 200s all day. And it was handing out network state that hadn't been updated in a long time, because the thing that refreshed it had quietly stopped.

Nobody got paged. Why would they. Every SLO we had measured whether the service answered, not whether the answer was still true.

This was at a bank I worked at, supporting trading desks. We were standing up the first real error budget program there, with a 99.95 availability target on the important stuff. Critical incidents ended up down by more than a third. Part of getting there was writing SLOs about things other than "is it up." The one I'd push on anyone is freshness.

## the idea

For anything that serves state produced somewhere else, there are two questions. Is it answering? And is what it's answering with recent enough to trust? The second one is its own SLI. It doesn't fall out of the first.

So the SLI is: fraction of time the state is younger than some threshold. That's it. Pick the threshold from what the consumers can actually tolerate, not from what the refresh job normally does.

## emit the right timestamp

The whole thing hangs off one gauge: when did state last successfully land. Not when the job started. Not when it last ran. When it finished and the result was actually applied.

```python
from prometheus_client import Gauge

LAST_SUCCESS = Gauge(
    "network_state_last_success_timestamp_seconds",
    "unix time the last refresh was fully applied",
)

def refresh():
    state = fetch_upstream()   # can hang, can partially fail
    validate(state)            # raises on garbage
    apply(state)
    LAST_SUCCESS.set_to_current_time()  # only after apply. not before.
```

Got this wrong the first time. The gauge was set at the top of the loop. Job hung on the upstream call, timestamp said "just now," freshness looked perfect. Basically measuring that a loop started, which is worthless.

Also bump it on every successful run even if nothing changed. "Last changed" is a different metric. If upstream is quiet for an hour that's fine. If you couldn't confirm it for an hour, that's not.

## the recording rule

```yaml
groups:
  - name: netstate-slo
    interval: 30s
    rules:
      # 1 if fresh, 0 if stale. 300s is a placeholder, set it from consumers.
      - record: netstate:fresh:bool
        expr: |
          ((time() - max(network_state_last_success_timestamp_seconds)) < bool 300)
          or vector(0)
```

The `or vector(0)` is the line that matters. Without it, if the exporter dies or the target drops out of scrape, `max()` returns nothing, the rule returns nothing, and `avg_over_time` happily averages over the gaps. Your SLO looks great because there's no data saying it isn't. Missing data has to count as stale. Same reason I fail closed everywhere else.

## alerting on burn, not on age

First instinct is an alert like "age > 300 for 5m." Works, but it pages on every blip and teaches people to ignore it. Burn rate is better. Same multi-window pattern as for availability:

```yaml
      - alert: NetstateFreshnessFastBurn
        # 0.001 = budget for a 99.9 target. example number, pick yours.
        expr: |
          (1 - avg_over_time(netstate:fresh:bool[1h])) > (14.4 * 0.001)
          and
          (1 - avg_over_time(netstate:fresh:bool[5m])) > (14.4 * 0.001)
        for: 2m
        labels:
          severity: page
      - alert: NetstateFreshnessSlowBurn
        expr: |
          (1 - avg_over_time(netstate:fresh:bool[6h])) > (6 * 0.001)
          and
          (1 - avg_over_time(netstate:fresh:bool[30m])) > (6 * 0.001)
        for: 15m
        labels:
          severity: ticket
```

Long window says it's real. Short window says it's still happening. Both have to be true. Check the multipliers against your target and SLO period, those are the usual ones for 30 days but don't just paste them.

## what it caught

Two incidents, caught early. Availability checks had nothing to say about either one, because the service was answering fine. The freshness burn alert is what flagged them. That's the whole pitch. Without it you're waiting for someone downstream to notice the data looks weird.

## stuff that bit me

**Clock skew.** `time()` is the Prometheus server's clock, the timestamp is the app's clock. If they drift by a minute your freshness drifts by a minute. Keep NTP sane, or keep the threshold loose enough that it doesn't matter.

**Multiple replicas.** If several refreshers write state, `max()` says fresh as long as any one of them is working. Sometimes that's right. If each replica owns a shard, it's wrong, and you want `min()` across shards instead. Know which one you have.

**The threshold debate.** People want it tight because tight sounds rigorous. Then it fires every time upstream has a slow minute and the budget's gone by Wednesday. Start loose, tighten it once you know what normal looks like.

**Getting it into the review.** Freshness budget went in the same weekly error budget review as availability. If it's on a separate dashboard nobody opens, it doesn't exist.

Up is the easy part to measure. If your service hands out state, measure whether the state is still true, because that's the part your users are trusting.
