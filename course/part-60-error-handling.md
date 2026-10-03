# Part 60: Advanced Error Handling
## ขั้นตอนที่ 2281-2320

## 🎯 เป้าหมายของ Part นี้
- ใช้ Either type (fpdart) สำหรับ functional error handling
- สร้าง Global Error Boundary ด้วย ErrorWidget.builder
- ผสาน Firebase Crashlytics สำหรับ crash reporting
- สร้าง Error Reporting Service ที่ครบวงจร
- สร้างหน้า user-friendly error screens
- ใช้ Retry Mechanism พร้อม Exponential Backoff

---

## ขั้นตอนที่ 2281: Dependencies

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  fpdart: ^1.1.0
  firebase_core: ^2.24.2
  firebase_crashlytics: ^3.4.9
  dio: ^5.3.3
  flutter_riverpod: ^2.4.9
  riverpod: ^2.4.9

dev_dependencies:
  flutter_test:
    sdk: flutter
  mocktail: ^1.0.1
```

---

## ขั้นตอนที่ 2282: Domain Failure Types

```dart
// lib/core/error/failures.dart

/// Base failure class
abstract class Failure {
  final String message;
  final String? code;
  final dynamic cause;

  const Failure({
    required this.message,
    this.code,
    this.cause,
  });

  @override
  String toString() =>
      '${runtimeType}(message: $message, code: $code, cause: $cause)';
}

// --- Network Failures ---

class NetworkFailure extends Failure {
  const NetworkFailure({
    super.message = 'Network error occurred',
    super.code,
    super.cause,
  });
}

class TimeoutFailure extends Failure {
  const TimeoutFailure({
    super.message = 'Request timed out',
    super.code = 'TIMEOUT',
    super.cause,
  });
}

class NoInternetFailure extends Failure {
  const NoInternetFailure({
    super.message = 'No internet connection',
    super.code = 'NO_INTERNET',
    super.cause,
  });
}

// --- Server Failures ---

class ServerFailure extends Failure {
  final int? statusCode;

  const ServerFailure({
    super.message = 'Server error',
    super.code,
    super.cause,
    this.statusCode,
  });
}

class NotFoundFailure extends Failure {
  const NotFoundFailure({
    super.message = 'Resource not found',
    super.code = 'NOT_FOUND',
    super.cause,
  });
}

class UnauthorizedFailure extends Failure {
  const UnauthorizedFailure({
    super.message = 'Authentication required',
    super.code = 'UNAUTHORIZED',
    super.cause,
  });
}

class ForbiddenFailure extends Failure {
  const ForbiddenFailure({
    super.message = 'Access denied',
    super.code = 'FORBIDDEN',
    super.cause,
  });
}

// --- Validation Failures ---

class ValidationFailure extends Failure {
  final Map<String, List<String>> fieldErrors;

  const ValidationFailure({
    super.message = 'Validation failed',
    super.code = 'VALIDATION_ERROR',
    super.cause,
    this.fieldErrors = const {},
  });
}

// --- Cache/Local Failures ---

class CacheFailure extends Failure {
  const CacheFailure({
    super.message = 'Cache error',
    super.code = 'CACHE_ERROR',
    super.cause,
  });
}

// --- Unknown Failures ---

class UnknownFailure extends Failure {
  const UnknownFailure({
    super.message = 'An unexpected error occurred',
    super.code = 'UNKNOWN',
    super.cause,
  });
}
```

---

## ขั้นตอนที่ 2283: Either Type with fpdart

```dart
// lib/core/error/either_extensions.dart
import 'package:fpdart/fpdart.dart';
import 'failures.dart';

/// Type alias for the common Result pattern
typedef AppResult<T> = Either<Failure, T>;
typedef AppTaskResult<T> = TaskEither<Failure, T>;

/// Extensions for working with Either
extension EitherExtensions<L extends Failure, R> on Either<L, R> {
  /// Returns value if Right, or throws if Left
  R getOrThrow() => getOrElse((l) => throw Exception(l.message));

