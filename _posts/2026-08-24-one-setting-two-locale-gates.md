---
layout: post
title: "One Setting, Two Locale Gates"
date: 2026-08-24 08:00:00 -0600
summary: "An in-app language picker is the single source of truth for an iOS app, its widgets, and its watch app. Two locale gates route every string through it, so users switch languages live and the app never reads the device language. Plus the String Catalog tests that keep 26 translations complete."
tags: [swift, ios, localization, i18n, architecture]
---

## The Problem

A person who reads their phone in German but wants their weather in Japanese should get Japanese in the app, in the home-screen widget, and on the watch face, from one tap in one settings row. That is the requirement: language is a per-app choice, picked once inside the app, applied everywhere text appears, switchable live without a relaunch, and never inherited from the phone.

[Hello Weather](https://helloweather.com) makes that harder than it sounds because it is not one process. The app draws text, and so do its widgets, its watch app, and its notifications. Each runs as a separate process that shares none of the app's environment.

Apple offers two obvious levers, and both are traps.

The first is `AppleLanguages`, the `UserDefaults` array iOS reads to pick a process's language. Writing to it fails the goal three ways. The value is cached at launch, so a change needs a relaunch. It persists across app updates in a way you do not control. And it is the exact key iOS's own per-app language row writes, so setting it from inside the app fights the system row instead of replacing it.

The second is raw `String(localized:)`. It resolves against `Bundle.main`, which resolves against the device language. Every string localized that way silently bypasses the in-app picker. The user picks Japanese and half the UI stays in German because it went through the wrong API.

Underneath both is the fact that "the language" is not one thing in SwiftUI. A `Text("...")` literal is localized by the environment locale. A plain `String` built for a notification or an accessibility label is localized by whatever API you call. Fix only one path and the device language leaks into a UI you thought you had translated. So the architecture question came before any translation: what is the single source of truth for the app's language, and how does every string on screen get routed through it?

## The Solution

One setting owns the language, and everything that renders text resolves against it through one of exactly two gates. The setting is a plain string in shared storage, backed by an enum:

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

Twenty-seven cases, string-backed so the raw value doubles as the locale identifier and the persisted value. The two Chinese cases carry explicit raw values because their identifiers are not bare language codes. In the app each case also carries a `displayName` autonym ("Deutsch", "日本語", "简体中文"). The picker lists each language by its name in the current app language, with the autonym as a subtitle, so a user scanning the list can find their own script.

The default is deliberately conservative. The app soft-launched, so its output stays English until a user opts in:

```swift
func languageDefault() -> String {
    // TODO: detect the device locale and return the matching Language, if supported.
    return Language.en.rawValue
}
```

Auto-adopting the device language on first launch is a product decision with support consequences. It is easy to turn on later by returning a mapped device code instead of a hard `en`. Starting conservative means no user is surprised by a language they did not pick.

### Gate One: the Environment Locale, Pinned at Every Root

SwiftUI localizes `Text("...")` literals through the `locale` in the environment. Set that environment locale to the language setting and every literal in the view tree follows it, with no wrapper and no call-site helper. The catch is that you cannot set it once. A widget is a different process that inherits nothing from the app, so the pin has to be repeated at every process root that renders text. In the app that root is the scene:

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

The widget lives in its own target and starts cold, so it re-pins the same locale inside its configuration's content closure:

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

The pin appears once in each block and reads from the same `SettingsManager.shared`. Our repo has 35 of them: the app root, every widget entry view, the watch root, and each complication view. That repetition looks like a smell until you see what it prevents. A widget that forgets the pin renders in the device language, and it looks correct in a simulator running in English, so the bug ships. The pins are the seam where the multi-process reality of an iOS app meets the one-setting intent.

### Gate Two: the `localized(_:)` Helper for String-Typed Copy

Views are only half the strings. Notifications, accessibility labels, dictionary values in a manager, and chart-unit suffixes need a real `String`, and a `String` has no environment to read. `String(localized: "…")` is banned because it resolves against the device language. The gate is a one-function helper, built on the `Language` enum above:

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

Two choices carry it. First, the parameter is a `LocalizedStringResource`, not a `String`. When you write `localized("Ranges")` the compiler sees a resource literal and pulls `"Ranges"` into the catalog exactly as it would for a `Text`. A `String` parameter would have thrown extraction away. Second, the helper reads the language from the same shared key the setting writes, defaults to English, reassigns `resource.locale`, and only then calls `String(localized:)`. That reassignment is the entire gate. The prebuilt `languageLocales` dictionary just avoids constructing a `Locale` on every call.

At the call site it is unremarkable: `localized("Ranges")`, or `localized("\(minutes) minutes")` to interpolate into a plural key. Every `String`-typed piece of user-facing copy goes through it. Between the environment pins for `Text` and this helper for `String`, there is no third path, and neither path can reach the device language.

Each property of the design maps to a rejected lever. Switching is live because the environment locale is read on every render and the helper reads the setting on every call. Change the setting and the next render is in the new language, which `AppleLanguages` cannot do while it is cached until restart. There is no fight with iOS because we never touch `AppleLanguages`, so the app's language and the system's per-app row never clobber each other. The limit is that the per-app language row iOS shows for the app in Settings is visible but inert, because nothing consults `Bundle.main`'s language. A user who changes it there sees nothing happen. Anyone copying this pattern should decide whether to explain that in-app.

## The String Catalog Fact That Bites Hardest

The two gates route every string to the right language. What renders once it arrives is the String Catalog (`Localizable.xcstrings`), and its sharpest edge is this: a key missing a row for the current language renders the raw key, not the English source value. People assume localization degrades gracefully to the source language. It does not. If the Japanese table is missing `"Air Quality"`, a Japanese user sees `Air Quality` only because that key happens to read like English. A key named `"aqi.title"` would put `aqi.title` on screen.

That makes the catalog all-or-nothing. A key is either fully translated across every shipped language or it is a latent bug waiting for one user in one language. We enforce it with a test that reads the source catalog directly, so it catches a gap before a build can repackage it:

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

A key must appear in all 26 non-English languages or in none of them. The "none" case is intentionally deferred, flag-gated copy. Any partial row set fails. The required set is the `Language` enum minus `en`, so adding a language to the enum adds it to the test. The loader keeps only the set of language codes under each key and drops the values: the test asks which languages are *present*, not what they say.

## The Build Prunes Keys It Cannot See

The nastiest hazard is a tooling one, because it deletes work silently. A command-line build can regenerate the catalog and prune keys it does not observe being referenced, taking their translations with them. We have watched a key like `"1 min"`, live in a view but referenced in a way the command-line pass did not catch, lose all 26 of its translations in one regenerated file.

The defense is two-layered. First, a mechanical check before committing the catalog after a build:

```bash
git diff -- HelloWeather/HelloWeather/Resources/Localizable.xcstrings \
  | grep -c '^-.*"value"'
```

A non-zero count means translated values are being deleted. The fix is to restore the file, not commit it. Second, a sentinel test pins the handful of easily-pruned keys so a regression fails loudly instead of shipping:

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

The keys listed have few referents in code, which is what makes them prune easily. The test fails if any of them is absent from the catalog or has lost a language.

## Results

- One source of truth: a 27-case enum persisted as a single shared-storage string, defaulting to English so the soft launch shipped in English until users opted in.
- 35 environment-locale pins across the app root, every widget entry view, the watch root, and its complications. That is the cost of localizing four separate processes instead of one.
- Live switching with no relaunch and no interaction with `AppleLanguages`. The accepted downside is that the iOS per-app language row for the app is inert, which the localization plan flags as a support-confusion risk.
- Completeness is held by test, not discipline: keys are all-or-nothing across 26 languages, and a sentinel test plus a diff gate catch build-time pruning before a commit.

## Lessons Learned

- **Any surface that starts its own process must re-pin the locale.** A forgotten pin passes in an English simulator, so every new widget or complication is an obligation to add one.
- **A localization helper takes the resource type, not a string.** The compiler then extracts literals into the catalog from the helper exactly as it does from a view.
- **Assume no per-key fallback until you have proven otherwise.** A missing row that renders the raw key turns completeness into an invariant to test on every commit.
- **Gate the diff of any file both a build and a human write.** Regenerated output can delete work silently, so check for removed values and pin the keys that keep disappearing.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the user requirement instead of the product, the "Why This Beats the Alternatives" subsection is folded into the second gate as its limit, Results is trimmed to the cost and the accepted downside, and Lessons Learned is cut from seven bullets to four. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The settings and helper excerpts now read the language from an app-group `UserDefaults` suite instead of `UserDefaults.standard` (the real store is shared so widgets and the watch can read it), `language` is a computed property over that store whose setter publishes, and the widget uses `AppIntentConfiguration` rather than `StaticConfiguration`. The two catalog tests were rewritten to match the real file: `JSONSerialization` instead of `Decodable` structs, a required-language set derived from the `Language` enum minus `en`, the real test names, the real catalog path, and the full nine-key sentinel list. The picker description now says names are shown in the app language with the autonym as a subtitle, the pin count is stated as 35 exactly, and "generates support questions" was softened to the plan's own wording, a support-confusion risk.
