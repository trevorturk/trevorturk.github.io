---
layout: post
title: "Fix Only Observed Harm"
date: 2026-08-05 08:40:00 -0600
summary: "A six-line rule for agents and reviewers: don't build a fix or a guard without evidence you can point at, write down what would reopen a won't-fix, and let a guard's own counter decide when it comes out."
tags: [ai-agents, workflow, code-review]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Late in a long data-quality session on [Hello Weather](https://helloweather.com), the audit stopped on a note that had nothing to do with the code. It's quoted in the commit that came out of it:

> we're clearly reaching for things to fix, instead of finding actual issues.

Coding agents fail this way often. Ask one to audit a data-source adapter and it will find things, because finding things looks like progress. A field that could be nil in theory. A value that isn't quite what its name says. A vendor format change that hasn't happened but could. Each finding comes with a plausible fix, each fix is defensible on its own, and the session drifts from investigating problems we've seen to inventing problems we might see.

That drift costs something. Speculative fixes add branches nothing runs, guards for failures nobody has hit, and churn in recorded outputs that hides the changes that matter. Some of them make the product worse while making the code "righter."

Our answer is six lines across two files.

## The Gate

One paragraph went into `AGENTS.md`, the instructions every agent session loads. It has to live there because the gate needs to apply when a fix is first thought of, not only when it's reviewed:

> Fix only observed harm: before building a fix or guard, name its evidence (a
> fixture value, user report, Sentry event, or support thread) — "semantically
> impure" and "could theoretically break" don't qualify. Won't-fix with a written
> revive trigger is a first-class outcome. If a change makes rendered output read
> worse to a user, technically-correct loses (see the reviewbot skill).

Two rows went into the code-review skill's anti-pattern table, for the fixes that get past that first moment:

| Anti-pattern | Response |
|---|---|
| Technically-correct change that reads worse on screen | Describe what a user sees before/after; if after is worse, revert — nulls where a coherent value stood are the classic case |
| Fix diff dominated by snapshot/cassette churn | Each wire-visible delta must be the point, not a ride-along; enumerate or shrink |

That's all of it. The rules around it already existed. The review skill already said "do not hide real bugs behind defensive code" and flagged any `rescue` "for a failure mode not verified to occur." The session audit found two gaps next to those rules: nothing stopped a speculative fix before it was built, and nothing asked what a user would see. These lines close those two gaps.

The evidence list does the work. Everything on it is something you can point at. "Semantically impure" and "could theoretically break" are things you can only argue about, and an agent will argue for them well, because they're always true of something.

## Won't-Fix Is an Outcome, Not a Failure

The gate only works if declining to fix something counts as finishing. Otherwise every audit finding turns into a diff, because a diff is the only thing that reads as "done." So won't-fix is a real outcome, with one requirement: we write down what would reopen it.

An audit of precipitation handling shaped that requirement. One of our smaller data sources occasionally reported an hour of snow that hadn't fallen at coarse mountain grid points, and we passed it into the accumulation field as-is. We designed a gate: cap the implied snow-to-liquid ratio and null anything above it. Then we applied the evidence test. The defect had only shown up at edge locations we went looking for. It had never appeared in a fixture, a user report, or an error event. Meanwhile the cap had a real cost. Dry, powdery snow runs at 35–40:1 ratios, so the guard could null true data to suppress data nobody had complained about.

The outcome was a documented known defect, marked won't-fix. The entry, which has since moved from the audit plan into the weather-sources skill, records the defect, the reason, and the revive trigger: *any user report or fixture evidence, at which point the gate design is written and ready to ship.* That last clause makes the won't-fix cheap to reverse. The design is parked, not thrown away, and the condition for un-parking it is written down. Nobody has to work out the fix again or reargue the decision. They only have to see the trigger.

## When Technically Correct Loses

The gate's last sentence came from one revert.

A hardening pass over a forecast source touched how we build today's daily row. The vendor splits each day into a day part and a night part. By mid-afternoon the day part has expired, and today renders as a "Tonight" row: night icon, tonight's temperatures, night data throughout. The night slot's UV index is 0.

The proposed fix: once the day part is gone, today's UV should be nil, because 0 is the *night* UV, not the *day's* UV, and showing it as the day's value is wrong. That's true. We reverted it in review because of what it does on screen. A "Tonight" row showing UV 0, which is true after sunset, becomes a row with a blank where a value was. What the user sees is worse, and no user had reported the 0. A real full-day UV worked out from the hourly data would be an improvement, as its own deliberate change, not a nil slipped in under "correctness."

That revert became the model for the first review-table row. The row's test is meant to be mechanical: describe what a user sees before and after. Not what the type system sees, and not what a purist sees. If the after is worse, the change loses, whatever else is good about it.

The same commit recorded the other half of the gate. A second reviewer had proposed a guard against a vendor format change that hadn't happened. Three-plus years of recorded vendor traffic showed the format padding with nulls, never truncating. Nothing observed, so no guard, and the reasoning went in the commit. The whole hardening pass shipped with its recorded outputs the same bytes as on main, which is what the second table row asks for. When a diff claims to change nothing a user sees, the snapshots should prove it. When it does change what goes over the wire, every difference should be one you can name as the point.

## Let the Instrument Authorize the Deletion

The strongest use of the gate ran the other way. It deleted a guard we had already shipped.

The guard has [its own post](/gzip-without-content-encoding/). A vendor sometimes sent compressed bytes labeled as plain content, a CDN cached the bad responses, and the fix was to check the first bytes of every body at the one place all vendor traffic passes through: detect the mislabeled body, decompress it under a hard size limit, and count every repair with its own per-host metric. That post said, almost in passing, that the protective parts were speculative and "deleting it later loses nothing else."

Fourteen days after deploy, the counter read zero.

The zero meant something because of how the counter was wired. It fired every time the check matched, whether or not a repair followed, and its sibling counters on the same telemetry pipeline were firing on every request. So the pipeline was alive. Zero didn't mean "instrumentation broken" or "nobody looked." The vendor had fixed their origin after the incident, and the problem the guard existed for had stopped happening.

So the guard came out, all of it. The check, the decompress branch, the size limit, the counter, the extra rescue clause, the two requires, and the three tests that pinned them: 55 lines deleted, one line kept. The removal commit states the revive path the same way the won't-fix record does. If the failure ever comes back, the generic bad-gateway monitoring will catch it, which is how we found it the first time, and the deletion PR's own git history is the ready-made patch.

When you do ship a guard, ship it with a counter, and let the counter decide its future. A guard without a counter can never be safely deleted, because you can't tell "it never happened" from "nobody looked." The 2026-07 incident was observed harm, real errors and real users, so building the guard passed the gate. Two weeks of silence from an instrument we knew was working was evidence of the same kind that the harm had ended.

## Lessons Learned

- **Put the gate where a fix is first thought of, not only where it's reviewed.** Review tables catch what slips through. Always-loaded instructions stop the speculative fix from being written at all.
- **Evidence is something you can point at.** If the reason for a fix is an adjective, it doesn't count.
- **Won't-fix needs a written revive trigger.** It turns "we ignored it" into "we're waiting for evidence, and the fix is drafted."
- **Judge a change by what a user sees before and after.** A null where a value stood is righter in the type system and worse on screen.
- **Ship guards with counters, so their own silence can retire them.** Zero on an instrument you know works is evidence as good as the evidence that justified the guard.
