# iOS Migration Plan

This document tracks the migration of `everything-claude-code` repository from web development (React/Next.js/Node.js) to iOS development (Swift/SwiftUI).

## Target Specifications

- **iOS Version**: iOS 18+
- **Swift Version**: Swift 6.2
- **UI Framework**: SwiftUI (Stateless View + ViewModel pattern)
- **Testing**: Swift Testing framework
- **Data Persistence**: SwiftData
- **Architecture**: Clean Architecture + Modular Architecture

## Key Patterns

### View-ViewModel Pattern (CRITICAL)
```swift
struct MyView: View {
    @State private var viewModel = ViewModel()
    var body: some View { /* ... */ }
}

extension MyView {
    @Observable @MainActor
    final class ViewModel {
        // State and actions
    }
}
```

### Core Principles
1. **Pure Functions** - minimize side effects
2. **No Forced Unwrap (!)** - except compile-time guaranteed values
3. **Stateless Views** - all state in ViewModel
4. **TDD** - write tests first
5. **80% coverage** minimum

---

## Migration Status

### Phase 1: Skills (COMPLETED)

| File | Status | Description |
|------|--------|-------------|
| `swift-coding-standards.md` | ✅ Done | Swift API Guidelines, naming, error handling |
| `swift-concurrency.md` | ✅ Done | Swift 6.2 async/await, Actor, Sendable |
| `swiftui-patterns.md` | ✅ Done | iOS 18+ SwiftUI, View-ViewModel pattern |
| `swiftdata.md` | ✅ Done | #Predicate, #Expression, ModelActor |
| `xcode-project.md` | ✅ Done | Workspace, Target, SPM, dependencies |
| `ios-architecture.md` | ✅ Done | Clean Architecture, Modular Architecture |
| `tdd-workflow/SKILL.md` | ✅ Done | Swift Testing framework |
| `security-review/SKILL.md` | ✅ Done | Keychain, ATS, Data Protection |
| `project-guidelines-example.md` | ✅ Done | iOS project example |

**Deleted (web-specific):**
- `frontend-patterns.md`
- `backend-patterns.md`
- `coding-standards.md`
- `clickhouse-io.md`

---

### Phase 2: Agents (COMPLETED)

Files modified in `/agents/`:

| File | Status | Description |
|------|--------|-------------|
| `code-reviewer.md` | ✅ Done | Swift/SwiftUI review, View-ViewModel, Keychain, ATS |
| `tdd-guide.md` | ✅ Done | Swift Testing, @Suite/@Test, #expect, xcodebuild |
| `planner.md` | ✅ Done | iOS planning, View-ViewModel, Clean Architecture |
| `architect.md` | ✅ Done | iOS Clean Architecture, SwiftUI, SwiftData |
| `security-reviewer.md` | ✅ Done | Keychain, ATS, Data Protection, iOS security |
| `build-error-resolver.md` | ✅ Done | xcodebuild, SPM, Swift compiler errors |
| `e2e-runner.md` | ✅ Done | XCUITest, Page Object pattern |
| `refactor-cleaner.md` | ✅ Done | Swift patterns, @Observable, View-ViewModel |
| `doc-updater.md` | ✅ Done | Swift documentation, xcconfig, SwiftData models |

---

### Phase 3: Commands (COMPLETED)

Files modified in `/commands/`:

| File | Status | Description |
|------|--------|-------------|
| `tdd.md` | ✅ Done | Swift Testing, xcodebuild test, @Suite/@Test |
| `plan.md` | ✅ Done | iOS planning with SwiftUI/SwiftData examples |
| `e2e.md` | ✅ Done | XCUITest, Page Object pattern |
| `code-review.md` | ✅ Done | Swift security, concurrency, SwiftUI patterns |
| `build-fix.md` | ✅ Done | xcodebuild, SPM, common Swift errors |
| `refactor-clean.md` | ✅ Done | Periphery, SwiftLint, Swift dead code |
| `test-coverage.md` | ✅ Done | xccov, xcresult, coverage reports |
| `update-codemaps.md` | ✅ Done | Swift modules, iOS architecture |
| `update-docs.md` | ✅ Done | Package.swift, xcconfig, Xcode docs |

---

### Phase 4: Rules (COMPLETED)

Files modified in `/rules/`:

| File | Status | Description |
|------|--------|-------------|
| `coding-style.md` | ✅ Done | Swift immutability, Codable, pure functions |
| `testing.md` | ✅ Done | Swift Testing, XCUITest examples |
| `git-workflow.md` | ✅ Done | iOS-specific git considerations |
| `security.md` | ✅ Done | Keychain, ATS, iOS security |
| `performance.md` | ✅ Done | SwiftUI optimization patterns |
| `patterns.md` | ✅ Done | View-ViewModel, Repository, Use Case patterns |
| `agents.md` | ✅ Done | iOS-specific agent descriptions |
| `hooks.md` | ✅ Done | SwiftFormat, SwiftLint examples |

---

### Phase 5: Other Files (COMPLETED)

| Directory | Files | Status |
|-----------|-------|--------|
| `examples/` | CLAUDE.md, user-CLAUDE.md | ✅ Done |
| `hooks/` | hooks.json | ✅ Done |
| `mcp-configs/` | mcp-servers.json | ✅ Done |
| `README.md` | Repository docs | ✅ Done |

---

## Notes

- All documents should be written in **English**
- Use Swift code examples throughout
- Reference the new skills files for consistency
- Keep structure/format similar to original, just change content
