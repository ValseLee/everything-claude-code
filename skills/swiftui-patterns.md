---
name: swiftui-patterns
description: SwiftUI component patterns for iOS 18+, including stateless View philosophy, View-ViewModel pattern, state management, and performance optimization.
---

# SwiftUI Patterns (iOS 18+)

Modern SwiftUI patterns emphasizing stateless views and clear data flow.

## View Philosophy

### Stateless View (CRITICAL)

Views should be stateless renderers. All state lives in the ViewModel.

```swift
// ✅ GOOD: View is a pure function of ViewModel state
struct ProfileView: View {
    @State private var viewModel = ViewModel()

    var body: some View {
        // View only reads state and sends actions
        VStack {
            if viewModel.isLoading {
                ProgressView()
            } else if let user = viewModel.user {
                UserCard(user: user)
            }

            Button("Refresh") {
                viewModel.refresh()  // Action → ViewModel
            }
        }
        .task {
            await viewModel.loadUser()
        }
    }
}

// ❌ BAD: View manages its own state
struct ProfileView: View {
    @State private var isLoading = false  // State in View
    @State private var user: User?         // State in View
    @State private var error: Error?       // State in View

    var body: some View {
        // Logic mixed with UI
    }
}
```

### Body as Pure Rendering

```swift
// ✅ GOOD: Body only describes UI
var body: some View {
    List(viewModel.items) { item in
        ItemRow(item: item)
    }
}

// ❌ BAD: Logic in body
var body: some View {
    let filteredItems = items.filter { $0.isActive }  // Logic in body
    let sortedItems = filteredItems.sorted { $0.date > $1.date }  // More logic

    List(sortedItems) { item in
        ItemRow(item: item)
    }
}
// Move filtering/sorting to ViewModel
```

## View-ViewModel Pattern (Recommended)

### ViewModel as View Extension

```swift
// ✅ RECOMMENDED: ViewModel is extension of View
struct UserListView: View {
    @State private var viewModel = ViewModel()

    var body: some View {
        NavigationStack {
            content
                .navigationTitle("Users")
                .task { await viewModel.loadUsers() }
                .refreshable { await viewModel.refresh() }
        }
    }
}

// MARK: - Subviews

private extension UserListView {
    @ViewBuilder
    var content: some View {
        switch viewModel.state {
        case .loading:
            ProgressView()
        case .loaded(let users):
            userList(users)
        case .error(let message):
            errorView(message)
        }
    }

    func userList(_ users: [User]) -> some View {
        List(users) { user in
            UserRow(user: user)
        }
    }

    func errorView(_ message: String) -> some View {
        ContentUnavailableView(
            "Error",
            systemImage: "exclamationmark.triangle",
            description: Text(message)
        )
    }
}

// MARK: - ViewModel

extension UserListView {
    @Observable @MainActor
    final class ViewModel {
        // State
        private(set) var state: ViewState = .loading

        enum ViewState {
            case loading
            case loaded([User])
            case error(String)
        }

        // Dependencies
        private let userService: UserService

        init(userService: UserService = .shared) {
            self.userService = userService
        }

        // Actions
        func loadUsers() async {
            state = .loading
            do {
                let users = try await userService.fetchUsers()
                state = .loaded(users)
            } catch {
                state = .error(error.localizedDescription)
            }
        }

        func refresh() async {
            await loadUsers()
        }
    }
}
```

### Key Principles

```swift
// 1. View and ViewModel are 1:1
// Each View has its own ViewModel

// 2. ViewModel is View-specific (not reusable)
// Shared logic goes to Service/Repository

// 3. ViewModel structure
extension SomeView {
    @Observable @MainActor
    final class ViewModel {
        // @Observable: Automatic observation
        // @MainActor: Safe UI updates
        // final class: No inheritance, reference semantics
    }
}

// 4. State is private(set)
private(set) var users: [User] = []  // Read externally, write internally

// 5. Actions are functions
func didTapSubmit() { }
func loadData() async { }
```

### Passing Dependencies

```swift
// ✅ GOOD: Inject via ViewModel init
struct OrderView: View {
    @State private var viewModel: ViewModel

    init(orderId: String, orderService: OrderService = .shared) {
        _viewModel = State(initialValue: ViewModel(
            orderId: orderId,
            orderService: orderService
        ))
    }

    var body: some View {
        // ...
    }
}

extension OrderView {
    @Observable @MainActor
    final class ViewModel {
        private let orderId: String
        private let orderService: OrderService

        init(orderId: String, orderService: OrderService) {
            self.orderId = orderId
            self.orderService = orderService
        }
    }
}
```

## State Management

