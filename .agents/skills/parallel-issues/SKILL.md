---
name: parallel-issues
description: Iterate through ready-for-agent issues in batches of up to 4 parallel Sonnet subagents. Each subagent works in its own git worktree and terminates in a PR with `Fixes #N`. After each batch, the orchestrator reviews each PR against `review-rubric.md`, squash-merges passing ones, relabels failures as `needs-info`, and continues to the next batch until no eligible issues remain. Use when the user says "/parallel-issues", asks to "iterate through agent issues", or wants autonomous dispatch + merge of agent-ready work.
---

# Parallel Issues

Autonomous orchestrator that loops through `ready-for-agent` issues. Each iteration: dispatch a batch of up to 4 parallel Sonnet subagents in isolated git worktrees, wait for all to terminate, review each resulting PR against the rubric, squash-merge passing PRs, relabel failures as `needs-info`, then re-evaluate eligibility and dispatch the next batch. Loop exits when no eligible issues remain, when a whole batch produces zero merges (no-progress fixed-point), or when an opt-in `--max-batches` cap is hit.

The rubric the orchestrator applies lives in `review-rubric.md` (shared with `agent-prompt.md` so subagent and reviewer never drift).

## Arguments

`/parallel-issues [issue-numbers...] [--plan] [--max-batches N] [--resume]`

- **No args**: scan all open `ready-for-agent` issues, loop through them in batches of up to 4 until exhausted or no-progress.
- **Issue numbers** (e.g. `/parallel-issues 3 4 7`): one-shot batch over exactly those issues. After the batch (review + merge + needs-info handling), exit without iterating. Refuse if any are not currently eligible.
- **`--plan`**: print the dependency graph + which issues would be dispatched in the *next* (single) batch + which are stalled. Spawns no agents. Does not loop.
- **`--max-batches N`**: opt-in cap. Exit after N batches even if more eligible issues remain. No-op when issue numbers are passed.
- **`--resume`**: skip the open-PR pre-flight check. Treat existing open `agent/issue-*` PRs as a phantom batch to review-and-merge first, then continue with normal dispatch. Use after an ESC interrupt to pick up where the loop left off.

## Vocabulary

This skill assumes the project follows the `to-issues` / `triage` agent-skills convention:

- Issues live on GitHub, managed via `gh` CLI.
- Triage labels in `docs/agents/triage-labels.md` — read this file at runtime to get the project's exact label strings. The canonical roles are `ready-for-agent`, `ready-for-human`, `needs-info`.
- Each issue body has `## Blocked by` (with `#N` references) and optionally `## Parent`, `## Acceptance criteria`, `## Out of scope` sections.

## Hard invariants

1. **No retry loops.** A single PR is reviewed once. Failures bail to `needs-info`.
2. **No orchestrator-fixes-it-itself.** The orchestrator never edits subagent code. If review or merge fails, the issue routes to a human.
3. **Interrupt-safe.** ESC at any moment must leave consistent on-disk + remote state. No destructive cleanup happens before its corresponding merge succeeds. Resume via `--resume`.
4. **Single source of truth for the rubric.** `review-rubric.md` is the only place rubric items are defined. `SKILL.md` and `agent-prompt.md` reference it.

## Workflow

### 1. Pre-flight (runs once at skill startup; skipped on iterations 2+)

Refuse and exit with a precise reason if any of these fail:

