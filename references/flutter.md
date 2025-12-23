# Flutter Observability

Instrumentation patterns for Flutter applications (Dart/Flutter).

## SDK Setup

### Sentry

**pubspec.yaml:**

```yaml
dependencies:
  sentry_flutter: ^7.18.0

dev_dependencies:
  sentry_dart_plugin: ^1.7.0

sentry:
  upload_debug_symbols: true
  upload_source_maps: false
  upload_sources: false
  project: your-project
  org: your-org
  auth_token: your-token
```

**main.dart:**

```dart
import 'package:flutter/material.dart';
import 'package:sentry_flutter/sentry_flutter.dart';

Future<void> main() async {
  await SentryFlutter.init(
    (options) {
      options.dsn = 'https://your-dsn@sentry.io/project';

      // Environment
      options.environment = kReleaseMode ? 'production' : 'development';

      // Release tracking
      options.release = 'app@1.0.0+1';
      options.dist = '1';

      // Performance sampling
      options.tracesSampleRate = kReleaseMode ? 0.2 : 1.0;
      options.profilesSampleRate = 0.1;

      // Auto-instrumentation
      options.enableAutoPerformanceTracing = true;
      options.enableAutoNavigationTracking = true;

      // Breadcrumbs
      options.maxBreadcrumbs = 100;

      // Attachments
      options.attachScreenshot = true;
      options.attachViewHierarchy = true;

      // Session tracking
      options.enableAutoSessionTracking = true;

      // PII
      options.sendDefaultPii = false;

      // Before send hook
      options.beforeSend = (event, hint) {
        // Strip PII
        if (event.user != null) {
          event = event.copyWith(
            user: event.user!.copyWith(
              email: null,
              ipAddress: null,
            ),
          );
        }
        return event;
      };
    },
    appRunner: () => runApp(const MyApp()),
  );
}
```

**Alternative: Manual initialization (custom error handling):**

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await Sentry.init(
    (options) {
      options.dsn = 'https://your-dsn@sentry.io/project';
      // ... other options
    },
  );

  // Custom error handling
  FlutterError.onError = (FlutterErrorDetails details) {
    Sentry.captureException(
      details.exception,
      stackTrace: details.stack,
    );
    FlutterError.presentError(details);
  };

  PlatformDispatcher.instance.onError = (error, stack) {
    Sentry.captureException(error, stackTrace: stack);
    return true;
  };

  runZonedGuarded(
    () => runApp(const MyApp()),
    (error, stack) {
      Sentry.captureException(error, stackTrace: stack);
    },
  );
}
```

### Debug Symbols Upload

**Build script (iOS):**

```bash
# Add to Xcode build phase (AFTER "Compile Sources")
if [ "$CONFIGURATION" == "Release" ]; then
    /bin/sh "$FLUTTER_ROOT/packages/flutter_tools/bin/sentry_utils.sh"
fi
```

**Build script (Android):**

Automatic with `sentry_dart_plugin`. Verify `android/build.gradle`:

```gradle
plugins {
    id 'io.sentry.android.gradle' version '4.0.0'
}

sentry {
    includeProguardMapping = true
    autoUploadProguardMapping = true
    uploadNativeSymbols = true
}
```

**Obfuscated builds:**

```bash
# Build with symbols
flutter build apk --release --split-debug-info=build/app/outputs/symbols

# Upload symbols
sentry-cli upload-dif \
    --org your-org \
    --project your-project \
    build/app/outputs/symbols
```

### Datadog Alternative

**pubspec.yaml:**

```yaml
dependencies:
  datadog_flutter_plugin: ^2.0.0