### Property Wrappers

```swift
// @State: ViewModel instance (View owns it)
@State private var viewModel = ViewModel()

// @Binding: Two-way connection to parent
struct ChildView: View {
    @Binding var isPresented: Bool
}

// @Bindable: Create bindings from @Observable
struct FormView: View {
    @Bindable var viewModel: ViewModel

    var body: some View {
        TextField("Name", text: $viewModel.name)
    }
}

// @Environment: App-wide dependencies
@Environment(\.colorScheme) var colorScheme
@Environment(AuthManager.self) var authManager
```

### @Observable (iOS 17+)

```swift
// ✅ GOOD: @Observable replaces ObservableObject
@Observable @MainActor
final class ViewModel {
    var count = 0           // Automatically observed
    var items: [Item] = []  // Changes trigger view updates

    // No @Published needed!
}

// In View
struct CounterView: View {
    @State private var viewModel = ViewModel()

    var body: some View {
        Text("\(viewModel.count)")  // Automatically updates
    }
}
```

### @Bindable

```swift
// When you need bindings from @Observable
struct EditUserView: View {
    @Bindable var viewModel: ViewModel

    var body: some View {
        Form {
            TextField("Name", text: $viewModel.name)
            TextField("Email", text: $viewModel.email)
            Toggle("Active", isOn: $viewModel.isActive)
        }
    }
}

extension EditUserView {
    @Observable @MainActor
    final class ViewModel {
        var name: String
        var email: String
        var isActive: Bool

        init(user: User) {
            self.name = user.name
            self.email = user.email
            self.isActive = user.isActive
        }
    }
}
```

## Data Flow

### Unidirectional Data Flow

```
┌─────────────────────────────────────────┐
│                  View                   │
│  ┌─────────────────────────────────┐    │
│  │     Renders viewModel.state     │    │
│  └─────────────────────────────────┘    │
│                   │                     │
│                   ▼                     │
│  ┌─────────────────────────────────┐    │
│  │   User Action (tap, input)      │    │
│  └─────────────────────────────────┘    │
│                   │                     │
└───────────────────│─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│              ViewModel                  │
│  ┌─────────────────────────────────┐    │
│  │   viewModel.action()            │    │
│  └─────────────────────────────────┘    │
│                   │                     │
│                   ▼                     │
│  ┌─────────────────────────────────┐    │
│  │   State mutation                │    │
│  └─────────────────────────────────┘    │
│                   │                     │
└───────────────────│─────────────────────┘
                    │
                    ▼ (triggers view update)
```

### Action/State Pattern

```swift
extension TaskListView {
    @Observable @MainActor
    final class ViewModel {
        // MARK: - State

        private(set) var tasks: [Task] = []
        private(set) var isLoading = false
        private(set) var filter: Filter = .all

        enum Filter: CaseIterable {
            case all, active, completed
        }

        // MARK: - Computed

        var filteredTasks: [Task] {
            switch filter {
            case .all: tasks
            case .active: tasks.filter { !$0.isCompleted }
            case .completed: tasks.filter { $0.isCompleted }
            }
        }

        // MARK: - Actions

        func loadTasks() async {
            isLoading = true
            defer { isLoading = false }

            tasks = await taskService.fetchAll()
        }

        func addTask(_ title: String) {
            let task = Task(title: title)
            tasks.append(task)
        }

        func toggleComplete(_ task: Task) {
            guard let index = tasks.firstIndex(where: { $0.id == task.id }) else {
                return
            }
            tasks[index].isCompleted.toggle()
        }

        func setFilter(_ filter: Filter) {
            self.filter = filter
        }
    }
}
```

### Minimize Parent-Child Binding

```swift
// ✅ GOOD: Pass data down, actions up
struct ParentView: View {
    @State private var viewModel = ViewModel()

    var body: some View {
        ChildView(
            item: viewModel.selectedItem,
            onDelete: { viewModel.deleteItem() }
        )
    }
}

struct ChildView: View {
    let item: Item          // Data down (read-only)
    let onDelete: () -> Void  // Action up (callback)

    var body: some View {
        VStack {
            Text(item.name)
            Button("Delete", action: onDelete)
        }
    }
}

// ❌ AVOID: Excessive Binding
struct ChildView: View {
    @Binding var item: Item  // Two-way binding when not needed
}
```

## View Composition

### ViewBuilder

```swift
struct CardView<Content: View>: View {
    @ViewBuilder let content: () -> Content

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            content()
        }
        .padding()
        .background(.regularMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

// Usage
CardView {
    Text("Title")
        .font(.headline)
    Text("Description")
        .foregroundStyle(.secondary)
}
```

