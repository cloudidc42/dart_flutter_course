# Part 59: Advanced Navigation with GoRouter
## ขั้นตอนที่ 2241-2280

## 🎯 เป้าหมายของ Part นี้
- ใช้ GoRouter พร้อม ShellRoute สำหรับ nested navigation
- Bottom navigation bar ที่มี persistent state ต่อแต่ละ tab
- Modal routes และ dialogs ผ่าน GoRouter
- Route guards และ authentication redirect
- Deep link handling ด้วย GoRouter
- Navigation analytics และ logging

---

## ขั้นตอนที่ 2241: Dependencies

```yaml
# pubspec.yaml additions
dependencies:
  go_router: ^13.2.0
  flutter_riverpod: ^2.4.9
  riverpod: ^2.4.9
  shared_preferences: ^2.2.2
```

---

## ขั้นตอนที่ 2242: Route Constants

```dart
// lib/core/navigation/app_routes.dart

class AppRoutes {
  AppRoutes._();

  // --- Root ---
  static const String splash = '/';
  static const String login = '/login';
  static const String register = '/register';

  // --- Shell (Bottom Nav) ---
  static const String home = '/home';
  static const String explore = '/explore';
  static const String notifications = '/notifications';
  static const String profile = '/profile';

  // --- Home Stack ---
  static const String homeDetails = '/home/details/:id';
  static const String homeCreate = '/home/create';

  // --- Explore Stack ---
  static const String exploreSearch = '/explore/search';
  static const String exploreItem = '/explore/item/:id';

  // --- Profile Stack ---
  static const String profileEdit = '/profile/edit';
  static const String profileSettings = '/profile/settings';
  static const String profileAbout = '/profile/settings/about';

  // --- Modal / Dialogs ---
  static const String imageViewer = '/image-viewer';
  static const String confirmDialog = '/confirm';

  // Helper to generate parametrized paths
  static String homeDetailsPath(String id) => '/home/details/$id';
  static String exploreItemPath(String id) => '/explore/item/$id';
}
```

---

## ขั้นตอนที่ 2243: Auth State Provider

```dart
// lib/core/auth/auth_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';

enum AuthStatus { unknown, authenticated, unauthenticated }

class AuthState {
  final AuthStatus status;
  final String? userId;
  final String? displayName;

  const AuthState({
    required this.status,
    this.userId,
    this.displayName,
  });

  const AuthState.initial() : this(status: AuthStatus.unknown);
  const AuthState.authenticated({
    required String userId,
    required String displayName,
  }) : this(
          status: AuthStatus.authenticated,
          userId: userId,
          displayName: displayName,
        );
  const AuthState.unauthenticated()
      : this(status: AuthStatus.unauthenticated);

  bool get isAuthenticated => status == AuthStatus.authenticated;
}

class AuthNotifier extends AsyncNotifier<AuthState> {
  @override
  Future<AuthState> build() async {
    return _checkStoredAuth();
  }

  Future<AuthState> _checkStoredAuth() async {
    final prefs = await SharedPreferences.getInstance();
    final token = prefs.getString('auth_token');
    final userId = prefs.getString('user_id');
    final displayName = prefs.getString('display_name');

    if (token != null && userId != null && displayName != null) {
      return AuthState.authenticated(
          userId: userId, displayName: displayName);
    }
    return const AuthState.unauthenticated();
  }

  Future<void> login(String email, String password) async {
    state = const AsyncValue.loading();
    try {
      // Simulate API call
      await Future.delayed(const Duration(seconds: 1));

      if (email == 'test@example.com' && password == 'password') {
        final prefs = await SharedPreferences.getInstance();
        await prefs.setString('auth_token', 'mock_token_123');
        await prefs.setString('user_id', 'user_001');
        await prefs.setString('display_name', 'John Doe');

        state = const AsyncValue.data(AuthState.authenticated(
          userId: 'user_001',
          displayName: 'John Doe',
        ));
      } else {
        throw Exception('Invalid credentials');
      }
    } catch (e, st) {
      state = AsyncValue.error(e, st);
    }
  }

  Future<void> logout() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.remove('auth_token');
    await prefs.remove('user_id');
    await prefs.remove('display_name');
    state = const AsyncValue.data(AuthState.unauthenticated());
  }
}

final authProvider = AsyncNotifierProvider<AuthNotifier, AuthState>(() {
  return AuthNotifier();
});
```

