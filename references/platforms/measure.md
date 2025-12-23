# Measure.sh Integration Guide

Self-hosted mobile observability platform with complete data sovereignty. Automatic crash, ANR, and performance tracking for Android, iOS, and Flutter.

## Quick Start

### Android (Kotlin)

```kotlin
// build.gradle.kts (app-level)
plugins {
    id("sh.measure.android.gradle") version "0.11.0"
}

dependencies {
    implementation("sh.measure:measure-android:0.15.1")
}
```

```xml
<!-- AndroidManifest.xml -->
<application>
    <meta-data
        android:name="sh.measure.android.API_KEY"
        android:value="YOUR_API_KEY"/>
    <meta-data
        android:name="sh.measure.android.API_URL"
        android:value="https://measure-api.yourcompany.com"/>
</application>
```

```kotlin
// Application.kt
import sh.measure.android.Measure
import sh.measure.android.config.MeasureConfig

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        Measure.init(
            context = this,
            config = MeasureConfig(
                samplingRateForErrorFreeSessions = 1.0f,  // 100% sampling
                enableLogging = BuildConfig.DEBUG,
                trackScreenshotOnCrash = true,
                sessionEndThresholdMs = 30_000  // 30s background = session end
            )
        )
    }
}
```

**Requirements**: Android Gradle Plugin 8.0.2+, Min SDK 21, Target SDK 35

### iOS (Swift)

```ruby
# Podfile
pod 'measure-sh'
```

**Or via Swift Package Manager**:

```swift
// Package.swift
dependencies: [
    .package(
        url: "https://github.com/measure-sh/measure.git",
        branch: "ios-v0.8.1"
    )
]
```

```swift
// AppDelegate.swift
import MeasureSDK

func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {

    let config = BaseMeasureConfig(
        samplingRateForErrorFreeSessions: 1.0,
        enableLogging: AppConfig.isDebug,
        trackScreenshotOnCrash: true
    )

    let clientInfo = ClientInfo(
        apiKey: "YOUR_API_KEY",
        apiUrl: "https://measure-api.yourcompany.com"
    )

    Measure.initialize(with: clientInfo, config: config)

    return true
}
```

**Requirements**: Xcode 15.0+, iOS 12.0+, Swift 5.10+

### Flutter

```yaml
# pubspec.yaml
dependencies:
  measure_flutter: ^0.3.1
```

```dart
// main.dart
import 'package:measure_flutter/measure_flutter.dart';
import 'dart:io' show Platform;

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await Measure.instance.init(
    () => runApp(MeasureWidget(child: MyApp())),
    config: const MeasureConfig(
      samplingRateForErrorFreeSessions: 1.0,
      enableLogging: kDebugMode,
    ),
    clientInfo: ClientInfo(
      apiKey: Platform.isAndroid
          ? "ANDROID_API_KEY"
          : "IOS_API_KEY",
      apiUrl: "https://measure-api.yourcompany.com",
    ),
  );
}
```

**Additional Android Setup**: Add Gradle plugin and AndroidManifest entries (see Android section above).

**Requirements**: Flutter 3.10+, Dart 3.0+

---

## Key Features

### Automatic Crash Detection

Captures unhandled exceptions with full context:

```kotlin
// Android - Crashes tracked automatically
// Optional: Add custom attributes before crash
Measure.addAttribute("checkout_step", "payment")
Measure.addAttribute("cart_value", "129.99")

// Crash occurs...
throw RuntimeException("Payment gateway timeout")
// → Captured with attributes, screenshot, session timeline
```

```swift
// iOS - Crashes tracked automatically
// Optional: Add custom attributes
Measure.addAttribute("checkout_step", value: "payment")
Measure.addAttribute("cart_value", value: "129.99")

// Crash occurs...
fatalError("Payment gateway timeout")
// → Captured with attributes, screenshot, session timeline
```

**Captured Context**:
- Stack traces with symbolication
- Screenshot at crash moment
- Complete session timeline (clicks, navigations, network)
- Device state (memory, battery, network connectivity)
- Custom attributes

### ANR Detection (Android)

Automatically detects Application Not Responding events:

```kotlin
// Automatically tracked - no code needed
// Captures when main thread is blocked for >5s

// View in dashboard:
// - Stack trace of blocking thread
// - Screenshot when ANR occurred
// - Session timeline leading to ANR
```

