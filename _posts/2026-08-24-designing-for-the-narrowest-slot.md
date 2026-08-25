---
layout: post
title: "Designing for the Narrowest Slot: Character Budgets, Native Hour Counters, and a 390-Point Gate"
date: 2026-08-24 09:20:00 -0600
summary: "Designing localized copy for the smallest slot a weather app owns: character budgets for compact date labels, layout-derived point budgets for stat cards, and tests that measure real device ink across 27 languages."
tags: [swift, ios, localization, i18n, tooling]
---

## The Problem

A typical weather app shows a number and an icon. [Hello Weather](https://helloweather.com) shows sentences: pressure trends, precip timing, moon phases, air-quality advice, all localized into 27 languages. The text is not decoration around the layout, it *is* the layout, and most of it lives in the narrowest slot the product owns: a weekday tick on a chart axis, an hour label under a bar, a two-line stat card in a two-column grid, a couple of characters on a watch complication. Those slots do not wrap and do not grow.

English always fits, which is the trap. You draw the chart with `Mon Tue Wed`, the hour axis with `9am 10am`, the stat card with `Falling fast`, and every one looks perfect. Then a model translates into 25 more languages, and nothing in the toolchain knows how wide any of those strings will be. The German label that is one character too wide clips exactly the same way in the app, the widget, and on the watch, because it is the same string in the same slot. You find out from a screenshot, or from a customer, after you ship.

This is a one-person product with no localization agency behind it, so the fix has to be mechanical: something that fails a build, not something that relies on a native speaker eyeballing 27 screenshots. Two instruments do that here. Dates count characters, because a weekday rail is a contract with a hard cap. Stat cards measure ink, because a card is a layout and rendered glyph width is what overflows it.

## The Solution

Route every date and time string through one rulebook so the constrained surfaces are enumerable, then gate them with **character-budget** contract tests. Derive the stat-card **point budgets** from the actual grid formula, then gate them with tests that measure real device ink. Neither number is hand-picked, and the exceptions in both are declared as data, so a string somebody decided to keep re-flags the moment its value changes.

### Compact date labels get a character budget

Every date string flows through one enum of intents, each case naming a UI surface (`hourlyChartWeekday`, `statsChartHour`, `complicationHour`) rather than a format. That is covered in the [date rulebook post](/date-format-rulebook/); what matters here is that because the surfaces are enumerated, a test can iterate the ones that render in a tight slot and assert a hard budget on all of them, in all 27 languages.

The compact weekday rail — the row of day ticks along a chart — carries the tightest rule: each label is **at most three characters, letters and digits only, and the seven labels for a week must be pairwise distinct**. That last clause matters most. A language whose abbreviations collapse two days onto the same three letters renders two identical ticks on one axis: a silent data bug, not a visible truncation.

Most languages satisfy this straight from the system calendar's abbreviated or standalone-short symbols. Six do not: their CLDR abbreviations either blow the three-character budget or collide on distinctness, so the classifier returns a hand-declared seven-symbol array with the ruling attached. Here is the resolver and the contract test that enforces it, self-contained:

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
    case .en, .fr, .ja, .ko: return .abbreviated
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
        var cal = Calendar(identifier: .gregorian)
        cal.locale = Locale(identifier: language.rawValue)
        return cal.shortStandaloneWeekdaySymbols  // Sun…Sat for the demo
    }
}

@Test func compactWeekdaysStayCompactAndDistinct() {
    for language in Language.allCases {
        let labels = compactWeekdays(for: language)
        for label in labels {
            #expect(label.count <= 3, "\(language.rawValue): \(label)")
            #expect(label.unicodeScalars.allSatisfy { $0.properties.isAlphabetic || ("0"..."9").contains(Character($0)) },
                    "\(language.rawValue): \(label)")
        }
        #expect(Set(labels).count == 7, "\(language.rawValue): \(labels)")
    }
}
```

Notice that the exception is data, not a code path: the array is reviewable, the same test proves it fits, and the next person can see exactly why German stopped reading its symbols from the calendar. Overriding the platform's locale data feels wrong until you remember that data was optimized for legibility in a paragraph, not for a three-character axis tick that must stay distinct from six neighbors. When the platform optimized for a different constraint than you have, override with declared data and attach the reason.

The hour labels get a parallel budget: at most five characters, English pinned exactly to `12am…11pm` and `0:00…23:00`. That is why the house voice is `4pm` rather than the CLDR-correct `4 PM` or `4 p.m.`: the labels sit under bars in the hourly chart and cannot be wider than the bars, so the compact form is what the design was drawn with, and the rulebook produces it by assigning bare lowercase meridiem symbols when the formatter is built rather than lowercasing its output. CJK is the interesting decision. Chinese, Japanese, and Korean render a **native hour counter** (`9時`, `9시`) instead of a meridiem, because that is how those languages write a compact clock hour, and because a Latin `am`/`pm` on a CJK digit is both wrong and wider. The watch complication in 24-hour mode goes tighter still — a bare `17`, no colon, no meridiem. Every one of these is pinned in a contract test whose English anchors are typed by a person and never machine-updated, so a careless snapshot re-record cannot quietly bless a regression.

### Stat cards measure ink, and derive the budget

A weekday tick is a discrete contract, so a character count is exactly the right instrument. A stat card is not: it is a two-line cell in an adaptive grid, and what overflows it is ink — the rendered width of the glyphs. A Cyrillic string and a Latin string of the same length are different widths; an 18-point bold subtitle and a 13-point regular description do not share a budget. So the stat half measures real device ink, and derives the budget it measures against from the grid arithmetic the SwiftUI view actually uses. Change the layout and the budget follows in one line:

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

The 390 is the whole point: the narrowest iPhone still on sale. A legacy 375-point figure lives in the file too, but it is informational only — reported so you can see what would clip on older hardware, never gating, and never a reason to shorten copy. There is also no separate English ceiling: an English string over the grid is a finding, not a new budget invented to make it pass. `descriptionBudget` here works out to 142 points, with a `MARGIN` band 5% under it worth a device glance.

### The always-on gate pins the known-tight strings

That helper runs at two tiers. The always-on tier is small, fast, and fails `bin/unit-test` if a hand-curated set of the known-tightest lines goes over — the widest pressure trend per language, timed precip strings, a few fixed singles. It is the critical set, and it measures on the device font every run:

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

    private func resolved(_ key: String, _ language: String) -> String {
        var resource = LocalizedStringResource(String.LocalizationValue(key), bundle: .main)
        resource.locale = Locale(identifier: language)
        return String(localized: resource)
    }

    @Test(arguments: languages)
    func pressureLineFitsTheFloorGrid(language: String) {
        let trend = Self.fallingFast[language] ?? "Falling fast"
        for level in ["Low pressure", "Normal pressure", "High pressure"] {
            let line = "\(resolved(level, language)) and \(trend.lowercased())."
            #expect(grade(line) != .over, "\(language): \"\(line)\"")
        }
    }
}
```

