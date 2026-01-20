# OpenTelemetry Mobile Integration Guide

OpenTelemetry setup for iOS and Android mobile apps. Use when you need vendor-neutral instrumentation or custom backend integration.

## Current Status (December 2025)

| Platform | Version | Tracing | Metrics | Logs | Maturity |
|----------|---------|---------|---------|------|----------|
| **iOS** | v2.3.0 | ✅ Stable | ⚠️ Outdated spec | 🧪 Beta | Production-ready for traces |
| **Android** | v1.0.1 | ✅ Stable | ✅ Stable | ✅ Stable | Production-ready |
| **React Native** | N/A | ❌ | ❌ | ❌ | Use vendor SDKs |

**Key Differences**:
- iOS: Swift package, manual instrumentation-focused
- Android: Auto-instrumentation for lifecycle, ANR, crashes, network, UI performance
- Both: Export to any OTLP backend (self-hosted or vendor)

---

## iOS Setup (opentelemetry-swift)

### Installation

#### Swift Package Manager (Recommended)

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/open-telemetry/opentelemetry-swift", from: "2.2.0")
]

targets: [
    .target(
        name: "YourApp",
        dependencies: [
            .product(name: "OpenTelemetryApi", package: "opentelemetry-swift"),
            .product(name: "OpenTelemetrySdk", package: "opentelemetry-swift"),
            .product(name: "ResourceExtension", package: "opentelemetry-swift"),
            .product(name: "URLSessionInstrumentation", package: "opentelemetry-swift"),
            .product(name: "OtlpHttpTraceExporter", package: "opentelemetry-swift")
        ]
    )
]
```

#### CocoaPods

```ruby
# Podfile
pod 'OpenTelemetrySwift', '~> 2.2'
pod 'OpenTelemetryApi', '~> 2.2'
pod 'OpenTelemetrySdk', '~> 2.2'
pod 'ResourceExtension', '~> 2.2'
pod 'URLSessionInstrumentation', '~> 2.2'
pod 'OtlpHttpTraceExporter', '~> 2.2'
```

### Basic Configuration

```swift
// AppDelegate.swift
import OpenTelemetryApi
import OpenTelemetrySdk
import ResourceExtension
import OtlpHttpTraceExporter

class AppDelegate: UIResponder, UIApplicationDelegate {

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {

        // Configure resource attributes
        let resource = Resource(attributes: [
            "service.name": AttributeValue.string("my-ios-app"),
            "service.version": AttributeValue.string(Bundle.main.appVersion),
            "deployment.environment": AttributeValue.string(AppConfig.environment),
            "device.model": AttributeValue.string(UIDevice.current.model),
            "os.name": AttributeValue.string("iOS"),
            "os.version": AttributeValue.string(UIDevice.current.systemVersion)
        ])

        // Configure OTLP exporter
        let otlpConfiguration = OtlpConfiguration(
            timeout: TimeInterval(10),
            headers: [
                ("x-api-key", "YOUR_API_KEY")  // If using vendor backend
            ]
        )

        let otlpExporter = OtlpHttpTraceExporter(
            endpoint: URL(string: "https://otlp.yourbackend.com/v1/traces")!,
            config: otlpConfiguration
        )

        // Configure span processor
        let spanProcessor = BatchSpanProcessor(
            spanExporter: otlpExporter,
            scheduleDelay: TimeInterval(5)
        )

        // Build tracer provider
        let tracerProvider = TracerProviderBuilder()
            .with(resource: resource)
            .add(spanProcessor: spanProcessor)
            .build()

        OpenTelemetry.registerTracerProvider(tracerProvider: tracerProvider)

        return true
    }
}
```

### URL Session Instrumentation

```swift
import URLSessionInstrumentation

// Create instrumented URLSession
let sessionConfig = URLSessionConfiguration.default
let urlSession = URLSession(
    configuration: sessionConfig,
    delegate: URLSessionInstrumentation(
        configuration: URLSessionInstrumentationConfiguration(
            shouldRecordPayload: { request in
                // Don't record request/response bodies for these endpoints
                !request.url?.absoluteString.contains("/auth") ?? true
            },
            shouldInstrument: { request in
                // Only instrument first-party APIs
                request.url?.host?.contains("yourapi.com") ?? false
            },
            nameSpan: { request in
                // Custom span naming
                "\(request.httpMethod ?? "GET") \(request.url?.path ?? "unknown")"
            }
        )
    ),
    delegateQueue: nil
)

