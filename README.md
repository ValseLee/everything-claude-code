# Everything Claude Code (iOS Edition)

**The complete collection of Claude Code configs for iOS development.**

This repo contains production-ready agents, skills, hooks, commands, rules, and MCP configurations optimized for iOS development with Swift, SwiftUI, and Xcode.

**Target Stack:**
- iOS 18+, Swift 6.2
- SwiftUI with View-ViewModel pattern
- SwiftData for persistence
- Swift Testing framework
- Clean Architecture + Modular Architecture

---

## Read the Full Guide First

**Before diving into these configs, read the complete guide on X:**


<img width="592" height="445" alt="image" src="https://github.com/user-attachments/assets/1a471488-59cc-425b-8345-5245c7efbcef" />


**[The Shorthand Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2012378465664745795)**



The guide explains:
- What each config type does and when to use it
- How to structure your Claude Code setup
- Context window management (critical for performance)
- Parallel workflows and advanced techniques
- The philosophy behind these configs

**This repo is configs only! Tips, tricks and more examples are in my X articles and videos (links will be appended to this readme as it evolves).**

---

## What's Inside

```
everything-claude-code/
|-- agents/           # Specialized subagents for delegation
|   |-- planner.md           # Feature implementation planning
|   |-- architect.md         # iOS system design decisions
|   |-- tdd-guide.md         # Swift Testing TDD workflow
|   |-- code-reviewer.md     # Swift/SwiftUI code review
|   |-- security-reviewer.md # iOS security analysis
|   |-- build-error-resolver.md  # xcodebuild/SPM errors
|   |-- e2e-runner.md        # XCUITest E2E testing
|   |-- refactor-cleaner.md  # Swift dead code cleanup
|   |-- doc-updater.md       # Documentation sync
|
|-- skills/           # Workflow definitions and domain knowledge
|   |-- swift-coding-standards.md  # Swift API Guidelines
|   |-- swift-concurrency.md       # async/await, Actor, Sendable
|   |-- swiftui-patterns.md        # iOS 18+ SwiftUI patterns
|   |-- swiftdata.md               # #Predicate, ModelActor
|   |-- xcode-project.md           # Workspace, SPM, targets
|   |-- ios-architecture.md        # Clean + Modular Architecture
|   |-- project-guidelines-example.md  # Example project skill
|   |-- tdd-workflow/              # TDD methodology
|   |-- security-review/           # iOS security checklist
|
|-- commands/         # Slash commands for quick execution
|   |-- tdd.md              # /tdd - Swift Testing TDD
|   |-- plan.md             # /plan - Implementation planning
|   |-- e2e.md              # /e2e - XCUITest generation
|   |-- code-review.md      # /code-review - Swift review
|   |-- build-fix.md        # /build-fix - xcodebuild errors
|   |-- refactor-clean.md   # /refactor-clean - Periphery cleanup
|   |-- test-coverage.md    # /test-coverage - xccov analysis
|   |-- update-codemaps.md  # /update-codemaps - Module docs
|   |-- update-docs.md      # /update-docs - Sync documentation
|
|-- rules/            # Always-follow guidelines
|   |-- security.md         # Keychain, ATS, Data Protection
|   |-- coding-style.md     # Swift immutability, Codable
|   |-- testing.md          # Swift Testing, 80% coverage
|   |-- git-workflow.md     # Commit format, PR process
|   |-- agents.md           # When to delegate to subagents
|   |-- performance.md      # SwiftUI optimization
|   |-- patterns.md         # View-ViewModel, Repository
|   |-- hooks.md            # Hook documentation
|
|-- hooks/            # Trigger-based automations
|   |-- hooks.json          # SwiftFormat, SwiftLint hooks
|
|-- mcp-configs/      # MCP server configurations
|   |-- mcp-servers.json    # GitHub, memory, filesystem
|
|-- examples/         # Example configurations
    |-- CLAUDE.md           # Example project-level config
    |-- user-CLAUDE.md      # Example user-level config
    |-- statusline.json     # Custom status line config
```

---

## Quick Start

### 1. Copy what you need

```bash
# Clone the repo
git clone https://github.com/affaan-m/everything-claude-code.git

# Copy agents to your Claude config
cp everything-claude-code/agents/*.md ~/.claude/agents/

# Copy rules
cp everything-claude-code/rules/*.md ~/.claude/rules/

# Copy commands
cp everything-claude-code/commands/*.md ~/.claude/commands/

# Copy skills
cp -r everything-claude-code/skills/* ~/.claude/skills/
```

