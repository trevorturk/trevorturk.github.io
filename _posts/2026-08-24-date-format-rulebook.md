---
layout: post
title: "A Date-Format Rulebook"
date: 2026-08-24 08:10:00 -0600
summary: "Why one weather app routes every date and time string through a single enum of UI intents and one exhaustive switch, keeps per-language quirks in named classifiers instead of scattered language lists, and treats a lowercase 'pm' as a construction decision rather than a string hack."
tags: [swift, ios, localization, i18n, dates]
---

## The Problem

The design wants "5pm", not "5 PM", so a screen writes `"ha"` and lowercases the result. `formatter.string(from: date).lowercased()` gives "5pm" in English and quietly breaks everywhere else. Hungarian's meridiem, stripped and lowercased, becomes "de", the word for "but". Japanese, Korean, and Chinese do not use a meridiem for short hours at all, and their dates want a template-driven year-month-day shape no Western pattern produces. Vietnamese writes weekday and month as numerals, so a month-first header is three numbers colliding.

Another screen writes `"EEE"`, a third copies the lowercasing line, and now the formatters disagree on capitalization and meridiem. The knowledge of what each surface should look like is smeared across dozens of call sites. When a translator reports one of these breaks, there is no one switch to fix. There is a search-and-replace and a hope that you found them all.

