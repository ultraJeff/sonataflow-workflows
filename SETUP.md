# SonataFlow Workflows Setup

## Prerequisites

### 1. SonataFlowPlatform CR
Must exist in `sonataflow-infra` namespace with data-index and job services enabled:
```yaml
spec:
  services:
    dataIndex:
      enabled: true
    jobService:
      enabled: true
```
This is managed in the `openshift-bootstrap-gitops` repo at `cluster-configs/serverless/sonataflow-platform.yaml`.

### 2. PostgreSQL
Helm chart workflows require a PostgreSQL instance in `sonataflow-infra`. See `infrastructure/postgresql.yaml`. The secret `sonataflow-psql-postgresql` must exist with keys `postgres-username` and `postgres-password` before deploying workflows.

### 3. RHDH Orchestrator Backend
The orchestrator backend plugin in RHDH must be configured to point to the data-index service:
```yaml
orchestrator:
  dataIndexService:
    url: http://sonataflow-platform-data-index-service.sonataflow-infra
  sonataflow:
    baseUrl: http://sonataflow-platform-data-index-service.sonataflow-infra
```
**Note:** The service listens on port 80 (not 8080). Do not include a port number.

### 4. RBAC
The orchestrator uses these permission names (RHDH 1.9+):
```csv
p, role:default/plugins, orchestrator.workflow, read, allow
p, role:default/plugins, orchestrator.workflow.use, update, allow
```
**Important:** The action for `orchestrator.workflow.use` is `update`, not `use`. The generic permissions act as wildcards for all workflows.

## Installing Workflows via Helm

```bash
helm repo add orchestrator-workflows https://rhdhorchestrator.io/serverless-workflows
helm repo update

# List available charts
helm search repo orchestrator-workflows

# Install (e.g. greeting)
helm install greeting orchestrator-workflows/greeting -n sonataflow-infra
```

## Available Helm Charts
- `greeting` — simple hello world, good for testing
- `mta-v7` — MTA analysis workflow
- `create-ocp-project` — namespace provisioning
- `modify-vm-resources` — OpenShift Virtualization VM management
- `request-vm-cnv` — VM provisioning
- `mtv-plan` / `mtv-migration` — VM migration assessment and execution

## Troubleshooting

### Workflows not showing in RHDH UI
1. Check RBAC — the permission name is `orchestrator.workflow` with action `read` (not `orchestrator.workflow.read`)
2. Check data-index has the workflow registered: `oc exec deployment/sonataflow-platform-data-index-service -n sonataflow-infra -- curl -s http://localhost:8080/graphql -H "Content-Type: application/json" -d '{"query":"{ ProcessDefinitions { id } }"}'`
3. Check RHDH logs for permission denials: `oc logs deployment/backstage-developer-hub -n rhdh | grep 'orchestrator.*DENY'`

### Pod CreateContainerConfigError
Usually means the `sonataflow-psql-postgresql` secret is missing. Create the secret and the PostgreSQL deployment first.
