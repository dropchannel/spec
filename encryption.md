# Encryption

All payload content in DropChannel is end-to-end encrypted between endpoints. The relay
and any intermediate Rafts store and forward opaque blobs only. No participant in the
pipeline other than the two endpoints holds or has access to the shared secret.

---

## Algorithm

**AES-256-GCM** — authenticated symmetric encryption.

| Parameter | Value |
|-----------|-------|
| Key size | 256 bits (32 bytes) |
| Nonce size | 96 bits (12 bytes), randomly generated per message |
| Tag size | 128 bits (16 bytes, GCM default) |

---

## Wire Format

```
base64( nonce[12 bytes] + ciphertext + tag[16 bytes] )
```

The nonce is randomly generated for every message and prepended to the ciphertext. The
relay stores and serves this concatenated blob without modification. The tag is appended
by AES-GCM and included in the ciphertext bytes as returned by the algorithm.

---

## Key Distribution

The shared secret is a 256-bit (32-byte) pre-shared key (PSK) distributed out-of-band
between the two endpoint operators. It is never transmitted through the relay or any
intermediate Raft.

Both directions of a channel use the same shared key.

---

## Security Properties

- A tampered ciphertext is detectable: AES-GCM's authentication tag will fail
  verification on decryption, raising an error before any plaintext is returned.
- Nonce reuse with the same key would be catastrophic for AES-GCM confidentiality.
  Random 96-bit nonce generation per message makes collision probability negligible
  at all realistic message volumes.
- The relay's transport security (e.g. HTTPS) is additive. Confidentiality does not
  depend on it — termination at the relay does not expose plaintext.
- **The encryption guarantee is independent of storage security.** Because Rafts and
  clients are the hard targets (see [`security-model.md`](security-model.md)) and
  storage is the primary attack surface, the encryption layer is specifically designed
  to hold even against a fully compromised storage provider. An attacker with complete
  read access to storage obtains only ciphertext. This guarantee holds only so long as
  endpoints remain isolated from inbound connections and the shared secret is not
  exposed through other means.

**Known gap — replay attacks.** The current wire format does not include a sequence
number or other nonce-freshness binding in the authenticated additional data (AAD). A
captured valid ciphertext blob can be re-inserted into an empty Waterway and will pass
AES-GCM authentication at the receiving endpoint. Adding a monotonically increasing
sequence number to the AAD field would close this gap and is a candidate for a future
wire format revision.

---

## Scope

This specification covers payload encryption only. Heartbeat metadata (introduced in
v0.6) is transmitted in plaintext. See [`security-model.md`](security-model.md) for the
full scope of what is and is not protected.
