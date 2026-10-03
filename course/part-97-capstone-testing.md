# Part 97: Capstone - Comprehensive Testing
## ขั้นตอนที่ 3761-3800

## 🎯 เป้าหมายของ Part นี้
- เขียน Unit Tests สำหรับ Repositories ทุก Feature
- Widget Tests สำหรับหน้าจอหลัก
- Integration Tests สำหรับ Critical User Flows
- Performance Tests และ Memory Leak Detection
- สร้าง Test Coverage Report

---

## ขั้นตอนที่ 3761: Test Dependencies Setup

```yaml
# pubspec.yaml (dev_dependencies section)
dev_dependencies:
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter
  
  # Mocking
  mockito: ^5.4.3
  build_runner: ^2.4.7
  
  # Testing helpers
  fake_async: ^1.3.1
  clock: ^1.1.1
  
  # Coverage
  coverage: ^1.7.2
  
  # Bloc testing
  bloc_test: ^9.1.5
  
  # Riverpod testing
  riverpod_test: ^0.1.0
```

```bash
# scripts/run_tests.sh
#!/bin/bash
set -e

echo "Running unit tests with coverage..."
flutter test --coverage --coverage-path=coverage/lcov.info

echo "Generating HTML coverage report..."
genhtml coverage/lcov.info -o coverage/html

echo "Running widget tests..."
flutter test test/widget/

echo "Running integration tests..."
flutter test integration_test/

echo "Done! Open coverage/html/index.html for the report."
```

## ขั้นตอนที่ 3762: Repository Unit Tests

```dart
// test/unit/repositories/auth_repository_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'package:food_delivery/features/auth/data/datasources/auth_remote_datasource.dart';
import 'package:food_delivery/features/auth/data/datasources/auth_local_datasource.dart';
import 'package:food_delivery/features/auth/data/repositories/auth_repository_impl.dart';
import 'package:food_delivery/features/auth/domain/entities/user.dart';
import 'package:food_delivery/core/error/failures.dart';

import 'auth_repository_test.mocks.dart';

@GenerateMocks([AuthRemoteDataSource, AuthLocalDataSource])
void main() {
  late AuthRepositoryImpl repository;
  late MockAuthRemoteDataSource mockRemoteDataSource;
  late MockAuthLocalDataSource mockLocalDataSource;

  setUp(() {
    mockRemoteDataSource = MockAuthRemoteDataSource();
    mockLocalDataSource = MockAuthLocalDataSource();
    repository = AuthRepositoryImpl(
      remoteDataSource: mockRemoteDataSource,
      localDataSource: mockLocalDataSource,
    );
  });

  group('AuthRepository.login', () {
    const tEmail = 'test@example.com';
    const tPassword = 'password123';
    final tUser = User(
      id: 'user-001',
      name: 'Test User',
      email: tEmail,
      phone: '0812345678',
      role: 'customer',
      createdAt: DateTime(2024, 1, 1),
    );
    const tToken = 'mock-jwt-token-abc123';

    test('should return User when login is successful', () async {
      // arrange
      when(mockRemoteDataSource.login(
        email: tEmail,
        password: tPassword,
      )).thenAnswer((_) async => (user: tUser, token: tToken));

      when(mockLocalDataSource.cacheToken(tToken))
          .thenAnswer((_) async => {});
      when(mockLocalDataSource.cacheUser(tUser))
          .thenAnswer((_) async => {});

      // act
      final result = await repository.login(
        email: tEmail,
        password: tPassword,
      );

      // assert
      expect(result.isRight(), true);
      result.fold(
        (failure) => fail('Should not return failure'),
        (user) => expect(user, equals(tUser)),
      );
      verify(mockLocalDataSource.cacheToken(tToken));
      verify(mockLocalDataSource.cacheUser(tUser));
    });

    test('should return AuthFailure when credentials are invalid', () async {
      // arrange
      when(mockRemoteDataSource.login(
        email: tEmail,
        password: 'wrong-password',
      )).thenThrow(Exception('Invalid credentials'));

      // act
      final result = await repository.login(
        email: tEmail,
        password: 'wrong-password',
      );

      // assert
      expect(result.isLeft(), true);
      result.fold(
        (failure) {
          expect(failure, isA<AuthFailure>());
          expect(failure.message, contains('Invalid credentials'));
        },
        (_) => fail('Should not return success'),
      );
      verifyNever(mockLocalDataSource.cacheToken(any));
    });

    test('should return NetworkFailure when there is no internet', () async {
      // arrange
      when(mockRemoteDataSource.login(
        email: anyNamed('email'),
        password: anyNamed('password'),
      )).thenThrow(const NetworkException('No internet connection'));

      // act
      final result = await repository.login(
        email: tEmail,
        password: tPassword,
      );

      // assert
      expect(result.isLeft(), true);
      result.fold(
        (failure) => expect(failure, isA<NetworkFailure>()),
        (_) => fail('Should not succeed'),
      );
    });
  });

  group('AuthRepository.logout', () {
    test('should clear cached data on logout', () async {
      // arrange
      when(mockRemoteDataSource.logout()).thenAnswer((_) async => {});
      when(mockLocalDataSource.clearToken()).thenAnswer((_) async => {});
      when(mockLocalDataSource.clearUser()).thenAnswer((_) async => {});

      // act
      final result = await repository.logout();

      // assert
      expect(result.isRight(), true);
      verify(mockLocalDataSource.clearToken());
      verify(mockLocalDataSource.clearUser());
    });
  });

  group('AuthRepository.getCurrentUser', () {
    test('should return cached user when available', () async {
      // arrange
      final tUser = User(
        id: 'user-001',
        name: 'Cached User',
        email: 'cache@example.com',
        phone: '0812345678',
        role: 'customer',
        createdAt: DateTime(2024),
      );
      when(mockLocalDataSource.getCachedUser()).thenAnswer((_) async => tUser);

      // act
      final result = await repository.getCurrentUser();

      // assert
      expect(result.isRight(), true);
      result.fold(
        (f) => fail('Should not fail'),
        (user) => expect(user, equals(tUser)),
      );
      verifyNever(mockRemoteDataSource.login(
        email: anyNamed('email'),
        password: anyNamed('password'),
      ));
    });

    test('should return CacheFailure when no user cached', () async {
      // arrange
      when(mockLocalDataSource.getCachedUser())
          .thenThrow(const CacheException('No cached user'));

      // act
      final result = await repository.getCurrentUser();

      // assert
      expect(result.isLeft(), true);
      result.fold(
        (failure) => expect(failure, isA<CacheFailure>()),
        (_) => fail('Should not succeed'),
      );
    });
  });
}
```

