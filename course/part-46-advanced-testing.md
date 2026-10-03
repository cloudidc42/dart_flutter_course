# Part 46: Advanced Testing in Flutter
## ขั้นตอนที่ 1721-1760

## 🎯 เป้าหมายของ Part นี้
- เขียน Unit Tests ด้วย mocktail สำหรับ mock dependencies
- เขียน Widget Tests เพื่อทดสอบ UI components
- เขียน Integration Tests สำหรับ user flows ที่สมบูรณ์
- ทำ Golden Tests (Screenshot Testing) เพื่อ visual regression testing
- ตั้งค่า Test Coverage และ CI integration

---

## ขั้นตอนที่ 1721: Project Setup และ pubspec.yaml

```yaml
# pubspec.yaml
name: advanced_testing_demo
description: Advanced Flutter Testing Demo
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.4.0
  http: ^1.1.0
  get_it: ^7.6.4

dev_dependencies:
  flutter_test:
    sdk: flutter
  mocktail: ^1.0.1
  integration_test:
    sdk: flutter
  golden_toolkit: ^0.15.0
  flutter_lints: ^3.0.0
  coverage: ^1.7.2

flutter:
  uses-material-design: true
  assets:
    - assets/images/
    - assets/fonts/
```

---

## ขั้นตอนที่ 1722: โครงสร้าง Test Files

```
test/
├── unit/
│   ├── repositories/
│   │   └── user_repository_test.dart
│   ├── usecases/
│   │   └── get_user_usecase_test.dart
│   └── models/
│       └── user_model_test.dart
├── widget/
│   ├── screens/
│   │   └── home_screen_test.dart
│   └── components/
│       └── user_card_test.dart
├── golden/
│   ├── goldens/
│   │   └── user_card.png  (auto-generated)
│   └── user_card_golden_test.dart
integration_test/
├── app_test.dart
└── flows/
    └── login_flow_test.dart
```

---

## ขั้นตอนที่ 1723: Domain Models

```dart
// lib/models/user.dart
class User {
  final int id;
  final String name;
  final String email;
  final String? avatarUrl;

  const User({
    required this.id,
    required this.name,
    required this.email,
    this.avatarUrl,
  });

  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'] as int,
      name: json['name'] as String,
      email: json['email'] as String,
      avatarUrl: json['avatar_url'] as String?,
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'name': name,
      'email': email,
      'avatar_url': avatarUrl,
    };
  }

  User copyWith({
    int? id,
    String? name,
    String? email,
    String? avatarUrl,
  }) {
    return User(
      id: id ?? this.id,
      name: name ?? this.name,
      email: email ?? this.email,
      avatarUrl: avatarUrl ?? this.avatarUrl,
    );
  }

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is User &&
        other.id == id &&
        other.name == name &&
        other.email == email;
  }

  @override
  int get hashCode => Object.hash(id, name, email);

  @override
  String toString() => 'User(id: $id, name: $name, email: $email)';
}
```

---

## ขั้นตอนที่ 1724: Repository Interface และ Implementation

```dart
// lib/repositories/user_repository.dart
abstract class UserRepository {
  Future<List<User>> getUsers();
  Future<User> getUserById(int id);
  Future<User> createUser({required String name, required String email});
  Future<void> deleteUser(int id);
}

// lib/repositories/user_repository_impl.dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import '../models/user.dart';
import 'user_repository.dart';

class UserRepositoryImpl implements UserRepository {
  final http.Client httpClient;
  final String baseUrl;

  const UserRepositoryImpl({
    required this.httpClient,
    this.baseUrl = 'https://jsonplaceholder.typicode.com',
  });

  @override
  Future<List<User>> getUsers() async {
    final response = await httpClient.get(
      Uri.parse('$baseUrl/users'),
    );

    if (response.statusCode == 200) {
      final List<dynamic> jsonList = json.decode(response.body);
      return jsonList.map((json) => User.fromJson(json)).toList();
    } else {
      throw Exception('Failed to load users: ${response.statusCode}');
    }
  }

  @override
  Future<User> getUserById(int id) async {
    final response = await httpClient.get(
      Uri.parse('$baseUrl/users/$id'),
    );

    if (response.statusCode == 200) {
      return User.fromJson(json.decode(response.body));
    } else if (response.statusCode == 404) {
      throw UserNotFoundException(id);
    } else {
      throw Exception('Failed to load user: ${response.statusCode}');
    }
  }

  @override
  Future<User> createUser({
    required String name,
    required String email,
  }) async {
    final response = await httpClient.post(
      Uri.parse('$baseUrl/users'),
      headers: {'Content-Type': 'application/json'},
      body: json.encode({'name': name, 'email': email}),
    );

    if (response.statusCode == 201) {
      return User.fromJson(json.decode(response.body));
    } else {
      throw Exception('Failed to create user: ${response.statusCode}');
    }
  }

  @override
  Future<void> deleteUser(int id) async {
    final response = await httpClient.delete(
      Uri.parse('$baseUrl/users/$id'),
    );

    if (response.statusCode != 200) {
      throw Exception('Failed to delete user: ${response.statusCode}');
    }
  }
}

class UserNotFoundException implements Exception {
  final int userId;
  const UserNotFoundException(this.userId);

  @override
  String toString() => 'UserNotFoundException: User with id $userId not found';
}
```

