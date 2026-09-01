---
layout: post
title: "StoreKit 2 Implementation Guide"
date: 2026-03-05 08:00:00 -0600
summary: "A production StoreKit 2 implementation with nothing hidden: verified transactions, two update streams, a persisted record that widgets and the watch app can read, a guard against downgrading on a stale flag, and paywall UI built on ProductView."
tags: [swift, ios, storekit, subscriptions, in-app-purchase]
---

## The Problem

Three processes need to know whether the user has paid: the app, its widgets, and the watch app. The app talks to the App Store and writes what it learns to shared storage. The other two read from there, so the stored answer has to be right, including on a cold launch before the App Store has been asked.

StoreKit 2 verifies every transaction cryptographically and hands the rest back to the app:

1. **Monitor in real time** so renewals, cancellations, and refunds land as they happen
2. **Persist state** in a form every process can read without calling StoreKit
3. **Handle the edge cases**: interrupted purchases, family sharing, sandbox testing
4. **Build the paywall** on SwiftUI and the new `ProductView`

The documentation covers each API but not a complete production example. This post is one, with nothing hidden.

## The Solution

Three layers, so StoreKit's surface stays out of the business logic:

- **StoreService** calls StoreKit, verifies transactions, and runs the real-time observers
- **StoreManager** holds state, persists transactions, and derives entitlements from them
- **TransactionRecord** is the `Codable` model that gets persisted

## Implementation

### StoreService: The API Layer

StoreService owns every call into StoreKit and is `@MainActor`, so nothing it hands to StoreManager crosses a thread boundary.

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

The awaits in `activate()` run in order, and the next four sections follow it.

### Real-Time Transaction Monitoring

StoreKit 2 exposes changes as async sequences. Two are needed, each in its own long-lived task.

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

Both loops run for the life of the app and catch renewals, cancellations, and refunds even when the app is backgrounded.

### Transaction Verification and Processing

Every transaction, whichever stream delivered it, goes through one switch on its verification result.

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

An `.unverified` result is logged and dropped, not crashed on. The renewal info is fetched alongside because `willAutoRenew` is the only way to tell "Subscribed" from "Cancelling" later.

### Handling Unfinished Transactions

The App Store holds every transaction until the app calls `finish()`. A crash mid-purchase leaves one behind, and they accumulate, so draining them is part of every launch.

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

`updateCurrentEntitlements` also flips `hasUpdatedCurrentEntitlements`, the flag that later lets StoreManager trust a downgrade.

### Restore Purchases

Restore has to work on a new device with empty local storage. `AppStore.sync()` may prompt for Apple ID sign-in and may fail, so the function returns a `Bool`.

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

The local list is replaced, not merged. The paywall uses the `Bool` to tell a failed sync from one that found nothing.

### Pre-fetching Products

Fetching at activation means prices are in memory before the paywall appears.

```swift
func fetchProducts() async {
    guard storeManager.unpaid else { return }

    do {
        let products = try await StoreKit.Product.products(
            for: StoreManager.Plan.allPaywall
        )
        storeManager.products = products
    } catch {
        Logger.error("fetchProducts: \(error.localizedDescription)")
    }
}
```

Paid users skip it. They never see the paywall.

---

## StoreManager: State and Persistence

StoreManager is the single source of truth for purchase state, and everything it exposes derives from the persisted transaction list.

```swift
import Foundation
import StoreKit

@MainActor
class StoreManager: ObservableObject {
    static let shared = StoreManager()

    private lazy var savedDataManager = SavedDataManager.shared

    var products: [StoreKit.Product] = [] {
        didSet { objectWillChange.send() }
    }

    private let maxTransactions = 9999
}
```

### Defining Product IDs

Twelve product IDs, six current and six legacy, live as static properties on one enum.

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
}
```

The legacy IDs are in `allActive` but in neither paywall list, so an old purchase still unlocks the app and the plan cannot be bought again.

### The Paid Flag

`paid` is the one boolean the rest of the app checks. Its setter runs the transition handler before writing.

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
```

Feature gates in SwiftUI read `unpaid` through an `@ObservedObject`:

```swift
if storeManager.unpaid {
    Button("Upgrade to Pro") { showPaywall = true }
}
```

