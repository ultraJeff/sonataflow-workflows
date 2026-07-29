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

## Known Issues

- The Helm chart does not inject required env vars — they must be added manually
- MTA uses a separate Keycloak instance from RHDH, requiring cross-namespace secret copying
- The `tackle-ui` client in MTA's Keycloak is a public client — the `credentials.secret` is empty

## TODO

- [ ] Use a shared Keycloak/RHBK instance instead of MTA's built-in one — MTA deploys its own RHBK operator and Keycloak instance, which is redundant when RHDH already has Keycloak. Consolidating to a single Keycloak would eliminate the cross-namespace secret copying and simplify auth management. This requires configuring MTA to use an external identity provider instead of its embedded one.
