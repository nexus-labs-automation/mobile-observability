# iOS Native Observability

Instrumentation patterns for native iOS applications (Swift/Objective-C).

## SDK Setup

### Sentry

```swift
// AppDelegate.swift or App.swift
import Sentry

@main
struct MyApp: App {
    init() {
        SentrySDK.start { options in
            options.dsn = "https://your-dsn@sentry.io/project"

            // Environment
            options.environment = Configuration.isProduction ? "production" : "development"

            // Release tracking
            options.releaseName = "\(Bundle.main.bundleIdentifier ?? "")@\(appVersion)+\(buildNumber)"
            options.dist = buildNumber

            // Performance
            options.tracesSampleRate = 0.2
            options.profilesSampleRate = 0.1

            // Enable all auto-instrumentation
            options.enableAutoPerformanceTracing = true
            options.enableUIViewControllerTracing = true
            options.enableNetworkTracking = true
            options.enableFileIOTracing = true
            options.enableCoreDataTracing = true

            // Breadcrumbs
            options.maxBreadcrumbs = 100
            options.enableAutoBreadcrumbTracking = true

            // Attachments
            options.attachScreenshot = true
            options.attachViewHierarchy = true

            // Session tracking
            options.enableAutoSessionTracking = true

            // Before send hook
            options.beforeSend = { event in
                // Strip PII
                var modifiedEvent = event
                modifiedEvent.user?.email = nil
                modifiedEvent.user?.ipAddress = nil
                return modifiedEvent
            }
        }
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

// Computed properties
private var appVersion: String {
    Bundle.main.infoDictionary?["CFBundleShortVersionString"] as? String ?? "unknown"
}

private var buildNumber: String {
    Bundle.main.infoDictionary?["CFBundleVersion"] as? String ?? "0"
}
```

### dSYM Upload (Critical)

Without dSYMs, crash reports show memory addresses instead of symbols.

**Xcode Build Phase Script:**

```bash
# Add as "Run Script" build phase AFTER "Compile Sources"
if [ "$CONFIGURATION" == "Release" ]; then
    export SENTRY_ORG="your-org"
    export SENTRY_PROJECT="your-project"
    export SENTRY_AUTH_TOKEN="your-token"

    ERROR=$(sentry-cli upload-dif "$DWARF_DSYM_FOLDER_PATH" 2>&1 >/dev/null)
    if [ ! $? -eq 0 ]; then
        echo "warning: sentry-cli - $ERROR"
    fi
fi
```

**Fastlane:**

```ruby
lane :upload_symbols do
  sentry_upload_dif(
    auth_token: ENV['SENTRY_AUTH_TOKEN'],
    org_slug: 'your-org',
    project_slug: 'your-project',
    path: './build/MyApp.xcarchive/dSYMs'
  )
end
```

### Datadog Alternative

```swift
import Datadog

Datadog.initialize(
    appContext: .init(),
    trackingConsent: .granted,
    configuration: Datadog.Configuration
        .builderUsing(
            rumApplicationID: "app-id",
            clientToken: "client-token",
            environment: "production"
        )
        .set(serviceName: "ios-app")
        .trackUIKitRUMViews()
        .trackUIKitRUMActions()
        .trackURLSession()
        .build()
)

RUM.enable(with: .init(applicationID: "app-id"))
Logs.enable()
Trace.enable()
```

## Error Capture

### Global Exception Handler

```swift
// Automatic with Sentry SDK, but for custom handling:
NSSetUncaughtExceptionHandler { exception in
    SentrySDK.capture(exception: exception)

    // Custom logging
    Logger.error("Uncaught exception: \(exception.name) - \(exception.reason ?? "")")
}
```

### Capturing Handled Errors

