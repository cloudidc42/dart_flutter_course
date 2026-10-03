# Part 61: Flutter Advanced Testing
## ขั้นตอนที่ 2321-2360

## 🎯 เป้าหมายของ Part นี้
- เรียนรู้ BLoC testing ด้วย bloc_test package
- ทดสอบ Riverpod ด้วย ProviderContainer
- Mock HTTP ด้วย http_mock_adapter
- Integration test ด้วย patrol package
- สร้าง Test fixtures และ factories

---

## ขั้นตอนที่ 2321: Project Setup & pubspec.yaml

```yaml
# pubspec.yaml
name: flutter_advanced_testing
description: Advanced Testing in Flutter

environment:
  sdk: ">=3.0.0 <4.0.0"
  flutter: ">=3.10.0"

dependencies:
  flutter:
    sdk: flutter
  flutter_bloc: ^8.1.3
  bloc: ^8.1.2
  flutter_riverpod: ^2.4.0
  riverpod: ^2.4.0
  dio: ^5.3.3
  equatable: ^2.0.5

dev_dependencies:
  flutter_test:
    sdk: flutter
  bloc_test: ^9.1.5
  mocktail: ^1.0.1
  http_mock_adapter: ^0.6.0
  patrol: ^2.6.0
  build_runner: ^2.4.6
  flutter_lints: ^3.0.0
```

---

## ขั้นตอนที่ 2322: Counter BLoC Implementation

```dart
// lib/blocs/counter/counter_event.dart
import 'package:equatable/equatable.dart';

abstract class CounterEvent extends Equatable {
  const CounterEvent();

  @override
  List<Object?> get props => [];
}

class CounterIncrementPressed extends CounterEvent {
  const CounterIncrementPressed();
}

class CounterDecrementPressed extends CounterEvent {
  const CounterDecrementPressed();
}

class CounterResetPressed extends CounterEvent {
  const CounterResetPressed();
}

class CounterIncrementByAmount extends CounterEvent {
  final int amount;
  const CounterIncrementByAmount(this.amount);

  @override
  List<Object?> get props => [amount];
}
```

```dart
// lib/blocs/counter/counter_state.dart
import 'package:equatable/equatable.dart';

enum CounterStatus { initial, loading, success, failure }

class CounterState extends Equatable {
  final int count;
  final CounterStatus status;
  final String? errorMessage;

  const CounterState({
    this.count = 0,
    this.status = CounterStatus.initial,
    this.errorMessage,
  });

  CounterState copyWith({
    int? count,
    CounterStatus? status,
    String? errorMessage,
  }) {
    return CounterState(
      count: count ?? this.count,
      status: status ?? this.status,
      errorMessage: errorMessage ?? this.errorMessage,
    );
  }

  @override
  List<Object?> get props => [count, status, errorMessage];

  @override
  String toString() => 'CounterState(count: $count, status: $status)';
}
```

```dart
// lib/blocs/counter/counter_bloc.dart
import 'package:bloc/bloc.dart';
import 'counter_event.dart';
import 'counter_state.dart';

class CounterBloc extends Bloc<CounterEvent, CounterState> {
  static const int maxCount = 100;
  static const int minCount = -100;

  CounterBloc() : super(const CounterState()) {
    on<CounterIncrementPressed>(_onIncrementPressed);
    on<CounterDecrementPressed>(_onDecrementPressed);
    on<CounterResetPressed>(_onResetPressed);
    on<CounterIncrementByAmount>(_onIncrementByAmount);
  }

  void _onIncrementPressed(
    CounterIncrementPressed event,
    Emitter<CounterState> emit,
  ) {
    if (state.count >= maxCount) {
      emit(state.copyWith(
        status: CounterStatus.failure,
        errorMessage: 'Maximum count reached: $maxCount',
      ));
      return;
    }
    emit(state.copyWith(
      count: state.count + 1,
      status: CounterStatus.success,
      errorMessage: null,
    ));
  }

  void _onDecrementPressed(
    CounterDecrementPressed event,
    Emitter<CounterState> emit,
  ) {
    if (state.count <= minCount) {
      emit(state.copyWith(
        status: CounterStatus.failure,
        errorMessage: 'Minimum count reached: $minCount',
      ));
      return;
    }
    emit(state.copyWith(
      count: state.count - 1,
      status: CounterStatus.success,
      errorMessage: null,
    ));
  }

  void _onResetPressed(
    CounterResetPressed event,
    Emitter<CounterState> emit,
  ) {
    emit(const CounterState());
  }

  void _onIncrementByAmount(
    CounterIncrementByAmount event,
    Emitter<CounterState> emit,
  ) {
    final newCount = state.count + event.amount;
    if (newCount > maxCount) {
      emit(state.copyWith(
        status: CounterStatus.failure,
        errorMessage: 'Would exceed maximum: $maxCount',
      ));
      return;
    }
    emit(state.copyWith(
      count: newCount,
      status: CounterStatus.success,
    ));
  }
}
```

