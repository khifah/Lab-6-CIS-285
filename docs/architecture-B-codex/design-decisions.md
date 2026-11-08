# Architecture B design decisions

All decisions below are proposed engineering choices, subject to constitution-required human review. They do not amend customer requirements or existing Spec Kit artifacts. [Architecture](./architecture.md) provides requirement-to-verification traceability; [data model](./data-model.md) specifies invariants.

| ID | Decision and reason | Alternatives, disadvantages, and tradeoffs |
|---|---|---|
| B-01 | Thin Swing client and centralized Java 21 server from the pilot. Sharing authority early prepares future clients and tests real authorization/concurrency boundaries. | A's local desktop is simpler to install and works without a server. B requires network availability, operational ownership, TLS, and a database even for the pilot. No offline editing is promised. |
| B-02 | Maven multi-module build with presentation, application, domain, persistence, contract, and composition artifacts. Feature packages cover all eight requested areas. | A uses one Maven project with packages. B makes dependencies enforceable at build time but adds POMs, DTO mapping, and release coordination. Avoid one artifact per tiny feature. |
| B-03 | One layered server monolith, Spring Boot MVC/JDBC adapters, PostgreSQL and migrations. Frameworks stay outside domain. | Microservices add distributed financial consistency and operations without a stated need. A's SQLite is smaller; B pays server costs for shared transactions and immediate central persistence. Exact compatible versions are implementation work. |
| B-04 | Compute balances from approved expense shares, payments, and authorized corrections. No second posting ledger. | A maintains immutable balance postings. B removes posting synchronization and lets final approval take effect transactionally through status. Queries are more complex; every path must apply identical status/currency rules. A dedicated ledger offers a simpler financial event stream. Benchmark B before adding caches. |
| B-05 | Separate command/query application interfaces, sharing one database and transaction model. | No separate read database, event sourcing, or eventual financial projections. Keeps read-after-write behavior understandable; large report queries still compete with writes. |
| B-06 | Define versioned HTTPS/JSON and decimal-string DTO conventions now; opaque revocable sessions, server authorization, persistent mutation idempotency. | A defers API/session contracts. B enables early contract testing but commits to maintaining transport compatibility and securing sessions. Unknown request outcomes require explicit client retry behavior. |
| B-07 | Deferred Notifications use a transactional outbox and deduplicated notices; financial work stays synchronous. | Direct external delivery during submission risks coupling expense success to provider failure. Outbox requires retry/cleanup operations; channel and delivery timing remain unresolved. No notification feature is claimed for the pilot. |
| B-08 | Encapsulated BigDecimal Money, explicit currency scale, payer remainder for equal split. | Required by constitution/spec. Percentage residual-to-payer is provisional; tolerance and residual semantics need review. Binary floating-point is incompatible with the baseline. |
| B-09 | Retain financial records and membership periods; snapshot approval policy; use explicit compensations for future corrections. | Mutable/deleted history simplifies CRUD but loses auditability. B needs more retained data and a specified correction workflow; that workflow is not invented as an accepted requirement. |
| B-10 | JUnit 5 at domain, use-case, HTTP-contract, and real PostgreSQL boundaries; manual Swing/usability plus end-to-end performance measurements. | More setup than A's local SQLite checks. In-memory substitutes alone cannot prove PostgreSQL locking, precision, or transaction behavior. No tests are claimed executed by these design documents. |

## Architecture A versus B

| Dimension | Architecture A as documented | Proposed Architecture B |
|---|---|---|
| Pilot topology | Swing, application and SQLite in one process | Swing client, Java server, PostgreSQL |
| Calls | In-process use cases | Versioned HTTPS boundary from first pilot |
| Authority | Local single-device pilot; central service later | Server-owned identity, authorization and financial data immediately |
| Build | One Maven project; module split deferred | Maven artifacts enforce layer boundaries immediately |
| Persistence evolution | SQLite now; future central database undecided | PostgreSQL from pilot; no local-to-central financial migration |
| Balance truth | Immutable balance postings derived transactionally from source actions | Approved source shares and payments aggregated directly; no posting ledger |
| Concurrency work | Local transactions initially | Network retries, concurrent sessions, request keys and database locking initially |
| Notifications | Deferred full-product adapter | Also deferred, with durable outbox design selected |
| Platform future | Server/API and clients added later | Web/Android/iOS attach to existing API; web UI still must be delivered |
| Main advantage | Smaller pilot deployment and implementation scope | Earlier shared authority and client reuse; fewer financial representations |
| Main cost | Later central-service/API transition | Higher pilot complexity and operational dependency |

Both designs intentionally share Java 21, Maven, Swing, JUnit 5, inward business-rule dependencies, BigDecimal accuracy, auditability, and the complete customer inventory. Those are constraints, not distinguishing claims. B is appropriate if central shared use and API reuse justify server setup now; A is preferable if the immediate goal is the smallest single-device demonstration.

## Review and implementation gates

Retain all specification open questions: notification channel/timing, overpayments and any transfer initiation, percentage tolerance, non-date report filters, and allowed expense/payment dates. Record B's additional proposed decisions for review: percentage residual handling, historical balance-report recognition, correction permissions, session lifecycle, and hosting/backup ownership. Payments record settlements only; no bank integration is proposed. Network architecture does not approve feature expansion or count as delivering the primary web product.

Before implementation acceptance, reviewers should check all eight constitution principles, the full traceability table, transaction/locking design, security controls, and diagram consistency. Before product acceptance, execute the mapped JUnit 5 and integration suites, real delivery checks where applicable, usability sessions, and the specified latency measurement. This deliverable contains design evidence only.
