---
name: swift-concurrency
description: Swift 6.2 concurrency patterns including async/await, Actor, Sendable, structured concurrency, and data race prevention.
---

# Swift 6.2 Concurrency

Modern concurrency patterns for Swift 6 with strict concurrency checking.

## Core Concepts

### async/await Basics

```swift
// ✅ GOOD: Clear async function
func fetchUser(id: String) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(id)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

// Calling async functions
Task {
    do {
        let user = try await fetchUser(id: "123")
        print(user.name)
    } catch {
        print("Failed: \(error)")
    }
}
```

### Task and TaskGroup

```swift
// Single Task
let task = Task {
    try await fetchUser(id: "123")
}

// Later...
let user = try await task.value

// Detached Task (inherits no context)
Task.detached {
    await heavyComputation()
}

// TaskGroup for parallel work
func fetchAllUsers(ids: [String]) async throws -> [User] {
    try await withThrowingTaskGroup(of: User.self) { group in
        for id in ids {
            group.addTask {
                try await fetchUser(id: id)
            }
        }

        var users: [User] = []
        for try await user in group {
            users.append(user)
        }
        return users
    }
}
```

### Structured Concurrency

```swift
// ✅ GOOD: Structured - child tasks bounded by scope
func processOrder(_ order: Order) async throws -> Receipt {
    // Both tasks complete before function returns
    async let validation = validateOrder(order)
    async let inventory = checkInventory(order.items)

    let (isValid, hasStock) = try await (validation, inventory)

    guard isValid && hasStock else {
        throw OrderError.cannotProcess
    }

    return try await submitOrder(order)
}

// ❌ BAD: Unstructured - task escapes scope
func processOrder(_ order: Order) {
    Task {
        // This task lives beyond the function
        // Hard to track, cancel, or handle errors
        try? await submitOrder(order)
    }
}
```

## Actor

### Actor Definition and Isolation

```swift
// ✅ GOOD: Actor protects mutable state
actor BankAccount {
    private var balance: Decimal = 0

    func deposit(_ amount: Decimal) {
        balance += amount
    }

    func withdraw(_ amount: Decimal) throws {
        guard balance >= amount else {
            throw BankError.insufficientFunds
        }
        balance -= amount
    }

    func getBalance() -> Decimal {
        balance
    }
}

// Usage - all access is async
let account = BankAccount()
await account.deposit(100)
let balance = await account.getBalance()
```

### @MainActor

```swift
// ✅ GOOD: UI updates on main thread
@MainActor
final class ViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    @Published var error: Error?

    func loadUsers() async {
        isLoading = true
        defer { isLoading = false }

        do {
            users = try await api.fetchUsers()
        } catch {
            self.error = error
        }
    }
}

// ✅ GOOD: Single method on MainActor
func updateUI() async {
    let data = await fetchData()  // Background

    await MainActor.run {
        // UI update on main thread
        self.label.text = data.title
    }
}
```

### nonisolated Keyword

```swift
actor DataStore {
    private var cache: [String: Data] = [:]

    // Computed property accessing actor state - needs isolation
    var count: Int {
        cache.count
    }

    // ✅ nonisolated: No actor state access
    nonisolated let identifier = UUID()

    // ✅ nonisolated func: Pure computation, no state
    nonisolated func validate(key: String) -> Bool {
        !key.isEmpty && key.count < 100
    }

    // ✅ nonisolated for Hashable conformance
    nonisolated func hash(into hasher: inout Hasher) {
        hasher.combine(identifier)
    }
}
```

### GlobalActor Pattern

```swift
// Define custom global actor
@globalActor
actor DatabaseActor {
    static let shared = DatabaseActor()
}

// Use for database operations
@DatabaseActor
final class DatabaseManager {
    private var connection: Connection?

    func query(_ sql: String) async throws -> [Row] {
        // All calls automatically serialized
    }
}

// Mark individual functions
@DatabaseActor
func performDatabaseWork() async {
    // Runs on DatabaseActor
}
```

## Sendable

### Sendable Protocol

```swift
// ✅ GOOD: Immutable struct is Sendable
struct User: Sendable {
    let id: String
    let name: String
}

// ✅ GOOD: Enum with Sendable associated values
enum Result<T: Sendable>: Sendable {
    case success(T)
    case failure(Error)
}

// ✅ GOOD: Final class with let properties
final class Configuration: Sendable {
    let apiKey: String
    let baseURL: URL

    init(apiKey: String, baseURL: URL) {
        self.apiKey = apiKey
        self.baseURL = baseURL
    }
}

// ❌ BAD: Mutable class is not Sendable
class Counter {  // Not Sendable - has mutable state
    var count = 0
}
```

### @Sendable Closures

```swift
// ✅ GOOD: @Sendable closure
func performAsync(work: @Sendable @escaping () async -> Void) {
    Task {
        await work()
    }
}

// Usage
performAsync { // @Sendable closure
    // Can only capture Sendable values
    await processData()
}

// ❌ BAD: Capturing non-Sendable in @Sendable closure
class NonSendable {
    var value = 0
}

let obj = NonSendable()
performAsync {
    obj.value += 1  // ❌ Compile error in Swift 6
}
```

### @unchecked Sendable (Use with Caution)

