---
title: Business Logic Observability
description: Instrumenting decision points, rule evaluation, computed state, and background processing
token_estimate: 900
topics:
  - business-logic
  - decisions
  - state-machines
  - background-jobs
platforms: [ios, android, react-native]
vendors: [generic]
complexity: intermediate
last_updated: 2025-12-20
---

# Business Logic Observability

Instrumentation for the code users never see—but that determines whether your app works correctly.

## Why This Matters

Modern mobile apps run significant business logic client-side:
- Eligibility checks
- Calculations and aggregations
- Rule evaluation
- State machine transitions
- Background sync and processing

These can fail **silently**. The app doesn't crash—it just produces wrong results. Without proper instrumentation, you won't know until a user complains or an audit catches it.

## The Core Problem

| User-Facing Failure | System Failure |
|---------------------|----------------|
| App crashes | Wrong output, no crash |
| Button doesn't work | Calculation off by 0.01% |
| Screen won't load | Rule didn't fire when it should |
| Visible error message | Silent data corruption |

System failures are harder to detect because **the app appears to work**.

---

## Decision Point Instrumentation

### The Pattern

Every significant decision should log:
1. **Inputs** — what data went into the decision
2. **Logic path** — which branch was taken
3. **Output** — what was decided
4. **Confidence signal** — optional validation

### Example: Eligibility Check

```swift
struct EligibilityDecision {
    let feature: String
    let inputs: [String: Any]
    let result: Bool
    let reason: String
    let evaluatedRules: [String]
}

func checkEligibility(for feature: String, context: UserContext) -> EligibilityDecision {
    let inputs: [String: Any] = [
        "user_tier": context.tier.rawValue,
        "account_age_days": context.accountAgeDays,
        "region": context.region,
        "feature_flags": context.activeFlags
    ]

    let (result, reason, rules) = evaluateRules(feature: feature, context: context)

    let decision = EligibilityDecision(
        feature: feature,
        inputs: inputs,
        result: result,
        reason: reason,
        evaluatedRules: rules
    )

    // Log the decision
    Observability.trackEvent("decision.eligibility", properties: [
        "feature": feature,
        "result": result,
        "reason": reason,
        "rules_evaluated": rules.count,
        "user_tier": context.tier.rawValue,
        "region": context.region
    ])

    return decision
}
```

### What to Capture

| Field | Purpose | Example |
|-------|---------|---------|
| `decision_type` | Category of decision | "eligibility", "routing", "pricing" |
| `decision_id` | Correlation ID | UUID for tracing |
| `inputs_hash` | Fingerprint of inputs | For detecting input drift |
| `result` | The decision made | true/false, option A/B/C |
| `reason` | Why this result | "rule_x_matched", "default" |
| `duration_ms` | Evaluation time | Performance tracking |

---

## Rule Evaluation Tracing

When business rules determine behavior, trace which rules fired.

### Pattern: Rule Evaluation Log

```swift
struct RuleEvaluation {
    let ruleId: String
    let matched: Bool
    let inputs: [String: Any]
    let reason: String?
}

class RuleEngine {
    func evaluate(rules: [Rule], context: Context) -> (result: Any, trace: [RuleEvaluation]) {
        var trace: [RuleEvaluation] = []

        for rule in rules {
            let matched = rule.condition(context)
            trace.append(RuleEvaluation(
                ruleId: rule.id,
                matched: matched,
                inputs: rule.relevantInputs(from: context),
                reason: matched ? nil : rule.failureReason(context)
            ))

            if matched {
                // Log which rule won
                Observability.trackEvent("rule.matched", properties: [
                    "rule_id": rule.id,
                    "rule_priority": rule.priority,
                    "rules_evaluated": trace.count,
                    "rules_total": rules.count
                ])

                return (rule.action(context), trace)
            }
        }

        // No rule matched - log this too
        Observability.trackEvent("rule.no_match", properties: [
            "rules_evaluated": rules.count,
            "context_summary": context.summary
        ])

        return (defaultAction(context), trace)
    }
}
```