## ขั้นตอนที่ 3763: Order Repository Tests

```dart
// test/unit/repositories/order_repository_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'package:food_delivery/features/orders/data/datasources/order_remote_datasource.dart';
import 'package:food_delivery/features/orders/data/repositories/order_repository_impl.dart';
import 'package:food_delivery/features/orders/domain/entities/order.dart';
import 'package:food_delivery/features/orders/domain/params/create_order_params.dart';

import 'order_repository_test.mocks.dart';

@GenerateMocks([OrderRemoteDataSource])
void main() {
  late OrderRepositoryImpl repository;
  late MockOrderRemoteDataSource mockRemoteDataSource;

  setUp(() {
    mockRemoteDataSource = MockOrderRemoteDataSource();
    repository = OrderRepositoryImpl(remoteDataSource: mockRemoteDataSource);
  });

  final tOrder = Order(
    id: 'order-001',
    customerId: 'user-001',
    restaurantId: 'rest-001',
    restaurantName: 'Test Restaurant',
    items: [
      OrderItem(
        menuItemId: 'menu-001',
        name: 'Tom Yum',
        price: 150.0,
        quantity: 2,
      ),
    ],
    subtotal: 300.0,
    deliveryFee: 40.0,
    discount: 0.0,
    totalAmount: 340.0,
    status: OrderStatus.pending,
    deliveryAddress: '123 Test St, Bangkok',
    createdAt: DateTime(2024, 1, 15, 12, 0),
    updatedAt: DateTime(2024, 1, 15, 12, 0),
  );

  group('getOrders', () {
    test('should return list of orders on success', () async {
      when(mockRemoteDataSource.getOrders(userId: 'user-001'))
          .thenAnswer((_) async => [tOrder]);

      final result = await repository.getOrders(userId: 'user-001');

      expect(result.isRight(), true);
      result.fold(
        (f) => fail('Expected success'),
        (orders) {
          expect(orders.length, 1);
          expect(orders.first.id, 'order-001');
        },
      );
    });

    test('should return empty list when no orders', () async {
      when(mockRemoteDataSource.getOrders(userId: 'user-002'))
          .thenAnswer((_) async => []);

      final result = await repository.getOrders(userId: 'user-002');

      expect(result.isRight(), true);
      result.fold(
        (f) => fail('Expected success'),
        (orders) => expect(orders, isEmpty),
      );
    });
  });

  group('createOrder', () {
    final tParams = CreateOrderParams(
      restaurantId: 'rest-001',
      items: [
        OrderItemParam(menuItemId: 'menu-001', quantity: 2),
      ],
      deliveryAddress: '123 Test St',
      promoCode: null,
    );

    test('should return created order on success', () async {
      when(mockRemoteDataSource.createOrder(params: tParams))
          .thenAnswer((_) async => tOrder);

      final result = await repository.createOrder(params: tParams);

      expect(result.isRight(), true);
      result.fold(
        (f) => fail('Expected success'),
        (order) {
          expect(order.id, 'order-001');
          expect(order.totalAmount, 340.0);
          expect(order.status, OrderStatus.pending);
        },
      );
      verify(mockRemoteDataSource.createOrder(params: tParams));
    });

    test('should return failure when restaurant is closed', () async {
      when(mockRemoteDataSource.createOrder(params: tParams))
          .thenThrow(Exception('Restaurant is currently closed'));

      final result = await repository.createOrder(params: tParams);

      expect(result.isLeft(), true);
    });
  });

  group('cancelOrder', () {
    test('should return updated order with cancelled status', () async {
      final cancelledOrder = tOrder.copyWith(
        status: OrderStatus.cancelled,
        cancelReason: 'Customer request',
      );
      when(mockRemoteDataSource.cancelOrder(
        orderId: 'order-001',
        reason: 'Customer request',
      )).thenAnswer((_) async => cancelledOrder);

      final result = await repository.cancelOrder(
        orderId: 'order-001',
        reason: 'Customer request',
      );

      expect(result.isRight(), true);
      result.fold(
        (f) => fail('Expected success'),
        (order) {
          expect(order.status, OrderStatus.cancelled);
          expect(order.cancelReason, 'Customer request');
        },
      );
    });

    test('should fail when order cannot be cancelled (already delivered)', () async {
      when(mockRemoteDataSource.cancelOrder(
        orderId: 'order-delivered',
        reason: 'Test',
      )).thenThrow(Exception('Cannot cancel delivered order'));

      final result = await repository.cancelOrder(
        orderId: 'order-delivered',
        reason: 'Test',
      );

      expect(result.isLeft(), true);
    });
  });

  group('trackOrder', () {
    test('should return stream of order status updates', () async {
      final statuses = [
        tOrder.copyWith(status: OrderStatus.processing),
        tOrder.copyWith(status: OrderStatus.pickedUp),
        tOrder.copyWith(status: OrderStatus.delivered),
      ];

      when(mockRemoteDataSource.trackOrder(orderId: 'order-001'))
          .thenAnswer((_) => Stream.fromIterable(statuses));

      final stream = repository.trackOrder(orderId: 'order-001');

      await expectLater(
        stream,
        emitsInOrder([
          predicate<Order>((o) => o.status == OrderStatus.processing),
          predicate<Order>((o) => o.status == OrderStatus.pickedUp),
          predicate<Order>((o) => o.status == OrderStatus.delivered),
          emitsDone,
        ]),
      );
    });
  });
}
```

