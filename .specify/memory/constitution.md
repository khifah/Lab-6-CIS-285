<!--
Sync Impact Report
Version change: uninitialized template -> 1.0.0
Modified principles: template placeholders -> I. Customer Outcomes and Traceability;
  II. Java 21 and Maven; III. Swing with Business-Logic Separation;
  IV. Object-Oriented Design; V. Exact Monetary Values; VI. JUnit 5 Verification;
  VII. PlantUML Design Records; VIII. Human Review of AI Output
Added sections: Customer Requirements and Implementation Decisions;
  Development Workflow and Quality Gates
Removed sections: none
Follow-up TODOs: confirm the original ratification date.
-->
# SplitSmart Constitution

## Core Principles

### I. Customer Outcomes and Traceability
Every customer-facing capability MUST trace to the original customer request or a
subsequently approved requirement. Keep requirements distinct from implementation
decisions, and map each requirement to relevant design and verification evidence.
Changes, assumptions, and conflicts MUST be recorded rather than presented as
customer-approved behavior. This preserves the customer's intent as implementation
choices evolve.

### II. Java 21 and Maven
The application MUST use Java 21 and Maven as its build and dependency-management
baseline. Build configuration and dependencies MUST be declared in Maven project
files so that builds and tests can be reproduced consistently.

### III. Swing with Business-Logic Separation
The desktop user interface MUST use Java Swing. Swing event handlers MUST delegate
to business logic that is independent of Swing components; business rules MUST be
testable without launching the GUI. This boundary prevents presentation changes
from silently changing expense and balance behavior.

### IV. Object-Oriented Design
Code MUST use object-oriented design with clear ownership of state and behavior,
encapsulation, and focused responsibilities. Types and abstractions MUST clarify
the domain or reduce coupling; unnecessary indirection MUST NOT be introduced.

### V. Exact Monetary Values
Money and monetary calculations MUST use `BigDecimal`, never binary floating-point
types. Rounding rules MUST be explicit, and split allocations MUST reconcile to
the original expense amount. This avoids precision errors that can produce
incorrect balances.

### VI. JUnit 5 Verification
Automated tests MUST use JUnit 5. Tests MUST cover business rules and relevant
boundary cases, including expense splits, approvals, balance updates, payments,
and report calculations. A requirement MUST NOT be described as verified unless
its relevant tests or other stated evidence have actually been checked.

### VII. PlantUML Design Records
PlantUML MUST be used for maintained design diagrams when a diagram is needed to
explain architecture, domain relationships, or important workflows. Diagrams MUST
reflect the current intended design and MUST NOT substitute for requirement
traceability or executable verification.

### VIII. Human Review of AI Output
AI-generated code, tests, diagrams, and documentation MUST receive human review
before acceptance. The reviewer MUST check correctness, requirement alignment,
test quality, and relevant risks; AI output or AI-generated test results alone
are not evidence of approval.

## Customer Requirements and Implementation Decisions

The customer requirements are the requested outcomes in `UserRequirements.md`:
user accounts and login; group management; expenses with amount, date, description,
equal or percentage splits, optional receipt images, and participant selection;
notifications of group expenses; an expense approval workflow; balances between
users; payment recording; and expense, balance, and payment reports with date-range
and other parameters. The request identifies the primary supported platform as
web-based.

The Java 21, Maven, Swing, JUnit 5, PlantUML, object-oriented design, GUI/business
logic separation, and `BigDecimal` choices in this constitution are implementation
and engineering decisions, not customer requirements. Requirement traceability
and human review of AI output are governance decisions. These classifications
MUST remain explicit in specifications, plans, and reviews.

The confirmed scope is an interim Java Swing desktop pilot followed by a
primary web-based product. The desktop pilot MUST NOT be represented as
satisfying the customer's web-based platform requirement. Specifications,
plans, and reviews MUST preserve this distinction until the primary web product
is delivered.

## Development Workflow and Quality Gates

For each feature, identify the originating customer requirement and record
implementation decisions separately. Link requirements to relevant design
artifacts, including PlantUML diagrams where useful, and to JUnit 5 tests or other
clearly stated verification evidence. Reviews MUST check that those links are
current and that no implementation decision has been mislabeled as customer
acceptance.

Changes MUST be built and tested using the Maven project configuration. Reviewers
MUST examine AI-generated contributions under Principle VIII. Any unresolved
requirement conflict or failed quality gate MUST be reported; it MUST NOT be
hidden by claiming successful verification.

## Governance

This constitution governs project decisions and reviews. Amendments MUST be
proposed with their rationale, affected principles, requirement impact, and
necessary updates to related specifications, diagrams, and tests. The project
maintainers MUST review and approve amendments before they take effect.

Use semantic versioning for this constitution: MAJOR for backward-incompatible
governance changes or principle removals/redefinitions, MINOR for new principles
or materially expanded obligations, and PATCH for clarifications and
non-semantic wording changes. Every amendment MUST update the version and
last-amended date.

Each change review MUST assess compliance with the applicable principles and
quality gates. Exceptions MUST be documented with rationale, scope, and approval;
they MUST NOT silently weaken a principle. Reviewers MUST confirm that requirement
traceability is preserved and that implementation choices remain distinguishable
from customer requirements.

**Version**: 1.0.0 | **Ratified**: TODO(RATIFICATION_DATE): original adoption date is unknown | **Last Amended**: 2026-10-30
