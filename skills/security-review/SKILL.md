---
name: security-review
description: Use this skill when handling authentication, user data, secrets, network communication, or implementing sensitive features. Provides comprehensive iOS security checklist and patterns.
---

# iOS Security Review

This skill ensures all iOS code follows security best practices and identifies potential vulnerabilities.

## When to Activate

- Implementing authentication or authorization
- Storing sensitive data (tokens, credentials, user data)
- Making network requests
- Handling user input
- Implementing biometric authentication
- Working with Keychain
- Processing payments or sensitive transactions

## Security Checklist

### 1. Secrets Management

#### ❌ NEVER Hardcode Secrets
```swift
// DANGEROUS - Hardcoded secrets
let apiKey = "sk-proj-xxxxx"
let secretToken = "abc123secret"
```

#### ✅ Use Configuration Files (Not in Git)
```swift
// Config.swift (gitignored)
enum Config {
    static let apiKey: String = {
        guard let key = Bundle.main.object(forInfoDictionaryKey: "API_KEY") as? String else {
            fatalError("API_KEY not configured")
        }
        return key
    }()
}

// In Info.plist: API_KEY = $(API_KEY)
// Set in Xcode scheme or xcconfig
```

#### ✅ Use Keychain for Runtime Secrets
```swift
// Store secret securely
try KeychainManager.save(key: "authToken", value: token)

// Retrieve secret
let token = try KeychainManager.retrieve(key: "authToken")
```

#### Verification Steps
- [ ] No hardcoded API keys, tokens, or passwords in source
- [ ] Secrets in xcconfig files (gitignored)
- [ ] Runtime secrets stored in Keychain
- [ ] No secrets in git history
- [ ] Info.plist values populated at build time

### 2. Keychain Usage

#### Secure Keychain Manager
```swift
import Security

enum KeychainError: Error {
    case itemNotFound
    case duplicateItem
    case unexpectedStatus(OSStatus)
}

final class KeychainManager {

    static func save(key: String, value: String) throws {
        guard let data = value.data(using: .utf8) else {
            throw KeychainError.unexpectedStatus(errSecParam)
        }

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly
        ]

        // Delete existing item first
        SecItemDelete(query as CFDictionary)

        let status = SecItemAdd(query as CFDictionary, nil)

        guard status == errSecSuccess else {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    static func retrieve(key: String) throws -> String {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess,
              let data = result as? Data,
              let value = String(data: data, encoding: .utf8) else {
            throw KeychainError.itemNotFound
        }

        return value
    }

    static func delete(key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key
        ]

        let status = SecItemDelete(query as CFDictionary)

        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unexpectedStatus(status)
        }
    }
}
```

#### Keychain Access Levels
```swift
// Choose appropriate accessibility:

// After first unlock, device only (RECOMMENDED for most cases)
kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly

// When unlocked only (more secure, less convenient)
kSecAttrAccessibleWhenUnlockedThisDeviceOnly

// Always accessible (avoid - security risk)
kSecAttrAccessibleAlways  // ❌ AVOID

// With biometric protection
kSecAttrAccessControl: SecAccessControlCreateWithFlags(
    nil,
    kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
    .biometryCurrentSet,
    nil
)
```

#### Verification Steps
- [ ] Sensitive data stored in Keychain (not UserDefaults)
- [ ] Appropriate accessibility level chosen
- [ ] ThisDeviceOnly flag used (prevents iCloud sync)
- [ ] Biometric protection for high-security items
- [ ] Error handling for Keychain operations

### 3. App Transport Security (ATS)

#### Default ATS (Recommended)
```xml
<!-- Info.plist - Default is secure, no changes needed -->
<!-- ATS enforces HTTPS by default -->
```

#### Exception for Specific Domain (If Required)
```xml
<!-- Info.plist - Only if absolutely necessary -->
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSExceptionDomains</key>
    <dict>
        <key>legacy-api.example.com</key>
        <dict>
            <key>NSTemporaryExceptionAllowsInsecureHTTPLoads</key>
            <true/>
            <key>NSTemporaryExceptionMinimumTLSVersion</key>
            <string>TLSv1.2</string>
        </dict>
    </dict>
</dict>
```

