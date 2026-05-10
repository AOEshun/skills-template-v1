---
name: ui-verify
description: Verify a UI-heavy issue's PR by driving Chrome MCP against a worktree's running dev server. Loads the issue's triager-authored `## UI verification` block, starts the dev server, runs a smoke check, walks the prose steps with a vision-equipped model, and returns pass/fail. Called by `parallel-issues` and `parallel-tdd` orchestrators in §8 when a PR's issue carries the `ui-heavy` label. Not user-invocable.
---

# UI Verify

Sub-skill called by `parallel-issues` / `parallel-tdd` orchestrators. Verifies that a UI-heavy issue's PR actually renders correctly before the orchestrator merges it.

The verifier is an orchestrator-side gate, not a subagent capability — Chrome MCP tools live in the orchestrator's harness, and §8 is sequential, so one Chrome session suffices.

## Inputs

The caller passes:

- `issue_number` — the issue this PR closes.
- `worktree_path` — `.worktrees/issue-<N>/` (parallel-issues) or `.worktrees/tdd-issue-<N>/` (parallel-tdd).
- `pr_head_sha` — used in reject-comment markers.
- `issue_body` — the issue body, including its `## UI verification` block.
- `acceptance_criteria` — the parsed AC list (used as the contract anchor).

## Returns

One of:

- `pass` — orchestrator continues to merge.
- `fail(findings)` — orchestrator bails with marker `<skill>-skill:verify-fail:<pr-head-sha>`. `findings` is a list of `{step_index, expected, observed, evidence}` entries the caller quotes verbatim in the reject comment.
- `error(reason)` — orchestrator bails with the same marker but cause line `UI verification could not run: <reason>` (e.g. dev server failed to come up). Distinct from `fail` so a human knows to investigate infra rather than the diff.

## Hard invariants

1. **Triager-authored only.** The contract is the issue's `## UI verification` block. The PR body's `## How to verify` is read for context only and never overrides it.
2. **One Chrome session per call.** Always open a fresh tab via `tabs_create_mcp`; close it on exit (success or failure). Never reuse tabs across calls.
3. **No mutating actions outside the dev server.** The verifier may navigate, click, type, and read. It does not log into external services, send emails, or call third-party APIs the dev server fronts.
4. **Dev server is killed on every exit path,** including `error`. Leaking a process across orchestrator iterations breaks the next call's port check.

## Pre-flight

Refuse with `error(reason)` if any fail:

- `CLAUDE.md` (or `AGENTS.md`, whichever the project uses) lacks a `### UI verification config` block. Reason: `Project missing ui-verify config; add a "### UI verification config" block to CLAUDE.md per ui-verify/VERIFICATION-FORMAT.md.`
- The configured port is already bound (`lsof -i :<port>`). Reason: `Port <port> already bound — kill the orphan process or change the port.`
- The issue body has no `## UI verification` block. Reason: `Issue #<N> is labeled ui-heavy but has no '## UI verification' section in the body.` (The orchestrator should have caught this in pre-flight, so reaching here is a contract violation — surface it loudly.)

## Workflow

See [VERIFICATION-FORMAT.md](VERIFICATION-FORMAT.md) for the `## UI verification` grammar and [STEP-EVAL.md](STEP-EVAL.md) for the per-step prompt the vision model receives.

1. **Parse config.** Read `### UI verification config` from CLAUDE.md/AGENTS.md. Extract `start`, `port`, `ready-path` (default `/`), `ready-timeout` (default `60`).
2. **Parse `## UI verification`.** Extract `Route:`, `Preconditions:`, and the numbered prose steps. Refuse with `error` if route is missing or steps list is empty.
3. **Start dev server.** From `worktree_path`, run the `start` command with `Bash` `run_in_background: true`. Poll `http://localhost:<port><ready-path>` every 2s until 200 or `ready-timeout` elapses.
4. **Open a fresh tab** at the route: `tabs_create_mcp` with `http://localhost:<port><Route>`.
5. **Smoke check (always).** After page load: `read_console_messages` (any `error`-level entry → smoke fail) and `read_network_requests` (any 4xx/5xx for same-origin requests → smoke fail). Document.
6. **Apply preconditions.** If preconditions cite seed/login routines that the project's CLAUDE.md declares (e.g. a `seed-user` script), run them. If preconditions cite a state the verifier cannot reach automatically, return `error` — triage was supposed to catch this.
7. **Walk the steps.** For each numbered step, in order:
   - Navigate / click / type as the step requires (use `find` + `javascript_tool` for interactions).
   - Take a `read_page` snapshot. If the step describes a visible outcome, also let the vision-capable orchestrator look at the page (it already has the screenshot via `read_page`).
   - Apply the per-step prompt in [STEP-EVAL.md](STEP-EVAL.md). Record `pass` / `fail` / `ambiguous`.
8. **Vibes check (always, after all steps).** Pass the final-state screenshot + the issue body to the orchestrator's vision model with the prompt in [STEP-EVAL.md](STEP-EVAL.md) → "vibes" section. Record any visible regression beyond what the steps captured.
9. **Tear down.** Close the tab via `tabs_close_mcp`. Kill the dev server PID. Confirm port is free.
10. **Decide.**
    - All steps `pass` and smoke + vibes both clean → return `pass`.
    - Any step `fail`, any `ambiguous` after one retry, smoke fail, or vibes fail → return `fail(findings)`.
    - Pre-flight or runtime failure that prevented evaluation → `error(reason)`.

## Files in this skill

- `SKILL.md` — this file (the contract called by orchestrators).
- `VERIFICATION-FORMAT.md` — `## UI verification` block grammar + the `### UI verification config` CLAUDE.md block grammar.
- `STEP-EVAL.md` — the prompt templates used per-step and for the vibes check.