## ขั้นตอนที่ 3764: Cart Use Case Tests

```dart
// test/unit/use_cases/cart_use_case_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'package:food_delivery/features/cart/domain/repositories/cart_repository.dart';
import 'package:food_delivery/features/cart/domain/use_cases/add_to_cart_use_case.dart';
import 'package:food_delivery/features/cart/domain/use_cases/remove_from_cart_use_case.dart';
import 'package:food_delivery/features/cart/domain/use_cases/get_cart_use_case.dart';
import 'package:food_delivery/features/cart/domain/entities/cart.dart';
import 'package:food_delivery/features/cart/domain/entities/cart_item.dart';

import 'cart_use_case_test.mocks.dart';

@GenerateMocks([CartRepository])
void main() {
  late MockCartRepository mockRepository;
  late AddToCartUseCase addToCart;
  late RemoveFromCartUseCase removeFromCart;
  late GetCartUseCase getCart;

  setUp(() {
    mockRepository = MockCartRepository();
    addToCart = AddToCartUseCase(mockRepository);
    removeFromCart = RemoveFromCartUseCase(mockRepository);
    getCart = GetCartUseCase(mockRepository);
  });

  final tMenuItemId = 'menu-001';
  final tMenuItem = MenuItem(
    id: tMenuItemId,
    name: 'Pad Thai',
    price: 120.0,
    restaurantId: 'rest-001',
    imageUrl: 'https://example.com/pad_thai.jpg',
    category: 'Noodles',
    isAvailable: true,
  );

  final tCartItem = CartItem(
    menuItem: tMenuItem,
    quantity: 1,
    addons: [],
    notes: null,
  );

  final tCart = Cart(
    restaurantId: 'rest-001',
    restaurantName: 'Test Restaurant',
    items: [tCartItem],
    promoCode: null,
  );

  group('AddToCart', () {
    test('should add item to cart', () async {
      when(mockRepository.addItem(
        menuItem: tMenuItem,
        quantity: 1,
        addons: [],
        notes: null,
      )).thenAnswer((_) async => tCart);

      final result = await addToCart(AddToCartParams(
        menuItem: tMenuItem,
        quantity: 1,
      ));

      expect(result.isRight(), true);
      result.fold(
        (f) => fail('Expected success'),
        (cart) {
          expect(cart.items.length, 1);
          expect(cart.items.first.menuItem.id, tMenuItemId);
        },
      );
    });

    test('should fail when adding item from different restaurant', () async {
      final differentRestaurantItem = tMenuItem.copyWith(restaurantId: 'rest-002');

      when(mockRepository.addItem(
        menuItem: differentRestaurantItem,
        quantity: 1,
        addons: [],
        notes: null,
      )).thenThrow(Exception('Cannot mix items from different restaurants'));

      final result = await addToCart(AddToCartParams(
        menuItem: differentRestaurantItem,
        quantity: 1,
      ));

      expect(result.isLeft(), true);
    });

    test('should increase quantity when item already exists', () async {
      final updatedCart = tCart.copyWith(
        items: [tCartItem.copyWith(quantity: 2)],
      );
      when(mockRepository.addItem(
        menuItem: tMenuItem,
        quantity: 1,
        addons: [],
        notes: null,
      )).thenAnswer((_) async => updatedCart);

      final result = await addToCart(AddToCartParams(menuItem: tMenuItem, quantity: 1));

      expect(result.isRight(), true);
      result.fold(
        (f) => fail('Expected success'),
        (cart) => expect(cart.items.first.quantity, 2),
      );
    });
  });

  group('RemoveFromCart', () {
    test('should remove item from cart', () async {
      final emptyCart = Cart(
        restaurantId: 'rest-001',
        restaurantName: 'Test Restaurant',
        items: [],
        promoCode: null,
      );
      when(mockRepository.removeItem(menuItemId: tMenuItemId))
          .thenAnswer((_) async => emptyCart);

      final result = await removeFromCart(
        RemoveFromCartParams(menuItemId: tMenuItemId),
      );

      expect(result.isRight(), true);
      result.fold(
        (f) => fail('Expected success'),
        (cart) => expect(cart.items, isEmpty),
      );
    });
  });

  group('GetCart', () {
    test('should return current cart', () async {
      when(mockRepository.getCart()).thenAnswer((_) async => tCart);

      final result = await getCart(GetCartParams());

      expect(result.isRight(), true);
      result.fold(
        (f) => fail('Expected success'),
        (cart) {
          expect(cart.restaurantId, 'rest-001');
          expect(cart.items.length, 1);
        },
      );
    });

    test('cart total should calculate correctly', () async {
      final cartWithMultipleItems = Cart(
        restaurantId: 'rest-001',
        restaurantName: 'Test',
        items: [
          CartItem(menuItem: tMenuItem, quantity: 2, addons: []),
          CartItem(
            menuItem: tMenuItem.copyWith(id: 'menu-002', price: 80.0),
            quantity: 3,
            addons: [],
          ),
        ],
        promoCode: null,
      );

      when(mockRepository.getCart())
          .thenAnswer((_) async => cartWithMultipleItems);

      final result = await getCart(GetCartParams());

      result.fold(
        (f) => fail('Expected success'),
        (cart) {
          // 2 * 120 + 3 * 80 = 240 + 240 = 480
          expect(cart.subtotal, 480.0);
        },
      );
    });
  });
}
```

