# Testing Requirements

## Minimum Test Coverage: 80%

Test Types (ALL required):
1. **Unit Tests** - Individual functions, ViewModels, use cases (Swift Testing)
2. **Integration Tests** - Repository, API clients, database operations
3. **UI Tests** - Critical user flows (XCUITest)

## Test-Driven Development

MANDATORY workflow:
1. Write test first (RED)
2. Run test - it should FAIL
3. Write minimal implementation (GREEN)
4. Run test - it should PASS
5. Refactor (IMPROVE)
6. Verify coverage (80%+)

## Swift Testing Framework

```swift
import Testing
@testable import MyApp

@Suite("User Service")
struct UserServiceTests {

    @Test("fetches user successfully")
    func fetchUser() async throws {
        let service = UserService(api: MockAPI())
        let user = try await service.fetchUser(id: "123")
        #expect(user.id == "123")
    }

    @Test("throws error for invalid ID")
    func fetchUserInvalidID() async {
        let service = UserService(api: MockAPI())
        await #expect(throws: UserError.notFound) {
            try await service.fetchUser(id: "invalid")
        }
    }
}
```

## XCUITest for E2E

```swift
import XCTest

final class LoginUITests: XCTestCase {

    func testSuccessfulLogin() {
        let app = XCUIApplication()
        app.launch()

        app.textFields["email"].tap()
        app.textFields["email"].typeText("test@example.com")

        app.secureTextFields["password"].tap()
        app.secureTextFields["password"].typeText("password123")

        app.buttons["Login"].tap()

        XCTAssertTrue(app.staticTexts["Welcome"].waitForExistence(timeout: 5))
    }
}
```

## Running Tests

```bash
# Run all tests
xcodebuild test -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 15'

# With coverage
xcodebuild test -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 15' \
  -enableCodeCoverage YES

# Run specific test suite
xcodebuild test -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 15' \
  -only-testing:MyAppTests/UserServiceTests
```

## Troubleshooting Test Failures

1. Use **tdd-guide** agent
2. Check test isolation (no shared state)
3. Verify mocks are correct
4. Fix implementation, not tests (unless tests are wrong)

## Agent Support

- **tdd-guide** - Use PROACTIVELY for new features, enforces write-tests-first
- **e2e-runner** - XCUITest E2E testing specialist
