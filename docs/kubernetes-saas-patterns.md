---
inclusion: always
---

# Kubernetes SaaS Patterns for open-sbt

This steering document identifies common SaaS user needs and maps them to open-sbt abstractions and Kubernetes deployment patterns, based on analysis of the AWS SaaS Factory EKS Reference Architecture.

## User Need Categories

### 1. Multi-Tenant Application Deployment

**User Need:** "I want to deploy my SaaS application with proper tenant isolation on Kubernetes"

**open-sbt Pattern:** Use the **IProvisioner** interface called from the **Hub Tenant Management API**

**Implementation Approach:**
```go
// IProvisioner is implemented by the Hub Tenant Management API.
// It generates Helm values / Crossplane manifests and commits a new tenant folder
// to the central GitOps repo. ArgoCD on the Hub picks it up and syncs to the Spoke.
provisioner := gitops.NewGitOpsProvisioner(gitops.Config{
    RepoURL: "...",
})

// Hub Tenant Management API commits tenant state to central GitOps repo
result, err := provisioner.CommitTenantState(ctx, opensbt.ProvisionRequest{
    TenantID: "tenant-123",
    Tier:     "premium",
    Resources: []opensbt.ResourceSpec{
        {Type: "namespace", Name: "tenant-123"},
        {Type: "deployment", Name: "app-services"},
        {Type: "service", Name: "app-service"},
        {Type: "ingress", Name: "app-ingress"},
    },
})
```

**Kubernetes Deployment Patterns:**
- **Namespace-per-Tenant**: Each tenant gets dedicated namespace
- **Service Account per Tenant**: RBAC isolation via service accounts
- **Network Policies**: Prevent cross-tenant network access
- **Resource Quotas**: Limit tenant resource consumption by tier

### 2. Tenant Onboarding Automation

**User Need:** "I want automated tenant provisioning when new customers sign up"

**open-sbt Pattern:** Use **Hub Tenant Management API (Control Plane)** with **IProvisioner**

**Implementation Approach:**
```go
// Hub Tenant Management API commits infrastructure intent to Git
func (c *ControlPlane) CreateTenant(ctx context.Context, req opensbt.CreateTenantRequest) error {
    // 1. Save to PostgreSQL (Status: Pending)
    c.db.InsertTenant(...)
    
    // 2. Strict GitOps: Generate manifests and commit new tenant folder to central GitOps repo
    err := c.gitopsProvisioner.CommitTenantState(ctx, opensbt.ProvisionRequest{
        TenantID: req.TenantID,
        Tier:     req.Tier,
    })
    
    // 3. Publish NATS event for non-infra side effects only (Billing, Notifications)
    c.eventBus.Publish("opensbt_tenantCreated", req)
    
    return err
}
```

**Kubernetes Automation:**
- **GitOps Workflow**: Commit tenant config → ArgoCD applies manifests
- **Crossplane Compositions**: Declarative infrastructure provisioning
- **Argo Workflows**: Complex multi-step provisioning orchestration
### 3. Tenant-Aware Authentication

**User Need:** "I want users to authenticate and be automatically routed to their tenant context"

**open-sbt Pattern:** Use **IAuth** interface with **Multi-Tenant Microservice Libraries**

**Implementation Approach:**
```go
// Use open-sbt's IAuth abstraction
auth := ory.NewOryAuth(ory.Config{
    KratosURL: "http://kratos:4433",
    HydraURL:  "http://hydra:4444",
})

// Use Identity Token Manager library
tokenManager := tenantcontext.NewIdentityTokenManager(
    auth.GetWellKnownEndpoint(),
    auth.GetJWTIssuer(),
    auth.GetJWTAudience()[0],
)

// Gin middleware for automatic tenant context
r.Use(tokenManager.GinMiddleware())
```

**Kubernetes Authentication Patterns:**
- **JWT-based Authentication**: Ory Hydra issues tenant-scoped JWTs
- **Ingress-level Routing**: Route by subdomain or path to tenant namespaces
- **Service Mesh Integration**: Istio/Linkerd for fine-grained auth policies

### 4. Per-Tenant Data Isolation

**User Need:** "I want each tenant's data completely isolated from other tenants"

**open-sbt Pattern:** Use **IStorage** interface with **Database Isolation Helper**

**Implementation Approach:**
```go
// Use open-sbt's IStorage abstraction
storage := postgres.NewPostgresStorage(postgres.Config{
    Host:     "postgres",
    Database: "opensbt",
})

// Use Database Isolation Helper library
tenantDB := tenantdb.NewTenantDB(db)

// Automatic RLS context setting
rows, err := tenantDB.QueryContext(ctx, "SELECT * FROM orders")
// → Automatically sets app.tenant_id for RLS
```

