---
title: GitOps & Helm Tenant Onboarding
description: The adapted ApplicationSet + Helm onboarding pattern for zero-ops platform using open-sbt
inclusion: always
---

# GitOps & Helm Tenant Onboarding (The Adapted Design)

**Version:** 1.0  
**Date:** 2026-03-17  
**Purpose:** Defines how the `zero-ops` platform utilizes the `open-sbt` toolkit to onboard tenants using an adapted, low-latency GitOps architecture.

## Executive Summary

To achieve an enterprise-grade, one-click SaaS factory without the operational burden of a massive platform team, the `zero-ops` platform adopts a highly optimized GitOps onboarding flow. By converging Red Hat/IBM GitOps patterns with `open-sbt`'s Control/Application Plane architecture, we eliminate custom Kubernetes operators, bypass traditional GitOps polling latency, and implement "Warm Pooling" to achieve sub-second onboarding for shared tiers.

This document analyzes the convergence of multiple GitOps and SaaS patterns to create a unified architecture for automated tenant onboarding. The analysis covers IBM's GitOps patterns, Red Hat's onboarding approaches, Red Hat Community of Practice (COP) projects, open-sbt design principles, and AWS SaaS architecture patterns.

**Key Finding:** The patterns converge into a three-layer architecture combining SaaS business logic, GitOps orchestration, and Kubernetes resource management.

## Pattern Analysis Overview

### Analyzed Patterns

This design is the synthesis of our convergence analysis across six major architectural patterns:

1. **IBM GitOps Pattern (openshift-clusterconfig-gitops):** Adopted the concept of "T-Shirt Sizing" (Basic, Standard, Premium, Enterprise) injected via global Helm values.
2. **Red Hat GitOps Onboarding (Blog Pattern):** Adopted the core mechanism: an ArgoCD `ApplicationSet` using a Git Directory Generator to stamp out tenant Helm charts dynamically.
3. **Red Hat COP Projects (namespace-configuration-operator, etc.):** *Explicitly Rejected* the use of custom Go-based K8s operators for tenant K8s resource generation to reduce operational complexity. Replaced entirely by ArgoCD + Helm.
4. **open-sbt Design Principles:** Leveraged the strict separation of Control Plane (API, Auth) and Application Plane (GitOps, NATS events, Provisioning).
5. **SaaS Architecture Principles (AWS SaaS Lens):** Enforced identity-tenant binding (Ory), defense-in-depth isolation (PostgreSQL RLS, Namespaces), and tier-based resource allocation.
6. **Kubernetes SaaS Patterns:** Mapped user needs to K8s primitives (NetworkPolicies, ResourceQuotas, Crossplane Compositions).

## The Adapted Design: Architecture

The `open-sbt` toolkit provides the `IProvisioner` and `IEventBus` interfaces. The `zero-ops` platform implements these to execute the following architecture:

**Note**: The `IEventBus` (NATS) is used for coordination events between Control Plane and Application Plane, not for status streaming. Status updates follow the Status Controller Pattern described below.

### 1. The GitOps Repository Structure
All tenant infrastructure state lives in a single, centralized Git repository managed by the Hub Tenant Management API.

```text
gitops-repo/
├── base-charts/
│   └── tenant-factory/         # Universal Helm chart for all tenants
├── tenants/
│   ├── warm-pool-01/           # Pre-provisioned unassigned tenant
│   │   └── values.yaml
│   ├── warm-pool-02/           # Pre-provisioned unassigned tenant
│   │   └── values.yaml
│   ├── acme-corp-123/          # Active assigned tenant
│   │   └── values.yaml
│   └── stark-ind-456/          # Enterprise BYOC tenant
│       └── values.yaml
└── applicationset.yaml         # The ArgoCD AppSet generator
```

### 2. The ArgoCD ApplicationSet
A single ArgoCD `ApplicationSet` watches the `tenants/` directory. Whenever the Hub Tenant Management API commits a new folder, ArgoCD automatically generates an `Application` custom resource.

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

### 3. The Universal Tenant Helm Chart
Instead of creating Go-based K8s operators, a single Helm chart handles all resource manifestation based on the T-Shirt size defined in `values.yaml`.

**Critical Design Principle: Helm vs. Crossplane Boundary**

