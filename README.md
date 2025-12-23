# Mobile Observability Plugin

Teaches Claude how to instrument mobile apps correctly — what to measure, what context to attach, and what mistakes to avoid.

## The Problem

Most mobile teams instrument poorly:

- **Too little** — Can't debug production issues
- **Too much** — Noise, cost, battery drain
- **Wrong context** — Errors without enough data to act on

This plugin teaches Claude **User-Focused Observability**: linking user intentions with app telemetry so you can answer *"Why did users fail to complete their goal?"*

## Installation

```bash
claude plugin marketplace add calube/mobile-observability
claude plugin install mobile-observability
```

## Usage

### Commands

```
/instrument ios
/audit features/checkout
```

### Natural Prompts

```
"How should I instrument this payment flow?"
"What context should I attach to crashes?"
"Set up session replay with Bitdrift"
"Review this code for observability anti-patterns"
"What's missing from our crash reporting setup?"
```

## Commands

| Command | Description |
|---------|-------------|
| `/instrument [platform]` | Generate prioritized instrumentation plan for iOS, Android, React Native, or Flutter |
| `/audit [path]` | Scan existing code for instrumentation gaps and anti-patterns |

## Agents

Agent definitions that Claude reads and follows when performing analysis tasks.

### codebase-analyzer

Explores mobile codebases to understand architecture and identify instrumentation opportunities.

**Focus Areas:**
- Platform and architecture detection (MVVM, TCA, Clean Architecture)
- Existing telemetry SDK inventory
- Entry points and key user flows
- Network, persistence, and state management layers

**Output:**
- Platform summary with language/version
- Architecture pattern identification
- Existing SDK coverage assessment
- Gap analysis with priority ranking

### instrumentation-reviewer

Reviews code changes for observability issues before they ship.

**Focus Areas:**
- Anti-patterns (PII leaks, high cardinality, sync telemetry)
- Missing context (no user ID, no session, no screen)
- Naming consistency
- Vendor best practices

**Output:**
- Issues by severity with file:line references
- Specific fixes with code examples
- Vendor guideline references

## Skills

8 skills that activate automatically based on context:

| Skill | Trigger |
|-------|---------|
| `instrumentation-planning` | "What should I measure?" |
| `crash-instrumentation` | "How to capture crashes with context" |
| `session-replay` | "Set up session replay" |
| `interaction-latency` | "Track button response time" |
| `navigation-latency` | "Track screen load time" |
| `network-tracing` | "Trace API requests" |
| `user-journey-tracking` | "Track user funnels" |
| `symbolication-setup` | "Configure dSYM upload" |

## When to Use

**Use this plugin for:**
- Adding observability to a new or existing mobile app
- Setting up crash reporting, performance monitoring, or session replay
- Choosing between vendors (Sentry, Datadog, Embrace, Bitdrift, Firebase, etc.)
- Reviewing instrumentation code for anti-patterns
- Understanding what context to attach to errors

**Don't use for:**
- Backend/server observability
- Web frontend (different patterns)
- General logging questions unrelated to mobile

## Directory Structure

```
mobile-observability/
├── commands/       # /instrument, /audit
├── agents/         # codebase-analyzer, instrumentation-reviewer
├── skills/         # 8 instrumentation skills
├── hooks/          # Anti-pattern warnings for Swift/Kotlin/TypeScript/Dart
└── references/     # 25+ guides covering methodology, platforms, vendors
```

## Philosophy

1. **Start with crashes** — Get crash reporting and symbolication right before adding anything else
2. **Context over volume** — One error with full context beats 100 without
3. **User intent + system state** — Link what users tried to do with device conditions (memory, network, battery)
4. **Avoid anti-patterns early** — PII leaks and high cardinality are expensive to fix later

## Author

[Caleb Davis](https://github.com/calube)

## License

MIT — see [LICENSE](LICENSE)