### Handling State Transitions

A change in `paid` is where the app reconfigures itself. The two directions are not symmetric.

```swift
func handlePaidChange(oldValue: Bool, newValue: Bool) {
    switch (oldValue, newValue) {
    case (false, true):
        // User just subscribed
        if settingsManager.showOnboarding {
            settingsManager.apiSource = settingsManager.apiSourceDefault(paid: true)
            syncService.sync()
        }

    case (true, false):
        // Subscription expired or refunded
        guard hasUpdatedCurrentEntitlements else {
            // Don't downgrade until we've synced with App Store
            return
        }

        settingsManager.apiSource = settingsManager.apiSourceDefault(paid: false)
        savedDataManager.store.removeObject(forKey: SavedDataManager.Keys.radarLayer.rawValue)
        await PushManager.shared.pushEnabledChanged()
        syncService.sync()

    default:
        break
    }
}
```

Notice the guard on `hasUpdatedCurrentEntitlements`. On a cold launch the persisted `paid` flag may be stale, and without it the app would strip features before the App Store had been consulted. Downgrades wait for the initial sync; upgrades never do.

### Transaction Persistence

Transactions are JSON in UserDefaults under an app group, so widgets and the watch app read the same list.

```swift
var transactions: [TransactionRecord] {
    get {
        guard let val = savedDataManager.store.data(
            forKey: SavedDataManager.Keys.transactions.rawValue
        ) else { return [] }
        return (try? JSONDecoder().decode(
            [TransactionRecord].self,
            from: val
        )) ?? []
    }
    set {
        let normalized = normalizedTransactions(newValue)
        guard let val = try? JSONEncoder().encode(normalized) else { return }
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

`process(transaction:)` replaces by ID, so a transaction delivered by both streams produces one record. Every write recomputes `paid`, so the flag cannot drift from the list.

### Computed Entitlement Properties

Nothing about the subscription is stored on its own. Lifetime, expiration, and auto-renew are filters over the transaction array.

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

Paid or not is enough for feature gating. The settings screen needs to say what is actually happening.

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
    } else if hasPaid {
        return .cancelled
    } else {
        return .unpaid
    }
}
```

---

## TransactionRecord: The Persistence Model