  /// Returns value if Right, or null if Left
  R? getOrNull() => fold((_) => null, (r) => r);

  /// Map failure type
  Either<Failure, R> mapLeft<T>(Failure Function(L) f) =>
      fold((l) => Left(f(l)), (r) => Right(r));

  /// Convert to nullable
  R? toNullable() => getOrNull();
}

/// Safe wrapper for async operations
Future<AppResult<T>> runCatching<T>(Future<T> Function() fn) async {
  try {
    final result = await fn();
    return Right(result);
  } on UnauthorizedFailure catch (e) {
    return Left(e);
  } on Failure catch (e) {
    return Left(e);
  } catch (e, st) {
    return Left(UnknownFailure(message: e.toString(), cause: st));
  }
}
```

---

## ขั้นตอนที่ 2284: DIO Exception Mapper

```dart
// lib/core/network/dio_error_mapper.dart
import 'package:dio/dio.dart';
import '../error/failures.dart';

class DioErrorMapper {
  static Failure map(DioException e) {
    switch (e.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.sendTimeout:
      case DioExceptionType.receiveTimeout:
        return TimeoutFailure(cause: e);

      case DioExceptionType.connectionError:
        return NoInternetFailure(cause: e);

      case DioExceptionType.badResponse:
        return _mapStatusCode(e.response?.statusCode, e);

      case DioExceptionType.cancel:
        return NetworkFailure(
          message: 'Request cancelled',
          code: 'CANCELLED',
          cause: e,
        );

      case DioExceptionType.unknown:
      default:
        if (e.error is FormatException) {
          return ServerFailure(
            message: 'Invalid response format',
            code: 'FORMAT_ERROR',
            cause: e,
          );
        }
        return UnknownFailure(message: e.message ?? 'Unknown error', cause: e);
    }
  }

  static Failure _mapStatusCode(int? statusCode, DioException e) {
    switch (statusCode) {
      case 400:
        final data = e.response?.data;
        if (data is Map<String, dynamic> && data.containsKey('errors')) {
          return ValidationFailure(
            message: data['message'] ?? 'Validation failed',
            cause: e,
            fieldErrors: _parseFieldErrors(data['errors']),
          );
        }
        return ServerFailure(
          message: 'Bad request',
          code: 'BAD_REQUEST',
          statusCode: 400,
          cause: e,
        );
      case 401:
        return UnauthorizedFailure(cause: e);
      case 403:
        return ForbiddenFailure(cause: e);
      case 404:
        return NotFoundFailure(cause: e);
      case 422:
        return ValidationFailure(
          message: 'Unprocessable entity',
          cause: e,
        );
      case 429:
        return ServerFailure(
          message: 'Too many requests. Please try again later.',
          code: 'RATE_LIMITED',
          statusCode: 429,
          cause: e,
        );
      default:
        return ServerFailure(
          message: 'Server error (${statusCode ?? "unknown"})',
          statusCode: statusCode,
          cause: e,
        );
    }
  }

  static Map<String, List<String>> _parseFieldErrors(dynamic errors) {
    if (errors is! Map<String, dynamic>) return {};
    return errors.map((key, value) {
      if (value is List) return MapEntry(key, value.cast<String>());
      return MapEntry(key, [value.toString()]);
    });
  }
}
```

---

## ขั้นตอนที่ 2285: Repository with Either Pattern

```dart
// lib/features/users/data/repositories/user_repository.dart
import 'package:dio/dio.dart';
import 'package:fpdart/fpdart.dart';
import '../../../../core/error/either_extensions.dart';
import '../../../../core/error/failures.dart';
import '../../../../core/network/dio_error_mapper.dart';

class User {
  final String id;
  final String name;
  final String email;
  final String? avatarUrl;

  const User({
    required this.id,
    required this.name,
    required this.email,
    this.avatarUrl,
  });

  factory User.fromJson(Map<String, dynamic> json) => User(
        id: json['id'] as String,
        name: json['name'] as String,
        email: json['email'] as String,
        avatarUrl: json['avatarUrl'] as String?,
      );
}

