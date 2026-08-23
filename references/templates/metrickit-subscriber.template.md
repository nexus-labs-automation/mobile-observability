# MetricKit Subscriber Template

Copy-paste implementation for MetricKit integration with correlation patterns.

## Basic Subscriber

```swift
// MetricKitSubscriber.swift
import MetricKit

final class MetricKitSubscriber: NSObject, MXMetricManagerSubscriber {
    static let shared = MetricKitSubscriber()

    private override init() {
        super.init()
    }

    // MARK: - Lifecycle

    func start() {
        if #available(iOS 13.0, *) {
            MXMetricManager.shared.add(self)
        }
    }

    func stop() {
        if #available(iOS 13.0, *) {
            MXMetricManager.shared.remove(self)
        }
    }

    // MARK: - MXMetricManagerSubscriber

    /// Receives daily aggregated metrics
    func didReceive(_ payloads: [MXMetricPayload]) {
        for payload in payloads {
            processMetrics(payload)
        }
    }

    /// Receives diagnostic reports with stack traces (iOS 14+)
    @available(iOS 14.0, *)
    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        for payload in payloads {
            processDiagnostics(payload)
        }
    }

    // MARK: - Metrics Processing

    private func processMetrics(_ payload: MXMetricPayload) {
        // App exit metrics (iOS 14+)
        if #available(iOS 14.0, *),
           let exitMetrics = payload.applicationExitMetrics {
            processExitMetrics(exitMetrics)
        }

        // Launch metrics
        if let launchMetrics = payload.applicationLaunchMetrics {
            processLaunchMetrics(launchMetrics)
        }

        // Memory metrics
        if let memoryMetrics = payload.memoryMetrics {
            processMemoryMetrics(memoryMetrics)
        }
    }

    @available(iOS 14.0, *)
    private func processExitMetrics(_ metrics: MXAppExitMetric) {
        let foreground = metrics.foregroundExitData
        let background = metrics.backgroundExitData

        // Log OOM exits (counts only, no stack traces)
        let oomCount = foreground.cumulativeMemoryResourceLimitExitCount +
                       background.cumulativeMemoryResourceLimitExitCount

        if oomCount > 0 {
            // TODO: Replace with your analytics/telemetry SDK
            // Examples: Observability.track(), SentrySDK.capture(), logger.info()
            print("MetricKit OOM exits - foreground: \(foreground.cumulativeMemoryResourceLimitExitCount), background: \(background.cumulativeMemoryResourceLimitExitCount)")
        }

        // Log watchdog exits
        let watchdogCount = foreground.cumulativeAppWatchdogExitCount +
                           background.cumulativeAppWatchdogExitCount

        if watchdogCount > 0 {
            print("MetricKit watchdog exits - foreground: \(foreground.cumulativeAppWatchdogExitCount), background: \(background.cumulativeAppWatchdogExitCount)")
        }
    }

    private func processLaunchMetrics(_ metrics: MXAppLaunchMetric) {
        // Cold launch time histogram
        if let coldLaunch = metrics.histogrammedTimeToFirstDraw {
            // TODO: Extract and upload histogram data to your analytics backend
            print("MetricKit cold launch histogram available")
        }
    }

    private func processMemoryMetrics(_ metrics: MXMemoryMetric) {
        let peakMB = metrics.peakMemoryUsage.averageMeasurement.doubleValue(for: .megabytes)
        // TODO: Replace with your analytics/telemetry SDK
        print("MetricKit peak memory: \(peakMB) MB")
    }

    // MARK: - Diagnostics Processing

    @available(iOS 14.0, *)
    private func processDiagnostics(_ payload: MXDiagnosticPayload) {
        let timeRange = payload.timeStampBegin...payload.timeStampEnd

        // Process each diagnostic type
        payload.crashDiagnostics?.forEach { processCrash($0, timeRange: timeRange) }
        payload.hangDiagnostics?.forEach { processHang($0, timeRange: timeRange) }
        payload.cpuExceptionDiagnostics?.forEach { processCPUException($0, timeRange: timeRange) }
        payload.diskWriteExceptionDiagnostics?.forEach { processDiskWriteException($0, timeRange: timeRange) }
    }

    @available(iOS 14.0, *)
    private func processCrash(_ crash: MXCrashDiagnostic, timeRange: ClosedRange<Date>) {
        let diagnostic = DiagnosticReport(
            type: .crash,
            timeRange: timeRange,
            callStack: crash.callStackTree.jsonRepresentation(),
            attributes: [
                "exception_type": crash.exceptionType?.intValue as Any,
                "exception_code": crash.exceptionCode?.intValue as Any,
                "signal": crash.signal?.intValue as Any,
                "termination_reason": crash.terminationReason as Any
            ]
        )
        correlateAndUpload(diagnostic)
    }

    @available(iOS 14.0, *)
    private func processHang(_ hang: MXHangDiagnostic, timeRange: ClosedRange<Date>) {
        let diagnostic = DiagnosticReport(
            type: .hang,
            timeRange: timeRange,
            callStack: hang.callStackTree.jsonRepresentation(),
            attributes: [
                "duration_ms": hang.hangDuration.converted(to: .milliseconds).value
            ]
        )
        correlateAndUpload(diagnostic)
    }

    @available(iOS 14.0, *)
    private func processCPUException(_ exception: MXCPUExceptionDiagnostic, timeRange: ClosedRange<Date>) {
        let diagnostic = DiagnosticReport(
            type: .cpuException,
            timeRange: timeRange,
            callStack: exception.callStackTree.jsonRepresentation(),
            attributes: [
                "total_cpu_time_s": exception.totalCPUTime.converted(to: .seconds).value,
                "total_sampled_time_s": exception.totalSampledTime.converted(to: .seconds).value
            ]
        )
        correlateAndUpload(diagnostic)
    }

    @available(iOS 14.0, *)
    private func processDiskWriteException(_ exception: MXDiskWriteExceptionDiagnostic, timeRange: ClosedRange<Date>) {
        let diagnostic = DiagnosticReport(
            type: .diskWriteException,
            timeRange: timeRange,
            callStack: exception.callStackTree.jsonRepresentation(),
            attributes: [
                "total_writes_mb": exception.totalWritesCaused.converted(to: .megabytes).value
            ]
        )
        correlateAndUpload(diagnostic)
    }

    // MARK: - Correlation

    private func correlateAndUpload(_ diagnostic: DiagnosticReport) {
        // Choose your correlation strategy:
        // 1. Generic (timestamp-based)
        // 2. Sentry integration
        // 3. Datadog integration

        // Default: Generic correlation
        uploadToBackend(diagnostic)
    }

    private func uploadToBackend(_ diagnostic: DiagnosticReport) {
        // Implement your upload logic
    }
}

// MARK: - Supporting Types

struct DiagnosticReport {
    enum DiagnosticType: String {
        case crash, hang, cpuException, diskWriteException
    }

    let type: DiagnosticType
    let timeRange: ClosedRange<Date>
    let callStack: Data
    let attributes: [String: Any]

    var timeRangeStart: Date { timeRange.lowerBound }
    var timeRangeEnd: Date { timeRange.upperBound }
}
```

