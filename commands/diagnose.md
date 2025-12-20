# Diagnose Command

Analyze crashes, ANRs, and performance issues to identify root causes.

## Usage

```
/diagnose [type] [input]
```

**Types:**
- `crash` - Crash logs and stack traces
- `performance` - Performance issues and slowdowns
- `anr` - Application Not Responding / hangs
- `oom` - Out of memory issues
- `auto` - Auto-detect from input (default)

**Input:**
- Paste crash log/stack trace directly
- Provide file path: `./crash-2024-01-15.txt`
- Describe the issue for performance problems

**Examples:**
```
/diagnose crash
# Then paste the crash log

/diagnose ./crash-log.txt

/diagnose performance
# Then describe: "Screen X takes 5 seconds to load"

/diagnose anr
# Then paste ANR trace
```

---

## Workflow

### Step 1: Detect Issue Type

Auto-detect from input:

| Indicator | Type |
|-----------|------|
| `EXC_`, `SIGABRT`, `SIGSEGV`, exception | Crash |
| `ANR`, "not responding", blocked main thread | ANR |
| `OutOfMemory`, memory pressure | OOM |
| slow, latency, duration, performance, lag | Performance |

### Step 2: Launch Issue Analyzer

Invoke the `issue-analyzer` agent with:
- The crash log/stack trace/description
- Detected issue type
- Platform (if identifiable)

### Step 3: Load Reference Context

Based on issue type:

| Issue Type | References |
|------------|------------|
| Crash | `crash-reporting.md`, `{platform}-native.md` |
| ANR | `crash-reporting.md` (ANR section), `mobile-challenges.md` |
| OOM | `mobile-challenges.md`, `crash-reporting.md` |
| Performance | `performance.md`, `ui-performance.md` |

### Step 4: Apply Relevant Skill

| Issue Type | Skill |
|------------|-------|
| Crash/ANR/OOM | `crash-debugging` |
| Performance | `performance-optimization` |

---

## Crash Analysis Flow

### Parse Crash Log

Extract:
- Crash type/signal
- Exception message
- Thread information
- Stack trace frames
- Device/OS info
- App version

### Check Symbolication

**iOS:**
- If addresses like `0x0000000102a3b4c5` without symbols → Guide to dSYM upload

**Android:**
- If obfuscated names like `a.b.c()` → Guide to ProGuard/R8 mapping upload

### Analyze Stack Trace

1. **Find crash frame** - Top of stack
2. **Find app code** - First frame in user code (not system/library)
3. **Identify thread** - Main thread = UI issue

### Common Patterns

**iOS: Force Unwrap**
```swift
// Crash: EXC_BAD_INSTRUCTION
let value = optionalValue!  // nil at runtime

// Fix
guard let value = optionalValue else { return }
```

**Android: Lifecycle Issue**
```kotlin
// Crash: NullPointerException
// Accessing view after fragment destroyed

// Fix
viewLifecycleOwner.lifecycleScope.launch { ... }
```

**ANR: Main Thread Blocking**
- Look for network calls, heavy computation, deadlocks
- Check `main` thread stack for blocking calls

---

## Performance Analysis Flow

### Identify Domain

| Symptom | Domain |
|---------|--------|
| "App takes forever to open" | App Start |
| "Screen is slow to load" | Screen Load |
| "Scrolling is janky" | Rendering |
| "API calls are slow" | Network |
| "Data loads slowly" | Database |

### Check Instrumentation

Before fixing: Is this being measured?
- If no metrics → Recommend adding spans first
- If metrics exist → Analyze p50/p95/p99

### Recommend Profiling

- iOS: Instruments (Time Profiler, Core Animation)
- Android: Perfetto, Android Studio Profiler
- RN: Flipper, Hermes profiler

---

## Output Format

### For Crashes/ANRs

```markdown
## Crash Analysis

### Summary
- **Type:** [EXC_BAD_ACCESS / NullPointerException / etc.]
- **Severity:** [Critical/High/Medium]
- **Platform:** [iOS/Android/RN]

### Stack Trace (Key Frames)
```
[Annotated relevant frames]
```

### Root Cause
[Detailed explanation of what happened]

### Affected Code
**File:** [path:line]
```[language]
// Problematic code
```

### Recommended Fix
```[language]
// Fixed code
```

### Prevention
1. [How to prevent similar issues]
2. [Testing recommendations]
3. [Monitoring to add]
```

### For Performance Issues

```markdown
## Performance Analysis

### Summary
- **Domain:** [App Start/Screen Load/etc.]
- **Severity:** [Critical/High/Medium]
- **Current State:** [Measured values if known]

### Root Cause
[Analysis of bottleneck]

### Recommended Fixes
1. [Prioritized fixes with code]

### Verification
- [ ] How to verify fix worked
- [ ] Metrics to monitor
```

---

## Skills Used

- `crash-debugging` - Crash/ANR/OOM analysis
- `performance-optimization` - Performance issue diagnosis

## Agents Used

- `issue-analyzer` - Issue analysis

## Reference Files

- `references/crash-reporting.md`
- `references/performance.md`
- `references/ui-performance.md`
- `references/mobile-challenges.md`
- `references/{platform}-native.md`
