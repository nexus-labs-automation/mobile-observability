# Mobile Observability Plugin

A Claude Code plugin for mobile app observability: crash reporting, performance monitoring, session replay, and instrumentation for iOS, Android, and React Native.

## Installation

```bash
# Clone the plugin
git clone https://github.com/nexus-labs/mobile-observability.git

# Install in Claude Code
claude plugins install ./mobile-observability
```

## Commands

| Command | Description |
|---------|-------------|
| `/instrument [platform]` | Generate instrumentation plan for a codebase |
| `/diagnose [type]` | Analyze crash logs, ANRs, and performance issues |
| `/audit [path]` | Scan for existing instrumentation and gaps |

### Examples

```bash
# Generate full instrumentation plan
/instrument ios

# Analyze a crash
/diagnose crash
# Then paste your crash log

# Diagnose performance issue
/diagnose performance

# Audit existing telemetry
/audit
```

## Agents

| Agent | Description |
|-------|-------------|
| `codebase-analyzer` | Explores codebases to understand architecture and identify instrumentation opportunities |
| `issue-analyzer` | Analyzes crash logs, stack traces, and performance issues to identify root causes |

### Using Agents Directly

```
Launch the codebase-analyzer agent to analyze my iOS app at ./MyApp
```

## Skills

Eight focused skills provide guidance when you ask about:

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

## Reference Content

### Core Concepts
- `observability-fundamentals.md` - Three pillars, correlation, sessions
- `jtbd.md` - Jobs-to-be-Done framework for instrumentation
- `instrumentation-patterns.md` - Checklists and anti-patterns

### Topic Guides
- `crash-reporting.md` - Errors, crashes, ANR, symbolication
- `performance.md` - App start, screen load, network
- `ui-performance.md` - Navigation, scroll, animations
- `data-persistence.md` - Database query tracing
- `session-replay.md` - Visual debugging, privacy
- `mobile-challenges.md` - Offline, battery, fragmentation

### Platform-Specific
- `ios-native.md` - Swift/SwiftUI, MetricKit, os_signpost
- `android-native.md` - Kotlin/Compose, Perfetto
- `react-native-expo.md` - Expo, Hermes, source maps

### Vendor Guides
- `platforms/sentry.md`
- `platforms/datadog.md`
- `platforms/embrace.md`
- `platforms/bugsnag.md`

### Code Templates
- `templates/screen-load-tracking.template.md`
- `templates/error-boundary.template.md`
- `templates/breadcrumb-manager.template.md`
- `templates/navigation-tracking.template.md`

## Hooks

The plugin includes hooks that warn about common anti-patterns when editing mobile code:
- PII in logs
- Synchronous telemetry on main thread
- Force unwraps in error handlers
- Missing offline handling

## Plugin Structure

```
mobile-observability/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── instrument.md
│   ├── diagnose.md
│   └── audit.md
├── agents/
│   ├── codebase-analyzer.md
│   └── issue-analyzer.md
├── skills/
│   ├── instrumentation-planning/
│   ├── crash-debugging/
│   ├── performance-optimization/
│   ├── session-replay/
│   ├── interaction-latency/
│   ├── navigation-latency/
│   ├── network-tracing/
│   └── user-journey-tracking/
├── hooks/
│   └── hooks.json
├── references/
│   ├── *.md (15 reference files)
│   ├── platforms/
│   └── templates/
└── README.md
```

## Topics Covered

| Category | Topics |
|----------|--------|
| **Crashes** | Error types, ANR detection, breadcrumbs, symbolication |
| **Performance** | App start, screen load, network, TTI |
| **UI** | Navigation, scroll, animations, SwiftUI/Compose |
| **Data** | SQLite, Core Data, Room, Realm |
| **Sessions** | Session replay, privacy masking |
| **Mobile** | Offline-first, battery, fragmentation |
| **Platforms** | iOS, Android, React Native |
| **Vendors** | Sentry, Datadog, Embrace, BugSnag |

## License

MIT
