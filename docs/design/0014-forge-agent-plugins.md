# 0014 — forge-agent-plugins

- **Status:** Draft
- **Owner:** Nathan Klick
- **Date:** 2026-09-15
- **Summary:** `forge-agent-plugins` is one Go module that builds four first-party plugin executables —
  system facts, packages, files, and services — released together on one version. Each binary ships
  with a SHA-256, a keyless cosign bundle signed from GitHub Actions, and a CycloneDX SBOM, so
  `forge-agent` can verify, pin, and install exactly the plugin versions its desired state names.

> An initial draft with concrete proposals, bounded by the
> [Resolved decisions](0001-project-repositories.md#resolved-decisions) in 0001. Conventions other
> repositories depend on are summarized in [CONVENTIONS.md](CONVENTIONS.md).

## Context & goals

0001 lists `forge-agent-plugins` as the first-party plugin executables, built on
`forge-agent-plugin-sdk` and seeded from `go-cli-starter`
([Repository inventory](0001-project-repositories.md#repository-inventory)). The agent verifies each
plugin's Sigstore signature against trusted publisher identities and pins its SHA-256 before every
launch ([Agent plugin ecosystem](0001-project-repositories.md#agent-plugin-ecosystem)). The contract,
grants, and environment check come from [0013](0013-forge-agent-plugin-sdk.md).

**Goals**

- A modest, justified first plugin set for common server convergence.
- A layout that keeps each binary's dependencies and privileges separate.
- Release artifacts that meet the agent's verification: signature, digest, SBOM, and manifest.
- A documented path from a release to a pinned plugin on a host.

**Non-goals**

- Verification, installation, and sandboxing — [0012](0012-forge-agent.md). The contract — 0013.
- Third-party plugins — [0015](0015-forge-plugin-starter.md). Agentless devices —
  [0011](0011-forge-provisioner.md).
- Templating or scripting on hosts; Tengo runs in the agent and provisioner (0001).

## Proposal

### Responsibilities

- Source, tests, manifests, and releases for the first-party plugins.
- Per-plugin build matrices, SBOMs, signatures, and a signed release index.
- A compatibility statement across plugin releases, SDK versions, and protocol versions.

### Interfaces

#### Repository layout

Proposed: **one Go module with one `cmd/` directory per plugin.**

```
forge-agent-plugins/
├── cmd/forge-plugin-{sysfacts,packages,files,services}/main.go   # serve.Main wiring only
├── internal/
│   ├── sysfacts/  packages/  files/  services/   # one tree per plugin; no imports between them
│   ├── execx/                           # exec: absolute paths, no shell, clean env, caps
│   ├── fsx/                             # os.Root-confined, no-follow, atomic writes
│   └── version/
├── manifests/<name>.yaml                # embedded in each binary; published per release
├── deps/<name>.allow                    # linked-module allowlist per binary
├── plugins.yaml                         # per-plugin platforms, read by Taskfile and CI
├── e2e/                                 # nested module: container-based resource tests
└── Taskfile.yaml, .releaserc.json
```

- **Why one module** — plugins are executables nobody imports. One `go.mod` means one set of
  dependency versions (one gRPC version), one Dependabot stream, and one `govulncheck` run. Go links
  only imported packages, so a dependency added for `sysfacts` stays out of `packages`.
- **Isolation** — a golangci-lint `depguard` rule forbids imports between plugin trees.
  `deps/<name>.allow` is checked against
  `go list -deps -f '{{with .Module}}{{.Path}}{{end}}' ./cmd/forge-plugin-<name>`.

#### Initial plugin set

| Plugin     | Capability                                | Kinds (`forge.servercurio.com/v1alpha1`) | Privileges             |
|------------|-------------------------------------------|------------------------------------------|------------------------|
| `sysfacts` | `facts`                                   | —                                        | unprivileged           |
| `packages` | `resource:…/Package`                      | `Package`                                | root, exec, network    |
| `files`    | `resource:…/File`, `resource:…/Directory` | `File`, `Directory`                      | root, granted paths    |
| `services` | `resource:…/Service`                      | `Service`                                | root, exec `systemctl` |

- **`sysfacts`** — OS release, kernel, CPU, memory, filesystems, interfaces, and uptime. Linux first,
  read from `/proc`, `/sys`, and `/etc/os-release` with `golang.org/x/sys/unix`, which gRPC already
  links. No facts that require root.
- **`packages`** — `Package` (`name`, optional `version`, `state`) through `apt-get`/`dpkg-query` and
  `dnf`/`rpm`. Arguments go in arrays after `--`, and names must match the schema pattern. Repository
  configuration is out of scope at first.
- **`files`** — `File` (`path`, `content`, `mode`, `owner`, `group`, `state`) and `Directory`.
  Content arrives already rendered, so the root plugin has no template engine. Writes are confined to
  granted prefixes by [`os.Root`](https://pkg.go.dev/os#Root): written to a temporary file, synced, and
  renamed.
- **`services`** — `Service` (`name`, `state`, `enabled`). `Plan` reads
  `systemctl show --property=ActiveState,UnitFileState`; `Apply` calls `systemctl`. systemd only.
- **Why these four** — package, file, and service cover basic server convergence, much like the core
  modules of configuration-management tools such as Ansible's builtin `package`, `copy`, and `service`
  (not re-checked). Separate binaries keep the unprivileged fact collector away from root and network
  grants.
- **Deferred** — users and groups, firewall, scheduled jobs, containers, Windows services, and non-Linux
  facts. Each needs its own privilege review.

The kinds' schemas and Go types live in `forge-api-schema` (`desiredstatev1alpha1`, 0002), which has no
dependencies.

### Dependencies

- **Forge** — `forge-agent-plugin-sdk`, `forge-common` (`logging`, `environment`), and
  `forge-api-schema`.
- **Third party** — only the SDK's 14 measured modules (0013) plus zerolog through `forge-common`: about
  16 per binary. That is an estimate until the Forge modules exist; `deps/*.allow` records the actual
  list.
- **Measured, not chosen** — `shirou/gopsutil/v4` v4.26.8 (BSD) adds 3 linked modules per OS:
  - linux: `tklauser/go-sysconf`, `tklauser/numcpus`;
  - darwin: `ebitengine/purego`, `go-sysconf`;
  - windows: `go-ole/go-ole`, `yusufpapurcu/wmi`.

  Measured 2026-09-15; revisit when non-Linux facts are in scope.
- **Tools** (not in `go.mod`) — cyclonedx-gomod v1.12.0 (latest, as the starter pins); cosign v3.1.3
  through `sigstore/cosign-installer` v4.1.2; golangci-lint v2.13.2; and the starter's semantic-release.
  syft goes along with the container build.

### Data & storage

Plugins keep no state. Installed binaries on hosts belong to 0012.

### Security

- **Exec** — `execx` runs only absolute paths from the grant, never through a shell. It sets an
  explicit environment (`LC_ALL=C`, `DEBIAN_FRONTEND=noninteractive`), puts `--` before operands,
  enforces context timeouts, and caps captured output at 1 MiB.
- **Filesystem** — `os.Root` rejects escapes through symlinks; writes are atomic; owner and mode changes
  are explicit.
- **Parsers** — `os-release`, `dpkg-query`, `rpm`, and `systemctl show` output are fuzzed against golden
  fixtures.
- **Supply chain** — SHA-pinned actions and `harden-runner`, as in the starter. `id-token: write` only
  in the release job. `CODEOWNERS` on `.github/workflows/`, since whoever changes the signing workflow
  controls what the identity signs. `govulncheck` and CodeQL.
- **Identity scope** — Fulcio puts the signing workflow's `job_workflow_ref` in the certificate SAN.
  That is the reusable 800 workflow in this repository, so the identity names `forge-agent-plugins`.

### Environment awareness

Plugins require `environment` configuration and perform 0013's check. Tier logic uses `forge-common`.
For example, `packages` makes installing from unauthenticated repositories the last-resort feature
`unauthenticated-packages`, which `production` refuses unless overridden.

### Logging & telemetry

JSON logs go to stderr and the agent re-emits them (0013). Fields: `forge.resource.kind`,
`forge.resource.name`, `forge.resource.changed`. Commands are logged by executable and argument count
only, since arguments can carry resource data. No telemetry export.

### Configuration

Prefixes are `FORGE_PLUGIN_SYSFACTS`, `_PACKAGES`, `_FILES`, and `_SERVICES`, with the SDK's
`environment` and `rpc` keys and `forge-common`'s `logging`. Grants carry paths and executables, so
plugin keys stay few:

| YAML          | Variable                             | Default                          |
|---------------|--------------------------------------|----------------------------------|
| `manager`     | `FORGE_PLUGIN_PACKAGES_MANAGER`      | `auto` (`apt` or `dnf` by probe) |
| `lockTimeout` | `FORGE_PLUGIN_PACKAGES_LOCK_TIMEOUT` | `5m`                             |
| `collectors`  | `FORGE_PLUGIN_SYSFACTS_COLLECTORS`   | all                              |

### Build, release & versioning

#### Build matrix

`plugins.yaml` declares each plugin's platforms, and `task build` loops over them with the starter's
`CGO_ENABLED=0`, `-trimpath`, and `-ldflags "-s -w"`. All four plugins ship `linux/amd64` and
`linux/arm64` first. The starter's other four targets (darwin and windows) are added per plugin when
implemented, and pull requests cross-compile every declared platform. The Dockerfile, GHCR push, and
container SBOM are removed, since plugins are host binaries.

#### One release train

Proposed: **one semantic-release version for the repository** (`vX.Y.Z`, `v0.x` per CONVENTIONS).
Every release rebuilds every plugin, and commit scopes such as `feat(packages): …` group the notes.

- **Why** — changes to the shared `go.mod`, `go.sum`, or `internal/execx`, such as a gRPC security fix,
  must release every binary. semantic-release-monorepo 8.0.2 assigns commits to a package only by files
  under that package's directory, so a dependency fix at the repository root would release nothing.
- **Cost** — every release gives each plugin a new version and digest. The agent pins plugins
  individually, so operators still upgrade them one at a time.

#### Assets, signing, and SBOMs

Per plugin and platform, with `<asset>` = `forge-plugin-<name>-<os>-<arch>`:

| Asset                                          | Purpose                                                    |
|------------------------------------------------|------------------------------------------------------------|
| `<asset>`, `<asset>.sha256`                    | executable and the digest the agent pins                   |
| `<asset>.sigstore.json`                        | cosign bundle: signature, certificate, Rekor proof         |
| `<asset>.cdx.json` + bundle                    | CycloneDX SBOM with licenses                               |
| `forge-plugin-<name>.manifest.yaml` + bundle   | capabilities, privileges, protocol versions                |
| `plugins-index.json` + bundle                  | name, version, platform, SHA-256, protocols, SDK version   |

`.releaserc.json` keeps the starter's analyzer rules, with
`publishCmd: "task build && task hash && task sign && task sbom && task index && task verify"`. The
release job installs cosign first. `task sign` and `task verify` run:

```sh
repo=servercurio/forge-agent-plugins
wf=.github/workflows/800-call-semantic-release.yaml
cosign sign-blob --yes --bundle "bin/${f}.sigstore.json" "bin/${f}"
cosign verify-blob "bin/${f}" --bundle "bin/${f}.sigstore.json" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity "https://github.com/${repo}/${wf}@${GITHUB_REF}"
```

- **Keyless** — GitHub OIDC (`https://token.actions.githubusercontent.com`) through Fulcio, so there
  are no long-lived keys. The bundle carries the Rekor inclusion proof, so agents can verify offline
  with a `trusted_root.json`; cosign v3.1.3 deprecates `--offline` in favor of `--bundle` plus
  `--trusted-root`.
- **`task verify`** fails the release if the identity drifts, for example after a workflow rename.
- **SBOMs** — `cyclonedx-gomod app -licenses -main cmd/forge-plugin-<name>` per plugin and platform,
  since each binary links a different package set. Whether it honors `GOOS`/`GOARCH` is unverified, so a
  test compares its components with `go version -m <asset>`. `-licenses` surfaces the MPL-2.0 go-plugin
  and yamux modules.
- **Kept and replaced** — the starter's `attest-build-provenance` and `attest-sbom` steps stay. Cosign
  bundles replace the starter's GPG `.sha256.asc` files, leaving agents one signature system. Release
  commits stay GPG-signed.

#### How the agent installs trusted versions

Proposed for [0011](0011-forge-provisioner.md) and [0012](0012-forge-agent.md), with the kinds in 0002:

```yaml
apiVersion: forge.servercurio.com/v1alpha1
kind: PluginPublisher
metadata: { name: servercurio }
spec:
  keyless:
    issuer: https://token.actions.githubusercontent.com
    repository: servercurio/forge-agent-plugins
    workflow: .github/workflows/800-call-semantic-release.yaml
    refs: [refs/heads/main, "refs/heads/release/*"]
---
apiVersion: forge.servercurio.com/v1alpha1
kind: AgentPlugin
metadata: { name: packages }
spec:
  publisher: servercurio
  version: 0.4.0
  baseURL: https://github.com/servercurio/forge-agent-plugins/releases/download/v0.4.0
  sha256: { linux/amd64: "…", linux/arm64: "…" }
  grant: { capabilities: [resource:forge.servercurio.com/v1alpha1/Package] }
```

1. **Import** — `forge-provisioner` verifies `plugins-index.json` against the publisher and writes the
   digests into `AgentPlugin`. Nothing auto-updates to "latest".
2. **Download** — the agent fetches `<asset>` and its bundle from `baseURL`. That may be a mirror,
   because detached bundles survive mirroring.
3. **Verify** — the agent checks the SHA-256 against the directive and the bundle against the
   publisher. The identity is built from structured fields, never a free-form regular expression.
4. **Compatibility** — the manifest's `protocolVersions` must overlap the agent's.
5. **Install** — into a root-owned `…/plugins/<name>/<sha256>/`, keeping the previous digest for
   rollback. Every launch pins that digest through `SecureConfig`.

#### SDK compatibility

Manifests record `protocolVersions`, and `plugins-index.json` records the SDK version from build info.
Release notes carry a table of plugin release, SDK version, and protocols. Dependabot keeps the SDK
current, and when the SDK drops a protocol, plugins keep serving N-1 until the agent's window closes
(0013).

### Testing

- **Unit** — `execx.Runner` fakes, golden command output, and handlers through
  `plugintest.InProcess`.
- **Conformance** — each built binary through `plugintest.Launch` and `Conformance`, including
  idempotence.
- **End to end** (nested `e2e`):
  - `packages` in `ubuntu:noble` and `rockylinux:9` containers, and `files` in a container;
  - `services` on a GitHub-hosted Ubuntu runner, assuming `sudo systemctl` is allowed (unverified).
- **Hygiene** — `-race`, fuzzers, allowlists, cross-compiling all declared platforms, and a release dry
  run.

## Alternatives considered

- **Module per plugin** (`go.work`) — four Dependabot streams and possible gRPC skew, with no consumer
  benefit.
- **Repository per plugin** — contradicts 0001's single `forge-agent-plugins` repository.
- **Per-plugin versions with semantic-release-monorepo** — misses dependency fixes made at the root.
- **One multi-call binary** — every plugin would link every dependency and share one digest, and fact
  collection would run from the root binary.
- **[GoReleaser](https://goreleaser.com) v2.18.1** — a capable release tool, but a second toolchain
  beside the starter's semantic-release and Taskfile.
- **Key-based cosign (KMS)** — adds key custody and rotation. Keyless ties signatures to this workflow
  and still verifies offline. Third parties may use keys (0015).
- **GitHub artifact attestations only** — also Sigstore-backed, but fetched per artifact from GitHub,
  while bundles next to assets work with mirrors.
- **gopsutil** (measured above) and **go-systemd over D-Bus** (adds `godbus`; not measured).
- **Deviation from [CONVENTIONS.md](CONVENTIONS.md)** (*Go modules and layout*: binaries take the repo
  name) — binaries are named `forge-plugin-<name>`, which maps to `FORGE_PLUGIN_<NAME>`.

## Open questions

- **First release** — Linux only, with apt and dnf. Acceptable?
- **Core facts** — do OS, architecture, and hostname belong in the agent (0012), leaving `sysfacts` for
  extended facts?
- **Trusted root** — how does `trusted_root.json` reach air-gapped agents and stay current (0012)?
- **GPG** — keep GPG hash signatures beside cosign bundles for manual verification?
- **Index** — is `plugins-index.json` defined here, or as a kind in 0002?
- **Kinds** — the kind names and the `forge.servercurio.com/v1alpha1` group, to confirm with 0002 and
  0011.
- **Growth** — at what plugin count, if any, do per-plugin versions pay for their tooling?

## References

- [0001](0001-project-repositories.md), [CONVENTIONS.md](CONVENTIONS.md),
  [0002](0002-forge-api-schema.md), [0004](0004-forge-common.md), [0013](0013-forge-agent-plugin-sdk.md).
- [go-cli-starter](https://github.com/servercurio/go-cli-starter) — `Taskfile.yaml` (six targets, `hash`,
  `sign`, `sbom`), `.releaserc.json`, `800-call-semantic-release.yaml` (`id-token: write`, attest steps).
- Cosign — [signing blobs](https://docs.sigstore.dev/cosign/signing/signing_with_blobs/),
  [verifying](https://docs.sigstore.dev/cosign/verifying/verify/), and
  [Sigstore bundle](https://docs.sigstore.dev/about/bundle/).
- [cosign v3.1.3 verify options](https://github.com/sigstore/cosign/blob/v3.1.3/cmd/cosign/cli/options/verify.go)
  — `--offline` deprecation, `--trusted-root`.
- [OIDC in Fulcio](https://docs.sigstore.dev/certificate_authority/oidc-in-fulcio/) — GitHub SAN
  `https://github.com/{job_workflow_ref}`.
- [Fulcio OID info](https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md);
  [GitHub Actions OIDC](https://docs.github.com/en/actions/reference/security/oidc).
- [sigstore/cosign-installer](https://github.com/sigstore/cosign-installer);
  [GitHub artifact attestations](https://docs.github.com/en/actions/concepts/security/artifact-attestations).
- [semantic-release-monorepo](https://github.com/pmowrer/semantic-release-monorepo);
  [commit-analyzer](https://github.com/semantic-release/commit-analyzer).
- [cyclonedx-gomod](https://github.com/CycloneDX/cyclonedx-gomod); [gopsutil](https://github.com/shirou/gopsutil);
  [Go `os.Root`](https://pkg.go.dev/os#Root); [GoReleaser](https://goreleaser.com).
- [Ansible builtin modules](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html)
  — not re-checked (rate-limited 2026-09-15).
