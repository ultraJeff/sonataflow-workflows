# MTA v7 Analysis Workflow

Runs a Migration Toolkit for Applications (MTA) analysis against a git repository and reports the results back to RHDH.

## Source

Extracted from the `orchestrator-workflows/mta-v7` Helm chart v1.6.0. The Helm chart bundles MTA operator resources (Namespace, OperatorGroup, Subscription, Tackle CR) that conflict with an existing MTA installation, so the workflow manifests were extracted and applied individually.

## Prerequisites

1. **MTA operator and Tackle instance** — must be deployed in `openshift-mta` (managed separately via ArgoCD)
2. **PostgreSQL** — see `../infrastructure/postgresql.yaml`
3. **SonataFlowPlatform** — data-index and job services running in `sonataflow-infra`

## Configuration

### Environment Variables

The SonataFlow CR needs these env vars in `spec.podTemplate.container.env`:

| Variable | Value | Source |
|---|---|---|
| `MTA_URL` | `http://mta-ui.openshift-mta.svc.cluster.local:8080` | Hardcoded — MTA UI service |
| `BACKSTAGE_NOTIFICATIONS_URL` | `http://backstage-developer-hub.rhdh.svc.cluster.local:80/api/notifications` | Hardcoded — RHDH notifications API |
| `NOTIFICATIONS_BEARER_TOKEN` | RHDH service token | From secret `mta-analysis-v7-secrets` |
| `MTA_ADMIN_PASSWORD` | MTA Keycloak admin password | From secret `mta-analysis-v7-secrets` (copied from `mta-keycloak-rhbk` in `openshift-mta`) |

**Note:** The Helm chart does not inject `MTA_URL` or `BACKSTAGE_NOTIFICATIONS_URL` — these must be added manually to the SonataFlow CR. The `application.properties` references them as `${MTA_URL}` and `${BACKSTAGE_NOTIFICATIONS_URL}`.

### MTA Authentication

MTA uses its own Keycloak instance (`mta-rhbk-service` in `openshift-mta`). The workflow authenticates via OIDC resource owner password grant. This is configured in `application.properties` via:

```properties
quarkus.oidc-client.mta.auth-server-url = https://mta-rhbk-service.openshift-mta.svc.cluster.local:8443/realms/tackle
quarkus.oidc-client.mta.client-id = tackle-ui
quarkus.oidc-client.mta.grant.type = password
quarkus.oidc-client.mta.grant-options.password.username = admin
quarkus.oidc-client.mta.grant-options.password.password = ${MTA_ADMIN_PASSWORD}
quarkus.oidc-client.mta.tls.verification = none
quarkus.rest-client.mta_json.auth.oauth2.client-name = mta
```

The `MTA_ADMIN_PASSWORD` is sourced from the `mta-keycloak-rhbk` secret in `openshift-mta` and must be copied into the `mta-analysis-v7-secrets` secret in `sonataflow-infra`.

### Secrets Setup

The `mta-analysis-v7-secrets` secret needs:

```bash
# Get MTA Keycloak password
MTA_PASS=$(oc get secret mta-keycloak-rhbk -n openshift-mta -o jsonpath='{.data.password}' | base64 -d)

# Patch the workflow secret
oc patch secret mta-analysis-v7-secrets -n sonataflow-infra \
  --type merge -p "{\"stringData\":{\"MTA_ADMIN_PASSWORD\":\"$MTA_PASS\"}}"
```

## RBAC

The Orchestrator RBAC permissions must be set in RHDH (see `../SETUP.md`).

## Usage

1. Open RHDH → Orchestrator
2. Find "MTA Analysis v7.x" → Run
3. Enter a git repository URL (e.g. `https://github.com/konveyor/example-applications.git`)
4. Set "Export to Issue Manager" to `false` (unless Jira is configured)
5. Add a recipient for notifications (e.g. `user:default/admin`)

## Status: NOT WORKING WITH MTA 8.x

**This workflow does not work with MTA 8.1.** The workflow (Helm chart v1.6.0) was built and tested against MTA 6.2.x / Konveyor 0.2.x. MTA 8.x has breaking changes that are not accounted for:

1. **Authentication:** MTA 8.x uses RHBK (Keycloak) for all Hub API auth. The older MTA versions had no auth or simple basic auth. The workflow's `application.properties` has no OIDC client config, and the basic auth placeholders (`username`/`password`) don't work against MTA 8.x.

2. **Keycloak context path:** MTA 8.x's RHBK uses the legacy `/auth` context path. The realm is `mta`, not `tackle`. The client is `admin-cli` (public). The MTA realm admin user has a separate password from the Keycloak admin.

3. **NetworkPolicies:** MTA 8.x deploys `mta-deny-all` and `mta-namespace` NetworkPolicies that block all cross-namespace traffic. A `allow-sonataflow-to-mta` NetworkPolicy must be created to allow the workflow pod in `sonataflow-infra` to reach MTA services in `openshift-mta`.