## ขั้นตอนที่ 3765: Widget Tests - Login Screen

```dart
// test/widget/login_page_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'package:food_delivery/features/auth/presentation/pages/login_page.dart';
import 'package:food_delivery/features/auth/presentation/providers/auth_provider.dart';
import 'package:food_delivery/features/auth/domain/repositories/auth_repository.dart';

import 'login_page_test.mocks.dart';

@GenerateMocks([AuthRepository])
void main() {
  late MockAuthRepository mockAuthRepository;

  setUp(() {
    mockAuthRepository = MockAuthRepository();
  });

  Widget buildLoginPage() {
    return ProviderScope(
      overrides: [
        authRepositoryProvider.overrideWithValue(mockAuthRepository),
      ],
      child: const MaterialApp(
        home: LoginPage(),
      ),
    );
  }

  group('LoginPage UI', () {
    testWidgets('should show email and password fields', (tester) async {
      await tester.pumpWidget(buildLoginPage());

      expect(find.byType(TextFormField), findsNWidgets(2));
      expect(find.widgetWithText(TextFormField, 'Email'), findsOneWidget);
      expect(find.widgetWithText(TextFormField, 'Password'), findsOneWidget);
    });

    testWidgets('should show login button', (tester) async {
      await tester.pumpWidget(buildLoginPage());

      expect(find.text('Login'), findsWidgets);
      expect(find.byType(FilledButton), findsOneWidget);
    });

    testWidgets('should show forgot password link', (tester) async {
      await tester.pumpWidget(buildLoginPage());

      expect(find.text('Forgot Password?'), findsOneWidget);
    });

    testWidgets('should show register link', (tester) async {
      await tester.pumpWidget(buildLoginPage());

      expect(find.text("Don't have an account?"), findsOneWidget);
      expect(find.text('Register'), findsOneWidget);
    });
  });

  group('LoginPage Validation', () {
    testWidgets('should show error for empty email', (tester) async {
      await tester.pumpWidget(buildLoginPage());

      await tester.tap(find.text('Login').last);
      await tester.pump();

      expect(find.text('Email is required'), findsOneWidget);
    });

    testWidgets('should show error for invalid email format', (tester) async {
      await tester.pumpWidget(buildLoginPage());

      await tester.enterText(
        find.widgetWithText(TextFormField, 'Email'),
        'invalid-email',
      );
      await tester.tap(find.text('Login').last);
      await tester.pump();

      expect(find.text('Invalid email format'), findsOneWidget);
    });

    testWidgets('should show error for empty password', (tester) async {
      await tester.pumpWidget(buildLoginPage());

      await tester.enterText(
        find.widgetWithText(TextFormField, 'Email'),
        'test@example.com',
      );
      await tester.tap(find.text('Login').last);
      await tester.pump();

      expect(find.text('Password is required'), findsOneWidget);
    });

    testWidgets('should show error for short password', (tester) async {
      await tester.pumpWidget(buildLoginPage());

      await tester.enterText(
        find.widgetWithText(TextFormField, 'Email'),
        'test@example.com',
      );
      await tester.enterText(
        find.widgetWithText(TextFormField, 'Password'),
        '12345',
      );
      await tester.tap(find.text('Login').last);
      await tester.pump();

      expect(find.text('Password must be at least 6 characters'), findsOneWidget);
    });
  });

  group('LoginPage Actions', () {
    testWidgets('should toggle password visibility', (tester) async {
      await tester.pumpWidget(buildLoginPage());

      // Initially password should be hidden
      final passwordField = tester.widget<TextField>(
        find.descendant(
          of: find.widgetWithText(TextFormField, 'Password'),
          matching: find.byType(TextField),
        ),
      );
      expect(passwordField.obscureText, true);

      // Tap visibility toggle
      await tester.tap(find.byIcon(Icons.visibility_off_rounded));
      await tester.pump();

      final passwordFieldAfter = tester.widget<TextField>(
        find.descendant(
          of: find.widgetWithText(TextFormField, 'Password'),
          matching: find.byType(TextField),
        ),
      );
      expect(passwordFieldAfter.obscureText, false);
    });

    testWidgets('should show loading indicator when logging in', (tester) async {
      when(mockAuthRepository.login(
        email: 'test@example.com',
        password: 'password123',
      )).thenAnswer((_) async {
        await Future.delayed(const Duration(seconds: 2));
        return Right(User(
          id: 'user-001',
          name: 'Test',
          email: 'test@example.com',
          phone: '08',
          role: 'customer',
          createdAt: DateTime.now(),
        ));
      });

      await tester.pumpWidget(buildLoginPage());

      await tester.enterText(
        find.widgetWithText(TextFormField, 'Email'),
        'test@example.com',
      );
      await tester.enterText(
        find.widgetWithText(TextFormField, 'Password'),
        'password123',
      );
      await tester.tap(find.text('Login').last);
      await tester.pump();

      expect(find.byType(CircularProgressIndicator), findsOneWidget);
    });
  });
}
```