The Universal Helm Chart has a single, focused responsibility: generate the Crossplane `AINativeSaaS` XR (Composite Resource) and pass down the T-shirt size and tenant ID. It does NOT directly generate Kubernetes primitives like Namespaces, ResourceQuotas, or NetworkPolicies.

* **Helm's Job:** Generate the `AINativeSaaS` Crossplane XR with tenant metadata (tenant ID, tier, configuration)
* **Crossplane's Job:** `Composition A` (Starter) or `Composition B` (Enterprise) reads the XR and generates:
  - Dual namespaces (`tenant-{id}-cp` for Control Plane, `tenant-{id}-app` for Application Plane)
  - NetworkPolicies (dual-plane isolation policy)
  - ResourceQuotas (based on T-shirt size)
  - Logical database and role inside the shared CNPG cluster (Starter tier)
  - Dedicated CAPI cluster (Enterprise tier)

This keeps all infrastructure reconciliation in one engine (Crossplane) and prevents split responsibility between Helm and Crossplane.

**Example: Universal Helm Chart Output**

```yaml
# Generated by Universal Helm Chart for tenant-123 (Starter tier)
apiVersion: zero-ops.io/v1alpha1
kind: AINativeSaaS
metadata:
  name: tenant-123
  namespace: default
spec:
  tenantId: "tenant-123"
  tier: starter  # Maps to Composition A
  tshirtSize: basic  # basic, standard, premium, enterprise
  region: eu-central-1
  features:
    - ai-runtime
    - vector-search
  controlPlaneRepo:
    url: https://github.com/tenant-123/control-plane
    path: manifests
    targetRevision: main
  appPlaneRepo:
    url: https://github.com/tenant-123/app-plane
    path: manifests
    targetRevision: main
```

Crossplane then reads this XR and applies the appropriate Composition (A for starter, B for enterprise) to provision all infrastructure.

### 4. Crossplane Compositions: Dual-Plane Tenant Isolation

To safely host 100 tenants (200 planes) on a single Spoke Pool cluster, Crossplane `Composition A` (Starter tier) must dynamically generate two namespaces per tenant and wire them securely.

#### Composition A: Starter Tier (Shared Spoke Pool)

When the Universal Helm Chart generates an `AINativeSaaS` XR with `tier: starter`, Crossplane applies `Composition A`, which provisions:

1. **Dual Namespaces:**
   - `tenant-{id}-cp` (Control Plane namespace) - labeled with `zero-ops.io/tenant-id: "{id}"` and `zero-ops.io/plane: "control"`
   - `tenant-{id}-app` (Application Plane namespace) - labeled with `zero-ops.io/tenant-id: "{id}"` and `zero-ops.io/plane: "application"`

2. **Dual-Plane Network Policy:**
   Each tenant's Application Plane namespace gets a NetworkPolicy that strictly binds it to its own Control Plane namespace:

```yaml
# Generated by Crossplane Composition A for tenant-123
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: tenant-123-app
spec:
  podSelector: {} # Applies to all pods in the APP plane
  policyTypes: [Ingress, Egress]
  ingress:
  # 1. Allow traffic FROM this tenant's Control Plane namespace
  - from:
    - namespaceSelector:
        matchLabels:
          zero-ops.io/tenant-id: "tenant-123"
          zero-ops.io/plane: "control"
  # 2. Allow traffic FROM the public Ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
  egress:
  # 1. Allow traffic TO the shared CNPG Database
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: hub-platform-data
    ports:
    - protocol: TCP
      port: 5432
  # 2. Allow DNS
  - ports:
    - protocol: UDP
      port: 53
```

This policy renders `tenant-123-app` invisible to `tenant-124-app`, enforcing strict tenant isolation at the network layer.

3. **ResourceQuota:**
   Based on the T-shirt size (Basic/Standard/Premium), Crossplane generates appropriate ResourceQuotas for both namespaces.

4. **Logical Database:**
   Crossplane provisions a logical database and role inside the shared CNPG cluster with PostgreSQL RLS policies enforcing tenant isolation.

#### Composition B: Enterprise Tier (Dedicated Spoke)

When the Universal Helm Chart generates an `AINativeSaaS` XR with `tier: enterprise`, Crossplane applies `Composition B`, which provisions:

1. **Dedicated CAPI Cluster:** A full Hetzner cluster via Cluster API
2. **Dedicated CNPG Database:** A dedicated PostgreSQL cluster (no RLS needed)
3. **Dual Namespaces:** Same dual-namespace pattern as Composition A
4. **Network Policies:** Same dual-plane isolation policy

