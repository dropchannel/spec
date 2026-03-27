# ADR-XXXX: Org-Wide Versioning Strategy

**Status:** Accepted  
**Deciders:** Paul Czajka  
**Date:** [to be set at commit time]  
**Governs:** All repositories in the `dropchannel/` org

---

## Context

The DropChannel org contains three categories of repository, each with a distinct versioning concern:

- **Protocol repos** (`tideway-protocol/`, `riverway-protocol/`, and any future protocol repos) — define wire behaviour; versioned independently of each other and of any implementation
- **Implementation repos** (`dropchannel-py/`, and future language implementations) — package one or more protocol handlers; versioned on their own release cadence
- **Org-level spec** (`spec/`) — living conceptual documentation; not independently versioned

This ADR defines the versioning strategy that governs the org from the 1.0.0 release onwards. It also records the one-time remediation required to bring existing repositories to a 1.0.0-ready state; the detailed remediation checklist is maintained separately at `governance/pending/versioning-remediation-checklist.md`.

Operational conventions (changelog templates, `history/` file formats, `PROTOCOLS.md` format) are defined in `governance/pending/versioning.md`.

---

## Part 1: Steady-State Versioning (1.0.0 and Beyond)

This section describes how versioning works for all repositories at and after the 1.0.0 release. It is the authoritative reference for ongoing governance.

### 1.1 Version Numbering

All repositories — protocol and implementation alike — use **semantic versioning** (`MAJOR.MINOR.PATCH`) from 1.0.0 onwards.

For protocol repos:

| Increment | Meaning |
|-----------|---------|
| MAJOR | Breaking wire change — a peer implementing the previous version cannot interoperate |
| MINOR | Additive or behavioural change that is backward-compatible |
| PATCH | Clarification, correction, or editorial change with no wire impact |

For implementation repos:

| Increment | Meaning |
|-----------|---------|
| MAJOR | Breaking change to the public API or CLI surface, or adoption of a protocol MAJOR bump |
| MINOR | New handler, new feature, or adoption of a protocol MINOR bump |
| PATCH | Bug fix, dependency update, or adoption of a protocol PATCH |

Any new protocol or implementation repo created after the org-wide 1.0.0 release begins at `1.0.0` directly. There is no pre-1.0 sequential phase for new repos.

### 1.2 System-Level Compatibility Contract

DropChannel does not use an umbrella or distribution version. There is intentionally no single version number that describes "the system."

**Protocol version parity is the system-level compatibility contract.** Two implementations are compatible if and only if they implement the same protocol at the same version. A deployment's compatibility posture is fully described by the set of protocol versions its implementation supports — not by the implementation's own version number.

This means:

- `dropchannel-py` and `dropchannel-ru` are expected to carry different version numbers as they evolve independently. This is correct and not a problem.
- Interoperability questions are answered by comparing protocol versions, not implementation versions. "Are these two endpoints compatible?" is answered by "do both implement tideway at the same version?"
- The `PROTOCOLS.md` manifest in each implementation repo is the static declaration of this compatibility posture.
- At runtime, each implementation logs its protocol versions at startup, providing the operational expression of the same contract.

This approach is consistent with protocol-defined systems such as MQTT, Matrix, and ActivityPub, where independent implementations interoperate through protocol compliance rather than through shared versioning.

### 1.3 Cross-Implementation Coordination and the MAJOR Bump Problem

The compatibility contract in §1.2 correctly identifies incompatibility when it exists, but it does not prevent incompatibility from arising. This section names the coordination risk explicitly.

**The problem:** A protocol MAJOR bump — by definition a breaking wire change — requires every implementation to ship a matching update before any deployment can upgrade. If `dropchannel-py` adopts tideway v2.0.0 while `dropchannel-ru` remains at tideway v1.x, the tideway pipeline between them breaks. The protocol version parity contract correctly describes this as incompatible, but offers no mechanism to prevent or coordinate the transition.

In a single-owner project this is manageable as a social contract: the protocol maintainer coordinates the bump across implementations before tagging, and implementations release in lockstep. Under distributed or independent ownership — different maintainers, different release cadences, different language ecosystems — this coordination becomes structurally difficult and the window of cross-implementation incompatibility can be indefinite.

**Current position:** At the present scale of this project, with a single owner across all repos, the social contract is the accepted mechanism. Protocol MAJOR bumps are a significant event and are treated as requiring coordinated releases across all active implementations before the protocol tag is created.

**Known mitigation paths not yet adopted:**

- *Protocol compatibility ranges* — `PROTOCOLS.md` declares a supported range (e.g. tideway `>=v1.0.0 <v2.0.0`) rather than a single version. Implementations that support a range can interoperate with any peer within that range. This is the pattern used by most mature protocol ecosystems (HTTP, MQTT, TLS) and is the natural next step as the implementation count grows.

- *Version negotiation at channel establishment* — endpoints advertise supported protocol versions during handshake and negotiate down to the highest mutually-supported version. This eliminates the coordination problem entirely but constitutes a meaningful protocol-level design addition, not a versioning convention.

Both paths are deferred. The decision to adopt either, and the design of the mechanism, is left to a future ADR. That ADR should be considered a prerequisite before any implementation with an independent maintainer is admitted to the org.

### 1.4 Git Tags

Tags are created **spec-first** for protocol repos: a protocol version is tagged when its spec is finalised and merged, independent of whether any implementation has shipped it yet.

Tags are created **on release** for implementation repos.

All tags are annotated and use the `v`-prefixed version number:

```
git tag -a v1.2.0 -m "Add node heartbeat slot semantics"
```

