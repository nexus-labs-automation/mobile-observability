# Data Persistence Tracing

Instrumentation patterns for database and persistence layer observability: query performance, batch operations, migrations, and debugging.

## Table of Contents
1. [Overview](#overview)
2. [Universal Query Tracing Pattern](#universal-query-tracing-pattern)
3. [SQLite Direct Tracing](#sqlite-direct-tracing)
4. [iOS Core Data](#ios-core-data)
5. [iOS SwiftData](#ios-swiftdata)
6. [iOS GRDB](#ios-grdb)
7. [Android Room](#android-room)
8. [Android SQLDelight](#android-sqldelight)
9. [Realm (Cross-Platform)](#realm-cross-platform)
10. [React Native Persistence](#react-native-persistence)
11. [MCP Server for Database Observability](#mcp-server-for-database-observability)

---

## Overview

### What to Trace

| Metric | Why It Matters | Alert Threshold |
|--------|----------------|-----------------|
| Query duration | User-perceived latency | >100ms |
| Rows scanned | Index effectiveness | >1000 for simple queries |
| Rows returned | Over-fetching detection | >100 for UI lists |
| Write transaction duration | Main thread blocking | >50ms |
| Connection pool wait | Contention issues | >10ms |
| Migration duration | Upgrade experience | >5s |
| Cache hit rate | Memory efficiency | <80% |

### Tracing Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Application Layer                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │  SwiftData  │  │  Core Data  │  │    Room     │              │
│  │  /GRDB      │  │             │  │  /SQLDelight│              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                │                │                      │
│         └────────────────┼────────────────┘                      │
│                          ▼                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Database Tracing Layer                        │  │
│  │  • Query interception                                      │  │
│  │  • Timing measurement                                      │  │
│  │  • Parameter capture (sanitized)                           │  │
│  │  • Explain plan analysis                                   │  │
│  └─────────────────────────┬─────────────────────────────────┘  │
│                            │                                     │
│                            ▼                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    SQLite Engine                           │  │
│  │  • sqlite3_trace_v2 (low-level)                           │  │
│  │  • Statement profiling                                     │  │
│  │  • WAL checkpointing                                       │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  Observability  │
                    │  Backend        │
                    └─────────────────┘
```

---

## Universal Query Tracing Pattern

Regardless of the ORM/framework, capture these dimensions:

```swift
// QueryTrace.swift - Universal trace structure
struct QueryTrace: Codable {
    let id: UUID
    let timestamp: Date
    let operation: QueryOperation
    let sql: String              // Normalized (no literal values)
    let parameters: [String]?    // Sanitized parameter types
    let durationMs: Double
    let rowsAffected: Int?
    let rowsReturned: Int?
    let explainPlan: String?     // EXPLAIN QUERY PLAN output
    let source: QuerySource
    let context: QueryContext

    enum QueryOperation: String, Codable {
        case select, insert, update, delete,
             transaction, migration, vacuum, checkpoint
    }

    struct QuerySource: Codable {
        let file: String
        let function: String
        let line: Int
    }

    struct QueryContext: Codable {
        let screenName: String?
        let userAction: String?
        let transactionId: String?
        let batchSize: Int?
    }
}

// Thresholds for alerting
enum QueryPerformanceThresholds {
    static let slowQueryMs = 100.0
    static let verySlowQueryMs = 500.0
    static let maxRowsForListQuery = 100
    static let maxRowsScanned = 1000
    static let slowTransactionMs = 50.0
}
```

```kotlin
// QueryTrace.kt - Android equivalent
data class QueryTrace(
    val id: String = UUID.randomUUID().toString(),
    val timestamp: Long = System.currentTimeMillis(),
    val operation: QueryOperation,
    val sql: String,
    val parameters: List<String>? = null,
    val durationMs: Double,
    val rowsAffected: Int? = null,
    val rowsReturned: Int? = null,
    val explainPlan: String? = null,
    val source: QuerySource,
    val context: QueryContext
) {
    enum class QueryOperation {
        SELECT, INSERT, UPDATE, DELETE,
        TRANSACTION, MIGRATION, VACUUM, CHECKPOINT
    }

    data class QuerySource(
        val file: String,
        val function: String,
        val line: Int
    )

    data class QueryContext(
        val screenName: String? = null,
        val userAction: String? = null,
        val transactionId: String? = null,
        val batchSize: Int? = null
    )
}
```

---

## SQLite Direct Tracing

The lowest level - works with any SQLite wrapper.

### iOS: sqlite3_trace_v2

```swift
// SQLiteTracer.swift
import SQLite3

final class SQLiteTracer {
    typealias TraceCallback = (String, Double, Int) -> Void

    private var db: OpaquePointer?
    private var traceCallback: TraceCallback?

    func enableTracing(on database: OpaquePointer, callback: @escaping TraceCallback) {
        self.db = database
        self.traceCallback = callback

        // Enable statement profiling
        sqlite3_trace_v2(
            database,
            UInt32(SQLITE_TRACE_STMT | SQLITE_TRACE_PROFILE),
            { (mask, context, p, x) -> Int32 in
                guard let context = context else { return 0 }
                let tracer = Unmanaged<SQLiteTracer>.fromOpaque(context).takeUnretainedValue()

                if mask == UInt32(SQLITE_TRACE_STMT) {
                    // Statement started
                    if let stmt = OpaquePointer(p),
                       let sqlPtr = sqlite3_expanded_sql(stmt) {
                        let sql = String(cString: sqlPtr)
                        sqlite3_free(sqlPtr)
                        // Store start time
                        tracer.recordStatementStart(sql: sql)
                    }
                } else if mask == UInt32(SQLITE_TRACE_PROFILE) {
                    // Statement completed
                    if let stmt = OpaquePointer(p) {
                        let nanoseconds = x.map { $0.load(as: Int64.self) } ?? 0
                        let durationMs = Double(nanoseconds) / 1_000_000

                        if let sqlPtr = sqlite3_expanded_sql(stmt) {
                            let sql = String(cString: sqlPtr)
                            sqlite3_free(sqlPtr)

                            let changes = sqlite3_changes(tracer.db)
                            tracer.traceCallback?(sql, durationMs, Int(changes))
                        }
                    }
                }

                return 0
            },
            Unmanaged.passUnretained(self).toOpaque()
        )
    }

    private var pendingStatements: [String: CFAbsoluteTime] = [:]

    private func recordStatementStart(sql: String) {
        pendingStatements[sql] = CFAbsoluteTimeGetCurrent()
    }

    func disableTracing() {
        guard let db = db else { return }
        sqlite3_trace_v2(db, 0, nil, nil)
    }
}

// Usage
let tracer = SQLiteTracer()
tracer.enableTracing(on: dbPointer) { sql, durationMs, rowsAffected in
    if durationMs > 100 {
        Observability.recordMetric(
            name: "sqlite.slow_query",
            value: durationMs,
            unit: .milliseconds,
            tags: [
                "operation": sql.hasPrefix("SELECT") ? "select" : "write",
                "table": extractTableName(from: sql)
            ]
        )
    }
}
```

### EXPLAIN QUERY PLAN Analysis

```swift
// QueryPlanAnalyzer.swift
struct QueryPlanAnalyzer {
    let db: OpaquePointer

    struct QueryPlan {
        let steps: [PlanStep]
        let usesIndex: Bool
        let scanType: ScanType
        let estimatedRows: Int?

        enum ScanType: String {
            case indexScan = "SEARCH"
            case tableScan = "SCAN"
            case coveringIndex = "COVERING INDEX"
        }
    }

    struct PlanStep {
        let id: Int
        let parent: Int
        let detail: String
    }

    func analyze(sql: String) -> QueryPlan? {
        var stmt: OpaquePointer?
        let explainSQL = "EXPLAIN QUERY PLAN \(sql)"

        guard sqlite3_prepare_v2(db, explainSQL, -1, &stmt, nil) == SQLITE_OK else {
            return nil
        }
        defer { sqlite3_finalize(stmt) }

        var steps: [PlanStep] = []

        while sqlite3_step(stmt) == SQLITE_ROW {
            let id = Int(sqlite3_column_int(stmt, 0))
            let parent = Int(sqlite3_column_int(stmt, 1))
            let detail = String(cString: sqlite3_column_text(stmt, 3))
            steps.append(PlanStep(id: id, parent: parent, detail: detail))
        }

        let usesIndex = steps.contains { $0.detail.contains("USING INDEX") }
        let scanType: QueryPlan.ScanType = {
            if steps.contains(where: { $0.detail.contains("COVERING INDEX") }) {
                return .coveringIndex
            } else if usesIndex {
                return .indexScan
            } else {
                return .tableScan
            }
        }()

        return QueryPlan(
            steps: steps,
            usesIndex: usesIndex,
            scanType: scanType,
            estimatedRows: extractEstimatedRows(from: steps)
        )
    }

    private func extractEstimatedRows(from steps: [PlanStep]) -> Int? {
        // Parse (~N rows) from plan output
        for step in steps {
            if let range = step.detail.range(of: #"\(~(\d+) rows\)"#, options: .regularExpression) {
                let match = step.detail[range]
                let number = match.filter { $0.isNumber }
                return Int(number)
            }
        }
        return nil
    }

    func warnIfFullTableScan(sql: String) {
        guard let plan = analyze(sql: sql) else { return }

        if plan.scanType == .tableScan {
            Observability.captureMessage(
                "Full table scan detected",
                level: .warning,
                extras: [
                    "sql": sql,
                    "plan": plan.steps.map(\.detail).joined(separator: "\n")
                ]
            )
        }
    }
}
```

### Android: SQLite Tracing

```kotlin
// SQLiteTracer.kt
import android.database.sqlite.SQLiteDatabase
import android.os.Build

object SQLiteTracer {

    // API 28+ tracing
    @RequiresApi(Build.VERSION_CODES.P)
    fun enableTracing(db: SQLiteDatabase, callback: (String, Long) -> Unit) {
        // SQLite tracing via reflection (not officially supported)
        // Use Room's query callback instead for production
    }

    // Manual timing wrapper
    inline fun <T> trace(
        db: SQLiteDatabase,
        sql: String,
        crossinline block: () -> T
    ): T {
        val startTime = System.nanoTime()
        return try {
            block()
        } finally {
            val durationMs = (System.nanoTime() - startTime) / 1_000_000.0
            recordQuery(sql, durationMs)
        }
    }

    fun recordQuery(sql: String, durationMs: Double) {
        if (durationMs > 100) {
            Observability.recordMetric(
                name = "sqlite.slow_query",
                value = durationMs,
                unit = MetricUnit.MILLISECONDS,
                tags = mapOf(
                    "operation" to extractOperation(sql),
                    "table" to extractTableName(sql)
                )
            )
        }
    }

    private fun extractOperation(sql: String): String {
        return sql.trim().split(" ").firstOrNull()?.uppercase() ?: "UNKNOWN"
    }

    private fun extractTableName(sql: String): String {
        val patterns = listOf(
            Regex("FROM\\s+(\\w+)", RegexOption.IGNORE_CASE),
            Regex("INTO\\s+(\\w+)", RegexOption.IGNORE_CASE),
            Regex("UPDATE\\s+(\\w+)", RegexOption.IGNORE_CASE),
            Regex("DELETE\\s+FROM\\s+(\\w+)", RegexOption.IGNORE_CASE)
        )

        for (pattern in patterns) {
            pattern.find(sql)?.groupValues?.getOrNull(1)?.let { return it }
        }
        return "unknown"
    }
}
```

---

## iOS Core Data

### Fetch Request Tracing

```swift
// CoreDataTracer.swift
import CoreData
import os.signpost

final class CoreDataTracer {
    static let shared = CoreDataTracer()

    private let signpostLog = OSLog(subsystem: Bundle.main.bundleIdentifier!, category: "CoreData")

    // MARK: - Fetch Request Tracing

    func trace<T: NSManagedObject>(
        _ request: NSFetchRequest<T>,
        in context: NSManagedObjectContext,
        screen: String? = nil,
        action: String? = nil
    ) throws -> [T] {
        let signpostID = OSSignpostID(log: signpostLog)
        let entityName = request.entityName ?? "Unknown"
        let predicateDesc = request.predicate?.description ?? "none"

        os_signpost(.begin, log: signpostLog, name: "CoreData.fetch", signpostID: signpostID,
                    "entity: %{public}s, predicate: %{public}s", entityName, predicateDesc)

        let startTime = CFAbsoluteTimeGetCurrent()

        do {
            let results = try context.fetch(request)
            let durationMs = (CFAbsoluteTimeGetCurrent() - startTime) * 1000

            os_signpost(.end, log: signpostLog, name: "CoreData.fetch", signpostID: signpostID,
                        "rows: %d, duration_ms: %.2f", results.count, durationMs)

            recordFetch(
                entity: entityName,
                predicate: predicateDesc,
                sortDescriptors: request.sortDescriptors?.map(\.description) ?? [],
                fetchLimit: request.fetchLimit,
                rowsReturned: results.count,
                durationMs: durationMs,
                screen: screen,
                action: action
            )

            // Warn on potential issues
            if results.count > 100 && request.fetchLimit == 0 {
                Observability.captureMessage(
                    "Large unbounded fetch",
                    level: .warning,
                    extras: [
                        "entity": entityName,
                        "rows_returned": results.count,
                        "predicate": predicateDesc
                    ]
                )
            }

            return results
        } catch {
            os_signpost(.end, log: signpostLog, name: "CoreData.fetch", signpostID: signpostID,
                        "error: %{public}s", error.localizedDescription)
            throw error
        }
    }

    private func recordFetch(
        entity: String,
        predicate: String,
        sortDescriptors: [String],
        fetchLimit: Int,
        rowsReturned: Int,
        durationMs: Double,
        screen: String?,
        action: String?
    ) {
        Observability.recordMetric(
            name: "coredata.fetch.duration",
            value: durationMs,
            unit: .milliseconds,
            tags: [
                "entity": entity,
                "has_predicate": (predicate != "none").description,
                "has_limit": (fetchLimit > 0).description,
                "screen": screen ?? "unknown"
            ]
        )

        Observability.recordMetric(
            name: "coredata.fetch.rows",
            value: Double(rowsReturned),
            tags: ["entity": entity]
        )

        if durationMs > QueryPerformanceThresholds.slowQueryMs {
            Observability.captureMessage(
                "Slow Core Data fetch",
                level: .warning,
                extras: [
                    "entity": entity,
                    "predicate": predicate,
                    "duration_ms": durationMs,
                    "rows": rowsReturned,
                    "screen": screen ?? "unknown"
                ]
            )
        }
    }

    // MARK: - Save Context Tracing

    func traceSave(
        context: NSManagedObjectContext,
        operation: String = "save"
    ) throws {
        guard context.hasChanges else { return }

        let signpostID = OSSignpostID(log: signpostLog)
        let insertCount = context.insertedObjects.count
        let updateCount = context.updatedObjects.count
        let deleteCount = context.deletedObjects.count

        os_signpost(.begin, log: signpostLog, name: "CoreData.save", signpostID: signpostID,
                    "inserts: %d, updates: %d, deletes: %d", insertCount, updateCount, deleteCount)

        let startTime = CFAbsoluteTimeGetCurrent()

        do {
            try context.save()
            let durationMs = (CFAbsoluteTimeGetCurrent() - startTime) * 1000

            os_signpost(.end, log: signpostLog, name: "CoreData.save", signpostID: signpostID,
                        "duration_ms: %.2f", durationMs)

            recordSave(
                insertCount: insertCount,
                updateCount: updateCount,
                deleteCount: deleteCount,
                durationMs: durationMs,
                operation: operation
            )
        } catch {
            os_signpost(.end, log: signpostLog, name: "CoreData.save", signpostID: signpostID,
                        "error: %{public}s", error.localizedDescription)
            throw error
        }
    }

    private func recordSave(
        insertCount: Int,
        updateCount: Int,
        deleteCount: Int,
        durationMs: Double,
        operation: String
    ) {
        let totalChanges = insertCount + updateCount + deleteCount

        Observability.recordMetric(
            name: "coredata.save.duration",
            value: durationMs,
            unit: .milliseconds,
            tags: ["operation": operation]
        )

        Observability.recordMetric(
            name: "coredata.save.changes",
            value: Double(totalChanges),
            tags: [
                "inserts": String(insertCount),
                "updates": String(updateCount),
                "deletes": String(deleteCount)
            ]
        )

        if durationMs > QueryPerformanceThresholds.slowTransactionMs {
            Observability.captureMessage(
                "Slow Core Data save",
                level: .warning,
                extras: [
                    "duration_ms": durationMs,
                    "inserts": insertCount,
                    "updates": updateCount,
                    "deletes": deleteCount,
                    "total_changes": totalChanges
                ]
            )
        }
    }
}

// MARK: - NSManagedObjectContext Extension

extension NSManagedObjectContext {
    func tracedFetch<T: NSManagedObject>(
        _ request: NSFetchRequest<T>,
        screen: String? = nil,
        action: String? = nil
    ) throws -> [T] {
        try CoreDataTracer.shared.trace(request, in: self, screen: screen, action: action)
    }

    func tracedSave(operation: String = "save") throws {
        try CoreDataTracer.shared.traceSave(context: self, operation: operation)
    }
}
```

### Core Data SQL Logging (Debug)

```swift
// Enable in scheme arguments:
// -com.apple.CoreData.SQLDebug 1
// -com.apple.CoreData.Logging.stderr 1

// Or programmatically capture:
class CoreDataSQLObserver {
    init() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(handleStoreNotification),
            name: .NSPersistentStoreCoordinatorStoresDidChange,
            object: nil
        )
    }

    @objc func handleStoreNotification(_ notification: Notification) {
        // Log store changes
    }
}
```

---

## iOS SwiftData

### ModelContext Tracing

```swift
// SwiftDataTracer.swift
import SwiftData
import os.signpost

@available(iOS 17.0, macOS 14.0, *)
final class SwiftDataTracer {
    static let shared = SwiftDataTracer()

    private let signpostLog = OSLog(subsystem: Bundle.main.bundleIdentifier!, category: "SwiftData")

    // MARK: - Fetch Tracing

    func trace<T: PersistentModel>(
        _ descriptor: FetchDescriptor<T>,
        in context: ModelContext,
        screen: String? = nil
    ) throws -> [T] {
        let signpostID = OSSignpostID(log: signpostLog)
        let modelName = String(describing: T.self)

        os_signpost(.begin, log: signpostLog, name: "SwiftData.fetch", signpostID: signpostID,
                    "model: %{public}s", modelName)

        let startTime = CFAbsoluteTimeGetCurrent()

        do {
            let results = try context.fetch(descriptor)
            let durationMs = (CFAbsoluteTimeGetCurrent() - startTime) * 1000

            os_signpost(.end, log: signpostLog, name: "SwiftData.fetch", signpostID: signpostID,
                        "rows: %d, duration_ms: %.2f", results.count, durationMs)

            recordFetch(
                model: modelName,
                hasFilter: descriptor.predicate != nil,
                fetchLimit: descriptor.fetchLimit,
                rowsReturned: results.count,
                durationMs: durationMs,
                screen: screen
            )

            return results
        } catch {
            os_signpost(.end, log: signpostLog, name: "SwiftData.fetch", signpostID: signpostID,
                        "error: %{public}s", error.localizedDescription)
            throw error
        }
    }

    private func recordFetch(
        model: String,
        hasFilter: Bool,
        fetchLimit: Int?,
        rowsReturned: Int,
        durationMs: Double,
        screen: String?
    ) {
        Observability.recordMetric(
            name: "swiftdata.fetch.duration",
            value: durationMs,
            unit: .milliseconds,
            tags: [
                "model": model,
                "has_filter": hasFilter.description,
                "screen": screen ?? "unknown"
            ]
        )

        if durationMs > QueryPerformanceThresholds.slowQueryMs {
            Observability.captureMessage(
                "Slow SwiftData fetch",
                level: .warning,
                extras: [
                    "model": model,
                    "duration_ms": durationMs,
                    "rows": rowsReturned
                ]
            )
        }
    }

    // MARK: - Save Tracing

    func traceSave(context: ModelContext, operation: String = "save") throws {
        guard context.hasChanges else { return }

        let signpostID = OSSignpostID(log: signpostLog)

        os_signpost(.begin, log: signpostLog, name: "SwiftData.save", signpostID: signpostID)

        let startTime = CFAbsoluteTimeGetCurrent()

        do {
            try context.save()
            let durationMs = (CFAbsoluteTimeGetCurrent() - startTime) * 1000

            os_signpost(.end, log: signpostLog, name: "SwiftData.save", signpostID: signpostID,
                        "duration_ms: %.2f", durationMs)

            Observability.recordMetric(
                name: "swiftdata.save.duration",
                value: durationMs,
                unit: .milliseconds,
                tags: ["operation": operation]
            )
        } catch {
            os_signpost(.end, log: signpostLog, name: "SwiftData.save", signpostID: signpostID,
                        "error: %{public}s", error.localizedDescription)
            throw error
        }
    }
}

// MARK: - ModelContext Extension

@available(iOS 17.0, macOS 14.0, *)
extension ModelContext {
    func tracedFetch<T: PersistentModel>(
        _ descriptor: FetchDescriptor<T>,
        screen: String? = nil
    ) throws -> [T] {
        try SwiftDataTracer.shared.trace(descriptor, in: self, screen: screen)
    }

    func tracedSave(operation: String = "save") throws {
        try SwiftDataTracer.shared.traceSave(context: self, operation: operation)
    }
}
```

---

## iOS GRDB

GRDB has excellent built-in tracing support.

### Statement Statistics

```swift
// GRDBTracer.swift
import GRDB
import os.signpost

final class GRDBTracer {
    static let shared = GRDBTracer()

    private let signpostLog = OSLog(subsystem: Bundle.main.bundleIdentifier!, category: "GRDB")
    private var queryStats: [String: QueryStats] = [:]
    private let queue = DispatchQueue(label: "grdb.tracer")

    struct QueryStats {
        var count: Int = 0
        var totalDurationMs: Double = 0
        var maxDurationMs: Double = 0
        var minDurationMs: Double = .infinity

        var avgDurationMs: Double {
            count > 0 ? totalDurationMs / Double(count) : 0
        }
    }

    // MARK: - Database Configuration

    func configure(_ configuration: inout Configuration) {
        // Enable tracing
        configuration.prepareDatabase { db in
            db.trace(options: .statement) { event in
                self.handleTraceEvent(event)
            }
        }

        // Add SQL logging in debug
        #if DEBUG
        configuration.prepareDatabase { db in
            db.trace { print("SQL: \($0)") }
        }
        #endif
    }

    private func handleTraceEvent(_ event: Database.TraceEvent) {
        switch event {
        case .statement(let statement):
            handleStatement(statement)
        case .profile(let statement, let duration):
            handleProfile(statement: statement, duration: duration)
        }
    }

    private func handleStatement(_ statement: Statement) {
        let signpostID = OSSignpostID(log: signpostLog)
        os_signpost(.begin, log: signpostLog, name: "GRDB.statement", signpostID: signpostID,
                    "sql: %{public}s", statement.sql)
    }

    private func handleProfile(statement: Statement, duration: TimeInterval) {
        let durationMs = duration * 1000
        let sql = statement.sql
        let normalizedSQL = normalizeSQL(sql)

        // Update stats
        queue.async {
            var stats = self.queryStats[normalizedSQL] ?? QueryStats()
            stats.count += 1
            stats.totalDurationMs += durationMs
            stats.maxDurationMs = max(stats.maxDurationMs, durationMs)
            stats.minDurationMs = min(stats.minDurationMs, durationMs)
            self.queryStats[normalizedSQL] = stats
        }

        // Record metric
        Observability.recordMetric(
            name: "grdb.query.duration",
            value: durationMs,
            unit: .milliseconds,
            tags: [
                "operation": extractOperation(from: sql),
                "table": extractTableName(from: sql)
            ]
        )

        // Slow query alert
        if durationMs > QueryPerformanceThresholds.slowQueryMs {
            Observability.captureMessage(
                "Slow GRDB query",
                level: .warning,
                extras: [
                    "sql": sql,
                    "duration_ms": durationMs,
                    "arguments": statement.arguments.description
                ]
            )
        }
    }

    // MARK: - Query Plan Analysis

    func analyzeQueryPlan(db: Database, sql: String) throws -> [String] {
        let rows = try Row.fetchAll(db, sql: "EXPLAIN QUERY PLAN \(sql)")
        return rows.map { row -> String in
            let id: Int = row["id"]
            let parent: Int = row["parent"]
            let detail: String = row["detail"]
            return "[\(id):\(parent)] \(detail)"
        }
    }

    // MARK: - Transaction Tracing

    func traceTransaction<T>(
        _ db: Database,
        operation: String,
        block: () throws -> T
    ) rethrows -> T {
        let signpostID = OSSignpostID(log: signpostLog)

        os_signpost(.begin, log: signpostLog, name: "GRDB.transaction", signpostID: signpostID,
                    "operation: %{public}s", operation)

        let startTime = CFAbsoluteTimeGetCurrent()

        defer {
            let durationMs = (CFAbsoluteTimeGetCurrent() - startTime) * 1000

            os_signpost(.end, log: signpostLog, name: "GRDB.transaction", signpostID: signpostID,
                        "duration_ms: %.2f", durationMs)

            Observability.recordMetric(
                name: "grdb.transaction.duration",
                value: durationMs,
                unit: .milliseconds,
                tags: ["operation": operation]
            )
        }

        return try block()
    }

    // MARK: - Stats Export

    func exportStats() -> [String: QueryStats] {
        queue.sync { queryStats }
    }

    func resetStats() {
        queue.async { self.queryStats.removeAll() }
    }

    // MARK: - Helpers

    private func normalizeSQL(_ sql: String) -> String {
        // Replace literal values with placeholders for grouping
        sql.replacingOccurrences(
            of: #"'[^']*'"#,
            with: "?",
            options: .regularExpression
        ).replacingOccurrences(
            of: #"\b\d+\b"#,
            with: "?",
            options: .regularExpression
        )
    }

    private func extractOperation(from sql: String) -> String {
        sql.trimmingCharacters(in: .whitespaces)
            .components(separatedBy: " ")
            .first?
            .uppercased() ?? "UNKNOWN"
    }

    private func extractTableName(from sql: String) -> String {
        let patterns = [
            #"FROM\s+(\w+)"#,
            #"INTO\s+(\w+)"#,
            #"UPDATE\s+(\w+)"#,
            #"DELETE\s+FROM\s+(\w+)"#
        ]

        for pattern in patterns {
            if let regex = try? NSRegularExpression(pattern: pattern, options: .caseInsensitive),
               let match = regex.firstMatch(in: sql, range: NSRange(sql.startIndex..., in: sql)),
               let range = Range(match.range(at: 1), in: sql) {
                return String(sql[range])
            }
        }
        return "unknown"
    }
}

// MARK: - Database Extension

extension Database {
    func tracedWrite<T>(_ operation: String, _ block: () throws -> T) rethrows -> T {
        try GRDBTracer.shared.traceTransaction(self, operation: operation, block: block)
    }
}
```

### GRDB Database Pool with Tracing

```swift
// TracedDatabasePool.swift
import GRDB

class TracedDatabasePool {
    let pool: DatabasePool

    init(path: String) throws {
        var config = Configuration()
        GRDBTracer.shared.configure(&config)

        // Connection pool stats
        config.maximumReaderCount = 5

        pool = try DatabasePool(path: path, configuration: config)
    }

    // Trace read operations
    func read<T>(_ block: (Database) throws -> T) throws -> T {
        let startTime = CFAbsoluteTimeGetCurrent()
        let result = try pool.read(block)
        let durationMs = (CFAbsoluteTimeGetCurrent() - startTime) * 1000

        Observability.recordMetric(
            name: "grdb.pool.read",
            value: durationMs,
            unit: .milliseconds
        )

        return result
    }

    // Trace write operations
    func write<T>(_ block: (Database) throws -> T) throws -> T {
        let startTime = CFAbsoluteTimeGetCurrent()
        let result = try pool.write(block)
        let durationMs = (CFAbsoluteTimeGetCurrent() - startTime) * 1000

        Observability.recordMetric(
            name: "grdb.pool.write",
            value: durationMs,
            unit: .milliseconds
        )

        return result
    }
}
```

---

## Android Room

### RoomDatabase Query Callback

```kotlin
// RoomTracer.kt
import androidx.room.RoomDatabase
import android.os.Trace
import java.util.concurrent.ConcurrentHashMap

object RoomTracer {
    private val queryStats = ConcurrentHashMap<String, QueryStats>()

    data class QueryStats(
        var count: Int = 0,
        var totalDurationMs: Double = 0.0,
        var maxDurationMs: Double = 0.0,
        var minDurationMs: Double = Double.MAX_VALUE
    ) {
        val avgDurationMs: Double
            get() = if (count > 0) totalDurationMs / count else 0.0
    }

    // Room query callback (added to database builder)
    val queryCallback = object : RoomDatabase.QueryCallback {
        override fun onQuery(sqlQuery: String, bindArgs: List<Any?>) {
            val startTime = System.nanoTime()

            // Log query start for systrace
            Trace.beginSection("Room: ${sqlQuery.take(50)}")

            // Store start time in thread local for later
            queryStartTimes.set(startTime)
        }
    }

    private val queryStartTimes = ThreadLocal<Long>()

    // Call this after query completes
    fun recordQueryComplete(sql: String, rowCount: Int) {
        val startTime = queryStartTimes.get() ?: return
        val durationMs = (System.nanoTime() - startTime) / 1_000_000.0

        Trace.endSection()
        queryStartTimes.remove()

        // Update stats
        val normalizedSQL = normalizeSQL(sql)
        queryStats.compute(normalizedSQL) { _, stats ->
            (stats ?: QueryStats()).apply {
                count++
                totalDurationMs += durationMs
                maxDurationMs = maxOf(maxDurationMs, durationMs)
                minDurationMs = minOf(minDurationMs, durationMs)
            }
        }

        // Record metric
        Observability.recordMetric(
            name = "room.query.duration",
            value = durationMs,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf(
                "operation" to extractOperation(sql),
                "table" to extractTableName(sql)
            )
        )

        // Slow query alert
        if (durationMs > 100) {
            Observability.captureMessage(
                message = "Slow Room query",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "sql" to sql,
                    "duration_ms" to durationMs,
                    "rows" to rowCount
                )
            )
        }
    }

    private fun normalizeSQL(sql: String): String {
        return sql
            .replace(Regex("'[^']*'"), "?")
            .replace(Regex("\\b\\d+\\b"), "?")
    }

    private fun extractOperation(sql: String): String {
        return sql.trim().split(" ").firstOrNull()?.uppercase() ?: "UNKNOWN"
    }

    private fun extractTableName(sql: String): String {
        val patterns = listOf(
            Regex("FROM\\s+(\\w+)", RegexOption.IGNORE_CASE),
            Regex("INTO\\s+(\\w+)", RegexOption.IGNORE_CASE),
            Regex("UPDATE\\s+(\\w+)", RegexOption.IGNORE_CASE)
        )

        for (pattern in patterns) {
            pattern.find(sql)?.groupValues?.getOrNull(1)?.let { return it }
        }
        return "unknown"
    }

    fun exportStats(): Map<String, QueryStats> = queryStats.toMap()

    fun resetStats() = queryStats.clear()
}

// Database builder configuration
@Database(entities = [Product::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun productDao(): ProductDao

    companion object {
        fun create(context: Context): AppDatabase {
            return Room.databaseBuilder(
                context,
                AppDatabase::class.java,
                "app.db"
            )
            .setQueryCallback(RoomTracer.queryCallback, Executors.newSingleThreadExecutor())
            .build()
        }
    }
}
```

### Room DAO Tracing Wrapper

```kotlin
// TracedDao.kt
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.onEach
import kotlin.system.measureTimeMillis

// Wraps DAO methods with timing
class TracedProductDao(private val delegate: ProductDao) : ProductDao {

    override suspend fun getAll(): List<Product> {
        val startTime = System.nanoTime()
        val result = delegate.getAll()
        val durationMs = (System.nanoTime() - startTime) / 1_000_000.0

        recordDaoCall("ProductDao.getAll", durationMs, result.size)
        return result
    }

    override suspend fun getById(id: String): Product? {
        val startTime = System.nanoTime()
        val result = delegate.getById(id)
        val durationMs = (System.nanoTime() - startTime) / 1_000_000.0

        recordDaoCall("ProductDao.getById", durationMs, if (result != null) 1 else 0)
        return result
    }

    override fun observeAll(): Flow<List<Product>> {
        return delegate.observeAll().onEach { products ->
            // Track each emission
            Observability.recordMetric(
                name = "room.flow.emit",
                value = products.size.toDouble(),
                tags = mapOf("dao" to "ProductDao", "method" to "observeAll")
            )
        }
    }

    override suspend fun insert(product: Product) {
        val durationMs = measureTimeMillis {
            delegate.insert(product)
        }.toDouble()

        recordDaoCall("ProductDao.insert", durationMs, 1)
    }

    override suspend fun insertAll(products: List<Product>) {
        val durationMs = measureTimeMillis {
            delegate.insertAll(products)
        }.toDouble()

        recordDaoCall("ProductDao.insertAll", durationMs, products.size)
    }

    private fun recordDaoCall(method: String, durationMs: Double, rowCount: Int) {
        Trace.beginSection(method)

        Observability.recordMetric(
            name = "room.dao.duration",
            value = durationMs,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf("method" to method)
        )

        if (durationMs > 100) {
            Observability.captureMessage(
                message = "Slow DAO method",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "method" to method,
                    "duration_ms" to durationMs,
                    "rows" to rowCount
                )
            )
        }

        Trace.endSection()
    }
}
```

### Room Migration Tracing

```kotlin
// TracedMigration.kt
import androidx.room.migration.Migration
import androidx.sqlite.db.SupportSQLiteDatabase

class TracedMigration(
    startVersion: Int,
    endVersion: Int,
    private val migrationBlock: (SupportSQLiteDatabase) -> Unit
) : Migration(startVersion, endVersion) {

    override fun migrate(database: SupportSQLiteDatabase) {
        Trace.beginSection("Room.migration.$startVersion->$endVersion")

        val startTime = System.nanoTime()

        try {
            migrationBlock(database)

            val durationMs = (System.nanoTime() - startTime) / 1_000_000.0

            Observability.recordMetric(
                name = "room.migration.duration",
                value = durationMs,
                unit = MetricUnit.MILLISECONDS,
                tags = mapOf(
                    "from_version" to startVersion.toString(),
                    "to_version" to endVersion.toString()
                )
            )

            if (durationMs > 5000) {
                Observability.captureMessage(
                    message = "Slow database migration",
                    level = SentryLevel.WARNING,
                    extras = mapOf(
                        "from_version" to startVersion,
                        "to_version" to endVersion,
                        "duration_ms" to durationMs
                    )
                )
            }
        } catch (e: Exception) {
            Observability.captureError(
                e,
                extras = mapOf(
                    "context" to "room_migration",
                    "from_version" to startVersion,
                    "to_version" to endVersion
                )
            )
            throw e
        } finally {
            Trace.endSection()
        }
    }
}

// Usage
val MIGRATION_1_2 = TracedMigration(1, 2) { database ->
    database.execSQL("ALTER TABLE products ADD COLUMN category TEXT")
}
```

---

## Android SQLDelight

```kotlin
// SQLDelightTracer.kt
import app.cash.sqldelight.db.SqlDriver
import app.cash.sqldelight.driver.android.AndroidSqliteDriver
import android.content.Context

object SQLDelightTracer {

    fun createTracedDriver(context: Context, schema: SqlDriver.Schema, name: String): SqlDriver {
        return TracedSqlDriver(
            AndroidSqliteDriver(schema, context, name)
        )
    }
}

class TracedSqlDriver(private val delegate: SqlDriver) : SqlDriver by delegate {

    override fun execute(
        identifier: Int?,
        sql: String,
        parameters: Int,
        binders: (SqlPreparedStatement.() -> Unit)?
    ): QueryResult<Long> {
        val startTime = System.nanoTime()

        return try {
            delegate.execute(identifier, sql, parameters, binders).also { result ->
                val durationMs = (System.nanoTime() - startTime) / 1_000_000.0
                recordQuery(sql, durationMs, result.value.toInt())
            }
        } catch (e: Exception) {
            recordQueryError(sql, e)
            throw e
        }
    }

    override fun <R> executeQuery(
        identifier: Int?,
        sql: String,
        mapper: (SqlCursor) -> QueryResult<R>,
        parameters: Int,
        binders: (SqlPreparedStatement.() -> Unit)?
    ): QueryResult<R> {
        val startTime = System.nanoTime()

        return try {
            delegate.executeQuery(identifier, sql, mapper, parameters, binders).also {
                val durationMs = (System.nanoTime() - startTime) / 1_000_000.0
                recordQuery(sql, durationMs, null)
            }
        } catch (e: Exception) {
            recordQueryError(sql, e)
            throw e
        }
    }

    private fun recordQuery(sql: String, durationMs: Double, rowsAffected: Int?) {
        Observability.recordMetric(
            name = "sqldelight.query.duration",
            value = durationMs,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf(
                "operation" to sql.trim().split(" ").first().uppercase()
            )
        )

        if (durationMs > 100) {
            Observability.captureMessage(
                message = "Slow SQLDelight query",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "sql" to sql,
                    "duration_ms" to durationMs,
                    "rows_affected" to (rowsAffected ?: "unknown")
                )
            )
        }
    }

    private fun recordQueryError(sql: String, error: Exception) {
        Observability.captureError(
            error,
            extras = mapOf(
                "context" to "sqldelight_query",
                "sql" to sql
            )
        )
    }
}
```

---

## Realm (Cross-Platform)

### iOS Realm Tracing

```swift
// RealmTracer.swift
import RealmSwift

final class RealmTracer {
    static let shared = RealmTracer()

    // MARK: - Query Tracing

    func trace<T: Object>(
        _ type: T.Type,
        in realm: Realm,
        filter: String? = nil,
        sortKeyPath: String? = nil,
        screen: String? = nil
    ) -> Results<T> {
        let startTime = CFAbsoluteTimeGetCurrent()

        var results = realm.objects(type)

        if let filter = filter {
            results = results.filter(filter)
        }

        if let sortKeyPath = sortKeyPath {
            results = results.sorted(byKeyPath: sortKeyPath)
        }

        let durationMs = (CFAbsoluteTimeGetCurrent() - startTime) * 1000

        recordQuery(
            type: String(describing: type),
            filter: filter,
            sort: sortKeyPath,
            count: results.count,
            durationMs: durationMs,
            screen: screen
        )

        return results
    }

    private func recordQuery(
        type: String,
        filter: String?,
        sort: String?,
        count: Int,
        durationMs: Double,
        screen: String?
    ) {
        Observability.recordMetric(
            name: "realm.query.duration",
            value: durationMs,
            unit: .milliseconds,
            tags: [
                "type": type,
                "has_filter": (filter != nil).description,
                "screen": screen ?? "unknown"
            ]
        )

        if durationMs > QueryPerformanceThresholds.slowQueryMs {
            Observability.captureMessage(
                "Slow Realm query",
                level: .warning,
                extras: [
                    "type": type,
                    "filter": filter ?? "none",
                    "sort": sort ?? "none",
                    "count": count,
                    "duration_ms": durationMs
                ]
            )
        }
    }

    // MARK: - Write Transaction Tracing

    func traceWrite(
        realm: Realm,
        operation: String,
        block: () throws -> Void
    ) throws {
        let startTime = CFAbsoluteTimeGetCurrent()

        try realm.write {
            try block()
        }

        let durationMs = (CFAbsoluteTimeGetCurrent() - startTime) * 1000

        Observability.recordMetric(
            name: "realm.write.duration",
            value: durationMs,
            unit: .milliseconds,
            tags: ["operation": operation]
        )

        if durationMs > QueryPerformanceThresholds.slowTransactionMs {
            Observability.captureMessage(
                "Slow Realm write",
                level: .warning,
                extras: [
                    "operation": operation,
                    "duration_ms": durationMs
                ]
            )
        }
    }

    // MARK: - Sync Tracing (Realm Sync)

    func traceSyncSession(_ session: SyncSession) {
        // Connection state changes
        session.addProgressNotification(
            for: .download,
            mode: .reportIndefinitely
        ) { progress in
            Observability.recordMetric(
                name: "realm.sync.download",
                value: Double(progress.transferredBytes),
                unit: .bytes,
                tags: ["complete": progress.isTransferComplete.description]
            )
        }

        session.addProgressNotification(
            for: .upload,
            mode: .reportIndefinitely
        ) { progress in
            Observability.recordMetric(
                name: "realm.sync.upload",
                value: Double(progress.transferredBytes),
                unit: .bytes,
                tags: ["complete": progress.isTransferComplete.description]
            )
        }
    }
}
```

### Android Realm Tracing

```kotlin
// RealmTracer.kt
import io.realm.kotlin.Realm
import io.realm.kotlin.query.RealmResults
import io.realm.kotlin.types.RealmObject
import kotlin.reflect.KClass

object RealmTracer {

    inline fun <reified T : RealmObject> traceQuery(
        realm: Realm,
        query: String? = null,
        screen: String? = null,
        crossinline block: () -> RealmResults<T>
    ): RealmResults<T> {
        val startTime = System.nanoTime()

        val results = block()

        val durationMs = (System.nanoTime() - startTime) / 1_000_000.0

        recordQuery(
            type = T::class.simpleName ?: "Unknown",
            query = query,
            count = results.size,
            durationMs = durationMs,
            screen = screen
        )

        return results
    }

    private fun recordQuery(
        type: String,
        query: String?,
        count: Int,
        durationMs: Double,
        screen: String?
    ) {
        Observability.recordMetric(
            name = "realm.query.duration",
            value = durationMs,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf(
                "type" to type,
                "has_query" to (query != null).toString(),
                "screen" to (screen ?: "unknown")
            )
        )

        if (durationMs > 100) {
            Observability.captureMessage(
                message = "Slow Realm query",
                level = SentryLevel.WARNING,
                extras = mapOf(
                    "type" to type,
                    "query" to (query ?: "none"),
                    "count" to count,
                    "duration_ms" to durationMs
                )
            )
        }
    }

    suspend fun <T> traceWrite(
        realm: Realm,
        operation: String,
        block: suspend () -> T
    ): T {
        val startTime = System.nanoTime()

        val result = realm.write { block() }

        val durationMs = (System.nanoTime() - startTime) / 1_000_000.0

        Observability.recordMetric(
            name = "realm.write.duration",
            value = durationMs,
            unit = MetricUnit.MILLISECONDS,
            tags = mapOf("operation" to operation)
        )

        return result
    }
}
```

---

## React Native Persistence

### AsyncStorage Tracing

```typescript
// asyncStorageTracer.ts
import AsyncStorage from '@react-native-async-storage/async-storage';

type TraceCallback = (operation: string, key: string, durationMs: number, size?: number) => void;

let traceCallback: TraceCallback | null = null;

export function setTraceCallback(callback: TraceCallback) {
  traceCallback = callback;
}

export const TracedAsyncStorage = {
  async getItem(key: string): Promise<string | null> {
    const startTime = performance.now();
    const result = await AsyncStorage.getItem(key);
    const durationMs = performance.now() - startTime;

    traceCallback?.('getItem', key, durationMs, result?.length);

    if (durationMs > 100) {
      console.warn(`Slow AsyncStorage.getItem: ${key} took ${durationMs}ms`);
    }

    return result;
  },

  async setItem(key: string, value: string): Promise<void> {
    const startTime = performance.now();
    await AsyncStorage.setItem(key, value);
    const durationMs = performance.now() - startTime;

    traceCallback?.('setItem', key, durationMs, value.length);

    if (durationMs > 100) {
      console.warn(`Slow AsyncStorage.setItem: ${key} took ${durationMs}ms`);
    }
  },

  async multiGet(keys: string[]): Promise<readonly [string, string | null][]> {
    const startTime = performance.now();
    const results = await AsyncStorage.multiGet(keys);
    const durationMs = performance.now() - startTime;

    const totalSize = results.reduce((acc, [_, value]) => acc + (value?.length ?? 0), 0);
    traceCallback?.('multiGet', `[${keys.length} keys]`, durationMs, totalSize);

    return results;
  },

  async multiSet(keyValuePairs: [string, string][]): Promise<void> {
    const startTime = performance.now();
    await AsyncStorage.multiSet(keyValuePairs);
    const durationMs = performance.now() - startTime;

    const totalSize = keyValuePairs.reduce((acc, [_, value]) => acc + value.length, 0);
    traceCallback?.('multiSet', `[${keyValuePairs.length} pairs]`, durationMs, totalSize);
  },

  async getAllKeys(): Promise<readonly string[]> {
    const startTime = performance.now();
    const keys = await AsyncStorage.getAllKeys();
    const durationMs = performance.now() - startTime;

    traceCallback?.('getAllKeys', '*', durationMs, keys.length);

    return keys;
  },

  async clear(): Promise<void> {
    const startTime = performance.now();
    await AsyncStorage.clear();
    const durationMs = performance.now() - startTime;

    traceCallback?.('clear', '*', durationMs);
  },
};
```

### WatermelonDB Tracing

```typescript
// watermelonTracer.ts
import { Database, Query, Model } from '@nozbe/watermelondb';
import { Performance } from 'react-native-performance';

export function traceQuery<T extends Model>(
  query: Query<T>,
  screen?: string
): Query<T> {
  const originalFetch = query.fetch.bind(query);
  const originalObserve = query.observe.bind(query);

  query.fetch = async () => {
    const startTime = Performance.now();
    const results = await originalFetch();
    const durationMs = Performance.now() - startTime;

    recordWatermelonQuery(
      query.modelClass.table,
      'fetch',
      durationMs,
      results.length,
      screen
    );

    return results;
  };

  query.observe = () => {
    return originalObserve().map((results) => {
      recordWatermelonQuery(
        query.modelClass.table,
        'observe.emit',
        0, // Can't measure observe timing easily
        results.length,
        screen
      );
      return results;
    });
  };

  return query;
}

function recordWatermelonQuery(
  table: string,
  operation: string,
  durationMs: number,
  count: number,
  screen?: string
) {
  // Send to observability
  Observability.recordMetric({
    name: 'watermelondb.query.duration',
    value: durationMs,
    unit: 'milliseconds',
    tags: {
      table,
      operation,
      screen: screen ?? 'unknown',
    },
  });

  if (durationMs > 100) {
    Observability.captureMessage('Slow WatermelonDB query', {
      level: 'warning',
      extras: {
        table,
        operation,
        duration_ms: durationMs,
        count,
        screen,
      },
    });
  }
}

// Trace write operations
export async function traceWrite<T>(
  database: Database,
  operation: string,
  block: () => Promise<T>
): Promise<T> {
  const startTime = Performance.now();

  const result = await database.write(block);

  const durationMs = Performance.now() - startTime;

  Observability.recordMetric({
    name: 'watermelondb.write.duration',
    value: durationMs,
    unit: 'milliseconds',
    tags: { operation },
  });

  return result;
}
```

---

## MCP Server for Database Observability

Universal MCP server for agents to query database performance across frameworks.

### Tool Definitions

```json
{
  "tools": [
    {
      "name": "db_query_stats",
      "description": "Get aggregated query statistics (count, avg/p50/p95/p99 duration) grouped by normalized SQL",
      "inputSchema": {
        "type": "object",
        "properties": {
          "framework": {
            "type": "string",
            "enum": ["coredata", "swiftdata", "grdb", "room", "sqldelight", "realm", "watermelondb"],
            "description": "Which persistence framework to query"
          },
          "timeRangeSeconds": {
            "type": "integer",
            "description": "How far back to query (default: 3600)"
          },
          "minDurationMs": {
            "type": "number",
            "description": "Only return queries slower than this threshold"
          },
          "table": {
            "type": "string",
            "description": "Filter by table/entity name"
          }
        },
        "required": ["framework"]
      }
    },
    {
      "name": "db_slow_queries",
      "description": "Get list of slow queries with full SQL and context",
      "inputSchema": {
        "type": "object",
        "properties": {
          "framework": { "type": "string" },
          "thresholdMs": { "type": "number", "default": 100 },
          "limit": { "type": "integer", "default": 20 }
        },
        "required": ["framework"]
      }
    },
    {
      "name": "db_explain_query",
      "description": "Run EXPLAIN QUERY PLAN on a SQL statement",
      "inputSchema": {
        "type": "object",
        "properties": {
          "sql": { "type": "string", "description": "SQL to analyze" },
          "framework": { "type": "string" }
        },
        "required": ["sql", "framework"]
      }
    },
    {
      "name": "db_table_stats",
      "description": "Get statistics about a table (row count, size, indexes)",
      "inputSchema": {
        "type": "object",
        "properties": {
          "table": { "type": "string" },
          "framework": { "type": "string" }
        },
        "required": ["table", "framework"]
      }
    },
    {
      "name": "db_index_usage",
      "description": "Analyze index usage and suggest missing indexes",
      "inputSchema": {
        "type": "object",
        "properties": {
          "framework": { "type": "string" },
          "table": { "type": "string" }
        },
        "required": ["framework"]
      }
    },
    {
      "name": "db_migration_history",
      "description": "Get migration history with timing",
      "inputSchema": {
        "type": "object",
        "properties": {
          "framework": { "type": "string" }
        },
        "required": ["framework"]
      }
    }
  ]
}
```

### MCP Server Implementation

```swift
// DatabaseMCPServer.swift
import Foundation

final class DatabaseMCPServer {
    private let coreDataTracer = CoreDataTracer.shared
    private let grdbTracer = GRDBTracer.shared
    private let realmTracer = RealmTracer.shared

    struct MCPRequest: Codable {
        let method: String
        let params: [String: AnyCodable]
    }

    struct MCPResponse: Codable {
        let success: Bool
        let data: AnyCodable?
        let error: String?
    }

    func handleRequest(_ request: MCPRequest) -> MCPResponse {
        switch request.method {
        case "db_query_stats":
            return handleQueryStats(request.params)
        case "db_slow_queries":
            return handleSlowQueries(request.params)
        case "db_explain_query":
            return handleExplainQuery(request.params)
        case "db_table_stats":
            return handleTableStats(request.params)
        case "db_index_usage":
            return handleIndexUsage(request.params)
        default:
            return MCPResponse(success: false, data: nil, error: "Unknown method")
        }
    }

    private func handleQueryStats(_ params: [String: AnyCodable]) -> MCPResponse {
        guard let framework = params["framework"]?.value as? String else {
            return MCPResponse(success: false, data: nil, error: "framework required")
        }

        let stats: [String: Any]

        switch framework {
        case "grdb":
            let grdbStats = grdbTracer.exportStats()
            stats = grdbStats.mapValues { stat in
                [
                    "count": stat.count,
                    "avg_ms": stat.avgDurationMs,
                    "max_ms": stat.maxDurationMs,
                    "min_ms": stat.minDurationMs,
                    "total_ms": stat.totalDurationMs
                ]
            }
        default:
            return MCPResponse(success: false, data: nil, error: "Unsupported framework: \(framework)")
        }

        return MCPResponse(success: true, data: AnyCodable(stats), error: nil)
    }

    private func handleSlowQueries(_ params: [String: AnyCodable]) -> MCPResponse {
        // Implementation varies by framework
        // Return recent queries exceeding threshold
        return MCPResponse(success: true, data: nil, error: nil)
    }

    private func handleExplainQuery(_ params: [String: AnyCodable]) -> MCPResponse {
        guard let sql = params["sql"]?.value as? String else {
            return MCPResponse(success: false, data: nil, error: "sql required")
        }

        // Run EXPLAIN QUERY PLAN
        // Return plan steps
        return MCPResponse(success: true, data: nil, error: nil)
    }

    private func handleTableStats(_ params: [String: AnyCodable]) -> MCPResponse {
        guard let table = params["table"]?.value as? String else {
            return MCPResponse(success: false, data: nil, error: "table required")
        }

        // Query sqlite_master, count rows, etc.
        return MCPResponse(success: true, data: nil, error: nil)
    }

    private func handleIndexUsage(_ params: [String: AnyCodable]) -> MCPResponse {
        // Analyze query patterns vs available indexes
        // Suggest missing indexes
        return MCPResponse(success: true, data: nil, error: nil)
    }
}
```

---

## Summary: Key Metrics by Framework

| Framework | Query Timing | Transaction Timing | Explain Plan | Batch Stats |
|-----------|--------------|-------------------|--------------|-------------|
| **SQLite** | sqlite3_trace_v2 | Manual | EXPLAIN QUERY PLAN | Manual |
| **Core Data** | Fetch wrapper | Save wrapper | N/A (use SQLDebug) | insertedObjects.count |
| **SwiftData** | Fetch wrapper | Save wrapper | N/A | hasChanges |
| **GRDB** | Built-in trace | Write wrapper | Built-in | Statement stats |
| **Room** | QueryCallback | @Transaction | EXPLAIN | Manual |
| **SQLDelight** | Driver wrapper | Transaction | Manual | Manual |
| **Realm** | Query wrapper | write() wrapper | N/A | Sync progress |
| **WatermelonDB** | Query wrapper | write() wrapper | N/A | Batch tracking |

---

## CLI Tools Reference

| Platform | Tool | Command |
|----------|------|---------|
| iOS | Instruments | `xctrace record --template 'Core Data'` |
| iOS | sqlite3 | `sqlite3 app.db '.stats on'` |
| Android | Perfetto | `adb shell perfetto -c - -o trace database` |
| Android | adb | `adb shell dumpsys dbinfo <package>` |
| React Native | Flipper | `flipper-server` + Database plugin |
| All | sqlite3 | `sqlite3 db.sqlite 'EXPLAIN QUERY PLAN SELECT...'` |
