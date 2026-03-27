# DropChannel ADR Master List

One ADR per meaningful, non-obvious decision with lasting consequences.
Use [TEMPLATE.md] for new ADRs.

---

## Terminology reference

Some terms used in earlier ADRs have been renamed by subsequent decisions. This table maps old terms to their current equivalents for readers navigating historical entries.

| Old term | Current term | Renamed in |
|----------|--------------|------------|
| ChannelProvider | Dock | ADR-0037 |
| Endpoint | Client | ADR-0037 |
| Slot | Waterway | ADR-0037 |
| Node (forwarding participant) | Raft | ADR-0037 |
| dropchannel-endpoint | dropchannel-client | ADR-0037 |
| dropchannel-channel | dropchannel-dock | ADR-0038 |
| DockProvider | Dock | ADR-0038 |
| Relay / Server | Dock (impl concern only) | ADR-0038 |

---

## From SPEC_v0.1 — Initial Relay

| # | Title | Decision | Alternatives considered |
|---|-------|----------|------------------------|
| 0001 | GCS as blob state store | Use GCS as the sole state store; blob presence = pending, absence = handled | Firestore (overkill for raw blobs), Redis (always-on cost), Pub/Sub (complexity) |
| 0002 | AES-256-GCM PSK encryption | Symmetric pre-shared key; nonce randomly generated per message and prepended to ciphertext | Asymmetric key exchange (out of scope for two-party trust model) |
| 0003 | Server is encryption-agnostic | Server stores and serves opaque blobs only; crypto is strictly an endpoint concern | Server-side encryption (breaks E2E guarantee) |
| 0004 | Fire-and-forget with polling | ClientA posts and polls separately; server returns 202 immediately | Long-polling (billing during wait), synchronous response (Cloud Run cost) |

---

## From SPEC_v0.2 — Symmetric Redesign

| # | Title | Decision | Alternatives considered |
|---|-------|----------|------------------------|
| 0005 | Symmetric peers | Both peers run identical client code configured with send/recv slots; asymmetric ClientA/ClientB roles removed | Keeping distinct role-specific codebases |
| 0006 | Generic slot names | Slot names are arbitrary strings agreed out-of-band; server treats them as opaque path components | Hardcoded `request`/`response` slot names (encoded topology in the server) |
| 0007 | Write guard: one blob per slot | Server returns 409 if a slot is already occupied; blob presence = pending enforced as an invariant | Queuing/overwriting (would require sequencing and complexity) |

---

## From SPEC_v0.3 — ChannelProvider Abstraction

| # | Title | Decision | Alternatives considered |
|---|-------|----------|------------------------|
| 0008 | ChannelProvider abstraction and HTTP relay as one provider | Extract a `ChannelProvider` ABC; demote Cloud Run/GCS from required infrastructure to one provider among many (`httprelay`); Dropbox, local filesystem, and direct GCS become first-class alternatives | Keeping GCS/Cloud Run as the only transport |

---

## From SPEC_v0.4 — Node Primitive and Propagation Protocol

