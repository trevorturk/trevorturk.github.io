---
layout: post
title: "Golden Files Per Language, and a Fast Test Tier That Must Match Them Byte for Byte"
date: 2026-08-24 08:20:00 -0600
summary: "One committed golden file per language turns a localized rendering matrix into a reviewable diff, and a six-second macOS test tier reads the same files without ever being allowed to rewrite them."
tags: [swift, ios, testing, snapshot-testing, localization]
---

## The Problem

[Hello Weather](https://helloweather.com) renders dates on far more surfaces, and in far more languages, than a weather app usually bothers with: 24 date and time surfaces, 27 languages, both 12- and 24-hour clocks — maintained by one person with a model doing the translations. A rendering system that large fails one language at a time, silently. A German weekday abbreviation is fine, a Vietnamese header collides two numerals, a Japanese hour drops its counter symbol. Nothing crashes, the app builds, and the regression only shows up on a screen in a language nobody on the team reads, in a build that already shipped.

The obvious test asserts the English output and stops there. It passes forever while every other language drifts, because English is the one case that matches the source strings and the developer's intuition; a localized output checked only in `en` cannot see the bug it exists to catch.

The opposite instinct fails too. Snapshot every language against every intent against every clock and you get one enormous file, and a real one-language change buries itself in a regeneration that rewrites the whole thing. Nobody reads a seventy-thousand-line diff; they approve it and move on, which is the same as having no test.

The date path here is a rulebook: every user-visible date string flows through one enum of intents, each mapped to a render plan by one exhaustive switch (the design is its own [post](/date-format-rulebook/)). A rulebook is only trustworthy if a change to it produces a diff a human can read. This post is about the snapshot layer that makes that true, and a second test tier that runs the same checks in six seconds without being allowed to corrupt them. It is the sequel to [porting the snapshot ergonomics](/port-the-ergonomics-not-the-library/) — a drop-in file-snapshot helper — pointed at a full localization matrix and then made fast.

## The Solution

Two rules carry the design, one for each trap above.

**Give every language-sweeping suite one golden file per language, generated from `allCases`.** Not one file for the matrix — one per language, named by the language code, each holding that language's full grid. A change to German touches `..._de.snap.txt` and nothing else, so the review surface for a German rendering change is a German-sized diff.

**Let the fast test tier read the goldens but never write them.** A six-second run is worth having for iteration, but only if it can never become the thing that blesses a change. The slow, app-hosted, on-device-font run stays the single source of truth; the fast tier compares against it and reports disagreement rather than resolving it. Together the rules turn a localized rendering matrix into something reviewed like copy and iterated on like unit tests.

### One file per language, generated from `allCases`

Language is the axis a rendering change moves along, so it is the axis the files split on. The matrix suite loops over every language and every intent and writes one snapshot per language:

{% raw %}
```swift
import Testing
import Foundation

enum Language: String, CaseIterable { case en, de, fr, ja /* one case per supported language */ }
enum DateFormatIntent: String, CaseIterable { case dailyHeader, sunEventTime, moonDate, complicationHour }

// Two fixed instants: a January morning and an August evening.
let sampleInstants = [Date(timeIntervalSince1970: 1_736_240_700), Date(timeIntervalSince1970: 1_755_286_740)]

func render(_ date: Date, _ intent: DateFormatIntent, language: Language, is24Hour: Bool) -> String {
    // Real code routes through the date rulebook; a locale-driven formatter stands in here.
    let formatter = DateFormatter()
    formatter.locale = Locale(identifier: language.rawValue)
    return formatter.string(from: date)  // intent + is24Hour would select the template
}

// Escape whitespace and format scalars so invisible characters survive a diff,
// while ordinary letters stay legible.
func escaped(_ value: String) -> String {
    value.unicodeScalars.map { scalar in
        if scalar == "\\" { return "\\\\" }
        if scalar.isASCII {
            return scalar.value < 0x20 ? String(format: "\\u{%04X}", scalar.value) : String(scalar)
        }
        if scalar.properties.isWhitespace || scalar.properties.generalCategory == .format {
            return String(format: "\\u{%04X}", scalar.value)  // narrow spaces, directional marks
        }
        return String(scalar)
    }.joined()
}

@Test func everyLanguageMatchesItsBlessedSnapshot() {
    for language in Language.allCases {
        var lines = ["intent | morning 12h | evening 12h | morning 24h | evening 24h"]

        for intent in DateFormatIntent.allCases {
            var cells: [String] = []
            for is24Hour in [false, true] {
                for date in sampleInstants {
                    cells.append(render(date, intent, language: language, is24Hour: is24Hour))
                }
            }
            #expect(!cells.contains(""), "\(language.rawValue) \(intent.rawValue) rendered empty — refusing to bless")
            lines.append("\(intent.rawValue) | \(cells.map(escaped).joined(separator: " | "))")
        }

        assertMatchesSnapshot(lines.joined(separator: "\n") + "\n", named: language.rawValue)
    }
}
```
{% endraw %}

The whole matrix is driven by two `allCases`, which is what keeps it maintainable: add a `DateFormatIntent` case and every language file grows a row on the next record; add a `Language` and a new file appears. There is no separate list of "things to test" to fall out of sync with the feature. The rows are the intents (weekday rails, headers, hours, event times) and the columns are the two instants crossed with the two clocks, so a German file reads like a table you can scan:

```
intent | morning 12h | evening 12h | morning 24h | evening 24h
dailyHeader | Mi., Jan. 7 | Sa., Aug. 15 | Mi., Jan. 7 | Sa., Aug. 15
sunEventTime | 9:05am | 5:39pm | 9:05 | 17:39
forecastUpdatedDateTime | Mi., Jan. 7 @ 9:05am | Sa., Aug. 15 @ 5:39pm | Mi., Jan. 7 @ 9:05 | Sa., Aug. 15 @ 17:39
complicationHour | 9am | 5pm | 9 | 17
```

Two small decisions make the file safe to trust. The `#expect(!cells.contains(""))` guard fails the record when a rendering path returns an empty string, because blank output is the one result a snapshot must never accept: it looks like success and reads like nothing, and usually means a missing symbol or a nil that fell through. And the `escaped` helper earns its place on real data. Localized date strings are full of invisible characters — narrow no-break spaces, directional marks, format-category scalars ICU inserts between numerals and symbols — and rendered raw, a normal space turning into a narrow no-break space is invisible in a diff, which is exactly the change you most need to see. Escaping whitespace and format scalars to `\u{...}` while leaving letters legible means the dangerous invisible shows up as a readable `\u{202F}`.

### The assertion that reads the file, and the guard that stops CI from writing

The suite is only as trustworthy as the helper under it, which has to get two things right: never let a regeneration feel like a rubber stamp, and never let an automated environment record a golden it should only compare. It is small enough to read whole:

```swift
import Foundation
import Testing

enum Snapshots {
    static let directory = URL(fileURLWithPath: #filePath)
        .deletingLastPathComponent()
        .appendingPathComponent("snapshots")

    static var updating: Bool { ProcessInfo.processInfo.environment["UPDATE_SNAPSHOTS"] == "1" }
    static var locked: Bool { ProcessInfo.processInfo.environment["CI"] != nil }

    static func sanitized(_ component: String) -> String {
        component.lowercased()
            .replacingOccurrences(of: "[^a-z0-9]+", with: "_", options: .regularExpression)
            .trimmingCharacters(in: CharacterSet(charactersIn: "_"))
    }

    // The first N lines that differ — a short read, not the whole file.
    static func firstDifferences(recorded: String, current: String, limit: Int = 8) -> String {
        let a = recorded.components(separatedBy: "\n"), b = current.components(separatedBy: "\n")
        var diffs: [String] = []
        for i in 0..<max(a.count, b.count) where diffs.count < limit {
            let left = i < a.count ? a[i] : "<missing>", right = i < b.count ? b[i] : "<missing>"
            if left != right { diffs.append("line \(i + 1):\n  recorded: \(left)\n  current:  \(right)") }
        }
        return diffs.joined(separator: "\n")
    }
}

func assertMatchesSnapshot(
    _ value: String,
    named name: String,
    filePath: String = #filePath,
    sourceLocation: SourceLocation = #_sourceLocation
) {
    let suite = Snapshots.sanitized(URL(fileURLWithPath: filePath).deletingPathExtension().lastPathComponent)
    let file = "\(Snapshots.sanitized(name)).snap.txt"
    let url = Snapshots.directory.appendingPathComponent(suite).appendingPathComponent(file)
    let relativePath = "snapshots/\(suite)/\(file)"

    // Recording is a deliberate local act: the flag must be set, and we must not be in CI.
    if Snapshots.updating {
        guard !Snapshots.locked else {
            Issue.record("\(relativePath): snapshots are locked under CI — record locally and commit the file", sourceLocation: sourceLocation)
            return
        }
        try? FileManager.default.createDirectory(at: url.deletingLastPathComponent(), withIntermediateDirectories: true)
        try? value.write(to: url, atomically: true, encoding: .utf8)
        return
    }

    guard let recorded = try? String(contentsOf: url, encoding: .utf8) else {
        Issue.record("\(relativePath) is missing — record it with UPDATE_SNAPSHOTS=1", sourceLocation: sourceLocation)
        return
    }
    if recorded == value { return }
    Issue.record(
        "output drifted from \(relativePath) — regenerate with UPDATE_SNAPSHOTS=1, then review every changed line. First differences:\n\(Snapshots.firstDifferences(recorded: recorded, current: value))",
        sourceLocation: sourceLocation
    )
}
```

Notice what the drift message does. It names at most eight differing lines, not the whole file; it carries the exact regenerate command; and it says *review every changed line* rather than *rerun to fix*, because a regeneration is a set of user-visible edits somebody has to sign off on. That is the rule the whole system rests on: a snapshot diff is a behavior change and gets the same review as a copy change. Every line that moves in a golden is a change a person approved.

The `locked` guard is the other half. When `CI` is set, an update request fails instead of writing. Auto-recording is a convenience for a developer on a branch; in an automated environment a missing golden would be silently written as "correct" and a real regression would record itself as the new truth. Recording is a local, human-reviewed act; the locked environment can only ever compare.

### Width probes store strings, not widths

The same one-file-per-language shape carries three other sweeps: the "x ago" relative-time strings, the CLDR unit-conformance tables, and the width probes. The width probes are worth pausing on, because they answer a pixel question — will the widest rendering fit the slot? — and yet store no pixel measurement. For each intent and clock the suite finds the widest-rendering string across a probe set, selecting by the real device font but storing the winning *string*, never its width. A point width drifts with OS versions, font revisions, and rendering hosts, so a golden built on it would fail for reasons unrelated to the app. The string it chose is a stable fact: the file records that in German the widest daily-full-header is `Donnerstag, Sept. 24`, which any tier on any machine can compare byte for byte.

### The fast tier that reads the goldens but cannot rewrite them

The app-hosted run is the authoritative gate: it builds the app scheme, signs it, boots a simulator, and measures the real device font. It also takes about forty seconds warm, too slow for the tight iteration a pure-logic file like the date rulebook needs — that file went through hundreds of passes. So there is a second tier that compiles a subset of sources plus the pure-logic and snapshot suites into a macOS-hosted bundle with no simulator, host, or signing, and runs in about six seconds.

Two tiers now read the same goldens, and the danger is in that sentence. If the fast one could *write* them, it could rebase the source of truth to whatever macOS ICU renders, which can differ from iOS ICU by a locale or two — the fast, unsigned, hostless tier defining reality for the slow, authoritative one. So the runner refuses the record flag and strips the record environment variable before it runs anything:

```bash
#!/bin/bash
# Fast logic tier: macOS-hosted, no simulator, no app host, no signing.
# It READS the committed goldens and must match them; it may never WRITE them.

# Recording is authoritative on the device tier only. Refuse the flag here...
for arg in "$@"; do
  if [ "$arg" = "--update-snapshots" ]; then
    echo "❌ Recording from the macOS host is not allowed — the goldens are"
    echo "   device-authoritative. A macOS parity failure is stop-the-line, never a re-record."
    exit 1
  fi
done

# ...and strip the variable so nothing downstream can record either.
unset UPDATE_SNAPSHOTS

xcodebuild test \
  -workspace App.xcworkspace \
  -scheme LogicTests \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO
```

The parity rule is the trust bar: the macOS run reads the *same* committed goldens and must render them identically, so a passing fast run produces zero diffs and zero re-records. When two tiers share a source of truth, one writes it and the other is mechanically prevented from trying. And when the fast tier disagrees, that is not a bug to paper over by recording on macOS — it is a signal that macOS and iOS ICU diverge for some locale, and the fix is to drop that suite from the macOS bundle and keep it on the authoritative tier only. The fast tier buys its speed by giving up any claim to authority.

## Results

- About 111 committed golden files, ~564K total, across six suites — regenerated only on device and reviewed as copy.
- A one-language rendering fix now produces a one-language diff, not a full-matrix regeneration to scan for the rows that moved.
- Coverage tracks the feature with no maintenance list: a new intent or language shows up in every relevant golden on the next record, because every sweep is driven by `allCases`.
- Iterating on the date rulebook dropped from a ~40-second app-hosted cycle to a ~6-second macOS one, on the identical goldens, with no path by which the fast tier can rebase them.
- An empty rendered cell and a CI-time missing snapshot both stop the run now, instead of recording themselves as expected.

## Lessons Learned

- **Split the golden along the axis your changes move.** The unit of a good snapshot is the unit of a good review; language is where our changes land, so one file per language turns a one-language edit into a one-language diff.

- **Generate coverage from `allCases`, not a hand-kept list.** If the set under test is derived from the same enum the feature is built on, coverage cannot fall behind the feature, and there is no second list to forget.

- **Treat a snapshot diff as a behavior change.** It is a copy edit, not a green-keeping chore, so make the failure message say "review every changed line," carry the exact regenerate command, and show only the first few differences.

- **Store the artifact a metric chose, not the metric.** When a measurement only selects among candidates, commit the candidate: the chosen string is stable across machines; the width that chose it drifts with the machine.

- **Escape the invisible characters.** Narrow spaces and directional marks are exactly the changes a raw diff hides, so escape whitespace and format scalars to `\u{...}` and leave the letters legible.

- **A faster second tier must be unable to write the source of truth.** Let it read the goldens and forbid it from writing them — refuse the record flag, strip the record variable, treat disagreement as stop-the-line — or it becomes the authority it was never meant to be.

- **Never kill a language-cycling run mid-way.** The sweeps set the app language and restore it at the end, so stopping mid-flight strands the test store on a foreign language and later runs fail the formatting suites until you clear it.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.
