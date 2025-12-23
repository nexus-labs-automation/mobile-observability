# Scout Report: MetricKit Integration for Mobile Observability Plugin

## Executive Summary

**Task:** Add MetricKit support to the mobile observability plugin for iOS crash and diagnostic instrumentation.

**Confidence Level:** High (85%)

**Recommendation:** Create a new skill (`metrickit-integration`) and reference doc (`metrickit.md`) as an **optional** integration for teams that want it. MetricKit is best understood as **supplementary diagnostic data** that augments your existing telemetry—not a crash reporting system unto itself.

---

## Key Conceptual Framework

### MetricKit Has Two Distinct Parts

| Part | Payload Class | What It Provides | Mental Model |
|------|---------------|------------------|--------------|
| **Metrics** | `MXMetricPayload` | Counts and measurements (launch times, exit counts, battery, network) | Telemetry |
| **Diagnostics** | `MXDiagnosticPayload` | Stack traces, call trees | Debug attachments |

**Critical insight:** Think of MetricKit diagnostics as **supplementary diagnostic attachments** that augment your telemetry, not as a standalone crash reporting system. You still need your own telemetry (spans, breadcrumbs, events) to understand *what* happened. MetricKit diagnostics help you understand *why*.

---

## Key Findings

### 1. What MetricKit Actually Captures

#### Diagnostics (`MXDiagnosticPayload`) - Stack Traces Available

| Diagnostic Type | Class | Data | Terminal? |
|-----------------|-------|------|-----------|
| **Crashes** | `MXCrashDiagnostic` | Exception code, signal, call stack | Yes |
| **Hangs** | `MXHangDiagnostic` | Duration, call stack | Rarely |
| **CPU Exceptions** | `MXCPUExceptionDiagnostic` | CPU time exceeded, call stack | Sometimes |
| **Disk Write Exceptions** | `MXDiskWriteExceptionDiagnostic` | Write count, call stack | No |

#### Metrics (`MXMetricPayload`) - Counts Only, No Stack Traces

| Metric | Class | Data |
|--------|-------|------|
| **Exit Reasons** | `MXAppExitMetric` | Counts of foreground/background exits by type |
| **Memory Pressure Exits** | `cumulativeMemoryPressureExitCount` | Count only |
| **Memory Limit Exits (OOM)** | `cumulativeMemoryResourceLimitExitCount` | Count only |
| **Watchdog Exits** | Various termination codes | Count only |

**Important:** MetricKit does **NOT** provide stack traces for OOMs or jetsam kills. `MXAppExitMetric` gives you *counts* of these exits, not diagnostics. OOMs are tracked as a number, not as crash reports with call stacks.

#### Watchdog Terminations

Watchdog kills show up with termination codes like:
- `0x8badf00d` ("ate bad food") - App took too long (hang timeout)
- Other codes exist but are rare

These are in `MXCrashDiagnostic` when the system kills the app, but the stack trace may be limited.

### 2. Payload Characteristics

- **Anonymous:** No user identity attached
- **Time range, not timestamp:** `timeStampBegin` and `timeStampEnd` cover a period, not exact crash time
- **Metadata:** OS version, app version, device type
- **Delivery timing:**
  - iOS 13-14: Aggregated once per day
  - iOS 15+: Immediate delivery on next launch

### 3. Prior Art - Vendor Implementations

| Vendor | MetricKit Support | Approach |
|--------|------------------|----------|
| **Sentry** | Partial (8.14.0+) | Converts hangs, CPU, disk exceptions. **Skips `MXCrashDiagnostic`** |
| **Datadog** | Indirect | References MetricKit for hang thresholds only |
| **Embrace** | Optional | Disables when using CrashlyticsReporter |
| **Bugsnag** | Not yet | On roadmap, no timeline |

**On Sentry skipping crash diagnostics:** Sentry's rationale is that their in-process handler has better context. However, this may be missing the point—MetricKit diagnostics should be treated as *supplementary evidence* correlated with your own events by timestamp/session, not as standalone events that need enrichment. By skipping `MXCrashDiagnostic`, they may be discarding useful diagnostic data for crashes their handler missed.

### 4. Limitations & Known Issues

| Limitation | Impact | Mitigation |
|------------|--------|------------|
| **Unsymbolicated stacks** | Raw addresses | Requires dSYM symbolication |
| **Limited stack frames** | Some reports have only 2-3 frames | Filter noise in processing |
| **iOS 17.2-17.5 bug** | No crash/hang payloads delivered | Fixed in iOS 18 |
| **No custom context** | Can't attach breadcrumbs to MetricKit events | Correlate via timestamp/session with your own telemetry |
| **JSON locale issues** | Swift's JSONEncoder has locale bugs ([SR-6631](https://github.com/swiftlang/swift-corelibs-foundation/issues/4487)) - French locale may produce invalid JSON with comma decimals | Use typed API properties, not raw JSON for non-stack data. Consider [ChimeHQ/Meter](https://github.com/ChimeHQ/Meter) library |
| **Missing NSException details** | Exception name/message not captured | Supplement with in-process handler |
| **TestFlight delivery** | May require TestFlight builds to receive data | Test with TestFlight |

### 5. Plugin Integration Points

