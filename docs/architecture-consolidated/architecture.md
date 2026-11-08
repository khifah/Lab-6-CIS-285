# SplitSmart consolidated architecture

**Selected foundation: Architecture A (Spec Kit).** Selection is owner-approved; this detailed consolidation is proposed for review. Documentation only, 2026-11-08. No application capability or test pass is claimed.

## Purpose, authority and scope

Build a manageable local desktop pilot while preserving the whole customer product. Authority flows from [original customer requirements](../../UserRequirements.md), [inventory](../../requirements.md), [feature specification](../../specs/001-split-expense-tracking/spec.md), and [constitution](../../.specify/memory/constitution.md). The [selection ADR](./architecture-decision-record.md) records rationale and canonical changes. Original A/B directories are historical alternatives; canonical Spec Kit planning artifacts and this proposed consolidated record describe the selected design.

Three categories stay distinct: CR-001–CR-009 are original customer outcomes; verified-email login, payer participation, direct membership addition and equal-split payer remainder are recorded specification clarifications; Java/Maven/Swing/SQLite/layering/BigDecimal/JUnit/PlantUML are implementation decisions. Future Android/iOS are planning requests, not original customer requirements.

| Delivery boundary | Retained capabilities |
|---|---|
| Initial Swing MVP | Local account access; group management; expenses with supported splits/optional receipts; one authorized reviewer; pairwise balances; manual payment recording |
| Full product / later phases | Complete verification/reset delivery; notifications; configurable multi-step approvals; configurable expense/balance/payment reports; centralized multi-user access; primary web UI; possible Android/iOS |

The local demonstration cannot meet full FR-001/AC-001 without real email verification/reset. Any demo identity/verification mechanism must be labeled local/demo and never bypass production verification implicitly. The MVP does not satisfy CR-009/FR-014/AC-010. Percentage residual allocation, overpayments and other open policies gate their affected implementation; they do not remove requirements.

## Technology and deployment

Java 21, compiler release 21, one Maven project, standard source/test layout; Swing views and presenters/controllers; JDBC with a pinned SQLite driver; protected local receipt directory; BigDecimal financial calculations; JUnit 5 Jupiter through Maven; maintained PlantUML. Pin compatible dependencies/plugins during implementation, not by assuming a validated stack here. There is no pilot API, application server, PostgreSQL requirement, service mesh or multi-module build.

One desktop process owns one local database and managed receipts. Accounts/group interactions can be demonstrated on that installation but are not remotely shared accounts. Multiple processes opening the file are not a supported collaboration model; adapter tests should nevertheless exercise competing SQLite connections for integrity. Slow database/file work stays off Swing's Event Dispatch Thread (EDT); views update on it.

## Layers and dependency rules

| Layer / proposed package | Responsibilities |
|---|---|
| Presentation `ui.swing` | Views, presenters/controllers, input text, validation feedback, visible status, background work and EDT updates. Calls application use cases only. |
| Application `application` | Authenticate local principal, authorize current access/roles, coordinate commands/queries, own UnitOfWork and repository/receipt/delivery ports, idempotency and transaction boundaries. |
| Domain `domain` | Encapsulated entities, Money, allocation strategies, policy state transitions, posting creation/netting and reconciliation rules. No Swing, JDBC, filesystem or framework imports. |
| Infrastructure `persistence.sqlite`, `receipt`, later external adapters | Implement application ports, exact persistence mapping, migrations, transactions and protected receipt access. No authorization bypass through presentation. |

Compile-time dependencies: presentation → application → domain; infrastructure → application ports/domain. A small desktop composition root wires concrete adapters. Runtime application calls repository interfaces; inward dependency direction is not the same as call direction. Feature packages live within these layers, not separate deployed services. Avoid cyclic feature calls, giant service classes, or business logic hidden in SQL/UI callbacks.

## Feature ownership

| Module | State/behavior owned and collaborators |
|---|---|
| Accounts | UserAccount and local authentication; private profile queries. Future session/verification/reset authority moves to server. |
| Groups | Group, Membership periods, roles/current membership; authorizes group visibility and manager changes. |
| Expenses | Expense, ExpenseShare, allocation and receipt reference. Coordinates snapshot of approval policy through application service. |
| Approvals | ApprovalPolicy/Step and ReviewDecision; authorizes ordered decisions; one step initially. Final approval coordinates Expense and BalancePosting in one UnitOfWork. |
| Notifications, deferred | Durable notices to other eligible members at expense creation; delivery retries/permissions; current expense status remains visible even if delivery fails. |
| Balances | Append-only BalancePosting semantics, pairwise netting, source reconciliation; no editable balance value. |
| Payments | Immutable settlement records and compensating postings; only involved authorized members record payments. No transfer initiation. |
| Reporting, deferred | Permitted expense/balance/payment summaries, dates/filters and zero totals; financial balances always use posting rules. |

All modules share a local relational database with explicit table ownership. Cross-module writes occur through coordinated application transactions and focused domain behavior, not arbitrary repository access from another UI feature. Small shared Money/ID types are acceptable.

## Authorization and security

