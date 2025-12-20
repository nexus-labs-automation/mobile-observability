# Datadog Integration Guide

Datadog RUM, APM, and Logs setup for iOS, Android, and React Native mobile apps.

## Quick Start

### iOS (Swift)

```bash
# SPM
.package(url: "https://github.com/DataDog/dd-sdk-ios", from: "2.0.0")
```

```swift
// AppDelegate.swift
import DatadogCore
import DatadogRUM
import DatadogLogs
import DatadogTrace

func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {

    Datadog.initialize(
        with: Datadog.Configuration(
            clientToken: "YOUR_CLIENT_TOKEN",
            env: AppConfig.environment,
            service: "my-ios-app",
            bundle: .main
        ),
        trackingConsent: .granted
    )

    // RUM
    RUM.enable(
        with: RUM.Configuration(
            applicationID: "YOUR_APPLICATION_ID",
            sessionSampleRate: 100,
            telemetrySampleRate: 20,
            trackUIKitRUMViews: true,
            trackUIKitRUMActions: true,
            trackBackgroundEvents: true
        )
    )

    // Logs
    Logs.enable(
        with: Logs.Configuration(
            eventMapper: { log in
                // Sanitize PII
                return sanitizeLog(log)
            }
        )
    )

    // Trace
    Trace.enable(
        with: Trace.Configuration(
            sampleRate: 20,
            networkInfoEnabled: true
        )
    )

    // Set user info
    Datadog.setUserInfo(id: userId, name: nil, email: nil)

    return true
}
```

### Android (Kotlin)

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.datadoghq:dd-sdk-android-rum:2.0.0")
    implementation("com.datadoghq:dd-sdk-android-logs:2.0.0")
    implementation("com.datadoghq:dd-sdk-android-trace:2.0.0")
    implementation("com.datadoghq:dd-sdk-android-okhttp:2.0.0")
}
```

```kotlin
// Application.kt
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        val config = Configuration.Builder(
            clientToken = "YOUR_CLIENT_TOKEN",
            env = BuildConfig.BUILD_TYPE,
            variant = BuildConfig.FLAVOR
        )
            .useSite(DatadogSite.US1)
            .trackInteractions()
            .trackLongTasks(250L)
            .setFirstPartyHosts(listOf("api.yourapp.com"))
            .build()

        Datadog.initialize(this, config, TrackingConsent.GRANTED)
        Datadog.setUserInfo(userId)

        // RUM
        val rumConfig = RumConfiguration.Builder("YOUR_APPLICATION_ID")
            .trackUserInteractions()
            .trackLongTasks()
            .useViewTrackingStrategy(ActivityViewTrackingStrategy(true))
            .setSessionSampleRate(100f)
            .setTelemetrySampleRate(20f)
            .build()
        Rum.enable(rumConfig)

        // Logs
        val logsConfig = LogsConfiguration.Builder().build()
        Logs.enable(logsConfig)

        // Trace
        val traceConfig = TraceConfiguration.Builder().build()
        Trace.enable(traceConfig)
    }
}
```

### React Native

```bash
npm install @datadog/mobile-react-native
```

```typescript
// App.tsx
import { DdSdkReactNative, DdSdkReactNativeConfiguration } from '@datadog/mobile-react-native';

const config = new DdSdkReactNativeConfiguration(
  'YOUR_CLIENT_TOKEN',
  'production',
  'YOUR_APPLICATION_ID',
  true, // track interactions
  true, // track resources
  true  // track errors
);
config.site = 'US1';
config.nativeCrashReportEnabled = true;
config.sessionSamplingRate = 100;

await DdSdkReactNative.initialize(config);

// Set user
DdSdkReactNative.setUser({ id: userId });
```

---

## Key Features

### RUM Views

```swift
// iOS - Manual view tracking
import DatadogRUM

class ProductViewController: UIViewController {
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        RUMMonitor.shared().startView(
            viewController: self,
            name: "ProductDetail",
            attributes: ["product_id": productId]
        )
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        RUMMonitor.shared().stopView(viewController: self)
    }
}
```

```kotlin
// Android - Manual view tracking
GlobalRumMonitor.get().startView(
    key = this,
    name = "ProductDetail",
    attributes = mapOf("product_id" to productId)
)

