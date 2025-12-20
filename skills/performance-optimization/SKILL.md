---
name: performance-optimization
description: Diagnose mobile performance issues. Use when app is slow, laggy, or users complain about performance.
---

# Performance Optimization

Identify and fix performance bottlenecks.

## Symptom → Domain

| Symptom | Domain | Reference |
|---------|--------|-----------|
| "App takes forever to open" | App Start | `performance.md` |
| "Screen loads slowly" | Screen Load | `performance.md` |
| "Scrolling is janky" | Rendering | `ui-performance.md` |
| "API calls are slow" | Network | `performance.md` |
| "Data loads slowly" | Database | `data-persistence.md` |

## Key Thresholds

| Metric | Good | Acceptable | Poor |
|--------|------|------------|------|
| Cold start | <2s | <4s | >4s |
| Warm start | <1s | <2s | >2s |
| Screen TTI | <1s | <2s | >2s |
| API p95 | <2s | <5s | >5s |
| Frame rate | 60fps | 30fps | <30fps |

## Diagnosis Steps

1. **Identify domain** from symptom
2. **Check instrumentation** - is it being measured?
3. **Analyze metrics** - p50/p95/p99 values
4. **Profile** - Instruments (iOS), Perfetto (Android)

## Quick Wins

- Move work off main thread
- Add loading states
- Cache network responses
- Lazy load images
- Reduce view hierarchy depth

## Implementation

See `references/performance.md` for app start, screen load, network.
See `references/ui-performance.md` for rendering, scroll, animations.
See `references/data-persistence.md` for database optimization.
