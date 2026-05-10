# UI verification format

Two grammars defined here. The first is the per-issue `## UI verification` block written by the triager and pinned in the agent brief. The second is the per-repo `### UI verification config` block in `CLAUDE.md` (or `AGENTS.md`) that tells `ui-verify` how to start the project's dev server.

## Per-issue: `## UI verification` block

Goes in the agent brief comment posted on a `ui-heavy` issue. The triager authors it. The subagent reads it as additional contract; the verifier executes it.

### Grammar

```markdown
## UI verification

**Route:** <path on the dev server, starting with `/`>
**Preconditions:** <one-line description, or `none` if the fresh tab is enough>

1. <Observable behavior 1, in the issue's vocabulary.>
2. <Observable behavior 2.>
3. ...
```

### Rules

- **Route is required** and must be a path starting with `/`. The verifier will navigate to `http://localhost:<port><Route>`. Do not include the host or port — they come from the per-repo config.
- **Preconditions are required** but may be `none`. If preconditions are not automatable from a clean tab using only the project's documented seed/login helpers, do not apply `ui-heavy` — fall back to `ready-for-agent` / `ready-for-tdd-agent` without UI verification, or grill the reporter / project owner to add an automatable seed routine first.
- **At least one numbered step** is required. An empty step list fails pre-flight.
- **Steps are prose**, not selectors. Write what a human reviewer would see, in the issue's vocabulary. Examples:
  - Good: `The user's full name appears in the top-right header.`
  - Good: `Clicking the avatar opens a dropdown with "Settings" and "Sign out" entries.`
  - Bad: `[data-testid="user-name"] has innerText "Jane Doe".` — selector-coupled; brittle.
  - Bad: `It works.` — not observable.
- **Each step is one observable behavior.** Compound steps split into two. The verifier evaluates one step at a time; ambiguous joints make findings hard to localize.
- **Steps may include interactions** (click, type) as long as the visible outcome is named. `Clicking "Profile" navigates to /profile and the email field is editable.` is one step (one cause → one outcome), even though it involves a click.

### Worked example

```markdown
## UI verification

**Route:** /dashboard
**Preconditions:** Logged in as a user with at least one project. (See seed-user in CLAUDE.md.)

1. The user's full name appears in the top-right header.
2. Clicking the avatar opens a dropdown with "Settings" and "Sign out".
3. The dropdown has a new "Profile" link, between "Settings" and "Sign out".
4. Clicking "Profile" navigates to /profile and the email field is editable.
```

## Per-repo: `### UI verification config` block in CLAUDE.md / AGENTS.md

Tells `ui-verify` how to bring the project's dev server up. One block per repo. The skill greps for the heading exactly.

### Grammar

```markdown
### UI verification config

- start: <shell command>
- port: <integer>
- ready-path: <path to poll, default `/`>
- ready-timeout: <seconds, default `60`>
```

### Rules

- **start** is a single shell command. The verifier runs it from the worktree root with `run_in_background: true`. Examples: `npm run dev`, `pnpm dev`, `bin/server -p 3000`, `bun run dev`.
- **port** is the integer the dev server binds. The verifier `lsof`s this port pre-start; if already bound, refuse.
- **ready-path** is the route to poll for 200 OK. Defaults to `/`. Override if the root path requires auth and a healthcheck path is more reliable.
- **ready-timeout** caps how long the verifier waits for the server to come up. Defaults to 60s. Increase for slow boots; decrease to fail fast.

### Worked example

```markdown
### UI verification config

- start: npm run dev
- port: 3000
- ready-path: /
- ready-timeout: 60
```

### Optional helpers (referenced from `## UI verification` preconditions)

If `ui-heavy` issues commonly need seeded state, document the helpers here too — the triager can then cite them in `Preconditions:`. Convention only; the verifier does not auto-run helpers it has not been told about in step prose.

```markdown
**seed-user:** `npm run seed:user` — creates a user with one project and signs them in via cookie.
```

If seed routines mutate persistent state outside the worktree (e.g. a shared dev DB), they are not safe for parallel runs — but `ui-verify` is sequential per orchestrator §8, so single-orchestrator usage is fine.
