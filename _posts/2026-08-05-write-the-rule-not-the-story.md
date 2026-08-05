---
layout: post
title: "Write the Rule, Not the Story"
date: 2026-08-05 09:00:00 -0600
summary: "When a plan dies, its durable knowledge moves into a skill — and it keeps arriving as a changelog. A 59-section audit found only 44% met the bar, and the fix was a one-paragraph standard: state what a session must do now, never how the decision was reached."
tags: [ai-agents, documentation, workflow]
---

## The Problem

We've written about the lifecycle before: [plans are disposable, skills are durable](/plans-disposable-skills-durable/). A plan ships, its durable knowledge — thresholds, never-dos, revive triggers — migrates into the owning skill, and the plan is deleted. That policy runs across all three [Hello Weather](https://helloweather.com) repos, and it works.

This post is about the moment the policy doesn't cover: the migration itself. Because there's a failure mode hiding in "migrate the durable knowledge," and it's this — **the knowledge arrives as a changelog.**

The agent doing the migration has just read the whole plan. It knows the dates, the measurements, the rejected alternatives, the session where the decision flipped. All of that feels like durable knowledge, so all of it gets moved. The skill section that lands looks like:

- **Dates in headings** — `### Tracing (decided 2026-07-29; free-wins pass 2026-07-31)`
- **Audit provenance** — "per the six-batch audit reviewed today," "confirmed in the follow-up pass"
- **War-story narration** — "decided," "verified on," "rolled back after," "opened and withdrawn"
- **Why-it-was-right justification** — the benchmark that settled the argument, restated forever
- **Measurements no future session will ever apply** — 6.3M span events in nine days, sitting behind a rule that is simply *"tracing stays off"*

None of that is wrong. It's all true, all verified, all hard-won. And that's exactly what makes it insidious — [the doc audit](/agent-doc-audit/) taught us to delete fiction, but this content passes every grep. It's not fiction. It's *history*. The test it fails is a different one.

A week after the web repo's plans sweep, a review audited all 59 migrated skill sections against that different test. **Only 44% met the bar.** The rest carried decision-log narration a future session would have to read past to find the rule.

## The Test: Will a Session Apply This?

A skill is read hundreds of times, by sessions that need instructions, not history. Every sentence of narration is a tax charged on every one of those reads — context spent, attention diluted — and it's a tax that compounds, because narration invites more narration. Once a section has one dated entry, the natural next update appends a second one, and now the skill is a changelog with a rule buried in it.

The standard that came out of the audit, codified in the planning skill of all three repos, quoted verbatim:

> **Write the rule, not the story.** Migrated text states what a session must do now — thresholds, never-dos, revive triggers — never how the decision was reached. Strip decision dates, session/audit provenance, "decided / verified on / rolled back after" narration, why-it-was-right justification, and measurements no session would apply; one trailing `(#NNNN)` is the entire citation budget, and dates never appear in headings. If the migrated section reads like a changelog entry, it belongs in the PR body, not the skill.

Two design choices in that paragraph are worth calling out.

**The citation budget is one trailing PR number.** Not zero — provenance matters, and a future session that doubts a rule needs somewhere to go. But the PR body is where the story lives: the dates, the measurements, the five rejected alternatives, the review argument. The skill gets the rule plus a pointer. `git log` and the PR archive are the changelog; the skill is the law.

**The last sentence is the test, not a style rule.** "If the migrated section reads like a changelog entry, it belongs in the PR body" is something an agent can actually evaluate at authorship time, without a linter. Deliberately, the standard was *not* added to the review bot or to per-skill headers — one rule at the point of authorship, no style-guide machinery. The doc audit's lesson about always-loaded token cost applies to enforcement text too.

## Before and After

The audit rewrote six log-style sections outright and trimmed a dozen more — every threshold, never-do, and revive trigger preserved; the history now lives only in the PRs. Here's the shape of the fix. The "before" below is an illustrative reconstruction of the pattern the audit describes (dated heading, provenance, war-story measurements), not a real section:

```markdown
### Tracing (decided 2026-07-29; free-wins pass 2026-07-31)

Distributed tracing was enabled during the June capacity investigation
(see the monitoring audit, batch 4) and generated 6.3M span events in
nine days against a 10M/mo ingest budget. We evaluated sampling at 10%
and 1% before concluding on 2026-07-29 that traces weren't answering
any question the slow-query log didn't already answer. Rolled back
after the review; verified quiet on 2026-07-31.
```

Eighty words. One rule. The rule, after:

```markdown
### Tracing

Tracing stays off — it consumes most of the ingest budget without
answering anything the slow-query log doesn't. Revive only if a
cross-service latency question can't be localized any other way (#NNNN).
```

Notice what survived: the never-do, the *reason stated as a present-tense fact* (it consumes budget without answering questions — still true, still applicable), and the revive trigger. What died: every date, the batch number, the sampling experiments, the 6.3M measurement. A session that needs to know whether to enable tracing gets its answer in one sentence. A session that needs to know *how we found out* follows the PR number — and that session is rare enough that it can pay the lookup cost itself, instead of every other session paying it forever.

The distinction that takes practice: **a number a session would apply is a threshold; a number a session would only nod at is a war story.** "Alert when the cache rate drops below 90%" stays. "The incident that taught us this peaked at 4.2M requests" goes. Same digits, opposite fates.

## Three Companion Rules

The standard travels with three adjacent rules that landed in the same push, each closing a different leak in the migration path.

