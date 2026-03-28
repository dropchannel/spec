# Branch Naming Convention

## Format

```
{type}/{description}
```

`type` is one of the values from the table below. `description` is a brief, lowercase, kebab-case summary of the change. Length is left to judgment — prefer specificity over brevity where the two conflict.

## Types

| Type | Use for |
|------|---------|
| `adr` | Authoring or amending an ADR, or implementing changes mandated by one |
| `feat` | Adding new content or capability |
| `fix` | Correcting existing content |
| `refactor` | Restructuring without behavioral or semantic change |
| `docs` | Documentation updates (READMEs, conventions, standards) |
| `chore` | Maintenance that doesn't fit the above (template updates, renaming, cleanup) |

## Examples

```
adr/dock-list-operation
feat/add-ack-timeout-spec
fix/correct-write-guard-atomicity-note
refactor/runtime-directory-structure
docs/add-branching-convention
chore/update-adr-template
```

## Cross-repo ADR branches

An ADR in `dropchannel/` may mandate changes across multiple repos. In that case, a branch named `adr/{description}` should be created in each affected repo to carry the related work. Consistent naming across repos makes it easy to track which repos still have outstanding implementation work for a given decision — when all corresponding branches have merged, the ADR's implementation is complete.

## `main`

`main` is the protected trunk. All work should be done on a branch and merged via pull request. Minor non-functional changes — documentation corrections, typos — may be committed directly to `main` at the author's discretion. This laxity is expected to tighten as the project matures.

## Branch lifetime

Branches are working space. Delete them after merging.
