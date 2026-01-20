---
name: swiftdata
description: SwiftData patterns including @Model, #Predicate, #Expression, ModelActor for background operations, and relationship management.
---

# SwiftData

Modern data persistence with SwiftData for iOS 17+.

## Basic Setup

### @Model Macro

```swift
import SwiftData

@Model
final class User {
    var id: UUID
    var name: String
    var email: String
    var createdAt: Date

    // Relationships
    @Relationship(deleteRule: .cascade)
    var posts: [Post] = []

    init(name: String, email: String) {
        self.id = UUID()
        self.name = name
        self.email = email
        self.createdAt = Date()
    }
}

@Model
final class Post {
    var id: UUID
    var title: String
    var content: String
    var publishedAt: Date?

    // Inverse relationship
    var author: User?

    init(title: String, content: String, author: User? = nil) {
        self.id = UUID()
        self.title = title
        self.content = content
        self.author = author
    }
}
```

### ModelContainer and ModelContext

```swift
// App setup
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: [User.self, Post.self])
    }
}

// In SwiftUI View
struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var users: [User]

    var body: some View {
        List(users) { user in
            Text(user.name)
        }
    }

    func addUser() {
        let user = User(name: "John", email: "john@example.com")
        modelContext.insert(user)
        // Auto-save is enabled by default
    }
}
```

### Schema Versioning

```swift
// Define schema versions
enum SchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] {
        [User.self]
    }

    @Model
    final class User {
        var name: String
        init(name: String) { self.name = name }
    }
}

enum SchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)
    static var models: [any PersistentModel.Type] {
        [User.self]
    }

    @Model
    final class User {
        var name: String
        var email: String?  // New field

        init(name: String, email: String? = nil) {
            self.name = name
            self.email = email
        }
    }
}

// Migration plan
enum MigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [SchemaV1.self, SchemaV2.self]
    }

    static var stages: [MigrationStage] {
        [migrateV1toV2]
    }

    static let migrateV1toV2 = MigrationStage.lightweight(
        fromVersion: SchemaV1.self,
        toVersion: SchemaV2.self
    )
}

// Use migration plan
let container = try ModelContainer(
    for: User.self,
    migrationPlan: MigrationPlan.self
)
```

## #Predicate

### Basic Syntax

```swift
// ✅ GOOD: Simple predicates
let activePredicate = #Predicate<User> { user in
    user.isActive == true
}

let namePredicate = #Predicate<User> { user in
    user.name.contains("John")
}

// Use with @Query
@Query(filter: #Predicate<User> { $0.isActive })
private var activeUsers: [User]

// Use with FetchDescriptor
let descriptor = FetchDescriptor<User>(
    predicate: #Predicate { $0.isActive }
)
let users = try modelContext.fetch(descriptor)
```

### Supported Operations

```swift
// Comparison operators
#Predicate<User> { $0.age >= 18 }
#Predicate<User> { $0.name == "John" }
#Predicate<User> { $0.score != nil }

// String operations
#Predicate<User> { $0.name.contains("jo") }
#Predicate<User> { $0.name.localizedStandardContains("john") }  // Case-insensitive
#Predicate<User> { $0.name.hasPrefix("J") }
#Predicate<User> { $0.name.hasSuffix("n") }

// Logical operators
#Predicate<User> { $0.isActive && $0.age >= 18 }
#Predicate<User> { $0.isPremium || $0.isAdmin }
#Predicate<User> { !$0.isDeleted }

// Date comparisons
let now = Date()
#Predicate<User> { $0.createdAt < now }
#Predicate<User> { $0.lastLogin > $0.createdAt }

// Optional handling
#Predicate<User> { $0.email != nil }
#Predicate<Post> { $0.publishedAt != nil }
```

### Compound Predicates

```swift
// ✅ GOOD: Combine predicates
func searchPredicate(query: String, isActive: Bool?) -> Predicate<User> {
    #Predicate<User> { user in
        // Text search
        (query.isEmpty || user.name.localizedStandardContains(query)) &&
        // Optional filter
        (isActive == nil || user.isActive == isActive)
    }
}

// Usage
@Query(filter: searchPredicate(query: "john", isActive: true))
private var filteredUsers: [User]
```

