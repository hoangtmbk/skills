# Install: browser-QA harness

A `/browser-qa` slash command that runs YAML-defined user Stories against a live web app in parallel and produces PASS/FAIL reports with PNG/console/network/trace artifacts.

The harness ships three things together — they only make sense as a set:

- `playwright-qa` skill — capability layer over `@playwright/cli`
- `bowser-qa-agent` — per-Story subagent contract
- `/browser-qa` + `/qa-clean` slash commands — orchestrator + retention

## Install via Claude Code plugin (recommended)

If you've already installed the `hoangta-skills` plugin in Claude Code, you already have everything. Otherwise:

1. Add `hoangtmbk/skills` as a plugin via the Claude Code marketplace (see the top-level `README.md` of this repo for the install command).
2. Restart Claude Code so the plugin's skills, agents, and commands are registered.
3. Run `/setup-hoangta-skills` once per repo (sets up `docs/agents/` config the other engineering skills consume — optional for `playwright-qa` but installed alongside).

After install, the slash command `/browser-qa` is available in any project.

## Install via manual copy (no plugin)

If you can't or don't want to install the plugin, copy these 5 files from `hoangtmbk/skills` into your target repo, preserving relative paths:

```
.claude/skills/playwright-qa/SKILL.md       <- skills/engineering/playwright-qa/SKILL.md
.claude/skills/playwright-qa/INSTALL.md     <- skills/engineering/playwright-qa/INSTALL.md (this file)
.claude/agents/bowser-qa-agent.md           <- agents/bowser-qa-agent.md
.claude/commands/browser-qa.md              <- commands/browser-qa.md
.claude/commands/qa-clean.md                <- commands/qa-clean.md
```

Restart Claude Code after copying so the new `.claude/` files are picked up.

## One-time per machine

Install Microsoft's `@playwright/cli` and download Chromium:

```bash
npm install -g @playwright/cli
npx playwright install chromium
```

Verify:

```bash
playwright-cli --version    # should print 0.1.x
```

The CLI itself is small (~few MB); Chromium binary is ~200 MB. Both live outside the target repo so no per-repo install is needed.

## One-time per target repo

Append to the repo's `.gitignore` (or create one):

```
.playwright-cli/
ai_review/runs/
```

`.playwright-cli/` is the CLI's working dir (snapshot YAMLs, console logs, traces — all throwaway). `ai_review/runs/` is where this harness writes per-Run artifacts. The `ai_review/user_stories/` directory (where Story YAMLs live) should be checked in.

## Writing your first Story

Create `ai_review/user_stories/smoke.yaml`:

```yaml
stories:
  - name: "Example.com loads"
    url: "https://example.com/"
    workflow: |
      Navigate to https://example.com/.
      Verify the page heading reads "Example Domain".
      Verify there is a link whose text is "Learn more".
```

Each YAML file holds one or more Stories. A Story has a free-text `workflow` that the `bowser-qa-agent` subagent decomposes into ordered Steps.

## Running

In Claude Code at the repo root:

```
/browser-qa
```

Runs every Story in `ai_review/user_stories/*.yaml`, in parallel (one subagent per Story), and prints an aggregate Run report. Artifacts land in `ai_review/runs/<timestamp>_<uuid>/<story-slug>/`.

Useful arguments:

- `/browser-qa smoke` — filter by filename substring (only runs `smoke.yaml`).
- `/browser-qa headed` — show the browser window (default headless).
- `/browser-qa vision` — vision-mode verification on every Step (default text-only).
- `/browser-qa save-test` — persist a Playwright trace.zip and emit a starter `tests/<story>.spec.ts` per Story.
- `/qa-clean` — prune `ai_review/runs/` older than 7 days. `/qa-clean all` for full wipe, `/qa-clean --dry-run` to preview.

Flags compose. `/browser-qa vision save-test login` runs only stories whose filename contains `login`, in vision-verify mode, with traces persisted.

## Secrets via env vars

Workflow prose may reference environment variables with `${VAR_NAME}` syntax:

```yaml
workflow: |
  Navigate to https://app.example.com/login.
  Fill the email field with ${APP_EMAIL}.
  Fill the password field with ${APP_PASSWORD}.
  Click the Sign In button.
```

Before invoking `/browser-qa`, export the vars in your shell:

```bash
export APP_EMAIL=alice@example.com
export APP_PASSWORD='hunter2'
```

The subagent resolves these through `bash` at execute time so plaintext values never enter the LLM context, never appear in logs, never get written to Reports. A missing required env var fails the Story BEFORE any browser opens (no Session is leaked).

## Optional repo-local glossary

If your repo has a `CONTEXT.md` at the root, the `/browser-qa` orchestrator will load it for any project-specific terminology. The harness works without it.

## Verifying the install

After installing the plugin (or copying files manually), restart Claude Code, then run:

```
/browser-qa smoke
```

If `smoke.yaml` does not exist yet, the command exits with `no stories found at ai_review/user_stories/*.yaml`. Add the example Story above and re-run. A clean install produces a PASS Run report with PNG artifacts under `ai_review/runs/<timestamp>_<uuid>/example-com-loads/`.

## Common gotchas

- **Subagent not registered.** Restart Claude Code after install — the agents directory is scanned at session start.
- **`playwright-cli: command not found`.** The CLI is a global npm package. If you use `nvm`, install on the active Node version (`nvm use` first).
- **Sessions leak after a crashed Run.** `playwright-cli list` shows leftovers; `playwright-cli kill-all` flushes them. The `/browser-qa` orchestrator runs `close-all` defensively at the end of every Run.
- **Vision mode triggers unexpectedly.** The literal phrase "visually verify" in Workflow prose switches that Step to vision mode. Avoid the phrase in text-only Workflows.

## Where the design lives (in the hoangtmbk/skills repo)

- `skills/engineering/playwright-qa/SKILL.md` — what the harness can do, CLI command reference, noise policy, save-test contract, failure-capture sequence.
- `agents/bowser-qa-agent.md` — per-Story execution contract: input fields, Setup/Execute/Close phases, env-var handling, Output format template.
- `commands/browser-qa.md` — orchestrator: Phase 1 Discover, Phase 2 Spawn (parallel), Phase 3 Collect + cleanup, Phase 4 Aggregate report.
- `commands/qa-clean.md` — retention command for `ai_review/runs/`.

If something behaves unexpectedly, those four files are the source of truth.