---

## Mitigating the Major Concerns

To successfully run this as a one-person AI-Native company, we implement three critical mitigations:

### Mitigation 1: Eliminating Operational Complexity
* **No Custom K8s Operators:** The entire K8s footprint is declarative via Helm and Crossplane.
* **GitOps DR:** Disaster recovery requires only bootstrapping ArgoCD and pointing it to the Git repository. Crossplane and ArgoCD will reconstruct the entire SaaS platform automatically.
* **AI-Operated Platform:** The platform is managed via the `zero-ops-api` MCP server. The Platform Admin's AI agent uses tools like `sync_application` to trigger GitOps syncs and `get_cluster_diagnostics` to debug.

### Mitigation 2: Performance at Scale
* **PostgreSQL RLS Optimization:** RLS is restricted to Shared Pool tiers. The `open-sbt` data layer uses `sqlc` to enforce strict `(tenant_id)` composite indexing on all queries, guaranteeing millisecond query times even with billions of rows. PostgREST provides additional REST API access for dashboards, reporting, and **Spoke Controller status writes to Hub Database**.
* **Webhook-Driven Syncs (No Polling):** ArgoCD's 3-minute Git polling is disabled. The Hub Tenant Management API fires a direct Webhook to the ArgoCD API immediately after committing to Git, triggering instant reconciliation and eliminating API server thrashing.

### Mitigation 3: Solving Onboarding Latency (The Warm Pool Pattern)
Standard GitOps provisioning takes 45-80 seconds. `open-sbt` splits the onboarding flow into two latency-optimized paths:

### Spoke Pool Capacity & Cell-Based Architecture

To scale from 100 tenants to 10,000, the platform implements a cell-based architecture where Spoke Pool clusters are dynamically provisioned and registered with the Hub.

#### SpokePool XR (Composite Resource)

Platform Admins create `SpokePool` XRs in the Hub to provision new Spoke Pool clusters:

```yaml
apiVersion: zero-ops.io/v1alpha1
kind: SpokePool
metadata:
  name: spoke-pool-eu-central-1
spec:
  region: eu-central-1
  capacity: 100  # Max tenants this pool can host
  tier: starter  # Pool hosts starter/growth tier tenants
  clusterClass: hetzner-prod-ubuntu-v1
  nodeCount: 5
```

#### Cell Registration Flow

1. **Provision:** Platform Admin creates a `SpokePool` XR in the Hub
2. **Crossplane Provisioning:** Crossplane provisions a new Hetzner cluster via CAPI
3. **Bootstrap Injection:** Using CAPI `ClusterResourceSet`, the Hub injects:
   - ArgoCD Agent (connects to Hub ArgoCD)
   - NATS Leaf Node (connects to Hub NATS)
   - SPIRE Agent (connects to Hub SPIRE Server)
   - Grafana Alloy (metrics forwarding to Hub VictoriaMetrics)
4. **Cell Registration:** A Hub controller watches for new `SpokePool` resources and registers them with:
   - Hub ArgoCD (labels: `capacity: 100`, `spoke-type: pool`, `region: eu-central-1`)
   - Hub Fleet Registry (updates available capacity)
5. **Tenant Assignment:** The Tenant API assigns new tenants to available Spoke Pools based on capacity and region

#### Capacity Management

The Hub maintains a Fleet Registry that tracks:
- Total capacity per Spoke Pool (e.g., 100 tenants)
- Current utilization (e.g., 73 tenants assigned)
- Available capacity (e.g., 27 slots remaining)

When `spoke-pool-1` reaches 100 tenants, the Tenant API automatically assigns new tenants to `spoke-pool-2`. If no pools have capacity, the Platform Admin is alerted to provision a new `SpokePool` XR.

### Status Controller Pattern

The Zero-Ops Platform implements a strict **Status Controller Pattern** for all infrastructure status tracking:

1. **Source of Truth**: Hub Database (PostgreSQL) contains all tenant and infrastructure status
2. **Status Updates**: Spoke Controllers watch Crossplane claim conditions and write status to Hub DB via PostgREST
3. **Status Reads**: Frontend/MCP tools read status directly from Hub Database
4. **Real-time Updates**: Three options for real-time frontend updates:
   - **Database Polling**: Frontend polls Hub Database at regular intervals
   - **PostgREST Listening**: Use PostgREST's native listening capabilities for real-time updates
   - **Hub Event Router**: Broadcasts DB `pg_notify` updates over NATS WebSockets

