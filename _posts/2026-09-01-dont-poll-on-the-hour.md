---
layout: post
title: "Don't Poll on the Hour"
date: 2026-09-01 16:00:00 -0600
summary: "Fourteen days of CDN logs showed one upstream source failing twenty times more often in the fifteen minutes after the top of the hour than after the half hour. The fix on paper is boundary-adjacent refresh targets with deterministic per-device jitter, and a cohort experiment to prove it. This is the measurement and the design record; the client change has not shipped."
tags: [performance, reliability, scheduling, ios, cloudfront]
---

## The Problem

Over fourteen days of CDN logs ending February 22, 2026, one upstream weather source returned a server error on 0.0205% of requests made between :00 and :14, and on 0.0009% of requests made between :30 and :44. A second source showed the same shape, 0.0262% against 0.0000%, and most of its failures were gateway timeouts. Nothing in the app's own traffic explains that asymmetry. It fans its background refreshes out over the same five minutes after :00 and after :30, so its own load on the two windows is identical.

The difference is everyone else. Clients across the industry schedule refreshes on the hour and half hour, and the providers' own best-practice pages warn against exactly that synchronized polling. The top of the hour is where the largest shared burst lands, and the half hour is not. [Hello Weather](https://helloweather.com) had already solved the version of this problem it could see. Its server sends a silent push to every device every thirty minutes, and since March 2025 it spreads those pushes over a random delay of up to five minutes so its own fleet is not a wall (see [Silent Push Refresh Without a Location Database](/apns-background-refresh/)). The fleet stopped hitting the app's servers in the same second. It kept hitting the vendors' servers in the same fifteen minutes as the rest of the world.

The assumption under the original schedule was that the boundary is where the freshest data lives, so the boundary is where a client should be. The measurement said the boundary is where the failures live, and the two facts are not in conflict: one provider documents a ten-minute model cadence, and data on that cadence is no staler at :05 than at :00.

## Measure by Minute of Hour

Error counts hide this. A source with ten times the traffic has ten times the errors, and a spike at :00 looks like a spike in demand rather than a spike in failure rate. The plan that recorded these findings starts from two rules: normalize by request volume, and separate transport failures from semantic responses a source returns on purpose. A 404 for an unsupported alert region is not an outage, and one vendor's migration in February 2026 had turned that response from a 204 into a 404, which is what started the investigation.

The mechanism is two views over the CDN access logs, which this app keeps in CloudWatch for every upstream source (see [CloudFront Logging](/cloudfront-logging/)). The first buckets fourteen days of requests into four fifteen-minute windows of the hour and reports the 5xx rate per source per window, weighted by volume. The second buckets five days into five-minute slots and looks for outliers.

| Window | Source A | Source B | Source C |
|--------|----------|----------|----------|
| :00 to :14 | 0.0205% | 0.0262%, mostly 504 | near zero |
| :30 to :44 | 0.0009% | 0.0000% | near zero |

The five-minute view is the one that changed the conclusion. Source A's largest spikes over the five days landed near :57. Source B's largest landed around :12. Source C stayed near zero in every slot. Neither hotspot sits on the boundary the app was avoiding, and neither sits on the boundary it was targeting. Whatever those vendors do at :57 and :12, a client that is told "spread out after :00" walks straight into the second one.

What the view cannot tell you is why. A spike at :57 could be a vendor's own pre-publish work, another large client's schedule, or a cache expiring. The plan does not guess. It records the hotspot per source and treats "the safe window differs by provider" as the finding, which is what makes per-source overrides part of the design below.

## What the Fleet Already Did

Both halves of the app already jitter. The server's enqueue job runs every thirty minutes and gives each device's push a random delay of zero to five minutes. It was tightened to four minutes in April 2025 so the last push landed no later than :05, then widened back to five in May 2025. In December 2025 the enqueue itself, thousands of job inserts in under a second, caused database latency spikes, and a throttle was added and later removed once the pool was unlimited (see [ActiveRecord Under a Fiber Reactor](/activerecord-under-a-fiber-reactor/)). The fanout is cheap on the wire and was not free on the database.

The client jitters too, in the one place it controls its own timing: the widget timeline. Since March 2025, an install with widgets asks for its next refresh at the boundary plus a random offset, trimmed here to the timing logic:

```swift
// Widget timeline policy: .after(nextRefreshTime())
func nextRefreshTime(lastFetch: Date, calendar: Calendar = .current) -> Date {
    var minTime = lastFetch.addingTimeInterval(1800) // 30 minutes
    if minTime < Date() { minTime = Date() }

    switch calendar.component(.minute, from: minTime) {
    case 0...5, 30...35:
        return minTime // already inside a fanout window
    default:
        let hour = calendar.component(.hour, from: minTime)
        let candidates = [
            calendar.date(bySettingHour: hour, minute: 0, second: 1, of: minTime),
            calendar.date(bySettingHour: hour, minute: 30, second: 1, of: minTime),
            calendar.date(bySettingHour: hour + 1, minute: 0, second: 1, of: minTime),
            calendar.date(bySettingHour: hour + 1, minute: 30, second: 1, of: minTime),
        ].compactMap { $0 }
        let target = candidates.first { $0 >= minTime } ?? minTime
        return target.addingTimeInterval(TimeInterval(Int.random(in: 0...300)))
    }
}
```

The push path has its own guard: a device woken by the silent push skips the fetch if it refreshed within the last 25 minutes, which is the 30-minute cycle minus the fanout, so a widget refresh and a push refresh in the same window cost one request, not two.

Two properties of this code are the problem. The window is anchored to the boundary itself, so the whole fleet lands in :00 to :05, inside the fifteen minutes the logs flagged. And the offset is drawn fresh every cycle, so an install that landed in a quiet slot this half hour has no memory of it, and the population re-rolls itself every thirty minutes. Random jitter spreads a burst. It does not move one.

## Move Off the Boundary, and Stop Re-Rolling

The proposed policy has four parts, and the plan is explicit that they apply to background refresh only. A manual refresh runs immediately, because a user pulling to refresh is not a burst and should never be held back to make a chart look better.

- Background refresh targets :05 and :35 rather than :00 and :30.
- Each device carries a deterministic offset of roughly two to three minutes either side, derived from a stable per-install value, so the same device lands in the same slot every cycle.
- Transport failures, 502, 504, and 429, get a bounded client retry with backoff and jitter.
- Statuses a source returns on purpose, the ones the adapter rules classify as expected, are never retried.

Why deterministic rather than random each cycle? The plan lists that as an open question, then designs the experiment around the deterministic form, and three reasons favor it. A stable offset makes the cohort experiment measurable, because a device is in one arm for the whole test instead of drifting across the window. It stops the population from re-rolling, so a slot that was quiet stays quiet for the devices in it. And it costs nothing: the offset is a hash, not a stored setting.

```swift
import Foundation

/// The same install always gets the same offset. Swift's `Hasher` is seeded
/// per process, so `hashValue` will not do; FNV-1a is stable and dependency-free.
func stableJitter(installID: String, spreadSeconds: Int) -> TimeInterval {
    var hash: UInt64 = 0xcbf29ce484222325
    for byte in installID.utf8 {
        hash ^= UInt64(byte)
        hash = hash &* 0x100000001b3
    }
    let offset = Int(hash % UInt64(spreadSeconds * 2 + 1)) - spreadSeconds
    return TimeInterval(offset)
}

/// Next background refresh: :05 or :35, shifted by this install's fixed offset.
func nextBackgroundRefresh(after date: Date, installID: String,
                           calendar: Calendar = .current) -> Date {
    let jitter = stableJitter(installID: installID, spreadSeconds: 150) // ±2.5 min
    let thisHour = calendar.dateInterval(of: .hour, for: date)!.start
    let nextHour = calendar.date(byAdding: .hour, value: 1, to: thisHour)!
    let targets = [thisHour, nextHour].flatMap { hour in
        [5, 35].map { hour.addingTimeInterval(TimeInterval($0 * 60) + jitter) }
    }
    return targets.first { $0 > date } ?? nextHour
}
```

The spread is the same five minutes the fleet uses today, shifted so its earliest edge is :02:30 rather than :00:01. Nothing about the population's width changes; only where it sits. The install identifier has to be something the app already keeps and never sends, since the point of the push design is that the server knows nothing about devices beyond a token.

What the code cannot enforce is the retry budget. A bounded retry on 504 is a second request to a source that just timed out, during the window it is most likely to time out again. The plan lists "source-hit amplification" as one of the four things each cohort is measured on, and the server side of this app has already moved in the opposite direction, cutting its own transport retries to one attempt for the same reason (see [Retries Your Vendor Already Billed](/retries-your-vendor-already-billed/)). A client retry that the server then retries is four hits for one forecast.

