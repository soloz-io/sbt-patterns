---
title: MCP-First Development
description: Prioritize MCP server discovery and usage before writing code
inclusion: manual
---

# MCP-First Development

Before writing code for any platform component, always check if an MCP server exists that provides the needed functionality. MCP servers are the primary interface for platform capabilities and should be discovered and used first.

## Core Principle

**MCP First, Code Second**: Always attempt to use existing MCP servers before implementing new code. This ensures consistency, reduces duplication, and leverages tested platform capabilities.

## Discovery Workflow

When starting any development task, follow this sequence:

### 1. Identify the Component/Domain
Determine which platform component you're working with:
- **tenant-core**: Core platform API and tenant management
- **agent-core**: Agent execution and lifecycle
- **cluster-core**: Kubernetes cluster management (CAPI/CAPH)
- **monitoring-core**: Observability and metrics
- **auth-core**: Authentication and authorization (Ory stack)
- **gitops-core**: ArgoCD and deployment management
- **storage-core**: PostgreSQL, S3, and data operations

### 2. Check for Existing MCP Tools
Before writing any code, search for existing MCP tools in the domain folders:

**Search Pattern:**
```bash
# Look for MCP tools in domain folders
find internal/ -type d -name "mcp" | grep -i <component-name>

# Domain MCP tool locations
# - internal/tenant-core/mcp/
# - internal/agent-core/mcp/
# - internal/cluster-core/mcp/
# - internal/monitoring-core/mcp/
```

**What to Look For:**
- Tool definitions and schemas in domain folders
- Tool registration in `cmd/mcp-server/main.go`
- Authentication and authorization patterns
- Example usage and documentation

### 3. Discover Available Tools
Check the single MCP server for available tools:

**Using MCP Protocol:**
```json
// Request: List available tools from the single MCP server
{
  "jsonrpc": "2.0",
  "method": "tools/list",
  "id": 1
}

// Response: All available tools from all domains
{
  "jsonrpc": "2.0",
  "result": {
    "tools": [
      {
        "name": "create_tenant",
        "description": "Create a new tenant in the platform",
        "inputSchema": { ... }
      },
      {
        "name": "execute_agent", 
        "description": "Execute an agent with parameters",
        "inputSchema": { ... }
      }
    ]
  }
}
```


### 4. Review Tool Schemas and Capabilities
Examine the tool's input schema and capabilities:

**Key Questions:**
- Does this tool provide the functionality I need?
- What are the required and optional parameters?
- What authentication/authorization is required?
- What is the expected response format?
- Are there rate limits or quotas?
- Is tenant context automatically handled?

### 5. Use the MCP Tool
If the tool meets your needs, use it via the single MCP server instead of writing new code:

**Example: Creating a Tenant**
```go
// Instead of writing direct database code:
// db.Exec("INSERT INTO tenants ...")

// Use the MCP tool via the single server:
result, err := mcpClient.CallTool(ctx, "create_tenant", map[string]interface{}{
    "name": "acme-corp",
    "tier": "premium",
    "owner_email": "admin@acme.com",
})
```

### 6. Only Write New Code If Needed
Write new code only if:
- No MCP tool exists for this functionality in any domain folder
- Existing MCP tools don't provide the needed functionality
- You're implementing a new MCP tool in a domain folder
- You're working on internal implementation (not exposed via MCP)

**When adding new tools:**
- Add the tool implementation to the appropriate domain folder (e.g., `internal/tenant-core/mcp/`)
- Register the tool in `cmd/mcp-server/main.go`
- All tools are exposed through the single MCP server endpoint


## Component-Specific MCP Tools

### Tenant Core MCP Tools
**Location**: `internal/tenant-core/mcp/`

**Expected Tools:**
- `create_tenant`: Create new tenant
- `get_tenant`: Retrieve tenant details
- `update_tenant`: Update tenant configuration
- `delete_tenant`: Remove tenant
- `list_tenants`: List all tenants
- `get_tenant_metrics`: Retrieve tenant metrics
- `get_tenant_costs`: Get tenant cost attribution

