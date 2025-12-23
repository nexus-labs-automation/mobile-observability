# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