class UserRepository {
  final Dio _dio;

  const UserRepository(this._dio);

  Future<AppResult<User>> getUserById(String id) async {
    return runCatching(() async {
      final response = await _dio.get('/users/$id');
      return User.fromJson(response.data as Map<String, dynamic>);
    });
  }

  Future<AppResult<List<User>>> getUsers({int page = 1, int limit = 20}) async {
    try {
      final response = await _dio.get('/users', queryParameters: {
        'page': page,
        'limit': limit,
      });

      final users = (response.data as List)
          .cast<Map<String, dynamic>>()
          .map(User.fromJson)
          .toList();

      return Right(users);
    } on DioException catch (e) {
      return Left(DioErrorMapper.map(e));
    } catch (e, st) {
      return Left(UnknownFailure(message: e.toString(), cause: st));
    }
  }

  Future<AppResult<User>> updateUser({
    required String id,
    String? name,
    String? email,
  }) async {
    return runCatching(() async {
      final response = await _dio.patch('/users/$id', data: {
        if (name != null) 'name': name,
        if (email != null) 'email': email,
      });
      return User.fromJson(response.data as Map<String, dynamic>);
    });
  }

  // TaskEither for chaining multiple operations
  AppTaskResult<User> getUserAndValidate(String id) {
    return TaskEither.tryCatch(
      () async {
        final response = await _dio.get('/users/$id');
        final user = User.fromJson(response.data as Map<String, dynamic>);
        if (user.email.isEmpty) {
          throw const ValidationFailure(message: 'User email is empty');
        }
        return user;
      },
      (error, stackTrace) {
        if (error is DioException) return DioErrorMapper.map(error);
        if (error is Failure) return error;
        return UnknownFailure(message: error.toString(), cause: stackTrace);
      },
    );
  }
}
```

---

## ขั้นตอนที่ 2286: Error Reporting Service

```dart
// lib/core/error/error_reporting_service.dart
import 'package:firebase_crashlytics/firebase_crashlytics.dart';
import 'package:flutter/foundation.dart';
import 'failures.dart';

class ErrorReportingService {
  final FirebaseCrashlytics _crashlytics;

  ErrorReportingService(this._crashlytics);

  static ErrorReportingService? _instance;

  static ErrorReportingService get instance {
    assert(
      _instance != null,
      'ErrorReportingService must be initialized before use',
    );
    return _instance!;
  }

  static Future<void> initialize() async {
    final crashlytics = FirebaseCrashlytics.instance;
    await crashlytics.setCrashlyticsCollectionEnabled(!kDebugMode);

    _instance = ErrorReportingService(crashlytics);

    // Set Flutter error handler
    FlutterError.onError = _instance!._handleFlutterError;

    // Set platform error handler
    PlatformDispatcher.instance.onError = _instance!._handlePlatformError;
  }

  void _handleFlutterError(FlutterErrorDetails details) {
    if (kDebugMode) {
      FlutterError.dumpErrorToConsole(details);
    } else {
      _crashlytics.recordFlutterFatalError(details);
    }
  }

  bool _handlePlatformError(Object error, StackTrace stack) {
    if (kDebugMode) {
      debugPrint('[PlatformError] $error\n$stack');
      return true;
    }
    _crashlytics.recordError(error, stack, fatal: true);
    return true;
  }

  Future<void> reportError(
    Object error,
    StackTrace stackTrace, {
    String? reason,
    Map<String, dynamic>? extras,
    bool fatal = false,
  }) async {
    debugPrint('[ErrorReporting] $error');

    if (extras != null) {
      for (final entry in extras.entries) {
        await _crashlytics.setCustomKey(entry.key, entry.value.toString());
      }
    }

    await _crashlytics.recordError(
      error,
      stackTrace,
      reason: reason,
      fatal: fatal,
    );
  }