### Common Mistakes and Solutions

```swift
// ❌ BAD: Variable capture issues
let searchTerm = "John"
let predicate = #Predicate<User> { user in
    user.name.contains(searchTerm)  // May not work as expected
}

// ✅ GOOD: Use localizedStandardContains for variables
let searchTerm = "John"
let predicate = #Predicate<User> { user in
    user.name.localizedStandardContains(searchTerm)
}

// ❌ BAD: Complex computed properties
#Predicate<User> { $0.fullName.contains("John") }  // fullName is computed

// ✅ GOOD: Use stored properties
#Predicate<User> { $0.firstName.contains("John") || $0.lastName.contains("John") }

// ❌ BAD: Method calls not supported
#Predicate<User> { $0.isValidEmail() }  // Methods not allowed

// ✅ GOOD: Use properties or inline logic
#Predicate<User> { $0.email.contains("@") }

// ❌ BAD: Array operations with closures
#Predicate<User> { $0.posts.filter { $0.isPublished }.count > 5 }  // Not supported

// ✅ GOOD: Simple collection checks
#Predicate<User> { $0.posts.isEmpty == false }
```

## #Expression

### Custom Expressions

```swift
// Define custom expression for sorting
let sortByPostCount = #Expression<User, Int> { user in
    user.posts.count
}

// Use in FetchDescriptor
var descriptor = FetchDescriptor<User>()
descriptor.sortBy = [
    SortDescriptor(sortByPostCount, order: .reverse)
]
```

### Aggregation Functions

```swift
// ✅ GOOD: Count expression
let postCountExpression = #Expression<User, Int> { user in
    user.posts.count
}

// ✅ GOOD: Conditional count
let publishedPostCount = #Expression<User, Int> { user in
    user.posts.filter { $0.publishedAt != nil }.count
}

// Using expressions in predicates
let activeAuthors = #Predicate<User> { user in
    user.posts.count > 0
}
```

### Sort Expressions

```swift
// Sort by computed value
var descriptor = FetchDescriptor<Post>(
    sortBy: [
        SortDescriptor(\Post.publishedAt, order: .reverse)
    ]
)

// Custom sort expression
let engagementScore = #Expression<Post, Int> { post in
    post.likes + post.comments * 2
}

// Multiple sort criteria
descriptor.sortBy = [
    SortDescriptor(\Post.isPinned, order: .reverse),  // Pinned first
    SortDescriptor(\Post.publishedAt, order: .reverse)  // Then by date
]
```

## Background Operations (ModelActor)

### @ModelActor Macro

```swift
import SwiftData

// ✅ GOOD: ModelActor for background work
@ModelActor
actor DataManager {
    // modelContainer and modelExecutor are auto-synthesized

    func importUsers(from data: [UserDTO]) async throws {
        for dto in data {
            let user = User(name: dto.name, email: dto.email)
            modelContext.insert(user)
        }
        try modelContext.save()
    }

    func fetchUserCount() async -> Int {
        let descriptor = FetchDescriptor<User>()
        return (try? modelContext.fetchCount(descriptor)) ?? 0
    }

    func deleteAllUsers() async throws {
        try modelContext.delete(model: User.self)
        try modelContext.save()
    }
}
```

### Creating ModelActor

```swift
// Initialize with ModelContainer
let container = try ModelContainer(for: User.self)
let dataManager = DataManager(modelContainer: container)

// Use in SwiftUI
struct ContentView: View {
    @Environment(\.modelContext) private var modelContext

    var body: some View {
        Button("Import Data") {
            Task {
                let container = modelContext.container
                let manager = DataManager(modelContainer: container)
                try await manager.importUsers(from: userData)
            }
        }
    }
}
```

### Main Context Synchronization

