---
layout: post
title: "One Setting, Two Locale Gates"
date: 2026-08-24 08:00:00 -0600
summary: "One in-app language picker sets the language for an iOS app, its widgets, and its watch app. Every string gets its language from that setting through one of two gates, so users can switch languages without relaunching and the app never reads the phone's language. Plus the String Catalog tests that keep 26 translations complete."
tags: [swift, ios, localization, i18n, architecture]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
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
