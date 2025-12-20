# Android Native Observability

Instrumentation patterns for native Android applications (Kotlin/Java).

## SDK Setup

### Sentry

**build.gradle.kts (app-level):**

```kotlin
plugins {
    id("io.sentry.android.gradle") version "4.0.0"
}

sentry {
    // Uploads ProGuard/R8 mappings automatically
    includeProguardMapping = true
    autoUploadProguardMapping = true

    // Uploads native symbols (NDK)
    uploadNativeSymbols = true
    includeNativeSources = true

    // Source context
    includeSourceContext = true

    // Performance
    tracingInstrumentation {
        enabled = true
        features = setOf(
            InstrumentationFeature.DATABASE,
            InstrumentationFeature.FILE_IO,
            InstrumentationFeature.OKHTTP,
            InstrumentationFeature.COMPOSE
        )
    }
}
```

**Application class:**

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        SentryAndroid.init(this) { options ->
            options.dsn = BuildConfig.SENTRY_DSN

            // Environment
            options.environment = if (BuildConfig.DEBUG) "development" else "production"

            // Release tracking
            options.release = "${BuildConfig.APPLICATION_ID}@${BuildConfig.VERSION_NAME}+${BuildConfig.VERSION_CODE}"
            options.dist = BuildConfig.VERSION_CODE.toString()

            // Performance sampling
            options.tracesSampleRate = if (BuildConfig.DEBUG) 1.0 else 0.2
            options.profilesSampleRate = 0.1

            // Enable all auto-instrumentation
            options.isEnableAutoActivityLifecycleTracing = true
            options.isEnableActivityLifecycleBreadcrumbs = true
            options.isEnableAppLifecycleBreadcrumbs = true
            options.isEnableSystemEventBreadcrumbs = true
            options.isEnableNetworkEventBreadcrumbs = true
            options.isEnableUserInteractionBreadcrumbs = true

            // Breadcrumbs
            options.maxBreadcrumbs = 100

            // Attachments
            options.isAttachScreenshot = true
            options.isAttachViewHierarchy = true

            // ANR (Application Not Responding) detection
            options.isAnrEnabled = true
            options.anrTimeoutIntervalMillis = 5000

            // Before send
            options.beforeSend = SentryOptions.BeforeSendCallback { event, _ ->
                // Strip PII
                event.user?.let { user ->
                    user.email = null
                    user.ipAddress = null
                }
                event
            }
        }
    }
}
```

**AndroidManifest.xml:**

```xml
<application>
    <!-- Enable auto-init -->
    <meta-data
        android:name="io.sentry.auto-init"
        android:value="true" />

    <!-- DSN (prefer BuildConfig instead) -->
    <meta-data
        android:name="io.sentry.dsn"
        android:value="@string/sentry_dsn" />
</application>
```

### ProGuard/R8 Mapping Upload

Critical for release builds to deobfuscate stack traces.

**sentry.properties:**

```properties
defaults.org=your-org
defaults.project=your-project
auth.token=your-token
```

**Manual upload (CI/CD):**

```bash
sentry-cli upload-proguard \
    --org your-org \
    --project your-project \
    app/build/outputs/mapping/release/mapping.txt
```

### Datadog Alternative

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        val configuration = Configuration.Builder(
            clientToken = "client-token",
            env = "production",
            variant = BuildConfig.BUILD_TYPE
        )
            .useSite(DatadogSite.US1)
            .trackInteractions()
            .trackLongTasks()
            .useViewTrackingStrategy(ActivityViewTrackingStrategy(true))
            .build()

        Datadog.initialize(this, configuration, TrackingConsent.GRANTED)

        val rumConfiguration = RumConfiguration.Builder("app-id")
            .trackUserInteractions()
            .trackLongTasks()
            .useViewTrackingStrategy(ActivityViewTrackingStrategy(true))
            .build()

        Rum.enable(rumConfiguration)
        Logs.enable(LogsConfiguration.Builder().build())
        Trace.enable(TraceConfiguration.Builder().build())
    }
}
```

## Error Capture

### Capturing Exceptions

