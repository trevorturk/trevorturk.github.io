---
layout: post
title: "A Date-Format Rulebook: Intents, Per-Language Classifiers, and a Ban on Post-Processing"
date: 2026-08-24 08:10:00 -0600
summary: "Why one weather app routes every date and time string through a single enum of UI intents and one exhaustive switch, keeps per-language quirks in named classifiers instead of scattered language lists, and treats a lowercase 'pm' as a construction decision rather than a string hack."
tags: [swift, ios, localization, i18n, dates]
---

## The Problem

[Hello Weather](https://helloweather.com) has far more text than a typical weather app, and the text is part of the layout rather than decoration on top of it. Dates alone render on 24 distinct surfaces — a sunrise time under an icon, a weekday rail under the hourly chart, a watch complication with room for three characters — in 27 languages, with a user-selectable 12- or 24-hour clock. One person maintains it, with a model doing the translating and no agency behind it, so there is no room for date code that only one screen understands.

Date formatting rots quietly under that load. One screen writes `"EEE"`, another writes `"ha"` and lowercases it because the design wants "5pm" and not "5 PM", a third copies the second — and now the formatters disagree on capitalization and meridiem, with the knowledge of what each surface *should* look like smeared across dozens of call sites.

That lowercasing habit is the specific trap: `formatter.string(from: date).lowercased()` gives you "5pm" in English and quietly breaks everywhere else. Hungarian's meridiem, stripped and lowercased, becomes "de" — the word for "but". Vietnamese headers need day-first order or the numerals run together ambiguously. Japanese, Korean, and Chinese do not use a meridiem for short hours at all, and their dates want a template-driven year-month-day shape no Western pattern produces. When a translator reports one of these, you do not have one switch to fix — you have a search-and-replace and a hope that you found them all.

## The Solution

The governing idea is that **date formatting is declared configuration, not imperative code.** Every date string in the app, watch, and widgets flows through three stages: an *intent enum* whose cases name UI surfaces (`sunEventTime`, `dailyHeader`), not formats; one exhaustive *`plan(for:)` switch* mapping an intent plus a language to a render plan; and *named classifiers* on the language enum that hold per-language quirks, each a `switch` over every language rather than a language list buried in an intent arm.

The call site is deliberately dumb. `Text(sunset.withFormat(.sunEventTime))` carries no format string, no lowercasing, no locale plumbing; `withFormat` looks up the current language and time zone, asks the intent for its plan, and renders.

### One enum of intents, one exhaustive plan

Start here for auditability: a reviewer fixing how sun times render in Finnish should have exactly one place to look, and adding a dated screen should be impossible without deciding how it reads in every language. An exhaustive switch with no `default` delivers both. The block below is the whole shape in miniature — plan struct, a slice of the intent enum, three classifiers, and the switch that ties them together — and compiles as written.

```swift
import Foundation

enum Language: String, CaseIterable {
    case en, de, fr, hu, ja, vi
    // one case per supported language; ~two dozen more elided
}

enum Meridiem {
    case localeDefault   // keep CLDR's own am/pm symbols
    case plainLower      // bare lowercase: "5:39pm"
}

extension Language {
    enum DateOrder { case western, cjk }
    enum HeaderOrder { case monthDay, dayMonth }

    // Most languages want a bare lowercase meridiem. Three keep the punctuated
    // native form because the stripped version collides with a real word:
    // Hungarian "de" is the word for "but".
    var standardMeridiem: Meridiem {
        switch self {
        case .hu:                       return .localeDefault
        case .en, .de, .fr, .ja, .vi:   return .plainLower
        }
    }

    // CJK renders native 年月日 via ICU templates; Western uses explicit patterns.
    var dateOrder: DateOrder {
        switch self {
        case .ja:                       return .cjk
        case .en, .de, .fr, .hu, .vi:   return .western
        }
    }

    // Exactly one language orders headers day-first: Vietnamese, where
    // month-day order runs its numerals into an ambiguous string.
    var headerOrder: HeaderOrder {
        switch self {
        case .vi:                       return .dayMonth
        case .en, .de, .fr, .hu, .ja:   return .monthDay
        }
    }
}

struct DateRenderPlan {
    enum Kind {
        case pattern(String)    // an explicit, ordered format
        case template(String)   // an ICU skeleton; ICU picks the order
    }

    let kind: Kind
    let twentyFourHourVariant: String?
    let capitalized: Bool
    let meridiem: Meridiem

    static func pattern(_ p: String, capitalized: Bool) -> DateRenderPlan {
        DateRenderPlan(kind: .pattern(p), twentyFourHourVariant: nil,
                       capitalized: capitalized, meridiem: .localeDefault)
    }
    static func time(_ p: String, or h24: String, meridiem: Meridiem) -> DateRenderPlan {
        DateRenderPlan(kind: .pattern(p), twentyFourHourVariant: h24,
                       capitalized: false, meridiem: meridiem)
    }
    static func template(_ t: String) -> DateRenderPlan {
        DateRenderPlan(kind: .template(t), twentyFourHourVariant: nil,
                       capitalized: false, meridiem: .localeDefault)
    }
}

enum DateFormatIntent: String, CaseIterable {
    case summaryWeekday, dailyHeader, sunEventTime, forecastUpdatedDateTime
    // 24 cases total, one per UI surface; the rest elided

    // Exhaustive, no default: a new case does not compile until it has an arm.
    func plan(for language: Language) -> DateRenderPlan {
        switch self {
        case .summaryWeekday:
            return .pattern("E", capitalized: false)
        case .dailyHeader:
            switch language.dateOrder {
            case .cjk: return .template("MMMdE")
            case .western:
                switch language.headerOrder {
                case .monthDay: return .pattern("E, MMM d", capitalized: true)
                case .dayMonth: return .pattern("E, d MMM", capitalized: true)
                }
            }
        case .sunEventTime:
            return .time("h:mma", or: "H:mm", meridiem: language.standardMeridiem)
        case .forecastUpdatedDateTime:
            return .time("E, MMM d @ h:mma", or: "E, MMM d @ H:mm",
                         meridiem: language.standardMeridiem)
        }
    }
}
```

Notice what the intent arms do *not* contain: no list of languages. `dailyHeader` asks `language.dateOrder` and `language.headerOrder` and branches on the answers — the intent knows the shape of a header, the classifier knows which languages deviate. And because `plan(for:)` has no `default`, adding a case fails to compile until you write its arm, so a dated surface cannot ship without an explicit decision for every language group.

### Per-language quirks live in named classifiers, not intent arms

The moment a `switch self { case .fi, .hu: … }` appears *inside* a formatting rule, the reason for the deviation is lost, and the next language that needs the same treatment gets added somewhere else. A named classifier keeps the quirk and its reason together, and every deviation in the app becomes one arm you can point at.

Each one exists because a real language broke a naive rule. `standardMeridiem` records that Hungarian keeps its punctuated form — "keep the period, bare 'de' is the Hungarian word for 'but'" — a note you would not think to write from first principles. `dateOrder` is why CJK languages get ICU templates instead of a hardcoded Western sequence, and why their short hours render as "9時"/"9时"/"9시" where no meridiem applies rather than "9pm". `headerOrder` is a targeted arm for Vietnamese rather than a vague "some languages are day-first" category that would sweep in languages that do not want it: when one language breaks a pattern, give it its own arm and write down why.

### "5:39pm" is symbol assignment, not string surgery

The house voice wants "5:39pm" — lowercase, no space, no periods — produced *correctly*, not by lowercasing output. Lowercasing the rendered string is wrong twice: it also lowercases any weekday or month name that should stay capitalized, and it applies ASCII casing rules unless you pass a locale, which mangles languages like Turkish. The place to decide the meridiem is when the formatter is built, by shaping its AM/PM symbols before it renders. This function does that, and runs as written.

```swift
import Foundation

enum Meridiem { case localeDefault, plainLower }

/// Shapes AM/PM symbols at construction so the formatter emits the house voice
/// directly. Never a `.lowercased()` on the formatter's output.
func applyMeridiem(_ meridiem: Meridiem, to formatter: DateFormatter,
                   dateFormat: String, locale: Locale) {
    guard dateFormat.contains("a") else { return }   // no meridiem to shape
    switch meridiem {
    case .localeDefault:
        return
    case .plainLower:
        let bareLowercase: (String) -> String = { symbol in
            let letters = symbol.unicodeScalars
                .filter { CharacterSet.alphanumerics.contains($0) }
                .map(Character.init)
            return String(letters).lowercased(with: locale)
        }
        if let am = formatter.amSymbol { formatter.amSymbol = bareLowercase(am) }
        if let pm = formatter.pmSymbol { formatter.pmSymbol = bareLowercase(pm) }
    }
}
```

It strips non-alphanumeric scalars from the CLDR symbol and lowercases with the locale's own casing rules: "5:39 p. m." becomes "5:39pm", and Turkish "ÖS" becomes "ös" using Turkish casing, not ASCII. The meridiem is correct by construction, so nothing downstream touches the string, and the guard on the `a` pattern character means formatters with no meridiem skip the work. When a call site genuinely needs a different case, it passes a declared `case:` parameter the formatting layer applies once, locale-aware; production paths never do, because the voice they want is already in the symbols.

### Bans the tests enforce by reading the source

"Never lowercase a rendered date" only holds if it cannot be quietly reintroduced, and the compiler cannot see a `.lowercased()` habit the way it sees a missing switch arm. So a test reads every Swift file in the app, runs a regex over it, and fails with the offending file and line. This is the whole check, under Swift Testing, as written.

```swift
import Testing
import Foundation

@Suite struct DateFormatCallsiteTests {
    static let sourceRoot = URL(fileURLWithPath: #filePath)
        .deletingLastPathComponent()
        .deletingLastPathComponent()
        .appendingPathComponent("App")   // your source tree

    static func swiftSources() -> [(url: URL, text: String)] {
        guard let files = FileManager.default.enumerator(
            at: sourceRoot, includingPropertiesForKeys: nil) else { return [] }
        return files.compactMap { item in
            guard let url = item as? URL, url.pathExtension == "swift",
                  let text = try? String(contentsOf: url, encoding: .utf8)
            else { return nil }
            return (url, text)
        }
    }

    @Test func caseMethodsNeverChainOnFormattedDates() throws {
        let sources = Self.swiftSources()
        try #require(sources.count > 100, "source scan found too few files")

        let chain = try NSRegularExpression(pattern:
            #"withFormat\([^()]*\)\.(?:lowercased|uppercased|capitalized)"#)
        var offenders: [String] = []
        for (url, text) in sources {
            let range = NSRange(text.startIndex..<text.endIndex, in: text)
            for match in chain.matches(in: text, range: range) {
                if let r = Range(match.range, in: text) {
                    offenders.append("\(url.lastPathComponent): \(text[r])")
                }
            }
        }
        #expect(offenders.isEmpty,
            "case methods chained on rendered dates: \(offenders). Use "
            + "withFormat's `case:` parameter so the transform runs once, locale-aware.")
    }
}
```

Two sibling tests round out the bans. One fails if any intent case has no call site: an intent with no UI is either dead code or a wiring bug, and either way the test forces the question. The other freezes the small set of raw format strings the frozen alternate-style catalog may still use, so new UI cannot smuggle in a raw `dateFormat` — it must add an intent. And two further exhaustive switches classify every intent by width budget and by worst-case sample, so adding one case fails three separate compiles until it is classified three ways. There is no path to a half-specified date format.

### Hand-verified English anchors the machines never touch

The rendering of every intent in every language is captured in committed golden files (a [companion post](/golden-files-per-language/) covers that matrix and the fast macOS test tier), machine-recorded and machine-updated — which is a hazard, because a careless record run could bless a regression into the English golden and the diff would look like every other regeneration. So a handful of English outputs are also pinned by hand, in a table a human edits and a record run never rewrites.

```swift
@Test func englishAnchorsHoldAtTheEveningInstant() {
    // A hand-picked instant (2026-08-15, 17:39 local), not a machine sample.
    // withFormat / withAppLanguage / DateFormatSamples are the test harness.
    let anchors: [(DateFormatIntent, String)] = [
        (.dailyHeader,             "Sat, Aug 15"),
        (.sunEventTime,            "5:39pm"),
        (.forecastUpdatedDateTime, "Sat, Aug 15 @ 5:39pm"),
    ]
    withAppLanguage(.en) {
        for (intent, expected) in anchors {
            #expect(DateFormatSamples.evening.withFormat(intent) == expected,
                    intent.rawValue)
        }
    }
}
```

These deliberately overlap the English rows in the golden table, and the duplication is the point: the golden records what the code currently produces, the anchors what a human decided it *should* produce. If a record run ever drifts the English output, the goldens follow it silently and the anchors go red — a second, human-authored source of truth guarding the machine-authored one.

## Results

- A rendering fix for any language is a single-arm edit instead of a codebase-wide hunt: every date string across app, watch, and widgets flows through 24 intents and one exhaustive `plan(for:)` switch.
- Every per-language deviation — the Hungarian meridiem, the Vietnamese day-first header, the CJK hour counters, the six declared weekday arrays — is one arm in a named classifier, carrying its reason.
- "We forgot to handle Vietnamese" is now a build error, not a shipped bug: a new dated surface fails three separate compiles until it is classified.
- Bans the compiler cannot see are held by source-scanning tests — no lowercasing chained onto a rendered date, no orphan intents, no new raw format strings — and hand-verified English anchors guard the machine-recorded goldens.

## Lessons Learned

- **Name the UI surface, not the format.** A call site that says `.sunEventTime` stays correct after you change how sun times render in Finnish; `"h:mma"` does not. The intent is the interface; the format string belongs in exactly one switch.
- **Keep per-language variance in named classifiers.** A language list inside a formatting rule loses the reason for the deviation, so the next language that needs it gets added elsewhere; a named classifier keeps the quirk and its reason together.
- **A lowercase meridiem is construction, not post-processing.** Shape the AM/PM symbols when you build the formatter; lowercasing the rendered string applies the wrong casing to the wrong characters, the exact habit that makes date code un-auditable.
- **Make the compiler refuse an unclassified format.** An exhaustive switch with no `default` turns a forgotten language from a runtime bug into a build error, and three of them over one enum are three questions you must answer before shipping.
- **Guard machine-recorded goldens with hand-written anchors.** Wherever a test regenerates its own expectations, a small parallel set only a human edits catches the regression the regeneration would otherwise bless.

The rulebook is specific to dates, but the shape transfers to any formatting problem with a combinatorial explosion behind it — currency, units, addresses, names. Route everything through named intents, push variance into named classifiers, ban post-processing, and let exhaustive switches refuse anything half-specified. It sits beside the [rendered-width validation](/rendered-width-validation/) and [vendor language probing](/probing-vendor-language-support/) that guard the rest of the localization work.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.
