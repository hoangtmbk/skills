# Usage — playwright-qa

How to author Stories, run them, and read the results. Already installed? Good. Need to install? See [`INSTALL.md`](./INSTALL.md).

## What you invoke

You only ever type three things at the prompt:

| Command | What it does |
| --- | --- |
| `/browser-qa` | Run all Stories in `ai_review/user_stories/*.yaml` in parallel. |
| `/browser-qa <filter> [flags]` | Run a subset, with mode flags. |
| `/qa-clean` | Prune `ai_review/runs/`. |

The `bowser-qa-agent` subagent and the `playwright-qa` skill are internal — `/browser-qa` spawns one subagent per Story; each subagent calls the skill for browser control. You never invoke them directly.

## Authoring a Story

### Schema

Every YAML file under `ai_review/user_stories/` declares one top-level key — `stories:` — whose value is a list. Each list item has three fields:

```yaml
stories:
  - name: "Front page loads with posts"        # free-form string; used in the report
    url: "https://news.ycombinator.com/"        # starting URL the workflow operates against
    workflow: |                                  # free-text body; see "Workflow prose" below
      Navigate to https://news.ycombinator.com/.
      Verify the front page loads.
      Verify at least 10 posts are visible.
```

A single file MAY hold multiple Stories — useful for grouping by feature area:

```yaml
stories:
  - name: "Element not found"
    url: "https://example.com/"
    workflow: |
      Navigate to https://example.com/.
      Click the "Buy Now" button.
      Verify a checkout overlay appears.
  - name: "Wrong heading assertion"
    url: "https://example.com/"
    workflow: |
      Navigate to https://example.com/.
      Verify the page heading reads "Hacker News".
```

There is no separate `steps:`, `expected:`, `assertions:`, or `artifacts:` key. Everything the subagent needs is in the prose of `workflow:`.

### Workflow prose

The subagent decomposes the workflow into ordered Steps. Each Step is one discrete action OR one verification. Any free-text shape works:

- Imperative prose (recommended):
  ```
  Navigate to /login.
  Fill the email field with alice@example.com.
  Click Sign In.
  Verify the URL becomes /dashboard.
  ```
- Given/When/Then BDD — same idea, different phrasing.
- Numbered list — same idea, indented under `|`.

Two phrasings trigger special behavior:

| Phrase in workflow prose | Effect |
| --- | --- |
| `${VAR_NAME}` | Resolved through the shell at execute time; secrets never enter the LLM context. See "Secrets" below. |
| `visually verify` (case-insensitive) | Switches that Step to vision verify — the subagent ingests the screenshot pixels to judge the expectation. |

Everything else is plain prose. Verifications are just sentences that start with "Verify" / "Confirm" / "Check" — the subagent infers them.

### Naming

Story `name:` is free-form. The subagent slugifies it (lowercase, kebab-case, alphanumeric + hyphens) for filenames and the Run report:

```
"Front page loads with posts"  →  front-page-loads-with-posts
```

File names can be anything matching `*.yaml`. Group by feature, surface area, or owner. The filename substring is what the orchestrator filters on (see "Running" below).

## Running

From the repo root with Stories in `ai_review/user_stories/`:

```
/browser-qa                       # every Story, headless, text verify
/browser-qa smoke                 # only files whose name contains "smoke"
/browser-qa headed checkout       # visible browser, files matching "checkout"
/browser-qa vision                # vision verify on every Step (every Story)
/browser-qa save-test             # persist trace.zip + starter tests/<slug>.spec.ts
/browser-qa headed vision login   # all three: visible, vision, files matching "login"
```

Flags are case-insensitive and order-independent. Any token that isn't a known flag becomes the filename filter (last one wins if you pass multiple).

The filter matches **filenames**, not Story names. To run a single Story out of a multi-Story file, put it in its own file or pre-filter by editing the YAML.

### What happens under the hood

