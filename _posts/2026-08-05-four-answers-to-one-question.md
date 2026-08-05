---
layout: post
title: "Four Answers to One Question"
date: 2026-08-05 08:50:00 -0600
summary: "A codebase answered 'is this precipitation probability worth showing?' four different ways in four places - the fix was one pure chokepoint function, a rule that keeps it singular, and a sweep that proved it complete."
tags: [swift, architecture, code-review]
---

## The Screenshot

The trigger was a screenshot of [Hello Weather](https://helloweather.com): an hourly strip showing a row of 20-25% precipitation-probability labels, and no "precip later" pill anywhere on screen. The labels were advertising rain that the pill refused to mention. Two UI elements, same forecast, same screen, disagreeing about whether it was going to rain.

That's not a bug in either element. Each one was correctly implementing its own answer to the question "is this probability worth showing?" The bug was that the codebase had four answers:

1. **Hourly and daily labels** (app, watch, widgets) showed precip when the *rounded* probability cleared the floor - so a raw 17.5% rendered as "20%" and got a bar.
2. **The precip-later pill** triggered on the *raw* value at a *higher* floor, and its don't-cover-a-rendered-bar suppression window was also computed on raw values - so an hour could render a "20%" bar without suppressing the pill that floated over it.
3. **One detail card** had its own raw-value "off" band at a *third* number, below both of the others - so a probability could simultaneously be "None" on the card and a labeled value in the strip.
4. **Push notification copy** gated on the raw value while *displaying* the rounded one - the same string could pass the gate and round down, or fail the gate while an identical-looking value elsewhere displayed fine.

(The specific numbers - a 5% rounding step, a 20% floor - are ours and are illustrative. Nothing below depends on them; the pattern is what transfers.)

None of these were written carelessly. Each divergence had a local justification at the time it was introduced: the pill's higher floor came from a deliberate decision to stop advertising marginal rain; the detail card's band predated the rounding helper; the push gate was written against the value it had in hand. That's the point worth internalizing: **when the same product question is answered in multiple places, the answers don't diverge because someone was sloppy. They diverge because each site evolves under local pressure, and nothing ties them together.** This family of duplicated thresholds had already shipped at least two real bugs before the screenshot - a raw-vs-rounded mismatch on the suppression window, and a pill that stopped suppressing over rendered bars after one floor moved and the other didn't.

## The Chokepoint

The fix is not "align the constants." Aligning the constants leaves four call sites that can drift again the day any one of them changes. The fix is making the question answerable in exactly one place:

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

That's the entire mechanism: a pure function in a shared extensions file, compiled into all four targets (app, watch, widgets, push service). Every display gate - hourly and daily labels, bar colors, the pill's trigger and its headline scan, VoiceOver's per-hour precipitation clause, push copy, the stat cards' on/off states and icons, the detail charts' point icons, and both lock-screen slots - routes through it. Forty-four call sites.

Two design choices in that tiny function do most of the work:

**All call sites pass raw values; the chokepoint rounds internally.** This kills the raw-vs-rounded bug class outright, because no caller gets to choose which representation to compare. The rounding is idempotent - `toRoundedPrecip` of an already-rounded value is itself - so it doesn't matter whether a caller's value has been through the display formatter already. That property is not assumed; it's test-pinned:

```swift
@Test func agreesForRawAndPreRoundedInputs() {
    for raw in stride(from: Float(0), through: 1, by: 0.001) {
        #expect(PrecipDisplay.showPrecip(val: raw) ==
                PrecipDisplay.showPrecip(val: raw.toRoundedPrecip))
    }
}
```

Alongside it, boundary tests pin the exact rounding cliff (0.174 hides, 0.175 shows). The sweep test is the important one, though: it's the executable statement of the invariant that makes "pass whatever you have" safe.

**The threshold is a member of a named type, not a constant.** `PrecipDisplay` is an enum with no cases - it exists purely as a namespace. That sounds like ceremony until you read the standing rule that shipped with it:

> Any future second threshold must land as a named `PrecipDisplay` member, never an inline constant at a call site.

This is the part that keeps the unification unified. The original divergence didn't start as four thresholds; it started as one threshold and then a perfectly reasonable inline `0.3` at one call site. The rule doesn't forbid a second threshold - product reality may well demand one - it forbids an *anonymous* one. A named member sits next to its sibling, gets reviewed as a deliberate fork of the question, and is findable by anyone auditing the family. An inline constant at a call site is invisible until it ships a screenshot.

## Writing the Rule Where Review Will Trip Over It

A rule that lives only in a commit message is a rule that lasts until the commit scrolls off the first page of `git log`. This one landed in the code-review skill - the checklist the review agent walks on every PR - as two rewritten bullets. Sanitized but structurally intact:

> - [ ] Precip *display* gates (hourly/daily labels and bar colors, the precip-later button, VoiceOver per-hour precip clauses, push-copy precip lines, stat card/detail on-off states and icons) route through `PrecipDisplay.showPrecip` - new or modified display gates must call it, never hardcode a literal. Deliberate exceptions (relevance thresholds, the chart-summary `> 0` accessibility gates, the frozen style catalogs) are inventoried in the unification plan.
>
> - [ ] The precip-later button must never cover a rendered bar. Since the unification, bars, labels, and the button share **one floor**, so "first qualifying index ≥ visible-column threshold" alone guarantees no rendered bar sits under it - there is deliberately no separate suppression clause. **If a second floor ever returns**, it must land as a named `PrecipDisplay` member, and the explicit suppression clause - rounded on BOTH sides; the raw/rounded split shipped a bug once - becomes load-bearing again.

The second bullet is doing something we've come to think of as essential to any simplification: it records **what becomes load-bearing again if the simplification is ever reversed**. With one predicate, the pill's old on-screen suppression clause is provably redundant - "the first qualifying hour is past the visible window" and "no rendered bar in the visible window" are the same statement when bars and pill share a predicate - so it was deleted, with the proof sketched in the PR. But that equivalence *only* holds while there's one floor. The checklist doesn't just say "we deleted X"; it says "X returns, in this exact form, with rounding on both sides, the day the floors fork." Deleting code is easy. Deleting code while leaving behind an accurate map of the conditions under which it must come back is the version that doesn't cost your successors a shipped bug.

The same discipline applied to a decision that got superseded along the way. The designer had proposed a two-threshold model for the pill - display gates forgiving, the attention-grabbing pill stricter, so it only fires on evidence worth interrupting for. The shipped change went uniform instead: simplest rule first, tweak with field evidence in hand. But the designer's variant wasn't discarded - it's preserved in the plan doc as the ready alternative, pre-built on a branch, with explicit instructions that reinstating it means a *named* member and the restored suppression clause. Neither side had field evidence, so the plan records the two options as equal-footing, not default-vs-exception. Superseding a design verdict without recording it is how the same debate gets re-litigated from scratch in six months.

## The Feature That Had Never Rendered

Here's the part that makes threshold unification more than hygiene. Sweeping every comparison site into one function forces you to actually *read* every comparison site - and one of them didn't parse.

The lock-screen rectangular widget and the matching watch complication have a designed slot: on rainy hours, the precip percentage replaces the temperature. Nice touch. The gate for it compared the probability - a 0-to-1 fraction - against an integer percent:

```swift
// before: probability is 0.0-1.0; the gate wants "20"
if hour.precipProbability >= 20 { ... }
```

A fraction is never ≥ 20. The gate was always false. The slot had **never rendered, for anyone, since the day it shipped** - and nobody noticed, because "the gate is always false" is visually indistinguishable from "it hasn't rained lately." A units mismatch with a plausible-looking failure mode is the quietest bug there is. Routing the site through the chokepoint fixed it as a side effect - the chokepoint takes fractions, full stop - and the feature rendered for the first time. (Which produced its own rider: localized percent strings like "100 %" overflow a 31-point slot, hence a `minimumScaleFactor` on the label. Features that have never rendered have also never been through layout QA.)

This is the strongest practical argument for the chokepoint pattern: **a function with one signature is also a units contract.** Four scattered comparisons can each pick their own units and be wrong independently. Forty-four call sites feeding one `Float`-taking function cannot.

## What Deliberately Stayed Out

An exercise like this fails in two directions: leaving divergent sites out, and sweeping in sites that only *look* like the same question. The second failure is subtler. Not every fractional comparison against a probability is a display gate:

- **Relevance and ranking thresholds** - the filter deciding whether precip is significant enough to mention to an AI summarizer, and the watch smart-stack's relevance scores - answer "does this matter right now?", not "should this value be shown?" Different question, deliberately different (and higher) floors.
- **Accessibility chart summaries** gate on `> 0`, deliberately broader than the display floor: a VoiceOver description of a whole chart ("up to N percent") should describe data the chart plots, and the detail charts plot sub-threshold values.
- **A frozen catalog of legacy forecast styles** carries dozens of per-style thresholds. It's frozen - behind a debug-only setting, byte-stable by policy - and rewriting frozen code to satisfy a new convention is how you un-freeze it by accident.

The important move isn't excluding them - it's that the exclusions are *inventoried*, in the plan doc the checklist bullet points at, each with the reason it's a different question. An undocumented exemption is indistinguishable from a site the sweep missed. A documented one is a decision.

## Proving the Chokepoint Is Actually Complete

"We routed everything through one function" is a claim, and claims from the session that did the routing are worth exactly as much as any other self-review - which is to say, they inherit every blind spot of the authoring context. So before merge, the change went through the [adversarial review process](/adversarial-review-rounds/): four independent, read-only reviewers, none with the session's reasoning.

The one worth singling out here was an **exhaustive-sweep reviewer** whose brief was not "review this diff" but "prove or refute the completeness claim": find every fractional probability comparison in the codebase and account for it. Its report: outside the frozen style catalog, exactly four such comparisons exist - the chokepoint, plus the three documented relevance-scoring exemptions. Not "the diff looks complete." An enumeration, checked against the inventory, with zero unaccounted-for sites.

That's a materially different kind of assurance, and it's cheap to ask for. A unification PR's central claim is universally quantified - *no* other site answers this question - and a diff review can't verify a universal claim because the sites that falsify it are, by definition, not in the diff. Give one reviewer the whole codebase and the quantifier as its brief.

The other lenses earned their keep too: a behavioral pass traced boundary values through every surface and confirmed the suppression-clause deletion with predicate math, and its two confirmed findings (an anchor-selection gap in the pill's headline scan, and an off-by-one where the "starting soon" copy announced the precip *type* of the hour after the match - "rain possible in 3h" for what was actually snow) were fixed before merge. Known geometry caveats - wide-layout column counts on tablets and landscape phones are hand-maintained estimates rather than measured - were recorded in the plans as deferred, with the measurement rework named as the real fix.

## Lessons Learned

- **Duplicated answers to one question will diverge.** Not might - will. Each site evolves under local pressure with local justification, and no individual change looks wrong. The screenshot where two elements disagree on the same screen is the end state, not an anomaly.
- **Align the place, not the constants.** Making four sites agree on a number fixes today's screenshot and leaves tomorrow's drift fully armed. The durable fix is one pure function that every site calls - after which the constants *can't* disagree.
- **Take raw inputs; normalize inside; pin idempotence.** Letting call sites choose the representation they compare is how raw-vs-rounded bugs happen. One signature, internal rounding, and a sweep test asserting raw and pre-rounded inputs always agree.
- **Write the rule for the second threshold before anyone wants one.** The unification's real enemy is the future inline constant. "Any new floor must be a named member of this type" turns silent drift into a reviewable, findable decision - it forbids anonymity, not evolution.
- **A chokepoint is a units contract.** Our sweep surfaced a designed feature that had never once rendered because its gate compared a fraction to a percent. Scattered comparisons can each be wrong in their own units; one function signature can't.
- **When you delete redundant code, record what resurrects it.** The suppression clause was provably redundant *given one floor*. The checklist now says exactly what becomes load-bearing again if a second floor returns, in what form, with which past bug as the warning. That sentence is the cheap insurance.
- **Inventory the exemptions.** Relevance scores, accessibility gates, and frozen catalogs answer different questions and stayed out - in writing, with reasons. An undocumented exemption is indistinguishable from a missed site.
- **Completeness claims need a completeness reviewer.** "Everything routes through the chokepoint" is universally quantified, and a diff can't prove a universal. One fresh-context reviewer with the whole repo and the quantifier as its brief turned the claim into an enumeration: exactly four comparisons, all accounted for.
- **Record the superseded design, not just the shipped one.** The stricter-pill variant lost the first round without field evidence either way. It lives in the plan as an equal-footing, pre-built alternative - so the future decision starts from the recorded debate instead of re-deriving it.

---

## How This Post Was Made

**Prompt 1:** "see recent work in ~/Code/helloweather, perhaps a blog post about our opus 4.8 agents and why we decided to do that? perhaps something about the swift testing + snapshots inspired by minitest-snapshots? anything else? bring me a list of potential post ideas for review."

**Prompt 2:** "skip 4, 5, 6, 9 but create posts for each of the others in the 1-9 list. also add Four Answers to One Question, and Write the Rule, Not the Story -- show me a concise version of your plan and then I can approve" — then "proceed, one pr per post"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.
