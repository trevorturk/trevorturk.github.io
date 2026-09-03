---
layout: post
title: "StoreKit 2 Implementation Guide"
date: 2026-03-05 08:00:00 -0600
summary: "Our full StoreKit 2 code, nothing hidden: verified transactions, two update streams, a saved record that widgets can read and a paid flag that reaches the watch, a guard against downgrading on a stale flag, and a paywall built on ProductView."
tags: [swift, ios, storekit, subscriptions, in-app-purchase]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Three processes need to know whether the user has paid: the app, its widgets, and the watch app. Only the app talks to the App Store. It writes what it learns to app-group storage, and the widgets read that directly. The watch app gets the `paid` flag over WatchConnectivity and writes it into its own copy of the same store. Neither the widgets nor the watch call StoreKit, so the saved answer has to be right, even on a cold launch before the App Store has been asked.

StoreKit 2 checks each transaction's signature for you. The rest is up to the app:

1. **Watch for changes** so renewals, cancellations, and refunds land as they happen
2. **Save the state** in a form every process can read without calling StoreKit
3. **Handle the edge cases**: interrupted purchases, family sharing, sandbox testing
4. **Build the paywall** on SwiftUI and the new `ProductView`

Apple's docs cover each API on its own but don't show a complete production example. This post is one, with nothing hidden.

## The Solution

Three layers, so the StoreKit calls stay out of the business logic:

- **StoreService** calls StoreKit, verifies transactions, and runs the two update listeners
- **StoreManager** holds state, saves transactions, and works out from them what the user is entitled to
- **TransactionRecord** is the `Codable` model that gets saved

## Implementation

### StoreService: The API Layer

StoreService owns every call into StoreKit. It's `@MainActor`, so nothing it hands to StoreManager crosses a thread.

```swift
import StoreKit

@MainActor
class StoreService {
    static let shared = StoreService()

    private lazy var storeManager = StoreManager.shared

    private var transactionUpdatesTask: Task<Void, Never>?
    private var subscriptionStatusUpdatesTask: Task<Void, Never>?

    func activate() {
        Task {
            await observeTransactionUpdates()
            await observeSubscriptionStatusUpdates()
            await checkForUnfinishedTransactions()
            await updateCurrentEntitlements()
            await fetchProducts()
        }
    }
}
```

The awaits in `activate()` run in order, and the next four sections follow the same order.

### Real-Time Transaction Monitoring

StoreKit 2 delivers changes as async sequences. We listen to two of them, each in its own task that lives as long as the app does.

```swift
func observeTransactionUpdates() async {
    self.transactionUpdatesTask = Task { [weak self] in
        for await verificationResult in Transaction.updates {
            guard let self else { break }
            await self.process(verificationResult)
        }
    }
}

func observeSubscriptionStatusUpdates() async {
    subscriptionStatusUpdatesTask = Task { [weak self] in
        for await status in StoreKit.Product.SubscriptionInfo.Status.updates {
            guard let self else { break }
            await self.process(status.transaction)
        }
    }
}
```

Both loops catch renewals, cancellations, and refunds, even while the app is in the background.

### Transaction Verification and Processing

Every transaction goes through the same switch on its verification result, whichever stream delivered it.

```swift
func process(_ verificationResult: VerificationResult<Transaction>) async {
    switch verificationResult {
    case .verified(let transaction):
        let renewalInfo = await fetchRenewalInfo(transaction)
        storeManager.process(transaction: transaction, renewalInfo: renewalInfo)
        await transaction.finish()
    case .unverified(_, let error):
        // Log but don't crash - could be jailbroken device or corruption
        Logger.error(error.localizedDescription)
    }
}

func fetchRenewalInfo(_ transaction: Transaction) async
    -> StoreKit.Product.SubscriptionInfo.RenewalInfo? {
    guard let verificationResult = await transaction.subscriptionStatus?.renewalInfo
        else { return nil }

    switch verificationResult {
    case .verified(let renewalInfo):
        return renewalInfo
    case .unverified(_, let error):
        Logger.error(error.localizedDescription)
        return nil
    }
}
```