  Future<void> reportFailure(
    Failure failure, {
    StackTrace? stackTrace,
    Map<String, dynamic>? extras,
  }) async {
    // Don't report user errors (no internet, auth, etc.)
    if (failure is NoInternetFailure ||
        failure is UnauthorizedFailure ||
        failure is ForbiddenFailure ||
        failure is ValidationFailure) {
      debugPrint('[ErrorReporting] Skipping user error: $failure');
      return;
    }

    await reportError(
      Exception(failure.message),
      stackTrace ?? StackTrace.current,
      reason: failure.code,
      extras: {
        'failure_type': failure.runtimeType.toString(),
        'failure_message': failure.message,
        if (failure.code != null) 'failure_code': failure.code!,
        ...?extras,
      },
    );
  }

  Future<void> setUserIdentifier(String userId) async {
    await _crashlytics.setUserIdentifier(userId);
  }

  Future<void> log(String message) async {
    await _crashlytics.log(message);
    debugPrint('[CrashlyticsLog] $message');
  }

  Future<void> setCustomKey(String key, dynamic value) async {
    await _crashlytics.setCustomKey(key, value.toString());
  }
}
```

---

## ขั้นตอนที่ 2287: Retry Mechanism with Exponential Backoff

```dart
// lib/core/network/retry_handler.dart
import 'dart:math';
import 'package:flutter/foundation.dart';
import 'package:fpdart/fpdart.dart';
import '../error/failures.dart';

/// Configuration for retry behavior
class RetryConfig {
  final int maxAttempts;
  final Duration initialDelay;
  final double backoffFactor;
  final Duration maxDelay;
  final bool Function(Failure)? shouldRetry;

  const RetryConfig({
    this.maxAttempts = 3,
    this.initialDelay = const Duration(seconds: 1),
    this.backoffFactor = 2.0,
    this.maxDelay = const Duration(seconds: 30),
    this.shouldRetry,
  });

  static const RetryConfig network = RetryConfig(
    maxAttempts: 3,
    initialDelay: Duration(seconds: 1),
    backoffFactor: 2.0,
    maxDelay: Duration(seconds: 30),
  );

  static const RetryConfig aggressive = RetryConfig(
    maxAttempts: 5,
    initialDelay: Duration(milliseconds: 500),
    backoffFactor: 1.5,
    maxDelay: Duration(seconds: 60),
  );
}

/// Retry handler with exponential backoff
class RetryHandler {
  static Future<Either<Failure, T>> retry<T>(
    Future<Either<Failure, T>> Function() operation, {
    RetryConfig config = RetryConfig.network,
  }) async {
    int attempt = 0;
    Failure? lastFailure;

    while (attempt < config.maxAttempts) {
      attempt++;

      final result = await operation();

      if (result.isRight()) return result;

      lastFailure = result.fold((f) => f, (_) => null);

      // Check if this failure type should be retried
      if (lastFailure != null) {
        final shouldRetry = config.shouldRetry?.call(lastFailure) ??
            _defaultShouldRetry(lastFailure);

        if (!shouldRetry) {
          debugPrint('[Retry] Failure not retryable: $lastFailure');
          return Left(lastFailure);
        }
      }

      if (attempt < config.maxAttempts) {
        final delay = _calculateDelay(attempt, config);
        debugPrint('[Retry] Attempt $attempt failed. Retrying in ${delay.inSeconds}s...');
        await Future.delayed(delay);
      }
    }

    debugPrint('[Retry] All $attempt attempts failed.');
    return Left(lastFailure ?? const UnknownFailure());
  }

  static Duration _calculateDelay(int attempt, RetryConfig config) {
    final exponentialDelay = config.initialDelay *
        pow(config.backoffFactor, attempt - 1).toDouble();

    // Add jitter (±20%) to prevent thundering herd
    final jitter = (Random().nextDouble() * 0.4 - 0.2);
    final withJitter = exponentialDelay * (1 + jitter);

    return withJitter > config.maxDelay ? config.maxDelay : withJitter;
  }

