---
inclusion: always
---
<!------------------------------------------------------------------------------------
   Add rules to this file or a short description and have Kiro refine them for you.
   
   Learn about inclusion modes: https://kiro.dev/docs/steering/#inclusion-modes
-------------------------------------------------------------------------------------> 

# open-sbt Design Principles

**Version:** 1.0  
**Date:** 2026-03-16  
**Based on:** AWS SaaS Builder Toolkit (SBT-AWS) Architecture

## Executive Summary

open-sbt is a Go-based SaaS builder toolkit for the zero-ops platform, inspired by AWS SBT-AWS. It provides reusable abstractions for building multi-tenant SaaS applications with a clear separation between Control Plane and Application Plane.

**Key Differences from SBT-AWS:**
- Language: Go (not TypeScript/CDK)
- Event Bus: NATS (not AWS EventBridge)
- Auth: Ory Stack (not AWS Cognito)
- Provisioning: Crossplane + ArgoCD GitOps (not CloudFormation + CodeBuild)
- Database: PostgreSQL with sqlc for Go APIs and PostgREST for dashboards (not DynamoDB)
- API: Gin framework (not API Gateway + Lambda)

## Core Architecture Principles

### 1. Control Plane + Application Plane Separation

**Principle:** Maintain clear boundaries between tenant management (Control Plane) and tenant workloads (Application Plane).

**Control Plane Responsibilities:**
- Tenant lifecycle management (CRUD operations)
- Tenant registration and onboarding workflows
- User management per tenant
- Tenant configuration storage
- Authentication and authorization
- Billing and metering (optional)
- Event orchestration

**Application Plane Responsibilities:**
- Tenant workload execution within the Spoke cluster
- Responding to Control Plane events (non-infra coordination)
- Tenant-specific runtime management within the Spoke


**Communication Pattern:**
```
Infrastructure Provisioning (95%): Hub Tenant Management API → Git Commit → ArgoCD → Crossplane → Spoke Cluster → Status Controller → PostgreSQL
Coordination/Orchestration (5%): Control Plane → NATS Event → Application Plane (cross-cluster, non-Git actions)
```

### 2. Interface-Based Abstraction

**Principle:** Define stable Go interfaces for all pluggable components to allow implementation swapping without breaking user code.

**Core Interfaces:**

```go
// IAuth - Authentication and authorization provider
type IAuth interface {
    // User management
    CreateUser(ctx context.Context, user User) error
    GetUser(ctx context.Context, userID string) (*User, error)
    UpdateUser(ctx context.Context, userID string, updates UserUpdates) error
    DeleteUser(ctx context.Context, userID string) error
    DisableUser(ctx context.Context, userID string) error
    EnableUser(ctx context.Context, userID string) error
    ListUsers(ctx context.Context, filters UserFilters) ([]User, error)
    
    // Authentication
    AuthenticateUser(ctx context.Context, credentials Credentials) (*Token, error)
    ValidateToken(ctx context.Context, token string) (*Claims, error)
    RefreshToken(ctx context.Context, refreshToken string) (*Token, error)
    
    // Admin user creation
    CreateAdminUser(ctx context.Context, props CreateAdminUserProps) error
    
    // Token configuration
    GetJWTIssuer() string
    GetJWTAudience() []string
    GetTokenEndpoint() string
    GetWellKnownEndpoint() string
}
```


```go
// IEventBus - Message bus for Control Plane ↔ Application Plane communication
type IEventBus interface {
    // Event publishing
    Publish(ctx context.Context, event Event) error
    PublishAsync(ctx context.Context, event Event) error
    
    // Event subscription
    Subscribe(ctx context.Context, eventType string, handler EventHandler) error
    SubscribeQueue(ctx context.Context, eventType string, queueGroup string, handler EventHandler) error
    
    // Event definitions
    GetControlPlaneEventSource() string
    GetApplicationPlaneEventSource() string
    CreateControlPlaneEvent(detailType string) EventDefinition
    CreateApplicationPlaneEvent(detailType string) EventDefinition
    CreateCustomEvent(detailType string, source string) EventDefinition
    
    // Standard events
    GetStandardEvents() map[string]EventDefinition
    
    // Permissions
    GrantPublishPermissions(grantee string) error
}

// IProvisioner - Tenant resource provisioning
// ProvisionTenant commits an AINativeSaaS CR to the tenant's Git control plane repo
// and returns immediately. It does NOT block on infrastructure readiness.
// Status is reported asynchronously by the Spoke Controller (controller-runtime),
// which watches Crossplane claim conditions and writes to Hub Centralised DB via PostgREST.
// Callers MUST read status from the DB — never poll the provisioner.
type IProvisioner interface {
    // Provision tenant resources — commits to Git, returns 202-equivalent immediately
    ProvisionTenant(ctx context.Context, req ProvisionRequest) (*ProvisionResult, error)
    
    // Deprovision tenant resources — commits deletion to Git, Crossplane handles teardown
    DeprovisionTenant(ctx context.Context, req DeprovisionRequest) (*DeprovisionResult, error)
    
    // Update tenant resources
    UpdateTenantResources(ctx context.Context, req UpdateRequest) (*UpdateResult, error)
    
    // NOTE: GetProvisioningStatus is intentionally absent.
    // Status is written by the Spoke Controller to Hub Centralised DB and read from there.
}
```


```go
// IStorage - Data persistence layer
type IStorage interface {
    // Tenant management
    CreateTenant(ctx context.Context, tenant Tenant) error
    GetTenant(ctx context.Context, tenantID string) (*Tenant, error)
    UpdateTenant(ctx context.Context, tenantID string, updates TenantUpdates) error
    DeleteTenant(ctx context.Context, tenantID string) error
    ListTenants(ctx context.Context, filters TenantFilters) ([]Tenant, error)
    
    // Tenant registration
    CreateTenantRegistration(ctx context.Context, reg TenantRegistration) error
    GetTenantRegistration(ctx context.Context, regID string) (*TenantRegistration, error)
    UpdateTenantRegistration(ctx context.Context, regID string, updates RegistrationUpdates) error
    DeleteTenantRegistration(ctx context.Context, regID string) error
    ListTenantRegistrations(ctx context.Context, filters RegistrationFilters) ([]TenantRegistration, error)
    
    // Tenant configuration
    SetTenantConfig(ctx context.Context, tenantID string, config map[string]interface{}) error
    GetTenantConfig(ctx context.Context, tenantID string) (map[string]interface{}, error)
    DeleteTenantConfig(ctx context.Context, tenantID string) error
}

// IBilling - Billing integration (optional)
type IBilling interface {
    // Customer management
    CreateCustomer(ctx context.Context, customer BillingCustomer) error
    DeleteCustomer(ctx context.Context, customerID string) error
    
    // Usage tracking
    RecordUsage(ctx context.Context, usage UsageRecord) error
    GetUsage(ctx context.Context, customerID string, period TimePeriod) (*UsageReport, error)
    
    // Webhook handling
    HandleWebhook(ctx context.Context, payload []byte) error
}
```


