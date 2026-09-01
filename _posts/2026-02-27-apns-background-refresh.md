---
layout: post
title: "Silent Push Refresh Without a Location Database"
date: 2026-02-27 17:00:00 -0600
summary: "How Hello Weather refreshes widgets with a silent push every 30 minutes while the server stores nothing but anonymous push tokens. The device fetches its own weather, so no customer location ever lands in a server database."
tags: [ios, apns, privacy, architecture]
---

## The Problem

Widgets and complications need fresh weather while the app is closed. The obvious way to supply it, and the way most weather apps work, is for the server to store every user's locations, fetch weather for them on a schedule, and push the results down.

That design commits the server to a lot. It holds a record of where every user lives and works, and it has to protect it. The client's location list and the server's copy have to stay in sync. The app needs the Significant Location API so the server hears when a user moves. [Hello Weather](https://helloweather.com) takes privacy seriously, and we wanted fresh data without any of this.

## The Solution

Flip the direction. The device fetches its own weather, and the server's only job is to remind it when. Twice an hour a recurring job sends a silent push to every registered device. The push carries no alert, sound, or badge, only the `content-available` flag that tells iOS to wake the app.

### The Architecture

```
Server                          iOS App
  │                                │
  ├─ Cron job (:00, :30)           │
  │    │                           │
  │    └─ Silent push ─────────────┤
  │       (content-available: 1,   │
  │        fanned out over 5 min)  │
  │                                ├─ App wakes up
  │                                │
  │                                ├─ Resolves its location
  │                                │
  │   ┌────────────────────────────┤
  │   │  Weather request (lat/lon) │
  │   │                            │
  ├───┘                            │
  │                                │
  │   Weather response ────────────┤
  │                                │
  │                                └─ Updates widgets, complications
```

The server holds anonymous push tokens and nothing else.

### What the Server Knows

```
apns_tokens table:
- token (anonymous device identifier)
- env (sandbox or production)
- topic (the app's bundle id)
- created_at
- updated_at
```

No lat/lon, no user accounts, no location history. `updated_at` is touched when the app re-registers, and a daily job deletes tokens untouched for 90 days, so a deleted app falls out of the table on its own.

### The Cron Job

The scheduled part of the backend is one recurring job. It runs every 30 minutes, which lands at :00 and :30, and it does not send the pushes itself. It enqueues one small job per token with a random delay of up to five minutes, so the pushes fan out instead of hitting APNs and the database at once:

```ruby
# config/recurring.yml: schedule: every 30 minutes
class ApnsTokenPingEnqueueJob < ApplicationJob
  def perform
    ApnsToken.find_in_batches do |tokens|
      jobs = tokens.map do |token|
        ApnsTokenPingJob.new(token).set(wait: rand(5.minutes))
      end

      ActiveJob.perform_all_later(jobs)
    end
  end
end

class ApnsTokenPingJob < ApplicationJob
  retry_on StandardError, wait: :polynomially_longer, attempts: 6
  discard_on ActiveJob::DeserializationError

  def perform(token)
    raise unless token.ping
  end
end
```

`ApnsToken#ping` posts to APNs with `apns-push-type: background`, priority 5, and the body `{"aps": {"content-available": 1}}`. A 410 from APNs, or a 400 with `BadDeviceToken`, deletes the token, so the table cleans itself. A silent push shows no notification. Its only effect is to wake the app in the background.

### The iOS Side

When the push arrives:

1. iOS wakes the app in the background
2. App resolves its location: the selected saved location, or the device's current location (already authorized)
3. App requests weather for that location, unless it fetched within the last 25 minutes
4. App updates widgets and complications
5. App goes back to sleep

The 25-minute window keeps a device that refreshed just before the ping from fetching twice. The location leaves the device only inside the weather request, and that request is the same as any manual refresh.

## Why This Works

### Privacy

We cannot leak location data we do not have. There is no database to breach, no logs to subpoena, and no data to sell. The server is architecturally incapable of tracking users.

### Simplicity

| Server-side locations | Client-side refresh |
|----------------------|---------------------|
| User accounts | Anonymous tokens |
| Location database | No location storage |
| Sync protocol | No sync needed |
| Significant Location API | No location tracking |
| Background fetch jobs per user | One cron job for everyone |
| Location update webhooks | Nothing |

Every row on the left is a system to build and maintain.

### Reliability

No sync means no sync bugs. The app is the source of truth for locations, so adding one on the phone just works, with no server round-trip.

### Cost

Server-side tracking fetches weather for every saved location on every cycle. A user with 5 saved locations costs 5 API calls per refresh. Client-side refresh fetches only what the user or their widgets need, which most of the time is one location, wherever they are right now.

## The Trade-off

The design depends on Apple delivering the push. Silent pushes are not guaranteed. iOS may delay or skip them based on battery, network conditions, or how often the user opens the app.

In practice this is good enough. Widgets update reliably for active users, and a missed push costs at most 30 minutes, because the next one is already scheduled. Weather does not change that fast.

## The Traffic Pattern

The schedule shows up on the server as the "spike" pattern other posts refer to:

- **:00-:05** - Silent pushes fan out; devices wake up and request weather
- **:06-:29** - Normal traffic (manual refreshes, app opens)
- **:30-:35** - Silent pushes fan out again
- **:36-:59** - Normal traffic

This is why the [CloudFront Logging](/cloudfront-logging/) and [Heroku Capacity](/heroku-capacity/) skills have `spike_00` and `spike_30` capture windows. The load is bursty but predictable, and the fanout is what keeps the first minute of each window from being a wall.

## Lessons Learned

- **Not storing data is simpler than storing it securely.** The privacy win and the simplicity win came from the same decision.
- **"The server stores user data" is a default, not a requirement.** Ask what the server must know to do its job. Here it was a push token.
- **A silent push is a coordination signal, not a notification.** It can tell every device when to act without the server knowing anything about any of them.
- **Accept missed updates when the next one is cheap.** A 30-minute cadence covers a dropped push for data that changes slowly.
- **Spread the signal.** Waking every device in the same second is a self-inflicted outage; a random delay of a few minutes costs nothing the user can see.

---

## How This Post Was Made

**Prompt:** "Add another post about the APNS schedule - top and middle of the hour. Backend doesn't have a db for customers for privacy reasons. Instead, they just send a push token. We save that on the server (anon) and then we have a cron job for top and middle of the hour to send a silent push to the client which then triggers the app to do its own refresh. Then we don't need lat/lon on the server in a db. We also don't need the significant location api, pinging the api to update locations etc. Hello Weather takes privacy very seriously. However this is also a huge simplification, because we don't need to sync the client and server database, etc. The only change to the backend is to have a cron job!"

Generated by Claude (Opus 4.5) using the blog-post-generator skill.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The Problem now names the server-side design and its costs once instead of as a bullet list that Why This Works repeated, the Solution opens on the mechanism, the title dropped its subtitle, and Lessons Learned is four rules that transfer. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The cron job excerpt was invented and is replaced with the real two-job shape, an enqueue job on a 30-minute schedule that fans each device's push out over a random delay of up to five minutes, and a per-token ping job with retries; the push is not payload-free but a background push carrying `content-available: 1`, and the token table gained its real columns (`env`, `topic`, `updated_at`) plus the 90-day cleanup. The traffic table and diagram now show the five-minute fanout windows instead of a single-minute burst, and the iOS steps note that the app uses its selected location when one is set and skips the fetch if it refreshed within the last 25 minutes.
