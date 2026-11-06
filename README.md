# distributed-systems-reading-notes

Short reading notes in own words on Designing Data-Intensive Applications themes and Kubernetes / Ray GPU scheduling.

No long book excerpts. Cross-check primary sources below.

## Notes

1. [`notes/01-ddia.md`](notes/01-ddia.md) — DDIA themes
2. [`notes/02-k8s-ray-gpu-scheduling.md`](notes/02-k8s-ray-gpu-scheduling.md) — K8s device plugin + Ray dual scheduler

## Related repos

| Repo | Focus |
|------|--------|
| [rust-queue-worker](https://github.com/uabali/rust-queue-worker) | Tokio queue + worker pool + retry/backoff |
| [mini-CDN](https://github.com/uabali/mini-CDN) | Consistent hashing, partial failure, load shedding |
| [inference-worker-lab](https://github.com/uabali/inference-worker-lab) | Redis Streams + mock/vLLM + lag autoscaler |
| [linux-networking-lab](https://github.com/uabali/linux-networking-lab) | namespaces, cgroups, iptables/nftables, eBPF |

## Primary sources

### DDIA

- Martin Kleppmann, *Designing Data-Intensive Applications* (O’Reilly) — paraphrase only here
- https://dataintensive.net/

### Kubernetes GPU scheduling

- [Schedule GPUs | Kubernetes](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [NVIDIA k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA DRA Driver for GPUs](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/dra-intro-install.html)

### Ray on Kubernetes

- [Using GPUs (KubeRay)](https://docs.ray.io/en/latest/cluster/kubernetes/user-guides/gpu.html)
- [Scheduling on Kubernetes (KubeRay)](https://docs.ray.io/en/latest/cluster/kubernetes/user-guides/scheduling.html)
- [Ray Serve resource allocation](https://docs.ray.io/en/latest/serve/resource-allocation.html)

## Layout

```
notes/
  01-ddia.md
  02-k8s-ray-gpu-scheduling.md
README.md
LICENSE
```

## License

MIT
