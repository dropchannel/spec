# Observability

**Related:** `heartbeat.md`, `telemetry.md`

---

## Overview

DropChannel provides two distinct observability mechanisms. They operate at different
layers, serve different consumers, and answer different questions. Neither is a
substitute for the other.

| | Heartbeat | Telemetry |
|---|---|---|
| **Layer** | Protocol liveness | External monitoring |
| **Consumer** | Participants in the pipeline | Monitoring tools, operators |
| **Transport** | Meta slots on payload channel positions | Dedicated side-channel (`telemetry-` prefix) |
| **Answers** | "Is my upstream neighbor still alive?" | "What is the full topology and live state?" |
| **Reaction** | Protocol-defined (participants can infer stall location, `closing` intent) | None — purely observational |
| **Requires external tooling** | No | Yes |
| **Encrypted** | No (DRIP_KEY deferred) | No |
| **Scope** | Per-hop, within a single pipeline | All participants across all pipelines in a deployment |

---

## Heartbeat — protocol liveness

Heartbeat is part of the protocol's operational behavior. Each node runs a heartbeat
cycle concurrently with its primary slot state machine, reading upstream liveness
signals and relaying them forward. Clients write status signals at transitions.

Because nodes relay upstream content, a participant at any position can read the full
liveness chain of all upstream participants from a single meta slot file. A broken
chain — indicated by the `HEARTBEAT_NOT_FOUND` sentinel — pinpoints the hop where
liveness has been lost.

Heartbeat works without any external tooling. A client can read its own recv-side
meta slot and determine the health of the entire upstream pipeline.

Heartbeat also carries the `closing` status signal: a client writing `closing`
before shutdown makes intentional departure distinguishable from unexpected
disappearance.

→ See `heartbeat.md` for the full specification.

---

## Telemetry — external observability

Telemetry is outside the protocol's operational flow. Each participant periodically
writes a self-describing blob to a dedicated telemetry side-channel, reporting its
role, channel addresses, slot occupancy, and current state. A monitoring tool
reconstructs the full topology by joining participants on their channel IDs — no
config files required.

Telemetry provides information that heartbeat cannot: the full graph structure of the
deployment, slot occupancy state at every hop, and multi-pipeline visibility in a
single view. It is the data source for operator-facing monitoring tools.

Telemetry works across all implementations. Any participant — regardless of language
or runtime — can emit telemetry by writing the specified JSON blob to the shared
telemetry channel.

→ See `telemetry.md` for the full specification.

---

## Choosing between them

They are not alternatives. A well-instrumented DropChannel deployment runs both:

- **Heartbeat** provides lightweight, always-on liveness signaling that participants
  can act on directly. It requires no external infrastructure beyond the storage
  backend already in use.

- **Telemetry** provides the rich, operator-facing view needed to understand and
  diagnose the system. It requires a monitoring tool reading the telemetry channel.

If only one can be implemented, heartbeat has lower overhead and no external
dependencies. Telemetry is most valuable once the system is running in an environment
where operators need ongoing visibility.

---

## What neither mechanism provides

- **Payload integrity verification.** Neither heartbeat nor telemetry validates that
  payload blobs are correctly encrypted or uncorrupted. That is the responsibility
  of the endpoint crypto layer.

- **Authentication of participants.** A participant can write any `participant_id`
  or UUID it chooses. Neither mechanism validates participant identity against an
  authoritative registry.

- **Alerting or notification.** Both mechanisms expose state. Acting on that state —
  sending alerts, triggering retries, notifying operators — is left to the application
  or monitoring layer.
