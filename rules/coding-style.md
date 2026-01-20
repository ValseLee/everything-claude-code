# Coding Style

## Pure Functions (CRITICAL)

Minimize side effects and write predictable functions:

```swift
// ✅ GOOD: Pure Function
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

## Immutability

Prefer immutable data structures:

```swift
// ✅ GOOD: Immutable with let
let user = User(name: "John", age: 30)
let updatedUser = User(name: user.name, age: 31)  // New instance

// ✅ GOOD: Value types (struct)
struct User {
    let id: UUID
    var name: String  // var is OK in struct - copy semantics
}

// ❌ BAD: Mutating reference types
class User {
    var name: String  // Shared mutable state
}
```

## File Organization

MANY SMALL FILES > FEW LARGE FILES:
- High cohesion, low coupling
- 200-400 lines typical, 800 max
- Extract utilities from large files
- Organize by feature/domain, not by type

## Error Handling

ALWAYS handle errors comprehensively:

```swift
// ✅ GOOD: Proper error handling
do {
    let result = try await riskyOperation()
    return result
} catch let error as NetworkError {
    logger.error("Network error: \(error.localizedDescription)")
    throw AppError.networkFailed(error)
} catch {
    logger.error("Unexpected error: \(error)")
    throw AppError.unknown(error)
}

// ❌ BAD: Force try
let result = try! riskyOperation()  // Crash risk!
```

## Input Validation

ALWAYS validate user input:

```swift
// ✅ GOOD: Codable with validation
struct UserInput: Codable {
    let email: String
    let age: Int

    init(email: String, age: Int) throws {
        guard email.contains("@") && email.contains(".") else {
            throw ValidationError.invalidEmail
        }
        guard (0...150).contains(age) else {
            throw ValidationError.invalidAge
        }
        self.email = email
        self.age = age
    }
}

// ✅ GOOD: Failable initializer
struct Email {
    let value: String

    init?(_ value: String) {
        guard value.contains("@") else { return nil }
        self.value = value
    }
}
```

## Naming Conventions

Follow Swift API Design Guidelines:

```swift
// Type names: UpperCamelCase
struct UserProfile { }
enum ConnectionState { }
protocol DataProviding { }

// Properties, methods, parameters: lowerCamelCase
var userName: String
func fetchUserData(for userId: String) async throws -> User

// Boolean properties: is, has, can, should prefixes
var isEmpty: Bool
var hasUnsavedChanges: Bool
var canSubmit: Bool
```

## Code Quality Checklist

Before marking work complete:
- [ ] Code is readable and well-named
- [ ] Functions are small (<50 lines)
- [ ] Files are focused (<800 lines)
- [ ] No deep nesting (>4 levels)
- [ ] Proper error handling
- [ ] No print statements in production
- [ ] No hardcoded values
- [ ] No forced unwrap (!) except compile-time guaranteed
- [ ] Pure functions where possible
