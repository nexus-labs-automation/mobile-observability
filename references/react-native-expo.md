# React Native & Expo Observability

Instrumentation guide for React Native apps, with emphasis on Expo managed workflow.

## SDK Setup

### Sentry (Recommended Primary)

```bash
npx expo install @sentry/react-native
```

```typescript
// app/_layout.tsx or App.tsx (root)
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: process.env.EXPO_PUBLIC_SENTRY_DSN,

  // Environment tagging
  environment: __DEV__ ? 'development' : 'production',

  // Release tracking (critical for source maps)
  release: `${Application.applicationId}@${Application.nativeApplicationVersion}+${Application.nativeBuildVersion}`,
  dist: Application.nativeBuildVersion,

  // Performance sampling
  tracesSampleRate: __DEV__ ? 1.0 : 0.2,  // 20% in prod
  profilesSampleRate: 0.1,                  // 10% profiling

  // Error sampling (always 100% for errors)
  sampleRate: 1.0,

  // Breadcrumb limits
  maxBreadcrumbs: 100,

  // Attach screenshots on crash (iOS/Android)
  attachScreenshot: true,

  // Session tracking for crash-free rate
  enableAutoSessionTracking: true,
  sessionTrackingIntervalMillis: 30000,

  // Filter before sending
  beforeSend(event) {
    // Strip PII from user object
    if (event.user) {
      delete event.user.email;
      delete event.user.ip_address;
    }
    // Drop development errors
    if (__DEV__) return null;
    return event;
  },

  beforeBreadcrumb(breadcrumb) {
    // Sanitize URLs
    if (breadcrumb.category === 'fetch' && breadcrumb.data?.url) {
      breadcrumb.data.url = sanitizeUrl(breadcrumb.data.url);
    }
    return breadcrumb;
  },
});
```

### Source Maps (Critical)

Without source maps, stack traces are useless minified garbage.

```json
// app.json
{
  "expo": {
    "plugins": [
      [
        "@sentry/react-native/expo",
        {
          "organization": "your-org",
          "project": "your-project"
        }
      ]
    ],
    "hooks": {
      "postPublish": [
        {
          "file": "sentry-expo/upload-sourcemaps",
          "config": {
            "organization": "your-org",
            "project": "your-project"
          }
        }
      ]
    }
  }
}
```

```bash
# EAS Build (recommended)
# Set in eas.json or Expo secrets
SENTRY_AUTH_TOKEN=your-token
SENTRY_ORG=your-org
SENTRY_PROJECT=your-project
```

## Error Boundaries

### Root Error Boundary

```typescript
// components/ErrorBoundary.tsx
import * as Sentry from '@sentry/react-native';
import { ErrorBoundary as SentryErrorBoundary } from '@sentry/react-native';

export function RootErrorBoundary({ children }: { children: React.ReactNode }) {
  return (
    <SentryErrorBoundary
      fallback={({ error, resetError }) => (
        <ErrorFallbackScreen error={error} onRetry={resetError} />
      )}
      onError={(error, componentStack) => {
        // Already captured by Sentry, but add context
        Sentry.withScope((scope) => {
          scope.setTag('error.boundary', 'root');
          scope.setContext('component_stack', { stack: componentStack });
        });
      }}
    >
      {children}
    </SentryErrorBoundary>
  );
}
```

### Screen-Level Boundaries

Wrap each screen to isolate failures and enable partial recovery:

```typescript
// Higher-order component for screens
export function withScreenErrorBoundary<P extends object>(
  WrappedComponent: React.ComponentType<P>,
  screenName: string
) {
  return function ScreenWithBoundary(props: P) {
    return (
      <SentryErrorBoundary
        fallback={<ScreenErrorFallback screenName={screenName} />}
        onError={(error) => {
          Sentry.withScope((scope) => {
            scope.setTag('error.boundary', 'screen');
            scope.setTag('screen.name', screenName);
          });
        }}
      >
        <WrappedComponent {...props} />
      </SentryErrorBoundary>
    );
  };
}
```

## Navigation Instrumentation

### React Navigation Integration

```typescript
// navigation/index.tsx
import * as Sentry from '@sentry/react-native';
import { NavigationContainer } from '@react-navigation/native';

const routingInstrumentation = new Sentry.ReactNavigationInstrumentation();

// In Sentry.init()
Sentry.init({
  integrations: [
    new Sentry.ReactNativeTracing({
      routingInstrumentation,
      // Track these as child spans
      tracingOrigins: ['localhost', 'api.yourapp.com', /^\//],
    }),
  ],
});

// In NavigationContainer
export function Navigation() {
  const navigation = useNavigationContainerRef();

  return (
    <NavigationContainer
      ref={navigation}
      onReady={() => {
        routingInstrumentation.registerNavigationContainer(navigation);
      }}
    >
      {/* screens */}
    </NavigationContainer>
  );
}
```

### Expo Router Integration

```typescript
// app/_layout.tsx
import * as Sentry from '@sentry/react-native';
import { useNavigationContainerRef, Slot } from 'expo-router';

const routingInstrumentation = new Sentry.ReactNavigationInstrumentation();

export default function RootLayout() {
  const ref = useNavigationContainerRef();

  useEffect(() => {
    if (ref) {
      routingInstrumentation.registerNavigationContainer(ref);
    }
  }, [ref]);

  return <Slot />;
}
```

## Performance Monitoring

### App Start Measurement

