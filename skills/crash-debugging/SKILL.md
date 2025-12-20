---
name: crash-debugging
description: Analyze mobile crashes, ANRs, and fatal errors. Use when debugging crash logs, stack traces, or "why is my app crashing?"
---

# Crash Debugging

Diagnose crashes across iOS, Android, and React Native.

## Quick Identification

| Signal | Platform | Likely Cause |
|--------|----------|--------------|
| `EXC_BAD_ACCESS` | iOS | Null pointer, dangling reference |
| `SIGABRT` | iOS | Force unwrap, assertion |
| `SIGKILL` | iOS | Watchdog, memory pressure |
| `NullPointerException` | Android | Missing null check |
| `ANR` | Android | Main thread blocked >5s |
| `OutOfMemoryError` | Android | Memory leak |

## Analysis Steps

1. **Check symbolication** - addresses without symbols = need dSYM/ProGuard
2. **Find crash frame** - top of stack trace
3. **Find app code** - first frame in your code (not system)
4. **Check thread** - main thread = UI issue

## Common Fixes

**iOS force unwrap:**
```swift
// Bad: let x = optional!
// Good: guard let x = optional else { return }
```

**Android lifecycle:**
```kotlin
// Bad: accessing view after onDestroyView
// Good: use viewLifecycleOwner.lifecycleScope
```

## Implementation

See `references/crash-reporting.md` for:
- Crash type deep dives
- Symbolication setup
- Breadcrumb strategies

See `references/{platform}-native.md` for platform-specific patterns.
