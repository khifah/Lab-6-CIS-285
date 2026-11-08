# Architecture B: Centralized layered application with a Swing client

**Status:** Proposed design for human review, 2026-11-08. Documentation only; no implementation or verified product capabilities are claimed.

## Basis and scope

This alternative follows [UserRequirements.md](../../UserRequirements.md), the [customer inventory](../../requirements.md), [feature specification](../../specs/001-split-expense-tracking/spec.md), and [constitution](../../.specify/memory/constitution.md). Architecture A was reviewed through its [plan](../architecture-A-speckit/plan.md), [diagram](../architecture-A-speckit/architecture.puml), [data model](../architecture-A-speckit/data-model.md), [research](../architecture-A-speckit/research.md), [quickstart](../architecture-A-speckit/quickstart.md), and [platform contract](../architecture-A-speckit/contracts/platform-boundaries.md).

The customer capabilities remain CR-001–CR-009. Java 21, Maven, Swing, JUnit 5, layering, and BigDecimal are engineering constraints. Android and iOS are future platforms requested for this design, not original customer requirements. Swing remains an interim desktop pilot: neither its network connection nor the presence of an HTTP API satisfies CR-009/FR-014. A usable primary web product must follow.

Keep the specification's pilot feature boundary: accounts, groups, expense entry and receipts, single-reviewer approval, balances, and manual payment recording. Notification delivery, configurable reporting, and multi-step approval are full-product work. Account verification/reset delivery remains an explicit integration gate: a simulated email adapter supports demonstrations only and does not satisfy full FR-001/AC-001.

## Meaningful alternative

Architecture A starts with one desktop process, one Maven project, local SQLite, and in-process use-case calls. B starts with **two application processes**, a thin Swing client and a centrally deployed Java application server, backed by PostgreSQL. All authoritative rules and data live on the server from the pilot onward. This adds deployment and network costs deliberately to avoid a later move from local authority to shared authority.

B remains a modular monolith on the server, not a set of microservices. It uses Maven artifacts to enforce layer boundaries, a defined transport boundary immediately, and balances calculated from approved source records rather than a second balance-posting ledger. These are changes to deployment, build structure, contracts, and persistence semantics, not renamed packages.

## Layers and dependencies

| Layer / proposed Maven artifact | Responsibilities and allowed dependencies |
|---|---|
| Presentation: `splitsmart-desktop` | Swing views and presenters, field feedback, session handling, asynchronous HTTP requests. Depends on `api-contract`; never on repositories or server domain classes. |
| Presentation: `splitsmart-http` | Versioned request mapping, session authentication, error translation. Depends on application and API contracts. Never implements allocation rules. |
| Application: `splitsmart-application` | Account/group/expense/review/payment commands and balance/report queries; current-membership authorization; transactions and idempotency. Depends on domain; owns persistence, email, receipt, and notification ports. |
| Domain: `splitsmart-domain` | Encapsulated entities, Money value object, allocation strategies, approval state machine, payment and pairwise balance rules. No Swing, HTTP, JDBC, or framework dependencies. |
| Persistence: `splitsmart-persistence-postgres` | JDBC repositories, exact numeric mapping, migrations, transaction implementation, query aggregation, durable outbox. Implements application ports; depends on application/domain. |
| Supporting artifacts | `api-contract` contains transport DTOs and schema, without domain/persistence dependencies. `splitsmart-server` composes HTTP, application, persistence, managed receipt storage, and email/delivery adapters. Parent Maven POM aggregates all artifacts. |

These are planned artifacts, not files created by this task. Each domain/application artifact contains focused feature packages for the eight modules below; avoid a Maven artifact for every feature-layer combination. Only the composition root knows concrete adapters. Application services call domain behavior and repository ports, never SQL. Persistence implements those ports, so compile-time dependencies point inward even though runtime calls reach the database.

## Feature modules

