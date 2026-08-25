---
layout: post
title: "Beside CLDR, Not Against It: A Unit-String Conformance Report That Is Deliberately Not a Gate"
date: 2026-08-24 09:10:00 -0600
summary: "How to compare an app's compact unit strings against ICU/CLDR's canonical forms without a pass/fail test fighting your product voice: emit one descriptive report per language, adjudicate every divergence in a dated header, and review the diff like a copy change."
tags: [swift, ios, localization, i18n, testing]
---

## The Problem

[Hello Weather](https://helloweather.com) carries far more text than a typical weather app, most of it dense on purpose. A wind reading is `12mph`, not `12 mi/h`. Pressure is `1013hPa`, not `1.013 hPa`. A temperature is `72°` with no `C` or `F`, because the unit is a global setting the user already picked and repeating it on every reading is noise. These are deliberate decisions, made string by string, drawn to fit stat cards and a watch complication that has room for a couple of characters.

Apple hands you a correct localizer for free. `MeasurementFormatter` and `Duration.UnitsFormatStyle` render units the way the Unicode CLDR says a given locale renders them: `12 mi/h` in English, `12 ми/ч` in Russian, `1.013 hPa` with a locale-grouped thousands separator in German, a non-breaking space between number and unit, and the CLDR minus sign U+2212 before negative temperatures. For most apps that canonical rendering is exactly what you want. The compact idiom above walks away from it in dozens of small ways across 27 languages.

That is where the trouble starts. When one person translates an app into 27 languages with a model and no agency, the compact forms and CLDR disagree everywhere, and after the fills land nobody can say which disagreements are the house voice and which are mistakes. A glued Turkish speed abbreviation and a wrong Turkish speed abbreviation look identical in a diff. **You have deliberately walked away from the platform's correct answer, so nothing checks your answer anymore.** You took on the localizer's job and lost the localizer's test.

The naive fix is a conformance test that asserts your string equals the CLDR string. It fails on the first row, because you diverge on purpose, so you add an exception, then a hundred exceptions, and every time the product voice and CLDR disagree the build breaks and someone must decide whether the app is wrong or the test is stale. A gate that fights the product on every run is a gate nobody trusts. The other naive fix is no test at all, eyeball English, and ship, which is how a wrong decimal separator reaches a Russian user.

We wanted the third thing: a way to see our form beside CLDR's for every unit in every language, all at once, with a verdict on each row, that never blocks a build and never argues with a decision we already made.

## The Solution

The pattern is a **descriptive conformance report, not a pass/fail gate.** For Hello Weather this is `CLDRConformanceTests`, blessed as one snapshot file per language (27 files). For every unit-bearing rendering the app shows, a row pairs OUR form against what ICU/CLDR renders for the same unit and locale, verdicted `MATCH` or `DIFFERS` by exact string equality. It is a review aid, not an assertion.

The rows group into seven sections that map to what the app renders: WIND, VISIBILITY, PRESSURE, PRECIP, TEMP, PERCENT, and DURATIONS. Here is the top of the German file, header and first section as they land in the blessed snapshot:

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

Every `DIFFERS` there is intended, and the report's job is not to turn them green. Its job is so that when a row you did *not* mean to change flips from `MATCH` to `DIFFERS`, a human sees it in a diff and asks why.

> A gate answers "is this allowed?" A report answers "is this what you meant?" For deliberate style divergence, the second question is the only useful one, and only a human can answer it.

Three design decisions make the report trustworthy. Each is a transferable rule.

### Replicate production composition, never read the app's singletons

The most tempting shortcut is the most wrong one: call the app's own formatting helpers inside the test and print what they return. Do that and you are not comparing your production rendering to CLDR. You are comparing your test-host configuration of those helpers to CLDR, and the two drift the instant a helper reads a setting the test host does not have.

So the report reconstructs the rendering from the same source data production uses. In this app the unit strings are server-composed: the backend glues a bare number to a localized unit symbol and a locale decimal separator, both read from the web repo's locale files. The report reproduces that gluing, then renders the CLDR side with the platform formatters, and returns one verdicted row per unit. Everything below compiles against Foundation alone:

```swift
import Foundation

struct ConformanceRow {
    let label: String
    let ours: String   // reconstructed from source data, never read from the app
    let cldr: String   // what ICU/CLDR produces for the same unit + locale
    var verdict: String { ours == cldr ? "MATCH" : "DIFFERS" }
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
    let hourAbbrev = "%dh"       // e.g. "3ч" for ru

    return [
        ConformanceRow(label: "speed km/h",
                       ours: "12" + speedSymbol,   // glued, no space: the compact idiom
                       cldr: measured(12, UnitSpeed.kilometersPerHour, .short)),
        ConformanceRow(label: "visibility mi (long)",
                       ours: milesLong,
                       cldr: measured(10, UnitLength.miles, .long)),
        ConformanceRow(label: "3h (abbrev)",
                       ours: hourAbbrev.replacingOccurrences(of: "%d", with: "3"),
                       cldr: duration(3 * 3600, [.hours], .abbreviated)),
    ]
}
```

Notice that the `ours` cell is built by string composition and the `cldr` cell by the real Apple formatters, so the row measures production shape against the platform, not one helper against another. The `verdict` here is raw equality; the full report recomputes it *after* the visualization step below, so hidden characters count. The rule generalizes past this codebase: **if a test reads live app state, it tests the test's environment; to compare rendering, rebuild the rendering from the same source data the real path uses.**

### Make the invisible characters legible

The divergences that hurt most are the ones you cannot see. A non-breaking space (U+00A0, or its narrow cousin U+202F) is identical to a regular space in any diff viewer. The CLDR minus sign (U+2212) reads as a hyphen. If the report prints those glyphs raw, a reviewer scanning the diff sees `45 %` beside `45 %`, reads `MATCH`, and misses a hidden NBSP.

So every cell runs through a visualizer that turns confusable scalars into visible tokens before the comparison and before printing. It depends on nothing but the standard library:

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

Now the Russian percent row reads honestly: `45%` versus `45⍽%`, and the `DIFFERS` verdict is plainly about that `⍽`. The Russian visibility row shows `5000m` versus `5⍽000 м`, surfacing the grouping separator and the space at once. Because exact equality decides the verdict *after* this transform, the glyph difference is what the verdict is about. The rule: **when a test compares strings that can hold invisible or confusable characters, turn them into visible tokens before you compare and before you print, or the diff will lie to you.**

### Adjudicate every deliberate divergence in the file itself

A report full of `DIFFERS` is worthless if a reviewer cannot tell an intended divergence from a regression. That is what the dated dispositions header at the top of every file is for, shown in the German excerpt above: each family of deliberate divergence, named, with its reason, signed and dated by the owner.

That list is the review contract. A `DIFFERS` row under a listed family is expected and needs no thought; a row that flips and is *not* covered by a disposition is the signal. The dispositions live in the generated snapshot, so they travel with the data and land in every diff instead of a wiki nobody opens. The last bullet is a genuine open question — adopt U+2212 or keep the hyphen-minus — parked in the file where it stays visible. The rule: **for a report whose whole point is expected divergence, the expected divergences must be enumerated and dated inside the artifact, or the report degrades into noise.**

### How it runs

The report is a snapshot test, gated on an environment variable so it runs only when asked and never under CI. The snapshot helper writes the file when blessing and compares against it otherwise; the human reading the diff is the check, not the assertion:

```swift
import Foundation
import Testing

enum Language: String, CaseIterable {
    case en, de, ru, ja   // one case per supported language
}

func assertMatchesSnapshot(_ report: String, named name: String,
                           file: StaticString = #filePath) throws {
    let url = URL(fileURLWithPath: "\(file)")
        .deletingLastPathComponent()
        .appendingPathComponent("snapshots/\(name).snap.txt")
    if ProcessInfo.processInfo.environment["UPDATE_SNAPSHOTS"] != nil {
        try FileManager.default.createDirectory(
            at: url.deletingLastPathComponent(), withIntermediateDirectories: true)
        try report.write(to: url, atomically: true, encoding: .utf8)
        return
    }
    let expected = (try? String(contentsOf: url, encoding: .utf8)) ?? ""
    #expect(report == expected, "snapshot \(name) differs — review the diff, then bless it")
}

@MainActor
struct ConformanceReportTests {
    nonisolated static let reportsEnabled =
        ProcessInfo.processInfo.environment["CLDR_REPORTS"] != nil
        && ProcessInfo.processInfo.environment["CI"] == nil

    @Test(.enabled(if: ConformanceReportTests.reportsEnabled))
    func conformanceTable() throws {
        for language in Language.allCases {
            let body = conformanceRows(language: language.rawValue).map { row -> String in
                let ours = visualize(row.ours)
                let cldr = visualize(row.cldr)
                return "| \(row.label) | \(ours) | \(cldr) | \(ours == cldr ? "MATCH" : "DIFFERS") |"
            }.joined(separator: "\n")
            try assertMatchesSnapshot(body, named: language.rawValue)
        }
    }
}
```

Notice the two guards on `reportsEnabled`: the report never runs unless `CLDR_REPORTS` is set, and it bails under CI so it never blocks a build, and the verdict is recomputed on the visualized cells so the invisible characters decide it. You regenerate the files with `bin/cldr-report`, a wrapper that converts the web repo's locale YAML to JSON in a temp directory, forwards it so the server-composed rows fill in, and delegates to the test runner; without a web checkout those rows read `(web checkout absent)` rather than failing. Blessing an intended change is `bin/cldr-report --update-snapshots`, then reviewing the regenerated per-language diff exactly like a copy change. It is the same discipline as our on-device width report in [Measuring Strings Before You Translate Them](/rendered-width-validation/): the report is not the gate, the human reading the diff is.

## Results

- **27 blessed snapshot files**, one per language, covering seven unit families with roughly two dozen verdicted rows each. The steady state is "no unexplained divergence": every remaining `DIFFERS` is either fixed or adjudicated in a header.
- **It caught mistakes a gate would have buried.** Five languages' speed abbreviations — Turkish, Danish, Dutch, Indonesian, and Norwegian — turned out to be wrong hand-rolled forms rather than deliberate house voice, invisible until they sat beside CLDR. We adopted CLDR's abbreviations for all five.
- **The same lens drove a run of server fixes** once the divergences were visible side by side: Scandinavian miles moved to CLDR forms, wind speed picked up CLDR symbols for km/h and m/s, composed decimals switched from a hard-coded dot to the locale separator, and Cyrillic and Greek unit symbols were localized. Before the report, none of these were legible as errors.
- **Zero CI cost, zero false alarms.** Because it is descriptive and env-gated, it never runs on CI, never flakes, and never asks an engineer to relitigate a decision the product already made.

## Lessons Learned

- **A descriptive report beats a gate when the "correct" answer is a style decision.** A pass/fail test encodes one right answer; a product that deliberately diverges from the platform default has no single right answer to encode, so the gate becomes a fight. Emit the comparison, verdict each row, and let a human read the diff.
- **Replicate production composition, or you test the wrong thing.** Calling your app's own formatters inside a test measures the test host's configuration, not production, so rebuild the rendering from the same source data the real path uses and the comparison stays honest.
- **Make invisible characters visible before you compare.** NBSP, narrow NBSP, and the CLDR minus sign are the divergences that matter most and the ones a diff viewer hides, so transform confusable scalars into printable tokens first and the verdict is about what you can actually see.
- **Adjudicate divergences inside the artifact, dated.** A report of expected differences is noise unless those differences are enumerated where the diff will show them, so put the dispositions in the generated file with an owner, a date, and the open questions, and review becomes a scan instead of an investigation.
- **A conformance lens pays off across repos.** The same beside-CLDR comparison that corrected five languages' abbreviations on the client surfaced a half-dozen server-side fixes, because a divergence you cannot see is a bug you cannot find. Related client-side language work is in [Probing Vendor Language Support](/probing-vendor-language-support/).

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.