---

## ขั้นตอนที่ 2323: BLoC Tests with bloc_test

```dart
// test/blocs/counter_bloc_test.dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_advanced_testing/blocs/counter/counter_bloc.dart';
import 'package:flutter_advanced_testing/blocs/counter/counter_event.dart';
import 'package:flutter_advanced_testing/blocs/counter/counter_state.dart';

void main() {
  group('CounterBloc', () {
    late CounterBloc counterBloc;

    setUp(() {
      counterBloc = CounterBloc();
    });

    tearDown(() {
      counterBloc.close();
    });

    test('initial state is CounterState with count 0', () {
      expect(counterBloc.state, const CounterState());
      expect(counterBloc.state.count, equals(0));
      expect(counterBloc.state.status, equals(CounterStatus.initial));
    });

    blocTest<CounterBloc, CounterState>(
      'emits [CounterState(count: 1)] when CounterIncrementPressed is added',
      build: () => CounterBloc(),
      act: (bloc) => bloc.add(const CounterIncrementPressed()),
      expect: () => [
        const CounterState(count: 1, status: CounterStatus.success),
      ],
    );

    blocTest<CounterBloc, CounterState>(
      'emits [CounterState(count: -1)] when CounterDecrementPressed is added',
      build: () => CounterBloc(),
      act: (bloc) => bloc.add(const CounterDecrementPressed()),
      expect: () => [
        const CounterState(count: -1, status: CounterStatus.success),
      ],
    );

    blocTest<CounterBloc, CounterState>(
      'emits initial state when CounterResetPressed is added',
      build: () => CounterBloc(),
      seed: () => const CounterState(count: 50, status: CounterStatus.success),
      act: (bloc) => bloc.add(const CounterResetPressed()),
      expect: () => [const CounterState()],
    );

    blocTest<CounterBloc, CounterState>(
      'emits failure state when count exceeds maximum',
      build: () => CounterBloc(),
      seed: () => const CounterState(
        count: 100,
        status: CounterStatus.success,
      ),
      act: (bloc) => bloc.add(const CounterIncrementPressed()),
      expect: () => [
        const CounterState(
          count: 100,
          status: CounterStatus.failure,
          errorMessage: 'Maximum count reached: 100',
        ),
      ],
    );

    blocTest<CounterBloc, CounterState>(
      'emits multiple states when multiple events are added',
      build: () => CounterBloc(),
      act: (bloc) {
        bloc
          ..add(const CounterIncrementPressed())
          ..add(const CounterIncrementPressed())
          ..add(const CounterIncrementPressed())
          ..add(const CounterDecrementPressed());
      },
      expect: () => [
        const CounterState(count: 1, status: CounterStatus.success),
        const CounterState(count: 2, status: CounterStatus.success),
        const CounterState(count: 3, status: CounterStatus.success),
        const CounterState(count: 2, status: CounterStatus.success),
      ],
    );

    blocTest<CounterBloc, CounterState>(
      'emits correct state when CounterIncrementByAmount is added',
      build: () => CounterBloc(),
      act: (bloc) => bloc.add(const CounterIncrementByAmount(10)),
      expect: () => [
        const CounterState(count: 10, status: CounterStatus.success),
      ],
    );

    blocTest<CounterBloc, CounterState>(
      'emits failure when increment by amount exceeds max',
      build: () => CounterBloc(),
      seed: () => const CounterState(count: 95, status: CounterStatus.success),
      act: (bloc) => bloc.add(const CounterIncrementByAmount(10)),
      expect: () => [
        const CounterState(
          count: 95,
          status: CounterStatus.failure,
          errorMessage: 'Would exceed maximum: 100',
        ),
      ],
    );

    test('state stream emits in order', () {
      expectLater(
        counterBloc.stream,
        emitsInOrder([
          const CounterState(count: 1, status: CounterStatus.success),
          const CounterState(count: 2, status: CounterStatus.success),
        ]),
      );
      counterBloc
        ..add(const CounterIncrementPressed())
        ..add(const CounterIncrementPressed());
    });
  });
}
```

---

## ขั้นตอนที่ 2324: Auth BLoC with Repository Pattern

```dart
// lib/repositories/auth_repository.dart
abstract class AuthRepository {
  Future<String> login({required String email, required String password});
  Future<void> logout();
  Future<bool> isLoggedIn();
}

class AuthRepositoryImpl implements AuthRepository {
  final String _baseUrl;

  AuthRepositoryImpl({required String baseUrl}) : _baseUrl = baseUrl;

  @override
  Future<String> login({
    required String email,
    required String password,
  }) async {
    await Future.delayed(const Duration(milliseconds: 500));
    if (email == 'test@example.com' && password == 'password123') {
      return 'mock_token_12345';
    }
    throw Exception('Invalid credentials');
  }

  @override
  Future<void> logout() async {
    await Future.delayed(const Duration(milliseconds: 200));
  }

  @override
  Future<bool> isLoggedIn() async {
    return false;
  }
}
```