```

**main.dart:**

```dart
import 'package:datadog_flutter_plugin/datadog_flutter_plugin.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  final configuration = DatadogConfiguration(
    clientToken: 'client-token',
    env: 'production',
    site: DatadogSite.us1,
    nativeCrashReportEnabled: true,
    loggingConfiguration: DatadogLoggingConfiguration(),
    rumConfiguration: DatadogRumConfiguration(
      applicationId: 'app-id',
      sessionSamplingRate: 100.0,
    ),
  );

  await DatadogSdk.instance.initialize(configuration);

  // Use DatadogSdk.runApp for automatic error handling
  DatadogSdk.runApp(
    configuration,
    TrackingConsent.granted,
    () => runApp(const MyApp()),
  );
}
```

## Error Capture

### Comprehensive Error Handling Setup

Flutter has three error channels that must all be handled:

```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await Sentry.init((options) {
    options.dsn = 'your-dsn';
  });

  // 1. Flutter framework errors (widgets, rendering)
  FlutterError.onError = (FlutterErrorDetails details) {
    Sentry.captureException(
      details.exception,
      stackTrace: details.stack,
    );
    // Still show in debug console
    FlutterError.presentError(details);
  };

  // 2. Platform channel errors (iOS/Android method calls)
  PlatformDispatcher.instance.onError = (error, stack) {
    Sentry.captureException(error, stackTrace: stack);
    return true; // Handled
  };

  // 3. Async errors (Future errors not caught by try/catch)
  runZonedGuarded(
    () => runApp(const MyApp()),
    (error, stack) {
      Sentry.captureException(error, stackTrace: stack);
    },
  );
}
```

### Capturing Handled Errors

```dart
// Caught exception
try {
  await riskyOperation();
} catch (error, stackTrace) {
  await Sentry.captureException(
    error,
    stackTrace: stackTrace,
    withScope: (scope) {
      scope.setTag('flow', 'payment');
      scope.setContexts('order', {'orderId': orderId});
    },
  );
}

// With custom scope
await Sentry.captureException(
  error,
  withScope: (scope) {
    scope.level = SentryLevel.warning;
    scope.setTag('feature', 'checkout');
    scope.setExtra('debug', debugData);
  },
);

// Message (non-error event)
await Sentry.captureMessage('User attempted invalid operation');
```

### Custom Exception Types

```dart
// Domain-specific exceptions for better grouping
class NetworkException implements Exception {
  final String message;
  final int? statusCode;
  final String? endpoint;

  NetworkException(this.message, {this.statusCode, this.endpoint});

  @override
  String toString() => 'NetworkException: $message (HTTP $statusCode)';
}

class ValidationException implements Exception {
  final String message;
  final String field;
  final dynamic value;

  ValidationException(this.message, {required this.field, this.value});

  @override
  String toString() => 'ValidationException: $message (field: $field)';
}

// Capture with context
await Sentry.captureException(
  NetworkException(
    'Request failed',
    statusCode: 500,
    endpoint: '/api/users',
  ),
  withScope: (scope) {
    scope.setTag('error.type', 'network');
  },
);
```

### Widget Error Boundaries

```dart
class ErrorBoundary extends StatefulWidget {
  final Widget child;
  final Widget Function(Object error, StackTrace? stack)? errorBuilder;

  const ErrorBoundary({
    super.key,
    required this.child,
    this.errorBuilder,
  });

  @override
  State<ErrorBoundary> createState() => _ErrorBoundaryState();
}

class _ErrorBoundaryState extends State<ErrorBoundary> {
  Object? _error;
  StackTrace? _stack;

  @override
  Widget build(BuildContext context) {
    if (_error != null) {
      return widget.errorBuilder?.call(_error!, _stack) ??
          ErrorWidget(_error!);
    }

    return ErrorHandler(
      onError: (error, stack) {
        Sentry.captureException(error, stackTrace: stack);
        setState(() {
          _error = error;
          _stack = stack;
        });
      },
      child: widget.child,
    );
  }
}

class ErrorHandler extends StatelessWidget {
  final Widget child;
  final void Function(Object error, StackTrace? stack) onError;

  const ErrorHandler({
    super.key,
    required this.child,
    required this.onError,
  });