Every use case obtains the authenticated actor; no caller-supplied user ID grants authority. Managers create/manage groups and directly add registered users; members submit expenses/view group history; only configured active reviewers decide; payment recorder must be involved. Non-members cannot inspect balances, receipts, notices or reports. Removal revokes current group access without deleting records. Recheck permissions within the financial transaction to avoid a change between authorization and persistence.

Adaptive salted password hashing is required; select a maintained library/algorithm and work factor during security review. Never store plaintext, log credentials or expose hashes in views. Protect database, journal/WAL, backups and receipt directory using host-account permissions. SQLite file permissions cannot stop the same host user or administrator from inspecting/modifying data; the pilot is not a production security boundary. No encryption-at-rest claim is made.

Receipt adapters validate decoded content, allowed type and size; proposed JPEG/PNG and a configurable limit require review before acceptance. Use generated storage keys, canonical managed paths and authorized retrieval, never a user-supplied path. Stage files, commit metadata/link atomically in SQLite, then clean orphaned files; filesystem writes and database transactions are not one atomic resource. Expose and recover staged/missing-file failures. See [data model](./data-model.md).

Future server sessions, HTTPS, verification/reset token expiry/single use, protected token hashes, rate limits, secret management and browser/mobile credential handling require a separate security design and tests. A simulated email flow cannot satisfy complete account acceptance.

## Financial rules and transactions

Money wraps BigDecimal and currency. Parse decimal strings, never float/double; validate positivity and currency precision before normalization. Numeric equality uses compareTo. One currency per group; no conversion or artificial maximum business amount. Payer is among active selected participants. Equal split rounds preliminary shares DOWN to minor-unit scale and assigns the whole residual to the payer; all shares must reconcile. Percentage inputs must meet existing spec validation, while tolerance/residual options remain explicitly unresolved in [clarification log](./requirements-clarification-log.md).

Use A's append-only ledger. A posting `fromUserId → toUserId` means an increase of debt from the first user to the second. Approved positive non-payer shares post participant → expense payer. Payment payer P to payee Q posts Q → P, reducing P's debt to Q when opposite directions are netted. Zero shares create no postings; no self-debt. [data-model.md](./data-model.md) defines formula, keys and correction semantics.

The SQLite UnitOfWork uses one connection per transaction with foreign keys enabled. A proposed `BEGIN IMMEDIATE` write protocol acquires the write reservation before checking state, active membership and operation key. This serializes competing writers; use bounded busy handling, no locks held while awaiting user input, and surface retryable contention. Exact JDBC begin/commit behavior must be integration-tested with the selected driver; avoid nested transactions.

- Submission: claim operation key/fingerprint; validate membership, details, shares and snapshot policy; persist PENDING expense/shares/audit/result in one transaction; no postings.
- Review: claim key; recheck authorized actor/current status; record decision; if rejected retain REJECTED with no postings; if incomplete retain PENDING; final approval atomically saves APPROVED and all expected postings with audit/result. Failure rolls back every change.
- Payment: claim key; validate parties/amount and resolved overpayment policy; save Payment, reverse-direction posting, audit and operation result atomically. Unknown commit outcome is retried with the same key.
- Existing key with same payload returns the saved result; changed payload is a conflict. Terminal review actions cannot create another decision/posting even under a new request key. Uniqueness complements state validation; it is not a substitute for it.

Corrections preserve originals through authorized compensating records/postings; workflow/authority is not yet approved, so no pilot edit/delete shortcut is permitted. Reconcile sources and ledger using BigDecimal in a consistent read snapshot. Do not silently repair discrepancies or switch to B's source-only financial view.

## Persistence, future migration and limits

Persist normalized exact decimal strings in SQLite TEXT columns; never SQLite REAL or SQL floating-point SUM for money. Reconstruct BigDecimal and aggregate in Java. This avoids signed-64-bit minor-unit limits while retaining exact scale; storage/runtime resources remain finite and failures explicit. Proposed indexes target group/status/date, source keys and participants; measure before adding rebuildable caches.

[Migration strategy](./migration-strategy.md) reuses domain rules and UI-independent use cases, replaces SQLite/receipt adapters, moves authentication to server authority, exposes versioned HTTPS DTOs, migrates source and posting records together, and adds a usable primary web UI. Swing widgets, EDT code and local paths remain client-specific. This is future work, not a centralized pilot.

Local-only authority, finite resources, unresolved financial policies and a later operational migration are accepted tradeoffs. [Roadmap](./implementation-roadmap.md), [testing](./testing-strategy.md), [traceability](./requirements-traceability.md), and [constitution assessment](./constitution-compliance.md) define the handoff. No implementation, performance, usability or security acceptance is asserted.

## Maintained diagrams

- [Components and dependencies](./diagrams/component-diagram.puml)
- [Business classes](./diagrams/class-diagram.puml)
- [Logical database ERD](./diagrams/database-erd.puml)
- [Expense approval transaction](./diagrams/expense-approval-sequence.puml)
- [Payment transaction](./diagrams/payment-sequence.puml)
- [Pilot and future deployment](./diagrams/deployment-diagram.puml)
