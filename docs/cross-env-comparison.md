# Cross-Environment GPU Metrics Comparison

`gpu-metrics` collects GPU metrics with the same binary and output schema regardless of whether your workload runs on Kubernetes, SLURM, Flux, or bare metal. This lets you compare GPU performance across on-prem and cloud environments without post-processing.

## How it works

The `--env` flag selects the environment. The default is `auto`, which inspects process environment variables to detect the orchestrator:

| Value        | Detection signal                    |
|--------------|-------------------------------------|
| `auto`       | inspect env vars (default)          |
| `k8s`        | force Kubernetes                    |
| `slurm`      | force SLURM                         |
| `flux`       | force Flux                          |
| `standalone` | force bare-metal / no scheduler     |

Detection priority when `auto` is used: **SLURM → Flux → Kubernetes → standalone**.

## Unified JSON schema

Every environment emits the same top-level JSON structure so you can feed outputs from any environment into the same analysis pipeline:

```json
{
  "environment": {
    "orchestrator": "<k8s|slurm|flux|standalone>",
    "node": "<node or hostname>",
    "job_id": "<job id, if any>",
    "task_rank": 0,
    "pod_name": "<k8s only>",
    "namespace": "<k8s only>",
    "partition": "<slurm only>",
    "flux_uri": "<flux only>"
  },
  "collected_at": "2026-06-17T10:00:00Z",
  "devices": [
    {
      "Index": 0,
      "UUID": "GPU-aaaa-1111",
      "Name": "NVIDIA H100 SXM5 80GB",
      "GPUUtilization": 85,
      "MemoryUtilization": 70,
      "MemoryUsedMiB": 57344,
      "MemoryTotalMiB": 81920,
      "TemperatureCelsius": 72,
      "PowerDrawWatts": 650,
      "PowerLimitWatts": 700,
      "PCIeTxKBps": 4096,
      "PCIeRxKBps": 2048,
      "NVLinkTxMBps": 120000,
      "NVLinkRxMBps": 118000
    }
  ]
}
```

## Usage examples

### Kubernetes (auto-detected inside a pod)

```bash
# Downward API env vars supply node/pod/namespace automatically.
gpu-metrics --format json
```

```json
{
  "environment": {
    "orchestrator": "k8s",
    "node": "gpu-node-42",
    "pod_name": "train-job-0",
    "namespace": "ml-workloads"
  },
  "collected_at": "2026-06-17T10:00:00Z",
  "devices": [...]
}
```

Kubernetes Deployment snippet to expose Downward API fields:

```yaml
env:
  - name: NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  - name: POD_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
```

### SLURM

```bash
# Inside a SLURM job step; SLURM_JOB_ID, SLURM_STEP_GPUS, etc. are set by sbatch.
srun --gpus-per-task=1 gpu-metrics --format json
```

```json
{
  "environment": {
    "orchestrator": "slurm",
    "node": "compute-01",
    "job_id": "123456",
    "task_rank": 0,
    "partition": "gpu"
  },
  "collected_at": "2026-06-17T10:00:00Z",
  "devices": [...]
}
```

### Flux

```bash
flux run -n 4 --gpus-per-task=1 gpu-metrics --format json
```

```json
{
  "environment": {
    "orchestrator": "flux",
    "job_id": "f-abc123def456",
    "task_rank": 0,
    "flux_uri": "local:///run/flux/local"
  },
  "collected_at": "2026-06-17T10:00:00Z",
  "devices": [...]
}
```

### Standalone (bare metal / no scheduler)

```bash
gpu-metrics --env standalone --format json
```

```json
{
  "environment": {
    "orchestrator": "standalone",
    "node": "dev-workstation"
  },
  "collected_at": "2026-06-17T10:00:00Z",
  "devices": [...]
}
```

## Comparing across environments

Because the schema is identical, you can use standard tools to compare runs:

```bash
# Collect from on-prem SLURM job
srun gpu-metrics --format json > slurm-run.json

# Collect from Kubernetes pod (copy binary into pod or use DaemonSet)
kubectl exec -it train-pod-0 -- gpu-metrics --format json > k8s-run.json

# Compare GPU utilisation across environments using jq
jq -s '
  map({
    env: .environment.orchestrator,
    node: .environment.node,
    avg_util: (.devices | map(.GPUUtilization) | add / length)
  })
' slurm-run.json k8s-run.json
```

Example output:

```json
[
  { "env": "slurm",  "node": "compute-01",   "avg_util": 84 },
  { "env": "k8s",   "node": "gpu-node-42",  "avg_util": 71 }
]
```

## CSV output

CSV prepends four environment columns before the GPU columns:

```
orchestrator,node,job_id,task_rank,index,uuid,name,gpu_util_pct,...
slurm,compute-01,123456,0,0,GPU-aaaa-1111,NVIDIA H100,...
```

This makes it straightforward to import into pandas, DuckDB, or any spreadsheet tool and group by environment.

## Continuous collection for benchmarking

```bash
# Collect every 5 seconds during a training run
gpu-metrics --interval 5s --format json >> training-metrics.jsonl
```

Each line is a complete JSON document with an `environment` block, so the file self-describes where each sample was captured.