### Small, Focused Views

```swift
// ✅ GOOD: Break down into small views
struct OrderDetailView: View {
    @State private var viewModel = ViewModel()

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                headerSection
                itemsSection
                totalSection
            }
            .padding()
        }
    }
}

private extension OrderDetailView {
    var headerSection: some View {
        VStack(alignment: .leading) {
            Text("Order #\(viewModel.order.id)")
                .font(.headline)
            Text(viewModel.order.date.formatted())
                .foregroundStyle(.secondary)
        }
    }

    var itemsSection: some View {
        ForEach(viewModel.order.items) { item in
            OrderItemRow(item: item)
        }
    }

    var totalSection: some View {
        HStack {
            Text("Total")
                .font(.headline)
            Spacer()
            Text(viewModel.order.total, format: .currency(code: "USD"))
        }
    }
}
```

### #Preview Macro

```swift
// ✅ iOS 17+: #Preview macro
#Preview {
    UserListView()
}

#Preview("Loading State") {
    let viewModel = UserListView.ViewModel()
    viewModel.state = .loading
    return UserListView(viewModel: viewModel)
}

#Preview("With Users") {
    let viewModel = UserListView.ViewModel()
    viewModel.state = .loaded([.mock, .mock, .mock])
    return UserListView(viewModel: viewModel)
}

// Traits
#Preview(traits: .sizeThatFitsLayout) {
    UserRow(user: .mock)
}
```

## Performance Optimization

### Lazy Containers

```swift
// ✅ GOOD: Lazy loading for long lists
ScrollView {
    LazyVStack(spacing: 12) {
        ForEach(viewModel.items) { item in
            ItemRow(item: item)  // Only created when visible
        }
    }
}

// ❌ BAD: Regular VStack loads all items immediately
ScrollView {
    VStack {
        ForEach(viewModel.items) { item in
            ItemRow(item: item)  // All created at once
        }
    }
}
```

### Equatable Views

```swift
// ✅ Conform to Equatable for optimized diffing
struct ItemRow: View, Equatable {
    let item: Item

    static func == (lhs: ItemRow, rhs: ItemRow) -> Bool {
        lhs.item.id == rhs.item.id &&
        lhs.item.title == rhs.item.title
    }

    var body: some View {
        Text(item.title)
    }
}
```

### task(id:) Modifier

```swift
// ✅ GOOD: Re-run task when id changes
struct UserDetailView: View {
    let userId: String
    @State private var viewModel = ViewModel()

    var body: some View {
        content
            .task(id: userId) {
                // Cancels previous task and starts new one
                await viewModel.loadUser(id: userId)
            }
    }
}

// ❌ BAD: task without id doesn't re-run
.task {
    await viewModel.loadUser(id: userId)  // Won't re-run when userId changes
}
```

### Avoid Expensive Operations in Body

```swift
// ❌ BAD: Filtering in body
var body: some View {
    List(items.filter { $0.isActive }.sorted { $0.date > $1.date }) { item in
        // ...
    }
}

// ✅ GOOD: Compute in ViewModel
var body: some View {
    List(viewModel.filteredSortedItems) { item in
        // ...
    }
}

extension MyView {
    @Observable @MainActor
    final class ViewModel {
        var items: [Item] = []

        var filteredSortedItems: [Item] {
            items
                .filter { $0.isActive }
                .sorted { $0.date > $1.date }
        }
    }
}
```

## iOS 18 Features

### @Previewable Macro

```swift
// ✅ iOS 18: @Previewable for state in previews
#Preview {
    @Previewable @State var count = 0

    VStack {
        Text("Count: \(count)")
        Button("+") { count += 1 }
    }
}
```

### Container Values

```swift
// Custom container values (iOS 18)
extension ContainerValues {
    @Entry var customSpacing: CGFloat = 8
}

struct CustomStack<Content: View>: View {
    @ViewBuilder var content: Content

    var body: some View {
        VStack {
            ForEach(subviewOf: content) { subview in
                subview
                    .padding(.bottom, subview.containerValues.customSpacing)
            }
        }
    }
}
```

## Best Practices Checklist

- [ ] Views are stateless (state lives in ViewModel)
- [ ] ViewModel is extension of View
- [ ] ViewModel uses @Observable @MainActor final class
- [ ] State is private(set) in ViewModel
- [ ] Unidirectional data flow (action → state → view)
- [ ] Minimal use of @Binding between views
- [ ] Use LazyVStack/LazyHStack for lists
- [ ] Use task(id:) when task depends on parameters
- [ ] No expensive operations in body
- [ ] Small, focused view components