---

## ขั้นตอนที่ 2244: Navigation Analytics Observer

```dart
// lib/core/navigation/navigation_observer.dart
import 'package:flutter/foundation.dart';
import 'package:go_router/go_router.dart';

class NavigationAnalyticsObserver {
  final List<String> _history = [];

  List<String> get history => List.unmodifiable(_history);
  String? get currentRoute => _history.isNotEmpty ? _history.last : null;

  void onRouteChanged(String? previousRoute, String currentRoute) {
    _history.add(currentRoute);

    // Limit history size
    if (_history.length > 100) _history.removeAt(0);

    debugPrint(
      '[Navigation] $previousRoute → $currentRoute',
    );

    _trackAnalytics(previousRoute, currentRoute);
  }

  void _trackAnalytics(String? from, String to) {
    // In a real app, send to analytics service
    // FirebaseAnalytics.instance.logScreenView(screenName: to);
    debugPrint('[Analytics] Screen view: $to (from: $from)');
  }

  void reset() => _history.clear();
}

/// GoRouter redirect logs - called on every navigation
class AppNavigationLogger extends NavigatorObserver {
  @override
  void didPush(Route<dynamic> route, Route<dynamic>? previousRoute) {
    debugPrint('[Navigator] PUSH: ${route.settings.name}');
  }

  @override
  void didPop(Route<dynamic> route, Route<dynamic>? previousRoute) {
    debugPrint('[Navigator] POP: ${route.settings.name}');
  }

  @override
  void didReplace({Route<dynamic>? newRoute, Route<dynamic>? oldRoute}) {
    debugPrint(
        '[Navigator] REPLACE: ${oldRoute?.settings.name} → ${newRoute?.settings.name}');
  }
}
```

---

## ขั้นตอนที่ 2245: GoRouter Setup with ShellRoute

