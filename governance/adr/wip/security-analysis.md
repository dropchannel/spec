# DropChannel Security Analysis

## Attack Vectors, Existing Defenses, and Candidate Mitigations

*Status: Working document — analysis only. Items marked CANDIDATE require design work before spec inclusion.*
*Scope: Tideway protocol on DropChannel topology. Storage-layer threat model.*

---

## Foundational Security Observations

### The Isolation Property

Rafts and clients in a conformant DropChannel deployment operate with **no inbound network connections**. Every participant polls outbound only. This yields an attack surface that is roughly equivalent to an air-gapped machine making outbound HTTP calls:

- No open ports, no listening services, no network-exposed attack surface
- No ability for an attacker to inject commands through the protocol — Rafts process opaque blobs; a malformed blob can only cause a rejection, not execution
- No push from storage to Raft — the Raft pulls; storage never initiates
- Physical or OS-level compromise is the only remaining Raft/client vector, which is outside the protocol threat model

This is a **hard target** profile. Compromising a Raft or client requires targeted, high-cost, physical or privileged access.

### The Storage Inversion

DropChannel inverts the conventional security model:

| Component | Conventional systems | DropChannel |
|-----------|---------------------|-------------|
| Application logic (Rafts/clients) | Soft target — exposed services | **Hard target** — isolated, outbound-only |
| Storage | Relatively safe — behind the application | **Soft target** — exposed to provider credentials, misconfig, insider threats |

**The storage layer is the primary attack surface.** This drives every design decision in the security model. The protocol's job is to make storage-level access as unrewarding as possible through cryptographic means alone — not through operational security of the provider, not through access controls, not through provider-specific features.

This principle should hold regardless of whether the provider is GCS, Dropbox, a self-hosted httprelay, or a local filesystem. The protocol makes no assumptions about the confidentiality or integrity of the storage medium.

---

## Threat Model Scope

### In scope

- Attacker with full read/write access to one or more storage providers in the pipeline
- Attacker observing Waterway existence, timing, and blob sizes (metadata)
- Attacker injecting, deleting, substituting, or replaying blobs at the storage layer
- Attacker attempting to infer communication patterns from metadata

### Out of scope

- Compromise of a Raft or client machine (physical/OS-level)
- Compromise of the SHARED_SECRET (established out-of-band, never transmitted through the system)
- Traffic analysis resistance beyond what protocol-level mechanisms can provide
- Anonymity or identity concealment of endpoints

---

## Attack Vector Catalog

---

### AV-01: Payload Injection

**Description:** Attacker writes a crafted blob into an empty Waterway, hoping the receiving endpoint will process it as a legitimate message.

**Access required:** Write access to any provider in the pipeline.

**Current defense:** AES-256-GCM authentication. A foreign blob without a valid authentication tag will fail decryption at the endpoint. The attacker cannot forge a valid ciphertext without the SHARED_SECRET, which is never transmitted through the system.

**Status: NEUTRALIZED by existing design.** No spec changes required.

---

### AV-02: Waterway Squatting

**Description:** Attacker monitors Waterway existence via `exists()` or equivalent storage API, detects when a Waterway empties, and writes garbage into it before the legitimate participant can write. With continuous monitoring, the attacker can hold a Waterway hostage indefinitely — the legitimate writer receives `False` on every write attempt and stalls.

**Access required:** Read + write access to any provider in the pipeline.

**Current defense:** None. The write() atomicity guarantee prevents concurrent legitimate writers from colliding, but does not distinguish a legitimate write from a hostile one.

**Impact:** Denial of service. Pipeline stalls indefinitely. The legitimate participant has no way to distinguish a squatted Waterway from a slow upstream participant.

**Status: OPEN GAP.**

**Candidate mitigation — Waterway name derivation:**
Derive Waterway names from the SHARED_SECRET using a KDF:

```
waterway_name = HKDF(SHARED_SECRET, channel || waterway_role || message_seq)
```

An attacker cannot predict the next valid Waterway name without the SHARED_SECRET. Squatting requires knowing the target Waterway name in advance — with derived names, the attacker must brute-force or observe the name as it appears, reducing the squatting window to a race condition rather than an indefinite hold.

*Design questions: How are derived Waterway names communicated to Rafts? Does this require Rafts to hold key material? See AV-05 and the Raft Trust section below.*

---

