# Getting Started

## Prerequisites

- Kubernetes cluster with NVIDIA GPU worker nodes (OKE, GKE, EKS, AKS, or bare metal)
- [KEDA v2.10+](https://keda.sh/docs/latest/deploy/) installed
- NVIDIA GPU drivers and [Device Plugin](https://github.com/NVIDIA/k8s-device-plugin) installed

> **Note:** The scaler links NVML and loads `libnvidia-ml.so` at runtime. The binary (and the container image) will not start on a host that lacks the NVIDIA driver. On GPU nodes the driver supplies this library; for local development without a GPU, use the mock collector exercised by the test suite instead of running the binary directly.

## Deploy the Scaler

Make sure KEDA is already running, then deploy:

```bash
kubectl apply -f deploy/manifests.yaml
```

Or use Helm:

```bash
helm install keda-gpu-scaler deploy/helm/keda-gpu-scaler \
  --namespace keda \
  --set nodeSelector."nvidia\.com/gpu\.present"=true
```

This puts a pod on every GPU node, polling NVML and serving metrics over gRPC on port 6000.

## Attach to Your Workload

Point a ScaledObject at the GPU scaler:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: vllm-inference-scaler
  namespace: ai-workloads
spec:
  scaleTargetRef:
    name: vllm-deepseek-deployment
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
    - type: external
      metadata:
        scalerAddress: "keda-gpu-scaler.keda.svc.cluster.local:6000"
        profile: "vllm-inference"
```

KEDA will now scale your deployment based on GPU utilization and VRAM pressure.

## Verify It Works

```bash
# Check scaler pods are running on GPU nodes
kubectl get pods -n keda -l app=keda-gpu-scaler -o wide

# Check KEDA sees the ScaledObject
kubectl get scaledobject -A

# Watch HPA in action
kubectl get hpa -w
```

## What to read next

- [Configuration Reference](configuration.md) for profiles, aggregation, and all parameters
- [Architecture](DESIGN.md) if you want to understand the design
- [Migration Guide](MIGRATION.md) if you're replacing dcgm-exporter + Prometheus