#### ❌ NEVER Disable ATS Globally
```xml
<!-- DANGEROUS - Never do this in production -->
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>  <!-- ❌ NEVER -->
</dict>
```

#### Verification Steps
- [ ] ATS enabled (default)
- [ ] No NSAllowsArbitraryLoads = true
- [ ] Exceptions documented and justified
- [ ] TLS 1.2+ required
- [ ] Certificate validation not bypassed

### 4. Data Protection

#### File Protection Levels
```swift
// Write file with protection
let data = sensitiveData
let url = documentsDirectory.appendingPathComponent("sensitive.dat")

try data.write(
    to: url,
    options: [.completeFileProtection]  // Encrypted when locked
)

// Protection levels:
// .completeFileProtection - Encrypted when device locked (MOST SECURE)
// .completeFileProtectionUnlessOpen - Encrypted unless file is open
// .completeFileProtectionUntilFirstUserAuthentication - Encrypted until first unlock
// .noFileProtection - No encryption (AVOID for sensitive data)
```

#### Core Data Protection
```swift
let container = NSPersistentContainer(name: "Model")

// Add file protection
let storeDescription = container.persistentStoreDescriptions.first
storeDescription?.setOption(
    FileProtectionType.complete as NSObject,
    forKey: NSPersistentStoreFileProtectionKey
)
```

#### SwiftData Protection
```swift
let schema = Schema([User.self, Order.self])
let config = ModelConfiguration(
    schema: schema,
    url: storeURL,
    cloudKitDatabase: .none  // Disable CloudKit for sensitive data
)

// Note: SwiftData uses SQLite which respects file protection
// Set file protection on the store directory
```

#### Verification Steps
- [ ] Sensitive files use .completeFileProtection
- [ ] Database files are encrypted
- [ ] Temporary files cleaned up
- [ ] Cache contains no sensitive data
- [ ] CloudKit sync disabled for sensitive data

### 5. Secure Network Communication

#### URLSession with Certificate Pinning
```swift
final class PinnedSessionDelegate: NSObject, URLSessionDelegate {
    private let pinnedCertificates: [SecCertificate]

    init(certificates: [SecCertificate]) {
        self.pinnedCertificates = certificates
    }

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge
    ) async -> (URLSession.AuthChallengeDisposition, URLCredential?) {

        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            return (.cancelAuthenticationChallenge, nil)
        }

        // Evaluate server trust
        let policy = SecPolicyCreateSSL(true, challenge.protectionSpace.host as CFString)
        SecTrustSetPolicies(serverTrust, policy)

        var error: CFError?
        guard SecTrustEvaluateWithError(serverTrust, &error) else {
            return (.cancelAuthenticationChallenge, nil)
        }

        // Verify certificate is pinned
        guard let serverCertificate = SecTrustGetCertificateAtIndex(serverTrust, 0),
              pinnedCertificates.contains(where: {
                  CFEqual($0, serverCertificate)
              }) else {
            return (.cancelAuthenticationChallenge, nil)
        }

        return (.useCredential, URLCredential(trust: serverTrust))
    }
}
```

#### Secure Request Headers
```swift
func makeAuthenticatedRequest(to url: URL) async throws -> Data {
    var request = URLRequest(url: url)
    request.httpMethod = "GET"

    // Add authentication
    let token = try KeychainManager.retrieve(key: "authToken")
    request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")

    // Security headers
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.setValue("no-cache", forHTTPHeaderField: "Cache-Control")

    let (data, response) = try await URLSession.shared.data(for: request)

    guard let httpResponse = response as? HTTPURLResponse,
          200..<300 ~= httpResponse.statusCode else {
        throw NetworkError.requestFailed
    }

    return data
}
```

#### Verification Steps
- [ ] HTTPS used for all requests
- [ ] Certificate pinning for sensitive APIs
- [ ] Authentication tokens not logged
- [ ] Request/response not cached for sensitive data
- [ ] Timeout configured appropriately

### 6. Biometric Authentication