  static bool _defaultShouldRetry(Failure failure) {
    // Don't retry these failures
    if (failure is UnauthorizedFailure) return false;
    if (failure is ForbiddenFailure) return false;
    if (failure is ValidationFailure) return false;
    if (failure is NotFoundFailure) return false;

    // Retry network and server errors
    if (failure is NetworkFailure) return true;
    if (failure is TimeoutFailure) return true;
    if (failure is NoInternetFailure) return false; // No point retrying
    if (failure is ServerFailure) {
      final code = failure.statusCode;
      // Retry 5xx but not 4xx
      return code == null || code >= 500;
    }

    return true; // Default: retry unknown failures
  }
}

/// Extension for easy retry usage
extension RetryableTask<T> on Future<Either<Failure, T>> Function() {
  Future<Either<Failure, T>> withRetry([RetryConfig config = RetryConfig.network]) {
    return RetryHandler.retry(this, config: config);
  }
}
```

---

## ขั้นตอนที่ 2288: Global Error Boundary

```dart
// lib/core/error/error_boundary.dart
import 'package:flutter/material.dart';
import 'error_reporting_service.dart';

class AppErrorBoundary extends StatefulWidget {
  final Widget child;
  final Widget Function(Object error, StackTrace? stackTrace)? errorBuilder;

  const AppErrorBoundary({
    super.key,
    required this.child,
    this.errorBuilder,
  });

  @override
  State<AppErrorBoundary> createState() => _AppErrorBoundaryState();
}

class _AppErrorBoundaryState extends State<AppErrorBoundary> {
  Object? _error;
  StackTrace? _stackTrace;

  @override
  Widget build(BuildContext context) {
    if (_error != null) {
      return widget.errorBuilder?.call(_error!, _stackTrace) ??
          _DefaultErrorScreen(
            error: _error!,
            onRetry: () => setState(() {
              _error = null;
              _stackTrace = null;
            }),
          );
    }

    return widget.child;
  }
}

/// Sets up global error handling for the entire app
void setupGlobalErrorHandling() {
  // Custom error widget for Flutter framework errors
  ErrorWidget.builder = (FlutterErrorDetails details) {
    ErrorReportingService.instance.reportError(
      details.exception,
      details.stack ?? StackTrace.current,
      reason: 'Widget build error',
    );

    return _WidgetErrorScreen(details: details);
  };
}

class _WidgetErrorScreen extends StatelessWidget {
  final FlutterErrorDetails details;

  const _WidgetErrorScreen({required this.details});

  @override
  Widget build(BuildContext context) {
    if (kDebugMode) {
      return ErrorWidget(details.exception);
    }

    return Container(
      color: Colors.red.shade50,
      child: const Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(Icons.broken_image, color: Colors.red, size: 32),
            SizedBox(height: 8),
            Text(
              'Something went wrong',
              style: TextStyle(color: Colors.red),
            ),
          ],
        ),
      ),
    );
  }
}

class _DefaultErrorScreen extends StatelessWidget {
  final Object error;
  final VoidCallback onRetry;

