# Alloy remote write path (cross-cluster metrics)

Key factors to consider:

- Alloy already supports Prometheus remote write natively
- VictoriaMetrics exposes a remote write endpoint
- TLS and authentication needed for cross-cluster traffic
- Tenant label must be injected so Hub can filter by tenant

Recommendation: Alloy prometheus.remote_write with tenant label relabelling

# Alloy config on spoke (per tenant)
prometheus.remote_write "victoriametrics" {
  endpoint {
    url = "https://victoria.hub.zero-ops.io/api/v1/write"
    
    basic_auth {
      username = env("SPOKE_TENANT_ID")
      password = env("ALLOY_REMOTE_WRITE_TOKEN")
    }
    
    # Inject tenant label on every metric
    write_relabel_config {
      target_label  = "tenant_id"
      replacement   = env("SPOKE_TENANT_ID")
    }
  }
}
```

VictoriaMetrics on Hub receives metrics with `tenant_id` label. Grafana dashboards filter by `tenant_id`. Platform Admin sees all tenants. Tenant Admin dashboard filters to their own `tenant_id` only.

The `ALLOY_REMOTE_WRITE_TOKEN` is a static bearer token created during spoke bootstrap, stored as a Kubernetes Secret, mounted into the Alloy daemonset. Per-spoke token means a compromised spoke cannot write metrics under another tenant's label.

---

## Item 4: Spoke Controller credentials (bootstrap sequence)

**Key factors to consider:**
- Credential must be per-spoke — not a shared platform credential
- Must exist before the Spoke Controller starts
- Must be rotatable without redeploying the controller
- Should use your existing identity infrastructure (Hydra/Kratos)

**Recommendation: Hydra client credentials grant per spoke, provisioned by Crossplane**
```
Crossplane Composition provisions spoke
    ↓
Composition includes a ProviderConfig resource that calls Hub API
    ↓
Hub API creates a Hydra OAuth2 client (client_credentials grant)
    client_id: spoke-{tenant-id}
    client_secret: generated
    scope: hub:resource-status:write
    ↓
Hub API stores client_secret in Hub CNPG (spoke registry)
    ↓
Crossplane writes client_id + client_secret as Kubernetes Secret on spoke
    name: hub-api-credentials
    namespace: spoke-system
    ↓
Spoke Controller mounts secret, requests token from Hydra on startup
    ↓
Uses token for PostgREST writes (Item 1 above)
```

Token rotation is automatic — client credentials tokens are short-lived (1h). The controller requests a new token before expiry. No manual rotation needed.

---

## How the four pieces connect
```
SPOKE BOOTSTRAP SEQUENCE
1. Crossplane provisions spoke cluster
2. Crossplane creates Hydra client → stores credentials as K8s Secret on spoke
3. CRS installs ArgoCD on spoke
4. ArgoCD deploys: sbt-auth, Spoke Controller, Alloy, NATS Leaf Node

SPOKE RUNTIME
sbt-auth          → validates JWTs via in-memory JWKS cache (no Hub call per request)
Spoke Controller  → watches Crossplane → PostgREST → Hub Centralised DB
                    (auth: Hydra client_credentials token, auto-refreshed)
Alloy             → scrapes KSM → remote_write → VictoriaMetrics
                    (auth: per-spoke bearer token, provisioned at bootstrap)
NATS Leaf Node    → billing/lifecycle/notification events → Hub Event Router
                    (auth: NATS credentials, provisioned at bootstrap)

HUB EVENT ROUTER
spoke.*.billing.>       → ClickHouse
spoke.*.notifications.> → notification service  
spoke.*.lifecycle.>     → Hub workflows

## Key principle to codify

All cross-cluster write paths use short-lived tokens issued by Hydra with narrowly scoped permissions. Credentials are provisioned during spoke bootstrap by Crossplane, stored as Kubernetes Secrets, and never shared across spokes. No cross-cluster path exposes raw database ports.