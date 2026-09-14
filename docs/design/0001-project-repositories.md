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
   Forge's own deployment:  forge-infrastructure (YAML + OPA policies) ──► applied by forge-agent
```

All `forge-sdk` traffic — operator commands, third-party calls, and the agent's inventory reports,
identity/token exchanges, and directive pulls — enters through `forge-gateway`, which enforces
authentication and authorization before routing to `forge-identity`, `forge-inventory`,
`forge-provisioner`, or other domain services.

## Repository inventory

| Repository               | Axis · Category                    | Purpose                                                                                                                                                                                                                                                                                                | Starter baseline              |
|--------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------|
| `forge` (this)           | Platform · Docs & site             | Documentation, design assets, website; shared-meta source of truth                                                                                                                                                                                                                                     | — (already exists)            |
| `forge-api-schema`       | Platform · Shared library          | API contracts (protobuf/OpenAPI) — the inter-service and client schema                                                                                                                                                                                                                                 | `go-library-starter`          |
| `forge-sdk`              | Platform · Shared library          | Generated Go client SDK for the public API; used by `forge-cli`, `forge-agent`, and 3rd-party clients                                                                                                                                                                                                  | `go-library-starter`          |
| `forge-gateway`          | Platform · Go service / API        | Edge/API gateway: routing, authN/Z enforcement, rate limiting                                                                                                                                                                                                                                          | `go-echo-starter`             |
| `forge-identity`         | Platform · Go service / API        | Internal identity **and IdP**: internal SAML/OIDC provider for platform users; accounts, API tokens, RBAC/tenancy, session issuance                                                                                                                                                                    | `go-echo-starter`             |
| `forge-sso`              | Platform · Go service + site       | SSO federation broker: fronts login; authenticates against the internal `forge-identity` IdP or via SAML/OIDC exchange with an external IdP; hosts the login/SSO site                                                                                                                                  | `go-echo-starter`             |
| `forge-cli`              | Platform · Go CLI                  | Operator CLI; talks to the gateway via `forge-sdk`                                                                                                                                                                                                                                                     | `go-cli-starter`              |
| `forge-infrastructure`   | Platform · Infra / deployment      | Definitions that deploy the Forge microservices **themselves** — YAML resource definitions, Open Policy Agent (OPA) policies that gate them, and environment config; applied by `forge-agent`. Forge's *own* operational infra                                                                         | — (YAML + OPA; no Go starter) |
| `forge-inventory`        | Product domain · Go service / API  | Source-of-truth catalog and schemas of the managed servers, network devices, and remote endpoints; upstream for `forge-agent`'s reported inventory                                                                                                                                                     | `go-echo-starter`             |
| `forge-provisioner`      | Product domain · Go service / API  | Desired-state authority: owns directives/rules and reconciliation; enforces agentless devices directly, and hands directives to `forge-agent` for agent-capable endpoints                                                                                                                              | `go-echo-starter`             |
| `forge-agent`            | Product domain · Go daemon         | Endpoint daemon on managed hosts that can run it; enforces desired state locally, collects inventory, runs plugins as separate processes, and applies `forge-infrastructure` definitions. Reaches `forge-inventory`, `forge-identity`, and `forge-provisioner` through `forge-gateway` via `forge-sdk` | `go-cli-starter`              |
| `forge-agent-plugin-sdk` | Product domain · Shared library    | Plugin interface/contract + host-side helpers that every agent plugin builds against — the stable extension point for `forge-agent`                                                                                                                                                                    | `go-library-starter`          |
| `forge-agent-plugins`    | Product domain · Plugin collection | First-party / officially-maintained agent plugin executables, built against `forge-agent-plugin-sdk`                                                                                                                                                                                                   | `go-cli-starter`              |
| `forge-plugin-starter`   | Product domain · Template          | Project-owned scaffold third parties clone to author a new agent plugin executable (pre-wired to `forge-agent-plugin-sdk`)                                                                                                                                                                             | `go-cli-starter`              |

A few decisions are baked into the table above and worth calling out explicitly:

- **No `forge-common`.** Shared internal libraries (config, logging, middleware, telemetry) are baked
  into the `go-*-starter` baselines, so there is no separate common-library repository.
- **`go-cli-starter` covers daemons.** It provides both CLI and long-running daemon scaffolding, so
  `forge-agent` and every plugin executable start from it.
- **`forge-infrastructure` has no Go starter.** It is declarative YAML definitions and OPA policies, not
  a Go service; `forge-agent` applies them.
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

### Auth split

- **`forge-identity`** implements an internal SAML/OIDC IdP and owns internal accounts, tokens, RBAC,
  and tenancy.
- **`forge-sso`** is the federation broker that fronts login. It authenticates users either against the
  internal `forge-identity` IdP *or* via SAML/OIDC token exchange with an external IdP (Okta, Auth0, …).
  Either path resolves to a `forge-identity` principal.

Separating the broker from the IdP and identity store lets the internet-facing SSO surface be hardened
independently of the internal identity system.

### Agent plugin ecosystem

`forge-agent` is extensible via plugins. Plugins run as **separate processes** launched and supervised
by `forge-agent`, so each plugin is its own executable and a failing plugin is isolated from the agent.

- **`forge-agent-plugin-sdk`** — the stable contract plugins build against.
- **`forge-agent-plugins`** — the first-party plugins maintained by the project.
- **`forge-plugin-starter`** — the project-owned scaffold third parties clone to author their own.

Note the distinction from the `go-*-starter` family: those are **external, general-purpose** baselines
that seed Forge repos, whereas `forge-plugin-starter` is a **Forge-specific, project-owned** template
that depends on `forge-agent-plugin-sdk`.

### Forge's own infrastructure

`forge-infrastructure` describes the environments Forge's own services run in. It is declarative rather
than a Go project:

- **YAML definitions** — the desired state of Forge's deployment: hosts, services, and per-environment
  configuration.
- **OPA policies** — Rego policies evaluated against those definitions before they are applied (for
  example, "`forge-identity` is never exposed on a public interface"). OPA returns policy decisions; it
  does not change anything itself.
- **`forge-agent` applies them.** Forge deploys itself with its own agent, so the platform runs on the
  same on-host enforcement path it offers for managed endpoints. This is the one deliberate crossing of
  the two axes: a Platform repository supplies the definitions, and a Product-domain component applies
  them.
- **Possibly a Tengo-based plugin** — an agent plugin that runs [Tengo](https://github.com/d5/tengo)
  scripts for logic YAML can't express. Not yet decided.

## Naming & conventions

- **Slug rule.** Every repository is `forge-<name>` (kebab-case), except this repository (`forge`) and
  the external `go-*-starter` baselines.
- **Shared meta.** Each new repository inherits the meta set this repository defines — GPG + DCO commit
  signing, Conventional Commits (validated on PR titles), the numeric-prefix CI convention
  (200 = PR-triggered, 300 = main-push, 100 = operational/release, 800 = reusable), `CODEOWNERS`, and
  the Apache-2.0 `LICENSE`.
- **One source of truth per contract.** The wire contract lives once in `forge-api-schema`; clients
  consume the generated `forge-sdk` rather than re-deriving types.

## Bootstrapping a repository from a starter

The repeatable procedure for standing up any repository in the inventory:

1. **Create from the baseline.** Instantiate the repo from the mapped `go-*-starter` template (or, for
   `forge-infrastructure`, from the YAML-definitions + OPA-policies layout described above).
2. **Rename** the Go module path and any template placeholders to the `forge-<name>` identity.
3. **Adopt shared meta** from `forge` — signing config, CI workflows (numeric-prefix), `CODEOWNERS`,
   license, and the contribution/security docs.
4. **Wire the contract** — services and clients pin `forge-api-schema` / `forge-sdk`; plugins pin
   `forge-agent-plugin-sdk`.
5. **Register ownership** in `CODEOWNERS` and enable branch protection + PR-title validation.

## Sequencing / phases

A suggested order that keeps each step shippable and unblocks the next:

1. **Contract first** — `forge-api-schema`, then `forge-sdk`. Everything downstream depends on these.
2. **Operational infra** — `forge-infrastructure` definitions and policies, plus a minimal `forge-agent`
   able to apply them, so the first services deploy through the intended path from day one.
3. **Auth spine** — `forge-identity`, then `forge-sso` and `forge-gateway`, so requests can be
   authenticated end to end.
4. **First domain slice** — `forge-inventory` + a thin `forge-cli` path to prove the full request loop.
5. **Enforcement** — `forge-provisioner`, then the full `forge-agent` (directives, inventory reporting)
   and the plugin repos (`forge-agent-plugin-sdk`, `forge-agent-plugins`, `forge-plugin-starter`).

## Open questions

- **Service decomposition depth** — is the current split right, or should some services merge/split?
- **SDK modularity** — single module vs. multi-module `forge-sdk` (per-service clients).
- **Agent bootstrap** — how `forge-agent` applies `forge-infrastructure` before `forge-gateway` and
  `forge-identity` exist (e.g. a local-only apply mode), and how it enrolls once they do.
- **Tengo plugin** — whether a Tengo-scripted agent plugin is needed, and how its scripts are
  sandboxed.
- **Trust zones** — how the Platform/Product-domain boundary maps to network trust zones, especially
  the `forge-agent` → `forge-gateway` ingress from managed hosts.
- **Plugin transport and isolation** — plugins run out-of-process; still open are the transport (e.g.
  gRPC via `hashicorp/go-plugin`), process isolation, and the plugin signing model.

## References

- `CLAUDE.md` — repository intent; application code belongs in separate project repositories.
- `.claude/module-structure.md` — `docs/` role, the CI numeric-prefix convention, intended layout.
- `.claude/conventions.md` — commit signing, Conventional Commits, one-source-of-truth rules.
- [Hugo](https://gohugo.io) — the static-site generator this repository's site is built on.
- [Open Policy Agent](https://www.openpolicyagent.org/docs/latest/) — policy engine and the Rego
  language used for `forge-infrastructure` policies.
- [Tengo](https://github.com/d5/tengo) — embeddable Go scripting language considered for an agent
  plugin.
- [`hashicorp/go-plugin`](https://github.com/hashicorp/go-plugin) — an out-of-process Go plugin system
  over RPC/gRPC.
