---
layout: post
title: "An Agent Fan-Out Pipeline with a Hard Isolation Contract"
date: 2026-07-29 09:00:00 -0600
summary: "Running dozens of coding agents in parallel doesn't need a smarter reviewer. It needs a brief schema, central wiring done first, a mechanical verifier, and an isolation contract that makes throwaway-quality output safe to merge."
tags: [ai-agents, parallelism, swiftui, android, workflow]
---

## The Problem

Fanning out parallel coding agents is easy. Getting a mergeable result out the other end is not.

Run twenty agents at once on the same repo and you get the same four failures every time:

1. **Merge conflicts.** Every agent needs its work registered in some shared file — an enum, a switch, a router table — and they all edit that file simultaneously.
2. **Incoherent output.** Twenty agents given twenty vague prompts produce twenty different interpretations of "good."
3. **Silent partial work.** Agents die mid-write. API errors, spend limits, timeouts. You get a half-written folder that looks superficially complete.
4. **Review collapse.** Even if each unit is fine, nobody can meaningfully review 400 files of fast-written code. The reviewer becomes the bottleneck, and then the reviewer becomes a rubber stamp.

The instinct is to fix this with better judgment — a smarter reviewing agent, a stricter prompt, more careful reading. That doesn't scale, because judgment is exactly the resource you ran out of.