**Skills never link into `plans/`.** Plans are temporary by policy, so a skill that links to one holds a link that is *guaranteed* to rot — the plan's deletion is scheduled, not hypothetical. An audit pass found a dozen skill-to-plan filename references and rerouted every one: cite the PR instead, and if the skill actually needs something the plan holds, that's the signal to migrate it now. (The plans index linking to plans is fine — that's the index's job.)

**Gotcha docs admit only traps that can still bite.** From the always-loaded agents file, verbatim:

> Document a gotcha here or in a skill only if it can still bite after its fix lands — a fixed bug's record is its PR, its code comment, and its test.

This rule has a good origin story: the PR that fixed an HTTP request-line-length trap — scanner probes with huge query strings were being dropped before the app ever saw them — initially documented its own war story in the agents file. An earlier revision of that same PR added the trap's full mechanism as a gotcha. It was cut for failing its own test: the fix was shipped, pinned by a regression test, and explained by a comment at the point of use. A documented gotcha that can no longer bite is a changelog entry wearing a warning label.

**Close out in the shipping diff.** From the planning skill:

> The PR that ships a plan's last step migrates and removes the plan in the same diff; if work remains, it re-dates the plan's status/trigger in that diff. A release-gated flip closes in the release's follow-up PR.

This is what prevents the migration backlog from existing at all. If close-out is a separate later task, "later" accumulates — that's how one repo reached 135 plan files. Bundling migration and removal into the diff that finishes the work means there is never a pile of dead plans waiting for a sweep, and the migrated text is written while the author still knows which sentence is the rule and which is the story.

## Sequencing the Sweep

The sweep that surfaced all this was itself structured to make aggressive deletion safe — three PRs, strictly ordered:

1. **Additions only.** Every durable nugget in every removal candidate is rehomed into the owning skill first. No plan is removed. This PR can be reviewed as pure gain: if it's wrong, nothing was lost.
2. **Removals.** With everything rehomed, the second PR deletes the completed plans — 22 in this round — knowing nothing in the diff is the last copy of anything. Verification is mechanical: an orphan sweep of the index, plus a repo-wide grep for all 22 deleted basenames.
3. **Consolidation.** Three overlapping dormant plans behind one trigger, with mutually inconsistent stale baselines, merged into one — reopen gates copied verbatim, the corrected baseline re-derived from the config file that is actually ground truth.

The plans directory went 114 → 90 files in a day, and the ordering is why that was boring instead of terrifying. It also produced the natural checkpoint: the review of PR 1 — the additions — is where the changelog smell was caught, because that's the PR where all the migrated prose is visible in one diff. Sequence your sweep this way and the write-the-rule audit has an obvious place to happen.

The same day the web repo codified the standard, the iOS repo's in-flight sweep adopted it mid-PR and the Android repo added it to its planning skill — two paragraphs, same wording. Cross-repo consistency here isn't cosmetic: agents move between these repos, and a standard that only one planning skill states is a standard that two-thirds of sessions never load.

## The Standard Isn't Fully Complied With Where It's Written

Honest coda. The Android planning skill — the same file that now says dates never appear and one PR number is the entire citation budget — states, a few lines below the new standard:

> PLANS.md is a navigation and ranking surface, not a document store. Duplicated prose goes stale the moment the underlying plan moves (policy per [the project owner], 2026-07-23).

A dated, attributed provenance note, in the document that just banned dated provenance notes. The web planning skill does it too — "(owner decision, 2026-07-31)," "(owner rule, 2026-07-30)" — as section-heading garnish on the very rules this post quotes.

There's a defensible reading: those attributions mark *who has authority to reverse the rule*, which is arguably a revive trigger, not narration. But mostly it's the honest reading — the pull toward recording how a decision happened is strong enough that it survives inside the rule against it. Which is the best argument for having the rule written down at all: the failure mode isn't a mistake agents make, it's a gradient everything slides down. You don't fix a gradient with intentions. You fix it with a test applied at authorship, and you expect to keep applying it.

## Lessons Learned

- **The knowledge that survives a plan is the rule, not the story.** Thresholds, never-dos, and revive triggers migrate; dates, provenance, and war stories stay in the PR body, where `git log` already preserves them for free.
- **One trailing `(#NNNN)` is the entire citation budget.** Provenance gets a pointer, not a paragraph. The rare session that doubts the rule pays one lookup; every other session pays nothing.
- **A number a session would apply is a threshold; a number it would nod at is a war story.** Same digits, opposite fates.
- **Test gotchas for future teeth.** If the fix ships with a test and a point-of-use comment, the trap can't bite again — and its writeup is a changelog entry, not a gotcha.
- **Skills never link into plans.** One document type is scheduled for deletion; linking into it from a permanent document is pre-ordered link rot.
- **Close out in the shipping diff.** Migration written at ship time is written by the one author who still knows which sentence is load-bearing.
- **Additions, then removals, then consolidation.** The additions-only PR makes deletion safe *and* concentrates all migrated prose into one reviewable diff — which is exactly where changelog-style text gets caught.
- **Expect imperfect compliance, including in the rule's own file.** Narration is a gradient, not a blunder. Write the test where authorship happens and keep running it.

---

## How This Post Was Made

**Prompt 1:** "see recent work in ~/Code/helloweather, perhaps a blog post about our opus 4.8 agents and why we decided to do that? perhaps something about the swift testing + snapshots inspired by minitest-snapshots? anything else? bring me a list of potential post ideas for review."

**Prompt 2:** "skip 4, 5, 6, 9 but create posts for each of the others in the 1-9 list. also add Four Answers to One Question, and Write the Rule, Not the Story -- show me a concise version of your plan and then I can approve" — then "proceed, one pr per post"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.
