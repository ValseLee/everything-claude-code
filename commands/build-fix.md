# Build and Fix

Incrementally fix Swift compilation and build errors:

1. Run build:
   ```bash
   xcodebuild build \
     -scheme MyApp \
     -destination 'platform=iOS Simulator,name=iPhone 16' \
     2>&1 | xcbeautify
   ```

   Or for SPM packages:
   ```bash
   swift build
   ```

2. Parse error output:
   - Group by file
   - Sort by severity (error > warning)
   - Identify dependency chains

3. For each error:
   - Show error context (5 lines before/after)
   - Explain the issue
   - Propose fix
   - Apply fix
   - Re-run build
   - Verify error resolved

4. Stop if:
   - Fix introduces new errors
   - Same error persists after 3 attempts
   - User requests pause

5. Show summary:
   - Errors fixed
   - Errors remaining
   - Warnings to address

## Common Swift Build Errors

### Type Mismatch
```swift
// Error: Cannot convert value of type 'String' to expected argument type 'Int'
let count: Int = "5"

// Fix: Use proper type or conversion
let count: Int = Int("5") ?? 0
```

### Missing Conformance
```swift
// Error: Type 'MyModel' does not conform to protocol 'Sendable'
struct MyModel {
    var mutableState: [String]
}

// Fix: Make immutable or add @unchecked Sendable with care
struct MyModel: Sendable {
    let immutableState: [String]
}
```

### Actor Isolation
```swift
// Error: Main actor-isolated property 'items' can not be mutated from a non-isolated context
@MainActor
class ViewModel {
    var items: [Item] = []
}

// Fix: Ensure caller is also on MainActor or use Task { @MainActor in }
```

### Missing Import
```swift
// Error: Cannot find type 'View' in scope

// Fix: Add import
import SwiftUI
```

### Optional Handling
```swift
// Error: Value of optional type 'String?' must be unwrapped

// Fix: Use optional binding
if let value = optionalString {
    print(value)
}
```

## SPM-Specific Issues

### Dependency Resolution
```bash
# Error: Dependencies could not be resolved
swift package resolve

# If persists, clean and retry
swift package reset
swift package resolve
```

### Version Conflicts
```swift
// Package.swift
// Error: version solving failed

// Fix: Check version constraints are compatible
dependencies: [
    .package(url: "...", from: "2.0.0"),  // Ensure versions align
]
```

## Xcode Build Settings Issues

### Code Signing
```bash
# Error: Signing requires a development team

# Fix: Set team in project settings or via xcodebuild
xcodebuild -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO
```

### Swift Version
```bash
# Error: Module compiled with Swift X.X cannot be imported by Swift Y.Y

# Fix: Clean derived data and rebuild
rm -rf ~/Library/Developer/Xcode/DerivedData
xcodebuild clean build -scheme MyApp
```

## Quick Commands

```bash
# Build for simulator (no signing)
xcodebuild build \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16'

# Build release
xcodebuild build \
  -scheme MyApp \
  -configuration Release \
  -destination 'generic/platform=iOS'

# Clean build
xcodebuild clean build -scheme MyApp

# Show build settings
xcodebuild -scheme MyApp -showBuildSettings

# List available destinations
xcodebuild -scheme MyApp -showdestinations
```

Fix one error at a time for safety!
