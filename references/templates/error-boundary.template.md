# Error Boundary Template

Reusable patterns for crash handling and error capture across platforms.

## iOS

### CrashHandler

```swift
// CrashHandler.swift
import Foundation

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

### Handled Error Capture

```swift
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

## Android

### CrashHandler

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
                    file.delete()
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

```kotlin
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

## React Native

### Root Error Boundary

```typescript
// components/ErrorBoundary.tsx
import * as Sentry from '@sentry/react-native';
import { ErrorBoundary as SentryErrorBoundary } from '@sentry/react-native';

export function RootErrorBoundary({ children }: { children: React.ReactNode }) {
  return (
    <SentryErrorBoundary
      fallback={({ error, resetError }) => (
        <ErrorFallbackScreen error={error} onRetry={resetError} />
      )}
      onError={(error, componentStack) => {
        Sentry.withScope((scope) => {
          scope.setTag('error.boundary', 'root');
          scope.setContext('component_stack', { stack: componentStack });
        });
      }}
    >
      {children}
    </SentryErrorBoundary>
  );
}
```

### Screen-Level Boundaries

```typescript
export function withScreenErrorBoundary<P extends object>(
  WrappedComponent: React.ComponentType<P>,
  screenName: string
) {
  return function ScreenWithBoundary(props: P) {
    return (
      <SentryErrorBoundary
        fallback={<ScreenErrorFallback screenName={screenName} />}
        onError={(error) => {
          Sentry.withScope((scope) => {
            scope.setTag('error.boundary', 'screen');
            scope.setTag('screen.name', screenName);
          });
        }}
      >
        <WrappedComponent {...props} />
      </SentryErrorBoundary>
    );
  };
}
```