**When to Use:**
- Tenant lifecycle management
- Tenant configuration changes
- Tenant metrics and monitoring
- Cost attribution queries

**Before Writing Code:**
```go
// ❌ DON'T: Write direct database queries
func CreateTenant(ctx context.Context, name string) error {
    _, err := db.Exec("INSERT INTO tenants (name) VALUES ($1)", name)
    return err
}

// ✅ DO: Check tenant-core MCP tools first
// 1. Search for internal/tenant-core/mcp/
// 2. Check if create_tenant tool exists
// 3. Use the tool via the single MCP server
// 4. Only add new tool to domain folder if needed
```

### Agent Core MCP Tools
**Location**: `internal/agent-core/mcp/`

**Expected Tools:**
- `create_agent`: Create new agent
- `execute_agent`: Run agent with parameters
- `get_agent_status`: Check agent execution status
- `list_agents`: List tenant's agents
- `get_agent_logs`: Retrieve agent execution logs
- `cancel_agent`: Stop running agent

**When to Use:**
- Agent lifecycle management
- Agent execution and monitoring
- Agent configuration
- Kagents workflow operations

**Before Writing Code:**
```go
// ❌ DON'T: Implement agent execution directly
func ExecuteAgent(ctx context.Context, agentID string) error {
    // Complex agent execution logic...
}

// ✅ DO: Check agent-core MCP tools first
// 1. Search for internal/agent-core/mcp/
// 2. Check if execute_agent tool exists
// 3. Review tool schema and capabilities
// 4. Use the tool via the single MCP server
```

### Cluster Core MCP Tools
**Location**: `internal/cluster-core/mcp/`

**Expected Tools:**
- `create_cluster`: Provision new Kubernetes cluster (CAPI/CAPH)
- `get_cluster`: Retrieve cluster details
- `scale_cluster`: Scale cluster nodes
- `delete_cluster`: Remove cluster
- `get_cluster_health`: Check cluster status
- `list_clusters`: List tenant's clusters

**When to Use:**
- Kubernetes cluster provisioning
- Cluster lifecycle management
- CAPI/CAPH operations
- Cluster monitoring and health checks

**Before Writing Code:**
```go
// ❌ DON'T: Implement CAPI operations directly
func CreateCluster(ctx context.Context, spec ClusterSpec) error {
    // Direct CAPI API calls...
}

// ✅ DO: Check cluster-core MCP tools first
// 1. Search for internal/cluster-core/mcp/
// 2. Check if create_cluster tool exists
// 3. Verify it supports CAPI/CAPH
// 4. Use the tool via the single MCP server
```

### Monitoring Core MCP Tools
**Location**: `internal/monitoring-core/mcp/`

**Expected Tools:**
- `query_metrics`: Query VictoriaMetrics
- `get_tenant_metrics`: Retrieve tenant-specific metrics
- `search_logs`: Search OpenSearch logs
- `create_dashboard`: Create Grafana dashboard
- `create_alert`: Configure alerting rules
- `get_cluster_diagnostics`: Run K8sGPT diagnostics

**When to Use:**
- Metrics queries and analysis
- Log searching and aggregation
- Dashboard creation
- Alert configuration
- Cluster diagnostics

### GitOps Core MCP Tools
**Location**: `internal/gitops-core/mcp/`

**Expected Tools:**
- `create_application`: Create ArgoCD application
- `sync_application`: Trigger ArgoCD sync
- `get_application_status`: Check deployment status
- `create_workflow`: Create Argo Workflow
- `list_applications`: List ArgoCD applications

**When to Use:**
- ArgoCD application management
- GitOps deployments
- Argo Workflows execution
- Deployment status monitoring


## MCP Tool Discovery Process