- `gh auth status` does not succeed.
- `git status --porcelain` is non-empty (working tree dirty). Tell the user to commit or stash.
- Current branch is not the repo's default branch. Detect via `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`.
- **Open-PR check (skipped if `--resume` is set):** any open PR matches branch prefix `agent/issue-` — i.e. a previous batch has unmerged work. Check via `gh pr list --state open --search "head:agent/issue-" --json number,title,headRefName`.
- **Branch-protection check:** `main` requires GitHub-side approving reviewers. Check via `gh api repos/:owner/:repo/branches/<default-branch>/protection 2>/dev/null | jq '.required_pull_request_reviews.required_approving_review_count // 0'`. If > 0, refuse — autonomous merging is incompatible with required-reviewer rules. (If the API returns 404, branch is unprotected; that's fine.)
- `docs/agents/triage-labels.md` is missing.

Soft / auto-fix:

- If `.worktrees/` is not in `.gitignore`, append it. Commit the `.gitignore` change with message `chore: ignore .worktrees/` before proceeding.
- Warn (do not refuse) if `CONTEXT.md` is missing. Subagent prompts will fall back to any top-level `*.md` files.

### 2. Resume handling (only when `--resume` is set)

Before entering the main loop:

- Scan for open `agent/issue-*` PRs: `gh pr list --state open --search "head:agent/issue-" --json number,title,headRefName,headRefOid`.
- Treat the result as a "phantom batch": route each PR through the **review-and-merge** pipeline (§7) exactly as if its subagent had just finished. Mergeable PRs squash-merge; failures get the standard reject treatment.
- After resume processing, fall through into the main loop normally.

Notes:
- Re-running review on a previously-rejected PR is idempotent: the rubric is read-only against the diff, and the SHA-keyed marker means a stale comment doesn't block a re-review.
- A successfully-merged PR from the resume phase counts toward the next iteration's "did we make progress" check.

### 3. Loop body

Each iteration runs steps 4–9. After step 9, decide:

- **Exit (exhausted):** the next §4–§5 evaluation produces zero eligible issues.
- **Exit (no-progress fixed-point):** the iteration that just ran produced zero merges (every dispatched issue review-failed, merge-failed, or verification-failed). Do not loop — the same eligible set with the same agents is unlikely to behave differently.
- **Exit (cap hit):** if `--max-batches N` was passed, exit after N iterations.
- **Exit (one-shot):** if the user passed explicit issue numbers, exit after the first batch regardless of remaining eligibility.
- **Otherwise:** continue to the next iteration.

Print the iteration header before dispatch:

```
=== Iteration <K> — dispatching <count>: #<n1> (<tier1>/<Model1>), #<n2> (<tier2>/<Model2>), ... ===
```

For unscored issues, use `unscored → Sonnet` in the header and print a warning line for each one immediately after the header:

```
⚠ Warning: #<N> has no complexity:* label — defaulting to Sonnet. Add a complexity label to make dispatch auditable.
```

### 4. Load issue graph

Fetch all open issues:

```
gh issue list --state open --json number,title,body,labels,state --limit 100
```

Also fetch closed issues (needed to evaluate blockers and parent-comment scan):

```
gh issue list --state closed --json number,title,labels --limit 200
```

For each open issue, parse:

- **Labels**: extract names. Identify `ready-for-agent`, `ready-for-human`. Also collect any `complexity:*` labels (e.g. `complexity:simple`, `complexity:medium`, `complexity:complex`).
- **Blocked by**: find the `## Blocked by` (case-insensitive) heading, take the next 5 lines, extract every `#N`.
- **Parent**: find `## Parent`, take the next 3 lines, extract `#N`. Parent references are NOT blockers.
- **Complexity tier**: from the collected `complexity:*` labels, determine the tier:
  - Exactly one `complexity:simple` → tier `simple`, model `"haiku"`.
  - Exactly one `complexity:medium` → tier `medium`, model `"sonnet"`.
  - Exactly one `complexity:complex` → tier `complex`, model `"opus"`.
  - No `complexity:*` label → tier `unscored`, model `"sonnet"` (default). Record for warning output.
  - **More than one `complexity:*` label → refuse, name the issue, and exit immediately. Do not dispatch anything.**

If any open `ready-for-agent` issue has a `## Blocked by` section that fails to parse cleanly, refuse and name the issue. Do not silently misclassify.

### 5. Compute eligibility

For each open `ready-for-agent` issue:

- For each blocker `#B`:
  - If `#B` is closed → cleared.
  - If `#B` is open and labeled `ready-for-human` → **stalled-on-human**. Record the issue as stalled with a pointer to `#B`.
  - If `#B` is open and labeled `ready-for-agent` → blocked-by-agent-work; record as deferred.
- If all blockers are cleared → eligible.

### 6. Apply argument filter and tie-break

- If user passed explicit issue numbers: keep only those. If any specified issue is not in the eligible set, refuse and explain why (which blocker, which label).
- If `--plan` is set: print the plan (eligible / deferred / stalled-on-human / chosen-N with tie-break reason) and exit. No worktrees, no agents. Each eligible issue is shown with its resolved tier and model in parentheses:
  - Scored issues: `#<N> (<tier> → <Model>)` — e.g. `#12 (complex → Opus)`.
  - Unscored issues: `#<N> (unscored → Sonnet, defaulted)` — so the maintainer can spot scoring gaps before dispatch.
- If more than 4 issues are eligible: sort by descending **transitive downstream count** (number of issues that have this issue, directly or via chain, in their `## Blocked by`), then by ascending issue number. Take the top 4.

On the **first** iteration, if more than 4 were eligible, pause and require user confirmation before continuing. On subsequent iterations no confirmation is needed (the user already opted into the loop).

### 7. Spawn subagents (this iteration's batch)

For each selected issue `#N`:

- **Slug**: lowercase the title, replace any non-alphanumeric run with `-`, trim to 40 chars, strip leading/trailing `-`.
- **Branch**: `agent/issue-<N>-<slug>`.
- **Worktree path**: `.worktrees/issue-<N>/` relative to repo root.
- Create the worktree from the default branch (now updated with prior iterations' merges):
  ```
  git worktree add -b agent/issue-<N>-<slug> .worktrees/issue-<N> <default-branch>
  ```
- Build the subagent prompt from `agent-prompt.md` by substituting the placeholders. Inline the full issue body fetched in §4.
- Resolve the model for this issue from the complexity tier determined in §4:
  - `complexity:simple` → `"haiku"`
  - `complexity:medium` → `"sonnet"`
  - `complexity:complex` → `"opus"`
  - unscored (no `complexity:*` label) → `"sonnet"` (default; a warning was already printed in the iteration header)
- Spawn the subagent via the `Agent` tool with:
  - `subagent_type: "general-purpose"`
  - `model`: the resolved model string for this issue (e.g. `"haiku"`, `"sonnet"`, or `"opus"`)
  - `run_in_background: true`
  - `description: "Implement issue #<N>"`
  - `prompt`: the rendered template

Spawn all selected subagents in a **single message** with multiple `Agent` tool calls so they run in parallel. Do not spawn them sequentially.

### 8. Wait, verify, review, merge

Wait for all background subagents in this batch to complete. Then process each issue `#N` in **ascending issue-number order**:

#### 8a. Existence check

`gh pr list --head agent/issue-<N>-<slug> --state open --json number,body,title,headRefOid`. If empty → verification fail (no PR opened). Bail per §8e with marker `parallel-issues-skill:fail` (legacy compatibility).

#### 8b. Mechanical verification (rubric item 1)

The PR body must contain:

- The literal string `Fixes #<N>` (or `Closes #<N>`).
- A `## Decisions made` heading.
- An acceptance-criteria checklist with every item from the original issue marked `- [x]` or `- [ ]` (all-blank → fail).

If any fail → bail per §8e with marker `parallel-issues-skill:fail`.

#### 8c. Diff review (rubric items 2–5)

Run the rubric in `review-rubric.md` against the PR's diff:

- Fetch the diff: `gh pr diff <M>`.
- Read the issue body (already in memory from §4).
- Read `CLAUDE.md`, `CONTEXT.md`, and `docs/adr/` if present.
- Apply rubric items 2 (scope), 3 (acceptance-criteria coverage), 4 (convention match), 5 (smell check).

If any fail → bail per §8e with marker `parallel-issues-skill:review-fail:<pr-head-sha>`. Comment must enumerate which rubric items failed and quote offending hunks.

#### 8d. Merge

Run `gh pr merge <M> --squash --delete-branch`.

If it fails:
- **Conflict** (gh reports merge conflict): bail per §8e with cause `Merge conflict with #<priorM> (already merged this batch). Rebase your branch on main and resolve.`
- **CI red** (gh reports required-check failure): cause `Required check '<workflow>' failed. See run: <URL>` (extract from `gh pr checks <M>`).
- **Other gh errors**: cause `Branch protection requires <rule> — merge can't be automated. Human merge needed.` or the gh-reported reason.

All merge failures use marker `parallel-issues-skill:merge-fail:<pr-head-sha>`.

On success:
- Capture the squash-commit sha from `gh pr view <M> --json mergeCommit --jq .mergeCommit.oid` for the iteration report.
- Remove worktree: `git worktree remove --force .worktrees/issue-<N>`.
- Delete local branch ref: `git branch -D agent/issue-<N>-<slug>` (remote already deleted by `--delete-branch`).
- No comment posted on issue or PR. The squash-merge commit is the artifact; PR is auto-closed by GitHub.

#### 8e. Bail procedure (used by 8a, 8b, 8c, 8d on failure)

For the failed issue `#N`:

1. `gh issue edit <N> --remove-label ready-for-agent --add-label needs-info`.
2. **Issue comment** (if not already present, keyed by sha + marker):
   ```
   <!-- parallel-issues-skill:<marker>:<pr-head-sha> -->
   Automated <verification|review|merge> failed — relabeled needs-info.
   PR: #<M>           (omit if no PR was opened)
   Branch: agent/issue-<N>-<slug>
   Tier: <tier> (<model>)
   <Cause line — see review-rubric.md for review-fail / merge-fail formats>
   Detail on the PR. (omit if no PR was opened)
   ```
   where `<tier>` is `simple`, `medium`, `complex`, or `unscored`, and `<model>` is the model that was dispatched (or would have been dispatched) for this issue. The `complexity:*` label on the issue is **not** removed and **not** auto-bumped during bail; only the `ready-for-agent` → `needs-info` label transition (step 1 above) is applied.
3. **PR comment** (only if PR was opened, keyed by sha + marker):
   ```
   <!-- parallel-issues-skill:<marker>:<pr-head-sha> -->
   This PR was auto-<verification|review|merge>-checked and rejected. The associated issue has been relabeled `needs-info`.

   **Failed items:**
   - **<rubric item or cause>**: <one-line reason>

   **Findings:** (review-fail only — quote diff hunks)
   <path>:<line> — <reason>
   ```
4. **Branch + worktree handling:**
   - If commits exist: leave branch in place; remove worktree only (`git worktree remove --force .worktrees/issue-<N>`).
   - If no commits: remove worktree and delete branch (`git worktree remove --force .worktrees/issue-<N>`; `git branch -D agent/issue-<N>-<slug>`).
5. **Continue with the next issue in the batch.** A single failure does not abort the iteration.

#### 8f. Idempotency notes

- All comment markers carry the PR's head sha. If a human pushes a fixup, the sha changes, and a future `--resume` (or future `ready-for-agent` retriage) won't see a stale marker.
- Before posting a comment, check existing comments on the issue/PR for the exact marker string. Skip if present.

### 9. Iteration report

Print after each iteration:

```
Iteration <K> report — <S> of <T> merged, <F> failed

Merged this batch (in order):
  ✓ #<N1> → PR #<M1> squash-merged onto main (sha <short-sha>)
  ✓ #<N2> → PR #<M2> squash-merged onto main (sha <short-sha>)

Failed:
  ✗ #<N3> → PR #<M3> review-fail. Reason: <one-line>
  ✗ #<N4> → no PR opened. Reason: subagent bailed on out-of-scope expansion

Stalled on human:
  #<X> blocked by #<Y> (ready-for-human)

Deferred:
  #<Z> (still blocked by #<X>)
```

If all 4 dispatched issues failed → loop exits via no-progress fixed-point after this report.

### 10. Between-iteration cleanup

After the iteration report, before the next iteration's §4 fetch:

1. **Pull main:**
   ```
   git checkout <default-branch>
   git pull origin <default-branch>
   ```
   This refreshes the orchestrator's working tree with the squash-commits just merged.
2. **Prune dead worktrees:** `git worktree prune` and remove any `.worktrees/issue-*` whose branch no longer has an open PR (i.e. PR was merged or branch was deleted out-of-band).
3. **Skip pre-flight (§1) re-runs.** Loop invariants stay valid; no need to re-check `gh auth`, working-tree dirty, or open-PR-prefix (the latter is now expected to be non-empty if any failures preserved their PRs).

### 11. Parent-comment scan (runs once per iteration, after step 9)

For every open issue with a `## Parent` section:

- Parent issue = `#P`.
- Find all closed children of `#P` (children = open *or* closed issues that reference `#P` in their `## Parent`). Practically: scan the closed-issues list from §4 for those whose body parents are `#P`.
- For each closed child `#C`:
  - Fetch existing comments on `#P`: `gh issue view P --json comments --jq '.comments[].body'`.
  - Skip if any existing comment contains the marker `<!-- parallel-issues-skill:landed:C -->`.
  - Otherwise post:
    ```
    <!-- parallel-issues-skill:landed:C -->
    Slice landed: closes #C. <child title>. <X> child slices remaining.
    ```
    where `X` = count of still-open children of `#P`.

This makes the comment posting idempotent across runs without persistent state.

### 12. Final session report (printed once when the loop exits)

```
Loop exited: <reason>
  reason ∈ {exhausted | no-progress | max-batches=<N> | one-shot | user-interrupt}

Iterations: <K> batches
Merged: <total-merged> PRs landed on main
Needs-info this session: <count>
  #<N> — "<title>" — <one-line cause>

Stalled (still blocked on humans):
  #<X> blocked by #<Y>

Open agent PRs preserved for human review:
  PR #<M> (issue #<N>, branch agent/issue-<N>-<slug>)

Merged PRs this session (with squash-commit shas):
  PR #<M1> for #<N1> — "<title>" — sha <short-sha>
  PR #<M2> for #<N2> — "<title>" — sha <short-sha>
  ...
```

If no eligible issues at startup: report `Nothing to dispatch — all ready-for-agent issues are blocked or completed.` and list any stalled-on-human / deferred entries.

## Concurrency

Two simultaneous invocations of this skill in the same repo would clash on worktree directories and the open-PR check. The pre-flight catches this on the second invocation: the first invocation will have created `agent/issue-*` branches that show up as open PRs (or as worktrees on disk). Refuse cleanly. No mutex needed.

`--resume` is the documented escape hatch for the legitimate case (single user resuming after ESC). It's NOT a concurrency primitive — running two `--resume` invocations in parallel is not supported.

## Files in this skill

- `SKILL.md` — this file (orchestrator workflow).
- `agent-prompt.md` — the prompt template each subagent receives. Placeholders: `{{ISSUE_NUMBER}}`, `{{ISSUE_TITLE}}`, `{{ISSUE_BODY}}`, `{{BRANCH_NAME}}`, `{{WORKTREE_PATH}}`, `{{DEFAULT_BRANCH}}`, `{{ACCEPTANCE_CRITERIA_CHECKLIST}}`. Includes a "How your PR will be reviewed" section sourced from `review-rubric.md`.
- `review-rubric.md` — single source of truth for the 5-step review rubric and the comment formats. Both `SKILL.md` and `agent-prompt.md` reference it.
