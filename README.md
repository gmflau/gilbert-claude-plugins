# User Guide: gilbert-plugin

A Claude Code plugin for DevOps engineers working with **Solo Enterprise for kgateway**. Generates and reviews Gateway API routing manifests for upstream services with best practices.

Instead of manually authoring `Gateway`, `HTTPRoute`, and `EnterpriseKgatewayTrafficPolicy` YAML from scratch, you describe what you need in plain language and the plugin generates correct, production-hardened manifests — then automatically reviews them against a 7-step checklist before you ever run `kubectl apply`. Every generation and review session queries the **live Solo.io documentation MCP server** at runtime, so the output always reflects the current API schema rather than stale training data.

**Marketplace:** [github.com/gmflau/gilbert-claude-plugins](https://github.com/gmflau/gilbert-claude-plugins)

---

## Prerequisites

- Claude Code CLI installed (`claude --version` to verify)
- Access to the internet (the plugin queries `search.solo.io/mcp` at runtime — no local install required)
- A project directory where you work with kgateway configs

---

## Installation

### 1. Launch Claude Code

```bash
claude
```

### 2. Add the marketplace and install the plugin

Inside Claude TUI:
```
/plugin marketplace add gmflau/gilbert-claude-plugins
```
```
/plugin install gilbert-plugin@team-tools
```
When prompted, choose **`Install for you (user scope)`**.

This installs the plugin into `~/.claude/plugins/` — your personal Claude Code profile — so it is available in every project you open on this machine without needing to copy anything into each repo. Use project scope only if you want the plugin committed into a specific repository so your whole team gets it automatically when they clone.

### 3. Reload plugins

```
/reload-plugins
```

Claude Code loads plugins at startup. `/reload-plugins` applies the install to the current session without requiring a restart — you will see the new skills become available immediately after running it.

### 4. Confirm the installation

Type `/kgateway` in the Claude TUI — autocomplete will show the two installed skills, confirming the plugin is active:
```
/kgateway
```
Expected output:
```
/kgateway-review              Reviews Solo Enterprise for kgateway Gateway and HTTPRoute YAML manifests for correctness, security, and best practices. Use PROACTIVELY when: - A Gateway or HTTPRoute           
                              manifest has just been generated and needs validation - An engineer asks to review or audit existing kgateway routing YAML - Checking for deprecated CRDs (Gloo Gateway 2.0 v…    
/kgateway-generate            Full generate-then-review workflow for Solo Enterprise for kgateway routing manifests. Use PROACTIVELY when a DevOps engineer wants to create NEW routing configuration from      
                              scratch. This skill orchestrates the full pipeline: 1. Collects requirements (service, port, hostname, routing logic) 2. Delegates generation to the kgateway-routing agent 3… 
```

Type `/mcp` in the Claude TUI - 
```
/mcp
```
Expected output:
```
Manage MCP serverss
4 servers
  
  User MCPs (/Users/gilbetlau/.claude.json)
> soloio-docs-mcp · ✔ connected · 3 tools 
```

`soloio-docs-mcp · ✔ connected · 3 tools` confirms the plugin wired up the **Solo.io documentation MCP server** at `https://search.solo.io/mcp`. The 3 tools (`search`, `get_chunks`, `get_full_page`) are what the `kgateway-routing` agent calls at runtime to look up live API schemas before generating or validating any manifest — ensuring output always reflects the current Solo Enterprise for kgateway release, not stale training data.

Type `/agents` in the Claude TUI, then navigate to the **Library** tab to verify the routing agent was installed:
```
/agents
```
Expected output in the Library tab:
```
Create new agent 

Project agents (/Users/gilbertlau/Documents/anthropic/SOLUTION/.claude/agents)
kgateway-routing · sonnet 
```

`kgateway-routing · sonnet` confirms the plugin's domain expert agent is registered and ready. This agent is invoked automatically by the `kgateway-generate` and `kgateway-review` skills — you never call it directly. It runs on Claude Sonnet, carries the full Solo Enterprise for kgateway API knowledge, and is the component that queries the `soloio-docs-mcp` server for live schema lookups during generation.

---

## What the Plugin Gives You

| Component | What it does |
|---|---|
| `/gilbert-plugin:kgateway-generate` | Collects your requirements, generates Gateway + HTTPRoute YAML, and automatically reviews it before returning the result |
| `/gilbert-plugin:kgateway-review` | Runs a 7-step review on any existing manifest — paste YAML or point at a file |
| `kgateway-routing` agent | The underlying domain expert (invoked automatically — no need to call it directly) |
| Solo.io docs MCP | Live documentation search used during generation and review so manifests always match the current API schema |

---

## Skill 1: Generate a New Routing Config

**Invoke:** `/gilbert-plugin:kgateway-generate`
Or just describe what you need — Claude will invoke it automatically.

### What Claude will ask you

Before generating anything, the skill collects:

| Field | Example |
|---|---|
| Upstream service name | `payments-api` |
| Upstream service port | `8080` |
| Upstream service namespace | `payments` |
| Hostname(s) to route | `api.example.com` |
| Gateway name | `http` |
| Gateway namespace | `kgateway-system` |
| Routing logic | path prefix `/api/v1`, header match, traffic split |
| Traffic policy (optional) | timeouts, retries |
| TLS/HTTPS (optional) | yes / no |

### Example prompt

```
I am using Solo Enterprise for kgateway 2.1.4. Create an HTTPRoute for my
payments-api service on api.example.com. Gateway is 'http' in kgateway-system.
Service is in the payments namespace on port 8080. Route all traffic with
path prefix /api/v1. Store generated artifacts in the Solo/ folder.
```

### What you get back

1. **Generated manifests** — complete, `kubectl apply`-ready YAML separated by `---`:
   - `Gateway` (if not already in the cluster)
   - `HTTPRoute` with your routing rules
   - `EnterpriseKgatewayTrafficPolicy` (only if you requested traffic policies)

2. **Review report** — automatically run on the generated YAML, formatted as:
   ```
   ## kgateway Manifest Review Report

   ### ✅ Passed Checks
   ### ⚠️ Warnings (should fix before production)
   ### 🚨 Critical Issues (must fix before applying)
   ### 💡 Recommendations

   ### Verdict: READY TO APPLY | NEEDS FIXES | BLOCKED
   ```

3. **Next steps** — suggestions for hardening, TLS, or additional policies

---

## Skill 2: Review an Existing Manifest

**Invoke:** `/gilbert-plugin:kgateway-review`
Or paste a manifest and ask Claude to review it — it will invoke the skill automatically.

### Example prompts

```
Review all resources in the Solo/ folder for correctness and enhance security.
```

```
/gilbert-plugin:kgateway-review
```
Then paste your YAML.

### What the 7-step review checks

| Step | Checks |
|---|---|
| 1. Resource identification | Lists all kinds found; flags unknown or deprecated CRDs immediately |
| 2. API version | Blocks `gateway.solo.io/v2` (Gloo 2.0 — incompatible with kgateway 2.1+) and `v1beta1` |
| 3. Gateway validation | `gatewayClassName: enterprise-kgateway`, listener protocol, TLS certs, `allowedRoutes` scope |
| 4. HTTPRoute validation | `parentRefs` correctness, `hostnames`, `backendRefs` with port, cross-namespace `ReferenceGrant` |
| 5. Traffic policy (if present) | `targetRefs`, timeout values, valid Envoy retry conditions |
| 6. Security | Wildcard hostnames, missing HTTPS redirect, hardcoded secrets, namespace isolation |
| 7. Structured report | `✅ / ⚠️ / 🚨` report with a final verdict |

### Common issues caught

| Issue | Severity |
|---|---|
| Wrong `gatewayClassName` | Critical |
| Missing `parentRefs.namespace` | Critical |
| `GlooTrafficPolicy` used (Gloo 2.0 CRD) | Critical |
| Cross-namespace `backendRef` without `ReferenceGrant` | Critical |
| Missing `port` in `backendRefs` | Critical |
| Missing `hostnames` (routes all hosts) | Warning |
| `allowedRoutes: All` in production | Warning |
| No HTTPS redirect on external HTTP listener | Warning |

---

## Resources the Plugin Understands

| Resource | apiVersion | Description |
|---|---|---|
| `Gateway` | `gateway.networking.k8s.io/v1` | Envoy proxy + listeners |
| `HTTPRoute` | `gateway.networking.k8s.io/v1` | Routing rules to upstream services |
| `EnterpriseKgatewayTrafficPolicy` | `enterprisekgateway.solo.io/v1alpha1` | Retries, timeouts, ext auth, JWT |

---

## Full Demo Walkthrough

### 1. Create a Gateway resource to work from

```bash
mkdir Solo

cat << 'EOF' > ./Solo/gateway.yaml
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: http
  namespace: kgateway-system
spec:
  gatewayClassName: enterprise-kgateway
  listeners:
  - protocol: HTTP
    port: 8080
    name: http
    allowedRoutes:
      namespaces:
        from: All
EOF
```

### 2. Open Claude Code
```bash
claude
```

### 3. Generate a route
Inside the Claude TUI, enter the following:
```
I am using Solo Enterprise for kgateway 2.1.4. Create an HTTPRoute for my payments-api service on api.example.com based on Solo/gateway.yaml. Store generated artifacts in the Solo folder.
```

Claude will invoke `kgateway-generate`, ask any missing questions, generate the YAML, auto-review it, and return the report.    

Provide the input to the prompt:
Enter
```
service port 8080, service namespace payments-api, routing logic path prefix /api/v1
```
Then continue to respond to subseqent prompts to improve the gatewy and httproute yaml configurations.


### 4. Review and harden existing configs
```
Review all resources in Solo folder for correctness and enhance security
```

Claude will invoke `kgateway-review` on each resource and return a structured report with any warnings or blockers.

---

