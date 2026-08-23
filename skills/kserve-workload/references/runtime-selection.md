# Runtime Selection

Official docs: https://kserve.github.io/website/docs/model-serving/predictive-inference/frameworks/overview

Always confirm available runtimes on the live cluster first:

```bash
kubectl get clusterservingruntimes \
  -o custom-columns='NAME:.metadata.name,FORMATS:.spec.supportedModelFormats[*].name,AUTO:.spec.supportedModelFormats[*].autoSelect'
```

## Decision Guide

**LLM — GPU available?**
- Yes → `kserve-vllmserver` (OpenAI-compatible, best throughput)
- No / CPU only → `kserve-huggingfaceserver`

**Multi-framework (TensorRT, ONNX, TensorFlow, PyTorch in one cluster)?**
- `kserve-tritonserver` — supports multiple backends in one runtime

**Traditional ML (scikit-learn, XGBoost, LightGBM)?**
- sklearn → `kserve-sklearnserver`
- XGBoost → `kserve-xgbserver`
- LightGBM → `kserve-lgbserver`

**PMML or ONNX?**
- PMML → `kserve-pmmlserver`
- ONNX → `kserve-tritonserver` or custom runtime

**Multinode LLM (tensor parallelism across nodes)?**
- Use `kserve-huggingfaceserver-multinode` — check cluster support:
  ```bash
  kubectl get clusterservingruntime kserve-huggingfaceserver-multinode
  ```

## Custom Runtime

When no built-in runtime fits, create a `ClusterServingRuntime`:

```bash
kubectl explain clusterservingruntime.spec.containers --recursive
```

Reference an existing runtime for the template:

```bash
kubectl get clusterservingruntime kserve-sklearnserver -o yaml
```
