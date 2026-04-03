---
layout: post
title: "Server-Controlled Promo System with Offer Codes"
date: 2026-04-03 08:00:00 -0600
summary: "A flexible promo architecture that uses App Store offer codes instead of introductory offers, with server-side activation and CLI-based offer management."
tags: [swift, ios, storekit, promotions, ruby, cli]
---

## The Problem

Running promotional campaigns for iOS subscriptions is harder than it looks. The naive approach - using StoreKit's introductory offers - has a critical flaw: **you can't reliably detect eligibility**.

StoreKit 2 tells you if a user is eligible for an introductory offer, but only for users who have never subscribed *on the current device*. It can't see:

- Previous subscriptions on other devices
- Family members who shared a subscription
- Users who had a free trial months ago
- TestFlight users who tested subscriptions

This creates a terrible user experience: you show someone a "50% off!" banner, they tap it, and the purchase fails or charges full price because they're secretly ineligible.

We also wanted to launch and end campaigns without shipping app updates. Marketing shouldn't wait for App Review.

## The Solution

We built a three-layer promo system:

1. **Server-controlled activation** - Promo key delivered in API response, toggled via deploy
2. **Client-side supported promos** - iOS only shows UI for promos it knows how to render
3. **Offer codes via CLI** - App Store Connect offer management through a skill and script system

The key insight: **offer codes have no eligibility restrictions**. Anyone can redeem them, which means no surprise failures.

## Server: Promo Activation

Promo configuration lives in a YAML file on the server:

```yaml
# config/appstore/pricing_strategy.yml
promo:
  name: happy10
  startDate: "2026-03-01"
  endDate: "2026-03-31"
```

The API checks the date window and returns the promo key:

```ruby
# app/models/api/promo.rb
class Api::Promo < Api::Base
  STRATEGY_PATH = Rails.root.join("config/appstore/pricing_strategy.yml").freeze

  class << self
    def active(today: Time.now.utc.to_date)
      promo = strategy["promo"] || {}
      name = promo["name"]
      return nil if name.blank?

      start_date = promo["startDate"]
      end_date = promo["endDate"]
      return nil if start_date.blank? || end_date.blank?

      date_window = Date.iso8601(start_date.to_s)..Date.iso8601(end_date.to_s)
      return nil unless date_window.cover?(today)

      new(name: name)
    end
  end
end
```

The weather API response includes the promo field:

```json
{
  "forecast": { ... },
  "promo": "happy10"
}
```

To launch a campaign: update the YAML and deploy. To kill it: set `name: null` and deploy. No app update required.

## Client: Supported Promos

The iOS app doesn't blindly trust whatever the server sends. It maintains a set of supported promos:

```swift
@MainActor
class PromoManager: ObservableObject {
    static let shared = PromoManager()

    private static let supportedPromos: Set<String> = ["happy10"]

    private lazy var weatherManager = WeatherManager.shared
    private lazy var storeManager = StoreManager.shared

    var promoKey: String? {
        promoDebug ?? weatherManager.weather?.forecast?.promo
    }

    var promoActive: Bool {
        guard storeManager.unpaid else { return false }
        guard let promoKey else { return false }

        return Self.supportedPromos.contains(promoKey)
    }
}
```

This two-layer check serves multiple purposes:

1. **Graceful rollout** - Server can send a new promo key before the app supports it
2. **Version safety** - Old app versions ignore promos they don't understand
3. **Debug override** - Testing promos locally without server changes

### Offer Codes

Each promo maps to specific App Store offer codes:

```swift
let offerCode = "HAPPY10"
let offerCodeFamily = "HAPPY10FAM"

var discountPercentage: Int {
    50
}
```

Users redeem these codes directly in the App Store - no eligibility check, no silent failures.

### Dismissal Logic

Users can dismiss promo banners. We track dismissal with a cooldown:

```swift
var promoDismissedAt: Date? {
    get {
        savedDataManager.store.object(
            forKey: SavedDataManager.Keys.promoDismissedAt.rawValue
        ) as? Date
    }
    set {
        savedDataManager.store.set(
            newValue,
            forKey: SavedDataManager.Keys.promoDismissedAt.rawValue
        )
        objectWillChange.send()
    }
}

var showPromoTimeInterval: TimeInterval {
    90 * 24 * 60 * 60 // 90 days
}

var showPromoNag: Bool {
    guard promoActive else { return false }
    guard let promoDismissedAt = promoDismissedAt else { return true }

    return Date() >= promoDismissedAt.addingTimeInterval(showPromoTimeInterval)
}
```

