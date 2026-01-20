# Project Guidelines Skill (Example)

This is an example of a project-specific skill. Use this as a template for your own iOS projects.

---

## When to Use

Reference this skill when working on the specific project it's designed for. Project skills contain:
- Architecture overview
- File structure
- Code patterns
- Testing requirements
- Build and deployment workflow

---

## Architecture Overview

**Tech Stack:**
- **UI Framework**: SwiftUI (iOS 18+)
- **Architecture**: Clean Architecture + MVVM
- **Data Persistence**: SwiftData
- **Networking**: URLSession with async/await
- **Dependency Injection**: Manual (Protocol-based)
- **Testing**: Swift Testing framework
- **CI/CD**: GitHub Actions + Fastlane

**Module Structure:**
```
┌─────────────────────────────────────────────────────────────┐
│                           App                               │
│  Main app target, composition root, entry point             │
└─────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
┌────────────────┐   ┌────────────────┐   ┌────────────────┐
│   FeatureHome  │   │  FeatureOrder  │   │ FeatureProfile │
│    (Module)    │   │    (Module)    │   │    (Module)    │
└───────┬────────┘   └───────┬────────┘   └───────┬────────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │     Domain     │
                    │  (UseCases,    │
                    │   Entities)    │
                    └───────┬────────┘
                            │
                ┌───────────┼───────────┐
                ▼           ▼           ▼
          ┌──────────┐ ┌──────────┐ ┌──────────┐
          │   Data   │ │ Network  │ │   Core   │
          │(SwiftData│ │(API      │ │(Shared   │
          │ Repos)   │ │ Client)  │ │ Utils)   │
          └──────────┘ └──────────┘ └──────────┘
```

---

## File Structure

```
MyApp/
├── App/
│   ├── MyApp.swift                 # @main entry point
│   ├── DependencyContainer.swift   # Composition root
│   └── AppConfiguration.swift      # App-wide config
│
├── Modules/
│   ├── Core/
│   │   ├── Extensions/
│   │   ├── Utilities/
│   │   └── Package.swift
│   │
│   ├── Domain/
│   │   ├── Entities/
│   │   ├── UseCases/
│   │   ├── Repositories/           # Protocols only
│   │   └── Package.swift
│   │
│   ├── Data/
│   │   ├── Repositories/           # Implementations
│   │   ├── DataSources/
│   │   ├── Models/                 # DTOs, SwiftData models
│   │   └── Package.swift
│   │
│   ├── Network/
│   │   ├── APIClient.swift
│   │   ├── Endpoints/
│   │   ├── DTOs/
│   │   └── Package.swift
│   │
│   └── Features/
│       ├── Home/
│       │   ├── Views/
│       │   │   ├── HomeView.swift
│       │   │   └── HomeView+ViewModel.swift
│       │   ├── Components/
│       │   └── Package.swift
│       │
│       ├── Order/
│       └── Profile/
│
├── Resources/
│   ├── Assets.xcassets
│   ├── Localizable.xcstrings
│   └── Info.plist
│
├── Tests/
│   ├── UnitTests/
│   ├── IntegrationTests/
│   └── UITests/
│
└── MyApp.xcworkspace
```

---

## Code Patterns

### View-ViewModel Pattern

```swift
// HomeView.swift
import SwiftUI

struct HomeView: View {
    @State private var viewModel: ViewModel

    init(dependencies: HomeDependencies) {
        _viewModel = State(initialValue: ViewModel(
            fetchItemsUseCase: dependencies.fetchItemsUseCase
        ))
    }

    var body: some View {
        content
            .task { await viewModel.loadItems() }
    }

    @ViewBuilder
    private var content: some View {
        switch viewModel.state {
        case .loading:
            ProgressView()
        case .loaded(let items):
            itemList(items)
        case .error(let message):
            errorView(message)
        }
    }
}

// HomeView+ViewModel.swift
extension HomeView {
    @Observable @MainActor
    final class ViewModel {
        private(set) var state: ViewState = .loading

        enum ViewState: Equatable {
            case loading
            case loaded([Item])
            case error(String)
        }

        private let fetchItemsUseCase: FetchItemsUseCase

        init(fetchItemsUseCase: FetchItemsUseCase) {
            self.fetchItemsUseCase = fetchItemsUseCase
        }

        func loadItems() async {
            state = .loading
            do {
                let items = try await fetchItemsUseCase.execute()
                state = .loaded(items)
            } catch {
                state = .error(error.localizedDescription)
            }
        }
    }
}
```

### Repository Pattern

```swift
// Domain/Repositories/ItemRepository.swift (Protocol)
protocol ItemRepository {
    func fetchAll() async throws -> [Item]
    func fetch(id: UUID) async throws -> Item
    func save(_ item: Item) async throws
    func delete(id: UUID) async throws
}

// Data/Repositories/ItemRepositoryImpl.swift (Implementation)
final class ItemRepositoryImpl: ItemRepository {
    private let networkService: NetworkService
    private let localDataSource: ItemLocalDataSource

    init(networkService: NetworkService, localDataSource: ItemLocalDataSource) {
        self.networkService = networkService
        self.localDataSource = localDataSource
    }

    func fetchAll() async throws -> [Item] {
        // Try local first
        let localItems = try await localDataSource.fetchAll()
        if !localItems.isEmpty {
            return localItems
        }

        // Fetch from network
        let dtos = try await networkService.request(endpoint: .items)
        let items = dtos.map { ItemMapper.toDomain($0) }

        // Cache locally
        try await localDataSource.saveAll(items)

        return items
    }
}
```