  @override
  Widget build(BuildContext context) {
    return ErrorWidget.builder = (FlutterErrorDetails details) {
      onError(details.exception, details.stack);
      return ErrorWidget(details.exception);
    };
    return child;
  }
}
```

## Performance Monitoring

### App Launch Measurement

```dart
class MyApp extends StatefulWidget {
  const MyApp({super.key});

  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  ISentrySpan? _launchSpan;

  @override
  void initState() {
    super.initState();

    // Start launch span
    _launchSpan = Sentry.startTransaction(
      'app.launch',
      'app.lifecycle',
    );

    // Mark first frame
    WidgetsBinding.instance.addPostFrameCallback((_) {
      _launchSpan?.finish();
    });
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: const HomeScreen(),
      navigatorObservers: [
        SentryNavigatorObserver(),
      ],
    );
  }
}
```

### Route/Screen Tracking

**Automatic with NavigatorObserver:**

```dart
MaterialApp(
  navigatorObservers: [
    SentryNavigatorObserver(),
  ],
  routes: {
    '/': (context) => const HomeScreen(),
    '/profile': (context) => const ProfileScreen(),
  },
);

// Or with GoRouter
final router = GoRouter(
  observers: [SentryNavigatorObserver()],
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomeScreen(),
    ),
  ],
);
```

**Manual tracking:**

```dart
class TrackedScreen extends StatefulWidget {
  final String screenName;
  final Widget child;

  const TrackedScreen({
    super.key,
    required this.screenName,
    required this.child,
  });

  @override
  State<TrackedScreen> createState() => _TrackedScreenState();
}

class _TrackedScreenState extends State<TrackedScreen> {
  ISentrySpan? _span;

  @override
  void initState() {
    super.initState();
    _span = Sentry.startTransaction(
      widget.screenName,
      'ui.load',
    );
  }

  @override
  void dispose() {
    _span?.finish();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return widget.child;
  }
}

// Usage
class ProfileScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return TrackedScreen(
      screenName: 'ProfileScreen',
      child: Scaffold(
        // ... screen content
      ),
    );
  }
}
```

### Widget Rebuild Tracking

```dart
class RebuildTracker extends StatefulWidget {
  final String widgetName;
  final Widget child;

  const RebuildTracker({
    super.key,
    required this.widgetName,
    required this.child,
  });

  @override
  State<RebuildTracker> createState() => _RebuildTrackerState();
}

class _RebuildTrackerState extends State<RebuildTracker> {
  int _buildCount = 0;

  @override
  Widget build(BuildContext context) {
    _buildCount++;

    // Track excessive rebuilds
    if (_buildCount > 10) {
      Sentry.addBreadcrumb(
        Breadcrumb(
          message: 'Excessive rebuilds: ${widget.widgetName}',
          level: SentryLevel.warning,
          category: 'performance',
          data: {'buildCount': _buildCount},
        ),
      );
    }

    return widget.child;
  }
}
```

### Network Request Instrumentation

**With Dio:**

```dart
import 'package:dio/dio.dart';
import 'package:sentry_dio/sentry_dio.dart';

final dio = Dio()
  ..addSentry(); // Automatic instrumentation

// Manual
Future<T> tracedRequest<T>(
  String name,
  Future<T> Function() request,
) async {
  final span = Sentry.getSpan()?.startChild(
    'http.client',
    description: name,
  );

  try {
    final result = await request();
    span?.finish(status: SpanStatus.ok());
    return result;
  } catch (error) {
    span?.finish(status: SpanStatus.internalError());
    rethrow;
  }
}

// Usage
final users = await tracedRequest('GET /api/users', () async {
  return await dio.get<List>('/api/users');
});
```

**With http package:**

```dart
import 'package:http/http.dart' as http;
import 'package:sentry_flutter/sentry_flutter.dart';

final client = SentryHttpClient(); // Automatic instrumentation