| # | Title | Decision | Alternatives considered |
|---|-------|----------|------------------------|
| 0009 | `peek()` as fifth ChannelProvider operation | Add a non-consuming read to the interface; required for nodes to forward blobs without triggering the ACK cascade prematurely | Nodes using `read()` and re-writing (would corrupt the delete-as-ACK signal) |
| 0010 | Delete-as-ACK cascade | Delivery confirmation propagates backward as each node detects its send-slot emptied and deletes its recv-slot; terminating endpoint's `read()` is the only destructive read | Explicit ACK messages (additional state and mechanism); no delivery confirmation (V0.1–0.3 behavior) |
| 0011 | Node is crypto-blind by design | Nodes have no `SHARED_SECRET` and never encrypt or decrypt; blobs are opaque bytes to a node | Nodes with decryption capability (breaks E2E guarantee) |
| 0012 | Node state machine: two steady-state polling modes | Node polls only one slot per cycle in steady state (recv-slot in recv-polling mode, send-slot in send-polling mode); both slots inspected only at startup | Polling both slots every cycle (double the provider operations per cycle) |
| 0013 | Slot name is invariant across hops | Every participant in a physical pipeline uses the same slot name on both recv and send sides; the slot name does not change as a blob moves through nodes | Per-hop renaming (would require topology knowledge at each node) |
| 0014 | Directional env var namespacing for nodes | Node provider env vars prefixed `RECV_` / `SEND_` (e.g. `RECV_RELAY_URL`, `SEND_GCS_BUCKET_NAME`); endpoint env vars unchanged; per-direction providers fall back to `CHANNEL_PROVIDER` | Separate config files per side; positional arguments |
| 0015 | `send` polls send-slot for ACK, not recv-slot for response | After writing, originating endpoint polls its own send-slot for `exists() == false` as delivery confirmation; response (if any) is a separate application concern on the return pipeline | Polling recv-slot for response (conflates delivery confirmation with application-layer reply) |

---

## From SPEC_v0.5 — Monorepo Package Split

| # | Title | Decision | Alternatives considered |
|---|-------|----------|------------------------|
| 0016 | Monorepo split into four packages by role | `dropchannel-channel`, `dropchannel-endpoint`, `dropchannel-node`, `dropchannel-server` as independently installable packages; single shared `dropchannel` namespace package | Single package (obscures role boundaries); separate repositories (cross-cutting changes require multi-repo coordination) |
| 0017 | Crypto isolation enforced at package boundary | `dropchannel-node` and `dropchannel-server` do not depend on `cryptography`; isolation is a hard package-manager property, not a convention | Convention-only (relies on discipline; fails silently if violated) |
| 0018 | `check_isolation.py` as build-time executable check | Subprocess-based script that verifies no `cryptography` module appears in `sys.modules` after importing node/server; runs before pytest in `make check` | Pytest test (in-process; false pass if `cryptography` already imported by earlier test) |
| 0019 | `http` provider renamed to `httprelay` | Provider identifier `http` replaced with `httprelay` everywhere; name describes the drop mechanism (an HTTP relay), not the transfer protocol | Keeping `http` (misleadingly implies HTTP is the transport layer, not just one drop mechanism) |
| 0020 | Tests remain in single top-level `tests/` directory | Single pytest run covers all four packages from monorepo root; per-package subsets filtered by filename pattern | Per-package `tests/` directories (complicates coordinated interface changes) |

---

## From SPEC_v0.6 — Heartbeat Protocol

