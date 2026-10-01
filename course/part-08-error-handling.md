# Part 08: Error Handling และ Exceptions
## ขั้นตอนที่ 211-240

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Exception Hierarchy ใน Dart
- ใช้ try-catch-finally
- สร้าง Custom Exceptions
- เข้าใจ Error vs Exception
- ใช้ Result Pattern
- จัดการ Async Errors
- เขียน Defensive Programming

---

## ขั้นตอนที่ 211: Exception พื้นฐาน

```dart
void main() {
  // ─────────────── try-catch ───────────────
  try {
    int result = 10 ~/ 0;  // หารด้วยศูนย์
    print(result);
  } catch (e) {
    print('Error: $e');
  }
  
  // ─────────────── catch กับ StackTrace ───────────────
  try {
    List<int> list = [1, 2, 3];
    print(list[10]);  // Index out of range
  } catch (e, stackTrace) {
    print('Error: $e');
    print('Stack trace: $stackTrace');
  }
  
  // ─────────────── catch เฉพาะ type ───────────────
  try {
    String text = 'abc';
    int number = int.parse(text);
    print(number);
  } on FormatException catch (e) {
    print('Format error: ${e.message}');
  } on RangeError catch (e) {
    print('Range error: $e');
  } catch (e) {
    print('Unknown error: $e');
  }
  
  // ─────────────── finally ───────────────
  // finally ทำงานเสมอ ไม่ว่าจะมี error หรือไม่
  try {
    print('ทำงานปกติ');
    // throw Exception('Test error');
  } catch (e) {
    print('จัดการ error');
  } finally {
    print('finally เสมอ');
  }
  
  // ─────────────── rethrow ───────────────
  void processData(String data) {
    try {
      int value = int.parse(data);
      if (value < 0) throw RangeError('ค่าต้องไม่ติดลบ');
    } on FormatException {
      print('รูปแบบข้อมูลผิด: $data');
      rethrow;  // โยน exception ต่อไปยัง caller
    }
  }
  
  try {
    processData('abc');
  } catch (e) {
    print('Caught in main: $e');
  }
}
```

---

## ขั้นตอนที่ 212: Exception Hierarchy

```dart
void main() {
  // ─────────────── Exception Hierarchy ───────────────
  //
  // Object
  //  └── Throwable (ไม่มีใน Dart จริง แต่แนวคิด)
  //      ├── Error (ปัญหาร้ายแรงจากโปรแกรม)
  //      │   ├── AssertionError
  //      │   ├── NoSuchMethodError
  //      │   ├── NullThrownError
  //      │   ├── OutOfMemoryError
  //      │   ├── RangeError
  //      │   ├── StackOverflowError
  //      │   ├── StateError
  //      │   ├── TypeError
  //      │   └── UnimplementedError
  //      └── Exception (สิ่งที่คาดว่าอาจเกิดขึ้นได้)
  //          ├── FormatException
  //          ├── IOException
  //          │   └── FileSystemException
  //          ├── IsolateSpawnException
  //          └── TimeoutException
  
  // ─────────────── Common Exceptions ───────────────
  
  // FormatException: รูปแบบข้อมูลผิด
  try {
    int.parse('not a number');
  } on FormatException catch (e) {
    print('FormatException: ${e.message}');
  }
  
  // RangeError: ค่าอยู่นอกช่วงที่กำหนด
  try {
    List<int> list = [1, 2, 3];
    print(list[5]);
  } on RangeError catch (e) {
    print('RangeError: $e');
  }
  
  // StateError: Object อยู่ในสถานะที่ไม่ถูกต้อง
  try {
    List<int> empty = [];
    empty.removeLast();
  } on StateError catch (e) {
    print('StateError: $e');
  }
  
  // ArgumentError: Argument ไม่ถูกต้อง
  try {
    throw ArgumentError.value(-5, 'age', 'อายุต้องไม่ติดลบ');
  } on ArgumentError catch (e) {
    print('ArgumentError: $e');
  }
  
  // TypeError: Type ไม่ถูกต้อง
  try {
    Object value = 'hello';
    int number = value as int;  // cast ผิด type
  } on TypeError catch (e) {
    print('TypeError: $e');
  }
  
  // UnimplementedError: Method ยังไม่ได้ implement
  try {
    throw UnimplementedError('method ยังไม่ได้ implement');
  } on UnimplementedError catch (e) {
    print('UnimplementedError: $e');
  }
}
```

