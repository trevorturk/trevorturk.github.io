---
layout: post
title: "Am I Allowed to Be Slow Here? One Enum That Gates Every Expensive Path"
date: 2026-08-25 08:20:00 -0600
summary: "The same weather-fetch code runs in six single-purpose processes with wildly different time budgets, so one app-wide context enum is set on entry, reset on exit, and consulted before anything slow runs."
tags: [swift, ios, architecture, patterns]
---

## The Problem

The code that fetches a forecast, decodes it, and hands it to the UI is written once. It runs in at least six very different places. [Hello Weather](https://helloweather.com) is an iOS app, a watch app, a Home Screen widget, a watch complication, a silent-push background handler, and a locations-list preview, and every one of those pulls in the same services from the same shared source. That is the whole point of a shared codebase: you do not want six copies of the fetch-and-decode path drifting apart.

But the moments they run in are not interchangeable. The foreground app can take as long as it likes; the user is looking at a spinner and will wait. A widget timeline refresh has something like fifteen seconds before the system kills it. A silent push has roughly thirty. A complication has less than either. And the expensive things the fetch path can do (an on-device AI summary that needs several seconds of model time, a live radar map made of UIKit tiles, scheduling local notifications, syncing state out to the widget and watch) are exactly the things that are fine in one place and fatal in another. Run the on-device summary inside a widget refresh and the widget does not slow down, it gets terminated and renders blank. Schedule a "rain starting" notification from the foreground app and you have just notified someone about weather they are looking at.

This is a one-person product. The fix could not be "remember which context each service is safe in and be careful at every call site." It had to be something a service could ask, at the moment it runs, without every caller threading an answer down to it. The obvious SwiftUI-shaped answer, an `@Environment` value passed through every view and view model and service, was rejected: the services that need the answer are singletons far below the view tree, reached from a background push handler that has no view tree at all.

## The Solution

There is one app-wide enum, `RunContext`, with a case per process-and-moment. A single shared manager holds the current value. Every entry point that is not the plain foreground app sets the context as its first act and resets it on the way out. Services that can do something expensive read the current context and decide for themselves whether they are allowed. The value is set once per invocation, in one place, and every layer below sees it without being handed anything.

It began life in late 2024 as a field on a telemetry event — a way to tag operational stats with where they came from. It earned its keep later, when it turned out that "where am I running?" was the exact question the expensive paths needed to ask. By early 2025 there was a commit whose title is the entire argument for this post: *"Try to fix OnActive being called during bg task / ping / etc."* Background refreshes were triggering foreground-only view work. The enum was already there to stop it.

### The same code, six times, with different budgets

Start with the thing everyone reads. The enum names the situation, not the format, and it carries the one derivation every caller actually wants: is this a moment where a human is waiting, or a moment on a clock.

```swift
import Foundation

enum RunContext: String {
    case app            // main app, foreground, effectively no time limit
    case watch          // watch app, foreground
    case locations      // the locations-list preview
    case widget         // WidgetKit timeline refresh (~15s before termination)
    case complication   // watch complication (tighter still)
    case ping           // silent-push background handler (~30s)

    var isForeground: Bool {
        switch self {
        case .app, .watch, .locations: true
        case .widget, .complication, .ping: false
        }
    }

    var isBackground: Bool { !isForeground }
}

@MainActor
final class AppContext {
    static let shared = AppContext()
    private init() {}

    #if os(iOS)
    private static let base: RunContext = .app
    #else
    private static let base: RunContext = .watch
    #endif

    private(set) var current: RunContext = base

    func set(_ context: RunContext) { current = context }
    func reset() { current = Self.base }
}
```

Notice what `isForeground` groups and what it splits. `.app`, `.watch`, and `.locations` are all "a person is looking at this"; `.widget`, `.complication`, and `.ping` are all "a clock is running and nobody is watching." Most callers do not care about the six-way distinction. They care about the two-way one, and they get it as a computed property so the grouping lives in exactly one place. When someone adds a seventh context, they add one case and the compiler makes them answer the one question that matters: which side of the line is it on. The default is derived per-platform at compile time, so a fresh process on iOS starts as `.app` and on the watch as `.watch` without anyone setting anything — the base state is correct, and only the unusual entry points have to override it.

### Set it at the door, reset it on the way out

The rule that makes a global safe here is a discipline, not a mechanism: every entry point that is not the plain foreground app sets the context as its first statement and guarantees a reset on every exit path. If any provider forgets the reset, the next thing to run in that process inherits a stale context and starts making decisions for a situation it is not in. So the reset is not optional and it is not conditional — it runs whether the body succeeded or threw.

Here are two real entry points, reduced to structure. The widget timeline provider sets `.widget`; the silent-push handler on the app delegate sets `.ping`. Both use `defer` so the reset cannot be skipped by an early return or a thrown error.

```swift
import WidgetKit
import UIKit

struct WeatherEntry: TimelineEntry {
    let date: Date
    var nextRefresh: Date { date.addingTimeInterval(15 * 60) }
}

func refreshWeather() async -> WeatherEntry { WeatherEntry(date: Date()) }

// 1. WidgetKit timeline provider — set on entry, reset on every exit path.
@MainActor
struct WeatherTimelineProvider: TimelineProvider {
    func placeholder(in context: Context) -> WeatherEntry { WeatherEntry(date: Date()) }
    func getSnapshot(in context: Context, completion: @escaping (WeatherEntry) -> Void) {
        completion(WeatherEntry(date: Date()))
    }
    func getTimeline(in context: Context, completion: @escaping (Timeline<WeatherEntry>) -> Void) {
        AppContext.shared.set(.widget)
        Task {
            defer { AppContext.shared.reset() }
            let entry = await refreshWeather()
            completion(Timeline(entries: [entry], policy: .after(entry.nextRefresh)))
        }
    }
}

// 2. Silent-push handler on the app delegate.
final class AppDelegate: NSObject, UIApplicationDelegate {
    func application(_ application: UIApplication,
                     didReceiveRemoteNotification userInfo: [AnyHashable: Any],
                     fetchCompletionHandler completion: @escaping (UIBackgroundFetchResult) -> Void) {
        Task { @MainActor in
            AppContext.shared.set(.ping)
            defer { AppContext.shared.reset() }
            _ = await refreshWeather()
            completion(.newData)
        }
    }
}
```

Notice the `defer`. The repository today spells the reset out explicitly on each exit branch, once in the success path and once in the `catch`, which is the same guarantee written by hand and easy to get wrong when a third branch appears. `defer { reset() }` is the tidy equivalent: one line, no branch can escape it. The watch complication provider is the third instance of exactly this shape, setting `.complication`. There is nothing clever in any of them. The cleverness is that there is only one thing to remember, and it is the same thing every time.

### Let each expensive path ask before it runs

Threading a budget parameter through the fetch path would mean every intermediate function grows an argument it only passes along. Instead the expensive leaf asks the shared context directly, at the moment it is about to spend time. This is where the enum pays for itself: the decision lives next to the cost, not up at the call site that has no idea what the leaf is about to do.

A service that can do something slow reads the context and refuses when the budget is wrong. Here is the shape, with the enforcing test beside it:

```swift
import Foundation
import Testing

struct WeatherFetch {
    let context: RunContext

    // The on-device summary needs several seconds of model time.
    // Fine in the foreground app; fatal anywhere on a background clock.
    var maySummarizeOnDevice: Bool { context == .app }

    // A conservative request ceiling; the foreground app can wait longer
    // than a widget the system is about to terminate.
    var requestTimeout: TimeInterval {
        context.isForeground ? 10 : 6
    }
}

@Test func widgetGetsAShortBudgetAndSkipsTheSummary() {
    let widget = WeatherFetch(context: .widget)
    #expect(widget.requestTimeout == 6)
    #expect(widget.maySummarizeOnDevice == false)

    let app = WeatherFetch(context: .app)
    #expect(app.requestTimeout == 10)
    #expect(app.maySummarizeOnDevice == true)
}
```

The test is the specification: in a widget, the summary is skipped and the timeout is short; in the app, both open up. That is the invariant AGENTS.md states in one line — *"SummaryService is app-context only; do not use it in ping/widget flows"* — turned into something a build can check.

In the real code the split of labor is worth being precise about. The on-device refusals are enforced on the client, because only the client knows whether it is a widget or the foreground app: the AI summary is app-context only, and the live radar map — a UIKit tile view that is expensive to spin up — renders only when `context == .app`, so a widget or a preview never pays for a map it cannot show. The network timeout is different. The client keeps a single conservative HTTP ceiling rather than tuning it per context; it sends its context to the server as a request parameter, and the server budgets per weather source and current load. The enum makes both halves possible: it is how the client decides what to refuse locally, and what it tells the server so the server can decide the rest.

The background-only guard is the mirror image of the summary guard. Local notifications (a "rain starting soon," a severe-weather alert) must fire only from a background refresh, never from the foreground app where the user is already looking at the forecast. So the notification-scheduling paths open with `guard AppContext.shared.current.isBackground else { return }`, the exact inverse of the summary's `context == .app`. Same enum, opposite side of the same line. The scene-phase view work that started this whole story guards the other way again: the refresh that runs when the app becomes active first checks `isForeground`, so a background ping that happens to wake a view modifier does not trigger foreground-only work. One value, read three ways, keeps three different things in their lane.

### The honest cost: it is a global

This is global mutable state: a single `var` on a shared singleton, written by whichever entry point ran last, read by everyone. The reasons you are taught to avoid that are real. If two contexts were ever live at once in one process, the last writer would win and the reset could fire in the middle of the other one's work. The commit that fixed "OnActive being called during bg task / ping" is a scar from exactly this class of problem: a background task and a foreground view modifier disagreeing about the moment they were in.

It is chosen anyway, and the justification is a property of these particular processes, not a general endorsement. Each context is a single-purpose, short-lived process that does one thing and exits: a widget refresh, a complication refresh, a ping handler, a foreground session. Within one process there is one context at a time, set at the single entry point, so there is no concurrent writer to race against. The global is safe because the runtime shape guarantees only one value matters at once.

Where would it break? The moment a single process runs two contexts concurrently: say a foreground app that spawns an independent background task that should be treated as `.ping` while the foreground work continues as `.app`. A shared `var` cannot represent both, and you would need to carry the context explicitly for that path. The code already hedges toward this. The forecast service takes an optional context override in its initializer, so a specific call can pin its own context instead of trusting the global. That override is the escape hatch, and its existence is the tell that the global is a deliberate simplification, not a claim that one value is always enough. When the process model stops guaranteeing one context at a time, this pattern is the first thing that has to change.

## Results

- Six execution contexts share one fetch-and-decode path, and each one gets the right behavior — summary or no summary, notifications or no notifications, live map or no map — without any caller passing a mode down through the layers. The decision lives next to the cost.
- The background-only guard means severe-weather and precipitation notifications fire from silent-push refreshes and never from the foreground app, so nobody is notified about weather already on their screen.
- A widget refresh and a complication refresh cannot start the several-seconds-long on-device summary or the UIKit radar map, which is the difference between a widget that renders and a widget the system terminates blank.
- Adding a context is a one-case change that the compiler turns into a checklist: the `isForeground` switch will not build until the new case declares which side of the foreground/background line it belongs on.

## Lessons Learned

- **Set ambient state at the single entry point, and reset it unconditionally.** A context that every layer reads is only safe if exactly one place writes it and the reset cannot be skipped; a `defer` at the top of each provider is worth more than a reset carefully placed in each branch, because branches multiply and `defer` does not.
- **Put the "am I allowed" check next to the cost, not at the call site.** The function that is about to spend three seconds knows what it is about to do; the caller five layers up does not. Reading a shared context at the leaf keeps every intermediate function free of a parameter it would only forward.
- **Split what only the client knows from what the server can decide.** The client refuses on-device work by context because only it knows it is a widget; it hands the same context to the server and lets the server budget the network, because that is where the load information lives. One value serves both.
- **A global is a bet on your process model, so name the bet.** Ambient mutable state is safe here only because each context is a single-purpose, short-lived process with one context at a time; write that assumption down, keep an explicit-override escape hatch for the path that violates it, and treat the day two contexts run at once as the day this pattern is retired.

---

## How This Post Was Made

**Prompt 1:** "please kick off a big batch to look through all skills looking for other topics that might be interesting to blog about. we could look at git history, but I think since we've been using claude/codex for the last ~year we should have most of the interesting stuff built into the skills by now. however, you can also look at the changelog view in the iOS repo for other highlights that might be worth dispatching research about. come back to me with a list of possible topics (that haven't already been covered in the blog) …"

**Prompt 2:** "lets do 4, 20, 21, 22 -- the others I think are not worth it"

Ten Claude agents mined the iOS, web, and Android skills, the iOS changelog, and the plan indexes for uncovered topics; the owner picked four from the ranked list. This post was researched and drafted by one agent from the cited skills, plans, and code, under the why-then-how voice and self-contained-code brief settled in the previous localization batch, then reviewed before publishing.
