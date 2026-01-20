# Update Codemaps

Analyze the iOS codebase structure and update architecture documentation:

1. Scan all Swift source files for imports, types, and dependencies
2. Generate token-lean codemaps in the following format:
   - `codemaps/architecture.md` - Overall architecture (modules, layers)
   - `codemaps/features.md` - Feature modules structure
   - `codemaps/domain.md` - Domain models and services
   - `codemaps/data.md` - Data layer (repositories, persistence)

3. Calculate diff percentage from previous version
4. If changes > 30%, request user approval before updating
5. Add freshness timestamp to each codemap
6. Save reports to `.reports/codemap-diff.txt`

## Analysis Commands

```bash
# List all Swift files
find . -name "*.swift" -not -path "*/.*" -not -path "*/.build/*"

# Count lines per module
find Sources -name "*.swift" -exec wc -l {} + | sort -n

# Find all imports
grep -r "^import " Sources --include="*.swift" | sort | uniq -c | sort -rn

# Find all public types
grep -r "public \(class\|struct\|enum\|protocol\|actor\)" Sources --include="*.swift"

# Find dependencies between modules
grep -r "@testable import\|import " Tests --include="*.swift" | grep -v "Testing\|XCTest"
```

## Codemap Format

### architecture.md

```markdown
# Architecture Overview

Last updated: 2024-01-15 10:30:00

## Module Structure

```
MyApp/
├── Sources/
│   ├── App/                 # App entry, configuration
│   │   ├── MyApp.swift
│   │   └── AppDelegate.swift
│   │
│   ├── Features/            # Feature modules
│   │   ├── Home/
│   │   ├── Portfolio/
│   │   ├── Settings/
│   │   └── Onboarding/
│   │
│   ├── Domain/              # Business logic
│   │   ├── Models/
│   │   ├── Services/
│   │   └── UseCases/
│   │
│   ├── Data/                # Data layer
│   │   ├── Repositories/
│   │   ├── Network/
│   │   └── Persistence/
│   │
│   └── Core/                # Shared utilities
│       ├── Extensions/
│       ├── Components/
│       └── Utils/
│
└── Tests/
    ├── UnitTests/
    ├── IntegrationTests/
    └── UITests/
```

## Layer Dependencies

```
┌─────────────────────────────────────┐
│           Presentation              │
│  (Features: Views + ViewModels)     │
└─────────────────┬───────────────────┘
                  │ depends on
                  ▼
┌─────────────────────────────────────┐
│             Domain                  │
│   (Models, Services, UseCases)      │
└─────────────────┬───────────────────┘
                  │ depends on
                  ▼
┌─────────────────────────────────────┐
│              Data                   │
│  (Repositories, Network, Storage)   │
└─────────────────────────────────────┘
```

## Module Metrics

| Module | Files | Lines | Public Types |
|--------|-------|-------|--------------|
| Features/Home | 8 | 450 | 4 |
| Features/Portfolio | 12 | 820 | 6 |
| Domain/Models | 15 | 380 | 15 |
| Domain/Services | 8 | 520 | 8 |
| Data/Network | 6 | 340 | 4 |
```

### features.md

```markdown
# Feature Modules

Last updated: 2024-01-15 10:30:00

## Home Feature

```
Features/Home/
├── HomeView.swift           # Main view
├── HomeViewModel.swift      # @Observable ViewModel
├── Components/
│   ├── MarketCard.swift
│   └── QuickActions.swift
└── HomeCoordinator.swift    # Navigation
```

**Dependencies:** Domain.MarketService, Domain.UserService

## Portfolio Feature

```
Features/Portfolio/
├── PortfolioView.swift
├── PortfolioViewModel.swift
├── PortfolioDetailView.swift
├── PortfolioDetailViewModel.swift
├── Components/
│   ├── AssetRow.swift
│   ├── PerformanceChart.swift
│   └── AllocationPie.swift
└── PortfolioCoordinator.swift
```

**Dependencies:** Domain.PortfolioService, Domain.AssetService, Charts
```

### domain.md

```markdown
# Domain Layer

Last updated: 2024-01-15 10:30:00

## Models

| Model | Properties | Conformances |
|-------|------------|--------------|
| User | id, name, email, createdAt | Identifiable, Codable, Sendable |
| Portfolio | id, name, assets, value | Identifiable, Codable, Sendable |
| Asset | id, symbol, quantity, price | Identifiable, Codable, Sendable |

## Services

| Service | Protocol | Methods |
|---------|----------|---------|
| AuthService | AuthServiceProtocol | signIn, signOut, refreshToken |
| PortfolioService | PortfolioServiceProtocol | fetch, create, update, delete |
| MarketService | MarketServiceProtocol | getQuotes, search, subscribe |

## Actors

| Actor | Purpose |
|-------|---------|
| CacheActor | Thread-safe caching |
| SyncActor | Background sync coordination |
```

## Generating Codemaps

```bash
# 1. Collect module information
for dir in Sources/*/; do
  module=$(basename "$dir")
  files=$(find "$dir" -name "*.swift" | wc -l)
  lines=$(find "$dir" -name "*.swift" -exec cat {} + | wc -l)
  echo "$module: $files files, $lines lines"
done

# 2. Find public API surface
grep -rn "public \(func\|var\|let\|class\|struct\|enum\|protocol\)" Sources

# 3. Map dependencies
for file in Sources/**/*.swift; do
  echo "=== $file ==="
  grep "^import " "$file" | grep -v "Foundation\|SwiftUI\|Combine"
done

# 4. Generate Mermaid diagram
echo "graph TD"
for module in Features Domain Data Core; do
  imports=$(grep -rh "^import " "Sources/$module" | sort | uniq)
  # Parse and output edges
done
```

## Diff Tracking

When updating codemaps, track changes:

```markdown
## Changes Since Last Update

### Added
- Features/Onboarding module (4 files)
- Domain/Services/SyncService.swift

### Modified
- Domain/Models/User.swift (added preferences property)
- Data/Network/APIClient.swift (added retry logic)

### Removed
- Features/Legacy/* (deprecated feature)

### Metrics Delta
- Total files: 45 → 48 (+3)
- Total lines: 4,200 → 4,580 (+380)
- Test coverage: 78% → 82% (+4%)
```

Use Swift and Xcode tools for analysis. Focus on high-level structure, not implementation details.
