---
layout: post
title: "Delete the Fiction Your Agents Believe"
date: 2026-07-29 08:10:00 -0600
summary: "A four-tier audit of every agent-facing doc across three repos. Wrong content is worse than no content, because agents act on it - and deleting a third to nearly half of the corpus made the agents better."
tags: [ai-agents, documentation, skills, workflow]
model: "Claude"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

## The Problem

An Android billing skill listed six product IDs in two tiers, "Current v4 plans" and "Legacy v3 plans (still valid)". A Rails skill banned, in capital letters, a pattern seven of the app's controllers use. An iOS README described CI that had never existed in the repo's history. All three lived in agent-facing docs: the `AGENTS.md`, `CLAUDE.md`, `.claude/skills/**/SKILL.md` files, and permission settings that an agent reads before it acts.

Agent docs rot the way any docs do. Someone writes a skill, the code moves, nobody re-reads the skill. For human documentation, "mostly true" is survivable. A developer reads "use the Navigation component," glances at the code, sees a custom router, and silently corrects for it. An agent reads the skill, believes it, and acts. A stale rule is not quietly ignored; it gets implemented. The worst case is the agent confidently "fixing" working code to match a doctrine that was never true here.

So wrong content is worse than no content. That premise drove a full audit of every agent-facing doc across the three [Hello Weather](https://helloweather.com) repos, web (Rails), iOS (Swift), and Android (Kotlin), in July 2026. The audit expected duplication, bloat, and drift, and found them. It also found a good deal of outright fiction: not stale, not imprecise, but describing software that has never existed.

## The Solution

Four tiers, run in order, each one a separate PR per repo. Safety comes before content, deletion before deduplication, deduplication before merging.

- **Tier 0 - permission safety.** Fix what the agent is *allowed* to do before touching what it *knows*.
- **Tier 1 - delete the fiction.** Anything actively wrong comes out. No rewriting, no softening.
- **Tier 2 - de-duplicate.** Every rule gets exactly one canonical owner. Everywhere else gets a pointer.
- **Tier 3 - merge and trim.** Collapse overlapping skills, cut stock knowledge, keep only project-specific content.

One rule sits under all four: every deletion is verified against the codebase first. Grep for the symbol. Open the manifest. Check the dependency list. Nothing is deleted because it feels stale. That verification pass found more drift than the audit itself had.

## Tier 0: Permission Safety

This tier goes first because a documentation audit is an agent editing the files that constrain the agent. Most of the findings were rules that looked like protection and were not:

- **Dead allow-rules.** All three repos had an allow entry scoped to read-only GitHub API calls whose prefix pattern could never match a real invocation. It granted nothing while looking specific, so the actual behavior was the default. Fixed by removing the dead rule and explicitly denying the mutating HTTP methods.
- **Interpreters on the allow list.** `ruby` and `python3` were auto-allowed. A one-line `ruby -e 'system(...)'` routes around every push, deploy, and staging guard above it. Moved to ask-by-default.
- **Blanket `find`.** `find` was allowed wholesale, which quietly includes `-exec`, `-ok`, and `-delete`. Plain `find` stays allowed; the executing and deleting forms now prompt.
- **Mutating `curl`.** The documented intent was "read-only research." The rules didn't enforce it on both sides. Now they do.
- **Self-escalation.** An agent with unconditional Edit and Write access could widen its own allowlist by editing the settings file. One repo closed this: edits to the agent's own settings files now require confirmation there.
- **An empty deny list.** One repo's deny array was literally `[]`, so pushes to `main` were ask-only.

One more turned up in a machine-local settings file that is not in version control: a read grant covering the entire home directory. Silent, unprompted reads of everything outside the project. Worth checking for specifically, because it lives in the file nobody reviews.

## Tier 1: Delete the Fiction

### Invented product IDs in the billing path

The Android billing skill documented a `ProductIds` object: six constants across two labeled tiers, "Current v4 plans" and "Legacy v3 plans (still valid)", with a derived `ALL_ACTIVE` list. Every string in it was invented:

```kotlin
// Deleted. Real SKUs replaced with placeholders here;
// the originals were equally fake.
object ProductIds {
    const val MONTHLY = "PRODUCT_MONTHLY_PLACEHOLDER"
    const val LIFETIME = "PRODUCT_LIFETIME_PLACEHOLDER"
    // ...six total, in two tiers, none of them real
}
```

Grepping the entire tree for those strings returned three hits, all inside the skill file itself, and none in Kotlin, Gradle, XML, or JSON. An earlier code sample in the same file used a third, different naming convention. An agent asked to "add the new yearly plan" would have followed a convention that does not exist, in the one subsystem where a wrong string means lost revenue and a broken migration.

The replacement skill opens by naming its own source of truth and conceding to it:

> All billing knowledge here is derived from the source of truth: `feature/fanclub/FanClubHelper.kt` (SKUs + entitlement logic) and `app/PaymentProcessor.kt` (Billing Library client). If this file and the code disagree, the code wins - update this file.

### Architecture claims the source contradicts

Android's `AGENTS.md` told agents to "use the Navigation component" and "follow single-activity architecture where possible." The app uses a hand-rolled `Router` class that dispatches URL strings across a dozen Activities, the opposite of single-activity. `androidx.navigation` is not a dependency; grepping every Gradle file for "navigation" returns nothing at all.

The same file claimed "widgets run in a separate process." There is no `android:process` attribute anywhere in the manifest. Widget code runs in-process; only the RemoteViews are rendered by the launcher, which changes what is safe to share.

Also in that batch: a Koin dependency-injection example citing an `ApiService.create()` that returns zero grep hits, and a "check for memory leaks with LeakCanary" instruction for a project that does not depend on LeakCanary.

### Doctrine imported from other projects

The web repo's config was effectively two codebases interleaved. Roughly six skills carried doctrine from a previous, unrelated project. They described models and multi-tenancy conventions this app has never had. A database skill insisted on UUID primary keys when the schema uses bigint everywhere; the word "uuid" appears exactly once in the entire schema.

The sharpest instance was a rule stated with maximum authority:

```markdown
## 🚨 CRITICAL: Never Use `before_action :set_*` Patterns
```

The codebase uses `before_action :set_*` in seven controllers, including `application_controller.rb` and the primary web controller. The rule was in the frontmatter description too, so it fired on keyword triggering. An obedient agent asked to clean up controllers would have "fixed" the entire app to comply.

iOS had the same disease from a different donor. Its PR skill was contaminated with a Rails project's content, including this in an *iOS* PR description template:

```markdown
## Test Results
bin/rails test
# X runs, Y assertions, 0 failures ✅
```

Alongside it: instructions to delete debug artifacts named `*.rb`, an "organize changes by Database → Model → Controller → View" section, and security-review templates referencing a Ruby static analyzer.

### Self-review theater and borrowed examples

Both the iOS and web PR skills instructed the agent to append this to its own pull requests:

```markdown
## Code Review Summary

**Grade: A-** - Ready to merge

### Strengths
- ✅ Clean architecture
- ✅ Comprehensive testing
- ✅ All conventions followed
```

An agent grading its own work an A- and declaring it ready to merge is a confidence signal with no information content. It degrades human review by putting a green checkmark where scrutiny should go.

The same two skills carried an "Examples from Recent PRs" section citing PR #145, #144, and #132 as templates to study. The web repo was at #1600 at the time. In the iOS repo those numbers point at real PRs, just entirely different ones: forecast views, JSON encoders, and a color manager rather than the subjects named. The section advising agents not to name your own app in upstream open-source PRs named the wrong app while doing it.

### Infrastructure that never existed

The iOS README stated: "Tests are automatically run on GitHub Actions for pull requests and pushes to main." There was no `.github/` directory, and there never had been one in the repo's history. (Twelve days after the audit, CI was actually built. On July 16, the claim was false.)

Android's `AGENTS.md` had a "Pending Upgrades" section listing "Target SDK 35: required by August 31, 2025." The deadline was ten and a half months in the past, and the requirement itself was one SDK generation stale. It was replaced with a single line pointing at the one file that tracks deadlines, plus a warning not to trust version claims found elsewhere.

### Restated defaults, and skills that never loaded

A ~230-word section titled "AI Assistant Self-Reflection and Critical Thinking" instructed the agent to "maintain a stance of being extraordinarily skeptical of your own correctness" and "live in constant fear of being wrong," with subsections on red-teaming its own code. This is what the model does anyway, and it sat in an always-loaded file. Deleted.

Three skills, one in each repo, had no YAML frontmatter at all. With no `name` or `description`, the listing degrades to the bare title and keyword triggering never fires. All three had been effectively invisible for months, and nothing surfaced an error.

One more breaks only on someone else's machine. A web skill was tracked in git as lowercase `skill.md` while all 33 of its siblings were `SKILL.md`. On a case-insensitive filesystem it resolves fine. On a case-sensitive checkout, Linux CI or a case-sensitive volume, that skill does not exist.

## Tier 2: One Canonical Owner

With the fiction gone, the next problem is the same truth stated in four places, drifting apart at four different rates. Android's crash-prevention rules (no force unwraps, bounds-check collection access, no swallowed exceptions) existed in triplicate: `AGENTS.md`, the Kotlin conventions skill, and the code-review skill. Three copies means three chances to be wrong and no way to tell which is current.

The fix is mechanical: pick the canonical owner, delete the copies, leave pointers. For rules, the owner is the always-loaded file, `AGENTS.md`, and skills keep only their skill-specific additions. On iOS, the widget and watch skills each carried a ~45-line generic crash-prevention section that was textbook Swift, not project knowledge. Each collapsed to one sentence plus the unique parts: the fixed-slot `safeCount` pattern for widgets, and for the watch, the framing that matters:

> The canonical rules live in AGENTS.md "Crash Prevention (Non-Negotiable)" - no force unwraps, no unguarded indexing/casts/division, fallbacks for missing API data. They apply doubly here: a watch crash often presents as a complication that silently stops updating.

While rewriting that section we found the watch skill's own crash-prevention example doing this:

```swift
let nextRefresh = Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
```

A force unwrap, inside a crash-prevention skill, in the code sample demonstrating crash prevention.

For facts, the owner is whatever the build reads. Android's `AGENTS.md` had an eleven-row dependency/version table duplicated word-for-word with the README, and both were wrong in different places. Now the README holds one table and `build.gradle` is named as ground truth:

> The dependency/version table lives in README.md "Key Technologies" (single home - version tables rot fastest); `app/build.gradle` is the ground truth.

On web the same pass took `AGENTS.md` down 18%, including one ~200-word bullet on fresh-worktree setup compressed to a sentence plus a pointer. It also added a bullet: the app server shares a fiber reactor, so blocking CPU on the request path stalls every concurrent request in the process. That is what agent docs should hold, an operational invariant you cannot infer by reading a single file.

## Tier 3: Merge and Trim to Project-Specific Content

The last tier asks a harder question of every remaining paragraph: does the model already know this?

iOS had two conventions skills, `swift-conventions` and `swiftui-conventions`, totaling 2,671 words. Most of it was stock knowledge: deprecated-to-modern API lists, a Swift concurrency tutorial, a string formatting reference:

```swift
// ❌ Deprecated
.foregroundColor(.red)
.cornerRadius(8)
NavigationView { }
.accentColor(.blue)

// ✅ Modern (iOS 15+)
.foregroundStyle(.red)
.clipShape(.rect(cornerRadius: 8))
NavigationStack { }
.tint(.blue)
```

The model has known this for years, and paying tokens to tell it costs context that project-specific rules need. The two merged into one 587-word skill, a 78% cut, framed by a sentence that states the policy:

> Stock modern-API knowledge (prefer `foregroundStyle`, `NavigationStack`, `clipShape(.rect(cornerRadius:))`, `Task.sleep(for:)`, etc.) is assumed - this file records only project decisions.

What survived is what no model could know: this project deliberately stays on `ObservableObject` rather than migrating to `@Observable` (verified: ~27 files use it, zero use the newer macro), and the exact exception to the ForEach-identity rule that fixed forecast slots depend on.

One deleted skill cited its source as a PDF in the author's `~/Downloads`. No future agent can read that. If the provenance is not in the repo, the claim is not verifiable.

The verification pass turned up the rest:

- An onboarding skill documenting a `regionBasedSource()` function that does not exist, with routing logic that bore no relation to the real routing.
- A widget skill documenting a Control Center widget the app does not have.
- A watch skill claiming a complication used "cached data only" when it does fetch, and describing a `@Published` var that is actually computed.
- A localization skill with a 27-row table of translation folders, two tiers, high-priority and extended, every row with a `values-XX` path. The resource directory contains no translation folders at all. Replaced with one honest line: *"Current state (verified 2026-07-16): no translations exist yet."*
- Two skills recommending opposite things: a performance skill suggesting a silent `|| "unknown"` fallback for unmapped weather codes, which the weather-sources skill explicitly forbids.
- A debugging guide whose central command was on the repo's own permission deny list. The guide existed to tell agents to open gem source; opening gem source was blocked. It had been unusable since the day the deny rule landed.

The hard-won content survived verbatim: a troubleshooting section tied to a specific PR, a membership-exceptions rule, a geometry check with the commit SHA that introduced it. Those references were re-verified against git log before being kept.

One Tier 3 item isn't about skills at all: personal contact details for the business owners were sitting in the web README - a file every agent session reads - and moved to a separate document with a pointer left behind. Worth auditing your always-loaded files for anything you wouldn't want in a context window.

## Results

| Repo | Skills | Corpus words | Change |
|---|---|---|---|
| Web | 34 → 28 | 45,459 → 31,140 | −31% |
| iOS | 27 → 22 | 30,535 → 20,437 | −33% |
| Android | 7 → 5 | 5,847 → 3,111 | −47% |

On Android, `AGENTS.md` also dropped ~300 words with zero loss of project-specific signal. On iOS, individual skill rewrites went further, the PR skill from 1,894 to 901 words.

One cost was hiding in the frontmatter. iOS skill frontmatter, the blurbs that load at session start before any skill is selected, totaled roughly 1,100 words. One skill's description alone ran ~90 words. That is paid on every session whether or not any skill is used, and trimming a description to two sentences is free.

Deleting a third to nearly half of the agent docs made the agents better, not merely no worse. Every deleted fiction is a class of confident wrong action that can no longer happen, and every deleted paragraph of stock knowledge is context returned to project-specific rules.

The pattern outlived the audit. Two weeks later, a routine PR in the web repo moved a section out of the README and into a dedicated skill, unprompted. Single-source became the default reflex rather than a cleanup task.

## Lessons Learned

- **Grep every claim in a skill against the source.** Function names, SKUs, dependencies, file paths. What fails the grep is deleted, not softened.
- **Version tables are the fastest-rotting content.** Dependency versions, SDK deadlines, tool versions. Replace them with a pointer at whatever file the build actually reads.
- **Always-loaded content is charged on every session.** Anything generic in `AGENTS.md` or a skill description dilutes the specific instructions the model actually needed.
- **A source outside the repo is no source.** If a claim's provenance is a file on one person's machine, no future agent can check it. Treat the claim as unverified.
