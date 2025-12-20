# Mobile Alert Thresholds & SLOs

Mobile-specific service level objectives and alerting best practices.

## Core Mobile SLIs

### Crash-Free Rate

The primary mobile stability metric.

| Metric | Definition |
|--------|------------|
| Crash-Free Sessions | % of sessions with zero crashes |
| Crash-Free Users | % of users who experienced zero crashes (7-day) |

**Industry Benchmarks:**

| Tier | Crash-Free Sessions | Crash-Free Users |
|------|---------------------|------------------|
| World-class | >99.9% | >99.5% |
| Good | >99.5% | >99.0% |
| Acceptable | >99.0% | >98.0% |
| Needs work | <99.0% | <98.0% |

**Alert Thresholds:**

```yaml
# P0 - Page on-call immediately
crash_free_sessions:
  critical: < 99.0%  # Something is very wrong
  window: 1 hour

# P1 - Investigate same day
crash_free_sessions:
  warning: < 99.5%
  window: 24 hours

# Regression alert - new version issue
crash_rate_by_release:
  critical: > 2x previous release
  window: 4 hours after release
```

### Error Rate

Errors that don't crash but degrade experience.

**Target:** <1% of sessions should encounter an error.

```yaml
error_rate:
  warning: > 1%
  critical: > 3%
  window: 1 hour

# Per-screen error monitoring
screen_error_rate:
  warning: > 2%
  critical: > 5%
  window: 1 hour
  groupBy: screen_name
```

### App Start Time

Time from tap-to-interactive.

**Benchmarks:**

| Start Type | Target | Acceptable | Poor |
|------------|--------|------------|------|
| Cold Start | <2s | <4s | >4s |
| Warm Start | <1s | <2s | >2s |
| Hot Start | <500ms | <1s | >1s |

```yaml
app_start_cold:
  p50_warning: > 2s
  p95_warning: > 4s
  p99_critical: > 6s
  window: 1 hour

app_start_warm:
  p50_warning: > 1s
  p95_warning: > 2s
  window: 1 hour
```

### Screen Load Time (TTI)

Time to interactive for each screen.

**Target:** <1s for all screens, <300ms for navigation transitions.

```yaml
screen_load:
  p50_warning: > 1s
  p95_warning: > 2s
  p99_critical: > 4s
  window: 1 hour
  groupBy: screen_name

# Critical screens get tighter thresholds
screen_load_checkout:
  p50_warning: > 800ms
  p95_critical: > 2s
```

### Network Request Performance

API call latency and reliability.

```yaml
api_latency:
  p50_warning: > 500ms
  p95_warning: > 2s
  p99_critical: > 5s
  window: 15 minutes

api_error_rate:
  warning: > 1%
  critical: > 5%
  window: 15 minutes

# By endpoint priority
api_latency_payments:
  p95_critical: > 3s
```

### Frame Rate / UI Jank

Smooth UI is <60fps with no dropped frames.

```yaml
# Frozen frames: >700ms with no render
frozen_frame_rate:
  warning: > 1%
  critical: > 3%

# Slow frames: >16ms render time
slow_frame_rate:
  warning: > 10%
  critical: > 20%
```

## Alert Configuration Patterns

### Tiered Alerting

Don't alert on everything equally.

```yaml
# Tier 1: Page immediately (P0)
tier1_alerts:
  - crash_rate > 1% in 30min
  - crash_free_users < 98% in 1hr
  - payment_flow_error_rate > 2%
  - app_completely_unusable

# Tier 2: Urgent but not paging (P1)
tier2_alerts:
  - crash_rate > 0.5% in 1hr
  - error_rate > 3% in 1hr
  - p95_latency > 5s
  - login_error_rate > 5%

# Tier 3: Investigate during business hours (P2)
tier3_alerts:
  - crash_rate regression vs previous release
  - new error type affecting >100 users
  - performance regression >20%
```

### Release-Based Alerts

Catch regressions early.

```yaml
# Compare to baseline (previous release)
release_regression:
  trigger: 4 hours after release detected
  conditions:
    - crash_rate > 1.5x baseline
    - error_rate > 2x baseline
    - p95_latency > 1.3x baseline
  action: Notify release owner

# Automatic rollback trigger
critical_regression:
  trigger: 1 hour after release
  conditions:
    - crash_rate > 3x baseline
    - affecting > 5% of users
  action: Consider rollback
```

### Device/OS Segmented Alerts

Mobile fragmentation requires segmented monitoring.

