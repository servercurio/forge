<!--
  ~ SPDX-License-Identifier: Apache-2.0
-->

# 0001 — Project Repositories

- **Status:** Draft
- **Owner:** Nathan Klick
- **Date:** 2026-07-11
- **Summary:** A first-draft inventory and bootstrapping plan for the repositories the Rackmarshal
  microservices system needs, and how each is seeded from an external `go-*-starter` baseline.

> This is a **first draft** meant to be argued with. Repository names, the service decomposition, and
> the starter mappings are proposals — not commitments. Treat every table row as a starting point.

## Context & goals

`rackmarshal` is the documentation, design, and website repository for the Server Curio project family. Its
brand tagline is **"Rackmarshal Infrastructure Management"**: Rackmarshal is a product for managing servers,
network infrastructure devices, and other remote endpoints, built as a **microservices** system.

Application code does not live in `rackmarshal` — services, backends, and persistence belong in separate
project repositories. Those repositories have never been enumerated anywhere. This document fills that
gap.

**Goals**

- **Enumerate the repositories** Rackmarshal needs, each with a clear single responsibility and owner.
- **Draw the boundaries** between them so responsibilities don't overlap (auth, desired-state
  enforcement, Rackmarshal's own infra vs. the infra Rackmarshal manages).
- **Standardize bootstrapping** — every repo starts from an external `go-*-starter` baseline and then
  adopts the shared meta this repo already defines.

**Non-goals**

- No application code, services, or persistence in `rackmarshal` itself.
- No premature decomposition — the service split here is a defensible starting point, to be revised as
  the product takes shape.
- Not a deployment or API design. Those are later, per-repository design docs.

## Two axes, kept distinct

Every repository sits on one of two axes. Keeping them separate is the correction that motivated this
document.

- **Platform** — how Rackmarshal itself is built and operated: the microservices, shared libraries, the
  operator CLI, and the IaC that deploys *Rackmarshal*.
- **Product domain** — what Rackmarshal manages: the servers, network devices, and other remote endpoints,
  their schemas, and the agent that reports on and enforces state on them.

The clearest trap to avoid: `rackmarshal-infrastructure` (Platform — IaC that deploys the Rackmarshal services)
is **not** the same as `rackmarshal-inventory` (Product domain — the catalog and schemas of the endpoints
Rackmarshal manages). They are deliberately separate repositories.

The tooling follows the same split: `rackmarshal-infrastructure` deploys Rackmarshal with **Ansible + OPA**, while
`rackmarshal-provisioner` and `rackmarshal-agent` manage endpoints with **custom YAML + OPA + Tengo**.

## Architecture at a glance

Rackmarshal is a **polyrepo**: one repository per independently deployable service, shared library, CLI, or
template. This keeps ownership, versioning, CI, and access control aligned to a single responsibility,
and lets each repo be seeded from — and track — its `go-*-starter` baseline independently. The cost is
cross-repo coordination (the shared API contract and SDK exist precisely to absorb that).

```
   external IdP (Okta / Auth0 / SAML / OIDC)
                │
                ▼
            rackmarshal-sso ──────────────────► rackmarshal-identity (internal SAML/OIDC IdP, accounts,
                                                ▲        tokens, RBAC, tenancy)
                                                │
   rackmarshal-cli (operator) ──┐                     │
   3rd-party clients ─────┼─ rackmarshal-sdk ─► rackmarshal-gateway
   rackmarshal-agent ───────────┘                     │
     │  ▲           ┌───────────────────────────┼───────────────────┐
     │  │           ▼                           ▼                   ▼
     │  │    rackmarshal-inventory            rackmarshal-provisioner  (other domain services…)
     │  │                                   │       │
     │  └── desired-state directives ───────┘       └──► agentless devices (device APIs)
     ▼
   agent plugins (separate processes, built on rackmarshal-agent-plugin-sdk)

   Shared contract/clients: rackmarshal-api-schema ──► rackmarshal-sdk (used by rackmarshal-cli, rackmarshal-agent, 3rd party)
   Shared libraries:        rackmarshal-common (logging, telemetry) ──► every Rackmarshal Go repository
   Rackmarshal's own deployment:  rackmarshal-infrastructure (Ansible + OPA policies)
   Managed desired state:   custom YAML + OPA policies + Tengo ──► rackmarshal-provisioner, rackmarshal-agent
```

All `rackmarshal-sdk` traffic — operator commands, third-party calls, and the agent's inventory reports,
identity/token exchanges, and directive pulls — enters through `rackmarshal-gateway`, which enforces
authentication and authorization before routing to `rackmarshal-identity`, `rackmarshal-inventory`,
`rackmarshal-provisioner`, or other domain services.

