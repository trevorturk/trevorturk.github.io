---
layout: post
title: "Fix Only Observed Harm"
date: 2026-08-05 08:40:00 -0600
summary: "A six-line gate for agents and reviewers: no fix or guard gets built without named evidence, won't-fix with a revive trigger is a real outcome, and a guard's own counter can authorize its deletion."
tags: [ai-agents, workflow, code-review]
---

## The Problem

Late in a long data-quality session on [Hello Weather](https://helloweather.com), the
audit hit a wall that had nothing to do with the code. The calibration note, quoted
verbatim in the commit that came out of it:

> we're clearly reaching for things to fix, instead of finding actual issues.

This is a failure mode coding agents are unusually good at. Ask an agent to audit a
source adapter and it will find things — it always finds things, because "finding
things" is the visible unit of progress. A field that could theoretically be nil. A
value that's semantically impure. A vendor format change that hasn't happened but
could. Each finding arrives with a plausible fix attached, each fix is individually
defensible, and the session drifts from *investigating observed problems* to
*manufacturing plausible ones*.

The output of that drift isn't neutral. Speculative fixes add branches nothing
executes, guards for failures nobody has seen, and churn in recorded outputs that
buries the deltas that matter. Worse, some of them make the product worse while making
the code "righter."

The countermeasure we shipped is six lines across two files.

## The Gate

The first piece went into `AGENTS.md` — the always-loaded agent instructions — because
the gate has to apply at the moment a fix is *conceived*, not just when it's reviewed:

> Fix only observed harm: before building a fix or guard, name its evidence (a
> fixture value, user report, error event, or support thread) — "semantically
> impure" and "could theoretically break" don't qualify. Won't-fix with a written
> revive trigger is a first-class outcome. If a change makes rendered output read
> worse to a user, technically-correct loses (see the reviewbot skill).

The second piece is two rows in the code-review skill's anti-pattern table, for the
cases that slip past the build moment:

| Anti-pattern | Response |
|---|---|
| Technically-correct change that reads worse on screen | Describe what a user sees before/after; if after is worse, revert — nulls where a coherent value stood are the classic case |
| Fix diff dominated by snapshot/cassette churn | Each wire-visible delta must be the point, not a ride-along; enumerate or shrink |

That's the whole artifact. It's deliberately small because the adjacent rules already
existed — the review skill already said "no defensive code for failures not verified to
occur." What the session audit exposed were the two gaps around that rule: nothing
gated the *pre-build* moment, and nothing forced the *user's-eye* test. These lines
close exactly those gaps and nothing else.

The evidence list is the load-bearing part. "A fixture value, user report, error event,
or support thread" are all things you can point at. "Semantically impure" and "could
theoretically break" are things you can only argue about — and an agent will argue for
them fluently, because they're always technically true of something.

## Won't-Fix Is an Outcome, Not a Failure

The gate only works if declining to fix something is a respectable way to close a
question. Otherwise every audit finding converts to a diff, because a diff is the only
artifact that reads as "done."

So the second sentence makes won't-fix first-class — with one requirement: a written
revive trigger.

The instance that shaped it: an audit of precipitation handling found a low-share data
source occasionally reporting phantom hourly snow at coarse mountain grid points,
relayed into the accumulation field as-is. A gate was designed — cap the implied
snow-to-liquid ratio and null anything above it. Then the evidence test was applied.
The defect had appeared only at deliberately hunted edge locations. Never in a fixture.
Never in a user report. Never in an error event. Meanwhile the proposed cap had a
concrete cost: real cold-smoke snowfall runs 35–40:1 ratios, so the guard risked
nulling true data to suppress data nobody had complained about.

The outcome was a documented known defect, won't-fix — recorded in the audit doc with
the defect, the rationale, and the revive trigger: *any user report or fixture
evidence, at which point the gate design is written and ready to ship.* That last part
is what makes won't-fix cheap to reverse. The thinking isn't discarded; it's parked,
with the condition for un-parking it in writing. Nobody has to re-derive the fix or
re-litigate the decision — they just have to observe the trigger.

## When Technically Correct Loses

The third sentence of the gate exists because of a specific revert.

A hardening pass over a forecast source touched how day-0 daily rows are built. The
vendor splits each day into a day part and a night part; by mid-afternoon the day part
expires, and day 0 renders as a "Tonight" row — night icon, tonight's temperatures,
night-slot data throughout. The night slot's UV index is 0.

The proposed fix: day 0's UV should be nil once the day part is gone, because 0 is the
*night* UV, not the *day's* UV, and presenting it as the day's value is wrong. That's
correct. And it was reverted in review, because of what it does on screen: a coherent
"Tonight" row showing UV 0 — which is literally true after sunset — becomes a row with
a blank where a value stood. The user-visible before/after is strictly worse, and no
user had reported the 0. (A true full-day UV derived from hourly data would be a real
improvement — as a separate, deliberate change, not a nil smuggled in under
"correctness.")

That revert became the archetype for the first review-table row. The row's test is
mechanical on purpose: *describe what a user sees before and after.* Not what the type
system sees, not what a semantics purist sees. If the after-description is worse, the
change loses, whatever its internal virtues.

The same commit banked the other half of the gate too. An adversarial reviewer had
proposed an alignment guard against a hypothetical vendor format change. Three-plus
years of recorded vendor traffic showed the format padding with nulls, never
truncating. No observation, no guard — skipped, with the reasoning written down. And
the whole hardening bundle shipped with its recorded outputs byte-identical to main,
which is what the second table row demands: when your diff claims to change nothing
user-visible, the snapshots should prove it, and when it does change the wire, every
delta should be one you can name as the point.

## Let the Instrument Authorize the Deletion

The strongest expression of the gate ran in the other direction: it deleted a guard
we'd already shipped.

The guard's origin story is [its own post](/gzip-without-content-encoding/): a vendor
intermittently sent compressed bytes mislabeled as plain content, a CDN cached the
poisoned responses, and the fix was a magic-byte sniff at the single transport
chokepoint — detect the mislabeled body, inflate it under a hard output cap, and count
every repair with a dedicated per-host metric. That post noted, almost in passing, that
the protective parts leaned speculative and "deleting it later loses nothing else."

Fourteen days after deploy, the counter read zero.

That zero was meaningful for a reason worth spelling out: the counter fired on *every*
engagement of the branch — any time the sniff matched, repaired or not — and its
sibling counters on the same telemetry pipeline were firing on every request. The
pipeline was demonstrably alive. Zero wasn't "instrumentation broken" or "nobody
looked." Zero was zero. The vendor had fixed their origin after the incident, and the
defect the guard existed for no longer occurred.

So the guard came out — all of it. The sniff, the inflate branch, the bomb cap, the
counter, the extra rescue clause, the two requires, and the three tests that pinned
them: 55 lines deleted, one line kept. The removal commit states the revive path the
same way the won't-fix doc does: if the failure ever recurs, the generic bad-gateway
monitoring surfaces it — that's how it was found the first time — and the deletion PR's
own git history is the ready-made patch.

The pattern generalizes: **when you do ship a guard, ship it with a counter, and let
the counter adjudicate its future.** A guard without instrumentation can never be
safely deleted, because absence of evidence is indistinguishable from absence of
looking. A guard *with* instrumentation carries its own expiration test. The
2026-07 incident was observed harm — real errors, real users — so building the guard
passed the gate cleanly. Two weeks of production silence, measured by an instrument
known to be working, was equally hard evidence that the harm had ended. The same
standard that authorized the fix authorized the deletion.

## Lessons Learned

- **Agents reach for fixes because findings look like progress.** The drift from
  "investigate observed problems" to "manufacture plausible ones" is gradual and
  well-intentioned. It needs a named gate, not vigilance.
- **Evidence must be pointable.** A fixture value, a user report, an error event, a
  support thread. If the justification is an adjective — impure, fragile,
  theoretically breakable — it doesn't clear the bar.
- **Won't-fix needs a written revive trigger to be first-class.** The trigger converts
  "we ignored it" into "we're waiting for evidence, and the fix is drafted." Cheap to
  park, cheap to revive, nothing re-litigated.
- **Adjudicate changes by the rendered before/after, not the semantic one.** Nulls
  where a coherent value stood is the recurring case: righter in the type system, worse
  on the screen. The screen wins.
- **Every wire-visible delta must be the point.** A fix diff dominated by
  snapshot churn is hiding either accidental changes or unearned ones. Byte-identical
  snapshots are the strongest claim a refactor can make.
- **Instrument guards so their own telemetry can retire them.** Zero events on a
  counter that demonstrably works — sibling counters firing on every request — is
  evidence of the same quality that justified the guard. Fixes get built on observed
  harm; deletions get authorized by observed silence.
- **Put the gate where code gets conceived, not just where it gets reviewed.** Review
  tables catch what slips through; the always-loaded instructions stop the speculative
  fix from being written at all.

---

## How This Post Was Made

**Prompt 1:** "see recent work in ~/Code/helloweather, perhaps a blog post about our opus 4.8 agents and why we decided to do that? perhaps something about the swift testing + snapshots inspired by minitest-snapshots? anything else? bring me a list of potential post ideas for review."

**Prompt 2:** "skip 4, 5, 6, 9 but create posts for each of the others in the 1-9 list. also add Four Answers to One Question, and Write the Rule, Not the Story -- show me a concise version of your plan and then I can approve" — then "proceed, one pr per post"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.
