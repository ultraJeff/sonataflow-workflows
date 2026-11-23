# SonataFlow Workflows

Orchestrator workflows for RHDH on tallgeese (OCP 4.21).

## Building workflow images

Workflows run in **gitops profile** with pre-built container images. Build using the SonataFlow builder image — version must match the Serverless Logic operator (currently 1.37.0):

```bash
podman build -t registry.ultra.lab/sonataflow/<workflow>:latest -f <workflow>/Containerfile <workflow>/
podman login --tls-verify=false registry.ultra.lab -u admin -p <nexus-password>
podman push --tls-verify=false registry.ultra.lab/sonataflow/<workflow>:latest
```

Build happens on wing (192.168.8.100) since it has registry.redhat.io access. Must `podman login registry.redhat.io` first.

## Build-time vs runtime properties

**Most properties are build-time only** — they get compiled into the Quarkus app and cannot be overridden by the operator's managed-props ConfigMap at runtime. These MUST be in `application.properties` before building:

- `kogito.service.url` — workflow callback URL
- `mp.messaging.outgoing.*.connector` and `*.url` — messaging channel config
- `quarkus.rest-client.<name>.url` — OpenAPI function endpoints
- `quarkus.openapi-generator.<name>.auth.*` — OpenAPI auth config

## Use OpenAPI functions, NOT custom REST

Do NOT use `type: custom` with `operation: "rest:post:/path"`. The Vert.x WebClient used by custom REST functions has incompatibilities with nginx reverse proxies (unexplained 400 errors).

Instead, write a minimal OpenAPI spec for the target API and reference it via `workflow-uri-definitions`:

```yaml
extensions:
  - extensionid: workflow-uri-definitions
    definitions:
      myapi: "specs/my-openapi.yaml"
functions:
  - name: myFunction
    operation: myapi#operationId
```

Auth is handled via `quarkus.openapi-generator.<alias>.auth.BearerToken.bearer-token` in properties.

## Registry

Images push to `registry.ultra.lab` (nginx TLS proxy → Nexus Docker connector on port 5000). The cluster trusts the `ultra-lab-ca` wildcard cert.

## Deploying

Deployed by the `sonataflow-workflows` Argo CD Application (openshift-bootstrap-gitops `applications/sonataflow-workflows.yaml`) from the root `kustomization.yaml`. It uses manual sync: commit, then sync the app. Sync waves aren't needed; the operator retries until the configmaps/secret exist. The operator adds an ownerReference from the SonataFlow CR to `<workflow>-props`, so deleting a CR garbage-collects that ConfigMap and the Argo app goes OutOfSync; re-sync to restore it. After deploying, the workflow pod must start after the shared data-index service is ready, or the workflow won't appear in the RHDH orchestrator UI.

## Avoid multi-key maps in flows (operator hot loop)

As of logic-operator 1.39.2, a flow containing any free-form map with 2+ keys (e.g. `functionRef.arguments: {payload: ..., recipients: ...}`) sends the operator into an endless status-update loop. `status.flowCRC` is computed by gob-encoding the flow, and gob map order is random. `namespace-approval` and `mta-v7` are **disabled** for this reason; see the NOTE at the top of each SonataFlow CR. A new workflow should either keep every map to a single key or be watched after deploy: if `resourceVersion` keeps climbing while `generation` stays put, it's looping.
