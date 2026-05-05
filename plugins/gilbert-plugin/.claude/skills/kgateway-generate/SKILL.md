---
name: kgateway-generate
description: >
  Full generate-then-review workflow for Solo Enterprise for kgateway routing manifests.
  Use PROACTIVELY when a DevOps engineer wants to create NEW routing configuration
  from scratch. This skill orchestrates the full pipeline:
    1. Collects requirements (service, port, hostname, routing logic)
    2. Delegates generation to the kgateway-routing agent
    3. Automatically passes the generated YAML through kgateway-review
    4. Returns a reviewed, kubectl-ready manifest with a review report
  Invoke this as the DEFAULT entry point for any new kgateway routing request.
  Only skip this and call kgateway-routing directly if the engineer explicitly
  wants generation WITHOUT review.
user-invocable: true
---

# kgateway Generate + Review Orchestration

You are orchestrating a two-phase workflow to produce a reviewed, production-ready
Solo Enterprise for kgateway routing manifest.

## Phase 1 — Gather Requirements

Before generating anything, collect ALL of the following. Ask for missing fields:

| Field | Example | Required? |
|---|---|---|
| Upstream service name | `payments-api` | ✅ Yes |
| Upstream service port | `8080` | ✅ Yes |
| Upstream service namespace | `payments` | ✅ Yes |
| Hostname(s) to route | `api.example.com` | ✅ Yes |
| Gateway name | `http` (default) | ✅ Yes |
| Gateway namespace | `kgateway-system` (default) | ✅ Yes |
| Routing logic | path prefix `/api/v1`, header match, traffic split | ✅ Yes |
| Traffic policy needed? | timeouts, retries, redirects | Optional |
| TLS/HTTPS required? | yes/no | Optional |

Do NOT proceed to Phase 2 until all required fields are confirmed.

---

## Phase 2 — Generate Manifests

Using the collected requirements, generate complete `kubectl apply`-ready YAML for:

1. **`Gateway`** resource (if not already present in the cluster)
2. **`HTTPRoute`** resource with the specified routing logic
3. **`EnterpriseKgatewayTrafficPolicy`** (only if traffic policy was requested)

Apply all rules from the kgateway-routing agent:
- `gatewayClassName: enterprise-kgateway`
- Correct `parentRefs` with Gateway name and namespace
- Correct `apiVersion` for each resource kind
- Complete `backendRefs` with service name and port
- Named listeners and `sectionName` if HTTPS is involved

Separate each resource with `---` and label with a comment header:
```yaml
# --- Gateway ---
...
---
# --- HTTPRoute ---
...
```

---

## Phase 3 — Automatic Review

After generating the manifests, immediately run the `kgateway-review` skill on the
generated YAML **without asking the engineer**. This is automatic.

Perform the full 7-step review from the kgateway-review skill:
1. Identify resources
2. API version check
3. Gateway validation
4. HTTPRoute validation
5. Traffic policy validation (if present)
6. Security checks
7. Output report

---

## Phase 4 — Output

Present results in this order:

### 1. Generated Manifests
```yaml
# Paste the complete, reviewed YAML here
```

### 2. Review Report
```
## kgateway Manifest Review Report
[paste the structured report from Phase 3]
```

### 3. Next Steps
Suggest any follow-up actions:
- Add TLS if HTTP listener is exposed externally
- Add `EnterpriseKgatewayTrafficPolicy` for retries/timeouts
- Restrict `allowedRoutes.namespaces` for production hardening
- Add `ReferenceGrant` if cross-namespace backend refs are needed

---

## Failure Modes

If requirements in Phase 1 are ambiguous:
- Default to the safest option and note the assumption
- Example: if no `allowedRoutes` specified → default to `from: Same`, note it