// Manual
Future<http.Response> tracedGet(String url) async {
  final span = Sentry.getSpan()?.startChild(
    'http.client',
    description: 'GET $url',
  );

  try {
    final response = await http.get(Uri.parse(url));

    span?.setData('http.status_code', response.statusCode);
    span?.setData('http.response_content_length', response.contentLength);
    span?.finish(status: SpanStatus.ok());

    return response;
  } catch (error) {
    span?.finish(status: SpanStatus.internalError());
    rethrow;
  }
}
```

### Database Instrumentation

```dart
// Wrap database operations
Future<T> tracedDbQuery<T>(
  String operation,
  Future<T> Function() query,
) async {
  final span = Sentry.getSpan()?.startChild(
    'db.query',
    description: operation,
  );

  try {
    final result = await query();
    span?.finish(status: SpanStatus.ok());
    return result;
  } catch (error) {
    span?.finish(status: SpanStatus.internalError());
    rethrow;
  }
}

// Usage with sqflite
Future<List<User>> getUsers() async {
  return tracedDbQuery('SELECT users', () async {
    final db = await database;
    final maps = await db.query('users');
    return maps.map((map) => User.fromMap(map)).toList();
  });
}
```

## Breadcrumbs

### Automatic Breadcrumbs

Enabled by default:
- Navigation events (with SentryNavigatorObserver)
- HTTP requests (with instrumented clients)
- App lifecycle events

### Manual Breadcrumbs

```dart
// User action
Sentry.addBreadcrumb(
  Breadcrumb(
    message: 'Tapped checkout button',
    level: SentryLevel.info,
    category: 'user',
    data: {'cartItems': 3},
  ),
);

// Navigation
Sentry.addBreadcrumb(
  Breadcrumb.navigation(
    from: 'HomeScreen',
    to: 'ProfileScreen',
  ),
);

// State change
Sentry.addBreadcrumb(
  Breadcrumb(
    message: 'User logged in',
    level: SentryLevel.info,
    category: 'state',
    data: {
      'userId': user.id,
      'isFirstLogin': user.isFirstLogin,
    },
  ),
);

// HTTP request
Sentry.addBreadcrumb(
  Breadcrumb.http(
    url: Uri.parse('https://api.example.com/users'),
    method: 'GET',
    statusCode: 200,
  ),
);
```

### App Lifecycle Breadcrumbs

```dart
class AppLifecycleObserver extends WidgetsBindingObserver {
  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    final message = switch (state) {
      AppLifecycleState.resumed => 'App resumed (foreground)',
      AppLifecycleState.inactive => 'App inactive',
      AppLifecycleState.paused => 'App paused (background)',
      AppLifecycleState.detached => 'App detached',
      AppLifecycleState.hidden => 'App hidden',
    };

    Sentry.addBreadcrumb(
      Breadcrumb(
        message: message,
        level: SentryLevel.info,
        category: 'app.lifecycle',
        data: {'state': state.name},
      ),
    );
  }
}

// Register in main app widget
class _MyAppState extends State<MyApp> {
  final _observer = AppLifecycleObserver();

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(_observer);
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(_observer);
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(home: const HomeScreen());
  }
}
```

## User Context

```dart
// On login
void onUserLogin(User user) {
  Sentry.configureScope((scope) {
    scope.setUser(
      SentryUser(
        id: user.id, // Internal ID only
        username: user.username,
        data: {
          'subscription': user.subscriptionTier,
          'accountAge': user.accountAgeDays,
        },
      ),
    );

    scope.setTag('user.tier', user.subscriptionTier);
  });
}

// On logout
void onUserLogout() {
  Sentry.configureScope((scope) {
    scope.setUser(null);
  });
}

// Add custom context
Sentry.configureScope((scope) {
  scope.setContexts('cart', {
    'itemCount': 3,
    'total': 99.99,
    'currency': 'USD',
  });
});
```

## Platform Channel Instrumentation

### Method Channel Calls

```dart
class InstrumentedMethodChannel {
  final MethodChannel _channel;
  final String name;

