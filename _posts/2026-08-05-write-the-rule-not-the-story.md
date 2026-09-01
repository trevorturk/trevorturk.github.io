---
layout: post
title: "Write the Rule, Not the Story"
date: 2026-08-05 09:00:00 -0600
summary: "When a plan dies, its durable knowledge moves into a skill, and it tends to arrive as a changelog. An audit of 59 migrated sections found only 44% met the bar. The fix was a one-paragraph standard: state what a session must do now, never how the decision was reached."
tags: [ai-agents, documentation, workflow]
---

## The Problem

Less than an hour after the web repo's plans sweep landed, a review of its first PR checked all 59 skill sections the sweep had migrated out of plans. Only 44% met the bar. The rest carried decision-log narration that a future session would have to read past to find the rule.

The lifecycle that produced those sections is one we've written about: [plans are disposable, skills are durable](/plans-disposable-skills-durable/). A plan ships, its durable knowledge (thresholds, never-dos, revive triggers) moves into the owning skill, and the plan is deleted. That policy runs across all three [Hello Weather](https://helloweather.com) repos, and it works. What it didn't cover was the migration itself, and the migration has a failure mode: the knowledge arrives as a changelog.

The agent doing the migration has just read the whole plan. It knows the dates, the measurements, the rejected alternatives, the session where the decision flipped. All of that feels durable, so all of it gets moved. The section that lands looks like:

- **Dates in headings** — `### Tracing (decided 2026-07-29; free-wins pass 2026-07-31)`
- **Audit provenance** — "per the six-batch audit reviewed today," "confirmed in the follow-up pass"
- **War-story narration** — "decided," "verified on," "rolled back after," "opened and withdrawn"
- **Why-it-was-right justification** — the benchmark that settled the argument, restated forever
- **Measurements no future session will ever apply** — 6.3M span events in nine days, sitting behind a rule that is simply *"tracing stays off"*

None of that is wrong. It's all true, all verified, all hard-won, and that is what makes it hard to catch. [The doc audit](/agent-doc-audit/) taught us to delete fiction, but this content passes every grep. It isn't fiction. It's history, and it fails a different test.

## The Test: Will a Session Apply This?

A skill is read hundreds of times, by sessions that need instructions, not history. Every sentence of narration is a tax on every one of those reads. The tax compounds, because narration invites more narration. Once a section has one dated entry, the next update appends a second, and the skill becomes a changelog with a rule buried in it.

The standard that came out of the audit, now in the planning skill of all three repos, quoted verbatim:

> **Write the rule, not the story.** Migrated text states what a session must do now — thresholds, never-dos, revive triggers — never how the decision was reached. Strip decision dates, session/audit provenance, "decided / verified on / rolled back after" narration, why-it-was-right justification, and measurements no session would apply; one trailing `(#NNNN)` is the entire citation budget, and dates never appear in headings. If the migrated section reads like a changelog entry, it belongs in the PR body, not the skill.

Two choices in that paragraph deserve a closer look.

- **The citation budget is one trailing PR number.** Not zero. Provenance matters, and a session that doubts a rule needs somewhere to go. But the story lives in the PR body: the dates, the measurements, the five rejected alternatives, the review argument. The skill gets the rule plus a pointer. `git log` and the PR archive are the changelog; the skill is the law.
- **The last sentence is a test, not a style rule.** "If the migrated section reads like a changelog entry, it belongs in the PR body" is something an agent can evaluate at authorship time, without a linter. The standard was deliberately not added to the review bot or to per-skill headers. One rule at the point of authorship, no style-guide machinery. The doc audit's lesson about always-loaded token cost applies to enforcement text too.

## Before and After

The audit rewrote six log-style sections outright and trimmed a dozen more. Every threshold, never-do, and revive trigger survived; the history now lives only in the PRs. The "before" below is an illustrative reconstruction of the pattern the audit describes (dated heading, provenance, war-story measurements), not a real section:

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

Notice what survived: the never-do, the reason stated as a present-tense fact (it consumes budget without answering questions, which is still true and still applicable), and the revive trigger. What died: every date, the batch number, the sampling experiments, the 6.3M measurement. A session deciding whether to enable tracing gets its answer in one sentence. A session that needs to know how we found out follows the PR number. That session is rare enough to pay the lookup cost itself, instead of every other session paying it forever.

The distinction that takes practice: a number a session would apply is a threshold, and a number a session would only nod at is a war story. "Alert when the cache rate drops below 90%" stays. "The incident that taught us this peaked at 4.2M requests" goes. Same digits, opposite fates.

## Three Companion Rules

The standard landed alongside three adjacent rules, each closing a different leak in the migration path.

### Skills never link into `plans/`

Plans are temporary by policy, so a skill that links to one holds a link guaranteed to rot. The plan's deletion is scheduled, not hypothetical. An audit pass found a dozen skill-to-plan filename references and rerouted every one. Cite the PR instead, and if the skill actually needs something the plan holds, that's the signal to migrate it now. (The plans index linking to plans is fine; that's the index's job.)

### Gotcha docs admit only traps that can still bite

From the always-loaded agents file, verbatim:

> Document a gotcha here or in a skill only if it can still bite after its fix lands — a fixed bug's record is its PR, its code comment, and its test.

