<!--
  ~ SPDX-License-Identifier: Apache-2.0
-->

# Design document conventions

Cross-cutting conventions for the per-repository design documents that follow
[0001](0001-project-repositories.md). This is not a design document: it collects the proposals from
[0002](0002-forge-api-schema.md), [0003](0003-forge-sdk.md), and [0004](0004-forge-common.md) that other
repositories depend on, so documents 0005–0015 stay consistent. Every item is a **Draft proposal** until
those documents are accepted. 0001's Resolved decisions always win; a document that deviates from a
convention says so under Alternatives considered and links the convention it breaks.

## Numbering and document shape

| #    | Repository             | #    | Repository               |
|------|------------------------|------|--------------------------|
| 0002 | `forge-api-schema`     | 0009 | `forge-inventory`        |
| 0003 | `forge-sdk`            | 0010 | `forge-cli`              |
| 0004 | `forge-common`         | 0011 | `forge-provisioner`      |
| 0005 | `forge-infrastructure` | 0012 | `forge-agent`            |
| 0006 | `forge-identity`       | 0013 | `forge-agent-plugin-sdk` |
| 0007 | `forge-sso`            | 0014 | `forge-agent-plugins`    |
| 0008 | `forge-gateway`        | 0015 | `forge-plugin-starter`   |

- File `NNNN-forge-<name>.md`, title `# NNNN — forge-<name>`, header and sections from
  [`TEMPLATE.md`](TEMPLATE.md). Proposal subsections, in order, as they apply: Responsibilities;
  Interfaces; Dependencies; Data & storage; Security; Environment awareness; Logging & telemetry;
  Configuration; Build, release & versioning; Testing.
- Link 0001 by heading anchor (`0001-project-repositories.md#environment-identity`) and siblings by file
  name. Label proposals; put unknowns in Open questions; cite every external claim in References and
  mark anything unverified. Wrap at about 105 characters; `—` em dashes; no HTML.

## Go modules and layout