Agents use a dedicated `rackmarshal-gateway` listener authenticated with mutual TLS, using per-agent
certificates issued by `rackmarshal-identity` at enrollment. It is separate from the operator and third-party
entry point, so managed hosts never share an ingress with administrators. See
[Agent enrollment](#agent-enrollment).

## Repository inventory

| Repository               | Axis · Category                    | Purpose                                                                                                                                                                                                                                                                                                                                       | Starter baseline                 |
|--------------------------|------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------|
| `rackmarshal` (this)           | Platform · Docs & site             | Documentation, design assets, website; shared-meta source of truth                                                                                                                                                                                                                                                                            | — (already exists)               |
| `rackmarshal-api-schema`       | Platform · Shared library          | API contracts (protobuf/OpenAPI) — the inter-service and client schema                                                                                                                                                                                                                                                                        | `go-library-starter`             |
| `rackmarshal-sdk`              | Platform · Shared library          | Generated Go client SDK for the public API, published as a single Go module; used by `rackmarshal-cli`, `rackmarshal-agent`, and 3rd-party clients                                                                                                                                                                                                        | `go-library-starter`             |
| `rackmarshal-common`           | Platform · Shared library          | Shared logging and telemetry: wraps the starters' zerolog logging, correlates logs with traces, and exports OpenTelemetry data over OTLP/HTTP without gRPC; used by every Rackmarshal Go repository                                                                                                                                                 | `go-library-starter`             |
| `rackmarshal-gateway`          | Platform · Go service / API        | Edge/API gateway: routing, authN/Z enforcement, rate limiting; separate mutual-TLS ingress for agents                                                                                                                                                                                                                                         | `go-echo-starter`                |
| `rackmarshal-identity`         | Platform · Go service / API        | Internal identity **and IdP**: internal SAML/OIDC provider for platform users; accounts, API tokens, RBAC/tenancy, session issuance; internal CA for service and agent certificates (enrollment tokens, CRL/OCSP)                                                                                                                             | `go-echo-starter`                |
| `rackmarshal-sso`              | Platform · Go service + site       | SSO federation broker: fronts login; authenticates against the internal `rackmarshal-identity` IdP or via SAML/OIDC exchange with an external IdP; hosts the login/SSO site                                                                                                                                                                         | `go-echo-starter`                |
| `rackmarshal-cli`              | Platform · Go CLI                  | Operator CLI; talks to the gateway via `rackmarshal-sdk`                                                                                                                                                                                                                                                                                            | `go-cli-starter`                 |
| `rackmarshal-infrastructure`   | Platform · Infra / deployment      | Ansible playbooks, roles, and inventories (YAML) that deploy the Rackmarshal microservices **themselves** to Kubernetes, Docker/Podman hosts, or supported operating systems, gated by Open Policy Agent (OPA) policies. Rackmarshal's *own* operational infra                                                                                            | — (Ansible + OPA; no Go starter) |
| `rackmarshal-inventory`        | Product domain · Go service / API  | Source-of-truth catalog and schemas of the managed servers, network devices, and remote endpoints; upstream for `rackmarshal-agent`'s reported inventory                                                                                                                                                                                            | `go-echo-starter`                |
| `rackmarshal-provisioner`      | Product domain · Go service / API  | Desired-state authority: owns directives/rules (custom YAML + OPA policies + Tengo scripts) and reconciliation; enforces agentless devices directly, and hands directives to `rackmarshal-agent` for agent-capable endpoints                                                                                                                        | `go-echo-starter`                |
| `rackmarshal-agent`            | Product domain · Go daemon         | Endpoint daemon on managed hosts that can run it; enforces desired-state directives (custom YAML + OPA policies + Tengo scripts) locally, collects inventory, and runs plugins as separate processes. Reaches `rackmarshal-inventory`, `rackmarshal-identity`, and `rackmarshal-provisioner` through `rackmarshal-gateway`'s mutual-TLS agent ingress via `rackmarshal-sdk` | `go-cli-starter`                 |
| `rackmarshal-agent-plugin-sdk` | Product domain · Shared library    | Plugin interface/contract + host-side helpers that every agent plugin builds against — the stable extension point for `rackmarshal-agent`                                                                                                                                                                                                           | `go-library-starter`             |
| `rackmarshal-agent-plugins`    | Product domain · Plugin collection | First-party / officially-maintained agent plugin executables, built against `rackmarshal-agent-plugin-sdk`, including the core plugins (`sigstore` validator, `sysfacts`) bundled with `rackmarshal-agent`                                                                                                                                                | `go-cli-starter`                 |
| `rackmarshal-plugin-starter`   | Product domain · Template          | Project-owned scaffold third parties clone to author a new agent plugin executable (pre-wired to `rackmarshal-agent-plugin-sdk`)                                                                                                                                                                                                                    | `go-cli-starter`                 |
| `rackmarshal-portal`           | Platform · Go service + site       | Tenant-facing web portal: the endpoints a tenant owns, the desired state applied to them, and whether reality matches. Renders plans before they are applied. Holds no API token in the browser                                                                                                                                               | `go-echo-starter`                |
| `rackmarshal-console`          | Platform · Go service + site       | Platform administration web console: tenants, identity and federation, the environment CA and key backend, agent enrollment, plugin publishers, and the audit chain. Holds the two-person approval queue                                                                                                                                      | `go-echo-starter`                |

A few decisions are baked into the table above and worth calling out explicitly:

- **One `rackmarshal-common` for logging and telemetry.** Logging and telemetry live in a single shared
  library so every Rackmarshal repo emits consistent logs, traces, and metrics. Rackmarshal repos swap the
  starter's own logging and telemetry for `rackmarshal-common` during bootstrap; the `go-*-starter`
  baselines stay general-purpose and never depend on it. Other shared concerns (config, middleware)
  stay in the starters.
- **`go-cli-starter` covers daemons.** It provides both CLI and long-running daemon scaffolding, so
  `rackmarshal-agent` and every plugin executable start from it.
- **`rackmarshal-infrastructure` has no Go starter.** It is Ansible (YAML) playbooks and OPA policies, not a
  Go service.
- **The service split is provisional.** New domain services will appear as the product grows.

### Desired-state model — one authority, two enforcement paths

`rackmarshal-provisioner` is the **single** desired-state authority. It owns the directives/rules and drives
reconciliation for every managed endpoint. Enforcement then takes one of two paths, chosen by endpoint
class:

- **Agent-capable endpoints** receive the directives from `rackmarshal-provisioner` (through
  `rackmarshal-gateway`) and enforce them *on-host* via `rackmarshal-agent`.
- **Agentless devices** (switches, appliances, cloud/vendor APIs that cannot run the agent) are
  enforced *remotely* by `rackmarshal-provisioner` through each device's own API.

Both paths converge desired↔actual against the source-of-truth in `rackmarshal-inventory`, so every endpoint
is covered by exactly one path with no overlapping authority.

### Desired-state format

`rackmarshal-provisioner` and `rackmarshal-agent` share one format for directives, separate from the Ansible used
to deploy Rackmarshal itself:

- **Custom YAML** — Rackmarshal's own schema describing the desired state of managed endpoints. Every
  document declares an `apiVersion` (e.g. `rackmarshal.servercurio.com/v1alpha1`) and a `kind`, and the JSON
  Schemas for each version are published from `rackmarshal-api-schema`.
- **OPA policies** — Rego policies that validate and authorize directives, evaluated by OPA embedded as
  a Go library in both services: `rackmarshal-provisioner` checks directives when they are written and before
  dispatch, and `rackmarshal-agent` re-checks them on the host before enforcing. OPA returns policy
  decisions; it does not change anything itself.
- **Tengo scripts** — [Tengo](https://github.com/d5/tengo), a scripting language embedded in Go, for
  logic YAML can't express. Both `rackmarshal-provisioner` and `rackmarshal-agent` embed the Tengo runtime. Scripts
  may import only allowlisted pure standard-library modules (e.g. `text`, `math`, `json`) plus
  Rackmarshal-provided functions — no `os` or file access — and every run has an allocation cap and a
  timeout.

### Auth split

- **`rackmarshal-identity`** implements an internal SAML/OIDC IdP and owns internal accounts, tokens, RBAC,
  and tenancy.
- **`rackmarshal-sso`** is the federation broker that fronts login. It authenticates users either against the
  internal `rackmarshal-identity` IdP *or* via SAML/OIDC token exchange with an external IdP (Okta, Auth0, …).
  Either path resolves to a `rackmarshal-identity` principal.
- **Tokens are environment-bound.** Both services sign tokens with keys belonging to the environment
  and include its environment ID; see [Environment identity](#environment-identity).

Separating the broker from the IdP and identity store lets the internet-facing SSO surface be hardened
independently of the internal identity system.

### Agent enrollment

`rackmarshal-identity` runs the environment's internal certificate authority (CA). Its intermediate CA signs
agent and service certificates; the environment root CA (see
[Environment identity](#environment-identity)) stays offline in an HSM or cloud key-management service.
The
intermediate CA key is held in one of two backends:

- **HSM or cloud KMS (preferred)** — `rackmarshal-identity` signs through the HSM or KMS API, so the key never
  enters the service's memory or disk. This is the expected backend for production.
- **KEK-sealed store in `rackmarshal-identity` (last resort)** — the key is stored encrypted and decrypted in
  memory to sign. Its key-encryption key (KEK) comes from an external secret manager at startup, is held
  only in memory, and never sits on disk beside the encrypted CA key. Use it for testing and staging, or
  in `production` only when no HSM or KMS is available, through the explicit, logged override described
  under [Environment awareness](#environment-awareness). While the key is in memory, a compromised
  `rackmarshal-identity` can steal it and issue agent certificates.

A new agent bootstraps as follows, modeled on
[`kubeadm join`](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/):

1. **Create a token.** An operator creates an enrollment token with `rackmarshal-cli`. It is single-use, valid
   for 1 hour by default (operators may set up to 24 hours per token for batch provisioning), bound to an
   environment, a tenant, and optional host labels, and carries the environment ID and the SHA-256 hash
   of that environment's CA certificate.
2. **Generate a key on the host.** `rackmarshal-agent` generates its private key locally, and the key never
   leaves the host. It is stored in the TPM or OS keystore when one is available, otherwise in a file
   readable only by the agent's user.
3. **Verify the gateway.** The agent connects to the enrollment route on the agent ingress over regular
   TLS and checks the gateway's certificate chain against the CA hash from the token, and its SPIFFE ID
   against the token's environment ID. This is the only
   route on the agent ingress that does not require a client certificate.
4. **Request a certificate.** The agent sends a certificate signing request (CSR) with the token.
   `rackmarshal-identity` validates the token, marks it used, and returns a certificate signed by the
   intermediate CA that carries the agent's SPIFFE ID (`spiffe://<environment-id>/agent/<agent-id>`).
5. **Operate over mutual TLS.** All further agent traffic uses the certificate. The agent renews it over
   its existing mutual-TLS connection at two-thirds of its lifetime (e.g. day 20 of a 30-day
   certificate), leaving the final third to retry through outages.

Certificate lifetime and revocation:

- **Lifetime** — configurable per tenant from 30 to 90 days, defaulting to 30 days.
- **Revocation** — `rackmarshal-identity` publishes a certificate revocation list (CRL,
  [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280)) and runs an OCSP responder
  ([RFC 6960](https://www.rfc-editor.org/rfc/rfc6960)). `rackmarshal-gateway` checks each agent certificate
  with OCSP and caches the responses, falls back to the latest CRL when the responder is unreachable,
  and rejects the agent when neither is available within the cache window (fail closed).
- **Cache window** — the gateway caches OCSP responses and the CRL until their `nextUpdate` time, which
  `rackmarshal-identity` sets to 1 hour for OCSP and 24 hours for the CRL. A revoked agent is normally cut off
  within an hour, and within 24 hours if the OCSP responder is unavailable.

### Agent plugin ecosystem

`rackmarshal-agent` is extensible via plugins. Plugins run as **separate processes** launched and supervised
by `rackmarshal-agent`, so each plugin is its own executable and a failing plugin is isolated from the agent.

Plugins talk to `rackmarshal-agent` over gRPC using [`hashicorp/go-plugin`](https://github.com/hashicorp/go-plugin).
Two kinds of plugins are trusted differently:

- **Core plugins** — `sigstore` (the on-host Sigstore validator) and `sysfacts` are built in
  `rackmarshal-agent-plugins`, bundled in every `rackmarshal-agent` package, enabled by default, and installed
  root-owned and read-only. Operators may disable them in root-owned local configuration but cannot
  replace them with binaries that are not core-signed. Each ships a
  [DSSE](https://github.com/secure-systems-lab/dsse/blob/master/protocol.md) envelope over its name,
  version, platform, SHA-256, and protocol versions, signed with a Rackmarshal core-plugin key (ECDSA P-256,
  held in an HSM or cloud KMS and used only by the `rackmarshal-agent-plugins` release workflow). The agent
  embeds the current and next public keys and verifies the envelope with standard-library ECDSA before
  install and before every launch, so core-signed updates can arrive without a new agent release.
- **Other plugins** — releases are signed with
  [Sigstore](https://docs.sigstore.dev/cosign/signing/overview/) (cosign). `rackmarshal-provisioner` verifies
  those signatures against trusted publisher identities when an operator imports a plugin release, and
  records the verified SHA-256 digests, with each publisher identity, in the directive bundles it signs
  with its environment service certificate. Before install, the downloaded digest must match the pin and
  the core `sigstore` validator must verify the release's Sigstore bundle against that identity, using a
  trusted root verified through [TUF](https://theupdateframework.github.io/specification/latest/). The
  unprivileged agent process refreshes TUF metadata through the validator, with egress only to
  Sigstore's TUF repository or a configured mirror; the network-free privileged executor verifies it.

Before every launch, the agent re-checks the core signature or bundle pin, then pins the binary's
SHA-256 through go-plugin's
[`SecureConfig`](https://pkg.go.dev/github.com/hashicorp/go-plugin#SecureConfig). The agent binary links
no Sigstore verifier: sigstore-go v1.3.0's verifier compiles in 71 modules, and the validator plugin that
runs it links 79 with go-plugin (measured 2026-09-15). The agent also passes its
environment ID to each plugin at startup, and a plugin refuses to serve an agent whose environment ID
differs from the one it was configured for.

- **`rackmarshal-agent-plugin-sdk`** — the stable contract plugins build against.
- **`rackmarshal-agent-plugins`** — the first-party plugins maintained by the project, including the core
  plugins bundled with `rackmarshal-agent`.
- **`rackmarshal-plugin-starter`** — the project-owned scaffold third parties clone to author their own.

Note the distinction from the `go-*-starter` family: those are **external, general-purpose** baselines
that seed Rackmarshal repos, whereas `rackmarshal-plugin-starter` is a **Rackmarshal-specific, project-owned** template
that depends on `rackmarshal-agent-plugin-sdk`.

### Logging and telemetry (`rackmarshal-common`)

`rackmarshal-common` gives every Rackmarshal Go repository the same logging and telemetry with as few dependencies
as possible:

- **Logging** — wraps the zerolog-based `logging` package the `go-*-starter` baselines already ship.
  There is no official OpenTelemetry bridge for zerolog, so a zerolog
  [`Hook`](https://pkg.go.dev/github.com/rs/zerolog#Hook) reads the span context from
  [`Event.GetCtx()`](https://pkg.go.dev/github.com/rs/zerolog#Event.GetCtx) and adds `trace_id` and
  `span_id` to each event logged with a context.
- **Traces and metrics** — the OpenTelemetry Go API and official SDK; both signals are stable
  ([project status](https://github.com/open-telemetry/opentelemetry-go#project-status)).
- **Export** — a Rackmarshal-built OTLP/HTTP exporter on `go.opentelemetry.io/proto/slim/otlp` and `net/http`.
  The official OTLP exporters link 15 third-party modules, including gRPC, even when exporting over
  HTTP (measured on otel v1.46.0;
  [opentelemetry-go#2579](https://github.com/open-telemetry/opentelemetry-go/issues/2579)). The custom
  exporter measures 16 linked modules (0004). Rackmarshal keeps this exporter permanently and owns its retries,
  compression, TLS, and configuration.
- **Environment** — every log event and exported telemetry resource carries the environment name; see
  [Environment awareness](#environment-awareness).

### Rackmarshal's own infrastructure

`rackmarshal-infrastructure` deploys and manages the environments Rackmarshal's own services run in. It is an
Ansible project rather than a Go project, and it does not depend on `rackmarshal-agent`:

- **Ansible (YAML)** — playbooks, roles, and inventories describing how each Rackmarshal service is deployed
  and configured per environment. Each environment's inventory declares its environment name and tier.
- **Deployment targets** — every environment runs on one of three supported target types, and the same
  Ansible and OPA pipeline deploys all of them:
  - **Kubernetes** — Ansible installs each service's Helm chart with
    [`kubernetes.core.helm`](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/helm_module.html).
  - **Containers** — Podman hosts run [Quadlet](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) units, and Docker hosts run
    [Compose](https://docs.docker.com/compose/) files.
  - **Directly on a compatible operating system** — signed deb and rpm packages with systemd units on
    Enterprise Linux and Debian/Ubuntu LTS (amd64 and arm64), and installer-based Windows services on
    Windows Server.
- **OPA policies** — Rego policies evaluated against the Ansible inventories and variables, and against
  the rendered Helm, Compose, and Quadlet output, before a run
  (for example, "`rackmarshal-identity` is never exposed on a public interface"), so non-compliant changes are
  blocked before they reach an environment.
- **Execution** — [Conftest](https://www.conftest.dev) evaluates the OPA policies in pull-request CI; a
  dedicated control node (e.g. [AWX](https://github.com/ansible/awx)) runs the merged playbooks, so
  deployment credentials never live in CI.
- **Service enrollment** — the control node holds its own environment certificate and delivers
  single-use service enrollment tokens to each service it deploys on container and operating-system
  targets, while Kubernetes pods enroll with their service account tokens; see
  [Environment identity](#environment-identity).

Deploying Rackmarshal with Ansible keeps the two axes fully separate: Rackmarshal does not depend on its own agent
or provisioner to deploy itself.

## Naming & conventions

- **Slug rule.** Every repository is `rackmarshal-<name>` (kebab-case), except this repository (`rackmarshal`) and
  the external `go-*-starter` baselines.
- **Shared meta.** Each new repository inherits the meta set this repository defines — GPG + DCO commit
  signing, Conventional Commits (validated on PR titles), the numeric-prefix CI convention
  (200 = PR-triggered, 300 = main-push, 100 = operational/release, 800 = reusable), `CODEOWNERS`, the
  Apache-2.0 `LICENSE`, and SPDX license headers enforced by license-eye.
- **One source of truth per contract.** The wire contract lives once in `rackmarshal-api-schema`; clients
  consume the generated `rackmarshal-sdk` rather than re-deriving types. Logging and telemetry are likewise
  implemented once, in `rackmarshal-common`, rather than per repository.

## Cross-cutting requirements

### Environment awareness

Every Rackmarshal component — the services, `rackmarshal-cli`, `rackmarshal-agent`, agent plugins, and
`rackmarshal-infrastructure` — must know which environment it runs in and behave accordingly.

- **Tiers** — four fixed tiers: `production`, `staging`, `test`, and `development`. A deployment may use
  its own environment name (e.g. `qa-east`) but must declare its tier; behavior follows the tier, never
  the name. An unknown tier is rejected at startup.
- **Required at startup** — there is no default. The environment name and tier are set through each
  component's configuration file or `<PREFIX>_*` environment variables, like the starters' other
  settings, and a component started without them refuses to start. Local development tooling sets
  `development` explicitly. `rackmarshal-agent` records its environment from the deployment it enrolls with.
- **Hardened defaults by tier** — `production` and `staging` default to hardened settings (for example
  TLS, secure cookies, and the OpenAPI UI disabled); `development` relaxes them. This replaces the
  per-setting "change this in production" guidance in the `go-*-starter` configuration.
- **Last-resort features gated** — features marked last resort or testing-only, such as the KEK-sealed
  CA key store, are refused in `production` unless explicitly overridden, and every override is logged.
- **Isolation** — each environment has its own CA, enrollment tokens, and credentials. A service or
  agent from one environment is rejected by another; see [Environment identity](#environment-identity).
- **Tagged logs and telemetry** — `rackmarshal-common` adds the environment name to every log event and sets
  the OpenTelemetry
  [`deployment.environment.name`](https://opentelemetry.io/docs/specs/semconv/registry/attributes/deployment/)
  resource attribute on exported telemetry.

### Environment identity

Every environment has a cryptographic identity, so each component can prove it is talking to the correct
deployment:

- **Environment ID** — a random, stable ID generated when the environment is created and embedded in the
  environment's root CA certificate as its SPIFFE trust domain (`spiffe://<environment-id>`). During root
  CA rotation, components trust the old and new roots together; both carry the same ID, so the identity
  survives rotation.
- **Certificates** — every Rackmarshal certificate carries one SPIFFE ID, such as
  `spiffe://<environment-id>/service/rackmarshal-inventory` or `spiffe://<environment-id>/agent/<agent-id>`,
  in the [X.509-SVID](https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md) format. A peer is
  trusted only if its certificate chains to the environment's root and its SPIFFE ID names the same
  environment. Rackmarshal uses the SPIFFE ID format without running SPIRE.
- **Service-to-service** — all internal calls between Rackmarshal services use mutual TLS with these
  certificates, issued by the environment's intermediate CA in `rackmarshal-identity`.
- **Service certificate bootstrap** — when an environment is created, a key ceremony uses the offline
  root CA to sign `rackmarshal-identity`'s intermediate CA and the `rackmarshal-infrastructure` control node's
  certificate; `rackmarshal-identity` then issues its own service certificate from the intermediate. Every
  other service instance enrolls with a CSR, as an agent does, using a credential that depends on its
  deployment target:
  - **Containers and operating systems** — the control node requests a single-use service enrollment
    token from `rackmarshal-identity` and delivers it through Ansible.
  - **Kubernetes** — an enrollment init container presents the pod's short-lived, audience-bound
    [projected service account token](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection).
    `rackmarshal-identity` verifies it offline against the cluster's service account issuer
    ([issuer discovery](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-issuer-discovery)), which the
    environment-creation ceremony registers together with a mapping from service accounts to Rackmarshal
    service identities. Only the mapped service account can obtain, for example,
    `spiffe://<environment-id>/service/rackmarshal-inventory`, and pods enroll on start, scale-out, and
    rescheduling without an Ansible run.

  Service certificates last 7 days and renew automatically at two-thirds of their lifetime, with the
  same OCSP and CRL checks as agent certificates.
- **Agents and plugins** — the enrollment token carries the environment ID, and the agent checks it
  against the gateway's certificate before enrolling. The agent passes its environment ID to plugins in
  the first RPC of the plugin protocol, and it is the only source of that value on the host.
- **User and API tokens** — tokens from `rackmarshal-identity` and `rackmarshal-sso` are signed with keys belonging
  to the environment and carry its ID (e.g. in the issuer and audience claims), so a token from one
  environment is rejected in another.
- **`rackmarshal-cli` and infrastructure** — each `rackmarshal-cli` profile pins an environment ID and CA bundle and
  verifies the gateway before sending credentials. `rackmarshal-infrastructure` inventories supply each
  environment's ID and CA bundle to the services they deploy.

### License headers and license files

Every Rackmarshal repository — `rackmarshal`, each `rackmarshal-*` repository, and the `go-*-starter` baselines they are
seeded from — carries an Apache-2.0 `LICENSE` file at its root, and every tracked file starts with an
[SPDX license identifier](https://spdx.dev/learn/handling-license-info/) in its own comment syntax:

- **Header** — the identifier only, for example `// SPDX-License-Identifier: Apache-2.0` in Go, `#` in
  YAML and shell, `--` in SQL, and an HTML comment in Markdown and SVG. Copyright is carried by
  `LICENSE` and version history, not by each file.
- **Enforced** — [license-eye](https://github.com/apache/skywalking-eyes) (`header check`, configured by
  `.licenserc.yaml`) runs on every pull request and on `main` through an `800-call-license-headers`
  reusable workflow, so a missing header fails the build. That workflow runs `task lint:license`, which
  installs license-eye with `go install` at the version pinned once in `Taskfile.yaml`, rather than
  through its GitHub Action, which references other actions by tag.
- **Exceptions** — only files that cannot hold a comment (`LICENSE`, JSON, `go.sum`, `.gitkeep`, and
  embedded data such as version strings) are listed in `.licenserc.yaml`.
- **Generated files** — generators emit the header themselves, so regenerating never fails the check.

Repositories that third parties create from `rackmarshal-plugin-starter` choose their own license and SPDX
identifier.

## Bootstrapping a repository from a starter

The repeatable procedure for standing up any repository in the inventory:

1. **Create from the baseline.** Instantiate the repo from the mapped `go-*-starter` template (or, for
   `rackmarshal-infrastructure`, from the Ansible + OPA layout described above).
2. **Rename** the Go module path and any template placeholders to the `rackmarshal-<name>` identity.
3. **Adopt shared meta** from `rackmarshal` — signing config, CI workflows (numeric-prefix), `CODEOWNERS`,
   license, `.licenserc.yaml` with the license-header check, and the contribution/security docs.
4. **Wire shared dependencies** — services and clients pin `rackmarshal-api-schema` / `rackmarshal-sdk`; plugins
   pin `rackmarshal-agent-plugin-sdk`; every Go repository replaces the starter's logging and telemetry
   with `rackmarshal-common`.
5. **Register ownership** in `CODEOWNERS` and enable branch protection + PR-title validation.

## Sequencing / phases

A suggested order that keeps each step shippable and unblocks the next:

1. **Contract and shared libraries first** — `rackmarshal-api-schema`, then `rackmarshal-sdk`, alongside
   `rackmarshal-common`. Everything downstream depends on these.
2. **Operational infra** — `rackmarshal-infrastructure` Ansible playbooks and OPA policies, so the first
   services deploy through the intended path from day one.
3. **Auth spine** — `rackmarshal-identity`, then `rackmarshal-sso` and `rackmarshal-gateway`, so requests can be
   authenticated end to end.
4. **First domain slice** — `rackmarshal-inventory` + a thin `rackmarshal-cli` path to prove the full request loop.
5. **Enforcement** — `rackmarshal-provisioner`, then `rackmarshal-agent` and the plugin repos
   (`rackmarshal-agent-plugin-sdk`, `rackmarshal-agent-plugins`, `rackmarshal-plugin-starter`).
6. **Browser surfaces** — `rackmarshal-portal` and `rackmarshal-console` ([0016](0016-web-ui-architecture.md)),
   after the APIs they render exist. Neither is on the critical path: `rackmarshal-cli` covers every
   operation, so the portals are additive.

## Resolved decisions

Answers to this document's earlier open questions (2026-09-14 to 2026-09-15). The sections above reflect them.

- **Service decomposition depth** — Keep the current split. Revisit after the first domain slice
  (`rackmarshal-inventory` + `rackmarshal-cli`) proves the full request loop.
- **SDK modularity** — `rackmarshal-sdk` is a single Go module with one version: simplest to release while the
  API is still changing, and it can be split per service later.
- **Telemetry stack** — `rackmarshal-common` wraps the starters' zerolog logging, correlates logs with traces
  through a zerolog hook, and uses the OpenTelemetry API and SDK with a Rackmarshal-built OTLP/HTTP exporter.
  This was the lowest-dependency OpenTelemetry option measured, because the official OTLP exporters link
  gRPC even over HTTP. See [Logging and telemetry](#logging-and-telemetry-rackmarshal-common).
- **Ansible execution** — Conftest checks OPA policies in pull-request CI; a dedicated control node runs
  the merged playbooks, keeping deployment credentials out of CI.
- **OPA evaluation** — Embedded in both `rackmarshal-provisioner` and `rackmarshal-agent`, so a tampered or stale
  directive is still caught on the host.
- **Desired-state schema** — Kubernetes-style `apiVersion`/`kind` with JSON Schemas published from
  `rackmarshal-api-schema`, supporting alpha/beta/stable stages and side-by-side versions.
- **Tengo sandboxing** — Allowlisted pure standard-library modules plus Rackmarshal-provided functions, with an
  allocation cap and a timeout on every run; no `os` or file access on managed hosts.
- **Trust zones** — A dedicated mutual-TLS agent ingress on `rackmarshal-gateway`, with per-agent certificates
  issued by `rackmarshal-identity`, kept apart from the operator and third-party entry point.
- **Agent enrollment** — `rackmarshal-identity` runs an internal CA and signs CSRs presented with a single-use
  enrollment token created in `rackmarshal-cli` (1 hour by default, 24 hours maximum); the token pins the CA
  hash for first contact. The intermediate CA key lives in an HSM or cloud KMS (preferred); a store
  sealed with a KEK from an external secret manager is a last resort for testing, staging, or
  production without an HSM or KMS. Agent keys are generated on the host and
  hardware-backed when available. Certificates last 30–90 days per tenant (default 30) and renew at
  two-thirds of their lifetime, with OCSP checks, CRL fallback, and fail-closed revocation. See
  [Agent enrollment](#agent-enrollment).
- **Revocation cache window** — The gateway caches OCSP responses and the CRL until `nextUpdate`, which
  `rackmarshal-identity` sets to 1 hour and 24 hours respectively, then fails closed.
- **Custom exporter footprint** — Keep the Rackmarshal-built OTLP/HTTP exporter permanently rather than
  switching to the official exporter if opentelemetry-go#2579 is fixed.
- **Environment awareness** — Every component requires an environment name and one of four tiers
  (`production`, `staging`, `test`, `development`) at startup and refuses to start without them. The
  tier drives hardened defaults, gates last-resort features, isolates CAs and credentials, and is tagged
  on all logs and telemetry. See [Environment awareness](#environment-awareness).
- **Environment identity** — A stable environment ID anchored in each environment's root CA and carried
  as a SPIFFE ID in every Rackmarshal certificate. Service-to-service mTLS, agents and plugins, user and API
  tokens, `rackmarshal-cli`, and `rackmarshal-infrastructure` all verify it. See
  [Environment identity](#environment-identity).
- **Service certificate bootstrap** — An environment-creation key ceremony signs `rackmarshal-identity`'s
  intermediate CA and the control node's certificate from the offline root. Services on container and
  operating-system targets enroll with single-use tokens delivered by Ansible; Kubernetes pods enroll
  with projected service account tokens verified against the registered cluster issuer and service
  account mapping. Service certificates last 7 days and renew automatically.
- **Deployment targets** — Kubernetes (Helm charts), containers (Podman Quadlet and Docker Compose), and
  direct installation on Enterprise Linux and Debian/Ubuntu LTS (signed deb and rpm packages with
  systemd) or Windows Server (installer-based services), all deployed by the same Ansible and OPA
  pipeline. See [Rackmarshal's own infrastructure](#rackmarshals-own-infrastructure).
- **Plugin transport** — gRPC through `hashicorp/go-plugin`, which handles the handshake, process
  lifecycle, and optional mutual TLS.
- **Plugin signing** — Core plugins (`sigstore`, the on-host Sigstore validator, and `sysfacts`) are
  bundled in every `rackmarshal-agent` package, enabled by default, and root-owned and read-only; operators
  may disable but not replace them. They carry DSSE signatures from a Rackmarshal core-plugin ECDSA P-256 key
  held in an HSM or cloud KMS, which the agent verifies against embedded current and next public keys
  before install and every launch. Other plugin releases carry Sigstore (cosign) signatures:
  `rackmarshal-provisioner` verifies them against trusted publisher identities at import and pins the verified
  SHA-256 digests, with the publisher identity, in the directive bundles it signs; a host installs one
  only when the digest matches the pin and the core validator verifies the Sigstore bundle against that
  identity with a TUF-verified trusted root. TUF metadata is refreshed by the unprivileged agent process
  with egress only to Sigstore's TUF repository or a mirror, and verified in the network-free executor.
  Every launch pins the SHA-256 through go-plugin `SecureConfig`. The agent binary links no Sigstore
  verifier: sigstore-go v1.3.0's verifier compiles in 71 modules, confined to the validator plugin (79
  with go-plugin; measured 2026-09-15).
- **License headers** — Every repository carries an Apache-2.0 `LICENSE` and identifier-only SPDX
  headers in every file, enforced by license-eye in pull-request and main-branch checks, with an explicit
  ignore list for files that cannot hold a comment. See
  [License headers and license files](#license-headers-and-license-files).

## Open questions

None at present. Answered questions are recorded under
[Resolved decisions](#resolved-decisions).

## References

- `CLAUDE.md` — repository intent; application code belongs in separate project repositories.
- `.claude/module-structure.md` — `docs/` role, the CI numeric-prefix convention, intended layout.
- `.claude/conventions.md` — commit signing, Conventional Commits, one-source-of-truth rules.
- [Hugo](https://gohugo.io) — the static-site generator this repository's site is built on.
- [Ansible](https://docs.ansible.com/) — automation tool whose YAML playbooks deploy Rackmarshal's own
  services from `rackmarshal-infrastructure`.
- [Open Policy Agent](https://www.openpolicyagent.org/docs/latest/) — policy engine and the Rego
  language used for `rackmarshal-infrastructure` deployment policies and desired-state directive policies.
- [Conftest](https://www.conftest.dev) — runs OPA policies against structured configuration files,
  such as Ansible YAML, typically in CI.
- [AWX](https://github.com/ansible/awx) — self-hosted Ansible execution server; an example control node.
- [`kubernetes.core.helm`](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/helm_module.html) — Ansible
  module that installs Helm charts on Kubernetes targets.
- [Podman Quadlet](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) — systemd units for Podman
  containers.
- [Docker Compose](https://docs.docker.com/compose/) — container definitions for Docker hosts.
- [Projected service account tokens](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection)
  and [service account issuer discovery](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-issuer-discovery) —
  Kubernetes pod credentials used for service enrollment.
- [Tengo](https://github.com/d5/tengo) — embeddable Go scripting language used in desired-state
  directives by `rackmarshal-provisioner` and `rackmarshal-agent`.
- [zerolog](https://github.com/rs/zerolog) — structured logger used by the `go-*-starter` logging
  packages and wrapped by `rackmarshal-common`.
- [OpenTelemetry](https://opentelemetry.io/docs/) — vendor-neutral standard for traces, metrics, and
  logs.
- [OpenTelemetry deployment attributes](https://opentelemetry.io/docs/specs/semconv/registry/attributes/deployment/)
  — defines the `deployment.environment.name` resource attribute.
- [opentelemetry-go#2579](https://github.com/open-telemetry/opentelemetry-go/issues/2579) — upstream
  issue: the OTLP/HTTP exporters depend on gRPC.
- [`go.opentelemetry.io/proto/slim/otlp`](https://github.com/open-telemetry/opentelemetry-proto-go/blob/main/slim/otlp/go.mod)
  — OTLP protobuf types without gRPC; the basis for the custom exporter.
- [`hashicorp/go-plugin`](https://github.com/hashicorp/go-plugin) — an out-of-process Go plugin system
  over RPC/gRPC.
- [Sigstore cosign](https://docs.sigstore.dev/cosign/signing/overview/) — artifact signing used for
  plugin executables.
- [sigstore-go](https://github.com/sigstore/sigstore-go) — Sigstore verifier used by `rackmarshal-provisioner`
  and the core `sigstore` validator plugin.
- [DSSE](https://github.com/secure-systems-lab/dsse/blob/master/protocol.md) — signing envelope for core
  plugin statements.
- [The Update Framework specification](https://theupdateframework.github.io/specification/latest/) —
  secure delivery of Sigstore's trusted root.
- [sigstore/root-signing](https://github.com/sigstore/root-signing) — Sigstore's TUF repository,
  published at `https://tuf-repo-cdn.sigstore.dev`.
- [Kubernetes API versioning](https://kubernetes.io/docs/reference/using-api/#api-versioning) — model
  for `apiVersion`/`kind` desired-state documents.
- [JSON Schema](https://json-schema.org/) — schema format published from `rackmarshal-api-schema`.
- [SPDX license identifiers](https://spdx.dev/learn/handling-license-info/) and
  [skywalking-eyes (license-eye)](https://github.com/apache/skywalking-eyes) — per-file license headers
  and their enforcement.
- [`kubeadm join`](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/) — token plus
  CA-hash bootstrap model used for agent enrollment.
- [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280) — X.509 certificates and certificate revocation
  lists (CRLs).
- [RFC 6960](https://www.rfc-editor.org/rfc/rfc6960) — Online Certificate Status Protocol (OCSP).
- [SPIFFE ID](https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE-ID.md) — URI format for
  environment and workload identities.
- [X.509-SVID](https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md) — how a SPIFFE ID is carried in an
  X.509 certificate.
