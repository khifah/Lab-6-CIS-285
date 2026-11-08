# Architecture B data model

Proposed centralized PostgreSQL model; see [architecture.md](./architecture.md) for scope and traceability and [architecture.puml](./architecture.puml) for the maintained relationship diagram. All financial calculations use Java `BigDecimal`. Financial source rows and audit history are retained; balance and report results are query models.

## Entities and ownership

| Entity / owner | Key fields | Constraints and relationships |
|---|---|---|
| UserAccount / Accounts | userId, normalizedEmail, passwordHash, verifiedAt?, status, createdAt | Unique email, adaptive salted password hash. Private profile fields never appear in group DTOs. |
| AccountToken / Accounts | tokenId, userId, tokenHash, purpose, expiresAt, consumedAt? | Verification/reset; hashed secret, single use. Many tokens per user. |
| Session / Accounts | sessionId, userId, secretHash, expiresAt, revokedAt? | Opaque credentials; revocable server session. |
| Group / Groups | groupId, name, currencyCode, minorUnitScale, managerId, createdAt | One immutable currency once financial activity exists; no currency conversion. |
| Membership / Groups | membershipId, groupId, userId, role, joinedAt, leftAt? | Membership periods retained; at most one active period per group/user. Rejoining creates a new period. Removal does not cascade-delete financial records. |
| ApprovalPolicy / Approvals | policyId, groupId, version, createdBy, createdAt | Immutable version; unique group/version. |
| PolicyStep / Approvals | policyId, sequence, reviewerUserId | Unique policy/sequence. Reviewer must be an eligible member when acting. Pilot has one step. |
| Expense / Expenses | expenseId, groupId, submittedBy, payerId, amount, expenseDate, description, splitMethod, policyId, status, submittedAt, approvedAt?, version, receiptId? | Positive amount, permitted currency precision, payer among selected participants, all eligible members at submission. Status PENDING/APPROVED/REJECTED. No maximum business amount. |
| ExpenseShare / Expenses | expenseId, participantId, percentage?, allocatedAmount | Composite key expense/participant; nonnegative shares, sum equals expense amount. Selected participants include payer; zero shares are allowed. Amounts and percentages immutable after submission. |
| ReviewDecision / Approvals | decisionId, expenseId, stepSequence, reviewerId, decision, comment?, decidedAt | Unique expense/step accepted decision; preserve actor/time. Ordered authorized decisions; rejection is terminal. Current authority is rechecked, not inferred from stale client data. |
| Receipt / Expenses | receiptId, storageKey, contentType, checksum, uploadedBy, uploadedAt, state | Expense has zero or one receipt. Bytes stored on server, never a client filesystem path. Validate content and authorize retrieval. Staged uploads are finalized on linking; clean unlinked uploads after a retention period. |
| Payment / Payments | paymentId, groupId, payerId, payeeId, amount, paymentDate, description?, recordedBy, recordedAt | Positive amount, distinct parties, currency matches group; recorder is involved and authorized. Immutable original record. Overpayment handling remains open. |
| FinancialCorrection / Payments or Expenses | correctionId, groupId, sourceType, sourceId, debtorId, creditorId, signedDebtDelta, effectiveDate, reason, authorizedBy, recordedAt | Proposed future auditable compensation, not a delivered correction workflow. Same group/currency and original reference required. Only an authorized committed correction contributes; workflow/permissions need specification. |
| IdempotencyRecord / Application | actorId, operation, requestKey, fingerprint, resourceId, resultCode, createdAt | Unique actor/operation/key; stored atomically with mutation. Retain financial-action keys for the financial record lifetime to prevent delayed duplicate effects. |
| AuditEvent / Application | auditId, groupId?, actorId, operation, resourceType, resourceId, occurredAt | Append-only audit metadata; never log passwords, tokens, or receipt bytes. |
| OutboxEvent / Notifications (deferred) | eventId, groupId, expenseId, eventType, payload, createdAt, attempts, nextAttemptAt, processedAt? | Written with expense creation, worker retries after commit. No external broker required. |
| Notification / Notifications (deferred) | notificationId, eventId, recipientId, groupId, expenseId, readAt?, deliveryState | Unique event/recipient. Capture other active members at creation, recheck permission before delivery/read; removed members cannot retrieve group content. |
| ReportQuery/Result / Reporting (deferred) | groupId, reportType, dateFrom, dateTo, filters; rows, totals, appliedFilters, snapshotAt | Transient read model. Authorize before querying; no matching activity returns empty rows/zero totals. |

