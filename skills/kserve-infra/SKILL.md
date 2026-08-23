---
name: kserve-infra
description: "Install, configure, and troubleshoot KServe infrastructure: Helm install, deployment mode selection (Knative vs Standard/raw), cert-manager, Istio, Envoy Gateway, Gateway API, and ingress setup. Use when installing KServe, choosing a deployment mode, fixing gateway or routing issues, or configuring dependencies. Don't use for model deployment (use kserve-workload) or debugging a specific InferenceService (use kserve-platform)."
compatibility: Requires kubectl and helm connected to the target cluster.
allowed-tools: Bash(kubectl:*) Bash(helm:*)
---

# KServe Infra

Installs and configures KServe and its dependencies. Confirm cluster state before any install or upgrade.

## Quick Reference

| | |
|---|---|
| Install steps & version pins | [references/install-guide.md](references/install-guide.md) |
| Deployment mode details | [references/install-modes.md](references/install-modes.md) |
| Related skills | kserve-workload (deploy models), kserve-platform (debug routing failures) |

## Rules

1. Check existing install before any `helm install/upgrade`: `helm ls -A | grep kserve`.
2. Pin all dependency versions — never use `latest`. Read [references/install-guide.md](references/install-guide.md) for current versions.
3. `LLMInferenceService` requires Standard (raw) mode — do not install Knative if the cluster only uses LLMISVC.
4. For routing issues after install, verify gateway before debugging InferenceService — hand off to kserve-platform only after gateway is confirmed healthy.

## Step 0 — Gather context

Auto-discover what you can, then ask for what's missing before running any install or upgrade.

```bash
# What's already installed?
helm ls -A | grep -E "kserve|istio|knative|cert-manager|keda|gateway"
kubectl get crd | grep -E "kserve|knative|gateway|cert-manager"
kubectl version --short 2>/dev/null || kubectl version
# Current context (confirm with user before any install)
kubectl config current-context
```

Required inputs — resolve from the user's prompt or ask:

| Input | Auto-discoverable? | Ask if missing |
|---|---|---|
| Target cluster / context | Yes (from kubeconfig) | Confirm before any install — "I'll install on context `<ctx>`, OK?" |
| Deployment mode | Sometimes in prompt | Ask — "Knative (scale-to-zero) or Standard (raw)? Default: Standard." |
| KServe version | Check latest release | Ask if user has a preference; default to latest stable |
| Namespace | Defaults to `kserve` | Only ask if user wants a custom namespace |

Do not run any `helm install` or `kubectl apply` before confirming the target context with the user.

## Step 1 — Choose deployment mode

| Mode | When to use |
|---|---|
| **Standard (Raw)** | `LLMInferenceService`; no Knative; simpler ops |
| **Knative (Serverless)** | Scale-to-zero for `InferenceService` |

Default for new installs: **Standard** unless scale-to-zero is explicitly required.

For setup details per mode: [references/install-modes.md](references/install-modes.md)

Check current cluster default:
```bash
kubectl get configmap inferenceservice-config -n kserve \
  -o jsonpath='{.data.deploy}' | python3 -m json.tool | grep deploymentMode
```

## Step 2 — Install dependencies and KServe

Load [references/install-guide.md](references/install-guide.md) for official install docs and version pins.

Verify after install:
```bash
kubectl get pods -n kserve
kubectl get crd | grep kserve
```

## Step 3 — Verify gateway

```bash
kubectl get gatewayclass
kubectl get gateway -A
kubectl get inferenceservice -A -o wide
```

## Diagnosing gateway/routing failures

```bash
# Is the InferenceService URL assigned?
kubectl get inferenceservice <name> -n <ns> -o jsonpath='{.status.url}'

# Standard mode: HTTPRoute accepted?
kubectl get httproute -n <ns> -l serving.kserve.io/inferenceservice=<name>

# Knative mode: Knative route admitted?
kubectl get route -n <ns> -l serving.kserve.io/inferenceservice=<name>

# Gateway controller logs
kubectl logs -n envoy-gateway-system \
  -l app.kubernetes.io/name=gateway-helm --tail=50
```

## Gotchas

- `cert-manager` must be fully Ready before applying KServe CRDs — the webhook rejects resources otherwise. Always `kubectl wait` before proceeding.
- `LLMInferenceService` will silently fail to route in Knative mode — the controller won't create a Knative Service. Set `defaultDeploymentMode: Standard` in `inferenceservice-config` or use the per-resource annotation.
- Gateway API CRDs (`GatewayClass`, `HTTPRoute`, etc.) must be installed before the KServe chart in Gateway API mode. Check: `kubectl get crd gatewayclasses.gateway.networking.k8s.io`.
- Knative and Standard modes can coexist in the same cluster; the per-resource annotation overrides the cluster default.

## Progressive Disclosure

| Need | Load |
|---|---|
| Install steps, version pins, official docs | [references/install-guide.md](references/install-guide.md) |
| Detailed per-mode setup (Knative vs Standard) | [references/install-modes.md](references/install-modes.md) |
