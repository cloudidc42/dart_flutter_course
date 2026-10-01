# Part 17: Testing
## ขั้นตอนที่ 561-600

---

## 🎯 เป้าหมายของ Part นี้

- Unit Testing
- Widget Testing
- Integration Testing
- Test-Driven Development (TDD)
- Mock objects
- Test coverage

---

## ขั้นตอนที่ 561: Unit Testing พื้นฐาน

```dart
// test/unit_test.dart
// pubspec.yaml:
// dev_dependencies:
//   test: ^1.24.0
//   flutter_test:
//     sdk: flutter
//   mockito: ^5.4.4
//   build_runner: ^2.4.0

import 'package:test/test.dart';

// ─── Code ที่จะ test ───
class Calculator {
  double add(double a, double b) => a + b;
  double subtract(double a, double b) => a - b;
  double multiply(double a, double b) => a * b;
  double divide(double a, double b) {
    if (b == 0) throw ArgumentError('Cannot divide by zero');
    return a / b;
  }
  
  bool isPrime(int n) {
    if (n < 2) return false;
    for (int i = 2; i <= n ~/ 2; i++) {
      if (n % i == 0) return false;
    }
    return true;
  }
}

class StringUtils {
  static String reverse(String s) => s.split('').reversed.join();
  
  static bool isPalindrome(String s) {
    String cleaned = s.toLowerCase().replaceAll(RegExp(r'[^a-z0-9]'), '');
    return cleaned == reverse(cleaned);
  }
  
  static List<String> wordCount(String text) {
    return text.trim().split(RegExp(r'\s+'));
  }
}

// ─── Tests ───
void main() {
  group('Calculator', () {
    late Calculator calc;
    
    setUp(() {
      calc = Calculator();  // รัน setup ก่อนทุก test
    });
    
    tearDown(() {
      // cleanup after each test
    });
    
    group('Basic operations', () {
      test('add returns correct sum', () {
        expect(calc.add(2, 3), equals(5));
        expect(calc.add(-1, 1), equals(0));
        expect(calc.add(0.5, 0.5), equals(1.0));
      });
      
      test('subtract returns correct difference', () {
        expect(calc.subtract(5, 3), equals(2));
        expect(calc.subtract(3, 5), equals(-2));
      });
      
      test('multiply returns correct product', () {
        expect(calc.multiply(3, 4), equals(12));
        expect(calc.multiply(-2, 3), equals(-6));
        expect(calc.multiply(0, 100), equals(0));
      });
      
      test('divide returns correct quotient', () {
        expect(calc.divide(10, 2), equals(5));
        expect(calc.divide(7, 2), equals(3.5));
      });
      
      test('divide by zero throws ArgumentError', () {
        expect(
          () => calc.divide(5, 0),
          throwsA(isA<ArgumentError>()),
        );
        
        expect(
          () => calc.divide(5, 0),
          throwsA(
            isA<ArgumentError>().having(
              (e) => e.message,
              'message',
              contains('zero'),
            ),
          ),
        );
      });
    });
    
    group('Prime numbers', () {
      test('correctly identifies prime numbers', () {
        List<int> primes = [2, 3, 5, 7, 11, 13, 17, 19, 23];
        for (int p in primes) {
          expect(calc.isPrime(p), isTrue, reason: '$p should be prime');
        }
      });
      
      test('correctly identifies non-prime numbers', () {
        List<int> nonPrimes = [0, 1, 4, 6, 8, 9, 10, 15, 20];
        for (int n in nonPrimes) {
          expect(calc.isPrime(n), isFalse, reason: '$n should not be prime');
        }
      });
    });
  });
  
  group('StringUtils', () {
    test('reverse reverses string', () {
      expect(StringUtils.reverse('hello'), equals('olleh'));
      expect(StringUtils.reverse(''), equals(''));
      expect(StringUtils.reverse('a'), equals('a'));
    });
    
    test('isPalindrome detects palindromes', () {
      expect(StringUtils.isPalindrome('racecar'), isTrue);
      expect(StringUtils.isPalindrome('A man a plan a canal Panama'), isTrue);
      expect(StringUtils.isPalindrome('hello'), isFalse);
    });
    
    test('wordCount splits words correctly', () {
      expect(StringUtils.wordCount('hello world'), hasLength(2));
      expect(StringUtils.wordCount('  spaced  words  '), hasLength(2));
    });
  });
}
```