Foreign keys enforce identities; composite group/resource references prevent cross-group links. Cross-row invariants (share reconciliation, payer inclusion, snapshot completion) are enforced by domain behavior and a transaction before commit. Lock the expense for decisions; coordinate group membership mutations and financial authorization using a group lock in a consistent order. Optimistic versions detect stale edits. No cascade deletes of audit or financial history.

## Exact money and splits

Money encapsulates `BigDecimal amount` and currency. Parse from decimal strings, compare numerically with `compareTo`, and reject excess fractional precision before normalizing to the currency scale. Never use `double` or `float`. PostgreSQL unconstrained `NUMERIC` avoids an artificial fixed application amount cap; its finite storage limits still exist and failures must be explicit. JDBC uses decimal accessors; JSON uses decimal strings.

Equal split: for positive total T and n selected participants, calculate base = T / n rounded DOWN to currency scale. Assign base to every participant, then add T − n × base entirely to the payer. USD 10.00 across three participants gives payer 3.34 and others 3.33 each. Shares always sum to T.

Percentage split: reject negative percentages and require a sum of exactly 100 for the provisional design; tolerance remains unresolved. Proposed rounding is DOWN per share followed by the entire residual to the payer, with the rounding policy recorded on the expense for reproducibility. This is a design proposal requiring review, particularly when payer percentage is zero; it is not an approved product rule. Do not implement tolerance or another residual policy silently.

## Pairwise balances without balance postings

For an ordered pair A, B in one group, define positive `D(A,B)` to mean A owes B:

`D(A,B) = shares owed by A on approved expenses paid by B − shares owed by B on approved expenses paid by A − payments from A to B + payments from B to A + authorized signed correction deltas for this direction`.

Only non-payer shares contribute. A payer's own share creates no self-debt. `D(B,A) = −D(A,B)`; present the positive direction once. Never sum balances across currencies or unrelated groups. Example: A owes B 12.00 from approved shares; A records a 5.00 payment to B; remaining debt is 7.00. A pending 9.00 expense contributes zero. Opposite approved debt of 2.00 reduces 7.00 to 5.00.

There is no `BalancePosting` or editable `Balance` table in B. Query the source records under one database snapshot. The final approval transaction updates status and approvedAt, making existing shares eligible without copying them. Payment commit likewise changes the next query. Corrections preserve original records; until their workflow is specified, corrections are not an exposed pilot action.

Indexes start with memberships `(groupId,userId)` for active access, expenses `(groupId,status,expenseDate)`, shares `(expenseId,participantId)`, payments `(groupId,paymentDate,payerId,payeeId)`, decisions `(expenseId,stepSequence)`, and unique idempotency keys. Measure before introducing a materialized balance cache. Any future cache must be rebuildable and must not silently weaken read-after-write consistency.

## Time, reporting, and history

Business dates use `LocalDate`; recorded/approved timestamps use UTC instants. Expense reports filter expenseDate and payment reports filter paymentDate with inclusive date bounds. Report output identifies status and applied filters; pending and rejected expenses can appear in activity reports but never in financial totals.

Proposed balance-report semantics: closing balance through dateTo; dateFrom selects movements and opening balance is the accumulated debt before dateFrom. Show opening + movements = closing, with expense recognition no earlier than full approval and correction effective dates explicit. This is a proposed clarification, not a resolved specification requirement; agree recognition/as-of rules before implementing historical balance reports. Current balances always use currently approved source records. Current members may see retained group history, while removed members lose current group access. Membership removal never erases payer/reviewer references.

Database integration tests must prove rollback, source-filter correctness, duplicate/concurrent reviews and payments, exact decimal round trips, and preserved history. These are planned checks; this documentation adds no schema or runnable tests.
