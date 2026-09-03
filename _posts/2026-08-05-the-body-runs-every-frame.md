---
layout: post
title: "The Body Runs Every Frame"
date: 2026-08-05 08:30:00 -0600
summary: "A choppy watch drag traced to computed properties re-read inside a ForEach on every body pass. The hoisting rules learned fixing it, and why a green, twice-reviewed 51-file fix was closed and re-landed in slices."
tags: [swiftui, ios, performance, workflow]
---

## The Problem

[Hello Weather](https://helloweather.com)'s watch app has an hourly strip: 24 columns, one per hour, that you drag sideways. On a real watch the drag was choppy. The cause applies to almost any SwiftUI list. While you drag, SwiftUI re-runs the view's `body` every frame, and the work inside that body was repeated for every one of the 24 columns.

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

Each one looks harmless on its own. Together they add up. `hourly[index]` builds a fresh array for every subscript. Each hour column reads `maxHourlyTemp`, `minHourlyTemp`, and `showHourlyPrecips`, and each of those takes a prefix of the data and maps over it. The check for a day boundary between two neighboring hours built a new `Calendar` for each comparison, 23 of them for 24 hours. That comes to **roughly 142 array allocations and 23 `Calendar` constructions per body evaluation**. During a drag, that means per frame, on a watch.

This is what builds up over four years when "add a computed var to the shared extension" is the easiest way to add a value. Nobody notices until a gesture makes the body run constantly.

## The Fix Is a `let`

SwiftUI does not cache a computed property. It runs the property's code every time it is read. A `let` at the top of a `@ViewBuilder` runs once per body pass. So the fix is to read each unchanging value once, above the loop, into a `let`. Moving work out of a loop this way is called hoisting, and the rest of this post uses that word.

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
- Hoisting does not break SwiftUI's change tracking. SwiftUI re-runs the body when an observed object changes (`@ObservedObject`, `@EnvironmentObject`), not when an individual property changes. When the view model changes, the body re-runs and takes fresh snapshots. A `let` inside body is as fresh as the body pass that created it.

That was one `ForEach` and one afternoon. Then three agents audited the app, watch, and widget targets for the same pattern and found it in **51 files**. The shape was the same everywhere: settings pickers rebuilding a dictionary of localized names for every option, a radar legend measuring label widths for every label on every animation frame, a locations list decoding JSON three times per row, stats charts recomputing their ranges for every data point.

## Hoisting Rules, Learned the Hard Way

The sweep that fixed all 51 files went through two review rounds with seven independent reviewers, following the practice in [/adversarial-review-rounds/](/adversarial-review-rounds/). The reviewers' confirmed findings turned "hoist the invariants" from an instinct into a set of rules. Each rule below exists because the obvious hoist was wrong.

**Hoist only what never changes in the loop AND is always evaluated.** Those are two separate tests. A sparkline view read its precipitation range only behind `if showsCurve`. Hoisting that read above the guard made it run for every daily row, including rows where the server's `min...max` bounds could be backwards, and a backwards `ClosedRange` crashes when it is built. The hoist made the view both slower and more likely to crash. If the original read sat behind a condition, the hoist keeps that condition, with placeholder values for the other branch, or it does not happen.

**Hoist INTO deferred builders, never out of them.** The content closures of `Menu` and `Picker` do not run when the body runs. They run when the menu opens. A names dictionary hoisted from inside a `Menu` builder up to body scope changes what the user sees: they now get a snapshot from whenever the body last ran, not from when they opened the menu. The second review round moved one of ours back inside each menu builder for this reason. The other direction is safe. Work already inside a deferred builder that runs once per option can move to the top of that builder, and it still costs nothing until the menu opens.

**Leave `onAppear` and action-closure reads alone.** Same principle, other direction. Those closures run after the body does. They must see the world as it is when they fire, not a snapshot from body time.

**Pass optionals through. Do not replace them with `?? ""`.** Several hoists turned force-unwraps and dictionary lookups into parameters. Where the receiving view's parameter is `Optional`, pass the optional. With `nil`, SwiftUI skips the subview entirely. With `""`, it renders an empty view that still takes up layout space. Those are different view trees, and a diff that "just adds a fallback" is a visible behavior change.

**Harden server-fed ranges while you are there.** Any `min...max` built from network data gets clamped (`lower...max(lower, upper)`), and any `stride` gets a `> 0` step guard. Review found a real stride-by-zero crash sitting next to one of the hoists.

A "mechanical" performance sweep is not mechanical. Every one of those findings came from a reviewer with no reason to want the sweep to be simple.

## Green Everywhere, Merged Nowhere

By every formal measure, the sweep was done. Build and unit-test gates were green at every commit. It had survived two review rounds, with every finding settled and fixed. It was 51 files of consistent change that applied one pattern. We closed it anyway.

Nothing was wrong with it. But **51 files is more than a person can review with confidence.** Each review round had found new issues, and that is the best evidence that another round would too. "All checks passed" only measures what the checks measure. The question at merge time is whether a human can vouch for the diff. For 51 files that are 90% mechanical and 10% judgment, the answer was no, because the 10% hides in the 90%.

So the sweep became a **patch source** instead of a merge candidate. We closed it, kept its branch on origin under a stable name, and wrote a plan that split it into 11 slices. Most slices were one to four files. The stats-chart and settings slices ran to eight and twelve. The slices were ordered, and each had notes with what an implementer needs to know. Two slices change behavior on purpose: the range clamping above, and a rewrite that stops force-unwrapping a nullable timestamp, so a missing time skips a separator instead of crashing. Both are flagged in the plan and **must say so in their PR descriptions**. Once a change is sliced, "mostly mechanical" is no longer an acceptable summary. Each slice either changes no behavior or says what it changes.

The extraction command matters more than it looks:

```
git diff origin/main...origin/fix/foreach-hoisting -- <this slice's paths>
```

That applies the **latest** state of each file from the banked branch, never the original sweep commit alone. The review-fix commits amend the sweep, so the first commit by itself contains exactly the bugs the reviewers caught. And since `main` moves while slices land one at a time, each slice re-checks its files against fresh `main` before applying. Each slice PR cites the banked branch and the reviews it already passed, so reviewing a slice is cheap. The reviewer confirms the copy is faithful and the slice stands on its own, and does not re-argue the pattern.

## Landed So Far

**The watch strip** went first, pulled out of the slice sequence into its own release-blocker lane. One file, the same bytes as the banked version, checked against branch history to confirm the review-fix commits never touched it. Its PR states the expectation up front: dragging with a finger improves a lot, but the momentum glide after you let go stays juddery. The deceleration loop updates state from a `Task.sleep(16ms)` loop, and watchOS has no display link to align those ticks with the screen refresh, so cheaper frames cannot fix uneven presentation. That expectation was written down before the hands-on device check by the teammate who reported the choppiness. The check decides whether a separate paged-navigation rewrite (also built, also banked) ships or closes. Recording the expected outcome first is what keeps the check honest.

**The widget strips** went second: pure hoists in the hourly and minutely widget views, where body cost matters most because widgets render under a time limit. A read-only mini-review returned zero findings and one good nit. With the hoists, an empty data array now evaluates the three getters once instead of zero times. That is safe, and it is exactly the kind of edge a per-slice review can hold in its head. That nit is the sliced approach working as designed.

**Since then (as of September 2026).** Slice 3 landed the night before this post. The remaining slices (a twelfth was added when an audit of the banked diff found two minutely views unassigned) all landed the next day, each with a read-only mini-review posted to its PR description. The plan was deleted rather than annotated once the last slice merged. The paged-navigation rewrite was closed after a hardware bake-off. The momentum loop was instead replaced in place with a single spring retarget, the smallest change on top of shipped code, and the strip shipped after hardware QA on TestFlight builds.

## The Audit Found More Than the Sweep Fixed

The same audit produced a Phase-2 list that the sweep leaves out on purpose, because these need design work rather than a `let`:

- **Per-row calendar chains.** Each hour column formats its timestamp by resolving a calendar and scanning the daily data for sun events. That is about 190 `Calendar` constructions and 72 linear scans per strip body evaluation, in every target that shares the row view. The fix is a calendar and a sun-event map computed once by the parent and passed down, which touches shared view APIs.
- **An O(24²) lookup.** A forecast helper builds its day view with `allHoursInDay.map { first(where:) }`, a linear search per hour. It needs a dictionary keyed by `Date`.
- **The root multiplier.** The localization helper builds a fresh `UserDefaults(suiteName:)` and `Locale` on *every string lookup*. Every "rebuilds the names dictionary per row" finding in the sweep was this cost times N. The plan flagged it as needing a cache that clears when the language changes, in its own careful PR.

Two of the three shipped the afternoon this post went up. The lookup became a `Date`-keyed dictionary with a contract test. The localization fix avoided cache invalidation entirely: it reads the language setting through a store that is already shared on every call, and caches only the table from identifier to `Locale`, which never changes. The calendar chains were parked. A re-check found that the app's hourly views re-run body about a dozen times per fling, not once per frame, so that cost is roughly a millisecond at low frequency. The item now waits for a profile trace or a user report rather than a count of operations from reading the code.

The sweep fixed the call sites. The audit found the systems behind them. Hoisting removes repeated work at each site; the bigger win is making the expensive thing cheap once, for everyone.

## Lessons Learned

- **SwiftUI will not cache a computed property for you.** A computed property is a function. A `let` at the top of the builder is the cache, and it is refreshed on every body pass.
- **Audit for the pattern, not the one instance.** One choppy drag became 51 files across three targets, plus a Phase-2 list of structural fixes that no single-site patch would have found.
- **Green and reviewed is not the same as reviewable.** If every review round finds new issues, the change is too big to review. Split it.
- **A slice that changes behavior says so.** Splitting a "mechanical" change removes the cover that word gave it. Any slice with a behavior change discloses it in the PR description, or the split did not help.
- **Write the expected outcome before the hardware check.** "Tracking improves, judder remains", recorded in advance, makes the device check a real test instead of an impression.

---

## How This Post Was Made

**Prompt 1:** "see recent work in ~/Code/helloweather, perhaps a blog post about our opus 4.8 agents and why we decided to do that? perhaps something about the swift testing + snapshots inspired by minitest-snapshots? anything else? bring me a list of potential post ideas for review."

**Prompt 2:** "skip 4, 5, 6, 9 but create posts for each of the others in the 1-9 list. also add Four Answers to One Question, and Write the Rule, Not the Story -- show me a concise version of your plan and then I can approve" — then "proceed, one pr per post"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. This was a light pass: stacked sentences were split, a few flourishes cut, the merge-call argument tightened, and Lessons Learned went from eight bullets to five by dropping the ones that repeated a heading. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The first code block now matches the shared extension as it stood before the sweep (`Forecast.Hourly.Hour`, the feels-like branch, `showHourlyPrecips` scanning `hourly`), and the second uses the real separator view name with a note about the omitted pinning geometry. The slice sizes were corrected (the stats-chart and settings slices were eight and twelve files, not one to four), the extraction command now matches the plan's `origin/` form, the hardware check is attributed to the teammate who reported the bug rather than the project owner, and two dated paragraphs record what happened after publication: all slices landed the next day, the paged rewrite closed banked, and two of the three Phase-2 items shipped while the third was parked.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. One Claude Fable 5.1 agent wrote a three-line plain explanation of the post first, then rewrote the prose from it. "Computed property", "hoisting", and "deferred builder" are now defined where they first appear; "byte-identical" became "the same bytes"; "adversarial" stays only in the link to the review-rounds post; "surface" as a verb, "crash surface", and the "floor and ceiling" closer are gone; "PR body" became "PR description" throughout; and sentences that carried two facts were split. Judgment calls: the summary line was reworded to drop "loop-invariant" and "per ForEach element per body pass"; the seven reviewers are described as independent rather than adversarial, which is what the linked post means by the word. Code blocks, headings, dates, numbers, links, and the earlier meta paragraphs are unchanged. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"
