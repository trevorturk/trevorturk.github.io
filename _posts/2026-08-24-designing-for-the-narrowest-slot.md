---
layout: post
title: "Designing for the Narrowest Slot"
date: 2026-08-24 09:20:00 -0600
summary: "The smallest slots in a weather app hold three letters. We cap compact date labels by character count, work the stat-card budget out from the grid math, and test both in 27 languages by measuring what the phone actually draws."
tags: [swift, ios, localization, i18n, tooling]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

The weekday rail under a chart holds seven ticks, and each tick holds three characters. `Mon Tue Wed` fits. So do `9am 10am` under the bars of the hourly chart, `Falling fast` on a two-line stat card, and a bare hour on a watch complication. We drew every one of these slots in English, they look right in English, and none of them wraps or grows.

Then a model translates the strings into 26 more languages, and nothing in our tools knows how wide any of them will be. A German label one character too wide clips in the app, the widget, and the watch, because it's the same string in the same slot. Some languages are worse: two weekdays shrink to the same three letters, so the axis shows two identical ticks. That's a data bug, and nothing looks cut off. Six of our languages fail one of those two checks if we take the system calendar's abbreviations as they come. Without a test, the first we'd hear of it is a screenshot or a customer, after shipping.

[Hello Weather](https://helloweather.com) shows sentences where most weather apps show a number and an icon: pressure trends, when the rain starts, air-quality advice, in 27 languages. Most of that text sits in slots like these. One person runs it, with no localization agency, so the fix has to fail a build. Nobody is going to eyeball 27 screenshots. We use two measures. Dates count characters, because a weekday tick has a hard cap. Stat cards measure rendered width, because a card is a layout and what overflows it is the width of the glyphs.

## The Solution

Every date string goes through one rulebook, so we can list the slots that are tight and test each one against a character cap. Stat-card budgets come from the same grid formula the view uses, and tests measure rendered width against them. Neither number was picked by hand. Exceptions in both are written down as data with their exact value, so a string somebody decided to keep is flagged again the moment it changes.

### Compact date labels get a character budget

Every date string goes through one enum of intents. Each case names a slot in the UI (`hourlyChartWeekday`, `statsChartHour`, `complicationHour`), not a format. The [date rulebook post](/date-format-rulebook/) covers that enum. For this post, the enum matters because a test can loop over the cases drawn in a tight slot and check a hard cap on each of them, in all 27 languages.

The compact weekday rail has the tightest rule. Each label is at most three characters and three Unicode scalars, letters and digits only, and the seven labels in a week must all differ. Most languages pass with the system calendar's abbreviated or short standalone symbols. For the six that don't, the resolver returns a seven-symbol array we wrote by hand, with a comment saying why. Here's the resolver and the test that checks it, self-contained:

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

The exception is data, not a code path. The array can be reviewed, the same test proves it fits, and the next person can see why German stopped reading its symbols from the calendar. Overriding the platform's locale data feels wrong at first. But that data was chosen for reading in a paragraph, not for a three-character tick that has to look different from its six neighbors. When the platform was solving a different problem than yours, override it with your own data and write down the reason.

Hour labels get a cap of five characters, and English is pinned to `12am…11pm` and `0:00…23:00`. The standard locale data (CLDR) would give `4 PM` or `4 p.m.`. We write `4pm`, because the labels sit under the bars of the hourly chart and can't be wider than the bars. The rulebook gets that form by setting bare lowercase am/pm symbols when it builds the formatter, not by lowercasing the output afterwards. Chinese, Japanese, and Korean use their own hour counter (`9時`, `9시`) instead of am/pm. That's how those languages write a short clock hour, and a Latin `am` on a CJK digit is wrong and wider. The watch complication in 24-hour mode is tighter still: a bare `17`, no colon, no am/pm. Each of these is pinned in a test. The English anchors were typed by a person and are never updated by a script, so a careless snapshot re-record can't quietly approve a regression.

### Stat cards measure rendered width

A weekday tick has a fixed character count, so counting characters is the right check. A stat card is a two-line cell in a grid that adapts to the screen, and what overflows it is the rendered width of the text. A Cyrillic string and a Latin string with the same character count are different widths. An 18-point bold subtitle and a 13-point regular description can't share a budget either. So the stat tests measure each string on the real device font. The budget they measure against comes from the same grid arithmetic the SwiftUI view uses. Change the layout and the budget follows, in one line:

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

The 390 is the narrowest iPhone still on sale. The file also keeps the older 375-point width, but only for information. The report shows what would clip on older phones, it never fails a test, and it's never a reason to shorten copy. There's no separate budget for English either. An English string wider than the grid is a finding, not a new budget invented so it passes. `descriptionBudget` here works out to 142 points, and the `MARGIN` band 5% under it means the string is worth a look on a device.

### The always-on gate pins the known-tight strings

That helper runs at two levels. The always-on level is small and fast. It fails `bin/unit-test` if any of a hand-picked set of the tightest known lines goes over: the widest pressure trend per language, the timed precip strings, and a few fixed one-offs. It measures on the device font every run:

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

The second level is a full sweep, turned on by an environment variable so it never runs in CI. It grades every stat-card slot in all 27 languages, against both the catalog values and the live strings from our server. Many of the widest strings are built on the server and come back already localized. A wrapper script converts the web repo's locale files to JSON so the report can read them. The output is a markdown report that's committed and reviewed on every change, like a copy edit. A row that's over budget on purpose goes in an `accepted` map, pinned by its exact value, so an accepted string is flagged again the moment it changes. "This canonical term has no shorter natural form" is a fine reason to accept a row. "We stopped looking" is not. With the value pinned, an accepted string that changes has to be accepted again, with a reason.

A third test ties the two levels together. It fails the build when the set of stat cards changes, and names the files to update. Without it, someone could add a card that the always-on test keeps passing while the report quietly stops covering it.

### The runtime policy is shrink, never truncate

Budgets and tests keep copy inside the slot when it's written. The view still needs a safety net when it draws, for the string that slipped through or the accessibility text size that makes everything bigger. The policy is to scale the text down and never cut it off mid-word. Descriptions may shrink a little more than the bold, tighter subtitles, and both stay on one line so the grid's row height never jumps:

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

The `OVER` rows left in the full report are the ones this net catches. Each was looked at, judged covered by the scale factor, and recorded, so the next reader knows it was a decision. The scale factor is the last resort, not a reason to skip the budget. A card whose default text has to shrink on the narrowest phone is still a finding.

## Results

- One test checks the weekday rule in all 27 languages. Six (Danish, German, Norwegian, Portuguese, Romanian, Thai) use hand-written arrays with the reason attached, so "why does the German label look different?" is answered by a committed array, not a guess.
- A width regression fails `bin/unit-test` instead of reaching a screenshot. The env-gated sweep covers server-owned strings too, at the cost of a committed report to re-approve on every copy change.
- Moving the device width from 375 to 390 let a batch of over-shortened server phrases go back to their fuller wording, and the report showed each longer string fit.
- The over-budget rows that remain are covered by `minimumScaleFactor` rather than shorter copy, and each one is recorded as a decision.

## Lessons Learned

- **Budget for the narrowest device you actually sell.** Not the smallest one that ever existed, and not the one on your desk. Writing the number down settles later arguments about whether a row "really" overflows.
- **Work the budget out from the layout, don't pick it by hand.** A number computed from the grid arithmetic can't drift from the view. A hand-picked one goes stale when someone changes a spacing constant.
- **Pin every exception by exact value.** An override of platform data or an accepted over-budget string is fine as data with a reason attached. It's flagged again the moment the value changes.
- **Fail the build when the covered set changes.** A report that goes stale without anyone noticing is worse than no report.
