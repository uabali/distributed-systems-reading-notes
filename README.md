# distributed-systems-reading-notes

**Lab 5/5** of the Distributed Systems series for a fal-style Software Engineer (Distributed Systems) path.

Short reading notes in **my own words** — not a book dump, not long quotes. Goal: turn theory into interview / design vocabulary and connect it to the hands-on labs.

> **TR (kısa):** Bu repo, Designing Data-Intensive Applications (DDIA) temalarını ve Kubernetes / Ray GPU scheduling’i kendi cümlelerimle özetler. Telif: kitaptan uzun alıntı yok. Lab serisiyle bağ kurar (queue, CDN, inference, Linux).

## How to read

1. Start with [`notes/01-ddia.md`](notes/01-ddia.md) — one theme at a time (`idea → why it matters → fal / DS link`).
2. Then [`notes/02-k8s-ray-gpu-scheduling.md`](notes/02-k8s-ray-gpu-scheduling.md) — how GPUs actually get onto pods and into Ray workers.
3. Cross-check the primary sources listed below; do not treat these notes as a substitute for the book or the docs.

## Lab series (siblings)

| # | Repo | Focus |
|---|------|--------|
| 1 | [rust-queue-worker](https://github.com/uabali/rust-queue-worker) | Tokio queue + worker pool + retry/backoff |
| 2 | [mini-CDN](https://github.com/uabali/mini-CDN) | Consistent hashing, partial failure, load shedding, Prometheus/Grafana |
| 3 | [inference-worker-lab](https://github.com/uabali/inference-worker-lab) | Redis Streams + mock/vLLM worker + lag autoscaler |
| 4 | [linux-networking-lab](https://github.com/uabali/linux-networking-lab) | namespaces, cgroups, iptables/nftables, eBPF demos |
| 5 | **this repo** | DDIA themes + K8s/Ray GPU scheduling notes |

## Primary sources (verify yourself)

### DDIA

- Book: Martin Kleppmann, *Designing Data-Intensive Applications* (O’Reilly). Notes here are **paraphrase + original examples only** — no long excerpts, no invented page numbers.
- Author site / related material: https://dataintensive.net/

### Kubernetes GPU scheduling

- [Schedule GPUs | Kubernetes](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [NVIDIA k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA DRA Driver for GPUs (GPU Operator)](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/dra-intro-install.html) — newer path beyond classic device plugins

### Ray on Kubernetes (GPU + two-layer scheduling)

- [Using GPUs (KubeRay)](https://docs.ray.io/en/latest/cluster/kubernetes/user-guides/gpu.html)
- [Scheduling on Kubernetes (KubeRay)](https://docs.ray.io/en/latest/cluster/kubernetes/user-guides/scheduling.html)
- [Ray Serve resource allocation](https://docs.ray.io/en/latest/serve/resource-allocation.html)

## Repo layout

```
notes/
  01-ddia.md                      # DDIA themes in own words
  02-k8s-ray-gpu-scheduling.md    # K8s device plugin + Ray dual scheduler
README.md
LICENSE
```

## License

MIT — learning notes. Read the originals for authoritative wording.
