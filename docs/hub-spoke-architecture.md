---
purpose: Hub-Spoke architecture patterns, fleet management, and tenant provisioning
scope: Hub cluster infrastructure, spoke provisioning, fleet registry, ArgoCD agents, observability
topics: [hub-spoke-pattern, fleet-registry, argocd-agents, spoke-provisioning, tenant-onboarding]
update_criteria: Hub-spoke architecture changes, fleet management patterns, spoke provisioning decisions
---

# Hub-Spoke Architecture

## Hub Cluster (Management Cluster)

### Current Infrastructure
- **Cluster Name:** `mothership`
- **Nodes:** 3 control plane + 2 workers
- **Provider:** Hetzner Cloud
- **OS:** Ubuntu 24.04 LTS
- **Kubernetes:** v1.31.6
- **Container Runtime:** containerd 1.7.26

### Installed Components
- **CAPI Stack** (namespace: `capi-operator-system`):
  - capi-controller-manager (Cluster API core)
  - caph-controller-manager (Hetzner provider)
  - capi-kubeadm-bootstrap-controller-manager
  - capi-kubeadm-control-plane-controller-manager
  - capi-operator-controller-manager
- **ArgoCD** (namespace: `argocd`):
  - argocd-application-controller
  - argocd-applicationset-controller
  - argocd-repo-server
  - argocd-server
  - argocd-dex-server
  - argocd-redis
  - argocd-notifications-controller
- **CloudNativePG** (namespace: `cnpg-system`):
  - cnpg-cloudnative-pg operator
- **Kyverno** (namespace: `kyverno`):
  - kyverno-controller (policy engine for CAPI to ArgoCD cluster registration)

### ClusterClass Templates
- **Existing:** `hetzner-mgmt-ubuntu-v1` (in zero-ops-system namespace)
- **Needed for MVP:**
  - `hetzner-prod-ubuntu-v1` (3 CP + auto-scaling workers, production spoke)
  - `hetzner-staging-ubuntu-v1` (3 CP + min 1 worker, staging spoke)

### Crossplane Infrastructure
- **XRDs (Composite Resource Definitions):**
  - `AINativeSaaS` - Tenant infrastructure provisioning
  - `SpokePool` - Spoke Pool cluster provisioning for cell-based scaling
- **Compositions:**
  - `Composition A` - Shared Spoke Pool (Starter/Growth tiers, up to 100 tenants per pool)
  - `Composition B` - Dedicated Spoke (Premium/Enterprise tiers, 1 tenant per cluster)
- **See:** [Crossplane Compositions](./crossplane-compositions.md) for detailed specifications

### Hub Bootstrap Implementation
- **Location:** `zero-ops/internal/hub/`
- **Orchestrator:** `bootstrap/orchestrator.go`
  - Phases: Bootstrap → CAPI Init → Cluster Provision → Pivot → ClusterClass Deploy → PostBoot
  - Recovery/resume capability via state manager
  - Idempotent operations throughout
- **Component Installer:** `components/installer.go`
  - Installs ArgoCD via Helm (version 5.51.6)
  - Installs Kyverno for CAPI→ArgoCD cluster registration
  - Installs CloudNativePG operator
  - Installs Hetzner CSI driver
- **ArgoCD Integration:** ArgoCD IS part of hub orchestration (installed in PostBoot phase)

### Bootstrap Handoff Pattern

**Critical Design Principle:** ClusterResourceSet is used ONLY for injecting the ArgoCD Agent. All other platform components are deployed via ArgoCD ApplicationSets.

**Why?**
- CRS is "fire-and-forget" with no drift reconciliation
- ArgoCD provides observability, health checks, and easy upgrades
- Upgrading CNPG across 1,000 clusters via CRS is a nightmare
- ArgoCD makes it a simple Git commit

**Cluster Registration Flow (Kyverno):**
1. CAPI provisions cluster and creates kubeconfig Secret
2. Kyverno ClusterPolicy watches `Cluster.status.phase=Provisioned`
3. Kyverno extracts kubeconfig from CAPI Secret
4. Kyverno generates ArgoCD cluster Secret with labels: `spoke-type: pool`, `cell-id: <cluster-name>`
5. ArgoCD discovers cluster within 30 seconds

