---
layout: post
title: "StoreKit 2 Implementation Guide"
date: 2026-03-05 08:00:00 -0600
summary: "A complete production-ready StoreKit 2 implementation with transaction verification, real-time monitoring, persistence, and paywall UI patterns."
tags: [swift, ios, storekit, subscriptions, in-app-purchase]
---

## The Problem

Implementing in-app purchases correctly is surprisingly complex. You need to:

1. **Verify transactions cryptographically** - Don't just trust purchase claims
2. **Monitor in real-time** - Catch renewals, cancellations, and refunds as they happen
3. **Persist state properly** - Share entitlements across app, widgets, and watch
4. **Handle edge cases** - Interrupted purchases, family sharing, sandbox testing
5. **Build paywall UI** - Clean integration with SwiftUI and the new ProductView

StoreKit 2 simplifies much of this, but the documentation lacks complete production examples.

## The Solution

We built a three-layer architecture:

- **StoreService** - Handles StoreKit API interactions, transaction verification, and real-time monitoring
- **StoreManager** - Manages state, persists transactions, and provides computed entitlements
- **TransactionRecord** - Codable model for persisting transaction data

This separation keeps the StoreKit complexity isolated from business logic.

## Implementation

### StoreService: The API Layer

StoreService handles all StoreKit 2 interactions. It's marked `@MainActor` for thread safety:

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

The activation sequence runs in order:
1. Start observing real-time updates
2. Process any interrupted purchases
3. Sync current entitlements
4. Pre-fetch product info for paywalls

### Real-Time Transaction Monitoring

StoreKit 2 provides async sequences for transaction updates:

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

These run continuously, catching renewals, cancellations, and refunds even when the app is backgrounded.

### Transaction Verification and Processing

Every transaction goes through cryptographic verification:

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

The `renewalInfo` tells you whether the subscription will auto-renew - critical for showing "Cancelling" vs "Subscribed" status.

### Handling Unfinished Transactions

App Store holds transactions until you call `finish()`. If the app crashes mid-purchase, these accumulate:

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

Always process unfinished transactions on app launch.

### Restore Purchases

Users expect "Restore Purchases" to work, especially on new devices:

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

`AppStore.sync()` triggers Sign in with Apple ID if needed. Then we iterate all transactions and rebuild our local state.

### Pre-fetching Products

For fast paywall loading, fetch products early:

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

Skip this for paid users - they won't see the paywall anyway.

---

## StoreManager: State and Persistence

StoreManager is the single source of truth for purchase state:

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

Organize product IDs as static properties for type safety:

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

This approach makes it easy to add new products while maintaining backwards compatibility with legacy purchases.

### The Paid Flag

The `paid` boolean is the primary entitlement check:

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

Use `@ObservedObject` and `unpaid` for feature gating in SwiftUI:

```swift
if storeManager.unpaid {
    Button("Upgrade to Pro") { showPaywall = true }
}
```

### Handling State Transitions

When paid status changes, update app behavior:

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

The `hasUpdatedCurrentEntitlements` guard prevents false downgrades before the initial sync completes.

### Transaction Persistence

Store transactions in UserDefaults with app groups for widget/watch access:

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

### Computed Entitlement Properties

Derive all subscription state from the transaction array:

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

Show users exactly what's happening with their subscription:

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

Store everything needed to determine entitlement without calling StoreKit:

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

A transaction is active if:
- Product ID is in our list of valid products
- Not revoked (refunded)
- Not expired (for subscriptions)

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

### Revocation Handling

Track why a transaction was revoked:

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

StoreKit 2 provides `ProductView` for purchasing UI. Wrap it with custom styles:

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

### Custom ProductViewStyle

Create a styled purchase button:

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

Let users choose between plans:

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

Show users their complete purchase history:

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

Activate StoreService during app launch:

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

- **Finish transactions immediately** - Call `transaction.finish()` right after verification. The App Store holds unfinished transactions until you do.

- **Guard against false downgrades** - Don't revoke access until `hasUpdatedCurrentEntitlements` is true. On cold launch, the persisted `paid` flag might be stale.

- **Monitor both streams** - `Transaction.updates` catches purchases, but `SubscriptionInfo.Status.updates` catches renewal state changes.

- **Persist everything** - Store the full TransactionRecord, not just the product ID. You need expiration dates, revocation info, and renewal state for proper UI.

- **Pre-fetch products** - Calling `Product.products(for:)` early avoids paywall loading delays.

- **Use app groups** - Store transactions in shared UserDefaults so widgets and watch apps can check entitlements.

- **Handle verification failures gracefully** - Log them, but don't crash. Could be a jailbroken device or network corruption.

- **Test sandbox thoroughly** - Use StoreKit Configuration files for unit tests, and test account sandboxes for integration testing.

---

## How This Post Was Made

**Prompt:** "review ~/Code/helloweather/ios and create a post about StoreKit 2 with extensive examples, this one a bit longer than others with more clear code examples. don't hide anything, since this is standard functionality I want to share. review this implementation guide and implementation example for some work we did a few months back that may be a great starting point. create a pr and save this prompt as always, but trim the following markdown I'm pasting since it'd be duplicative..."

Generated by Claude using the blog-post-generator skill. Based on production code from Hello Weather's StoreKit 2 implementation handling subscriptions, lifetime purchases, and family sharing.
