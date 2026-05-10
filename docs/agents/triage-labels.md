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

## TDD Eligibility Label

`/parallel-tdd` filters on a separate, additive eligibility label. An issue can carry both `ready-for-agent` and `ready-for-tdd-agent`; the latter signals that the issue is suitable for test-driven development (red→green→refactor on parseable acceptance criteria).

| Canonical label       | Label in our tracker  | Meaning                                                          |
| --------------------- | --------------------- | ---------------------------------------------------------------- |
| `ready-for-tdd-agent` | `ready-for-tdd-agent` | Fully specified AND suitable for TDD discipline (logic-heavy AC) |

Apply to issues whose acceptance criteria are testable behaviors (state machines, detectors, sizing/math, persistence). Skip when the AC is dominated by visualization/UX work where unit-test discipline maps poorly.

## Complexity Labels

Some skills (e.g. `/parallel-issues`, `/parallel-tdd`) route issues to different model tiers based on complexity. The mapping:

| Canonical label    | Label in our tracker | Model tier |
| ------------------ | -------------------- | ---------- |
| `complexity:simple`  | `complexity:simple`  | Haiku      |
| `complexity:medium`  | `complexity:medium`  | Sonnet     |
| `complexity:complex` | `complexity:complex` | Opus       |

When a skill needs to pick a model tier from an issue's complexity label, use the corresponding label string from this table.
