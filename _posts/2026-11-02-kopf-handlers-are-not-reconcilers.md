---
layout: post
title: "on.create is not reconciliation"
date: 2026-11-02
tags: [kubernetes, operators, kopf, python, platform-engineering]
---

Somebody deletes a ResourceQuota by hand. Tidying up, testing something, whatever. The custom resource that created it still says `phase: Ready`. The operator that owns it is running, healthy, logs are quiet. And the quota is never coming back, because nothing about the custom resource changed, so no handler ever fires.

That's the Kopf trap. It makes writing event handlers so easy that you end up with an operator that reacts to edits on its own CR and is blind to everything else. Which is not an operator. It's a webhook with extra steps.

I write a few Kopf operators at my day job for platform plumbing, the stuff that's too small to justify a Go operator and too stateful for a Helm chart. (The bigger ones are in Go.) Every one of them started as `@kopf.on.create` and got rewritten.

## one function, every trigger

The fix was boring. One idempotent `reconcile` that converges the world to the spec, hung off every way the operator can find out it should look.

```python
# teamspace operator. names made up, shape is real.
import kopf
import kubernetes
from kubernetes.client.rest import ApiException

GROUP, VERSION, PLURAL = "platform.example.com", "v1", "teamspaces"


@kopf.on.startup()
def startup(settings: kopf.OperatorSettings, **_):
    kubernetes.config.load_incluster_config()
    settings.persistence.finalizer = "platform.example.com/teamspace"


def apply(create, patch, body):
    # create-or-patch. 409 is fine, it means we're converging, not failing.
    try:
        create(body)
    except ApiException as e:
        if e.status != 409:
            raise
        patch(body)


@kopf.on.resume(GROUP, VERSION, PLURAL)
@kopf.on.create(GROUP, VERSION, PLURAL)
@kopf.on.update(GROUP, VERSION, PLURAL, field="spec")
@kopf.timer(GROUP, VERSION, PLURAL, interval=300, idle=60)
def reconcile(spec, name, body, patch, logger, **_):
    core = kubernetes.client.CoreV1Api()
    ns = f"team-{name}"

    ns_body = {"metadata": {"name": ns, "labels": {"team": name}}}
    # TeamSpace is cluster-scoped, so it can own a namespace. GC cleans up.
    kopf.append_owner_reference(ns_body, owner=body)
    apply(
        lambda b: core.create_namespace(b),
        lambda b: core.patch_namespace(ns, b),
        ns_body,
    )

    quota = {
        "metadata": {"name": "team-quota", "namespace": ns},
        "spec": {"hard": spec.get("quota", {})},
    }
    apply(
        lambda b: core.create_namespaced_resource_quota(ns, b),
        lambda b: core.patch_namespaced_resource_quota("team-quota", ns, b),
        quota,
    )

    patch.status["phase"] = "Ready"
    patch.status["namespace"] = ns
```

What each trigger is for:

- `create` and `update` are the fast path. Someone edits the spec, you act now.
- `resume` runs on every CR when the operator starts. Without it, anything that changed while the operator was down gets ignored until the next edit.
- `timer` is the actual reconciliation. Every five minutes, compare and fix. `idle=60` keeps it off while someone's actively editing so it doesn't race the update handler.

The timer is the one people skip and it's the one that brings the quota back. Five minutes is a guess. Pick it off how long you can live with drift, not how chatty you want the operator to be.

Heads up, check the decorator kwargs against the Kopf version you pinned. `field=` on `on.update` and the timer's `idle` have been around a while, but I've been burned by assuming before.

## idempotent or it'll double up on you

Kopf retries a handler that raises. That's good. It also means a handler that did half its work, then hit a timeout on step three, runs steps one and two again. If step one was "create a thing with a random name," you now have two things.

So every side effect is create-or-patch on a deterministic name. Never generate a name inside a handler. Never append to a list in an external system without checking it's already there. If you can't make a call idempotent, that's a sign it doesn't belong in the handler.

Same deal for the error path. Raise `kopf.TemporaryError("quota api flaky", delay=30)` when retrying makes sense, `kopf.PermanentError` when it doesn't. A bad spec is permanent. Retrying it every minute forever just fills the logs.

## your CRD schema can eat the operator's memory

This one cost me an afternoon. Handlers kept re-running on an object that hadn't changed. Over and over.

Kopf keeps its own bookkeeping on the object: which handlers finished, what the last handled spec looked like. Depending on the version and settings that lives in annotations, under `status`, or both. If your CRD has a structural schema that doesn't declare those fields, the API server prunes them on write. Kopf writes "done," the API server silently throws it away, Kopf reads back "not done," runs it again.

```yaml
# in the CRD version schema
openAPIV3Schema:
  type: object
  properties:
    spec:
      type: object
      x-kubernetes-preserve-unknown-fields: true   # or a real schema, better
    status:
      type: object
      x-kubernetes-preserve-unknown-fields: true   # let kopf keep its stuff
subresources:
  status: {}
```

If you'd rather keep `status` strict, move Kopf's progress storage to annotations in the startup settings. Either way, decide it on purpose.

## finalizers are a promise to stay up

Add an `@kopf.on.delete` handler and Kopf puts a finalizer on every CR. Now deletes wait on your operator. Operator crashlooping, or scaled to zero during some migration, and every `kubectl delete` hangs. If the CR lives in a namespace someone's deleting, the namespace sits in `Terminating` too, and that ticket comes to you.

That's why the code above uses an owner reference instead. Namespace is owned by the TeamSpace, garbage collector deletes it, no finalizer, no dependency on my pod being alive. I only reach for `on.delete` when there's cleanup outside the cluster that GC can't do. When there is, it's `optional=True` unless leaking that external thing is actually worse than a stuck delete. Usually it isn't.

## one replica, Recreate

Run two replicas of a Kopf operator without peering set up and both of them handle every event. Run one replica with the default RollingUpdate and you get two for a few seconds every deploy, which is the same bug with better timing.

```yaml
spec:
  replicas: 1
  strategy:
    type: Recreate   # old pod gone before new one starts
```

A few seconds of no operator during a rollout is fine. `resume` picks up anything it missed. Two operators fighting over the same objects is not fine.

If you do want HA, install Kopf's peering CRDs and let it pick a leader. For platform plumbing I haven't needed it. The timer covers the gap.

An operator that only acts when its own CR changes isn't watching your cluster. It's waiting for someone to edit YAML.