---

## ขั้นตอนที่ 562: Testing กับ Async

```dart
import 'package:test/test.dart';
import 'dart:async';

// ─── Code ที่ test ───
class UserService {
  final List<Map<String, dynamic>> _db = [
    {'id': 1, 'name': 'Alice', 'email': 'alice@example.com'},
    {'id': 2, 'name': 'Bob', 'email': 'bob@example.com'},
  ];
  
  Future<Map<String, dynamic>?> getUser(int id) async {
    await Future.delayed(const Duration(milliseconds: 10));
    try {
      return _db.firstWhere((u) => u['id'] == id);
    } catch (_) {
      return null;
    }
  }
  
  Stream<int> countDown(int from) async* {
    for (int i = from; i >= 0; i--) {
      await Future.delayed(const Duration(milliseconds: 10));
      yield i;
    }
  }
  
  Future<List<Map<String, dynamic>>> search(String query) async {
    await Future.delayed(const Duration(milliseconds: 100));
    return _db.where((u) =>
      u['name'].toString().toLowerCase().contains(query.toLowerCase())
    ).toList();
  }
}

// ─── Tests ───
void main() {
  group('UserService', () {
    late UserService service;
    
    setUp(() => service = UserService());
    
    // Async test
    test('getUser returns user when found', () async {
      Map<String, dynamic>? user = await service.getUser(1);
      
      expect(user, isNotNull);
      expect(user!['name'], equals('Alice'));
      expect(user['email'], equals('alice@example.com'));
    });
    
    test('getUser returns null when not found', () async {
      Map<String, dynamic>? user = await service.getUser(999);
      expect(user, isNull);
    });
    
    // Test Futures
    test('search returns matching users', () async {
      List<Map<String, dynamic>> results = await service.search('alice');
      
      expect(results, hasLength(1));
      expect(results.first['name'], equals('Alice'));
    });
    
    test('search is case insensitive', () async {
      List<Map<String, dynamic>> upper = await service.search('ALICE');
      List<Map<String, dynamic>> lower = await service.search('alice');
      
      expect(upper.length, equals(lower.length));
    });
    
    // Test Streams
    test('countDown emits correct sequence', () async {
      List<int> values = await service.countDown(3).toList();
      expect(values, equals([3, 2, 1, 0]));
    });
    
    test('countDown completes', () {
      expect(service.countDown(2), emitsInOrder([2, 1, 0, emitsDone]));
    });
    
    // Timeout test
    test('search completes within 500ms', () async {
      await service.search('bob').timeout(const Duration(milliseconds: 500));
    });
  });
}
```

---

## ขั้นตอนที่ 563: Mocking

