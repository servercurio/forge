# 0004 — forge-common

- **Status:** Draft
- **Owner:** Nathan Klick
- **Date:** 2026-09-15
- **Summary:** `forge-common` gives every Forge Go executable the same logging, environment handling, and
  telemetry. It provides a zerolog wrapper with trace correlation and service and environment fields, an
  `environment` package for the four tiers, and OpenTelemetry setup whose Forge-built OTLP/HTTP exporter
  sends traces and metrics without linking gRPC.

> An initial draft with concrete proposals, bounded by the
> [Resolved decisions](0001-project-repositories.md#resolved-decisions) in 0001. Conventions other
> repositories depend on are summarized in [CONVENTIONS.md](CONVENTIONS.md).

## Context & goals

0001 settles the stack in
[Logging and telemetry](0001-project-repositories.md#logging-and-telemetry-forge-common). The library wraps
the starters' zerolog `logging` package and correlates logs with traces through a zerolog hook. Traces and
metrics use the OpenTelemetry API and SDK, exported by a permanent Forge-built OTLP/HTTP exporter on
`go.opentelemetry.io/proto/slim/otlp`. Every log event and telemetry resource carries the environment
name ([Environment awareness](0001-project-repositories.md#environment-awareness)). During bootstrap,
every Go repository replaces its starter's logging and telemetry with `forge-common`, while config
loading and middleware stay in the starters
([Bootstrapping](0001-project-repositories.md#bootstrapping-a-repository-from-a-starter)).

The starters' logging differs by template:

- `go-library-starter` has `logging.Default`, `Initialize`, and `<PREFIX>_LOG_*`.
- `go-echo-starter` and `go-cli-starter` have a `Daemon` logger under `<PREFIX>_DAEMON_LOG_*`.
- `go-echo-starter` adds an `Access` logger under `<PREFIX>_HTTP_ACCESS_LOG_*`, with camelCase fields
  such as `remoteIp`.

**Goals**

- Replace the starters' logging with minimal call-site changes.
- Tag every log event and telemetry resource with the service and its environment.
- Export traces and metrics over OTLP/HTTP without gRPC, from a measured dependency set enforced in CI.
- Implement tier parsing, validation, hardened defaults, and last-resort gating once.

**Non-goals**

- Config file loading and web-framework middleware, which stay in the starters (0001).
- Exporting logs over OTLP; logs stay on stdout (see Open questions).
- Certificates, SPIFFE IDs, and TLS configuration — [0003](0003-forge-sdk.md).
- Collector deployment and telemetry backends — [0005](0005-forge-infrastructure.md).

## Proposal

### Responsibilities

- **`logging`** — zerolog setup, the default and access loggers, the trace hook, and service and
  environment fields.
- **`environment`** — environment config, tier rules, and the last-resort gate. It lives here because
  logs and telemetry need the environment and 0001 requires one behavior everywhere, though that goes
  beyond "logging and telemetry" (see Open questions).
- **`service`** — service name, version, and instance ID, shared by logs and telemetry.
- **`telemetry`** — provider setup, W3C propagation, HTTP wrappers, and the OTLP/HTTP exporter.

### Interfaces

#### Package layout

```
forge-common/
├── pkg/
│   ├── environment/          # Tier, Config, Validate, Hardened, AllowLastResort
│   ├── service/              # Info{Name, Version, InstanceID}, NewInstanceID
│   ├── logging/              # Config, LoggerConfig, Initialize, Default, Access, TraceHook
│   ├── telemetry/            # Config, Setup, WrapTransport, WrapHandler
│   └── telemetry/otlphttp/   # TraceExporter, MetricExporter
├── internal/
│   ├── envvar/               # os.LookupEnv helpers using the starters' AddPrefix rules
│   └── transform/            # SDK data to OTLP protobuf, ported from opentelemetry-go
├── parity/                   # nested module: compares transform output with the official exporter
└── deps.allow                # module allowlist enforced in CI
```

#### Wiring in an executable

```go
// After the starter's Configure(): defaults → file → FORGE_INVENTORY_* → flags.
if err := cfg.Environment.Validate(); err != nil { return err } // name and known tier required
svc := service.Info{Name: "forge-inventory", Version: version.Number(), InstanceID: service.NewInstanceID()}
logging.Initialize(cfg.Logging, cfg.Environment, svc)

shutdown, err := telemetry.Setup(ctx, cfg.Telemetry, cfg.Environment, svc,
    telemetry.WithClientTLS(certs.ClientTLSConfig)) // e.g. built with forge-sdk's tlsconfig
if err != nil { return err }
defer shutdown(context.Background())

logging.Default.Info().Ctx(ctx).Str("forge.tenant.id", tenantID).Msg("endpoint registered")
```

The starters build a logger from environment variables before config is loaded. `Initialize` may
therefore run twice: first without environment fields, then again once the environment validates.

#### `environment`

```go
type Tier string // "production", "staging", "test", "development"

type Config struct {
    Name      string   `yaml:"name" json:"name"`
    Tier      Tier     `yaml:"tier" json:"tier"`
    ID        string   `yaml:"id" json:"id"`
    CABundle  string   `yaml:"caBundle" json:"caBundle"`
    Overrides []string `yaml:"overrides" json:"overrides"`
}

func (c *Config) FromEnv(prefix string)  // <prefix>_ENVIRONMENT_NAME, _TIER, _ID, _CA_BUNDLE, _OVERRIDES
func (c *Config) Validate() error        // name is a DNS label; tier is one of four; ID syntax if set
func (c *Config) RequireIdentity() error // ID and CABundle present, for mutual-TLS components
func (c *Config) Hardened() bool         // production or staging
func (c *Config) AllowLastResort(feature string, log *zerolog.Logger) error
```

- **Required values** — `Validate` has no defaults: a missing name or tier, or an unknown tier, fails
  startup.
- **Trust domain** — `RequireIdentity` checks only that the values are present. Matching the ID against
  the roots' trust domain happens in `forge-sdk`'s `tlsconfig`, so SPIFFE parsing exists once.
- **Last-resort gate** — `AllowLastResort` returns an error in `production` unless `feature` is in
  `Overrides`. Every use in `production` is logged at `warn`, and at `info` in other tiers, with
  `forge.override.feature`. Feature names are kebab-case and owned by their documents, such as
  `kek-sealed-ca-store` in [0006](0006-forge-identity.md).

#### `logging`

- **Config** — keeps the starters' `LoggerConfig` (`enabled`, `level`, `prettyPrint`, `includeCaller`)
  inside `Config{Default, Access}`, read from `<PREFIX>_LOG_*` and `<PREFIX>_ACCESS_LOG_*`.
- **Output** — JSON lines on stdout at level `info` by default. `prettyPrint` defaults to true only in
  `development`, and enabling it elsewhere logs a warning.
- **Timestamps** — `time` is written in RFC 3339 with nanoseconds, in UTC, using
  `func() time.Time { return time.Now().UTC() }`. All three starters set
  `zerolog.TimestampFunc = time.Now().UTC`, which is a method value: it captures a single instant, so
  every event carries the time of `Initialize`. A test run on 2026-09-15 confirmed this, and the fix
  should also go upstream to the starters.
- **Fields** — zerolog's field-name globals are pinned to `time`, `level`, `message`, and `error`. Every
  event also carries `service.name`, `service.version`, `service.instance.id`,
  `deployment.environment.name`, `forge.environment.tier`, and `forge.environment.id` when set.
- **Trace correlation** — `TraceHook` implements [`zerolog.Hook`](https://pkg.go.dev/github.com/rs/zerolog#Hook).
  For events logged with `Event.Ctx(ctx)`, it reads the span context from `Event.GetCtx()` and adds
  `trace_id`, `span_id`, and `trace_flags` in lowercase hex, following
  [trace context in non-OTLP logs](https://opentelemetry.io/docs/specs/otel/compatibility/logging_trace_context/).
  It imports only `go.opentelemetry.io/otel/trace`.

Moving from the starters, including OpenTelemetry
[HTTP attribute](https://opentelemetry.io/docs/specs/semconv/registry/attributes/http/) names for access
logs:

| Starter                                              | `forge-common`                                     |
|------------------------------------------------------|----------------------------------------------------|
| `logging.Daemon`                                     | `logging.Default`                                  |
| `<PREFIX>_DAEMON_LOG_*`, `<PREFIX>_HTTP_ACCESS_LOG_*` | `<PREFIX>_LOG_*`, `<PREFIX>_ACCESS_LOG_*`         |
| `method`, `status`                                   | `http.request.method`, `http.response.status_code` |
| `uriPath`, `routePath`, `remoteIp`                   | `url.path`, `http.route`, `client.address`         |
| Unix-second `time`                                   | RFC 3339 UTC `time`                                |

#### `telemetry`

- **`Setup`** builds a resource with the same `service.*`, `deployment.environment.name`, and
  `forge.environment.*` attributes. It then creates a `TracerProvider` with a batch span processor and
  `ParentBased(TraceIDRatioBased(ratio))` sampling, a `MeterProvider` with a periodic reader, and the
  global W3C `propagation.TraceContext` propagator. OpenTelemetry errors go to `logging.Default`,
  rate-limited. When disabled, `Setup` installs no-op providers but keeps the propagator.
- **Standard variables** — `OTEL_*` variables are not read; configuration has one path.
- **Wrappers** — `WrapTransport(http.RoundTripper)` injects `traceparent` and records client spans, for
  `forge-sdk`'s `WithTransportWrapper`. `WrapHandler(http.Handler, ...Option)` extracts the incoming
  context, records server spans, and records the `http.server.request.duration` histogram. Services
  adapt it to Echo in their own middleware.
- **Incoming traces** — `WithTrustIncoming(false)`, the default for public ingress, starts a new trace
  linked to the caller's span instead of adopting it. Internal mutual-TLS listeners set it to true.
- **Instrumentation** — Forge code imports only the OpenTelemetry API (`otel`, `otel/trace`,
  `otel/metric`), never the SDK or the exporters.

#### `telemetry/otlphttp`

- **Interfaces** — `TraceExporter` implements
  [`sdktrace.SpanExporter`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#SpanExporter).
  `MetricExporter` implements
  [`sdkmetric.Exporter`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/metric#Exporter), with
  cumulative temporality by default.
- **Wire format** — protobuf `ExportTraceServiceRequest` and `ExportMetricsServiceRequest` from the slim
  module, POSTed as `application/x-protobuf` to `<endpoint>/v1/traces` and `<endpoint>/v1/metrics`,
  gzip-compressed by default, per the [OTLP specification](https://opentelemetry.io/docs/specs/otlp/).
- **Retries** — 429, 502, 503, and 504 are retried with exponential backoff and jitter, honoring
  `Retry-After`, within the export timeout. Other failures drop the batch and log once; partial success
  logs the rejected count at `warn`.
- **Transport** — no redirects; response bodies are read up to 64 KiB; TLS is verified against
  `tls.caBundle`, or the environment CA bundle by default. A client certificate, if any, comes from the
  `WithClientTLS` callback, so renewed service certificates apply without restarts.
- **Transform** — ported from opentelemetry-go's `exporters/otlp/otlptrace/internal/tracetransform` and
  `exporters/otlp/otlpmetric/otlpmetrichttp/internal/transform` (both present at v1.46.0), with imports
  switched from `go.opentelemetry.io/proto/otlp` to the slim module. Apache-2.0 notices are kept, and the
  port is re-synced on every OpenTelemetry upgrade.

### Dependencies

- **Forge repositories** — none. Consumed by every Forge Go executable and by `forge-sdk`'s examples.
- **Third-party modules** — measured on 2026-09-15 with a throwaway module using zerolog v1.35.1, the
  OpenTelemetry API, SDK, and metric SDK v1.46.0, and `proto/slim/otlp` v1.11.0, plus a protobuf marshal
  and a `net/http` POST. It links **16 modules**:
  - zerolog stack: `rs/zerolog`, `mattn/go-colorable`, `mattn/go-isatty`, `golang.org/x/sys`
  - OpenTelemetry: `otel`, `otel/trace`, `otel/metric`, `otel/sdk`, `otel/sdk/metric`, `auto/sdk`,
    `proto/slim/otlp`
  - Supporting: `google.golang.org/protobuf`, `go-logr/logr`, `go-logr/stdr`, `google/uuid`,
    `cespare/xxhash/v2`

  `go list -m all` reports 31 modules and no gRPC. 0001 estimates "about 8" for the exporter on an
  unverified counting basis, so this measured list, not the estimate, seeds `deps.allow`.
- **Versions** — v1.46.0 is the latest stable OpenTelemetry Go release (v1.47.0-rc.1 exists), pinned
  exactly.
- **Tests** use the standard library and `testify`, which every starter already requires. The official
  exporters appear only in the nested `parity` module.

### Data & storage

None beyond in-memory export batches. The span processor's queue is bounded and logs its drop count.

### Security

- **No secrets in telemetry.** Resource attributes and default fields hold no credentials. Collector
  headers come from `headersFile` and are never logged. Values marked `x-forge-sensitive`
  ([0002](0002-forge-api-schema.md)) are never logged.
- **TLS for telemetry.** A plaintext `http://` endpoint is a last-resort feature (`plaintext-telemetry`):
  refused in `production` without an override, and warned in `staging`.
- **Bounded input.** Incoming `traceparent` values at public ingress start new traces, exporter
  responses are size-capped, and zerolog JSON-escapes values, so field content cannot forge log lines.

### Environment awareness

| Default              | `production`              | `staging`       | `test`          | `development`   |
|----------------------|---------------------------|-----------------|-----------------|-----------------|
| Log format           | JSON                      | JSON            | JSON            | console         |
| Trace sample ratio   | 0.1                       | 0.1             | 1.0             | 1.0             |
| Plaintext telemetry  | refused unless overridden | allowed, warned | allowed         | allowed         |
| Last-resort features | refused unless overridden | allowed, logged | allowed, logged | allowed, logged |

Explicit configuration overrides every default except the `production` last-resort gate, which requires
`overrides` (0001).

### Logging & telemetry

This repository is the implementation; see Interfaces. Its own diagnostics, such as exporter failures and
dropped batches, go to `logging.Default`, rate-limited, and are not exported, which avoids feedback loops.

### Configuration

| YAML                           | Variable                                 | Default                     |
|--------------------------------|------------------------------------------|-----------------------------|
| `logging.default.level`        | `<PREFIX>_LOG_LEVEL`                     | `info`                      |
| `logging.access.enabled`       | `<PREFIX>_ACCESS_LOG_ENABLED`            | `true`                      |
| `environment.name` / `.tier`   | `<PREFIX>_ENVIRONMENT_NAME` / `_TIER`    | none — required             |
| `telemetry.enabled`            | `<PREFIX>_TELEMETRY_ENABLED`             | `true`                      |
| `telemetry.endpoint`           | `<PREFIX>_TELEMETRY_ENDPOINT`            | none — required if enabled  |
| `telemetry.timeout`            | `<PREFIX>_TELEMETRY_TIMEOUT`             | `10s`                       |
| `telemetry.headersFile`        | `<PREFIX>_TELEMETRY_HEADERS_FILE`        | empty                       |
| `telemetry.traces.sampleRatio` | `<PREFIX>_TELEMETRY_TRACES_SAMPLE_RATIO` | by tier                     |
| `telemetry.metrics.interval`   | `<PREFIX>_TELEMETRY_METRICS_INTERVAL`    | `60s`                       |
| `telemetry.tls.caBundle`       | `<PREFIX>_TELEMETRY_TLS_CA_BUNDLE`       | `environment.caBundle`      |

### Build, release & versioning

- **Bootstrap** from `go-library-starter`. Its `logging` package seeds `pkg/logging`; `greeter`, `pool`,
  `health`, `config`, and `obfusicate` are removed, and the version accessor drops `Masterminds/semver`.
- **Upgrades** — a Dependabot group moves `go.opentelemetry.io/*` together. Each upgrade re-syncs the
  transform port, passes `parity`, and ships as a minor release.
- **Versioning** — `v0.x` until accepted. Renaming or removing a log field, attribute, or config key is
  a breaking change.
- **CI** — the starters' workflows plus the `deps.allow` check from [CONVENTIONS.md](CONVENTIONS.md).

### Testing

- **`logging`** — golden JSON per tier; timestamps advance between events; trace fields appear only for
  valid spans.
- **`environment`** — tables for validation, variable hydration, and `AllowLastResort` across tiers and
  overrides.
- **Exporter** — an `httptest` collector decodes protobuf and gzip. Tests cover retry codes and
  `Retry-After` with a fake clock, partial success, the response cap, client-certificate rotation, and
  flush on shutdown.
- **Parity** — for fixed inputs, the ported transform's protobuf output equals the official exporter's,
  in the nested `parity` module.
- **Hygiene** — `-race`, allocation benchmarks on the export path, and the module allowlist.

## Alternatives considered

- **Official OTLP/HTTP exporters** — they link gRPC even over HTTP; rejected by 0001.
- **[`log/slog`](https://pkg.go.dev/log/slog) with an OpenTelemetry bridge** — standard library, but it
  replaces the starters' zerolog in every repository.
- **OTLP log export** — OpenTelemetry Go lists logs as Release Candidate. It would add a bridge and SDK
  modules, and stdout JSON meets current needs.
- **Prometheus pull exporter** — adds `client_golang` and its dependencies plus a second metrics path.
- **`otelhttp` contrib instrumentation** — an extra module for what two small wrappers do.
- **Honoring standard `OTEL_*` variables** — familiar, but a second configuration path that bypasses the
  starters' config validation and dump.
- **Reverse-domain attribute prefix** (`com.servercurio.forge.*`) — what the semantic-convention
  [naming guidance](https://opentelemetry.io/docs/specs/semconv/general/naming/) recommends, but verbose.
- **`environment` in each repository or in `forge-sdk`** — duplicates tier logic, or mixes
  configuration into the security library.

## Open questions

- **Scope** — do the `environment` and `service` packages fit a library 0001 describes as logging and
  telemetry?
- **Log export** — should logs move to OTLP once the Go logs signal is stable?
- **Attribute prefix** — `forge.*` or `com.servercurio.forge.*`?
- **Sampling** — are the default ratios right, and is tail sampling at the collector in scope for
  [0005](0005-forge-infrastructure.md)?
- **Collector identity** — does the in-environment collector hold a Forge certificate, and under which
  SPIFFE path, given that `/service/<repository>` names only Forge repositories?
- **Temporality** — cumulative (proposed) or delta?
- **Timestamps** — do existing log pipelines depend on the starters' Unix-second timestamps?

## References

- [0001 — Project Repositories](0001-project-repositories.md) — telemetry stack, environment awareness,
  bootstrap procedure.
- [0003 — forge-sdk](0003-forge-sdk.md) — TLS configuration and the transport wrapper hook.
- [CONVENTIONS.md](CONVENTIONS.md) — log fields, attribute names, environment keys.
- [zerolog](https://github.com/rs/zerolog) — [`Hook`](https://pkg.go.dev/github.com/rs/zerolog#Hook),
  `Event.Ctx`, `Event.GetCtx`.
- [OpenTelemetry Go](https://github.com/open-telemetry/opentelemetry-go) — signal status and the
  [OTLP/HTTP exporter](https://github.com/open-telemetry/opentelemetry-go/tree/main/exporters/otlp/otlptrace/otlptracehttp).
- [opentelemetry-go#2579](https://github.com/open-telemetry/opentelemetry-go/issues/2579) — the official
  OTLP/HTTP exporters depend on gRPC.
- [`go.opentelemetry.io/proto/slim/otlp`](https://github.com/open-telemetry/opentelemetry-proto-go/blob/main/slim/otlp/go.mod)
  — OTLP protobuf types without gRPC.
- [OTLP specification](https://opentelemetry.io/docs/specs/otlp/) — paths, content type, gzip, retryable
  codes, `Retry-After`.
- [Trace context in non-OTLP logs](https://opentelemetry.io/docs/specs/otel/compatibility/logging_trace_context/).
- [Deployment](https://opentelemetry.io/docs/specs/semconv/registry/attributes/deployment/),
  [service](https://opentelemetry.io/docs/specs/semconv/registry/attributes/service/), and
  [HTTP](https://opentelemetry.io/docs/specs/semconv/registry/attributes/http/) attribute registries.
- [Semantic convention naming](https://opentelemetry.io/docs/specs/semconv/general/naming/).
- [OpenTelemetry SDK environment variables](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/).
- [`sdktrace.SpanExporter`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/trace#SpanExporter) and
  [`sdkmetric.Exporter`](https://pkg.go.dev/go.opentelemetry.io/otel/sdk/metric#Exporter).
- [W3C Trace Context](https://www.w3.org/TR/trace-context/).
- [Go method values](https://go.dev/ref/spec#Method_values) — why `time.Now().UTC` captures one instant.
- [go-library-starter](https://github.com/servercurio/go-library-starter),
  [go-echo-starter](https://github.com/servercurio/go-echo-starter), and
  [go-cli-starter](https://github.com/servercurio/go-cli-starter) — the `logging` packages being replaced.
