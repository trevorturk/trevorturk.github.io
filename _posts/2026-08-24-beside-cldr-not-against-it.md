---
layout: post
title: "Beside CLDR, Not Against It"
date: 2026-08-24 09:10:00 -0600
summary: "We compare our compact unit strings with what CLDR says each language does, without a pass/fail test that fights our own style: one report per language, a dated header listing the differences we chose, and a person reading the diff like a copy change."
tags: [swift, ios, localization, i18n, testing]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

A Turkish speed abbreviation we glued to the number on purpose and a Turkish speed abbreviation that's wrong look the same in a diff. One person translated [Hello Weather](https://helloweather.com) into 27 languages with a model and no agency. After that, our compact unit strings disagreed all over the place with CLDR, the Unicode locale data that says how each language writes units. Nobody could say which disagreements were our style and which were mistakes. Whether each upstream data source could serve those 27 languages was a separate fight, covered in [Never Trust the Vendor's Language List](/probing-vendor-language-support/).

Our style is dense on purpose. A wind reading is `12km/h`, with no space, where most languages' CLDR form is `12 km/h`. Pressure is `1013hPa`, without the digit grouping CLDR adds (`1,013hPa` in English, `1.013 hPa` in German). A temperature is `72°` with no `C` or `F`, because the user already picked the unit in settings and repeating it on every reading is noise. We decided each of these string by string, to fit stat cards and a watch complication with room for a couple of characters.

Apple already ships a correct localizer. `MeasurementFormatter` and `Duration.UnitsFormatStyle` render units the way CLDR says each locale does. English gets `12mph`, German `12 mi/h`, Russian `12 ми/ч`. German groups the thousands: `1.013 hPa`. Finnish and French put a non-breaking space between the number and the unit. Finnish, Swedish, and Norwegian put the CLDR minus sign, U+2212, before a negative temperature. For most apps that's what you want. Our compact style walks away from it in dozens of small ways across 27 languages, and once we walked away from the platform's answer, nothing checked ours. We had taken over the localizer's job without its tests.

The obvious fix is a test that asserts our string equals the CLDR string. It fails on the first row, because we differ on purpose. So we'd add an exception, then a hundred, and every time our style and CLDR disagree the build breaks and someone has to decide whether the app is wrong or the test is stale. Nobody trusts a test that fights the product on every run. The other obvious fix is no test at all: check English by eye and ship, which is how a wrong decimal separator reaches a Russian user.

We wanted a third thing: our form next to CLDR's for every unit in every language, with a verdict on each row, that never blocks a build and never argues with a decision we already made.

## The Solution

We wrote a report, not a pass/fail test. `CLDRConformanceTests` writes one snapshot file per language, 27 files, and we commit them. Each row pairs one of our unit strings with what ICU/CLDR renders for the same unit and locale, and marks it `MATCH` or `DIFFERS` by exact string equality. It's something to read, not something that fails.

The rows sit in seven sections, one per kind of thing the app shows: WIND, VISIBILITY, PRESSURE, PRECIP, TEMP, PERCENT, and DURATIONS. At the top of each file is a header listing the families of differences we chose on purpose; the file calls them dispositions. Here are that header and the first section of the German file, as committed (a short legend comes before the header):

```
Dispositions (Trevor, 2026-08-07) — deliberate DIFFERS families, all intended:
- glued number+symbol (compactness idiom); only pressure inHg/kPa keep a space.
- percent strips the whitespace CLDR inserts (toProbability).
- bare unit-agnostic degree (no °C/°F suffix).
- compact durations run tighter than CLDR's abbreviations.
- server numbers carry no digit grouping.
- negative temps keep the hyphen-minus (CLDR uses it in 24/27 of our languages;
  the U+2212 adoption question is with design).

## WIND

| Label                      | Ours       | CLDR            | Verdict |
|----------------------------|------------|-----------------|---------|
| speed mph                  | 12mph      | 12 mi/h         | DIFFERS |
| speed km/h                 | 12km/h     | 12 km/h         | DIFFERS |
| speed m/s                  | 5m/s       | 5 m/s           | DIFFERS |
| gust range mph (endpoints) | 12-18mph   | 12 mi/h–18 mi/h | DIFFERS |
```

Every `DIFFERS` there is intended, and the report's job isn't to turn them green. Its job is to make a row we didn't mean to change show up in a diff when it flips from `MATCH` to `DIFFERS`, so a person asks why.

> A gate answers "is this allowed?" A report answers "is this what you meant?" For deliberate style divergence, the second question is the only useful one, and only a human can answer it.

Three decisions make the report worth trusting.

### Rebuild the string from source data, don't call the app's helpers

