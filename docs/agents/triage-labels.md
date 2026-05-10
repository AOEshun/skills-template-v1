# Triage Labels

The skills speak in terms of five canonical triage roles. This file maps those roles to the actual label strings used in this repo's issue tracker.

| Label in mattpocock/skills | Label in our tracker | Meaning                                  |
| -------------------------- | -------------------- | ---------------------------------------- |
| `needs-triage`             | `needs-triage`       | Maintainer needs to evaluate this issue  |
| `needs-info`               | `needs-info`         | Waiting on reporter for more information |
| `ready-for-agent`          | `ready-for-agent`    | Fully specified, ready for an AFK agent  |
| `ready-for-human`          | `ready-for-human`    | Requires human implementation            |
| `wontfix`                  | `wontfix`            | Will not be actioned                     |

When a skill mentions a role (e.g. "apply the AFK-ready triage label"), use the corresponding label string from this table.

Edit the right-hand column to match whatever vocabulary you actually use.

## Complexity

The skills use three complexity tiers to route issues to appropriately-capable models. This table maps canonical complexity names to the actual label strings used in this repo's issue tracker and the model each tier dispatches to.

| Canonical name       | Label in our tracker  | Model dispatched |
| -------------------- | --------------------- | ---------------- |
| `complexity:simple`  | `complexity:simple`   | Haiku            |
| `complexity:medium`  | `complexity:medium`   | Sonnet           |
| `complexity:complex` | `complexity:complex`  | Opus             |

When a skill mentions a complexity tier (e.g. "this is a simple issue"), apply the corresponding label string from this table.

Edit the right-hand column to match whatever label vocabulary you actually use. The canonical names in the left-hand column must not be changed.
