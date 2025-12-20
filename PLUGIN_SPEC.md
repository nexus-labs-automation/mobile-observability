# Mobile Observability Plugin - Specification

## Purpose

Codify mobile observability expertise into a distributable Claude Code plugin that helps teams:
- Apply instrumentation best practices
- Debug crashes and performance issues
- Audit existing telemetry coverage

## Target Users

Mobile developers who need to:
1. **Implement** - Add observability to their apps
2. **Debug** - Investigate crashes, ANRs, performance issues
3. **Audit** - Assess current instrumentation coverage

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      COMMANDS                            │
│  /instrument    /diagnose    /audit                      │
└─────────┬───────────┬───────────┬───────────────────────┘
          │           │           │
          ▼           ▼           ▼
┌─────────────────────────────────────────────────────────┐
│                       AGENTS                             │
│  codebase-analyzer          issue-analyzer               │
└─────────┬───────────────────────┬───────────────────────┘
          │                       │
          ▼                       ▼
┌─────────────────────────────────────────────────────────┐
│                       SKILLS                             │
│  instrumentation-planning   crash-debugging              │
│  performance-optimization   session-replay (optional)    │
└─────────┬───────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────┐
│                    REFERENCES                            │
│  *.md files (15+)   platforms/   templates/              │
└─────────────────────────────────────────────────────────┘
```

---

## Commands

### `/instrument [platform]`

**Purpose:** Generate instrumentation plan for a codebase

**Flow:**
1. Detect platform (iOS/Android/RN) or use specified
2. Invoke `codebase-analyzer` to understand architecture
3. Apply `instrumentation-planning` skill (JTBD framework)
4. Generate prioritized implementation plan

**Output:** Markdown plan with code snippets, file locations, priority order

### `/diagnose [type]`

**Purpose:** Analyze crash logs or performance issues

**Types:** `crash`, `performance`, `anr`, `oom`, `auto`

**Flow:**
1. Detect issue type from input or prompt user
2. Invoke `issue-analyzer` agent
3. Apply appropriate skill (crash-debugging or performance-optimization)
4. Generate root cause analysis + fix recommendations

**Output:** Analysis with root cause, affected code, recommended fix

### `/audit`

**Purpose:** Scan codebase for existing telemetry and gaps

**Flow:**
1. Invoke `codebase-analyzer` to find existing SDKs/telemetry
2. Compare against `instrumentation-planning` checklist
3. Identify gaps and anti-patterns

**Output:** Coverage report with gaps and recommendations

---

## Agents

### `codebase-analyzer`

**Purpose:** Explore and understand mobile codebase structure

**Outputs:**
- Platform (iOS/Android/RN)
- Architecture pattern (MVC, MVVM, TCA, MVI, REDUX, etc.)
- Entry points (AppDelegate, Application, etc.)
- Navigation pattern
- Network layer
- Data persistence
- Existing telemetry SDKs

**Used by:** `/instrument`, `/audit`

### `issue-analyzer`

**Purpose:** Analyze crash logs and performance issues

**Capabilities:**
- Parse iOS crash logs (EXC_BAD_ACCESS, SIGABRT, etc.)
- Parse Android crash logs (ANR, exceptions)
- Identify crash type and severity
- Detect symbolication issues
- Identify performance bottleneck type
- Identify memory leaks
- Identify race conditions, data race, and deadlocks

**Used by:** `/diagnose`

---

## Skills

### `instrumentation-planning`

**Trigger:** "What should I measure?"

**Scope:**
- JTBD framework for prioritization
- Instrumentation checklist (tiers 1-5)
- Anti-patterns to avoid
- Platform-agnostic patterns

**References:**
- `jtbd.md`
- `instrumentation-patterns.md`
- `observability-fundamentals.md`

### `crash-debugging`

**Trigger:** "Why is my app crashing?"

**Scope:**
- Crash type identification
- Symbolication guidance
- Breadcrumb analysis
- Common crash patterns by platform
- Fix recommendations

**References:**
- `crash-reporting.md`
- `ios-native.md` / `android-native.md` / `react-native-expo.md`

### `performance-optimization`

**Trigger:** "Why is my app slow?"

**Scope:**
- App start optimization
- Screen load / TTI
- Network performance
- UI rendering / jank
- Database query performance

**References:**
- `performance.md`
- `ui-performance.md`
- `data-persistence.md`

### `session-replay` (Optional)

**Trigger:** "Set up session replay"

**Scope:**
- Capture strategies
- Privacy masking
- Integration patterns
- Performance impact

**References:**
- `session-replay.md`

---

## Reference Files (unchanged)

```
references/
├── observability-fundamentals.md   # Logs/metrics/traces decision framework
├── jtbd.md                         # Jobs-to-be-Done framework
├── instrumentation-patterns.md     # Checklists and anti-patterns
├── crash-reporting.md              # Crash/error handling
├── performance.md                  # App start, network
├── ui-performance.md               # Navigation, scroll, render
├── data-persistence.md             # Database tracing
├── session-replay.md               # Visual debugging
├── mobile-challenges.md            # Offline, battery, fragmentation
├── alert-thresholds.md             # SLOs
├── ios-native.md                   # iOS patterns
├── android-native.md               # Android patterns
├── react-native-expo.md            # RN patterns
├── native-mobile.md                # Low-level APIs
├── user-journeys.md                # Funnels (referenced by skills)
├── platforms/                      # Vendor guides
│   ├── sentry.md
│   ├── datadog.md
│   ├── embrace.md
│   └── bugsnag.md
└── templates/                      # Code templates
    ├── screen-load-tracking.template.md
    ├── error-boundary.template.md
    ├── breadcrumb-manager.template.md
    └── navigation-tracking.template.md
```

---

## Final Plugin Structure

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
│   │   └── SKILL.md
│   ├── crash-debugging/
│   │   └── SKILL.md
│   ├── performance-optimization/
│   │   └── SKILL.md
│   └── session-replay/           # Optional
│       └── SKILL.md
├── references/
│   └── [existing content]
├── PLUGIN_SPEC.md                 # This file
├── CLAUDE.md
└── README.md
```

---

## Implementation Order

1. **Phase 1: Foundation**
   - Update plugin.json
   - Create skill SKILL.md files (4)
   - Clean up existing command/agent files

2. **Phase 2: Agents**
   - Implement codebase-analyzer
   - Implement issue-analyzer

3. **Phase 3: Commands**
   - Implement /instrument
   - Implement /diagnose
   - Implement /audit

4. **Phase 4: Testing**
   - Test on Wikipedia iOS
   - Test on sample Android/RN projects
   - Iterate on skill guidance

---

## Success Criteria

- [ ] `/instrument ios` produces actionable plan for Wikipedia iOS
- [ ] `/diagnose crash` correctly identifies crash type from log
- [ ] `/audit` finds existing telemetry and identifies gaps
- [ ] Skills provide production-ready code snippets
- [ ] Plugin installs and works in fresh Claude Code session
