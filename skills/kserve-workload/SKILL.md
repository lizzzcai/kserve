---
name: kserve-workload
description: "Generate KServe workload manifests: InferenceService, LLMInferenceService, ClusterServingRuntime, InferenceGraph. Use when deploying a model, creating a serving runtime, building a routing pipeline, or generating YAML for vLLM / HuggingFace / Triton / sklearn. Don't use for debugging a live deployment (use kserve-platform) or installing KServe itself (use kserve-infra)."
compatibility: Requires kubectl connected to a cluster with KServe installed.
allowed-tools: Bash(kubectl:*)
---

# KServe Workload

Generates correct, apply-ready KServe manifests grounded in the live cluster schema.

## Quick Reference

| | |
|---|---|
| API field docs | `kubectl explain <resource>.<field>` |
| Live CRD versions | `kubectl get crd inferenceservices.serving.kserve.io -o jsonpath='{.spec.versions[*].name}'` |
| Resource types & API docs | [references/resource-types.md](references/resource-types.md) |
| Runtime selection | [references/runtime-selection.md](references/runtime-selection.md) |
| Related skills | kserve-platform (debug), kserve-infra (install) |

## Rules

1. Run `kubectl explain` before generating any spec — never guess at field names or structure.
2. Always `--dry-run=client -o yaml` before asking to apply.
3. For `LLMInferenceService`: requires `Standard` (raw) deployment mode — Knative is not supported.
4. Prefer `autoSelect: true` runtimes over hardcoding a runtime name, unless the user specifies one.
5. Use `storageUri` for `InferenceService` and `model.uri` for `LLMInferenceService` — the field names differ.
6. Infer namespace from `kubectl config view --minify -o jsonpath='{..namespace}'`; default to `default`.

## Workflow

### 0. Gather context

Auto-discover what you can, then ask for what's missing before generating anything.

```bash
# Current namespace
kubectl config view --minify -o jsonpath='{..namespace}' 2>/dev/null || echo default
# Current context (sanity check with user)
kubectl config current-context
```

Required inputs — resolve from the user's prompt or ask:

| Input | Auto-discoverable? | Ask if missing |
|---|---|---|
| Resource name | No | Yes — "What should the InferenceService be named?" |
| Namespace | Yes (from kubeconfig context) | Only if context has no default namespace |
| Model type / framework | Usually in prompt | Yes — "What framework or model server? (vLLM, HuggingFace, sklearn, Triton, …)" |
| Storage URI | No | Yes — "Where is the model stored? (s3://, gs://, pvc://, oci://…)" |
| GPU / CPU / memory | No | Ask — "What compute resources are available? Any GPU?" |

Do not generate a manifest until name and storageUri are known.

### 1. Discover resource type

| User wants | API | Group/Version |
|---|---|---|
| Standard model serving | `InferenceService` | `serving.kserve.io/v1beta1` |
| LLM with disaggregated serving / LoRA / KEDA | `LLMInferenceService` | `serving.kserve.io/v1alpha2` |
| Routing pipeline (ensemble, A/B, canary) | `InferenceGraph` | `serving.kserve.io/v1alpha1` |
| Custom container spec for a model server | `ClusterServingRuntime` | `serving.kserve.io/v1alpha1` |

Unsure which to use? Load [references/resource-types.md](references/resource-types.md).

### 2. Check schema

```bash
kubectl explain inferenceservice.spec.predictor --recursive
kubectl explain llminferenceservice.spec --recursive
```

### 3. Identify runtime

For `InferenceService`, list available runtimes:

```bash
kubectl get clusterservingruntimes -o custom-columns='NAME:.metadata.name,FORMATS:.spec.supportedModelFormats[*].name'
```

Runtime selection guide:

| Model type | Default runtime | modelFormat name |
|---|---|---|
| LLM (GPU) | `kserve-vllmserver` | `vLLM` |
| LLM (CPU/small) | `kserve-huggingfaceserver` | `huggingface` |
| Triton (multi-framework) | `kserve-tritonserver` | `triton` |
| sklearn / joblib | `kserve-sklearnserver` | `sklearn` |
| XGBoost | `kserve-xgbserver` | `xgboost` |

Unsure? Load [references/runtime-selection.md](references/runtime-selection.md).

### 4. Generate manifest

Get live spec examples from the source runtimes:

```bash
kubectl get clusterservingruntime kserve-vllmserver -o yaml   # vLLM reference
kubectl get clusterservingruntime kserve-sklearnserver -o yaml # sklearn reference
```

Then produce a manifest using the user's storageUri, resource requests, and any custom args.

### 5. Validate and apply

```bash
kubectl apply --dry-run=client -o yaml -f manifest.yaml
# If dry-run passes:
kubectl apply -f manifest.yaml
kubectl get inferenceservice <name> -w
```

## Gotchas

- `LLMInferenceService` requires `serving.kserve.io/deploymentMode: Standard` annotation or cluster default. Fails silently if Knative is the cluster default.
- `storageUri` scheme matters: `gs://`, `s3://`, `pvc://`, `oci://` each require different credentials/config. The storage-initializer must support the scheme.
- `InferenceGraph` `root` node is mandatory — the graph fails to deploy if `nodes.root` is absent.
- For `Splitter`, the weights of all steps must sum to 100.
- `ClusterServingRuntime` is cluster-scoped; `ServingRuntime` is namespace-scoped. Prefer cluster-scoped for shared runtimes.
- The vLLM runtime rewrites the image to the `-cpu` variant automatically when no GPU resource is requested — this is controller behavior, not something to set manually.

## Progressive Disclosure

| Need | Load |
|---|---|
| Which API to use | [references/resource-types.md](references/resource-types.md) |
| Runtime / model server selection | [references/runtime-selection.md](references/runtime-selection.md) |