### Step-by-Step Discovery

**1. Identify Your Task**
```
Task: "Implement tenant creation endpoint"
Component: tenant-core
```

**2. Search for MCP Tools**
```bash
# Search in domain folders
find internal/ -path "*/tenant-core/mcp/*" -type f

# Expected locations:
# - internal/tenant-core/mcp/tools.go
# - internal/tenant-core/mcp/tenant_tools.go
# - internal/tenant-core/mcp/README.md
```

**3. Check Single MCP Server Registration**
```bash
# Check tool registration in single server
cat cmd/mcp-server/main.go

# Look for:
# - Tool registration from domain folders
# - Server initialization
# - Available endpoints
```

**4. Examine Tool Definitions**
```go
// Look for tool definitions in domain folder
// File: internal/tenant-core/mcp/tenant_tools.go

type CreateTenantTool struct {
    Name        string
    Description string
    InputSchema map[string]interface{}
}

func (t *CreateTenantTool) Execute(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    // Implementation details
}
```

**5. Test the Tool**
```bash
# Use MCP client to test via single server
curl -X POST http://localhost:8080/mcp/tools/call \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "tool": "create_tenant",
    "arguments": {
      "name": "test-tenant",
      "tier": "basic"
    }
  }'
```

**6. Integrate or Implement**
```go
// If tool exists and works, use it via single server:
result, err := mcpClient.CallTool(ctx, "create_tenant", params)

// If tool doesn't exist, implement it in domain folder:
// 1. Create new tool in internal/<domain>/mcp/
// 2. Register tool in cmd/mcp-server/main.go
// 3. Follow MCP protocol standards
// 4. Include tenant context handling
// 5. Add authentication/authorization
// 6. Document the new tool
```

## Decision Tree

```
Need to implement functionality?
│
├─ Are there MCP tools for this functionality?
│  │
│  ├─ YES: Does the existing tool meet my needs?
│  │  │
│  │  ├─ YES: Use the existing tool via single MCP server ✅
│  │  │
│  │  └─ NO: Should this be an MCP tool?
│  │     │
│  │     ├─ YES: Add tool to appropriate domain folder (internal/<domain>/mcp/)
│  │     │      Register in cmd/mcp-server/main.go
│  │     │
│  │     └─ NO: Implement as internal function
│  │
│  └─ NO: Should this functionality be an MCP tool?
│     │
│     ├─ YES: Create new tool in domain folder (internal/<domain>/mcp/)
│     │      Register in cmd/mcp-server/main.go
│     │
│     └─ NO: Implement as internal service
```


## When to Create New MCP Tools

### Criteria for MCP Tool Creation

Create a new MCP tool when the functionality:
- **Is tenant-facing**: Tenants or agents need to invoke it
- **Crosses service boundaries**: Requires coordination between components
- **Needs standardization**: Should follow consistent interface patterns
- **Requires authorization**: Needs tenant context and permission checks
- **Should be discoverable**: Agents should be able to find and use it
- **Is reusable**: Multiple consumers will need this capability

### Do NOT Create MCP Tool When:
- **Internal implementation detail**: Only used within a single service
- **Performance-critical path**: MCP overhead is unacceptable
- **Temporary/experimental**: Not ready for production use
- **Already exists**: Duplicate functionality in another domain's MCP tools

## Single MCP Server Implementation Pattern

### MCP Server Structure (Go)

```go
// File: cmd/mcp-server/main.go
package main

import (
    "context"
    "github.com/gin-gonic/gin"
    
    // Import domain MCP tools
    tenantTools "internal/tenant-core/mcp"
    agentTools "internal/agent-core/mcp"
    clusterTools "internal/cluster-core/mcp"
)

type Server struct {
    tools map[string]Tool
    db    *sql.DB
    keto  *keto.Client
}

func NewServer(db *sql.DB, keto *keto.Client) *Server {
    s := &Server{
        tools: make(map[string]Tool),
        db:    db,
        keto:  keto,
    }
    
    // Register tools from all domains
    s.RegisterTools(tenantTools.GetTools(db))
    s.RegisterTools(agentTools.GetTools(db))
    s.RegisterTools(clusterTools.GetTools(db))
    
    return s
}
```


