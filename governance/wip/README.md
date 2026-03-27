# governance/wip/

This directory holds documents in active investigative or research phase — recorded analysis, identified concerns, and assessed options where **no direction has yet been selected**.

"WIP" in this project means the thinking is ongoing. Documents here capture what has been learned and what the problem space looks like, but they do not represent settled decisions. They are working surfaces, not conclusions.

---

## What belongs here

- Security analysis and threat assessments being explored towards eventual remediation
- Candidate evaluations (storage backends, protocols, libraries) where no selection has been made
- Design explorations where tradeoffs are still being worked through
- Research notes that need to be preserved but aren't ready to drive a decision

---

## What does not belong here

- Documents where a direction has been chosen but action is deferred — those belong in `governance/pending/`
- Ephemeral scratch notes not worth preserving — those don't need to be tracked at all
- Finalised decisions — those belong in `governance/adr/`

---

## The distinction from `governance/pending/`

| | `wip/` | `pending/` |
|---|--------|------------|
| **Decision state** | No direction selected | Direction selected, not yet actionable |
| **Content** | Analysis, research, options | Settled documents awaiting a gate condition |
| **Next step** | More thinking, then a decision | Execute when the named gate condition is met |

A document graduates from `wip/` to `pending/` when a direction is chosen. It graduates from `pending/` to its permanent home when its promotion condition is met.

---

## On graduation

When a document in `wip/` reaches a decision point, move it to `governance/pending/` and add it to that directory's promotion table. Documents should not linger in `wip/` after a direction has been chosen — the distinction between "still thinking" and "decided but waiting" matters for anyone trying to understand the state of the project.
