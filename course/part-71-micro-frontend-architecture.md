# Part 71: Micro-Frontend Architecture ใน Flutter
## ขั้นตอนที่ 2721-2760

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ Module Federation แบบ Flutter
- สร้าง Feature Modules แยกเป็น Package
- จัดการ Shared Package สำหรับ UI/Utils ร่วมกัน
- บริหาร Package Dependencies อย่างมืออาชีพ
- สื่อสารระหว่าง Modules ด้วย Event Bus
- สร้าง Federated Plugin Structure ที่ใช้งานได้จริง

---

## ขั้นตอนที่ 2721: ทำความเข้าใจ Micro-Frontend Architecture

Micro-Frontend คือแนวคิดที่แบ่ง App ขนาดใหญ่ออกเป็น Feature Modules อิสระ ซึ่งสามารถพัฒนา ทดสอบ และ Deploy แยกกันได้

```
my_flutter_app/
├── packages/
│   ├── core/                    # Shared core package
│   │   ├── lib/
│   │   │   ├── core.dart
│   │   │   ├── models/
│   │   │   ├── services/
│   │   │   └── widgets/
│   │   └── pubspec.yaml
│   ├── feature_auth/            # Auth feature module
│   │   ├── lib/
│   │   │   └── feature_auth.dart
│   │   └── pubspec.yaml
│   ├── feature_home/            # Home feature module
│   │   ├── lib/
│   │   │   └── feature_home.dart
│   │   └── pubspec.yaml
│   └── feature_profile/         # Profile feature module
│       ├── lib/
│       │   └── feature_profile.dart
│       └── pubspec.yaml
├── lib/
│   └── main.dart                # Shell app (orchestrator)
└── pubspec.yaml
```

---

## ขั้นตอนที่ 2722: สร้าง Core Shared Package

```dart
// packages/core/lib/core.dart
library core;

export 'src/models/user.dart';
export 'src/models/result.dart';
export 'src/services/event_bus.dart';
export 'src/services/navigation_service.dart';
export 'src/services/auth_service.dart';
export 'src/widgets/loading_widget.dart';
export 'src/widgets/error_widget.dart';
export 'src/theme/app_theme.dart';
export 'src/constants/app_constants.dart';
```

```dart
// packages/core/lib/src/models/user.dart
class User {
  final String id;
  final String name;
  final String email;
  final String? avatarUrl;
  final UserRole role;
  final DateTime createdAt;

  const User({
    required this.id,
    required this.name,
    required this.email,
    this.avatarUrl,
    required this.role,
    required this.createdAt,
  });

  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'] as String,
      name: json['name'] as String,
      email: json['email'] as String,
      avatarUrl: json['avatarUrl'] as String?,
      role: UserRole.values.byName(json['role'] as String),
      createdAt: DateTime.parse(json['createdAt'] as String),
    );
  }

  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'email': email,
    'avatarUrl': avatarUrl,
    'role': role.name,
    'createdAt': createdAt.toIso8601String(),
  };

  User copyWith({
    String? id,
    String? name,
    String? email,
    String? avatarUrl,
    UserRole? role,
    DateTime? createdAt,
  }) {
    return User(
      id: id ?? this.id,
      name: name ?? this.name,
      email: email ?? this.email,
      avatarUrl: avatarUrl ?? this.avatarUrl,
      role: role ?? this.role,
      createdAt: createdAt ?? this.createdAt,
    );
  }

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is User && runtimeType == other.runtimeType && id == other.id;

  @override
  int get hashCode => id.hashCode;

  @override
  String toString() => 'User(id: $id, name: $name, email: $email, role: $role)';
}

enum UserRole { admin, user, guest }
```

---

## ขั้นตอนที่ 2723: สร้าง Result Model สำหรับ Error Handling

