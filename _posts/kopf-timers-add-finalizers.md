---
layout: post
title: "Your Kopf timer just put a finalizer on everything"
date: 2026-11-10
tags: [kubernetes, operators, kopf, python, platform-engineering]
---

Somebody deletes a test namespace. It sits in `Terminating` for an hour. `kubectl get ns -o yaml` says it's waiting on finalizers for one of our custom resources. The finalizer is `kopf.zalando.org/KopfFinalizerMarker`. The operator that owns it was scaled to zero the day before for an unrelated upgrade.

Nobody wrote a deletion handler for that CRD. Nobody asked for a finalizer. Kopf added one anyway, because someone added a timer.

I write a fair amount of Kopf at my day job, next to some Go operators (EMQX ones mostly). Kopf is great for small platform operators. You get a working controller in an afternoon. The catch is that it hides enough of the machinery that it's easy to ship something that doesn't behave like a controller at all.

## handlers aren't reconcilers

Here's what most first Kopf operators look like:

```python
@kopf.on.create("platform.example.com", "v1", "tenantconfigs")
def create_fn(spec, name, namespace, **_):
    make_the_configmap(name, namespace, spec)
```

Works in the demo. Then someone hand-edits the ConfigMap, or deletes it, and nothing happens. Ever. `on.create` fired once and that was its whole job.

Kopf is event-driven. Handlers run on changes to the object they watch. That's not the same as level-triggered reconciliation, where you keep comparing desired against actual and fix the difference. If you come from controller-runtime in Go, this is the thing that'll catch you, because there `Reconcile` is the only thing and drift gets fixed on the next resync for free.

So you write one reconcile function, make it idempotent, and hang it off every trigger:

```python
import kopf
import kubernetes
from kubernetes.client import ApiException

GROUP, VERSION, PLURAL = "platform.example.com", "v1", "tenantconfigs"  # names made up


@kopf.on.startup()
def startup(settings: kopf.OperatorSettings, **_):
    try:
        kubernetes.config.load_incluster_config()
    except kubernetes.config.ConfigException:
        kubernetes.config.load_kube_config()  # local dev


def desired(name, namespace, spec):
    return {
        "apiVersion": "v1",
        "kind": "ConfigMap",
        "metadata": {
            "name": f"{name}-config",
            "namespace": namespace,
            "labels": {"app.kubernetes.io/managed-by": "tenant-operator"},
        },
        "data": {k: str(v) for k, v in spec.get("settings", {}).items()},
    }


@kopf.on.create(GROUP, VERSION, PLURAL)
@kopf.on.update(GROUP, VERSION, PLURAL)
@kopf.on.resume(GROUP, VERSION, PLURAL)  # operator restarted, check everything again
def reconcile(body, spec, name, namespace, logger, **_):
    cm = desired(name, namespace, spec)
    kopf.adopt(cm, owner=body)  # ownerReference, so GC deletes it with the parent
    api = kubernetes.client.CoreV1Api()
    cm_name = cm["metadata"]["name"]

    try:
        live = api.read_namespaced_config_map(cm_name, namespace)
    except ApiException as e:
        if e.status != 404:
            raise
        api.create_namespaced_config_map(namespace, cm)
        logger.info(f"created {cm_name}")
        return

    if (live.data or {}) != cm["data"]:
        cm["metadata"]["resourceVersion"] = live.metadata.resource_version
        try:
            api.replace_namespaced_config_map(cm_name, namespace, cm)
        except ApiException as e:
            if e.status == 409:
                raise kopf.TemporaryError("conflict, retrying", delay=10)
            raise
        logger.info(f"fixed drift on {cm_name}")
```

Replace, not patch. A merge patch on `data` leaves keys somebody added by hand. If the operator owns the object, it owns the whole thing.

And `kopf.adopt` is doing more than it looks like. With an owner reference, Kubernetes garbage collection cleans up the ConfigMap when the parent goes away. No `on.delete` handler needed. That matters for the next part.

## the timer

That's still not drift detection. It fixes the child when the *parent* changes or the operator restarts. If someone deletes the ConfigMap on a quiet Tuesday, it stays gone until one of those happens.

The obvious fix is a timer:

```python
@kopf.timer(GROUP, VERSION, PLURAL, interval=300, initial_delay=30)
def resync(body, spec, name, namespace, logger, **_):
    reconcile(body=body, spec=spec, name=name, namespace=namespace, logger=logger)
```

Five minutes, every object. Done. Except this is the line that wedged the namespace.

Kopf adds its finalizer to every object it serves when any handler for that resource needs one. Non-optional `on.delete` handlers need one, which most people know. Timers and daemons need one too, which most people don't. Kopf wants to stop the timer cleanly before the object is gone, so it holds deletion until it can. I went and read the source to be sure: in the version I'm on (1.4x) both `@kopf.timer` and `@kopf.daemon` register with `requires_finalizer=True`. No flag to turn it off.

So the moment you add a timer, every CR of that kind gets a finalizer. Operator is up, nobody notices. Operator is down, every delete of that kind hangs, and anything containing them hangs too. Namespace deletes. Teardown jobs in CI. `helm uninstall` on a test env.

## what I do now

**If you only need a few checks, watch the children instead.** For "someone deleted my ConfigMap," you don't need a timer at all. Label what you create and react to its events. `on.event` doesn't add finalizers.

```python
@kopf.on.event("", "v1", "configmaps", labels={"app.kubernetes.io/managed-by": "tenant-operator"})
def child_event(event, body, logger, **_):
    if event["type"] != "DELETED":
        return
    for ref in body["metadata"].get("ownerReferences", []):
        if ref["kind"] == "TenantConfig":
            ns = body["metadata"]["namespace"]
            parent = kubernetes.client.CustomObjectsApi().get_namespaced_custom_object(
                GROUP, VERSION, ns, PLURAL, ref["name"]
            )
            reconcile(body=parent, spec=parent["spec"], name=ref["name"], namespace=ns, logger=logger)
```

That's why `desired()` sets the label. Heads up, the parent might be mid-delete itself when this fires, so in real code check its `deletionTimestamp` before you recreate anything.

**If you keep the timer, treat the operator like it's on the critical path.** Because it is now. Two replicas, a PDB, peering on. Heads up on peering: Kopf auto-detects it from whether the peering CRDs are installed, and without them it runs standalone, which means two replicas both handle every event. With peering it's active/passive, one does the work and the other waits. You want the second one. Check the peering docs for your version.

**Write the runbook before you need it.** When the operator really is gone for good, you strip the finalizer by hand. Don't null out the whole list, other controllers might have their own on there. Remove just Kopf's:

```bash
kubectl get tenantconfigs -A -o json \
  | jq -r '.items[] | select(.metadata.finalizers // [] | index("kopf.zalando.org/KopfFinalizerMarker"))
           | "\(.metadata.namespace) \(.metadata.name)"' \
  | while read ns n; do
      kubectl -n "$ns" patch tenantconfig "$n" --type=json -p "$(
        kubectl -n "$ns" get tenantconfig "$n" -o json \
          | jq -c '[{op:"replace", path:"/metadata/finalizers",
                     value:(.metadata.finalizers - ["kopf.zalando.org/KopfFinalizerMarker"])}]')"
    done
```

**Uninstall order is a thing.** Delete the CRs, wait, then delete the operator. Reverse it and you're back at the top of this post. Put that in whatever tears environments down, not in a wiki page.

Kopf makes the happy path short. The finalizers it adds for you are still yours to own.