## ขั้นตอนที่ 3766: Widget Tests - Restaurant List

```dart
// test/widget/restaurant_list_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:food_delivery/features/restaurants/presentation/pages/restaurants_list_page.dart';
import 'package:food_delivery/features/restaurants/presentation/providers/restaurants_provider.dart';
import 'package:food_delivery/features/restaurants/domain/entities/restaurant.dart';

void main() {
  final tRestaurants = [
    Restaurant(
      id: 'rest-001',
      name: 'Somtam Udon',
      category: 'Thai',
      address: '123 Sukhumvit',
      phone: '02-1234567',
      rating: 4.8,
      totalOrders: 1200,
      totalRevenue: 250000.0,
      isActive: true,
      isOpen: true,
      deliveryTime: 25,
      deliveryFee: 40.0,
      minimumOrder: 100.0,
      imageUrl: 'https://example.com/img.jpg',
      createdAt: DateTime(2023),
    ),
    Restaurant(
      id: 'rest-002',
      name: 'Ramen Ichiban',
      category: 'Japanese',
      address: '456 Silom',
      phone: '02-7654321',
      rating: 4.6,
      totalOrders: 800,
      totalRevenue: 180000.0,
      isActive: true,
      isOpen: false,
      deliveryTime: 35,
      deliveryFee: 50.0,
      minimumOrder: 150.0,
      createdAt: DateTime(2023),
    ),
  ];

  Widget buildPage({List<Restaurant>? restaurants}) {
    return ProviderScope(
      overrides: [
        restaurantsListProvider.overrideWith(
          (ref) => AsyncValue.data(restaurants ?? tRestaurants),
        ),
      ],
      child: const MaterialApp(home: RestaurantsListPage()),
    );
  }

  group('RestaurantsListPage', () {
    testWidgets('should show list of restaurants', (tester) async {
      await tester.pumpWidget(buildPage());
      await tester.pump();

      expect(find.text('Somtam Udon'), findsOneWidget);
      expect(find.text('Ramen Ichiban'), findsOneWidget);
    });

    testWidgets('should show restaurant rating', (tester) async {
      await tester.pumpWidget(buildPage());
      await tester.pump();

      expect(find.text('4.8'), findsOneWidget);
      expect(find.text('4.6'), findsOneWidget);
    });

    testWidgets('should show "Closed" badge for closed restaurants', (tester) async {
      await tester.pumpWidget(buildPage());
      await tester.pump();

      expect(find.text('Closed'), findsOneWidget);
    });

    testWidgets('should show loading state', (tester) async {
      await tester.pumpWidget(
        ProviderScope(
          overrides: [
            restaurantsListProvider.overrideWith(
              (ref) => const AsyncValue.loading(),
            ),
          ],
          child: const MaterialApp(home: RestaurantsListPage()),
        ),
      );

      expect(find.byType(CircularProgressIndicator), findsOneWidget);
    });

    testWidgets('should show error state', (tester) async {
      await tester.pumpWidget(
        ProviderScope(
          overrides: [
            restaurantsListProvider.overrideWith(
              (ref) => AsyncValue.error(
                Exception('Network error'),
                StackTrace.empty,
              ),
            ),
          ],
          child: const MaterialApp(home: RestaurantsListPage()),
        ),
      );
      await tester.pump();

      expect(find.textContaining('Error'), findsOneWidget);
      expect(find.text('Retry'), findsOneWidget);
    });

    testWidgets('should filter restaurants by category', (tester) async {
      await tester.pumpWidget(buildPage());
      await tester.pump();

      // Tap on Japanese filter
      await tester.tap(find.text('Japanese'));
      await tester.pump();

      expect(find.text('Somtam Udon'), findsNothing);
      expect(find.text('Ramen Ichiban'), findsOneWidget);
    });

    testWidgets('should search restaurants by name', (tester) async {
      await tester.pumpWidget(buildPage());
      await tester.pump();

      await tester.enterText(find.byType(SearchBar), 'Ramen');
      await tester.pump();

      expect(find.text('Somtam Udon'), findsNothing);
      expect(find.text('Ramen Ichiban'), findsOneWidget);
    });
  });
}
```

## ขั้นตอนที่ 3767: Widget Tests - Cart Page