### Standard Tool Implementation

```go
// File: internal/tenant-core/mcp/tenant_tools.go
package mcp

import (
    "context"
    "database/sql"
)

type CreateTenantTool struct {
    db *sql.DB
}

func NewCreateTenantTool(db *sql.DB) *CreateTenantTool {
    return &CreateTenantTool{db: db}
}

func (t *CreateTenantTool) Name() string {
    return "create_tenant"
}

func (t *CreateTenantTool) Description() string {
    return "Create a new tenant in the platform"
}

func (t *CreateTenantTool) InputSchema() map[string]interface{} {
    return map[string]interface{}{
        "type": "object",
        "properties": map[string]interface{}{
            "name": map[string]interface{}{
                "type":        "string",
                "description": "Tenant name",
            },
            "tier": map[string]interface{}{
                "type":        "string",
                "enum":        []string{"basic", "standard", "premium", "enterprise"},
                "description": "Tenant tier",
            },
        },
        "required": []string{"name", "tier"},
    }
}

// GetTools returns all tenant-core MCP tools
func GetTools(db *sql.DB) []Tool {
    return []Tool{
        NewCreateTenantTool(db),
        NewGetTenantTool(db),
        NewUpdateTenantTool(db),
        // ... other tenant tools
    }
}
```


func (t *CreateTenantTool) Execute(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    // 1. Extract tenant context from JWT (already in ctx)
    tenantID := ctx.Value("tenant_id").(string)
    
    // 2. Validate parameters
    name, ok := params["name"].(string)
    if !ok {
        return nil, errors.New("name is required")
    }
    
    tier, ok := params["tier"].(string)
    if !ok {
        return nil, errors.New("tier is required")
    }
    
    // 3. Check authorization (via Ory Keto)
    // Only platform admins can create tenants
    
    // 4. Execute business logic
    tenant, err := t.createTenant(ctx, name, tier)
    if err != nil {
        return nil, err
    }
    
    // 5. Return result
    return map[string]interface{}{
        "tenant_id": tenant.ID,
        "name":      tenant.Name,
        "tier":      tenant.Tier,
        "created_at": tenant.CreatedAt,
    }, nil
}

func (t *CreateTenantTool) createTenant(ctx context.Context, name, tier string) (*Tenant, error) {
    // Use sqlc generated queries
    // Include tenant isolation
    // Emit NATS events
    // Return tenant object
}
```


## MCP Tool Best Practices

### 1. Always Include Tenant Context
```go
// Extract tenant context from JWT (set by middleware)
tenantID := ctx.Value("tenant_id").(string)
tenantTier := ctx.Value("tenant_tier").(string)

// Use tenant context in all operations
result, err := queries.GetAgentByTenant(ctx, GetAgentByTenantParams{
    TenantID: tenantID,
    AgentID:  agentID,
})
```

### 2. Validate Authorization
```go
// Check permissions via Ory Keto
allowed, err := ketoClient.Check(ctx, &ketoapi.RelationQuery{
    Namespace: "tenants",
    Object:    tenantID,
    Relation:  "admin",
    Subject:   &ketoapi.Subject{ID: userID},
})

if err != nil || !allowed {
    return nil, ErrUnauthorized
}
```

### 3. Use Type-Safe Database Queries
```go
// Use sqlc generated queries (not raw SQL) for Go APIs
// File: queries.sql
-- name: GetAgentByTenant :one
SELECT * FROM agents 
WHERE tenant_id = $1 AND agent_id = $2;

// Generated Go code
agent, err := queries.GetAgentByTenant(ctx, GetAgentByTenantParams{
    TenantID: tenantID,
    AgentID:  agentID,
})