```dart
// lib/core/navigation/app_router.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../auth/auth_provider.dart';
import 'app_routes.dart';
import 'navigation_observer.dart';
import '../../features/home/presentation/home_page.dart';
import '../../features/explore/presentation/explore_page.dart';
import '../../features/notifications/presentation/notifications_page.dart';
import '../../features/profile/presentation/profile_page.dart';
import '../../features/auth/presentation/login_page.dart';
import '../../features/auth/presentation/register_page.dart';
import '../../features/home/presentation/home_details_page.dart';
import '../../features/home/presentation/home_create_page.dart';
import '../../features/explore/presentation/explore_search_page.dart';
import '../../features/explore/presentation/explore_item_page.dart';
import '../../features/profile/presentation/profile_edit_page.dart';
import '../../features/profile/presentation/profile_settings_page.dart';
import '../../features/splash/splash_page.dart';
import '../../shared/widgets/main_scaffold.dart';

final _analytics = NavigationAnalyticsObserver();

final routerProvider = Provider<GoRouter>((ref) {
  final authNotifier = ref.watch(authProvider.notifier);

  return GoRouter(
    initialLocation: AppRoutes.splash,
    debugLogDiagnostics: true,
    observers: [AppNavigationLogger()],

    // ---- Route Guards ----
    redirect: (context, state) {
      final authState = ref.read(authProvider);

      // While auth is loading, stay on splash
      if (authState.isLoading) {
        return state.matchedLocation == AppRoutes.splash
            ? null
            : AppRoutes.splash;
      }

      final isAuthenticated = authState.valueOrNull?.isAuthenticated ?? false;
      final isOnAuthPage = state.matchedLocation == AppRoutes.login ||
          state.matchedLocation == AppRoutes.register;
      final isOnSplash = state.matchedLocation == AppRoutes.splash;

      if (isOnSplash && !authState.isLoading) {
        return isAuthenticated ? AppRoutes.home : AppRoutes.login;
      }

      if (!isAuthenticated && !isOnAuthPage) {
        return AppRoutes.login;
      }

      if (isAuthenticated && isOnAuthPage) {
        return AppRoutes.home;
      }

      return null;
    },

    routes: [
      // ---- Splash ----
      GoRoute(
        path: AppRoutes.splash,
        builder: (context, state) => const SplashPage(),
      ),

      // ---- Auth ----
      GoRoute(
        path: AppRoutes.login,
        builder: (context, state) => const LoginPage(),
      ),
      GoRoute(
        path: AppRoutes.register,
        builder: (context, state) => const RegisterPage(),
      ),

      // ---- Shell (Bottom Nav) ----
      ShellRoute(
        builder: (context, state, child) => MainScaffold(child: child),
        routes: [
          // Home branch
          GoRoute(
            path: AppRoutes.home,
            pageBuilder: (context, state) =>
                const NoTransitionPage(child: HomePage()),
            routes: [
              GoRoute(
                path: 'details/:id',
                builder: (context, state) => HomeDetailsPage(
                  id: state.pathParameters['id']!,
                ),
              ),
              GoRoute(
                path: 'create',
                builder: (context, state) => const HomeCreatePage(),
              ),
            ],
          ),

          // Explore branch
          GoRoute(
            path: AppRoutes.explore,
            pageBuilder: (context, state) =>
                const NoTransitionPage(child: ExplorePage()),
            routes: [
              GoRoute(
                path: 'search',
                builder: (context, state) => const ExploreSearchPage(),
              ),
              GoRoute(
                path: 'item/:id',
                builder: (context, state) => ExploreItemPage(
                  id: state.pathParameters['id']!,
                ),
              ),
            ],
          ),

          // Notifications branch
          GoRoute(
            path: AppRoutes.notifications,
            pageBuilder: (context, state) =>
                const NoTransitionPage(child: NotificationsPage()),
          ),

          // Profile branch
          GoRoute(
            path: AppRoutes.profile,
            pageBuilder: (context, state) =>
                const NoTransitionPage(child: ProfilePage()),
            routes: [
              GoRoute(
                path: 'edit',
                builder: (context, state) => const ProfileEditPage(),
              ),
              GoRoute(
                path: 'settings',
                builder: (context, state) => const ProfileSettingsPage(),
                routes: [
                  GoRoute(
                    path: 'about',
                    builder: (context, state) => const AboutPage(),
                  ),
                ],
              ),
            ],
          ),
        ],
      ),

      // ---- Full-screen Modals ----
      GoRoute(
        path: AppRoutes.imageViewer,
        pageBuilder: (context, state) {
          final imageUrl = state.uri.queryParameters['url'] ?? '';
          return _buildModalPage(
            context,
            state,
            ImageViewerPage(imageUrl: imageUrl),
          );
        },
      ),
    ],

    errorBuilder: (context, state) => ErrorPage(error: state.error),
  );
});

Page<void> _buildModalPage(
    BuildContext context, GoRouterState state, Widget child) {
  return CustomTransitionPage(
    key: state.pageKey,
    child: child,
    barrierColor: Colors.black54,
    opaque: false,
    transitionsBuilder: (context, animation, _, child) {
      return SlideTransition(
        position: Tween<Offset>(
          begin: const Offset(0, 1),
          end: Offset.zero,
        ).animate(CurvedAnimation(
          parent: animation,
          curve: Curves.easeOutCubic,
        )),
        child: child,
      );
    },
  );
}
```

---

## ขั้นตอนที่ 2246: MainScaffold with Persistent Tab State

