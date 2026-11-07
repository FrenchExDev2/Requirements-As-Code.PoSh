# 11. Execution models and deployment

Sections 6–7 describe a Runtime that is linked directly into a Host process. This section adds
two more ways to run the same Runtime — as a long-lived **Service** on a VM, or as a
**container** — without changing anything in sections 3–5: the data model and the verification
algorithm are identical no matter where the Runtime executes.

## 11.1 Three execution models

- **`embedded`** — Contracts+Runtime are linked directly into the Host process (sections 6, 7).
  No network boundary, no extra deployment surface, the default, and the only model with zero
  external dependency — consistent with [1.3](01-overview.md#13-non-goals)'s non-goals.
- **`service`** — the Runtime runs as its own long-running process, exposing the **Verification
  Service API** ([11.2](#112-verification-service-api)) over a network or local transport; a
  Host becomes a client of that API ([11.3](#113-service-clients)) instead of linking the
  Runtime. `service` covers both remaining deployment targets — a VM and a container are
  **packaging profiles** of the same model ([11.4](#114-vm-deployment-profile),
  [11.5](#115-container-deployment-profile)): the API and the Runtime's internal behavior never
  change between them, only how the process is started, supervised and reached.

A binding MAY implement `embedded` only, `service` only, or both; [11.7](#117-execution-conformance)
defines how that is declared.

## 11.2 Verification Service API

A `service`-mode Runtime MUST expose this API over HTTP+JSON (the required baseline transport,
reachable from any binding without extra tooling) and MAY additionally expose a gRPC service
mirroring it 1:1 (field-for-field, same names) for bindings that want it.

| Method & path | Request body | Response | Purpose |
|---|---|---|---|
| `GET /v1/health` | — | `{ "status": "ok" }` | Liveness. |
| `GET /v1/ready` | — | `{ "status": "ok" }` or `503` | Readiness (can accept `verify` calls now). |
| `GET /v1/info` | — | `urn:frenchexdev:rac:v1:service-info` | Standard version, Conformance Level, declared [Execution Conformance](#117-execution-conformance) tags. |
| `POST /v1/catalogs` | a Catalog ([3.4](03-core-model.md#34-catalog)) | `urn:frenchexdev:rac:v1:catalog-registration` | Register a Catalog; resolves it per section 4 and reports load-time errors as `400`. |
| `POST /v1/catalogs/{id}/{version}/verify` | `urn:frenchexdev:rac:v1:verify-request` | a Verification Report ([5.3](05-verification-algorithm.md#53-verification-report)) | Resolve + verify synchronously. |
| `POST /v1/catalogs/{id}/{version}/verify-async` | `urn:frenchexdev:rac:v1:verify-request` | `{ "runId": "string" }` | Start verification, return immediately. |
| `GET /v1/runs/{runId}` | — | `urn:frenchexdev:rac:v1:run-status` | Poll an async run; `status: "running"` until the Report is ready. |

```json
{
  "$id": "urn:frenchexdev:rac:v1:verify-request",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "severityThreshold": { "$ref": "urn:frenchexdev:rac:v1:severity" },
    "shortCircuit": { "enum": ["fail-fast", "collect-all"] },
    "subjects": { "type": "array", "items": { "type": "string" } }
  }
}
```

```json
{
  "$id": "urn:frenchexdev:rac:v1:run-status",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["runId", "status"],
  "properties": {
    "runId": { "type": "string" },
    "status": { "enum": ["running", "completed", "failed"] },
    "report": { "$ref": "urn:frenchexdev:rac:v1:verification-report" }
  }
}
```

A malformed request or an unresolvable Catalog MUST return HTTP `400` with an
`urn:frenchexdev:rac:v1:api-error` body (`{ "error": "code", "message": "string", "details": [] }`)
listing the section-4 resolution errors — never a bare `500` that hides *why* resolution failed.

Transport security: a Service reachable over anything other than loopback MUST support TLS
(configured per [11.6](#116-service-mode-configuration)). A Service MAY require a bearer token
(`Authorization: Bearer <token>`) and MUST respond `401` when one is required and missing or
invalid; this standard does not mandate a specific identity provider or token format.

## 11.3 Service clients

Even a binding that only ships `embedded` Contracts+Runtime MAY offer a thin **Service Client**:
not a Runtime, just a caller of [11.2](#112-verification-service-api) that presents the same
calling convention a Host would use against an in-process Runtime (`Verify(catalog, config) →
Report`). From the Host's point of view the call looks identical whether it runs in-process or
crosses the network — this is what lets configuration ([section 8](08-configuration.md),
`executionModel`) choose `embedded` vs `service` without changing Host code, per
[1.4](01-overview.md#14-design-principles)'s "configuration over code" principle.

## 11.4 VM deployment profile

- The Service ([11.1](#111-three-execution-models), [11.2](#112-verification-service-api)) runs
  as a long-running OS process on a virtual or physical machine, started and supervised by the
  host OS's service manager (a `systemd` unit on Linux, a Windows Service via the hosting
  conventions already referenced in [7.3](07-language-bindings.md#73-powershell), or an
  equivalent elsewhere).
- It MUST read configuration the same way any other Host does ([section 8](08-configuration.md)),
  plus the service-mode fields in [11.6](#116-service-mode-configuration).
- It MUST terminate gracefully on the OS's standard stop signal (`SIGTERM` on POSIX, the
  service-stop control on Windows): stop accepting new `verify`/`verify-async` requests, let
  in-flight ones finish within `shutdownGraceSeconds` ([11.6](#116-service-mode-configuration)),
  then exit `0`. It MUST NOT drop an in-flight Verification Report silently — silence must never
  be the only evidence of an outcome, the same rule [5.4](05-verification-algorithm.md#54-severity-and-thresholds)
  states for Check results.
- Logs MUST go to the OS's standard service log sink (journal, Windows Event Log, or
  stdout/stderr when run in the foreground) in a line-structured, machine-parseable form; this
  standard does not mandate a specific log schema in v1.

## 11.5 Container deployment profile

The Service is packaged as an OCI-compliant container image. The image:

- MUST expose the Verification Service API on the port given by `bindPort`
  ([11.6](#116-service-mode-configuration), default `8743`);
- MUST implement `GET /v1/health` and `GET /v1/ready` as its container health check
  (`HEALTHCHECK` instruction, or the orchestrator-native probe equivalent);
- MUST read `RAC_CONFIG` from an environment variable or a mounted file, and MUST accept
  Catalog data via a mounted volume as well as `POST /v1/catalogs`, so Catalogs need not be
  baked into the image;
- MUST handle `SIGTERM` exactly as the VM profile ([11.4](#114-vm-deployment-profile)) — a
  container runtime sends the same signal before a hard kill;
- SHOULD carry OCI annotation labels identifying the standard:
  `org.frenchexdev.rac.standard-version=1.0.0`, `org.frenchexdev.rac.conformance-level=<L0..L3>`,
  `org.frenchexdev.rac.binding=<language>`.

Running the Service under an orchestrator (Kubernetes, Nomad, Compose) is an ordinary
consequence of it being a conformant container; this standard adds no orchestrator-specific
requirement beyond the health/readiness/shutdown contract above, which already matches what
every orchestrator expects. A **sidecar** deployment (one Service per Host instance, reachable
over `localhost`) and a **centralized** deployment (one Service shared across a cluster) are
both conformant — the choice is an operational decision, not a standard requirement.

## 11.6 Service-mode configuration

[Section 8](08-configuration.md)'s configuration schema carries the fields that apply only when
`executionModel: service`: `serviceEndpoint` (client side — the base URL of a running Service),
and, server side, `bindAddress`, `bindPort`, `shutdownGraceSeconds` and `tls` (see
[8.1](08-configuration.md#81-schema) for their exact shape and defaults).

```yaml
executionModel: service
serviceEndpoint: "https://rac.internal:8743"
```

A Host configured with `executionModel: service` MUST use a Service Client
([11.3](#113-service-clients)) instead of an in-process Runtime. A Host left at the default,
`embedded`, MUST NOT require any Service to be reachable — embedded stays conformant with zero
network dependency.

## 11.7 Execution conformance

Alongside the Conformance Level ([9.1](09-conformance-and-testing.md#91-conformance-levels)), a
binding declares an orthogonal **Execution Conformance** tag set, reported in `GET /v1/info`
when applicable:

- **`Embeddable`** — implements the in-process API of sections 6–7.
- **`Serviceable`** — implements [11.2](#112-verification-service-api) and is deployable per
  [11.4](#114-vm-deployment-profile).
- **`Containerizable`** — additionally satisfies [11.5](#115-container-deployment-profile)'s
  image contract.

A binding MAY claim any non-empty subset. `Serviceable`/`Containerizable` without `Embeddable`
is valid (a Service-only binding, e.g. a Rust binary with no library surface) — but it MUST
still pass the cross-language conformance vectors
([7.1](07-language-bindings.md#71-conformance-test-vectors)) by calling the Verification
Service API, since those vectors assert on the Verification Report, not on how it was produced.
