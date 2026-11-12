# Implementation Plan

## Status

Architecture and v1 design are complete. No implementation vertical has started.

The project will be built as functional verticals. Each vertical should leave a coherent user-visible capability rather than completing an entire technical layer in isolation.

## Ordered implementation roadmap

| Step | Vertical | Outcome |
|---|---|---|
| 0 | Foundation | Runnable Go service with configuration, composition, HTTP server, PostgreSQL connection, and migrations |
| 1 | Source onboarding | Owner can create a Source, receive its credential once, and list Sources |
| 2 | Event acceptance | Authenticated Source can submit a valid Event; persistence defines acceptance |
| 3 | Telegram delivery | Accepted Event triggers one Telegram attempt; the result is recorded as Delivery when possible |
| 4 | History | Owner can list and inspect Events with `success`, `failed`, and `not recorded` semantics |
| 5 | Credential lifecycle | Owner can rotate one Source credential without losing Source identity or history |
| 6 | Export and hard delete | Owner can export Source history and permanently delete Source, Events, and Deliveries |
| 7 | Reliability pass | Context propagation, timeouts, cancellation, graceful shutdown, concurrency, and failure paths are reviewed end to end |
| 8 | Testing and completion | Boundaries and critical failure cases receive systematic tests; the v1 end-to-end flow is verified |

## Step 0 — Foundation

Goal: create the smallest runnable base on which real verticals can be built.

Planned work:

- initialize the Go application;
- define required application configuration;
- create Startup / Composition;
- connect PostgreSQL;
- introduce migrations and the initial schema required by the first vertical;
- start the HTTP server;
- add a minimal readiness or startup check.

Exit condition: the application starts from configuration, validates required settings, connects to PostgreSQL, applies migrations, and serves HTTP with dependencies assembled externally.

No Source, Event, Telegram delivery, or History behavior is claimed at this stage.

## Step 1 — Source onboarding

Goal: register the first real producer.

Planned capabilities:

- create a Source with a unique name;
- generate and store credential verification material;
- disclose the plaintext credential once;
- protect the operation with Owner authentication;
- list Sources without disclosing credentials.

Exit condition: the owner can create `github-ci`, store its credential in GitHub Actions secrets, and later see the registered Source without being able to re-read the secret.

## Step 2 — Event acceptance

Goal: make the Source-to-Hub half of the product real before adding Telegram.

Planned flow:

```text
GitHub Actions
  -> Source authentication
  -> Event validation
  -> internal Event construction
  -> PostgreSQL persistence
  -> accepted response
```

Exit condition: a successful response proves the Event already exists in PostgreSQL. Bad credentials, invalid input, and Event-persistence failure create no accepted Event and trigger no Telegram call.

## Step 3 — Telegram delivery

Goal: complete the first end-to-end product flow.

Planned work:

- add Delivery persistence;
- add the Telegram boundary and message formatting;
- set an external-call timeout;
- call Telegram only after Event persistence;
- translate confirmed success and known failures into Delivery;
- attempt to persist the result without changing Event acceptance.

Exit condition: a real GitHub Actions Event is accepted, stored, sent to Telegram once, and accompanied by `Delivery(success|failed)` when Delivery persistence succeeds.

## Step 4 — History

Goal: make the Event and Delivery flow observable to the owner.

Planned capabilities:

- list a bounded set of recent Events ordered by `received_at`;
- filter by `success`, `failed`, or `not recorded`;
- inspect Event details and known Delivery information;
- protect History with Owner authentication.

Exit condition: the owner can diagnose accepted Events, including Telegram failure and missing Delivery records.

## Step 5 — Credential lifecycle

Goal: replace Source access without destroying its identity or data.

Planned capabilities:

- rotate the credential of one Source;
- invalidate the old credential;
- disclose the new credential once;
- preserve Source, Events, and Deliveries.

Exit condition: old access fails, new access works, and history remains unchanged.

## Step 6 — Export and hard delete

Goal: complete the Source data lifecycle.

Planned capabilities:

- export one Source's Events and existing Deliveries;
- hard-delete the Source and all owned data;
- ensure its credential can no longer authenticate.

Exit condition: export is non-mutating, and deletion leaves no Source, Event, Delivery, or valid Source credential inside the Hub.

The export format remains an implementation decision.

## Step 7 — Reliability pass

Goal: review the working system against its documented synchronous guarantees.

Planned focus:

- context propagation;
- external-call and database timeouts;
- request cancellation;
- graceful shutdown;
- concurrent requests and credential rotation behavior;
- failures at every Event and Delivery boundary;
- configuration validation and safe logging.

This pass must not accidentally claim retry, recovery, idempotency, strict ordering, or eventual delivery.

## Step 8 — Testing and v1 completion

Goal: systematically verify the complete v1 contract.

Planned coverage:

- Coordinator tests for stop/continue rules;
- Storage integration tests for Source, Event, Delivery, relationships, rotation, export, and deletion;
- HTTP tests for Management, Ingestion, History, authentication, validation, and error mapping;
- Telegram boundary tests for confirmed success, explicit error, timeout, ambiguous failure, and formatting;
- end-to-end smoke test from Source creation through History;
- clean-environment startup, migrations, configuration documentation, and secret checks.

The implementation should write focused tests during earlier verticals. This final pass completes and systematizes the strategy rather than postponing all testing until the end.

## Completion criteria for v1

v1 is complete when:

- the main GitHub Actions-to-Hub-to-Telegram flow works;
- accepted Events are durable before delivery;
- critical failure paths are tested;
- Source creation, use, rotation, export, and hard deletion work end to end;
- History accurately distinguishes `success`, `failed`, and `not recorded`;
- security boundaries and secret-handling rules are verified;
- startup, migrations, configuration documentation, and clean-environment operation are confirmed;
- documented limitations still match actual behavior.
