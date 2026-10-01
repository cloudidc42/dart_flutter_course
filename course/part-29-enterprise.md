# Part 29: Enterprise Architecture
## ขั้นตอนที่ 1041-1080

---

## 🎯 เป้าหมายของ Part นี้

- Modular architecture (multi-package)
- Micro-frontend patterns
- Plugin architecture
- Cross-cutting concerns
- Monitoring & Observability

---

## ขั้นตอนที่ 1041: Modular Package Structure

```
# Enterprise Flutter Project Structure
packages/
├── core/                   # Shared utilities, base classes
│   ├── lib/
│   │   ├── src/
│   │   │   ├── network/
│   │   │   ├── storage/
│   │   │   ├── logging/
│   │   │   └── analytics/
│   │   └── core.dart
│   └── pubspec.yaml
│
├── design_system/          # UI components
│   ├── lib/
│   │   ├── src/
│   │   │   ├── tokens/     # colors, typography, spacing
│   │   │   ├── components/
│   │   │   └── themes/
│   │   └── design_system.dart
│   └── pubspec.yaml
│
├── features/
│   ├── auth/               # Authentication feature
│   │   ├── lib/
│   │   │   ├── src/
│   │   │   │   ├── domain/
│   │   │   │   ├── data/
│   │   │   │   └── presentation/
│   │   │   └── auth.dart
│   │   └── pubspec.yaml
│   │
│   ├── shop/               # E-commerce feature
│   │   └── ...
│   │
│   └── profile/            # User profile feature
│       └── ...
│
└── app/                    # Main app shell
    ├── lib/
    │   └── main.dart
    └── pubspec.yaml
```

---

## ขั้นตอนที่ 1042: Plugin Architecture

```dart
// ─── Plugin Interface ───
abstract class AppPlugin {
  String get id;
  String get name;
  String get version;

  Future<void> initialize(PluginContext context);
  Future<void> onStart();
  Future<void> onStop();
  List<Route> get routes;
  Map<String, dynamic>? get config;
}

class PluginContext {
  final EventBus eventBus;
  final LogService logger;
  final AnalyticsService analytics;
  final NavigationService navigation;

  const PluginContext({
    required this.eventBus,
    required this.logger,
    required this.analytics,
    required this.navigation,
  });
}

// ─── Plugin Registry ───
class PluginRegistry {
  static final PluginRegistry _instance = PluginRegistry._();
  factory PluginRegistry() => _instance;
  PluginRegistry._();

  final Map<String, AppPlugin> _plugins = {};
  PluginContext? _context;

  void setContext(PluginContext ctx) => _context = ctx;

  Future<void> register(AppPlugin plugin) async {
    if (_plugins.containsKey(plugin.id)) {
      throw Exception('Plugin ${plugin.id} already registered');
    }

    await plugin.initialize(_context!);
    _plugins[plugin.id] = plugin;
    print('Plugin registered: ${plugin.name} v${plugin.version}');
  }

  Future<void> unregister(String pluginId) async {
    AppPlugin? plugin = _plugins[pluginId];
    if (plugin != null) {
      await plugin.onStop();
      _plugins.remove(pluginId);
    }
  }

  AppPlugin? getPlugin(String id) => _plugins[id];

  List<AppPlugin> get allPlugins => _plugins.values.toList();

  Future<void> startAll() async {
    for (AppPlugin plugin in _plugins.values) {
      await plugin.onStart();
    }
  }

  List<Route> get allRoutes {
    return _plugins.values.expand((p) => p.routes).toList();
  }
}

// ─── Example Plugin ───
class ShopPlugin implements AppPlugin {
  @override
  String get id => 'shop';

  @override
  String get name => 'Shop Plugin';

  @override
  String get version => '1.0.0';

  late PluginContext _context;

  @override
  Future<void> initialize(PluginContext context) async {
    _context = context;
    _context.logger.info('Shop plugin initialized');
  }

  @override
  Future<void> onStart() async {
    _context.analytics.track('plugin_started', {'plugin': id});
  }

  @override
  Future<void> onStop() async {}

  @override
  List<Route> get routes => [
    // Route definitions
  ];

  @override
  Map<String, dynamic>? get config => null;
}

// Placeholder classes
class EventBus {}
class LogService {
  void info(String msg) => print('[INFO] $msg');
}
class AnalyticsService {
  void track(String event, [Map<String, dynamic>? props]) {}
}
class NavigationService {}
class Route {}
```