```dart
// pubspec.yaml dev_dependencies:
//   mockito: ^5.4.4

import 'package:test/test.dart';
import 'package:mockito/mockito.dart';
import 'package:mockito/annotations.dart';

// ─── Dependencies ───
abstract class HttpClient {
  Future<String> get(String url);
  Future<String> post(String url, {Map<String, dynamic>? body});
}

abstract class UserRepository {
  Future<Map<String, dynamic>?> findById(String id);
  Future<void> save(Map<String, dynamic> user);
}

// ─── Mocks (generated ด้วย mockito) ───
// สร้าง mock ด้วย:
// @GenerateMocks([HttpClient, UserRepository])

class MockHttpClient extends Mock implements HttpClient {}
class MockUserRepository extends Mock implements UserRepository {}

// ─── Service ที่ depend on dependencies ───
class UserApiService {
  final HttpClient _http;
  final UserRepository _repo;
  
  UserApiService(this._http, this._repo);
  
  Future<Map<String, dynamic>?> getUser(String id) async {
    // Check cache
    Map<String, dynamic>? cached = await _repo.findById(id);
    if (cached != null) return cached;
    
    // Fetch from API
    String json = await _http.get('https://api.example.com/users/$id');
    Map<String, dynamic> user = {'id': id, 'data': json};
    
    // Cache it
    await _repo.save(user);
    return user;
  }
}

// ─── Tests ───
void main() {
  group('UserApiService', () {
    late MockHttpClient mockHttp;
    late MockUserRepository mockRepo;
    late UserApiService service;
    
    setUp(() {
      mockHttp = MockHttpClient();
      mockRepo = MockUserRepository();
      service = UserApiService(mockHttp, mockRepo);
    });
    
    test('returns cached user without HTTP call', () async {
      // Arrange
      Map<String, dynamic> cachedUser = {'id': '1', 'name': 'Alice'};
      when(mockRepo.findById('1')).thenAnswer((_) async => cachedUser);
      
      // Act
      Map<String, dynamic>? result = await service.getUser('1');
      
      // Assert
      expect(result, equals(cachedUser));
      verifyNever(mockHttp.get(any));  // ไม่ควร call HTTP
    });
    
    test('fetches from API when not cached', () async {
      // Arrange
      when(mockRepo.findById('2')).thenAnswer((_) async => null);
      when(mockHttp.get('https://api.example.com/users/2'))
          .thenAnswer((_) async => '{"name":"Bob"}');
      when(mockRepo.save(any)).thenAnswer((_) async {});
      
      // Act
      Map<String, dynamic>? result = await service.getUser('2');
      
      // Assert
      expect(result, isNotNull);
      expect(result!['id'], equals('2'));
      verify(mockHttp.get('https://api.example.com/users/2')).called(1);
      verify(mockRepo.save(any)).called(1);
    });
    
    test('throws when HTTP fails', () async {
      // Arrange
      when(mockRepo.findById('3')).thenAnswer((_) async => null);
      when(mockHttp.get(any)).thenThrow(Exception('Network error'));
      
      // Act & Assert
      expect(() => service.getUser('3'), throwsA(isA<Exception>()));
    });
  });
}
```

---

## ขั้นตอนที่ 564: Widget Testing