```typescript
// Measure cold start vs warm start
import * as Sentry from '@sentry/react-native';

// This is automatic with ReactNativeTracing, but for custom measurement:
const appStartSpan = Sentry.startTransaction({
  name: 'app.start',
  op: 'app.start.cold', // or 'app.start.warm'
});

// When app is interactive
function onAppReady() {
  appStartSpan?.finish();

  // Also measure Time to Interactive
  Sentry.setMeasurement('time_to_interactive', performance.now(), 'millisecond');
}
```

### Screen Load Transactions

```typescript
// hooks/useScreenTransaction.ts
import * as Sentry from '@sentry/react-native';
import { useEffect, useRef } from 'react';

export function useScreenTransaction(screenName: string) {
  const transactionRef = useRef<Sentry.Transaction | null>(null);

  useEffect(() => {
    transactionRef.current = Sentry.startTransaction({
      name: screenName,
      op: 'ui.load',
    });

    return () => {
      transactionRef.current?.finish();
    };
  }, [screenName]);

  // Return helper to add child spans
  return {
    startSpan: (name: string, op: string) => {
      return transactionRef.current?.startChild({ op, description: name });
    },
  };
}

// Usage in screen
function ProfileScreen() {
  const { startSpan } = useScreenTransaction('ProfileScreen');

  useEffect(() => {
    const fetchSpan = startSpan('fetch_profile', 'http.client');
    fetchProfile().finally(() => fetchSpan?.finish());
  }, []);
}
```

### Network Request Instrumentation

Already handled by ReactNativeTracing, but for custom fetch wrapper:

```typescript
// utils/instrumentedFetch.ts
import * as Sentry from '@sentry/react-native';

export async function instrumentedFetch(
  url: string,
  options?: RequestInit
): Promise<Response> {
  const span = Sentry.startSpan({
    op: 'http.client',
    name: `${options?.method || 'GET'} ${new URL(url).pathname}`,
  });

  try {
    const response = await fetch(url, options);

    span?.setHttpStatus(response.status);
    span?.setData('response_size', response.headers.get('content-length'));

    if (!response.ok) {
      Sentry.addBreadcrumb({
        category: 'http',
        message: `HTTP ${response.status}: ${url}`,
        level: 'warning',
        data: { url, status: response.status },
      });
    }

    return response;
  } catch (error) {
    span?.setStatus('internal_error');
    throw error;
  } finally {
    span?.finish();
  }
}
```

## User Context

```typescript
// On login
function onUserLogin(user: User) {
  Sentry.setUser({
    id: user.id,  // Required - use internal ID, not email
    // Optional - be careful with PII
    username: user.username,
    segment: user.subscriptionTier,  // Custom data
  });

  Sentry.setTag('user.tier', user.subscriptionTier);
}

// On logout
function onUserLogout() {
  Sentry.setUser(null);
}
```

## Breadcrumb Strategy

### What to Track (High Value)

```typescript
// User interactions
Sentry.addBreadcrumb({
  category: 'ui.click',
  message: 'Tapped "Submit Order" button',
  level: 'info',
});

// State transitions
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

### What NOT to Track (Noise)

- Every keystroke in text inputs
- Scroll events
- Animation frames
- Verbose debug logs
- Sensitive data (passwords, tokens, full addresses)

### App Lifecycle Breadcrumbs

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

## Offline Handling

Mobile apps lose connectivity. Queue errors for later:

```typescript
// Sentry handles this automatically with:
Sentry.init({
  enableAutoSessionTracking: true,
  // Events are cached and sent when back online
});

// For custom offline awareness:
import NetInfo from '@react-native-community/netinfo';

NetInfo.addEventListener((state) => {
  Sentry.setTag('network.connected', state.isConnected ? 'yes' : 'no');
  Sentry.setTag('network.type', state.type);

  if (!state.isConnected) {
    Sentry.addBreadcrumb({
      category: 'network',
      message: 'Device went offline',
      level: 'warning',
    });
  }
});
```

## Common Gotchas

### 1. Missing Native Crashes on Expo Go
Expo Go can't capture native crashes. Test with development builds:
```bash
npx expo run:ios  # or run:android
```

### 2. Hermes Stack Traces
Hermes requires source maps. Without them, you get bytecode offsets.
Verify upload: `sentry-cli sourcemaps explain <event-id>`

### 3. Release Mismatch
If stack traces show `<unknown>`, release string doesn't match uploaded source maps.
Use exact same format: `bundle-id@version+buildNumber`

### 4. Over-Sampling Performance
20% trace sample rate is sufficient for most apps. Higher rates:
- Drain battery faster
- Increase data costs for users
- Hit Sentry quotas quickly

### 5. Breadcrumb Overflow
Default is 100 breadcrumbs. In long sessions, early breadcrumbs are lost.
Consider: Starting new breadcrumb trail on significant events (login, major navigation).

## Testing Instrumentation

```typescript
// utils/testObservability.ts
export function triggerTestCrash() {
  throw new Error('Test crash from observability check');
}

export function triggerNativeCrash() {
  Sentry.nativeCrash();  // Actually crashes the app
}

export function sendTestEvent() {
  Sentry.captureMessage('Test event', 'info');
}

// Verify in Sentry dashboard within 30 seconds
```

---

## Reusable Templates

- **[templates/screen-load-tracking.template.md](templates/screen-load-tracking.template.md)** - React Native screen load tracking hooks
- **[templates/error-boundary.template.md](templates/error-boundary.template.md)** - React Native error boundaries
- **[templates/breadcrumb-manager.template.md](templates/breadcrumb-manager.template.md)** - React Native breadcrumb strategies
- **[templates/navigation-tracking.template.md](templates/navigation-tracking.template.md)** - React Navigation/Expo Router instrumentation
