---
layout: post
title: "Argo CD will happily deploy into a fire"
date: 2026-10-15
tags: [argocd, gitops, kubernetes, platform-engineering]
---

Shared dependency goes unhealthy. Say the message broker half the apps talk to. Somebody's on it. Meanwhile three teams merge PRs, Argo CD syncs all three, and now the incident channel has three new rollouts to rule out before anyone can get back to the actual problem.

Argo did exactly what it was told. Git changed, cluster should match git. It has no idea the thing your app needs is on fire.

I ran into this at a bank I worked at, supporting trading desks. We went GitOps with Argo CD using App of Apps, and the number of teams deploying through it grew from 3 to 15. With 3 teams you can just yell in a channel "nobody deploy." With 15 that stops working. People are in different time zones, auto-sync is on, and the merge already happened an hour ago.

## sync waves aren't the answer

First thing everyone reaches for is sync waves on the child apps in the App of Apps. Put shared stuff in wave -1, team apps in wave 0, done.

Two problems with that.

First, heads up: since Argo CD 1.8 the health check for `Application` resources was removed, so a parent app doesn't actually wait for a child app to be healthy before moving to the next wave. You have to put it back yourself in `argocd-cm`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # restore health for child Applications so waves actually wait
  resource.customizations.health.argoproj.io_Application: |
    hs = {}
    hs.status = "Progressing"
    hs.message = ""
    if obj.status ~= nil then
      if obj.status.health ~= nil then
        hs.status = obj.status.health.status
        if obj.status.health.message ~= nil then
          hs.message = obj.status.health.message
        end
      end
    end
    return hs
```

Lots of people have waves configured and think they're getting ordering they're not getting. Check this one.

Second, and the bigger one: waves only order things inside a single sync of the parent. A team's app syncing on its own because they merged something doesn't care what wave the broker was in last Tuesday. The ordering you actually need is "don't roll me out right now if my dependencies are sick," at the moment of each individual sync.

## the gate

The fix was a PreSync hook. Small Python job that runs before every sync of an app. It reads the Argo `Application` objects for whatever that app depends on, checks their health, checks for critical alerts in the app's own namespace, and exits non-zero if anything looks bad. Non-zero PreSync hook means the sync fails and nothing gets applied.

{% raw %}
```python
#!/usr/bin/env python3
# presync gate: refuse to roll out if what we depend on is unhealthy
import os
import sys

import requests
from kubernetes import client, config

ARGO_NS = os.environ.get("ARGO_NAMESPACE", "argocd")
DEPS = [d.strip() for d in os.environ.get("DEPENDS_ON", "").split(",") if d.strip()]
PROM = os.environ.get("PROM_URL", "http://prometheus.monitoring:9090")
APP_NS = os.environ["APP_NAMESPACE"]


def dep_problems():
    config.load_incluster_config()
    api = client.CustomObjectsApi()
    out = []
    for name in DEPS:
        try:
            app = api.get_namespaced_custom_object(
                "argoproj.io", "v1alpha1", ARGO_NS, "applications", name
            )
        except client.ApiException as e:
            out.append(f"{name}: can't read app ({e.status})")
            continue
        health = app.get("status", {}).get("health", {}).get("status", "Unknown")
        if health != "Healthy":
            out.append(f"{name}: health={health}")
    return out


def alert_problems():
    q = f'ALERTS{{alertstate="firing",severity="critical",namespace="{APP_NS}"}}'
    r = requests.get(f"{PROM}/api/v1/query", params={"query": q}, timeout=10)
    r.raise_for_status()  # prometheus down = blocked. on purpose.
    return [
        f"alert firing: {s['metric'].get('alertname', '?')}"
        for s in r.json()["data"]["result"]
    ]


def main():
    if os.environ.get("GATE_BYPASS") == "true":
        print("gate bypassed")
        return 0
    problems = dep_problems() + alert_problems()
    for p in problems:
        print(f"BLOCKED {p}")
    if not problems:
        print(f"ok: {len(DEPS)} deps healthy, no critical alerts")
    return 1 if problems else 0


if __name__ == "__main__":
    sys.exit(main())
```
{% endraw %}

And the hook in each app's manifests. The golden path template shipped this by default so teams only had to fill in `DEPENDS_ON`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: presync-gate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  backoffLimit: 0             # one shot, argo handles retries
  activeDeadlineSeconds: 120
  template:
    spec:
      serviceAccountName: presync-gate
      restartPolicy: Never
      containers:
        - name: gate
          image: registry.example.com/platform/presync-gate:1.4.0
          env:
            - name: DEPENDS_ON
              value: "shared-broker,platform-ingress"   # names made up
            - name: APP_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
```

The service account needs `get` on `applications.argoproj.io` in the `argocd` namespace. That's a Role in `argocd` bound to a ServiceAccount in the team's namespace. Read-only, one resource. Don't hand it more.

Then on the `Application`, a retry policy so a blocked sync tries again on its own once things recover:

```yaml
syncPolicy:
  automated: {}
  retry:
    limit: 5
    backoff:
      duration: 1m
      factor: 2
      maxDuration: 10m
```

Nobody has to remember to re-sync after the incident. The deploy just lands when the deps go green.

## stuff I changed after running it

**Dropped the sync status check.** First version also required deps to be `Synced`. Bad idea. Any dep that was mid-deploy or had a tiny bit of drift blocked everyone downstream. Health is the thing you care about. Sync status is noise here.

**Fail closed.** If the gate can't reach Prometheus or the Kubernetes API, it blocks. I went back and forth on this. But a gate that quietly passes when it can't see is worse than no gate, because people trust it.

**The bypass goes through git.** `GATE_BYPASS` is an env var in the manifest, so turning it off is a PR. Somebody reviews it, there's a record of who and why. You'll need it, usually when the fix for the incident *is* the deploy being blocked. Which happens. Plan for it.

**Keep `DEPENDS_ON` short.** Teams wanted to list everything. If an app lists 12 dependencies it basically never deploys. Direct, hard runtime deps only. If your app degrades gracefully without something, it's not a gate dependency.

**Print why.** The job log is what people see in the Argo UI when a sync fails. `BLOCKED shared-broker: health=Degraded` saves a ping to the platform channel. A bare exit 1 doesn't.

This tied straight into the error budget stuff we were doing at the same time. A lot of what burns budget during an incident isn't the original failure, it's everything people change on top of it while it's still going.

A deploy pipeline that can't see the state of the system it's deploying into isn't doing GitOps, it's just doing `kubectl apply` on a timer.
