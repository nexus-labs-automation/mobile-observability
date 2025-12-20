# Instrument Command

Generate a comprehensive instrumentation plan for a mobile codebase.

## Usage

```
/instrument [platform] [--vendor=<vendor>]
```

**Arguments:**
- `platform`: `ios`, `android`, `react-native`, or `auto` (detect from codebase)
- `--vendor`: Optional. `sentry`, `datadog`, `embrace`, `bugsnag`

**Examples:**
```
/instrument ios
/instrument android --vendor=sentry
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
2. Confirm with user if ambiguous

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
- `references/ios-native.md` | `references/android-native.md` | `references/react-native-expo.md`
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

## Skills Used

- `instrumentation-planning` - JTBD framework and prioritization

## Agents Used

- `codebase-analyzer` - Codebase exploration

## Reference Files

- `references/jtbd.md`
- `references/instrumentation-patterns.md`
- `references/performance.md`
- `references/crash-reporting.md`
- `references/{platform}-native.md`
- `references/platforms/*.md`