// Note: PostgREST is used separately for dashboard APIs
// GET /agents?tenant_id=eq.123&agent_id=eq.456
// PostgREST automatically respects RLS policies
```


### 4. Emit Events for Async Processing
```go
// Emit events to NATS for downstream processing
event := TenantCreatedEvent{
    TenantID:  tenant.ID,
    Name:      tenant.Name,
    Tier:      tenant.Tier,
    Timestamp: time.Now(),
}

err = natsClient.Publish("opensbt_tenantCreated", event)
if err != nil {
    log.WithError(err).Error("Failed to publish opensbt_tenantCreated event")
}
```

### 5. Include Comprehensive Logging
```go
// Always log with tenant context
log.WithFields(log.Fields{
    "tenant_id": tenantID,
    "user_id":   userID,
    "tool":      "create_tenant",
    "params":    params,
}).Info("Executing create_tenant tool")

// Log results
log.WithFields(log.Fields{
    "tenant_id":     tenantID,
    "new_tenant_id": result.TenantID,
    "duration_ms":   duration.Milliseconds(),
}).Info("create_tenant completed successfully")
```

### 6. Implement Proper Error Handling
```go
// Return structured errors
type MCPError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Details map[string]interface{} `json:"details,omitempty"`
}

// Example error responses
return nil, &MCPError{
    Code:    "TENANT_ALREADY_EXISTS",
    Message: "A tenant with this name already exists",
    Details: map[string]interface{}{
        "name": name,
    },
}
```


### 7. Add Metrics and Monitoring
```go
// Track tool execution metrics
toolDuration := prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name: "mcp_tool_duration_seconds",
        Help: "Duration of MCP tool execution",
    },
    []string{"tool", "tenant_id", "status"},
)

// Record metrics
start := time.Now()
result, err := tool.Execute(ctx, params)
duration := time.Since(start)

status := "success"
if err != nil {
    status = "error"
}

toolDuration.WithLabelValues(
    tool.Name(),
    tenantID,
    status,
).Observe(duration.Seconds())
```

## Testing MCP Tools

### E2E Testing with Testcontainers-Go

```go
func TestCreateTenantTool(t *testing.T) {
    // Start PostgreSQL with Testcontainers
    ctx := context.Background()
    pgContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:15-alpine"),
    )
    require.NoError(t, err)
    defer pgContainer.Terminate(ctx)
    
    // Get connection string
    connStr, err := pgContainer.ConnectionString(ctx)
    require.NoError(t, err)
    
    // Connect to database
    db, err := sql.Open("postgres", connStr)
    require.NoError(t, err)
    defer db.Close()
```


    // Run migrations
    err = runMigrations(db)
    require.NoError(t, err)
    
    // Create MCP tool
    tool := NewCreateTenantTool(db)
    
    // Test: Create tenant
    result, err := tool.Execute(ctx, map[string]interface{}{
        "name": "test-tenant",
        "tier": "basic",
    })
    require.NoError(t, err)
    
    // Verify result
    assert.NotEmpty(t, result["tenant_id"])
    assert.Equal(t, "test-tenant", result["name"])
    assert.Equal(t, "basic", result["tier"])
    
    // Test: Duplicate tenant should fail
    _, err = tool.Execute(ctx, map[string]interface{}{
        "name": "test-tenant",
        "tier": "basic",
    })
    assert.Error(t, err)
    assert.Contains(t, err.Error(), "already exists")
}
```

## Quick Reference Checklist

Before writing code for any component:

