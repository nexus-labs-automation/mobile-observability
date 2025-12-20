# Mobile-Specific Challenges

Solving unique mobile observability challenges: offline-first, device fragmentation, battery constraints, and client-backend correlation.

## Table of Contents
1. [Offline-First Observability](#offline-first-observability)
2. [Device Fragmentation](#device-fragmentation)
3. [Battery & Resource Constraints](#battery--resource-constraints)
4. [Client-Backend Correlation](#client-backend-correlation)
5. [App Lifecycle Complexity](#app-lifecycle-complexity)
6. [Network Variability](#network-variability)

---

## Offline-First Observability

### The Offline Challenge

Mobile apps operate in environments where connectivity is unreliable:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Connectivity Timeline                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Online │ Offline │ Weak Signal │ Online │ Airplane │ Online   │
│  ──────   ───────   ──────────   ──────   ────────   ──────    │
│    │        │           │           │         │          │       │
│  Events  Events      Events      Flush     Events    Flush     │
│  sent    queued      queued      queue     queued    queue     │
│                                                                  │
│  Challenge: Don't lose telemetry, maintain chronological order   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### iOS Offline Queue

```swift
// OfflineQueue.swift
import Foundation
import SQLite3

final class OfflineQueue {
    static let shared = OfflineQueue()

    private var db: OpaquePointer?
    private let queue = DispatchQueue(label: "offline.queue", qos: .utility)
    private let maxQueueSize = 10000
    private let maxAgeSeconds: TimeInterval = 7 * 24 * 3600 // 7 days

    init() {
        setupDatabase()
    }

    // MARK: - Database Setup

    private func setupDatabase() {
        let dbPath = FileManager.default.urls(for: .applicationSupportDirectory, in: .userDomainMask)
            .first!.appendingPathComponent("observability_queue.db").path

        if sqlite3_open(dbPath, &db) == SQLITE_OK {
            let createTable = """
                CREATE TABLE IF NOT EXISTS events (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    type TEXT NOT NULL,
                    payload BLOB NOT NULL,
                    timestamp REAL NOT NULL,
                    retry_count INTEGER DEFAULT 0,
                    priority INTEGER DEFAULT 0
                );
                CREATE INDEX IF NOT EXISTS idx_timestamp ON events(timestamp);
                CREATE INDEX IF NOT EXISTS idx_priority ON events(priority DESC, timestamp ASC);
            """
            sqlite3_exec(db, createTable, nil, nil, nil)
        }
    }

    // MARK: - Enqueue

    func enqueue(_ event: QueuedEvent) {
        queue.async {
            self.insertEvent(event)
            self.pruneIfNeeded()
        }
    }

    func enqueueBatch(_ events: [QueuedEvent]) {
        queue.async {
            sqlite3_exec(self.db, "BEGIN TRANSACTION", nil, nil, nil)
            for event in events {
                self.insertEvent(event)
            }
            sqlite3_exec(self.db, "COMMIT", nil, nil, nil)
            self.pruneIfNeeded()
        }
    }

    private func insertEvent(_ event: QueuedEvent) {
        let sql = "INSERT INTO events (type, payload, timestamp, priority) VALUES (?, ?, ?, ?)"
        var stmt: OpaquePointer?

        if sqlite3_prepare_v2(db, sql, -1, &stmt, nil) == SQLITE_OK {
            sqlite3_bind_text(stmt, 1, event.type, -1, nil)
            sqlite3_bind_blob(stmt, 2, (event.payload as NSData).bytes, Int32(event.payload.count), nil)
            sqlite3_bind_double(stmt, 3, event.timestamp.timeIntervalSince1970)
            sqlite3_bind_int(stmt, 4, Int32(event.priority.rawValue))
            sqlite3_step(stmt)
        }
        sqlite3_finalize(stmt)
    }

    // MARK: - Dequeue

    func dequeue(limit: Int = 100) -> [QueuedEvent] {
        var events: [QueuedEvent] = []

        queue.sync {
            let sql = """
                SELECT id, type, payload, timestamp, retry_count, priority
                FROM events
                ORDER BY priority DESC, timestamp ASC
                LIMIT ?
            """
            var stmt: OpaquePointer?

            if sqlite3_prepare_v2(db, sql, -1, &stmt, nil) == SQLITE_OK {
                sqlite3_bind_int(stmt, 1, Int32(limit))

                while sqlite3_step(stmt) == SQLITE_ROW {
                    let id = sqlite3_column_int64(stmt, 0)
                    let type = String(cString: sqlite3_column_text(stmt, 1))
                    let payloadPtr = sqlite3_column_blob(stmt, 2)
                    let payloadLen = sqlite3_column_bytes(stmt, 2)
                    let timestamp = sqlite3_column_double(stmt, 3)
                    let retryCount = sqlite3_column_int(stmt, 4)
                    let priority = sqlite3_column_int(stmt, 5)

                    let payload = Data(bytes: payloadPtr!, count: Int(payloadLen))

                    events.append(QueuedEvent(
                        id: id,
                        type: type,
                        payload: payload,
                        timestamp: Date(timeIntervalSince1970: timestamp),
                        retryCount: Int(retryCount),
                        priority: EventPriority(rawValue: Int(priority)) ?? .normal
                    ))
                }
            }
            sqlite3_finalize(stmt)
        }

        return events
    }

    func markSent(ids: [Int64]) {
        guard !ids.isEmpty else { return }

        queue.async {
            let placeholders = ids.map { _ in "?" }.joined(separator: ",")
            let sql = "DELETE FROM events WHERE id IN (\(placeholders))"
            var stmt: OpaquePointer?

            if sqlite3_prepare_v2(self.db, sql, -1, &stmt, nil) == SQLITE_OK {
                for (index, id) in ids.enumerated() {
                    sqlite3_bind_int64(stmt, Int32(index + 1), id)
                }
                sqlite3_step(stmt)
            }
            sqlite3_finalize(stmt)
        }
    }

    func markFailed(ids: [Int64]) {
        guard !ids.isEmpty else { return }

        queue.async {
            let placeholders = ids.map { _ in "?" }.joined(separator: ",")
            let sql = "UPDATE events SET retry_count = retry_count + 1 WHERE id IN (\(placeholders))"
            var stmt: OpaquePointer?

            if sqlite3_prepare_v2(self.db, sql, -1, &stmt, nil) == SQLITE_OK {
                for (index, id) in ids.enumerated() {
                    sqlite3_bind_int64(stmt, Int32(index + 1), id)
                }
                sqlite3_step(stmt)
            }
            sqlite3_finalize(stmt)

            // Remove events that failed too many times
            self.removeFailedEvents()
        }
    }

    // MARK: - Maintenance

    private func pruneIfNeeded() {
        // Remove old events
        let cutoff = Date().timeIntervalSince1970 - maxAgeSeconds
        sqlite3_exec(db, "DELETE FROM events WHERE timestamp < \(cutoff)", nil, nil, nil)

        // Keep queue size manageable
        let countSql = "SELECT COUNT(*) FROM events"
        var stmt: OpaquePointer?
        if sqlite3_prepare_v2(db, countSql, -1, &stmt, nil) == SQLITE_OK {
            if sqlite3_step(stmt) == SQLITE_ROW {
                let count = sqlite3_column_int(stmt, 0)
                if count > maxQueueSize {
                    // Remove oldest low-priority events
                    let deleteCount = Int(count) - maxQueueSize
                    sqlite3_exec(db, """
                        DELETE FROM events WHERE id IN (
                            SELECT id FROM events
                            ORDER BY priority ASC, timestamp ASC
                            LIMIT \(deleteCount)
                        )
                    """, nil, nil, nil)
                }
            }
        }
        sqlite3_finalize(stmt)
    }

    private func removeFailedEvents() {
        sqlite3_exec(db, "DELETE FROM events WHERE retry_count > 5", nil, nil, nil)
    }

    // MARK: - Stats

    func getQueueStats() -> QueueStats {
        var stats = QueueStats()

        queue.sync {
            var stmt: OpaquePointer?

            // Total count
            if sqlite3_prepare_v2(db, "SELECT COUNT(*) FROM events", -1, &stmt, nil) == SQLITE_OK {
                if sqlite3_step(stmt) == SQLITE_ROW {
                    stats.totalCount = Int(sqlite3_column_int(stmt, 0))
                }
            }
            sqlite3_finalize(stmt)

            // Oldest event
            if sqlite3_prepare_v2(db, "SELECT MIN(timestamp) FROM events", -1, &stmt, nil) == SQLITE_OK {
                if sqlite3_step(stmt) == SQLITE_ROW {
                    let timestamp = sqlite3_column_double(stmt, 0)
                    if timestamp > 0 {
                        stats.oldestEvent = Date(timeIntervalSince1970: timestamp)
                    }
                }
            }
            sqlite3_finalize(stmt)

            // By type
            if sqlite3_prepare_v2(db, "SELECT type, COUNT(*) FROM events GROUP BY type", -1, &stmt, nil) == SQLITE_OK {
                while sqlite3_step(stmt) == SQLITE_ROW {
                    let type = String(cString: sqlite3_column_text(stmt, 0))
                    let count = Int(sqlite3_column_int(stmt, 1))
                    stats.countByType[type] = count
                }
            }
            sqlite3_finalize(stmt)
        }

        return stats
    }
}

struct QueuedEvent {
    var id: Int64 = 0
    let type: String
    let payload: Data
    let timestamp: Date
    var retryCount: Int = 0
    let priority: EventPriority
}

enum EventPriority: Int {
    case low = 0
    case normal = 1
    case high = 2
    case critical = 3 // Crashes, etc.
}

struct QueueStats {
    var totalCount: Int = 0
    var oldestEvent: Date?
    var countByType: [String: Int] = [:]
}
```

### Android Offline Queue

```kotlin
// OfflineQueue.kt
import android.content.Context
import androidx.room.*
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

@Entity(tableName = "events")
data class QueuedEvent(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val type: String,
    val payload: ByteArray,
    val timestamp: Long = System.currentTimeMillis(),
    val retryCount: Int = 0,
    val priority: Int = EventPriority.NORMAL.ordinal
)

enum class EventPriority { LOW, NORMAL, HIGH, CRITICAL }

@Dao
interface EventDao {
    @Insert
    suspend fun insert(event: QueuedEvent)

    @Insert
    suspend fun insertAll(events: List<QueuedEvent>)

    @Query("""
        SELECT * FROM events
        ORDER BY priority DESC, timestamp ASC
        LIMIT :limit
    """)
    suspend fun dequeue(limit: Int = 100): List<QueuedEvent>

    @Query("DELETE FROM events WHERE id IN (:ids)")
    suspend fun delete(ids: List<Long>)

    @Query("UPDATE events SET retryCount = retryCount + 1 WHERE id IN (:ids)")
    suspend fun incrementRetry(ids: List<Long>)

    @Query("DELETE FROM events WHERE retryCount > 5")
    suspend fun removeFailedEvents()

    @Query("DELETE FROM events WHERE timestamp < :cutoff")
    suspend fun removeOldEvents(cutoff: Long)

    @Query("SELECT COUNT(*) FROM events")
    suspend fun count(): Int

    @Query("""
        DELETE FROM events WHERE id IN (
            SELECT id FROM events
            ORDER BY priority ASC, timestamp ASC
            LIMIT :count
        )
    """)
    suspend fun removeOldest(count: Int)

    @Query("SELECT type, COUNT(*) as count FROM events GROUP BY type")
    suspend fun countByType(): List<TypeCount>
}

data class TypeCount(val type: String, val count: Int)

@Database(entities = [QueuedEvent::class], version = 1)
abstract class OfflineDatabase : RoomDatabase() {
    abstract fun eventDao(): EventDao

    companion object {
        @Volatile
        private var INSTANCE: OfflineDatabase? = null

        fun getInstance(context: Context): OfflineDatabase {
            return INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    OfflineDatabase::class.java,
                    "observability_queue.db"
                ).build().also { INSTANCE = it }
            }
        }
    }
}

class OfflineQueue(context: Context) {
    private val dao = OfflineDatabase.getInstance(context).eventDao()
    private val maxQueueSize = 10000
    private val maxAgeMs = 7 * 24 * 3600 * 1000L // 7 days

    suspend fun enqueue(event: QueuedEvent) = withContext(Dispatchers.IO) {
        dao.insert(event)
        pruneIfNeeded()
    }

    suspend fun enqueueBatch(events: List<QueuedEvent>) = withContext(Dispatchers.IO) {
        dao.insertAll(events)
        pruneIfNeeded()
    }

    suspend fun dequeue(limit: Int = 100): List<QueuedEvent> = withContext(Dispatchers.IO) {
        dao.dequeue(limit)
    }

    suspend fun markSent(ids: List<Long>) = withContext(Dispatchers.IO) {
        dao.delete(ids)
    }

    suspend fun markFailed(ids: List<Long>) = withContext(Dispatchers.IO) {
        dao.incrementRetry(ids)
        dao.removeFailedEvents()
    }

    private suspend fun pruneIfNeeded() {
        val cutoff = System.currentTimeMillis() - maxAgeMs
        dao.removeOldEvents(cutoff)

        val count = dao.count()
        if (count > maxQueueSize) {
            dao.removeOldest(count - maxQueueSize)
        }
    }

    suspend fun getStats(): QueueStats = withContext(Dispatchers.IO) {
        QueueStats(
            totalCount = dao.count(),
            countByType = dao.countByType().associate { it.type to it.count }
        )
    }
}

data class QueueStats(
    val totalCount: Int = 0,
    val countByType: Map<String, Int> = emptyMap()
)
```

### Flush Strategy

```swift
// FlushManager.swift
final class FlushManager {
    static let shared = FlushManager()

    private var flushTimer: Timer?
    private let queue = OfflineQueue.shared

    private var flushInterval: TimeInterval = 30 // seconds
    private var batchSize = 100

    func start() {
        // Periodic flush
        flushTimer = Timer.scheduledTimer(withTimeInterval: flushInterval, repeats: true) { [weak self] _ in
            self?.flushIfOnline()
        }

        // Flush on network change
        NetworkMonitor.shared.onStatusChange { [weak self] status in
            if status == .online {
                self?.flush()
            }
        }

        // Flush when app enters background
        NotificationCenter.default.addObserver(
            forName: UIApplication.didEnterBackgroundNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.flushInBackground()
        }
    }

    func flushIfOnline() {
        guard NetworkMonitor.shared.isOnline else { return }
        flush()
    }

    func flush() {
        let events = queue.dequeue(limit: batchSize)
        guard !events.isEmpty else { return }

        // Group by type for efficient batch uploads
        let grouped = Dictionary(grouping: events) { $0.type }

        for (type, typeEvents) in grouped {
            uploadBatch(type: type, events: typeEvents) { success in
                if success {
                    self.queue.markSent(ids: typeEvents.map { $0.id })
                } else {
                    self.queue.markFailed(ids: typeEvents.map { $0.id })
                }
            }
        }
    }

    private func flushInBackground() {
        // Request background time for flush
        var backgroundTask: UIBackgroundTaskIdentifier = .invalid
        backgroundTask = UIApplication.shared.beginBackgroundTask {
            UIApplication.shared.endBackgroundTask(backgroundTask)
        }

        flush()

        // End background task after flush
        DispatchQueue.main.asyncAfter(deadline: .now() + 5) {
            UIApplication.shared.endBackgroundTask(backgroundTask)
        }
    }

    private func uploadBatch(type: String, events: [QueuedEvent], completion: @escaping (Bool) -> Void) {
        // Upload to observability backend
    }
}
```

---

## Device Fragmentation

### The Fragmentation Challenge

```
iOS Devices              Android Devices
────────────             ───────────────
~30 active models        ~24,000+ active models
~5 OS versions           ~10+ OS versions
Unified hardware         Varied hardware capabilities
Predictable behavior     Manufacturer customizations
```

### Device Context Collection

```swift
// iOS Device Context
struct DeviceContext: Codable {
    let platform: String = "ios"
    let osVersion: String
    let deviceModel: String
    let deviceModelRaw: String
    let screenSize: CGSize
    let screenScale: CGFloat
    let isSimulator: Bool
    let processorCount: Int
    let physicalMemoryGB: Double
    let isLowPowerMode: Bool
    let thermalState: String
    let batteryLevel: Float
    let batteryState: String
    let orientation: String
    let locale: String
    let timezone: String
    let carrier: String?
    let appVersion: String
    let buildNumber: String

    static var current: DeviceContext {
        let device = UIDevice.current
        let screen = UIScreen.main

        return DeviceContext(
            osVersion: device.systemVersion,
            deviceModel: mapDeviceModel(device.modelIdentifier),
            deviceModelRaw: device.modelIdentifier,
            screenSize: screen.bounds.size,
            screenScale: screen.scale,
            isSimulator: ProcessInfo.processInfo.environment["SIMULATOR_DEVICE_NAME"] != nil,
            processorCount: ProcessInfo.processInfo.processorCount,
            physicalMemoryGB: Double(ProcessInfo.processInfo.physicalMemory) / 1_073_741_824,
            isLowPowerMode: ProcessInfo.processInfo.isLowPowerModeEnabled,
            thermalState: mapThermalState(ProcessInfo.processInfo.thermalState),
            batteryLevel: device.batteryLevel,
            batteryState: mapBatteryState(device.batteryState),
            orientation: mapOrientation(device.orientation),
            locale: Locale.current.identifier,
            timezone: TimeZone.current.identifier,
            carrier: CTCarrier()?.carrierName,
            appVersion: Bundle.main.appVersion,
            buildNumber: Bundle.main.buildNumber
        )
    }

    private static func mapDeviceModel(_ identifier: String) -> String {
        let mapping: [String: String] = [
            "iPhone16,1": "iPhone 15 Pro",
            "iPhone16,2": "iPhone 15 Pro Max",
            "iPhone15,4": "iPhone 15",
            "iPhone15,5": "iPhone 15 Plus",
            // ... add more mappings
        ]
        return mapping[identifier] ?? identifier
    }
}

extension UIDevice {
    var modelIdentifier: String {
        var systemInfo = utsname()
        uname(&systemInfo)
        let machineMirror = Mirror(reflecting: systemInfo.machine)
        return machineMirror.children.reduce("") { identifier, element in
            guard let value = element.value as? Int8, value != 0 else { return identifier }
            return identifier + String(UnicodeScalar(UInt8(value)))
        }
    }
}
```

```kotlin
// Android Device Context
data class DeviceContext(
    val platform: String = "android",
    val osVersion: String = Build.VERSION.RELEASE,
    val sdkVersion: Int = Build.VERSION.SDK_INT,
    val deviceModel: String = Build.MODEL,
    val deviceManufacturer: String = Build.MANUFACTURER,
    val deviceBrand: String = Build.BRAND,
    val screenWidthPx: Int,
    val screenHeightPx: Int,
    val screenDensity: Float,
    val isEmulator: Boolean,
    val processorCount: Int = Runtime.getRuntime().availableProcessors(),
    val totalMemoryMB: Long,
    val isLowRamDevice: Boolean,
    val batteryLevel: Float,
    val isCharging: Boolean,
    val isPowerSaveMode: Boolean,
    val orientation: String,
    val locale: String = Locale.getDefault().toString(),
    val timezone: String = TimeZone.getDefault().id,
    val carrier: String?,
    val appVersion: String = BuildConfig.VERSION_NAME,
    val buildNumber: Int = BuildConfig.VERSION_CODE
) {
    companion object {
        fun current(context: Context): DeviceContext {
            val displayMetrics = context.resources.displayMetrics
            val activityManager = context.getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager
            val batteryManager = context.getSystemService(Context.BATTERY_SERVICE) as BatteryManager
            val powerManager = context.getSystemService(Context.POWER_SERVICE) as PowerManager
            val telephonyManager = context.getSystemService(Context.TELEPHONY_SERVICE) as? TelephonyManager

            val memoryInfo = ActivityManager.MemoryInfo()
            activityManager.getMemoryInfo(memoryInfo)

            return DeviceContext(
                screenWidthPx = displayMetrics.widthPixels,
                screenHeightPx = displayMetrics.heightPixels,
                screenDensity = displayMetrics.density,
                isEmulator = isEmulator(),
                totalMemoryMB = memoryInfo.totalMem / (1024 * 1024),
                isLowRamDevice = activityManager.isLowRamDevice,
                batteryLevel = batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY) / 100f,
                isCharging = batteryManager.isCharging,
                isPowerSaveMode = powerManager.isPowerSaveMode,
                orientation = when (context.resources.configuration.orientation) {
                    Configuration.ORIENTATION_LANDSCAPE -> "landscape"
                    Configuration.ORIENTATION_PORTRAIT -> "portrait"
                    else -> "unknown"
                },
                carrier = telephonyManager?.networkOperatorName
            )
        }

        private fun isEmulator(): Boolean {
            return Build.FINGERPRINT.startsWith("generic") ||
                   Build.FINGERPRINT.startsWith("unknown") ||
                   Build.MODEL.contains("google_sdk") ||
                   Build.MODEL.contains("Emulator") ||
                   Build.MODEL.contains("Android SDK built for x86") ||
                   Build.MANUFACTURER.contains("Genymotion") ||
                   Build.PRODUCT.contains("sdk_google") ||
                   Build.PRODUCT.contains("vbox86p")
        }
    }
}
```

### Device-Aware Sampling

```swift
// DeviceAwareSampling.swift
struct DeviceAwareSampler {

    static func getSampleRate(for context: DeviceContext) -> Double {
        var rate = 0.1 // Base 10%

        // Increase for newer devices (more capable)
        if context.physicalMemoryGB >= 6 {
            rate = min(rate * 1.5, 1.0)
        }

        // Decrease for low-power mode
        if context.isLowPowerMode {
            rate *= 0.5
        }

        // Decrease for thermal throttling
        if context.thermalState == "serious" || context.thermalState == "critical" {
            rate *= 0.25
        }

        // Decrease for low battery
        if context.batteryLevel < 0.2 && context.batteryState != "charging" {
            rate *= 0.5
        }

        return rate
    }

    static func shouldSample(for context: DeviceContext) -> Bool {
        let rate = getSampleRate(for: context)
        return Double.random(in: 0...1) < rate
    }
}
```

---

## Battery & Resource Constraints

### Battery-Aware Telemetry

```swift
// BatteryAwareTelemetry.swift
final class BatteryAwareTelemetry {
    static let shared = BatteryAwareTelemetry()

    private var currentMode: TelemetryMode = .normal

    enum TelemetryMode {
        case full       // All telemetry
        case normal     // Standard sampling
        case reduced    // Critical only
        case minimal    // Errors only
    }

    func updateMode() {
        let batteryLevel = UIDevice.current.batteryLevel
        let isCharging = UIDevice.current.batteryState == .charging || UIDevice.current.batteryState == .full
        let isLowPower = ProcessInfo.processInfo.isLowPowerModeEnabled
        let thermalState = ProcessInfo.processInfo.thermalState

        if isCharging {
            currentMode = .full
        } else if isLowPower || thermalState == .critical {
            currentMode = .minimal
        } else if batteryLevel < 0.1 || thermalState == .serious {
            currentMode = .reduced
        } else if batteryLevel < 0.2 {
            currentMode = .reduced
        } else {
            currentMode = .normal
        }

        applyMode()
    }

    private func applyMode() {
        switch currentMode {
        case .full:
            SessionReplayCapture.shared.setFPS(4)
            FlushManager.shared.setInterval(15)
            Sampler.shared.setRate(1.0)

        case .normal:
            SessionReplayCapture.shared.setFPS(2)
            FlushManager.shared.setInterval(30)
            Sampler.shared.setRate(0.1)

        case .reduced:
            SessionReplayCapture.shared.setFPS(0.5)
            FlushManager.shared.setInterval(60)
            Sampler.shared.setRate(0.05)

        case .minimal:
            SessionReplayCapture.shared.stop()
            FlushManager.shared.setInterval(120)
            Sampler.shared.setRate(0) // Only errors
        }
    }

    func shouldCollect(_ eventType: EventType) -> Bool {
        switch currentMode {
        case .full:
            return true
        case .normal:
            return true
        case .reduced:
            return eventType.priority >= .high
        case .minimal:
            return eventType == .error || eventType == .crash
        }
    }
}

enum EventType {
    case debug, info, metric, trace, error, crash

    var priority: EventPriority {
        switch self {
        case .debug: return .low
        case .info, .metric: return .normal
        case .trace: return .normal
        case .error: return .high
        case .crash: return .critical
        }
    }
}
```

### Memory Pressure Handling

```swift
// MemoryPressureHandler.swift
final class MemoryPressureHandler {
    static let shared = MemoryPressureHandler()

    private var dispatchSource: DispatchSourceMemoryPressure?

    func start() {
        dispatchSource = DispatchSource.makeMemoryPressureSource(eventMask: [.warning, .critical], queue: .main)

        dispatchSource?.setEventHandler { [weak self] in
            guard let self = self else { return }

            let event = self.dispatchSource?.data ?? []

            if event.contains(.critical) {
                self.handleCriticalMemoryPressure()
            } else if event.contains(.warning) {
                self.handleMemoryWarning()
            }
        }

        dispatchSource?.resume()

        // Also handle UIKit memory warnings
        NotificationCenter.default.addObserver(
            forName: UIApplication.didReceiveMemoryWarningNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.handleMemoryWarning()
        }
    }

    private func handleMemoryWarning() {
        // Reduce telemetry buffers
        BreadcrumbManager.shared.reduceBuffer(to: 50)
        SessionReplayCapture.shared.reduceFrameBuffer(to: 100)

        // Log for debugging
        Observability.captureMessage(
            "Memory warning - reduced telemetry buffers",
            level: .warning,
            extras: ["free_memory_mb": DeviceInfo.freeMemoryMB]
        )
    }

    private func handleCriticalMemoryPressure() {
        // Aggressive reduction
        BreadcrumbManager.shared.reduceBuffer(to: 20)
        SessionReplayCapture.shared.stop()

        // Flush what we have
        FlushManager.shared.flush()

        Observability.captureMessage(
            "Critical memory pressure - stopped session replay",
            level: .error,
            extras: ["free_memory_mb": DeviceInfo.freeMemoryMB]
        )
    }
}
```

---

## Client-Backend Correlation

### Distributed Tracing Headers

```swift
// DistributedTracing.swift
struct DistributedTracing {

    // W3C Trace Context headers
    static func getTraceHeaders() -> [String: String] {
        guard let context = TraceContext.current else {
            return [:]
        }

        // traceparent: version-traceid-parentid-flags
        let traceparent = "00-\(context.traceId)-\(context.spanId)-01"

        // tracestate: vendor-specific data
        let tracestate = "app=ios,session=\(SessionManager.shared.sessionId)"

        return [
            "traceparent": traceparent,
            "tracestate": tracestate,
            "X-Request-Id": UUID().uuidString,
            "X-Session-Id": SessionManager.shared.sessionId,
            "X-Device-Id": DeviceInfo.deviceId,
            "X-App-Version": Bundle.main.appVersion
        ]
    }

    // Parse incoming trace context
    static func parseTraceParent(_ header: String) -> (traceId: String, parentId: String)? {
        let parts = header.split(separator: "-")
        guard parts.count == 4 else { return nil }

        let traceId = String(parts[1])
        let parentId = String(parts[2])

        return (traceId, parentId)
    }
}

// URLSession integration
extension URLRequest {
    mutating func addTraceHeaders() {
        let headers = DistributedTracing.getTraceHeaders()
        for (key, value) in headers {
            self.setValue(value, forHTTPHeaderField: key)
        }
    }
}

// OkHttp integration (Android/Kotlin)
class TracingInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val original = chain.request()

        val traceId = TraceContext.current?.traceId ?: generateTraceId()
        val spanId = TraceContext.current?.spanId ?: generateSpanId()

        val traceparent = "00-$traceId-$spanId-01"
        val tracestate = "app=android,session=${SessionManager.sessionId}"

        val request = original.newBuilder()
            .header("traceparent", traceparent)
            .header("tracestate", tracestate)
            .header("X-Request-Id", UUID.randomUUID().toString())
            .header("X-Session-Id", SessionManager.sessionId)
            .header("X-Device-Id", DeviceInfo.deviceId)
            .header("X-App-Version", BuildConfig.VERSION_NAME)
            .build()

        return chain.proceed(request)
    }
}
```

### Backend Log Correlation

```swift
// Backend can correlate using headers:

// Example backend log entry (structured):
// {
//   "timestamp": "2024-01-15T10:30:00Z",
//   "level": "error",
//   "message": "Payment processing failed",
//   "trace_id": "abc123...",           // From traceparent
//   "span_id": "def456...",            // From traceparent
//   "session_id": "session_xyz",       // From X-Session-Id
//   "device_id": "device_123",         // From X-Device-Id
//   "app_version": "2.1.0",            // From X-App-Version
//   "request_id": "req_789"            // From X-Request-Id
// }

// Client can now search: "Show me all backend errors for session_xyz"
// Backend can correlate: "This error happened 200ms after client API call"
```

---

## App Lifecycle Complexity

### State Transitions

```
┌─────────────────────────────────────────────────────────────────┐
│                     iOS App Lifecycle                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Launch ──▶ Active ──▶ Inactive ──▶ Background ──▶ Suspended    │
│    │          │           │             │              │         │
│    │          │           │             │              │         │
│    │          │           │             │              ▼         │
│    │          │           │             └──────▶ Terminated      │
│    │          │           │                                      │
│    │          └───────────┴──────────────────▶ Active           │
│    │                                                             │
│    └─ Cold start metrics                                         │
│       Active: Full telemetry                                     │
│       Background: Reduced/queued                                 │
│       Suspended: No telemetry                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Lifecycle-Aware Telemetry

```swift
// LifecycleObserver.swift
final class LifecycleObserver {
    static let shared = LifecycleObserver()

    private var currentState: AppState = .launching

    enum AppState: String {
        case launching, active, inactive, background, suspended
    }

    func start() {
        NotificationCenter.default.addObserver(
            forName: UIApplication.didBecomeActiveNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.transitionTo(.active)
        }

        NotificationCenter.default.addObserver(
            forName: UIApplication.willResignActiveNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.transitionTo(.inactive)
        }

        NotificationCenter.default.addObserver(
            forName: UIApplication.didEnterBackgroundNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.transitionTo(.background)
        }

        NotificationCenter.default.addObserver(
            forName: UIApplication.willEnterForegroundNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            // Resuming from background
            self?.onResumingFromBackground()
        }

        NotificationCenter.default.addObserver(
            forName: UIApplication.willTerminateNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            self?.onTermination()
        }
    }

    private func transitionTo(_ newState: AppState) {
        let previousState = currentState
        currentState = newState

        // Breadcrumb
        BreadcrumbManager.shared.systemEvent(
            "App state: \(previousState.rawValue) → \(newState.rawValue)"
        )

        // Adjust telemetry
        switch newState {
        case .active:
            onBecameActive()
        case .background:
            onEnteredBackground()
        case .inactive, .launching, .suspended:
            break
        }

        // Metric
        Observability.recordMetric(
            name: "app.state_transition",
            value: 1,
            tags: [
                "from": previousState.rawValue,
                "to": newState.rawValue
            ]
        )
    }

    private func onBecameActive() {
        // Resume full telemetry
        SessionReplayCapture.shared.startCapture()
        FlushManager.shared.flush()
        BatteryAwareTelemetry.shared.updateMode()
    }

    private func onEnteredBackground() {
        // Reduce telemetry
        SessionReplayCapture.shared.stopCapture()

        // Flush critical data
        FlushManager.shared.flush()

        // Save session state
        SessionManager.shared.saveState()
    }

    private func onResumingFromBackground() {
        // Check if session should continue or start new
        if SessionManager.shared.shouldStartNewSession() {
            SessionManager.shared.startNewSession()
        } else {
            SessionManager.shared.resumeSession()
        }
    }

    private func onTermination() {
        // Final flush
        FlushManager.shared.flushSync()

        // Record termination
        Observability.trackEvent(
            name: "app_terminated",
            properties: [
                "session_duration_seconds": SessionManager.shared.sessionDuration,
                "screens_visited": NavigationState.shared.screensVisited.count
            ]
        )
    }
}
```

---

## Network Variability

### Network Monitoring

```swift
// NetworkMonitor.swift
import Network

final class NetworkMonitor {
    static let shared = NetworkMonitor()

    private let monitor = NWPathMonitor()
    private let queue = DispatchQueue(label: "network.monitor")

    var isOnline: Bool = true
    var connectionType: ConnectionType = .unknown

    private var statusChangeHandlers: [(ConnectionStatus) -> Void] = []

    enum ConnectionType: String {
        case wifi, cellular, ethernet, unknown, offline
    }

    enum ConnectionStatus {
        case online(ConnectionType)
        case offline
    }

    func start() {
        monitor.pathUpdateHandler = { [weak self] path in
            guard let self = self else { return }

            let wasOnline = self.isOnline
            self.isOnline = path.status == .satisfied

            if path.usesInterfaceType(.wifi) {
                self.connectionType = .wifi
            } else if path.usesInterfaceType(.cellular) {
                self.connectionType = .cellular
            } else if path.usesInterfaceType(.wiredEthernet) {
                self.connectionType = .ethernet
            } else if path.status == .unsatisfied {
                self.connectionType = .offline
            } else {
                self.connectionType = .unknown
            }

            // Notify handlers
            let status: ConnectionStatus = self.isOnline ? .online(self.connectionType) : .offline

            DispatchQueue.main.async {
                for handler in self.statusChangeHandlers {
                    handler(status)
                }
            }

            // Breadcrumb on change
            if wasOnline != self.isOnline {
                BreadcrumbManager.shared.systemEvent(
                    "Network: \(self.isOnline ? "online" : "offline") (\(self.connectionType.rawValue))"
                )
            }

            // Metric
            Observability.recordMetric(
                name: "network.status_change",
                value: 1,
                tags: [
                    "online": String(self.isOnline),
                    "type": self.connectionType.rawValue
                ]
            )
        }

        monitor.start(queue: queue)
    }

    func onStatusChange(_ handler: @escaping (ConnectionStatus) -> Void) {
        statusChangeHandlers.append(handler)
    }
}
```

### Adaptive Upload Strategy

```swift
// AdaptiveUploader.swift
final class AdaptiveUploader {
    static let shared = AdaptiveUploader()

    private var uploadStrategy: UploadStrategy = .immediate

    enum UploadStrategy {
        case immediate      // Upload now
        case batched        // Batch and upload
        case wifiOnly       // Wait for WiFi
        case queued         // Queue for later
    }

    func determineStrategy() -> UploadStrategy {
        let network = NetworkMonitor.shared

        guard network.isOnline else {
            return .queued
        }

        switch network.connectionType {
        case .wifi, .ethernet:
            return .immediate
        case .cellular:
            // Check data size and user preference
            if UserPreferences.shared.uploadOnCellular {
                return .batched
            } else {
                return .wifiOnly
            }
        default:
            return .batched
        }
    }

    func upload(data: Data, priority: EventPriority, completion: @escaping (Bool) -> Void) {
        let strategy = determineStrategy()

        switch (strategy, priority) {
        case (.queued, _):
            queueForLater(data: data, priority: priority)
            completion(false)

        case (_, .critical):
            // Always upload critical data immediately
            uploadNow(data: data, completion: completion)

        case (.immediate, _):
            uploadNow(data: data, completion: completion)

        case (.batched, _):
            addToBatch(data: data, priority: priority)
            completion(true)

        case (.wifiOnly, _):
            if NetworkMonitor.shared.connectionType == .wifi {
                uploadNow(data: data, completion: completion)
            } else {
                queueForWifi(data: data, priority: priority)
                completion(false)
            }
        }
    }

    private func uploadNow(data: Data, completion: @escaping (Bool) -> Void) {
        // Actual upload implementation
    }

    private func queueForLater(data: Data, priority: EventPriority) {
        OfflineQueue.shared.enqueue(QueuedEvent(
            type: "telemetry",
            payload: data,
            timestamp: Date(),
            priority: priority
        ))
    }

    private func addToBatch(data: Data, priority: EventPriority) {
        // Add to batch buffer
    }

    private func queueForWifi(data: Data, priority: EventPriority) {
        // Similar to queueForLater but with wifi flag
    }
}
```

---

## Summary

| Challenge | Solution |
|-----------|----------|
| **Offline** | SQLite queue, priority-based retention, background flush |
| **Fragmentation** | Device context collection, device-aware sampling |
| **Battery** | Mode-based telemetry, memory pressure handling |
| **Correlation** | W3C trace headers, session/device IDs |
| **Lifecycle** | State-aware telemetry, graceful shutdown |
| **Network** | Adaptive upload, WiFi-preferred batching |

### Resource Budgets

| Resource | Normal Mode | Low Power Mode | Critical |
|----------|-------------|----------------|----------|
| **CPU** | <5% | <2% | <1% |
| **Memory** | <50MB | <20MB | <10MB |
| **Network** | On WiFi | WiFi only | Critical only |
| **Battery** | <2%/hr | <0.5%/hr | 0% |

---

## Integration Points

- **[observability-fundamentals.md](observability-fundamentals.md)** - Correlation context
- **[performance.md](performance.md)** - Device-aware performance budgets
- **[crash-reporting.md](crash-reporting.md)** - Offline crash persistence
- **[session-replay.md](session-replay.md)** - Battery-aware capture
