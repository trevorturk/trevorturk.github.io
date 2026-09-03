---
layout: post
title: "Privacy-First Crash Reporting"
date: 2026-03-04 08:00:00 -0600
summary: "Crashes only: collect stack traces, check every SDK update for telemetry that's on by default, and confirm nothing else ships."
tags: [ios, privacy, sentry, mobile]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

We added a crash reporting SDK to the iOS app so we'd know when it crashes. On its defaults, the SDK also records user sessions, performance timings, network requests, user interactions, and breadcrumbs (a trail of what happened before a crash). That's an analytics platform, and we didn't want one. We wanted the stack trace and nothing else.

## The Philosophy

Crashes only.

Collect:
- Crash stack traces
- Device model and OS version (these don't identify anyone)

Don't collect:
- Sessions or performance telemetry
- User interactions or breadcrumbs
- Network requests or timing
- App hangs or diagnostic reports
- Any form of analytics or metrics

## The Challenge: SDK Updates

A configuration that's right today can be wrong after the next SDK update, because new versions add features and some of them are on by default. There's a second problem. Turning off a feature doesn't always turn off the machinery under it. Turning off `enableAutoPerformanceTracing` doesn't necessarily turn off `enableDataSwizzling`. Swizzling means the SDK swaps its own code into a system method so it can watch every call. That watching keeps running; it just stops reporting.

## Evaluation Protocol for SDK Updates

We run four checks before we update the SDK.

### 1. Check the changelog for defaults

Search the changelog for "enabled by default", and for the words tracking, monitoring, breadcrumbs, metrics, and swizzling. Each hit is something to look into before the update lands.

### 2. Audit mechanisms, not just feature flags

A feature that's off can leave its machinery running. Search the SDK source for:
- Swizzling or method interception
- Timer or observer registration
- Network monitoring hooks
- File system observers

If one of those is running, it's collecting data, whether or not anything sends that data yet.

### 3. Watch for these red flags

Look hard at anything involving these, or turn it off:

- Performance monitoring / tracing
- User interaction tracking
- Session replay or recording
- Diagnostic reports / MetricKit integration
- Network request monitoring
- Breadcrumb collection
- Any form of analytics or metrics
- Swizzling or other data collection infrastructure

### 4. Default to off

If we can't tell whether a feature collects user data, we turn it off. We can turn it on later. We can't take back data that's already been sent.

## Configuration Example

This is our shared Sentry helper, on sentry-cocoa 9.x. The iOS app, the watch app, and the widget providers all call it. We removed the DSN (the key that tells the SDK where to send reports) and trimmed a ticket reference from one comment; otherwise it's the real file. The same approach works with any crash reporting SDK:

```swift
import Sentry

class SentryHelper {
    static func activate() {
        #if targetEnvironment(simulator)
            // Never report from the simulator
        #else
            SentrySDK.start { options in
                options.dsn = "your-dsn"

                options.enableAutoSessionTracking = false           // Privacy: No usage telemetry
                options.enableAutoPerformanceTracing = false        // Privacy: No UI/animation/Core Data tracing
                options.enableMetrics = false                       // Privacy: No custom metrics telemetry

                options.enableAppHangTracking = false               // Unreliable, generates false positives
                options.enableWatchdogTerminationTracking = false   // Still reports hangs despite enableAppHangTracking = false
                options.experimental.enableWatchdogTerminationsV2 = false

                options.enableNetworkBreadcrumbs = false            // Privacy: No HTTP URL tracking
                options.enableNetworkTracking = false               // Privacy: No HTTP performance data
                options.enableCaptureFailedRequests = false         // Privacy: No HTTP error events

                options.enableAutoBreadcrumbTracking = false        // Caused widget crashes
                options.maxBreadcrumbs = 0                          // Explicit clarity

                options.enableSwizzling = false                     // Unnecessary overhead with all auto-instrumentation disabled
                options.enableDataSwizzling = false                 // Privacy: Monitors NSData file I/O operations

                options.sendClientReports = false                   // Privacy: SDK telemetry (sample rates, dropped events)

                #if os(iOS)
                options.enableMetricKit = false                     // iOS system reports hangs via MetricKit independently
                options.enablePreWarmedAppStartTracing = false      // Continuous app launch telemetry
                options.reportAccessibilityIdentifier = false       // Privacy: Exposes UI element structure
                #endif
            }
        #endif
    }
}
```

The two lines to notice are `enableSwizzling` and `enableDataSwizzling`. Swizzling is how the SDK does much of the work above: its own docs list touch and navigation breadcrumbs, view controller and HTTP instrumentation, and NSData file I/O under it. Turning off swizzling itself is more reliable than turning off each feature that uses it. `enableDataSwizzling` is a separate switch, on by default, so we turn it off too.

## Verification

A config file doesn't prove anything on its own. Check that nothing else ships:

1. **Build in Release mode** - Debug builds can behave differently
2. **Run on a real device** - The simulator can skip code paths
3. **Trigger a test crash** - Make sure it shows up in the dashboard
4. **Check for other events** - Make sure no sessions, hangs, breadcrumbs, or performance data show up

If something unexpected shows up in the dashboard, a setting is letting it through. Find it and turn it off.

## Platform Considerations

Our app runs on iOS, on watchOS, and in widgets, and the configuration has to hold on all three:

- Don't use UIKit-only options on watchOS
- Test widgets on their own; they have a different lifecycle
- Put the configuration in one shared helper so the three don't drift

## Results

- As of March 2026, crash reports come in with usable stack traces and device context, and nothing about what the user was doing.
- The cost is that a person reviews every SDK update. Without it, the collection would grow on its own.
- When we tell users "crash reports only", it's true.

## Lessons Learned

- **Turn off the mechanism, not just the feature.** A feature flag turned off above a running mechanism stops the report, not the collection.
- **Check what's missing, not just what's there.** The test crash shows reports get through. An otherwise empty dashboard, from a Release build on a real device, shows the config works.
- **Write down why each option is off.** A block of flags set to false with no comments looks like a mistake to the next person.