### AV-03: Cascade Disruption (Premature Deletion)

**Description:** Attacker deletes a blob from a lower_dock before the downstream Raft polls it. The upstream Raft sees its lower_dock cleared and re-enters recv-polling mode, deleting its upper_dock. The payload is lost. From the originator's perspective, the ACK cascade completed normally — delivery is falsely confirmed.

**Access required:** Write (delete) access to any provider in the pipeline.

**Current defense:** None. The `delete()` operation is idempotent by design (crash recovery requirement), and the protocol has no mechanism to distinguish a legitimate ACK cascade deletion from a hostile one.

**Impact:** Silent payload loss with false delivery confirmation. The originator believes the message was delivered; the recipient never received it.

**Status: OPEN GAP — highest severity.**

**Candidate mitigation — transition sanity check (implementation guidance):**
When a Raft detects its lower_dock cleared (transition to recv-polling mode), it should verify that its upper_dock is also empty before concluding a legitimate ACK occurred. A legitimate cascade clears both Dock-sides in sequence; a hostile delete clears only the lower_dock.

*Note: This is implementation guidance, not a protocol change. It reduces false positives but does not prevent the attack.*

**Candidate mitigation — Waterway envelope MAC (see AV-04 and Raft Trust section):**
A MAC covering the Waterway contents would allow Rafts to detect substitution but not premature deletion. Deletion leaves no artifact to verify against. Full defense against cascade disruption may require an application-level delivery acknowledgment separate from the ACK cascade.

---

### AV-04: Blob Substitution

**Description:** Attacker reads a blob from a Waterway (capturing the ciphertext), deletes it, and writes a different blob in its place. Downstream Rafts forward the substituted blob. The receiving endpoint attempts decryption, GCM authentication fails, and the message is silently dropped.

**Access required:** Read + write access to any provider in the pipeline.

**Current defense:** AES-256-GCM authentication causes the substituted blob to fail decryption. The payload is not compromised. However, the message is lost and the pipeline stalls in a manner indistinguishable from a legitimate stall.

**Impact:** Selective message blocking. An attacker can choose which messages to block without being detected until the endpoint notices missing delivery.

**Status: PARTIAL — content protected, delivery not.**

**Candidate mitigation — Waterway envelope MAC:**
Introduce a Waterway-level MAC computed with a `wateway_mac_key` derived from the SHARED_SECRET but distinct from the payload encryption key:

```
stored_bytes = MAC(wateway_mac_key, ciphertext) || ciphertext
```

Rafts verify the MAC before forwarding. A substituted blob produces an invalid MAC and is rejected at the Raft rather than forwarded to the endpoint. This catches substitution at the earliest possible hop.

*Design questions: waterway_mac_key must be shared with Rafts. See Raft Trust section below.*

---

### AV-05: Replay Injection

**Description:** Attacker captures a previously valid ciphertext blob (read from storage at any point in the past) and re-injects it into an empty Waterway. Unlike AV-01, this blob has a valid GCM authentication tag. The receiving endpoint successfully decrypts it and processes a stale message as current.

**Access required:** Read access (to capture), then write access (to replay) to any provider in the pipeline.

**Current defense:** Dependent on wire format. AES-256-GCM generates a unique nonce per encryption, but if the spec does not mandate nonce sequencing or freshness verification, replays are not detected.

**Impact:** Stale message injection. In protocols carrying commands or state updates, replaying old messages could have significant semantic consequences.

**Status: OPEN GAP — dependent on encryption spec.**

**Candidate mitigation — sequence number in AAD:**
Mandate a monotonically increasing sequence number in the GCM Additional Authenticated Data (AAD). The AAD is authenticated by GCM's tag but not encrypted, so endpoints can verify sequence without decryption overhead. A replayed blob carries a stale sequence number, which the receiving endpoint rejects.

*Spec location: encryption.md — wire format section.*

---

### AV-06: Traffic Analysis (Metadata Observation)

**Description:** Attacker observes Waterway existence, timing of appearance/disappearance, and blob sizes without reading content. Over time, this reveals communication patterns: when endpoints are active, message cadence, and approximate payload sizes.

**Access required:** Read access (or even just `exists()` polling) on any provider.

**Current defense:** Acknowledged and accepted in the existing threat model. No protocol-level mitigation.

**Impact:** Behavioral information leakage. Does not compromise message content.

