---
name: tdd-workflow
description: Use this skill when writing new features, fixing bugs, or refactoring code. Enforces test-driven development with Swift Testing framework and 80%+ coverage.
---

# Test-Driven Development Workflow

This skill ensures all code development follows TDD principles using Swift Testing framework.

## When to Activate

- Writing new features or functionality
- Fixing bugs or issues
- Refactoring existing code
- Adding new models or services
- Creating new views with ViewModels

## Core Principles

### 1. Tests BEFORE Code
ALWAYS write tests first, then implement code to make tests pass.

### 2. Coverage Requirements
- Minimum 80% coverage (unit + integration)
- All edge cases covered
- Error scenarios tested
- Boundary conditions verified

### 3. Test Types

#### Unit Tests
- Individual functions and methods
- ViewModel logic
- Use cases
- Pure functions and utilities

#### Integration Tests
- Repository implementations
- API client behavior
- Database operations
- Module interactions

#### UI Tests (XCUITest)
- Critical user flows
- Complete workflows
- UI interactions

## TDD Workflow Steps

### Step 1: Write User Stories
```
As a [role], I want to [action], so that [benefit]

Example:
As a user, I want to search for products by name,
so that I can find items without browsing all categories.
```

### Step 2: Generate Test Cases (RED)

```swift
import Testing
@testable import MyApp

@Suite("Product Search")
struct ProductSearchTests {

    @Test("returns matching products for query")
    func searchReturnsMatchingProducts() async throws {
        let repository = MockProductRepository()
        let useCase = SearchProductsUseCase(repository: repository)

        let results = try await useCase.execute(query: "iPhone")

        #expect(results.count > 0)
        #expect(results.allSatisfy { $0.name.contains("iPhone") })
    }

    @Test("returns empty array for no matches")
    func searchReturnsEmptyForNoMatches() async throws {
        let repository = MockProductRepository()
        let useCase = SearchProductsUseCase(repository: repository)

        let results = try await useCase.execute(query: "xyznonexistent")

        #expect(results.isEmpty)
    }

    @Test("handles empty query gracefully")
    func searchHandlesEmptyQuery() async throws {
        let repository = MockProductRepository()
        let useCase = SearchProductsUseCase(repository: repository)

        let results = try await useCase.execute(query: "")

        #expect(results.isEmpty)
    }
}
```

### Step 3: Run Tests (They Should Fail)
```bash
# In Xcode: ⌘ + U
# Or via command line:
xcodebuild test -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 15'
```

### Step 4: Implement Code (GREEN)

```swift
// Implementation guided by tests
protocol SearchProductsUseCase {
    func execute(query: String) async throws -> [Product]
}

final class SearchProductsUseCaseImpl: SearchProductsUseCase {
    private let repository: ProductRepository

    init(repository: ProductRepository) {
        self.repository = repository
    }

    func execute(query: String) async throws -> [Product] {
        guard !query.isEmpty else { return [] }
        return try await repository.search(query: query)
    }
}
```

### Step 5: Run Tests Again
```bash
# Tests should now pass
xcodebuild test -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 15'
```

### Step 6: Refactor (IMPROVE)
Improve code quality while keeping tests green:
- Remove duplication
- Improve naming
- Optimize performance
- Enhance readability

### Step 7: Verify Coverage
```bash
# Generate coverage report
xcodebuild test -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 15' \
  -enableCodeCoverage YES

# View in Xcode: Report Navigator → Coverage
```

## Swift Testing Patterns

### Basic Test Structure

```swift
import Testing
@testable import MyApp

@Suite("Calculator")
struct CalculatorTests {

    // Arrange once for all tests
    let calculator = Calculator()

    @Test("adds two numbers correctly")
    func addition() {
        let result = calculator.add(2, 3)
        #expect(result == 5)
    }

    @Test("subtracts two numbers correctly")
    func subtraction() {
        let result = calculator.subtract(5, 3)
        #expect(result == 2)
    }
}
```

### Async Tests

```swift
@Suite("User Service")
struct UserServiceTests {

    @Test("fetches user successfully")
    func fetchUser() async throws {
        let service = UserService(api: MockAPI())

        let user = try await service.fetchUser(id: "123")

        #expect(user.id == "123")
        #expect(user.name == "John")
    }

    @Test("throws error for invalid user ID")
    func fetchUserInvalidID() async {
        let service = UserService(api: MockAPI())

        await #expect(throws: UserError.notFound) {
            try await service.fetchUser(id: "invalid")
        }
    }
}
```

### Parameterized Tests