```swift
// ⚠️ CAUTION: Only when you guarantee thread safety
final class ThreadSafeCache: @unchecked Sendable {
    private let lock = NSLock()
    private var storage: [String: Any] = [:]

    func get(_ key: String) -> Any? {
        lock.lock()
        defer { lock.unlock() }
        return storage[key]
    }

    func set(_ key: String, value: Any) {
        lock.lock()
        defer { lock.unlock() }
        storage[key] = value
    }
}

// ⚠️ WARNING: @unchecked Sendable bypasses compiler checks
// - Only use when manual synchronization is in place
// - Document why it's safe
// - Prefer actors when possible
```

## Swift 6 Strict Concurrency

### Complete Checking Mode

```swift
// In Package.swift or Build Settings
// Enable: SWIFT_STRICT_CONCURRENCY = complete

// All data races become compile-time errors
```

### Data Race Prevention

```swift
// ❌ BAD: Data race - shared mutable state
class UnsafeCounter {
    var count = 0  // Accessed from multiple threads

    func increment() {
        count += 1  // Race condition!
    }
}

// ✅ GOOD: Actor eliminates data race
actor SafeCounter {
    var count = 0

    func increment() {
        count += 1  // Actor isolation guarantees safety
    }
}

// ✅ GOOD: Sendable value types
struct ImmutableData: Sendable {
    let values: [Int]  // Immutable, safe to share
}
```

### Migration Guide

```swift
// Step 1: Enable warnings first
// SWIFT_STRICT_CONCURRENCY = targeted

// Step 2: Fix warnings one by one

// Common fixes:

// 1. Make types Sendable
struct MyData: Sendable { }

// 2. Add @MainActor to UI code
@MainActor
class ViewModel { }

// 3. Use actors for shared mutable state
actor SharedState { }

// 4. Mark closures as @Sendable
func async(work: @Sendable () -> Void) { }

// Step 3: Enable complete checking
// SWIFT_STRICT_CONCURRENCY = complete
```

## Practical Patterns

### AsyncStream / AsyncThrowingStream

```swift
// ✅ GOOD: Bridging callback-based API to async
func notifications(for name: Notification.Name) -> AsyncStream<Notification> {
    AsyncStream { continuation in
        let observer = NotificationCenter.default.addObserver(
            forName: name,
            object: nil,
            queue: nil
        ) { notification in
            continuation.yield(notification)
        }

        continuation.onTermination = { _ in
            NotificationCenter.default.removeObserver(observer)
        }
    }
}

// Usage
for await notification in notifications(for: .userDidLogin) {
    handleLogin(notification)
}

// ✅ AsyncThrowingStream for error cases
func downloadProgress(url: URL) -> AsyncThrowingStream<Double, Error> {
    AsyncThrowingStream { continuation in
        let task = URLSession.shared.downloadTask(with: url) { _, _, error in
            if let error {
                continuation.finish(throwing: error)
            } else {
                continuation.finish()
            }
        }

        // Observe progress...
        task.resume()
    }
}
```

### Task Cancellation

```swift
// ✅ GOOD: Check for cancellation
func processItems(_ items: [Item]) async throws -> [Result] {
    var results: [Result] = []

    for item in items {
        // Check if task was cancelled
        try Task.checkCancellation()

        // Or check without throwing
        if Task.isCancelled {
            break
        }

        let result = try await process(item)
        results.append(result)
    }

    return results
}

// ✅ GOOD: withTaskCancellationHandler
func fetchWithCancellation() async throws -> Data {
    let session = URLSession.shared
    var dataTask: URLSessionDataTask?

    return try await withTaskCancellationHandler {
        try await withCheckedThrowingContinuation { continuation in
            dataTask = session.dataTask(with: url) { data, _, error in
                if let error {
                    continuation.resume(throwing: error)
                } else if let data {
                    continuation.resume(returning: data)
                }
            }
            dataTask?.resume()
        }
    } onCancel: {
        dataTask?.cancel()
    }
}
```

### withCheckedContinuation (Legacy Bridging)

```swift
// ✅ GOOD: Bridge callback-based API
func fetchUser(id: String) async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        legacyAPI.fetchUser(id: id) { result in
            switch result {
            case .success(let user):
                continuation.resume(returning: user)
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}

// ⚠️ WARNING: Must resume exactly once
// - Resuming twice = crash
// - Never resuming = task hangs forever
```

### Combine to async/await

```swift
// ✅ Bridge Combine publisher to async
extension Publisher where Failure == Never {
    func firstValue() async -> Output? {
        await withCheckedContinuation { continuation in
            var cancellable: AnyCancellable?
            cancellable = self.first().sink { value in
                continuation.resume(returning: value)
                cancellable?.cancel()
            }
        }
    }
}

// ✅ Bridge async to Combine
extension Publisher {
    func asyncMap<T>(
        _ transform: @escaping (Output) async -> T
    ) -> Publishers.FlatMap<Future<T, Never>, Self> {
        flatMap { value in
            Future { promise in
                Task {
                    let result = await transform(value)
                    promise(.success(result))
                }
            }
        }
    }
}
```

## Best Practices Checklist

- [ ] Use structured concurrency (async let, TaskGroup) over unstructured Task { }
- [ ] Mark all UI-related code with @MainActor
- [ ] Make value types Sendable when shared across tasks
- [ ] Use actors for shared mutable state
- [ ] Always handle Task cancellation
- [ ] Enable strict concurrency checking (complete mode)
- [ ] Avoid @unchecked Sendable unless absolutely necessary
- [ ] Resume continuations exactly once