```dart
// test/widget_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

// ─── Widget ที่จะ test ───
class CounterWidget extends StatefulWidget {
  final int initialValue;
  final int step;
  final void Function(int)? onChanged;
  
  const CounterWidget({
    super.key,
    this.initialValue = 0,
    this.step = 1,
    this.onChanged,
  });
  
  @override
  State<CounterWidget> createState() => _CounterWidgetState();
}

class _CounterWidgetState extends State<CounterWidget> {
  late int _count;
  
  @override
  void initState() {
    super.initState();
    _count = widget.initialValue;
  }
  
  void _increment() {
    setState(() {
      _count += widget.step;
      widget.onChanged?.call(_count);
    });
  }
  
  void _decrement() {
    setState(() {
      _count -= widget.step;
      widget.onChanged?.call(_count);
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('$_count', key: const Key('counter_value')),
        Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              key: const Key('decrement_button'),
              onPressed: _decrement,
              child: const Text('-'),
            ),
            const SizedBox(width: 16),
            ElevatedButton(
              key: const Key('increment_button'),
              onPressed: _increment,
              child: const Text('+'),
            ),
          ],
        ),
      ],
    );
  }
}

// ─── Widget Tests ───
void main() {
  group('CounterWidget', () {
    testWidgets('shows initial value', (WidgetTester tester) async {
      await tester.pumpWidget(
        const MaterialApp(
          home: Scaffold(body: CounterWidget(initialValue: 5)),
        ),
      );
      
      expect(find.text('5'), findsOneWidget);
    });
    
    testWidgets('increments when + button pressed', (tester) async {
      await tester.pumpWidget(
        const MaterialApp(home: Scaffold(body: CounterWidget())),
      );
      
      expect(find.text('0'), findsOneWidget);
      
      await tester.tap(find.byKey(const Key('increment_button')));
      await tester.pump();  // rebuild
      
      expect(find.text('1'), findsOneWidget);
      expect(find.text('0'), findsNothing);
    });
    
    testWidgets('decrements when - button pressed', (tester) async {
      await tester.pumpWidget(
        const MaterialApp(home: Scaffold(body: CounterWidget(initialValue: 5))),
      );
      
      await tester.tap(find.byKey(const Key('decrement_button')));
      await tester.pump();
      
      expect(find.text('4'), findsOneWidget);
    });
    
    testWidgets('uses step parameter', (tester) async {
      await tester.pumpWidget(
        const MaterialApp(home: Scaffold(body: CounterWidget(step: 5))),
      );
      
      await tester.tap(find.byKey(const Key('increment_button')));
      await tester.pump();
      
      expect(find.text('5'), findsOneWidget);
    });
    
    testWidgets('calls onChanged callback', (tester) async {
      int? lastValue;
      
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: CounterWidget(onChanged: (v) => lastValue = v),
          ),
        ),
      );
      
      await tester.tap(find.byKey(const Key('increment_button')));
      await tester.pump();
      
      expect(lastValue, equals(1));
    });
    
    testWidgets('has increment and decrement buttons', (tester) async {
      await tester.pumpWidget(
        const MaterialApp(home: Scaffold(body: CounterWidget())),
      );
      
      expect(find.byKey(const Key('increment_button')), findsOneWidget);
      expect(find.byKey(const Key('decrement_button')), findsOneWidget);
      expect(find.byType(ElevatedButton), findsNWidgets(2));
    });
    
    testWidgets('multiple increments work correctly', (tester) async {
      await tester.pumpWidget(
        const MaterialApp(home: Scaffold(body: CounterWidget())),
      );
      
      for (int i = 0; i < 5; i++) {
        await tester.tap(find.byKey(const Key('increment_button')));
        await tester.pump();
      }
      
      expect(find.text('5'), findsOneWidget);
    });
  });
  
  // Testing a more complex widget
  group('LoginForm Widget', () {
    testWidgets('shows validation errors when submitting empty form', (tester) async {
      await tester.pumpWidget(
        MaterialApp(home: Scaffold(body: _LoginForm())),
      );
      
      // Tap submit without filling form
      await tester.tap(find.text('Login'));
      await tester.pump();
      
      expect(find.text('กรุณาใส่ Email'), findsOneWidget);
      expect(find.text('กรุณาใส่ Password'), findsOneWidget);
    });
    
    testWidgets('accepts valid input', (tester) async {
      await tester.pumpWidget(
        MaterialApp(home: Scaffold(body: _LoginForm())),
      );
      
      await tester.enterText(
        find.byKey(const Key('email_field')),
        'test@example.com',
      );
      await tester.enterText(
        find.byKey(const Key('password_field')),
        'password123',
      );
      
      await tester.tap(find.text('Login'));
      await tester.pump();
      
      expect(find.text('กรุณาใส่ Email'), findsNothing);
      expect(find.text('กรุณาใส่ Password'), findsNothing);
    });
  });
}

class _LoginForm extends StatefulWidget {
  @override
  State<_LoginForm> createState() => _LoginFormState();
}

class _LoginFormState extends State<_LoginForm> {
  final _formKey = GlobalKey<FormState>();
  
  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      child: Column(
        children: [
          TextFormField(
            key: const Key('email_field'),
            decoration: const InputDecoration(labelText: 'Email'),
            validator: (v) => v?.isEmpty == true ? 'กรุณาใส่ Email' : null,
          ),
          TextFormField(
            key: const Key('password_field'),
            decoration: const InputDecoration(labelText: 'Password'),
            validator: (v) => v?.isEmpty == true ? 'กรุณาใส่ Password' : null,
          ),
          ElevatedButton(
            onPressed: () => _formKey.currentState?.validate(),
            child: const Text('Login'),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 565: Test-Driven Development (TDD)

```dart
// TDD Cycle: Red → Green → Refactor
//
// 1. RED: เขียน test ที่ fail ก่อน
// 2. GREEN: เขียน code ให้ test ผ่าน (minimal implementation)
// 3. REFACTOR: ปรับปรุง code โดยยังคง test ผ่าน

import 'package:test/test.dart';

