---
layout: post
title: "One Setting, Two Locale Gates"
date: 2026-08-24 08:00:00 -0600
summary: "One in-app language picker sets the language for an iOS app, its widgets, and its watch app. Every string gets its language from that setting through one of two gates, so users can switch languages without relaunching and the app never reads the phone's language. Plus the String Catalog tests that keep 26 translations complete."
tags: [swift, ios, localization, i18n, architecture]
---

## The Problem

Someone who reads their phone in German but wants their weather in Japanese should get Japanese in the app, in the home-screen widget, and on the watch face, from one tap in one settings row. That's the requirement. The language is chosen once inside the app, it applies everywhere text appears, it changes without a relaunch, and it never comes from the phone's language.

[Hello Weather](https://helloweather.com) makes that harder than it sounds because it isn't one program. The app draws text, and so do its widgets, its watch app, and its notifications. Each one runs as its own process, and nothing the app sets up at launch carries over to the others.

Apple offers two obvious ways to do this, and both fail.

The first is `AppleLanguages`, an array in `UserDefaults` that iOS reads to pick a process's language. Writing to it fails the goal three ways. iOS reads the value at launch and caches it, so a change needs a relaunch. It persists across app updates in a way you don't control. And it's the same key that iOS's own per-app language row writes, so setting it from inside the app fights the system row instead of replacing it.

The second is plain `String(localized:)`. It looks strings up in `Bundle.main`, which uses the device language. Every string localized that way skips the in-app picker. The user picks Japanese and half the UI stays in German because it went through the wrong API.

Under both is the fact that "the language" isn't one thing in SwiftUI. A `Text("...")` literal gets its language from the locale in the environment. A plain `String` built for a notification or an accessibility label gets its language from whichever API you call. Fix one path and the device language leaks in through the other. So before any translation, we had an architecture question: where does the app's language live, and how does every string on screen get its language from there?

## The Solution

One setting owns the language, and everything that renders text reads it through one of two paths, which we call gates. The setting is a plain string in shared storage, backed by an enum:

```swift
import Foundation

enum Language: String, CaseIterable, Identifiable {
    case cs, da, de, el, en, es, fi, fr, hi, hu, id, it
    case ja, ko, nb, nl, pl, pt, ro, ru, sv, th, tr, uk, vi
    case zhHans = "zh-Hans"
    case zhHant = "zh-Hant"

    var id: String { rawValue }
    var locale: Locale { Locale(identifier: rawValue) }
}
```

Twenty-seven cases, backed by strings, so the raw value is both the locale identifier and the value we store. The two Chinese cases carry explicit raw values because their identifiers aren't bare language codes. In the app each case also carries a `displayName`, the language's native name ("Deutsch", "日本語", "简体中文"). The picker lists each language by its name in the current app language, with the native name as a subtitle, so a user scanning the list can find their own script.

The default is English on purpose. The app soft-launched, so it stays in English until a user opts in:

```swift
func languageDefault() -> String {
    // TODO: detect the device locale and return the matching Language, if supported.
    return Language.en.rawValue
}
```

Adopting the device language on first launch is a product decision with support consequences. It's easy to turn on later by returning the matching device code instead of a hard `en`. Starting with English means nobody gets a language they didn't pick.

### Gate One: the Environment Locale, Pinned at Every Root

SwiftUI localizes `Text("...")` literals using the `locale` in the environment. Set that locale from the language setting and every literal in the view tree follows it, with no wrapper and no helper at the call site. The catch is that you can't set it once. A widget is a different process and inherits nothing from the app, so the locale has to be set again, which we call pinning, at the root of every process that renders text. In the app that root is the scene:

```swift
import SwiftUI

// An app-group suite, so the widget and watch processes read the same value.
let sharedStore = UserDefaults(suiteName: "group.example.weather")!

final class SettingsManager: ObservableObject {
    static let shared = SettingsManager()

    var language: String {
        get { sharedStore.string(forKey: "language") ?? languageDefault() }
        set {
            sharedStore.set(newValue, forKey: "language")
            objectWillChange.send()
        }
    }

    var languageLocale: Locale {
        Language(rawValue: language)?.locale ?? Locale(identifier: "en")
    }
}

@main
struct WeatherApp: App {
    @ObservedObject private var settingsManager = SettingsManager.shared

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.locale, settingsManager.languageLocale)
        }
    }
}
```

The widget is its own target and starts cold, so it pins the same locale again inside its configuration's content closure:

```swift
import SwiftUI
import WidgetKit

struct SmallCurrentWidget: Widget {
    let kind = "SmallCurrent"

    var body: some WidgetConfiguration {
        AppIntentConfiguration(kind: kind, intent: Configuration.self, provider: Provider()) { entry in
            SmallCurrentView(configuration: entry.configuration, viewModel: entry)
                .environment(\.locale, SettingsManager.shared.languageLocale)
        }
        .configurationDisplayName("Small Current")
        .supportedFamilies([.systemSmall])
    }
}
```

The pin is one line in each block, and each one reads from the same `SettingsManager.shared`. Our repo has 35 of them: the app root, every widget entry view, the watch root, and each complication view. That repetition looks like a smell until you see what it prevents. A widget that forgets the pin renders in the device language. It looks right in a simulator running in English, so the bug ships.

### Gate Two: the `localized(_:)` Helper for String-Typed Copy

Views are only half the strings. Notifications, accessibility labels, dictionary values in a manager, and chart-unit suffixes need a real `String`, and a `String` has no environment to read. We don't allow `String(localized: "…")` because it uses the device language. Instead, every one of those strings goes through one helper, built on the `Language` enum above:

```swift
private let languageLocales: [String: Locale] = Dictionary(
    uniqueKeysWithValues: Language.allCases.map { ($0.rawValue, Locale(identifier: $0.rawValue)) }
)

// Resolves in the app language, not the device language.
func localized(_ resource: LocalizedStringResource) -> String {
    var resource = resource
    let language = sharedStore.string(forKey: "language") ?? Language.en.rawValue
    resource.locale = languageLocales[language] ?? Locale(identifier: language)
    return String(localized: resource)
}
```

Two choices make it work. First, the parameter is a `LocalizedStringResource`, not a `String`. When you write `localized("Ranges")` the compiler sees a resource literal and pulls `"Ranges"` into the catalog, the same as it would for a `Text`. With a `String` parameter the compiler would extract nothing. Second, the helper reads the language from the same shared key the setting writes, defaults to English, sets `resource.locale`, and only then calls `String(localized:)`. That one assignment is the gate. The `languageLocales` dictionary is built once so we don't construct a `Locale` on every call.

At the call site it looks ordinary: `localized("Ranges")`, or `localized("\(minutes) minutes")` to fill in a plural key. Every `String` a user can see goes through it. Between the environment pins for `Text` and this helper for `String`, there's no third path, and neither path can reach the device language.

Each thing we wanted maps to one of the levers we rejected. Switching is live because the environment locale is read on every render and the helper reads the setting on every call. Change the setting and the next render is in the new language. `AppleLanguages` can't do that, because it's cached until restart. There's no fight with iOS because we never touch `AppleLanguages`, so the app's language and the system's per-app row never overwrite each other. The limit is that the per-app language row iOS shows for the app in Settings is still there but does nothing, because nothing in the app reads `Bundle.main`'s language. A user who changes it there sees nothing happen. Anyone copying this pattern should decide whether to explain that in the app.

## The String Catalog Fact That Bites Hardest

The two gates send every string to the right language. What shows up on screen comes from the String Catalog (`Localizable.xcstrings`), and its sharpest edge is this: a key with no row for the current language renders the raw key, not the English source value. People assume localization falls back to the source language. It doesn't. If the Japanese table is missing `"Air Quality"`, a Japanese user sees `Air Quality` only because that key happens to read like English. A key named `"aqi.title"` would put `aqi.title` on screen.

That makes the catalog all-or-nothing. A key is either translated in every shipped language or it's a bug waiting for one user in one language. We enforce that with a test that reads the catalog file directly, so it catches a gap before a build can repackage it:

```swift
import Testing
import Foundation
@testable import HelloWeather

@Suite("String catalog completeness")
struct XcstringsCompletenessTests {
    // Every Language case except the source language.
    private static let requiredLanguages = Set(
        Language.allCases.map(\.rawValue)
    ).subtracting(["en"])

    // Key -> the set of language codes that have a row. Values are discarded.
    private func loadCatalog(_ name: String) throws -> [String: Set<String>] {
        let url = URL(fileURLWithPath: #filePath)
            .deletingLastPathComponent()
            .deletingLastPathComponent()
            .appendingPathComponent("HelloWeather/HelloWeather/Resources/\(name).xcstrings")
        let data = try Data(contentsOf: url)
        let json = try #require(try JSONSerialization.jsonObject(with: data) as? [String: Any])
        let strings = try #require(json["strings"] as? [String: Any])

        return strings.mapValues { entry in
            let localizations = (entry as? [String: Any])?["localizations"] as? [String: Any] ?? [:]
            return Set(localizations.keys)
        }
    }

    @Test("Localizable keys are fully translated or fully untranslated")
    func localizableKeysAreAllOrNothing() throws {
        let catalog = try loadCatalog("Localizable")

        for (key, languages) in catalog {
            let missing = Self.requiredLanguages.subtracting(languages)
            #expect(missing.isEmpty || missing == Self.requiredLanguages,
                    "\"\(key)\" is partially translated; missing: \(missing.sorted())")
        }
    }
}
```

A key must appear in all 26 non-English languages or in none of them. The "none" case is for copy we've deliberately held back behind a flag. Any partial set fails. The required set is the `Language` enum minus `en`, so adding a language to the enum adds it to the test. The loader keeps only the set of language codes under each key and drops the values, because the test asks which languages are *present*, not what they say.

## The Build Prunes Keys It Cannot See

The nastiest hazard is in the tooling, because it deletes work silently. A command-line build can regenerate the catalog and remove any key it doesn't see referenced, along with that key's translations. We've watched a key like `"1 min"`, used in a view but referenced in a way the command-line pass didn't catch, lose all 26 of its translations in one regenerated file.

The defense has two layers. First, a mechanical check before committing the catalog after a build:

```bash
git diff -- HelloWeather/HelloWeather/Resources/Localizable.xcstrings \
  | grep -c '^-.*"value"'
```

A non-zero count means translated values are being deleted. The fix is to restore the file, not commit it. Second, a test checks a fixed list of the keys that prune easily, so a regression fails loudly instead of shipping:

```swift
@Test("Keys with few referents stay translated across catalog regeneration")
func sentinelKeysStayTranslated() throws {
    let catalog = try loadCatalog("Localizable")

    let sentinels = [
        "1 min", "5 min",
        "Rename", "Custom Name", "Favorite", "Unfavorite",
        "Customize locations",
        "Tap a location's name to rename it, or tap a star to fave. Drag to reorder.",
        "Reset Locations Tip",
    ]

    for key in sentinels {
        let languages = try #require(catalog[key], "\"\(key)\" pruned from the catalog")
        #expect(Self.requiredLanguages.subtracting(languages).isEmpty)
    }
}
```

The keys in the list have few references in code, which is why they prune easily. The test fails if any of them is missing from the catalog or has lost a language.

## Results

- One source of truth: a 27-case enum stored as a single string in shared storage, defaulting to English so the soft launch shipped in English until users opted in.
- 35 environment-locale pins across the app root, every widget entry view, the watch root, and its complications. That's the cost of localizing four separate processes instead of one.
- Live switching with no relaunch and no use of `AppleLanguages`. The downside we accepted is that the iOS per-app language row for the app does nothing, which the localization plan flags as a support-confusion risk.
- Completeness is held by tests, not discipline: keys are all-or-nothing across 26 languages, and the fixed-list test plus the diff check catch build-time pruning before a commit.

## Lessons Learned

- **Anything that runs as its own process must pin the locale again.** A forgotten pin passes in an English simulator, so every new widget or complication means one more pin to add.
- **A localization helper takes the resource type, not a string.** The compiler then extracts literals into the catalog from the helper the same way it does from a view.
- **Assume there's no per-key fallback until you've proven otherwise.** A missing row that renders the raw key means completeness has to be tested on every commit.
- **Check the diff of any file that both a build and a person write.** Regenerated output can delete work silently, so look for removed values and keep a test on the keys that keep disappearing.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the user requirement instead of the product, the "Why This Beats the Alternatives" subsection is folded into the second gate as its limit, Results is trimmed to the cost and the accepted downside, and Lessons Learned is cut from seven bullets to four. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The settings and helper excerpts now read the language from an app-group `UserDefaults` suite instead of `UserDefaults.standard` (the real store is shared so widgets and the watch can read it), `language` is a computed property over that store whose setter publishes, and the widget uses `AppIntentConfiguration` rather than `StaticConfiguration`. The two catalog tests were rewritten to match the real file: `JSONSerialization` instead of `Decodable` structs, a required-language set derived from the `Language` enum minus `en`, the real test names, the real catalog path, and the full nine-key sentinel list. The picker description now says names are shown in the app language with the autonym as a subtitle, the pin count is stated as 35 exactly, and "generates support questions" was softened to the plan's own wording, a support-confusion risk.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 3, run after batch 2 (#69) merged. Every paragraph was redrafted from an ELI5 of the post: "autonym" became "native name", "pin" is defined where it first appears, "sentinel" in the prose became "a fixed list of keys", the closing line about pins being "the seam" was cut as a flourish, "surface" in the first lesson became "anything that runs as its own process", and the rest is contractions, "we", and subject-first sentences. Code blocks, headings, numbers, quoted text, and links are unchanged. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"

**Prompt 9:** "usage looks fine, merge it and run batch 2"

**Prompt 10:** "usage is fine, please continue -- one more thing -- at the end (or perhaps with future batches?) I'd like to change the "How This Post Was Made" sections in all posts to not have the prompt in the post itself, rather, the prompts should be moved into PR body if editable, or comments, then the "How This Post Was Made" can have the last edit date and a link to the Pull Requests / Prompts -- then there's less cruft at the end for readers that just want to copy paste a post into their agent -- wdyt?"
