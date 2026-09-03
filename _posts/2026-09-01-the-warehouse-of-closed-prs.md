---
layout: post
title: "The Warehouse of Closed PRs"
date: 2026-09-01 18:00:00 -0600
summary: "We had 67 finished, flag-gated feature PRs and room to ship a handful, so we closed them unmerged and let GitHub keep the refs. The same move in reverse landed a 51-file reviewed PR as eleven slices in two days, and any drift showed up as a git apply conflict instead of going unnoticed."
tags: [git, github, workflow, agents, ios]
model: "Claude Fable 5.1"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

In the first two weeks of July 2026, agent sessions on the [Hello Weather](https://helloweather.com) iOS app produced 67 pull requests. Each one was a finished feature: watch detail views, Live Activities, a widget builder, CarPlay, a menu bar app, alert severity, per-location storage. Each one was behind a debug flag that was off by default, so the shipped app didn't change. Each one built on all three targets. On July 15 the owner decided that none of them mattered more than the work already in line: accessibility, localization, the iOS 27 pass, and a featuring nomination. So we had sixty-odd finished features that wouldn't ship for months, and we had to put them somewhere.

The usual answer is a branch per feature, and it's the wrong one. A branch nobody is merging drifts away from main, conflicts pile up, and eventually someone has to rebase it, which means learning the code again. Sixty of those is a chore that never ends. A tracking issue that lists them goes stale the same way. We needed somewhere to put finished work that costs nothing while it waits and comes back intact when we want it.

Three weeks later we hit the opposite problem. A performance change touching 51 files had been through two rounds of review with seven reviewers, and every round found something new. The code was correct and tested, but it was too big for anyone to review with confidence in one go. This work needed to be broken up, not stored.

## The Solution

Two practices, and they're the same move in opposite directions:

- Store a finished implementation by closing its PR without merging it. The plans index is the only catalog.
- Close a reviewed PR and land it again as small pieces, each copied out of the closed PR's final state.

We call the first kind a banked PR and each of the small pieces a slice.

### Bank it in a closed PR

The alternative was to keep all 67 PRs open and mergeable. We had just done that once, merging main into every conflicting branch to get them all green at the same time. We didn't want to do it again every time main moved.

This works because of a GitHub behavior most people never use. GitHub keeps every pull request's last commit at `refs/pull/<n>/head`, permanently, whether the PR is open, merged, or closed, and whether or not the branch still exists. A closed PR whose branch was deleted also shows a *Restore branch* button. So a closed, unmerged PR is a safe place to keep code, and the description, review history, and QA notes stay with it. On July 15 we closed all 67 PRs with the same comment, deleted their branches, and added a link to each banked PR in the plans index, next to its plan, with the flag name beside it. The index is the ranking list from [Plans Are Disposable, Skills Are Durable](/plans-disposable-skills-durable/), and a banked PR is just one more thing it points at.

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

The index defines the term at the top, so nobody has to guess: a banked PR is "a complete, flag-gated, build-verified reference implementation preserved in a closed PR". Those three adjectives are the entry requirements. If the work isn't complete, it's a draft. If it isn't behind a flag, it can't come back as a merge that changes nothing for users. If it was never built and tested, it's a sketch.

Banking doesn't keep the code mergeable. A banked PR is a patch against main as it was the day it closed. To bring one back, you pull its diff out and fix it up against current main by hand, which is the recipe in the next section. We accepted that because it's paid once, when someone wants the feature, instead of a little every week forever.

### Close the reviewed PR and land it again in slices

Three weeks later the 51-file change arrived. It was a hoisting sweep across the app, watch, and widgets. A hoist moves a computation out of a loop so it runs once instead of on every pass, and this sweep pulled computed properties out of `ForEach` closures. We started it after measuring a watch view that rebuilt its body on every drag frame, with about 142 array allocations and 23 calendar constructions each time. The PR passed every check the repo had. The owner closed it anyway, because 51 files of "mechanical" change is more than a person can review with confidence. Every round had found new issues, and to the owner that meant the change was too big to reason about, not that it was finally clean.

We could have rewritten the work as small PRs from scratch, but that throws away two rounds of review. Instead the closed PR became the source, and a plan split it into eleven slices. Each slice is a list of files plus the notes an implementer needs: which slices change behavior on purpose, which files have to move together because they share a Swift extension, and which hoists have to stay behind the same condition as the original code.

Copying a slice out takes four lines:

```bash
git checkout -b <slice-branch> origin/main
git fetch origin refs/pull/<n>/head
git diff origin/main...FETCH_HEAD -- <slice files> | git apply
# slice-specific adjustments, then the full test gates, then the PR
```

The three-dot diff compares the closed PR's final state with the point where it branched off main, so the fixes made during review come along. The plan says this outright: never cherry-pick the first commit alone for a file that the review commits touched. Each slice PR says where it came from, so the reviewer knows the code was already reviewed three times and can concentrate on where the slice boundary falls.

Some slices can't be a straight copy. Two of the eleven were closed the same day they opened. On top of the hoists, they added range checks on values from our own server, and the owner rejected that as defending against a problem we control. Both were cut again with only the hoists. Those two are new code, not copies, and their PR descriptions say so. A copied slice is trustworthy because it came out of a reviewed PR. A new one has to be reviewed again.

### Sequential, not stacked

The obvious way to land eleven slices from one source is a stack: slice two on top of slice one, and so on. We landed them one at a time on main instead. Each slice was cut in its own worktree off fresh main (the worktree rules are in [Never Touch the Human's Checkout](/never-touch-the-humans-checkout/)) and copied out against main as it stood at that moment. We did it that way because it's safer. If main has changed under a slice's files, a stacked branch carries the stale base along without telling anyone, and the conflict shows up at the end in whichever slice is unlucky. A fresh copy fails at `git apply` with a conflict on the exact file. When that happens, the plan says to fix it by hand and say so in the PR description.

The per-location storage PR went the same way in August. It had been reviewed three times and bundled four separate changes: a split in reset semantics, a per-slot store, a batch fetch, and a preview swap. Small PRs review better, so we closed it as the source, and its plan lists four slices in landing order with the reason each file goes where it does. Slice A is one file, plus 26 and minus 22 lines. Its PR description says it was copied exactly from the closed PR minus one line that belongs to slice B, so the three review rounds carry over. It has been open since August 1, waiting for its turn. On August 13 we merged current main into it and ran both gates again. That's what waiting costs when the code is 48 lines: one merge and one test run in twelve days.

Priority isn't the only reason to bank work. The design queue's policy is that merged doesn't mean design-approved, so a contested piece of UI goes behind a flag from the start. If the verdict is "pull back", a small PR removes the UI and the implementation stays, flagged or banked. That verdict is cheap to give because nothing is thrown away.

## Results

- On July 15, 2026, we closed the 67 PRs unmerged in one pass and deleted their branches, after trying *Restore branch* on one closed PR first. The plans index links more than sixty banked implementations today, each with its flag name where it has one.
- The 51-file hoisting sweep closed on August 4. Eleven slice PRs merged on August 5, two of them cut again with only the hoists after the originals were rejected the same day. We deleted the plan on August 6 as complete.
- The per-location storage PR closed on August 1. Slice A has been open and green since that day, refreshed against main on August 13. Slices B through D haven't been copied out yet, so that one is still in progress.
- The cost: a banked PR isn't mergeable, and bringing one back means copying the diff out and fixing it up by hand. The other cost is discipline. A banked PR the index doesn't link is lost, because once the branch is deleted the PR page is the only way to find it.

## Lessons Learned

- A closed PR is a better warehouse than a branch. It can't drift, it keeps its review history, and GitHub keeps its ref at no cost.
- Define what counts as banked: complete, flag-gated, build-verified. Don't bank drafts.
- Link every banked PR from the index in the same change that closes it. Without the link, it's lost.
- Copy from the source PR's final state, never from its first commit, so review fixes travel with the code.
- Land slices one at a time against current main so drift fails at apply time. A stack hides the same conflict until the end.
- If a slice isn't a straight copy, say so in the PR and review it as new code.
