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

**Outbound-only polling.** No participant ever accepts an inbound connection. Every Raft,
endpoint, and relay operates by polling outbound. This means DropChannel works behind
home routers, corporate NAT, or any environment where inbound connectivity is unavailable
or undesirable.

**Crypto-blind forwarding Rafts.** Intermediate Rafts never hold encryption keys and
never see plaintext. Untrusted third-party infrastructure — a cloud storage bucket, a
synced folder, a friend's server — can serve as a forwarding hop without being granted
any trust. The cryptographic boundary is strictly at the endpoints.

**Asynchronous by design.** Participants do not need to be online simultaneously. Payloads
wait durably at each hop until the downstream participant is ready to process them. This
is a structural property of the system, not a resilience feature layered on top.

**Provider-agnostic composition.** Any storage medium that implements the DockProvider
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
by intermediate Rafts, or the infrastructure of the other endpoint.

### Raft

A Raft is a process that forwards blobs from one Dock to the next along a
physical pipeline. Rafts are crypto-blind: they forward opaque bytes and perform no
cryptographic operations. A Raft has no SHARED_SECRET and no knowledge of message
semantics.

A Raft operates in exactly one direction on exactly one Waterway. It is pure
forwarding infrastructure. Its behavior is determined by the Waterway name prefix,
which identifies the coordination protocol to apply.

### Dock

A Dock is any storage medium that implements the five-operation interface
defined below. Docks are the physical hops through which blobs move. A Dock has
no knowledge of the coordination protocol operating above it — it simply stores and
retrieves blobs on demand.

Conformant Dock implementations include but are not limited to: cloud object
storage, synced filesystem directories, HTTP relay servers, and local filesystems.

### Relay

A Relay is a network-accessible service that exposes a Dock interface over
HTTP (or another network protocol). A relay allows Rafts or endpoints on different
networks to share a provider hop without direct access to the underlying storage medium.

A relay is a Dock, not a Raft. It performs no coordination logic — it stores
and retrieves blobs as directed by the participants polling it.

---

## The DockProvider Interface

Any storage medium implementing the following five operations is a conformant
Dock:

| Operation | Signature | Description |
|-----------|-----------|-------------|
| `write`  | `(channel, waterway, filename, data) → bool` | Write blob to Waterway. Returns `True` on success, `False` if Waterway is occupied. Must be atomic with respect to concurrent writers. |
| `read`   | `(channel, waterway, filename) → bytes\|None` | Read and delete blob. Returns bytes if present, `None` if empty. Consume-on-read. Called only by the terminating endpoint. |
| `peek`   | `(channel, waterway, filename) → bytes\|None` | Read blob without consuming it. Blob remains in Waterway after this call. Called by Rafts and the originating endpoint. |
| `exists` | `(channel, waterway, filename) → bool` | Return `True` if Waterway is occupied, `False` if empty. The primary polling operation. |
| `delete` | `(channel, waterway, filename) → None` | Delete blob from Waterway. Idempotent — no error if Waterway is already empty. Used during ACK cascade. |

The distinction between `read` and `peek` is fundamental to the system's delivery
semantics. `read` is destructive (consume-on-read from the Waterway) and triggers the ACK
cascade. `peek` is non-destructive and is used by all other participants during the
forward pass.

---

## Channel Naming and Protocol Dispatch

### Channels

A Channel is a named path between two fixed endpoints. It is a pure namespace — an
operator-chosen label with no embedded protocol or direction information:

```
{channel-name}
```

Examples: `chat`, `file-transfer`, `sensor-feed`

A Channel may contain any number of Waterways simultaneously. All Waterways within a
Channel connect the same two endpoints. A single Channel can carry independent Waterways
with different protocols or directionalities — for example, a Tideway for turn-passing
request/response alongside a Riverway for continuous status emission.

### Waterways

A Waterway is a named, protocol-typed flow within a Channel. Its name is composed of a
**protocol prefix** and an **identifier**, separated by a hyphen:

