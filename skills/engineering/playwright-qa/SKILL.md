---
name: playwright-qa
description: Headless functional QA of running web apps via Microsoft @playwright/cli. Snapshots of the ARIA tree are written to disk and read selectively by ref — never dumped wholesale into the context window — so the LLM only ingests the few elements it needs to act on.
---

This skill drives `@playwright/cli` to execute QA Stories against a running web app. The agent extracts Steps from a Story's Workflow, runs them in a named Session, and produces one screenshot Artifact per Step.

**Central rule:** every Snapshot is written to `.playwright-cli/page-*.yml` on disk. Read only the refs you need from that file — never paste the whole tree into chat. This is the entire token-efficiency story.

## Pre-flight check

Before doing anything else, verify the CLI is installed:

```bash
playwright-cli --version
```

If it errors, **stop and tell the user** to run:

```bash
npm install -g @playwright/cli
npx playwright install chromium
```

Do not auto-install. Resume only after the user confirms.

## Core 3-step pattern

Every interaction follows: **open → snapshot → interact by ref**.

```bash
playwright-cli -s=<session> open <url> --persistent > /dev/null   # 1. open in a named Session (also writes & echoes a Snapshot)
playwright-cli -s=<session> snapshot --filename .playwright-cli/page-<NN>.yml > /dev/null   # 2. write ARIA tree to disk
# 3. Read the YAML file, find the ref you need (e.g. e17), then act:
playwright-cli -s=<session> click e17
playwright-cli -s=<session> fill e42 "user@example.com"
```

Refs (`e17`, `e354`) are stable only within the most recent Snapshot for that Session.

**`--filename` is mandatory on `snapshot`.** Calling `snapshot` bare echoes the full ARIA tree to stdout instead of writing a file — which defeats the entire token-efficiency rule. Always pass `--filename` and redirect stdout to `/dev/null` (the file content is the YAML; stdout duplicates it). The same applies to `open --persistent`: it both writes a Snapshot file *and* echoes the tree on stdout — redirect stdout to keep the tree out of your context.

## Named Sessions

Every Story owns exactly **one** named Session. The name is derived from the Story slug so parallel Stories don't collide.

```bash
playwright-cli -s=<story-slug> open <url> --persistent
# ... Steps ...
playwright-cli -s=<story-slug> close   # always close at end of Story
```

`--persistent` preserves cookies/localStorage across calls within the Session. Closing the Session releases the browser instance.

## Snapshot discipline

- A Snapshot lives on disk at `.playwright-cli/page-*.yml`. The Workflow agent opens the file, greps/scans for the element it needs, and pulls **only that ref** into its reasoning.
- **Never** paste the full YAML tree into a chat message or an LLM prompt. That defeats the point of the CLI.
- **Re-snapshot after every navigation or DOM mutation.** Refs from a stale Snapshot are the #1 cause of silent failures. Triggers that demand a fresh snapshot: any `goto`, any `click` that changes the URL, route changes, dialog open/close, async content load, single-page-app view swap, form submit.
- Treat Snapshots as cheap. Re-take liberally; never reuse a ref across a navigation boundary.

## Command reference

**Navigation**
- `open <url> --persistent` — open page in this Session
- `goto <url>` — navigate the existing Session's page
- `press <key>` — keyboard press (e.g. `Enter`, `Escape`)
- `keydown <key>` / `mousemove <x> <y>` — low-level input

**Interaction**
- `click <ref>` — click element by ref from latest Snapshot
- `fill <ref> <text>` — fill form input by ref
- `route <pattern>` — install request route/mock (see references for mocking syntax)

**Info / State**
- `snapshot --filename <path>` — write ARIA-tree YAML to `<path>`. WITHOUT `--filename` the command echoes the tree to stdout and writes no file — never call it bare, or the YAML lands in your LLM context.
- `tab-list` — list open tabs in the Session
- `state-save <file>` / `state-load <file>` — persist or restore storage state
- `console` — dump JS console messages from the page (use on FAIL)
- `run-code <js>` — evaluate JS in the page context