  InstrumentedMethodChannel(this.name) : _channel = MethodChannel(name);

  Future<T?> invokeMethod<T>(String method, [dynamic arguments]) async {
    final span = Sentry.getSpan()?.startChild(
      'platform.channel',
      description: '$name.$method',
    );

    try {
      final result = await _channel.invokeMethod<T>(method, arguments);
      span?.finish(status: SpanStatus.ok());
      return result;
    } catch (error, stackTrace) {
      span?.finish(status: SpanStatus.internalError());

      // Platform exceptions won't auto-report, must manually capture
      await Sentry.captureException(
        error,
        stackTrace: stackTrace,
        withScope: (scope) {
          scope.setTag('platform.channel', name);
          scope.setTag('platform.method', method);
        },
      );

      rethrow;
    }
  }
}

// Usage
final channel = InstrumentedMethodChannel('com.example.app/battery');
final batteryLevel = await channel.invokeMethod<int>('getBatteryLevel');
```

### Event Channel Streams

```dart
class InstrumentedEventChannel {
  final EventChannel _channel;
  final String name;

  InstrumentedEventChannel(this.name) : _channel = EventChannel(name);

  Stream<T> receiveBroadcastStream<T>([dynamic arguments]) {
    return _channel.receiveBroadcastStream(arguments).transform(
      StreamTransformer<dynamic, T>.fromHandlers(
        handleData: (data, sink) {
          Sentry.addBreadcrumb(
            Breadcrumb(
              message: 'Platform event received',
              category: 'platform.event',
              data: {'channel': name, 'event': data.toString()},
            ),
          );
          sink.add(data as T);
        },
        handleError: (error, stackTrace, sink) {
          Sentry.captureException(
            error,
            stackTrace: stackTrace,
            withScope: (scope) {
              scope.setTag('platform.channel', name);
            },
          );
          sink.addError(error, stackTrace);
        },
      ),
    );
  }
}
```

## Isolate Error Handling

```dart
// Spawning isolates with error handling
Future<void> spawnInstrumentedIsolate<T>(
  void Function(SendPort) entryPoint,
  void Function(T) onMessage,
) async {
  final receivePort = ReceivePort();

  // Handle errors from the isolate
  final errorPort = ReceivePort();
  errorPort.listen((errorData) {
    if (errorData is List && errorData.length == 2) {
      final error = errorData[0];
      final stack = errorData[1];

      Sentry.captureException(
        error,
        stackTrace: stack is String ? StackTrace.fromString(stack) : null,
        withScope: (scope) {
          scope.setTag('error.source', 'isolate');
        },
      );
    }
  });

  await Isolate.spawn(
    entryPoint,
    receivePort.sendPort,
    onError: errorPort.sendPort,
    onExit: receivePort.sendPort,
    errorsAreFatal: false,
  );

  receivePort.listen((message) {
    if (message is T) {
      onMessage(message);
    }
  });
}

// Worker isolate with error handling
void isolateEntryPoint(SendPort sendPort) {
  // Critical: Set up error handling in the isolate
  Isolate.current.addErrorListener(
    RawReceivePort((errorData) {
      // Error will be forwarded to parent isolate
    }).sendPort,
  );

  try {
    // Perform work
    final result = performHeavyComputation();
    sendPort.send(result);
  } catch (error, stackTrace) {
    // Manually send error to parent
    sendPort.send({'error': error, 'stackTrace': stackTrace.toString()});
  }
}
```

## Flutter-Specific Considerations

### Hot Reload vs Production

Development vs production behavior differs significantly:

```dart
Future<void> main() async {
  await SentryFlutter.init(
    (options) {
      options.dsn = 'your-dsn';

      // Different sampling in debug
      options.tracesSampleRate = kReleaseMode ? 0.2 : 1.0;

      // Don't spam in development
      options.beforeSend = (event, hint) {
        if (kDebugMode) {
          debugPrint('Sentry event: ${event.message}');
          return null; // Don't send in debug
        }
        return event;
      };
    },
    appRunner: () => runApp(const MyApp()),
  );
}
```

### Async Error Timing

```dart
// BAD: Error happens outside zone
void main() {
  Future.delayed(Duration.zero, () {
    throw Exception('Not caught!'); // Won't be captured
  });

  runZonedGuarded(
    () => runApp(const MyApp()),
    (error, stack) {
      Sentry.captureException(error, stackTrace: stack);
    },
  );
}

