---
title: OpenTelemetry on Mobile
description: Current state of OTel for mobile, compatibility strategies, and future-proofing
token_estimate: 800
topics:
  - opentelemetry
  - vendor-agnostic
  - future-proofing
platforms: [ios, android, react-native]
vendors: [generic]
complexity: intermediate
last_updated: 2026-01-20
---

# OpenTelemetry on Mobile

OpenTelemetry is the industry standard for observability, but mobile support is **not production-ready** on all platforms. This document covers the current landscape and how to future-proof your instrumentation.

## Current State (January 2026)

| Platform | SDK | Status | Production Ready? |
|----------|-----|--------|-------------------|
| iOS | `opentelemetry-swift` | Stable (Traces), Beta (Logs) | Yes (Traces only) |
| Android | `opentelemetry-android` | Stable | Yes |
| Kotlin Multiplatform | `opentelemetry-kotlin` | Alpha (Traces), Alpha (Logs) | No |
| React Native | None official | N/A | No |

### Why OTel Struggles on Mobile

OTel was designed for server-side workloads. Mobile has unique constraints:

| Challenge | Server Reality | Mobile Reality |
|-----------|---------------|----------------|
| Connectivity | Always on | Intermittent |
| Resources | Abundant | Battery/memory constrained |
| Lifecycle | Long-running | Background kills, suspends |
| Crashes | Process dumps | Symbolication needed |
| Sessions | Request-based | User session continuity |
| Replay | Not needed | Critical for debugging |

The existing OTel distributions don't solve all these constraints but are improving over time.

## Recommended Approach

### Use Vendor SDKs, Export to OTel Backends

```
Mobile App
    │
    ├── Sentry/Datadog/Embrace SDK (production-ready)
    │
    └── Vendor Backend
            │
            └── Export to OTel Collector (optional)
                    │
                    └── Your OTel-compatible backend
```

**Vendors with OTel export:**
- Sentry: Can forward to OTel collectors
- Datadog: OTel ingestion supported
- Embrace: OTel export in roadmap

### Future-Proof with OTel-Compatible Naming

Use OTel semantic conventions now. Zero runtime cost, easier migration later.

Semantic conventions are defined in the OpenTelemetry semconv specification: https://opentelemetry.io/docs/specs/semconv/

#### Span Naming

```
# OTel Semantic Conventions
http.request.method     → GET, POST
http.response.status_code → 200, 404
url.path                → /api/users
network.transport       → tcp, udp

# Mobile-specific (proposed)
app.lifecycle.state     → foreground, background
screen.name             → HomeScreen
user.interaction.type   → tap, swipe, scroll
```

#### Recommended Span Names

| Operation | OTel-Compatible Name |
|-----------|---------------------|
| HTTP request | `http.client.request` |
| Database query | `db.query` |
| Screen load | `ui.screen.load` |
| User interaction | `ui.interaction` |
| App start | `app.start.cold` / `app.start.warm` |
| Navigation | `ui.navigation` |

#### Attribute Naming

```typescript
// Good: OTel-compatible attributes
span.setAttributes({
  'http.request.method': 'POST',
  'http.response.status_code': 200,
  'url.path': '/api/appointments',
  'http.response.body.size': 4523,
});

// Avoid: Vendor-specific naming
span.setAttributes({
  'sentry.op': 'http',  // Vendor lock-in
  'method': 'POST',     // Non-standard
});
```

### Resource Attributes

Set these at SDK initialization for correlation:

```typescript
// OTel-compatible resource attributes
const resourceAttributes = {
  'service.name': 'cannon-hvac-mobile',
  'service.version': '1.0.0',
  'deployment.environment': 'production',

  // Mobile-specific
  'device.id': deviceId,
  'os.type': Platform.OS,
  'os.version': Platform.Version,
  'app.version': appVersion,
};
```

## When to Consider Native OTel

Wait for these signals before adopting OTel SDKs directly:

- [ ] Official stable release (not alpha/beta)
- [ ] Offline queue support
- [ ] Battery-optimized batching
- [ ] React Native SDK available
- [ ] Session replay integration
- [ ] Crash symbolication pipeline

**Estimated timeline:** Full mobile parity 6-12 months

## Migration Path

When OTel mobile is ready:

1. **Phase 1**: Add OTel SDK alongside vendor SDK
2. **Phase 2**: Dual-write to both
3. **Phase 3**: Validate data parity
4. **Phase 4**: Deprecate vendor SDK (optional)

If you used OTel-compatible naming from the start, Phase 1-3 are significantly easier.

## References

- [OTel Swift SDK](https://github.com/open-telemetry/opentelemetry-swift)
- [OTel Android SDK](https://github.com/open-telemetry/opentelemetry-android)
- [OTel Semantic Conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/)
- [Mobile Semantic Conventions (Draft)](https://github.com/open-telemetry/semantic-conventions/issues/mobile)