```dart
// lib/shared/widgets/main_scaffold.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import '../../core/navigation/app_routes.dart';

class MainScaffold extends StatelessWidget {
  final Widget child;

  const MainScaffold({super.key, required this.child});

  static const _tabs = [
    _TabItem(
      path: AppRoutes.home,
      icon: Icons.home_outlined,
      activeIcon: Icons.home,
      label: 'Home',
    ),
    _TabItem(
      path: AppRoutes.explore,
      icon: Icons.explore_outlined,
      activeIcon: Icons.explore,
      label: 'Explore',
    ),
    _TabItem(
      path: AppRoutes.notifications,
      icon: Icons.notifications_outlined,
      activeIcon: Icons.notifications,
      label: 'Activity',
    ),
    _TabItem(
      path: AppRoutes.profile,
      icon: Icons.person_outlined,
      activeIcon: Icons.person,
      label: 'Profile',
    ),
  ];

  int _currentIndex(String location) {
    final cleanLocation = location.split('?').first;

    for (int i = 0; i < _tabs.length; i++) {
      if (cleanLocation.startsWith(_tabs[i].path)) return i;
    }
    return 0;
  }

  @override
  Widget build(BuildContext context) {
    final location = GoRouterState.of(context).matchedLocation;
    final currentIndex = _currentIndex(location);

    return Scaffold(
      body: child,
      bottomNavigationBar: NavigationBar(
        selectedIndex: currentIndex,
        onDestinationSelected: (index) {
          if (index == currentIndex) return; // Already on tab
          context.go(_tabs[index].path);
        },
        destinations: _tabs
            .asMap()
            .entries
            .map((e) => NavigationDestination(
                  icon: Icon(e.value.icon),
                  selectedIcon: Icon(e.value.activeIcon),
                  label: e.value.label,
                ))
            .toList(),
      ),
    );
  }
}

class _TabItem {
  final String path;
  final IconData icon;
  final IconData activeIcon;
  final String label;

  const _TabItem({
    required this.path,
    required this.icon,
    required this.activeIcon,
    required this.label,
  });
}
```

---

## ขั้นตอนที่ 2247: Login Page with Navigation

```dart
// lib/features/auth/presentation/login_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../core/auth/auth_provider.dart';
import '../../../core/navigation/app_routes.dart';

class LoginPage extends ConsumerStatefulWidget {
  const LoginPage({super.key});

  @override
  ConsumerState<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends ConsumerState<LoginPage> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController(text: 'test@example.com');
  final _passwordController = TextEditingController(text: 'password');

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  Future<void> _login() async {
    if (!_formKey.currentState!.validate()) return;

    await ref.read(authProvider.notifier).login(
          _emailController.text.trim(),
          _passwordController.text,
        );
  }

  @override
  Widget build(BuildContext context) {
    final authState = ref.watch(authProvider);

    // Show error if login failed
    ref.listen(authProvider, (previous, next) {
      if (next.hasError) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            content: Text(next.error.toString()),
            backgroundColor: Colors.red,
          ),
        );
      }
    });

    return Scaffold(
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Form(
            key: _formKey,
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                const Text(
                  'Welcome Back',
                  style: TextStyle(fontSize: 28, fontWeight: FontWeight.bold),
                  textAlign: TextAlign.center,
                ),
                const SizedBox(height: 32),
                TextFormField(
                  controller: _emailController,
                  decoration: const InputDecoration(
                    label: Text('Email'),
                    border: OutlineInputBorder(),
                  ),
                  keyboardType: TextInputType.emailAddress,
                  validator: (v) =>
                      v?.contains('@') == true ? null : 'Invalid email',
                ),
                const SizedBox(height: 16),
                TextFormField(
                  controller: _passwordController,
                  decoration: const InputDecoration(
                    label: Text('Password'),
                    border: OutlineInputBorder(),
                  ),
                  obscureText: true,
                  validator: (v) =>
                      (v?.length ?? 0) >= 6 ? null : 'Too short',
                ),
                const SizedBox(height: 24),
                ElevatedButton(
                  onPressed:
                      authState.isLoading ? null : _login,
                  child: authState.isLoading
                      ? const SizedBox(
                          height: 20,
                          width: 20,
                          child: CircularProgressIndicator(strokeWidth: 2),
                        )
                      : const Text('Login'),
                ),
                const SizedBox(height: 16),
                TextButton(
                  onPressed: () => context.push(AppRoutes.register),
                  child: const Text('Create account'),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2248: Deep Link Configuration & Handling

```dart
// lib/core/navigation/deep_link_handler.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