---

## ขั้นตอนที่ 213: Custom Exceptions

```dart
// ─────────────── Basic Custom Exception ───────────────
class AppException implements Exception {
  final String message;
  final String? code;
  
  const AppException(this.message, {this.code});
  
  @override
  String toString() => code != null 
      ? 'AppException[$code]: $message'
      : 'AppException: $message';
}

// ─────────────── Specific Exceptions ───────────────
class ValidationException extends AppException {
  final Map<String, String> errors;
  
  const ValidationException(this.errors)
      : super('Validation failed');
  
  @override
  String toString() {
    String errList = errors.entries
        .map((e) => '${e.key}: ${e.value}')
        .join(', ');
    return 'ValidationException: {$errList}';
  }
}

class AuthException extends AppException {
  const AuthException(String message) 
      : super(message, code: 'AUTH_ERROR');
}

class NetworkException extends AppException {
  final int? statusCode;
  
  const NetworkException(String message, {this.statusCode})
      : super(message, code: 'NETWORK_ERROR');
  
  bool get isNotFound => statusCode == 404;
  bool get isUnauthorized => statusCode == 401;
  bool get isServerError => statusCode != null && statusCode! >= 500;
  
  @override
  String toString() => statusCode != null
      ? 'NetworkException[$statusCode]: $message'
      : super.toString();
}

class DatabaseException extends AppException {
  const DatabaseException(String message)
      : super(message, code: 'DB_ERROR');
}

// ─────────────── การใช้งาน ───────────────
class UserService {
  void createUser(String name, String email, String password) {
    Map<String, String> errors = {};
    
    // Validation
    if (name.trim().isEmpty) errors['name'] = 'ชื่อต้องไม่ว่าง';
    if (!email.contains('@')) errors['email'] = 'Email ไม่ถูกต้อง';
    if (password.length < 8) errors['password'] = 'Password ต้องมีอย่างน้อย 8 ตัวอักษร';
    
    if (errors.isNotEmpty) {
      throw ValidationException(errors);
    }
    
    // สมมติว่า user มีอยู่แล้ว
    if (email == 'exists@example.com') {
      throw AppException('Email นี้ถูกใช้แล้ว', code: 'DUPLICATE_EMAIL');
    }
    
    print('สร้าง User สำเร็จ: $name ($email)');
  }
  
  String login(String email, String password) {
    if (email.isEmpty || password.isEmpty) {
      throw AuthException('กรุณากรอก Email และ Password');
    }
    
    if (email != 'user@example.com' || password != 'password123') {
      throw AuthException('Email หรือ Password ไม่ถูกต้อง');
    }
    
    return 'token_${DateTime.now().millisecondsSinceEpoch}';
  }
}

void main() {
  UserService service = UserService();
  
  // Test validation
  print('--- Test Validation ---');
  try {
    service.createUser('', 'invalid-email', '123');
  } on ValidationException catch (e) {
    print(e);
    e.errors.forEach((field, error) {
      print('  ❌ $field: $error');
    });
  }
  
  // Test auth
  print('\n--- Test Auth ---');
  try {
    service.login('wrong@email.com', 'wrongpass');
  } on AuthException catch (e) {
    print(e);
  }
  
  // Test success
  print('\n--- Test Success ---');
  try {
    service.createUser('Alice', 'alice@example.com', 'secure_pass');
    String token = service.login('user@example.com', 'password123');
    print('Login สำเร็จ! Token: $token');
  } on AppException catch (e) {
    print(e);
  }
}
```

---

## ขั้นตอนที่ 214: Result Pattern