```dart
// packages/core/lib/src/models/result.dart
sealed class Result<T> {
  const Result();

  bool get isSuccess => this is Success<T>;
  bool get isFailure => this is Failure<T>;

  T? get valueOrNull => switch (this) {
    Success<T>(value: final v) => v,
    Failure<T>() => null,
  };

  String? get errorOrNull => switch (this) {
    Success<T>() => null,
    Failure<T>(message: final m) => m,
  };

  R when<R>({
    required R Function(T value) success,
    required R Function(String message, Exception? exception) failure,
  }) {
    return switch (this) {
      Success<T>(value: final v) => success(v),
      Failure<T>(message: final m, exception: final e) => failure(m, e),
    };
  }

  Result<R> map<R>(R Function(T value) transform) {
    return switch (this) {
      Success<T>(value: final v) => Success(transform(v)),
      Failure<T>(message: final m, exception: final e) =>
          Failure(message: m, exception: e),
    };
  }

  Future<Result<R>> mapAsync<R>(Future<R> Function(T value) transform) async {
    return switch (this) {
      Success<T>(value: final v) => Success(await transform(v)),
      Failure<T>(message: final m, exception: final e) =>
          Failure(message: m, exception: e),
    };
  }
}

final class Success<T> extends Result<T> {
  final T value;
  const Success(this.value);

  @override
  String toString() => 'Success($value)';
}

final class Failure<T> extends Result<T> {
  final String message;
  final Exception? exception;
  final StackTrace? stackTrace;

  const Failure({
    required this.message,
    this.exception,
    this.stackTrace,
  });

  @override
  String toString() => 'Failure(message: $message, exception: $exception)';
}

// Helper to wrap Future in Result
Future<Result<T>> resultOf<T>(Future<T> Function() action) async {
  try {
    final value = await action();
    return Success(value);
  } on Exception catch (e, st) {
    return Failure(
      message: e.toString(),
      exception: e,
      stackTrace: st,
    );
  }
}
```

---

## ขั้นตอนที่ 2724: สร้าง Event Bus สำหรับ Module Communication

```dart
// packages/core/lib/src/services/event_bus.dart
import 'dart:async';

/// Type-safe event bus for inter-module communication
class EventBus {
  EventBus._();
  static final EventBus instance = EventBus._();

  final Map<Type, StreamController<dynamic>> _controllers = {};

  /// Emit an event to all listeners
  void emit<T extends AppEvent>(T event) {
    final type = T;
    if (_controllers.containsKey(type)) {
      (_controllers[type] as StreamController<T>).add(event);
    }
  }

  /// Listen to specific event type
  Stream<T> on<T extends AppEvent>() {
    final type = T;
    if (!_controllers.containsKey(type)) {
      _controllers[type] = StreamController<T>.broadcast();
    }
    return (_controllers[type] as StreamController<T>).stream;
  }

  /// Dispose all controllers
  void dispose() {
    for (final controller in _controllers.values) {
      controller.close();
    }
    _controllers.clear();
  }
}

/// Base class for all app events
abstract class AppEvent {
  final DateTime timestamp;
  AppEvent() : timestamp = DateTime.now();
}

// Auth Events
class UserLoggedInEvent extends AppEvent {
  final String userId;
  final String email;
  UserLoggedInEvent({required this.userId, required this.email});
}

class UserLoggedOutEvent extends AppEvent {}

class UserProfileUpdatedEvent extends AppEvent {
  final String userId;
  final Map<String, dynamic> changes;
  UserProfileUpdatedEvent({required this.userId, required this.changes});
}

// Navigation Events
class NavigateToModuleEvent extends AppEvent {
  final String moduleName;
  final Map<String, dynamic>? params;
  NavigateToModuleEvent({required this.moduleName, this.params});
}

// Cart Events (for e-commerce micro-frontends)
class ItemAddedToCartEvent extends AppEvent {
  final String productId;
  final int quantity;
  ItemAddedToCartEvent({required this.productId, required this.quantity});
}

class CartUpdatedEvent extends AppEvent {
  final int itemCount;
  final double total;
  CartUpdatedEvent({required this.itemCount, required this.total});
}

// Example usage of EventBus
void demonstrateEventBus() {
  final bus = EventBus.instance;

  // Feature Auth module emits login event
  final loginSubscription = bus.on<UserLoggedInEvent>().listen((event) {
    print('User logged in: ${event.userId} at ${event.timestamp}');
  });

  // Feature Home module listens for cart updates
  final cartSubscription = bus.on<CartUpdatedEvent>().listen((event) {
    print('Cart updated: ${event.itemCount} items, total: ${event.total}');
  });

  // Emit events
  bus.emit(UserLoggedInEvent(userId: 'user123', email: 'test@test.com'));
  bus.emit(CartUpdatedEvent(itemCount: 3, total: 99.99));

  // Cleanup
  loginSubscription.cancel();
  cartSubscription.cancel();
}
```

