---
layout: post
title: "CloudFront Logging: Time-Boxed Investigations"
date: 2026-02-27 15:00:00 -0600
summary: "A CLI that turns CloudFront logging on, captures normal and APNS-spike traffic, recommends per-source timeouts from multiple runs, and turns logging off again."
tags: [skills, scripts, cloudfront, aws, performance]
model: "Claude Opus 4.5"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Twice an hour, at the top and the bottom, Apple Push Notification Service wakes every [Hello Weather](https://helloweather.com) device within a few minutes, and each one asks for fresh data. Every request goes through CloudFront to one of several upstream weather providers, each with its own latency profile. A timeout that works at 2:15 might fail at 2:00, when thousands of devices refresh simultaneously.

So when users report slowness, the question has three parts: which source is slow, how slow, and under what conditions. CloudFront logs answer all three. Leaving them on permanently is expensive, so the investigation has to be time-boxed: enable logging, capture data, analyze it, disable logging again.

## The Solution

A skill/script pair for time-boxed logging campaigns. `bin/cloudfront` is a CLI that manages the full lifecycle.

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

Logging is on only between the first command and the last. Everything in between reads captured samples.

## Implementation

### Capture Modes

Baseline latency and spike latency are different measurements, so every capture is tagged with the traffic pattern it ran under:

| Mode | Backfill window | Purpose |
|------|-----------------|---------|
| `normal` | :10 to :40 | Baseline latency |
| `spike_00` | :00 to :08 | APNS refresh spike |
| `spike_30` | :30 to :38 | APNS mid-hour spike |

A single `capture` tags whatever window ends now (`--minutes` sets its length); `capture backfill` uses the fixed per-mode windows above for every completed hour. The spike windows are where problems surface. Normal traffic tells one story; the eight minutes after the hour tell another.

### Multi-Run Stability Analysis

A single sample lies. `recommend stability` pools every captured run for a mode and source, picks the timeout from the pooled samples, and reports how many runs individually meet the target at that timeout, so a recommendation backed by one good run is visible as such:

```bash
bin/cloudfront recommend stability --mode normal \
  --target-success 0.999 \
  --min-timeout 1.5 \
  --max-timeout 3.0 \
  --profile production --json
```

The min and max flags bound the recommendation: the search starts at the floor, steps up by 0.1s until the success target is met, and returns the cap when nothing in range meets it. A source with fewer than 100 samples gets no recommendation at all.

### Backfill Campaigns

An ad-hoc shell loop over `capture` handles none of the hard parts: lining each mode's window up on the right minutes of every hour, resuming after an interruption without re-reading windows it already has, and putting results somewhere queryable. `capture backfill` handles all three over several days, and skipping already-captured windows is the default:

```bash
# Run captures over 3 days across all modes and sources
bin/cloudfront capture backfill \
  --days 3 \
  --modes normal,spike_00,spike_30 \
  --all-sources \
  --profile production
```

### SQLite Storage

Capture data goes into `tmp/cloudfront.sqlite3`. Raw samples never enter an agent's context window, and any run can be queried again later:

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

The floor came from testing rather than a guess:

- **1.5s floor, 3.0s cap.** Raise a timeout only with stability evidence across runs.
- **Below 1.5s, requests drop.** We tested `1.4`, `1.3`, `1.2`, and the failure rate climbed.
- **No global 2.0s minimum.** Per-source exceptions produce better results than one number for every provider.
- **Spike windows need longer timeouts**, `spike_00` most of all.

Weather refreshes can tolerate slightly longer waits than tap interactions, but users still expect bounded response times.

## The Workflow in Practice

### Investigation: "Users report slowness around 6pm"

Enable, capture the problem window and the next spike, compare, tune the one slow source, disable:

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

The tuning step names one source. The slow provider gets its floor raised to 2.0s rather than every provider's timeout moving at once.

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

The spike window is held to a looser target than baseline traffic, 0.995 against 0.999.

## Operational Guardrails

One command turns on a cost, so the skill carries the safety rules. As written in February 2026:

- **Default state is OFF.** Logging costs money; enable it only while investigating.
- **Start small.** Begin with one source before enabling all of them.
- **Always `--dry-run`.** Preview a mutation before applying it.
- **Short windows.** Capture what the question needs, then disable.
- **Retention limits.** 3-7 days at most; logs do not accumulate.
- **Never tune from one run.** A change needs both normal and spike data.

As of March 2026 the first, fourth, and fifth rules were replaced: logging stays on with 90-day retention, backfills always request the full rolling window (`--hours 2160 --skip-existing`), and timeouts are recomputed on a fixed cadence (weekly) from `--window-hours 2160` rather than from incident windows. The `--dry-run` and both-modes rules remain, plus a new one: before accepting a decrease from `timeouts write`, check `recommend stability --json` for that source and mode and keep the prior value when run-level evidence is sparse.

## Results

- Logging runs as a campaign with a start and an end instead of a permanent line item. (The campaign model lasted two weeks; the same tooling now runs a rolling 90-day window, see the guardrails above.)
- Timeouts are set per source and per traffic pattern, from captured behavior rather than one global number.
- Every capture sits in SQLite, so an analysis can be re-run against the same samples without enabling logging again.
- The trade-off is operational: logging switched on for a capture has to be switched off afterward, which is what the guardrails above exist for.

## Lessons Learned

- **Sample the traffic pattern you are tuning for.** A predictable spike, like a 30-minute push cycle, is a different population from baseline and needs its own captures.
- **Set limits per dependency, not per system.** Providers with different latency profiles do not share a correct timeout.
- **Treat expensive observability as a campaign.** Enable, capture, decide, disable, with off as the default state.
- **Keep raw samples out of the context window.** Write them to a database and query it, so re-analysis is a query rather than a new capture.
