---
layout: post
title: "An Agent Fan-Out Pipeline with a Hard Isolation Contract"
date: 2026-07-29 09:00:00 -0600
summary: "Running dozens of coding agents in parallel doesn't need a smarter reviewer. It needs a brief schema, central wiring done first, a mechanical verifier, and an isolation contract that makes throwaway-quality output safe to merge."
tags: [ai-agents, parallelism, swiftui, ios, android, workflow]
---

## The Problem

An iOS visual-style catalog went from 41 to roughly 168 entries in eleven days. An Android translation pass produced 191 strings across 27 locales. Both are [Hello Weather](https://helloweather.com) work, both were written by coding agents running about twenty at a time, and both used the same five-phase pipeline, because running twenty agents at once on one repo produces the same four failures every time:

1. **Merge conflicts.** Every agent needs its work registered in some shared file, an enum, a switch, a router table, and they all edit that file at the same moment.
2. **Incoherent output.** Twenty agents given twenty vague prompts produce twenty different interpretations of "good."
3. **Silent partial work.** Agents die mid-write on API errors, spend limits, and timeouts. You get a half-written folder that looks complete.
4. **Review collapse.** Nobody can meaningfully review 400 files of fast-written code, so the reviewer becomes the bottleneck and then a rubber stamp.

The instinct is to fix this with better judgment: a smarter reviewing agent, a stricter prompt, more careful reading. That doesn't scale, because judgment is the resource that ran out. Structure does scale. The pipeline below is the structure, and the isolation contract after it is what made the output safe to merge.

## The Pipeline

### Phase 1: Parallel research, fixed brief schema

One research agent per aesthetic cluster: graphic-design movements, retro computing and games, transit and wayfinding signage, print and paper craft, instruments and broadcast hardware. Each agent uses web search rather than memory alone, and each returns briefs in one shape:

> - **Name** + one-line identity (the elevator pitch)
> - **Grounding**: the specific real works/rules/hardware, era-accurate details with sources
> - **Palette**: 4-8 hex values; flag interpretive hexes honestly when no official spec exists
> - **Typography**: system-font approximations
> - **Section mapping**: hero / hourly / daily / stats, concrete and clever
> - **Signature element to nail**: the one thing that makes it instantly recognizable
> - **Feasibility** and **wow factor**
> - **Distinctness check** vs the existing catalog

The schema does two jobs. Briefs become comparable, so a human can rank thirty of them in one sitting. And briefs become executable: a downstream implementation agent gets the palette, the typography, and the section mapping without inventing any of it.

The quality bar is one sentence:

> **The quality bar: the DATA wears the aesthetic.** The best ideas make the forecast become the artwork, not decorate it. Reject briefs where the style is only chrome around a generic chart.

A filter applied at the idea stage rejects for free. That line removed more bad output than any amount of code review.

One more required field applies to any project that draws on cultural reference:

> **IP cautions**: evoke grammar, never copy sprites/logos/mascots/likenesses.

A visual grammar (a palette, a grid, a typographic register, a way of drawing a dial) is fair game. The specific artwork is not. As a required field, it is checked before anyone writes a line of code, not after.

### Phase 2: A human checkpoint before any code

Research agents produce one ranked list. Then everything stops.

> Synthesize into one ranked list and stop for discussion. No task lists, no worktrees, no implementation until the set is picked.

This is the cheapest phase in the pipeline and the highest leverage. An idea cut here costs minutes. An idea cut after implementation costs an agent-hour and a pile of merged code that has to come back out. Light sketches that survive the cut go back for a second research round to become full briefs.

### Phase 3: Wire centrally, then fan out

This phase eliminates merge conflicts, and it is the one most easily skipped.

Before any implementation agent launches, the orchestrator adds every new entry to every shared file: the enum and its display-name switch, the view dispatch, the styling extensions, and the two exhaustive switches in the widget and watch targets. Those two switches broke the first run. They need an explicit arm per case or the build fails.

Hand-editing six files for dozens of entries is a mistake. A Python codemod does it instead:

> For large packs, a Python codemod with per-insertion assertions (fail loudly on a missing anchor, print per-file counts) beats dozens of hand edits.

The assertions matter more than the automation. A codemod that silently no-ops on a missing anchor is worse than a hand edit, because it produces a plausible diff with a hole in it. Compare the per-file counts to what you expected.

Once central wiring lands, every shared file is finished, and implementation agents may only create files inside their own folder, as a hard rule. Twenty agents, zero overlapping writes, zero conflicts. Not because the agents coordinated, but because nothing was left to coordinate over.

### Phase 4: One agent per unit, brief pasted inline

The fan-out prompt is a template, and its most important property is repetition:

- Absolute paths in every read and write (worktrees make relative paths a coin flip)
- Read a named reference implementation first, plus the two plan-doc sections describing available data and cross-cutting rules
- Create exactly one folder, with a prescribed file layout and a required type-name prefix on every declaration
- The hard rules **restated verbatim in every prompt**: no comments, no force unwraps or raw indexing, guard zero denominators, no bundled assets, deterministic index-hash randomness only, nothing time- or random-seeded at render time
- The full brief pasted inline
- "Do not build; report summary + files + line counts"

Restating the rules for every agent feels redundant while writing the dispatcher. It isn't. Parallel agents share no memory, so a rule stated once in a document an agent may or may not read holds maybe 80% of the time. Eighty percent across 67 units is thirteen violations.

Agents do not build. One build at the end, by the orchestrator, is far cheaper than twenty agents each spinning up a compiler and fixing errors in shared state. Concurrency has a practical ceiling of around 20 agents in flight. The rest queue and dispatch as completions free up slots.

### Phase 5: Verify-then-repair, as a real phase

> **Verify-then-repair**: agents can die mid-write (API errors, spend limits). Survey folders: entry view present + `swiftc -parse` clean → keep; clearly partial → delete the folder and relaunch fresh. **Never assume a completion.**

Treat the fan-out as unreliable by construction. After every batch, enumerate what should exist, enumerate what does exist, diff the two, and relaunch the gaps from scratch. Relaunch, not repair. A half-written unit is cheaper to delete and redo than to diagnose.

Then audit greps run across all new folders for the rules that were supposed to hold: comment lines, force-unwraps and force-casts, non-verbatim string constructors, raw index access. Each is a regex, not a judgment call.

Only then: one build, fix only clearly diagnosed errors, one commit, a human QA pause, then push.

## The Isolation Contract

The pipeline produces a lot of code. The third pack alone landed 81,000 lines in one pull request, and by late July the catalog was about 173,000 lines. The isolation contract is what made that safe to merge.

The catalog is debug-only. There is no user-facing picker, the whole thing sits behind an internal debug gate, and the default path renders production views untouched. On its own that isn't enough. Shared helper code is still a coupling channel, and coupling is what turns throwaway-quality code into a permanent tax on the production codebase.

So the contract, as written on 2026-07-26, said: **shared style components are copies, never references.**

> Styles work makes minimal-to-zero production changes, so production features can launch and evolve without worrying about styles. Everything under the shared styles directory is **copies** — fork in-flux production views as prefixed types (mechanical copy first, tokenize second); never reference or modify them from shared style code.

Two bounded exceptions, both deliberate. A style's own scroll view may call stable production leaf views read-only. The alerts view is always reused by reference, because the safety-critical surface must have exactly one implementation.

Copies are usually the wrong instinct. Here they were the mechanism. Production stat charts were mid-rewrite behind a debug flag during all of this. If the catalog referenced them, every production change would have needed validation against 168 downstream consumers of unknown quality. Because the catalog held copies, the rewrite proceeded without ever considering them.

That rule was a fan-out-era rule, and it has since been retired. On 2026-08-21, the same week a triage pass deleted 97 entries from the catalog, the skill replaced it:

> **Styles share the real views.** Production is just the v4 style, and v5 is a candidate to replace it, so there is no wall to copy across. A style calls production views by reference (sheets, leaf views, the alerts view) and parameterizes them when it needs a variation; it copies a view only when it genuinely needs different behavior, never for isolation.

The shared styles directory is gone. The copies rule held while 168 entries of unknown quality landed in eleven days; once the catalog had been culled and the default view became one candidate among the survivors, the wall had nothing left to protect.

The rest of the policy follows from that:

- **Zero-change invariant**: no catalog work may alter default rendering.
- **Frozen as-built**: when features land on the default path, catalog entries are not updated. Divergence is expected and accepted, and paid down only if an entry is ever promoted.
- **Accessibility debt is deliberate**, not oversight. It sits on the promotion checklist, not the build checklist.
- **Don't refactor across entries.** Persisted raw values stay stable; display names can change.

The partition is verified by a test rather than by discipline. Every entry must belong to exactly one bucket: production, near-shipping, kept, pending-deletion.

```swift
@Test("Every style belongs to a group")
func groupsCoverAllStyles() {
    let all = Set(SettingsManager.ForecastStyle.allCases)
    let union = SettingsManager.ForecastStyle.styleGroupSets
        .reduce(into: Set<SettingsManager.ForecastStyle>()) { $0.formUnion($1.1) }
    let missing = all.subtracting(union)

    #expect(missing.isEmpty, "Ungrouped styles: \(missing.map(\.rawValue).sorted())")
}

@Test("Production is exactly the default style")
func productionIsTheDefault() {
    #expect(SettingsManager.ForecastStyle.productionStyles == [.helloWeather])
}
```

The suite checks coverage, disjointness, exact count, and that production contains exactly one thing. Adding an entry without classifying it fails the test. The boundary is an assertion, not a convention.

### Two performance findings worth keeping

Fan-out produces work nobody has profiled, and two problems recurred:

- **Hundreds of concurrently animated views is a meltdown.** Sequence groups so only one animates at a time, and add `drawingGroup()` per row to collapse compositing layers.
- **`rotation3DEffect` at exactly ±90° with perspective** produces a degenerate transform and logs "ignoring singular matrix" every frame. Clamp to ±89.9°.

Hold any entrance choreography until loading clears, so animation never fights launch and refresh work.

## Case Study: 27 Locales

Same shape, a different domain, and a much sharper verifier. The task: translate 191 Android strings into 27 locales, one agent per locale, 27 in parallel.

The brief schema became an anchor rather than a template. Each agent was pointed at the sibling iOS app's professionally reviewed translation file for the same locale and told to match its terminology. That removes the largest source of variance in parallel translation, twenty-seven agents independently deciding how to render "feels like" or "chance of rain." Those decisions were already made, by humans, and paid for.

Central wiring came first, exactly as on iOS: a separate, behaviorally inert commit declaring 28 locales in the locale config (English plus the 27 targets), wiring the per-app language picker, and adding the AppCompat service that persists the user's choice across OS versions. No translation files in that commit at all. With English-only resources present it changes nothing, and the shared configuration file was merged before a single translation agent ran.

The verifier is completely mechanical. Every locale must pass:

- Key-set equality with the source file (no added strings, no dropped strings, no renamed `name` attributes)
- Format-specifier preservation: `%1$s`, `%d`, `%1$.0f`, `%%`, present, correct, and in the right order
- Escaping rules: backslash-escaped apostrophes and `@`, entities or CDATA for ampersands and ellipses
- XML validity
- A full resource merge and compile with all 27 locales present

None of those checks requires reading the translation. A parallel pipeline needs a gate that says yes or no without a human in the loop, and "does this Czech sentence read well" is not that gate. Key-set equality is. The residual risk is named rather than pretended away: mechanical checks confirm structural correctness, not fluency. Professional post-editing was planned as a separate pass, then waived on 2026-07-31 in favor of the model-only quality pass that had cleared the sibling iOS launch.

The platform wrinkle is the isolation contract in an unfamiliar costume. Android auto-enables any `values-XX/strings.xml` present in the build:

> Android automatically enables any `values-XX/strings.xml` present in the build — unlike iOS, there is no debug-only option. Only create translation files when ready to ship them to all users.

There is no debug gate, so merging the branch is the launch. The branch itself became the isolation boundary. It stays a deliberate draft, complete and verified and unmerged, until someone decides to ship localization, and the commit message says so explicitly.

On iOS, what shipped is a catalog of roughly 168 entries, up from 41 in eleven days: about 173,000 lines of debug-only code that never alters default rendering and that a test keeps partitioned from production. The accepted cost is divergence. Entries are frozen as built, carry deliberate accessibility debt, and are paid down only if one is promoted. On Android, what exists is 191 strings across 27 locales, verified and unmerged, held for the release that ships localization; the professional post-editing pass was waived on 2026-07-31.

## Lessons Learned

- **Fix a fan-out with structure, not judgment.** A brief schema, central wiring, and a mechanical gate each remove a failure class. "Review more carefully" removes none.
- **A gate that needs taste won't run at scale.** Key-set equality, format-specifier order, a clean parse, and audit greps say yes or no without a human.
- **Copies beat references when quality is uneven.** Coupling fast-written code to production makes every later production change an N-way compatibility problem. Once the catalog was culled to survivors, the copies rule was retired; it was scaffolding for the fan-out, not a permanent architecture.
- **The isolation boundary can live at any layer, but it must exist and be written down.** A debug flag, a copy-not-reference rule, and an unmerged branch all work.
- **Name the residual risk instead of hiding it.** Deliberate accessibility debt and the translation review decision, including its later waiver, are both written down. Undocumented debt is the kind that surprises you.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the two runs and their figures, each pipeline phase and the contract say their part once, the case study closes with what shipped and what it cost in place of a Results section, and Lessons Learned went from eleven bullets to five. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The 41-to-168 jump took eleven days (2026-07-15 to 07-26), not a month; 81,000 lines was the third pack alone, so the catalog total (about 173,000 lines) is now stated separately; the copies-not-references contract is dated to 2026-07-26 and the post now records that the skill retired it on 2026-08-21 in favor of styles sharing the real views, and the stat-chart rewrite is described as proceeding behind a flag rather than shipped. The Swift test excerpt now matches the real file, the locale config is described as 28 entries (English plus the 27 translated locales), and the Android professional post-editing pass is recorded as waived on 2026-07-31 rather than still pending.