---

## ขั้นตอนที่ 2725: สร้าง Navigation Service

```dart
// packages/core/lib/src/services/navigation_service.dart
import 'package:flutter/material.dart';

/// Abstract interface for navigation - allows each module to navigate
/// without depending on specific route implementations
abstract class NavigationService {
  void navigateTo(String routeName, {Map<String, dynamic>? params});
  void navigateBack({dynamic result});
  void navigateAndReplace(String routeName, {Map<String, dynamic>? params});
  void navigateAndClearStack(String routeName, {Map<String, dynamic>? params});
  bool canPop();
}

/// Module route definition
class ModuleRoute {
  final String path;
  final String moduleName;
  final WidgetBuilder builder;
  final List<RouteGuard> guards;

  const ModuleRoute({
    required this.path,
    required this.moduleName,
    required this.builder,
    this.guards = const [],
  });
}

/// Route guard interface for authentication/authorization
abstract class RouteGuard {
  Future<bool> canActivate(String routeName, Map<String, dynamic>? params);
  String? get redirectTo; // redirect if guard fails
}

/// Auth guard implementation
class AuthGuard implements RouteGuard {
  final bool Function() isAuthenticated;

  AuthGuard({required this.isAuthenticated});

  @override
  Future<bool> canActivate(String routeName, Map<String, dynamic>? params) async {
    return isAuthenticated();
  }

  @override
  String? get redirectTo => '/auth/login';
}

/// Module registry - tracks all registered feature modules
class ModuleRegistry {
  ModuleRegistry._();
  static final ModuleRegistry instance = ModuleRegistry._();

  final Map<String, FeatureModule> _modules = {};
  final Map<String, ModuleRoute> _routes = {};

  void registerModule(FeatureModule module) {
    _modules[module.name] = module;
    for (final route in module.routes) {
      _routes[route.path] = route;
    }
    module.initialize();
  }

  FeatureModule? getModule(String name) => _modules[name];

  ModuleRoute? getRoute(String path) => _routes[path];

  List<String> get registeredModules => _modules.keys.toList();

  Map<String, WidgetBuilder> get routeMap {
    return {
      for (final entry in _routes.entries)
        entry.key: entry.value.builder,
    };
  }
}

/// Abstract feature module interface
abstract class FeatureModule {
  String get name;
  String get version;
  List<ModuleRoute> get routes;
  List<String> get dependencies;

  void initialize();
  void dispose();
}
```

---

## ขั้นตอนที่ 2726: สร้าง Shared Theme Package

```dart
// packages/core/lib/src/theme/app_theme.dart
import 'package:flutter/material.dart';

class AppTheme {
  AppTheme._();

  // Color palette
  static const Color primary = Color(0xFF6200EE);
  static const Color primaryVariant = Color(0xFF3700B3);
  static const Color secondary = Color(0xFF03DAC6);
  static const Color secondaryVariant = Color(0xFF018786);
  static const Color background = Color(0xFFFFFFFF);
  static const Color surface = Color(0xFFFFFFFF);
  static const Color error = Color(0xFFB00020);

  static ThemeData get lightTheme {
    return ThemeData(
      useMaterial3: true,
      colorScheme: const ColorScheme.light(
        primary: primary,
        secondary: secondary,
        error: error,
      ),
      appBarTheme: const AppBarTheme(
        backgroundColor: primary,
        foregroundColor: Colors.white,
        elevation: 0,
      ),
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: primary,
          foregroundColor: Colors.white,
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 12),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(8),
          ),
        ),
      ),
      cardTheme: const CardTheme(
        elevation: 2,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.all(Radius.circular(12)),
        ),
      ),
      inputDecorationTheme: InputDecorationTheme(
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(8),
        ),
        contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
      ),
    );
  }

  static ThemeData get darkTheme {
    return ThemeData(
      useMaterial3: true,
      brightness: Brightness.dark,
      colorScheme: ColorScheme.dark(
        primary: primary,
        secondary: secondary,
        error: error,
        surface: Colors.grey[900]!,
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2727: สร้าง Feature Auth Module

```dart
// packages/feature_auth/lib/feature_auth.dart
library feature_auth;

