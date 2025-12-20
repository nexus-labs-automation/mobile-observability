# Sentry Integration Guide

Sentry setup and best practices for iOS, Android, and React Native mobile apps.

## Quick Start

### iOS (Swift)

```bash
# SPM
.package(url: "https://github.com/getsentry/sentry-cocoa", from: "8.0.0")
```

```swift
// AppDelegate.swift
import Sentry

func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    SentrySDK.start { options in
        options.dsn = "https://YOUR_DSN@sentry.io/PROJECT_ID"
        options.environment = AppConfig.environment
        options.releaseName = "\(Bundle.main.appVersion)+\(Bundle.main.buildNumber)"

        // Performance
        options.tracesSampleRate = 0.2  // 20% of transactions
        options.profilesSampleRate = 0.1 // 10% of profiled transactions

        // Session Replay
        options.experimental.sessionReplay.sessionSampleRate = 0.1
        options.experimental.sessionReplay.onErrorSampleRate = 1.0

        // Integrations
        options.enableAutoSessionTracking = true
        options.enableWatchdogTerminationTracking = true
        options.attachScreenshot = true
        options.attachViewHierarchy = true

        // Privacy
        options.beforeSend = { event in
            // Scrub PII
            return sanitizeEvent(event)
        }
    }
    return true
}
```

### Android (Kotlin)

```kotlin
// build.gradle.kts
plugins {
    id("io.sentry.android.gradle") version "4.0.0"
}

dependencies {
    implementation("io.sentry:sentry-android:7.0.0")
}

sentry {
    autoUploadProguardMapping.set(true)
    uploadNativeSymbols.set(true)
}
```

```kotlin
// Application.kt
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        SentryAndroid.init(this) { options ->
            options.dsn = "https://YOUR_DSN@sentry.io/PROJECT_ID"
            options.environment = BuildConfig.BUILD_TYPE
            options.release = "${BuildConfig.VERSION_NAME}+${BuildConfig.VERSION_CODE}"

            // Performance
            options.tracesSampleRate = 0.2
            options.profilesSampleRate = 0.1

            // Session Replay
            options.experimental.sessionReplay.sessionSampleRate = 0.1
            options.experimental.sessionReplay.onErrorSampleRate = 1.0

            // Integrations
            options.isEnableAutoSessionTracking = true
            options.isAttachScreenshot = true
            options.isAttachViewHierarchy = true

            // ANR detection
            options.anrTimeoutIntervalMillis = 5000

            // Privacy
            options.beforeSend = SentryOptions.BeforeSendCallback { event, _ ->
                sanitizeEvent(event)
            }
        }
    }
}
```

### React Native

```bash
npm install @sentry/react-native
npx @sentry/wizard@latest -i reactNative
```

```typescript
// App.tsx
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: 'https://YOUR_DSN@sentry.io/PROJECT_ID',
  environment: __DEV__ ? 'development' : 'production',

  // Performance
  tracesSampleRate: 0.2,
  profilesSampleRate: 0.1,

  // Session Replay
  _experiments: {
    replaysSessionSampleRate: 0.1,
    replaysOnErrorSampleRate: 1.0,
  },

  // React Navigation integration
  integrations: [
    new Sentry.ReactNavigationInstrumentation(),
  ],
});

// Wrap your app
export default Sentry.wrap(App);
```

---

## Key Features

### Error Tracking

```swift
// iOS - Capture errors
do {
    try riskyOperation()
} catch {
    SentrySDK.capture(error: error) { scope in
        scope.setTag(value: "checkout", key: "feature")
        scope.setExtra(value: cartItems.count, key: "cart_size")
    }
}

// Capture message
SentrySDK.capture(message: "User exceeded rate limit") { scope in
    scope.setLevel(.warning)
}
```

```kotlin
// Android - Capture errors
try {
    riskyOperation()
} catch (e: Exception) {
    Sentry.captureException(e) { scope ->
        scope.setTag("feature", "checkout")
        scope.setExtra("cart_size", cartItems.size)
    }
}
```

