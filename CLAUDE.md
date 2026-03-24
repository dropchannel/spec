# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is the **authoritative system specification** for DropChannel — a multi-protocol, storage-backed coordination runtime for encrypted, asynchronous, point-to-point communication. It contains only markdown specification documents. There is no code, no build system, and no tests.

Actual implementations live in sibling repositories (e.g., `dropchannel/dropchannel-py`). Individual protocol specifications live in their own repos (`dropchannel/tideway-protocol`, `dropchannel/riverway-protocol`).

## Working in This Repo

**Starting a new session:** For a full context handoff, `HANDOFF.md` contains a complete system summary. For targeted work, paste or refer to the relevant spec file(s).

**Spec files will need updating when:**

* The DockProvider interface gains new operations
* A new protocol is registered or an existing one gets a spec repo
* Heartbeat encryption (`DRIP_KEY`) is specced out
* The security model scope changes
* The Agent config schema changes (e.g. new top-level sections)

## Working Rules

* Spec files are source of truth. Implementation behavior defers to spec.
* Do not retroactively reorganize committed files. Correct errors; don't restructure.
* Do not overclaim. Scope security assertions carefully:
  * Heartbeat metadata is plaintext.
  * "Shared storage" not "untrusted shared infrastructure".
* One concern per file. Propose a new file rather than expanding an existing one's scope.

## Spec File Map

| File | Owns |
|------|------|
| `README.md` | System overview, component definitions, DockProvider interface summary, protocol registry, conformance rules |
| `dock-provider.md` | Full DockProvider interface: 5 primary Waterway ops + 4 meta Waterway ops, contracts, Waterway naming |
| `encryption.md` | AES-256-GCM wire format, key distribution, security properties |
| `security-model.md` | Trust boundaries, threat model, open items |
| `protocol-registry.md` | Registered protocols, dispatch rules, how to add a protocol |
| `observability.md` | Two-layer observability overview (heartbeat vs. telemetry) |
| `heartbeat.md` | Per-hop liveness protocol using meta Waterways |
| `telemetry.md` | External monitoring side-channel: blob schema, topology reconstruction |
| `agent.md` | Agent/Worker runtime: TOML config schema, worker lifecycle, backoff, telemetry participation |

## Architecture Summary

**Four components:** Endpoint (encrypts/decrypts, holds SHARED_SECRET), Raft (crypto-blind forwarder), Dock (storage interface), Relay (Dock over HTTP).

**Channel and Waterway naming:** A Channel is a bare namespace (e.g. `chat`) with no protocol prefix. Waterways live inside Channels and carry the prefix: `{prefix}-{identifier}` (e.g. `tideway-bob`). A single Channel may contain multiple Waterways. The Waterway prefix drives protocol dispatch. Registered prefixes: `tideway-`, `riverway-`, `telemetry-`, `heartbeat-`.

**Key design constraint:** Rafts must never hold encryption keys and must not have cryptographic library dependencies for payload operations. This is verifiable at the package level in conformant implementations.

**DockProvider interface:** `write`, `read` (destructive/consume-on-read), `peek` (non-destructive), `exists`, `delete` — plus meta Waterway variants (`meta_write`, `meta_read`, `meta_delete`, `meta_list`). All operations take `(channel, waterway, filename, ...)` parameters.

**Agent/Worker runtime:** Agent is a host-level supervisor owning a 5-section TOML config (`[agent]`, `[log]`, `[telemetry]`, `[docks]`, `[workers]`). Workers are subprocesses that receive config via stdin as JSON and emit structured JSON logs. Worker restart uses exponential backoff capped at 30s; degraded state triggers after 5 crashes within 60s of spawn.

## Known Gaps (as of v0.6)

* Heartbeat encryption via `DRIP_KEY` — deferred
* Replay attack mitigation (sequence number in AAD) — not yet specced
* Waterway integrity and existence opacity mitigations — not yet specced
