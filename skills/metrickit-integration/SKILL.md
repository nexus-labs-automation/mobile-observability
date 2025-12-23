---
name: metrickit-integration
description: Add MetricKit diagnostics to iOS apps for supplementary crash context. Use when integrating iOS system diagnostics, tracking app exit reasons, or correlating MetricKit with existing telemetry.
triggers:
  - "metrickit"
  - "app exit metrics"
  - "hang diagnostics"
  - "cpu exception diagnostics"
  - "disk write exception"
  - "ios system diagnostics"
  - "mxmetricmanager"
priority: 3
---

# MetricKit Integration (iOS)

Add supplementary diagnostic data from iOS to your existing telemetry.

## Core Principle

MetricKit provides **supplementary diagnostic evidence** that augments your telemetry—not a crash reporting system unto itself. You still need your own telemetry (spans, breadcrumbs, events) to understand *what* happened. MetricKit diagnostics help you understand *why*.

## What You Get vs Don't Get

| You Get | You Don't Get |
|---------|---------------|
| Crash stack traces (some) | OOM stack traces (counts only) |
| Hang diagnostics with call trees | NSException name/message |
| CPU exception stack traces | User context or breadcrumbs |
| Disk write exception traces | Exact crash timestamps |
| App exit reason counts | Real-time delivery |

**Critical:** MetricKit does NOT provide OOM stack traces. You only get a count of memory exits via `MXAppExitMetric`. For OOM investigation, use memory tracking + profiling tools.

## When to Use

- Want additional crash diagnostics the OS captured
- Need system-level metrics (launch time, hang rate, exit reasons)
- Have capacity to correlate with existing telemetry
- Already have dSYM upload workflow (stacks are unsymbolicated)

## When NOT to Use

- As primary/only crash reporter (lacks context)
- Expecting OOM stack traces (you only get counts)
- Need real-time incident detection (24hr+ delay)
- Without symbolication setup

## Quick Setup

```swift
import MetricKit

class AppDelegate: UIResponder, UIApplicationDelegate, MXMetricManagerSubscriber {

    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {

        if #available(iOS 13.0, *) {
            MXMetricManager.shared.add(self)
        }
        return true
    }

    // Metrics: aggregated stats (daily delivery)
    func didReceive(_ payloads: [MXMetricPayload]) {
        for payload in payloads {
            // Exit counts, launch times, etc.
            processMetrics(payload)
        }
    }

    // Diagnostics: stack traces (iOS 14+, next-launch delivery)
    @available(iOS 14.0, *)
    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        for payload in payloads {
            // Correlate with your telemetry by timestamp
            processDiagnostics(payload)
        }
    }
}
```

## Correlation Strategy

MetricKit diagnostics arrive 24hrs+ after events. Correlate using timestamp ranges:

```swift
func processDiagnostics(_ payload: MXDiagnosticPayload) {
    let timeRange = payload.timeStampBegin...payload.timeStampEnd

    // Query your telemetry for events in this time range
    let relatedEvents = TelemetryStore.events(in: timeRange)

    // Attach diagnostic as supplementary evidence
    if let hangDiagnostic = payload.hangDiagnostics?.first {
        Observability.attachDiagnostic(
            type: "metrickit.hang",
            callStack: hangDiagnostic.callStackTree,
            relatedEvents: relatedEvents
        )
    }
}
```

## Diagnostic Types

| Type | Class | Has Stack Trace? | When Captured |
|------|-------|------------------|---------------|
| Crashes | `MXCrashDiagnostic` | Yes (some) | App terminated by OS |
| Hangs | `MXHangDiagnostic` | Yes | Main thread blocked |
| CPU Exceptions | `MXCPUExceptionDiagnostic` | Yes | CPU time exceeded |
| Disk Writes | `MXDiskWriteExceptionDiagnostic` | Yes | Excessive disk I/O |
| OOM Exits | `MXAppExitMetric` | **No** | Memory limit exceeded |

## Known Issues

| Issue | Impact | Mitigation |
|-------|--------|------------|
| iOS 17.2-17.5 bug | No crash/hang payloads | Fixed in iOS 18 |
| JSON locale bug | French locale produces invalid JSON | Use typed API properties, not raw JSON |
| Limited stack frames | Some reports have 2-3 frames | Filter noise in processing |
| Unsymbolicated | Raw addresses | Require dSYM upload |

## Implementation

See `references/metrickit.md` for:
- Full payload processing
- Vendor correlation patterns (Sentry, Datadog)
- Symbolication integration

See `references/templates/metrickit-subscriber.template.md` for copy-paste code.

## Related Skills

- See `skills/crash-instrumentation` for primary crash reporting
- See `skills/symbolication-setup` for readable stack traces