### Performance Monitoring

```swift
// iOS - Custom transaction
let transaction = SentrySDK.startTransaction(name: "Load Product", operation: "ui.load")

let fetchSpan = transaction.startChild(operation: "http.client", description: "Fetch product data")
let product = try await api.fetchProduct(id: productId)
fetchSpan.finish()

let renderSpan = transaction.startChild(operation: "ui.render", description: "Render product view")
// ... render
renderSpan.finish()

transaction.finish()
```

```kotlin
// Android - Custom transaction
val transaction = Sentry.startTransaction("Load Product", "ui.load")

val fetchSpan = transaction.startChild("http.client", "Fetch product data")
val product = api.fetchProduct(productId)
fetchSpan.finish()

val renderSpan = transaction.startChild("ui.render", "Render product view")
// ... render
renderSpan.finish()

transaction.finish()
```

### Breadcrumbs

```swift
// iOS - Add breadcrumbs
SentrySDK.addBreadcrumb(Breadcrumb(
    level: .info,
    category: "ui.tap"
))

// Auto breadcrumbs are collected for:
// - UI interactions
// - Network requests
// - App lifecycle
// - Console logs
```

### User Context

```swift
// iOS - Set user
let user = User()
user.userId = "user_123"
user.email = "user@example.com"  // Consider PII implications
user.segment = "premium"
SentrySDK.setUser(user)

// Clear on logout
SentrySDK.setUser(nil)
```

### Session Replay

```swift
// iOS - Mask sensitive views
import SentryPrivacy

class CreditCardField: UITextField {
    override var sentryReplayMask: Bool { true }
}

// SwiftUI
Text(creditCardNumber)
    .sentryReplayMask()
```

---

## Advanced Configuration

### Sampling Strategy

```swift
// iOS - Dynamic sampling
options.tracesSampler = { context in
    // Always sample errors
    if context.transactionContext.name.contains("error") {
        return 1.0
    }

    // Higher sampling for critical flows
    if context.transactionContext.name.contains("checkout") {
        return 0.5
    }

    // Default
    return 0.1
}
```

### Release Health

```swift
// Track session state
// Sentry auto-tracks:
// - Session starts
// - Session ends
// - Crash-free session rate
// - User count

// View in Sentry: Releases > Release Health
```

### Source Maps / dSYMs

```bash
# iOS - Upload dSYMs (in Xcode build phase)
if [ "${CONFIGURATION}" = "Release" ]; then
    export SENTRY_ORG="your-org"
    export SENTRY_PROJECT="your-project"
    sentry-cli upload-dif "$DWARF_DSYM_FOLDER_PATH"
fi

# Android - Auto-uploaded via Gradle plugin
# Verify: sentry-cli releases files YOUR_RELEASE list
```

---

## Best Practices

| Practice | Implementation |
|----------|----------------|
| **Tag releases** | `options.release = "1.0.0+100"` |
| **Set environment** | `options.environment = "production"` |
| **Add context** | Use tags, extras, and user context |
| **Mask PII** | Use `beforeSend` and replay masking |
| **Sample wisely** | 100% errors, 10-20% transactions |
| **Upload symbols** | Automate in CI/CD |

---

## Useful Queries

```
# Find all errors for a user
user.id:user_123

# Errors in checkout flow
tags[feature]:checkout level:error

# Slow transactions
transaction.duration:>2000

# Errors on specific release
release:1.0.0 level:error

# ANRs
tags[mechanism]:ANR
```

---

## Links

- [Sentry iOS SDK Docs](https://docs.sentry.io/platforms/apple/)
- [Sentry Android SDK Docs](https://docs.sentry.io/platforms/android/)
- [Sentry React Native Docs](https://docs.sentry.io/platforms/react-native/)
- [Session Replay](https://docs.sentry.io/product/session-replay/)
