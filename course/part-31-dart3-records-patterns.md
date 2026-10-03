# Part 31: Dart 3.0 - Records, Patterns & Sealed Classes
## ขั้นตอนที่ 1121-1160

---

## 🎯 เป้าหมายของ Part นี้

- Records (tuple-like value types)
- Pattern Matching
- Sealed Classes
- Switch expressions
- Exhaustiveness checking

---

## ขั้นตอนที่ 1121: Records

```dart
// Records คือ anonymous immutable aggregate types

// ─── Basic Records ───
void main() {
  // Record ด้วย positional fields
  (String, int) person = ('Alice', 30);
  print(person.$1);  // Alice
  print(person.$2);  // 30

  // Record ด้วย named fields
  ({String name, int age}) namedPerson = (name: 'Bob', age: 25);
  print(namedPerson.name);  // Bob
  print(namedPerson.age);   // 25

  // Mixed
  (String, {int age, String city}) mixed = ('Charlie', age: 35, city: 'Bangkok');
  print(mixed.$1);        // Charlie
  print(mixed.age);       // 35
  print(mixed.city);      // Bangkok

  // Destructuring
  var (name, age) = person;
  print('$name is $age years old');

  var (:name2, :age2) = (name2: 'Dave', age2: 40);
  print('$name2 is $age2');

  // Records เป็น value types (equality by structure)
  (int, String) r1 = (1, 'hello');
  (int, String) r2 = (1, 'hello');
  print(r1 == r2);  // true ✅

  // Return multiple values จาก function
  (double lat, double lng) getCoordinates() => (13.7563, 100.5018);
  var (lat, lng) = getCoordinates();
  print('Bangkok: $lat, $lng');
}

// ─── Records ใน Collections ───
List<(String, double)> getPrices() {
  return [
    ('Apple', 29.99),
    ('Banana', 9.99),
    ('Cherry', 49.99),
  ];
}

void processPrices() {
  List<(String, double)> prices = getPrices();

  // Iterate with destructuring
  for (var (name, price) in prices) {
    print('$name: ฿$price');
  }

  // Sort by price
  prices.sort((a, b) => a.$2.compareTo(b.$2));
  print(prices.map((p) => p.$1).toList());  // sorted names

  // Filter expensive items
  List<(String, double)> expensive = prices.where((p) => p.$2 > 20).toList();
  print('Expensive: ${expensive.map((p) => p.$1).join(', ')}');
}

// ─── Records ใน Real App ───
({String token, String refreshToken, DateTime expiresAt}) parseAuthResponse(
  Map<String, dynamic> json,
) {
  return (
    token: json['access_token'],
    refreshToken: json['refresh_token'],
    expiresAt: DateTime.parse(json['expires_at']),
  );
}

// แทนที่ Map<String, dynamic> ที่ไม่ type-safe
({double min, double max, double avg}) calculateStats(List<double> values) {
  double sum = values.reduce((a, b) => a + b);
  return (
    min: values.reduce((a, b) => a < b ? a : b),
    max: values.reduce((a, b) => a > b ? a : b),
    avg: sum / values.length,
  );
}
```

---

## ขั้นตอนที่ 1122: Pattern Matching

