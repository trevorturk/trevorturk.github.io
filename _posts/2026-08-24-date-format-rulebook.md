---
layout: post
title: "A Date-Format Rulebook"
date: 2026-08-24 08:10:00 -0600
summary: "How one weather app sends every date and time string through one list of UI intents and one exhaustive switch, keeps each language's quirks in named classifiers instead of scattered language lists, and gets a lowercase 'pm' by shaping the formatter instead of lowercasing its output."
tags: [swift, ios, localization, i18n, dates]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

The design wants "5pm", not "5 PM". So a screen writes `"ha"` and lowercases the result: `formatter.string(from: date).lowercased()`. That gives "5pm" in English and quietly breaks everywhere else. Hungarian's meridiem, stripped and lowercased, becomes "de", the word for "but". Japanese, Korean, and Chinese don't use a meridiem for short hours at all, and their dates want a year-month-day shape that no Western pattern produces. Vietnamese writes the weekday and the month as numbers, so a month-first header is three numbers in a row.

Another screen writes `"EEE"`, a third copies the lowercasing line, and now the formatters disagree on capitalization and meridiem. What each screen should look like is spread across dozens of call sites. When a translator reports one of these breaks, there's no one switch to fix. There's a search-and-replace and a hope that you found them all.

The scale makes this expensive. [Hello Weather](https://helloweather.com) shows dates in 24 distinct places: a sunrise time under an icon, a weekday rail under the hourly chart, a watch complication hour capped at five characters. It does that in 27 languages, with a user-selectable 12- or 24-hour clock. One person maintains it, with a model doing the translating and no agency behind it, so there's no room for date code only one screen understands. This post is the date piece of that localization work. [Rendered-width validation](/rendered-width-validation/) and [vendor language probing](/probing-vendor-language-support/) guard the rest.

## The Solution

We treat date formatting as configuration, not as code each screen writes for itself. Every date string in the app, watch, and widgets goes through three stages:

- An intent enum whose cases name places in the UI (`sunEventTime`, `dailyHeader`), not formats.
- One exhaustive `plan(for:)` switch that maps an intent plus a language to a render plan.
- Named classifiers on the language enum. A classifier is a property that answers one question about a language ("does it keep its punctuated am/pm?") with a `switch` over every language, so no language list sits inside an intent arm.

The call site knows nothing. `Text(sunset.withFormat(.sunEventTime))` carries no format string, no lowercasing, no locale plumbing. `withFormat` looks up the current language and time zone, asks the intent for its plan, and renders.

### One enum of intents, one exhaustive plan

A reviewer fixing how sun times render in Finnish should have one place to look. Adding a dated screen should be impossible without deciding how it reads in every language. An exhaustive switch with no `default` gives us both. The block below shows the shape in miniature: the plan struct, a slice of the intent enum, three classifiers, and the switch that ties them together. It compiles as written.

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

enum MeridiemForm { case nativePunctuated, plainLetters }
enum DateOrderGroup { case western, cjk }
enum HeaderDateOrder { case monthDay, dayMonth }

extension Language {
    // Most languages take bare lowercase letters. Three keep the punctuated
    // native form: stripped, the Czech and Finnish markers are not words, and
    // Hungarian's "de" is the word for "but".
    var meridiemForm: MeridiemForm {
        switch self {
        case .hu:                       return .nativePunctuated
        case .en, .de, .fr, .ja, .vi:   return .plainLetters
        }
    }

    var standardMeridiem: Meridiem {
        switch meridiemForm {
        case .nativePunctuated: return .localeDefault
        case .plainLetters:     return .plainLower
        }
    }

    // CJK renders native 年月日 via ICU templates; Western uses explicit patterns.
    var dateOrderGroup: DateOrderGroup {
        switch self {
        case .ja:                       return .cjk
        case .en, .de, .fr, .hu, .vi:   return .western
        }
    }

    // Exactly one language orders headers day-first: Vietnamese, whose numeric
    // weekday and month collide with the day in month-first order.
    var headerDateOrder: HeaderDateOrder {
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
    let context: Formatter.Context   // .beginningOfSentence capitalizes the first word
    let meridiem: Meridiem

    static func pattern(_ p: String, capitalized: Bool) -> DateRenderPlan {
        DateRenderPlan(kind: .pattern(p), twentyFourHourVariant: nil,
                       context: capitalized ? .beginningOfSentence : .unknown,
                       meridiem: .localeDefault)
    }
    static func time(_ p: String, or h24: String, meridiem: Meridiem) -> DateRenderPlan {
        DateRenderPlan(kind: .pattern(p), twentyFourHourVariant: h24,
                       context: .unknown, meridiem: meridiem)
    }
    static func template(_ t: String, or h24: String? = nil) -> DateRenderPlan {
        DateRenderPlan(kind: .template(t), twentyFourHourVariant: h24,
                       context: .unknown, meridiem: .localeDefault)
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
            switch language.dateOrderGroup {
            case .cjk: return .template("MMMdE")
            case .western:
                switch language.headerDateOrder {
                case .monthDay: return .pattern("E, MMM d", capitalized: true)
                case .dayMonth: return .pattern("E, d MMM", capitalized: true)
                }
            }
        case .sunEventTime:
            switch language.dateOrderGroup {
            case .cjk:     return .template("hmm", or: "Hmm")
            case .western: return .time("h:mma", or: "H:mm", meridiem: language.standardMeridiem)
            }
        case .forecastUpdatedDateTime:
            switch language.dateOrderGroup {
            case .cjk: return .template("MMMdEhmm", or: "MMMdEHmm")
            case .western:
                switch language.headerDateOrder {
                case .monthDay: return .time("E, MMM d @ h:mma", or: "E, MMM d @ H:mm",
                                             meridiem: language.standardMeridiem)
                case .dayMonth: return .time("E, d MMM @ h:mma", or: "E, d MMM @ H:mm",
                                             meridiem: language.standardMeridiem)
                }
            }
        }
    }
}
```

The intent arms hold no list of languages. `dailyHeader` asks `language.dateOrderGroup` and `language.headerDateOrder` and branches on the answers. The intent knows the shape of a header, and the classifier knows which languages deviate from it.

### Per-language quirks in named classifiers

Once a `switch self { case .fi, .hu: … }` shows up inside a formatting rule, the reason for the exception is lost, and the next language that needs the same treatment gets added somewhere else. A named classifier keeps the quirk and its reason together, so every exception in the app is one arm you can point at.

Each classifier exists because a real language broke a naive rule. `meridiemForm` records that Hungarian keeps its punctuated form. The locale test that pins that rendering carries the reason, "Adjudicated: keep the period — bare 'de' is the Hungarian word for 'but'", which nobody would think to write from first principles. `dateOrderGroup` is why the CJK languages (Chinese, Japanese, Korean) get ICU templates, where the system picks the field order, instead of a hardcoded Western order. A sibling classifier, `compactHourStyle`, is why their short hours render as "9時"/"9时"/"9시" rather than "9pm". `headerDateOrder` is a single arm for Vietnamese. We didn't make a vague "some languages are day-first" category, because it would pull in languages that don't want it.

### Shaping the meridiem at construction

The house voice wants "5:39pm": lowercase, no space, no periods. Lowercasing the rendered string is wrong twice over. It lowercases any weekday or month name that should stay capitalized, and unless you pass a locale it uses ASCII casing rules, which mangles languages like Turkish. The right moment to decide the meridiem is when the formatter is built, by changing its AM/PM symbols before it renders anything. This function does that, and it runs as written. In the app it's a private method on the formatter cache.

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
            let letters = symbol.unicodeScalars.filter {
                CharacterSet.letters.contains($0) || CharacterSet.decimalDigits.contains($0)
            }
            return String(letters).lowercased(with: locale)
        }
        if let am = formatter.amSymbol { formatter.amSymbol = bareLowercase(am) }
        if let pm = formatter.pmSymbol { formatter.pmSymbol = bareLowercase(pm) }
    }
}
```

It keeps only the letters and decimal digits from the locale's own symbol and lowercases them with that locale's casing rules. "5:39 p. m." becomes "5:39pm", and Turkish "ÖS" becomes "ös" under Turkish casing, not ASCII. The guard on the `a` pattern character lets formatters with no meridiem skip the work. When a call site really needs a different case, it passes a declared `case:` parameter, and the formatting layer applies it once, with the locale. A handful of alternate styles do that, to set a rail in caps. The rest never do, because the voice they want is already in the symbols.

### Source-scanning tests for the bans

"Never lowercase a rendered date" only holds if nobody can quietly bring it back, and the compiler can't see a `.lowercased()` habit the way it sees a missing switch arm. So a test reads every Swift file in the app, runs a regex over each one, and fails with the matches it found. This is the complete check, under Swift Testing, as written.

```swift
import Testing
import Foundation

@Suite struct DateFormatCallsiteTests {
    static let sourceRoot = URL(fileURLWithPath: #filePath)
        .deletingLastPathComponent()
        .deletingLastPathComponent()
        .appendingPathComponent("App")   // your source tree

    static func swiftSources() -> [String] {
        guard let files = FileManager.default.enumerator(
            at: sourceRoot, includingPropertiesForKeys: nil) else { return [] }
        return files.compactMap { item in
            guard let url = item as? URL, url.pathExtension == "swift" else { return nil }
            return try? String(contentsOf: url, encoding: .utf8)
        }
    }

    @Test func caseMethodsNeverChainOnFormattedDates() throws {
        let sources = Self.swiftSources()
        try #require(sources.count > 100, "source scan found too few files")

        let chain = try NSRegularExpression(pattern:
            #"withFormat(?:ForDeviceTimeZone)?\([^()]*\)\.(?:lowercased|uppercased|capitalized)"#
            + #"|withRelativeFormat\(\)\.(?:lowercased|uppercased|capitalized)"#)
        var offenders: [String] = []
        for source in sources {
            let range = NSRange(source.startIndex..<source.endIndex, in: source)
            for match in chain.matches(in: source, range: range) {
                if let r = Range(match.range, in: source) {
                    offenders.append(String(source[r]))
                }
            }
        }
        #expect(offenders.isEmpty,
            "case methods chained on rendered date strings: \(offenders). Use withFormat's "
            + "`case:` parameter (.lower/.upper) so the formatting layer applies the transform "
            + "once, locale-aware. (Scan catches case chained directly on a withFormat(...) call "
            + "only; case reached through an intermediate variable or wrapper must be caught in review.)")
    }
}
```

The failure message admits the scan's limit. It catches a case method chained directly on the call, and a transform reached through a wrapper or an intermediate variable is left to review. Two sibling tests round out the bans. One fails if any intent case has no call site, because an intent with no UI is either dead code or a wiring bug, and either way the test forces the question. The other freezes the small set of raw format strings the frozen alternate-style catalog may still use, so new UI can't slip in a raw `dateFormat` and has to add an intent instead. Two more exhaustive switches classify every intent by width budget and by worst-case sample, so adding a case fails three separate compiles until it's classified three ways.

### Hand-pinned English anchors

Every intent's rendering in every language is captured in committed golden files, which a machine records and a machine updates. A [companion post](/golden-files-per-language/) covers that matrix and the fast macOS test tier. The risk is that a careless record run could bless a regression into the English golden, and the diff would look like every other regeneration. So a handful of English outputs are also pinned by hand, in a table a person edits and a record run never rewrites.

```swift
@Test func englishAnchorsHoldAtTheEveningInstant() {
    // A hand-picked instant (2026-08-15, 17:39 local), not a machine sample.
    // withFormat / withAppLanguage / DateFormatSamples are the test harness.
    // Three of the eight 12-hour anchors; the real table also pins the 24-hour clock.
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

These overlap the English rows in the golden table on purpose. The golden records what the code produces today, and the anchors record what a person decided it should produce. If a record run ever drifts the English output, the goldens follow it silently and the anchors go red.

## Results

- Every date string across app, watch, and widgets goes through 24 intents and one exhaustive `plan(for:)` switch. A rendering fix for any language is a one-arm edit instead of a codebase-wide hunt. Each per-language exception, from the Hungarian meridiem and the Vietnamese day-first header to the CJK hour counters and the six declared weekday arrays, is one classifier arm that carries its reason.
- "We forgot to handle Vietnamese" is a build error, not a shipped bug. The cost is friction: a new dated screen fails three separate compiles until it's classified by plan, width budget, and worst-case sample.
- The bans the compiler can't see (no lowercasing chained on a rendered date, no orphan intents, no new raw format strings) hold in source-scanning tests. Hand-pinned English anchors duplicate golden rows on purpose, to guard the machine-recorded ones.

## Lessons Learned

- **Give a one-language exception its own arm and write down why.** A category invented to hold it, like "some languages are day-first", pulls in languages that don't want it.
- **Decide the output shape when you build the formatter, not by editing what it emits.** Post-processing applies the wrong rule to the wrong characters and hides where the decision was made.
- **Where a test regenerates its own expectations, keep a small parallel set only a person edits.** Otherwise the regeneration blesses the regression it was meant to catch.
- **The same shape works for any formatting problem with a lot of variation: currency, units, addresses, names.** Route through named intents, push the variation into classifiers, ban post-processing, and let exhaustive switches reject a case that isn't decided for every language.
