---
layout: post
title: "Fix Only Observed Harm"
date: 2026-08-05 08:40:00 -0600
summary: "A six-line rule for agents and reviewers: don't build a fix or a guard without evidence you can point at, write down what would reopen a won't-fix, and let a guard's own counter decide when it comes out."
tags: [ai-agents, workflow, code-review]
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

---

## How This Post Was Made

**Prompt 1:** "see recent work in ~/Code/helloweather, perhaps a blog post about our opus 4.8 agents and why we decided to do that? perhaps something about the swift testing + snapshots inspired by minitest-snapshots? anything else? bring me a list of potential post ideas for review."

**Prompt 2:** "skip 4, 5, 6, 9 but create posts for each of the others in the 1-9 list. also add Four Answers to One Question, and Write the Rule, Not the Story -- show me a concise version of your plan and then I can approve" — then "proceed, one pr per post"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. In this post the hard-wrapped paragraphs were unwrapped, the three example sections no longer open on the same "the Nth sentence" construction, stacked sentences were split, and Lessons Learned shrank from seven bullets to five. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The quoted `AGENTS.md` rule now reads "Sentry event" where the post had softened it to "error event"; the review skill's pre-existing rule is quoted in its real wording ("do not hide real bugs behind defensive code" and the `rescue` table row) instead of a paraphrase in quotation marks; and the won't-fix record is described as having migrated from the audit plan into the weather-sources skill, which happened before this post was written. The 55-deleted/1-kept, 14-day, 35–40:1, three-plus-years, 2026-07, and six-lines-two-files figures all matched their commits.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 2, run after batch 1 (#68) merged. One Claude Fable 5.1 agent wrote an ELI5 of the post first and redrafted the prose from it: first person and contractions throughout, coined terms replaced with ordinary words ("magic-byte sniff" and "bomb cap" became "check the first bytes" and "size limit", "day 0" became "today's row", "cold-smoke snowfall" became "dry, powdery snow", "alignment guard" lost its adjective), "adversarial reviewer" became "a second reviewer", "byte-identical" became "the same bytes as on main", the closing aphorisms on the gate and deletion sections were cut or folded into the fact before them, and the evidence list is no longer repeated in prose after the blockquote that already gives it. Blockquotes, the table, the italicized revive trigger, all numbers, the link, and the headings are unchanged, and no facts were added. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"

**Prompt 9:** "usage looks fine, merge it and run batch 2"