```dart
// ─────────────── Result Type ───────────────
sealed class Result<T> {
  const Result();
  
  bool get isSuccess => this is Success<T>;
  bool get isFailure => this is Failure<T>;
  
  T get valueOrThrow {
    return switch (this) {
      Success(:var value) => value,
      Failure(:var error) => throw error,
    };
  }
  
  T getOrElse(T defaultValue) {
    return switch (this) {
      Success(:var value) => value,
      Failure() => defaultValue,
    };
  }
  
  Result<R> map<R>(R Function(T) fn) {
    return switch (this) {
      Success(:var value) => Success(fn(value)),
      Failure(:var error) => Failure(error),
    };
  }
  
  Result<R> flatMap<R>(Result<R> Function(T) fn) {
    return switch (this) {
      Success(:var value) => fn(value),
      Failure(:var error) => Failure(error),
    };
  }
  
  void when({
    required void Function(T) onSuccess,
    required void Function(Exception) onFailure,
  }) {
    switch (this) {
      case Success(:var value):
        onSuccess(value);
      case Failure(:var error):
        onFailure(error);
    }
  }
}

class Success<T> extends Result<T> {
  final T value;
  const Success(this.value);
  
  @override
  String toString() => 'Success($value)';
}

class Failure<T> extends Result<T> {
  final Exception error;
  const Failure(this.error);
  
  @override
  String toString() => 'Failure($error)';
}

// ─────────────── ใช้ Result Pattern ───────────────
class ApiService {
  Result<Map<String, dynamic>> getUser(String id) {
    try {
      if (id.isEmpty) {
        return Failure(ArgumentError('ID ต้องไม่ว่าง'));
      }
      if (id == 'not_found') {
        return Failure(NetworkException('ไม่พบ User', statusCode: 404));
      }
      return Success({'id': id, 'name': 'User $id', 'email': '$id@example.com'});
    } catch (e) {
      return Failure(AppException('Unexpected error: $e'));
    }
  }
  
  Result<String> getUserEmail(String userId) {
    return getUser(userId).map((user) => user['email'] as String);
  }
  
  Result<String> getInitials(String userId) {
    return getUserEmail(userId).flatMap((email) {
      if (!email.contains('@')) {
        return Failure(FormatException('Invalid email: $email'));
      }
      String name = email.split('@')[0];
      return Success(name[0].toUpperCase());
    });
  }
}

class NetworkException implements Exception {
  final String message;
  final int? statusCode;
  NetworkException(this.message, {this.statusCode});
  @override
  String toString() => 'NetworkException[$statusCode]: $message';
}

class AppException implements Exception {
  final String message;
  AppException(this.message);
  @override
  String toString() => 'AppException: $message';
}

void main() {
  ApiService api = ApiService();
  
  // Success case
  print('--- Success ---');
  var result1 = api.getUser('alice');
  result1.when(
    onSuccess: (user) => print('Got user: $user'),
    onFailure: (e) => print('Error: $e'),
  );
  
  // Failure case
  print('\n--- Not Found ---');
  var result2 = api.getUser('not_found');
  print(result2);
  print('Value or default: ${result2.getOrElse({'id': 'unknown'})}');
  
  // Chained operations
  print('\n--- Chain ---');
  var email = api.getUserEmail('bob');
  print('Email: $email');
  
  var initials = api.getInitials('charlie');
  print('Initials: $initials');
  
  // Error case
  var emptyResult = api.getInitials('');
  print('Empty ID: $emptyResult');
}
```

---

## ขั้นตอนที่ 215: Async Error Handling

