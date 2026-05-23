# Maintaining

How to manage this fork and use these skills in other repos.

## Repo layout

- Source of truth: `/Users/hoangta/workspace/skills/`
- Published at: https://github.com/hoangtmbk/skills (public)
- Upstream: https://github.com/mattpocock/skills

## Git remotes

```
origin    git@github.com:hoangtmbk/skills.git    # my fork
upstream  https://github.com/mattpocock/skills   # original
```

## Pulling changes from upstream

Matt ships new skills regularly. To pick up his changes:

```bash
git fetch upstream
git log --oneline main..upstream/main          # what's new upstream
git log --oneline upstream/main..main          # what I've changed locally
```

Then either:

- **Cherry-pick selectively** (recommended for most updates):
  ```bash
  git cherry-pick <sha>
  ```
- **Merge everything**:
  ```bash
  git merge upstream/main
  # resolve conflicts (most likely in renamed skill folder)
  ```

### Expected conflicts

- `skills/engineering/setup-matt-pocock-skills/` no longer exists locally — it's `setup-hoangta-skills/`. Any upstream edit to the old folder needs to be applied to the new folder by hand.
- `README.md`, `CONTEXT.md`, `.claude-plugin/plugin.json` — local versions diverge from upstream.

Most other skills (`tdd`, `diagnose`, `triage`, etc.) are unchanged from upstream, so they merge cleanly.

## Adding a new custom skill

1. Create `skills/<bucket>/<name>/SKILL.md` with frontmatter:
   ```yaml
   ---
   name: <name>
   description: <one line — used to decide relevance>
   ---
   ```
2. Add a reference to it in:
   - Top-level `README.md` (under the right bucket)
   - `skills/<bucket>/README.md`
   - `.claude-plugin/plugin.json` (if it's in `engineering/`, `productivity/`, or `misc/`)
3. Bucket rules (from `CLAUDE.md`):
   - `engineering/` / `productivity/` / `misc/` — must be in `README.md` + `plugin.json`
   - `personal/` / `in-progress/` / `deprecated/` — must NOT be in either

## Adding a new agent or slash command

The plugin can also ship subagent definitions and slash commands alongside skills (e.g. the `playwright-qa` harness ships its `bowser-qa-agent` subagent and `/browser-qa` + `/qa-clean` commands as a set). Claude Code auto-discovers these from top-level dirs in the plugin:

- `agents/<name>.md` — one subagent per file. Frontmatter declares `name`, `description`, and `tools`. Referenced from the same plugin's commands as `subagent_type: <name>` (unprefixed — siblings in the same plugin don't need the `hoangta-skills:` namespace).
- `commands/<name>.md` — one slash command per file. Frontmatter declares `description` and optionally `argument-hint`. Invoked by the user as `/<name>` (no prefix needed).

No entry in `plugin.json` is needed for agents or commands — the directories are auto-discovered. The only file you need to touch besides the new agent/command file itself is whichever SKILL.md "owns" the harness, plus its `INSTALL.md` and the engineering bucket's `README.md` (to mention the bundled commands so users know what they get with the skill).

**Caveat: skills.sh installer does not ship agents or commands.** The `npx skills@latest add hoangtmbk/skills` path only copies SKILL.md files into the target repo's `.claude/skills/`. If a skill depends on a sibling agent or command, document that the user must install via the Claude Code plugin path (Option B below), not via skills.sh.

## Local install (this machine)

Skills currently live at `~/.claude/skills/` as symlinks pointing into `~/.agents/skills/` (the old skills.sh install from before this fork existed).

To switch the source of truth to this workspace:

```bash
# WARNING: removes the old install pointers
rm -rf ~/.agents/skills

# Re-link from workspace
./scripts/link-skills.sh
```

After this, edits in the workspace go live immediately — no rebuild step.

**Agents and commands are NOT covered by `link-skills.sh`.** The script only scans `skills/**/SKILL.md`. To get the plugin's top-level `agents/` and `commands/` registered on this machine, install via the Claude Code plugin marketplace (Option B below) — or symlink each file under `~/.claude/agents/` and `~/.claude/commands/` by hand (Claude Code auto-discovers user-level files in those dirs).

## Installing in another repo

Two paths.

### Option A — skills.sh installer (works in any agent)

In any project directory:

```bash
npx skills@latest add hoangtmbk/skills
```

The interactive flow asks which skills you want and which agent(s) to install them into (Claude Code, Codex, Gemini CLI, etc.). **Always select `setup-hoangta-skills`** — engineering skills depend on it.

Then in the agent, run:

```
/setup-hoangta-skills
```

This seeds three files under `docs/agents/` so the engineering skills know:
- which issue tracker to use (GitHub, GitLab, local markdown)
- what the repo's triage label vocabulary is
- where `CONTEXT.md` and ADRs live

**Limitation: skills.sh ships skills only, not subagents or slash commands.** Skills that bundle a sibling subagent or slash command (e.g. `playwright-qa` ships with `bowser-qa-agent` + `/browser-qa` + `/qa-clean`) won't be fully functional via this path. Use Option B for those, or fall back to the skill's own `INSTALL.md` for the manual copy recipe.

### Option B — Claude Code plugin

Add this repo as a plugin in Claude Code via the marketplace, pointing at `hoangtmbk/skills`. The `.claude-plugin/plugin.json` manifest controls which skills load. Same `/setup-hoangta-skills` step applies once installed.

### Per-repo setup checklist

After installing in a new repo:

1. Run `/setup-hoangta-skills` — answers go into `docs/agents/`.
2. Optional: write a starter `CONTEXT.md` at the repo root (or `CONTEXT-MAP.md` for monorepos) — `/grill-with-docs` and `/improve-codebase-architecture` use it.
3. Optional: create `docs/adr/` for architectural decisions.

## Branding split (important)

There's an intentional split between the GitHub identity and the in-repo identity:

| Thing | Value |
|---|---|
| GitHub repo URL | `hoangtmbk/skills` |
| skills.sh install | `npx skills@latest add hoangtmbk/skills` |
| Plugin name in `plugin.json` | `hoangta-skills` |
| Setup slash command | `/setup-hoangta-skills` |
| Setup skill folder | `skills/engineering/setup-hoangta-skills/` |

This is because `hoangtmbk` is the authenticated GitHub account but `hoangta` is the local git user.name. Leaving the slug as `hoangta` keeps the slash command short and matches the in-repo identity.

## Commit conventions

Match Matt's style (see `git log upstream/main --oneline`):

- Imperative mood, sentence case
- Subject under ~70 chars, no period
- Body explains *why* if not obvious
- One concern per commit (rebrand-only commits stay separate from skill-content changes)

## Files to never touch from upstream merges

- `MAINTAINING.md` (this file — local-only)
- `.claude-plugin/plugin.json` `name` field
- README install commands
- `CONTEXT.md` title

If a merge tries to touch these, take the local version.

## Future TODOs

- Move `~/.claude/skills/drawio/` into `skills/personal/` or `skills/productivity/` once decided
- Move `~/.claude/skills/library/` into the workspace (it's currently its own git repo)
- Switch `~/.claude/skills/` symlinks to point at the workspace instead of `~/.agents/skills/`
