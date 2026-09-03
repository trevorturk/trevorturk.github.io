---
layout: post
title: "Sync Only What the Watch Reads"
date: 2026-07-29 09:40:00 -0600
summary: "We checked every settings key the phone sends to the watch, replaced the send-everything list with an opt-in one, and fixed a sync rule where a missing key unlocked the paid watch app for anyone."
tags: [swift, watchos, sync, ios]
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

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and plan docs, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the preview-cache key that exposed the bug, the title lost its subtitle, the three audit findings became a list, the bolded closer in the dead-picker section became plain prose, and Lessons Learned dropped the three bullets the body already states. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The notification-toggle finding said six toggles were nearly dropped; the plan's re-verification record shows two were already on the list and four were the near-miss, so the count is now four (the helper still ORs all six). The allowlist comment in the shipped-shape excerpt now matches the current source, which points at the watch-app skill rather than a since-deleted plan doc; the old-protocol filter closure and the payload test were aligned with the real code (the test's allowlist-membership assertion was missing from the excerpt).

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 2, run after batch 1 (#68) merged. The prose now speaks in first person with contractions, the "landmine", "footgun", and "arms the trap" figures became plain statements of the bug, "fetch mutex" became "fetch lock", and the two aphorism closers (the dead-picker section and the Results section) became plain reasons. The summary and the bolded entitlement warning drop the retired "for free" in favor of "without paying" and "unlocked"; code blocks, headings, numbers, quoted text, the table, and the link are unchanged. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"

**Prompt 9:** "usage looks fine, merge it and run batch 2"
