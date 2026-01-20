---
description: Enforce test-driven development workflow. Scaffold protocols, generate tests FIRST using Swift Testing, then implement minimal code to pass. Ensure 80%+ coverage.
---

# TDD Command

This command invokes the **tdd-guide** agent to enforce test-driven development methodology for iOS/Swift projects.

## What This Command Does

1. **Scaffold Protocols** - Define protocols/interfaces first
2. **Generate Tests First** - Write failing tests (RED) using Swift Testing
3. **Implement Minimal Code** - Write just enough to pass (GREEN)
4. **Refactor** - Improve code while keeping tests green (REFACTOR)
5. **Verify Coverage** - Ensure 80%+ test coverage

## When to Use

Use `/tdd` when:
- Implementing new features
- Adding new functions/ViewModels/Services
- Fixing bugs (write test that reproduces bug first)
- Refactoring existing code
- Building critical business logic

## How It Works

The tdd-guide agent will:

1. **Define protocols** for inputs/outputs
2. **Write tests that will FAIL** (because code doesn't exist yet)
3. **Run tests** and verify they fail for the right reason
4. **Write minimal implementation** to make tests pass
5. **Run tests** and verify they pass
6. **Refactor** code while keeping tests green
7. **Check coverage** and add more tests if below 80%

## TDD Cycle

```
RED → GREEN → REFACTOR → REPEAT

RED:      Write a failing test
GREEN:    Write minimal code to pass
REFACTOR: Improve code, keep tests passing
REPEAT:   Next feature/scenario
```

## Example Usage

```
User: /tdd I need a function to calculate portfolio risk score

Agent (tdd-guide):
# TDD Session: Portfolio Risk Score Calculator

## Step 1: Define Protocol (SCAFFOLD)

```swift
// Sources/Domain/Models/PortfolioData.swift
struct PortfolioData: Sendable {
    let totalValue: Decimal
    let volatility: Double
    let assetCount: Int
    let lastRebalanceDate: Date
}

// Sources/Domain/Services/RiskCalculating.swift
protocol RiskCalculating: Sendable {
    func calculateRiskScore(portfolio: PortfolioData) -> Int
}

// Sources/Domain/Services/RiskCalculator.swift
struct RiskCalculator: RiskCalculating {
    func calculateRiskScore(portfolio: PortfolioData) -> Int {
        // TODO: Implementation
        fatalError("Not implemented")
    }
}
```

## Step 2: Write Failing Test (RED)

```swift
// Tests/DomainTests/RiskCalculatorTests.swift
import Testing
@testable import Domain

@Suite("RiskCalculator Tests")
struct RiskCalculatorTests {
    let sut = RiskCalculator()

    @Test("Returns high score for diversified low-volatility portfolio")
    func highScoreForLowRisk() {
        let portfolio = PortfolioData(
            totalValue: 100_000,
            volatility: 0.05,
            assetCount: 20,
            lastRebalanceDate: Date()
        )

        let score = sut.calculateRiskScore(portfolio: portfolio)

        #expect(score > 80)
        #expect(score <= 100)
    }

    @Test("Returns low score for concentrated high-volatility portfolio")
    func lowScoreForHighRisk() {
        let portfolio = PortfolioData(
            totalValue: 10_000,
            volatility: 0.8,
            assetCount: 2,
            lastRebalanceDate: Calendar.current.date(byAdding: .day, value: -180, to: Date())!
        )

        let score = sut.calculateRiskScore(portfolio: portfolio)

        #expect(score < 30)
        #expect(score >= 0)
    }

    @Test("Returns zero for empty portfolio")
    func zeroForEmptyPortfolio() {
        let portfolio = PortfolioData(
            totalValue: 0,
            volatility: 0,
            assetCount: 0,
            lastRebalanceDate: Date()
        )

        let score = sut.calculateRiskScore(portfolio: portfolio)

        #expect(score == 0)
    }
}
```

## Step 3: Run Tests - Verify FAIL

```bash
xcodebuild test \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  -only-testing:DomainTests/RiskCalculatorTests \
  2>&1 | xcbeautify

Test Suite RiskCalculatorTests started
✗ highScoreForLowRisk - Fatal error: Not implemented
✗ lowScoreForHighRisk - Fatal error: Not implemented
✗ zeroForEmptyPortfolio - Fatal error: Not implemented

Executed 3 tests, with 3 failures
```

✅ Tests fail as expected. Ready to implement.

## Step 4: Implement Minimal Code (GREEN)

```swift
// Sources/Domain/Services/RiskCalculator.swift
struct RiskCalculator: RiskCalculating {
    func calculateRiskScore(portfolio: PortfolioData) -> Int {
        // Handle empty portfolio edge case
        guard portfolio.totalValue > 0 else { return 0 }

        // Calculate component scores (0-100 scale)
        let volatilityScore = max(100 - Int(portfolio.volatility * 100), 0)
        let diversificationScore = min(portfolio.assetCount * 5, 100)

        // Recent rebalance bonus
        let daysSinceRebalance = Calendar.current.dateComponents(
            [.day],
            from: portfolio.lastRebalanceDate,
            to: Date()
        ).day ?? 365
        let recencyScore = max(100 - daysSinceRebalance, 0)

        // Weighted average
        let weightedScore = Double(volatilityScore) * 0.4 +
                           Double(diversificationScore) * 0.4 +
                           Double(recencyScore) * 0.2

        return min(max(Int(weightedScore), 0), 100)
    }
}
```

## Step 5: Run Tests - Verify PASS

```bash
xcodebuild test \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  -only-testing:DomainTests/RiskCalculatorTests \
  2>&1 | xcbeautify

Test Suite RiskCalculatorTests started
✓ highScoreForLowRisk (0.003s)
✓ lowScoreForHighRisk (0.002s)
✓ zeroForEmptyPortfolio (0.001s)

Executed 3 tests, with 0 failures
```

✅ All tests passing!

## Step 6: Refactor (IMPROVE)

```swift
// Sources/Domain/Services/RiskCalculator.swift
struct RiskCalculator: RiskCalculating {
    private enum Weight {
        static let volatility = 0.4
        static let diversification = 0.4
        static let recency = 0.2
    }

    private enum Scale {
        static let diversificationMultiplier = 5
        static let maxScore = 100
    }

    func calculateRiskScore(portfolio: PortfolioData) -> Int {
        guard portfolio.totalValue > 0 else { return 0 }

        let components = calculateComponents(portfolio: portfolio)
        let weightedScore = applyWeights(components)

        return clamp(Int(weightedScore), min: 0, max: Scale.maxScore)
    }

    private func calculateComponents(portfolio: PortfolioData) -> (volatility: Int, diversification: Int, recency: Int) {
        let volatility = Scale.maxScore - Int(portfolio.volatility * 100)
        let diversification = portfolio.assetCount * Scale.diversificationMultiplier
        let recency = Scale.maxScore - daysSinceRebalance(portfolio.lastRebalanceDate)

        return (
            clamp(volatility, min: 0, max: Scale.maxScore),
            clamp(diversification, min: 0, max: Scale.maxScore),
            clamp(recency, min: 0, max: Scale.maxScore)
        )
    }

    private func applyWeights(_ components: (volatility: Int, diversification: Int, recency: Int)) -> Double {
        Double(components.volatility) * Weight.volatility +
        Double(components.diversification) * Weight.diversification +
        Double(components.recency) * Weight.recency
    }

    private func daysSinceRebalance(_ date: Date) -> Int {
        Calendar.current.dateComponents([.day], from: date, to: Date()).day ?? 365
    }

    private func clamp(_ value: Int, min: Int, max: Int) -> Int {
        Swift.min(Swift.max(value, min), max)
    }
}
```

## Step 7: Verify Tests Still Pass

```bash
xcodebuild test \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  -only-testing:DomainTests/RiskCalculatorTests \
  2>&1 | xcbeautify

Test Suite RiskCalculatorTests started
✓ highScoreForLowRisk (0.003s)
✓ lowScoreForHighRisk (0.002s)
✓ zeroForEmptyPortfolio (0.001s)

Executed 3 tests, with 0 failures
```

✅ Refactoring complete, tests still passing!

## Step 8: Check Coverage

```bash
xcodebuild test \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  -enableCodeCoverage YES \
  2>&1 | xcbeautify

xcrun xccov view --report Build/Logs/Test/*.xcresult --files-for-target Domain

RiskCalculator.swift    100.0%
PortfolioData.swift     100.0%

Coverage: 100% ✅ (Target: 80%)
```

✅ TDD session complete!
```

## TDD Best Practices

**DO:**
- ✅ Write the test FIRST, before any implementation
- ✅ Run tests and verify they FAIL before implementing
- ✅ Write minimal code to make tests pass
- ✅ Refactor only after tests are green
- ✅ Add edge cases and error scenarios
- ✅ Aim for 80%+ coverage (100% for critical code)
- ✅ Use `@Suite` and `@Test` from Swift Testing
- ✅ Use `#expect` for assertions

**DON'T:**
- ❌ Write implementation before tests
- ❌ Skip running tests after each change
- ❌ Write too much code at once
- ❌ Ignore failing tests
- ❌ Test implementation details (test behavior)
- ❌ Mock everything (prefer real dependencies when possible)
- ❌ Use force unwrap (!) in test setup

## Test Types to Include

**Unit Tests** (Function-level):
- Happy path scenarios
- Edge cases (empty, nil, max values)
- Error conditions
- Boundary values

**Integration Tests** (Module-level):
- ViewModel with real services
- Repository with mock persistence
- Use case combinations

**UI Tests** (use `/e2e` command):
- Critical user flows
- Multi-step processes
- Full app integration

## Coverage Requirements

- **80% minimum** for all code
- **100% required** for:
  - Financial calculations
  - Authentication logic
  - Security-critical code
  - Core business logic

## Important Notes

**MANDATORY**: Tests must be written BEFORE implementation. The TDD cycle is:

1. **RED** - Write failing test
2. **GREEN** - Implement to pass
3. **REFACTOR** - Improve code

Never skip the RED phase. Never write code before tests.

## Swift Testing Quick Reference

```swift
import Testing

// Test Suite
@Suite("Feature Tests")
struct FeatureTests {
    // Setup (runs before each test)
    let sut: MyService

    init() {
        sut = MyService()
    }

    // Basic test
    @Test("Description of what it tests")
    func testSomething() {
        #expect(sut.value == expected)
    }

    // Test with arguments
    @Test(arguments: [1, 2, 3])
    func testMultipleValues(value: Int) {
        #expect(value > 0)
    }

    // Async test
    @Test("Async operation completes")
    func testAsync() async throws {
        let result = try await sut.fetchData()
        #expect(result.isEmpty == false)
    }

    // Test that throws
    @Test("Throws error for invalid input")
    func testThrows() {
        #expect(throws: ValidationError.self) {
            try sut.validate(input: "")
        }
    }
}
```

## Integration with Other Commands

- Use `/plan` first to understand what to build
- Use `/tdd` to implement with tests
- Use `/build-fix` if build errors occur
- Use `/code-review` to review implementation
- Use `/test-coverage` to verify coverage

## Related Agents

This command invokes the `tdd-guide` agent located at:
`~/.claude/agents/tdd-guide.md`

And can reference the `tdd-workflow` skill at:
`~/.claude/skills/tdd-workflow/`
