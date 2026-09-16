<!--
  ~ SPDX-License-Identifier: Apache-2.0
-->

# Design documents

Numbered design documents and RFCs for Forge. Each proposes a decision, records the reasoning and the
alternatives considered, and tracks its status as the decision matures.

## Index

| #                                      | Title                  | Status | Date       |
|----------------------------------------|------------------------|--------|------------|
| [0001](0001-project-repositories.md)   | Project Repositories   | Draft  | 2026-07-11 |
| [0002](0002-forge-api-schema.md)       | forge-api-schema       | Draft  | 2026-09-15 |
| [0003](0003-forge-sdk.md)              | forge-sdk              | Draft  | 2026-09-15 |
| [0004](0004-forge-common.md)           | forge-common           | Draft  | 2026-09-15 |
| [0005](0005-forge-infrastructure.md)   | forge-infrastructure   | Draft  | 2026-09-15 |
| [0006](0006-forge-identity.md)         | forge-identity         | Draft  | 2026-09-15 |
| [0007](0007-forge-sso.md)              | forge-sso              | Draft  | 2026-09-15 |
| [0008](0008-forge-gateway.md)          | forge-gateway          | Draft  | 2026-09-15 |
| [0009](0009-forge-inventory.md)        | forge-inventory        | Draft  | 2026-09-15 |
| [0010](0010-forge-cli.md)              | forge-cli              | Draft  | 2026-09-15 |
| [0011](0011-forge-provisioner.md)      | forge-provisioner      | Draft  | 2026-09-15 |
| [0012](0012-forge-agent.md)            | forge-agent            | Draft  | 2026-09-15 |
| [0013](0013-forge-agent-plugin-sdk.md) | forge-agent-plugin-sdk | Draft  | 2026-09-15 |
| [0014](0014-forge-agent-plugins.md)    | forge-agent-plugins    | Draft  | 2026-09-15 |
| [0015](0015-forge-plugin-starter.md)   | forge-plugin-starter   | Draft  | 2026-09-15 |
| [0016](0016-web-ui-architecture.md)    | Web UI architecture    | Draft  | 2026-09-16 |
| [0017](0017-forge-portal.md)           | forge-portal           | Draft  | 2026-09-16 |
| [0018](0018-forge-console.md)          | forge-console          | Draft  | 2026-09-16 |
| [0019](0019-brand-identity.md)         | Brand identity         | Draft  | 2026-09-16 |

## Statuses

- **Draft** — open for discussion; expect changes.
- **Accepted** — the decision stands; later changes need a new document or an explicit revision.
- **Superseded** — replaced by a later document; link it (`Superseded by [NNNN](NNNN-slug.md)`).
- **Withdrawn** — abandoned without adoption; kept for the record.

## Conventions

[`CONVENTIONS.md`](CONVENTIONS.md) collects the cross-cutting conventions — API style, SPIFFE IDs,
configuration, logging fields, and dependency rules — that the per-repository documents share.

## Adding a design doc

1. Pick the next unused number and copy [`TEMPLATE.md`](TEMPLATE.md) to `NNNN-<kebab-slug>.md`.
2. Fill in the header and sections, starting at **Draft**.
3. Add a row to the index above in the same pull request, using a `docs:` commit subject.
4. When the status changes, update both the document header and its index row.
