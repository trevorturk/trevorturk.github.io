---
layout: post
title: "App Store Pricing: 175 Territories, One CLI"
date: 2026-02-27 12:00:00 -0600
summary: "How we set six products' prices in 175 App Store territories with one CLI: purchasing-power adjustments, Apple's price tiers, offer codes, and a verify step that reads every price back."
tags: [skills, scripts, app-store, pricing]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

[Hello Weather](https://helloweather.com) sells 6 products: Monthly Single, Monthly Family, Yearly Single, Yearly Family, Lifetime Single, and Lifetime Family. Apple sells in 175 territories, so that's over 1,000 prices. Each one has to be one of the roughly 800 price points Apple allows in that territory. Each territory also has its own currency, its own exchange rate, and its own idea of what a fair price is. A price that looks right in one territory can be badly wrong in another. Setting a thousand of them by hand in App Store Connect is slow and easy to get wrong.

We wanted a tool that could:

- Work out a price for each territory from purchasing power parity (PPP), a measure of what money buys locally
- Snap that price to the nearest tier Apple allows
- Check that the six prices make sense next to each other (a year should cost less than 12 months)
- Send the changes to App Store Connect from a script
- Manage offer code campaigns

## The Solution

We built a command-line tool, `bin/appstore`, and a skill that tells Claude how to run it. The tool keeps its reference data in SQLite and its settings in YAML.

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
- `config/appstore/approved_prices.yml` - the prices we've approved
- `config/appstore/pricing_strategy.yml` - the floor and ceiling rules, plus overrides

### The Pattern: Validate -> Dry-Run -> Submit -> Verify

A price change goes through a few checks before it reaches App Store Connect, and one more after:

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

The dry run has caught many mistakes before they reached App Store Connect. The `--date` flag sets when the new prices take effect. It defaults to 72 hours out, and Apple requires at least 24 hours' notice for subscription changes. One-time purchases can change right away. The last step, `verify`, reads the prices back from App Store Connect and compares them to the approved file. We added it because a submit can report success and still leave a product unchanged. Our first version skipped two older products, and it compared subscriptions against the current price instead of the price that would be in effect on the target date. `verify` is there to catch that kind of gap.

## Implementation

### PPP-Based Pricing

A $20 yearly subscription that's affordable in the US can be expensive in Vietnam. We scale the base price by the territory's PPP ratio, so the target moves with local buying power:

```ruby
target_percent = clamp(ppp_ratio, floor, ceiling)  # 69% - 120%
target_usd = base_price * (target_percent / 100)
approved_price = find_nearest_apple_tier(territory, target_usd)
```

The floor of 69% keeps prices within reach in lower-income territories. The ceiling of 120% lets us charge more in high-income ones.

### Coherence Rules

Each price can be right on its own and still wrong next to the others. The six prices also have to make sense as a set:

| Rule | Requirement | Why |
|------|-------------|-----|
| Annual guardrail | `YS < 12*MS`, `YF < 12*MF` | Annual should be a deal |
| Coherence target | LS near 5x YS, LF near 5x YF | Lifetime should feel fair |
| Neutrality | No subscription/lifetime bias | Don't steer users |
| Style pairing | Round both lifetimes together | Looks intentional |

### Natural Pricing

A price should look like a person chose it, not a formula:

- **Round numbers**: JPY 2000, KRW 19900
- **.99 endings**: USD 19.99, EUR 19.99
- **Lucky 8s**: CNY 88, HKD 168 (8 is considered lucky there)

### Quarterly Review Workflow

Every quarter we review the prices. Territories with the same prices are reviewed together, which turns 175 territories into about 40 groups. Claude reads the data files and proposes a change for each group:

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

When we approve a group, Claude edits `approved_prices.yml` right away, not in a batch at the end.

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

Offer prices are worked out from `approved_prices.yml` when the command runs, so there's no second price file to keep in sync.

## Results

As of February 2026:

- **175 territories** managed from one config file
- **Quarterly reviews** take 30 minutes instead of hours
- **No pricing errors** since we added the verify step
- **Offer campaigns** launch in minutes instead of days

## Lessons Learned

- **Keep reference data in SQLite, not YAML.** Our first version kept Apple's price points in 176 YAML files, one per territory, about 1,600 lines each. Reading them filled the context window. With SQLite and ActiveRecord models we pull only the rows we need.
- **Group identical rows before asking a person to review them.** About 40 groups fit in one sitting. 175 rows don't.
- **A submit isn't done until you've read it back.** A write can succeed and still leave the remote side wrong. After every write, read the live values and compare them to what you meant to write.
