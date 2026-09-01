---
layout: post
title: "Heroku Capacity: Scaling for Traffic Spikes"
date: 2026-02-27 13:00:00 -0600
summary: "A capture/analyze/recommend script for Heroku dynos: sample latency during the APNS spike windows, score it against guardrail thresholds, and get HOLD, SCALE_UP, PROBE_DOWN, or COLLECT_MORE with the evidence recorded in YAML."
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

The guardrails file defines what healthy looks like, and every capture is scored against it. `thresholds` are hard limits; `targets` say how much headroom justifies probing down:

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

The load threshold is looser for spike captures than for normal ones. A burst is allowed to run hot; steady traffic is not.

### Formation YAML

The formation file records the current dyno counts and sizes, their bounds, a rollback target, and a history of every change with its reason:

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

The history is the audit trail. The next decision starts from why the last one was made, not from memory. The capture JSON files behind each change are referenced in the PR that made it, not in this file.

### Capture Windows

Like CloudFront logging, capture windows align with traffic patterns:

| Window | Timing | Purpose |
|--------|--------|---------|
| `normal` | Off-peak | Baseline capacity |
| `spike_00` | :00, five minutes | APNS top-of-hour |
| `spike_30` | :30, five minutes | APNS mid-hour |

The script does not wait for the clock. `--window` sets the label, the number of log pulls (8 for a spike, 4 for normal), and which load threshold applies; the operator starts a spike capture at :00 or :30. The spike windows are where capacity problems surface. A `normal` capture on its own is a baseline, not a verdict.

### The One-Command Check

For a quick operational read, `check` does it all:

```bash
bin/heroku check --json
```

It runs `status`, then a normal capture and a spike capture (`--spike-window`, default `spike_30`), and recommends across both. Start it at :00 or :30 so the spike capture earns its label. Trimmed output:

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

The recommend command outputs one of four decisions:

| Decision | Meaning | Action |
|----------|---------|--------|
| `SCALE_UP` | A recent capture failed a guardrail | Increase dynos |
| `COLLECT_MORE` | Not enough clean captures yet | Capture again |
| `HOLD` | Guardrails pass, but a capture is above the ideal load target | Do nothing |
| `PROBE_DOWN` | Every capture is under the ideal load target | Consider reducing |

`PROBE_DOWN` is deliberately hard to earn. It requires at least three clean captures (`--min-captures`), at least one of them a spike (`--min-spike`), and every one under `load_p95_ideal_max`. Anything short of that is `COLLECT_MORE` or `HOLD`. `recommend` reads the most recent capture files by count (`--recent`, default 6), not by age, so a weeks-old failed capture keeps driving `SCALE_UP` until newer files displace it; the skill recorded that trap in August 2026.

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
# decision=SCALE_UP or decision=HOLD
```

### Post-Incident Analysis

After users report slowness, capture during the problem, then scale and record the change:

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

When costs look high, the same tools run in reverse. Reduce by one dyno and watch the next spike window before deciding:

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

The skill holds rules the script cannot enforce. Four landed with it:

- **Never scale from one short capture.** Require a normal and a spike window.
- **Keep formation.yml current.** Update it after every change, including dyno size.
- **Store capture artifacts.** Reference their JSON paths in decisions and PRs.
- **Use `--json` for automation.** Structured output is what scripts consume.

A fifth was added in mid-2026: the recommendation is decision support. `SCALE_UP` and a failed guardrail get immediate attention; `HOLD` gets a first-principles review, not an automatic refusal to probe.

## Results

- Every scaling change now has captures behind it and a `formation.yml` history entry naming the new count and the reason. The two on record when the skill landed are both step-downs: web 8 on 2026-02-24 after a CPU optimization pass, and web 6 on 2026-02-26 after guardrail captures showed sustained headroom.
- The APNS pattern is encoded in the capture windows instead of known only from experience.
- The cost is patience. A decision waits on a normal and a spike window, a probe-down on three clean captures, and a spike capture means waiting for the next :00 or :30.
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

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The guardrails and formation YAML excerpts were invented and are replaced with the real files as of 2026-02-26 (thresholds keyed `router_p95_ms` and friends, formation `current`/`bounds`/`rollback`, history entries with date, count, and reason but no capture list); the fabricated "web 3 -> 4 on 2026-02-15" history entry is replaced with the two real step-downs on 2026-02-24 and 2026-02-26. The `check` output was rewritten to the script's real `formation_status`/`captures`/`recommendation` shape, `check` is described as running both a normal and a spike capture, the fourth decision `COLLECT_MORE` and the three-clean-capture rule for `PROBE_DOWN` were added, the capture windows now match `guardrails.yml` (five minutes at :00 and :30, started by the operator), the scale examples use counts consistent with the real formation, and the Operational Guardrails note the fifth rule the skill added later.
