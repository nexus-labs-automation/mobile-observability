# Screen Load Tracking Template

Reusable patterns for measuring screen load performance with phase tracking.

## iOS

### ScreenLoadTracker

```swift
// ScreenLoadTracker.swift
import Foundation
import os.signpost

final class ScreenLoadTracker {
    private let screenName: String
    private let startTime: CFAbsoluteTime
    private var phases: [String: CFAbsoluteTime] = [:]
    private var isComplete = false
    private let signpostID: OSSignpostID
    private let signpostLog: OSLog

    init(screenName: String) {
        self.screenName = screenName
        self.startTime = CFAbsoluteTimeGetCurrent()
        self.signpostLog = OSLog(subsystem: Bundle.main.bundleIdentifier!, category: "ScreenLoad")
        self.signpostID = OSSignpostID(log: signpostLog)

        os_signpost(.begin, log: signpostLog, name: "ScreenLoad", signpostID: signpostID,
                    "screen: %{public}s", screenName)
    }

    func markViewCreated() {
        phases["view_created"] = CFAbsoluteTimeGetCurrent()
    }

    func markViewAppeared() {
        phases["view_appeared"] = CFAbsoluteTimeGetCurrent()
    }

    func markDataReady() {
        phases["data_ready"] = CFAbsoluteTimeGetCurrent()
    }

    func markFirstContentfulPaint() {
        phases["fcp"] = CFAbsoluteTimeGetCurrent()
    }

    func markInteractive(metadata: [String: Any] = [:]) {
        guard !isComplete else { return }
        isComplete = true

        phases["interactive"] = CFAbsoluteTimeGetCurrent()

        os_signpost(.end, log: signpostLog, name: "ScreenLoad", signpostID: signpostID,
                    "screen: %{public}s, status: complete", screenName)

        reportScreenLoad(metadata: metadata)
    }

    private func reportScreenLoad(metadata: [String: Any]) {
        guard let interactiveTime = phases["interactive"] else { return }

        let totalTime = (interactiveTime - startTime) * 1000
        let phaseDurations = calculatePhaseDurations()

        Observability.recordMetric(
            name: "screen_load.duration",
            value: totalTime,
            unit: .milliseconds,
            tags: [
                "screen": screenName,
                "bucket": durationBucket(totalTime)
            ]
        )

        for (phase, duration) in phaseDurations {
            Observability.recordMetric(
                name: "screen_load.phase",
                value: duration,
                unit: .milliseconds,
                tags: ["screen": screenName, "phase": phase]
            )
        }

        let threshold = ScreenLoadThresholds.threshold(for: screenName)
        if totalTime > threshold {
            Observability.captureMessage(
                "Slow screen load: \(screenName)",
                level: .warning,
                extras: [
                    "screen": screenName,
                    "duration_ms": totalTime,
                    "threshold_ms": threshold,
                    "phases": phaseDurations,
                    "metadata": metadata
                ]
            )
        }
    }

    private func calculatePhaseDurations() -> [String: Double] {
        var durations: [String: Double] = [:]
        let orderedPhases = ["view_created", "view_appeared", "data_ready", "fcp", "interactive"]

        var lastTime = startTime
        for phase in orderedPhases {
            if let phaseTime = phases[phase] {
                durations[phase] = (phaseTime - lastTime) * 1000
                lastTime = phaseTime
            }
        }

        return durations
    }

    private func durationBucket(_ ms: Double) -> String {
        switch ms {
        case ..<200: return "fast"
        case ..<400: return "good"
        case ..<800: return "acceptable"
        case ..<1500: return "slow"
        default: return "very_slow"
        }
    }
}

enum ScreenLoadThresholds {
    static func threshold(for screen: String) -> Double {
        switch screen {
        case "HomeScreen": return 1000
        case "ProductDetail": return 800
        case "Checkout": return 1200
        case "Search": return 600
        default: return 1000
        }
    }
}
```

### SwiftUI Integration

```swift
struct TrackedScreen<Content: View>: View {
    let screenName: String
    let tracker: ScreenLoadTracker
    let content: () -> Content

    @State private var isLoaded = false

    init(screenName: String, @ViewBuilder content: @escaping () -> Content) {
        self.screenName = screenName
        self.tracker = ScreenLoadTracker(screenName: screenName)
        self.content = content
    }

    var body: some View {
        content()
            .onAppear {
                tracker.markViewAppeared()
            }
            .background(
                GeometryReader { _ in
                    Color.clear.onAppear {
                        if !isLoaded {
                            isLoaded = true
                            tracker.markFirstContentfulPaint()
                        }
                    }
                }
            )
    }

    func markDataReady() {
        tracker.markDataReady()
    }

    func markInteractive(metadata: [String: Any] = [:]) {
        tracker.markInteractive(metadata: metadata)
    }
}
```

## Android

### ScreenLoadTracker

