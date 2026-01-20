# Performance Optimization

## Model Selection Strategy

**Haiku 4.5** (90% of Sonnet capability, 3x cost savings):
- Lightweight agents with frequent invocation
- Pair programming and code generation
- Worker agents in multi-agent systems

**Sonnet 4.5** (Best coding model):
- Main development work
- Orchestrating multi-agent workflows
- Complex coding tasks

**Opus 4.5** (Deepest reasoning):
- Complex architectural decisions
- Maximum reasoning requirements
- Research and analysis tasks

## Context Window Management

Avoid last 20% of context window for:
- Large-scale refactoring
- Feature implementation spanning multiple files
- Debugging complex interactions

Lower context sensitivity tasks:
- Single-file edits
- Independent utility creation
- Documentation updates
- Simple bug fixes

## SwiftUI Performance

### Use Lazy Containers

```swift
// ✅ GOOD: Lazy loading for long lists
ScrollView {
    LazyVStack(spacing: 12) {
        ForEach(items) { item in
            ItemRow(item: item)  // Only created when visible
        }
    }
}

// ❌ BAD: Regular VStack loads all items
ScrollView {
    VStack {
        ForEach(items) { item in
            ItemRow(item: item)  // All created at once
        }
    }
}
```

### Avoid Expensive Operations in Body

```swift
// ❌ BAD: Filtering in body
var body: some View {
    List(items.filter { $0.isActive }.sorted { $0.date > $1.date }) { item in
        // Re-computed on every render
    }
}

// ✅ GOOD: Compute in ViewModel
var body: some View {
    List(viewModel.filteredSortedItems) { item in
        // Computed once, cached
    }
}
```

### Use task(id:) Modifier

```swift
// ✅ GOOD: Re-run task when id changes
.task(id: userId) {
    await viewModel.loadUser(id: userId)
}

// ❌ BAD: task without id doesn't re-run
.task {
    await viewModel.loadUser(id: userId)  // Won't re-run
}
```

## Ultrathink + Plan Mode

For complex tasks requiring deep reasoning:
1. Use `ultrathink` for enhanced thinking
2. Enable **Plan Mode** for structured approach
3. "Rev the engine" with multiple critique rounds
4. Use split role sub-agents for diverse analysis

## Build Troubleshooting

If build fails:
1. Use **build-error-resolver** agent
2. Analyze xcodebuild error messages
3. Fix incrementally
4. Verify after each fix
