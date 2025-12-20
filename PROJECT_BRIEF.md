# Mobile Observability Skill - Project Brief

## Executive Summary

Build an **agentic router** that enables Claude to surgically load mobile observability reference content based on query intent, platform, and token budget. Transform ~61K tokens of reference material into a skill that typically loads 1-5K tokens per query.

**Goal**: 78% token reduction (56K baseline → 10-15K per task)

## Starting Point

The `references/` folder contains 18 reference files + 4 templates covering:
- iOS, Android, React Native observability
- Crash reporting, performance, session replay
- Database instrumentation, user journeys
- Vendor integrations (Sentry, Datadog, Embrace, BugSnag)

**Current state**: Raw content with no optimization metadata.

---

## Requirements

### Phase 1: Metadata Layer

Add YAML front matter to all reference files.

**Schema** (`reference-metadata/v1.0`):
```yaml
---
schema: reference-metadata/v1.0
title: "Human-readable title"
description: "Brief description"
token_estimate: 4500
topics:
  - topic-one
  - topic-two
platforms:
  - ios
  - android
  - react-native
  - cross-platform
vendors:
  - sentry
  - datadog
  - generic
complexity: intermediate  # beginner | intermediate | advanced
last_updated: 2025-12-20
---
```

**Token formula**: 4 tokens per line of content

**Deliverables**:
- YAML front matter on all 18 reference files
- `scripts/estimate-tokens.py` - Calculate token counts
- `docs/schemas/reference-metadata.schema.yaml` - Schema definition

### Phase 2: Semantic Anchoring

Add section markers to files >800 lines for surgical extraction.

**Anchor format**:
```markdown
<!-- SECTION:section-id -->
## Section Heading
```

**Rules**:
- Place anchor on line immediately before h2 heading
- HTML comments are invisible in rendered markdown
- IDs: kebab-case, URL-safe
- Only h2 headings get anchors (not h3/h4)

**Target files** (8 largest):
| File | Target Sections |
|------|-----------------|
| ui-performance.md | 8 |
| data-persistence.md | 11 |
| observability-fundamentals.md | 9 |
| crash-reporting.md | 8 |
| performance.md | 6 |
| native-mobile.md | 6 |
| mobile-challenges.md | 6 |
| session-replay.md | 6 |

**Deliverables**:
- 60+ section anchors added
- `scripts/extract-section.sh <file> <section-id>` - Extract by anchor
- `docs/conventions/SEMANTIC_ANCHORS.md` - Convention documentation

### Phase 3: Agentic Router

Transform SKILL.md into active routing logic.

**Intent Classification** (6 types):