export 'src/auth_module.dart';
export 'src/screens/login_screen.dart';
export 'src/screens/register_screen.dart';
export 'src/services/auth_service_impl.dart';
```

```dart
// packages/feature_auth/lib/src/auth_module.dart
import 'package:flutter/material.dart';
import 'package:core/core.dart';

class AuthModule implements FeatureModule {
  @override
  String get name => 'auth';

  @override
  String get version => '1.0.0';

  @override
  List<String> get dependencies => ['core'];

  @override
  List<ModuleRoute> get routes => [
    ModuleRoute(
      path: '/auth/login',
      moduleName: name,
      builder: (context) => const LoginScreen(),
    ),
    ModuleRoute(
      path: '/auth/register',
      moduleName: name,
      builder: (context) => const RegisterScreen(),
    ),
  ];

  @override
  void initialize() {
    debugPrint('AuthModule v$version initialized');
  }

  @override
  void dispose() {
    debugPrint('AuthModule disposed');
  }
}
```

```dart
// packages/feature_auth/lib/src/screens/login_screen.dart
import 'package:flutter/material.dart';
import 'package:core/core.dart';

class LoginScreen extends StatefulWidget {
  const LoginScreen({super.key});

  @override
  State<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  bool _isLoading = false;
  bool _obscurePassword = true;

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  Future<void> _handleLogin() async {
    if (!_formKey.currentState!.validate()) return;

    setState(() => _isLoading = true);

    try {
      // Simulate auth
      await Future.delayed(const Duration(seconds: 1));

      // Emit login event for other modules to react
      EventBus.instance.emit(
        UserLoggedInEvent(
          userId: 'user_${DateTime.now().millisecondsSinceEpoch}',
          email: _emailController.text,
        ),
      );

      if (mounted) {
        Navigator.of(context).pushReplacementNamed('/home');
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Login failed: $e')),
        );
      }
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: Center(
          child: SingleChildScrollView(
            padding: const EdgeInsets.all(24),
            child: Form(
              key: _formKey,
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.stretch,
                children: [
                  const Icon(
                    Icons.flutter_dash,
                    size: 80,
                    color: AppTheme.primary,
                  ),
                  const SizedBox(height: 24),
                  Text(
                    'Welcome Back',
                    style: Theme.of(context).textTheme.headlineMedium?.copyWith(
                      fontWeight: FontWeight.bold,
                    ),
                    textAlign: TextAlign.center,
                  ),
                  const SizedBox(height: 32),
                  TextFormField(
                    controller: _emailController,
                    keyboardType: TextInputType.emailAddress,
                    decoration: const InputDecoration(
                      labelText: 'Email',
                      prefixIcon: Icon(Icons.email_outlined),
                    ),
                    validator: (value) {
                      if (value == null || value.isEmpty) return 'Email required';
                      if (!value.contains('@')) return 'Invalid email';
                      return null;
                    },
                  ),
                  const SizedBox(height: 16),
                  TextFormField(
                    controller: _passwordController,
                    obscureText: _obscurePassword,
                    decoration: InputDecoration(
                      labelText: 'Password',
                      prefixIcon: const Icon(Icons.lock_outlined),
                      suffixIcon: IconButton(
                        icon: Icon(
                          _obscurePassword
                              ? Icons.visibility_off
                              : Icons.visibility,
                        ),
                        onPressed: () =>
                            setState(() => _obscurePassword = !_obscurePassword),
                      ),
                    ),
                    validator: (value) {
                      if (value == null || value.isEmpty) return 'Password required';
                      if (value.length < 6) return 'Minimum 6 characters';
                      return null;
                    },
                  ),
                  const SizedBox(height: 24),
                  ElevatedButton(
                    onPressed: _isLoading ? null : _handleLogin,
                    child: _isLoading
                        ? const SizedBox(
                            height: 20,
                            width: 20,
                            child: CircularProgressIndicator(
                              strokeWidth: 2,
                              color: Colors.white,
                            ),
                          )
                        : const Text('Login'),
                  ),
                  const SizedBox(height: 16),
                  TextButton(
                    onPressed: () => Navigator.pushNamed(context, '/auth/register'),
                    child: const Text("Don't have an account? Register"),
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2728: สร้าง Feature Home Module

```dart
// packages/feature_home/lib/src/home_module.dart
import 'package:flutter/material.dart';
import 'package:core/core.dart';

class HomeModule implements FeatureModule {
  @override
  String get name => 'home';

  @override
  String get version => '1.0.0';

  @override
  List<String> get dependencies => ['core', 'auth'];

  @override
  List<ModuleRoute> get routes => [
    ModuleRoute(
      path: '/home',
      moduleName: name,
      builder: (context) => const HomeScreen(),
      guards: [AuthGuard(isAuthenticated: () => true)],
    ),
    ModuleRoute(
      path: '/home/details',
      moduleName: name,
      builder: (context) => const DetailsScreen(),
    ),
  ];

  @override
  void initialize() {
    // Listen to events from other modules
    EventBus.instance.on<UserLoggedInEvent>().listen((event) {
      debugPrint('HomeModule: User logged in - ${event.userId}');
    });

    EventBus.instance.on<UserLoggedOutEvent>().listen((_) {
      debugPrint('HomeModule: User logged out - clearing cache');
    });
  }

  @override
  void dispose() {}
}

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Home'),
        actions: [
          IconButton(
            icon: const Icon(Icons.person),
            onPressed: () => Navigator.pushNamed(context, '/profile'),
          ),
        ],
      ),
      body: GridView.builder(
        padding: const EdgeInsets.all(16),
        gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
          crossAxisCount: 2,
          childAspectRatio: 0.8,
          crossAxisSpacing: 16,
          mainAxisSpacing: 16,
        ),
        itemCount: 10,
        itemBuilder: (context, index) {
          return _FeatureCard(
            title: 'Feature ${index + 1}',
            icon: Icons.star,
            onTap: () => Navigator.pushNamed(context, '/home/details'),
          );
        },
      ),
    );
  }
}

class _FeatureCard extends StatelessWidget {
  final String title;
  final IconData icon;
  final VoidCallback onTap;

  const _FeatureCard({
    required this.title,
    required this.icon,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      child: InkWell(
        onTap: onTap,
        borderRadius: BorderRadius.circular(12),
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Icon(icon, size: 48, color: AppTheme.primary),
              const SizedBox(height: 8),
              Text(
                title,
                style: Theme.of(context).textTheme.titleMedium,
                textAlign: TextAlign.center,
              ),
            ],
          ),
        ),
      ),
    );
  }
}

class DetailsScreen extends StatelessWidget {
  const DetailsScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Details')),
      body: const Center(child: Text('Details from Home Module')),
    );
  }
}
```

---

## ขั้นตอนที่ 2729: สร้าง Shell App (Orchestrator)

```dart
// lib/main.dart - Shell App that orchestrates all modules
import 'package:flutter/material.dart';
import 'package:core/core.dart';
import 'package:feature_auth/feature_auth.dart';
import 'package:feature_home/feature_home.dart';
import 'package:feature_profile/feature_profile.dart';

void main() {
  // Register all modules
  final registry = ModuleRegistry.instance;
  registry.registerModule(AuthModule());
  registry.registerModule(HomeModule());
  registry.registerModule(ProfileModule());

  runApp(const ShellApp());
}

class ShellApp extends StatelessWidget {
  const ShellApp({super.key});

  @override
  Widget build(BuildContext context) {
    final registry = ModuleRegistry.instance;

    return MaterialApp(
      title: 'Micro-Frontend Flutter App',
      theme: AppTheme.lightTheme,
      darkTheme: AppTheme.darkTheme,
      themeMode: ThemeMode.system,
      initialRoute: '/auth/login',
      routes: registry.routeMap,
      onUnknownRoute: (settings) {
        return MaterialPageRoute(
          builder: (_) => const _NotFoundScreen(),
        );
      },
    );
  }
}

class _NotFoundScreen extends StatelessWidget {
  const _NotFoundScreen();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Not Found')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.error_outline, size: 64, color: Colors.red),
            const SizedBox(height: 16),
            const Text('Page not found'),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () => Navigator.pushReplacementNamed(context, '/auth/login'),
              child: const Text('Go to Login'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2730: pubspec.yaml สำหรับแต่ละ Package

```yaml
# packages/core/pubspec.yaml
name: core
description: Shared core package for the Flutter micro-frontend app
version: 1.0.0

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
```

```yaml
# packages/feature_auth/pubspec.yaml
name: feature_auth
description: Authentication feature module
version: 1.0.0

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter
  core:
    path: ../core

dev_dependencies:
  flutter_test:
    sdk: flutter
```

```yaml
# packages/feature_home/pubspec.yaml
name: feature_home
description: Home feature module
version: 1.0.0

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter
  core:
    path: ../core

dev_dependencies:
  flutter_test:
    sdk: flutter
```

```yaml
# Shell app pubspec.yaml
name: my_flutter_app
description: Micro-frontend Flutter shell application
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter
  core:
    path: packages/core
  feature_auth:
    path: packages/feature_auth
  feature_home:
    path: packages/feature_home
  feature_profile:
    path: packages/feature_profile

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0

flutter:
  uses-material-design: true
```

---

## ขั้นตอนที่ 2731: Module Communication Pattern - Inter-Module Events

```dart
// ตัวอย่างการสื่อสารระหว่าง modules แบบ bidirectional
// packages/core/lib/src/services/module_communication.dart

import 'dart:async';

/// Request-Response pattern for inter-module communication
class ModuleBridge {
  ModuleBridge._();
  static final ModuleBridge instance = ModuleBridge._();

  final Map<String, _RequestHandler> _handlers = {};
  final Map<String, Completer<dynamic>> _pendingRequests = {};

  /// Register a handler for a specific request type
  void registerHandler<TRequest, TResponse>(
    String requestType,
    Future<TResponse> Function(TRequest request) handler,
  ) {
    _handlers[requestType] = _RequestHandler(
      type: requestType,
      handle: (request) => handler(request as TRequest),
    );
  }

  /// Send a request and wait for response
  Future<TResponse> request<TRequest, TResponse>(
    String requestType,
    TRequest data,
  ) async {
    final handler = _handlers[requestType];
    if (handler == null) {
      throw StateError('No handler registered for: $requestType');
    }
    return await handler.handle(data) as TResponse;
  }

  void unregisterHandler(String requestType) {
    _handlers.remove(requestType);
  }
}

class _RequestHandler {
  final String type;
  final Future<dynamic> Function(dynamic request) handle;

  _RequestHandler({required this.type, required this.handle});
}

// Usage example:
// In ProfileModule:
void setupProfileBridge() {
  ModuleBridge.instance.registerHandler<String, Map<String, dynamic>>(
    'getUserProfile',
    (userId) async {
      // fetch profile from local DB or API
      await Future.delayed(const Duration(milliseconds: 100));
      return {'id': userId, 'name': 'John Doe', 'points': 1500};
    },
  );
}

// In HomeModule:
Future<void> loadUserProfile(String userId) async {
  final profile = await ModuleBridge.instance.request<String, Map<String, dynamic>>(
    'getUserProfile',
    userId,
  );
  print('Profile: $profile');
}
```

---

## ขั้นตอนที่ 2732: Testing Modules Independently

```dart
// packages/feature_auth/test/auth_module_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:core/core.dart';
import 'package:feature_auth/feature_auth.dart';

void main() {
  group('AuthModule', () {
    late AuthModule module;

    setUp(() {
      module = AuthModule();
    });

    test('should have correct module name', () {
      expect(module.name, equals('auth'));
    });

    test('should define required routes', () {
      final paths = module.routes.map((r) => r.path).toList();
      expect(paths, contains('/auth/login'));
      expect(paths, contains('/auth/register'));
    });

    test('should have core as dependency', () {
      expect(module.dependencies, contains('core'));
    });

    testWidgets('LoginScreen renders correctly', (tester) async {
      await tester.pumpWidget(
        MaterialApp(
          theme: AppTheme.lightTheme,
          home: const LoginScreen(),
        ),
      );

      expect(find.text('Welcome Back'), findsOneWidget);
      expect(find.byType(TextFormField), findsNWidgets(2));
      expect(find.text('Login'), findsOneWidget);
    });

    testWidgets('LoginScreen validates empty email', (tester) async {
      await tester.pumpWidget(
        MaterialApp(
          theme: AppTheme.lightTheme,
          home: const LoginScreen(),
        ),
      );

      // Tap login without filling form
      await tester.tap(find.text('Login'));
      await tester.pump();

      expect(find.text('Email required'), findsOneWidget);
    });
  });

  group('EventBus', () {
    late EventBus bus;

    setUp(() {
      bus = EventBus.instance;
    });

    test('should deliver events to listeners', () async {
      final events = <UserLoggedInEvent>[];
      final sub = bus.on<UserLoggedInEvent>().listen(events.add);

      bus.emit(UserLoggedInEvent(userId: 'u1', email: 'a@b.com'));
      await Future.delayed(Duration.zero);

      expect(events.length, equals(1));
      expect(events.first.userId, equals('u1'));

      await sub.cancel();
    });

    test('should not deliver events to wrong type listener', () async {
      final logoutEvents = <UserLoggedOutEvent>[];
      final sub = bus.on<UserLoggedOutEvent>().listen(logoutEvents.add);

      bus.emit(UserLoggedInEvent(userId: 'u1', email: 'a@b.com'));
      await Future.delayed(Duration.zero);

      expect(logoutEvents, isEmpty);
      await sub.cancel();
    });
  });
}
```

---

## ขั้นตอนที่ 2733: Lazy Module Loading

```dart
// packages/core/lib/src/services/lazy_module_loader.dart
import 'package:flutter/material.dart';

/// Lazy module loader - loads modules on demand
class LazyModuleLoader {
  LazyModuleLoader._();
  static final LazyModuleLoader instance = LazyModuleLoader._();

  final Map<String, Future<FeatureModule> Function()> _factories = {};
  final Map<String, FeatureModule> _loaded = {};

  /// Register a module factory (not loaded until needed)
  void register(String name, Future<FeatureModule> Function() factory) {
    _factories[name] = factory;
  }

  /// Load a module by name (cached after first load)
  Future<FeatureModule> load(String name) async {
    if (_loaded.containsKey(name)) return _loaded[name]!;

    final factory = _factories[name];
    if (factory == null) throw StateError('Module not registered: $name');

    debugPrint('LazyModuleLoader: Loading module "$name"...');
    final module = await factory();
    module.initialize();
    _loaded[name] = module;
    debugPrint('LazyModuleLoader: Module "$name" loaded successfully');

    return module;
  }

  bool isLoaded(String name) => _loaded.containsKey(name);

  List<String> get loadedModules => _loaded.keys.toList();
}

// Usage: lazy-load the cart module only when user visits shop
void setupLazyModules() {
  LazyModuleLoader.instance.register(
    'cart',
    () async {
      // In a real app, this could be a dynamic import
      await Future.delayed(const Duration(milliseconds: 50)); // simulate load time
      return CartModule();
    },
  );
}

// Dummy CartModule for illustration
class CartModule implements FeatureModule {
  @override String get name => 'cart';
  @override String get version => '1.0.0';
  @override List<ModuleRoute> get routes => [];
  @override List<String> get dependencies => ['core'];
  @override void initialize() => debugPrint('CartModule initialized');
  @override void dispose() => debugPrint('CartModule disposed');
}
```

---

## ขั้นตอนที่ 2734: Feature Flags สำหรับ Module Toggling

```dart
// packages/core/lib/src/services/feature_flags.dart

/// Feature flag service - enables/disables features per environment/user
class FeatureFlags {
  FeatureFlags._();
  static final FeatureFlags instance = FeatureFlags._();

  final Map<String, bool> _flags = {};
  final Map<String, dynamic> _remoteFlags = {};

  /// Initialize with defaults
  void initialize({required Map<String, bool> defaults}) {
    _flags.addAll(defaults);
  }

  /// Load remote flags (from Firebase Remote Config, LaunchDarkly, etc.)
  Future<void> loadRemoteFlags(Map<String, dynamic> remoteConfig) async {
    _remoteFlags.addAll(remoteConfig);
    // Override local flags with remote
    for (final entry in _remoteFlags.entries) {
      if (entry.value is bool) {
        _flags[entry.key] = entry.value as bool;
      }
    }
  }

  bool isEnabled(String flag) => _flags[flag] ?? false;

  void override(String flag, bool value) {
    _flags[flag] = value;
  }

  Map<String, bool> get allFlags => Map.unmodifiable(_flags);
}

// Common feature flags
class AppFlags {
  static const String newHomeUi = 'new_home_ui';
  static const String cartModule = 'cart_module';
  static const String darkMode = 'dark_mode';
  static const String analyticsV2 = 'analytics_v2';
  static const String betaFeatures = 'beta_features';
}

// Usage in Shell App setup:
void initializeFeatureFlags() {
  FeatureFlags.instance.initialize(defaults: {
    AppFlags.newHomeUi: false,
    AppFlags.cartModule: true,
    AppFlags.darkMode: true,
    AppFlags.analyticsV2: false,
    AppFlags.betaFeatures: false,
  });
}

// In a Widget:
class ConditionalFeatureWidget extends StatelessWidget {
  const ConditionalFeatureWidget({super.key});

  @override
  Widget build(BuildContext context) {
    final showNewUi = FeatureFlags.instance.isEnabled(AppFlags.newHomeUi);

    return showNewUi
        ? const _NewHomeUIWidget()
        : const _LegacyHomeUIWidget();
  }
}

class _NewHomeUIWidget extends StatelessWidget {
  const _NewHomeUIWidget();
  @override
  Widget build(BuildContext context) => const Text('New UI');
}

class _LegacyHomeUIWidget extends StatelessWidget {
  const _LegacyHomeUIWidget();
  @override
  Widget build(BuildContext context) => const Text('Legacy UI');
}
```

---

## ขั้นตอนที่ 2735: Module Versioning และ Compatibility

```dart
// packages/core/lib/src/services/module_version_checker.dart

class SemanticVersion implements Comparable<SemanticVersion> {
  final int major;
  final int minor;
  final int patch;

  const SemanticVersion(this.major, this.minor, this.patch);

  factory SemanticVersion.parse(String version) {
    final parts = version.split('.');
    if (parts.length != 3) throw FormatException('Invalid version: $version');
    return SemanticVersion(
      int.parse(parts[0]),
      int.parse(parts[1]),
      int.parse(parts[2]),
    );
  }

  bool isCompatibleWith(SemanticVersion required) {
    // Same major version required
    if (major != required.major) return false;
    // Minor version must be >= required
    if (minor < required.minor) return false;
    return true;
  }

  @override
  int compareTo(SemanticVersion other) {
    if (major != other.major) return major.compareTo(other.major);
    if (minor != other.minor) return minor.compareTo(other.minor);
    return patch.compareTo(other.patch);
  }

  @override
  String toString() => '$major.$minor.$patch';
}

class ModuleVersionChecker {
  static bool checkCompatibility({
    required String moduleName,
    required String currentVersion,
    required String requiredVersion,
  }) {
    try {
      final current = SemanticVersion.parse(currentVersion);
      final required = SemanticVersion.parse(requiredVersion);
      final compatible = current.isCompatibleWith(required);

      if (!compatible) {
        print(
          'WARNING: Module "$moduleName" version $currentVersion '
          'is incompatible with required $requiredVersion',
        );
      }

      return compatible;
    } catch (e) {
      print('Error checking version for $moduleName: $e');
      return false;
    }
  }

  static void validateAllModules(List<FeatureModule> modules) {
    final moduleVersions = {for (final m in modules) m.name: m.version};

    for (final module in modules) {
      for (final dep in module.dependencies) {
        if (dep == 'core') continue; // core is always available
        if (!moduleVersions.containsKey(dep)) {
          print('ERROR: Module "${module.name}" requires "$dep" which is not registered');
        }
      }
    }
    print('Module validation complete for ${modules.length} modules');
  }
}
```

---

**← [Part 70](part-70-performance-optimization.md)**
**ต่อไป: [Part 72 →](part-72-advanced-animations-physics.md)**
