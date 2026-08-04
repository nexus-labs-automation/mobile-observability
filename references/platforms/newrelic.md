# New Relic Integration Guide

New Relic Mobile observability for iOS, Android, React Native, and Flutter — full-stack APM with crash reporting, distributed tracing, session replay, and custom event analytics.

## Quick Start

### iOS (Swift)

```swift
// Package.swift / Xcode → Package Dependencies
.package(url: "https://github.com/newrelic/newrelic-ios-agent-spm", from: "7.7.4")
```

```swift
// AppDelegate.swift
import NewRelic

func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {

    NewRelic.start(withApplicationToken: "YOUR_IOS_APP_TOKEN")

    return true
}
```

### Android (Kotlin)

```kotlin
// build.gradle.kts (project)
plugins {
    id("com.newrelic.agent.android") version "7.7.7" apply false
}

// build.gradle.kts (app)
plugins {
    id("com.newrelic.agent.android")
}

dependencies {
    implementation("com.newrelic.agent.android:android-agent:7.7.7")
}
```

> **Keep the plugin and agent versions identical.** They ship in lockstep; a
> mismatch is a common cause of missing instrumentation or build failures.

```kotlin
// MainActivity.kt
import com.newrelic.agent.android.NewRelic

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        NewRelic
            .withApplicationToken("YOUR_ANDROID_APP_TOKEN")
            .start(this.application)

        setContentView(R.layout.activity_main)
    }
}
```

```xml
<!-- AndroidManifest.xml — required permissions -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

### React Native

```bash
npm install newrelic-react-native-agent
```

```typescript
// index.js
import NewRelic from 'newrelic-react-native-agent';
import { Platform, AppRegistry } from 'react-native';
import App from './App';
import { name as appName } from './app.json';
import appVersion from './package.json';

const appToken = Platform.OS === 'ios'
    ? 'YOUR_IOS_APP_TOKEN'
    : 'YOUR_ANDROID_APP_TOKEN';

const agentConfiguration = {
    // Android Specific — collect MobileSession / MobileBreadcrumb / MobileRequest events
    analyticsEventEnabled: true,

    // Android Specific — capture native (C/C++) crashes
    nativeCrashReportingEnabled: false,

    crashReportingEnabled: true,
    interactionTracingEnabled: false,
    networkRequestEnabled: true,
    networkErrorRequestEnabled: true,
    httpResponseBodyCaptureEnabled: true,
    loggingEnabled: true,
    logLevel: NewRelic.LogLevel.INFO,

    // iOS Specific
    webViewInstrumentation: true,

    offlineStorageEnabled: true,

    // iOS Specific
    backgroundReportingEnabled: false,
    newEventSystemEnabled: false,

    distributedTracingEnabled: true,
};

NewRelic.startAgent(appToken, agentConfiguration);
NewRelic.setJSAppVersion(appVersion.version);
AppRegistry.registerComponent(appName, () => App);
```

> **Order matters.** Call `startAgent` *before* `AppRegistry.registerComponent` so the agent is live when your root component mounts.

### Flutter

```yaml
# pubspec.yaml — pin a real version; `^latest` is not a valid pub constraint.
# Check pub.dev/packages/newrelic_mobile for the current release.
dependencies:
  newrelic_mobile: ^1.2.9
```

```dart
// main.dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:newrelic_mobile/newrelic_mobile.dart';
import 'package:newrelic_mobile/config.dart';

void main() {
    var appToken = "";

    if (Platform.isAndroid) {
        appToken = "<android app token>";
    } else if (Platform.isIOS) {
        appToken = "<ios app token>";
    }

    Config config = Config(
        accessToken: appToken,

        // Android Specific
        analyticsEventEnabled: true,
        networkErrorRequestEnabled: true,
        networkRequestEnabled: true,
        crashReportingEnabled: true,
        interactionTracingEnabled: true,
        httpResponseBodyCaptureEnabled: true,
        loggingEnabled: true,

        // iOS Specific
        webViewInstrumentation: true,

        printStatementAsEventsEnabled: true,
        httpInstrumentationEnabled: true,
        fedRampEnabled: false,
        offlineStorageEnabled: true,

        // iOS Specific
        backgroundReportingEnabled: false,
        newEventSystemEnabled: false,

        distributedTracingEnabled: true,
        logLevel: LogLevel.DEBUG,
    );

    NewrelicMobile.instance.start(config, () {
        runApp(MyApp());
    });
}
```

> **Flutter crash capture requires extra setup** beyond `start()` — follow the [Flutter setup guide](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile-flutter/monitor-your-flutter-application/) to wire up `FlutterError.onError` and zoned async error handling.

---

## Key Features

### Custom Attributes & User Identification

Attributes attach to every subsequent event in the session. User ID is searchable in the dashboard.

📖 [Create attribute](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile/mobile-sdk/create-attribute/)

```swift
// iOS
NewRelic.setAttribute("user_tier", value: "premium")
NewRelic.setAttribute("cart_value", value: cart.total)