```kotlin
// ScreenLoadTracker.kt
import android.os.Trace

class ScreenLoadTracker(private val screenName: String) {
    private val startTimeNanos = System.nanoTime()
    private val phases = mutableMapOf<String, Long>()
    private var isComplete = false

    init {
        Trace.beginAsyncSection("ScreenLoad:$screenName", screenName.hashCode())
    }

    fun markViewCreated() {
        phases["view_created"] = System.nanoTime()
    }

    fun markViewResumed() {
        phases["view_resumed"] = System.nanoTime()
    }

    fun markDataReady() {
        phases["data_ready"] = System.nanoTime()
    }

    fun markFirstContentfulPaint() {
        phases["fcp"] = System.nanoTime()
    }

    fun markInteractive(metadata: Map<String, Any> = emptyMap()) {
        if (isComplete) return
        isComplete = true

        phases["interactive"] = System.nanoTime()

        Trace.endAsyncSection("ScreenLoad:$screenName", screenName.hashCode())

        reportScreenLoad(metadata)
    }

    private fun reportScreenLoad(metadata: Map<String, Any>) {
        val interactiveTime = phases["interactive"] ?: return
        val totalTimeMs = (interactiveTime - startTimeNanos) / 1_000_000.0
        val phaseDurations = calculatePhaseDurations()

        Observability.recordMetric(
            name = "screen_load.duration",
            value = totalTimeMs,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf(
                "screen" to screenName,
                "bucket" to durationBucket(totalTimeMs)
            )
        )

        phaseDurations.forEach { (phase, duration) ->
            Observability.recordMetric(
                name = "screen_load.phase",
                value = duration,
                unit = MetricUnit.MILLISECONDS,
                tags = mapOf("screen" to screenName, "phase" to phase)
            )
        }

        val threshold = ScreenLoadThresholds.threshold(screenName)
        if (totalTimeMs > threshold) {
            Observability.captureMessage(
                message = "Slow screen load: $screenName",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "screen" to screenName,
                    "duration_ms" to totalTimeMs,
                    "threshold_ms" to threshold,
                    "phases" to phaseDurations,
                    "metadata" to metadata
                )
            )
        }
    }

    private fun calculatePhaseDurations(): Map<String, Double> {
        val orderedPhases = listOf("view_created", "view_resumed", "data_ready", "fcp", "interactive")
        val durations = mutableMapOf<String, Double>()

        var lastTime = startTimeNanos
        orderedPhases.forEach { phase ->
            phases[phase]?.let { phaseTime ->
                durations[phase] = (phaseTime - lastTime) / 1_000_000.0
                lastTime = phaseTime
            }
        }

        return durations
    }

    private fun durationBucket(ms: Double): String = when {
        ms < 200 -> "fast"
        ms < 400 -> "good"
        ms < 800 -> "acceptable"
        ms < 1500 -> "slow"
        else -> "very_slow"
    }
}

object ScreenLoadThresholds {
    fun threshold(screen: String): Double = when (screen) {
        "HomeScreen" -> 1000.0
        "ProductDetail" -> 800.0
        "Checkout" -> 1200.0
        "Search" -> 600.0
        else -> 1000.0
    }
}
```

### Compose Integration

```kotlin
@Composable
fun <T> TrackedScreen(
    screenName: String,
    state: UiState<T>,
    onDataReady: (T) -> Unit = {},
    content: @Composable (T) -> Unit
) {
    val tracker = remember { ScreenLoadTracker(screenName) }
    var hasReportedFcp by remember { mutableStateOf(false) }
    var hasReportedInteractive by remember { mutableStateOf(false) }

    LaunchedEffect(Unit) {
        tracker.markViewCreated()
    }

    when (state) {
        is UiState.Loading -> {
            LoadingIndicator()
        }
        is UiState.Success -> {
            if (!hasReportedFcp) {
                hasReportedFcp = true
                tracker.markDataReady()
            }

            Box(modifier = Modifier.drawWithContent {
                drawContent()
                if (!hasReportedInteractive) {
                    hasReportedInteractive = true
                    tracker.markFirstContentfulPaint()
                    tracker.markInteractive()
                }
            }) {
                content(state.data)
            }
        }
        is UiState.Error -> {
            ErrorView(state.message)
        }
    }
}
```

## React Native

### useScreenTransaction Hook

```typescript
// hooks/useScreenTransaction.ts
import * as Sentry from '@sentry/react-native';
import { useEffect, useRef } from 'react';

export function useScreenTransaction(screenName: string) {
  const transactionRef = useRef<Sentry.Transaction | null>(null);

  useEffect(() => {
    transactionRef.current = Sentry.startTransaction({
      name: screenName,
      op: 'ui.load',
    });

    return () => {
      transactionRef.current?.finish();
    };
  }, [screenName]);

  return {
    startSpan: (name: string, op: string) => {
      return transactionRef.current?.startChild({ op, description: name });
    },
  };
}

// Usage
function ProfileScreen() {
  const { startSpan } = useScreenTransaction('ProfileScreen');

  useEffect(() => {
    const fetchSpan = startSpan('fetch_profile', 'http.client');
    fetchProfile().finally(() => fetchSpan?.finish());
  }, []);
}
```