**Timeline:**
1. **Infrastructure:** Crossplane + CAPI provision Hetzner VMs
2. **Bootstrap:** CRS injects ArgoCD Agent (Secret Zero)
3. **Registration:** Kyverno auto-generates ArgoCD cluster Secret
4. **Platform:** ArgoCD Cluster Generator deploys edge-catalog
5. **Tenants:** ArgoCD Git Generator deploys tenant workloads

See [Crossplane Compositions](./crossplane-compositions.md) for detailed XRD specifications.

## Spoke Cluster Architecture

### Provisioning Pattern
- **Declarative:** CAPI Cluster resources committed to Git
- **ClusterClass-based:** Use templates for consistent topology
- **HA Control Plane:** 3 master nodes across 3 availability zones
- **Auto-scaling Workers:** Worker pools with min/max replicas
- **Network Isolation:** 
  - Dedicated Spoke: Private network per tenant
  - Shared Spoke Pool: Dual-namespace + NetworkPolicy isolation (see [Crossplane Compositions](./crossplane-compositions.md))

### Spoke Components (Edge Catalog)

**Bootstrap Phase (ClusterResourceSet):**
- **ArgoCD Agent ONLY:** Injected via CAPI ClusterResourceSet (Secret Zero)

**Platform Phase (ArgoCD ApplicationSet - Cluster Generator with Sync Waves):**
- **Wave 0:** Database extensions (if needed)
- **Wave 1:** CNPG Operator & Shared Cluster + PgBouncer (PostgreSQL for tenant schemas)
- **Wave 2:** Atlas Operator (GitOps-driven schema migrations and drift detection)
- **Wave 3:** PostgREST (auto-generated REST API with JWT validation and schema routing)
- **Wave 4:** NATS Leaf Node (event bus connection to Hub)
- **Wave 4:** SPIRE Agent (identity federation with Hub)
- **Wave 4:** Grafana Alloy (metrics forwarding to Hub VictoriaMetrics)
- **Wave 4:** Spoke Controller (watches Crossplane claims, writes status to Hub DB)

**Tenant Phase (ArgoCD ApplicationSet - Git Generator):**
- **Tenant Workloads:** Control plane + application plane per tenant

## Fleet Registry Pattern

### Proposed Structure (Monorepo)
```
zero-ops/
├── fleet-registry/
│   ├── tenants/
│   │   ├── tenant-acme/
│   │   │   └── values.yaml
│   │   ├── tenant-globex/
│   │   │   └── values.yaml
│   │   └── ...
│   ├── applicationsets/
│   │   ├── control-plane-appset.yaml
│   │   └── app-plane-appset.yaml
│   └── README.md
├── charts/
│   └── universal-tenant/
│       ├── Chart.yaml
│       ├── values.yaml (defaults)
│       └── templates/
│           ├── ainativesaas-xr.yaml
│           ├── atlasmigration-cr.yaml
│           ├── namespace.yaml
│           └── rbac.yaml
```

### Tenant Values Schema (Universal Tenant Helm Chart Pattern)
```yaml
# fleet-registry/tenants/tenant-acme/values.yaml
# Tenant intent (input), not implementation (CRs)
tenantId: acme
orgName: Acme Corporation
plan: dedicated  # free, shared, dedicated
features:
  - ai-runtime
  - vector-search
database:
  # Deterministic schema naming for safe migration replay
  schemaName: tenant_acme  # Format: tenant_<id>
  migrations:
    gitRepo: https://github.com/zero-ops/migrations
    path: tenant-baseline
    targetRevision: main
controlPlaneRepo:
  url: https://github.com/acme/control-plane
  path: manifests
  targetRevision: main
appPlaneRepo:
  url: https://github.com/acme/app-plane
  path: manifests
  targetRevision: main
clusterRef: spoke-acme
environment: production  # production, staging
```

**Pattern**: MCP → values.yaml → Git → Helm → CRs → ArgoCD → deploy
- **Chart**: Platform policy (templates define structure, sync waves, security)
- **Values**: Tenant input (schema name, capacity, features)
- **Helm generates**: `AINativeSaaS` XR, `AtlasMigration` CR, namespace, RBAC from templates

