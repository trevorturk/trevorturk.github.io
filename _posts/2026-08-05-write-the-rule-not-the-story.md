---
layout: post
title: "Write the Rule, Not the Story"
date: 2026-08-05 09:00:00 -0600
summary: "When a plan is finished, what's still useful in it moves into a skill, and it tends to arrive as a changelog. An audit of 59 moved sections found only 44% were just the rule. The fix was a one-paragraph standard: say what a session must do now, not how we decided."
tags: [ai-agents, documentation, workflow]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Less than an hour after the web repo's plans sweep landed, we audited its first PR. The audit checked all 59 skill sections the sweep had moved out of plans. Only 44% passed. The rest told the story of how we'd decided, and a future session would have to read past that story to find the rule.

We've written about the lifecycle behind those sections: [plans are disposable, skills are durable](/plans-disposable-skills-durable/). A plan is a working document for one piece of work. A skill is a document that agents load every time they work in its area. When a plan ships, whatever in it still matters moves into the skill that owns that area, and the plan is deleted. That means thresholds, never-dos, and revive triggers, meaning the conditions for picking the work back up. All three [Hello Weather](https://helloweather.com) repos work this way, and it works. What the policy didn't cover was the move itself. The move has a failure mode: the knowledge arrives as a changelog.

The agent doing the move has just read the whole plan. It knows the dates, the measurements, the alternatives we rejected, and the session where the decision flipped. All of that feels worth keeping, so all of it gets moved. The section that lands looks like this:

- **Dates in headings** — `### Tracing (decided 2026-07-29; free-wins pass 2026-07-31)`
- **Notes on where the rule came from** — "per the six-batch audit reviewed today," "confirmed in the follow-up pass"
- **The story of the decision** — "decided," "verified on," "rolled back after," "opened and withdrawn"
- **Why we were right** — the benchmark that settled the argument, kept forever
- **Measurements no session will ever use** — 6.3M span events in nine days, behind a rule that just says *"tracing stays off"*

None of that is wrong. It's all true and all checked, which is why it's hard to catch. [The doc audit](/agent-doc-audit/) taught us to delete claims that aren't true, and this content passes every grep because it is true. It's history, and history fails a different test.

## The Test: Will a Session Apply This?

A skill is read hundreds of times, by sessions that need instructions. Every sentence of story costs a little on each of those reads. It also gets worse over time, because story invites more story. Once a section has one dated entry, the next update adds a second, and the skill turns into a changelog with a rule buried in it.

Here's the standard that came out of the audit. It's now in the planning skill of all three repos, quoted as written:

> **Write the rule, not the story.** Migrated text states what a session must do now — thresholds, never-dos, revive triggers — never how the decision was reached. Strip decision dates, session/audit provenance, "decided / verified on / rolled back after" narration, why-it-was-right justification, and measurements no session would apply; one trailing `(#NNNN)` is the entire citation budget, and dates never appear in headings. If the migrated section reads like a changelog entry, it belongs in the PR body, not the skill.

Two choices in that paragraph are worth explaining.

- **The rule gets one PR number at the end, and nothing else.** Not zero. A session that doubts a rule needs somewhere to go. The story lives in the PR body: the dates, the measurements, the five alternatives we rejected, the review argument. The skill gets the rule and a pointer. `git log` and the PRs are the changelog. The skill only holds the rules.
- **The last sentence is a test, not a style rule.** An agent can ask "does this read like a changelog entry?" while it's writing, with no linter. We chose not to add the standard to the review bot or to the top of every skill. It's one rule, checked at the moment of writing. The doc audit's lesson applies here too: text that loads on every session costs tokens on every session, and that includes text that enforces rules.

## Before and After

The audit rewrote six changelog-style sections from scratch and trimmed a dozen more. Every threshold, never-do, and revive trigger survived. The history now lives only in the PRs. The "before" below is a made-up example of the pattern the audit describes, not a real section:

```markdown
### Tracing (decided 2026-07-29; free-wins pass 2026-07-31)

Distributed tracing was enabled during the June capacity investigation
(see the monitoring audit, batch 4) and generated 6.3M span events in
nine days against a 10M/mo ingest budget. We evaluated sampling at 10%
and 1% before concluding on 2026-07-29 that traces weren't answering
any question the slow-query log didn't already answer. Rolled back
after the review; verified quiet on 2026-07-31.
```

That's eighty words for one rule. Here's the rule after:

```markdown
### Tracing

Tracing stays off — it consumes most of the ingest budget without
answering anything the slow-query log doesn't. Revive only if a
cross-service latency question can't be localized any other way (#NNNN).
```

What survived: the never-do, the reason stated in the present tense (tracing eats budget without answering questions, which is still true), and the revive trigger. What went: every date, the batch number, the sampling experiments, and the 6.3M number. A session deciding whether to turn tracing on gets its answer in one sentence. A session that needs to know how we found out follows the PR number. That session is rare, so it's fine for it to do the lookup, instead of every other session reading the story.