```dart
// lib/blocs/auth/auth_bloc.dart
import 'package:bloc/bloc.dart';
import 'package:equatable/equatable.dart';
import '../../repositories/auth_repository.dart';

part 'auth_event.dart';
part 'auth_state.dart';

class AuthBloc extends Bloc<AuthEvent, AuthState> {
  final AuthRepository _authRepository;

  AuthBloc({required AuthRepository authRepository})
      : _authRepository = authRepository,
        super(const AuthInitial()) {
    on<AuthLoginRequested>(_onLoginRequested);
    on<AuthLogoutRequested>(_onLogoutRequested);
    on<AuthCheckRequested>(_onCheckRequested);
  }

  Future<void> _onLoginRequested(
    AuthLoginRequested event,
    Emitter<AuthState> emit,
  ) async {
    emit(const AuthLoading());
    try {
      final token = await _authRepository.login(
        email: event.email,
        password: event.password,
      );
      emit(AuthAuthenticated(token: token));
    } catch (e) {
      emit(AuthFailure(error: e.toString()));
    }
  }

  Future<void> _onLogoutRequested(
    AuthLogoutRequested event,
    Emitter<AuthState> emit,
  ) async {
    await _authRepository.logout();
    emit(const AuthInitial());
  }

  Future<void> _onCheckRequested(
    AuthCheckRequested event,
    Emitter<AuthState> emit,
  ) async {
    final isLoggedIn = await _authRepository.isLoggedIn();
    if (isLoggedIn) {
      emit(const AuthAuthenticated(token: 'existing_token'));
    } else {
      emit(const AuthInitial());
    }
  }
}
```

```dart
// lib/blocs/auth/auth_event.dart
part of 'auth_bloc.dart';

abstract class AuthEvent extends Equatable {
  const AuthEvent();
  @override
  List<Object?> get props => [];
}

class AuthLoginRequested extends AuthEvent {
  final String email;
  final String password;
  const AuthLoginRequested({required this.email, required this.password});
  @override
  List<Object?> get props => [email, password];
}

class AuthLogoutRequested extends AuthEvent {
  const AuthLogoutRequested();
}

class AuthCheckRequested extends AuthEvent {
  const AuthCheckRequested();
}
```

```dart
// lib/blocs/auth/auth_state.dart
part of 'auth_bloc.dart';

abstract class AuthState extends Equatable {
  const AuthState();
  @override
  List<Object?> get props => [];
}

class AuthInitial extends AuthState {
  const AuthInitial();
}

class AuthLoading extends AuthState {
  const AuthLoading();
}

class AuthAuthenticated extends AuthState {
  final String token;
  const AuthAuthenticated({required this.token});
  @override
  List<Object?> get props => [token];
}

class AuthFailure extends AuthState {
  final String error;
  const AuthFailure({required this.error});
  @override
  List<Object?> get props => [error];
}
```

---

## ขั้นตอนที่ 2325: Auth BLoC Tests with Mocktail

```dart
// test/blocs/auth_bloc_test.dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:flutter_advanced_testing/blocs/auth/auth_bloc.dart';
import 'package:flutter_advanced_testing/repositories/auth_repository.dart';

class MockAuthRepository extends Mock implements AuthRepository {}

void main() {
  group('AuthBloc', () {
    late AuthBloc authBloc;
    late MockAuthRepository mockAuthRepository;

    setUp(() {
      mockAuthRepository = MockAuthRepository();
      authBloc = AuthBloc(authRepository: mockAuthRepository);
    });

    tearDown(() {
      authBloc.close();
    });

    test('initial state is AuthInitial', () {
      expect(authBloc.state, isA<AuthInitial>());
    });

    group('AuthLoginRequested', () {
      blocTest<AuthBloc, AuthState>(
        'emits [AuthLoading, AuthAuthenticated] on successful login',
        build: () {
          when(
            () => mockAuthRepository.login(
              email: 'test@example.com',
              password: 'password123',
            ),
          ).thenAnswer((_) async => 'mock_token');
          return AuthBloc(authRepository: mockAuthRepository);
        },
        act: (bloc) => bloc.add(const AuthLoginRequested(
          email: 'test@example.com',
          password: 'password123',
        )),
        expect: () => [
          const AuthLoading(),
          const AuthAuthenticated(token: 'mock_token'),
        ],
        verify: (_) {
          verify(
            () => mockAuthRepository.login(
              email: 'test@example.com',
              password: 'password123',
            ),
          ).called(1);
        },
      );

      blocTest<AuthBloc, AuthState>(
        'emits [AuthLoading, AuthFailure] on login failure',
        build: () {
          when(
            () => mockAuthRepository.login(
              email: any(named: 'email'),
              password: any(named: 'password'),
            ),
          ).thenThrow(Exception('Invalid credentials'));
          return AuthBloc(authRepository: mockAuthRepository);
        },
        act: (bloc) => bloc.add(const AuthLoginRequested(
          email: 'wrong@example.com',
          password: 'wrongpass',
        )),
        expect: () => [
          const AuthLoading(),
          isA<AuthFailure>(),
        ],
      );
    });

    group('AuthLogoutRequested', () {
      blocTest<AuthBloc, AuthState>(
        'emits [AuthInitial] after logout',
        build: () {
          when(() => mockAuthRepository.logout())
              .thenAnswer((_) async {});
          return AuthBloc(authRepository: mockAuthRepository);
        },
        seed: () => const AuthAuthenticated(token: 'some_token'),
        act: (bloc) => bloc.add(const AuthLogoutRequested()),
        expect: () => [const AuthInitial()],
        verify: (_) {
          verify(() => mockAuthRepository.logout()).called(1);
        },
      );
    });
  });
}
```

