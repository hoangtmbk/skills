---
name: grill-the-experts
description: A grilling where expert subagents answer the evidence questions; only preference calls come back to you.
disable-model-invocation: true
---

`/grilling` with one difference: the relentless one-at-a-time interview is unchanged, but a **panel** of advisor subagents answers instead of the user, who keeps only what evidence can't settle — the destination and their preferences.

## The rubric

Before any fan-out, run a short `/grilling` pass to pin three things: the **destination**, the hard constraints, and the user's standing preferences ("bias to boring tech", "TypeScript unless strongly contraindicated"). Together these are the **rubric** — the standard every advisor answer is judged against.

## The decisions document

One markdown file, created when the rubric is settled — named for the effort, placed where this repo keeps its docs or specs (ask during the brief if unclear). The header holds the destination and rubric; each resolved question appends an entry:

```markdown
### <question, as a title naming the answer>

**Question.** <the decision>
**Decided by:** panel (n/m advisors) | user | user via escalation
**Answer.** <the decision taken>
**Evidence.** <the claims it rests on, each attributed to its advisor and source>
**Dissent / reversal conditions.** <disagreement worth keeping, and what evidence would reopen the decision>
```

Append as you go.

## Walk the tree

Work the decision tree one question at a time, dependencies first. For each question, classify it: **would two well-informed experts disagree because of evidence, or because of preference?**

- **Preference, scope, risk appetite** → the user's. Ask inline with a recommended answer, `/grilling` style, and wait.
- **Evidence-resolvable** → the panel's. Convene it.

## Convene the panel

Route 2–3 advisors by what the answer hangs on; mixed questions mix rows:

| The answer hangs on              | Advisor                                                            |
| -------------------------------- | ------------------------------------------------------------------ |
| facts in the repo or workspace   | an explorer subagent                                               |
| the ecosystem's current state    | a websearch subagent (libraries, benchmarks, what prior art chose) |
| a design tradeoff                | expert-persona subagents with distinct, **opposed** lenses         |

Run the panel concurrently.

Each advisor's brief is self-contained — the advisor sees none of this conversation — and carries the destination, the rubric, the one question, and a mandatory **answer shape**: verdict first; numbered points each tagged `[fact]` (with source URL or `path:line`), `[inference]`, or `[opinion]`; closing with the evidence that would change its mind.

## Adjudicate

Judge the returns against the rubric, evidence-weighted: checkable `[fact]`s outrank `[inference]`, which outranks `[opinion]`, and an advisor conceding against its own lens is the strongest convergence signal. Verify any load-bearing checkable claim before adopting it. Record the entry in the decisions document, keeping the dissent and the union of the advisors' reversal conditions.

- **Contested after adjudication** → `/ask-codex` as tiebreaker: one lens, the contested claim inlined in the brief.
- **Still contested** → it was a preference call in disguise. Escalate to the user.

## Done

The tree is walked when no questions remain and nothing is silently assumed. Present the decisions document and confirm shared understanding — it's ready fuel for `/to-spec`, planning, or a wayfinder map. Do not start building the thing it describes; that's the next effort's job.
