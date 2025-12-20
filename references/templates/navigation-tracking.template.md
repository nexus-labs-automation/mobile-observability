# Navigation Tracking Template

Measure tap-to-interactive for every screen transition. Target: <400ms P95.

## Mental Model

```
User taps button          Screen appears           Content loaded
      |                        |                        |
      v                        v                        v
   +------------------------------------------------------+
   |  TAP  |  TRANSITION  |  VIEW INIT  |  DATA LOAD     |
   +------------------------------------------------------+
      |         |              |               |
      +-------------------------------------------+
              Navigation Latency (P95 target: <400ms)
```

---

## iOS

### NavigationLatencyTracker

```swift
// NavigationLatencyTracker.swift
import Foundation
import os.signpost

final class NavigationLatencyTracker {
    static let shared = NavigationLatencyTracker()

    private let signpostLog = OSLog(subsystem: Bundle.main.bundleIdentifier!, category: "Navigation")
    private var pendingNavigations: [String: NavigationMeasurement] = [:]
    private let queue = DispatchQueue(label: "navigation.tracker", qos: .utility)

    struct NavigationMeasurement {
        let sourceScreen: String
        let destinationScreen: String
        let tapTime: CFAbsoluteTime
        let signpostID: OSSignpostID
        var phases: [String: CFAbsoluteTime] = [:]
    }

    /// Call when user initiates navigation (tap, swipe, etc.)
    func onNavigationStart(from source: String, to destination: String) {
        let signpostID = OSSignpostID(log: signpostLog)
        os_signpost(.begin, log: signpostLog, name: "Navigation", signpostID: signpostID,
                    "%{public}s -> %{public}s", source, destination)

        queue.async {
            self.pendingNavigations[destination] = NavigationMeasurement(
                sourceScreen: source,
                destinationScreen: destination,
                tapTime: CFAbsoluteTimeGetCurrent(),
                signpostID: signpostID
            )
        }
    }

    /// Call when view controller/view appears
    func onViewAppear(screen: String) {
        queue.async {
            self.pendingNavigations[screen]?.phases["view_appear"] = CFAbsoluteTimeGetCurrent()
        }
    }

    /// Call when initial UI chrome is visible (before data)
    func onFirstPaint(screen: String) {
        queue.async {
            self.pendingNavigations[screen]?.phases["first_paint"] = CFAbsoluteTimeGetCurrent()
        }
    }

    /// Call when primary content is rendered and interactive
    func onContentReady(screen: String, metadata: [String: Any] = [:]) {
        queue.async {
            guard let measurement = self.pendingNavigations.removeValue(forKey: screen) else { return }

            let endTime = CFAbsoluteTimeGetCurrent()
            let totalLatency = (endTime - measurement.tapTime) * 1000

            os_signpost(.end, log: self.signpostLog, name: "Navigation",
                        signpostID: measurement.signpostID,
                        "latency_ms: %{public}.2f", totalLatency)

            // Calculate phase durations
            var phaseDurations: [String: Double] = [:]
            var lastTime = measurement.tapTime

            let orderedPhases = ["view_appear", "first_paint"]
            for phase in orderedPhases {
                if let phaseTime = measurement.phases[phase] {
                    phaseDurations[phase] = (phaseTime - lastTime) * 1000
                    lastTime = phaseTime
                }
            }
            phaseDurations["content_render"] = (endTime - lastTime) * 1000

            DispatchQueue.main.async {
                self.reportNavigation(
                    from: measurement.sourceScreen,
                    to: measurement.destinationScreen,
                    totalMs: totalLatency,
                    phases: phaseDurations,
                    metadata: metadata
                )
            }
        }
    }

    private func reportNavigation(from: String, to: String, totalMs: Double,
                                   phases: [String: Double], metadata: [String: Any]) {
        Observability.recordMetric(
            name: "navigation.latency",
            value: totalMs,
            unit: .milliseconds,
            tags: [
                "source": from,
                "destination": to,
                "bucket": latencyBucket(totalMs)
            ]
        )

        if totalMs > 1000 {
            Observability.captureMessage(
                "Slow navigation detected",
                level: .warning,
                extras: [
                    "from": from,
                    "to": to,
                    "latency_ms": totalMs,
                    "phases": phases
                ]
            )
        }
    }

    private func latencyBucket(_ ms: Double) -> String {
        switch ms {
        case ..<100: return "fast"
        case ..<300: return "good"
        case ..<600: return "acceptable"
        case ..<1000: return "slow"
        default: return "very_slow"
        }
    }
}
```

### SwiftUI Integration