```go
// IMetering - Usage metering (optional)
type IMetering interface {
    // Meter management
    CreateMeter(ctx context.Context, meter Meter) error
    GetMeter(ctx context.Context, meterID string) (*Meter, error)
    UpdateMeter(ctx context.Context, meterID string, updates MeterUpdates) error
    DeleteMeter(ctx context.Context, meterID string) error
    ListMeters(ctx context.Context, filters MeterFilters) ([]Meter, error)
    
    // Usage ingestion
    IngestUsageEvent(ctx context.Context, event UsageEvent) error
    
    // Usage queries
    GetUsage(ctx context.Context, meterID string, period TimePeriod) (*UsageData, error)
    CancelUsageEvents(ctx context.Context, eventIDs []string) error
}
```

**Implementation Strategy:**
1. Define interfaces in `pkg/interfaces/` package
2. Provide default implementations in `pkg/providers/` package
3. Allow users to provide custom implementations
4. Use dependency injection for flexibility

**Example Usage:**
```go
// User can choose implementation
auth := ory.NewOryAuth(oryConfig)
// OR
auth := keycloak.NewKeycloakAuth(keycloakConfig)
// OR
auth := custom.NewCustomAuth(customConfig)

// Control plane works with any IAuth implementation
controlPlane := opensbt.NewControlPlane(opensbt.ControlPlaneConfig{
    Auth: auth,  // Interface, not concrete type
    EventBus: natsEventBus,
    Storage: postgresStorage,
})
```


### 3. Event-Driven Communication

**Principle:** NATS is strictly for coordination (5%) such as cross-cluster execution and user-initiated restarts. Infrastructure provisioning and status updates (95%) are strictly handled via GitOps and the Status Controller updating PostgreSQL.

**Standard Events:**

**Control Plane Events** (source: `opensbt.control.plane`):
- `opensbt_tenantCreated` - Tenant created (for billing/notifications)
- `opensbt_offboardingRequest` - Tenant offboarding initiated
- `opensbt_activateRequest` - Tenant activation requested
- `opensbt_deactivateRequest` - Tenant deactivation requested
- `opensbt_tenantUserCreated` - User created in tenant
- `opensbt_tenantUserDeleted` - User deleted from tenant
- `opensbt_billingSuccess` - Billing operation succeeded
- `opensbt_billingFailure` - Billing operation failed

**Application Plane Events** (source: `opensbt.application.plane`):
- `opensbt_onboardingSuccess` - Tenant onboarded successfully
- `opensbt_onboardingFailure` - Tenant onboarding failed
- `opensbt_offboardingSuccess` - Tenant offboarded successfully
- `opensbt_offboardingFailure` - Tenant offboarding failed
- `opensbt_provisionSuccess` - Resources provisioned successfully
- `opensbt_provisionFailure` - Resource provisioning failed
- `opensbt_deprovisionSuccess` - Resources deprovisioned successfully
- `opensbt_deprovisionFailure` - Resource deprovisioning failed
- `opensbt_activateSuccess` - Tenant activated successfully
- `opensbt_activateFailure` - Tenant activation failed
- `opensbt_deactivateSuccess` - Tenant deactivated successfully
- `opensbt_deactivateFailure` - Tenant deactivation failed
- `opensbt_ingestUsage` - Usage data ingested


**Event Structure:**
```go
type Event struct {
    ID          string                 `json:"id"`
    Version     string                 `json:"version"`
    DetailType  string                 `json:"detailType"`
    Source      string                 `json:"source"`
    Time        time.Time              `json:"time"`
    Region      string                 `json:"region,omitempty"`
    Resources   []string               `json:"resources,omitempty"`
    Detail      map[string]interface{} `json:"detail"`
}

// Example: Tenant Created Event
{
    "id": "6a7e8feb-b491-4cf7-a9f1-bf3703467718",
    "version": "1.0",
    "detailType": "opensbt_tenantCreated",
    "source": "opensbt.control.plane",
    "time": "2026-03-16T18:43:48Z",
    "detail": {
        "tenantId": "e6878e03-ae2c-43ed-a863-08314487318b",
        "tier": "premium",
        "name": "acme-corp",
        "email": "admin@acme.com"
    }
}
```

**Event Flow Example:**
```
1. Hub Tenant Management API: create_tenant received
2. Hub Tenant Management API: Generates Helm values manifest and commits new tenant folder to central GitOps repo
3. Hub Tenant Management API: Returns 202 Accepted to caller (async provisioning begins)
4. ArgoCD (Hub): Detects Git commit and syncs manifests
5. Crossplane: Reconciles infrastructure (CAPI, CNPG, etc.) — provisions Spoke cluster
6. Spoke: Control Plane + Application Plane come up inside the Spoke
7. Spoke Controller detects Crossplane claim Ready=True → writes status "ready" to Hub Centralised DB
   via Hub-side PostgREST (Bearer JWT, RLS enforced)
8. Spoke Controller publishes spoke.{tenant}.lifecycle.created to NATS Leaf Node
9. Hub Event Router consumes event → triggers non-infra side effects (billing, notifications)
```


### 4. Tenant Lifecycle Management

**Principle:** Provide standardized workflows for tenant onboarding, management, and offboarding with optimized latency paths.

**Tenant States:**
- `pending` - Registration created, awaiting provisioning
- `provisioning` - Resources being provisioned
- `active` - Tenant fully operational
- `suspended` - Tenant temporarily disabled
- `deprovisioning` - Resources being removed
- `deleted` - Tenant removed

**Warm Pool Pattern for Low-Latency Onboarding:**
The zero-ops platform implements a dual-path onboarding strategy to optimize for different tenant tiers:

**Path A: Shared Pool Tiers (Basic/Standard) - Latency < 2 Seconds**
1. Hub Tenant Management API maintains pre-provisioned "warm" tenant slots (warm-pool-01, warm-pool-02, etc.) in Git
2. Hub Tenant Management API instantly claims an available warm slot for the new tenant in PostgreSQL
3. Ory Keto creates tenant authorization relationships immediately
4. API returns success in < 2 seconds
5. Async refill: Hub Tenant Management API updates the claimed slot's Git configuration and commits a new warm slot folder to replace it

