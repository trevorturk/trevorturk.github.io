---
layout: post
title: "Silent Push Refresh Without a Location Database"
date: 2026-02-27 17:00:00 -0600
summary: "How Hello Weather refreshes widgets with a silent push at :00 and :30 while the server stores nothing but anonymous push tokens. The device fetches its own weather, so no customer location ever lands in a server database."
tags: [ios, apns, privacy, architecture]
---

## The Problem

Widgets and complications need fresh weather while the app is closed. The obvious way to supply it, and the way most weather apps work, is for the server to store every user's locations, fetch weather for them on a schedule, and push the results down.

That design commits the server to a lot. It holds a record of where every user lives and works, and it has to protect it. The client's location list and the server's copy have to stay in sync. The app needs the Significant Location API so the server hears when a user moves. [Hello Weather](https://helloweather.com) takes privacy seriously, and we wanted fresh data without any of this.

## The Solution

Flip the direction. The device fetches its own weather, and the server's only job is to remind it when. Twice an hour a cron job sends a silent push, with no payload, to every registered device.

### The Architecture

```
Server                          iOS App
  │                                │
  ├─ Cron job (:00, :30)           │
  │    │                           │
  │    └─ Silent push ─────────────┤
  │       (no payload)             │
  │                                ├─ App wakes up
  │                                │
  │                                ├─ Gets current location
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
push_tokens table:
- token (anonymous device identifier)
- created_at
- last_seen_at
```

No lat/lon, no user accounts, no location history.

### The Cron Job

The entire backend change is one job, scheduled at :00 and :30:

```ruby
# :00 and :30 every hour
class ApnsPingJob < ApplicationJob
  def perform
    ApnsToken.find_each do |token|
      ApnsService.send_silent_push(token.token)
    end
  end
end
```

A silent push shows no notification. Its only effect is to wake the app in the background.

### The iOS Side

When the push arrives:

1. iOS wakes the app in the background
2. App gets the device's current location (already authorized)
3. App requests weather for that location
4. App updates widgets and complications
5. App goes back to sleep

The location leaves the device only inside the weather request, and that request is the same as any manual refresh.

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

- **:00** - All devices receive silent push, wake up, request weather
- **:01-:29** - Normal traffic (manual refreshes, app opens)
- **:30** - All devices receive silent push again
- **:31-:59** - Normal traffic

This is why the [CloudFront Logging](/cloudfront-logging/) and [Heroku Capacity](/heroku-capacity/) skills have `spike_00` and `spike_30` capture modes. The load is bursty but predictable.

## Lessons Learned

- **Not storing data is simpler than storing it securely.** The privacy win and the simplicity win came from the same decision.
- **"The server stores user data" is a default, not a requirement.** Ask what the server must know to do its job. Here it was a push token.
- **A silent push is a coordination signal, not a notification.** It can tell every device when to act without the server knowing anything about any of them.
- **Accept missed updates when the next one is cheap.** A 30-minute cadence covers a dropped push for data that changes slowly.

---

## How This Post Was Made

**Prompt:** "Add another post about the APNS schedule - top and middle of the hour. Backend doesn't have a db for customers for privacy reasons. Instead, they just send a push token. We save that on the server (anon) and then we have a cron job for top and middle of the hour to send a silent push to the client which then triggers the app to do its own refresh. Then we don't need lat/lon on the server in a db. We also don't need the significant location api, pinging the api to update locations etc. Hello Weather takes privacy very seriously. However this is also a huge simplification, because we don't need to sync the client and server database, etc. The only change to the backend is to have a cron job!"

Generated by Claude (Opus 4.5) using the blog-post-generator skill.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The Problem now names the server-side design and its costs once instead of as a bullet list that Why This Works repeated, the Solution opens on the mechanism, the title dropped its subtitle, and Lessons Learned is four rules that transfer. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
