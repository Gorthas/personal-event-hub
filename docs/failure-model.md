# Failure Model

## Status

This document records the guarantees and known gaps of the approved synchronous v1 design. It does not claim retry, recovery, idempotency, or eventual-delivery behavior.

## Core guarantees

v1 is designed around four rules:

1. Authentication or validation failure creates no Event and causes no Telegram call.
2. Telegram is called only after Event persistence succeeds.
3. An Event is accepted only after its persistence succeeds.
4. Failures after Event persistence do not revoke Event acceptance.

These rules define what the Hub can state confidently. They do not remove ambiguity created by network failures or process crashes.

## Failure matrix

| Failure point | Event state | Telegram attempt | Delivery state | Source-facing meaning | Owner-facing meaning |
|---|---|---|---|---|---|
| Malformed HTTP input | Absent | None | Absent | Event not accepted | Nothing to show in Event history |
| Source authentication fails | Absent | None | Absent | Event not accepted | Nothing to show in Event history |
| Event validation fails | Absent | None | Absent | Event not accepted | Nothing to show in Event history |
| Event persistence explicitly fails | Not accepted | None | Absent | Event not accepted | No accepted Event |
| Telegram returns explicit error | Persisted and accepted | One failed attempt | `failed`, if stored | Event accepted | Failed Delivery with a sanitized reason |
| Telegram request times out or result is ambiguous | Persisted and accepted | One attempt with unconfirmed outcome | `failed`, if stored | Event accepted | Failed Delivery; actual external receipt may be unknown |
| Delivery persistence fails | Persisted and accepted | Completed | No record | Event accepted | `not recorded` |
| Process crashes after Event persistence and before a Delivery is stored | Persisted and accepted | None, incomplete, or unknown depending on timing | No record | Response may be lost | `not recorded` after restart |

## Meaning of History states

### `success`

The Hub stored a Delivery whose Telegram outcome was confirmed as successful.

### `failed`

The Hub stored a Delivery for which it could not confirm successful Telegram delivery. This includes explicit Telegram errors and ambiguous outcomes such as timeouts.

`failed` does not always prove that Telegram received nothing. For example, Telegram may have accepted a request while the response was lost.

### `not recorded`

The Event exists, but no Delivery record exists. Possible causes include:

- a process crash after Event persistence;
- a crash during or after the Telegram call;
- failure to persist a known Delivery result.

`not recorded` does not mean pending, scheduled, or automatically recoverable.

## Ambiguous Event persistence

A database or network failure can make the Event-persistence outcome uncertain from the caller's perspective. For example, PostgreSQL may commit while the response to the Hub is lost.

Because v1 has no idempotency or deduplication:

```text
Source sees failure or no response
        -> Source submits again
        -> the original Event may already exist
        -> a duplicate Event may be created
```

The Hub must not claim that retries are safe in this situation.

## Lost HTTP response after acceptance

The Hub may persist the Event and finish some or all downstream processing, but the Source may not receive the final HTTP response.

From the Hub's perspective, the Event is accepted. From the Source's perspective, the result is unknown. A repeated submission can create a duplicate because v1 has no idempotency contract.

## Telegram ambiguity

Timeout, connection loss, or a lost response can produce this state:

```text
Telegram may have accepted the message
Hub did not receive confirmation
Hub records failed, if it can record anything
```

The design favors an honest statement of unconfirmed success over claiming delivery. It does not retry, because a retry could produce a duplicate notification and v1 has no delivery idempotency mechanism.

## Crash gap between Event and Delivery

Event and Delivery are not presented as one atomic operation. Event persistence completes before the external Telegram call, and Delivery is stored afterward.

Consequently, a crash can leave:

```text
Event: persisted
Delivery: absent
```

This is why the domain relationship is `Event 1 -> 0..1 Delivery` and why History has `not recorded`.

## Delivery-persistence failure

If Telegram outcome is known but Storage cannot save Delivery:

- the Event remains accepted;
- ingestion remains successful at the Event contract level;
- the known result can be lost from persistent history;
- History later reports `not recorded`.

v1 has no recovery log or replay mechanism for reconstructing that Delivery.

## Explicit non-guarantees

v1 provides none of the following:

- automatic Telegram retry;
- recovery of missing Delivery records;
- idempotent Event submission;
- Event deduplication;
- delivery idempotency;
- eventual delivery;
- strict ordering;
- at-least-once or exactly-once delivery semantics;
- manual Delivery retry;
- multiple delivery-attempt history.

These are conscious boundaries, not features implied by the current design.

## Operational implications

- Producers should interpret success as Event acceptance, not Telegram success.
- Owners use History to distinguish recorded success, recorded failure, and missing outcome.
- Logs may contain developer-oriented diagnostics, but History contains sanitized user-facing errors.
- Reliability work must test cancellation, timeouts, graceful shutdown, concurrent requests, and each boundary above without silently upgrading the documented guarantees.