---

## ขั้นตอนที่ 1725: Unit Tests ด้วย mocktail

```dart
// test/unit/repositories/user_repository_test.dart
import 'dart:convert';
import 'package:flutter_test/flutter_test.dart';
import 'package:http/http.dart' as http;
import 'package:mocktail/mocktail.dart';
import '../../../lib/models/user.dart';
import '../../../lib/repositories/user_repository_impl.dart';

// Create mock classes using mocktail
class MockHttpClient extends Mock implements http.Client {}
class MockResponse extends Mock implements http.Response {}

void main() {
  late MockHttpClient mockHttpClient;
  late UserRepositoryImpl repository;

  // Setup before each test
  setUp(() {
    mockHttpClient = MockHttpClient();
    repository = UserRepositoryImpl(httpClient: mockHttpClient);

    // Register fallback values for complex types
    registerFallbackValue(Uri.parse('https://example.com'));
  });

  // Teardown after each test
  tearDown(() {
    reset(mockHttpClient);
  });

  group('UserRepositoryImpl.getUsers', () {
    final tUserList = [
      const User(id: 1, name: 'Alice', email: 'alice@example.com'),
      const User(id: 2, name: 'Bob', email: 'bob@example.com'),
    ];

    final tJsonResponse = json.encode([
      {'id': 1, 'name': 'Alice', 'email': 'alice@example.com'},
      {'id': 2, 'name': 'Bob', 'email': 'bob@example.com'},
    ]);

    test('should return list of users when HTTP call is successful', () async {
      // Arrange
      when(() => mockHttpClient.get(any())).thenAnswer(
        (_) async => http.Response(tJsonResponse, 200),
      );

      // Act
      final result = await repository.getUsers();

      // Assert
      expect(result, equals(tUserList));
      verify(() => mockHttpClient.get(
            Uri.parse('https://jsonplaceholder.typicode.com/users'),
          )).called(1);
    });

    test('should throw Exception when HTTP call returns non-200', () async {
      // Arrange
      when(() => mockHttpClient.get(any())).thenAnswer(
        (_) async => http.Response('Server Error', 500),
      );

      // Act & Assert
      expect(
        () => repository.getUsers(),
        throwsA(isA<Exception>()),
      );
    });

    test('should throw Exception when network is unavailable', () async {
      // Arrange
      when(() => mockHttpClient.get(any())).thenThrow(
        Exception('Network unavailable'),
      );

      // Act & Assert
      expect(
        () => repository.getUsers(),
        throwsA(isA<Exception>()),
      );
    });
  });

  group('UserRepositoryImpl.getUserById', () {
    const tUser = User(id: 1, name: 'Alice', email: 'alice@example.com');
    final tJsonResponse = json.encode({
      'id': 1,
      'name': 'Alice',
      'email': 'alice@example.com',
    });

    test('should return user when found', () async {
      // Arrange
      when(() => mockHttpClient.get(any())).thenAnswer(
        (_) async => http.Response(tJsonResponse, 200),
      );

      // Act
      final result = await repository.getUserById(1);

      // Assert
      expect(result, equals(tUser));
    });

    test('should throw UserNotFoundException when user not found (404)',
        () async {
      // Arrange
      when(() => mockHttpClient.get(any())).thenAnswer(
        (_) async => http.Response('Not Found', 404),
      );

      // Act & Assert
      expect(
        () => repository.getUserById(999),
        throwsA(isA<UserNotFoundException>()),
      );
    });
  });

  group('UserRepositoryImpl.createUser', () {
    test('should return created user on success', () async {
      // Arrange
      when(() => mockHttpClient.post(
            any(),
            headers: any(named: 'headers'),
            body: any(named: 'body'),
          )).thenAnswer(
        (_) async => http.Response(
          json.encode({'id': 11, 'name': 'Charlie', 'email': 'charlie@example.com'}),
          201,
        ),
      );

      // Act
      final result = await repository.createUser(
        name: 'Charlie',
        email: 'charlie@example.com',
      );

      // Assert
      expect(result.name, 'Charlie');
      expect(result.email, 'charlie@example.com');
      expect(result.id, 11);
    });
  });
}
```

