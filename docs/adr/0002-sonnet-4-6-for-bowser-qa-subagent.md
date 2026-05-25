# Pin `bowser-qa-agent` to `claude-sonnet-4-6` to cut the dominant token line

The `/browser-qa` orchestrator fans out one `bowser-qa-agent` subagent per Story per Run. The subagent is ~95% of the LLM bill — it parses workflows, scans ARIA-tree YAMLs, judges verify outcomes, and (opt-in) ingests screenshots or emits Playwright specs. The orchestrator itself just parses YAML, emits `Agent()` calls, and renders a table. We pin the subagent's model to `claude-sonnet-4-6` via the `model:` field in `agents/bowser-qa-agent.md` (the canonical file in this plugin; symlinked into `~/.claude/agents/` after install). The orchestrator continues to inherit whatever model the interactive session uses.

> The bake-off below was conducted in the `browser-automation/` bootstrap repo against its `ai_review/user_stories/*.yaml` fixture set. That repo is being deprecated; the Run artifact paths referenced under "Bake-off raw" are historical and may not be reachable after deprecation. The verdicts and decision are what carry forward.

## Why

- **The default workload is a fit for Sonnet.** Text verify is well-structured prompt-following over a constrained tool set: parse Workflow → Steps, scan ARIA YAML for a ref, click/fill, check expected text + URL, apply the noise filter. Sonnet 4.6 handles this; paying Opus rates for it is the cost line we're targeting.
- **Uniform Sonnet beats tiered routing at this stage.** Vision verify and `save-test` are opt-in flags, not the daily driver. If they regress, we'll carve them back to Opus then (move to tiered) rather than pre-paying complexity for unmeasured risk.
- **Full model ID, not the `sonnet` alias.** A cost/quality-sensitive change shouldn't ride an alias that can silently roll to a new generation. Roll-forward should be an intentional one-line edit.

## Trade-off accepted

We give up Opus's procedural-reliability headroom on the subagent's harder paths (vision verify on subjective layout claims; spec generation from trace JSON). If a regression shows up, the rollback is a one-line revert in `agents/bowser-qa-agent.md`.

## Validation

Bake-off against `ai_review/user_stories/*.yaml` (5 stories: 3 PASS-expected, 2 designed-to-FAIL in `failures.yaml`). 3 Runs per side. Pass bar: every Story has the same verdict on all 3 Sonnet Runs and that verdict matches Opus and matches expected. The 2 `failures.yaml` Stories are canaries — a false-PASS on either is an automatic regression regardless of other signals.

Note: `finai-login.yaml` requires `FINAI_EMAIL` and `FINAI_PASSWORD` env vars. They aren't set in the bake-off environment, so that Story pre-flight FAILs on both sides identically — it contributes verdict parity but no model-exercising signal. Effective bake-off corpus is 4 stories.

| Side | Run 1 verdicts | Run 2 verdicts | Run 3 verdicts | Subagent token total (sum across 4 dispatches × 3 Runs) |
| --- | --- | --- | --- | --- |
| Opus 4.7 | FAIL/FAIL/PASS/FAIL (+ pre-flight) | same | same | ~227K |
| Sonnet 4.6 | FAIL/FAIL/PASS/FAIL (+ pre-flight) | same | same | ~240K |

100% verdict parity across all 6 Runs (4 effective stories × 3 Runs × 2 models). The vision Story consistently FAILs on both sides — the Story prose ("centered text box with a white background") contradicts the actual `#eeeeee` page background; both models correctly call the mismatch. This is a Story-prose bug to fix as a follow-up, not a model regression.

**Projected cost savings:** Sonnet 4.6 is ~1/5 the per-token price of Opus 4.7 (~$3/$15 vs ~$15/$75 per M input/output tokens). Subagent token counts are within ~6% between the two sides, so the per-token ratio dominates: ~**80% reduction in the dominant cost line** (the per-Story subagent dispatch, which is ~95% of the total Run cost). Hard dollar figures not captured (`/cost` was not run manually between Runs); the projection holds because token volumes are observed and the price ratio is published.

## Outcome

**A. Clean pass.** Ship Sonnet 4.6. The `model: claude-sonnet-4-6` pin is in `agents/bowser-qa-agent.md` in this commit.

## Follow-ups (not blocking)

