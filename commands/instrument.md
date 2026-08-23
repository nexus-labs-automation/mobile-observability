# Instrument Command

Generate a comprehensive instrumentation plan for a mobile codebase.

## Usage

```
/instrument [platform] [--vendor=<vendor>]
```

**Arguments:**
- `platform`: `ios`, `android`, `react-native`, `flutter`, or `auto` (detect from codebase)
  - Invalid values: Error with "Invalid platform. Use: ios|android|react-native|flutter|auto"
- `--vendor`: Optional. `sentry`, `datadog`, `embrace`, `bugsnag`, `bitdrift`, `firebase`, `newrelic`, `opentelemetry`, `measure`
  - Format: `--vendor=sentry` or `--vendor sentry` (both accepted)
  - Invalid values: Warning with "Unknown vendor, using generic patterns"

**Examples:**
```
/instrument ios
/instrument android --vendor=sentry
/instrument flutter --vendor=firebase
/instrument auto
```

---

## Workflow

### Step 1: Detect Platform

If platform is `auto` or not specified:
1. Search for platform indicators:
   - iOS: `*.swift`, `*.xcodeproj`, `Podfile`, `Package.swift`
   - Android: `*.kt`, `build.gradle`, `AndroidManifest.xml`
   - React Native: `react-native` in `package.json`, `metro.config.js`
   - Flutter: `flutter` in `pubspec.yaml`, `lib/main.dart`
2. If multiple platforms detected (e.g., React Native or Flutter with native modules):
   - Default to highest-level platform (Flutter/React Native > native)
   - Inform user: "Detected [Flutter/React Native] with iOS/Android modules. Use `/instrument [flutter/react-native]` or `/instrument ios` for native-only."
3. If no platform detected:
   - Error: "Unable to detect platform. Please specify: `/instrument ios|android|react-native|flutter`"
   - Show project structure to help diagnose
4. Confirm with user if ambiguous

### Step 2: Analyze Codebase

Launch the `codebase-analyzer` agent to identify:
- Entry points (AppDelegate, Application class, App component)
- Architecture patterns (MVVM, TCA, MVI, etc.)
- Navigation layer
- Network layer
- Data persistence
- Existing telemetry (if any)

### Step 3: Load Reference Context

Based on detected platform, read:

**Always:**
- `references/jtbd.md` - Jobs-to-be-Done framework
- `references/instrumentation-patterns.md` - Instrumentation checklist

**Platform-specific:**
- `references/ios-native.md` | `references/android-native.md` | `references/react-native-expo.md` | `references/flutter.md`
- `references/performance.md`
- `references/crash-reporting.md`

**Vendor-specific (if specified):**
- `references/platforms/{vendor}.md`

### Step 4: Apply JTBD Framework

Identify user jobs for this codebase and map to instrumentation:

| User Job | Success Metric | Failure Signal |
|----------|----------------|----------------|
| [From JTBD analysis] | [Completion, duration] | [Error, abandonment] |

Reference: `skills/instrumentation-planning/SKILL.md`

### Step 5: Generate Instrumentation Plan

Output format:
```markdown
## Instrumentation Plan: [Project Name]

### Overview
- **Platform:** [iOS/Android/React Native]
- **Architecture:** [Pattern detected]
- **Vendor:** [Sentry/Datadog/etc. or "Vendor-agnostic"]
- **Existing Telemetry:** [SDKs found or "None"]

### User Jobs to Instrument

| Job | Success Metric | Failure Signal | Priority |
|-----|----------------|----------------|----------|
| [Core user flow] | [Metric] | [Signal] | P0 |

### Implementation Tiers

#### Tier 1: Foundation (P0)
- [ ] SDK integration in [AppDelegate/Application]
- [ ] Symbolication setup ([dSYM/ProGuard])
- [ ] User context on authentication

#### Tier 2: Core Performance (P1)
- [ ] App start tracking
- [ ] Screen load TTI for key screens
- [ ] Network request tracing

#### Tier 3: Context (P1)
- [ ] Navigation breadcrumbs
- [ ] User action breadcrumbs
- [ ] Error context enrichment

#### Tier 4: Business Metrics (P2)
- [ ] Key funnel instrumentation
- [ ] Feature usage tracking

### Key Files to Modify

| File | Changes | Priority |
|------|---------|----------|
| [path] | [specific changes] | [P0/P1/P2] |

### Code Snippets

[Production-ready, platform-specific code for each tier]
```

### Step 6: Offer Implementation

After presenting plan:
```
Ready to implement? Options:
1. Start with Tier 1 (foundation)
2. Generate detailed code for a specific item
3. Export plan to file
```

---

## Error Handling

- **Platform detection fails**: Show directory structure and prompt for explicit platform
- **Agent timeout**: Retry with smaller scope, or suggest `/audit` for quick scan
- **Missing references**: Continue with available references, note gaps in output

## Skills Used

- `instrumentation-planning` - JTBD framework and prioritization
- `crash-instrumentation` - When implementing crash tracking
- `network-tracing` - When implementing network instrumentation
- `navigation-latency` - When implementing screen tracking
- `interaction-latency` - When implementing interaction tracking
- `symbolication-setup` - When configuring symbolication

## Agents Used

- `codebase-analyzer` - Codebase exploration

## Reference Files

- `references/jtbd.md`
- `references/instrumentation-patterns.md`
- `references/performance.md`
- `references/crash-reporting.md`
- `references/{platform}-native.md`
- `references/platforms/*.md`
