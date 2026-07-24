---
name: ask-codex
description: Get a second opinion from Codex — an independent expert that reads the repo. Use when the user asks for Codex's take, an outside expert angle, or a fact-check, or when a load-bearing claim stays contested after you've inspected the code yourself.
---

# Ask Codex

Consult the Codex CLI as a read-only outside expert. Codex sees the repo but none of this conversation — the brief you write is everything it knows. One lens per consultation; for multiple expert angles, run separate consultations.

Announce the consultation before starting ("Consulting Codex on X — takes a few minutes"); the run blocks for roughly 3–5 minutes.

1. Create a fresh directory `$DIR` in the scratchpad and write `$DIR/brief.md` with five sections:
   - **Lens** — the expert Codex plays.
   - **Question** — the one thing to answer.
   - **Context** — every conversation fact the answer depends on, inlined: decisions already made, the contested claim, the constraints. No secrets.
   - **Repo pointers** — file paths worth reading. Codex will roam beyond them; that's expected.
   - **Answer shape** — verdict first; numbered points; each repo claim tagged as observed fact (with `path:line`), inference, or opinion; ends with the files actually read.
2. Run it synchronously with a 10-minute timeout:

   ```sh
   codex exec --sandbox read-only -o "$DIR/answer.md" < "$DIR/brief.md" > /dev/null 2> "$DIR/log.txt"
   ```

   stdout duplicates the answer and stderr carries the progress log — hence the redirects. If it exits 1 with `Not inside a trusted directory`, re-run with `--skip-git-repo-check`.
3. Verify `$DIR/answer.md` exists and is non-empty — exit 0 does not guarantee the write. If the run failed (or `codex login status` exits non-zero), report that Codex was not consulted; never invent an answer on its behalf.
4. Deliver the synthesis: for every numbered point Codex made, state the claim attributed to Codex, then whether you agree or disagree and why — verifying any repo fact against the code before repeating it. The synthesis is complete when each of Codex's numbers is accounted for.

**Follow-up** — the session id is in `$DIR/log.txt`; continue the same consultation with `codex exec resume <session-id> "follow-up question" -o "$DIR/answer-2.md"`. (`resume --last` is racy when other Codex sessions interleave.)

**Model** — omit `-m` so the user's Codex config decides; pass it only when the user names a model.