```kotlin
// Caught exception
try {
    riskyOperation()
} catch (e: Exception) {
    Sentry.captureException(e) { scope ->
        scope.setTag("flow", "payment")
        scope.setContexts("order", mapOf("orderId" to orderId))
    }
}

// With explicit scope
Sentry.withScope { scope ->
    scope.level = SentryLevel.WARNING
    scope.setTag("feature", "checkout")
    Sentry.captureException(PaymentException("Card declined"))
}

// Message (non-error event)
Sentry.captureMessage("User attempted invalid action")
```

### Custom Exception Types

```kotlin
// Create domain-specific exceptions for better grouping
class NetworkException(
    message: String,
    val statusCode: Int? = null,
    val endpoint: String? = null,
    cause: Throwable? = null
) : Exception(message, cause)

class ValidationException(
    message: String,
    val field: String,
    val value: Any?
) : Exception(message)

// Capture with context
Sentry.captureException(
    NetworkException("Request failed", statusCode = 500, endpoint = "/api/users")
) { scope ->
    scope.setTag("error.type", "network")
}
```

### ANR Detection

Automatic with `isAnrEnabled = true`. Customize:

```kotlin
options.anrTimeoutIntervalMillis = 5000  // 5 seconds
options.isAnrReportInDebug = false  // Skip in debug builds
```

## Performance Monitoring

### App Start Measurement

Automatic, but for custom spans:

```kotlin
class MyApplication : Application() {
    private var coldStartSpan: ISpan? = null

    override fun onCreate() {
        super.onCreate()

        // After Sentry init
        coldStartSpan = Sentry.startTransaction(
            "app.start.cold",
            "app.lifecycle",
            TransactionOptions().apply {
                isBindToScope = true
            }
        )

        // Initialization work...
    }

    fun onAppReady() {
        coldStartSpan?.finish(SpanStatus.OK)
        coldStartSpan = null
    }
}
```

### Activity Tracking

Automatic with `isEnableAutoActivityLifecycleTracing`. For custom:

```kotlin
abstract class BaseActivity : AppCompatActivity() {
    private var activitySpan: ISpan? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        activitySpan = Sentry.startTransaction(
            this::class.simpleName ?: "Activity",
            "ui.load"
        )
    }

    override fun onResume() {
        super.onResume()
        activitySpan?.finish(SpanStatus.OK)
    }

    override fun onDestroy() {
        activitySpan?.finish(SpanStatus.CANCELLED)
        super.onDestroy()
    }
}
```

### Fragment Tracking

```kotlin
abstract class BaseFragment : Fragment() {
    private var fragmentSpan: ISpan? = null

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        fragmentSpan = Sentry.getSpan()?.startChild(
            "ui.load.fragment",
            this::class.simpleName ?: "Fragment"
        )
        return super.onCreateView(inflater, container, savedInstanceState)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        fragmentSpan?.finish(SpanStatus.OK)
    }
}
```

### Jetpack Compose Tracking

```kotlin
@Composable
fun TrackedScreen(
    screenName: String,
    content: @Composable () -> Unit
) {
    var span by remember { mutableStateOf<ISpan?>(null) }

    DisposableEffect(screenName) {
        span = Sentry.startTransaction(screenName, "ui.load")
        onDispose {
            span?.finish(SpanStatus.OK)
        }
    }

    content()
}

// Usage
@Composable
fun ProfileScreen() {
    TrackedScreen("ProfileScreen") {
        // Screen content
    }
}
```

### Network Instrumentation (OkHttp)

```kotlin
// Automatic with Sentry OkHttp integration
val okHttpClient = OkHttpClient.Builder()
    .addInterceptor(SentryOkHttpInterceptor())
    .build()

// For Retrofit
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com")
    .client(okHttpClient)
    .build()
```

**Manual instrumentation:**

```kotlin
suspend fun <T> tracedApiCall(
    name: String,
    block: suspend () -> T
): T {
    val span = Sentry.getSpan()?.startChild("http.client", name)

    return try {
        val result = block()
        span?.finish(SpanStatus.OK)
        result
    } catch (e: Exception) {
        span?.apply {
            throwable = e
            finish(SpanStatus.INTERNAL_ERROR)
        }
        throw e
    }
}

// Usage
val users = tracedApiCall("GET /api/users") {
    apiService.getUsers()
}
```

