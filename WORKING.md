# spec — Working Notes

## What this repo is

The system-level specification for the DropChannel runtime. Owns concerns that cut
across all protocols and all implementations: the ChannelProvider interface, encryption
standard, security model, and protocol dispatch rules.

## Current state

The following files exist and are current through v0.6 of the implementation:

| File | Contents | Status |
|------|----------|--------|
| `channel-provider.md` | Full ChannelProvider interface: 5 primary slot operations + 4 meta slot operations, contracts, provider implementations table, slot naming convention | Current |
| `encryption.md` | AES-256-GCM spec, wire format, key distribution, security properties, scope note re: heartbeat plaintext | Current |
| `security-model.md` | What is/isn't protected, trust boundaries, threat model scope | Current |
| `protocol-registry.md` | Registered protocols (winch, ring, piston), dispatch rules, protocol selection rationale, how to add a protocol | Current |

## What does not exist yet

- `README.md` — a top-level overview of the spec repo tying all files together and
  explaining the relationship between `spec`, `winch-protocol`, and the implementations.
  This should be generated next.
- `ring-protocol` and `piston-protocol` content is not represented here yet; only
  `protocol-registry.md` acknowledges their existence.

## Pending updates to existing files

None currently known. Files will need updating when:

- The ChannelProvider interface gains new operations
- A new protocol is registered
- The heartbeat encryption (`DRIP_KEY`) is specced out
- The security model scope changes

## How to start a new session here

Paste this file and the relevant spec file(s) into a new conversation. For interface
changes, also include the current `winch-protocol/README.md` for context on how the
interface is used.

Typical prompt:
> "Read WORKING.md and [relevant file(s)]. I need to update [file] to reflect the
> following change: [description]."

To generate the missing `README.md`:
> "Read WORKING.md and all four spec files. Generate a README.md for the spec repo
> that introduces the repo, explains what it owns vs. what lives in protocol repos,
> and links to each file with a brief description."