GlobalRumMonitor.get().stopView(key = this)
```

### RUM Actions

```swift
// iOS - Track user actions
RUMMonitor.shared().addAction(
    type: .tap,
    name: "Add to Cart",
    attributes: ["product_id": productId, "quantity": quantity]
)
```

```kotlin
// Android - Track user actions
GlobalRumMonitor.get().addAction(
    type = RumActionType.TAP,
    name = "Add to Cart",
    attributes = mapOf("product_id" to productId, "quantity" to quantity)
)
```

### RUM Errors

```swift
// iOS - Track errors
RUMMonitor.shared().addError(
    error: error,
    source: .source,
    attributes: ["screen": "checkout", "step": "payment"]
)
```

```kotlin
// Android - Track errors
GlobalRumMonitor.get().addError(
    message = error.message ?: "Unknown error",
    source = RumErrorSource.SOURCE,
    throwable = error,
    attributes = mapOf("screen" to "checkout", "step" to "payment")
)
```

### Logs

```swift
// iOS - Structured logging
import DatadogLogs

let logger = Logger.create(
    with: Logger.Configuration(
        name: "my-logger",
        networkInfoEnabled: true,
        remoteLogThreshold: .info
    )
)

logger.info("User added item to cart", attributes: [
    "product_id": productId,
    "quantity": quantity
])

logger.error("Payment failed", error: error, attributes: [
    "payment_method": method
])
```

```kotlin
// Android - Structured logging
val logger = Logger.Builder()
    .setNetworkInfoEnabled(true)
    .setRemoteLogThreshold(Log.INFO)
    .build()

logger.i("User added item to cart", attributes = mapOf(
    "product_id" to productId,
    "quantity" to quantity
))

logger.e("Payment failed", error, attributes = mapOf(
    "payment_method" to method
))
```

### Distributed Tracing

```swift
// iOS - OkHttp/URLSession tracing auto-instrumented
// For manual spans:
let span = Tracer.shared().startSpan(operationName: "fetchProduct")
span.setTag(key: "product_id", value: productId)
// ... do work
span.finish()
```

```kotlin
// Android - OkHttp auto-instrumented
val okHttpClient = OkHttpClient.Builder()
    .addInterceptor(DatadogInterceptor())
    .eventListenerFactory(DatadogEventListener.Factory())
    .build()

// Manual spans
val span = GlobalTracer.get().buildSpan("fetchProduct")
    .withTag("product_id", productId)
    .start()
// ... do work
span.finish()
```

---

## Advanced Configuration

### Custom Attributes

```swift
// iOS - Global attributes
RUMMonitor.shared().addAttribute(forKey: "user_tier", value: "premium")
RUMMonitor.shared().addAttribute(forKey: "ab_variant", value: "checkout_v2")
```

### Resource Tracking

```swift
// iOS - Track network resources
RUMMonitor.shared().startResource(
    resourceKey: requestId,
    httpMethod: .get,
    urlString: url.absoluteString
)

// On completion
RUMMonitor.shared().stopResource(
    resourceKey: requestId,
    statusCode: response.statusCode,
    kind: .fetch
)
```

### Session Sampling

```swift
// iOS - Conditional sampling
let config = RUM.Configuration(
    applicationID: "YOUR_APP_ID",
    sessionSampleRate: shouldSampleUser() ? 100 : 0
)
```

---

## Best Practices

| Practice | Implementation |
|----------|----------------|
| **Name views clearly** | Use screen names, not class names |
| **Add context** | Product IDs, user segments, feature flags |
| **Connect traces** | Set `firstPartyHosts` for backend correlation |
| **Log levels** | Debug local, info+ remote |
| **Sample strategically** | 100% RUM, 20% traces |

---

## Useful Queries

```
# RUM Explorer
@type:error @view.name:Checkout

# Logs
service:my-ios-app status:error @usr.id:user_123

# APM
env:production service:my-ios-app @http.status_code:>=400
```

---

## Links

- [Datadog iOS SDK Docs](https://docs.datadoghq.com/real_user_monitoring/ios/)
- [Datadog Android SDK Docs](https://docs.datadoghq.com/real_user_monitoring/android/)
- [Datadog React Native Docs](https://docs.datadoghq.com/real_user_monitoring/reactnative/)
