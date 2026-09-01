---
layout: post
title: "Four Answers to One Question"
date: 2026-08-05 08:50:00 -0600
summary: "A codebase answered 'is this precipitation probability worth showing?' four different ways in four places - the fix was one pure chokepoint function, a rule that keeps it singular, and a sweep that proved it complete."
tags: [swift, ios, architecture, code-review]
---

## The Screenshot

The trigger was a screenshot of [Hello Weather](https://helloweather.com): an hourly strip showing a row of 20-25% precipitation-probability labels, and no "precip later" pill anywhere on screen. The labels were advertising rain that the pill refused to mention. Two elements, one forecast, one screen, disagreeing about whether it was going to rain.

Neither element had a bug. Each was correctly implementing its own answer to "is this probability worth showing?" The codebase had four answers:

1. **Hourly and daily labels** (app, watch, widgets) showed precip when the *rounded* probability cleared the floor, so a raw 17.5% rendered as "20%" and got a bar.
2. **The precip-later pill** triggered on the *raw* value at a *higher* floor. Its don't-cover-a-rendered-bar suppression window was also computed on raw values, so an hour could render a "20%" bar without suppressing the pill floating over it.
3. **One detail card** had its own raw-value "off" band at a *third* number, below both of the others. A probability could be "None" on the card and a labeled value in the strip at the same time.
4. **Push notification copy** gated on the raw value while *displaying* the rounded one. The same string could pass the gate and round down, or fail the gate while an identical-looking value elsewhere displayed fine.

(The specific numbers - a 5% rounding step, a 20% floor - are ours and are illustrative. Nothing below depends on them; the pattern is what transfers.)

None of these were written carelessly. The pill's higher floor came from a deliberate decision to stop advertising marginal rain. The detail card's band predated the rounding helper. The push gate was written against the value it had in hand. When one product question is answered in several places, the answers diverge because each site evolves under local pressure and nothing ties them together, not because someone was sloppy. This family of thresholds had already shipped at least two real bugs before the screenshot: a raw-vs-rounded mismatch on the suppression window, and a pill that stopped suppressing over rendered bars after one floor moved and the other didn't.

## The Chokepoint

Aligning the constants would fix the screenshot and leave four call sites free to drift again the day any one of them changes. The fix is making the question answerable in exactly one place:

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

That is the whole mechanism: a pure function in a shared extensions file, compiled into all four targets (app, widgets, watch app, watch extension). Every display gate routes through it: hourly and daily labels, bar colors, the pill's trigger and its headline scan, VoiceOver's per-hour precipitation clause, push copy, the stat cards' on/off states and icons, the detail charts' point icons, and both lock-screen slots. Forty-four call sites, by the PR's count.

Two design choices in that small function do most of the work.

Every call site passes a raw value, and the chokepoint rounds internally. No caller gets to choose which representation to compare, which removes the raw-vs-rounded bug class outright. The rounding is idempotent, since `toRoundedPrecip` of an already-rounded value is itself, so it does not matter whether a caller's value has already been through the display formatter. That property is test-pinned rather than assumed:

```swift
@Test func agreesForRawAndPreRoundedInputs() {
    for raw in stride(from: Float(0), through: 1, by: 0.001) {
        #expect(PrecipDisplay.showPrecip(val: raw) ==
                PrecipDisplay.showPrecip(val: raw.toRoundedPrecip))
    }
}
```

Boundary tests alongside it pin the exact rounding cliff (0.174 hides, 0.175 shows). The sweep test matters more: it is the executable statement of the invariant that makes "pass whatever you have" safe.

The threshold is a member of a named type, not a loose constant. `PrecipDisplay` is an enum with no cases; it exists purely as a namespace. That looks like ceremony until you read the standing rule that shipped with it:

> Any future second threshold must land as a named `PrecipDisplay` member, never an inline constant at a call site.

This is what keeps the unification unified. The original divergence did not start as four thresholds. It started as one threshold and a perfectly reasonable inline `0.3` at one call site. The rule does not forbid a second threshold, since product reality may demand one. It forbids an *anonymous* one. A named member sits next to its sibling, gets reviewed as a deliberate fork of the question, and can be found by anyone auditing the family. An inline constant at a call site is invisible until it ships a screenshot.

## Writing the Rule Where Review Will Trip Over It

A rule that lives only in a commit message lasts until the commit scrolls off the first page of `git log`. This one landed in the code-review skill, the checklist the review agent walks on every PR, as two rewritten bullets. Their wording as of September 2026, sanitized and trimmed of PR numbers and the geometry caveats (the `significantPrecip` clauses were added later; more on that below):

> - [ ] Precip *display* gates (hourly/daily labels and bar colors, the precip-later button, VoiceOver per-hour precip clauses, push-copy precip lines, precip stat card/detail on-off states and icons) route through `PrecipDisplay.showPrecip` (rounded ≥ 0.2) - new or modified display gates must call it, never hardcode a literal - and hourly-strip *window* decisions (lane, bars, pill) route through `PrecipDisplay.significantPrecip`. Deliberate exceptions (relevance thresholds, the chart-summary `> 0` accessibility gates, and the frozen style catalogs in all three targets) are inventoried in the unification plan.
>
> - [ ] The button must never cover a rendered precip bar. Since the unification, bars, labels, and the button share **one per-hour floor**: `PrecipDisplay.showPrecip` (rounded ≥ 20%). One **window-significance** predicate sits above it, `PrecipDisplay.significantPrecip`, so lane, bars, and button live or die together. Within a significant window the single per-hour floor means "first `showPrecip` index ≥ visible-column threshold" alone guarantees no rendered bar sits under the button; in an insignificant window nothing renders at all - there is deliberately no separate suppression clause. **If the per-hour floors ever diverge** (the significance rule is a window shape test, not a second per-hour floor), the divergence must land as a named `PrecipDisplay` member and the explicit suppression clause - rounded on BOTH sides; the raw/rounded split shipped a bug once - becomes load-bearing again.

The second bullet records what has to come back if the simplification is ever reversed. With one predicate, the pill's old on-screen suppression clause is provably redundant: "the first qualifying hour is past the visible window" and "no rendered bar in the visible window" are the same statement when bars and pill share a predicate. So it was deleted, with the proof sketched in the PR. That equivalence only holds while there is one floor. The checklist does not just say "we deleted X." It says X returns, in this exact form, with rounding on both sides, the day the floors fork. Deleting code is easy. Deleting it while leaving an accurate map of the conditions under which it must return is the version that does not cost a successor a shipped bug.

The same discipline applied to a decision that was superseded along the way. The designer had proposed a two-threshold model: forgiving display gates, and a stricter pill that only fires on evidence worth interrupting for. The shipped change went uniform instead, simplest rule first, to be tweaked with field evidence in hand. The designer's variant was not discarded. It is preserved in the plan doc as the ready alternative, pre-built on a branch, with explicit instructions that reinstating it means a *named* member and the restored suppression clause. Neither side had field evidence, so the plan records the two options as equal-footing rather than default-vs-exception. Superseding a design verdict without recording it is how the same debate gets re-litigated from scratch in six months.

The evidence arrived at the end of August: a lone icon-less 20% hour rendered one bar and a pill, and the owner and designer agreed it was misleading. It pointed at a shape, a weak singleton, not a floor. The fix landed on 2026-08-31 the way the rule demands: a named member, `PrecipDisplay.significantPrecip`, a window-level test that hides the hourly strip's lane, bars, and pill when the only qualifying hour in a 12-hour-plus window rounds below 30%. The per-hour floor is untouched, the data surfaces still gate per hour, and the short lock-screen strips deliberately stay on `showPrecip`, inventoried in the plan.

## The Feature That Had Never Rendered

Sweeping every comparison site into one function forces you to read every comparison site, and one of them did not parse.

The lock-screen rectangular widget and the matching watch complication have a designed slot: on rainy hours, the precip percentage replaces the temperature. The gate for it compared the probability, a 0-to-1 fraction, against an integer percent:

```swift
// before: precipProbabilityValue is the rounded 0.0-1.0 probability
if precipProbabilityValue >= 20 { ... }
```

A fraction is never ≥ 20. The gate was always false, so the slot had never rendered, for anyone, since the day it shipped. Nobody noticed, because "the gate is always false" looks exactly like "it hasn't rained lately." Routing the site through the chokepoint fixed it as a side effect, since the chokepoint takes fractions, and the feature rendered for the first time. That produced its own rider: localized percent strings like "100 %" overflow a 31-point slot, hence a `minimumScaleFactor` on the label. A feature that has never rendered has also never been through layout QA.

This is the strongest practical argument for the chokepoint: a function with one signature is also a units contract. Four scattered comparisons can each pick their own units and be wrong independently. Forty-four call sites feeding one `Float`-taking function cannot.

## What Deliberately Stayed Out

An exercise like this fails in two directions: leaving divergent sites out, and sweeping in sites that only *look* like the same question. The second is subtler. Not every fractional comparison against a probability is a display gate:

- **Relevance and ranking thresholds**, the filter deciding whether precip is significant enough to mention to an AI summarizer and the watch smart-stack's relevance scores, answer "does this matter right now?" rather than "should this value be shown?" Different question, deliberately different and higher floors.
- **Accessibility chart summaries** gate on `> 0`, deliberately broader than the display floor. A VoiceOver description of a whole chart ("up to N percent") should describe the data the chart plots, and the detail charts plot sub-threshold values.
- **A frozen catalog of legacy forecast styles** carries dozens of per-style thresholds. It is frozen, behind a debug-only setting and byte-stable by policy, and rewriting frozen code to satisfy a new convention is how you un-freeze it by accident.

Excluding them is the easy part. What matters is that the exclusions are *inventoried*, in the plan doc the checklist bullet points at, each with the reason it is a different question. An undocumented exemption is indistinguishable from a site the sweep missed. A documented one is a decision.

## Proving the Chokepoint Is Actually Complete

"We routed everything through one function" is a claim, and a claim from the session that did the routing inherits every blind spot of the authoring context. So before merge, the change went through the [adversarial review process](/adversarial-review-rounds/): four independent, read-only reviewers, none with the session's reasoning.

One of them was an exhaustive-sweep reviewer whose brief was not "review this diff" but "prove or refute the completeness claim": find every fractional probability comparison in the codebase and account for it. Its report: outside the frozen style catalog, exactly four such comparisons exist, the chokepoint plus the three documented relevance-scoring exemptions. Not "the diff looks complete." An enumeration, checked against the inventory, with zero unaccounted-for sites.

That is a different kind of assurance, and it is cheap to ask for. A unification PR's central claim is universally quantified: *no* other site answers this question. A diff review cannot verify a universal claim, because the sites that falsify it are by definition not in the diff. Give one reviewer the whole codebase and the quantifier as its brief.

The other lenses found real things too. A behavioral pass traced boundary values through every surface and confirmed the suppression-clause deletion with predicate math. Its two confirmed findings were fixed before merge: an anchor-selection gap in the pill's headline scan, and an off-by-one where the "starting soon" copy announced the precip *type* of the hour after the match ("rain possible in 3h" for what was actually snow). Known geometry caveats, wide-layout column counts on tablets and landscape phones being hand-maintained estimates rather than measured, were recorded in the plans as deferred, with the measurement rework named as the real fix.

## Lessons Learned

- **Take raw inputs, normalize inside, and pin idempotence.** Letting call sites choose the representation they compare is how raw-vs-rounded bugs happen. A sweep test that raw and pre-rounded inputs agree makes "pass whatever you have" safe.
- **Forbid the anonymous threshold, not the second one.** "Any new floor must be a named member of this type" turns silent drift into a reviewable decision without blocking evolution.
- **One function signature is a units contract.** Scattered comparisons can each be wrong in their own units. Expect a sweep to surface at least one that never worked.
- **When you delete provably redundant code, write down what resurrects it.** The condition, the exact form, and the past bug it guards against, in the checklist reviewers actually read.
- **Record the design that lost, on equal footing.** A variant rejected without field evidence belongs in the plan as a pre-built alternative, so the next round starts from the debate instead of re-deriving it.

---

## How This Post Was Made

**Prompt 1:** "see recent work in ~/Code/helloweather, perhaps a blog post about our opus 4.8 agents and why we decided to do that? perhaps something about the swift testing + snapshots inspired by minitest-snapshots? anything else? bring me a list of potential post ideas for review."

**Prompt 2:** "skip 4, 5, 6, 9 but create posts for each of the others in the 1-9 list. also add Four Answers to One Question, and Write the Rule, Not the Story -- show me a concise version of your plan and then I can approve" — then "proceed, one pr per post"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post keeps its screenshot opening, loses the bolded asides and flourishes in the body, splits the chained sentences, and trims Lessons Learned from nine bullets to five that do not repeat a section heading. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The four targets are app, widgets, watch app, and watch extension (push copy lives in the app, not a separate target); the "forty-four" figure is the PR's own count and is now attributed as such; the lock-screen "before" excerpt now matches the real gate, which compared an already-rounded fraction against 20; the two checklist bullets were updated to their current wording, which gained window-significance clauses on 2026-08-31; and a paragraph records that the field evidence the plan was waiting on arrived and the fix landed as a named `PrecipDisplay` member, as the rule required.
