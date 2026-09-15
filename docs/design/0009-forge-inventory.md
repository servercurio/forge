# 0009 — forge-inventory

- **Status:** Draft
- **Owner:** Nathan Klick
- **Date:** 2026-09-15
- **Summary:** `forge-inventory` is the tenant-scoped source of truth for every managed endpoint. Each
  endpoint has a class that fixes its enforcement path and attribute schema, operator-declared labels and
  attributes, agent- or provisioner-reported facts, and a revision history. Data lives in PostgreSQL behind
  row-level security, migrated with goose. The service accepts agent reports through the gateway's agent
  ingress and feeds `forge-provisioner` a change stream.

> An initial draft with concrete proposals, bounded by the
> [Resolved decisions](0001-project-repositories.md#resolved-decisions) in 0001. Conventions other
> repositories depend on are summarized in [CONVENTIONS.md](CONVENTIONS.md).

## Context & goals

0001 makes `forge-inventory` the "source-of-truth catalog and schemas of the managed servers, network
devices, and remote endpoints" and the upstream for `forge-agent`'s reported inventory
([Repository inventory](0001-project-repositories.md#repository-inventory)). Both enforcement paths, agent
and agentless, converge against it, and the endpoint class decides which path covers an endpoint
([Desired-state model](0001-project-repositories.md#desired-state-model--one-authority-two-enforcement-paths)).
Agents reach it only through the gateway's mutual-TLS agent ingress. It is part of the first domain slice
that proves the full request loop ([Sequencing](0001-project-repositories.md#sequencing--phases)).

**Goals**

- A resource model that separates what operators declare from what endpoints report.
- Hard tenant isolation, enforced in the database as well as in code.
- Cheap, idempotent ingestion of frequent agent reports.
- A queryable history and a change stream that `forge-provisioner` can reconcile from.

**Non-goals**

- Desired state, directives, and compliance results — [0011](0011-forge-provisioner.md).
- Agent identity, certificates, and tenants as accounts — [0006](0006-forge-identity.md).
- Device credentials for agentless management — not stored here (see Open questions).
- Fact collection on hosts — [0012](0012-forge-agent.md) and its plugins.

## Proposal

### Responsibilities

- **Catalog** — endpoints, endpoint classes, and relationships per tenant.
- **Schemas** — class-defined JSON Schemas for declared attributes and optional fact schemas.
- **Ingestion** — agent reports (agent audience) and agentless facts from `forge-provisioner` (internal).
- **History** — revisions of declared data and of facts, with retention.
- **Change stream** — ordered endpoint events for `forge-provisioner`.

### Interfaces

#### Resource model

| Concept          | Owner                       | Examples                                                   |
|------------------|-----------------------------|------------------------------------------------------------|
| `EndpointClass`  | built-in or tenant operator | `linux-server`, `windows-server`, `network-switch`         |
| `enforcement`    | class, immutable            | `agent` or `agentless` — exactly one path per endpoint     |
| `capabilities`   | class                       | `packages`, `services`, `interfaces`, `vlans`              |
| `labels`         | operators                   | `site=dc1`, `role=web`; used by selectors                  |
| `attributes`     | operators                   | `owner`, `rack`, `managementAddress`; validated by class   |
| `facts`          | agent or provisioner        | OS, hardware, interfaces, plugin data; read-only via API   |
| `relationships`  | operators or provisioner    | `connected-to`, `hosted-on`, `member-of`                   |
| revisions        | the service                 | who changed what, when                                     |

- **Declared versus reported.** Labels and attributes change only through operator calls. Facts change
  only through reports. Neither overwrites the other: a reported hostname never renames an endpoint.
- **Classes.** Built-in classes ship as embedded YAML and cannot be changed. Tenants may add classes with
  names that do not collide with built-ins. A class's `attributeSchema` and optional `factSchema` are
  JSON Schema 2020-12, matching the dialect of 0002. Removing a class that endpoints still use is refused.
- **Labels** follow the Kubernetes syntax: an optional DNS-subdomain prefix, then a name of 63 characters
  or fewer. The `forge.servercurio.com/` prefix is reserved. Selectors support `=`, `!=`, `in`, `notin`,
  and key existence.
- **Capabilities** — the class declares them. An agent reports the capabilities it actually has, based on
  installed plugins, as the fact `capabilities.observed`.
- **IDs** — opaque 26-character random base32 strings, like agent IDs. Names are unique among a tenant's
  non-retired endpoints.

```json
{
  "id": "b7k2m4q6r3t5v7x2z4c6d3f5g7",
  "name": "web-01.dc1",
  "class": "linux-server",
  "labels": { "site": "dc1", "role": "web" },
  "attributes": { "owner": "platform-team", "rack": "r12" },
  "management": { "mode": "agent", "agentId": "h2j4l6n3p5s7u2w4y6a3e5i7o2" },
  "lifecycle": "active",
  "factsReportedAt": "2026-09-15T10:04:05Z",
  "resourceVersion": "42",
  "createdAt": "2026-09-01T08:00:00Z",
  "updatedAt": "2026-09-15T10:04:05Z"
}
```

`lifecycle` is `registered`, `active`, `decommissioning`, or `retired`. Facts are returned by a separate
sub-resource so lists stay small.

#### API sketch

All paths are `/inventory/v1alpha1/...`. The tenant comes from the verified principal, as proposed in 0002
and [0008](0008-forge-gateway.md), never from the path.

| Method and path                                  | Audience             | Notes                                     |
|--------------------------------------------------|----------------------|-------------------------------------------|
| `GET /endpoints`                                 | operator, internal   | `labelSelector`, `class`, `lifecycle`     |
| `POST /endpoints`                                | operator             | declare an endpoint                       |
| `GET /endpoints/{endpointId}`                    | operator, internal   | `ETag` = `resourceVersion`                |
| `PATCH /endpoints/{endpointId}`                  | operator             | JSON Merge Patch; `If-Match` required     |
| `DELETE /endpoints/{endpointId}`                 | operator             | sets `retired`; purged after retention    |
| `GET /endpoints/{endpointId}/facts`              | operator, internal   | current facts                             |
| `PUT /endpoints/{endpointId}/facts`              | internal             | agentless facts from `forge-provisioner`  |
| `GET /endpoints/{endpointId}/revisions`          | operator             | `kind=declared` or `kind=facts`           |
| `GET`, `POST /endpoint-classes`                  | operator             | built-ins are listed but read-only        |
| `GET`, `PUT`, `DELETE /endpoint-classes/{name}`  | operator             | `PUT` for tenant classes only             |
| `GET /relationships`, `POST /relationships`      | operator, internal   | filter by `endpointId`, `type`            |
| `DELETE /relationships/{relationshipId}`         | operator             |                                           |
| `POST /agent-reports`                            | agent                | `x-forge-idempotent: true`                |
| `GET /endpoint-events`                           | internal             | `cursor`, `limit`, `waitSeconds` ≤ 30     |

```yaml
  /inventory/v1alpha1/agent-reports:
    post:
      operationId: createAgentReport
      x-forge-audience: [agent]
      x-forge-idempotent: true
      security: [{ mutualTLS: [] }]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/AgentReport" }
      responses:
        "202":
          description: Report accepted.
          content:
            application/json:
              schema: { $ref: "#/components/schemas/AgentReportReceipt" }
        default:
          $ref: "../../common/v1/components.yaml#/components/responses/Problem"
```

Lists use `limit` and `cursor`, with keyset pagination on `(id)` inside the tenant. Updates use
`application/merge-patch+json` ([RFC 7396](https://www.rfc-editor.org/rfc/rfc7396)) with `If-Match`
([RFC 9110](https://www.rfc-editor.org/rfc/rfc9110#name-if-match)). A stale version returns `412` with
the code `resource_version_conflict`.

#### Agent report ingestion

```
forge-agent ──mTLS 1.3──► forge-gateway (agent ingress)
            ──mTLS + X-Forge-Principal {agent, tenantId}──► forge-inventory
                                                             POST /inventory/v1alpha1/agent-reports
```

1. **Request** — the agent sends `{ reportId, sequence, collectedAt, previousDigest?, facts? }`. It
   never sends an agent or endpoint ID; identity comes only from the principal header, which is accepted
   only from the gateway's SPIFFE ID.
2. **Binding** — inventory finds the endpoint bound to the agent ID. On first contact it auto-registers
   one: the class comes from the reported OS family through `autoRegistration.classByOsFamily`, the name
   from the reported hostname (a numeric suffix resolves collisions), and the labels from the enrollment
   token's host labels, read from `forge-identity` through an `internal` operation that 0006 defines.
3. **Unchanged facts** — if `previousDigest` equals the digest the server issued last time, `facts` may
   be omitted. The report then only updates `factsReportedAt`. The digest is an opaque server value, so
   agents need no canonical JSON.
4. **Validation** — core facts are checked against the `AgentReport` schema from `forge-api-schema`, and
   `facts.plugins.<name>` against the class's `factSchema` where one is defined. Limits: 8 MiB
   decompressed per report, 256 KiB per plugin, nesting depth 32.
5. **Write** — one transaction: lock the endpoint row, ignore a `sequence` that is not newer (a replayed
   `reportId` returns the original receipt), store the facts, add a facts revision if the digest changed,
   and add an event.
6. **Receipt** — `202 { endpointId, factsDigest, nextReportAfter }`. The server sets the report cadence
   (default 5 minutes, with jitter), which spreads load across the fleet.

#### Relationship to `forge-provisioner`

- **Reads** — `forge-provisioner` selects endpoints with `labelSelector` through `internal` list and get
  operations, called service to service over mutual TLS, never through the gateway.
- **Watches** — it long-polls `GET /endpoint-events` with a durable cursor. Events are
  `endpoint.created`, `endpoint.declared-updated`, `endpoint.facts-updated`, `endpoint.retired`, and
  `class.updated`, each carrying `endpointId` and `resourceVersion`, not full documents. PostgreSQL
  `LISTEN`/`NOTIFY` wakes long-polls; the table is authoritative.
- **Writes** — for agentless devices only, it `PUT`s facts it discovered through device APIs, with its
  own principal as the revision actor.
- **Authority** — inventory stores no desired state and makes no reconciliation decisions.
  `forge-provisioner` compares its directives with declared attributes and facts. Internal callers name
  the tenant with `X-Forge-Tenant-Id`, which is honored only from SPIFFE IDs in `internalCallers`.

### Dependencies

- **Forge** — `forge-api-schema` (models, embedded document, `AgentReport` schema), `forge-sdk` (`spiffe`,
  `tlsconfig`, `revocation`, `enroll`, `principal`, and the identity client), `forge-common`.
- **Kept from the starter** — Echo v5.3.1; `jackc/pgx/v5` v5.11.0; `pressly/goose/v3` v3.28.0, which adds
  `mfridman/interpolate`, `sethvargo/go-retry`, `go.uber.org/multierr`, and `golang.org/x/sync`.
- **New** — `santhosh-tekuri/jsonschema/v6` v6.0.3, already pinned in CONVENTIONS, for class schemas.
- **Removed** — swaggo and `cmd/openapi-gen` (per 0002). **Proposed: drop `uptrace/bun`** and use
  `pgxpool` with hand-written SQL. Bun and `pgdialect` link 7 modules (`bun`, `pgdialect`,
  `jinzhu/inflection`, `puzpuzpuz/xsync/v3`, `tmthrgd/go-hex`, `vmihailenco/msgpack/v5`,
  `vmihailenco/tagparser/v2`), and the queries here (JSONB containment, `FOR UPDATE`, `SET LOCAL`) are
  plain SQL anyway. Goose keeps running through `pgx/v5/stdlib`.
- **Measured footprint** — with bun, the proposed set plus `forge-common`'s OpenTelemetry stack links 40
  third-party modules (123 in `go list -m all`). Without bun, subtracting its 7 gives about 33, not
  separately measured.

### Data & storage

PostgreSQL. The proposed minimum version is 16; see Open questions. Goose SQL migrations are embedded as
in the starter (`internal/database/migrations/sql/YYYYMMDDHHMMSS_*.sql`). Sketch:

```sql
-- +goose Up
CREATE TABLE endpoint_classes (
  tenant_id text,                         -- NULL for built-in classes
  name text NOT NULL,
  enforcement text NOT NULL CHECK (enforcement IN ('agent', 'agentless')),
  capabilities text[] NOT NULL DEFAULT '{}',
  attribute_schema jsonb NOT NULL, fact_schema jsonb,
  resource_version bigint NOT NULL DEFAULT 1,
  created_at timestamptz NOT NULL DEFAULT now(), updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX endpoint_classes_tenant_name ON endpoint_classes (tenant_id, name)
  WHERE tenant_id IS NOT NULL;
CREATE UNIQUE INDEX endpoint_classes_builtin_name ON endpoint_classes (name) WHERE tenant_id IS NULL;

CREATE TABLE endpoints (
  tenant_id text NOT NULL, id text NOT NULL, name text NOT NULL, class_name text NOT NULL,
  labels jsonb NOT NULL DEFAULT '{}', attributes jsonb NOT NULL DEFAULT '{}',
  management_mode text NOT NULL CHECK (management_mode IN ('agent', 'agentless')),
  agent_id text UNIQUE,                   -- agent IDs are unique within the environment
  lifecycle text NOT NULL DEFAULT 'registered',
  facts jsonb NOT NULL DEFAULT '{}', facts_digest bytea, facts_reported_at timestamptz,
  last_report_id text, last_report_sequence bigint NOT NULL DEFAULT 0,
  resource_version bigint NOT NULL DEFAULT 1,
  created_at timestamptz NOT NULL DEFAULT now(), updated_at timestamptz NOT NULL DEFAULT now(),
  retired_at timestamptz,
  PRIMARY KEY (tenant_id, id),
  CHECK (management_mode = 'agentless' OR lifecycle = 'registered' OR agent_id IS NOT NULL)
);
CREATE UNIQUE INDEX endpoints_live_name ON endpoints (tenant_id, name) WHERE lifecycle <> 'retired';
CREATE INDEX endpoints_labels ON endpoints USING gin (labels);   -- jsonb_ops: @> and ?

CREATE TABLE endpoint_revisions (
  tenant_id text NOT NULL, endpoint_id text NOT NULL,
  kind text NOT NULL CHECK (kind IN ('declared', 'facts')), revision bigint NOT NULL, actor text NOT NULL, recorded_at timestamptz NOT NULL DEFAULT now(),
  document jsonb NOT NULL,
  PRIMARY KEY (tenant_id, endpoint_id, kind, revision),
  FOREIGN KEY (tenant_id, endpoint_id) REFERENCES endpoints (tenant_id, id)
);

CREATE TABLE endpoint_relationships (
  tenant_id text NOT NULL, id text NOT NULL, type text NOT NULL,
  source_id text NOT NULL, target_id text NOT NULL, created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, id),
  FOREIGN KEY (tenant_id, source_id) REFERENCES endpoints (tenant_id, id),
  FOREIGN KEY (tenant_id, target_id) REFERENCES endpoints (tenant_id, id)
);

CREATE TABLE endpoint_events (
  sequence bigserial PRIMARY KEY, tenant_id text NOT NULL, endpoint_id text,
  type text NOT NULL, resource_version bigint, occurred_at timestamptz NOT NULL DEFAULT now()
);

ALTER TABLE endpoints ENABLE ROW LEVEL SECURITY;
ALTER TABLE endpoints FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON endpoints
  USING (tenant_id = current_setting('forge.tenant_id', true));
-- same ENABLE/FORCE/POLICY for revisions and relationships; classes also allow tenant_id IS NULL on read

-- +goose Down
DROP TABLE endpoint_events, endpoint_relationships, endpoint_revisions, endpoints, endpoint_classes;
```

- **Label selectors** compile to `labels @> '{"site":"dc1"}'` for equality and `in`, and `labels ? 'k'` for
  existence. Both use the default `jsonb_ops` GIN index; `jsonb_path_ops` lacks `?`.
- **Retention** — a background job deletes, in batches, facts revisions older than 90 days, declared
  revisions older than 400 days, events older than 7 days, and retired endpoints after 30 days.
  Partitioning waits until volume requires it.
- **Declared revisions** store full snapshots of labels and attributes (small). Facts revisions are stored
  only when the digest changes.

### Security

- **Tenant isolation, twice.** Every query runs in a transaction that begins
  `SELECT set_config('forge.tenant_id', $1, true)` from the principal. Row-level security then filters
  rows even if a query forgets its `WHERE`. `FORCE ROW LEVEL SECURITY` applies the policy to the table
  owner too. The runtime role owns no tables and lacks `BYPASSRLS`; goose runs as a separate migration
  role.
- **Principal trust.** `X-Forge-Principal` is accepted only from
  `spiffe://<environment-id>/service/forge-gateway`, and `X-Forge-Tenant-Id` only from `internalCallers`
  (default `forge-provisioner`). Agents can act only on their own endpoint.
- **Schema safety.** Tenant-supplied JSON Schemas compile with remote `$ref` loading disabled, and are
  capped at 64 KiB and depth 32. Whether jsonschema v6's default loader fetches over the network is
  unverified; the service installs an explicit loader with no network access regardless.
- **Confidential facts.** Facts are tenant data. They are never logged, never placed in metrics labels,
  and only their size and digest appear in traces.
- **Database transport.** `sslmode=verify-full` against the environment CA bundle. A plaintext database
  connection is the last-resort feature `plaintext-database`.
- **Service identity.** Mutual TLS 1.3 on the service listener, with `forge-sdk` `tlsconfig.Server` and
  revocation checks.

### Environment awareness

- **Startup** — `name`, `tier`, `id`, and `caBundle` are required, because the service accepts mutual TLS.
- **Hardened tiers** — `production` and `staging` turn off the OpenAPI UI and require a DSN with
  `sslmode=verify-full`. `AllowLastResort("plaintext-database")` refuses in `production` unless
  overridden.
- **Migrations** — `database.autoMigrate` defaults to true only in `development` and `test`. Hardened tiers
  run `forge-inventory migrate` as a separate, logged deployment step ([0005](0005-forge-infrastructure.md)).

### Logging & telemetry

- **Fields** — `forge.tenant.id`, `forge.endpoint.id`, `forge.agent.id`, `forge.report.bytes`, and
  `forge.report.unchanged`; never label values, attributes, or facts.
- **Metrics** — `forge.inventory.reports` (by result), `forge.inventory.report.size` (histogram),
  `forge.inventory.endpoints` (gauge by lifecycle, without a tenant dimension), and
  `forge.inventory.events.lag`.
- **Spans** — database spans carry the statement name, never parameters.

### Configuration

Prefix `FORGE_INVENTORY_`. Starter server, logging, environment, and telemetry keys are omitted.

| YAML                                | Variable                                         | Default               |
|-------------------------------------|--------------------------------------------------|-----------------------|
| `database.dsnFile`                  | `FORGE_INVENTORY_DATABASE_DSN_FILE`              | required              |
| `database.migrationDsnFile`         | `FORGE_INVENTORY_DATABASE_MIGRATION_DSN_FILE`    | required to migrate   |
| `database.autoMigrate`              | `FORGE_INVENTORY_DATABASE_AUTO_MIGRATE`          | by tier               |
| `reports.maxBytes`                  | `FORGE_INVENTORY_REPORTS_MAX_BYTES`              | `8MiB`                |
| `reports.interval`                  | `FORGE_INVENTORY_REPORTS_INTERVAL`               | `5m`                  |
| `autoRegistration.enabled`          | `FORGE_INVENTORY_AUTO_REGISTRATION_ENABLED`      | `true`                |
| `history.factsRetention`            | `FORGE_INVENTORY_HISTORY_FACTS_RETENTION`        | `2160h`               |
| `history.declaredRetention`         | `FORGE_INVENTORY_HISTORY_DECLARED_RETENTION`     | `9600h`               |
| `events.retention`                  | `FORGE_INVENTORY_EVENTS_RETENTION`               | `168h`                |
| `internalCallers`                   | `FORGE_INVENTORY_INTERNAL_CALLERS`               | `forge-provisioner`   |

The DSN comes from a file, per [CONVENTIONS.md](CONVENTIONS.md), which replaces the starter's inline
`database.dsn`.

### Build, release & versioning

- **Bootstrap** from `go-echo-starter`: swap bun for `pgxpool`, remove swaggo, and keep goose and the
  embedded migrations. The binary is `cmd/forge-inventory`, with `serve` and `migrate` subcommands.
- **Migrations** are forward-only in hardened tiers. Every `Down` is tested, but rollback happens by
  restoring a backup.
- **Versioning** — `v0.x`. The API follows the stages in 0002, and a schema change that breaks a stored
  class requires a data migration in the same release.

### Testing

- **Database tests** against a real PostgreSQL started by the Taskfile (container), with every migration
  applied up and down.
- **Isolation tests** — for every repository method, tenant B cannot read or change tenant A's rows, even
  through a query deliberately missing `WHERE tenant_id`.
- **Ingestion** — replayed `reportId`, out-of-order `sequence`, unchanged digest, oversized plugin data,
  auto-registration name collisions, and a principal header from a non-gateway peer.
- **Selectors** — a parser fuzz test plus table tests comparing SQL results with an in-memory evaluator.
- **Contract** — responses validated against the embedded OpenAPI document (0002's conformance approach).

## Alternatives considered

- **Keep `uptrace/bun`** (the starter's ORM) — familiar scaffolding, but 7 extra linked modules for queries
  that are mostly JSONB and locking SQL.
- **EAV tables for attributes and facts** — strongly typed columns, but schema-per-class churn and costly
  queries. JSONB with JSON Schema validation fits class-defined shapes.
- **A tenant schema or database per tenant** — stronger isolation, but migrations multiply with tenants.
  Row-level security plus code filters is proposed first.
- **`/tenants/{tenantId}` in paths** — explicit, but duplicates the principal's tenant, and a mismatch
  becomes a new bug class (0002 open question).
- **A message broker** (NATS, Kafka) for the change stream — push delivery, but a new stateful system and
  client modules. An outbox table with long-polling is enough for one consumer.
- **Agent-computed digests over canonical JSON** ([RFC 8785](https://www.rfc-editor.org/rfc/rfc8785)) —
  lets agents skip uploads without a server round trip, but every agent and plugin must canonicalize
  identically.
- **Storing reconciliation status** on endpoints — a single view for operators, but it blurs the single
  desired-state authority that 0001 assigns to `forge-provisioner`.

## Open questions

- **Pre-registration** — should operators be able to claim an agent onto a declared endpoint, e.g. an
  enrollment token that names an `endpointId` ([0006](0006-forge-identity.md))?
- **Retirement** — should retiring an agent-managed endpoint revoke the agent's certificate automatically?
- **Agentless credentials** — where do device API credentials live: `forge-provisioner`, an external
  secret manager, or `forge-identity`?
- **Compliance view** — where do operators see reconciliation status ([0011](0011-forge-provisioner.md))?
- **Cross-tenant operators** — can an environment administrator query across tenants, and how does that
  interact with row-level security?
- **PostgreSQL floor** — 16 is proposed; which versions will [0005](0005-forge-infrastructure.md) operate?
- **Fact scrubbing** — who strips secrets that plugins might report, the agent ([0012](0012-forge-agent.md))
  or inventory?
- **Retention defaults** — are 90 days of facts history and 400 days of declared history right?

## References

- [0001 — Project Repositories](0001-project-repositories.md) — inventory role, desired-state model,
  agent enrollment, environment identity.
- [0002 — forge-api-schema](0002-forge-api-schema.md), [0003 — forge-sdk](0003-forge-sdk.md),
  [0004 — forge-common](0004-forge-common.md), [0008 — forge-gateway](0008-forge-gateway.md),
  [CONVENTIONS.md](CONVENTIONS.md).
- [go-echo-starter](https://github.com/servercurio/go-echo-starter) — `internal/database` (pgx stdlib, bun,
  goose embedded migrations) and `go.mod`.
- [pgx](https://github.com/jackc/pgx), [goose](https://github.com/pressly/goose),
  [bun](https://github.com/uptrace/bun), and
  [santhosh-tekuri/jsonschema](https://github.com/santhosh-tekuri/jsonschema).
- [PostgreSQL row security policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) —
  `FORCE ROW LEVEL SECURITY` and owner bypass.
- [PostgreSQL JSON types, jsonb indexing](https://www.postgresql.org/docs/current/datatype-json.html#JSON-INDEXING)
  — `jsonb_ops` versus `jsonb_path_ops`.
- [PostgreSQL `LISTEN`](https://www.postgresql.org/docs/current/sql-listen.html) and
  [versioning policy](https://www.postgresql.org/support/versioning/).
- [Kubernetes labels and selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
  — label syntax and set-based selectors.
- [JSON Schema 2020-12](https://json-schema.org/draft/2020-12).
- [RFC 7396](https://www.rfc-editor.org/rfc/rfc7396) — JSON Merge Patch;
  [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110) — `ETag`, `If-Match`, `412`;
  [RFC 8785](https://www.rfc-editor.org/rfc/rfc8785) — JSON canonicalization;
  [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) — problem details.
