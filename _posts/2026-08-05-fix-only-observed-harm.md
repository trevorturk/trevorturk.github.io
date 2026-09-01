---
layout: post
title: "Fix Only Observed Harm"
date: 2026-08-05 08:40:00 -0600
summary: "A six-line gate for agents and reviewers: no fix or guard gets built without named evidence, won't-fix with a revive trigger is a real outcome, and a guard's own counter can authorize its deletion."
tags: [ai-agents, workflow, code-review]
---

## The Problem

Late in a long data-quality session on [Hello Weather](https://helloweather.com), the audit stopped on a note that had nothing to do with the code. It is quoted verbatim in the commit that came out of it:

> we're clearly reaching for things to fix, instead of finding actual issues.

Coding agents are unusually good at this failure. Ask one to audit a source adapter and it will find things, because finding things is the visible unit of progress. A field that could theoretically be nil. A value that is semantically impure. A vendor format change that has not happened but could. Each finding arrives with a plausible fix attached, each fix is individually defensible, and the session drifts from investigating observed problems to manufacturing plausible ones.

The drift has a cost. Speculative fixes add branches nothing executes, guards for failures nobody has seen, and churn in recorded outputs that buries the deltas that matter. Some of them make the product worse while making the code "righter."

The countermeasure is six lines across two files.

## The Gate

One paragraph went into `AGENTS.md`, the always-loaded agent instructions, because the gate has to apply when a fix is conceived, not only when it is reviewed:

> Fix only observed harm: before building a fix or guard, name its evidence (a
> fixture value, user report, error event, or support thread) — "semantically
> impure" and "could theoretically break" don't qualify. Won't-fix with a written
> revive trigger is a first-class outcome. If a change makes rendered output read
> worse to a user, technically-correct loses (see the reviewbot skill).

Two rows went into the code-review skill's anti-pattern table, for the cases that slip past the build moment:

| Anti-pattern | Response |
|---|---|
| Technically-correct change that reads worse on screen | Describe what a user sees before/after; if after is worse, revert — nulls where a coherent value stood are the classic case |
| Fix diff dominated by snapshot/cassette churn | Each wire-visible delta must be the point, not a ride-along; enumerate or shrink |

That is the whole artifact. The adjacent rule already existed; the review skill already said "no defensive code for failures not verified to occur." The session audit exposed two gaps around it: nothing gated the pre-build moment, and nothing forced the user's-eye test. These lines close those gaps and nothing else.

The evidence list does the work. A fixture value, a user report, an error event, or a support thread are all things you can point at. "Semantically impure" and "could theoretically break" are things you can only argue about, and an agent will argue for them fluently, because they are always technically true of something.

## Won't-Fix Is an Outcome, Not a Failure

The gate only works if declining to fix something is a respectable way to close a question. Otherwise every audit finding converts to a diff, because a diff is the only artifact that reads as "done." So won't-fix is first-class, with one requirement: a written revive trigger.

An audit of precipitation handling shaped that requirement. A low-share data source occasionally reported phantom hourly snow at coarse mountain grid points, relayed into the accumulation field as-is. A gate was designed: cap the implied snow-to-liquid ratio and null anything above it. Then the evidence test was applied. The defect had appeared only at deliberately hunted edge locations. Never in a fixture, never in a user report, never in an error event. Meanwhile the proposed cap had a concrete cost. Real cold-smoke snowfall runs 35–40:1 ratios, so the guard risked nulling true data to suppress data nobody had complained about.

The outcome was a documented known defect, won't-fix. The audit doc records the defect, the rationale, and the revive trigger: *any user report or fixture evidence, at which point the gate design is written and ready to ship.* That last clause makes won't-fix cheap to reverse. The thinking is parked rather than discarded, with the condition for un-parking it in writing. Nobody has to re-derive the fix or re-litigate the decision. They only have to observe the trigger.

## When Technically Correct Loses

A specific revert produced the gate's last sentence.

A hardening pass over a forecast source touched how day-0 daily rows are built. The vendor splits each day into a day part and a night part. By mid-afternoon the day part expires, and day 0 renders as a "Tonight" row: night icon, tonight's temperatures, night-slot data throughout. The night slot's UV index is 0.

The proposed fix: day 0's UV should be nil once the day part is gone, because 0 is the *night* UV, not the *day's* UV, and presenting it as the day's value is wrong. That is correct. It was reverted in review because of what it does on screen. A coherent "Tonight" row showing UV 0, which is literally true after sunset, becomes a row with a blank where a value stood. The user-visible before/after is strictly worse, and no user had reported the 0. A true full-day UV derived from hourly data would be a real improvement, as a separate, deliberate change rather than a nil smuggled in under "correctness."

That revert became the archetype for the first review-table row. The row's test is mechanical on purpose: describe what a user sees before and after. Not what the type system sees, not what a semantics purist sees. If the after-description is worse, the change loses, whatever its internal virtues.

The same commit banked the other half of the gate. An adversarial reviewer had proposed an alignment guard against a hypothetical vendor format change. Three-plus years of recorded vendor traffic showed the format padding with nulls, never truncating. No observation, no guard: skipped, with the reasoning written down. The whole hardening bundle shipped with its recorded outputs byte-identical to main, which is what the second table row demands. When a diff claims to change nothing user-visible, the snapshots should prove it. When it does change the wire, every delta should be one you can name as the point.

## Let the Instrument Authorize the Deletion

The strongest expression of the gate ran the other direction. It deleted a guard we had already shipped.

The guard's origin is [its own post](/gzip-without-content-encoding/). A vendor intermittently sent compressed bytes mislabeled as plain content, a CDN cached the poisoned responses, and the fix was a magic-byte sniff at the single transport chokepoint: detect the mislabeled body, inflate it under a hard output cap, and count every repair with a dedicated per-host metric. That post noted, almost in passing, that the protective parts leaned speculative and "deleting it later loses nothing else."

Fourteen days after deploy, the counter read zero.

The zero meant something because of how the counter was wired. It fired on *every* engagement of the branch, any time the sniff matched, repaired or not, and its sibling counters on the same telemetry pipeline were firing on every request. The pipeline was demonstrably alive. Zero was not "instrumentation broken" or "nobody looked." The vendor had fixed their origin after the incident, and the defect the guard existed for no longer occurred.

So the guard came out, all of it. The sniff, the inflate branch, the bomb cap, the counter, the extra rescue clause, the two requires, and the three tests that pinned them: 55 lines deleted, one line kept. The removal commit states the revive path the same way the won't-fix doc does. If the failure ever recurs, the generic bad-gateway monitoring surfaces it, which is how it was found the first time, and the deletion PR's own git history is the ready-made patch.

When you do ship a guard, ship it with a counter, and let the counter adjudicate its future. A guard without instrumentation can never be safely deleted, because absence of evidence is indistinguishable from absence of looking. The 2026-07 incident was observed harm, real errors and real users, so building the guard passed the gate cleanly. Two weeks of production silence, measured by an instrument known to be working, was equally hard evidence that the harm had ended. The same standard that authorized the fix authorized the deletion.

## Lessons Learned

- **Put the gate where a fix is conceived, not only where it is reviewed.** Review tables catch what slips through. Always-loaded instructions stop the speculative fix from being written at all.
- **Evidence is something you can point at.** A fixture, a report, an error event. If the justification is an adjective, it does not clear the bar.
- **Won't-fix needs a written revive trigger.** It turns "we ignored it" into "we are waiting for evidence, and the fix is drafted."
- **Judge a change by what a user sees before and after.** Nulls where a coherent value stood are righter in the type system and worse on screen.
- **Ship guards with counters, so their own silence can retire them.** Zero on an instrument known to work is evidence of the same quality that justified the guard.

---

## How This Post Was Made

**Prompt 1:** "see recent work in ~/Code/helloweather, perhaps a blog post about our opus 4.8 agents and why we decided to do that? perhaps something about the swift testing + snapshots inspired by minitest-snapshots? anything else? bring me a list of potential post ideas for review."

**Prompt 2:** "skip 4, 5, 6, 9 but create posts for each of the others in the 1-9 list. also add Four Answers to One Question, and Write the Rule, Not the Story -- show me a concise version of your plan and then I can approve" — then "proceed, one pr per post"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. In this post the hard-wrapped paragraphs were unwrapped, the three example sections no longer open on the same "the Nth sentence" construction, stacked sentences were split, and Lessons Learned shrank from seven bullets to five. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