The rule has a good origin story. The PR that fixed an HTTP request-line-length trap (scanner probes with huge query strings were being dropped before the app ever saw them) initially documented its own war story in the agents file. An earlier revision of that same PR added the trap's full mechanism as a gotcha. It was cut for failing its own test: the fix was shipped, pinned by a regression test, and explained by a comment at the point of use. A documented gotcha that can no longer bite is a changelog entry wearing a warning label.

### Close out in the shipping diff

From the iOS and Android planning skills, which share this wording:

> The PR that ships a plan's last step migrates and removes the plan in the same diff; if work remains, it re-dates the plan's status/trigger in that diff. A release-gated flip closes in the release's follow-up PR.

This is what keeps a migration backlog from existing at all. If close-out is a separate later task, "later" accumulates; that's how one repo reached 135 plan files. Bundling migration and removal into the diff that finishes the work means there is never a pile of dead plans waiting for a sweep. It also means the migrated text is written while the author still knows which sentence is the rule and which is the story.

## Sequencing the Sweep

The sweep that surfaced all this was itself structured to make aggressive deletion safe: three PRs, strictly ordered.

1. **Additions only.** Every durable nugget in every removal candidate is rehomed into the owning skill first. No plan is removed. This PR reviews as pure gain: if it's wrong, nothing was lost.
2. **Removals.** With everything rehomed, the second PR deletes the completed plans, 22 in this round, knowing nothing in the diff is the last copy of anything. Verification is mechanical: an orphan sweep of the index, plus a repo-wide grep for all 22 deleted basenames.
3. **Consolidation.** Three overlapping dormant plans behind one trigger, with mutually inconsistent stale baselines, merged into one. Reopen gates were copied verbatim, and the corrected baseline was re-derived from the config file that is actually ground truth.

The plans directory went 114 → 90 files in a day, and the ordering is why that was boring instead of frightening. It also produced the natural checkpoint. The review of PR 1, the additions, is where the changelog smell was caught, because that's the PR where all the migrated prose is visible in one diff. Sequence a sweep this way and the write-the-rule audit has an obvious place to happen.

The same day the web repo codified the standard, the iOS repo's in-flight sweep adopted it mid-PR and the Android repo added it to its planning skill: two paragraphs, same wording. Cross-repo consistency isn't cosmetic. Agents move between these repos, and a standard that only one planning skill states is one that two-thirds of sessions never load.

## The Standard Isn't Fully Complied With Where It's Written

The Android planning skill, the same file that now says dates never appear and one PR number is the entire citation budget, states a few lines below the new standard:

> PLANS.md is a navigation and ranking surface, not a document store. Duplicated prose goes stale the moment the underlying plan moves (policy per [the project owner], 2026-07-23).

A dated, attributed provenance note, in the document that just banned dated provenance notes. The web planning skill does it too: "(owner decision, 2026-07-31)," "(owner rule, 2026-07-30)," as section-heading garnish on the very rules this post quotes.

There is a defensible reading. Those attributions mark who has authority to reverse the rule, which is arguably a revive trigger, not narration. But the plainer reading is that the pull toward recording how a decision happened is strong enough to survive inside the rule against it. That is the best argument for writing the rule down at all. The failure mode isn't a mistake agents make; it's a gradient everything slides down. You don't fix a gradient with intentions. You fix it with a test applied at authorship, and you expect to keep applying it.

## Lessons Learned

- **Provenance gets a pointer, not a paragraph.** The rare reader who doubts a rule pays one lookup; every other reader pays nothing.
- **A permanent document never links into a temporary one.** If the permanent one needs something the temporary one holds, migrate it now.
- **Rehome before you delete.** An additions-only PR makes removal safe and puts all migrated prose in one diff, where changelog-style text is easiest to catch.
- **Migrate in the diff that finishes the work.** Deferred close-out becomes a backlog, and the author who knew which sentence was the rule is gone.
- **Narration is a gradient, not a blunder.** Expect the rule's own file to drift, and keep applying the test where authorship happens.

---

## How This Post Was Made

**Prompt 1:** "see recent work in ~/Code/helloweather, perhaps a blog post about our opus 4.8 agents and why we decided to do that? perhaps something about the swift testing + snapshots inspired by minitest-snapshots? anything else? bring me a list of potential post ideas for review."

**Prompt 2:** "skip 4, 5, 6, 9 but create posts for each of the others in the 1-9 list. also add Four Answers to One Question, and Write the Rule, Not the Story -- show me a concise version of your plan and then I can approve" — then "proceed, one pr per post"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the audit's 44% finding instead of a recap of the lifecycle post, the two design choices and three companion rules became list items and h3s, the coda lost its "honest" framing, and Lessons Learned went from eight bullets to five that the body does not already state. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The audit did not happen "a week after" the sweep: the three sweep PRs and the write-the-rule PR all landed on 2026-07-31, the audit about forty minutes after the sweep's last PR, so the opening now says so. The close-out quote is the iOS and Android planning skills' wording (the web skill states the rule in its own words), and the attribution now says which skills it comes from. The standard, the gotcha rule, the coda's quotes, the 59/44% figures, the 22 removals, 114 to 90 files, the dozen rerouted links, and the 135-file count all matched the source.
