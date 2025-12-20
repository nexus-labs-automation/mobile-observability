# Native Mobile Instrumentation

Platform-specific patterns for native iOS (Swift/SwiftUI) and Android (Kotlin/Compose), including low-level APIs like os_signpost, MetricKit, and Perfetto.

## Table of Contents
1. [Architecture Differences](#architecture-differences)
2. [iOS Signpost Instrumentation](#ios-signpost-instrumentation)
3. [Signpost Extraction for Agents](#signpost-extraction-for-agents)
4. [MetricKit Integration](#metrickit-integration)
5. [Android Perfetto/Trace](#android-perfetto-trace)
6. [Platform-Specific Observability APIs](#platform-specific-observability-apis)

---

## Architecture Differences

| Aspect | iOS | Android |
|--------|-----|---------|
| **Tracing** | os_signpost, Instruments | Perfetto, Systrace |
| **System Metrics** | MetricKit (24h delay) | Android Vitals (delayed) |
| **Crash Reports** | MetricKit, 3rd party | Tombstones, 3rd party |
| **Power Metrics** | MetricKit, Energy Log | Battery Historian |
| **Programmatic Access** | OSLogStore (limited) | Perfetto SDK, Debug API |

---

## iOS Signpost Instrumentation

`os_signpost` is Apple's high-performance tracing API, used by Instruments.

### Basic Signpost Usage

```swift
import os.signpost

// Create a dedicated log for your subsystem
let performanceLog = OSLog(subsystem: "com.yourapp.performance", category: "Navigation")
let networkLog = OSLog(subsystem: "com.yourapp.performance", category: "Network")

// MARK: - Interval Signposts (measure duration)

func loadProduct(id: String) async throws -> Product {
    let signpostID = OSSignpostID(log: performanceLog)

    os_signpost(.begin, log: performanceLog, name: "LoadProduct", signpostID: signpostID,
                "product_id: %{public}s", id)

    defer {
        os_signpost(.end, log: performanceLog, name: "LoadProduct", signpostID: signpostID,
                    "product_id: %{public}s, status: complete", id)
    }

    let product = try await api.fetchProduct(id: id)
    return product
}

// MARK: - Event Signposts (point in time)

func trackButtonTap(_ buttonName: String) {
    os_signpost(.event, log: performanceLog, name: "ButtonTap",
                "button: %{public}s, screen: %{public}s", buttonName, currentScreen)
}

// MARK: - Points of Interest (shows in Instruments timeline)

let poiLog = OSLog(subsystem: "com.yourapp", category: .pointsOfInterest)

func markCheckoutStarted() {
    os_signpost(.event, log: poiLog, name: "Checkout",
                "event: started, cart_items: %d", cart.items.count)
}
```

### Structured Signpost Wrapper

```swift
// SignpostTracer.swift
import os.signpost

final class SignpostTracer {
    static let shared = SignpostTracer()

    private let logs: [String: OSLog] = [
        "navigation": OSLog(subsystem: Bundle.main.bundleIdentifier!, category: "Navigation"),
        "network": OSLog(subsystem: Bundle.main.bundleIdentifier!, category: "Network"),
        "render": OSLog(subsystem: Bundle.main.bundleIdentifier!, category: "Render"),
        "data": OSLog(subsystem: Bundle.main.bundleIdentifier!, category: "DataLoad"),
        "poi": OSLog(subsystem: Bundle.main.bundleIdentifier!, category: .pointsOfInterest)
    ]

    struct Span {
        let log: OSLog
        let name: StaticString
        let signpostID: OSSignpostID
        let startTime: CFAbsoluteTime

        func end(metadata: String = "") {
            os_signpost(.end, log: log, name: name, signpostID: signpostID, "%{public}s", metadata)
        }
    }

    func startSpan(category: String, name: StaticString, metadata: String = "") -> Span {
        let log = logs[category] ?? logs["poi"]!
        let signpostID = OSSignpostID(log: log)

        os_signpost(.begin, log: log, name: name, signpostID: signpostID, "%{public}s", metadata)

        return Span(log: log, name: name, signpostID: signpostID, startTime: CFAbsoluteTimeGetCurrent())
    }

    func event(category: String, name: StaticString, metadata: String = "") {
        let log = logs[category] ?? logs["poi"]!
        os_signpost(.event, log: log, name: name, "%{public}s", metadata)
    }

    // Convenience for async operations
    func trace<T>(category: String, name: StaticString, metadata: String = "",
                  operation: () async throws -> T) async rethrows -> T {
        let span = startSpan(category: category, name: name, metadata: metadata)
        defer { span.end() }
        return try await operation()
    }
}

// Usage
let product = await SignpostTracer.shared.trace(
    category: "network",
    name: "FetchProduct",
    metadata: "id: \(productId)"
) {
    try await api.fetchProduct(id: productId)
}
```

### SwiftUI View Signposts

```swift
// SignpostViewModifier.swift
import SwiftUI
import os.signpost

struct SignpostViewModifier: ViewModifier {
    let name: String
    let category: String

    private let log: OSLog
    @State private var signpostID: OSSignpostID?
    @State private var hasAppeared = false

    init(name: String, category: String = "Render") {
        self.name = name
        self.category = category
        self.log = OSLog(subsystem: Bundle.main.bundleIdentifier!, category: category)
    }

    func body(content: Content) -> some View {
        content
            .onAppear {
                guard !hasAppeared else { return }
                hasAppeared = true

                let id = OSSignpostID(log: log)
                signpostID = id
                os_signpost(.begin, log: log, name: "ViewRender", signpostID: id,
                            "view: %{public}s", name)
            }
            .background(
                GeometryReader { _ in
                    Color.clear
                        .onAppear {
                            if let id = signpostID {
                                os_signpost(.end, log: log, name: "ViewRender", signpostID: id,
                                            "view: %{public}s, status: rendered", name)
                                signpostID = nil
                            }
                        }
                }
            )
    }
}

extension View {
    func signpost(_ name: String, category: String = "Render") -> some View {
        modifier(SignpostViewModifier(name: name, category: category))
    }
}

// Usage
struct ProductDetailView: View {
    var body: some View {
        ScrollView {
            ProductHeader()
                .signpost("ProductHeader")
            ProductImages()
                .signpost("ProductImages")
            ProductDescription()
                .signpost("ProductDescription")
        }
        .signpost("ProductDetailView")
    }
}
```

---

## Signpost Extraction for Agents

Apple doesn't make signpost data easily accessible programmatically, but there are several approaches for agent/MCP server access.

### Approach 1: xctrace CLI (Development/CI)

Best for: Automated testing, CI pipelines, development debugging

```bash
#!/bin/bash
# signpost_capture.sh - Capture and export signpost traces

APP_BUNDLE="com.yourapp"
OUTPUT_DIR="./traces"
DURATION=30

# Record trace
xctrace record \
    --template 'os_signpost' \
    --device-name 'iPhone 15 Pro' \
    --time-limit "${DURATION}s" \
    --launch "$APP_BUNDLE" \
    --output "${OUTPUT_DIR}/recording.trace"

# Export signpost data to XML
xctrace export \
    --input "${OUTPUT_DIR}/recording.trace" \
    --output "${OUTPUT_DIR}/signposts.xml" \
    --xpath '//os-signpost'

# Alternative: Export specific tables
xctrace export \
    --input "${OUTPUT_DIR}/recording.trace" \
    --output "${OUTPUT_DIR}/signpost-intervals.json" \
    --xpath '//os-signpost-interval-schema'
```

#### Parsing xctrace Output

```python
# parse_xctrace.py
import xml.etree.ElementTree as ET
import json
from dataclasses import dataclass, asdict
from typing import Optional
from datetime import datetime

@dataclass
class SignpostSpan:
    name: str
    subsystem: str
    category: str
    start_ns: int
    end_ns: Optional[int]
    duration_ms: Optional[float]
    metadata: dict

def parse_xctrace_xml(xml_path: str) -> list[SignpostSpan]:
    tree = ET.parse(xml_path)
    root = tree.getroot()

    spans = []
    open_intervals = {}  # signpost_id -> SignpostSpan

    for event in root.findall('.//os-signpost'):
        signpost_type = event.get('type')  # 'Begin', 'End', 'Event'
        signpost_id = event.get('signpost-id')
        name = event.get('name')
        subsystem = event.get('subsystem')
        category = event.get('category')
        timestamp = int(event.get('timestamp', 0))
        message = event.get('message', '')

        if signpost_type == 'Begin':
            open_intervals[signpost_id] = SignpostSpan(
                name=name,
                subsystem=subsystem,
                category=category,
                start_ns=timestamp,
                end_ns=None,
                duration_ms=None,
                metadata={'message': message}
            )

        elif signpost_type == 'End':
            if signpost_id in open_intervals:
                span = open_intervals.pop(signpost_id)
                span.end_ns = timestamp
                span.duration_ms = (timestamp - span.start_ns) / 1_000_000
                spans.append(span)

        elif signpost_type == 'Event':
            spans.append(SignpostSpan(
                name=name,
                subsystem=subsystem,
                category=category,
                start_ns=timestamp,
                end_ns=timestamp,
                duration_ms=0,
                metadata={'message': message}
            ))

    return spans

def export_to_otlp(spans: list[SignpostSpan]) -> dict:
    """Convert to OpenTelemetry format for ingestion"""
    return {
        "resourceSpans": [{
            "resource": {
                "attributes": [
                    {"key": "service.name", "value": {"stringValue": "ios-app"}},
                    {"key": "telemetry.sdk.name", "value": {"stringValue": "os_signpost"}}
                ]
            },
            "scopeSpans": [{
                "scope": {"name": "os_signpost"},
                "spans": [
                    {
                        "traceId": generate_trace_id(),
                        "spanId": generate_span_id(),
                        "name": span.name,
                        "startTimeUnixNano": span.start_ns,
                        "endTimeUnixNano": span.end_ns or span.start_ns,
                        "attributes": [
                            {"key": "signpost.subsystem", "value": {"stringValue": span.subsystem}},
                            {"key": "signpost.category", "value": {"stringValue": span.category}},
                        ]
                    }
                    for span in spans
                ]
            }]
        }]
    }

if __name__ == "__main__":
    spans = parse_xctrace_xml("./traces/signposts.xml")
    print(json.dumps([asdict(s) for s in spans], indent=2))
```

### Approach 2: OSLogStore (In-App, iOS 15+)

Best for: Production apps, real-time export, user-triggered debugging

```swift
// SignpostExporter.swift
import OSLog
import Foundation

struct ExportedSignpost: Codable {
    let name: String
    let subsystem: String
    let category: String
    let type: String // "begin", "end", "event"
    let timestamp: Date
    let signpostID: UInt64
    let message: String
}

struct ReconstructedSpan: Codable {
    let name: String
    let subsystem: String
    let category: String
    let startTime: Date
    let endTime: Date
    let durationMs: Double
    let metadata: [String: String]
}

final class SignpostExporter {
    enum ExportError: Error {
        case storeUnavailable
        case exportFailed(String)
    }

    /// Query signposts from the current process
    func querySignposts(
        subsystem: String? = nil,
        category: String? = nil,
        since: TimeInterval = 3600 // Last hour
    ) throws -> [ExportedSignpost] {
        guard #available(iOS 15.0, macOS 12.0, *) else {
            throw ExportError.storeUnavailable
        }

        let store = try OSLogStore(scope: .currentProcessIdentifier)
        let position = store.position(timeIntervalSinceLatestBoot: -since)

        var results: [ExportedSignpost] = []

        for entry in try store.getEntries(at: position) {
            guard let signpost = entry as? OSLogEntrySignpost else { continue }

            // Apply filters
            if let subsystem = subsystem, signpost.subsystem != subsystem { continue }
            if let category = category, signpost.category != category { continue }

            let typeString: String
            switch signpost.signpostType {
            case .intervalBegin: typeString = "begin"
            case .intervalEnd: typeString = "end"
            case .event: typeString = "event"
            case .undefined: typeString = "undefined"
            @unknown default: typeString = "unknown"
            }

            results.append(ExportedSignpost(
                name: signpost.signpostName,
                subsystem: signpost.subsystem,
                category: signpost.category,
                type: typeString,
                timestamp: signpost.date,
                signpostID: signpost.signpostIdentifier.rawValue,
                message: signpost.composedMessage
            ))
        }

        return results.sorted { $0.timestamp < $1.timestamp }
    }

    /// Reconstruct intervals from begin/end pairs
    func reconstructSpans(
        subsystem: String? = nil,
        category: String? = nil,
        since: TimeInterval = 3600
    ) throws -> [ReconstructedSpan] {
        let signposts = try querySignposts(subsystem: subsystem, category: category, since: since)

        var openSpans: [String: ExportedSignpost] = [:] // key: "name-signpostID"
        var completedSpans: [ReconstructedSpan] = []

        for signpost in signposts {
            let key = "\(signpost.name)-\(signpost.signpostID)"

            switch signpost.type {
            case "begin":
                openSpans[key] = signpost

            case "end":
                if let beginSignpost = openSpans.removeValue(forKey: key) {
                    let duration = signpost.timestamp.timeIntervalSince(beginSignpost.timestamp) * 1000
                    completedSpans.append(ReconstructedSpan(
                        name: signpost.name,
                        subsystem: signpost.subsystem,
                        category: signpost.category,
                        startTime: beginSignpost.timestamp,
                        endTime: signpost.timestamp,
                        durationMs: duration,
                        metadata: parseMetadata(begin: beginSignpost.message, end: signpost.message)
                    ))
                }

            case "event":
                completedSpans.append(ReconstructedSpan(
                    name: signpost.name,
                    subsystem: signpost.subsystem,
                    category: signpost.category,
                    startTime: signpost.timestamp,
                    endTime: signpost.timestamp,
                    durationMs: 0,
                    metadata: ["message": signpost.message]
                ))

            default:
                break
            }
        }

        return completedSpans.sorted { $0.startTime < $1.startTime }
    }

    private func parseMetadata(begin: String, end: String) -> [String: String] {
        // Parse key: value pairs from signpost messages
        var metadata: [String: String] = [:]

        for message in [begin, end] {
            let pairs = message.components(separatedBy: ", ")
            for pair in pairs {
                let parts = pair.components(separatedBy: ": ")
                if parts.count == 2 {
                    metadata[parts[0].trimmingCharacters(in: .whitespaces)] = parts[1]
                }
            }
        }

        return metadata
    }

    /// Export to JSON file
    func exportToFile(
        subsystem: String? = nil,
        category: String? = nil,
        since: TimeInterval = 3600
    ) throws -> URL {
        let spans = try reconstructSpans(subsystem: subsystem, category: category, since: since)

        let encoder = JSONEncoder()
        encoder.dateEncodingStrategy = .iso8601
        encoder.outputFormatting = [.prettyPrinted, .sortedKeys]

        let data = try encoder.encode(spans)

        let fileURL = FileManager.default.temporaryDirectory
            .appendingPathComponent("signposts_\(Date().timeIntervalSince1970).json")

        try data.write(to: fileURL)
        return fileURL
    }
}
```

### Approach 3: MCP Server for Agent Access

Best for: AI agent integration, Claude Code workflows, automated debugging

```swift
// SignpostMCPServer.swift
import Foundation
import OSLog

// MARK: - MCP Protocol Types

struct MCPRequest: Codable {
    let method: String
    let params: SignpostQueryParams?
}

struct SignpostQueryParams: Codable {
    let subsystem: String?
    let category: String?
    let signpostName: String?
    let timeRangeSeconds: Int?
    let outputFormat: OutputFormat?

    enum OutputFormat: String, Codable {
        case json
        case otlp
        case csv
    }
}

struct MCPResponse: Codable {
    let success: Bool
    let data: SignpostData?
    let error: String?
}

struct SignpostData: Codable {
    let spans: [ReconstructedSpan]
    let summary: SpanSummary
}

struct SpanSummary: Codable {
    let totalSpans: Int
    let totalDurationMs: Double
    let avgDurationMs: Double
    let p50DurationMs: Double
    let p95DurationMs: Double
    let p99DurationMs: Double
    let spansByCategory: [String: Int]
}

// MARK: - MCP Server

final class SignpostMCPServer {
    private let exporter = SignpostExporter()

    func handleRequest(_ request: MCPRequest) -> MCPResponse {
        switch request.method {
        case "signpost.query":
            return handleQuery(request.params)
        case "signpost.export":
            return handleExport(request.params)
        case "signpost.summary":
            return handleSummary(request.params)
        default:
            return MCPResponse(success: false, data: nil, error: "Unknown method: \(request.method)")
        }
    }

    private func handleQuery(_ params: SignpostQueryParams?) -> MCPResponse {
        do {
            let spans = try exporter.reconstructSpans(
                subsystem: params?.subsystem,
                category: params?.category,
                since: TimeInterval(params?.timeRangeSeconds ?? 3600)
            )

            // Filter by name if specified
            let filteredSpans: [ReconstructedSpan]
            if let name = params?.signpostName {
                filteredSpans = spans.filter { $0.name == name }
            } else {
                filteredSpans = spans
            }

            let summary = calculateSummary(filteredSpans)

            return MCPResponse(
                success: true,
                data: SignpostData(spans: filteredSpans, summary: summary),
                error: nil
            )
        } catch {
            return MCPResponse(success: false, data: nil, error: error.localizedDescription)
        }
    }

    private func handleExport(_ params: SignpostQueryParams?) -> MCPResponse {
        do {
            let fileURL = try exporter.exportToFile(
                subsystem: params?.subsystem,
                category: params?.category,
                since: TimeInterval(params?.timeRangeSeconds ?? 3600)
            )

            // Return file path for agent to access
            return MCPResponse(
                success: true,
                data: SignpostData(
                    spans: [],
                    summary: SpanSummary(
                        totalSpans: 0,
                        totalDurationMs: 0,
                        avgDurationMs: 0,
                        p50DurationMs: 0,
                        p95DurationMs: 0,
                        p99DurationMs: 0,
                        spansByCategory: ["exported_to": fileURL.path]
                    )
                ),
                error: nil
            )
        } catch {
            return MCPResponse(success: false, data: nil, error: error.localizedDescription)
        }
    }

    private func handleSummary(_ params: SignpostQueryParams?) -> MCPResponse {
        do {
            let spans = try exporter.reconstructSpans(
                subsystem: params?.subsystem,
                category: params?.category,
                since: TimeInterval(params?.timeRangeSeconds ?? 3600)
            )

            let summary = calculateSummary(spans)

            return MCPResponse(
                success: true,
                data: SignpostData(spans: [], summary: summary),
                error: nil
            )
        } catch {
            return MCPResponse(success: false, data: nil, error: error.localizedDescription)
        }
    }

    private func calculateSummary(_ spans: [ReconstructedSpan]) -> SpanSummary {
        guard !spans.isEmpty else {
            return SpanSummary(
                totalSpans: 0,
                totalDurationMs: 0,
                avgDurationMs: 0,
                p50DurationMs: 0,
                p95DurationMs: 0,
                p99DurationMs: 0,
                spansByCategory: [:]
            )
        }

        let durations = spans.map(\.durationMs).sorted()
        let total = durations.reduce(0, +)

        return SpanSummary(
            totalSpans: spans.count,
            totalDurationMs: total,
            avgDurationMs: total / Double(spans.count),
            p50DurationMs: percentile(durations, 0.50),
            p95DurationMs: percentile(durations, 0.95),
            p99DurationMs: percentile(durations, 0.99),
            spansByCategory: Dictionary(grouping: spans, by: \.category).mapValues(\.count)
        )
    }

    private func percentile(_ sortedValues: [Double], _ p: Double) -> Double {
        guard !sortedValues.isEmpty else { return 0 }
        let index = Int(Double(sortedValues.count - 1) * p)
        return sortedValues[index]
    }
}

// MARK: - MCP Tool Definition (for agent registration)

let signpostMCPTools = """
{
  "tools": [
    {
      "name": "signpost_query",
      "description": "Query os_signpost traces from iOS app. Returns spans with timing data.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "subsystem": {
            "type": "string",
            "description": "Filter by signpost subsystem (e.g., 'com.yourapp.performance')"
          },
          "category": {
            "type": "string",
            "description": "Filter by category (e.g., 'Navigation', 'Network', 'Render')"
          },
          "signpostName": {
            "type": "string",
            "description": "Filter by specific signpost name"
          },
          "timeRangeSeconds": {
            "type": "integer",
            "description": "How far back to query (default: 3600 = 1 hour)"
          }
        }
      }
    },
    {
      "name": "signpost_summary",
      "description": "Get statistical summary of signpost data (p50, p95, p99 latencies)",
      "inputSchema": {
        "type": "object",
        "properties": {
          "subsystem": { "type": "string" },
          "category": { "type": "string" },
          "timeRangeSeconds": { "type": "integer" }
        }
      }
    },
    {
      "name": "signpost_export",
      "description": "Export signpost data to JSON file for external analysis",
      "inputSchema": {
        "type": "object",
        "properties": {
          "subsystem": { "type": "string" },
          "category": { "type": "string" },
          "timeRangeSeconds": { "type": "integer" },
          "outputFormat": {
            "type": "string",
            "enum": ["json", "otlp", "csv"]
          }
        }
      }
    }
  ]
}
"""
```

### Approach 4: Hybrid Architecture

For your universal observability MCP server, combine approaches:

```
┌─────────────────────────────────────────────────────────────────┐
│                     iOS App                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  os_signpost instrumentation                             │    │
│  │  • Navigation: "screen_load", "navigation"               │    │
│  │  • Network: "api_request", "image_load"                  │    │
│  │  • Render: "view_body", "list_diff"                      │    │
│  └─────────────────────┬───────────────────────────────────┘    │
│                        │                                         │
│  ┌─────────────────────▼───────────────────────────────────┐    │
│  │  SignpostBridge                                          │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │    │
│  │  │ OSLogStore  │  │  Real-time  │  │  Export to  │      │    │
│  │  │   Query     │  │   Stream    │  │  Sentry/DD  │      │    │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘      │    │
│  │         │                │                │              │    │
│  │         └────────────────┼────────────────┘              │    │
│  │                          ▼                               │    │
│  │  ┌───────────────────────────────────────────────────┐  │    │
│  │  │  Unified Span Format                               │  │    │
│  │  │  { name, category, start, end, duration, meta }   │  │    │
│  │  └───────────────────────────────────────────────────┘  │    │
│  └─────────────────────────┬───────────────────────────────┘    │
└─────────────────────────────┼───────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
   │  MCP Server │    │   Sentry    │    │   Datadog   │
   │  (Agents)   │    │   (Prod)    │    │   (Prod)    │
   └─────────────┘    └─────────────┘    └─────────────┘
```

### Comparison of Approaches

| Approach | Latency | Fidelity | Requires Mac | Production Ready | Agent Access |
|----------|---------|----------|--------------|------------------|--------------|
| xctrace CLI | Post-hoc | Full | Yes | No (dev only) | ✅ Via files |
| OSLogStore | Real-time | Good* | No | Yes | ✅ Via MCP |
| log CLI | Real-time | Partial | Yes (USB) | No | ⚠️ Limited |
| Custom export | Real-time | Full | No | Yes | ✅ Via API |

*OSLogStore may miss some signposts under high load

### Limitations & Gotchas

1. **OSLogStore Scope**: Can only access current process signposts
2. **Timing Precision**: OSLogStore timestamps are less precise than Instruments
3. **Buffer Limits**: System may drop signposts under extreme load
4. **Privacy**: Some signpost data marked `%{private}` won't be readable
5. **Background**: OSLogStore queries may not work in background
6. **Simulator vs Device**: Some APIs behave differently

---

## MetricKit Integration

MetricKit provides system-level metrics with 24-hour aggregation.

### Setup

```swift
// MetricKitManager.swift
import MetricKit

final class MetricKitManager: NSObject, MXMetricManagerSubscriber {
    static let shared = MetricKitManager()

    override init() {
        super.init()
        MXMetricManager.shared.add(self)
    }

    deinit {
        MXMetricManager.shared.remove(self)
    }

    // MARK: - MXMetricManagerSubscriber

    func didReceive(_ payloads: [MXMetricPayload]) {
        for payload in payloads {
            processMetricPayload(payload)
        }
    }

    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        for payload in payloads {
            processDiagnosticPayload(payload)
        }
    }

    // MARK: - Metric Processing

    private func processMetricPayload(_ payload: MXMetricPayload) {
        // App Launch Metrics
        if let launchMetrics = payload.applicationLaunchMetrics {
            processLaunchMetrics(launchMetrics)
        }

        // Memory Metrics
        if let memoryMetrics = payload.memoryMetrics {
            Observability.recordMetric(
                name: "metrickit.memory.peak",
                value: memoryMetrics.peakMemoryUsage.doubleValue(for: .megabytes),
                unit: .megabytes
            )
            Observability.recordMetric(
                name: "metrickit.memory.average_suspended",
                value: memoryMetrics.averageSuspendedMemory.averageMeasurement.doubleValue(for: .megabytes),
                unit: .megabytes
            )
        }

        // CPU Metrics
        if let cpuMetrics = payload.cpuMetrics {
            Observability.recordMetric(
                name: "metrickit.cpu.cumulative_time",
                value: cpuMetrics.cumulativeCPUTime.doubleValue(for: .seconds),
                unit: .seconds
            )
            if let instructions = cpuMetrics.cumulativeCPUInstructions {
                Observability.recordMetric(
                    name: "metrickit.cpu.instructions",
                    value: instructions.doubleValue(for: .kiloInstructions),
                    unit: .count
                )
            }
        }

        // Disk I/O Metrics
        if let diskMetrics = payload.diskIOMetrics {
            Observability.recordMetric(
                name: "metrickit.disk.writes_cumulative",
                value: diskMetrics.cumulativeLogicalWrites.doubleValue(for: .megabytes),
                unit: .megabytes
            )
        }

        // Network Metrics
        if let networkMetrics = payload.networkTransferMetrics {
            Observability.recordMetric(
                name: "metrickit.network.cellular_upload",
                value: networkMetrics.cumulativeCellularUpload.doubleValue(for: .megabytes),
                unit: .megabytes
            )
            Observability.recordMetric(
                name: "metrickit.network.wifi_download",
                value: networkMetrics.cumulativeWifiDownload.doubleValue(for: .megabytes),
                unit: .megabytes
            )
        }

        // Animation Metrics (iOS 14+)
        if let animationMetrics = payload.animationMetrics {
            Observability.recordMetric(
                name: "metrickit.animation.scroll_hitch_rate",
                value: animationMetrics.scrollHitchTimeRatio.doubleValue(for: .milliseconds),
                unit: .ratio
            )
        }

        // App Exit Metrics (iOS 14+)
        if let exitMetrics = payload.applicationExitMetrics {
            processExitMetrics(exitMetrics)
        }
    }

    private func processLaunchMetrics(_ metrics: MXAppLaunchMetric) {
        // Histogram data
        let coldLaunch = metrics.histogrammedTimeToFirstDraw
        let resume = metrics.histogrammedApplicationResumeTime

        // Extract percentiles from histogram
        Observability.recordMetric(
            name: "metrickit.launch.cold.p50",
            value: extractPercentile(coldLaunch, 0.50),
            unit: .milliseconds
        )
        Observability.recordMetric(
            name: "metrickit.launch.cold.p95",
            value: extractPercentile(coldLaunch, 0.95),
            unit: .milliseconds
        )

        // iOS 16+ extended launch metrics
        if #available(iOS 16.0, *) {
            if let optimizedLaunch = metrics.histogrammedOptimizedTimeToFirstDraw {
                Observability.recordMetric(
                    name: "metrickit.launch.optimized.p50",
                    value: extractPercentile(optimizedLaunch, 0.50),
                    unit: .milliseconds
                )
            }
        }
    }

    private func processExitMetrics(_ metrics: MXAppExitMetric) {
        let bg = metrics.backgroundExitData
        let fg = metrics.foregroundExitData

        // Background exits
        Observability.recordMetric(name: "metrickit.exit.bg.normal", value: Double(bg.cumulativeNormalAppExitCount))
        Observability.recordMetric(name: "metrickit.exit.bg.memory_limit", value: Double(bg.cumulativeMemoryResourceLimitExitCount))
        Observability.recordMetric(name: "metrickit.exit.bg.cpu_limit", value: Double(bg.cumulativeCPUResourceLimitExitCount))
        Observability.recordMetric(name: "metrickit.exit.bg.suspended_memory", value: Double(bg.cumulativeSuspendedWithLockedFileExitCount))
        Observability.recordMetric(name: "metrickit.exit.bg.bad_access", value: Double(bg.cumulativeBadAccessExitCount))
        Observability.recordMetric(name: "metrickit.exit.bg.watchdog", value: Double(bg.cumulativeAppWatchdogExitCount))

        // Foreground exits (crashes visible to user)
        Observability.recordMetric(name: "metrickit.exit.fg.normal", value: Double(fg.cumulativeNormalAppExitCount))
        Observability.recordMetric(name: "metrickit.exit.fg.memory", value: Double(fg.cumulativeMemoryResourceLimitExitCount))
        Observability.recordMetric(name: "metrickit.exit.fg.bad_access", value: Double(fg.cumulativeBadAccessExitCount))
        Observability.recordMetric(name: "metrickit.exit.fg.watchdog", value: Double(fg.cumulativeAppWatchdogExitCount))
        Observability.recordMetric(name: "metrickit.exit.fg.abnormal", value: Double(fg.cumulativeAbnormalExitCount))
    }

    // MARK: - Diagnostic Processing

    private func processDiagnosticPayload(_ payload: MXDiagnosticPayload) {
        // Crash Diagnostics
        if let crashes = payload.crashDiagnostics {
            for crash in crashes {
                Observability.captureMessage(
                    "MetricKit Crash",
                    level: .fatal,
                    extras: [
                        "exception_type": crash.exceptionType?.description ?? "unknown",
                        "exception_code": crash.exceptionCode?.description ?? "unknown",
                        "signal": crash.signal?.description ?? "unknown",
                        "termination_reason": crash.terminationReason ?? "unknown"
                    ]
                )
            }
        }

        // Hang Diagnostics (ANRs)
        if let hangs = payload.hangDiagnostics {
            for hang in hangs {
                Observability.captureMessage(
                    "MetricKit Hang",
                    level: .warning,
                    extras: [
                        "hang_duration_ms": hang.hangDuration.doubleValue(for: .milliseconds)
                    ]
                )
            }
        }

        // CPU Exception Diagnostics
        if let cpuExceptions = payload.cpuExceptionDiagnostics {
            for exception in cpuExceptions {
                Observability.captureMessage(
                    "MetricKit CPU Exception",
                    level: .warning,
                    extras: [
                        "total_cpu_time_ms": exception.totalCPUTime.doubleValue(for: .milliseconds),
                        "total_sampled_time_ms": exception.totalSampledTime.doubleValue(for: .milliseconds)
                    ]
                )
            }
        }

        // Disk Write Exception Diagnostics
        if let diskExceptions = payload.diskWriteExceptionDiagnostics {
            for exception in diskExceptions {
                Observability.captureMessage(
                    "MetricKit Disk Write Exception",
                    level: .warning,
                    extras: [
                        "total_writes_mb": exception.totalWritesCaused.doubleValue(for: .megabytes)
                    ]
                )
            }
        }

        // iOS 16+ App Launch Diagnostics
        if #available(iOS 16.0, *) {
            if let launchDiagnostics = payload.appLaunchDiagnostics {
                for diagnostic in launchDiagnostics {
                    Observability.captureMessage(
                        "MetricKit Slow Launch",
                        level: .warning,
                        extras: [
                            "launch_duration_ms": diagnostic.launchDuration.doubleValue(for: .milliseconds)
                        ]
                    )
                }
            }
        }
    }

    // MARK: - Helpers

    private func extractPercentile(_ histogram: MXHistogram<UnitDuration>, _ percentile: Double) -> Double {
        // MXHistogram provides bucket data, need to estimate percentile
        var cumulativeCount = 0
        let totalCount = histogram.bucketEnumerator.allObjects.reduce(0) { $0 + ($1 as! MXHistogramBucket<UnitDuration>).bucketCount }
        let targetCount = Int(Double(totalCount) * percentile)

        for bucket in histogram.bucketEnumerator.allObjects {
            let mxBucket = bucket as! MXHistogramBucket<UnitDuration>
            cumulativeCount += mxBucket.bucketCount
            if cumulativeCount >= targetCount {
                return mxBucket.bucketEnd.doubleValue(for: .milliseconds)
            }
        }

        return 0
    }
}
```

### Testing MetricKit

```bash
# Trigger diagnostic report generation (simulator only)
xcrun simctl diagnose

# On device, use Xcode Organizer or wait 24 hours

# Force MetricKit payload delivery (development only)
# Add to scheme environment variables:
# MXMetricManagerSimulatorDeliveryEnabled = 1
```

---

## Android Perfetto/Trace

Android's equivalent to Instruments for system-level tracing.

### Programmatic Tracing

```kotlin
// PerfettoTracer.kt
import android.os.Trace

object PerfettoTracer {

    // Simple synchronous trace
    inline fun <T> trace(label: String, block: () -> T): T {
        Trace.beginSection(label)
        try {
            return block()
        } finally {
            Trace.endSection()
        }
    }

    // Async trace (for operations spanning multiple methods)
    fun beginAsyncSection(methodName: String, cookie: Int) {
        Trace.beginAsyncSection(methodName, cookie)
    }

    fun endAsyncSection(methodName: String, cookie: Int) {
        Trace.endAsyncSection(methodName, cookie)
    }

    // Counter for tracking values over time
    fun setCounter(name: String, value: Long) {
        Trace.setCounter(name, value)
    }
}

// Usage
class ProductRepository {
    suspend fun loadProduct(id: String): Product {
        return PerfettoTracer.trace("ProductRepository.loadProduct") {
            val response = PerfettoTracer.trace("API.fetchProduct") {
                api.fetchProduct(id)
            }

            PerfettoTracer.trace("Product.parse") {
                response.toProduct()
            }
        }
    }
}

// Compose integration
@Composable
fun ProductScreen(viewModel: ProductViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    Trace.beginSection("ProductScreen.composition")

    when (state) {
        is Loading -> {
            Trace.beginSection("ProductScreen.Loading")
            LoadingView()
            Trace.endSection()
        }
        is Success -> {
            Trace.beginSection("ProductScreen.Success")
            ProductContent(state.product)
            Trace.endSection()
        }
    }

    Trace.endSection()
}
```

### Capturing Perfetto Traces

```bash
# Capture trace via ADB
adb shell perfetto \
    -c - \
    -o /data/misc/perfetto-traces/trace.perfetto-trace \
    <<EOF
buffers: {
    size_kb: 63488
    fill_policy: RING_BUFFER
}
data_sources: {
    config {
        name: "linux.ftrace"
        ftrace_config {
            ftrace_events: "sched/sched_switch"
            ftrace_events: "power/suspend_resume"
            atrace_categories: "view"
            atrace_categories: "wm"
            atrace_categories: "am"
            atrace_apps: "com.yourapp"
        }
    }
}
data_sources: {
    config {
        name: "android.surfaceflinger.frametimeline"
    }
}
duration_ms: 10000
EOF

# Pull trace
adb pull /data/misc/perfetto-traces/trace.perfetto-trace ./

# Open in Perfetto UI
# https://ui.perfetto.dev
```

### Parsing Perfetto Traces for Agents

```python
# parse_perfetto.py
from perfetto.trace_processor import TraceProcessor

def analyze_trace(trace_path: str) -> dict:
    tp = TraceProcessor(trace=trace_path)

    # Query app-specific slices
    app_slices = tp.query("""
        SELECT
            name,
            ts / 1e6 as start_ms,
            dur / 1e6 as duration_ms,
            category
        FROM slice
        WHERE name LIKE '%Product%'
        ORDER BY ts
    """).as_pandas_dataframe()

    # Query frame timing
    frames = tp.query("""
        SELECT
            ts / 1e6 as timestamp_ms,
            dur / 1e6 as duration_ms,
            name
        FROM slice
        WHERE name LIKE '%Choreographer%' OR name LIKE '%doFrame%'
        ORDER BY ts
    """).as_pandas_dataframe()

    # Calculate jank
    frame_durations = frames['duration_ms'].tolist()
    jank_frames = [d for d in frame_durations if d > 16.67]

    return {
        "app_slices": app_slices.to_dict('records'),
        "frame_stats": {
            "total_frames": len(frame_durations),
            "jank_frames": len(jank_frames),
            "jank_rate": len(jank_frames) / len(frame_durations) if frame_durations else 0,
            "p50_frame_ms": sorted(frame_durations)[len(frame_durations)//2] if frame_durations else 0,
            "p95_frame_ms": sorted(frame_durations)[int(len(frame_durations)*0.95)] if frame_durations else 0
        }
    }
```

---

## Platform-Specific Observability APIs

### iOS-Specific APIs

| API | Purpose | Availability |
|-----|---------|--------------|
| `os_signpost` | High-perf tracing | iOS 12+ |
| `OSLogStore` | Log/signpost queries | iOS 15+ |
| `MetricKit` | System metrics | iOS 13+ |
| `MXDiagnosticPayload` | Crash/hang diagnostics | iOS 14+ |
| `CADisplayLink` | Frame timing | iOS 3.1+ |
| `mach_absolute_time` | High-res timestamps | Always |

### Android-Specific APIs

| API | Purpose | Availability |
|-----|---------|--------------|
| `android.os.Trace` | System tracing | API 18+ |
| `Perfetto SDK` | Custom trace points | API 23+ |
| `Choreographer` | Frame callbacks | API 16+ |
| `FrameMetrics` | Per-frame timing | API 24+ |
| `System.nanoTime()` | High-res timestamps | Always |

---

## Summary

| Capability | iOS | Android |
|------------|-----|---------|
| **Tracing API** | os_signpost | Trace API |
| **Profiler** | Instruments | Perfetto |
| **CLI Tool** | xctrace | perfetto |
| **Programmatic Access** | OSLogStore | TraceProcessor |
| **System Metrics** | MetricKit | Android Vitals |
| **MCP Integration** | Via OSLogStore + export | Via TraceProcessor |

For your universal MCP server, the recommended approach is:
1. **Development**: xctrace/Perfetto CLI for full fidelity
2. **Production**: OSLogStore/custom export with sampling
3. **Agent access**: MCP tools wrapping both approaches
