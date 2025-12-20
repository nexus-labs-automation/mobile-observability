# Performance Monitoring

Comprehensive guide to performance instrumentation: app start, screen load, network requests, and resource monitoring.

## Table of Contents
1. [App Start Performance](#app-start-performance)
2. [Screen Load Tracking](#screen-load-tracking)
3. [Network Performance](#network-performance)
4. [Resource Monitoring](#resource-monitoring)
5. [Background Performance](#background-performance)
6. [Performance Budgets](#performance-budgets)

---

## App Start Performance

### Start Types

| Type | Definition | Target | Typical Cause |
|------|------------|--------|---------------|
| **Cold** | Process created from scratch | <2s | First launch, after termination |
| **Warm** | Process exists, activity recreated | <1s | After background kill |
| **Hot** | Process + activity exist | <500ms | After recent background |

### iOS App Start Measurement

```swift
// AppStartTracker.swift
import Foundation
import os.signpost

final class AppStartTracker {
    static let shared = AppStartTracker()

    private let signpostLog = OSLog(subsystem: Bundle.main.bundleIdentifier!, category: "AppStart")

    // Process start time (set in main.swift or AppDelegate)
    var processStartTime: CFAbsoluteTime = 0

    private var phases: [String: CFAbsoluteTime] = [:]
    private var startType: StartType = .cold
    private var isComplete = false

    enum StartType: String {
        case cold, warm, hot
    }

    // MARK: - Phase Markers

    /// Call from AppDelegate didFinishLaunching start
    func markPreMain() {
        phases["pre_main"] = CFAbsoluteTimeGetCurrent()
    }

    /// Call after basic app setup
    func markAppDelegateComplete() {
        phases["app_delegate"] = CFAbsoluteTimeGetCurrent()
    }

    /// Call when first view controller loads
    func markFirstViewControllerLoad() {
        phases["first_vc_load"] = CFAbsoluteTimeGetCurrent()
    }

    /// Call when first frame is rendered
    func markFirstFrame() {
        phases["first_frame"] = CFAbsoluteTimeGetCurrent()
    }

    /// Call when app is interactive (user can use it)
    func markInteractive() {
        guard !isComplete else { return }
        isComplete = true

        phases["interactive"] = CFAbsoluteTimeGetCurrent()

        reportAppStart()
    }

    // MARK: - Reporting

    private func reportAppStart() {
        guard let interactiveTime = phases["interactive"] else { return }

        let totalTime = (interactiveTime - processStartTime) * 1000

        // Calculate phase durations
        let phaseDurations = calculatePhaseDurations()

        // Signpost
        os_signpost(.event, log: signpostLog, name: "AppStart",
                    "type: %{public}s, duration_ms: %.2f", startType.rawValue, totalTime)

        // Metrics
        Observability.recordMetric(
            name: "app_start.duration",
            value: totalTime,
            unit: .milliseconds,
            tags: [
                "start_type": startType.rawValue,
                "bucket": durationBucket(totalTime)
            ]
        )

        // Phase breakdown
        for (phase, duration) in phaseDurations {
            Observability.recordMetric(
                name: "app_start.phase",
                value: duration,
                unit: .milliseconds,
                tags: [
                    "phase": phase,
                    "start_type": startType.rawValue
                ]
            )
        }

        // Trace
        let transaction = Tracer.shared.startTransaction(
            operation: "app.start.\(startType.rawValue)",
            description: "App Start"
        )
        Tracer.shared.setData("duration_ms", value: totalTime, on: transaction)
        Tracer.shared.setData("phases", value: phaseDurations, on: transaction)
        Tracer.shared.finishSpan(transaction)

        // Alert on slow starts
        if totalTime > 4000 {
            Observability.captureMessage(
                "Slow app start",
                level: .warning,
                extras: [
                    "start_type": startType.rawValue,
                    "duration_ms": totalTime,
                    "phases": phaseDurations
                ]
            )
        }
    }

    private func calculatePhaseDurations() -> [String: Double] {
        var durations: [String: Double] = [:]
        let orderedPhases = ["pre_main", "app_delegate", "first_vc_load", "first_frame", "interactive"]

        var lastTime = processStartTime
        for phase in orderedPhases {
            if let phaseTime = phases[phase] {
                durations[phase] = (phaseTime - lastTime) * 1000
                lastTime = phaseTime
            }
        }

        return durations
    }

    private func durationBucket(_ ms: Double) -> String {
        switch ms {
        case ..<1000: return "fast"
        case ..<2000: return "good"
        case ..<4000: return "slow"
        default: return "very_slow"
        }
    }

    // MARK: - Start Type Detection

    func determineStartType() {
        // Check if this is a warm/hot start
        if wasRecentlyBackgrounded() {
            startType = wasActivityRecreated() ? .warm : .hot
        } else {
            startType = .cold
        }
    }

    private func wasRecentlyBackgrounded() -> Bool {
        // Check UserDefaults for last background time
        let lastBackground = UserDefaults.standard.double(forKey: "last_background_time")
        return lastBackground > 0
    }

    private func wasActivityRecreated() -> Bool {
        // iOS doesn't have direct equivalent; use scene state
        return false
    }
}

// main.swift integration
import UIKit

// Capture process start time as early as possible
AppStartTracker.shared.processStartTime = CFAbsoluteTimeGetCurrent()

UIApplicationMain(
    CommandLine.argc,
    CommandLine.unsafeArgv,
    nil,
    NSStringFromClass(AppDelegate.self)
)
```

### Android App Start Measurement

```kotlin
// AppStartTracker.kt
import android.app.Application
import android.os.Process
import android.os.SystemClock
import androidx.lifecycle.ProcessLifecycleOwner
import androidx.lifecycle.DefaultLifecycleObserver
import androidx.lifecycle.LifecycleOwner

object AppStartTracker : DefaultLifecycleObserver {
    // Process start time in realtime millis
    private val processStartTime = SystemClock.elapsedRealtime()

    private val phases = mutableMapOf<String, Long>()
    private var startType = StartType.COLD
    private var isComplete = false
    private var wasInBackground = false

    enum class StartType { COLD, WARM, HOT }

    fun init(application: Application) {
        ProcessLifecycleOwner.get().lifecycle.addObserver(this)
    }

    // Phase markers
    fun markApplicationCreate() {
        phases["application_create"] = SystemClock.elapsedRealtime()
    }

    fun markActivityCreate() {
        phases["activity_create"] = SystemClock.elapsedRealtime()
    }

    fun markFirstFrame() {
        phases["first_frame"] = SystemClock.elapsedRealtime()
    }

    fun markInteractive() {
        if (isComplete) return
        isComplete = true

        phases["interactive"] = SystemClock.elapsedRealtime()
        reportAppStart()
    }

    override fun onStart(owner: LifecycleOwner) {
        if (wasInBackground) {
            startType = StartType.HOT
            // Reset for warm/hot start measurement
            phases.clear()
            phases["resume_start"] = SystemClock.elapsedRealtime()
            isComplete = false
        }
    }

    override fun onStop(owner: LifecycleOwner) {
        wasInBackground = true
    }

    private fun reportAppStart() {
        val interactiveTime = phases["interactive"] ?: return
        val startTime = if (startType == StartType.COLD) processStartTime else phases["resume_start"] ?: processStartTime

        val totalTimeMs = (interactiveTime - startTime).toDouble()
        val phaseDurations = calculatePhaseDurations(startTime)

        // Systrace
        Trace.beginSection("AppStart:${startType.name}")
        Trace.endSection()

        // Metrics
        Observability.recordMetric(
            name = "app_start.duration",
            value = totalTimeMs,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf(
                "start_type" to startType.name.lowercase(),
                "bucket" to durationBucket(totalTimeMs)
            )
        )

        // Phase breakdown
        phaseDurations.forEach { (phase, duration) ->
            Observability.recordMetric(
                name = "app_start.phase",
                value = duration,
                unit = MetricUnit.MILLISECONDS,
                tags = mapOf(
                    "phase" to phase,
                    "start_type" to startType.name.lowercase()
                )
            )
        }

        // Transaction
        val transaction = Tracer.startTransaction(
            operation = "app.start.${startType.name.lowercase()}",
            description = "App Start"
        )
        Tracer.setData("duration_ms", totalTimeMs, transaction)
        Tracer.setData("phases", phaseDurations, transaction)
        Tracer.finishSpan(transaction)

        // Alert on slow starts
        if (totalTimeMs > 4000) {
            Observability.captureMessage(
                message = "Slow app start",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "start_type" to startType.name,
                    "duration_ms" to totalTimeMs,
                    "phases" to phaseDurations
                )
            )
        }
    }

    private fun calculatePhaseDurations(startTime: Long): Map<String, Double> {
        val orderedPhases = listOf("application_create", "activity_create", "first_frame", "interactive")
        val durations = mutableMapOf<String, Double>()

        var lastTime = startTime
        orderedPhases.forEach { phase ->
            phases[phase]?.let { phaseTime ->
                durations[phase] = (phaseTime - lastTime).toDouble()
                lastTime = phaseTime
            }
        }

        return durations
    }

    private fun durationBucket(ms: Double): String = when {
        ms < 1000 -> "fast"
        ms < 2000 -> "good"
        ms < 4000 -> "slow"
        else -> "very_slow"
    }
}

// Application integration
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        AppStartTracker.markApplicationCreate()
        AppStartTracker.init(this)
    }
}

// Activity integration with ReportFullyDrawn
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        AppStartTracker.markActivityCreate()

        setContent {
            MyApp {
                // When content is ready
                LaunchedEffect(Unit) {
                    AppStartTracker.markFirstFrame()
                    reportFullyDrawn() // Android framework method
                    AppStartTracker.markInteractive()
                }
            }
        }
    }
}
```

---

## Screen Load Tracking

### Screen Load Phases

```
User initiates navigation
         │
         ▼
    ┌─────────────────────────────────────────────────────────────┐
    │  INIT  │  DATA FETCH  │  RENDER  │  INTERACTIVE            │
    └─────────────────────────────────────────────────────────────┘
         │         │            │              │
         │         │            │              └── Time to Interactive (TTI)
         │         │            └── First Contentful Paint (FCP)
         │         └── Data Ready
         └── View Created

    Target: <400ms from tap to interactive for P95
```

### iOS Screen Load Tracker

```swift
// ScreenLoadTracker.swift
import Foundation
import os.signpost

final class ScreenLoadTracker {
    private let screenName: String
    private let startTime: CFAbsoluteTime
    private var phases: [String: CFAbsoluteTime] = [:]
    private var isComplete = false
    private let signpostID: OSSignpostID
    private let signpostLog: OSLog

    init(screenName: String) {
        self.screenName = screenName
        self.startTime = CFAbsoluteTimeGetCurrent()
        self.signpostLog = OSLog(subsystem: Bundle.main.bundleIdentifier!, category: "ScreenLoad")
        self.signpostID = OSSignpostID(log: signpostLog)

        os_signpost(.begin, log: signpostLog, name: "ScreenLoad", signpostID: signpostID,
                    "screen: %{public}s", screenName)
    }

    func markViewCreated() {
        phases["view_created"] = CFAbsoluteTimeGetCurrent()
    }

    func markViewAppeared() {
        phases["view_appeared"] = CFAbsoluteTimeGetCurrent()
    }

    func markDataReady() {
        phases["data_ready"] = CFAbsoluteTimeGetCurrent()
    }

    func markFirstContentfulPaint() {
        phases["fcp"] = CFAbsoluteTimeGetCurrent()
    }

    func markInteractive(metadata: [String: Any] = [:]) {
        guard !isComplete else { return }
        isComplete = true

        phases["interactive"] = CFAbsoluteTimeGetCurrent()

        os_signpost(.end, log: signpostLog, name: "ScreenLoad", signpostID: signpostID,
                    "screen: %{public}s, status: complete", screenName)

        reportScreenLoad(metadata: metadata)
    }

    private func reportScreenLoad(metadata: [String: Any]) {
        guard let interactiveTime = phases["interactive"] else { return }

        let totalTime = (interactiveTime - startTime) * 1000
        let phaseDurations = calculatePhaseDurations()

        // Metric
        Observability.recordMetric(
            name: "screen_load.duration",
            value: totalTime,
            unit: .milliseconds,
            tags: [
                "screen": screenName,
                "bucket": durationBucket(totalTime)
            ]
        )

        // Phase breakdown
        for (phase, duration) in phaseDurations {
            Observability.recordMetric(
                name: "screen_load.phase",
                value: duration,
                unit: .milliseconds,
                tags: ["screen": screenName, "phase": phase]
            )
        }

        // Transaction
        let span = Tracer.shared.startChild(
            operation: "ui.load",
            description: screenName
        )
        Tracer.shared.setData("duration_ms", value: totalTime, on: span)
        Tracer.shared.setTag("screen", value: screenName, on: span)
        for (phase, duration) in phaseDurations {
            Tracer.shared.setData("phase.\(phase)_ms", value: duration, on: span)
        }
        Tracer.shared.finishSpan(span)

        // Alert on slow screens
        let threshold = ScreenLoadThresholds.threshold(for: screenName)
        if totalTime > threshold {
            Observability.captureMessage(
                "Slow screen load: \(screenName)",
                level: .warning,
                extras: [
                    "screen": screenName,
                    "duration_ms": totalTime,
                    "threshold_ms": threshold,
                    "phases": phaseDurations,
                    "metadata": metadata
                ]
            )
        }
    }

    private func calculatePhaseDurations() -> [String: Double] {
        var durations: [String: Double] = [:]
        let orderedPhases = ["view_created", "view_appeared", "data_ready", "fcp", "interactive"]

        var lastTime = startTime
        for phase in orderedPhases {
            if let phaseTime = phases[phase] {
                durations[phase] = (phaseTime - lastTime) * 1000
                lastTime = phaseTime
            }
        }

        return durations
    }

    private func durationBucket(_ ms: Double) -> String {
        switch ms {
        case ..<200: return "fast"
        case ..<400: return "good"
        case ..<800: return "acceptable"
        case ..<1500: return "slow"
        default: return "very_slow"
        }
    }
}

// Per-screen thresholds
enum ScreenLoadThresholds {
    static func threshold(for screen: String) -> Double {
        switch screen {
        case "HomeScreen": return 1000
        case "ProductDetail": return 800
        case "Checkout": return 1200
        case "Search": return 600
        default: return 1000
        }
    }
}

// SwiftUI Integration
struct TrackedScreen<Content: View>: View {
    let screenName: String
    let tracker: ScreenLoadTracker
    let content: () -> Content

    @State private var isLoaded = false

    init(screenName: String, @ViewBuilder content: @escaping () -> Content) {
        self.screenName = screenName
        self.tracker = ScreenLoadTracker(screenName: screenName)
        self.content = content
    }

    var body: some View {
        content()
            .onAppear {
                tracker.markViewAppeared()
            }
            .background(
                GeometryReader { _ in
                    Color.clear.onAppear {
                        if !isLoaded {
                            isLoaded = true
                            tracker.markFirstContentfulPaint()
                        }
                    }
                }
            )
    }

    func markDataReady() {
        tracker.markDataReady()
    }

    func markInteractive(metadata: [String: Any] = [:]) {
        tracker.markInteractive(metadata: metadata)
    }
}
```

### Android Screen Load Tracker

```kotlin
// ScreenLoadTracker.kt
import android.os.Trace

class ScreenLoadTracker(private val screenName: String) {
    private val startTimeNanos = System.nanoTime()
    private val phases = mutableMapOf<String, Long>()
    private var isComplete = false

    init {
        Trace.beginAsyncSection("ScreenLoad:$screenName", screenName.hashCode())
    }

    fun markViewCreated() {
        phases["view_created"] = System.nanoTime()
    }

    fun markViewResumed() {
        phases["view_resumed"] = System.nanoTime()
    }

    fun markDataReady() {
        phases["data_ready"] = System.nanoTime()
    }

    fun markFirstContentfulPaint() {
        phases["fcp"] = System.nanoTime()
    }

    fun markInteractive(metadata: Map<String, Any> = emptyMap()) {
        if (isComplete) return
        isComplete = true

        phases["interactive"] = System.nanoTime()

        Trace.endAsyncSection("ScreenLoad:$screenName", screenName.hashCode())

        reportScreenLoad(metadata)
    }

    private fun reportScreenLoad(metadata: Map<String, Any>) {
        val interactiveTime = phases["interactive"] ?: return
        val totalTimeMs = (interactiveTime - startTimeNanos) / 1_000_000.0
        val phaseDurations = calculatePhaseDurations()

        // Metric
        Observability.recordMetric(
            name = "screen_load.duration",
            value = totalTimeMs,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf(
                "screen" to screenName,
                "bucket" to durationBucket(totalTimeMs)
            )
        )

        // Phase breakdown
        phaseDurations.forEach { (phase, duration) ->
            Observability.recordMetric(
                name = "screen_load.phase",
                value = duration,
                unit = MetricUnit.MILLISECONDS,
                tags = mapOf("screen" to screenName, "phase" to phase)
            )
        }

        // Transaction
        val span = Tracer.startChild(
            operation = "ui.load",
            description = screenName
        )
        Tracer.setData("duration_ms", totalTimeMs, span)
        Tracer.setTag("screen", screenName, span)
        phaseDurations.forEach { (phase, duration) ->
            Tracer.setData("phase.${phase}_ms", duration, span)
        }
        Tracer.finishSpan(span)

        // Alert
        val threshold = ScreenLoadThresholds.threshold(screenName)
        if (totalTimeMs > threshold) {
            Observability.captureMessage(
                message = "Slow screen load: $screenName",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "screen" to screenName,
                    "duration_ms" to totalTimeMs,
                    "threshold_ms" to threshold,
                    "phases" to phaseDurations,
                    "metadata" to metadata
                )
            )
        }
    }

    private fun calculatePhaseDurations(): Map<String, Double> {
        val orderedPhases = listOf("view_created", "view_resumed", "data_ready", "fcp", "interactive")
        val durations = mutableMapOf<String, Double>()

        var lastTime = startTimeNanos
        orderedPhases.forEach { phase ->
            phases[phase]?.let { phaseTime ->
                durations[phase] = (phaseTime - lastTime) / 1_000_000.0
                lastTime = phaseTime
            }
        }

        return durations
    }

    private fun durationBucket(ms: Double): String = when {
        ms < 200 -> "fast"
        ms < 400 -> "good"
        ms < 800 -> "acceptable"
        ms < 1500 -> "slow"
        else -> "very_slow"
    }
}

object ScreenLoadThresholds {
    fun threshold(screen: String): Double = when (screen) {
        "HomeScreen" -> 1000.0
        "ProductDetail" -> 800.0
        "Checkout" -> 1200.0
        "Search" -> 600.0
        else -> 1000.0
    }
}

// Compose Integration
@Composable
fun <T> TrackedScreen(
    screenName: String,
    state: UiState<T>,
    onDataReady: (T) -> Unit = {},
    content: @Composable (T) -> Unit
) {
    val tracker = remember { ScreenLoadTracker(screenName) }
    var hasReportedFcp by remember { mutableStateOf(false) }
    var hasReportedInteractive by remember { mutableStateOf(false) }

    LaunchedEffect(Unit) {
        tracker.markViewCreated()
    }

    when (state) {
        is UiState.Loading -> {
            LoadingIndicator()
        }
        is UiState.Success -> {
            if (!hasReportedFcp) {
                hasReportedFcp = true
                tracker.markDataReady()
            }

            Box(modifier = Modifier.drawWithContent {
                drawContent()
                if (!hasReportedInteractive) {
                    hasReportedInteractive = true
                    tracker.markFirstContentfulPaint()
                    tracker.markInteractive()
                }
            }) {
                content(state.data)
            }
        }
        is UiState.Error -> {
            ErrorView(state.message)
        }
    }
}
```

---

## Network Performance

### HTTP Request Instrumentation

```swift
// iOS Network Instrumentation
final class InstrumentedURLSession {
    static let shared = InstrumentedURLSession()

    private let session: URLSession

    init() {
        let config = URLSessionConfiguration.default
        self.session = URLSession(configuration: config, delegate: NetworkDelegate(), delegateQueue: nil)
    }

    func request<T: Decodable>(
        _ request: URLRequest,
        decodeTo type: T.Type
    ) async throws -> T {
        let spanContext = Tracer.shared.startChild(
            operation: "http.client",
            description: "\(request.httpMethod ?? "GET") \(request.url?.path ?? "")"
        )

        let startTime = CFAbsoluteTimeGetCurrent()

        // Add trace headers
        var tracedRequest = request
        tracedRequest.setValue(spanContext.traceId, forHTTPHeaderField: "X-Trace-Id")
        tracedRequest.setValue(spanContext.spanId, forHTTPHeaderField: "X-Span-Id")

        do {
            let (data, response) = try await session.data(for: tracedRequest)
            let duration = (CFAbsoluteTimeGetCurrent() - startTime) * 1000

            guard let httpResponse = response as? HTTPURLResponse else {
                throw NetworkError.invalidResponse
            }

            // Record metrics
            recordNetworkMetrics(
                request: tracedRequest,
                response: httpResponse,
                dataSize: data.count,
                duration: duration,
                error: nil,
                spanContext: spanContext
            )

            // Decode
            let decoded = try JSONDecoder().decode(type, from: data)

            Tracer.shared.finishSpan(spanContext, status: .ok)
            return decoded

        } catch {
            let duration = (CFAbsoluteTimeGetCurrent() - startTime) * 1000

            recordNetworkMetrics(
                request: tracedRequest,
                response: nil,
                dataSize: 0,
                duration: duration,
                error: error,
                spanContext: spanContext
            )

            Tracer.shared.setData("error", value: error.localizedDescription, on: spanContext)
            Tracer.shared.finishSpan(spanContext, status: .error)
            throw error
        }
    }

    private func recordNetworkMetrics(
        request: URLRequest,
        response: HTTPURLResponse?,
        dataSize: Int,
        duration: Double,
        error: Error?,
        spanContext: SpanContext
    ) {
        let url = request.url
        let method = request.httpMethod ?? "GET"
        let statusCode = response?.statusCode ?? 0
        let host = url?.host ?? "unknown"
        let path = url?.path ?? "/"

        // Span data
        Tracer.shared.setData("http.method", value: method, on: spanContext)
        Tracer.shared.setData("http.url", value: url?.absoluteString ?? "", on: spanContext)
        Tracer.shared.setData("http.status_code", value: statusCode, on: spanContext)
        Tracer.shared.setData("http.response_content_length", value: dataSize, on: spanContext)
        Tracer.shared.setData("duration_ms", value: duration, on: spanContext)

        // Metrics
        Observability.recordMetric(
            name: "http.request.duration",
            value: duration,
            unit: .milliseconds,
            tags: [
                "method": method,
                "host": host,
                "status_code": String(statusCode),
                "bucket": durationBucket(duration)
            ]
        )

        Observability.recordMetric(
            name: "http.request.size",
            value: Double(dataSize),
            unit: .bytes,
            tags: ["host": host, "method": method]
        )

        // Breadcrumb
        BreadcrumbManager.shared.networkRequest(
            method: method,
            url: path,
            statusCode: statusCode,
            duration: duration / 1000
        )

        // Error tracking
        if let error = error {
            Observability.captureError(error, context: [
                "url": url?.absoluteString ?? "",
                "method": method,
                "duration_ms": duration
            ])
        } else if statusCode >= 400 {
            Observability.captureMessage(
                "HTTP error response",
                level: statusCode >= 500 ? .error : .warning,
                extras: [
                    "url": url?.absoluteString ?? "",
                    "method": method,
                    "status_code": statusCode,
                    "duration_ms": duration
                ]
            )
        }
    }

    private func durationBucket(_ ms: Double) -> String {
        switch ms {
        case ..<100: return "fast"
        case ..<300: return "good"
        case ..<1000: return "acceptable"
        case ..<3000: return "slow"
        default: return "very_slow"
        }
    }
}
```

```kotlin
// Android Network Instrumentation with OkHttp
class ObservabilityInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()

        val spanContext = Tracer.startChild(
            operation = "http.client",
            description = "${request.method} ${request.url.encodedPath}"
        )

        // Add trace headers
        val tracedRequest = request.newBuilder()
            .header("X-Trace-Id", spanContext.traceId)
            .header("X-Span-Id", spanContext.spanId)
            .build()

        val startTime = System.nanoTime()

        return try {
            val response = chain.proceed(tracedRequest)
            val duration = (System.nanoTime() - startTime) / 1_000_000.0

            recordNetworkMetrics(
                request = tracedRequest,
                response = response,
                duration = duration,
                error = null,
                spanContext = spanContext
            )

            Tracer.finishSpan(spanContext, Tracer.Span.SpanStatus.OK)
            response

        } catch (e: Exception) {
            val duration = (System.nanoTime() - startTime) / 1_000_000.0

            recordNetworkMetrics(
                request = tracedRequest,
                response = null,
                duration = duration,
                error = e,
                spanContext = spanContext
            )

            Tracer.setData("error", e.message, spanContext)
            Tracer.finishSpan(spanContext, Tracer.Span.SpanStatus.ERROR)
            throw e
        }
    }

    private fun recordNetworkMetrics(
        request: Request,
        response: Response?,
        duration: Double,
        error: Exception?,
        spanContext: SpanContext
    ) {
        val method = request.method
        val url = request.url
        val statusCode = response?.code ?: 0
        val responseSize = response?.body?.contentLength() ?: 0

        // Span data
        Tracer.setData("http.method", method, spanContext)
        Tracer.setData("http.url", url.toString(), spanContext)
        Tracer.setData("http.status_code", statusCode, spanContext)
        Tracer.setData("http.response_content_length", responseSize, spanContext)
        Tracer.setData("duration_ms", duration, spanContext)

        // Metrics
        Observability.recordMetric(
            name = "http.request.duration",
            value = duration,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf(
                "method" to method,
                "host" to url.host,
                "status_code" to statusCode.toString(),
                "bucket" to durationBucket(duration)
            )
        )

        // Breadcrumb
        BreadcrumbManager.networkRequest(
            method = method,
            url = url.encodedPath,
            statusCode = statusCode,
            durationMs = duration.toLong()
        )

        // Error tracking
        if (error != null) {
            Observability.captureError(error, mapOf(
                "url" to url.toString(),
                "method" to method,
                "duration_ms" to duration
            ))
        } else if (statusCode >= 400) {
            Observability.captureMessage(
                message = "HTTP error response",
                level = if (statusCode >= 500) SentryLevel.ERROR else SentryLevel.WARNING,
                extras = mapOf(
                    "url" to url.toString(),
                    "method" to method,
                    "status_code" to statusCode,
                    "duration_ms" to duration
                )
            )
        }
    }

    private fun durationBucket(ms: Double): String = when {
        ms < 100 -> "fast"
        ms < 300 -> "good"
        ms < 1000 -> "acceptable"
        ms < 3000 -> "slow"
        else -> "very_slow"
    }
}

// OkHttp setup
val client = OkHttpClient.Builder()
    .addInterceptor(ObservabilityInterceptor())
    .build()
```

---

## Resource Monitoring

### Memory Monitoring

```swift
// iOS Memory Monitor
final class MemoryMonitor {
    static let shared = MemoryMonitor()

    private var timer: Timer?

    func startMonitoring(interval: TimeInterval = 30) {
        timer = Timer.scheduledTimer(withTimeInterval: interval, repeats: true) { [weak self] _ in
            self?.recordMemoryMetrics()
        }
    }

    func stopMonitoring() {
        timer?.invalidate()
        timer = nil
    }

    func recordMemoryMetrics() {
        var info = mach_task_basic_info()
        var count = mach_msg_type_number_t(MemoryLayout<mach_task_basic_info>.size) / 4

        let result = withUnsafeMutablePointer(to: &info) {
            $0.withMemoryRebound(to: integer_t.self, capacity: 1) {
                task_info(mach_task_self_, task_flavor_t(MACH_TASK_BASIC_INFO), $0, &count)
            }
        }

        guard result == KERN_SUCCESS else { return }

        let usedMB = Double(info.resident_size) / (1024 * 1024)
        let totalMB = Double(ProcessInfo.processInfo.physicalMemory) / (1024 * 1024)

        Observability.recordMetric(
            name: "memory.used",
            value: usedMB,
            unit: .megabytes,
            tags: ["type": "resident"]
        )

        Observability.recordMetric(
            name: "memory.available",
            value: totalMB - usedMB,
            unit: .megabytes
        )

        // Alert on high memory
        let usagePercent = (usedMB / totalMB) * 100
        if usagePercent > 80 {
            Observability.captureMessage(
                "High memory usage",
                level: .warning,
                extras: [
                    "used_mb": usedMB,
                    "total_mb": totalMB,
                    "usage_percent": usagePercent
                ]
            )
        }
    }
}
```

```kotlin
// Android Memory Monitor
object MemoryMonitor {
    private var job: Job? = null

    fun startMonitoring(scope: CoroutineScope, intervalMs: Long = 30000) {
        job = scope.launch {
            while (isActive) {
                recordMemoryMetrics()
                delay(intervalMs)
            }
        }
    }

    fun stopMonitoring() {
        job?.cancel()
        job = null
    }

    private fun recordMemoryMetrics() {
        val runtime = Runtime.getRuntime()
        val usedMB = (runtime.totalMemory() - runtime.freeMemory()) / (1024 * 1024).toDouble()
        val maxMB = runtime.maxMemory() / (1024 * 1024).toDouble()

        Observability.recordMetric(
            name = "memory.used",
            value = usedMB,
            unit = MetricUnit.MEGABYTES,
            tags = mapOf("type" to "heap")
        )

        Observability.recordMetric(
            name = "memory.max",
            value = maxMB,
            unit = MetricUnit.MEGABYTES
        )

        // Native memory (if using NDK)
        val nativeHeap = Debug.getNativeHeapAllocatedSize() / (1024 * 1024).toDouble()
        Observability.recordMetric(
            name = "memory.used",
            value = nativeHeap,
            unit = MetricUnit.MEGABYTES,
            tags = mapOf("type" to "native")
        )

        // Alert on high memory
        val usagePercent = (usedMB / maxMB) * 100
        if (usagePercent > 80) {
            Observability.captureMessage(
                message = "High memory usage",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "used_mb" to usedMB,
                    "max_mb" to maxMB,
                    "usage_percent" to usagePercent
                )
            )
        }
    }
}
```

---

## Background Performance

### Background Task Monitoring

```swift
// iOS Background Task Tracker
final class BackgroundTaskTracker {
    static let shared = BackgroundTaskTracker()

    private var activeTasks: [UIBackgroundTaskIdentifier: TaskInfo] = [:]

    struct TaskInfo {
        let name: String
        let startTime: CFAbsoluteTime
        let spanContext: SpanContext
    }

    func beginTask(name: String) -> UIBackgroundTaskIdentifier {
        let spanContext = Tracer.shared.startChild(
            operation: "task.background",
            description: name
        )

        var taskId: UIBackgroundTaskIdentifier = .invalid

        taskId = UIApplication.shared.beginBackgroundTask(withName: name) { [weak self] in
            self?.expireTask(taskId, name: name)
        }

        if taskId != .invalid {
            activeTasks[taskId] = TaskInfo(
                name: name,
                startTime: CFAbsoluteTimeGetCurrent(),
                spanContext: spanContext
            )

            BreadcrumbManager.shared.systemEvent("Background task started: \(name)")
        }

        return taskId
    }

    func endTask(_ taskId: UIBackgroundTaskIdentifier) {
        guard let info = activeTasks.removeValue(forKey: taskId) else { return }

        let duration = (CFAbsoluteTimeGetCurrent() - info.startTime) * 1000

        Observability.recordMetric(
            name: "background_task.duration",
            value: duration,
            unit: .milliseconds,
            tags: ["task": info.name, "status": "completed"]
        )

        Tracer.shared.setData("duration_ms", value: duration, on: info.spanContext)
        Tracer.shared.finishSpan(info.spanContext, status: .ok)

        BreadcrumbManager.shared.systemEvent("Background task completed: \(info.name)")

        UIApplication.shared.endBackgroundTask(taskId)
    }

    private func expireTask(_ taskId: UIBackgroundTaskIdentifier, name: String) {
        guard let info = activeTasks.removeValue(forKey: taskId) else { return }

        let duration = (CFAbsoluteTimeGetCurrent() - info.startTime) * 1000

        Observability.recordMetric(
            name: "background_task.duration",
            value: duration,
            unit: .milliseconds,
            tags: ["task": info.name, "status": "expired"]
        )

        Observability.captureMessage(
            "Background task expired",
            level: .warning,
            extras: [
                "task": info.name,
                "duration_ms": duration
            ]
        )

        Tracer.shared.setData("expired", value: true, on: info.spanContext)
        Tracer.shared.finishSpan(info.spanContext, status: .cancelled)

        UIApplication.shared.endBackgroundTask(taskId)
    }
}
```

---

## Performance Budgets

### Budget Definition

| Metric | Budget | Measurement |
|--------|--------|-------------|
| Cold start | <2s P95 | Process start → interactive |
| Warm start | <1s P95 | Background → interactive |
| Screen load | <400ms P95 | Tap → interactive |
| API latency | <500ms P95 | Request → response |
| Frame rate | 60fps | No dropped frames during scroll |
| Memory | <200MB | Resident memory |
| Bundle size | <50MB | Download size |

### Budget Monitoring

```swift
// PerformanceBudget.swift
struct PerformanceBudget {
    static let budgets: [String: Double] = [
        "app_start.cold": 2000,
        "app_start.warm": 1000,
        "screen_load": 400,
        "api.latency": 500,
        "memory.used": 200
    ]

    static func check(metric: String, value: Double) -> BudgetStatus {
        guard let budget = budgets[metric] else {
            return .undefined
        }

        let ratio = value / budget

        switch ratio {
        case ..<0.75:
            return .good(usedPercent: ratio * 100)
        case ..<1.0:
            return .warning(usedPercent: ratio * 100)
        default:
            return .exceeded(usedPercent: ratio * 100, exceededBy: value - budget)
        }
    }

    enum BudgetStatus {
        case good(usedPercent: Double)
        case warning(usedPercent: Double)
        case exceeded(usedPercent: Double, exceededBy: Double)
        case undefined
    }
}

// Usage in CI
func checkPerformanceBudgets() -> Bool {
    let results = [
        ("app_start.cold", getP95Metric("app_start.cold")),
        ("screen_load", getP95Metric("screen_load")),
        ("api.latency", getP95Metric("api.latency"))
    ]

    var allPassed = true

    for (metric, value) in results {
        let status = PerformanceBudget.check(metric: metric, value: value)

        switch status {
        case .exceeded(let percent, let exceeded):
            print("❌ \(metric): \(value)ms exceeds budget by \(exceeded)ms (\(percent)%)")
            allPassed = false
        case .warning(let percent):
            print("⚠️ \(metric): \(value)ms approaching budget (\(percent)%)")
        case .good(let percent):
            print("✅ \(metric): \(value)ms within budget (\(percent)%)")
        case .undefined:
            print("? \(metric): no budget defined")
        }
    }

    return allPassed
}
```

---

## Summary

| Component | Key Metrics | Target |
|-----------|-------------|--------|
| **App Start** | Cold/warm/hot duration | <2s/<1s/<500ms |
| **Screen Load** | Tap to interactive | <400ms P95 |
| **Network** | Request duration, error rate | <500ms P95, <1% errors |
| **Memory** | Resident/heap usage | <200MB |
| **Background** | Task completion rate | >99% |

---

## Integration Points

- **[ui-performance.md](ui-performance.md)** - Navigation and render metrics
- **[observability-fundamentals.md](observability-fundamentals.md)** - Span and metric fundamentals
- **[mobile-challenges.md](mobile-challenges.md)** - Battery-aware performance monitoring
- **[alert-thresholds.md](alert-thresholds.md)** - Performance alerting configuration

## Reusable Templates

- **[templates/screen-load-tracking.template.md](templates/screen-load-tracking.template.md)** - Copy-paste screen load tracking implementation
- **[templates/navigation-tracking.template.md](templates/navigation-tracking.template.md)** - Navigation latency measurement patterns
