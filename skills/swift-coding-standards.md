---
name: swift-coding-standards
description: Swift coding standards, API Design Guidelines, naming conventions, type safety, error handling, and fundamental principles for Swift development.
---

# Swift Coding Standards

Core coding standards and best practices for Swift development.

## Swift API Design Guidelines

### Clarity Over Brevity

```swift
// ✅ GOOD: Clear names
func insert(_ element: Element, at index: Int)
func removeElement(at index: Int) -> Element

// ❌ BAD: Ambiguous names
func insert(_ element: Element, position: Int)
func remove(_ index: Int) -> Element
```

### Naming Conventions

#### Type Names (UpperCamelCase)

```swift
// ✅ GOOD
struct UserProfile { }
class NetworkManager { }
enum ConnectionState { }
protocol DataProviding { }

// ❌ BAD
struct userProfile { }
class network_manager { }
```

#### Properties, Methods, Parameters (lowerCamelCase)

```swift
// ✅ GOOD
var userName: String
func fetchUserData(for userId: String) async throws -> User
let maximumRetryCount = 3

// ❌ BAD
var UserName: String
func FetchUserData(For UserId: String) async throws -> User
let maximum_retry_count = 3
```

#### Boolean Properties

```swift
// ✅ GOOD: Use is, has, can, should prefixes
var isEmpty: Bool
var hasUnsavedChanges: Bool
var canSubmit: Bool
var shouldRefresh: Bool

// ❌ BAD
var empty: Bool
var unsavedChanges: Bool
var submit: Bool
```

### Documentation Comments

```swift
/// Fetches user profile from the server.
///
/// Requires network connection and authenticated user.
///
/// - Parameters:
///   - userId: Unique identifier of the user to fetch
///   - includeDetails: Whether to include detailed info (default: false)
/// - Returns: User profile information
/// - Throws: `NetworkError.unauthorized` on auth failure
///           `NetworkError.notFound` when user doesn't exist
func fetchProfile(
    for userId: String,
    includeDetails: Bool = false
) async throws -> UserProfile {
    // Implementation
}
```

## Code Quality Principles

### Pure Functions (CRITICAL)

Minimize side effects and write predictable functions.

```swift
// ✅ GOOD: Pure Function
// - Same input → Same output
// - No external state modification
// - Easy to test
func calculateTotal(items: [Item]) -> Decimal {
    items.reduce(0) { $0 + $1.price }
}

func filterActiveUsers(from users: [User]) -> [User] {
    users.filter { $0.isActive }
}

// ❌ BAD: Impure Function (Side Effects)
var totalCount = 0  // External state

func addToTotal(value: Int) {
    totalCount += value  // Modifies external state
    print("Added \(value)")  // I/O side effect
}
```

### KISS (Keep It Simple)

```swift
// ✅ GOOD: Simple and intuitive
func isValidEmail(_ email: String) -> Bool {
    email.contains("@") && email.contains(".")
}

// ❌ BAD: Overly complex
func isValidEmail(_ email: String) -> Bool {
    let pattern = "^[A-Z0-9a-z._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,64}$"
    let regex = try? NSRegularExpression(pattern: pattern)
    let range = NSRange(email.startIndex..., in: email)
    return regex?.firstMatch(in: email, range: range) != nil
}
// (Keep it simple unless regex is truly necessary)
```

### Early Return Pattern

```swift
// ✅ GOOD: Early return improves readability
func processUser(_ user: User?) -> String {
    guard let user else {
        return "No user"
    }

    guard user.isActive else {
        return "Inactive user"
    }

    guard user.hasPermission else {
        return "No permission"
    }

    return "Welcome, \(user.name)"
}

// ❌ BAD: Deep nesting
func processUser(_ user: User?) -> String {
    if let user {
        if user.isActive {
            if user.hasPermission {
                return "Welcome, \(user.name)"
            } else {
                return "No permission"
            }
        } else {
            return "Inactive user"
        }
    } else {
        return "No user"
    }
}
```

### Guard Statement Usage

```swift
// ✅ GOOD: Use guard to state preconditions
func updateProfile(with data: [String: Any]) throws {
    guard let name = data["name"] as? String else {
        throw ValidationError.missingField("name")
    }

    guard let age = data["age"] as? Int, age >= 0 else {
        throw ValidationError.invalidField("age")
    }

    // Main logic (all preconditions satisfied)
    self.name = name
    self.age = age
}
```

## Type Safety

### Optional Handling

```swift
// ✅ GOOD: guard let (Early exit)
func processValue(_ value: String?) {
    guard let value else { return }
    print(value)
}

// ✅ GOOD: if let (Conditional execution)
if let userName = user?.name {
    greet(userName)
}

// ✅ GOOD: nil coalescing (Default value)
let displayName = user?.name ?? "Anonymous"

// ✅ GOOD: Optional chaining
let uppercased = user?.name?.uppercased()

// ✅ GOOD: map/flatMap
let length = userName.map { $0.count }
let profile = userId.flatMap { users[$0] }
```

### No Forced Unwrap (!) (CRITICAL)