### Use Case Pattern

```swift
// Domain/UseCases/FetchItemsUseCase.swift
protocol FetchItemsUseCase {
    func execute() async throws -> [Item]
}

final class FetchItemsUseCaseImpl: FetchItemsUseCase {
    private let repository: ItemRepository

    init(repository: ItemRepository) {
        self.repository = repository
    }

    func execute() async throws -> [Item] {
        let items = try await repository.fetchAll()
        return items.filter { $0.isActive }.sorted { $0.createdAt > $1.createdAt }
    }
}
```

### API Client

```swift
// Network/APIClient.swift
actor APIClient {
    private let baseURL: URL
    private let session: URLSession

    init(baseURL: URL, session: URLSession = .shared) {
        self.baseURL = baseURL
        self.session = session
    }

    func request<T: Decodable>(endpoint: Endpoint) async throws -> T {
        let request = try endpoint.makeRequest(baseURL: baseURL)

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw APIError.invalidResponse
        }

        guard 200..<300 ~= httpResponse.statusCode else {
            throw APIError.httpError(httpResponse.statusCode)
        }

        return try JSONDecoder().decode(T.self, from: data)
    }
}
```

---

## Testing Requirements

### Unit Tests (Swift Testing)

```bash
# Run all tests
xcodebuild test -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 15'

# Run with coverage
xcodebuild test -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 15' \
  -enableCodeCoverage YES
```

**Test structure:**
```swift
import Testing
@testable import Domain

@Suite("FetchItemsUseCase")
struct FetchItemsUseCaseTests {
    @Test("returns filtered and sorted items")
    func fetchItems() async throws {
        let mockRepository = MockItemRepository()
        mockRepository.items = [
            Item(id: UUID(), name: "B", isActive: true, createdAt: .now),
            Item(id: UUID(), name: "A", isActive: false, createdAt: .now),
            Item(id: UUID(), name: "C", isActive: true, createdAt: .now.addingTimeInterval(-100))
        ]

        let useCase = FetchItemsUseCaseImpl(repository: mockRepository)
        let result = try await useCase.execute()

        #expect(result.count == 2)  // Only active items
        #expect(result[0].name == "B")  // Sorted by date desc
    }
}
```

### Coverage Requirements

- **Minimum**: 80% line coverage
- **Domain layer**: 90%+ coverage
- **Critical paths**: 100% coverage

---

## Build and Deployment

### Pre-Release Checklist

- [ ] All tests passing locally
- [ ] Build succeeds for Release configuration
- [ ] No hardcoded secrets in code
- [ ] Info.plist privacy descriptions complete
- [ ] Version and build number updated
- [ ] Changelog updated

### Build Commands

```bash
# Build for testing
xcodebuild build-for-testing \
  -scheme MyApp \
  -destination 'platform=iOS Simulator,name=iPhone 15'

# Archive for distribution
xcodebuild archive \
  -scheme MyApp \
  -archivePath ./build/MyApp.xcarchive \
  -destination 'generic/platform=iOS'

# Export IPA
xcodebuild -exportArchive \
  -archivePath ./build/MyApp.xcarchive \
  -exportPath ./build \
  -exportOptionsPlist ExportOptions.plist
```

### Environment Configuration

```swift
// AppConfiguration.swift
enum AppConfiguration {
    enum Environment {
        case debug
        case staging
        case production
    }

    static var current: Environment {
        #if DEBUG
        return .debug
        #elseif STAGING
        return .staging
        #else
        return .production
        #endif
    }

    static var apiBaseURL: URL {
        switch current {
        case .debug:
            return URL(string: "https://api-dev.example.com")!
        case .staging:
            return URL(string: "https://api-staging.example.com")!
        case .production:
            return URL(string: "https://api.example.com")!
        }
    }
}
```

---

## Critical Rules

1. **Pure Functions** - minimize side effects
2. **No Forced Unwrap** - never use `!` except for compile-time guaranteed values
3. **Stateless Views** - all state in ViewModel
4. **TDD** - write tests before implementation
5. **80% coverage** minimum
6. **Many small files** - 200-400 lines typical, 800 max
7. **No print statements** in production code
8. **Proper error handling** with typed errors
9. **Keychain for secrets** - never UserDefaults

---

## Related Skills

- `swift-coding-standards.md` - Swift coding best practices
- `swift-concurrency.md` - Async/await and actor patterns
- `swiftui-patterns.md` - SwiftUI component patterns
- `swiftdata.md` - Data persistence patterns
- `ios-architecture.md` - Clean and modular architecture
- `xcode-project.md` - Project configuration
- `tdd-workflow/` - Test-driven development methodology
- `security-review/` - iOS security checklist