---

## ขั้นตอนที่ 2326: Riverpod Providers Setup

```dart
// lib/providers/user_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:equatable/equatable.dart';

// Models
class User extends Equatable {
  final String id;
  final String name;
  final String email;
  final bool isPremium;

  const User({
    required this.id,
    required this.name,
    required this.email,
    this.isPremium = false,
  });

  User copyWith({
    String? id,
    String? name,
    String? email,
    bool? isPremium,
  }) {
    return User(
      id: id ?? this.id,
      name: name ?? this.name,
      email: email ?? this.email,
      isPremium: isPremium ?? this.isPremium,
    );
  }

  @override
  List<Object?> get props => [id, name, email, isPremium];
}

// Repository interface
abstract class UserRepository {
  Future<User> fetchUser(String id);
  Future<List<User>> fetchUsers();
  Future<void> updateUser(User user);
}

// Mock repository
class MockUserRepository implements UserRepository {
  final Map<String, User> _users = {
    '1': const User(id: '1', name: 'Alice', email: 'alice@example.com'),
    '2': const User(id: '2', name: 'Bob', email: 'bob@example.com', isPremium: true),
    '3': const User(id: '3', name: 'Charlie', email: 'charlie@example.com'),
  };

  @override
  Future<User> fetchUser(String id) async {
    await Future.delayed(const Duration(milliseconds: 100));
    final user = _users[id];
    if (user == null) throw Exception('User not found: $id');
    return user;
  }

  @override
  Future<List<User>> fetchUsers() async {
    await Future.delayed(const Duration(milliseconds: 200));
    return _users.values.toList();
  }

  @override
  Future<void> updateUser(User user) async {
    await Future.delayed(const Duration(milliseconds: 100));
    _users[user.id] = user;
  }
}

// Providers
final userRepositoryProvider = Provider<UserRepository>((ref) {
  return MockUserRepository();
});

final currentUserIdProvider = StateProvider<String?>((ref) => null);

final userProvider = FutureProvider.family<User, String>((ref, userId) async {
  final repository = ref.watch(userRepositoryProvider);
  return repository.fetchUser(userId);
});

final allUsersProvider = FutureProvider<List<User>>((ref) async {
  final repository = ref.watch(userRepositoryProvider);
  return repository.fetchUsers();
});

final premiumUsersProvider = Provider<AsyncValue<List<User>>>((ref) {
  return ref.watch(allUsersProvider).whenData(
    (users) => users.where((u) => u.isPremium).toList(),
  );
});
```

---

## ขั้นตอนที่ 2327: Riverpod Notifier for Complex State

