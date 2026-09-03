---
layout: post
title: "Adversarial Review Rounds"
date: 2026-07-29 08:00:00 -0600
summary: "A reviewer that knows nothing about the session catches what the session can't. Before deciding to merge a risky change, we send it to two or more read-only reviewer agents, each with a different job and none of our reasoning, and then judge their findings ourselves."
tags: [ai-agents, code-review, workflow]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

In July 2026 we were fixing a run of watch-app bugs on [Hello Weather](https://helloweather.com). A fix would pass the agent's own review, get applied, and then fail. Each time, the diagnosis was wrong or a number in the plan was wrong, and the review had believed the same wrong thing it was supposed to check.

Ask a coding agent to review its own work and it does a decent job. It re-reads the diff, checks the obvious ways things break, and reports back. What it can't do is question where it started. If the session decided two hours ago that the bug was in the caching layer, every later review starts from there. The agent re-reads the cache code carefully and finds nothing wrong with the fix, because the fix is a correct fix for the wrong problem. Numbers go the same way. Once "the label is 128pt wide" is in the session, it gets repeated for the rest of the session and nobody measures it again.

So for review, context is a liability. An agent writes a good change because it knows the problem, the constraints, and the four approaches we already rejected. The same knowledge makes it a bad auditor of that change.

## The Solution

Before we decide to merge a risky change, we send it to two or more separate reviewer agents. They're read-only. Each gets a different job, and none of them gets our reasoning. They send back findings. The main session then marks each finding as confirmed, refuted, or a judgment call, writes the rounds up as one report for the human, and applies the fixes once after the go-ahead.

This is written into all three Hello Weather repos (web, iOS, Android) as an "Adversarial Review Rounds" section of the code-review skill (web keeps it in its `reviewbot` skill), with a pointer from the pull-requests skill at the point where you decide whether to merge. It landed on 2026-07-28 as four bullets. As of September 2026 it reads:

> For changes where a wrong claim is expensive — cross-target invariants, persistence/entitlement, layout/geometry, plan docs about to be implemented — propose independent review rounds before the merge call; dispatch only on the user's go-ahead (policy per Trevor, 2026-07-28). Why: an authoring session cannot review its own blind spots; fresh-context reviewers have repeatedly caught wrong root causes and confidently-wrong numbers that survived multiple informed rounds.
>
> - **Independent and diverse.** 2+ agents in parallel, each with a genuinely different lens (e.g. fresh-eyes with no session context, claims audit of every stated fact, devil's advocate on a named decision, implementer dry-run for plan docs). Diversity beats copies; keep each brief minimal — never include the session's reasoning.
> - **Read-only, report-back-only.** No builds or side effects, nothing posted or edited. Reviewers produce findings, not verdicts: the main session adjudicates each (confirmed / refuted / judgment-call) and digests the rounds into one cohesive analysis. Reviewer briefs are delegable work under AGENTS.md's model-selection rule; the verdict is not.
> - **User checkpoint before any fix.** The digested analysis goes to the user first — no edits, commits, or `gh` mutations (PR comments, PR-body updates, issue filing) from review findings until the user has reviewed the report and approved (recurring failure mode; policy per Trevor, 2026-08-04). Fixes then land once, in the main session.
> - **The digest is a filter, not a to-do list.** Expect noise: reviewers reliably surface unrelated, minor, or stylistic findings. Judge each on importance and scope, and reject freely with a short reasoned objection — "all findings rejected, nothing to change" is a valid and common verdict, not a failed review. Real-but-out-of-scope findings become proposed GitHub issues (`gh-issues` skill), offered in the report, never folded into the PR.
> - **Evidence over assertion.** Findings need a location and a concrete failure scenario; unmeasured claims are labeled estimates. Reviews are sometimes wrong too — re-verify anything load-bearing.
> - **Converge, bounded.** Usually one round; each re-dispatch needs its own user go-ahead, and only after material rework. A round must not automatically produce a batch of changes — one yielding only rejected, minor, or proposed-as-issue findings ends the cycle.

The iOS copy adds a seventh bullet on running the lenses one after another in a tool that has no parallel subagents. The two middle bullets were added on 2026-08-04, after fixes from review findings had been applied before the owner saw the report, more than once. The rest is written as principles rather than steps, so it stays useful as models get better at the mechanics.

## The Lenses

The reviewers have to be different, not just several. Three copies of the same reviewer agree with each other and tell you nothing. We use four lenses, meaning four different jobs to give a reviewer.

**Fresh eyes.** No session context at all. Here's the diff or the plan, here's the repo, tell me what's wrong. This one catches wrong root causes, because it has nothing invested in the diagnosis.

**Claims audit.** Take every factual claim in the change, from the commit message, plan doc, code comments, and PR body, and check each one against the code or a measurement. This one catches confident wrong numbers. On one iOS layout plan, a claims-audit reviewer found two label widths listed as passing that were under the raw budget but over the plan's own 128pt bar. In the same round, the implementer dry-run added that the redrafted strings couldn't be measured at all. So the plan shipped with a real measurement script (`tools/measure-stat-widths.swift`) instead of remembered numbers. That script has since been folded into the width report tests.

**Devil's advocate on a named decision.** Not "review this change" but "we decided X; argue that X is wrong." Naming the decision matters. A general "argue against this" prompt gets you general skepticism, which is noise. A specific one gets you a real counter-case or an honest concession.

**Implementer dry-run.** This is for plan documents about to be handed to another agent. Pretend you're implementing this step by step, and report everywhere the plan can't be built, is ambiguous, or is wrong about the code it points at. It catches plans that are right in concept and useless in practice. A round against a widget plan found that every widget time label took its timezone from the app's stored weather data, so a widget pinned to another city would show the app's clock. An earlier shelved implementation had the same defect. The plan had to gain a step: render times in the pinned location's timezone at the production formatting sites.

## Independence Is the Whole Trick

The mistake that quietly kills this practice is contaminating the brief. It's tempting to write:

> "We think the bug is in the cache invalidation path and we fixed it by clearing on write. Please review."

That hands the reviewer the conclusion you needed it to challenge, and you get back a careful review of the cache invalidation path.

The skill's rule is blunt: *keep each brief minimal - never include the session's reasoning.* A good brief names the artifact and the lens, and stops. If the reviewer needs to know something, it can read the repo. If it had to go looking, that tells you something about whether the change explains itself.

Reviewers also have to be read-only. No builds, no edits, nothing posted to the PR. Partly that's safety, because you don't want four agents racing to fix the same file. Partly it's about roles. A reviewer that can edit starts solving instead of reporting, and you lose the finding.

A usable brief template, in full:

```
You are a code reviewer with no prior context on this work.

Artifact: the diff on branch `fix-widget-cache` (git diff main...HEAD)
Lens: fresh eyes. Read the diff and whatever repo files you need.

Report findings only. Do not edit files, do not build, do not run
anything with side effects, do not post anywhere.

For each finding give:
  - file:line
  - a concrete failure scenario (inputs, timing, or state that
    triggers it), not a style objection
  - severity: blocker / important / nit

If a claim you make is not verified against the code, label it an
estimate. If you find nothing above nit level, say so.
```

Swap the `Lens:` line for the other three and you have the round. What's missing matters as much as what's there: the bug we thought we were fixing, the approaches we rejected, and any sentence beginning "we believe."

## The Adjudication Step

Reviewers send back findings, not verdicts. Ask an agent for a verdict and it'll give you one, confidently, and you'll have no way to weigh it against the three other confident verdicts you just got. Ask for findings and you get a list of specific claims with locations, each of which you can check.

The main session now has the context as an asset rather than a liability. It goes through every finding and marks it:

- **Confirmed** - real, fix it
- **Refuted** - the reviewer was wrong, and *why* it was wrong gets written down
- **Judgment call** - real trade-off, escalate to the human or record the decision

The report goes to the human before anything changes. Since 2026-08-04 the skill forbids any edit, commit, or PR comment based on review findings until the owner has read the report and approved. Then the session applies all the fixes in one pass, not four rounds of patching as reports come in.

The refuted bucket isn't wasted work. Reviewers are sometimes wrong, and knowing exactly how a reasonable reviewer misread your code often means the code needs a comment or a clearer name. The step also runs the other way. On our web repo, two separate checks killed a proposed fix for provider-cache behavior with live evidence. A sweep of 133 coordinate pairs, chosen to break the fix, found 10.8% of them changing timezone name. The commit that recorded it, "Refute the coordinate-rounding fix with live evidence," was worth more than the fix would have been.

One note on cost. Running the lenses is bounded, well-specified work you can hand to a cheaper model. Judging conflicting findings against a codebase you understand is not. Keep the verdict on the most capable model you have, in the session that holds the context.

## What It Caught

### A revenue bug on Android

A billing fix made `HomeViewModel` reconcile entitlement (whether the user gets paid features) for paying members as well as free users. That was correct on its own. A separate reviewer noticed it also made a `clearAll()` path in `PaymentProcessor.fetchInAppPurchases`, previously unreachable, reachable for active subscribers, and gave two concrete ways it fails:

1. `queryPurchasesAsync` can briefly return OK with an *empty* list (after a reconnect, while the Play Store is updating, or during propagation lag). An active subscriber opening Home in that window gets their prefs wiped and the widget flipped to the upsell. That's the same symptom the fix shipped to remove, back through a different door.
2. Grandfathered one-time purchase SKUs may have been *consumed* by very old app versions, so the purchase query never returns them. Those lifetime members would be wiped on every single Home open.

Neither was hypothetical, and neither had come up in informed review, because the informed session was thinking about the entitlement bug it had just fixed. The resulting commit is titled "Never revoke entitlement on empty purchase-query results" and states the rule directly:

> Rule: entitlement is never revoked on the ABSENCE of purchases, only on positive signal or natural lease expiry.

Three unit tests cover empty-while-subscribed, empty-for-lifetime, and the real lapsed-subscription cleanup path. The fix is on Android's paused Release 1 branch, whose PR is still open as of September 2026.

### Wrong root causes on iOS

We wrote the practice down because of the watch-app run above, where reviewers with no session context kept finding that the diagnosis was wrong, not the patch. One typical round against a location-caching change used a reviewbot lens, a cold-read lens with no session context, and a plan-conformance check, and confirmed three findings in one pass. Location identity was compared with `==` on raw doubles, so the slot factory never matched and the forecast dictionary could crash on duplicate keys. A travel-distance guard sat on the fetch decision rather than the render path, so a stale record for the previous city could still be *displayed*. And a cache wipe hung off a reset method that location selection also called, so tapping a row wiped the cache the screen had just built.

All three had passed informed review. None got past a reviewer that had never heard the story.

### A quieter one on web

A debug flag added for a one-time production investigation got a separate review and came back with two findings. The flag's truthiness check meant setting it to `"false"` would silently *enable* it. And reading a non-2xx response body could raise on a truncated vendor response, turning a handled 502 into an unhandled 500 during exactly the window the flag existed to observe. Small change, small review, two real bugs.

## The Failure Case

Review rounds are evidence, not proof. The counter-example is recorded in our web repo's `plans/sentry-fiber-isolation.md`, retired on 2026-08-06 when the upstream fix shipped.

Our async server runs all in-flight requests as fibers on one thread. They were sharing one error-reporting hub, so tags from one request leaked into another's error reports. We had a patch to give each fiber its own scope. Version 2 of that patch **passed a full review round while containing a blocker.** Ruby's fiber storage is inherited by `Thread.new` *as the same object*, so the background-job thread pool would have shared a single hub. Running it confirmed that: four concurrent jobs, one hub, jobs reading each other's tags. A separate counter would have kept one request's hub for the life of the process.

The reviewers were competent and the round was run properly, and the bug went through. Running the code caught it.

If "it passed two review rounds" became a merge criterion, that would be a new and more expensive kind of false confidence. Review rounds raise the floor. They don't replace running the code, and they don't replace the human on the merge call. The plan recorded what we chose, which was not to ship the patch and to wait for the upstream fix, and named the blocker the review missed as the deciding evidence. That fix arrived in sentry-ruby 6.7.0 on 2026-08-06, with our `Thread.new` report folded in, and the plan closed with it.

## Dogfooding

The pull requests that added this skill to all three repos got the same treatment, with two lenses: fresh eyes on the skill writing and devil's advocate. The reviewers found real problems in the first cut:

- An **unverifiable dated anecdote** used to justify the section. It sounded authoritative and couldn't be checked, so it came out.
- A **citation to a skill that didn't exist.** The Android text pointed at a `reviewbot` skill that only the iOS and web repos have. It was copy-pasted across repos, and a reviewer that actually went looking for the file caught it.
- A **transcript quote** that added length without adding a rule.
- **Ambiguity about who initiates.** The first draft didn't say whether the agent dispatches reviewers on its own. It now proposes, and dispatches only on the user's go-ahead.
- **Vagueness about when.** "Risky or tricky changes" became a concrete list of triggers: cross-target invariants, persistence/entitlement, layout/geometry, plan docs about to be implemented.
- **No stopping rule.** "Iterate to convergence" became "usually one round; stop when a round returns only nits." The current wording is stricter: a round that yields only rejected, minor, or proposed-as-issue findings ends the cycle.

The authoring session had read every one of those a dozen times without seeing them.

## Lessons Learned

- **Name the decision you want attacked.** "Argue against this" returns general skepticism. "We decided X; argue X is wrong" returns a counter-case or a concession.
- **Write down why a refuted finding was wrong.** If a reasonable reviewer misread your code, the code probably needs a comment or a clearer name.
- **Delegate the lenses, keep the verdict.** Reviewer briefs are bounded work for a cheaper model. Judging the findings needs the context and the most capable model you have.
- **Propose the round; don't run it by default.** Parallel reviewers cost real time and tokens. Let the human decide it's worth it, and stop when a round returns only nits.
- **A passed round is evidence, not proof.** Ours passed a full round with a blocker still in it. Still run the code.