| Intent | Signals | Route To |
|--------|---------|----------|
| `crash-debug` | crash, error, ANR, hang | crash-reporting.md sections |
| `performance` | slow, jank, latency, start | performance.md + ui-performance.md |
| `implementation` | implement, add, set up | Topic-specific sections |
| `debugging` | debug, fix, investigate | Breadcrumbs + error context |
| `learning` | understand, basics, explain | Fundamentals + JTBD |
| `vendor-setup` | Sentry, Datadog, Embrace, BugSnag | platforms/*.md |

**Platform Detection**:

| Signals | Platform |
|---------|----------|
| Swift, SwiftUI, iOS, UIKit, MetricKit, Xcode, CocoaPods, SPM | `ios` |
| Kotlin, Compose, Android, Room, Perfetto, Gradle, Jetpack | `android` |
| React Native, Expo, Hermes, bridge, Metro | `react-native` |

**Token Budget Tiers**:

| Tier | Max Tokens | When |
|------|------------|------|
| `minimal` | 1,000 | "what is X", quick lookup |
| `focused` | 2,500 | Single implementation task |
| `comprehensive` | 5,000 | Debug complex issue |
| `full-context` | 10,000 | Architecture decisions |

**Deliverables**:
- Rewritten SKILL.md with routing logic
- `docs/conventions/ROUTING.md` - Routing conventions

### Phase 4: Master Index

Create machine-readable routing index.

**INDEX.yaml structure**:
```yaml
schema: reference-index/v1.0

references:
  - file: crash-reporting.md
    title: "Crash Reporting"
    token_estimate: 4485
    topics: [crash-handling, breadcrumbs, anr-detection]
    platforms: [ios, android, react-native]
    complexity: intermediate

section_index:
  app-start:
    file: performance.md
    tokens: 1218
    platforms: [ios, android]
  swiftui-observability:
    file: ui-performance.md
    tokens: 1158
    platforms: [ios]

topic_index:
  crash-handling: [crash-reporting.md]
  performance-budgets: [performance.md]

task_routing:
  "implement crash reporting":
    sections: [crash-handling, breadcrumbs]
    tokens: 2417
    platforms: [all]
  "debug ANR/hang":
    sections: [anr-detection, app-start]
    tokens: 1800
    platforms: [android]
  # ... 14 common tasks total

by_platform:
  ios: [ios-native.md, ...]
  android: [android-native.md, ...]

routing_rules:
  intent_patterns: { ... }
  budget_selection: { ... }
```

**Deliverables**:
- `references/INDEX.yaml` - Complete routing index
- `scripts/update-token-counts.py` - Refresh estimates

### Phase 5: Validation Integration

Connect routing with verification.

**Deliverables**:
- `scripts/validate-anchors.sh` - Verify anchor coverage
- Unit tests for routing logic
- Schema validation tests

### Phase 6: Documentation & Testing

End-to-end validation.

**Test categories**:
- Unit: Template code validity
- Validation: Schema/structure checks
- Smoke: End-to-end routing verification

**Deliverables**:
- `tests/config.yaml` - Test configuration
- Comprehensive test coverage (80%+)
- Updated README

---

## Quick Routes (14 Common Tasks)

Pre-calculated routing for common queries:

```yaml
# Crash & Errors
"implement crash reporting": crash-handling + breadcrumbs (~2.4K)
"debug ANR/hang": anr-detection + app-start (~1.8K)

# Performance
"app starts slowly": app-start + performance-budgets (~1.7K)
"fix scroll jank": scroll-performance + animation-performance (~1.6K)

# UI Framework
"SwiftUI rendering issues": swiftui-observability (~1.2K)
"Compose recomposition": compose-observability (~0.9K)

# Database
"trace Room queries": android-room (~0.9K)
"Core Data observability": core-data (~0.8K)

# Session Replay
"implement session replay": session-replay-concepts + capture-strategies + privacy-masking (~2.2K)

# User Journeys
"implement deep link attribution": mobile-entry-funnels + journey-fundamentals (~0.9K)
"track onboarding funnel": onboarding-funnels + journey-fundamentals (~0.8K)
"push notification re-engagement": mobile-entry-funnels (~0.6K)
"track error recovery flows": error-recovery + session-continuity (~0.8K)

# Basics
"understand observability": three-pillars + correlation-strategy (~0.7K)
"what should I measure?": jtbd.md + instrumentation-patterns.md (~3.7K)
```

---

## Success Criteria

- [ ] All 18 reference files have valid YAML front matter
- [ ] 8 largest files have semantic anchors (60+ total)
- [ ] INDEX.yaml routes 14 common tasks correctly
- [ ] Token reduction verified: 56K → <15K per typical task
- [ ] Section extraction works for all anchored files
- [ ] Platform filtering reduces irrelevant content
- [ ] Vendor queries route to correct platform guides

---

## Constraints

1. **Preserve existing content**: Reference files contain domain knowledge - only add metadata, don't rewrite content
2. **Token formula**: Always use 4 tokens per line
3. **Anchor placement**: Only h2 headings, not h3/h4
4. **Schema compatibility**: All metadata must match defined schemas
5. **Cross-platform**: Solutions must work for iOS, Android, and React Native

---

## File Reference

```
references/
├── INDEX.yaml                      # Master index (to create)
├── observability-fundamentals.md   # Core concepts (~4.7K)
├── jtbd.md                         # What to measure (~2.2K)
├── instrumentation-patterns.md     # Checklist patterns (~1.5K)
├── user-journeys.md                # Funnels, attribution (~2.1K)
├── crash-reporting.md              # Errors, ANR, breadcrumbs (~4.5K)
├── performance.md                  # App start, screen load (~4.7K)
├── ui-performance.md               # Navigation, scroll, UI (~8.9K)
├── session-replay.md               # Visual debugging (~3.5K)
├── data-persistence.md             # SQLite, Core Data, Room (~7K)
├── mobile-challenges.md            # Offline, battery (~4.4K)
├── native-mobile.md                # MetricKit, Perfetto (~4.7K)
├── ios-native.md                   # Swift/SwiftUI patterns (~1.8K)
├── android-native.md               # Kotlin/Compose patterns (~2.2K)
├── react-native-expo.md            # Expo, Hermes (~1.6K)
├── alert-thresholds.md             # SLOs, alerting (~1.3K)
├── platforms/
│   ├── sentry.md                   # Sentry integration (~1K)
│   ├── datadog.md                  # Datadog RUM (~1.1K)
│   ├── embrace.md                  # Embrace (~1K)
│   └── bugsnag.md                  # BugSnag (~0.5K)
└── templates/
    ├── screen-load-tracking.template.md
    ├── error-boundary.template.md
    ├── breadcrumb-manager.template.md
    └── navigation-tracking.template.md
```

---

## Clarifying Questions (for ambiguous queries)

| Ambiguity | Question |
|-----------|----------|
| No platform | "Which platform: iOS, Android, or React Native?" |
| Generic "performance" | "What symptom: slow startup, UI jank, or network latency?" |
| No vendor | "Which vendor are you using, or want recommendations?" |
| Scope unclear | "Implementing new or debugging existing?" |
