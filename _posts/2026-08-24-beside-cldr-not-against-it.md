---
layout: post
title: "Beside CLDR, Not Against It"
date: 2026-08-24 09:10:00 -0600
summary: "How to compare an app's compact unit strings against ICU/CLDR's canonical forms without a pass/fail test fighting your product voice: emit one descriptive report per language, adjudicate every divergence in a dated header, and review the diff like a copy change."
tags: [swift, ios, localization, i18n, testing]
---

## The Problem

A deliberately glued Turkish speed abbreviation and a wrong Turkish speed abbreviation look identical in a diff. After one person translated [Hello Weather](https://helloweather.com) into 27 languages with a model and no agency, the app's compact unit strings and the Unicode CLDR disagreed everywhere, and nobody could say which disagreements were the house voice and which were mistakes. Which of those 27 languages each upstream data source could actually serve was a separate fight, covered in [Never Trust the Vendor's Language List](/probing-vendor-language-support/).

The house voice is dense on purpose. A wind reading is `12km/h`, glued, where most locales' CLDR form is `12 km/h`. Pressure is `1013hPa`, with none of the digit grouping CLDR adds (`1,013hPa` in English, `1.013 hPa` in German). A temperature is `72°` with no `C` or `F`, because the unit is a global setting the user already picked and repeating it on every reading is noise. Each of these was decided string by string, to fit stat cards and a watch complication with room for a couple of characters.

Apple hands you a correct localizer for free. `MeasurementFormatter` and `Duration.UnitsFormatStyle` render units the way CLDR says a given locale renders them: `12mph` in English but `12 mi/h` in German and `12 ми/ч` in Russian, `1.013 hPa` with a locale-grouped thousands separator in German, a non-breaking space between number and unit in Finnish and French, and the CLDR minus sign U+2212 before negative temperatures in Finnish, Swedish, and Norwegian. For most apps that canonical rendering is exactly what you want. The compact idiom walks away from it in dozens of small ways across 27 languages, and once you walk away from the platform's correct answer, nothing checks your answer anymore. You took on the localizer's job and lost the localizer's test.

The naive fix is a conformance test asserting your string equals the CLDR string. It fails on the first row, because you diverge on purpose. So you add an exception, then a hundred exceptions, and every time the product voice and CLDR disagree the build breaks and someone has to decide whether the app is wrong or the test is stale. A gate that fights the product on every run is a gate nobody trusts. The other naive fix is no test at all: eyeball English and ship, which is how a wrong decimal separator reaches a Russian user.

We wanted a third thing: our form beside CLDR's for every unit in every language, with a verdict on each row, that never blocks a build and never argues with a decision we already made.

## The Solution

A descriptive conformance report, not a pass/fail gate. `CLDRConformanceTests` is blessed as one snapshot file per language, 27 files. For every unit-bearing rendering the app shows, a row pairs our form against what ICU/CLDR renders for the same unit and locale, verdicted `MATCH` or `DIFFERS` by exact string equality. It is a review aid, not an assertion.

The rows group into seven sections that map to what the app renders: WIND, VISIBILITY, PRESSURE, PRECIP, TEMP, PERCENT, and DURATIONS. Here are the dispositions header and first section of the German file, as they land in the blessed snapshot (a short legend precedes the header):

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

Every `DIFFERS` there is intended, and the report's job is not to turn them green. Its job is that when a row you did *not* mean to change flips from `MATCH` to `DIFFERS`, a human sees it in a diff and asks why.

> A gate answers "is this allowed?" A report answers "is this what you meant?" For deliberate style divergence, the second question is the only useful one, and only a human can answer it.

Three design decisions make the report trustworthy.

### Replicate production composition, never read the app's singletons

The tempting shortcut is to call the app's own formatting helpers inside the test and print what they return. Do that and you are not comparing production rendering to CLDR. You are comparing your test host's configuration of those helpers to CLDR, and the two drift the instant a helper reads a setting the test host does not have.

So the report rebuilds the rendering from the same source data production uses. In this app the unit strings are server-composed: the backend glues a bare number to a localized unit symbol and a locale decimal separator, both read from the web repo's locale files. The report reproduces that gluing, renders the CLDR side with the platform formatters, and returns one verdicted row per unit. Everything below compiles against Foundation alone:

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

The `ours` cell is built by string composition and the `cldr` cell by the real Apple formatters, so the row measures production shape against the platform, not one helper against another. The row carries no verdict; the report computes it after the visualization step below, so hidden characters count.

### Make the invisible characters legible

The divergences that hurt most are the ones you cannot see. A non-breaking space (U+00A0, or its narrow cousin U+202F) is identical to a regular space in any diff viewer. The CLDR minus sign (U+2212) reads as a hyphen. If the report prints those glyphs raw, a reviewer scanning the diff sees `45 %` beside `45 %`, reads `MATCH`, and misses a hidden NBSP.

So every cell runs through a visualizer that turns confusable scalars into visible tokens, before the comparison and before printing. It depends on nothing but the standard library:

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

Now the Russian percent row reads honestly: `45%` versus `45⍽%`, and the `DIFFERS` verdict is plainly about that `⍽`. The Russian visibility row shows `5000м` versus `5⍽000 м`, surfacing the grouping separator and the space at once. Because equality is decided after this transform, the glyph difference is what the verdict is about.

### Adjudicate every deliberate divergence in the file itself

A report full of `DIFFERS` is worthless if a reviewer cannot tell an intended divergence from a regression. That is what the dated dispositions header at the top of every file is for, shown in the German excerpt above: each family of deliberate divergence, named, with its reason, signed and dated by the owner.

That list is the review contract. A `DIFFERS` row under a listed family is expected and needs no thought. A row that flips and is *not* covered by a disposition is the signal. The dispositions live in the generated snapshot, so they travel with the data and land in every diff instead of a wiki nobody opens. The last bullet is a genuine open question, adopt U+2212 or keep the hyphen-minus, parked in the file where it stays visible.

### How it runs

The report is a snapshot test, gated on an environment variable so it runs only when asked and never under CI. The snapshot helper compares against the committed file, records a missing one locally, refuses to record under CI, and rewrites the file when blessing. The human reading the diff is the check, not the assertion (the snapshot path is simplified here):

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

Notice the two guards on `reportsEnabled`: the report never runs unless `CLDR_REPORTS` is set, and it bails under CI so it never blocks a build. The verdict is recomputed on the visualized cells, so the invisible characters decide it. You regenerate the files with `bin/cldr-report`, a wrapper that converts the web repo's locale YAML to JSON in a temp directory, forwards it so the server-composed rows fill in, and delegates to the test runner. Without a web checkout those rows read `(web checkout absent)` rather than failing. Blessing an intended change is `bin/cldr-report --update-snapshots`, then reviewing the regenerated per-language diff exactly like a copy change. It is the same discipline as our on-device width report in [Measure the String Before You Translate It](/rendered-width-validation/): the report is not the gate, the human reading the diff is.

## Results

- **27 blessed snapshot files**, one per language, seven unit families, roughly two dozen verdicted rows each. The steady state is no unexplained divergence: every remaining `DIFFERS` is either fixed or adjudicated in the header.
- **Five languages' speed abbreviations were wrong**, not house voice: Turkish, Danish, Dutch, Indonesian, and Norwegian. A Turkish user flagged `km/h` the day after the localization launch; the table agreed (`km/sa`) and showed the other four. A gate would have buried them under exceptions. Beside CLDR they were legible as errors, and we adopted CLDR's abbreviations for all five.
- **A run of server fixes followed** once divergences sat side by side: Scandinavian miles moved to CLDR forms, wind speed picked up CLDR symbols for km/h and m/s, composed decimals switched from a hard-coded dot to the locale separator, and Cyrillic and Greek unit symbols were localized.
- **Zero CI cost, zero false alarms.** Descriptive and env-gated, it never runs on CI, never flakes, and never asks an engineer to relitigate a product decision. The trade-off is that nothing fails on its own: a regression surfaces only when someone regenerates the report and reads the diff, which is why the repo's agent instructions make `bin/cldr-report` part of the done gate for any unit-string change.

## Lessons Learned

- **A gate fits one right answer; a report fits a style decision.** A pass/fail test encodes a single correct string. A product that deliberately diverges from the platform default has none to encode, so the gate becomes a fight.
- **Compute the verdict on what the reviewer sees.** If confusable characters are made visible only at print time, the verdict and the diff can disagree. Transform first, then compare.
- **Park open questions in the artifact, not a wiki.** The U+2212 decision sits in the dispositions header, where every diff shows it, instead of in a document nobody opens.
- **A comparison you can see finds bugs on both sides of it.** Putting client strings beside CLDR corrected five client abbreviations and surfaced a half-dozen server fixes the report was never aimed at.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the two indistinguishable Turkish abbreviations instead of on the app, each design rule is stated once in its section rather than again as a bolded closer and again in Lessons Learned, the vendor-language cross-link moved from Lessons Learned into the body, and the title dropped its subtitle. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The CLDR examples were corrected against the blessed snapshots: English CLDR renders `12mph` and `1,013hPa` (the spaced `12 mi/h` is German), the NBSP and U+2212 renderings are named to the locales that actually produce them, and the Russian visibility cell reads `5000м`. The first code block dropped an invented `verdict` property and now uses the catalog's `%lld` placeholder and the real `miles_long` row label; the snapshot helper was rewritten to the real semantics (`UPDATE_SNAPSHOTS == "1"`, records missing files locally, refuses under CI, reports drift via `Issue.record`) under the real `CLDRConformanceTests` name. The five-abbreviation result now notes the Turkish user report that preceded the table's confirmation, and the trade-off sentence notes that the iOS agent instructions require `bin/cldr-report` for unit-string changes.
