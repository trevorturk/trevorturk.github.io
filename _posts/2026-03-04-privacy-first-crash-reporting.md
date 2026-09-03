---
layout: post
title: "Privacy-First Crash Reporting"
date: 2026-03-04 08:00:00 -0600
summary: "A philosophy for minimal crash reporting: collect only stack traces, audit SDK updates for hidden telemetry, and verify nothing extra ships."
tags: [ios, privacy, sentry, mobile]
model: "Claude Opus 4.5"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

## The Problem

A crash reporting SDK on its defaults collects performance metrics, user sessions, network requests, breadcrumbs, and interaction traces, most of them enabled out of the box. For a privacy-focused app that is the wrong trade. We wanted crash reports, not an accidental user analytics platform.

## The Philosophy

Crashes only, nothing else.

Collect:
- Crash stack traces
- Device model and OS version (non-PII context)

Don't collect:
- Sessions or performance telemetry
- User interactions or breadcrumbs
- Network requests or timing
- App hangs or diagnostic reports
- Any form of analytics or metrics

## The Challenge: SDK Updates

A configuration that is correct today can break on the next dependency update. SDKs add features, and some arrive enabled by default. Worse, disabling a high-level feature does not always disable the collection mechanism under it. Turning off `enableAutoPerformanceTracing` might not disable `enableDataSwizzling`. The infrastructure behind performance tracing keeps running and just stops reporting.

## Evaluation Protocol for SDK Updates

Four checks before updating the SDK.

### 1. Check the changelog for defaults

Search the changelog for "enabled by default" and for anything new in telemetry: tracking, monitoring, breadcrumbs, metrics, swizzling. Each is a change to investigate before the update lands.

### 2. Audit mechanisms, not just feature flags

A disabled feature can leave its infrastructure active. Search the SDK source for:
- Swizzling or method interception
- Timer or observer registration
- Network monitoring hooks
- File system observers

An active mechanism is collecting data somewhere, whether or not anything is sending it yet.

### 3. Watch for these red flags

Scrutinize or explicitly disable anything involving:

- Performance monitoring / tracing
- User interaction tracking
- Session replay or recording
- Diagnostic reports / MetricKit integration
- Network request monitoring
- Breadcrumb collection
- Any form of analytics or metrics
- Swizzling or other data collection infrastructure

### 4. Default to off

When it is unclear whether a feature collects user data, disable it. A feature can be enabled later. Data already sent cannot be un-collected.

## Configuration Example

This is the shared Sentry helper (sentry-cocoa 9.x), called from the iOS app, the watch app, and the widget providers. The DSN is removed and a crash-ticket reference is trimmed from one comment; otherwise it is the real file. The principles apply to any crash reporting SDK:

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

`enableSwizzling` and `enableDataSwizzling` are the lines to notice. Swizzling is the mechanism behind many of the features above (the SDK's own docs list touch and navigation breadcrumbs, view controller and HTTP instrumentation, and NSData file I/O under it), and turning it off at the infrastructure level is more reliable than turning off each feature that depends on it. `enableDataSwizzling` is a separate switch with its own default of on, so it is set off as well.

## Verification

Configuration alone proves nothing. Confirm that nothing extra ships:

1. **Build in Release mode** - Debug builds may behave differently
2. **Run on a real device** - Simulators may skip certain code paths
3. **Trigger a test crash** - Confirm it appears in your dashboard
4. **Check for other events** - Confirm NO sessions, hangs, breadcrumbs, or performance data appear

Anything unexpected in the dashboard points at a setting. Find it and disable it.

## Platform Considerations

An app that runs on iOS, watchOS, and widgets needs the configuration to hold on all three:

- Avoid UIKit-specific options on watchOS
- Test widgets separately - they have different lifecycle
- Use a shared configuration helper to prevent drift

## Results

- As of March 2026, crash reports arrive with usable stack traces and device context, and no user behavior data.
- The cost is a manual review of every SDK update. Collection no longer expands silently, and reviewer time is the price.
- "Crash reports only" means exactly that to users.

## Lessons Learned

- **Disable the mechanism, not just the feature.** A flag turned off above a running mechanism suppresses the report, not the collection.
- **Uncertainty resolves toward off.** Off can become on later. Sent data cannot be recalled.
- **Verify by absence.** The test crash proves the pipeline. The empty rest of the dashboard, in Release on hardware, proves the configuration.
- **Write down why each option is off.** A block of flags set to false with no comment reads as an oversight to the next maintainer.