---

## AppDelegate Integration

```swift
// AppDelegate.swift
import UIKit

@main
class AppDelegate: UIResponder, UIApplicationDelegate {

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {

        // Start MetricKit subscriber
        MetricKitSubscriber.shared.start()

        return true
    }

    func applicationWillTerminate(_ application: UIApplication) {
        MetricKitSubscriber.shared.stop()
    }
}
```

---

## Sentry Correlation

```swift
import Sentry
import MetricKit

extension MetricKitSubscriber {

    @available(iOS 14.0, *)
    func uploadToSentry(_ diagnostic: DiagnosticReport) {
        let event = Event(level: diagnostic.type == .crash ? .error : .warning)

        event.message = SentryMessage(
            formatted: "MetricKit \(diagnostic.type.rawValue) detected"
        )

        event.tags = [
            "metrickit.type": diagnostic.type.rawValue,
            "metrickit.source": "system"
        ]

        // Add diagnostic attributes
        event.extra = diagnostic.attributes.mapValues { "\($0)" }

        // Attach call stack as JSON
        let attachment = Attachment(
            data: diagnostic.callStack,
            filename: "metrickit_\(diagnostic.type.rawValue)_callstack.json",
            contentType: "application/json"
        )

        SentrySDK.capture(event: event) { scope in
            scope.addAttachment(attachment)

            // Tag as supplementary diagnostic
            scope.setTag(value: "true", key: "metrickit.supplementary")

            // Add time range for correlation
            scope.setContext(value: [
                "time_range_start": ISO8601DateFormatter().string(from: diagnostic.timeRangeStart),
                "time_range_end": ISO8601DateFormatter().string(from: diagnostic.timeRangeEnd)
            ], key: "metrickit")
        }
    }
}
```

---

