---
layout: post
title: "Server-Controlled Promo System with Offer Codes"
date: 2026-04-03 08:00:00 -0600
summary: "A promo architecture built on App Store offer codes instead of introductory offers: a server-side date window activates the campaign, the iOS client renders only promos it knows, and a CLI manages the offers in App Store Connect."
tags: [swift, ios, storekit, promotions, ruby, cli]
model: "Claude"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

## The Problem

A user taps a "50% off!" banner, and the purchase fails or charges full price. They were ineligible for the introductory offer, and nothing in the app could have known.

StoreKit 2 reports introductory-offer eligibility, but the earlier client-side promo could not reliably tell who was ineligible. Anyone who had ever subscribed, or used a free trial months ago, was refused the introductory offer, and the earlier promo's gate on local transaction history was torn out along with the rest of it before this system was built.

The banner is a promise the app cannot keep for an unknown share of the people who see it. Campaigns also had to launch and end without an app update, because marketing should not wait for App Review.

## The Solution

Three layers, each answering one part of the problem:

1. **Server-controlled activation** - a promo key in the API response, toggled by a deploy
2. **Client-side supported promos** - the iOS app shows UI only for promos it knows how to render
3. **Offer codes via CLI** - App Store Connect offer management through a skill and script system

Offer codes replace introductory offers because they carry no eligibility restriction. Anyone can redeem one, so the banner never promises a discount the store will refuse.

## Server: Promo Activation

Promo configuration is a YAML file on the server:

```yaml
# config/appstore/pricing_strategy.yml
# Optional promo, active only when today's UTC date is within startDate/endDate, inclusive.
promo:
  name: happy10
  startDate: 2026-04-01
  endDate: 2026-04-30
```

The API checks the date window and returns the promo key:

```ruby
# app/models/api/promo.rb
class Api::Promo < Api::Base
  STRATEGY_PATH = Rails.root.join("config/appstore/pricing_strategy.yml").freeze

  attr_accessor :name

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

    private

    def strategy
      YAML.safe_load_file(STRATEGY_PATH, permitted_classes: [Date]) || {}
    end
  end
end
```

YAML parses the unquoted dates as `Date` objects, which is why the loader permits that class and `active` calls `to_s` before `Date.iso8601`.

The weather API response carries the promo key as a top-level scalar next to the forecast fields, `"happy10"` inside the window and `null` outside it:

```json
{
  "latitude": 41.88,
  "currently": { ... },
  "daily": { ... },
  "promo": "happy10"
}
```

Launching a campaign is a YAML edit and a deploy. Ending one early is `name: null` and a deploy. Past `endDate`, `active` returns nil on its own. No app update is involved.

## Client: Supported Promos

The iOS app does not render whatever key the server sends. It keeps a set of promos it knows how to draw:

```swift
@MainActor
class PromoManager: ObservableObject {
    static let shared = PromoManager()

    private static let supportedPromos: Set<String> = ["happy10"]

    private lazy var savedDataManager = SavedDataManager.shared
    private lazy var storeManager = StoreManager.shared
    private lazy var weatherManager = WeatherManager.shared

    var promoDebug: String? {
        get {
            savedDataManager.store.string(forKey: SavedDataManager.Keys.promoDebug.rawValue)
        }
        set {
            savedDataManager.store.set(newValue, forKey: SavedDataManager.Keys.promoDebug.rawValue)
            objectWillChange.send()
        }
    }

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

The second check exists for three reasons:

1. **Graceful rollout** - the server can send a new promo key before the app supports it
2. **Version safety** - old app versions ignore promos they don't understand
3. **Debug override** - `promoDebug` tests a promo locally without a server change

Notice the first guard: `promoActive` also requires `storeManager.unpaid`, so a paying subscriber never sees the banner. The set holds one key at a time: when the next campaign shipped in June 2026, its key replaced `happy10` rather than joining it.

### Offer Codes

Each promo maps to App Store offer codes, one for the single plan and one for the family plan:

```swift
var discountPercentage: Int {
    50
}

