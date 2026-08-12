---
layout: post
title: "The Body Runs Every Frame"
date: 2026-08-05 08:30:00 -0600
summary: "A choppy watch drag traced to loop-invariant computed properties re-evaluated per ForEach element per body pass - the hoisting rules learned fixing it, and why a green, twice-reviewed 51-file fix was closed and re-landed in slices."
tags: [swiftui, ios, performance, workflow]
---

## The Problem

[Hello Weather](https://helloweather.com)'s watch app has an hourly strip - 24 hour columns you drag horizontally. On hardware, the drag was choppy, and it was choppy for a reason that generalizes to almost any SwiftUI list: **during a drag, the body runs every frame, and computed properties don't know they're in a loop.**

The strip's drag is a `@GestureState` driving an `.offset` through a computed property, so every touch event re-evaluates the whole `body`. Inside that body is a `ForEach` over 24 hours, and inside the `ForEach` are reads of computed properties that are invariant across the loop:

```swift
// In a shared extension - every one of these is a computed property.
extension HourlyView {
    var hourlyData: [Forecast.Hour] {
        viewModel.weather?.forecast?.hourly?.data ?? Forecast.Fallback.hourlyData
    }

    var hourly: [Forecast.Hour] {
        Array(hourlyData.prefix(min(hourCount, hourlyData.count)))
    }

    var maxHourlyTemp: Int {
        let safeCount = min(hourCount + 2, hourlyData.count)
        return (hourlyData.prefix(safeCount).map { $0.temperature ?? 0 }.max() ?? 0).toInt
    }
    // ...minHourlyTemp and showHourlyPrecips, same shape
}
```

Each looks innocent alone. But `hourly[index]` materializes a fresh array per subscript, each hour column reads `maxHourlyTemp`, `minHourlyTemp`, and `showHourlyPrecips` (each a prefix-and-map over the data), and the day-boundary check between adjacent hours constructed a `Calendar` per comparison - 23 of them for 24 hours. Totaled up: **roughly 142 array allocations and 23 `Calendar` constructions per body evaluation**, which during a drag means per frame, on a watch.

Nobody wrote that. It's what accretes when "add a computed var to the shared extension" is the path of least resistance for four years, and it's invisible until a gesture makes the body hot.

## The Fix Is a `let`

SwiftUI evaluates a computed property every time it's read; it has no memoization. But a `let` at the top of a `@ViewBuilder` evaluates exactly once per body pass. So the fix is almost embarrassingly small - snapshot the invariants above the loop:

```swift
private var hourlyChart: some View {
    let hours = hourly
    let maxTemp = maxHourlyTemp
    let minTemp = minHourlyTemp
    let showPrecips = showHourlyPrecips
    let dayCalendar = settingsManager.hourlyIsGrouped
        ? viewModel.weather?.forecast?.currentCalendar
        : nil

    return HStack(spacing: 2) {
        ForEach(hours.indices, id: \.self) { index in
            let hour = hours[index]

            if index > 0,
               let calendar = dayCalendar,
               let currentTime = hour.time,
               let previousTime = hours[index - 1].time,
               calendar.startOfDay(for: currentTime) != calendar.startOfDay(for: previousTime) {
                DaySeparatorView(day: currentTime)
            }

            Hour(max: maxTemp, min: minTemp, bar: hour, showPrecips: showPrecips)
        }
    }
}
```

Per body pass: one array, one max, one min, one precip scan, one calendar. Two details worth noting because they bit us:

- The snapshots use *different names* (`hours = hourly`), not Swift's shadowing shorthand. `let x = x` works inside `if let`, but at declaration scope it does not compile - use a distinct name or `self.x`.
- Hoisting doesn't break observation. Invalidation in SwiftUI is object-level (`@ObservedObject`, `@EnvironmentObject`), not property-level - when the view model changes, the body re-runs and re-snapshots. A `let` inside body is always exactly as fresh as the body pass that created it.

One `ForEach`, one afternoon. Except a three-agent audit then went looking for the same class across the app, watch, and widget targets - and found it in **51 files**. Same shape everywhere: settings pickers rebuilding localized name dictionaries per option, a radar legend measuring label widths per label per animation frame, a locations list running three JSON decodes per row, stats charts re-deriving ranges per data point.

## Hoisting Rules, Learned the Hard Way

The sweep that fixed all 51 files went through two adversarial review rounds (seven independent reviewers, run per the practice in [/adversarial-review-rounds/](/adversarial-review-rounds/)), and the reviewers' confirmed findings turned "hoist the invariants" from an instinct into a ruleset. This is the part worth stealing, because every one of these rules exists because the naive hoist was wrong:

**Hoist only what is provably invariant AND unconditionally evaluated.** These are two separate tests. A sparkline view read its precip range only behind `if showsCurve` - hoisting that read above the guard made it run for *every* daily row, including rows where the server's `min...max` bounds could be inverted, which traps `ClosedRange` at construction. The hoist widened both the performance cost and the crash surface. If the original read sat behind a condition, the hoist keeps the same condition (with placeholder values for the other branch), or it doesn't happen.

**Hoist INTO deferred builders, never out of them.** `Menu` and `Picker` content closures don't evaluate at body time - they evaluate at *presentation* time. A names dictionary hoisted from inside a `Menu` builder up to body scope changes freshness: the user now sees a snapshot from whenever the body last ran, not from when they opened the menu. Round two of review moved one of ours back inside each menu builder for exactly this reason. The flip side is free wins: work already *inside* a deferred builder that runs per option can be hoisted to the top of that builder, and it still only costs anything when the menu opens.

**Leave `onAppear` and action-closure reads live.** Same principle, other direction: those closures run later than body, and they should see the world as it is when they fire, not a body-time snapshot.

**Pass optionals through; don't `?? ""`.** Several hoists turned force-unwraps and dictionary lookups into parameters. Where the receiving view's parameter is `Optional`, pass the optional: `nil` makes SwiftUI skip the subview entirely, while `""` renders an empty shell that occupies layout. They are not the same view tree, and the diff that "just adds a fallback" is a visible behavior change.

**Harden server-fed ranges while you're in there.** Any `min...max` built from network data gets clamped (`lower...max(lower, upper)`), and any `stride` gets a `> 0` step guard - review found an empirically verified stride-by-zero trap sitting next to one of the hoists.

The meta-rule: a "mechanical" perf sweep isn't mechanical. Every one of those findings came from a reviewer with no stake in the sweep being simple.

## Green Everywhere, Merged Nowhere

Here's the process half, and the more transferable lesson.

The all-at-once sweep PR was, by every formal measure, done. Build and unit-test gates green at every commit. Two adversarial review rounds survived, every finding adjudicated and fixed. Fifty-one files of consistent, pattern-applying change.

We closed it anyway.

Not because anything was wrong with it - because **51 files is beyond confident human review**, and each review round had surfaced new issues, which is the strongest possible evidence that another round would too. "All checks passed" measures what the checks measure. The honest question at the merge call is "can a human vouch for this diff?", and for 51 files of changes that are 90% mechanical and 10% judgment, the answer was no - the 10% hides in the 90%.

So the PR became a **patch source** instead of a merge candidate: closed, with its branch kept on origin under a stable name, and a plan doc decomposing it into 11 slices - one to four files each, ordered, with per-slice notes carrying exactly what an implementer needs. The two slices that carry deliberate behavior deltas (the range-clamping hardening, and a rewrite that stops force-unwrapping a nullable timestamp so a missing time skips a separator instead of crashing) are flagged in the table and **must disclose those deltas in their PR bodies**. In a sliced re-land, "mostly mechanical" is no longer an acceptable summary - each slice is either behavior-identical or it says what changed.

The extraction recipe matters more than it looks:

```
git diff main...fix/foreach-hoisting -- <this slice's paths>
```

applying the **latest** state of each file from the banked branch - never cherry-picking the original sweep commit alone. The review-fix commits *amend* the sweep, so commit one by itself contains exactly the bugs the reviewers caught. And since `main` drifts while slices land one at a time, each slice re-verifies its files against fresh `main` before applying. Each slice PR cites the banked branch and the reviews it already survived, so slice review is cheap: confirm the extraction is faithful and the slice is self-contained, not re-litigate the pattern.

## Landed So Far

**The watch strip** went first, promoted out of the slice sequence into its own release-blocker lane. One file, byte-identical to the banked version, verified against branch history that the review-fix commits never touched it. Its PR states the honest expectation up front: finger-tracked dragging improves markedly, but the momentum glide after release stays juddery - the deceleration loop mutates state from a `Task.sleep(16ms)` loop, and watchOS has no display-link to align ticks to vsync, so cheaper frames can't fix irregular presentation. That expectation is written down *before* the hands-on device check by the project owner, which is the decision gate for whether a separate paged-navigation rewrite (also built, also banked) ships or closes. Writing the expected outcome down before the check is what keeps the check honest.

**The widget strips** went second: pure hoists in the hourly and minutely widget views, where body cost matters most because widget rendering runs in a time-constrained context. A read-only mini-review returned zero findings and one good nit - with the hoists, an empty data array now evaluates the three getters once instead of zero times, which is safe but is exactly the kind of edge a per-slice review can actually hold in its head. That nit is the sliced approach working as designed.

## The Audit Found More Than the Sweep Fixed

The same audit surfaced a Phase-2 list that the mechanical sweep deliberately excludes, because these need design rather than a `let`:

- **Per-row calendar chains.** Each hour column formats its timestamp by resolving a calendar and scanning daily data for sun events - about 190 `Calendar` constructions and 72 linear scans per strip body evaluation, across every target that shares the row view. The fix is a parent-computed calendar and sun-event map passed down, which touches shared view APIs.
- **An O(24²) lookup.** A forecast helper builds its day view with `allHoursInDay.map { first(where:) }` - a linear search per hour. Needs a `Date`-keyed dictionary.
- **The root multiplier.** The localization helper constructs a fresh `UserDefaults(suiteName:)` and `Locale` on *every string resolution*. Every "rebuild the names dictionary per row" finding in the sweep was this cost times N; caching it needs language-change invalidation and its own careful PR.

That's the payoff of auditing a bug class instead of fixing a bug: the sweep repaired the sites, but the audit found the systems. Hoisting is the floor. The ceiling is making the expensive thing cheap once, for everyone.

## Lessons Learned

- **The body runs every frame.** Any gesture, animation, or timer that invalidates a view turns its body into a hot loop. Computed properties read inside a `ForEach` are the first place to look, because cost there multiplies by element count.
- **SwiftUI won't memoize for you.** A computed property is a function. A `let` at the top of the builder is the cache, it's one line, and it's always exactly as fresh as the body pass.
- **Hoisting has a ruleset, not a reflex.** Invariant AND unconditional; into deferred builders, never out; leave late-running closures live; pass optionals through. Each rule exists because the naive version shipped a bug to review.
- **Audit the class, not the instance.** One choppy drag became 51 files across three targets, plus a Phase-2 list of structural fixes no single-site patch would have found.
- **Green and reviewed is not the same as reviewable.** If every review round finds new issues, the change is telling you it exceeds review capacity. Believe it.
- **Bank, don't merge.** A closed PR with a named branch on origin is a patch source: the work, the review history, and the fixes are all preserved, and slices extract the latest state per file - never the first commit alone, because review amends the sweep.
- **Slices must confess.** Decomposing a "mechanical" change removes the cover that word provided. Any slice with a behavior delta discloses it in the PR body, or the slicing was theater.
- **Write the expected outcome before the hardware check.** "Tracking improves, judder remains" recorded in advance makes the device check a real experiment instead of a vibe.

---

## How This Post Was Made

**Prompt 1:** "see recent work in ~/Code/helloweather, perhaps a blog post about our opus 4.8 agents and why we decided to do that? perhaps something about the swift testing + snapshots inspired by minitest-snapshots? anything else? bring me a list of potential post ideas for review."

**Prompt 2:** "skip 4, 5, 6, 9 but create posts for each of the others in the 1-9 list. also add Four Answers to One Question, and Write the Rule, Not the Story -- show me a concise version of your plan and then I can approve" — then "proceed, one pr per post"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.
