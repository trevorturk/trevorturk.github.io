---
layout: post
title: "The Agent-Doc Audit: Deleting the Fiction Your AI Has Been Believing"
date: 2026-07-29 08:10:00 -0600
summary: "A four-tier audit of every agent-facing doc across three repos. Wrong content is worse than no content, because agents act on it - and deleting a quarter to a third of the corpus made the agents better."
tags: [ai-agents, documentation, skills, workflow]
---

## The Problem

Agent-facing documentation - `AGENTS.md`, `CLAUDE.md`, `.claude/skills/**/SKILL.md`, permission settings - accumulates the same way any documentation does. Someone writes a skill, the code moves, nobody re-reads the skill. A year later you have a corpus that is mostly true.

For human documentation, "mostly true" is survivable. A developer reads "use the Navigation component," glances at the code, sees a custom router, and silently corrects for it. Humans triangulate.

Agents don't. An agent reads your skill file, believes it, and acts. A stale rule doesn't get quietly ignored - it gets *implemented*. The worst case isn't the agent failing to help; it's the agent confidently "fixing" working code to match a doctrine that was never true here.

So: **wrong content is worse than no content.** That premise drove a full audit of every agent-facing doc across the three [Hello Weather](https://helloweather.com) repos - web (Rails), iOS (Swift), and Android (Kotlin) - in July 2026.

The audit found what you'd expect: duplication, bloat, drift. What it didn't expect was how much of the content was outright *fiction* - not stale, not imprecise, but describing software that has never existed.

## The Solution

Four tiers, run in order, each one a separate PR per repo. The ordering matters: safety before content, deletion before deduplication, deduplication before merging.

- **Tier 0 - permission safety.** Fix what the agent is *allowed* to do before touching what it *knows*.
- **Tier 1 - delete the fiction.** Anything actively wrong comes out. No rewriting, no softening. Delete.
- **Tier 2 - de-duplicate.** Every rule gets exactly one canonical owner. Everywhere else gets a pointer.
- **Tier 3 - merge and trim.** Collapse overlapping skills, cut stock knowledge, keep only project-specific content.

And one non-negotiable rule underneath all four: **every deletion is verified against the codebase first.** Grep for the symbol. Open the manifest. Check the dependency list. Don't delete because it *feels* stale.

That rule turned out to be the whole point. The verification pass found more drift than the audit did.

## Tier 0: Permission Safety

This tier isn't about content at all, and it goes first because a documentation audit means an agent editing the files that constrain the agent.

The findings were mostly about rules that looked like protection but weren't:

- **Dead allow-rules.** All three repos had an allow entry scoped to read-only GitHub API calls whose prefix pattern could never match a real invocation. It granted nothing while looking like it granted something specific - so the actual behavior was the default, not the documented intent. Fixed by removing the dead rule and explicitly denying the mutating HTTP methods.
- **Interpreters on the allow list.** `ruby` and `python3` were auto-allowed. A one-line `ruby -e 'system(...)'` routes around every push, deploy, and staging guard sitting above it. Moved to ask-by-default.
- **Blanket `find`.** `find` was allowed wholesale, which quietly includes `-exec`, `-ok`, and `-delete`. Plain `find` stays allowed; the executing and deleting forms now prompt.
- **Mutating `curl`.** The documented intent was "read-only research." The rules didn't enforce it on both sides. Now they do.
- **Self-escalation.** An agent with unconditional Edit and Write access could widen its own allowlist by editing the settings file. Edits to the agent's own settings files now require confirmation.
- **An empty deny list.** One repo's deny array was literally `[]` - pushes to `main` were ask-only.

One more, found in a machine-local settings file that isn't in version control: a read grant covering the **entire home directory**. Silent, unprompted reads of everything outside the project. That one is worth checking for specifically, because it lives in the file nobody reviews.

## Tier 1: Delete the Fiction

This is the tier that justifies the whole exercise.

**Fictional product IDs in the revenue path.** The Android billing skill documented a `ProductIds` object with six constants across two labeled tiers - "Current plans" and "Legacy plans (still valid)" - complete with a derived `ALL_ACTIVE` list. It looked authoritative. Every string in it was invented:

```kotlin
// Deleted. Real SKUs replaced with placeholders here;
// the originals were equally fake.
object ProductIds {
    const val MONTHLY = "PRODUCT_MONTHLY_PLACEHOLDER"
    const val LIFETIME = "PRODUCT_LIFETIME_PLACEHOLDER"
    // ...six total, in two tiers, none of them real
}
```

Grepping the entire tree for those strings returned three hits, all of them *inside the skill file itself*. Zero hits in Kotlin, Gradle, XML, or JSON. The skill even contradicted itself - an earlier code sample in the same file used a third, different naming convention.

This is the nightmare case. It's in the billing path. An agent asked to "add the new yearly plan" would have followed a naming convention that doesn't exist, in the one subsystem where a wrong string means lost revenue and a broken migration.

The replacement skill opens by naming its own source of truth and conceding to it:

> All billing knowledge here is derived from the source of truth: `FanClubHelper.kt` (SKUs + entitlement logic) and `PaymentProcessor.kt` (Billing Library client). If this file and the code disagree, the code wins - update this file.

**Architecture claims contradicted by the source.** Android's `AGENTS.md` told agents to "use the Navigation component" and "follow single-activity architecture where possible." The app uses a hand-rolled `Router` class that dispatches URL strings across a dozen Activities - the opposite of single-activity. And `androidx.navigation` isn't a dependency; grepping every Gradle file for "navigation" returns nothing at all.

The same file claimed **"widgets run in a separate process."** There is no `android:process` attribute anywhere in the manifest. Widget code runs in-process; only the RemoteViews are rendered by the launcher. That's a meaningfully different mental model, and an agent reasoning about widget state from the false version would reach wrong conclusions about what's safe to share.

Also in that batch: a Koin dependency-injection example citing an `ApiService.create()` that returns zero grep hits, and a "check for memory leaks with LeakCanary" instruction for a project that doesn't depend on LeakCanary.

**Cross-project contamination.** The web repo's headline finding was that its config was effectively *two codebases interleaved*. Roughly six skills carried doctrine from a previous, unrelated project - models and multi-tenancy conventions this app has never had, a database skill insisting on UUID primary keys when the schema uses bigint everywhere (the word "uuid" appears exactly once in the entire schema).

The sharpest instance was a rule stated with maximum authority:

```markdown
## 🚨 CRITICAL: Never Use `before_action :set_*` Patterns
```

The codebase uses `before_action :set_*` in seven controllers, including `application_controller.rb` and the primary web controller. It's in the frontmatter description too, so it fires on keyword triggering. An obedient agent asked to clean up controllers would have "fixed" the entire app to comply with a rule imported from somewhere else.

iOS had the same disease from a different donor. Its PR skill was contaminated with a Rails project's content - including, in an *iOS* PR description template:

```markdown
## Test Results
bin/rails test
# X runs, Y assertions, 0 failures ✅
```

Alongside it: instructions to delete debug artifacts named `*.rb`, a "organize changes by Database → Model → Controller → View" section, and security-review templates referencing a Ruby static analyzer.

**Self-review theater.** Both the iOS and web PR skills instructed the agent to append this to its own pull requests:

```markdown
## Code Review Summary

**Grade: A-** - Ready to merge

### Strengths
- ✅ Clean architecture
- ✅ Comprehensive testing
- ✅ All conventions followed
```

An agent grading its own work an A- and declaring it ready to merge is not a code review. It's a confidence signal with no information content, and it actively degrades human review by putting a green checkmark where scrutiny should go.

**Examples from another repo's PR numbers.** Both repos' PR skills carried an "Examples from Recent PRs" section citing PR #145, #144, and #132 as templates to study. The web repo was at #1600 at the time. Those numbers point at real PRs - just entirely different ones, about weather forecast views rather than the subjects named. My favorite detail: the section advising agents *not to name your own app in upstream open-source PRs* named the wrong app while doing it.

**Documentation of CI that never existed.** The iOS README stated: "Tests are automatically run on GitHub Actions for pull requests and pushes to main." There was no `.github/` directory. Not a broken one - the directory had never existed in the repo's history. (Twelve days after the audit, CI was actually built. The audit told the truth about July 16; the repo then went and made a better truth.)

**Stale deadline tables.** Android's `AGENTS.md` had a "Pending Upgrades" section listing "Target SDK 35: required by August 31, 2025." The deadline was eleven months in the past, and the requirement itself was two SDK generations stale. It was replaced with a single line pointing at the one file that tracks deadlines, plus an explicit warning not to trust version claims found elsewhere.

**Content that only restates default model behavior.** A ~230-word section titled "AI Assistant Self-Reflection and Critical Thinking" instructed the agent to "maintain a stance of being extraordinarily skeptical of your own correctness" and "live in constant fear of being wrong," with subsections on red-teaming its own code. This is what the model does anyway. It costs tokens in an always-loaded file, and volume of generic instruction is exactly what causes specific instructions to get diluted. Deleted.

**Skills that were silently not loading.** Two skills - one on Android, one on web - had no YAML frontmatter at all. The file opened directly with an H1. With no `name` or `description`, the listing degrades to the bare title and keyword triggering never fires. These skills had been effectively invisible for months, and nothing surfaced an error.

And one that only breaks on someone else's machine: a web skill was tracked in git as lowercase `skill.md` while all 33 of its siblings were `SKILL.md`. On a case-insensitive filesystem it resolves fine. On a case-sensitive checkout - Linux CI, a case-sensitive volume - that skill simply does not exist.

## Tier 2: One Canonical Owner

With the fiction gone, the next problem is the same truth stated in four places, drifting apart at four different rates.

Android's crash-prevention rules - no force unwraps, bounds-check collection access, no swallowed exceptions - existed in **triplicate**: `AGENTS.md`, the Kotlin conventions skill, and the code-review skill. Three copies means three chances to be wrong and no way to tell which is current.

The fix is mechanical: pick the canonical owner, delete the copies, leave pointers.

For rules, the canonical owner should be the **always-loaded file** (`AGENTS.md`). Skills then keep only their genuinely skill-specific additions. On iOS, the widget and watch skills each carried a ~45-line generic crash-prevention section that was textbook Swift, not project knowledge. Collapsed to one sentence each plus the unique parts - the fixed-slot `safeCount` pattern for widgets, and for the watch, the framing that actually matters:

> The canonical rules live in AGENTS.md "Crash Prevention (Non-Negotiable)". They apply doubly here: a watch crash often presents as a complication that silently stops updating.

While rewriting that section we found the watch skill's own crash-prevention example doing this:

```swift
let nextRefresh = Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
```

A force unwrap, inside a crash-prevention skill, in the code sample demonstrating crash prevention.

For **facts**, the canonical owner should be whatever the build reads. Android's `AGENTS.md` had an eleven-row dependency/version table duplicated word-for-word with the README. Both were wrong in different places. Now the README holds one table and `build.gradle` is named as ground truth:

> The dependency/version table lives in README.md "Key Technologies" (single home - version tables rot fastest); `app/build.gradle` is the ground truth.

On web the same pass took `AGENTS.md` down 18% - including one ~200-word bullet on fresh-worktree setup that got compressed to a sentence plus a pointer. It also added something: a bullet documenting that the app server shares a fiber reactor, so blocking CPU on the request path stalls every concurrent request in the process. That's the kind of content agent docs *should* hold - a non-obvious operational invariant you cannot infer by reading a single file.

## Tier 3: Merge and Trim to Project-Specific Content

The last tier asks a harder question of every remaining paragraph: **does the model already know this?**

iOS had two conventions skills, `swift-conventions` and `swiftui-conventions`, totaling 2,671 words. Most of it was stock knowledge - deprecated-to-modern API tables, a Swift concurrency tutorial, string formatting reference:

```markdown
| Deprecated | Modern Replacement |
|------------|-------------------|
| `foregroundColor()` | `foregroundStyle()` |
| `NavigationStack` | ... |
```

The model knows this. It has known it for years. Paying tokens to tell it costs context that project-specific rules need. Merged into one 587-word skill, a 78% cut, framed by a sentence that states the policy:

> Stock modern-API knowledge is assumed - this file records only project decisions.

What survived is the stuff no model could know: that this project deliberately stays on `ObservableObject` rather than migrating to `@Observable` (verified: ~27 files use it, zero use the newer macro), and the exact exception to the ForEach-identity rule that fixed forecast slots depend on.

One deleted skill cited its source as a PDF in the author's `~/Downloads`. No future agent can read that. If the provenance isn't in the repo, the claim isn't verifiable.

Then the verification pass, which found more than the audit had:

- An onboarding skill documenting a `regionBasedSource()` function that doesn't exist, with routing logic that bore no relation to the real routing.
- A widget skill documenting a **Control Center widget the app does not have.**
- A watch skill claiming a complication used "cached data only" when it does fetch, and describing a `@Published` var that is actually computed.
- A localization skill with a 27-row table of translation folders - two tiers, high-priority and extended, every row with a `values-XX` path. The resource directory contains no translation folders at all. Replaced with one honest line: *"Current state (verified 2026-07-16): no translations exist yet."*
- Two skills recommending opposite things: a performance skill suggesting a silent `|| "unknown"` fallback for unmapped weather codes, which the weather-sources skill explicitly forbids.
- A debugging guide whose central command was on the repo's own permission **deny list**. The guide existed to tell agents to open gem source; opening gem source was blocked. It had been unusable since the day the deny rule landed.

The gold survived verbatim - a hard-won troubleshooting section tied to a specific PR, a membership-exceptions rule, a geometry check with the commit SHA that introduced it. Those references were re-verified against git log before being kept.

One Tier 3 item isn't about skills at all: personal contact details for the business owners were sitting in the web README - a file every agent session reads - and moved to a separate document with a pointer left behind. Worth auditing your always-loaded files for anything you wouldn't want in a context window.

## Results

| Repo | Skills | Corpus words | Change |
|---|---|---|---|
| Web | 34 → 28 | 40,658 → 31,140 | −23% |
| iOS | 27 → 22 | ~28,000 → ~21,000 | −25% |
| Android | 7 → 5 | 5,847 → 3,111 | −47% |

Plus, on Android, `AGENTS.md` down ~300 words with zero loss of project-specific signal, and on iOS, individual skill rewrites like the PR skill going 1,894 → 901 words.

One number deserves its own line. iOS skill **descriptions** - the frontmatter blurbs that load at session start, before any skill is selected - totaled roughly 4,000 words. One skill's description alone ran ~120 words. That's the always-loaded cost of a large skill corpus, paid on every single session whether or not any skill is used. Trimming that description to two sentences is free performance.

The counterintuitive part: **deleting a quarter to a third of your agent docs makes the agents better.** Not "no worse." Better. Every deleted fiction is an entire class of confident wrong action that can no longer happen, and every deleted paragraph of stock knowledge is context returned to the rules that are actually load-bearing.

## Lessons Learned

**Grep every claim in your skill against the source.** This is the one-line test, and it's the whole audit compressed. If a skill names a function, grep for it. If it names a SKU, grep for it. If it names a dependency, check the manifest. If it names a file path, `ls` it. Claims that survive that pass are worth keeping; claims that don't were actively harming you.

**Wrong is worse than missing.** A missing rule produces an agent that asks or guesses conservatively. A wrong rule produces an agent that acts confidently against your codebase. If you can't verify a claim, delete it - the absence is strictly safer than the fiction.

**Version tables are the fastest-rotting content.** Dependency versions, SDK deadlines, tool versions, language-support tables. They are wrong within months, they look authoritative, and they're the easiest thing to replace with a pointer at whatever file the build actually reads.

**One rule, one owner, pointers everywhere else.** Duplication isn't just bloat - it's drift with extra steps. Three copies of a rule means three independent decay curves and no signal about which one is current. Rules belong in the always-loaded file; skills keep only their skill-specific additions.

**Always-loaded content has a token cost, and it's charged on every session.** `AGENTS.md`, `CLAUDE.md`, and every skill description load before the work starts. Anything generic in there - "be skeptical," "think carefully," a stock API table - is diluting the specific instructions you actually needed the model to follow.

**If the model already knows it, don't write it.** Your docs exist to record decisions the model cannot infer: which pattern this project chose and why, which SKU strings are live in production, which subsystem crashes silently. Not the language reference.

**Fix permissions before you edit docs.** Tier 0 first is not ceremony. A documentation audit is an agent editing the files that constrain the agent, and self-escalation paths - unconditional write access to your own settings file - should be closed before that starts.

**The pattern outlives the audit.** The clearest sign it worked: two weeks later, a routine PR in the web repo moved a section out of the README and into a dedicated skill, unprompted. Single-source became the default reflex rather than a cleanup task.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.