**Critical Rule**: The database is always the source of truth. Direct NATS messages from Spoke to frontend bypass this pattern and should be avoided.

#### Path A: Shared Pool Tiers (Basic / Standard) - Latency < 2 Seconds
1. The `open-sbt` App Plane maintains a baseline of 10 "warm" (pre-provisioned) namespaces and Postgres schemas in the cluster.
2. **API Call:** User requests a Basic tier SaaS instance.
3. **Instant Claim:** The Control Plane DB instantly marks `warm-pool-01` as belonging to `tenant-123`.
4. **Auth Binding:** Ory Keto instantly creates the relationship `tenant:123#admin@user:xyz`.
5. **Response:** API returns `200 OK` in < 2 seconds. The user can use the platform immediately.
6. **Async Refill:** The Hub Tenant Management API updates `warm-pool-01`'s `values.yaml` in Git to reflect its new owner, and commits a new `warm-pool-11` folder to Git to replace the consumed warm slot.

#### Path B: Dedicated / BYOC Tiers (Premium / Enterprise) - Latency 2-5 Minutes
1. **API Call:** User requests an Enterprise tier (Dedicated Hetzner CAPI Cluster or Dedicated CNPG Database).
2. **Response:** API returns `202 Accepted` instantly.
3. **AI UX Masking:** The frontend AI agent engages the user in a configuration conversation while the provisioning happens.
4. **GitOps Execution:** The Hub Tenant Management API generates the Helm values manifest and commits a new `tenants/<tenant-id>` folder to the central GitOps repo with `tier: enterprise`.
5. **Crossplane Provisioning:** ArgoCD syncs the Helm chart, which creates a Crossplane `TenantCluster` XR. Crossplane provisions the Hetzner nodes.
6. **Status Updates:** Spoke Controller watches Crossplane claim conditions and writes status updates to Hub Database via PostgREST (Bearer JWT, RLS enforced).
7. **Real-time Frontend Updates:** Frontend polls Hub Database for status updates OR uses PostgREST's native listening capabilities OR Hub Event Router broadcasts DB `pg_notify` updates over NATS WebSockets.

**Important:** The **Hub Database remains the source of truth** for all status information. Real-time updates to the frontend must go through the database layer, not direct NATS messages from the Spoke.

---

## Alignment with SaaS Architecture Principles

1. **Identity Binding:** The onboarding flow intrinsically links the user to the tenant via Ory Kratos (Identity) and Ory Keto (Relationships). The resulting JWT contains the `tenant_id` and `tenant_tier`.
2. **Defense-in-Depth Isolation:**
   * *Data Layer:* PostgreSQL RLS enforced by `open-sbt` middleware.
   * *Compute Layer:* Kubernetes Namespaces + NetworkPolicies (generated via Helm).
   * *Execution Layer:* AgentSandbox (`runsc`) sandboxing for AI Agent execution workflows.
3. **Tier-Based Provisioning:** The entire infrastructure footprint is controlled by a single `tier` property in the tenant's `values.yaml`, feeding into the Crossplane Compositions and K8s Quotas.

---

## Detailed Pattern Analysis

### 1. IBM GitOps Pattern (openshift-clusterconfig-gitops)

**Architecture Overview:**
IBM's approach uses a dual-folder structure with T-Shirt sizing for standardized configurations.

**Key Components Analyzed:**
- **T-Shirt Sizing**: Global values with Basic/Standard/Premium/Enterprise configurations defined in `values-global.yaml`
- **Helm Templating**: Values-based configuration management using helper-proj-onboarding chart
- **Folder Structure**: 
  - `tenant-onboarding/`: Team onboarding configurations
  - `tenants/`: Individual tenant application configurations
  - `clusters/`: Cluster-wide base configurations

**Observed Implementation:**
```yaml
# From tenant-onboarding/values-global.yaml
global:
  application_gitops_namespace: gitops-application
  envs:
    - name: in-cluster
      url: https://kubernetes.default.svc
  tshirt_sizes:
    - name: Enterprise
      quota:
        pods: 100
        limits:
          cpu: 4
          memory: 4Gi
    - name: Basic
      quota:
        limits:
          cpu: 1
          memory: 1Gi
```

