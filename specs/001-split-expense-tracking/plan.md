# Implementation Plan: SplitSmart Baseline Architecture

**Consolidation 2026-11-08:** The owner selected Architecture A. The [consolidated architecture](../../docs/architecture-consolidated/architecture.md) is the proposed detailed design record, with [ADR/change rationale](../../docs/architecture-consolidated/architecture-decision-record.md), [traceability](../../docs/architecture-consolidated/requirements-traceability.md), and [clarification gates](../../docs/architecture-consolidated/requirements-clarification-log.md). Original A/B documentation remains evaluation snapshots. Selection approval does not approve unresolved product policies or verify implementation. Retain one local Swing/SQLite process, one Maven project and the append-only posting ledger; no centralized MVP is introduced.

**Branch**: `001-split-expense-tracking` | **Date**: 2026-11-08 | **Spec**: [spec.md](./spec.md)

## Summary

Define a Java 21 / Maven, modular-monolith architecture for an interim Java Swing desktop MVP. Keep Swing as a thin adapter over UI-independent application use cases and domain rules, with local SQLite persistence behind repository ports. Preserve the full customer feature set and design the domain/application boundaries so a later server API can support web, Android, and iOS clients.

The full product's customer-specified primary platform remains web (CR-009). Android and iOS are included here as future client platforms per this planning request; they are not requirements stated in `UserRequirements.md`. The Swing pilot is an interim desktop deliverable, not the web product and not a substitute for any future client.

## Technical Context

**Language/Version**: Java 21; configure Maven Compiler Plugin with `release` 21.  
**Primary Dependencies**: Java Swing and JDBC from the JDK; Maven; JUnit 5 Jupiter; a pinned SQLite JDBC driver for the local pilot. Avoid a UI framework or dependency-injection framework in the baseline.  
**Storage**: Local SQLite database for the single-device desktop pilot, with repositories isolating SQL. A shared server-side database (e.g. PostgreSQL) is a future deployment decision, not part of this pilot.  
**Testing**: JUnit 5 domain and application tests; SQLite repository integration tests; limited Swing wiring/manual acceptance tests.  
**Target Platform**: Interim Java 21 desktop pilot. Full-product target: web, with future Android and iOS clients using a server API.  
**Project Type**: Single Maven desktop application, structured as a modular monolith.  
**Performance Goals**: Preserve NFR-005: for groups up to 100 members and 10,000 expenses, at least 95% of balance and date-filtered report requests display within 3 seconds under normal operating conditions. Define the measurement environment before performance acceptance.  
**Constraints**: `BigDecimal` for all money and percentage calculations; explicit currency minor-unit scale and rounding; no balance impact before all required approvals; retain audit history and prevent duplicate balance effects; enforce access boundaries. One currency per group. No transfers initiated by the application.  
**Scale/Scope**: Desktop MVP is a local pilot; no user-count target is stated for it. The full-product performance test volume is 100 members and 10,000 expenses per group. Do not introduce distributed services before deployment needs justify them.

### Scope and traceability

The architecture retains all customer capabilities in the specification's CR-001–CR-009 inventory: accounts, groups, expenses and splits, notifications, approval, balances, payments, reports, and a primary web product. It also retains FR-001–FR-015 and their access, audit, performance, and history constraints.

The proposed Swing pilot subset is limited to account access, group management, expense entry, one authorized review step, pairwise balances, and manual payment recording. Notifications, configurable reports, and multi-step approval remain full-product capabilities, not delivered Swing-MVP capabilities. The pilot also lacks cross-device synchronization and external email delivery for verification/reset; those require a shared service. These are delivery boundaries, not deletions or revisions of the customer requirements.

## System architecture overview

Use a layered modular monolith with explicit inward dependencies:

1. **Presentation adapters**: Swing views and presenters/controllers translate user actions and display results. Event handlers call application use cases; they contain no expense, approval, or balance rules.
2. **Application services**: use cases authorize and coordinate operations such as submitting/reviewing an expense, recording a payment, and querying balances/reports. They define transaction boundaries and depend on repository/notification/receipt ports.
3. **Domain**: focused user, group, expense, approval, payment, allocation, and balance types enforce business invariants. This layer has no Swing, database, network, or filesystem dependency.
4. **Infrastructure adapters**: SQLite repositories, receipt storage, and (in the full product) notification delivery implement ports. The pilot uses local storage; it does not expose a production network API.

For the full product, web, Android, and iOS clients communicate with a server-side application through a versioned HTTPS API. The server enforces authentication and authorization and uses server-owned persistence and notification adapters. Client-specific UI technologies are intentionally undecided. The API boundary and reuse of domain rules are shown in [contracts/platform-boundaries.md](./contracts/platform-boundaries.md).

### Architecture diagram

The maintained PlantUML source is [architecture.puml](./architecture.puml). Solid components represent the interim pilot; the future server/client architecture is shown separately.

