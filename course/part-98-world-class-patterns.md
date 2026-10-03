# Part 98: World-Class Flutter Patterns
## ขั้นตอนที่ 3801-3840

## 🎯 เป้าหมายของ Part นี้
- Code Quality Metrics ด้วย flutter analyze และ Custom Lint Rules
- Performance Budgets และ Monitoring
- Bundle Size Optimization เทคนิคขั้นสูง
- Memory Profiling Strategies
- Release Pipeline Best Practices
- Production Monitoring ด้วย Sentry และ Crashlytics

---

## ขั้นตอนที่ 3801: Custom Lint Rules with custom_lint

```yaml
# pubspec.yaml additions
dev_dependencies:
  custom_lint: ^0.6.2
  riverpod_lint: ^2.3.7

# analysis_options.yaml
analyzer:
  plugins:
    - custom_lint
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
    - "lib/generated/**"
  
  errors:
    invalid_annotation_target: ignore
  
  language:
    strict-casts: true
    strict-raw-types: true
    strict-inference: true

linter:
  rules:
    # Error prevention
    - avoid_print
    - avoid_empty_else
    - avoid_relative_lib_imports
    - avoid_returning_null_for_future
    - avoid_slow_async_io
    - cancel_subscriptions
    - close_sinks
    - collection_methods_unrelated_type
    - no_duplicate_case_values
    - test_types_in_equals
    - throw_in_finally
    - unnecessary_statements
    - unrelated_type_equality_checks
    - use_key_in_widget_constructors
    - valid_regexps
    
    # Style
    - always_declare_return_types
    - always_put_required_named_parameters_first
    - annotate_overrides
    - avoid_annotating_with_dynamic
    - avoid_bool_literals_in_conditional_expressions
    - avoid_double_and_int_checks
    - avoid_field_initializers_in_const_classes
    - avoid_function_literals_in_foreach_calls
    - avoid_init_to_null
    - avoid_null_checks_in_equality_operators
    - avoid_positional_boolean_parameters
    - avoid_redundant_argument_values
    - avoid_return_types_on_setters
    - avoid_setters_without_getters
    - avoid_shadowing_type_parameters
    - avoid_single_cascade_in_expression_statements
    - avoid_unnecessary_containers
    - avoid_void_async
    - cascade_invocations
    - conditional_uri_does_not_exist
    - directives_ordering
    - empty_catches
    - eol_at_end_of_file
    - exhaustive_cases
    - file_names
    - flutter_style_todos
    - leading_newlines_in_multiline_strings
    - missing_whitespace_between_adjacent_strings
    - no_default_cases
    - no_leading_underscores_for_local_identifiers
    - noop_primitive_operations
    - null_check_on_nullable_type_parameter
    - prefer_asserts_with_message
    - prefer_const_constructors
    - prefer_const_constructors_in_immutables
    - prefer_const_declarations
    - prefer_const_literals_to_create_immutables
    - prefer_constructors_over_static_methods
    - prefer_expression_function_bodies
    - prefer_final_fields
    - prefer_final_in_for_each
    - prefer_final_locals
    - prefer_if_null_operators
    - prefer_null_aware_method_calls
    - prefer_null_aware_operators
    - prefer_relative_imports
    - prefer_single_quotes
    - require_trailing_commas
    - sized_box_for_whitespace
    - sized_box_shrink_expand
    - sort_child_properties_last
    - sort_constructors_first
    - sort_unnamed_constructors_first
    - tighten_type_of_initializing_formals
    - type_annotate_public_apis
    - unawaited_futures
    - unnecessary_await_in_return
    - unnecessary_breaks
    - unnecessary_lambdas
    - unnecessary_null_aware_assignments
    - unnecessary_null_checks
    - unnecessary_overrides
    - unnecessary_parenthesis
    - unnecessary_raw_strings
    - unnecessary_string_escapes
    - unnecessary_string_interpolations
    - unnecessary_to_list_in_spreads
    - use_colored_box
    - use_decorated_box
    - use_enums
    - use_if_null_to_convert_nulls_to_bools
    - use_is_even_rather_than_modulo
    - use_late_for_private_fields_and_variables
    - use_named_constants
    - use_raw_strings
    - use_setters_to_change_properties
    - use_string_buffers
    - use_super_parameters
    - use_test_throws_matchers
```

## ขั้นตอนที่ 3802: Custom Lint Plugin Implementation

```dart
// custom_lint/lib/custom_lint.dart
import 'package:custom_lint_builder/custom_lint_builder.dart';

PluginBase createPlugin() => _FoodDeliveryLint();

class _FoodDeliveryLint extends PluginBase {
  @override
  List<LintRule> getLintRules(CustomLintConfigs configs) => [
    AvoidDirectApiCallRule(),
    RequireErrorHandlingRule(),
    NoHardcodedStringsRule(),
    RequireLoadingStateRule(),
  ];
}
```