What does scale is structure. Below are two runs from [Hello Weather](https://helloweather.com) that used the same five-phase shape: an iOS visual-style catalog that went from 41 to roughly 168 entries in about a month, and an Android translation pass that produced 191 strings across 27 locales.

## The Pipeline

### Phase 1: Parallel research, fixed brief schema

One research agent per aesthetic cluster — graphic-design movements, retro computing and games, transit and wayfinding signage, print and paper craft, instruments and broadcast hardware. Each agent uses web search rather than memory alone, and each returns briefs in an identical shape:

> - **Name** + one-line identity (the elevator pitch)
> - **Grounding**: the specific real works/rules/hardware, era-accurate details with sources
> - **Palette**: 4-8 hex values; flag interpretive hexes honestly when no official spec exists
> - **Typography**: system-font approximations
> - **Section mapping**: hero / hourly / daily / stats, concrete and clever
> - **Signature element to nail**: the one thing that makes it instantly recognizable
> - **Feasibility** and **wow factor**
> - **Distinctness check** vs the existing catalog

The schema is doing two jobs. It makes briefs comparable to each other, so a human can rank thirty of them in one sitting. And it makes briefs *executable* — a downstream implementation agent gets the palette, the typography, and the section mapping without having to invent any of it.

Then a quality bar, stated as one sentence:

> **The quality bar: the DATA wears the aesthetic.** The best ideas make the forecast become the artwork, not decorate it. Reject briefs where the style is only chrome around a generic chart.

That single line kills more bad output than any amount of code review. It's a filter applied at the idea stage, where rejecting something costs nothing.

There is also an explicit constraint in the brief format worth stealing verbatim for any project that draws on cultural reference:

> **IP cautions**: evoke grammar, never copy sprites/logos/mascots/likenesses.

A visual grammar — a palette, a grid, a typographic register, a way of drawing a dial — is fair game. The specific artwork is not. Making that a required field in the brief means every idea gets checked against it before anyone writes a line of code, rather than after.

### Phase 2: A human checkpoint before any code

Research agents produce one ranked list. Then everything stops.

> Synthesize into one ranked list and stop for discussion. No task lists, no worktrees, no implementation until the set is picked.

This is the cheapest phase in the pipeline and the highest leverage. Ideas that get cut here cost minutes. Ideas that get cut after implementation cost an agent-hour each and a pile of merged code that has to come back out. Light sketches that survive the cut get sent back for a second research round to become full briefs before implementation.

### Phase 3: Wire centrally, then fan out

This is the phase that eliminates merge conflicts, and it is the one most easily skipped.

Before any implementation agent launches, the orchestrator adds *every* new entry to *every* shared file: the enum and its display-name switch, the view dispatch, the styling extensions, and — the trap that broke the first run — the two exhaustive switches in the widget and watch targets, which need an explicit arm per case or the build fails.

For dozens of entries at once, hand-editing six files is a mistake. A Python codemod with per-insertion assertions does it instead:

> For large packs, a Python codemod with per-insertion assertions (fail loudly on a missing anchor, print per-file counts) beats dozens of hand edits.

The assertions matter more than the automation. A codemod that silently no-ops on a missing anchor is worse than a hand edit, because it produces a plausible-looking diff with a hole in it. Fail on the missing anchor, print a per-file insertion count, and compare that count to what you expected.

Once central wiring lands, every shared file is done. Implementation agents are told, as a hard rule, that they may only create files inside their own folder. Twenty agents, zero overlapping writes, zero conflicts — not because the agents coordinated, but because there was nothing left to coordinate over.

### Phase 4: One agent per unit, brief pasted inline

The fan-out prompt is a template, and its most important property is that it is *repetitive*:

- Absolute paths in every read and write (worktrees make relative paths a coin flip)
- Read a named reference implementation first, plus the two plan-doc sections describing available data and cross-cutting rules
- Create exactly one folder, with a prescribed file layout and a required type-name prefix on every declaration
- The hard rules **restated verbatim in every prompt**: no comments, no force unwraps or raw indexing, guard zero denominators, no bundled assets, deterministic index-hash randomness only — nothing time- or random-seeded at render time
- The full brief pasted inline
- "Do not build; report summary + files + line counts"

Restating the rules in full for every single agent feels redundant when you're writing the dispatcher. It isn't. There is no shared memory between parallel agents; a rule stated once in a document that an agent may or may not read is a rule that holds maybe 80% of the time. Eighty percent across 67 units is thirteen violations.

Note the last line: agents do not build. One build at the end, by the orchestrator, is far cheaper than twenty agents each spinning up a compiler and each trying to fix errors in shared state.

Concurrency has a practical ceiling — around 20 agents in flight, with the rest queued and dispatched as completions free up slots.

### Phase 5: Verify-then-repair, as a real phase

The most valuable single lesson from the whole exercise:

> **Verify-then-repair**: agents can die mid-write (API errors, spend limits). Survey folders: entry view present + `swiftc -parse` clean → keep; clearly partial → delete the folder and relaunch fresh. **Never assume a completion.**

Treat the fan-out as *unreliable by construction*. After every batch, enumerate what should exist, enumerate what does exist, diff the two, and relaunch the gaps from scratch. Not repair — relaunch. A half-written unit is cheaper to delete and redo than to diagnose.

Then mechanical audit greps across all new folders, looking for the rules that were supposed to hold: comment lines, force-unwraps and force-casts, non-verbatim string constructors, raw index access. Every one of those is a regex, not a judgment call.

Only then: one build, fix only clearly-diagnosed errors, one commit, human QA pause, then push.

## The Isolation Contract

Everything above makes the fan-out *produce* a lot of code. The isolation contract is what makes it *safe to merge* a lot of fast-written code.

The catalog is debug-only. It is never shipped to users; there is no user-facing picker, and the whole thing sits behind an internal debug gate. The default path renders production views and is untouched. That's the starting point, but on its own it isn't enough, because shared helper code is still a coupling channel — and coupling is what turns 81,000 lines of throwaway-quality code into a permanent tax on the production codebase.

So the contract says: **shared style components are copies, never references.**

> Styles work makes minimal-to-zero production changes, so production features can launch and evolve without worrying about styles. Everything under the shared styles directory is **copies** — fork in-flux production views as prefixed types (mechanical copy first, tokenize second); never reference or modify them from shared style code.

Two bounded exceptions, both deliberate: a style's own scroll view may *call* stable production leaf views read-only, and the alerts view is always reused by reference, because the safety-critical surface must have exactly one implementation.

Copies are usually the wrong instinct. Here they're the entire point. Production charts were mid-rewrite during all of this. If the catalog referenced them, every production change would have had to be validated against 168 downstream consumers of unknown quality. Because the catalog holds copies, the production rewrite shipped without ever considering them.

The rest of the policy follows from that:

- **Zero-change invariant**: no catalog work may alter default rendering.
- **Frozen as-built**: when features land on the default path, catalog entries are *not* updated. Divergence is expected and accepted, and paid down only if an entry is ever promoted.
- **Accessibility debt is deliberate**, not oversight — it's on the promotion checklist, not the build checklist.
- **Don't refactor across entries.** Persisted raw values stay stable; display names can change.

And the partition is verified by a test rather than by discipline. Every entry must belong to exactly one bucket — production, near-shipping, kept, pending-deletion:

```swift
@Test("Every style belongs to a group")
func groupsCoverAllStyles() {
    let all = Set(SettingsManager.ForecastStyle.allCases)
    let union = SettingsManager.ForecastStyle.styleGroupSets
        .reduce(into: Set<SettingsManager.ForecastStyle>()) { $0.formUnion($1.1) }

    #expect(all.subtracting(union).isEmpty)
}

@Test("Production is exactly the default style")
func productionIsTheDefault() {
    #expect(SettingsManager.ForecastStyle.productionStyles == [.helloWeather])
}
```

Coverage, disjointness, exact count, and "production contains exactly one thing." Adding an entry without classifying it fails the test. The isolation boundary is an assertion, not a convention.

### Two performance findings worth keeping

Fan-out produces work nobody has profiled, and two problems showed up repeatedly:

- **Hundreds of concurrently animated views is a meltdown.** The fix is to sequence groups so only one animates at a time, and to add `drawingGroup()` per row to collapse compositing layers.
- **`rotation3DEffect` at exactly ±90° with perspective** produces a degenerate transform and logs "ignoring singular matrix" every frame. Clamp to ±89.9°.

Also: hold any entrance choreography until loading clears, so animation never fights launch and refresh work.

## Case Study: 27 Locales

The same shape, a completely different domain, and a much sharper verifier.

The task: translate 191 Android strings into 27 locales. One agent per locale, 27 in parallel.

**The brief-schema equivalent** was an anchor rather than a template. Each agent was pointed at the sibling iOS app's professionally-reviewed translation file and told to match its terminology for the same locale. That single anchor removes the largest source of variance in parallel translation — twenty-seven agents independently deciding how to render "feels like" or "chance of rain." The terminology decisions were already made, by humans, and paid for.

**Central wiring came first**, exactly as in the iOS case: a separate, behaviorally inert commit declaring all 28 locales in the locale config, wiring the per-app language picker, and adding the AppCompat service that persists the user's choice across OS versions. No translation files in that commit at all. With English-only resources present, it changes nothing — which is the point. The shared configuration file was finished and merged before a single translation agent ran.

**The verifier is completely mechanical.** Every locale must pass:

- Key-set equality with the source file (no added strings, no dropped strings, no renamed `name` attributes)
- Format-specifier preservation — `%1$s`, `%d`, `%1$.0f`, `%%` — present, correct, and in the right order
- Escaping rules: backslash-escaped apostrophes and `@`, entities or CDATA for ampersands and ellipses
- XML validity
- And then the whole thing has to survive a full resource merge and compile with all 27 locales present

Not one of those checks requires reading the translation. That's the design goal. A parallel pipeline needs a gate that says *yes* or *no* without a human in the loop, and "does this Czech sentence read well" is not that gate. Key-set equality is.

The residual risk is real and named rather than pretended away: mechanical checks confirm structural correctness, not fluency. Professional post-editing is planned as a separate pass. The verifier's job is to guarantee nothing is *broken*, not to guarantee everything is *good*.

**The platform wrinkle** is the interesting part, and it's the isolation contract showing up in an unfamiliar costume. Android auto-enables any `values-XX/strings.xml` present in the build:

> Android automatically enables any `values-XX/strings.xml` present in the build — unlike iOS, there is no debug-only option. Only create translation files when ready to ship them to all users.

There is no debug gate available. Merging the branch *is* the launch. So the branch itself became the isolation boundary: it stays a deliberate draft, complete and verified and unmerged, until someone decides to ship localization. The commit message says so explicitly.

That's the general lesson. Every fan-out needs a boundary that lets fast-produced output exist without being live. On iOS it was a debug flag plus a copy-not-reference rule. On Android the platform refused to provide one, so the boundary moved up a level to version control. What matters is that the boundary exists and is written down — not which layer it lives at.

## Lessons Learned

- **Parallel agents don't need a smarter reviewer, they need a contract.** Every fix here is structural. None of them is "review more carefully."
- **A brief schema is a coordination protocol.** Fixed fields make outputs comparable for humans and executable for downstream agents. Freeform research briefs give you neither.
- **Wire centrally, then fan out.** Finish every shared file *before* launching parallel workers, and the entire class of merge conflicts disappears. This is the single highest-value phase.
- **Codemods need per-insertion assertions.** A codemod that silently skips a missing anchor is worse than a hand edit, because the resulting diff looks fine.
- **Restate hard rules verbatim in every prompt.** Parallel agents share no memory. A rule stated once holds most of the time, and "most" times sixty-seven is a lot of violations.
- **Prefer a mechanical verifier to a judgment-based one.** Key-set equality, format-specifier preservation, XML validity, a clean parse, audit greps. If the gate needs taste, it won't run at scale.
- **Verify-then-repair is a phase, not an afterthought.** Agents die mid-write. Enumerate expected versus actual, and relaunch the gaps from scratch rather than debugging partial output.
- **Copies beat references when quality is uneven.** Coupling fast-written code to production code makes every future production change a 168-way compatibility problem.
- **Make the boundary a test.** A partition that's enforced by a coverage-and-disjointness test can't quietly rot; a partition enforced by convention will.
- **Name the residual risk instead of hiding it.** Deliberate accessibility debt and pending professional translation review are both written down. Undocumented debt is the kind that surprises you.
- **Evoke the grammar, never copy the artwork.** Make it a required field in the brief, so it's checked before implementation rather than after.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.