**ANR Triggers**:
- Main thread blocked >5 seconds
- UI freeze during user interaction
- Excessive background processing

### Session Timelines

Every session captures full user journey:

```kotlin
// Android - Automatic tracking
Measure.trackScreen("ProductDetails")  // Manual screen tracking
```

```swift
// iOS - Automatic tracking
Measure.trackScreen("ProductDetails")  // Manual screen tracking
```

**Auto-Tracked Events**:
- Screen navigations
- User taps and gestures
- HTTP requests/responses
- App lifecycle (foreground/background)
- Logs and console output
- Memory warnings
- Cold/warm app launches

### Performance Tracing

Track custom operations with spans:

```kotlin
// Android - Custom trace
val span = Measure.startSpan("load_product_details")
span?.setAttribute("product_id", productId)

try {
    val product = api.fetchProduct(productId)
    val reviews = api.fetchReviews(productId)

    span?.setAttribute("reviews_count", reviews.size.toString())
} finally {
    span?.end()  // Records duration
}
```

```swift
// iOS - Custom trace
let span = Measure.startSpan("load_product_details")
span?.setAttribute("product_id", value: productId)

do {
    let product = try await api.fetchProduct(productId)
    let reviews = try await api.fetchReviews(productId)

    span?.setAttribute("reviews_count", value: "\(reviews.count)")
} catch {
    span?.setAttribute("error", value: error.localizedDescription)
}

span?.end()  // Records duration
```

**Auto-Tracked Performance**:
- App launch time (cold/warm)
- Screen load time
- HTTP request latency
- Frame drops and UI jank

### Bug Reporting

Capture user-initiated bug reports:

```kotlin
// Android - Shake to report
// Automatically enabled - device shake triggers report UI

// Manual trigger
Measure.reportBug(
    title = "Can't complete checkout",
    description = "Payment button not responding"
)
```

```swift
// iOS - Shake to report
// Automatically enabled - device shake triggers report UI

// Manual trigger
Measure.reportBug(
    title: "Can't complete checkout",
    description: "Payment button not responding"
)
```

**Bug Report Includes**:
- User description
- Screenshot
- Full session timeline
- Device logs
- Network activity
- Custom attributes

### Custom Events

Track business-specific events:

```kotlin
// Android
Measure.trackEvent(
    name = "purchase_completed",
    attributes = mapOf(
        "order_id" to orderId,
        "total_amount" to "99.99",
        "payment_method" to "credit_card",
        "items_count" to "3"
    )
)
```

```swift
// iOS
Measure.trackEvent(
    name: "purchase_completed",
    attributes: [
        "order_id": orderId,
        "total_amount": "99.99",
        "payment_method": "credit_card",
        "items_count": "3"
    ]
)
```

```dart
// Flutter
Measure.instance.trackEvent(
  'purchase_completed',
  attributes: {
    'order_id': orderId,
    'total_amount': '99.99',
    'payment_method': 'credit_card',
    'items_count': '3',
  },
);
```

### User Identification

Associate sessions with users:

```kotlin
// Android - Set user
Measure.setUserId("user_12345")

// Add user attributes
Measure.addUserAttribute("subscription_tier", "premium")
Measure.addUserAttribute("region", "US-West")

// Clear on logout
Measure.clearUserId()
```

```swift
// iOS - Set user
Measure.setUserId("user_12345")

// Add user attributes
Measure.addUserAttribute("subscription_tier", value: "premium")
Measure.addUserAttribute("region", value: "US-West")

// Clear on logout
Measure.clearUserId()
```

---

## Self-Hosting Setup

### Prerequisites

- **VM**: x86-64 Linux (Ubuntu 24.04 LTS or Debian 12)
- **Resources**: 4+ vCPUs, 16GB RAM, 100GB disk
- **Network**: Ports 80/443 open, SSH access
- **DNS**: Two subdomains (dashboard and API)
- **Email**: SMTP provider (SendGrid, Mailgun, etc.)

### Installation Steps

```bash
# 1. Clone repository (use stable release tag)
git clone https://github.com/measure-sh/measure.git -b v1.2.3
cd measure/self-host

# 2. Run installation script
sudo ./install.sh

# Optional: Use podman instead of docker
sudo ./install.sh --podman
```

