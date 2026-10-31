# Feature Specification: SplitSmart Expense Sharing

**Feature Branch**: `001-split-expense-tracking`

**Created**: 2026-10-30

**Status**: Draft

**Input**: User description: "Read UserRequirement.md in this workspace as the authoritative customer requirements. Write a testable SplitSmart feature specification: actors, user scenarios, functional and non-functional requirements, acceptance criteria, requirement IDs (FR-001, etc.), and open questions. Preserve all original customer features in the requirements inventory; label the Java Swing MVP subset separately. Do not write Java code. Do not edit UserRequirement.md."

## Clarifications

### Session 2026-10-30

- Q: Must the person who pays an expense also be one of the people who share it? → A: The payer must be included among the expense participants.
- Q: How should a person become a member of a group? → A: A group manager adds a registered user directly; membership starts immediately.
- Q: When an equal split leaves leftover cents, how should the system assign them? → A: Assign the entire rounding remainder to the payer.
- Q: Which expense amounts should the system reject as invalid? → A: Reject zero or negative amounts and amounts with more decimal places than the currency permits; do not impose a maximum.
- Q: What should a person use to log in to a SplitSmart account? → A: A verified email address and password, with password reset.

## User Scenarios & Testing *(mandatory)*

### Actors

- **Registered user**: Creates or is added to groups, records expenses and payments, and views permitted expense, balance, and payment information.
- **Group manager**: Creates and manages group membership and selects the group's expense-review policy.
- **Expense submitter**: A registered user who records a shared expense and submits it for review.
- **Expense reviewer**: A group member authorized by the group to review, approve, or reject submitted expenses.
- **Notified group member**: A group member who receives notice of an expense or its review status.

### User Story 1 - Access an account and organize a group (Priority: P1)

A person creates an account or logs in, then a group manager adds them to a group, so that expense sharing is limited to the people involved.

**Why this priority**: Accounts and groups establish the people and boundaries needed for all shared-expense activity.

**Independent Test**: Create two accounts, create a group with one account, have its manager directly add the other, and verify that the added user immediately gains access while an unrelated account cannot access the group.

**Acceptance Scenarios**:

1. **Given** a person without an account, **When** they register with an email address and password and verify the email address, **Then** an account is created and they can log in.
2. **Given** an existing account, **When** its owner supplies valid login credentials, **Then** they can access their own expense and balance information.
3. **Given** a registered user who cannot remember their password, **When** they complete the password-reset process, **Then** they can set a new password and regain account access.
4. **Given** a group manager and registered users, **When** the manager creates a group and directly adds members, **Then** those users immediately become group members and can access the group's shared information.
5. **Given** a user who is not a group member, **When** they attempt to access that group's shared information, **Then** access is denied.

### User Story 2 - Submit an expense for a group (Priority: P1)

A group member records an expense with its amount, date, description, split method, participants, and optional receipt, so the people involved can review their shared obligation.

**Why this priority**: Recording a complete expense is the source of the balances and reports the product provides.

**Independent Test**: Submit an expense with valid details and participants, verify its details and pending status, and verify that no balance changes before approval.

**Acceptance Scenarios**:

1. **Given** a group member with an active group, **When** they submit a valid expense with participants and an equal split, **Then** the expense is saved for review with the submitted details.
2. **Given** an expense with a percentage split, **When** the percentages total 100% and the expense is submitted, **Then** each participant's allocation reflects their specified share.
3. **Given** an expense submission, **When** its receipt image is omitted or attached, **Then** the expense can be submitted and the receipt is retained only when supplied.
4. **Given** missing required details or an invalid split, **When** a user attempts to submit the expense, **Then** the system identifies the invalid fields and does not submit it.

### User Story 3 - Review and approve expenses (Priority: P1)

A designated reviewer reviews a submitted expense and approves or rejects it, so balances include reviewed expenses rather than unapproved claims.

**Why this priority**: Review is an explicit customer requirement and guards the correctness of shared balances.

**Independent Test**: Submit an expense, approve it as an authorized reviewer, and verify that it enters group totals and affects balances only after approval; repeat with a rejection and verify no balance change.

**Acceptance Scenarios**:

1. **Given** a pending expense and an authorized reviewer, **When** the reviewer approves it, **Then** it is marked approved and is included in balance calculations.
2. **Given** a pending expense and an authorized reviewer, **When** the reviewer rejects it, **Then** it is marked rejected, its submitter can see the decision, and it does not affect balances.
3. **Given** an expense requiring multiple review steps, **When** only some required reviewers have approved it, **Then** it remains pending and does not affect balances.
4. **Given** a user without review permission, **When** they attempt to approve or reject an expense, **Then** the decision is not applied.

