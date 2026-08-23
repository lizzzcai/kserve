# KServe Resource Types

Official docs: https://kserve.github.io/website/docs/concepts/resources

## InferenceService (v1beta1)

The standard API for deploying a single model. Supports predictor + optional transformer + optional explainer.

```bash
kubectl explain inferenceservice.spec --recursive
```

Key fields:
- `spec.predictor.model.modelFormat.name` — matches a `ClusterServingRuntime` supportedModelFormat
- `spec.predictor.model.storageUri` — model artifact location
- `spec.predictor.minReplicas` / `maxReplicas` — scaling bounds
- `spec.predictor.model.resources` — CPU/memory/GPU requests and limits
- `spec.transformer` — optional pre/post-processing sidecar
- `spec.canary[]` — named canary predictors for progressive rollout

Deployment modes (annotation `serving.kserve.io/deploymentMode`):
- `Knative` — serverless, scale-to-zero, requires Knative Serving
- `Standard` — raw Kubernetes Deployment + Service, no Knative dependency

## LLMInferenceService (v1alpha2)

Purpose-built for LLMs. Adds:
- Disaggregated prefill/decode (`spec.prefill`)
- LoRA adapter management (`spec.model.lora.adapters`)
- KEDA autoscaling out of the box
- Gateway API routing (`spec.router`)
- BaseRef inheritance from `LLMInferenceServiceConfig`

**Requires Standard (raw) deployment mode.** Does not support Knative.

```bash
kubectl explain llminferenceservice.spec --recursive
```

## InferenceGraph (v1alpha1)

Official docs: https://kserve.github.io/website/docs/concepts/resources/inferencegraph

Routes requests across multiple InferenceServices in a pipeline.

Router types:
- `Sequence` — chain steps, passing output to next input
- `Splitter` — weighted random routing (weights must sum to 100)
- `Ensemble` — fan-out to all steps, merge responses
- `Switch` — condition-based routing

Must have a `root` node. Steps reference `InferenceService` by URL or soft target.

```bash
kubectl explain inferencegraph.spec --recursive
```

## ClusterServingRuntime / ServingRuntime (v1alpha1)

Official docs: https://kserve.github.io/website/docs/concepts/resources/servingruntime

Defines a container template for a model server. The controller injects model-specific args (model path, name) at pod creation time.

- `ClusterServingRuntime` — cluster-scoped, available to all namespaces
- `ServingRuntime` — namespace-scoped override

Built-in runtimes live in `config/runtimes/` in the kserve/kserve repo and are installed at cluster setup time.

```bash
kubectl get clusterservingruntimes
kubectl explain clusterservingruntime.spec --recursive
```