```swift
@ModelActor
actor BackgroundProcessor {

    func processLargeDataset(_ items: [ItemDTO]) async throws -> Int {
        var processedCount = 0

        // Process in batches
        let batchSize = 100
        for batch in items.chunked(into: batchSize) {
            for dto in batch {
                let item = Item(data: dto)
                modelContext.insert(item)
                processedCount += 1
            }

            // Save periodically to avoid memory pressure
            try modelContext.save()
        }

        return processedCount
    }
}

// Helper extension
extension Array {
    func chunked(into size: Int) -> [[Element]] {
        stride(from: 0, to: count, by: size).map {
            Array(self[$0 ..< Swift.min($0 + size, count)])
        }
    }
}

// Usage - changes automatically sync to main context
struct ImportView: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var items: [Item]  // Auto-updates after background insert

    func performImport() async {
        let processor = BackgroundProcessor(modelContainer: modelContext.container)
        let count = try await processor.processLargeDataset(largeDataset)
        print("Imported \(count) items")
        // @Query automatically reflects changes
    }
}
```

### Large Dataset Patterns

```swift
@ModelActor
actor BatchProcessor {

    // ✅ GOOD: Streaming fetch for large datasets
    func processAllUsers(handler: @Sendable (User) async -> Void) async throws {
        let batchSize = 50
        var offset = 0

        while true {
            var descriptor = FetchDescriptor<User>()
            descriptor.fetchLimit = batchSize
            descriptor.fetchOffset = offset

            let users = try modelContext.fetch(descriptor)

            if users.isEmpty { break }

            for user in users {
                await handler(user)
            }

            offset += batchSize
        }
    }

    // ✅ GOOD: Batch delete with predicate
    func deleteInactiveUsers(olderThan date: Date) async throws -> Int {
        let predicate = #Predicate<User> { user in
            user.isActive == false && user.lastLogin < date
        }

        var descriptor = FetchDescriptor<User>(predicate: predicate)
        let users = try modelContext.fetch(descriptor)
        let count = users.count

        for user in users {
            modelContext.delete(user)
        }

        try modelContext.save()
        return count
    }

    // ✅ GOOD: Bulk update
    func markAllAsRead(for userId: UUID) async throws {
        let predicate = #Predicate<Notification> { notification in
            notification.userId == userId && notification.isRead == false
        }

        let descriptor = FetchDescriptor<Notification>(predicate: predicate)
        let notifications = try modelContext.fetch(descriptor)

        for notification in notifications {
            notification.isRead = true
        }

        try modelContext.save()
    }
}
```

## Relationships

### @Relationship

```swift
@Model
final class Author {
    var name: String

    // One-to-many with cascade delete
    @Relationship(deleteRule: .cascade, inverse: \Book.author)
    var books: [Book] = []

    init(name: String) {
        self.name = name
    }
}

@Model
final class Book {
    var title: String

    // Many-to-one (inverse is auto-inferred if specified above)
    var author: Author?

    // Many-to-many
    @Relationship
    var tags: [Tag] = []

    init(title: String) {
        self.title = title
    }
}

@Model
final class Tag {
    var name: String

    @Relationship(inverse: \Book.tags)
    var books: [Book] = []

    init(name: String) {
        self.name = name
    }
}
```

### Delete Rules

```swift
// .cascade - Delete related objects
@Relationship(deleteRule: .cascade)
var posts: [Post] = []

// .nullify - Set relationship to nil (default)
@Relationship(deleteRule: .nullify)
var category: Category?

// .deny - Prevent deletion if relationships exist
@Relationship(deleteRule: .deny)
var orders: [Order] = []

// .noAction - Do nothing (handle manually)
@Relationship(deleteRule: .noAction)
var references: [Reference] = []
```

### Inverse Relationships

```swift
// ✅ GOOD: Explicit inverse
@Model
final class Parent {
    @Relationship(deleteRule: .cascade, inverse: \Child.parent)
    var children: [Child] = []
}

@Model
final class Child {
    var parent: Parent?  // Auto-managed inverse
}

// Usage
let parent = Parent()
let child = Child()

// Either direction works
parent.children.append(child)
// OR
child.parent = parent

// Both relationships are kept in sync automatically
```

## Best Practices Checklist

- [ ] Use @Model for all persistent types
- [ ] Define schema versions for migrations
- [ ] Use #Predicate with supported operations only
- [ ] Use @ModelActor for background operations
- [ ] Save periodically during batch operations
- [ ] Define explicit inverse relationships
- [ ] Choose appropriate delete rules
- [ ] Use fetchLimit/fetchOffset for large datasets
- [ ] Avoid complex computations in #Predicate
- [ ] Test migrations before release