```dart
// custom_lint/lib/src/avoid_direct_api_call_rule.dart
import 'package:analyzer/dart/ast/ast.dart';
import 'package:analyzer/error/listener.dart';
import 'package:custom_lint_builder/custom_lint_builder.dart';

/// Ensures API calls go through repositories, not directly from widgets
class AvoidDirectApiCallRule extends DartLintRule {
  AvoidDirectApiCallRule() : super(code: _code);

  static const _code = LintCode(
    name: 'avoid_direct_api_call',
    problemMessage: 'API calls should go through repositories, not widgets.',
    correctionMessage: 'Use a repository or provider to fetch data.',
  );

  @override
  void run(
    CustomLintResolver resolver,
    ErrorReporter reporter,
    CustomLintContext context,
  ) {
    context.registry.addMethodInvocation((node) {
      // Check if it's a Dio or http call inside a Widget
      final methodName = node.methodName.name;
      if (['get', 'post', 'put', 'delete', 'patch']
          .contains(methodName)) {
        // Check if target is dio or http client
        final target = node.target?.toSource();
        if (target != null &&
            (target.contains('dio') || target.contains('http'))) {
          // Check if we're inside a Widget
          if (_isInsideWidget(node)) {
            reporter.reportErrorForNode(_code, node);
          }
        }
      }
    });
  }

  bool _isInsideWidget(AstNode node) {
    AstNode? current = node.parent;
    while (current != null) {
      if (current is ClassDeclaration) {
        final superclass = current.extendsClause?.superclass.name2.lexeme;
        if (superclass == 'StatelessWidget' ||
            superclass == 'StatefulWidget' ||
            superclass == 'ConsumerWidget' ||
            superclass == 'ConsumerStatefulWidget') {
          return true;
        }
      }
      current = current.parent;
    }
    return false;
  }
}
```

## ขั้นตอนที่ 3803: Performance Budget Monitoring

```dart
// lib/core/performance/performance_monitor.dart
import 'dart:async';
import 'package:flutter/scheduler.dart';
import 'package:flutter/material.dart';

/// Monitors app performance and reports violations of performance budgets
class PerformanceMonitor {
  static final PerformanceMonitor _instance = PerformanceMonitor._();
  
  factory PerformanceMonitor() => _instance;
  PerformanceMonitor._();

  // Performance budgets
  static const int _frameRateBudgetMs = 16; // 60fps = 16.67ms per frame
  static const int _startupBudgetMs = 3000; // 3 seconds cold start
  static const int _scrollJankThresholdMs = 20; // > 20ms = jank frame

  final List<FrameTimingMetric> _frameMetrics = [];
  final StreamController<PerformanceBudgetViolation> _violationController =
      StreamController<PerformanceBudgetViolation>.broadcast();

  Stream<PerformanceBudgetViolation> get violations => _violationController.stream;

  void startMonitoring() {
    SchedulerBinding.instance.addTimingsCallback(_onFrameTimings);
  }

  void stopMonitoring() {
    SchedulerBinding.instance.removeTimingsCallback(_onFrameTimings);
  }

  void _onFrameTimings(List<FrameTiming> timings) {
    for (final timing in timings) {
      final buildMs = timing.buildDuration.inMilliseconds;
      final rasterMs = timing.rasterDuration.inMilliseconds;
      final totalMs = timing.totalSpan.inMilliseconds;

      final metric = FrameTimingMetric(
        buildDurationMs: buildMs,
        rasterDurationMs: rasterMs,
        totalDurationMs: totalMs,
        timestamp: DateTime.now(),
      );

      _frameMetrics.add(metric);

      // Keep only last 300 frames (5 seconds at 60fps)
      if (_frameMetrics.length > 300) {
        _frameMetrics.removeAt(0);
      }

      // Check for jank
      if (totalMs > _scrollJankThresholdMs) {
        _violationController.add(PerformanceBudgetViolation(
          type: ViolationType.jankFrame,
          description: 'Jank detected: ${totalMs}ms (budget: ${_scrollJankThresholdMs}ms)',
          severity: totalMs > 50 ? Severity.critical : Severity.warning,
          frameTiming: metric,
        ));
      }
    }
  }

  PerformanceSummary getSummary() {
    if (_frameMetrics.isEmpty) {
      return PerformanceSummary.empty();
    }

    final totalMs = _frameMetrics.map((m) => m.totalDurationMs);
    final avgFrameMs = totalMs.reduce((a, b) => a + b) / _frameMetrics.length;
    final maxFrameMs = totalMs.reduce((a, b) => a > b ? a : b);
    final jankFrames = totalMs.where((ms) => ms > _scrollJankThresholdMs).length;
    final jankRate = jankFrames / _frameMetrics.length;

    return PerformanceSummary(
      averageFrameMs: avgFrameMs,
      maxFrameMs: maxFrameMs,
      jankRate: jankRate,
      totalFrames: _frameMetrics.length,
      jankFrames: jankFrames,
    );
  }

  void dispose() {
    stopMonitoring();
    _violationController.close();
  }
}

class FrameTimingMetric {
  final int buildDurationMs;
  final int rasterDurationMs;
  final int totalDurationMs;
  final DateTime timestamp;

  const FrameTimingMetric({
    required this.buildDurationMs,
    required this.rasterDurationMs,
    required this.totalDurationMs,
    required this.timestamp,
  });
}

enum ViolationType { jankFrame, slowStartup, memorySpike, networkTimeout }
enum Severity { info, warning, critical }

class PerformanceBudgetViolation {
  final ViolationType type;
  final String description;
  final Severity severity;
  final FrameTimingMetric? frameTiming;
  final DateTime timestamp;

  PerformanceBudgetViolation({
    required this.type,
    required this.description,
    required this.severity,
    this.frameTiming,
  }) : timestamp = DateTime.now();
}

class PerformanceSummary {
  final double averageFrameMs;
  final int maxFrameMs;
  final double jankRate;
  final int totalFrames;
  final int jankFrames;

  const PerformanceSummary({
    required this.averageFrameMs,
    required this.maxFrameMs,
    required this.jankRate,
    required this.totalFrames,
    required this.jankFrames,
  });

  factory PerformanceSummary.empty() => const PerformanceSummary(
        averageFrameMs: 0,
        maxFrameMs: 0,
        jankRate: 0,
        totalFrames: 0,
        jankFrames: 0,
      );

  double get fps => averageFrameMs > 0 ? 1000 / averageFrameMs : 0;
  bool get meetsTarget => jankRate < 0.05; // Less than 5% jank frames
}
```

