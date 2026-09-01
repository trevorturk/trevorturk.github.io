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

**Give every language-sweeping suite one golden file per language, generated from `allCases`.** Not one file for the matrix, one per language, named by the language code, each holding that language's full grid. A change to German touches `..._de.snap.txt` and nothing else, so a German rendering change gets a German-sized diff.

**Let the fast test tier read the goldens but never write them.** A six-second run is worth having for iteration, but only if it can never become the thing that blesses a change. The slow, app-hosted, on-device-font run stays the single source of truth. The fast tier compares against it and reports disagreement rather than resolving it.

Together, the matrix gets reviewed like copy and iterated on like unit tests.

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

Notice what the drift message does. It names at most eight differing lines, not the whole file. It carries the exact regenerate command. And it says *review every changed line* rather than *rerun to fix*, because a regeneration is a set of user-visible edits somebody has to sign off on. Every line that moves in a golden is a change a person approved.

The `locked` guard is the other half. When `CI` is set, an update request fails instead of writing. Auto-recording is a convenience for a developer on a branch. In an automated environment a missing golden would be silently written as correct, and a real regression would record itself as the new truth.

### Width probes store strings, not widths

The same one-file-per-language shape carries three other sweeps: the "x ago" relative-time strings, the CLDR unit-conformance tables, and the width probes. The width probes answer a pixel question, will the widest rendering fit the slot, and yet store no pixel measurement. For each intent and clock the suite finds the widest-rendering string across a probe set, selecting by the real device font but storing the winning *string*, never its width. A point width drifts with OS versions, font revisions, and rendering hosts, so a golden built on it would fail for reasons unrelated to the app. The string it chose is stable: the file records that in German the widest daily-full-header is `Donnerstag, Sept. 24`, which any tier on any machine can compare byte for byte.

### The fast tier that reads the goldens but cannot rewrite them

The app-hosted run is the authoritative gate. It builds the app scheme, signs it, boots a simulator, and measures the real device font. It also takes about forty seconds warm, too slow for a pure-logic file like the date rulebook, which went through hundreds of passes. So a second tier compiles a subset of sources plus the pure-logic and snapshot suites into a macOS-hosted bundle, with no simulator, host, or signing, and runs in about six seconds.

Two tiers now read the same goldens. If the fast one could *write* them, it could rebase the source of truth to whatever macOS ICU renders, which can differ from iOS ICU by a locale or two. The unsigned, hostless tier would be defining reality for the authoritative one. So the runner refuses the record flag and strips the record environment variable before it runs anything:

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