/// Android: AndroidManifest.xml intent-filter for deep links
/// iOS: Info.plist URL schemes
///
/// Example deep links:
///   myapp://home/details/123
///   https://myapp.com/explore/item/456
///
/// GoRouter handles these automatically via the `path` matching.
/// No additional code needed for basic deep links!

/// For custom schemes, configure in GoRouter:
class DeepLinkConfig {
  static const String scheme = 'myapp';
  static const String host = 'myapp.com';

  /// Test deep link locally:
  /// adb shell am start -W -a android.intent.action.VIEW \
  ///   -d "myapp://home/details/123" com.example.myapp
  ///
  /// For iOS:
  /// xcrun simctl openurl booted "myapp://explore/item/456"
}

/// Custom page for handling unknown routes (404)
class ErrorPage extends StatelessWidget {
  final Exception? error;

  const ErrorPage({super.key, this.error});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Page Not Found')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.error_outline, size: 64, color: Colors.red),
            const SizedBox(height: 16),
            const Text(
              '404 - Page Not Found',
              style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
            ),
            if (error != null) ...[
              const SizedBox(height: 8),
              Text(
                error.toString(),
                style: const TextStyle(color: Colors.grey),
                textAlign: TextAlign.center,
              ),
            ],
            const SizedBox(height: 24),
            ElevatedButton(
              onPressed: () => context.go('/home'),
              child: const Text('Go Home'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2249: Sample Pages (Minimal but Compilable)

```dart
// lib/features/home/presentation/home_page.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import '../../../core/navigation/app_routes.dart';

class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Home')),
      body: ListView.builder(
        padding: const EdgeInsets.all(16),
        itemCount: 10,
        itemBuilder: (context, index) {
          return Card(
            margin: const EdgeInsets.only(bottom: 8),
            child: ListTile(
              title: Text('Item ${index + 1}'),
              subtitle: Text('Tap to see details'),
              trailing: const Icon(Icons.chevron_right),
              onTap: () => context.push(
                AppRoutes.homeDetailsPath('item_${index + 1}'),
              ),
            ),
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => context.push(AppRoutes.homeCreate),
        child: const Icon(Icons.add),
      ),
    );
  }
}

// lib/features/home/presentation/home_details_page.dart
class HomeDetailsPage extends StatelessWidget {
  final String id;
  const HomeDetailsPage({super.key, required this.id});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Details: $id')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('ID: $id', style: const TextStyle(fontSize: 24)),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () => context.pop(),
              child: const Text('Back'),
            ),
          ],
        ),
      ),
    );
  }
}

// lib/features/home/presentation/home_create_page.dart
class HomeCreatePage extends StatelessWidget {
  const HomeCreatePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Create Item')),
      body: const Center(child: Text('Create form here')),
    );
  }
}

// lib/features/explore/presentation/explore_page.dart
class ExplorePage extends StatelessWidget {
  const ExplorePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Explore')),
      body: Center(
        child: ElevatedButton.icon(
          icon: const Icon(Icons.search),
          label: const Text('Search'),
          onPressed: () => context.push(AppRoutes.exploreSearch),
        ),
      ),
    );
  }
}

class ExploreSearchPage extends StatelessWidget {
  const ExploreSearchPage({super.key});
  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('Search')),
        body: const Center(child: Text('Search results')),
      );
}

class ExploreItemPage extends StatelessWidget {
  final String id;
  const ExploreItemPage({super.key, required this.id});
  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: Text('Item $id')),
        body: Center(child: Text('Item: $id')),
      );
}

class NotificationsPage extends StatelessWidget {
  const NotificationsPage({super.key});
  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('Notifications')),
        body: const Center(child: Text('No notifications')),
      );
}

class ProfilePage extends ConsumerWidget {
  const ProfilePage({super.key});
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      appBar: AppBar(title: const Text('Profile')),
      body: Column(
        children: [
          ListTile(
            title: const Text('Edit Profile'),
            onTap: () => context.push(AppRoutes.profileEdit),
          ),
          ListTile(
            title: const Text('Settings'),
            onTap: () => context.push(AppRoutes.profileSettings),
          ),
          ListTile(
            title: const Text('Logout'),
            onTap: () => ref.read(authProvider.notifier).logout(),
          ),
        ],
      ),
    );
  }
}

class ProfileEditPage extends StatelessWidget {
  const ProfileEditPage({super.key});
  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('Edit Profile')),
        body: const Center(child: Text('Edit form')),
      );
}

class ProfileSettingsPage extends StatelessWidget {
  const ProfileSettingsPage({super.key});
  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('Settings')),
        body: ListTile(
          title: const Text('About'),
          onTap: () => context.push(AppRoutes.profileAbout),
        ),
      );
}

class AboutPage extends StatelessWidget {
  const AboutPage({super.key});
  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('About')),
        body: const Center(child: Text('App v1.0.0')),
      );
}

class RegisterPage extends StatelessWidget {
  const RegisterPage({super.key});
  @override
  Widget build(BuildContext context) => Scaffold(
        appBar: AppBar(title: const Text('Register')),
        body: const Center(child: Text('Register form')),
      );
}

class SplashPage extends StatelessWidget {
  const SplashPage({super.key});
  @override
  Widget build(BuildContext context) => const Scaffold(
        body: Center(child: CircularProgressIndicator()),
      );
}

class ImageViewerPage extends StatelessWidget {
  final String imageUrl;
  const ImageViewerPage({super.key, required this.imageUrl});
  @override
  Widget build(BuildContext context) => Scaffold(
        backgroundColor: Colors.black,
        appBar: AppBar(
          backgroundColor: Colors.transparent,
          iconTheme: const IconThemeData(color: Colors.white),
        ),
        body: Center(
          child: imageUrl.isNotEmpty
              ? Image.network(imageUrl)
              : const Icon(Icons.image, color: Colors.white, size: 64),
        ),
      );
}
```

---

## ขั้นตอนที่ 2250: Main App Entry Point

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'core/navigation/app_router.dart';

void main() {
  runApp(const ProviderScope(child: MyApp()));
}

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);

    return MaterialApp.router(
      title: 'Advanced Navigation Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      routerConfig: router,
    );
  }
}
```

---

## ขั้นตอนที่ 2251: GoRouter Dialog Route

```dart
// Usage example: opening a dialog via GoRouter
// This pattern allows dialogs to be deep-linkable and bookmarkable