```dart
// lib/providers/cart_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:equatable/equatable.dart';

class CartItem extends Equatable {
  final String id;
  final String name;
  final double price;
  final int quantity;

  const CartItem({
    required this.id,
    required this.name,
    required this.price,
    required this.quantity,
  });

  CartItem copyWith({int? quantity}) {
    return CartItem(
      id: id,
      name: name,
      price: price,
      quantity: quantity ?? this.quantity,
    );
  }

  double get subtotal => price * quantity;

  @override
  List<Object?> get props => [id, name, price, quantity];
}

class CartState extends Equatable {
  final List<CartItem> items;
  final bool isLoading;

  const CartState({
    this.items = const [],
    this.isLoading = false,
  });

  double get total => items.fold(0, (sum, item) => sum + item.subtotal);
  int get itemCount => items.fold(0, (sum, item) => sum + item.quantity);

  CartState copyWith({List<CartItem>? items, bool? isLoading}) {
    return CartState(
      items: items ?? this.items,
      isLoading: isLoading ?? this.isLoading,
    );
  }

  @override
  List<Object?> get props => [items, isLoading];
}

class CartNotifier extends Notifier<CartState> {
  @override
  CartState build() => const CartState();

  void addItem(CartItem item) {
    final existingIndex = state.items.indexWhere((i) => i.id == item.id);
    if (existingIndex >= 0) {
      final updatedItems = List<CartItem>.from(state.items);
      updatedItems[existingIndex] = updatedItems[existingIndex].copyWith(
        quantity: updatedItems[existingIndex].quantity + item.quantity,
      );
      state = state.copyWith(items: updatedItems);
    } else {
      state = state.copyWith(items: [...state.items, item]);
    }
  }

  void removeItem(String itemId) {
    state = state.copyWith(
      items: state.items.where((i) => i.id != itemId).toList(),
    );
  }

  void updateQuantity(String itemId, int quantity) {
    if (quantity <= 0) {
      removeItem(itemId);
      return;
    }
    state = state.copyWith(
      items: state.items.map((item) {
        return item.id == itemId ? item.copyWith(quantity: quantity) : item;
      }).toList(),
    );
  }

  void clearCart() {
    state = const CartState();
  }
}

final cartProvider = NotifierProvider<CartNotifier, CartState>(
  CartNotifier.new,
);
```

---

## ขั้นตอนที่ 2328: Riverpod Tests with ProviderContainer

```dart
// test/providers/riverpod_test.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:flutter_advanced_testing/providers/user_providers.dart';
import 'package:flutter_advanced_testing/providers/cart_providers.dart';

class MockUserRepository extends Mock implements UserRepository {}

void main() {
  group('UserProviders', () {
    test('userProvider returns correct user', () async {
      final mockRepo = MockUserRepository();
      when(() => mockRepo.fetchUser('1')).thenAnswer(
        (_) async => const User(
          id: '1',
          name: 'Alice',
          email: 'alice@example.com',
        ),
      );

      final container = ProviderContainer(
        overrides: [
          userRepositoryProvider.overrideWithValue(mockRepo),
        ],
      );
      addTearDown(container.dispose);

      final user = await container.read(userProvider('1').future);
      expect(user.name, equals('Alice'));
      expect(user.email, equals('alice@example.com'));

      verify(() => mockRepo.fetchUser('1')).called(1);
    });

    test('allUsersProvider returns all users', () async {
      final mockRepo = MockUserRepository();
      when(() => mockRepo.fetchUsers()).thenAnswer(
        (_) async => [
          const User(id: '1', name: 'Alice', email: 'alice@example.com'),
          const User(id: '2', name: 'Bob', email: 'bob@example.com', isPremium: true),
        ],
      );

      final container = ProviderContainer(
        overrides: [
          userRepositoryProvider.overrideWithValue(mockRepo),
        ],
      );
      addTearDown(container.dispose);

      final users = await container.read(allUsersProvider.future);
      expect(users.length, equals(2));
      expect(users.first.name, equals('Alice'));
    });

    test('userProvider throws on missing user', () async {
      final mockRepo = MockUserRepository();
      when(() => mockRepo.fetchUser('999'))
          .thenThrow(Exception('User not found: 999'));

      final container = ProviderContainer(
        overrides: [
          userRepositoryProvider.overrideWithValue(mockRepo),
        ],
      );
      addTearDown(container.dispose);

      expect(
        () => container.read(userProvider('999').future),
        throwsA(isA<Exception>()),
      );
    });
  });

  group('CartNotifier', () {
    late ProviderContainer container;

    setUp(() {
      container = ProviderContainer();
    });

    tearDown(() {
      container.dispose();
    });

    test('initial state is empty cart', () {
      final cart = container.read(cartProvider);
      expect(cart.items, isEmpty);
      expect(cart.total, equals(0.0));
      expect(cart.itemCount, equals(0));
    });

    test('addItem adds item to cart', () {
      container.read(cartProvider.notifier).addItem(
        const CartItem(
          id: 'p1',
          name: 'Apple',
          price: 1.99,
          quantity: 2,
        ),
      );

      final cart = container.read(cartProvider);
      expect(cart.items.length, equals(1));
      expect(cart.items.first.name, equals('Apple'));
      expect(cart.itemCount, equals(2));
      expect(cart.total, closeTo(3.98, 0.001));
    });

    test('addItem merges quantity for existing item', () {
      final notifier = container.read(cartProvider.notifier);
      notifier.addItem(
        const CartItem(id: 'p1', name: 'Apple', price: 1.99, quantity: 2),
      );
      notifier.addItem(
        const CartItem(id: 'p1', name: 'Apple', price: 1.99, quantity: 3),
      );

      final cart = container.read(cartProvider);
      expect(cart.items.length, equals(1));
      expect(cart.items.first.quantity, equals(5));
    });

    test('removeItem removes item from cart', () {
      final notifier = container.read(cartProvider.notifier);
      notifier.addItem(
        const CartItem(id: 'p1', name: 'Apple', price: 1.99, quantity: 2),
      );
      notifier.addItem(
        const CartItem(id: 'p2', name: 'Banana', price: 0.99, quantity: 3),
      );
      notifier.removeItem('p1');

      final cart = container.read(cartProvider);
      expect(cart.items.length, equals(1));
      expect(cart.items.first.id, equals('p2'));
    });

    test('updateQuantity updates item quantity', () {
      container.read(cartProvider.notifier).addItem(
        const CartItem(id: 'p1', name: 'Apple', price: 1.99, quantity: 2),
      );
      container.read(cartProvider.notifier).updateQuantity('p1', 5);

      expect(container.read(cartProvider).items.first.quantity, equals(5));
    });

    test('updateQuantity with 0 removes item', () {
      container.read(cartProvider.notifier).addItem(
        const CartItem(id: 'p1', name: 'Apple', price: 1.99, quantity: 2),
      );
      container.read(cartProvider.notifier).updateQuantity('p1', 0);

      expect(container.read(cartProvider).items, isEmpty);
    });

    test('clearCart empties the cart', () {
      final notifier = container.read(cartProvider.notifier);
      notifier.addItem(
        const CartItem(id: 'p1', name: 'Apple', price: 1.99, quantity: 2),
      );
      notifier.addItem(
        const CartItem(id: 'p2', name: 'Banana', price: 0.99, quantity: 1),
      );
      notifier.clearCart();

      expect(container.read(cartProvider).items, isEmpty);
    });
  });
}
```

