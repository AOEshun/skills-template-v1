# Review rubric (parallel-tdd delta)

This file extends `parallel-issues/review-rubric.md` with TDD-discipline checks. The orchestrator applies parallel-issues' rubric for items 2, 4, and 5 unchanged, and applies the extensions defined here for items 1 and 3.

Read `parallel-issues/review-rubric.md` first — it is the source of truth for items 2, 4, 5 and for the comment shapes on reject. This file documents only what changes for parallel-tdd.

The reject markers used by parallel-tdd are `parallel-tdd-skill:<phase>:<sha>` instead of `parallel-issues-skill:<phase>:<sha>`. Marker semantics are otherwise identical.

## Item 1. Mechanical structure (extension)

In addition to the parallel-issues item 1 requirements (`Fixes #N`, `## Decisions made` heading, populated AC checklist), the PR body must contain:

- A `## TDD log` heading.
- Under the heading, at least one `### Cycle N — <behavior>` entry.
- Each entry must have a `Red:` line citing a commit sha and a `Green:` line citing a commit sha. The `Refactor:` line is optional.

A missing `## TDD log` heading, an empty log, or a malformed cycle entry (no red sha or no green sha) fails item 1 with marker `parallel-tdd-skill:fail:<sha>`.

## Item 2. Scope (unchanged)

See `parallel-issues/review-rubric.md` item 2.

## Item 3. Acceptance-criteria coverage + TDD discipline (extension)

In addition to the parallel-issues item 3 requirements (each `[x]` AC has a corresponding diff hunk; deferrals justified):

- For each `[x]` AC item, the TDD log must contain at least one `### Cycle N — <behavior>` entry whose declared behavior maps to the AC. Mapping is by reading; the orchestrator does a sanity check, not a string match. A `[x]` AC with no corresponding cycle entry fails item 3 with cause `AC item "<text>" has no corresponding cycle in the ## TDD log.`
- For each cycle entry, the cited **red commit must be test-only**. The orchestrator verifies via `git show --stat <red-sha>` (run against the PR head's branch) and checks that every modified path satisfies the test-file definition below. If any non-test path appears in a red sha, the cycle is invalid → item 3 fails with cause `Cycle <N> red sha <sha> contains non-test files: <path1>, <path2>. Tests and implementation must be in separate commits.`
- At least one cycle in the PR must have a valid (test-only) red sha. If every red sha is invalid, the PR did horizontal slicing — item 3 fails with cause `No cycle has a test-only red sha — every cycle violates the test-first discipline.`

### Test-file definition

A path counts as a test file if **any** of the following hold:

- It contains a `test/`, `tests/`, `__tests__/`, or `spec/` directory segment (case-sensitive on Linux, case-insensitive in matching).
- The filename starts with `test_` (e.g. `test_user.py`, `test_checkout.rb`).
- The filename ends with `_test.go`, `_test.py`, or `_test.rb` (e.g. `user_test.go`).
- The filename ends with `.test.{js,ts,tsx,jsx}` or `.spec.{js,ts,tsx,jsx,rb}` (e.g. `user.test.tsx`, `cart.spec.rb`).

Project-specific overrides (e.g. an `e2e/` directory used for integration tests) are informational only — the orchestrator does not auto-extend the definition from `CLAUDE.md`. If your project's test layout doesn't match the patterns above, name the test files unambiguously (e.g. `tests/e2e_checkout.py` rather than `e2e/checkout.py`) so they match.

## Item 4. Convention match (unchanged)

See `parallel-issues/review-rubric.md` item 4.

## Item 5. Smell check (unchanged)

See `parallel-issues/review-rubric.md` item 5.

## Reject comment formats

Same shape as `parallel-issues/review-rubric.md`, with marker prefix `parallel-tdd-skill:` instead of `parallel-issues-skill:`:

- Mechanical-structure failure: `parallel-tdd-skill:fail:<sha>` (legacy `parallel-tdd-skill:fail` for the no-PR case).
- Review failure: `parallel-tdd-skill:review-fail:<sha>`.
- Merge failure: `parallel-tdd-skill:merge-fail:<sha>`.
- Parent slice landed: `parallel-tdd-skill:landed:<C>`.

The TDD-discipline reject reasons that fit under item 3:

- `Cycle <N> red sha <sha> contains non-test files: <path1>, <path2>. Tests and implementation must be in separate commits.`
- `AC item "<text>" has no corresponding cycle in the ## TDD log.`
- `## TDD log has no cycle entries.`
- `No cycle has a test-only red sha — every cycle violates the test-first discipline.`

These are quoted on the PR comment under "Failed rubric items: item 3".
