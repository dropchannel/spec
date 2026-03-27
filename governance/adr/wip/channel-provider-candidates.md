# DockProvider Candidates — Deferred Assessment

*Captured from design exploration session. Not a spec. For later evaluation and prioritization.*

---

## Context

The existing Dock implementations are: `gcs`, `dropbox`, `local`, `httprelay`.

This session explored additional backend candidates, with a focus on old, stable, battle-tested file/storage protocols. The DockProvider interface requirements are minimal:

- `write(channel, waterway, filename, data) -> bool` — write only if filename absent (atomic if possible)
- `peek(channel, waterway, filename) -> bytes|None` — non-consuming read
- `read(channel, waterway, filename) -> bytes|None` — consuming read (terminating endpoint only)
- `exists(channel, waterway, filename) -> bool` — primary polling operation
- `delete(channel, waterway, filename) -> None` — idempotent; used in ACK cascade

The ACK cascade imposes one non-obvious requirement: **a Raft must be able to poll a Waterway it wrote on a *remote* store and detect that a third party has deleted it.** This rules out backends where Waterway identity is server-assigned and opaque to the writer.

---

## Candidates

### SFTP

**Status: Recommended for implementation**

File semantics map cleanly to the DockProvider interface. Atomic write via temp-file + rename (`SSH_FXP_RENAME`). Strong auth via SSH keys — no credential exposure. Widely deployed on hosting infrastructure. Python `paramiko` library provides a clean implementation path.

ACK cascade: no issues. Waterway identity is a filepath chosen by the writer; `exists()` is a stat call on that path.

**Known concerns:**

- Port 22 occasionally firewalled where 443 is not
- Key management adds operational overhead vs. PSK-style auth
- Rename atomicity is POSIX-guaranteed on local filesystems but SSH server-dependent for remote; most real implementations handle it correctly

**Open questions:**

- Confirm `paramiko` rename behavior on common server implementations (OpenSSH, ProFTPD SFTP subsystem)
- Define key distribution model — does it fit within existing PSK/env-var conventions or require separate handling?

---

### WebDAV

**Status: Recommended for implementation**

HTTP-based (RFC 4918). Methods map directly: `PUT` = write, `GET` = peek/read, `HEAD` = exists (no body transfer), `DELETE` = delete. Traverses proxies and firewalls trivially (port 443). `LOCK`/`UNLOCK` not used.

ACK cascade: no issues. Waterway identity is a URL path chosen by the writer.

Deployment targets with free tiers: Nextcloud, ownCloud, Box, many NAS devices (Synology, QNAP), Apache `mod_dav`.

**Known concerns:**

- No standard conditional PUT ("write only if absent") in the WebDAV spec. Workaround: `PUT` then check; or use `If-None-Match: *` header if server supports it (not universally implemented). TOCTOU race window exists.
- Atomicity of `PUT` itself is server-dependent; most implementations are atomic at the filesystem level
- Server implementations vary in standards compliance; test against target deployments

**Open questions:**

- Audit `If-None-Match: *` support across target server implementations (Nextcloud, Apache mod_dav, nginx WebDAV module)
- Determine whether the TOCTOU gap on write is acceptable given the two-party, cooperative-Raft threat model

---

### FTP / FTPS

**Status: Low priority — viable but superseded by SFTP**

File semantics work. Atomic write requires `STOR` to temp file + `RNFR`/`RNTO` rename — same pattern as SFTP but more cumbersome. Widely deployed on shared hosting.

ACK cascade: no issues (same filepath-based Waterway identity as SFTP).

**Known concerns:**

- Credentials travel plaintext without FTPS (explicit TLS). Plain FTP should be considered unacceptable; FTPS required.
- Active vs. passive mode complexity; passive mode always used in outbound-only polling context, which resolves this
- SFTP is strictly better in every dimension where FTPS is adequate

**Verdict:** Only worth implementing if a deployment target exposes FTP/FTPS but not SFTP. Treat as fallback.

---

### IMAP

**Status: Interesting — requires design decision before implementation**

