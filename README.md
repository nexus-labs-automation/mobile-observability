# Mobile Observability Plugin for Claude Code

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Plugin-blueviolet)](https://claude.ai/code)
[![Platform](https://img.shields.io/badge/Platform-iOS%20%7C%20Android%20%7C%20React%20Native-green)](https://github.com/calube/mobile-observability)

A Claude Code plugin that teaches Claude mobile observability best practices — what to instrument, with what data, so your telemetry is actually actionable.

## What This Plugin Does

This plugin gives Claude expert knowledge in mobile instrumentation so it can help you:

- **Decide what to measure** — not everything, just what matters for debugging and understanding user behavior
- **Attach the right context** — so errors and events are actionable, not just noise
- **Avoid common mistakes** — high cardinality tags, PII leaks, unbounded payloads, missing correlation
- **Follow platform patterns** — iOS (Swift/SwiftUI), Android (Kotlin/Compose), React Native

## The Problem It Solves

Most teams instrument poorly because they don't know what data makes telemetry actionable. They either:

- **Instrument too little** → can't debug production issues ("why did onboarding fail for 12% of users?")
- **Instrument too much** → noise, high costs, battery drain
- **Attach wrong data** → errors without context, high-cardinality metrics, missing user journey correlation

This plugin teaches Claude "User-Focused Observability" — linking user intentions and outcomes with app telemetry so you can answer "why did users fail?" See [`references/user-focused-observability.md`](references/user-focused-observability.md) for the methodology.

## Who Is This For?

- Mobile developers adding observability to iOS, Android, or React Native apps
- Teams setting up Sentry, Datadog, Embrace, or Bugsnag
- Anyone who wants to instrument their app correctly the first time

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
- "What should I measure in my onboarding flow?"

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

Nine focused skills provide expert guidance:

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
| `symbolication-setup` | "dSYM", "ProGuard", "source maps", "stack traces unreadable" |

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

The plugin warns about observability anti-patterns when editing `.swift`, `.kt`, or `.ts` files:

| Anti-Pattern | Why It Matters |
|--------------|----------------|
| **High cardinality tags** | User IDs or UUIDs as metric labels explode storage costs |
| **Unbounded payloads** | Attaching entire state objects bloats events and hits limits |
| **Unstructured logging** | String interpolation breaks log querying and aggregation |
| **Inconsistent naming** | `fetchUser` vs `user.fetch` breaks dashboards and alerts |
| **PII in telemetry** | Compliance risk; never log emails, phones, or free text |
| **Missing context** | Events without screen/session/version are hard to debug |

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
| Business Logic | Decisions, rules, state machines, background jobs |
| Feature Flags | Flag evaluation, change events, what not to do |
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
├── skills/             # 9 focused skills
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
