# kgateway DevOps Plugin

A Claude Code plugin for DevOps engineers working with **Solo Enterprise for kgateway**. Generates and reviews Gateway API routing manifests for upstream services with best practices.

## Plugin Structure

```
kgateway-devops-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── .mcp.json                    # Solo.io official docs search MCP (search.solo.io/mcp)
├── agents/
│   └── kgateway-routing.md      # Routing expert agent
└── skills/
    ├── kgateway-generate/
    │   └── SKILL.md             # Entry point: collects requirements + delegates to agent
    └── kgateway-review/
        └── SKILL.md             # Standalone YAML reviewer
```

## How It Works

```
Main session
├── kgateway-generate skill   ← user entry point, collects requirements
│     └── delegates to → kgateway-routing agent  (generates YAML, searches solo-docs MCP)
└── kgateway-review skill     ← auto-invoked in main session to review the output
```

Each component has one responsibility — no duplication, no unnecessary nesting.

## Solo Docs MCP Server

The plugin includes a `.mcp.json` that connects to **Solo.io's official documentation search MCP**
at `https://search.solo.io/mcp`. This gives the routing agent live access to accurate,
current API docs — so generated manifests always reflect the latest schema, not stale training data.

**No local dependencies required** — it's a remote HTTP MCP server, nothing to install.

**Used by:** `kgateway-routing` agent — searches docs before generating
`EnterpriseKgatewayTrafficPolicy` or any field it is uncertain about.

## Components

### `kgateway-routing` Agent
The core routing expert. Knows the full Solo Enterprise for kgateway API surface:
- `Gateway` and `HTTPRoute` (standard Kubernetes Gateway API)
- `EnterpriseKgatewayTrafficPolicy` (Solo extensions)
- All routing patterns: path matching, header matching, traffic splitting, redirects, rewrites, timeouts, retries
- Searches `search.solo.io/mcp` for live schema validation before generating

Claude delegates to this agent automatically when routing questions arise.

### `kgateway-generate` Skill ← **Start here**
The recommended entry point for new routing configs:
1. Collects service name, port, namespace, hostname(s), and routing logic from the engineer
2. Delegates YAML generation to the `kgateway-routing` agent
3. Returns `kubectl apply`-ready manifests

**Invoke:** `/kgateway-devops-plugin:kgateway-generate`
Or just describe what you need — Claude will invoke it automatically.

### `kgateway-review` Skill
Standalone 7-step reviewer for existing manifests. Auto-invoked in the main session
after generation, or manually triggered on any pasted YAML. Checks:
- API versions and deprecated CRDs (Gloo Gateway 2.0 vs Solo Enterprise for kgateway 2.1+)
- Gateway + HTTPRoute structural correctness
- Security posture (wildcard hosts, missing HTTPS redirect, cross-namespace refs)
- Traffic policy attachments

**Invoke:** `/kgateway-devops-plugin:kgateway-review`
Or paste a manifest and ask Claude to review it.

---

## Demonstration

Create a gateway resource:
```bash
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

```bash
claude
```

## Usage Examples inside `Claude TUI`

# Generate a new route
I am using Solo Enterprise for kgateway 2.1.4. Create an HTTPRoute for my payments-api service on api.example.com based on Solo/gateway.yaml. Store generated artifacts inside Solo folder.

# Review an existing manifest and enhance security
Review all resources in Solo folder for correctness and enhance security
```

## Resources Supported

| Resource | apiVersion | Description |
|---|---|---|
| `Gateway` | `gateway.networking.k8s.io/v1` | Envoy proxy + listeners |
| `HTTPRoute` | `gateway.networking.k8s.io/v1` | Routing rules to upstream services |
| `EnterpriseKgatewayTrafficPolicy` | `enterprisekgateway.solo.io/v1alpha1` | Retries, timeouts, ext auth, JWT |