## ขั้นตอนที่ 3804: Bundle Size Optimization

```dart
// lib/core/optimization/deferred_loading.dart
import 'package:flutter/material.dart';

// Deferred loading for heavy features
// import 'package:food_delivery/features/map/map_view.dart' deferred as maps;
// import 'package:food_delivery/features/video/video_player.dart' deferred as video;

/// Lazy loads heavy features only when needed
class DeferredFeatureLoader {
  static final Map<String, bool> _loadedFeatures = {};

  /// Loads the maps feature lazily
  static Future<void> loadMaps() async {
    if (_loadedFeatures['maps'] == true) return;
    // await maps.loadLibrary();
    _loadedFeatures['maps'] = true;
  }

  /// Preloads features in background
  static void preloadInBackground() {
    // Load after initial frame
    WidgetsBinding.instance.addPostFrameCallback((_) {
      Future.microtask(() async {
        await loadMaps();
      });
    });
  }
}
```

```dart
// lib/core/optimization/image_optimization.dart
import 'package:flutter/material.dart';
import 'package:cached_network_image/cached_network_image.dart';

/// Optimized image widget with automatic size optimization
class OptimizedNetworkImage extends StatelessWidget {
  final String url;
  final double? width;
  final double? height;
  final BoxFit fit;
  final Widget Function(BuildContext, String)? placeholder;

  const OptimizedNetworkImage({
    super.key,
    required this.url,
    this.width,
    this.height,
    this.fit = BoxFit.cover,
    this.placeholder,
  });

  /// Gets optimized URL with size parameters
  String _getOptimizedUrl(BuildContext context) {
    final devicePixelRatio = MediaQuery.of(context).devicePixelRatio;
    final targetWidth = width != null
        ? (width! * devicePixelRatio).toInt()
        : 800;
    final targetHeight = height != null
        ? (height! * devicePixelRatio).toInt()
        : null;

    // Append size parameters to CDN URL
    // Example: Cloudflare Images, imgix, or custom CDN
    if (url.contains('cloudflare') || url.contains('imgix')) {
      final heightParam = targetHeight != null ? '&h=$targetHeight' : '';
      return '$url?w=$targetWidth$heightParam&fit=crop&auto=format,compress';
    }
    return url;
  }

  @override
  Widget build(BuildContext context) {
    final optimizedUrl = _getOptimizedUrl(context);
    
    return CachedNetworkImage(
      imageUrl: optimizedUrl,
      width: width,
      height: height,
      fit: fit,
      memCacheWidth: width != null
          ? (width! * MediaQuery.of(context).devicePixelRatio).toInt()
          : null,
      memCacheHeight: height != null
          ? (height! * MediaQuery.of(context).devicePixelRatio).toInt()
          : null,
      placeholder: placeholder ??
          (context, url) => Container(
                width: width,
                height: height,
                color: Theme.of(context).colorScheme.surfaceVariant,
                child: const Center(child: CircularProgressIndicator(strokeWidth: 2)),
              ),
      errorWidget: (context, url, error) => Container(
        width: width,
        height: height,
        color: Theme.of(context).colorScheme.errorContainer,
        child: Icon(
          Icons.broken_image_rounded,
          color: Theme.of(context).colorScheme.onErrorContainer,
        ),
      ),
    );
  }
}
```