// Use instrumented session
let task = urlSession.dataTask(with: request) { data, response, error in
    // Spans automatically created with:
    // - http.method
    // - http.url
    // - http.status_code
    // - http.response_content_length
}
task.resume()
```

### Manual Spans

```swift
import OpenTelemetryApi

class CheckoutViewModel {
    private let tracer = OpenTelemetry.instance.tracerProvider
        .get(instrumentationName: "com.myapp.checkout", instrumentationVersion: "1.0.0")

    func processOrder() async throws {
        // Create parent span
        let span = tracer.spanBuilder(spanName: "process_order").startSpan()
        span.setAttribute(key: "order.items", value: cartItems.count)
        span.setAttribute(key: "order.total", value: orderTotal)

        defer { span.end() }

        do {
            // Child span for validation
            let validationSpan = tracer.spanBuilder(spanName: "validate_order")
                .setParent(span)
                .startSpan()
            try validateOrder()
            validationSpan.end()

            // Child span for payment
            let paymentSpan = tracer.spanBuilder(spanName: "process_payment")
                .setParent(span)
                .startSpan()
            try await processPayment()
            paymentSpan.end()

            span.setAttribute(key: "order.status", value: "completed")

        } catch {
            span.status = .error(description: error.localizedDescription)
            span.setAttribute(key: "error.type", value: String(describing: type(of: error)))
            throw error
        }
    }
}
```

### Available iOS Instrumentations

```swift
// 1. URLSession (HTTP requests)
import URLSessionInstrumentation

// 2. Network Status (connectivity changes)
import NetworkStatus

// 3. SDK Resource Extension (device info)
import SDKResourceExtension

// 4. SignPost Integration (os_signpost bridging)
import SignPostIntegration

// 5. Sessions Event Instrumentation (app lifecycle)
import SessionsEventInstrumentation
```

---

## Android Setup (opentelemetry-android)

### Installation

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        mavenCentral()
    }
}

// app/build.gradle.kts
android {
    compileOptions {
        // Required for coreLibraryDesugaring on API < 26
        isCoreLibraryDesugaringEnabled = true
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
}

dependencies {
    // BOM for version management
    implementation(platform("io.opentelemetry.android:opentelemetry-android-bom:1.0.1"))

    // Core SDK
    implementation("io.opentelemetry.android:instrumentation")

    // Auto-instrumentation modules
    implementation("io.opentelemetry.android:instrumentation-activity")
    implementation("io.opentelemetry.android:instrumentation-anr")
    implementation("io.opentelemetry.android:instrumentation-crash")
    implementation("io.opentelemetry.android:instrumentation-network")
    implementation("io.opentelemetry.android:instrumentation-slowrendering")
    implementation("io.opentelemetry.android:instrumentation-startup")

    // Exporters
    implementation("io.opentelemetry.android:exporter-otlp")

    // Desugaring for API < 26
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.0.4")
}
```

### Basic Configuration

```kotlin
// Application.kt
import android.app.Application
import io.opentelemetry.android.OpenTelemetryRum
import io.opentelemetry.android.config.OtelRumConfig
import io.opentelemetry.api.common.Attributes
import io.opentelemetry.semconv.ServiceAttributes
import io.opentelemetry.semconv.incubating.DeviceIncubatingAttributes

class MyApplication : Application() {

    override fun onCreate() {
        super.onCreate()

        val config = OtelRumConfig.builder(application = this)
            // Resource attributes
            .setResource(
                ResourceBuilder()
                    .put(ServiceAttributes.SERVICE_NAME, "my-android-app")
                    .put(ServiceAttributes.SERVICE_VERSION, BuildConfig.VERSION_NAME)
                    .put("deployment.environment", BuildConfig.BUILD_TYPE)
                    .build()
            )
            // OTLP exporter
            .addTracerProviderCustomizer { tracerProviderBuilder, app ->
                tracerProviderBuilder.addSpanProcessor(
                    BatchSpanProcessor.builder(
                        OtlpHttpSpanExporter.builder()
                            .setEndpoint("https://otlp.yourbackend.com/v1/traces")
                            .addHeader("x-api-key", "YOUR_API_KEY")
                            .build()
                    ).build()
                )
            }
            // Global attributes (added to all spans)
            .addSessionIdChangeListener { sessionId ->
                GlobalAttributesSpanAppender.addAttribute(
                    AttributeKey.stringKey("session.id"),
                    sessionId
                )
            }
            .build()

        OpenTelemetryRum.builder(this, config)
            .build()
    }
}
```

