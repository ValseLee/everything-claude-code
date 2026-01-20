# Security Guidelines

## Mandatory Security Checks

Before ANY commit:
- [ ] No hardcoded secrets (API keys, passwords, tokens)
- [ ] Sensitive data stored in Keychain (not UserDefaults)
- [ ] ATS enabled (HTTPS only)
- [ ] All user inputs validated
- [ ] File protection enabled for sensitive data
- [ ] No sensitive data in logs
- [ ] Biometric authentication uses proper fallback

## Secret Management

```swift
// ❌ NEVER: Hardcoded secrets
let apiKey = "sk-proj-xxxxx"

// ✅ GOOD: Configuration file (gitignored)
enum Config {
    static let apiKey: String = {
        guard let key = Bundle.main.object(forInfoDictionaryKey: "API_KEY") as? String else {
            fatalError("API_KEY not configured")
        }
        return key
    }()
}

// ✅ GOOD: Keychain for runtime secrets
let token = try KeychainManager.retrieve(key: "authToken")
```

## Keychain Usage

```swift
// ✅ GOOD: Store sensitive data securely
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrAccount as String: "authToken",
    kSecValueData as String: tokenData,
    kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
]
SecItemAdd(query as CFDictionary, nil)

// ❌ NEVER: Store secrets in UserDefaults
UserDefaults.standard.set(token, forKey: "authToken")
```

## App Transport Security

```xml
<!-- Default ATS is secure - no changes needed -->
<!-- NEVER disable ATS globally: -->
<!-- NSAllowsArbitraryLoads = true  ❌ NEVER -->
```

## Security Response Protocol

If security issue found:
1. STOP immediately
2. Use **security-reviewer** agent
3. Fix CRITICAL issues before continuing
4. Rotate any exposed secrets
5. Review entire codebase for similar issues
