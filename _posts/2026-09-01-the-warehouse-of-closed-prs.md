---
layout: post
title: "The Warehouse of Closed PRs"
date: 2026-09-01 18:00:00 -0600
summary: "Sixty-seven finished, flag-gated feature PRs and capacity to ship a handful: close them unmerged and let GitHub keep the refs. The same move in reverse re-lands a 51-file reviewed PR as eleven slices in two days, with drift showing up as an apply conflict instead of quiet divergence."
tags: [git, github, workflow, agents, ios]
---

## The Problem

In the first two weeks of July 2026, a run of agent sessions on the [Hello Weather](https://helloweather.com) iOS app produced 67 pull requests. Each one was a finished feature: watch detail views, Live Activities, a widget builder, CarPlay, a menu bar app, alert severity, per-location storage. Each one sat behind a debug flag that was off by default, so with the flag off the app behaved the same as production. Each one built cleanly on all three targets. On July 15 the owner decided that none of them mattered more than the work already in line: accessibility, localization, the iOS 27 pass, and a featuring nomination. So the question was what to do with sixty-odd finished features that would not ship for months.

The usual answer is to keep a branch per feature, and it is the wrong one. A branch that nobody is merging drifts away from main. Conflicts pile up. Eventually someone has to rebase it, and to do that they have to learn the code again. Sixty of those branches is a chore that never ends. A tracking issue that lists them goes stale the same way, one step removed. What the team needed was a place to put finished work that costs nothing while it waits and comes back intact when its turn arrives.

Three weeks later the same repository hit the opposite problem. A performance change touching 51 files had been through two rounds of review with seven reviewers, and every round found something new. The code was correct and tested, but it was too big for anyone to review with confidence in one go. This finished work needed to come apart, not wait.

## The Solution

Two practices, which turn out to be the same move in opposite directions:

- Store a finished implementation by closing its PR without merging it. The plans index is the only catalog.
- Close a reviewed PR and land it again as small pieces, each copied out of the closed PR's final state.

This post calls the first kind a banked PR and each of the small pieces a slice.

### Bank it in a closed PR

The alternative was to keep all 67 PRs open and mergeable. The team had just done a sweep that merged main into every conflicting branch to get them all green at once. Doing that again every time main moved was exactly the chore the team wanted to avoid.

The trick relies on a GitHub behavior most people never use. Every pull request's last commit is kept permanently at a hidden address, `refs/pull/<n>/head`. It stays there whether the PR is open, merged, or closed, and whether or not the branch still exists. A closed PR whose branch was deleted also shows a *Restore branch* button. So a closed, unmerged PR is a stable, permanent place to keep code, and it carries its own description, review history, and QA notes with it. On July 15 all 67 PRs were closed with the same comment, their branches were deleted, and the plans index got a link to each banked PR next to the matching plan, with the flag name beside it. The index is the ranking list described in [Plans Are Disposable, Skills Are Durable](/plans-disposable-skills-durable/). A banked PR is one more thing the index points at, not something it holds.

The closing comment says everything a future reader needs:

```text
Closed unmerged as a banked reference implementation (2026-07-15
consolidation): this work ranks below the current localization /
featuring / iOS 27 sequence, so it is preserved here instead of being
kept mergeable. Its plans index entry links back to this PR. Restore
anytime via the Restore branch button or:

    git fetch origin pull/<n>/head
```

And the index entry is one line, next to the plan it belongs to:

```markdown
- **[Watch Feature Parity](plans/watch-parity.md)** - Location and source
  switching, precip complications. Banked PR: #<n> (`watchExtrasEnabled`).
```

The definition of a banked PR sits at the top of the index, so nobody has to guess what the word means: a banked PR is "a complete, flag-gated, build-verified reference implementation preserved in a closed PR". Those three adjectives are the entry requirements. Work that is not complete is a draft. Work that is not behind a flag cannot come back as a merge that changes nothing for users. Work that was never built and tested is a sketch.

What banking cannot do is keep the code mergeable. A banked PR is a patch against main as it was on the day it closed. Bringing one back later means pulling its diff out and fixing it up against current main by hand, which is the recipe in the next section. The team accepted that cost because someone pays it once, when they want the feature, instead of everyone paying a little of it forever.

### Close the reviewed PR and land it again in slices

Three weeks later the 51-file change arrived. It was a hoisting sweep across the app, watch, and widgets. A hoist moves a computation out of a loop so it runs once instead of on every pass, and this sweep pulled computed properties out of `ForEach` closures. The work started after a watch view was measured rebuilding its body on every drag frame, with about 142 array allocations and 23 calendar constructions each time. The PR passed every check the repo had. The owner closed it anyway, on the grounds that 51 files of "mechanical" change is more than a person can review with confidence. Each review round finding new issues was a sign that the change was too big to reason about, not a sign that it was finally clean.

The alternative was to rewrite the work as small PRs from scratch. That would throw away two rounds of review and every finding those rounds settled. Instead the closed PR became the source, and a plan split it into eleven slices. Each slice is a list of files plus the notes an implementer needs: which slices change behavior on purpose, which files have to move together because they share a Swift extension, and which hoists have to stay behind the same condition the original code sat behind.

Copying a slice out takes four lines:

```bash
git checkout -b <slice-branch> origin/main
git fetch origin refs/pull/<n>/head
git diff origin/main...FETCH_HEAD -- <slice files> | git apply
# slice-specific adjustments, then the full test gates, then the PR
```

The three-dot diff takes each file's final state in the closed PR, relative to where the PR branched off main, so the fixes made during review come along. The plan says this outright: never cherry-pick the first commit alone for a file that the review commits touched. Each slice PR then says where it came from, so the reviewer knows the code was already reviewed three times and can spend their attention on where the slice boundary falls.

Not every slice can be copied straight out of the source PR. Two of the eleven were closed the day they opened. Besides the hoists, they also added defensive checks, clamping ranges and guarding strides, on values that come from the team's own server. The owner rejected those checks as guarding against a problem the team controls. Both were cut again as new slices carrying only the hoists. That made them new code rather than copies, and their PR descriptions say so. A copy is trusted because it came from a reviewed PR. New code has to earn that trust again, with the build gates and a fresh review.

### Sequential, not stacked

The obvious way to land eleven slices from one source is a stack: slice two on top of slice one, and so on. The team landed them one at a time on main instead. Each later slice was cut in its own worktree off fresh main (the worktree rules are in [Never Touch the Human's Checkout](/never-touch-the-humans-checkout/)) and copied out again against main as it stood at that moment. The reason is safety. If main has changed under a slice's files, a stacked branch carries the stale base along quietly, and the conflict shows up at the end in whichever slice is unlucky. A fresh copy fails at `git apply` with a conflict on the exact file. The plan's instruction for that case is to fix it by hand and say so in the PR description. Drift shows up as an apply conflict, not as a quiet divergence.

The per-location storage PR went the same way in August, from the other direction. It was a three-times-reviewed implementation that bundled four separate concerns: a split in reset semantics, a per-slot store, a batch fetch, and a preview swap. Small PRs review better, so it was closed as the source and its plan lists four slices in landing order, with the reason each file rides where it does. Slice A is one file, plus 26 and minus 22 lines. Its PR description notes it was copied exactly from the closed PR minus one line that belongs to slice B, so the three review rounds carry over. It has been open since August 1 waiting for its landing window. On August 13 current main was merged into it and both gates ran again. That is what waiting costs when the code is 48 lines: one merge and one test run in twelve days.

Priority is not the only reason work gets banked. The design queue's policy is that merged does not mean design-approved, and a contested piece of UI goes behind a flag from the start. A "pull back" verdict is then a small PR that removes the UI while the implementation lives on, flagged or banked. Banking is what makes that verdict cheap to give.

## Results

- On July 15, 2026, the 67 banked PRs were closed unmerged in one pass and their origin branches deleted, after *Restore branch* was tried on one closed PR first. The plans index links more than sixty banked implementations today, each with its flag name where it has one.
- The 51-file hoisting sweep closed on August 4. Eleven slice PRs merged on August 5, two of them cut again with only the hoists after the originals were rejected the same day. The plan was deleted on August 6 as complete.
- The per-location storage PR closed on August 1. Slice A has been open and green since that day, refreshed against main on August 13. Slices B through D are still to be copied out, so that re-landing is a design record in progress, not a finished one.
- The accepted cost: a banked PR is not mergeable, and every revival means copying the diff out and fixing it up by hand. The other cost is discipline. A banked PR that the index does not link is lost, because once the branch is deleted the PR page is the only handle.

## Lessons Learned

- A closed PR is a better warehouse than a branch. It cannot drift, it keeps its review history, and GitHub keeps its ref at no cost.
- Define what counts as a deposit: complete, flag-gated, build-verified. Anything short of that is a draft, and a banked draft is a lost draft.
- The catalog is what makes a bank retrievable. Link every banked PR from the index in the same change that closes it.
- Copy from the source PR's final state, never from its first commit, so review fixes travel with the code.
- Land slices one at a time against current main so drift fails at apply time. A stack hides the same conflict until the end.
- If a slice is not a straight copy, say so in the PR and review it as new code.

---

## How This Post Was Made

**Prompt 1:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

**Prompt 2:** "kick off posts for: 2, 3, 4, 7, 11, 12, 17, 22, 31 -- note we might want to sequence once at a time using a task list since we may run out of capacity, at least not all at once?"

**Prompt 3:** "kick off the remainder in sub-agents, we have capacity"

Generated by Claude Fable 5.1 using the blog-post-generator skill. One agent researched the iOS repository and proposed this post among a dozen candidates; a second agent verified the claims and wrote it. Sources: the plans index and its banked-PR definition, the July 15, 2026 consolidation PR and the closing comment left on each banked PR, the July 2026 session tracking issue, the per-location storage split plan and the reference PR's body and review rounds, slice A's PR body and its August 13 update, the ForEach hoisting plan as first committed on August 4, the hoisting sweep PR's closing comment, the two rejected slices' closing comments and the lite re-cut's body, the plan retirement PR of August 6, and the design queue policy in the index. PR states were confirmed with the GitHub CLI and the `refs/pull/<n>/head` refs were confirmed to resolve on the remote for three banked PRs. Judgment calls: the repository is private, so no PR is linked or numbered in the body and features are named by kind; the owner and the designer are unnamed; the closing comment and index line are quoted with the PR number replaced by a placeholder; the count of banked implementations is given as "more than sixty" because the index's banked lines reference 66 closed, unmerged PRs and two of those are cross-references rather than deposits.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. The reader's quoted paragraph (the "lite slice" one) was replaced with the plain version from the skill, fitted to the facts. Hoist is now defined where it first appears, "lite slice" is gone in favor of "cut again with only the hoists", and the code-copying step is called copying throughout instead of extraction. Retired words removed: byte-identical, adversarial, lean on, surface, the whole contract, for free, bespoke, "the thing extraction buys", "has to be paid". The closing aphorism in Lessons Learned became a plain rule. Judgment calls: "adversarial review rounds" became "rounds of review", since the word described the rounds rather than naming a practice; the summary line changed "surfacing" to "showing up"; and "flag-off behavior was byte-identical to production" became "behaved the same as production", since the claim is that the flag-off path is a no-op rather than a byte comparison. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"
