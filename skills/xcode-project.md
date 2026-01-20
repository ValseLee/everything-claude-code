---
name: xcode-project
description: Xcode project configuration including workspace structure, build targets, Swift Package Manager, build settings, and dependency management.
---

# Xcode Project Configuration

Fundamentals of Xcode project structure, targets, and dependency management.

## Project vs Workspace

### .xcodeproj (Single Project)

```
MyApp/
├── MyApp.xcodeproj/
│   ├── project.pbxproj        # Project configuration
│   └── xcshareddata/
│       └── xcschemes/         # Shared schemes
├── MyApp/
│   ├── Sources/
│   └── Resources/
└── MyAppTests/
```

- Single app or framework
- Self-contained
- Dependencies via SPM or embedded frameworks

### .xcworkspace (Multi-Project)

```
MyProject/
├── MyProject.xcworkspace/
│   └── contents.xcworkspacedata
├── App/
│   └── App.xcodeproj/
├── Core/
│   └── Core.xcodeproj/
├── Features/
│   ├── FeatureA/
│   │   └── FeatureA.xcodeproj/
│   └── FeatureB/
│       └── FeatureB.xcodeproj/
└── Packages/
    └── SharedUI/
```

- Multiple projects in one workspace
- Framework-based modularization
- Mix of .xcodeproj and SPM packages
- Shared build settings across projects

### When to Use Each

```
Single Project (.xcodeproj):
- Small to medium apps
- Solo developer / small team
- Simple dependency graph

Workspace (.xcworkspace):
- Large apps with multiple modules
- Team of 5+ developers
- Need independent module compilation
- Mix of frameworks and packages
```

## Build Targets

### Target Types

```
App Target:
- Produces .app bundle
- Entry point for application
- Links frameworks and libraries

Framework Target:
- Produces .framework bundle
- Reusable code module
- Can be embedded or linked

Static Library:
- Produces .a file
- Linked at compile time
- No resources support

Unit Test Target:
- Tests for specific target
- Links test host app or framework

UI Test Target:
- End-to-end UI tests
- Runs against app target
```

### Creating Framework Target

```
1. File → New → Target
2. Select "Framework"
3. Configure:
   - Product Name: Core
   - Language: Swift
   - Include Tests: Yes

Project structure after:
MyApp/
├── MyApp/           # App target
├── Core/            # Framework target
│   ├── Sources/
│   └── Core.h       # Umbrella header
├── CoreTests/       # Framework tests
└── MyApp.xcodeproj
```

### Target Dependencies

```
In Build Phases → Dependencies:
- Add targets that must build before this target

In Build Phases → Link Binary With Libraries:
- Add frameworks to link

Example:
App Target
├── Depends on: Core.framework
├── Depends on: FeatureA.framework
└── Links: Core.framework, FeatureA.framework

FeatureA Target
├── Depends on: Core.framework
└── Links: Core.framework
```

## Swift Package Manager

### Package.swift Structure

```swift
// swift-tools-version: 5.9

import PackageDescription

let package = Package(
    name: "MyLibrary",
    platforms: [
        .iOS(.v17),
        .macOS(.v14)
    ],
    products: [
        // What the package exposes
        .library(
            name: "MyLibrary",
            targets: ["MyLibrary"]
        ),
        .library(
            name: "MyLibraryUI",
            targets: ["MyLibraryUI"]
        )
    ],
    dependencies: [
        // External dependencies
        .package(url: "https://github.com/apple/swift-algorithms", from: "1.0.0"),
        .package(url: "https://github.com/pointfreeco/swift-dependencies", from: "1.0.0")
    ],
    targets: [
        // Main target
        .target(
            name: "MyLibrary",
            dependencies: [
                .product(name: "Algorithms", package: "swift-algorithms")
            ],
            path: "Sources/MyLibrary"
        ),
        // UI target depending on main
        .target(
            name: "MyLibraryUI",
            dependencies: ["MyLibrary"],
            path: "Sources/MyLibraryUI"
        ),
        // Test target
        .testTarget(
            name: "MyLibraryTests",
            dependencies: [
                "MyLibrary",
                .product(name: "Dependencies", package: "swift-dependencies")
            ]
        )
    ]
)
```

### Local Package

```
MyProject/
├── MyApp/
│   └── MyApp.xcodeproj
└── Packages/
    └── SharedUI/
        ├── Package.swift
        └── Sources/
            └── SharedUI/

In Xcode:
1. File → Add Package Dependencies
2. Select "Add Local..."
3. Choose Packages/SharedUI
```

### Remote Package

```swift
// In Package.swift dependencies:
dependencies: [
    // Exact version
    .package(url: "https://github.com/user/repo", exact: "1.2.3"),

    // Version range
    .package(url: "https://github.com/user/repo", from: "1.0.0"),
    .package(url: "https://github.com/user/repo", "1.0.0"..<"2.0.0"),

    // Branch
    .package(url: "https://github.com/user/repo", branch: "main"),

    // Commit
    .package(url: "https://github.com/user/repo", revision: "abc123")
]
```

### Version Management Strategy

```
✅ RECOMMENDED:
- Use semantic versioning (from: "1.0.0")
- Lock to minor version for stability
- Update dependencies regularly

⚠️ USE WITH CAUTION:
- Branch dependencies (unstable)
- Exact versions (may miss patches)

❌ AVOID:
- Commit-based dependencies in production
- Wide version ranges (1.0.0..<3.0.0)
```

## Build Settings

### Build Configuration

