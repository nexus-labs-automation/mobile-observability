---
title: Feature Flag Instrumentation
description: How to instrument feature flag evaluations and assignments for observability
token_estimate: 400
topics:
  - feature-flags
  - experiments
  - instrumentation
platforms: [ios, android, react-native]
vendors: [generic]
complexity: intermediate
last_updated: 2025-12-20
---

# Feature Flag Instrumentation

How to make feature flags observable without bloating every event payload.

## The Problem

You can't attach all active flags to every event:
- Payloads explode (1000 flags × thousands of events)
- Most flags are irrelevant to most events
- Cost and performance suffer

But you need to answer: "Did flag X cause this regression?"

## What to Instrument

### 1. Flag Evaluation Events

Log when a flag is **evaluated**, not just assigned:

```swift
func evaluateFlag(_ flagKey: String, context: EvaluationContext) -> FlagValue {
    let value = flagService.evaluate(flagKey, context: context)

    Observability.trackEvent("feature_flag.evaluated", properties: [
        "flag_key": flagKey,
        "value": String(describing: value),
        "evaluation_reason": value.reason,  // "targeting_match", "default", "error"
        "context_hash": context.stableHash   // For debugging, not full context
    ])

    return value
}
```

**Why evaluation, not just assignment?**
- Captures *when* the flag was checked
- Captures the *reason* (targeting rule, default, error)
- Creates timeline for correlation

### 2. Flag Change Events

Log when a flag value **changes** for a user:

```swift
class FlagObserver {
    private var previousValues: [String: FlagValue] = [:]

    func onFlagUpdated(_ flagKey: String, newValue: FlagValue) {
        let previousValue = previousValues[flagKey]

        if previousValue != newValue {
            Observability.trackEvent("feature_flag.changed", properties: [
                "flag_key": flagKey,
                "previous_value": previousValue.map { String(describing: $0) },
                "new_value": String(describing: newValue),
                "change_source": "realtime_update"  // or "app_start", "refresh"
            ])

            previousValues[flagKey] = newValue
        }
    }
}
```

### 3. Session Flag Snapshot (Optional)

Capture active flags once at session start for debugging:

```swift
func onSessionStart() {
    // Only capture flags relevant to core flows, not all 1000
    let criticalFlags = ["checkout_v2", "new_onboarding", "payment_provider"]

    let snapshot = criticalFlags.reduce(into: [String: String]()) { result, key in
        result[key] = String(describing: flagService.getValue(key))
    }

    Observability.setContext("flags", snapshot)
}
```

**Note:** This attaches to the session, not every event.

---

## What NOT to Do

| Anti-Pattern | Why It's Bad |
|--------------|--------------|
| Attach all flags to every event | Payload bloat, cost explosion |
| Log flag values without flag key | Can't query by specific flag |
| Skip evaluation reason | Can't debug targeting issues |
| Log on every render/check | Noise; log on evaluation or change only |
| Include user attributes in flag events | PII risk; use hashed context |

---

## Connecting Flags to Errors

When capturing errors, include **relevant** flags only:

```swift
func captureError(_ error: Error, feature: String) {
    // Only include flags that affect this specific feature
    let relevantFlags = flagRegistry.flagsForFeature(feature)

    Observability.captureError(error, context: [
        "feature": feature,
        "flags": relevantFlags.map { [$0: flagService.getValue($0)] }
    ])
}
```

This keeps payloads small while enabling correlation.

---

## Event Reference

| Event | When | Key Attributes |
|-------|------|----------------|
| `feature_flag.evaluated` | Flag checked | `flag_key`, `value`, `evaluation_reason` |
| `feature_flag.changed` | Value changed for user | `flag_key`, `previous_value`, `new_value` |
| `feature_flag.error` | Evaluation failed | `flag_key`, `error_type`, `fallback_used` |

---

## Correlation Happens Elsewhere

This plugin covers **instrumentation** — logging the right events with the right context.

**Correlation** (joining flag assignments with crash rates, regressions, etc.) happens in:
- Your observability vendor's dashboard
- Your data warehouse
- Your feature flag platform's analytics

The instrumentation patterns here give those tools the data they need.
