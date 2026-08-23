---
name: kserve-platform
description: "Observe and debug live KServe deployments. Use when an InferenceService, LLMInferenceService, or InferenceGraph is stuck, failing, or returning errors — symptoms like Unknown/False Ready condition, pod CrashLoop, OOMKilled, ImagePullBackOff, routing failures, slow first token, or unexpected 5xx from the inference endpoint. Don't use for generating new manifests (use kserve-workload) or installing KServe (use kserve-infra)."
compatibility: Requires kubectl connected to the cluster with the affected workload.
allowed-tools: Bash(kubectl:*)
---

# KServe Platform

Diagnoses live KServe deployments. Read-only triage first — propose manifest changes only after root cause is confirmed.

## Quick Reference

| | |
|---|---|
| Status conditions | `kubectl get inferenceservice <name> -o jsonpath='{.status.conditions}'` |
| Controller logs | `kubectl logs -n kserve -l control-plane=kserve-controller-manager --tail=50` |
| Failure patterns & docs | [references/failure-patterns.md](references/failure-patterns.md) |
| Related skills | kserve-workload (fix/regenerate manifests), kserve-infra (gateway/cert issues) |

## Rules

1. Always read conditions and events before reading logs — they identify which layer to investigate.
2. Never patch or delete resources without confirming root cause and getting user approval.
3. Check controller logs only after ruling out pod-level issues — most failures are visible in pod events/logs.
4. If root cause is a gateway, cert-manager, or Istio config: hand off to kserve-infra.

## Triage Workflow

### Step 0 — Gather context

Auto-discover what you can, then ask for what's missing before running any diagnostic commands.

```bash
# List all KServe resources across namespaces — often resolves name+namespace without asking
kubectl get inferenceservice,llminferenceservice,inferencegraph -A 2>/dev/null
# Current context
kubectl config current-context
```

Required inputs — resolve from the user's prompt or the listing above, then ask only for what remains:

| Input | Auto-discoverable? | Ask if missing |
|---|---|---|
| Resource name | Often in prompt; check listing | Yes — "What is the name of the resource?" |
| Namespace | From listing or kubeconfig context | Yes — "Which namespace is it in?" |
| Resource type | From listing or symptom | Infer from listing; ask if ambiguous |

Do not run any diagnostic command with a literal `<name>` or `<ns>` placeholder.

### Step 1 — Identify the resource and its status

```bash
# Determine resource type first
kubectl get inferenceservice,llminferenceservice,inferencegraph -A 2>/dev/null | grep <name>

# Get full status
kubectl get inferenceservice <name> -n <ns> -o yaml | grep -A 40 'status:'
```

Condition decision tree:

| Condition | Status | Meaning | Next step |
|---|---|---|---|
| `PredictorReady` | `False` or `Unknown` | Predictor pod not ready | Step 2 — check pod state |
| `PredictorRouteReady` | `False` or `Unknown` | Routing not configured | Step 4 — check routing, then kserve-infra |
| `LatestDeploymentReady` | `False` | Knative revision not ready (serverless only) | Step 2 — check pod state |
| `Ready` | `True` but inference fails | Service up, request failing | Step 5 — test endpoint |

Use `kubectl get inferenceservice <name> -n <ns> -o jsonpath='{.status.conditions}'` for the raw condition list.

### Step 2 — Check pod state

```bash
kubectl get pods -n <ns> -l serving.kserve.io/inferenceservice=<name>
kubectl describe pod <pod> -n <ns>   # Events section is key
```

Pod decision tree:

| Pod state | Likely cause | Action |
|---|---|---|
| `Pending` | Unschedulable (resource/taint) | Check node resources: `kubectl describe node` |
| `Init:Error` or `Init:CrashLoop` | storage-initializer failure | Step 3a |
| `CrashLoopBackOff` | Model server startup failure | Step 3b |
| `OOMKilled` | Memory limit too low | Increase `resources.limits.memory` |
| `ImagePullBackOff` | Bad image ref or missing pull secret | Check runtime image tag |
| `Running` but not Ready | Readiness probe failing | Step 3b |

### Step 3a — Storage initializer (init container) logs

```bash
kubectl logs <pod> -n <ns> -c storage-initializer
```

Common failures:

| Log pattern | Cause | Fix |
|---|---|---|
| `credential not found` | Missing storage secret | Create secret with correct key |
| `scheme not supported` | storageUri prefix unknown | Check `kubectl get clusterstoragecontainer default` |
| `permission denied` | IAM / RBAC on storage bucket | Fix bucket policy |

### Step 3b — Model server logs

```bash
kubectl logs <pod> -n <ns> -c kserve-container --tail=100
kubectl logs <pod> -n <ns> -c kserve-container --previous 2>/dev/null  # after crash
```

### Step 4 — Check routing (InferenceService URL)

```bash
kubectl get inferenceservice <name> -n <ns> -o jsonpath='{.status.url}'

# Knative mode: check Knative Service
kubectl get ksvc -n <ns> -l serving.kserve.io/inferenceservice=<name>

# Standard mode: check K8s Service
kubectl get svc -n <ns> -l serving.kserve.io/inferenceservice=<name>
```

If URL is empty or route not admitted → hand off to kserve-infra.

### Step 5 — Test the inference endpoint

```bash
kubectl port-forward svc/<name>-predictor -n <ns> 8080:80 &
# V1 protocol
curl -s http://localhost:8080/v1/models/<name>
# V2 protocol
curl -s http://localhost:8080/v2/models/<name>
# OpenAI-compatible (vLLM / HuggingFace)
curl -s http://localhost:8080/openai/v1/models
kill %1
```

### Step 6 — Controller logs (last resort)

```bash
kubectl logs -n kserve -l control-plane=kserve-controller-manager --tail=100 | grep -i "error\|<name>"
```

## InferenceGraph Triage

```bash
kubectl get inferencegraph <name> -n <ns> -o yaml | grep -A 20 'status:'
kubectl get pods -n <ns> -l serving.kserve.io/inferencegraph=<name>
kubectl logs <router-pod> -n <ns>
```

Common graph failures:
- Step target URL wrong or service not Ready → check each step's `InferenceService` status individually
- Splitter weights don't sum to 100 → webhook should catch this; if not, check the spec

## LLMInferenceService Triage

```bash
kubectl get llminferenceservice <name> -n <ns> -o yaml | grep -A 60 'status:'
# KEDA ScaledObject
kubectl get scaledobject -n <ns> -l app.kubernetes.io/name=<name>
# Scheduler (if disaggregated)
kubectl get pods -n <ns> -l app.kubernetes.io/component=scheduler,app.kubernetes.io/name=<name>
```

## Gotchas

- `LLMInferenceService` uses a separate controller. Check its logs for reconcile errors: `kubectl logs -n kserve -l control-plane=llmisvc-controller-manager --tail=100`.
- `Ready: True` reflects the *previous* spec generation until `observedGeneration` matches `metadata.generation`. A fresh spec change may show stale True briefly.
- In Knative mode, the Knative route can be Ready while the pod is still cold-starting — the first request may time out. Check `kubectl get revision -n <ns>`.
- The `storage-initializer` runs as an init container. Its logs disappear if the pod is replaced. Capture them immediately: `kubectl logs <pod> -c storage-initializer`.

## Progressive Disclosure

| Need | Load |
|---|---|
| Common failure patterns with fixes | [references/failure-patterns.md](references/failure-patterns.md) |
