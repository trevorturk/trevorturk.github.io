---
layout: post
title: "CloudFront Logging: Time-Boxed Investigations"
date: 2026-02-27 15:00:00 -0600
summary: "A command-line tool that turns CloudFront logging on, captures samples during normal traffic and the push-notification spike, recommends a timeout per source from several runs, and turns logging off again."
tags: [skills, scripts, cloudfront, aws, performance]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Twice an hour, at the top and the bottom, the Apple Push Notification Service (APNS) wakes every [Hello Weather](https://helloweather.com) device within a few minutes, and each one asks for fresh weather. Every request goes through CloudFront to one of several weather data sources, and each source has its own typical speed. A timeout that works at 2:15 can fail at 2:00, when thousands of devices refresh at once.

So when users report slowness, we need to know three things: which source is slow, how slow, and whether it's only during the spike. CloudFront logs answer all three. Leaving the logs on all the time costs money, so we turn logging on, capture some data, analyze it, and turn logging off again.

## The Solution

We wrote a skill and a script for these logging campaigns, where logging is on for a set stretch and then off again. The skill holds the rules, and `bin/cloudfront` is the command-line tool that runs the whole cycle.

### Enable -> Capture -> Recommend -> Write -> Disable

```bash
# 1. Enable logging for specific sources
bin/cloudfront logging enable --sources accuweather,aeris_weather --profile production

# 2. Capture samples during different traffic patterns
bin/cloudfront capture --mode normal --minutes 20 --profile production
bin/cloudfront capture --mode spike_00 --minutes 8 --profile production  # Top of hour
bin/cloudfront capture --mode spike_30 --minutes 8 --profile production  # Half hour

# 3. Generate timeout recommendations
bin/cloudfront recommend --mode normal --profile production --json
bin/cloudfront recommend stability --mode spike_00 --min-timeout 1.5 --max-timeout 3.0

# 4. Write optimized timeouts to config
bin/cloudfront timeouts write --min-timeout 1.5 --max-timeout 3.0

# 5. Disable logging when done
bin/cloudfront logging disable --profile production
```

Logging is on only between the first command and the last. The commands in between work from the captured samples.

## Implementation

### Capture Modes

Quiet-time speed and spike-time speed are different measurements, so every capture is tagged with the traffic pattern it ran under:

| Mode | Backfill window | Purpose |
|------|-----------------|---------|
| `normal` | :10 to :40 | Baseline latency |
| `spike_00` | :00 to :08 | APNS refresh spike |
| `spike_30` | :30 to :38 | APNS mid-hour spike |

A plain `capture` tags whatever window ends now, and `--minutes` sets its length. `capture backfill` uses the fixed windows above for every completed hour. Problems show up in the spike windows. Normal traffic can look fine while the eight minutes after the hour don't.

### Multi-Run Stability Analysis

One run can mislead. `recommend stability` pools every captured run for a mode and source and picks the timeout from the pooled samples. It also reports how many of those runs meet the target on their own at that timeout, so we can see when a recommendation rests on one good run:

```bash
bin/cloudfront recommend stability --mode normal \
  --target-success 0.999 \
  --min-timeout 1.5 \
  --max-timeout 3.0 \
  --profile production --json
```

The min and max flags set the range. The search starts at the floor and steps up by 0.1s until the success target is met. If nothing in the range meets it, the command returns the cap. A source with fewer than 100 samples gets no recommendation at all.

### Backfill Campaigns

We could have run `capture` in a shell loop, but a loop wouldn't handle the hard parts. Each mode's window has to land on the right minutes of every hour. A run that gets interrupted should pick up where it left off, without re-reading windows it already has. And the results need to go somewhere we can query. `capture backfill` does all three over several days, and it skips windows it already has by default:

```bash
# Run captures over 3 days across all modes and sources
bin/cloudfront capture backfill \
  --days 3 \
  --modes normal,spike_00,spike_30 \
  --all-sources \
  --profile production
```

### SQLite Storage

Captured samples go into `tmp/cloudfront.sqlite3`. An agent never has to read raw log lines into its context window, and we can query any run again later:

```sql
-- Each capture run gets an ID
SELECT id, window_start_utc, matched_events
FROM capture_runs WHERE mode = 'spike_00';

-- One row per request; percentiles are computed at query time
SELECT source, COUNT(*) AS samples,
       AVG(CASE WHEN time_taken_sec <= 1.5 THEN 1.0 ELSE 0.0 END) AS success_at_1_5s
FROM capture_samples
WHERE run_id = 12
GROUP BY source;
```

### Timeout Floor Guidance

We set the floor by testing, not by guessing:

- **1.5s floor, 3.0s cap.** Raise a timeout only when several runs agree it's needed.
- **Below 1.5s, requests drop.** We tested `1.4`, `1.3`, `1.2`, and the failure rate climbed.
- **No global 2.0s minimum.** A floor set per source works better than one number for every source.
- **Spike windows need longer timeouts**, `spike_00` most of all.

A background weather refresh can wait a little longer than a tap in the app, but users still expect it to finish in a reasonable time.

## The Workflow in Practice

### Investigation: "Users report slowness around 6pm"

Turn logging on, capture the problem window and the next spike, compare the two, raise the timeout on the one slow source, turn logging off:

```bash
# Start with logging
bin/cloudfront logging enable --profile production

# Capture during the problem window
bin/cloudfront capture --mode normal --minutes 30 --profile production

# Also capture next spike
bin/cloudfront capture --mode spike_00 --minutes 8 --profile production

# Check what we got
bin/cloudfront recommend --mode normal --profile production --json
bin/cloudfront recommend --mode spike_00 --profile production --json

# If source X is slow, tune its timeout
bin/cloudfront timeouts write --sources source_x --min-timeout 2.0

# Clean up
bin/cloudfront logging disable --profile production
```

The tuning step names one source, so only the slow one gets its floor raised to 2.0s. The others don't move.

### Quarterly Tune-Up

```bash
# Full backfill over a week
bin/cloudfront capture backfill --days 7 --modes normal,spike_00,spike_30 --all-sources

# Generate stability-based recommendations
bin/cloudfront recommend stability --mode normal --target-success 0.999 --json
bin/cloudfront recommend stability --mode spike_00 --target-success 0.995 --json

# Write to config
bin/cloudfront timeouts write --target-success 0.999 --min-timeout 1.5 --max-timeout 3.0
```

The spike window gets a looser target than normal traffic: 0.995 instead of 0.999.

## Operational Guardrails

One command starts a bill, so the skill lists the safety rules. As written in February 2026:

- **Default state is OFF.** Logging costs money, so turn it on only while investigating.
- **Start small.** Begin with one source before enabling all of them.
- **Always `--dry-run`.** Preview a change before applying it.
- **Short windows.** Capture what the question needs, then turn logging off.
- **Retention limits.** Keep logs for 3-7 days at most, so they don't pile up.
- **Never tune from one run.** A change needs both normal and spike data.

As of March 2026, the first, fourth, and fifth rules are gone. Logging now stays on, with logs kept for 90 days. Backfills always ask for the full rolling window (`--hours 2160 --skip-existing`). Timeouts are recomputed weekly from `--window-hours 2160` instead of from the window around an incident. The `--dry-run` rule and the both-modes rule still stand. There's also a new one: before accepting a lower timeout from `timeouts write`, check `recommend stability --json` for that source and mode. If only a few runs support the change, keep the old value.

## Results

- Logging ran as a campaign with a start and an end instead of a permanent bill. That lasted two weeks; the same tooling now keeps a rolling 90-day window, as the guardrails above say.
- Each source and each traffic pattern gets its own timeout, taken from captured samples instead of one number for everything.
- Every capture is in SQLite, so we can re-run an analysis on the same samples without turning logging back on.
- The cost is that someone has to remember to turn logging off after a capture. That's why the guardrails exist.

## Lessons Learned

- **Sample the traffic pattern you're tuning for.** A predictable spike, like a push cycle every 30 minutes, behaves differently from normal traffic and needs its own captures.
- **Set timeouts per dependency, not one for the whole system.** Two sources with different speeds don't share a correct timeout.
- **Treat expensive logging as a campaign.** Turn it on, capture, decide, turn it off, and leave it off by default.
- **Keep raw samples out of the agent's context window.** Write them to a database and query it, so looking again means a query, not a new capture.
