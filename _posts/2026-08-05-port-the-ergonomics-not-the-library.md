---
layout: post
title: "Port the Ergonomics, Not the Library"
date: 2026-08-05 08:10:00 -0600
summary: "When bringing snapshot testing from a Ruby codebase to Swift Testing, the thing worth porting turned out to be three ergonomic properties - drop-in assertion, automatic naming, one-flag update - not a package and not a dependency."
tags: [testing, swift, ios, ruby, snapshot-testing, workflow]
---

## The Problem

Snapshot testing lives or dies on authoring cost. If asserting "this output stays exactly like this" takes one line, people snapshot everything worth snapshotting. If it takes a recorder class, a file-naming decision, and a bespoke regeneration flag, people write the snapshot test for the one system that justified the ceremony and skip it everywhere else.

The [Hello Weather](https://helloweather.com) iOS repo was living the second case. It had two snapshot-shaped systems, both good, both bespoke: a golden-table recorder that rewrites a committed Swift file with every language x date-format combination, and a diff-on-fail report comparison on a refactor branch. Each one is a whole recorder/tests pair, hand-built for a single domain. Adding a *new* snapshot contract - a sync payload shape, a widget timeline dump - meant building a third one.

Meanwhile the Ruby web repo has had cheap snapshots for years: a ~120-line gem (`minitest-snapshots`) plus house conventions, and as a direct result, about 180 committed snapshot files across 17 test suites covering things nobody would have written a bespoke recorder for.

So we designed a port. The interesting part is what the port *is*: not the gem, not a Swift package, not a dependency on the well-known Swift snapshot-testing library. A single ~120-line internal helper file, because everything risky was already proven in-repo and the only missing piece was the ergonomic layer.

One thing to be clear about up front: **this is a design record, not a shipping report.** The decision and the full design landed as a plans-only PR (iOS #1484, 2026-08-04); the implementation is deliberately queued behind an in-flight refactor program so it adds no moving parts to a release-blocker lane. The code below is the reviewed design, including the parts flagged for verification. We think the decision record is worth publishing on its own, because the decision is the reusable part.

## The Origin: What Makes the Ruby Setup Work

The gem's mechanics are simple: `assert_matches_snapshot value` compares against `test/snapshots/<suite>/<test>__<n>.snap.yaml`, auto-created on first run and auto-numbered per call within a test; `rails test --update-snapshots` overwrites everything; and a CI lock makes a *missing* snapshot a hard failure under `ENV["CI"]`, so CI can never silently bless a new one.

Three ergonomic properties fall out of that, and they are the whole reason the tool gets used:

1. **Drop-in assertion.** The entire authoring cost is the one line. No file to create, no name to invent.
2. **Automatic naming.** The snapshot path is derived from suite + test name. Nobody ever decides where a snapshot lives.
3. **One-flag update.** A single command re-records everything the run touches, and then **the git diff is the review artifact.** Reviewing a behavior change means reading the snapshot diff, same as reviewing a copy change.

Everything else in the web repo is convention layered on that primitive, and two of those conventions are worth stealing independently.

### Snapshot the summary, not the payload

The most-copied misuse of snapshot testing is freezing a raw payload - a wall of JSON that nobody can review, where every diff is noise. The web repo's habit runs the other way: the snapshotted artifact is usually a **derived, human-readable summary** built specifically to be diffed.

The flagship example: each weather-data adapter's output is snapshotted as a comparison table against a reference adapter, field by field:

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

A reviewer scanning that diff can see at a glance whether a parser change moved a field, dropped one, or drifted from the reference - which is a categorically different experience from diffing raw vendor JSON.

The same move shows up at other layers. SQL behavior is frozen as a normalized statement sequence - literals replaced, comments stripped - so the snapshot captures query *shape* and count, not volatile values:

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

And HTTP concurrency behavior is frozen as a spy's serial/parallel request counts (a request counts as serial if it completed on the same fiber as the previous one), snapshotted as two-line YAML. A change that accidentally serializes a parallel fetch fails a test with a two-line diff.

In each case the code that *derives* the summary is the investment, and `assert_matches_snapshot` is the free part. That division of labor only works when the assertion is free.

### Coverage by metaprogramming

Because the assertion is one line, generating tests is cheap. The source smoke suite loops over every active data source and every unit system it supports, defining a snapshot test per combination:

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

Adding a new data source to the app automatically adds its comparison test, its per-unit-system output tests, and its request-count tests - the loop picks it up, the first run records the snapshots, and the review is the diff of the new files. Nobody writes tests for a new source; they review what the loop recorded.

### Two safety rules

Snapshot suites accumulate two failure modes, and the web repo has a written rule for each:

- **English-only snapshots hide non-English drift.** A localized output asserted only in `en` will happily pass while every other language regresses. The repo rule: when snapshotting localized content, assert a non-English language too.
- **A fix diff dominated by snapshot churn is a review smell.** The review checklist flags any PR where the snapshot delta outweighs the change: every wire-visible delta must be the point of the change, not a ride-along. If regenerating snapshots produced a hundred changed lines for a one-line fix, either the fix is bigger than claimed or the snapshots are frozen at the wrong altitude.

## The Decision: No Package, No Dependency

The obvious move for the iOS side was to adopt the well-known Swift snapshot-testing library. We decided against it - not because there's anything wrong with it, but because an audit of the repo showed **every risky mechanic was already proven in-house**, and the library's breadth (image strategies, a trait system) is surface area the text-snapshot use case doesn't need. It would also have been the first package ever linked into the test target, plus a new entry in the monthly dependency-update cycle. The library's core module remains the explicit upgrade path if image or SwiftUI-view snapshots are ever wanted.

The audit is the part worth copying, because "can our simulator tests even do this?" is the question that usually pushes teams toward a dependency. Three mechanics, three existing proofs:

1. **Simulator tests can write the host source tree.** The golden recorder already resolves `URL(fileURLWithPath: #filePath)` and rewrites a committed Swift file in place from an app-hosted simulator test - the simulator shares the host filesystem, so `#filePath` from a test file is a real, writable path into the repo checkout.
2. **Environment flags reach the test process.** `xcodebuild` does not forward arbitrary env vars to tests; it forwards only vars prefixed `TEST_RUNNER_`, stripping the prefix. The repo's `bin/unit-test` already plumbs the golden-record flag through exactly this mechanism.
3. **Readable diff-on-fail already existed** on a refactor branch: a first-eight-differing-lines failure message via `Issue.record`, naming the file and the regenerate command. Lift it.

With all three proven, the only thing missing was the ergonomic layer - which is ~120 lines. That's the thesis: **port the three properties that make the tool get used; don't import a library to get mechanics you already have.**

### Naming: match the origin literally

One deliberate ruling worth recording: the new surface matches the Ruby names exactly - `--update-snapshots` as the flag, `UPDATE_SNAPSHOTS` as the env var, `assertMatchesSnapshot` as the helper, a `snapshots/` directory mirroring the web repo's `test/snapshots/`. The repo's existing env flags carry an app-specific prefix; the ruling was to *not* carry that legacy prefix onto new surface for consistency's sake. When two codebases share a convention, an engineer (or an agent) moving between them should find the same words. Prefer the better name; don't propagate churn-avoidance naming into the future.

## The Design

The whole helper is one file in the test target. The core of it, as designed:

```swift
import Testing
import Foundation

enum Snapshots {
    static let directory = URL(fileURLWithPath: #filePath)
        .deletingLastPathComponent()
        .appendingPathComponent("snapshots")

    static var updating: Bool {
        ProcessInfo.processInfo.environment["UPDATE_SNAPSHOTS"] == "1"
    }

    static var locked: Bool {
        ProcessInfo.processInfo.environment["CI"] != nil
    }

    private static let lock = NSLock()
    private static var counters: [String: Int] = [:]

    static func nextIndex(forKey key: String) -> Int {
        lock.lock()
        defer { lock.unlock() }
        let next = (counters[key] ?? 0) + 1
        counters[key] = next
        return next
    }

    static func sanitized(_ component: String) -> String {
        component.lowercased()
            .replacingOccurrences(of: "[^a-z0-9]+", with: "_", options: .regularExpression)
            .trimmingCharacters(in: CharacterSet(charactersIn: "_"))
    }
}

func assertMatchesSnapshot(
    _ value: String,
    named name: String? = nil,
    filePath: String = #filePath,
    function: String = #function,
    sourceLocation: SourceLocation = #_sourceLocation
) {
    let suite = Snapshots.sanitized(
        URL(fileURLWithPath: filePath).deletingPathExtension().lastPathComponent)
    let test = Snapshots.sanitized(function.replacingOccurrences(of: "()", with: ""))
    let suffix = name.map(Snapshots.sanitized)
        ?? String(Snapshots.nextIndex(forKey: "\(suite)/\(test)"))
    let snapshotURL = Snapshots.directory
        .appendingPathComponent(suite)
        .appendingPathComponent("\(test)__\(suffix).snap.txt")

    if !Snapshots.updating,
       let recorded = try? String(contentsOf: snapshotURL, encoding: .utf8) {
        if recorded == value { return }
        Issue.record(
            "output drifted from the snapshot — regenerate with " +
            "bin/unit-test --update-snapshots, then review every changed line. " +
            "First differences:\n\(firstDifferences(recorded, value))",
            sourceLocation: sourceLocation)
        return
    }

    guard !Snapshots.locked else {
        Issue.record(
            "snapshot is missing or an update was requested, but snapshots " +
            "are locked under CI — record locally and commit the file",
            sourceLocation: sourceLocation)
        return
    }

    try? FileManager.default.createDirectory(
        at: snapshotURL.deletingLastPathComponent(),
        withIntermediateDirectories: true)
    try? value.write(to: snapshotURL, atomically: true, encoding: .utf8)
}
```

Reading it against the three properties: the assertion is one line at the call site; the file path is derived from suite + test with a per-test auto-incrementing counter (`__1`, `__2`, ...); a missing file records and passes on first run; `UPDATE_SNAPSHOTS=1` re-records everything; and under `CI`, both the missing-file path and update mode fail instead of writing - the same lock the gem calls `lock_snapshots`. Values are raw strings in `.snap.txt` files rather than the gem's `.snap.yaml`, because the YAML serializer is deliberately not ported: callers canonicalize to a string (pretty-printed JSON, joined lines), and structured-value overloads wait until a real consumer needs one.

The flag side is a few lines in the existing test runner script, using the `TEST_RUNNER_` mechanism already proven for the golden recorder:

```bash
# Snapshot update mode: bin/unit-test --update-snapshots (or UPDATE_SNAPSHOTS=1)
# rewrites every snapshot the run touches; review the snapshots/ diff before
# committing. CI is forwarded so the helper can lock snapshots.
for arg in "$@"; do
  if [ "$arg" = "--update-snapshots" ]; then
    UPDATE_SNAPSHOTS=1
  fi
done
if [ "$UPDATE_SNAPSHOTS" = "1" ]; then
  export TEST_RUNNER_UPDATE_SNAPSHOTS=1
fi
if [ -n "$CI" ]; then
  export TEST_RUNNER_CI="$CI"
fi
```

Note the last block: the CI lock does not work by accident. GitHub Actions sets `CI` in the runner shell, but only `TEST_RUNNER_`-prefixed vars cross into the test process - so plain `CI` never arrives unless the script forwards it explicitly. Miss that and the lock silently never engages, which is the worst kind of safety feature.

### What Swift Testing changes

A port is not a transliteration; the destination framework's semantics reshape three details.

**Parallel by default.** Minitest runs a suite's tests in one process where a simple counter suffices. Swift Testing runs tests in parallel by default, so the `__N` auto-numbering counter must be a lock-guarded dictionary keyed by suite + test. Calls *within* one test are sequential, so numbering stays deterministic per test - the lock only defends the map against concurrent tests touching it.

**Parameterized tests collide.** `@Test(arguments:)` runs one function many times, and every invocation shares the same `#function` string - so auto-numbering across parameterized cases would depend on execution order. Parameterized tests must pass an explicit `named:` argument; that's a documented requirement on the helper rather than runtime machinery.

**One spelling flagged for verification.** The `sourceLocation: SourceLocation = #_sourceLocation` default argument - which makes failures point at the caller's line rather than the helper's - is the documented pattern for custom assertion helpers, but the plan explicitly marks it `Verify:` against the toolchain's Swift Testing version before implementation. Design records should carry their own uncertainty; an implementing agent that hits a compile error on that line should find the plan already told it this might happen.

### What the helper deliberately does not replace

The existing golden-table system stays. Its value is compile-enforced exhaustiveness - the table is generated Swift covering `allCases` of language x format intent, so a newly added case *cannot* be silently missing from coverage. File snapshots can't match that property, and the plan records keeping it as a decision, not an oversight. The helper is for everything that today isn't worth a bespoke recorder: sync payload shapes, widget timeline dumps, notification content, and draining a couple of standalone validator tools into env-gated report-generating tests.

## Lessons Learned

- **Identify why the tool gets used before deciding what to port.** For snapshot testing it's three properties: drop-in assertion, automatic naming, one-flag update. A port that delivers those in ~120 lines beats a dependency that delivers them plus a hundred things you don't need.
- **Audit for proven mechanics before reaching for a library.** The risky parts - writing the host filesystem from a simulator test, env plumbing through `xcodebuild`, readable diff-on-fail - all had existing in-repo proofs. The dependency would have bought ergonomics, and ergonomics are the cheap part.
- **The git diff is the review artifact.** First-run auto-record and one-flag update only stay safe because the culture treats a snapshot diff as a behavior change that gets read line by line, like a copy change. The mechanism and the culture ship together or not at all.
- **Lock CI, and verify the lock's plumbing.** CI must never silently bless a new snapshot - and in an `xcodebuild` world, the `CI` variable doesn't reach the test process unless you forward it under the `TEST_RUNNER_` prefix. A lock that never engages looks identical to a lock that works.
- **Snapshot derived summaries, not raw payloads.** Comparison tables, normalized SQL sequences, request-count YAML: build the artifact to be diffed by a human, and spend your effort in the deriving code, not the asserting code.
- **Cheap assertions compound through metaprogramming.** When the assertion is one line, a loop over your adapters gives every new one full snapshot coverage for free - the review is the diff of the recorded files.
- **Adopt the origin's names literally.** Same flag, same env var, same directory name across both codebases. Don't carry a legacy prefix onto new surface out of habit.
- **Port the semantics gap explicitly.** Parallel-by-default and parameterized tests are exactly the kind of thing a naive transliteration gets wrong. Write down what the destination framework changes, and flag the parts you haven't verified as unverified.
- **A design record is a shippable artifact.** The plan carries the complete helper source, the runner patch, non-goals, and its open questions - so implementation is a pick-up task for whoever (or whatever) gets there when the trigger fires, not a re-derivation.

---

## How This Post Was Made

**Prompt 1:** "see recent work in ~/Code/helloweather, perhaps a blog post about our opus 4.8 agents and why we decided to do that? perhaps something about the swift testing + snapshots inspired by minitest-snapshots? anything else? bring me a list of potential post ideas for review."

**Prompt 2:** "skip 4, 5, 6, 9 but create posts for each of the others in the 1-9 list. also add Four Answers to One Question, and Write the Rule, Not the Story -- show me a concise version of your plan and then I can approve" — then "proceed, one pr per post"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.