1. **Rewrite the bake-off's `vision.yaml` story** so the prose matches what the page actually renders (`#eeeeee` body, no white-fill container). Currently both Opus and Sonnet correctly FAIL it because the prose is wrong — confusing data for someone reading the Run report at a glance.
2. **Investigate the playwright-cli session-slug socket-collision incident.** Observed on `wrong-heading-assertion` in Opus Run 2 and Sonnet Run 1 — the agent compensated by shortening the session slug, but this is the agent papering over a CLI/path-length bug that should be filed and fixed upstream (`@playwright/cli` or the skill's session-naming rule in `skills/engineering/playwright-qa/SKILL.md`).
3. **Consider having the orchestrator write `report.md` into the Run dir** (in addition to chat) so future bake-offs or regression analyses don't need to copy-paste reports out of chat history.

## Decision tree (pre-committed before the bake-off)

- **A. Clean pass** (100% verdict parity): ship. ← **realized outcome**
- **B. Intermittent flip** (1 story flips on 1 of 3 Sonnet Runs): re-run that story 3× more. If still flaky, root-cause via the artifacts under `ai_review/runs/<run>/<slug>/` and patch the relevant section of the agent contract.
- **C. Consistent flip on 1 story** (same Sonnet flip on all 3 Runs): diagnose-and-patch the contract; if still flips after patch, either rewrite the story to avoid the failure mode or move it to a tiered policy (Opus for that one).
- **D. Multiple stories flip** (2+ consistent Sonnet flips): abandon. Revert the `model:` line and record the failure pattern here.

## Rollback triggers (post-launch)

- An unexpected verdict flip on a known-truth Story in normal use. Two within a week → revert.
- A spike in Phase 3 contract-violation annotations (e.g. missing `requests.log` on FAIL rows) → Sonnet is rushing FAIL capture; revert.
- Any `failures.yaml`-style designed-to-FAIL Story flipping to PASS is automatic revert regardless of other signals.

Rollback mechanism: delete the `model:` line from `agents/bowser-qa-agent.md` and republish the plugin.

## Bake-off raw

### Opus 4.7 — Run 1 (`20260525T025421Z_d7295eac`)

| Status | Story |
| --- | --- |
| FAIL | Element not found |
| FAIL | Wrong heading assertion |
| FAIL (pre-flight) | Login to FinAI Findwise |
| PASS | Example.com loads |
| FAIL | Vision verify example.com layout |

Notes:
- `Element not found` and `Wrong heading assertion` FAILs are the designed `failures.yaml` outcomes.
- `Login to FinAI Findwise` pre-flight FAIL is the env-var miss (`FINAI_EMAIL` / `FINAI_PASSWORD` unset).
- **`Vision verify example.com layout` FAILed on Opus**, not PASSed as we'd assumed. The subagent's vision verify observed that example.com renders on a `#eeeeee` light-grey body background, not white. The Story prose asserts "centered text box with a white background". Opus correctly judged the prose against the actual pixels. **The Story's "expected PASS" assumption was wrong** — the prose contradicts the page.

### Opus 4.7 — Run 2 (`20260525T025822Z_97989d89`)

Identical verdicts to Run 1.

Minor incident: the `wrong-heading-assertion` subagent shortened its playwright-cli Session slug to `wrong-heading` after hitting a macOS Unix-socket path-length limit (stale truncated socket from a prior Run blocked listen). Artifact dir still uses the full Story slug; orchestrator artifact validation isn't affected. Not a model-quality signal.

### Opus 4.7 — Run 3 (`20260525T030102Z_903aa2b3`)

Identical verdicts to Runs 1-2. Third independent Opus vision Run reaching the same `#eee` background reasoning.

### Opus baseline summary (Runs 1-3)

100% verdict consistency across 3 Opus Runs. Recalibrated expected outcomes for the bake-off:

| Story | Expected (recalibrated) | Reason |
| --- | --- | --- |
| Element not found | FAIL | Designed-to-FAIL canary. |
| Wrong heading assertion | FAIL | Designed-to-FAIL canary. |
| Login to FinAI Findwise | FAIL (pre-flight) | Env vars not set. |
| Example.com loads | PASS | Story prose matches page. |
| Vision verify example.com layout | **FAIL** (was assumed PASS) | Story prose is wrong. |

### Sonnet 4.6 — Run 1 (`20260525T030612Z_5da72ca4`)

4/4 parity with Opus baseline. Vision call (highest-risk per Q5 of the design grilling) matched Opus's `#eee` reasoning on Run 1.

Same session-slug socket-collision incident as Opus Run 2 — agent compensated by shortening to `whav2`.

### Sonnet 4.6 — Run 2 (`20260525T030931Z_f49f008d`)

4/4 parity. No socket-collision recovery needed this time.

### Sonnet 4.6 — Run 3 (`20260525T031250Z_a979e1b8`)

4/4 parity. Same `#eeeeee` background reasoning — fourth time across Opus and Sonnet Runs combined that the same vision conclusion was reached independently.