4. **Service endpoints:** The Hub API is on the `mta-hub` service (port 8080), not the `mta-ui` service. The `mta-ui` service returns 302 redirects to Keycloak for all requests including `/hub` paths.

5. **Helm chart conflicts:** The chart bundles the MTA operator resources (Namespace, OperatorGroup, Subscription, Tackle CR, CRDs) which conflict with an existing MTA 8.x installation. Must use `--skip-crds` and filter out existing resources.

### What we tried

- OIDC client credentials grant against MTA's Keycloak (`mta-rhbk-service:8443/auth/realms/mta`)
- Basic auth against the Hub API
- Bearer token with the Hub addon token
- All returned 401 from the Hub API

### What's needed

An updated workflow that supports MTA 8.x's auth model, or documentation from the orchestrator team on configuring the OIDC client for the MTA Hub API in MTA 8.x. Check `rhdhorchestrator/serverless-workflows` for updates.

## Known Issues

- The Helm chart does not inject required env vars (`MTA_URL`, `BACKSTAGE_NOTIFICATIONS_URL`) — they must be added manually to the SonataFlow CR
- MTA uses a separate Keycloak instance from RHDH, requiring cross-namespace secret management
- The `tackle-ui` client in MTA's Keycloak is a public client — the `credentials.secret` is empty
- Flyway migration fails on restart if schema already exists — needs `quarkus.flyway.baseline-on-migrate=true`

## TODO

- [ ] **Blocker:** Get MTA 8.x auth working — either via an updated workflow chart or custom OIDC config
- [ ] Use a shared Keycloak/RHBK instance instead of MTA's built-in one — MTA deploys its own RHBK operator and Keycloak instance, which is redundant when RHDH already has Keycloak. Consolidating to a single Keycloak would eliminate the cross-namespace secret copying and simplify auth management.
- [ ] Check for updated Helm chart versions that support MTA 8.x

## Building a Custom MTA 8.x Workflow

If the upstream chart remains unmaintained, here's what a from-scratch MTA 8.x workflow would require:

### 1. Auth: Get an OIDC token from MTA's Keycloak

MTA 8.x Keycloak details (on tallgeese):
- Service: `mta-rhbk-service.openshift-mta.svc.cluster.local:8443`
- Context path: `/auth` (legacy Keycloak path)
- Realm: `mta`
- Client: `admin-cli` (public, no secret)
- Grant type: `password`
- Admin user: `admin` (password stored in `mta-analysis-v7-secrets`)
- Master realm works with the `mta-keycloak-rhbk` secret password; MTA realm admin has a separately managed password

Token endpoint: `https://mta-rhbk-service.openshift-mta:8443/auth/realms/mta/protocol/openid-connect/token`

### 2. API: Talk to the Hub service directly

- Service: `mta-hub.openshift-mta.svc.cluster.local:8080` (NOT `mta-ui`)
- The Hub API does not use a `/hub` prefix — API paths are directly on the root (e.g. `/applications`, `/taskgroups`)
- Requires a Bearer token from the MTA Keycloak
- The `mta-ui` service redirects all requests to Keycloak (302) — don't use it for API calls

### 3. NetworkPolicy

MTA 8.x deploys a `mta-deny-all` NetworkPolicy. Must create an allow policy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-sonataflow-to-mta
  namespace: openshift-mta
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: sonataflow-infra
  policyTypes:
    - Ingress
```

### 4. OpenAPI Spec

The existing `specs/mta.json` was generated from MTA 6.x/7.x. For MTA 8.x:
- Export the current spec from the running Hub: `curl -sk -H "Authorization: Bearer <token>" http://mta-hub.openshift-mta:8080/openapi.json`
- Or use the Konveyor tackle2-hub repo: `https://github.com/konveyor/tackle2-hub`
- Compare with the existing spec to identify changed endpoints, request/response schemas

### 5. Workflow Structure

The core flow is simple — the existing `mta.sw.yaml` states are mostly reusable:
```
CreateApplication → CreateTaskGroup → SubmitAnalysis → PollResults (loop) → GetAnalysis → CountIncidents → Notify
```

The main changes needed:
- Update function definitions to match MTA 8.x API endpoints
- Add OIDC token acquisition (either via Quarkus OIDC client or a custom function state)
- Update the OpenAPI spec
- Test each API call against the live MTA 8.x instance

### 6. Building the Image

Use the `dev` profile for testing (no pre-built image needed):
```yaml
annotations:
  sonataflow.org/profile: dev
```

For production, build with the SonataFlow builder or a Tekton pipeline and push to a registry.

### 7. Estimated Effort

- OpenAPI spec update + validation: 1-2 days
- Auth integration (OIDC client config): 1 day
- Workflow state updates + testing: 2-3 days
- Image build + deployment: 1 day
- **Total: ~1 week**