```swift
@Suite("Email Validation")
struct EmailValidationTests {

    @Test("validates email format", arguments: [
        ("test@example.com", true),
        ("invalid-email", false),
        ("user@domain", false),
        ("user.name@company.co.uk", true),
        ("", false)
    ])
    func validateEmail(email: String, isValid: Bool) {
        let validator = EmailValidator()
        #expect(validator.isValid(email) == isValid)
    }
}
```

### Test Tags

```swift
extension Tag {
    @Tag static var slow: Self
    @Tag static var integration: Self
    @Tag static var viewModel: Self
}

@Suite("Integration Tests")
struct IntegrationTests {

    @Test("database operations", .tags(.integration, .slow))
    func databaseTest() async throws {
        // Long-running integration test
    }
}

// Run specific tags:
// xcodebuild test -scheme MyApp -only-testing:IntegrationTests
```

### setUp and tearDown

```swift
@Suite("Database Tests")
struct DatabaseTests {
    var database: TestDatabase!

    init() async throws {
        // setUp - runs before each test
        database = try await TestDatabase.create()
    }

    deinit {
        // tearDown - runs after each test
        database.destroy()
    }

    @Test
    func insertUser() async throws {
        try await database.insert(User(name: "Test"))
        let users = try await database.fetchAll()
        #expect(users.count == 1)
    }
}
```

### Testing ViewModels

```swift
@Suite("User List ViewModel")
struct UserListViewModelTests {

    @Test("loads users on init")
    @MainActor
    func loadsUsers() async {
        let mockUseCase = MockFetchUsersUseCase()
        mockUseCase.result = [User.mock, User.mock]

        let viewModel = UserListView.ViewModel(
            fetchUsersUseCase: mockUseCase
        )

        await viewModel.loadUsers()

        #expect(viewModel.state == .loaded(mockUseCase.result))
    }

    @Test("shows error on failure")
    @MainActor
    func showsErrorOnFailure() async {
        let mockUseCase = MockFetchUsersUseCase()
        mockUseCase.error = NetworkError.connectionFailed

        let viewModel = UserListView.ViewModel(
            fetchUsersUseCase: mockUseCase
        )

        await viewModel.loadUsers()

        if case .error(let message) = viewModel.state {
            #expect(message.contains("connection"))
        } else {
            Issue.record("Expected error state")
        }
    }
}
```

### Testing Errors

```swift
@Suite("Order Service")
struct OrderServiceTests {

    @Test("throws insufficient funds error")
    func insufficientFunds() async {
        let service = OrderService()

        await #expect(throws: OrderError.insufficientFunds) {
            try await service.createOrder(amount: 1000, balance: 100)
        }
    }

    @Test("throws any error")
    func throwsError() async {
        let service = OrderService()

        await #expect(throws: (any Error).self) {
            try await service.createOrder(amount: -1, balance: 100)
        }
    }
}
```

## Test Doubles

### Mock Repository

```swift
final class MockUserRepository: UserRepository {
    var users: [User] = []
    var error: Error?
    var fetchCallCount = 0

    func fetchUser(id: UUID) async throws -> User {
        fetchCallCount += 1

        if let error { throw error }

        guard let user = users.first(where: { $0.id == id }) else {
            throw RepositoryError.notFound
        }
        return user
    }

    func save(_ user: User) async throws {
        if let error { throw error }
        users.append(user)
    }
}

// Usage in tests
@Test
func fetchesUser() async throws {
    let mock = MockUserRepository()
    mock.users = [User(id: testID, name: "John")]

    let service = UserService(repository: mock)
    let user = try await service.getUser(id: testID)

    #expect(user.name == "John")
    #expect(mock.fetchCallCount == 1)
}
```

### Spy for Verification

```swift
final class SpyAnalytics: AnalyticsService {
    private(set) var trackedEvents: [(name: String, params: [String: Any])] = []

    func track(event: String, parameters: [String: Any]) {
        trackedEvents.append((event, parameters))
    }
}

@Test
func tracksLoginEvent() async {
    let spy = SpyAnalytics()
    let viewModel = LoginViewModel(analytics: spy)

    await viewModel.login(email: "test@example.com", password: "password")

    #expect(spy.trackedEvents.count == 1)
    #expect(spy.trackedEvents[0].name == "user_login")
}
```

### Stub for Fixed Responses

```swift
struct StubNetworkClient: NetworkClient {
    let response: Data
    let statusCode: Int

    func request(_ endpoint: Endpoint) async throws -> (Data, URLResponse) {
        let url = URL(string: "https://api.example.com")!
        let response = HTTPURLResponse(
            url: url,
            statusCode: statusCode,
            httpVersion: nil,
            headerFields: nil
        )!
        return (self.response, response)
    }
}
```

