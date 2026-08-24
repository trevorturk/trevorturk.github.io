---
layout: post
title: "One Setting, Two Locale Gates: Localizing an iOS App Without Touching AppleLanguages"
date: 2026-08-24 08:00:00 -0600
summary: "An in-app language picker as the single source of truth, resolved through two locale gates, so users switch languages live without relaunching and without the app ever reading the device language."
tags: [swift, ios, localization, i18n, architecture]
---

## The Problem

[Hello Weather](https://helloweather.com) is not one process. The app draws text, and so do its widgets, its watch app, and its notifications — and each of those runs as a separate process that does not share the app's environment. That matters the moment you decide language should be a per-app choice: something a user picks once inside the app, applied everywhere text appears, switchable live without a relaunch, and never inherited from the phone's system language. A person who reads their phone in German but wants their weather in Japanese should get Japanese in the app, in the home-screen widget, and on the watch face, from one tap in one settings row.

Apple gives you two obvious levers for that, and both are traps.

The first is `AppleLanguages`, the `UserDefaults` array iOS reads to pick a process's language. Writing to it fails the goal in all three ways: the value is cached at launch, so a change needs a relaunch to take effect; it persists across app updates in a way you do not control; and it is the exact key iOS's own per-app language row writes, so setting it from inside the app fights the system row instead of replacing it.

The second is raw `String(localized:)`. It resolves against `Bundle.main`, which resolves against the device language. Every string localized that way silently bypasses your in-app picker and follows the phone. The user picks Japanese and half the UI stays in German because it happened to go through the wrong API.

Underneath both is the fact that "the language" is not one thing in SwiftUI. A `Text("...")` literal is localized by the environment locale. A plain `String` you build for a notification or an accessibility label is localized by whatever API you call. Two resolution paths, and fixing only one leaks the device language into a UI you thought you had translated. So the architecture question came before any translation did: what is the single source of truth for "what language is this app in," and how does every string on screen get routed through it?

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

Twenty-seven cases, string-backed so the raw value doubles as a locale identifier and the value you persist. The two Chinese cases carry explicit raw values because their identifiers are not bare language codes. In the app each case also carries a `displayName` autonym ("Deutsch", "日本語", "简体中文") so the picker shows every language in its own script, which is what a user scanning the list actually looks for.

The default is deliberately conservative. The app soft-launched, so its default output stays English until a user opts in:

```swift
func languageDefault() -> String {
    // Detect the device locale and map it here to auto-adopt on first launch.
    return Language.en.rawValue
}
```

Auto-adopting the device language on first launch is a product decision with support consequences, and it is easy to turn on later by returning a mapped device code instead of a hard `en`. Starting conservative means no user is surprised by a language they did not pick. Now the two gates.

### Gate One: the Environment Locale, Pinned at Every Root

SwiftUI localizes `Text("...")` literals through the `locale` in the environment. Set that environment locale to the language setting and every literal in the view tree follows it for free — no wrapper, no call-site helper. The catch is that you cannot set it once, because a widget is a different process that inherits nothing from the app. So the pin has to be repeated at every process root that renders text. In the app that root is the scene:

```swift
import SwiftUI

@MainActor
final class SettingsManager: ObservableObject {
    static let shared = SettingsManager()
    @Published var language = UserDefaults.standard.string(forKey: "language") ?? "en"
    var languageLocale: Locale { Locale(identifier: language) }
}

@main
struct WeatherApp: App {
    @ObservedObject private var settings = SettingsManager.shared

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.locale, settings.languageLocale)
        }
    }
}
```

The widget lives in its own target and starts cold, so it re-pins the same locale inside its configuration's content closure. Nothing about the app root reaches it:

```swift
import SwiftUI
import WidgetKit

struct CurrentConditionsWidget: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: "current", provider: Provider()) { entry in
            WidgetEntryView(entry: entry)
                .environment(\.locale, SettingsManager.shared.languageLocale)
        }
    }
}
```

Notice the pin appears once in each block and reads from the same `SettingsManager.shared`. In our repo that comes to roughly 35 such pins: the app root, every widget entry view, the watch root, and each complication view. That repetition looks like a smell until you see what it prevents. A widget that forgets the pin renders in the device language — and it looks correct in a simulator running in English, so the bug ships. Any surface that starts its own process must re-pin the environment locale, or it silently falls back to the phone. The pins are the seam where the multi-process reality of a modern iOS app meets your one-setting intent.

### Gate Two: the `localized(_:)` Helper for String-Typed Copy

Views are only half the strings. Notifications, accessibility labels, dictionary values in a manager, chart-unit suffixes: those need a real `String`, and a `String` has no environment to read. The banned move is `String(localized: "…")`, because it resolves against the device language. The gate is a one-function helper, built on the `Language` enum above:

```swift
private let languageLocales: [String: Locale] = Dictionary(
    uniqueKeysWithValues: Language.allCases.map { ($0.rawValue, $0.locale) }
)

func localized(_ resource: LocalizedStringResource) -> String {
    var resource = resource
    let language = UserDefaults.standard.string(forKey: "language") ?? Language.en.rawValue
    resource.locale = languageLocales[language] ?? Locale(identifier: language)
    return String(localized: resource)
}
```

Two choices carry the whole thing. First, the parameter is a `LocalizedStringResource`, not a `String`. That is what keeps literals extractable: when you write `localized("Ranges")` the compiler sees a resource literal and the extraction machinery pulls `"Ranges"` into the catalog exactly as it would for a `Text`. Device-independent resolution *and* automatic extraction from one call; a `String` parameter would have thrown extraction away. Second, it reads the language from the same shared key the setting writes, defaults to English, reassigns `resource.locale`, and only then calls `String(localized:)`. That single reassignment is the entire gate; the prebuilt `languageLocales` dictionary just avoids constructing a `Locale` on every call.

At the call site it is unremarkable, which is the point — `localized("Ranges")`, or `localized("\(minutes) minutes")` to interpolate into a plural key. Every `String`-typed piece of user-facing copy goes through it. Between the environment pins for `Text` and this helper for `String`, there is no third path, and neither path can reach the device language.

### Why This Beats the Alternatives

Each property maps to a rejected lever. Live switching with no relaunch, because the environment locale is read on every render and the helper reads the setting on every call — change the setting and the next render is in the new language, which `AppleLanguages` cannot do while it is cached until restart. No fight with iOS, because we never touch `AppleLanguages`, so the app's language and the system's per-app row never clobber each other. And the picker stays the truth, because nothing resolves against the device language.

There is one honest downside worth stating, because it generates support questions: since the app owns its language internally, the per-app language row iOS shows for the app in Settings is visible but inert. A user who changes it there sees nothing happen. We accepted that as the price of a picker that behaves, but anyone copying this pattern should decide whether to explain it in-app.

## The String Catalog Fact That Bites Hardest

The two gates route every string to the right language. What renders once it arrives is the String Catalog (`Localizable.xcstrings`), and its sharpest edge is not obvious until it draws blood: a key missing a row for the current language renders the raw key, not the English source value. People assume localization degrades gracefully to the source language. It does not. If the Japanese table is missing `"Air Quality"`, a Japanese user sees `Air Quality` only because that key happens to read like English — a key named `"aqi.title"` would put `aqi.title` on screen.

That makes the catalog all-or-nothing: a key is either fully translated across every shipped language or it is a latent bug waiting for one user in one language. We enforce it with a test that reads the source catalog directly, so it catches a gap before a build can repackage it:

```swift
import Testing
import Foundation

struct StringCatalog: Decodable {
    struct Entry: Decodable { let localizations: [String: Localization]? }
    struct Localization: Decodable {}   // presence is all we check
    let strings: [String: Entry]
}

@Suite("String catalog completeness")
struct StringCatalogTests {
    static let required: Set<String> = [
        "cs", "da", "de", "el", "es", "fi", "fr", "hi", "hu", "id", "it",
        "ja", "ko", "nb", "nl", "pl", "pt", "ro", "ru", "sv", "th", "tr",
        "uk", "vi", "zh-Hans", "zh-Hant",
    ]

    @Test("Every key is fully translated or fully untranslated")
    func keysAreAllOrNothing() throws {
        let url = URL(fileURLWithPath: #filePath)
            .deletingLastPathComponent()
            .appendingPathComponent("Localizable.xcstrings")
        let catalog = try JSONDecoder().decode(StringCatalog.self, from: Data(contentsOf: url))

        for (key, entry) in catalog.strings {
            let present = entry.localizations.map { Set($0.keys) } ?? []
            let missing = Self.required.subtracting(present)
            #expect(missing.isEmpty || missing == Self.required,
                    "\"\(key)\" is partially translated; missing: \(missing.sorted())")
        }
    }
}
```

A key must appear in all 26 non-English languages or in none of them — the "none" case is intentionally-deferred, flag-gated copy. Any partial row set fails. The `Localization` shape is empty on purpose: we are only asking which languages are *present*, not reading their values, so the decoder walks the JSON and stops. If a localization system has no per-key fallback, completeness is not a release-time check; it is an invariant a test holds on every commit.

## The Build Prunes Keys It Cannot See

The nastiest hazard is a tooling one, because it deletes work silently. A command-line build can regenerate the catalog and prune keys it does not observe being referenced, taking their translations with them. We have watched a key like `"1 min"` — live in a view, but referenced in a way the command-line pass did not catch — lose all 26 of its translations in one regenerated file.

The defense is two-layered. First, a mechanical check before committing the catalog after a build:

```bash
git diff -- HelloWeather/HelloWeather/Resources/Localizable.xcstrings \
  | grep -c '^-.*"value"'
```

A non-zero count means translated values are being deleted; the fix is to restore the file, not commit it. Second, a sentinel test that pins the handful of easily-pruned keys so a regression fails loudly instead of shipping:

```swift
@Test("Fragile keys survive catalog regeneration")
func sentinelKeysStayTranslated() throws {
    let catalog = try JSONDecoder()
        .decode(StringCatalog.self, from: Data(contentsOf: catalogURL))

    for key in ["1 min", "5 min", "Rename", "Custom Name"] {
        let entry = try #require(catalog.strings[key], "\"\(key)\" pruned from the catalog")
        let present = entry.localizations.map { Set($0.keys) } ?? []
        #expect(StringCatalogTests.required.subtracting(present).isEmpty)
    }
}
```

Treat the catalog as generated and hand-edited at once: a build can rewrite it, so gate the diff and pin the fragile keys.

## Results

- **One source of truth**: a 27-case enum persisted as a single shared-storage string, defaulting to English so the soft launch shipped in English until users opted in.
- **Two locale gates, zero device-language paths.** `Text` follows the pinned environment locale; every `String`-typed value goes through `localized(_:)`. Neither reads `Bundle.main`, so no surface can leak the phone's language.
- **Roughly 35 environment-locale pins** across the app root, every widget entry view, the watch root, and its complications — the cost of localizing four separate processes correctly instead of one.
- **Live switching with no relaunch**, and no interaction with `AppleLanguages` or the iOS per-app language row.
- **Completeness held by test, not discipline**: keys are all-or-nothing across 26 languages, with a sentinel test and a diff gate that catch build-time pruning before a commit.

## Lessons Learned

- **Name the single source of truth before you translate a word**, because the hard part of app localization is deciding what "the language" is and routing every string through it — not the translations themselves.

- **There are two locale gates, not one, and they are easy to conflate.** `Text` literals resolve through the environment locale while `String`-typed copy resolves through whatever API you call, so fixing only the first leaves notifications and accessibility labels leaking the device language.

- **Repeated environment pins are a feature of multi-process apps, not a smell**, because widgets and the watch start cold and inherit nothing — a forgotten pin fails silently in an English simulator, so treat each new widget or complication as an obligation to add one.

- **Reject `AppleLanguages` and raw `String(localized:)` on purpose**, because the first is process-cached, update-persistent, and clobbers the system row, and the second follows the device language — both look like the easy path and both undo the single source of truth.

- **Assume no per-key fallback until you have proven otherwise**, because many systems render the raw key rather than the source language for a missing row, which turns completeness into an invariant to test on every commit.

- **Your catalog is generated and hand-edited at the same time**, because a build can prune keys it does not see referenced and delete their translations — so gate the diff for removed values and pin the fragile keys.

- **Every honest architecture has a downside worth documenting**: owning the language internally makes the iOS per-app language row inert, which is a real support-confusion vector to explain rather than hide.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.
