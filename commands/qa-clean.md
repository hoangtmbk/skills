---
description: Prune ai_review/runs/ artifacts older than a retention window. Defaults to keeping the last 7 days.
argument-hint: [days|all|--dry-run]
---

# /qa-clean

Delete old browser-QA Run artifacts under `ai_review/runs/`. Without arguments, keeps the most recent 7 days of Runs and deletes the rest.

This command does NOT touch `ai_review/user_stories/` (Story YAMLs are inputs, never outputs), `.playwright-cli/` (CLI working dir, gitignored and self-managing), or any non-Run artifact.

## Arguments

Parse `$ARGUMENTS` as a whitespace-separated list of tokens. Recognized tokens:

| Token | Effect |
| --- | --- |
| `all` | Delete every Run directory, regardless of age. |
| (integer N) | Keep the last N days of Runs; delete everything older. Default 7. |
| `--dry-run` | Print what WOULD be deleted; delete nothing. Combine with `all` or an integer. |

If multiple integers appear, use the last one. If both `all` and an integer appear, `all` wins. Unknown tokens are ignored with a warning.

Examples:
- `/qa-clean` — keep last 7 days, delete the rest.
- `/qa-clean 14` — keep last 14 days.
- `/qa-clean all` — delete every Run.
- `/qa-clean 0` — delete every Run (same as `all`).
- `/qa-clean --dry-run` — preview a 7-day prune without deleting.
- `/qa-clean all --dry-run` — preview a full wipe without deleting.

## Phase 1 — Resolve

1. If `ai_review/runs/` does not exist, print `nothing to clean (ai_review/runs/ does not exist)` and exit.
2. Parse arguments per the table above. Compute `MODE` (`all` | `keep-N`) and `DRY_RUN` (bool).
3. List immediate subdirectories of `ai_review/runs/`. Each subdirectory name SHOULD match `YYYYMMDDTHHMMSSZ_<uuid8>` (the `/browser-qa` Run-dir format). Non-matching entries are skipped with a warning — do NOT delete them.

## Phase 2 — Select

For each candidate Run directory, parse its leading timestamp (`YYYYMMDDTHHMMSSZ`) into UTC and compute age in days from now.

- `MODE=all` → mark every Run for deletion.
- `MODE=keep-N` → mark for deletion any Run whose age in days is `> N`.

Build two lists: `to_delete` (sorted oldest-first) and `to_keep` (sorted newest-first). Record total bytes per list using `du -sb` (or `du -sk` and multiply on macOS where `du -sb` is unavailable).

## Phase 3 — Act (or preview)

If `DRY_RUN=true`:

```
DRY RUN — no files deleted

Would delete <K> runs (<HUMAN_BYTES>):
  ai_review/runs/<oldest>
  ai_review/runs/<next>
  ...

Would keep <M> runs (<HUMAN_BYTES>):
  ai_review/runs/<newest>
  ...
```

If `DRY_RUN=false`:

```
Deleting <K> runs (<HUMAN_BYTES>)...
```

Then `rm -rf` each `to_delete` path one by one. After deletion:

```
Done. Kept <M> runs (<HUMAN_BYTES>), freed <HUMAN_BYTES>.
```

## Safety

- Never use `rm -rf ai_review/runs/` (the root). Always target individual Run subdirectories you matched in Phase 2.
- Never delete an entry whose name does NOT match the Run-dir regex — that's something else and you don't own it.
- Never delete from `ai_review/user_stories/` or anywhere outside `ai_review/runs/`.
- Refuse to operate if `pwd` is outside a repo (no `.git` ancestor) — this is a destructive command and the working directory must be the project root.
