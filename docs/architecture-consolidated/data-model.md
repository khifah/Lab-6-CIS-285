# Selected Architecture A: consolidated logical data model

Design only; entities below are not a generated schema. [Architecture](./architecture.md), [ERD](./diagrams/database-erd.puml), and [classes](./diagrams/class-diagram.puml) use the same posting model. IDs are opaque application-generated IDs, proposed UUID text. Money/percentages use Java BigDecimal and exact decimal TEXT in SQLite. Business dates are LocalDate; audit timestamps are UTC instants.

## Entities, ownership and relationships

| Entity / owner | Identity and important fields | Invariants and relationships |
|---|---|---|
| UserAccount / Accounts | userId; normalizedEmail, passwordHash, status, emailVerifiedAt?, createdAt | Unique normalized email; salted adaptive hash. Many memberships; pilot status/demo verification does not satisfy actual email acceptance. |
| Group / Groups | groupId; name, currencyCode, minorUnitScale, createdBy, createdAt | One currency. Proposed currency freeze after financial activity; changes require migration design. Creator references account. |
| Membership / Groups | membershipId; groupId, userId, role, joinedAt, leftAt? | At most one active group/user period, enforced with a unique partial index. Rejoin creates another period. Financial references survive removal. This refines A's original composite identity to preserve multiple periods. |
| ApprovalPolicy / Approvals | policyId; groupId, version, mode, createdAt, createdBy | Immutable policy version; unique group/version. Expense references exact policy ID; later manager changes do not rewrite a pending expense's snapshot. |
| ApprovalStep / Approvals | stepId; policyId, sequence, reviewerMembershipId | Unique policy/sequence; one required step for pilot, ordered steps later. Reviewer eligibility rechecked when acting. Departure/replacement remains open. |
| Expense / Expenses | expenseId; groupId, submittedBy, payerId, amountText, currencyCode, expenseDate, description, splitMethod, status, policyId, version, submittedAt, approvedAt?, receiptId? | Positive valid precision, no business maximum; payer included among active participants. Required details validated. PENDING→APPROVED after complete policy or PENDING→REJECTED. Immutable submitted financial details; corrections do not overwrite. |
| ExpenseShare / Expenses | (expenseId, participantId); percentageText?, allocatedAmountText | Nonnegative values, unique participant, total equals expense. Payer's share reconciles but never posts self-debt. Zero share creates no posting. |
| ReviewDecision / Approvals | decisionId; expenseId, stepId, reviewerId, decision, decidedAt, comment? | Unique accepted decision per expense/step; authorized current step only. Actor/time immutable. Repeated final actions cannot add financial effects. |
| Payment / Payments | paymentId; groupId, payerId, payeeId, amountText, currencyCode, paymentDate, description?, recordedBy, recordedAt, operationId | Positive, permitted precision, distinct eligible parties in same group/currency; recorder involved. Immutable record. Overpayment policy unresolved. |
| BalancePosting / Balances | postingId; groupId, sourceType, sourceId, sourceLineKey, fromUserId, toUserId, amountText, currencyCode, postedAt, actorId, reversalOfPostingId? | Positive debt increment from→to, no self-debt, immutable. Source types EXPENSE, PAYMENT, CORRECTION. Unique (sourceType,sourceId,sourceLineKey); references validated to matching group/currency. No zero postings. |
| FinancialCorrection / source owner, future workflow | correctionId; groupId, originalSourceType/Id, reason, authorizedBy, recordedAt, operationId | Proposed auditable correction header referencing immutable source. Original remains. Reversal/replacement postings reference correction and original posting where applicable. Authority, limits and workflow require approval; no pilot correction UI implied. |
| OperationRecord / Application | operationId; actorId, operationType, requestKey, payloadFingerprint, resourceId, resultCode, createdAt | Unique actor/type/key, atomically persisted with mutation. Same fingerprint returns original result; different fingerprint conflicts. Preserve financial-action keys with retained records to prevent delayed replay. Sensitive payloads not copied into logs. |
| AuditEvent / Application | auditId; actorId, groupId?, operationId, resourceType/Id, action, occurredAt | Append-only actor/time/target metadata sufficient to connect financial sequence; no credentials or receipt content. |
| Receipt / Expenses | receiptId; storageKey, originalName, contentType, byteSize, checksum, uploadedBy, uploadedAt, state | Expense has zero/one receipt; generated managed key. Validate content/type/size, require permitted expense access on retrieval. State supports staged/linked/missing recovery. Unlinked cleanup policy proposed. |
| Notification / Notifications, deferred | notificationId; groupId, expenseId, recipientId, eventType, createdAt, readAt?, deliveryStatus | Unique event/recipient if durable event IDs adopted. Other eligible members notified on expense creation, not only approval. Recheck access at delivery/read. Failure cannot discard source expense. |
| ReportQuery/ReportResult / Reporting, deferred | query: groupId, type, dateFrom/dateTo, filters; result: rows, totals, appliedFilters | Not mutable financial sources; authorization, valid range and empty zero totals required. Expense/payment filters use respective business dates. Historical balance semantics unresolved. |

