---
layout: post
title: "The Warehouse of Closed PRs"
date: 2026-09-01 18:00:00 -0600
summary: "Sixty-seven finished, flag-gated feature PRs and capacity to ship a handful: close them unmerged and let GitHub keep the refs. The same move in reverse re-lands a 51-file reviewed PR as eleven slices in two days, with drift surfacing as an apply conflict instead of silent divergence."
tags: [git, github, workflow, agents, ios]
---

## The Problem

In the first two weeks of July 2026, a run of agent sessions on the [Hello Weather](https://helloweather.com) iOS app produced 67 pull requests. Each was a complete feature: watch detail views, Live Activities, a widget builder, CarPlay, a menu bar app, alert severity, per-location storage. Each sat behind an off-by-default debug flag so that flag-off behavior was byte-identical to production, and each built green on all three targets. On July 15 the owner decided that none of them outranked the standing sequence of accessibility, localization, the iOS 27 pass, and a featuring nomination. The question was what to do with sixty-odd finished features that would not ship for months.

The default answer is a long-lived feature branch per feature, and it is the wrong one. A branch that nobody is landing drifts from main, accumulates conflicts, and demands a rebase from someone who has to re-learn the code to do it. Sixty of them is a maintenance program. A tracking issue listing them rots the same way, one step removed. What the team needed was a way to store finished work that costs nothing while it waits and can be retrieved intact when its turn comes.

The same repository hit the mirror-image problem three weeks later. A 51-file performance sweep had survived two adversarial review rounds with seven reviewers, and every round kept finding new issues. It was correct, verified, and too large to review with confidence as one change. The finished work had to come apart, not wait.

## The Solution

Two practices, and they turn out to be the same mechanism used in opposite directions:

- Bank a finished implementation by closing its PR unmerged, with the plans index as the only catalog.
- Close a reviewed PR as the reference implementation and re-land it as small slices extracted from its final state.

### Bank it in a closed PR

The alternative considered was keeping the 67 PRs open and mergeable. A mergeability sweep had just merged post-styles main into every conflicting branch to get them all green at once. Doing that again every time main moved was the maintenance program the team was trying to avoid.

The mechanism relies on a GitHub behavior most people never lean on: every pull request's head commit is kept permanently at a `refs/pull/<n>/head` ref, whether the PR is open, merged, or closed, and whether or not the source branch still exists. A closed PR whose branch was deleted also shows a *Restore branch* button. So a closed, unmerged PR is a stable, addressable, permanently retrievable artifact, and it carries its own body, review rounds, and QA notes with it. On July 15 all 67 were closed with the same pointer comment, the origin branches were deleted, and the plans index gained a banked-PR link on every matching entry, with the flag name beside it. The index is the ranking surface described in [Plans Are Disposable, Skills Are Durable](/plans-disposable-skills-durable/), and a banked PR is one more thing it points at rather than holds.

The closing comment is the whole contract:

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

The definition itself sits at the top of the index, so a reader who meets the word for the first time does not have to guess: a banked PR is "a complete, flag-gated, build-verified reference implementation preserved in a closed PR". Those three adjectives are the acceptance criteria. Work that is not complete is a draft, not a bank deposit. Work that is not flag-gated cannot be revived as a no-op merge. Work that was never build-verified is a sketch.

What the mechanism cannot do is keep the deposit mergeable. A banked PR is a patch against the main of the day it closed. Reviving one later means extracting its diff against current main and reconciling by hand, which is the next section's recipe. The team accepted that cost because it is paid once, at revival, by someone who wants the feature, instead of continuously by nobody in particular.

### Close the reviewed PR and re-land it in slices

Three weeks later the 51-file hoisting sweep arrived: loop-invariant computed properties pulled out of `ForEach` closures across the app, watch, and widgets, found after a watch strip was measured re-evaluating its body per drag frame with about 142 array allocations and 23 calendar constructions. The PR was correct by every gate the repo had. The owner closed it anyway, on the grounds that 51 files of "mechanical" change is beyond confident human review, and that each adversarial round finding new issues was evidence the change was too big to reason about, not evidence that it was finally clean.

The alternative rejected was rewriting the work as small PRs from scratch. That throws away two review rounds of adjudicated findings. Instead the closed PR became the patch source, and a plan decomposed it into eleven slices, each a list of files with the notes an implementer needs: which slices carry deliberate behavior deltas, which files must move together because they share an extension, which hoists must stay behind the same condition the original read sat behind.

The extraction is four lines:

```bash
git checkout -b <slice-branch> origin/main
git fetch origin refs/pull/<n>/head
git diff origin/main...FETCH_HEAD -- <slice files> | git apply
# slice-specific adjustments, then the full test gates, then the PR
```

The three-dot diff takes each file's final state in the closed PR relative to the merge base, so review fixes that amended the original sweep come along. The plan says this explicitly: never cherry-pick the first commit alone for a file the review commits touched. Each slice PR then says where it came from, so the reviewer knows the code was already reviewed three times and can spend attention on the slice boundary instead.

The limit is that a slice sometimes cannot be a pure extraction. Two of the eleven were closed unmerged on the same day they opened: the owner rejected their hardening deltas, range clamps and stride guards on a server contract the team controls, as speculative defense. Both were re-cut as "lite" slices carrying only the hoists. A lite slice is a bespoke diff, and its PR body says so: the gates and a fresh read-only review are the verification, not provenance. Provenance is the thing extraction buys, and the moment a slice departs from the source, verification has to be paid for again.

### Sequential, not stacked

The obvious way to land eleven slices from one source is a stack: slice two on top of slice one, and so on. The team landed them sequentially on main instead, each later slice cut in its own worktree off fresh main (the worktree rules are in [Never Touch the Human's Checkout](/never-touch-the-humans-checkout/)) and re-extracted against the then-current main. The reason is a safety property. If main has moved under a slice's files, a stacked branch carries the stale base along silently and the conflict appears at the end, in whichever slice is unlucky. A fresh extraction fails at `git apply` with a conflict on the exact file, and the plan's instruction for that case is to reconcile by hand and say so in the PR body. Drift shows up as an apply conflict, not silent divergence.

The per-location storage PR went the same way in August, from the other direction. It was a three-times-reviewed implementation bundling four separable concerns: a reset-semantics split, a per-slot store, a batch fetch, and a preview swap. Small PRs review better, so it was closed as the reference implementation and its plan lists four slices in landing order, with the reason each file rides where it does. Slice A is one file, plus 26 and minus 22 lines, and its PR body notes it was extracted verbatim from the closed PR minus one line that belongs to slice B, so the three review rounds carry over. It has been open since August 1 waiting for its landing window, and on August 13 current main was merged into it and both gates re-run. That is what waiting costs when the code is 48 lines: one merge and one test run in twelve days.

One sentence on why work gets banked at all, since it is not only priority. The design queue's policy is that merged is not design-approved, and a contested surface goes behind a flag from the start; a "pull back" verdict is a small PR that removes the surface while the implementation survives flagged or banked. Banking is what makes that verdict cheap to give.

## Results

- On July 15, 2026, the 67 banked PRs were closed unmerged in one pass and their origin branches deleted, after *Restore branch* was verified on one closed PR first. The plans index links more than sixty banked implementations today, each with its flag name where it has one.
- The 51-file hoisting sweep closed on August 4. Eleven slice PRs merged on August 5, two of them as lite re-cuts after the originals were rejected the same day, and the plan was deleted on August 6 as complete.
- The per-location storage PR closed on August 1. Slice A has been open and green since that day, refreshed against main on August 13. Slices B through D are pending extraction, so that re-landing is a design record in progress, not a finished one.
- The accepted cost: a banked PR is not mergeable, and every revival pays an extraction and a hand reconciliation. The other cost is discipline. A banked PR that the index does not link is lost, because deleting the branch leaves the PR page as the only handle.

## Lessons Learned

- A closed PR is a better warehouse than a branch. It cannot drift, it keeps its review history, and the platform keeps its ref for free.
- Define the deposit: complete, flag-gated, build-verified. Anything short of that is a draft, and a draft banked is a draft lost.
- The catalog is the only thing that makes a bank retrievable. Link every banked PR from the index in the same change that closes it.
- Extract from the source's final state, never from its first commit, so review fixes travel with the code.
- Land slices sequentially against current main so drift fails loudly at apply time. A stack hides the same conflict until the end.
- When a slice has to diverge from its source, say so and re-verify. Provenance and verification are substitutes, and you need one of them.

---

## How This Post Was Made

**Prompt 1:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

**Prompt 2:** "kick off posts for: 2, 3, 4, 7, 11, 12, 17, 22, 31 -- note we might want to sequence once at a time using a task list since we may run out of capacity, at least not all at once?"

**Prompt 3:** "kick off the remainder in sub-agents, we have capacity"

Generated by Claude Fable 5.1 using the blog-post-generator skill. One agent researched the iOS repository and proposed this post among a dozen candidates; a second agent verified the claims and wrote it. Sources: the plans index and its banked-PR definition, the July 15, 2026 consolidation PR and the closing comment left on each banked PR, the July 2026 session tracking issue, the per-location storage split plan and the reference PR's body and review rounds, slice A's PR body and its August 13 update, the ForEach hoisting plan as first committed on August 4, the hoisting sweep PR's closing comment, the two rejected slices' closing comments and the lite re-cut's body, the plan retirement PR of August 6, and the design queue policy in the index. PR states were confirmed with the GitHub CLI and the `refs/pull/<n>/head` refs were confirmed to resolve on the remote for three banked PRs. Judgment calls: the repository is private, so no PR is linked or numbered in the body and features are named by kind; the owner and the designer are unnamed; the closing comment and index line are quoted with the PR number replaced by a placeholder; the count of banked implementations is given as "more than sixty" because the index's banked lines reference 66 closed, unmerged PRs and two of those are cross-references rather than deposits.