**Status: ACKNOWLEDGED — accepted in current threat model.**

**Candidate mitigation — epoch-derived Waterway name rotation:**
Waterway names rotate per message (or per time epoch), derived from SHARED_SECRET. An observer sees opaque, uncorrelated blob names — they cannot link Waterway activity across messages or correlate two Waterways as belonging to the same pipeline.

```
waterway_name = HKDF(SHARED_SECRET, channel || direction || message_seq)
```

**Candidate mitigation — blob size normalization:**
Pad all blobs to a fixed size (or to fixed size buckets) before writing to storage. Defeats size-based traffic analysis. Implementation concern rather than protocol change.

*Note: Timing analysis is inherent to any polling-based async system and cannot be fully eliminated without fundamentally changing the protocol's async nature.*

---

### AV-07: Storage Provider as Adversary (Availability Attack)

**Description:** The storage provider itself (or an attacker with equivalent access) deletes all content from a Channel, making it silently stall indefinitely. Distinct from AV-03 in that this is a blanket availability attack rather than targeted message blocking.

**Access required:** Administrative access to the storage provider.

**Current defense:** The heartbeat/observability protocol (SPEC_v0.6_resilience.md) surfaces stalls but does not prevent them. The bi-Dock design ensures no state corruption — the pipeline can be reconstructed — but content in transit is lost.

**Impact:** Channel denial of service. No confidentiality or integrity impact.

**Status: ACKNOWLEDGED — addressed by resilience spec, not security spec.**

---

## Raft Trust Extension

The current spec enforces strict crypto-blindness at Rafts: Rafts have no cryptographic keys and cannot inspect payload content. This is enforced at the package dependency level.

The candidate mitigations above (AV-02, AV-04) require Rafts to hold a `waterway_mac_key` derived from the SHARED_SECRET. This is a deliberate, minimal extension of Raft trust. The key distinction:

| Key type | Held by | Capability granted |
|----------|---------|-------------------|
| `payload_enc_key` | Endpoints only | Encrypt/decrypt message content |
| `waterway_mac_key` | Rafts + endpoints | Verify Waterway integrity — no content access |
| `waterway_name_key` | Rafts + endpoints | Derive valid Waterway names — no content access |

The original crypto-blindness constraint was about **payload content**, not about protocol-level integrity operations. A Raft holding a MAC verification key gains no ability to read, modify, or forge message content. The trust extension is justified by the isolation property: Rafts are hard targets, and the security benefit of Waterway integrity verification outweighs the minimal trust extension required.

**Proposed key derivation tree:**

```
SHARED_SECRET
├── HKDF("waterway-names")   → waterway_name_key    (rafts + endpoints)
├── HKDF("waterway-mac")     → waterway_mac_key     (rafts + endpoints)
└── HKDF("payload-enc")  → payload_enc_key  (endpoints only)
```

---

## Cryptographic Storage Independence

The combination of the above mitigations yields a property worth naming explicitly in the spec:

> **Cryptographic storage independence:** The security guarantee holds regardless of the confidentiality or integrity of the underlying storage medium. Physical access to storage grants an attacker access only to opaque blobs with unpredictable names, with no ability to verify or forge their integrity, and no ability to correlate Waterway activity across messages or pipelines.

This is a stronger claim than the current spec's threat model, which frames storage as "untrusted infrastructure" defended by payload encryption. With Waterway envelopes and derived names, storage is **cryptographically opaque** rather than merely encrypted.

---

## Implementation Priority

| ID | Attack | Severity | Complexity | Recommended action |
|----|--------|----------|------------|-------------------|
| AV-05 | Replay injection | High | Low | Add sequence number to AAD in encryption.md |
| AV-04 | Blob substitution | High | Medium | Spec Waterway envelope MAC; extend Raft key model |
| AV-02 | Waterway squatting | Medium | Medium | Spec derived Waterway names; coordinate with Raft key model |
| AV-03 | Cascade disruption | High | High | Implementation guidance now; deeper mitigation TBD |
| AV-06 | Traffic analysis | Low | Medium | Derived Waterway names partially address; accept remainder |
| AV-01 | Payload injection | — | — | Already neutralized |
| AV-07 | Provider availability | Low | — | Addressed by resilience spec |

---

*End of document. See spec/encryption.md and spec/security-model.md for normative spec language derived from this analysis.*
