# Selected Architecture A: future platform migration

Architecture A remains a local Swing/SQLite pilot. This document plans evolution, not an API/server implementation or a claim that CR-009 is satisfied. [ADR](./architecture-decision-record.md) records the approved selection; [architecture](./architecture.md) defines current authority. Android/iOS are optional future planning clients.

## Reuse and replacement boundaries

| Preserve and reuse | Replace/add for centralized product |
|---|---|
| Money, exact allocation rules, approval behavior, posting direction/netting, reconciliation and immutable source semantics | Server persistence adapters, central database transactions/locking, migration tooling |
| UI-independent SubmitExpense, ReviewExpense, RecordPayment, Group and query services with repository ports | HTTPS presentation adapter, validation/error DTOs, sessions and authenticated actor provider |
| Stable IDs, source-line keys, operation fingerprints, policy snapshots, historical membership/actor/time | Cross-installation identity linking, operation-key namespace/import ownership, server secret handling |
| Receipt storage interface and authorized use-case boundary | Protected server directory/object-store adapter and scoped download/upload |
| Domain/application JUnit 5 acceptance scenarios | Server repository/API/browser/load/security/integration suites |

Swing widgets/presenters, EDT/background execution, local file chooser and local filesystem paths stay outside reusable domain code. Native clients reuse server behavior through an API, not Swing or desktop JVM assumptions. Platform frameworks, central database choice and hosting remain future decisions; this plan does not adopt B's centralized pilot or source-only balances.

## Identity and contract transition

Future server authenticates accounts and rechecks membership/roles on every operation. A client cannot assert its actor/approval status or calculated balance. Introduce revocable server sessions, HTTPS, verified email and secure reset tokens with expiry/single use, protected token hashes, redacted logs and rate controls. Do not promote pilot demo verification flags to verified server identities. Require explicit account linking/reverification; never merge identities solely because local email text matches.

Define a versioned API such as `/api/v1` exposing application commands/queries, decimal-string money/percentages, currency codes, stable IDs, LocalDate business dates and UTC audit instants. Include stable error/field codes, authorization failures, expected versions and persistent mutation keys. DTO schemas and session transport need review. Use compatible contract changes within a version and an explicit deprecation path; test old clients before rollout. Final approval/source postings and payment/posting writes remain one server transaction, preserving A's ledger model.

## Controlled data migration

1. Inventory installations and owners; confirm import consent/authorization, central tenancy/group mapping, supported source schema and migration version. Back up SQLite, WAL/journal as appropriate for a consistent snapshot, receipts and checksums. Test restore before cutover.
2. Choose a bounded write freeze/export window. Do not allow simultaneous desktop and server writes to the same logical group; offline synchronization is separate unapproved scope. For a pilot with disposable data, explicitly choose fresh initialization rather than silently importing untrusted records.
3. Export normalized accounts, group/membership periods, policies/steps, expenses/shares, decisions, payments, postings, corrections if present, operation records/audit and receipt manifest. Preserve IDs or record deterministic ID mappings and import-batch namespaces for collisions.
4. Validate currency precision, original actors/timestamps, source keys, group references and policy completion. Link accounts through a reviewed mapping; require email verification/reset where trust/hash compatibility is insufficient. Do not import demo tokens/sessions or silently lose audit identity.
5. Import into staging in referential order with repeatable batch identifiers/checkpoints. Reconcile approved source expectations with imported postings and pairwise totals using BigDecimal. Compare counts, exact monetary totals per currency, unique keys, membership periods and receipt checksums. Quarantine invalid sources rather than silently fixing/deleting financial history.
6. Copy receipt bytes to private storage, validate content/size under approved policy, preserve authorized links/checksums, and ensure local paths are absent from API payloads. Reconcile missing/orphaned files before release or explicitly record exceptions for review.
7. Run dry-run acceptance/security/import tests, then final frozen import and reconciliation. Cut clients to server authority and keep the old pilot read-only. A usable browser UI must pass AC-010 before claiming the primary web requirement delivered.

## Recovery and rollout risks

Record backup owner, recovery objectives, restore procedure, schema version and import reconciliation signoff. Before server writes begin, failed cutover can restore the local snapshot. After server accepts writes, reverting clients to the stale local file would lose new activity; recovery must preserve/replay server records under a separately tested procedure. Prefer controlled fix-forward or server restore with explicit reconciliation, not automatic dual writes.

Risks include identity/email collisions, incompatible password hashes, duplicate imports, altered timestamps, late approvals, receipt leakage/loss and operation-key collisions. Migration is not only swapping repository classes. Central adapters require realistic transaction/concurrency tests; server-only authorization requires a threat model, operational monitoring and backup drills. Deferred reports/notices/multi-step approval can be developed around this transition, but production identity/delivery acceptance waits for the required shared infrastructure. See [roadmap](./implementation-roadmap.md) phases 9–10.