```
Debug:
- SWIFT_OPTIMIZATION_LEVEL = -Onone
- DEBUG_INFORMATION_FORMAT = dwarf
- ENABLE_TESTABILITY = YES
- SWIFT_ACTIVE_COMPILATION_CONDITIONS = DEBUG

Release:
- SWIFT_OPTIMIZATION_LEVEL = -O
- DEBUG_INFORMATION_FORMAT = dwarf-with-dsym
- ENABLE_TESTABILITY = NO
- SWIFT_ACTIVE_COMPILATION_CONDITIONS = (empty)
```

### Settings Inheritance

```
Xcode Default
    ↓
Project Settings
    ↓
Target Settings
    ↓
xcconfig files (if used)

Higher levels override lower levels.
Use $(inherited) to include parent values.
```

### xcconfig Files

```
// Base.xcconfig
PRODUCT_BUNDLE_IDENTIFIER = com.company.app
SWIFT_VERSION = 5.9
IPHONEOS_DEPLOYMENT_TARGET = 17.0

// Debug.xcconfig
#include "Base.xcconfig"
SWIFT_OPTIMIZATION_LEVEL = -Onone
OTHER_SWIFT_FLAGS = $(inherited) -D DEBUG

// Release.xcconfig
#include "Base.xcconfig"
SWIFT_OPTIMIZATION_LEVEL = -O
SWIFT_COMPILATION_MODE = wholemodule

// Project structure
Config/
├── Base.xcconfig
├── Debug.xcconfig
└── Release.xcconfig
```

### Important Build Settings

```
// Swift
SWIFT_VERSION = 5.9
SWIFT_STRICT_CONCURRENCY = complete  // Swift 6 concurrency checking

// Deployment
IPHONEOS_DEPLOYMENT_TARGET = 17.0
TARGETED_DEVICE_FAMILY = 1,2  // 1=iPhone, 2=iPad

// Code Signing
CODE_SIGN_STYLE = Automatic
DEVELOPMENT_TEAM = XXXXXXXXXX

// Optimization
SWIFT_OPTIMIZATION_LEVEL = -O  // Release
SWIFT_COMPILATION_MODE = wholemodule  // Faster builds

// Linking
OTHER_LDFLAGS = $(inherited) -ObjC
DEAD_CODE_STRIPPING = YES
```

## Dependency Direction

### Unidirectional Dependencies (CRITICAL)

```
✅ CORRECT: Dependencies flow one direction

App
 │
 ├──► FeatureA ──► Domain ──► Core
 │
 ├──► FeatureB ──► Domain
 │
 └──► FeatureC ──► Domain

Each module only knows about modules to its right.
```

### Circular Dependency (FORBIDDEN)

```
❌ WRONG: Circular dependency

ModuleA ──► ModuleB
    ▲           │
    └───────────┘

This creates:
- Build failures
- Tight coupling
- Impossible to test in isolation
```

### Breaking Circular Dependencies

```swift
// ❌ BAD: Direct dependency both ways
// ModuleA
import ModuleB
class ServiceA {
    let serviceB: ServiceB  // Depends on ModuleB
}

// ModuleB
import ModuleA
class ServiceB {
    let serviceA: ServiceA  // Depends on ModuleA - CIRCULAR!
}

// ✅ GOOD: Use protocol (Dependency Inversion)
// Core module (no dependencies)
protocol ServiceBProtocol {
    func doWork()
}

// ModuleA
import Core
class ServiceA {
    let serviceB: ServiceBProtocol  // Depends on protocol, not concrete
}

// ModuleB
import Core
class ServiceB: ServiceBProtocol {
    func doWork() { }
}

// App (composition root)
let serviceB = ServiceB()
let serviceA = ServiceA(serviceB: serviceB)
```

### Interface Module Pattern

```
MyProject/
├── Core/                    # Shared utilities, no dependencies
├── DomainInterfaces/        # Protocols only
├── Domain/                  # Implements DomainInterfaces
├── DataInterfaces/          # Repository protocols
├── Data/                    # Implements DataInterfaces
├── FeatureA/                # Uses DomainInterfaces
└── App/                     # Wires everything together

Dependency Graph:
App ──► FeatureA ──► DomainInterfaces ◄── Domain
                          │                  │
                          ▼                  ▼
                    DataInterfaces ◄──── Data ──► Core
```

### Layered Architecture Dependencies

```
// Layer dependencies (from Clean Architecture)

Presentation Layer (Views, ViewModels)
         │
         ▼
Domain Layer (UseCases, Entities)
         │
         ▼
Data Layer (Repositories, DataSources)
         │
         ▼
Infrastructure (Network, Database, etc.)

// Each layer only depends on the layer below
// Never upward dependencies
```

## Build Optimization

### Parallel Builds

```
In Build Settings:
SWIFT_COMPILATION_MODE = wholemodule  // For release
SWIFT_COMPILATION_MODE = incremental  // For debug

In Scheme → Build:
☑️ Parallelize Build
☑️ Find Implicit Dependencies
```

### Module Stability

```
// For distributing binary frameworks
BUILD_LIBRARY_FOR_DISTRIBUTION = YES

// Generates .swiftinterface files
// Allows framework to work with different Swift versions
```

### Explicit Module Dependencies

```
In Build Settings:
SWIFT_ENABLE_EXPLICIT_MODULES = YES

// Improves build parallelism
// Better incremental builds
// Available in Xcode 15+
```

## Best Practices Checklist

- [ ] Use workspace for multi-module projects
- [ ] Separate framework targets for each module
- [ ] Dependencies flow in one direction only
- [ ] No circular dependencies between modules
- [ ] Use protocols (interfaces) to break dependencies
- [ ] Use xcconfig files for build settings
- [ ] Enable strict concurrency checking
- [ ] Use SPM for external dependencies
- [ ] Prefer local packages for internal modules
- [ ] Configure appropriate optimization levels per configuration
