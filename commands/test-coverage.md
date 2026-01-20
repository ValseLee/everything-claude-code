# Test Coverage

Analyze test coverage and generate missing tests for iOS/Swift projects:

1. Run tests with coverage:
   ```bash
   xcodebuild test \
     -scheme MyApp \
     -destination 'platform=iOS Simulator,name=iPhone 16' \
     -enableCodeCoverage YES \
     -resultBundlePath TestResults.xcresult
   ```

2. Extract coverage report:
   ```bash
   xcrun xccov view --report TestResults.xcresult --json > coverage.json

   # Or human-readable
   xcrun xccov view --report TestResults.xcresult
   ```

3. Identify files below 80% coverage threshold

4. For each under-covered file:
   - Analyze untested code paths
   - Generate unit tests for functions
   - Generate integration tests for ViewModels
   - Generate UI tests for critical flows

5. Verify new tests pass

6. Show before/after coverage metrics

7. Ensure project reaches 80%+ overall coverage

## Coverage Commands

```bash
# Run tests with coverage
xcodebuild test \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  -enableCodeCoverage YES \
  -resultBundlePath TestResults.xcresult \
  2>&1 | xcbeautify

# View coverage summary
xcrun xccov view --report TestResults.xcresult

# View coverage for specific file
xcrun xccov view --report TestResults.xcresult --files-for-target MyApp | grep "ViewModel"

# Export as JSON for processing
xcrun xccov view --report TestResults.xcresult --json > coverage.json

# View line-by-line coverage
xcrun xccov view --file Sources/ViewModels/HomeViewModel.swift TestResults.xcresult
```

## Coverage Report Format

```
╔══════════════════════════════════════════════════════════════╗
║                   Test Coverage Report                        ║
╠══════════════════════════════════════════════════════════════╣
║ Target: MyApp                                                 ║
║ Overall Coverage: 73.2%  (Target: 80%)                       ║
╚══════════════════════════════════════════════════════════════╝

Files Below Threshold (80%):

| File                          | Coverage | Lines   | Gap    |
|-------------------------------|----------|---------|--------|
| ViewModels/SettingsViewModel  | 45.2%    | 120/265 | 34.8%  |
| Services/AuthService          | 62.1%    | 87/140  | 17.9%  |
| Utils/DateFormatter           | 71.3%    | 62/87   | 8.7%   |

Files Meeting Threshold:

| File                          | Coverage | Lines   |
|-------------------------------|----------|---------|
| ViewModels/HomeViewModel      | 94.5%    | 189/200 |
| Models/User                   | 100.0%   | 45/45   |
| Services/APIClient            | 88.2%    | 150/170 |

Action Required:
- Add 23 tests to reach 80% coverage
- Focus on SettingsViewModel (highest gap)
```

## Generating Missing Tests

For each under-covered file, analyze and generate tests:

```swift
// Example: SettingsViewModel with 45% coverage

// Untested methods identified:
// - updateNotificationSettings() - 0% covered
// - exportData() - 0% covered
// - deleteAccount() - 20% covered (only error path)

// Generated tests:
import Testing
@testable import MyApp

@Suite("SettingsViewModel Tests")
struct SettingsViewModelTests {

    @Test("Update notification settings saves to preferences")
    @MainActor
    func updateNotificationSettings() async {
        let mockPreferences = MockUserPreferences()
        let sut = SettingsViewModel(preferences: mockPreferences)

        await sut.updateNotificationSettings(enabled: true)

        #expect(mockPreferences.notificationsEnabled == true)
        #expect(sut.isLoading == false)
    }

    @Test("Export data generates valid file")
    @MainActor
    func exportDataSuccess() async throws {
        let mockExporter = MockDataExporter()
        let sut = SettingsViewModel(exporter: mockExporter)

        let url = try await sut.exportData()

        #expect(url != nil)
        #expect(mockExporter.exportCalled == true)
    }

    @Test("Delete account clears all data")
    @MainActor
    func deleteAccountSuccess() async throws {
        let mockAuth = MockAuthService()
        let mockStorage = MockStorageService()
        let sut = SettingsViewModel(auth: mockAuth, storage: mockStorage)

        try await sut.deleteAccount()

        #expect(mockAuth.signOutCalled == true)
        #expect(mockStorage.clearAllCalled == true)
    }

    @Test("Delete account handles network error")
    @MainActor
    func deleteAccountNetworkError() async {
        let mockAuth = MockAuthService()
        mockAuth.shouldFail = true
        let sut = SettingsViewModel(auth: mockAuth)

        await #expect(throws: AuthError.networkError) {
            try await sut.deleteAccount()
        }
    }
}
```

## Focus Areas for Coverage

**Unit Tests** (Function-level):
- Happy path scenarios
- Edge cases (empty, nil, max values)
- Error conditions
- Boundary values

**Integration Tests** (Module-level):
- ViewModel with real/mock services
- Repository operations
- Use case combinations

**UI Tests** (for critical flows only):
- Onboarding flow
- Authentication
- Primary user actions

## Coverage Targets

- **80% minimum** overall coverage
- **90%+ recommended** for:
  - ViewModels
  - Services
  - Business logic
- **100% required** for:
  - Financial calculations
  - Authentication logic
  - Security-critical code

## Excluding Files from Coverage

Some files should be excluded from coverage metrics:

```swift
// In test plan or xccov command, exclude:
// - Generated code (e.g., SwiftGen output)
// - Preview providers
// - App/Scene delegates (minimal logic)
// - UI-only Views (test via UI tests)
```

## CI Integration

```yaml
# .github/workflows/coverage.yml
- name: Run Tests with Coverage
  run: |
    xcodebuild test \
      -scheme MyApp \
      -destination 'platform=iOS Simulator,name=iPhone 16' \
      -enableCodeCoverage YES \
      -resultBundlePath TestResults.xcresult

- name: Check Coverage Threshold
  run: |
    COVERAGE=$(xcrun xccov view --report TestResults.xcresult --json | jq '.lineCoverage')
    if (( $(echo "$COVERAGE < 0.80" | bc -l) )); then
      echo "Coverage $COVERAGE is below 80% threshold"
      exit 1
    fi

- name: Upload Coverage Report
  uses: actions/upload-artifact@v3
  with:
    name: coverage-report
    path: TestResults.xcresult
```

## Quick Reference

```bash
# Run tests with coverage
xcodebuild test -scheme MyApp -enableCodeCoverage YES -resultBundlePath Results.xcresult

# View overall coverage
xcrun xccov view --report Results.xcresult

# View file coverage
xcrun xccov view --report Results.xcresult --files-for-target MyApp

# Export JSON
xcrun xccov view --report Results.xcresult --json > coverage.json

# View specific file line coverage
xcrun xccov view --file path/to/file.swift Results.xcresult
```

Focus on:
- Happy path scenarios
- Error handling
- Edge cases (nil, empty, boundary values)
- Async operations