We log an `.unverified` result and drop it rather than crash. We also fetch the renewal info here, because its `willAutoRenew` is the only way to tell "Subscribed" from "Cancelling" later.

### Handling Unfinished Transactions

The App Store keeps every transaction until the app calls `finish()` on it. A crash in the middle of a purchase leaves one behind, and they pile up, so we drain them on every launch.

```swift
func checkForUnfinishedTransactions() async {
    for await verificationResult in Transaction.unfinished {
        await self.process(verificationResult)
    }
}

func updateCurrentEntitlements() async {
    for await verificationResult in Transaction.currentEntitlements {
        await self.process(verificationResult)
    }

    storeManager.hasUpdatedCurrentEntitlements = true
    storeManager.transactionsDidChange()
}
```

`updateCurrentEntitlements` also sets `hasUpdatedCurrentEntitlements`. That's the flag StoreManager checks later before it trusts a downgrade.

### Restore Purchases

Restore has to work on a new device where local storage is empty. `AppStore.sync()` may ask the user to sign in to their Apple ID, and it may fail, so the function returns a `Bool`.

```swift
func restorePurchases() async -> Bool {
    do {
        try await AppStore.sync()
    } catch {
        Logger.error(error.localizedDescription)
        return false
    }

    var restored: [TransactionRecord] = []

    for await verificationResult in Transaction.all {
        switch verificationResult {
        case .verified(let transaction):
            let renewalInfo = await fetchRenewalInfo(transaction)
            restored.append(TransactionRecord(
                transaction: transaction,
                renewalInfo: renewalInfo
            ))
            await transaction.finish()
        case .unverified(_, let error):
            Logger.error(error.localizedDescription)
        }
    }

    storeManager.hasUpdatedCurrentEntitlements = true
    storeManager.replaceTransactions(restored)
    return true
}
```

We replace the local list rather than merge into it. The paywall uses the `Bool` to tell a sync that failed from one that worked and found nothing.

### Pre-fetching Products

We fetch the products at activation so the prices are already in memory when the paywall appears.

```swift
func fetchProducts() async {
    guard storeManager.unpaid else { return }

    do {
        let products = try await StoreKit.Product.products(
            for: StoreManager.Plan.allPaywall
        )
        storeManager.products = products

        let yearly = products.first { $0.id == StoreManager.Plan.yearly }
        storeManager.introEligible = await yearly?.subscription?.isEligibleForIntroOffer
    } catch {
        Logger.error("fetchProducts: \(error.localizedDescription)")
    }
}
```

Paid users skip this, since they never see the paywall. `introEligible` says whether the user can still get the yearly plan's intro offer. Apple allows one intro offer per subscription group per account, so the paywall uses this flag to drop the free-trial wording for anyone who has already used theirs.

---

## StoreManager: State and Persistence

StoreManager is the one place the app asks about purchase state. Everything it exposes is computed from the saved transaction list.

```swift
import Foundation
import StoreKit

@MainActor
class StoreManager: ObservableObject {
    static let shared = StoreManager()

    private lazy var savedDataManager = SavedDataManager.shared
    private lazy var settingsManager = SettingsManager.shared
    private lazy var syncService = SyncService.shared

    var products: [StoreKit.Product] = [] {
        didSet { objectWillChange.send() }
    }

    var introEligible: Bool? {
        didSet { objectWillChange.send() }
    }

    private var yearlyProduct: StoreKit.Product? {
        products.first { $0.id == Plan.yearly }
    }

    var yearlyDisplayPrice: String? {
        yearlyProduct?.displayPrice
    }

    private let maxTransactions = 9999
}
```

### Defining Product IDs

All twelve product IDs, six current and six legacy, are static properties on one enum.

