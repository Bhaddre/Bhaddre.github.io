---
layout: post
title: "The SLO that paged before anything was down"
date: 2026-10-17
tags: [sre, slo, prometheus, kubernetes, incidents]
---

Availability SLO at 100 for the week. Error rate flat. Latency flat. And a trading desk is on the phone because traffic is going somewhere it shouldn't.

The system was up. It was just serving stale state. Nothing in the availability math can see that, because every request got a 200. The 200s were wrong.

This was at a bank I worked at. I ran the first formal error-budget program there, 99.95 availability target across the Kubernetes estate behind the trading desks. The availability SLOs did their job, critical incidents dropped by more than a third over the time we ran it. But the SLO I'd keep if I could only keep one wasn't an availability SLO. It was a freshness SLO on network state, and it caught two incidents early that nothing else would have.

## what "network state" means here

We had a controller that took network state from a source of truth (routes, policies, endpoint info, that kind of thing) and pushed it into the clusters. Classic reconciler. Read desired, apply, loop.

The failure mode of a reconciler is almost never "it crashed." Crashes are easy. Pod restarts, you get an alert, done. The bad one is: the pod is Running, the readiness probe is green, the loop is spinning, and it hasn't applied anything new in a long time. Upstream token expired. Upstream stopped producing. One cluster fell off the fan-out. The loop logs "reconciled" every 30 seconds and means nothing by it.

Availability can't see this. Liveness can't see this. The only question that catches it is: how old is the state this cluster is actually running on right now?

## the metric

One gauge per cluster. The important part is what you put in it.

Don't set it to `time.time()` after the loop runs. That's a heartbeat, and a heartbeat only tells you the loop is alive, which we already knew from the readiness probe. Set it to the timestamp of the state you just applied, as stamped by the source of truth. Then if upstream freezes, the gauge freezes with it, and staleness keeps growing even though your controller is "working."

```python
from prometheus_client import Gauge

# generated_at comes from the source of truth, not from us
APPLIED_STATE_TS = Gauge(
    "netstate_applied_generated_at_seconds",
    "source timestamp of the network state last verified in the cluster",
    ["cluster"],
)

def reconcile(cluster):
    desired = fetch_desired(cluster)
    apply(cluster, desired)
    actual = read_back(cluster)
    if actual != desired:
        # don't stamp. applied != verified.
        raise ReconcileError(f"{cluster}: readback mismatch")
    APPLIED_STATE_TS.labels(cluster=cluster).set(desired.generated_at)
```

Two things in there that matter. Stamp after readback, not after apply. And stamp the source's time, not yours.

## the SLO

Staleness is `now - that gauge`. The SLI is the fraction of time staleness stays under a bound. We used a couple of minutes, pick yours off what the consumers can actually tolerate, not off what the controller usually does.

{% raw %}
```yaml
groups:
  - name: netstate-freshness
    rules:
      - record: netstate:staleness_seconds
        expr: time() - max by (cluster) (netstate_applied_generated_at_seconds)

      # 1 when fresh, 0 when stale. average it over a window and that's the SLI
      - record: netstate:fresh
        expr: netstate:staleness_seconds < bool 120

      - record: netstate:fresh:ratio_rate5m
        expr: avg_over_time(netstate:fresh[5m])
      - record: netstate:fresh:ratio_rate1h
        expr: avg_over_time(netstate:fresh[1h])

      # fast burn, standard multiwindow. 14.4x on a 99.95 objective
      - alert: NetworkStateFreshnessBurn
        expr: |
          (1 - netstate:fresh:ratio_rate1h) > (14.4 * 0.0005)
          and
          (1 - netstate:fresh:ratio_rate5m) > (14.4 * 0.0005)
        for: 2m
        labels:
          severity: page
        annotations:
          summary: "{{ $labels.cluster }} network state is stale, burning budget fast"

      # the dumb one. keep it. if the series disappears, nothing above fires.
      - alert: NetworkStateMetricMissing
        expr: absent(netstate_applied_generated_at_seconds)
        for: 5m
        labels:
          severity: page
```
{% endraw %}

Heads up on the `absent()` rule. If the controller dies outright, Prometheus marks the series stale after a few minutes and it drops out of every expression above. `avg_over_time` happily averages the samples that exist and reports everything fine. So the freshness alert goes quiet exactly when things are worst. `absent()` with `by (cluster)` doesn't do what you'd want either, you can't be absent per label value you never saw. We ended up with one absent rule per cluster, generated from the cluster list. Ugly. Required.

## what it bought us

The two incidents it caught weren't outages. That's the point. Both were the kind of thing where nothing is down yet, the controller is alive, and state has quietly stopped moving. Freshness alert fired, somebody looked, upstream got fixed before any desk noticed. Under an availability-only SLO both would've been found by a human noticing something weird, which at a bank means found by the people you least want finding it.

It also changed the error budget conversation. Teams were used to "availability is fine, ship it." Once staleness was burning budget too, "fine" had to mean the data was current, not just that the pods answered. That's a better definition of fine.

## where this generalizes

Anything with a reconciler, a cache, a replica, or a pipeline has this problem. Config pushers. DNS. Feature flags. Service discovery. Every GitOps setup (your Argo app is Synced and Healthy against a commit from how long ago?). RAG indexes, which I think about more now than routes. In every one of them the availability SLO will be green while the thing serves yesterday.

If your system's job is to reflect something else, measure how far behind the something else it is. Up isn't the same as right.
