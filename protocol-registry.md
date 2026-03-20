# Protocol Registry

A DropChannel node is a protocol-multiplexing forwarder. The protocol governing a
physical pipeline is determined by the channel name prefix — the substring before the
first `-` in the channel ID. The node reads the prefix on startup, selects the
appropriate state machine, and runs it. The prefix is metadata; it is never part of the
encrypted payload.

---

## Registered Protocols

| Prefix | Protocol | Spec | Semantics |
|--------|----------|------|-----------|
| `tide-` | Tide | [`tide-protocol`](https://github.com/dropchannel/tide-protocol) | Hold-and-cascade: blob accumulates at every hop during forward pass; terminating endpoint's read triggers deletion cascade backward through the chain. Structural backpressure; implicit delivery confirmation. |

---

## Dispatch Rules

**Prefix extraction.** The protocol prefix is the substring of the channel ID up to and
including the first `-`. A channel ID of `tide-commands` has prefix `tide-`; a channel
ID of `tide-commands` has prefix `tide-`.

**Unrecognized prefix: halt and log.** If a node encounters a channel ID whose prefix
does not match any registered protocol, the node halts and logs the unrecognized prefix.
It does not skip, apply a default protocol, or continue silently. Silent continuation on
an unrecognized prefix would mask misconfiguration and risk data loss or corruption.

**No default protocol.** There is no fallback protocol for an unprefixed or
unrecognized channel ID. Every valid channel ID must begin with a registered prefix
followed by `-`.

---

## Protocol Selection Rationale

**Tide** is the correct choice when:

- Each message must be confirmed delivered before the next can be sent
- The pipeline carries commands, file transfers, or discrete task handoffs
- Backpressure is desirable — the producer should block until the consumer has taken the
  previous message

---

## Adding a Protocol

A new protocol requires:

1. A spec repository under `github.com/dropchannel/{name}-protocol` containing a
   `README.md` with the full protocol specification.
2. A registered prefix in this file, with a link to the spec repo.
3. A state machine implementation in each runtime that wishes to support the protocol
   (e.g. `dropchannel-py`).
4. The state machine must operate exclusively through the `ChannelProvider` interface.
   See [`channel-provider.md`](channel-provider.md).

A protocol prefix must be a lowercase ASCII string containing no `-` characters, ending
with `-` as the channel ID delimiter. Examples: `tide-`, `telemetry-`, `heartbeat-`.