### Debugging Rule Failures

When a rule should have fired but didn't:

```swift
// Attach full trace to errors for debugging
func handleUnexpectedBehavior(expected: String, actual: String, trace: [RuleEvaluation]) {
    Observability.captureMessage("Unexpected rule outcome", level: .warning, extras: [
        "expected": expected,
        "actual": actual,
        "rule_trace": trace.map { [
            "rule": $0.ruleId,
            "matched": $0.matched,
            "reason": $0.reason ?? "matched"
        ]}
    ])
}
```

---

## Computed State Validation

For complex calculations, validate outputs make sense.

### Pattern: Sanity Checks

```swift
struct CalculationResult<T> {
    let value: T
    let inputs: [String: Any]
    let intermediates: [String: Any]
    let isValid: Bool
    let validationErrors: [String]
}

func calculateTotal(items: [LineItem], context: OrderContext) -> CalculationResult<Decimal> {
    let subtotal = items.reduce(0) { $0 + $1.price * Decimal($1.quantity) }
    let discounts = applyDiscounts(subtotal: subtotal, context: context)
    let fees = calculateFees(subtotal: subtotal - discounts, context: context)
    let total = subtotal - discounts + fees

    // Sanity checks
    var validationErrors: [String] = []

    if total < 0 {
        validationErrors.append("total_negative")
    }
    if discounts > subtotal {
        validationErrors.append("discounts_exceed_subtotal")
    }
    if total > subtotal * 2 {
        validationErrors.append("total_suspiciously_high")
    }

    let result = CalculationResult(
        value: total,
        inputs: ["item_count": items.count, "subtotal": subtotal],
        intermediates: ["discounts": discounts, "fees": fees],
        isValid: validationErrors.isEmpty,
        validationErrors: validationErrors
    )

    // Log calculation
    Observability.trackEvent("calculation.total", properties: [
        "subtotal": NSDecimalNumber(decimal: subtotal).doubleValue,
        "discounts": NSDecimalNumber(decimal: discounts).doubleValue,
        "fees": NSDecimalNumber(decimal: fees).doubleValue,
        "total": NSDecimalNumber(decimal: total).doubleValue,
        "item_count": items.count,
        "is_valid": result.isValid
    ])

    // Alert on validation failures
    if !result.isValid {
        Observability.captureMessage("Calculation validation failed", level: .error, extras: [
            "errors": validationErrors,
            "inputs": result.inputs,
            "intermediates": result.intermediates
        ])
    }

    return result
}
```

### What to Validate

| Check | Catches |
|-------|---------|
| `result >= 0` | Sign errors, underflow |
| `result <= reasonable_max` | Overflow, multiplication bugs |
| `sum(parts) == total` | Rounding errors, missing components |
| `result changed` | Stale cache, failed update |
| `result matches estimate` | Logic errors, wrong inputs |

---

## State Machine Instrumentation

Complex state requires transition logging.

### Pattern: State Transition Events

```swift
class StateMachine<State: Hashable, Event> {
    private(set) var currentState: State
    private let transitions: [State: [Event: State]]
    private let machineName: String

    func send(_ event: Event) -> Bool {
        let previousState = currentState

        guard let nextState = transitions[currentState]?[event] else {
            // Invalid transition
            Observability.trackEvent("state.transition_rejected", properties: [
                "machine": machineName,
                "from_state": String(describing: previousState),
                "event": String(describing: event),
                "reason": "no_valid_transition"
            ])
            return false
        }

        currentState = nextState

        // Log successful transition
        Observability.trackEvent("state.transition", properties: [
            "machine": machineName,
            "from_state": String(describing: previousState),
            "to_state": String(describing: nextState),
            "event": String(describing: event)
        ])

        return true
    }
}
```

### Detecting State Anomalies

