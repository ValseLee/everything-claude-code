---
description: Generate and run end-to-end UI tests with XCUITest. Creates test journeys, runs tests, captures screenshots, and generates reports.
---

# E2E Command

This command invokes the **e2e-runner** agent to generate, maintain, and execute end-to-end UI tests using XCUITest for iOS applications.

## What This Command Does

1. **Generate Test Journeys** - Create XCUITest tests for user flows
2. **Run UI Tests** - Execute tests on simulators/devices
3. **Capture Artifacts** - Screenshots and recordings on failures
4. **Generate Reports** - Test results and coverage
5. **Identify Flaky Tests** - Quarantine unstable tests

## When to Use

Use `/e2e` when:
- Testing critical user journeys (onboarding, authentication, purchases)
- Verifying multi-step flows work end-to-end
- Testing UI interactions and navigation
- Validating integration between screens
- Preparing for App Store submission

## How It Works

The e2e-runner agent will:

1. **Analyze user flow** and identify test scenarios
2. **Generate XCUITest** using Page Object pattern
3. **Run tests** on specified simulator/device
4. **Capture failures** with screenshots and recordings
5. **Generate report** with results and artifacts
6. **Identify flaky tests** and recommend fixes

## Example Usage

```
User: /e2e Test the portfolio creation and viewing flow

Agent (e2e-runner):
# E2E Test Generation: Portfolio Creation and Viewing Flow

## Test Scenario Identified

**User Journey:** Create Portfolio → Add Assets → View Portfolio → Check Performance

**Test Coverage:**
1. Navigate to portfolio tab
2. Tap create new portfolio
3. Enter portfolio name
4. Add assets to portfolio
5. Save portfolio
6. Verify portfolio appears in list
7. View portfolio details
8. Verify performance chart renders

## Generated Test Code

```swift
// UITests/Portfolio/PortfolioCreationTests.swift
import XCTest

final class PortfolioCreationTests: XCTestCase {
    private var app: XCUIApplication!
    private var portfolioPage: PortfolioPage!
    private var createPortfolioPage: CreatePortfolioPage!

    override func setUpWithError() throws {
        continueAfterFailure = false
        app = XCUIApplication()
        app.launchArguments = ["--uitesting"]
        app.launch()

        portfolioPage = PortfolioPage(app: app)
        createPortfolioPage = CreatePortfolioPage(app: app)
    }

    override func tearDownWithError() throws {
        app = nil
    }

    @MainActor
    func testUserCanCreatePortfolioAndViewDetails() throws {
        // 1. Navigate to portfolio tab
        portfolioPage.navigateToPortfolios()

        // Verify page loaded
        XCTAssertTrue(portfolioPage.isDisplayed)
        XCTAssertTrue(portfolioPage.headerTitle.exists)

        // 2. Tap create new portfolio
        portfolioPage.tapCreateButton()

        // 3. Enter portfolio name
        let portfolioName = "Test Portfolio \(Date().timeIntervalSince1970)"
        createPortfolioPage.enterName(portfolioName)

        // 4. Add assets to portfolio
        createPortfolioPage.tapAddAsset()
        createPortfolioPage.selectAsset(named: "AAPL")
        createPortfolioPage.enterQuantity("10")
        createPortfolioPage.confirmAsset()

        // Take screenshot of asset added
        let screenshot = XCTAttachment(screenshot: app.screenshot())
        screenshot.name = "Asset Added"
        screenshot.lifetime = .keepAlways
        add(screenshot)

        // 5. Save portfolio
        createPortfolioPage.tapSave()

        // Wait for save to complete
        let saveExpectation = expectation(
            for: NSPredicate(format: "exists == true"),
            evaluatedWith: portfolioPage.portfolioCell(named: portfolioName)
        )
        wait(for: [saveExpectation], timeout: 5)

        // 6. Verify portfolio appears in list
        XCTAssertTrue(portfolioPage.portfolioCell(named: portfolioName).exists)

        // 7. View portfolio details
        portfolioPage.tapPortfolio(named: portfolioName)

        // 8. Verify performance chart renders
        let detailsPage = PortfolioDetailsPage(app: app)
        XCTAssertTrue(detailsPage.isDisplayed)
        XCTAssertTrue(detailsPage.performanceChart.waitForExistence(timeout: 3))
        XCTAssertTrue(detailsPage.assetList.exists)

        // Take screenshot of portfolio details
        let detailsScreenshot = XCTAttachment(screenshot: app.screenshot())
        detailsScreenshot.name = "Portfolio Details"
        detailsScreenshot.lifetime = .keepAlways
        add(detailsScreenshot)
    }

