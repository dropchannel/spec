# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is the **authoritative system specification** for DropChannel — a multi-protocol, storage-backed coordination runtime for encrypted, asynchronous, point-to-point communication. It contains only markdown specification documents. There is no code, no build system, and no tests.

Actual implementations live in sibling repositories (e.g., `dropchannel/dropchannel-py`). Individual protocol specifications live in their own repos (`dropchannel/tideway-protocol`, `dropchannel/riverway-protocol`).

**Scope boundary:** Protocol-specific state machines do not belong here — they live in their own repos. If an edit belongs only in `dropchannel/tideway-protocol` or `dropchannel/riverway-protocol`, do not duplicate it here.

## Working in This Repo

**Spec files will need updating when:**

* The DockProvider interface gains new operations
* A new protocol is registered or an existing one gets a spec repo
* Heartbeat encryption (`DRIP_KEY`) is specced out
* The security model scope changes
* The Agent config schema changes (e.g. new top-level sections)

## Working Rules

* Do not retroactively reorganize committed files unless instructed. Correct errors; don't restructure.
* Do not overclaim. Scope security assertions carefully:
  * Heartbeat metadata is plaintext.
  * "Shared storage" not "untrusted shared infrastructure".
* One concern per file. Propose a new file rather than expanding an existing one's scope.
* Cross-repo links (to other repos in the DropChannel org) use <https://github.com/dropchannel/> as the URL root. Intra-repo links use relative URLs.
* When producing spec changes that add a new file, rename a component, resolve a Known Gap, or promote an ADR into `governance/adr/`, also update CLAUDE.md to reflect the new state.

**Do not use retired terms.** Current vocabulary:

| Term | Retired terms |
|---|---|
| Dock / DockProvider | ChannelProvider, channel-provider |
| Waterway | slot, reach |
| Raft | Node |
| Channel | channel_id (as namespace) |
| Riverway | Conveyer, Current |
| Tideway | (none) |

## Spec File Map

| File | Owns |
|------|------|
| `README.md` | System overview, component definitions, DockProvider interface summary, protocol registry, conformance rules |
| `spec/dock-provider.md` | Full DockProvider interface: 5 primary Waterway ops + 4 meta Waterway ops, contracts, Waterway naming |
| `spec/encryption.md` | AES-256-GCM wire format, key distribution, security properties |
| `spec/security-model.md` | Trust boundaries, threat model, open items |
| `spec/protocol-registry.md` | Registered protocols, dispatch rules, how to add a protocol |
| `runtime/observability.md` | Two-layer observability overview (heartbeat vs. telemetry) |
| `runtime/heartbeat.md` | Per-hop liveness protocol using meta Waterways |
| `runtime/telemetry.md` | External monitoring side-channel: blob schema, topology reconstruction |
| `runtime/agent.md` | Agent/Worker runtime: TOML config schema, worker lifecycle, backoff, telemetry participation |

## Architecture Summary

**Four components:** Endpoint (encrypts/decrypts, holds SHARED_SECRET), Raft (crypto-blind forwarder), Dock (storage interface), Relay (Dock over HTTP).

**Channel and Waterway naming:** A Channel is a bare namespace (e.g. `chat`) with no protocol prefix. Waterways live inside Channels and carry the prefix: `{prefix}-{identifier}` (e.g. `tideway-bob`). A single Channel may contain multiple Waterways. The Waterway prefix drives protocol dispatch. Registered prefixes: `tideway-`, `riverway-`, `telemetry-`, `heartbeat-`.

**Key design constraint:** Rafts must never hold encryption keys and must not have cryptographic library dependencies for payload operations. This is verifiable at the package level in conformant implementations.

**DockProvider interface:** `write`, `read` (destructive/consume-on-read), `peek` (non-destructive), `exists`, `delete` — plus meta Waterway variants (`meta_write`, `meta_read`, `meta_delete`, `meta_list`). All operations take `(channel, waterway, filename, ...)` parameters.

**Agent/Worker runtime:** Agent is a host-level supervisor owning a 5-section TOML config (`[agent]`, `[log]`, `[telemetry]`, `[docks]`, `[workers]`). Workers are subprocesses that receive config via stdin as JSON and emit structured JSON logs. Worker restart uses exponential backoff capped at 30s; degraded state triggers after 5 crashes within 60s of spawn.

## ADR format

ADRs live in the [`dropchannel`](https://github.com/dropchannel/dropchannel) repo.
ADR numbers use 4-digit zero-padded format: `ADR-NNNN`.

Standard sections and Format: See [governance/adr/TEMPLATE.md](governance/adr/TEMPLATE.md)

The three tiers:

* **Full ADR** — load-bearing decisions where the Consequences section is the point. Write as individual `ADR-NNNN.md`.
* **Short ADR** — clear rationale, fits in 2–3 paragraphs. Individual file.
* **Log entry** — table row in the `dropchannel` repo's ADR index is sufficient.

Priority candidates for full ADR treatment: 0008, 0010, 0017, 0021, 0033.

Individual ADR files (`ADR-0001.md` through `ADR-0035.md`) have not been written yet —
the master index in the `dropchannel` repo is the current extent of ADR content.
ADR-0036 and ADR-0037 have individual files.

The 35 earlier ADRs have not been triaged into full / short / log-entry tiers.

**Starting a new ADR-writing session:** provide the `dropchannel` repo's ADR index,
then specify which ADRs to write and at what depth. Example:
> "Read CLAUDE.md and the ADR index. Write full ADR documents for ADRs 0008, 0010,
> and 0017. Use the standard ADR format: Title, Status, Context, Decision, Consequences."

## Known Gaps (as of v0.6)

* Heartbeat encryption via `DRIP_KEY` — deferred
* Replay attack mitigation (sequence number in AAD) — not yet specced
* Waterway integrity and existence opacity mitigations — not yet specced