1. Orchestrator globs `ai_review/user_stories/*.yaml` and applies the filter.
2. **Env-var pre-flight** — every `${VAR}` reference is checked before any browser opens. Stories with unset vars are marked "pre-flight FAIL" and never spawn a subagent.
3. One subagent per Story spawns **in parallel** — wall-clock ≈ slowest Story, not the sum.
4. Each subagent opens a named browser Session, walks its Steps, writes artifacts, closes the Session.
5. Orchestrator collects results, runs `playwright-cli close-all` defensively, prints the aggregate report.

Parallelism is the point. Don't iterate Stories sequentially in your head — let the orchestrator fan out.

## Reading a Run

Every Run writes to `ai_review/runs/<timestamp>_<uuid8>/`:

```
ai_review/runs/20260524T101530Z_a1b2c3d4/
├── front-page-loads-with-posts/         ← one dir per Story (slug of Story name)
│   ├── 00_navigate-to-news-yc.png        ← per-Step screenshot, zero-padded index
│   ├── 01_verify-front-page-loads.png
│   └── 02_verify-at-least-10-posts.png
└── checkout-flow-completes/              ← failed Story has extra artifacts
    ├── 00_navigate-to-cart.png
    ├── 01_add-item-to-cart.png
    ├── 02_click-pay-button.png           ← failing Step's screenshot
    ├── console.log                       ← JS errors at failure time
    └── requests.log                      ← network traffic, for 5xx / 4xx triage
```

The orchestrator prints an aggregate Markdown table to chat:

```
# Run report — 20260524T101530Z_a1b2c3d4

Stories 2 | Passed 1 | Failed 1

| Status | Story                       | Artifacts                                                |
| ------ | --------------------------- | -------------------------------------------------------- |
| PASS   | Front page loads with posts | ai_review/runs/<run>/front-page-loads-with-posts/        |
| FAIL   | Checkout flow completes     | ai_review/runs/<run>/checkout-flow-completes/            |

## Failures

### Checkout flow completes
- Failing step: Click Pay button
- Console log: ai_review/runs/<run>/checkout-flow-completes/console.log
- Network log: ai_review/runs/<run>/checkout-flow-completes/requests.log
- Subagent report excerpt:
  RESULT: FAIL
  - Steps: 2/5 passed (failed at step 02)
  ...

Run dir: ai_review/runs/20260524T101530Z_a1b2c3d4
```

Two report layers exist — read whichever you need:

| Want to know… | Look at… |
| --- | --- |
| Did the whole Run pass? | Aggregate table in chat. |
| Which Step failed in a given Story? | `<run-dir>/<slug>/` — Steps are named `NN_<step-slug>.png`. The highest NN that exists is the failing one (subsequent Steps were SKIPPED). |
| What did the browser see at failure time? | The PNG. |
| What did the page log? | `console.log` (only written on FAIL). |
| Was it a network problem? | `requests.log` (only written on FAIL). |

## Failure classes

| Class | What it means | Artifacts |
| --- | --- | --- |
| **FAIL** | Subagent opened a Session and a Step failed. | PNG of failing Step + `console.log` + `requests.log`. |
| **FAIL (pre-flight)** | Orchestrator rejected the Story before opening a browser (e.g. missing env var). | None — no Session was opened. |
| **Contract violation** | Subagent returned FAIL but didn't write a required log file. | The Story still counts as FAIL; the aggregate report annotates the row with `(contract violation: missing requests.log)`. |

Pre-flight failures don't have artifacts on purpose — they happen before any browser opens, so there's nothing to capture. If a Story consistently pre-flight-FAILs, check that the env vars its workflow references are exported in your shell.

## Secrets via env vars

Reference any secret in the workflow body with `${VAR_NAME}`:

```yaml
workflow: |
  Navigate to https://app.example.com/login.
  Fill the email field with ${APP_EMAIL}.
  Fill the password field with ${APP_PASSWORD}.
  Click Sign In.
```

Before running, export in your shell:

```bash
export APP_EMAIL='alice@example.com'
export APP_PASSWORD='hunter2'
```

The subagent resolves the variable through bash at execute time (`bash -c '... fill <ref> "$APP_PASSWORD"'`). The plaintext value never enters the LLM context, never lands in a screenshot caption, never gets written to a Report. The Step row in the Report reads `Fill password field with ${APP_PASSWORD}` (the placeholder, verbatim).

