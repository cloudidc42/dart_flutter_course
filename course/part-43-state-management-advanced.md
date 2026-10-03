# Part 43: Advanced State Management
## ขั้นตอนที่ 1601-1640

---

## 🎯 เป้าหมายของ Part นี้

- Riverpod Advanced (code gen, testing)
- HydratedBloc (persistent state)
- MobX
- Redux pattern
- State management comparison

---

## ขั้นตอนที่ 1601: Riverpod Advanced - Code Generation

```yaml
# pubspec.yaml
dependencies:
  flutter_riverpod: ^2.5.1
  riverpod_annotation: ^2.3.5

dev_dependencies:
  riverpod_generator: ^2.4.0
  build_runner: ^2.4.9
```

```dart
// lib/features/auth/providers/auth_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
part 'auth_provider.g.dart';

// ─── User Model ───
class User {
  final String id;
  final String name;
  final String email;
  const User({required this.id, required this.name, required this.email});
}

// ─── Auth State ───
class AuthState {
  final User? user;
  final bool isLoading;
  final String? error;

  const AuthState({this.user, this.isLoading = false, this.error});

  bool get isAuthenticated => user != null;

  AuthState copyWith({User? user, bool? isLoading, String? error}) => AuthState(
    user: user ?? this.user,
    isLoading: isLoading ?? this.isLoading,
    error: error,
  );
}

// ─── Auth Notifier (code gen) ───
@riverpod
class Auth extends _$Auth {
  @override
  AuthState build() => const AuthState();

  Future<void> login(String email, String password) async {
    state = state.copyWith(isLoading: true, error: null);

    try {
      await Future.delayed(const Duration(seconds: 1)); // API call
      User user = User(id: '1', name: 'John', email: email);
      state = state.copyWith(user: user, isLoading: false);
    } catch (e) {
      state = state.copyWith(isLoading: false, error: e.toString());
    }
  }

  Future<void> logout() async {
    state = const AuthState();
  }

  Future<void> updateProfile({String? name}) async {
    User? current = state.user;
    if (current == null) return;

    state = state.copyWith(isLoading: true);
    await Future.delayed(const Duration(milliseconds: 500));

    User updated = User(id: current.id, name: name ?? current.name, email: current.email);
    state = state.copyWith(user: updated, isLoading: false);
  }
}

// ─── Derived providers ───
@riverpod
bool isLoggedIn(IsLoggedInRef ref) {
  return ref.watch(authProvider).isAuthenticated;
}

@riverpod
User? currentUser(CurrentUserRef ref) {
  return ref.watch(authProvider).user;
}

// ─── Async provider with cancellation ───
@riverpod
Future<List<String>> userPosts(UserPostsRef ref, String userId) async {
  ref.onDispose(() => print('Provider disposed, cancel any requests'));

  await Future.delayed(const Duration(seconds: 1));
  return List.generate(10, (i) => 'Post $i by $userId');
}

// ─── Family provider ───
@riverpod
Future<Map<String, dynamic>> productDetails(
  ProductDetailsRef ref,
  String productId,
) async {
  await Future.delayed(const Duration(milliseconds: 500));
  return {'id': productId, 'name': 'Product $productId', 'price': 99.99};
}

// ─── Stream provider ───
@riverpod
Stream<int> counter(CounterRef ref) async* {
  int count = 0;
  while (true) {
    await Future.delayed(const Duration(seconds: 1));
    yield count++;
  }
}
```

---

## ขั้นตอนที่ 1602: Riverpod Testing

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:riverpod/riverpod.dart';

// ─── Testing Riverpod providers ───
void main() {
  group('Auth Tests', () {
    test('initial state is unauthenticated', () {
      ProviderContainer container = ProviderContainer();
      addTearDown(container.dispose);

      AuthState state = container.read(authProvider);
      expect(state.isAuthenticated, false);
      expect(state.isLoading, false);
      expect(state.error, null);
    });

    test('login sets user', () async {
      ProviderContainer container = ProviderContainer();
      addTearDown(container.dispose);

      await container.read(authProvider.notifier).login('test@test.com', 'password');

      AuthState state = container.read(authProvider);
      expect(state.isAuthenticated, true);
      expect(state.user?.email, 'test@test.com');
    });

    test('logout clears user', () async {
      ProviderContainer container = ProviderContainer();
      addTearDown(container.dispose);

      await container.read(authProvider.notifier).login('test@test.com', 'pass');
      await container.read(authProvider.notifier).logout();

      AuthState state = container.read(authProvider);
      expect(state.isAuthenticated, false);
    });

    test('override provider for testing', () async {
      // Mock the auth service
      ProviderContainer container = ProviderContainer(
        overrides: [
          // Override specific providers
          // authServiceProvider.overrideWithValue(MockAuthService()),
        ],
      );
      addTearDown(container.dispose);

      AuthState state = container.read(authProvider);
      expect(state, isNotNull);
    });
  });

  group('Widget Tests with Riverpod', () {
    testWidgets('shows user name when logged in', (tester) async {
      await tester.pumpWidget(
        ProviderScope(
          overrides: [
            authProvider.overrideWith(() => _MockAuthNotifier()),
          ],
          child: const MaterialApp(home: ProfileScreen()),
        ),
      );

      await tester.pump();
      expect(find.text('John Doe'), findsOneWidget);
    });
  });
}

// Mock notifier for testing
class _MockAuthNotifier extends Auth {
  @override
  AuthState build() => AuthState(
    user: User(id: '1', name: 'John Doe', email: 'john@test.com'),
  );
}