### 2. Add hooks to settings.json

Copy the hooks from `hooks/hooks.json` to your `~/.claude/settings.json`.

### 3. Configure MCPs

Copy desired MCP servers from `mcp-configs/mcp-servers.json` to your `~/.claude.json`.

**Important:** Replace `YOUR_*_HERE` placeholders with your actual API keys.

### 4. Read the guide

Seriously, [read the guide](https://x.com/affaanmustafa/status/2012378465664745795). These configs make 10x more sense with context.

---

## Key Concepts

### Agents

Subagents handle delegated tasks with limited scope. Example:

```markdown
---
name: code-reviewer
description: Reviews Swift code for quality, security, and SwiftUI patterns
tools: Read, Grep, Glob, Bash
model: opus
---

You are a senior iOS code reviewer...
```

### Skills

Skills are workflow definitions invoked by commands or agents:

```markdown
# TDD Workflow (Swift Testing)

1. Define protocols first
2. Write failing tests with @Test (RED)
3. Implement minimal code (GREEN)
4. Refactor (IMPROVE)
5. Verify 80%+ coverage with xccov
```

### Hooks

Hooks fire on tool events. Example - warn about print():

```json
{
  "matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\\\.swift$\"",
  "hooks": [{
    "type": "command",
    "command": "#!/bin/bash\ngrep -n 'print(' \"$file_path\" && echo '[Hook] Remove print()' >&2"
  }]
}
```

### Rules

Rules are always-follow guidelines. Keep them modular:

```
~/.claude/rules/
  security.md      # Keychain for secrets, ATS enabled
  coding-style.md  # let over var, no force unwrap
  testing.md       # Swift Testing, coverage requirements
```

---

## iOS-Specific Patterns

### View-ViewModel Pattern

```swift
struct HomeView: View {
    @State private var viewModel = ViewModel()
    var body: some View { /* ... */ }
}

extension HomeView {
    @Observable @MainActor
    final class ViewModel {
        // State and actions
    }
}
```

### Clean Architecture Layers

```
UI Layer (SwiftUI Views, ViewModels)
    |
Domain Layer (Use Cases, Entities, Repository Protocols)
    |
Data Layer (Repository Implementations, Network, SwiftData)
```

### Key Principles

1. **Pure Functions** - minimize side effects
2. **No Forced Unwrap (!)** - except compile-time guaranteed values
3. **Stateless Views** - all state in ViewModel
4. **TDD** - write tests first
5. **80% coverage** minimum

---

## Contributing

**Contributions are welcome and encouraged.**

This repo is meant to be a community resource. If you have:
- Useful agents or skills
- Clever hooks
- Better MCP configurations
- Improved rules

Please contribute! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### Ideas for Contributions

- Additional SwiftUI patterns
- watchOS/tvOS/visionOS specific configs
- CI/CD configurations (Xcode Cloud, Fastlane)
- Advanced Swift Testing strategies
- SwiftData migration patterns

---

## Background

I've been using Claude Code since the experimental rollout. Won the Anthropic x Forum Ventures hackathon in Sep 2025 building [zenith.chat](https://zenith.chat) with [@DRodriguezFX](https://x.com/DRodriguezFX) - entirely using Claude Code.

These configs are battle-tested across multiple production applications.

---

## Important Notes

### Context Window Management

**Critical:** Don't enable all MCPs at once. Your 200k context window can shrink to 70k with too many tools enabled.

Rule of thumb:
- Have 20-30 MCPs configured
- Keep under 10 enabled per project
- Under 80 tools active

Use `disabledMcpServers` in project config to disable unused ones.

### Customization

These configs work for my workflow. You should:
1. Start with what resonates
2. Modify for your stack
3. Remove what you don't use
4. Add your own patterns

---

## Links

- **Full Guide:** [The Shorthand Guide to Everything Claude Code](https://x.com/affaanmustafa/status/2012378465664745795)
- **Follow:** [@affaanmustafa](https://x.com/affaanmustafa)
- **zenith.chat:** [zenith.chat](https://zenith.chat)

---

## License

MIT - Use freely, modify as needed, contribute back if you can.

---

**Star this repo if it helps. Read the guide. Build something great.**