```dart
import 'dart:async';

// ─────────────── Async Exceptions ───────────────
Future<int> fetchData(String url) async {
  await Future.delayed(Duration(milliseconds: 100));  // จำลอง network
  
  if (url.isEmpty) throw ArgumentError('URL ต้องไม่ว่าง');
  if (url.contains('error')) throw NetworkException('Server Error', statusCode: 500);
  if (url.contains('notfound')) throw NetworkException('Not Found', statusCode: 404);
  
  return 42;
}

class NetworkException implements Exception {
  final String message;
  final int? statusCode;
  NetworkException(this.message, {this.statusCode});
  @override
  String toString() => 'NetworkException[$statusCode]: $message';
}

Future<void> asyncErrorHandling() async {
  // ─────────────── try-catch กับ async ───────────────
  try {
    int data = await fetchData('https://api.example.com/data');
    print('Data: $data');
  } on NetworkException catch (e) {
    print('Network error: $e');
  } on ArgumentError catch (e) {
    print('Argument error: $e');
  } catch (e) {
    print('Unknown error: $e');
  }
  
  // ─────────────── Future .then().catchError() ───────────────
  fetchData('https://api.example.com/error')
      .then((data) => print('Data: $data'))
      .catchError((e) {
        print('Caught: $e');
        return 0;  // default value
      });
  
  // ─────────────── Future.wait กับ errors ───────────────
  List<String> urls = [
    'https://api.example.com/1',
    'https://api.example.com/error',
    'https://api.example.com/3',
  ];
  
  try {
    List<int> results = await Future.wait(
      urls.map((url) => fetchData(url)),
      eagerError: true,  // ถ้า error หนึ่งตัว หยุดทันที
    );
    print('All results: $results');
  } catch (e) {
    print('One or more failed: $e');
  }
  
  // ─────────────── Timeout ───────────────
  try {
    int result = await fetchData('https://api.example.com/data')
        .timeout(
          Duration(milliseconds: 50),  // timeout 50ms
          onTimeout: () => throw TimeoutException('Request timeout'),
        );
    print('Result: $result');
  } on TimeoutException catch (e) {
    print('Timeout: $e');
  }
}

// ─────────────── Stream Error Handling ───────────────
Stream<int> generateNumbers() async* {
  for (int i = 1; i <= 5; i++) {
    if (i == 3) throw Exception('Error at 3');
    yield i;
  }
}

Future<void> streamErrorHandling() async {
  // handleError ใน stream
  await generateNumbers()
      .handleError(
        (error) => print('Stream error: $error'),
        test: (e) => e is Exception,
      )
      .forEach((n) => print('Number: $n'));
  
  // try-catch กับ await for
  try {
    await for (int n in generateNumbers()) {
      print('Got: $n');
    }
  } catch (e) {
    print('Stream caught: $e');
  }
}

void main() async {
  print('=== Async Error Handling ===\n');
  
  await asyncErrorHandling();
  
  print('\n=== Stream Error Handling ===\n');
  
  await streamErrorHandling();
}
```

---

## ขั้นตอนที่ 216: Defensive Programming

