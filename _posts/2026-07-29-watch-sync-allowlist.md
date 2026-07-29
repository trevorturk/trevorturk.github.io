---
layout: post
title: "Sync Only What the Watch Reads: An Allowlist Inversion"
date: 2026-07-29 09:40:00 -0600
summary: "Auditing every settings key in a phone-to-watch sync payload, inverting a denylist into an opt-in allowlist, and defusing a protocol where a missing key gave away the paid app for free."
tags: [swift, watchos, sync, ios]
---

## The Problem

[Hello Weather](https://helloweather.com) syncs settings from the phone to the Apple Watch over WatchConnectivity. The implementation looked reasonable when it was written:

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

One enum listed every key in the shared app-group `UserDefaults`. That same enum silently doubled as the watch sync manifest. Every key registered for *storage* was automatically enrolled in *transfer*, and that dictionary shipped on every `updateApplicationContext` and every complication refresh.

Nobody decided this. It's what `CaseIterable` does when you use it as a schema.

Four years of feature work later, the payload was about 90 keys and rising. Riding along on every complication update:

- A purchase-transaction cache that could hold thousands of records
- Saved and recent location arrays the watch **cannot decode** — the manager that models them isn't compiled into the watch target
- Push tokens, delivered-alert bookkeeping, and notification scheduling state
- Radar map state, onboarding progress, review-nag timestamps, migration bookkeeping
- A per-process fetch mutex — a timestamp meaning "a fetch is running *in this process*" — copied across devices, where it can suppress a fetch on the other one

The trigger was a new preview cache. Adding a normal storage key for a phone-only screen quietly enlisted it into every watch transfer. That's when the shape of the bug became obvious: **storage registration was implying sync enrollment**, and there was no place in the codebase where anyone was asked to decide.

## The Audit

The fix isn't interesting. The method is.

Before writing any code, we mapped every one of the ~90 keys to actual reads and writes **in the watch-compiled source set** — not "does the watch have this file", but "is this file a member of the watch target", using the target's file-membership exceptions in `project.pbxproj` as ground truth. Each key got one of three verdicts: read on watch, written on watch, or neither.

The result: roughly 25 keys belonged. About 50 were pure payload — bytes that had never been read on the other side of the connection. The rest were judgment calls that only surfaced *because* we were forced to write down a reason for each key.

Three findings that a quick eyeball would have gotten wrong, in both directions:

**Six notification toggles looked phone-only. They aren't.** Each maps to an iOS notification category, so excluding them was the obvious call. But a helper ORs all six into a single "notifications on?" boolean, and the watch's location service branches on it — `requestAlwaysAuthorization()` versus `requestWhenInUseAuthorization()`. Dropping six booleans would have silently downgraded the watch's location authorization request for every user with notifications enabled. Six bytes, real consequence.

**A key written on the watch but never read there.** The watch stores the device location, then geocodes its own copy anyway; every reader of the stored value is phone-only UI. It's a genuine trim candidate — and we kept it, because dropping it is an unforced behavior change with no upside. "Unused" and "safe to remove in this PR" are different questions.

**A flag whose only reader is compiled out today.** The feature it gates isn't available on watchOS yet, so the read sits inside a `#if canImport(...)` that never fires there. But the code path that consults it runs unconditionally in watch context, so the key goes live the day the framework arrives. Enrolled ahead of time.

We also wrote down the high-risk keys explicitly, because their failure modes are silent rather than loud: entitlement flags, the language key (a localization helper reads the store directly, so losing it reverts the whole watch to English), the chart-style key (a style resolver reads the store directly, bypassing the settings manager), and the API parameter keys the watch uses to build its own forecast URL.

## The Inversion

Two mechanics of the old protocol shaped the design, and both had to be verified rather than assumed.

**The payload unions registered defaults.** `dictionaryRepresentation()` returns every key that has a `registerDefaults` entry, whether or not a user ever touched it. The sync wasn't "everything the user changed" — it was "everything the app ever declared."

**Absence meant deletion.** The apply side cleared *every* known key before writing the incoming values:

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

That `return true` is deliberate — it keeps the watch app working during the window before the first sync lands, rather than flashing a paywall at a paying customer. But combine it with wipe-then-apply and you get a landmine: **any allowlist that omits the entitlement key hands out the paid watch app for free.** Get the polarity backwards on a different key and you revoke it from someone who paid. A one-line mistake in a list of key names, and the app's business model is a coin flip.

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

Three changes worth separating, because each does distinct work:

1. **The allowlist is opt-in.** Adding a storage key no longer enrolls it in anything. Enrollment is a deliberate, reviewed act with a comment pointing at the reasoning.
2. **Clear only synced keys.** Exclusion now means "watch-local," never "delete." Unsynced keys the watch owns — its own fetch mutex, its own local bookkeeping — survive a sync instead of being wiped by it.
3. **Apply only synced keys.** The filter runs on the receiving side too. During the version-skew window, an older phone still sends the full 90-key payload; without the receive-side filter it would smuggle the excluded keys straight back onto the watch.

We also dropped `CaseIterable` from the enum. There were zero remaining uses, and `allCases` was the exact footgun being removed — leaving it available is leaving the trap armed for whoever needs "a list of all the keys" next.

The allowlist is covered by tests, and the interesting one is not the count assertion:

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

A count test says "36." The exclusion test says *why* — it names the keys whose presence would be a bug and fails with a sentence a future reader can act on.

## The Bug the Audit Found

Halfway through mapping keys to reads, one key came back with an answer that didn't fit the categories: the data-source preference was **written on the watch**.

Which raised an immediate question, because the sync is one-directional. The watch's `sync()` only reloads complications; its `requestSync()` *pulls* phone state. There is no watch-to-phone data path at all.

So a source picked on the watch was reverted by the next phone sync. Always. The write went into the shared store, looked like it worked, and got wiped the next time the phone said anything.

The reflex is to build the missing direction. We didn't, for two reasons.

First, we checked whether anyone could actually reach the picker — and they couldn't. The picker view had been unreachable since 2024, when the button that presented it was turned into a display-only label. The bug was real but latent: dead code with a live-looking failure mode, sitting in the repo for nearly two years.

Second, even if it *had* been reachable, "add a watch-to-phone settings channel" is a transport project, not a bug fix. Building a reverse sync channel to serve one picker nobody had asked for would have been the tail wagging the dog.

So we deleted the picker. The read-only source label stays; the dead view is gone. The restore path is written down instead of built — watch source selection is a phase of the watch-parity plan, explicitly gated on the watch-to-phone settings channel that a different plan owns, and if it comes back it comes back with the Automatic option the phone has.

**Delete the UI that lies.** A control that appears to work and silently reverts is worse than no control. When you find one, the choice is fix the plumbing or remove the control — and removing it is legitimate, as long as you write down what restoring it would require.

## Results

The payload went from ~90 keys to 36 — and the excluded set is where the weight was: the transaction cache, both location arrays, the delivered-alerts record, the preview cache. Transfers on the complication path carry the settings the watch reads and nothing else.

The entitlement landmine is defused. The allowlist can no longer wipe a key it doesn't list, so an omission is now a missing-setting bug instead of a free-paid-app bug. That's the change that mattered most: not the bytes, but converting a silent revenue failure into a visible, boring one.

Two riders came out of the audit for free. A key that had been a hardcoded literal since 2024 and never touched the store at all was deleted. And the watch, on first touch of a shared manager, was writing an iOS paywall timestamp into the shared store — harmless under wipe-everything semantics because the next sync erased it, but permanent under clear-only-synced. Changing the deletion rule turned a self-correcting accident into a persistent one, so it's now compiled out with `#if !os(watchOS)`.

One consequence we accepted rather than fixed: watches that already synced the ~50 excluded keys keep them frozen in their local store forever. Nothing sends them and nothing clears them anymore, so the win is transfer-only. It's correctness-neutral — none of them is read on the watch — and a one-time purge is written down as an optional follow-up rather than shipped speculatively.

The last piece is documentation, because the inversion has a cost that only shows up months later. Opt-in means a new key that the watch genuinely needs will silently not arrive. So the feature-flags skill grew a sixth touch point:

| Step | Pattern |
|---|---|
| Watch sync | **Only if watch-compiled code reads the flag**: enroll it in `Keys.synced`, else skip — keys do NOT sync automatically |

When you invert a default, the new default's failure mode moves. Denylist fails by shipping too much. Allowlist fails by shipping too little — quietly, in a target you're not looking at. That's a strictly better failure to have, and it still has to be written down where the next person will hit it.

## Lessons Learned

- **"Absence means deletion" is a dangerous protocol default.** Wipe-then-apply is the easy way to make a sync converge, and it turns every omission into a destructive operation. Make deletion explicit — a tombstone, an explicit key list, anything — or scope the wipe to exactly the keys you're authoritative for.
- **Check what absence *means* on the receiving side.** A missing key is rarely neutral. Ours meant "paid." Somewhere in your codebase, a `nil` check has a default that was written for a different situation than the one your sync protocol creates.
- **Allowlist over denylist for anything crossing a device boundary.** Payloads only grow, and they grow by accident — by someone adding a storage key for an unrelated screen. Under an allowlist that's a no-op; under a denylist it's a silent enrollment.
- **Audit first, decide second.** Mapping every key to real reads took an afternoon and produced three findings that contradicted the obvious answer in both directions. Half the value wasn't the list — it was being forced to write a reason next to each key.
- **Ground the audit in the build system, not the file tree.** "Does the watch read this?" is a question about target membership. Grepping the repo would have gotten several keys wrong.
- **When a one-way channel is pretending to be two-way, delete the UI that lies.** Don't build the missing direction to justify a control nobody uses. Remove the control, write down what restoring it requires, and let the transport work happen when something actually needs it.
- **Removing the footgun means removing the tool.** Dropping `CaseIterable` was the point of the change, not a tidy-up. As long as `allCases` exists, someone will reach for it.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and plan docs, then reviewed before publishing.