**Kubernetes Data Patterns:**
- **Database per Tenant**: Separate PostgreSQL instances (premium tiers)
- **Schema per Tenant**: Shared database, separate schemas
- **Row-Level Security (RLS)**: Shared tables with tenant_id filtering
- **Tenant-Scoped Secrets**: Kubernetes secrets per namespace

### 5. Tier-Based Resource Allocation

**User Need:** "I want different resource limits and features based on customer tiers"

**open-sbt Pattern:** Use **Tier as Tenant Attribute** with **Provisioning Logic**

**Implementation Approach:**
```go
// Tier-based provisioning via IProvisioner — commits AINativeSaaS CR to Git.
// Tier-specific resource expansion is handled by Crossplane Compositions, not Go code:
//   spec.tier: starter    → Crossplane Composition A (shared pool, namespace isolation)
//   spec.tier: enterprise → Crossplane Composition B (dedicated Spoke Silo cluster, BYOC)
// ProvisionTenant commits to Git and returns immediately — it never blocks on infrastructure.
func (p *CrossplaneProvisioner) ProvisionTenant(ctx context.Context, req ProvisionRequest) (*ProvisionResult, error) {
    cr := buildAINativeSaasCR(req.TenantID, req.Tier, req.Region, req.Cloud)
    return p.gitClient.CommitManifest(ctx, req.TenantID, cr)
    // Crossplane reconciles the correct Composition based on spec.tier.
    // Status is reported asynchronously by the Spoke Controller via Hub-side PostgREST.
}
```

**Kubernetes Tier Patterns:**
- **Resource Quotas**: CPU/memory limits per tier
- **Storage Classes**: Different storage performance per tier
- **Replica Counts**: High availability for premium tiers
- **Feature Gates**: Enable/disable features via ConfigMaps
### 6. Tenant-Aware Observability

**User Need:** "I want monitoring and logging that shows per-tenant metrics and isolates tenant data"

**open-sbt Pattern:** Use **Logging Manager** and **Metrics Manager** libraries

**Implementation Approach:**
```go
// Use open-sbt's observability libraries
logger := tenantlogging.NewTenantLogger()
metrics := tenantmetrics.NewTenantMetrics("my_service")

// Automatic tenant context in logs and metrics
r.Use(metrics.GinMiddleware())  // Auto-tag metrics with tenant_id

// In handlers
logger.Info(ctx, "Order created", map[string]interface{}{
    "order_id": orderID,
    "amount":   amount,
})
// → Automatically includes tenant_id, tenant_tier, user_id
```

**Kubernetes Observability Patterns:**
- **VictoriaMetrics**: Tenant-tagged metrics — written by Grafana Alloy remote_write pipeline
- **Loki**: Log aggregation — written by Grafana Alloy, queried via Grafana (Grafana-native pipeline)
- **OpenSearch**: Event search and diagnostics — k8s-events, K8sGPT findings, infra-changes,
  ArgoCD sync events. Not used for general log aggregation — Loki handles that.
- **Grafana Dashboards**: Per-tenant and global views across VictoriaMetrics and Loki
- **Alertmanager**: Tenant-specific alerting rules (driven by VictoriaMetrics vmalert)

### 7. Multi-Tenant CI/CD

**User Need:** "I want to deploy application updates across all tenants safely with rollback capabilities"

**open-sbt Pattern:** Use **GitOps-First** approach with **ArgoCD + Argo Workflows**

**Implementation Approach:**
```go
// open-sbt handles GitOps through IProvisioner
provisioner.UpdateTenantResources(ctx, opensbt.UpdateRequest{
    TenantID: "tenant-123",
    Updates: []opensbt.ResourceUpdate{
        {Type: "deployment", Name: "app", Image: "myapp:v2.1.0"},
    },
})
// → Commits to Git → ArgoCD applies → Argo Workflows orchestrates rollout
```

**Kubernetes CI/CD Patterns:**
- **ArgoCD Applications**: One Application per tenant namespace
- **Argo Workflows**: Complex deployment orchestration (blue/green, canary)
- **Kustomize Overlays**: Tenant-specific configuration management
- **OCI Artifacts**: Versioned service catalog packages

### 8. Tenant Self-Service Portal

**User Need:** "I want tenants to manage their own users, settings, and view their usage"

**open-sbt Pattern:** Use **Dual Admin Console Architecture** with **MCP Tools**

**Implementation Approach:**
```go
// Tenant Admin Console calls Control Plane via MCP
mcpServer := mcp.NewControlPlaneMCPServer(controlPlane)

// Tenant-scoped MCP tools
tools := []mcp.Tool{
    {Name: "list_tenant_users", Handler: s.handleListTenantUsers},
    {Name: "create_tenant_user", Handler: s.handleCreateTenantUser},
    {Name: "get_tenant_config", Handler: s.handleGetTenantConfig},
    {Name: "get_tenant_usage", Handler: s.handleGetTenantUsage},
}
```