```swift
enum Plan {
    static let monthly  = "hw_v4_monthly_single"
    static let yearly   = "hw_v4_yearly_single"
    static let lifetime = "hw_v4_lifetime_single"

    static let monthly_family  = "hw_v4_monthly_family"
    static let yearly_family   = "hw_v4_yearly_family"
    static let lifetime_family = "hw_v4_lifetime_family"

    // Legacy plans for migration support
    static let v3_monthly_1 = "hw_monthly_099"
    static let v3_monthly_2 = "hw_1_month_auto"
    static let v3_yearly_1  = "hw_1_year_auto"
    static let v3_yearly_2  = "hw_1_year_auto_2"
    static let v3_lifetime_1 = "hw_lifetime_499"
    static let v3_lifetime_2 = "hw_lifetime_299"

    static var allActive: [String] {
        [
            monthly, yearly, lifetime,
            monthly_family, yearly_family, lifetime_family,
            v3_monthly_1, v3_monthly_2, v3_yearly_1, v3_yearly_2,
            v3_lifetime_1, v3_lifetime_2,
        ]
    }

    static var allLifetime: [String] {
        [lifetime, lifetime_family, v3_lifetime_1, v3_lifetime_2]
    }

    static var paywallIndividual: [String] {
        [monthly, yearly, lifetime]
    }

    static var paywallFamily: [String] {
        [monthly_family, yearly_family, lifetime_family]
    }

    static var allPaywall: [String] {
        paywallIndividual + paywallFamily
    }

    static var allYearly: [String] {
        [yearly, yearly_family]
    }

    static func isYearly(_ id: String) -> Bool {
        allYearly.contains(id)
    }
}
```

The legacy IDs are in `allActive` but not in either paywall list. An old purchase still unlocks the app, but nobody can buy those plans again.

Two static helpers turn an ID into a title. `planTitle` labels the purchase-history rows, and it treats every legacy plan as a family plan because they all were. `planTitleBase` drops the family suffix for the paywall cards. `localized` is the app's String Catalog lookup.

```swift
static func planTitle(_ id: String?) -> String {
    switch id {
    case Plan.monthly:  return localized("Monthly")
    case Plan.yearly:   return localized("Yearly")
    case Plan.lifetime: return localized("Lifetime")

    case Plan.monthly_family:  return localized("Monthly (Family)")
    case Plan.yearly_family:   return localized("Yearly (Family)")
    case Plan.lifetime_family: return localized("Lifetime (Family)")

    case Plan.v3_monthly_1, Plan.v3_monthly_2:   return localized("Monthly (Family)")
    case Plan.v3_yearly_1, Plan.v3_yearly_2:     return localized("Yearly (Family)")
    case Plan.v3_lifetime_1, Plan.v3_lifetime_2: return localized("Lifetime (Family)")

    default: return localized("Unknown")
    }
}

static func planTitleBase(_ id: String?) -> String {
    switch id {
    case Plan.monthly, Plan.monthly_family, Plan.v3_monthly_1, Plan.v3_monthly_2:
        return localized("Monthly")
    case Plan.yearly, Plan.yearly_family, Plan.v3_yearly_1, Plan.v3_yearly_2:
        return localized("Yearly")
    case Plan.lifetime, Plan.lifetime_family, Plan.v3_lifetime_1, Plan.v3_lifetime_2:
        return localized("Lifetime")
    default:
        return localized("Unknown")
    }
}
```

### The Paid Flag

`paid` is the one boolean the rest of the app checks. Its setter runs `handlePaidChange` before it writes the new value.

```swift
var paid: Bool {
    get {
        savedDataManager.store.bool(forKey: SavedDataManager.Keys.paid.rawValue)
    }
    set {
        handlePaidChange(oldValue: paid, newValue: newValue)
        savedDataManager.store.set(newValue, forKey: SavedDataManager.Keys.paid.rawValue)
        objectWillChange.send()
    }
}

var unpaid: Bool {
    !paid
}

var hasUpdatedCurrentEntitlements: Bool {
    get {
        savedDataManager.store.bool(
            forKey: SavedDataManager.Keys.hasUpdatedCurrentEntitlements.rawValue
        )
    }
    set {
        savedDataManager.store.set(
            newValue,
            forKey: SavedDataManager.Keys.hasUpdatedCurrentEntitlements.rawValue
        )
        objectWillChange.send()
    }
}
```

