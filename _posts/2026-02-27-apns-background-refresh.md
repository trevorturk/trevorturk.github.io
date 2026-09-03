---
layout: post
title: "Silent Push Refresh Without a Location Database"
date: 2026-02-27 17:00:00 -0600
summary: "Hello Weather refreshes widgets with a silent push every 30 minutes, and the server stores nothing but anonymous push tokens. The device fetches its own weather, so no customer location lands in a server database."
tags: [ios, apns, privacy, architecture]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Widgets, and complications (the watch-face version of a widget), need fresh weather while the app is closed. The obvious way to get it there, and the way most weather apps work, is for the server to store every user's locations, fetch weather for them on a schedule, and push the results down.

That design asks a lot of the server. It holds a record of where every user lives and works, and it has to protect that record. The device's location list and the server's copy have to stay in sync. The app has to use the Significant Location API, which tells the server when a user moves. We take privacy seriously at [Hello Weather](https://helloweather.com), and we wanted fresh data without any of this.

## The Solution

We flipped it. The device fetches its own weather, and the server's only job is to tell it when. Twice an hour a cron job sends a silent push to every registered device. A silent push has no alert, sound, or badge. It carries only the `content-available` flag, which tells iOS to wake the app.

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

No lat/lon, no user accounts, no location history. `updated_at` changes each time the app re-registers. A daily job deletes tokens that haven't changed in 90 days, so a deleted app drops out of the table on its own.

### The Cron Job

The only scheduled work on the server is one cron job. It runs every 30 minutes, at :00 and :30. It doesn't send the pushes itself. It queues one small job per token, each with a random delay of up to five minutes, so the pushes spread out instead of hitting APNs and our database all at once:

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

`ApnsToken#ping` posts to APNs with `apns-push-type: background`, priority 5, and the body `{"aps": {"content-available": 1}}`. If APNs answers 410, or 400 with `BadDeviceToken`, we delete the token, so the table cleans itself.

### The iOS Side

When the push arrives:

1. iOS wakes the app in the background
2. The app picks its location: the saved location the user has selected, or the device's current location, which the user already authorized
3. The app requests weather for that location, unless it fetched within the last 25 minutes
4. The app updates widgets and complications
5. The app goes back to sleep

The 25-minute check keeps a device that refreshed just before the push from fetching twice. The location leaves the device only inside the weather request, and that's the same request a manual refresh makes.

## Why This Works

### Privacy

We can't leak location data we don't have. There's no location database to breach, and the server can't track users because it never learns where they are.

### Simplicity

| Server-side locations | Client-side refresh |
|----------------------|---------------------|
| User accounts | Anonymous tokens |
| Location database | No location storage |
| Sync protocol | No sync needed |
| Significant Location API | No location tracking |
| Background fetch jobs per user | One cron job for everyone |
| Location update webhooks | Nothing |

Each row on the left is something we'd have to build and keep running.

### Reliability

With no sync there are no sync bugs. The app owns the location list, so adding a location on the device just works, with no server round-trip.

### Cost

A server that tracks locations fetches weather for every saved location on every cycle. A user with 5 saved locations costs 5 API calls per refresh. A device refreshing itself fetches only what its widgets need, and most of the time that's one location: wherever the user is right now.

## The Trade-off

This only works if Apple delivers the push, and Apple doesn't promise to. iOS may delay or skip a silent push depending on battery, network conditions, or how often the user opens the app.

In practice that's good enough. Widgets update reliably for people who use the app, and a missed push costs at most 30 minutes because the next one is already scheduled. Weather doesn't change that fast.

## The Traffic Pattern

On the server, the schedule shows up as the "spike" pattern other posts mention:

- **:00-:05** - Silent pushes fan out; devices wake up and request weather
- **:06-:29** - Normal traffic (manual refreshes, app opens)
- **:30-:35** - Silent pushes fan out again
- **:36-:59** - Normal traffic

That's why the [CloudFront Logging](/cloudfront-logging/) and [Heroku Capacity](/heroku-capacity/) skills have `spike_00` and `spike_30` capture windows. The load comes in bursts, but we know when. The random delay spreads each burst over five minutes instead of piling it into the first minute.

## Lessons Learned

- **Not storing data is simpler than storing it safely.** The privacy win and the simplicity win came from the same decision.
- **Storing user data on the server is a habit, not a requirement.** Ask what the server needs to know to do its job. Here it was a push token.
- **Use a silent push as a wake-up call, not a message.** It can tell every device when to act without the server knowing anything about them.
- **Accept a missed update when the next one is coming soon.** A push every 30 minutes covers a dropped one for data that changes slowly.
- **Spread the pushes out.** Waking every device in the same second is an outage we'd cause ourselves. A random delay of a few minutes costs nothing the user can see.