| Module | Ownership and collaboration |
|---|---|
| User accounts | Account identity, verified email/password login, reset/verification tokens, sessions; supplies authenticated principal IDs. Store adaptive salted password hashes; never return hashes in DTOs. |
| Groups | Group currency, manager roles, direct addition of registered members, membership history and current permissions. Supplies membership checks to every group use case. |
| Expenses | Expense details, payer and participants, equal/percentage allocation, receipt reference; requests an approval-policy snapshot on submission. |
| Approvals | Versioned group policies, ordered review steps and immutable decisions; coordinates final approval with Expenses inside one application transaction. Pilot has one step. |
| Notifications | Full-product durable notices for other members on expense creation, unread state, delivery retries. Failure must not undo or lose the expense. Channel/timing remains a product question. |
| Balances | Read-only calculation over approved expense shares, payments, and approved correction records. Owns canonical pair direction; no independently editable balance table. |
| Payments | Records completed settlements and audit history; validates membership, positive amount, parties, and duplicate requests. Does not initiate bank transfers. |
| Reporting | Full-product permitted expense/payment/balance summaries, date filters, echoed parameters, zero totals for empty ranges. Uses the same balance formula as Balances. |

Feature collaboration goes through application services and explicit interfaces, not another module's table writes. Shared identity references and Money are small common types; avoid a universal service or domain object with all responsibilities.

## Diagrams and deployment

[architecture.puml](./architecture.puml) contains maintained PlantUML deployment/layer, entity-relationship, and approval-sequence diagrams. The deployment diagram distinguishes pilot components from future clients and deferred capabilities.

Pilot deployment: Swing client → HTTPS `/api/v1` → Java 21 server → PostgreSQL and managed receipt storage. A development environment can place these on one host; the process and authority boundaries remain separate. Clients never connect to PostgreSQL. No offline writes, local database, synchronization protocol, broker, or service mesh is proposed.

The future web client is the primary product; Android and iOS use the same API later. They reuse the server rules and contracts, not Swing widgets or Java bytecode on iOS. Client technologies remain undecided. The server and database are deployed together as a versioned release, with migration compatibility considered before rollout.

## Request and consistency model

1. Swing collects intent and invokes HTTP on a background executor; component updates return to the Event Dispatch Thread. Display validation, network failure, and unknown submission outcome explicitly.
2. The API authenticates an opaque server-managed session. Application services recheck authorization against current membership; never trust caller-supplied roles, calculated shares, or approval status.
3. Submission validates currency/participants and stores expense, immutable shares, policy snapshot, actor/time, and idempotency result in one database transaction. Deferred notifications add an outbox record in that same transaction when implemented.
4. Review locks the expense and serializes membership/policy checks with conflicting changes. It records a decision and changes status to APPROVED only when the snapshot's required decisions are complete. Any rejection is terminal. No asynchronous financial update is involved.
5. Balance queries read approved shares and recorded payments using one consistent database snapshot. A committed final approval is immediately eligible for the next query. Reporting uses a repeatable-read snapshot when several queries compose a result.
6. A timed-out mutation is retried with its original idempotency key. Unique `(actor, operation, key)` plus request fingerprint ensures identical retries return the stored result; reuse for a different payload is rejected. Concurrent duplicates serialize on that key. Financial records and key results commit together.

An expense's status transition makes its existing shares financially effective; B does not create balance postings on approval. This removes dual-write reconciliation between source records and a ledger, but makes queries more dependent on indexes and consistent source filtering. See [data-model.md](./data-model.md) for the exact formula.

## Initial API contract direction

Proposed resource families are `/api/v1/accounts`, `/sessions`, `/groups`, `/groups/{id}/expenses`, `/expenses/{id}/decisions`, `/groups/{id}/payments`, and `/groups/{id}/balances`. Future families add notices and reports. Verification/reset and receipt upload/download are account/expense subresources. These are design proposals, not implemented endpoints or approved scope expansion.

Use JSON decimal strings for money and percentages, ISO currency codes, ISO local dates for business dates, UTC instants for audit timestamps, and opaque IDs. Server computes shares. Mutations carry an idempotency key; decisions carry an expected expense version. Errors expose a stable code, field violations, and correlation ID; distinguish unauthenticated, forbidden, validation, conflict, and transient failure. Never expose SQL or account secrets. Endpoint schemas and session expiry/revocation policy need implementation review before clients are built.

## Technology choices