```swift
// Swift Error
do {
    try riskyOperation()
} catch {
    SentrySDK.capture(error: error) { scope in
        scope.setTag(value: "payment", key: "flow")
        scope.setContext(value: ["orderId": orderId], key: "order")
    }
}

// NSError
let nsError = NSError(domain: "MyApp", code: 1001, userInfo: [
    NSLocalizedDescriptionKey: "Payment failed",
    "orderId": orderId
])
SentrySDK.capture(error: nsError)

// Message (for non-error events)
SentrySDK.capture(message: "User attempted invalid operation")
```

### Attaching Context

```swift
// Scoped context (temporary)
SentrySDK.configureScope { scope in
    scope.setTag(value: "checkout", key: "flow")
    scope.setContext(value: [
        "cartItems": cartItems.count,
        "cartTotal": cartTotal
    ], key: "cart")
}

// Event-level context
SentrySDK.capture(error: error) { scope in
    scope.setExtra(value: debugData, key: "debug")
    scope.setLevel(.warning)
}
```

## Performance Monitoring

### App Launch Measurement

```swift
// Sentry auto-captures, but for custom spans:
class AppDelegate: UIResponder, UIApplicationDelegate {

    private var launchSpan: Span?

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {

        // Start launch span
        launchSpan = SentrySDK.startTransaction(
            name: "app.launch",
            operation: "app.lifecycle"
        )

        // Mark cold start detection
        let isColdStart = launchOptions == nil
        launchSpan?.setTag(value: isColdStart ? "cold" : "warm", key: "start_type")

        // ... initialization code ...

        return true
    }

    func applicationDidBecomeActive(_ application: UIApplication) {
        // Finish when app is truly active
        launchSpan?.finish()
        launchSpan = nil
    }
}
```

### View Controller Tracking

Automatic with `enableUIViewControllerTracing`, but for custom:

```swift
class BaseViewController: UIViewController {
    private var viewLoadSpan: Span?

    override func viewDidLoad() {
        super.viewDidLoad()

        viewLoadSpan = SentrySDK.startTransaction(
            name: String(describing: type(of: self)),
            operation: "ui.load"
        )
    }

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)

        viewLoadSpan?.finish()
        viewLoadSpan = nil
    }
}
```

### SwiftUI View Tracking

SwiftUI requires explicit instrumentation:

```swift
struct ContentView: View {
    @State private var span: Span?

    var body: some View {
        VStack {
            // content
        }
        .onAppear {
            span = SentrySDK.startTransaction(
                name: "ContentView",
                operation: "ui.load"
            )
        }
        .task {
            // Async work complete = view interactive
            span?.finish()
        }
    }
}

// Reusable modifier
struct TrackedView<Content: View>: View {
    let name: String
    let content: () -> Content
    @State private var span: Span?

    var body: some View {
        content()
            .onAppear {
                span = SentrySDK.startTransaction(name: name, operation: "ui.load")
            }
            .onDisappear {
                span?.finish()
            }
    }
}
```

### Network Request Instrumentation

```swift
// URLSession automatic with enableNetworkTracking
// For custom wrapping:

func performRequest(_ request: URLRequest) async throws -> Data {
    let span = SentrySDK.span?.startChild(
        operation: "http.client",
        description: "\(request.httpMethod ?? "GET") \(request.url?.path ?? "")"
    )

    do {
        let (data, response) = try await URLSession.shared.data(for: request)

        if let httpResponse = response as? HTTPURLResponse {
            span?.setData(value: httpResponse.statusCode, key: "http.status_code")
            span?.setData(value: data.count, key: "http.response_content_length")
        }

        span?.finish(status: .ok)
        return data

    } catch {
        span?.finish(status: .internalError)
        throw error
    }
}
```

## Breadcrumbs

### Automatic Breadcrumbs

Enabled by `enableAutoBreadcrumbTracking`:
- UIKit touch events
- View controller lifecycle
- Network requests
- App lifecycle events

### Manual Breadcrumbs

