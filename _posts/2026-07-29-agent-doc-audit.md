---
layout: post
title: "Delete the Fiction Your Agents Believe"
date: 2026-07-29 08:10:00 -0600
summary: "We audited every agent-facing doc across three repos in four tiers. Wrong content is worse than none, because agents act on it. Deleting a third to nearly half of the docs made the agents better."
tags: [ai-agents, documentation, skills, workflow]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

An Android billing skill listed six product IDs in two tiers, "Current v4 plans" and "Legacy v3 plans (still valid)". A Rails skill banned, in capital letters, a pattern seven of the app's controllers use. An iOS README described CI that had never existed in the repo's history. All three lived in the docs an agent reads before it acts: `AGENTS.md`, `CLAUDE.md`, the `.claude/skills/**/SKILL.md` files, and the permission settings.

Agent docs rot like any other docs. Someone writes a skill, the code moves on, and nobody rereads the skill. A human reader survives that. A developer reads "use the Navigation component", looks at the code, sees a custom router, and quietly adjusts. An agent reads the skill, believes it, and acts on it. A stale rule doesn't get ignored; it gets implemented. In the worst case the agent "fixes" working code to match a rule that was never true here.

So wrong content is worse than no content. That's why we audited every agent-facing doc across the three [Hello Weather](https://helloweather.com) repos, web (Rails), iOS (Swift), and Android (Kotlin), in July 2026. We expected duplication, bloat, and drift, and found all three. We also found a lot of outright fiction: docs describing software that has never existed.

## The Solution

Four tiers, run in order, one PR per repo per tier. Safety first, then deletion, then de-duplication, then merging.

- **Tier 0 - permission safety.** Fix what the agent is *allowed* to do before touching what it *knows*.
- **Tier 1 - delete the fiction.** Anything wrong comes out. No rewriting, no softening.
- **Tier 2 - de-duplicate.** Every rule gets one home. Everywhere else gets a pointer.
- **Tier 3 - merge and trim.** Collapse overlapping skills, cut what the model already knows, keep the project-specific parts.

One rule applies to all four: we check every deletion against the codebase first. Grep for the symbol. Open the manifest. Read the dependency list. Nothing gets deleted because it feels stale. That checking pass found more drift than the audit itself had.

## Tier 0: Permission Safety

This tier goes first because a doc audit is an agent editing the files that constrain the agent. Most of what we found were rules that looked like protection and weren't:

- **Dead allow-rules.** All three repos allowed read-only GitHub API calls through a prefix pattern that could never match a real command. It granted nothing while looking specific, so the default applied. We removed the dead rule and denied the HTTP methods that change things.
- **Interpreters on the allow list.** `ruby` and `python3` ran without a prompt. A one-line `ruby -e 'system(...)'` gets around every push, deploy, and staging guard above it. Both now ask by default.
- **Blanket `find`.** `find` was allowed wholesale, which quietly includes `-exec`, `-ok`, and `-delete`. Plain `find` is still allowed; the forms that run or delete things now prompt.
- **Mutating `curl`.** The docs said "read-only research." The rules didn't enforce that. Now they do.
- **Self-escalation.** An agent with unconditional Edit and Write access could widen its own allowlist by editing the settings file. One repo closed this: editing the agent's own settings files now needs confirmation there.
- **An empty deny list.** One repo's deny array was `[]`, so a push to `main` only asked first.

One more turned up in a machine-local settings file that isn't in version control: a read grant covering the whole home directory. That's silent reads of everything outside the project. Check for it specifically, because it lives in the file nobody reviews.

## Tier 1: Delete the Fiction

### Invented product IDs in the billing path

The Android billing skill documented a `ProductIds` object: six constants in two labeled tiers, "Current v4 plans" and "Legacy v3 plans (still valid)", plus a derived `ALL_ACTIVE` list. Every string in it was made up:

```kotlin
// Deleted. Real SKUs replaced with placeholders here;
// the originals were equally fake.
object ProductIds {
    const val MONTHLY = "PRODUCT_MONTHLY_PLACEHOLDER"
    const val LIFETIME = "PRODUCT_LIFETIME_PLACEHOLDER"
    // ...six total, in two tiers, none of them real
}
```

Grepping the whole tree for those strings returned three hits, all inside the skill file itself. None in Kotlin, Gradle, XML, or JSON. An earlier code sample in the same file used a third naming convention. An agent asked to "add the new yearly plan" would have followed a convention that doesn't exist, in the one subsystem where a wrong string means lost revenue and a broken migration.

The replacement skill opens by naming its own source of truth and deferring to it:

> All billing knowledge here is derived from the source of truth: `feature/fanclub/FanClubHelper.kt` (SKUs + entitlement logic) and `app/PaymentProcessor.kt` (Billing Library client). If this file and the code disagree, the code wins - update this file.

### Architecture claims the source contradicts