`hasUpdatedCurrentEntitlements` is saved too, so it survives a relaunch. SwiftUI views that gate a feature read `unpaid` through an `@ObservedObject`:

```swift
if storeManager.unpaid {
    Button("Upgrade to Pro") { showPaywall = true }
}
```

### Handling State Transitions

When `paid` changes, the app reconfigures itself. The two directions aren't handled the same way.

```swift
func handlePaidChange(oldValue: Bool, newValue: Bool) {
    switch (oldValue, newValue) {
    case (false, true):
        // User just subscribed
        Task {
            if settingsManager.showOnboarding {
                settingsManager.apiSource = settingsManager.apiSourceDefault(paid: true)
                syncService.sync()
            }
        }

    case (true, false):
        // Subscription expired or refunded
        guard hasUpdatedCurrentEntitlements else {
            // Don't downgrade until we've synced with App Store
            return
        }

        Task {
            settingsManager.apiSource = settingsManager.apiSourceDefault(paid: false)
            savedDataManager.store.removeObject(forKey: SavedDataManager.Keys.radarLayer.rawValue)
            await PushManager.shared.pushEnabledChanged()
            syncService.sync()
        }

    case (false, false), (true, true):
        break  // Logged only
    }
}
```

The side effects run inside a `Task` because `pushEnabledChanged()` is async and the `paid` setter isn't. Notice the guard on `hasUpdatedCurrentEntitlements`. On a cold launch the saved `paid` flag may be stale. Without the guard, the app would strip features before it had asked the App Store. So a downgrade waits for the first sync, and an upgrade never does.

### Transaction Persistence

Transactions are stored as JSON in UserDefaults under an app group, so the widgets read the same list. `JSONDecoder.decoder` and `JSONEncoder.encoder` are shared instances set to the `.iso8601` date format.

```swift
var transactions: [TransactionRecord] {
    get {
        guard let val = savedDataManager.store.data(
            forKey: SavedDataManager.Keys.transactions.rawValue
        ) else { return [] }
        return (try? JSONDecoder.decoder.decode(
            [TransactionRecord].self,
            from: val
        )) ?? []
    }
    set {
        let normalized = normalizedTransactions(newValue)
        guard let val = try? JSONEncoder.encoder.encode(normalized) else { return }
        savedDataManager.store.set(val, forKey: SavedDataManager.Keys.transactions.rawValue)
        transactionsDidChange()
        objectWillChange.send()
    }
}

func process(transaction: Transaction,
             renewalInfo: StoreKit.Product.SubscriptionInfo.RenewalInfo?) {
    let record = TransactionRecord(transaction: transaction, renewalInfo: renewalInfo)
    var updated = transactions.filter { $0.id != record.id }
    updated.append(record)
    transactions = updated
}

func transactionsDidChange() {
    paid = activeTransactions.isNotEmpty
}
```

`process(transaction:)` replaces any record with the same ID, so a transaction that arrives on both streams ends up as one record. Every write recomputes `paid`, so the flag can't drift from the list.

The watch is a separate device, so it can't read this app group. The phone sends just the `paid` flag, not the list, in its WatchConnectivity application context. The watch writes it into its own app-group defaults under the same key. Until the first sync lands, the watch assumes the user has paid rather than flash a paywall at them.

```swift
class WatchStoreManager: ObservableObject {
    private static let sharedStore = UserDefaults(suiteName: SavedDataManager.appGroup)!

    var store: UserDefaults { Self.sharedStore }

    var paid: Bool {
        store.bool(forKey: SavedDataManager.Keys.paid.rawValue)
    }

    var paidWatch: Bool {
        if store.object(forKey: SavedDataManager.Keys.paid.rawValue) == nil {
            return true  // Not synced yet: optimistic default
        } else {
            return paid
        }
    }
}
```

This is simplified. The real `paidWatch` also keeps access for users who migrated from the previous app version.

### Computed Entitlement Properties

We don't store anything about the subscription on its own. Lifetime, expiration date, and auto-renew are all computed by filtering the transaction list.