    @MainActor
    func testEmptyStateShowsCreatePrompt() throws {
        // Navigate with empty state flag
        app.launchArguments.append("--empty-portfolio-state")
        app.launch()

        portfolioPage.navigateToPortfolios()

        // Verify empty state
        XCTAssertTrue(portfolioPage.emptyStateView.exists)
        XCTAssertTrue(portfolioPage.emptyStateCreateButton.exists)

        // Tap create from empty state
        portfolioPage.emptyStateCreateButton.tap()

        // Verify create page opens
        XCTAssertTrue(createPortfolioPage.isDisplayed)
    }

    @MainActor
    func testCanCancelPortfolioCreation() throws {
        portfolioPage.navigateToPortfolios()
        portfolioPage.tapCreateButton()

        // Enter some data
        createPortfolioPage.enterName("Cancelled Portfolio")

        // Cancel
        createPortfolioPage.tapCancel()

        // Verify back on portfolio list
        XCTAssertTrue(portfolioPage.isDisplayed)

        // Verify portfolio was not created
        XCTAssertFalse(portfolioPage.portfolioCell(named: "Cancelled Portfolio").exists)
    }
}
```

## Page Objects

```swift
// UITests/Pages/PortfolioPage.swift
import XCTest

struct PortfolioPage {
    private let app: XCUIApplication

    init(app: XCUIApplication) {
        self.app = app
    }

    // MARK: - Elements

    var isDisplayed: Bool {
        app.navigationBars["Portfolios"].exists
    }

    var headerTitle: XCUIElement {
        app.navigationBars["Portfolios"].staticTexts["Portfolios"]
    }

    var createButton: XCUIElement {
        app.buttons["createPortfolioButton"]
    }

    var emptyStateView: XCUIElement {
        app.otherElements["emptyStateView"]
    }

    var emptyStateCreateButton: XCUIElement {
        app.buttons["emptyStateCreateButton"]
    }

    func portfolioCell(named name: String) -> XCUIElement {
        app.cells.containing(.staticText, identifier: name).firstMatch
    }

    // MARK: - Actions

    func navigateToPortfolios() {
        app.tabBars.buttons["Portfolio"].tap()
    }

    func tapCreateButton() {
        createButton.tap()
    }

    func tapPortfolio(named name: String) {
        portfolioCell(named: name).tap()
    }
}

// UITests/Pages/CreatePortfolioPage.swift
struct CreatePortfolioPage {
    private let app: XCUIApplication

    init(app: XCUIApplication) {
        self.app = app
    }

    var isDisplayed: Bool {
        app.navigationBars["Create Portfolio"].exists
    }

    var nameTextField: XCUIElement {
        app.textFields["portfolioNameField"]
    }

    func enterName(_ name: String) {
        nameTextField.tap()
        nameTextField.typeText(name)
    }

    func tapAddAsset() {
        app.buttons["addAssetButton"].tap()
    }

    func selectAsset(named name: String) {
        app.cells.containing(.staticText, identifier: name).firstMatch.tap()
    }

    func enterQuantity(_ quantity: String) {
        let field = app.textFields["quantityField"]
        field.tap()
        field.typeText(quantity)
    }

    func confirmAsset() {
        app.buttons["confirmAssetButton"].tap()
    }

    func tapSave() {
        app.buttons["savePortfolioButton"].tap()
    }

    func tapCancel() {
        app.buttons["Cancel"].tap()
    }
}
```

## Running Tests

```bash
# Run the generated test
xcodebuild test \
  -scheme MyAppUITests \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  -only-testing:MyAppUITests/PortfolioCreationTests \
  2>&1 | xcbeautify

Test Suite PortfolioCreationTests started
✓ testUserCanCreatePortfolioAndViewDetails (4.2s)
✓ testEmptyStateShowsCreatePrompt (1.8s)
✓ testCanCancelPortfolioCreation (2.1s)

Executed 3 tests, with 0 failures (8.1s)

Artifacts generated:
- TestResults/Asset Added.png
- TestResults/Portfolio Details.png
```

## Test Report

```
╔══════════════════════════════════════════════════════════════╗
║                    E2E Test Results                          ║
╠══════════════════════════════════════════════════════════════╣
║ Status:     ✅ ALL TESTS PASSED                              ║
║ Total:      3 tests                                          ║
║ Passed:     3 (100%)                                         ║
║ Failed:     0                                                ║
║ Flaky:      0                                                ║
║ Duration:   8.1s                                             ║
╚══════════════════════════════════════════════════════════════╝

Artifacts:
📸 Screenshots: 2 files
📹 Videos: 0 files (only on failure)
📊 Test Results: TestResults/

View results: open TestResults/
```

✅ E2E test suite ready for CI/CD integration!
```

## Test Artifacts

When tests run, the following artifacts are captured:

**On All Tests:**
- Test results in `.xcresult` bundle
- Performance metrics

