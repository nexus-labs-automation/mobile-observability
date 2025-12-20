# Audit Command

Scan a codebase for existing observability instrumentation and identify gaps.

## Usage

```
/audit [path]
```

**Arguments:**
- `path`: Optional. Directory to scan (defaults to current directory)

**Examples:**
```
/audit
/audit ./src
/audit ~/projects/my-app
```

---

## Workflow

### Step 1: Launch Codebase Analyzer

Invoke the `codebase-analyzer` agent to:
1. Detect platform (iOS/Android/React Native)
2. Identify architecture patterns
3. Find existing telemetry SDKs
4. Map instrumentation coverage

### Step 2: Load Reference Context

Read:
- `references/instrumentation-patterns.md` - Instrumentation checklist
- `references/platforms/*.md` - Vendor-specific patterns

### Step 3: Evaluate Coverage

Check each instrumentation area against the codebase:

| Area | What to Find | Priority |
|------|--------------|----------|
| **SDK Init** | SDK setup in AppDelegate/Application | P0 |
| **Crash Reporting** | Error capture, crash handlers | P0 |
| **User Context** | `setUser`, user ID setting | P0 |
| **Symbolication** | dSYM upload, ProGuard config | P0 |
| **App Start** | Launch timing, startup spans | P1 |
| **Screen Load** | TTI tracking, screen spans | P1 |
| **Network** | Request interceptors, API tracing | P1 |
| **Breadcrumbs** | Navigation/action logging | P1 |
| **Database** | Query tracing | P2 |
| **Custom Spans** | Business flow instrumentation | P2 |

### Step 4: Identify Anti-Patterns

Flag issues like:
- PII in breadcrumbs/logs
- Synchronous telemetry on main thread
- Missing symbolication config
- Over-instrumentation (battery impact)
- Inconsistent span naming

### Step 5: Generate Report

Output format:

```markdown
## Telemetry Audit Report

### Platform
- **Type:** [iOS/Android/React Native]
- **Language:** [Swift/Kotlin/TypeScript]

### Existing SDKs

| SDK | Version | Location | Status |
|-----|---------|----------|--------|
| Sentry | 8.0.0 | Podfile:23 | Active |
| Firebase | 10.0.0 | Podfile:25 | Crashlytics only |

### Coverage Assessment

| Area | Status | Details |
|------|--------|---------|
| Crash Reporting | ✅ Covered | Sentry SDK initialized in AppDelegate |
| Error Tracking | ⚠️ Partial | Only in network layer |
| App Start | ❌ Missing | No launch timing |
| Screen Load | ⚠️ Partial | 3/12 screens tracked |
| Network | ✅ Covered | URLSession instrumented |
| Breadcrumbs | ❌ Missing | No breadcrumb logging |
| User Context | ✅ Covered | User ID set on login |
| Symbolication | ⚠️ Partial | dSYM upload configured, but not verified |

### Coverage Score: 62%

```
Foundation:  ████████░░ 80%
Performance: ████░░░░░░ 40%
Context:     ██░░░░░░░░ 20%
Advanced:    ░░░░░░░░░░ 0%
```

### Gaps Identified

| Gap | Priority | Impact | Effort |
|-----|----------|--------|--------|
| No app start tracking | P1 | Can't measure launch perf | Low |
| No screen load TTI | P1 | Can't identify slow screens | Medium |
| No breadcrumbs | P1 | Poor crash debugging | Low |
| Missing database spans | P2 | Can't trace slow queries | Medium |

### Anti-Patterns Found

1. **[Issue]** - [Location] - [Recommendation]

### Recommendations

#### Quick Wins (This Week)
1. [Low effort, high impact items]

#### Medium Term (This Month)
1. [Moderate effort items]

#### Nice to Have
1. [Lower priority items]
```

### Step 6: Offer Next Steps

```
Would you like me to:
1. Fix the highest-priority gap
2. Generate full instrumentation plan (/instrument)
3. Deep-dive on a specific area
```

---

## Skills Used

- `instrumentation-planning` - Checklist and gap analysis

## Agents Used

- `codebase-analyzer` - Codebase exploration and telemetry detection

## Reference Files

- `references/instrumentation-patterns.md`
- `references/platforms/*.md`