The part that takes practice is telling the two kinds of number apart. A number a session would act on is a threshold. A number a session would only nod at is a story. "Alert when the cache rate drops below 90%" stays. "The incident that taught us this peaked at 4.2M requests" goes.

## Three Companion Rules

The standard landed with three other rules. Each one closes a different way for story to leak into a skill.

### Skills never link into `plans/`

Plans get deleted by policy, so a link from a skill to a plan will break. It's a matter of when. An audit pass found a dozen skills that named a plan file, and we rerouted every one. Cite the PR instead. If the skill really needs something the plan holds, that's the sign to move it now. (The plans index can link to plans. That's its job.)

### Gotcha docs admit only traps that can still bite

From the agents file, which every session loads, quoted as written:

> Document a gotcha here or in a skill only if it can still bite after its fix lands — a fixed bug's record is its PR, its code comment, and its test.

The rule came out of one PR. That PR fixed a trap in HTTP request-line length: scanner probes with huge query strings were being dropped before the app ever saw them. An earlier revision of the PR wrote the trap up as a gotcha in the agents file, with the full mechanism. We cut it because it failed its own test. The fix had shipped, a regression test pinned it, and a comment at the point of use explained it. Once the trap can't bite, the write-up is history, and history goes in the PR.

### Close out in the shipping diff

From the iOS and Android planning skills, which use the same wording:

> The PR that ships a plan's last step migrates and removes the plan in the same diff; if work remains, it re-dates the plan's status/trigger in that diff. A release-gated flip closes in the release's follow-up PR.

This rule is why there's no backlog of plans waiting to be moved. If closing out a plan is a separate task for later, later piles up. That's how one repo reached 135 plan files. When the diff that finishes the work also moves the knowledge and deletes the plan, dead plans don't pile up waiting for a sweep. The moved text also gets written while the author still knows which sentence is the rule and which is the story.

## Sequencing the Sweep

The sweep that turned all this up was built to make deleting a lot of files safe. It was three PRs, in a fixed order.

1. **Additions only.** Everything worth keeping from every plan we meant to delete is moved into its skill first. No plan is removed. This PR is safe to review: if something in it is wrong, nothing has been lost.
2. **Removals.** The second PR deletes the finished plans, 22 in this round. Nothing in that diff is the last copy of anything. Checking it is mechanical: look for index entries that point at nothing, and grep the whole repo for all 22 deleted filenames.
3. **Consolidation.** Three dormant plans overlapped: they shared one revive trigger and each quoted a different stale baseline. The third PR merged them into one. The revive triggers were copied word for word, and the baseline was taken fresh from the config file, which is the real source.

The plans directory went from 114 files to 90 in a day. Because of the ordering, that was boring rather than scary. The ordering also gave us the checkpoint. We caught the changelog problem while reviewing PR 1, the additions, because that's the one diff where all the moved prose is visible at once. Order a sweep this way and the audit has an obvious place to happen.

The web repo wrote the standard down, and the same day the iOS repo's sweep, still in progress, adopted it mid-PR. The Android repo added it to its planning skill that day too. Both copies are two paragraphs with the same wording. That matters because agents move between these repos. A standard that only one planning skill states is one that two-thirds of sessions never see.

## The Standard Isn't Fully Complied With Where It's Written

The Android planning skill now says dates never appear in headings and one PR number is the only citation allowed. A few lines below the new standard, the same file says:

> PLANS.md is a navigation and ranking surface, not a document store. Duplicated prose goes stale the moment the underlying plan moves (policy per [the project owner], 2026-07-23).

That's a dated note on who decided, in the file that just banned dated notes on who decided. The web planning skill does it too. It tags the very rules this post quotes with "(owner decision, 2026-07-31)" and "(owner rule, 2026-07-30)" in their section headings.

You could defend those notes. They say who has the authority to reverse the rule, and you could call that a revive trigger rather than story. The plainer reading is that the urge to record how a decision happened is strong enough to show up inside the rule against it. That's the best reason to write the rule down. This isn't a mistake agents make now and then. It's the direction everything drifts unless something stops it. Good intentions don't stop it. A test applied while writing does, and we expect to keep applying it.

## Lessons Learned

- **Cite the PR, don't retell it.** The rare reader who doubts a rule can look it up. Every other reader skips the story.
- **Don't link a permanent document to a temporary one.** If the permanent one needs something from it, move that now.
- **Move before you delete.** An additions-only PR makes removal safe and puts all the moved prose in one diff, where changelog text is easiest to spot.
- **Close out the plan in the diff that finishes the work.** Put it off and it becomes a backlog, and the author who knew which sentence was the rule has moved on.
- **Expect drift, even in the rule's own file.** Story creeps back in, so keep applying the test at the moment of writing.
