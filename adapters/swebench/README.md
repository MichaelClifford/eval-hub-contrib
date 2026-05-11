# SWE-bench Adapter

Evaluates LLM-generated code patches against the [SWE-bench](https://www.swebench.com/) benchmark using Kubernetes Jobs.

## How It Works

The adapter orchestrates evaluation by creating K8s Jobs for each SWE-bench instance:

1. Receives a predictions JSON file (standard SWE-bench format)
2. Creates one K8s Job per instance using pre-built SWE-bench container images
3. Each Job applies the model's patch, runs the test suite, and exits
4. The adapter grades results using SWE-bench's grading infrastructure
5. Reports `resolve_rate` (% of instances where the patch passes all tests) as the primary metric

## Prerequisites

- **Kubernetes cluster** with permissions to create/delete Jobs
- **Pre-built SWE-bench images** in a container registry accessible from the cluster
  - Default: `docker.io/swebench/sweb.eval.x86_64.<instance_id>:latest`
  - Configurable via `k8s_registry` parameter for private registries

## Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `predictions_path` | string | required | Path to predictions JSON, or `"gold"` for ground-truth patches |
| `k8s_registry` | string | `docker.io/swebench` | Container registry prefix for SWE-bench images |
| `k8s_namespace` | string | `swe-bench` | Kubernetes namespace for evaluation Jobs |
| `max_workers` | integer | `10` | Maximum concurrent evaluation Jobs |
| `timeout_per_instance` | integer | `1800` | Per-instance timeout in seconds |
| `split` | string | `test` | Dataset split (`test` or `dev`) |
| `instance_ids` | array | `null` | Specific instance IDs to evaluate (all if null) |

## Supported Benchmarks

| Benchmark ID | Dataset | Instances |
|-------------|---------|-----------|
| `swebench_verified` | SWE-bench Verified | 500 |
| `swebench_lite` | SWE-bench Lite | 300 |
| `swebench_full` | SWE-bench Full | 2294 |

## Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `resolve_rate` | accuracy | Fraction of instances where the patch resolves the issue (primary metric) |
| `resolved_count` | count | Number of resolved instances |
| `completed_count` | count | Number of instances that completed evaluation |
| `total_instances` | count | Total instances evaluated |
| `error_count` | count | Instances that errored during evaluation |

## Predictions Format

The adapter accepts the standard SWE-bench predictions format:

```json
{
  "instance_id_1": {
    "instance_id": "django__django-11099",
    "model_patch": "diff --git a/...",
    "model_name_or_path": "my-model"
  },
  "instance_id_2": { ... }
}
```

## RBAC Requirements

Unlike other eval-hub adapters which run evaluations inside a single pod,
the SWE-bench adapter creates **one K8s Job per evaluation instance** (each
instance needs its own isolated container with a different codebase and
dependencies).  This means the adapter pod's service account needs
permissions to manage Jobs -- a requirement unique to this adapter.

```yaml
rules:
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["create", "get", "list", "delete"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list"]
```

On eval-hub with the operator, bind this to the job service account:

```bash
oc create rolebinding swebench-eval-jobs \
  --role=swebench-eval-jobs \
  --serviceaccount=<namespace>:evalhub-<namespace>-job \
  -n <namespace>
```

## Local Development

```bash
# Run tests
cd adapters/swebench
pip install -r requirements.txt -r requirements-test.txt
pytest tests/ -v

# Build container
make image-swebench

# Test with gold predictions (requires K8s cluster)
EVALHUB_JOB_SPEC_PATH=adapters/swebench/meta/job.json \
  python adapters/swebench/main.py
```