let offerCode = "HAPPY10"
let offerCodeFamily = "HAPPY10FAM"
```

Users redeem the code directly in the App Store. There is no eligibility check, so there is no silent failure.

### Dismissal Logic

Users can dismiss the banner. The dismissal is stored with a 90-day cooldown, after which the banner returns if the promo is still active:

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

Managing offer codes through App Store Connect's web UI is tedious, and clicking through forms is neither repeatable nor auditable. A CLI skill and script system replaces it:

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

Offers live in the same pricing strategy file as the promo:

```yaml
# config/appstore/pricing_strategy.yml
offer_codes:
  happy10_yearly_single:
    product_id: hw_v4_yearly_single
    reference_name: HAPPY10_20260402
    offer_mode: pay_as_you_go
    duration: one_year
    number_of_periods: 1
    customer_eligibilities: [new, existing, expired]
    offer_eligibility: all
    discount_percent: 50
    rounding_policy: at_least_discount
    max_discount_drift_percent: 3.0
    enabled: true

  happy10_yearly_family:
    product_id: hw_v4_yearly_family
    reference_name: HAPPY10FAM_20260402
    offer_mode: pay_as_you_go
    duration: one_year
    number_of_periods: 1
    customer_eligibilities: [new, existing, expired]
    offer_eligibility: all
    discount_percent: 50
    rounding_policy: at_least_discount
    max_discount_drift_percent: 3.0
    enabled: true
```

The `customer_eligibilities` line is where the no-surprises promise becomes concrete: new, existing, and expired subscribers all qualify. `offer_eligibility: all` lets the code stack with the introductory trial, so a trial-eligible redeemer gets the trial first and then the discounted year. Prices are computed from `approved_prices.yml` at runtime: the discount is applied to each territory's approved price, then snapped to the highest Apple price tier at or below that target, so the real discount is never smaller than the promised one. `reference_name` is the immutable App Store Connect identifier, dated by convention, so renewing the campaign means a new name.

### Creating Redemption Codes

```bash
# Create a custom code with redemption limit
bin/appstore offer codes custom happy10_yearly_single \
  --code HAPPY10 \
  --limit 5000 \
  --expires 2026-04-30 \
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

If something goes wrong, deactivation has the same dry-run preview as creation:

```bash
# Preview
bin/appstore offer deactivate happy10_yearly_single --dry-run

# Deactivate
bin/appstore offer deactivate happy10_yearly_single
```

## Campaign Launch Workflow

A complete launch, from refreshing App Store Connect data to the server deploy:

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
  --code HAPPY10 --limit 5000 --expires 2026-04-30 --verbose
bin/appstore offer codes custom happy10_yearly_family \
  --code HAPPY10FAM --limit 5000 --expires 2026-04-30 --verbose

# 6. Verify everything matches
bin/appstore offer verify happy10_yearly_single --verbose
bin/appstore offer verify happy10_yearly_family --verbose

# 7. Update server config and deploy
# config/appstore/pricing_strategy.yml
#   promo:
#     name: happy10
#     startDate: 2026-04-01
#     endDate: 2026-04-30
```

## Why This Works

Each layer takes one failure off the table. Offer codes remove eligibility, so nobody who taps the banner is refused. The date window puts launch and shutdown behind a deploy instead of App Review. The supported set lets old app versions ignore a campaign they cannot render. The CLI makes offer creation repeatable and auditable.

What shipped is the YAML promo block, `Api::Promo`, `PromoManager` with its supported set and 90-day dismissal, and the `bin/appstore offer` commands. The cost is a split in what a deploy can do: launching or ending a campaign is server-only, but a promo key the app has never seen still needs an app release before it renders. Users also leave the app to redeem the code in the App Store. As of April 2026, the first campaign on this system had produced no eligibility-related support tickets; a second campaign ran on the same server window and supported set in June 2026.
