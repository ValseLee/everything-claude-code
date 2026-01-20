# Common Patterns

## View-ViewModel Pattern

```swift
struct UserListView: View {
    @State private var viewModel = ViewModel()

    var body: some View {
        List(viewModel.users) { user in
            UserRow(user: user)
        }
        .task { await viewModel.loadUsers() }
    }
}

extension UserListView {
    @Observable @MainActor
    final class ViewModel {
        private(set) var users: [User] = []

        func loadUsers() async {
            users = await userService.fetchAll()
        }
    }
}
```

## Repository Pattern

```swift
protocol UserRepository {
    func fetchAll() async throws -> [User]
    func fetchById(_ id: UUID) async throws -> User?
    func save(_ user: User) async throws
    func delete(_ id: UUID) async throws
}

final class UserRepositoryImpl: UserRepository {
    private let modelContext: ModelContext

    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }

    func fetchAll() async throws -> [User] {
        let descriptor = FetchDescriptor<User>()
        return try modelContext.fetch(descriptor)
    }

    // ... other methods
}
```

## Use Case Pattern

```swift
protocol FetchUserUseCase {
    func execute(id: UUID) async throws -> User
}

final class FetchUserUseCaseImpl: FetchUserUseCase {
    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func execute(id: UUID) async throws -> User {
        guard let user = try await repository.fetchById(id) else {
            throw DomainError.userNotFound
        }
        return user
    }
}
```

## Dependency Injection

```swift
// ✅ GOOD: Constructor injection
struct OrderView: View {
    @State private var viewModel: ViewModel

    init(orderId: String, orderService: OrderService = .shared) {
        _viewModel = State(initialValue: ViewModel(
            orderId: orderId,
            orderService: orderService
        ))
    }
}

// ✅ GOOD: Environment for app-wide dependencies
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(AuthManager.shared)
        }
    }
}
```

## Skeleton Projects

When implementing new functionality:
1. Search for battle-tested skeleton projects
2. Use parallel agents to evaluate options:
   - Security assessment
   - Extensibility analysis
   - Relevance scoring
   - Implementation planning
3. Clone best match as foundation
4. Iterate within proven structure
