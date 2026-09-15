# 0012 — forge-agent

- **Status:** Draft
- **Owner:** Nathan Klick
- **Date:** 2026-09-15
- **Summary:** `forge-agent` is one binary run as two processes: an unprivileged network daemon that
  enrolls, renews, pulls signed directive bundles, and reports, and a privileged executor with no network
  access that verifies bundles, re-checks OPA, runs Tengo, enforces resources, and launches verified
  plugins. Keys are TPM-backed where possible; plugin signatures are verified centrally by
  `forge-provisioner`, so the agent links no Sigstore verifier.

> An initial draft with concrete proposals, bounded by the
> [Resolved decisions](0001-project-repositories.md#resolved-decisions) in 0001. Conventions other
> repositories depend on are summarized in [CONVENTIONS.md](CONVENTIONS.md).

## Context & goals

0001 defines the agent's enrollment ([Agent enrollment](0001-project-repositories.md#agent-enrollment)),
its on-host OPA re-check and Tengo sandbox
([Desired-state format](0001-project-repositories.md#desired-state-format)), its plugin model
([Agent plugin ecosystem](0001-project-repositories.md#agent-plugin-ecosystem)), and that it records its
environment from enrollment ([Environment awareness](0001-project-repositories.md#environment-awareness)).
It reaches Forge only through the gateway's agent ingress via `forge-sdk`.

**Goals**

- Nothing that parses network input runs as root.
- Idempotent, converge-then-verify enforcement that keeps working while offline.
- Every plugin launch verified: a SHA-256 pin from a provisioner-signed bundle, for a release whose
  Sigstore signature `forge-provisioner` verified.
- A measured dependency set, with the heaviest pieces confined and questioned.

**Non-goals**

- The plugin gRPC contract — [0013](0013-forge-agent-plugin-sdk.md); first-party plugins —
  [0014](0014-forge-agent-plugins.md).
- Desired-state authoring, targeting, bundle signing, and plugin signature verification —
  [0011](0011-forge-provisioner.md).
- Inventory schemas and storage — [0009](0009-forge-inventory.md); CA and token format —
  [0006](0006-forge-identity.md).

## Proposal

### Responsibilities

| Process                               | Runs as                          | Does                                                        |
|---------------------------------------|----------------------------------|-------------------------------------------------------------|
| `forge-agent serve`                   | `forge-agent` user (+ `tss` group) | enrollment, key use, renewal, bundle pull, CRL fetch, reporting |
| `forge-agent executor`                | root, no network                 | bundle verification, OPA, Tengo, enforcement, plugins, privileged inventory |

The processes share only a spool directory. `serve` writes bundles and CRLs to `spool/inbox` (group
`forge-agent`, `0770`); the executor treats them as untrusted, and writes reports and inventory to
`spool/outbox`. Other commands: `enroll`, `status`, `version`, and `plugin verify <path>`.

### Interfaces

#### Enrollment

`forge-agent enroll --token-file <path>` (or the token on stdin, never as an argument, so it cannot leak
through the process list), built on `forge-sdk` `pkg/enroll` ([0003](0003-forge-sdk.md)):

1. `enroll.ParseToken` reads the environment ID and CA certificate hash offline.
2. **Key** — `keystore.auto` picks, in order: TPM 2.0 on Linux (`/dev/tpmrm0` via `go-tpm`, an ECC P-256
   signing key under the storage root key, stored as TPM-wrapped blobs), the Windows Platform Crypto
   Provider (`certtostore`), then `enroll.FileKeyStore` (`0600`, owned by `forge-agent`). The chosen
   backend is reported in inventory as `keyProtection` so policies can require hardware keys.
3. `enroll.Enroll` checks that the gateway's chain matches the token's CA hash and that its SPIFFE ID is
   `spiffe://<environment-id>/service/forge-gateway`. Only then does it send the CSR.
4. The certificate carries `spiffe://<environment-id>/agent/<agent-id>`. `Result` is written atomically
   to `identity/` with the environment `id`, `name`, `tier`, and `caBundle`.

`serve` refuses to start without an enrolled identity. If configuration also sets `environment.*`, the
values must match the recorded ones, which catches a host pointed at the wrong deployment. `enroll.Renewer`
renews at two-thirds of the lifetime with a **new key**, keeps the old key until the new certificate is
installed, and retries through the final third. An expired or revoked certificate stops `serve` until
the host is re-enrolled.

#### Directive pull and host evaluation

- **Pull** — `serve` long-polls `GET /provisioner/v1alpha1/directive-bundles/current` with
  `If-None-Match` and `waitSeconds=55`, with jittered backoff on errors (0011). It also fetches the
  environment CRL (path per 0006) so the offline executor can check the signer.
- **Verify** (executor, fail closed): the DSSE signature; a signer chain to the environment roots
  evaluated at `issuedAt`; a signer SPIFFE ID of `spiffe://<environment-id>/service/forge-provisioner`,
  not revoked by a CRL whose `nextUpdate` has not passed; `environmentId` and `agentId` equal to the
  recorded values; `generation` greater than the last accepted one (no rollback); and `now < notAfter`.
- **Validate** each resource against its JSON Schema, embedded from `forge-api-schema`.
- **Policy** — OPA evaluates, in order, the embedded agent baseline (for example, deny kinds disabled in
  local config), root-owned local policies in `/etc/forge-agent/policy.d/*.rego` that may only add
  denials, and the bundle's `host` policies. Same contract, capability filter, and 500 ms deadline as
  0011; an error or any `deny` rejects the whole bundle.
- **Scripts** — `host`-phase Tengo with the same allowlist and limits as 0011. The `forge` module exposes
  read-only host `facts()`; scripts compute values and never act.

#### Enforcement model

- **Handlers** — built-in kinds `File`, `Directory`, `Service` (systemd, Windows SCM, launchd), and
  `Package` (the OS package manager). Everything else is provided by plugins. Each handler implements
  `Observe`, `Diff`, `Apply`, and `Verify`.
- **Idempotence** — `Apply` runs only when `Diff` is non-empty; `Verify` re-observes and must be empty.
  Files are written to a temp file, `fsync`ed, and renamed.
- **Ordering** — resources run in `dependsOn` order; a failure skips its dependents and continues the
  others.
- **Cadence** — on every new bundle, and every `enforce.interval` (default 30 minutes) to correct drift.
  `mode: audit` observes and diffs only.
- **Reports** — per resource: status, a digest of observed state (never file content), timings, and a
  `reportId`, posted to `POST /provisioner/v1alpha1/enforcement-reports`.

#### Inventory

Collectors use `gopsutil` (host, CPU, memory, disks, interfaces), `/etc/os-release`, and the package
database. Privileged facts such as DMI serials come from the executor through the outbox. Full reports
every 6 hours and changed-digest deltas every 5 minutes go to `forge-inventory` (operation per 0009).

#### Plugin host

- **Store** — `/var/lib/forge-agent/plugins/sha256/<digest>/<name>`, root-owned, `0555`. go-plugin's
  `SecureConfig.Check` hashes `cmd.Path` and then execs that path
  ([client.go L662](https://github.com/hashicorp/go-plugin/blob/v1.8.0/client.go#L662),
  [L735](https://github.com/hashicorp/go-plugin/blob/v1.8.0/client.go#L735)), so a writable store would
  leave a swap window. Nothing but root can write this store.
- **Install** (executor): a downloaded plugin is hashed, and only a SHA-256 equal to a pin in the
  accepted, signature-verified bundle is moved into the store. `forge-provisioner` verified that digest
  against the publisher's Sigstore signature when the release was imported (0011); the agent does not
  verify Sigstore bundles.
- **Before every launch** (executor): the digest must still be pinned by the currently accepted bundle.
  Then go-plugin launches with
  `SecureConfig{Checksum: pin, Hash: sha256.New()}`, `AllowedProtocols: [ProtocolGRPC]`, `AutoMTLS: true`,
  and `SkipHostEnv: true`.
- **Environment** — the plugin's environment has `FORGE_PLUGIN_<NAME>_ENVIRONMENT_ID`, `_NAME`, and
  `_TIER`, and 0013's first RPC repeats them. The plugin refuses a different ID.
- **Privileges** — plugins run as the `forge-plugin` user by default (`SysProcAttr.Credential`). Root is
  granted only when local, root-owned config lists the plugin under `plugins.privileged`; a bundle cannot
  grant it.
- **Limits** — on Linux, each plugin starts directly in its own child cgroup (`SysProcAttr.UseCgroupFD`)
  under the executor's delegated subtree, with `memory.max`, `cpu.max`, and `pids.max`. On Windows, a Job
  Object (`CreateJobObject`, `JOBOBJECT_EXTENDED_LIMIT_INFORMATION`). On macOS, `setrlimit` only.
- **Supervision** — restart with exponential backoff; quarantine after 5 crashes in 10 minutes, reported
  as a condition.

### Dependencies

- **Forge** — `forge-sdk`, `forge-api-schema`, `forge-common`, `forge-agent-plugin-sdk`.
- **Starter** — cobra and `ants` from `go-cli-starter`. Its database packages (pgx, bun, goose) are
  removed: the agent keeps files, not a database.
- **New, measured** on 2026-09-15 in throwaway `linux/amd64` modules (`CGO_ENABLED=0`, stripped), counting
  modules in `go list -deps`:

| Module                                                                               | Version        | Linked | Binary   | Notes                                            |
|--------------------------------------------------------------------------------------|----------------|--------|----------|--------------------------------------------------|
| `hashicorp/go-plugin`                                                                | v1.8.0         | 14     | 10.8 MiB | gRPC, genproto, hclog, yamux, `fatih/color`      |
| `open-policy-agent/opa/v1/rego`                                                      | v1.20.2        | 26     | 22.0 MiB | jwx, logrus, gqlparser; see 0011                 |
| `sigstore/sigstore-go` (not linked)                                                  | v1.3.0         | 71     | 17.5 MiB | measured below; verification moved to 0011       |
| `d5/tengo/v2`                                                                        | pseudo-version | 1      | 3.4 MiB  | see 0011                                         |
| `google/go-tpm`                                                                      | v0.9.8         | 2      | 2.6 MiB  | + `x/sys`                                        |
| `shirou/gopsutil/v4`                                                                 | v4.26.8        | 4      | 3.3 MiB  | `go-ole` and `wmi` on Windows, `purego` on macOS |
| `google/certtostore` (Windows)                                                       | v1.0.7         | 6      | —        | `go-ole`, `google/deck`, `StackExchange/wmi`     |
| **Proposed agent set** — above, without certtostore and sigstore-go, with jsonschema | —              | **44** | 29.4 MiB | 165 in `go list -m all`                          |
| For comparison, with sigstore-go                                                     | —              | 102    | 33.3 MiB | 428 in `go list -m all`; 104 linked on Windows   |

#### Sigstore verifier measurements

Measured again on 2026-09-15 with Go 1.27.1: throwaway `linux/amd64` programs (`CGO_ENABLED=0`,
`-trimpath`) that import only the listed packages, counting the modules
[`go version -m`](https://pkg.go.dev/cmd/go#hdr-Print_Go_version) reports as compiled into the binary.

| Import set                                                     | Version  | Modules in the binary | `go list -m all` |
|----------------------------------------------------------------|----------|-----------------------|------------------|
| `sigstore-go/pkg/verify`                                       | v1.3.0   | 71                    | 367              |
| `pkg/verify`, `pkg/bundle`, `pkg/root`, and `pkg/tuf`          | v1.3.0   | 71                    | 367              |
| `sigstore-go/pkg/bundle`                                       | v1.3.0   | 71                    | 367              |
| `sigstore-go/pkg/root`                                         | v1.3.0   | 18                    | 209              |
| `sigstore/protobuf-specs` bundle types (`gen/pb-go/bundle/v1`) | v0.5.2   | 3                     | 128              |
| `sigstore/sigstore/pkg/signature`                              | v1.10.10 | 11                    | 69               |
| `transparency-dev/merkle/proof`                                | v0.0.2   | 1                     | 2                |
| `google/certificate-transparency-go/x509`                      | v1.3.3   | 2                     | 147              |

The 71 are sigstore-go and 70 others, from about 26 upstream projects: 23 `go-openapi` modules (its
`swag` package ships as many small modules), 7 `golang.org/x`, 6 `sigstore`, 4 gRPC, protobuf, and
genproto, and 4 OpenTelemetry. The import chains show that most come from service clients rather than
verification cryptography:

- `pkg/verify` → `pkg/tlog` → the generated Rekor v1 OpenAPI client → `go-openapi/runtime` →
  OpenTelemetry;
- `pkg/verify` → `rekor-tiles/v2` protobuf types → gRPC;
- `pkg/verify` → `sigstore/sigstore/pkg/signature` → `go-containerregistry` (image-name parsing);
- `pkg/verify` → `certificate-transparency-go/ctutil` → `loglist3` → `k8s.io/klog/v2`;
- `pkg/verify` → `pkg/root` → `pkg/tuf` → `go-tuf/v2`, even when the trusted root is loaded from a file.

**Decision: verify centrally.** A Forge-built verifier on the small building blocks above would link
about 4–6 modules (an estimate; that combination is not built), but it would re-implement
security-critical checks: the Fulcio certificate chain and identity, SCTs, and transparency-log
inclusion proofs and checkpoints. Instead, `forge-provisioner` verifies publisher signatures with
sigstore-go when a plugin release is imported ([0011](0011-forge-provisioner.md)), and the agent trusts
only SHA-256 pins in bundles it has verified with standard-library ECDSA. The trade-off is that agents
trust the provisioner's environment-issued signing certificate for plugin approval instead of checking
Sigstore themselves. OPA is now the largest addition; gRPC comes in with go-plugin regardless.

### Data & storage

Under `/var/lib/forge-agent` (`%ProgramData%\forge-agent` on Windows); every write is atomic (temp file,
`fsync`, rename):

| Path              | Owner and mode             | Contents                                                            |
|-------------------|----------------------------|---------------------------------------------------------------------|
| `identity/`       | `forge-agent`, `0700`      | certificate, chain, key or TPM blobs, environment record            |
| `spool/inbox/`    | root:`forge-agent`, `0770` | fetched bundles and CRLs, untrusted                                 |
| `spool/outbox/`   | root:`forge-agent`, `0750` | reports and inventory, capped at 50 MiB, oldest dropped and counted |
| `state/`          | root, `0700`               | last accepted generation and bundle, handler state                  |
| `plugins/sha256/` | root, `0555` files         | plugin executables whose SHA-256 matches a bundle pin               |

### Security

- **Privilege separation** — `serve` has no root and no write access to `state/` or `plugins/`. The
  executor's systemd unit sets `IPAddressDeny=any` and `RestrictAddressFamilies=AF_UNIX`
  ([systemd.resource-control](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html),
  [systemd.exec](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html)). It acts only
  on bundles signed by the provisioner, so compromising `serve` or the gateway does not yield root.
- **Hardening** — `serve` uses `NoNewPrivileges=yes`, `ProtectSystem=strict`, `ProtectHome=yes`, and
  `ReadWritePaths` limited to `identity/` and `spool/`.
- **Keys** stay on the host; TPM keys are non-exportable. Tokens are read from files or stdin and never
  logged.
- **Fail closed** on any verification, policy, or pin failure; the previous accepted bundle keeps running.
- **Fuzzing** covers DSSE and bundle decoding and the spool readers.

### Environment awareness

The four environment values come from enrollment ([CONVENTIONS.md](CONVENTIONS.md#environment)); logs,
reports, bundles, and plugins are all checked against the recorded ID.

| Behavior                                                 | `production` / `staging`                  | `test` / `development` |
|----------------------------------------------------------|-------------------------------------------|------------------------|
| File key store when no TPM                               | allowed, logged at `warn`                 | allowed                |
| Local plugin not pinned by a bundle (`unpinned-plugins`) | refused in `production` unless overridden | allowed, logged        |
| Local policy `print`, verbose plans                      | off                                       | on                     |

### Offline behavior

- The executor re-applies the last accepted bundle every `enforce.interval` until its `notAfter`, then
  switches to `audit` mode and raises a condition.
- A new generation is accepted only with a CRL whose `nextUpdate` has not passed; a stale CRL blocks new
  changes but not re-application of the accepted bundle.
- Reports queue in the outbox; `serve` renews in the final third of the certificate lifetime when it
  reconnects, and an expired certificate requires re-enrollment.

### Logging & telemetry

Through `forge-common` in both processes, with `forge.agent.id`, `forge.bundle.generation`,
`forge.resource.kind`, and `forge.plugin.name`. Metrics: `forge.agent.enforce.duration`,
`forge.agent.resources.drifted`, `forge.agent.plugin.restarts`, and `forge.agent.outbox.dropped`.
Telemetry export runs only from `serve`; the executor writes its metrics to the outbox.

### Configuration

Prefix `FORGE_AGENT_`; the gateway client uses `FORGE_AGENT_GATEWAY_*` per 0003.

| YAML                     | Variable                              | Default                               |
|--------------------------|---------------------------------------|---------------------------------------|
| `stateDirectory`         | `FORGE_AGENT_STATE_DIRECTORY`         | `/var/lib/forge-agent`                |
| `keystore.backend`       | `FORGE_AGENT_KEYSTORE_BACKEND`        | `auto` (`tpm`, `windows-pcp`, `file`) |
| `enforce.interval`       | `FORGE_AGENT_ENFORCE_INTERVAL`        | `30m`                                 |
| `enforce.disabledKinds`  | `FORGE_AGENT_ENFORCE_DISABLED_KINDS`  | empty                                 |
| `inventory.fullInterval` | `FORGE_AGENT_INVENTORY_FULL_INTERVAL` | `6h`                                  |
| `plugins.privileged`     | YAML only                             | empty                                 |
| `plugins.memoryMax`      | `FORGE_AGENT_PLUGINS_MEMORY_MAX`      | `256MiB`                              |
| `outbox.maxBytes`        | `FORGE_AGENT_OUTBOX_MAX_BYTES`        | `52428800`                            |

### Build, release & versioning

- **Platforms** — Linux `amd64` and `arm64` with systemd (tier 1); Windows `amd64` as two services, with
  the executor as LocalSystem and `serve` as a virtual service account (tier 2); macOS `arm64` with
  launchd (tier 2, file key store). Built with `CGO_ENABLED=0`.
- **Packaging** — deb, rpm, and apk through [nfpm](https://nfpm.goreleaser.com) v2.47.0 run with
  `go run`; an MSI for Windows and a pkg for macOS (tooling open). Packages include the units, users, and
  cosign-signed checksums plus the starter's signed SBOM.
- **Upgrades** — through the OS package manager, which a bundle may drive with a `Package` resource for
  `forge-agent`. The executor applies it last and restarts both units. An agent accepts the current and
  previous bundle `apiVersion`.

### Testing

- **Enrollment** against `sdktest` with a software TPM simulator where available (unverified choice).
- **Verification tables** — wrong signer SPIFFE ID, wrong environment, rollback, expired `notAfter`,
  stale CRL, and a plugin digest that no accepted bundle pins.
- **Handlers** — idempotence (a second apply is a no-op) in containers per distribution.
- **Sandbox and limits** — Tengo escapes, OPA timeouts, plugin memory and pid limits, and crash
  quarantine. Fuzzing, `-race`, and the module allowlist.

## Alternatives considered

- **Single root process** — simpler, but the TLS stack, HTTP client, and bundle parsing would run as
  root.
- **Fully unprivileged agent with sudo rules** — too coarse to express per-resource needs, and hard to
  audit.
- **Verifying Sigstore signatures on every agent** — sigstore-go adds about 59 modules to the agent (102
  instead of 44), and a Forge-built verifier would re-implement security-critical checks; see
  [Sigstore verifier measurements](#sigstore-verifier-measurements).
- **Keyed cosign signatures checked on agents with standard-library ECDSA** — no new modules, but it
  drops the keyless publisher identities that 0001 requires.
- **Standard-library-only inventory** — avoids gopsutil's 4 modules, but means per-OS code for Windows
  and macOS.
- **Direct NCrypt calls instead of certtostore** — avoids `go-ole` and `wmi`, but more Windows code to
  own.
- **A local database (bbolt or SQLite)** — unnecessary for a few small, atomically replaced files.

## Open questions

- **Plugin distribution** — how binaries reach the host: a gateway-served artifact route, or an OCI
  registry?
- **CRL delivery** — does `pkg/revocation` accept an offline CRL source, and which route serves the CRL
  on the agent ingress (0003, 0006, 0008)?
- **macOS keys** — Secure Enclave needs cgo; is a file key store acceptable there?
- **Windows packaging** tool for the MSI, and the service account model.
- **Agent ID** — assigned by `forge-identity` at enrollment (assumed) or derived from the key?

## References

- [0001 — Project Repositories](0001-project-repositories.md), [CONVENTIONS.md](CONVENTIONS.md),
  [0003](0003-forge-sdk.md), [0004](0004-forge-common.md), [0011](0011-forge-provisioner.md).
- [`hashicorp/go-plugin`](https://github.com/hashicorp/go-plugin) —
  [`SecureConfig`](https://pkg.go.dev/github.com/hashicorp/go-plugin#SecureConfig), `SkipHostEnv`,
  `AutoMTLS`, `UnixSocketConfig`; source read at v1.8.0.
- [sigstore-go](https://github.com/sigstore/sigstore-go) —
  [`verify`](https://pkg.go.dev/github.com/sigstore/sigstore-go/pkg/verify) and
  [`root`](https://pkg.go.dev/github.com/sigstore/sigstore-go/pkg/root) packages — measured, not linked
  by the agent.
- [sigstore/protobuf-specs](https://github.com/sigstore/protobuf-specs),
  [transparency-dev/merkle](https://github.com/transparency-dev/merkle),
  [sigstore/sigstore](https://github.com/sigstore/sigstore), and
  [certificate-transparency-go](https://github.com/google/certificate-transparency-go) — measured building
  blocks.
- [`go version -m`](https://pkg.go.dev/cmd/go#hdr-Print_Go_version) — module list compiled into a binary.
- [Sigstore cosign](https://docs.sigstore.dev/cosign/signing/overview/).
- [OPA `v1/rego`](https://pkg.go.dev/github.com/open-policy-agent/opa/v1/rego) and
  [Tengo](https://github.com/d5/tengo).
- [go-tpm](https://github.com/google/go-tpm), [certtostore](https://github.com/google/certtostore), and
  [gopsutil](https://github.com/shirou/gopsutil).
- [DSSE](https://github.com/secure-systems-lab/dsse/blob/master/protocol.md).
- [`syscall.SysProcAttr` (Linux)](https://pkg.go.dev/syscall?GOOS=linux#SysProcAttr) — `Credential`,
  `UseCgroupFD`, `CgroupFD`; [cgroup v2](https://docs.kernel.org/admin-guide/cgroup-v2.html).
- [`golang.org/x/sys/windows`](https://pkg.go.dev/golang.org/x/sys/windows) — `CreateJobObject`,
  `JOBOBJECT_EXTENDED_LIMIT_INFORMATION`.
- [systemd.exec](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html) and
  [systemd.resource-control](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html).
- [nfpm](https://nfpm.goreleaser.com) — deb, rpm, and apk packaging.
- [`kubeadm join`](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/).
- [go-cli-starter](https://github.com/servercurio/go-cli-starter) — cobra, `ants`, `serve` daemon.
