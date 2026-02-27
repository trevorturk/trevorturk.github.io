---
layout: post
title: "Heroku Capacity: Scaling for Traffic Spikes"
date: 2026-02-27
summary: "Dyno scaling decisions backed by data, not guesswork, with guardrails that prevent both over-provisioning and capacity crunches."
tags: [skills, scripts, heroku, scaling, performance]
---

## The Problem

Scaling dynos is easy. Knowing *when* to scale is hard.

Hello Weather experiences predictable traffic spikes from APNS (Apple Push Notification Service) background refreshes. Every 30 minutes, thousands of devices wake up and request fresh weather data. This creates bursty demand that can saturate dynos if we're under-provisioned.

But over-provisioning is expensive. We needed a system to answer: Is our current capacity sufficient? Should we scale up? Can we safely scale down?

## The Solution

A skill/script pair for capacity operations: `bin/heroku` CLI with capture/analyze/recommend workflow.

### Architecture

```
bin/heroku
├── status    # Current formation vs config
├── check     # One-command operational check
├── capture   # Sample metrics during window
├── analyze   # Process captured samples
└── recommend # Policy decision from data
```

**Config files:**
- `config/heroku/guardrails.yml` - Latency, error rate, load thresholds
- `config/heroku/formation.yml` - Current formation, bounds, history

### The Pattern: Status -> Capture -> Analyze -> Recommend

```bash
# 1. Check current state
bin/heroku status

# 2. Capture during normal traffic
bin/heroku capture --window normal --json

# 3. Capture during spike window
bin/heroku capture --window spike_30 --json

# 4. Get scaling recommendation
bin/heroku recommend --json
```

## Implementation

### Guardrails YAML

The guardrails define what "good" looks like:

```yaml
# config/heroku/guardrails.yml
latency:
  p50_max_ms: 200
  p95_max_ms: 500
  p99_max_ms: 1000

error_rate:
  max_percent: 0.1

load:
  queue_depth_max: 5
  connect_time_max_ms: 100
```

Every capture is evaluated against these thresholds.

### Formation YAML

The formation tracks current state and history:

```yaml
# config/heroku/formation.yml
apps:
  helloweather:
    web:
      current: 4
      min: 2
      max: 10
    worker:
      current: 1
      min: 1
      max: 2

history:
  - date: 2026-02-15
    change: web 3 -> 4
    reason: spike_00 latency exceeded p95 threshold
    captures: [spike_00_20260215_0001.json]
```

This creates an audit trail for scaling decisions.

### Capture Windows

Like CloudFront logging, capture windows align with traffic patterns:

| Window | Timing | Purpose |
|--------|--------|---------|
| `normal` | Off-peak | Baseline capacity |
| `spike_00` | :00-:08 | APNS top-of-hour |
| `spike_30` | :30-:38 | APNS mid-hour |

The spike windows are critical - that's when capacity problems surface.

### The One-Command Check

For quick operational checks, one command does it all:

```bash
bin/heroku check --json
```

This runs a capture, analyzes it, and returns a recommendation:

```json
{
  "formation": { "web": 4, "worker": 1 },
  "guardrails": { "passed": true },
  "latency": { "p50": 145, "p95": 312, "p99": 678 },
  "recommended_state": "HOLD",
  "captures_analyzed": 3
}
```

### Recommended States

The recommend command outputs one of three states:

| State | Meaning | Action |
|-------|---------|--------|
| `HOLD` | Capacity is appropriate | Do nothing |
| `SCALE_UP` | Guardrails breached | Increase dynos |
| `PROBE_DOWN` | Significant headroom | Consider reducing |

`PROBE_DOWN` is intentionally cautious - it suggests you *might* be able to scale down, but recommends capturing more data first.

### Interpreting Failures

Not all guardrail breaches mean "add more dynos":

**Upstream timeouts** - If latency spikes are caused by slow upstream providers (weather data sources), adding dynos won't help. This needs timeout tuning (see [CloudFront Logging](/2026/02/27/cloudfront-logging.html)).

**Router saturation** - Look for `connect` time growth, queue-like error codes, elevated load. This *does* indicate dyno pressure.

**Small samples** - Treat short capture windows as directional, not definitive.

## The Workflow in Practice

### Pre-Event Scaling

Before a known traffic event (app feature, press coverage):

```bash
# Verify current state
bin/heroku status

# Capture baseline
bin/heroku capture --window normal --json

# Capture spike (wait for next :00 or :30)
bin/heroku capture --window spike_00 --json

# Get recommendation
bin/heroku recommend --json
# → SCALE_UP or HOLD
```

### Post-Incident Analysis

After users report slowness:

```bash
# Capture during the problem
bin/heroku capture --window spike_00 --json

# Check guardrails
bin/heroku check --json

# If guardrails breached, scale up
heroku ps:scale web=6 -a helloweather

# Update formation.yml
# Document the change with capture references
```

### Probe Down Workflow

When costs seem high:

```bash
# Capture both windows
bin/heroku capture --window normal --json
bin/heroku capture --window spike_30 --json

# Check recommendation
bin/heroku recommend --json
# → PROBE_DOWN (maybe)

# If probe_down, reduce by 1
heroku ps:scale web=3 -a helloweather

# Monitor next spike window
bin/heroku capture --window spike_00 --json

# If guardrails still pass, hold. If not, scale back up.
```

## Operational Guardrails

The skill enforces careful decision-making:

- **Never scale from one capture** - Require multiple windows
- **Keep formation.yml current** - Update after every change
- **Store capture artifacts** - Reference them in PRs
- **Use `--json` for automation** - Structured output for scripts

## Results

- **Data-driven scaling** instead of guesswork
- **Clear audit trail** of why we scaled
- **Spike awareness** - We know APNS patterns affect capacity
- **Cost savings** - Confident probe-down when appropriate

## Lessons Learned

- **APNS creates predictable spikes** - Plan for them, don't be surprised
- **Distinguish dyno pressure from upstream slowness** - Adding dynos doesn't fix slow sources
- **Keep history** - Past scaling decisions inform future ones
- **Probe down cautiously** - Better to overpay slightly than to drop requests

---

## How This Post Was Made

**Prompt:** "Write 7+ in-depth blog posts documenting real engineering patterns from helloweather/web. These posts go deeper than the existing 'Skills and Scripts' overview, showing specific implementations."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Source: `.claude/skills/heroku-capacity/SKILL.md`
