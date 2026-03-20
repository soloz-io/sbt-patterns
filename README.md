# sbt-patterns

Design patterns and steering docs for building multi-tenant SaaS platforms using `open-sbt` — a Go-based SaaS Builder Toolkit inspired by AWS SBT-AWS.

## What's in here

| Doc | Purpose |
|-----|---------|
| `sbt-design-principles.md` | Core architecture: Control/App Plane separation, interface-based abstraction, event-driven comms, GitOps-first, MCP integration, multi-tenant security |
| `saas-architecture-principles.md` | AWS Well-Architected SaaS Lens adapted for this stack — tenant isolation, observability, cost attribution, tier management |
| `kubernetes-saas-patterns.md` | Maps common SaaS user needs to open-sbt abstractions and Kubernetes patterns |
| `microservice-utils.md` | Reusable Go libraries for tenant microservices: JWT middleware, logging, metrics, tracing, cost attribution, DB isolation |
| `gitops-helm-onboarding.md` | The adapted GitOps onboarding design — ArgoCD ApplicationSet + Universal Helm Chart + Warm Pool pattern for sub-2s onboarding |
| `tier-management.md` | Tier as a tenant attribute (matches SBT-AWS), quota enforcement, upgrade/downgrade workflows |
| `tenant-identity-token.md` | Token Vending Machine pattern — tenant-scoped credentials via JWT + Kubernetes Secrets |
| `open-sbt-patterns.md` | Implementation checklist, event naming conventions, provider patterns, common pitfalls |
| `mcp-first-development.md` | MCP-first workflow: always check for existing MCP tools before writing code |

## Key concepts

**Control Plane** handles tenant lifecycle, auth, billing, and event orchestration. **Application Plane** handles infrastructure provisioning and tenant workloads. They communicate via NATS (5% coordination) and GitOps (95% infrastructure).

**Stack**: Go · Gin · NATS · PostgreSQL + sqlc · Ory (Kratos/Hydra/Keto) · Crossplane · ArgoCD · Infisical · VictoriaMetrics

**Onboarding latency**: Shared tiers (Basic/Standard) < 2s via warm pool. Dedicated tiers (Premium/Enterprise) 2–5 min via GitOps + Crossplane.
