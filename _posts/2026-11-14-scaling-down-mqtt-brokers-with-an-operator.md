---
layout: post
title: "kubectl scale is not how you shrink an MQTT cluster"
date: 2026-11-14
tags: [kubernetes, operators, go, emqx, mqtt]
---

Take an EMQX cluster running as a StatefulSet. Go from 5 replicas to 4 with `kubectl scale`. The StatefulSet kills `emqx-4`, every client connected to it gets dropped in the same instant, and they all reconnect in the same few seconds. Onto the 4 nodes that are left. Which were already busy, because that's why you had 5.

Anything with a persistent session on that node loses it too. Clients that connected with clean start off and expected their queued messages to be waiting come back to nothing.

Nothing in Kubernetes stops this. A PodDisruptionBudget doesn't help, PDBs only cover evictions, and a StatefulSet scale-down isn't an eviction. The controller just deletes the pod.

At my day job the EMQX clusters are managed by a Go operator I wrote. Scale-down is the reason that operator exists. Scale-up is boring. Anybody can bump a replica count.

## what you actually want

Before a broker node goes away:

1. Tell EMQX to evacuate it. New connections stop landing there, existing clients get disconnected at a controlled rate, and sessions move to the other nodes.
2. Wait until it's basically empty.
3. Then drop the replica count by one.

One node at a time. Always the highest ordinal, because that's the one the StatefulSet is going to kill. Evacuating `emqx-2` and then watching the StatefulSet delete `emqx-4` is a fun way to cause two incidents.

EMQX 5 has node evacuation built in, exposed through the REST API. Heads up, which versions and editions ship it has changed over time, and the endpoint path and body fields have moved around too. Check the API docs for whatever you're running before you copy this.

```go
// emqx.go
// start evacuation on one node. rates are per second, keep them low.
func (c *Client) StartEvacuation(ctx context.Context, node string) error {
	body, _ := json.Marshal(map[string]any{
		"conn_evict_rate": 100,
		"sess_evict_rate": 100,
	})
	url := fmt.Sprintf("%s/api/v5/load_rebalance/%s/evacuation/start", c.base, node)
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, url, bytes.NewReader(body))
	if err != nil {
		return err
	}
	req.SetBasicAuth(c.key, c.secret) // api key + secret, not the dashboard login
	req.Header.Set("Content-Type", "application/json")

	resp, err := c.http.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	if resp.StatusCode >= 300 {
		b, _ := io.ReadAll(resp.Body)
		// "already evacuating" comes back as an error on some versions. that's fine for us.
		if bytes.Contains(b, []byte("already")) {
			return nil
		}
		return fmt.Errorf("evacuate %s: %d %s", node, resp.StatusCode, b)
	}
	return nil
}
```

Rates are placeholders. Set them off what your remaining nodes can absorb in reconnects per second, not off how fast you want the scale-down to finish.

## the reconcile loop

The part people get wrong writing their first operator: they put the whole drain inside one Reconcile call. Start evacuation, sleep in a loop until connections hit zero, scale down. Works on a laptop. In a cluster the operator pod gets rescheduled mid-drain and all that progress lived in a goroutine that's gone now.

Reconcile has to be a step function. Look at the world, do the next thing, write down where you are in status, requeue. If the operator restarts at any point, the next Reconcile reads status and picks up where it left off.

```go
// broker_controller.go
const drained = 10 // a few stragglers is fine, zero can take forever

func (r *BrokerReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := log.FromContext(ctx)

	var b mqv1.Broker
	if err := r.Get(ctx, req.NamespacedName, &b); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}
	var sts appsv1.StatefulSet
	if err := r.Get(ctx, req.NamespacedName, &sts); err != nil {
		return ctrl.Result{}, err
	}

	current := *sts.Spec.Replicas
	want := b.Spec.Replicas

	if want >= current {
		// scale up, or someone changed their mind mid-drain
		if b.Status.Evacuating != "" {
			// best effort. the node may already be gone (see below).
			if err := r.emqx.StopEvacuation(ctx, r.nodeName(&b, b.Status.Evacuating)); err != nil {
				log.Info("stop evacuation failed, moving on", "node", b.Status.Evacuating, "err", err)
			}
			b.Status.Evacuating = ""
			if err := r.Status().Update(ctx, &b); err != nil {
				return ctrl.Result{}, err
			}
		}
		return ctrl.Result{}, r.setReplicas(ctx, &sts, want)
	}

	// shrinking. only ever the highest ordinal.
	victim := fmt.Sprintf("%s-%d", sts.Name, current-1)
	node := r.nodeName(&b, victim) // emqx@<pod>.<headless svc>.<ns>.svc.cluster.local

	if b.Status.Evacuating != victim {
		if err := r.emqx.StartEvacuation(ctx, node); err != nil {
			return ctrl.Result{}, err
		}
		b.Status.Evacuating = victim
		if err := r.Status().Update(ctx, &b); err != nil {
			return ctrl.Result{}, err
		}
		return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
	}

	conns, err := r.emqx.Connections(ctx, node)
	if err != nil {
		return ctrl.Result{}, err
	}
	if conns > drained {
		log.Info("draining", "node", victim, "connections", conns)
		return ctrl.Result{RequeueAfter: 15 * time.Second}, nil
	}

	// empty enough. now the sts can have it.
	if err := r.setReplicas(ctx, &sts, current-1); err != nil {
		return ctrl.Result{}, err
	}
	b.Status.Evacuating = ""
	return ctrl.Result{}, r.Status().Update(ctx, &b)
}
```

Going from 5 to 2 runs this three times. Each pass drains one node, drops one replica, and the next Reconcile sees `want < current` again and starts on the next one. No loop inside Reconcile. The requeue is the loop.

Order matters at the bottom. Scale the StatefulSet first, then clear status. If the operator dies between the two, the next pass sees `Evacuating` set to a pod that no longer exists, starts evacuating the new highest ordinal, and that's correct anyway. Do it the other way around and a crash in between means you scale down a node you never evacuated.

## stuff that'll bite you

**Users editing the StatefulSet directly.** If someone runs `kubectl scale sts` the operator never sees a scale-down request, it just sees a StatefulSet that doesn't match spec. I reconcile it back. The operator owns replicas on that StatefulSet and nothing else gets to touch them. Say that loudly in the README.

**Drains that never finish.** Some clients reconnect to the same node by IP and ignore the evacuation. Some just hang there. Put a deadline on it, record when evacuation started in status, and past the deadline either proceed or set a condition and stop. I set a condition and stop. A human looking at a stuck drain beats an operator quietly killing whatever's left.

**Node drains are a separate path.** Cluster upgrades evict broker pods too, and the operator isn't in that loop. A preStop hook that starts evacuation and waits, with a `terminationGracePeriodSeconds` long enough to matter, covers it. It's a backstop though. It's time-boxed and you can't see progress. Planned scale-downs go through the operator.

**Status is the source of truth, not memory.** Anything the operator needs to know after a restart goes in `.status`. If you find yourself adding a map on the reconciler struct to track which nodes are draining, stop. That map is empty after the next rollout of the operator.

The whole point of an operator is encoding the runbook a human would follow. If your stateful thing has a "do this before you remove a node" step and your scale-down path skips it, `kubectl scale` with extra steps is all you've built.
