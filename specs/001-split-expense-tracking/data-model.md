# SplitSmart Data Model

**Selected Architecture A consolidation:** See the [detailed consolidated model](../../docs/architecture-consolidated/data-model.md) and [change rationale](../../docs/architecture-consolidated/architecture-decision-record.md). This canonical summary is aligned to the local Swing/SQLite posting-ledger design; original architecture alternatives remain unchanged. All refinements are documentation, not implemented schema or verified behavior.

This is the proposed logical model for the architecture in [plan.md](./plan.md). It distinguishes immutable business records from derived balance/report views. All amount fields use `BigDecimal`; each group has one currency with an explicit minor-unit scale.

## Entities

| Entity | Important fields | Validation and behavior |
|---|---|---|
| **UserAccount** | `id`, normalized `email`, `passwordHash`, `emailVerifiedAt?`, `createdAt`, `status` | Email is unique. Store a salted adaptive password hash, never a plaintext password. Verification/reset delivery requires a future shared service; local pilot identity is not a production identity boundary. |
| **Group** | `id`, `name`, `currencyCode`, `minorUnitScale`, `createdBy`, `createdAt`, `status` | One currency per group. Currency scale controls allowed expense/payment precision and allocation rounding. |
| **Membership** | `membershipId`, `groupId`, `userId`, `role`, `joinedAt`, `leftAt?`, `status` | Retain distinct membership periods on rejoin; unique active `(groupId, userId)`. A manager directly adds registered users. Removal ends current access but does not erase historical records. |
| **ApprovalPolicy** | `id`, `groupId`, `mode`, `version`, `createdAt` | `mode` is single-step in the Swing MVP; full product may configure multiple ordered approval steps. Preserve the policy version used for each submitted expense. |
| **ApprovalStep** | `id`, `policyId`, `sequence`, `reviewerMembershipId` | Defines an authorized reviewer and step order. Single-step policies have one required step. |
| **Expense** | `id`, `groupId`, `submittedBy`, `payerUserId`, `amount`, `currencyCode`, `expenseDate`, `description`, `splitMethod`, `status`, `policyVersion`, `submittedAt`, `approvedAt?`, `receiptId?` | Amount must be positive and no more precise than group currency scale. Payer must be an active participant. Pending and rejected expenses never create balance postings. No maximum amount is imposed. |
| **ExpenseShare** | `expenseId`, `participantUserId`, `percentage?`, `allocatedAmount` | One row per selected active participant; nonnegative shares sum exactly to total. Equal remainder goes to payer. Percentage baseline is 100%; tolerance and residual policy remain unresolved, blocking affected rounded-percentage implementation. Zero shares create no postings. |
| **ReviewDecision** | `id`, `expenseId`, `approvalStepId`, `reviewerUserId`, `decision`, `decidedAt`, `comment?` | Decision is approve or reject. Reviewer must be authorized for the current step. Retain all decisions; do not overwrite history. Rejection prevents posting. |
| **Receipt** | `id`, `storageKey`, `mediaType`, `originalName`, `uploadedBy`, `uploadedAt`, `checksum` | Optional and linked to an expense. Store metadata in the relational database and image bytes in managed storage; validate content and access on retrieval. Exact size/type limits remain deployment policy. |
| **Payment** | `id`, `groupId`, `payerUserId`, `payeeUserId`, `amount`, `currencyCode`, `paymentDate`, `description?`, `recordedBy`, `recordedAt`, `idempotencyKey?` | Amount must be positive; payer and payee must be valid group participants. Preserve record and audit fields. Whether overpayment is rejected or becomes a credit remains open. |
| **BalancePosting** | `id`, `groupId`, `sourceType`, `sourceId`, `sourceLineKey`, `fromUserId`, `toUserId`, `amount`, `postedAt`, `actorId`, `reversalOfPostingId?` | Append-only positive debt increment from→to; unique `(sourceType,sourceId,sourceLineKey)`. No self/zero entry. Approval/payment source and postings commit atomically. Corrections use authorized compensating records; authority/workflow remains open. |
| **Notification** | `id`, `groupId`, `recipientUserId`, `expenseId`, `eventType`, `createdAt`, `readAt?`, `deliveryStatus` | Full-product component. Recipient must be a permitted group member. The delivery channel and timing remain open. |
| **ReportQuery / ReportResult** | query: `groupId`, `reportType`, `dateFrom`, `dateTo`, optional filters; result: records/totals and applied filters | Read models, not mutable business records. Apply group authorization and report-date semantics (expense date for expenses, payment date for payments). Empty matches return zero totals. |

## Relationships

```text
UserAccount 1 --- * Membership * --- 1 Group
Group 1 --- * Expense
Expense 1 --- * ExpenseShare * --- 1 UserAccount
Expense 1 --- * ReviewDecision
Group 1 --- * ApprovalPolicy 1 --- * ApprovalStep
Expense 0..1 --- 1 Receipt
Group 1 --- * Payment (payer and payee each reference UserAccount)
Expense approval / Payment 1 --- * BalancePosting
Group 1 --- * Notification * --- 1 UserAccount
```

`Membership.leftAt` preserves historical participation. Access checks use current membership for current group activity; access to historic records follows the permission rules in the feature specification.

## Balance calculation

For each approved expense, retain all allocations; payer's own share creates no self-debt. Each positive non-payer share posts participant→expense payer. Posting X→Y means an increase of X's debt to Y. Payment payer P to payee Q therefore posts **Q→P**, compensating P's debt. For pair A,B, debt(A,B) = sum(A→B) − sum(B→A), using BigDecimal within one group/currency. A owes B 12 and pays B 5: net debt is 7, not 17. Skip zero shares.

SQLite stores normalized exact decimal TEXT, not REAL, with Java BigDecimal parsing/aggregation and no artificial business amount cap. OperationRecord keys/fingerprints, AuditEvent, source-line uniqueness and source reconciliation are detailed in the linked consolidated model. Proposed correction headers preserve original sources; a correction workflow cannot be inferred as approved. Membership changes preserve prior periods and all financial references.

Use database transactions to write the terminal approval decision and its expense postings atomically. Use a unique source reference/idempotency constraint so retries cannot duplicate a posting. Never create postings for pending/rejected expenses. Do not store an independently editable balance as the source of truth; an optional cache must be rebuildable from postings.

## State and validation rules

- Expense lifecycle: `PENDING -> APPROVED` after all required approvals, or `PENDING -> REJECTED`. Only approved expenses post balance entries.
- Approval policy version and decisions are immutable audit facts. A correction is a new compensating record or replacement submission, not silent mutation.
- Split percentages are not accepted with tolerance until the open question in `spec.md` is resolved.
- Report date range must be valid; filtering and returned totals must respect group access. Report filters must be echoed in results.
- Group currency conversion, payment overage policy, allowed historical/future dates, and receipt limits remain unresolved or out of scope as stated in `spec.md`.