### Auto-Instrumentation

Android SDK provides extensive auto-instrumentation out of the box:

```kotlin
// Application.kt - Enable auto-instrumentation
val otel = OpenTelemetryRum.builder(this, config)
    // Activity & Fragment lifecycle
    .addInstrumentation(ActivityLifecycleInstrumentation())

    // ANR detection (Application Not Responding)
    .addInstrumentation(AnrInstrumentation())

    // Crash reporting
    .addInstrumentation(CrashReporterInstrumentation())

    // Network connectivity changes
    .addInstrumentation(NetworkChangeInstrumentation())

    // Slow/frozen frames
    .addInstrumentation(SlowRenderingInstrumentation())

    // App startup time
    .addInstrumentation(AppStartupInstrumentation())

    // Screen orientation changes
    .addInstrumentation(ScreenOrientationInstrumentation())

    // View clicks
    .addInstrumentation(ViewClickInstrumentation())

    .build()
```

**What gets captured automatically**:

| Instrumentation | Span Name | Attributes |
|-----------------|-----------|------------|
| Activity lifecycle | `AppStart`, `Created`, `Resumed` | `screen.name`, `activity.class` |
| ANR | `ANR` | `thread.name`, `thread.state`, stack trace |
| Crash | `crash` | `exception.type`, `exception.message`, stack trace |
| Network change | `network.change` | `network.type`, `network.subtype` |
| Slow rendering | `slowRender` or `frozenRender` | `frame.count`, `frame.duration` |
| App startup | `AppStart` | `start.type` (cold/warm/hot) |
| View click | `Click` | `view.id`, `view.class` |

### Manual Spans

```kotlin
import io.opentelemetry.api.trace.Span
import io.opentelemetry.api.trace.StatusCode
import io.opentelemetry.context.Context

class CheckoutViewModel(
    private val tracer: Tracer = OpenTelemetry.get()
        .getTracer("com.myapp.checkout", "1.0.0")
) {

    suspend fun processOrder() {
        // Create parent span
        val span = tracer.spanBuilder("process_order")
            .setAttribute("order.items", cartItems.size.toLong())
            .setAttribute("order.total", orderTotal)
            .startSpan()

        try {
            Context.current().with(span).makeCurrent().use {
                // Child span for validation (auto-parented)
                tracer.spanBuilder("validate_order")
                    .startSpan()
                    .use { validationSpan ->
                        validateOrder()
                    }

                // Child span for payment
                tracer.spanBuilder("process_payment")
                    .startSpan()
                    .use { paymentSpan ->
                        processPayment()
                    }

                span.setAttribute("order.status", "completed")
            }
        } catch (e: Exception) {
            span.setStatus(StatusCode.ERROR, e.message ?: "Unknown error")
            span.setAttribute("error.type", e.javaClass.simpleName)
            span.recordException(e)
            throw e
        } finally {
            span.end()
        }
    }
}
```

### Offline Buffering

```kotlin
// Configure disk-based span exporter for offline support
val config = OtelRumConfig.builder(application = this)
    .addTracerProviderCustomizer { tracerProviderBuilder, app ->
        tracerProviderBuilder.addSpanProcessor(
            BatchSpanProcessor.builder(
                DiskBufferingSpanExporter(
                    delegate = OtlpHttpSpanExporter.builder()
                        .setEndpoint("https://otlp.yourbackend.com/v1/traces")
                        .build(),
                    cacheDir = app.cacheDir,
                    maxCacheSizeMb = 50
                )
            )
            .setMaxQueueSize(2048)
            .setMaxExportBatchSize(512)
            .setScheduleDelay(Duration.ofSeconds(5))
            .build()
        )
    }
    .build()
```

