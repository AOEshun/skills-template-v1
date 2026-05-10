---
name: parallel-tdd
description: Iterate through `ready-for-tdd-agent` issues in batches of up to 4 parallel subagents, each applying the test-driven development discipline from `.agents/skills/tdd/SKILL.md` (red→green→refactor, vertical slices, no horizontal "all tests first") in its own git worktree. Subagent model is selected by `complexity:*` label (haiku/sonnet/opus). Each PR is reviewed for both standard scope/structure and TDD discipline (`## TDD log` with one cycle per acceptance criterion, red commits test-only) before squash-merging. Use when the user says "/parallel-tdd", asks to iterate through TDD-flagged issues, or wants autonomous test-first dispatch.
---

# Parallel TDD

Sibling skill of `parallel-issues`. Same orchestrator shape — eligibility, batching by complexity, parallel dispatch, per-iteration review/merge — but each subagent applies the **test-driven development** discipline from `.agents/skills/tdd/SKILL.md`. The review rubric is extended to verify TDD discipline post-hoc via a structured `## TDD log` in the PR body.

## Relationship to parallel-issues

This skill is structurally a delta on `parallel-issues`. The orchestrator inherits the following from `parallel-issues/SKILL.md` unchanged:

- Pre-flight checks (§1), resume handling (§2), iteration loop (§3), exit conditions, between-iteration cleanup (§10), parent-comment scan (§11), final session report (§12).
- Worktree creation, branch naming mechanics, complexity-tier model dispatch (§7).
- Existence check (§8a), merge mechanics (§8d), bail procedure (§8e), idempotency notes (§8f).
- Concurrency rules.

What this skill changes:

1. **Label.** Reads `ready-for-tdd-agent` instead of `ready-for-agent` (§4, §5, §8e).
2. **Eligibility extension.** A `ready-for-tdd-agent` issue with zero parseable acceptance-criteria items is refused at §6 — a TDD agent with no behaviors to test is misclassified. Bail to `needs-info` immediately, do not dispatch.
3. **Subagent prompt.** `parallel-tdd/agent-prompt.md` (extends `parallel-issues/agent-prompt.md` with TDD-specific sections).
4. **Review rubric.** `parallel-tdd/review-rubric.md` (delta on `parallel-issues/review-rubric.md` extending items 1 and 3).
5. **Branch prefix.** `tdd/issue-<N>-<slug>` instead of `agent/issue-<N>-<slug>`. Pre-flight open-PR check (§1) looks for `head:tdd/issue-` instead of `head:agent/issue-`. Isolates the two skills' open-PR pools.
6. **Worktree path.** `.worktrees/tdd-issue-<N>/` instead of `.worktrees/issue-<N>/`. Same isolation rationale.
7. **Comment marker prefix.** `parallel-tdd-skill:` instead of `parallel-issues-skill:` so the two skills' issue/PR comments are distinguishable.

## Arguments

`/parallel-tdd [issue-numbers...] [--plan] [--max-batches N] [--resume]`

Same semantics as `parallel-issues`. See the "Arguments" section of `parallel-issues/SKILL.md`.

## Vocabulary

- `ready-for-tdd-agent` — eligibility label. Must be defined in `docs/agents/triage-labels.md`. Pre-flight refuses if the label is missing from that file.
- `complexity:simple|medium|complex` — same dispatch table as parallel-issues: `haiku` / `sonnet` / `opus`. Unscored issues default to `sonnet` with a warning.

## Hard invariants

In addition to the four invariants in `parallel-issues/SKILL.md`:

5. **TDD discipline is enforced post-hoc, not at runtime.** The orchestrator does not execute tests during review. It checks for the `## TDD log` structure and verifies that each cycle's red sha is test-only via `git show --stat`. Test execution is out of scope for v1.
6. **One cycle per acceptance criterion, minimum.** Each `[x]` AC item must correspond to at least one `### Cycle N` entry in the TDD log.