**Integration Analysis:**
The IBM T-Shirt sizing approach uses global values with predefined configurations that map to different resource allocations and limits.

### 2. Red Hat GitOps Onboarding Pattern

**Architecture Overview:**
Red Hat's approach emphasizes ApplicationSet-driven discovery with Helm templating for tenant-specific configurations, as described in their blog article "Project onboarding using GitOps and Helm".

**Key Components Analyzed:**
- **ApplicationSet Controller**: Uses Git Generator to walk folder structure and create Applications
- **Helm Charts**: Project-onboarding chart with helper-proj-onboarding dependency
- **Git Repository Structure**: `tenant-projects/{tenant}/{cluster}/values.yaml` pattern
- **T-Shirt Sizing**: Global values-file with predefined Basic/Standard/Premium/Enterprise configurations

**Observed Tenant Onboarding Flow:**
```
1. Create folder: tenant-projects/{tenant-name}/{cluster-name}/
2. Add values.yaml with tenant configuration
3. ApplicationSet Git Generator detects new folder
4. ApplicationSet creates ArgoCD Application
5. ArgoCD deploys using Helm chart with values.yaml
6. Tenant resources provisioned automatically
```

**Integration Analysis:**
The Red Hat ApplicationSet approach uses Git Generators to automatically detect folder structures and create ArgoCD Applications for tenant provisioning.

**ApplicationSet Template (from Red Hat article):**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: tenant-onboarding
spec:
  generators:
  - git:
      repoURL: https://github.com/org/tenant-configs
      revision: HEAD
      directories:
      - path: tenants/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: tenants
      source:
        repoURL: https://github.com/org/tenant-configs
        targetRevision: HEAD
        path: '{{path}}'
        helm:
          valueFiles:
          - values.yaml
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
```

### 3. Red Hat COP Projects Analysis

#### 3.1 namespace-configuration-operator

**Purpose:** Event-driven namespace provisioning with Go templating

**Analyzed Features:**
- **Event-Driven**: Watches for Group, User, and Namespace creation events
- **Go Templates**: Uses text/template for flexible resource generation
- **CRDs**: GroupConfig, UserConfig, NamespaceConfig for configuration
- **Drift Prevention**: Continuously reconciles desired state
- **Team Onboarding Example**: Creates 4 namespaces per team (build, dev, qa, prod)

**Observed Team Onboarding Implementation:**
```yaml
# From examples/team-onboarding/group-config.yaml
apiVersion: redhatcop.redhat.io/v1alpha1
kind: GroupConfig
metadata:
  name: team-onboarding
spec:
  labelSelector:
    matchLabels:
      type: devteam    
  templates:
    - objectTemplate: |
        apiVersion: v1
        kind: Namespace
        metadata:
          name: {{ .Name }}-build
        labels:
          team: {{ .Name }}
          type: build