**Media**
- `screenshot --filename <path>` — write a PNG to `<path>`. The positional argument is an *element selector/ref*, NOT a file path — `screenshot foo.png` silently treats `foo.png` as a selector and writes no file. Always use `--filename`.
- `video-start` / `video-stop` — record `.webm` for the Session

**Tracing**
- `tracing-start` / `tracing-stop` — record a Playwright trace (used by Save-test)

**Lifecycle**
- `close` — close the Session (required at end of Story)

## Verify mode

**Default = text Verify.** After a Step's action:

1. Re-`snapshot`.
2. Check the YAML for the expected element/text and URL.
3. Run `playwright-cli -s=<session> console error` and confirm no new JS errors at `error` level. Then run `playwright-cli -s=<session> requests --filter "/api/.*"` and confirm no 5xx responses on app/API traffic.

### Console + network noise policy

The CLI severity filter cuts most noise, but the browser still emits asset-load failures (favicon 404, blocked third-party trackers) as `error`-level console messages. Apply both filter layers — severity AND URL pattern — so every subagent sees the same signal:

1. **Severity filter (at the CLI):** `console error` — drops `warn` and `info`.
2. **URL post-filter (in the agent's reasoning):** ignore any captured error whose message references a known-benign asset URL pattern. Treat these as noise:
   - `*/favicon.ico`
   - `*.css`, `*.js`, `*.woff`, `*.woff2`, `*.ttf`, `*.eot`
   - `*.png`, `*.jpg`, `*.jpeg`, `*.gif`, `*.svg`, `*.webp`
   - Any third-party analytics/ad domain you can identify (`googletagmanager`, `doubleclick`, `hotjar`, etc.)
3. **Network signal (at the CLI):** `requests` (no `--static`) — static assets already excluded by default. Use `--filter "/api/.*"` to narrow to your app's API surface.
4. **Pass criteria:** zero `error`-level console messages AFTER URL post-filter AND zero 5xx responses on filtered requests. 4xx on app routes is a FAIL signal *only* when the Step's prose expected success (e.g. "submit returns 200" — a 401 is a FAIL; a 404 on `/favicon.ico` is never a FAIL).
5. Use `console error --clear` and `requests --clear` between Steps if you want strictly-additive verification; otherwise carry-over noise from earlier Steps will count against later ones.

The full `console.log` and `requests.log` written on FAIL contain the unfiltered traffic — they're for the reviewer to confirm the agent's noise-filtering judgment, not for the agent to re-ingest.

**Vision Verify is opt-in only.** Trigger it when:
- the Step contains the literal phrase **"visually verify"**, or
- the caller passed `--vision` to the Run.

In Vision mode, take a `screenshot` and prefix the command with the env var so the CLI returns the screenshot as an image response the LLM can inspect:

```bash
PLAYWRIGHT_MCP_CAPS=vision playwright-cli -s=<session> screenshot --filename <path>
```

## Failure capture

On any Step FAIL, before stopping the Story:

1. `playwright-cli -s=<session> screenshot --filename <run-dir>/<story-slug>/NN_<step>.png` (the Step's normal Artifact).
2. `playwright-cli -s=<session> console error > <run-dir>/<story-slug>/console.log` — capture JS error-level messages (use `console warn` if you need a wider net for diagnosis).
3. `playwright-cli -s=<session> requests > <run-dir>/<story-slug>/requests.log` — non-static network traffic, for diagnosing HTTP-level failures.
4. Mark all remaining Steps in the Story as **SKIPPED** (do not run them).
5. `playwright-cli -s=<session> close` — release the Session.
6. Return a structured FAILURE report referencing the screenshot, `console.log`, and `requests.log`.

## Session hygiene

Every Session opened with `-s=<name>` lives until explicitly closed. Stale browser processes are the #1 source of CI flakes; treat cleanup as load-bearing:

- **Always `close` at end of Story.** Both on PASS and FAIL — the Failure-capture step 5 above is non-negotiable.
- **`playwright-cli close-all`** flushes every Session in one call. The orchestrator (`/browser-qa`) runs this defensively after Phase 3, but also call it manually before a new Run when iterating locally.
- **`playwright-cli list`** should print `(no browsers)` between Runs. If it doesn't, run `playwright-cli kill-all` (forceful) before debugging anything else.
- A `.playwright-cli/` working directory will accumulate across Runs — gitignore it. Snapshot files inside are throwaway between calls.

## Save-test mode (opt-in)

Triggered when `save_test=true` is in scope for the Run. Wrap the Story's Steps:

```bash
playwright-cli -s=<session> tracing-start
# ... Story Steps ...
playwright-cli -s=<session> tracing-stop
```

`tracing-stop` prints the paths of the trace files inside `.playwright-cli/traces/`. After `close`:

1. **Zip the trace** into `<story-dir>/trace.zip`. `tracing-start` prints the trace file paths — capture the trace ID from that output and zip ONLY that ID's files (plus `resources/`) so concurrent save-test Stories don't cross-contaminate:
   ```bash
   TRACE_ID=<captured from tracing-start output, e.g. 1779537337235>
   STORY_DIR=<absolute path to story_dir>
   ( cd .playwright-cli/traces && zip -r "$STORY_DIR/trace.zip" "trace-${TRACE_ID}.trace" "trace-${TRACE_ID}.network" "trace-${TRACE_ID}.stacks" resources )
   ```
   The resulting `trace.zip` is replayable in Playwright Trace Viewer: `npx playwright show-trace <story-dir>/trace.zip`.
2. **Emit a starter spec** at `tests/<story-slug>.spec.ts`. Use the Workflow prose as the source of truth for intent (navigation URLs, fill values, click targets) and the trace as reference for the locators/selectors that actually worked. The spec is "starter" because it captures the happy path; the user is expected to refine assertions and add edge cases.
3. **Spec template:**
   ```ts
   import { test, expect } from '@playwright/test';

   test('<story name>', async ({ page }) => {
     await page.goto('<url>');
     // One playwright call per Step — derive selectors from the trace JSON
     await expect(page.getByRole('heading', { name: 'Example Domain' })).toBeVisible();
   });
   ```
4. **Do NOT include secrets** in the emitted spec. If the Workflow contained `${VAR}` references, preserve them as `process.env.VAR` reads in the spec: `await page.getByLabel('Password').fill(process.env.FINAI_PASSWORD!);`. Never bake the resolved value into the spec text.
5. The trace is the artifact; the spec is a convenience. If trace-to-spec proves unreliable for a given Story (complex SPA, dynamic refs), it's OK to emit a minimal spec with just `goto` + a TODO comment — flag this in the FAIL/PASS report so the user knows the spec needs hand-finishing.

## Headed mode (opt-in)

Default is headless. To watch the browser, pass `--headed` through to `open`:

```bash
playwright-cli -s=<session> open <url> --persistent --headed
```

Only flip this when the caller explicitly asks (e.g. the Run was invoked with a `headed` argument).

## Artifact paths

- **Run directory:** `ai_review/runs/<timestamp>_<uuid>/` — shared across all Stories in one Run.
- **Per-Story subdirectory:** `<run-dir>/<story-slug>/` — `mkdir -p` it before the first Step.
- **Per-Step screenshot:** `<run-dir>/<story-slug>/NN_<step-slug>.png` (zero-padded, e.g. `00_open_home.png`, `01_click_login.png`).
- **On failure:** `<run-dir>/<story-slug>/console.log`.
- **Save-test mode:** `<run-dir>/<story-slug>/trace.zip` and the emitted `tests/<story-slug>.spec.ts`.

Snapshots in `.playwright-cli/` are working files, not Artifacts — they may be overwritten freely and should be gitignored.
