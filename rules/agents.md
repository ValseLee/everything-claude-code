# Agent Orchestration

## Available Agents

Located in `~/.claude/agents/`:

| Agent | Purpose | When to Use |
|-------|---------|-------------|
| planner | Implementation planning | Complex features, refactoring |
| architect | System design | iOS architecture decisions |
| tdd-guide | Test-driven development | New features, bug fixes (Swift Testing) |
| code-reviewer | Code review | After writing Swift/SwiftUI code |
| security-reviewer | Security analysis | Keychain, ATS, data protection |
| build-error-resolver | Fix build errors | xcodebuild/SPM errors |
| e2e-runner | UI testing | XCUITest for critical flows |
| refactor-cleaner | Dead code cleanup | Periphery, SwiftLint |
| doc-updater | Documentation | Updating Swift docs |

## Immediate Agent Usage

No user prompt needed:
1. Complex feature requests - Use **planner** agent
2. Code just written/modified - Use **code-reviewer** agent
3. Bug fix or new feature - Use **tdd-guide** agent
4. Architectural decision - Use **architect** agent

## Parallel Task Execution

ALWAYS use parallel Task execution for independent operations:

```markdown
# GOOD: Parallel execution
Launch 3 agents in parallel:
1. Agent 1: Security analysis of AuthManager.swift
2. Agent 2: Performance review of ImageCache.swift
3. Agent 3: Code review of UserListView.swift

# BAD: Sequential when unnecessary
First agent 1, then agent 2, then agent 3
```

## Multi-Perspective Analysis

For complex problems, use split role sub-agents:
- Factual reviewer
- Senior iOS engineer
- Security expert
- Consistency reviewer
- Redundancy checker
