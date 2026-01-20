# Refactor Clean

Safely identify and remove dead code with test verification for iOS/Swift projects:

1. Run dead code analysis:
   - **Periphery**: Find unused code in Swift projects
   - **SwiftLint**: Identify code style and potential issues
   - **Xcode Build**: Check for unused warnings

2. Generate comprehensive report in `.reports/dead-code-analysis.md`

3. Categorize findings by severity:
   - SAFE: Unused private functions, test helpers
   - CAUTION: Unused public APIs, ViewModels
   - DANGER: App entry points, protocol conformances

4. Propose safe deletions only

5. Before each deletion:
   - Run full test suite
   - Verify tests pass
   - Apply change
   - Re-run tests
   - Rollback if tests fail

6. Show summary of cleaned items

## Swift Dead Code Analysis Tools

### Periphery (Recommended)

```bash
# Install
brew install peripheryapp/periphery/periphery

# Scan project
periphery scan --project MyApp.xcodeproj --schemes MyApp --targets MyApp

# Output example:
# MyApp/Sources/Services/OldService.swift:15:1: warning: Class 'OldService' is unused
# MyApp/Sources/Utils/StringExtensions.swift:42:5: warning: Function 'deprecated()' is unused
```

### SwiftLint Rules

```yaml
# .swiftlint.yml
opt_in_rules:
  - unused_import
  - unused_declaration
  - unused_closure_parameter

analyzer_rules:
  - unused_import
  - unused_declaration
```

### Xcode Build Warnings

Enable these warnings in Build Settings:
- `CLANG_WARN_UNREACHABLE_CODE = YES`
- `GCC_WARN_UNUSED_FUNCTION = YES`
- `GCC_WARN_UNUSED_VARIABLE = YES`

## Common Dead Code Patterns

### Unused Imports
```swift
// Before
import Foundation
import UIKit        // Unused - only using SwiftUI
import SwiftUI
import Combine      // Unused - using async/await

// After
import Foundation
import SwiftUI
```

### Unused Protocol Conformances
```swift
// Before - if Equatable is never used
struct Item: Identifiable, Equatable, Hashable {
    let id: UUID
    let name: String
}

// After - keep only what's needed
struct Item: Identifiable {
    let id: UUID
    let name: String
}
```

### Dead Feature Flags
```swift
// Before
struct FeatureFlags {
    static let newOnboarding = true      // Always true, remove flag
    static let legacyAPI = false         // Always false, remove code
    static let experimentalFeature = true
}

// After
struct FeatureFlags {
    static let experimentalFeature = true
}
// Also remove all code guarded by legacyAPI
```

### Unused Extensions
```swift
// Before - extension with no usages
extension String {
    func toSnakeCase() -> String {
        // Complex implementation never called
    }
}

// After - delete entire extension
```

## Analysis Process

```bash
# 1. Run Periphery scan
periphery scan \
  --project MyApp.xcodeproj \
  --schemes MyApp \
  --targets MyApp \
  --format json > .reports/periphery-results.json

# 2. Run tests to establish baseline
xcodebuild test \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  2>&1 | xcbeautify

# 3. For each finding, verify removal is safe
# - Check if symbol is used via reflection
# - Check if symbol is used by Storyboards/XIBs (if any)
# - Check if symbol is part of public API

# 4. Remove dead code incrementally
# - Remove one item
# - Run tests
# - Commit if tests pass
# - Rollback if tests fail
```

## Report Format

```markdown
# Dead Code Analysis Report

Generated: 2024-01-15 10:30:00

## Summary
- Total findings: 23
- Safe to remove: 18
- Needs review: 4
- Keep (false positive): 1

## Safe to Remove

### Unused Classes (5)
| File | Line | Symbol | Confidence |
|------|------|--------|------------|
| Services/LegacyAPI.swift | 1 | `class LegacyAPI` | HIGH |
| Utils/DeprecatedHelper.swift | 1 | `class DeprecatedHelper` | HIGH |

### Unused Functions (8)
| File | Line | Symbol | Confidence |
|------|------|--------|------------|
| Extensions/String+Utils.swift | 42 | `func toSnakeCase()` | HIGH |

### Unused Imports (5)
| File | Line | Import | Confidence |
|------|------|--------|------------|
| Views/HomeView.swift | 3 | `import Combine` | HIGH |

## Needs Review

### Potentially Used via Reflection
| File | Line | Symbol | Reason |
|------|------|--------|--------|
| Models/Config.swift | 15 | `struct RemoteConfig` | May be decoded from JSON |

## Estimated Impact
- Lines removed: ~450
- Files deleted: 3
- Build time improvement: ~5%
```

## Verification Commands

```bash
# Run full test suite
xcodebuild test \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16'

# Build release to catch any issues
xcodebuild build \
  -scheme MyApp \
  -configuration Release \
  -destination 'generic/platform=iOS'

# Check for runtime crashes (manual)
# Run app and exercise key flows
```

## Best Practices

**DO:**
- ✅ Run tests before and after each removal
- ✅ Remove dead code incrementally (one change per commit)
- ✅ Document why code was removed in commit message
- ✅ Check for reflection/dynamic usage before removing
- ✅ Keep public API changes separate from internal cleanup

**DON'T:**
- ❌ Remove code without running tests
- ❌ Remove multiple unrelated items in one commit
- ❌ Remove protocol conformances without checking usages
- ❌ Remove `@objc` marked code without checking ObjC usage
- ❌ Remove code that might be used by Interface Builder

## False Positive Patterns

Watch for these patterns that may appear unused but are actually used:

```swift
// Used by SwiftUI previews
#Preview {
    MyView()
}

// Used by Codable
struct Response: Codable {
    let unusedField: String  // May be in API response
}

// Used by @objc selectors
@objc private func buttonTapped() { }

// Used by property wrappers
@AppStorage("key") var setting: Bool

// Used via reflection (rare but possible)
Mirror(reflecting: self)
```

Never delete code without running tests first!
