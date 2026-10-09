---
layout: post
title: "Stop autoscaling vLLM on CPU"
date: 2026-10-09
tags: [kubernetes, vllm, keda, gpu]
---

Put vLLM behind a normal HPA and watch what happens. CPU sits around 20, the HPA thinks the pod is bored, and meanwhile requests are stacking up behind a GPU that's completely pinned. Users get a spinner. Your dashboards look green.

I've hit this exact bug before, just not with LLMs. At SMBC we had an async video processing pipeline that was blowing its SLA by 45 minutes. Workers were scaled on CPU. The real backlog was sitting in SQS where the HPA couldn't see it. Switched the scaler to queue depth with KEDA and the problem went away. Same fix works for inference.

## why CPU lies

vLLM does almost all its real work on the GPU. CPU is tokenization, scheduling, HTTP. Cheap stuff. So CPU% tells you basically nothing about whether the pod can take more load.

GPU util isn't much better. vLLM does continuous batching, so a GPU at 95 could be handling load fine or could be one request away from queueing. You can't tell from utilization.

The question you actually want answered: are requests waiting? Yes means add a replica. Nothing waiting for a while means remove one.

## the metric

vLLM exposes Prometheus metrics on `/metrics`. The ones I care about:

- `vllm:num_requests_waiting` - queued, not scheduled on the GPU yet
- `vllm:num_requests_running` - in flight

Heads up, metric names have moved around between vLLM versions (the KV cache gauge got renamed at some point). Curl `/metrics` on whatever version you're actually running before you wire alerts or scalers to it.

## KEDA config

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: vllm-llama
spec:
  scaleTargetRef:
    name: vllm-llama
  minReplicaCount: 1        # not 0, see below
  maxReplicaCount: 8
  pollingInterval: 15
  cooldownPeriod: 600
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 600
          policies:
            - type: Pods
              value: 1
              periodSeconds: 300
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring:9090
        query: sum(vllm:num_requests_waiting{deployment="vllm-llama"})
        threshold: "4"
```

Roughly one replica per 4 waiting requests. Tune the threshold against your TTFT target, don't guess it.

## stuff that will bite you

**Cold starts are minutes.** New pod has to land on a GPU node, pull a huge image, load weights, warm up. If the cluster autoscaler also has to bring up the node, add more. That's why min is 1. I'm a big fan of scale to zero for bursty internal stuff (moved 50+ services onto Knative at SMBC), but for a model people are waiting on, the first request after idle just times out. Keep one warm.

**Scale down slow.** Default scale-down behavior is tuned for stateless web pods. Here it thrashes: queue drains, replica gets killed, queue builds, new replica spends minutes loading weights. Long cooldown, one pod at a time. You burn a bit of idle GPU and save a lot of latency. I ended up tuning cooldowns per traffic window on the SMBC pipeline for the same reason.

**Isolate the GPU nodes.** Own node pool, taint it, tolerations plus an `nvidia.com/gpu` request on the vLLM pods. Otherwise some random workload lands on a GPU node and the next inference replica can't schedule right when you need it.

## how I check it's working

One dashboard, three panels:

- `num_requests_waiting` over time
- replica count
- p95 time to first token

Queue spikes, replicas follow with a lag about equal to your cold start, TTFT recovers. If replicas go up and TTFT doesn't come back, your bottleneck is somewhere else (usually node provisioning or image pulls) and scaling faster won't fix it.

Basically every autoscaling problem I've debugged was someone scaling on what was easy to measure instead of what users were actually waiting on. For LLM serving that's the queue.