```yaml
# iOS vs Android separate thresholds
ios_crash_rate:
  warning: > 0.3%
  baseline: lower due to platform stability

android_crash_rate:
  warning: > 0.5%
  baseline: slightly higher acceptable

# OS version degradation
os_version_crash_rate:
  trigger: any OS version > 3x overall average
  action: Investigate compatibility issue

# Device-specific
device_crash_rate:
  trigger: any device model > 5x average
  action: Device-specific bug
```

### User Impact Alerts

Not just rate, but reach.

```yaml
# Absolute numbers matter
users_affected:
  warning: > 1000 users hit same error in 1hr
  critical: > 10000 users hit same error in 1hr

# VIP/paying user priority
premium_user_errors:
  threshold: 50% lower than free users
  rationale: Revenue-impacting

# Geographic spread
error_concentration:
  trigger: > 80% of errors from single region
  indicates: Possible CDN/regional issue
```

## Dashboard Panels

### Real-Time Operations Dashboard

```
Row 1: Overall Health
├── Crash-Free Session Rate (current vs 24hr ago)
├── Active Users (current session count)
├── Error Rate (last hour)
└── App Start P95 (last hour)

Row 2: Release Health
├── Crash Rate by Version (line chart)
├── New Errors (appeared this release)
├── Adoption Curve (% users on latest)
└── Release Comparison Table

Row 3: Performance
├── App Start Time (cold/warm breakdown)
├── Screen Load Times (by screen)
├── API Latency (P50/P95/P99)
└── Network Error Rate

Row 4: User Impact
├── Users Affected by Errors (unique count)
├── Top Errors by User Impact
├── Error Rate by OS Version
└── Geographic Error Distribution
```

### Weekly Review Dashboard

```
Metrics for week-over-week comparison:

Stability:
├── Crash-Free User Rate (trend)
├── Total Crashes (absolute)
├── New vs Known Errors
└── Mean Time to Resolution

Performance:
├── App Start Time (trend)
├── Key Screen Load Times (trend)
├── API Latency P95 (trend)
└── Slow Frame Rate (trend)

User Impact:
├── Users Affected by Issues
├── Support Tickets from Errors
├── Store Rating Impact
└── Churn Correlation
```

## Alert Fatigue Prevention

### Strategies

1. **Aggregate similar alerts**
   - Group same error type across devices
   - Suppress duplicate alerts within 1 hour

2. **Require minimum impact**
   - Don't alert on 1 crash
   - Require: affecting >N users OR >X% rate

3. **Time-based suppression**
   - Known issues: snooze for fix duration
   - After hours: escalate only P0

4. **Auto-resolve**
   - Alert auto-closes when metric recovers
   - Reduces noise from transient issues

### Alert Quality Metrics

Track alert effectiveness:

| Metric | Target |
|--------|--------|
| Alert-to-incident ratio | >50% of alerts are real issues |
| Time to acknowledge | <15 min for P0 |
| False positive rate | <20% |
| Alert noise per week | <20 alerts per engineer |

## SLO Budgets

### Error Budget Concept

If your SLO is 99.5% crash-free:
- Monthly budget: 0.5% of sessions can crash
- If you have 1M sessions/month → 5,000 crashes allowed

```yaml
error_budget:
  slo: 99.5%
  monthly_sessions: 1000000
  budget: 5000 crashes

  # Alert when burning through budget too fast
  burn_rate_alert:
    warning: > 2x normal burn rate over 6hr
    critical: > 10x normal burn rate over 1hr
```

### Budget-Based Release Decisions

```
Budget remaining > 80%  → Ship freely
Budget remaining 50-80% → Ship with caution
Budget remaining 20-50% → Only critical fixes
Budget remaining < 20%  → Freeze, focus on stability
```

## Platform-Specific Thresholds

### iOS

Generally more stable platform:

```yaml
ios:
  crash_free_target: 99.7%
  cold_start_target: 1.5s
  notes: |
    - Consistent hardware makes debugging easier
    - Memory pressure handling is critical
    - Background task limits are strict
```

### Android

More variance due to fragmentation:

```yaml
android:
  crash_free_target: 99.3%
  cold_start_target: 2.5s
  notes: |
    - Wide device range increases crash variance
    - OEM modifications cause unexpected issues
    - Monitor by Android version and device tier
```

### React Native / Hybrid

Additional complexity:

```yaml
react_native:
  crash_free_target: 99.2%
  js_error_rate_target: < 0.5%
  native_crash_rate_target: < 0.2%
  notes: |
    - Separate JS errors from native crashes
    - Bridge errors are common failure mode
    - Source maps critical for debugging
```
