# User Journey Instrumentation

Patterns for tracking user flows, funnels, and conversions in mobile apps. Focus: Mobile-specific patterns that differentiate from web analytics.

## Table of Contents
1. [Journey Fundamentals](#journey-fundamentals)
2. [Onboarding Funnels](#onboarding-funnels)
3. [Conversion Tracking](#conversion-tracking)
4. [Mobile Entry Funnels](#mobile-entry-funnels)
5. [Error Recovery Funnels](#error-recovery-funnels)
6. [Session Continuity](#session-continuity)

---

## Journey Fundamentals

### Definitions

| Term | Scope | Duration | Example |
|------|-------|----------|---------|
| **Session** | Single app foreground period | Minutes | User opens app, browses, closes |
| **Journey** | Goal-oriented flow across sessions | Hours/days | User researches → compares → purchases |
| **Funnel** | Ordered steps toward conversion | Variable | Onboarding: Welcome → Profile → Permissions |

### Journey Identifiers

Every journey needs a **correlation ID** that persists across:
- Screen transitions within a session
- App backgrounding/foregrounding
- Multiple sessions (for long journeys)

```
journey_id: "checkout_abc123"
├── session_1: browse → add_to_cart
├── [app closed - 2 hours]
└── session_2: cart → checkout → purchase
```

### Timing Signals

Track these for every journey step:

| Signal | Purpose |
|--------|---------|
| `step_started_at` | When user entered this step |
| `step_completed_at` | When user completed/exited |
| `time_in_step_ms` | Duration (null if abandoned) |
| `is_final_step` | Whether journey completed |

---

## Onboarding Funnels

### Standard Mobile Onboarding Flow

```
Welcome → Profile Setup → Permissions → Activation
   │           │              │            │
   └─ skip?    └─ partial?    └─ denied?   └─ success metric
```

### Key Metrics

| Metric | Formula | Target |
|--------|---------|--------|
| **Completion Rate** | Activated / Started | >60% |
| **Drop-off Point** | Step with highest exit | Identify & fix |
| **Time to Value** | First meaningful action | <2 minutes |

### Permission Branching

Mobile-specific: Users can grant, deny, or defer permissions.

```
Permissions Step
├── notification_prompt_shown
│   ├── granted → track "notifications_enabled"
│   ├── denied → track "notifications_denied" + reason
│   └── not_determined → deferred (ask later)
├── location_prompt_shown
│   ├── always → full location access
│   ├── when_in_use → limited access
│   └── denied → degraded experience path
└── tracking_prompt_shown (iOS ATT)
    ├── authorized → full attribution
    └── denied → limited attribution path
```

### Onboarding Skip Detection

Track when users bypass onboarding:
- **Explicit skip**: User taps "Skip" button
- **Implicit skip**: User navigates away before completion
- **Partial completion**: Some steps done, others skipped

---

## Conversion Tracking

### E-commerce Patterns (Brief)

Standard checkout funnel—well-documented elsewhere:

```
Cart → Shipping → Payment → Confirmation
```

Key mobile additions:
- **Cart persistence**: Track cart state across app kills
- **Payment method**: Apple Pay / Google Pay vs. manual entry
- **Guest vs. authenticated**: Different conversion rates

### Subscription Conversion

Mobile-specific subscription patterns:

```
Free Trial Start
├── day_1: activation events
├── day_3: engagement check
├── day_7: paywall shown (if not converted)
├── trial_end - 1: reminder notification
└── trial_end: convert or churn
```

Track: Trial-to-paid rate, time-to-convert, cancellation timing.

### Abandonment Signals

| Signal | Meaning | Action |
|--------|---------|--------|
| `cart_abandoned` | Items in cart, no checkout start | Re-engagement push |
| `checkout_abandoned` | Started checkout, didn't complete | Recovery email |
| `payment_failed` | Technical failure | Error recovery flow |
| `user_cancelled` | Explicit cancellation | Exit survey |

---

## Mobile Entry Funnels

### Deep Link Attribution

Track the full deep link journey:

```
Link Click (external)
    │
    ├── App Installed?
    │   ├── Yes → App Opens → Navigate to Content → Action
    │   └── No → Store → Install → First Open → Deferred Deep Link → Content
    │
    └── Attribution Window: 24-48 hours for deferred links
```

#### Key Events

| Event | Attributes |
|-------|------------|
| `deep_link_received` | `url`, `source`, `campaign`, `is_deferred` |
| `deep_link_handled` | `destination_screen`, `load_time_ms` |
| `deep_link_action_completed` | `action_type`, `time_to_action_ms` |
| `deep_link_attribution_expired` | `original_url`, `days_since_click` |

#### Example Event Schema

```json
{
  "event": "deep_link_handled",
  "journey_id": "dl_abc123",
  "attributes": {
    "original_url": "https://app.example.com/product/123",
    "source": "email_campaign",
    "campaign_id": "summer_sale_2024",
    "is_deferred": false,
    "destination_screen": "ProductDetail",
    "destination_id": "123",
    "load_time_ms": 450,
    "was_app_cold_start": true,
    "link_type": "universal_link"
  }
}
```

#### Universal Links vs. App Links

| Platform | Mechanism | Fallback |
|----------|-----------|----------|
| iOS | Universal Links (AASA file) | Safari → App Store |
| Android | App Links (assetlinks.json) | Chrome → Play Store |

Track: Link type, fallback triggered, time-to-content.

#### Deferred Deep Link Challenges

When user installs app after clicking link:

| Challenge | Solution |
|-----------|----------|
| Attribution expires | Store click data server-side, match on install |
| User clears clipboard | Use fingerprinting as fallback (less reliable) |
| Multiple clicks before install | Attribute to last click within window |
| Install from different source | Track `install_source` separately from `deep_link_source` |

### Push Re-engagement

Push notification → app action funnel:

```
Notification Delivered
    │
    ├── notification_displayed (may not fire on Android)
    ├── notification_opened → app_foregrounded → intended_action
    ├── notification_dismissed
    └── notification_expired (TTL exceeded)
```

#### Silent Push → Deferred Action

```
Silent Push Received (background)
    │
    ├── background_refresh_triggered
    ├── content_prefetched
    └── [User opens app later]
        └── Track: time_since_push, was_content_ready
```

#### Rich Notification Interactions

iOS/Android support interactive notifications:

| Interaction | Example |
|-------------|---------|
| `action_button_tapped` | "Buy Now", "Reply" |
| `inline_reply_sent` | Text input in notification |
| `media_expanded` | Image/video preview viewed |

---

## Error Recovery Funnels

### Error → Retry → Success Pattern

```
Action Attempted
    │
    ├── Success → normal flow
    └── Error
        ├── error_displayed (user sees error)
        ├── retry_attempted (automatic or manual)
        │   ├── retry_count: 1, 2, 3...
        │   ├── retry_succeeded → track recovery_time
        │   └── retry_failed → escalate or abandon
        └── user_abandoned_after_error
```

### Key Metrics

| Metric | Purpose |
|--------|---------|
| `error_to_recovery_time` | How long to fix the issue |
| `retry_success_rate` | % of retries that succeed |
| `abandonment_after_error` | Users who give up |
| `error_type_distribution` | Network vs. auth vs. validation |

### Retry Context

Track enough context to debug:

```json
{
  "event": "error_recovery_attempt",
  "journey_id": "checkout_abc123",
  "attributes": {
    "original_error_code": "NETWORK_TIMEOUT",
    "original_error_message": "Request timed out after 30s",
    "retry_number": 2,
    "retry_strategy": "exponential_backoff",
    "time_since_first_error_ms": 5000,
    "network_type_at_retry": "wifi",
    "network_quality": "good",
    "battery_level": 45,
    "is_low_power_mode": false,
    "outcome": "success",
    "recovery_time_ms": 1200
  }
}
```

### Abandonment Attribution

When users give up, track why:

| Abandonment Type | Signal | Detection Method |
|------------------|--------|------------------|
| **Rage quit** | Multiple rapid taps → back/close | Gesture velocity + tap count |
| **Timeout** | No action for N seconds after error | Inactivity timer |
| **Alternative path** | User tried different flow | Navigation event after error |
| **App kill** | Force-closed app after error | No `app_backgrounded` before termination |
| **Support escalation** | User contacted support | In-app support trigger |

### Error Recovery Best Practices

| Practice | Why It Matters |
|----------|----------------|
| Track error-to-recovery time | Measures user friction, not just error rate |
| Distinguish auto vs. manual retry | Auto-retry success ≠ good UX if user waits |
| Record network state at error AND retry | Network may have changed |
| Track abandonment after successful retry | Error may have eroded trust |
| Link errors to journey context | Same error in checkout vs. browse has different impact |

---

## Session Continuity

### Background → Foreground Resumption

Mobile users frequently leave and return:

```
User in Flow (Step 3 of 5)
    │
    ├── app_backgrounded
    │   ├── time_in_background: 30s, 5m, 2h, 1d
    │   └── was_process_killed: true/false
    │
    └── app_foregrounded
        ├── state_preserved → resume from step 3
        ├── state_lost → restart from step 1 (track as "journey_reset")
        └── state_expired → show "session expired" (track as "journey_expired")
```

### State Preservation Tracking

| Event | Attributes |
|-------|------------|
| `journey_suspended` | `step`, `data_saved`, `timestamp` |
| `journey_resumed` | `step`, `time_in_background_ms`, `state_intact` |
| `journey_reset` | `reason`: memory_pressure, state_expired, user_logout |

### Memory Pressure Recovery

iOS and Android can terminate backgrounded apps:

```
App Terminated (memory pressure)
    │
    └── Next Launch
        ├── was_terminated_by_os: true
        ├── had_pending_journey: true
        ├── journey_data_recovered: true/false
        └── user_action: resumed | restarted | abandoned
```

### Cross-Session Journeys

For journeys spanning days:

```
Session 1: Research (browse products)
    │
    ├── journey_checkpoint_saved
    │
[24 hours pass]
    │
Session 2: Compare (read reviews)
    │
    ├── journey_restored (same journey_id)
    │
[48 hours pass]
    │
Session 3: Purchase (checkout)
    │
    └── journey_completed
        └── total_journey_time: 72 hours
        └── sessions_count: 3
```

### Identity Transitions

Track when anonymous users become identified:

```
anonymous_user_123
    │
    ├── browsing, adding to cart (anonymous journey)
    │
    └── user_logged_in
        ├── identity_merged: anonymous → user_456
        ├── journey_attributed: full history linked
        └── prior_sessions_count: 3
```

#### Identity Merge Event

```json
{
  "event": "identity_merged",
  "journey_id": "checkout_abc123",
  "attributes": {
    "anonymous_id": "anon_abc123",
    "identified_user_id": "user_456",
    "merge_trigger": "login",
    "anonymous_session_count": 3,
    "anonymous_event_count": 47,
    "time_as_anonymous_hours": 72,
    "had_cart_items": true,
    "cart_value": 129.99
  }
}
```

### Journey Persistence Strategies

| Strategy | Pros | Cons |
|----------|------|------|
| **Local storage** | Fast, works offline | Lost on reinstall |
| **Keychain/Keystore** | Survives reinstall | Limited size |
| **Server-side** | Most reliable | Requires network |
| **Hybrid** | Best of both | More complexity |

For critical journeys (checkout), use server-side persistence. For exploration journeys, local storage is sufficient.

---

## Quick Reference

### Journey Event Naming Convention

```
{domain}_{entity}_{action}

Examples:
- checkout_step_completed
- onboarding_permission_granted
- deeplink_content_viewed
- error_retry_attempted
- session_journey_resumed
```

### Minimum Viable Journey Tracking

For any journey, track at least:

| Event | When | Purpose |
|-------|------|---------|
| `journey_started` | First step entered | Funnel entry |
| `journey_step_completed` | Each step done | Progress tracking |
| `journey_abandoned` | Exit without completion | Drop-off detection |
| `journey_completed` | Final step done | Conversion |

---

## Integration Points

- **[observability-fundamentals.md](observability-fundamentals.md)** - Correlation IDs and tracing
- **[session-replay.md](session-replay.md)** - Visual journey debugging
- **[performance.md](performance.md)** - Step timing and latency
- **[mobile-challenges.md](mobile-challenges.md)** - Offline journey handling