---

## ขั้นตอนที่ 1726: UseCase Tests

```dart
// lib/usecases/get_users_usecase.dart
import '../models/user.dart';
import '../repositories/user_repository.dart';

class GetUsersUseCase {
  final UserRepository repository;

  const GetUsersUseCase({required this.repository});

  Future<List<User>> call() async {
    final users = await repository.getUsers();
    // Business logic: filter out inactive users (example)
    return users.where((u) => u.email.isNotEmpty).toList();
  }
}

// test/unit/usecases/get_users_usecase_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import '../../../lib/models/user.dart';
import '../../../lib/repositories/user_repository.dart';
import '../../../lib/usecases/get_users_usecase.dart';

class MockUserRepository extends Mock implements UserRepository {}

void main() {
  late MockUserRepository mockRepository;
  late GetUsersUseCase useCase;

  setUp(() {
    mockRepository = MockUserRepository();
    useCase = GetUsersUseCase(repository: mockRepository);
  });

  group('GetUsersUseCase', () {
    final tUsers = [
      const User(id: 1, name: 'Alice', email: 'alice@example.com'),
      const User(id: 2, name: 'Bob', email: 'bob@example.com'),
    ];

    test('should return users from repository', () async {
      // Arrange
      when(() => mockRepository.getUsers()).thenAnswer((_) async => tUsers);

      // Act
      final result = await useCase();

      // Assert
      expect(result, equals(tUsers));
      verify(() => mockRepository.getUsers()).called(1);
      verifyNoMoreInteractions(mockRepository);
    });

    test('should filter out users with empty emails', () async {
      // Arrange
      final usersWithEmpty = [
        const User(id: 1, name: 'Alice', email: 'alice@example.com'),
        const User(id: 2, name: 'Ghost', email: ''),
      ];
      when(() => mockRepository.getUsers())
          .thenAnswer((_) async => usersWithEmpty);

      // Act
      final result = await useCase();

      // Assert
      expect(result.length, 1);
      expect(result.first.name, 'Alice');
    });

    test('should propagate exceptions from repository', () async {
      // Arrange
      when(() => mockRepository.getUsers()).thenThrow(Exception('API Error'));

      // Act & Assert
      expect(() => useCase(), throwsA(isA<Exception>()));
    });
  });
}
```

---

## ขั้นตอนที่ 1727: Widget Tests

