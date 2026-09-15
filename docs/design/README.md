<!--
  ~ SPDX-License-Identifier: Apache-2.0
-->

# Design documents

Numbered design documents and RFCs for Forge. Each proposes a decision, records the reasoning and the
alternatives considered, and tracks its status as the decision matures.

## Index

| #                                    | Title                | Status | Date       |
|--------------------------------------|----------------------|--------|------------|
| [0001](0001-project-repositories.md) | Project Repositories | Draft  | 2026-07-11 |
| [0002](0002-forge-api-schema.md)     | forge-api-schema     | Draft  | 2026-09-15 |
| [0003](0003-forge-sdk.md)            | forge-sdk            | Draft  | 2026-09-15 |

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