```swift
// User action
let crumb = Breadcrumb(level: .info, category: "user")
crumb.message = "Tapped checkout button"
crumb.data = ["cartItems": 3]
SentrySDK.addBreadcrumb(crumb)

// Navigation
SentrySDK.addBreadcrumb(Breadcrumb(level: .info, category: "navigation") {
    $0.message = "Navigated to ProfileScreen"
})

// State change
SentrySDK.addBreadcrumb(Breadcrumb(level: .info, category: "state") {
    $0.message = "User logged in"
    $0.data = ["userId": user.id, "isFirstLogin": user.isFirstLogin]
})

// Error (non-fatal)
SentrySDK.addBreadcrumb(Breadcrumb(level: .error, category: "error") {
    $0.message = "Payment validation failed"
    $0.data = ["reason": "insufficient_funds"]
})
```

### App Lifecycle Breadcrumbs

```swift
class AppLifecycleObserver {
    init() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(didEnterBackground),
            name: UIApplication.didEnterBackgroundNotification,
            object: nil
        )

        NotificationCenter.default.addObserver(
            self,
            selector: #selector(willEnterForeground),
            name: UIApplication.willEnterForegroundNotification,
            object: nil
        )

        NotificationCenter.default.addObserver(
            self,
            selector: #selector(didReceiveMemoryWarning),
            name: UIApplication.didReceiveMemoryWarningNotification,
            object: nil
        )
    }

    @objc private func didEnterBackground() {
        SentrySDK.addBreadcrumb(Breadcrumb(level: .info, category: "app.lifecycle") {
            $0.message = "App entered background"
        })
    }

    @objc private func willEnterForeground() {
        SentrySDK.addBreadcrumb(Breadcrumb(level: .info, category: "app.lifecycle") {
            $0.message = "App entering foreground"
        })
    }

    @objc private func didReceiveMemoryWarning() {
        SentrySDK.addBreadcrumb(Breadcrumb(level: .warning, category: "app.lifecycle") {
            $0.message = "Received memory warning"
        })
    }
}
```

## User Context

```swift
// On login
func onUserLogin(user: User) {
    let sentryUser = Sentry.User()
    sentryUser.userId = user.id  // Internal ID only
    sentryUser.username = user.username  // Optional
    sentryUser.data = [
        "subscription": user.subscriptionTier,
        "accountAge": user.accountAgeDays
    ]

    SentrySDK.setUser(sentryUser)

    // Also set tags for filtering
    SentrySDK.configureScope { scope in
        scope.setTag(value: user.subscriptionTier, key: "user.tier")
    }
}

// On logout
func onUserLogout() {
    SentrySDK.setUser(nil)
}
```

## iOS-Specific Considerations

### Memory Pressure

iOS aggressively kills background apps. Track memory:

```swift
// Log memory state
func logMemoryPressure() {
    let memoryUsage = getMemoryUsage()
    SentrySDK.addBreadcrumb(Breadcrumb(level: .info, category: "memory") {
        $0.message = "Memory usage"
        $0.data = [
            "usedMB": memoryUsage.used / 1_000_000,
            "availableMB": memoryUsage.available / 1_000_000
        ]
    })
}

func getMemoryUsage() -> (used: UInt64, available: UInt64) {
    var info = mach_task_basic_info()
    var count = mach_msg_type_number_t(MemoryLayout<mach_task_basic_info>.size) / 4

    let result = withUnsafeMutablePointer(to: &info) {
        $0.withMemoryRebound(to: integer_t.self, capacity: 1) {
            task_info(mach_task_self_, task_flavor_t(MACH_TASK_BASIC_INFO), $0, &count)
        }
    }

    guard result == KERN_SUCCESS else {
        return (0, 0)
    }

    return (info.resident_size, ProcessInfo.processInfo.physicalMemory - info.resident_size)
}
```

### Background Task Tracking