The tempting shortcut is to call the app's own formatting helpers inside the test and print what they return. Then you're not comparing what production renders with CLDR. You're comparing how the test host configured those helpers with CLDR, and the two drift as soon as a helper reads a setting the test host doesn't have.

So the report rebuilds each string from the same source data production uses. In this app the server composes the unit strings: it glues a bare number to a localized unit symbol and a decimal separator, both read from the web repo's locale files. The report does the same gluing, renders the CLDR side with Apple's formatters, and returns one row per unit. Everything below compiles against Foundation alone:

```swift
import Foundation

struct ConformanceRow {
    let label: String
    let ours: String   // reconstructed from source data, never read from the app
    let cldr: String   // what ICU/CLDR produces for the same unit + locale
}

func conformanceRows(language: String) -> [ConformanceRow] {
    let locale = Locale(identifier: language)

    func measured(_ value: Double, _ unit: Dimension,
                  _ style: MeasurementFormatter.UnitStyle) -> String {
        let formatter = MeasurementFormatter()
        formatter.locale = locale
        formatter.unitStyle = style
        formatter.unitOptions = .providedUnit
        return formatter.string(from: Measurement(value: value, unit: unit))
    }

    func duration(_ seconds: Int, _ allowed: Set<Duration.UnitsFormatStyle.Unit>,
                  _ width: Duration.UnitsFormatStyle.UnitWidth) -> String {
        Duration.seconds(seconds).formatted(.units(allowed: allowed, width: width).locale(locale))
    }

    // Stand-ins for the per-language source data the real report reads from the
    // web locale JSON (unit symbols) and the string catalog (duration templates).
    let speedSymbol = "km/h"     // e.g. "км/ч" for ru
    let milesLong = "10 miles"   // plural form pulled from the locale file
    let hourAbbrev = "%lldh"     // catalog key "%lldh"; e.g. "3ч" for ru

    return [
        ConformanceRow(label: "speed km/h",
                       ours: "12" + speedSymbol,   // glued, no space: the compact idiom
                       cldr: measured(12, UnitSpeed.kilometersPerHour, .short)),
        ConformanceRow(label: "miles_long plural (10)",
                       ours: milesLong,
                       cldr: measured(10, UnitLength.miles, .long)),
        ConformanceRow(label: "3h (abbrev)",
                       ours: hourAbbrev.replacingOccurrences(of: "%lld", with: "3"),
                       cldr: duration(3 * 3600, [.hours], .abbreviated)),
    ]
}
```

The `ours` cell comes from string gluing and the `cldr` cell from the real Apple formatters, so the row compares what production shows with the platform, not one helper with another. The row has no verdict yet. The report computes it after the step below, so hidden characters count.

### Make the invisible characters visible

The differences that hurt most are the ones you can't see. A non-breaking space (U+00A0, or the narrow one, U+202F) looks like a regular space in any diff viewer. The CLDR minus sign (U+2212) looks like a hyphen. If the report printed those characters raw, a reviewer would see `45 %` next to `45 %`, read it as a match, and miss the hidden non-breaking space.

So every cell goes through a function that swaps those characters for visible marks, before the comparison and before printing. It needs nothing but the standard library:

```swift
func visualize(_ string: String) -> String {
    string.unicodeScalars.map { scalar -> String in
        switch scalar.value {
        case 0x00A0, 0x202F: return "⍽"          // NBSP and narrow NBSP
        case 0x2212: return "\\u{2212}"          // CLDR minus sign
        default: return String(scalar)
        }
    }.joined()
}
```

Now the Russian percent row reads `45%` next to `45⍽%`, and the `DIFFERS` verdict is plainly about that `⍽`. The Russian visibility row reads `5000м` next to `5⍽000 м`, which shows the grouping separator and the space at once. The verdict is computed after this swap, so it's about the characters the reviewer can see.

### List every deliberate difference in the file itself

A report full of `DIFFERS` is useless if a reviewer can't tell an intended difference from a regression. The dispositions header at the top of every file, shown in the German excerpt above, is there for that: each family of deliberate difference, named, with its reason, signed and dated by the owner.

That list is the rule for review. A `DIFFERS` row under a listed family is expected and needs no thought. A row that flips and isn't covered by the list is the signal. The list lives in the generated file, so it lands in every diff instead of in a wiki nobody opens. The last bullet is an open question, adopt U+2212 or keep the hyphen-minus, and it sits in the file where it stays visible.

### How it runs

The report is a snapshot test behind an environment variable, so it runs only when asked and never under CI. The snapshot helper compares the output with the committed file. If the file is missing, it writes one locally but refuses to under CI. With `UPDATE_SNAPSHOTS=1` it rewrites the file. The person reading the diff is the check, not the assertion (the snapshot path is simplified here):

