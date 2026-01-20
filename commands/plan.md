---
description: Restate requirements, assess risks, and create step-by-step implementation plan. WAIT for user CONFIRM before touching any code.
---

# Plan Command

This command invokes the **planner** agent to create a comprehensive implementation plan before writing any code for iOS/Swift projects.

## What This Command Does

1. **Restate Requirements** - Clarify what needs to be built
2. **Identify Risks** - Surface potential issues and blockers
3. **Create Step Plan** - Break down implementation into phases
4. **Wait for Confirmation** - MUST receive user approval before proceeding

## When to Use

Use `/plan` when:
- Starting a new feature
- Making significant architectural changes
- Working on complex refactoring
- Multiple files/modules will be affected
- Requirements are unclear or ambiguous

## How It Works

The planner agent will:

1. **Analyze the request** and restate requirements in clear terms
2. **Break down into phases** with specific, actionable steps
3. **Identify dependencies** between modules
4. **Assess risks** and potential blockers
5. **Estimate complexity** (High/Medium/Low)
6. **Present the plan** and WAIT for your explicit confirmation

## Example Usage

```
User: /plan I need to add real-time push notifications when portfolio values change significantly

Agent (planner):
# Implementation Plan: Real-Time Portfolio Push Notifications

## Requirements Restatement
- Send push notifications to users when their portfolio value changes by more than 5%
- Support both foreground and background notifications
- Include portfolio change amount and direction in notification
- Allow users to configure notification thresholds in settings

## Implementation Phases

### Phase 1: Data Layer
- Create `NotificationPreference` SwiftData model
  - `id: UUID`
  - `userID: String`
  - `threshold: Double` (default: 0.05)
  - `isEnabled: Bool`
  - `createdAt: Date`
- Create `PortfolioChange` model for tracking changes
- Add migration for new schema

### Phase 2: Domain Layer
- Create `NotificationService` protocol and implementation
  - `func shouldNotify(change: PortfolioChange) -> Bool`
  - `func scheduleNotification(for change: PortfolioChange) async throws`
- Create `PortfolioMonitor` actor for background monitoring
  - Track previous values
  - Calculate percentage changes
  - Coordinate with NotificationService

### Phase 3: Infrastructure Layer
- Configure UNUserNotificationCenter
- Request notification permissions
- Create notification content builder
- Handle notification actions (deep link to portfolio)
- Register for background app refresh

### Phase 4: Presentation Layer
- Create `NotificationSettingsView`
- Create `NotificationSettingsViewModel` (@Observable)
- Add notification preferences to Settings tab
- Handle notification permission states in UI

### Phase 5: Integration
- Wire up PortfolioMonitor with app lifecycle
- Handle `scenePhase` changes for background monitoring
- Test notification delivery in various states
- Add analytics tracking for notifications

## Dependencies
- UNUserNotificationCenter (iOS)
- BackgroundTasks framework (for background refresh)
- SwiftData (for persistence)

## Risks
- HIGH: Background execution limits may delay notifications
- MEDIUM: Users may not grant notification permissions
- MEDIUM: Rate limiting needed to prevent notification spam
- LOW: Deep link handling complexity

## Architecture Considerations

```
┌─────────────────────────────────────────────────────┐
│                  Presentation                        │
│  NotificationSettingsView ← NotificationSettingsVM   │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│                    Domain                           │
│  NotificationService  │  PortfolioMonitor (Actor)   │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│                     Data                            │
│  NotificationPreference  │  PortfolioChange         │
│              (SwiftData Models)                     │
└─────────────────────────────────────────────────────┘
```

## Estimated Complexity: MEDIUM
- Data Layer: Small
- Domain Layer: Medium
- Presentation: Small
- Integration: Medium

**WAITING FOR CONFIRMATION**: Proceed with this plan? (yes/no/modify)
```

## Important Notes

**CRITICAL**: The planner agent will **NOT** write any code until you explicitly confirm the plan with "yes" or "proceed" or similar affirmative response.

If you want changes, respond with:
- "modify: [your changes]"
- "different approach: [alternative]"
- "skip phase 2 and do phase 3 first"

## Plan Output Format

Every plan should include:

1. **Requirements Restatement** - Clear understanding of what's needed
2. **Implementation Phases** - Ordered steps with specific deliverables
3. **Dependencies** - Frameworks, services, or modules required
4. **Risks** - Potential issues with severity ratings
5. **Architecture Diagram** - Visual representation when helpful
6. **Complexity Estimate** - High/Medium/Low per phase

## iOS-Specific Considerations

When planning iOS features, always consider:

**SwiftUI Patterns:**
- View-ViewModel separation
- @Observable for state management
- Environment for dependency injection

**Concurrency:**
- Actor isolation for shared state
- MainActor for UI updates
- Structured concurrency with async/await

**Data Persistence:**
- SwiftData models and containers
- Migration strategies
- iCloud sync requirements

**Security:**
- Keychain for sensitive data
- App Transport Security
- Data Protection classes

## Integration with Other Commands

After planning:
- Use `/tdd` to implement with test-driven development
- Use `/build-fix` if build errors occur
- Use `/code-review` to review completed implementation

## Related Agents

This command invokes the `planner` agent located at:
`~/.claude/agents/planner.md`