The scale makes the smear expensive. [Hello Weather](https://helloweather.com) renders dates on 24 distinct surfaces, a sunrise time under an icon, a weekday rail under the hourly chart, a watch complication hour capped at five characters, in 27 languages, with a user-selectable 12- or 24-hour clock. One person maintains it, with a model doing the translating and no agency behind it, so there is no room for date code only one screen understands. This post is the date piece of that localization work. [Rendered-width validation](/rendered-width-validation/) and [vendor language probing](/probing-vendor-language-support/) guard the rest.

## The Solution

Date formatting is declared configuration, not imperative code. Every date string in the app, watch, and widgets flows through three stages:

- An intent enum whose cases name UI surfaces (`sunEventTime`, `dailyHeader`), not formats.
- One exhaustive `plan(for:)` switch mapping an intent plus a language to a render plan.
- Named classifiers on the language enum that hold per-language quirks, each a `switch` over every language rather than a language list buried in an intent arm.

The call site is deliberately dumb. `Text(sunset.withFormat(.sunEventTime))` carries no format string, no lowercasing, no locale plumbing. `withFormat` looks up the current language and time zone, asks the intent for its plan, and renders.

### One enum of intents, one exhaustive plan

A reviewer fixing how sun times render in Finnish should have exactly one place to look, and adding a dated screen should be impossible without deciding how it reads in every language. An exhaustive switch with no `default` delivers both. The block below is the whole shape in miniature, plan struct, a slice of the intent enum, three classifiers, and the switch that ties them together, and it compiles as written.

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

The intent arms contain no list of languages. `dailyHeader` asks `language.dateOrderGroup` and `language.headerDateOrder` and branches on the answers: the intent knows the shape of a header, and the classifier knows which languages deviate. Because `plan(for:)` has no `default`, a new case does not compile until it has an arm.

### Per-language quirks in named classifiers

The moment a `switch self { case .fi, .hu: … }` appears inside a formatting rule, the reason for the deviation is lost, and the next language that needs the same treatment gets added somewhere else. A named classifier keeps the quirk and its reason together, so every deviation in the app is one arm you can point at.

Each classifier exists because a real language broke a naive rule. `meridiemForm` records that Hungarian keeps its punctuated form; the locale test that pins the rendering carries the reason, "Adjudicated: keep the period — bare 'de' is the Hungarian word for 'but'", a note you would not think to write from first principles. `dateOrderGroup` is why CJK languages get ICU templates instead of a hardcoded Western sequence, and a sibling `compactHourStyle` is why their short hours render as "9時"/"9时"/"9시" rather than "9pm". `headerDateOrder` is a targeted arm for Vietnamese, not a vague "some languages are day-first" category that would sweep in languages that do not want it.

### Shaping the meridiem at construction

The house voice wants "5:39pm": lowercase, no space, no periods. Lowercasing the rendered string is wrong twice. It also lowercases any weekday or month name that should stay capitalized, and it applies ASCII casing rules unless you pass a locale, which mangles languages like Turkish. The place to decide the meridiem is when the formatter is built, by shaping its AM/PM symbols before it renders. This function does that, and runs as written; in the app it is a private method on the formatter cache.

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

It keeps only letters and decimal digits from the CLDR symbol and lowercases with the locale's own casing rules. "5:39 p. m." becomes "5:39pm", and Turkish "ÖS" becomes "ös" under Turkish casing, not ASCII. The guard on the `a` pattern character lets formatters with no meridiem skip the work. When a call site genuinely needs a different case, it passes a declared `case:` parameter that the formatting layer applies once, locale-aware. A handful of alternate styles do, to set a rail in caps; the rest never do, because the voice they want is already in the symbols.

### Source-scanning tests for the bans

"Never lowercase a rendered date" only holds if it cannot be quietly reintroduced, and the compiler cannot see a `.lowercased()` habit the way it sees a missing switch arm. So a test reads every Swift file in the app, runs a regex over it, and fails with the offending matches. This is the whole check, under Swift Testing, as written.

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

The failure message admits the scan's limit: it catches a case method chained directly on the call, and a transform reached through a wrapper or an intermediate variable is left to review. Two sibling tests round out the bans. One fails if any intent case has no call site: an intent with no UI is either dead code or a wiring bug, and either way the test forces the question. The other freezes the small set of raw format strings the frozen alternate-style catalog may still use, so new UI cannot smuggle in a raw `dateFormat` and has to add an intent instead. Two further exhaustive switches classify every intent by width budget and by worst-case sample, so adding a case fails three separate compiles until it is classified three ways.

### Hand-pinned English anchors

The rendering of every intent in every language is captured in committed golden files, machine-recorded and machine-updated. A [companion post](/golden-files-per-language/) covers that matrix and the fast macOS test tier. The hazard is that a careless record run could bless a regression into the English golden, and the diff would look like every other regeneration. So a handful of English outputs are also pinned by hand, in a table a human edits and a record run never rewrites.

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

These deliberately overlap the English rows in the golden table. The golden records what the code currently produces, and the anchors record what a human decided it should produce. If a record run ever drifts the English output, the goldens follow it silently and the anchors go red.

## Results

- Every date string across app, watch, and widgets flows through 24 intents and one exhaustive `plan(for:)` switch. A rendering fix for any language is a single-arm edit instead of a codebase-wide hunt, and each per-language deviation, from the Hungarian meridiem and the Vietnamese day-first header to the CJK hour counters and the six declared weekday arrays, is one classifier arm carrying its reason.
- "We forgot to handle Vietnamese" is a build error, not a shipped bug. The accepted cost is friction: a new dated surface fails three separate compiles until it is classified by plan, width budget, and worst-case sample.
- The bans the compiler cannot see, no lowercasing chained on a rendered date, no orphan intents, no new raw format strings, hold in source-scanning tests. Hand-pinned English anchors duplicate golden rows on purpose to guard the machine-recorded ones.

## Lessons Learned

- **Give a one-language exception its own arm and write down why.** A category invented to hold it, "some languages are day-first", sweeps in languages that do not want it.
- **Decide output shape when you build the formatter, never by editing what it emits.** Post-processing applies the wrong rule to the wrong characters and hides where the decision was made.
- **Where a test regenerates its own expectations, keep a small parallel set only a human edits.** Otherwise the regeneration blesses the regression it was meant to catch.
- **The shape transfers to any formatting problem with combinatorial variance: currency, units, addresses, names.** Route through named intents, push variance into classifiers, ban post-processing, and let exhaustive switches refuse the half-specified.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the lowercased meridiem that turns Hungarian's "de" into "but", the two sibling-post links moved from a closing paragraph into The Problem, the title dropped its subtitle, and Lessons Learned keeps only rules the sections do not already state. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The rulebook excerpt now uses the real classifier names (`meridiemForm`, `dateOrderGroup`, `headerDateOrder`), the real `Formatter.Context` field, and the CJK and day-first arms the source has for `sunEventTime` and `forecastUpdatedDateTime`; the meridiem shaper filters letters and digits as the source does; the callsite test carries the source's full regex and its message admitting the scan's limit; the Hungarian "but" note is attributed to the locale test where it lives; the CJK hour counters are credited to `compactHourStyle`; the watch complication is capped at five characters, not three; and the claim that production paths never pass `case:` is corrected, since a few alternate styles do.