```swift
struct NavigationTrackingModifier: ViewModifier {
    let screenName: String
    @State private var hasAppeared = false
    @State private var contentReady = false

    func body(content: Content) -> some View {
        content
            .onAppear {
                if !hasAppeared {
                    hasAppeared = true
                    NavigationLatencyTracker.shared.onViewAppear(screen: screenName)
                }
            }
            .background(
                GeometryReader { _ in
                    Color.clear
                        .onAppear {
                            NavigationLatencyTracker.shared.onFirstPaint(screen: screenName)
                        }
                }
            )
    }

    func markContentReady() {
        guard !contentReady else { return }
        contentReady = true
        NavigationLatencyTracker.shared.onContentReady(screen: screenName)
    }
}

extension View {
    func trackNavigation(_ screenName: String) -> some View {
        modifier(NavigationTrackingModifier(screenName: screenName))
    }
}

// Usage with NavigationLink
struct HomeView: View {
    @State private var selectedProduct: Product?

    var body: some View {
        NavigationStack {
            List(products) { product in
                Button {
                    NavigationLatencyTracker.shared.onNavigationStart(
                        from: "Home",
                        to: "ProductDetail"
                    )
                    selectedProduct = product
                } label: {
                    ProductRow(product: product)
                }
            }
            .navigationDestination(item: $selectedProduct) { product in
                ProductDetailView(product: product)
                    .trackNavigation("ProductDetail")
            }
        }
    }
}
```

---

## Android

### NavigationLatencyTracker

```kotlin
// NavigationLatencyTracker.kt
object NavigationLatencyTracker {
    private val pendingNavigations = ConcurrentHashMap<String, NavigationMeasurement>()

    data class NavigationMeasurement(
        val sourceScreen: String,
        val destinationScreen: String,
        val tapTimeNanos: Long = System.nanoTime(),
        val phases: MutableMap<String, Long> = mutableMapOf()
    )

    fun onNavigationStart(from: String, to: String) {
        Trace.beginAsyncSection("Navigation:$from->$to", to.hashCode())
        pendingNavigations[to] = NavigationMeasurement(
            sourceScreen = from,
            destinationScreen = to
        )
    }

    fun onViewCreated(screen: String) {
        pendingNavigations[screen]?.phases?.put("view_created", System.nanoTime())
    }

    fun onViewResumed(screen: String) {
        pendingNavigations[screen]?.phases?.put("view_resumed", System.nanoTime())
    }

    fun onContentReady(screen: String, metadata: Map<String, Any> = emptyMap()) {
        val measurement = pendingNavigations.remove(screen) ?: return
        val endTime = System.nanoTime()

        Trace.endAsyncSection("Navigation:${measurement.sourceScreen}->$screen", screen.hashCode())

        val totalLatencyMs = (endTime - measurement.tapTimeNanos) / 1_000_000.0

        // Calculate phases
        val phases = mutableMapOf<String, Double>()
        var lastTime = measurement.tapTimeNanos

        listOf("view_created", "view_resumed").forEach { phase ->
            measurement.phases[phase]?.let { phaseTime ->
                phases[phase] = (phaseTime - lastTime) / 1_000_000.0
                lastTime = phaseTime
            }
        }
        phases["content_render"] = (endTime - lastTime) / 1_000_000.0

        reportNavigation(
            from = measurement.sourceScreen,
            to = screen,
            totalMs = totalLatencyMs,
            phases = phases,
            metadata = metadata
        )
    }

    private fun reportNavigation(
        from: String,
        to: String,
        totalMs: Double,
        phases: Map<String, Double>,
        metadata: Map<String, Any>
    ) {
        Observability.recordMetric(
            name = "navigation.latency",
            value = totalMs,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf(
                "source" to from,
                "destination" to to,
                "bucket" to latencyBucket(totalMs)
            )
        )

        if (totalMs > 1000) {
            Observability.captureMessage(
                message = "Slow navigation detected",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "from" to from,
                    "to" to to,
                    "latency_ms" to totalMs,
                    "phases" to phases
                )
            )
        }
    }

    private fun latencyBucket(ms: Double): String = when {
        ms < 100 -> "fast"
        ms < 300 -> "good"
        ms < 600 -> "acceptable"
        ms < 1000 -> "slow"
        else -> "very_slow"
    }
}
```

### Compose Integration

