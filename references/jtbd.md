# Jobs-to-be-Done for Mobile Observability

A framework for instrumenting mobile apps based on the jobs users and developers are trying to accomplish.

## Table of Contents
1. [JTBD Framework](#jtbd-framework)
2. [User Jobs](#user-jobs)
3. [Developer Jobs](#developer-jobs)
4. [Instrumentation by Job](#instrumentation-by-job)
5. [Job-Based Alerting](#job-based-alerting)

---

## JTBD Framework

### The Core Question

> "What job is this telemetry helping someone accomplish?"

Every piece of instrumentation should answer a job. If you can't name the job, don't add the telemetry.

```
┌─────────────────────────────────────────────────────────────────┐
│                    JTBD Hierarchy                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  USER JOBS                         DEVELOPER JOBS                │
│  ──────────                        ──────────────                │
│                                                                  │
│  "Help me complete                 "Help me detect               │
│   my purchase"                      problems fast"               │
│       │                                 │                        │
│       ├── Find product                  ├── Know when it breaks  │
│       ├── Compare options               ├── Find root cause      │
│       ├── Add to cart                   ├── Measure impact       │
│       └── Pay securely                  └── Verify fix worked    │
│                                                                  │
│  Instrumentation serves BOTH                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Job Categories

| Category | Description | Example Jobs |
|----------|-------------|--------------|
| **Functional** | Core task completion | Complete purchase, find information |
| **Emotional** | Feel confident/secure | Trust payment is safe, know order status |
| **Social** | Share/communicate | Share product, invite friends |
| **Operational** | Developer/ops needs | Detect issues, diagnose problems |

---

## User Jobs

### Primary User Jobs

| Job | Success Criteria | Failure Signal |
|-----|------------------|----------------|
| **Complete a purchase** | Order confirmed | Cart abandonment, payment failure |
| **Find what I need** | Item found in <30s | Search exits, rage taps |
| **Get help when stuck** | Issue resolved | Support escalation, app uninstall |
| **Stay informed** | Notifications received | Missed alerts, stale data |
| **Feel confident** | No confusion/errors | Hesitation, back navigation |

### Job: Complete a Purchase

```swift
// Instrumentation for purchase job
struct PurchaseJob {

    // Success metrics
    static func trackPurchaseComplete(orderId: String, value: Double, items: Int) {
        Observability.trackEvent("job.purchase.complete", properties: [
            "order_id": orderId,
            "value": value,
            "item_count": items,
            "time_to_complete_seconds": sessionDuration()
        ])

        Observability.recordMetric(
            name: "job.purchase.success",
            value: 1,
            tags: ["value_bucket": valueBucket(value)]
        )
    }

    // Friction points
    static func trackFriction(step: String, reason: String) {
        Observability.trackEvent("job.purchase.friction", properties: [
            "step": step,
            "reason": reason,
            "time_in_step_seconds": stepDuration()
        ])
    }

    // Failure
    static func trackAbandonment(lastStep: String, reason: String?) {
        Observability.trackEvent("job.purchase.abandoned", properties: [
            "last_step": lastStep,
            "reason": reason ?? "unknown",
            "cart_value": cartValue(),
            "items_in_cart": cartItemCount()
        ])
    }
}

// Usage at each step
class CheckoutViewModel {
    func onPaymentFailed(_ error: PaymentError) {
        PurchaseJob.trackFriction(step: "payment", reason: error.userMessage)

        // Also track for developer job
        Observability.captureError(error, context: [
            "step": "payment",
            "payment_method": selectedPaymentMethod,
            "cart_value": cart.total
        ])
    }
}
```

### Job: Find What I Need

```swift
struct SearchJob {

    static func trackSearchStarted(query: String) {
        Observability.trackEvent("job.search.started", properties: [
            "query_length": query.count,
            "has_filters": hasActiveFilters()
        ])
    }

    static func trackSearchComplete(query: String, resultCount: Int, duration: TimeInterval) {
        Observability.trackEvent("job.search.complete", properties: [
            "query": sanitizeQuery(query),
            "result_count": resultCount,
            "duration_ms": duration * 1000,
            "success": resultCount > 0
        ])

        // Zero results = job failed
        if resultCount == 0 {
            trackSearchFailed(query: query, reason: "no_results")
        }
    }

    static func trackResultSelected(query: String, position: Int, resultCount: Int) {
        Observability.trackEvent("job.search.result_selected", properties: [
            "query": sanitizeQuery(query),
            "position": position,
            "result_count": resultCount,
            "position_bucket": positionBucket(position)
        ])

        // Job success!
        Observability.recordMetric(name: "job.search.success", value: 1)
    }

    static func trackSearchFailed(query: String, reason: String) {
        Observability.trackEvent("job.search.failed", properties: [
            "query": sanitizeQuery(query),
            "reason": reason
        ])
    }

    static func trackSearchAbandoned(query: String, resultCount: Int) {
        Observability.trackEvent("job.search.abandoned", properties: [
            "query": sanitizeQuery(query),
            "result_count": resultCount,
            "time_viewing_results_seconds": timeOnResultsScreen()
        ])
    }
}
```

---

## Developer Jobs

### Primary Developer Jobs

| Job | Question Answered | Telemetry Needed |
|-----|-------------------|------------------|
| **Detect problems** | "Is something broken?" | Alerts on errors, latency spikes |
| **Diagnose root cause** | "Why did it break?" | Traces, logs, breadcrumbs |
| **Measure impact** | "How bad is it?" | User counts, error rates |
| **Verify fixes** | "Did the fix work?" | Before/after metrics |
| **Understand behavior** | "What's normal?" | Baselines, distributions |

### Job: Detect Problems

```swift
// Instrumentation that enables problem detection
struct DetectionInstrumentation {

    // Error rate monitoring
    static func trackOperationResult(operation: String, success: Bool, error: Error? = nil) {
        Observability.recordMetric(
            name: "operation.result",
            value: 1,
            tags: [
                "operation": operation,
                "success": String(success)
            ]
        )

        if let error = error {
            Observability.captureError(error, context: [
                "operation": operation
            ])
        }
    }

    // Latency monitoring
    static func trackLatency(operation: String, duration: TimeInterval) {
        Observability.recordMetric(
            name: "operation.latency",
            value: duration * 1000,
            unit: .milliseconds,
            tags: ["operation": operation]
        )

        // Alert on anomaly
        if duration > expectedLatency(for: operation) * 2 {
            Observability.captureMessage(
                "Slow operation: \(operation)",
                level: .warning,
                extras: [
                    "duration_ms": duration * 1000,
                    "expected_ms": expectedLatency(for: operation) * 1000
                ]
            )
        }
    }

    // Availability monitoring
    static func trackServiceHealth(service: String, isHealthy: Bool) {
        Observability.recordMetric(
            name: "service.health",
            value: isHealthy ? 1 : 0,
            tags: ["service": service]
        )
    }
}
```

### Job: Diagnose Root Cause

```swift
// Instrumentation that enables diagnosis
struct DiagnosisInstrumentation {

    // Rich error context
    static func captureErrorWithContext(_ error: Error, operation: String) {
        Observability.captureError(error, context: [
            // What
            "operation": operation,
            "error_type": String(describing: type(of: error)),

            // Where
            "screen": NavigationState.shared.currentScreen,
            "user_journey": JourneyTracker.getCurrentJourneyContext(),

            // When
            "session_duration_seconds": SessionManager.shared.sessionDuration,
            "time_on_screen_seconds": NavigationState.shared.currentScreenDuration,

            // Who
            "user_segment": UserManager.shared.segment,
            "is_new_user": UserManager.shared.isNewUser,

            // How
            "entry_point": SessionManager.shared.entryPoint,
            "previous_screens": NavigationState.shared.recentScreens(5),

            // Device state
            "memory_pressure": DeviceInfo.memoryPressure,
            "network_type": NetworkMonitor.shared.connectionType.rawValue,
            "battery_level": UIDevice.current.batteryLevel
        ])
    }

    // Breadcrumb trail
    static func addDiagnosticBreadcrumb(category: String, message: String, data: [String: Any] = [:]) {
        BreadcrumbManager.shared.add(
            type: .user,
            category: category,
            message: message,
            data: data.mapValues { String(describing: $0) }
        )
    }
}
```

### Job: Measure Impact

```swift
// Instrumentation that enables impact measurement
struct ImpactInstrumentation {

    // User impact
    static func trackAffectedUsers(issue: String) {
        Observability.recordMetric(
            name: "issue.affected_users",
            value: 1,
            tags: [
                "issue": issue,
                "user_id": UserManager.shared.userId ?? "anonymous"
            ]
        )
    }

    // Business impact
    static func trackRevenueImpact(issue: String, lostValue: Double) {
        Observability.recordMetric(
            name: "issue.revenue_impact",
            value: lostValue,
            tags: ["issue": issue]
        )
    }

    // Time impact
    static func trackTimeImpact(issue: String, additionalTimeSeconds: TimeInterval) {
        Observability.recordMetric(
            name: "issue.time_impact",
            value: additionalTimeSeconds,
            tags: ["issue": issue]
        )
    }
}
```

---

## Instrumentation by Job

### Job → Telemetry Mapping

| User Job | Developer Job | Telemetry |
|----------|---------------|-----------|
| Complete purchase | Detect payment issues | Payment error rate, latency |
| Complete purchase | Diagnose failures | Payment error details, breadcrumbs |
| Find what I need | Detect search issues | Zero-result rate, latency |
| Find what I need | Improve relevance | Click-through positions |
| Get help | Detect support issues | FAQ bounce rate, escalations |
| Stay informed | Detect notification issues | Delivery rate, open rate |

### Implementation Pattern

```swift
// Pattern: Every user action has job-oriented instrumentation

protocol JobInstrumented {
    var jobName: String { get }
    func trackStart()
    func trackSuccess(metadata: [String: Any])
    func trackFailure(reason: String, error: Error?)
    func trackAbandonment(lastStep: String)
}

class PurchaseFlowInstrumentation: JobInstrumented {
    let jobName = "purchase"
    private var startTime: CFAbsoluteTime?
    private var currentStep: String = "start"

    func trackStart() {
        startTime = CFAbsoluteTimeGetCurrent()
        Observability.trackEvent("job.\(jobName).started")
    }

    func trackStepComplete(_ step: String) {
        currentStep = step
        Observability.trackEvent("job.\(jobName).step", properties: [
            "step": step,
            "duration_ms": stepDuration()
        ])
    }

    func trackSuccess(metadata: [String: Any]) {
        var props = metadata
        props["total_duration_seconds"] = totalDuration()

        Observability.trackEvent("job.\(jobName).success", properties: props)
        Observability.recordMetric(name: "job.\(jobName).success", value: 1)
    }

    func trackFailure(reason: String, error: Error?) {
        Observability.trackEvent("job.\(jobName).failed", properties: [
            "reason": reason,
            "step": currentStep
        ])

        if let error = error {
            DiagnosisInstrumentation.captureErrorWithContext(error, operation: jobName)
        }
    }

    func trackAbandonment(lastStep: String) {
        Observability.trackEvent("job.\(jobName).abandoned", properties: [
            "last_step": lastStep,
            "duration_seconds": totalDuration()
        ])
    }
}
```

---

## Job-Based Alerting

### Alert Philosophy

Alert on **job failures**, not just technical metrics.

| Bad Alert | Good Alert |
|-----------|------------|
| "Error rate > 1%" | "Purchase completion rate dropped 20%" |
| "Latency > 2s" | "Users can't complete checkout in acceptable time" |
| "Memory usage high" | "App crashes preventing users from completing tasks" |

### Alert Configuration

```yaml
# alerts.yaml - Job-oriented alerting

alerts:
  # User job: Complete purchase
  - name: purchase_completion_degraded
    description: "Users unable to complete purchases"
    metric: job.purchase.success_rate
    condition: "< 95%"
    window: 15m
    severity: critical
    context:
      - job.purchase.failed (group by reason)
      - job.purchase.friction (top steps)

  - name: purchase_time_degraded
    description: "Purchase taking too long"
    metric: job.purchase.duration_p95
    condition: "> 120s"
    window: 15m
    severity: warning

  # User job: Find what I need
  - name: search_degraded
    description: "Users can't find what they need"
    metric: job.search.success_rate
    condition: "< 80%"
    window: 15m
    severity: warning
    context:
      - job.search.failed (top queries)
      - job.search.abandoned

  # Developer job: Detect problems
  - name: error_spike
    description: "Unusual error activity"
    metric: errors.count
    condition: "> 200% of baseline"
    baseline_window: 7d
    severity: warning

  # Developer job: Verify fixes
  - name: regression_detected
    description: "Metric regressed after deploy"
    metric: "*"
    condition: "degraded vs pre-deploy baseline"
    window: 1h
    trigger_on: deploy
    severity: warning
```

### Job Success Dashboards

```
┌─────────────────────────────────────────────────────────────────┐
│                    Job Success Dashboard                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PURCHASE JOB                           SEARCH JOB               │
│  ───────────                           ──────────                │
│  Success Rate: 94.2% ⚠️                 Success Rate: 87.1% ✓    │
│  Avg Duration: 45s ✓                   Avg Results: 24 ✓         │
│  Friction Points:                      Zero Results: 8.2% ✓      │
│    • Payment: 12%                                                │
│    • Shipping: 8%                      Top Failed Queries:       │
│                                          "blue widget" (0 results)│
│  ONBOARDING JOB                         "xyz123" (0 results)     │
│  ──────────────                                                  │
│  Completion Rate: 72% ⚠️               SUPPORT JOB              │
│  Avg Duration: 3.2m ✓                  ────────────              │
│  Drop-off Step: permissions            Self-Serve Rate: 68% ⚠️   │
│                                        Escalation Rate: 12%       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary

| Principle | Application |
|-----------|-------------|
| **Name the job** | Every metric answers "helps someone do X" |
| **User + Developer** | Instrument for both perspectives |
| **Success & Failure** | Track both outcomes, not just failures |
| **Context is king** | Rich context enables diagnosis |
| **Alert on jobs** | "Users can't X" not "metric Y is high" |

### Checklist: Is This Instrumentation Job-Oriented?

- [ ] Can I name the user job it supports?
- [ ] Can I name the developer job it supports?
- [ ] Does it help detect when the job fails?
- [ ] Does it help diagnose why the job failed?
- [ ] Does it help measure impact of failures?
- [ ] Does it help verify fixes worked?

---

## Integration Points

- **[observability-fundamentals.md](observability-fundamentals.md)** - Telemetry types for each job
- **[crash-reporting.md](crash-reporting.md)** - Error context for diagnosis job
- **[performance.md](performance.md)** - Latency for user experience jobs
- **[alert-thresholds.md](alert-thresholds.md)** - Job-based alert configuration