**Configuration Wizard Prompts**:

```bash
# Company/team namespace
Enter namespace: acme-corp

# Dashboard URL
Enter dashboard URL: https://measure.acme.com

# API endpoint URL
Enter API URL: https://measure-api.acme.com

# Google OAuth (for team authentication)
Enter Google Client ID:
Enter Google Client Secret:

# GitHub OAuth (optional)
Enter GitHub Client ID:
Enter GitHub Client Secret:

# SMTP email provider
Enter SMTP host: smtp.sendgrid.net
Enter SMTP port: 587
Enter SMTP username: apikey
Enter SMTP password: SG.xxx

# Slack integration (optional)
Enter Slack webhook URL:
```

### Reverse Proxy Setup (Caddy)

```bash
# Install Caddy
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy

# Configure Caddy (example Caddyfile)
cat > /etc/caddy/Caddyfile <<EOF
measure.acme.com {
    reverse_proxy localhost:3000
}

measure-api.acme.com {
    reverse_proxy localhost:8080
}
EOF

# Reload Caddy
sudo systemctl reload caddy
```

### DNS Configuration

```bash
# Create A records pointing to VM external IP
measure.acme.com        A    203.0.113.10
measure-api.acme.com    A    203.0.113.10
```

### Verify Installation

```bash
# Check all services running
docker compose ps

# Expected services:
# - dashboard (web UI)
# - api (backend)
# - postgres (database)
# - redis (cache)
# - clickhouse (analytics)

# View logs
docker compose logs -f

# Access dashboard
open https://measure.acme.com
```

---

## Data Sovereignty Benefits

### Complete Data Control

```yaml
Data Storage:
  - All data stays on your infrastructure
  - No third-party access
  - Full GDPR/HIPAA compliance control
  - Custom retention policies

Privacy Advantages:
  - No external data sharing
  - PII handling under your control
  - Audit logs on your servers
  - Custom data encryption
```

### Cost Advantages

```bash
# No per-event pricing
# No per-session pricing
# No per-user pricing
# No data egress fees

# Costs:
# - VM hosting (~$100-200/month)
# - Storage (scales with retention)
# - No vendor markup on events
```

### Compliance Benefits

| Requirement | Self-Hosted Solution |
|-------------|---------------------|
| **GDPR** | Data residency in EU fully controllable |
| **HIPAA** | PHI storage under direct control |
| **SOC2** | Audit logs and access controls configurable |
| **Data Localization** | Deploy in required geography |
| **Right to Deletion** | Implement custom retention logic |

---

## Advanced Configuration

### Sampling Strategy

```kotlin
// Android - Reduce data volume for high-traffic apps
Measure.init(
    context = this,
    config = MeasureConfig(
        // Capture 100% of sessions with errors
        // Capture 10% of error-free sessions
        samplingRateForErrorFreeSessions = 0.1f,

        // Alternative: Dynamic sampling per user
        enableDynamicSampling = true  // Uses server-side sampling config
    )
)
```

```swift
// iOS - Sampling configuration
let config = BaseMeasureConfig(
    samplingRateForErrorFreeSessions: 0.1,  // 10% sampling
    enableDynamicSampling: true
)
```

### Data Retention

```bash
# Configure in self-hosted deployment
# Edit docker-compose.yml or .env

# Example retention policies:
RETENTION_RAW_EVENTS=30d        # Raw event data
RETENTION_AGGREGATED=90d        # Aggregated metrics
RETENTION_CRASH_REPORTS=1y      # Crash reports
```

### Network Filtering

```kotlin
// Android - Exclude sensitive endpoints from auto-capture
// Note: Check Measure.sh SDK docs for current API.
// URL filtering may be done via event processing or backend configuration.
//
// Example pattern (if SDK supports it):
// MeasureConfig(
//     httpUrlFilter = listOf(
//         "/api/auth/",
//         "/api/payment/"
//     )
// )
```

### Screenshot Privacy

```kotlin
// Android - Disable screenshots for sensitive screens
class PaymentActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Prevent screenshot capture in session recordings
        window.setFlags(
            WindowManager.LayoutParams.FLAG_SECURE,
            WindowManager.LayoutParams.FLAG_SECURE
        )
    }
}
```