## Workflow deltas

### §1 Pre-flight (delta)

In addition to parallel-issues' pre-flight, also refuse if:

- `docs/agents/triage-labels.md` does not contain `ready-for-tdd-agent`. Tell the user to add it (e.g. via the `triage` or `setup-skills` skill).
- Open-PR check (skipped on `--resume`): any open PR matches branch prefix `tdd/issue-`. Check via `gh pr list --state open --search "head:tdd/issue-" --json number,title,headRefName`.

### §4 Load issue graph (delta)

Filter to `ready-for-tdd-agent` instead of `ready-for-agent`. Eligibility logic (§5) is otherwise identical — `## Blocked by` references can point to issues from either pool.

### §6 Apply argument filter and tie-break (delta)

After computing the eligible set, additionally check each chosen issue:

- Parse `## Acceptance criteria` from the issue body. Count the bullet items. If the section is missing or the count is zero, the issue is **refused**. Bail to `needs-info` per §8e with cause:
  ```
  Issue has no parseable acceptance criteria — TDD requires behaviors to test. Add a "## Acceptance criteria" section with at least one bullet item.
  ```
  Do not dispatch.

For `--plan` output, each chosen issue's line shows the parsed AC count: `#<N> (<tier> → <Model>, <K> behaviors)`. Issues with `K == 0` appear in a separate "Refused (no behaviors to test)" section instead of the chosen list. `--plan` does not actually relabel; it just reports what would happen.

### §7 Spawn subagents (delta)

- Branch: `tdd/issue-<N>-<slug>`.
- Worktree: `.worktrees/tdd-issue-<N>/`.
- Prompt: rendered from `parallel-tdd/agent-prompt.md`.
- Model dispatch: identical to parallel-issues §7 (resolved from `complexity:*` label).

Spawn all selected subagents in a **single message** with multiple `Agent` tool calls so they run in parallel.

### §8b Mechanical verification (extension)

In addition to parallel-issues' §8b checks, the PR body must also contain a `## TDD log` heading with at least one `### Cycle N` entry. A missing `## TDD log` heading or an empty log fails this check with marker `parallel-tdd-skill:fail:<sha>`.

### §8c Diff review (delta)

Use `parallel-tdd/review-rubric.md`. The orchestrator reads `parallel-issues/review-rubric.md` for items 2/4/5 and `parallel-tdd/review-rubric.md` for the deltas to items 1/3.

For each `### Cycle N` entry in the TDD log:

1. Extract the cited red sha.
2. Run `git show --stat <red-sha>` against the PR head's branch (the worktree or fetched branch).
3. Verify every modified path matches the test-file definition in `parallel-tdd/review-rubric.md`.
4. If any non-test path appears in a red sha, the cycle is invalid → item 3 fails.

At least one cycle in the PR must have a valid (test-only) red sha. If every red sha is invalid, the PR did horizontal slicing → item 3 fails with cause `No cycle has a test-only red sha — every cycle violates the test-first discipline.`

### §8a, §8d, §8e (delta)

Identical to parallel-issues, except all comment markers use prefix `parallel-tdd-skill:` instead of `parallel-issues-skill:`.

### §11 Parent-comment scan (delta)

Identical to parallel-issues, except the marker is `parallel-tdd-skill:landed:<C>` instead of `parallel-issues-skill:landed:<C>`.

## Files in this skill

- `SKILL.md` — this file (orchestrator deltas).
- `agent-prompt.md` — subagent prompt template. Inherits scope-discipline + PR-shape rules from `parallel-issues/agent-prompt.md`; adds TDD methodology references (`.agents/skills/tdd/`), autonomous adaptations, and the `## TDD log` requirement.
- `review-rubric.md` — review-rubric deltas for TDD discipline. Inherits items 2/4/5 from `parallel-issues/review-rubric.md`; extends items 1 and 3.