---

## ขั้นตอนที่ 1043: Observability Stack

```dart
// ─── Logging ───
enum LogLevel { debug, info, warning, error, critical }

class LogEntry {
  final LogLevel level;
  final String message;
  final Map<String, dynamic>? data;
  final StackTrace? stackTrace;
  final DateTime timestamp;

  LogEntry({
    required this.level,
    required this.message,
    this.data,
    this.stackTrace,
  }) : timestamp = DateTime.now();

  Map<String, dynamic> toJson() => {
    'level': level.name,
    'message': message,
    'data': data,
    'timestamp': timestamp.toIso8601String(),
  };
}

abstract class LogHandler {
  Future<void> handle(LogEntry entry);
}

class ConsoleLogHandler implements LogHandler {
  @override
  Future<void> handle(LogEntry entry) async {
    String emoji = switch (entry.level) {
      LogLevel.debug => '🔍',
      LogLevel.info => 'ℹ️',
      LogLevel.warning => '⚠️',
      LogLevel.error => '❌',
      LogLevel.critical => '🚨',
    };
    print('$emoji [${entry.level.name.toUpperCase()}] ${entry.message}');
    if (entry.data != null) print('   Data: ${entry.data}');
    if (entry.stackTrace != null) print('   Stack: ${entry.stackTrace}');
  }
}

class FirebaseLogHandler implements LogHandler {
  @override
  Future<void> handle(LogEntry entry) async {
    if (entry.level.index >= LogLevel.error.index) {
      // Firebase Crashlytics.instance.recordError(...)
      print('[Firebase] Recording error: ${entry.message}');
    }
  }
}

class Logger {
  static final Logger _instance = Logger._();
  factory Logger() => _instance;
  Logger._();

  final List<LogHandler> _handlers = [];
  LogLevel minLevel = LogLevel.debug;

  void addHandler(LogHandler handler) => _handlers.add(handler);

  Future<void> _log(LogLevel level, String message, {
    Map<String, dynamic>? data,
    StackTrace? stackTrace,
  }) async {
    if (level.index < minLevel.index) return;

    LogEntry entry = LogEntry(
      level: level,
      message: message,
      data: data,
      stackTrace: stackTrace,
    );

    for (LogHandler handler in _handlers) {
      try {
        await handler.handle(entry);
      } catch (e) {
        print('Logger handler error: $e');
      }
    }
  }

  void debug(String msg, {Map<String, dynamic>? data}) =>
      _log(LogLevel.debug, msg, data: data);
  void info(String msg, {Map<String, dynamic>? data}) =>
      _log(LogLevel.info, msg, data: data);
  void warning(String msg, {Map<String, dynamic>? data}) =>
      _log(LogLevel.warning, msg, data: data);
  void error(String msg, {Map<String, dynamic>? data, StackTrace? stackTrace}) =>
      _log(LogLevel.error, msg, data: data, stackTrace: stackTrace);
  void critical(String msg, {Map<String, dynamic>? data, StackTrace? stackTrace}) =>
      _log(LogLevel.critical, msg, data: data, stackTrace: stackTrace);
}

// ─── Analytics ───
abstract class AnalyticsProvider {
  Future<void> init();
  Future<void> track(String event, Map<String, dynamic> properties);
  Future<void> identify(String userId, Map<String, dynamic> traits);
  Future<void> page(String name, Map<String, dynamic>? properties);
}

class MultiAnalytics implements AnalyticsProvider {
  final List<AnalyticsProvider> _providers;
  MultiAnalytics(this._providers);

  @override
  Future<void> init() async {
    await Future.wait(_providers.map((p) => p.init()));
  }

  @override
  Future<void> track(String event, Map<String, dynamic> properties) async {
    await Future.wait(_providers.map((p) => p.track(event, properties)));
  }

  @override
  Future<void> identify(String userId, Map<String, dynamic> traits) async {
    await Future.wait(_providers.map((p) => p.identify(userId, traits)));
  }

  @override
  Future<void> page(String name, Map<String, dynamic>? properties) async {
    await Future.wait(_providers.map((p) => p.page(name, properties)));
  }
}

// ─── Performance Monitoring ───
class PerformanceTracker {
  final Map<String, Stopwatch> _timers = {};

  void startTimer(String name) {
    _timers[name] = Stopwatch()..start();
  }

  int? stopTimer(String name) {
    Stopwatch? sw = _timers.remove(name);
    sw?.stop();
    return sw?.elapsedMilliseconds;
  }

  Future<T> track<T>(String name, Future<T> Function() operation) async {
    startTimer(name);
    try {
      T result = await operation();
      int? duration = stopTimer(name);
      Logger().info('Performance: $name', data: {'duration_ms': duration});
      return result;
    } catch (e) {
      stopTimer(name);
      rethrow;
    }
  }
}
```

