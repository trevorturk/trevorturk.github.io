---
layout: post
title: "Sync Only What the Watch Reads"
date: 2026-07-29 09:40:00 -0600
summary: "We checked every settings key the phone sends to the watch, replaced the send-everything list with an opt-in one, and fixed a sync rule where a missing key unlocked the paid watch app for anyone."
tags: [swift, watchos, sync, ios]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

We added a preview cache for a screen that only exists on the phone. That meant one new key in the shared `UserDefaults` store, and the key started shipping to the Apple Watch on every complication refresh. [Hello Weather](https://helloweather.com) sends settings from the phone to the watch over WatchConnectivity, and the list of what to send was the same enum that lists the storage keys:

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
    store.dictionaryRepresentation().filter { (key, val) in keys.contains(key) }
}
```

The enum listed every key in the shared app-group `UserDefaults`. Because it was `CaseIterable`, it also served as the list of keys to sync. So any key we stored was also a key we sent, and the whole dictionary went out on every `updateApplicationContext` call and every complication refresh.

After four years of feature work, the payload was about 90 keys and still growing. Riding along on every complication update:

- A purchase-transaction cache that could hold thousands of records
- Saved and recent location arrays the watch **cannot decode**, because the manager that models them isn't compiled into the watch target
- Push tokens, delivered-alert bookkeeping, and notification scheduling state
- Radar map state, onboarding progress, review-nag timestamps, migration bookkeeping
- A per-process fetch lock, a timestamp that means "a fetch is running *in this process*", copied to the other device, where it can stop that device from fetching

The preview cache made the bug easy to see. Storing a key enrolled it in sync, and nobody had ever been asked whether it should.

## The Audit

The code change is small. The work was deciding what goes on the list.

Before writing any code, we mapped each of the ~90 keys to the actual reads and writes in code that compiles into the watch. The question wasn't "Does the watch have this file". It was "Is this file a member of the watch target", and we used the target's file-membership exceptions in `project.pbxproj` as the source of truth. Each key got one of three verdicts: read on the watch, written on the watch, or neither.

About 25 keys belonged. About 50 had never been read on the watch side. The rest needed a decision, and we only noticed them because every key needed a written reason next to it.

Three findings a quick look would have gotten wrong, in both directions:

- **Four notification toggles looked phone-only. They aren't.** Each maps to an iOS notification category, so leaving them out looked obvious. But a helper ORs all six push toggles into one "notifications on?" boolean, and the watch's location service uses that boolean to choose between `requestAlwaysAuthorization()` and `requestWhenInUseAuthorization()`. Dropping four booleans would have quietly downgraded the watch's location permission request for everyone with notifications on.
- **A key written on the watch but never read there.** The watch stores the device location, then geocodes its own copy anyway. Everything that reads the stored value is phone-only UI. We could have trimmed it, and we kept it, because dropping it changes behavior for no gain. "Unused" and "safe to remove in this PR" are different questions.
- **A flag whose only reader is compiled out today.** The feature it gates isn't on watchOS yet, so the read sits inside a `#if canImport(...)` that never fires there. But the code path that checks it runs on the watch, so the key goes live the day the framework arrives. We enrolled it ahead of time.

We also wrote down the high-risk keys, because those fail quietly. They are the entitlement flags, the language key (a localization helper reads the store directly, so losing it puts the whole watch back in English), the chart-style key (a style resolver reads the store directly and skips the settings manager), and the API parameter keys the watch uses to build its own forecast URL.

## The Inversion

Two details of the old protocol shaped the design, and we checked both instead of assuming them.

First, the payload included registered defaults. `dictionaryRepresentation()` returns every key with a `registerDefaults` entry, whether or not a user ever touched it. So the sync wasn't "everything the user changed". It was "everything the app ever declared."

Second, a missing key meant delete. The apply side cleared every known key before writing the incoming values:

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

A key missing from the payload was erased on the watch. And in this codebase, a missing key has a meaning:

```swift
var paidWatch: Bool {
    if store.object(forKey: SavedDataManager.Keys.memberEntitlement.rawValue) == nil {
        return true          // no entitlement key at all: treat as paid
    } else {
        return paid || legacyMember
    }
}
```

That `return true` is on purpose. It keeps the watch app working before the first sync lands, instead of showing a paywall to a paying customer. Combine it with wipe-then-apply, though, and **any allowlist that leaves out the entitlement key gives everyone the paid watch app without paying.** Get the polarity backwards on a different key and you take it away from someone who paid.

So the allowlist and the delete rule had to change in the same commit. The shipped shape:

```swift
enum Keys: String {
    // ...all ~90 storage keys...

    // Watch sync is opt-in: see the watch-app skill before adding here (#1336).
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

Three changes, each doing its own job:

1. **The allowlist is opt-in.** Adding a storage key no longer enrolls it in anything. Enrolling is a deliberate, reviewed change with a comment pointing at the reasoning.
2. **Clear only synced keys.** Leaving a key off the list now means "watch-local", never "delete". Keys the watch owns, like its own fetch lock and its own local bookkeeping, survive a sync instead of being wiped by it.
3. **Apply only synced keys.** The receiving side filters too. During the version-skew window, an older phone still sends the full 90-key payload, and without the receive-side filter those excluded keys would land right back on the watch.

We also took `CaseIterable` off the enum. Nothing used it anymore, and `allCases` was the exact mistake we were removing. Leaving it there sets up the same trap for whoever needs "a list of all the keys" next.

Tests cover the allowlist, and the useful one isn't the count assertion:

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
    #expect(payload.keys.allSatisfy { key in
        SavedDataManager.Keys.synced.contains { $0.rawValue == key }
    })
    #expect(payload[SavedDataManager.Keys.temperatureUnit.rawValue] != nil)
}
```

A count test says "36." The exclusion test says why. It names the keys that would be a bug if present, and it fails with a sentence the next reader can act on.

## The Bug the Audit Found

Halfway through mapping keys to reads, one key came back with an answer that fit none of the three verdicts: the data-source preference was **written on the watch**.

The sync goes one way. The watch's `sync()` only reloads complications, and its `requestSync()` pulls phone state. There is no path from the watch to the phone at all. So a source picked on the watch was undone by the next phone sync, every time. The write went into the shared store, looked like it worked, and was wiped the next time the phone sent anything.

The reflex is to build the missing direction. We didn't, for two reasons.

First, nobody could reach the picker. The view had been unreachable since 2024, when the button that presented it became a display-only label. The bug was real but latent: dead code with a live-looking failure, sitting in the repo for nearly two years.

Second, even if the picker had been reachable, "add a watch-to-phone settings channel" is a transport project, not a bug fix. Building a reverse channel for one picker nobody had asked for would have been the wrong order of work.

So we deleted the picker. The read-only source label stays, and the dead view is gone. The way back is written down instead of built: watch source selection is a phase of the watch-parity plan, and it waits on the watch-to-phone settings channel that a different plan owns. If it comes back, it comes back with the Automatic option the phone has.

A control that looks like it works and then quietly reverts confuses people more than no control at all. We had two options, fix the plumbing or remove the control, and removing it is fine as long as we write down what putting it back would take.

## Results

The payload went from ~90 keys to 36, and the excluded set held most of the weight: the transaction cache, both location arrays, the delivered-alerts record, the preview cache. Transfers on the complication path now carry the settings the watch reads and nothing else.

The entitlement bug is defused. The allowlist can't wipe a key it doesn't list anymore, so leaving one off is now a missing-setting bug instead of an unlocked-watch-app bug. That mattered more than the bytes. A quiet revenue failure became a visible, boring one.

Two smaller fixes came out of the audit. A key that had been a hardcoded literal since 2024 and never touched the store was deleted. And the watch, the first time it touched a shared manager, was writing an iOS paywall timestamp into the shared store. Under wipe-everything the next sync erased it. Under clear-only-synced it would stick around. The write is now compiled out with `#if !os(watchOS)`.

One thing we accepted instead of fixing: watches that already synced the ~50 excluded keys keep them in their local store forever. Nothing sends them and nothing clears them anymore, so the win is transfer-only. It doesn't affect correctness, since none of them is read on the watch, and a one-time purge is written down as an optional follow-up rather than shipped on a guess.

The inversion has a cost that only shows up months later. Opt-in means a new key the watch really needs will quietly not arrive. So the feature-flags skill grew a sixth touch point:

| Step | Pattern |
|---|---|
| Watch sync | **Only if watch-compiled code reads the flag**: enroll it in `Keys.synced`, else skip — keys do NOT sync automatically |

Flipping the default moves the failure. A denylist fails by sending too much. An allowlist fails by sending too little, quietly, in a target you're not looking at. We'd rather have the second failure, and it still has to be written down where the next person will hit it.

## Lessons Learned

- **Don't let a missing key mean delete.** Wipe-then-apply is the easy way to make a sync converge, and it turns every omission into a deletion. Clear only the keys you're the source of truth for.
- **Check what a missing key means on the receiving side.** It's rarely neutral. Ours meant "paid." Somewhere in your codebase, a `nil` check has a default written for a different situation than the one your sync creates.
- **Use an allowlist for anything crossing a device boundary.** Payloads grow by accident, when someone adds a storage key for an unrelated screen. Under an allowlist that's a no-op. Under a denylist it's a quiet enrollment.
- **Ground the audit in the build system, not the file tree.** "Does the watch read this?" is a question about target membership. Grepping the repo would have gotten several keys wrong.