Java 21 and Maven apply to client and server; compiler release is 21. Swing is the desktop presentation technology. Proposed server choices are Spring Boot MVC for HTTP/composition and JDBC transactions, PostgreSQL with unconstrained NUMERIC for exact values, and Flyway for migrations. Framework dependencies stay outside the domain. JUnit 5 Jupiter runs domain/application/contract tests through Surefire and database integration tests through Failsafe; integration tests require a disposable PostgreSQL database. Pin compatible dependency/plugin versions during implementation; this document does not claim a validated dependency stack.

Server-managed receipt files use opaque keys and authenticated retrieval; metadata is relational. Pilot storage can be a protected server directory, later an object store implementing the same port. Deploy behind TLS, keep credentials server-side, protect session tokens and backups, hash reset tokens with expiry and single use, and redact secrets from logs. These are proposed controls requiring security review, not a security certification.

## Requirement traceability and planned evidence

All evidence below is **planned**, not executed. FR, NFR, AC, and SC identifiers refer to the existing specification; no new requirement IDs are invented.

| Source requirements | Responsible design | Planned verification |
|---|---|---|
| CR-001; FR-001; NFR-001 | Accounts, sessions, email port, authorized account queries | AC-001: JUnit 5 registration/token-expiry/reset/session tests plus actual email integration before full acceptance |
| CR-002; FR-002 | Groups and current-membership checks | AC-002: manager/direct-add tests and non-member denial |
| CR-003; FR-003–FR-005; NFR-002 | Expenses, Money, allocation strategies, receipt adapter | AC-003/004: precision, no maximum business amount, invalid participant/split tests and receipt authorization checks |
| CR-005; FR-006–FR-008 | Policy snapshots, review transaction, Expenses status | AC-005: unauthorized/repeated/concurrent reviews, rejection, incomplete multi-step approval; multi-step tests belong to full-product phase |
| CR-004; FR-009 | Deferred Notifications/outbox; visible expense status | AC-006: recipient filtering, retry/deduplication and delivery failure without expense loss |
| CR-006; FR-010; NFR-002 | Balances source aggregation | AC-007, SC-002/003: pair netting, zero self-debt, exclusion of pending/rejected records |
| CR-007; FR-011; NFR-003/006 | Payments and atomic idempotency | AC-008: positive payments, preserved actor/time, lost-response retry and concurrent duplicate tests |
| CR-008; FR-012/013 | Deferred Reporting with shared balance semantics | AC-009, SC-004: permission/date/parameter filtering and empty zero totals |
| CR-009; FR-014 | Future primary web presentation over existing server API | AC-010: browser acceptance once implemented; API and Swing alone do not qualify |
| FR-015; NFR-001/003 | Retained memberships/history, current access checks, audit records | AC-013: removal preserves history and revokes current group access |
| NFR-004 | Swing presenters, field feedback and visible review status | AC-011, SC-001/006: measured first-time-user sessions; JUnit cannot establish usability percentages |
| NFR-005 | Indexed source queries, network-aware UI | AC-012, SC-005: end-to-end display latency, groups up to 100 members/10,000 expenses, at least 95% within 3 seconds in a recorded environment |
| NFR-006 across submissions/reviews/payments | Server transaction and request-key uniqueness | Concurrent PostgreSQL integration tests, response-loss retries, rollback fault tests |

## Advantages, costs, and delivery gates

Advantages: one authority across desktop users from the outset; database-enforced concurrency; immediate reusable API boundary; stronger compile-time layer separation; no source-to-balance-ledger drift; future clients attach without migrating local user databases.

Disadvantages: the pilot needs a server, network, database administration, migrations, TLS, and recovery procedures; it cannot operate independently offline. Multiple Maven artifacts and transport DTOs add maintenance. Source-based balance aggregation can cost more than a precomputed ledger/projection; the specified latency requires measurement, including the network and Swing rendering.

Resolve percentage tolerance/rounding, overpayments, non-date report parameters, date boundaries, correction authorization, and notification channel/timing before affected full-product features are accepted. Multi-step approval and reporting remain deferred, not removed. Constitution compliance is proposed through traceability, Java/Maven/Swing/JUnit/BigDecimal, explicit OO ownership and maintained PlantUML. Human review is still required before acceptance; no exception or product-scope approval is implied.
