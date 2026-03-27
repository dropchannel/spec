# governance/

This directory contains the org-level decision record for DropChannel — the paper trail for how the system got to its current state and where it is going next.

---

## Structure

| Directory | Purpose |
|-----------|---------|
| [`adr/`](./adr/) | Numbered, accepted or resolved decisions on a known roadmap |
| [`adr/pending/`](./adr/pending/) | Decided but not yet actionable — awaiting a named gate condition |
| [`adr/wip/`](./adr/wip/) | Investigative — no direction selected yet |
| [`conventions/`](./conventions/) | Cross-repo conventions and naming standards |
| [`standards/`](./standards/) | Formal standards and compliance requirements |

---

## Lifecycle

Documents move through the directories as decisions mature:

```
adr/wip/  →  adr/pending/  →  adr/
```

- A document enters `adr/wip/` when a question is worth tracking but no direction has been chosen.
- It moves to `adr/pending/` when a direction is chosen but the change cannot yet be executed.
- It moves to `adr/` when it is assigned a number and is on the active roadmap or complete.

Not every investigation becomes an ADR. A `wip/` document may be closed without promotion if the question resolves without requiring a formal decision.

---

## What warrants an ADR

One ADR per meaningful, non-obvious decision with lasting consequences — terminology, interfaces, protocols, security model, cross-repo conventions, package structure. Implementation details internal to a single repo do not need an ADR here.

When in doubt: if a future contributor would otherwise have to reverse-engineer why something is the way it is, write an ADR.