```swift
import Foundation
import Testing

enum Language: String, CaseIterable {
    case en, de, ru, ja   // one case per supported language
}

func assertMatchesSnapshot(_ value: String, named name: String,
                           filePath: String = #filePath,
                           sourceLocation: SourceLocation = #_sourceLocation) {
    let environment = ProcessInfo.processInfo.environment
    let updating = environment["UPDATE_SNAPSHOTS"] == "1"
    let locked = environment["CI"] != nil
    let url = URL(fileURLWithPath: filePath)
        .deletingLastPathComponent()
        .appendingPathComponent("snapshots/cldr_conformance_table__\(name).snap.txt")

    if !updating, let recorded = try? String(contentsOf: url, encoding: .utf8) {
        if recorded == value { return }
        Issue.record("output drifted from \(name) — regenerate with --update-snapshots, then review every changed line",
                     sourceLocation: sourceLocation)
        return
    }
    guard !locked else {
        Issue.record("snapshot \(name) is missing, but snapshots are locked under CI — record locally and commit the file",
                     sourceLocation: sourceLocation)
        return
    }
    try? FileManager.default.createDirectory(
        at: url.deletingLastPathComponent(), withIntermediateDirectories: true)
    try? value.write(to: url, atomically: true, encoding: .utf8)
}

@MainActor
struct CLDRConformanceTests {
    nonisolated static let reportsEnabled =
        ProcessInfo.processInfo.environment["CLDR_REPORTS"] != nil
        && ProcessInfo.processInfo.environment["CI"] == nil

    @Test(.enabled(if: CLDRConformanceTests.reportsEnabled))
    func cldrConformanceTable() throws {
        for language in Language.allCases {
            let body = conformanceRows(language: language.rawValue).map { row -> String in
                let ours = visualize(row.ours)
                let cldr = visualize(row.cldr)
                return "| \(row.label) | \(ours) | \(cldr) | \(ours == cldr ? "MATCH" : "DIFFERS") |"
            }.joined(separator: "\n")
            assertMatchesSnapshot(body, named: language.rawValue)
        }
    }
}
```

Notice the two guards on `reportsEnabled`: the report doesn't run unless `CLDR_REPORTS` is set, and it stays off under CI so it can't block a build. The verdict is computed on the swapped cells, so the invisible characters decide it. You regenerate the files with `bin/cldr-report`. That wrapper converts the web repo's locale YAML to JSON in a temp directory, passes it along so the server-composed rows fill in, and hands off to the test runner. Without a web checkout those rows read `(web checkout absent)` instead of failing. To accept an intended change, you run `bin/cldr-report --update-snapshots` and review the regenerated diff for each language the way you'd review a copy change. It's the same habit as our on-device width report in [Measure the String Before You Translate It](/rendered-width-validation/): the report isn't the gate; the person reading the diff is.

## Results

- **27 committed snapshot files**, one per language, seven unit families, roughly two dozen rows each. The goal is no unexplained difference: every remaining `DIFFERS` is either fixed or listed in the header.
- **Five languages' speed abbreviations were wrong**, not our style: Turkish, Danish, Dutch, Indonesian, and Norwegian. A Turkish user flagged `km/h` the day after the localization launch. The report agreed (`km/sa`) and showed the other four. A pass/fail test would have buried them under exceptions. Next to CLDR they read as errors, and we adopted CLDR's abbreviations for all five.
- **A run of server fixes followed** once the differences sat side by side. Scandinavian miles moved to CLDR forms. Wind speed took CLDR's symbols for km/h and m/s. Composed decimals switched from a hard-coded dot to the locale separator. Cyrillic and Greek unit symbols were localized.
- **No CI cost and no false alarms.** The report doesn't run on CI, doesn't flake, and doesn't ask an engineer to reargue a product decision. The trade-off is that nothing fails on its own. A regression shows up only when someone regenerates the report and reads the diff, so the repo's agent instructions require `bin/cldr-report` before any unit-string change counts as done.

## Lessons Learned

- **Use a pass/fail test when there's one right answer, and a report when there's a style decision.** A product that differs from the platform on purpose has no single correct string to assert, so the test becomes a fight.
- **Compute the verdict on what the reviewer sees.** If you make hidden characters visible only when printing, the verdict and the diff can disagree. Swap first, then compare.
- **Keep open questions in the generated file, not a wiki.** The U+2212 decision sits in the dispositions header, where every diff shows it, instead of in a document nobody opens.
- **A side-by-side comparison finds bugs on both sides.** Putting our strings next to CLDR fixed five client abbreviations and turned up a half-dozen server fixes the report wasn't aimed at.
