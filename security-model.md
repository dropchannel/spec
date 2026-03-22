# Security Model

## What is protected

**Payload content is end-to-end encrypted.** Blobs deposited into any Waterway are
AES-256-GCM encrypted before leaving the originating endpoint. They are decrypted only
at the terminating endpoint. No intermediate participant — relay, Raft, or storage
provider — can read message content.

**The relay and Rafts are encryption-agnostic by design.** No component other than an
endpoint has a dependency on the cryptography layer. In the Python reference
implementation, this is enforced as a hard package-boundary property: `dropchannel-raft`
and `dropchannel-server` do not depend on the `cryptography` package and cannot import
it.

**The shared secret never transits the pipeline.** Key distribution is out-of-band
between the two endpoint operators.

**Rafts and clients are strongly isolated when operated with outbound-only connectivity.**
A Raft or client that accepts no inbound connections exposes no network attack surface.
It cannot be reached by an external attacker through the network — there are no open
ports, no listening services, and no mechanism by which a malicious blob in storage can
cause the Raft to execute attacker-controlled code. The Raft pulls from storage; storage
never pushes to the Raft. This makes a correctly operated Raft functionally equivalent
to an air-gapped machine that makes outbound-only HTTP calls.

This isolation property holds only so long as Rafts and clients are operated without
inbound connectivity. Any inbound-accessible service on the same host, or any relaxation
of the outbound-only constraint, narrows or eliminates this guarantee.

**The primary attack surface is storage, not application logic.** This is a deliberate
inversion of the conventional security model. In most networked systems, application
logic is the exposed surface and storage sits safely behind it. In DropChannel, the
application logic (Rafts and clients) is the hard target — effectively unreachable when
correctly operated — and the storage provider is the soft target, accessible to anyone
who obtains storage credentials or exploits a provider misconfiguration.

This inversion is structural, not accidental. The protocol is designed so that the
security guarantee holds regardless of the confidentiality or integrity of the underlying
storage provider. A fully compromised storage layer reveals ciphertext and metadata but
not plaintext. Future protocol revisions aim to extend this guarantee to cover Waterway
integrity and existence opacity as well (see open items below).

---

## What is not protected

**Heartbeat metadata is plaintext.** The observability layer (introduced in v0.6)
writes heartbeat files containing Raft/client UUIDs, timestamps, uptime counters, and
status strings. This content is not encrypted. It reveals that a participant with a given
UUID is active, but contains no payload content and no routing secrets.

Encryption of heartbeat content via a dedicated key (`DRIP_KEY`) is deferred to a future
revision.

**Channel IDs and Waterway names are not secret.** Any party with access to the shared
storage location can observe which channel IDs and Waterway names exist. Payload content
remains protected regardless.

**Write guard atomicity is best-effort.** The one-blob-per-Waterway invariant is enforced
by a check-before-write operation. On shared storage providers this has an inherent race
window. At human interaction cadence this is acceptable; high-frequency concurrent
writers are out of scope.

---

## Trust boundaries

| Component | Trusted with | Isolation property |
|-----------|-------------|-------------------|
| Endpoint | Plaintext, shared secret | Hard target when inbound connections disabled |
| Raft | Opaque ciphertext only | Hard target when inbound connections disabled |
| Relay / storage provider | Opaque ciphertext only | Soft target — primary attack surface |
| Heartbeat metadata | UUIDs, timestamps, status strings (no payload content) | Plaintext at rest in storage |

**On Raft key custody.** The original crypto-blindness constraint on Rafts is specifically
scoped to payload encryption keys — Rafts must never hold material that could decrypt
message content. It does not preclude Rafts from holding protocol-level integrity keys
(such as Waterway MAC keys) that are distinct from the payload encryption key and derived
separately. The isolation property makes a correctly operated Raft a trustworthy
custodian of such material. This distinction is relevant to future protocol extensions
and should not be conflated with the payload encryption boundary.

---

## Threat model scope

DropChannel is designed for two known, mutually trusting parties communicating through
infrastructure they may not control or fully trust. It is not designed for:

- Anonymous or pseudonymous communication
- Protection against a compromised endpoint
- High-assurance environments requiring formal verification
- More than two parties per channel

---

## Open items

The following storage-layer attack vectors are known and not yet addressed by the
protocol. They are candidates for future spec revisions:

- **Waterway integrity** — an attacker with storage write access can delete, substitute, or
  squat Waterways. No protocol-level mechanism currently detects or prevents this.
- **Existence opacity** — Waterway names and their appearance/disappearance are visible to
  any party with storage read access. This enables traffic analysis even without payload
  access.
- **Replay injection** — a captured valid ciphertext blob can be re-inserted into an
  empty Waterway. The current wire format does not include a sequence number or nonce
  freshness check that would allow endpoints to reject replayed messages.
- **Silent delivery corruption** — the ACK cascade confirms that a terminating endpoint
  consumed a Waterway, but not that decryption succeeded. A substituted blob produces a
  completed cascade with a silently dropped message.

These items share a common mitigation direction: a cryptographic Waterway envelope (covering
Waterway naming, integrity MAC, and sequence binding) that makes storage manipulation
detectable by protocol participants without compromising the Raft crypto-blindness
constraint on payload content.
