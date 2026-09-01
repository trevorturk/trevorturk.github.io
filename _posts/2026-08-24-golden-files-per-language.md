---
layout: post
title: "Golden Files Per Language"
date: 2026-08-24 08:20:00 -0600
summary: "One committed golden file per language turns a localized rendering matrix into a reviewable diff, and a six-second macOS test tier reads the same files without ever being allowed to rewrite them."
tags: [swift, ios, testing, snapshot-testing, localization]
---

## The Problem

A Japanese hour that drops its counter symbol does not crash anything. The app builds, the English screens look right, and the regression ships to a screen in a language nobody on the team reads. A localized date renderer fails that way: one language at a time, silently. [Hello Weather](https://helloweather.com) renders dates on 24 surfaces, in 27 languages, on both 12- and 24-hour clocks, maintained by one person with a model doing the translations. That is a lot of ways to fail quietly.

The obvious test asserts the English output and stops there. It passes forever while every other language drifts, because English is the one case that matches the source strings.

The opposite instinct fails too. Snapshot every language against every intent against every clock and you get one enormous file. A real one-language change buries itself in a regeneration that rewrites the whole thing. Nobody reads a seventy-thousand-line diff; they approve it and move on, which is the same as having no test.

The date path here is a rulebook: every user-visible date string flows through one enum of intents, each mapped to a render plan by one exhaustive switch (the design is its own [post](/date-format-rulebook/)). A rulebook is only trustworthy if a change to it produces a diff a human can read. This post covers the snapshot layer that makes that true, plus a second test tier that runs the same checks in six seconds without being allowed to corrupt them. It picks up where [porting the snapshot ergonomics](/port-the-ergonomics-not-the-library/) left off: the same drop-in file-snapshot helper, pointed at a full localization matrix and then made fast.

## The Solution

Two rules, one for each trap above.

**Give every language-sweeping suite one golden file per language, generated from `allCases`.** Not one file for the matrix, one per language, named by the language code, each holding that language's full grid. A change to German touches `..._blessed_snapshot__de.snap.txt` and nothing else, so a German rendering change gets a German-sized diff.

**Let the fast test tier read the goldens but never write them.** A six-second run is worth having for iteration, but only if it can never become the thing that blesses a change. The slow, app-hosted, on-device-font run stays the single source of truth. The fast tier compares against it and reports disagreement rather than resolving it.

Together, the matrix gets reviewed like copy and iterated on like unit tests.

### One file per language, generated from `allCases`

Language is the axis a rendering change moves along, so it is the axis the files split on. The matrix suite loops over every language and every intent and writes one snapshot per language:

{% raw %}
```swift
import Testing
import Foundation

enum Language: String, CaseIterable { case en, de, fr, ja /* one case per supported language */ }
enum TimeFormat { case ampm, twentyFourHour }
final class Settings { static let shared = Settings(); var language = Language.en; var timeFormat = TimeFormat.ampm }
enum DateFormatIntent: String, CaseIterable { case dailyHeader, sunEventTime, moonDate, complicationHour }

// Two fixed instants, built from components in the display time zone:
// Wed 2026-01-07 09:05 (AM, single-digit day and hour) and Sat 2026-08-15 17:39 (PM, double digits).
enum Samples {
    static func date(year: Int, month: Int, day: Int, hour: Int, minute: Int) -> Date {
        var calendar = Calendar(identifier: .gregorian)
        calendar.timeZone = .current  // real code uses the forecast's time zone
        return calendar.date(from: DateComponents(year: year, month: month, day: day, hour: hour, minute: minute))!
    }
    static var morning: Date { date(year: 2026, month: 1, day: 7, hour: 9, minute: 5) }
    static var evening: Date { date(year: 2026, month: 8, day: 15, hour: 17, minute: 39) }
}

// Set the app's language and clock, run the body, and restore both — always, via defer.
func withAppLanguage(_ language: Language, timeFormat: TimeFormat = .ampm, body: () -> Void) {
    let originalLanguage = Settings.shared.language, originalTimeFormat = Settings.shared.timeFormat
    defer { Settings.shared.language = originalLanguage; Settings.shared.timeFormat = originalTimeFormat }
    Settings.shared.language = language
    Settings.shared.timeFormat = timeFormat
    body()
}

extension Date {
    // Real code routes through the date rulebook: intent + current settings select the template.
    func withFormat(_ intent: DateFormatIntent) -> String { /* ... */ }
}

// Escape quotes, backslashes, control characters, and non-ASCII whitespace or
// format scalars so invisible characters survive a diff while letters stay legible.
func escaped(_ value: String) -> String {
    value.unicodeScalars.map { scalar in
        switch scalar {
        case "\"": return "\\\""
        case "\\": return "\\\\"
        default:
            if scalar.isASCII {
                return scalar.value < 0x20 ? String(format: "\\u{%04X}", scalar.value) : String(scalar)
            }
            if scalar.properties.isWhitespace || scalar.properties.generalCategory == .format {
                return String(format: "\\u{%04X}", scalar.value)  // narrow spaces, directional marks
            }
            return String(scalar)
        }
    }.joined()
}

@MainActor
@Suite("Date format snapshots", .serialized)
struct DateFormatSnapshotTests {
    @Test func everyLanguageMatchesItsBlessedSnapshot() {
        for language in Language.allCases {
            var lines = ["intent | morning 12h | evening 12h | morning 24h | evening 24h"]

            for intent in DateFormatIntent.allCases {
                var cells: [String] = []

                withAppLanguage(language) {
                    cells.append(Samples.morning.withFormat(intent))
                    cells.append(Samples.evening.withFormat(intent))
                }
                withAppLanguage(language, timeFormat: .twentyFourHour) {
                    cells.append(Samples.morning.withFormat(intent))
                    cells.append(Samples.evening.withFormat(intent))
                }

                #expect(!cells.contains(""), "\(language.rawValue) \(intent.rawValue) rendered empty — refusing to bless")
                lines.append("\(intent.rawValue) | \(cells.map(escaped).joined(separator: " | "))")
            }

            assertMatchesSnapshot(lines.joined(separator: "\n") + "\n", named: language.rawValue)
        }
    }
}
```
{% endraw %}

Two `allCases` drive the whole matrix. Add a `DateFormatIntent` case and every language file grows a row on the next record; add a `Language` and a new file appears. No separate list of things to test can fall out of sync with the feature. Rows are the intents, columns are the two instants crossed with the two clocks, so a German file reads like a table:

```
intent | morning 12h | evening 12h | morning 24h | evening 24h
dailyHeader | Mi., Jan. 7 | Sa., Aug. 15 | Mi., Jan. 7 | Sa., Aug. 15
sunEventTime | 9:05am | 5:39pm | 9:05 | 17:39
forecastUpdatedDateTime | Mi., Jan. 7 @ 9:05am | Sa., Aug. 15 @ 5:39pm | Mi., Jan. 7 @ 9:05 | Sa., Aug. 15 @ 17:39
complicationHour | 9am | 5pm | 9 | 17
```

Two details make the file safe to trust. The `#expect(!cells.contains(""))` guard fails the record when a rendering path returns an empty string. Blank output is the one result a snapshot must never accept: it looks like success, reads like nothing, and usually means a missing symbol or a nil that fell through. The `escaped` helper handles the other silent case. Localized date strings are full of invisible characters: narrow no-break spaces, directional marks, and the format-category scalars ICU inserts between numerals and symbols. Rendered raw, a normal space turning into a narrow no-break space is invisible in a diff. Escaped, with the letters left legible, it shows up as a readable `\u{202F}`.

### The assertion that reads the file, and the guard that stops CI from writing

The suite is only as trustworthy as the helper under it. It has to get two things right: a regeneration must never feel like a rubber stamp, and an automated environment must never record a golden it should only compare. It is small enough to read whole:

```swift
import Foundation
import Testing

enum Snapshots {
    static let directory = URL(fileURLWithPath: #filePath)
        .deletingLastPathComponent()
        .appendingPathComponent("snapshots")

    static var updating: Bool { ProcessInfo.processInfo.environment["UPDATE_SNAPSHOTS"] == "1" }
    static var locked: Bool { ProcessInfo.processInfo.environment["CI"] != nil }

    // camelCase -> snake_case, then anything else to underscores: DateFormatSnapshotTests -> date_format_snapshot_tests
    static func sanitized(_ component: String) -> String {
        component
            .replacingOccurrences(of: "([a-z0-9])([A-Z])", with: "$1_$2", options: .regularExpression)
            .replacingOccurrences(of: "([A-Z])([A-Z][a-z])", with: "$1_$2", options: .regularExpression)
            .lowercased()
            .replacingOccurrences(of: "[^a-z0-9]+", with: "_", options: .regularExpression)
            .trimmingCharacters(in: CharacterSet(charactersIn: "_"))
    }

    // The first eight lines that differ — a short read, not the whole file.
    static func firstDifferences(recorded: String, current: String) -> String {
        let a = recorded.components(separatedBy: "\n"), b = current.components(separatedBy: "\n")
        var diffs: [String] = []
        for i in 0..<max(a.count, b.count) where diffs.count < 8 {
            let left = i < a.count ? a[i] : "<missing>", right = i < b.count ? b[i] : "<missing>"
            if left != right { diffs.append("line \(i + 1):\n  recorded: \(left)\n  current:  \(right)") }
        }
        return diffs.joined(separator: "\n")
    }
}

// Simplified: the real helper also auto-numbers snapshots when `named:` is omitted,
// so parameterized tests sharing one function name get distinct files.
func assertMatchesSnapshot(
    _ value: String,
    named name: String,
    filePath: String = #filePath,
    function: String = #function,
    sourceLocation: SourceLocation = #_sourceLocation
) {
    let suite = Snapshots.sanitized(URL(fileURLWithPath: filePath).deletingPathExtension().lastPathComponent)
    let test = Snapshots.sanitized(function.replacingOccurrences(of: "()", with: ""))
    let file = "\(test)__\(Snapshots.sanitized(name)).snap.txt"
    let url = Snapshots.directory.appendingPathComponent(suite).appendingPathComponent(file)
    let relativePath = "snapshots/\(suite)/\(file)"

    // The common path: a committed golden exists and no update was requested. Compare and report.
    if !Snapshots.updating, let recorded = try? String(contentsOf: url, encoding: .utf8) {
        if recorded == value { return }
        Issue.record(
            "output drifted from \(relativePath) — regenerate with bin/unit-test --update-snapshots, then review every changed line. First differences:\n\(Snapshots.firstDifferences(recorded: recorded, current: value))",
            sourceLocation: sourceLocation
        )
        return
    }

    // Otherwise we are about to write: the golden is missing, or an update was requested. Never under CI.
    guard !Snapshots.locked else {
        Issue.record(
            "snapshot \(relativePath) is missing or an update was requested, but snapshots are locked under CI — record locally and commit the file",
            sourceLocation: sourceLocation
        )
        return
    }

    do {
        try FileManager.default.createDirectory(at: url.deletingLastPathComponent(), withIntermediateDirectories: true)
        try value.write(to: url, atomically: true, encoding: .utf8)
    } catch {
        Issue.record("failed to write snapshot \(relativePath): \(error)", sourceLocation: sourceLocation)
    }
}
```

Notice what the drift message does. It names at most eight differing lines, not the whole file. It carries the exact regenerate command. And it says *review every changed line* rather than *rerun to fix*, because a regeneration is a set of user-visible edits somebody has to sign off on. Every line that moves in a golden is a change a person approved.

The `locked` guard is the other half. Locally, a missing golden is recorded on first run and an update request rewrites it; that auto-recording is a convenience for a developer on a branch. When `CI` is set, both paths fail instead of writing. Otherwise a missing golden in an automated environment would be silently written as correct, and a real regression would record itself as the new truth.

### Width probes store strings, not widths

The same one-file-per-language shape carries three other sweeps: the "x ago" relative-time strings, the CLDR unit-conformance tables, and the width probes. The width probes answer a pixel question, will the widest rendering fit the slot, and yet store no pixel measurement. For each intent and clock the suite finds the widest-rendering string across a probe set, selecting by the real device font but storing the winning *string*, never its width. A point width drifts with OS versions, font revisions, and rendering hosts, so a golden built on it would fail for reasons unrelated to the app. The string it chose is stable: the file records that in German the widest daily-full-header is `Donnerstag, Sept. 24`, which any tier on any machine can compare byte for byte.

### The fast tier that reads the goldens but cannot rewrite them

The app-hosted run is the authoritative gate. It builds the app scheme, signs it, boots a simulator, and measures the real device font. It also takes about forty seconds warm, too slow for iterating on a pure-logic file like the date rulebook. So a second tier compiles a subset of sources plus the pure-logic and snapshot suites into a macOS-hosted bundle, with no simulator, host, or signing, and runs in about six seconds.

Two tiers now read the same goldens. If the fast one could *write* them, it could rebase the source of truth to whatever macOS ICU renders, which can differ from iOS ICU by a locale or two. The unsigned, hostless tier would be defining reality for the authoritative one. So the runner refuses the record flag and strips the record environment variable before it runs anything:

```bash
#!/bin/bash
# Fast logic tier: macOS-hosted, no simulator, no app host, no signing.
# It READS the committed goldens and must match them; it may never WRITE them.

# Recording is iOS-authoritative: bless via bin/unit-test --update-snapshots. Refuse the flag here...
for arg in "$@"; do
  if [ "$arg" = "--update-snapshots" ]; then
    echo "❌ Recording from the macOS host is not allowed — the goldens are"
    echo "   iOS-authoritative (bless with bin/unit-test --update-snapshots)."
    echo "   A macOS parity failure is a stop-the-line finding, never a re-record."
    exit 1
  fi
done

# ...and strip the variable so nothing downstream can record either.
unset UPDATE_SNAPSHOTS

# Simplified: the real script also pins a shared DerivedData, SPM cache, and compilation cache.
xcodebuild test \
  -workspace HelloWeather.xcworkspace \
  -scheme HelloWeatherLogicTests \
  -destination 'platform=macOS' \
  -quiet \
  CODE_SIGNING_ALLOWED=NO
```

A passing fast run reads the same committed goldens and renders them identically, so it produces zero diffs and zero re-records. When it disagrees, that is not a bug to paper over by recording on macOS. It is a signal that macOS and iOS ICU diverge for some locale, and the fix is to drop that suite from the macOS bundle and keep it on the authoritative tier only. The fast tier buys its speed by giving up any claim to authority.

## Results

- About 111 committed golden files, ~564K total, across six suites, regenerated only on device and reviewed as copy.
- A one-language rendering fix produces a one-language diff instead of a full-matrix regeneration to scan for the rows that moved.
- Iterating on the date rulebook dropped from a ~40-second app-hosted cycle to a ~6-second macOS one, on the identical goldens. The trade-off: a suite where macOS ICU and iOS ICU diverge leaves the fast bundle and runs only on the slow tier.
- An empty rendered cell and a CI-time missing snapshot both stop the run, instead of recording themselves as expected.

## Lessons Learned

- **Treat a snapshot diff as a behavior change.** It is a copy edit, not a green-keeping chore. Make the failure message say "review every changed line," carry the regenerate command, and show only the first few differences.

- **Store the artifact a metric chose, not the metric.** When a measurement only selects among candidates, commit the candidate. The chosen string is stable across machines; the width that chose it is not.

- **Escape the invisible characters.** Narrow spaces and directional marks are exactly the changes a raw diff hides. Escape whitespace and format scalars to `\u{...}` and leave the letters legible.

- **Never kill a language-cycling run mid-way.** The sweeps set the app language and restore it at the end. Stopping mid-flight strands the test store on a foreign language, and later runs fail the formatting suites until you clear it.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the silent failure instead of the product, the title lost its subtitle, each mechanism section says its part once, Results is what changed and what it cost, and Lessons Learned fell from seven bullets to the four the body does not already state. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The matrix suite's sample instants were 2025 epoch values that contradicted the German table (a Tuesday and a Friday, not the Wednesday and Saturday the goldens show); they are now the real 2026 component-built dates, and the loop uses the real set-and-restore `withAppLanguage` shape. The snapshot helper was rewritten to the real control flow (compare when a golden exists and no update is requested; otherwise record unless `CI` is set, which the earlier excerpt got backwards for a locally missing file), with the real camelCase-to-snake_case file naming and `test__name` filenames. The fast-tier script now carries the real refusal message and scheme names, and the unverifiable "hundreds of passes" count was dropped. All figures checked out: 111 files, 564K, six suites, 27 languages, 24 intents, ~40s versus ~6s, and the never-stop-a-language-cycling-run rule.
