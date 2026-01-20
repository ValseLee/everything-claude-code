---
name: ios-architecture
description: iOS application architecture including Clean Architecture, Modular Architecture, dependency management, and practical implementation patterns.
---

# iOS Architecture

Application architecture patterns for scalable, testable iOS apps.

**Prerequisites**: Familiarity with [xcode-project.md](./xcode-project.md) (targets, dependencies, SPM).

## Clean Architecture

### Layer Overview

```
┌─────────────────────────────────────────────────────┐
│                    Presentation                      │
│              (Views, ViewModels)                     │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                      Domain                          │
│           (UseCases, Entities, Protocols)           │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                       Data                           │
│    (Repositories, DataSources, DTOs, Mappers)       │
└─────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────┐
│                   Infrastructure                     │
│         (Network, Database, External SDKs)          │
└─────────────────────────────────────────────────────┘
```

### The Dependency Rule

```
Dependencies point INWARD only.

Outer layers know about inner layers.
Inner layers know NOTHING about outer layers.

Presentation → Domain → (nothing)
Data → Domain → (nothing)
Infrastructure → Data → Domain

Domain layer has ZERO external dependencies.
```

### Domain Layer

```swift
// MARK: - Entities (Business Objects)

struct User: Identifiable, Equatable {
    let id: UUID
    var name: String
    var email: String
    var isPremium: Bool
}

struct Order: Identifiable, Equatable {
    let id: UUID
    let userId: UUID
    var items: [OrderItem]
    var status: OrderStatus

    var total: Decimal {
        items.reduce(0) { $0 + $1.price * Decimal($1.quantity) }
    }
}

enum OrderStatus {
    case pending, confirmed, shipped, delivered, cancelled
}
```

```swift
// MARK: - Repository Protocols (Interfaces)

protocol UserRepository {
    func fetchUser(id: UUID) async throws -> User
    func fetchAllUsers() async throws -> [User]
    func save(_ user: User) async throws
    func delete(id: UUID) async throws
}

protocol OrderRepository {
    func fetchOrders(for userId: UUID) async throws -> [Order]
    func createOrder(_ order: Order) async throws -> Order
    func updateStatus(_ orderId: UUID, status: OrderStatus) async throws
}
```

```swift
// MARK: - Use Cases (Business Logic)

protocol FetchUserUseCase {
    func execute(id: UUID) async throws -> User
}

final class FetchUserUseCaseImpl: FetchUserUseCase {
    private let userRepository: UserRepository

    init(userRepository: UserRepository) {
        self.userRepository = userRepository
    }

    func execute(id: UUID) async throws -> User {
        try await userRepository.fetchUser(id: id)
    }
}

// More complex use case with business logic
protocol CreateOrderUseCase {
    func execute(userId: UUID, items: [OrderItem]) async throws -> Order
}

final class CreateOrderUseCaseImpl: CreateOrderUseCase {
    private let orderRepository: OrderRepository
    private let userRepository: UserRepository

    init(orderRepository: OrderRepository, userRepository: UserRepository) {
        self.orderRepository = orderRepository
        self.userRepository = userRepository
    }

    func execute(userId: UUID, items: [OrderItem]) async throws -> Order {
        // Business rule: validate user exists and is active
        let user = try await userRepository.fetchUser(id: userId)

        // Business rule: premium users get discount
        let processedItems = user.isPremium
            ? items.map { $0.withDiscount(0.1) }
            : items

        let order = Order(
            id: UUID(),
            userId: userId,
            items: processedItems,
            status: .pending
        )

        return try await orderRepository.createOrder(order)
    }
}
```

### Data Layer

```swift
// MARK: - DTOs (Data Transfer Objects)

struct UserDTO: Codable {
    let id: String
    let name: String
    let email: String
    let is_premium: Bool  // API naming convention
}

struct OrderDTO: Codable {
    let id: String
    let user_id: String
    let items: [OrderItemDTO]
    let status: String
}
```