  const _DefaultErrorScreen({
    required this.error,
    required this.onRetry,
  });

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.error_outline, size: 64, color: Colors.red),
            const SizedBox(height: 16),
            const Text('Something went wrong',
                style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            const Text('We\'ve been notified and are working on a fix.',
                style: TextStyle(color: Colors.grey)),
            const SizedBox(height: 24),
            ElevatedButton(onPressed: onRetry, child: const Text('Retry')),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2289: User-Friendly Error Screens

```dart
// lib/core/error/error_screens.dart
import 'package:flutter/material.dart';
import 'failures.dart';

class ErrorScreen extends StatelessWidget {
  final Failure failure;
  final VoidCallback? onRetry;
  final VoidCallback? onGoBack;

  const ErrorScreen({
    super.key,
    required this.failure,
    this.onRetry,
    this.onGoBack,
  });

  factory ErrorScreen.fromFailure({
    required Failure failure,
    VoidCallback? onRetry,
    VoidCallback? onGoBack,
  }) {
    return ErrorScreen(
      failure: failure,
      onRetry: onRetry,
      onGoBack: onGoBack,
    );
  }

  _ErrorConfig get _config => _getConfig();

  _ErrorConfig _getConfig() {
    if (failure is NoInternetFailure) {
      return const _ErrorConfig(
        icon: Icons.wifi_off_rounded,
        iconColor: Colors.orange,
        title: 'No Internet Connection',
        description:
            'Please check your connection and try again.',
        canRetry: true,
      );
    }
    if (failure is TimeoutFailure) {
      return const _ErrorConfig(
        icon: Icons.hourglass_empty_rounded,
        iconColor: Colors.orange,
        title: 'Request Timed Out',
        description: 'The server is taking too long to respond.',
        canRetry: true,
      );
    }
    if (failure is NotFoundFailure) {
      return const _ErrorConfig(
        icon: Icons.search_off_rounded,
        iconColor: Colors.blue,
        title: 'Not Found',
        description: 'The item you\'re looking for doesn\'t exist.',
        canRetry: false,
      );
    }
    if (failure is UnauthorizedFailure) {
      return const _ErrorConfig(
        icon: Icons.lock_outline_rounded,
        iconColor: Colors.red,
        title: 'Session Expired',
        description: 'Please log in again to continue.',
        canRetry: false,
        actionLabel: 'Log In',
      );
    }
    if (failure is ServerFailure) {
      return const _ErrorConfig(
        icon: Icons.cloud_off_rounded,
        iconColor: Colors.red,
        title: 'Server Error',
        description: 'Our servers are having trouble. Please try again later.',
        canRetry: true,
      );
    }
    if (failure is ValidationFailure) {
      return _ErrorConfig(
        icon: Icons.warning_amber_rounded,
        iconColor: Colors.orange,
        title: 'Invalid Data',
        description: failure.message,
        canRetry: false,
      );
    }
    return _ErrorConfig(
      icon: Icons.error_outline_rounded,
      iconColor: Colors.red,
      title: 'Something Went Wrong',
      description: failure.message,
      canRetry: onRetry != null,
    );
  }

  @override
  Widget build(BuildContext context) {
    final config = _config;
    return Padding(
      padding: const EdgeInsets.all(32),
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        crossAxisAlignment: CrossAxisAlignment.center,
        children: [
          Icon(config.icon, size: 80, color: config.iconColor),
          const SizedBox(height: 24),
          Text(
            config.title,
            style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                  fontWeight: FontWeight.bold,
                ),
            textAlign: TextAlign.center,
          ),
          const SizedBox(height: 12),
          Text(
            config.description,
            style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                  color: Colors.grey.shade600,
                ),
            textAlign: TextAlign.center,
          ),
          if (failure is ValidationFailure &&
              (failure as ValidationFailure).fieldErrors.isNotEmpty) ...[
            const SizedBox(height: 16),
            _buildFieldErrors((failure as ValidationFailure).fieldErrors),
          ],
          const SizedBox(height: 32),
          if (config.canRetry && onRetry != null)
            SizedBox(
              width: double.infinity,
              child: ElevatedButton.icon(
                onPressed: onRetry,
                icon: const Icon(Icons.refresh),
                label: Text(config.actionLabel ?? 'Try Again'),
              ),
            ),
          if (onGoBack != null) ...[
            const SizedBox(height: 12),
            SizedBox(
              width: double.infinity,
              child: OutlinedButton(
                onPressed: onGoBack,
                child: const Text('Go Back'),
              ),
            ),
          ],
        ],
      ),
    );
  }

  Widget _buildFieldErrors(Map<String, List<String>> fieldErrors) {
    return Card(
      color: Colors.orange.shade50,
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: fieldErrors.entries
              .map((e) => Padding(
                    padding: const EdgeInsets.only(bottom: 4),
                    child: Text(
                      '${e.key}: ${e.value.join(', ')}',
                      style: const TextStyle(
                          color: Colors.orange, fontSize: 12),
                    ),
                  ))
              .toList(),
        ),
      ),
    );
  }
}

class _ErrorConfig {
  final IconData icon;
  final Color iconColor;
  final String title;
  final String description;
  final bool canRetry;
  final String? actionLabel;

  const _ErrorConfig({
    required this.icon,
    required this.iconColor,
    required this.title,
    required this.description,
    required this.canRetry,
    this.actionLabel,
  });
}
```

---

## ขั้นตอนที่ 2290: Complete Usage Example

```dart
// lib/features/users/presentation/users_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:fpdart/fpdart.dart';
import '../../../core/error/error_screens.dart';
import '../../../core/error/failures.dart';
import '../../../core/network/retry_handler.dart';
import '../data/repositories/user_repository.dart';

final usersProvider = FutureProvider.autoDispose<List<User>>((ref) async {
  final repo = ref.watch(userRepositoryProvider);

  // With retry
  final result = await RetryHandler.retry(
    () => repo.getUsers(),
    config: RetryConfig.network,
  );

  return result.fold(
    (failure) => throw failure,
    (users) => users,
  );
});

final userRepositoryProvider = Provider<UserRepository>((ref) {
  // In real app, inject Dio instance
  throw UnimplementedError('Provide a real Dio instance');
});

class UsersPage extends ConsumerWidget {
  const UsersPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final usersAsync = ref.watch(usersProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Users')),
      body: usersAsync.when(
        data: (users) => _UsersList(users: users),
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, stackTrace) {
          final failure = error is Failure
              ? error
              : UnknownFailure(message: error.toString(), cause: stackTrace);

          return ErrorScreen(
            failure: failure,
            onRetry: () => ref.invalidate(usersProvider),
            onGoBack: () => Navigator.of(context).pop(),
          );
        },
      ),
    );
  }
}

class _UsersList extends StatelessWidget {
  final List<User> users;
  const _UsersList({required this.users});

  @override
  Widget build(BuildContext context) {
    return ListView.separated(
      itemCount: users.length,
      separatorBuilder: (_, __) => const Divider(height: 1),
      itemBuilder: (context, index) {
        final user = users[index];
        return ListTile(
          leading: CircleAvatar(
            backgroundImage: user.avatarUrl != null
                ? NetworkImage(user.avatarUrl!)
                : null,
            child: user.avatarUrl == null
                ? Text(user.name[0].toUpperCase())
                : null,
          ),
          title: Text(user.name),
          subtitle: Text(user.email),
        );
      },
    );
  }
}
```

---

## ขั้นตอนที่ 2291: Main with Error Reporting Setup

```dart
// lib/main.dart
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'core/error/error_boundary.dart';
import 'core/error/error_reporting_service.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Initialize Firebase (requires google-services.json / GoogleService-Info.plist)
  await Firebase.initializeApp();

  // Initialize error reporting
  await ErrorReportingService.initialize();

  // Setup global error handling
  setupGlobalErrorHandling();

  runApp(
    const ProviderScope(
      child: AppErrorBoundary(
        child: MyApp(),
      ),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Error Handling Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const DemoPage(),
    );
  }
}

class DemoPage extends StatelessWidget {
  const DemoPage({super.key});

  @override
  Widget build(BuildContext context) {
    final failures = [
      const NoInternetFailure(),
      const TimeoutFailure(),
      const NotFoundFailure(),
      const UnauthorizedFailure(),
      const ServerFailure(statusCode: 500),
      ValidationFailure(
        message: 'Please fix the errors below',
        fieldErrors: {
          'email': ['Invalid email format'],
          'password': ['Too short', 'Must contain a number'],
        },
      ),
    ];

    return Scaffold(
      appBar: AppBar(title: const Text('Error Screens Demo')),
      body: ListView.builder(
        padding: const EdgeInsets.all(16),
        itemCount: failures.length,
        itemBuilder: (context, index) {
          final failure = failures[index];
          return Card(
            margin: const EdgeInsets.only(bottom: 8),
            child: ListTile(
              title: Text(failure.runtimeType.toString()),
              subtitle: Text(failure.message),
              trailing: const Icon(Icons.chevron_right),
              onTap: () {
                showModalBottomSheet(
                  context: context,
                  builder: (_) => SizedBox(
                    height: 400,
                    child: ErrorScreen(
                      failure: failure,
                      onRetry: failure is NoInternetFailure ||
                              failure is TimeoutFailure ||
                              failure is ServerFailure
                          ? () => Navigator.pop(context)
                          : null,
                      onGoBack: () => Navigator.pop(context),
                    ),
                  ),
                );
              },
            ),
          );
        },
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2292: Unit Tests for Error Handling

```dart
// test/error_handling_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:fpdart/fpdart.dart';
import 'package:mocktail/mocktail.dart';
import 'package:dio/dio.dart';
import 'package:myapp/core/error/failures.dart';
import 'package:myapp/core/network/dio_error_mapper.dart';
import 'package:myapp/core/network/retry_handler.dart';

void main() {
  group('DioErrorMapper', () {
    test('maps 401 to UnauthorizedFailure', () {
      final error = DioException(
        requestOptions: RequestOptions(path: '/test'),
        response: Response(
          requestOptions: RequestOptions(path: '/test'),
          statusCode: 401,
        ),
        type: DioExceptionType.badResponse,
      );

      final failure = DioErrorMapper.map(error);
      expect(failure, isA<UnauthorizedFailure>());
    });

    test('maps 404 to NotFoundFailure', () {
      final error = DioException(
        requestOptions: RequestOptions(path: '/test'),
        response: Response(
          requestOptions: RequestOptions(path: '/test'),
          statusCode: 404,
        ),
        type: DioExceptionType.badResponse,
      );

      final failure = DioErrorMapper.map(error);
      expect(failure, isA<NotFoundFailure>());
    });

    test('maps timeout to TimeoutFailure', () {
      final error = DioException(
        requestOptions: RequestOptions(path: '/test'),
        type: DioExceptionType.connectionTimeout,
      );

      final failure = DioErrorMapper.map(error);
      expect(failure, isA<TimeoutFailure>());
    });
  });

  group('RetryHandler', () {
    test('succeeds on first attempt', () async {
      int callCount = 0;

      final result = await RetryHandler.retry(() async {
        callCount++;
        return const Right<Failure, int>(42);
      });

      expect(result.isRight(), true);
      expect(result.getOrElse((_) => 0), 42);
      expect(callCount, 1);
    });

    test('retries on retriable failure and eventually succeeds', () async {
      int callCount = 0;

      final result = await RetryHandler.retry(
        () async {
          callCount++;
          if (callCount < 3) {
            return const Left<Failure, int>(TimeoutFailure());
          }
          return const Right<Failure, int>(99);
        },
        config: RetryConfig(
          maxAttempts: 3,
          initialDelay: Duration.zero,
        ),
      );

      expect(result.isRight(), true);
      expect(callCount, 3);
    });

    test('does not retry UnauthorizedFailure', () async {
      int callCount = 0;

      final result = await RetryHandler.retry<int>(() async {
        callCount++;
        return const Left(UnauthorizedFailure());
      });

      expect(result.isLeft(), true);
      expect(callCount, 1);
    });

    test('returns last failure after max attempts', () async {
      final result = await RetryHandler.retry<int>(
        () async => const Left(TimeoutFailure()),
        config: RetryConfig(
          maxAttempts: 2,
          initialDelay: Duration.zero,
        ),
      );

      expect(result.isLeft(), true);
      expect(result.fold((f) => f, (_) => null), isA<TimeoutFailure>());
    });
  });

  group('Failure types', () {
    test('ValidationFailure contains field errors', () {
      const failure = ValidationFailure(
        message: 'Validation failed',
        fieldErrors: {
          'email': ['Invalid format'],
          'name': ['Too short'],
        },
      );

      expect(failure.fieldErrors['email'], contains('Invalid format'));
      expect(failure.fieldErrors['name'], contains('Too short'));
    });

    test('ServerFailure with status code', () {
      const failure = ServerFailure(statusCode: 503, message: 'Service unavailable');
      expect(failure.statusCode, 503);
    });
  });
}
```

---

**← [Part 59](part-59-advanced-navigation.md)**
**ต่อไป: [Part 61 →](part-61-advanced-testing.md)**
