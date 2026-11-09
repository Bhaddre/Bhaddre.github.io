---
layout: post
title: "Your Kopf create handler is going to run more than once"
date: 2026-11-09
tags: [kubernetes, operators, kopf, python, platform-engineering]
---

Custom resource gets created. Operator calls an external API to provision the thing, then tries to write a Secret with the credentials. Secret create fails. RBAC was missing one verb. Kopf does what it's supposed to do and retries the handler a minute later.

From the top. So now there are two of the thing on the provider side, one Secret pointing at the second one, and an orphan nobody knows about until the bill or a quota shows up.

I write Kopf operators at my day job for platform glue, the stuff where a full Go operator is overkill. (The heavy stuff, like the one running EMQX, is Go.) I like Kopf a lot. But it makes it really easy to write an operator that only works when nothing fails, and the first time something fails you find out your handler wasn't safe to run twice.

## the actual problem

controller-runtime pushes you toward level-triggered code. You get a name, you look at the world, you make it match. Every call to Reconcile is "figure out what's missing." Running it twice is the normal case.

Kopf hands you events. `on.create`, `on.update`, `on.delete`. That reads like "this runs once when the thing is created." It doesn't. It runs until it succeeds. Any exception, Kopf backs off and calls it again with the same body. If the first half of your function did something with side effects, that half runs again too.

Kubernetes objects mostly save you here because a second create gets a 409. External APIs usually don't. Call "create bucket" or "create key" twice and you often just get two.

## the fix

Three things, all boring.

Split the work into separate handlers with their own ids. Kopf tracks progress per handler (in annotations on the object). If step 2 fails, step 1 is already marked done and doesn't run again on the retry. One big handler means one big retry.

Make each step look before it creates. Deterministic names, never random ones, so "does mine already exist" is a lookup.

Record the external id the second you have it.

```python
import kopf
import kubernetes
from kubernetes.client.rest import ApiException

import provider  # thin client for whatever external API. made up.

GROUP, VERSION, PLURAL = "platform.example.com", "v1", "tenants"


@kopf.on.startup()
def startup(**_):
    kubernetes.config.load_incluster_config()


def ext_name(namespace, name):
    return f"{namespace}-{name}"  # deterministic, so we can find it again


# step 1. own id. if step 2 fails, this doesn't run again.
@kopf.on.create(GROUP, VERSION, PLURAL, id="remote", retries=10, backoff=30)
def create_remote(name, namespace, spec, patch, logger, **_):
    tier = spec.get("tier")
    if tier not in ("small", "large"):
        raise kopf.PermanentError(f"bad tier {tier!r}")  # retrying won't fix a bad spec

    existing = provider.find(name=ext_name(namespace, name))
    if existing:
        logger.info(f"adopting {existing.id}, earlier attempt got this far")
        rid = existing.id
    else:
        try:
            rid = provider.create(name=ext_name(namespace, name), tier=tier).id
        except provider.RateLimited as e:
            raise kopf.TemporaryError(str(e), delay=60)
    patch.status["remoteId"] = rid


def ensure_secret(name, namespace, body, rid):
    secret = {
        "apiVersion": "v1",
        "kind": "Secret",
        "metadata": {"name": f"{name}-creds"},
        "stringData": {"id": rid, "token": provider.token_for(rid)},
    }
    kopf.adopt(secret, owner=body)  # ownerRef + namespace, GC deletes it with the CR
    try:
        kubernetes.client.CoreV1Api().create_namespaced_secret(namespace, secret)
    except ApiException as e:
        if e.status != 409:  # already exists from an earlier try = fine
            raise


# step 2. reads what step 1 recorded instead of assuming it ran in this pass.
@kopf.on.create(GROUP, VERSION, PLURAL, id="secret")
def create_secret(name, namespace, body, status, **_):
    rid = status.get("remoteId")
    if not rid:
        raise kopf.TemporaryError("remoteId not recorded yet", delay=5)
    ensure_secret(name, namespace, body, rid)
```

Heads up on `PermanentError` vs everything else. Kopf retries arbitrary exceptions by default, so a typo in the spec retries forever unless you tell it not to. `PermanentError` for "a human has to fix this," `TemporaryError` with a delay for "try later." Check the retry defaults for the Kopf version you're on, they've changed before.

Also check your CRD schema allows the status fields you write. If the CRD has a structural schema and `status` doesn't allow unknown fields, the API server prunes what you patch. Quietly. Then step 2 waits forever on a `remoteId` that keeps getting dropped. Either declare the field or put `x-kubernetes-preserve-unknown-fields: true` on status.

## create handlers don't catch drift

Somebody deletes the Secret by hand. `on.create` already succeeded, it's done forever. Nothing puts it back.

This is the other half of being edge-triggered. Add a timer that calls the same ensure function:

```python
@kopf.timer(GROUP, VERSION, PLURAL, interval=300, initial_delay=60)
def drift(name, namespace, body, status, **_):
    rid = status.get("remoteId")
    if rid:
        ensure_secret(name, namespace, body, rid)  # same function, safe to run any time
```

Because `ensure_secret` was already written to be run twice, the timer is a few lines. If it wasn't, you'd be writing the reconcile logic a second time. That's really the test: if you can't call it from a timer without worrying, it's not safe in a create handler either.

## stuff that bit me after

**Two replicas = double handling.** Someone bumped the Deployment to 2 for "HA." Without peering, both pods handle every event, and your look-before-create logic now has a race. Kopf has peering so only one instance is active, but it needs its peering CRDs installed and behavior when they're missing has varied between versions. I just run one replica with `strategy: Recreate`. A glue operator being down for 30 seconds during a rollout is fine.

**Delete handlers add a finalizer.** Needed for external cleanup, because there's no ownerReference on the provider side. But it means if the operator is gone, `kubectl delete` on the CR hangs. Make the delete handler treat 404 as done, and never uninstall the operator before its CRs. If cleanup is nice-to-have, `optional=True` on the delete handler skips the finalizer.

**Log the step.** `adopting existing x` in the logs is how you know a retry happened and the guard caught it. Without that line you can't tell the difference between "worked" and "worked the second time."

Operators don't get to assume anything ran exactly once. Write every handler like it's going to run again, because it is.
