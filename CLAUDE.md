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

Apply manifests in order: secret → configmaps → sonataflow CR. After deploying, the workflow pod must start after the shared data-index service is ready, or the workflow won't appear in the RHDH orchestrator UI.