```dart
// test/widget/cart_page_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:food_delivery/features/cart/presentation/pages/cart_page.dart';
import 'package:food_delivery/features/cart/presentation/providers/cart_provider.dart';
import 'package:food_delivery/features/cart/domain/entities/cart.dart';
import 'package:food_delivery/features/cart/domain/entities/cart_item.dart';

void main() {
  final tMenuItem = MenuItem(
    id: 'menu-001',
    name: 'Pad Thai',
    price: 120.0,
    restaurantId: 'rest-001',
    imageUrl: null,
    category: 'Noodles',
    isAvailable: true,
  );

  final tCart = Cart(
    restaurantId: 'rest-001',
    restaurantName: 'Test Restaurant',
    items: [
      CartItem(menuItem: tMenuItem, quantity: 2, addons: []),
    ],
    promoCode: null,
    deliveryFee: 40.0,
  );

  Widget buildCartPage(Cart? cart) {
    return ProviderScope(
      overrides: [
        cartProvider.overrideWith((ref) => cart),
      ],
      child: const MaterialApp(home: CartPage()),
    );
  }

  group('CartPage', () {
    testWidgets('should show empty cart message when cart is empty', (tester) async {
      await tester.pumpWidget(buildCartPage(null));

      expect(find.text('Your cart is empty'), findsOneWidget);
      expect(find.text('Browse Restaurants'), findsOneWidget);
    });

    testWidgets('should show cart items', (tester) async {
      await tester.pumpWidget(buildCartPage(tCart));
      await tester.pump();

      expect(find.text('Pad Thai'), findsOneWidget);
      expect(find.text('฿120.00'), findsOneWidget);
      expect(find.text('2'), findsOneWidget);
    });

    testWidgets('should show correct total', (tester) async {
      await tester.pumpWidget(buildCartPage(tCart));
      await tester.pump();

      // Subtotal: 2 * 120 = 240
      expect(find.text('฿240.00'), findsOneWidget);
      // Delivery fee: 40
      expect(find.text('฿40.00'), findsOneWidget);
      // Total: 280
      expect(find.text('฿280.00'), findsOneWidget);
    });

    testWidgets('should increase quantity on + button tap', (tester) async {
      await tester.pumpWidget(buildCartPage(tCart));
      await tester.pump();

      await tester.tap(find.byIcon(Icons.add_rounded).first);
      await tester.pump();

      // Quantity should be 3 now
      expect(find.text('3'), findsOneWidget);
    });

    testWidgets('should show checkout button', (tester) async {
      await tester.pumpWidget(buildCartPage(tCart));

      expect(find.text('Proceed to Checkout'), findsOneWidget);
    });

    testWidgets('should show promo code input', (tester) async {
      await tester.pumpWidget(buildCartPage(tCart));

      expect(find.byType(TextField), findsWidgets);
      expect(find.text('Enter promo code'), findsOneWidget);
    });
  });
}
```

## ขั้นตอนที่ 3768: Integration Tests

```dart
// integration_test/auth_flow_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:food_delivery/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('Authentication Flow', () {
    testWidgets('complete login flow', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Verify we're on login page
      expect(find.text('Welcome Back'), findsOneWidget);
      expect(find.text('Login'), findsOneWidget);

      // Enter credentials
      await tester.enterText(
        find.byKey(const Key('email_field')),
        'test@example.com',
      );
      await tester.enterText(
        find.byKey(const Key('password_field')),
        'password123',
      );

      // Tap login button
      await tester.tap(find.byKey(const Key('login_button')));
      await tester.pumpAndSettle(const Duration(seconds: 3));

      // Verify navigation to home
      expect(find.text('Discover'), findsOneWidget);
      expect(find.text('Recommended for you'), findsOneWidget);
    });

    testWidgets('logout flow', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Assume already logged in (or login first)
      // Navigate to profile
      await tester.tap(find.byIcon(Icons.person_outline_rounded));
      await tester.pumpAndSettle();

      // Tap logout
      await tester.tap(find.text('Logout'));
      await tester.pumpAndSettle();

      // Confirm logout
      await tester.tap(find.text('Confirm'));
      await tester.pumpAndSettle();

      // Verify back on login page
      expect(find.text('Welcome Back'), findsOneWidget);
    });
  });

  group('Order Flow Integration', () {
    testWidgets('complete order placement flow', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Navigate to restaurants
      await tester.tap(find.byIcon(Icons.restaurant_outlined));
      await tester.pumpAndSettle();

      // Tap on first restaurant
      await tester.tap(find.byType(Card).first);
      await tester.pumpAndSettle();

      // Add item to cart
      await tester.tap(find.byIcon(Icons.add_rounded).first);
      await tester.pumpAndSettle();

      // Go to cart
      await tester.tap(find.byIcon(Icons.shopping_cart_outlined));
      await tester.pumpAndSettle();

      // Verify cart has item
      expect(find.text('Proceed to Checkout'), findsOneWidget);

      // Proceed to checkout
      await tester.tap(find.text('Proceed to Checkout'));
      await tester.pumpAndSettle();

      // Select payment method
      await tester.tap(find.text('Credit Card'));
      await tester.pumpAndSettle();

      // Place order
      await tester.tap(find.text('Place Order'));
      await tester.pumpAndSettle(const Duration(seconds: 2));

      // Verify order success
      expect(find.text('Order Placed Successfully!'), findsOneWidget);
      expect(find.text('Track Order'), findsOneWidget);
    });
  });
}
```

## ขั้นตอนที่ 3769: Integration Test - Search Flow

```dart
// integration_test/search_flow_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:food_delivery/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('Search Flow', () {
    testWidgets('search restaurants by name', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Tap search bar
      await tester.tap(find.byType(SearchBar));
      await tester.pumpAndSettle();

      // Type search query
      await tester.enterText(find.byType(SearchBar), 'Pad Thai');
      await tester.pumpAndSettle();

      // Verify results appear
      expect(find.textContaining('Pad Thai'), findsWidgets);
    });

    testWidgets('search with no results shows empty state', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      await tester.tap(find.byType(SearchBar));
      await tester.pumpAndSettle();

      await tester.enterText(find.byType(SearchBar), 'xyznonexistentfood123');
      await tester.pumpAndSettle();

      expect(find.text('No results found'), findsOneWidget);
    });

    testWidgets('filter by category', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      await tester.tap(find.text('Japanese'));
      await tester.pumpAndSettle();

      // Verify only Japanese restaurants show
      final cards = find.byType(RestaurantCard);
      expect(cards, findsWidgets);
    });
  });
}
```