**Path B: Dedicated Tiers (Premium/Enterprise) - Latency 2-5 Minutes**
1. API returns 202 Accepted immediately
2. Hub Tenant Management API generates the Helm values manifest and commits a new `tenants/<tenant-id>` folder to the central GitOps repo
3. ArgoCD syncs Helm chart with Crossplane XRs — provisions a dedicated Spoke cluster
4. Each Spoke gets its own Control Plane + Application Plane deployed inside it
5. Spoke Controller watches Crossplane claim conditions and writes status updates to Hub Centralised DB via Hub-side PostgREST (Bearer JWT, RLS enforced)

**Standard Tenant Registration Workflow:**
```
1. Hub Tenant Management API receives request
   → Inserts 'pending' record in PostgreSQL
   → Generates manifests and commits new tenant folder to central GitOps repo

2. ArgoCD (Hub) & Crossplane
   → Syncs and provisions Spoke infrastructure automatically

3. Spoke Controller (controller-runtime)
   → Watches Crossplane claim conditions (Ready=True)
   → Writes status updates to Hub Centralised DB via Hub-side PostgREST
   → Uses Bearer JWT with RLS enforcement for secure writes
```

**Tenant Offboarding Workflow:**
```
1. DELETE /tenants/{tenantId}
   → Updates tenant (status: deprovisioning)
   → Commits deletion to Git (removes tenant folder OR marks warm pool slot as available)
   
2. ArgoCD & Crossplane
   → Detects Git change and reconciles deletion
   → Deprovisions resources (returns to warm pool OR destroys dedicated resources)
   
3. Spoke Controller
   → Watches Crossplane claim deletion and writes final status to Hub Centralised DB via PostgREST
   → Publishes opensbt_deprovisionSuccess event to NATS for non-infra side effects
   
4. Control Plane receives success event
   → Deletes tenant record
   → Deletes registration record
   → Returns success to caller
```


### 5. Provisioning Abstraction

**Principle:** Abstract tenant resource provisioning using the Crossplane + ArgoCD GitOps pattern.

**IMPORTANT:** The Zero-Ops Platform specifically uses the **Crossplane + ArgoCD GitOps implementation** for all provisioning operations. This follows the established pattern: **Hub API → Commits to Git → ArgoCD ApplicationSet → Crossplane XRs → Infrastructure**. Argo Workflows is NOT used in the provisioning engine to maintain simplicity and operational efficiency.

**Zero-Ops Platform Implementation:**

The Zero-Ops Platform strictly implements the **Crossplane GitOps Provisioner** following the established pattern:
**Hub API → Commits to Git → ArgoCD ApplicationSet → Crossplane XRs → Infrastructure**

**Crossplane GitOps Provisioner:**
```go
// CrossplaneProvisioner is called by the Hub Tenant Management API (Control Plane).
// It generates the Crossplane manifest and commits it to the central GitOps repo.
// ArgoCD on the Hub picks it up and provisions the Spoke cluster.
type CrossplaneProvisioner struct {
    gitClient GitClient
    config    CrossplaneConfig
}

func (p *CrossplaneProvisioner) ProvisionTenant(ctx context.Context, req ProvisionRequest) (*ProvisionResult, error) {
    // Build the Crossplane Composition manifest
    composition := &compositionv1.Composition{
        ObjectMeta: metav1.ObjectMeta{
            Name: fmt.Sprintf("tenant-%s", req.TenantID),
        },
        Spec: compositionv1.CompositionSpec{
            Resources: []compositionv1.ComposedTemplate{
                {Resource: createNamespaceTemplate(req)},
                {Resource: createDatabaseTemplate(req)},
                {Resource: createS3BucketTemplate(req)},
            },
        },
    }
    
    // Hub Tenant Management API commits manifest to central GitOps repo
    return p.gitClient.CommitManifest(ctx, req.TenantID, composition)
}
```

**Provisioning Request Structure:**
```go
type ProvisionRequest struct {
    TenantID    string                 `json:"tenantId"`
    Tier        string                 `json:"tier"`
    Name        string                 `json:"name"`
    Email       string                 `json:"email"`
    Config      map[string]interface{} `json:"config"`
    Resources   []ResourceSpec         `json:"resources"`
}

type ResourceSpec struct {
    Type       string                 `json:"type"`  // namespace, database, s3bucket, etc.
    Name       string                 `json:"name"`
    Parameters map[string]interface{} `json:"parameters"`
}
```


### 6. GitOps-First Approach

**Principle:** All infrastructure changes must go through Git, leveraging ArgoCD for continuous deployment. The zero-ops platform explicitly rejects custom Kubernetes operators for tenant resource generation to reduce operational complexity.

**GitOps Workflow:**
```
1. Hub Tenant Management API receives create_tenant request
2. Hub Tenant Management API generates Helm values manifest and commits new tenant folder to central GitOps repo
3. ArgoCD ApplicationSet (Hub) detects Git change via Directory Generator
4. ArgoCD applies Universal Tenant Helm Chart → provisions Spoke cluster
5. Each Spoke gets its own Control Plane + Application Plane deployed inside it
6. Crossplane provisions Spoke infrastructure based on XRs
7. ArgoCD reports sync status
8. Status Controller updates tenant status in PostgreSQL to 'ready'
9. Hub Tenant Management API publishes opensbt_provisionSuccess to NATS for non-infra side effects
```

**Key Architectural Decision:**
Instead of using Red Hat COP operators (namespace-configuration-operator, gitops-generator), the zero-ops platform uses a simplified approach:
- **Helm templating** for resource generation
- **Crossplane compositions** for infrastructure provisioning  
- **ArgoCD ApplicationSets** for deployment orchestration

This approach reduces operational complexity while maintaining the same functionality.

**Repository Structure:**
```
gitops-repo/
├── base-charts/
│   └── tenant-factory/         # Universal Helm chart for all tenants
│       ├── Chart.yaml
│       ├── values.yaml         # Default values
│       └── templates/
│           ├── namespace.yaml
│           ├── resourcequota.yaml
│           ├── networkpolicy.yaml
│           ├── rolebinding.yaml
│           └── crossplane-xr.yaml  # Crossplane CompositeResources
├── tenants/
│   ├── warm-pool-01/           # Pre-provisioned unassigned tenant
│   │   └── values.yaml         # Helm values: tier=basic, assigned=false
│   ├── warm-pool-02/           # Pre-provisioned unassigned tenant
│   │   └── values.yaml
│   ├── acme-corp-123/          # Active assigned tenant
│   │   └── values.yaml         # Helm values: tier=premium, assigned=true
│   └── stark-ind-456/          # Enterprise BYOC tenant
│       └── values.yaml         # Helm values: tier=enterprise, byoc=true
└── applicationset.yaml         # ArgoCD ApplicationSet with Git Directory Generator
```

