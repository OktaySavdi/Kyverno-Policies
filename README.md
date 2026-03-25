# Kyverno Production Policy Set

> Kyverno v1.17 | `policies.kyverno.io/v1` CEL-based | Production-Ready

---

## Overview

All policies use the **stable Kyverno v1.17 API** (`policies.kyverno.io/v1`) with CEL
expressions. The old `kyverno.io/v1 ClusterPolicy` pattern-based API is superseded by
the dedicated CEL-based kinds below.

---

## Kyverno v1.17 Policy Kinds (all Stable)

| Kind | API | Replaces |
|------|-----|----------|
| `ValidatingPolicy` | `policies.kyverno.io/v1` | ClusterPolicy validate rules |
| `MutatingPolicy` | `policies.kyverno.io/v1` | ClusterPolicy mutate rules |

### How enforcement works

**ValidatingPolicy** — `validationActions`:
- `[Deny]` — blocks the admission request (hard enforce)
- `[Audit]` — records to PolicyReport, does not block

**MutatingPolicy** — patches via CEL `Object{}` (ApplyConfiguration) or `JSONPatch{}`.

---

## File Structure

```
Kyverno/
├── README.md                       # This file
├── 01-security.yaml                # ValidatingPolicy — container security
├── 02-high-availability.yaml       # ValidatingPolicy — HA & resilience
├── 03-reliability.yaml             # ValidatingPolicy — resources & labels
├── 04-governance.yaml              # ValidatingPolicy — namespace governance
├── 05-mutate.yaml                  # MutatingPolicy   — inject defaults
└── 07-enforcement-strategy.md      # Rollout guide: Audit -> Enforce
```

---

## Policy Summary

### 01-security.yaml — Container & Pod Security (ValidatingPolicy)

All remaining validators are `[Deny]` only for things that **cannot be safely auto-fixed** by a mutation.

| # | Name | Action | Note |
|---|------|--------|------|
| 1 | `disallow-latest-tag` | Deny | |
| 2 | `restrict-image-registries` | Deny | update allowlist |
| 3 | `disallow-host-namespaces` | Deny | hostNetwork/PID/IPC |
| 4 | `disallow-host-ports` | Deny | |
| 5 | `restrict-automount-sa-token` | Deny | default SA only |
| 6 | `disallow-share-process-namespace` | Deny | |
| 7 | `disallow-secrets-from-env` | Deny | blocks secretKeyRef/secretRef in env |
| 8 | `disallow-unsafe-sysctls` | Deny | kernel safe-sysctl allowlist only |

### 02-high-availability.yaml — HA & Resilience (ValidatingPolicy)

| # | Name | Action | Note |
|---|------|--------|------|
| 1 | `require-minimum-replicas` | Deny | ha-exempt opt-out |
| 2 | `require-liveness-probe` | Deny | |
| 3 | `require-readiness-probe` | Deny | |
| 4 | `require-pod-anti-affinity` | Audit | |
| 5 | `require-pod-disruption-budget` | Audit | multi-replica only |

### 03-reliability.yaml — Resources & Labels (ValidatingPolicy)

| # | Name | Action | Note |
|---|------|--------|------|
| 1 | `require-resource-requests-limits` | Deny | requests.cpu/memory + limits.memory (no CPU limit) |
| 2 | `require-required-labels` | Deny | app, team, env |
| 3 | `disallow-host-path` | Deny | |

### 04-governance.yaml — Namespace & Cluster Governance (ValidatingPolicy)

| # | Name | Action | Note |
|---|------|--------|------|
| 1 | `restrict-default-namespace` | Deny | |
| 2 | `disallow-nodeport-services` | Deny | |
| 3 | `require-namespace-owner-label` | Deny | team + env labels |
| 4 | `require-ingress-tls` | Deny | |
| 5 | `restrict-rbac-wildcards` | Audit | ClusterRoles only |

### 05-mutate.yaml — Auto-Inject Defaults (MutatingPolicy)

| # | Name | What it injects |
|---|------|----------------|
| 1 | `mutate-add-standard-labels` | managed-by, env labels |
| 2 | `mutate-add-default-resource-limits` | cpu/memory requests + memory limit (no CPU limit injected) |
| 3 | `mutate-set-allow-privilege-escalation-false` | always sets `false` — overwrites explicit `true` |
| 4 | `mutate-set-privileged-false` | always sets `false` — overwrites explicit `true` |
| 5 | `mutate-set-readonly-root-filesystem` | always sets `true` — overwrites explicit `false` |
| 6 | `mutate-drop-all-capabilities` | ensures `drop:[ALL]` — adds it even when a caps block exists |
| 7 | `mutate-set-run-as-non-root` | always sets `runAsNonRoot:true`; sets `runAsUser:65534` only if missing/root |
| 8 | `mutate-set-seccomp-runtime-default` | sets `RuntimeDefault` — overwrites explicit `Unconfined` |

**Total: 24 policies** (8 ValidatingPolicy, 8 MutatingPolicy, 1 strategy doc)

### Auto-Fix Security Context enforcement

MutatingPolicies in `05-mutate.yaml` run **before** any ValidatingPolicy and
**always correct** security context fields — whether missing or explicitly unsafe:

```
Admission request
       │
       ▼
  MutatingPolicy (05-mutate.yaml)          ← runs first, ALWAYS corrects
  • mutate-set-privileged-false                   privileged: true  → false
  • mutate-set-allow-privilege-escalation-false   allowPrivilegeEscalation: true  → false
  • mutate-set-readonly-root-filesystem           readOnlyRootFilesystem: false  → true
  • mutate-drop-all-capabilities                  adds ALL to drop list
  • mutate-set-run-as-non-root                    runAsNonRoot: false/0  → true/65534
  • mutate-set-seccomp-runtime-default            Unconfined  → RuntimeDefault
       │
       │   Service owners who forget or mis-set security context are NEVER blocked.
       ▼
  ValidatingPolicy (01-security.yaml)      ← Deny only for non-auto-fixable risks
  • disallow-latest-tag            — image pinning cannot be auto-corrected
  • restrict-image-registries      — supply-chain boundary cannot be auto-corrected
  • disallow-host-namespaces       — host-level access must never silently change
  • disallow-host-ports            — node IP binding must never silently change
  • restrict-automount-sa-token    — API token exposure must be explicit
  • disallow-share-process-namespace
  • disallow-secrets-from-env      — secret data must not leak via env
  • disallow-unsafe-sysctls        — kernel param safety
       │
       ▼
  Pod admitted
```

**Result**: service owners never get blocked for forgetting or mis-setting a
security context field. Every workload always meets the security baseline.
Hard `[Deny]` blocks are reserved for configurations that silently auto-fixing
would hide a genuine design risk.

---

## Quick Start

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno \
  --namespace kyverno --create-namespace \
  --version 3.3.x \
  --set admissionController.replicas=3 \
  --set backgroundController.replicas=2

# Apply all policies (start Audit, then promote to Deny)
kubectl apply -f Kyverno/
kubectl get policyreport -A
```