```swift
var activeTransactions: [TransactionRecord] {
    transactions.filter { $0.active == true }
}

var inActiveTransactions: [TransactionRecord] {
    transactions.filter { $0.active == false }
}

var paidLifetime: Bool {
    activeTransactions.filter { $0.lifetime == true }.isNotEmpty
}

var expirationDate: Date? {
    guard paidLifetime == false else { return nil }
    return activeTransactions.compactMap { $0.expirationDate }.max()
}

var originalPurchaseDate: Date? {
    return activeTransactions.compactMap { $0.originalPurchaseDate }.min()
}

var willAutoRenew: Bool {
    guard paidLifetime == false else { return false }
    return activeTransactions.filter { $0.willAutoRenew == true }.isNotEmpty
}
```

### Detailed Paid Status

Paid or not is enough to gate features. The settings screen needs to say what's actually going on.

```swift
enum PaidStatus: String {
    case lifetime = "Lifetime"
    case subscribed = "Subscribed"   // Active, will auto-renew
    case cancelling = "Cancelling"    // Active, won't renew
    case cancelled = "Cancelled"      // Expired
    case unpaid = "Unpaid"            // Never purchased
}

var paidStatus: PaidStatus {
    if paidLifetime {
        return .lifetime
    } else if willAutoRenew {
        return .subscribed
    } else if paid {
        return .cancelling
    } else if transactions.isNotEmpty {
        return .cancelled
    } else {
        return .unpaid
    }
}
```

`.cancelled` means the list has records but none of them is active. The user paid once and doesn't any more.

---

## TransactionRecord: The Persistence Model

The record copies every field the entitlement decision needs, so making that decision never requires a StoreKit call.

```swift
import Foundation
import StoreKit

struct TransactionRecord: Codable, Identifiable, Equatable {
    let environment: String?
    let id: UInt64?
    let originalID: UInt64?
    let webOrderLineItemID: String?
    let productID: String?
    let productType: String?
    let purchaseDate: Date?
    let originalPurchaseDate: Date?
    let expirationDate: Date?
    let revocationDate: Date?
    let revocationReason: Int?
    let ownershipType: String?
    let willAutoRenew: Bool?
    let currency: String?
    let price: Decimal?
    let gracePeriodExpirationDate: Date?

    init(transaction: Transaction,
         renewalInfo: StoreKit.Product.SubscriptionInfo.RenewalInfo?) {
        self.environment = transaction.environment.rawValue
        self.id = transaction.id
        self.originalID = transaction.originalID
        self.webOrderLineItemID = transaction.webOrderLineItemID
        self.productID = transaction.productID
        self.productType = transaction.productType.rawValue
        self.purchaseDate = transaction.purchaseDate
        self.originalPurchaseDate = transaction.originalPurchaseDate
        self.expirationDate = transaction.expirationDate
        self.revocationDate = transaction.revocationDate
        self.revocationReason = transaction.revocationReason?.rawValue
        self.ownershipType = transaction.ownershipType.rawValue
        self.willAutoRenew = renewalInfo?.willAutoRenew
        self.currency = transaction.currency?.identifier
        self.price = transaction.price
        self.gracePeriodExpirationDate = renewalInfo?.gracePeriodExpirationDate
    }

    var formattedPrice: String? {
        guard let price = price, let currency = currency else { return nil }

        let formatter = NumberFormatter()
        formatter.numberStyle = .currency
        formatter.currencyCode = currency

        return formatter.string(from: price as NSDecimalNumber)
    }
}
```

`gracePeriodExpirationDate` comes from the renewal info, like `willAutoRenew`, and is only set while a renewal is failing.

### Determining Active Status

A transaction is active when all three hold:
- Its product ID is in the list of valid products
- It hasn't been revoked (refunded)
- For a subscription, it hasn't expired, or it's inside a billing grace period

```swift
var active: Bool {
    guard StoreManager.Plan.allActive.contains(productID ?? "") else {
        return false
    }
    guard revocationDate == nil else {
        return false
    }

    if let gracePeriodExpirationDate, Date() < gracePeriodExpirationDate {
        return true
    }

    if let expirationDate = expirationDate {
        return Date() < expirationDate
    }

    return true  // Lifetime purchase with no expiration
}

var inactive: Bool {
    !active
}

var lifetime: Bool {
    active && StoreManager.Plan.allLifetime.contains(productID ?? "")
}
```