Foreign keys enforce identities with deletion restricted for referenced history; no cascading financial deletion. Composite group/resource constraints or transactional validation prevent cross-group links. Expense shares/decision/source invariants spanning rows require domain validation within the same UnitOfWork. Generic posting source references require explicit repository validation, since a single polymorphic foreign key cannot enforce multiple source tables. DB constraints alone do not prove reconciliation.

## Money and allocation

Money encapsulates currency and BigDecimal; no binary floating-point path. Reject zero/negative expense/payment and unsupported precision before normalization. Parse decimal text directly and use compareTo for numeric equality. Do not encode monetary TEXT with SQLite NUMERIC affinity or aggregate with SQL SUM; arithmetic is Java BigDecimal. Persist canonical plain decimal strings at group scale and round-trip very large amounts in integration tests. This imposes no artificial business maximum, although finite resource failures must be explicit.

Equal split: base = total / participant count, rounded DOWN to minor-unit scale; payer receives base + (total − count × base), others receive base. USD 10.00/3 gives payer 3.34 and others 3.33. USD 0.01/3 gives payer 0.01 and others zero, with no zero posting. Selected active participants include payer and shares sum exactly to total.

Percentage validation rejects negative inputs and totals inconsistent with FR-004; exact 100 is the current spec baseline, tolerance unresolved. Residual-to-payer (original A proposal) and largest-remainder allocation are options, neither approved here. Percentage cases needing rounding are blocked until policy approval; do not silently ship one. Exact-representable percentage examples can be tested independently. Persist the chosen allocation-policy version when approved.

## Ledger direction and examples

Each positive posting X→Y increases what X owes Y. For a pair A,B:

`debt(A,B) = sum(amount for A→B) − sum(amount for B→A)`.

Aggregate within one group/currency using BigDecimal. Positive means A owes B; negative means B owes A; show one direction only. Approved expenses create participant→payer entries for positive non-payer shares. A payment from P to Q creates Q→P, compensating P's obligation. USD example: approved share A→B 12.00; A pays B 5.00, posting B→A 5.00; debt(A,B)=7.00. If an approved opposite expense adds B→A 2.00, debt becomes 5.00. Pending/rejected claims contribute zero. Payment overage handling remains open; this example does not approve credits.

Source line keys: expense uses participantId; payment uses a fixed settlement line; correction uses a stable reversal/replacement line ID. These permit multiple legitimate lines for one expense but prevent retries duplicating any line. Skip zero amounts. Original posting is never updated/deleted; an authorized reversal appends the opposite direction with its own source and reversal reference. Further corrected liability uses separate replacement lines. Correction workflow is deferred pending clarification, not a silent editing shortcut.

## Atomicity, reconciliation and history

Submit expense/shares/policy reference/audit/operation result atomically with no postings. Final approval writes last decision, APPROVED status, all expected postings, audit and operation result atomically. Payment writes its source, compensating posting, audit and operation result atomically. Rollback on any failure, including uniqueness failure. SQLite write reservation, state/version guards and source uniqueness ensure repeated/concurrent attempts cannot post twice; busy retries retain the same request key. See sequence diagrams and [tests](./testing-strategy.md).

Reconciliation reads a consistent snapshot and independently generates expected postings from immutable approved shares, payments and authorized corrections. Compare source-line sets, direction, currency and exact amounts; flag missing/extra/duplicate lines and postings for pending/rejected sources. Compare per-pair source expectation to ledger net. Retain an audit report; do not auto-create repair postings. Repair authority must be specified. Display balances from ledger, not from B's source-only formula; source aggregation is a verification oracle only.

Removal ends membership access, not account/expense/payment/decision history. Rejoining opens a new membership period; its authorization follows current roles without changing old decisions. Reports filter expenseDate/paymentDate as specified; historical opening/closing balance, late approval and correction dates need human decisions. No current report may treat unapproved expenses as debt. Financial policy proposals and status are tracked in [clarification log](./requirements-clarification-log.md).
