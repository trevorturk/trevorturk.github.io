---
layout: post
title: "Golden Files Per Language"
date: 2026-08-24 08:20:00 -0600
summary: "We keep one committed golden file per language, so a change to one language shows up as a diff a person can read, and a six-second macOS test run checks the same files but can't rewrite them."
tags: [swift, ios, testing, snapshot-testing, localization]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

When a Japanese hour loses its counter symbol (the character that marks the number as an hour), nothing crashes. The app builds, the English screens look right, and the bug ships to a screen in a language nobody on the team reads. Localized date formatting fails like that: one language at a time, quietly. [Hello Weather](https://helloweather.com) shows dates in 24 places, in 27 languages, on both 12- and 24-hour clocks. One person maintains it and a model does the translating.

The obvious test checks the English output and stops. It keeps passing while every other language drifts, because English is the one language that matches the source strings.

Going the other way fails too. If we snapshot every language, every intent, and both clocks together, we get one enormous file. A real change to one language gets buried when the whole file is regenerated. Nobody reads a seventy-thousand-line diff. They approve it and move on, and then the test isn't protecting anything.

In our app, every date string a user sees goes through one enum of intents (what the date is for: a daily header, a sunrise time) and one exhaustive switch that maps each intent to a format. We call that the date rulebook, and its design is its own [post](/date-format-rulebook/). We can only trust the rulebook if a change to it produces a diff a person can read. This post is about the snapshot tests that make that true, and about a second, faster test run that checks the same files in six seconds but isn't allowed to change them. It continues from [porting the snapshot ergonomics](/port-the-ergonomics-not-the-library/): the same small file-snapshot helper, now pointed at the full language matrix and then made fast.

## The Solution

Two rules, one for each trap.

- **One golden file per language, generated from `allCases`.** Every test that sweeps the languages writes one file per language, named by the language code, holding that language's whole grid. A German change touches `..._blessed_snapshot__de.snap.txt` and nothing else, so it gets a German-sized diff.
- **The fast run reads the goldens but never writes them.** A six-second run is good for iterating, but only if it can't become the thing that approves a change. The slow run, hosted in the app on a simulator with the real device font, stays the single source of truth. The fast run compares against those files and reports a disagreement. It doesn't settle one.

### One file per language, generated from `allCases`

We split the files by language because that's how rendering changes arrive: one language at a time. The matrix test loops over every language and every intent and checks one snapshot per language:

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

The two `allCases` loops drive the whole matrix. Add a `DateFormatIntent` case and every language file grows a row the next time we record. Add a `Language` and a new file appears. There's no separate list of things to test that can fall behind the feature. Rows are the intents and columns are the two sample instants crossed with the two clocks, so the German file reads like a table:

```
intent | morning 12h | evening 12h | morning 24h | evening 24h
dailyHeader | Mi., Jan. 7 | Sa., Aug. 15 | Mi., Jan. 7 | Sa., Aug. 15
sunEventTime | 9:05am | 5:39pm | 9:05 | 17:39
forecastUpdatedDateTime | Mi., Jan. 7 @ 9:05am | Sa., Aug. 15 @ 5:39pm | Mi., Jan. 7 @ 9:05 | Sa., Aug. 15 @ 17:39
complicationHour | 9am | 5pm | 9 | 17
```

Two details make the file safe to trust. The `#expect(!cells.contains(""))` check fails the run when a rendering path returns an empty string. A snapshot must never accept blank output, because it looks like success and usually means a missing symbol or a nil that fell through. The `escaped` helper handles the other quiet failure. Localized date strings carry characters you can't see, like the narrow no-break space ICU puts between a number and its symbol. In a raw diff, a normal space turning into a narrow no-break space is invisible. Escaped, it shows up as `\u{202F}`, and the letters around it stay readable.

### The assertion that reads the file, and the guard that stops CI from writing

The test is only as trustworthy as the helper under it. The helper has to get two things right. Regenerating a golden must never feel like a rubber stamp, and CI must never record a golden it should only compare against. It's small enough to read whole:

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

The drift message does three things. It shows at most eight differing lines, not the whole file. It includes the exact regenerate command. And it says *review every changed line* rather than *rerun to fix*, because every changed line in a golden is a change to what users see, and someone has to sign off on it.

The `locked` guard is the other half. On a developer's machine, a missing golden gets recorded on the first run, and `UPDATE_SNAPSHOTS=1` rewrites it. When `CI` is set, both of those fail instead of writing. Without that guard, CI would quietly write a missing golden as correct, and a real regression would record itself as the new expected output.

### Width probes store strings, not widths

Three other sweeps use the same one-file-per-language shape: the "x ago" relative-time strings, the CLDR unit-conformance tables, and the width probes. The width probes answer a pixel question, whether the widest rendering fits its slot, but they don't store any pixel measurement. For each intent and clock, the test renders a set of candidate dates in the real device font, finds the widest one, and stores that *string*. It never stores the width. A width in points changes with OS versions, font revisions, and which machine is rendering, so a golden built on it would fail for reasons that have nothing to do with the app. The chosen string is stable. The German file records that the widest daily full header is `Donnerstag, Sept. 24`, and any test run on any machine can compare that.

### The fast run that reads the goldens but cannot rewrite them

The app-hosted run is the one that counts. It builds the app, signs it, boots a simulator, and measures text in the real device font. It also takes about forty seconds warm, which is too slow for iterating on a pure-logic file like the date rulebook. So we added a second run that compiles a subset of the sources, plus the logic and snapshot tests, into a macOS test bundle. No simulator, no app host, no signing, and it finishes in about six seconds.

Now two runs read the same goldens. If the fast one could *write* them, it could reset the source of truth to whatever macOS ICU renders, and macOS ICU differs from iOS ICU in a locale or two. The unsigned macOS run would be deciding what's correct for the iOS one. So the fast script refuses the update flag and unsets the update variable before it runs anything:

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

When the fast run passes, it rendered the committed goldens the same way iOS did, and nothing gets rewritten. When it disagrees, we don't fix that by recording on macOS. It means macOS and iOS ICU differ for some locale, and the fix is to drop that test from the macOS bundle and keep it on the app-hosted run only.

## Results

- About 111 committed golden files, ~564K in total, across six test suites. They're only regenerated on the iOS run and reviewed like copy.
- A fix to one language produces a one-language diff, instead of a whole-matrix regeneration we'd have to scan for the rows that moved.
- Iterating on the date rulebook went from a ~40-second app-hosted cycle to a ~6-second macOS one, against the same goldens. The trade-off is that a suite where macOS and iOS ICU disagree leaves the fast bundle and runs only on the slow one.
- An empty rendered cell and a missing snapshot under CI both fail the run instead of recording themselves as correct.

## Lessons Learned

- **Treat a snapshot diff as a behavior change.** It's a copy edit, not a chore to get the build green. Have the failure message say "review every changed line," include the regenerate command, and show only the first few differences.

- **Store what the measurement picked, not the measurement.** When a number only chooses among candidates, commit the candidate. The chosen string is stable across machines and the width isn't.

- **Escape the invisible characters.** A raw diff hides a narrow space or a directional mark. Escape whitespace and format scalars to `\u{...}` and leave the letters readable.

- **Don't stop a language-cycling run partway.** The sweeps set the app language and restore it at the end. If you kill the run in the middle, the stored test settings stay on a foreign language, and later runs fail the formatting tests until you clear it.