```dart
// Patterns คือ syntax สำหรับ destructure และ match values

void main() {
  // ─── Variable Pattern ───
  List<int> nums = [1, 2, 3];
  var [a, b, c] = nums;
  print('$a $b $c');  // 1 2 3

  // ─── List Pattern ───
  List<int> list = [1, 2, 3, 4, 5];
  if (list case [int first, ...List<int> rest]) {
    print('First: $first, Rest: $rest');  // First: 1, Rest: [2, 3, 4, 5]
  }

  // ─── Map Pattern ───
  Map<String, dynamic> json = {'name': 'Alice', 'age': 30};
  if (json case {'name': String name, 'age': int age}) {
    print('$name is $age');  // Alice is 30
  }

  // ─── Object Pattern ───
  Object shape = Circle(radius: 5);
  if (shape case Circle(radius: double r)) {
    print('Circle with radius $r');
  }

  // ─── Logical OR pattern ───
  int x = 3;
  if (x case 1 || 2 || 3) {
    print('x is 1, 2, or 3');
  }

  // ─── Guard clause (when) ───
  List<int> numbers = [1, -2, 3, -4, 5];
  for (int n in numbers) {
    switch (n) {
      case int positive when positive > 0:
        print('Positive: $positive');
      case int negative when negative < 0:
        print('Negative: $negative');
      case _:
        print('Zero');
    }
  }
}

// ─── Pattern ใน Switch Expression ───
String describeShape(Object shape) => switch (shape) {
  Circle(radius: double r) => 'Circle r=$r',
  Rectangle(width: double w, height: double h) => 'Rectangle ${w}x${h}',
  Triangle(base: double b, height: double h) => 'Triangle base=$b h=$h',
  _ => 'Unknown shape',
};

double area(Object shape) => switch (shape) {
  Circle(radius: double r) => 3.14159 * r * r,
  Rectangle(width: double w, height: double h) => w * h,
  Triangle(base: double b, height: double h) => 0.5 * b * h,
  _ => throw ArgumentError('Unknown shape'),
};

class Circle {
  final double radius;
  Circle({required this.radius});
}

class Rectangle {
  final double width, height;
  Rectangle({required this.width, required this.height});
}

class Triangle {
  final double base, height;
  Triangle({required this.base, required this.height});
}
```

---

## ขั้นตอนที่ 1123: Sealed Classes

```dart
// Sealed classes บังคับ exhaustiveness ใน switch

// ─── Result Type กับ Sealed Class ───
sealed class Result<T> {
  const Result();
}

class Success<T> extends Result<T> {
  final T data;
  const Success(this.data);
}

class Failure<T> extends Result<T> {
  final String message;
  final Object? error;
  const Failure(this.message, {this.error});
}

class Loading<T> extends Result<T> {
  const Loading();
}

// Switch ที่ compiler บังคับให้ handle ทุก case
String resultToString<T>(Result<T> result) => switch (result) {
  Success(data: T data) => 'Success: $data',
  Failure(message: String msg) => 'Error: $msg',
  Loading() => 'Loading...',
  // ถ้าลืม case ใดจะ compile error ✅
};

// ─── State Machine กับ Sealed Classes ───
sealed class OrderStatus {
  const OrderStatus();
}

class Pending extends OrderStatus {
  final DateTime createdAt;
  const Pending(this.createdAt);
}

class Processing extends OrderStatus {
  final String trackingId;
  const Processing(this.trackingId);
}

class Shipped extends OrderStatus {
  final String courier;
  final DateTime estimatedDelivery;
  const Shipped({required this.courier, required this.estimatedDelivery});
}

class Delivered extends OrderStatus {
  final DateTime deliveredAt;
  const Delivered(this.deliveredAt);
}

class Cancelled extends OrderStatus {
  final String reason;
  const Cancelled(this.reason);
}

// Exhaustive switch
String getStatusText(OrderStatus status) => switch (status) {
  Pending(createdAt: DateTime t) => 'รอดำเนินการ (สั่งเมื่อ ${t.day}/${t.month})',
  Processing(trackingId: String id) => 'กำลังดำเนินการ (เลข: $id)',
  Shipped(courier: String c, estimatedDelivery: DateTime d) =>
    'กำลังจัดส่งโดย $c (คาดว่าจะถึง ${d.day}/${d.month})',
  Delivered(deliveredAt: DateTime t) => 'ส่งแล้วเมื่อ ${t.day}/${t.month}/${t.year}',
  Cancelled(reason: String r) => 'ยกเลิกแล้ว: $r',
};

bool canCancel(OrderStatus status) => switch (status) {
  Pending() || Processing() => true,
  Shipped() || Delivered() || Cancelled() => false,
};

// ─── HTTP Response sealed class ───
sealed class HttpResponse<T> {
  const HttpResponse();
}

class HttpSuccess<T> extends HttpResponse<T> {
  final T body;
  final int statusCode;
  const HttpSuccess(this.body, {this.statusCode = 200});
}

class HttpError<T> extends HttpResponse<T> {
  final int statusCode;
  final String message;
  const HttpError({required this.statusCode, required this.message});
}

class HttpNetworkError<T> extends HttpResponse<T> {
  final String message;
  const HttpNetworkError(this.message);
}

T handleResponse<T>(HttpResponse<T> response) => switch (response) {
  HttpSuccess(body: T data) => data,
  HttpError(statusCode: int code, message: String msg) =>
    throw Exception('HTTP $code: $msg'),
  HttpNetworkError(message: String msg) =>
    throw Exception('Network error: $msg'),
};
```

