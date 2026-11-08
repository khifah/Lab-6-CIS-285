# Platform Boundary Contract

## Interim Swing MVP

- The Java Swing desktop application has no external network API in this MVP.
- Swing views/presenters submit typed user intent to application use cases; they do not access SQL or implement business rules.
- Application use cases own authorization, validation, transaction boundaries, and explicit success/failure results.
- Repository and receipt-storage ports isolate local SQLite/filesystem adapters.
- MVP behavior is limited to the subset in `spec.md`; local persistence does not imply multi-device synchronization, web access, notifications, or production identity management.

## Future full product

- Web is the primary platform in customer requirement CR-009. Android and iOS are additional future clients requested in the planning prompt, not requirements in the original `UserRequirements.md`.
- Future clients communicate with a server-owned, versioned HTTPS API. The server, not the clients, is authoritative for authentication, group membership authorization, expense approval, balance postings, payments, and report filtering.
- Client requests must not be trusted to assert approval completion, account ownership, group membership, or calculated balances. The server validates and recalculates those values.
- Notifications and receipt access are mediated by server-side adapters, with per-recipient authorization.

## Intentionally unspecified

No HTTP endpoints, payload schemas, authentication protocol, error envelope, API versioning policy, notification channels, hosting topology, or client UI frameworks are defined by this baseline. Specify these contracts when the web/API implementation is planned; do not infer them from the Swing UI or local persistence model.