| # | Title | Decision | Alternatives considered |
|---|-------|----------|------------------------|
| 0021 | Heartbeat runs on meta slots, separate from primary slots | Observability layer uses a distinct namespace (`meta/`) within the same provider position; primary slot semantics are completely unchanged | Piggyback on primary slots (couples observability to payload timing); out-of-band channel (requires additional infrastructure) |
| 0022 | Heartbeat content is unencrypted | Meta files contain only UUIDs, timestamps, counters, and `HEARTBEAT_NOT_FOUND`; no payload content or routing secrets | Encrypted heartbeat via shared `DRIP_KEY` (deferred; noted in out-of-scope) |
| 0023 | Nodes propagate upstream heartbeat content, clients do not | A node reads the non-self heartbeat file from one side and appends its own line before writing to the other side, forming a chain; clients write only their own file to both sides | Clients also propagating (unnecessary; clients are pipeline terminals) |
| 0024 | `HEARTBEAT_NOT_FOUND` as explicit sentinel | When a node finds no non-self heartbeat on a given side, it writes the literal string `HEARTBEAT_NOT_FOUND` as the upstream content rather than an empty prefix | Omitting the field (makes it ambiguous whether the node looked and found nothing vs. didn't look) |
| 0025 | Persistent UUID identity per participant | Each node and endpoint generates a UUID4 on first startup and persists it to `dropchannel-identity.json`; reloaded on all subsequent startups | Ephemeral UUID per session (makes heartbeat chain entries unrecognizable across restarts) |
| 0026 | Startup purge of own heartbeat files | On startup, each participant deletes all its own heartbeat files from both meta slots before writing fresh ones; each type purges only its own files | No purge (stale files from a previous session could confuse chain readers) |
| 0027 | Client heartbeat is event-driven at status transitions, then HI-driven | Immediate write on `ready`/`busy`/`closing` transitions; HI-interval repeat writes while status holds; `closing` written once with no follow-up | Purely interval-driven (status changes may not be visible for up to one full HI interval) |

---

## From Naming and Architecture Sessions

| # | Title | Decision | Alternatives considered |
|---|-------|----------|------------------------|
| 0028 | Protocol named Winch | Hold-and-cascade propagation protocol named Winch | PartyHorn, Tideway, and others explored |
| 0029 | DropChannel as GitHub org, not Winch | Org name reflects the runtime/brand; Winch is one protocol within it | `winch` as org name (would imply the org is a single-protocol project) |
| 0030 | Protocol repos named `{name}-protocol` | `winch-protocol`, `ring-protocol`, `piston-protocol` | `{name}` alone (ambiguous kind), `{name}-spec` (redundant with repo purpose), combined `specs/` repo |
| 0031 | `dropchannel` repo as system-level specification | Owns the ChannelProvider interface, protocol registry, dispatch rules, security model, and conformance requirements; separate from any protocol repo | Embedding system spec inside `winch-protocol` (wrong scope) or `dropchannel-py` (wrong layer) |
| 0032 | `dropchannel-py` as Python implementation repo name | Language-suffixed; anticipates `dropchannel-ru` etc. | `dropchannel` alone (no language signal), `relay` (predates the rename) |
| 0033 | Channel name prefix as protocol dispatch mechanism | Node reads `winch-`, `ring-`, `piston-` prefix to select the coordination state machine; prefix is metadata, not payload | Separate config field per protocol; topology-level protocol assignment |
| 0034 | Unrecognized prefix: halt and log | Node halts with a log entry on an unrecognized channel prefix | Skip and continue (silent data loss), apply default protocol (masks misconfiguration) |
| 0035 | Ring as a separate protocol for streaming use cases | Backpressure is the correct behavior for Winch use cases; streaming/telemetry needs a rolling-window no-ACK protocol; Ring is a sibling, not a Winch variant | Adding a "no-ACK mode" to Winch (conflates two semantically distinct protocols) |

---

## From SPEC_v0.9 — Agent and Worker Model

| # | Title | Decision | Alternatives considered |
|---|-------|----------|------------------------|
| [0036](./ADR-0036.md) | Agent merged into dropchannel-node rather than separate package | Merge the Agent into dropchannel-node as an agent/ submodule. Do not create a separate dropchannel-agent package. | Separate dropchannel-agent package (incorrect deployment unit); Threads instead of subprocesses (would have affected placement) |

---

## From Terminology and Architecture Consolidation

| # | Title | Decision | Alternatives considered |
|---|-------|----------|------------------------|
| [0037](./ADR-0037.md) | Unified Vocabulary and Protocol Renaming | Adopt a waterway system metaphor as the unified conceptual framework for all DropChannel structural and protocol terminology. | Maritime shipping metaphor (diluted core concept of passive delivery) |
| [0038](./ADR-0038.md) | Consolidate Storage Abstraction Terminology to "Dock" | "Dock" is the single term for both the interface and the conceptual unit; "DockProvider", "Relay", and "Server" retired as spec-level terms; `dropchannel-channel` package renamed to `dropchannel-dock`. | Keep DockProvider/Dock distinction (not maintained in practice); DockProxy as replacement for Relay (overstates importance of HTTP server concept at spec level) |
