---
description: Discover YAML user-stories, fan out bowser-qa-agent subagents in parallel against a running web app, aggregate results into one Run report.
argument-hint: [headed] [vision] [save-test] [filename-filter]
---

# /browser-qa

Orchestrate end-to-end browser QA across every Story defined in `ai_review/user_stories/*.yaml`. Use this when you want a single command to fan out one `bowser-qa-agent` subagent per Story in parallel, then collect their PASS/FAIL verdicts into a unified Run report.

If a `CONTEXT.md` exists at the repo root, Read it first to load canonical terms (Story, Workflow, Run, Session, Artifact, Save-test). If it does not exist, proceed — the terms are defined in the `playwright-qa` skill and `bowser-qa-agent` contract; CONTEXT.md is an optional repo-local glossary. Do NOT re-implement subagent or Playwright behavior — that lives in the `bowser-qa-agent` subagent and the `playwright-qa` skill respectively. Your only jobs are: discover, fan out, collect, report.

This command runs in four strictly ordered phases. Each phase has a single responsibility; do not interleave them. If Phase 1 finds no Stories, the command exits before Phase 2 — no Run dir, no subagents.

## Arguments

Parse `$ARGUMENTS` as a whitespace-separated list of tokens. Each token is one of:

| Token | Effect |
| --- | --- |
| `headed` | Pass `headed=true` to every subagent (default: headless). |
| `vision` | Pass `vision=true` to every subagent (enables screenshot-driven verification). |
| `save-test` or `save_test` | Pass `save_test=true` to every subagent (persist a reusable test artifact). |
| anything else | Treated as a **filename substring filter** — only story files whose basename contains this token are run. |

Flag detection is case-insensitive and order-independent. At most one filename-filter token is expected; if multiple non-flag tokens appear, treat the LAST one as the filter and ignore earlier ones.

Examples:

- `/browser-qa` — run every Story headless, no vision, no save-test.
- `/browser-qa headed checkout` — run every Story whose filename contains `checkout`, with a visible browser.
- `/browser-qa vision save-test` — run every Story with vision verification and persist a reusable test for each.

## Phase 1 — Discover

1. Verify `ai_review/user_stories/` exists. If it does not, print:
   `no stories found at ai_review/user_stories/*.yaml (directory missing)` and exit. Do NOT create the directory.
2. Glob `ai_review/user_stories/*.yaml`. If the filter token is set, keep only files whose basename contains it (substring, case-insensitive). Record the count of matched files for the banner.
3. For each remaining file, Read it and parse the top-level `stories:` array. Each entry has `name`, `url`, and `workflow` (block scalar). Tolerate a missing or empty `stories:` key by skipping that file (do not error).
4. Build a flat list of `{file, name, url, workflow}` tuples. Preserve discovery order: stable-sort by file basename, then by the Story's position within its file.
5. **Env-var pre-flight.** For each tuple, regex-scan the `workflow` text for `\${[A-Z_][A-Z0-9_]*}` patterns and collect the unique var names referenced by that Story. For each name, check it is set in the env (`bash -c '[ -n "${VAR+x}" ]'`). If ANY var is missing for a given Story, mark that Story as **pre-flight FAIL** with reason `missing env vars: <comma-separated names>` — record it in the Phase 4 report and DO NOT spawn a subagent for it. The subagent would just have to re-discover the same failure and would risk leaking a Session. Other Stories with all their vars set still proceed normally.
6. If the discovered list is empty (no Stories at all), print:
   `no stories found at ai_review/user_stories/*.yaml` (append `(filter: <token>)` if a filter was applied) and exit. Do NOT proceed to Phase 2.
7. Generate the Run directory ONCE — every Story this command spawns belongs to this single Run:
   - `TIMESTAMP` = current UTC time formatted `YYYYMMDDTHHMMSSZ`
   - `UUID8` = first 8 chars of a fresh UUIDv4 (hex, no dashes)
   - `RUN_DIR=ai_review/runs/${TIMESTAMP}_${UUID8}`
   - `mkdir -p "$RUN_DIR"`
8. Echo a one-line banner: `Run: $RUN_DIR — <N> stories across <M> files` (plus `(<K> pre-flight FAIL)` if env-var pre-flight rejected any Stories).
9. If every discovered Story was pre-flight-failed, skip Phase 2 entirely and go straight to Phase 4 — the report will show only pre-flight failures, with no PNGs or console.logs (no Session was opened).

## Phase 2 — Spawn

Emit ALL Agent invocations in a SINGLE assistant message so they execute concurrently. One `Agent` call per Story tuple from Phase 1. Use `subagent_type: bowser-qa-agent` for every call. Pass the SAME `run_dir` to every subagent so all Artifacts land under the same Run.

