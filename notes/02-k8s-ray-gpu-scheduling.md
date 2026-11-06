# Kubernetes + Ray GPU scheduling — notes in my own words

Primary sources (read these, not just this file):

1. [Schedule GPUs | Kubernetes](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
2. [NVIDIA k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin)
3. [Using GPUs (KubeRay)](https://docs.ray.io/en/latest/cluster/kubernetes/user-guides/gpu.html)
4. [Scheduling on Kubernetes (KubeRay)](https://docs.ray.io/en/latest/cluster/kubernetes/user-guides/scheduling.html)
5. [Ray Serve — Resource allocation](https://docs.ray.io/en/latest/serve/resource-allocation.html)
6. (Emerging) [NVIDIA DRA Driver for GPUs](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/dra-intro-install.html)

---

## The problem

GPUs are scarce, non-fungible (A100 ≠ L4), and expensive when idle.  
Kubernetes must **discover** devices on nodes, **advertise** them as schedulable capacity, and **bind** pods that asked for them — without letting a random CPU pod steal a GPU-shaped machine forever.

For ML serving (vLLM, Ray Serve, custom workers) there is a second problem: even after a pod has a GPU, an **application scheduler** (Ray) must place *tasks/actors/replicas* onto those pods. Two layers. Two failure modes.

---

## How classic Kubernetes GPU scheduling works

**Idea.** Vendor **device plugins** (DaemonSets) talk to the kubelet. For NVIDIA, the common resource name is `nvidia.com/gpu`. Drivers live on the node; the plugin exposes a count of healthy GPUs as an extended resource.

**Request shape (important quirk).** Custom devices like GPUs are meant to be listed under `limits`. Kubernetes treats the limit as the request if you omit requests; if you set both, they must be equal. You cannot request a GPU without a limit. That is unlike CPU, where request < limit is normal for oversubscription.

**Heterogeneous clusters.** Label nodes by accelerator type / memory and use nodeSelector or affinity so a pod that needs 80 GB HBM does not land on a 24 GB card. Node Feature Discovery (NFD) and vendor labelers automate much of this.

**Taints.** GPU nodes are often tainted so only pods that tolerate `nvidia.com/gpu` (or that request the resource, with ExtendedResourceToleration on managed clouds) may land there. This protects expensive nodes from being filled by unrelated CPU workloads.

**Tradeoffs of the device-plugin model.**

| Upside | Downside |
|--------|----------|
| Simple: `limits: nvidia.com/gpu: 1` | Whole-GPU allocation by default; sharing/MIG needs extra machinery |
| Works across clouds with the same mental model | Health checking / cleanup historically imperfect (see plugin README caveats) |
| Stable K8s path since v1.26 for GPU management via plugins | Newer DRA path is more flexible but needs newer K8s + CDI + operator versions |

**DRA (Dynamic Resource Allocation)** — briefly. NVIDIA’s DRA driver moves allocation toward `ResourceClaim` / device classes (`gpu.nvidia.com`, MIG, VFIO) with richer sharing and reconfiguration. Treat it as the next-gen path; most tutorials and Ray docs still center on `nvidia.com/gpu` extended resources. Do not mix classic device plugin and DRA allocation on the same nodes without reading the operator docs (they conflict if both try to own GPUs).

---

## Ray on Kubernetes: two schedulers stacked

KubeRay runs each Ray head/worker as a **Pod**. Then:

1. **Kubernetes** places the Pod on a machine (CPU/mem/GPU requests, selectors, taints, quotas).
2. **Ray** places tasks/actors onto Ray nodes (those pods), reading `@ray.remote(num_gpus=…)` and related options.

Ray **cannot invent** GPUs Kubernetes did not grant. If K8s never scheduled a GPU worker pod, Ray’s GPU actors stay pending forever — even if your Python looks fine.

### Capacity agreement (where people get burned)

KubeRay advertises container **GPU limits** from the **first** container in the pod to Ray (`--num-gpus`). Per the scheduling guide:

- Sidecar limits do not count.
- Memory/GPU **requests** are ignored for Ray’s logical capacity — set requests equal to limits for GPUs so both layers agree.
- CPU millicores are rounded **up** to whole logical CPUs for Ray.

Symptom dictionary:

| What you see | Likely layer |
|--------------|--------------|
| Pod `Pending` | Kubernetes (capacity, taint, selector, quota, autoscaler) |
| Pods Running, Ray tasks/actors pending | Ray (infeasible resources, placement groups, wrong logical capacity) |

### GPU worker group pattern (KubeRay)

Typical pattern from the GPU guide:

- Worker group with `limits.nvidia.com/gpu: 1` (and matching request).
- `minReplicas: 0`, `maxReplicas: N`, in-tree autoscaling on — GPUs are costly; scale from zero.
- GPU image (`rayproject/ray:…-gpu`).
- Workloads: `@ray.remote(num_gpus=1)` or Serve `ray_actor_options={"num_gpus": 1}` (fractional GPUs possible for co-located small models).

**Multi-tenancy footgun.** Pods that do **not** request `nvidia.com/gpu` may still see all devices via `NVIDIA_VISIBLE_DEVICES=all` unless you set it to `void` (documented NVIDIA/KubeRay issue). Never assume “no GPU request” means “no GPU visibility.”

---

## Tradeoffs that matter for inference products

1. **Bin-packing vs isolation.** Fractional `num_gpus` or MIG increases utilization; noisy neighbors and memory fragmentation increase blast radius. Product SLOs usually prefer isolation for large LLMs, packing for tiny encoders.
2. **Spread vs pack (Ray Serve).** Default spread helps HA; pack scheduling reduces idle nodes (cost) but concentrates risk. Explicit `placement_group_bundles` + `STRICT_PACK` matter for tensor-parallel vLLM across multiple GPUs.
3. **Two autoscalers.** Ray autoscaler asks for more worker pods based on **logical** demand; cluster autoscaler then adds nodes. Stall can be either layer. Lag-based app autoscaling (as in the lab) is yet a third signal — useful, but reconcile it with GPU cold-start time.
4. **Head node hygiene.** Often set head `num-cpus: "0"` in `rayStartParams` so CPU/GPU work stays on workers. Head still consumes real K8s CPU; you only changed Ray’s view.

---

## Bridge to `inference-worker-lab`

The lab simulates what production still needs after you leave the laptop:

| Lab concept | Production K8s/Ray analogue |
|-------------|-----------------------------|
| Redis Streams queue + consumer group | Decouple API from GPU workers; absorb spikes |
| MOCK vs VLLM modes | CPU CI vs GPU pool |
| Lag → desired replicas | HPA / KEDA / Ray autoscaler / custom controller on queue depth |
| Workers pulling jobs | Ray Serve replicas or dedicated GPU Deployment with `nvidia.com/gpu` |

What the lab deliberately **does not** solve (homework for the notes):

- Binding a replica to a **specific GPU type** (node labels + Serve `label_selector`).
- Preventing CPU pods from landing on GPU nodes (taints/tolerations).
- Multi-GPU tensor parallel placement (placement groups).
- Agreeing K8s limits with Ray logical GPUs (KubeRay first-container rule).

Mental model to keep: **queue lag tells you *how many* workers you want; the device plugin + scheduler tell you *whether those workers can actually get GPUs*.** Optimizing only one side creates either idle GPUs or Pending pods.

---

## Minimal checklist before blaming “the model”

1. Does the node show allocatable `nvidia.com/gpu`? (`kubectl describe node`)
2. Does the pod request GPU in **limits** (and matching request)?
3. Is the pod Pending (K8s) or Running with pending actors (Ray)?
4. Is the GPU limit on the **first** container KubeRay reads?
5. Are non-GPU pods accidentally seeing devices (`NVIDIA_VISIBLE_DEVICES`)?

If those five are clean, then look at vLLM args, weights, and batching — not before.
