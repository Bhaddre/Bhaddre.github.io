---
layout: post
title: "Your multi-node GPU job is probably running over TCP"
date: 2026-11-02
tags: [gpu, nvidia, infiniband, nccl, openshift]
---

Two nodes, eight GPUs each, InfiniBand between them. Distributed job starts, loss goes down, nobody complains. It's just slow. Slower than one node, some steps.

Nothing errored. NCCL couldn't use the IB fabric from inside the pod, so it picked the pod's normal network interface and carried on over TCP. It does that by default. No warning you'll ever see unless you go looking.

I ran on-prem NVIDIA GPU clusters on InfiniBand with OpenShift AI at a bank I worked at. This was the single most common way a "working" multi-node setup was actually broken. Expensive fabric sitting idle while all-reduce went over the same Ethernet as everything else.

## how to tell

Set `NCCL_DEBUG=INFO` and read the first few lines from rank 0. You're looking for which network NCCL picked.

Good:

```
NCCL INFO NET/IB : Using [0]mlx5_0:1/IB [1]mlx5_1:1/IB ...
```

Bad:

```
NCCL INFO NET/Socket : Using [0]eth0:10.128.2.15<0>
```

If you see `NET/Socket` on a box with IB cards, that's your problem. If you set up GPUDirect RDMA, the channel lines should also mention `GDRDMA`. Exact log format shifts between NCCL versions, so grep loosely.

The real fix is making NCCL refuse to fall back. `NCCL_NET=IB` forces the IB transport, and if it can't initialize, the job fails at startup instead of limping along. Failing at minute one beats finding out from a throughput graph a week later. Check that your NCCL version honors it, it's been around a while but not forever.

## why the pod can't see the fabric

The host has IB. The container doesn't automatically. On OpenShift it took four separate things, and missing any one of them gets you `NET/Socket`.

**The RDMA device has to be in the pod.** The NVIDIA Network Operator runs the RDMA shared device plugin, which exposes the HCAs as an extended resource. Request it like you request GPUs. The resource name comes from the plugin config, so check yours, mine below is the common default.

**The memlock limit.** RDMA pins memory. Default container memlock is tiny, verbs registration fails, NCCL shrugs and moves to sockets. You need `IPC_LOCK`, and on OpenShift that means an SCC that allows it. `restricted-v2` won't. Make a narrow one for the training service account. Don't hand out `privileged` because it's faster.

**`/dev/shm`.** Not IB, but it bites the same jobs. NCCL uses shared memory between GPUs on the same node and the container default is 64Mi. Mount a memory-backed emptyDir.

**GPUDirect RDMA needs the peer memory module.** Without it traffic still goes over IB, but bounces through host memory. In the GPU Operator that's the RDMA setting on the driver (`driver.rdma.enabled`), which loads `nvidia-peermem`. Check the flag name for your operator version.

Here's the shape of a PyTorchJob worker with all of that in it:

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: finetune
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template: &pod
        metadata:
          annotations:
            k8s.v1.cni.cncf.io/networks: ib-net   # secondary net via multus, if you use one
        spec:
          serviceAccountName: training           # bound to the IPC_LOCK scc
          containers:
            - name: pytorch
              image: registry.example.com/ml/train:2.4
              env:
                - name: NCCL_DEBUG
                  value: INFO
                - name: NCCL_NET
                  value: IB                      # fail, don't fall back
                - name: NCCL_IB_HCA
                  value: mlx5                    # prefix match, skip the ethernet nics
              securityContext:
                capabilities:
                  add: ["IPC_LOCK"]
              resources:
                limits:
                  nvidia.com/gpu: 8
                  rdma/rdma_shared_device_a: 1   # name depends on your plugin config
              volumeMounts:
                - name: dshm
                  mountPath: /dev/shm
          volumes:
            - name: dshm
              emptyDir:
                medium: Memory
                sizeLimit: 32Gi
    Worker:
      replicas: 1
      restartPolicy: OnFailure
      template: *pod
```

YAML anchors so Master and Worker can't drift. They drifted on us before I did that.

## the preflight

Even with `NCCL_NET=IB`, the error you get is a NCCL stack trace that data scientists forward to the platform channel without reading. So the entrypoint runs a dumb check first that says what's wrong in English.

```python
#!/usr/bin/env python3
# preflight: is this pod actually able to use IB? run before torchrun.
import glob
import os
import resource
import sys

problems = []

if not glob.glob("/dev/infiniband/uverbs*"):
    problems.append("no /dev/infiniband/uverbs*. rdma resource not requested or plugin not running")

active = []
for port in glob.glob("/sys/class/infiniband/*/ports/*"):
    try:
        state = open(f"{port}/state").read().strip()        # e.g. "4: ACTIVE"
        layer = open(f"{port}/link_layer").read().strip()   # InfiniBand vs Ethernet
    except OSError:
        continue
    if "ACTIVE" in state and layer == "InfiniBand":
        active.append(port)
if not active:
    problems.append("no ACTIVE InfiniBand port visible in the pod")

soft, _ = resource.getrlimit(resource.RLIMIT_MEMLOCK)
if soft != resource.RLIM_INFINITY:
    problems.append(f"memlock is {soft} bytes, not unlimited. check IPC_LOCK / scc / ulimits")

shm = os.statvfs("/dev/shm")
if shm.f_blocks * shm.f_frsize < 1 << 30:
    problems.append("/dev/shm under 1GiB. mount a memory emptyDir")

for p in problems:
    print(f"PREFLIGHT FAIL {p}", file=sys.stderr)
if not problems:
    print(f"preflight ok: {len(active)} active IB ports")
sys.exit(1 if problems else 0)
```

Heads up on memlock. Some setups grant a big finite limit instead of unlimited and RDMA works fine. If that's you, compare against a number instead. I'd rather the check be strict and loosen it on purpose.

## then measure it once

A passing preflight means the pieces are there. It doesn't mean they're fast. Before handing a cluster to users, run `all_reduce_perf` from nccl-tests across two nodes and write the bus bandwidth down somewhere. Compare it to what the HCAs are rated for. If it's way off, it's usually GPU and HCA topology (GPU on one socket, NIC on the other) or a bad cable, and `nvidia-smi topo -m` is where you start.

Also, that number is your baseline. Next time someone says training got slower after an upgrade, you rerun it and know in ten minutes whether it's the fabric or their code.

NCCL's default is to keep going on whatever network it can find, and that's the wrong default when you paid for InfiniBand. Make it fail loud.
