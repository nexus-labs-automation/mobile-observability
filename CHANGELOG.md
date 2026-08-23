# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.0] - 2026-08-04

### Added
- **New Relic vendor support** (`references/platforms/newrelic.md`)
  - iOS (SPM 7.7.4+), Android (Gradle plugin 7.7.7), React Native, and Flutter SDK integration
  - Custom attributes, events, breadcrumbs, handled exceptions, and interaction traces
  - Auto-instrumented network capture and W3C distributed tracing on iOS/Android
  - Session replay support (iOS, Android, React Native; Flutter coming soon)
  - Symbolication: dSYM upload (iOS), ProGuard auto-upload via NR Gradle plugin (Android)
  - Recommended alert thresholds and example NRQL queries

### Changed
- Updated `agents/codebase-analyzer.md` with New Relic detection row
- Updated `commands/instrument.md` to accept `--vendor=newrelic`
- Updated `skills/symbolication-setup/SKILL.md` with New Relic verification entry
- Updated `skills/network-tracing/SKILL.md` with New Relic auto-instrumentation row
- Updated `skills/session-replay/SKILL.md` vendor configuration list
- Updated `references/otel-mobile.md` and `references/platforms/opentelemetry.md`
  with New Relic OTLP ingestion and session replay support
- Updated `references/instrumentation-patterns.md`, `references/ui-performance.md`,
  and `README.md` vendor lists to include New Relic
- Updated `.claude-plugin/plugin.json` and `marketplace.json` to v1.2.0

## [1.1.0] - 2025-12-22

### Added
- **Flutter platform support** (`references/flutter.md`)
  - Dart error handling (3-channel: Dart, Platform, Isolate)
  - Native bridge instrumentation patterns
  - Widget lifecycle and performance tracing
  - Dart hook with Flutter-specific anti-patterns
- **Firebase vendor support** (`references/platforms/firebase.md`)
  - Crashlytics integration for iOS/Android/Flutter
  - Analytics with custom events and user properties
  - Performance Monitoring with custom traces
  - Remote Config for feature flags
- **OpenTelemetry vendor support** (`references/platforms/opentelemetry.md`)
  - opentelemetry-swift v2.3.0
  - opentelemetry-android v1.0.0-rc.1
  - Decision tree for OTel vs vendor SDKs
- **Measure.sh vendor support** (`references/platforms/measure.md`)
  - Self-hosted mobile observability
  - iOS, Android, and Flutter SDK integration
  - Data sovereignty and privacy focus
- **Anti-pattern: localized strings in telemetry**
  - Added to `references/instrumentation-patterns.md`
  - Added hooks for Swift, Kotlin, TypeScript, Dart

### Changed
- Updated `agents/codebase-analyzer.md` with Flutter detection
- Updated `commands/instrument.md` with Flutter platform and new vendors

## [1.0.0] - 2025-12-20

### Added
- Initial release
- iOS native instrumentation guide
- Android native instrumentation guide
- React Native/Expo instrumentation guide
- Sentry, Datadog, Embrace, Bugsnag, Bitdrift vendor guides
- `/instrument` and `/audit` commands
- `codebase-analyzer` and `instrumentation-reviewer` agents
- 8 focused instrumentation skills
- Anti-pattern hooks for Swift, Kotlin, TypeScript