NewRelic.setUserId("user_123")

// Clear on logout
NewRelic.setUserId("")
```

```kotlin
// Android
NewRelic.setAttribute("user_tier", "premium")
NewRelic.setAttribute("cart_value", cart.total)

NewRelic.setUserId("user_123")
```

```typescript
// React Native
NewRelic.setAttribute('user_tier', 'premium');
NewRelic.setAttribute('cart_value', cart.total);

NewRelic.setUserId('user_123');
```

```dart
// Flutter
NewrelicMobile.instance.setAttribute(name: 'user_tier', value: 'premium');
NewrelicMobile.instance.setAttribute(name: 'cart_value', value: cart.total);

NewrelicMobile.instance.setUserId('user_123');
```

### Custom Events

Custom events surface as queryable rows in NRDB under `MobileEvent` (or your custom event type).

📖 [Record custom events](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile/mobile-sdk/record-custom-events/)

```swift
// iOS
NewRelic.recordCustomEvent(
    "Checkout",
    name: "payment_submitted",
    attributes: [
        "method": paymentMethod,
        "amount_cents": amountCents,
        "step": "review"
    ]
)
```

```kotlin
// Android
NewRelic.recordCustomEvent(
    "Checkout",
    "payment_submitted",
    mapOf(
        "method" to paymentMethod,
        "amount_cents" to amountCents,
        "step" to "review"
    )
)
```

```typescript
// React Native
NewRelic.recordCustomEvent('Checkout', 'payment_submitted', {
    method: paymentMethod,
    amount_cents: amountCents,
    step: 'review',
});
```

```dart
// Flutter
NewrelicMobile.instance.recordCustomEvent(
    'Checkout',
    eventName: 'payment_submitted',
    eventAttributes: {
        'method': paymentMethod,
        'amount_cents': amountCents,
        'step': 'review',
    },
);
```

### Breadcrumbs

Lightweight session timeline — visible alongside crashes and errors for context.

📖 [Record breadcrumb](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile/mobile-sdk/record-breadcrumb/)

```swift
NewRelic.recordBreadcrumb(
    "ProductViewed",
    attributes: ["product_id": productId, "category": category]
)
```

```kotlin
NewRelic.recordBreadcrumb(
    "ProductViewed",
    mapOf("product_id" to productId, "category" to category)
)
```

```typescript
NewRelic.recordBreadcrumb('ProductViewed', {
    product_id: productId,
    category: category,
});
```

```dart
NewrelicMobile.instance.recordBreadcrumb(
    'ProductViewed',
    eventAttributes: {'product_id': productId, 'category': category},
);
```

### Handled Exceptions

Capture caught errors that don't crash the app — they appear in the Errors dashboard with stack traces.

📖 [Record handled exceptions](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile/mobile-sdk/record-handled-exceptions/)

```swift
// iOS uses recordError, not recordHandledException
do {
    try riskyOperation()
} catch {
    NewRelic.recordError(error, attributes: ["context": "checkout"])
}
```

```kotlin
// Android
try {
    riskyOperation()
} catch (e: Exception) {
    NewRelic.recordHandledException(e, mapOf("context" to "checkout"))
}
```

```typescript
// React Native
try {
    await riskyOperation();
} catch (error) {
    NewRelic.recordError(error);
}
```

```dart
// Flutter
try {
    await riskyOperation();
} catch (error, stackTrace) {
    NewrelicMobile.instance.recordError(
        error,
        stackTrace,
        attributes: {'context': 'checkout'},
    );
}
```

### Custom Interactions (Traces)

Wrap a logical operation to measure its duration and capture the call tree underneath. Interactions appear as traces in the dashboard.

📖 [Start interaction](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile/mobile-sdk/start-interaction/) · [Stop interaction](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile/mobile-sdk/stop-interaction/)

```swift
// iOS — startInteraction returns an interactionId String
let interactionId = NewRelic.startInteraction(withName: "checkout_flow")

