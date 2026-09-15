<!--
  ~ SPDX-License-Identifier: Apache-2.0
-->

# 0001 — Project Repositories

- **Status:** Draft
- **Owner:** Nathan Klick
- **Date:** 2026-07-11
- **Summary:** A first-draft inventory and bootstrapping plan for the repositories the Forge
  microservices system needs, and how each is seeded from an external `go-*-starter` baseline.

> This is a **first draft** meant to be argued with. Repository names, the service decomposition, and
> the starter mappings are proposals — not commitments. Treat every table row as a starting point.

## Context & goals

`forge` is the documentation, design, and website repository for the Server Curio project family. Its
brand tagline is **"Forge Infrastructure Management"**: Forge is a product for managing servers,
network infrastructure devices, and other remote endpoints, built as a **microservices** system.

Application code does not live in `forge` — services, backends, and persistence belong in separate
project repositories. Those repositories have never been enumerated anywhere. This document fills that
gap.

**Goals**

- **Enumerate the repositories** Forge needs, each with a clear single responsibility and owner.
- **Draw the boundaries** between them so responsibilities don't overlap (auth, desired-state
  enforcement, Forge's own infra vs. the infra Forge manages).
- **Standardize bootstrapping** — every repo starts from an external `go-*-starter` baseline and then
  adopts the shared meta this repo already defines.

**Non-goals**

- No application code, services, or persistence in `forge` itself.
- No premature decomposition — the service split here is a defensible starting point, to be revised as
  the product takes shape.
- Not a deployment or API design. Those are later, per-repository design docs.

## Two axes, kept distinct

Every repository sits on one of two axes. Keeping them separate is the correction that motivated this
document.

- **Platform** — how Forge itself is built and operated: the microservices, shared libraries, the
  operator CLI, and the IaC that deploys *Forge*.
- **Product domain** — what Forge manages: the servers, network devices, and other remote endpoints,
  their schemas, and the agent that reports on and enforces state on them.

The clearest trap to avoid: `forge-infrastructure` (Platform — IaC that deploys the Forge services)
is **not** the same as `forge-inventory` (Product domain — the catalog and schemas of the endpoints
Forge manages). They are deliberately separate repositories.

The tooling follows the same split: `forge-infrastructure` deploys Forge with **Ansible + OPA**, while
`forge-provisioner` and `forge-agent` manage endpoints with **custom YAML + OPA + Tengo**.

## Architecture at a glance

Forge is a **polyrepo**: one repository per independently deployable service, shared library, CLI, or
template. This keeps ownership, versioning, CI, and access control aligned to a single responsibility,
and lets each repo be seeded from — and track — its `go-*-starter` baseline independently. The cost is
cross-repo coordination (the shared API contract and SDK exist precisely to absorb that).

```
   external IdP (Okta / Auth0 / SAML / OIDC)
                │
                ▼
            forge-sso ──────────────────► forge-identity (internal SAML/OIDC IdP, accounts,
                                                ▲        tokens, RBAC, tenancy)
                                                │
   forge-cli (operator) ──┐                     │
   3rd-party clients ─────┼─ forge-sdk ─► forge-gateway
   forge-agent ───────────┘                     │
     │  ▲           ┌───────────────────────────┼───────────────────┐
     │  │           ▼                           ▼                   ▼
     │  │    forge-inventory            forge-provisioner  (other domain services…)
     │  │                                   │       │
     │  └── desired-state directives ───────┘       └──► agentless devices (device APIs)
     ▼
   agent plugins (separate processes, built on forge-agent-plugin-sdk)

   Shared contract/clients: forge-api-schema ──► forge-sdk (used by forge-cli, forge-agent, 3rd party)
   Shared libraries:        forge-common (logging, telemetry) ──► every Forge Go repository
   Forge's own deployment:  forge-infrastructure (Ansible + OPA policies)
   Managed desired state:   custom YAML + OPA policies + Tengo ──► forge-provisioner, forge-agent
```

All `forge-sdk` traffic — operator commands, third-party calls, and the agent's inventory reports,
identity/token exchanges, and directive pulls — enters through `forge-gateway`, which enforces
authentication and authorization before routing to `forge-identity`, `forge-inventory`,
`forge-provisioner`, or other domain services.

