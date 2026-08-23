# KServe Install Modes

## Standard (Raw) Mode

No Knative required. KServe creates plain Kubernetes `Deployment` + `Service` objects.

Full setup guide: https://kserve.github.io/website/docs/admin-guide/kubernetes-deployment

**When to use:**
- `LLMInferenceService` (required — Knative not supported)
- Clusters where Knative is not available or not desired
- Simpler operational model preferred

**Set as cluster default:**
```bash
kubectl edit configmap inferenceservice-config -n kserve
# Set: "defaultDeploymentMode": "Standard"
```

## Knative (Serverless) Mode

KServe creates `Knative Service` objects. Knative Serving handles pod lifecycle and routing.

Full setup guide: https://kserve.github.io/website/docs/admin-guide/serverless

**Benefits:** Scale-to-zero, traffic splitting across revisions, built-in gradual rollout.

**Limitations:**
- `LLMInferenceService` does NOT work in Knative mode
- Cold-start latency on first request after scale-to-zero
- Additional operational complexity from Knative + Istio stack

## Coexistence

All three modes can be active in the same cluster. The `inferenceservice-config` ConfigMap sets the cluster default; individual resources override via annotation:

```yaml
metadata:
  annotations:
    serving.kserve.io/deploymentMode: Standard  # or Knative
```

Check current cluster default:
```bash
kubectl get configmap inferenceservice-config -n kserve \
  -o jsonpath='{.data.deploy}' | python3 -m json.tool | grep deploymentMode
```
