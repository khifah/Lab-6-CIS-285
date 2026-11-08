# Selected Architecture A: requirements traceability

Sources: [original request](../../UserRequirements.md), [inventory](../../requirements.md), [specification with FR/NFR/AC/SC identifiers](../../specs/001-split-expense-tracking/spec.md). [Architecture](./architecture.md) and [testing strategy](./testing-strategy.md) describe the proposed design and test cases. **Actual verification evidence for every row: none; no implemented application or executed requirement tests were found in the inspected project artifacts.** Documentation checks do not verify product behavior.

| Original ID / outcome | FR / NFR | Component / responsible module | Phase | Planned verification | Actual evidence | Open decisions |
|---|---|---|---|---|---|---|
| CR-001 accounts/login/private data | FR-001; NFR-001/004 | Account use cases, password adapter / Accounts | Local access MVP; complete email verification/reset later | T-04; AC-001, SC-001; actual verification/reset delivery acceptance later | None | Q-09, Q-15 |
| CR-002 create/manage groups | FR-002; NFR-001/003 | Group services, membership repositories / Groups | MVP | T-05/12; AC-002/013; immediate direct-add and denied non-member checks | None | Q-12, Q-13 |
| CR-003 expenses/equal and percentage splits/participants/receipt | FR-003/004/005; NFR-002/003 | Money/allocation, Expense service and receipt port / Expenses | MVP, percentage rounding gated | T-01/02/03/06/10; AC-003/004, SC-002 | None | Q-01, Q-06, Q-07 |
| CR-004 notices of expense creation | FR-009; NFR-001/006 | Notification port and future delivery adapter / Notifications | Deferred full product | T-14; AC-006; notice on creation, retries, recipient access and provider failure | None | Q-03 |
| CR-005 configurable expense approval | FR-006/007/008; NFR-001/003/006 | Review use cases, policy state machine and UnitOfWork / Approvals, Expenses, Balances | Single step MVP; ordered multi-step later | T-07/08/09; AC-005/008, SC-003 | None | Q-04, Q-12 |
| CR-006 pairwise balances | FR-010; NFR-002/003/005/006 | Ledger/posting repositories, BigDecimal netting / Balances | MVP | T-08/09/11/16; AC-007/012, SC-002/003/005 | None | Q-08, Q-10 |
| CR-007 payment recording and balance adjustment | FR-011; NFR-002/003/006 | Payment use case and atomic compensating postings / Payments, Balances | MVP; overpayment branch gated | T-03/09/11; AC-008, SC-002 | None | Q-02, Q-06, Q-10 |
| CR-008 expense/balance/payment reports with filters | FR-012/013; NFR-001/002/005 | Authorized query services and ledger summaries / Reporting | Deferred full product | T-13/16; AC-009/012, SC-004/005 | None | Q-05, Q-08 |
| CR-009 primary web platform | FR-014; NFR-001/004/005 | Future HTTPS adapter and browser UI / Presentation, all server use cases | Deferred primary product | T-17; AC-010; accepted browser workflows, not merely API availability | None | Q-15; web technology/hosting |

FR-015 is a cross-cutting specification requirement: T-12/AC-013 verify historical expense, decision, payment and posting preservation after membership removal. NFR-003 actor/time retention is checked by T-06–12 including corrections once specified; NFR-006 by T-09 across submission, approval and payment.

## Recorded specification clarifications, not implementation inventions

Payer belongs to selected participants (FR-003/AC-003); manager directly adds registered members with immediate access (FR-002/AC-002); entire equal remainder goes to payer (FR-004/AC-004); invalid expense amounts rejected without maximum (FR-003); verified email/password and reset required (FR-001/AC-001). Full account acceptance remains deferred despite local demo access.

## Non-functional and success evidence

| Requirement | Planned evidence / phase |
|---|---|
| NFR-001 privacy | T-04/05/10/12/13/14: invoke application boundaries directly, not only hidden UI controls |
| NFR-002 monetary accuracy | T-01/02/03/08/11; exact smallest-unit reconciliation; SC-002 requires 100% of approved expense/payment cases |
| NFR-003 auditability | Actor/time/source/operation reconstruction and immutable correction history in T-06–12 |
| NFR-004 usability | T-15/AC-011: at least 90% submit and identify status unassisted; SC-001 additionally 90% complete account/group/expense within five minutes; SC-006 90% identify debt direction and status |
| NFR-005 responsiveness | T-16/AC-012/SC-005: at least 95% of balance and filtered-report requests display within three seconds, up to 100 members/10,000 expenses per group, recorded normal environment. Filtered reports cannot be accepted before implementation. |
| NFR-006 reliability | T-09: same-key retry, changed-key payload conflicts, concurrent same-source actions, rollback and lost-response scenarios |

All AC-001–AC-013 and SC-001–SC-006 are represented above. Proposed/deferred decisions and their approval status remain in the [clarification log](./requirements-clarification-log.md). Java/Maven/Swing/SQLite/JUnit/BigDecimal/PlantUML are engineering decisions assessed separately in [constitution compliance](./constitution-compliance.md).
