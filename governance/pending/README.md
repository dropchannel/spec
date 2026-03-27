# governance/pending/

This directory holds documents that are **decided but not yet actionable** — content that has been worked through and is considered correct, but cannot be committed to its permanent home until a specific condition is met.

"Pending" in this project means waiting on a defined gate, not waiting on further design work. Documents here are not drafts. If a document is still being worked out, it doesn't belong here yet — it belongs in `governance/wip/`.

---

## What belongs here

- ADRs whose number will be assigned at commit time, awaiting a known gate condition
- Convention or reference documents that are correct but depend on a coordinated change not yet complete

---

## What does not belong here

- Documents still under active design — keep those in `governance/wip/` until decisions are settled
- Permanent reference material that is already actionable — commit it directly to its home
- Ephemeral working notes — these don't need to be tracked at all

---

## Promotion table

Each document here has a named condition under which it is promoted to its permanent location. When a document is promoted, remove its row from this table and delete the file from this directory. This directory should trend toward empty over time, not accumulate indefinitely.

| File | Permanent home | Promotion condition |
|------|----------------|---------------------|
| `ADR-XXXX-versioning-strategy.md` | `governance/adr/ADR-NNNN-versioning-strategy.md` | Org-wide 1.0.0 remediation complete; assign ADR number at commit time |
| `versioning.md` | `spec/conventions/versioning.md` | Org-wide 1.0.0 remediation complete |
| `versioning-remediation-checklist.md` | Close/archive (or convert to GitHub Issue) | All checklist items complete |

---

## On promotion

When the gate condition for a document is met, promote it to its permanent home, remove its row from the table above, and delete the file from this directory. If a gate condition changes, update the table — do not leave stale conditions in place.
