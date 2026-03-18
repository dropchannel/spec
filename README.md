# DropChannel System Specification

This repository is the authoritative definition of the DropChannel system. It describes
the components that make up a DropChannel deployment, the contracts between them, the
family of coordination protocols that can operate within the system, and the requirements
for a conformant implementation.

Individual coordination protocol specifications are maintained in their own repositories
and referenced from this document.

---

## What DropChannel Is

DropChannel is a **multi-protocol, storage-backed coordination runtime** for encrypted,
asynchronous, point-to-point communication between two endpoints.

The system is designed around four constraints that distinguish it from conventional
messaging infrastructure:

**Outbound-only polling.** No participant ever accepts an inbound connection. Every node,
endpoint, and relay operates by polling outbound. This means DropChannel works behind
home routers, corporate NAT, or any environment where inbound connectivity is unavailable
or undesirable.

**Crypto-blind forwarding nodes.** Intermediate nodes never hold encryption keys and
never see plaintext. Untrusted third-party infrastructure — a cloud storage bucket, a
synced folder, a friend's server — can serve as a forwarding hop without being granted
any trust. The cryptographic boundary is strictly at the endpoints.

**Asynchronous by design.** Participants do not need to be online simultaneously. Payloads
wait durably at each hop until the downstream participant is ready to process them. This
is a structural property of the system, not a resilience feature layered on top.

**Provider-agnostic composition.** Any storage medium that implements the ChannelProvider
interface is a valid hop. A pipeline may mix heterogeneous providers. The coordination
protocol is defined at the interface level, not the transport level.

---

## System Components

A DropChannel deployment is composed of the following components. Each is defined by its
role and its interface contract — not by its implementation.

### Endpoint

An Endpoint is a process that originates or consumes messages on behalf of a user or
application. Endpoints are the **only** participants that perform encryption and
decryption. An endpoint holds the SHARED_SECRET for its channel and uses it to encrypt
outbound payloads and decrypt inbound payloads before delivering them to the application.

An endpoint has no knowledge of the number of hops in its pipeline, the providers used
by intermediate nodes, or the infrastructure of the other endpoint.

### Node

A Node is a process that forwards blobs from one ChannelProvider to the next along a
physical pipeline. Nodes are crypto-blind: they forward opaque bytes and perform no
cryptographic operations. A node has no SHARED_SECRET and no knowledge of message
semantics.

A node operates in exactly one direction on exactly one physical pipeline. It is pure
forwarding infrastructure. Its behavior on any given channel is determined by the channel
name prefix, which identifies the coordination protocol to apply.

### ChannelProvider

A ChannelProvider is any storage medium that implements the five-operation interface
defined below. Providers are the physical hops through which blobs move. A provider has
no knowledge of the coordination protocol operating above it — it simply stores and
retrieves blobs on demand.

Conformant ChannelProvider implementations include but are not limited to: cloud object
storage, synced filesystem directories, HTTP relay servers, and local filesystems.

### Relay

A Relay is a network-accessible service that exposes a ChannelProvider interface over
HTTP (or another network protocol). A relay allows nodes or endpoints on different
networks to share a provider hop without direct access to the underlying storage medium.

A relay is a ChannelProvider, not a Node. It performs no coordination logic — it stores
and retrieves blobs as directed by the participants polling it.

---

## The ChannelProvider Interface

Any storage medium implementing the following five operations is a conformant
ChannelProvider:

| Operation | Signature | Description |
|-----------|-----------|-------------|
| `write`  | `(channel_id, slot, data) → bool` | Write blob to slot. Returns `True` on success, `False` if slot is occupied. Must be atomic with respect to concurrent writers. |
| `read`   | `(channel_id, slot) → bytes\|None` | Read and delete blob. Returns bytes if present, `None` if empty. Consume-on-read. Called only by the terminating endpoint. |
| `peek`   | `(channel_id, slot) → bytes\|None` | Read blob without consuming it. Blob remains in slot after this call. Called by nodes and the originating endpoint. |
| `exists` | `(channel_id, slot) → bool` | Return `True` if slot is occupied, `False` if empty. The primary polling operation. |
| `delete` | `(channel_id, slot) → None` | Delete blob from slot. Idempotent — no error if slot is already empty. Used during ACK cascade. |

The distinction between `read` and `peek` is fundamental to the system's delivery
semantics. `read` is destructive and triggers the ACK cascade. `peek` is non-destructive
and is used by all other participants during the forward pass.

---

## Channel Naming and Protocol Dispatch

### Channel Names

A channel name is a string composed of a **protocol prefix** and a **channel identifier**,
separated by a hyphen:

```
{prefix}-{identifier}
```

Examples: `winch-documents`, `conveyer-telemetry`, `piston-chat`