## Datadog Correlation

```swift
import DatadogLogs
import DatadogRUM
import MetricKit

extension MetricKitSubscriber {

    @available(iOS 14.0, *)
    func uploadToDatadog(_ diagnostic: DiagnosticReport) {
        let logger = Logger.create(
            with: Logger.Configuration(name: "metrickit")
        )

        // Base attributes
        var attributes: [String: Encodable] = [
            "metrickit.type": diagnostic.type.rawValue,
            "metrickit.time_range_start": diagnostic.timeRangeStart.timeIntervalSince1970,
            "metrickit.time_range_end": diagnostic.timeRangeEnd.timeIntervalSince1970
        ]

        // Add diagnostic-specific attributes
        for (key, value) in diagnostic.attributes {
            if let encodable = value as? Encodable {
                attributes["metrickit.\(key)"] = encodable as? any Encodable
            }
        }

        switch diagnostic.type {
        case .crash:
            // Report as RUM error
            RUMMonitor.shared().addError(
                message: "MetricKit crash diagnostic",
                type: "MetricKit.Crash",
                source: .custom,
                attributes: attributes
            )

        case .hang:
            // Report as warning log
            logger.warn("MetricKit hang detected", attributes: attributes)

        case .cpuException:
            logger.warn("MetricKit CPU exception", attributes: attributes)

        case .diskWriteException:
            logger.info("MetricKit disk write exception", attributes: attributes)
        }
    }
}
```

---

## Generic Backend Upload

```swift
import Foundation

extension MetricKitSubscriber {

    func uploadToBackend(_ diagnostic: DiagnosticReport) {
        Task {
            await uploadAsync(diagnostic)
        }
    }

    private func uploadAsync(_ diagnostic: DiagnosticReport) async {
        let endpoint = URL(string: "https://api.yourbackend.com/v1/metrickit")!

        var request = URLRequest(url: endpoint)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue(apiKey, forHTTPHeaderField: "Authorization")

        let payload: [String: Any] = [
            "type": diagnostic.type.rawValue,
            "time_range_start": ISO8601DateFormatter().string(from: diagnostic.timeRangeStart),
            "time_range_end": ISO8601DateFormatter().string(from: diagnostic.timeRangeEnd),
            "attributes": diagnostic.attributes,
            "call_stack_base64": diagnostic.callStack.base64EncodedString(),
            "app_version": Bundle.main.appVersion,
            "os_version": UIDevice.current.systemVersion,
            "device_model": UIDevice.current.model
        ]

        do {
            request.httpBody = try JSONSerialization.data(withJSONObject: payload)
            let (_, response) = try await URLSession.shared.data(for: request)

            if let httpResponse = response as? HTTPURLResponse,
               httpResponse.statusCode >= 400 {
                // Store locally for retry
                storeForRetry(diagnostic)
            }
        } catch {
            storeForRetry(diagnostic)
        }
    }

    private func storeForRetry(_ diagnostic: DiagnosticReport) {
        // TODO: Implement local persistence for retry
        // Example: Use UserDefaults or a local database
        //
        // let encoder = JSONEncoder()
        // if let data = try? encoder.encode(diagnostic) {
        //     let key = "metrickit_pending_\(UUID().uuidString)"
        //     UserDefaults.standard.set(data, forKey: key)
        // }
        print("⚠️ MetricKit: storeForRetry not implemented - diagnostic may be lost")
    }

    private var apiKey: String {
        // TODO: Replace with your actual API key
        // Recommended: Load from environment or secure storage
        // Example: ProcessInfo.processInfo.environment["METRICKIT_API_KEY"] ?? ""
        guard let key = ProcessInfo.processInfo.environment["METRICKIT_API_KEY"], !key.isEmpty else {
            assertionFailure("MetricKit API key not configured. Set METRICKIT_API_KEY environment variable.")
            return ""
        }
        return key
    }
}

private extension Bundle {
    var appVersion: String {
        infoDictionary?["CFBundleShortVersionString"] as? String ?? "unknown"
    }
}
```

---

## What NOT to Track

- Raw JSON payloads without validation (locale bug risk)
- Diagnostics with very short stack traces (<5 frames)
- Every metric payload field (focus on actionable data)
- PII that might be in stack trace function names

---

## Integration Points

- **[metrickit.md](../metrickit.md)** - Full reference documentation
- **[crash-reporting.md](../crash-reporting.md)** - Primary crash reporting patterns
- **[error-boundary.template.md](error-boundary.template.md)** - Error capture with context
