# Mobile Observability Plugin

## Project Context

Claude Code plugin for mobile app observability. Provides commands, agents, skills, and hooks for iOS, Android, and React Native instrumentation.

## Plugin Components

| Component | Location | Description |
|-----------|----------|-------------|
| Commands | `commands/` | `/instrument`, `/diagnose`, `/audit` |
| Agents | `agents/` | `codebase-analyzer`, `issue-analyzer` |
| Skills | `skills/` | 8 focused skills (see below) |
| Hooks | `hooks/hooks.json` | Anti-pattern warnings for Swift/Kotlin/TypeScript |
| References | `references/` | 15 reference files + templates + vendor guides |

## Key Commands

```bash
/instrument ios              # Generate instrumentation plan
/diagnose                    # Analyze crash log or performance issue
/audit                       # Scan existing instrumentation
```

## Skills

| Skill | Trigger |
|-------|---------|
| `instrumentation-planning` | "What should I measure?" |
| `crash-debugging` | "Why is my app crashing?" |
| `performance-optimization` | "Why is my app slow?" |
| `session-replay` | "Set up session replay" |
| `interaction-latency` | "Track button response time" |
| `navigation-latency` | "Track screen load time" |
| `network-tracing` | "Trace API requests" |
| `user-journey-tracking` | "Track user funnels" |

## Reference Navigation

| Topic | File |
|-------|------|
| What to measure | `references/jtbd.md` |
| Instrumentation checklist | `references/instrumentation-patterns.md` |
| Crashes | `references/crash-reporting.md` |
| Performance | `references/performance.md`, `references/ui-performance.md` |
| Database | `references/data-persistence.md` |
| iOS | `references/ios-native.md` |
| Android | `references/android-native.md` |
| React Native | `references/react-native-expo.md` |
| OpenTelemetry | `references/otel-mobile.md` |
| Vendors | `references/platforms/*.md` |

## Development

```bash
# Test plugin locally
claude plugins install ./

# Verify structure
ls -la .claude-plugin/ commands/ agents/ skills/ hooks/ references/
```
