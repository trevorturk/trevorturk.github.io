---
layout: post
title: "Adversarial Review Rounds"
date: 2026-07-29 08:00:00 -0600
summary: "Fresh-context reviewers catch what informed review can't. Before the merge call, dispatch two or more read-only reviewer agents with deliberately different lenses and no session context, then adjudicate their findings yourself."
tags: [ai-agents, code-review, workflow]
---

## The Problem

During a run of watch-app bug fixes on [Hello Weather](https://helloweather.com) in July 2026, fixes kept passing an informed self-review, getting applied, and then failing. Each time, the root cause was wrong or a number in the plan was wrong, and the review had inherited the same wrong belief it was supposed to be checking.

Ask a coding agent to review its own work and it does a competent job. It re-reads the diff, checks the obvious failure modes, and reports back. What it cannot do is question the premise it started from. If the session concluded two hours ago that the bug was in the caching layer, every later review runs downstream of that conclusion. The agent re-reads the cache code carefully and finds nothing wrong with the fix, because the fix is a correct fix for the wrong problem. Numbers behave the same way. Once "the label is 128pt wide" lands in the session's context, it gets cited confidently for the rest of the session, and nobody re-measures.

So context is a liability for review. Full knowledge of the problem, the constraints, and the four approaches already rejected is what makes an agent good at writing the change. It is exactly what makes the same agent bad at auditing it.

## The Solution

Before the merge call on a risky change, dispatch two or more independent, read-only reviewer agents. Each gets a deliberately different lens and none of the authoring session's reasoning. They return findings. The main session adjudicates each finding as confirmed, refuted, or a judgment call, and applies the fixes once.

This is codified across all three Hello Weather repos (web, iOS, Android) as an "Adversarial Review Rounds" section in the code-review skill, with a pointer from the pull-requests skill at the merge decision point. The whole thing is four bullets:

> For changes where a wrong claim is expensive — cross-target invariants, persistence/entitlement, layout/geometry, plan docs about to be implemented — propose independent review rounds before the merge call; dispatch only on the user's go-ahead. Why: an authoring session cannot review its own blind spots; fresh-context reviewers have repeatedly caught wrong root causes and confidently-wrong numbers that survived multiple informed rounds.
>
> - **Independent and diverse.** 2+ agents in parallel, each with a genuinely different lens (e.g. fresh-eyes with no session context, claims audit of every stated fact, devil's advocate on a named decision, implementer dry-run for plan docs). Diversity beats copies; keep each brief minimal — never include the session's reasoning.
> - **Read-only, report-back-only.** No builds or side effects, nothing posted or edited. Reviewers produce findings, not verdicts: the main session adjudicates each (confirmed / refuted / judgment-call) and applies fixes once. Reviewer briefs are delegable work under AGENTS.md's model-selection rule; the verdict is not.
> - **Evidence over assertion.** Findings need a location and a concrete failure scenario; unmeasured claims are labeled estimates. Reviews are sometimes wrong too — re-verify anything load-bearing.
> - **Converge, bounded.** Usually one round; re-dispatch fresh eyes only after material rework, and stop when a round returns only nits.

It is written as principles rather than procedure, so it stays useful as models get better at the mechanics.

## The Lenses

Reviewers must be different, not merely multiple. Three copies of the same reviewer agree with each other and tell you nothing. We use four lenses.

**Fresh eyes.** Zero session context. Here is the diff or the plan doc, here is the repo, tell me what's wrong. This lens catches wrong root causes, because it has no investment in the diagnosis.

**Claims audit.** Take every factual assertion in the change, across the commit message, plan doc, code comments, and PR body, and verify each one against the code or a measurement. This lens catches confidently-wrong numbers. On one iOS layout plan, a claims-audit reviewer found that several width claims were unverifiable as written. That forced us to commit an actual measurement harness (`tools/measure-stat-widths.swift`) instead of quoting remembered numbers, and two more label widths turned out to be over the bar once measured.

**Devil's advocate on a named decision.** Not "review this change" but "we decided X; argue that X is wrong." Naming the decision matters. A general adversarial prompt produces general skepticism, which is noise. A specific one produces a real counter-case or an honest concession.

**Implementer dry-run.** For plan documents about to be handed to another agent: pretend you are implementing this step by step, and report everywhere the plan is unbuildable, ambiguous, or wrong about the code it references. This catches plans that are conceptually right and operationally useless. One round against a widget plan turned up that time labels resolve their timezone from a stored payload. The whole pinned-widget feature therefore needed location-local rendering threaded through the production formatting sites, a step the plan did not have at all.

## Independence Is the Whole Trick

The failure mode that quietly kills this practice is contaminating the brief. It is tempting to write:

> "We think the bug is in the cache invalidation path and we fixed it by clearing on write. Please review."

That hands the reviewer the conclusion you needed it to challenge, and you get back a careful review of the cache invalidation path.

The rule in the skill is blunt: *keep each brief minimal - never include the session's reasoning.* A good brief names the artifact and the lens, and stops. If the reviewer needs to know something, it can read the repo. The fact that it had to go find it is itself signal about whether the change is self-explanatory.

The corollary is that reviewers must be read-only. No builds, no edits, nothing posted to the PR. Partly this is safety, since you do not want four agents racing to fix the same file. Partly it is role clarity: a reviewer that can edit starts solving instead of reporting, and you lose the finding.

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

Swap the `Lens:` line for the other three and you have the round. What is absent matters as much as what is there: the bug we thought we were fixing, the approaches we rejected, and any sentence beginning "we believe."

## The Adjudication Step

Reviewers produce findings, not verdicts. An agent asked for a verdict will produce one, and it will be confident, and you will have no way to weigh it against the three other confident verdicts you just got. An agent asked for findings produces a list of specific claims with locations, each of which you can check.

The main session, whose context is now an asset rather than a liability, goes through every finding and marks it:

- **Confirmed** - real, fix it
- **Refuted** - the reviewer was wrong, and *why* it was wrong gets written down
- **Judgment call** - real trade-off, escalate to the human or record the decision

Then it applies all the fixes in a single pass, not four rounds of patching as reports trickle in.

The refuted bucket is not wasted work. Reviews are sometimes wrong, and knowing exactly how a plausible reviewer misread your code often means the code needs a comment or a clearer name. The step also runs in the other direction. On our web repo, a proposed fix for provider-cache behavior was killed by two independent checks that refuted it with live evidence: a 133-pair sweep found 10.8% of adversarial coordinate pairs changing timezone name. The commit that recorded the refutation, "Refute the coordinate-rounding fix with live evidence," was worth more than the fix would have been.

One note on cost. Running the lenses is bounded, well-specified work you can hand to a cheaper model. Adjudicating conflicting findings against a codebase you understand is not. Keep the verdict on the most capable model you have, in the session that holds the context.

## What It Caught

### A revenue bug on Android

A billing fix made `HomeViewModel` reconcile entitlement for paying members as well as free users, which was correct in itself. An adversarial reviewer noticed that this made a previously-unreachable `clearAll()` path in `PaymentProcessor.fetchInAppPurchases` newly reachable for active subscribers, and produced two concrete failure scenarios:

1. `queryPurchasesAsync` can transiently return OK with an *empty* list (post-reconnect, Play Store updating, propagation lag). An active subscriber opening Home during that window gets their prefs wiped and the widget flipped to the upsell, reintroducing through a different trigger the exact symptom the fix was shipped to remove.
2. Grandfathered one-time purchase SKUs may have been *consumed* by ancient app versions, so the purchase query never returns them. Those lifetime members would be deterministically wiped on every single Home open.

Neither was hypothetical, and neither had surfaced in informed review, because the informed session was thinking about the entitlement bug it had just fixed. The resulting commit is titled "Never revoke entitlement on empty purchase-query results" and states the invariant directly:

> Rule: entitlement is never revoked on the ABSENCE of purchases, only on positive signal or natural lease expiry.

Three unit tests now pin empty-while-subscribed, empty-for-lifetime, and the genuinely-lapsed cleanup path.

### Wrong root causes on iOS

The practice was codified because of the watch-app run above, where fresh-context reviewers kept finding that the diagnosis was wrong, not the patch. One representative round against a location-caching change, with a reviewbot lens, a cold adversarial lens, and an implementer dry-run, produced three confirmed findings in a single pass. Location identity was compared with `==` on raw doubles, so the slot factory never matched and the forecast dictionary could trap on duplicate keys. A travel-distance guard sat on the fetch decision rather than the render path, so a stale record for the previous city could still be *displayed*. And a cache wipe hung off a reset method also called by location selection, so tapping a row wiped the cache the screen had just built.

All three survived informed review. None survived a reviewer that had never heard the story.

### A quieter one on web

A debug flag added for a one-time production investigation got an adversarial pass and came back with two findings. The flag's truthiness check meant setting it to `"false"` would silently *enable* it. And reading a non-2xx response body could raise on a truncated vendor response, turning a handled 502 into an unhandled 500 during exactly the diagnostic window the flag existed to observe. Small change, small review, two real bugs.

## The Failure Case

Review rounds are evidence, not proof. The counter-example lives in our web repo's `plans/sentry-fiber-isolation.md`.

Our async server runs all in-flight requests as fibers on one thread, and they were sharing one error-reporting hub, cross-contaminating tags between requests. We had a patch to isolate the scope per fiber. Version 2 of that patch **passed a full adversarial review round while containing a blocker.** Ruby's fiber storage is inherited by `Thread.new` *as the same object*, so the background-job worker thread pool would have shared a single hub. Execution later confirmed it: four concurrent jobs, one hub, jobs reading each other's tags. A separate counter would have adopted one request's hub for the lifetime of the process.

The reviewers were competent, the round was properly run, and the bug went through. It got caught by running the thing.

Treating "it passed two adversarial rounds" as a merge criterion would be a new and more expensive kind of false confidence. Adversarial review raises the floor. It does not replace execution, and it does not replace the human on the merge call. The plan records the outcome we chose, not to ship the patch at all and to wait for the upstream fix instead, and notes that the deciding evidence was the blocker review missed.

## Dogfooding

The pull requests that introduced this skill to all three repos were themselves adversarially reviewed, with two lenses: fresh-eyes skill-craft and devil's advocate. The reviewers found real problems in the first cut:

- An **unverifiable dated anecdote** used as the section's justification. It sounded authoritative and could not be checked, so it came out.
- A **citation to a skill that didn't exist.** The Android version of the text pointed at a `reviewbot` skill that only the iOS and web repos have. Copy-paste across repos, caught by a reviewer that actually went looking for the file.
- A **transcript quote** that added length without adding rule.
- **Ambiguity about who initiates.** The first draft didn't resolve whether the agent dispatches reviewers on its own. It now proposes, and dispatches only on the user's go-ahead.
- **Vagueness about when.** "Risky or tricky changes" became a concrete list of trigger classes: cross-target invariants, persistence/entitlement, layout/geometry, plan docs about to be implemented.
- **No stopping rule.** "Iterate to convergence" became "usually one round; stop when a round returns only nits."

Every one of those is a thing the authoring session had read a dozen times without seeing.

## Lessons Learned

- **Name the decision you want attacked.** A general adversarial prompt returns general skepticism. "We decided X; argue X is wrong" returns a counter-case or a concession.
- **Write down why a refuted finding was wrong.** A plausible reviewer misreading your code is usually a sign the code needs a comment or a clearer name.
- **Delegate the lenses, keep the verdict.** Reviewer briefs are bounded work for a cheaper model. Adjudication needs the context and the most capable model you have.
- **Propose the round; don't run it by default.** Parallel reviewers cost real time and tokens. Let the human decide it's worth it, and stop when a round returns only nits.
- **A passed round is evidence, not proof.** Ours passed a full round with a blocker still in it. Still run the code.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the July 2026 run of failed fixes rather than a general observation, the title dropped its subtitle, stacked sentences were split throughout, and Lessons Learned went from ten bullets to five that the body does not already state as headings or bolded rules. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
