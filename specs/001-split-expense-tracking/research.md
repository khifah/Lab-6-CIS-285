# Architecture Research

**Selected Architecture A:** [Consolidated design decisions](../../docs/architecture-consolidated/design-decisions.md) refine exact SQLite storage, authorization, idempotency and migration while retaining one local Swing/SQLite deployment and the posting ledger. See [ADR/change rationale](../../docs/architecture-consolidated/architecture-decision-record.md). These are proposed designs, not tested implementation evidence.

## Java 21 and Maven

**Decision**: Use Java 21 and Maven's standard source layout; configure the Maven Compiler Plugin with `release` 21.

**Rationale**: This meets the project constitution and makes the language, bytecode, and available Java APIs target explicit. Start with one Maven project; split modules only when server deployment or independent artifact reuse becomes concrete.

**Alternatives considered**: A multi-module build was deferred because the current deliverable is a single Swing application. Gradle or another Java version would conflict with the stated project baseline.

**Sources**: [Maven Java release guide](https://maven.apache.org/guides/mini/guide-java-target.html#using-the-release-option); [Maven standard directory layout](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html).

## Swing and business-logic separation

**Decision**: Make Swing views/presenters a thin adapter over application use cases. Keep business rules independent of Swing, persistence, and networking.

**Rationale**: Expense and balance rules can be tested without launching a GUI. Swing component access is confined to the Event Dispatch Thread (EDT); potentially slow work must not block it.

**Alternatives considered**: Putting calculations in event handlers would couple business behavior to the UI and prevent focused domain tests. Replacing Swing conflicts with the required MVP technology.

**Sources**: [Oracle Swing concurrency tutorial](https://docs.oracle.com/javase/tutorial/uiswing/concurrency/dispatch.html); Alistair Cockburn, [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/).

## BigDecimal expense allocation

**Decision**: Use `BigDecimal` for amounts, percentages, allocations, payments, and balance postings. Make currency scale and rounding explicit; reconcile each allocation to the original amount. Follow the spec's equal-split rule assigning the full remainder to the payer.

**Rationale**: Binary floating-point cannot represent many decimal amounts exactly. `BigDecimal` supports explicit rounding, but non-terminating division needs a rounding mode, and `equals` is scale-sensitive; use `compareTo` for numeric comparisons. The spec leaves percentage tolerance open, so the baseline requires an exact 100% total pending a product decision.

**Alternatives considered**: Floating-point money is rejected by the constitution. Both largest-remainder and residual-to-payer percentage policies require human decision; neither is approved by this research. The original residual-to-payer suggestion remains a proposal only. Block affected percentage-rounding implementation until resolved; do not confuse the confirmed equal-split payer remainder with percentage policy.

**Sources**: [Java SE 21 BigDecimal API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/math/BigDecimal.html); [Java SE 21 RoundingMode API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/math/RoundingMode.html); [spec.md assumptions and open questions](./spec.md).

## JUnit 5 verification

**Decision**: Use JUnit 5 Jupiter through Maven Surefire. Test domain calculations and state transitions as ordinary Java code, then test SQLite adapters and critical UI wiring at their boundaries.

**Rationale**: The selected framework satisfies the constitution and supports isolating allocation, approval, balance, payment, and reporting behaviors from Swing.

**Alternatives considered**: UI-only tests would be slower and would not isolate business rules. Other test frameworks conflict with the project's mandated JUnit 5 baseline.

**Sources**: [JUnit 5 User Guide](https://docs.junit.org/5.11.4/user-guide/); [Maven Surefire JUnit Platform guide](https://maven.apache.org/surefire/maven-surefire-plugin/examples/junit-platform.html).

## Desktop-to-full-product boundary

**Decision**: Treat Swing as an interim desktop adapter. Keep domain/application boundaries independent of Swing and define a future versioned HTTPS API for web, Android, and iOS clients. Do not prescribe their UI technologies here.

**Rationale**: The feature specification names web as the customer's primary platform and explicitly states the desktop pilot does not satisfy it. The current planning request additionally asks the architecture to preserve Android and iOS as future clients. Native clients call the server API; they do not depend on Swing or assume the desktop JVM is their runtime.

**Alternatives considered**: Delivering web/API first aligns more directly with CR-009 but does not produce the requested Swing MVP. A Swing-only product would fail to represent the requested full-platform direction. Premature microservices or client-specific technology selections are deferred.

**Sources**: [Java SE 21 `java.desktop` module](https://docs.oracle.com/en/java/javase/21/docs/api/java.desktop/module-summary.html); [Android architecture guidance](https://developer.android.com/topic/architecture); [Apple SwiftUI documentation](https://developer.apple.com/documentation/swiftui); [spec.md](./spec.md).

## Storage boundary

**Decision**: Use SQLite behind repository ports for the single-device pilot. Keep a future centralized server database as a separate full-product decision.

**Rationale**: A local relational store is a small, testable persistence choice for a desktop pilot and avoids hardwiring SQL into business logic. It is not a shared account service or a production web/mobile backend.

**Alternatives considered**: In-memory-only storage would not preserve pilot activity. A server database and API in the MVP would increase deployment scope and require decisions not present in the spec. The chosen pilot storage must be replaced or supplemented for shared multi-device operation.

**Sources**: The storage selection is an architecture proposal based on the scope in [spec.md](./spec.md); it is not a customer requirement.
