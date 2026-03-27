# Versioning Remediation Checklist

**Purpose:** One-time remediation required to bring all existing repositories to a 1.0.0-ready state, as mandated by `docs/adr/ADR-XXXX-versioning-strategy.md`.

**Gate:** All items below must be complete before any org-wide 1.0.0 release is tagged.

**Scope:** Existing repos only. Repos created after 1.0.0 are not subject to this checklist.

---

## Protocol repos (`tideway-protocol/`, `riverway-protocol/`)

- [ ] Create `CHANGELOG.md` at repo root; populate with a single `v1.0.0` entry summarising the protocol at initial stable release; link to existing `history/` files as the pre-1.0.0 record
- [ ] Ensure all existing `history/v*.md` files meet the minimum content spec defined in `docs/conventions/versioning.md`
- [ ] Create annotated git tag `v1.0.0`

## Implementation repo (`dropchannel-py/`)

- [ ] Create `CHANGELOG.md` at repo root; populate with a single `v1.0.0` entry; link to existing `history/` files as the pre-1.0.0 record
- [ ] Ensure all existing `history/v*.md` files meet the minimum content spec defined in `docs/conventions/versioning.md`
- [ ] Create `PROTOCOLS.md` declaring protocol compatibility at `v1.0.0`
- [ ] Implement startup protocol version logging per `docs/conventions/versioning.md`
- [ ] Create annotated git tag `v1.0.0`

## Org-level `spec/`

- [ ] Remove all inline "as of vX.Y" version qualifiers from markdown files
- [ ] Add protocol repo links where specific behaviour is described

## `.github/` ADR index

- [ ] Re-index `docs/adr/README.md` chronologically using `ADR-NNNN` sequential numbering
- [ ] Move any version context from index categories into individual ADR bodies
- [ ] Update any cross-references to ADR identifiers in other files

## Final gate

- [ ] Commit `docs/adr/ADR-XXXX-versioning-strategy.md` to its permanent location with assigned ADR number
- [ ] Commit `docs/conventions/versioning.md` to its permanent location
- [ ] Archive or close this checklist
