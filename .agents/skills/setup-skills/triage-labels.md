# Triage Labels

The skills speak in terms of six canonical triage roles. This file maps those roles to the actual label strings used in this repo's issue tracker.

| Label in mattpocock/skills | Label in our tracker    | Meaning                                                        |
| -------------------------- | ----------------------- | -------------------------------------------------------------- |
| `needs-triage`             | `needs-triage`          | Maintainer needs to evaluate this issue                        |
| `needs-info`               | `needs-info`            | Waiting on reporter for more information                       |
| `ready-for-agent`          | `ready-for-agent`       | Fully specified, ready for an AFK agent                        |
| `ready-for-tdd-agent`      | `ready-for-tdd-agent`   | Fully specified, ready for a TDD-discipline agent (red→green)  |
| `ready-for-human`          | `ready-for-human`       | Requires human implementation                                  |
| `wontfix`                  | `wontfix`               | Will not be actioned                                           |

When a skill mentions a role (e.g. "apply the AFK-ready triage label"), use the corresponding label string from this table.

`ready-for-agent` and `ready-for-tdd-agent` are mutually exclusive eligibility labels. Apply `ready-for-tdd-agent` when the issue has clear behavioral acceptance criteria that should be implemented test-first — the `parallel-tdd` skill dispatches this pool. Apply `ready-for-agent` for everything else (refactors, chores, config, one-line fixes) — the `parallel-issues` skill dispatches that pool.

Edit the right-hand column to match whatever vocabulary you actually use.

## UI verification

`ui-heavy` is an additive label, orthogonal to `ready-for-agent` / `ready-for-tdd-agent`. The `parallel-issues` and `parallel-tdd` orchestrators call the `ui-verify` skill as a pre-merge gate when this label is present.

| Canonical name | Label in our tracker | Meaning                                                     |
| -------------- | -------------------- | ----------------------------------------------------------- |
| `ui-heavy`     | `ui-heavy`           | PR must pass `ui-verify` (Chrome MCP–driven) before merging |

Edit the right-hand column to match whatever vocabulary you actually use. Removing this row entirely is fine — the orchestrators only consult the label when it's present.

`ui-heavy` issues require a `## UI verification` block in their agent brief and a `### UI verification config` block in `CLAUDE.md` / `AGENTS.md`. See `.agents/skills/ui-verify/VERIFICATION-FORMAT.md`.

## Complexity

The skills use three complexity tiers to route issues to appropriately-capable models. This table maps canonical complexity names to the actual label strings used in this repo's issue tracker and the model each tier dispatches to.

| Canonical name       | Label in our tracker  | Model dispatched |
| -------------------- | --------------------- | ---------------- |
| `complexity:simple`  | `complexity:simple`   | Haiku            |
| `complexity:medium`  | `complexity:medium`   | Sonnet           |
| `complexity:complex` | `complexity:complex`  | Opus             |

When a skill mentions a complexity tier (e.g. "this is a simple issue"), apply the corresponding label string from this table.

Edit the right-hand column to match whatever label vocabulary you actually use. The canonical names in the left-hand column must not be changed.