```dart
// lib/widgets/user_card.dart
import 'package:flutter/material.dart';
import '../models/user.dart';

class UserCard extends StatelessWidget {
  final User user;
  final VoidCallback? onTap;
  final VoidCallback? onDelete;

  const UserCard({
    super.key,
    required this.user,
    this.onTap,
    this.onDelete,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      key: Key('user_card_${user.id}'),
      child: ListTile(
        leading: CircleAvatar(
          backgroundColor: Theme.of(context).colorScheme.primary,
          child: Text(
            user.name[0].toUpperCase(),
            style: const TextStyle(color: Colors.white),
          ),
        ),
        title: Text(
          user.name,
          key: const Key('user_name_text'),
        ),
        subtitle: Text(
          user.email,
          key: const Key('user_email_text'),
        ),
        trailing: onDelete != null
            ? IconButton(
                key: const Key('delete_button'),
                icon: const Icon(Icons.delete),
                onPressed: onDelete,
              )
            : null,
        onTap: onTap,
      ),
    );
  }
}

// test/widget/components/user_card_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import '../../../lib/models/user.dart';
import '../../../lib/widgets/user_card.dart';

void main() {
  const tUser = User(id: 1, name: 'Alice Johnson', email: 'alice@example.com');

  group('UserCard Widget', () {
    testWidgets('should display user name and email', (tester) async {
      // Arrange & Act
      await tester.pumpWidget(
        const MaterialApp(
          home: Scaffold(
            body: UserCard(user: tUser),
          ),
        ),
      );

      // Assert
      expect(find.text('Alice Johnson'), findsOneWidget);
      expect(find.text('alice@example.com'), findsOneWidget);
    });

    testWidgets('should show avatar with first letter of name', (tester) async {
      await tester.pumpWidget(
        const MaterialApp(
          home: Scaffold(
            body: UserCard(user: tUser),
          ),
        ),
      );

      expect(find.text('A'), findsOneWidget);
    });

    testWidgets('should call onTap when tapped', (tester) async {
      bool tapped = false;

      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: UserCard(
              user: tUser,
              onTap: () => tapped = true,
            ),
          ),
        ),
      );

      await tester.tap(find.byType(ListTile));
      await tester.pump();

      expect(tapped, isTrue);
    });

    testWidgets('should show delete button when onDelete provided',
        (tester) async {
      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: UserCard(
              user: tUser,
              onDelete: () {},
            ),
          ),
        ),
      );

      expect(find.byKey(const Key('delete_button')), findsOneWidget);
    });

    testWidgets('should not show delete button when onDelete is null',
        (tester) async {
      await tester.pumpWidget(
        const MaterialApp(
          home: Scaffold(
            body: UserCard(user: tUser),
          ),
        ),
      );

      expect(find.byKey(const Key('delete_button')), findsNothing);
    });

    testWidgets('should call onDelete when delete button is pressed',
        (tester) async {
      bool deleted = false;

      await tester.pumpWidget(
        MaterialApp(
          home: Scaffold(
            body: UserCard(
              user: tUser,
              onDelete: () => deleted = true,
            ),
          ),
        ),
      );

      await tester.tap(find.byKey(const Key('delete_button')));
      await tester.pump();

      expect(deleted, isTrue);
    });
  });
}
```

---

## ขั้นตอนที่ 1728: Screen Widget Tests