Unorthodox but genuinely workable. Email folders as Waterway containers; messages as Waterway contents. One folder per Waterway (`{channel}.{waterway}.upper`, `{channel}.{waterway}.lower`). Waterway occupancy = message count in folder (0 or 1). `APPEND` = write, `FETCH` = peek/read, folder `EXISTS` response count = exists(), `STORE \Deleted` + `EXPUNGE` = delete.

This design restores stateless restartability: a restarted Raft inspects both folders by message count, no UID tracking needed.

ACK cascade: **works under the one-folder-per-Waterway design.** A Raft polls its send-folder for message count; when it drops to 0, the downstream party has deleted it, triggering the upstream cascade step. UID tracking is not required.

**Known concerns:**

- No conditional APPEND ("write only if folder empty"). Must SELECT, check EXISTS == 0, then APPEND. TOCTOU race window — acceptable under cooperative-Raft model, notable in spec.
- Folder isolation semantics are implementation-dependent. Most servers handle one-message-per-folder correctly but this is not RFC-guaranteed.
- Custom folder naming may be subject to server-imposed restrictions or normalization (e.g., case folding, namespace prefixes like `INBOX.`).
- Connection overhead: per-poll TCP+TLS handshake unless connection is held open. IMAP IDLE not used (push model incompatible with outbound-only polling).
- Deployment target: any IMAP server with folder creation privileges. Free tier: Fastmail, Gmail (app password), self-hosted (Dovecot, Cyrus).

**Open questions:**

- Is the "any email account as a relay" story worth the implementation complexity vs. WebDAV?
- Define folder naming convention and namespace handling across server implementations
- Specify behavior if server imposes message count limits or folder quotas
- Confirm that `EXPUNGE` by one client is immediately visible to another client's subsequent `SELECT` (RFC 3501 guarantees this; verify in practice against Dovecot/Gmail)

---

### S3-compatible (MinIO, Backblaze B2, Cloudflare R2)

**Status: Near-identical to GCS — low incremental effort**

Object storage semantics are essentially the same as GCS. Conditional PUT via `If-None-Match: *` header gives atomic write-if-absent. Strong consistency (S3 since 2020, R2 always, B2 always).

**Known concerns:**

- Existing GCS implementation should be reviewed for how much is GCS-specific vs. generic object storage. May be near-zero incremental work.
- Credential model varies (HMAC key pairs vs. service accounts vs. API tokens) — needs abstraction

**Open questions:**

- Audit GCS provider for portability; estimate effort to extract a generic S3-compatible provider

---

### SMB/CIFS

**Status: Niche — home NAS hop use case only**

File semantics work. Python `smbprotocol` library. Good fit for a home NAS (Synology, QNAP, TrueNAS) acting as a hop Raft on the home network segment.

**Known concerns:**

- Should never be exposed to the public internet. LAN/VPN only.
- SMBv1/v2 have known vulnerabilities; SMBv3 with encryption required
- Adds meaningful implementation complexity for a narrow deployment scenario

**Verdict:** Defer indefinitely unless there is a specific deployment need.

---

### NFS

**Status: Not recommended**

No encryption without a VPN tunnel. LAN-only. Concurrent-write safety is server-dependent. No meaningful advantage over SMB for home NAS use cases, and SMB is itself low priority.

---

### Gopher

**Status: Discarded**

Client-initiated writes are not part of the Gopher protocol. Read-only from the client side. Not viable as a DockProvider without implementing a custom server, at which point Gopher is not meaningfully in use.

---

## Priority Order for Implementation

1. **WebDAV** — best firewall traversal, free backends already deployed (Nextcloud/ownCloud), HTTP-native
2. **SFTP** — cleanest semantics, strong auth, widely deployed on hosting
3. **IMAP** — unique "any email account" story; design question on TOCTOU and folder semantics needs resolution first
4. **S3-compatible** — low effort if GCS provider is sufficiently generic; audit first
5. **FTP/FTPS** — fallback only; implement if a deployment target requires it