**ArgoCD ApplicationSet Template:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: zero-ops-tenant-factory
spec:
  generators:
  - git:
      repoURL: https://github.com/zero-ops/gitops-repo
      revision: HEAD
      directories:
      - path: tenants/*
  template:
    metadata:
      name: 'tenant-{{path.basename}}'
    spec:
      project: tenants
      source:
        repoURL: https://github.com/zero-ops/gitops-repo
        targetRevision: HEAD
        path: base-charts/tenant-factory
        helm:
          valueFiles:
          - ../../tenants/{{path.basename}}/values.yaml
      destination:
        server: https://kubernetes.default.svc
        namespace: 'tenant-{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```


### 7. MCP-First Integration

**Principle:** All Control Plane capabilities must be exposed via MCP tools for agent consumption.

**MCP Tool Structure:**
```go
// Control Plane MCP Tools
type ControlPlaneMCPServer struct {
    controlPlane *ControlPlane
}

func (s *ControlPlaneMCPServer) GetTools() []mcp.Tool {
    return []mcp.Tool{
        {
            Name:        "create_tenant",
            Description: "Create a new tenant in the platform",
            InputSchema: createTenantSchema,
            Handler:     s.handleCreateTenant,
        },
        {
            Name:        "get_tenant",
            Description: "Retrieve tenant details",
            InputSchema: getTenantSchema,
            Handler:     s.handleGetTenant,
        },
        {
            Name:        "update_tenant",
            Description: "Update tenant configuration",
            InputSchema: updateTenantSchema,
            Handler:     s.handleUpdateTenant,
        },
        {
            Name:        "delete_tenant",
            Description: "Remove a tenant",
            InputSchema: deleteTenantSchema,
            Handler:     s.handleDeleteTenant,
        },
        {
            Name:        "list_tenants",
            Description: "List all tenants",
            InputSchema: listTenantsSchema,
            Handler:     s.handleListTenants,
        },
    }
}
```

**MCP Tool Implementation Example:**
```go
func (s *ControlPlaneMCPServer) handleCreateTenant(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    // Extract tenant context from JWT (already in ctx)
    tenantID := ctx.Value("tenant_id").(string)
    
    // Parse parameters
    name := params["name"].(string)
    tier := params["tier"].(string)
    email := params["email"].(string)
    
    // Create tenant via Control Plane
    tenant, err := s.controlPlane.CreateTenant(ctx, CreateTenantRequest{
        Name:  name,
        Tier:  tier,
        Email: email,
    })
    if err != nil {
        return nil, err
    }
    
    return map[string]interface{}{
        "tenant_id":  tenant.ID,
        "name":       tenant.Name,
        "tier":       tenant.Tier,
        "status":     tenant.Status,
        "created_at": tenant.CreatedAt,
    }, nil
}
```


### 8. Multi-Tenant Security

**Principle:** Implement defense-in-depth security with tenant isolation at every layer.

**Security Layers:**

**1. API Layer (Gin + JWT)**
```go
func TenantContextMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // Extract JWT claims
        claims := jwt.ExtractClaims(c)
        tenantID := claims["tenant_id"].(string)
        tenantTier := claims["tenant_tier"].(string)
        
        // Add to context
        ctx := context.WithValue(c.Request.Context(), "tenant_id", tenantID)
        ctx = context.WithValue(ctx, "tenant_tier", tenantTier)
        c.Request = c.Request.WithContext(ctx)
        
        c.Next()
    }
}
```

**2. Authorization Layer (Ory Keto)**
```go
func CheckTenantPermission(ctx context.Context, keto *keto.Client, tenantID, permission string) error {
    userID := ctx.Value("user_id").(string)
    
    allowed, err := keto.Check(ctx, &ketoapi.RelationQuery{
        Namespace: "tenants",
        Object:    tenantID,
        Relation:  permission,
        Subject:   &ketoapi.Subject{ID: userID},
    })
    
    if err != nil || !allowed {
        return ErrUnauthorized
    }
    
    return nil
}
```

**3. Database Layer (PostgreSQL RLS)**
```sql
-- Enable RLS on all tenant tables
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenant_users ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenant_configs ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only access their tenant's data
CREATE POLICY tenant_isolation ON tenants
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

CREATE POLICY tenant_user_isolation ON tenant_users
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Set tenant context before queries
SET app.tenant_id = 'tenant-uuid-here';
```


**4. Kubernetes Layer (Namespace Isolation)**
```yaml
# Namespace per tenant (for premium/enterprise tiers)
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-{{.TenantID}}
  labels:
    tenant-id: {{.TenantID}}
    tenant-tier: {{.Tier}}

---
# Network Policy: Deny cross-tenant traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-tenant
  namespace: tenant-{{.TenantID}}
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant-id: {{.TenantID}}
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tenant-id: {{.TenantID}}
```

**5. Agent Execution Layer (AgentSandbox)**
```go
// Agent execution with AgentSandbox sandboxing
type AgentExecutor struct {
    runtime string // "runsc" for AgentSandbox
}

func (e *AgentExecutor) ExecuteAgent(ctx context.Context, req AgentExecutionRequest) error {
    // Create pod with AgentSandbox runtime
    pod := &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("agent-%s", req.AgentID),
            Namespace: fmt.Sprintf("tenant-%s", req.TenantID),
            Annotations: map[string]string{
                "io.kubernetes.cri.untrusted-workload": "true",
            },
        },
        Spec: corev1.PodSpec{
            RuntimeClassName: &e.runtime, // "agentsandbox"
            Containers: []corev1.Container{
                {
                    Name:  "agent",
                    Image: req.AgentImage,
                    // Resource limits, security context, etc.
                },
            },
        },
    }
    
    return e.k8sClient.Create(ctx, pod)
}
```


### 9. Observability and Monitoring

**Principle:** Provide tenant-aware observability with global and per-tenant views.

**Metrics Collection:**
```go
// Tenant-aware metrics
var (
    tenantRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "opensbt_tenant_request_duration_seconds",
            Help: "Duration of tenant requests",
        },
        []string{"tenant_id", "tier", "method", "path", "status"},
    )
    
    tenantResourceUsage = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "opensbt_tenant_resource_usage",
            Help: "Tenant resource usage",
        },
        []string{"tenant_id", "tier", "resource_type"},
    )
)

// Record metrics
func RecordTenantRequest(tenantID, tier, method, path string, status int, duration time.Duration) {
    tenantRequestDuration.WithLabelValues(
        tenantID,
        tier,
        method,
        path,
        strconv.Itoa(status),
    ).Observe(duration.Seconds())
}
```

**Logging Pattern:**
```go
// Tenant-aware logging
log.WithFields(log.Fields{
    "tenant_id":   tenantID,
    "tenant_tier": tier,
    "user_id":     userID,
    "action":      "create_tenant",
    "resource_id": resourceID,
}).Info("Tenant created successfully")
```

**Monitoring Dashboards:**
- Global dashboard: Overall platform health
- Tenant dashboard: Per-tenant metrics and health
- Tier dashboard: Metrics aggregated by tier
- Cost dashboard: Per-tenant cost attribution


### 10. Database Architecture Pattern

**Principle:** Use a layered database access pattern with sqlc for Go APIs and PostgREST for dashboards and reporting.

**Architecture Overview:**
```
┌─────────────────────────────────────────────────────────────┐
│                    Go Applications                           │
│  (Control Plane, Application Plane, MCP Servers)            │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Control Plane│  │   App Plane  │  │ MCP Servers  │     │
│  │   (sqlc)     │  │   (sqlc)     │  │   (sqlc)     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────┬───────────────────────────────────┘
                          │ Type-safe SQL queries
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    PostgreSQL Database                       │
│  - Row-Level Security (RLS) for tenant isolation            │
│  - CNPG operator for high availability                      │
│  - pgvector for AI similarity search                        │
│  - Composite indexes on (tenant_id, ...)                   │
└─────────────────────────┬───────────────────────────────────┘
                          │ Auto-generated REST API
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      PostgREST                              │
│  - Auto-generated REST endpoints                            │
│  - Used for dashboards and reporting                        │
│  - Respects PostgreSQL RLS policies                         │
│  - Read-only access for most use cases                      │
└─────────────────────────────────────────────────────────────┘
```

**sqlc Usage (Primary Go APIs):**
- **Purpose**: Type-safe database access for all Go applications
- **Benefits**: Compile-time SQL validation, type safety, performance
- **Use Cases**: Control Plane APIs, Application Plane logic, MCP tools
- **Pattern**: Write SQL queries, generate Go code, use in applications

**PostgREST Usage (Dashboards & Reporting):**
- **Purpose**: Auto-generated REST API for frontend applications and Spoke Controller writes
- **Benefits**: No backend code needed, automatic API generation, RLS support
- **Use Cases**: 
  - Admin dashboards and tenant portals (read operations)
  - Reporting interfaces (read operations)
  - Spoke Controller status writes to Hub Centralised DB (write operations via Hub-side PostgREST)
- **Pattern**: Define database schema, PostgREST exposes REST endpoints
- **Deployment**: Two instances - Hub-side (accepts writes from Spoke Controllers) and Spoke-side (operational data, read-only)

**Example Implementation:**
```go
// sqlc usage in Go applications
func (s *TenantService) GetTenant(ctx context.Context, tenantID uuid.UUID) (*Tenant, error) {
    // Type-safe query generated by sqlc
    return s.queries.GetTenantByID(ctx, tenantID)
}

// PostgREST usage in frontend applications
// GET /tenants?tenant_id=eq.123&select=name,tier,created_at
// Automatically respects RLS policies
```

**Security Considerations:**
- Both sqlc and PostgREST respect PostgreSQL RLS policies
- PostgREST should be configured with read-only access for most endpoints
- Use JWT authentication for PostgREST endpoints
- Apply rate limiting to PostgREST endpoints via AgentGateway

### 11. Testing Strategy

**Principle:** Use E2E tests with Testcontainers-Go to validate multi-tenant scenarios.

**Test Structure:**
```go
func TestTenantIsolation(t *testing.T) {
    // Start PostgreSQL with Testcontainers
    ctx := context.Background()
    pgContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:15-alpine"),
    )
    require.NoError(t, err)
    defer pgContainer.Terminate(ctx)
    
    // Start NATS with Testcontainers
    natsContainer, err := nats.RunContainer(ctx)
    require.NoError(t, err)
    defer natsContainer.Terminate(ctx)
    
    // Setup: Create Control Plane
    controlPlane := setupControlPlane(t, pgContainer, natsContainer)
    
    // Test: Create two tenants
    tenant1, err := controlPlane.CreateTenant(ctx, CreateTenantRequest{
        Name:  "tenant-1",
        Tier:  "basic",
        Email: "admin@tenant1.com",
    })
    require.NoError(t, err)
    
    tenant2, err := controlPlane.CreateTenant(ctx, CreateTenantRequest{
        Name:  "tenant-2",
        Tier:  "basic",
        Email: "admin@tenant2.com",
    })
    require.NoError(t, err)
    
    // Test: Tenant 1 creates a resource
    resource1, err := createResource(ctx, tenant1.ID, "resource-1")
    require.NoError(t, err)
    
    // Test: Tenant 2 cannot access tenant 1's resource
    _, err = getResource(ctx, tenant2.ID, resource1.ID)
    assert.Error(t, err, "Tenant 2 should not access tenant 1's resource")
    
    // Test: Tenant 1 can access their own resource
    resource, err := getResource(ctx, tenant1.ID, resource1.ID)
    assert.NoError(t, err)
    assert.Equal(t, resource1.ID, resource.ID)
}
```


## Package Structure

```
open-sbt/
├── pkg/
│   ├── interfaces/           # Core interfaces (IAuth, IEventBus, etc.)
│   │   ├── auth.go
│   │   ├── eventbus.go
│   │   ├── provisioner.go
│   │   ├── controlplanestore.go  # IControlPlaneStore interface
│   │   ├── storage.go
│   │   ├── billing.go
│   │   └── metering.go
│   │
│   ├── providers/            # Default implementations
│   │   ├── ory/             # Ory Stack auth implementation
│   │   ├── nats/            # NATS event bus implementation
│   │   ├── crossplane/      # Crossplane provisioner
│   │   ├── postgres/        # PostgreSQL storage implementation (sqlc)
│   │   ├── postgrest/       # PostgREST dashboard API implementation
│   │   ├── hubstore/        # HubStore (Spoke Pool - direct connection)
│   │   └── localstore/      # LocalStore (Spoke Silo - local DB + NATS sync)
│   │
│   ├── controlplane/         # Control Plane components
│   │   ├── controlplane.go  # Main Control Plane struct
│   │   ├── api.go           # API server (Gin)
│   │   ├── tenant.go        # Tenant management service
│   │   ├── registration.go  # Tenant registration service
│   │   ├── user.go          # User management service
│   │   └── config.go        # Tenant configuration service
│   │
│   ├── applicationplane/     # Application Plane components
│   │   ├── applicationplane.go
│   │   ├── provisioner.go
│   │   └── workflows.go
│   │
│   ├── spokecontroller/      # Spoke Controller (status sync)
│   │   ├── controller.go    # Watches Crossplane claims
│   │   ├── postgrest.go     # Writes to Hub via PostgREST
│   │   └── reconciler.go    # Reconciliation logic
│   │
│   ├── events/               # Event definitions and handlers
│   │   ├── definitions.go
│   │   ├── handlers.go
│   │   └── publisher.go
│   │
│   ├── mcp/                  # MCP server implementation
│   │   ├── server.go
│   │   ├── tools.go
│   │   └── handlers.go
│   │
│   └── models/               # Data models
│       ├── tenant.go
│       ├── user.go
│       ├── registration.go
│       └── events.go
│
├── examples/                 # Example implementations
│   ├── basic/               # Basic Control Plane + App Plane
│   ├── with-billing/        # With billing integration
│   └── with-metering/       # With metering integration
│
├── docs/                     # Documentation
│   ├── open-sbt-design-principles.md
│   ├── getting-started.md
│   └── api-reference.md
│
└── tests/                    # E2E tests
    ├── integration/
    └── e2e/
```


## Usage Example

### Basic Control Plane Setup

```go
package main

import (
    "context"
    "log"
    
    opensbt "github.com/zero-ops/open-sbt/pkg"
    "github.com/zero-ops/open-sbt/pkg/providers/ory"
    "github.com/zero-ops/open-sbt/pkg/providers/nats"
    "github.com/zero-ops/open-sbt/pkg/providers/postgres"
)

func main() {
    ctx := context.Background()
    
    // Initialize providers
    auth := ory.NewOryAuth(ory.Config{
        KratosURL: "http://kratos:4433",
        HydraURL:  "http://hydra:4444",
        KetoURL:   "http://keto:4466",
    })
    
    eventBus := nats.NewNATSEventBus(nats.Config{
        URL: "nats://nats:4222",
    })
    
    storage := postgres.NewPostgresStorage(postgres.Config{
        Host:     "postgres",
        Port:     5432,
        Database: "opensbt",
        User:     "opensbt",
        Password: "password",
    })
    
    // Create Control Plane
    controlPlane, err := opensbt.NewControlPlane(ctx, opensbt.ControlPlaneConfig{
        Auth:              auth,
        EventBus:          eventBus,
        Storage:           storage,
        SystemAdminEmail:  "admin@example.com",
        SystemAdminName:   "admin",
        APIPort:           8080,
    })
    if err != nil {
        log.Fatal(err)
    }
    
    // Start Control Plane
    if err := controlPlane.Start(ctx); err != nil {
        log.Fatal(err)
    }
}
```


### Basic Application Plane Setup

```go
package main

import (
    "context"
    "log"
    
    opensbt "github.com/zero-ops/open-sbt/pkg"
    "github.com/zero-ops/open-sbt/pkg/providers/nats"
    "github.com/zero-ops/open-sbt/pkg/providers/crossplane"
)

func main() {
    ctx := context.Background()
    
    // Initialize providers
    eventBus := nats.NewNATSEventBus(nats.Config{
        URL: "nats://nats:4222",
    })
    
    provisioner := crossplane.NewCrossplaneProvisioner(crossplane.Config{
        KubeConfig: "/path/to/kubeconfig",
        GitClient:  gitClient,
    })
    
    // Create Application Plane
    appPlane, err := opensbt.NewApplicationPlane(ctx, opensbt.ApplicationPlaneConfig{
        EventBus:    eventBus,
        Provisioner: provisioner,
    })
    if err != nil {
        log.Fatal(err)
    }
    
    // Register event handlers for non-infrastructure coordination (5%)
    appPlane.OnTenantCreated(func(ctx context.Context, event opensbt.TenantCreatedEvent) error {
        log.Printf("Initializing billing for new tenant: %s", event.TenantID)
        
        // Setup billing via external API (no direct K8s mutations)
        err := billingProvider.CreateCustomer(ctx, event.TenantID, event.Email)
        if err != nil {
            return appPlane.PublishBillingFailure(ctx, event.TenantID, err)
        }
        
        return appPlane.PublishBillingSuccess(ctx, event.TenantID)
    })
    
    // Start Application Plane
    if err := appPlane.Start(ctx); err != nil {
        log.Fatal(err)
    }
}
```


### Custom Provider Implementation

```go
// Example: Custom authentication provider
package custom

import (
    "context"
    opensbt "github.com/zero-ops/open-sbt/pkg/interfaces"
)

type CustomAuth struct {
    config CustomAuthConfig
    client *CustomAuthClient
}

func NewCustomAuth(config CustomAuthConfig) opensbt.IAuth {
    return &CustomAuth{
        config: config,
        client: NewCustomAuthClient(config),
    }
}

func (a *CustomAuth) CreateUser(ctx context.Context, user opensbt.User) error {
    // Custom implementation
    return a.client.CreateUser(ctx, user)
}

func (a *CustomAuth) AuthenticateUser(ctx context.Context, credentials opensbt.Credentials) (*opensbt.Token, error) {
    // Custom implementation
    return a.client.Authenticate(ctx, credentials)
}

// Implement all other IAuth methods...

// Use custom provider
func main() {
    customAuth := custom.NewCustomAuth(custom.CustomAuthConfig{
        // Custom configuration
    })
    
    controlPlane, err := opensbt.NewControlPlane(ctx, opensbt.ControlPlaneConfig{
        Auth: customAuth,  // Works seamlessly
        // ...
    })
}
```


## Implementation Roadmap

### Phase 1: Core Interfaces and Models (Week 1-2)
- [ ] Define all core interfaces (IAuth, IEventBus, IProvisioner, IStorage)
- [ ] Define data models (Tenant, User, Registration, Event)
- [ ] Create package structure
- [ ] Write interface documentation

### Phase 2: Default Providers (Week 3-4)
- [ ] Implement Ory Stack auth provider
- [ ] Implement NATS event bus provider
- [ ] Implement PostgreSQL storage provider (with sqlc)
- [ ] Implement PostgREST dashboard provider
- [ ] Implement Crossplane provisioner

### Phase 3: Control Plane (Week 5-6)
- [ ] Implement Control Plane core
- [ ] Implement Tenant Management Service
- [ ] Implement Tenant Registration Service
- [ ] Implement User Management Service
- [ ] Implement Tenant Configuration Service
- [ ] Implement API server (Gin)
- [ ] Add JWT middleware
- [ ] Add tenant context middleware

### Phase 4: Application Plane (Week 7-8)
- [ ] Implement Application Plane core
- [ ] Implement event handlers
- [ ] Implement provisioning workflows
- [ ] Integrate with Crossplane
- [ ] Integrate with ArgoCD

### Phase 5: MCP Integration (Week 9)
- [ ] Implement MCP server
- [ ] Create MCP tools for all Control Plane operations
- [ ] Add MCP tool documentation
- [ ] Test MCP integration

### Phase 6: Testing and Documentation (Week 10-11)
- [ ] Write E2E tests with Testcontainers-Go
- [ ] Write integration tests
- [ ] Create example implementations
- [ ] Write getting started guide
- [ ] Write API reference documentation

### Phase 7: Advanced Features (Week 12+)
- [ ] Implement billing integration (optional)
- [ ] Implement metering integration (optional)
- [ ] Add observability helpers
- [ ] Add CLI tool
- [ ] Create Helm charts for deployment


## Key Design Decisions

### 1. Why Go Instead of TypeScript/CDK?
- **Reason**: Zero-ops platform is Go-based, better performance, simpler deployment
- **Trade-off**: Lose CDK's CloudFormation synthesis, but gain Kubernetes-native tooling

### 2. Why NATS Instead of EventBridge?
- **Reason**: Cloud-agnostic, better performance, simpler operations, BYOC model
- **Trade-off**: Need to manage NATS infrastructure, but gain flexibility

### 3. Why Ory Stack Instead of Cognito?
- **Reason**: Open-source, self-hosted, cloud-agnostic, better customization
- **Trade-off**: More operational overhead, but gain control and cost savings

### 4. Why Crossplane + ArgoCD Instead of CloudFormation?
- **Reason**: Kubernetes-native, GitOps-first, cloud-agnostic, better for BYOC
- **Trade-off**: Steeper learning curve, but gain flexibility and portability

### 5. Why PostgreSQL Instead of DynamoDB?
- **Reason**: Relational model fits tenant data, better query capabilities, RLS support
- **Trade-off**: Need to manage PostgreSQL, but gain SQL power and RLS

### 6. Why Interface-Based Abstraction?
- **Reason**: Allow provider swapping without breaking user code, future-proof
- **Trade-off**: More upfront design work, but gain long-term flexibility

### 7. Why GitOps-First?
- **Reason**: Audit trail, rollback capability, declarative, aligns with platform principles
- **Trade-off**: Slower provisioning, but gain safety and traceability

### 8. Why MCP-First?
- **Reason**: Agent-friendly interface, standardized protocol, discoverable capabilities
- **Trade-off**: Additional abstraction layer, but gain agent integration


## Comparison: SBT-AWS vs open-sbt

| Aspect | SBT-AWS | open-sbt |
|--------|---------|----------|
| **Language** | TypeScript | Go |
| **Infrastructure** | AWS CDK | Kubernetes-native |
| **Event Bus** | AWS EventBridge | NATS |
| **Authentication** | AWS Cognito | Ory Stack (Kratos/Hydra/Keto) |
| **Provisioning** | CloudFormation + CodeBuild | Crossplane + ArgoCD GitOps |
| **Database** | DynamoDB | PostgreSQL + sqlc + PostgREST |
| **API** | API Gateway + Lambda | Gin (Go HTTP framework) |
| **Deployment** | CloudFormation | GitOps (ArgoCD) |
| **Isolation** | IAM + VPC | RLS + Ory Keto + K8s Namespaces |
| **Agent Integration** | N/A | MCP Protocol |
| **Cloud Model** | AWS-only | Cloud-agnostic (BYOC) |
| **Package Manager** | npm | Go modules |
| **Testing** | Jest | Testcontainers-Go |

## Similarities with SBT-AWS

1. **Control Plane + Application Plane Architecture**: Same conceptual separation
2. **Interface-Based Abstraction**: Both use interfaces for pluggable components
3. **Event-Driven Communication**: Both use events for plane-to-plane communication
4. **Tenant Lifecycle Management**: Same workflows (onboarding, offboarding, etc.)
5. **Pluggable Providers**: Both allow swapping implementations
6. **Standard Events**: Similar event naming and structure
7. **Multi-Tenant Security**: Both emphasize tenant isolation
8. **Observability**: Both provide tenant-aware monitoring


## Best Practices

### 1. Always Use Interfaces
```go
// ❌ DON'T: Depend on concrete types
func NewControlPlane(oryAuth *ory.OryAuth) *ControlPlane {
    // Tightly coupled to Ory
}

// ✅ DO: Depend on interfaces
func NewControlPlane(auth opensbt.IAuth) *ControlPlane {
    // Works with any IAuth implementation
}
```

### 2. Always Include Tenant Context
```go
// ❌ DON'T: Query without tenant context
func GetUser(ctx context.Context, userID string) (*User, error) {
    return db.Query("SELECT * FROM users WHERE id = $1", userID)
}

// ✅ DO: Always filter by tenant
func GetUser(ctx context.Context, tenantID, userID string) (*User, error) {
    return db.Query("SELECT * FROM users WHERE tenant_id = $1 AND id = $2", tenantID, userID)
}
```

### 3. Always Use GitOps for Infrastructure
```go
// ❌ DON'T: Apply Kubernetes resources directly
func ProvisionTenant(ctx context.Context, req ProvisionRequest) error {
    return k8sClient.Create(ctx, namespace)
}

// ✅ DO: Commit to Git, let ArgoCD apply
func ProvisionTenant(ctx context.Context, req ProvisionRequest) error {
    return gitClient.CommitTenantConfig(ctx, req.TenantID, config)
}
```

### 4. Always Emit Events for Non-Infrastructure Async Operations
```go
// ❌ DON'T: Route infrastructure provisioning through NATS events
func OnboardTenant(ctx context.Context, req OnboardRequest) (*Tenant, error) {
    tenant := createTenant(req)
    eventBus.Publish(ctx, OnboardingRequestEvent{TenantID: tenant.ID})
    // ❌ Wrong: App Plane would then do the Git commit — this is the anti-pattern
    return tenant, nil
}

// ✅ DO: Hub Tenant Management API commits to Git directly, then emits NATS for side effects only
func OnboardTenant(ctx context.Context, req OnboardRequest) (*Tenant, error) {
    tenant := createTenant(req)
    // Infrastructure: commit new tenant folder to central GitOps repo
    gitProvisioner.CommitTenantState(ctx, ProvisionRequest{TenantID: tenant.ID, Tier: tenant.Tier})
    // Non-infra side effects only (billing, notifications)
    eventBus.Publish(ctx, TenantCreatedEvent{TenantID: tenant.ID})
    return tenant, nil
}
```


### 5. Always Log with Tenant Context
```go
// ❌ DON'T: Log without tenant information
log.Info("User created")

// ✅ DO: Include tenant context in all logs
log.WithFields(log.Fields{
    "tenant_id": tenantID,
    "user_id":   userID,
    "action":    "create_user",
}).Info("User created successfully")
```

### 6. Always Tag Metrics with Tenant ID
```go
// ❌ DON'T: Record metrics without tenant labels
requestDuration.Observe(duration.Seconds())

// ✅ DO: Tag with tenant_id
requestDuration.WithLabelValues(tenantID, tier, method, path).Observe(duration.Seconds())
```

### 7. Always Test Multi-Tenant Scenarios
```go
// ❌ DON'T: Test single-tenant only
func TestCreateUser(t *testing.T) {
    user, err := CreateUser(ctx, "user-1")
    assert.NoError(t, err)
}

// ✅ DO: Test tenant isolation
func TestTenantIsolation(t *testing.T) {
    tenant1 := createTenant(t, "tenant-1")
    tenant2 := createTenant(t, "tenant-2")
    
    user1 := createUser(t, tenant1.ID, "user-1")
    
    // Tenant 2 should not access tenant 1's user
    _, err := getUser(t, tenant2.ID, user1.ID)
    assert.Error(t, err)
}
```

### 8. Always Use Type-Safe Database Queries
```go
// ❌ DON'T: Use raw SQL strings
func GetTenant(ctx context.Context, tenantID string) (*Tenant, error) {
    var tenant Tenant
    err := db.QueryRow("SELECT * FROM tenants WHERE id = $1", tenantID).Scan(&tenant)
    return &tenant, err
}

// ✅ DO: Use sqlc generated queries
func GetTenant(ctx context.Context, tenantID string) (*Tenant, error) {
    return queries.GetTenant(ctx, tenantID)
}
```


## Conclusion

open-sbt provides a Go-based SaaS builder toolkit that adapts the proven architecture patterns from AWS SBT-AWS to the zero-ops platform's technology stack. By following these design principles, developers can build multi-tenant SaaS applications with:

- **Clear separation of concerns** (Control Plane vs Application Plane)
- **Pluggable components** (swap implementations without breaking code)
- **Event-driven architecture** (async communication via NATS)
- **Strong tenant isolation** (defense-in-depth security)
- **GitOps-first operations** (declarative, auditable, rollback-capable)
- **MCP-first integration** (agent-friendly interface)
- **Cloud-agnostic design** (BYOC model, portable across clouds)

The toolkit abstracts away the complexity of multi-tenant SaaS infrastructure while maintaining flexibility for customization and future evolution.

## Additional Design Sections

The following additional design patterns were discovered from thorough SBT-AWS analysis:

### Tier Management
**Location:** `docs/open-sbt-tier-management.md`

**Key Insight:** Tiers are attributes on tenants, not separate entities. This flexible approach allows:
- Simple tier-based provisioning logic
- Optional centralized tier configuration for quotas
- Easy tier upgrades/downgrades via tenant updates

### Multi-Tenant Microservice Libraries
**Location:** `docs/open-sbt-microservice-libraries.md`

**Key Insight:** Inspired by SBT-AWS Token Vending Machine, open-sbt provides Go libraries that handle:
- Identity token management (JWT validation + tenant context extraction)
- Tenant-aware logging (auto-inject tenant_id)
- Tenant-aware metrics (auto-tag with tenant_id)
- Token vending machine (tenant-scoped credentials)
- Database isolation (automatic PostgreSQL RLS)

**Benefit:** Application developers focus on business logic while libraries handle multi-tenancy.

## References

- [AWS SaaS Builder Toolkit (SBT-AWS)](https://github.com/awslabs/sbt-aws)
- [AWS Well-Architected SaaS Lens](https://docs.aws.amazon.com/wellarchitected/latest/saas-lens/saas-lens.html)
- [Crossplane Documentation](https://docs.crossplane.io/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [NATS Documentation](https://docs.nats.io/)
- [Ory Documentation](https://www.ory.sh/docs/)

---

**Document Version:** 1.1  
**Last Updated:** 2026-03-16  
**Status:** Complete - Ready for Review

**Changelog:**
- v1.1 (2026-03-16): Added Tier Management and Multi-Tenant Microservice Libraries sections based on SBT-AWS analysis
- v1.0 (2026-03-16): Initial version with core architecture principles

## Additional Design Principles

### 11. Dual Admin Console Architecture

**Principle:** Provide separate admin consoles for SaaS Provider (platform admin) and Tenant Admins with distinct capabilities.

**SaaS Provider Admin Console:**
- **Purpose**: Platform-wide management and operations
- **Technology**: TypeScript/React (Platform Console)
- **Capabilities**:
  - Tenant management (create, update, delete, suspend)
  - Tier management (define tiers, pricing, quotas)
  - Platform monitoring (all tenants, global metrics)
  - System configuration
  - Billing and metering oversight
  - User management (platform admins)
  - Audit logs (all tenant activities)
  - Resource allocation and quotas
  - Feature flag management

**Tenant Admin Console:**
- **Purpose**: Tenant-specific management
- **Technology**: TypeScript/React (embedded or standalone)
- **Capabilities**:
  - Tenant user management (CRUD operations)
  - Tenant configuration
  - Tenant-specific monitoring and metrics
  - Billing and usage reports (own tenant)
  - Audit logs (own tenant only)
  - API key management
  - Webhook configuration
  - Integration settings


**Console Architecture:**
```
┌─────────────────────────────────────────────────────────────┐
│                    SaaS Provider Admin Console               │
│  (Platform Console - TypeScript/React)                       │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Tenants    │  │    Tiers     │  │  Monitoring  │      │
│  │  Management  │  │  Management  │  │   (Global)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Billing    │  │  Audit Logs  │  │   System     │      │
│  │  (All Tenants)│  │  (Platform)  │  │   Config     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Control Plane API
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      Control Plane (Go)                      │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Tenant     │  │     Tier     │  │     User     │      │
│  │  Management  │  │  Management  │  │  Management  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Tenant API
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Tenant Admin Console                      │
│  (Tenant-specific - TypeScript/React)                        │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │    Users     │  │    Config    │  │  Monitoring  │      │
│  │  (Own Tenant)│  │  (Own Tenant)│  │  (Own Tenant)│      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Billing    │  │  Audit Logs  │  │   API Keys   │      │
│  │  (Own Tenant)│  │  (Own Tenant)│  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