// GOOD: All async work inside zone
void main() {
  runZonedGuarded(
    () {
      WidgetsFlutterBinding.ensureInitialized();

      Future.delayed(Duration.zero, () {
        throw Exception('Caught!'); // Will be captured
      });

      runApp(const MyApp());
    },
    (error, stack) {
      Sentry.captureException(error, stackTrace: stack);
    },
  );
}
```

### State Management Integration

**Provider:**

```dart
class InstrumentedChangeNotifier extends ChangeNotifier {
  @override
  void notifyListeners() {
    try {
      super.notifyListeners();
    } catch (error, stackTrace) {
      Sentry.captureException(error, stackTrace: stackTrace);
      rethrow;
    }
  }
}
```

**Riverpod:**

```dart
final observabilityProvider = Provider((ref) {
  ref.onDispose(() {
    Sentry.addBreadcrumb(
      Breadcrumb(
        message: 'Provider disposed',
        category: 'state',
      ),
    );
  });

  return MyService();
});
```

**BLoC:**

```dart
abstract class InstrumentedBloc<Event, State> extends Bloc<Event, State> {
  InstrumentedBloc(super.initialState) {
    on<Event>((event, emit) async {
      final span = Sentry.startTransaction(
        '${runtimeType}.${event.runtimeType}',
        'bloc.event',
      );

      try {
        await onEvent(event, emit);
        span.finish(status: SpanStatus.ok());
      } catch (error, stackTrace) {
        span.finish(status: SpanStatus.internalError());
        Sentry.captureException(error, stackTrace: stackTrace);
        rethrow;
      }
    });
  }

  Future<void> onEvent(Event event, Emitter<State> emit);
}
```

### Memory Leaks

```dart
// Track widget disposal to detect leaks
class LeakDetector extends StatefulWidget {
  final String widgetName;
  final Widget child;

  const LeakDetector({
    super.key,
    required this.widgetName,
    required this.child,
  });

  @override
  State<LeakDetector> createState() => _LeakDetectorState();
}

class _LeakDetectorState extends State<LeakDetector> {
  late final DateTime _created;

  @override
  void initState() {
    super.initState();
    _created = DateTime.now();
  }

  @override
  void dispose() {
    final lifetime = DateTime.now().difference(_created);

    // Warn if widget lived too long
    if (lifetime.inMinutes > 30) {
      Sentry.addBreadcrumb(
        Breadcrumb(
          message: 'Long-lived widget: ${widget.widgetName}',
          level: SentryLevel.warning,
          category: 'memory',
          data: {'lifetimeMinutes': lifetime.inMinutes},
        ),
      );
    }

    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return widget.child;
  }
}
```

## Common Flutter Gotchas

### 1. Obfuscation Without Symbol Upload

Obfuscated builds show mangled stack traces. Always upload symbols:

```bash
# Build with debug info
flutter build apk --release --split-debug-info=build/symbols --obfuscate

# Upload to Sentry
sentry-cli upload-dif --org your-org --project your-project build/symbols
```

### 2. PlatformException Not Captured

Platform channel errors need manual capture:

```dart
// Won't auto-report
try {
  await platform.invokeMethod('someMethod');
} catch (error, stackTrace) {
  // Must manually capture
  await Sentry.captureException(error, stackTrace: stackTrace);
}
```

### 3. Widget Build Errors in Release

Release mode hides widget errors. Don't disable them:

```dart
// BAD: Hides production errors
ErrorWidget.builder = (details) => const SizedBox.shrink();

