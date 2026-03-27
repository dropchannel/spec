# Protocol Registry

A DropChannel Raft is a protocol-multiplexing forwarder. The protocol governing a
Waterway is determined by the Waterway name prefix — the substring before the first `-`
in the Waterway name. The Raft reads the prefix on startup, selects the appropriate
state machine, and runs it. The prefix is metadata; it is never part of the encrypted
payload.

---

## Registered Protocols

| Prefix | Protocol | Spec | Semantics |
|--------|----------|------|-----------|
| `tideway-` | Tideway | [`tideway-protocol`](https://github.com/dropchannel/tideway-protocol) | Turn-passing, fixed initiator at Upper-side, bidirectional on a single Waterway. Structural backpressure; implicit delivery confirmation via ACK cascade. |
| `riverway-` | Riverway | [`riverway-protocol`](https://github.com/dropchannel/riverway-protocol) | Continuous, unidirectional, overwrite-always, no ACK. |
| `telemetry-` | Riverway | [telemetry.md](../runtime/telemetry.md) | Observability side-channel. Each participant writes a self-describing state blob. One shared channel per deployment; one Waterway per participant. No ACK, no backpressure, plaintext. |
| `heartbeat-` | Meta Waterway | [heartbeat.md](../runtime/heartbeat.md) | Per-hop liveness chain. Rafts relay upstream heartbeat content forward; clients write status signals. Operates on meta Waterways alongside primary payload Waterways. Plaintext. |

---

## Dispatch Rules

**Prefix extraction.** The protocol prefix is the substring of the Waterway name up to
and including the first `-`. A Waterway named `tideway-commands` has prefix `tideway-`;
a Waterway named `riverway-telemetry` has prefix `riverway-`.

**Unrecognized prefix: halt and log.** If a Raft encounters a Waterway name whose prefix
does not match any registered protocol, the Raft halts and logs the unrecognized prefix.
It does not skip, apply a default protocol, or continue silently. Silent continuation on
an unrecognized prefix would mask misconfiguration and risk data loss or corruption.

**No default protocol.** There is no fallback protocol for an unprefixed or unrecognized
Waterway name. Every valid Waterway name must begin with a registered prefix followed
by `-`.

---

## Protocol Selection Rationale

**Tideway** is the correct choice when:

- Each message must be confirmed delivered before the next can be sent
- The pipeline carries commands, file transfers, or discrete task handoffs
- Backpressure is desirable — the producer should block until the consumer has taken the
  previous message

**Riverway** is the correct choice when:

- Delivery confirmation is not required
- Continuous or frequent writes should not block on consumer availability
- Overwrite semantics are acceptable (most recent value wins)

---

## Adding a Protocol

A new protocol requires:

1. A spec repository under `github.com/dropchannel/{name}-protocol` containing a
   `README.md` with the full protocol specification.
2. A registered prefix in this file, with a link to the spec repo.
3. A state machine implementation in each runtime that wishes to support the protocol
   (e.g. `dropchannel-py`).
4. The state machine must operate exclusively through the `DockProvider` interface.
   See [`dock-provider.md`](dock-provider.md).

A Waterway prefix must be a lowercase ASCII string containing no `-` characters, ending
with `-` as the Waterway name delimiter. Examples: `tideway-`, `telemetry-`, `heartbeat-`.
