<!--
  ~ SPDX-License-Identifier: Apache-2.0
-->

# 0002 — forge-api-schema

- **Status:** Draft
- **Owner:** Nathan Klick
- **Date:** 2026-09-15
- **Summary:** `forge-api-schema` is the single source of truth for Forge's HTTP API contracts, written
  first in OpenAPI 3.1, and for desired-state schemas in JSON Schema 2020-12. It publishes the documents,
  generated Go models with no third-party dependencies, and the lint and breaking-change checks that
  every other repository relies on.

> An initial draft with concrete proposals. Formats, tools, layout, and policies are proposals to argue
> with, bounded by the [Resolved decisions](0001-project-repositories.md#resolved-decisions) in 0001.
> Conventions other repositories depend on are summarized in [CONVENTIONS.md](CONVENTIONS.md).

## Context & goals

[0001](0001-project-repositories.md#repository-inventory) makes `forge-api-schema` the home of "the
inter-service and client schema" and requires that the wire contract live there once, with clients using
the generated `forge-sdk` rather than re-deriving types
([Naming & conventions](0001-project-repositories.md#naming--conventions)). It also publishes the JSON
Schemas for desired-state documents with Kubernetes-style `apiVersion` and `kind`
([Desired-state format](0001-project-repositories.md#desired-state-format)). It is first in the
[build order](0001-project-repositories.md#sequencing--phases), so its choices constrain every other
repository.

**Goals**

- One contract format and API style for every Forge HTTP API, internal and external.
- Versioning, deprecation, and error rules that work across independently released repositories.
- Go models with no third-party dependencies, plus embedded documents that services serve and test against.
- Catch drift and breaking changes in CI before they reach `forge-sdk` or a service.

**Non-goals**

- The agent plugin contract — gRPC over `hashicorp/go-plugin`, owned by
  [0013](0013-forge-agent-plugin-sdk.md).
- Client behavior such as retries, authentication, and TLS — [0003](0003-forge-sdk.md).
- Token formats, RBAC, and the enrollment token encoding — [0006](0006-forge-identity.md).
- Gateway routing, rate limits, and principal propagation — [0008](0008-forge-gateway.md).
- The resources each service exposes — each service's own document.
- SAML and OIDC protocol endpoints in `forge-sso` and `forge-identity`; they follow their standards.

## Proposal

### Responsibilities

- One OpenAPI 3.1 document per service per API version, plus a shared components document.
- One JSON Schema 2020-12 file per desired-state `kind` per `apiVersion`.
- Generated Go models for API documents; Go types for desired-state kinds.
- Embedded access to every document through `embed.FS`, so services serve the exact contract.
- The Forge vacuum ruleset, the oasdiff compatibility policy, examples, and drift checks.

### Interfaces

#### Contract format and API style

Proposed: **contract-first OpenAPI 3.1, REST with JSON for every API.** Internal calls use the same
contract over mutual TLS; there is no service-to-service gRPC. Rationale:

- **Matches the starters.** `go-echo-starter` already serves `/openapi.yaml` and `/openapi.json` and
  checks spec drift in CI (`800-call-openapi-drift.yaml`), and third-party clients need nothing beyond
  HTTP and JSON.
- **One schema dialect.** OpenAPI 3.1's Schema Object is JSON Schema 2020-12, so API schemas and
  desired-state schemas share a dialect and API bodies can `$ref` desired-state schemas.
- **Tools support 3.1.** oapi-codegen v2.8.0 supports OpenAPI 3.0 and 3.1, including `type: [T, "null"]`;
  kin-openapi lists 3.1; oasdiff lists 3.1 and 3.2; vacuum is built on libopenapi, which lists 3.0 to 3.2.
- **Keeps gRPC out of services.** 0001's
  [telemetry decision](0001-project-repositories.md#logging-and-telemetry-forge-common) went out of its
  way to avoid gRPC; it stays confined to `forge-agent` and plugins.
- **Contract before code.** `go-echo-starter` generates OpenAPI 3.0 from route metadata after the code
  exists. `forge-sdk`, the gateway, and the first services are built in parallel, so the contract must
  come first.

#### Repository layout

```
forge-api-schema/
├── openapi/
│   ├── common/v1/components.yaml        # Problem, list envelope, parameters, security schemes
│   ├── identity/v1alpha1/openapi.yaml
│   ├── inventory/v1alpha1/openapi.yaml
│   └── provisioner/v1alpha1/openapi.yaml
├── schemas/forge.servercurio.com/v1alpha1/<kind>.schema.json
├── examples/                            # valid and invalid samples for every operation and kind
├── pkg/
│   ├── openapi/                         # embed.FS; Document(service, version string) ([]byte, error)
│   ├── schemas/                         # embed.FS; Schema(apiVersion, kind string) ([]byte, error)
│   ├── common/v1/                       # package commonv1 (generated)
│   ├── inventory/v1alpha1/              # package inventoryv1alpha1 (generated)
│   └── desiredstate/v1alpha1/           # package desiredstatev1alpha1 (hand-written kinds)
├── codegen/                             # one oapi-codegen config per document
├── rules/forge.vacuum.yaml              # Spectral-compatible ruleset
├── conformance/                         # nested Go module: validation and round-trip tests
└── Taskfile.yaml
```

Every API starts at `v1alpha1`. `forge-gateway` and `forge-sso` add documents only for JSON APIs they own.

#### Paths, operations, and extensions

Paths follow `/<service>/<version>/<plural-resource>[/{id}]` in kebab-case, and `operationId` is a
lowerCamelCase verb-noun unique within its document. Three Forge extensions carry metadata other
repositories act on:

| Extension            | Applies to      | Values                                   | Used by                            |
|----------------------|-----------------|------------------------------------------|------------------------------------|
| `x-forge-audience`   | operation       | array of `operator`, `agent`, `internal` | gateway routing, SDK docs, lint    |
| `x-forge-sensitive`  | schema property | `true`                                   | SDK redaction, logging rules       |
| `x-forge-idempotent` | POST operation  | `true`                                   | SDK retry policy                   |

`common/v1` defines two security schemes: `bearerAuth` (`type: http`, `scheme: bearer`) and `mutualTLS`
(a scheme type OpenAPI 3.1 defines). `operator` operations require `bearerAuth`, while `agent` and
`internal` operations require `mutualTLS`. The only exceptions are the agent enrollment operation, which
0001 makes the single route without a client certificate
([Agent enrollment](0001-project-repositories.md#agent-enrollment)), and service enrollment. Both declare
`security: []`. The gateway-to-service hop is always mutual TLS and is not modeled per operation.

```yaml
paths:
  /inventory/v1alpha1/endpoints:
    get:
      operationId: listEndpoints
      x-forge-audience: [operator]
      security: [{ bearerAuth: [] }]
      parameters:
        - $ref: "../../common/v1/components.yaml#/components/parameters/Limit"
        - $ref: "../../common/v1/components.yaml#/components/parameters/Cursor"
      responses:
        "200":
          description: One page of endpoints.
          content:
            application/json:
              schema: { $ref: "#/components/schemas/EndpointList" }
        default:
          $ref: "../../common/v1/components.yaml#/components/responses/Problem"
```

Lists use `limit` and an opaque `cursor`, returning `items` and `nextCursor`. Properties are
lowerCamelCase, timestamps are RFC 3339 in UTC, and IDs are opaque strings.

#### Error model

Every non-2xx response uses `application/problem+json` from
[RFC 9457](https://www.rfc-editor.org/rfc/rfc9457), defined once in `common/v1`, with three Forge
extension members:

```json
{
  "type": "<problem-base-url>/validation_failed",
  "title": "Request body failed validation",
  "status": 400,
  "code": "validation_failed",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [{ "pointer": "#/labels/site", "detail": "must be a lowercase DNS label" }]
}
```

`code` is a stable lower_snake_case identifier clients branch on, and `title` and `detail` are for
people. `errors` follows the shape of RFC 9457's own example. `traceId` connects a report to the trace.

#### Versioning and compatibility

- **API versions** follow Kubernetes-style stages, as desired-state documents already do (0001):
  `v1alpha1` may break between releases; beta breaks only by adding a new beta version beside the old
  one; a stable `v1` never breaks, so breaking changes ship as `v2` served side by side. This mirrors the
  [Kubernetes deprecation policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/).
- **Enforcement** — oasdiff compares every pull request with the documents at the last release tag and
  fails on a breaking change to a beta or stable document.
- **Deprecation** — `deprecated: true` in the document. The serving service sends `Deprecation`
  ([RFC 9745](https://www.rfc-editor.org/rfc/rfc9745)) and `Sunset`
  ([RFC 8594](https://www.rfc-editor.org/rfc/rfc8594)) headers.
- **Module version** — the Go module uses semantic versioning independently of API versions. Each
  document's `info.version` is set to the module release version by `task generate`, which the
  starters' semantic-release `prepareCmd` already runs.
- **Desired-state schemas** follow the same stages under the `forge.servercurio.com` group from 0001.
  Converting between versions belongs to `forge-provisioner` ([0011](0011-forge-provisioner.md)).

#### Go packages

- **API models** are generated by oapi-codegen with `generate: { models: true }`, one package per
  document, and `import-mapping` for references to `common/v1`. In a test on 2026-09-15 with v2.8.0,
  models using `string`, `integer`, `date-time`, arrays, and maps imported nothing outside the standard
  library. `format: uuid`, `date`, and `email` generated `github.com/oapi-codegen/runtime/types`
  imports, so the ruleset bans those formats and schemas use `pattern` instead.
- **Desired-state types** are hand-written Go structs with `json` and `yaml` tags. Conformance tests keep
  them honest against the schemas (see Testing).
- **Embedded documents** — `pkg/openapi` and `pkg/schemas` use [`embed`](https://pkg.go.dev/embed).
  Services serve `/openapi.yaml` from them instead of `go-echo-starter`'s route-metadata generator.
- **The root module's `go.mod` has no requirements.** Validators live in the nested `conformance` module,
  and code generators run through `go run <module>@<version>`, so neither reaches consumers.

### Dependencies

- **Forge repositories** — none upstream. Consumers: `forge-sdk` (models and documents), every service
  (models and embedded documents), `forge-gateway` (audience metadata), and `forge-provisioner` and
  `forge-agent` (desired-state schemas and types).
- **Root module** — the Go standard library only.
- **`conformance` module** — [kin-openapi](https://github.com/getkin/kin-openapi) v0.149.0 to validate
  examples against operations, and
  [santhosh-tekuri/jsonschema/v6](https://github.com/santhosh-tekuri/jsonschema) v6.0.3 for desired-state
  schemas. Its `go.mod` requires only `golang.org/x/text`, plus `dlclark/regexp2` for its own tests.
- **Tools** (not in `go.mod`) — [oapi-codegen](https://github.com/oapi-codegen/oapi-codegen) v2.8.0,
  [oasdiff](https://github.com/oasdiff/oasdiff) v1.32.0, [vacuum](https://github.com/daveshanley/vacuum)
  v0.30.6.

### Data & storage

None. Documents live in Git; the module holds no runtime state.

### Security

- **Security lives in the contract.** Lint fails on any operation without an explicit `security`. An
  empty `security: []` is allowed only on an allowlisted set: enrollment and health.
- **Audiences bound exposure.** The gateway builds its route tables from `x-forge-audience`, so an
  operation is never reachable on an ingress it was not declared for ([0008](0008-forge-gateway.md)).
- **Secrets are marked.** Tokens, enrollment tokens, and projected service account tokens carry
  `x-forge-sensitive: true`. The SDK redacts them and `forge-common` logging rules exclude them.
- **Review.** `CODEOWNERS` requires the identity and gateway owners on changes to security schemes,
  `security`, or `x-forge-audience`.
- **Supply chain.** Generators are pinned by version, and releases publish the starters' signed SBOMs.

### Environment awareness

The contract is environment-neutral: no environment names in paths, and `servers` lists only `/`.
Environment binding — tokens carrying the environment ID and SPIFFE trust domains
([Environment identity](0001-project-repositories.md#environment-identity)) — is described in the
security scheme descriptions and enforced by `forge-identity` and `forge-gateway`. Services that serve the
embedded document keep the OpenAPI UI off in `production` and `staging`, per 0001's
[hardened defaults](0001-project-repositories.md#environment-awareness).

### Logging & telemetry

The repository does not log. The contract carries `traceId` in problem details, and `common/v1` documents
W3C `traceparent` and `tracestate` propagation ([Trace Context](https://www.w3.org/TR/trace-context/)) and
`X-Request-Id` without modeling them per operation.

### Configuration

None at runtime. Generator settings live in `codegen/`, and lint rules in `rules/forge.vacuum.yaml`.

### Build, release & versioning

- **Bootstrap** from `go-library-starter`, then remove its example and runtime packages (`greeter`,
  `pool`, `health`, `config`, `logging`, `obfusicate`, `errors`, `env`). Keep `version.txt` behind a
  standard-library-only accessor, which drops `Masterminds/semver`.
- **Tasks** — `lint:openapi` (vacuum with the Forge ruleset), `breaking` (oasdiff against the last tag),
  `generate` (oapi-codegen and `info.version`), `check:drift` (regenerate and diff), `test` (root and
  `conformance`).
- **CI** — the 200-series pull request workflow runs lint, breaking, drift, and tests; 300 repeats them on
  `main`; the 100-series release runs semantic-release and then asks `forge-sdk` to regenerate
  ([0003](0003-forge-sdk.md)).
- **Versioning** — `v0.x` until accepted, per [CONVENTIONS.md](CONVENTIONS.md).

### Testing

- **Lint** every document with the Forge ruleset: extensions present, `security` explicit, banned
  formats absent, problem responses referenced, and naming rules.
- **Examples** — every file in `examples/` is validated against its operation or kind. Valid samples must
  pass and invalid samples must fail with the expected pointer.
- **Round trip** — desired-state examples decode into the Go types, re-encode, and still validate, which
  catches fields missing from the hand-written types.
- **Drift and compatibility** — regenerated models must match committed code, and oasdiff must pass.
- **Dependency budget** — a test fails if the root `go.mod` gains any requirement.

## Alternatives considered

- **Protobuf and gRPC, with grpc-gateway for REST** — puts gRPC in every service, which 0001's telemetry
  decision worked to avoid.
- **Protobuf with [Connect](https://connectrpc.com/docs/go/getting-started/)** — light
  (`connectrpc.com/connect` v1.21.0 requires only `google.golang.org/protobuf` and `go-cmp`) and
  gRPC-compatible. It was not chosen because it diverges from the Echo and OpenAPI starters, adds a
  protobuf toolchain such as [Buf](https://buf.build/docs/), and serves third-party REST clients worse.
  Worth revisiting if internal call volume makes JSON costly.
- **Code-first OpenAPI in each service** (the starter's generator) — the contract would appear only after
  implementation, split across repositories, so `forge-sdk` could not come first.
- **OpenAPI 3.0.3** — the broadest tool support and the starter's current output, but its schema dialect
  differs from JSON Schema 2020-12, which would split the API and desired-state schema styles.
- **[ogen](https://github.com/ogen-go/ogen)** — its `go.mod` (v1.24.0) requires OpenTelemetry, zap,
  fasthttp, and more; too heavy for a zero-dependency module.
- **oapi-codegen's `echo5-server` or strict server in services** — checks conformance at compile time, but
  links `github.com/oapi-codegen/runtime`, whose `go.mod` requires gin, iris, and Echo v4. Left open.
- **Spectral or Redocly for linting** — Node toolchains. vacuum is Go, Spectral-compatible, and fits the
  starters' Taskfile.
- **Generating models only in `forge-sdk`** — services would import a client SDK to get server types.

## Open questions

- **Base URL** for problem `type` URIs and schema `$id`s. `forge.servercurio.com` appears in 0001 only as
  an `apiVersion` group, and whether the project controls that domain is unverified.
- **Tenancy** — derive the tenant from the token's principal (proposed) or put `/tenants/{tenantId}` in
  paths?
- **Service conformance** — models, the starter router, and contract tests (proposed), or generated
  strict servers despite the runtime dependency?
- **Desired-state types** — adopt a JSON Schema-to-Go generator, or keep hand-written types behind
  round-trip tests?
- **3.1 validation** — kin-openapi's `openapi3filter` behavior on OpenAPI 3.1 documents is unverified;
  libopenapi-validator is the fallback.
- **IDs** — opaque strings (proposed), or a standard sortable format?

## References

- [0001 — Project Repositories](0001-project-repositories.md) — inventory, desired-state format,
  environment identity, and resolved decisions.
- [CONVENTIONS.md](CONVENTIONS.md) — cross-cutting conventions derived from this document.
- [OpenAPI Specification 3.1.1](https://spec.openapis.org/oas/v3.1.1.html) — including the
  [Security Scheme Object](https://spec.openapis.org/oas/v3.1.1.html#security-scheme-object) and
  `mutualTLS`.
- [JSON Schema 2020-12](https://json-schema.org/draft/2020-12) — dialect for desired-state schemas.
- [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) — Problem Details for HTTP APIs.
- [RFC 9745](https://www.rfc-editor.org/rfc/rfc9745) — the Deprecation HTTP response header.
- [RFC 8594](https://www.rfc-editor.org/rfc/rfc8594) — the Sunset HTTP header.
- [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) — timestamp format.
- [Kubernetes API versioning](https://kubernetes.io/docs/reference/using-api/#api-versioning) and
  [deprecation policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/).
- [W3C Trace Context](https://www.w3.org/TR/trace-context/) — `traceparent` and `tracestate`.
- [oapi-codegen](https://github.com/oapi-codegen/oapi-codegen) — README: OpenAPI 3.0/3.1 support,
  `echo5-server`, `import-mapping`, models-only generation.
- [oapi-codegen/runtime `go.mod`](https://github.com/oapi-codegen/runtime/blob/main/go.mod) — requires
  gin, iris, and Echo v4.
- [kin-openapi](https://github.com/getkin/kin-openapi) — OpenAPI 3.0/3.1 parsing and `openapi3filter`.
- [oasdiff](https://github.com/oasdiff/oasdiff) — breaking-change detection; README lists 3.1/3.2 support.
- [vacuum](https://github.com/daveshanley/vacuum) — Spectral-compatible linter;
  [libopenapi](https://github.com/pb33f/libopenapi) — its parser, supporting OpenAPI 3.0–3.2.
- [santhosh-tekuri/jsonschema](https://github.com/santhosh-tekuri/jsonschema) — JSON Schema validator.
- [Connect for Go](https://connectrpc.com/docs/go/getting-started/) and its
  [v1.21.0 `go.mod`](https://github.com/connectrpc/connect-go/blob/v1.21.0/go.mod).
- [ogen](https://github.com/ogen-go/ogen) — alternative OpenAPI generator.
- [Buf](https://buf.build/docs/) — protobuf toolchain named in alternatives.
- [Go `embed` package](https://pkg.go.dev/embed).
- [go-echo-starter](https://github.com/servercurio/go-echo-starter) — `internal/openapi` (OpenAPI 3.0 from
  route metadata) and `800-call-openapi-drift.yaml`.
- [go-library-starter `.releaserc.json`](https://github.com/servercurio/go-library-starter/blob/main/.releaserc.json)
  — `prepareCmd` runs `task generate`.
- [`hashicorp/go-plugin`](https://github.com/hashicorp/go-plugin) — plugin transport kept out of this repo.