class ProfileScreen extends ConsumerWidget {
  const ProfileScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    User? user = ref.watch(currentUserProvider);
    return Scaffold(
      body: Center(child: Text(user?.name ?? 'Not logged in')),
    );
  }
}
```

---

## ขั้นตอนที่ 1603: HydratedBloc (Persistent BLoC)

```yaml
# pubspec.yaml
dependencies:
  hydrated_bloc: ^9.1.5
  path_provider: ^2.1.2
```

```dart
import 'dart:convert';
import 'package:hydrated_bloc/hydrated_bloc.dart';
import 'package:path_provider/path_provider.dart';

// ─── Setup ───
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  HydratedBloc.storage = await HydratedStorage.build(
    storageDirectory: await getApplicationDocumentsDirectory(),
  );

  runApp(const MyApp());
}

// ─── Theme Cubit (persistent) ───
class ThemeState {
  final bool isDark;
  const ThemeState({this.isDark = false});
}

class ThemeCubit extends HydratedCubit<ThemeState> {
  ThemeCubit() : super(const ThemeState());

  void toggleTheme() => emit(ThemeState(isDark: !state.isDark));
  void setDark(bool isDark) => emit(ThemeState(isDark: isDark));

  @override
  ThemeState? fromJson(Map<String, dynamic> json) {
    return ThemeState(isDark: json['isDark'] as bool? ?? false);
  }

  @override
  Map<String, dynamic>? toJson(ThemeState state) {
    return {'isDark': state.isDark};
  }
}

// ─── Cart Cubit (persistent) ───
class CartItem {
  final String id;
  final String name;
  final double price;
  int quantity;

  CartItem({
    required this.id,
    required this.name,
    required this.price,
    this.quantity = 1,
  });

  Map<String, dynamic> toJson() => {
    'id': id, 'name': name, 'price': price, 'quantity': quantity,
  };

  factory CartItem.fromJson(Map<String, dynamic> json) => CartItem(
    id: json['id'], name: json['name'],
    price: json['price'].toDouble(), quantity: json['quantity'],
  );
}

class CartCubit extends HydratedCubit<List<CartItem>> {
  CartCubit() : super([]);

  void addItem(CartItem item) {
    List<CartItem> items = [...state];
    int existingIndex = items.indexWhere((i) => i.id == item.id);

    if (existingIndex >= 0) {
      items[existingIndex].quantity++;
      emit([...items]);
    } else {
      emit([...items, item]);
    }
  }

  void removeItem(String id) {
    emit(state.where((i) => i.id != id).toList());
  }

  void decreaseQuantity(String id) {
    List<CartItem> items = [...state];
    int index = items.indexWhere((i) => i.id == id);
    if (index < 0) return;

    if (items[index].quantity > 1) {
      items[index].quantity--;
      emit([...items]);
    } else {
      removeItem(id);
    }
  }

  void clear() => emit([]);

  double get total => state.fold(0, (sum, item) => sum + item.price * item.quantity);
  int get totalItems => state.fold(0, (sum, item) => sum + item.quantity);

  @override
  List<CartItem>? fromJson(Map<String, dynamic> json) {
    List items = json['items'] ?? [];
    return items.map((i) => CartItem.fromJson(i)).toList();
  }

  @override
  Map<String, dynamic>? toJson(List<CartItem> state) {
    return {'items': state.map((i) => i.toJson()).toList()};
  }
}

// ─── Usage Widget ───
class CartBadge extends StatelessWidget {
  final Widget child;
  const CartBadge({super.key, required this.child});

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<CartCubit, List<CartItem>>(
      builder: (context, items) {
        int total = items.fold(0, (sum, item) => sum + item.quantity);
        return Badge(
          isLabelVisible: total > 0,
          label: Text('$total'),
          child: child,
        );
      },
    );
  }
}
```

---

## ขั้นตอนที่ 1604: Comparison - When to Use What

```dart
/*
 State Management Comparison
 
 ─── setState ───
 ✅ Simple local state
 ✅ Single widget
 ✅ No external dependencies
 ❌ No state sharing between widgets
 
 Example: form input, toggle, counter
 
 ─── Provider / ChangeNotifier ───
 ✅ Medium complexity
 ✅ Share state between widgets
 ✅ Easy to learn
 ❌ Can rebuild too often
 
 Example: user settings, shopping cart
 
 ─── Riverpod ───
 ✅ Modern, no BuildContext needed
 ✅ Code generation
 ✅ Easy testing
 ✅ Auto dispose
 ✅ Best for complex async
 
 Example: Large apps, async state, complex dependencies
 
 ─── BLoC/Cubit ───
 ✅ Explicit events/states
 ✅ Excellent testability
 ✅ Good for complex business logic
 ✅ HydratedBloc for persistence
 ❌ More boilerplate
 
 Example: Auth flows, forms, complex state machines
*/

// ─── Decision Guide ───
enum StateManagementChoice {
  setState,      // Simple, local
  provider,      // Simple sharing
  riverpod,      // Modern, complex
  bloc,          // Enterprise, testable
}

StateManagementChoice recommend({
  required bool isComplex,
  required bool needsTesting,
  required bool shareAcrossWidgets,
  required bool hasAsyncOps,
  required bool needsPersistence,
}) {
  if (!shareAcrossWidgets && !isComplex) return StateManagementChoice.setState;
  if (needsPersistence || needsTesting) return StateManagementChoice.bloc;
  if (hasAsyncOps || isComplex) return StateManagementChoice.riverpod;
  return StateManagementChoice.provider;
}
```

---

**← [Part 42 - Background Tasks](part-42-background-tasks.md)**

**ต่อไป: [Part 44 - GraphQL →](part-44-graphql.md)**