### Database Instrumentation (Room)

Automatic with Sentry plugin. For manual:

```kotlin
suspend fun <T> tracedDbQuery(
    operation: String,
    block: suspend () -> T
): T {
    val span = Sentry.getSpan()?.startChild("db.query", operation)

    return try {
        val result = block()
        span?.finish(SpanStatus.OK)
        result
    } catch (e: Exception) {
        span?.finish(SpanStatus.INTERNAL_ERROR)
        throw e
    }
}

// In DAO
@Query("SELECT * FROM users")
suspend fun getUsers(): List<User>

// Usage
val users = tracedDbQuery("SELECT users") {
    userDao.getUsers()
}
```

## Breadcrumbs

### Automatic Breadcrumbs

Enabled by default:
- Activity/Fragment lifecycle
- System events (low battery, connectivity)
- User interactions (clicks)
- Network events

### Manual Breadcrumbs

```kotlin
// User action
Sentry.addBreadcrumb(Breadcrumb().apply {
    category = "user"
    message = "Tapped checkout button"
    level = SentryLevel.INFO
    setData("cartItems", 3)
})

// Navigation
Sentry.addBreadcrumb(Breadcrumb.navigation(
    from = "HomeScreen",
    to = "ProfileScreen"
))

// State change
Sentry.addBreadcrumb(Breadcrumb().apply {
    category = "state"
    message = "User logged in"
    level = SentryLevel.INFO
    setData("userId", user.id)
    setData("isFirstLogin", user.isFirstLogin)
})

// HTTP request (manual)
Sentry.addBreadcrumb(Breadcrumb.http(
    url = "https://api.example.com/users",
    method = "GET",
    statusCode = 200
))
```

### Lifecycle Breadcrumbs

```kotlin
class LifecycleObserver : DefaultLifecycleObserver {

    override fun onStart(owner: LifecycleOwner) {
        Sentry.addBreadcrumb(Breadcrumb().apply {
            category = "app.lifecycle"
            message = "App started (foreground)"
            level = SentryLevel.INFO
        })
    }

    override fun onStop(owner: LifecycleOwner) {
        Sentry.addBreadcrumb(Breadcrumb().apply {
            category = "app.lifecycle"
            message = "App stopped (background)"
            level = SentryLevel.INFO
        })
    }
}

// Register in Application
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        ProcessLifecycleOwner.get().lifecycle.addObserver(LifecycleObserver())
    }
}
```

## User Context

```kotlin
// On login
fun onUserLogin(user: User) {
    Sentry.setUser(io.sentry.protocol.User().apply {
        id = user.id  // Internal ID only
        username = user.username  // Optional
        data = mapOf(
            "subscription" to user.subscriptionTier,
            "accountAge" to user.accountAgeDays
        )
    })

    Sentry.setTag("user.tier", user.subscriptionTier)
}

// On logout
fun onUserLogout() {
    Sentry.setUser(null)
}
```

## Android-Specific Considerations

### Multi-Process Apps

Each process needs its own Sentry init:

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // Check which process we're in
        val processName = getProcessName()

        SentryAndroid.init(this) { options ->
            options.dsn = BuildConfig.SENTRY_DSN
            options.setTag("process", processName)

            // Different sampling for different processes
            when {
                processName.contains(":sync") -> {
                    options.tracesSampleRate = 0.1
                }
                else -> {
                    options.tracesSampleRate = 0.2
                }
            }
        }
    }
}
```

### Low Memory Handling

```kotlin
class MyApplication : Application(), ComponentCallbacks2 {

    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)

        val levelName = when (level) {
            TRIM_MEMORY_RUNNING_LOW -> "RUNNING_LOW"
            TRIM_MEMORY_RUNNING_CRITICAL -> "RUNNING_CRITICAL"
            TRIM_MEMORY_COMPLETE -> "COMPLETE"
            else -> "LEVEL_$level"
        }

        Sentry.addBreadcrumb(Breadcrumb().apply {
            category = "memory"
            message = "Memory trim: $levelName"
            level = if (level >= TRIM_MEMORY_RUNNING_CRITICAL)
                SentryLevel.WARNING
            else
                SentryLevel.INFO
        })

        // Report if critical
        if (level >= TRIM_MEMORY_RUNNING_CRITICAL) {
            Sentry.captureMessage("Critical memory pressure", SentryLevel.WARNING)
        }
    }
}
```

### Battery Optimization

```kotlin
// Don't spam events when battery is low
class BatteryAwareObservability(context: Context) {