## Test File Organization

```
MyApp/
├── Sources/
│   ├── Domain/
│   │   ├── UseCases/
│   │   │   └── FetchUserUseCase.swift
│   │   └── Entities/
│   │       └── User.swift
│   └── Presentation/
│       └── UserList/
│           ├── UserListView.swift
│           └── UserListView+ViewModel.swift
└── Tests/
    ├── UnitTests/
    │   ├── Domain/
    │   │   └── FetchUserUseCaseTests.swift
    │   └── Presentation/
    │       └── UserListViewModelTests.swift
    ├── IntegrationTests/
    │   └── UserRepositoryIntegrationTests.swift
    └── Mocks/
        ├── MockUserRepository.swift
        └── MockNetworkClient.swift
```

## Common Testing Mistakes to Avoid

### ❌ WRONG: Testing Implementation Details
```swift
// Don't test private methods or internal state
#expect(viewModel.internalCache.count == 5)
```

### ✅ CORRECT: Test Observable Behavior
```swift
// Test what users/consumers observe
#expect(viewModel.displayedItems.count == 5)
```

### ❌ WRONG: No Test Isolation
```swift
// Tests share state - dangerous!
static var sharedUser: User?

@Test func createsUser() {
    Self.sharedUser = User(name: "Test")
}

@Test func updatesUser() {
    Self.sharedUser?.name = "Updated"  // Depends on previous test!
}
```

### ✅ CORRECT: Independent Tests
```swift
// Each test creates its own data
@Test func createsUser() {
    let user = User(name: "Test")
    #expect(user.name == "Test")
}

@Test func updatesUser() {
    var user = User(name: "Test")
    user.name = "Updated"
    #expect(user.name == "Updated")
}
```

### ❌ WRONG: Testing Multiple Things
```swift
@Test func userOperations() {
    let user = createUser()
    #expect(user != nil)

    updateUser(user)
    #expect(user.name == "Updated")

    deleteUser(user)
    #expect(fetchUser(user.id) == nil)
}
```

### ✅ CORRECT: One Concept Per Test
```swift
@Test func createsUser() { }
@Test func updatesUserName() { }
@Test func deletesUser() { }
```

## Coverage Verification

### Enable Code Coverage
1. Edit Scheme → Test → Options
2. Check "Gather coverage for all targets"

### View Coverage Report
1. Run tests (⌘ + U)
2. Report Navigator (⌘ + 9)
3. Select test run → Coverage tab

### Coverage Thresholds
```
Minimum Requirements:
- Lines: 80%
- Functions: 80%
- Branches: 80%

Critical Code (100% required):
- Authentication logic
- Payment processing
- Data encryption
- Core business rules
```

## Continuous Testing

### Xcode Watch Mode
```bash
# Build and test on save (via Xcode)
# Product → Build For → Testing
# Then tests run automatically on relevant changes
```

### Pre-Commit Hook
```bash
#!/bin/sh
# .git/hooks/pre-commit

xcodebuild test -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 15' \
  -quiet || exit 1

echo "✅ Tests passed"
```

### CI/CD Integration (GitHub Actions)
```yaml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4

      - name: Run Tests
        run: |
          xcodebuild test \
            -scheme MyApp \
            -destination 'platform=iOS Simulator,name=iPhone 15' \
            -enableCodeCoverage YES \
            -resultBundlePath TestResults.xcresult

      - name: Upload Coverage
        uses: codecov/codecov-action@v4
        with:
          xcode: true
          xcode_archive_path: TestResults.xcresult
```

## Best Practices

1. **Write Tests First** - Always TDD
2. **One Concept Per Test** - Focus on single behavior
3. **Descriptive Test Names** - Explain what's tested
4. **Arrange-Act-Assert** - Clear test structure
5. **Use Test Doubles** - Mock, Stub, Spy appropriately
6. **Test Edge Cases** - nil, empty, boundary values
7. **Test Error Paths** - Not just happy paths
8. **Keep Tests Fast** - Unit tests < 100ms each
9. **Maintain Test Independence** - No shared state
10. **Review Coverage Reports** - Identify gaps

## Success Metrics

- 80%+ code coverage achieved
- All tests passing (green)
- No skipped or disabled tests
- Fast test execution (< 30s for unit tests)
- UI tests cover critical user flows
- Tests catch bugs before production

---

**Remember**: Tests are not optional. They are the safety net that enables confident refactoring, rapid development, and production reliability.
