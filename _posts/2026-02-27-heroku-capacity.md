---
layout: post
title: "Heroku Capacity: Scaling for Traffic Spikes"
date: 2026-02-27 13:00:00 -0600
summary: "A script for Heroku dynos that samples metrics during the APNS spike windows, scores them against guardrail thresholds, and answers HOLD, SCALE_UP, PROBE_DOWN, or COLLECT_MORE, with the evidence recorded in YAML."
tags: [skills, scripts, heroku, scaling, performance]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Every 30 minutes, thousands of devices wake up and ask [Hello Weather](https://helloweather.com) for fresh weather data. The refresh comes from APNS (Apple Push Notification Service) background pushes, so the burst lands at the top and bottom of every hour. With too few dynos, the burst saturates them. With too many, we pay for idle capacity around the clock.

Adding a dyno on Heroku is one command. Knowing when to add one, or when it's safe to remove one, was the hard part. We wanted answers to three questions that weren't guesses: is the current capacity enough, should we scale up, and can we safely scale down?

## The Solution

We wrote a `bin/heroku` script and a matching skill. The script samples Heroku metrics during a traffic window, scores them against thresholds, and recommends a change to the formation (the count and size of each dyno type). The skill tells the agent which command to run and when.

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

- `config/heroku/guardrails.yml` - Latency, error rate, load, and memory thresholds
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

The guardrails file says what healthy looks like, and every capture is scored against it. `thresholds` are hard limits. `targets` say how much headroom we want before we try running fewer dynos:

```yaml
# config/heroku/guardrails.yml
thresholds:
  router_p95_ms: 700
  router_p99_ms: 1200
  api_p95_ms: 750
  error_rate_percent: 0.2
  error_rate_min_count: 2
  load_p95_normal: 0.70
  load_p95_spike: 1.00
  memory_p95_mb: 860
  swap_p95_mb: 50

targets:
  load_p95_ideal_max: 0.50
  memory_percent_ideal_max: 75

spike_windows:
  - name: top_of_hour
    minute: 0
    duration_sec: 300
  - name: mid_hour
    minute: 30
    duration_sec: 300
```

The load threshold is looser for spike captures than for normal ones. We let a burst run hot for a few minutes, but not steady traffic.

### Formation YAML

The formation file records the current dyno counts and sizes, the floor and ceiling we'll allow, a rollback target, and a history of every change with its reason:

```yaml
# config/heroku/formation.yml (as of 2026-02-26, trimmed)
app: helloweather

current:
  web: 6
  worker: 1
  web_type: Standard-2X
  worker_type: Standard-1X

bounds:
  web_floor: 6
  web_ceiling: 40

rollback:
  web: 14

history:
  - date: "2026-02-26"
    web: 6
    reason: Reduced after guardrail captures showed sustained headroom
  - date: "2026-02-24"
    web: 8
    reason: Stepped down after CPU optimization pass
```

The history is our audit trail, so the next decision starts from the reason for the last one instead of from memory. The capture files behind each change are linked from the PR that made it, not from this file.

### Capture Windows

As with our CloudFront logging, the capture windows line up with the traffic pattern:

| Window | Timing | Purpose |
|--------|--------|---------|
| `normal` | Off-peak | Baseline capacity |
| `spike_00` | :00, five minutes | APNS top-of-hour |
| `spike_30` | :30, five minutes | APNS mid-hour |

The script doesn't wait for the clock. `--window` sets the label, the number of log pulls (8 for a spike, 4 for normal), and which load threshold applies. Whoever runs it starts a spike capture at :00 or :30. Capacity problems show up in the spike windows, so a `normal` capture on its own is a baseline, not a verdict.

### The One-Command Check

When we want a quick read, `check` runs the whole pass:

```bash
bin/heroku check --json
```

It runs `status`, then a normal capture, then a spike capture (`--spike-window`, default `spike_30`), and makes a recommendation from both. Start it at :00 or :30 so the spike capture actually covers a spike. Trimmed output:

```json
{
  "formation_status": {
    "configured": { "web": 6, "worker": 1 },
    "live": { "web": 6, "worker": 1 },
    "drift": false
  },
  "captures": {
    "normal": { "summary": { "guardrails": { "passed": true, "recommended_state": "PROBE_DOWN" } } },
    "spike_30": { "summary": { "guardrails": { "passed": true, "recommended_state": "HOLD" } } }
  },
  "recommendation": {
    "decision": "HOLD",
    "captures_considered": 2,
    "clean_captures": 2,
    "clean_spike_captures": 1,
    "failed_captures": 0
  }
}
```

### Recommended States

`recommend` returns one of four decisions:

| Decision | Meaning | Action |
|----------|---------|--------|
| `SCALE_UP` | A recent capture failed a guardrail | Increase dynos |
| `COLLECT_MORE` | Not enough clean captures yet | Capture again |
| `HOLD` | Guardrails pass, but a capture is above the ideal load target | Do nothing |
| `PROBE_DOWN` | Every capture is under the ideal load target | Consider reducing |

We made `PROBE_DOWN` hard to get on purpose. It needs at least three clean captures (`--min-captures`), at least one of them a spike (`--min-spike`), and all of them under `load_p95_ideal_max`. Anything short of that comes back as `COLLECT_MORE` or `HOLD`. One trap: `recommend` reads the most recent capture files by count (`--recent`, default 6), not by age. A failed capture from weeks ago keeps producing `SCALE_UP` until newer files push it out. The skill recorded that in August 2026.

### Interpreting Failures

A failed guardrail doesn't always mean we need more dynos:

- **Upstream timeouts** - If the latency comes from slow weather data providers, adding dynos won't help. That's a timeout tuning problem (see [CloudFront Logging](/cloudfront-logging/)).
- **Router saturation** - Rising `connect` time, queue-related error codes, and high load do mean the dynos are under pressure.
- **Small samples** - A short capture window tells you which way things are going, not whether to act.

## The Workflow in Practice

### Pre-Event Scaling

Before a launch or press coverage, the full pass tells us whether the current formation will hold:

```bash
# Verify current state
bin/heroku status

# Capture baseline
bin/heroku capture --window normal --json

# Capture spike (wait for next :00 or :30)
bin/heroku capture --window spike_00 --json

# Get recommendation
bin/heroku recommend --json
# decision=SCALE_UP or decision=HOLD
```

### Post-Incident Analysis

When users report slowness, we capture while the problem is happening, then scale and record the change:

```bash
# Capture during the problem
bin/heroku capture --window spike_00 --json

# Check guardrails across recent captures
bin/heroku recommend --json

# If guardrails breached, scale up
heroku ps:scale web=8 -a helloweather

# Update formation.yml
# Document the change with capture references
```

### Probe Down Workflow

When the bill looks high, the same tools run in reverse. We drop one dyno and watch the next spike window before deciding anything else:

```bash
# Capture both windows
bin/heroku capture --window normal --json
bin/heroku capture --window spike_30 --json

# Check recommendation
bin/heroku recommend --json
# decision=PROBE_DOWN, or COLLECT_MORE if fewer than 3 clean captures

# If PROBE_DOWN, reduce by 1
heroku ps:scale web=5 -a helloweather

# Monitor next spike window
bin/heroku capture --window spike_00 --json

# If guardrails still pass, hold. If not, scale back up.
```

## Operational Guardrails

The skill holds the rules the script can't enforce. Four came with the first version:

- **Don't scale from one short capture.** Get a normal window and a spike window first.
- **Keep formation.yml current.** Update it after every change, including a change in dyno size.
- **Keep the capture files.** Link their JSON paths from decisions and PRs.
- **Use `--json` for automation.** Scripts read structured output, not prose.

A fifth rule came in mid-2026: the recommendation is advice, not an order. A `SCALE_UP` or a failed guardrail gets immediate attention. A `HOLD` gets a fresh look at the numbers rather than an automatic no to probing down.

## Results

- Each scaling change now has captures behind it and a `formation.yml` history entry with the new count and the reason. The two entries on record when the skill landed are both step-downs: web 8 on 2026-02-24 after a CPU optimization pass, and web 6 on 2026-02-26 after guardrail captures showed sustained headroom.
- The APNS pattern is written into the capture windows instead of living in someone's head.
- The cost is waiting. A decision needs a normal and a spike window, a probe-down needs three clean captures, and a spike capture can't start until the next :00 or :30.
- We didn't measure a dollar figure for probing down. A scale-down is now a one-dyno step with the rollback target written down first, instead of a guess.

## Lessons Learned

- **Give a predictable spike its own capture window.** Baseline metrics won't show the problem. Sample the minutes that do.
- **Work out whether it's the dynos or the upstream before scaling.** Rising connect time and queueing point at the dynos. Slow providers don't, and more dynos won't fix them.
- **Probe down one step at a time, and decide the rollback first.** Paying a little extra costs less than dropping requests.
