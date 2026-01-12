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
    id("io.embrace.gradle") version "8.0.0" apply false
}

// build.gradle.kts (app)
plugins {
    id("io.embrace.gradle")
}

dependencies {
    implementation("io.embrace:embrace-android-sdk:8.0.0")
}
```

```kotlin
// Application.kt
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        Embrace.start(this)
    }
}
```

```xml
<!-- embrace-config.json -->
{
  "app_id": "YOUR_APP_ID",
  "api_token": "YOUR_API_TOKEN",
  "ndk_enabled": true,
  "sdk_config": {
    "crash_handler": {
      "enabled": true
    }
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
Embrace.addSessionProperty("user_tier", "premium", true)
Embrace.addSessionProperty("cart_value", cart.total.toString(), false)
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
Embrace.logInfo("Payment processing started", mapOf("method" to paymentMethod))
Embrace.logError("Payment failed", mapOf("error_code" to error.code))
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
Embrace.setUserIdentifier("user_123")
Embrace.setUsername("John Doe")
Embrace.setUserEmail("john@example.com")

// Clear on logout
Embrace.clearUserInfo()
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
val span = Embrace.createSpan("fetch_product")
span.addAttribute("product_id", productId)
span.start()

// ... perform operation

span.stop()
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
// iOS - hang detection is automatic
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
- Custom traces

### Crash Analysis

- Grouped by root cause
- Affected user count
- Device/OS breakdown
- Stack traces with source maps

### Performance Metrics

- App startup time
- Screen load time
- Network latency
- Custom trace timing

---

## Error Handling

### iOS Error Capture

```swift
// Capture non-fatal errors
func handleError(_ error: Error, context: String) {
    Embrace.client?.log(
        message: "Error in \(context)",
        severity: .error,
        properties: [
            "error_type": String(describing: type(of: error)),
            "error_message": error.localizedDescription,
            "context": context
        ]
    )
}

// Error boundary for async operations
func safeAsync<T>(
    context: String,
    operation: @escaping () async throws -> T
) async -> T? {
    do {
        return try await operation()
    } catch {
        handleError(error, context: context)
        return nil
    }
}
```

### Android Error Capture

```kotlin
// Capture non-fatal errors
fun handleError(error: Throwable, context: String) {
    Embrace.logError(
        "Error in $context",
        mapOf(
            "error_type" to error.javaClass.simpleName,
            "error_message" to (error.message ?: "unknown"),
            "context" to context
        )
    )
}

// Global error handler
Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
    Embrace.logError(
        "Uncaught exception",
        mapOf(
            "thread" to thread.name,
            "error_type" to throwable.javaClass.simpleName
        )
    )
    // Let Embrace crash reporter handle it
}
```

---

## React Native Hooks

```typescript
// useEmbraceSpan.ts
import { useEffect, useRef } from 'react';
import { startSpan, endSpan } from '@embrace-io/react-native';

export function useEmbraceSpan(name: string, attributes?: Record<string, string>) {
  const spanRef = useRef<string | null>(null);

  useEffect(() => {
    spanRef.current = startSpan(name, attributes);

    return () => {
      if (spanRef.current) {
        endSpan(spanRef.current);
      }
    };
  }, [name]);

  return spanRef.current;
}

// useEmbraceScreen.ts
import { useEffect } from 'react';
import { logScreenView } from '@embrace-io/react-native';

export function useEmbraceScreen(screenName: string) {
  useEffect(() => {
    logScreenView(screenName);
  }, [screenName]);
}

// Usage in component
function CheckoutScreen() {
  useEmbraceScreen('Checkout');
  useEmbraceSpan('checkout_flow', { step: 'payment' });

  return <PaymentForm />;
}
```

### Error Boundary for React Native

```typescript
import { Component, ReactNode } from 'react';
import { logError } from '@embrace-io/react-native';

interface Props {
  children: ReactNode;
  fallback: ReactNode;
}

interface State {
  hasError: boolean;
}

class EmbraceErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(): State {
    return { hasError: true };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    logError(error.message, {
      componentStack: errorInfo.componentStack ?? 'unknown',
      errorName: error.name,
    });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}
```

---

## Alert Configuration

Configure in Embrace Dashboard → Alerts:

### Recommended Thresholds

| Alert Type | Threshold | Action |
|------------|-----------|--------|
| **Crash rate spike** | >1% in 15 min | Page on-call |
| **ANR rate** | >0.5% | Slack notification |
| **Trace failure** | >5% for key flows | Slack notification |
| **New crash type** | Any | Email team |
| **Startup regression** | >20% slower | Daily digest |

### PagerDuty Integration

```
Dashboard → Settings → Integrations → PagerDuty
- Crash rate > 2%: P1
- Crash rate > 5%: P0
- Key flow failure > 10%: P2
```

---

## Best Practices

| Practice | Implementation |
|----------|----------------|
| **Name traces clearly** | `checkout_flow`, `search_complete` |
| **Add context** | Session properties for user segments |
| **Track user ID** | Enable user timeline search |
| **Mark startup end** | Call `endAppStartup()` when truly ready |
| **Use severity levels** | Info for events, error for failures |

---

## Anti-Patterns

| Don't | Why | Do Instead |
|-------|-----|------------|
| Start spans without ending | Memory leak, unclear timelines | Always pair start/end |
| Log PII in messages | Compliance risk | Use sanitized IDs |
| Skip user identification | Can't find specific sessions | Set user ID on auth |
| Ignore trace failures | Misses conversion issues | Add failure conditions |

---

## Links

- [Embrace iOS Docs](https://embrace.io/docs/ios/)
- [Embrace Android Docs](https://embrace.io/docs/android/)
- [Embrace React Native Docs](https://embrace.io/docs/react-native/)