---

## ขั้นตอนที่ 1044: Resilience Patterns

```dart
import 'dart:async';

// ─── Circuit Breaker ───
enum CircuitState { closed, open, halfOpen }

class CircuitBreaker {
  final String name;
  final int failureThreshold;
  final Duration timeout;
  final Duration resetTimeout;

  CircuitState _state = CircuitState.closed;
  int _failures = 0;
  DateTime? _openedAt;

  CircuitBreaker({
    required this.name,
    this.failureThreshold = 5,
    this.timeout = const Duration(seconds: 30),
    this.resetTimeout = const Duration(seconds: 10),
  });

  CircuitState get state {
    if (_state == CircuitState.open) {
      if (_openedAt != null &&
          DateTime.now().difference(_openedAt!) > resetTimeout) {
        _state = CircuitState.halfOpen;
      }
    }
    return _state;
  }

  Future<T> execute<T>(Future<T> Function() operation) async {
    if (state == CircuitState.open) {
      throw Exception('Circuit breaker "$name" is OPEN');
    }

    try {
      T result = await operation().timeout(timeout);
      _onSuccess();
      return result;
    } catch (e) {
      _onFailure();
      rethrow;
    }
  }

  void _onSuccess() {
    _failures = 0;
    _state = CircuitState.closed;
  }

  void _onFailure() {
    _failures++;
    if (_failures >= failureThreshold || _state == CircuitState.halfOpen) {
      _state = CircuitState.open;
      _openedAt = DateTime.now();
      print('Circuit breaker "$name" OPENED after $_failures failures');
    }
  }
}

// ─── Retry with Exponential Backoff ───
class RetryPolicy {
  final int maxAttempts;
  final Duration initialDelay;
  final double backoffMultiplier;
  final Duration maxDelay;

  const RetryPolicy({
    this.maxAttempts = 3,
    this.initialDelay = const Duration(seconds: 1),
    this.backoffMultiplier = 2.0,
    this.maxDelay = const Duration(seconds: 30),
  });

  Future<T> execute<T>(Future<T> Function() operation, {
    bool Function(dynamic error)? retryIf,
  }) async {
    Duration delay = initialDelay;
    Exception? lastError;

    for (int attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        return await operation();
      } catch (e) {
        lastError = Exception(e.toString());

        bool shouldRetry = retryIf?.call(e) ?? true;
        if (!shouldRetry || attempt == maxAttempts) rethrow;

        print('Attempt $attempt failed, retrying in ${delay.inSeconds}s...');
        await Future.delayed(delay);

        delay = Duration(
          milliseconds: (delay.inMilliseconds * backoffMultiplier).toInt(),
        );
        if (delay > maxDelay) delay = maxDelay;
      }
    }

    throw lastError!;
  }
}

// ─── Bulkhead Pattern ───
class Bulkhead {
  final int maxConcurrent;
  int _current = 0;
  final _queue = StreamController<Completer<void>>();

  Bulkhead({this.maxConcurrent = 10});

  Future<T> execute<T>(Future<T> Function() operation) async {
    if (_current >= maxConcurrent) {
      Completer<void> completer = Completer<void>();
      _queue.add(completer);
      await completer.future;
    }

    _current++;
    try {
      return await operation();
    } finally {
      _current--;
    }
  }
}
```

---

**← [Part 28 - Advanced Patterns](part-28-advanced-patterns.md)**

**ต่อไป: [Part 30 - World-Class App Architecture →](part-30-world-class.md)**