**Kubernetes Self-Service Patterns:**
- **Tenant Admin RBAC**: Limited permissions within tenant namespace
- **Custom Resources**: Tenant-manageable configuration objects
- **Admission Controllers**: Validate tenant self-service operations
- **Resource Quotas**: Prevent tenant resource abuse

### 9. Secure Tenant Workload Execution

**User Need:** "I want to run tenant code safely without compromising the platform or other tenants"

**open-sbt Pattern:** Use **AgentSandbox Sandboxing** with **Agent Execution**

**Implementation Approach:**
```go
// open-sbt's agent execution with AgentSandbox
executor := agent.NewAgentExecutor(agent.Config{
    Runtime: "agentsandbox",  // Sandboxed execution
})

// Execute tenant agent safely
result, err := executor.ExecuteAgent(ctx, agent.ExecutionRequest{
    TenantID:   "tenant-123",
    AgentImage: "tenant-agent:latest",
    Resources:  getResourceLimits("premium"),
})
```

**Kubernetes Security Patterns:**
- **AgentSandbox Runtime**: Kernel-level isolation for tenant workloads
- **Pod Security Standards**: Enforce security policies per namespace
- **Network Policies**: Micro-segmentation between tenant services
- **Service Mesh**: mTLS and fine-grained access control

### 10. Cost Attribution and Billing

**User Need:** "I want to track resource usage per tenant for billing and cost optimization"

**open-sbt Pattern:** Use **IBilling** and **IMetering** interfaces

**Implementation Approach:**
```go
// Use open-sbt's billing abstractions
billing := mockbilling.NewMockBilling()  // or real billing provider
metering := metering.NewMetering(metering.Config{
    Storage: storage,
})

// Automatic usage tracking
metering.IngestUsageEvent(ctx, metering.UsageEvent{
    TenantID:     "tenant-123",
    MeterName:    "api_requests",
    Value:        1,
    Timestamp:    time.Now(),
})
```

**Kubernetes Cost Patterns:**
- **Resource Quotas**: Track CPU/memory/storage usage per tenant
- **Custom Metrics**: Application-level usage (API calls, storage, etc.)
- **Cost Allocation**: Kubernetes cost monitoring tools (KubeCost)
- **Usage Reports**: Automated billing data export

## Implementation Decision Matrix

| User Need | Isolation Level | open-sbt Pattern | K8s Pattern | Complexity |
|-----------|----------------|------------------|-------------|------------|
| **Basic Multi-Tenancy** | Namespace | IProvisioner | Namespace-per-tenant | Low |
| **High Security** | Pod-level | AgentSandbox + IProvisioner | RuntimeClass + NetworkPolicy | High |
| **Cost Optimization** | Resource-based | IMetering + Quotas | ResourceQuota + Monitoring | Medium |
| **Self-Service** | RBAC-based | MCP Tools + IAuth | Custom RBAC + CRDs | Medium |
| **Enterprise Scale** | Full Isolation | All Patterns | Multi-cluster + GitOps | High |

## Quick Start Recommendations

### For New SaaS Applications:
1. Start with **Namespace-per-tenant** isolation
2. Use **open-sbt Control Plane** for tenant management
3. Implement **Multi-tenant Microservice Libraries** for automatic context
4. Add **GitOps workflows** for deployment automation

### For Existing Applications:
1. Implement **IAuth interface** for authentication abstraction
2. Add **tenant context middleware** to existing APIs
3. Migrate to **PostgreSQL RLS** for data isolation
4. Gradually adopt **open-sbt provisioning patterns**

### For High-Security Requirements:
1. Use **AgentSandbox runtime** for workload isolation
2. Implement **Network Policies** for network segmentation
3. Add **Pod Security Standards** enforcement
4. Consider **separate clusters** for sensitive tenants

## Anti-Patterns to Avoid

❌ **Shared Database without RLS**: Leads to data leakage risks
❌ **Manual Tenant Provisioning**: Doesn't scale, error-prone
❌ **Hard-coded Tenant Logic**: Makes adding new tenants difficult
❌ **No Resource Limits**: Allows tenant resource abuse
❌ **Shared Secrets**: Compromises tenant isolation
❌ **Direct Kubernetes API Access**: Bypasses GitOps and audit trails

## Success Metrics

- **Tenant Onboarding Time**: < 5 minutes automated
- **Resource Utilization**: > 70% cluster efficiency
- **Security Incidents**: Zero cross-tenant data access
- **Deployment Frequency**: Multiple deployments per day
- **Mean Time to Recovery**: < 15 minutes for tenant issues