```
{prefix}-{identifier}
```

Examples: `tideway-bob`, `riverway-status`

The prefix is the machine-readable protocol selector. A Raft reads the Waterway name
prefix and instantiates the appropriate coordination protocol handler. No additional
configuration is required for protocol dispatch.

The identifier is an arbitrary string chosen by the Channel operators. It must be unique
within the Channel's scope on the storage provider it uses.

### Protocol Registry

The following Waterway prefixes are defined. Each maps to an authoritative protocol
specification:

| Prefix | Protocol | Semantics | Spec |
|--------|----------|-----------|------|
| `tideway-` | Tideway | Turn-passing, backpressure, bidirectional on a single Waterway | [dropchannel/tideway-protocol](https://github.com/dropchannel/tideway-protocol) |
| `riverway-` | Riverway | Continuous, unidirectional, overwrite-always, no ACK | [dropchannel/riverway-protocol](https://github.com/dropchannel/riverway-protocol) |
| `telemetry-` | Riverway | Observability side-channel. Each participant writes a self-describing state blob to a shared Waterway, one Waterway per participant keyed by participant ID. No ACK, no backpressure, plaintext. Consumed by external monitoring tools. | [telemetry.md](telemetry.md) |
| `heartbeat-` | Meta Waterway | Per-hop liveness chain operating on meta Waterways alongside primary payload Waterways. Rafts relay upstream heartbeat content forward; clients write status signals. Plaintext. | [heartbeat.md](heartbeat.md) |

The `telemetry-` and `heartbeat-` prefixes are reserved for the observability layer.
They are listed here for completeness and prefix reservation — a conformant Raft
does not dispatch payload traffic on these Waterways. See `spec/observability.md`.

### Unrecognized Prefixes

A conformant Raft encountering an unrecognized **Waterway name prefix** must halt
processing on that Waterway, log the unrecognized prefix, and continue processing other
Waterways unaffected. It must not silently skip, must not guess, and must not apply a
default protocol.

---

## Security Model

### Trust Boundary

The cryptographic trust boundary in DropChannel is strictly at the endpoints. Only
endpoints hold the SHARED_SECRET. All other components — Rafts, providers, relays —
are explicitly untrusted with respect to payload content.

This is not merely a configuration choice. It is a structural property enforced at the
implementation level: a conformant Raft implementation must not have a dependency on any
cryptographic library used for payload encryption/decryption. This is verifiable at the
package dependency level.

### SHARED_SECRET

The SHARED_SECRET is a symmetric key shared exclusively between the two endpoints of a
channel. It is established out-of-band before the channel is used and is never
transmitted through any component of the DropChannel system.

The specific encryption algorithm is a per-protocol or per-implementation concern. The
system requires only that Rafts cannot derive plaintext from the blobs they forward.

### Threat Model

DropChannel is designed to be secure against a **compromised forwarding layer**. An
adversary with full read/write access to a storage provider learns only:

- That a channel exists (channel name is visible)
- Blob sizes and timing (metadata)
- Nothing about payload content

DropChannel does not provide anonymity, traffic analysis resistance, or protection
against a compromised endpoint. It is not a general-purpose anonymity network.

---

## Conformance

An implementation is a conformant DropChannel implementation if it satisfies all of the
following:

- It implements the DockProvider interface with the exact semantics defined above
- Its Raft component performs no cryptographic operations on payload content
- Its Raft component dispatches coordination protocol by Waterway name prefix as defined above
- Its Raft component halts on unrecognized prefixes rather than guessing or defaulting
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
  spec/                 ← This repository: system specification and governance/ADR record (you are here)
  tideway-protocol/    ← Tideway protocol specification
  riverway-protocol/   ← Riverway protocol specification
  dc-monitor/              ← Implementation-agnostic topology visualizer
  dropchannel-py/       ← Python reference implementation
  .github/              ← Org profile
```

This repository (`spec`) owns the system-level specification: the DockProvider
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