The record copies every field the entitlement decision needs, so that decision never requires a StoreKit call.

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
    }
}
```

### Determining Active Status

A transaction is active when all three hold:
- Its product ID is in the list of valid products
- It has not been revoked (refunded)
- It has not expired, for subscriptions

```swift
var active: Bool {
    guard StoreManager.Plan.allActive.contains(productID ?? "") else {
        return false
    }
    guard revocationDate == nil else {
        return false
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

A record with no expiration date is a lifetime purchase and stays active until revoked.

### Revocation Handling

Revocation reasons arrive as integers. Mapping them once, on the record, gives the history UI its label.

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

StoreKit 2's `ProductView` loads the product, shows the price, and runs the purchase. A `ProductViewStyle` lets the app's own buttons wrap it.

### The Main Paywall

```swift
import SwiftUI
import StoreKit

@MainActor
struct PaywallView: View {
    @ObservedObject private var storeManager = StoreManager.shared

    @State private var showPlans: Bool = false
    @State private var showRestoreNoPurchases: Bool = false
    @State private var showRestoreSyncFailed: Bool = false

    var body: some View {
        VStack(spacing: 16) {
            // Hero content...

            Text("Try 1 week free, then \(storeManager.yearlyDisplayPrice ?? "...")/year")

            ProductView(id: StoreManager.Plan.yearly)
                .productViewStyle(TrialButton(buttonText: "Try 1 week free"))

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
        .alert("No active purchases found.", isPresented: $showRestoreNoPurchases) {
            Button("Done", action: {})
        }
        .alert("Couldn't connect to App Store.", isPresented: $showRestoreSyncFailed) {
            Button("Done", action: {})
        }
        .onChange(of: storeManager.paid) {
            guard storeManager.paid == true else { return }
            // Dismiss paywall on successful purchase
        }
    }
}
```

The Restore button separates two outcomes that look identical to the user: the App Store could not be reached, and it was reached and had nothing. Dismissal keys off `storeManager.paid` changing, not a purchase callback, so a purchase that completes by any route closes the paywall.

### Custom ProductViewStyle

While loading, the style shows the real button redacted. On success it wraps the same button around `configuration.purchase()`.

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

Three `ProductView`s in a row with a selectable style, a family-sharing toggle that swaps the set and moves the selection to the matching yearly plan, and a fourth `ProductView` as the Continue button.

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

On success the style holds the real product, so the card shows the App Store's localized `displayPrice`. A plan that fails to load disappears rather than showing an error.

```swift
struct SelectablePlanStyle: ProductViewStyle {
    @Binding var selected: String

    func makeBody(configuration: Configuration) -> some View {
        switch configuration.state {
        case .loading:
            PlanCard(selected: $selected, id: "", title: "Loading", price: "...")

        case .success(let product):
            Button(action: { selected = product.id }) {
                PlanCard(
                    selected: $selected,
                    id: product.id,
                    title: StoreManager.planTitle(product.id),
                    price: product.displayPrice,
                    badgeText: StoreManager.Plan.isYearly(product.id)
                        ? "Best deal"
                        : nil
                )
            }
            .buttonStyle(.plain)

        case .failure, .unavailable:
            EmptyView()

        @unknown default:
            EmptyView()
        }
    }
}
```

---

## Transaction History UI

Every recorded transaction, split into active and inactive, with a tap through to detail.

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

The detail view reads only the persisted record. Request Refund opens Apple's `refundRequestSheet` for that transaction ID.

```swift
struct TransactionDetailView: View {
    let transaction: TransactionRecord

    @State private var refundRequestSheetIsPresented = false

    var body: some View {
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
                DetailRow(title: "Environment", value: transaction.environment)
                DetailRow(title: "Price", value: transaction.formattedPrice)
                DetailRow(title: "Will Auto Renew",
                         value: transaction.willAutoRenew?.description)
            }

            Section("Dates") {
                DetailRow(title: "Purchase Date",
                         value: transaction.purchaseDate?.formatted())
                if let expiration = transaction.expirationDate {
                    DetailRow(title: "Expiration", value: expiration.formatted())
                }
                if let revocation = transaction.revocationDate {
                    DetailRow(title: "Revoked", value: revocation.formatted())
                    DetailRow(title: "Reason",
                             value: transaction.revocationReasonString)
                }
            }

            Section("Manage") {
                Button("Request Refund") {
                    refundRequestSheetIsPresented = true
                }
            }
        }
        .refundRequestSheet(
            for: transaction.id ?? 0,
            isPresented: $refundRequestSheetIsPresented
        )
    }
}
```

---

## Initialization

StoreService is activated once, from the app's `init`.

```swift
@main
struct HelloWeatherApp: App {
    init() {
        AppMonitor.activate()  // Calls StoreService.shared.activate()
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

---

## Lessons Learned

- **A persisted entitlement flag is a cache.** Never revoke access on its say-so. Wait for the first sync with the store before downgrading.
- **Persist the record, not the verdict.** Keep every field the entitlement decision needs, so other processes and a history screen answer without the store.
- **Subscribe to both update streams.** `Transaction.updates` catches purchases; `SubscriptionInfo.Status.updates` catches renewal-state changes.
- **An unverified transaction is noise, not a crash.** Log it and move on. A jailbroken device or corrupted data should not take the app down.
- **Test against a StoreKit Configuration file, then a sandbox account.** Unit tests run on the file; integration tests need the sandbox.

---

## How This Post Was Made

**Prompt:** "review ~/Code/helloweather/ios and create a post about StoreKit 2 with extensive examples, this one a bit longer than others with more clear code examples. don't hide anything, since this is standard functionality I want to share. review this implementation guide and implementation example for some work we did a few months back that may be a great starting point. create a pr and save this prompt as always, but trim the following markdown I'm pasting since it'd be duplicative..."

Generated by Claude using the blog-post-generator skill. Based on production code from Hello Weather's StoreKit 2 implementation handling subscriptions, lifetime purchases, and family sharing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the three processes that need the entitlement answer instead of on how complex purchases are, the prose after each code block says what to notice rather than re-walking the code, and Lessons Learned is down from eight bullets to five that the body does not already state. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
