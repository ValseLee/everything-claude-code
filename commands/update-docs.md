# Update Documentation

Sync documentation from source-of-truth for iOS/Swift projects:

1. Read Package.swift or .xcodeproj for project configuration
   - Generate dependencies reference table
   - Document build configurations

2. Read configuration files
   - `.xcconfig` files for build settings
   - `Info.plist` for app configuration
   - Environment-specific settings

3. Generate `docs/CONTRIB.md` with:
   - Development environment setup
   - Build commands
   - Testing procedures
   - Code style guidelines

4. Generate `docs/RUNBOOK.md` with:
   - Build and release procedures
   - TestFlight deployment
   - App Store submission
   - Common issues and fixes

5. Identify obsolete documentation:
   - Find docs not modified in 90+ days
   - List for manual review

6. Show diff summary

## Source Files to Analyze

### Package.swift (SPM)
```swift
// Extract dependencies
let package = Package(
    dependencies: [
        .package(url: "https://github.com/...", from: "1.0.0"),
    ]
)
```

### Project.pbxproj
```bash
# List targets
xcodebuild -list -project MyApp.xcodeproj

# List schemes
xcodebuild -list -project MyApp.xcodeproj -json
```

### Info.plist
```bash
# Read plist values
/usr/libexec/PlistBuddy -c "Print :CFBundleVersion" MyApp/Info.plist
```

## Generated Documentation

### docs/CONTRIB.md

```markdown
# Contributing Guide

## Prerequisites

- Xcode 16.0+
- iOS 18.0+ Simulator
- SwiftLint (optional): `brew install swiftlint`

## Getting Started

1. Clone the repository
   ```bash
   git clone https://github.com/org/myapp.git
   cd myapp
   ```

2. Open in Xcode
   ```bash
   open MyApp.xcodeproj
   # or for workspace
   open MyApp.xcworkspace
   ```

3. Select scheme and destination
   - Scheme: `MyApp`
   - Destination: `iPhone 16`

4. Build and run (⌘R)

## Build Commands

| Command | Description |
|---------|-------------|
| `xcodebuild build -scheme MyApp` | Build debug |
| `xcodebuild build -scheme MyApp -configuration Release` | Build release |
| `xcodebuild test -scheme MyApp` | Run tests |
| `xcodebuild test -scheme MyApp -enableCodeCoverage YES` | Run tests with coverage |

## Code Style

- Follow Swift API Design Guidelines
- Use SwiftLint for consistency
- Run `swiftlint` before committing

## Testing

### Unit Tests
```bash
xcodebuild test \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  -only-testing:MyAppTests
```

### UI Tests
```bash
xcodebuild test \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16' \
  -only-testing:MyAppUITests
```

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| swift-collections | 1.0.0 | Data structures |
| swift-algorithms | 1.0.0 | Sequence algorithms |

## Architecture

See `docs/ARCHITECTURE.md` for detailed architecture documentation.
```

### docs/RUNBOOK.md

```markdown
# Operations Runbook

## Build & Release

### Debug Build
```bash
xcodebuild build \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 16'
```

### Release Build
```bash
xcodebuild build \
  -scheme MyApp \
  -configuration Release \
  -destination 'generic/platform=iOS' \
  -archivePath build/MyApp.xcarchive \
  archive
```

### Export IPA
```bash
xcodebuild -exportArchive \
  -archivePath build/MyApp.xcarchive \
  -exportOptionsPlist ExportOptions.plist \
  -exportPath build/
```

## TestFlight Deployment

1. Archive the app
   ```bash
   xcodebuild archive \
     -scheme MyApp \
     -configuration Release \
     -archivePath build/MyApp.xcarchive
   ```

2. Export for App Store
   ```bash
   xcodebuild -exportArchive \
     -archivePath build/MyApp.xcarchive \
     -exportOptionsPlist ExportOptions-AppStore.plist \
     -exportPath build/
   ```

3. Upload to App Store Connect
   ```bash
   xcrun altool --upload-app \
     -f build/MyApp.ipa \
     -t ios \
     -u "apple-id@example.com" \
     -p "@keychain:AC_PASSWORD"
   ```

## Common Issues

### Code Signing Failed
```bash
# Reset signing
xcodebuild -scheme MyApp \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO \
  -destination 'platform=iOS Simulator,name=iPhone 16'
```

### Derived Data Issues
```bash
# Clean derived data
rm -rf ~/Library/Developer/Xcode/DerivedData
xcodebuild clean -scheme MyApp
```

### Simulator Issues
```bash
# Reset simulator
xcrun simctl shutdown all
xcrun simctl erase all
```

### Package Resolution Failed
```bash
# Reset package cache
rm -rf .build
rm Package.resolved
xcodebuild -resolvePackageDependencies
```

## Monitoring

### Crash Reports
- Xcode Organizer: Window > Organizer > Crashes
- App Store Connect: App Analytics > Crashes

### Performance
- Xcode Instruments: Product > Profile
- MetricKit integration for production metrics

## Rollback Procedure

1. In App Store Connect, select previous build
2. Submit for review with expedited request
3. Or use phased release pause

## Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `API_BASE_URL` | API endpoint | `https://api.example.com` |
| `ANALYTICS_KEY` | Analytics service | `ak_xxx` |

Configure in scheme: Product > Scheme > Edit Scheme > Arguments
```

## Automation Script

```bash
#!/bin/bash
# update-docs.sh

# Extract project info
PROJECT_NAME=$(xcodebuild -list -json | jq -r '.project.name')
SCHEMES=$(xcodebuild -list -json | jq -r '.project.schemes[]')

# Generate dependencies table from Package.swift
if [ -f "Package.swift" ]; then
  echo "## Dependencies"
  swift package show-dependencies --format json | jq -r '.dependencies[] | "| \(.name) | \(.version) |"'
fi

# Check for outdated docs
find docs -name "*.md" -mtime +90 -exec echo "STALE: {}" \;
```

Single source of truth: Package.swift, project settings, and .xcconfig files.