**On Failure Only:**
- Screenshot of the failing state
- Screen recording (if enabled)
- UI hierarchy dump
- Console logs

## Viewing Artifacts

```bash
# Open test results
open Build/Logs/Test/*.xcresult

# Extract screenshots from xcresult
xcrun xcresulttool get --path TestResults.xcresult --format json

# View in Xcode
# Xcode > Window > Devices and Simulators > View Device Logs
```

## Flaky Test Detection

If a test fails intermittently:

```
⚠️  FLAKY TEST DETECTED: PortfolioCreationTests/testUserCanCreatePortfolio

Test passed 7/10 runs (70% pass rate)

Common failure:
"Failed to find element 'savePortfolioButton' within 2.0 seconds"

Recommended fixes:
1. Increase timeout: waitForExistence(timeout: 5)
2. Add explicit wait for loading state to complete
3. Check for animation completion before asserting
4. Verify accessibility identifier is set correctly

Quarantine recommendation: Mark with XCTExpectFailure until fixed
```

## Device Configuration

Tests can run on multiple simulators:

```bash
# Run on multiple devices
xcodebuild test \
  -scheme MyAppUITests \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  -destination 'platform=iOS Simulator,name=iPad Pro 13-inch (M4)'

# Run on specific iOS version
xcodebuild test \
  -scheme MyAppUITests \
  -destination 'platform=iOS Simulator,name=iPhone 16,OS=18.0'
```

## CI/CD Integration

Add to your CI pipeline:

```yaml
# .github/workflows/ui-tests.yml
- name: Run UI Tests
  run: |
    xcodebuild test \
      -scheme MyAppUITests \
      -destination 'platform=iOS Simulator,name=iPhone 16' \
      -resultBundlePath TestResults.xcresult \
      2>&1 | xcbeautify

- name: Upload Test Results
  if: always()
  uses: actions/upload-artifact@v3
  with:
    name: test-results
    path: TestResults.xcresult
```

## Critical Flows to Test

Prioritize these E2E tests for iOS apps:

**CRITICAL (Must Always Pass):**
1. User can complete onboarding
2. User can sign in / sign up
3. User can view main content
4. User can perform primary action
5. User can access settings
6. User can sign out
7. Deep links work correctly

**IMPORTANT:**
1. Push notification handling
2. Background refresh
3. Offline mode behavior
4. Accessibility features work
5. Dynamic Type support
6. Dark mode appearance

## Best Practices

**DO:**
- ✅ Use Page Object pattern for maintainability
- ✅ Use accessibility identifiers for selectors
- ✅ Wait for elements explicitly, not arbitrary delays
- ✅ Test critical user journeys end-to-end
- ✅ Run tests before merging to main
- ✅ Review artifacts when tests fail
- ✅ Use `XCTAttachment` for screenshots

**DON'T:**
- ❌ Use sleep() for timing (use waitForExistence)
- ❌ Test implementation details
- ❌ Run tests against production APIs
- ❌ Ignore flaky tests
- ❌ Test every edge case with E2E (use unit tests)
- ❌ Hard-code test data

## XCUITest Quick Reference

```swift
// Finding elements
app.buttons["identifier"]
app.staticTexts["Label"]
app.textFields["placeholder"]
app.cells.containing(.staticText, identifier: "text")

// Waiting
element.waitForExistence(timeout: 5)
let predicate = NSPredicate(format: "exists == true")
expectation(for: predicate, evaluatedWith: element)

// Interactions
element.tap()
element.doubleTap()
element.swipeUp()
element.typeText("text")

// Assertions
XCTAssertTrue(element.exists)
XCTAssertEqual(element.label, "expected")
XCTAssertTrue(element.isHittable)

// Screenshots
let screenshot = XCTAttachment(screenshot: app.screenshot())
screenshot.lifetime = .keepAlways
add(screenshot)
```

## Integration with Other Commands

- Use `/plan` to identify critical journeys to test
- Use `/tdd` for unit tests (faster, more granular)
- Use `/e2e` for integration and user journey tests
- Use `/code-review` to verify test quality

## Related Agents

This command invokes the `e2e-runner` agent located at:
`~/.claude/agents/e2e-runner.md`

## Quick Commands

```bash
# Run all UI tests
xcodebuild test -scheme MyAppUITests -destination 'platform=iOS Simulator,name=iPhone 16'

# Run specific test file
xcodebuild test -scheme MyAppUITests -only-testing:MyAppUITests/PortfolioCreationTests

# Run with recording (for debugging)
xcodebuild test -scheme MyAppUITests -destination 'platform=iOS Simulator,name=iPhone 16' \
  -enableScreenRecording YES

# Generate test plan
# Xcode > Product > Test Plan > New Test Plan

# Debug test
# Xcode > Product > Test (with breakpoints set)
```
