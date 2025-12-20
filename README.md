# Mobile Observability Plugin for Claude Code

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Plugin-blueviolet)](https://claude.ai/code)
[![Platform](https://img.shields.io/badge/Platform-iOS%20%7C%20Android%20%7C%20React%20Native-green)](https://github.com/calube/mobile-observability)

A Claude Code plugin that gives Claude expert knowledge in mobile app observability—crash reporting, performance monitoring, session replay, and instrumentation for iOS, Android, and React Native.

## Why This Plugin?

Mobile observability is hard:
- **Fragmented knowledge** across iOS, Android, and cross-platform frameworks
- **Vendor-specific patterns** that don't transfer between Sentry, Datadog, Embrace, etc.
- **Easy to miss** critical instrumentation (symbolication, breadcrumbs, user context)
- **Performance pitfalls** like blocking the main thread with telemetry

This plugin gives Claude deep expertise in mobile observability best practices, so you can:
- Get instrumentation plans tailored to your codebase
- Debug crashes with expert analysis
- Avoid common anti-patterns automatically
- Follow industry standards (OTel-compatible naming, JTBD framework)

## Who Is This For?

- **Mobile developers** adding observability to iOS/Android/React Native apps
- **Teams adopting** Sentry, Datadog, Embrace, or Bugsnag
- **Anyone debugging** mobile crashes, ANRs, or performance issues
- **Developers wanting** to instrument apps the right way from the start

## Prerequisites

- [Claude Code CLI](https://claude.ai/code) installed
- A mobile project (iOS, Android, or React Native)

## Installation

```bash
# Clone the plugin
git clone https://github.com/calube/mobile-observability.git

# Install in Claude Code
claude plugins install ./mobile-observability
```

## Quick Start

### 1. Generate an Instrumentation Plan

Open Claude Code in your mobile project and run:

```
/instrument ios
```

Claude will analyze your codebase and generate a prioritized instrumentation plan covering crashes, performance, breadcrumbs, and user context.

### 2. Debug a Crash

```
/diagnose crash
```

Then paste your crash log. Claude will identify the root cause, explain what happened, and suggest fixes.

### 3. Audit Existing Telemetry

```
/audit
```

Claude scans your codebase for existing instrumentation and identifies gaps.

### 4. Ask Natural Questions

The plugin's skills activate automatically when you ask questions like:

- "Why is my app crashing on launch?"
- "How do I track screen load time?"
- "Set up session replay with Sentry"
- "What should I measure in my checkout flow?"

## Commands

| Command | Description |
|---------|-------------|
| `/instrument [platform]` | Generate instrumentation plan for a codebase |
| `/diagnose [type]` | Analyze crash logs, ANRs, and performance issues |
| `/audit [path]` | Scan for existing instrumentation and gaps |

### Command Examples

```bash
# iOS instrumentation plan
/instrument ios

# Android with specific vendor
/instrument android --vendor=sentry

# Analyze a crash
/diagnose crash

# Analyze performance issue
/diagnose performance

# Audit current directory
/audit

# Audit specific path
/audit ./src
```

## Skills

Eight focused skills provide expert guidance:

| Skill | Triggers When You Ask About |
|-------|----------------------------|
| `instrumentation-planning` | "What should I measure?", "How do I instrument..." |
| `crash-debugging` | "Why is my app crashing?", "Debug this crash log" |
| `performance-optimization` | "App is slow", "Optimize performance" |
| `session-replay` | "Set up session replay", "Visual debugging" |
| `interaction-latency` | "Track button taps", "Measure response time" |
| `navigation-latency` | "Track screen loads", "Measure TTI" |
| `network-tracing` | "Trace API calls", "Monitor network requests" |
| `user-journey-tracking` | "Track funnels", "Measure conversion" |

## Agents

| Agent | Use Case |
|-------|----------|
| `codebase-analyzer` | Explores your codebase to understand architecture and find instrumentation opportunities |
| `issue-analyzer` | Deep-dives into crash logs, stack traces, and performance issues |

**Use agents directly:**
```
Launch the codebase-analyzer agent to analyze my iOS app
```

## Anti-Pattern Hooks

The plugin automatically warns you about common mistakes when editing mobile code:

- **PII in logs** — Never log email, phone, or user-provided text
- **Sync telemetry** — Never block main thread for analytics
- **Force unwraps in crash handlers** — Avoid `!` (Swift) or `!!` (Kotlin) in error handling
- **Missing offline handling** — Queue events when offline

Works with `.swift`, `.kt`, and `.ts` files.

## Reference Content

### Platforms
- **iOS** — Swift/SwiftUI, MetricKit, os_signpost
- **Android** — Kotlin/Compose, Perfetto, ANR detection
- **React Native** — Expo, Hermes, source maps, native modules

### Vendors
- **Sentry** — Full setup guide with session replay
- **Datadog** — RUM, APM, and log correlation
- **Embrace** — Mobile-first observability
- **Bugsnag** — Error monitoring setup

### Topics
| Category | Coverage |
|----------|----------|
| Crashes | Error types, ANR, breadcrumbs, symbolication |
| Performance | App start, screen load, network, TTI |
| UI | Navigation, scroll, animations |
| Data | SQLite, Core Data, Room, Realm |
| Sessions | Session replay, privacy masking |
| Mobile | Offline-first, battery, fragmentation |

### Code Templates
Ready-to-use templates for:
- Screen load tracking
- Error boundaries
- Breadcrumb managers
- Navigation tracking

## Plugin Structure

```
mobile-observability/
├── commands/           # /instrument, /diagnose, /audit
├── agents/             # codebase-analyzer, issue-analyzer
├── skills/             # 8 focused skills
├── hooks/              # Anti-pattern detection
├── references/         # 15+ guides + templates + vendor docs
└── .claude-plugin/     # Plugin manifest
```

## Contributing

Contributions welcome! Areas that would be helpful:

- Additional vendor guides (New Relic, AppDynamics, etc.)
- More code templates
- Platform-specific examples
- Bug reports and feedback

## License

MIT — see [LICENSE](LICENSE)

---

Built with Claude Code. If you find this useful, star the repo!
