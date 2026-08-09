# The canonical install block

One install story, one wording. `README.md`, `.changeset/*`, and every page under `docs/` must say **this** and nothing else. Change it here first, then propagate.

`hoangta-skills` is **not** in Claude Code's official marketplace — that listing is upstream's. This fork ships itself as a single-plugin marketplace via `.claude-plugin/marketplace.json`, so the marketplace has to be added once per machine before the plugin can be installed. There is no auto-update guarantee; a `/plugin update` or a re-install is how new versions arrive.

## Claude Code — the plugin

<canonical-block name="claude-code">

**One-time per machine** — register the marketplace:

```
/plugin marketplace add hoangtmbk/skills
```

**Per target repo** — install the plugin (project scope):

```
/plugin install hoangta-skills@hoangtmbk-skills
```

Or from your shell:

```bash
claude plugin marketplace add hoangtmbk/skills
claude plugin install hoangta-skills@hoangtmbk-skills
```

Restart Claude Code after install.

</canonical-block>

## Codex, and other agents — skills.sh

The plugin is Claude Code only. Everywhere else, [skills.sh](https://skills.sh/hoangtmbk/skills) copies editable skill files into the project. Use the whole-set form on `README.md`:

<canonical-block name="skills-sh-whole-set">

```bash
npx skills@latest add hoangtmbk/skills
```

Pick the skills you want, and which coding agents to install them on. **The installer lets you choose which skills to take — make sure `setup-hoangta-skills` is one of them.**

</canonical-block>

…and the single-skill form wherever one skill is named on its own. Note that **`docs/` pages are not a consumer of this block**: ai-hero renders the install widget above the body, so a page that writes the commands out duplicates it. See [writing-docs.md](./writing-docs.md).

<canonical-block name="skills-sh-one-skill">

```bash
npx skills@latest add hoangtmbk/skills --skill=<name>
```

```bash
npx skills@latest update <name>
```

</canonical-block>

`skills@latest` is the pinned spelling in all three. The pages under `docs/` used to carry their own copy of these commands; those blocks are now deleted rather than corrected, because the site renders the install commands itself.

## The two routes are exclusive

The plugin is a managed, read-only bundle you subscribe to. skills.sh writes files you own and edit. Installing both leaves the user with every skill twice — always say "pick one".

## Not the install story

The upstream repo [mattpocock/skills](https://github.com/mattpocock/skills) is listed in Claude Code's official marketplace as `mattpocock-skills`. That is upstream's install story, not this fork's — never document it here.