```dart
// ─────────────── Assertions ───────────────
class Rectangle {
  final double width;
  final double height;
  
  Rectangle(this.width, this.height)
      : assert(width > 0, 'Width must be positive'),
        assert(height > 0, 'Height must be positive');
  
  double get area => width * height;
  double get diagonal {
    assert(width > 0 && height > 0, 'Invalid dimensions');
    return (width * width + height * height).sqrt2();
  }
}

extension on double {
  double sqrt2() {
    if (this <= 0) return 0;
    double r = this;
    for (int i = 0; i < 50; i++) r = (r + this / r) / 2;
    return r;
  }
}

// ─────────────── Guard Clauses ───────────────
class UserValidator {
  // แบบที่ไม่ดี: nested if
  String? validateBad(String? name, String? email, int? age) {
    if (name != null) {
      if (name.isNotEmpty) {
        if (email != null) {
          if (email.contains('@')) {
            if (age != null) {
              if (age >= 18) {
                return null; // valid
              } else {
                return 'อายุน้อยกว่า 18 ปี';
              }
            } else {
              return 'กรุณาระบุอายุ';
            }
          } else {
            return 'Email ไม่ถูกต้อง';
          }
        } else {
          return 'กรุณาระบุ Email';
        }
      } else {
        return 'ชื่อต้องไม่ว่าง';
      }
    } else {
      return 'กรุณาระบุชื่อ';
    }
  }
  
  // แบบที่ดี: Guard Clauses
  String? validate(String? name, String? email, int? age) {
    if (name == null) return 'กรุณาระบุชื่อ';
    if (name.isEmpty) return 'ชื่อต้องไม่ว่าง';
    if (email == null) return 'กรุณาระบุ Email';
    if (!email.contains('@')) return 'Email ไม่ถูกต้อง';
    if (age == null) return 'กรุณาระบุอายุ';
    if (age < 18) return 'อายุน้อยกว่า 18 ปี';
    return null; // valid
  }
}

// ─────────────── Contract Programming ───────────────
class MathLibrary {
  // Precondition: ตรวจสอบ input
  // Postcondition: ตรวจสอบ output
  // Invariant: สิ่งที่ต้องเป็นจริงเสมอ
  
  double sqrt(double n) {
    // Precondition
    if (n < 0) throw ArgumentError('Cannot take sqrt of negative: $n');
    
    if (n == 0) return 0;
    
    double result = n;
    for (int i = 0; i < 100; i++) {
      result = (result + n / result) / 2;
    }
    
    // Postcondition
    assert((result * result - n).abs() < 0.0001, 'sqrt result is incorrect');
    
    return result;
  }
  
  int factorial(int n) {
    // Precondition
    if (n < 0) throw ArgumentError('Factorial undefined for negative: $n');
    
    int result = 1;
    for (int i = 2; i <= n; i++) {
      result *= i;
    }
    
    // Postcondition
    assert(result >= 1, 'Factorial must be >= 1');
    
    return result;
  }
}

void main() {
  // Assertions (เฉพาะ debug mode)
  try {
    Rectangle r1 = Rectangle(5, 3);
    print('Area: ${r1.area}');
    
    // จะ throw ใน debug mode
    // Rectangle r2 = Rectangle(-1, 3);
  } catch (e) {
    print('Error: $e');
  }
  
  // Guard Clauses
  UserValidator validator = UserValidator();
  
  List<(String?, String?, int?)> testCases = [
    (null, null, null),
    ('', 'test@email.com', 25),
    ('Alice', 'invalid-email', 25),
    ('Alice', 'alice@email.com', 16),
    ('Alice', 'alice@email.com', 25),  // valid
  ];
  
  for (var (name, email, age) in testCases) {
    String? error = validator.validate(name, email, age);
    print(error == null ? '✅ Valid' : '❌ $error');
  }
}
```

---

## ขั้นตอนที่ 217-220: โปรเจกต์ - Form Validation System