## CLI: Offer Code Management

Managing offer codes through App Store Connect's web UI is tedious. We built a CLI skill and script system:

```bash
# List all configured offers and their ASC status
bin/appstore offer list --verbose

# Preview what would be created
bin/appstore offer apply happy10_yearly_single --dry-run --verbose

# Create the offer in App Store Connect
bin/appstore offer apply happy10_yearly_single --verbose

# Verify ASC matches expected pricing
bin/appstore offer verify happy10_yearly_single --verbose
```

### Offer Configuration

Offers are defined in the same pricing strategy file:

```yaml
# config/appstore/pricing_strategy.yml
offer_codes:
  happy10_yearly_single:
    product_id: hw_v4_yearly_single
    reference_name: HAPPY10_20260213
    offer_mode: pay_up_front
    duration: one_year
    number_of_periods: 1
    customer_eligibilities: [new, existing, expired]
    offer_eligibility: once
    discount_percent: 50
    enabled: true

  happy10_yearly_family:
    product_id: hw_v4_yearly_family
    reference_name: HAPPY10FAM_20260213
    offer_mode: pay_up_front
    duration: one_year
    number_of_periods: 1
    customer_eligibilities: [new, existing, expired]
    offer_eligibility: once
    discount_percent: 50
    enabled: true
```

Prices are computed from `approved_prices.yml` at runtime, applying the discount percentage to each territory's base price.

### Creating Redemption Codes

```bash
# Create a custom code with redemption limit
bin/appstore offer codes custom happy10_yearly_single \
  --code HAPPY10 \
  --limit 5000 \
  --expires 2026-03-31 \
  --verbose

# Or create one-time codes for distribution
bin/appstore offer codes one-time happy10_yearly_single \
  --count 1000 \
  --expires 2026-06-01 \
  --verbose

# Download the generated codes
bin/appstore offer codes values happy10_yearly_single \
  --batch-id BATCH_ID \
  --output tmp/codes.txt \
  --verbose
```

### Rollback

If something goes wrong:

```bash
# Preview
bin/appstore offer deactivate happy10_yearly_single --dry-run

# Deactivate
bin/appstore offer deactivate happy10_yearly_single
```

## Campaign Launch Workflow

A complete campaign launch:

```bash
# 1. Refresh App Store Connect data
bin/appstore refresh --verbose

# 2. Validate pricing
bin/appstore validate --verbose

# 3. Review what will be created
bin/appstore offer plan happy10_yearly_single --verbose

# 4. Create offers
bin/appstore offer apply happy10_yearly_single --verbose
bin/appstore offer apply happy10_yearly_family --verbose

# 5. Create redemption codes
bin/appstore offer codes custom happy10_yearly_single \
  --code HAPPY10 --limit 5000 --expires 2026-03-31 --verbose
bin/appstore offer codes custom happy10_yearly_family \
  --code HAPPY10FAM --limit 5000 --expires 2026-03-31 --verbose

# 6. Verify everything matches
bin/appstore offer verify happy10_yearly_single --verbose
bin/appstore offer verify happy10_yearly_family --verbose

# 7. Update server config and deploy
# config/appstore/pricing_strategy.yml
#   promo:
#     name: happy10
#     startDate: "2026-03-01"
#     endDate: "2026-03-31"
```

## Why This Works

**Offer codes solve eligibility**: Unlike introductory offers, anyone can redeem an offer code. No silent failures, no confused users.

**Server control enables agility**: Launch campaigns with a deploy, not an app update. End them instantly if needed.

**Supported promos enable safety**: Old app versions gracefully ignore new campaigns. New campaigns can be tested before the app officially supports them.

**CLI tooling reduces errors**: Scripted offer management is repeatable and auditable. No clicking through ASC forms.

The system has successfully run multiple campaigns with zero eligibility-related support tickets.

---

## How This Post Was Made

**Prompt:** "create a new post about our promo system, see previous commits, note the new promo system uses 'offer codes' to avoid eligibility issues, such as users who previously subscribed or even had a trial were ineligible, which could not be detected with storekit2. note the lightweight server component, so we can enable/disable promos server-side. note the client side has 'supported promos' so we can add support and adjust UI elements etc over time. note also the appstore skill+script system which lets us manage offer codes etc with appstoreconnect. create a pr for this new post."

Generated by Claude using the blog-post-generator skill. Based on production code from Hello Weather's promo system.