// ─── Step 1: RED - เขียน test ก่อน ───
// (ตอนนี้ยังไม่มี ShoppingCart class)

// ─── Step 2: GREEN - เขียน implementation ───
class CartItem {
  final String productId;
  final String name;
  final double price;
  int quantity;
  
  CartItem({
    required this.productId,
    required this.name,
    required this.price,
    this.quantity = 1,
  });
  
  double get subtotal => price * quantity;
}

class ShoppingCart {
  final List<CartItem> _items = [];
  double _discountPercent = 0;
  
  List<CartItem> get items => List.unmodifiable(_items);
  
  int get itemCount => _items.fold(0, (sum, item) => sum + item.quantity);
  
  double get subtotal => _items.fold(0, (sum, item) => sum + item.subtotal);
  
  double get discount => subtotal * _discountPercent / 100;
  
  double get total => subtotal - discount;
  
  bool get isEmpty => _items.isEmpty;
  
  void addItem(CartItem item) {
    int idx = _items.indexWhere((i) => i.productId == item.productId);
    if (idx != -1) {
      _items[idx].quantity += item.quantity;
    } else {
      _items.add(item);
    }
  }
  
  void removeItem(String productId) {
    _items.removeWhere((i) => i.productId == productId);
  }
  
  void updateQuantity(String productId, int quantity) {
    int idx = _items.indexWhere((i) => i.productId == productId);
    if (idx != -1) {
      if (quantity <= 0) {
        _items.removeAt(idx);
      } else {
        _items[idx].quantity = quantity;
      }
    }
  }
  
  void applyDiscount(double percent) {
    if (percent < 0 || percent > 100) {
      throw ArgumentError('Discount must be between 0 and 100');
    }
    _discountPercent = percent;
  }
  
  void clear() {
    _items.clear();
    _discountPercent = 0;
  }
}

// ─── Tests ───
void main() {
  group('ShoppingCart (TDD)', () {
    late ShoppingCart cart;
    
    setUp(() => cart = ShoppingCart());
    
    // Basic state
    test('new cart is empty', () {
      expect(cart.isEmpty, isTrue);
      expect(cart.itemCount, equals(0));
      expect(cart.subtotal, equals(0));
    });
    
    // Adding items
    test('can add item', () {
      cart.addItem(CartItem(productId: 'p1', name: 'Apple', price: 10, quantity: 2));
      
      expect(cart.isEmpty, isFalse);
      expect(cart.itemCount, equals(2));
      expect(cart.subtotal, equals(20));
    });
    
    test('adding same product increases quantity', () {
      cart.addItem(CartItem(productId: 'p1', name: 'Apple', price: 10, quantity: 1));
      cart.addItem(CartItem(productId: 'p1', name: 'Apple', price: 10, quantity: 3));
      
      expect(cart.items, hasLength(1));
      expect(cart.itemCount, equals(4));
    });
    
    test('can add multiple different products', () {
      cart.addItem(CartItem(productId: 'p1', name: 'Apple', price: 10));
      cart.addItem(CartItem(productId: 'p2', name: 'Banana', price: 5));
      
      expect(cart.items, hasLength(2));
      expect(cart.subtotal, equals(15));
    });
    
    // Removing
    test('can remove item', () {
      cart.addItem(CartItem(productId: 'p1', name: 'Apple', price: 10));
      cart.addItem(CartItem(productId: 'p2', name: 'Banana', price: 5));
      
      cart.removeItem('p1');
      
      expect(cart.items, hasLength(1));
      expect(cart.subtotal, equals(5));
    });
    
    // Updating
    test('can update quantity', () {
      cart.addItem(CartItem(productId: 'p1', name: 'Apple', price: 10, quantity: 1));
      cart.updateQuantity('p1', 5);
      
      expect(cart.itemCount, equals(5));
      expect(cart.subtotal, equals(50));
    });
    
    test('removes item when quantity set to 0', () {
      cart.addItem(CartItem(productId: 'p1', name: 'Apple', price: 10));
      cart.updateQuantity('p1', 0);
      
      expect(cart.isEmpty, isTrue);
    });
    
    // Discount
    test('applies percentage discount correctly', () {
      cart.addItem(CartItem(productId: 'p1', name: 'Apple', price: 100));
      cart.applyDiscount(10);  // 10% off
      
      expect(cart.discount, equals(10));
      expect(cart.total, equals(90));
    });
    
    test('throws on invalid discount', () {
      expect(() => cart.applyDiscount(-5), throwsA(isA<ArgumentError>()));
      expect(() => cart.applyDiscount(110), throwsA(isA<ArgumentError>()));
    });
    
    // Clear
    test('clears all items and resets discount', () {
      cart.addItem(CartItem(productId: 'p1', name: 'Apple', price: 10));
      cart.applyDiscount(20);
      cart.clear();
      
      expect(cart.isEmpty, isTrue);
      expect(cart.discount, equals(0));
    });
    
    // ─── Step 3: REFACTOR ─── (ปรับปรุง implementation โดยยังผ่าน tests)
    // การ refactor ควรทำเมื่อ code ทำงานได้แล้ว (tests ผ่าน)
    // ตัวอย่าง: แยก CartItem แบบ immutable, เพิ่ม validation
  });
}
```

---

## ขั้นตอนที่ 566-600: Integration Testing

```dart
// integration_test/app_test.dart
// pubspec.yaml:
// dev_dependencies:
//   integration_test:
//     sdk: flutter

