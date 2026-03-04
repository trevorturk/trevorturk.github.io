---
layout: post
title: "Privacy-First Crash Reporting"
date: 2026-03-04 08:00:00 -0600
summary: "A philosophy for minimal crash reporting: collect only stack traces, audit SDK updates for hidden telemetry, and verify nothing extra ships."
tags: [ios, privacy, sentry, mobile]
---

## The Problem

Crash reporting SDKs want to help you. They'll collect performance metrics, user sessions, network requests, breadcrumbs, and interaction traces. Most of this ships enabled by default.

For a privacy-focused app, this is a problem. You want crash reports. You don't want to accidentally ship a user analytics platform.

## The Philosophy

**Crashes only. Nothing else.**

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

Crash reporting SDKs evolve. New features get added. Some get enabled by default. Your carefully configured privacy settings can break with a single dependency update.

**The trap:** Disabling high-level features doesn't always disable underlying collection mechanisms.

For example, turning off `enableAutoPerformanceTracing` might not disable `enableDataSwizzling` - the infrastructure that makes performance tracing possible is still running, just not reporting.

## Evaluation Protocol for SDK Updates

Before updating your crash reporting SDK:

### 1. Check the changelog for defaults

Look for phrases like "enabled by default", "now automatically", or "improved telemetry". These are red flags that require investigation.

### 2. Audit mechanisms, not just feature flags

Don't trust that disabling a feature disables its infrastructure. Search the SDK source for:
- Swizzling or method interception
- Timer or observer registration
- Network monitoring hooks
- File system observers

If the mechanism is active, data is being collected somewhere - even if it's not being sent yet.

### 3. Watch for these red flags

Scrutinize or explicitly disable anything involving:

- Performance monitoring / tracing
- User interaction tracking
- Session replay or recording
- Diagnostic reports / MetricKit integration
- Network request monitoring
- Breadcrumb collection
- Any form of analytics or metrics
- "Improved crash context" (often means more data collection)

### 4. Default to off

If you're uncertain whether a feature collects user data, disable it. You can always enable it later if needed. You can't un-collect data that's already been sent.

## Configuration Example

Here's how we configure Sentry for iOS - the same principles apply to any crash reporting SDK:

```swift
SentrySDK.start { options in
    options.dsn = "your-dsn"

    // Core crash reporting only
    options.enableCrashHandler = true

    // Disable everything else explicitly
    options.enableAutoPerformanceTracing = false
    options.enableUIViewControllerTracing = false
    options.enableNetworkTracking = false
    options.enableFileIOTracing = false
    options.enableCoreDataTracing = false
    options.enableSwizzling = false  // Critical: disables the mechanism
    options.enableAutoBreadcrumbTracking = false
    options.enableNetworkBreadcrumbs = false
    options.attachScreenshot = false
    options.attachViewHierarchy = false
    options.enableMetricKit = false
    options.enableTimeToFullDisplayTracing = false

    // No session tracking
    options.enableAutoSessionTracking = false
    options.sessionTrackingIntervalMillis = 0
}
```

The key insight: we disable `enableSwizzling` entirely. This is the mechanism that powers many features. Disabling it at the infrastructure level is more reliable than disabling individual features that depend on it.

## Verification

Configuration isn't enough. Verify that nothing extra ships:

1. **Build in Release mode** - Debug builds may behave differently
2. **Run on a real device** - Simulators may skip certain code paths
3. **Trigger a test crash** - Confirm it appears in your dashboard
4. **Check for other events** - Confirm NO sessions, hangs, breadcrumbs, or performance data appear

If anything unexpected shows up, investigate which setting is responsible and disable it.

## Platform Considerations

If your app runs on multiple platforms (iOS, watchOS, widgets), ensure your crash reporting configuration works everywhere:

- Avoid UIKit-specific options on watchOS
- Test widgets separately - they have different lifecycle
- Use a shared configuration helper to prevent drift

## Results

With this approach:
- Crash reports arrive with useful stack traces and device context
- No user behavior data is collected
- SDK updates require review but don't silently expand data collection
- Users can trust that "crash reports only" means exactly that

## Lessons Learned

- **Audit mechanisms, not features** - Disabling a feature doesn't disable its infrastructure
- **Default to off** - Enable features deliberately, not by SDK default
- **Verify in production builds** - Debug builds may behave differently
- **Review every SDK update** - New defaults can silently expand collection
- **Document your philosophy** - Future maintainers need to know why settings are disabled

---

## How This Post Was Made

**Prompt:** "let's write (one or more) posts about the skills we have in helloweather web and ios. I'm thinking perhaps one about sentry in the ios repo, where we document how we want to maintain privacy and beware of new settings that might be enabled by default, to ensure we only do the minimal crash reporting and respect privacy."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Based on the iOS Sentry skill from helloweather/ios, generalized to apply to any crash reporting SDK.
