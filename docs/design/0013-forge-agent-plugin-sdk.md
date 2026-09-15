# 0013 — forge-agent-plugin-sdk

- **Status:** Draft
- **Owner:** Nathan Klick
- **Date:** 2026-09-15
- **Summary:** `forge-agent-plugin-sdk` defines the versioned gRPC contract between `forge-agent` and
  its plugins over `hashicorp/go-plugin`. The agent gets launch helpers that pin each binary's SHA-256,
  use automatic mutual TLS, and start plugins with a clean environment. Plugins get a `serve` package
  that enforces the environment check and capability grants, plus a test harness. The only third-party
  code is the go-plugin and gRPC stack, measured at 14 linked modules.

> An initial draft with concrete proposals, bounded by the
> [Resolved decisions](0001-project-repositories.md#resolved-decisions) in 0001. Conventions other
> repositories depend on are summarized in [CONVENTIONS.md](CONVENTIONS.md).

## Context & goals

In 0001, plugins are separate processes that `forge-agent` launches over gRPC with
`hashicorp/go-plugin`. `forge-provisioner` verifies plugin release signatures at import, and before
every launch the agent pins the binary's SHA-256, taken from its signed directive bundle, through
`SecureConfig`. The agent passes its environment ID, and a plugin refuses an agent from
another environment
([Agent plugin ecosystem](0001-project-repositories.md#agent-plugin-ecosystem),
[Environment identity](0001-project-repositories.md#environment-identity)). This repository is "the
stable contract plugins build against", and [CONVENTIONS.md](CONVENTIONS.md) makes it the only home of
Forge's gRPC contract.

**Goals**

- A versioned wire contract, so the agent and plugins release independently.
- One implementation of the go-plugin setup: hash pinning, mutual TLS, a clean environment, and
  protocol negotiation.
- The environment check and capability gates enforced once, for every plugin.
- A small footprint and a harness authors can run without a real agent.

**Non-goals**

- Signature verification, installation, OS sandboxing, and restarts — [0012](0012-forge-agent.md).
- Desired-state kind schemas — [0002](0002-forge-api-schema.md).
- First-party plugins ([0014](0014-forge-agent-plugins.md)) and the author template
  ([0015](0015-forge-plugin-starter.md)).
- Agentless device drivers, which `forge-provisioner` enforces remotely
  ([Desired-state model](0001-project-repositories.md#desired-state-model--one-authority-two-enforcement-paths)).

## Proposal

### Responsibilities

- **Contract** — protobuf definitions, generated Go code, handshake values, and protocol versions.
- **Manifest** — plugin names, capabilities, and declared privileges, with validation.
- **Host and plugin sides** — `host` launches and calls plugins; `serve` is the plugin boilerplate
  with its interceptors.
- **Trace context and testing** — W3C propagation, the harnesses, and a conformance suite.

### Interfaces

#### Package layout

```
forge-agent-plugin-sdk/
├── proto/forge/agent/plugin/v1alpha1/   # plugin.proto, facts.proto, resource.proto
├── buf.yaml, buf.gen.yaml
├── pkg/
│   ├── plugin/v1alpha1/                 # package pluginv1alpha1 (generated, committed)
│   ├── handshake/                       # Config, SupportedProtocols
│   ├── manifest/                        # Manifest, Capability, Privileges, ValidateName
│   ├── serve/                           # Main, Options, FactsCollector, ResourceHandler, Granted
│   ├── host/                            # Launch, Config, Plugin, ErrEnvironmentMismatch
│   ├── tracecontext/                    # Carrier, UnaryClient, UnaryServer
│   └── plugintest/                      # InProcess, Launch, Conformance
└── Taskfile.yaml
```

#### Wire contract

Proposed: one protobuf package per protocol version, with services split by capability, using buf's
`STANDARD` naming (`<Rpc>Request`/`<Rpc>Response`, services suffixed `Service`).

```proto
syntax = "proto3";
package forge.agent.plugin.v1alpha1;
option go_package = "github.com/servercurio/forge-agent-plugin-sdk/pkg/plugin/v1alpha1;pluginv1alpha1";

service PluginService {
  rpc GetManifest(GetManifestRequest) returns (GetManifestResponse); // allowed before Init
  rpc Init(InitRequest) returns (InitResponse);                      // environment check, grants
  rpc Check(CheckRequest) returns (CheckResponse);                   // liveness
}
service FactsService { rpc CollectFacts(CollectFactsRequest) returns (CollectFactsResponse); }
service ResourceService {
  rpc ValidateResource(ValidateResourceRequest) returns (ValidateResourceResponse);
  rpc PlanResource(PlanResourceRequest) returns (PlanResourceResponse);    // read-only diff
  rpc ApplyResource(ApplyResourceRequest) returns (ApplyResourceResponse); // idempotent
}
message Environment { string name = 1; string tier = 2; string id = 3; }
message InitRequest { Environment environment = 1; string agent_id = 2; Grant grant = 3; }
message Grant { repeated string capabilities = 1; Privileges privileges = 2; }
message Resource { string api_version = 1; string kind = 2; string name = 3; bytes spec_json = 4; }
message Fact { string name = 1; bytes value_json = 2; } // name: <plugin>.<snake_case>
```

- **JSON payloads** — specs and fact values keep their 0002 JSON Schema form, with no protobuf copies
  of kinds. The agent schema-validates and OPA-checks specs before sending them (0001), and validates
  facts before reporting them.
- **Errors** — gRPC status codes with `google.rpc.ErrorInfo`: `reason` is a stable lower_snake_case
  code, as in CONVENTIONS' problem `code`, and `domain` is the plugin name. `errdetails` comes from
  `genproto/googleapis/rpc`, which gRPC already links.
- **Size** — gRPC's default 4 MiB receive limit. The agent passes content inline; plugins fetch nothing.

#### Handshake and protocol versions

| Protocol | Protobuf package              | Stage | Status        |
|----------|-------------------------------|-------|---------------|
| `1`      | `forge.agent.plugin.v1alpha1` | alpha | first release |

- **Cookie** — `FORGE_AGENT_PLUGIN` with a fixed random value. go-plugin calls the cookie "not a
  security measure, just a UX feature" (`server.go`). A binary run by hand prints help and exits.
- **Negotiation** — `host` offers the agent's protocols in `VersionedPlugins`. go-plugin passes them in
  `PLUGIN_PROTOCOL_VERSIONS`, and `serve` answers with one they share.
- **Compatibility** — additive fields and RPCs stay in the package. A host that calls a missing RPC
  gets `UNIMPLEMENTED` and treats the feature as absent. Any wire- or generated-code-breaking change
  creates a new package and protocol number, alpha included (see Alternatives).
- **Window** — the agent supports protocols N and N-1 and warns on N-1. SDK releases serve every
  protocol they implement.

#### Capability model

Each plugin embeds a `manifest.yaml`. `GetManifest` returns it, and each release publishes it
([0014](0014-forge-agent-plugins.md)):

```yaml
name: packages                  # lowercase DNS label; binary forge-plugin-packages
version: 0.4.0
protocolVersions: [1]
capabilities: [resource:forge.servercurio.com/v1alpha1/Package]
privileges:
  runAsRoot: true
  execPaths: [/usr/bin/apt-get, /usr/bin/dpkg-query, /usr/bin/dnf, /usr/bin/rpm]
  network: true                 # package managers download from mirrors
platforms: [linux/amd64, linux/arm64]
```

| Capability                          | Service           | Meaning                            |
|-------------------------------------|-------------------|------------------------------------|
| `facts`                             | `FactsService`    | read-only inventory facts          |
| `resource:<group>/<version>/<Kind>` | `ResourceService` | validate, plan, and apply one kind |

1. **Declare** — after verifying and launching the plugin (0012), the agent reads its manifest. The
   manifest is trusted only as far as the verified publisher.
2. **Grant** — the grant is the intersection of the manifest and the operator's policy for the
   plugin, never more than the manifest declares.
3. **Gate** — `Init` carries the grant. `serve` rejects RPCs outside it with `PERMISSION_DENIED`
   (`capability_not_granted`), and `serve.Granted(ctx)` exposes the privilege grant to handlers.
4. **Enforce** — in-plugin checks are defense in depth. The boundary is the OS sandbox the agent builds
   from the grant (dedicated user, no network, path limits) through go-plugin's `RunnerFunc` (0012).

#### Environment check

1. **Startup** — `serve.Main` validates the plugin's own `environment` block through `forge-common`.
   `name`, `tier`, and `id` are required, and the plugin exits before the handshake if any is missing.
2. **Gate** — until `Init` succeeds, every RPC except `GetManifest` and `Check` returns
   `FAILED_PRECONDITION` (`not_initialized`). `Init` is accepted once.
3. **Compare** — `Init` compares the agent's name, tier, and ID with the plugin's. On a mismatch it
   returns `PERMISSION_DENIED` (`environment_mismatch`), logs both IDs, and exits with code 78. `host`
   returns `ErrEnvironmentMismatch`, so the agent does not restart the plugin in a loop.
4. **Independence** — the agent's values come from enrollment and the plugin's from its own
   configuration file. `host.Launch` refuses `Env` entries that set `<PREFIX>_ENVIRONMENT_*`.

#### Plugin side (`serve`)

```go
type ResourceHandler interface {
    Validate(ctx context.Context, r serve.Resource) error
    Plan(ctx context.Context, r serve.Resource) (serve.Plan, error)                  // no host changes
    Apply(ctx context.Context, r serve.Resource, p serve.Plan) (serve.Result, error) // idempotent
}

func main() {
    os.Exit(serve.Main(serve.Options{
        Manifest:  manifestYAML,     // //go:embed manifest.yaml
        Version:   version.Number(),
        Configure: loadConfig,       // defaults → file → FORGE_PLUGIN_<NAME>_* → *environment.Config
        Logger:    initLogging,      // receives the original stderr (see Logging & telemetry)
        Facts:     example.Facts{},  // FactsCollector: CollectFacts(ctx) ([]serve.Fact, error)
        Resources: map[string]serve.ResourceHandler{"plugins.example.com/v1alpha1/Marker": marker},
    }))
}
```

`serve.Main` checks for the magic cookie and a handler for every declared capability, and keeps the
original stderr for logging. It installs interceptors for panic recovery (`INTERNAL`, `plugin_panic`,
no stack in the reply), the `Init` and capability gates, trace extraction, and message size, then calls
`plugin.Serve` with gRPC only.

#### Host side (`host`)

```go
p, err := host.Launch(ctx, host.Config{
    Name:   "packages",
    Path:   "/var/lib/forge-agent/plugins/packages/<sha256>/forge-plugin-packages",
    SHA256: digest,                             // from the verified install record (0012)
    Env:    map[string]string{"FORGE_PLUGIN_PACKAGES_LOG_LEVEL": "info"},
    Stderr: logSink,                            // receives the plugin's JSON log lines
    ClientInterceptors: []grpc.UnaryClientInterceptor{tracecontext.UnaryClient(inject)},
})
if err != nil { return err }
defer p.Close()
m, err := p.Manifest(ctx)
err = p.Init(ctx, host.InitRequest{Environment: env, AgentID: agentID, Grant: policy.Grant(m)})
plan, err := p.Resources().PlanResource(ctx, &pluginv1alpha1.PlanResourceRequest{Resource: r})
```

| go-plugin setting  | `host.Launch` value               | Reason                                         |
|--------------------|-----------------------------------|------------------------------------------------|
| `AllowedProtocols` | gRPC only                         | no `net/rpc` gob decoding                      |
| `SecureConfig`     | caller's SHA-256, required        | 0001; constant-time compare (`client.go`)      |
| `AutoMTLS`         | `true`                            | other local processes cannot use the socket    |
| `SkipHostEnv`      | `true`                            | agent credentials and proxies not inherited    |
| `UnixSocketConfig` | agent-owned `0700` directory      | socket unreachable by other users              |
| `Logger`           | `hclog.NewNullLogger()`           | the SDK does not log; lines go to `Stderr`     |
| `RunnerFunc`       | optional, from the caller         | sandboxing belongs to 0012                     |

`host` requires an absolute path to a regular file. On Unix, the file and its parent directories must
not be writable by group or others.

#### Trace context

`tracecontext.Carrier` implements `Get`, `Set`, and `Keys` over gRPC metadata for `traceparent` and
`tracestate` only (no baggage). The interceptors take inject and extract functions, such as closures
over `otel.GetTextMapPropagator()`, so the SDK never imports OpenTelemetry. The agent records a client
span per plugin RPC. The plugin extracts the remote context, so `forge-common`'s `TraceHook` stamps
`trace_id` on its logs without the plugin exporting spans.

### Dependencies

- **Root module** — `hashicorp/go-plugin` v1.8.0 (latest), `google.golang.org/grpc` v1.83.2,
  `google.golang.org/protobuf` v1.36.12, and `forge-common` (`environment` only).
- **Measured 2026-09-15** — throwaway gRPC plugin and host programs (`Serve`, `NewClient` with
  `SecureConfig`, `AutoMTLS`, `SkipHostEnv`) each link the same **14 modules** on linux and windows:
  `go-plugin`, `go-hclog`, `yamux`, `oklog/run`, `fatih/color`, `mattn/go-colorable`,
  `mattn/go-isatty`, `golang/protobuf`, `grpc`, `protobuf`, `genproto/googleapis/rpc`, `x/net`,
  `x/sys`, and `x/text`. `go list -m all` reports 61. `jhump/protoreflect` is required but not linked.
  A stripped linux/amd64 plugin binary was 12.6 MB.
- **gRPC floor** — go-plugin v1.8.0 requires gRPC v1.61.0 and protobuf v1.36.6. The SDK requires
  current versions, so minimal version selection gives every plugin current security fixes.
- **Not used** — `sigstore-go` v1.3.0: its verifier links 71 modules, including gRPC, OpenTelemetry,
  and `go-openapi`, and `go list -m all` reports 367
  ([0012](0012-forge-agent.md#sigstore-verifier-measurements)). Verification runs in
  `forge-provisioner` ([0011](0011-forge-provisioner.md)), not in plugins or the agent.
- **Tools** (not in `go.mod`) — buf v1.73.0, `protoc-gen-go` v1.36.12, and `protoc-gen-go-grpc` v1.6.2.
  All three were verified to run through `go run <module>@<version>`. `task tools` installs them into
  `.tools/bin` for `buf.gen.yaml`'s `local:` plugins.
- **Licenses** — go-plugin and yamux are MPL-2.0: binary distributors must tell recipients where the
  source is available (MPL FAQ). The rest are Apache-2.0, BSD-3-Clause, or MIT.

### Data & storage

None; only in-memory grant and environment state per plugin process.

### Security

- **Launch chain** — publisher signature verification at import (0011), a provisioner-signed pin and a
  digest-checked install (0012), then the SHA-256 in `SecureConfig` on
  every launch, mutual TLS on the socket, the environment check, and capability gates.
- **`SecureConfig` gap** — go-plugin hashes `Path`, then executes `Path`, so a writer could swap the
  file in between. 0012's root-owned, content-addressed install directories and the writable-path
  refusal close that window in practice.
- **Clean environment** — plugins never receive the agent's variables, certificate, key, or token, and
  have no path to `forge-gateway`.
- **Untrusted replies** — the agent validates and size-caps facts and never shows plugin error details
  to operators verbatim.
- **Least privilege** — manifests default to no root, exec, writes, or network. Fuzzing covers manifest,
  capability, and fact-name parsing.

### Environment awareness

Plugins require name, tier, and ID at startup and refuse agents from another environment (0001). Tier
behavior goes through `forge-common`'s `Hardened()` and `AllowLastResort`. `host.Launch` has no insecure
mode, and only `plugintest` uses `development` fixtures.

### Logging & telemetry

- **Stderr, not stdout** — go-plugin uses stdout for its handshake line, and `plugin.Serve` then
  redirects `os.Stdout` and `os.Stderr` to `SyncStdout` and `SyncStderr` streams (`server.go`). `serve`
  gives the original stderr to `forge-common`'s logger. go-plugin reads it line by line into
  `host.Config.Stderr`, splitting lines longer than its 64 KiB buffer.
- **Re-emitted by the agent** — the agent parses each JSON line, adds `forge.plugin.name` and
  `forge.plugin.version`, rate-limits, and writes to its stdout. Non-JSON lines, such as panics, become
  `warn` events.
- **Fields and telemetry** — `service.name` is `forge-plugin-<name>`. Plugins export no telemetry in
  `v1alpha1`; the agent records `forge.agent.plugin.rpc.duration` (0012).

### Configuration

A plugin's prefix is `FORGE_PLUGIN_<NAME>`: the name upper-cased, hyphens as underscores. The SDK's
config mounts under the child keys `environment` and `rpc`:

| YAML                                 | Variable                                    | Default             |
|--------------------------------------|---------------------------------------------|---------------------|
| `environment.name` / `.tier` / `.id` | `FORGE_PLUGIN_<NAME>_ENVIRONMENT_NAME` / …  | none — required     |
| `rpc.maxMessageBytes`                | `FORGE_PLUGIN_<NAME>_RPC_MAX_MESSAGE_BYTES` | `4194304`           |
| `rpc.initTimeout`                    | `FORGE_PLUGIN_<NAME>_RPC_INIT_TIMEOUT`      | `30s`, then exit    |

### Build, release & versioning

- **Bootstrap** from `go-library-starter`, removing its runtime packages as
  [0002](0002-forge-api-schema.md) does.
- **Tasks** — `tools`, `generate`, `check:drift`, `test`, `deps:check`, `proto:lint` (`buf lint`,
  `STANDARD`), and `proto:breaking` (`buf breaking` against the last tag with `FILE`, buf's default and
  strictest category, on every package).
- **CI** — the starters' 200, 300, and 100 workflows plus these checks.
- **Versioning** — `v0.x` per CONVENTIONS, independent of protocol numbers. `SupportedProtocols` and
  each release note list the protocols served. Dropping one is breaking and waits for the N-1 window.

### Testing

- **`plugintest.InProcess`** — go-plugin's `TestPluginGRPCConn`, using only the standard `testing`
  package.
- **`plugintest.Launch`** — a fake agent: it pins the binary's hash, enables mutual TLS and a clean
  environment, sends a `development` environment, and forwards stderr to `t.Log`.
- **`plugintest.Conformance`** — a valid manifest with handlers; refusal before `Init`, on environment
  mismatch (with exit), and for ungranted capabilities; panics as `INTERNAL`; JSON log lines; and no
  changes from `Plan` after `Apply` for author-supplied samples.
- **SDK tests** — negotiation (host `{1,2}` against plugin `{1}`), a checksum mismatch, the
  writable-path refusal, fuzzers, `-race`, and the allowlist.

## Alternatives considered

- **go-plugin `net/rpc`** — gob encoding with no schema; 0001 chose gRPC.
- **`protoc`** — a C++ binary that `go run` cannot install, with no lint or breaking checks.
- **Buf Schema Registry remote plugins** — need network access and an account to generate.
- **In-process WebAssembly plugins** — strong sandboxing, but contradicts 0001's separate processes.
- **Plugin configuration sent in `Init`** — the environment would then come from the agent being
  checked.
- **Certificate-backed environment proof** — the agent's X.509-SVID signs a nonce from the plugin. It
  is stronger, but the parent process already controls the binary and its environment, so the main
  risk is misconfiguration, which an ID comparison catches. Revisit if plugins hold environment secrets.
- **Signature verification in `host`** — `sigstore-go` would add 71 linked modules for every plugin
  author and the agent; `forge-provisioner` verifies at import instead.
- **Logs over a `GRPCBroker` callback** — structured, but crash output is lost and a second connection
  is needed.
- **`otelgrpc` interceptors** — would import OpenTelemetry into the SDK.
- **Deviations from [CONVENTIONS.md](CONVENTIONS.md)**:
  - Plugins log to stderr, not stdout (*Logging and telemetry*).
  - Alpha protobuf packages never break in place (*API versioning*).
  - go-plugin's auto mutual TLS uses ephemeral ECDSA P-521 certificates (`mtls.go`), not P-256
    (*Security and identity*). They are not Forge certificates and carry no SPIFFE ID.
  - `id` is required without Forge mutual TLS (*Environment*).

## Open questions

- **Plugin environment source** — CONVENTIONS says plugins "receive" the environment from the agent,
  but 0001 says they check it against their own configuration. Proposed: a root-owned
  `/etc/forge/plugins/<name>.yaml` written by the host installer. Who writes it?
- **Log writer** — 0004's `logging.Initialize` needs an option to write to stderr.
- **Plugin spans** — no export (proposed), forwarding through the agent, or direct export with a network
  grant?
- **Third-party kinds** — which groups can they use, and how do their schemas reach the agent and
  [0011](0011-forge-provisioner.md)?
- **Long operations** — unary `ApplyResource` with a deadline (proposed), or streamed progress?
- **Scope** — `GRPCBroker` host callbacks (secrets, content), and host-attached device drivers?
- **Hardening** — execute from a verified file descriptor to close the `SecureConfig` gap? Windows ACL
  checks?

## References

- [0001](0001-project-repositories.md), [CONVENTIONS.md](CONVENTIONS.md),
  [0002](0002-forge-api-schema.md), [0003](0003-forge-sdk.md), [0004](0004-forge-common.md).
- [`hashicorp/go-plugin` v1.8.0](https://github.com/hashicorp/go-plugin/tree/v1.8.0) (MPL-2.0):
  - `client.go` — `SecureConfig`, `SkipHostEnv`, `AutoMTLS`, `RunnerFunc`, `logStderr`, 64 KiB buffer;
  - `server.go` — cookie comment, `PLUGIN_PROTOCOL_VERSIONS`, stdio redirection, Windows TCP listener;
  - `mtls.go`, `testing.go`, and [`go.mod`](https://github.com/hashicorp/go-plugin/blob/v1.8.0/go.mod);
  - [package docs](https://pkg.go.dev/github.com/hashicorp/go-plugin).
- [grpc-go](https://github.com/grpc/grpc-go) — `server.go` 4 MiB default;
  [`errdetails`](https://pkg.go.dev/google.golang.org/genproto/googleapis/rpc/errdetails).
- [Buf lint rules](https://buf.build/docs/lint/rules/) and
  [breaking rules](https://buf.build/docs/breaking/rules/) — `FILE`, `PACKAGE`, `WIRE_JSON`, `WIRE`.
- [protobuf-go](https://github.com/protocolbuffers/protobuf-go);
  [`protoc-gen-go-grpc`](https://pkg.go.dev/google.golang.org/grpc/cmd/protoc-gen-go-grpc).
- [sigstore-go](https://github.com/sigstore/sigstore-go) — measured verifier footprint.
- [MPL 2.0 FAQ](https://www.mozilla.org/en-US/MPL/2.0/FAQ/);
  [W3C Trace Context](https://www.w3.org/TR/trace-context/).
