---
layout: post
title: "Deleting the Workarounds"
date: 2026-07-29 09:10:00 -0600
summary: "Seven months of Apple Watch Digital Crown scrolling workarounds turned out to be the bug. Two structural root causes, one deletion PR, and how to hand-roll scroll momentum on watchOS."
tags: [swift, swiftui, watchos, ios, debugging]
model: "Claude"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

## The Problem

The Digital Crown in the [Hello Weather](https://helloweather.com) Apple Watch app scrolled *sometimes*. After a finger drag but not before one. From the app list but not from a complication. It went dead after a refresh, and dead again after dismissing an error alert. It never worked at all inside pushed detail screens, or on the root screen after navigating back. A crown that never scrolled would have been easy. Sometimes was the hard case.

One customer report finally described the mechanism instead of the symptom. On a large watch, the crown was never dead. Turning it silently flung the horizontal "Coming up" hourly strip about thirty hours sideways. On a smaller watch the strip sits below the fold, so the same behavior reads as "the crown does nothing."

That detail reframed everything. The crown wasn't unfocused. It was focused on the wrong thing.

## The Workaround Pile

Over seven months, each symptom got its own patch. In December the vertical scroll view got `@FocusState` and a focus-on-appear. In March, alerts appearing at launch caused a focus race, so a delayed retry arrived, with an `onChange` refocus on the alert count and a `scenePhase` handler for returning from the background. In May, focus didn't stick after an idle launch, so the single retry became a ladder of three, with a defocus-then-refocus reset in the middle; two weeks later the hourly strip, already suspected of stealing the crown, had its scroll view marked `.focusable(false)`. Error alerts stole focus too. That one was only diagnosed, in July, and never got its own handler.

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

Three magic intervals. A defocus/refocus cycle with a 25ms gap. A cancellable task tracking view visibility by hand. Four separate triggers calling into it.

Every pushed detail screen carried its own three-line version of the same idea:

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

None of it worked completely. Each patch fixed the case it was written for and left the others alone.

## Finding the Real Causes

Two changes in method broke the stalemate.

The first was to stop trusting the folklore that the simulator can't reproduce crown bugs. It can. Plain scroll-wheel events don't register as crown input; you have to post continuous-phase scroll events (`began`/`changed`/`ended`), validated against the watch's own Settings app as a control. That turned every experiment from a device session into a simulator run.

The second was to test with crown-only input, before any touch. Every prior QA pass had tapped or dragged something first, which silently rebound the crown and hid the launch state. Two structural findings fell out immediately.

### Cause 1: nested scrollables compete for crown ownership

watchOS binds the Digital Crown to exactly one scrollable at a time. A horizontal `ScrollView` nested inside the vertical one is still a scroll view, and at launch, it won. Crown rotation drove the hourly strip sideways instead of scrolling the page.

`.focusable(false)` does not prevent this. Neither does removing the strip's programmatic `scrollTo`. The nested scroll view claimed the crown merely by existing. Every launch-path workaround was fighting for ownership that the layout was handing away.

### Cause 2: explicit focus management suppresses native crown routing

The detail screens had never crown-scrolled. That bug survived eight-plus experiments in a single June research session, including `@FocusState` reclaim, `.focusable(interactions: .edit)`, `NavigationStack` migration, and `.id`-nonce view recreation.

All of those experiments assumed the focus machinery was part of the solution. It was the problem. Remove `.focusable()`/`.focused()`/`onAppear`-focus from a pushed `ScrollView` and watchOS routes the crown to it natively, immediately, in every case. Apple's own documented crown examples never put a raw focusable `ScrollView` in the happy path. When your configuration appears nowhere in the platform's sample code, that's evidence, not coincidence.

We did the upstream homework before committing to a rewrite. OS release notes, beta notes, and the year's SwiftUI session content contained zero crown focus-ownership changes, while adjacent crown and scroll-view bugs *were* being triaged. The same complication-launch failure had been reported against a first-party Apple app. Nothing was coming from the platform, so we fixed it ourselves.

## The Fix Is a Deletion

The change removed both root causes and every workaround built on top of them:

- The nested horizontal `ScrollView` became a clipped `HStack` panned by a `DragGesture`.
- With no competing scrollable, the root `ScrollView` bound the crown natively, so all the focus machinery went, along with the `scenePhase` and alert refocus handlers.
- The three detail screens lost their `.focusable()`/`.focused()`/focus-on-appear blocks.

Net, in app code: 204 lines added, 121 removed (the PR's 391/122 total includes the plan doc), and nearly all of the additions are the hand-rolled pan, not new crown logic. `ForecastView` ended up simpler than it had been *before* the first workaround shipped.

### Hand-rolling the pan

Replacing a `ScrollView` means replacing its physics. The strip tracks an offset, clamps it to content width, and applies UIScrollView-style exponential velocity decay after release:

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

Three details carry the pattern:

- **Decay per millisecond, not per frame.** `velocity *= pow(0.998, dt * 1000)` gives identical physics whether the loop ticks at 60Hz or drops frames. A flat per-frame constant does not.
- **Clamp `dt`.** `min(dt, 0.05)` keeps a resuming app from teleporting the strip to the far edge on its first tick.
- **Latch the axis by `startLocation`, not by a reset in `onEnded`.** Cancelled gestures never call `onEnded`, so reset bookkeeping there goes stale and deadens the *next* pan. Keying the latch to the start location means every gesture re-latches and there is nothing to clean up.

That is the July shape. As of September 2026 the strip has moved on twice: on 2026-08-06 the sleep loop became a single spring animation toward the gesture's predicted end, because watchOS has no display-link and a sleep loop can never align with vsync, and on 2026-08-15 the pan gave way to a native horizontal `ScrollView` with `.scrollDisabled(true)`, paged by tap and swipe. The focus machinery deleted here has not come back.

## Results

Every case in the matrix, exercised with crown-only input before any touch:

| Case | Before | After |
|---|---|---|
| Cold launch | Dead vertically; crown flings the hourly strip | Scrolls the page |
| Successful refresh | Dead | Works |
| Failed refresh → alert → dismiss | Dead | Works |
| Crown right after swiping the strip | Crown drives the strip | Scrolls the page |
| Crown inside pushed detail screens | Dead | Works |
| Crown on root after Back | Dead | Works |

One accepted trade-off, found by hostile runtime QA rather than code review: a vertical page flick that *starts* on the strip is occasionally swallowed, because on watchOS a descendant `DragGesture` starves the outer scroll view's pan. `.simultaneousGesture`, plain `.gesture`, and a larger `minimumDistance` all behave identically. We shipped it anyway. The failure is visible and self-healing (flick again from anywhere else), which is strictly better than a dead crown that gives no feedback at all.

## Lessons Learned

- **N workarounds for one symptom means the workarounds are the bug.** Four patches that each half-fixed the same complaint were four pieces of evidence for a shared cause nobody had named yet.
- **Reproduce the mechanism, not the symptom.** "Crown doesn't scroll" was unfixable for months. "Crown scrolls the wrong view" was fixed the next day: the mechanism was reproduced in the simulator on 2026-07-21 and the deletion PR merged on 2026-07-22.
- **Test the entry state.** Any QA step that touches the screen first destroys the launch-state bug you're hunting.
- **A workaround that survives long enough becomes an assumption.** The focus machinery was treated as part of the fix, so every experiment kept it and iterated around it.
- **Adversarial review earns its keep on deletions.** Four review passes caught a refresh-starvation regression, a snap-back bug, a stale gesture latch, and a lost VoiceOver scroll action, none of which the original QA matrix covered.
