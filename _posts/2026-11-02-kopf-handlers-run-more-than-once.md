---
layout: post
title: "Your Kopf create handler is going to run more than once"
date: 2026-11-02
tags: [kubernetes, operators, kopf, python, platform-engineering]
---

Namespace stuck in `Terminating` for days. Nothing in it but a handful of custom resources, each with a finalizer on it. The operator that was supposed to remove those finalizers lived in the same namespace. It got deleted first. So nobody was ever coming back to clean up.

That one's a classic. The other one is quieter. Create handler calls some external API, the API times out on the response but actually did the thing, Kopf retries the handler, and now you have two of whatever it made. One of them is tracked. One of them you find on the bill.

I write Kopf operators at my day job, alongside some Go ones (the EMQX operator is controller-runtime). Kopf is great for this. You get a working operator in 40 lines of Python. It also makes it really easy to forget that every handler is a retry loop.

## handlers are retries, write them like it

When a Kopf handler raises, Kopf records progress on the object and calls the whole handler again later. Not the line that failed. The whole function. Arbitrary exceptions are treated as temporary by default, so they retry with backoff. `kopf.PermanentError` stops it. `kopf.TemporaryError` lets you pick the delay.

So the rule is the same one controller-runtime people learn the hard way: a handler that isn't safe to run twice is a bug, you just haven't hit it yet.

```python
# operator.py
import logging

import kopf

import extsvc  # whatever you're managing outside the cluster. made up here.

GROUP, VERSION, PLURAL = "platform.example.com", "v1", "widgets"


@kopf.on.startup()
def configure(settings: kopf.OperatorSettings, **_):
    # name it so `kubectl get -o yaml` tells people whose finalizer this is
    settings.persistence.finalizer = "platform.example.com/cleanup"
    # info logs become k8s Events by default. that gets loud.
    settings.posting.level = logging.WARNING


def ext_name(meta):
    # deterministic. same CR -> same external name on every retry.
    # uid in there so delete+recreate with the same name doesn't collide.
    return f"{meta['namespace']}-{meta['name']}-{meta['uid'][:8]}"


@kopf.on.create(GROUP, VERSION, PLURAL, retries=20, backoff=15)
def create_widget(spec, meta, logger, **_):
    name = ext_name(meta)
    if extsvc.get(name) is None:      # we might be a retry. look first.
        extsvc.create(name, size=spec["size"])
        logger.info(f"created {name}")
    if not extsvc.ready(name):
        raise kopf.TemporaryError(f"{name} not ready", delay=20)
    return {"externalName": name}     # lands in status.create_widget


@kopf.on.delete(GROUP, VERSION, PLURAL, retries=20, backoff=15)
def delete_widget(meta, logger, **_):
    name = ext_name(meta)
    if extsvc.get(name) is None:
        return                        # already gone counts as done
    extsvc.delete(name)


@kopf.timer(GROUP, VERSION, PLURAL, interval=300, initial_delay=60)
def drift_check(meta, patch, logger, **_):
    # kopf only reacts to changes on the CR. if someone deletes the
    # external thing by hand, nothing fires. this is how you notice.
    if extsvc.get(ext_name(meta)) is None:
        logger.warning("external resource missing")
        patch.status["drift"] = "missing"
    else:
        patch.status["drift"] = None
```

Couple of things in there that came from getting burned.

**Deterministic names.** If the name has a random suffix or a timestamp, the "look first" check can't work, because the retry is looking for a different name. Derive it from the object. If the external API supports a client-supplied idempotency key, even better, use that too.

**Cap the retries.** Default is retry forever. That's fine until the external API rejects something permanently and the handler sits there for a week with a stack trace nobody reads. `retries` plus a sane `backoff` means it eventually stops and the failure shows up in status.

**Delete has to be idempotent too.** "Not found" on delete is success. If you raise on 404, the finalizer never comes off and you're back to a stuck object for a thing that was already gone.

**The timer.** People expect an operator to reconcile like a controller-runtime one, where the loop keeps comparing desired to actual. Kopf handlers are event driven on the CR. If the CR doesn't change, `create` and `update` never run again. Drift outside the cluster is invisible unless you go looking on a schedule. Heads up, timer and daemon params have shifted a bit across Kopf releases, check yours.

## the finalizer trap

Kopf adds its finalizer to every object as soon as you have a non-optional `@kopf.on.delete` handler. That's what you want. Nothing should vanish from Kubernetes while the external thing still exists.

It also means the operator has to be running for any of those objects to be deleted. Which goes wrong in exactly two ways I keep seeing:

1. Operator deployed in the same namespace as the CRs it manages. `kubectl delete ns` deletes the operator's Deployment and the CRs in parallel. Operator pod dies first. Namespace hangs forever.
2. Uninstall deletes the operator before the CRs. Same result, cluster-wide.

The fix is boring. Operator gets its own namespace. Uninstall deletes CRs first and waits for them to actually go, then removes the operator, then the CRD. If you ship it in Helm, a pre-delete hook that deletes the CRs and waits is worth the extra template.

When you're already stuck, here's how to see what's holding the namespace:

```bash
kubectl get ns team-a -o jsonpath='{.status.conditions}' | jq .

kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n1 kubectl get -n team-a --ignore-not-found --show-kind -o name
```

And the last resort, which everyone finds on Stack Overflow and runs without thinking:

```bash
kubectl patch widget my-widget -n team-a --type=merge \
  -p '{"metadata":{"finalizers":null}}'
```

That unsticks the object. It also skips your delete handler completely, so whatever it was managing outside the cluster is now orphaned. Bring the operator back up and let it clean up instead if you possibly can. If you can't, write down what you orphaned before you patch.

## one more: replicas

Two replicas of a Kopf operator without peering set up means both of them handle every event. That's two creates racing each other, which the idempotent handler mostly survives, but it's noisy and you'll get conflicts patching status. Run one replica, or set up Kopf's peering so only one is active. What Kopf does when the peering CRDs are missing has changed between versions, so read the startup logs instead of assuming.

Kopf makes writing an operator easy, which is the problem. The 40-line version works in the demo and stops working the first time an API call fails halfway through.
