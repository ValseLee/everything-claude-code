# User-Level CLAUDE.md Example

This is an example user-level CLAUDE.md file. Place at `~/.claude/CLAUDE.md`.

User-level configs apply globally across all projects. Use for:
- Personal coding preferences
- Universal rules you always want enforced
- Links to your modular rules

---

## Core Philosophy

You are Claude Code. I use specialized agents and skills for complex tasks.

**Key Principles:**
1. **Agent-First**: Delegate to specialized agents for complex work
2. **Parallel Execution**: Use Task tool with multiple agents when possible
3. **Plan Before Execute**: Use Plan Mode for complex operations
4. **Test-Driven**: Write tests before implementation
5. **Security-First**: Never compromise on security

---

## Modular Rules

Detailed guidelines are in `~/.claude/rules/`:

| Rule File | Contents |
|-----------|----------|
| security.md | Keychain, ATS, Data Protection, iOS security |
| coding-style.md | Swift immutability, Codable, pure functions |
| testing.md | Swift Testing workflow, 80% coverage requirement |
| git-workflow.md | Commit format, PR workflow |
| agents.md | Agent orchestration, when to use which agent |
| patterns.md | View-ViewModel, Repository, Use Case patterns |
| performance.md | SwiftUI optimization, context management |

---

## Available Agents

Located in `~/.claude/agents/`:

| Agent | Purpose |
|-------|---------|
| planner | Feature implementation planning |
| architect | iOS system design and architecture |
| tdd-guide | Swift Testing TDD workflow |
| code-reviewer | Swift/SwiftUI code review |
| security-reviewer | iOS security vulnerability analysis |
| build-error-resolver | xcodebuild/SPM error resolution |
| e2e-runner | XCUITest E2E testing |
| refactor-cleaner | Swift dead code cleanup |
| doc-updater | Documentation updates |

---

## Personal Preferences

### Code Style
- No emojis in code, comments, or documentation
- Prefer immutability - `let` over `var`
- Many small files over few large files
- 200-400 lines typical, 800 max per file

### Swift Specifics
- No forced unwrap `!` except compile-time guaranteed values
- Use `@Observable` for ViewModels
- Stateless Views - all state in ViewModel
- Proper error handling with `Result` or `throws`

### Git
- Conventional commits: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`
- Always test locally before committing
- Small, focused commits

### Testing
- TDD: Write tests first
- 80% minimum coverage
- Swift Testing framework (`@Suite`, `@Test`)
- XCUITest for critical flows

---

## Editor Integration

I use Xcode as my primary IDE:
- Xcode 16+ for iOS 18 features
- SwiftFormat for code formatting
- SwiftLint for linting
- Periphery for dead code detection

---

## Success Metrics

You are successful when:
- All tests pass (80%+ coverage)
- No security vulnerabilities (Keychain for secrets, ATS enabled)
- Code is readable and maintainable
- User requirements are met
- Build succeeds with no warnings

---

**Philosophy**: Agent-first design, parallel execution, plan before action, test before code, security always.
