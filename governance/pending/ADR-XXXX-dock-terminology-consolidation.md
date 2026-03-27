# ADR-XXXX: Consolidate Storage Abstraction Terminology to "Dock"

**Status:** Draft  
**Deciders:** Paul Czajka  
**Date:** [to be set at commit time]  
**Governs:** `spec`, `dropchannel-py`

---

## Context

The DropChannel spec and implementation used several overlapping terms for storage-related concepts:

- **DockProvider** — the interface defining the 5-operation contract for storage access
- **Dock** — used informally to mean an instance of a DockProvider
- **Relay** — a network-accessible service exposing a Dock over HTTP
- **Server** — used interchangeably with Relay in implementation code and documentation

This created ambiguity. "DockProvider" implied the interface while "Dock" implied an instance, but they were frequently used interchangeably. "Relay" implied a forwarding participant in the pipeline, which conflates it with a Raft. "Server" carried no DropChannel-specific meaning at all.

---

## Decision

**"Dock" is the single term for both the interface and the conceptual unit.**

A Dock is any storage medium that satisfies the 5-operation primary interface. Concrete implementations are named by their backing store: `LocalDock`, `GcsDock`, `DropboxDock`, `HttpDock`, etc.

**"DockProvider" is retired** as an architectural term. It may appear as an internal implementation detail (e.g. a base class name) but carries no spec-level meaning.

**"Relay" and "Server" are retired** as DropChannel architectural terms. They are demoted to implementation-level language at most.

---

## Alternatives Considered

- **Keep "DockProvider" as the interface name, "Dock" as the instance term** — rejected because the distinction was not being maintained in practice and added cognitive overhead without meaningful benefit.
- **"DockProxy" instead of retiring "Relay"** — considered as a more precise replacement term for the HTTP server concept. Rejected because the server-side of an `HttpDock` is not a DropChannel architectural concept at all; naming it anything at the spec level overstates its importance. It belongs in `HttpDock` implementation documentation only.
- **Rename "Relay" to something else** — rejected for the same reason as above. The concept doesn't warrant a spec-level name.

---

## Reasoning

From the perspective of anyone configuring or operating a Channel, only Docks exist. An `HttpDock` is a Dock whose backing store happens to be remote and accessed over HTTP. What runs behind that HTTP endpoint is an implementation concern of the `HttpDock` — not a DropChannel protocol or spec concern. It is documented alongside the `HttpDock` implementation, not in the core spec.

"Relay" implies a forwarding participant in a pipeline chain. That role belongs to a Raft. A service backing an `HttpDock` does not forward anything — it exposes a storage resource. "Relay" is therefore a misleading term for it.

Collapsing "DockProvider" and "Dock" into a single term eliminates a distinction that was never being meaningfully maintained and removes a common source of reader confusion in the spec.

---

## Consequences

- `dock-provider.md` in the `spec` repo is renamed to `dock.md`.
- All spec references to "DockProvider" as a conceptual term are updated to "Dock."
- "Relay" and "Server" are removed from the spec vocabulary. Existing references in implementation code (e.g. `dropchannel-server` package in `dropchannel-py`) are not immediately renamed — that refactor is deferred until the `HttpDock` API contract stabilizes, at which point the server component is a candidate for extraction to a separate repository.
- The `dropchannel-channel` package in `dropchannel-py` is renamed to `dropchannel-dock`. The package contains Dock interface and implementation code only; the `-channel` name was overbroad and implied Channel-level concerns it does not own. Both `dropchannel-raft` and `dropchannel-endpoint` depend on this package.
- Concrete provider class names in `dropchannel-py` (e.g. `GCSProvider`) are implementation details and may be updated on their own schedule.