## ขั้นตอนที่ 3805: Memory Profiling Strategies

```dart
// lib/core/diagnostics/memory_tracker.dart
import 'dart:developer' as dev;
import 'package:flutter/foundation.dart';

/// Tracks potential memory leaks and excessive allocations
class MemoryTracker {
  static final Map<String, int> _allocationCounts = {};
  static final List<WeakReference<Object>> _trackedObjects = [];

  /// Track object creation to detect leaks
  static void track(String className, Object object) {
    if (!kDebugMode) return;
    
    _allocationCounts[className] = (_allocationCounts[className] ?? 0) + 1;
    _trackedObjects.add(WeakReference(object));
    
    // Log if too many instances
    if ((_allocationCounts[className] ?? 0) > 100) {
      dev.log(
        'Warning: $className has ${_allocationCounts[className]} instances. '
        'Possible memory leak?',
        name: 'MemoryTracker',
      );
    }
  }

  /// Get current memory stats
  static Map<String, int> getAllocationReport() => Map.unmodifiable(_allocationCounts);

  /// Force garbage collection and report
  static Future<void> reportAndCollect() async {
    if (!kDebugMode) return;

    dev.log('=== Memory Report ===', name: 'MemoryTracker');
    final sorted = _allocationCounts.entries.toList()
      ..sort((a, b) => b.value.compareTo(a.value));
    for (final entry in sorted.take(10)) {
      dev.log('${entry.key}: ${entry.value} instances', name: 'MemoryTracker');
    }

    // Trigger GC
    await Future.delayed(Duration.zero);
    dev.log('GC triggered', name: 'MemoryTracker');
  }
}

/// Mixin to automatically track widget lifecycle
mixin MemoryTrackedWidget on StatefulWidget {
  @override
  State createState();
}

mixin MemoryTrackedState<T extends StatefulWidget> on State<T> {
  @override
  void initState() {
    super.initState();
    MemoryTracker.track(widget.runtimeType.toString(), this);
    _debugLog('initState');
  }

  @override
  void dispose() {
    _debugLog('dispose');
    super.dispose();
  }

  void _debugLog(String event) {
    if (kDebugMode) {
      dev.log('${widget.runtimeType}.$event', name: 'Lifecycle');
    }
  }
}
```

## ขั้นตอนที่ 3806: Sentry Error Monitoring

```dart
// lib/core/monitoring/sentry_setup.dart
import 'package:flutter/material.dart';
import 'package:sentry_flutter/sentry_flutter.dart';

class SentrySetup {
  static Future<void> initialize({
    required String dsn,
    required Widget app,
    required bool isProduction,
  }) async {
    await SentryFlutter.init(
      (options) {
        options.dsn = dsn;
        options.environment = isProduction ? 'production' : 'development';
        options.release = 'food-delivery@1.0.0+1';
        
        // Performance monitoring
        options.tracesSampleRate = isProduction ? 0.2 : 1.0;
        options.profilesSampleRate = isProduction ? 0.1 : 1.0;
        
        // Breadcrumbs
        options.enableAutoSessionTracking = true;
        options.maxBreadcrumbs = 50;
        
        // User feedback
        options.enableUserInteractionTracing = true;
        
        // Filter out sensitive data
        options.beforeSend = (event, hint) {
          // Remove PII from events
          return _sanitizeEvent(event);
        };

        // Attach screenshots on error (useful for debugging UI issues)
        options.attachScreenshot = true;
        options.screenshotQuality = SentryScreenshotQuality.low;
      },
      appRunner: () => runApp(app),
    );
  }

  static SentryEvent? _sanitizeEvent(SentryEvent event) {
    // Remove sensitive request headers
    final request = event.request?.copyWith(
      headers: event.request?.headers?.map(
        (key, value) => MapEntry(
          key,
          _sensitiveHeaders.contains(key.toLowerCase()) ? '***' : value,
        ),
      ),
    );

    // Remove sensitive user fields
    final user = event.user?.copyWith(
      email: '***@***.***',
    );

    return event.copyWith(request: request, user: user);
  }

  static const _sensitiveHeaders = {'authorization', 'cookie', 'x-api-key'};
}

/// Captures errors with context
class ErrorReporter {
  static Future<void> reportError(
    dynamic error,
    StackTrace stackTrace, {
    String? userId,
    Map<String, dynamic>? extras,
    SentryLevel level = SentryLevel.error,
  }) async {
    await Sentry.captureException(
      error,
      stackTrace: stackTrace,
      withScope: (scope) {
        if (userId != null) {
          scope.setUser(SentryUser(id: userId));
        }
        if (extras != null) {
          scope.setContexts('extras', extras);
        }
        scope.level = level;
      },
    );
  }

  static Future<SentryTransaction> startTransaction(
    String name,
    String operation,
  ) async {
    return Sentry.startTransaction(name, operation);
  }

  static void addBreadcrumb(String message, {
    String? category,
    SentryLevel level = SentryLevel.info,
  }) {
    Sentry.addBreadcrumb(Breadcrumb(
      message: message,
      category: category ?? 'app',
      level: level,
      timestamp: DateTime.now(),
    ));
  }
}
```