Agents use a dedicated `forge-gateway` listener authenticated with mutual TLS, using per-agent
certificates issued by `forge-identity` at enrollment. It is separate from the operator and third-party
entry point, so managed hosts never share an ingress with administrators. See
[Agent enrollment](#agent-enrollment).

## Repository inventory

| Repository               | Axis · Category                    | Purpose                                                                                                                                                                                                                                                                                                                                       | Starter baseline                 |
|--------------------------|------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------|
| `forge` (this)           | Platform · Docs & site             | Documentation, design assets, website; shared-meta source of truth                                                                                                                                                                                                                                                                            | — (already exists)               |
| `forge-api-schema`       | Platform · Shared library          | API contracts (protobuf/OpenAPI) — the inter-service and client schema                                                                                                                                                                                                                                                                        | `go-library-starter`             |
| `forge-sdk`              | Platform · Shared library          | Generated Go client SDK for the public API, published as a single Go module; used by `forge-cli`, `forge-agent`, and 3rd-party clients                                                                                                                                                                                                        | `go-library-starter`             |
| `forge-common`           | Platform · Shared library          | Shared logging and telemetry: wraps the starters' zerolog logging, correlates logs with traces, and exports OpenTelemetry data over OTLP/HTTP without gRPC; used by every Forge Go repository                                                                                                                                                 | `go-library-starter`             |
| `forge-gateway`          | Platform · Go service / API        | Edge/API gateway: routing, authN/Z enforcement, rate limiting; separate mutual-TLS ingress for agents                                                                                                                                                                                                                                         | `go-echo-starter`                |
| `forge-identity`         | Platform · Go service / API        | Internal identity **and IdP**: internal SAML/OIDC provider for platform users; accounts, API tokens, RBAC/tenancy, session issuance; internal CA for agent certificates (enrollment tokens, CRL/OCSP)                                                                                                                                         | `go-echo-starter`                |
| `forge-sso`              | Platform · Go service + site       | SSO federation broker: fronts login; authenticates against the internal `forge-identity` IdP or via SAML/OIDC exchange with an external IdP; hosts the login/SSO site                                                                                                                                                                         | `go-echo-starter`                |
| `forge-cli`              | Platform · Go CLI                  | Operator CLI; talks to the gateway via `forge-sdk`                                                                                                                                                                                                                                                                                            | `go-cli-starter`                 |
| `forge-infrastructure`   | Platform · Infra / deployment      | Ansible playbooks, roles, and inventories (YAML) that deploy the Forge microservices **themselves**, gated by Open Policy Agent (OPA) policies. Forge's *own* operational infra                                                                                                                                                               | — (Ansible + OPA; no Go starter) |
| `forge-inventory`        | Product domain · Go service / API  | Source-of-truth catalog and schemas of the managed servers, network devices, and remote endpoints; upstream for `forge-agent`'s reported inventory                                                                                                                                                                                            | `go-echo-starter`                |
| `forge-provisioner`      | Product domain · Go service / API  | Desired-state authority: owns directives/rules (custom YAML + OPA policies + Tengo scripts) and reconciliation; enforces agentless devices directly, and hands directives to `forge-agent` for agent-capable endpoints                                                                                                                        | `go-echo-starter`                |
| `forge-agent`            | Product domain · Go daemon         | Endpoint daemon on managed hosts that can run it; enforces desired-state directives (custom YAML + OPA policies + Tengo scripts) locally, collects inventory, and runs plugins as separate processes. Reaches `forge-inventory`, `forge-identity`, and `forge-provisioner` through `forge-gateway`'s mutual-TLS agent ingress via `forge-sdk` | `go-cli-starter`                 |
| `forge-agent-plugin-sdk` | Product domain · Shared library    | Plugin interface/contract + host-side helpers that every agent plugin builds against — the stable extension point for `forge-agent`                                                                                                                                                                                                           | `go-library-starter`             |
| `forge-agent-plugins`    | Product domain · Plugin collection | First-party / officially-maintained agent plugin executables, built against `forge-agent-plugin-sdk`                                                                                                                                                                                                                                          | `go-cli-starter`                 |
| `forge-plugin-starter`   | Product domain · Template          | Project-owned scaffold third parties clone to author a new agent plugin executable (pre-wired to `forge-agent-plugin-sdk`)                                                                                                                                                                                                                    | `go-cli-starter`                 |

A few decisions are baked into the table above and worth calling out explicitly:

- **One `forge-common` for logging and telemetry.** Logging and telemetry live in a single shared
  library so every Forge repo emits consistent logs, traces, and metrics. Forge repos swap the
  starter's own logging and telemetry for `forge-common` during bootstrap; the `go-*-starter`
  baselines stay general-purpose and never depend on it. Other shared concerns (config, middleware)
  stay in the starters.
- **`go-cli-starter` covers daemons.** It provides both CLI and long-running daemon scaffolding, so
  `forge-agent` and every plugin executable start from it.
- **`forge-infrastructure` has no Go starter.** It is Ansible (YAML) playbooks and OPA policies, not a
  Go service.
- **The service split is provisional.** New domain services will appear as the product grows.

### Desired-state model — one authority, two enforcement paths

`forge-provisioner` is the **single** desired-state authority. It owns the directives/rules and drives
reconciliation for every managed endpoint. Enforcement then takes one of two paths, chosen by endpoint
class:

- **Agent-capable endpoints** receive the directives from `forge-provisioner` (through
  `forge-gateway`) and enforce them *on-host* via `forge-agent`.
- **Agentless devices** (switches, appliances, cloud/vendor APIs that cannot run the agent) are
  enforced *remotely* by `forge-provisioner` through each device's own API.

Both paths converge desired↔actual against the source-of-truth in `forge-inventory`, so every endpoint
is covered by exactly one path with no overlapping authority.

### Desired-state format

`forge-provisioner` and `forge-agent` share one format for directives, separate from the Ansible used
to deploy Forge itself:

- **Custom YAML** — Forge's own schema describing the desired state of managed endpoints. Every
  document declares an `apiVersion` (e.g. `forge.servercurio.com/v1alpha1`) and a `kind`, and the JSON
  Schemas for each version are published from `forge-api-schema`.
- **OPA policies** — Rego policies that validate and authorize directives, evaluated by OPA embedded as
  a Go library in both services: `forge-provisioner` checks directives when they are written and before
  dispatch, and `forge-agent` re-checks them on the host before enforcing. OPA returns policy
  decisions; it does not change anything itself.
- **Tengo scripts** — [Tengo](https://github.com/d5/tengo), a scripting language embedded in Go, for
  logic YAML can't express. Both `forge-provisioner` and `forge-agent` embed the Tengo runtime. Scripts
  may import only allowlisted pure standard-library modules (e.g. `text`, `math`, `json`) plus
  Forge-provided functions — no `os` or file access — and every run has an allocation cap and a
  timeout.

### Auth split

- **`forge-identity`** implements an internal SAML/OIDC IdP and owns internal accounts, tokens, RBAC,
  and tenancy.
- **`forge-sso`** is the federation broker that fronts login. It authenticates users either against the
  internal `forge-identity` IdP *or* via SAML/OIDC token exchange with an external IdP (Okta, Auth0, …).
  Either path resolves to a `forge-identity` principal.

Separating the broker from the IdP and identity store lets the internet-facing SSO surface be hardened
independently of the internal identity system.

### Agent enrollment

`forge-identity` runs an internal certificate authority (CA) for agent identities. Its intermediate CA
signs agent certificates; the root CA stays offline in an HSM or cloud key-management service. The
intermediate CA key is held in one of two backends, chosen per deployment:

- **HSM or cloud KMS** — `forge-identity` signs through the HSM or KMS API, so the key never enters the
  service's memory or disk.
- **Encrypted store in `forge-identity`** — the key is stored encrypted and decrypted in memory to sign.
  Its key-encryption key (KEK) comes from an external secret manager at startup, is held only in
  memory, and never sits on disk beside the encrypted CA key.

A new agent bootstraps as follows, modeled on
[`kubeadm join`](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/):

1. **Create a token.** An operator creates an enrollment token with `forge-cli`. It is single-use, valid
   for 1 hour by default (operators may set up to 24 hours per token for batch provisioning), bound to a
   tenant and optional host labels, and carries the SHA-256 hash of the Forge CA certificate.
2. **Generate a key on the host.** `forge-agent` generates its private key locally, and the key never
   leaves the host. It is stored in the TPM or OS keystore when one is available, otherwise in a file
   readable only by the agent's user.
3. **Verify the gateway.** The agent connects to the enrollment route on the agent ingress over regular
   TLS and checks the gateway's certificate chain against the CA hash from the token. This is the only
   route on the agent ingress that does not require a client certificate.
4. **Request a certificate.** The agent sends a certificate signing request (CSR) with the token.
   `forge-identity` validates the token, marks it used, and returns a certificate signed by the
   intermediate CA.
5. **Operate over mutual TLS.** All further agent traffic uses the certificate. The agent renews it over
   its existing mutual-TLS connection at two-thirds of its lifetime (e.g. day 20 of a 30-day
   certificate), leaving the final third to retry through outages.

Certificate lifetime and revocation:

- **Lifetime** — configurable per tenant from 30 to 90 days, defaulting to 30 days.
- **Revocation** — `forge-identity` publishes a certificate revocation list (CRL,
  [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280)) and runs an OCSP responder
  ([RFC 6960](https://www.rfc-editor.org/rfc/rfc6960)). `forge-gateway` checks each agent certificate
  with OCSP and caches the responses, falls back to the latest CRL when the responder is unreachable,
  and rejects the agent when neither is available within the cache window (fail closed).
- **Cache window** — the gateway caches OCSP responses and the CRL until their `nextUpdate` time, which
  `forge-identity` sets to 1 hour for OCSP and 24 hours for the CRL. A revoked agent is normally cut off
  within an hour, and within 24 hours if the OCSP responder is unavailable.

### Agent plugin ecosystem

`forge-agent` is extensible via plugins. Plugins run as **separate processes** launched and supervised
by `forge-agent`, so each plugin is its own executable and a failing plugin is isolated from the agent.

Plugins talk to `forge-agent` over gRPC using [`hashicorp/go-plugin`](https://github.com/hashicorp/go-plugin).
Before every launch, the agent verifies the plugin's
[Sigstore](https://docs.sigstore.dev/cosign/signing/overview/) (cosign) signature against trusted
publisher identities, then pins the binary's SHA-256 through go-plugin's
[`SecureConfig`](https://pkg.go.dev/github.com/hashicorp/go-plugin#SecureConfig).

- **`forge-agent-plugin-sdk`** — the stable contract plugins build against.
- **`forge-agent-plugins`** — the first-party plugins maintained by the project.
- **`forge-plugin-starter`** — the project-owned scaffold third parties clone to author their own.

Note the distinction from the `go-*-starter` family: those are **external, general-purpose** baselines
that seed Forge repos, whereas `forge-plugin-starter` is a **Forge-specific, project-owned** template
that depends on `forge-agent-plugin-sdk`.

### Logging and telemetry (`forge-common`)

`forge-common` gives every Forge Go repository the same logging and telemetry with as few dependencies
as possible:

- **Logging** — wraps the zerolog-based `logging` package the `go-*-starter` baselines already ship.
  There is no official OpenTelemetry bridge for zerolog, so a zerolog
  [`Hook`](https://pkg.go.dev/github.com/rs/zerolog#Hook) reads the span context from
  [`Event.GetCtx()`](https://pkg.go.dev/github.com/rs/zerolog#Event.GetCtx) and adds `trace_id` and
  `span_id` to each event logged with a context.
- **Traces and metrics** — the OpenTelemetry Go API and official SDK; both signals are stable
  ([project status](https://github.com/open-telemetry/opentelemetry-go#project-status)).
- **Export** — a Forge-built OTLP/HTTP exporter on `go.opentelemetry.io/proto/slim/otlp` and `net/http`.
  The official OTLP exporters link 15 third-party modules, including gRPC, even when exporting over
  HTTP (measured on otel v1.46.0;
  [opentelemetry-go#2579](https://github.com/open-telemetry/opentelemetry-go/issues/2579)). The custom
  exporter is estimated at about 8. Forge keeps this exporter permanently and owns its retries,
  compression, TLS, and configuration.

### Forge's own infrastructure

`forge-infrastructure` deploys and manages the environments Forge's own services run in. It is an
Ansible project rather than a Go project, and it does not depend on `forge-agent`:

- **Ansible (YAML)** — playbooks, roles, and inventories describing how each Forge service is deployed
  and configured per environment.
- **OPA policies** — Rego policies evaluated against the Ansible inventories and variables before a run
  (for example, "`forge-identity` is never exposed on a public interface"), so non-compliant changes are
  blocked before they reach an environment.
- **Execution** — [Conftest](https://www.conftest.dev) evaluates the OPA policies in pull-request CI; a
  dedicated control node (e.g. [AWX](https://github.com/ansible/awx)) runs the merged playbooks, so
  deployment credentials never live in CI.

Deploying Forge with Ansible keeps the two axes fully separate: Forge does not depend on its own agent
or provisioner to deploy itself.

## Naming & conventions

- **Slug rule.** Every repository is `forge-<name>` (kebab-case), except this repository (`forge`) and
  the external `go-*-starter` baselines.
- **Shared meta.** Each new repository inherits the meta set this repository defines — GPG + DCO commit
  signing, Conventional Commits (validated on PR titles), the numeric-prefix CI convention
  (200 = PR-triggered, 300 = main-push, 100 = operational/release, 800 = reusable), `CODEOWNERS`, and
  the Apache-2.0 `LICENSE`.
- **One source of truth per contract.** The wire contract lives once in `forge-api-schema`; clients
  consume the generated `forge-sdk` rather than re-deriving types. Logging and telemetry are likewise
  implemented once, in `forge-common`, rather than per repository.

## Bootstrapping a repository from a starter

The repeatable procedure for standing up any repository in the inventory:

1. **Create from the baseline.** Instantiate the repo from the mapped `go-*-starter` template (or, for
   `forge-infrastructure`, from the Ansible + OPA layout described above).
2. **Rename** the Go module path and any template placeholders to the `forge-<name>` identity.
3. **Adopt shared meta** from `forge` — signing config, CI workflows (numeric-prefix), `CODEOWNERS`,
   license, and the contribution/security docs.
4. **Wire shared dependencies** — services and clients pin `forge-api-schema` / `forge-sdk`; plugins
   pin `forge-agent-plugin-sdk`; every Go repository replaces the starter's logging and telemetry
   with `forge-common`.
5. **Register ownership** in `CODEOWNERS` and enable branch protection + PR-title validation.

## Sequencing / phases

A suggested order that keeps each step shippable and unblocks the next:

1. **Contract and shared libraries first** — `forge-api-schema`, then `forge-sdk`, alongside
   `forge-common`. Everything downstream depends on these.
2. **Operational infra** — `forge-infrastructure` Ansible playbooks and OPA policies, so the first
   services deploy through the intended path from day one.
3. **Auth spine** — `forge-identity`, then `forge-sso` and `forge-gateway`, so requests can be
   authenticated end to end.
4. **First domain slice** — `forge-inventory` + a thin `forge-cli` path to prove the full request loop.
5. **Enforcement** — `forge-provisioner`, then `forge-agent` and the plugin repos
   (`forge-agent-plugin-sdk`, `forge-agent-plugins`, `forge-plugin-starter`).

## Resolved decisions

Answers to this document's earlier open questions (2026-09-14 to 2026-09-15). The sections above reflect them.

- **Service decomposition depth** — Keep the current split. Revisit after the first domain slice
  (`forge-inventory` + `forge-cli`) proves the full request loop.
- **SDK modularity** — `forge-sdk` is a single Go module with one version: simplest to release while the
  API is still changing, and it can be split per service later.
- **Telemetry stack** — `forge-common` wraps the starters' zerolog logging, correlates logs with traces
  through a zerolog hook, and uses the OpenTelemetry API and SDK with a Forge-built OTLP/HTTP exporter.
  This was the lowest-dependency OpenTelemetry option measured, because the official OTLP exporters link
  gRPC even over HTTP. See [Logging and telemetry](#logging-and-telemetry-forge-common).
- **Ansible execution** — Conftest checks OPA policies in pull-request CI; a dedicated control node runs
  the merged playbooks, keeping deployment credentials out of CI.
- **OPA evaluation** — Embedded in both `forge-provisioner` and `forge-agent`, so a tampered or stale
  directive is still caught on the host.
- **Desired-state schema** — Kubernetes-style `apiVersion`/`kind` with JSON Schemas published from
  `forge-api-schema`, supporting alpha/beta/stable stages and side-by-side versions.
- **Tengo sandboxing** — Allowlisted pure standard-library modules plus Forge-provided functions, with an
  allocation cap and a timeout on every run; no `os` or file access on managed hosts.
- **Trust zones** — A dedicated mutual-TLS agent ingress on `forge-gateway`, with per-agent certificates
  issued by `forge-identity`, kept apart from the operator and third-party entry point.
- **Agent enrollment** — `forge-identity` runs an internal CA and signs CSRs presented with a single-use
  enrollment token created in `forge-cli` (1 hour by default, 24 hours maximum); the token pins the CA
  hash for first contact. The intermediate CA key lives in an HSM or cloud KMS, or in an encrypted store
  whose KEK comes from an external secret manager. Agent keys are generated on the host and
  hardware-backed when available. Certificates last 30–90 days per tenant (default 30) and renew at
  two-thirds of their lifetime, with OCSP checks, CRL fallback, and fail-closed revocation. See
  [Agent enrollment](#agent-enrollment).
- **Revocation cache window** — The gateway caches OCSP responses and the CRL until `nextUpdate`, which
  `forge-identity` sets to 1 hour and 24 hours respectively, then fails closed.
- **Custom exporter footprint** — Keep the Forge-built OTLP/HTTP exporter permanently rather than
  switching to the official exporter if opentelemetry-go#2579 is fixed.
- **Plugin transport** — gRPC through `hashicorp/go-plugin`, which handles the handshake, process
  lifecycle, and optional mutual TLS.
- **Plugin signing** — Sigstore (cosign) signatures verified against trusted publisher identities, plus
  SHA-256 pinning through go-plugin `SecureConfig` before every launch.

## Open questions

None at present. Answered questions are recorded under
[Resolved decisions](#resolved-decisions).

## References

- `CLAUDE.md` — repository intent; application code belongs in separate project repositories.
- `.claude/module-structure.md` — `docs/` role, the CI numeric-prefix convention, intended layout.
- `.claude/conventions.md` — commit signing, Conventional Commits, one-source-of-truth rules.
- [Hugo](https://gohugo.io) — the static-site generator this repository's site is built on.
- [Ansible](https://docs.ansible.com/) — automation tool whose YAML playbooks deploy Forge's own
  services from `forge-infrastructure`.
- [Open Policy Agent](https://www.openpolicyagent.org/docs/latest/) — policy engine and the Rego
  language used for `forge-infrastructure` deployment policies and desired-state directive policies.
- [Conftest](https://www.conftest.dev) — runs OPA policies against structured configuration files,
  such as Ansible YAML, typically in CI.
- [AWX](https://github.com/ansible/awx) — self-hosted Ansible execution server; an example control node.
- [Tengo](https://github.com/d5/tengo) — embeddable Go scripting language used in desired-state
  directives by `forge-provisioner` and `forge-agent`.
- [zerolog](https://github.com/rs/zerolog) — structured logger used by the `go-*-starter` logging
  packages and wrapped by `forge-common`.
- [OpenTelemetry](https://opentelemetry.io/docs/) — vendor-neutral standard for traces, metrics, and
  logs.
- [opentelemetry-go#2579](https://github.com/open-telemetry/opentelemetry-go/issues/2579) — upstream
  issue: the OTLP/HTTP exporters depend on gRPC.
- [`go.opentelemetry.io/proto/slim/otlp`](https://github.com/open-telemetry/opentelemetry-proto-go/blob/main/slim/otlp/go.mod)
  — OTLP protobuf types without gRPC; the basis for the custom exporter.
- [`hashicorp/go-plugin`](https://github.com/hashicorp/go-plugin) — an out-of-process Go plugin system
  over RPC/gRPC.
- [Sigstore cosign](https://docs.sigstore.dev/cosign/signing/overview/) — artifact signing used for
  plugin executables.
- [Kubernetes API versioning](https://kubernetes.io/docs/reference/using-api/#api-versioning) — model
  for `apiVersion`/`kind` desired-state documents.
- [JSON Schema](https://json-schema.org/) — schema format published from `forge-api-schema`.
- [`kubeadm join`](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/) — token plus
  CA-hash bootstrap model used for agent enrollment.
- [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280) — X.509 certificates and certificate revocation
  lists (CRLs).
- [RFC 6960](https://www.rfc-editor.org/rfc/rfc6960) — Online Certificate Status Protocol (OCSP).