await processCheckout()

NewRelic.stopCurrentInteraction(interactionId)
```

```kotlin
// Android
val interactionId = NewRelic.startInteraction("checkout_flow")

processCheckout()

NewRelic.endInteraction(interactionId)
```

```typescript
// React Native — returns Promise<string>
const interactionId = await NewRelic.startInteraction('checkout_flow');

await processCheckout();

NewRelic.endInteraction(interactionId);
```

```dart
// Flutter
String interactionId = await NewrelicMobile.instance.startInteraction('checkout_flow');

await processCheckout();

NewrelicMobile.instance.endInteraction(interactionId: interactionId);
```

### Network Capture & Distributed Tracing

- **iOS & Android:** HTTP requests via `URLSession` / `OkHttp` / `HttpURLConnection` are **auto-instrumented** — no manual code needed. Failed requests land in `MobileRequestError`; successful requests in `MobileRequest`.
- **React Native & Flutter:** controlled by the `networkRequestEnabled` and `networkErrorRequestEnabled` config flags shown in Quick Start.
- **Distributed tracing:** the agent **automatically injects W3C `traceparent` headers** on outbound requests when `distributedTracingEnabled` is `true` (the default). Backend spans correlate with the mobile span automatically — no manual header propagation required.
- **No per-call opt-out.** Distributed tracing is configured once at agent start; there is no per-request API to suppress the header. If a third-party API rejects unknown headers, disable `distributedTracingEnabled` globally or route that call through a client the agent doesn't instrument.

---

## Session Replay

New Relic Mobile Session Replay records the rendered UI of user sessions and aligns it with crashes, errors, and traces — so you can *see* what the user did before a failure.

**Platform support:**

| Platform | Status |
|----------|--------|
| iOS | ✅ Supported |
| Android | ✅ Supported |
| React Native | ✅ Supported |
| Flutter | 🚧 Coming soon |

**Privacy is the default.** Session replay masks input fields, text, and images by default. Override per-view to unmask non-sensitive UI.

📖 [Session Replay introduction](https://docs.newrelic.com/docs/mobile-monitoring/mobile-monitoring-ui/mobile-session-replay/introduction/)
📖 [Configure privacy settings](https://docs.newrelic.com/docs/mobile-monitoring/mobile-monitoring-ui/mobile-session-replay/configure-privacy-settings/)

> Always review the privacy settings doc before enabling replay in production. The default mask-everything policy is the right starting point — relax it deliberately, never wholesale.

---

## Crash Reporting

| Platform | Auto-capture |
|----------|--------------|
| iOS | ✅ Native crashes captured automatically |
| Android | ✅ Native + JVM crashes + ANRs captured automatically |
| React Native | ✅ Native + JS errors when `crashReportingEnabled: true` |
| Flutter | ✅ When the [setup guide](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile-flutter/monitor-your-flutter-application/) is followed (wire `FlutterError.onError` + `runZonedGuarded`) |

**App tokens.** Use **one app token per platform** (one for iOS, one for Android). Don't share tokens across platforms — the dashboard groups crashes by app token, and mixing them muddies platform-specific crash signatures.

---

## Symbolication

Without symbolication, NR shows crash stacks as memory addresses or obfuscated class names. Detail lives in `skills/symbolication-setup`; vendor specifics:

| Platform | Mechanism |
|----------|-----------|
| **iOS** | Build script or `newrelic-cli` upload of dSYMs. See [upload dSYMs/Bitcode](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile-ios/configuration/upload-dsyms-bitcode-apps/#command-line-manual-upload). |
| **Android** | The NR Gradle plugin auto-uploads ProGuard/R8 mappings on every release build. No manual step. |
| **React Native** | Source map upload **not yet supported** (expected ~August 2026). Until then, JS stack traces appear minified. |
| **Flutter** | Dart symbolication **not currently supported.** Native crashes (Android JVM, iOS Obj-C/Swift) symbolicate normally. |

---

## Alert Configuration

Alerts are configured in **NR One → Alerts & AI → Alert Conditions**, scoped to your mobile app entity.

📖 [Mobile monitoring alert information](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile/get-started/mobile-monitoring-alert-information/)

### Recommended Thresholds

| Alert Type | Threshold | Action |
|------------|-----------|--------|
| **Crash rate spike** | >1% in 15 min | Page on-call |
| **ANR rate** (Android) | >0.5% | Slack notification |
| **HTTP error rate** | >5% on key endpoints | Slack notification |
| **New crash signature** | Any | Email team |
| **Startup regression** | >20% slower vs 7-day baseline | Daily digest |

### NRQL Examples

```sql
-- Crash rate by app version (crash-free rate = 100 minus this)
SELECT percentage(uniqueCount(sessionId), WHERE category = 'Crash') AS 'Crash rate'
FROM MobileSession, MobileCrash
FACET appVersion
SINCE 24 hours ago

