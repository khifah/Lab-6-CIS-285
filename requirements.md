# SplitSmart Customer Requirements Inventory

**Source**: [UserRequirements.md](./UserRequirements.md)  
**Scope**: Original customer-requested product capabilities. This inventory does not include implementation decisions.

| Requirement ID | Customer requirement |
|---|---|
| CR-001 | Each person has an individual user account, can log in, and can access their own expense and balance information. |
| CR-002 | Users can create and manage groups of people, such as roommates, trip participants, or party participants, to track shared expenses and balances. |
| CR-003 | Users can create expenses with an amount, date, description, equal or percentage-based split, optional receipt image, and selected involved users. |
| CR-004 | Users are notified when other group members create an expense. |
| CR-005 | Expenses are reviewed and approved before they are added and used to adjust balances; a group may use one review step or multiple approval levels. |
| CR-006 | The system tracks the balance owed by each user to every other user, taking entered expenses into account. |
| CR-007 | Users can record payments made to settle balances, and balances reflect those payments. |
| CR-008 | Users can view reports and summaries of expenses, balances, and payments, including custom date ranges and other parameters. |
| CR-009 | The primary supported platform is web-based. |

## Platform Scope Note

The source specifies a primary web-based platform. It does not specify Android or iOS support. The separately requested Java Swing MVP is a desktop implementation decision and does not replace or satisfy CR-009.
