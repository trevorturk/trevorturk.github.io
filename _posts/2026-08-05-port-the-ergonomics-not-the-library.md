---
layout: post
title: "Port the Ergonomics, Not the Library"
date: 2026-08-05 08:10:00 -0600
summary: "We brought snapshot testing from our Ruby server to Swift Testing without a package or a dependency. The part worth porting was three habits: a one-line assertion, file names worked out from the test name, and one flag to re-record. A design record for a ~100-line helper that shipped two days later."
tags: [testing, swift, ios, ruby, snapshot-testing, workflow]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

A snapshot test records a piece of output the first time it runs and fails whenever that output changes. The [Hello Weather](https://helloweather.com) iOS repo had two systems that worked this way, both hand-built for one job. One was a golden-table recorder: it rewrites a committed Swift file with every language x date-format combination. The other compared reports and printed a diff on failure, on a refactor branch. Each one is its own recorder plus its own tests. If we wanted to snapshot something new, a sync payload or a widget timeline, we'd have to build a third.

Our Ruby web repo has had the opposite experience for years. A ~170-line gem (`minitest-snapshots`) plus a few house conventions produced about 90 committed snapshot files across nine test suites. Most of them cover things nobody would have built a recorder for. When asserting "this output stays exactly like this" takes one line, people snapshot everything worth snapshotting. When it takes a recorder class, a file-naming decision, and a regeneration flag, they write the one snapshot test that's clearly worth the trouble and skip the rest.

So we designed a port. Not the gem, not a Swift package, and not a dependency on the well-known Swift snapshot-testing library. We wrote one ~100-line helper file in the test target, because everything risky already worked somewhere in the repo and the only missing piece was the convenience layer.

One caveat up front: this post was written as a design record, not a shipping report. The decision and the full design landed as a plans-only PR (iOS #1484, 2026-08-04). We queued the implementation behind a refactor that was already in flight, so it wouldn't add moving parts to work that was blocking a release. The queue was short. The helper shipped two days later (iOS #1514, 2026-08-06) with its first adopter, and as of 2026-09-01 it holds 111 snapshot files across six suites. The code below is the helper as it landed, which is the plan's code with two `Verify:` flags resolved. The decision is still the part you can reuse.

## The Origin: What Makes the Ruby Setup Work

The gem is simple. `assert_matches_snapshot value` compares the value against `test/snapshots/<suite>/<test>__<n>.snap.yaml`. The file is created on the first run, and the number goes up with each call inside a test. `rails test --update-snapshots` overwrites everything. Under `ENV["CI"]` a missing snapshot is a failure, not a new file, so CI can't quietly bless one.

Three things fall out of that, and they're why the tool gets used:

1. **One-line assertion.** Writing the test costs one line. There's no file to create and no name to invent.
2. **Automatic naming.** The file path comes from the suite and test names. Nobody decides where a snapshot lives.
3. **One-flag update.** One command re-records everything the run touches, and then you review the git diff. A behavior change gets reviewed by reading the snapshot diff, the same way a copy change does.

Everything else in the web repo is convention built on top of that. Two of those conventions carry over on their own.

### Snapshot the summary, not the payload

The most common way to misuse snapshot testing is to freeze a raw payload. That gives you a wall of JSON nobody can review, and every diff is noise. The web repo does the opposite. What it snapshots is usually a readable summary, built to be diffed.

The best example is the weather-data adapters. Each adapter's output is snapshotted as a table that compares it, field by field, against a reference adapter:

```
+-------------------------------+----------------------+----------------------+
|                               | Adapter Under Test   | Reference            |
+-------------------------------+----------------------+----------------------+
|                      timezone | America/Chicago      | America/Chicago      |
|         currently.temperature | 50.95                | 47.0                 |
|                currently.icon | cloudy               | cloudy               |
|            currently.humidity | 0.5                  | 0.61                 |
|         currently.windBearing | 120                  | 90                   |
```

A reviewer scanning that diff can see whether a parser change moved a field, dropped one, or drifted from the reference. A diff of raw vendor JSON tells you none of that.

The same idea shows up at other layers. SQL behavior is snapshotted as the list of statements the block ran, with numbers replaced by `?` and comments stripped. The snapshot captures the shape and count of the queries, not values that change from run to run:

```ruby
def assert_sql(&block)
  sql = []

  subscriber = ->(_name, _start, _finish, _id, payload) do
    sql << payload[:sql].split("/*").first.gsub(/\d+/, "?")
  end

  ActiveSupport::Notifications.subscribed(subscriber, "sql.active_record", &block)

  assert_matches_snapshot sql.join("\n") + "\n"
end
```

HTTP concurrency is snapshotted as two numbers: how many requests ran serially and how many in parallel, counted by a spy and saved as a two-key YAML hash. A request counts as serial if it finished on the same fiber as the one before it. A change that accidentally makes a parallel fetch serial fails a test with a two-line diff.

In each case the work goes into the code that builds the summary, and `assert_matches_snapshot` costs nothing. The split only works because the assertion is cheap.

### Coverage by metaprogramming

Because the assertion is one line, generating tests in a loop is cheap. The source smoke suite loops over every active data source and every unit system it supports, and defines a snapshot test for each combination:

```ruby
Api::Weather::ACTIVE_SOURCES.excluding(REFERENCE_SOURCE).each do |source|
  define_method "test_compare_#{source}_to_reference" do
    vcr_use_cassette("smoke_test_#{source}_us_forecast") do
      table = Api::Table.new(sources: [source, REFERENCE_SOURCE])
      assert_matches_snapshot table.pack.to_s
    end
  end
end

Api::Weather::ACTIVE_SOURCES.each do |source|
  Api::Weather.source_class(source).supported_source_units.each do |units|
    define_method("test_#{source}_#{units}") do
      # ...build the output table for this source + units...
      assert_matches_snapshot table.pack.to_s
    end
  end
end
```

Adding a new data source adds its comparison test, its per-unit-system output tests, and its request-count tests. The loop picks it up and the first run records the snapshots. Nobody writes tests for a new source. They review what the loop recorded.

### Two safety rules

Snapshot suites go wrong in two ways over time, and the web repo has a written rule for each:

- **English-only snapshots hide non-English drift.** A localized output that's only asserted in `en` keeps passing while every other language regresses. When you snapshot localized content, assert a non-English language too.
- **A fix whose diff is mostly snapshot churn is suspicious.** The review checklist flags any PR where the snapshot changes outweigh the code change. Anything that changes what goes over the wire has to be the point of the PR, not a side effect. If regenerating snapshots changed a hundred lines for a one-line fix, either the fix is bigger than it claims or the snapshots are freezing the wrong level of detail.

## The Decision: No Package, No Dependency

The obvious move was to adopt the well-known Swift snapshot-testing library. We decided against it. Nothing is wrong with it, but when we audited the repo, every risky mechanic already worked somewhere in-house. The library does a lot we don't need for text snapshots, like image strategies and a trait system. It would also have been the first package ever linked into the test target, and one more entry in the monthly dependency-update cycle. If we ever want image or SwiftUI-view snapshots, the library's core module is the upgrade path.

The audit is the part to copy, because "can our simulator tests even do this?" is the question that usually pushes a team toward a dependency. Three mechanics, and three places they were already proven:

1. **Simulator tests can write to the source checkout.** The golden recorder already resolves `URL(fileURLWithPath: #filePath)` and rewrites a committed Swift file in place, from a simulator test hosted in the app. The simulator shares the Mac's filesystem, so `#filePath` in a test file is a real, writable path into the repo.
2. **Environment flags reach the test process.** `xcodebuild` doesn't forward arbitrary environment variables to tests. It forwards only the ones prefixed `TEST_RUNNER_`, and strips the prefix. The repo's `bin/unit-test` already passes the golden-record flag through this way.
3. **A readable failure message already existed** on a refactor branch. It printed the first eight differing lines through `Issue.record`, named the file, and gave the regenerate command. We lifted it.

With all three proven, the only thing missing was the convenience layer, which is ~100 lines. We ported the three things that make the tool get used instead of importing a library for mechanics we already had.

### Naming: match the origin literally

The new names match the Ruby ones: `--update-snapshots` for the flag, `UPDATE_SNAPSHOTS` for the env var, `assertMatchesSnapshot` for the helper, and a `snapshots/` directory like the web repo's `test/snapshots/`. The repo's existing env flags carry an app-specific prefix, and we decided not to put that prefix on the new names just to match. When two codebases share a convention, a person or an agent moving between them should find the same words.

## The Design

The whole helper is one file in the test target. Here it is as it landed, lightly trimmed:

```swift
import Testing
import Foundation

enum Snapshots {
    static let directory = URL(fileURLWithPath: #filePath)
        .deletingLastPathComponent()
        .appendingPathComponent("snapshots")

    nonisolated(unsafe) static var environment = ProcessInfo.processInfo.environment

    static var updating: Bool {
        environment["UPDATE_SNAPSHOTS"] == "1"
    }

    static var locked: Bool {
        environment["CI"] != nil
    }

    private static let lock = NSLock()
    nonisolated(unsafe) private static var counters: [String: Int] = [:]

    static func nextIndex(forKey key: String) -> Int {
        lock.lock()
        defer { lock.unlock() }
        let next = (counters[key] ?? 0) + 1
        counters[key] = next
        return next
    }

    // Ruby-style snake_case: "DateFormatSnapshotTests" -> "date_format_snapshot_tests"
    static func sanitized(_ component: String) -> String {
        component
            .replacingOccurrences(of: "([a-z0-9])([A-Z])", with: "$1_$2", options: .regularExpression)
            .replacingOccurrences(of: "([A-Z])([A-Z][a-z])", with: "$1_$2", options: .regularExpression)
            .lowercased()
            .replacingOccurrences(of: "[^a-z0-9]+", with: "_", options: .regularExpression)
            .trimmingCharacters(in: CharacterSet(charactersIn: "_"))
    }

    static func firstDifferences(recorded: String, current: String) -> String {
        let recordedLines = recorded.components(separatedBy: "\n")
        let currentLines = current.components(separatedBy: "\n")
        var differences: [String] = []

        for index in 0..<max(recordedLines.count, currentLines.count) where differences.count < 8 {
            let recordedLine = index < recordedLines.count ? recordedLines[index] : "<missing>"
            let currentLine = index < currentLines.count ? currentLines[index] : "<missing>"
            if recordedLine != currentLine {
                differences.append("line \(index + 1):\n  recorded: \(recordedLine)\n  current:  \(currentLine)")
            }
        }

        return differences.joined(separator: "\n")
    }
}

// Parameterized tests (@Test(arguments:)) share one function name — pass named:.
func assertMatchesSnapshot(
    _ value: String,
    named name: String? = nil,
    filePath: String = #filePath,
    function: String = #function,
    sourceLocation: SourceLocation = #_sourceLocation
) {
    let suite = Snapshots.sanitized(URL(fileURLWithPath: filePath).deletingPathExtension().lastPathComponent)
    let test = Snapshots.sanitized(function.replacingOccurrences(of: "()", with: ""))
    let suffix = name.map(Snapshots.sanitized) ?? String(Snapshots.nextIndex(forKey: "\(suite)/\(test)"))
    let snapshotURL = Snapshots.directory
        .appendingPathComponent(suite)
        .appendingPathComponent("\(test)__\(suffix).snap.txt")
    let relativePath = "Tests/snapshots/\(suite)/\(test)__\(suffix).snap.txt"

    if !Snapshots.updating, let recorded = try? String(contentsOf: snapshotURL, encoding: .utf8) {
        if recorded == value { return }
        Issue.record(
            "output drifted from \(relativePath) — regenerate with bin/unit-test --update-snapshots, then review every changed line. First differences:\n\(Snapshots.firstDifferences(recorded: recorded, current: value))",
            sourceLocation: sourceLocation
        )
        return
    }

    guard !Snapshots.locked else {
        Issue.record(
            "snapshot \(relativePath) is missing or an update was requested, but snapshots are locked under CI — record locally and commit the file",
            sourceLocation: sourceLocation
        )
        return
    }

    do {
        try FileManager.default.createDirectory(at: snapshotURL.deletingLastPathComponent(), withIntermediateDirectories: true)
        try value.write(to: snapshotURL, atomically: true, encoding: .utf8)
    } catch {
        Issue.record("failed to write snapshot \(relativePath): \(error)", sourceLocation: sourceLocation)
    }
}
```

The line to notice is the `guard !Snapshots.locked` after the comparison. Under `CI`, a missing file and an explicit update both fail instead of writing, which is the same lock the gem calls `lock_snapshots`. The `environment` variable is there so the helper's own tests can exercise the lock without a real `CI` variable. Values are plain strings in `.snap.txt` files rather than the gem's `.snap.yaml`, because we chose not to port the YAML serializer. Callers turn their value into a string first, and overloads for structured values can wait until someone needs one.

The flag side is a few lines in the existing test runner script, using the same `TEST_RUNNER_` mechanism the golden recorder already used:

```bash
# Snapshot update mode: bin/unit-test --update-snapshots (or UPDATE_SNAPSHOTS=1)
# rewrites every snapshot the run touches (Snapshots.swift); review the
# Tests/snapshots/ diff before committing. CI is forwarded so the
# helper can lock snapshots (missing files fail instead of auto-recording).
for arg in "$@"; do
  if [ "$arg" = "--update-snapshots" ]; then
    UPDATE_SNAPSHOTS=1
  fi
done
if [ "$UPDATE_SNAPSHOTS" = "1" ]; then
  export TEST_RUNNER_UPDATE_SNAPSHOTS=1
  echo "Updating snapshots (review the Tests/snapshots/ diff)"
fi
if [ -n "$CI" ]; then
  export TEST_RUNNER_CI="$CI"
fi
```

The last block isn't optional. GitHub Actions sets `CI` in the runner shell, but only `TEST_RUNNER_`-prefixed variables cross into the test process, so plain `CI` never arrives unless the script forwards it. Miss that and the lock never turns on, and nothing tells you.

### What Swift Testing changes

A port isn't a word-for-word copy. Swift Testing changes three details:

- **Parallel by default.** Minitest runs a suite's tests in one process, so a plain counter is enough. Swift Testing runs tests in parallel by default, so the `__N` counter has to be a dictionary keyed by suite and test, with a lock around it. Calls inside one test still run in order, so the numbering is stable per test. The lock only protects the dictionary from two tests touching it at once.
- **Parameterized tests collide.** `@Test(arguments:)` runs one function many times, and every run has the same `#function` string. Auto-numbering across those runs would depend on which one ran first. So parameterized tests have to pass an explicit `named:` argument. That's a rule written on the helper, not something the helper enforces at runtime.
- **One spelling flagged for verification.** The `sourceLocation: SourceLocation = #_sourceLocation` default argument makes a failure point at the caller's line instead of the helper's. It's the documented pattern for custom assertion helpers. The plan marked it `Verify:` against the toolchain's Swift Testing version, and it compiled as written when the helper landed. A design record should say what it isn't sure of. An agent that hit a compile error on that line would find the plan had already warned about it.

### What the helper deliberately does not replace

The plan said the golden-table system would stay. Its value was that the compiler checked coverage: the table was generated Swift covering `allCases` of language x format intent, so a newly added case couldn't go missing without a compile error. The plan recorded keeping it as a decision, not an oversight, and left the golden's future as an open question. The landing PR answered it the other way. The date matrix became the helper's first adopter, rendered from the same `allCases` loops into one snapshot file per language (27 files, the same bytes the golden had asserted). The golden recorder, its generated file, and its flag were deleted. Coverage now comes from the loops in the test plus exhaustive switches in the date-format contract tests. The plan's other targets are still the helper's to-do list: sync payload shapes, widget timeline dumps, notification content, and turning a couple of standalone validator tools into report-generating tests that run behind an env flag. The report-generating tests are the part that has happened.

## Lessons Learned

- **Check what already works in your repo before adding a library.** Writing the host filesystem, passing env vars through `xcodebuild`, a readable failure message: all three were already proven in-repo. The library would only have added convenience, which is the cheap part.
- **Review the snapshot diff like a copy change.** Recording on the first run and re-recording with one flag are only safe if someone reads every changed snapshot line.
- **Check that the CI lock actually turns on.** Under `xcodebuild` the `CI` variable doesn't reach the test process unless the script forwards it as `TEST_RUNNER_CI`. A lock that never turns on looks the same as one that works.
- **Write down what the new framework changes.** A word-for-word port gets parallel-by-default and parameterized tests wrong. List the differences, and flag what you haven't verified.
- **A plan with the full code in it can be picked up by anyone.** Ours had the helper source, the runner patch, non-goals, and open questions. It was picked up two days later, and the landed helper is the plan's code with the two `Verify:` flags resolved.
