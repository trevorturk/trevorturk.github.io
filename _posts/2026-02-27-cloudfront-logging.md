---
layout: post
title: "CloudFront Logging: Time-Boxed Investigations"
date: 2026-02-27 15:00:00 -0600
summary: "Running targeted logging campaigns to investigate timeout behavior and tune per-source timeouts."
tags: [skills, scripts, cloudfront, aws, performance]
---

## The Problem

Hello Weather proxies requests through CloudFront to multiple upstream weather data providers. Each provider has different latency characteristics. When users experience slowness, we need to answer: which source is slow, how slow, and under what conditions?

CloudFront logging provides the answers, but it's expensive to leave on permanently. We needed a way to run targeted investigations - enable logging, capture data, analyze it, then disable logging again.

The twist: traffic isn't uniform. Apple Push Notification Service (APNS) creates traffic spikes at the top and bottom of each hour when all devices refresh simultaneously. Normal traffic patterns tell you one story; spike patterns tell another.

## The Solution

A skill/script pair for time-boxed logging campaigns: `bin/cloudfront` CLI that manages the full lifecycle.

### The Pattern: Enable -> Capture -> Recommend -> Write -> Disable

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

## Implementation

### Capture Modes

Three capture modes align with traffic patterns:

| Mode | Timing | Purpose |
|------|--------|---------|
| `normal` | Anytime | Baseline latency |
| `spike_00` | :00 and :01 | APNS refresh spike |
| `spike_30` | :30 and :31 | APNS mid-hour spike |

The spike windows are when problems surface. A timeout that works at 2:15 might fail at 2:00 when thousands of devices refresh simultaneously.

### Multi-Run Stability Analysis

Single samples lie. The `recommend stability` command analyzes multiple capture runs:

```bash
bin/cloudfront recommend stability --mode normal \
  --target-success 0.999 \
  --min-timeout 1.5 \
  --max-timeout 3.0 \
  --profile production --json
```

This looks across all captured runs for a mode and source, requiring consistent behavior before recommending timeout changes.

### Backfill Campaigns

For thorough investigations, the backfill command automates multi-day capture:

```bash
# Run captures over 3 days across all modes and sources
bin/cloudfront capture backfill \
  --days 3 \
  --modes normal,spike_00,spike_30 \
  --all-sources \
  --profile production
```

This is better than ad-hoc shell loops because it handles scheduling, interruptions, and stores results in SQLite for later analysis.

### SQLite Storage

Capture data goes into `tmp/cloudfront.sqlite3`:

```sql
-- Each capture run gets an ID
SELECT * FROM capture_runs WHERE mode = 'spike_00';

-- Latency samples per source
SELECT source, p50, p95, p99, success_rate
FROM capture_samples
WHERE run_id = 12;
```

This keeps raw data out of context windows while enabling complex queries.

### Timeout Floor Guidance

We learned some hard lessons about timeout floors:

- **Avoid global 2.0s minimum** - Per-source exceptions produce better results
- **Current stance**: 1.5s floor, 3.0s cap, raise only with stability evidence
- **Spike windows need longer timeouts** - especially `spike_00`
- **Below 1.5s = dropped requests** - We tested `1.4`, `1.3`, `1.2`... failure rate climbed

Weather refreshes can tolerate slightly longer waits than tap interactions, but users still expect bounded response times.

## The Workflow in Practice

### Investigation: "Users report slowness around 6pm"

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

## Operational Guardrails

The skill includes critical safety rules:

- **Default state is OFF** - Logging costs money; enable only when investigating
- **Start small** - Begin with one source before enabling all
- **Always `--dry-run`** - Preview mutations before applying
- **Short windows** - Capture just what you need, then disable
- **Retention limits** - 3-7 days max; don't let logs accumulate
- **Never tune from one run** - Require both normal and spike data

## Results

- **Targeted investigations** instead of always-on logging
- **Per-source timeouts** tuned to actual behavior
- **Spike awareness** - Different timeouts for spike vs normal traffic
- **Reproducible analysis** - SQLite captures enable re-analysis

## Lessons Learned

- **APNS creates non-obvious patterns** - The 30-minute cycle affects everything
- **Stability analysis beats single samples** - One good run doesn't mean reliable behavior
- **Per-source > global** - Different providers need different timeouts
- **Time-box everything** - Logging is expensive; treat it as a campaign, not a default

---

## How This Post Was Made

**Prompt:** "Write 7+ in-depth blog posts documenting real engineering patterns from helloweather/web. These posts go deeper than the existing 'Skills and Scripts' overview, showing specific implementations."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Source: `.claude/skills/cloudfront-logging/SKILL.md`
