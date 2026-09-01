---
layout: post
title: "App Store Pricing: 175 Territories, One CLI"
date: 2026-02-27 12:00:00 -0600
summary: "Managing App Store subscription pricing across 175 territories with PPP adjustments, offer codes, and a CLI that prevents mistakes."
tags: [skills, scripts, app-store, pricing]
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

---

## How This Post Was Made

**Prompt:** "Write 7+ in-depth blog posts documenting real engineering patterns from helloweather/web. These posts go deeper than the existing 'Skills and Scripts' overview, showing specific implementations."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Source: `.claude/skills/appstore-pricing/SKILL.md`

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the six products and thousand-plus prices instead of a general remark about App Store pricing, the facts about dry runs, silent failures, future-dated subscription changes, and territory grouping moved from Lessons Learned into the sections they belong to, and Lessons Learned kept the three rules that transfer. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The 69% floor, 120% ceiling, six products, 175 territories, coherence rules, review prompt, and command names all matched the pricing skill, CLI, and config. Corrected: Apple offers roughly 800 price points per territory, not over 900; `--date` is optional with a 72-hour default and a 24-hour minimum for subscriptions, not required; the one-time code example used `--count 100`, below the CLI's minimum of 500; identical-price groups number about 40 today, not 30-40; the "silent failure" rationale for `verify` was replaced with the actual submit gaps it was built to catch; the SQLite lesson now names the 176 per-territory YAML files it replaced; and the Results list, whose outcome claims could not be checked, is dated as of February 2026.