```swift
// MARK: - Mappers

enum UserMapper {
    static func toDomain(_ dto: UserDTO) -> User? {
        guard let id = UUID(uuidString: dto.id) else { return nil }
        return User(
            id: id,
            name: dto.name,
            email: dto.email,
            isPremium: dto.is_premium
        )
    }

    static func toDTO(_ user: User) -> UserDTO {
        UserDTO(
            id: user.id.uuidString,
            name: user.name,
            email: user.email,
            is_premium: user.isPremium
        )
    }
}
```

```swift
// MARK: - Repository Implementation

final class UserRepositoryImpl: UserRepository {
    private let networkService: NetworkService
    private let cacheService: CacheService

    init(networkService: NetworkService, cacheService: CacheService) {
        self.networkService = networkService
        self.cacheService = cacheService
    }

    func fetchUser(id: UUID) async throws -> User {
        // Try cache first
        if let cached: UserDTO = cacheService.get(key: "user-\(id)") {
            if let user = UserMapper.toDomain(cached) {
                return user
            }
        }

        // Fetch from network
        let dto: UserDTO = try await networkService.request(
            endpoint: .user(id: id.uuidString)
        )

        // Cache result
        cacheService.set(key: "user-\(id)", value: dto)

        guard let user = UserMapper.toDomain(dto) else {
            throw RepositoryError.mappingFailed
        }

        return user
    }

    // ... other methods
}
```

### Presentation Layer

```swift
// MARK: - ViewModel

extension UserDetailView {
    @Observable @MainActor
    final class ViewModel {
        private(set) var user: User?
        private(set) var isLoading = false
        private(set) var error: Error?

        private let fetchUserUseCase: FetchUserUseCase

        init(fetchUserUseCase: FetchUserUseCase) {
            self.fetchUserUseCase = fetchUserUseCase
        }

        func loadUser(id: UUID) async {
            isLoading = true
            error = nil

            do {
                user = try await fetchUserUseCase.execute(id: id)
            } catch {
                self.error = error
            }

            isLoading = false
        }
    }
}

// MARK: - View

struct UserDetailView: View {
    @State private var viewModel: ViewModel

    init(userId: UUID, fetchUserUseCase: FetchUserUseCase) {
        _viewModel = State(initialValue: ViewModel(
            fetchUserUseCase: fetchUserUseCase
        ))
    }

    var body: some View {
        content
            .task { await viewModel.loadUser(id: userId) }
    }
}
```

## Modular Architecture

### Module Types

```
Core Module:
- Shared utilities, extensions, base types
- No business logic
- Zero external dependencies (except Foundation)

Domain Module:
- Entities, use case protocols
- Business rules
- Repository protocols
- Depends only on Core

Data Module:
- Repository implementations
- Network/database services
- Depends on Domain (for protocols)

Feature Module:
- UI for specific feature
- Views, ViewModels
- Depends on Domain

App Module:
- Composition root
- Dependency injection
- App entry point
- Depends on all modules
```

### Workspace Structure

```
MyApp.xcworkspace/
├── App/
│   └── App.xcodeproj           # Main app target
├── Modules/
│   ├── Core/
│   │   └── Core.xcodeproj      # Core utilities
│   ├── Domain/
│   │   └── Domain.xcodeproj    # Business logic
│   ├── Data/
│   │   └── Data.xcodeproj      # Data layer
│   └── Features/
│       ├── Home/
│       │   └── Home.xcodeproj
│       ├── Profile/
│       │   └── Profile.xcodeproj
│       └── Settings/
│           └── Settings.xcodeproj
└── Packages/
    └── SharedUI/               # SPM package for shared UI
```

### Module Dependency Graph