```dart
// lib/screens/users_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/user.dart';
import '../providers/users_provider.dart';
import '../widgets/user_card.dart';

class UsersScreen extends ConsumerWidget {
  const UsersScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final usersAsync = ref.watch(usersProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Users'),
        key: const Key('users_appbar'),
      ),
      body: usersAsync.when(
        loading: () => const Center(
          child: CircularProgressIndicator(key: Key('loading_indicator')),
        ),
        error: (error, stack) => Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(Icons.error, size: 48, color: Colors.red),
              const SizedBox(height: 16),
              Text(
                'Error: $error',
                key: const Key('error_message'),
                textAlign: TextAlign.center,
              ),
              ElevatedButton(
                key: const Key('retry_button'),
                onPressed: () => ref.refresh(usersProvider),
                child: const Text('Retry'),
              ),
            ],
          ),
        ),
        data: (users) => users.isEmpty
            ? const Center(
                child: Text(
                  'No users found',
                  key: Key('empty_state'),
                ),
              )
            : ListView.builder(
                key: const Key('users_list'),
                itemCount: users.length,
                itemBuilder: (_, index) => UserCard(
                  user: users[index],
                  key: Key('user_card_${users[index].id}'),
                ),
              ),
      ),
    );
  }
}

// lib/providers/users_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/user.dart';
import '../repositories/user_repository.dart';

final usersProvider = FutureProvider<List<User>>((ref) async {
  final repository = ref.read(userRepositoryProvider);
  return repository.getUsers();
});

final userRepositoryProvider = Provider<UserRepository>((ref) {
  throw UnimplementedError('Override in tests');
});

// test/widget/screens/users_screen_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import '../../../lib/models/user.dart';
import '../../../lib/providers/users_provider.dart';
import '../../../lib/repositories/user_repository.dart';
import '../../../lib/screens/users_screen.dart';

class MockUserRepository extends Mock implements UserRepository {}

void main() {
  late MockUserRepository mockRepository;

  setUp(() {
    mockRepository = MockUserRepository();
  });

  Widget buildSubject() {
    return ProviderScope(
      overrides: [
        userRepositoryProvider.overrideWithValue(mockRepository),
      ],
      child: const MaterialApp(home: UsersScreen()),
    );
  }

  group('UsersScreen', () {
    testWidgets('shows loading indicator while fetching', (tester) async {
      when(() => mockRepository.getUsers()).thenAnswer(
        (_) async {
          await Future.delayed(const Duration(seconds: 1));
          return [];
        },
      );

      await tester.pumpWidget(buildSubject());

      expect(find.byKey(const Key('loading_indicator')), findsOneWidget);
    });

    testWidgets('shows users when loaded successfully', (tester) async {
      final users = [
        const User(id: 1, name: 'Alice', email: 'alice@test.com'),
        const User(id: 2, name: 'Bob', email: 'bob@test.com'),
      ];

      when(() => mockRepository.getUsers()).thenAnswer((_) async => users);

      await tester.pumpWidget(buildSubject());
      await tester.pumpAndSettle();

      expect(find.byKey(const Key('users_list')), findsOneWidget);
      expect(find.text('Alice'), findsOneWidget);
      expect(find.text('Bob'), findsOneWidget);
    });

    testWidgets('shows error state on failure', (tester) async {
      when(() => mockRepository.getUsers())
          .thenThrow(Exception('Network Error'));

      await tester.pumpWidget(buildSubject());
      await tester.pumpAndSettle();

      expect(find.byKey(const Key('error_message')), findsOneWidget);
      expect(find.byKey(const Key('retry_button')), findsOneWidget);
    });

    testWidgets('shows empty state when no users', (tester) async {
      when(() => mockRepository.getUsers()).thenAnswer((_) async => []);

      await tester.pumpWidget(buildSubject());
      await tester.pumpAndSettle();

      expect(find.byKey(const Key('empty_state')), findsOneWidget);
    });
  });
}
```

---

## ขั้นตอนที่ 1729: Golden Tests (Screenshot Testing)

```dart
// test/golden/user_card_golden_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:golden_toolkit/golden_toolkit.dart';
import '../../lib/models/user.dart';
import '../../lib/widgets/user_card.dart';

void main() {
  setUpAll(() async {
    // Load fonts for golden tests
    await loadAppFonts();
  });

  group('UserCard Golden Tests', () {
    testGoldens('UserCard renders correctly in light theme', (tester) async {
      const user = User(id: 1, name: 'Alice Johnson', email: 'alice@test.com');

      await tester.pumpWidgetBuilder(
        const UserCard(user: user),
        wrapper: materialAppWrapper(
          theme: ThemeData.light(),
        ),
        surfaceSize: const Size(400, 80),
      );

      await screenMatchesGolden(tester, 'user_card_light');
    });

    testGoldens('UserCard renders correctly in dark theme', (tester) async {
      const user = User(id: 1, name: 'Alice Johnson', email: 'alice@test.com');

      await tester.pumpWidgetBuilder(
        const UserCard(user: user),
        wrapper: materialAppWrapper(
          theme: ThemeData.dark(),
        ),
        surfaceSize: const Size(400, 80),
      );

      await screenMatchesGolden(tester, 'user_card_dark');
    });

    testGoldens('UserCard with delete button renders correctly', (tester) async {
      const user = User(id: 1, name: 'Alice Johnson', email: 'alice@test.com');

      await tester.pumpWidgetBuilder(
        UserCard(user: user, onDelete: () {}),
        wrapper: materialAppWrapper(
          theme: ThemeData.light(),
        ),
        surfaceSize: const Size(400, 80),
      );

      await screenMatchesGolden(tester, 'user_card_with_delete');
    });

    testGoldens('UserCard multi-device golden test', (tester) async {
      const user = User(id: 1, name: 'Alice Johnson', email: 'alice@test.com');

      await tester.pumpWidgetBuilder(
        const UserCard(user: user),
        wrapper: materialAppWrapper(theme: ThemeData.light()),
      );

      await multiScreenGolden(
        tester,
        'user_card_multi_device',
        devices: [
          Device.phone,
          Device.iphone11,
          Device.tabletLandscape,
        ],
      );
    });
  });
}
```