```swift
func performBackgroundTask(name: String, work: @escaping () async -> Void) {
    var backgroundTask: UIBackgroundTaskIdentifier = .invalid

    backgroundTask = UIApplication.shared.beginBackgroundTask(withName: name) {
        // Expiration handler
        SentrySDK.addBreadcrumb(Breadcrumb(level: .warning, category: "background") {
            $0.message = "Background task expired: \(name)"
        })
        UIApplication.shared.endBackgroundTask(backgroundTask)
    }

    let span = SentrySDK.startTransaction(name: "background.\(name)", operation: "task")

    Task {
        await work()
        span.finish()
        UIApplication.shared.endBackgroundTask(backgroundTask)
    }
}
```

### Extension Crashes

App Extensions (widgets, share extensions) need separate initialization:

```swift
// WidgetBundle.swift
@main
struct MyWidgetBundle: WidgetBundle {
    init() {
        SentrySDK.start { options in
            options.dsn = "your-dsn"
            options.releaseName = "widget@\(appVersion)+\(buildNumber)"
            // Extensions have strict memory limits
            options.maxBreadcrumbs = 50
            options.attachScreenshot = false
        }
    }

    var body: some Widget {
        MyWidget()
    }
}
```

### Core Data Tracking

```swift
// Automatic with enableCoreDataTracing, but for custom:
func fetchUsers() async throws -> [User] {
    let span = SentrySDK.span?.startChild(
        operation: "db.query",
        description: "Fetch users"
    )

    defer { span?.finish() }

    let request = User.fetchRequest()
    let users = try context.fetch(request)

    span?.setData(value: users.count, key: "db.rows_affected")

    return users
}
```

## Common iOS Gotchas

### 1. Bitcode Disabled

Bitcode prevents dSYM upload from working. Ensure it's disabled or use App Store Connect symbol upload.

### 2. Swift Async Stack Traces

Async/await can produce fragmented stack traces. Sentry 8.x+ handles this, but update if seeing issues.

### 3. Crash in Initialization

If app crashes during Sentry init, it won't be captured. Initialize as early as possible:

```swift
@main
struct MyApp: App {
    init() {
        // First thing, before any other initialization
        SentrySDK.start { options in ... }

        // Then other setup
        configureServices()
    }
}
```

### 4. TestFlight vs Production

TestFlight builds use different provisioning. Tag appropriately:

```swift
options.environment = isTestFlight ? "testflight" : "production"

var isTestFlight: Bool {
    Bundle.main.appStoreReceiptURL?.lastPathComponent == "sandboxReceipt"
}
```

### 5. Symbolication Delay

dSYM processing takes 1-5 minutes. Don't panic if initial crashes show raw addresses.

---

## MetricKit Integration (Optional)

MetricKit provides supplementary diagnostic data from iOS—crash reports, hang call trees, and system metrics that augment your existing telemetry.

### Quick Setup

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

    func didReceive(_ payloads: [MXMetricPayload]) {
        // Daily aggregated metrics
    }

    @available(iOS 14.0, *)
    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        // Diagnostic reports with stack traces
    }
}
```

### What You Get

- Crash diagnostics (some stack traces)
- Hang diagnostics with call trees
- CPU/disk exception stack traces
- App exit reason counts

### What You Don't Get

- **OOM stack traces** (only exit counts)
- NSException name/message details
- User context or breadcrumbs
- Real-time delivery (24hr+ delay)

### When to Use

MetricKit is best as **supplementary diagnostics** that correlate with your existing telemetry by timestamp—not as a primary crash reporter.

See `references/metrickit.md` for comprehensive coverage.
See `skills/metrickit-integration` for quick setup.
See `references/templates/metrickit-subscriber.template.md` for copy-paste code.

---

## Reusable Templates

- **[templates/screen-load-tracking.template.md](templates/screen-load-tracking.template.md)** - iOS screen load tracking with os_signpost
- **[templates/error-boundary.template.md](templates/error-boundary.template.md)** - iOS crash handlers and signal handling
- **[templates/breadcrumb-manager.template.md](templates/breadcrumb-manager.template.md)** - iOS breadcrumb collection
- **[templates/navigation-tracking.template.md](templates/navigation-tracking.template.md)** - iOS navigation latency measurement
