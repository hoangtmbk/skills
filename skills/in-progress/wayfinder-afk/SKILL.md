---
name: wayfinder-afk
description: Work a wayfinder map AFK — expert panels resolve the grilling tickets; preference calls are parked for the human instead of blocking.
disable-model-invocation: true
---

`/wayfinder` with one substitution: grilling tickets resolve through `/grill-the-experts` instead of `/grilling` and `/domain-modeling`, so a session can work the map with no human in the loop. Everything else — the map, tickets, fog of war, claiming, one ticket per session — is unchanged, and the map format is identical: a map charted by `/wayfinder` can be worked by this skill and vice versa, session by session.

Only the **work through the map** mode exists here. Charting has no AFK version — naming the destination is grilling the human — so invoke this skill with a map, and chart with `/wayfinder`.

## The rubric is the map header

`/grill-the-experts` refuses to fan out without a rubric. Here the rubric already exists on the map: **Destination** plus **Notes**. Skip that skill's opening `/grilling` pass and read the rubric off the map instead.

That makes Notes load-bearing: it must carry the user's standing preferences ("bias to boring tech", risk appetite), not just skill pointers. If Notes is too thin to adjudicate against, that thinness is itself a preference call — park it (below) rather than inventing preferences.

## Grilling tickets go AFK

The grilling ticket type flips from HITL to AFK: the panel answers, the session adjudicates against the rubric. Every other type keeps its `/wayfinder` classification — a prototype still needs the human to react to it, so prototype and HITL task tickets stay off this skill's menu.

Choosing a ticket, take the first frontier ticket that is AFK-workable — grilling, research, or an AFK task — skipping parked and HITL tickets. If the frontier holds only HITL tickets, stop and say so: the map is waiting on its human.

## The resolution comment is the decisions entry

No separate decisions document — on a map, each decision already lives in exactly one place, its ticket. Write the ticket's resolution comment in `/grill-the-experts`' entry format (Decided by / Answer / Evidence / Dissent & reversal conditions), then index it in Decisions-so-far as usual.

## Preference calls park, never block

`/grill-the-experts` ends its escalation chain at the user; AFK, there is no user. When a question classifies as preference — or survives adjudication with `/ask-codex` still contested — **park it**: post the panel's recommendation and the dissent as a comment, unclaim the ticket, and mark it parked in the tracker's terms (a `wayfinder:parked` label where labels exist; a `PARKED` line atop the body otherwise). Never answer a preference call yourself.

A parked ticket waits for a `/wayfinder` session with the human, who resolves it — the park comment's recommendation as the opening question — and clears the mark. If parking claims the session's chosen ticket, the session ends there: parking is a legitimate session result.