-- HTTP error rate by endpoint
SELECT percentage(count(*), WHERE statusCode >= 400)
FROM MobileRequest
FACET requestUrl
SINCE 1 hour ago

-- p95 interaction duration for key flow
SELECT percentile(duration, 95)
FROM Mobile
WHERE name = 'checkout_flow'
SINCE 1 day ago
```

> **Crash rate must query two event types.** `category` lives on `MobileCrash`,
> while the session denominator lives on `MobileSession` — so
> `FROM MobileSession, MobileCrash` is required. Querying `MobileSession` alone
> has no `category` attribute to filter on and will not produce a crash rate.
> Count `uniqueCount(sessionId)`, not `count(*)`, so multiple events from one
> session don't skew the ratio.
>
> To scope a query to a single app, add an entity filter — NR One's chart builder
> emits `WHERE (entityGuid = '<guid>' OR entity.guid = '<guid>')`, covering both
> attribute spellings across event types.

---

## Best Practices

| Practice | Implementation |
|----------|----------------|
| **One token per platform** | Don't reuse iOS token on Android — splits dashboards correctly |
| **Name interactions clearly** | `checkout_flow`, `search_complete` — avoid generic `screen_load` |
| **Set user ID on auth** | Enables session search and user impact attribution |
| **Keep attribute keys low-cardinality** | Use `user_tier=premium` not `user_id` as a *facet* |
| **Use `recordError` over generic logs** | Errors get stack traces and grouping; logs don't |
| **Disable `httpResponseBodyCaptureEnabled` if responses contain PII** | Default is on — review what your API returns |

---

## Anti-Patterns

| Don't | Why | Do Instead |
|-------|-----|------------|
| Start interactions without ending them | Memory leak; trace duration appears as session length | Always pair `startInteraction` / `endInteraction` (or `stopCurrentInteraction` on iOS) |
| Set unbounded `setAttribute` keys (e.g. user_id as attribute) | Cardinality explosion in NRDB; expensive queries | Use `setUserId` for identity; reserve attributes for low-cardinality dimensions |
| Skip Flutter crash setup | `start()` alone won't capture Dart errors | Wire `FlutterError.onError` + `runZonedGuarded` per the Flutter setup guide |
| Capture HTTP response bodies in production blindly | PII in responses gets logged | Audit API responses; disable `httpResponseBodyCaptureEnabled` for sensitive endpoints |
| Use the same app token for iOS and Android | Crashes group across platforms; obscures platform-specific signatures | One token per platform |

---

## Links

- [iOS — Introduction](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile-ios/get-started/introduction-new-relic-mobile-ios/)
- [Android — Install with Gradle](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile-android/install-configure/install-android-agent-gradle/)
- [React Native — Monitor your app](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-monitoring-react-native/monitor-your-react-native-application/)
- [Flutter — Monitor your app](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile-flutter/monitor-your-flutter-application/)
- [Mobile SDK API guide](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile/mobile-sdk/mobile-sdk-api-guide/)
- [Session Replay](https://docs.newrelic.com/docs/mobile-monitoring/mobile-monitoring-ui/mobile-session-replay/introduction/)
- [Mobile alert information](https://docs.newrelic.com/docs/mobile-monitoring/new-relic-mobile/get-started/mobile-monitoring-alert-information/)
