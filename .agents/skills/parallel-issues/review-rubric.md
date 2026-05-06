# Review rubric

This is the single source of truth for the rubric the orchestrator applies to every subagent PR before merging. `SKILL.md` (the orchestrator) and `agent-prompt.md` (the subagent) both reference this file. Update only here; the other two files quote it.

The rubric is a **5-step gate**. Any one item failing rejects the PR. The issue is relabeled `ready-for-agent` → `needs-info`, comments are posted on both the issue and the PR with the cause, and a human resolves it.

## 1. Mechanical structure

The PR body must contain:

- The literal string `Fixes #<N>` (or `Closes #<N>`) for the issue.
- A `## Decisions made` heading. Body under the heading may say "none — followed acceptance criteria literally", but the heading must exist.
- An acceptance-criteria checklist with every item from the original issue marked `- [x]` (verified) or `- [ ]` (deferred).

An all-blank checklist (no `[x]` or `[ ]` marks) fails. A missing `## Decisions made` heading fails.

## 2. Scope

Every file path in the diff must be justified by the issue.

- If the issue has an `## Out of scope` section, any file matching it in the diff is an immediate reject.
- New ADRs, new top-level config, new shared modules are out-of-scope unless the issue explicitly directs them.
- Diffs that touch code another open `ready-for-agent` issue claims as its territory are an immediate reject.

## 3. Acceptance-criteria coverage

For each `- [x]` item in the PR's checklist, the reviewer must be able to point to a specific change in the diff that delivers it.

- A `[x] Tests added` mark with no test file in the diff fails.
- A `[x] Documented in CONTEXT.md` mark with no `CONTEXT.md` change in the diff fails.
- Vague claims without a corresponding diff hunk fail.

For each `- [ ]` deferred item, the PR's `## Decisions made` section must justify the deferral. Unjustified deferrals fail.

## 4. Convention match

The diff must match the project's documented conventions:

- Vocabulary aligns with `CONTEXT.md` (if present).
- File layout, naming, and idioms align with the existing codebase.
- Architectural decisions in `docs/adr/` (if present) are respected.

Obvious mismatches — wrong terminology, files in unconventional locations, patterns the project explicitly avoids — fail.

## 5. Smell check

The diff is rejected if any of the following are present:

- Dead code (functions, imports, files that nothing references).
- Debug prints, `console.log`, `print()` left in.
- Hardcoded secrets, tokens, or credentials.
- TODO comments without an issue reference.
- Suspiciously large diffs relative to the issue's scope (e.g. a 500-line diff for a one-line acceptance criterion).
- Generated files committed without explanation.

## Comment formats on reject

When the orchestrator rejects, it posts comments on both the **issue** and the **PR**, keyed by the PR's head sha so fixups don't permanently suppress.

**Issue comment:**

```
<!-- parallel-issues-skill:review-fail:<pr-head-sha> -->
Automated review failed — relabeled needs-info.
PR: #<M>
Branch: agent/issue-<N>-<slug>
Rubric reject: <one-line cause>
Detail on the PR.
```

**PR comment:**

```
<!-- parallel-issues-skill:review-fail:<pr-head-sha> -->
This PR was auto-reviewed and rejected. The associated issue has been relabeled `needs-info`.

**Failed rubric items:**
- **<rubric item name>**: <one-line reason>

**Findings:**
<path>:<line> — <one-line on what's wrong>
...
```

For merge failures (conflict, CI red, branch protection), the marker is `parallel-issues-skill:merge-fail:<sha>` and the cause line is one of:

- `Merge conflict with #<M> (already merged this batch). Rebase your branch on main and resolve.`
- `Required check '<workflow>' failed. See run: <URL>`
- `Branch protection requires <rule> — merge can't be automated. Human merge needed.`

For mechanical-structure failures (rubric item 1), the legacy marker `parallel-issues-skill:fail` is used for backward compatibility.

On approval + merge: **no comment is posted.** The squash-merge commit on `main` is the artifact; the PR is auto-closed by GitHub.
