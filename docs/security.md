# Security

## Status

This document defines the approved v1 trust and credential model. Exact cryptographic algorithms, libraries, header formats, and storage representations remain implementation decisions for the Security work.

## Trust boundaries

v1 has two independent authenticated caller roles:

| Caller | Credential | Allowed area |
|---|---|---|
| Registered Source | Its own Source credential | Submit Events for that Source |
| Owner | Separate Owner credential | Manage Sources and inspect/export History |

A Source credential must not authorize Management or History operations. The Owner credential is not used as a producer identity.

## Source credentials

Each Source has its own credential. Authentication establishes Source identity; the Hub does not trust a Source identifier supplied inside the Event payload.

The approved lifecycle is:

```text
Create Source
  -> generate credential
  -> store only the material needed to verify it
  -> disclose plaintext once

Rotate credential
  -> invalidate old credential
  -> generate and store new verification material
  -> disclose new plaintext once

Delete Source
  -> remove Source and credential verification data
```

The final implementation must use securely generated credentials, safe verification/comparison, and a representation that does not require recoverable plaintext Source secrets. The design does not yet select the algorithm or library.

### Rotation guarantees

- Exactly one Source credential is current in v1.
- Rotation preserves Source identity and all history.
- The old credential no longer authenticates after successful rotation.
- The new plaintext credential is not available for later re-reading.

Multiple active keys, overlap windows, expiration dates, credential labels, and key history are outside v1 scope.

## Owner credential

Management and History operations use a credential separate from every Source credential. It is provided through external application configuration rather than stored as a Source.

The Owner credential protects operations that can expose history, rotate access, export data, or permanently delete a Source. The exact HTTP authentication form and verifier representation are deferred.

v1 does not define multiple owners, user accounts, roles, sessions, or fine-grained authorization.

## Telegram credential and target

The Telegram bot credential and single target are deployment configuration, not domain entities and not manageable through the v1 API.

They are loaded at startup and made available only to the Telegram boundary. Changing either value requires updating external configuration and restarting the application.

The Hub validates that required configuration is present and structurally acceptable before it reports readiness. The design does not require a live Telegram network probe on every startup.

## Configuration and secret handling requirements

- Real credentials must remain outside version control.
- A future example configuration file may document required names but must contain no usable secrets.
- Secrets must not appear in normal application logs or owner-facing history.
- Source list and History responses must not expose Source credentials.
- Delivery errors must be human-readable and sanitized rather than raw external responses or internal stack traces.
- Public deployment requires HTTPS so credentials and Event data are protected in transit.
- GitHub Actions stores the relevant Source credential in its secret mechanism rather than in the workflow source.

The exact configuration loader and deployment secret store have not been selected.

## Authentication and validation are different

Authentication answers:

> Which registered Source is making this request, and is its credential valid?

Validation answers:

> Does the authenticated Source's Event input satisfy the Hub's contract?

Both must pass before Event construction and persistence. A valid credential does not make an invalid payload acceptable, and a valid payload does not bypass Source authentication.

## Data deletion and confidentiality

Hard-deleting a Source removes its credential verification data, Events, and Deliveries. The Hub keeps no soft-deleted or hidden archival copy in its own domain model.

An owner may export history before deletion. Once exported, that copy is outside the Hub's lifecycle and must be protected by the environment in which it is stored.

## Security work deferred to implementation

The following are required decisions for implementation, not claims about existing code:

- secure random credential format and entropy;
- verifier or hashing approach;
- safe credential comparison;
- Owner credential verification mechanism;
- HTTP credential transport convention;
- secret injection and local-development configuration workflow;
- log redaction and error sanitization checks;
- tests for invalid, old, missing, and cross-boundary credentials.