### User Story 4 - Follow group activity and settle balances (Priority: P1)

A group member sees new expense activity, checks what they owe or are owed, and records a payment, so the group's outstanding balances reflect settlements.

**Why this priority**: Notifications and payment records make the balances actionable and keep settlement status current.

**Independent Test**: Approve an expense, verify group members receive an expense notification and the related balances update, record a payment, and verify the remaining balance.

**Acceptance Scenarios**:

1. **Given** a group member submits an expense, **When** it is created, **Then** other group members are notified of the new expense.
2. **Given** approved expenses involving group members, **When** a member views balances, **Then** the amount owed between each relevant pair is shown and reflects those expenses.
3. **Given** an outstanding balance, **When** a payer records a payment to the payee, **Then** the payment is recorded and the outstanding balance is reduced by the payment amount.
4. **Given** a non-positive payment amount, **When** a user submits it, **Then** the system rejects it without changing payment history or balances.

### User Story 5 - Review expense, balance, and payment reports (Priority: P2)

A group member reviews summaries or chooses report filters, including a date range, to understand group spending and settlement history.

**Why this priority**: Reporting provides a consolidated view of the activity recorded through the higher-priority workflows.

**Independent Test**: Create approved expenses and payments on dates inside and outside a selected range, generate each report type, and verify that only matching permitted records and totals appear.

**Acceptance Scenarios**:

1. **Given** recorded activity for a group, **When** a member requests an expense, balance, or payment summary, **Then** the summary reflects the matching records.
2. **Given** a valid date range and report parameters, **When** a member requests a custom report, **Then** the report includes only matching records and totals.
3. **Given** an invalid date range, **When** a member requests a report, **Then** the system explains the invalid range and does not present it as a valid report.

### Edge Cases

- Equal splits that leave one or more leftover minor currency units MUST assign the entire remainder to the payer, and the shares MUST add up to the expense total.
- Percentage splits that do not total 100%, include a negative share, or produce an allocation beyond the expense total MUST be rejected.
- An expense with no eligible participants, a non-positive amount, an amount with more decimal places than the currency permits, or missing required details MUST NOT be submitted. No maximum expense amount is imposed.
- Any allowed historical or future expense-date boundary remains undecided; see Open Questions.
- A group member leaving or being removed MUST NOT erase historical expenses, approvals, payments, or reports.
- Duplicate submissions or repeated approval actions MUST NOT apply an expense or payment to balances more than once.
- A notification delivery failure MUST NOT silently discard the expense; the expense and its review status remain available in the group.
- Rejected or pending expenses MUST NOT affect settled or outstanding balances.
- A payment dispute, reversal, or correction MUST preserve the original payment history and require an auditable correction rather than silently deleting it; the correction workflow remains to be specified.
- Reports MUST handle a date range with no matching activity by returning an empty result with zero totals.

## Requirements *(mandatory)*

### Customer Requirements Inventory (Full Product)

The following inventory preserves the customer request in [UserRequirements.md](../../UserRequirements.md). The customer's requested primary platform is web-based. These are customer requirements, not implementation choices.

| Inventory ID | Customer-requested capability |
|---|---|
| CR-001 | Each person has an account, can log in, and can access their own expense and balance information. |
| CR-002 | Users can create and manage groups such as roommate, trip, or party groups for shared expenses and balances. |
| CR-003 | Users can create expenses containing amount, date, description, equal or percentage split, optional receipt image, and involved users. |
| CR-004 | Group members are notified when another group member creates an expense. |
| CR-005 | Expenses are reviewed and approved before being included in the system and used to adjust balances; a group may use a single or multiple review steps. |
| CR-006 | The system tracks what each user owes every other user based on entered expenses. |
| CR-007 | Users can record payments made to settle balances, and balances reflect those payments. |
| CR-008 | Users can view expense, balance, and payment reports or summaries, including custom date ranges and other parameters. |
| CR-009 | The primary supported platform is web-based. |

### Java Swing MVP Subset (Implementation Decision; Not a Customer Requirement)

This proposed MVP subset is separate from the full customer requirements inventory and does not remove or reclassify any customer capability:

- Account access and group creation and management.
- Expense entry with amount, date, description, participants, equal or percentage splits, and optional receipt.
- A single authorized reviewer approval step.
- Pairwise balance tracking for approved expenses and manual payment recording.

Notifications and configurable reports are retained as full-product customer requirements but are outside this proposed MVP subset. Multi-step approval is also retained for the full product; the MVP subset uses one reviewer.

