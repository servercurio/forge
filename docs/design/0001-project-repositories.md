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
   operator ── forge-cli ─┐   │
   3rd-party client ──────┤   ▼
                          ▼ forge-sso ──► forge-identity (internal SAML/OIDC IdP,
                    forge-gateway ◄────────┘             accounts, tokens, RBAC, tenancy)
                          │
          ┌───────────────┼────────────────┐
          ▼               ▼                 ▼
   forge-inventory   forge-provisioner   (other domain services…)
        ▲                  │
        │ inventory        │ desired-state directives
        │                  ▼
        └──────────── forge-agent ──► loads plugins (forge-agent-plugin-sdk)
                     (on agent-capable hosts)

   forge-provisioner ──► agentless devices (via each device's own API)

   Shared contract/clients: forge-api-schema ──► forge-sdk (used by forge-cli, forge-agent, 3rd party)
   Forge's own deployment:  forge-infrastructure (IaC for everything above)
```

## Repository inventory

| Repository              | Axis · Category                | Purpose                                                                 | Starter baseline                 |
|-------------------------|--------------------------------|-------------------------------------------------------------------------|----------------------------------|
| `forge` (this)          | Platform · Docs & site         | Documentation, design assets, website; shared-meta source of truth      | — (already exists)               |
| `forge-api-schema`      | Platform · Shared library      | API contracts (protobuf/OpenAPI) — the inter-service and client schema  | `go-library-starter`             |
| `forge-sdk`             | Platform · Shared library      | Generated Go client SDK for the public API; used by `forge-cli`, `forge-agent`, and 3rd-party clients | `go-library-starter` |
| `forge-gateway`         | Platform · Go service / API    | Edge/API gateway: routing, authN/Z enforcement, rate limiting           | `go-echo-starter`                |
| `forge-identity`        | Platform · Go service / API    | Internal identity **and IdP**: internal SAML/OIDC provider for platform users; accounts, API tokens, RBAC/tenancy, session issuance | `go-echo-starter` |
| `forge-sso`             | Platform · Go service + site   | SSO federation broker: fronts login; authenticates against the internal `forge-identity` IdP or via SAML/OIDC exchange with an external IdP; hosts the login/SSO site | `go-echo-starter` |
| `forge-infrastructure`  | Platform · Infra / deployment  | IaC and tooling that deploys the Forge microservices **themselves** (Terraform, Helm/Kustomize, env config) — Forge's *own* operational infra | — (IaC; no Go starter) |
| `forge-inventory`       | Product domain · Go service / API | Source-of-truth catalog and schemas of the managed servers, network devices, and remote endpoints; upstream for `forge-agent`'s reported inventory | `go-echo-starter` |
| `forge-provisioner`     | Product domain · Go service / API | Desired-state authority: owns directives/rules and reconciliation; enforces agentless devices directly, and hands directives to `forge-agent` for agent-capable endpoints | `go-echo-starter` |
| `forge-agent`           | Product domain · Go daemon     | Endpoint daemon on managed hosts that can run it; enforces desired state locally, collects inventory, loads plugins; talks to the platform via `forge-sdk` | `go-cli-starter` *(daemon baseline)* |
| `forge-agent-plugin-sdk`| Product domain · Shared library | Plugin interface/contract + host-side helpers that every agent plugin builds against — the stable extension point for `forge-agent` | `go-library-starter` |
| `forge-agent-plugins`   | Product domain · Plugin collection | First-party / officially-maintained agent plugins, built against `forge-agent-plugin-sdk` | `go-library-starter` |
| `forge-plugin-starter`  | Product domain · Template      | Project-owned scaffold third parties clone to author a new agent plugin (pre-wired to `forge-agent-plugin-sdk`) | `go-library-starter` |
| `forge-cli`             | Product domain · Go CLI        | Operator/admin CLI; talks to the gateway via `forge-sdk`                | `go-cli-starter`                 |

A few decisions are baked into the table above and worth calling out explicitly:

- **No `forge-common`.** Shared internal libraries (config, logging, middleware, telemetry) are baked
  into the `go-*-starter` baselines, so there is no separate common-library repository.
- **`forge-infrastructure` has no Go starter.** It is IaC (Terraform modules, Helm/Kustomize charts,
  environment config), not a Go service.
- **The service split is provisional.** New domain services will appear as the product grows.

### Desired-state model — one authority, two enforcement paths

`forge-provisioner` is the **single** desired-state authority. It owns the directives/rules and drives
reconciliation for every managed endpoint. Enforcement then takes one of two paths, chosen by endpoint
class:

- **Agent-capable endpoints** receive the directives from `forge-provisioner` and enforce them
  *on-host* via `forge-agent`.
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

`forge-agent` is extensible via plugins:

- **`forge-agent-plugin-sdk`** — the stable contract plugins build against.
- **`forge-agent-plugins`** — the first-party plugins maintained by the project.
- **`forge-plugin-starter`** — the project-owned scaffold third parties clone to author their own.

Note the distinction from the `go-*-starter` family: those are **external, general-purpose** baselines
that seed Forge repos, whereas `forge-plugin-starter` is a **Forge-specific, project-owned** template
that depends on `forge-agent-plugin-sdk`.

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
   `forge-infrastructure`, from the IaC layout).
2. **Rename** the Go module path and any template placeholders to the `forge-<name>` identity.
3. **Adopt shared meta** from `forge` — signing config, CI workflows (numeric-prefix), `CODEOWNERS`,
   license, and the contribution/security docs.
4. **Wire the contract** — services and clients pin `forge-api-schema` / `forge-sdk`; plugins pin
   `forge-agent-plugin-sdk`.
5. **Register ownership** in `CODEOWNERS` and enable branch protection + PR-title validation.

## Sequencing / phases

A suggested order that keeps each step shippable and unblocks the next:

1. **Contract first** — `forge-api-schema`, then `forge-sdk`. Everything downstream depends on these.
2. **Auth spine** — `forge-identity`, then `forge-sso` and `forge-gateway`, so requests can be
   authenticated end to end.
3. **First domain slice** — `forge-inventory` + a thin `forge-cli` path to prove the full request loop.
4. **Enforcement** — `forge-provisioner`, then `forge-agent` and the plugin repos
   (`forge-agent-plugin-sdk`, `forge-agent-plugins`, `forge-plugin-starter`).
5. **Operational infra** — `forge-infrastructure`, brought up alongside the first deployable services.

## Open questions

- **Service decomposition depth** — is the current split right, or should some services merge/split?
- **SDK modularity** — single module vs. multi-module `forge-sdk` (per-service clients).
- **`forge-infrastructure` target** — Kubernetes/Helm, Terraform, or both.
- **Trust zones** — how the Platform/Product-domain boundary maps to network trust zones, especially
  the `forge-agent` → `forge-inventory` ingress.
- **Plugin mechanism** — in-process Go plugins vs. out-of-process subprocess/gRPC (e.g.
  `hashicorp/go-plugin`), and the plugin isolation and signing model.

## References

- `CLAUDE.md` — repository intent; application code belongs in separate project repositories.
- `.claude/module-structure.md` — `docs/` role, the CI numeric-prefix convention, intended layout.
- `.claude/conventions.md` — commit signing, Conventional Commits, one-source-of-truth rules.
- [Hugo](https://gohugo.io) — the static-site generator this repository's site is built on.
