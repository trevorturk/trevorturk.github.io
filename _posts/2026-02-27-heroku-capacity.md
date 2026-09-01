---
layout: post
title: "Heroku Capacity: Scaling for Traffic Spikes"
date: 2026-02-27 13:00:00 -0600
summary: "A capture/analyze/recommend script for Heroku dynos: sample latency during the APNS spike windows, score it against guardrail thresholds, and get HOLD, SCALE_UP, or PROBE_DOWN with the evidence recorded in YAML."
tags: [skills, scripts, heroku, scaling, performance]
---

## The Problem

Scaling dynos is easy. Knowing *when* to scale is hard.

Every 30 minutes, thousands of devices wake up and ask [Hello Weather](https://helloweather.com) for fresh weather data. The trigger is APNS (Apple Push Notification Service) background refresh, so the burst lands at the top and bottom of every hour. Under-provision and the burst saturates the dynos. Over-provision and the idle capacity is paid for around the clock.

Three questions needed answers that were not guesses: Is current capacity sufficient? Should we scale up? Can we safely scale down?

## The Solution

A `bin/heroku` script and a matching skill. The script samples Heroku metrics during a traffic window, scores them against thresholds, and recommends a formation change. The skill says which command to run and when.

### Architecture

```
bin/heroku
├── status    # Current formation vs config
├── check     # One-command operational check
├── capture   # Sample metrics during window
├── analyze   # Process captured samples
└── recommend # Policy decision from data
```

Two YAML files hold the state:

- `config/heroku/guardrails.yml` - Latency, error rate, load thresholds
- `config/heroku/formation.yml` - Current formation, bounds, history

### The Pattern: Status -> Capture -> Analyze -> Recommend

A full pass is four commands:

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

The guardrails file defines what healthy looks like, and every capture is scored against it:

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

### Formation YAML

The formation file records the current dyno counts, their bounds, and a history of every change with the captures that justified it:

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

The history is the audit trail. The next decision starts from why the last one was made, not from memory.

### Capture Windows

Like CloudFront logging, capture windows align with traffic patterns:

| Window | Timing | Purpose |
|--------|--------|---------|
| `normal` | Off-peak | Baseline capacity |
| `spike_00` | :00-:08 | APNS top-of-hour |
| `spike_30` | :30-:38 | APNS mid-hour |

The spike windows are where capacity problems surface. A `normal` capture on its own is a baseline, not a verdict.

### The One-Command Check

For a quick operational read, `check` does it all:

```bash
bin/heroku check --json
```

It runs a capture, analyzes it, and returns a recommendation along with how many captures stand behind it:

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

`PROBE_DOWN` is deliberately cautious. It says you *might* be able to scale down, and asks for more captures before you do.

### Interpreting Failures

A breached guardrail does not always mean more dynos:

- **Upstream timeouts** - If latency spikes come from slow upstream providers (weather data sources), adding dynos will not help. That needs timeout tuning (see [CloudFront Logging](/cloudfront-logging/)).
- **Router saturation** - Growing `connect` time, queue-like error codes, and elevated load *do* indicate dyno pressure.
- **Small samples** - Treat short capture windows as directional, not definitive.

## The Workflow in Practice

### Pre-Event Scaling

Before a known traffic event (an app feature, press coverage), the full pass says whether the formation holds:

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

After users report slowness, capture during the problem, then scale and record the change:

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

When costs look high, the same tools run in reverse. Reduce by one dyno and watch the next spike window before deciding:

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

The skill holds four rules the script cannot enforce:

- **Never scale from one capture.** Require multiple windows.
- **Keep formation.yml current.** Update it after every change.
- **Store capture artifacts.** Reference them in PRs.
- **Use `--json` for automation.** Structured output is what scripts consume.

## Results

- Every scaling change now has captures behind it and a `formation.yml` history entry naming them and the reason. The one on record, web 3 -> 4 on 2026-02-15, followed a `spike_00` capture that breached the p95 threshold.
- The APNS pattern is encoded in the capture windows instead of known only from experience.
- The cost is patience. A decision waits on at least two windows, and a spike capture means waiting for the next :00 or :30.
- No dollar figure was measured for probing down. What changed is that a scale-down is a one-dyno probe with a defined rollback, not a gamble.

## Lessons Learned

- **Predictable spikes get their own capture window.** Baseline metrics never show the problem. Sample the minutes that do.
- **Separate dyno pressure from upstream slowness before scaling.** Growing connect time and queueing point at dynos. Slow providers do not, and more dynos will not fix them.
- **Probe down one step, with the rollback decided first.** Overpaying slightly costs less than dropped requests.

---

## How This Post Was Made

**Prompt:** "Write 7+ in-depth blog posts documenting real engineering patterns from helloweather/web. These posts go deeper than the existing 'Skills and Scripts' overview, showing specific implementations."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Source: `.claude/skills/heroku-capacity/SKILL.md`

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The opening now leads with the 30-minute APNS burst, each section says its part once, Results states what changed and what it cost and admits that savings were not measured, and Lessons Learned dropped the bullets that repeated the Operational Guardrails. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