---

## ขั้นตอนที่ 1730: Integration Tests

```dart
// integration_test/app_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:advanced_testing_demo/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('App Integration Tests', () {
    testWidgets('full user flow test', (tester) async {
      // Launch the app
      app.main();
      await tester.pumpAndSettle();

      // Verify home screen loads
      expect(find.byKey(const Key('users_appbar')), findsOneWidget);

      // Wait for users to load
      await tester.pumpAndSettle(const Duration(seconds: 3));

      // Verify users list is displayed
      expect(find.byType(ListView), findsOneWidget);

      // Tap on first user
      final firstUserCard = find.byType(ListTile).first;
      await tester.tap(firstUserCard);
      await tester.pumpAndSettle();

      // Verify navigation to user details
      expect(find.byKey(const Key('user_detail_screen')), findsOneWidget);
    });
  });
}

// integration_test/flows/login_flow_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:advanced_testing_demo/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('Login Flow Integration Tests', () {
    testWidgets('user can log in with valid credentials', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Find login form elements
      final emailField = find.byKey(const Key('email_field'));
      final passwordField = find.byKey(const Key('password_field'));
      final loginButton = find.byKey(const Key('login_button'));

      // Enter credentials
      await tester.enterText(emailField, 'test@example.com');
      await tester.enterText(passwordField, 'password123');

      // Dismiss keyboard
      await tester.testTextInput.receiveAction(TextInputAction.done);
      await tester.pump();

      // Tap login
      await tester.tap(loginButton);
      await tester.pumpAndSettle(const Duration(seconds: 3));

      // Verify logged in
      expect(find.byKey(const Key('home_screen')), findsOneWidget);
      expect(find.text('Welcome!'), findsOneWidget);
    });

    testWidgets('shows error on invalid credentials', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      await tester.enterText(
        find.byKey(const Key('email_field')),
        'wrong@example.com',
      );
      await tester.enterText(
        find.byKey(const Key('password_field')),
        'wrongpassword',
      );

      await tester.tap(find.byKey(const Key('login_button')));
      await tester.pumpAndSettle(const Duration(seconds: 3));

      expect(find.byKey(const Key('error_snackbar')), findsOneWidget);
    });
  });
}
```

---

## ขั้นตอนที่ 1731: Test Coverage Setup

```bash
# scripts/run_coverage.sh
#!/bin/bash
set -e

echo "🧪 Running Flutter Tests with Coverage..."

# Run tests with coverage
flutter test --coverage

# Generate HTML coverage report (requires lcov)
if command -v genhtml &> /dev/null; then
  genhtml coverage/lcov.info -o coverage/html
  echo "📊 Coverage report generated at coverage/html/index.html"
else
  echo "⚠️  Install lcov to generate HTML report: brew install lcov"
fi

# Print coverage summary
if command -v lcov &> /dev/null; then
  lcov --summary coverage/lcov.info
fi

echo "✅ Done!"
```

```yaml
# .github/workflows/test.yml
name: Flutter Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.16.0'
          channel: 'stable'

      - name: Install dependencies
        run: flutter pub get

      - name: Run unit and widget tests
        run: flutter test --coverage

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: coverage/lcov.info
          fail_ci_if_error: true
```

---

## ขั้นตอนที่ 1732: Test Helpers และ Utilities

```dart
// test/helpers/test_helpers.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import '../../lib/repositories/user_repository.dart';
import '../../lib/providers/users_provider.dart';

class MockUserRepository extends Mock implements UserRepository {}

/// Helper to pump a widget with all necessary providers
extension WidgetTesterExtension on WidgetTester {
  Future<void> pumpApp(
    Widget widget, {
    List<Override> overrides = const [],
    ThemeData? theme,
  }) async {
    await pumpWidget(
      ProviderScope(
        overrides: overrides,
        child: MaterialApp(
          theme: theme ?? ThemeData.light(),
          home: widget,
        ),
      ),
    );
  }
}

/// Creates a mock repository with default behavior
MockUserRepository createMockRepository({
  List<dynamic>? users,
  Exception? error,
}) {
  final mock = MockUserRepository();
  if (error != null) {
    when(() => mock.getUsers()).thenThrow(error);
  } else {
    when(() => mock.getUsers()).thenAnswer((_) async => users ?? []);
  }
  return mock;
}

// test/helpers/pump_app.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

Future<void> pumpMaterialApp(
  WidgetTester tester,
  Widget widget, {
  ThemeData? theme,
}) async {
  await tester.pumpWidget(
    MaterialApp(
      theme: theme,
      home: Scaffold(body: widget),
    ),
  );
}
```

