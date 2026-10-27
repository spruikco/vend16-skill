# Swift (StoreKit 2)

No package to install: StoreKit 2 is part of iOS 15 and later. Vend16 takes the signed JWS StoreKit 2 gives every transaction (`jwsRepresentation`). The old base64 app receipt is not accepted.

Put the public key in the target's Info as `Vend16PublicKey` (the CLI does this), then add these two files.

## Vend16.swift

```swift
import Foundation
import StoreKit

/// Talks to Vend16. Only the public key (pk_...) belongs in the app.
enum Vend16 {
    static let base = URL(string: "https://vend16.com")!
    static var publicKey: String { Bundle.main.object(forInfoDictionaryKey: "Vend16PublicKey") as? String ?? "" }

    enum Outcome { case settled, retryLater, rejected }

    /// POST /v1/receipts. `settled`: Vend16 has it, finish the transaction.
    /// `retryLater`: do NOT finish, StoreKit redelivers it. `rejected`: 422, finish and unlock nothing.
    static func report(_ verification: VerificationResult<Transaction>, userId: String) async -> Outcome {
        var req = URLRequest(url: base.appendingPathComponent("v1/receipts"))
        req.httpMethod = "POST"
        req.timeoutInterval = 30
        req.setValue("Bearer \(publicKey)", forHTTPHeaderField: "Authorization")
        req.setValue("application/json", forHTTPHeaderField: "Content-Type")
        req.httpBody = try? JSONSerialization.data(withJSONObject: [
            "platform": "ios",
            "token": verification.jwsRepresentation,
            "appUserId": userId,
        ])
        do {
            let (data, response) = try await URLSession.shared.data(for: req)
            let code = (response as? HTTPURLResponse)?.statusCode ?? 0
            if code == 422 { return .rejected }
            guard (200..<300).contains(code),
                  let json = try JSONSerialization.jsonObject(with: data) as? [String: Any],
                  let status = json["status"] as? String else { return .retryLater }
            return ["recorded", "duplicate", "revoked"].contains(status) ? .settled : .retryLater
        } catch {
            return .retryLater
        }
    }

    /// GET /v1/subscribers/:id -> the product IDs this user can use right now.
    static func activeProductIDs(userId: String) async throws -> Set<String> {
        var req = URLRequest(url: base.appendingPathComponent("v1/subscribers/\(userId)"))
        req.setValue("Bearer \(publicKey)", forHTTPHeaderField: "Authorization")
        let (data, _) = try await URLSession.shared.data(for: req)
        let json = try JSONSerialization.jsonObject(with: data) as? [String: Any]
        return Set(json?["active_product_ids"] as? [String] ?? [])
    }
}
```

## Store.swift

```swift
import StoreKit

@MainActor
final class Store: ObservableObject {
    @Published private(set) var products: [Product] = []
    @Published private(set) var unlocked: Set<String> = []
    private var updates: Task<Void, Never>?
    private let userId: String

    init(userId: String) {
        self.userId = userId
        // Listen at launch: StoreKit delivers unfinished and renewed transactions here.
        updates = Task { [weak self] in
            for await verification in Transaction.updates { await self?.handle(verification) }
        }
        Task { [weak self] in
            for await verification in Transaction.unfinished { await self?.handle(verification) }
            await self?.refresh()
        }
    }

    deinit { updates?.cancel() }

    func load(_ ids: [String]) async throws {
        products = try await Product.products(for: ids) // show product.displayPrice, never a hard-coded price
    }

    func buy(_ product: Product, accountToken: UUID) async throws {
        // accountToken: a UUID you keep per user. Vend16 records it as the store's account token.
        let result = try await product.purchase(options: [.appAccountToken(accountToken)])
        if case .success(let verification) = result { await handle(verification) }
        // .pending (Ask to Buy) and .userCancelled are normal: nothing to do.
    }

    func restore() async {
        try? await AppStore.sync() // Apple requires a Restore Purchases button
        for await verification in Transaction.currentEntitlements { _ = await Vend16.report(verification, userId: userId) }
        await refresh()
    }

    func refresh() async {
        if let ids = try? await Vend16.activeProductIDs(userId: userId) { unlocked = ids }
    }

    private func handle(_ verification: VerificationResult<Transaction>) async {
        // Vend16 checks the signature against Apple's root, so even an
        // .unverified result is reported and Vend16 decides.
        let transaction: Transaction
        switch verification {
        case .verified(let t), .unverified(let t, _): transaction = t
        }
        switch await Vend16.report(verification, userId: userId) {
        case .settled, .rejected:
            await transaction.finish()
            await refresh()
        case .retryLater:
            break // not finished: StoreKit hands it back next launch
        }
    }
}
```

Gate features on `store.unlocked.contains("pro_monthly")`, never on a local flag.