---

## ขั้นตอนที่ 2329: HTTP Service with Dio

```dart
// lib/services/api_service.dart
import 'package:dio/dio.dart';

class Product {
  final int id;
  final String title;
  final double price;
  final String description;
  final String category;

  const Product({
    required this.id,
    required this.title,
    required this.price,
    required this.description,
    required this.category,
  });

  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id: json['id'] as int,
      title: json['title'] as String,
      price: (json['price'] as num).toDouble(),
      description: json['description'] as String,
      category: json['category'] as String,
    );
  }

  Map<String, dynamic> toJson() => {
        'id': id,
        'title': title,
        'price': price,
        'description': description,
        'category': category,
      };
}

class ApiService {
  final Dio _dio;
  static const String baseUrl = 'https://fakestoreapi.com';

  ApiService({Dio? dio})
      : _dio = dio ??
            Dio(
              BaseOptions(
                baseUrl: baseUrl,
                connectTimeout: const Duration(seconds: 10),
                receiveTimeout: const Duration(seconds: 10),
              ),
            );

  Future<List<Product>> getProducts() async {
    try {
      final response = await _dio.get('/products');
      final List<dynamic> data = response.data as List<dynamic>;
      return data.map((json) => Product.fromJson(json as Map<String, dynamic>)).toList();
    } on DioException catch (e) {
      throw _handleDioError(e);
    }
  }

  Future<Product> getProductById(int id) async {
    try {
      final response = await _dio.get('/products/$id');
      return Product.fromJson(response.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw _handleDioError(e);
    }
  }

  Future<List<Product>> getProductsByCategory(String category) async {
    try {
      final response = await _dio.get('/products/category/$category');
      final List<dynamic> data = response.data as List<dynamic>;
      return data.map((json) => Product.fromJson(json as Map<String, dynamic>)).toList();
    } on DioException catch (e) {
      throw _handleDioError(e);
    }
  }

  Exception _handleDioError(DioException e) {
    switch (e.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.receiveTimeout:
        return Exception('Connection timeout');
      case DioExceptionType.badResponse:
        return Exception('Server error: ${e.response?.statusCode}');
      default:
        return Exception('Network error: ${e.message}');
    }
  }
}
```

---

## ขั้นตอนที่ 2330: HTTP Mock Tests with http_mock_adapter