---

## ขั้นตอนที่ 1733: Async Test Patterns

```dart
// test/unit/async_patterns_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

// Testing async streams
class DataStream {
  Stream<int> countStream(int max) async* {
    for (int i = 0; i <= max; i++) {
      await Future.delayed(const Duration(milliseconds: 10));
      yield i;
    }
  }
}

void main() {
  group('Async Pattern Tests', () {
    test('stream emits values in order', () async {
      final stream = DataStream();
      final values = await stream.countStream(3).toList();
      expect(values, [0, 1, 2, 3]);
    });

    test('future completes with correct value', () async {
      Future<int> delayedValue() async {
        await Future.delayed(const Duration(milliseconds: 50));
        return 42;
      }

      final result = await delayedValue();
      expect(result, 42);
    });

    test('future throws expected exception', () async {
      Future<int> failingFuture() async {
        await Future.delayed(const Duration(milliseconds: 10));
        throw ArgumentError('Bad input');
      }

      expect(failingFuture, throwsA(isA<ArgumentError>()));
    });

    test('completes within timeout', () async {
      Future<String> slowOperation() async {
        await Future.delayed(const Duration(milliseconds: 100));
        return 'done';
      }

      final result = await slowOperation().timeout(
        const Duration(seconds: 1),
      );
      expect(result, 'done');
    });
  });
}
```

---

## ขั้นตอนที่ 1734: Faking Time in Tests

```dart
// test/unit/fake_async_test.dart
import 'package:fake_async/fake_async.dart';
import 'package:flutter_test/flutter_test.dart';

class Debouncer {
  final Duration delay;
  void Function()? _callback;
  Timer? _timer;

  Debouncer({required this.delay});

  void run(void Function() callback) {
    _callback = callback;
    _timer?.cancel();
    _timer = Timer(delay, () => _callback?.call());
  }

  void dispose() {
    _timer?.cancel();
  }
}

void main() {
  group('Debouncer Tests with FakeAsync', () {
    test('only calls callback once after delay', () {
      fakeAsync((fake) {
        int callCount = 0;
        final debouncer = Debouncer(delay: const Duration(seconds: 1));

        debouncer.run(() => callCount++);
        debouncer.run(() => callCount++);
        debouncer.run(() => callCount++);

        expect(callCount, 0); // Not called yet

        fake.elapse(const Duration(milliseconds: 999));
        expect(callCount, 0); // Still not called

        fake.elapse(const Duration(milliseconds: 1));
        expect(callCount, 1); // Called exactly once

        debouncer.dispose();
      });
    });
  });
}
```

---

## ขั้นตอนที่ 1735: Running Tests and Coverage Commands

```bash
# Run all unit tests
flutter test test/unit/

# Run widget tests
flutter test test/widget/

# Run all tests with verbose output
flutter test --reporter=expanded

# Run specific test file
flutter test test/unit/repositories/user_repository_test.dart

# Run tests with coverage
flutter test --coverage

# Update golden files
flutter test --update-goldens test/golden/

# Run integration tests on connected device
flutter test integration_test/app_test.dart

# Run integration tests with specific driver
flutter drive \
  --driver=test_driver/integration_test.dart \
  --target=integration_test/app_test.dart

# Check test coverage threshold
flutter test --coverage && \
  lcov --summary coverage/lcov.info | \
  grep "lines" | \
  awk '{if ($2 < 80) exit 1}'
```

---

**← [Part 45 - DevTools & Profiling](part-45-devtools-profiling.md)**
**ต่อไป: [Part 47 - Advanced Extensions & Mixins →](part-47-dart-advanced-extensions-mixins.md)**
