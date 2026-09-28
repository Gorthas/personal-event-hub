# API Boundaries

## Status and level of detail

The v1 API surface is approved at the semantic-operation level. Concrete endpoint paths, HTTP methods, status codes, headers, request/response schemas, pagination mechanics, and export format are intentionally not specified yet.

The design has three external API areas with two separate caller identities:

```text
Owner
  -> Management API
  -> History API

Registered Source
  -> Ingestion API
```

These API areas are conceptual boundaries. They do not imply separate servers or processes.

## Management API

Management operations are protected by the Owner credential.

### Create Source

Purpose: register a new external producer.

Input:

- unique Source name.

Successful outcome:

- a Source is created;
- a Source-specific credential is generated;
- the plaintext credential is disclosed once for configuring the external producer.

The exact credential representation and response schema are deferred.

### List Sources

Purpose: show which producers are registered.

The list exposes stable Source identity and name. It does not expose Source credentials. Counts, last-event data, delivery statistics, and other dashboard fields are outside v1 scope.

### Rotate Source credential

Purpose: recover from a lost or compromised Source credential without deleting the Source or its history.

Successful outcome:

- the old credential becomes invalid;
- one new credential becomes current;
- the new plaintext credential is disclosed once;
- Source identity, name, Events, and Deliveries remain unchanged.

v1 has one active credential per Source. Parallel keys, expiry schedules, and credential history are not part of the design.

### Export Source history

Purpose: copy a Source's Events and existing Deliveries outside the Hub, especially before hard deletion.

Export is read-only. It does not delete, archive, disable, or mutate the Source. Import and restore are outside v1 scope. The export format remains open.

### Delete Source

Purpose: permanently remove a connection and all data owned by it.

Successful outcome:

- Source is removed;
- its credential becomes unusable;
- all of its Events are removed;
- all Deliveries belonging to those Events are removed.

This is hard delete, not archive or soft delete. A user interface should warn the owner before invoking it, but the API operation itself has destructive semantics once called.

## Ingestion API

Ingestion is authenticated with the credential of one registered Source.

### Submit Event

Purpose: accept one generic event from an external producer.

Input data:

| Field | Requirement | Meaning |
|---|---|---|
| `title` | Required, non-empty | Short description of what happened |
| `message` | Required, non-empty | Human-readable details |
| `occurred_at` | Required, structurally valid timestamp | When the Source says the event occurred |

Successful outcome:

- authentication and validation have passed;
- the Event has been persisted;
- the Event is accepted by the Hub;
- the response can identify the accepted Event.

The response does not disclose Telegram success, failure, or diagnostics. Those results belong to owner-facing History.

Failure semantics:

- authentication failure: no Event is created;
- validation failure: no Event is created;
- Event-persistence failure: no acceptance is reported and Telegram is not called;
- Telegram or Delivery-persistence failure after Event persistence: the Event remains accepted.

## History API

History operations are protected by the Owner credential.

### List Event history

Purpose: show a bounded set of recent accepted Events, ordered by `received_at`.

The summary view includes enough information to identify the Event, Source, title, occurrence time, and derived delivery state. The exact page or result limit remains an implementation question.

### Filter by delivery state

Supported semantic filters:

- `success`;
- `failed`;
- `not recorded`;
- no filter, meaning all available recent Events.

`not recorded` means no Delivery record exists for the Event. It is not a stored Delivery result.

### Get Event details

Purpose: inspect one accepted Event and the known result of its Telegram attempt.

The detail view combines:

- Source identity or name;
- title and message;
- `occurred_at` and `received_at`;
- Delivery result and `attempted_at`, if recorded;
- the human-readable failure reason for a failed Delivery;
- `not recorded` when Delivery is absent.

Events cannot be edited or deleted individually.

## Application configuration boundary

The following values do not belong to the runtime API:

- PostgreSQL connection settings;
- Telegram bot credential;
- the single Telegram target;
- Owner credential;
- HTTP/server settings.

They are supplied externally when the application starts. Changing Telegram configuration in v1 requires changing the external configuration and restarting the application.

## Operations intentionally absent from v1

- Source rename or general update;
- Source archive or disable;
- repeated reading of an existing Source secret;
- mandatory standalone Source-details operation;
- Event edit or individual delete;
- Delivery edit, manual retry, or attempt history;
- Telegram configuration management at runtime;
- Destination or Route management;
- Source-based History filter;
- text search, date-range search, or analytics;
- fully specified pagination;
- import or restore of exported data.

These omissions keep the API tied to approved user scenarios rather than a speculative full CRUD surface.