// GOOD: Show error + report
ErrorWidget.builder = (details) {
  Sentry.captureException(details.exception, stackTrace: details.stack);
  return kReleaseMode
      ? const Center(child: Text('Something went wrong'))
      : ErrorWidget(details.exception);
};
```

### 4. Initialization Race Conditions

```dart
// BAD: Sentry not initialized yet
void main() {
  Sentry.captureMessage('App starting'); // May not send

  SentryFlutter.init(
    (options) => options.dsn = 'your-dsn',
    appRunner: () => runApp(const MyApp()),
  );
}

// GOOD: Wait for init
Future<void> main() async {
  await SentryFlutter.init(
    (options) => options.dsn = 'your-dsn',
    appRunner: () {
      Sentry.captureMessage('App starting'); // Will send
      runApp(const MyApp());
    },
  );
}
```

### 5. Navigator Context Errors

```dart
// BAD: Using BuildContext after async gap
void _handleTap(BuildContext context) async {
  await Future.delayed(const Duration(seconds: 1));
  Navigator.push(context, route); // May crash if widget unmounted
}

// GOOD: Check mounted
void _handleTap(BuildContext context) async {
  await Future.delayed(const Duration(seconds: 1));
  if (!mounted) return;
  Navigator.push(context, route);
}
```

### 6. Isolate Errors Silent Failures

```dart
// BAD: Isolate errors not propagated
await Isolate.spawn(workerFunction, message);

// GOOD: Handle isolate errors
final errorPort = ReceivePort();
await Isolate.spawn(
  workerFunction,
  message,
  onError: errorPort.sendPort,
);

errorPort.listen((errorData) {
  Sentry.captureMessage('Isolate error: $errorData');
});
```

### 7. Build-Time vs Runtime Errors

```dart
// Build errors show in debug, crash in release
class MyWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // This throws in release if data is null
    final data = Provider.of<MyData>(context);
    return Text(data.value); // NPE if data is null
  }
}

// Safe approach
class MyWidget extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    try {
      final data = Provider.of<MyData>(context);
      return Text(data.value);
    } catch (error, stackTrace) {
      Sentry.captureException(error, stackTrace: stackTrace);
      return const Text('Error loading data');
    }
  }
}
```

## Testing

```dart
// Test crash
void triggerTestCrash() {
  throw Exception('Test crash');
}

// Test async error
Future<void> triggerAsyncCrash() async {
  await Future.delayed(const Duration(milliseconds: 100));
  throw Exception('Test async crash');
}

// Test message
void sendTestEvent() {
  Sentry.captureMessage('Test event from Flutter');
}

// Verify initialization in tests
void main() {
  setUp(() async {
    await Sentry.init(
      (options) => options.dsn = 'test-dsn',
    );
  });

  test('error reporting works', () async {
    expect(
      () => Sentry.captureException(Exception('test')),
      returnsNormally,
    );
  });
}
```

---

## Reusable Templates

- **[templates/screen-load-tracking.template.md](templates/screen-load-tracking.template.md)** - Flutter screen load tracking with Timeline API
- **[templates/error-boundary.template.md](templates/error-boundary.template.md)** - Flutter error zones and handlers
- **[templates/breadcrumb-manager.template.md](templates/breadcrumb-manager.template.md)** - Flutter breadcrumb collection
- **[templates/navigation-tracking.template.md](templates/navigation-tracking.template.md)** - Flutter navigation latency measurement

## Sources

- [Sentry Flutter SDK](https://docs.sentry.io/platforms/flutter/)
- [Datadog Flutter RUM](https://docs.datadoghq.com/real_user_monitoring/flutter/)
- [Flutter Error Handling](https://docs.flutter.dev/testing/errors)
- [Flutter Crash Reporting Best Practices](https://uxcam.com/blog/flutter-crash-reporting/)
