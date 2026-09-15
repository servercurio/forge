# 0005 — forge-infrastructure

- **Status:** Draft
- **Owner:** Nathan Klick
- **Date:** 2026-09-15
- **Summary:** `forge-infrastructure` is an Ansible project that deploys Forge's own services as
  Podman Quadlet units on Linux hosts, one inventory per environment. Conftest checks its OPA policies
  in pull-request CI, and a dedicated control node runs signed, merged commits with credentials pulled
  from an external secret manager. It also defines the key ceremony that creates an environment and
  the delivery of single-use service enrollment tokens.

> An initial draft with concrete proposals, bounded by the
> [Resolved decisions](0001-project-repositories.md#resolved-decisions) in 0001. Conventions other
> repositories depend on are summarized in [CONVENTIONS.md](CONVENTIONS.md).

## Context & goals

0001 makes this repository Forge's *own* deployment: Ansible playbooks, roles, and inventories gated
by OPA policies, with Conftest in pull-request CI and a control node that runs merged playbooks so
deployment credentials never live in CI
([Forge's own infrastructure](0001-project-repositories.md#forges-own-infrastructure)). It must not
depend on `forge-agent` or `forge-provisioner`. Each inventory supplies its environment's name, tier,
ID, and CA bundle, and the control node holds a root-signed certificate and delivers service
enrollment tokens ([Environment identity](0001-project-repositories.md#environment-identity)).

**Goals**

- A repository layout where one inventory fully describes one environment.
- Policies that block unsafe inventories and tasks before merge, and again before every run.
- A repeatable, auditable ceremony to create, rotate, and destroy an environment's trust anchors.
- Service certificate bootstrap with no long-lived shared secrets on service hosts.

**Non-goals**

- Managing customer endpoints — that is `forge-provisioner` and `forge-agent`.
- Certificate issuance logic and token formats — [0006](0006-forge-identity.md).
- Service-internal configuration semantics — each service's own document.
- Choosing a telemetry backend; this repository deploys only an in-environment OTLP collector.

## Proposal

### Responsibilities

- Inventories, roles, and playbooks for every Forge service, PostgreSQL, and the OTLP collector.
- Rego policies for inventories and Ansible content, with their unit tests.
- The control node's own configuration and its run procedure.
- Runbooks for the environment key ceremony, rotation, and destruction (`docs/runbooks/`).

### Interfaces

#### Deployment target

Proposed: **Linux hosts running each service as a Podman Quadlet `.container` unit**, with images
pinned by digest. Quadlet turns each `.container` file into a systemd service, accepts image digests in
`Image=`, and mounts Podman secrets as files through `Secret=`
([podman-systemd.unit](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)). The
`go-*-starter` release already publishes multi-arch GHCR images with signed SBOMs
([go-echo-starter `.releaserc.json`](https://github.com/servercurio/go-echo-starter/blob/main/.releaserc.json)).

Rationale: 0001 has the control node deliver a single-use token to **each service instance**. A
long-lived host with its own key file matches that model; Kubernetes pods scaled or rescheduled by the
cluster would each need a new token the control node never sees (see Alternatives).

#### Repository layout

```
forge-infrastructure/
├── ansible.cfg
├── requirements.yml                  # collections, exact versions
├── execution-environment.yml         # ansible-builder definition for the pinned runtime image
├── inventories/
│   ├── qa-east/
│   │   ├── hosts.yaml                # groups: identity, gateway_operator, gateway_agent, sso, ...
│   │   └── group_vars/all/
│   │       ├── environment.yaml      # name, tier, id, caBundle
│   │       ├── images.yaml           # image@sha256 digest per service
│   │       └── secrets.yaml          # secret-manager references only, never values
│   └── prod-east/…
├── roles/
│   ├── forge_host/                   # OS hardening, podman, firewall, time sync
│   ├── forge_postgresql/
│   ├── forge_otel_collector/
│   ├── forge_service/                # shared: Quadlet unit, config file, enrollment token delivery
│   ├── forge_identity/  forge_gateway/  forge_sso/  forge_inventory/  forge_provisioner/
│   └── forge_control_node/
├── playbooks/
│   ├── site.yaml                     # full converge, in dependency order
│   ├── service.yaml                  # one service: -e forge_service=forge-inventory
│   └── rotate-credentials.yaml
├── policy/
│   ├── inventory/*.rego              # evaluated against `ansible-inventory --list` JSON
│   ├── content/*.rego                # evaluated against role and playbook YAML
│   └── **/*_test.rego
├── molecule/                         # role scenarios with the podman driver
└── docs/runbooks/                    # ceremony, rotation, destruction
```

`inventories/<name>/group_vars/all/environment.yaml` is the environment's single declaration:

```yaml
forge_environment:
  name: qa-east
  tier: staging
  id: 3k7q2m9x4t6v8b1n5c0d2f7h9j   # 26-char base32, from the ceremony
  caBundle: files/qa-east/roots.pem # public roots only; two during root rotation
  overrides: []                     # last-resort feature names, see Environment awareness
```

The `forge_service` role renders these into each service's YAML `environment` block (CONVENTIONS
keys) and never into `FORGE_*` variables, so `ps` and unit files do not expose configuration.

#### Pinned toolchain

| Component                                                                     | Version                  |
|-------------------------------------------------------------------------------|--------------------------|
| [ansible-core](https://github.com/ansible/ansible/releases)                   | 2.21.4                   |
| [ansible-navigator](https://github.com/ansible/ansible-navigator/releases)    | v26.8.0                  |
| [ansible-lint](https://github.com/ansible/ansible-lint/releases)              | v26.8.0                  |
| [molecule](https://github.com/ansible/molecule/releases)                      | v26.8.0                  |
| [Conftest](https://github.com/open-policy-agent/conftest/releases)            | v0.70.0                  |
| [OPA](https://github.com/open-policy-agent/opa/releases) (`opa test`)         | v1.20.2                  |
| `containers.podman`, `community.postgresql`, `ansible.posix`                  | 1.20.2, 5.0.0, 2.2.2     |

Versions checked on 2026-09-15 from GitHub releases and the
[Galaxy API](https://galaxy.ansible.com/). Collections come from `requirements.yml` with exact
versions and are baked into the execution environment image, which is pinned by digest. Dependabot does
not cover Galaxy collections (unverified); a 100-series workflow proposes bumps.

#### Policies

Policies use Rego v1 syntax. Two input shapes cover what Conftest cannot see in raw YAML:

- **Inventory policies** run on the merged view: CI runs
  `ansible-inventory -i inventories/<env> --list` and pipes the JSON to
  `conftest test --namespace forge.inventory`. Raw `group_vars` files would miss precedence and merges.
- **Content policies** run on role and playbook YAML with `--namespace forge.content`.

```rego
package forge.inventory

import rego.v1

tiers := {"production", "staging", "test", "development"}

deny contains msg if {
    env := input._meta.hostvars[host].forge_environment
    not env.tier in tiers
    msg := sprintf("%s: unknown tier %q", [host, env.tier])
}

deny contains msg if {
    some host in input.identity.hosts
    input._meta.hostvars[host].forge_public_interface
    msg := sprintf("%s: forge-identity must not bind a public interface", [host])
}
```

Initial rule set (proposed):

| Rule                                                                                      | Scope     |
|-------------------------------------------------------------------------------------------|-----------|
| `name` is a DNS label, `tier` is one of four, `id` is 26 lowercase base32 characters      | inventory |
| `forge-identity` and PostgreSQL never bind public interfaces                               | inventory |
| Every image is referenced by `@sha256:` digest                                            | inventory |
| `production`: `identity_ca_backend` is not `kek-sealed` unless in `overrides`             | inventory |
| `production`: no `plaintext-telemetry` or other last-resort feature without an override    | inventory |
| Service enrollment token TTL ≤ 1h; service certificate lifetime is exactly 7 days          | inventory |
| One environment ID per inventory, not reused by any other inventory in the repository      | inventory |
| Values in `secrets.yaml` are secret-manager references, never literals                     | inventory |
| Tasks that use a secret variable set `no_log: true`                                        | content   |
| No `shell`/`command` tasks without `changed_when`, and no `validate_certs: false`          | content   |

#### CI and the control node

- **`200-flow-pull-request-checks.yaml`** — ansible-lint, `ansible-playbook --syntax-check`,
  `opa test policy/`, Conftest on every inventory and on content, and molecule for changed roles. It
  has no deployment credentials and no network path to any environment.
- **`300-flow-main-branch-checks.yaml`** — repeats the checks on `main`.
- **Control node** — proposed as a dedicated, hardened VM per environment tier group that runs
  `ansible-navigator` in the pinned execution environment. A systemd timer fetches `main`, runs
  `git verify-commit` against an allowlist of maintainers' GPG keys (every Forge repository requires
  GPG-signed commits, per [Naming & conventions](0001-project-repositories.md#naming--conventions)),
  re-runs Conftest locally, and only then runs
  `site.yaml` in check mode, followed by apply for inventories whose `autoApply` is true. `production`
  applies require a manual `forge-deploy apply <env> <commit>` on the node.
- **AWX** was 0001's example. Its releases have been paused since 24.6.1 (July 2024) during a
  refactor ([AWX README](https://github.com/ansible/awx)), so it is kept as an alternative.

### Dependencies

- **Forge repositories** — container images released by each service repository; enrollment and
  renewal endpoints from [0006](0006-forge-identity.md). No Go modules.
- **Runtime** — Podman on target hosts, PostgreSQL from distribution packages, and an OpenTelemetry
  Collector image pinned by digest.
- **Secret manager** — one external secret manager per environment, reached only from the control node
  (proposed: HashiCorp Vault through `community.hashi_vault` 7.1.0, or the cloud provider's manager).

### Data & storage

No application data. The repository holds inventories, public CA bundles, and policies. The control node
keeps run logs and the last applied commit per environment; ceremony records (signed transcripts,
certificate fingerprints) are committed under `docs/runbooks/records/<env>/`.

### Security

#### Environment-creation key ceremony

Performed by two people on an offline, freshly imaged workstation, with a written transcript:

1. **Generate the environment ID** — 128 random bits, 26 characters of lowercase base32 (CONVENTIONS).
   IDs are never reused, even after destruction.
2. **Create the root CA** in an offline HSM or an isolated cloud KMS project: ECDSA P-384 (proposed),
   10-year validity, a `spiffe://<environment-id>` URI SAN, and a URI name constraint permitting only
   the trust domain. Go matches URI name constraints against the URI host
   ([`constraints.go`](https://github.com/golang/go/blob/master/src/crypto/x509/constraints.go)),
   which for a SPIFFE ID is the environment ID.
3. **Sign `forge-identity`'s intermediate** from a CSR whose key was generated inside the environment's
   HSM or KMS ([0006](0006-forge-identity.md)): path length 0, same name constraint, 2-year validity,
   renewed by a repeat ceremony at two-thirds of its lifetime.
4. **Sign the control node's bootstrap certificate** from a CSR generated on the control node:
   `spiffe://<environment-id>/control-node/<node-name>`, client authentication only, 30 days. After
   `forge-identity` is running, the control node renews through 0006's renewal endpoint like a service.
5. **Publish** the root certificate and an initial root CRL (`nextUpdate` 180 days, re-signed at each
   ceremony) and commit the bundle and fingerprints in a pull request that adds the inventory.

`production` requires an HSM or KMS for both root and intermediate. `test` and `staging` may use the
KEK-sealed intermediate store, gated as in 0006.

#### Service enrollment token delivery

1. `forge_service` checks the instance's certificate. A valid certificate with more than one third of
   its lifetime left needs nothing: services renew themselves.
2. Otherwise the control node calls `forge-identity` over mutual TLS with its control-node certificate,
   requesting a service enrollment token for `service/<repository>` and this host, TTL 15 minutes.
3. The token is written with `no_log: true` into a Podman secret mounted at
   `/run/secrets/forge-enrollment-token`, then the unit starts. The service enrolls with a CSR through
   `forge-sdk` `pkg/enroll` and keeps its ECDSA P-256 key in a `0600` file on its volume.
4. When `/readyz` passes, the role removes the Podman secret. An unused token expires on its own.

#### Secrets handling

- Secret values never enter Git, Ansible Vault files, CI, or unit files. `secrets.yaml` holds only
  references that lookups resolve on the control node at run time.
- Secrets reach services as mounted files named by `...File` keys (CONVENTIONS), never as variables.
- The control node authenticates to the secret manager with a short-lived, host-bound credential, and
  can read only its environments' paths.
- The KEK for a KEK-sealed store is delivered as a secret file at startup and never written beside the
  sealed key (0001).

### Environment awareness

- The inventory's `forge_environment.tier` drives role defaults: `production` and `staging` enable
  TLS-only listeners, disable OpenAPI UIs, and require `@sha256` images; `development` may run all
  services on one host.
- Last-resort features appear in `overrides` only through a pull request that Conftest flags for
  `CODEOWNERS` review, so every override is visible in history as well as in service logs.
- Control nodes are dedicated per tier group: `production` inventories are unreachable from the
  non-production control node.

### Logging & telemetry

Ansible runs on the control node use the JSON callback, written to the node's journal with
`deployment.environment.name`, `forge.environment.tier`, and `forge.environment.id`. Each run records the
commit, the playbook, and a changed-task summary. Deployed collectors receive OTLP/HTTP from services
and forward to the environment's backend.

### Configuration

Ansible variables use the `forge_` prefix in snake_case. Rendered service configuration uses
CONVENTIONS YAML keys and `...File` references. There is no `FORGE_INFRASTRUCTURE_` prefix because
nothing here is a Go executable.

### Build, release & versioning

- Not versioned as a library: `main` is the deployable state, and each environment records its applied
  commit. Tags `vX.Y.Z` mark execution environment image releases.
- The execution environment image is built with ansible-builder by a 100-series workflow, signed, and
  pinned by digest in the control node's configuration.

### Testing

- `opa test` with fixtures for every rule, including passing and failing inventories.
- molecule scenarios per role using the podman driver, with idempotence checks.
- A disposable `development` inventory exercised nightly end to end, including a simulated ceremony
  with SoftHSM and enrollment of every service.

## Alternatives considered

- **Kubernetes with the starters' Helm charts** — the charts exist and harden pods well, but per-pod
  enrollment conflicts with 0001's control-node token delivery; it would need a per-node enrollment
  helper or StatefulSets with persistent volumes. Worth revisiting for large deployments.
- **AWX as the control node** — web UI, RBAC, and credential injection, but releases are paused and
  it adds a Kubernetes operator and database to Forge's footprint.
- **[Semaphore UI](https://github.com/semaphoreui/semaphore)** (v2.19.12) — lighter web runner; an extra
  service to secure for one operator's use.
- **Ansible Vault for secrets** — simple, but encrypted secrets would live in Git and the vault
  password becomes a long-lived shared secret.
- **Conftest on raw YAML only** — misses variable precedence; kept only for content rules.
- **Native binaries under systemd** — fewer moving parts, but loses image digests, signed image SBOMs,
  and the Podman secret mount.
- **Root-signed service certificates for every service** — avoids the token step, but keeps the root
  online, which 0001 rules out.

## Open questions

- **Collector and PostgreSQL identity** — neither is a Forge repository, so neither fits the
  CONVENTIONS SPIFFE paths. Proposed: a separate per-environment infrastructure CA held by the secret
  manager, never trusted for Forge mutual TLS. Needs a CONVENTIONS decision (see also 0004).
- **Service enrollment route** — this draft calls `forge-identity` directly, as proposed in 0006;
  confirm with [0008](0008-forge-gateway.md).
- **Control-node certificate lapse** — if the control node is down past its certificate's lifetime, is
  a root ceremony acceptable to recover, or should two control nodes cross-renew?
- **Secret manager** — Vault, cloud managers, or both behind one lookup plugin?
- **Intermediate revocation checks** — should peers check the root CRL for the intermediate, and
  who publishes it between ceremonies?
- **Galaxy dependency updates** — Dependabot support for `requirements.yml` is unverified.

## References

- [0001 — Project Repositories](0001-project-repositories.md) — Forge's own infrastructure,
  environment awareness and identity, service certificate bootstrap.
- [0003 — forge-sdk](0003-forge-sdk.md) — `pkg/enroll`;
  [0004 — forge-common](0004-forge-common.md) — last-resort features and collector questions;
  [0006](0006-forge-identity.md) — CA and tokens.
- [CONVENTIONS.md](CONVENTIONS.md) — SPIFFE IDs, ID syntax, environment keys, `...File` secrets.
- [Ansible](https://docs.ansible.com/) and
  [ansible-builder execution environments](https://docs.ansible.com/projects/builder/en/latest/)
  (URL unverified).
- [Conftest](https://www.conftest.dev) and [Open Policy Agent](https://www.openpolicyagent.org/docs/latest/).
- [Podman Quadlet `podman-systemd.unit`](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
  — `Image=` digests, `Secret=`, generated services.
- [AWX](https://github.com/ansible/awx) — README notice that releases are paused.
- [Semaphore UI](https://github.com/semaphoreui/semaphore).
- [Go `crypto/x509` name constraints](https://github.com/golang/go/blob/master/src/crypto/x509/constraints.go)
  — URI constraints match the URI host (read from the Go 1.27.1 source).
- [SPIFFE ID](https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE-ID.md) and
  [X.509-SVID](https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md).
- [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280) — name constraints, path length, CRLs.
- [`community.hashi_vault`](https://galaxy.ansible.com/ui/repo/published/community/hashi_vault/),
  [`containers.podman`](https://galaxy.ansible.com/ui/repo/published/containers/podman/),
  [`community.postgresql`](https://galaxy.ansible.com/ui/repo/published/community/postgresql/).
- [go-echo-starter release configuration](https://github.com/servercurio/go-echo-starter/blob/main/.releaserc.json)
  — images, SBOMs, and Helm charts published per release.