#### Face ID / Touch ID
```swift
import LocalAuthentication

final class BiometricAuthManager {

    enum BiometricError: Error {
        case notAvailable
        case notEnrolled
        case authenticationFailed
        case cancelled
    }

    func authenticate(reason: String) async throws {
        let context = LAContext()
        var error: NSError?

        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
            if let error {
                throw mapError(error)
            }
            throw BiometricError.notAvailable
        }

        do {
            let success = try await context.evaluatePolicy(
                .deviceOwnerAuthenticationWithBiometrics,
                localizedReason: reason
            )

            guard success else {
                throw BiometricError.authenticationFailed
            }
        } catch let error as LAError {
            throw mapLAError(error)
        }
    }

    private func mapLAError(_ error: LAError) -> BiometricError {
        switch error.code {
        case .biometryNotAvailable:
            return .notAvailable
        case .biometryNotEnrolled:
            return .notEnrolled
        case .userCancel, .appCancel:
            return .cancelled
        default:
            return .authenticationFailed
        }
    }
}
```

#### Info.plist Required Keys
```xml
<key>NSFaceIDUsageDescription</key>
<string>Authenticate to access your account</string>
```

#### Verification Steps
- [ ] Biometric used as second factor, not sole authentication
- [ ] Fallback to passcode available
- [ ] NSFaceIDUsageDescription provided
- [ ] Graceful handling of unavailable biometrics
- [ ] Context invalidated after use

### 7. Input Validation

#### User Input Sanitization
```swift
// ✅ GOOD: Validate and sanitize input
struct UserInput {
    let email: String
    let name: String

    init(email: String, name: String) throws {
        // Email validation
        let emailRegex = /^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$/
        guard email.wholeMatch(of: emailRegex) != nil else {
            throw ValidationError.invalidEmail
        }

        // Name validation
        let sanitizedName = name
            .trimmingCharacters(in: .whitespacesAndNewlines)
            .prefix(100)

        guard !sanitizedName.isEmpty else {
            throw ValidationError.emptyName
        }

        self.email = email
        self.name = String(sanitizedName)
    }
}
```

#### Prevent Injection in URL Construction
```swift
// ❌ BAD: String interpolation
let url = URL(string: "https://api.example.com/users/\(userId)")!

// ✅ GOOD: URLComponents
var components = URLComponents(string: "https://api.example.com/users")!
components.path.append("/\(userId)")

// For query parameters
components.queryItems = [
    URLQueryItem(name: "filter", value: userInput)  // Auto-escaped
]

guard let url = components.url else {
    throw URLError.invalidURL
}
```

#### Verification Steps
- [ ] All user inputs validated before use
- [ ] Length limits enforced
- [ ] Special characters sanitized
- [ ] URL parameters properly encoded
- [ ] No user input in format strings

### 8. Secure Data in Memory

#### Clear Sensitive Data
```swift
// Clear sensitive string from memory
func clearSensitiveString(_ string: inout String) {
    string.withUTF8 { buffer in
        guard let baseAddress = UnsafeMutableRawPointer(mutating: buffer.baseAddress) else {
            return
        }
        memset(baseAddress, 0, buffer.count)
    }
    string = ""
}

// Clear Data
func clearSensitiveData(_ data: inout Data) {
    data.withUnsafeMutableBytes { buffer in
        guard let baseAddress = buffer.baseAddress else { return }
        memset(baseAddress, 0, buffer.count)
    }
    data = Data()
}
```

#### Prevent Screenshots of Sensitive Content
```swift
// In SceneDelegate or SwiftUI App
func sceneWillResignActive(_ scene: UIScene) {
    // Add blur overlay when app goes to background
    let blurView = UIVisualEffectView(effect: UIBlurEffect(style: .regular))
    blurView.tag = 999
    blurView.frame = window?.bounds ?? .zero
    window?.addSubview(blurView)
}

func sceneDidBecomeActive(_ scene: UIScene) {
    // Remove blur overlay
    window?.viewWithTag(999)?.removeFromSuperview()
}
```

#### Verification Steps
- [ ] Sensitive data cleared after use
- [ ] Screenshots blocked for sensitive screens
- [ ] No sensitive data in logs
- [ ] Memory properly deallocated