```

**Integration Analysis:**
The namespace-configuration-operator provides event-driven namespace provisioning that reacts to Kubernetes resource creation events.

**Note:** While the namespace-configuration-operator provides useful patterns, the zero-ops platform explicitly rejects the use of custom Go-based K8s operators for tenant resource generation to reduce operational complexity. This functionality is replaced entirely by ArgoCD + Helm.

#### 3.2 gitops-generator

**Purpose:** Resource generation using GeneratorOptions pattern

**Analyzed Features:**
- **GeneratorOptions API**: Structured configuration for resource generation
- **Multiple Resource Types**: Deployments, Services, Routes, Ingresses
- **Template-based**: Uses Go templates for resource generation
- **Kustomize Integration**: Generates Kustomize-compatible structures

**Observed API Structure:**
```go
// From api/v1alpha1/generator_options.go
type GeneratorOptions struct {
    Name        string `json:"name"`
    Namespace   string `json:"namespace,omitempty"`
    Application string `json:"application"`
    Replicas    int    `json:"replicas,omitempty"`
    TargetPort  int    `json:"targetPort,omitempty"`
    ContainerImage string `json:"containerImage,omitempty"`
    // ... additional fields
}
```

**Integration Analysis:**
The gitops-generator provides structured resource generation through its GeneratorOptions API and template-based approach.

**Note:** While gitops-generator offers structured resource generation, the zero-ops platform achieves similar functionality through the Universal Tenant Helm Chart approach, avoiding the need for additional operators.

#### 3.3 gitops-catalog

**Purpose:** Reusable GitOps patterns and components library

**Key Features:**
- **Component Library**: Pre-built GitOps patterns
- **Best Practices**: Proven configurations
- **Composability**: Mix and match components

**Integration Analysis:**
The gitops-catalog provides a library of reusable GitOps patterns and pre-built configurations for common use cases.

## Pattern Convergence Analysis

### Three-Layer Architecture Deep Dive

#### Layer 1: SaaS Business Logic (open-sbt)

**Responsibilities:**
- Tenant lifecycle management (CRUD operations)
- Event-driven communication between Control Plane and Application Plane
- Authentication and authorization (Ory Stack integration)
- Billing and metering abstractions
- MCP tools for agent integration
- Multi-tenant observability

**Observed open-sbt Interface Alignment:**
The analyzed patterns show compatibility with open-sbt's interface-based architecture through the IAuth, IEventBus, and IProvisioner interfaces.

**Observed GitOps Integration Points:**
The analyzed patterns show integration opportunities where Control Plane operations could trigger GitOps workflows, while Application Plane components could respond to Git-driven deployment events.

#### Layer 2: GitOps Orchestration (Red Hat + IBM)

**Responsibilities:**
- Git-based configuration management
- ApplicationSet-driven tenant discovery
- Helm templating and T-Shirt sizing
- ArgoCD continuous deployment
- Configuration drift prevention

**Key Components:**
```yaml
# ApplicationSet for tenant discovery
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: tenant-factory
spec:
  generators:
  - git:
      repoURL: https://github.com/org/tenant-configs
      revision: HEAD
      directories:
      - path: tenants/*
  template:
    metadata:
      name: 'tenant-{{path.basename}}'
    spec:
      source:
        helm:
          valueFiles:
          - values-{{path.basename}}.yaml
          parameters:
          - name: tenantId
            value: '{{path.basename}}'
          - name: tier
            value: '{{path.basename}}'
```

**Observed GitOps Orchestration Components:**
The Red Hat and IBM patterns demonstrate GitOps orchestration through ApplicationSet controllers, Helm templating, and ArgoCD continuous deployment for configuration management and deployment automation.

#### Layer 3: Kubernetes Resource Management (Simplified Approach)

**Responsibilities:**
- Automatic namespace configuration via Helm templates
- Resource template application through Universal Tenant Helm Chart
- RBAC and security policy enforcement via Helm-generated manifests
- Monitoring and logging setup through Crossplane compositions
- Resource quota management via Kubernetes ResourceQuotas

**Simplified Implementation:**
Instead of using Red Hat COP operators, the zero-ops platform uses a simplified approach:

```yaml
# Universal Tenant Helm Chart generates resources based on tier
{{- if eq .Values.tier "premium" }}
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-{{ .Values.tenantId }}-quota
  namespace: tenant-{{ .Values.tenantId }}
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
{{- else if eq .Values.tier "basic" }}
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-{{ .Values.tenantId }}-quota
  namespace: tenant-{{ .Values.tenantId }}
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 2Gi
    limits.cpu: "2"
    limits.memory: 4Gi
{{- end }}
```

**Key Difference from Red Hat COP Approach:**
The zero-ops platform explicitly avoids custom Kubernetes operators for tenant resource management, instead relying on:
- **Helm templating** for resource generation
- **Crossplane compositions** for infrastructure provisioning
- **ArgoCD ApplicationSets** for deployment orchestration

This approach reduces operational complexity while maintaining the same functionality.

### Analysis Summary

The convergence analysis reveals that the IBM GitOps patterns, Red Hat onboarding approaches, Red Hat COP projects, and open-sbt design principles show compatibility across three architectural layers for SaaS tenant management.

**Key Findings:**

#### 1. Pattern Compatibility
- **Business Logic Layer**: open-sbt provides SaaS-specific abstractions
- **Orchestration Layer**: GitOps handles deployment and configuration management  
- **Resource Management Layer**: Helm templates and Crossplane handle resource lifecycle (avoiding custom operators)

#### 2. Technology Consistency
- **Go-based**: Core platform components use Go for implementation consistency
- **Event-driven**: NATS events and Git events provide coordination
- **Declarative**: Configuration defined as code in Git repositories
- **Operator-free**: Eliminates custom Kubernetes operators for reduced operational complexity

#### 3. Operational Characteristics
- **GitOps**: All infrastructure changes tracked in Git with audit trails
- **Automation**: Minimal manual intervention through ArgoCD and Crossplane
- **Scalability**: Each layer operates independently with clear interfaces
- **Simplicity**: Reduced operational burden through elimination of custom operators

#### 4. Implementation Flexibility
- **Pluggable**: Each layer can use different implementations
- **Extensible**: New patterns can be added through Helm templates and Crossplane compositions
- **Cloud-agnostic**: Works across different Kubernetes distributions
- **Warm Pool Optimization**: Sub-second onboarding for shared tiers

## Analysis Conclusion

The analysis of IBM GitOps patterns, Red Hat onboarding approaches, Red Hat COP projects, and open-sbt design principles reveals compatibility across different aspects of SaaS tenant management. However, the zero-ops platform takes a simplified approach that eliminates operational complexity:

- **Proven patterns** from IBM and Red Hat implementations (ApplicationSet + Helm)
- **Simplified automation** through GitOps workflows without custom operators
- **Interface abstractions** through open-sbt's design
- **Event coordination** through NATS-driven communication
- **Operational simplicity** through elimination of custom Kubernetes operators
- **Performance optimization** through warm pool patterns and webhook-driven syncs

The adapted design demonstrates that enterprise-grade SaaS tenant onboarding can be achieved with minimal operational overhead by carefully selecting and combining proven patterns while avoiding unnecessary complexity.

## Summary of Architectural Refinements

### 1. Clear Helm vs. Crossplane Boundary

The Universal Helm Chart has a single, focused responsibility: generate the Crossplane `AINativeSaaS` XR with tenant metadata. It does NOT generate Kubernetes primitives directly.

**Helm's Job:**
- Generate `AINativeSaaS` XR
- Pass down T-shirt size and tenant ID
- Reference tenant Git repositories

**Crossplane's Job (Composition A - Shared Pool):**
- Generate dual namespaces (`tenant-{id}-cp` and `tenant-{id}-app`)
- Generate NetworkPolicies for dual-plane isolation
- Generate ResourceQuotas based on T-shirt size
- Provision logical database in shared CNPG cluster with PostgreSQL RLS

**Crossplane's Job (Composition B - Dedicated Spoke):**
- Provision dedicated Hetzner CAPI cluster
- Provision dedicated CNPG database cluster
- Generate dual namespaces and NetworkPolicies in dedicated cluster

This keeps all infrastructure reconciliation in one engine (Crossplane) and prevents split responsibility.

### 2. Dual-Plane Network Isolation

Each tenant in a Spoke Pool cluster gets two namespaces with strict network isolation:

- `tenant-{id}-cp` (Control Plane) - labeled with `zero-ops.io/tenant-id: "{id}"` and `zero-ops.io/plane: "control"`
- `tenant-{id}-app` (Application Plane) - labeled with `zero-ops.io/tenant-id: "{id}"` and `zero-ops.io/plane: "application"`

The NetworkPolicy in the Application Plane namespace:
- Allows ingress ONLY from the tenant's own Control Plane namespace (matched by labels)
- Allows ingress from the public Ingress controller
- Allows egress to the shared CNPG database and DNS
- Renders each tenant's Application Plane invisible to other tenants

This enables safely hosting 100 tenants (200 namespaces) on a single Spoke Pool cluster.

### 3. Cell-Based Scaling with SpokePool XR

To scale from 100 tenants to 10,000, the platform implements a cell-based architecture:

**SpokePool XR:**
- Platform Admins create `SpokePool` XRs to provision new Spoke Pool clusters
- Each pool has a defined capacity (e.g., 100 tenants)
- Crossplane provisions the cluster and injects ArgoCD Agent, NATS Leaf Node, SPIRE Agent, Grafana Alloy

**Fleet Registry:**
- Tracks utilization and available capacity across all Spoke Pools
- Tenant API assigns new tenants to available pools based on capacity and region
- When a pool reaches capacity, new tenants are assigned to the next available pool

**Horizontal Scaling:**
- No single cluster bottleneck
- Linear scaling by adding more Spoke Pools
- Regional distribution for compliance and latency

For detailed specifications, see [Crossplane Compositions](./crossplane-compositions.md).