The tag is the canonical VCS anchor for a version. A version is not considered published until both its tag and its `CHANGELOG.md` entry exist.

### 1.5 Changelog and History Artefacts

Each protocol and implementation repo maintains a `CHANGELOG.md` at root and per-version `history/v<version>.md` files. Format requirements and templates are defined in `governance/pending/versioning.md`.

### 1.6 Protocol Compatibility Manifest (`PROTOCOLS.md`)

Each implementation repo carries a `PROTOCOLS.md` file at the repository root declaring which protocol version each handler targets. Format and update rules are defined in `governance/pending/versioning.md`.

### 1.7 Org-Level `spec/` Repo

`spec/` contains living conceptual documentation. It is not independently versioned and carries no `CHANGELOG.md` or `history/` directory.

Where `spec/` content describes behaviour that is protocol-specific, it links to the relevant protocol repo rather than embedding a version claim inline. `spec/` is updated when conceptual understanding changes, not on a protocol release cadence.

### 1.8 ADR Index Structure

ADRs are indexed by sequential number and date. Format: `ADR-NNNN` with four-digit zero-padded sequence number.

Index entry format:

```
| ADR-0012 | 2025-03-01 | Adopt spec-first git tagging for protocol repos |
```

Protocol or implementation version context belongs in the ADR body, not the index.

### 1.9 Runtime Protocol Version Logging

Each implementation logs its protocol version compliance at startup. This is the operational expression of the compatibility contract defined in §1.2. At minimum, startup output must identify each supported protocol by name and version, sufficient for an operator to confirm compatibility between two endpoints without consulting source artefacts. The mechanism is an implementation concern; accuracy and timing (before channel activity begins) are not.

---

## Part 2: Pre-1.0.0 Existing Repos

This section applies exclusively to repositories that existed before the org-wide 1.0.0 release. It has no bearing on any repo created after 1.0.0.

### 2.1 Pre-1.0.0 State

Existing protocol and implementation repos used sequential versioning (`v0.1`, `v0.2`, ...) during the design and development phase. This was intentional: semver's increment semantics carry false precision during active design-phase work where the entire interface is subject to change.

During this phase: version history was recorded in `history/v*.md` files; no git tags were created; no `CHANGELOG.md` or `PROTOCOLS.md` existed; the ADR index used spec-version categories rather than sequential numbering.

None of the pre-1.0.0 version history will be retroactively tagged. The existing `history/v*.md` files are the permanent prose record of that phase and are left as-is.

### 2.2 Remediation

A detailed remediation checklist covering all repos is maintained at `governance/pending/versioning-remediation-checklist.md`. All items in that checklist must be completed as a prerequisite to the org-wide 1.0.0 release.

---

## Consequences

### What becomes easier

- "What does `dropchannel-py v1.2.0` implement?" is answerable in one file without reading changelogs.
- "Are these two endpoints compatible?" is answerable by comparing protocol versions — no umbrella version required.
- Protocol versions can be referenced stably across repos and external documentation using git tags.
- ADRs are findable by number and date without requiring knowledge of which spec version they relate to.
- A second implementation language can adopt the same conventions with no structural invention required.
- The `spec/` repo becomes genuinely living — no version references to maintain or drift.
- New protocol repos start at 1.0.0 with no pre-1.0 phase to navigate.

### What requires discipline

- `PROTOCOLS.md` must be kept in sync with implementation changes. No automated enforcement is specified here.
- Startup version strings must be kept accurate. Stale constants are a silent correctness failure.
- Protocol MAJOR bumps require coordinated releases across all active implementations. Under the current social contract model this is a process obligation on the protocol maintainer.
- Protocol tags must be created before implementation releases that reference them.

### What this does not address

- Automated enforcement of `PROTOCOLS.md` correctness (future CI concern).
- Protocol compatibility ranges or version negotiation — see §1.3 for the deferred mitigation paths.
- Deprecation policy for protocol versions (deferred to post-1.0.0).

---

## Alternatives Considered

**Umbrella / distribution version for the system as a whole**  
Rejected. DropChannel is a protocol-defined system: two endpoints are compatible if and only if they implement the same protocol versions. An umbrella version would add a coordination burden without adding information. Protocol version parity is the complete and sufficient compatibility contract.

**Independent 1.0.0 releases per repo**  
Rejected for existing repos. The system as a whole has its initial stable release at a single point in time; independent 1.0.0 declarations would misrepresent a coordinated release as a series of independent ones.

**Semver throughout, including pre-1.0.0**  
Rejected. Sequential versioning during the design phase signals intentional fluidity. Semver's increment semantics are only meaningful once an interface is stable enough to reason about breaking vs. additive changes.

**Retroactive git tagging of pre-1.0.0 versions**  
Rejected. The pre-1.0.0 history is adequately recorded in `history/v*.md` prose files. Retroactive tagging adds VCS complexity without meaningful benefit, since no external consumers exist for those versions.

**Single `CHANGELOG.md` with no `history/` files**  
Rejected for protocol repos. The per-version `history/` files serve as normative spec records, not just changelog summaries. Their content warrants a dedicated file per version.

**Keep ADR index categorised by spec version**  
Rejected. Spec-version categorisation couples the index to a version axis that spans multiple repos and becomes ambiguous as protocols diverge. Sequential date-ordered indexing decouples discoverability from version knowledge.

**Adopt compatibility ranges or version negotiation now**  
Deferred, not rejected. Both are sound mitigations for the MAJOR bump coordination problem identified in §1.3. They are deferred because the current single-owner scale does not warrant the added complexity, and because version negotiation is a protocol-level design decision that warrants its own ADR. See §1.3 for the trigger condition under which deferral should be revisited.