The second tier is a full sweep, env-gated so it never runs under CI. It grades every stat-card slot in all 27 languages against both the catalog values and the live strings from our server — many of the widest strings are composed server-side and returned localized, so the report reads the web repo's locale files directly and pulls them in. Its output is a blessed markdown report, reviewed on every change like a copy edit, and it carries its own adjudications: a row that is over budget on purpose is pinned by exact value in an `accepted` map, so an allowed over-budget term re-flags the instant its string changes. "This canonical term has no shorter natural form" is a legitimate outcome; "we stopped looking" is not — pinning the value is the difference.

A drift tripwire keeps the two tiers honest: it fails the build the moment the set of stat cards changes and names the files to update, so nobody can add a card that the always-on gate keeps passing while the report silently stops covering it.

### The runtime policy is shrink, never truncate

Budgets and tests keep copy inside the slot at author time; the view still needs a safety net at render time for the string that slips through, or the accessibility size that inflates everything. The policy is to scale down, never clip mid-word. Descriptions tolerate a gentler `minimumScaleFactor` than the bolder, tighter subtitles, and both are pinned to a single line so the grid row height never jumps:

```swift
import SwiftUI

struct StatCard: View {
    let title: String       // e.g. "Pressure"
    let subtitle: String    // 18pt bold, very tight — "1013 hPa"
    let description: String  // 13pt regular — "High pressure and falling fast."

    var body: some View {
        VStack(alignment: .leading, spacing: 4) {
            Text(title)
                .font(.subheadline.weight(.semibold))
                .lineLimit(1)
            Text(subtitle)
                .font(.system(size: 18, weight: .bold))
                .lineLimit(1)
                .minimumScaleFactor(0.8)
            Text(description)
                .font(.system(size: 13))
                .lineLimit(1)
                .minimumScaleFactor(0.9)
        }
        .padding(16)
        .frame(maxWidth: .infinity, alignment: .leading)
    }
}
```

The residual `OVER` rows in the full report are the ones this net catches: adjudicated as scale-covered and recorded so the next reader knows they were a decision. The scale factor is the floor of last resort, not a license to skip the budget — a card whose default text needs shrinking on the narrowest phone is still a finding.

## Results

- **Compact weekday rail:** at most three characters, letters and digits only, seven distinct labels per language, enforced across all 27 languages by one contract test. Six languages (Danish, German, Norwegian, Portuguese, Romanian, Thai) carry declared arrays with the ruling attached, because their CLDR abbreviations broke the budget or the distinctness rule.
- **Compact hours** pin English exactly and render native counters (`9時`, `9시`) for CJK instead of a bolted-on Latin meridiem; the 24-hour complication hour is a bare digit. Support answers "why does the German label look different?" with a committed array, not a guess.
- **Stat-card budgets** are derived from the 390-point grid formula, so a spacing change is a one-line edit and every budget follows. The legacy 375-point floor is informational only, and there is no separate English ceiling.
- **Two measurement tiers** — an always-on critical set on the device font, plus an env-gated full sweep that also pulls server-owned strings — mean a width regression fails a build instead of reaching a screenshot. Recalibrating the floor from 375 to 390 let a batch of over-shortened server phrases go back to their fuller wording, with the report proving each longer string now fit.

## Lessons Learned

- **Budget the narrowest slot you actually sell** — the narrowest hardware currently on the market, not the smallest that ever existed and not the one on your desk. Writing that one number down settles a hundred later arguments about whether a row "really" overflows.
- **Derive budgets from the layout, don't hand-pick them.** The stat budgets are grid arithmetic copied into the test, so they can never drift from the view; a hand-picked number rots the moment someone changes a spacing constant.
- **Count characters for contracts, measure ink for review.** A weekday rail is a hard discrete cap, so a font-independent character budget is exactly right; a stat card is continuous layout, so only rendered width on the real device font tells the truth.
- **Declare the exceptions as data.** German's weekday array and an accepted over-budget row are both data with a reason attached, which is what lets you override the platform or keep an unshrinkable string without the override quietly rotting — the test still proves the array fits, and the pinned value re-flags if it changes.
- **Make the generator fail when reality moves.** A committed report nobody notices has gone stale is worse than no report; a tripwire that fails the build when the stat-card set changes, and names the files to fix, is what keeps the artifact honest.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.