---

## ขั้นตอนที่ 1124: Switch Expressions กับ Flutter

```dart
import 'package:flutter/material.dart';

// ─── Widget selection ด้วย switch expression ───
class StatusIcon extends StatelessWidget {
  final OrderStatus status;
  const StatusIcon({super.key, required this.status});

  @override
  Widget build(BuildContext context) {
    return switch (status) {
      Pending() => const Icon(Icons.hourglass_empty, color: Colors.orange),
      Processing() => const Icon(Icons.settings, color: Colors.blue),
      Shipped() => const Icon(Icons.local_shipping, color: Colors.purple),
      Delivered() => const Icon(Icons.check_circle, color: Colors.green),
      Cancelled() => const Icon(Icons.cancel, color: Colors.red),
    };
  }
}

// ─── Color scheme ด้วย switch expression ───
Color getStatusColor(OrderStatus status) => switch (status) {
  Pending() => Colors.orange,
  Processing() => Colors.blue,
  Shipped() => Colors.purple,
  Delivered() => Colors.green,
  Cancelled() => Colors.red,
};

// ─── Pattern matching กับ JSON parsing ───
List<Map<String, dynamic>> parseApiResponse(Map<String, dynamic> response) {
  return switch (response) {
    {'data': List items, 'total': int _} =>
      items.cast<Map<String, dynamic>>(),
    {'items': List items} =>
      items.cast<Map<String, dynamic>>(),
    {'error': String msg} =>
      throw Exception('API Error: $msg'),
    _ => throw FormatException('Unexpected response format'),
  };
}

// ─── Real-world example: Form validation ด้วย patterns ───
sealed class ValidationResult {
  const ValidationResult();
}
class Valid extends ValidationResult {
  const Valid();
}
class Invalid extends ValidationResult {
  final String message;
  const Invalid(this.message);
}

ValidationResult validateEmail(String email) => switch (email) {
  '' => const Invalid('กรุณากรอก email'),
  String e when !e.contains('@') => const Invalid('email ต้องมี @'),
  String e when e.length < 5 => const Invalid('email สั้นเกินไป'),
  _ => const Valid(),
};

ValidationResult validatePassword(String password) => switch (password) {
  '' => const Invalid('กรุณากรอก password'),
  String p when p.length < 8 => const Invalid('password ต้องมีอย่างน้อย 8 ตัวอักษร'),
  String p when !p.contains(RegExp(r'[A-Z]')) => const Invalid('ต้องมีตัวพิมพ์ใหญ่'),
  String p when !p.contains(RegExp(r'[0-9]')) => const Invalid('ต้องมีตัวเลข'),
  _ => const Valid(),
};

// Widget แสดง validation
class ValidatedTextField extends StatefulWidget {
  final String label;
  final ValidationResult Function(String) validator;

  const ValidatedTextField({
    super.key,
    required this.label,
    required this.validator,
  });

  @override
  State<ValidatedTextField> createState() => _ValidatedTextFieldState();
}

class _ValidatedTextFieldState extends State<ValidatedTextField> {
  ValidationResult _result = const Valid();

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        TextField(
          decoration: InputDecoration(
            labelText: widget.label,
            border: const OutlineInputBorder(),
            suffixIcon: switch (_result) {
              Valid() => const Icon(Icons.check_circle, color: Colors.green),
              Invalid() => const Icon(Icons.error, color: Colors.red),
            },
          ),
          onChanged: (v) => setState(() => _result = widget.validator(v)),
        ),
        if (_result case Invalid(message: String msg))
          Padding(
            padding: const EdgeInsets.only(top: 4, left: 12),
            child: Text(msg, style: const TextStyle(color: Colors.red, fontSize: 12)),
          ),
      ],
    );
  }
}
```

---

**← [Part 30 - World-Class App](part-30-world-class.md)**

**ต่อไป: [Part 32 - Generics & Type System Advanced →](part-32-generics-advanced.md)**
