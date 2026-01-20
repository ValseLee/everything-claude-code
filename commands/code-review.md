# Code Review

Comprehensive security and quality review of uncommitted changes for iOS/Swift projects:

1. Get changed files: git diff --name-only HEAD

2. For each changed file, check for:

**Security Issues (CRITICAL):**
- Hardcoded credentials, API keys, tokens
- Sensitive data not stored in Keychain
- Missing App Transport Security exceptions justification
- Insecure data storage (UserDefaults for sensitive data)
- Missing Data Protection class
- Logging sensitive information
- Insecure URL schemes
- Missing input validation
- Force unwrap on external data

**Code Quality (HIGH):**
- Functions > 50 lines
- Files > 500 lines
- Nesting depth > 4 levels
- Missing error handling (try without catch context)
- print() statements in production code
- TODO/FIXME comments
- Missing documentation for public APIs
- Force unwrap (!) without compile-time guarantee
- Implicit unwrap (IUO) in new code

**Concurrency Issues (HIGH):**
- Data races (shared mutable state without actor)
- Missing @MainActor for UI updates
- Blocking main thread with synchronous calls
- Missing Sendable conformance for cross-actor data
- Unsafe continuation usage

**SwiftUI Patterns (MEDIUM):**
- View with business logic (should be in ViewModel)
- Missing @Observable for ViewModel
- State in View that should be in ViewModel
- Heavy computation in body property
- Missing .task for async work on appear

**Best Practices (MEDIUM):**
- Mutation patterns (prefer let over var)
- Missing tests for new code
- Accessibility issues (missing labels, traits)
- Missing localization for user-facing strings
- Deprecated API usage

3. Generate report with:
   - Severity: CRITICAL, HIGH, MEDIUM, LOW
   - File location and line numbers
   - Issue description
   - Suggested fix

4. Block commit if CRITICAL or HIGH issues found

## Swift-Specific Checks

```swift
// BAD: Hardcoded secret
let apiKey = "sk-1234567890"  // CRITICAL: Use Keychain

// BAD: Sensitive data in UserDefaults
UserDefaults.standard.set(password, forKey: "password")  // CRITICAL: Use Keychain

// BAD: Force unwrap external data
let user = response.user!  // HIGH: Use guard let or if let

// BAD: Missing MainActor
class ViewModel {  // HIGH: Add @MainActor
    var items: [Item] = []
}

// BAD: Business logic in View
var body: some View {
    let filtered = items.filter { $0.isActive }  // MEDIUM: Move to ViewModel
}

// BAD: Print in production
print("User logged in: \(user.email)")  // HIGH: Remove or use proper logging
```

## Auto-Fix Suggestions

For common issues, provide copyable fixes:

```swift
// Issue: Force unwrap on optional
// Before:
let name = user.name!

// After:
guard let name = user.name else {
    // Handle nil case
    return
}
```

Never approve code with security vulnerabilities!