```dart
// form_validation.dart

typedef Validator<T> = String? Function(T value);

class FormField<T> {
  final String name;
  final T? value;
  final List<Validator<T>> validators;
  
  FormField({
    required this.name,
    this.value,
    this.validators = const [],
  });
  
  List<String> validate() {
    if (value == null) return [];
    
    return validators
        .map((v) => v(value as T))
        .where((e) => e != null)
        .cast<String>()
        .toList();
  }
  
  bool get isValid => validate().isEmpty;
}

class Validators {
  // Required
  static Validator<String> required([String message = 'จำเป็นต้องกรอก']) {
    return (value) => value.trim().isEmpty ? message : null;
  }
  
  // Min length
  static Validator<String> minLength(int min, [String? message]) {
    return (value) => value.length < min
        ? (message ?? 'ต้องมีอย่างน้อย $min ตัวอักษร')
        : null;
  }
  
  // Max length
  static Validator<String> maxLength(int max, [String? message]) {
    return (value) => value.length > max
        ? (message ?? 'ต้องมีไม่เกิน $max ตัวอักษร')
        : null;
  }
  
  // Email
  static Validator<String> email([String message = 'Email ไม่ถูกต้อง']) {
    RegExp emailRegex = RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$');
    return (value) => emailRegex.hasMatch(value) ? null : message;
  }
  
  // Pattern
  static Validator<String> pattern(RegExp regex, String message) {
    return (value) => regex.hasMatch(value) ? null : message;
  }
  
  // Numeric
  static Validator<String> numeric([String message = 'ต้องเป็นตัวเลขเท่านั้น']) {
    return (value) => RegExp(r'^\d+$').hasMatch(value) ? null : message;
  }
  
  // Range (for numbers)
  static Validator<num> range(num min, num max, [String? message]) {
    return (value) => value < min || value > max
        ? (message ?? 'ต้องอยู่ระหว่าง $min และ $max')
        : null;
  }
  
  // Custom
  static Validator<T> custom<T>(
    bool Function(T) check,
    String message,
  ) {
    return (value) => check(value) ? null : message;
  }
}

class Form {
  final Map<String, FormField> _fields = {};
  
  void addField(FormField field) {
    _fields[field.name] = field;
  }
  
  Map<String, List<String>> validate() {
    Map<String, List<String>> errors = {};
    
    for (var field in _fields.values) {
      List<String> fieldErrors = field.validate();
      if (fieldErrors.isNotEmpty) {
        errors[field.name] = fieldErrors;
      }
    }
    
    return errors;
  }
  
  bool get isValid => validate().isEmpty;
  
  void printErrors() {
    var errors = validate();
    if (errors.isEmpty) {
      print('✅ ข้อมูลถูกต้องทั้งหมด');
    } else {
      print('❌ พบข้อผิดพลาด:');
      errors.forEach((field, errs) {
        for (var err in errs) {
          print('  $field: $err');
        }
      });
    }
  }
}

void main() {
  // Registration Form
  Form registrationForm = Form();
  
  registrationForm.addField(FormField<String>(
    name: 'username',
    value: 'ab',  // too short
    validators: [
      Validators.required(),
      Validators.minLength(3, 'ชื่อผู้ใช้ต้องมีอย่างน้อย 3 ตัว'),
      Validators.maxLength(20, 'ชื่อผู้ใช้ต้องมีไม่เกิน 20 ตัว'),
      Validators.pattern(
        RegExp(r'^[a-zA-Z0-9_]+$'),
        'ใช้ได้เฉพาะ a-z, A-Z, 0-9, และ _',
      ),
    ],
  ));
  
  registrationForm.addField(FormField<String>(
    name: 'email',
    value: 'invalid-email',
    validators: [
      Validators.required(),
      Validators.email(),
    ],
  ));
  
  registrationForm.addField(FormField<String>(
    name: 'password',
    value: 'weak',
    validators: [
      Validators.required(),
      Validators.minLength(8),
      Validators.custom(
        (p) => RegExp(r'[A-Z]').hasMatch(p),
        'ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว',
      ),
      Validators.custom(
        (p) => RegExp(r'[0-9]').hasMatch(p),
        'ต้องมีตัวเลขอย่างน้อย 1 ตัว',
      ),
    ],
  ));
  
  registrationForm.addField(FormField<num>(
    name: 'age',
    value: 15,  // too young
    validators: [
      Validators.range(18, 100, 'ต้องมีอายุ 18-100 ปี'),
    ],
  ));
  
  print('Registration Form Validation:');
  registrationForm.printErrors();
  
  // Valid Form
  print('\nValid Form:');
  Form validForm = Form();
  
  validForm.addField(FormField<String>(
    name: 'username',
    value: 'alice_developer',
    validators: [
      Validators.required(),
      Validators.minLength(3),
      Validators.maxLength(20),
    ],
  ));
  
  validForm.addField(FormField<String>(
    name: 'email',
    value: 'alice@example.com',
    validators: [
      Validators.required(),
      Validators.email(),
    ],
  ));
  
  validForm.addField(FormField<String>(
    name: 'password',
    value: 'SecurePass123',
    validators: [
      Validators.required(),
      Validators.minLength(8),
    ],
  ));
  
  validForm.printErrors();
}
```

---

## ขั้นตอนที่ 221-240: สรุปและ Advanced Error Handling

### Error Logging System

