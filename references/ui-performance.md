# UI Performance Observability

Measuring what users actually see: navigation latency, render performance, scroll smoothness, animations.

## Table of Contents
1. [Navigation Latency](#navigation-latency)
2. [SwiftUI Observability](#swiftui-observability)
3. [Jetpack Compose Observability](#jetpack-compose-observability)
4. [Scroll Performance](#scroll-performance)
5. [Animation Performance](#animation-performance)
6. [Time to Interactive](#time-to-interactive)
7. [Entry Point Latency](#entry-point-latency)
8. [Image Loading](#image-loading)

---

## Navigation Latency

The time from user tap to fully rendered, interactive destination screen.

### Mental Model

```
User taps button          Screen appears           Content loaded
      │                        │                        │
      ▼                        ▼                        ▼
   ┌──────────────────────────────────────────────────────┐
   │  TAP  │  TRANSITION  │  VIEW INIT  │  DATA LOAD     │
   └──────────────────────────────────────────────────────┘
      │         │              │               │
      └─────────┴──────────────┴───────────────┘
              Navigation Latency (P95 target: <400ms)
```

### iOS Navigation Measurement

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

    // MARK: - Lifecycle Methods

    /// Call when user initiates navigation (tap, swipe, etc.)
    func onNavigationStart(from source: String, to destination: String) {
        let signpostID = OSSignpostID(log: signpostLog)
        os_signpost(.begin, log: signpostLog, name: "Navigation", signpostID: signpostID,
                    "%{public}s → %{public}s", source, destination)

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
            let totalLatency = (endTime - measurement.tapTime) * 1000 // Convert to ms

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

            // Report to observability backend
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
        // Sentry span
        let span = SentrySDK.span?.startChild(
            operation: "ui.navigation",
            description: "\(from) → \(to)"
        )
        span?.setData(value: totalMs, key: "duration_ms")
        phases.forEach { span?.setData(value: $0.value, key: "phase.\($0.key)_ms") }
        span?.finish()

        // Custom metric
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

        // Alert on slow navigations
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

### SwiftUI Navigation Integration

```swift
// NavigationModifiers.swift
import SwiftUI

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
                            // First layout pass complete
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

struct ProductDetailView: View {
    let product: Product
    @State private var details: ProductDetails?
    @State private var isLoading = true

    var body: some View {
        Group {
            if isLoading {
                ProgressView()
            } else if let details {
                ProductContent(product: product, details: details)
                    .onAppear {
                        NavigationLatencyTracker.shared.onContentReady(
                            screen: "ProductDetail",
                            metadata: ["product_id": product.id]
                        )
                    }
            }
        }
        .task {
            details = await ProductService.fetchDetails(for: product.id)
            isLoading = false
        }
    }
}
```

### Android Navigation Measurement

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
        Trace.beginAsyncSection("Navigation:$from→$to", to.hashCode())
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

        Trace.endAsyncSection("Navigation:${measurement.sourceScreen}→$screen", screen.hashCode())

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

        // Report
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
        // Sentry transaction
        val transaction = Sentry.startTransaction(
            "navigation",
            "ui.navigation",
            TransactionOptions().apply { isBindToScope = true }
        )
        transaction.setTag("source", from)
        transaction.setTag("destination", to)
        transaction.setData("duration_ms", totalMs)
        phases.forEach { (phase, duration) ->
            transaction.setData("phase.${phase}_ms", duration)
        }
        transaction.finish()

        // Custom metric
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
    }

    private fun latencyBucket(ms: Double): String = when {
        ms < 100 -> "fast"
        ms < 300 -> "good"
        ms < 600 -> "acceptable"
        ms < 1000 -> "slow"
        else -> "very_slow"
    }
}

// Fragment extension
abstract class TrackedFragment(@LayoutRes layoutId: Int) : Fragment(layoutId) {
    abstract val screenName: String

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        NavigationLatencyTracker.onViewCreated(screenName)
    }

    override fun onResume() {
        super.onResume()
        NavigationLatencyTracker.onViewResumed(screenName)
    }

    protected fun markContentReady(metadata: Map<String, Any> = emptyMap()) {
        NavigationLatencyTracker.onContentReady(screenName, metadata)
    }
}
```

### Jetpack Compose Navigation Integration

```kotlin
// ComposeNavigationTracking.kt
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

// Usage
@Composable
fun ProductDetailScreen(productId: String, viewModel: ProductViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    TrackScreenContent(
        screenName = "ProductDetail",
        isContentReady = uiState is ProductUiState.Success,
        metadata = mapOf("product_id" to productId)
    ) {
        when (val state = uiState) {
            is ProductUiState.Loading -> LoadingIndicator()
            is ProductUiState.Success -> ProductContent(state.product)
            is ProductUiState.Error -> ErrorView(state.message)
        }
    }
}
```

---

## SwiftUI Observability

SwiftUI's declarative nature creates unique observability challenges: view body re-evaluations, diffing, state propagation.

### The Problem

SwiftUI re-evaluates `body` when:
- `@State` changes
- `@Binding` changes
- `@ObservedObject` / `@StateObject` publishes
- `@Observable` properties change
- Parent view re-evaluates
- Environment values change

**You can't see this happening** without instrumentation.

### Body Evaluation Tracking

```swift
// BodyEvaluationTracker.swift
import os.signpost

enum ViewRenderTracker {
    private static let signpostLog = OSLog(subsystem: Bundle.main.bundleIdentifier!, category: "SwiftUIRender")
    private static var evaluationCounts: [String: Int] = [:]
    private static let queue = DispatchQueue(label: "render.tracker")

    static func trackBodyEvaluation(_ viewName: String, file: String = #file, line: Int = #line) {
        let signpostID = OSSignpostID(log: signpostLog)
        os_signpost(.event, log: signpostLog, name: "BodyEvaluation", signpostID: signpostID,
                    "view: %{public}s, file: %{public}s, line: %{public}d",
                    viewName, file, line)

        queue.async {
            evaluationCounts[viewName, default: 0] += 1
        }

        #if DEBUG
        print("🔄 [\(viewName)] body evaluated (file: \(file), line: \(line))")
        #endif
    }

    static func getEvaluationCount(for viewName: String) -> Int {
        queue.sync { evaluationCounts[viewName] ?? 0 }
    }

    static func reportAndReset() {
        queue.async {
            let counts = evaluationCounts
            evaluationCounts.removeAll()

            // Report excessive re-renders
            for (view, count) in counts where count > 10 {
                Observability.captureMessage(
                    "Excessive view re-renders",
                    level: .warning,
                    extras: [
                        "view": view,
                        "evaluation_count": count,
                        "period": "1 second"
                    ]
                )
            }

            // Aggregate metric
            let totalEvaluations = counts.values.reduce(0, +)
            Observability.recordMetric(
                name: "swiftui.body_evaluations",
                value: Double(totalEvaluations),
                tags: ["period": "1s"]
            )
        }
    }
}

// View extension for easy tracking
extension View {
    func trackRender(_ name: String = #function) -> some View {
        ViewRenderTracker.trackBodyEvaluation(name)
        return self
    }
}

// Usage
struct ProductCard: View {
    let product: Product

    var body: some View {
        let _ = Self._printChanges() // Debug: prints what triggered re-render
        let _ = trackRender("ProductCard")

        VStack {
            // ... content
        }
    }
}
```

### Render Performance Modifier

```swift
// RenderPerformanceModifier.swift
struct RenderPerformanceModifier: ViewModifier {
    let viewName: String
    @State private var renderCount = 0
    @State private var lastRenderTime: CFAbsoluteTime = 0

    func body(content: Content) -> some View {
        let startTime = CFAbsoluteTimeGetCurrent()

        return content
            .background(
                GeometryReader { _ in
                    Color.clear
                        .onAppear {
                            let renderTime = (CFAbsoluteTimeGetCurrent() - startTime) * 1000
                            renderCount += 1

                            // Track render time
                            if renderTime > 16.67 { // Slower than 60fps
                                Observability.recordMetric(
                                    name: "swiftui.slow_render",
                                    value: renderTime,
                                    unit: .milliseconds,
                                    tags: ["view": viewName]
                                )
                            }

                            // Detect render storms (>5 renders in 100ms)
                            let now = CFAbsoluteTimeGetCurrent()
                            if now - lastRenderTime < 0.1 && renderCount > 5 {
                                Observability.captureMessage(
                                    "Render storm detected",
                                    level: .warning,
                                    extras: [
                                        "view": viewName,
                                        "renders_in_100ms": renderCount
                                    ]
                                )
                            }
                            lastRenderTime = now
                        }
                }
            )
    }
}

extension View {
    func measureRender(_ name: String) -> some View {
        modifier(RenderPerformanceModifier(viewName: name))
    }
}
```

### @Observable vs @ObservedObject Impact

```swift
// Demonstrating observation scope differences

// ❌ BAD: Entire view re-renders on ANY property change
class ProductViewModel: ObservableObject {
    @Published var name: String = ""
    @Published var price: Double = 0
    @Published var description: String = ""
    @Published var reviews: [Review] = []
    @Published var relatedProducts: [Product] = []
}

struct ProductView: View {
    @ObservedObject var viewModel: ProductViewModel

    var body: some View {
        // This ENTIRE body re-evaluates when ANY @Published changes
        VStack {
            Text(viewModel.name)      // Only needs name
            Text("$\(viewModel.price)") // Only needs price
            ReviewList(reviews: viewModel.reviews) // Only needs reviews
        }
    }
}

// ✅ BETTER: iOS 17+ @Observable with granular tracking
@Observable
class ProductViewModel {
    var name: String = ""
    var price: Double = 0
    var description: String = ""
    var reviews: [Review] = []
    var relatedProducts: [Product] = []
}

struct ProductView: View {
    var viewModel: ProductViewModel

    var body: some View {
        // Only re-evaluates when accessed properties change
        VStack {
            NameView(name: viewModel.name)           // Only re-renders on name change
            PriceView(price: viewModel.price)        // Only re-renders on price change
            ReviewList(reviews: viewModel.reviews)   // Only re-renders on reviews change
        }
    }
}

// Observation tracking for @Observable
@Observable
class TrackedProductViewModel {
    var name: String = "" {
        didSet {
            ObservationTracker.recordPropertyAccess(
                type: "ProductViewModel",
                property: "name",
                trigger: "write"
            )
        }
    }
    // ... similar for other properties
}

enum ObservationTracker {
    private static var accessLog: [(type: String, property: String, trigger: String, time: Date)] = []

    static func recordPropertyAccess(type: String, property: String, trigger: String) {
        accessLog.append((type, property, trigger, Date()))

        #if DEBUG
        print("📊 [\(type).\(property)] \(trigger)")
        #endif
    }

    static func analyzeAccessPatterns() -> [String: Int] {
        // Group by type.property to find hot properties
        Dictionary(grouping: accessLog) { "\($0.type).\($0.property)" }
            .mapValues { $0.count }
    }
}
```

### State Change Propagation Visualization

```swift
// StateFlowDebugger.swift
// Helps visualize how state changes propagate through view hierarchy

struct StateFlowDebugModifier: ViewModifier {
    let viewName: String
    let depth: Int

    @Environment(\.stateFlowDebugEnabled) var debugEnabled

    func body(content: Content) -> some View {
        if debugEnabled {
            content
                .overlay(alignment: .topLeading) {
                    Text(viewName)
                        .font(.caption2)
                        .padding(2)
                        .background(Color.blue.opacity(0.3))
                }
                .border(Color.blue.opacity(0.3))
                .onAppear {
                    print(String(repeating: "  ", count: depth) + "↳ \(viewName)")
                }
        } else {
            content
        }
    }
}

private struct StateFlowDebugEnabledKey: EnvironmentKey {
    static let defaultValue = false
}

extension EnvironmentValues {
    var stateFlowDebugEnabled: Bool {
        get { self[StateFlowDebugEnabledKey.self] }
        set { self[StateFlowDebugEnabledKey.self] = newValue }
    }
}

extension View {
    func debugStateFlow(_ name: String, depth: Int = 0) -> some View {
        modifier(StateFlowDebugModifier(viewName: name, depth: depth))
    }
}
```

### Diffing Overhead Detection

```swift
// Detect when SwiftUI diffing is expensive

struct DiffingPerformanceModifier<ID: Hashable>: ViewModifier {
    let items: [ID]
    let viewName: String

    @State private var previousItems: [ID] = []
    @State private var lastDiffTime: CFAbsoluteTime = 0

    func body(content: Content) -> some View {
        content
            .onChange(of: items) { oldValue, newValue in
                let startTime = CFAbsoluteTimeGetCurrent()

                // Measure diff cost
                let added = Set(newValue).subtracting(Set(oldValue)).count
                let removed = Set(oldValue).subtracting(Set(newValue)).count
                let total = newValue.count

                let diffTime = (CFAbsoluteTimeGetCurrent() - startTime) * 1000

                if diffTime > 5 || added + removed > 50 {
                    Observability.recordMetric(
                        name: "swiftui.diff_overhead",
                        value: diffTime,
                        unit: .milliseconds,
                        tags: [
                            "view": viewName,
                            "items_added": String(added),
                            "items_removed": String(removed),
                            "total_items": String(total)
                        ]
                    )
                }
            }
    }
}

extension View {
    func trackDiffing<ID: Hashable>(items: [ID], name: String) -> some View {
        modifier(DiffingPerformanceModifier(items: items, viewName: name))
    }
}

// Usage
struct ProductListView: View {
    @State private var products: [Product] = []

    var body: some View {
        List(products) { product in
            ProductRow(product: product)
        }
        .trackDiffing(items: products.map(\.id), name: "ProductList")
    }
}
```

---

## Jetpack Compose Observability

Compose's recomposition model has similar challenges to SwiftUI.

### Recomposition Tracking

```kotlin
// RecompositionTracker.kt
import androidx.compose.runtime.*
import android.util.Log

object RecompositionTracker {
    private val recompositionCounts = mutableMapOf<String, Int>()
    private var lastReportTime = System.currentTimeMillis()

    fun trackRecomposition(composableName: String) {
        recompositionCounts[composableName] = (recompositionCounts[composableName] ?: 0) + 1

        if (BuildConfig.DEBUG) {
            Log.d("Recomposition", "🔄 $composableName recomposed")
        }

        // Report every second
        val now = System.currentTimeMillis()
        if (now - lastReportTime > 1000) {
            reportAndReset()
            lastReportTime = now
        }
    }

    private fun reportAndReset() {
        val counts = recompositionCounts.toMap()
        recompositionCounts.clear()

        // Find problematic composables
        counts.filter { it.value > 10 }.forEach { (name, count) ->
            Observability.captureMessage(
                message = "Excessive recomposition",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "composable" to name,
                    "count" to count,
                    "period" to "1 second"
                )
            )
        }

        // Aggregate metric
        Observability.recordMetric(
            name = "compose.recompositions",
            value = counts.values.sum().toDouble(),
            tags = mapOf("period" to "1s")
        )
    }
}

// Composable helper
@Composable
inline fun <T> TrackRecomposition(
    name: String,
    crossinline content: @Composable () -> T
): T {
    SideEffect {
        RecompositionTracker.trackRecomposition(name)
    }
    return content()
}

// Usage
@Composable
fun ProductCard(product: Product) {
    TrackRecomposition("ProductCard") {
        Card {
            // content
        }
    }
}
```

### Stability Analysis

```kotlin
// Compose compiler reports help identify unstable classes
// build.gradle.kts
// composeCompiler {
//     reportsDestination = layout.buildDirectory.dir("compose_reports")
//     metricsDestination = layout.buildDirectory.dir("compose_metrics")
// }

// Runtime stability checker
@Composable
fun <T> StabilityDebugger(
    value: T,
    name: String,
    content: @Composable () -> Unit
) {
    val previousValue = remember { mutableStateOf<T?>(null) }

    LaunchedEffect(value) {
        if (previousValue.value != null && previousValue.value != value) {
            // Value changed, check if it's the same content
            val prev = previousValue.value
            val structurallyEqual = prev == value
            val referentiallyEqual = prev === value

            if (structurallyEqual && !referentiallyEqual) {
                Log.w("Stability", "⚠️ $name: Structurally equal but different reference - consider making stable")

                Observability.recordMetric(
                    name = "compose.unstable_parameter",
                    value = 1.0,
                    tags = mapOf("parameter" to name)
                )
            }
        }
        previousValue.value = value
    }

    content()
}

// Mark classes as stable for Compose
@Stable
data class ProductUiModel(
    val id: String,
    val name: String,
    val price: Double,
    val imageUrl: String
)

// Or use @Immutable for truly immutable data
@Immutable
data class CategoryInfo(
    val id: String,
    val name: String,
    val iconRes: Int
)
```

### Composition Performance Measurement

```kotlin
// CompositionPerformance.kt
@Composable
fun MeasuredComposition(
    name: String,
    threshold: Long = 16L, // ms, targeting 60fps
    content: @Composable () -> Unit
) {
    var compositionStartTime by remember { mutableStateOf(0L) }

    SideEffect {
        compositionStartTime = System.nanoTime()
    }

    content()

    DisposableEffect(Unit) {
        val compositionTime = (System.nanoTime() - compositionStartTime) / 1_000_000

        if (compositionTime > threshold) {
            Observability.recordMetric(
                name = "compose.slow_composition",
                value = compositionTime.toDouble(),
                unit = MetricUnit.MILLISECONDS,
                tags = mapOf(
                    "composable" to name,
                    "threshold_ms" to threshold.toString()
                )
            )
        }

        onDispose { }
    }
}

// Layout measurement
@Composable
fun MeasuredLayout(
    name: String,
    modifier: Modifier = Modifier,
    content: @Composable () -> Unit
) {
    var layoutStartTime = 0L

    Layout(
        content = content,
        modifier = modifier,
        measurePolicy = { measurables, constraints ->
            layoutStartTime = System.nanoTime()

            val placeables = measurables.map { it.measure(constraints) }
            val width = placeables.maxOfOrNull { it.width } ?: 0
            val height = placeables.sumOf { it.height }

            val layoutTime = (System.nanoTime() - layoutStartTime) / 1_000_000

            if (layoutTime > 8) { // Half a frame
                Observability.recordMetric(
                    name = "compose.slow_layout",
                    value = layoutTime.toDouble(),
                    unit = MetricUnit.MILLISECONDS,
                    tags = mapOf("composable" to name)
                )
            }

            layout(width, height) {
                var y = 0
                placeables.forEach { placeable ->
                    placeable.place(0, y)
                    y += placeable.height
                }
            }
        }
    )
}
```

### LazyList Performance

```kotlin
// LazyListPerformance.kt
@Composable
fun <T> TrackedLazyColumn(
    items: List<T>,
    listName: String,
    modifier: Modifier = Modifier,
    key: ((T) -> Any)? = null,
    contentType: ((T) -> Any?)? = null,
    itemContent: @Composable LazyItemScope.(T) -> Unit
) {
    val listState = rememberLazyListState()
    var visibleItemsCount by remember { mutableStateOf(0) }
    var firstVisibleIndex by remember { mutableStateOf(0) }

    LaunchedEffect(listState) {
        snapshotFlow { listState.layoutInfo }
            .collect { layoutInfo ->
                val newVisibleCount = layoutInfo.visibleItemsInfo.size
                val newFirstVisible = layoutInfo.visibleItemsInfo.firstOrNull()?.index ?: 0

                // Detect rapid scrolling (potential jank)
                if (abs(newFirstVisible - firstVisibleIndex) > 10) {
                    Observability.recordMetric(
                        name = "compose.rapid_scroll",
                        value = abs(newFirstVisible - firstVisibleIndex).toDouble(),
                        tags = mapOf("list" to listName)
                    )
                }

                visibleItemsCount = newVisibleCount
                firstVisibleIndex = newFirstVisible
            }
    }

    LazyColumn(
        state = listState,
        modifier = modifier
    ) {
        items(
            count = items.size,
            key = key?.let { k -> { index -> k(items[index]) } },
            contentType = contentType?.let { ct -> { index -> ct(items[index]) } }
        ) { index ->
            val item = items[index]
            MeasuredComposition("$listName.item[$index]") {
                itemContent(item)
            }
        }
    }
}
```

---

## Scroll Performance

Detecting jank, dropped frames, and scroll hitches.

### iOS Scroll Performance

```swift
// ScrollPerformanceTracker.swift
import QuartzCore

final class ScrollPerformanceTracker {
    static let shared = ScrollPerformanceTracker()

    private var displayLink: CADisplayLink?
    private var lastFrameTimestamp: CFTimeInterval = 0
    private var frameDrops: [FrameDrop] = []
    private var isTracking = false
    private var scrollViewName: String = ""

    struct FrameDrop {
        let timestamp: Date
        let droppedFrames: Int
        let duration: TimeInterval
    }

    func startTracking(scrollViewName: String) {
        guard !isTracking else { return }

        self.scrollViewName = scrollViewName
        isTracking = true
        frameDrops.removeAll()

        displayLink = CADisplayLink(target: self, selector: #selector(frameCallback))
        displayLink?.add(to: .main, forMode: .common)
        lastFrameTimestamp = CACurrentMediaTime()
    }

    func stopTracking() -> ScrollPerformanceReport {
        displayLink?.invalidate()
        displayLink = nil
        isTracking = false

        return generateReport()
    }

    @objc private func frameCallback(_ link: CADisplayLink) {
        let currentTime = link.timestamp
        let frameDuration = currentTime - lastFrameTimestamp
        let expectedFrameDuration = link.targetTimestamp - link.timestamp

        // Detect dropped frames (> 1.5x expected frame time)
        if frameDuration > expectedFrameDuration * 1.5 {
            let droppedFrames = Int(frameDuration / expectedFrameDuration) - 1
            frameDrops.append(FrameDrop(
                timestamp: Date(),
                droppedFrames: droppedFrames,
                duration: frameDuration
            ))
        }

        lastFrameTimestamp = currentTime
    }

    private func generateReport() -> ScrollPerformanceReport {
        let totalDroppedFrames = frameDrops.reduce(0) { $0 + $1.droppedFrames }
        let maxConsecutiveDrops = frameDrops.map(\.droppedFrames).max() ?? 0

        let report = ScrollPerformanceReport(
            scrollViewName: scrollViewName,
            totalDroppedFrames: totalDroppedFrames,
            maxConsecutiveDrops: maxConsecutiveDrops,
            dropEvents: frameDrops.count,
            jankScore: calculateJankScore(totalDroppedFrames, maxConsecutiveDrops)
        )

        // Report to observability
        Observability.recordMetric(
            name: "scroll.dropped_frames",
            value: Double(totalDroppedFrames),
            tags: ["scroll_view": scrollViewName]
        )

        if maxConsecutiveDrops > 3 {
            Observability.captureMessage(
                "Scroll jank detected",
                level: .warning,
                extras: [
                    "scroll_view": scrollViewName,
                    "max_consecutive_drops": maxConsecutiveDrops,
                    "total_dropped": totalDroppedFrames
                ]
            )
        }

        return report
    }

    private func calculateJankScore(_ totalDropped: Int, _ maxConsecutive: Int) -> Double {
        // 0-100 score, lower is better
        let dropPenalty = min(Double(totalDropped) * 2, 50)
        let consecutivePenalty = min(Double(maxConsecutive) * 10, 50)
        return dropPenalty + consecutivePenalty
    }
}

struct ScrollPerformanceReport {
    let scrollViewName: String
    let totalDroppedFrames: Int
    let maxConsecutiveDrops: Int
    let dropEvents: Int
    let jankScore: Double

    var isSmooth: Bool { jankScore < 20 }
}

// SwiftUI integration
struct ScrollPerformanceModifier: ViewModifier {
    let name: String
    @State private var tracker = ScrollPerformanceTracker()

    func body(content: Content) -> some View {
        content
            .onAppear {
                ScrollPerformanceTracker.shared.startTracking(scrollViewName: name)
            }
            .onDisappear {
                let report = ScrollPerformanceTracker.shared.stopTracking()
                print("📊 Scroll report for \(name): \(report)")
            }
    }
}

extension View {
    func trackScrollPerformance(_ name: String) -> some View {
        modifier(ScrollPerformanceModifier(name: name))
    }
}
```

### Android Scroll Performance

```kotlin
// ScrollPerformanceTracker.kt
class ScrollPerformanceTracker(private val scrollViewName: String) {
    private var choreographer: Choreographer? = null
    private var lastFrameTimeNanos: Long = 0
    private var frameDrops = mutableListOf<FrameDrop>()
    private var isTracking = false

    data class FrameDrop(
        val timestamp: Long,
        val droppedFrames: Int,
        val durationMs: Double
    )

    private val frameCallback = object : Choreographer.FrameCallback {
        override fun doFrame(frameTimeNanos: Long) {
            if (!isTracking) return

            if (lastFrameTimeNanos != 0L) {
                val frameDurationNanos = frameTimeNanos - lastFrameTimeNanos
                val frameDurationMs = frameDurationNanos / 1_000_000.0
                val expectedFrameMs = 16.67 // 60fps

                if (frameDurationMs > expectedFrameMs * 1.5) {
                    val droppedFrames = (frameDurationMs / expectedFrameMs).toInt() - 1
                    frameDrops.add(FrameDrop(
                        timestamp = System.currentTimeMillis(),
                        droppedFrames = droppedFrames,
                        durationMs = frameDurationMs
                    ))
                }
            }

            lastFrameTimeNanos = frameTimeNanos
            choreographer?.postFrameCallback(this)
        }
    }

    fun startTracking() {
        if (isTracking) return
        isTracking = true
        frameDrops.clear()
        lastFrameTimeNanos = 0
        choreographer = Choreographer.getInstance()
        choreographer?.postFrameCallback(frameCallback)
    }

    fun stopTracking(): ScrollPerformanceReport {
        isTracking = false
        choreographer?.removeFrameCallback(frameCallback)

        return generateReport()
    }

    private fun generateReport(): ScrollPerformanceReport {
        val totalDroppedFrames = frameDrops.sumOf { it.droppedFrames }
        val maxConsecutiveDrops = frameDrops.maxOfOrNull { it.droppedFrames } ?: 0

        // Report metrics
        Observability.recordMetric(
            name = "scroll.dropped_frames",
            value = totalDroppedFrames.toDouble(),
            tags = mapOf("scroll_view" to scrollViewName)
        )

        if (maxConsecutiveDrops > 3) {
            Observability.captureMessage(
                message = "Scroll jank detected",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "scroll_view" to scrollViewName,
                    "max_consecutive_drops" to maxConsecutiveDrops,
                    "total_dropped" to totalDroppedFrames
                )
            )
        }

        return ScrollPerformanceReport(
            scrollViewName = scrollViewName,
            totalDroppedFrames = totalDroppedFrames,
            maxConsecutiveDrops = maxConsecutiveDrops,
            dropEvents = frameDrops.size
        )
    }
}

data class ScrollPerformanceReport(
    val scrollViewName: String,
    val totalDroppedFrames: Int,
    val maxConsecutiveDrops: Int,
    val dropEvents: Int
) {
    val jankScore: Double
        get() {
            val dropPenalty = minOf(totalDroppedFrames * 2.0, 50.0)
            val consecutivePenalty = minOf(maxConsecutiveDrops * 10.0, 50.0)
            return dropPenalty + consecutivePenalty
        }

    val isSmooth: Boolean get() = jankScore < 20
}

// Compose integration
@Composable
fun TrackedScrollColumn(
    name: String,
    modifier: Modifier = Modifier,
    content: @Composable ColumnScope.() -> Unit
) {
    val tracker = remember { ScrollPerformanceTracker(name) }

    DisposableEffect(Unit) {
        tracker.startTracking()
        onDispose {
            val report = tracker.stopTracking()
            Log.d("ScrollPerf", "Report for $name: $report")
        }
    }

    Column(
        modifier = modifier.verticalScroll(rememberScrollState()),
        content = content
    )
}
```

---

## Animation Performance

Track animation smoothness, dropped frames, and hitches.

### iOS Animation Tracking

```swift
// AnimationPerformanceTracker.swift
import QuartzCore

final class AnimationPerformanceTracker {
    static let shared = AnimationPerformanceTracker()

    private var displayLink: CADisplayLink?
    private var animationStartTime: CFTimeInterval = 0
    private var expectedDuration: TimeInterval = 0
    private var animationName: String = ""
    private var frameTimestamps: [CFTimeInterval] = []

    func trackAnimation(
        name: String,
        duration: TimeInterval,
        animation: @escaping () -> Void
    ) {
        animationName = name
        expectedDuration = duration
        frameTimestamps.removeAll()
        animationStartTime = CACurrentMediaTime()

        // Start frame monitoring
        displayLink = CADisplayLink(target: self, selector: #selector(recordFrame))
        displayLink?.add(to: .main, forMode: .common)

        // Run animation
        animation()

        // Schedule completion check
        DispatchQueue.main.asyncAfter(deadline: .now() + duration + 0.1) { [weak self] in
            self?.finishTracking()
        }
    }

    @objc private func recordFrame(_ link: CADisplayLink) {
        frameTimestamps.append(link.timestamp)
    }

    private func finishTracking() {
        displayLink?.invalidate()
        displayLink = nil

        guard frameTimestamps.count > 1 else { return }

        // Analyze frame intervals
        var droppedFrames = 0
        var maxGap: TimeInterval = 0
        let targetInterval: TimeInterval = 1.0 / 60.0 // 60fps

        for i in 1..<frameTimestamps.count {
            let interval = frameTimestamps[i] - frameTimestamps[i-1]
            maxGap = max(maxGap, interval)

            if interval > targetInterval * 1.5 {
                droppedFrames += Int(interval / targetInterval) - 1
            }
        }

        let actualDuration = frameTimestamps.last! - frameTimestamps.first!
        let smoothness = 1.0 - (Double(droppedFrames) / Double(frameTimestamps.count))

        // Report
        Observability.recordMetric(
            name: "animation.smoothness",
            value: smoothness * 100,
            tags: [
                "animation": animationName,
                "duration_ms": String(Int(expectedDuration * 1000))
            ]
        )

        if droppedFrames > 2 {
            Observability.captureMessage(
                "Animation hitch detected",
                level: .warning,
                extras: [
                    "animation": animationName,
                    "dropped_frames": droppedFrames,
                    "max_gap_ms": maxGap * 1000,
                    "smoothness_percent": smoothness * 100
                ]
            )
        }
    }
}

// SwiftUI animation tracking
extension View {
    func trackedAnimation<V: Equatable>(
        _ name: String,
        value: V,
        animation: Animation = .default,
        body: @escaping () -> Void
    ) -> some View {
        self.onChange(of: value) { _, _ in
            let duration = animation.duration ?? 0.3
            AnimationPerformanceTracker.shared.trackAnimation(
                name: name,
                duration: duration
            ) {
                withAnimation(animation) {
                    body()
                }
            }
        }
    }
}

// Animation extension to get duration
extension Animation {
    var duration: TimeInterval? {
        // Mirror to extract duration from Animation struct
        let mirror = Mirror(reflecting: self)
        for child in mirror.children {
            if let duration = child.value as? TimeInterval {
                return duration
            }
        }
        return nil
    }
}
```

### Android Animation Tracking

```kotlin
// AnimationPerformanceTracker.kt
class AnimationPerformanceTracker {
    private var choreographer: Choreographer? = null
    private var frameTimestamps = mutableListOf<Long>()
    private var animationName: String = ""
    private var expectedDurationMs: Long = 0
    private var startTimeNanos: Long = 0

    private val frameCallback = object : Choreographer.FrameCallback {
        override fun doFrame(frameTimeNanos: Long) {
            frameTimestamps.add(frameTimeNanos)

            val elapsedMs = (frameTimeNanos - startTimeNanos) / 1_000_000
            if (elapsedMs < expectedDurationMs + 100) {
                choreographer?.postFrameCallback(this)
            } else {
                finishTracking()
            }
        }
    }

    fun trackAnimation(
        name: String,
        durationMs: Long,
        animation: () -> Unit
    ) {
        animationName = name
        expectedDurationMs = durationMs
        frameTimestamps.clear()

        choreographer = Choreographer.getInstance()
        startTimeNanos = System.nanoTime()
        choreographer?.postFrameCallback(frameCallback)

        animation()
    }

    private fun finishTracking() {
        choreographer?.removeFrameCallback(frameCallback)

        if (frameTimestamps.size < 2) return

        var droppedFrames = 0
        var maxGapMs = 0.0
        val targetIntervalMs = 16.67 // 60fps

        for (i in 1 until frameTimestamps.size) {
            val intervalMs = (frameTimestamps[i] - frameTimestamps[i-1]) / 1_000_000.0
            maxGapMs = maxOf(maxGapMs, intervalMs)

            if (intervalMs > targetIntervalMs * 1.5) {
                droppedFrames += (intervalMs / targetIntervalMs).toInt() - 1
            }
        }

        val smoothness = 1.0 - (droppedFrames.toDouble() / frameTimestamps.size)

        Observability.recordMetric(
            name = "animation.smoothness",
            value = smoothness * 100,
            tags = mapOf(
                "animation" to animationName,
                "duration_ms" to expectedDurationMs.toString()
            )
        )

        if (droppedFrames > 2) {
            Observability.captureMessage(
                message = "Animation hitch detected",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "animation" to animationName,
                    "dropped_frames" to droppedFrames,
                    "max_gap_ms" to maxGapMs,
                    "smoothness_percent" to smoothness * 100
                )
            )
        }
    }

    companion object {
        val shared = AnimationPerformanceTracker()
    }
}
```

---

## Time to Interactive

Measuring when the screen is fully usable, not just visible.

### TTI Definition

```
App Launch / Navigation Start
        │
        ▼
   First Paint (FP)          ← Something visible
        │
        ▼
   First Contentful Paint    ← Meaningful content visible
        │
        ▼
   Time to Interactive       ← User can interact
        │
        ▼
   Fully Loaded              ← All async content loaded
```

### iOS TTI Tracker

```swift
// TTITracker.swift
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

    func markFullyLoaded() {
        guard milestones["fully_loaded"] == nil else { return }
        milestones["fully_loaded"] = CFAbsoluteTimeGetCurrent()
        reportFullyLoaded()
    }

    private func reportTTI() {
        guard !isComplete, let interactive = milestones["interactive"] else { return }
        isComplete = true

        let tti = (interactive - startTime) * 1000

        var breakdown: [String: Double] = [:]
        var lastTime = startTime

        for milestone in ["first_paint", "contentful_paint", "interactive"] {
            if let time = milestones[milestone] {
                breakdown[milestone] = (time - lastTime) * 1000
                lastTime = time
            }
        }

        Observability.recordMetric(
            name: "screen.tti",
            value: tti,
            unit: .milliseconds,
            tags: [
                "screen": screenName,
                "bucket": ttiBucket(tti)
            ]
        )

        // Record breakdown
        for (milestone, duration) in breakdown {
            Observability.recordMetric(
                name: "screen.tti.phase",
                value: duration,
                unit: .milliseconds,
                tags: [
                    "screen": screenName,
                    "phase": milestone
                ]
            )
        }

        if tti > 3000 {
            Observability.captureMessage(
                "Slow TTI",
                level: .warning,
                extras: [
                    "screen": screenName,
                    "tti_ms": tti,
                    "breakdown": breakdown
                ]
            )
        }
    }

    private func reportFullyLoaded() {
        guard let fullyLoaded = milestones["fully_loaded"] else { return }

        let totalTime = (fullyLoaded - startTime) * 1000

        Observability.recordMetric(
            name: "screen.fully_loaded",
            value: totalTime,
            unit: .milliseconds,
            tags: ["screen": screenName]
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

// SwiftUI usage
struct ProductDetailView: View {
    let productId: String
    @StateObject private var viewModel = ProductDetailViewModel()
    @State private var ttiTracker: TTITracker?

    var body: some View {
        Group {
            if viewModel.isLoading {
                LoadingView()
                    .onAppear {
                        ttiTracker?.markFirstPaint()
                    }
            } else if let product = viewModel.product {
                ProductContent(product: product)
                    .onAppear {
                        ttiTracker?.markContentfulPaint()
                        ttiTracker?.markInteractive()
                    }
            }
        }
        .onAppear {
            ttiTracker = TTITracker(screenName: "ProductDetail")
        }
        .task {
            await viewModel.loadProduct(id: productId)
            ttiTracker?.markFullyLoaded()
        }
    }
}
```

### Android TTI Tracker

```kotlin
// TTITracker.kt
class TTITracker(private val screenName: String) {
    private val startTimeNanos = System.nanoTime()
    private val milestones = mutableMapOf<String, Long>()
    private var isComplete = false

    fun markFirstPaint() {
        if ("first_paint" !in milestones) {
            milestones["first_paint"] = System.nanoTime()
        }
    }

    fun markContentfulPaint() {
        if ("contentful_paint" !in milestones) {
            milestones["contentful_paint"] = System.nanoTime()
        }
    }

    fun markInteractive() {
        if ("interactive" !in milestones) {
            milestones["interactive"] = System.nanoTime()
            reportTTI()
        }
    }

    fun markFullyLoaded() {
        if ("fully_loaded" !in milestones) {
            milestones["fully_loaded"] = System.nanoTime()
            reportFullyLoaded()
        }
    }

    private fun reportTTI() {
        if (isComplete) return
        val interactiveTime = milestones["interactive"] ?: return
        isComplete = true

        val ttiMs = (interactiveTime - startTimeNanos) / 1_000_000.0

        val breakdown = mutableMapOf<String, Double>()
        var lastTime = startTimeNanos

        listOf("first_paint", "contentful_paint", "interactive").forEach { milestone ->
            milestones[milestone]?.let { time ->
                breakdown[milestone] = (time - lastTime) / 1_000_000.0
                lastTime = time
            }
        }

        Observability.recordMetric(
            name = "screen.tti",
            value = ttiMs,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf(
                "screen" to screenName,
                "bucket" to ttiBucket(ttiMs)
            )
        )

        breakdown.forEach { (milestone, duration) ->
            Observability.recordMetric(
                name = "screen.tti.phase",
                value = duration,
                unit = MetricUnit.MILLISECONDS,
                tags = mapOf(
                    "screen" to screenName,
                    "phase" to milestone
                )
            )
        }

        if (ttiMs > 3000) {
            Observability.captureMessage(
                message = "Slow TTI",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "screen" to screenName,
                    "tti_ms" to ttiMs,
                    "breakdown" to breakdown
                )
            )
        }
    }

    private fun reportFullyLoaded() {
        val fullyLoadedTime = milestones["fully_loaded"] ?: return
        val totalMs = (fullyLoadedTime - startTimeNanos) / 1_000_000.0

        Observability.recordMetric(
            name = "screen.fully_loaded",
            value = totalMs,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf("screen" to screenName)
        )
    }

    private fun ttiBucket(ms: Double): String = when {
        ms < 1000 -> "fast"
        ms < 2000 -> "good"
        ms < 3000 -> "acceptable"
        ms < 5000 -> "slow"
        else -> "very_slow"
    }
}

// Compose usage
@Composable
fun ProductDetailScreen(
    productId: String,
    viewModel: ProductDetailViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val ttiTracker = remember { TTITracker("ProductDetail") }

    when (val state = uiState) {
        is ProductUiState.Loading -> {
            LoadingView()
            LaunchedEffect(Unit) { ttiTracker.markFirstPaint() }
        }
        is ProductUiState.Success -> {
            ProductContent(state.product)
            LaunchedEffect(Unit) {
                ttiTracker.markContentfulPaint()
                ttiTracker.markInteractive()
            }
        }
        is ProductUiState.Error -> ErrorView(state.message)
    }

    LaunchedEffect(uiState) {
        if (uiState is ProductUiState.Success) {
            ttiTracker.markFullyLoaded()
        }
    }
}
```

---

## Entry Point Latency

Different entry points have different performance characteristics.

### Entry Point Types

| Entry Point | What to Measure | Target |
|-------------|-----------------|--------|
| Cold Launch | Process start → interactive | <2s |
| Warm Launch | App foregrounded → interactive | <500ms |
| Deep Link | Link tap → target screen interactive | <1s |
| Push Notification | Tap → relevant screen | <800ms |
| Widget | Widget tap → app screen | <600ms |
| Spotlight/Shortcuts | Shortcut → target screen | <600ms |
| Share Extension | Share sheet → extension ready | <400ms |

### iOS Entry Point Tracking

```swift
// EntryPointTracker.swift
import UIKit

enum EntryPoint: String {
    case coldLaunch = "cold_launch"
    case warmLaunch = "warm_launch"
    case deepLink = "deep_link"
    case pushNotification = "push_notification"
    case widget = "widget"
    case spotlight = "spotlight"
    case shortcut = "shortcut"
    case shareExtension = "share_extension"
    case handoff = "handoff"
}

final class EntryPointTracker {
    static let shared = EntryPointTracker()

    private var currentEntryPoint: EntryPoint?
    private var entryTime: CFAbsoluteTime = 0
    private var metadata: [String: Any] = [:]

    // Call from AppDelegate or SceneDelegate
    func trackLaunch(options: [UIApplication.LaunchOptionsKey: Any]?) {
        entryTime = CFAbsoluteTimeGetCurrent()

        if let url = options?[.url] as? URL {
            currentEntryPoint = .deepLink
            metadata["url_scheme"] = url.scheme ?? "unknown"
            metadata["url_host"] = url.host ?? "unknown"
        } else if let notification = options?[.remoteNotification] as? [AnyHashable: Any] {
            currentEntryPoint = .pushNotification
            metadata["notification_type"] = notification["type"] as? String ?? "unknown"
        } else if options?[.shortcutItem] != nil {
            currentEntryPoint = .shortcut
        } else if let userActivity = options?[.userActivityDictionary] as? [String: Any],
                  let activityType = userActivity["UIApplicationLaunchOptionsUserActivityKey"] as? String {
            if activityType.contains("CSSearchableItemActionType") {
                currentEntryPoint = .spotlight
            } else {
                currentEntryPoint = .handoff
            }
        } else {
            currentEntryPoint = ProcessInfo.processInfo.arguments.contains("-warm") ? .warmLaunch : .coldLaunch
        }
    }

    func trackWidgetLaunch(widgetKind: String, deepLink: URL?) {
        entryTime = CFAbsoluteTimeGetCurrent()
        currentEntryPoint = .widget
        metadata["widget_kind"] = widgetKind
        if let link = deepLink {
            metadata["deep_link"] = link.absoluteString
        }
    }

    func markInteractive(screen: String) {
        guard let entryPoint = currentEntryPoint else { return }

        let latency = (CFAbsoluteTimeGetCurrent() - entryTime) * 1000

        Observability.recordMetric(
            name: "entry_point.latency",
            value: latency,
            unit: .milliseconds,
            tags: [
                "entry_point": entryPoint.rawValue,
                "destination_screen": screen,
                "bucket": latencyBucket(latency, for: entryPoint)
            ]
        )

        // Add metadata
        var extras = metadata
        extras["latency_ms"] = latency
        extras["destination_screen"] = screen

        if isSlowForEntryPoint(latency, entryPoint: entryPoint) {
            Observability.captureMessage(
                "Slow entry point",
                level: .warning,
                extras: extras
            )
        }

        // Reset
        currentEntryPoint = nil
        metadata.removeAll()
    }

    private func latencyBucket(_ ms: Double, for entryPoint: EntryPoint) -> String {
        let thresholds: (good: Double, acceptable: Double, slow: Double)

        switch entryPoint {
        case .coldLaunch:
            thresholds = (1000, 2000, 4000)
        case .warmLaunch:
            thresholds = (300, 500, 1000)
        case .deepLink, .pushNotification:
            thresholds = (500, 800, 1500)
        case .widget, .spotlight, .shortcut:
            thresholds = (400, 600, 1000)
        case .shareExtension:
            thresholds = (200, 400, 800)
        case .handoff:
            thresholds = (500, 1000, 2000)
        }

        switch ms {
        case ..<thresholds.good: return "fast"
        case ..<thresholds.acceptable: return "good"
        case ..<thresholds.slow: return "acceptable"
        default: return "slow"
        }
    }

    private func isSlowForEntryPoint(_ ms: Double, entryPoint: EntryPoint) -> Bool {
        switch entryPoint {
        case .coldLaunch: return ms > 3000
        case .warmLaunch: return ms > 800
        case .deepLink, .pushNotification: return ms > 1200
        case .widget, .spotlight, .shortcut: return ms > 800
        case .shareExtension: return ms > 600
        case .handoff: return ms > 1500
        }
    }
}

// SceneDelegate integration
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    func scene(_ scene: UIScene, willConnectTo session: UISceneSession,
               options connectionOptions: UIScene.ConnectionOptions) {

        // Track URL contexts (deep links)
        if let urlContext = connectionOptions.urlContexts.first {
            EntryPointTracker.shared.trackLaunch(options: [
                .url: urlContext.url
            ])
        }

        // Track user activities (Spotlight, Handoff)
        if let userActivity = connectionOptions.userActivities.first {
            EntryPointTracker.shared.trackLaunch(options: [
                .userActivityDictionary: ["UIApplicationLaunchOptionsUserActivityKey": userActivity.activityType]
            ])
        }

        // Track shortcuts
        if let shortcut = connectionOptions.shortcutItem {
            EntryPointTracker.shared.trackLaunch(options: [
                .shortcutItem: shortcut
            ])
        }

        // Track push notifications
        if let notification = connectionOptions.notificationResponse {
            EntryPointTracker.shared.trackLaunch(options: [
                .remoteNotification: notification.notification.request.content.userInfo
            ])
        }
    }
}
```

### Android Entry Point Tracking

```kotlin
// EntryPointTracker.kt
enum class EntryPoint(val value: String) {
    COLD_LAUNCH("cold_launch"),
    WARM_LAUNCH("warm_launch"),
    DEEP_LINK("deep_link"),
    PUSH_NOTIFICATION("push_notification"),
    WIDGET("widget"),
    SHORTCUT("shortcut"),
    SHARE("share"),
    QUICK_SETTINGS("quick_settings")
}

object EntryPointTracker {
    private var currentEntryPoint: EntryPoint? = null
    private var entryTimeNanos: Long = 0
    private var metadata: MutableMap<String, Any> = mutableMapOf()

    fun trackLaunch(intent: Intent?) {
        entryTimeNanos = System.nanoTime()
        metadata.clear()

        currentEntryPoint = when {
            intent?.action == Intent.ACTION_VIEW && intent.data != null -> {
                metadata["uri"] = intent.data.toString()
                metadata["scheme"] = intent.data?.scheme ?: "unknown"
                EntryPoint.DEEP_LINK
            }
            intent?.hasExtra("notification_id") == true -> {
                metadata["notification_type"] = intent.getStringExtra("notification_type") ?: "unknown"
                EntryPoint.PUSH_NOTIFICATION
            }
            intent?.hasExtra("appwidget_id") == true -> {
                metadata["widget_id"] = intent.getIntExtra("appwidget_id", -1)
                EntryPoint.WIDGET
            }
            intent?.action?.startsWith("android.intent.action.CREATE_SHORTCUT") == true -> {
                EntryPoint.SHORTCUT
            }
            intent?.action == Intent.ACTION_SEND || intent?.action == Intent.ACTION_SEND_MULTIPLE -> {
                metadata["mime_type"] = intent.type ?: "unknown"
                EntryPoint.SHARE
            }
            ProcessInfo.isWarmStart() -> EntryPoint.WARM_LAUNCH
            else -> EntryPoint.COLD_LAUNCH
        }
    }

    fun markInteractive(screen: String) {
        val entryPoint = currentEntryPoint ?: return

        val latencyMs = (System.nanoTime() - entryTimeNanos) / 1_000_000.0

        Observability.recordMetric(
            name = "entry_point.latency",
            value = latencyMs,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf(
                "entry_point" to entryPoint.value,
                "destination_screen" to screen,
                "bucket" to latencyBucket(latencyMs, entryPoint)
            )
        )

        if (isSlowForEntryPoint(latencyMs, entryPoint)) {
            val extras = metadata.toMutableMap()
            extras["latency_ms"] = latencyMs
            extras["destination_screen"] = screen

            Observability.captureMessage(
                message = "Slow entry point",
                level = SentryLevel.WARNING,
                extras = extras
            )
        }

        currentEntryPoint = null
        metadata.clear()
    }

    private fun latencyBucket(ms: Double, entryPoint: EntryPoint): String {
        val thresholds = when (entryPoint) {
            EntryPoint.COLD_LAUNCH -> Triple(1000.0, 2000.0, 4000.0)
            EntryPoint.WARM_LAUNCH -> Triple(300.0, 500.0, 1000.0)
            EntryPoint.DEEP_LINK, EntryPoint.PUSH_NOTIFICATION -> Triple(500.0, 800.0, 1500.0)
            EntryPoint.WIDGET, EntryPoint.SHORTCUT -> Triple(400.0, 600.0, 1000.0)
            EntryPoint.SHARE -> Triple(200.0, 400.0, 800.0)
            EntryPoint.QUICK_SETTINGS -> Triple(300.0, 500.0, 800.0)
        }

        return when {
            ms < thresholds.first -> "fast"
            ms < thresholds.second -> "good"
            ms < thresholds.third -> "acceptable"
            else -> "slow"
        }
    }

    private fun isSlowForEntryPoint(ms: Double, entryPoint: EntryPoint): Boolean {
        return when (entryPoint) {
            EntryPoint.COLD_LAUNCH -> ms > 3000
            EntryPoint.WARM_LAUNCH -> ms > 800
            EntryPoint.DEEP_LINK, EntryPoint.PUSH_NOTIFICATION -> ms > 1200
            EntryPoint.WIDGET, EntryPoint.SHORTCUT -> ms > 800
            EntryPoint.SHARE -> ms > 600
            EntryPoint.QUICK_SETTINGS -> ms > 600
        }
    }
}

// Helper to detect warm starts
object ProcessInfo {
    private var hasLaunchedBefore = false

    fun isWarmStart(): Boolean {
        val isWarm = hasLaunchedBefore
        hasLaunchedBefore = true
        return isWarm
    }
}

// Activity integration
abstract class TrackedActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        EntryPointTracker.trackLaunch(intent)
    }

    protected fun markScreenInteractive(screenName: String) {
        EntryPointTracker.markInteractive(screenName)
    }
}
```

---

## Image Loading

Track image load performance, failures, and cache effectiveness.

### iOS Image Loading Tracking

```swift
// ImageLoadingTracker.swift
import Foundation

struct ImageLoadMetrics {
    let url: URL
    let loadTimeMs: Double
    let source: ImageSource
    let sizeBytes: Int64
    let dimensions: CGSize?
    let error: Error?

    enum ImageSource: String {
        case memory = "memory"
        case disk = "disk"
        case network = "network"
    }
}

protocol ImageLoadingObserver {
    func onImageLoadStarted(url: URL)
    func onImageLoadCompleted(metrics: ImageLoadMetrics)
}

final class ImageLoadingTracker: ImageLoadingObserver {
    static let shared = ImageLoadingTracker()

    private var pendingLoads: [URL: CFAbsoluteTime] = [:]
    private let queue = DispatchQueue(label: "image.tracker")

    func onImageLoadStarted(url: URL) {
        queue.async {
            self.pendingLoads[url] = CFAbsoluteTimeGetCurrent()
        }
    }

    func onImageLoadCompleted(metrics: ImageLoadMetrics) {
        // Cache hit/miss tracking
        Observability.recordMetric(
            name: "image.load.source",
            value: 1,
            tags: [
                "source": metrics.source.rawValue,
                "host": metrics.url.host ?? "unknown"
            ]
        )

        // Load time tracking (network only)
        if metrics.source == .network {
            Observability.recordMetric(
                name: "image.load.duration",
                value: metrics.loadTimeMs,
                unit: .milliseconds,
                tags: [
                    "host": metrics.url.host ?? "unknown",
                    "bucket": loadTimeBucket(metrics.loadTimeMs)
                ]
            )

            // Size tracking
            if metrics.sizeBytes > 0 {
                Observability.recordMetric(
                    name: "image.load.size",
                    value: Double(metrics.sizeBytes),
                    unit: .bytes,
                    tags: ["host": metrics.url.host ?? "unknown"]
                )
            }

            // Slow image warning
            if metrics.loadTimeMs > 2000 {
                Observability.captureMessage(
                    "Slow image load",
                    level: .warning,
                    extras: [
                        "url": metrics.url.absoluteString,
                        "load_time_ms": metrics.loadTimeMs,
                        "size_bytes": metrics.sizeBytes
                    ]
                )
            }
        }

        // Error tracking
        if let error = metrics.error {
            Observability.captureError(
                error,
                extras: [
                    "url": metrics.url.absoluteString,
                    "context": "image_loading"
                ]
            )
        }
    }

    private func loadTimeBucket(_ ms: Double) -> String {
        switch ms {
        case ..<200: return "fast"
        case ..<500: return "good"
        case ..<1000: return "acceptable"
        case ..<2000: return "slow"
        default: return "very_slow"
        }
    }
}
```

### SwiftUI AsyncImage Tracking

```swift
// TrackedAsyncImage.swift
import SwiftUI

struct TrackedAsyncImage<Content: View, Placeholder: View>: View {
    let url: URL?
    let scale: CGFloat
    let transaction: Transaction
    let content: (Image) -> Content
    let placeholder: () -> Placeholder

    @State private var loadStartTime: CFAbsoluteTime = 0
    @State private var phase: AsyncImagePhase = .empty

    var body: some View {
        AsyncImage(
            url: url,
            scale: scale,
            transaction: transaction
        ) { phase in
            switch phase {
            case .empty:
                placeholder()
                    .onAppear {
                        loadStartTime = CFAbsoluteTimeGetCurrent()
                        if let url = url {
                            ImageLoadingTracker.shared.onImageLoadStarted(url: url)
                        }
                    }
            case .success(let image):
                content(image)
                    .onAppear {
                        guard let url = url else { return }
                        let loadTime = (CFAbsoluteTimeGetCurrent() - loadStartTime) * 1000
                        ImageLoadingTracker.shared.onImageLoadCompleted(
                            metrics: ImageLoadMetrics(
                                url: url,
                                loadTimeMs: loadTime,
                                source: .network,
                                sizeBytes: 0,
                                dimensions: nil,
                                error: nil
                            )
                        )
                    }
            case .failure(let error):
                placeholder()
                    .onAppear {
                        guard let url = url else { return }
                        let loadTime = (CFAbsoluteTimeGetCurrent() - loadStartTime) * 1000
                        ImageLoadingTracker.shared.onImageLoadCompleted(
                            metrics: ImageLoadMetrics(
                                url: url,
                                loadTimeMs: loadTime,
                                source: .network,
                                sizeBytes: 0,
                                dimensions: nil,
                                error: error
                            )
                        )
                    }
            @unknown default:
                placeholder()
            }
        }
    }
}

// Convenience initializer
extension TrackedAsyncImage where Placeholder == ProgressView<EmptyView, EmptyView> {
    init(url: URL?, content: @escaping (Image) -> Content) {
        self.url = url
        self.scale = 1
        self.transaction = Transaction()
        self.content = content
        self.placeholder = { ProgressView() }
    }
}
```

### Android Image Loading Tracking

```kotlin
// ImageLoadingTracker.kt
data class ImageLoadMetrics(
    val url: String,
    val loadTimeMs: Double,
    val source: ImageSource,
    val sizeBytes: Long,
    val width: Int?,
    val height: Int?,
    val error: Throwable?
) {
    enum class ImageSource(val value: String) {
        MEMORY("memory"),
        DISK("disk"),
        NETWORK("network")
    }
}

object ImageLoadingTracker {
    private val pendingLoads = ConcurrentHashMap<String, Long>()

    fun onLoadStarted(url: String) {
        pendingLoads[url] = System.nanoTime()
    }

    fun onLoadCompleted(metrics: ImageLoadMetrics) {
        pendingLoads.remove(metrics.url)

        // Source tracking
        Observability.recordMetric(
            name = "image.load.source",
            value = 1.0,
            tags = mapOf(
                "source" to metrics.source.value,
                "host" to Uri.parse(metrics.url).host.orEmpty()
            )
        )

        // Network loads only
        if (metrics.source == ImageLoadMetrics.ImageSource.NETWORK) {
            Observability.recordMetric(
                name = "image.load.duration",
                value = metrics.loadTimeMs,
                unit = MetricUnit.MILLISECONDS,
                tags = mapOf(
                    "host" to Uri.parse(metrics.url).host.orEmpty(),
                    "bucket" to loadTimeBucket(metrics.loadTimeMs)
                )
            )

            if (metrics.sizeBytes > 0) {
                Observability.recordMetric(
                    name = "image.load.size",
                    value = metrics.sizeBytes.toDouble(),
                    unit = MetricUnit.BYTES,
                    tags = mapOf("host" to Uri.parse(metrics.url).host.orEmpty())
                )
            }

            if (metrics.loadTimeMs > 2000) {
                Observability.captureMessage(
                    message = "Slow image load",
                    level = SentryLevel.WARNING,
                    extras = mapOf(
                        "url" to metrics.url,
                        "load_time_ms" to metrics.loadTimeMs,
                        "size_bytes" to metrics.sizeBytes
                    )
                )
            }
        }

        metrics.error?.let { error ->
            Observability.captureError(
                error,
                extras = mapOf(
                    "url" to metrics.url,
                    "context" to "image_loading"
                )
            )
        }
    }

    private fun loadTimeBucket(ms: Double): String = when {
        ms < 200 -> "fast"
        ms < 500 -> "good"
        ms < 1000 -> "acceptable"
        ms < 2000 -> "slow"
        else -> "very_slow"
    }
}

// Coil integration
class ObservabilityEventListener : EventListener {
    private var startTimeNanos: Long = 0

    override fun fetchStart(request: ImageRequest, data: Any) {
        startTimeNanos = System.nanoTime()
        ImageLoadingTracker.onLoadStarted(request.data.toString())
    }

    override fun fetchEnd(request: ImageRequest, data: Any, result: FetchResult?) {
        val loadTimeMs = (System.nanoTime() - startTimeNanos) / 1_000_000.0
        val source = when (result) {
            is FetchResult.SourceResult -> ImageLoadMetrics.ImageSource.NETWORK
            else -> ImageLoadMetrics.ImageSource.MEMORY
        }

        ImageLoadingTracker.onLoadCompleted(
            ImageLoadMetrics(
                url = request.data.toString(),
                loadTimeMs = loadTimeMs,
                source = source,
                sizeBytes = result?.source?.source()?.buffer?.size ?: 0,
                width = null,
                height = null,
                error = null
            )
        )
    }

    override fun onError(request: ImageRequest, result: ErrorResult) {
        ImageLoadingTracker.onLoadCompleted(
            ImageLoadMetrics(
                url = request.data.toString(),
                loadTimeMs = 0.0,
                source = ImageLoadMetrics.ImageSource.NETWORK,
                sizeBytes = 0,
                width = null,
                height = null,
                error = result.throwable
            )
        )
    }
}
```

---

## Summary: Key Metrics to Track

| Category | Metric | Target | Alert Threshold |
|----------|--------|--------|-----------------|
| Navigation | Tap-to-interactive | <400ms P95 | >1000ms |
| SwiftUI | Body evaluations/sec | <50 | >100 |
| Compose | Recompositions/sec | <50 | >100 |
| Scroll | Dropped frames | <2% | >5% |
| Animation | Smoothness | >95% | <90% |
| TTI | Time to interactive | <2s | >3s |
| Cold Launch | Entry to interactive | <2s | >4s |
| Deep Link | Tap to screen | <1s | >1.5s |
| Images | Network load time | <500ms P95 | >2s |
| Images | Cache hit rate | >80% | <60% |

---

## Integration Points

This reference connects to:

- **[observability-fundamentals.md](observability-fundamentals.md)** - Correlation context for linking UI events to traces
- **[performance.md](performance.md)** - App start and screen load spans
- **[native-mobile.md](native-mobile.md)** - Platform-specific APIs (MetricKit, Perfetto)
- **[mobile-challenges.md](mobile-challenges.md)** - Offline collection of UI metrics
- **Platform docs** - Sentry/Datadog/Embrace specific APIs for these metrics

## Reusable Templates

- **[templates/navigation-tracking.template.md](templates/navigation-tracking.template.md)** - Navigation latency and TTI measurement
- **[templates/screen-load-tracking.template.md](templates/screen-load-tracking.template.md)** - Screen load phase tracking