```swift
// ❌ NEVER: Forced unwrap
let name = user!.name  // Crash risk!
let value = dictionary["key"]!  // Crash risk!

// ✅ GOOD: Safe alternatives
// 1. guard let
guard let user else {
    fatalError("User must exist at this point - programming error")
}
let name = user.name

// 2. if let
if let value = dictionary["key"] {
    process(value)
}

// 3. nil coalescing
let value = dictionary["key"] ?? defaultValue

// 4. Optional chaining
let name = user?.name

// ✅ Exception: Compile-time guaranteed cases only
let url = URL(string: "https://apple.com")!  // Literals are OK
let image = UIImage(systemName: "star.fill")!  // SF Symbols are OK
```

### Result Type

```swift
// ✅ GOOD: Explicit success/failure
func fetchUser(id: String) async -> Result<User, NetworkError> {
    do {
        let user = try await api.getUser(id: id)
        return .success(user)
    } catch let error as NetworkError {
        return .failure(error)
    } catch {
        return .failure(.unknown(error))
    }
}

// Usage
let result = await fetchUser(id: "123")
switch result {
case .success(let user):
    display(user)
case .failure(let error):
    showError(error)
}
```

### Strongly Typed IDs

```swift
// ✅ GOOD: Type-safe IDs
struct UserID: Hashable, Codable {
    let rawValue: String
}

struct OrderID: Hashable, Codable {
    let rawValue: String
}

func fetchUser(id: UserID) async throws -> User { }
func fetchOrder(id: OrderID) async throws -> Order { }

// Compiler prevents mistakes
let userId = UserID(rawValue: "user-123")
let orderId = OrderID(rawValue: "order-456")

fetchUser(id: orderId)  // ❌ Compile error!

// ❌ BAD: String types can be confused
func fetchUser(id: String) async throws -> User { }
func fetchOrder(id: String) async throws -> Order { }

fetchUser(id: orderId)  // Compiles even with mistake
```

## Error Handling

### throws / do-catch

```swift
// ✅ GOOD: Clear error definition
enum NetworkError: Error, LocalizedError {
    case invalidURL
    case requestFailed(statusCode: Int)
    case decodingFailed(Error)
    case unauthorized

    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "Invalid URL"
        case .requestFailed(let code):
            return "Request failed (code: \(code))"
        case .decodingFailed:
            return "Failed to parse data"
        case .unauthorized:
            return "Authentication required"
        }
    }
}

// Throwing errors
func fetchData(from urlString: String) async throws -> Data {
    guard let url = URL(string: urlString) else {
        throw NetworkError.invalidURL
    }

    let (data, response) = try await URLSession.shared.data(from: url)

    guard let httpResponse = response as? HTTPURLResponse else {
        throw NetworkError.requestFailed(statusCode: -1)
    }

    guard httpResponse.statusCode == 200 else {
        throw NetworkError.requestFailed(statusCode: httpResponse.statusCode)
    }

    return data
}

// Handling errors
do {
    let data = try await fetchData(from: url)
    process(data)
} catch let error as NetworkError {
    handleNetworkError(error)
} catch {
    handleUnknownError(error)
}
```

### Typed Throws (Swift 6)

```swift
// ✅ Swift 6: Typed throws
func fetchUser(id: String) async throws(NetworkError) -> User {
    guard !id.isEmpty else {
        throw .invalidURL
    }
    // ...
}

// Caller knows the error type
do {
    let user = try await fetchUser(id: "123")
} catch {
    // error is inferred as NetworkError
    switch error {
    case .unauthorized:
        promptLogin()
    case .requestFailed(let code):
        showError("Failed: \(code)")
    default:
        showGenericError()
    }
}
```

## File/Code Structure

### Extension Organization (MARK Comments)

```swift
// MARK: - MyView.swift

struct MyView: View {
    @State private var viewModel = ViewModel()

    var body: some View {
        // ...
    }
}

// MARK: - Subviews

private extension MyView {
    var headerView: some View {
        // ...
    }

    var contentView: some View {
        // ...
    }
}

// MARK: - Actions

private extension MyView {
    func handleSubmit() {
        // ...
    }
}

// MARK: - ViewModel

extension MyView {
    @Observable @MainActor
    final class ViewModel {
        // ...
    }
}
```

### Access Control

```swift
// Start with the most restrictive access level
public struct APIClient {  // Exposed outside module

    public let baseURL: URL  // Readable externally

    internal var session: URLSession  // Same module only

    fileprivate var cache: [String: Data]  // Same file only

    private var authToken: String?  // This type only

    public init(baseURL: URL) {
        self.baseURL = baseURL
        self.session = .shared
        self.cache = [:]
    }
}

// Principle: Start private, expand only when needed
```

### File Size Guidelines

- **Recommended**: 200-400 lines
- **Maximum**: 800 lines
- Consider splitting files over 800 lines
- Follow Single Responsibility Principle (SRP)

## Code Quality Checklist

Before marking work complete:
- [ ] Are functions pure where possible?
- [ ] No forced unwrap (!) present?
- [ ] Functions under 50 lines?
- [ ] Files under 800 lines?
- [ ] Nesting depth under 4 levels?
- [ ] Proper error handling?
- [ ] Clear naming?
- [ ] Minimal access control?