- [ ] Identify the component/domain (tenant-core, agent-core, cluster-core, etc.)
- [ ] Search for existing MCP tools in domain folder (`internal/<domain>/mcp/`)
- [ ] Check if domain has the tool you need
- [ ] Review tool schema and capabilities
- [ ] Test the tool via single MCP server with sample data
- [ ] Use the tool if it meets requirements
- [ ] Only write new code if no suitable tool exists
- [ ] If creating new tool, add to appropriate domain folder
- [ ] Register new tool in `cmd/mcp-server/main.go`
- [ ] Include tenant context, authorization, logging, metrics
- [ ] Write E2E tests with Testcontainers-Go
- [ ] Document the new tool in domain MCP README


## Common Scenarios

### Scenario 1: Implementing Tenant Management
**Task**: Add endpoint to update tenant tier

**MCP-First Approach:**
1. Search for `internal/tenant-core/mcp/`
2. Check for `update_tenant` tool
3. If exists: Use it in your endpoint handler via single MCP server
4. If not: Add `update_tenant` tool to `internal/tenant-core/mcp/`
5. Register tool in `cmd/mcp-server/main.go`
6. Expose via HTTP endpoint that calls MCP tool

### Scenario 2: Agent Execution
**Task**: Implement agent execution API

**MCP-First Approach:**
1. Search for `internal/agent-core/mcp/`
2. Check for `execute_agent` tool
3. Review tool capabilities (sync vs async, parameters, etc.)
4. Use tool via single MCP server instead of implementing execution logic
5. Tool handles: AgentSandbox sandboxing, NATS queuing, Kagents workflows

### Scenario 3: Cluster Provisioning
**Task**: Add cluster creation endpoint

**MCP-First Approach:**
1. Search for `internal/cluster-core/mcp/`
2. Check for `create_cluster` tool
3. Verify it supports CAPI/CAPH and Hetzner
4. Use tool via single MCP server for cluster provisioning
5. Tool handles: Crossplane, CAPI, ArgoCD deployment

### Scenario 4: Metrics and Monitoring
**Task**: Add endpoint to query tenant metrics

**MCP-First Approach:**
1. Search for `internal/monitoring-core/mcp/`
2. Check for `query_metrics` or `get_tenant_metrics` tool
3. Use tool via single MCP server to query VictoriaMetrics
4. Tool handles: PromQL queries, tenant filtering, time ranges
5. Return formatted results to client

### Scenario 5: GitOps Deployment
**Task**: Trigger deployment for tenant

**MCP-First Approach:**
1. Search for `internal/gitops-core/mcp/`
2. Check for `sync_application` tool
3. Use tool via single MCP server to trigger ArgoCD sync
4. Tool handles: ArgoCD API, tenant context, status monitoring
5. Return deployment status to client

## Benefits of MCP-First Approach

### Consistency
- All platform capabilities follow same interface pattern
- Standardized authentication and authorization
- Consistent error handling and logging
- Uniform metrics and monitoring

### Discoverability
- Agents can discover available tools via MCP protocol
- Self-documenting via tool schemas
- Easy to understand what platform can do
- Reduces need for extensive documentation


### Reusability
- Tools can be used by multiple consumers
- HTTP endpoints, CLI tools, agents all use same tools
- Reduces code duplication
- Single source of truth for functionality

### Testability
- Tools are isolated and testable
- E2E tests with Testcontainers-Go
- Mock MCP servers for integration tests
- Clear input/output contracts

### Maintainability
- Changes to functionality in one place
- Clear separation of concerns
- Easy to add new capabilities
- Version tools independently

## Summary

**Always follow this sequence:**
1. **Discover**: Search for existing MCP tools in domain folders
2. **Evaluate**: Check if tools meet your requirements
3. **Use**: Integrate existing tools via the single MCP server
4. **Extend**: Add new tools to domain folders if needed
5. **Register**: Register new tools in `cmd/mcp-server/main.go`
6. **Document**: Update domain MCP documentation
7. **Test**: Write E2E tests for new tools

**Remember**: MCP First, Code Second. The platform's capabilities should be exposed through MCP tools in domain folders, all accessible via the single MCP server, making them discoverable, reusable, and consistent across all consumers.

