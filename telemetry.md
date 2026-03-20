# Telemetry

**Status:** Draft  
**Prefix:** `telemetry-`  
**Protocol:** Conveyer (overwrite-always, no ACK)  
**Related:** `protocol-registry.md`, `conveyer-protocol/README.md`, `channel-provider.md`

---

## Purpose

The telemetry channel is a dedicated, always-on observability surface for DropChannel
participants. Each participant — originator, terminator, or node — periodically writes
a self-describing blob to its telemetry slot describing its current role, channel
addresses, and slot occupancy state.

A monitoring tool can reconstruct the full topology and live state of a DropChannel
system by reading the telemetry channel alone, without access to payload channels,
config files, or out-of-band communication.

**Telemetry is not:**

- A control channel. No participant changes its behavior based on another's telemetry.
- A heartbeat channel. Heartbeat (when specified) serves liveness signaling with
  protocol-defined response semantics. Telemetry is purely observational.
- A payload channel. Payload content is never included in telemetry blobs.
- An encrypted channel. Telemetry blobs are written in plaintext. See
  [Security Considerations](#security-considerations).

---

## Channel naming

Telemetry channels use the `telemetry-` prefix, placing them in the protocol registry
alongside `tide-`, `conveyer-`, and other protocol prefixes.

A DropChannel deployment uses a **single shared telemetry channel** for all
participants. All nodes and clients in the system write to the same channel, each
to a uniquely-named slot identified by their `participant_id`.

**Naming convention:**

```
telemetry-{deployment_id}
```

Example: `telemetry-homelab-01`

The `deployment_id` is an operator-chosen label scoping the telemetry channel to a
specific DropChannel deployment. All participants in that deployment write to the
same channel.

---

## Protocol semantics

The telemetry channel uses **Conveyer protocol semantics**:

- **Overwrite-always:** Each write replaces the previous value unconditionally.
- **No ACK:** There is no acknowledgement or backpressure. A participant writes and
  moves on.
- **No required consumer:** The monitoring tool is optional. Participants emit
  telemetry regardless of whether anything is reading it.
- **Single recv-slot per participant:** Each participant's slot is identified by
  its `participant_id`. The slot name is the participant's unique identifier.

A participant emits telemetry by writing its blob to:

```
channel: telemetry-{deployment_id}
slot:    {participant_id}
```

Example:

```
channel: telemetry-homelab-01
slot:    node-alpha
```

Emission frequency is implementation-defined. A reasonable default is once per
poll cycle. Implementations SHOULD emit telemetry at least as frequently as their
nominal poll interval to keep staleness detectable.

---

## Telemetry blob schema

Each participant writes a JSON blob to its slot. The schema is versioned; this is
schema version `1`.

### Common fields (all participants)

| Field              | Type    | Required | Description |
|--------------------|---------|----------|-------------|
| `schema_version`   | integer | yes      | Schema version. Currently `1`. |
| `participant_id`   | string  | yes      | Unique identifier for this participant within the deployment. Used as the slot name. |
| `role`             | string  | yes      | `"originator"`, `"terminator"`, or `"node"` |
| `pipeline`         | string  | yes      | Operator-assigned label for the pipeline this participant belongs to (e.g. `"a_to_b"`). Used by monitoring tools to group participants visually. |
| `state`            | string  | yes      | Current operational state. See [Participant states](#participant-states). |
| `poll_interval_ms` | integer | yes      | Nominal polling interval in milliseconds. Used by monitoring tools to calculate expected emission frequency and infer staleness. |
| `last_updated`     | string  | yes      | ISO-8601 UTC timestamp of when this blob was written. Format: `YYYY-MM-DDTHH:MM:SS.sssZ`. |

### Node fields (role: `"node"`)

| Field              | Type    | Required | Description |
|--------------------|---------|----------|-------------|
| `recv_channel_id`  | string  | yes      | Channel ID of this node's recv slot. |
| `send_channel_id`  | string  | yes      | Channel ID of this node's send slot. |
| `recv_occupied`    | boolean | yes      | Whether the recv slot is currently occupied. |
| `send_occupied`    | boolean | yes      | Whether the send slot is currently occupied. |

### Originator fields (role: `"originator"`)

| Field              | Type    | Required | Description |
|--------------------|---------|----------|-------------|
| `send_channel_id`  | string  | yes      | Channel ID of this originator's send slot. |
| `send_occupied`    | boolean | yes      | Whether the send slot is currently occupied. |

### Terminator fields (role: `"terminator"`)

| Field              | Type    | Required | Description |
|--------------------|---------|----------|-------------|
| `recv_channel_id`  | string  | yes      | Channel ID of this terminator's recv slot. |
| `recv_occupied`    | boolean | yes      | Whether the recv slot is currently occupied. |

### Example blobs

**Node:**

```json
{
  "schema_version": 1,
  "participant_id": "node-alpha",
  "role": "node",
  "pipeline": "a_to_b",
  "recv_channel_id": "tide-atb-0-1",
  "send_channel_id": "tide-atb-1-2",
  "recv_occupied": true,
  "send_occupied": true,
  "state": "send_polling",
  "poll_interval_ms": 800,
  "last_updated": "2025-03-16T12:00:00.000Z"
}
```

**Originator:**

```json
{
  "schema_version": 1,
  "participant_id": "client-a",
  "role": "originator",
  "pipeline": "a_to_b",
  "send_channel_id": "tide-atb-0-1",
  "send_occupied": true,
  "state": "awaiting_ack",
  "poll_interval_ms": 500,
  "last_updated": "2025-03-16T12:00:00.000Z"
}
```

**Terminator:**

```json
{
  "schema_version": 1,
  "participant_id": "client-b",
  "role": "terminator",
  "pipeline": "a_to_b",
  "recv_channel_id": "tide-atb-3-b",
  "recv_occupied": false,
  "state": "recv_polling",
  "poll_interval_ms": 500,
  "last_updated": "2025-03-16T12:00:00.000Z"
}
```

---

## Participant states

The `state` field communicates the participant's current position in its operational
state machine. Valid values:

| State            | Applicable roles         | Description |
|------------------|--------------------------|-------------|
| `idle`           | originator               | No blob queued; originator is waiting for application layer. |
| `recv_polling`   | node, terminator         | Polling recv slot; nothing held yet. |
| `send_polling`   | node                     | Blob received and forwarded; polling send slot for ACK cascade. |
| `awaiting_ack`   | originator               | Blob written to send slot; waiting for ACK cascade to clear it. |
| `reading`        | terminator               | Recv slot occupied; consuming payload. |
| `offline`        | any                      | Participant has emitted this state explicitly before going offline, or monitoring tool has inferred it from staleness. See [Staleness](#staleness). |

Implementations SHOULD emit `offline` as the final telemetry write before a
participant intentionally shuts down, giving monitoring tools a clean signal
distinguishing graceful shutdown from disappearance.

---

## Topology reconstruction

A monitoring tool reconstructs the pipeline graph from a flat set of telemetry blobs
using a single join rule:

> If `participant[i].send_channel_id == participant[j].recv_channel_id`, then
> participant `i` is the immediate upstream neighbor of participant `j`.

This join requires no config files, no shared secret, and no knowledge of deployment
topology beyond what participants self-report.

**Algorithm:**

1. Read all `{participant_id}.json` blobs from the telemetry channel.
2. Build two maps:
   - `send_map`: `channel_id → participant_id` (who sends on this channel)
   - `recv_map`: `channel_id → participant_id` (who receives on this channel)
3. For each entry in `send_map`, look up the same channel in `recv_map`. A match
   is a directed edge: sender → receiver.
4. Group participants by `pipeline` label.
5. Within each pipeline, order participants by following the edge chain from the
   participant with no incoming edge (the originator) to the participant with no
   outgoing edge (the terminator).

**Partial topologies** are valid. A monitoring tool MUST handle the case where some
participants are not emitting telemetry (offline, not yet implemented, or using an
older schema version). Edges should only be drawn for participants with matching
channel IDs that are currently present in the telemetry channel.

---

## Staleness

A telemetry blob is considered **stale** if:

```
now - last_updated > staleness_threshold
```

The recommended default `staleness_threshold` is:

```
staleness_threshold = max(5s, 3 × poll_interval_ms)
```

A stale participant should be treated as **presumed offline** by monitoring tools.
This is a monitoring-layer inference, not a protocol assertion. A participant that
goes offline without emitting `state: "offline"` will be detected as stale after
the threshold elapses.

Monitoring tools SHOULD visually distinguish between:

- `state: "offline"` — participant declared its own offline state (graceful)
- stale but last known state was not `offline` — participant disappeared (ungraceful)

---

## Security considerations

Telemetry blobs are written in **plaintext** to the telemetry channel. This is a
deliberate design constraint: nodes are crypto-blind and cannot encrypt telemetry
using payload keys they do not hold.

**What telemetry exposes:**

- `participant_id` values — operator-chosen labels, not cryptographic identifiers
- `channel_id` values — the addresses of payload slots on the storage backend
- Slot occupancy state — whether a blob is in transit at a given hop
- Pipeline labels and roles — the logical structure of the deployment

**What telemetry does not expose:**

- Payload content — never included in telemetry blobs
- Encryption keys — nodes do not hold payload keys; originators and terminators
  hold keys but do not emit them
- Timing of payload content — only slot occupancy (present/absent), not when
  a specific payload was written

**Threat model note:** An attacker with read access to the telemetry channel learns
the channel IDs of payload slots. If they also have write access to the storage
backend, they could attempt slot squatting or other storage-layer attacks. This is
consistent with the existing threat model documented in `security-model.md`, where
the storage backend is the primary attack surface. Telemetry does not increase
the attack surface meaningfully beyond what is already present in the payload
channels.

Operators with heightened confidentiality requirements MAY choose not to enable
telemetry emission, accepting reduced observability in exchange.

---

## Protocol registry entry

The following entry belongs in `protocol-registry.md`:

| Prefix       | Protocol | Description |
|--------------|----------|-------------|
| `telemetry-` | Conveyer | Observability channel. Participants emit self-describing state blobs. One shared channel per deployment; one slot per participant. No ACK, no backpressure, plaintext. |
