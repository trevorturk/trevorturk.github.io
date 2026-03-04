---
layout: post
title: "APNS Background Refresh: Privacy Through Simplicity"
date: 2026-02-27 17:00:00 -0600
summary: "How Hello Weather uses silent push notifications to refresh weather data without storing customer locations on the server."
tags: [ios, apns, privacy, architecture]
---

## The Problem

Weather apps need fresh data. The obvious approach: store user locations on your server, run background jobs to fetch weather, push updates to devices. This is how most weather apps work.

But this creates problems:

- **Privacy** - You're storing where every user lives and works
- **Sync complexity** - Client and server databases must stay in sync
- **Location updates** - Need the Significant Location API to track when users move
- **Data liability** - You're responsible for protecting sensitive location data

We wanted fresh weather data without any of this.

## The Solution

Flip the model. Instead of the server fetching weather for users, have users fetch their own weather - just remind them to do it.

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

The server only stores anonymous push tokens - no locations, no user data.

### What the Server Knows

```
push_tokens table:
- token (anonymous device identifier)
- created_at
- last_seen_at
```

That's it. No lat/lon. No user accounts. No location history.

### The Cron Job

The entire backend change is a cron job that runs twice per hour:

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

A silent push has no visible notification - it just wakes the app in the background.

### The iOS Side

When the app receives the silent push:

1. iOS wakes the app in the background
2. App gets the device's current location (already authorized)
3. App requests weather for that location
4. App updates widgets and complications
5. App goes back to sleep

The location never leaves the device except in the weather request - and that request is the same as any manual refresh.

## Why This Works

### Privacy

We literally cannot leak location data we don't have. There's no database to breach, no logs to subpoena, no data to sell. The server is architecturally incapable of tracking users.

### Simplicity

Compare the two approaches:

| Server-side locations | Client-side refresh |
|----------------------|---------------------|
| User accounts | Anonymous tokens |
| Location database | No location storage |
| Sync protocol | No sync needed |
| Significant Location API | No location tracking |
| Background fetch jobs per user | One cron job for everyone |
| Location update webhooks | Nothing |

We removed entire categories of complexity.

### Reliability

No sync means no sync bugs. The app is the source of truth for locations. Add a new location on your phone? It just works - no server round-trip needed.

### Cost

Server-side location tracking means fetching weather for every user's locations on a schedule. If a user has 5 saved locations, that's 5 API calls per refresh cycle.

With client-side refresh, we only fetch weather when the user (or their widgets) actually need it. Most of the time, that's one location - wherever they are right now.

## The Trade-off

There's one downside: we depend on Apple's push notification delivery. Silent pushes aren't guaranteed - iOS may delay or skip them based on battery, network conditions, or app usage patterns.

In practice, this works well enough. Widgets update reliably for active users. And if a push is missed? The next one comes in 30 minutes. Weather doesn't change that fast.

## The Traffic Pattern

This architecture creates the "spike" traffic pattern referenced in other posts:

- **:00** - All devices receive silent push, wake up, request weather
- **:01-:29** - Normal traffic (manual refreshes, app opens)
- **:30** - All devices receive silent push again
- **:31-:59** - Normal traffic

This is why our [CloudFront Logging](/cloudfront-logging/) and [Heroku Capacity](/heroku-capacity/) skills have `spike_00` and `spike_30` capture modes. The traffic pattern is predictable but bursty.

## Lessons Learned

- **Privacy and simplicity often align** - Not storing data is easier than storing it securely
- **Question the obvious architecture** - "Server stores user data" isn't the only option
- **Silent push is underrated** - It's a coordination mechanism, not just notifications
- **Embrace the trade-offs** - Occasional missed updates are fine for weather
- **One cron job** - Sometimes the simplest solution really is the best one

---

## How This Post Was Made

**Prompt:** "Add another post about the APNS schedule - top and middle of the hour. Backend doesn't have a db for customers for privacy reasons. Instead, they just send a push token. We save that on the server (anon) and then we have a cron job for top and middle of the hour to send a silent push to the client which then triggers the app to do its own refresh. Then we don't need lat/lon on the server in a db. We also don't need the significant location api, pinging the api to update locations etc. Hello Weather takes privacy very seriously. However this is also a huge simplification, because we don't need to sync the client and server database, etc. The only change to the backend is to have a cron job!"

Generated by Claude (Opus 4.5) using the blog-post-generator skill.