If a variable is unset when you invoke `/browser-qa`, the orchestrator pre-flight-FAILs that Story with `missing env vars: <names>` — no browser opens, no Session leaks.

## Vision verify

Default verification is text-based: re-snapshot the ARIA tree, check the YAML for the expected element/text, check console + network for noise.

Switch to vision verify when:

1. The Step's prose contains the literal phrase **"visually verify"** (case-insensitive), or
2. The Run was invoked with the `vision` flag (applies to every Step in every Story).

In vision mode, the subagent ingests the screenshot pixels to make the call. Use when text-verify can't see what matters — layout regressions, image/canvas content, computed styles that don't surface in the ARIA tree.

Cost: vision verify is slower and burns more tokens (the model ingests a PNG per Step). Don't reach for it as a default — `visually verify` per-Step in workflow prose is the right granularity.

## Save-test mode

Run with `save-test`:

```
/browser-qa save-test login
```

For every Story, the subagent:

1. Wraps execution in `tracing-start` / `tracing-stop`.
2. Zips the Playwright trace to `<run-dir>/<slug>/trace.zip`.
3. Emits a starter Playwright spec at `tests/<slug>.spec.ts`.

The spec is a **starter** — it captures the happy path with selectors that worked. You're expected to refine assertions and add edge cases by hand.

Replay the trace:

```bash
npx playwright show-trace ai_review/runs/<…>/<slug>/trace.zip
```

The trace is the source-of-truth artifact; the spec is a convenience. If a Story has dynamic refs or a complex SPA shape, the emitted spec may be a thin `goto` + TODO — that's flagged in the Run report.

`${VAR}` references in the workflow are preserved as `process.env.VAR!` reads in the spec. Resolved secret values never bake into spec text.

## Pruning runs

`ai_review/runs/` accumulates. Throw away old ones with `/qa-clean`:

```
/qa-clean                  # keep last 7 days (default)
/qa-clean 14               # keep last 14 days
/qa-clean all              # delete every Run
/qa-clean 0                # same as `all`
/qa-clean --dry-run        # preview a 7-day prune
/qa-clean all --dry-run    # preview a full wipe
```

`/qa-clean` only touches `ai_review/runs/` subdirectories matching the `<timestamp>_<uuid8>` pattern. Story YAMLs (`ai_review/user_stories/`) and the CLI working dir (`.playwright-cli/`) are never touched. Anything inside `ai_review/runs/` whose name doesn't match the Run-dir pattern is left alone with a warning.

## When something feels off

| Symptom | First place to look |
| --- | --- |
| `playwright-cli: command not found` | `INSTALL.md` § "One-time per machine". |
| `no stories found at ai_review/user_stories/*.yaml` | Check `pwd` is the repo root and the directory exists. |
| Every Story pre-flight-FAILs | Env vars referenced in workflow aren't exported in your shell. |
| Random Story passes locally but fails in CI (or vice-versa) | Leaked Session from a prior Run. `playwright-cli list` should print `(no browsers)` between Runs; `playwright-cli kill-all` flushes. |
| Verify catches favicon 404s as errors | The skill's noise policy already drops asset URL patterns — file the regression against `SKILL.md` § "Console + network noise policy". |
| A failure has no `requests.log` | Contract violation — the subagent split the FAIL capture into two Bash calls. The orchestrator's aggregate row will note it; the fix is in `bowser-qa-agent.md` § "Close". |

For deeper investigation, the design docs are authoritative:

- **What the harness can do** — [`SKILL.md`](./SKILL.md) (CLI command reference, noise policy, save-test, failure capture)
- **Per-Story execution contract** — [`../../../agents/bowser-qa-agent.md`](../../../agents/bowser-qa-agent.md) (input fields, Setup/Execute/Close, env-var handling, output template)
- **Orchestrator behavior** — [`../../../commands/browser-qa.md`](../../../commands/browser-qa.md) (discover/spawn/collect/report phases, pre-flight, defensive cleanup)
- **Retention command** — [`../../../commands/qa-clean.md`](../../../commands/qa-clean.md)
