---
layout: post
title: "Deleting the Workarounds"
date: 2026-07-29 09:10:00 -0600
summary: "For seven months we added workarounds for Digital Crown scrolling in our Apple Watch app, and the workarounds were the bug. Two root causes, one PR that mostly deleted code, and how we wrote our own scroll momentum on watchOS."
tags: [swift, swiftui, watchos, ios, debugging]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

The Digital Crown (the dial on the side of an Apple Watch) scrolled the [Hello Weather](https://helloweather.com) watch app *sometimes*. It worked after you dragged the screen with a finger, but not before. It worked when you opened the app from the app list, but not from a complication (the app's tile on the watch face). It stopped working after a refresh, and again after you dismissed an error alert. It never worked inside the detail screens, or on the root screen after you came back from one. A crown that never scrolled would have been easy to debug. Sometimes was the hard case.

One customer report finally described what was happening instead of what wasn't. On a large watch, the crown was never dead. Turning it flung the horizontal "Coming up" hourly strip about thirty hours sideways. On a smaller watch that strip sits below the fold, so the same thing looks like "the crown does nothing."

So the crown wasn't unfocused. It was focused on the wrong thing.

## The Workaround Pile

Over seven months, each symptom got its own workaround. In December the vertical scroll view got `@FocusState` and code to grab focus when the screen appeared. In March, alerts showing at launch grabbed focus first, so we added a delayed retry, an `onChange` refocus when the alert count changed, and a `scenePhase` handler for coming back from the background. In May, focus still didn't stick after an idle launch, so the single retry became three retries, with a defocus and refocus in the middle. Two weeks after that we marked the hourly strip's scroll view `.focusable(false)`, because we already suspected it was stealing the crown. Error alerts stole focus too. We diagnosed that one in July and never shipped a handler for it.

By July, `ForecastView` looked like this:

```swift
struct ForecastView: View {
    @Environment(\.scenePhase) private var scenePhase

    private static let focusRetryIntervalsNanoseconds: [UInt64] = [0, 150_000_000, 500_000_000]
    private static let focusResetDelayNanoseconds: UInt64 = 25_000_000

    @FocusState private var isScrollViewFocused: Bool
    @State private var focusRetryTask: Task<Void, Never>?
    @State private var isViewVisible: Bool = false

    // ...

    private func refocusScrollView() {
        guard isViewVisible else { return }
        focusRetryTask?.cancel()

        focusRetryTask = Task { @MainActor in
            for interval in Self.focusRetryIntervalsNanoseconds {
                try? await Task.sleep(nanoseconds: interval)
                guard Task.isCancelled == false, isViewVisible else { return }
                await setScrollViewFocus()
            }

            focusRetryTask = nil
        }
    }

    private func setScrollViewFocus() async {
        isScrollViewFocused = false

        try? await Task.sleep(nanoseconds: Self.focusResetDelayNanoseconds)
        guard Task.isCancelled == false, isViewVisible else { return }
        isScrollViewFocused = true
    }
}
```

Three magic intervals. A defocus and refocus with a 25ms gap between them. A cancellable task that tracks by hand whether the view is visible. Four separate triggers calling into it.

Every detail screen carried its own three-line version of the same idea:

```swift
@FocusState private var isScrollViewFocused: Bool

var body: some View {
    ScrollView(.vertical, showsIndicators: false) {
        // ...
    }
    .focusable()
    .focused($isScrollViewFocused)
    .onAppear { isScrollViewFocused = true }
}
```

None of it worked completely. Each workaround fixed the case it was written for and left the others alone.

## Finding the Real Causes

Two changes in how we tested broke the stalemate.

First, we stopped believing that the simulator can't reproduce crown bugs. It can. Plain scroll-wheel events don't count as crown input, but scroll events with a began, changed, and ended phase do. We checked the simulated input against the watch's own Settings app to make sure it behaved like a real crown. After that, every experiment was a simulator run instead of a device session.

Second, we tested with the crown only, before touching the screen. Every earlier QA pass had tapped or dragged something first. That silently gave the crown a new owner and hid the launch state. Two findings fell out right away.

### Cause 1: nested scrollables compete for crown ownership

watchOS binds the crown to one scrollable view at a time. A horizontal `ScrollView` nested inside the vertical one is still a scroll view, and at launch it won. Turning the crown moved the hourly strip sideways instead of scrolling the page.

`.focusable(false)` doesn't stop this. Removing the strip's programmatic `scrollTo` doesn't either. The nested scroll view claimed the crown just by being there. Every launch-path workaround was fighting for ownership that the layout was handing away.

### Cause 2: explicit focus management suppresses native crown routing

The detail screens had never crown-scrolled. That bug survived eight-plus experiments in a single June research session, including reclaiming focus with `@FocusState`, `.focusable(interactions: .edit)`, moving to `NavigationStack`, and recreating the view with an `.id` nonce.

All of those experiments assumed the focus code was part of the solution. It was the problem. Remove `.focusable()`, `.focused()`, and the focus-on-appear from a pushed `ScrollView`, and watchOS routes the crown to it on its own, immediately, in every case. Apple's own documented crown examples never put a raw focusable `ScrollView` in the happy path.

Before rewriting anything, we checked whether Apple was about to fix this. The OS release notes, beta notes, and the year's SwiftUI session content had no crown focus-ownership changes, though nearby crown and scroll-view bugs were being triaged. The same complication-launch failure had been reported against one of Apple's own apps. Nothing was coming from the platform, so we fixed it ourselves.

## The Fix Is a Deletion

The change removed both root causes and every workaround built on top of them:

- The nested horizontal `ScrollView` became a clipped `HStack` panned by a `DragGesture`.
- With no competing scrollable, the root `ScrollView` got the crown on its own, so all the focus code went, along with the `scenePhase` and alert refocus handlers.
- The three detail screens lost their `.focusable()`/`.focused()`/focus-on-appear blocks.

In app code, that's 204 lines added and 121 removed (the PR's 391/122 total includes the plan doc). Nearly all of the additions are the hand-rolled pan, not new crown logic. `ForecastView` ended up simpler than it was *before* the first workaround shipped.