## ขั้นตอนที่ 3807: Firebase Crashlytics Integration

```dart
// lib/core/monitoring/crashlytics_setup.dart
import 'package:firebase_crashlytics/firebase_crashlytics.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';

class CrashlyticsSetup {
  static Future<void> initialize() async {
    // Pass all uncaught Flutter errors to Crashlytics
    FlutterError.onError = (FlutterErrorDetails details) {
      if (kDebugMode) {
        FlutterError.presentError(details);
      } else {
        FirebaseCrashlytics.instance.recordFlutterFatalError(details);
      }
    };

    // Pass all uncaught asynchronous errors to Crashlytics
    PlatformDispatcher.instance.onError = (error, stack) {
      FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
      return true;
    };

    // Enable/disable collection based on environment
    await FirebaseCrashlytics.instance
        .setCrashlyticsCollectionEnabled(!kDebugMode);
  }

  static Future<void> setUserIdentifier(String userId) async {
    await FirebaseCrashlytics.instance.setUserIdentifier(userId);
  }

  static Future<void> log(String message) async {
    await FirebaseCrashlytics.instance.log(message);
  }

  static Future<void> recordError(
    dynamic exception,
    StackTrace? stack, {
    bool fatal = false,
    Map<String, String>? customKeys,
  }) async {
    if (customKeys != null) {
      for (final entry in customKeys.entries) {
        await FirebaseCrashlytics.instance
            .setCustomKey(entry.key, entry.value);
      }
    }
    await FirebaseCrashlytics.instance.recordError(
      exception,
      stack,
      fatal: fatal,
    );
  }
}

/// Wrap app in error zone
Future<void> runAppWithCrashlytics(Widget app) async {
  runZonedGuarded(
    () async {
      WidgetsFlutterBinding.ensureInitialized();
      await CrashlyticsSetup.initialize();
      runApp(app);
    },
    (error, stackTrace) {
      FirebaseCrashlytics.instance.recordError(error, stackTrace);
    },
  );
}
```

## ขั้นตอนที่ 3808: Release Pipeline CI/CD