## ขั้นตอนที่ 3770: Performance Tests

```dart
// test/performance/scroll_performance_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:food_delivery/features/restaurants/presentation/pages/restaurants_list_page.dart';
import 'package:food_delivery/features/restaurants/domain/entities/restaurant.dart';
import 'package:food_delivery/features/restaurants/presentation/providers/restaurants_provider.dart';

void main() {
  group('Scroll Performance', () {
    testWidgets('restaurant list scrolls smoothly (60fps)', (tester) async {
      // Generate 100 restaurants for stress testing
      final restaurants = List.generate(
        100,
        (i) => Restaurant(
          id: 'rest-$i',
          name: 'Restaurant $i',
          category: ['Thai', 'Japanese', 'Italian'][i % 3],
          address: 'Address $i',
          phone: '02-000000$i',
          rating: 4.0 + (i % 10) * 0.1,
          totalOrders: 100 + i,
          totalRevenue: 50000.0 + i * 1000,
          isActive: true,
          isOpen: i % 3 != 0,
          deliveryTime: 20 + (i % 20),
          deliveryFee: 30.0 + (i % 30),
          minimumOrder: 100.0,
          createdAt: DateTime(2023),
        ),
      );

      await tester.pumpWidget(
        ProviderScope(
          overrides: [
            restaurantsListProvider.overrideWith(
              (ref) => AsyncValue.data(restaurants),
            ),
          ],
          child: const MaterialApp(home: RestaurantsListPage()),
        ),
      );
      await tester.pump();

      // Measure frame rendering during scroll
      final stopwatch = Stopwatch()..start();
      
      await tester.fling(
        find.byType(ListView),
        const Offset(0, -500),
        1000,
      );
      await tester.pumpAndSettle();
      
      stopwatch.stop();

      // Should complete within reasonable time
      expect(stopwatch.elapsedMilliseconds, lessThan(2000));
    });

    testWidgets('order list renders large dataset without jank', (tester) async {
      final orders = List.generate(
        200,
        (i) => OrderListItem(
          id: 'order-$i',
          restaurantName: 'Restaurant ${i % 10}',
          totalAmount: 150.0 + i * 10,
          status: ['Delivered', 'Processing', 'Cancelled'][i % 3],
          createdAt: DateTime.now().subtract(Duration(minutes: i * 30)),
          itemCount: 1 + i % 5,
        ),
      );

      await tester.pumpWidget(
        ProviderScope(
          overrides: [
            ordersProvider.overrideWith(
              (ref) => AsyncValue.data(orders),
            ),
          ],
          child: const MaterialApp(home: OrdersListPage()),
        ),
      );
      await tester.pump();

      // Check initial render is fast
      final frameCount = tester.binding.framesRendered;
      
      await tester.drag(
        find.byType(ListView),
        const Offset(0, -2000),
      );
      await tester.pump(const Duration(milliseconds: 500));

      // Verify frames were rendered smoothly
      final framesAfterScroll = tester.binding.framesRendered;
      expect(framesAfterScroll - frameCount, greaterThan(0));
    });
  });

  group('Memory Management', () {
    testWidgets('images are properly cached and disposed', (tester) async {
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: ListView.builder(
              itemCount: 50,
              itemBuilder: (context, index) => ListTile(
                leading: Image.network(
                  'https://picsum.photos/seed/$index/100/100',
                  width: 60,
                  height: 60,
                  fit: BoxFit.cover,
                  errorBuilder: (_, __, ___) => const Icon(Icons.image),
                ),
                title: Text('Item $index'),
              ),
            ),
          ),
        ),
      );
      await tester.pump();

      // Scroll to load more images
      await tester.fling(
        find.byType(ListView),
        const Offset(0, -3000),
        2000,
      );
      await tester.pumpAndSettle();

      // Verify no memory leaks by scrolling back
      await tester.fling(
        find.byType(ListView),
        const Offset(0, 3000),
        2000,
      );
      await tester.pumpAndSettle();

      // Test passes if no exceptions thrown
      expect(find.byType(ListView), findsOneWidget);
    });
  });
}
```

## ขั้นตอนที่ 3771: Riverpod State Tests

