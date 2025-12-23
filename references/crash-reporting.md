# Crash Reporting & Error Tracking

Comprehensive guide to crash reporting, error tracking, ANR detection, and breadcrumb collection for mobile apps.

## Table of Contents
1. [Error Types](#error-types)
2. [Crash Handling](#crash-handling)
3. [ANR Detection](#anr-detection)
4. [Breadcrumbs](#breadcrumbs)
5. [Error Context](#error-context)
6. [Symbolication](#symbolication)
7. [Grouping & Deduplication](#grouping--deduplication)
8. [Alerting Strategy](#alerting-strategy)

---

## Error Types

### Classification

| Type | Description | Severity | Example |
|------|-------------|----------|---------|
| **Crash** | App terminated by OS | Critical | Null pointer, OOM |
| **ANR** | App Not Responding | High | Main thread blocked >5s |
| **Handled Exception** | Caught and recovered | Medium | API error, validation |
| **Unhandled Exception** | Uncaught but non-fatal | High | Background task failure |
| **Assertion Failure** | Logic error detected | High | Invariant violated |
| **Watchdog** | System killed app | Critical | Background timeout |

### iOS Error Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                     iOS Error Types                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  CRASHES (App Terminated)                                    │
│  ├── EXC_BAD_ACCESS (SIGSEGV, SIGBUS)                       │
│  │   └── Memory access violation                             │
│  ├── EXC_CRASH (SIGABRT)                                    │
│  │   └── Assertion, fatalError, precondition                │
│  ├── EXC_RESOURCE                                           │
│  │   └── Memory/CPU limit exceeded                          │
│  └── Watchdog Termination                                   │
│      └── Background task timeout                             │
│                                                              │
│  NON-FATAL                                                   │
│  ├── Swift Error (thrown)                                   │
│  ├── NSException                                            │
│  └── Assertion (non-fatal)                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Android Error Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                    Android Error Types                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  CRASHES (App Terminated)                                    │
│  ├── Java/Kotlin Exception (uncaught)                       │
│  │   ├── NullPointerException                               │
│  │   ├── IllegalStateException                              │
│  │   └── OutOfMemoryError                                   │
│  ├── Native Crash (SIGSEGV, SIGABRT)                        │
│  │   └── NDK/JNI code failures                              │
│  └── ANR (Application Not Responding)                       │
│      └── Main thread blocked >5s                             │
│                                                              │
│  NON-FATAL                                                   │
│  ├── Handled Exception                                      │
│  ├── Coroutine cancellation                                 │
│  └── Custom error events                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Crash Handling

### iOS Crash Handler

```swift
// CrashHandler.swift
import Foundation
import MachExceptionHandler // Conceptual - real implementation uses signal handlers

final class CrashHandler {
    static let shared = CrashHandler()

    private var previousHandler: (@convention(c) (NSException) -> Void)?
    private var signalHandlersInstalled = false

    func install() {
        installUncaughtExceptionHandler()
        installSignalHandlers()
    }

    // MARK: - NSException Handler

    private func installUncaughtExceptionHandler() {
        previousHandler = NSGetUncaughtExceptionHandler()

        NSSetUncaughtExceptionHandler { exception in
            CrashHandler.shared.handleException(exception)

            // Forward to previous handler
            CrashHandler.shared.previousHandler?(exception)
        }
    }

    private func handleException(_ exception: NSException) {
        let crash = CrashReport(
            type: .nsException,
            name: exception.name.rawValue,
            reason: exception.reason ?? "Unknown",
            stackTrace: exception.callStackSymbols,
            context: CrashContext.current()
        )

        persistCrash(crash)
    }

    // MARK: - Signal Handlers

    private func installSignalHandlers() {
        guard !signalHandlersInstalled else { return }
        signalHandlersInstalled = true

        let signals: [Int32] = [SIGABRT, SIGBUS, SIGFPE, SIGILL, SIGSEGV, SIGTRAP]

        for sig in signals {
            signal(sig) { signal in
                CrashHandler.shared.handleSignal(signal)
            }
        }
    }

    private func handleSignal(_ signal: Int32) {
        let crash = CrashReport(
            type: .signal,
            name: signalName(signal),
            reason: "Signal \(signal) received",
            stackTrace: Thread.callStackSymbols,
            context: CrashContext.current()
        )

        persistCrash(crash)

        // Re-raise signal to let system handle it
        self.signalHandlersInstalled = false
        raise(signal)
    }

    private func signalName(_ signal: Int32) -> String {
        switch signal {
        case SIGABRT: return "SIGABRT"
        case SIGBUS: return "SIGBUS"
        case SIGFPE: return "SIGFPE"
        case SIGILL: return "SIGILL"
        case SIGSEGV: return "SIGSEGV"
        case SIGTRAP: return "SIGTRAP"
        default: return "SIGNAL_\(signal)"
        }
    }

    // MARK: - Persistence

    private func persistCrash(_ crash: CrashReport) {
        // Write to disk synchronously - minimal operations allowed
        let data = try? JSONEncoder().encode(crash)
        let url = crashFileURL()
        try? data?.write(to: url, options: .atomic)
    }

    private func crashFileURL() -> URL {
        let cacheDir = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask).first!
        return cacheDir.appendingPathComponent("pending_crash_\(Date().timeIntervalSince1970).json")
    }

    // MARK: - Upload on Next Launch

    func uploadPendingCrashes() {
        let cacheDir = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask).first!
        let crashFiles = try? FileManager.default.contentsOfDirectory(at: cacheDir, includingPropertiesForKeys: nil)
            .filter { $0.lastPathComponent.hasPrefix("pending_crash_") }

        for file in crashFiles ?? [] {
            if let data = try? Data(contentsOf: file),
               let crash = try? JSONDecoder().decode(CrashReport.self, from: data) {
                uploadCrash(crash) { success in
                    if success {
                        try? FileManager.default.removeItem(at: file)
                    }
                }
            }
        }
    }

    private func uploadCrash(_ crash: CrashReport, completion: @escaping (Bool) -> Void) {
        // Send to observability backend
    }
}

struct CrashReport: Codable {
    enum CrashType: String, Codable {
        case nsException, signal, watchdog
    }

    let id: String = UUID().uuidString
    let timestamp: Date = Date()
    let type: CrashType
    let name: String
    let reason: String
    let stackTrace: [String]
    let context: CrashContext
}

struct CrashContext: Codable {
    let sessionId: String
    let userId: String?
    let screenName: String
    let appVersion: String
    let osVersion: String
    let deviceModel: String
    let freeMemoryMB: Int
    let freeDiskMB: Int
    let batteryLevel: Float
    let isLowPowerMode: Bool
    let breadcrumbs: [Breadcrumb]

    static func current() -> CrashContext {
        CrashContext(
            sessionId: SessionManager.shared.sessionId,
            userId: UserManager.shared.userId,
            screenName: NavigationState.shared.currentScreen,
            appVersion: Bundle.main.appVersion,
            osVersion: UIDevice.current.systemVersion,
            deviceModel: DeviceInfo.modelName,
            freeMemoryMB: DeviceInfo.freeMemoryMB,
            freeDiskMB: DeviceInfo.freeDiskMB,
            batteryLevel: UIDevice.current.batteryLevel,
            isLowPowerMode: ProcessInfo.processInfo.isLowPowerModeEnabled,
            breadcrumbs: BreadcrumbManager.shared.getBreadcrumbs()
        )
    }
}
```

### Android Crash Handler

```kotlin
// CrashHandler.kt
import android.os.Process
import java.io.File
import java.io.PrintWriter
import java.io.StringWriter

object CrashHandler : Thread.UncaughtExceptionHandler {
    private var defaultHandler: Thread.UncaughtExceptionHandler? = null
    private lateinit var context: android.content.Context

    fun install(context: android.content.Context) {
        this.context = context.applicationContext
        defaultHandler = Thread.getDefaultUncaughtExceptionHandler()
        Thread.setDefaultUncaughtExceptionHandler(this)
    }

    override fun uncaughtException(thread: Thread, throwable: Throwable) {
        val crash = CrashReport(
            type = CrashType.JAVA_EXCEPTION,
            name = throwable.javaClass.simpleName,
            reason = throwable.message ?: "Unknown",
            stackTrace = getStackTrace(throwable),
            threadName = thread.name,
            context = CrashContext.current()
        )

        persistCrash(crash)

        // Forward to default handler (will terminate app)
        defaultHandler?.uncaughtException(thread, throwable)
    }

    private fun getStackTrace(throwable: Throwable): String {
        val sw = StringWriter()
        throwable.printStackTrace(PrintWriter(sw))
        return sw.toString()
    }

    private fun persistCrash(crash: CrashReport) {
        try {
            val file = File(context.cacheDir, "pending_crash_${System.currentTimeMillis()}.json")
            file.writeText(crash.toJson())
        } catch (e: Exception) {
            // Minimal error handling during crash
        }
    }

    fun uploadPendingCrashes() {
        context.cacheDir.listFiles()
            ?.filter { it.name.startsWith("pending_crash_") }
            ?.forEach { file ->
                try {
                    val crash = CrashReport.fromJson(file.readText())
                    uploadCrash(crash) { success ->
                        if (success) file.delete()
                    }
                } catch (e: Exception) {
                    file.delete() // Remove corrupted files
                }
            }
    }

    private fun uploadCrash(crash: CrashReport, callback: (Boolean) -> Unit) {
        // Send to observability backend
    }
}

data class CrashReport(
    val id: String = java.util.UUID.randomUUID().toString(),
    val timestamp: Long = System.currentTimeMillis(),
    val type: CrashType,
    val name: String,
    val reason: String,
    val stackTrace: String,
    val threadName: String,
    val context: CrashContext
) {
    enum class CrashType { JAVA_EXCEPTION, NATIVE_CRASH, ANR }

    fun toJson(): String = /* JSON serialization */
    companion object {
        fun fromJson(json: String): CrashReport = /* JSON deserialization */
    }
}

data class CrashContext(
    val sessionId: String,
    val userId: String?,
    val screenName: String,
    val appVersion: String,
    val osVersion: String,
    val deviceModel: String,
    val freeMemoryMB: Long,
    val freeDiskMB: Long,
    val batteryLevel: Float,
    val isLowPowerMode: Boolean,
    val breadcrumbs: List<Breadcrumb>
) {
    companion object {
        fun current() = CrashContext(
            sessionId = SessionManager.sessionId,
            userId = UserManager.userId,
            screenName = NavigationState.currentScreen,
            appVersion = BuildConfig.VERSION_NAME,
            osVersion = Build.VERSION.RELEASE,
            deviceModel = Build.MODEL,
            freeMemoryMB = Runtime.getRuntime().freeMemory() / (1024 * 1024),
            freeDiskMB = Environment.getDataDirectory().freeSpace / (1024 * 1024),
            batteryLevel = getBatteryLevel(),
            isLowPowerMode = isPowerSaveMode(),
            breadcrumbs = BreadcrumbManager.getBreadcrumbs()
        )
    }
}
```

### Handled Error Capture

```swift
// iOS - Error Capture
func captureError(
    _ error: Error,
    level: ErrorLevel = .error,
    context: [String: Any] = [:]
) {
    let errorEvent = ErrorEvent(
        id: UUID().uuidString,
        timestamp: Date(),
        level: level,
        type: String(describing: type(of: error)),
        message: error.localizedDescription,
        stackTrace: Thread.callStackSymbols,
        context: context.merging(CorrelationContext.current.asDictionary) { $1 },
        breadcrumbs: BreadcrumbManager.shared.getBreadcrumbs()
    )

    ErrorReporter.shared.report(errorEvent)
}

enum ErrorLevel: String, Codable {
    case debug, info, warning, error, fatal
}

// Usage
do {
    try riskyOperation()
} catch {
    captureError(error, context: [
        "operation": "riskyOperation",
        "input_size": inputData.count
    ])
}
```

```kotlin
// Android - Error Capture
fun captureError(
    error: Throwable,
    level: ErrorLevel = ErrorLevel.ERROR,
    context: Map<String, Any> = emptyMap()
) {
    val errorEvent = ErrorEvent(
        id = UUID.randomUUID().toString(),
        timestamp = System.currentTimeMillis(),
        level = level,
        type = error.javaClass.simpleName,
        message = error.message ?: "Unknown error",
        stackTrace = error.stackTraceToString(),
        context = context + CorrelationContext.current.asMap(),
        breadcrumbs = BreadcrumbManager.getBreadcrumbs()
    )

    ErrorReporter.report(errorEvent)
}

enum class ErrorLevel { DEBUG, INFO, WARNING, ERROR, FATAL }

// Usage
try {
    riskyOperation()
} catch (e: Exception) {
    captureError(e, context = mapOf(
        "operation" to "riskyOperation",
        "input_size" to inputData.size
    ))
}
```

---

## ANR Detection

### iOS Hang Detection

iOS doesn't have a formal ANR concept, but you can detect main thread hangs.

```swift
// HangDetector.swift
import Foundation

final class HangDetector {
    static let shared = HangDetector()

    private let watchdogQueue = DispatchQueue(label: "hang.detector", qos: .userInteractive)
    private var watchdogTimer: DispatchSourceTimer?
    private var lastPingTime: CFAbsoluteTime = 0
    private let hangThreshold: TimeInterval = 2.0 // seconds

    func start() {
        // Periodically check main thread responsiveness
        watchdogTimer = DispatchSource.makeTimerSource(queue: watchdogQueue)
        watchdogTimer?.schedule(deadline: .now(), repeating: 0.5)

        watchdogTimer?.setEventHandler { [weak self] in
            self?.checkMainThread()
        }

        watchdogTimer?.resume()

        // Ping from main thread
        pingMainThread()
    }

    func stop() {
        watchdogTimer?.cancel()
        watchdogTimer = nil
    }

    private func pingMainThread() {
        DispatchQueue.main.async { [weak self] in
            self?.lastPingTime = CFAbsoluteTimeGetCurrent()

            // Schedule next ping
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
                self?.pingMainThread()
            }
        }
    }

    private func checkMainThread() {
        let timeSinceLastPing = CFAbsoluteTimeGetCurrent() - lastPingTime

        if timeSinceLastPing > hangThreshold {
            reportHang(duration: timeSinceLastPing)
        }
    }

    private func reportHang(duration: TimeInterval) {
        // Capture main thread stack
        let mainThreadStack = captureMainThreadStack()

        let hang = HangEvent(
            duration: duration,
            mainThreadStack: mainThreadStack,
            context: CrashContext.current()
        )

        // Report hang
        Observability.captureMessage(
            "UI hang detected",
            level: .warning,
            extras: [
                "duration_seconds": duration,
                "main_thread_stack": mainThreadStack
            ]
        )
    }

    private func captureMainThreadStack() -> [String] {
        // In production, use a proper stack capture mechanism
        // This is a simplified version
        return Thread.callStackSymbols
    }
}

struct HangEvent: Codable {
    let id: String = UUID().uuidString
    let timestamp: Date = Date()
    let duration: TimeInterval
    let mainThreadStack: [String]
    let context: CrashContext
}
```

### Android ANR Detection

```kotlin
// ANRWatchdog.kt
import android.os.Handler
import android.os.Looper
import java.util.concurrent.Executors
import java.util.concurrent.TimeUnit
import java.util.concurrent.atomic.AtomicBoolean

class ANRWatchdog(
    private val threshold: Long = 5000L // 5 seconds, same as Android system
) {
    private val executor = Executors.newSingleThreadScheduledExecutor()
    private val mainHandler = Handler(Looper.getMainLooper())
    private val responded = AtomicBoolean(true)
    private var lastResponseTime = System.currentTimeMillis()

    fun start() {
        executor.scheduleAtFixedRate({
            checkMainThread()
        }, threshold, threshold / 2, TimeUnit.MILLISECONDS)

        pingMainThread()
    }

    fun stop() {
        executor.shutdown()
    }

    private fun pingMainThread() {
        mainHandler.post {
            responded.set(true)
            lastResponseTime = System.currentTimeMillis()

            // Schedule next ping
            mainHandler.postDelayed({ pingMainThread() }, 500)
        }
    }

    private fun checkMainThread() {
        if (!responded.getAndSet(false)) {
            val blockTime = System.currentTimeMillis() - lastResponseTime

            if (blockTime >= threshold) {
                reportANR(blockTime)
            }
        }
    }

    private fun reportANR(blockTimeMs: Long) {
        val mainThreadStack = Looper.getMainLooper().thread.stackTrace
            .map { it.toString() }

        val anr = ANREvent(
            blockTimeMs = blockTimeMs,
            mainThreadStack = mainThreadStack,
            allThreads = captureAllThreads(),
            context = CrashContext.current()
        )

        Observability.captureMessage(
            message = "ANR detected",
            level = SentryLevel.ERROR,
            extras = mapOf(
                "block_time_ms" to blockTimeMs,
                "main_thread_stack" to mainThreadStack.joinToString("\n")
            )
        )
    }

    private fun captureAllThreads(): Map<String, List<String>> {
        return Thread.getAllStackTraces().map { (thread, stack) ->
            thread.name to stack.map { it.toString() }
        }.toMap()
    }
}

data class ANREvent(
    val id: String = java.util.UUID.randomUUID().toString(),
    val timestamp: Long = System.currentTimeMillis(),
    val blockTimeMs: Long,
    val mainThreadStack: List<String>,
    val allThreads: Map<String, List<String>>,
    val context: CrashContext
)
```

---

## Breadcrumbs

### Breadcrumb Types

| Type | Description | Auto-collected? |
|------|-------------|-----------------|
| **Navigation** | Screen transitions | Yes |
| **UI** | Button taps, gestures | Configurable |
| **Network** | HTTP requests/responses | Yes |
| **Console** | Log messages | Configurable |
| **User** | Custom user actions | Manual |
| **System** | App lifecycle, memory warnings | Yes |
| **Error** | Non-fatal errors | Yes |

### iOS Breadcrumb Manager

```swift
// BreadcrumbManager.swift
import Foundation

struct Breadcrumb: Codable {
    enum BreadcrumbType: String, Codable {
        case navigation, ui, network, console, user, system, error
    }

    enum Level: String, Codable {
        case debug, info, warning, error
    }

    let timestamp: Date
    let type: BreadcrumbType
    let level: Level
    let category: String
    let message: String
    let data: [String: String]
}

final class BreadcrumbManager {
    static let shared = BreadcrumbManager()

    private var breadcrumbs: [Breadcrumb] = []
    private let queue = DispatchQueue(label: "breadcrumbs")
    private let maxBreadcrumbs = 100

    func add(
        type: Breadcrumb.BreadcrumbType,
        category: String,
        message: String,
        level: Breadcrumb.Level = .info,
        data: [String: String] = [:]
    ) {
        let breadcrumb = Breadcrumb(
            timestamp: Date(),
            type: type,
            level: level,
            category: category,
            message: message,
            data: data
        )

        queue.async {
            self.breadcrumbs.append(breadcrumb)

            // Keep only recent breadcrumbs
            if self.breadcrumbs.count > self.maxBreadcrumbs {
                self.breadcrumbs.removeFirst(self.breadcrumbs.count - self.maxBreadcrumbs)
            }
        }
    }

    func getBreadcrumbs() -> [Breadcrumb] {
        queue.sync { breadcrumbs }
    }

    func clear() {
        queue.async { self.breadcrumbs.removeAll() }
    }

    // MARK: - Convenience Methods

    func navigation(from: String, to: String) {
        add(
            type: .navigation,
            category: "navigation",
            message: "Navigated from \(from) to \(to)",
            data: ["from": from, "to": to]
        )
    }

    func buttonTap(_ buttonName: String, screen: String) {
        add(
            type: .ui,
            category: "ui.tap",
            message: "Tapped \(buttonName)",
            data: ["button": buttonName, "screen": screen]
        )
    }

    func networkRequest(method: String, url: String, statusCode: Int, duration: TimeInterval) {
        let level: Breadcrumb.Level = statusCode >= 400 ? .warning : .info
        add(
            type: .network,
            category: "http",
            message: "\(method) \(url) - \(statusCode)",
            level: level,
            data: [
                "method": method,
                "url": url,
                "status_code": String(statusCode),
                "duration_ms": String(Int(duration * 1000))
            ]
        )
    }

    func systemEvent(_ event: String, data: [String: String] = [:]) {
        add(
            type: .system,
            category: "app.lifecycle",
            message: event,
            data: data
        )
    }

    func userAction(_ action: String, data: [String: String] = [:]) {
        add(
            type: .user,
            category: "user",
            message: action,
            data: data
        )
    }
}

// MARK: - Auto-collection

extension BreadcrumbManager {
    func setupAutoCollection() {
        // App lifecycle
        NotificationCenter.default.addObserver(
            forName: UIApplication.didBecomeActiveNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.systemEvent("App became active")
        }

        NotificationCenter.default.addObserver(
            forName: UIApplication.willResignActiveNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.systemEvent("App will resign active")
        }

        NotificationCenter.default.addObserver(
            forName: UIApplication.didEnterBackgroundNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.systemEvent("App entered background")
        }

        NotificationCenter.default.addObserver(
            forName: UIApplication.didReceiveMemoryWarningNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.add(
                type: .system,
                category: "memory",
                message: "Memory warning received",
                level: .warning
            )
        }
    }
}
```

### Android Breadcrumb Manager

```kotlin
// BreadcrumbManager.kt
import java.util.concurrent.ConcurrentLinkedDeque

data class Breadcrumb(
    val timestamp: Long = System.currentTimeMillis(),
    val type: BreadcrumbType,
    val level: Level,
    val category: String,
    val message: String,
    val data: Map<String, String> = emptyMap()
) {
    enum class BreadcrumbType { NAVIGATION, UI, NETWORK, CONSOLE, USER, SYSTEM, ERROR }
    enum class Level { DEBUG, INFO, WARNING, ERROR }
}

object BreadcrumbManager {
    private val breadcrumbs = ConcurrentLinkedDeque<Breadcrumb>()
    private const val MAX_BREADCRUMBS = 100

    fun add(
        type: Breadcrumb.BreadcrumbType,
        category: String,
        message: String,
        level: Breadcrumb.Level = Breadcrumb.Level.INFO,
        data: Map<String, String> = emptyMap()
    ) {
        breadcrumbs.addLast(Breadcrumb(
            type = type,
            level = level,
            category = category,
            message = message,
            data = data
        ))

        while (breadcrumbs.size > MAX_BREADCRUMBS) {
            breadcrumbs.pollFirst()
        }
    }

    fun getBreadcrumbs(): List<Breadcrumb> = breadcrumbs.toList()

    fun clear() = breadcrumbs.clear()

    // Convenience methods
    fun navigation(from: String, to: String) {
        add(
            type = Breadcrumb.BreadcrumbType.NAVIGATION,
            category = "navigation",
            message = "Navigated from $from to $to",
            data = mapOf("from" to from, "to" to to)
        )
    }

    fun buttonTap(buttonName: String, screen: String) {
        add(
            type = Breadcrumb.BreadcrumbType.UI,
            category = "ui.tap",
            message = "Tapped $buttonName",
            data = mapOf("button" to buttonName, "screen" to screen)
        )
    }

    fun networkRequest(method: String, url: String, statusCode: Int, durationMs: Long) {
        val level = if (statusCode >= 400) Breadcrumb.Level.WARNING else Breadcrumb.Level.INFO
        add(
            type = Breadcrumb.BreadcrumbType.NETWORK,
            category = "http",
            message = "$method $url - $statusCode",
            level = level,
            data = mapOf(
                "method" to method,
                "url" to url,
                "status_code" to statusCode.toString(),
                "duration_ms" to durationMs.toString()
            )
        )
    }

    fun systemEvent(event: String, data: Map<String, String> = emptyMap()) {
        add(
            type = Breadcrumb.BreadcrumbType.SYSTEM,
            category = "app.lifecycle",
            message = event,
            data = data
        )
    }

    fun userAction(action: String, data: Map<String, String> = emptyMap()) {
        add(
            type = Breadcrumb.BreadcrumbType.USER,
            category = "user",
            message = action,
            data = data
        )
    }
}

// Auto-collection via Application.ActivityLifecycleCallbacks
class BreadcrumbLifecycleCallbacks : Application.ActivityLifecycleCallbacks {
    private var currentActivity: String? = null

    override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {
        BreadcrumbManager.systemEvent(
            "Activity created: ${activity.javaClass.simpleName}",
            mapOf("restored" to (savedInstanceState != null).toString())
        )
    }

    override fun onActivityResumed(activity: Activity) {
        val activityName = activity.javaClass.simpleName
        val previousActivity = currentActivity
        currentActivity = activityName

        if (previousActivity != null && previousActivity != activityName) {
            BreadcrumbManager.navigation(previousActivity, activityName)
        }
    }

    override fun onActivityPaused(activity: Activity) {}
    override fun onActivityStarted(activity: Activity) {}
    override fun onActivityStopped(activity: Activity) {}
    override fun onActivitySaveInstanceState(activity: Activity, outState: Bundle) {}
    override fun onActivityDestroyed(activity: Activity) {}
}

// Memory warning via ComponentCallbacks2
class MemoryCallback : ComponentCallbacks2 {
    override fun onTrimMemory(level: Int) {
        val levelName = when (level) {
            ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW -> "RUNNING_LOW"
            ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL -> "RUNNING_CRITICAL"
            ComponentCallbacks2.TRIM_MEMORY_COMPLETE -> "COMPLETE"
            else -> "LEVEL_$level"
        }

        BreadcrumbManager.add(
            type = Breadcrumb.BreadcrumbType.SYSTEM,
            category = "memory",
            message = "Memory trim: $levelName",
            level = Breadcrumb.Level.WARNING,
            data = mapOf("level" to level.toString())
        )
    }

    override fun onConfigurationChanged(newConfig: Configuration) {}
    override fun onLowMemory() {
        BreadcrumbManager.add(
            type = Breadcrumb.BreadcrumbType.SYSTEM,
            category = "memory",
            message = "Low memory warning",
            level = Breadcrumb.Level.ERROR
        )
    }
}
```

---

## Error Context

### Context Enrichment

Always include with errors:

| Context | Why | Example |
|---------|-----|---------|
| **Session ID** | Tie all events together | "abc123" |
| **User ID** | Support tickets, patterns | "user_456" |
| **Screen** | Where was user? | "CheckoutScreen" |
| **App State** | What state was app in? | foreground/background |
| **Memory** | OOM context | 45MB free |
| **Network** | Connectivity issues | wifi/cellular/offline |
| **Feature Flags** | Which code path? | {"newCheckout": true} |
| **A/B Variant** | Experiment correlation | "checkout_v2" |

```swift
// iOS Context Collector
struct ErrorContext {
    // Required
    let sessionId: String
    let screenName: String
    let appVersion: String
    let osVersion: String
    let deviceModel: String
    let timestamp: Date

    // User
    let userId: String?
    let isAuthenticated: Bool

    // App State
    let appState: AppState
    let timeSinceLaunch: TimeInterval
    let screenDuration: TimeInterval

    // Device State
    let freeMemoryMB: Int
    let totalMemoryMB: Int
    let freeDiskMB: Int
    let batteryLevel: Float
    let batteryState: BatteryState
    let isLowPowerMode: Bool
    let thermalState: ThermalState

    // Network
    let networkType: NetworkType
    let isReachable: Bool

    // Features
    let featureFlags: [String: Bool]
    let abVariants: [String: String]

    // Trace
    let traceId: String?
    let spanId: String?

    enum AppState: String, Codable {
        case foreground, background, inactive
    }

    enum BatteryState: String, Codable {
        case unknown, unplugged, charging, full
    }

    enum ThermalState: String, Codable {
        case nominal, fair, serious, critical
    }

    enum NetworkType: String, Codable {
        case wifi, cellular, ethernet, unknown, offline
    }

    static var current: ErrorContext {
        ErrorContext(
            sessionId: SessionManager.shared.sessionId,
            screenName: NavigationState.shared.currentScreen,
            appVersion: Bundle.main.appVersion,
            osVersion: UIDevice.current.systemVersion,
            deviceModel: DeviceInfo.modelName,
            timestamp: Date(),
            userId: UserManager.shared.userId,
            isAuthenticated: UserManager.shared.isAuthenticated,
            appState: UIApplication.shared.applicationState.toAppState,
            timeSinceLaunch: ProcessInfo.processInfo.systemUptime,
            screenDuration: NavigationState.shared.currentScreenDuration,
            freeMemoryMB: DeviceInfo.freeMemoryMB,
            totalMemoryMB: DeviceInfo.totalMemoryMB,
            freeDiskMB: DeviceInfo.freeDiskMB,
            batteryLevel: UIDevice.current.batteryLevel,
            batteryState: UIDevice.current.batteryState.toBatteryState,
            isLowPowerMode: ProcessInfo.processInfo.isLowPowerModeEnabled,
            thermalState: ProcessInfo.processInfo.thermalState.toThermalState,
            networkType: NetworkMonitor.shared.currentType,
            isReachable: NetworkMonitor.shared.isReachable,
            featureFlags: FeatureFlags.shared.allFlags,
            abVariants: ABTestManager.shared.activeVariants,
            traceId: TraceContext.current?.traceId,
            spanId: TraceContext.current?.spanId
        )
    }
}
```

---

## Symbolication

### iOS dSYM Management

```bash
# Find dSYMs after archive
find ~/Library/Developer/Xcode/Archives -name "*.dSYM" -mtime -7

# Upload to Sentry
sentry-cli upload-dif --org your-org --project your-project \
    ~/Library/Developer/Xcode/Archives/2024-01-15/YourApp.xcarchive/dSYMs

# Verify symbolication works
sentry-cli debug-files check <dSYM-UUID>

# Build phase script (add to Xcode)
if [ "${CONFIGURATION}" = "Release" ]; then
    sentry-cli upload-dif --include-sources \
        "${DWARF_DSYM_FOLDER_PATH}"
fi
```

### Android ProGuard/R8 Mapping

```kotlin
// build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
}

// Auto-upload mapping file
sentry {
    autoUploadProguardMapping = true
    uploadNativeSymbols = true
}
```

```bash
# Manual upload
sentry-cli upload-proguard \
    --org your-org \
    --project your-project \
    --android-manifest app/build/intermediates/merged_manifest/release/AndroidManifest.xml \
    app/build/outputs/mapping/release/mapping.txt
```

---

## Grouping & Deduplication

### Fingerprinting Strategy

```swift
// iOS Fingerprinting
extension CrashReport {
    var fingerprint: String {
        // Group by: error type + top 3 app frames
        let components = [
            type.rawValue,
            name,
            topAppFrames(3).joined(separator: "|")
        ]
        return components.joined(separator: "::").md5Hash
    }

    private func topAppFrames(_ count: Int) -> [String] {
        stackTrace
            .filter { $0.contains(Bundle.main.bundleIdentifier ?? "") }
            .prefix(count)
            .map { normalizeFrame($0) }
    }

    private func normalizeFrame(_ frame: String) -> String {
        // Remove memory addresses, keep class/method
        frame.replacingOccurrences(
            of: "0x[0-9a-fA-F]+",
            with: "",
            options: .regularExpression
        )
    }
}
```

```kotlin
// Android Fingerprinting
fun CrashReport.fingerprint(): String {
    val components = listOf(
        type.name,
        name,
        topAppFrames(3).joinToString("|")
    )
    return components.joinToString("::").md5()
}

private fun CrashReport.topAppFrames(count: Int): List<String> {
    return stackTrace.lines()
        .filter { it.contains(BuildConfig.APPLICATION_ID) }
        .take(count)
        .map { normalizeFrame(it) }
}

private fun normalizeFrame(frame: String): String {
    // Remove line numbers for grouping
    return frame.replace(Regex(":\\d+\\)"), ")")
}
```

---

## Alerting Strategy

### Alert Thresholds

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| **Crash-free sessions** | <99.5% | <99% | Page on-call |
| **New crash spike** | +50% | +100% | Investigate |
| **ANR rate** | >0.5% | >1% | Investigate |
| **Error rate** | >5% | >10% | Page on-call |
| **P95 latency** | >2x baseline | >3x baseline | Investigate |

### Alert Configuration

```yaml
# alerts.yaml
alerts:
  - name: crash_rate_critical
    metric: crash_free_sessions
    condition: "< 99.0"
    window: 1h
    severity: critical
    channels: [pagerduty, slack]

  - name: crash_spike
    metric: crashes_count
    condition: "increase > 100%"
    baseline_window: 24h
    severity: warning
    channels: [slack]

  - name: anr_rate_high
    metric: anr_rate
    condition: "> 1.0"
    window: 1h
    severity: warning
    channels: [slack]

  - name: new_error_type
    metric: error_types
    condition: "new_type_seen"
    severity: info
    channels: [slack]
```

---

## Summary

| Component | Key Insight |
|-----------|-------------|
| **Crash Types** | iOS: signals + NSException; Android: Java + native + ANR |
| **Handler** | Install early, persist synchronously, upload on next launch |
| **ANR** | Monitor main thread ping response time |
| **Breadcrumbs** | 50-100 recent events; auto-collect navigation/network/lifecycle |
| **Context** | Every error needs: session, screen, device state, user |
| **Symbolication** | Automate dSYM/mapping upload in CI |
| **Grouping** | Fingerprint = error type + top N app stack frames |
| **Alerting** | Crash-free >99%, ANR <0.5%, alert on spikes |

---

## Integration Points

- **[observability-fundamentals.md](observability-fundamentals.md)** - Error events and correlation context
- **[performance.md](performance.md)** - Slow operations leading to ANRs
- **[mobile-challenges.md](mobile-challenges.md)** - Offline crash storage
- **[platforms/](platforms/)** - Vendor-specific crash reporting setup
- **[metrickit.md](metrickit.md)** - iOS supplementary diagnostics (optional)

## Reusable Templates

- **[templates/error-boundary.template.md](templates/error-boundary.template.md)** - Copy-paste crash handlers and error capture
- **[templates/breadcrumb-manager.template.md](templates/breadcrumb-manager.template.md)** - Breadcrumb collection implementation
