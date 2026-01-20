# Example Project CLAUDE.md

This is an example project-level CLAUDE.md file. Place this in your iOS project root.

## Project Overview

[Brief description of your app - what it does, target iOS version, key frameworks]

## Critical Rules

### 1. Code Organization

- Many small files over few large files
- High cohesion, low coupling
- 200-400 lines typical, 800 max per file
- Organize by feature/module, not by type

### 2. Code Style

- No emojis in code, comments, or documentation
- Immutability always - prefer `let` over `var`
- No `print()` in production code
- Proper error handling with `do/catch` and `Result`
- No forced unwrap `!` except compile-time guaranteed values

### 3. Architecture

- View-ViewModel pattern with `@Observable`
- Stateless Views - all state in ViewModel
- Clean Architecture: UI -> Domain -> Data layers
- Repository pattern for data access

### 4. Testing

- TDD: Write tests first
- 80% minimum coverage
- Use Swift Testing framework (`@Suite`, `@Test`)
- Unit tests for ViewModels and Use Cases
- UI tests with XCUITest for critical flows

### 5. Security

- No hardcoded secrets
- Use Keychain for sensitive data
- App Transport Security (ATS) enabled
- Data Protection for files at rest
- Validate all user inputs

## File Structure

```
MyApp/
|-- Sources/
|   |-- Features/           # Feature modules
|   |   |-- Home/
|   |   |   |-- HomeView.swift
|   |   |   |-- HomeViewModel.swift
|   |   |-- Settings/
|   |-- Core/               # Shared domain logic
|   |   |-- Models/
|   |   |-- UseCases/
|   |   |-- Repositories/
|   |-- Infrastructure/     # External dependencies
|   |   |-- Network/
|   |   |-- Persistence/
|   |-- App/                # App entry point
|       |-- MyAppApp.swift
|-- Tests/
|   |-- UnitTests/
|   |-- UITests/
```

## Key Patterns

### View-ViewModel Pattern

```swift
struct HomeView: View {
    @State private var viewModel = ViewModel()

    var body: some View {
        List(viewModel.items) { item in
            Text(item.title)
        }
        .task { await viewModel.loadItems() }
    }
}

extension HomeView {
    @Observable @MainActor
    final class ViewModel {
        private(set) var items: [Item] = []
        private let useCase: FetchItemsUseCase

        init(useCase: FetchItemsUseCase = .init()) {
            self.useCase = useCase
        }

        func loadItems() async {
            items = await useCase.execute()
        }
    }
}
```

### Error Handling

```swift
enum AppError: LocalizedError {
    case networkUnavailable
    case invalidData
    case unauthorized

    var errorDescription: String? {
        switch self {
        case .networkUnavailable: "Network unavailable"
        case .invalidData: "Invalid data received"
        case .unauthorized: "Session expired"
        }
    }
}

func fetchData() async -> Result<Data, AppError> {
    do {
        let data = try await networkClient.fetch()
        return .success(data)
    } catch {
        return .failure(.networkUnavailable)
    }
}
```

## Environment Configuration

```swift
// Use xcconfig files for environment-specific values
// Debug.xcconfig
API_BASE_URL = https:/$()/api.dev.example.com

// Release.xcconfig
API_BASE_URL = https:/$()/api.example.com

// Access via Info.plist or Bundle
extension Bundle {
    var apiBaseURL: URL {
        URL(string: infoDictionary?["API_BASE_URL"] as? String ?? "")!
    }
}
```

## Available Commands

- `/tdd` - Test-driven development workflow
- `/plan` - Create implementation plan
- `/code-review` - Review code quality
- `/build-fix` - Fix xcodebuild errors

## Git Workflow

- Conventional commits: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`
- Never commit to main directly
- PRs require review
- All tests must pass before merge