```swift
// Track time in each state
class StateTimeTracker {
    private var stateEnteredAt: [String: Date] = [:]
    private let expectedDurations: [String: TimeInterval]

    func onStateEnter(_ state: String) {
        stateEnteredAt[state] = Date()
    }

    func onStateExit(_ state: String) {
        guard let entered = stateEnteredAt[state] else { return }
        let duration = Date().timeIntervalSince(entered)

        Observability.recordMetric(
            name: "state.duration",
            value: duration,
            tags: ["state": state]
        )

        // Anomaly detection
        if let expected = expectedDurations[state], duration > expected * 3 {
            Observability.trackEvent("state.stuck", properties: [
                "state": state,
                "duration_seconds": duration,
                "expected_seconds": expected
            ])
        }
    }
}
```

---

## Background Job Observability

Jobs running outside user interaction need explicit instrumentation.

### Pattern: Job Lifecycle Events

```swift
struct JobContext {
    let jobId: String
    let jobType: String
    let triggeredBy: String  // "schedule", "push", "user", "retry"
    let attempt: Int
}

class BackgroundJobRunner {
    func run<T>(job: BackgroundJob<T>, context: JobContext) async -> Result<T, Error> {
        let startTime = CFAbsoluteTimeGetCurrent()

        // Job started
        Observability.trackEvent("job.started", properties: [
            "job_id": context.jobId,
            "job_type": context.jobType,
            "triggered_by": context.triggeredBy,
            "attempt": context.attempt
        ])

        do {
            let result = try await job.execute()
            let duration = CFAbsoluteTimeGetCurrent() - startTime

            // Job succeeded
            Observability.trackEvent("job.completed", properties: [
                "job_id": context.jobId,
                "job_type": context.jobType,
                "duration_seconds": duration,
                "attempt": context.attempt
            ])

            Observability.recordMetric(
                name: "job.duration",
                value: duration,
                tags: ["job_type": context.jobType, "outcome": "success"]
            )

            return .success(result)

        } catch {
            let duration = CFAbsoluteTimeGetCurrent() - startTime

            // Job failed
            Observability.captureError(error, context: [
                "job_id": context.jobId,
                "job_type": context.jobType,
                "duration_seconds": duration,
                "attempt": context.attempt,
                "triggered_by": context.triggeredBy
            ])

            Observability.recordMetric(
                name: "job.duration",
                value: duration,
                tags: ["job_type": context.jobType, "outcome": "failure"]
            )

            return .failure(error)
        }
    }
}
```

### Key Metrics for Background Jobs

| Metric | Why It Matters |
|--------|----------------|
| `job.success_rate` | Are jobs completing? |
| `job.duration_p95` | Performance degradation |
| `job.retry_rate` | Flaky dependencies |
| `job.queue_depth` | Backlog building up |
| `job.staleness` | How old is the data being processed |

---

## Feature Flags

See [feature-flags.md](feature-flags.md) for dedicated coverage of:
- Flag evaluation events
- Flag change events
- What NOT to do (don't attach all flags to every event)

---

## Quick Reference

### Instrumentation Checklist

For any business logic:

- [ ] **Inputs logged** — what data drove the decision?
- [ ] **Output logged** — what was decided/calculated?
- [ ] **Path logged** — which rules/branches executed?
- [ ] **Validation** — sanity checks on output?
- [ ] **Timing** — how long did it take?
- [ ] **Correlation** — can you trace this to a user action?

### Event Naming Convention

```
{domain}.{entity}.{action}

Examples:
- decision.eligibility.evaluated
- rule.discount.matched
- calculation.total.validated
- state.order.transitioned
- job.sync.completed
```

### When to Add Business Logic Instrumentation

| Scenario | Instrumentation Priority |
|----------|-------------------------|
| Affects money/billing | Critical |
| User-visible output | High |
| Eligibility/access control | High |
| Complex multi-step calculation | High |
| Background sync | Medium |
| Caching decisions | Medium |
| Feature flags | Medium |

---

## Integration Points

- **[user-focused-observability.md](user-focused-observability.md)** — Link system decisions to user outcomes
- **[jtbd.md](jtbd.md)** — Frame business logic in terms of jobs
- **[performance.md](performance.md)** — Timing for computations
- **[data-persistence.md](data-persistence.md)** — State that feeds business logic