## Prove It With Cohorts

The plan refuses to ship the new targets fleet-wide on the strength of the logs. Two windows differing by a factor of twenty is a strong signal, but it is a signal about the world's traffic, and the app's own change might not move it at all. So the rollout is an experiment:

- **Cohort A, control.** The current schedule: boundary targets, random fanout.
- **Cohort B.** :05 and :35 targets with deterministic jitter.
- **Cohort C, optional.** Per-source timing overrides where the five-minute view shows a hotspot worth dodging, such as Source B's :12.

Each cohort is measured on user-visible failure rate, source 5xx and 504 rate, p95 latency, and source-hit amplification. The decision rule is one sentence: promote only if the failure reduction is meaningful without a material increase in latency or upstream request volume. The sequencing rule is borrowed from the rest of the plan: ship instrumentation first, collect seven days of baseline, then one mitigation at a time, and compare seven days before and after over the same windows with traffic normalized.

The limit is that Cohort C is only as good as the hotspot data, and hotspots move. The :57 and :12 spikes came from five days of logs. A per-source override tuned to them is a bet that the vendor's behavior is stable, and the plan marks it optional for that reason.

## Results

Nothing in the client has changed. As of September 1, 2026, the widget timeline still targets :00 and :30 with a random offset, the push path still uses its 25-minute guard, and there is no client retry policy. The plan's state is "Planning (trigger-gated)", last updated July 23, 2026, and the minute-of-hour dashboard it asked for in its first phase was never built; the numbers above came from ad hoc queries over the logs during the February session.

What did ship is the measurement, and it changed a belief. Before February 22 the working model was "the boundary is fresh, our own burst is the risk, so fan out after the boundary." After it, the model is "the boundary is where everyone's burst lands, hotspots are per-source and off-boundary, and our fanout puts us inside the bad window." The random fanout from 2025 stays, and it cost one database incident and a 25-minute guard on the push path.

The cost of the proposal, if it ships, is accepted up front: a background refresh that lands at :07 instead of :02 shows data five minutes older on a widget, for sources whose own update cadence is measured in minutes.

## Lessons Learned

- **Normalize by volume before reading a spike.** A burst in demand and a burst in failure rate look identical on a count chart.
- **Random jitter spreads a burst; it does not move one.** If the whole window is bad, widen nothing and shift the anchor.
- **Derive per-device offsets from a stable value.** Deterministic jitter keeps devices in one experimental arm and stops the population re-rolling each cycle.
- **Hold manual actions out of traffic shaping.** A user's tap is not a burst, and a deliberate delay on it is a bug users can see.
- **Count retries as upstream volume.** Every layer that retries multiplies the others, and the window you retry in is the one most likely to fail again.

---

## How This Post Was Made

**Prompt 1:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

**Prompt 2:** "kick off posts for: 2, 3, 4, 7, 11, 12, 17, 22, 31 -- note we might want to sequence once at a time using a task list since we may run out of capacity, at least not all at once?"

Generated by Claude Fable 5.1 using the blog-post-generator skill. One agent researched the web repository and proposed this post as a candidate; a second agent verified the claims and wrote it. Sources: `plans/data-source-reliability-investigation.md` (PR #1230, February 22, 2026, status mirror July 23, 2026), `app/jobs/apns_token_ping_enqueue_job.rb` and its history (random fanout added March 8, 2025; four-minute window April 3, 2025; enqueue throttle added December 17, 2025 and removed December 19, 2025), `config/recurring.yml`, and in the iOS repository `SettingsManager.swift` (boundary targeting with random fanout, March 11, 2025), `PushService.swift` (the 25-minute ping guard), and the widget and watch timeline providers. Judgment calls: the three sources are anonymized as A, B, and C and the vendor whose status change started the investigation is not named; the error rates are presented as this app's observed rates over its own CDN, not as vendor service levels; the device population size recorded in the December 2025 commit is omitted; the client excerpt is trimmed to the timing logic and takes the last fetch time as a parameter rather than reading it from a manager; the deterministic jitter excerpt is a proposal written for this post, since no such code exists in the client, and the post says so; the "weighted" windows are described as weighted by volume, following the plan's own rule to normalize by request volume.