Android's `AGENTS.md` told agents to "use the Navigation component" and "follow single-activity architecture where possible." The app uses a hand-written `Router` class that dispatches URL strings across a dozen Activities, which is the opposite of single-activity. `androidx.navigation` isn't a dependency; grepping every Gradle file for "navigation" returns nothing.

The same file said "widgets run in a separate process." There's no `android:process` attribute in the manifest. Widget code runs in-process. Only the RemoteViews are drawn by the launcher, and that changes what's safe to share.

The same batch had a Koin dependency-injection example citing an `ApiService.create()` that returns no grep hits, and a "check for memory leaks with LeakCanary" instruction in a project that doesn't depend on LeakCanary.

### Doctrine imported from other projects

The web repo's config was two codebases interleaved. About six skills carried rules from an earlier, unrelated project. They described models and multi-tenancy conventions this app has never had. A database skill insisted on UUID primary keys when the schema uses bigint everywhere; the word "uuid" appears once in the whole schema.

The sharpest case was this rule:

```markdown
## 🚨 CRITICAL: Never Use `before_action :set_*` Patterns
```

The codebase uses `before_action :set_*` in seven controllers, including `application_controller.rb` and the main web controller. The rule was in the frontmatter description too, so it fired on keyword triggering. An obedient agent asked to clean up controllers would have "fixed" the whole app to comply.

iOS had the same problem from a different donor. Its PR skill was mixed with content from a Rails project, including this in an *iOS* PR description template:

```markdown
## Test Results
bin/rails test
# X runs, Y assertions, 0 failures ✅
```

Next to it: instructions to delete debug files named `*.rb`, an "organize changes by Database → Model → Controller → View" section, and security-review templates that referenced a Ruby static analyzer.

### Self-review theater and borrowed examples

Both the iOS and web PR skills told the agent to append this to its own pull requests:

```markdown
## Code Review Summary

**Grade: A-** - Ready to merge

### Strengths
- ✅ Clean architecture
- ✅ Comprehensive testing
- ✅ All conventions followed
```

An agent grading its own work an A- and calling it ready to merge tells a reviewer nothing. It makes human review worse by putting a green checkmark where scrutiny should go.

The same two skills had an "Examples from Recent PRs" section citing PR #145, #144, and #132 as templates to study. The web repo was at #1600 at the time. In the iOS repo those numbers point at real PRs, just different ones: forecast views, JSON encoders, and a color manager, not the subjects named. The section that told agents not to name your own app in upstream open-source PRs named the wrong app while doing it.

### Infrastructure that never existed

The iOS README said: "Tests are automatically run on GitHub Actions for pull requests and pushes to main." There was no `.github/` directory, and there never had been one in the repo's history. (Twelve days after the audit, CI was built. On July 16, the claim was false.)

Android's `AGENTS.md` had a "Pending Upgrades" section listing "Target SDK 35: required by August 31, 2025." The deadline was ten and a half months in the past, and the requirement itself was one SDK generation stale. We replaced it with a single line pointing at the one file that tracks deadlines, plus a warning not to trust version claims found elsewhere.

### Restated defaults, and skills that never loaded

A ~230-word section titled "AI Assistant Self-Reflection and Critical Thinking" told the agent to "maintain a stance of being extraordinarily skeptical of your own correctness" and "live in constant fear of being wrong," with subsections on red-teaming its own code. The model does this anyway, and the section sat in an always-loaded file. Deleted.

Three skills, one in each repo, had no YAML frontmatter at all. With no `name` or `description`, the listing shows only the bare title and keyword triggering never fires. All three had been invisible for months, and nothing reported an error.

One more breaks only on someone else's machine. A web skill was tracked in git as lowercase `skill.md` while all 33 of its siblings were `SKILL.md`. On a case-insensitive filesystem it resolves fine. On a case-sensitive checkout, like Linux CI, that skill doesn't exist.

## Tier 2: One Canonical Owner

With the fiction gone, the next problem is the same rule written in several places, each copy drifting on its own. Android's crash-prevention rules (no force unwraps, bounds-check collection access, no swallowed exceptions) existed in three places: `AGENTS.md`, the Kotlin conventions skill, and the code-review skill. Three copies means three chances to be wrong and no way to tell which one is current.

The fix is mechanical: pick the owner, delete the copies, leave pointers. For rules, the owner is the always-loaded file, `AGENTS.md`, and skills keep only what's specific to them. On iOS, the widget and watch skills each carried a ~45-line generic crash-prevention section that was textbook Swift, not project knowledge. Each collapsed to one sentence plus the parts unique to it: the fixed-slot `safeCount` pattern for widgets, and for the watch, this framing:

> The canonical rules live in AGENTS.md "Crash Prevention (Non-Negotiable)" - no force unwraps, no unguarded indexing/casts/division, fallbacks for missing API data. They apply doubly here: a watch crash often presents as a complication that silently stops updating.

While rewriting that section we found the watch skill's own crash-prevention example doing this:

```swift
let nextRefresh = Calendar.current.date(byAdding: .hour, value: 1, to: Date())!
```

A force unwrap, inside a crash-prevention skill, in the sample that demonstrates crash prevention.

For facts, the owner is whatever the build reads. Android's `AGENTS.md` had an eleven-row dependency/version table duplicated word for word from the README, and both were wrong in different places. Now the README holds one table and `build.gradle` is named as ground truth:

> The dependency/version table lives in README.md "Key Technologies" (single home - version tables rot fastest); `app/build.gradle` is the ground truth.

On web the same pass cut `AGENTS.md` by 18%, including one ~200-word bullet on fresh-worktree setup that became a sentence plus a pointer. It also added a bullet: the app server shares a fiber reactor, so blocking CPU on the request path stalls every other request in the process. That's what agent docs should hold: a fact about how the system behaves that you can't work out by reading one file.

## Tier 3: Merge and Trim to Project-Specific Content

The last tier asks a harder question of every paragraph that's left: does the model already know this?

iOS had two conventions skills, `swift-conventions` and `swiftui-conventions`, 2,671 words between them. Most of it was stock knowledge: lists of deprecated APIs and their replacements, a Swift concurrency tutorial, a string formatting reference:

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

The model has known this for years. Spending tokens to tell it again takes context away from the project-specific rules. The two skills merged into one 587-word skill, a 78% cut, with a sentence that states the policy:

> Stock modern-API knowledge (prefer `foregroundStyle`, `NavigationStack`, `clipShape(.rect(cornerRadius:))`, `Task.sleep(for:)`, etc.) is assumed - this file records only project decisions.

What survived is what no model could know: this project deliberately stays on `ObservableObject` instead of migrating to `@Observable` (we checked: ~27 files use it, none use the newer macro), and the one exception to the ForEach-identity rule that fixed forecast slots depend on.

One deleted skill cited its source as a PDF in the author's `~/Downloads`. No future agent can read that. If the source isn't in the repo, the claim can't be checked.

The checking pass turned up the rest:

- An onboarding skill documenting a `regionBasedSource()` function that doesn't exist, with routing logic unrelated to the real routing.
- A widget skill documenting a Control Center widget the app doesn't have.
- A watch skill claiming a complication used "cached data only" when it does fetch, and describing a `@Published` var that is actually computed.
- A localization skill with a 27-row table of translation folders, in two tiers, high-priority and extended, every row with a `values-XX` path. The resource directory has no translation folders at all. Replaced with one honest line: *"Current state (verified 2026-07-16): no translations exist yet."*
- Two skills recommending opposite things: a performance skill suggesting a silent `|| "unknown"` fallback for unmapped weather codes, which the weather-sources skill forbids.
- A debugging guide whose central command was on the repo's own deny list. The guide existed to tell agents to open gem source, and opening gem source was blocked. It had been unusable since the day the deny rule landed.

The hard-won content stayed as it was: a troubleshooting section tied to a specific PR, a membership-exceptions rule, a geometry check with the commit SHA that introduced it. We rechecked those references against git log before keeping them.

One Tier 3 item isn't about skills at all. Personal contact details for the business owners were sitting in the web README, a file every agent session reads. We moved them to a separate document and left a pointer. It's worth checking your always-loaded files for anything you wouldn't want in a context window.

## Results

| Repo | Skills | Corpus words | Change |
|---|---|---|---|
| Web | 34 → 28 | 45,459 → 31,140 | −31% |
| iOS | 27 → 22 | 30,535 → 20,437 | −33% |
| Android | 7 → 5 | 5,847 → 3,111 | −47% |

On Android, `AGENTS.md` also dropped ~300 words with no loss of project-specific content. On iOS, some skill rewrites went further; the PR skill went from 1,894 to 901 words.

One cost was hiding in the frontmatter. iOS skill descriptions, the blurbs that load at session start before any skill is picked, added up to roughly 1,100 words. One description alone ran ~90 words. That cost is paid on every session whether or not a skill is used, and trimming a description to two sentences costs nothing.

Deleting a third to nearly half of the agent docs made the agents better, not just no worse. Every deleted fiction is a confident wrong action that can no longer happen, and every deleted paragraph of stock knowledge is context given back to the project-specific rules.

The pattern outlived the audit. Two weeks later, a routine PR in the web repo moved a section out of the README and into its own skill, unprompted. Single-source became the default rather than a cleanup task.

## Lessons Learned

- **Grep every claim in a skill against the source.** Function names, SKUs, dependencies, file paths. What fails the grep gets deleted, not softened.
- **Version tables rot fastest.** Dependency versions, SDK deadlines, tool versions. Replace them with a pointer at whatever file the build reads.
- **Always-loaded content costs something on every session.** Anything generic in `AGENTS.md` or a skill description dilutes the specific instructions the model needed.
- **A source outside the repo is no source.** If a claim's source is a file on one person's machine, no future agent can check it. Treat the claim as unverified.
