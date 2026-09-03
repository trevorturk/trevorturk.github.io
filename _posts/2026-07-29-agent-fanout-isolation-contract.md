---
layout: post
title: "An Agent Fan-Out Pipeline with a Hard Isolation Contract"
date: 2026-07-29 09:00:00 -0600
summary: "Running dozens of coding agents at once doesn't need a smarter reviewer. It needs one fixed shape for briefs, the shared files wired first, a mechanical check, and an isolation rule that makes rough code safe to merge."
tags: [ai-agents, parallelism, swiftui, ios, android, workflow]
---

## The Problem

An iOS catalog of visual styles went from 41 entries to about 168 in eleven days. An Android translation pass produced 191 strings in 27 locales. Both are [Hello Weather](https://helloweather.com) work, and both were written by coding agents running about twenty at a time. Both used the same five-phase pipeline, because twenty agents on one repo fail the same four ways every time:

1. **Merge conflicts.** Every agent has to register its work in a shared file, and they all edit that file at the same moment.
2. **Incoherent output.** Give twenty agents a vague prompt and you get twenty different ideas of "good."
3. **Silent partial work.** Agents die mid-write on API errors and spend limits. You get a half-written folder that looks finished.
4. **Review collapse.** Nobody can really review 400 files of fast-written code. The reviewer becomes the bottleneck and then a rubber stamp.

The instinct is to fix this with a smarter reviewer or a stricter prompt. That doesn't work, because careful attention is the thing that ran out. Structure does work at this scale. The pipeline below is the structure, and the isolation rule after it made the output safe to merge.

## The Pipeline

### Phase 1: Parallel research, fixed brief schema

We ran one research agent per family of aesthetics: graphic-design movements, retro computing and games, transit signage, print and paper craft, and broadcast hardware. Each agent uses web search rather than memory alone, and each returns briefs in one shape:

> - **Name** + one-line identity (the elevator pitch)
> - **Grounding**: the specific real works/rules/hardware, era-accurate details with sources
> - **Palette**: 4-8 hex values; flag interpretive hexes honestly when no official spec exists
> - **Typography**: system-font approximations
> - **Section mapping**: hero / hourly / daily / stats, concrete and clever
> - **Signature element to nail**: the one thing that makes it instantly recognizable
> - **Feasibility** and **wow factor**
> - **Distinctness check** vs the existing catalog

The fixed shape does two jobs. Briefs become comparable, so a person can rank thirty of them in one sitting. And a brief is enough to build from: the implementation agent gets the palette, the typography, and the section mapping, and doesn't have to invent any of it.

The quality bar is one sentence:

> **The quality bar: the DATA wears the aesthetic.** The best ideas make the forecast become the artwork, not decorate it. Reject briefs where the style is only chrome around a generic chart.

Rejecting an idea at the brief stage is cheap. That one line removed more bad output than any amount of code review.

One more required field applies to any project that draws on cultural references:

> **IP cautions**: evoke grammar, never copy sprites/logos/mascots/likenesses.

A style's grammar is fair game, meaning its palette, its grid, its typefaces, the way it draws a dial. The specific artwork is not. Because the field is required, this gets checked before anyone writes code, not after.

### Phase 2: A human checkpoint before any code

The research agents merge their work into one ranked list. Then everything stops.

> Synthesize into one ranked list and stop for discussion. No task lists, no worktrees, no implementation until the set is picked.

This is the cheapest phase in the pipeline and the one that matters most. An idea cut here costs minutes. An idea cut after implementation costs an agent-hour and a pile of merged code that has to come back out. Rough sketches that survive the cut go back for a second research round and come back as full briefs.

### Phase 3: Wire centrally, then fan out

This phase removes the merge conflicts, and it's the easiest one to skip.

Before any implementation agent launches, the orchestrator (the agent that dispatches the others) adds every new entry to every shared file. That means the enum and its display-name switch, the view dispatch, the styling extensions, and the two exhaustive switches in the widget and watch targets. Those two switches broke the first run. Each needs an explicit case per entry or the build fails.

Editing six files by hand for dozens of entries is a mistake, so a Python script does the editing instead:

> For large packs, a Python codemod with per-insertion assertions (fail loudly on a missing anchor, print per-file counts) beats dozens of hand edits.

The assertions matter more than the automation. A script that can't find the line it was supposed to insert after, and silently does nothing, is worse than a hand edit. It produces a plausible diff with a hole in it. Compare the per-file counts to what you expected.

Once the central wiring lands, every shared file is finished. From then on an implementation agent may only create files inside its own folder, and that's a hard rule. Twenty agents wrote to twenty folders and never touched the same file, so there was nothing to conflict over.

### Phase 4: One agent per entry, brief pasted inline

Every implementation agent gets the same prompt template. The important thing about the template is what it repeats:

- Absolute paths in every read and write, because a relative path inside a worktree can resolve to the wrong checkout
- Read a named reference implementation first, plus the two sections of the plan doc that describe the available data and the rules that apply to every entry
- Create exactly one folder, with a set file layout and a required type-name prefix on every declaration
- The hard rules, **restated verbatim in every prompt**: no comments, no force unwraps or raw indexing, guard zero denominators, no bundled assets, deterministic index-hash randomness only, nothing time- or random-seeded at render time
- The full brief pasted inline
- "Do not build; report summary + files + line counts"

Restating the rules for every agent feels redundant while you're writing the orchestrator's prompt. It isn't. Parallel agents share no memory, so a rule stated once in a document an agent may or may not read holds maybe 80% of the time. Eighty percent across 67 entries is thirteen violations.

Agents don't build. The orchestrator builds once at the end, which is far cheaper than twenty agents each starting a compiler and fixing errors in shared state. About 20 agents can be in flight at once. The rest queue and launch as slots free up.

### Phase 5: Verify-then-repair, as a real phase

> **Verify-then-repair**: agents can die mid-write (API errors, spend limits). Survey folders: entry view present + `swiftc -parse` clean → keep; clearly partial → delete the folder and relaunch fresh. **Never assume a completion.**

Treat the fan-out as unreliable by design. After every batch, list what should exist, list what does exist, diff the two, and relaunch the gaps from scratch. Relaunch, not repair. A half-written entry is cheaper to delete and redo than to diagnose.

Then we grep every new folder for the rules that were supposed to hold: comment lines, force unwraps and force casts, non-verbatim string constructors, raw index access. Each check is a regex, not a judgment call.

Only then: one build, fix only the clearly diagnosed errors, one commit, a pause for a person to QA, then push.

## The Isolation Contract

The pipeline produces a lot of code. The third pack alone landed 81,000 lines in one pull request, and by late July the catalog was about 173,000 lines. The isolation rule made that safe to merge.

The catalog is debug-only. There's no user-facing picker, the whole thing sits behind an internal debug flag, and the default path renders production views untouched. On its own that isn't enough. If catalog code calls production helpers, every production change has to account for the catalog, and rough code becomes a permanent tax on production.

So the rule, as written on 2026-07-26, said: **shared style components are copies, never references.**

> Styles work makes minimal-to-zero production changes, so production features can launch and evolve without worrying about styles. Everything under the shared styles directory is **copies** — fork in-flux production views as prefixed types (mechanical copy first, tokenize second); never reference or modify them from shared style code.

Two exceptions, both deliberate. A style's own scroll view may call stable production leaf views, read-only. The alerts view is always reused by reference, because there must be exactly one implementation of the safety-critical view.

Copies are usually the wrong instinct. Here they were the point. The production stat charts were mid-rewrite behind a debug flag during all of this. If the catalog referenced them, every production change would have needed checking against 168 consumers of unknown quality. Because the catalog held copies, the rewrite went ahead without considering them at all.

That was a fan-out-era rule, and it's since been retired. On 2026-08-21, the same week a triage pass deleted 97 entries from the catalog, the skill replaced it:

> **Styles share the real views.** Production is just the v4 style, and v5 is a candidate to replace it, so there is no wall to copy across. A style calls production views by reference (sheets, leaf views, the alerts view) and parameterizes them when it needs a variation; it copies a view only when it genuinely needs different behavior, never for isolation.

The shared styles directory is gone. The copies rule held while 168 entries of unknown quality landed in eleven days. Once the catalog had been culled and the default view became one candidate among the survivors, there was nothing left to isolate.

The rest of the policy follows from that:

- **Zero-change invariant**: no catalog work may change default rendering.
- **Frozen as built**: when features land on the default path, catalog entries aren't updated. They drift, that's expected, and we only fix the drift if an entry is ever promoted.
- **Accessibility debt is deliberate**, not an oversight. It's on the promotion checklist, not the build checklist.
- **Don't refactor across entries.** Persisted raw values stay stable; display names can change.

A test checks the partition, not discipline. Every entry must be in exactly one group: production, near-shipping, kept, or pending deletion.

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

The suite checks that every entry is in a group, that no entry is in two, that the count matches, and that production holds exactly one style. Add an entry without classifying it and the test fails.

### Two performance findings worth keeping

Fan-out produces work nobody has profiled, and two problems came up more than once:

- **Hundreds of views animating at once brings the app to a crawl.** Sequence the groups so only one animates at a time, and add `drawingGroup()` per row to collapse compositing layers.
- **`rotation3DEffect` at exactly ±90° with perspective** produces a degenerate transform and logs "ignoring singular matrix" every frame. Clamp to ±89.9°.

Hold any entrance animation until loading finishes, so it never competes with launch and refresh work.

## Case Study: 27 Locales

Same shape, different domain, and a much stricter check. The task: translate 191 Android strings into 27 locales, one agent per locale, all 27 at once.

Here the brief was an anchor rather than a template. Each agent was pointed at the iOS app's professionally reviewed translation file for the same locale and told to match its terminology. That removes the biggest source of variance in parallel translation: 27 agents each deciding on their own how to say "feels like" or "chance of rain." People had already made those decisions.

Central wiring came first, exactly as on iOS. One separate commit declared 28 locales in the locale config (English plus the 27 targets). The same commit wired the per-app language picker and added the AppCompat service that keeps the user's choice across OS versions. It had no translation files in it, so with only English resources present it changed nothing. It was merged before a single translation agent ran.

The check is entirely mechanical. Every locale must pass:

- Key-set equality with the source file (no added strings, no dropped strings, no renamed `name` attributes)
- Format-specifier preservation: `%1$s`, `%d`, `%1$.0f`, `%%`, present, correct, and in the right order
- Escaping rules: backslash-escaped apostrophes and `@`, entities or CDATA for ampersands and ellipses
- XML validity
- A full resource merge and compile with all 27 locales present

None of those checks requires reading the translation. A parallel pipeline needs a check that says yes or no without a person in the loop, and "does this Czech sentence read well" isn't that check. Key-set equality is. The remaining risk is written down rather than pretended away: mechanical checks confirm structure, not fluency. We planned professional post-editing as a separate pass, then waived it on 2026-07-31 in favor of the model-only quality pass that had cleared the iOS launch.

Android has its own version of the isolation problem. It turns on any `values-XX/strings.xml` present in the build:

> Android automatically enables any `values-XX/strings.xml` present in the build — unlike iOS, there is no debug-only option. Only create translation files when ready to ship them to all users.

There's no debug flag, so merging the branch is the launch. The branch itself became the isolation boundary. It stays a deliberate draft, complete and verified and unmerged, until someone decides to ship localization, and the commit message says so.

On iOS, what shipped is a catalog of about 168 entries, up from 41 in eleven days. That's about 173,000 lines of debug-only code that never changes default rendering, and a test keeps it partitioned from production. The accepted cost is drift. Entries are frozen as built and carry deliberate accessibility debt, and we only fix either if an entry is promoted. On Android, what exists is 191 strings across 27 locales, verified and unmerged, waiting for the release that ships localization. The professional post-editing pass was waived on 2026-07-31.

## Lessons Learned

- **Fix a fan-out with structure, not judgment.** A fixed brief shape, central wiring, and a mechanical check each remove one kind of failure. "Review more carefully" removes none.
- **A check that needs taste won't run at scale.** Key-set equality, format-specifier order, a clean parse, and audit greps say yes or no without a person.
- **Copies beat references when quality is uneven.** If rough code calls production code, every later production change has to be checked against all of it. Once the catalog was culled, the copies rule was retired; it was scaffolding for the fan-out, not permanent architecture.
- **The isolation boundary can live at any layer, but it has to exist and be written down.** A debug flag, a copy-not-reference rule, and an unmerged branch all work.
- **Write down the remaining risk instead of hiding it.** The deliberate accessibility debt and the translation review decision, including its later waiver, are both on record.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the two runs and their figures, each pipeline phase and the contract say their part once, the case study closes with what shipped and what it cost in place of a Results section, and Lessons Learned went from eleven bullets to five. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The 41-to-168 jump took eleven days (2026-07-15 to 07-26), not a month; 81,000 lines was the third pack alone, so the catalog total (about 173,000 lines) is now stated separately; the copies-not-references contract is dated to 2026-07-26 and the post now records that the skill retired it on 2026-08-21 in favor of styles sharing the real views, and the stat-chart rewrite is described as proceeding behind a flag rather than shipped. The Swift test excerpt now matches the real file, the locale config is described as 28 entries (English plus the 27 translated locales), and the Android professional post-editing pass is recorded as waived on 2026-07-31 rather than still pending.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 2, run after batch 1 (#68) merged. The prose now calls the isolation contract a rule (the title, slug, and h2 keep "contract"), defines orchestrator and the codemod's anchor at first use, drops the clefts and closers ("coupling is what turns", "the boundary is an assertion, not a convention", "undocumented debt is the kind that surprises you"), and settles on one word each for drift, the debug flag, the check, and the catalog entry. Judgment calls: the five aesthetic families and the six hard rules stay as lists because they are the facts, not decoration; "verifier" and "gate" became "check" everywhere except the quoted rules. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"

**Prompt 9:** "usage looks fine, merge it and run batch 2"