## Technology choices and design decisions

| Decision | Rationale | Tradeoff / alternative |
|---|---|---|
| Java 21 and Maven | Required project baseline; standard Maven layout and `maven.compiler.release=21` make the language/API baseline reproducible. | A multi-module build is deferred until a deployable server or separately versioned shared library exists. |
| Swing for the pilot | Required desktop MVP technology; keeps the prototype usable without choosing a web/mobile UI stack prematurely. | Swing does not meet the primary web platform requirement and cannot provide cross-device activity by itself. |
| Modular monolith and application ports | Keeps UI, domain rules, and persistence separate while avoiding premature service decomposition. The Swing adapter invokes use cases in-process. | A later server/API deployment will add transport, authentication, and server persistence adapters; this is not supplied by the desktop pilot. |
| SQLite for the pilot | Minimal local relational persistence for a single-device pilot and straightforward repository integration tests. | A local database is not a shared multi-user service, has limited concurrent-writer suitability, and does not provide production identity/security by itself. A future centralized database is a separate decision. |
| `BigDecimal` plus per-currency scale | Avoids binary floating-point error and makes allocations reconcile at the currency's smallest unit. | Rounding and percentage-total policies must be explicit; percentages are accepted only at exactly 100% until the open tolerance question is decided. |
| Append-only balance postings from approved expenses and payments | Makes balance impact auditable and supports idempotent posting; pairwise balances are a query/projection, not independently edited values. | A cached/materialized balance may be added only if measured performance requires it and it can be rebuilt from source records. |
| JUnit 5 Jupiter | Required test framework; supports isolated domain tests and Maven execution. | UI-level automation is limited; business correctness is verified below the Swing layer. |
| Future web/mobile clients via API | Keeps clients independent of desktop UI and lets the server enforce shared rules and access controls. | API format, authentication protocol, web framework, native client stacks, hosting, and delivery schedule remain future decisions. Do not assume Swing code runs on Android or iOS. |

### Java Swing MVP architecture

The desktop entry point constructs the application services, repository adapters, and Swing presenter/view composition. Views/presenters marshal Swing component work to the Event Dispatch Thread (EDT); database and other potentially slow operations must not block it. Results and errors are returned to the UI for explicit display. The flow is:

`Swing event -> presenter/controller -> application use case -> domain operation -> repository port -> SQLite adapter`

In the pilot, a single local database holds the pilot's users and group activity. It supports a desktop demonstration of group workflows but is not a remotely shared account service. Notifications, reporting workflows, multi-step approval, email verification/reset delivery, and cross-device access are outside the pilot boundary, while remaining in the full-product architecture.

### Expense splitting and balances

- Represent amounts, percentages, allocations, payments, and balance postings with `BigDecimal`; never convert through `float` or `double`. Parse decimal input from text and use numeric comparison (`compareTo`) for value comparisons.
- Associate each group with one currency and its minor-unit scale. Reject non-positive expenses/payments and expense amounts with excess fractional precision; the spec sets no maximum amount.
- Equal split uses selected participants, rounds each preliminary share down to the currency scale, and assigns the entire remainder to the payer, as specified by the feature clarifications.
- Percentage split validation retains the FR-004 100% baseline; tolerance remains unresolved. Rounding down with residual to payer was an unapproved proposal, not a selected product rule. Record residual-to-payer and largest-remainder options for human decision; block affected rounded-percentage implementation until approved, while retaining percentage capability in scope.
- Store the submitted expense and per-participant allocation. No balance posting occurs while pending or rejected. Once all required approvals are recorded, atomically post one immutable entry for each positive non-payer participant share owed to the payer. Skip zero shares. The payer's own share is recorded for reconciliation but is not a self-debt.
- A recorded payment posts the opposite-direction pairwise entry. A pair's displayed net balance is derived from its postings. Enforce unique source references/transaction boundaries so retries cannot apply the same expense or payment twice. The policy for payments exceeding an outstanding balance remains open.

Persist monetary values as normalized decimal TEXT and reconstruct/aggregate with Java BigDecimal, never SQLite REAL or floating-point SQL aggregation. This avoids an artificial fixed minor-unit integer cap. Posting from X to Y increases X's debt to Y; payment P to Q therefore posts Q to P. Use stable `(sourceType, sourceId, sourceLineKey)` uniqueness, per-operation keys/fingerprints and terminal-state guards. Save final approval/postings and payment/postings/audit/result in their respective single SQLite transactions; failures roll back completely. Proposed write reservation/driver handling, exact mapping, membership periods, receipt recovery and reconciliation are detailed in the [consolidated data model](../../docs/architecture-consolidated/data-model.md) and require implementation verification.

## Testing strategy

Use JUnit 5 with the standard Maven test lifecycle. Keep core behavior testable without constructing Swing components:

- **Domain unit tests**: exact/equal and percentage allocations, remainder reconciliation, currency precision, invalid amounts, participant/payer constraints, approval state transitions, and pairwise balance arithmetic.
- **Application use-case tests**: authorization, no balance effects before complete approval, authorized reviewer rules, payment recording/idempotency, and report access/filter behavior, using in-memory fakes where useful.
- **Persistence integration tests**: SQLite schema/repository behavior, transaction atomicity for approval/posting, uniqueness/idempotency, and historical records after membership removal.
- **Presentation checks**: focused Swing wiring/validation tests where practical, plus manual acceptance flows for account/group/expense/review/payment. Keep slow work off the EDT and test error display.
- **Requirement/performance verification**: map tests to FR/AC IDs; measure NFR-005 at the stated group/data volume on a recorded environment. Do not claim acceptance until the measured threshold is demonstrated.

Runnable setup and scenario guidance is in [quickstart.md](./quickstart.md).

## Security, scalability, and limitations

- **Security**: Treat Swing input as untrusted; enforce authorization in application use cases, not by hiding controls. Store only salted adaptive password hashes (never plaintext), and avoid putting credentials or database secrets in logs. The pilot's local SQLite file relies on host-account/filesystem protection; it is not a production security boundary. The full product needs HTTPS, server-side authorization on every group/resource access, secure password reset and email verification, secret management, protected receipt storage, audit controls, and a threat model. The email delivery and authentication protocols are not selected here.
- **Privacy and audit**: Scope every group query to active membership and preserve historical records. Keep expense/review/payment actor and timestamp data; correct balance-affecting records through auditable compensating records rather than silent mutation/deletion.
- **Scalability**: A modular monolith, indexed relational queries, transactional posting, and derived balances are appropriate for the stated baseline. Validate NFR-005 with realistic seeded data. If measured queries miss the threshold, optimize indexes/projections before considering service decomposition. SQLite is pilot-only; centralized concurrent web/mobile use requires server-side persistence.
- **Limitations/open decisions**: Original customer requirements state web as primary; this plan also records Android/iOS as requested future clients, but defines no delivery commitment or client technology. Notification channels/timing, overpayment handling, percentage tolerance, report filter set, and allowed historical/future dates remain open in [spec.md](./spec.md). Multi-currency conversion, initiating transfers, and notification/email delivery are not added as requirements.

## Constitution Check

The following PASS labels assess design intent only, not implemented/tested compliance. The current detailed [constitution assessment](../../docs/architecture-consolidated/constitution-compliance.md) distinguishes design alignment, unresolved percentage policy, unavailable execution evidence and required human review.

### Pre-design gate

- **Customer traceability**: PASS — keeps CR-001–CR-009 distinct from MVP and architecture decisions; retains the source requirement that web is primary.
- **Java 21 and Maven**: PASS — explicitly set as the build baseline.
- **Swing/business-logic separation**: PASS — Swing calls use cases; rules live in the independent domain/application layers.
- **Object-oriented design**: PASS — domain ownership and focused adapters are explicit; no microservices or unnecessary framework layers proposed.
- **Exact monetary values**: PASS — `BigDecimal`, currency scale, and remainder handling are specified.
- **JUnit 5**: PASS — strategy covers required approval, balance, payment, and report behavior.
- **PlantUML**: PASS — architecture has a maintained `.puml` source.
- **Human review**: PASS WITH REVIEW REQUIRED — this is a proposed baseline; an owner must review design and requirement alignment before implementation acceptance.
- **Open requirements**: PASS WITH IMPLEMENTATION GATES — known spec questions are carried forward explicitly; they are not silently assumed. Resolve them before implementing the affected policies.

### Post-design gate

PASS with the same review and implementation gates. Design artifacts preserve the Swing/web distinction, include the requested future platform boundary, and leave unresolved product policies visible. No constitution exception is requested.

## Project Structure

### Documentation (this feature)

```text
specs/001-split-expense-tracking/
├── plan.md
├── research.md
├── architecture.puml
├── data-model.md
├── quickstart.md
└── contracts/
    └── platform-boundaries.md
```

### Source Code (planned; not created by this planning task)

```text
src/
├── main/
│   ├── java/com/splitsmart/
│   │   ├── ui/swing/             # views, presenters, application composition
│   │   ├── application/          # use cases, authorization, ports
│   │   ├── domain/               # users, groups, expenses, approvals, balances
│   │   ├── persistence/sqlite/   # pilot repository adapters
│   │   └── receipt/              # receipt storage adapter
│   └── resources/db/migration/
└── test/
    └── java/com/splitsmart/
        ├── domain/
        ├── application/
        └── persistence/sqlite/
```

**Structure Decision**: One Maven project and one deployment unit for the Swing pilot. Keep package boundaries suitable for later server/API adapters, but do not create web, Android, iOS, or multi-module projects before their implementation scope and technology choices are approved.

## Complexity Tracking

No constitution violations or exceptions are proposed.
