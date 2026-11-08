# SplitSmart Architecture Validation Quickstart

This guide describes validation for the planned Java Swing MVP. The repository currently contains planning documents only; there is no Maven project or application implementation yet. These commands and outcomes become executable after the implementation phase creates the Maven scaffold.

## Prerequisites

- JDK 21 installed and selected (`java -version` reports 21).
- Maven installed (`mvn -version` uses JDK 21).
- A desktop environment for manual Swing acceptance flows.

## Build and tests

From the repository root, after the Maven project exists:

```powershell
mvn clean test
mvn verify
```

Expected outcomes: Maven compiles with `release` 21; JUnit 5 tests run; test failures return a non-zero exit code. The domain/application test suites must not require Swing startup. Persistence integration tests use a temporary SQLite database and clean up their own test data.

Launch the configured Swing main class from the IDE or the project's documented Maven run configuration. Do not add a runtime entry point as part of this planning-only task.

## Validation scenarios

1. **Equal split and rounding**: Submit an amount that does not divide evenly among participants. Verify all shares sum to the original amount and the full minor-unit remainder goes to the payer.
2. **Percentage split**: Submit percentages totaling exactly 100% and verify allocated shares reconcile. Verify a non-100% total is rejected pending a clarified tolerance policy.
3. **Approval and balance**: Submit an expense, verify no balance posting while pending, approve it as an authorized reviewer, then verify postings match the shares. Verify rejection and incomplete multi-step approval produce no postings.
4. **Payment and retry**: Record a positive payment, confirm its pairwise balance effect and audit fields, then repeat with the same idempotency key and verify no duplicate posting. Verify non-positive amounts are rejected.
5. **Authorization/history**: Verify a non-member cannot query group activity; remove a member and confirm historical records remain while current access follows the specified permissions.
6. **Reports**: Query expenses and payments across selected date ranges, verify report-specific date semantics, group permissions, applied filters, and zero totals for empty ranges.
7. **Responsiveness**: Seed up to 100 group members and 10,000 expenses; measure balance and date-filtered report requests and verify at least 95% display within 3 seconds under a recorded “normal operating conditions” environment.
8. **Swing responsiveness**: Exercise slow repository operations and verify the EDT remains responsive; surface validation and persistence errors rather than silently ignoring them.

## Test ownership

Use JUnit 5 domain and application tests for scenarios 1–5, and repository integration tests for transaction, uniqueness, and history behavior. UI wiring is covered with focused tests where practical plus a manual desktop run. The acceptance and performance criteria are defined in [spec.md](./spec.md); entity constraints are summarized in [data-model.md](./data-model.md).
