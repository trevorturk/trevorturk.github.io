---
layout: post
title: "App Store Pricing: 175 Territories, One CLI"
date: 2026-02-27 12:00:00 -0600
summary: "Managing App Store subscription pricing across 175 territories with PPP adjustments, offer codes, and a CLI that prevents mistakes."
tags: [skills, scripts, app-store, pricing]
---

## The Problem

App Store pricing is deceptively complex. Apple offers over 900 price tiers across 175 territories, each with its own currency, exchange rate fluctuations, and purchasing power differences. Setting prices manually in App Store Connect is tedious and error-prone. Worse, prices that look reasonable in one territory can be completely wrong for another due to local economic conditions.

Hello Weather has 6 products: Monthly Single, Monthly Family, Yearly Single, Yearly Family, Lifetime Single, and Lifetime Family. That's over 1,000 individual prices to manage. We needed a system that could:

- Calculate appropriate prices based on purchasing power parity (PPP)
- Find the nearest valid Apple price tier
- Ensure the price "ladder" makes sense (annual should cost less than 12x monthly)
- Submit changes to App Store Connect programmatically
- Manage offer code campaigns

## The Solution

A skill/script pair that handles the entire pricing workflow: `bin/appstore` CLI backed by SQLite for data and YAML for configuration.

### Architecture

```
bin/appstore
├── refresh    # Pull latest data from APIs
├── validate   # Check prices against rules
├── submit     # Push to App Store Connect
├── verify     # Confirm ASC matches approved
└── offer      # Manage offer code campaigns
```

**Data storage:**
- `db/appstore.sqlite3` - PPP ratios, exchange rates, territories, Apple price tiers
- `config/appstore/approved_prices.yml` - Source of truth for prices
- `config/appstore/pricing_strategy.yml` - Floor/ceiling rules, overrides

### The Pattern: Validate -> Dry-Run -> Submit -> Verify

This pattern prevents mistakes by requiring multiple checkpoints:

```bash
# 1. Refresh external data (PPP, exchange rates, Apple tiers)
bin/appstore refresh --verbose

# 2. Validate approved prices against rules
bin/appstore validate --verbose

# 3. Preview what would change
bin/appstore submit --dry-run --verbose

# 4. Submit with a future effective date (subscriptions require this)
bin/appstore submit --date 2026-03-01 --verbose

# 5. Verify App Store Connect matches approved prices
bin/appstore verify --date 2026-03-01 --verbose
```

## Implementation

### PPP-Based Pricing

Purchasing Power Parity adjusts prices based on local economic conditions. A $20/year subscription might be affordable in the US but expensive in Vietnam. PPP helps find the right local price.

```ruby
target_percent = clamp(ppp_ratio, floor, ceiling)  # 69% - 120%
target_usd = base_price * (target_percent / 100)
approved_price = find_nearest_apple_tier(territory, target_usd)
```

The floor (69%) ensures accessibility in lower-income markets. The ceiling (120%) allows premium pricing in high-income markets.

### Coherence Rules

Individual prices aren't enough - the whole ladder must make sense:

| Rule | Requirement | Why |
|------|-------------|-----|
| Annual guardrail | `YS < 12*MS`, `YF < 12*MF` | Annual should be a deal |
| Coherence target | LS near 5x YS, LF near 5x YF | Lifetime should feel fair |
| Neutrality | No subscription/lifetime bias | Don't steer users |
| Style pairing | Round both lifetimes together | Looks intentional |

### Natural Pricing

Prices should look intentional, not algorithmic:

- **Round numbers**: JPY 2000, KRW 19900
- **.99 endings**: USD 19.99, EUR 19.99
- **Lucky 8s**: CNY 88, HKD 168 (cultural preference)

### Quarterly Review Workflow

Claude reads the data files and computes recommendations:

```
Progress: 12 / 36 complete, 24 remaining
Territory Group: AFG, AGO, ALB, ARM... (47 territories)
Before: [0.99, 1.99, 12.99, 19.99, 64.99, 99.99]
After:  [1.29, 1.99, 12.99, 19.99, 64.99, 99.99]

Rationale:
- Restores coherent annual economics (YS below 12x MS)
- Maintains LS/LF coherence near ~5x of YS/YF
- Uses valid Apple tiers with natural local endings

Options:
1. Apply this group (Recommended)
2. Skip this group
3. Custom adjustment
```

After each approval, Claude immediately edits `approved_prices.yml` - never batching changes.

### Offer Codes

The same CLI handles offer code campaigns:

```bash
# Configure in pricing_strategy.yml, then:
bin/appstore offer plan CAMPAIGN_KEY --verbose
bin/appstore offer apply CAMPAIGN_KEY --dry-run --verbose
bin/appstore offer apply CAMPAIGN_KEY --verbose
bin/appstore offer verify CAMPAIGN_KEY --verbose

# Generate codes
bin/appstore offer codes one-time CAMPAIGN_KEY --count 100 --expires 2026-04-01
bin/appstore offer codes custom CAMPAIGN_KEY --code SPRINGDEAL --limit 500
```

Offer prices are computed from `approved_prices.yml` at runtime - no duplicate price files.

## Results

- **175 territories** managed from one config file
- **Quarterly reviews** take 30 minutes instead of hours
- **Zero pricing errors** since implementing verify step
- **Offer campaigns** launched in minutes, not days

## Lessons Learned

- **SQLite over YAML for data** - Our first version stored everything in YAML. Context windows exploded. Moving to SQLite with ActiveRecord models let us query only what we need.
- **Dry-run everything** - The `--dry-run` flag on submit has caught many mistakes before they hit App Store Connect.
- **Verify after submit** - App Store Connect can silently fail. Always verify that effective prices match approved prices.
- **Group similar territories** - Reviewing 175 territories individually is insane. Grouping by identical pricing cuts reviews to ~30-40 groups.
- **Future dates for subscriptions** - Subscriptions require scheduling price changes for a future date. IAPs can change immediately.

---

## How This Post Was Made

**Prompt:** "Write 7+ in-depth blog posts documenting real engineering patterns from helloweather/web. These posts go deeper than the existing 'Skills and Scripts' overview, showing specific implementations."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Source: `.claude/skills/appstore-pricing/SKILL.md`