The prefix is the machine-readable protocol selector. A node reads the prefix of a
channel name and instantiates the appropriate coordination protocol handler for that
channel. No additional configuration is required.

The identifier is an arbitrary string chosen by the channel operators. It must be unique
within the scope of the storage provider it uses.

### Protocol Registry

The following prefixes are defined. Each maps to an authoritative protocol specification:

| Prefix | Protocol | Semantics | Spec |
|--------|----------|-----------|------|
| `winch-` | Winch | Store-and-forward, backpressure, back-cascade ACK | [dropchannel/winch-protocol](https://github.com/dropchannel/winch-protocol) |
| `conveyer-` | Conveyer | Store-and-forward, no back-pressure, no ACK | [dropchannel/conveyer-protocol](https://github.com/dropchannel/conveyer-protocol) |
| `piston-` | Piston | TBD | TBD |
| `telemetry-` | Conveyer | Observability side-channel. Each participant writes a self-describing state blob to a shared channel, one slot per participant keyed by participant ID. No ACK, no backpressure, plaintext. Consumed by external monitoring tools. | [telemetry.md](telemetry.md) |
| `heartbeat-` | Meta slot | Per-hop liveness chain operating on meta slots alongside primary payload slots. Nodes relay upstream heartbeat content forward; clients write status signals. Plaintext. | [heartbeat.md](heartbeat.md) |

The `telemetry-` and `heartbeat-` prefixes are reserved for the observability layer.
They are listed here for completeness and prefix reservation — a conformant node
does not dispatch payload traffic on these channels. See `spec/observability.md`.

### Unrecognized Prefixes

A conformant node encountering an unrecognized channel name prefix **must** halt
processing on that channel, log the unrecognized prefix, and continue processing other
channels unaffected. It must not silently skip, must not guess, and must not apply a
default protocol.

---

## Security Model

### Trust Boundary

The cryptographic trust boundary in DropChannel is strictly at the endpoints. Only
endpoints hold the SHARED_SECRET. All other components — nodes, providers, relays —
are explicitly untrusted with respect to payload content.

This is not merely a configuration choice. It is a structural property enforced at the
implementation level: a conformant node implementation must not have a dependency on any
cryptographic library used for payload encryption/decryption. This is verifiable at the
package dependency level.

### SHARED_SECRET

The SHARED_SECRET is a symmetric key shared exclusively between the two endpoints of a
channel. It is established out-of-band before the channel is used and is never
transmitted through any component of the DropChannel system.

The specific encryption algorithm is a per-protocol or per-implementation concern. The
system requires only that nodes cannot derive plaintext from the blobs they forward.

### Threat Model

DropChannel is designed to be secure against a **compromised forwarding layer**. An
adversary with full read/write access to a storage provider learns only:

- That a channel exists (channel ID is visible)
- Blob sizes and timing (metadata)
- Nothing about payload content

DropChannel does not provide anonymity, traffic analysis resistance, or protection
against a compromised endpoint. It is not a general-purpose anonymity network.

---

## Conformance

An implementation is a conformant DropChannel implementation if it satisfies all of the
following:

- It implements the ChannelProvider interface with the exact semantics defined above
- Its Node component performs no cryptographic operations on payload content
- Its Node component dispatches coordination protocol by channel name prefix as defined above
- Its Node component halts on unrecognized prefixes rather than guessing or defaulting
- Its Endpoint component is the exclusive holder of the SHARED_SECRET for its channel
- It does not require inbound network connections at any component

Conformant implementations are listed in the [Implementations](#implementations) section.

---

## Implementations

| Implementation | Language | Status | Repository |
|----------------|----------|--------|------------|
| dropchannel-py | Python   | Active | [dropchannel/dropchannel-py](https://github.com/dropchannel/dropchannel-py) |

---

## Repository Structure

```
github.com/dropchannel/
  spec/                 ← This repository: system specification (you are here)
  winch-protocol/       ← Winch protocol specification
  conveyer-protocol/    ← Conveyer protocol specification
  dc-monitor/              ← Implementation-agnostic topology visualizer
  dropchannel-py/       ← Python reference implementation
  .github/              ← Org profile, ADRs, planning documents
```

This repository (`spec`) owns the system-level specification: the ChannelProvider
interface, encryption standard, security model, protocol dispatch rules, protocol
registry, the observability layer (`observability.md`, `heartbeat.md`, `telemetry.md`),
and the Agent/Worker runtime specification (`agent.md`).

The `dc-monitor/` repository owns the topology visualizer tool. It is
implementation-agnostic — any conformant DropChannel implementation can feed it by
emitting telemetry blobs as defined in `spec/telemetry.md`.

---

## License

System specification: [Creative Commons CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/)
(public domain dedication — implement freely without restriction)