A record with no expiration date is a lifetime purchase, so it stays active until it's revoked. A subscription whose renewal is failing stays active until its grace date passes, so the user isn't locked out while Apple retries the charge.

### Revocation Handling

Revocation reasons arrive as integers. We map them to text once, on the record, and the history screen uses that label.

```swift
var revocationReasonString: String {
    switch revocationReason {
    case 0: return "Canceled"
    case 1: return "Billing issue"
    case 2: return "Upgrade/Downgrade"
    case 3: return "Refunded"
    case 4: return "Suspected fraud"
    case 5: return "Pricing expired"
    default: return "Unknown"
    }
}
```

---

## Paywall UI with ProductView

StoreKit 2's `ProductView` loads the product, shows the price, and runs the purchase. A custom `ProductViewStyle` lets us wrap it in the app's own buttons.

### The Main Paywall

```swift
import SwiftUI
import StoreKit

@MainActor
struct PaywallView: View {
    @ObservedObject private var appViewModel = AppViewModel.shared
    @ObservedObject private var storeManager = StoreManager.shared

    @State private var showPlans: Bool = false
    @State private var showRestoreNoPurchases: Bool = false
    @State private var showRestoreSyncFailed: Bool = false

    var body: some View {
        VStack(spacing: 16) {
            // Hero content...

            if storeManager.introEligible == false {
                Text("Subscribe for \(storeManager.yearlyDisplayPrice ?? "...")/year.")
            } else {
                Text("Try 1 week free, then \(storeManager.yearlyDisplayPrice ?? "...")/year after that.")
            }
            Text("Your plan will auto-renew until canceled.")

            ProductView(id: StoreManager.Plan.yearly)
                .productViewStyle(TrialButton(
                    buttonText: storeManager.introEligible == false
                        ? "Subscribe"
                        : "Try 1 week free"
                ))

            HStack {
                Text("See All Plans")
                    .onTapGesture { showPlans = true }

                Text(" • ")

                Button("Restore") {
                    Task {
                        let synced = await StoreService.shared.restorePurchases()

                        if !synced {
                            showRestoreSyncFailed = true
                        } else if storeManager.paid == false {
                            showRestoreNoPurchases = true
                        }
                    }
                }
            }
        }
        .alert("Sorry, but we couldn't find any active purchases for your current App Store account.",
               isPresented: $showRestoreNoPurchases) {
            Button("Done", action: {})
        }
        .alert("Sorry, but we couldn't connect to the App Store. Please check your connection and try again.",
               isPresented: $showRestoreSyncFailed) {
            Button("Done", action: {})
        }
        .onChange(of: storeManager.paid) {
            guard storeManager.paid == true else { return }

            SyncService.shared.sync()

            appViewModel.showPaywallSheet = false
            appViewModel.showPaywallFullScreenCover = false
        }
    }
}
```

The Restore button tells apart two outcomes that look the same to the user: we couldn't reach the App Store, or we reached it and it had nothing. The paywall closes when `storeManager.paid` changes, not on a purchase callback, so a purchase that completes by any route closes it. The trial wording only switches off when `introEligible == false`. A `nil` means we don't know yet, and the trial wording stays.

### Custom ProductViewStyle

While the product loads, the style shows the real button redacted. (In the app it also shimmers, through a third-party modifier.) Once loaded, it wraps the same button around `configuration.purchase()`.

```swift
struct TrialButton: ProductViewStyle {
    var buttonText: LocalizedStringKey = "Try 1 week free"

    func makeBody(configuration: Configuration) -> some View {
        switch configuration.state {
        case .loading:
            PurpleButton(text: buttonText)
                .redacted(reason: .placeholder)

        case .success:
            Button {
                configuration.purchase()
            } label: {
                PurpleButton(text: buttonText)
            }
            .buttonStyle(.plain)

        case .failure(let error):
            let _ = Logger.error(error.localizedDescription)
            PurpleButton(text: "Sorry, an error occurred.", error: true)

        case .unavailable:
            PurpleButton(text: "Sorry, an error occurred.", error: true)

        @unknown default:
            PurpleButton(text: "Sorry, an error occurred.", error: true)
        }
    }
}
```

