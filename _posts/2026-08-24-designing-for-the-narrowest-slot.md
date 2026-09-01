---
layout: post
title: "Designing for the Narrowest Slot"
date: 2026-08-24 09:20:00 -0600
summary: "Designing localized copy for the smallest slot a weather app owns: character budgets for compact date labels, layout-derived point budgets for stat cards, and tests that measure real device ink across 27 languages."
tags: [swift, ios, localization, i18n, tooling]
---

## The Problem

A chart's weekday rail holds seven ticks, and each tick holds three characters. `Mon Tue Wed` fits. So does `9am 10am` under the bars of the hourly chart, `Falling fast` on a two-line stat card, and a bare hour on a watch complication. Every one of these slots was drawn in English, looks perfect in English, and does not wrap or grow.

Then a model translates the strings into 26 more languages, and nothing in the toolchain knows how wide any of them will be. A German label one character too wide clips the same way in the app, the widget, and on the watch, because it is the same string in the same slot. A language whose weekday abbreviations collapse two days onto the same three letters is worse: the axis renders two identical ticks, a data bug with no visible truncation. Six of the supported languages fail one test or the other straight from the system calendar's abbreviations. Without a gate, the first report comes from a screenshot or a customer, after shipping.

[Hello Weather](https://helloweather.com) shows sentences where a typical weather app shows a number and an icon: pressure trends, precip timing, moon phases, air-quality advice, in 27 languages, and most of that text lives in slots like these. It is a one-person product with no localization agency behind it, so the fix has to fail a build rather than rely on a native speaker eyeballing 27 screenshots. Two instruments do that. Dates count characters, because a weekday rail is a contract with a hard cap. Stat cards measure ink, because a card is a layout and rendered glyph width is what overflows it.

## The Solution

Route every date and time string through one rulebook so the constrained surfaces are enumerable, then gate them with character-budget contract tests. Derive the stat-card point budgets from the grid formula the view uses, then gate them with tests that measure real device ink. Neither number is hand-picked. The exceptions in both are declared as data, so a string somebody decided to keep re-flags the moment its value changes.

### Compact date labels get a character budget

Every date string flows through one enum of intents, each case naming a UI surface (`hourlyChartWeekday`, `statsChartHour`, `complicationHour`) rather than a format. The [date rulebook post](/date-format-rulebook/) covers that enum. What matters here is that enumerated surfaces let a test iterate the ones that render in a tight slot and assert a hard budget on all of them, in all 27 languages.

The compact weekday rail carries the tightest rule: each label is at most three characters and three Unicode scalars, letters and digits only, and the seven labels for a week must be pairwise distinct. Most languages satisfy it from the system calendar's abbreviated or standalone-short symbols. For the six that do not, the classifier returns a hand-declared seven-symbol array with the ruling attached. Here is the resolver and the contract test that enforces it, self-contained:

```swift
import Foundation
import Testing

enum Language: String, CaseIterable {
    case en, de, fr, ja, ko, da, nb, pt, ro, th
    // one case per supported language
}

enum WeekdayStyle {
    case abbreviated              // system calendar "E" symbols
    case standaloneShort          // system calendar "cccccc" symbols
    case declared([String])       // hand-chosen when the platform's break a rule
}

func weekdayStyle(for language: Language) -> WeekdayStyle {
    switch language {
    case .en, .ja, .ko: return .abbreviated
    case .fr: return .standaloneShort
    // Standard abbreviations blow the 3-char budget or collide, so declare them:
    case .de: return .declared(["So", "Mo", "Di", "Mi", "Do", "Fr", "Sa"])
    case .da, .nb: return .declared(["Sø", "Ma", "Ti", "On", "To", "Fr", "Lø"])
    case .pt: return .declared(["Dom", "Seg", "Ter", "Qua", "Qui", "Sex", "Sáb"])
    case .ro: return .declared(["Dum", "Lun", "Mar", "Mie", "Joi", "Vin", "Sâm"])
    case .th: return .declared(["อา", "จ", "อ", "พ", "พฤ", "ศ", "ส"])
    }
}

func compactWeekdays(for language: Language) -> [String] {
    switch weekdayStyle(for: language) {
    case .declared(let symbols):
        return symbols
    case .abbreviated, .standaloneShort:
        // Simplified: the real rulebook formats dates with the "E" / "cccccc" patterns.
        var cal = Calendar(identifier: .gregorian)
        cal.locale = Locale(identifier: language.rawValue)
        return cal.shortStandaloneWeekdaySymbols  // Sun…Sat for the demo
    }
}

@Test func compactWeekdaysStayCompactAndDistinct() {
    for language in Language.allCases {
        let labels = compactWeekdays(for: language)
        for label in labels {
            #expect(label.count <= 3 && label.unicodeScalars.count <= 3, "\(language.rawValue): \(label)")
            #expect(label.unicodeScalars.allSatisfy { $0.properties.isAlphabetic || ("0"..."9").contains(Character($0)) },
                    "\(language.rawValue): \(label)")
        }
        #expect(Set(labels).count == 7, "\(language.rawValue): \(labels)")
    }
}
```

Notice that the exception is data, not a code path. The array is reviewable, the same test proves it fits, and the next person can see why German stopped reading its symbols from the calendar. Overriding the platform's locale data feels wrong until you remember that data was optimized for legibility in a paragraph, not for a three-character tick that must stay distinct from six neighbors. When the platform optimized for a different constraint than yours, override with declared data and attach the reason.

Hour labels get a parallel budget of at most five characters, with English pinned exactly to `12am…11pm` and `0:00…23:00`. The house voice is `4pm` rather than the CLDR-correct `4 PM` or `4 p.m.` because the labels sit under bars in the hourly chart and cannot be wider than the bars. The rulebook produces that form by assigning bare lowercase meridiem symbols when the formatter is built, rather than lowercasing its output. Chinese, Japanese, and Korean render a native hour counter (`9時`, `9시`) instead of a meridiem. That is how those languages write a compact clock hour, and a Latin `am`/`pm` on a CJK digit is both wrong and wider. The watch complication in 24-hour mode goes tighter still: a bare `17`, no colon, no meridiem. Every one of these is pinned in a contract test whose English anchors are typed by a person and never machine-updated, so a careless snapshot re-record cannot quietly bless a regression.

### Stat cards measure ink

A weekday tick is a discrete contract, so a character count is the right instrument. A stat card is a two-line cell in an adaptive grid, and what overflows it is ink: the rendered width of the glyphs. A Cyrillic string and a Latin string of the same length are different widths, and an 18-point bold subtitle and a 13-point regular description do not share a budget. So the stat half measures real device ink, and derives the budget it measures against from the grid arithmetic the SwiftUI view uses. Change the layout and the budget follows in one line:

```swift
import UIKit

enum StatBudget {
    static let deviceWidth: CGFloat = 390     // narrowest currently-sold iPhone
    static let gridOuterPadding: CGFloat = 32
    static let gridSpacing: CGFloat = 10
    static let gridMinimumColumn: CGFloat = 165
    static let cardPadding: CGFloat = 32
    static let headroom: CGFloat = 0.95       // 5% safety band

    static var descriptionBudget: CGFloat {
        let available = deviceWidth - gridOuterPadding
        let columns = floor((available + gridSpacing) / (gridMinimumColumn + gridSpacing))
        let column = (available - (columns - 1) * gridSpacing) / columns
        return column - cardPadding
    }
    static var passBar: CGFloat { descriptionBudget * headroom }
}

enum Verdict { case ok, margin, over }

/// Measures a string on the real iOS system font and grades it against the budget.
func grade(_ string: String, size: CGFloat = 13, weight: UIFont.Weight = .regular) -> Verdict {
    let font = UIFont.systemFont(ofSize: size, weight: weight)
    let width = ceil(NSAttributedString(string: string, attributes: [.font: font]).size().width)
    if width > StatBudget.descriptionBudget { return .over }
    if width > StatBudget.passBar { return .margin }
    return .ok
}
```

The 390 is the narrowest iPhone still on sale. A legacy 375-point figure lives in the file too, but it is informational only: reported so you can see what would clip on older hardware, never gating, and never a reason to shorten copy. There is no separate English ceiling either. An English string over the grid is a finding, not a new budget invented to make it pass. `descriptionBudget` here works out to 142 points, with a `MARGIN` band 5% under it worth a device glance.

### The always-on gate pins the known-tight strings

That helper runs at two tiers. The always-on tier is small and fast. It fails `bin/unit-test` if a hand-curated set of the known-tightest lines goes over: the widest pressure trend per language, timed precip strings, a few fixed singles. It is the critical set, and it measures on the device font every run:

```swift
import Testing

struct StatWidthTests {
    static let languages = ["cs", "da", "de", "el", "en", "es", "fi", "fr", "hi",
                            "hu", "id", "it", "ja", "ko", "nb", "nl", "pl", "pt",
                            "ro", "ru", "sv", "th", "tr", "uk", "vi", "zh-Hans", "zh-Hant"]

    // "Falling fast" is the widest pressure trend; the full sweep pulls the live
    // value per language, this pins the ones that were tightest against the floor.
    static let fallingFast: [String: String] = [
        "en": "Falling fast", "de": "Fällt schnell", "el": "Ταχεία πτώση",
        "ru": "Резко падает", "pl": "Szybko spada", "ja": "急降下",
        // …one row per language
    ]

    private func resolved(_ resource: LocalizedStringResource, _ language: String) -> String {
        var resource = resource
        resource.locale = Locale(identifier: language)
        return String(localized: resource)
    }

    @Test(arguments: languages)
    func pressureLineFitsTheFloorGrid(language: String) {
        let trend = Self.fallingFast[language] ?? "Falling fast"
        for levelKey in ["Low pressure", "Normal pressure", "High pressure"] {
            let level = resolved(LocalizedStringResource(String.LocalizationValue(levelKey), bundle: .main), language)
            // The joiner is a catalog template too, so each language composes its own line.
            let line = resolved(
                LocalizedStringResource("%@ and %@.", defaultValue: "\(level) and \(trend.lowercased()).", bundle: .main),
                language
            )
            #expect(grade(line) != .over, "\(language): \"\(line)\"")
        }
    }
}
```

The second tier is a full sweep, env-gated so it never runs under CI. It grades every stat-card slot in all 27 languages against both the catalog values and the live strings from our server. Many of the widest strings are composed server-side and returned localized, so a wrapper script converts the web repo's locale files to JSON and the report reads them in. Its output is a blessed markdown report, reviewed on every change like a copy edit. A row that is over budget on purpose is pinned by exact value in an `accepted` map, so an allowed over-budget term re-flags the instant its string changes. "This canonical term has no shorter natural form" is a legitimate outcome. "We stopped looking" is not, and pinning the value is the difference.

A drift tripwire keeps the two tiers honest. It fails the build the moment the set of stat cards changes and names the files to update, so nobody can add a card that the always-on gate keeps passing while the report silently stops covering it.

### The runtime policy is shrink, never truncate

Budgets and tests keep copy inside the slot at author time. The view still needs a safety net at render time for the string that slips through, or the accessibility size that inflates everything. The policy is to scale down, never clip mid-word. Descriptions tolerate a gentler `minimumScaleFactor` than the bolder, tighter subtitles, and both are pinned to a single line so the grid row height never jumps:

```swift
import SwiftUI

struct StatCard: View {
    let title: String       // e.g. "Pressure"
    let subtitle: String    // 18pt bold, very tight — "1013 hPa"
    let description: String  // 13pt regular (.footnote) — "High pressure and falling fast."

    var body: some View {
        VStack(alignment: .leading, spacing: 4) {
            Text(title)
                .font(.caption2.weight(.semibold))
                .lineLimit(1)
            Text(subtitle)
                .font(.system(size: 18).weight(.bold))
                .lineLimit(1)
                .minimumScaleFactor(0.8)
            Text(description)
                .font(.footnote)
                .lineLimit(1)
                .minimumScaleFactor(0.9)
        }
        .padding(.vertical, 18)
        .padding(.horizontal, 16)
        .frame(maxWidth: .infinity, alignment: .leading)
    }
}
```

The residual `OVER` rows in the full report are the ones this net catches, adjudicated as scale-covered and recorded so the next reader knows they were a decision. The scale factor is the floor of last resort, not a license to skip the budget. A card whose default text needs shrinking on the narrowest phone is still a finding.

## Results

- One contract test enforces the weekday rule across all 27 languages. Six (Danish, German, Norwegian, Portuguese, Romanian, Thai) carry declared arrays with the ruling attached, so "why does the German label look different?" is answered by a committed array, not a guess.
- A width regression fails `bin/unit-test` instead of reaching a screenshot. The env-gated sweep extends that to server-owned strings, at the cost of a committed report to re-bless on every copy change.
- Recalibrating the floor from 375 to 390 let a batch of over-shortened server phrases go back to their fuller wording, with the report proving each longer string now fit.
- The residual over-budget rows are covered by `minimumScaleFactor` rather than shorter copy, and each one is recorded as a decision.

## Lessons Learned

- **Budget the narrowest slot you actually sell.** Not the smallest device that ever existed, and not the one on your desk. Writing that number down settles later arguments about whether a row "really" overflows.
- **Derive budgets from the layout, never hand-pick them.** A number copied from the grid arithmetic cannot drift from the view. A hand-picked one rots when someone changes a spacing constant.
- **Pin every exception by exact value.** An override of platform data or an accepted over-budget string is fine as data with a reason attached, because it re-flags the moment the value changes.
- **Fail the build when the covered set changes.** A report that goes stale without anyone catching it is worse than no report.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the weekday rail and the six languages whose abbreviations break it, the title is one clause, Results drops the bullets that restated the Solution, and Lessons Learned is four rules that transfer. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The test's language array has 27 entries, so "25 more languages" became 26; French moved from the abbreviated to the standalone-short weekday style, the weekday test gained the real three-scalar cap, the pressure-line test now resolves the localized "%@ and %@." template instead of hardcoding an English joiner, the card view's fonts and padding were matched to the real cell, and the full sweep is described as reading web locales through a wrapper script's JSON conversion rather than directly.
