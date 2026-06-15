# FAQ

## Does this replace dcgm-exporter?

For autoscaling, yes. If you only use dcgm-exporter to feed metrics into KEDA for scaling decisions, keda-gpu-scaler replaces the entire dcgm-exporter → Prometheus → PromQL pipeline.

If you also use dcgm-exporter for Grafana dashboards or alerting, keep it running alongside keda-gpu-scaler. They both read NVML independently and don't conflict.

## Does this require NVIDIA drivers on the host?

Yes. NVML reads GPU state through `libnvidia-ml.so`, which is installed with the NVIDIA driver. If you're running GPU workloads on Kubernetes, you already have this.

The binaries link NVML at runtime, so they **will not start on a machine that lacks `libnvidia-ml.so`** — for example, a laptop or CI runner without an NVIDIA driver. In that case the process exits immediately with an `nvml init failed` / `failed to initialize NVML` error. To develop without a GPU, use the mock collector (see below) rather than running the binary directly.

## Can I run this without a GPU for development?

Yes. The project has a mock GPU collector (`pkg/gpu/mock.go`) used by all unit and e2e tests. You can build, test, and develop the full gRPC path without GPU hardware.

## Why not use the KEDA Prometheus scaler with dcgm-exporter?

It works, but you're adding 3 extra components to the scaling path (dcgm-exporter, Prometheus, and PromQL queries), 15-30 seconds of scrape delay, and PromQL queries that break when DCGM changes metric names between versions. See the [migration guide](MIGRATION.md) for details.

## Does this support multi-GPU nodes?

Yes. The `aggregation` parameter controls how per-GPU metrics are combined: `max` (default), `avg`, `min`, or `sum`. You can also target a specific GPU with `gpuIndex`.

## Does this support scale-to-zero?

Yes. Set `activationThreshold` in the ScaledObject metadata, or use a profile that includes one (e.g., `vllm-inference` sets activation at 5%). When all GPU metrics drop below the activation threshold, KEDA scales the deployment to zero.

## What Kubernetes distributions are supported?

Any distribution with NVIDIA GPU nodes and KEDA v2.10+. Tested on:
- Oracle OKE
- Amazon EKS
- Google GKE
- Azure AKS
- Vanilla kubeadm clusters

## Does this work with MIG (Multi-Instance GPU)?

Not yet. MIG support is on the [roadmap](../README.md#roadmap). Each MIG instance exposes separate NVML handles, so the scaler would need to enumerate partitions within a physical GPU.

## What happens if the scaler pod crashes?

KEDA falls back to the last known metric value for a short grace period, then stops scaling (safe default). The DaemonSet controller restarts the scaler pod automatically. Since each node runs its own scaler instance, a crash on one node doesn't affect scaling on other nodes.

## How is this different from the Kubernetes Custom Metrics API?

The Custom Metrics API requires you to build and maintain a custom metrics server. keda-gpu-scaler is a pre-built solution that plugs directly into KEDA's external scaler interface — no custom code needed.