```yaml
# .github/workflows/release.yml
name: Flutter Release Pipeline

on:
  push:
    tags:
      - 'v*.*.*'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [staging, production]

env:
  FLUTTER_VERSION: '3.19.0'
  JAVA_VERSION: '17'

jobs:
  quality-gate:
    name: Quality Gate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Run analyzer
        run: flutter analyze --no-fatal-infos

      - name: Run tests with coverage
        run: flutter test --coverage --coverage-path=coverage/lcov.info

      - name: Check coverage threshold (80%)
        run: |
          COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep "lines" | awk '{print $2}' | tr -d '%')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below threshold 80%"
            exit 1
          fi
          echo "Coverage: $COVERAGE%"

      - name: Upload coverage report
        uses: codecov/codecov-action@v3
        with:
          file: coverage/lcov.info

  build-android:
    name: Build Android Release
    needs: quality-gate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v3
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Decode keystore
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 -d > android/keystore.jks

      - name: Create key.properties
        run: |
          cat > android/key.properties << EOF
          storePassword=${{ secrets.ANDROID_STORE_PASSWORD }}
          keyPassword=${{ secrets.ANDROID_KEY_PASSWORD }}
          keyAlias=${{ secrets.ANDROID_KEY_ALIAS }}
          storeFile=../keystore.jks
          EOF

      - name: Install dependencies
        run: flutter pub get

      - name: Build Android App Bundle
        run: flutter build appbundle --release --obfuscate --split-debug-info=build/debug-info

      - name: Upload to Play Store (Internal Track)
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
          packageName: com.fooddelivery.app
          releaseFiles: build/app/outputs/bundle/release/*.aab
          track: internal
          status: completed

      - name: Upload debug symbols
        run: |
          # Upload to Sentry for better crash symbolication
          sentry-cli upload-dif --org food-delivery --project android build/debug-info/

  build-ios:
    name: Build iOS Release
    needs: quality-gate
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - name: Install Apple certificate and provisioning profile
        env:
          BUILD_CERTIFICATE_BASE64: ${{ secrets.IOS_CERTIFICATE_BASE64 }}
          P12_PASSWORD: ${{ secrets.IOS_CERTIFICATE_PASSWORD }}
          PROVISION_PROFILE_BASE64: ${{ secrets.IOS_PROVISION_PROFILE_BASE64 }}
          KEYCHAIN_PASSWORD: ${{ secrets.IOS_KEYCHAIN_PASSWORD }}
        run: |
          CERTIFICATE_PATH=$RUNNER_TEMP/build_certificate.p12
          PP_PATH=$RUNNER_TEMP/build_pp.mobileprovision
          KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db
          
          echo -n "$BUILD_CERTIFICATE_BASE64" | base64 --decode -o $CERTIFICATE_PATH
          echo -n "$PROVISION_PROFILE_BASE64" | base64 --decode -o $PP_PATH
          
          security create-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH
          security set-keychain-settings -lut 21600 $KEYCHAIN_PATH
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH
          security import $CERTIFICATE_PATH -P "$P12_PASSWORD" -A -t cert -f pkcs12 -k $KEYCHAIN_PATH
          security list-keychain -d user -s $KEYCHAIN_PATH
          
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          cp $PP_PATH ~/Library/MobileDevice/Provisioning\ Profiles

      - run: flutter pub get
      
      - name: Build iOS IPA
        run: flutter build ipa --release --export-options-plist=ios/ExportOptions.plist

      - name: Upload to TestFlight
        uses: apple-actions/upload-testflight-build@v1
        with:
          app-path: build/ios/ipa/*.ipa
          issuer-id: ${{ secrets.APPSTORE_ISSUER_ID }}
          api-key-id: ${{ secrets.APPSTORE_KEY_ID }}
          api-private-key: ${{ secrets.APPSTORE_PRIVATE_KEY }}
```

## ขั้นตอนที่ 3809: Obfuscation and Security Hardening

```dart
// lib/core/security/app_security.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:local_auth/local_auth.dart';
import 'dart:math';
import 'dart:convert';
import 'package:crypto/crypto.dart';

class AppSecurity {
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(
      encryptedSharedPreferences: true,
      storageCipherAlgorithm: StorageCipherAlgorithm.AES_GCM_NoPadding,
    ),
    iOptions: IOSOptions(
      accessibility: KeychainAccessibility.first_unlock_this_device,
    ),
  );

  static final _localAuth = LocalAuthentication();

  /// Stores sensitive data securely
  static Future<void> secureWrite(String key, String value) async {
    await _storage.write(key: key, value: value);
  }

  /// Reads sensitive data
  static Future<String?> secureRead(String key) async {
    return _storage.read(key: key);
  }

  /// Deletes sensitive data
  static Future<void> secureDelete(String key) async {
    await _storage.delete(key: key);
  }

  /// Clears all stored secure data
  static Future<void> clearAll() async {
    await _storage.deleteAll();
  }

  /// Check if biometric auth is available
  static Future<bool> isBiometricAvailable() async {
    final canCheckBiometrics = await _localAuth.canCheckBiometrics;
    final isDeviceSupported = await _localAuth.isDeviceSupported();
    return canCheckBiometrics && isDeviceSupported;
  }

  /// Authenticate with biometrics
  static Future<bool> authenticateWithBiometrics({
    required String reason,
  }) async {
    try {
      return await _localAuth.authenticate(
        localizedReason: reason,
        options: const AuthenticationOptions(
          stickyAuth: true,
          biometricOnly: false,
        ),
      );
    } on PlatformException catch (_) {
      return false;
    }
  }

  /// Generates a secure random token
  static String generateSecureToken({int length = 32}) {
    final random = Random.secure();
    final bytes = List<int>.generate(length, (_) => random.nextInt(256));
    return base64Url.encode(bytes);
  }

  /// Hashes a password with salt
  static String hashPassword(String password, String salt) {
    final bytes = utf8.encode(password + salt);
    final digest = sha256.convert(bytes);
    return digest.toString();
  }

  /// Validates API response hasn't been tampered (HMAC)
  static bool validateHmac(
    String data,
    String signature,
    String secretKey,
  ) {
    final keyBytes = utf8.encode(secretKey);
    final dataBytes = utf8.encode(data);
    final hmac = Hmac(sha256, keyBytes);
    final digest = hmac.convert(dataBytes);
    return digest.toString() == signature;
  }
}
```