- Module path `github.com/servercurio/forge-<name>` (`/vN` suffix from v2, see
  [major version suffixes](https://go.dev/ref/mod#major-version-suffixes)); Go 1.27, as in the starters.
- Libraries export from `pkg/`, with versioned API packages at `pkg/<area>/<version>` named like
  `inventoryv1alpha1`. Executables keep `cmd/<binary>/` and `internal/`, and binaries take the repo name.

## API contract and style

- **One contract format.** Every Forge HTTP API is an OpenAPI 3.1 document in `forge-api-schema`
  (`openapi/<service>/<version>/openapi.yaml`); desired-state kinds are JSON Schema 2020-12 files
  (`schemas/<group>/<version>/<kind>.schema.json`). Contracts come first; code is generated from or
  tested against them.
- **REST + JSON, internal and external.** Services call each other over the same contract with mutual
  TLS. There is no service-to-service gRPC; gRPC appears only between `forge-agent` and plugins, whose
  go-plugin protobuf contract lives in `forge-agent-plugin-sdk`.
- **Paths** — `/<service>/<version>/<plural-resource>[/{id}]`, kebab-case segments, e.g.
  `/inventory/v1alpha1/endpoints/{endpointId}`. `forge-gateway` routes on the first segment.
- **Audience** — every operation declares `x-forge-audience` with one or more of `operator`, `agent`,
  `internal`. The gateway exposes `operator` operations on the operator ingress and `agent` operations on
  the agent ingress, and never routes `internal` ones.
- **Other extensions** — `x-forge-sensitive: true` on secret properties (never logged, redacted by
  the SDK); `x-forge-idempotent: true` on POST operations that are safe to retry.
- **JSON** — lowerCamelCase properties and query parameters; RFC 3339 UTC timestamps; opaque string IDs;
  no `format: uuid`, `date`, `email`, or `binary`, which generate `oapi-codegen/runtime` types. Lists
  take `?limit=&cursor=` and return `items` and `nextCursor`; no offset paging.
- **Headers** — `Authorization: Bearer <token>` for user and API tokens; W3C `traceparent` and
  `tracestate`; `X-Request-Id` echoed on every response. Health probes stay at `/livez`, `/readyz`, and
  `/healthz`, as in `go-echo-starter`, outside versioned paths.
- **Tooling** — vacuum (lint with the Forge ruleset), oasdiff (breaking changes), oapi-codegen v2.8.0
  (models in `forge-api-schema`, client in `forge-sdk`), run as `go run <module>@<version>` so tools
  never enter `go.mod`. Generated code is committed; CI regenerates and fails on drift.

## API versioning, deprecation, and errors

- **Stages** — `v1alpha1` → `v1beta1` → `v1`, matching 0001's desired-state `apiVersion`. Alpha may
  break between releases; beta breaks only by adding a new beta version beside the old one; a stable
  version never breaks, so breaking changes ship as `v2` side by side. oasdiff compares each pull
  request with the last release and fails on a breaking change to a beta or stable document.
- **Deprecation** — `deprecated: true` in the contract plus `Deprecation`
  ([RFC 9745](https://www.rfc-editor.org/rfc/rfc9745)) and `Sunset`
  ([RFC 8594](https://www.rfc-editor.org/rfc/rfc8594)) response headers.
- **Errors** — every non-2xx response is `application/problem+json`
  ([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)) with Forge members `code` (stable
  lower_snake_case, e.g. `endpoint_not_found`), `traceId`, and, for validation failures, `errors[]` of
  `pointer` and `detail`. `type` is `<problem-base-url>/<code>`; the base URL is an open question in
  0002. Problem bodies never carry secrets, stack traces, or another tenant's data.

## Security and identity

- **SPIFFE IDs** ([SPIFFE ID](https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE-ID.md)), one per
  certificate. No other path types without a design document:
  - `spiffe://<environment-id>/service/<repository>` — e.g. `/service/forge-inventory`
  - `spiffe://<environment-id>/agent/<agent-id>`
  - `spiffe://<environment-id>/control-node/<node-name>` — `forge-infrastructure` control nodes
- **ID syntax** — `<environment-id>` and `<agent-id>` are 128 random bits as 26 characters of lowercase,
  unpadded base32 ([RFC 4648](https://www.rfc-editor.org/rfc/rfc4648)), valid in a SPIFFE trust domain;
  `<node-name>` is a lowercase DNS label.
- **Shared helpers** — SPIFFE parsing, mutual-TLS `tls.Config` builders, OCSP/CRL checking, and CSR
  enrollment and renewal live in `forge-sdk` (`pkg/spiffe`, `pkg/tlsconfig`, `pkg/revocation`,
  `pkg/enroll`); no repository re-implements them. `go-spiffe` is not used — its `go.mod` requires gRPC.
- **TLS** — TLS 1.3 only where both ends are Forge components (services, agent ingress, control node);
  the operator and third-party ingress keeps the starter's hardened TLS 1.2+ configuration.
- **Keys and secrets** — leaf keys are ECDSA P-256, generated where used, and never leave that host.
  Tokens and keys are never logged or placed in URLs, and are read from files (`...File` keys).

## Configuration and environment variables

- Loading follows the starters: defaults → YAML file → `<PREFIX>_*` variables → flags. YAML keys are
  lowerCamelCase; variables are upper snake case under the prefix (`FORGE_INVENTORY_SERVER_HTTPS_PORT`).
- Each executable's prefix is `FORGE_` plus its repository name without `forge-`, upper-cased, hyphens
  as underscores: `FORGE_IDENTITY`, `FORGE_SSO`, `FORGE_GATEWAY`, `FORGE_INVENTORY`, `FORGE_CLI`,
  `FORGE_PROVISIONER`, `FORGE_AGENT`. An agent plugin uses `FORGE_PLUGIN_<NAME>`.
- Libraries define no prefix. Their config structs implement the starters' `FromEnv(prefix string)` and
  `Validate() error` and mount under fixed child keys: `logging` → `<PREFIX>_LOG_*` and
  `<PREFIX>_ACCESS_LOG_*`; `telemetry` → `<PREFIX>_TELEMETRY_*`; `environment` →
  `<PREFIX>_ENVIRONMENT_*`; the `forge-sdk` client → `<PREFIX>_GATEWAY_*`.

## Environment

- Keys under `environment` / `<PREFIX>_ENVIRONMENT_`: `name` / `_NAME` (lowercase DNS label, e.g.
  `qa-east`); `tier` / `_TIER` (`production`, `staging`, `test`, `development`); `id` / `_ID`;
  `caBundle` / `_CA_BUNDLE` (path to the root CA bundle); `overrides` / `_OVERRIDES` (comma-separated
  last-resort feature names).
- `name` and `tier` are required everywhere. `id` and `caBundle` are required wherever a component makes
  or accepts mutual-TLS connections, and `id` must match the trust domain of the bundle's roots.
  `forge-agent` records all four from enrollment; plugins receive them from the agent in `Init` and
  carry no environment configuration of their own.
- Tier logic goes through `forge-common`'s `environment` package: `Hardened()` is true for `production`
  and `staging`; last-resort features call `AllowLastResort("<feature>")`, which refuses in `production`
  unless the kebab-case feature name is in `overrides`, and logs every override at `warn`.

## Logging and telemetry

- Executables log and export telemetry only through `forge-common`. Other shared libraries return errors
  and accept injected transports instead of logging.
- Logs are JSON lines on stdout (console format only in `development`) with keys `time` (RFC 3339 UTC),
  `level`, `message`, `error`; `trace_id`, `span_id`, `trace_flags`
  ([trace context in logs](https://opentelemetry.io/docs/specs/otel/compatibility/logging_trace_context/));
  and on every event `service.name`, `service.version`, `service.instance.id`,
  `deployment.environment.name`, `forge.environment.tier`, `forge.environment.id`.
- Other fields use the OpenTelemetry semantic-convention name when one exists (`http.request.method`,
  `http.response.status_code`, `http.route`, `url.path`, `client.address`); Forge concepts use dotted,
  snake_case names under `forge.` (`forge.tenant.id`, `forge.agent.id`).
- Telemetry resources carry the same `service.*`, `deployment.environment.name`, and
  `forge.environment.*` attributes; custom metrics are named `forge.<component>.<name>`. Propagation is
  W3C Trace Context only, with no baggage. Spans and logs never contain `x-forge-sensitive` data.

## Deployment artifacts

Forge deploys to Kubernetes, Docker/Podman hosts, and supported operating systems
([0001](0001-project-repositories.md#forges-own-infrastructure)), so every service repository ships:

- **OCI image** — multi-architecture (`linux/amd64`, `linux/arm64`), non-root, referenced by digest, and
  signed; used by the Kubernetes, Podman, and Docker targets. A service that needs cgo (`forge-identity`,
  for PKCS#11) builds each architecture on a native runner instead of cross-compiling, on the oldest
  glibc in the support matrix (Enterprise Linux 9, glibc 2.34).
- **Helm chart** — in `charts/<repository>/`, as in `go-echo-starter`, including an enrollment init
  container that uses the same image and the pod's projected service account token.
- **Linux packages** — signed deb and rpm packages with a hardened systemd unit, for Enterprise Linux and
  Debian/Ubuntu LTS on amd64 and arm64.
- **Windows package** — a signed [NSIS](https://nsis.sourceforge.io) installer (`.exe`,
  Authenticode-signed) that installs the service as a Windows service on Windows Server. NSIS is
  zlib/libpng licensed and builds on Linux runners.

Quadlet units and Compose files are rendered by `forge-infrastructure` roles, not kept in service
repositories. Configuration, health probes, and enrollment behave the same on every target; only the
credential for first enrollment differs (a single-use token file, or a projected service account token).

## License headers

Per [0001](0001-project-repositories.md#license-headers-and-license-files):

- Every tracked file starts with `SPDX-License-Identifier: Apache-2.0` in its comment syntax, and
  `LICENSE` sits at the repository root. `.licenserc.yaml` is the single policy file, and
  `LICENSE_EYE_VERSION` in `Taskfile.yaml` is the only place the license-eye version is pinned.
- Comment styles that `license-eye header fix` does not infer are pinned in `.licenserc.yaml`: `go.mod`
  uses `//`, and `CODEOWNERS` and `.helmignore` use `#`.
- Every file under `charts/*/templates/` starts with `{{- /* SPDX-License-Identifier: Apache-2.0 */ -}}`,
  added by hand, so a disabled template renders nothing; a `#` header outside `{{- if }}` leaves a
  comment-only manifest that fails `helm install`. license-eye accepts any comment style there, so
  `.licenserc.yaml` checks templates in a separate block and `task lint:license` also verifies each
  template's first line.
- `paths-ignore` lists only files that cannot hold a comment: `LICENSE`, `**/go.sum`, `**/*.json`,
  `**/.gitkeep`, and embedded data such as `**/version/version.txt`.
- Generators write the header themselves (for example `openapi-gen` for YAML output); generated JSON is
  ignored.
- CI's `800-call-license-headers.yaml`, called from the 200 and 300 flows, installs Task and runs
  `task lint:license`, the same task developers run; `task lint` includes it and `task license:fix`
  adds missing headers.

## Dependencies, build, and release

- Every new direct dependency is justified in its document with version and what it pulls in. Libraries
  keep a checked-in module allowlist; CI fails when
  `go list -deps -f '{{with .Module}}{{.Path}}{{end}}' ./pkg/...` reports a module not on it.
- Test-only and tooling modules that would enlarge consumers' module graphs go in a nested module (e.g.
  `conformance/go.mod`) or run through `go run <module>@<version>`.
- Pinned choices: zerolog; the OpenTelemetry API and SDK with the Forge OTLP/HTTP exporter; OPA; Tengo;
  `hashicorp/go-plugin`; `santhosh-tekuri/jsonschema/v6` for desired-state validation (proposed).
- Taskfile, golangci-lint, `-race` tests, semantic-release, and signed CycloneDX SBOMs as in the
  starters; CI workflows keep the numeric prefixes.
- Tags are `vX.Y.Z` from Conventional Commits. Modules stay `v0.x` (the starters' `.releaserc.json` maps
  a breaking change to a minor release) until the owner declares 1.0, when that rule becomes major. A
  module's version is independent of the API versions it contains.