```kotlin
@Composable
fun <T> TrackedNavHost(
    navController: NavHostController,
    startDestination: String,
    modifier: Modifier = Modifier,
    builder: NavGraphBuilder.() -> Unit
) {
    var previousDestination by remember { mutableStateOf<String?>(null) }

    DisposableEffect(navController) {
        val listener = NavController.OnDestinationChangedListener { _, destination, _ ->
            val route = destination.route ?: return@OnDestinationChangedListener
            previousDestination?.let { from ->
                NavigationLatencyTracker.onNavigationStart(from = from, to = route)
            }
            previousDestination = route
        }
        navController.addOnDestinationChangedListener(listener)
        onDispose { navController.removeOnDestinationChangedListener(listener) }
    }

    NavHost(
        navController = navController,
        startDestination = startDestination,
        modifier = modifier,
        builder = builder
    )
}

@Composable
fun TrackScreenContent(
    screenName: String,
    isContentReady: Boolean,
    metadata: Map<String, Any> = emptyMap(),
    content: @Composable () -> Unit
) {
    var firstPaintReported by remember { mutableStateOf(false) }
    var contentReadyReported by remember { mutableStateOf(false) }

    LaunchedEffect(Unit) {
        NavigationLatencyTracker.onViewCreated(screenName)
    }

    Box(
        modifier = Modifier.drawWithContent {
            drawContent()
            if (!firstPaintReported) {
                firstPaintReported = true
                NavigationLatencyTracker.onViewResumed(screenName)
            }
        }
    ) {
        content()
    }

    LaunchedEffect(isContentReady) {
        if (isContentReady && !contentReadyReported) {
            contentReadyReported = true
            NavigationLatencyTracker.onContentReady(screenName, metadata)
        }
    }
}
```

---

## React Native

### Navigation Instrumentation

```typescript
// navigation/index.tsx
import * as Sentry from '@sentry/react-native';
import { NavigationContainer } from '@react-navigation/native';

const routingInstrumentation = new Sentry.ReactNavigationInstrumentation();

// In Sentry.init()
Sentry.init({
  integrations: [
    new Sentry.ReactNativeTracing({
      routingInstrumentation,
      tracingOrigins: ['localhost', 'api.yourapp.com', /^\//],
    }),
  ],
});

// In NavigationContainer
export function Navigation() {
  const navigation = useNavigationContainerRef();

  return (
    <NavigationContainer
      ref={navigation}
      onReady={() => {
        routingInstrumentation.registerNavigationContainer(navigation);
      }}
    >
      {/* screens */}
    </NavigationContainer>
  );
}
```

### Expo Router Integration

```typescript
// app/_layout.tsx
import * as Sentry from '@sentry/react-native';
import { useNavigationContainerRef, Slot } from 'expo-router';

const routingInstrumentation = new Sentry.ReactNavigationInstrumentation();

export default function RootLayout() {
  const ref = useNavigationContainerRef();

  useEffect(() => {
    if (ref) {
      routingInstrumentation.registerNavigationContainer(ref);
    }
  }, [ref]);

  return <Slot />;
}
```

---

## Time to Interactive (TTI)

### TTI Tracker Pattern

```swift
// iOS TTITracker.swift
final class TTITracker {
    private let screenName: String
    private let startTime: CFAbsoluteTime
    private var milestones: [String: CFAbsoluteTime] = [:]
    private var isComplete = false

    init(screenName: String) {
        self.screenName = screenName
        self.startTime = CFAbsoluteTimeGetCurrent()
    }

    func markFirstPaint() {
        guard milestones["first_paint"] == nil else { return }
        milestones["first_paint"] = CFAbsoluteTimeGetCurrent()
    }

    func markContentfulPaint() {
        guard milestones["contentful_paint"] == nil else { return }
        milestones["contentful_paint"] = CFAbsoluteTimeGetCurrent()
    }

    func markInteractive() {
        guard milestones["interactive"] == nil else { return }
        milestones["interactive"] = CFAbsoluteTimeGetCurrent()
        reportTTI()
    }

    private func reportTTI() {
        guard !isComplete, let interactive = milestones["interactive"] else { return }
        isComplete = true

        let tti = (interactive - startTime) * 1000

        Observability.recordMetric(
            name: "screen.tti",
            value: tti,
            unit: .milliseconds,
            tags: [
                "screen": screenName,
                "bucket": ttiBucket(tti)
            ]
        )
    }

    private func ttiBucket(_ ms: Double) -> String {
        switch ms {
        case ..<1000: return "fast"
        case ..<2000: return "good"
        case ..<3000: return "acceptable"
        case ..<5000: return "slow"
        default: return "very_slow"
        }
    }
}
```

---

## Key Metrics

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| Navigation latency P50 | <200ms | >400ms |
| Navigation latency P95 | <400ms | >800ms |
| TTI P95 | <2s | >3s |
| Slow navigation rate | <5% | >10% |

---

## Integration Points

- **[ui-performance.md](../ui-performance.md)** - Full UI performance context
- **[screen-load-tracking.template.md](screen-load-tracking.template.md)** - Screen load measurement
- **[performance.md](../performance.md)** - App start and general performance