```
                    ┌─────────────────┐
                    │       App       │
                    └────────┬────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│     Home       │  │    Profile     │  │   Settings     │
│   (Feature)    │  │   (Feature)    │  │   (Feature)    │
└───────┬────────┘  └───────┬────────┘  └───────┬────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
                            ▼
                   ┌────────────────┐
                   │     Domain     │
                   └───────┬────────┘
                           │
                           ▼
                   ┌────────────────┐
                   │      Core      │
                   └────────────────┘

Data Module:
┌────────────────┐
│      Data      │ ──► Domain ──► Core
└────────────────┘
```

### SPM-Based Modular Architecture

```swift
// Package.swift (monorepo style)

let package = Package(
    name: "MyAppModules",
    platforms: [.iOS(.v17)],
    products: [
        .library(name: "Core", targets: ["Core"]),
        .library(name: "Domain", targets: ["Domain"]),
        .library(name: "Data", targets: ["Data"]),
        .library(name: "FeatureHome", targets: ["FeatureHome"]),
        .library(name: "FeatureProfile", targets: ["FeatureProfile"]),
    ],
    targets: [
        // Core - no dependencies
        .target(name: "Core"),
        .testTarget(name: "CoreTests", dependencies: ["Core"]),

        // Domain - depends on Core
        .target(name: "Domain", dependencies: ["Core"]),
        .testTarget(name: "DomainTests", dependencies: ["Domain"]),

        // Data - depends on Domain
        .target(name: "Data", dependencies: ["Domain"]),
        .testTarget(name: "DataTests", dependencies: ["Data"]),

        // Features - depend on Domain
        .target(name: "FeatureHome", dependencies: ["Domain"]),
        .target(name: "FeatureProfile", dependencies: ["Domain"]),
    ]
)
```

### Feature Module Structure

```
FeatureHome/
├── Sources/
│   └── FeatureHome/
│       ├── Public/
│       │   └── HomeModule.swift      # Public entry point
│       ├── Internal/
│       │   ├── Views/
│       │   │   ├── HomeView.swift
│       │   │   └── HomeView+ViewModel.swift
│       │   ├── Components/
│       │   │   └── HomeCard.swift
│       │   └── Coordinator/
│       │       └── HomeCoordinator.swift
│       └── Resources/
│           └── Localizable.strings
└── Tests/
    └── FeatureHomeTests/
```

```swift
// HomeModule.swift - Public API for the module
public struct HomeModule {
    public static func makeHomeView(
        dependencies: HomeDependencies
    ) -> some View {
        HomeView(
            viewModel: .init(
                fetchItemsUseCase: dependencies.fetchItemsUseCase
            )
        )
    }
}

public protocol HomeDependencies {
    var fetchItemsUseCase: FetchItemsUseCase { get }
}
```

## Dependency Injection

### Composition Root

```swift
// App/DependencyContainer.swift

@MainActor
final class DependencyContainer {
    // MARK: - Infrastructure

    private lazy var networkService: NetworkService = {
        NetworkServiceImpl(baseURL: Config.apiBaseURL)
    }()

    private lazy var cacheService: CacheService = {
        CacheServiceImpl()
    }()

    // MARK: - Repositories

    private lazy var userRepository: UserRepository = {
        UserRepositoryImpl(
            networkService: networkService,
            cacheService: cacheService
        )
    }()

    private lazy var orderRepository: OrderRepository = {
        OrderRepositoryImpl(networkService: networkService)
    }()

    // MARK: - Use Cases

    func makeFetchUserUseCase() -> FetchUserUseCase {
        FetchUserUseCaseImpl(userRepository: userRepository)
    }

    func makeCreateOrderUseCase() -> CreateOrderUseCase {
        CreateOrderUseCaseImpl(
            orderRepository: orderRepository,
            userRepository: userRepository
        )
    }

    // MARK: - Feature Dependencies

    func makeHomeDependencies() -> HomeDependencies {
        HomeDependenciesImpl(
            fetchItemsUseCase: makeFetchItemsUseCase()
        )
    }
}
```

### Environment-Based DI

