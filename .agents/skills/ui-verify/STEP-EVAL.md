# Step evaluation prompts

Single source of truth for the prompts the orchestrator uses when running `ui-verify`. Two prompts: one per numbered step, one for the final vibes check. Keep them here so `SKILL.md` and the reject-comment formatter both reference one shape.

## Per-step prompt

After performing the step's interactions and capturing a `read_page` snapshot, the orchestrator (acting as the vision-equipped verifier) applies this prompt against the page state:

```
You are verifying step <N> of a UI verification block authored by the issue's triager.

Step text: "<verbatim step>"

Issue body (context, not contract): <truncated issue body>

Acceptance criteria (anchor — the step should map to one of these): <AC list>

You have the current page's screenshot, DOM, console messages, and recent network requests in scope.

Decide one of:
- pass — the step's claim is visibly true on the page.
- fail — the step's claim is visibly false. State the discrepancy in one sentence: what was expected vs what is rendered.
- ambiguous — the page is in a transient state, the claim is ambiguous as written, or you can't tell. Name what's missing.

Quote at most one DOM excerpt or console line as evidence. Do not speculate beyond what's on the page.

Output strictly:
DECISION: <pass|fail|ambiguous>
EVIDENCE: <one DOM/console/network line, or `none`>
NOTE: <one sentence>
```

### Retry rule for `ambiguous`

If a step returns `ambiguous`, the orchestrator may retry **once**: wait 1s for any in-flight transitions, re-snapshot, re-prompt. A second `ambiguous` is treated as `fail` — the verifier cannot resolve the step.

### `fail` and `ambiguous` map to findings

Each non-`pass` step produces one `findings` entry the caller quotes verbatim in the reject comment:

```
{
  step_index: <N>,
  expected: "<verbatim step text>",
  observed: "<NOTE from the prompt output>",
  evidence: "<EVIDENCE from the prompt output, or `none`>"
}
```

## Vibes check prompt

After all steps complete and before tearing down, the orchestrator runs one final pass:

```
You are doing a final visual sanity check on the page after a UI-heavy issue's verification steps have run.

Issue title: <title>
Issue body (for context on what was supposed to change): <truncated issue body>

You have the current page's screenshot.

Look for visible regressions or breakage that the per-step checks would miss:
- Layout broken (overlapping elements, content cut off, scroll traps).
- Text overflow or truncation that hides meaning.
- Obvious z-index / clipping bugs.
- Anything visually wrong that a reasonable reviewer would flag.

Decide one of:
- clean — nothing visibly wrong beyond what's expected from the issue body.
- regression — name the visible regression in one sentence and quote where on the screen it occurs (top-left, header, etc.).

You are NOT grading the issue's intent — only "does this look like something a human reviewer would let through". Avoid speculation about features not visible.

Output strictly:
DECISION: <clean|regression>
NOTE: <one sentence, or empty for clean>
```

### Vibes `regression` maps to a finding

A `regression` outcome appends one synthetic finding to the list:

```
{
  step_index: -1,         // -1 indicates "vibes check, not a step"
  expected: "no visible regression",
  observed: "<NOTE>",
  evidence: "screenshot at end of run"
}
```

The reject comment in `parallel-issues/review-rubric.md` quotes it under the same `**Findings:**` block as numbered-step findings, with `Step -1 (vibes)` as the prefix.