### Plan Selection View

Three `ProductView`s in a row, each with a selectable style. A family-sharing toggle swaps in the family set and moves the selection to the matching yearly plan. A fourth `ProductView` is the Continue button.

```swift
@MainActor
struct PaywallPlansView: View {
    @State private var selected: String = StoreManager.Plan.yearly
    @State private var showFamilyPlans: Bool = false

    var body: some View {
        VStack {
            HStack(spacing: 10) {
                ForEach(plans, id: \.self) { plan in
                    ProductView(id: plan)
                        .productViewStyle(SelectablePlanStyle(selected: $selected))
                        .tag(plan)
                }
            }

            Toggle("Add Family Sharing", isOn: $showFamilyPlans)
                .onChange(of: showFamilyPlans) {
                    selected = showFamilyPlans
                        ? StoreManager.Plan.yearly_family
                        : StoreManager.Plan.yearly
                }

            ProductView(id: selected)
                .productViewStyle(TrialButton(buttonText: "Continue"))
        }
    }

    private var plans: [String] {
        showFamilyPlans
            ? StoreManager.Plan.paywallFamily
            : StoreManager.Plan.paywallIndividual
    }
}
```

### Selectable Plan Style

Once loaded, the style has the real product, so the card shows the App Store's localized `displayPrice`. A plan that fails to load shows the same error button as the trial button, so the row never quietly loses a card.

```swift
struct SelectablePlanStyle: ProductViewStyle {
    @Binding var selected: String

    func makeBody(configuration: Configuration) -> some View {
        switch configuration.state {
        case .loading:
            PlanCard(selected: $selected, id: "", title: "Loading", price: "Loading", badgeText: nil)

        case .success(let product):
            Button(action: { selected = product.id }) {
                PlanCard(
                    selected: $selected,
                    id: product.id,
                    title: StoreManager.planTitleBase(product.id),
                    price: product.displayPrice,
                    badgeText: badgeText(for: product.id)
                )
            }
            .buttonStyle(.plain)

        case .failure(let error):
            let _ = Logger.error(error.localizedDescription)
            PurpleButton(text: "Sorry, an error occurred.", error: true)

        case .unavailable:
            PurpleButton(text: "Sorry, an error occurred.", error: true)

        @unknown default:
            PurpleButton(text: "Sorry, an error occurred.", error: true)
        }
    }

    private func badgeText(for id: String) -> LocalizedStringKey? {
        guard StoreManager.Plan.isYearly(id) else { return nil }

        return "Best deal"
    }
}
```

---

## Transaction History UI

Every saved transaction, split into active and inactive, with a tap through to a detail view.

```swift
@MainActor
struct TransactionsView: View {
    @ObservedObject private var storeManager = StoreManager.shared

    @State private var selectedTransaction: TransactionRecord?

    var body: some View {
        List {
            Section("Active (\(storeManager.activeTransactions.count))") {
                ForEach(storeManager.activeTransactions.reversed(), id: \.id) { tx in
                    TransactionRowView(transaction: tx)
                        .onTapGesture { selectedTransaction = tx }
                }
            }

            Section("Inactive (\(storeManager.inActiveTransactions.count))") {
                ForEach(storeManager.inActiveTransactions.reversed(), id: \.id) { tx in
                    TransactionRowView(transaction: tx)
                        .onTapGesture { selectedTransaction = tx }
                }
            }
        }
        .navigationTitle("Purchase History")
        .sheet(item: $selectedTransaction) { transaction in
            TransactionDetailView(transaction: transaction)
        }
    }
}
```

### Transaction Detail with Refund Request

The detail view reads only the saved record. Request Refund opens Apple's `refundRequestSheet` for that transaction ID. In the real file, the row labels go through `localized()`.

