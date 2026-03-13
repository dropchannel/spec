# Security Model

## What is protected

**Payload content is end-to-end encrypted.** Blobs deposited into any slot are
AES-256-GCM encrypted before leaving the originating endpoint. They are decrypted only
at the terminating endpoint. No intermediate participant — relay, node, or storage
provider — can read message content.

**The relay and nodes are encryption-agnostic by design.** No component other than an
endpoint has a dependency on the cryptography layer. In the Python reference
implementation, this is enforced as a hard package-boundary property: `dropchannel-node`
and `dropchannel-server` do not depend on the `cryptography` package and cannot import
it.

**The shared secret never transits the pipeline.** Key distribution is out-of-band
between the two endpoint operators.

---

## What is not protected

**Heartbeat metadata is plaintext.** The observability layer (introduced in v0.6)
writes heartbeat files containing node/client UUIDs, timestamps, uptime counters, and
status strings. This content is not encrypted. It reveals that a participant with a given
UUID is active, but contains no payload content and no routing secrets.

Encryption of heartbeat content via a dedicated key (`DRIP_KEY`) is deferred to a future
revision.

**Channel IDs and slot names are not secret.** Any party with access to the shared
storage location can observe which channel IDs and slot names exist. Payload content
remains protected regardless.

**Write guard atomicity is best-effort.** The one-blob-per-slot invariant is enforced
by a check-before-write operation. On shared storage providers this has an inherent race
window. At human interaction cadence this is acceptable; high-frequency concurrent
writers are out of scope.

---

## Trust boundaries

| Component | Trusted with |
|-----------|-------------|
| Endpoint | Plaintext, shared secret |
| Node | Opaque ciphertext only |
| Relay / storage provider | Opaque ciphertext only |
| Heartbeat metadata | UUIDs, timestamps, status strings (no payload content) |

---

## Threat model scope

DropChannel is designed for two known, mutually trusting parties communicating through
infrastructure they may not control or fully trust. It is not designed for:

- Anonymous or pseudonymous communication
- Protection against a compromised endpoint
- High-assurance environments requiring formal verification
- More than two parties per channel