This is the only step where parallelism matters. Sequencing the calls across multiple turns will multiply the wall-clock time by N (the number of Stories), which defeats the point of this command.

Literal template — repeat once per Story, substituting bracketed values:

```
Agent(
  subagent_type="bowser-qa-agent",
  description="QA: <story name>",
  prompt="""
Run the playwright-qa skill against this Story.

name: <story name>
url: <story url>
run_dir: <RUN_DIR>
headed: <true|false>
vision: <true|false>
save_test: <true|false>

workflow:
<verbatim workflow block from the YAML, preserving newlines and indentation>

End your final message with a `## Report` section and a final line exactly:
`RESULT: PASS` or `RESULT: FAIL`.
"""
)
```

Do not stagger spawns. Do not await one before starting another. The model MUST emit every Agent tool call in the same assistant turn — sequential turns will serialize them and defeat parallelism.

Flag values come from Phase 1's argument parse. If a flag was absent from `$ARGUMENTS`, pass `false` for that field explicitly (do not omit it). The subagent contract expects all four optional booleans plus `run_dir` to be present.

## Phase 3 — Collect

When all subagents return:

1. For each return, scan the final message for the LAST line matching `^RESULT:\s+(PASS|FAIL)\s*$`. Capture that verdict. Matching the LAST occurrence guards against the substring appearing earlier in narrative text.
2. Capture the `## Report` block (everything from `## Report` up to the `RESULT:` line) for inclusion in the aggregate.
3. Resolve the Story's Artifact subdirectory under `$RUN_DIR` (the subagent will have created one named after the Story).
4. Track running totals across the loop: `total`, `passed`, `failed`. Also keep an ordered list of per-Story rows so the report preserves discovery order.
5. If a subagent returned no `RESULT:` line OR errored: mark that Story as FAIL with the reason `subagent did not return RESULT` and continue — never abort the whole Run.

### Defensive session cleanup

After all subagents return AND before printing the Phase 4 report:

```bash
playwright-cli close-all
playwright-cli list
```

The `list` output must be `(no browsers)`. If a Session leaked (subagent crashed mid-Story, agent forgot to `close`), the row will show in `list`; follow with `playwright-cli kill-all` and note "leaked session(s) cleaned up" in the Phase 4 report's footer. Never start the next Run without this step — leaked browsers compound across Runs and produce flake.

### Artifact validation

For every FAIL row (including subagents that returned RESULT but excluding pre-flight FAILs):

- The Story's `<run_dir>/<story-slug>/console.log` MUST exist.
- The Story's `<run_dir>/<story-slug>/requests.log` MUST exist.

If either is missing, append `(contract violation: missing requests.log)` (or both) to that row in the Phase 4 report. Do not delete or retry the Story — surface the violation so the agent contract can be tightened.

## Phase 4 — Report

Print a single aggregate Run report using this literal template:

```
# Run report — <RUN_DIR basename>

Stories <N> | Passed <M> | Failed <K> (incl. <P> pre-flight)

| Status | Story | Artifacts |
| --- | --- | --- |
| PASS | <story name> | <run_dir>/<story-slug>/ |
| FAIL | <story name> | <run_dir>/<story-slug>/ |
| FAIL (pre-flight) | <story name> | — |
| ...  | ...          | ...                      |

## Failures

### <failing story name>
- Failing step: <step text from subagent report> (or `pre-flight: missing env vars: VAR_A, VAR_B`)
- Console log: <run_dir>/<story-slug>/console.log (omit for pre-flight FAILs)
- Network log: <run_dir>/<story-slug>/requests.log (omit for pre-flight FAILs)
- Subagent report excerpt:
  <first 5-10 lines of that subagent's `## Report` block> (omit for pre-flight FAILs)

(Repeat one block per failing Story. Omit this section entirely if K == 0.)

Run dir: <RUN_DIR>
```

Use `PASS` / `FAIL` as plain text status markers (no emoji unless the user requested them). Slugify Story names the same way the subagent did when creating its Artifact subdirectory. Pre-flight FAIL rows use status `FAIL (pre-flight)` and have no Artifact path because no Session was opened.

## Failure handling

- A subagent that crashes, times out, or returns without a `RESULT:` line counts as FAIL with reason `subagent did not return RESULT`. Continue aggregating the remaining Stories — one bad Story must not poison the Run report.
- Never delete or rewrite an existing Run dir — if `mkdir -p` reports the path already exists (collision), regenerate `UUID8` and retry up to 3 times before erroring out.
- Never modify the YAML story files. They are inputs only.
- If Phase 1 exits early (no stories, missing directory), do NOT create a Run dir and do NOT spawn any subagents.
- The exit status of `/browser-qa` is conceptual only — the command always finishes by printing the aggregate report. Callers read PASS/FAIL counts from the report, not from a return code.
