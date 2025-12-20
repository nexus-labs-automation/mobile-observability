# Session Replay

Visual debugging through session replay: capturing user interactions, privacy considerations, and integration patterns.

## Table of Contents
1. [Session Replay Concepts](#session-replay-concepts)
2. [Capture Strategies](#capture-strategies)
3. [Privacy & Masking](#privacy--masking)
4. [Integration Patterns](#integration-patterns)
5. [Performance Impact](#performance-impact)
6. [Replay Analysis](#replay-analysis)

---

## Session Replay Concepts

### What is Session Replay?

Session replay reconstructs user sessions as visual playback, showing exactly what users saw and did.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Session Replay Components                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CAPTURE                    PROCESS                    PLAYBACK  │
│  ───────                    ───────                    ────────  │
│  ┌─────────────┐           ┌─────────────┐           ┌─────────┐│
│  │ Screenshots │           │   Stitch    │           │  Video  ││
│  │   (frames)  │──────────▶│   Frames    │──────────▶│ Player  ││
│  └─────────────┘           └─────────────┘           └─────────┘│
│                                                                  │
│  ┌─────────────┐           ┌─────────────┐           ┌─────────┐│
│  │   Touch     │           │   Overlay   │           │  Event  ││
│  │   Events    │──────────▶│   on Video  │──────────▶│ Timeline││
│  └─────────────┘           └─────────────┘           └─────────┘│
│                                                                  │
│  ┌─────────────┐           ┌─────────────┐           ┌─────────┐│
│  │   Network   │           │  Correlate  │           │ Context ││
│  │    Logs     │──────────▶│   Events    │──────────▶│  Panel  ││
│  └─────────────┘           └─────────────┘           └─────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Capture Methods

| Method | Description | Pros | Cons |
|--------|-------------|------|------|
| **Screenshot** | Periodic screen captures | Accurate visual, simple | High bandwidth, storage |
| **View Hierarchy** | Serialize UI tree | Low bandwidth, reconstructable | Complex, platform-specific |
| **Hybrid** | Screenshots + events | Best of both | Implementation complexity |
| **Video** | Continuous recording | Perfect fidelity | Very high resources |

### When to Use Session Replay

| Use Case | Value |
|----------|-------|
| **Bug reproduction** | See exactly what user did before crash |
| **UX research** | Observe real user behavior patterns |
| **Support tickets** | Visual context for customer issues |
| **Conversion analysis** | Watch drop-off behavior |
| **Fraud detection** | Identify suspicious interaction patterns |

---

## Capture Strategies

### iOS Screenshot-Based Capture

```swift
// SessionReplayCapture.swift
import UIKit

final class SessionReplayCapture {
    static let shared = SessionReplayCapture()

    private var isCapturing = false
    private var frameBuffer: [ReplayFrame] = []
    private var touchBuffer: [TouchEvent] = []
    private var captureTimer: Timer?

    private let maxFramesPerSession = 1000
    private let captureInterval: TimeInterval = 0.5 // 2 FPS
    private let queue = DispatchQueue(label: "session.replay", qos: .utility)

    struct ReplayFrame {
        let timestamp: Date
        let screenName: String
        let imageData: Data // JPEG compressed
        let viewportSize: CGSize
    }

    struct TouchEvent {
        let timestamp: Date
        let type: TouchType
        let location: CGPoint
        let screenName: String

        enum TouchType: String {
            case tap, swipe, scroll, longPress
        }
    }

    // MARK: - Capture Control

    func startCapture() {
        guard !isCapturing else { return }
        isCapturing = true

        captureTimer = Timer.scheduledTimer(withTimeInterval: captureInterval, repeats: true) { [weak self] _ in
            self?.captureFrame()
        }

        setupTouchCapture()
    }

    func stopCapture() {
        isCapturing = false
        captureTimer?.invalidate()
        captureTimer = nil
    }

    // MARK: - Frame Capture

    private func captureFrame() {
        guard isCapturing else { return }

        DispatchQueue.main.async {
            guard let window = UIApplication.shared.windows.first(where: { $0.isKeyWindow }) else { return }

            // Capture screenshot
            let renderer = UIGraphicsImageRenderer(size: window.bounds.size)
            let image = renderer.image { context in
                window.drawHierarchy(in: window.bounds, afterScreenUpdates: false)
            }

            // Apply privacy masking
            let maskedImage = PrivacyMasker.shared.maskSensitiveViews(in: image, window: window)

            // Compress
            guard let imageData = maskedImage.jpegData(compressionQuality: 0.5) else { return }

            let frame = ReplayFrame(
                timestamp: Date(),
                screenName: NavigationState.shared.currentScreen,
                imageData: imageData,
                viewportSize: window.bounds.size
            )

            self.queue.async {
                self.frameBuffer.append(frame)

                // Limit buffer size
                if self.frameBuffer.count > self.maxFramesPerSession {
                    self.frameBuffer.removeFirst(100)
                }
            }
        }
    }

    // MARK: - Touch Capture

    private func setupTouchCapture() {
        // Swizzle UIWindow sendEvent to capture touches
        // In production, use proper gesture recognizers or UIKit hooks
    }

    func recordTouch(type: TouchEvent.TouchType, at location: CGPoint) {
        let event = TouchEvent(
            timestamp: Date(),
            type: type,
            location: location,
            screenName: NavigationState.shared.currentScreen
        )

        queue.async {
            self.touchBuffer.append(event)
        }
    }

    // MARK: - Export

    func exportSession() -> SessionReplayData {
        queue.sync {
            SessionReplayData(
                sessionId: SessionManager.shared.sessionId,
                startTime: frameBuffer.first?.timestamp ?? Date(),
                endTime: frameBuffer.last?.timestamp ?? Date(),
                frames: frameBuffer,
                touches: touchBuffer,
                events: BreadcrumbManager.shared.getBreadcrumbs()
            )
        }
    }

    func uploadSession(for errorId: String? = nil) {
        let data = exportSession()

        // Compress and upload
        queue.async {
            let payload = self.compressSessionData(data)
            self.upload(payload, errorId: errorId)
        }
    }

    private func compressSessionData(_ data: SessionReplayData) -> Data {
        // Compress frames and metadata
        // In production, use efficient encoding
        return Data()
    }

    private func upload(_ data: Data, errorId: String?) {
        // Upload to replay backend
    }
}

struct SessionReplayData {
    let sessionId: String
    let startTime: Date
    let endTime: Date
    let frames: [SessionReplayCapture.ReplayFrame]
    let touches: [SessionReplayCapture.TouchEvent]
    let events: [Breadcrumb]
}
```

### Android Screenshot-Based Capture

```kotlin
// SessionReplayCapture.kt
import android.graphics.Bitmap
import android.graphics.Canvas
import android.os.Handler
import android.os.Looper
import android.view.PixelCopy
import android.view.View
import android.view.Window
import java.io.ByteArrayOutputStream
import java.util.concurrent.ConcurrentLinkedQueue

object SessionReplayCapture {
    private var isCapturing = false
    private val frameBuffer = ConcurrentLinkedQueue<ReplayFrame>()
    private val touchBuffer = ConcurrentLinkedQueue<TouchEvent>()
    private val handler = Handler(Looper.getMainLooper())

    private const val MAX_FRAMES = 1000
    private const val CAPTURE_INTERVAL_MS = 500L // 2 FPS

    data class ReplayFrame(
        val timestamp: Long,
        val screenName: String,
        val imageData: ByteArray,
        val width: Int,
        val height: Int
    )

    data class TouchEvent(
        val timestamp: Long,
        val type: TouchType,
        val x: Float,
        val y: Float,
        val screenName: String
    ) {
        enum class TouchType { TAP, SWIPE, SCROLL, LONG_PRESS }
    }

    private val captureRunnable = object : Runnable {
        override fun run() {
            if (isCapturing) {
                captureFrame()
                handler.postDelayed(this, CAPTURE_INTERVAL_MS)
            }
        }
    }

    fun startCapture() {
        if (isCapturing) return
        isCapturing = true
        handler.post(captureRunnable)
    }

    fun stopCapture() {
        isCapturing = false
        handler.removeCallbacks(captureRunnable)
    }

    private fun captureFrame() {
        val activity = ActivityLifecycleTracker.currentActivity ?: return
        val window = activity.window
        val decorView = window.decorView

        // Create bitmap
        val bitmap = Bitmap.createBitmap(
            decorView.width,
            decorView.height,
            Bitmap.Config.ARGB_8888
        )

        // Capture using PixelCopy (API 24+) or Canvas fallback
        if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.O) {
            PixelCopy.request(window, bitmap, { result ->
                if (result == PixelCopy.SUCCESS) {
                    processFrame(bitmap)
                }
            }, handler)
        } else {
            // Fallback for older APIs
            val canvas = Canvas(bitmap)
            decorView.draw(canvas)
            processFrame(bitmap)
        }
    }

    private fun processFrame(bitmap: Bitmap) {
        // Apply privacy masking
        val maskedBitmap = PrivacyMasker.maskSensitiveViews(bitmap)

        // Compress to JPEG
        val stream = ByteArrayOutputStream()
        maskedBitmap.compress(Bitmap.CompressFormat.JPEG, 50, stream)
        val imageData = stream.toByteArray()

        val frame = ReplayFrame(
            timestamp = System.currentTimeMillis(),
            screenName = NavigationState.currentScreen,
            imageData = imageData,
            width = bitmap.width,
            height = bitmap.height
        )

        frameBuffer.add(frame)

        // Limit buffer
        while (frameBuffer.size > MAX_FRAMES) {
            frameBuffer.poll()
        }

        // Recycle bitmap
        maskedBitmap.recycle()
    }

    fun recordTouch(type: TouchEvent.TouchType, x: Float, y: Float) {
        touchBuffer.add(TouchEvent(
            timestamp = System.currentTimeMillis(),
            type = type,
            x = x,
            y = y,
            screenName = NavigationState.currentScreen
        ))
    }

    fun exportSession(): SessionReplayData {
        return SessionReplayData(
            sessionId = SessionManager.sessionId,
            startTime = frameBuffer.firstOrNull()?.timestamp ?: System.currentTimeMillis(),
            endTime = frameBuffer.lastOrNull()?.timestamp ?: System.currentTimeMillis(),
            frames = frameBuffer.toList(),
            touches = touchBuffer.toList(),
            events = BreadcrumbManager.getBreadcrumbs()
        )
    }

    fun uploadSession(errorId: String? = null) {
        val data = exportSession()
        // Compress and upload in background
    }
}

data class SessionReplayData(
    val sessionId: String,
    val startTime: Long,
    val endTime: Long,
    val frames: List<SessionReplayCapture.ReplayFrame>,
    val touches: List<SessionReplayCapture.TouchEvent>,
    val events: List<Breadcrumb>
)
```

---

## Privacy & Masking

### Privacy Requirements

| Data Type | Action | Example |
|-----------|--------|---------|
| **PII** | Mask/redact | Names, emails, phone numbers |
| **Financial** | Mask completely | Credit cards, bank accounts |
| **Passwords** | Never capture | All password fields |
| **Health** | Mask/exclude | Medical records, diagnoses |
| **Custom sensitive** | User-defined masking | App-specific data |

### iOS Privacy Masker

```swift
// PrivacyMasker.swift
import UIKit

final class PrivacyMasker {
    static let shared = PrivacyMasker()

    // Views to always mask
    private var sensitiveViewTypes: [UIView.Type] = [
        UITextField.self  // Mask all text inputs by default
    ]

    // Specific views marked as sensitive
    private var sensitiveViews: Set<ObjectIdentifier> = []

    // MARK: - Configuration

    func registerSensitiveViewType(_ type: UIView.Type) {
        sensitiveViewTypes.append(type)
    }

    func markSensitive(_ view: UIView) {
        sensitiveViews.insert(ObjectIdentifier(view))
        view.accessibilityIdentifier = "replay-mask"
    }

    func unmarkSensitive(_ view: UIView) {
        sensitiveViews.remove(ObjectIdentifier(view))
    }

    // MARK: - Masking

    func maskSensitiveViews(in image: UIImage, window: UIWindow) -> UIImage {
        let sensitiveRects = findSensitiveRects(in: window)

        guard !sensitiveRects.isEmpty else { return image }

        UIGraphicsBeginImageContextWithOptions(image.size, false, image.scale)
        defer { UIGraphicsEndImageContext() }

        image.draw(at: .zero)

        guard let context = UIGraphicsGetCurrentContext() else { return image }

        // Mask sensitive areas
        context.setFillColor(UIColor.black.cgColor)
        for rect in sensitiveRects {
            context.fill(rect)
        }

        return UIGraphicsGetImageFromCurrentImageContext() ?? image
    }

    private func findSensitiveRects(in window: UIWindow) -> [CGRect] {
        var rects: [CGRect] = []

        func traverse(_ view: UIView) {
            // Check if view type is sensitive
            let isSensitiveType = sensitiveViewTypes.contains { type(of: view) == $0 }

            // Check if specifically marked
            let isMarkedSensitive = sensitiveViews.contains(ObjectIdentifier(view))

            // Check accessibility identifier
            let hasPrivacyMarker = view.accessibilityIdentifier?.contains("replay-mask") == true
                || view.accessibilityIdentifier?.contains("private") == true

            // Check for secure text entry
            let isSecureInput = (view as? UITextField)?.isSecureTextEntry == true

            if isSensitiveType || isMarkedSensitive || hasPrivacyMarker || isSecureInput {
                let rect = view.convert(view.bounds, to: window)
                rects.append(rect)
            }

            // Traverse children
            for subview in view.subviews {
                traverse(subview)
            }
        }

        traverse(window)
        return rects
    }
}

// MARK: - SwiftUI Extension

extension View {
    func replayMask() -> some View {
        self.accessibilityIdentifier("replay-mask")
    }
}

// Usage
struct CreditCardField: View {
    @State private var cardNumber = ""

    var body: some View {
        TextField("Card Number", text: $cardNumber)
            .replayMask() // Will be masked in session replay
    }
}
```

### Android Privacy Masker

```kotlin
// PrivacyMasker.kt
import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.Paint
import android.graphics.Rect
import android.view.View
import android.widget.EditText
import java.lang.ref.WeakReference

object PrivacyMasker {
    // View types to always mask
    private val sensitiveViewTypes = mutableListOf<Class<out View>>(
        EditText::class.java
    )

    // Specific views marked as sensitive
    private val sensitiveViews = mutableSetOf<WeakReference<View>>()

    private val maskPaint = Paint().apply {
        color = android.graphics.Color.BLACK
        style = Paint.Style.FILL
    }

    fun registerSensitiveViewType(type: Class<out View>) {
        sensitiveViewTypes.add(type)
    }

    fun markSensitive(view: View) {
        sensitiveViews.add(WeakReference(view))
        view.setTag(R.id.replay_mask_tag, true)
    }

    fun unmarkSensitive(view: View) {
        sensitiveViews.removeAll { it.get() == view }
        view.setTag(R.id.replay_mask_tag, null)
    }

    fun maskSensitiveViews(bitmap: Bitmap): Bitmap {
        val activity = ActivityLifecycleTracker.currentActivity ?: return bitmap
        val decorView = activity.window.decorView

        val sensitiveRects = findSensitiveRects(decorView)
        if (sensitiveRects.isEmpty()) return bitmap

        val mutableBitmap = bitmap.copy(Bitmap.Config.ARGB_8888, true)
        val canvas = Canvas(mutableBitmap)

        for (rect in sensitiveRects) {
            canvas.drawRect(rect, maskPaint)
        }

        return mutableBitmap
    }

    private fun findSensitiveRects(rootView: View): List<Rect> {
        val rects = mutableListOf<Rect>()
        val locationOnScreen = IntArray(2)

        fun traverse(view: View) {
            // Check if view type is sensitive
            val isSensitiveType = sensitiveViewTypes.any { it.isInstance(view) }

            // Check if specifically marked
            val isMarkedSensitive = view.getTag(R.id.replay_mask_tag) == true

            // Check for password input
            val isPasswordInput = (view as? EditText)?.inputType?.let {
                it and android.text.InputType.TYPE_TEXT_VARIATION_PASSWORD != 0 ||
                it and android.text.InputType.TYPE_TEXT_VARIATION_VISIBLE_PASSWORD != 0 ||
                it and android.text.InputType.TYPE_NUMBER_VARIATION_PASSWORD != 0
            } ?: false

            if (isSensitiveType || isMarkedSensitive || isPasswordInput) {
                view.getLocationOnScreen(locationOnScreen)
                rects.add(Rect(
                    locationOnScreen[0],
                    locationOnScreen[1],
                    locationOnScreen[0] + view.width,
                    locationOnScreen[1] + view.height
                ))
            }

            // Traverse children
            if (view is android.view.ViewGroup) {
                for (i in 0 until view.childCount) {
                    traverse(view.getChildAt(i))
                }
            }
        }

        traverse(rootView)
        return rects
    }
}

// Compose extension
fun Modifier.replayMask(): Modifier = this.then(
    Modifier.semantics {
        testTag = "replay-mask"
    }
)

// Usage
@Composable
fun CreditCardField() {
    var cardNumber by remember { mutableStateOf("") }

    TextField(
        value = cardNumber,
        onValueChange = { cardNumber = it },
        modifier = Modifier.replayMask() // Will be masked
    )
}
```

### Privacy Configuration

```swift
// ReplayPrivacyConfig.swift
struct ReplayPrivacyConfig {
    // Global masking rules
    var maskAllTextInputs: Bool = true
    var maskImages: Bool = false
    var maskWebViews: Bool = true

    // Selectors for masking
    var maskSelectors: [String] = [
        "[data-sensitive]",
        ".private",
        "#credit-card",
        ".pii"
    ]

    // Allowlist - never mask these even if matched
    var allowlistSelectors: [String] = [
        ".replay-visible",
        "[data-replay-visible]"
    ]

    // Text replacement
    var textMaskCharacter: Character = "•"
    var emailMaskPattern: String = "***@***.***"

    // Sampling for privacy
    var captureOnlyOnError: Bool = false
    var sampleRate: Double = 0.1 // 10% of sessions

    // Retention
    var maxRetentionDays: Int = 30
    var deleteOnUserRequest: Bool = true
}
```

---

## Integration Patterns

### Error-Linked Replay

```swift
// ErrorLinkedReplay.swift
extension CrashHandler {

    func handleCrash(_ crash: CrashReport) {
        // Capture replay segment around crash
        let replaySegment = SessionReplayCapture.shared.captureSegment(
            beforeError: 30, // 30 seconds before
            afterError: 0
        )

        // Link replay to crash
        crash.replayId = replaySegment.id

        // Upload both
        persistCrash(crash)
        SessionReplayCapture.shared.uploadSegment(replaySegment, linkedTo: crash.id)
    }
}

extension ErrorEvent {
    var replayId: String?

    func attachReplay() {
        let segment = SessionReplayCapture.shared.captureSegment(
            beforeError: 15,
            afterError: 5
        )
        self.replayId = segment.id
    }
}

// Viewing replay with error
struct ErrorDetailView {
    let error: ErrorEvent

    func showReplay() {
        if let replayId = error.replayId {
            ReplayPlayer.shared.play(replayId: replayId, seekTo: error.timestamp)
        }
    }
}
```

### Journey-Linked Replay

```swift
// JourneyLinkedReplay.swift
extension JourneyTracker {

    func onJourneyAbandoned(_ journey: JourneyState) {
        // Capture replay of abandoned journey
        let segment = SessionReplayCapture.shared.captureSegment(
            from: journey.startTime,
            to: Date()
        )

        // Link to journey
        Observability.trackEvent(
            name: "journey_abandoned_with_replay",
            properties: [
                "journey_id": journey.journeyId,
                "replay_id": segment.id,
                "funnel": journey.funnel.rawValue,
                "last_step": journey.completedSteps.last?.step.id ?? "start"
            ]
        )
    }
}
```

### Support Ticket Replay

```swift
// SupportReplay.swift
final class SupportReplayManager {

    func prepareReplayForSupport() -> SupportReplayPackage {
        // Get recent session replay
        let replay = SessionReplayCapture.shared.exportSession()

        // Get related data
        let breadcrumbs = BreadcrumbManager.shared.getBreadcrumbs()
        let recentErrors = ErrorReporter.shared.getRecentErrors(limit: 10)
        let journeyContext = JourneyTracker.getCurrentJourneyContext()

        return SupportReplayPackage(
            replay: replay,
            breadcrumbs: breadcrumbs,
            errors: recentErrors,
            journey: journeyContext,
            deviceInfo: DeviceInfo.current,
            userConsent: getUserConsentForSharing()
        )
    }

    func shareWithSupport(package: SupportReplayPackage) {
        guard package.userConsent else {
            // Ask for consent
            requestConsentThenShare(package)
            return
        }

        // Upload and get shareable link
        uploadPackage(package) { url in
            // Generate support ticket reference
            let ticketRef = generateTicketReference(url: url)
            // Show to user or auto-attach to ticket
        }
    }
}
```

---

## Performance Impact

### Optimization Strategies

```swift
// ReplayOptimization.swift
final class OptimizedReplayCapture {

    // Adaptive frame rate based on activity
    private var currentFPS: Double = 1.0
    private let minFPS: Double = 0.5
    private let maxFPS: Double = 4.0

    func adjustFrameRate(userActivity: UserActivityLevel) {
        switch userActivity {
        case .idle:
            currentFPS = minFPS
        case .scrolling, .animating:
            currentFPS = maxFPS
        case .normal:
            currentFPS = 2.0
        }
    }

    // Capture only when screen changes
    private var lastFrameHash: Int = 0

    func shouldCaptureFrame(currentHash: Int) -> Bool {
        if currentHash != lastFrameHash {
            lastFrameHash = currentHash
            return true
        }
        return false
    }

    // Compress frames efficiently
    func compressFrame(_ image: UIImage) -> Data? {
        // Use HEIC on iOS 11+ for better compression
        if #available(iOS 11.0, *) {
            return image.heicData(compressionQuality: 0.3)
        } else {
            return image.jpegData(compressionQuality: 0.4)
        }
    }

    // Battery-aware capture
    func shouldCapture() -> Bool {
        let batteryLevel = UIDevice.current.batteryLevel
        let isLowPower = ProcessInfo.processInfo.isLowPowerModeEnabled

        if isLowPower || batteryLevel < 0.2 {
            // Reduce or stop capture
            return false
        }

        return true
    }

    // Network-aware upload
    func shouldUpload() -> Bool {
        let networkType = NetworkMonitor.shared.currentType

        switch networkType {
        case .wifi:
            return true
        case .cellular:
            return ReplayConfig.uploadOnCellular
        case .offline:
            return false
        default:
            return true
        }
    }
}

enum UserActivityLevel {
    case idle, normal, scrolling, animating
}
```

### Resource Budgets

| Resource | Budget | Mitigation |
|----------|--------|------------|
| **CPU** | <5% average | Reduce FPS, skip frames |
| **Memory** | <50MB buffer | Limit frame count, compress |
| **Battery** | <2% per hour | Pause in low power mode |
| **Storage** | <100MB per session | Aggressive compression |
| **Network** | Upload on WiFi only | Queue for WiFi |

---

## Replay Analysis

### Automated Insights

```swift
// ReplayAnalyzer.swift
struct ReplayAnalyzer {

    static func analyze(_ replay: SessionReplayData) -> ReplayInsights {
        var insights = ReplayInsights()

        // Rage clicks detection
        insights.rageClicks = detectRageClicks(replay.touches)

        // Dead clicks (clicks with no response)
        insights.deadClicks = detectDeadClicks(replay.touches, replay.frames)

        // Scroll depth
        insights.scrollDepth = calculateScrollDepth(replay.touches)

        // Time on screen
        insights.screenTimes = calculateScreenTimes(replay.frames)

        // Error correlation
        insights.errorsWithContext = correlateErrors(replay.events)

        // Hesitation points (long pauses)
        insights.hesitationPoints = detectHesitation(replay.touches, replay.frames)

        return insights
    }

    private static func detectRageClicks(_ touches: [SessionReplayCapture.TouchEvent]) -> [RageClick] {
        var rageClicks: [RageClick] = []

        // Find clusters of rapid taps in same area
        let tapEvents = touches.filter { $0.type == .tap }
        var i = 0

        while i < tapEvents.count - 2 {
            let window = tapEvents[i..<min(i + 5, tapEvents.count)]
            let timeSpan = window.last!.timestamp.timeIntervalSince(window.first!.timestamp)

            if timeSpan < 2.0 && window.count >= 3 {
                // Check if taps are near each other (within 50pt)
                let locations = window.map { $0.location }
                if areLocationsClusterd(locations, threshold: 50) {
                    rageClicks.append(RageClick(
                        timestamp: tapEvents[i].timestamp,
                        location: tapEvents[i].location,
                        screen: tapEvents[i].screenName,
                        tapCount: window.count
                    ))
                    i += window.count
                    continue
                }
            }
            i += 1
        }

        return rageClicks
    }

    private static func detectHesitation(
        _ touches: [SessionReplayCapture.TouchEvent],
        _ frames: [SessionReplayCapture.ReplayFrame]
    ) -> [HesitationPoint] {
        var hesitations: [HesitationPoint] = []

        // Find gaps > 5 seconds between interactions
        for i in 1..<touches.count {
            let gap = touches[i].timestamp.timeIntervalSince(touches[i-1].timestamp)
            if gap > 5.0 {
                hesitations.append(HesitationPoint(
                    startTime: touches[i-1].timestamp,
                    endTime: touches[i].timestamp,
                    duration: gap,
                    screen: touches[i-1].screenName
                ))
            }
        }

        return hesitations
    }

    private static func areLocationsClusterd(_ locations: [CGPoint], threshold: CGFloat) -> Bool {
        guard let first = locations.first else { return false }
        return locations.allSatisfy { point in
            let distance = sqrt(pow(point.x - first.x, 2) + pow(point.y - first.y, 2))
            return distance < threshold
        }
    }
}

struct ReplayInsights {
    var rageClicks: [RageClick] = []
    var deadClicks: [DeadClick] = []
    var scrollDepth: [String: Double] = [:] // Screen -> max scroll %
    var screenTimes: [String: TimeInterval] = [:]
    var errorsWithContext: [ErrorWithContext] = []
    var hesitationPoints: [HesitationPoint] = []
}

struct RageClick {
    let timestamp: Date
    let location: CGPoint
    let screen: String
    let tapCount: Int
}

struct HesitationPoint {
    let startTime: Date
    let endTime: Date
    let duration: TimeInterval
    let screen: String
}
```

---

## Summary

| Component | Key Insight |
|-----------|-------------|
| **Capture** | Screenshot-based is simplest; 1-4 FPS sufficient |
| **Privacy** | Mask PII, passwords, financial data by default |
| **Linking** | Connect replays to errors, journeys, support tickets |
| **Performance** | <5% CPU, pause on low battery, WiFi-only upload |
| **Analysis** | Detect rage clicks, hesitation, dead clicks |

### Sampling Strategy

| Scenario | Sample Rate | Duration |
|----------|-------------|----------|
| All sessions | 5-10% | Full session |
| Error sessions | 100% | 30s before error |
| Slow sessions | 100% | Full session |
| Support request | 100% | On-demand |

---

## Integration Points

- **[crash-reporting.md](crash-reporting.md)** - Error-linked replay
- **[user-journeys.md](user-journeys.md)** - Journey replay on abandonment
- **[mobile-challenges.md](mobile-challenges.md)** - Offline replay storage
- **[platforms/](platforms/)** - Vendor-specific replay setup