---

## When to Use OTel Directly vs Vendor SDKs

### Use OpenTelemetry When:

✅ **Self-hosted backend**: Sending to your own OTLP collector
✅ **Vendor neutrality**: Want to switch backends without code changes
✅ **Custom instrumentation**: Need fine-grained control over spans
✅ **Multi-vendor**: Exporting to multiple backends simultaneously
✅ **Standards compliance**: Team policy requires OpenTelemetry

### Use Vendor SDK When:

✅ **Session replay needed**: Sentry, Datadog, Embrace provide this; OTel doesn't
✅ **Rich integrations**: Vendor SDKs have better framework-specific support
✅ **Error grouping**: Smart error aggregation not available in raw OTel
✅ **Lower setup cost**: Vendor SDKs are plug-and-play
✅ **React Native**: OTel mobile SDKs don't support RN yet

### Hybrid Approach

Many vendors accept OTLP data while still using their SDK for advanced features:

```swift
// iOS - Export OTel spans to Datadog
let datadogExporter = DatadogTraceExporter(
    config: DatadogExporterConfiguration(
        endpoint: URL(string: "https://http-intake.logs.datadoghq.com/api/v2/logs")!,
        clientToken: "YOUR_CLIENT_TOKEN",
        source: "ios"
    )
)

// Use both
// 1. OTel SDK for custom instrumentation
// 2. Datadog SDK for session replay, RUM auto-instrumentation
```

---

## Exporters

### OTLP HTTP

```swift
// iOS
import OtlpHttpTraceExporter

let exporter = OtlpHttpTraceExporter(
    endpoint: URL(string: "https://otlp.yourbackend.com/v1/traces")!,
    config: OtlpConfiguration(
        timeout: TimeInterval(10),
        headers: [
            ("x-api-key", "YOUR_API_KEY"),
            ("x-custom-header", "value")
        ]
    )
)
```

```kotlin
// Android
import io.opentelemetry.exporter.otlp.http.trace.OtlpHttpSpanExporter

val exporter = OtlpHttpSpanExporter.builder()
    .setEndpoint("https://otlp.yourbackend.com/v1/traces")
    .addHeader("x-api-key", "YOUR_API_KEY")
    .setTimeout(Duration.ofSeconds(10))
    .build()
```

### OTLP gRPC

```kotlin
// Android only (iOS SDK doesn't have gRPC exporter yet)
dependencies {
    implementation("io.opentelemetry:opentelemetry-exporter-otlp:1.32.0")
}

val exporter = OtlpGrpcSpanExporter.builder()
    .setEndpoint("https://otlp.yourbackend.com:4317")
    .addHeader("x-api-key", "YOUR_API_KEY")
    .build()
```

### Multi-Exporter Setup

```kotlin
// Android - Export to multiple backends
val config = OtelRumConfig.builder(application = this)
    .addTracerProviderCustomizer { tracerProviderBuilder, app ->
        tracerProviderBuilder
            // Export to self-hosted collector
            .addSpanProcessor(
                BatchSpanProcessor.builder(
                    OtlpHttpSpanExporter.builder()
                        .setEndpoint("https://otel-collector.mycompany.com/v1/traces")
                        .build()
                ).build()
            )
            // Also export to vendor backend
            .addSpanProcessor(
                BatchSpanProcessor.builder(
                    OtlpHttpSpanExporter.builder()
                        .setEndpoint("https://otlp.vendor.com/v1/traces")
                        .addHeader("x-api-key", vendorApiKey)
                        .build()
                ).build()
            )
    }
    .build()
```

### Vendor OTLP Endpoints

| Vendor | OTLP Endpoint | Auth Header |
|--------|---------------|-------------|
| **Datadog** | `https://http-intake.logs.datadoghq.com/api/v2/logs` | `DD-API-KEY: <key>` |
| **Honeycomb** | `https://api.honeycomb.io/v1/traces` | `x-honeycomb-team: <key>` |
| **New Relic** | `https://otlp.nr-data.net/v1/traces` | `api-key: <key>` |
| **Elastic** | `https://<deployment>.apm.<region>.cloud.es.io:443` | `Authorization: Bearer <token>` |
| **Grafana Cloud** | `https://otlp-gateway-<region>.grafana.net/otlp/v1/traces` | `Authorization: Basic <base64>` |