extension GoRouterDialogExtension on BuildContext {
  Future<T?> pushDialog<T>({
    required String path,
    required Widget Function(BuildContext) builder,
  }) {
    return showDialog<T>(
      context: this,
      builder: builder,
    );
  }
}

// In your router, you can also use DialogPage for dialogs:
// GoRoute(
//   path: '/confirm',
//   pageBuilder: (context, state) => DialogPage(
//     builder: (_) => ConfirmDialog(
//       message: state.uri.queryParameters['message'] ?? '',
//     ),
//   ),
// );

class DialogPage<T> extends Page<T> {
  final WidgetBuilder builder;

  const DialogPage({required this.builder, super.key});

  @override
  Route<T> createRoute(BuildContext context) {
    return DialogRoute<T>(
      context: context,
      settings: this,
      builder: builder,
    );
  }
}

class ConfirmDialog extends StatelessWidget {
  final String message;
  final VoidCallback? onConfirm;

  const ConfirmDialog({
    super.key,
    required this.message,
    this.onConfirm,
  });

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: const Text('Confirm'),
      content: Text(message),
      actions: [
        TextButton(
          onPressed: () => context.pop(false),
          child: const Text('Cancel'),
        ),
        ElevatedButton(
          onPressed: () {
            onConfirm?.call();
            context.pop(true);
          },
          child: const Text('Confirm'),
        ),
      ],
    );
  }
}
```

---

**← [Part 58](part-58-design-system.md)**
**ต่อไป: [Part 60 →](part-60-error-handling.md)**