### Hand-rolling the pan

If you replace a `ScrollView`, you have to replace its physics too. The strip tracks an offset and clamps it to the content width. After you let go, it slows down the way a `UIScrollView` does, by shrinking the velocity a fixed fraction every millisecond:

```swift
private var horizontalDragGesture: some Gesture {
    DragGesture(minimumDistance: 8)
        .onChanged { value in
            decelerationTask?.cancel()

            guard dragStart != value.startLocation else { return }
            dragStart = value.startLocation
            dragAxis = abs(value.translation.width) > abs(value.translation.height) ? .horizontal : .vertical
        }
        .updating($dragOffset) { value, state, _ in
            guard dragAxis == .horizontal else {
                state = 0
                return
            }
            state = value.translation.width
        }
        .onEnded { value in
            guard dragAxis == .horizontal else { return }

            var transaction = Transaction()
            transaction.disablesAnimations = true
            withTransaction(transaction) {
                scrollOffset = clampedScrollOffset(scrollOffset + value.translation.width)
            }

            decelerate(initialVelocity: value.velocity.width)
        }
}

private func decelerate(initialVelocity: CGFloat) {
    decelerationTask?.cancel()
    guard abs(initialVelocity) > 50 else { return }

    decelerationTask = Task {
        var velocity = initialVelocity
        var lastTick = ContinuousClock.now

        while abs(velocity) > 12 {
            guard (try? await Task.sleep(for: .milliseconds(16))) != nil else { return }

            let now = ContinuousClock.now
            let dt = min(lastTick.duration(to: now) / .seconds(1), 0.05)
            lastTick = now

            let unclamped = scrollOffset + velocity * dt
            scrollOffset = clampedScrollOffset(unclamped)
            if scrollOffset != unclamped { break }

            velocity *= pow(0.998, dt * 1000)
        }
    }
}
```

Three things to notice:

- **Decay per millisecond, not per frame.** `velocity *= pow(0.998, dt * 1000)` gives the same physics whether the loop ticks at 60Hz or drops frames. A flat per-frame constant doesn't.
- **Clamp `dt`.** `min(dt, 0.05)` keeps an app resuming from the background from teleporting the strip to the far edge on its first tick.
- **Latch the axis by `startLocation`, not by a reset in `onEnded`.** A cancelled gesture never calls `onEnded`, so cleanup there goes stale and deadens the *next* pan. Keying the latch to the start location means every gesture re-latches and there's nothing to clean up.

That's the July version. As of September 2026 the strip has changed twice since. On 2026-08-06 the sleep loop became a single spring animation toward the gesture's predicted end point, because watchOS has no display link (a timer tied to the screen refresh) and a sleep loop can't line up with it. On 2026-08-15 the hand-rolled pan gave way to a native horizontal `ScrollView` with `.scrollDisabled(true)`, paged by tap and swipe. The focus code deleted here hasn't come back.

## Results

Every case in the matrix, tested with the crown only before any touch:

| Case | Before | After |
|---|---|---|
| Cold launch | Dead vertically; crown flings the hourly strip | Scrolls the page |
| Successful refresh | Dead | Works |
| Failed refresh → alert → dismiss | Dead | Works |
| Crown right after swiping the strip | Crown drives the strip | Scrolls the page |
| Crown inside pushed detail screens | Dead | Works |
| Crown on root after Back | Dead | Works |

One trade-off, found by testing the running app rather than by reading the code: a vertical page flick that *starts* on the strip is sometimes swallowed, because on watchOS a `DragGesture` on a child view blocks the outer scroll view's pan. `.simultaneousGesture`, plain `.gesture`, and a larger `minimumDistance` all behave the same. We shipped it anyway. You can see the failure, and flicking again from anywhere else fixes it. A dead crown gives no feedback at all.

## Lessons Learned

- **If one symptom has several workarounds, the workarounds are the bug.** Four of them each half-fixed the same complaint, which was four pieces of evidence for a shared cause nobody had named.
- **Reproduce the mechanism, not the symptom.** "Crown doesn't scroll" stayed unfixed for months. "Crown scrolls the wrong view" was fixed the next day: we reproduced it in the simulator on 2026-07-21 and merged the deletion PR on 2026-07-22.
- **Test the entry state.** Any QA step that touches the screen first destroys the launch-state bug you're hunting.
- **A workaround that stays long enough gets treated as part of the fix.** Every experiment kept the focus code and iterated around it.
- **Get a second review on deletions.** Four review passes caught a refresh-starvation regression, a snap-back bug, a stale gesture latch, and a lost VoiceOver scroll action. The original QA matrix covered none of them.