### ApplicationSet Pattern
- **Fleet Infrastructure ApplicationSet (Cluster Generator):** Deploys edge-catalog (CNPG, NATS, SPIRE, Alloy, Atlas Operator) to all Spoke Pool clusters with sync waves
- **Tenant ApplicationSet (Git Generator):** Watches `fleet-registry/tenants/*/values.yaml` for tenant values files
- **Auto-Discovery:** New tenant directories trigger Helm Application creation
- **Helm Rendering:** Universal Tenant Chart generates CRs from values.yaml (platform policy in templates, tenant input in values)
- **Sync Waves:** Enforce dependency ordering (Wave 0: Extensions → Wave 1: CNPG → Wave 2: Atlas → Wave 3: PostgREST → Wave 4: Workloads)
- **Health Checks:** Gate progression between waves (CNPG Ready before Atlas, Atlas Ready before PostgREST)
- **Control Plane ApplicationSet:** Deploys tenant control plane services
- **App Plane ApplicationSet:** Deploys tenant application workloads

#### Fleet Infrastructure ApplicationSet Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: spoke-pool-infrastructure
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          spoke-type: pool  # Targets all Spoke Pool clusters
  template:
    metadata:
      name: '{{name}}-edge-catalog'
    spec:
      project: platform
      source:
        repoURL: https://github.com/zero-ops/catalog
        targetRevision: HEAD
        path: edge-catalog/  # Contains CNPG, NATS, SPIRE, Alloy
      destination:
        server: '{{server}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## GitOps Flow (Declarative)

### Imperative Layer (Thin)
1. Create tenant record in PostgreSQL
2. Create/register tenant Git repositories
3. Commit tenant values to `fleet-registry/tenants/tenant-<id>/values.yaml`

### Declarative Layer (Controllers)
1. ApplicationSet detects new tenant directory in Git
2. Generates Helm-based ArgoCD Application with Universal Tenant Chart
3. Helm renders templates with tenant values → generates CRs (`AINativeSaaS` XR, `AtlasMigration` CR, namespace, RBAC)
4. CAPI provisions spoke cluster (if dedicated)
5. ArgoCD agent deployed to spoke cluster
6. Agent syncs edge catalog with dependency ordering:
   - Wave 1: CNPG cluster reaches Ready
   - Wave 2: Atlas Operator deploys, creates `tenant_<id>` schema, applies migrations
   - Wave 3: PostgREST deploys with schema routing to `tenant_<id>`
   - Wave 4: Tenant workloads deploy
7. Atlas Operator continuously reconciles Git migrations vs schema state (30-60s loop)
8. Tenant transitions to READY state

## Observability Architecture

### Hub Metrics (VictoriaMetrics)
- Tenant count by plan (free/shared/dedicated)
- Spoke cluster health (Ready/Degraded/Failed)
- ArgoCD sync status (Synced/OutOfSync/Unknown)
- Resource utilization (CPU/memory/storage)

### Spoke Metrics Collection
- **Grafana Alloy:** Lightweight agent on each spoke
- **Push Pattern:** Spokes push metrics to hub VictoriaMetrics
- **Buffering:** Alloy buffers metrics when hub unreachable
- **mTLS:** Secure spoke-to-hub communication

### Dashboards (Grafana)
- Platform health overview
- Tenant status by plan
- Cluster health by region
- ArgoCD sync status
- Resource utilization trends

## Tenant Onboarding Flow

### GitOps-Driven Provisioning (Enterprise-Grade)

**Flow:**
1. MCP server receives `tenant_create` call
2. MCP server validates tenant metadata and creates tenant record in PostgreSQL
3. MCP server commits tenant values to `fleet-registry/tenants/tenant-<id>/values.yaml`
4. ArgoCD ApplicationSet (Git Generator) detects new tenant directory
5. ApplicationSet creates Helm-based Application using Universal Tenant Chart
6. Helm renders templates with tenant values → generates CRs
7. Application deploys generated CRs to Spoke Pool cluster:
   - Wave 0: Database extensions (if needed)
   - Wave 1: CNPG Cluster + PgBouncer
   - Wave 2: Atlas Operator + `AtlasMigration` CR for `tenant_<id>` schema
   - Wave 3: PostgREST (after schema exists)
   - Wave 4: Tenant workloads
8. Atlas Operator creates deterministic schema: `tenant_<id>` (e.g., `tenant_acme`)
9. Atlas Operator applies baseline migrations from Git
10. PostgREST routes requests to `tenant_<id>` schema based on JWT `tenant_id` claim

