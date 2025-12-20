# Mobile Instrumentation Patterns

Platform-agnostic patterns for comprehensive mobile observability.

## Core Instrumentation Checklist

### Tier 1: Crashes & Errors (Required)

Without these, you're flying blind.

| Item | Priority | Notes |
|------|----------|-------|
| Crash reporting SDK integrated | P0 | Sentry, Datadog, Crashlytics, etc. |
| Native crash symbolication | P0 | Upload dSYMs (iOS) and mapping files (Android) |
| Unhandled exception handler | P0 | Global error boundary or equivalent |
| Unhandled promise rejection | P0 | Often missed, causes silent failures |
| Source maps uploaded (RN/hybrid) | P0 | Without these, JS stack traces are useless |
| Release/version tagging | P0 | Know which build has the bug |

### Tier 2: Context (Required)

Crashes without context are just stack traces.

| Item | Priority | Notes |
|------|----------|-------|
| User ID (hashed/internal) | P0 | Never use email as identifier |
| Session ID | P0 | Correlate events within single session |
| App version + build number | P0 | `1.2.3 (456)` format |
| OS version | P0 | Usually automatic |
| Device model | P1 | Identify device-specific bugs |
| Breadcrumbs enabled | P1 | Last N actions before crash |

### Tier 3: Performance (Recommended)

Performance issues are bugs users feel but don't report.

| Item | Priority | Notes |
|------|----------|-------|
| App start time (cold/warm) | P1 | <2s cold, <1s warm target |
| Screen load/TTI | P1 | When screen is interactive, not just rendered |
| Network request duration | P1 | Include status codes |
| Database/storage operations | P2 | Identify slow queries |
| Frame rate/jank detection | P2 | Dropped frames indicate UI issues |

### Tier 4: Breadcrumbs (Recommended)

The story of what happened before the crash.

| Item | Priority | Notes |
|------|----------|-------|
| Navigation events | P1 | Which screens user visited |
| User interactions | P1 | Button taps, form submissions |
| Network requests (sanitized) | P1 | URLs without tokens/PII |
| App lifecycle events | P2 | Background/foreground, terminate |
| Significant state changes | P2 | Login, purchase, major actions |

### Tier 5: Business Metrics (Optional)

Observability tied to business outcomes.

| Item | Priority | Notes |
|------|----------|-------|
| Funnel stage tracking | P2 | Where users drop off |
| Feature usage | P3 | Which features are used/broken |
| Revenue-impacting flows | P2 | Payment, subscription, checkout |

## Implementation Order

Always implement in this order to maximize value earliest:

```
1. Crash Reporting    → Immediate: Know when app is broken
2. Error Tracking     → Day 1: Catch exceptions before they crash
3. User Context       → Day 1: Know WHO is affected
4. Release Tracking   → Day 1: Know WHICH VERSION is broken
5. Breadcrumbs        → Week 1: Know WHAT user was doing
6. Performance Spans  → Week 2: Know WHY it's slow
7. Custom Events      → Month 1: Know business impact
```

## Error Classification

### Crash vs Error vs Warning

| Type | Definition | Example | Action |
|------|------------|---------|--------|
| Crash | App terminates | Null pointer, memory corruption | P0 - Fix immediately |
| Error | Handled failure, degraded UX | Network timeout caught, retry failed | P1 - Fix soon |
| Warning | Potential issue, no user impact | Deprecated API, slow query | P2 - Track, batch fix |
| Info | Operational insight | User completed checkout | No action |

### Error Fingerprinting

Good fingerprinting groups same root cause together:

```
GOOD: "PaymentError: Card declined" → 1 issue, 500 events
BAD:  500 separate issues for same error with different user IDs
```

**Fingerprint by:**
- Error type + message (without dynamic data)
- Stack trace (top 3-5 frames)
- Screen/component where it occurred

**Strip from fingerprints:**
- User IDs, timestamps, request IDs
- Dynamic URLs, tokens
- Random/incremented values

## User Context Best Practices

### What to Set

```
User ID       → Required, internal ID only (not email)
Session ID    → Required, regenerate on app start
Username      → Optional, if not PII-sensitive
Subscription  → Useful for priority (paid vs free)
Cohort        → Useful for A/B test debugging
```

### What to NEVER Set

```
Email         → PII, GDPR/CCPA concerns
Phone         → PII
Full name     → PII
IP Address    → Disable auto-collection
Location      → Unless essential for debugging
Payment info  → Never, ever
```

### When to Update

```
App start     → Set session ID
Login         → Set user ID + attributes
Logout        → Clear user, keep session
Account update → Refresh attributes
```

## Breadcrumb Strategy

### Breadcrumb Budget

Most SDKs keep 100-200 breadcrumbs. This fills up fast in active apps.