| Component | Integration Opportunity |
|-----------|------------------------|
| `references/metrickit.md` | **New** - Comprehensive reference |
| `skills/metrickit-integration/` | **New** - Optional setup skill |
| `references/ios-native.md` | Add MetricKit section |
| `references/crash-reporting.md` | Add as supplementary diagnostic source |
| `agents/codebase-analyzer.md` | Detect `MXMetricManager` usage |

---

## Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| **Confusion about OOMs** | Medium | Clearly document that exit counts ≠ crash diagnostics |
| **Duplicate events** | Medium | Frame as supplementary diagnostics, not duplicate crashes |
| **Symbolication complexity** | Medium | Reference `symbolication-setup` skill |
| **JSON parsing issues** | Low | Document locale caveat, recommend Meter library |

---

## Recommendations

### Design Principle

**Optional adoption:** Do not force iOS teams to adopt MetricKit. Provide the ability to integrate for teams that want supplementary diagnostics.

### Files to Create

1. **`references/metrickit.md`** - Comprehensive reference
   - Two-part architecture (metrics vs diagnostics)
   - What you get vs what you don't (no OOM stacks)
   - Implementation pattern
   - Correlation strategy with existing telemetry
   - JSON/locale caveats

2. **`skills/metrickit-integration/SKILL.md`** - Optional skill
   - When to use / when not to use
   - `MXMetricManagerSubscriber` setup
   - Processing both payload types
   - Correlation with vendor telemetry

3. **`references/templates/metrickit-subscriber.template.md`** - Copy-paste code
   - Subscriber protocol conformance
   - Payload extraction
   - Backend upload pattern

### Files to Update

| File | Changes |
|------|---------|
| `references/ios-native.md` | Add optional MetricKit section |
| `references/crash-reporting.md` | Clarify MetricKit as diagnostic supplement |
| `skills/crash-instrumentation/SKILL.md` | Reference MetricKit for additional diagnostics |
| `agents/codebase-analyzer.md` | Add `MXMetricManager` detection |

### Proposed Skill Structure

```markdown
# MetricKit Integration (Optional)

## What MetricKit Is
Supplementary diagnostic data from the OS. Two parts:
- **Metrics:** Counts and measurements (telemetry)
- **Diagnostics:** Stack traces (debug attachments)

## When to Use
- Want additional crash diagnostics the OS captured
- Need system-level metrics (launch time, hang rate, exit reasons)
- Team has capacity to correlate with existing telemetry

## When NOT to Use
- Expecting OOM stack traces (you only get counts)
- As primary/only crash reporter (lacks context)
- Without dSYM upload workflow

## What You Get vs Don't Get

| You Get | You Don't Get |
|---------|---------------|
| Crash stack traces (some) | OOM stack traces |
| Hang diagnostics | NSException details |
| CPU/disk exception stacks | User context |
| Exit reason counts | Exact timestamps |

## Setup
[MXMetricManagerSubscriber code]

## Correlation Strategy
[How to link MetricKit diagnostics with your telemetry]
```

---

## Confidence Assessment

| Aspect | Confidence | Rationale |
|--------|------------|-----------|
| Two-part architecture | 95% | Clearly documented in Apple APIs |
| OOMs = counts only | 90% | Verified via MXAppExitMetric structure |
| Locale/JSON issues | 80% | Swift bug SR-6631 confirmed; MetricKit impact inferred |
| Vendor approaches | 85% | Sentry documented; others from issues/blogs |
| Optional adoption | 95% | User requirement |

---

## Next Steps

1. **`/plan metrickit-integration`** - Generate implementation plan
2. Create `references/metrickit.md` with corrected framing
3. Create optional `skills/metrickit-integration/SKILL.md`
4. Update existing references with accurate MetricKit info

---

## Sources

- [Apple MetricKit Documentation](https://developer.apple.com/documentation/MetricKit)
- [Apple MXAppExitMetric](https://developer.apple.com/documentation/metrickit/mxappexitmetric)
- [Apple MXDiagnosticPayload](https://developer.apple.com/documentation/metrickit/mxdiagnosticpayload)
- [Sentry MetricKit Configuration](https://docs.sentry.io/platforms/apple/configuration/metric-kit/)
- [ChimeHQ/Meter Library](https://github.com/ChimeHQ/Meter)
- [MetricKit Crash Reporting - ChimeHQ](https://www.chimehq.com/blog/metrickit-crash-reporting)
- [MetricKit Crash Reporting Part 2 - ChimeHQ](https://www.chimehq.com/blog/metrickit-crash-reporting-part-2)
- [Swift JSONEncoder Locale Bug SR-6631](https://github.com/swiftlang/swift-corelibs-foundation/issues/4487)
- [Bugsnag MetricKit Support Issue #767](https://github.com/bugsnag/bugsnag-cocoa/issues/767)
- [Monitoring App Performance with MetricKit - Swift with Majid](https://swiftwithmajid.com/2025/12/09/monitoring-app-performance-with-metrickit/)

---

## Review Notes

This report was refined based on domain expert review with the following corrections:
1. MetricKit does NOT provide OOM stack traces - only exit counts
2. Reframed from "crash reporter alternative" to "supplementary diagnostics"
3. Sentry's approach of skipping crash diagnostics may be suboptimal
4. Added JSON locale caveats
5. Made integration explicitly optional
