# Versioning Conventions

This document defines the operational conventions for versioning artefacts across the `dropchannel/` org. It is the authoritative reference for format requirements and templates. The decisions behind these conventions are recorded in `docs/adr/ADR-XXXX-versioning-strategy.md`.

---

## Changelog Artefacts

Each protocol and implementation repo maintains two complementary artefacts.

### `CHANGELOG.md` (repo root)

A cumulative file, newest-first, following [Keep a Changelog](https://keepachangelog.com) format. This is the primary user-facing entry point. Each release has a summary entry with a link to the corresponding `history/` file for full detail.

Example entry:

```markdown
## [v1.2.0] — 2025-06-01

Added node heartbeat slot semantics. See [history/v1.2.0.md](history/v1.2.0.md) for full detail.

### Added
- Heartbeat slot write on idle cycles
- Configurable heartbeat interval

### Compatibility
Additive — backward-compatible with v1.1.x peers.
```

### `history/v<version>.md` files

Per-version detail records. For protocol repos these are normative: they describe the spec change in full. For implementation repos they describe what changed in the implementation at that version and reference any protocol versions adopted.

**Minimum required content — protocol repos:**

```markdown
## v<version> — YYYY-MM-DD

### Summary
One or two sentences.

### Changes
- ...

### Compatibility
Breaking / Additive / Clarification — and brief explanation.
```

**Minimum required content — implementation repos:**

```markdown
## v<version> — YYYY-MM-DD

### Summary
One or two sentences.

### Changes
- ...

### Protocol compatibility
See PROTOCOLS.md at this tag.
```

---

## Protocol Compatibility Manifest (`PROTOCOLS.md`)

Each implementation repo carries a `PROTOCOLS.md` file at the repository root. This is the canonical static declaration of which protocol version each handler targets.

**Format:**

```markdown
# Protocol Compatibility

| Protocol  | Handler Class    | Implements |
|-----------|------------------|------------|
| tideway   | TidewayHandler   | v1.0.0     |
| riverway  | RiverwayHandler  | v1.0.0     |
```

**Rules:**

- The table is updated as part of the same commit that introduces the implementation change, never after.
- The version in the table must correspond to a tagged release in the relevant protocol repo.
- If a handler is in-progress against an untagged draft spec, the version field reads `vX.Y.Z-dev` until the spec tag is created and the implementation is complete.

---

## Git Tag Format

All tags are annotated and use the `v`-prefixed version number:

```
git tag -a v1.2.0 -m "Short description of the change"
```

A version is not considered published until both its tag and its `CHANGELOG.md` entry exist.

---

## Runtime Protocol Version Logging

Each implementation logs its protocol version compliance at startup, before accepting or initiating channel activity. The log must identify each supported protocol by name and version.

Example (illustrative, not normative):

```
[dropchannel] tideway v1.2.0
[dropchannel] riverway v1.1.0
```

The mechanism — hardcoded constants, a generated module, or a bundled manifest — is an implementation concern. Accuracy is not optional; stale version strings are a silent correctness failure.

---

## ADR Index Entry Format

```
| ADR-0012 | 2025-03-01 | Short title describing the decision |
```

Four-digit zero-padded sequence number. Protocol or implementation version context belongs in the ADR body, not the index.
