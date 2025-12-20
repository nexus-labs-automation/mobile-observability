# Observability Fundamentals

Core concepts for mobile observability: metrics vs logs vs traces, correlation strategies, and Jobs-to-be-Done framework for instrumentation.

## Table of Contents
1. [The Three Pillars](#the-three-pillars)
2. [Metrics](#metrics)
3. [Logs](#logs)
4. [Traces](#traces)
5. [Correlation Strategy](#correlation-strategy)
6. [Jobs-to-be-Done Framework](#jobs-to-be-done-framework)
7. [Instrumentation Checklist](#instrumentation-checklist)
8. [Sampling Strategies](#sampling-strategies)
9. [Data Model Design](#data-model-design)

---

## The Three Pillars

```
┌─────────────────────────────────────────────────────────────────┐
│                    Observability Pillars                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   METRICS              LOGS                 TRACES               │
│   ────────             ────                 ──────               │
│   Aggregated           Discrete             Request flow         │
│   measurements         events               across time          │
│                                                                  │
│   "How many?"          "What happened?"     "How long?"          │
│   "How fast?"          "In what order?"     "Where's the         │
│   "What %?"            "What context?"       bottleneck?"        │
│                                                                  │
│   ┌─────────┐          ┌─────────┐          ┌─────────┐         │
│   │ counter │          │ event   │          │ span    │         │
│   │ gauge   │          │ message │          │ trace   │         │
│   │ histogram│         │ breadcrumb│        │ context │         │
│   └─────────┘          └─────────┘          └─────────┘         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### When to Use Each

| Pillar | Use When | Mobile Examples |
|--------|----------|-----------------|
| **Metrics** | Need aggregates, dashboards, alerting | Crash-free rate, P95 latency, DAU |
| **Logs** | Need context, debugging, audit trail | User actions, state changes, errors |
| **Traces** | Need request flow, latency breakdown | Screen load, API calls, user journeys |

### The Fourth Pillar: Sessions

Mobile adds a critical fourth dimension:

```
┌─────────────────────────────────────────────────────────────────┐
│                         SESSION                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ session_id: "abc123"                                        ││
│  │ user_id: "user456"                                          ││
│  │ device_id: "device789"                                      ││
│  │ app_version: "2.1.0"                                        ││
│  │ start_time: 2024-01-15T10:30:00Z                            ││
│  │ ─────────────────────────────────────────────────────────── ││
│  │  ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐          ││
│  │  │Trace│───│Trace│───│Trace│───│Trace│───│Trace│          ││
│  │  └──┬──┘   └──┬──┘   └──┬──┘   └──┬──┘   └──┬──┘          ││
│  │     │         │         │         │         │              ││
│  │  [logs]    [logs]    [logs]    [logs]    [logs]           ││
│  │     │         │         │         │         │              ││
│  │  metrics   metrics   metrics   metrics   metrics          ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

---

## Metrics

### Types of Metrics

| Type | Description | Mobile Use Case |
|------|-------------|-----------------|
| **Counter** | Monotonically increasing | Button taps, API calls, errors |
| **Gauge** | Point-in-time value | Memory usage, battery level |
| **Histogram** | Distribution of values | Response times, payload sizes |
| **Timer** | Duration measurements | Screen load time, API latency |

### iOS Metrics Implementation

```swift
// MetricsManager.swift
import Foundation

final class MetricsManager {
    static let shared = MetricsManager()

    private var counters: [String: Int] = [:]
    private var gauges: [String: Double] = [:]
    private var histograms: [String: [Double]] = [:]
    private let queue = DispatchQueue(label: "metrics.manager")

    // MARK: - Counter

    func increment(_ name: String, tags: [String: String] = [:]) {
        let key = metricKey(name, tags: tags)
        queue.async {
            self.counters[key, default: 0] += 1
        }
    }

    func increment(_ name: String, by value: Int, tags: [String: String] = [:]) {
        let key = metricKey(name, tags: tags)
        queue.async {
            self.counters[key, default: 0] += value
        }
    }

    // MARK: - Gauge

    func gauge(_ name: String, value: Double, tags: [String: String] = [:]) {
        let key = metricKey(name, tags: tags)
        queue.async {
            self.gauges[key] = value
        }
    }

    // MARK: - Histogram / Timer

    func record(_ name: String, value: Double, tags: [String: String] = [:]) {
        let key = metricKey(name, tags: tags)
        queue.async {
            self.histograms[key, default: []].append(value)
        }
    }

    func time<T>(_ name: String, tags: [String: String] = [:], operation: () throws -> T) rethrows -> T {
        let start = CFAbsoluteTimeGetCurrent()
        defer {
            let duration = (CFAbsoluteTimeGetCurrent() - start) * 1000
            record(name, value: duration, tags: tags)
        }
        return try operation()
    }

    func time<T>(_ name: String, tags: [String: String] = [:], operation: () async throws -> T) async rethrows -> T {
        let start = CFAbsoluteTimeGetCurrent()
        defer {
            let duration = (CFAbsoluteTimeGetCurrent() - start) * 1000
            record(name, value: duration, tags: tags)
        }
        return try await operation()
    }

    // MARK: - Flush

    func flush() -> MetricsPayload {
        queue.sync {
            let payload = MetricsPayload(
                counters: counters,
                gauges: gauges,
                histograms: histograms.mapValues { values in
                    HistogramSummary(
                        count: values.count,
                        sum: values.reduce(0, +),
                        min: values.min() ?? 0,
                        max: values.max() ?? 0,
                        p50: percentile(values, 0.50),
                        p95: percentile(values, 0.95),
                        p99: percentile(values, 0.99)
                    )
                }
            )

            // Reset counters and histograms, keep gauges
            counters.removeAll()
            histograms.removeAll()

            return payload
        }
    }

    private func metricKey(_ name: String, tags: [String: String]) -> String {
        if tags.isEmpty { return name }
        let tagString = tags.sorted { $0.key < $1.key }
            .map { "\($0.key)=\($0.value)" }
            .joined(separator: ",")
        return "\(name){\(tagString)}"
    }

    private func percentile(_ values: [Double], _ p: Double) -> Double {
        guard !values.isEmpty else { return 0 }
        let sorted = values.sorted()
        let index = Int(Double(sorted.count - 1) * p)
        return sorted[index]
    }
}

struct MetricsPayload: Codable {
    let counters: [String: Int]
    let gauges: [String: Double]
    let histograms: [String: HistogramSummary]
    let timestamp: Date = Date()
}

struct HistogramSummary: Codable {
    let count: Int
    let sum: Double
    let min: Double
    let max: Double
    let p50: Double
    let p95: Double
    let p99: Double
}
```

### Android Metrics Implementation

```kotlin
// MetricsManager.kt
import java.util.concurrent.ConcurrentHashMap
import java.util.concurrent.atomic.AtomicLong
import java.util.concurrent.atomic.AtomicReference

object MetricsManager {
    private val counters = ConcurrentHashMap<String, AtomicLong>()
    private val gauges = ConcurrentHashMap<String, AtomicReference<Double>>()
    private val histograms = ConcurrentHashMap<String, MutableList<Double>>()

    // Counter
    fun increment(name: String, tags: Map<String, String> = emptyMap()) {
        val key = metricKey(name, tags)
        counters.getOrPut(key) { AtomicLong(0) }.incrementAndGet()
    }

    fun increment(name: String, value: Long, tags: Map<String, String> = emptyMap()) {
        val key = metricKey(name, tags)
        counters.getOrPut(key) { AtomicLong(0) }.addAndGet(value)
    }

    // Gauge
    fun gauge(name: String, value: Double, tags: Map<String, String> = emptyMap()) {
        val key = metricKey(name, tags)
        gauges.getOrPut(key) { AtomicReference(0.0) }.set(value)
    }

    // Histogram / Timer
    fun record(name: String, value: Double, tags: Map<String, String> = emptyMap()) {
        val key = metricKey(name, tags)
        synchronized(histograms) {
            histograms.getOrPut(key) { mutableListOf() }.add(value)
        }
    }

    inline fun <T> time(
        name: String,
        tags: Map<String, String> = emptyMap(),
        operation: () -> T
    ): T {
        val start = System.nanoTime()
        try {
            return operation()
        } finally {
            val durationMs = (System.nanoTime() - start) / 1_000_000.0
            record(name, durationMs, tags)
        }
    }

    suspend inline fun <T> timeSuspend(
        name: String,
        tags: Map<String, String> = emptyMap(),
        crossinline operation: suspend () -> T
    ): T {
        val start = System.nanoTime()
        try {
            return operation()
        } finally {
            val durationMs = (System.nanoTime() - start) / 1_000_000.0
            record(name, durationMs, tags)
        }
    }

    fun flush(): MetricsPayload {
        val counterSnapshot = counters.mapValues { it.value.getAndSet(0) }
        val gaugeSnapshot = gauges.mapValues { it.value.get() }

        val histogramSnapshot = synchronized(histograms) {
            histograms.mapValues { (_, values) ->
                val sorted = values.sorted()
                HistogramSummary(
                    count = sorted.size,
                    sum = sorted.sum(),
                    min = sorted.minOrNull() ?: 0.0,
                    max = sorted.maxOrNull() ?: 0.0,
                    p50 = percentile(sorted, 0.50),
                    p95 = percentile(sorted, 0.95),
                    p99 = percentile(sorted, 0.99)
                ).also { values.clear() }
            }
        }

        return MetricsPayload(
            counters = counterSnapshot,
            gauges = gaugeSnapshot,
            histograms = histogramSnapshot
        )
    }

    private fun metricKey(name: String, tags: Map<String, String>): String {
        if (tags.isEmpty()) return name
        val tagString = tags.entries.sortedBy { it.key }
            .joinToString(",") { "${it.key}=${it.value}" }
        return "$name{$tagString}"
    }

    private fun percentile(sorted: List<Double>, p: Double): Double {
        if (sorted.isEmpty()) return 0.0
        val index = ((sorted.size - 1) * p).toInt()
        return sorted[index]
    }
}

data class MetricsPayload(
    val counters: Map<String, Long>,
    val gauges: Map<String, Double>,
    val histograms: Map<String, HistogramSummary>,
    val timestamp: Long = System.currentTimeMillis()
)

data class HistogramSummary(
    val count: Int,
    val sum: Double,
    val min: Double,
    val max: Double,
    val p50: Double,
    val p95: Double,
    val p99: Double
)
```

---

## Logs

### Log Levels

| Level | Purpose | Mobile Examples |
|-------|---------|-----------------|
| **Debug** | Development only | State dumps, variable values |
| **Info** | Normal operations | Screen views, user actions |
| **Warning** | Recoverable issues | Retry succeeded, fallback used |
| **Error** | Failures requiring attention | API errors, validation failures |
| **Fatal** | App cannot continue | Unrecoverable state, crash |

### Structured Logging

```swift
// iOS Structured Logger
struct LogEntry: Codable {
    let timestamp: Date
    let level: LogLevel
    let message: String
    let context: LogContext
    let attributes: [String: AnyCodable]

    enum LogLevel: String, Codable {
        case debug, info, warning, error, fatal
    }
}

struct LogContext: Codable {
    let sessionId: String
    let userId: String?
    let screenName: String
    let traceId: String?
    let spanId: String?
    let appVersion: String
    let osVersion: String
    let deviceModel: String
}

final class StructuredLogger {
    static let shared = StructuredLogger()

    private var buffer: [LogEntry] = []
    private let queue = DispatchQueue(label: "logger")
    private let maxBufferSize = 100

    func log(
        _ level: LogEntry.LogLevel,
        _ message: String,
        attributes: [String: Any] = [:],
        file: String = #file,
        function: String = #function,
        line: Int = #line
    ) {
        let entry = LogEntry(
            timestamp: Date(),
            level: level,
            message: message,
            context: currentContext(),
            attributes: attributes.mapValues { AnyCodable($0) }
        )

        queue.async {
            self.buffer.append(entry)
            if self.buffer.count >= self.maxBufferSize {
                self.flush()
            }
        }

        #if DEBUG
        print("[\(level.rawValue.uppercased())] \(message) - \(file):\(line)")
        #endif
    }

    func debug(_ message: String, _ attributes: [String: Any] = [:]) {
        log(.debug, message, attributes: attributes)
    }

    func info(_ message: String, _ attributes: [String: Any] = [:]) {
        log(.info, message, attributes: attributes)
    }

    func warning(_ message: String, _ attributes: [String: Any] = [:]) {
        log(.warning, message, attributes: attributes)
    }

    func error(_ message: String, _ attributes: [String: Any] = [:]) {
        log(.error, message, attributes: attributes)
    }

    func fatal(_ message: String, _ attributes: [String: Any] = [:]) {
        log(.fatal, message, attributes: attributes)
        flush() // Immediately flush fatal logs
    }

    private func currentContext() -> LogContext {
        LogContext(
            sessionId: SessionManager.shared.sessionId,
            userId: UserManager.shared.userId,
            screenName: NavigationState.shared.currentScreen,
            traceId: TraceContext.current?.traceId,
            spanId: TraceContext.current?.spanId,
            appVersion: Bundle.main.appVersion,
            osVersion: UIDevice.current.systemVersion,
            deviceModel: UIDevice.current.model
        )
    }

    func flush() {
        // Send buffer to backend
    }
}
```

```kotlin
// Android Structured Logger
data class LogEntry(
    val timestamp: Long = System.currentTimeMillis(),
    val level: LogLevel,
    val message: String,
    val context: LogContext,
    val attributes: Map<String, Any?>
) {
    enum class LogLevel { DEBUG, INFO, WARNING, ERROR, FATAL }
}

data class LogContext(
    val sessionId: String,
    val userId: String?,
    val screenName: String,
    val traceId: String?,
    val spanId: String?,
    val appVersion: String,
    val osVersion: String,
    val deviceModel: String
)

object StructuredLogger {
    private val buffer = mutableListOf<LogEntry>()
    private const val MAX_BUFFER_SIZE = 100

    fun log(
        level: LogEntry.LogLevel,
        message: String,
        attributes: Map<String, Any?> = emptyMap()
    ) {
        val entry = LogEntry(
            level = level,
            message = message,
            context = currentContext(),
            attributes = attributes
        )

        synchronized(buffer) {
            buffer.add(entry)
            if (buffer.size >= MAX_BUFFER_SIZE) {
                flush()
            }
        }

        if (BuildConfig.DEBUG) {
            Log.println(level.toAndroidLevel(), "App", message)
        }
    }

    fun debug(message: String, attributes: Map<String, Any?> = emptyMap()) =
        log(LogEntry.LogLevel.DEBUG, message, attributes)

    fun info(message: String, attributes: Map<String, Any?> = emptyMap()) =
        log(LogEntry.LogLevel.INFO, message, attributes)

    fun warning(message: String, attributes: Map<String, Any?> = emptyMap()) =
        log(LogEntry.LogLevel.WARNING, message, attributes)

    fun error(message: String, attributes: Map<String, Any?> = emptyMap()) =
        log(LogEntry.LogLevel.ERROR, message, attributes)

    fun fatal(message: String, attributes: Map<String, Any?> = emptyMap()) {
        log(LogEntry.LogLevel.FATAL, message, attributes)
        flush()
    }

    private fun currentContext() = LogContext(
        sessionId = SessionManager.sessionId,
        userId = UserManager.userId,
        screenName = NavigationState.currentScreen,
        traceId = TraceContext.current?.traceId,
        spanId = TraceContext.current?.spanId,
        appVersion = BuildConfig.VERSION_NAME,
        osVersion = Build.VERSION.RELEASE,
        deviceModel = Build.MODEL
    )

    private fun LogEntry.LogLevel.toAndroidLevel() = when (this) {
        LogEntry.LogLevel.DEBUG -> Log.DEBUG
        LogEntry.LogLevel.INFO -> Log.INFO
        LogEntry.LogLevel.WARNING -> Log.WARN
        LogEntry.LogLevel.ERROR -> Log.ERROR
        LogEntry.LogLevel.FATAL -> Log.ASSERT
    }

    fun flush() {
        // Send buffer to backend
    }
}
```

---

## Traces

### Trace Structure

```
Trace (trace_id: "abc123")
│
├── Span: "app_launch" (root span)
│   ├── Span: "dependency_init"
│   ├── Span: "config_fetch"
│   └── Span: "first_screen_render"
│       ├── Span: "view_init"
│       ├── Span: "data_fetch"
│       │   ├── Span: "http_request"
│       │   └── Span: "json_parse"
│       └── Span: "view_render"
│
└── Span: "user_action" (new root span, same trace)
    └── Span: "api_call"
```

### iOS Tracing Implementation

```swift
// Tracer.swift
import Foundation

final class Tracer {
    static let shared = Tracer()

    private var activeSpans: [String: Span] = [:]
    private let queue = DispatchQueue(label: "tracer")

    struct Span {
        let traceId: String
        let spanId: String
        let parentSpanId: String?
        let operation: String
        let description: String?
        let startTime: CFAbsoluteTime
        var endTime: CFAbsoluteTime?
        var status: SpanStatus = .ok
        var data: [String: Any] = [:]
        var tags: [String: String] = [:]

        enum SpanStatus: String {
            case ok, error, cancelled
        }

        var durationMs: Double? {
            guard let end = endTime else { return nil }
            return (end - startTime) * 1000
        }
    }

    func startTransaction(
        operation: String,
        description: String? = nil
    ) -> SpanContext {
        let traceId = UUID().uuidString.replacingOccurrences(of: "-", with: "").prefix(32).lowercased()
        let spanId = UUID().uuidString.replacingOccurrences(of: "-", with: "").prefix(16).lowercased()

        let span = Span(
            traceId: String(traceId),
            spanId: String(spanId),
            parentSpanId: nil,
            operation: operation,
            description: description,
            startTime: CFAbsoluteTimeGetCurrent()
        )

        queue.async {
            self.activeSpans[span.spanId] = span
        }

        let context = SpanContext(traceId: span.traceId, spanId: span.spanId)
        TraceContext.current = context

        return context
    }

    func startChild(
        operation: String,
        description: String? = nil,
        parent: SpanContext? = nil
    ) -> SpanContext {
        let parentContext = parent ?? TraceContext.current
        guard let traceId = parentContext?.traceId else {
            return startTransaction(operation: operation, description: description)
        }

        let spanId = UUID().uuidString.replacingOccurrences(of: "-", with: "").prefix(16).lowercased()

        let span = Span(
            traceId: traceId,
            spanId: String(spanId),
            parentSpanId: parentContext?.spanId,
            operation: operation,
            description: description,
            startTime: CFAbsoluteTimeGetCurrent()
        )

        queue.async {
            self.activeSpans[span.spanId] = span
        }

        return SpanContext(traceId: traceId, spanId: span.spanId)
    }

    func finishSpan(_ context: SpanContext, status: Span.SpanStatus = .ok) {
        queue.async {
            guard var span = self.activeSpans.removeValue(forKey: context.spanId) else { return }
            span.endTime = CFAbsoluteTimeGetCurrent()
            span.status = status

            // Export span
            self.exportSpan(span)
        }
    }

    func setData(_ key: String, value: Any, on context: SpanContext) {
        queue.async {
            self.activeSpans[context.spanId]?.data[key] = value
        }
    }

    func setTag(_ key: String, value: String, on context: SpanContext) {
        queue.async {
            self.activeSpans[context.spanId]?.tags[key] = value
        }
    }

    private func exportSpan(_ span: Span) {
        // Send to observability backend
    }
}

struct SpanContext {
    let traceId: String
    let spanId: String
}

enum TraceContext {
    static var current: SpanContext?
}

// Convenience extensions
extension Tracer {
    func trace<T>(
        _ operation: String,
        description: String? = nil,
        action: () throws -> T
    ) rethrows -> T {
        let context = startChild(operation: operation, description: description)
        do {
            let result = try action()
            finishSpan(context, status: .ok)
            return result
        } catch {
            setData("error", value: error.localizedDescription, on: context)
            finishSpan(context, status: .error)
            throw error
        }
    }

    func trace<T>(
        _ operation: String,
        description: String? = nil,
        action: () async throws -> T
    ) async rethrows -> T {
        let context = startChild(operation: operation, description: description)
        do {
            let result = try await action()
            finishSpan(context, status: .ok)
            return result
        } catch {
            setData("error", value: error.localizedDescription, on: context)
            finishSpan(context, status: .error)
            throw error
        }
    }
}
```

### Android Tracing Implementation

```kotlin
// Tracer.kt
import java.util.UUID
import java.util.concurrent.ConcurrentHashMap

object Tracer {
    private val activeSpans = ConcurrentHashMap<String, Span>()

    data class Span(
        val traceId: String,
        val spanId: String,
        val parentSpanId: String?,
        val operation: String,
        val description: String?,
        val startTimeNanos: Long = System.nanoTime(),
        var endTimeNanos: Long? = null,
        var status: SpanStatus = SpanStatus.OK,
        val data: MutableMap<String, Any?> = mutableMapOf(),
        val tags: MutableMap<String, String> = mutableMapOf()
    ) {
        enum class SpanStatus { OK, ERROR, CANCELLED }

        val durationMs: Double?
            get() = endTimeNanos?.let { (it - startTimeNanos) / 1_000_000.0 }
    }

    fun startTransaction(
        operation: String,
        description: String? = null
    ): SpanContext {
        val traceId = UUID.randomUUID().toString().replace("-", "").take(32)
        val spanId = UUID.randomUUID().toString().replace("-", "").take(16)

        val span = Span(
            traceId = traceId,
            spanId = spanId,
            parentSpanId = null,
            operation = operation,
            description = description
        )

        activeSpans[spanId] = span
        val context = SpanContext(traceId, spanId)
        TraceContext.current = context

        return context
    }

    fun startChild(
        operation: String,
        description: String? = null,
        parent: SpanContext? = null
    ): SpanContext {
        val parentContext = parent ?: TraceContext.current
            ?: return startTransaction(operation, description)

        val spanId = UUID.randomUUID().toString().replace("-", "").take(16)

        val span = Span(
            traceId = parentContext.traceId,
            spanId = spanId,
            parentSpanId = parentContext.spanId,
            operation = operation,
            description = description
        )

        activeSpans[spanId] = span
        return SpanContext(parentContext.traceId, spanId)
    }

    fun finishSpan(context: SpanContext, status: Span.SpanStatus = Span.SpanStatus.OK) {
        activeSpans.remove(context.spanId)?.let { span ->
            span.endTimeNanos = System.nanoTime()
            span.status = status
            exportSpan(span)
        }
    }

    fun setData(key: String, value: Any?, context: SpanContext) {
        activeSpans[context.spanId]?.data?.set(key, value)
    }

    fun setTag(key: String, value: String, context: SpanContext) {
        activeSpans[context.spanId]?.tags?.set(key, value)
    }

    private fun exportSpan(span: Span) {
        // Send to observability backend
    }
}

data class SpanContext(
    val traceId: String,
    val spanId: String
)

object TraceContext {
    var current: SpanContext? = null
}

// Extension functions
inline fun <T> Tracer.trace(
    operation: String,
    description: String? = null,
    action: () -> T
): T {
    val context = startChild(operation, description)
    return try {
        action().also { finishSpan(context, Tracer.Span.SpanStatus.OK) }
    } catch (e: Exception) {
        setData("error", e.message, context)
        finishSpan(context, Tracer.Span.SpanStatus.ERROR)
        throw e
    }
}

suspend inline fun <T> Tracer.traceSuspend(
    operation: String,
    description: String? = null,
    crossinline action: suspend () -> T
): T {
    val context = startChild(operation, description)
    return try {
        action().also { finishSpan(context, Tracer.Span.SpanStatus.OK) }
    } catch (e: Exception) {
        setData("error", e.message, context)
        finishSpan(context, Tracer.Span.SpanStatus.ERROR)
        throw e
    }
}
```

---

## Correlation Strategy

### The Correlation Context

Every piece of telemetry should carry context that links it to:
1. **Session** - The app session
2. **User** - The authenticated user (if any)
3. **Device** - The physical device
4. **Trace** - The current operation flow

```swift
// iOS Correlation Context
struct CorrelationContext: Codable {
    let sessionId: String
    let userId: String?
    let deviceId: String
    let traceId: String?
    let spanId: String?
    let screenName: String
    let appVersion: String
    let buildNumber: String
    let osVersion: String
    let deviceModel: String
    let locale: String
    let timezone: String

    static var current: CorrelationContext {
        CorrelationContext(
            sessionId: SessionManager.shared.sessionId,
            userId: UserManager.shared.userId,
            deviceId: DeviceInfo.deviceId,
            traceId: TraceContext.current?.traceId,
            spanId: TraceContext.current?.spanId,
            screenName: NavigationState.shared.currentScreen,
            appVersion: Bundle.main.appVersion,
            buildNumber: Bundle.main.buildNumber,
            osVersion: UIDevice.current.systemVersion,
            deviceModel: DeviceInfo.modelName,
            locale: Locale.current.identifier,
            timezone: TimeZone.current.identifier
        )
    }
}

// Extension for adding to HTTP headers
extension CorrelationContext {
    var httpHeaders: [String: String] {
        var headers: [String: String] = [
            "X-Session-Id": sessionId,
            "X-Device-Id": deviceId,
            "X-App-Version": appVersion
        ]
        if let userId = userId { headers["X-User-Id"] = userId }
        if let traceId = traceId { headers["X-Trace-Id"] = traceId }
        if let spanId = spanId { headers["X-Span-Id"] = spanId }
        return headers
    }
}
```

```kotlin
// Android Correlation Context
data class CorrelationContext(
    val sessionId: String,
    val userId: String?,
    val deviceId: String,
    val traceId: String?,
    val spanId: String?,
    val screenName: String,
    val appVersion: String,
    val buildNumber: String,
    val osVersion: String,
    val deviceModel: String,
    val locale: String,
    val timezone: String
) {
    companion object {
        val current: CorrelationContext
            get() = CorrelationContext(
                sessionId = SessionManager.sessionId,
                userId = UserManager.userId,
                deviceId = DeviceInfo.deviceId,
                traceId = TraceContext.current?.traceId,
                spanId = TraceContext.current?.spanId,
                screenName = NavigationState.currentScreen,
                appVersion = BuildConfig.VERSION_NAME,
                buildNumber = BuildConfig.VERSION_CODE.toString(),
                osVersion = Build.VERSION.RELEASE,
                deviceModel = Build.MODEL,
                locale = Locale.getDefault().toString(),
                timezone = TimeZone.getDefault().id
            )
    }

    val httpHeaders: Map<String, String>
        get() = buildMap {
            put("X-Session-Id", sessionId)
            put("X-Device-Id", deviceId)
            put("X-App-Version", appVersion)
            userId?.let { put("X-User-Id", it) }
            traceId?.let { put("X-Trace-Id", it) }
            spanId?.let { put("X-Span-Id", it) }
        }
}
```

### Client-Backend Correlation

```
Mobile App                           Backend Service
─────────────────────────────────────────────────────────────────

  ┌─────────────────┐               ┌─────────────────┐
  │ Start Span      │               │                 │
  │ trace_id: abc   │───HTTP───────▶│ Continue Span   │
  │ span_id: 123    │  Headers:     │ trace_id: abc   │
  │                 │  X-Trace-Id   │ parent_id: 123  │
  │                 │  X-Span-Id    │ span_id: 456    │
  └─────────────────┘               └─────────────────┘
         │                                  │
         │                                  │
         ▼                                  ▼
  ┌─────────────────┐               ┌─────────────────┐
  │ Observability   │               │ Observability   │
  │ Backend         │◀──Unified────▶│ Backend         │
  │                 │   View        │                 │
  └─────────────────┘               └─────────────────┘
```

---

## Jobs-to-be-Done Framework

### Instrumentation JTBD

Ask: **"What job will this telemetry help me do?"**

| Job | Telemetry Type | Example |
|-----|----------------|---------|
| **Detect problems** | Alerts on metrics | Crash rate > 1%, P95 > 2s |
| **Diagnose root cause** | Traces + logs | Why did checkout fail? |
| **Measure impact** | Metrics | How many users affected? |
| **Validate fixes** | Before/after metrics | Did the fix work? |
| **Understand behavior** | Event logs | What did user do before crash? |

### The Diagnostic Question

Before adding instrumentation, ask:

> "When something breaks, what questions will I need to answer?"

Common questions and their required telemetry:

| Question | Required Telemetry |
|----------|-------------------|
| "What was the user doing?" | Breadcrumbs, screen name, user journey |
| "What was the app state?" | Gauges (memory, battery), feature flags |
| "What happened before this?" | Ordered logs, trace spans |
| "How long did it take?" | Timer metrics, span durations |
| "Who is affected?" | User/device/session context |
| "How often does this happen?" | Counters, error rates |
| "What version/device/OS?" | Context tags |

## Instrumentation Checklist

Use this template for each feature/screen:

```markdown
# Instrumentation Checklist: [Feature Name]

### Detection (Can I know when it breaks?)
- [ ] Error tracking with context
- [ ] Performance threshold alerts
- [ ] Availability monitoring

### Diagnosis (Can I find out why?)
- [ ] Breadcrumbs for user actions
- [ ] State snapshots at key points
- [ ] Trace spans for operations

### Measurement (Can I quantify impact?)
- [ ] Success/failure counters
- [ ] Latency histograms
- [ ] User/session attribution

### Context (Can I reproduce?)
- [ ] Screen name
- [ ] User journey stage
- [ ] Feature flag state
- [ ] A/B test variant
```

---

## Sampling Strategies

### Why Sample?

| Concern | Sampling Helps |
|---------|----------------|
| Battery | Less data = less processing |
| Bandwidth | Less upload = faster sync |
| Cost | Less ingestion = lower bill |
| Noise | Less data = clearer signals |

### Sampling Approaches

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **Head-based** | Decide at trace start | General performance data |
| **Tail-based** | Decide after trace completes | Keep slow/error traces |
| **Rate limiting** | Max N per time window | High-volume events |
| **Priority** | Always keep certain types | Errors, slow transactions |

### iOS Sampling Implementation

```swift
// Sampler.swift
final class Sampler {
    static let shared = Sampler()

    struct Config {
        var defaultRate: Double = 0.1  // 10%
        var errorRate: Double = 1.0    // 100% of errors
        var slowThresholdMs: Double = 1000
        var slowRate: Double = 1.0     // 100% of slow
        var rateByOperation: [String: Double] = [:]
    }

    var config = Config()

    func shouldSample(
        operation: String,
        isError: Bool = false,
        durationMs: Double? = nil
    ) -> Bool {
        // Always sample errors
        if isError && Double.random(in: 0...1) < config.errorRate {
            return true
        }

        // Always sample slow operations
        if let duration = durationMs, duration > config.slowThresholdMs {
            if Double.random(in: 0...1) < config.slowRate {
                return true
            }
        }

        // Check operation-specific rate
        if let rate = config.rateByOperation[operation] {
            return Double.random(in: 0...1) < rate
        }

        // Fall back to default rate
        return Double.random(in: 0...1) < config.defaultRate
    }

    func decideSamplingAtTraceStart(operation: String) -> SamplingDecision {
        let sampled = Double.random(in: 0...1) < (config.rateByOperation[operation] ?? config.defaultRate)
        return SamplingDecision(
            sampled: sampled,
            canUpgrade: true  // Can upgrade to sampled if error/slow
        )
    }
}

struct SamplingDecision {
    var sampled: Bool
    let canUpgrade: Bool

    mutating func upgradeIfNeeded(isError: Bool, isSlow: Bool) {
        if !sampled && canUpgrade && (isError || isSlow) {
            sampled = true
        }
    }
}
```

### Recommended Sampling Rates

| Telemetry Type | Dev | Staging | Production |
|----------------|-----|---------|------------|
| Errors | 100% | 100% | 100% |
| Crashes | 100% | 100% | 100% |
| Slow traces (>P95) | 100% | 100% | 100% |
| Normal traces | 100% | 50% | 10-20% |
| Debug logs | 100% | 10% | 0% |
| Info logs | 100% | 50% | 10% |
| Metrics | 100% | 100% | 100% |

---

## Data Model Design

### Event Schema

```json
{
  "event_id": "uuid",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "type": "span|log|metric|error",

  "context": {
    "session_id": "string",
    "user_id": "string|null",
    "device_id": "string",
    "trace_id": "string|null",
    "span_id": "string|null",
    "parent_span_id": "string|null"
  },

  "device": {
    "platform": "ios|android",
    "os_version": "17.2",
    "model": "iPhone 15 Pro",
    "manufacturer": "Apple",
    "screen_density": 3.0,
    "locale": "en_US",
    "timezone": "America/Los_Angeles"
  },

  "app": {
    "version": "2.1.0",
    "build": "123",
    "environment": "production",
    "release_stage": "stable"
  },

  "payload": {
    // Type-specific data
  }
}
```

### Span Payload

```json
{
  "operation": "http.client",
  "description": "GET /api/products",
  "start_time": "2024-01-15T10:30:00.000Z",
  "end_time": "2024-01-15T10:30:00.250Z",
  "duration_ms": 250,
  "status": "ok|error|cancelled",
  "data": {
    "http.method": "GET",
    "http.url": "/api/products",
    "http.status_code": 200,
    "http.response_size": 4096
  },
  "tags": {
    "screen": "ProductList",
    "feature": "catalog"
  }
}
```

### Metric Payload

```json
{
  "name": "api.request.duration",
  "type": "histogram",
  "unit": "milliseconds",
  "value": {
    "count": 100,
    "sum": 25000,
    "min": 50,
    "max": 1200,
    "p50": 180,
    "p95": 450,
    "p99": 890
  },
  "tags": {
    "endpoint": "/api/products",
    "method": "GET"
  }
}
```

---

## Summary

| Concept | Key Insight |
|---------|-------------|
| **Three Pillars** | Metrics for "how much", logs for "what happened", traces for "how long" |
| **Sessions** | Mobile's fourth pillar - ties everything together |
| **Correlation** | Every event needs session_id, trace_id, device context |
| **JTBD** | Instrument to answer: "What was user doing when this broke?" |
| **Sampling** | 100% errors, 100% slow, 10-20% normal in production |
| **Schema** | Consistent event structure enables unified querying |

---

## Integration Points

- **[ui-performance.md](ui-performance.md)** - Navigation and render traces
- **[crash-reporting.md](crash-reporting.md)** - Error events and breadcrumbs
- **[performance.md](performance.md)** - App lifecycle and network spans
- **[mobile-challenges.md](mobile-challenges.md)** - Offline buffering, sampling adaptation