**High-value (always keep):**
- Navigation to new screen
- User taps on primary actions
- Network request completion (with status)
- State machine transitions
- Errors (even handled ones)

**Low-value (avoid):**
- Every keystroke
- Scroll events
- Animation frames
- Verbose debug logs
- Timer fires

### Breadcrumb Structure

```
Category:   Short classifier (nav, http, user, state)
Message:    Human-readable action ("Tapped Checkout button")
Level:      info, warning, error
Data:       Structured metadata (max 1KB)
Timestamp:  Automatic
```

**Good breadcrumbs tell a story:**
```
[info] nav: Opened app (cold start)
[info] http: GET /api/user → 200
[info] nav: Navigated to HomeScreen
[info] user: Tapped "View Cart"
[info] nav: Navigated to CartScreen
[info] http: GET /api/cart → 200
[info] user: Tapped "Checkout"
[error] http: POST /api/checkout → 500 ← problem here
[error] crash: Unhandled NetworkError
```

## Performance Spans

### Span Naming Conventions

Use consistent naming for queryability:

```
Transactions (top-level):
  app.start.cold
  app.start.warm
  screen.load.HomeScreen
  screen.load.ProfileScreen

Operations (child spans):
  http.client.GET./api/users
  db.query.SELECT.users
  ui.render.UserList
  file.read.cache
  auth.refresh_token
```

### Span Attributes

Include attributes that help debugging:

```
HTTP spans:
  http.method: GET
  http.status_code: 200
  http.response_size: 4523
  http.url: /api/users (sanitized)

DB spans:
  db.system: sqlite
  db.operation: SELECT
  db.rows_affected: 42

UI spans:
  ui.component: UserList
  ui.item_count: 25
```

## Mobile-Specific Instrumentation

### App Lifecycle

Track these state transitions:

```
Cold Start       → App launched from terminated state
Warm Start       → App launched from background
Background       → App moved to background
Foreground       → App returned to foreground
Terminate        → App being killed (if detectable)
Memory Warning   → iOS memory pressure notification
Low Battery      → Optional, correlate with issues
```

### Network Conditions

Mobile networks are unreliable:

```
network.type:       wifi, cellular, none
network.effective:  4g, 3g, 2g, slow-2g
network.connected:  true, false
```

### Device Context

Usually auto-collected, but verify:

```
device.model:       iPhone 12, Pixel 7
device.os:          iOS 17.1, Android 14
device.memory:      Low memory devices behave differently
device.battery:     Optional, correlate with performance
device.orientation: Portrait, landscape
device.locale:      en-US, ja-JP
```

## Sampling Strategies

### Error Sampling

**Always 100%** for errors. Every error matters.

```javascript
sampleRate: 1.0  // 100% of errors
```

### Performance Sampling

Scale based on volume:

| Daily Active Users | Trace Sample Rate |
|--------------------|-------------------|
| <10,000           | 100% (1.0)        |
| 10,000-100,000    | 20-50% (0.2-0.5)  |
| 100,000-1,000,000 | 5-20% (0.05-0.2)  |
| >1,000,000        | 1-5% (0.01-0.05)  |

### Dynamic Sampling

Increase sampling for important transactions:

```javascript
tracesSampler: ({ name, parentSampled }) => {
  // Always trace payment flows
  if (name.includes('checkout') || name.includes('payment')) {
    return 1.0;
  }
  // Higher rate for errors
  if (name.includes('error')) {
    return 1.0;
  }
  // Default rate
  return 0.2;
}
```

## Anti-Patterns

### 1. Logging Everything

```
BAD:  Log every function call, state change, render
GOOD: Log meaningful state transitions and user actions
```

### 2. Sensitive Data in Events

```
BAD:  { email: "user@example.com", password: "...", token: "abc123" }
GOOD: { userId: "u_abc123", hasToken: true }
```

### 3. High-Cardinality Tags

```
BAD:  tag("requestId", uuid())           // Millions of unique values
BAD:  tag("timestamp", Date.now())       // Infinite cardinality
GOOD: tag("region", "us-west")           // Bounded set
GOOD: tag("subscriptionTier", "premium") // Small set
```

### 4. Giant Payloads

```
BAD:  Attach entire Redux state (100KB+) to every event
GOOD: Attach relevant slice (1-5KB max)
```

### 5. Synchronous Heavy Lifting

```
BAD:  Block main thread to send analytics
GOOD: Queue events, send in background/batches
```

## Testing Checklist

Before shipping:

- [ ] Crash reporting captures test crash
- [ ] Stack traces are symbolicated (not memory addresses or bytecode)
- [ ] Release version appears correctly in dashboard
- [ ] User ID propagates to events
- [ ] Breadcrumbs appear in crash reports
- [ ] Performance spans show in traces
- [ ] Offline errors queue and send when online
- [ ] No PII appears in any event