### 9. Jailbreak Detection

#### Basic Jailbreak Detection
```swift
final class SecurityChecker {

    static var isJailbroken: Bool {
        #if targetEnvironment(simulator)
        return false
        #else
        // Check for common jailbreak files
        let jailbreakPaths = [
            "/Applications/Cydia.app",
            "/Library/MobileSubstrate/MobileSubstrate.dylib",
            "/bin/bash",
            "/usr/sbin/sshd",
            "/etc/apt",
            "/private/var/lib/apt/"
        ]

        for path in jailbreakPaths {
            if FileManager.default.fileExists(atPath: path) {
                return true
            }
        }

        // Check if app can write outside sandbox
        let testPath = "/private/jailbreak_test.txt"
        do {
            try "test".write(toFile: testPath, atomically: true, encoding: .utf8)
            try FileManager.default.removeItem(atPath: testPath)
            return true
        } catch {
            // Cannot write - not jailbroken
        }

        // Check for suspicious URL schemes
        if let url = URL(string: "cydia://"),
           UIApplication.shared.canOpenURL(url) {
            return true
        }

        return false
        #endif
    }
}

// Usage
if SecurityChecker.isJailbroken {
    // Warn user or restrict functionality
}
```

#### Verification Steps
- [ ] Jailbreak detection implemented (if required)
- [ ] Appropriate response to jailbreak
- [ ] Detection not easily bypassed
- [ ] Consider legitimate security research use

### 10. Logging and Debugging

#### Safe Logging
```swift
import os

final class SecureLogger {
    private let logger: Logger

    init(subsystem: String, category: String) {
        self.logger = Logger(subsystem: subsystem, category: category)
    }

    func info(_ message: String) {
        logger.info("\(message, privacy: .public)")
    }

    func error(_ message: String) {
        logger.error("\(message, privacy: .public)")
    }

    // Never log sensitive data
    func debug(_ message: String, sensitiveValue: String) {
        #if DEBUG
        logger.debug("\(message): \(sensitiveValue, privacy: .private)")
        #endif
    }
}

// ❌ NEVER log sensitive data in production
print("Token: \(authToken)")  // ❌
logger.info("Password: \(password)")  // ❌

// ✅ GOOD: Redact sensitive information
logger.info("User authenticated: \(userId)")
logger.info("Payment processed for last4: \(card.last4)")
```

#### Disable Debug Features in Release
```swift
#if DEBUG
// Debug-only code
func enableDebugFeatures() {
    // ...
}
#endif

// Or use compiler flags
#if !RELEASE
// Development features
#endif
```

#### Verification Steps
- [ ] No sensitive data in logs
- [ ] Debug code disabled in release
- [ ] os.Logger used with privacy annotations
- [ ] No print statements in production code
- [ ] Crash reports don't contain sensitive data

## Pre-Release Security Checklist

Before ANY App Store submission:

- [ ] **Secrets**: No hardcoded secrets in code
- [ ] **Keychain**: Sensitive data stored securely
- [ ] **ATS**: Enabled, no arbitrary loads
- [ ] **Data Protection**: Files encrypted at rest
- [ ] **Network**: HTTPS only, certificate pinning for sensitive APIs
- [ ] **Input Validation**: All user inputs validated
- [ ] **Biometrics**: Proper fallback and error handling
- [ ] **Memory**: Sensitive data cleared after use
- [ ] **Logging**: No sensitive data logged
- [ ] **Debug**: Debug features disabled in release
- [ ] **Entitlements**: Minimal required entitlements
- [ ] **Privacy**: Info.plist usage descriptions present
- [ ] **Dependencies**: Up to date, no known vulnerabilities

## Resources

- [Apple Security Documentation](https://developer.apple.com/documentation/security)
- [OWASP Mobile Security Guide](https://owasp.org/www-project-mobile-security/)
- [Apple App Store Review Guidelines - Security](https://developer.apple.com/app-store/review/guidelines/#safety)
- [iOS Security Guide](https://support.apple.com/guide/security/)

---

**Remember**: Security is not optional. One vulnerability can compromise user data and trust. When in doubt, err on the side of caution.
