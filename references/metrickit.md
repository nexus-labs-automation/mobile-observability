# MetricKit Integration

Supplementary diagnostic data from iOS for crash context, hang detection, and system-level metrics.

## Table of Contents

1. [Overview](#overview)
2. [Two-Part Architecture](#two-part-architecture)
3. [What You Get vs Don't Get](#what-you-get-vs-dont-get)
4. [Subscriber Setup](#subscriber-setup)
5. [Processing Metrics](#processing-metrics)
6. [Processing Diagnostics](#processing-diagnostics)
7. [Correlation Strategies](#correlation-strategies)
8. [Vendor Integration](#vendor-integration)
9. [Known Issues](#known-issues)
10. [When to Use](#when-to-use)

---

## Overview

MetricKit is an iOS framework (iOS 13+) that provides aggregated metrics and diagnostic data collected by the operating system. Think of it as **supplementary diagnostic attachments** that augment your telemetry—not a standalone crash reporting system.

**Key Insight:** You still need your own telemetry (spans, breadcrumbs, events) to understand *what* happened. MetricKit diagnostics help you understand *why*.

**Delivery Timing:**
- iOS 13-14: Aggregated once per day
- iOS 15+: Immediate delivery on next app launch

---

## Two-Part Architecture

MetricKit has two distinct subsystems:

| Part | Payload Class | What It Provides | Delivery |
|------|---------------|------------------|----------|
| **Metrics** | `MXMetricPayload` | Counts and measurements (launch times, exit counts, battery, network) | Daily aggregates |
| **Diagnostics** | `MXDiagnosticPayload` | Stack traces and call trees | Next launch (iOS 15+) |

### Metrics = Telemetry

Aggregated statistics about app behavior:
- App launch times (cold/warm/resume)
- Exit reasons and counts
- Memory usage histograms
- Battery consumption
- Network transfer sizes

### Diagnostics = Debug Attachments

Individual diagnostic reports with stack traces:
- Crash reports (some)
- Hang call trees
- CPU exception traces
- Disk write exception traces

---

## What You Get vs Don't Get

### Diagnostics with Stack Traces

| Diagnostic Type | Class | Stack Trace? | When Captured |
|-----------------|-------|--------------|---------------|
| Crashes | `MXCrashDiagnostic` | Yes | App terminated by signal/exception |
| Hangs | `MXHangDiagnostic` | Yes | Main thread blocked (customizable threshold) |
| CPU Exceptions | `MXCPUExceptionDiagnostic` | Yes | CPU time limit exceeded |
| Disk Write Exceptions | `MXDiskWriteExceptionDiagnostic` | Yes | Excessive disk I/O |

### Metrics Without Stack Traces

| Metric | Class | Data | Stack Trace? |
|--------|-------|------|--------------|
| Exit Reasons | `MXAppExitMetric` | Counts by exit type | **No** |
| Memory Pressure Exits | `cumulativeMemoryPressureExitCount` | Count only | **No** |
| Memory Limit Exits (OOM) | `cumulativeMemoryResourceLimitExitCount` | Count only | **No** |

### The OOM Limitation

**MetricKit does NOT provide OOM stack traces.** `MXAppExitMetric` gives you *counts* of memory exits, not crash reports with call stacks.

For OOM investigation, you need:
1. Memory tracking instrumentation
2. Memory pressure breadcrumbs
3. Profiling tools (Instruments, Xcode Memory Graph)
4. Custom memory snapshots before termination

---

## Subscriber Setup

### Basic Implementation

```swift
import MetricKit

@main
struct MyApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate

    var body: some Scene {
        WindowGroup { ContentView() }
    }
}

class AppDelegate: NSObject, UIApplicationDelegate, MXMetricManagerSubscriber {

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil
    ) -> Bool {

        // Register for MetricKit callbacks
        if #available(iOS 13.0, *) {
            MXMetricManager.shared.add(self)
        }

        return true
    }

    // MARK: - MXMetricManagerSubscriber

    func didReceive(_ payloads: [MXMetricPayload]) {
        // Daily aggregated metrics
        for payload in payloads {
            processMetrics(payload)
        }
    }

    @available(iOS 14.0, *)
    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        // Diagnostic reports with stack traces
        for payload in payloads {
            processDiagnostics(payload)
        }
    }
}
```

### Cleanup on App Termination

```swift
func applicationWillTerminate(_ application: UIApplication) {
    if #available(iOS 13.0, *) {
        MXMetricManager.shared.remove(self)
    }
}
```

---

## Processing Metrics

### App Exit Metrics

```swift
func processMetrics(_ payload: MXMetricPayload) {
    // Time range this payload covers
    let timeRange = "\(payload.timeStampBegin) - \(payload.timeStampEnd)"

    // App exit reasons (iOS 14+)
    if #available(iOS 14.0, *),
       let exitMetrics = payload.applicationExitMetrics {

        // Foreground exits
        let foreground = exitMetrics.foregroundExitData
        logExitCounts("foreground", [
            "normal": foreground.cumulativeNormalAppExitCount,
            "abnormal": foreground.cumulativeAbnormalExitCount,
            "watchdog": foreground.cumulativeAppWatchdogExitCount,
            "memory_resource_limit": foreground.cumulativeMemoryResourceLimitExitCount,
            "memory_pressure": foreground.cumulativeMemoryPressureExitCount,
            "suspended_with_locked_file": foreground.cumulativeSuspendedWithLockedFileExitCount
        ])

        // Background exits
        let background = exitMetrics.backgroundExitData
        logExitCounts("background", [
            "normal": background.cumulativeNormalAppExitCount,
            "abnormal": background.cumulativeAbnormalExitCount,
            "watchdog": background.cumulativeAppWatchdogExitCount,
            "memory_resource_limit": background.cumulativeMemoryResourceLimitExitCount,
            "memory_pressure": background.cumulativeMemoryPressureExitCount,
            "cpu_resource_limit": background.cumulativeCPUResourceLimitExitCount,
            "background_task_assertion_timeout": background.cumulativeBackgroundTaskAssertionTimeoutExitCount
        ])
    }

    // Launch metrics
    if let launchMetrics = payload.applicationLaunchMetrics {
        // Cold launch histogram
        if let coldLaunch = launchMetrics.histogrammedTimeToFirstDraw {
            Analytics.track("metrickit.launch.cold", attributes: [
                "p50_ms": coldLaunch.bucketEnumerator.allObjects.description
            ])
        }
    }

    // Memory metrics
    if let memoryMetrics = payload.memoryMetrics {
        Analytics.track("metrickit.memory", attributes: [
            "peak_memory_mb": memoryMetrics.peakMemoryUsage.averageMeasurement.doubleValue(for: .megabytes)
        ])
    }
}

private func logExitCounts(_ context: String, _ counts: [String: Int]) {
    for (reason, count) in counts where count > 0 {
        Analytics.track("metrickit.exit", attributes: [
            "context": context,
            "reason": reason,
            "count": count
        ])
    }
}
```

---

## Processing Diagnostics

### Diagnostic Payload Processing

```swift
@available(iOS 14.0, *)
func processDiagnostics(_ payload: MXDiagnosticPayload) {
    let timeRange = payload.timeStampBegin...payload.timeStampEnd

    // Process crash diagnostics
    if let crashes = payload.crashDiagnostics {
        for crash in crashes {
            processCrashDiagnostic(crash, timeRange: timeRange)
        }
    }

    // Process hang diagnostics
    if let hangs = payload.hangDiagnostics {
        for hang in hangs {
            processHangDiagnostic(hang, timeRange: timeRange)
        }
    }

    // Process CPU exception diagnostics
    if let cpuExceptions = payload.cpuExceptionDiagnostics {
        for exception in cpuExceptions {
            processCPUExceptionDiagnostic(exception, timeRange: timeRange)
        }
    }

    // Process disk write exception diagnostics
    if let diskExceptions = payload.diskWriteExceptionDiagnostics {
        for exception in diskExceptions {
            processDiskWriteExceptionDiagnostic(exception, timeRange: timeRange)
        }
    }
}
```

### Crash Diagnostic Processing

```swift
@available(iOS 14.0, *)
func processCrashDiagnostic(_ crash: MXCrashDiagnostic, timeRange: ClosedRange<Date>) {
    // Extract crash details
    let exceptionType = crash.exceptionType?.intValue
    let exceptionCode = crash.exceptionCode?.intValue
    let signal = crash.signal?.intValue
    let terminationReason = crash.terminationReason

    // Get call stack (requires symbolication)
    let callStackTree = crash.callStackTree
    let callStackJSON = callStackTree.jsonRepresentation()

    // Build diagnostic event
    let diagnostic = DiagnosticEvent(
        type: "crash",
        timeRange: timeRange,
        attributes: [
            "exception_type": exceptionType as Any,
            "exception_code": exceptionCode as Any,
            "signal": signal as Any,
            "termination_reason": terminationReason as Any
        ],
        callStack: callStackJSON
    )

    // Correlate and upload
    correlateAndUpload(diagnostic)
}
```

### Hang Diagnostic Processing

```swift
@available(iOS 14.0, *)
func processHangDiagnostic(_ hang: MXHangDiagnostic, timeRange: ClosedRange<Date>) {
    // Hang duration
    let hangDuration = hang.hangDuration

    // Get call stack
    let callStackTree = hang.callStackTree
    let callStackJSON = callStackTree.jsonRepresentation()

    let diagnostic = DiagnosticEvent(
        type: "hang",
        timeRange: timeRange,
        attributes: [
            "duration_ms": hangDuration.converted(to: .milliseconds).value
        ],
        callStack: callStackJSON
    )

    correlateAndUpload(diagnostic)
}
```

---

## Correlation Strategies

MetricKit diagnostics arrive 24hrs+ after events and lack user context. Correlate using timestamps.

### Strategy 1: Timestamp-Based Correlation

```swift
struct DiagnosticEvent {
    let type: String
    let timeRange: ClosedRange<Date>
    let attributes: [String: Any]
    let callStack: Data
}

func correlateAndUpload(_ diagnostic: DiagnosticEvent) {
    // Query your telemetry for events in this time range
    let relatedEvents = TelemetryStore.shared.events(
        in: diagnostic.timeRange,
        types: ["error", "crash", "screen_view", "user_action"]
    )

    // Find the most likely related event
    let correlatedEvent = relatedEvents.first { event in
        // Match by type or proximity
        event.type == "crash" || event.type == "error"
    }

    // Build combined payload
    var payload: [String: Any] = [
        "diagnostic_type": diagnostic.type,
        "time_range_start": ISO8601DateFormatter().string(from: diagnostic.timeRange.lowerBound),
        "time_range_end": ISO8601DateFormatter().string(from: diagnostic.timeRange.upperBound),
        "attributes": diagnostic.attributes,
        "call_stack_base64": diagnostic.callStack.base64EncodedString()
    ]

    if let correlated = correlatedEvent {
        payload["correlated_event_id"] = correlated.id
        payload["correlated_screen"] = correlated.attributes["screen"]
        payload["correlated_user_id"] = correlated.attributes["user_id"]
    }

    // Upload to backend
    MetricKitUploader.shared.upload(payload)
}
```

### Strategy 2: Session-Based Correlation

```swift
// Store session ID in UserDefaults for MetricKit correlation
class SessionManager {
    static let shared = SessionManager()

    private let sessionKey = "metrickit_session_id"

    var currentSessionId: String {
        if let existing = UserDefaults.standard.string(forKey: sessionKey) {
            return existing
        }
        let newId = UUID().uuidString
        UserDefaults.standard.set(newId, forKey: sessionKey)
        return newId
    }

    func rotateSession() {
        UserDefaults.standard.set(UUID().uuidString, forKey: sessionKey)
    }
}

// In diagnostic processing
func correlateBySession(_ diagnostic: DiagnosticEvent) {
    // MetricKit payloads don't include session ID, but we can
    // query our backend for sessions active during the time range
    Backend.querySessions(in: diagnostic.timeRange) { sessions in
        for session in sessions {
            // Attach diagnostic to each potentially affected session
            Backend.attachDiagnostic(sessionId: session.id, diagnostic: diagnostic)
        }
    }
}
```

---

## Vendor Integration

### Generic Backend

```swift
class MetricKitUploader {
    static let shared = MetricKitUploader()

    private let endpoint = URL(string: "https://api.yourbackend.com/metrickit")!

    func upload(_ payload: [String: Any]) {
        var request = URLRequest(url: endpoint)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.httpBody = try? JSONSerialization.data(withJSONObject: payload)

        URLSession.shared.dataTask(with: request) { _, response, error in
            if let error = error {
                // Store locally for retry
                LocalStorage.storePendingUpload(payload)
            }
        }.resume()
    }
}
```

### Sentry Integration

```swift
import Sentry

@available(iOS 14.0, *)
func processDiagnosticsForSentry(_ payload: MXDiagnosticPayload) {
    // Process hangs
    if let hangs = payload.hangDiagnostics {
        for hang in hangs {
            let event = Event(level: .warning)
            event.message = SentryMessage(formatted: "MetricKit hang detected")
            event.tags = [
                "metrickit.type": "hang",
                "metrickit.duration_ms": String(hang.hangDuration.converted(to: .milliseconds).value)
            ]

            // Attach call stack as JSON
            let attachment = Attachment(
                data: hang.callStackTree.jsonRepresentation(),
                filename: "metrickit_hang_callstack.json",
                contentType: "application/json"
            )

            SentrySDK.capture(event: event) { scope in
                scope.addAttachment(attachment)
            }
        }
    }

    // Process CPU exceptions
    if let cpuExceptions = payload.cpuExceptionDiagnostics {
        for exception in cpuExceptions {
            let event = Event(level: .warning)
            event.message = SentryMessage(formatted: "MetricKit CPU exception")
            event.tags = [
                "metrickit.type": "cpu_exception",
                "metrickit.cpu_time": String(exception.totalCPUTime.converted(to: .seconds).value)
            ]

            SentrySDK.capture(event: event)
        }
    }
}
```

### Datadog Integration

```swift
import DatadogLogs

@available(iOS 14.0, *)
func processDiagnosticsForDatadog(_ payload: MXDiagnosticPayload) {
    let logger = Logger.create(
        with: Logger.Configuration(name: "metrickit")
    )

    // Process hangs
    if let hangs = payload.hangDiagnostics {
        for hang in hangs {
            logger.warn("MetricKit hang detected", attributes: [
                "metrickit.type": "hang",
                "metrickit.duration_ms": hang.hangDuration.converted(to: .milliseconds).value,
                "metrickit.time_range_start": payload.timeStampBegin.timeIntervalSince1970,
                "metrickit.time_range_end": payload.timeStampEnd.timeIntervalSince1970
            ])
        }
    }

    // Process crashes (as RUM errors)
    if let crashes = payload.crashDiagnostics {
        for crash in crashes {
            RUMMonitor.shared().addError(
                message: "MetricKit crash diagnostic",
                type: "MetricKit",
                source: .custom,
                attributes: [
                    "metrickit.exception_type": crash.exceptionType?.intValue as Any,
                    "metrickit.signal": crash.signal?.intValue as Any,
                    "metrickit.termination_reason": crash.terminationReason as Any
                ]
            )
        }
    }
}
```

---

## Known Issues

### iOS 17.2-17.5 Delivery Bug

No crash or hang payloads delivered in these iOS versions. Fixed in iOS 18.

**Mitigation:** Check iOS version and log warning if affected:

```swift
let osVersion = ProcessInfo.processInfo.operatingSystemVersion
if osVersion.majorVersion == 17 && osVersion.minorVersion >= 2 && osVersion.minorVersion <= 5 {
    Logger.warn("MetricKit crash/hang delivery is broken on iOS 17.2-17.5. Relying on primary crash reporter.")
}
```

### JSON Locale Bug

Swift's JSONEncoder has a locale bug where French locale produces invalid JSON with comma decimals.

**Mitigation:** Use typed API properties, not raw JSON for non-stack data:

```swift
// Bad: May produce invalid JSON in French locale
let json = try JSONEncoder().encode(payload)

// Good: Use typed properties
let hangDuration = hang.hangDuration.value
let callStack = hang.callStackTree.jsonRepresentation()
```

### Unsymbolicated Stack Traces

MetricKit stacks contain raw memory addresses. You must symbolicate using dSYMs.

**Mitigation:** Ensure dSYM upload is configured. See `skills/symbolication-setup`.

### Limited Stack Frames

Some diagnostic reports have only 2-3 stack frames, providing limited context.

**Mitigation:** Filter out diagnostics with minimal stack data:

```swift
// MXCallStackTree uses JSON representation - check size as heuristic
let callStackJSON = callStack.jsonRepresentation()
let minimumStackSize = 500 // bytes, roughly 5+ frames
if callStackJSON.count < minimumStackSize {
    Logger.debug("Skipping MetricKit diagnostic with insufficient stack data")
    return
}
```

---

## When to Use

### Good Fit

- **Supplementary crash context:** Capture OS-level diagnostics your crash reporter might miss
- **Hang detection:** System-level hang detection with configurable threshold
- **Exit reason trends:** Track exit patterns over time (OOM counts, watchdog kills)
- **System metrics:** Battery, network, memory usage patterns

### Not a Good Fit

- **Primary crash reporter:** Lacks user context, breadcrumbs, custom data
- **OOM investigation:** Only provides counts, no stack traces
- **Real-time alerting:** 24hr+ delivery delay
- **Without symbolication:** Raw addresses are not useful

---

## Integration Points

- **[crash-reporting.md](crash-reporting.md)** - Primary crash reporting strategies
- **[ios-native.md](ios-native.md)** - iOS-specific instrumentation patterns
- **[templates/metrickit-subscriber.template.md](templates/metrickit-subscriber.template.md)** - Copy-paste implementation
- **[platforms/sentry.md](platforms/sentry.md)** - Sentry-specific configuration
- **[platforms/datadog.md](platforms/datadog.md)** - Datadog-specific configuration

---

## Sources

- [Apple MetricKit Documentation](https://developer.apple.com/documentation/MetricKit)
- [Apple MXAppExitMetric](https://developer.apple.com/documentation/metrickit/mxappexitmetric)
- [Apple MXDiagnosticPayload](https://developer.apple.com/documentation/metrickit/mxdiagnosticpayload)
- [Sentry MetricKit Configuration](https://docs.sentry.io/platforms/apple/configuration/metric-kit/)
