---
layout: post
title: "The Body Runs Every Frame"
date: 2026-08-05 08:30:00 -0600
summary: "A choppy watch drag traced to computed properties re-read inside a ForEach on every body pass. The hoisting rules we learned fixing it, and why we closed a green, twice-reviewed 51-file fix and re-landed it in slices."
tags: [swiftui, ios, performance, workflow]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

The [Hello Weather](https://helloweather.com) watch app has an hourly strip: 24 columns, one per hour, that you drag sideways. On a real watch the drag was choppy. While you drag, SwiftUI re-runs the view's `body` every frame, and the work inside our body was repeated for every one of the 24 columns. The same thing can happen in almost any SwiftUI list.

The drag is a `@GestureState` that feeds an `.offset` through a computed property, so every touch event re-runs the whole `body`. A computed property in Swift is a function that looks like a variable: it runs its code every time you read it. Inside the body is a `ForEach` over the 24 hours, and inside the `ForEach` are reads of computed properties whose values never change during the loop:

```swift
// In a shared extension - every one of these is a computed property.
extension HourlyView {
    var hourlyData: [Forecast.Hourly.Hour] {
        viewModel.weather?.forecast?.hourly?.data ?? Forecast.Fallback.hourlyData
    }

    var hourly: [Forecast.Hourly.Hour] {
        let safeCount = min(hourCount, hourlyData.count)
        return Array(hourlyData.prefix(safeCount))
    }

    var maxHourlyTemp: Int {
        let safeCount = min(hourCount + 2, hourlyData.count)

        if SettingsManager.shared.feelsLikeMode {
            return (hourlyData.prefix(safeCount).map { $0.apparentTemperature ?? $0.temperature ?? 0 }.max() ?? 0).toInt
        } else {
            return (hourlyData.prefix(safeCount).map { $0.temperature ?? 0 }.max() ?? 0).toInt
        }
    }
    // ...minHourlyTemp, same shape

    var showHourlyPrecips: Bool {
        hourly.contains { showPrecip(val: $0.precipProbability ?? 0) }
    }
}
```

Each one looks harmless on its own, but they add up. `hourly[index]` builds a fresh array for every subscript. Each hour column reads `maxHourlyTemp`, `minHourlyTemp`, and `showHourlyPrecips`, and each of those takes a prefix of the data and maps over it. The check for a day boundary between two neighboring hours built a new `Calendar` for each comparison, 23 of them for 24 hours. That comes to **roughly 142 array allocations and 23 `Calendar` constructions per body evaluation**. During a drag, that means per frame, on a watch.

That's four years of "add a computed var to the shared extension" being the easiest way to add a value. Nobody notices until a gesture makes the body run constantly.

## The Fix Is a `let`

SwiftUI doesn't cache a computed property. It runs the property's code every time it's read. A `let` at the top of a `@ViewBuilder` runs once per body pass. So the fix is to read each unchanging value once, above the loop, into a `let`. Moving work out of a loop like this is called hoisting, and that's the word the rest of this post uses.

```swift
private var hourlyChart: some View {
    let hours = hourly
    let maxTemp = maxHourlyTemp
    let minTemp = minHourlyTemp
    let showPrecips = showHourlyPrecips
    let dayCalendar = settingsManager.hourlyIsGrouped ? viewModel.weather?.forecast?.currentCalendar : nil

    return HStack(spacing: 2) {
        ForEach(hours.indices, id: \.self) { index in
            let hour = hours[index]

            if index > 0,
               let calendar = dayCalendar,
               let currentTime = hour.time,
               let previousTime = hours[index - 1].time,
               calendar.startOfDay(for: currentTime) != calendar.startOfDay(for: previousTime) {
                // Simplified: the real separator sits in a GeometryReader that pins it mid-glide.
                HourlyDaySeparatorView(day: currentTime)
            }

            Hour(max: maxTemp, min: minTemp, bar: hour, showPrecips: showPrecips)
        }
    }
}
```

Now each body pass does one array, one max, one min, one precipitation scan, and one calendar. Two details caught us:

- The snapshots use new names (`hours = hourly`), not Swift's shorthand for reusing a name. `let x = x` compiles inside `if let`, but not at declaration scope. Use a distinct name or `self.x`.
- Hoisting doesn't break SwiftUI's change tracking. SwiftUI re-runs the body when an observed object changes (`@ObservedObject`, `@EnvironmentObject`), not when an individual property changes. When the view model changes, the body re-runs and takes fresh snapshots.

That was one `ForEach` and one afternoon. Then three agents audited the app, watch, and widget targets for the same pattern and found it in **51 files**. It looked the same everywhere: settings pickers rebuilding a dictionary of localized names for every option, a radar legend measuring label widths for every label on every animation frame, a locations list decoding JSON three times per row, stats charts recomputing their ranges for every data point.

## Hoisting Rules, Learned the Hard Way

The sweep that fixed all 51 files went through two review rounds with seven independent reviewers, following [/adversarial-review-rounds/](/adversarial-review-rounds/). Their confirmed findings turned "hoist the invariants" from an instinct into rules. Each rule below exists because the obvious hoist was wrong somewhere.

**Hoist only what never changes in the loop AND is always evaluated.** Those are two separate tests. A sparkline view read its precipitation range only behind `if showsCurve`. Hoisting that read above the guard made it run for every daily row, including rows where the server's `min...max` bounds could be backwards, and a backwards `ClosedRange` crashes when it's built. If the original read sat behind a condition, the hoist keeps that condition, with placeholder values for the other branch, or it doesn't happen.

**Hoist INTO deferred builders, never out of them.** The content closures of `Menu` and `Picker` don't run when the body runs. They run when the menu opens. A names dictionary hoisted from inside a `Menu` builder up to body scope changes what the user sees: they now get a snapshot from whenever the body last ran, not from when they opened the menu. The second review round moved one of ours back inside each menu builder for this reason. The other direction is safe. Work already inside a deferred builder that runs once per option can move to the top of that builder, and it still costs nothing until the menu opens.

**Leave `onAppear` and action-closure reads alone.** Same rule, other direction. Those closures run after the body, so they need to see the world as it is when they fire, not a snapshot from body time.

**Pass optionals through. Don't replace them with `?? ""`.** Several hoists turned force-unwraps and dictionary lookups into parameters. Where the receiving view's parameter is `Optional`, pass the optional. With `nil`, SwiftUI skips the subview entirely. With `""`, it renders an empty view that still takes up layout space. Those are different view trees, so a diff that "just adds a fallback" changes what's on screen.

**Harden server-fed ranges while you're there.** Any `min...max` built from network data gets clamped (`lower...max(lower, upper)`), and any `stride` gets a `> 0` step guard. Review found a real stride-by-zero crash sitting next to one of the hoists.

So the "mechanical" sweep wasn't mechanical. Every one of those findings came from a reviewer who had no reason to want it to be simple.

## Green Everywhere, Merged Nowhere

By every check we had, the sweep was done. Build and unit tests were green at every commit. It had been through two review rounds, and every finding was settled and fixed. It was 51 files applying one pattern. We closed it anyway.

Nothing was wrong with it. But **51 files is more than a person can review with confidence.** Each review round had found new issues, which is good evidence that another round would too. "All checks passed" only measures what the checks measure. What matters at merge time is whether a human can vouch for the diff. For 51 files that are 90% mechanical and 10% judgment, we couldn't, because the 10% hides in the 90%.

So the sweep became a **patch source** instead of a merge candidate. We closed it, kept its branch on origin under a stable name, and wrote a plan that split it into 11 slices. Most slices were one to four files. The stats-chart and settings slices ran to eight and twelve. The slices were ordered, and each came with notes for whoever implements it. Two slices change behavior on purpose: the range clamping above, and a rewrite that stops force-unwrapping a nullable timestamp, so a missing time skips a separator instead of crashing. Both are flagged in the plan and **must say so in their PR descriptions**. Each slice either changes no behavior or says exactly what it changes.

The command that pulls a slice out matters more than it looks:

```
git diff origin/main...origin/fix/foreach-hoisting -- <this slice's paths>
```

That takes the **latest** state of each file from the banked branch, never the original sweep commit alone. The review-fix commits sit on top of the sweep, so the first commit by itself still has the bugs the reviewers caught. And because `main` keeps moving while slices land one at a time, each slice re-checks its files against fresh `main` before applying. Each slice PR cites the banked branch and the reviews it already passed, so reviewing a slice is cheap: the reviewer checks that the copy is faithful and the slice stands on its own, and doesn't re-argue the pattern.

## Landed So Far

**The watch strip** went first, pulled out of the slice order because it blocked a release. One file, copied unchanged from the banked branch; we checked the branch history to make sure the review-fix commits never touched it. Its PR states the expectation up front: dragging with a finger improves a lot, but the momentum glide after you let go stays juddery. The deceleration loop updates state from a `Task.sleep(16ms)` loop, and watchOS has no display link to line those ticks up with the screen refresh, so cheaper frames can't fix uneven presentation. The teammate who reported the choppiness wrote that expectation down before the hands-on device check. The check decides whether a separate paged-navigation rewrite (also built, also banked) ships or closes.

**The widget strips** went second: pure hoists in the hourly and minutely widget views, where body cost matters most because widgets render under a time limit. A read-only mini-review returned zero findings and one good nit. With the hoists, an empty data array now evaluates the three getters once instead of zero times. That's safe, and it's exactly the kind of edge case a per-slice review can hold in its head.

**Since then (as of September 2026).** Slice 3 landed the night before this post. The remaining slices (a twelfth was added when an audit of the banked diff found two minutely views unassigned) all landed the next day, each with a read-only mini-review posted to its PR description. We deleted the plan once the last slice merged. The paged-navigation rewrite was closed after a bake-off on real hardware. Instead, we replaced the momentum loop with a single spring retarget, the smallest change we could make on top of shipped code, and the strip shipped after hardware QA on TestFlight builds.

## The Audit Found More Than the Sweep Fixed

The same audit produced a Phase-2 list that the sweep leaves out on purpose, because these need design work rather than a `let`:

- **Per-row calendar chains.** Each hour column formats its timestamp by resolving a calendar and scanning the daily data for sun events. That's about 190 `Calendar` constructions and 72 linear scans per strip body evaluation, in every target that shares the row view. The fix is a calendar and a sun-event map computed once by the parent and passed down, which touches shared view APIs.
- **An O(24²) lookup.** A forecast helper builds its day view with `allHoursInDay.map { first(where:) }`, a linear search per hour. It needs a dictionary keyed by `Date`.
- **The root multiplier.** The localization helper builds a fresh `UserDefaults(suiteName:)` and `Locale` on *every string lookup*. Every "rebuilds the names dictionary per row" finding in the sweep was this cost times N. The plan flagged it as needing a cache that clears when the language changes, in its own careful PR.

Two of the three shipped the afternoon this post went up. The lookup became a `Date`-keyed dictionary with a contract test. The localization fix skipped cache invalidation entirely: it reads the language setting through a store that's already shared on every call, and caches only the table from identifier to `Locale`, which never changes. We parked the calendar chains. A re-check found that the app's hourly views re-run body about a dozen times per fling, not once per frame, so that cost is about a millisecond, and not often. It now waits for a profile trace or a user report rather than a count of operations from reading the code.

The sweep fixed the call sites, and the audit found the systems behind them. Hoisting removes repeated work at each site. The bigger win is making the expensive thing cheap once, so every call site gets it.

## Lessons Learned

- **SwiftUI won't cache a computed property for you.** A computed property is a function. A `let` at the top of the builder is the cache, and it's refreshed on every body pass.
- **Audit for the pattern, not the one instance.** One choppy drag became 51 files across three targets, plus a Phase-2 list of structural fixes that no single-site patch would have found.
- **Green and reviewed isn't the same as reviewable.** If every review round finds new issues, the change is too big to review. Split it.
- **A slice that changes behavior says so.** Splitting a "mechanical" change takes away the cover that word gave it. Any slice with a behavior change says so in its PR description, or the split didn't help.
- **Write the expected outcome before the hardware check.** "Tracking improves, judder remains", recorded in advance, makes the device check a real test instead of an impression.