```swift
// iOS - Mark views as sensitive
class CreditCardView: UIView {
    override var isSecure: Bool { true }  // Excluded from screenshots
}
```

---

## Best Practices

| Practice | Implementation |
|----------|----------------|
| **Sample wisely** | 100% errors, 10-20% error-free sessions |
| **Set user context** | `setUserId()` on login, `clearUserId()` on logout |
| **Track business events** | Use `trackEvent()` for conversions, checkouts |
| **Name screens consistently** | Use `trackScreen()` with clear naming scheme |
| **Filter sensitive data** | Exclude auth/payment endpoints from network tracking |
| **Version releases** | Tag builds with version for filtering in dashboard |
| **Monitor health metrics** | Review crash-free session rate, ANR rate weekly |

---

## Migration from SaaS Vendors

### From Sentry

```kotlin
// Before (Sentry)
Sentry.captureException(exception)

// After (Measure)
// Automatic crash capture - no code needed
// Or add context:
Measure.addAttribute("error_context", "checkout_flow")
throw exception  // Automatically captured
```

### From Firebase Crashlytics

```kotlin
// Before (Crashlytics)
FirebaseCrashlytics.getInstance().log("User action")
FirebaseCrashlytics.getInstance().recordException(exception)

// After (Measure)
Measure.trackEvent("user_action")
// Crashes captured automatically with full timeline
```

### Data Export

```bash
# Export from Measure self-hosted instance
# Direct PostgreSQL/ClickHouse access for migrations

# Example: Export crash data
docker exec measure-postgres pg_dump \
    -U measure -t crashes --data-only > crashes.sql
```

---

## Performance Impact

| Metric | Impact |
|--------|--------|
| **App size increase** | ~500KB (Android), ~400KB (iOS) |
| **Memory overhead** | <5MB during active session |
| **CPU overhead** | <1% on background thread |
| **Network usage** | ~50-100KB per session (depends on activity) |
| **Battery impact** | Negligible (<1%) |

**Optimization Tips**:
- Use sampling for high-traffic apps
- Batch network uploads (default: every 30s or on app background)
- Filter verbose network endpoints
- Adjust session timeout threshold

---

## Troubleshooting

### No Data Appearing in Dashboard

```kotlin
// Android - Enable debug logging
Measure.init(
    context = this,
    config = MeasureConfig(
        enableLogging = true  // Check Logcat for SDK logs
    )
)

// Verify API URL reachable
// Check AndroidManifest.xml API_URL value
// Confirm API key is correct
```

```swift
// iOS - Enable debug logging
let config = BaseMeasureConfig(
    enableLogging: true  // Check Console for SDK logs
)

// Verify API URL reachable from device/simulator
// Check network connectivity
// Confirm API key is correct
```

### Crashes Not Symbolicated

```bash
# Android - Verify ProGuard mapping uploaded
./gradlew uploadMeasureMappingFiles

# iOS - Verify dSYM uploaded
# Check Xcode build settings:
# - Debug Information Format: DWARF with dSYM File
# - Upload dSYMs as part of CI/CD
```

### High Data Volume

```kotlin
// Reduce sampling rate
Measure.init(
    context = this,
    config = MeasureConfig(
        samplingRateForErrorFreeSessions = 0.05f,  // 5% instead of 100%
    )
)

// Filter noisy endpoints
Measure.init(
    context = this,
    config = MeasureConfig(
        networkFilter = { request ->
            !request.url.contains("/api/metrics/")  // Exclude telemetry endpoints
        }
    )
)
```

---

## Links

- [GitHub Repository](https://github.com/measure-sh/measure)
- [SDK Integration Guide](https://github.com/measure-sh/measure/blob/main/docs/sdk-integration-guide.md)
- [Self-Hosting Guide](https://github.com/measure-sh/measure/blob/main/docs/hosting/README.md)
- [Feature Documentation](https://github.com/measure-sh/measure/tree/main/docs/features)
- [Contributing Guide](https://github.com/measure-sh/measure/blob/main/CONTRIBUTING.md)
- [FAQs](https://github.com/measure-sh/measure/blob/main/docs/faqs.md)