    private val batteryManager = context.getSystemService(Context.BATTERY_SERVICE) as BatteryManager

    fun shouldReduceTracking(): Boolean {
        val batteryLevel = batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
        val isCharging = batteryManager.isCharging

        // Reduce tracking below 20% when not charging
        return batteryLevel < 20 && !isCharging
    }

    fun adjustSampling() {
        val rate = if (shouldReduceTracking()) 0.05 else 0.2
        Sentry.configureScope { scope ->
            scope.setTag("battery.low_power_mode", shouldReduceTracking().toString())
        }
        // Note: Can't dynamically change sample rate, but can skip custom transactions
    }
}
```

### WorkManager Jobs

```kotlin
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val span = Sentry.startTransaction(
            "worker.sync",
            "task.background",
            TransactionOptions().apply { isBindToScope = true }
        )

        return try {
            performSync()
            span.finish(SpanStatus.OK)
            Result.success()
        } catch (e: Exception) {
            span.apply {
                throwable = e
                finish(SpanStatus.INTERNAL_ERROR)
            }
            Sentry.captureException(e)

            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}
```

### Deep Links / App Links

```kotlin
class DeepLinkActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val uri = intent.data

        Sentry.addBreadcrumb(Breadcrumb().apply {
            category = "navigation"
            message = "Deep link opened"
            level = SentryLevel.INFO
            setData("uri", uri?.toString()?.sanitize() ?: "null")
            setData("referrer", referrer?.toString() ?: "direct")
        })

        // Process deep link...
    }
}
```

## Common Android Gotchas

### 1. ProGuard Stripping Sentry

Keep Sentry classes:

```proguard
-keep class io.sentry.** { *; }
-keepnames class io.sentry.** { *; }
```

### 2. Missing Native Crashes

For NDK crashes, enable native symbolication:

```kotlin
sentry {
    uploadNativeSymbols = true
    includeNativeSources = true
}
```

### 3. Startup Delays

Sentry init is fast, but for cold start optimization:

```kotlin
// Async init (lose early crashes)
Thread {
    SentryAndroid.init(this) { options -> ... }
}.start()

// Or use App Startup library for deferred init
```

### 4. Fragment Transaction Crashes

Fragment transactions during lifecycle callbacks cause crashes:

```kotlin
// BAD: Direct navigation in lifecycle
override fun onResume() {
    navigateToProfile()  // May crash
}

// GOOD: Post to handler
override fun onResume() {
    view?.post { navigateToProfile() }
}
```

### 5. Build Variant Confusion

Ensure correct environment tagging:

```kotlin
options.environment = when {
    BuildConfig.DEBUG -> "development"
    BuildConfig.BUILD_TYPE == "staging" -> "staging"
    else -> "production"
}
```

### 6. Large Event Payloads

Android has stricter memory constraints:

```kotlin
options.maxAttachmentSize = 1024 * 1024  // 1MB max
options.maxBreadcrumbs = 100  // Don't exceed
```

## Testing

```kotlin
// Test crash
fun triggerTestCrash() {
    throw RuntimeException("Test crash")
}

// Test native crash
fun triggerNativeCrash() {
    Sentry.nativeCrash()
}

// Test message
fun sendTestEvent() {
    Sentry.captureMessage("Test event from Android")
}

// Verify in Sentry dashboard
```

---

## Reusable Templates

- **[templates/screen-load-tracking.template.md](templates/screen-load-tracking.template.md)** - Android screen load tracking with Systrace
- **[templates/error-boundary.template.md](templates/error-boundary.template.md)** - Android crash handlers and UncaughtExceptionHandler
- **[templates/breadcrumb-manager.template.md](templates/breadcrumb-manager.template.md)** - Android breadcrumb collection
- **[templates/navigation-tracking.template.md](templates/navigation-tracking.template.md)** - Android navigation latency measurement
