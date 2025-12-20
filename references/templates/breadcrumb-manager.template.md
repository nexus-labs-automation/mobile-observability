# Breadcrumb Manager Template

Capture the trail of events leading to errors. Limited to 100 most recent events.

## Breadcrumb Types

| Type | Description | Auto-collected? |
|------|-------------|-----------------|
| **Navigation** | Screen transitions | Yes |
| **UI** | Button taps, gestures | Configurable |
| **Network** | HTTP requests/responses | Yes |
| **Console** | Log messages | Configurable |
| **User** | Custom user actions | Manual |
| **System** | App lifecycle, memory warnings | Yes |
| **Error** | Non-fatal errors | Yes |

---

## iOS

### BreadcrumbManager

```swift
// BreadcrumbManager.swift
import Foundation

struct Breadcrumb: Codable {
    enum BreadcrumbType: String, Codable {
        case navigation, ui, network, console, user, system, error
    }

    enum Level: String, Codable {
        case debug, info, warning, error
    }

    let timestamp: Date
    let type: BreadcrumbType
    let level: Level
    let category: String
    let message: String
    let data: [String: String]
}

final class BreadcrumbManager {
    static let shared = BreadcrumbManager()

    private var breadcrumbs: [Breadcrumb] = []
    private let queue = DispatchQueue(label: "breadcrumbs")
    private let maxBreadcrumbs = 100

    func add(
        type: Breadcrumb.BreadcrumbType,
        category: String,
        message: String,
        level: Breadcrumb.Level = .info,
        data: [String: String] = [:]
    ) {
        let breadcrumb = Breadcrumb(
            timestamp: Date(),
            type: type,
            level: level,
            category: category,
            message: message,
            data: data
        )

        queue.async {
            self.breadcrumbs.append(breadcrumb)

            if self.breadcrumbs.count > self.maxBreadcrumbs {
                self.breadcrumbs.removeFirst(self.breadcrumbs.count - self.maxBreadcrumbs)
            }
        }
    }

    func getBreadcrumbs() -> [Breadcrumb] {
        queue.sync { breadcrumbs }
    }

    func clear() {
        queue.async { self.breadcrumbs.removeAll() }
    }

    // MARK: - Convenience Methods

    func navigation(from: String, to: String) {
        add(
            type: .navigation,
            category: "navigation",
            message: "Navigated from \(from) to \(to)",
            data: ["from": from, "to": to]
        )
    }

    func buttonTap(_ buttonName: String, screen: String) {
        add(
            type: .ui,
            category: "ui.tap",
            message: "Tapped \(buttonName)",
            data: ["button": buttonName, "screen": screen]
        )
    }

    func networkRequest(method: String, url: String, statusCode: Int, duration: TimeInterval) {
        let level: Breadcrumb.Level = statusCode >= 400 ? .warning : .info
        add(
            type: .network,
            category: "http",
            message: "\(method) \(url) - \(statusCode)",
            level: level,
            data: [
                "method": method,
                "url": url,
                "status_code": String(statusCode),
                "duration_ms": String(Int(duration * 1000))
            ]
        )
    }

    func systemEvent(_ event: String, data: [String: String] = [:]) {
        add(
            type: .system,
            category: "app.lifecycle",
            message: event,
            data: data
        )
    }

    func userAction(_ action: String, data: [String: String] = [:]) {
        add(
            type: .user,
            category: "user",
            message: action,
            data: data
        )
    }
}
```

### Auto-Collection

```swift
extension BreadcrumbManager {
    func setupAutoCollection() {
        NotificationCenter.default.addObserver(
            forName: UIApplication.didBecomeActiveNotification,
            object: nil, queue: .main
        ) { [weak self] _ in
            self?.systemEvent("App became active")
        }

        NotificationCenter.default.addObserver(
            forName: UIApplication.willResignActiveNotification,
            object: nil, queue: .main
        ) { [weak self] _ in
            self?.systemEvent("App will resign active")
        }

        NotificationCenter.default.addObserver(
            forName: UIApplication.didEnterBackgroundNotification,
            object: nil, queue: .main
        ) { [weak self] _ in
            self?.systemEvent("App entered background")
        }

        NotificationCenter.default.addObserver(
            forName: UIApplication.didReceiveMemoryWarningNotification,
            object: nil, queue: .main
        ) { [weak self] _ in
            self?.add(
                type: .system,
                category: "memory",
                message: "Memory warning received",
                level: .warning
            )
        }
    }
}
```

---

## Android

### BreadcrumbManager