```dart
// test/services/api_service_test.dart
import 'package:dio/dio.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:http_mock_adapter/http_mock_adapter.dart';
import 'package:flutter_advanced_testing/services/api_service.dart';

void main() {
  late Dio dio;
  late DioAdapter dioAdapter;
  late ApiService apiService;

  setUp(() {
    dio = Dio(BaseOptions(baseUrl: 'https://fakestoreapi.com'));
    dioAdapter = DioAdapter(dio: dio);
    apiService = ApiService(dio: dio);
  });

  group('ApiService.getProducts', () {
    test('returns list of products on success', () async {
      dioAdapter.onGet(
        '/products',
        (server) => server.reply(
          200,
          [
            {
              'id': 1,
              'title': 'Test Product',
              'price': 29.99,
              'description': 'A test product',
              'category': 'electronics',
            },
            {
              'id': 2,
              'title': 'Another Product',
              'price': 9.99,
              'description': 'Another test product',
              'category': 'clothing',
            },
          ],
        ),
      );

      final products = await apiService.getProducts();
      expect(products.length, equals(2));
      expect(products.first.title, equals('Test Product'));
      expect(products.first.price, equals(29.99));
    });

    test('throws exception on server error', () async {
      dioAdapter.onGet(
        '/products',
        (server) => server.reply(500, {'message': 'Internal Server Error'}),
      );

      expect(() => apiService.getProducts(), throwsA(isA<Exception>()));
    });

    test('returns empty list when no products', () async {
      dioAdapter.onGet(
        '/products',
        (server) => server.reply(200, []),
      );

      final products = await apiService.getProducts();
      expect(products, isEmpty);
    });
  });

  group('ApiService.getProductById', () {
    test('returns product with correct id', () async {
      dioAdapter.onGet(
        '/products/1',
        (server) => server.reply(
          200,
          {
            'id': 1,
            'title': 'Specific Product',
            'price': 49.99,
            'description': 'A specific product',
            'category': 'electronics',
          },
        ),
      );

      final product = await apiService.getProductById(1);
      expect(product.id, equals(1));
      expect(product.title, equals('Specific Product'));
    });

    test('throws exception when product not found', () async {
      dioAdapter.onGet(
        '/products/999',
        (server) => server.reply(404, {'message': 'Not Found'}),
      );

      expect(
        () => apiService.getProductById(999),
        throwsA(isA<Exception>()),
      );
    });
  });

  group('ApiService.getProductsByCategory', () {
    test('returns products filtered by category', () async {
      dioAdapter.onGet(
        '/products/category/electronics',
        (server) => server.reply(
          200,
          [
            {
              'id': 1,
              'title': 'Phone',
              'price': 599.99,
              'description': 'A smartphone',
              'category': 'electronics',
            },
          ],
        ),
      );

      final products = await apiService.getProductsByCategory('electronics');
      expect(products.length, equals(1));
      expect(products.first.category, equals('electronics'));
    });
  });
}
```

---

## ขั้นตอนที่ 2331: Test Fixtures and Factories

```dart
// test/fixtures/product_fixture.dart
import 'package:flutter_advanced_testing/services/api_service.dart';
import 'package:flutter_advanced_testing/providers/cart_providers.dart';

class ProductFixture {
  static Product basic({
    int id = 1,
    String title = 'Test Product',
    double price = 9.99,
    String description = 'A test product description',
    String category = 'test',
  }) {
    return Product(
      id: id,
      title: title,
      price: price,
      description: description,
      category: category,
    );
  }

  static Product electronics({int id = 1}) => basic(
        id: id,
        title: 'Laptop Pro',
        price: 999.99,
        category: 'electronics',
      );

  static Product clothing({int id = 2}) => basic(
        id: id,
        title: 'Cotton T-Shirt',
        price: 19.99,
        category: 'clothing',
      );

  static List<Product> list({int count = 3}) {
    return List.generate(
      count,
      (i) => basic(
        id: i + 1,
        title: 'Product ${i + 1}',
        price: (i + 1) * 10.0,
        category: i.isEven ? 'electronics' : 'clothing',
      ),
    );
  }

  static Map<String, dynamic> toJson(Product product) => product.toJson();

  static List<Map<String, dynamic>> listToJson(List<Product> products) =>
      products.map(toJson).toList();
}

class CartItemFixture {
  static CartItem basic({
    String id = 'item1',
    String name = 'Test Item',
    double price = 9.99,
    int quantity = 1,
  }) {
    return CartItem(
      id: id,
      name: name,
      price: price,
      quantity: quantity,
    );
  }

  static CartItem withQuantity(int quantity) =>
      basic(quantity: quantity);

  static List<CartItem> list({int count = 3}) {
    return List.generate(
      count,
      (i) => basic(
        id: 'item${i + 1}',
        name: 'Item ${i + 1}',
        price: (i + 1) * 5.0,
        quantity: i + 1,
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2332: Widget Tests

```dart
// test/widgets/product_card_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_advanced_testing/services/api_service.dart';
import '../fixtures/product_fixture.dart';

class ProductCard extends StatelessWidget {
  final Product product;
  final VoidCallback? onAddToCart;

