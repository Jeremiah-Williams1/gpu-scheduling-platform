# GPU Scheduling & Sharing on Kubernetes

Hands-on exploration of how Kubernetes shares and schedules GPUs across
workloads — what actually works, what breaks, and why the "no isolation"
warning in the docs means something different depending on the mechanism.

Built on a single T4 GPU (EC2 → minikube), as part of a broader ML
platform engineering project series. Full write-up: **[link to Medium
article once published]**

## What this covers

| Mechanism | Status | Key finding |
|---|---|---|
| Time-slicing | ✅ Hands-on | Shared VRAM pool, isolated failure — one pod OOM'd under contention, the other two kept running |
| MPS | ✅ Hands-on | Shared VRAM pool **and** shared execution context — one pod's OOM crashed all three |
| MIG | 📖 Conceptual | T4 (Turing) doesn't support MIG — needs Ampere+ (A100/H100). Planned for a follow-up on RunPod |
| DRA | 📖 Conceptual | Newer K8s-native resource allocation plumbing, GA in 1.34 — not itself a sharing mechanism |
| Kueue (quota, preemption, gang scheduling) | ✅ Hands-on | See below |

## Why this exists

Kubernetes treats a GPU as an indivisible unit by default — one pod, one
GPU, regardless of actual usage. That's expensive and wasteful once more
than one GPU workload exists. This repo is the hands-on half of
understanding the mechanisms that fix that: **sharing** one physical GPU
across workloads (time-slicing, MPS, MIG), and **queueing/admission**
across a GPU pool (Kueue) — two different layers of the same underlying
problem.

## Repo structure

```
manifests/
├── device-plugin/       # NVIDIA device plugin sharing configs
│   ├── timeslicing-config.yaml
│   └── mps-config.yaml
├── kueue/                # Queueing, quota, and priority objects
│   ├── queues.yaml            # ResourceFlavor, ClusterQueues, LocalQueues
│   ├── priority-classes.yaml  # WorkloadPriorityClasses
│   ├── filler-job.yaml        # Occupies partial quota, for the gang-scheduling test
│   └── gang-job.yaml          # 2-pod indexed Job, tests all-or-nothing admission
└── test-workloads/       # Load-generating pods used to observe real contention
    └── gpu-load-test-pods.yaml
```

## Setup (what this assumes)

- A node with an NVIDIA GPU (tested on a T4) and drivers installed
- minikube with `--driver=docker --gpus=all`
- Helm 3
- [NVIDIA device plugin](https://github.com/NVIDIA/k8s-device-plugin) (Helm chart, `v0.20.0`+ recommended — see note below)
- [Kueue](https://kueue.sigs.k8s.io/) `v0.19.4`

> **Version note:** the device plugin chart's MPS support had real bugs on
> `v0.14.5` (missing control daemonset — see write-up for the full
> debugging trail). `v0.20.0` works correctly. Don't pin to an old version
> just because time-slicing worked on it.

### 1. GPU sharing (pick one config, they're mutually exclusive)

```bash
helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update

# time-slicing
helm install nvdp nvdp/nvidia-device-plugin \
  --version=0.20.0 --namespace kube-system \
  --set-file config.map.config=./manifests/device-plugin/timeslicing-config.yaml \
  --set config.default=config

# OR mps — also needs these node labels (normally set by GPU Feature Discovery,
# not deployed here) and the GPU's compute mode set to EXCLUSIVE_PROCESS
kubectl label node <node-name> nvidia.com/gpu.present=true nvidia.com/mps.capable=true
sudo nvidia-smi -c EXCLUSIVE_PROCESS
helm install nvdp nvdp/nvidia-device-plugin \
  --version=0.20.0 --namespace kube-system \
  --set-file config.map.config=./manifests/device-plugin/mps-config.yaml \
  --set config.default=config --set mps.enabled=true
```

Verify: `kubectl describe node <node-name> | grep -A15 "^Capacity:"` should
show `nvidia.com/gpu: 4` (both configs advertise the same replica count
here — the difference is in how contention behaves, not the count).

### 2. Kueue

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/kueue/releases/download/v0.19.4/manifests.yaml
kubectl wait deploy/kueue-controller-manager -n kueue-system --for=condition=available --timeout=5m

kubectl create namespace team-a
kubectl create namespace team-b

kubectl apply -f manifests/kueue/queues.yaml
kubectl apply -f manifests/kueue/priority-classes.yaml
```

## Reproducing the key results

**Contention under time-slicing / MPS:**
```bash
kubectl apply -f manifests/test-workloads/gpu-load-test-pods.yaml
kubectl logs gpu-load-test-1   # watch for CUDA OOM, compare behavior across the two sharing configs
```

**Hard quota enforcement + gang scheduling:**
```bash
kubectl apply -f manifests/kueue/filler-job.yaml
kubectl apply -f manifests/kueue/gang-job.yaml
kubectl get pods -n team-a -w   # expect zero gang-job pods until filler-job completes,
                                  # then both gang-job pods appear together
```

**Preemption policy comparison** — toggle live and observe:
```bash
kubectl patch clusterqueue training-cq --type=merge \
  -p '{"spec":{"preemption":{"withinClusterQueue":"LowerPriority"}}}'
kubectl get workloads -n team-a -w
```

## Honest scope notes

- MIG and DRA are covered conceptually only — T4 hardware can't run MIG,
  and DRA is architecturally a different kind of thing (resource-request
  plumbing, not a sharing mechanism). Both get a real hands-on pass on an
  A100 in a planned follow-up.
- The MPS setup here involved a real, multi-layered debugging process
  (duplicate leftover daemonset, a stale config-reload bug, a chart
  version that silently never created its own control daemonset, and
  missing node labels normally provided by GPU Feature Discovery). That
  trail is documented in full in the write-up, not smoothed over here.