import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  
  group('End-to-End Tests', () {
    testWidgets('complete login flow', (tester) async {
      // Launch app
      await tester.pumpWidget(const MyApp());
      await tester.pumpAndSettle();
      
      // Should show login screen
      expect(find.text('เข้าสู่ระบบ'), findsOneWidget);
      
      // Enter credentials
      await tester.enterText(find.byKey(const Key('email_field')), 'test@test.com');
      await tester.enterText(find.byKey(const Key('password_field')), 'password123');
      
      // Tap login
      await tester.tap(find.text('เข้าสู่ระบบ'));
      await tester.pumpAndSettle(const Duration(seconds: 3));
      
      // Should navigate to home
      expect(find.text('Home'), findsOneWidget);
    });
    
    testWidgets('complete shopping flow', (tester) async {
      await tester.pumpWidget(const MyApp());
      await tester.pumpAndSettle();
      
      // Find product card
      expect(find.text('Product 1'), findsOneWidget);
      
      // Add to cart
      await tester.tap(find.text('เพิ่มในตะกร้า').first);
      await tester.pumpAndSettle();
      
      // Check cart badge
      expect(find.text('1'), findsWidgets);
      
      // Go to cart
      await tester.tap(find.byIcon(Icons.shopping_cart));
      await tester.pumpAndSettle();
      
      expect(find.text('ตะกร้าสินค้า'), findsOneWidget);
    });
  });
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Test App',
      home: const Scaffold(body: Center(child: Text('Test App'))),
    );
  }
}
```

### Testing Best Practices

```dart
// ✅ Good Tests คือ FIRST:
// F - Fast: รันเร็ว
// I - Independent: ไม่ depend กัน
// R - Repeatable: ผลเหมือนกันทุกครั้ง
// S - Self-validating: บอก pass/fail ชัดเจน
// T - Timely: เขียนก่อนหรือพร้อม code

// ✅ Test structure: Arrange, Act, Assert (AAA)
test('example test', () {
  // Arrange - setup
  Calculator calc = Calculator();
  
  // Act - เรียก method ที่ test
  int result = calc.add(2, 3);
  
  // Assert - ตรวจผล
  expect(result, equals(5));
});

// ✅ ชื่อ test ควรบอก behavior
// ✅ "it should {behavior} when {condition}"
test('should return null when user not found', () {...});
test('should throw ArgumentError when discount is negative', () {...});

// ❌ ชื่อ test ที่ไม่ดี
test('test1', () {...});
test('calculator', () {...});

// ✅ Test coverage command:
// flutter test --coverage
// genhtml coverage/lcov.info -o coverage/html
```

---

**← [Part 16 - Firebase](part-16-firebase.md)**

**ต่อไป: [Part 18 - Flutter Animations →](part-18-animations.md)**
