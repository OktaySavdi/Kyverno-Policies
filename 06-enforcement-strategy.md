# Enforcement Strategy: Kyverno v1.17 Rollout Guide

> Audit first. Fix. Enforce. Repeat per policy group.

---

## Why Start with Audit?

`validationActions: [Audit]` records violations to PolicyReports without blocking
anything. This lets you:

1. Discover all existing non-compliant workloads before enforcement
2. Give teams time to fix issues without causing outages
3. Build confidence that a policy is correct before it blocks admission

**Never** deploy a new policy directly to `[Deny]` in a production cluster
without an Audit period first — not even security-critical policies.

---

## Audit Mode -> Enforce Mode Promotion Checklist

Before changing `validationActions: [Audit]` to `validationActions: [Deny]`:

- [ ] Policy has been running in Audit for at least **1 sprint (2 weeks)**
- [ ] PolicyReport violation count is **zero** (or all remaining violations are
      accepted via PolicyExceptions)
- [ ] Teams have been notified of the upcoming enforcement date
- [ ] A rollback plan exists (see below)
- [ ] Policy has been tested with `kyverno test` in CI

```bash
# Check current violations for a policy
kubectl get policyreport -A -o json | \
  jq '.items[].results[] | select(.policy == "disallow-latest-tag")'

# Count violations per policy
kubectl get policyreport -A -o json | \
  jq '[.items[].results[] | select(.result == "fail")] | group_by(.policy) |
      map({policy: .[0].policy, count: length})'
```

---

## Recommended Rollout Timeline

### Week 1-2: Apply All Policies in Audit Mode

```bash
# All policies default to [Audit] — change validationActions before applying
kubectl apply -f Kyverno/01-security.yaml
kubectl apply -f Kyverno/02-high-availability.yaml
kubectl apply -f Kyverno/03-reliability.yaml
kubectl apply -f Kyverno/04-governance.yaml
kubectl apply -f Kyverno/05-mutate.yaml    # MutatingPolicy applies immediately
```

Mutate policies are safe to apply immediately — they only add resources, they never block.

### Week 3-4: Fix Violations

```bash
# Get all failing resources across all namespaces
kubectl get policyreport -A

# Get details for a specific namespace
kubectl describe policyreport -n my-app

# Watch violations in real time
kubectl get policyreport -A -w
```

For violations that cannot be fixed quickly, create a `PolicyException`:

```yaml
apiVersion: policies.kyverno.io/v1
kind: PolicyException
metadata:
  name: legacy-app-exceptions
  namespace: legacy
spec:
  exceptions:
    - policyName: require-resource-requests-limits
      ruleNames:
        - require-resource-requests-limits
  match:
    any:
      - resources:
          kinds: [Pod]
          names: ["legacy-app-*"]
          namespaces: [legacy]
```

### Week 5+: Promote to Deny Group by Group

Promote in this order (least to most risky):

1. **Governance** (`04`) — affects namespace creation, low blast radius
2. **Security** (`01`) — high value; includes `disallow-secrets-from-env` and
   `disallow-unsafe-sysctls` (new). Promote after auditing all workloads for
   Secret env usage and unusual sysctls.
   > Security context validators (`require-run-as-non-root`,
   > `enforce-readonly-root-filesystem`, `disallow-privilege-escalation`,
   > `require-drop-all-capabilities`, `require-seccomp-profile`) remain
   > **Audit-only permanently** — the MutatingPolicies auto-correct these values
   > so developers are never blocked for forgetting or mis-setting them.
3. **Reliability** (`03`) — resource requests + memory limits. Note: CPU limits
   are intentionally **not enforced** to avoid Linux CFS quota throttling. Only
   `requests.cpu`, `requests.memory`, and `limits.memory` are required.
4. **High Availability** (`02`) — replicas, probes (most disruptive for legacy apps)

```bash
# Example: promote disallow-latest-tag to Deny
kubectl edit validatingpolicy disallow-latest-tag
# Change: validationActions: [Audit]
# To:     validationActions: [Deny]
```

---

## Rollback Plan

```bash
# Immediately revert a policy to Audit if it causes an outage
kubectl patch validatingpolicy POLICY_NAME \
  --type=merge \
  -p '{"spec":{"validationActions":["Audit"]}}'

# Or delete the policy entirely if needed
kubectl delete validatingpolicy POLICY_NAME
```

---

## Policy Testing in CI (kyverno test)

Before merging any policy change, test it locally:

```bash
# Install Kyverno CLI
brew install kyverno

# Test a specific policy with a pass/fail resource fixture
kyverno test 01-security.yaml

# Test all policies
kyverno test Kyverno/
```

Recommended test fixture structure:

```
Kyverno/
  tests/
    01-security/
      disallow-privileged/
        pass-pod.yaml          # should pass
        fail-pod-privileged.yaml   # should fail
        kyverno-test.yaml      # test definition
```

---

## Summary: When to Use Each Mode

| Scenario | Mode |
|----------|------|
| New policy, unknown compliance state | `[Audit]` |
| Policy running 2+ weeks, zero violations | Promote to `[Deny]` |
| Known legacy exceptions exist | `[Audit]` + `PolicyException` |
| Security critical, green-field cluster | `[Deny]` from day 1 |
| Mutate / Generate policies | Always active (no action needed) |

---

## Useful Commands

```bash
# View all policy reports
kubectl get policyreport -A

# View cluster-wide reports
kubectl get clusterpolicyreport

# Count violations per namespace
kubectl get policyreport -A -o json | \
  jq '.items[] | {namespace: .metadata.namespace, fail: [.results[] | select(.result=="fail")] | length}'

# Check Kyverno admission controller logs
kubectl logs -n kyverno -l app.kubernetes.io/component=admission-controller -f
```