```swift
// Using SwiftUI Environment

private struct DependencyContainerKey: EnvironmentKey {
    static let defaultValue = DependencyContainer()
}

extension EnvironmentValues {
    var dependencies: DependencyContainer {
        get { self[DependencyContainerKey.self] }
        set { self[DependencyContainerKey.self] = newValue }
    }
}

// Usage in App
@main
struct MyApp: App {
    private let container = DependencyContainer()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.dependencies, container)
        }
    }
}

// Usage in View
struct SomeView: View {
    @Environment(\.dependencies) private var dependencies

    var body: some View {
        // Access dependencies
    }
}
```

## Practical Example: Complete Module

### Project Structure

```
OrderFeature/
├── Package.swift
├── Sources/
│   └── OrderFeature/
│       ├── OrderModule.swift              # Public entry
│       ├── Views/
│       │   ├── OrderListView.swift
│       │   └── OrderDetailView.swift
│       ├── ViewModels/
│       │   ├── OrderListView+ViewModel.swift
│       │   └── OrderDetailView+ViewModel.swift
│       └── Components/
│           └── OrderCard.swift
└── Tests/
    └── OrderFeatureTests/
        └── OrderListViewModelTests.swift
```

### Implementation

```swift
// OrderModule.swift
import SwiftUI
import Domain

public struct OrderModule {
    public static func makeOrderListView(
        dependencies: OrderDependencies
    ) -> some View {
        OrderListView(dependencies: dependencies)
    }
}

public protocol OrderDependencies {
    var fetchOrdersUseCase: FetchOrdersUseCase { get }
    var createOrderUseCase: CreateOrderUseCase { get }
}
```

```swift
// OrderListView.swift
import SwiftUI
import Domain

struct OrderListView: View {
    @State private var viewModel: ViewModel

    init(dependencies: OrderDependencies) {
        _viewModel = State(initialValue: ViewModel(
            fetchOrdersUseCase: dependencies.fetchOrdersUseCase
        ))
    }

    var body: some View {
        NavigationStack {
            content
                .navigationTitle("Orders")
                .task { await viewModel.loadOrders() }
                .refreshable { await viewModel.refresh() }
        }
    }

    @ViewBuilder
    private var content: some View {
        switch viewModel.state {
        case .loading:
            ProgressView()
        case .loaded(let orders):
            orderList(orders)
        case .error(let message):
            errorView(message)
        }
    }

    private func orderList(_ orders: [Order]) -> some View {
        List(orders) { order in
            NavigationLink(value: order) {
                OrderCard(order: order)
            }
        }
        .navigationDestination(for: Order.self) { order in
            OrderDetailView(order: order)
        }
    }

    private func errorView(_ message: String) -> some View {
        ContentUnavailableView(
            "Error",
            systemImage: "exclamationmark.triangle",
            description: Text(message)
        )
    }
}
```

```swift
// OrderListView+ViewModel.swift
import Domain

extension OrderListView {
    @Observable @MainActor
    final class ViewModel {
        private(set) var state: ViewState = .loading

        enum ViewState {
            case loading
            case loaded([Order])
            case error(String)
        }

        private let fetchOrdersUseCase: FetchOrdersUseCase

        init(fetchOrdersUseCase: FetchOrdersUseCase) {
            self.fetchOrdersUseCase = fetchOrdersUseCase
        }

        func loadOrders() async {
            state = .loading
            do {
                let orders = try await fetchOrdersUseCase.execute()
                state = .loaded(orders)
            } catch {
                state = .error(error.localizedDescription)
            }
        }

        func refresh() async {
            await loadOrders()
        }
    }
}
```

## Best Practices Checklist

- [ ] Dependencies point inward only (Dependency Rule)
- [ ] Domain layer has zero framework dependencies
- [ ] Use protocols for repository boundaries
- [ ] Keep use cases focused (Single Responsibility)
- [ ] DTOs are separate from domain entities
- [ ] Feature modules expose minimal public API
- [ ] Composition root wires all dependencies
- [ ] Each module is independently testable
- [ ] No circular dependencies between modules
- [ ] Consider build time when splitting modules