```kotlin
// BreadcrumbManager.kt
import java.util.concurrent.ConcurrentLinkedDeque

data class Breadcrumb(
    val timestamp: Long = System.currentTimeMillis(),
    val type: BreadcrumbType,
    val level: Level,
    val category: String,
    val message: String,
    val data: Map<String, String> = emptyMap()
) {
    enum class BreadcrumbType { NAVIGATION, UI, NETWORK, CONSOLE, USER, SYSTEM, ERROR }
    enum class Level { DEBUG, INFO, WARNING, ERROR }
}

object BreadcrumbManager {
    private val breadcrumbs = ConcurrentLinkedDeque<Breadcrumb>()
    private const val MAX_BREADCRUMBS = 100

    fun add(
        type: Breadcrumb.BreadcrumbType,
        category: String,
        message: String,
        level: Breadcrumb.Level = Breadcrumb.Level.INFO,
        data: Map<String, String> = emptyMap()
    ) {
        breadcrumbs.addLast(Breadcrumb(
            type = type,
            level = level,
            category = category,
            message = message,
            data = data
        ))

        while (breadcrumbs.size > MAX_BREADCRUMBS) {
            breadcrumbs.pollFirst()
        }
    }

    fun getBreadcrumbs(): List<Breadcrumb> = breadcrumbs.toList()

    fun clear() = breadcrumbs.clear()

    // Convenience methods
    fun navigation(from: String, to: String) {
        add(
            type = Breadcrumb.BreadcrumbType.NAVIGATION,
            category = "navigation",
            message = "Navigated from $from to $to",
            data = mapOf("from" to from, "to" to to)
        )
    }

    fun buttonTap(buttonName: String, screen: String) {
        add(
            type = Breadcrumb.BreadcrumbType.UI,
            category = "ui.tap",
            message = "Tapped $buttonName",
            data = mapOf("button" to buttonName, "screen" to screen)
        )
    }

    fun networkRequest(method: String, url: String, statusCode: Int, durationMs: Long) {
        val level = if (statusCode >= 400) Breadcrumb.Level.WARNING else Breadcrumb.Level.INFO
        add(
            type = Breadcrumb.BreadcrumbType.NETWORK,
            category = "http",
            message = "$method $url - $statusCode",
            level = level,
            data = mapOf(
                "method" to method,
                "url" to url,
                "status_code" to statusCode.toString(),
                "duration_ms" to durationMs.toString()
            )
        )
    }

    fun systemEvent(event: String, data: Map<String, String> = emptyMap()) {
        add(
            type = Breadcrumb.BreadcrumbType.SYSTEM,
            category = "app.lifecycle",
            message = event,
            data = data
        )
    }

    fun userAction(action: String, data: Map<String, String> = emptyMap()) {
        add(
            type = Breadcrumb.BreadcrumbType.USER,
            category = "user",
            message = action,
            data = data
        )
    }
}
```

### Auto-Collection via Lifecycle Callbacks

```kotlin
class BreadcrumbLifecycleCallbacks : Application.ActivityLifecycleCallbacks {
    private var currentActivity: String? = null

    override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {
        BreadcrumbManager.systemEvent(
            "Activity created: ${activity.javaClass.simpleName}",
            mapOf("restored" to (savedInstanceState != null).toString())
        )
    }

    override fun onActivityResumed(activity: Activity) {
        val activityName = activity.javaClass.simpleName
        val previousActivity = currentActivity
        currentActivity = activityName

        if (previousActivity != null && previousActivity != activityName) {
            BreadcrumbManager.navigation(previousActivity, activityName)
        }
    }

    override fun onActivityPaused(activity: Activity) {}
    override fun onActivityStarted(activity: Activity) {}
    override fun onActivityStopped(activity: Activity) {}
    override fun onActivitySaveInstanceState(activity: Activity, outState: Bundle) {}
    override fun onActivityDestroyed(activity: Activity) {}
}

// Memory warning callback
class MemoryCallback : ComponentCallbacks2 {
    override fun onTrimMemory(level: Int) {
        val levelName = when (level) {
            ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW -> "RUNNING_LOW"
            ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL -> "RUNNING_CRITICAL"
            ComponentCallbacks2.TRIM_MEMORY_COMPLETE -> "COMPLETE"
            else -> "LEVEL_$level"
        }

        BreadcrumbManager.add(
            type = Breadcrumb.BreadcrumbType.SYSTEM,
            category = "memory",
            message = "Memory trim: $levelName",
            level = Breadcrumb.Level.WARNING,
            data = mapOf("level" to level.toString())
        )
    }

    override fun onConfigurationChanged(newConfig: Configuration) {}
    override fun onLowMemory() {
        BreadcrumbManager.add(
            type = Breadcrumb.BreadcrumbType.SYSTEM,
            category = "memory",
            message = "Low memory warning",
            level = Breadcrumb.Level.ERROR
        )
    }
}
```

---

## React Native

### Breadcrumb Strategy

```typescript
import * as Sentry from '@sentry/react-native';

// User action
Sentry.addBreadcrumb({
  category: 'ui.click',
  message: 'Tapped "Submit Order" button',
  level: 'info',
});

// State transition
Sentry.addBreadcrumb({
  category: 'state',
  message: 'Cart updated',
  data: { itemCount: 3, total: 99.99 },
  level: 'info',
});

// Navigation
Sentry.addBreadcrumb({
  category: 'navigation',
  message: 'Navigated to CheckoutScreen',
  level: 'info',
});

// App lifecycle
Sentry.addBreadcrumb({
  category: 'app.lifecycle',
  message: 'App went to background',
  level: 'info',
});
```

### App Lifecycle Hook

```typescript
// hooks/useAppLifecycle.ts
import { useEffect } from 'react';
import { AppState, AppStateStatus } from 'react-native';
import * as Sentry from '@sentry/react-native';

export function useAppLifecycleBreadcrumbs() {
  useEffect(() => {
    const subscription = AppState.addEventListener('change', (state: AppStateStatus) => {
      Sentry.addBreadcrumb({
        category: 'app.lifecycle',
        message: `App state changed to: ${state}`,
        level: 'info',
        data: { state },
      });
    });

    return () => subscription.remove();
  }, []);
}
```

---

## What NOT to Track

- Every keystroke in text inputs
- Scroll events
- Animation frames
- Verbose debug logs
- Sensitive data (passwords, tokens, full addresses)

---

## Integration Points

- **[crash-reporting.md](../crash-reporting.md)** - Breadcrumbs attached to crashes
- **[error-boundary.template.md](error-boundary.template.md)** - Error capture with breadcrumb context