  const ProductCard({
    super.key,
    required this.product,
    this.onAddToCart,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  product.title,
                  key: const Key('product_title'),
                  style: Theme.of(context).textTheme.titleMedium,
                ),
                const SizedBox(height: 8),
                Text(
                  '\$${product.price.toStringAsFixed(2)}',
                  key: const Key('product_price'),
                  style: Theme.of(context).textTheme.bodyLarge?.copyWith(
                        color: Colors.green,
                        fontWeight: FontWeight.bold,
                      ),
                ),
                const SizedBox(height: 8),
                Text(
                  product.category,
                  key: const Key('product_category'),
                ),
              ],
            ),
          ),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: ElevatedButton(
              key: const Key('add_to_cart_button'),
              onPressed: onAddToCart,
              child: const Text('Add to Cart'),
            ),
          ),
        ],
      ),
    );
  }
}

void main() {
  group('ProductCard Widget', () {
    testWidgets('displays product information correctly', (tester) async {
      final product = ProductFixture.electronics();

      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: ProductCard(product: product),
          ),
        ),
      );

      expect(find.text(product.title), findsOneWidget);
      expect(find.text('\$${product.price.toStringAsFixed(2)}'), findsOneWidget);
      expect(find.text(product.category), findsOneWidget);
    });

    testWidgets('calls onAddToCart when button is pressed', (tester) async {
      final product = ProductFixture.basic();
      bool buttonPressed = false;

      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: ProductCard(
              product: product,
              onAddToCart: () => buttonPressed = true,
            ),
          ),
        ),
      );

      await tester.tap(find.byKey(const Key('add_to_cart_button')));
      await tester.pump();

      expect(buttonPressed, isTrue);
    });

    testWidgets('renders add to cart button', (tester) async {
      final product = ProductFixture.basic();

      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: ProductCard(product: product),
          ),
        ),
      );

      expect(find.byKey(const Key('add_to_cart_button')), findsOneWidget);
      expect(find.text('Add to Cart'), findsOneWidget);
    });
  });
}
```

---

## ขั้นตอนที่ 2333: Integration Test with patrol

```dart
// integration_test/app_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:patrol/patrol.dart';
import 'package:flutter_advanced_testing/main.dart' as app;

void main() {
  patrolTest(
    'Full app smoke test - counter increment',
    ($) async {
      app.main();
      await $.pumpAndSettle();

      // Verify initial state
      expect(find.text('0'), findsOneWidget);

      // Tap increment button
      await $('+').tap();
      await $.pumpAndSettle();

      expect(find.text('1'), findsOneWidget);

      // Tap increment button multiple times
      await $('+').tap();
      await $('+').tap();
      await $.pumpAndSettle();

      expect(find.text('3'), findsOneWidget);
    },
  );

  patrolTest(
    'Counter decrement works correctly',
    ($) async {
      app.main();
      await $.pumpAndSettle();

      await $('+').tap();
      await $('+').tap();
      await $('+').tap();
      await $.pumpAndSettle();

      await $('-').tap();
      await $.pumpAndSettle();

      expect(find.text('2'), findsOneWidget);
    },
  );
}
```

---

## ขั้นตอนที่ 2334: Main App Entry Point

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'blocs/counter/counter_bloc.dart';
import 'blocs/counter/counter_event.dart';
import 'blocs/counter/counter_state.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Advanced Testing',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: BlocProvider(
        create: (_) => CounterBloc(),
        child: const CounterPage(),
      ),
    );
  }
}

class CounterPage extends StatelessWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Counter'),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            BlocBuilder<CounterBloc, CounterState>(
              builder: (context, state) {
                if (state.status == CounterStatus.failure) {
                  return Column(
                    children: [
                      Text(
                        '${state.count}',
                        style: Theme.of(context).textTheme.displayMedium,
                      ),
                      const SizedBox(height: 8),
                      Text(
                        state.errorMessage ?? '',
                        style: const TextStyle(color: Colors.red),
                      ),
                    ],
                  );
                }
                return Text(
                  '${state.count}',
                  style: Theme.of(context).textTheme.displayMedium,
                );
              },
            ),
            const SizedBox(height: 24),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                FloatingActionButton(
                  heroTag: 'decrement',
                  onPressed: () => context
                      .read<CounterBloc>()
                      .add(const CounterDecrementPressed()),
                  child: const Text('-'),
                ),
                const SizedBox(width: 16),
                FloatingActionButton(
                  heroTag: 'increment',
                  onPressed: () => context
                      .read<CounterBloc>()
                      .add(const CounterIncrementPressed()),
                  child: const Text('+'),
                ),
              ],
            ),
            const SizedBox(height: 16),
            TextButton(
              onPressed: () => context
                  .read<CounterBloc>()
                  .add(const CounterResetPressed()),
              child: const Text('Reset'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

**← [Part 60](part-60-performance-optimization.md)**
**ต่อไป: [Part 62 →](part-62-security-best-practices.md)**