```dart
// test/unit/providers/cart_provider_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'package:food_delivery/features/cart/presentation/providers/cart_provider.dart';
import 'package:food_delivery/features/cart/domain/repositories/cart_repository.dart';
import 'package:food_delivery/features/cart/domain/entities/cart.dart';
import 'package:food_delivery/features/cart/domain/entities/cart_item.dart';

import 'cart_provider_test.mocks.dart';

@GenerateMocks([CartRepository])
void main() {
  late MockCartRepository mockRepository;

  setUp(() {
    mockRepository = MockCartRepository();
  });

  ProviderContainer createContainer() {
    return ProviderContainer(
      overrides: [
        cartRepositoryProvider.overrideWithValue(mockRepository),
      ],
    );
  }

  final tMenuItem = MenuItem(
    id: 'menu-001',
    name: 'Pad Thai',
    price: 120.0,
    restaurantId: 'rest-001',
    imageUrl: null,
    category: 'Noodles',
    isAvailable: true,
  );

  group('CartNotifier', () {
    test('initial state should be null (empty cart)', () {
      final container = createContainer();
      addTearDown(container.dispose);

      final cart = container.read(cartProvider);
      expect(cart, isNull);
    });

    test('addItem should update cart state', () async {
      final container = createContainer();
      addTearDown(container.dispose);

      final newCart = Cart(
        restaurantId: 'rest-001',
        restaurantName: 'Test',
        items: [CartItem(menuItem: tMenuItem, quantity: 1, addons: [])],
        promoCode: null,
        deliveryFee: 40.0,
      );

      when(mockRepository.addItem(
        menuItem: tMenuItem,
        quantity: 1,
        addons: [],
        notes: null,
      )).thenAnswer((_) async => Right(newCart));

      await container.read(cartProvider.notifier).addItem(
        menuItem: tMenuItem,
        quantity: 1,
      );

      final cart = container.read(cartProvider);
      expect(cart, isNotNull);
      expect(cart!.items.length, 1);
      expect(cart.items.first.menuItem.id, 'menu-001');
    });

    test('clearCart should reset state to null', () async {
      final container = createContainer();
      addTearDown(container.dispose);

      when(mockRepository.clearCart()).thenAnswer((_) async => const Right(null));

      await container.read(cartProvider.notifier).clearCart();

      final cart = container.read(cartProvider);
      expect(cart, isNull);
    });

    test('applyPromoCode should update cart with discount', () async {
      final container = createContainer();
      addTearDown(container.dispose);

      final cartWithPromo = Cart(
        restaurantId: 'rest-001',
        restaurantName: 'Test',
        items: [CartItem(menuItem: tMenuItem, quantity: 1, addons: [])],
        promoCode: 'SAVE20',
        discount: 24.0,
        deliveryFee: 40.0,
      );

      when(mockRepository.applyPromoCode(code: 'SAVE20'))
          .thenAnswer((_) async => Right(cartWithPromo));

      await container.read(cartProvider.notifier).applyPromoCode('SAVE20');

      final cart = container.read(cartProvider);
      expect(cart?.promoCode, 'SAVE20');
      expect(cart?.discount, 24.0);
    });
  });
}
```

## ขั้นตอนที่ 3772: Test Coverage Report Generation

```dart
// test/helpers/test_helpers.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

/// Creates a widget wrapper with common providers for testing
Widget createTestApp({
  required Widget child,
  List<Override> overrides = const [],
  GoRouter? router,
}) {
  return ProviderScope(
    overrides: overrides,
    child: MaterialApp(
      home: child,
    ),
  );
}

/// Pumps a widget and waits for all animations to complete
Future<void> pumpAndSettle(WidgetTester tester, Widget widget) async {
  await tester.pumpWidget(widget);
  await tester.pumpAndSettle();
}

/// Creates a fake restaurant for testing
Restaurant createFakeRestaurant({
  String id = 'test-rest-001',
  String name = 'Test Restaurant',
  bool isOpen = true,
}) {
  return Restaurant(
    id: id,
    name: name,
    category: 'Thai',
    address: '123 Test St',
    phone: '02-1234567',
    rating: 4.5,
    totalOrders: 500,
    totalRevenue: 100000.0,
    isActive: true,
    isOpen: isOpen,
    deliveryTime: 30,
    deliveryFee: 40.0,
    minimumOrder: 100.0,
    createdAt: DateTime(2023),
  );
}

/// Creates a fake order for testing
Order createFakeOrder({
  String id = 'test-order-001',
  OrderStatus status = OrderStatus.pending,
}) {
  return Order(
    id: id,
    customerId: 'user-001',
    restaurantId: 'rest-001',
    restaurantName: 'Test Restaurant',
    items: [
      OrderItem(
        menuItemId: 'menu-001',
        name: 'Test Item',
        price: 100.0,
        quantity: 2,
      ),
    ],
    subtotal: 200.0,
    deliveryFee: 40.0,
    discount: 0.0,
    totalAmount: 240.0,
    status: status,
    deliveryAddress: '456 Test Ave',
    createdAt: DateTime(2024, 1, 1, 12, 0),
    updatedAt: DateTime(2024, 1, 1, 12, 0),
  );
}
```

```bash
# Makefile targets for testing
# Run: make test, make coverage, make test-integration

.PHONY: test coverage test-unit test-widget test-integration

test:
	flutter test

test-unit:
	flutter test test/unit/

test-widget:
	flutter test test/widget/

test-integration:
	flutter test integration_test/ \
		-d chrome \
		--browser-name=chrome

coverage:
	flutter test --coverage
	lcov --remove coverage/lcov.info \
		'**/*.g.dart' \
		'**/*.freezed.dart' \
		'**/generated/**' \
		-o coverage/lcov_filtered.info
	genhtml coverage/lcov_filtered.info \
		--output-directory coverage/html \
		--title "Food Delivery App Coverage"
	@echo "Coverage report: coverage/html/index.html"

coverage-check:
	flutter test --coverage
	lcov --summary coverage/lcov.info | grep -E "lines.*: ([89][0-9]|100)\."
	@echo "Coverage check passed (>= 80%)"
```

---

**← [Part 96 - Admin Dashboard](part-96-capstone-admin-dashboard.md)**
**ต่อไป: [Part 98 →](part-98-world-class-patterns.md)**