**Key Principles:**
- **Tenant Intent (values.yaml)**: MCP commits tenant input, not implementation CRs
- **Platform Policy (Helm templates)**: Chart enforces sync waves, security, resource limits
- **Helm Generates CRs**: Templates + values → `AINativeSaaS` XR, `AtlasMigration` CR, namespace, RBAC
- **Deterministic Schema Naming**: `tenant_<id>` enables safe migration replay
- **GitOps-First**: All state changes via Git commits, ArgoCD reconciles
- **Sync Waves**: Health checks gate progression (CNPG Ready before Atlas, Atlas Ready before PostgREST)
- **Drift Detection**: Atlas Operator reconciles Git vs Schema state every 30-60s
- **Schema Isolation**: PostgREST sets `search_path=tenant_<id>` per request via PgBouncer transaction pooling

### Phase 1 MVP (Manual Repo Creation)
1. Platform admin calls MCP tool: `create_tenant`
2. MCP server validates tenant metadata
3. MCP server creates tenant record in PostgreSQL
4. Platform admin manually creates GitHub repos (control + app plane)
5. MCP server commits tenant values to `fleet-registry/tenants/tenant-<id>/values.yaml`
6. ApplicationSets auto-discover and provision via Helm + sync waves

### Phase 2 (Automated Repo Creation)
1. MCP server creates GitHub repos from templates
2. MCP server initializes repos with starter manifests
3. Rest of flow same as Phase 1

## Security Patterns

### mTLS for ArgoCD Agents
- **Certificate Authority:** cert-manager with self-signed CA
- **Agent Certificates:** Unique cert per spoke cluster
- **Certificate Rotation:** Automated via cert-manager
- **Hub Verification:** Hub ArgoCD verifies agent certificates

### Network Isolation
- **Dedicated Spokes:** Private network per tenant
- **Shared Spokes:** Namespace isolation + NetworkPolicies
- **Hub-to-Spoke:** No direct connections (agents pull from hub)
- **Spoke-to-Hub:** mTLS authenticated connections only

## Architectural Principles

### GitOps-First
- All provisioning via Git commits
- No imperative kubectl commands
- ArgoCD reconciles desired state
- Audit trail via Git history

### Declarative Provisioning
- CAPI Cluster resources define infrastructure
- ApplicationSets define application deployment
- Controllers converge to desired state
- No orchestration scripts

### Hub-Spoke Communication
- **Pull Model:** Agents pull from hub (no hub-to-spoke push)
- **Async Pattern:** Hub publishes events, spokes consume
- **Resilient:** Spokes work independently when hub unavailable
- **Secure:** mTLS everywhere, zero-trust

### Tenant Isolation
- **Dedicated Spoke:** 1 tenant = 1 cluster (Enterprise tier)
- **Shared Spoke:** Multi-tenant control plane, schema-level isolation (Starter tier)
- **Schema-Based Isolation:** Each tenant gets deterministic schema `tenant_<id>` in shared CNPG cluster
- **PostgREST Routing:** Sets `search_path=tenant_<id>` per request based on JWT `tenant_id` claim
- **PgBouncer Transaction Pooling:** Ensures connection reuse without session state leakage
- **Dual-Plane Pattern:** Each tenant gets two namespaces:
  - `tenant-{id}-cp` (Control Plane) - labeled with `zero-ops.io/tenant-id` and `zero-ops.io/plane: control`
  - `tenant-{id}-app` (Application Plane) - labeled with `zero-ops.io/tenant-id` and `zero-ops.io/plane: application`
- **Network Policies:** Enforce traffic isolation between tenants
  - Application Plane can only receive traffic from its own Control Plane namespace
  - Application Plane can only send traffic to shared CNPG database and DNS
  - This renders each tenant invisible to other tenants at the network layer
- **RBAC:** Tenant-specific access controls

### Cell-Based Scaling
- **SpokePool XR:** Platform Admins create `SpokePool` XRs to provision new Spoke Pool clusters
- **Capacity Management:** Each Spoke Pool has a defined capacity (e.g., 100 tenants)
- **Fleet Registry:** Tracks utilization and available capacity across all Spoke Pools
- **Automatic Assignment:** Tenant API assigns new tenants to available pools based on capacity and region
- **Horizontal Scaling:** When a pool reaches capacity, new tenants are assigned to the next available pool
- **See:** [Crossplane Compositions](./crossplane-compositions.md) for SpokePool XRD specification