```dart
// error_logging.dart

enum LogLevel { debug, info, warning, error, critical }

class LogEntry {
  final LogLevel level;
  final String message;
  final DateTime timestamp;
  final Map<String, dynamic>? context;
  final Exception? exception;
  
  LogEntry({
    required this.level,
    required this.message,
    this.context,
    this.exception,
  }) : timestamp = DateTime.now();
  
  @override
  String toString() {
    String emoji = switch (level) {
      LogLevel.debug => '🔍',
      LogLevel.info => 'ℹ️',
      LogLevel.warning => '⚠️',
      LogLevel.error => '❌',
      LogLevel.critical => '🔴',
    };
    
    String base = '[$emoji ${level.name.toUpperCase()}] '
        '[${timestamp.toIso8601String()}] $message';
    
    if (exception != null) base += '\n  Exception: $exception';
    if (context != null) base += '\n  Context: $context';
    
    return base;
  }
}

class Logger {
  static Logger? _instance;
  
  final List<LogEntry> _logs = [];
  LogLevel _minLevel = LogLevel.debug;
  final List<void Function(LogEntry)> _handlers = [];
  
  Logger._();
  
  static Logger get instance {
    _instance ??= Logger._();
    return _instance!;
  }
  
  void setMinLevel(LogLevel level) => _minLevel = level;
  
  void addHandler(void Function(LogEntry) handler) {
    _handlers.add(handler);
  }
  
  void _log(LogLevel level, String message, {
    Map<String, dynamic>? context,
    Exception? exception,
  }) {
    if (level.index < _minLevel.index) return;
    
    LogEntry entry = LogEntry(
      level: level,
      message: message,
      context: context,
      exception: exception,
    );
    
    _logs.add(entry);
    for (var handler in _handlers) {
      handler(entry);
    }
  }
  
  void debug(String msg, {Map<String, dynamic>? context}) =>
      _log(LogLevel.debug, msg, context: context);
  
  void info(String msg, {Map<String, dynamic>? context}) =>
      _log(LogLevel.info, msg, context: context);
  
  void warning(String msg, {Map<String, dynamic>? context}) =>
      _log(LogLevel.warning, msg, context: context);
  
  void error(String msg, {Exception? exception, Map<String, dynamic>? context}) =>
      _log(LogLevel.error, msg, exception: exception, context: context);
  
  void critical(String msg, {Exception? exception, Map<String, dynamic>? context}) =>
      _log(LogLevel.critical, msg, exception: exception, context: context);
  
  List<LogEntry> getLogs({LogLevel? minLevel}) {
    if (minLevel == null) return List.unmodifiable(_logs);
    return _logs.where((l) => l.level.index >= minLevel.index).toList();
  }
  
  void clearLogs() => _logs.clear();
}

void main() {
  Logger logger = Logger.instance;
  
  // เพิ่ม console handler
  logger.addHandler((entry) => print(entry));
  
  logger.debug('เริ่มต้น Application');
  logger.info('User logged in', context: {'userId': 'alice', 'ip': '192.168.1.1'});
  logger.warning('Rate limit approaching', context: {'requests': 95, 'limit': 100});
  
  try {
    throw Exception('Database connection failed');
  } catch (e) {
    logger.error('Database error', exception: e as Exception, context: {'host': 'db.example.com'});
  }
  
  logger.critical('System overload!', context: {'cpu': '99%', 'memory': '98%'});
  
  print('\n--- Error logs only ---');
  for (var log in logger.getLogs(minLevel: LogLevel.error)) {
    print(log);
  }
}
```

### สรุป Part 08

```
✅ ขั้นตอนที่ 211: Exception พื้นฐาน
✅ ขั้นตอนที่ 212: Exception Hierarchy
✅ ขั้นตอนที่ 213: Custom Exceptions
✅ ขั้นตอนที่ 214: Result Pattern
✅ ขั้นตอนที่ 215: Async Error Handling
✅ ขั้นตอนที่ 216: Defensive Programming
✅ ขั้นตอนที่ 217-220: โปรเจกต์ - Form Validation
✅ ขั้นตอนที่ 221-240: Error Logging System
```

---

**← [Part 07 - Inheritance และ Polymorphism](part-07-inheritance-polymorphism.md)**

**ต่อไป: [Part 09 - Flutter Widget เบื้องต้น →](part-09-flutter-widgets-basics.md)**
