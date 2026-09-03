---
layout: post
title: "App Store Pricing: 175 Territories, One CLI"
date: 2026-02-27 12:00:00 -0600
summary: "Managing App Store subscription pricing across 175 territories with PPP adjustments, offer codes, and a CLI that prevents mistakes."
tags: [skills, scripts, app-store, pricing]
model: "Claude Opus 4.5"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

## The Problem

[Hello Weather](https://helloweather.com) sells 6 products: Monthly Single, Monthly Family, Yearly Single, Yearly Family, Lifetime Single, and Lifetime Family. Across Apple's 175 territories, that is over 1,000 individual prices. Each one has to land on one of the roughly 800 Apple price points for its territory, in a territory with its own currency, exchange rate fluctuations, and purchasing power. A price that looks reasonable in one territory can be completely wrong in another, and setting that many by hand in App Store Connect is tedious and error-prone.

We needed a system that could:

- Calculate appropriate prices based on purchasing power parity (PPP)
- Find the nearest valid Apple price tier
- Ensure the price "ladder" makes sense (annual should cost less than 12x monthly)
- Submit changes to App Store Connect programmatically
- Manage offer code campaigns

## The Solution

A skill/script pair that handles the entire pricing workflow: a `bin/appstore` CLI backed by SQLite for data and YAML for configuration.

### Architecture

```
bin/appstore
├── refresh    # Pull latest data from APIs
├── validate   # Check prices against rules
├── submit     # Push to App Store Connect
├── verify     # Confirm ASC matches approved
└── offer      # Manage offer code campaigns
```

Data lives in three places:

- `db/appstore.sqlite3` - PPP ratios, exchange rates, territories, Apple price tiers
- `config/appstore/approved_prices.yml` - Source of truth for prices
- `config/appstore/pricing_strategy.yml` - Floor/ceiling rules, overrides

### The Pattern: Validate -> Dry-Run -> Submit -> Verify

A price change passes several checkpoints before it reaches App Store Connect, and one after:

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

The `--dry-run` flag has caught many mistakes before they hit App Store Connect. The `--date` flag sets the effective date; it defaults to 72 hours out, and subscription changes must be at least 24 hours in the future. IAPs can change immediately. The final `verify` step reads the effective prices back and compares them to the approved ones, because a submit that reports success can still leave a product unchanged. The first version skipped two legacy products and diffed subscriptions against the current price instead of the one effective on the target date, which is why `verify` exists.

## Implementation

### PPP-Based Pricing

A $20/year subscription might be affordable in the US but expensive in Vietnam. Purchasing power parity adjusts the base price for local economic conditions:

```ruby
target_percent = clamp(ppp_ratio, floor, ceiling)  # 69% - 120%
target_usd = base_price * (target_percent / 100)
approved_price = find_nearest_apple_tier(territory, target_usd)
```

The floor (69%) ensures accessibility in lower-income markets. The ceiling (120%) allows premium pricing in high-income markets.

### Coherence Rules

Individual prices aren't enough. The whole ladder must make sense:

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

Territories with identical pricing are reviewed as a group, which cuts 175 territories to about 40 groups. Claude reads the data files and computes a recommendation for each:

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

After each approval, Claude immediately edits `approved_prices.yml`, never batching changes.

### Offer Codes

The same CLI handles offer code campaigns:

```bash
# Configure in pricing_strategy.yml, then:
bin/appstore offer plan CAMPAIGN_KEY --verbose
bin/appstore offer apply CAMPAIGN_KEY --dry-run --verbose
bin/appstore offer apply CAMPAIGN_KEY --verbose
bin/appstore offer verify CAMPAIGN_KEY --verbose

# Generate codes
bin/appstore offer codes one-time CAMPAIGN_KEY --count 1000 --expires 2026-04-01
bin/appstore offer codes custom CAMPAIGN_KEY --code SPRINGDEAL --limit 500
```

Offer prices are computed from `approved_prices.yml` at runtime, so there are no duplicate price files.

## Results

As of February 2026:

- **175 territories** managed from one config file
- **Quarterly reviews** take 30 minutes instead of hours
- **Zero pricing errors** since implementing the verify step
- **Offer campaigns** launched in minutes, not days

## Lessons Learned

- **SQLite over YAML for data.** Our first version stored price points in 176 per-territory YAML files of about 1,600 lines each, and context windows exploded. SQLite with ActiveRecord models lets us query only what we need.
- **Group identical rows before asking a human to review.** About 40 groups get reviewed in one sitting. 175 rows do not.
- **A submit is not done until it is read back.** A write can succeed and still leave the remote wrong. Compare effective values to the source of truth after every write.
