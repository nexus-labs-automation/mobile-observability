# Embrace Integration Guide

Embrace mobile-first observability for iOS, Android, and React Native.

## Quick Start

### iOS (Swift)

```bash
# SPM
.package(url: "https://github.com/embrace-io/embrace-apple-sdk", from: "6.0.0")
```

```swift
// AppDelegate.swift
import EmbraceIO

func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {

    do {
        try Embrace
            .setup(
                options: Embrace.Options(
                    appId: "YOUR_APP_ID",
                    platform: .iOS,
                    captureServices: .automatic,
                    crashReporter: .embrace
                )
            )
            .start()
    } catch {
        print("Embrace setup failed: \(error)")
    }

    return true
}
```

### Android (Kotlin)

```kotlin
// build.gradle.kts (project)
plugins {
    id("io.embrace.swazzler") version "6.0.0" apply false
}

// build.gradle.kts (app)
plugins {
    id("io.embrace.swazzler")
}

dependencies {
    implementation("io.embrace:embrace-android-sdk:6.0.0")
}
```

```kotlin
// Application.kt
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        Embrace.getInstance().start(this)
    }
}
```

```xml
<!-- embrace-config.json -->
{
  "app_id": "YOUR_APP_ID",
  "api_token": "YOUR_API_TOKEN",
  "ndk_enabled": true,
  "crash_handler": {
    "enabled": true
  }
}
```

### React Native

```bash
npm install @embrace-io/react-native
```

```typescript
// index.js
import { initialize } from '@embrace-io/react-native';

initialize({
  sdkConfig: {
    ios: { appId: 'YOUR_IOS_APP_ID' },
    android: { appId: 'YOUR_ANDROID_APP_ID' }
  }
});
```

---

## Key Features

### User Sessions

Embrace is session-centric. Every user session is recorded with full context.

```swift
// iOS - Add session properties
Embrace.client?.metadata.addProperty(
    key: "user_tier",
    value: "premium",
    permanent: true  // Persists across sessions
)

// Temporary property (this session only)
Embrace.client?.metadata.addProperty(
    key: "cart_value",
    value: String(cart.total),
    permanent: false
)
```

```kotlin
// Android - Add session properties
Embrace.getInstance().addSessionProperty("user_tier", "premium", true)
Embrace.getInstance().addSessionProperty("cart_value", cart.total.toString(), false)
```

### Moments (User Flows)

Track critical user flows with start/end timing.

```swift
// iOS - Track a moment
Embrace.client?.startMoment(name: "checkout_flow")

// ... user completes checkout

Embrace.client?.endMoment(
    name: "checkout_flow",
    properties: ["order_value": String(order.total)]
)
```

```kotlin
// Android - Track a moment
Embrace.getInstance().startMoment("checkout_flow")

// ... user completes checkout

Embrace.getInstance().endMoment("checkout_flow", mapOf("order_value" to order.total.toString()))
```

### Log Messages

```swift
// iOS - Log with severity
Embrace.client?.log(
    message: "Payment processing started",
    severity: .info,
    properties: ["method": paymentMethod]
)

Embrace.client?.log(
    message: "Payment failed",
    severity: .error,
    properties: ["error_code": error.code]
)
```

```kotlin
// Android - Log with severity
Embrace.getInstance().logInfo("Payment processing started", mapOf("method" to paymentMethod))
Embrace.getInstance().logError("Payment failed", mapOf("error_code" to error.code))
```

### User Identification

```swift
// iOS - Set user
Embrace.client?.metadata.userIdentifier = "user_123"
Embrace.client?.metadata.userName = "John Doe"
Embrace.client?.metadata.userEmail = "john@example.com"

// Clear on logout
Embrace.client?.clearUserInfo()
```

```kotlin
// Android - Set user
Embrace.getInstance().setUserIdentifier("user_123")
Embrace.getInstance().setUsername("John Doe")
Embrace.getInstance().setUserEmail("john@example.com")

// Clear on logout
Embrace.getInstance().clearUserInfo()
```

### Spans & Traces

```swift
// iOS - Create spans for operations
let span = Embrace.client?.buildSpan(name: "fetch_product")
    .setAttribute(key: "product_id", value: productId)
    .startSpan()

// ... perform operation

span?.end()
```

```kotlin
// Android - Create spans
val span = Embrace.getInstance().createSpan("fetch_product")
span?.addAttribute("product_id", productId)
span?.start()

// ... perform operation

span?.stop()
```

### Network Capture

```swift
// iOS - Network requests auto-captured
// Embrace instruments URLSession automatically

// For custom network clients:
Embrace.client?.recordNetworkRequest(
    url: url,
    httpMethod: "POST",
    startTime: startTime,
    endTime: endTime,
    bytesSent: requestSize,
    bytesReceived: responseSize,
    statusCode: 200
)
```

---

## Advanced Features

### Crash Grouping

Embrace groups crashes intelligently by:
- Exception type
- Stack trace similarity
- App version
- Device characteristics

### ANR Detection

```swift
// iOS - ANR/hang detection is automatic
// Configure threshold in dashboard

// Android - ANR auto-detected
// Reports include:
// - Main thread stack trace
// - All thread stack traces
// - Device state at time of ANR
```

### Startup Performance

```swift
// iOS - Startup tracing automatic
// Reports:
// - Cold start duration
// - Time to first frame
// - SDK initialization time

// Custom startup completion marker
Embrace.client?.endAppStartup()
```

### View Tracking

```swift
// iOS - Automatic UIViewController tracking
// For SwiftUI:
struct ProductView: View {
    var body: some View {
        VStack {
            // content
        }
        .embraceView(name: "ProductDetail")
    }
}
```

```kotlin
// Android - Automatic Activity/Fragment tracking
// For Compose:
@Composable
fun ProductScreen() {
    EmbraceComposable(screenName = "ProductDetail") {
        // content
    }
}
```

---

## Dashboard Features

### User Timeline

Visual timeline of every user session:
- Screen views
- Network requests
- Logs
- Crashes
- ANRs
- Custom moments

### Crash Analysis

- Grouped by root cause
- Affected user count
- Device/OS breakdown
- Stack traces with source maps

### Performance Metrics

- App startup time
- Screen load time
- Network latency
- Custom moments timing

---

## Best Practices

| Practice | Implementation |
|----------|----------------|
| **Name moments clearly** | `checkout_flow`, `search_complete` |
| **Add context** | Session properties for user segments |
| **Track user ID** | Enable user timeline search |
| **Mark startup end** | Call `endAppStartup()` when truly ready |
| **Use severity levels** | Info for events, error for failures |

---

## Links

- [Embrace iOS Docs](https://embrace.io/docs/ios/)
- [Embrace Android Docs](https://embrace.io/docs/android/)
- [Embrace React Native Docs](https://embrace.io/docs/react-native/)