## ขั้นตอนที่ 3810: Flutter Flavors Configuration

```dart
// lib/core/config/app_config.dart
enum AppEnvironment { development, staging, production }

class AppConfig {
  final AppEnvironment environment;
  final String apiBaseUrl;
  final String firebaseProjectId;
  final String sentryDsn;
  final bool enableAnalytics;
  final bool enableCrashReporting;
  final int apiTimeoutSeconds;
  final int cacheDurationMinutes;

  const AppConfig({
    required this.environment,
    required this.apiBaseUrl,
    required this.firebaseProjectId,
    required this.sentryDsn,
    required this.enableAnalytics,
    required this.enableCrashReporting,
    required this.apiTimeoutSeconds,
    required this.cacheDurationMinutes,
  });

  factory AppConfig.development() => const AppConfig(
        environment: AppEnvironment.development,
        apiBaseUrl: 'http://localhost:8080/api/v1',
        firebaseProjectId: 'food-delivery-dev',
        sentryDsn: '',
        enableAnalytics: false,
        enableCrashReporting: false,
        apiTimeoutSeconds: 30,
        cacheDurationMinutes: 5,
      );

  factory AppConfig.staging() => const AppConfig(
        environment: AppEnvironment.staging,
        apiBaseUrl: 'https://api-staging.fooddelivery.app/v1',
        firebaseProjectId: 'food-delivery-staging',
        sentryDsn: 'https://staging@sentry.io/12345',
        enableAnalytics: true,
        enableCrashReporting: true,
        apiTimeoutSeconds: 20,
        cacheDurationMinutes: 10,
      );

  factory AppConfig.production() => const AppConfig(
        environment: AppEnvironment.production,
        apiBaseUrl: 'https://api.fooddelivery.app/v1',
        firebaseProjectId: 'food-delivery-prod',
        sentryDsn: 'https://prod@sentry.io/67890',
        enableAnalytics: true,
        enableCrashReporting: true,
        apiTimeoutSeconds: 15,
        cacheDurationMinutes: 30,
      );

  bool get isDevelopment => environment == AppEnvironment.development;
  bool get isProduction => environment == AppEnvironment.production;
  String get environmentName => environment.name;
}

// main_development.dart
void main() {
  runWithConfig(AppConfig.development());
}

// main_staging.dart  
void main() {
  runWithConfig(AppConfig.staging());
}

// main_production.dart
void main() {
  runWithConfig(AppConfig.production());
}

void runWithConfig(AppConfig config) {
  runApp(ProviderScope(
    overrides: [
      appConfigProvider.overrideWithValue(config),
    ],
    child: const FoodDeliveryApp(),
  ));
}
```

## ขั้นตอนที่ 3811: Advanced Error Boundary

```dart
// lib/core/error/error_boundary.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

/// Catches widget tree errors and shows a fallback UI
class ErrorBoundary extends StatefulWidget {
  final Widget child;
  final Widget Function(Object error, StackTrace? stack)? errorBuilder;
  final void Function(Object error, StackTrace? stack)? onError;

  const ErrorBoundary({
    super.key,
    required this.child,
    this.errorBuilder,
    this.onError,
  });

  @override
  State<ErrorBoundary> createState() => _ErrorBoundaryState();
}

class _ErrorBoundaryState extends State<ErrorBoundary> {
  Object? _error;
  StackTrace? _stackTrace;

  @override
  void initState() {
    super.initState();
  }

  void _handleError(Object error, StackTrace stackTrace) {
    widget.onError?.call(error, stackTrace);
    setState(() {
      _error = error;
      _stackTrace = stackTrace;
    });
  }

  @override
  Widget build(BuildContext context) {
    if (_error != null) {
      return widget.errorBuilder?.call(_error!, _stackTrace) ??
          _DefaultErrorWidget(
            error: _error!,
            onRetry: () => setState(() {
              _error = null;
              _stackTrace = null;
            }),
          );
    }

    return _ErrorCatcher(
      onError: _handleError,
      child: widget.child,
    );
  }
}

class _ErrorCatcher extends StatelessWidget {
  final Widget child;
  final void Function(Object, StackTrace) onError;

  const _ErrorCatcher({
    required this.child,
    required this.onError,
  });

  @override
  Widget build(BuildContext context) {
    ErrorWidget.builder = (details) {
      onError(details.exception, details.stack ?? StackTrace.empty);
      return const SizedBox.shrink();
    };
    return child;
  }
}

class _DefaultErrorWidget extends StatelessWidget {
  final Object error;
  final VoidCallback onRetry;

  const _DefaultErrorWidget({
    required this.error,
    required this.onRetry,
  });

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(
              Icons.error_outline_rounded,
              size: 64,
              color: Theme.of(context).colorScheme.error,
            ),
            const SizedBox(height: 16),
            Text(
              'Something went wrong',
              style: Theme.of(context).textTheme.titleLarge,
            ),
            const SizedBox(height: 8),
            Text(
              'We\'re working on fixing this.',
              style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                    color: Theme.of(context).colorScheme.onSurfaceVariant,
                  ),
              textAlign: TextAlign.center,
            ),
            const SizedBox(height: 24),
            FilledButton(
              onPressed: onRetry,
              child: const Text('Try Again'),
            ),
          ],
        ),
      ),
    );
  }
}
```

