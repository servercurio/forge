# 0008 — forge-gateway

- **Status:** Draft
- **Owner:** Nathan Klick
- **Date:** 2026-09-15
- **Summary:** `forge-gateway` is Forge's only edge. An operator ingress accepts bearer tokens over the
  starter's TLS 1.2+ configuration, and a separate agent ingress requires mutual TLS 1.3 with per-agent
  certificates. It routes only operations the contract exposes to that ingress, verifies tokens and
  revocation and fails closed, and forwards requests to services over mutual TLS. It is built on Echo
  with the standard library's `httputil.ReverseProxy`.

> An initial draft with concrete proposals, bounded by the
> [Resolved decisions](0001-project-repositories.md#resolved-decisions) in 0001. Conventions other
> repositories depend on are summarized in [CONVENTIONS.md](CONVENTIONS.md).

## Context & goals

0001 routes all `forge-sdk` traffic through `forge-gateway`, which "enforces authentication and
authorization before routing"
([Architecture at a glance](0001-project-repositories.md#architecture-at-a-glance)).
Agents use a dedicated mutual-TLS ingress, separate from the operator and third-party entry point, and the
enrollment route is the only one there that accepts a connection without a client certificate
([Agent enrollment](0001-project-repositories.md#agent-enrollment)). The gateway checks agent
certificates with OCSP, falls back to the CRL, caches both until `nextUpdate` (1 hour and 24 hours), and
fails closed. Tokens carry the environment ID, and every Forge certificate carries a SPIFFE ID
([Environment identity](0001-project-repositories.md#environment-identity)). 0002 leaves routing, rate
limits, and principal propagation to this document ([0002](0002-forge-api-schema.md)).

**Goals**

- Two isolated ingresses, with exposure decided by each operation's `x-forge-audience`, never by guesswork.
- Enforce authentication, environment binding, and revocation at the edge, failing closed.
- Hand services a verified principal over mutual TLS so they never parse credentials themselves.
- Rate-limit, correlate, and trace every request with the fewest possible added dependencies.

**Non-goals**

- Token formats, signing keys, RBAC model, and CA operation — [0006](0006-forge-identity.md).
- Login flows and the SSO site — [0007](0007-forge-sso.md).
- Fine-grained, resource-level authorization — each service (e.g. [0009](0009-forge-inventory.md)).
- Load balancers, DNS, and deployment topology — [0005](0005-forge-infrastructure.md).

## Proposal

### Responsibilities

- **Operator ingress** — TLS 1.2+, bearer tokens, `operator`-audience operations and the `/sso/` token
  endpoints.
- **Agent ingress** — TLS 1.3 with client certificates, `agent`-audience operations, one enrollment route.
- **Routing** — exact operation matching built from `forge-api-schema`; the first path segment selects
  the upstream service.
- **Edge security** — token verification, certificate revocation, rate limits, header hygiene.
- **Principal propagation** — `X-Forge-Principal` to services over mutual TLS.
- **Correlation** — `X-Request-Id`, W3C Trace Context, access logs, and metrics through `forge-common`.

### Interfaces

#### Listeners

| Listener   | Default port | TLS                        | Client authentication                          |
|------------|--------------|----------------------------|------------------------------------------------|
| `operator` | 8443         | 1.2+, starter cipher suites | `Authorization: Bearer`                       |
| `agent`    | 9443         | 1.3 only                   | X.509-SVID `spiffe://<env-id>/agent/<id>`      |
| `health`   | 8080         | none, private address only | none; `/livez`, `/readyz`, `/healthz` only    |

Both TLS listeners serve the gateway's service certificate,
`spiffe://<environment-id>/service/forge-gateway`, issued by `forge-identity` and renewed by `forge-sdk`'s `enroll.Renewer`. `forge-sdk` clients refuse a
gateway without that ID ([0003](0003-forge-sdk.md)). The operator certificate also needs the public DNS
names as SANs (see Open questions). The starter's hardened TLS 1.2 configuration
(`hardenedTLSConfig` in `go-echo-starter`) is kept for the operator listener. The starter's plain-HTTP
redirect and ACME `autocert` paths are removed. Both TLS listeners must sit behind TCP pass-through load
balancing, because TLS terminates at the gateway.

#### Routing and audience enforcement

Proposed: **deny by default, with exact operation matching.** At startup the gateway reads every
document from `forge-api-schema`'s `pkg/openapi` and builds two Echo routers:

1. For each operation, the method and path template (`{endpointId}` becomes `:endpointId`) go into the
   `operator` router if `x-forge-audience` contains `operator`, and into the `agent` router if it
   contains `agent`. Operations whose only audience is `internal` go into neither.
2. The first path segment selects the upstream: `inventory` → `forge-inventory`, `identity` →
   `forge-identity`, `provisioner` → `forge-provisioner`. An operation whose segment has no configured
   upstream fails startup.
3. Anything unmatched returns `404` with the problem code `route_not_found`, so an `internal` operation
   and a nonexistent one look the same from outside.

Before matching, the gateway rejects with `400` any path that contains `..` segments, `//`, percent-encoded
`/` or `\`, or invalid UTF-8, and any query string that `url.ParseQuery` rejects. That way the gateway
and the upstream service never interpret a request differently, a risk the `ReverseProxy.Rewrite`
documentation warns about.

Two non-contract route families exist. `GET /gateway/v1alpha1/environment` returns
`{ "id", "name", "tier" }` without authentication, so `forge-cli` can record a profile's name and tier
after pinning the certificate ([0010](0010-forge-cli.md)). The `/sso/` family, proposed as
`/sso/oauth2/device-authorization`, `/sso/oauth2/token`, and `/sso/oauth2/revoke`, is passed to
`forge-sso`, which owns those protocol endpoints ([0007](0007-forge-sso.md) decides). This keeps every
credential exchange on the environment-pinned host.

#### Reverse proxy

Proposed: **Echo v5 for listeners, middleware, and routing, and `net/http/httputil.ReverseProxy` with
`Rewrite` for forwarding.** No new module is needed. Echo's own `middleware.Proxy` is not used, because it
builds on `httputil.NewSingleHostReverseProxy` (`proxy.go` line 423 in v5.3.1), which uses `Director`. Go
documents `Director` as insecure: a client can list headers in `Connection` to strip headers the director
added, and inbound `X-Forwarded-*` headers survive. One `ReverseProxy` per upstream:

```go
proxy := &httputil.ReverseProxy{
    Rewrite: func(r *httputil.ProxyRequest) {
        r.SetURL(upstream)                     // https://forge-inventory.<internal>:8443
        r.Out.Host = upstream.Host
        stripForgeHeaders(r.Out.Header)        // X-Forge-* from clients, Authorization, Cookie
        r.Out.Header.Set("X-Forge-Principal", principal.Encode(r.In.Context()))
        r.Out.Header.Set("X-Request-Id", requestID(r.In.Context()))
        r.SetXForwarded()                      // client IP from the gateway's own extractor only
    },
    Transport:      telemetry.WrapTransport(upstreamTransport), // forge-sdk tlsconfig + forge-common spans
    ErrorHandler:   problemUpstreamError,      // 502 upstream_unavailable, 504 upstream_timeout
    ModifyResponse: scrubResponseHeaders,      // drop Server; enforce Cache-Control: no-store
}
```

`upstreamTransport` uses `tlsconfig.Client` from `forge-sdk` with TLS 1.3, the environment roots, and
`spiffe.Service("forge-inventory")` as the matcher. The gateway therefore verifies the upstream's trust
domain and its exact service path, so a compromised `forge-provisioner` cannot answer for
`forge-inventory`.

#### Principal propagation

The gateway removes the inbound `Authorization` header and sends `X-Forge-Principal`: unpadded
base64url JSON of the verified identity.

```json
{ "type": "user", "id": "<subject>", "tenantId": "<tenant>", "roles": ["inventory.viewer"],
  "tokenId": "<jti>", "expiresAt": "2026-09-15T12:15:00Z" }
```

For agents, `type` is `agent`, `id` is the agent ID from the certificate, and `tenantId` comes from a
cached lookup of the agent in `forge-identity` (an `internal` operation that 0006 defines). Services
accept the header only when their mutual-TLS peer is `spiffe://<environment-id>/service/forge-gateway`,
and reject it from any other peer. Mutual TLS protects the header's integrity, and the bearer token never
travels past the edge. A stdlib-only parser, proposed as `forge-sdk` `pkg/principal`, keeps every service
consistent (see Open questions).

#### Errors and headers

All gateway-generated errors use `application/problem+json` with the codes `route_not_found`,
`unauthenticated` (401, with `WWW-Authenticate: Bearer`), `forbidden`, `rate_limited` (429, with
`Retry-After`), `certificate_revoked`, `revocation_unavailable`, `upstream_unavailable`, and
`upstream_timeout`. Problem bodies never say why a token failed beyond `unauthenticated`. The reason goes
only to logs.

### Dependencies

- **Forge** — `forge-api-schema` (embedded documents), `forge-sdk` (`spiffe`, `tlsconfig`, `revocation`,
  `enroll`, `principal`), `forge-common` (`logging`, `environment`, `telemetry`).
- **New third-party module** — [`github.com/go-jose/go-jose/v4`](https://github.com/go-jose/go-jose)
  v4.1.5, for JWKS parsing and JWS verification. Its `go.mod` has no requirements, measured on 2026-09-15.
- **Kept from the starter** — Echo v5.3.1, which links only `golang.org/x/time` (rate limiter),
  `golang.org/x/net` (`netutil.LimitListener`), zerolog, `errorx`, and `yaml.v3`.
- **Removed from the starter** — `internal/database` (pgx, bun, goose), swaggo and `cmd/openapi-gen`
  (replaced by embedded contracts, per 0002), ACME `autocert`, and `Masterminds/semver`.
- **Measured footprint** — the starter's `cmd/daemon` links 49 third-party modules (151 in
  `go list -m all`). A throwaway module with the proposed set, plus `forge-common`'s measured OpenTelemetry
  stack, links **23** (44 in `go list -m all`). Forge modules themselves are not yet published and are
  not counted.

### Data & storage

No persistent state. In memory: the JWKS (refreshed every 5 minutes, or on an unknown `kid` at most once
every 30 seconds), revocation answers inside `revocation.Checker`, agent-to-tenant lookups (5 minutes), and
rate-limit buckets.

### Security

#### Token verification (operator ingress)

Token formats belong to [0006](0006-forge-identity.md). For JWT access tokens
([RFC 9068](https://www.rfc-editor.org/rfc/rfc9068)), the gateway enforces:

- **Algorithm allowlist** — `ES256` only, and `typ` must be `at+jwt`, following
  [RFC 8725](https://www.rfc-editor.org/rfc/rfc8725). `none`, HMAC, and embedded `jwk` or `x5u` headers
  are rejected.
- **Keys** — the environment's JWKS, fetched from `forge-identity` over mutual TLS only.
- **Environment binding** — `iss` must equal `auth.issuer`, which must contain `environment.id` (checked at
  startup). `aud` must contain `spiffe://<environment-id>/service/forge-gateway`. A token from another
  environment therefore fails on both issuer and audience.
- **Time** — `exp` is required, `nbf` and `iat` are honored, and clock skew is at most 60 seconds.
- **Coarse authorization** — a `bearerAuth` security requirement lists the role names the operation
  needs, which OpenAPI 3.1 permits for non-OAuth schemes. The gateway requires at least one of them in
  `roles`. Tenant- and resource-level checks stay in services.
- **Opaque API tokens**, if 0006 chooses them, go to `forge-identity` introspection
  ([RFC 7662](https://www.rfc-editor.org/rfc/rfc7662)) over mutual TLS, with results cached for at most
  30 seconds.

#### Agent ingress and revocation

- **TLS configuration** — `tlsconfig.Server` with TLS 1.3, `ClientAuth: VerifyClientCertIfGiven`, the
  environment roots as `ClientCAs`, and `spiffe.Agent()` as the matcher. Revocation is checked in
  `VerifyConnection`, which Go runs "for all connections, including resumptions". `VerifyPeerCertificate`
  is not used, because it is skipped on resumed sessions.
- **Enrollment route** — middleware rejects any request without a verified agent certificate, unless the
  matched operation is the agent enrollment operation that 0002 marks `security: []`. The allowlist holds
  one operation ID, tested from the contract. The enrollment route's body limit is 16 KiB, and its rate
  limit is the strictest.
- **Revocation** — `revocation.Checker` from `forge-sdk` implements 0001 exactly: OCSP first, then the
  CRL, each cached until `nextUpdate`, rejecting when neither is available within the window. The gateway
  re-checks the cached status on every request, not only at handshake, so a long-lived HTTP/2 connection
  is cut off once its certificate is revoked.
- **Responder addresses** — OCSP and CRL URLs come from gateway configuration (the internal
  `forge-identity` endpoints), not from certificate AIA or CDP extensions, so agent-supplied certificates
  cannot steer the gateway's outbound requests. This needs a `revocation` option in `forge-sdk`.

#### Rate limiting

Echo's `RateLimiter` middleware with its memory store (`golang.org/x/time/rate`), keyed per ingress:

| Bucket                       | Key                   | Proposed default          |
|------------------------------|-----------------------|---------------------------|
| Operator, before auth        | client IP             | 20 req/s, burst 40        |
| Operator, after auth         | principal `id`        | 10 req/s, burst 20        |
| `/sso/` token endpoints      | client IP             | 1 req/s, burst 5          |
| Agent                        | agent ID              | 1 req/s, burst 10         |
| Agent enrollment             | client IP             | 10 per minute, burst 5    |

The starter's `netutil.LimitListener` caps connections per listener before the TLS handshake. Client IPs
come from the starter's proxy extractor with `useDirectIP` by default. The starter's `X-Forwarded-For`
extractor always trusts private, loopback, and link-local ranges (`application_proxy.go`). The gateway
removes that implicit trust and honors only explicitly configured ranges, because its callers can come
from private networks.

#### Header and response hygiene

- **Inbound** — `X-Forge-*`, `Forwarded`, and `X-Forwarded-*` headers from clients are removed, and
  `Cookie` is dropped because Forge APIs do not use cookies. CSRF and CORS middleware are off; no
  browser origin calls the gateway (see Open questions).
- **Responses** — the starter's `Secure` middleware (HSTS, `nosniff`, frame denial) plus
  `Cache-Control: no-store`.
- **Body limits** — 1 MiB on the operator ingress and 8 MiB on the agent ingress (inventory reports). Both
  are enforced on the bytes received; decompression limits belong to the service that decodes the body.

### Environment awareness

- **Startup** — `environment.name`, `tier`, `id`, and `caBundle` are all required, and `id` must match the
  bundle's trust domain, the issuer, and the gateway's own certificate. A mismatch refuses startup.
- **Tiers** — all tiers enforce mutual TLS, token verification, and fail-closed revocation, because there
  is no insecure mode to leak into `production`. `Hardened()` turns off the OpenAPI UI and raises the log
  level floor to `info`. `development` may serve a filtered operator contract at `/openapi.yaml`.
- **Last-resort features** — none defined.

### Logging & telemetry

- **Access logs** — through `forge-common`, with `http.request.method`, `http.route` (the contract
  template, never the raw path), `http.response.status_code`, and `client.address`, plus `forge.ingress`,
  `forge.request.id`, `forge.upstream.service`, `forge.principal.type`, `forge.tenant.id`,
  `forge.agent.id`, and `forge.auth.failure_reason`. Tokens, the principal header, and query strings are
  never logged.
- **Request IDs** — an inbound `X-Request-Id` is kept only if it matches `^[A-Za-z0-9._-]{1,64}$`.
  Otherwise the gateway generates a 26-character random base32 ID. The ID is forwarded upstream and echoed
  on every response.
- **Tracing** — `WrapHandler` runs with `WithTrustIncoming(false)` on both external ingresses, starting a
  new trace linked to the caller's span ([0004](0004-forge-common.md)). `WrapTransport` injects
  `traceparent` upstream, and inbound `tracestate` is not forwarded.
- **Metrics** — `http.server.request.duration` plus `forge.gateway.auth.failures` (by reason),
  `forge.gateway.revocation.checks` (by source `ocsp`, `crl`, or `cache`, and by result),
  `forge.gateway.rate_limited` (by bucket), and `forge.gateway.upstream.errors` (by service).

### Configuration

Prefix `FORGE_GATEWAY_`. Starter keys (`LOG_*`, timeouts) and the `environment` and `telemetry` blocks are
omitted here.

| YAML                                   | Variable                                          | Default            |
|----------------------------------------|---------------------------------------------------|--------------------|
| `server.operator.port`                 | `FORGE_GATEWAY_SERVER_OPERATOR_PORT`              | `8443`             |
| `server.operator.maxBodySize`          | `FORGE_GATEWAY_SERVER_OPERATOR_MAX_BODY_SIZE`     | `1MiB`             |
| `server.agent.enabled` / `.port`       | `FORGE_GATEWAY_SERVER_AGENT_ENABLED` / `_PORT`    | `true` / `9443`    |
| `server.agent.maxBodySize`             | `FORGE_GATEWAY_SERVER_AGENT_MAX_BODY_SIZE`        | `8MiB`             |
| `server.health.port`                   | `FORGE_GATEWAY_SERVER_HEALTH_PORT`                | `8080`             |
| `certificate.dir`                      | `FORGE_GATEWAY_CERTIFICATE_DIR`                   | required           |
| `certificate.enrollmentTokenFile`      | `FORGE_GATEWAY_CERTIFICATE_ENROLLMENT_TOKEN_FILE` | first start only   |
| `certificate.serviceAccountTokenFile`  | `FORGE_GATEWAY_CERTIFICATE_SERVICE_ACCOUNT_TOKEN_FILE` | first start on Kubernetes |
| `upstreams.<segment>.url`              | `FORGE_GATEWAY_UPSTREAMS_INVENTORY_URL`, …        | required per route |
| `upstreams.<segment>.timeout`          | `FORGE_GATEWAY_UPSTREAMS_INVENTORY_TIMEOUT`, …    | `30s`              |
| `auth.issuer`                          | `FORGE_GATEWAY_AUTH_ISSUER`                       | required           |
| `auth.jwksUrl`                         | `FORGE_GATEWAY_AUTH_JWKS_URL`                     | required           |
| `auth.clockSkew`                       | `FORGE_GATEWAY_AUTH_CLOCK_SKEW`                   | `60s` (max `60s`)  |
| `revocation.ocspUrl` / `.crlUrl`       | `FORGE_GATEWAY_REVOCATION_OCSP_URL` / `_CRL_URL`  | required           |
| `rateLimit.<bucket>.rate` / `.burst`   | `FORGE_GATEWAY_RATE_LIMIT_AGENT_RATE`, …          | table above        |
| `proxy.trustedIPRanges`                | `FORGE_GATEWAY_PROXY_TRUSTED_IP_RANGES`           | empty              |

### Build, release & versioning

- **Bootstrap** from `go-echo-starter`, removing the database, swaggo, ACME, and HTTP-redirect code. The
  binary is `cmd/forge-gateway`.
- **Contract coupling** — a new `operator` or `agent` operation is reachable only after the gateway
  upgrades `forge-api-schema`. A 100-series workflow opens that pull request on each schema release, like
  `forge-sdk`'s regeneration workflow.
- **Deployment** — the gateway ships every artifact in
  [CONVENTIONS — Deployment artifacts](CONVENTIONS.md#deployment-artifacts): the signed multi-arch image,
  the starter's Helm chart in `charts/forge-gateway/` with the enrollment init container, signed deb and
  rpm packages with a hardened systemd unit, and a signed MSI. [0005](0005-forge-infrastructure.md)
  deploys them to Kubernetes, container, and OS targets with Ansible.
- **Versioning** — `v0.x`, per [CONVENTIONS.md](CONVENTIONS.md).

### Testing

- **Audience matrix** — generated from the contract. Every `internal`-only operation returns `404` on both
  ingresses, every `operator` operation returns `404` on the agent ingress, and the reverse.
- **TLS** — with `forge-sdk` `sdktest`: TLS 1.2 rejected on the agent ingress; missing, expired,
  wrong-trust-domain, and non-agent certificates rejected on every route except enrollment; revoked
  certificates rejected at handshake, on resumption, and mid-connection; OCSP down falls back to the CRL,
  and both down beyond `nextUpdate` rejects.
- **Tokens** — tables of wrong `alg`, `typ`, `iss`, and `aud`, other-environment keys, expired tokens, and
  unknown `kid` refresh throttling.
- **Smuggling and hygiene** — `Connection: X-Forge-Principal`, spoofed `X-Forwarded-For`, encoded
  slashes, and duplicate `Authorization` headers.
- **Fuzzing** of the path normalizer and principal encoder; `-race`; benchmarks of the per-request
  revocation re-check.

## Alternatives considered

- **Echo `middleware.Proxy`** — it uses `NewSingleHostReverseProxy` and `Director`, which Go documents as
  insecure. It also adds load-balancer features the gateway does not need.
- **Plain `net/http` without Echo** — removes one module, but gives up the starter's middleware, config,
  and rate limiter, and diverges from every other service.
- **Envoy or another off-the-shelf proxy** — mature, but not Go, and it cannot build route tables from
  `x-forge-audience` or use `forge-sdk`'s SPIFFE and revocation code without a control plane.
- **Prefix routing on the first segment only**, leaving audience checks to services — simpler and needs no
  contract coupling, but one missed check in a service would expose `internal` operations.
- **Forwarding the bearer token** so services re-verify it — defense in depth, but it spreads tokens and a
  JWT library to every service. It remains an option if 0006 prefers it.
- **[`golang-jwt/jwt/v5`](https://github.com/golang-jwt/jwt)** v5.3.1 — also has no requirements (measured),
  but it has no JWKS support, so key parsing would be hand-written.
- **A separate port for enrollment** with `RequireAndVerifyClientCert` on the main agent port — stronger
  TLS-layer enforcement, but 0001 describes enrollment as a route on the agent ingress. Worth
  reconsidering if the single-operation allowlist proves fragile.
- **A shared rate-limit store** (e.g. Redis) — exact global limits, but adds a client module and a stateful
  dependency. Per-replica limits are proposed first.
- **Advertising limits with `RateLimit` headers** — still an Internet-Draft
  ([draft-ietf-httpapi-ratelimit-headers-11](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/));
  `Retry-After` alone is used for now.

## Open questions

- **Operator certificate** — may `forge-identity` add public DNS SANs to the gateway's service certificate,
  or does the operator ingress use a second environment-CA certificate? A WebPKI-terminating load balancer
  would break client pinning (the same question is open in 0003).
- **Principal header** — accept `X-Forge-Principal` over mutual TLS, or forward tokens? Does
  `pkg/principal` belong in `forge-sdk`?
- **Unauthenticated operations** — 0002's lint allowlist names only enrollment and health. Add
  `GET /gateway/v1alpha1/environment`?
- **SSO routing** — does [0007](0007-forge-sso.md) accept `/sso/` token endpoints behind the gateway?
- **Agent tenant lookup** — cache TTL, and whether disabling an agent in `forge-identity` should also
  revoke its certificate.
- **Role names** in security requirements, or a dedicated `x-forge-permission` extension?
- **Browser clients** — will a web console need CORS on the operator ingress?
- **Rate limits** across replicas — are per-replica limits acceptable at expected scale?

## References

- [0001 — Project Repositories](0001-project-repositories.md) — architecture, agent enrollment,
  environment identity, resolved decisions.
- [0002 — forge-api-schema](0002-forge-api-schema.md), [0003 — forge-sdk](0003-forge-sdk.md),
  [0004 — forge-common](0004-forge-common.md), [CONVENTIONS.md](CONVENTIONS.md).
- [go-echo-starter](https://github.com/servercurio/go-echo-starter) — `internal/application/application_tls.go`
  (`hardenedTLSConfig`, `LimitListener`), `application_proxy.go` (implicit private-range trust),
  `config_ratelimit.go`, `config_security.go`.
- [Echo v5 middleware](https://github.com/labstack/echo/tree/v5.3.1/middleware) — `proxy.go`,
  `rate_limiter.go`, `request_id.go` (inspected at v5.3.1).
- [`httputil.ReverseProxy`](https://pkg.go.dev/net/http/httputil#ReverseProxy) — `Rewrite` and the
  `Director` security notes; [`ProxyRequest.SetXForwarded`](https://pkg.go.dev/net/http/httputil#ProxyRequest.SetXForwarded).
- [`tls.Config`](https://pkg.go.dev/crypto/tls#Config) — `VerifyConnection` runs on resumptions;
  `VerifyPeerCertificate` does not.
- [go-jose v4](https://github.com/go-jose/go-jose) and [golang-jwt v5](https://github.com/golang-jwt/jwt).
- [OpenAPI 3.1.1 Security Requirement Object](https://spec.openapis.org/oas/v3.1.1.html#security-requirement-object)
  — role names for non-OAuth schemes.
- [RFC 9068](https://www.rfc-editor.org/rfc/rfc9068) (JWT access tokens),
  [RFC 8725](https://www.rfc-editor.org/rfc/rfc8725) (JWT best practices),
  [RFC 7662](https://www.rfc-editor.org/rfc/rfc7662) (token introspection),
  [RFC 6750](https://www.rfc-editor.org/rfc/rfc6750) (bearer tokens, `WWW-Authenticate`).
- [RFC 5280](https://www.rfc-editor.org/rfc/rfc5280) and [RFC 6960](https://www.rfc-editor.org/rfc/rfc6960)
  — CRLs and OCSP.
- [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) — problem details;
  [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110#name-retry-after) — `Retry-After`.
- [SPIFFE ID](https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE-ID.md) and
  [X.509-SVID](https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md).
- [W3C Trace Context](https://www.w3.org/TR/trace-context/).
- [draft-ietf-httpapi-ratelimit-headers](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/).
