# KServe Failure Patterns

Debugging guide: https://kserve.github.io/website/docs/developer-guide/debugging
Storage credential setup: https://kserve.github.io/website/docs/model-serving/storage/overview

## Pattern: Model never becomes Ready — predictor pod Pending

Symptoms: `kubectl get pod` shows `Pending`; Events show `Insufficient nvidia.com/gpu` or `no nodes match pod affinity`.

Fix:
- Confirm node has the GPU resource and is not exclusively tainted: `kubectl get nodes -o json | jq '.items[].status.allocatable'`
- Reduce resource request or add a node selector matching an available GPU node.

## Pattern: Init:CrashLoopBackOff — storage-initializer fails

Symptoms: `kubectl logs <pod> -c storage-initializer` shows credential or scheme errors.

Common causes:
- `s3://` URI but no `awsAccessKeyID` / `awsSecretAccessKey` secret in the namespace
- `gs://` URI but no Workload Identity or `gcloud-application-credentials-secret` annotation
- `pvc://` URI but PVC does not exist or is in wrong namespace
- URI typo — storage-initializer logs the exact path it attempted

Fix: match the secret type to the URI scheme. See: https://kserve.github.io/website/docs/model-serving/storage/overview

## Pattern: CrashLoopBackOff — model server exits immediately

Symptoms: `kubectl logs <pod> -c kserve-container --previous` shows OOM or startup error.

Common causes:
- Model too large for memory limit → increase `resources.limits.memory`
- Wrong model path (model not at `/mnt/models`) → verify storage-initializer completed first
- vLLM CUDA OOM → reduce `--max-model-len` or `--gpu-memory-utilization` via `args` override in the InferenceService

## Pattern: Ready: True but requests return 503

Common causes (Knative mode):
- Pod scaled to zero, cold-starting — wait and retry, or set `minReplicas: 1`
- Knative ingress gateway not routing → use kserve-infra skill

Common causes (Standard mode):
- Service selector mismatch → `kubectl describe svc <name>-predictor -n <ns>` and verify `Endpoints`

## Pattern: InferenceGraph returns 500 for all requests

Symptoms: graph pod logs show connection refused to a step target.

Fix: each step's `InferenceService` must be Ready independently:
```bash
kubectl get inferenceservice -n <ns>
```
Fix the failing step first, then retest the graph.

## Pattern: LLMInferenceService stuck in Progressing

Common cause: KEDA ScaledObject not reconciled, or scheduler deployment not ready.

```bash
kubectl get scaledobject -n <ns>
kubectl get deployment -n <ns> -l app.kubernetes.io/name=<name>
# KEDA operator logs
kubectl logs -n keda -l app=keda-operator --tail=50 | grep <name>
```