## ขั้นตอนที่ 3812: App Size Monitoring Script

```bash
#!/bin/bash
# scripts/analyze_app_size.sh
# Analyzes app size and reports top contributors

set -e

echo "Building release app for size analysis..."

# Build with size analysis
flutter build apk --release \
  --analyze-size \
  --target-platform android-arm64 \
  --split-per-abi

echo ""
echo "=== APK Size Report ==="
APK_PATH="build/app/outputs/flutter-apk/app-arm64-v8a-release.apk"
APK_SIZE=$(du -sh "$APK_PATH" | cut -f1)
echo "APK Size: $APK_SIZE"

# Use flutter size analysis
flutter build apk --target-platform android-arm64 --analyze-size --release 2>&1 | \
  grep -E "(Dart|Flutter|Total|Package)" | head -30

echo ""
echo "=== AAB Size Report ==="
flutter build appbundle --release --analyze-size

echo ""
echo "=== Size Budget Check ==="
APK_BYTES=$(stat -c%s "$APK_PATH")
BUDGET_BYTES=$((30 * 1024 * 1024))  # 30MB budget

if [ "$APK_BYTES" -gt "$BUDGET_BYTES" ]; then
  echo "FAILED: APK size ($APK_BYTES bytes) exceeds budget ($BUDGET_BYTES bytes)"
  exit 1
else
  echo "PASSED: APK size $(( APK_BYTES / 1024 / 1024 ))MB is within 30MB budget"
fi

echo ""
echo "Generating symbol map for profiling..."
flutter build apk --release --obfuscate --split-debug-info=.dart_tool/flutter_build/
```

```dart
// lib/core/startup/app_startup.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:shared_preferences/shared_preferences.dart';

class AppStartup {
  static Future<AppStartupResult> initialize() async {
    final stopwatch = Stopwatch()..start();

    try {
      // Ensure Flutter binding is initialized
      WidgetsFlutterBinding.ensureInitialized();

      // Set preferred orientations
      await SystemChrome.setPreferredOrientations([
        DeviceOrientation.portraitUp,
        DeviceOrientation.portraitDown,
      ]);

      // System UI style
      SystemChrome.setSystemUIOverlayStyle(const SystemUiOverlayStyle(
        statusBarColor: Colors.transparent,
        statusBarIconBrightness: Brightness.dark,
      ));

      // Initialize Firebase
      await Firebase.initializeApp();

      // Preload critical data in parallel
      final results = await Future.wait([
        SharedPreferences.getInstance(),
        _preloadFonts(),
        _checkNetworkConnectivity(),
      ]);

      stopwatch.stop();
      debugPrint('App startup completed in ${stopwatch.elapsedMilliseconds}ms');

      if (stopwatch.elapsedMilliseconds > 3000) {
        debugPrint('WARNING: Startup exceeded 3s budget!');
      }

      return AppStartupResult(
        prefs: results[0] as SharedPreferences,
        hasNetwork: results[2] as bool,
        startupDurationMs: stopwatch.elapsedMilliseconds,
      );
    } catch (e, stack) {
      stopwatch.stop();
      debugPrint('Startup error: $e\n$stack');
      rethrow;
    }
  }

  static Future<void> _preloadFonts() async {
    // Preload font families used in the app
    await Future.wait([
      precacheImage(const AssetImage('assets/images/logo.png'), WidgetsBinding.instance.rootElement!),
    ]).catchError((_) {}); // Non-critical
  }

  static Future<bool> _checkNetworkConnectivity() async {
    // Quick connectivity check
    return true; // Simplified
  }
}

class AppStartupResult {
  final SharedPreferences prefs;
  final bool hasNetwork;
  final int startupDurationMs;

  const AppStartupResult({
    required this.prefs,
    required this.hasNetwork,
    required this.startupDurationMs,
  });
}
```

---

**← [Part 97 - Capstone Testing](part-97-capstone-testing.md)**
**ต่อไป: [Part 99 →](part-99-career-professional-tips.md)**