```swift
struct TransactionDetailView: View {
    let transaction: TransactionRecord

    @Environment(\.dismiss) private var dismiss

    @State private var refundRequestSheetIsPresented = false

    var body: some View {
        NavigationView {
            List {
                Section("Status") {
                    DetailRow(title: "Status",
                             value: transaction.active ? "Active" : "Inactive")
                }

                Section("Transaction Details") {
                    DetailRow(title: "Product",
                             value: StoreManager.planTitle(transaction.productID))
                    DetailRow(title: "Type", value: transaction.productType)
                    DetailRow(title: "ID", value: transaction.id?.description)

                    if transaction.id != transaction.originalID {
                        DetailRow(title: "Original ID",
                                 value: transaction.originalID?.description)
                    }

                    DetailRow(title: "Web ID", value: transaction.webOrderLineItemID)
                    DetailRow(title: "Environment", value: transaction.environment)
                    DetailRow(title: "Price", value: transaction.formattedPrice)
                    DetailRow(title: "Ownership Type", value: transaction.ownershipType)
                    DetailRow(title: "Will Auto Renew",
                             value: transaction.willAutoRenew?.description)
                }

                Section("Dates") {
                    DetailRow(title: "Purchase Date",
                             value: transaction.purchaseDate?.formatted())

                    if transaction.purchaseDate != transaction.originalPurchaseDate {
                        DetailRow(title: "Original Purchase",
                                 value: transaction.originalPurchaseDate?.formatted())
                    }

                    if let expirationDate = transaction.expirationDate {
                        DetailRow(title: "Expiration Date", value: expirationDate.formatted())
                    }

                    if let revocationDate = transaction.revocationDate {
                        DetailRow(title: "Revocation Date", value: revocationDate.formatted())
                        DetailRow(title: "Revocation Reason",
                                 value: transaction.revocationReasonString)
                    }
                }

                Section("Manage") {
                    Button("Request Refund") {
                        refundRequestSheetIsPresented = true
                    }
                }
            }
            .navigationTitle("Details")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button("Done") { dismiss() }
                }
            }
            .refundRequestSheet(
                for: transaction.id ?? 0,
                isPresented: $refundRequestSheetIsPresented
            )
        }
    }
}

private struct DetailRow: View {
    let title: String
    let value: String?

    var body: some View {
        HStack {
            Text(title)
            Spacer()
            Text(value ?? "-")
                .foregroundColor(.secondary)
        }
    }
}
```

---

## Initialization

StoreService is activated once, from a `.task` on the root view, after the data migrations have run. The app shows a loading view until `AppMonitor` sets `ready`. This is simplified: onboarding and the other services are left out.

```swift
@main
@MainActor
struct HelloWeatherApp: App {
    @StateObject private var appMonitor = AppMonitor.shared

    var body: some Scene {
        WindowGroup {
            Group {
                if appMonitor.ready {
                    AppContainerView()
                } else {
                    AppLoadingView()
                }
            }
            .task {
                await appMonitor.activate()
            }
        }
    }
}

@MainActor
class AppMonitor: ObservableObject {
    static let shared = AppMonitor()
    private init() {}

    @Published var ready = false

    func activate() async {
        await MigrationManager.shared.migrate()
        ready = true

        // ...other services
        StoreService.shared.activate()
    }
}
```

---

## Lessons Learned

- **A saved paid flag is a cache.** Don't take away access on its word alone. Wait for the first sync with the store before downgrading.
- **Save the record, not the verdict.** Keep every field the entitlement decision needs, so other processes and a history screen can answer without asking the store.
- **Listen to both update streams.** `Transaction.updates` catches purchases, and `SubscriptionInfo.Status.updates` catches renewal-state changes.
- **Log an unverified transaction and move on.** A jailbroken device or corrupted data shouldn't take the app down.
- **Keep the entitlement decision testable without StoreKit.** `TransactionRecord.active` has unit tests that run on JSON fixtures. A StoreKit Configuration file covers purchase flows in the simulator, and a sandbox account covers the rest.