---

## Best Practices

| Practice | Implementation |
|----------|----------------|
| **Sample strategically** | 100% errors, 10-20% normal transactions |
| **Minimize cardinality** | Avoid unique IDs in span names (use attributes) |
| **Batch exports** | 5-30 second delays, 512-2048 batch size |
| **Set resource attributes** | `service.name`, `service.version`, `deployment.environment` |
| **Use semantic conventions** | Follow [OTel semantic conventions](https://opentelemetry.io/docs/specs/semconv/) |
| **End spans in finally** | Ensure spans close even on exceptions |
| **Don't log PII** | Scrub sensitive data before export |
| **Monitor exporter health** | Watch for export failures, buffering limits |

### Span Naming Best Practices

```swift
// ❌ BAD - High cardinality
let span = tracer.spanBuilder(spanName: "fetch_product_\(productId)").startSpan()

// ✅ GOOD - Low cardinality span name, ID in attribute
let span = tracer.spanBuilder(spanName: "fetch_product").startSpan()
span.setAttribute(key: "product.id", value: productId)
```

### Sampling Configuration

```kotlin
// Android - Custom sampler
import io.opentelemetry.sdk.trace.samplers.Sampler

val sampler = Sampler.parentBased(
    Sampler.traceIdRatioBased(0.1)  // 10% base rate
)

// Override for specific operations
val customSampler = object : Sampler {
    override fun shouldSample(
        parentContext: Context,
        traceId: String,
        name: String,
        spanKind: SpanKind,
        attributes: Attributes,
        parentLinks: List<LinkData>
    ): SamplingResult {
        // Always sample errors
        if (attributes.get(AttributeKey.stringKey("error")) != null) {
            return SamplingResult.recordAndSample()
        }

        // Higher sampling for checkout
        if (name.contains("checkout")) {
            return if (Random.nextDouble() < 0.5) {
                SamplingResult.recordAndSample()
            } else {
                SamplingResult.drop()
            }
        }

        // Default 10%
        return if (Random.nextDouble() < 0.1) {
            SamplingResult.recordAndSample()
        } else {
            SamplingResult.drop()
        }
    }
}
```

### Performance Optimization

```swift
// iOS - Limit attribute size
span.setAttribute(key: "http.response_body", value: String(responseBody.prefix(1000)))

// Only set expensive attributes when needed
span.setAttribute(key: "computed.value", value: expensiveComputation())
```

---

## Debugging

### Verify Spans Are Exported

```kotlin
// Android - Enable debug logging
import io.opentelemetry.sdk.trace.export.SimpleSpanProcessor
import io.opentelemetry.exporter.logging.LoggingSpanExporter

// Add logging exporter in debug builds
if (BuildConfig.DEBUG) {
    tracerProviderBuilder.addSpanProcessor(
        SimpleSpanProcessor.create(LoggingSpanExporter.create())
    )
}
```

```swift
// iOS - Print spans before export
class DebugSpanExporter: SpanExporter {
    func export(spans: [SpanData]) -> SpanExporterResultCode {
        spans.forEach { span in
            print("Exporting span: \(span.name)")
            print("Attributes: \(span.attributes)")
        }
        return .success
    }
}
```

### Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| **Spans not appearing** | Exporter endpoint wrong | Verify URL, check network logs |
| **Missing attributes** | Not using semantic conventions | Follow [OTel semconv](https://opentelemetry.io/docs/specs/semconv/) |
| **High memory usage** | Batch size too large | Reduce `maxQueueSize` to 512-1024 |
| **Missing child spans** | Context not propagated | Use `Context.current().with(span).makeCurrent()` |
| **Duplicate spans** | Multiple span processors | Check for duplicate `addSpanProcessor()` calls |

---

## Links

- [OpenTelemetry Swift SDK](https://github.com/open-telemetry/opentelemetry-swift)
- [OpenTelemetry Android SDK](https://github.com/open-telemetry/opentelemetry-android)
- [OTel Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
- [OTLP Specification](https://opentelemetry.io/docs/specs/otlp/)
- [OTel Collector](https://opentelemetry.io/docs/collector/)
