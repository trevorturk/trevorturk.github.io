---
layout: post
title: "Four Answers to One Question"
date: 2026-08-05 08:50:00 -0600
summary: "The app answered 'is this rain chance worth showing?' four different ways in four places. We replaced them with one function, a rule that keeps it to one, and a review that proved nothing was missed."
tags: [swift, ios, architecture, code-review]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Screenshot

It started with a screenshot of [Hello Weather](https://helloweather.com). The hourly strip showed a row of 20-25% rain-chance labels, and the "precip later" pill, the small button that announces rain coming later, was nowhere on the screen. Two parts of one screen disagreed about whether it was going to rain.

Neither one was broken. Each was following its own rule for "is this probability worth showing?", and the codebase had four rules:

1. **Hourly and daily labels** (app, watch, widgets) rounded first and then compared to the floor. A raw 17.5% became "20%" and got a bar.
2. **The precip-later pill** compared the raw value against a higher floor. Its suppression clause, the check that keeps the pill from covering a rendered bar, also used raw values, so an hour could show a "20%" bar with the pill sitting on top of it.
3. **One detail card** used a third cutoff, lower than the other two, on the raw value. The card could say "None" while the strip showed a labeled value for the same hour.
4. **Push notification copy** checked the raw value but printed the rounded one. A value could pass the check and then round down, or fail the check while the same rounded number showed fine elsewhere in the app.

(The numbers here, a 5% rounding step and a 20% floor, are ours. Nothing below depends on them.)

None of these were careless. The pill's higher floor was a deliberate choice to stop advertising marginal rain. The detail card's cutoff was older than the rounding helper. The push code compared the value it had in hand. The four drifted apart because each one changed on its own and nothing tied them together. Two real bugs had already shipped from this group of thresholds before the screenshot: the raw-vs-rounded mismatch in the suppression clause, and a pill that stopped staying clear of rendered bars after one floor moved and the other didn't.

## The Chokepoint

We could have set the four constants to the same number. That would fix the screenshot and leave the four sites free to drift again the next time one of them changed. Instead we made one place that answers the question:

```swift
enum PrecipDisplay {
    static func showPrecip(val: Float) -> Bool {
        val.toRoundedPrecip >= 0.2
    }
}

extension BinaryFloatingPoint {
    var toRoundedPrecip: Self {
        self.rounded(to: 0.05)
    }
}
```

The function is pure, lives in a shared extensions file, and compiles into all four targets (app, widgets, watch app, watch extension). It's called from the hourly and daily labels, bar colors, the pill's trigger and its headline scan, VoiceOver's per-hour precipitation clause, push copy, the stat cards' on/off states and icons, the detail charts' point icons, and both lock-screen slots. Forty-four call sites, by the PR's count.

Two choices in that small function do most of the work.

First, every caller passes the raw value and the function rounds it. Callers don't choose which version to compare, so the raw-vs-rounded bug can't happen. Rounding an already-rounded value gives the same value back, so it doesn't matter if a caller's number has already been through the display formatter. A test checks that instead of assuming it:

```swift
@Test func agreesForRawAndPreRoundedInputs() {
    for raw in stride(from: Float(0), through: 1, by: 0.001) {
        #expect(PrecipDisplay.showPrecip(val: raw) ==
                PrecipDisplay.showPrecip(val: raw.toRoundedPrecip))
    }
}
```

Boundary tests next to it pin the rounding cliff (0.174 hides, 0.175 shows). The sweep test matters more, because it's the proof that "pass whatever you have" is safe.

Second, the threshold belongs to a named type instead of being a loose constant. `PrecipDisplay` is an enum with no cases. It's only a namespace. That looks like ceremony until you read the rule that shipped with it:

> Any future second threshold must land as a named `PrecipDisplay` member, never an inline constant at a call site.

The rule keeps the four from drifting apart again. The drift didn't start as four thresholds. It started as one threshold and a reasonable-looking inline `0.3` at one call site. The rule allows a second threshold, because the product may need one. It doesn't allow an anonymous one. A named member sits next to its sibling, gets reviewed as a deliberate fork, and can be found by anyone auditing the group. An inline constant at a call site is invisible until it shows up in a screenshot.

## Writing the Rule Where Review Will Trip Over It

A rule that only lives in a commit message lasts until the commit scrolls off the first page of `git log`. This one went into the code-review skill, the checklist the review agent walks on every PR, as two rewritten bullets. Here they are as of September 2026, sanitized and trimmed of PR numbers and the geometry caveats. The `significantPrecip` clauses were added later; more on that below.

> - [ ] Precip *display* gates (hourly/daily labels and bar colors, the precip-later button, VoiceOver per-hour precip clauses, push-copy precip lines, precip stat card/detail on-off states and icons) route through `PrecipDisplay.showPrecip` (rounded ≥ 0.2) - new or modified display gates must call it, never hardcode a literal - and hourly-strip *window* decisions (lane, bars, pill) route through `PrecipDisplay.significantPrecip`. Deliberate exceptions (relevance thresholds, the chart-summary `> 0` accessibility gates, and the frozen style catalogs in all three targets) are inventoried in the unification plan.
>
> - [ ] The button must never cover a rendered precip bar. Since the unification, bars, labels, and the button share **one per-hour floor**: `PrecipDisplay.showPrecip` (rounded ≥ 20%). One **window-significance** predicate sits above it, `PrecipDisplay.significantPrecip`, so lane, bars, and button live or die together. Within a significant window the single per-hour floor means "first `showPrecip` index ≥ visible-column threshold" alone guarantees no rendered bar sits under the button; in an insignificant window nothing renders at all - there is deliberately no separate suppression clause. **If the per-hour floors ever diverge** (the significance rule is a window shape test, not a second per-hour floor), the divergence must land as a named `PrecipDisplay` member and the explicit suppression clause - rounded on BOTH sides; the raw/rounded split shipped a bug once - becomes load-bearing again.

The second bullet says what has to come back if the floors ever split again. With one floor, the pill's old suppression clause was redundant. "The first qualifying hour is past the visible window" and "no rendered bar in the visible window" mean the same thing when bars and pill use the same test. So we deleted the clause and sketched the proof in the PR. That only holds while there's one floor. The checklist doesn't just say we deleted it. It says the clause comes back, rounded on both sides, the day the floors split, so the next person doesn't ship the old bug again.

We did the same for a decision that got overruled along the way. The designer had proposed two thresholds: forgiving display gates, and a stricter pill that only fires when rain is worth interrupting for. We shipped one uniform floor instead, because it was the simpler rule and we could adjust once we had field evidence. The designer's version wasn't thrown away. It's in the plan doc as the ready alternative, built on a branch, with instructions that bringing it back means a named member and the restored suppression clause. Neither side had field evidence, so the plan lists the two options as equals rather than as default and exception. If we hadn't written it down, we'd have had the same debate again from scratch in six months.

The evidence arrived at the end of August. A single 20% hour with no icon rendered one bar and a pill, and the owner and designer agreed that was misleading. The problem was the shape of the window, one weak hour on its own, not the floor. The fix landed on 2026-08-31 the way the rule requires, as a named member, `PrecipDisplay.significantPrecip`. It's a window-level test that hides the hourly strip's lane, bars, and pill when the only qualifying hour in a window of 12 hours or more rounds below 30%. The per-hour floor didn't change. The labels and cards still gate per hour, and the short lock-screen strips stay on `showPrecip` on purpose, with that listed in the plan.

## The Feature That Had Never Rendered

Routing every comparison through one function means reading every comparison, and one of them didn't make sense.

The rectangular lock-screen widget and the matching watch complication have a slot where, on rainy hours, the precip percentage replaces the temperature. The check for it compared the probability, a fraction from 0 to 1, against a whole-number percent:

```swift
// before: precipProbabilityValue is the rounded 0.0-1.0 probability
if precipProbabilityValue >= 20 { ... }
```

A fraction is never ≥ 20. The check was always false, so the slot had never shown for anyone since the day it shipped. Nobody noticed, because a check that's always false looks the same as "it hasn't rained lately." Routing the site through the chokepoint fixed it, because the chokepoint takes fractions, and the slot rendered for the first time. That turned up one more problem. Localized percent strings like "100 %" overflow a 31-point slot, so the label got a `minimumScaleFactor`. A feature that has never rendered has never been through layout QA either.

This is the best practical argument for the chokepoint. A function with one signature also fixes the units. Four scattered comparisons can each pick their own units and each be wrong. Forty-four call sites passing a `Float` to one function can't.

## What Deliberately Stayed Out

A sweep like this can fail two ways: missing a site, or pulling in a site that only looks like the same question. The second is harder to spot. Not every comparison against a probability is a display gate:

- **Relevance and ranking thresholds** answer "does this matter right now?", not "should this value be shown?" The filter that decides whether precip is worth mentioning to an AI summarizer and the watch smart stack's relevance scores are in this group. They use higher floors on purpose.
- **Accessibility chart summaries** check for `> 0`, which is wider than the display floor on purpose. A VoiceOver description of a whole chart ("up to N percent") should describe what the chart plots, and the detail charts plot values below the floor.
- **A frozen catalog of old forecast styles** has dozens of per-style thresholds. It sits behind a debug-only setting, and policy says its bytes don't change. Rewriting frozen code to fit a new convention un-freezes it.

Leaving them out is the easy part. Each one is also listed in the plan doc the checklist bullet points at, with the reason it's a different question. Without that list, a reader can't tell an exemption from a site the sweep missed.

## Proving the Chokepoint Is Actually Complete

"We routed everything through one function" is a claim, and the session that did the routing can't see its own blind spots. So before merge, the change went through our [adversarial review process](/adversarial-review-rounds/): four independent, read-only reviewers, none of them with the session's reasoning.

One reviewer's job wasn't "review this diff." It was "prove or refute the claim that this is complete": find every comparison against a probability in the codebase and account for each one. It found four outside the frozen style catalog, the chokepoint plus the three relevance-scoring exemptions from the plan, and nothing else. That's a list checked against the inventory, not "the diff looks complete."

That's a different kind of assurance, and it's cheap to ask for. The main claim of a unification PR is that no other site answers this question. A diff review can't check that, because any site that breaks the claim isn't in the diff. So give one reviewer the whole codebase and that claim as its job.

The other reviewers found real things too. One traced boundary values through every screen and confirmed with predicate math that deleting the suppression clause was safe. It found two bugs, and both were fixed before merge: a gap in how the pill's headline scan picked its anchor hour, and an off-by-one where the "starting soon" copy named the precip type of the hour after the match ("rain possible in 3h" for what was actually snow). One known gap stayed open. The column counts for wide layouts on tablets and landscape phones are hand-maintained estimates, not measured, and the plans record that as deferred, with measuring them named as the real fix.

## Lessons Learned

- **Take raw inputs and round inside.** If callers choose which version to compare, some will get it wrong. Add a test that raw and pre-rounded inputs give the same answer.
- **Forbid the anonymous threshold, not the second one.** A rule that any new floor must be a named member of one type makes drift visible in review without blocking change.
- **One function signature fixes the units.** Scattered comparisons can each be wrong in their own units. Expect a sweep to find at least one that never worked.
- **When you delete redundant code, write down what brings it back.** The condition, the form it must take, and the old bug it guards against, in the checklist reviewers read.
- **Record the design that lost, as an equal.** A variant set aside without field evidence belongs in the plan as a built alternative, so the next round starts there instead of from scratch.