**Confirmed scope decision**: The Java Swing MVP is an interim desktop pilot; the primary web-based product follows it. The desktop pilot does not satisfy CR-009 and MUST NOT be represented as doing so. The full product remains subject to CR-009.

### Actors and Permissions

- A registered user can view their own account information and information shared with groups they belong to.
- A group manager can create a group, directly add registered users with immediate membership, remove members, and designate the group's expense-review policy and authorized reviewers.
- A group member can submit expenses and record payments involving them.
- An authorized reviewer can approve or reject expenses submitted to the review process.
- A group member can view activity and reports for groups they belong to; a non-member cannot view group activity.

### Functional Requirements

- **FR-001**: The system MUST allow each person to create an individual account using an email address and password, verify the email address, log in, reset a forgotten password, and access their own expense and balance information.
- **FR-002**: The system MUST allow an authorized user to create a group, directly add registered users with immediate membership, remove members, and view the group's membership.
- **FR-003**: The system MUST allow a group member to submit an expense with a positive amount expressed to no more than the currency's permitted decimal places, date, description, payer, and one or more involved group members; the payer MUST be included among those participants. No maximum amount applies.
- **FR-004**: The system MUST support equal and percentage-based expense allocation among selected participants; percentage allocations MUST total 100%, with any permitted input tolerance to be confirmed.
- **FR-005**: The system MUST allow an expense submitter to attach an optional receipt image to an expense.
- **FR-006**: The system MUST place each submitted expense into the group's configured single-step or multi-step review workflow before that expense affects balances.
- **FR-007**: The system MUST allow authorized reviewers to approve or reject a pending expense and MUST preserve its decision and review status.
- **FR-008**: The system MUST include an expense in group balances only after all approvals required by that group's policy are complete; pending and rejected expenses MUST NOT affect balances.
- **FR-009**: The system MUST notify other members of the group when an expense is created and MUST make its current review status visible to those permitted to view it.
- **FR-010**: The system MUST calculate and show amounts owed between each pair of group members from approved expenses and recorded payments.
- **FR-011**: The system MUST allow a user to record a payment between group members, including payer, payee, amount, date, and optional description, and MUST update the corresponding outstanding balance.
- **FR-012**: The system MUST provide expense, balance, and payment summaries for a group, subject to the requesting user's access permissions.
- **FR-013**: The system MUST allow a user to filter reports by a date range and other available report parameters and MUST identify the filters used in the result.
- **FR-014**: The primary product MUST be accessible through a web-based platform; the interim Java Swing desktop pilot MUST NOT be counted as satisfying this requirement.
- **FR-015**: The system MUST preserve historical expense, review, payment, and balance-impact records when group membership changes.

### Non-Functional Requirements

- **NFR-001 (Privacy)**: A user MUST NOT be able to view another user's private account information or a group's activity without the access described under Actors and Permissions.
- **NFR-002 (Monetary accuracy)**: For each currency, allocations and payment adjustments MUST reconcile to the recorded amount at that currency's smallest unit; no rounding discrepancy may be lost.
- **NFR-003 (Auditability)**: Expense submission, review decisions, payment records, and corrections MUST retain actor and timestamp information sufficient to reconstruct the sequence of balance-affecting events.
- **NFR-004 (Usability)**: In usability testing, at least 90% of first-time participants MUST be able to submit a valid expense and identify its review status without assistance.
- **NFR-005 (Responsiveness)**: For groups with up to 100 members and 10,000 recorded expenses, at least 95% of balance and date-filtered report requests MUST display results within 3 seconds under normal operating conditions.
- **NFR-006 (Reliability)**: Repeating a submission, review, or payment action MUST NOT duplicate its effect on balances.

### Acceptance Criteria

