# spec — Working Notes

## What this repo is

The system-level specification for the DropChannel runtime. Owns concerns that cut
across all protocols and all implementations: the ChannelProvider interface, encryption
standard, security model, protocol dispatch rules, observability layer, and the
Agent/Worker runtime specification.

## Current state

The following files exist and are current through v0.6 of the implementation:

| File | Contents | Status |
|------|----------|--------|
| `channel-provider.md` | Full ChannelProvider interface: 5 primary slot operations + 4 meta slot operations, contracts, provider implementations table, slot naming convention | Current |
| `encryption.md` | AES-256-GCM spec, wire format, key distribution, security properties, known replay gap, scope note re: heartbeat plaintext | Current |
| `security-model.md` | What is/isn't protected, trust boundaries, threat model scope, open items (slot integrity, existence opacity, replay, silent corruption) | Current |
| `protocol-registry.md` | Registered protocols (tide, ring, piston), dispatch rules, protocol selection rationale, how to add a protocol | Current |
| `observability.md` | Two-layer observability overview: heartbeat vs. telemetry, when to use each, what neither provides | Current |
| `heartbeat.md` | Per-hop liveness protocol: meta slot model, node and client heartbeat cycles, startup purge, chain reading, security notes | Current |
| `telemetry.md` | External monitoring side-channel: blob schema v1, participant states, topology reconstruction algorithm, staleness, security considerations | Current |
| `agent.md` | Agent/Worker runtime spec: concepts, 5-section TOML config (agent, log, telemetry, backends, workers), worker lifecycle and backoff, telemetry participation and blob schema, security notes | Current |
| `README.md` | Top-level system overview: what DropChannel is, components, ChannelProvider interface summary, protocol registry, security model summary, conformance, implementations, repo structure | Current |

## What does not exist yet

- `ring-protocol` and `piston-protocol` spec repos have not been created. Both prefixes
  are registered in `protocol-registry.md` but point to non-existent repositories.
- Heartbeat encryption via `DRIP_KEY` is deferred to a future revision.
- No spec yet for replay attack mitigation (sequence number in AAD).
- No spec yet for slot integrity or existence opacity mitigations.

## Pending updates to existing files

None currently known. Files will need updating when:

- The ChannelProvider interface gains new operations
- A new protocol is registered or an existing one gets a spec repo
- The heartbeat encryption (`DRIP_KEY`) is specced out
- The security model scope changes
- The Agent config schema changes (e.g. new top-level sections)

## How to start a new session here

For a full-context handoff to a new conversation, use `HANDOFF.md` — it contains a
complete summary of the entire system including all spec files and the agent runtime.

For targeted work on a specific file, paste `WORKING.md` and the relevant spec file(s).

Typical prompt:
> "Read WORKING.md and [relevant file(s)]. I need to update [file] to reflect the
> following change: [description]."
