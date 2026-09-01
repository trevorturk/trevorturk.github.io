---
layout: post
title: "Sync Only What the Watch Reads"
date: 2026-07-29 09:40:00 -0600
summary: "Auditing every settings key in a phone-to-watch sync payload, inverting a denylist into an opt-in allowlist, and defusing a protocol where a missing key gave away the paid app for free."
tags: [swift, watchos, sync, ios]
---

## The Problem

Adding a preview cache for a phone-only screen meant adding one key to the shared `UserDefaults` store. That key immediately started shipping to the Apple Watch on every complication refresh. [Hello Weather](https://helloweather.com) syncs settings from the phone to the watch over WatchConnectivity, and the manifest for that sync was the same enum that lists the storage keys:

```swift
enum Keys: String, CaseIterable {
    case temperatureUnit
    case displayLanguage
    case memberEntitlement
    // ...and eighty-some more
}

nonisolated var keys: [String] {
    Keys.allCases.map { $0.rawValue }
}

func getDictionaryRepresentation() -> [String: Any] {
    store.dictionaryRepresentation().filter { (key, _) in keys.contains(key) }
}
```

One enum listed every key in the shared app-group `UserDefaults`, and `CaseIterable` let it double as the watch sync manifest. Every key registered for *storage* was automatically enrolled in *transfer*, and the whole dictionary shipped on every `updateApplicationContext` and every complication refresh.

Four years of feature work later, the payload was about 90 keys and rising. Riding along on every complication update:

- A purchase-transaction cache that could hold thousands of records
- Saved and recent location arrays the watch **cannot decode** — the manager that models them isn't compiled into the watch target
- Push tokens, delivered-alert bookkeeping, and notification scheduling state
- Radar map state, onboarding progress, review-nag timestamps, migration bookkeeping
- A per-process fetch mutex — a timestamp meaning "a fetch is running *in this process*" — copied across devices, where it can suppress a fetch on the other one

The preview cache made the shape of the bug plain: storage registration was implying sync enrollment, and nowhere in the codebase was anyone asked to decide.

## The Audit

The fix isn't interesting. The method is.

Before writing any code, we mapped every one of the ~90 keys to actual reads and writes in the watch-compiled source set. "Does the watch have this file" was not the question. "Is this file a member of the watch target" was, using the target's file-membership exceptions in `project.pbxproj` as ground truth. Each key got one of three verdicts: read on watch, written on watch, or neither.

Roughly 25 keys belonged. About 50 were pure payload, bytes that had never been read on the other side of the connection. The rest were judgment calls that only surfaced because every key needed a written reason next to it.

Three findings a quick eyeball would have gotten wrong, in both directions:

- **Six notification toggles looked phone-only. They aren't.** Each maps to an iOS notification category, so excluding them was the obvious call. But a helper ORs all six into a single "notifications on?" boolean, and the watch's location service branches on it: `requestAlwaysAuthorization()` versus `requestWhenInUseAuthorization()`. Dropping six booleans would have silently downgraded the watch's location authorization request for every user with notifications enabled.
- **A key written on the watch but never read there.** The watch stores the device location, then geocodes its own copy anyway. Every reader of the stored value is phone-only UI. It is a genuine trim candidate, and we kept it, because dropping it is an unforced behavior change with no upside. "Unused" and "safe to remove in this PR" are different questions.
- **A flag whose only reader is compiled out today.** The feature it gates isn't available on watchOS yet, so the read sits inside a `#if canImport(...)` that never fires there. But the code path that consults it runs unconditionally in watch context, so the key goes live the day the framework arrives. Enrolled ahead of time.

We also wrote down the high-risk keys explicitly, because their failure modes are silent rather than loud: the entitlement flags, the language key (a localization helper reads the store directly, so losing it reverts the whole watch to English), the chart-style key (a style resolver reads the store directly, bypassing the settings manager), and the API parameter keys the watch uses to build its own forecast URL.

## The Inversion

Two mechanics of the old protocol shaped the design, and both had to be verified rather than assumed.

The first: the payload unions registered defaults. `dictionaryRepresentation()` returns every key that has a `registerDefaults` entry, whether or not a user ever touched it. The sync was not "everything the user changed" but "everything the app ever declared."

The second: absence meant deletion. The apply side cleared *every* known key before writing the incoming values:

```swift
nonisolated func setDictionaryRepresentation(_ dictionary: [String: Any]) {
    for key in keys {
        store.removeObject(forKey: key)   // wipe everything...
    }
    for (key, val) in dictionary {
        store.set(val, forKey: key)       // ...then restore what arrived
    }
}
```

A key missing from the payload was erased on the watch. And on this codebase, absence has semantics:

```swift
var paidWatch: Bool {
    if store.object(forKey: SavedDataManager.Keys.memberEntitlement.rawValue) == nil {
        return true          // no entitlement key at all: treat as paid
    } else {
        return paid || legacyMember
    }
}
```

That `return true` is deliberate — it keeps the watch app working during the window before the first sync lands, rather than flashing a paywall at a paying customer. But combine it with wipe-then-apply and you get a landmine: **any allowlist that omits the entitlement key hands out the paid watch app for free.** Get the polarity backwards on a different key and you revoke it from someone who paid.

So the allowlist and the deletion semantics had to change in the same commit. The shipped shape:

```swift
enum Keys: String {
    // ...all ~90 storage keys...

    // Watch sync is opt-in: see plans/watch-sync-allowlist.md before adding here.
    static let synced: Set<Keys> = [
        .weather,
        .selectedLocation,
        .memberEntitlement,
        .legacyMember,
        .temperatureUnit,
        .displayLanguage,
        .chartStyle,
        // ...36 in total
    ]
}

nonisolated private static let syncedKeys = Set(Keys.synced.map(\.rawValue))

func syncPayload() -> [String: Any] {
    store.dictionaryRepresentation().filter { (key, _) in Self.syncedKeys.contains(key) }
}

nonisolated func applySyncPayload(_ payload: [String: Any]) {
    for key in Self.syncedKeys {
        store.removeObject(forKey: key)
    }

    for (key, val) in payload where Self.syncedKeys.contains(key) {
        store.set(val, forKey: key)
    }
}
```

Three changes, each doing distinct work:

1. **The allowlist is opt-in.** Adding a storage key no longer enrolls it in anything. Enrollment is a deliberate, reviewed act with a comment pointing at the reasoning.
2. **Clear only synced keys.** Exclusion now means "watch-local," never "delete." Unsynced keys the watch owns, its own fetch mutex and its own local bookkeeping, survive a sync instead of being wiped by it.
3. **Apply only synced keys.** The filter runs on the receiving side too. During the version-skew window an older phone still sends the full 90-key payload, and without the receive-side filter it would smuggle the excluded keys straight back onto the watch.

`CaseIterable` came off the enum as well. There were zero remaining uses, and `allCases` was the exact footgun being removed. Leaving it available arms the trap for whoever needs "a list of all the keys" next.

The allowlist is covered by tests, and the one that earns its place is not the count assertion:

```swift
@Test("Local-only keys stay out of the payload")
func localOnlyKeysExcluded() {
    let excluded: [SavedDataManager.Keys] = [
        .debugMode, .migratedVersion, .previewCache,
        .fetchInProgressAt, .savedPlaces, .recentPlaces,
    ]
    for key in excluded {
        #expect(SavedDataManager.Keys.synced.contains(key) == false,
                "\(key.rawValue) must not sync to the watch")
    }
}

@Test("syncPayload filters to the allowlist")
func syncPayloadFiltersToAllowlist() {
    let store = SavedDataManager.shared.store
    let marker = "testOnlyUnknownKey"
    store.set("junk", forKey: marker)
    defer { store.removeObject(forKey: marker) }

    let payload = SavedDataManager.shared.syncPayload()

    #expect(payload[marker] == nil)
    #expect(payload[SavedDataManager.Keys.temperatureUnit.rawValue] != nil)
}
```

A count test says "36." The exclusion test says why. It names the keys whose presence would be a bug and fails with a sentence a future reader can act on.

## The Bug the Audit Found

Halfway through mapping keys to reads, one key came back with an answer that fit none of the categories: the data-source preference was **written on the watch**.

The sync is one-directional. The watch's `sync()` only reloads complications, and its `requestSync()` *pulls* phone state. There is no watch-to-phone data path at all. So a source picked on the watch was reverted by the next phone sync, every time. The write went into the shared store, looked like it worked, and was wiped the next time the phone said anything.

The reflex is to build the missing direction. We didn't, for two reasons.

First, nobody could reach the picker. The view had been unreachable since 2024, when the button that presented it was turned into a display-only label. The bug was real but latent: dead code with a live-looking failure mode, sitting in the repo for nearly two years.

Second, even if it *had* been reachable, "add a watch-to-phone settings channel" is a transport project, not a bug fix. Building a reverse sync channel to serve one picker nobody had asked for would have been the wrong order of work.

So we deleted the picker. The read-only source label stays; the dead view is gone. The restore path is written down instead of built: watch source selection is a phase of the watch-parity plan, gated on the watch-to-phone settings channel that a different plan owns, and if it comes back it comes back with the Automatic option the phone has.

A control that appears to work and silently reverts is worse than no control. When you find one, the choice is fix the plumbing or remove the control, and removing it is legitimate as long as you write down what restoring it would require.

## Results

The payload went from ~90 keys to 36, and the excluded set is where the weight was: the transaction cache, both location arrays, the delivered-alerts record, the preview cache. Transfers on the complication path carry the settings the watch reads and nothing else.

The entitlement landmine is defused. The allowlist can no longer wipe a key it doesn't list, so an omission is now a missing-setting bug instead of a free-paid-app bug. That change mattered more than the bytes: a silent revenue failure became a visible, boring one.

Two riders came out of the audit. A key that had been a hardcoded literal since 2024 and never touched the store at all was deleted. And the watch, on first touch of a shared manager, was writing an iOS paywall timestamp into the shared store. Under wipe-everything semantics the next sync erased it; under clear-only-synced it would persist. The write is now compiled out with `#if !os(watchOS)`.

One consequence we accepted rather than fixed: watches that already synced the ~50 excluded keys keep them frozen in their local store forever. Nothing sends them and nothing clears them anymore, so the win is transfer-only. It is correctness-neutral, since none of them is read on the watch, and a one-time purge is written down as an optional follow-up rather than shipped speculatively.

The inversion also has a cost that only shows up months later. Opt-in means a new key that the watch genuinely needs will silently not arrive. So the feature-flags skill grew a sixth touch point:

| Step | Pattern |
|---|---|
| Watch sync | **Only if watch-compiled code reads the flag**: enroll it in `Keys.synced`, else skip — keys do NOT sync automatically |

When you invert a default, the failure mode moves. Denylist fails by shipping too much. Allowlist fails by shipping too little, quietly, in a target you're not looking at. That is a better failure to have, and it still has to be written down where the next person will hit it.

## Lessons Learned

- **"Absence means deletion" is a dangerous protocol default.** Wipe-then-apply is the easy way to make a sync converge, and it turns every omission into a destructive operation. Scope the wipe to exactly the keys you're authoritative for.
- **Check what absence *means* on the receiving side.** A missing key is rarely neutral. Ours meant "paid." Somewhere in your codebase, a `nil` check has a default written for a different situation than the one your sync protocol creates.
- **Allowlist over denylist for anything crossing a device boundary.** Payloads grow by accident, by someone adding a storage key for an unrelated screen. Under an allowlist that's a no-op; under a denylist it's a silent enrollment.
- **Ground the audit in the build system, not the file tree.** "Does the watch read this?" is a question about target membership. Grepping the repo would have gotten several keys wrong.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and plan docs, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the preview-cache key that exposed the bug, the title lost its subtitle, the three audit findings became a list, the bolded closer in the dead-picker section became plain prose, and Lessons Learned dropped the three bullets the body already states. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