| Criterion | Requirement(s) | Verifiable outcome |
|---|---|---|
| AC-001 | FR-001, NFR-001 | A person can register with email and password, verify their email, log in, and reset a forgotten password; an unrelated user cannot access their permitted information. |
| AC-002 | FR-002, NFR-001 | A manager can create a group and directly add a registered user; that user's membership and access begin immediately, while a non-member is denied access. |
| AC-003 | FR-003, FR-005 | A valid expense has a positive amount within the currency's permitted decimal precision, includes the payer among participants, and may include a receipt; zero, negative, excess-precision, or incomplete expenses are rejected. No maximum amount is applied. |
| AC-004 | FR-004, NFR-002 | An equal split reconciles to the total at the currency's smallest unit, assigning all leftover minor units to the payer. A percentage split totaling 100% reconciles to the total; allowed input tolerance remains unresolved and MUST NOT be assumed. |
| AC-005 | FR-006, FR-007, FR-008 | Only an authorized review decision satisfying the group's policy makes an expense balance-affecting; rejection and incomplete approval leave balances unchanged. |
| AC-006 | FR-009 | Creating an expense makes a notification available to other group members, and its status is visible to permitted members. |
| AC-007 | FR-010, NFR-002 | Pairwise balances reconcile to approved expenses and payments at the currency's smallest unit. |
| AC-008 | FR-011, NFR-003, NFR-006 | Recording a payment changes the appropriate balance once, retains who recorded it and when, and repeating the same action does not double-apply it. Expense submissions, review decisions, and corrections also retain their actor and timestamp. |
| AC-009 | FR-012, FR-013, NFR-001 | Expense, balance, and payment summaries return only permitted records matching the selected date range and parameters; empty ranges show zero totals. |
| AC-010 | FR-014 | The full primary product is usable through a web-based platform; the interim desktop pilot is not counted as satisfying this criterion. |
| AC-011 | NFR-004 | At least 90% of first-time usability-test participants complete expense submission and identify review status without assistance. |
| AC-012 | NFR-005 | At least 95% of tested balance and filtered-report requests for the stated data volume display results within 3 seconds under normal operating conditions. |
| AC-013 | FR-015, NFR-001 | After a member is removed, permitted current members can still access historical group records, while the removed member cannot access current group activity. |

### Key Entities

- **User account**: A person's identity and account-access information.
- **Group**: A shared-expense space with members, review policy, and shared activity.
- **Membership**: A user's participation and permissions within a group.
- **Expense**: A submitted claim with amount, date, description, payer, participants, split method and allocations, optional receipt, and review status.
- **Review decision**: An authorized reviewer's approval or rejection, timestamp, and optional explanation for an expense.
- **Notification**: A notice of group expense activity and its current review status for a group member.
- **Balance**: The current amount one member owes another, derived from approved expenses and recorded payments.
- **Payment**: A recorded settlement with payer, payee, amount, date, and optional description.
- **Report**: A view or summary of permitted expense, balance, or payment records under selected filters.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 90% of first-time users in usability testing complete account access, group selection, and a valid expense submission within 5 minutes without assistance.
- **SC-002**: 100% of approved test expenses and payments produce pairwise balances that reconcile to the currency's smallest unit.
- **SC-003**: 100% of pending, rejected, or incompletely approved test expenses leave balances unchanged.
- **SC-004**: 100% of report test cases include only records matching the selected date range, parameters, and user's group access.
- **SC-005**: At least 95% of tested balance and filtered-report requests for groups up to 100 members and 10,000 expenses display within 3 seconds under normal operating conditions.
- **SC-006**: At least 90% of usability-test participants can identify who owes whom and the current expense review state without help.

## Assumptions

- Each group uses one currency at a time; multi-currency conversion is not in the original customer request.
- The expense submitter is the payer unless they identify another group member as payer; the selected payer must be included among the participants who share the expense.
- Equal shares use the selected participants only; any indivisible remainder is assigned to the payer so shares sum to the original amount.
- Percentage splits are entered as participant percentages; the permitted input tolerance around 100% remains open as noted under Open Questions.
- A group manager can choose a single reviewer or an ordered list of reviewers; the expense remains pending until the configured policy is satisfied.
- The notification channel and delivery timing are not specified by the customer request and remain open as noted under Open Questions.
- The customer request requires recording payments already made; whether the product also initiates transfers is not specified and remains open for scope confirmation.
- Reports use the expense date for expense filtering and the payment date for payment filtering.
- Historical records remain visible to group members with permission after a member leaves; removed members lose access to current group activity.
- The complete customer requirements inventory describes the intended product. The separately labeled MVP subset is a proposed delivery boundary, not a customer-approved reduction in scope.
- The project constitution's Java 21, Maven, Java Swing, JUnit 5, PlantUML, object-oriented design, GUI/business-logic separation, and `BigDecimal` rules are implementation constraints. They do not replace or redefine customer requirements.

## Open Questions

1. What notification channels and delivery timing are required beyond making expense activity available to group members?
2. Should a payment greater than the payer's current balance be rejected, or recorded as a credit? Should the product initiate transfers, or only record settlements?
3. Must percentage shares total exactly 100%, or is a defined input tolerance permitted?
4. Which non-date report parameters are required for the initial release (for example, group, participant, or expense category)?
5. What historical or future expense and payment dates, if any, are disallowed?
