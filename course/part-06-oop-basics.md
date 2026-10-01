# Part 06: Object-Oriented Programming เบื้องต้น
## ขั้นตอนที่ 141-180

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจหลักการ OOP (Encapsulation, Abstraction)
- ประกาศและใช้ Class และ Object
- เข้าใจ Constructors ทุกประเภท
- ใช้ Getters และ Setters
- เข้าใจ Static members
- สร้าง Immutable Classes
- ใช้ Factory constructors

---

## ขั้นตอนที่ 141: Class พื้นฐาน

```dart
// ─────────────── Class พื้นฐาน ───────────────
class Person {
  // Instance variables (fields)
  String name;
  int age;
  String? email;
  
  // Constructor
  Person(this.name, this.age, {this.email});
  
  // Instance method
  void greet() {
    print('สวัสดี ฉันชื่อ $name อายุ $age ปี');
  }
  
  String introduce() {
    return 'ชื่อ: $name, อายุ: $age${email != null ? ", Email: $email" : ""}';
  }
  
  // toString override
  @override
  String toString() => 'Person(name: $name, age: $age)';
}

void main() {
  // สร้าง Object
  Person alice = Person('Alice', 25, email: 'alice@example.com');
  Person bob = Person('Bob', 30);
  
  // เรียกใช้ method
  alice.greet();
  bob.greet();
  
  // เข้าถึง properties
  print(alice.name);
  print(alice.age);
  print(alice.email);
  
  // แก้ไข properties
  alice.age = 26;
  alice.email = 'alice.new@example.com';
  
  print(alice.introduce());
  print(bob);  // เรียก toString()
  
  // Type checking
  print(alice is Person);   // true
  print(alice.runtimeType); // Person
}
```

---

## ขั้นตอนที่ 142: Constructors ทุกประเภท

```dart
class Temperature {
  final double celsius;
  
  // ─────────────── Default Constructor ───────────────
  Temperature(this.celsius);
  
  // ─────────────── Named Constructor ───────────────
  Temperature.fromFahrenheit(double fahrenheit)
      : celsius = (fahrenheit - 32) * 5 / 9;
  
  Temperature.fromKelvin(double kelvin)
      : celsius = kelvin - 273.15;
  
  Temperature.freezing() : celsius = 0;
  Temperature.boiling() : celsius = 100;
  Temperature.bodyTemp() : celsius = 37;
  
  // ─────────────── Factory Constructor ───────────────
  factory Temperature.fromJson(Map<String, dynamic> json) {
    String unit = json['unit'] as String;
    double value = json['value'] as double;
    
    return switch (unit) {
      'C' => Temperature(value),
      'F' => Temperature.fromFahrenheit(value),
      'K' => Temperature.fromKelvin(value),
      _ => throw ArgumentError('Unknown unit: $unit'),
    };
  }
  
  // Computed properties
  double get fahrenheit => celsius * 9 / 5 + 32;
  double get kelvin => celsius + 273.15;
  
  @override
  String toString() => '${celsius.toStringAsFixed(2)}°C';
}

void main() {
  // Default
  Temperature t1 = Temperature(100);
  
  // Named constructors
  Temperature t2 = Temperature.fromFahrenheit(212);
  Temperature t3 = Temperature.fromKelvin(373.15);
  Temperature t4 = Temperature.freezing();
  Temperature t5 = Temperature.boiling();
  
  print('t1: $t1 = ${t1.fahrenheit.toStringAsFixed(2)}°F = ${t1.kelvin.toStringAsFixed(2)}K');
  print('t2: $t2');
  print('t3: $t3');
  print('Freezing: $t4');
  print('Boiling: $t5');
  
  // Factory constructor
  Temperature t6 = Temperature.fromJson({'unit': 'F', 'value': 98.6});
  print('Body temp: $t6');
}
```

---

## ขั้นตอนที่ 143: Encapsulation - Private Members

```dart
class BankAccount {
  // ─────────────── Private fields (เริ่มด้วย _) ───────────────
  String _accountNumber;
  String _ownerName;
  double _balance;
  List<String> _transactions = [];
  
  // Constructor
  BankAccount({
    required String accountNumber,
    required String ownerName,
    double initialBalance = 0,
  })  : _accountNumber = accountNumber,
        _ownerName = ownerName,
        _balance = initialBalance;
  
  // ─────────────── Getters ───────────────
  String get accountNumber => _accountNumber;
  String get ownerName => _ownerName;
  double get balance => _balance;
  List<String> get transactions => List.unmodifiable(_transactions);
  
  // ─────────────── Setters ───────────────
  set ownerName(String name) {
    if (name.trim().isEmpty) throw ArgumentError('ชื่อต้องไม่ว่าง');
    _ownerName = name.trim();
  }
  
  // ─────────────── Methods ───────────────
  void deposit(double amount) {
    if (amount <= 0) throw ArgumentError('จำนวนเงินต้องมากกว่า 0');
    _balance += amount;
    _addTransaction('ฝาก: +฿${amount.toStringAsFixed(2)}');
    print('ฝากเงิน ฿${amount.toStringAsFixed(2)} สำเร็จ | ยอดคงเหลือ: ฿${_balance.toStringAsFixed(2)}');
  }
  
  void withdraw(double amount) {
    if (amount <= 0) throw ArgumentError('จำนวนเงินต้องมากกว่า 0');
    if (amount > _balance) throw StateError('เงินในบัญชีไม่เพียงพอ');
    _balance -= amount;
    _addTransaction('ถอน: -฿${amount.toStringAsFixed(2)}');
    print('ถอนเงิน ฿${amount.toStringAsFixed(2)} สำเร็จ | ยอดคงเหลือ: ฿${_balance.toStringAsFixed(2)}');
  }
  
  void transfer(BankAccount target, double amount) {
    withdraw(amount);
    target.deposit(amount);
    print('โอนเงิน ฿${amount.toStringAsFixed(2)} ไปบัญชี ${target._accountNumber} สำเร็จ');
  }
  
  void _addTransaction(String description) {
    _transactions.add('${DateTime.now().toIso8601String()} $description');
  }
  
  void printStatement() {
    print('=== Statement: $_accountNumber ===');
    print('เจ้าของ: $_ownerName');
    print('ยอดคงเหลือ: ฿${_balance.toStringAsFixed(2)}');
    print('รายการ:');
    for (var t in _transactions) {
      print('  $t');
    }
  }
  
  @override
  String toString() => 'Account($_accountNumber, ฿${_balance.toStringAsFixed(2)})';
}

void main() {
  BankAccount alice = BankAccount(
    accountNumber: '1234567890',
    ownerName: 'Alice Smith',
    initialBalance: 1000.0,
  );
  
  BankAccount bob = BankAccount(
    accountNumber: '0987654321',
    ownerName: 'Bob Jones',
  );
  
  alice.deposit(5000);
  alice.withdraw(2000);
  alice.transfer(bob, 1500);
  
  print('\n');
  alice.printStatement();
  print('\n');
  bob.printStatement();
}
```

---

## ขั้นตอนที่ 144: Getters และ Setters ขั้นสูง

```dart
class Circle {
  double _radius;
  
  Circle(this._radius) {
    _validate(_radius);
  }
  
  static void _validate(double radius) {
    if (radius <= 0) throw ArgumentError('รัศมีต้องมากกว่า 0');
  }
  
  // Getter/Setter สำหรับ radius
  double get radius => _radius;
  set radius(double value) {
    _validate(value);
    _radius = value;
  }
  
  // Computed Getters
  double get diameter => _radius * 2;
  double get area => 3.14159 * _radius * _radius;
  double get circumference => 2 * 3.14159 * _radius;
  
  // Setter จาก diameter
  set diameter(double d) {
    radius = d / 2;  // ใช้ radius setter เพื่อ validate
  }
  
  @override
  String toString() =>
      'Circle(r=${_radius.toStringAsFixed(2)}, '
      'area=${area.toStringAsFixed(2)})';
}

class Temperature2 {
  double _celsius = 0;
  
  // Getter/Setter สำหรับ celsius
  double get celsius => _celsius;
  set celsius(double value) {
    if (value < -273.15) throw ArgumentError('ต่ำกว่า Absolute Zero ไม่ได้');
    _celsius = value;
  }
  
  // Derived getters/setters
  double get fahrenheit => _celsius * 9 / 5 + 32;
  set fahrenheit(double f) => celsius = (f - 32) * 5 / 9;
  
  double get kelvin => _celsius + 273.15;
  set kelvin(double k) => celsius = k - 273.15;
}

void main() {
  Circle c = Circle(5);
  print(c);
  
  c.radius = 10;
  print('Diameter: ${c.diameter}');
  print('Area: ${c.area.toStringAsFixed(2)}');
  
  c.diameter = 6;  // ตั้ง diameter = 6 ทำให้ radius = 3
  print(c);
  
  Temperature2 t = Temperature2();
  t.celsius = 100;
  print('100°C = ${t.fahrenheit}°F = ${t.kelvin}K');
  
  t.fahrenheit = 32;
  print('32°F = ${t.celsius}°C');
}
```

---

## ขั้นตอนที่ 145: Static Members

```dart
class MathUtils {
  // ─────────────── Static constants ───────────────
  static const double pi = 3.14159265358979;
  static const double e = 2.71828182845905;
  static const double goldenRatio = 1.61803398874989;
  
  // ─────────────── Static variables ───────────────
  static int _callCount = 0;
  
  // Static getter
  static int get callCount => _callCount;
  
  // ─────────────── Static methods ───────────────
  static double circleArea(double r) {
    _callCount++;
    return pi * r * r;
  }
  
  static double factorial(int n) {
    _callCount++;
    if (n <= 1) return 1;
    return n * factorial(n - 1);
  }
  
  static bool isPrime(int n) {
    _callCount++;
    if (n < 2) return false;
    for (int i = 2; i * i <= n; i++) {
      if (n % i == 0) return false;
    }
    return true;
  }
  
  // ─────────────── Private constructor (Utility class) ───────────────
  MathUtils._();  // ป้องกันการ instantiate
}

class Counter {
  // ─────────────── Singleton Pattern ───────────────
  static Counter? _instance;
  
  int _count = 0;
  
  Counter._();  // private constructor
  
  // Static factory method (Singleton)
  static Counter get instance {
    _instance ??= Counter._();
    return _instance!;
  }
  
  void increment() => _count++;
  void decrement() => _count--;
  int get count => _count;
  
  static void reset() {
    _instance = null;  // สร้าง instance ใหม่ครั้งหน้า
  }
}

void main() {
  // Static usage - ไม่ต้อง instantiate
  print('Pi: ${MathUtils.pi}');
  print('Circle area (r=5): ${MathUtils.circleArea(5).toStringAsFixed(4)}');
  print('5! = ${MathUtils.factorial(5)}');
  print('17 is prime: ${MathUtils.isPrime(17)}');
  print('Call count: ${MathUtils.callCount}');
  
  // MathUtils()  // Error! constructor เป็น private
  
  // Singleton
  Counter c1 = Counter.instance;
  Counter c2 = Counter.instance;
  
  c1.increment();
  c1.increment();
  c2.increment();
  
  print('Same instance: ${identical(c1, c2)}');  // true
  print('Count: ${c1.count}');  // 3 (shared state)
}
```

---

## ขั้นตอนที่ 146: Immutable Classes

```dart
// ─────────────── Immutable Class ───────────────
// ทุก field เป็น final และ constructor เป็น const
class Point {
  final double x;
  final double y;
  
  const Point(this.x, this.y);
  const Point.origin() : x = 0, y = 0;
  
  // Methods คืน object ใหม่ (ไม่แก้ไข)
  Point translate(double dx, double dy) => Point(x + dx, y + dy);
  Point scale(double factor) => Point(x * factor, y * factor);
  
  double distanceTo(Point other) {
    double dx = x - other.x;
    double dy = y - other.y;
    return (dx * dx + dy * dy).sqrt(); // ต้องใช้ extension จาก part ก่อน
  }
  
  double get magnitude => (x * x + y * y).sqrt2();
  
  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Point && other.x == x && other.y == y;
  }
  
  @override
  int get hashCode => Object.hash(x, y);
  
  @override
  String toString() => 'Point($x, $y)';
}

extension on double {
  double sqrt() {
    double result = this;
    for (int i = 0; i < 100; i++) {
      result = (result + this / result) / 2;
    }
    return result;
  }
  double sqrt2() {
    if (this < 0) return double.nan;
    double result = this;
    for (int i = 0; i < 100; i++) {
      result = (result + this / result) / 2;
    }
    return result;
  }
}

// Value Object Pattern
class Money {
  final double amount;
  final String currency;
  
  const Money(this.amount, this.currency);
  const Money.zero(String currency) : amount = 0, currency = currency;
  
  Money operator +(Money other) {
    if (currency != other.currency) {
      throw ArgumentError('Cannot add different currencies');
    }
    return Money(amount + other.amount, currency);
  }
  
  Money operator -(Money other) {
    if (currency != other.currency) {
      throw ArgumentError('Cannot subtract different currencies');
    }
    return Money(amount - other.amount, currency);
  }
  
  Money operator *(double factor) => Money(amount * factor, currency);
  
  bool operator <(Money other) => amount < other.amount;
  bool operator >(Money other) => amount > other.amount;
  
  @override
  bool operator ==(Object other) {
    return other is Money && other.amount == amount && other.currency == currency;
  }
  
  @override
  int get hashCode => Object.hash(amount, currency);
  
  @override
  String toString() => '${amount.toStringAsFixed(2)} $currency';
}

void main() {
  // const objects - compile-time constant
  const Point p1 = Point(3, 4);
  const Point origin = Point.origin();
  
  Point p2 = p1.translate(1, 1);  // คืน object ใหม่
  
  print(p1);        // Point(3.0, 4.0)
  print(p2);        // Point(4.0, 5.0)
  print(p1 == Point(3, 4));  // true (value equality)
  
  // Money
  Money price1 = Money(100, 'THB');
  Money price2 = Money(50, 'THB');
  
  print(price1 + price2);  // 150.00 THB
  print(price1 * 1.07);    // 107.00 THB (VAT)
  print(price1 > price2);  // true
}
```

---

## ขั้นตอนที่ 147: Operator Overloading

```dart
class Vector2D {
  final double x;
  final double y;
  
  const Vector2D(this.x, this.y);
  const Vector2D.zero() : x = 0, y = 0;
  
  // ─────────────── Arithmetic Operators ───────────────
  Vector2D operator +(Vector2D other) => Vector2D(x + other.x, y + other.y);
  Vector2D operator -(Vector2D other) => Vector2D(x - other.x, y - other.y);
  Vector2D operator *(double scalar) => Vector2D(x * scalar, y * scalar);
  Vector2D operator /(double scalar) => Vector2D(x / scalar, y / scalar);
  Vector2D operator -() => Vector2D(-x, -y);  // Unary minus
  
  // ─────────────── Comparison Operators ───────────────
  bool operator ==(Object other) {
    return other is Vector2D && other.x == x && other.y == y;
  }
  
  // ─────────────── Index Operator ───────────────
  double operator [](int index) {
    if (index == 0) return x;
    if (index == 1) return y;
    throw RangeError.index(index, this);
  }
  
  // ─────────────── Properties ───────────────
  double get magnitude => (x * x + y * y).sqrt();
  double get angle => (y / x).atan();  // simplified
  
  Vector2D get normalized {
    double mag = magnitude;
    if (mag == 0) return Vector2D.zero();
    return Vector2D(x / mag, y / mag);
  }
  
  // ─────────────── Methods ───────────────
  double dot(Vector2D other) => x * other.x + y * other.y;
  
  @override
  int get hashCode => Object.hash(x, y);
  
  @override
  String toString() => 'Vector2D($x, $y)';
}

extension on double {
  double sqrt() {
    if (this <= 0) return 0;
    double r = this;
    for (int i = 0; i < 50; i++) r = (r + this / r) / 2;
    return r;
  }
  double atan() {
    // Simplified
    return this;
  }
}

void main() {
  Vector2D v1 = Vector2D(3, 4);
  Vector2D v2 = Vector2D(1, 2);
  
  print(v1 + v2);          // Vector2D(4.0, 6.0)
  print(v1 - v2);          // Vector2D(2.0, 2.0)
  print(v1 * 2);           // Vector2D(6.0, 8.0)
  print(-v1);              // Vector2D(-3.0, -4.0)
  print(v1[0]);            // 3.0 (index operator)
  print('|v1| = ${v1.magnitude}'); // 5.0
  print('dot: ${v1.dot(v2)}');     // 11.0
}
```

---

## ขั้นตอนที่ 148: Abstract Classes

```dart
// ─────────────── Abstract Class ───────────────
abstract class Shape {
  // Abstract properties (ต้อง implement ใน subclass)
  double get area;
  double get perimeter;
  String get name;
  
  // Concrete method (ใช้ร่วมกันได้)
  void printInfo() {
    print('Shape: $name');
    print('Area: ${area.toStringAsFixed(4)}');
    print('Perimeter: ${perimeter.toStringAsFixed(4)}');
  }
  
  // Abstract method
  bool contains(double x, double y);
  
  // Template method pattern
  String describe() {
    return '$name with area ${area.toStringAsFixed(2)}';
  }
}

// ─────────────── Concrete Implementations ───────────────
class Circle2 extends Shape {
  final double radius;
  
  Circle2(this.radius);
  
  @override
  String get name => 'Circle';
  
  @override
  double get area => 3.14159 * radius * radius;
  
  @override
  double get perimeter => 2 * 3.14159 * radius;
  
  @override
  bool contains(double x, double y) {
    return x * x + y * y <= radius * radius;
  }
}

class Rectangle extends Shape {
  final double width;
  final double height;
  
  Rectangle(this.width, this.height);
  
  @override
  String get name => 'Rectangle';
  
  @override
  double get area => width * height;
  
  @override
  double get perimeter => 2 * (width + height);
  
  @override
  bool contains(double x, double y) {
    return x >= 0 && x <= width && y >= 0 && y <= height;
  }
  
  bool get isSquare => width == height;
}

class Triangle extends Shape {
  final double a;
  final double b;
  final double c;
  
  Triangle(this.a, this.b, this.c) {
    if (a + b <= c || a + c <= b || b + c <= a) {
      throw ArgumentError('Invalid triangle sides');
    }
  }
  
  @override
  String get name => 'Triangle';
  
  @override
  double get perimeter => a + b + c;
  
  @override
  double get area {
    double s = perimeter / 2;  // semi-perimeter
    return (s * (s - a) * (s - b) * (s - c)).sqrt();
  }
  
  @override
  bool contains(double x, double y) => true; // simplified
}

extension on double {
  double sqrt() {
    if (this <= 0) return 0;
    double r = this;
    for (int i = 0; i < 50; i++) r = (r + this / r) / 2;
    return r;
  }
}

void main() {
  List<Shape> shapes = [
    Circle2(5),
    Rectangle(4, 6),
    Triangle(3, 4, 5),
    Rectangle(4, 4),  // square
  ];
  
  for (Shape shape in shapes) {
    shape.printInfo();
    
    if (shape is Rectangle && shape.isSquare) {
      print('  → เป็นสี่เหลี่ยมจัตุรัส!');
    }
    print('---');
  }
  
  // คำนวณพื้นที่รวม
  double totalArea = shapes.fold(0, (sum, s) => sum + s.area);
  print('พื้นที่รวม: ${totalArea.toStringAsFixed(2)}');
  
  // เรียงตามพื้นที่
  shapes.sort((a, b) => a.area.compareTo(b.area));
  print('เรียงตามพื้นที่:');
  for (var s in shapes) {
    print('  ${s.name}: ${s.area.toStringAsFixed(2)}');
  }
}
```

---

## ขั้นตอนที่ 149: Interfaces (ใน Dart ทุก Class เป็น Interface)

```dart
// ─────────────── Implicit Interface ───────────────
// ใน Dart ทุก class สามารถใช้เป็น interface ได้
class Flyable {
  void fly() => print('กำลังบิน');
  void land() => print('ลงจอด');
}

class Swimmable {
  void swim() => print('กำลังว่ายน้ำ');
}

// ─────────────── implements (Interface) ───────────────
// ใช้ implements เมื่อต้องการ implement interface
class Duck implements Flyable, Swimmable {
  String name;
  Duck(this.name);
  
  @override
  void fly() => print('$name กำลังบิน 🦆');
  
  @override
  void land() => print('$name ลงจอด');
  
  @override
  void swim() => print('$name กำลังว่ายน้ำ 🏊');
  
  void quack() => print('$name: เควก เควก!');
}

// ─────────────── Abstract Interface ───────────────
abstract interface class Serializable {
  Map<String, dynamic> toJson();
  
  // Static method (Dart 3.0+)
  static T fromJson<T>(Map<String, dynamic> json) {
    throw UnimplementedError();
  }
}

abstract interface class Comparable2<T> {
  int compareTo(T other);
  
  bool operator <(T other) => compareTo(other) < 0;
  bool operator >(T other) => compareTo(other) > 0;
  bool operator <=(T other) => compareTo(other) <= 0;
  bool operator >=(T other) => compareTo(other) >= 0;
}

class Student implements Serializable, Comparable2<Student> {
  final String name;
  final double gpa;
  
  Student(this.name, this.gpa);
  
  @override
  Map<String, dynamic> toJson() => {'name': name, 'gpa': gpa};
  
  @override
  int compareTo(Student other) => gpa.compareTo(other.gpa);
  
  @override
  String toString() => 'Student($name, $gpa)';
}

void main() {
  Duck donald = Duck('Donald');
  donald.fly();
  donald.swim();
  donald.quack();
  
  // Duck เป็นทั้ง Flyable และ Swimmable
  Flyable flyer = donald;
  Swimmable swimmer = donald;
  
  flyer.fly();
  swimmer.swim();
  
  // Student comparable
  List<Student> students = [
    Student('Charlie', 3.2),
    Student('Alice', 3.8),
    Student('Bob', 3.5),
  ];
  
  students.sort((a, b) => a.compareTo(b));
  for (var s in students) {
    print(s);
  }
  
  // Serializable
  var json = students[0].toJson();
  print('JSON: $json');
}
```

---

## ขั้นตอนที่ 150: Mixins

```dart
// ─────────────── Mixin ───────────────
mixin Logging {
  void log(String message) {
    print('[LOG ${runtimeType}] $message');
  }
  
  void logError(String message) {
    print('[ERROR ${runtimeType}] ❌ $message');
  }
}

mixin Validatable {
  bool validate() => true;
  
  void assertValid() {
    if (!validate()) {
      throw StateError('${runtimeType} is invalid');
    }
  }
}

mixin Cacheable<T> {
  final Map<String, T> _cache = {};
  
  T? getFromCache(String key) => _cache[key];
  void addToCache(String key, T value) => _cache[key] = value;
  void clearCache() => _cache.clear();
}

mixin Timestamped {
  late final DateTime createdAt = DateTime.now();
  DateTime? updatedAt;
  
  void markUpdated() => updatedAt = DateTime.now();
}

// ─────────────── ใช้ Mixin ───────────────
class UserService with Logging, Validatable, Cacheable<Map<String, dynamic>>, Timestamped {
  String _endpoint;
  
  UserService(this._endpoint) {
    log('UserService created for $_endpoint');
  }
  
  Map<String, dynamic>? getUser(String id) {
    // Check cache first
    var cached = getFromCache(id);
    if (cached != null) {
      log('Cache hit for user $id');
      return cached;
    }
    
    // Simulate API call
    log('Fetching user $id from $_endpoint');
    var user = {'id': id, 'name': 'User $id'};
    addToCache(id, user);
    markUpdated();
    return user;
  }
  
  @override
  bool validate() {
    return _endpoint.isNotEmpty && _endpoint.startsWith('http');
  }
}

// ─────────────── on Constraint ───────────────
// Mixin ที่ใช้ได้กับ specific class เท่านั้น
mixin Discountable on Product2 {
  double get discount => 0;
  double get discountedPrice => price * (1 - discount);
}

class Product2 {
  final String name;
  final double price;
  Product2(this.name, this.price);
}

class SaleProduct extends Product2 with Discountable {
  final double _discount;
  SaleProduct(String name, double price, this._discount) : super(name, price);
  
  @override
  double get discount => _discount;
}

void main() {
  UserService service = UserService('https://api.example.com/users');
  service.assertValid();
  
  var user1 = service.getUser('123');
  var user1Again = service.getUser('123');  // จาก cache
  
  print(user1);
  print('Created: ${service.createdAt}');
  print('Updated: ${service.updatedAt}');
  
  SaleProduct product = SaleProduct('Laptop', 30000, 0.2);
  print('\n${product.name}');
  print('ราคาปกติ: ฿${product.price}');
  print('ส่วนลด: ${product.discount * 100}%');
  print('ราคาหลังลด: ฿${product.discountedPrice}');
}
```

---

## ขั้นตอนที่ 151-160: โปรเจกต์ - Library System

```dart
// library_system.dart

import 'dart:collection';

// ─────────────── Enums ───────────────
enum BookStatus { available, borrowed, reserved }
enum MemberType { regular, premium, student }

// ─────────────── Book Class ───────────────
class Book {
  final String isbn;
  final String title;
  final String author;
  final int year;
  final List<String> genres;
  BookStatus _status = BookStatus.available;
  String? _borrowedBy;
  DateTime? _dueDate;
  
  Book({
    required this.isbn,
    required this.title,
    required this.author,
    required this.year,
    this.genres = const [],
  });
  
  BookStatus get status => _status;
  String? get borrowedBy => _borrowedBy;
  DateTime? get dueDate => _dueDate;
  bool get isAvailable => _status == BookStatus.available;
  
  void borrow(String memberId, {int days = 14}) {
    if (!isAvailable) throw StateError('หนังสือไม่ว่าง');
    _status = BookStatus.borrowed;
    _borrowedBy = memberId;
    _dueDate = DateTime.now().add(Duration(days: days));
  }
  
  void returnBook() {
    _status = BookStatus.available;
    _borrowedBy = null;
    _dueDate = null;
  }
  
  bool get isOverdue {
    if (_dueDate == null) return false;
    return DateTime.now().isAfter(_dueDate!);
  }
  
  @override
  String toString() => '"$title" by $author (${_status.name})';
}

// ─────────────── Member Class ───────────────
class Member {
  final String id;
  final String name;
  final MemberType type;
  final List<String> _borrowedBooks = [];
  int _lateReturnCount = 0;
  
  Member({required this.id, required this.name, this.type = MemberType.regular});
  
  int get maxBooksAllowed {
    return switch (type) {
      MemberType.regular => 3,
      MemberType.premium => 10,
      MemberType.student => 5,
    };
  }
  
  int get borrowDays {
    return switch (type) {
      MemberType.regular => 14,
      MemberType.premium => 30,
      MemberType.student => 21,
    };
  }
  
  List<String> get borrowedBooks => List.unmodifiable(_borrowedBooks);
  bool get canBorrow => _borrowedBooks.length < maxBooksAllowed;
  
  void borrowBook(String isbn) {
    if (!canBorrow) throw StateError('ยืมหนังสือครบ $maxBooksAllowed เล่มแล้ว');
    _borrowedBooks.add(isbn);
  }
  
  void returnBook(String isbn, {bool late = false}) {
    if (!_borrowedBooks.remove(isbn)) throw ArgumentError('ไม่ได้ยืมหนังสือ $isbn');
    if (late) _lateReturnCount++;
  }
  
  @override
  String toString() => 'Member($id, $name, ${type.name})';
}

// ─────────────── Library Class ───────────────
class Library {
  final String name;
  final Map<String, Book> _books = {};
  final Map<String, Member> _members = {};
  final List<String> _transactionLog = [];
  
  Library(this.name);
  
  // ─── Book Management ───
  void addBook(Book book) {
    if (_books.containsKey(book.isbn)) {
      throw ArgumentError('หนังสือ ISBN ${book.isbn} มีอยู่แล้ว');
    }
    _books[book.isbn] = book;
    _log('เพิ่มหนังสือ: ${book.title}');
  }
  
  Book? findBook(String isbn) => _books[isbn];
  
  List<Book> searchByTitle(String query) {
    String q = query.toLowerCase();
    return _books.values
        .where((b) => b.title.toLowerCase().contains(q))
        .toList();
  }
  
  List<Book> searchByAuthor(String author) {
    String a = author.toLowerCase();
    return _books.values
        .where((b) => b.author.toLowerCase().contains(a))
        .toList();
  }
  
  List<Book> getAvailableBooks() {
    return _books.values
        .where((b) => b.isAvailable)
        .toList()
      ..sort((a, b) => a.title.compareTo(b.title));
  }
  
  // ─── Member Management ───
  void registerMember(Member member) {
    if (_members.containsKey(member.id)) {
      throw ArgumentError('สมาชิก ID ${member.id} มีอยู่แล้ว');
    }
    _members[member.id] = member;
    _log('ลงทะเบียนสมาชิก: ${member.name}');
  }
  
  // ─── Transaction ───
  void borrowBook(String memberId, String isbn) {
    Member? member = _members[memberId];
    Book? book = _books[isbn];
    
    if (member == null) throw ArgumentError('ไม่พบสมาชิก $memberId');
    if (book == null) throw ArgumentError('ไม่พบหนังสือ $isbn');
    if (!book.isAvailable) throw StateError('หนังสือ ${book.title} ไม่ว่าง');
    if (!member.canBorrow) throw StateError('${member.name} ยืมครบแล้ว');
    
    book.borrow(memberId, days: member.borrowDays);
    member.borrowBook(isbn);
    
    _log('ยืม: ${member.name} ยืม "${book.title}" (คืนก่อน ${book.dueDate})');
    print('✅ ${member.name} ยืม "${book.title}" สำเร็จ (คืนภายใน ${member.borrowDays} วัน)');
  }
  
  void returnBook(String memberId, String isbn) {
    Member? member = _members[memberId];
    Book? book = _books[isbn];
    
    if (member == null) throw ArgumentError('ไม่พบสมาชิก $memberId');
    if (book == null) throw ArgumentError('ไม่พบหนังสือ $isbn');
    
    bool late = book.isOverdue;
    book.returnBook();
    member.returnBook(isbn, late: late);
    
    if (late) {
      print('⚠️ ${member.name} คืน "${book.title}" ล่าช้า!');
    } else {
      print('✅ ${member.name} คืน "${book.title}" สำเร็จ');
    }
    _log('คืน: ${member.name} คืน "${book.title}"${late ? " (ล่าช้า)" : ""}');
  }
  
  void _log(String message) {
    _transactionLog.add('[${DateTime.now().toIso8601String()}] $message');
  }
  
  void printReport() {
    print('\n📚 รายงานห้องสมุด: $name');
    print('═' * 50);
    print('หนังสือทั้งหมด: ${_books.length} เล่ม');
    print('หนังสือว่าง: ${getAvailableBooks().length} เล่ม');
    print('สมาชิกทั้งหมด: ${_members.length} คน');
    print('รายการ Transaction: ${_transactionLog.length} รายการ');
    
    print('\nหนังสือที่ถูกยืม:');
    _books.values.where((b) => !b.isAvailable).forEach((b) {
      print('  📖 ${b.title} - ยืมโดย: ${_members[b.borrowedBy]?.name ?? b.borrowedBy}');
    });
  }
}

void main() {
  Library lib = Library('ห้องสมุดประชาชน');
  
  // เพิ่มหนังสือ
  lib.addBook(Book(
    isbn: '978-0-7432-7356-5',
    title: 'The Great Gatsby',
    author: 'F. Scott Fitzgerald',
    year: 1925,
    genres: ['Fiction', 'Classic'],
  ));
  lib.addBook(Book(
    isbn: '978-0-06-112008-4',
    title: 'To Kill a Mockingbird',
    author: 'Harper Lee',
    year: 1960,
    genres: ['Fiction', 'Classic'],
  ));
  lib.addBook(Book(
    isbn: '978-0-14-028329-7',
    title: 'Dart Programming',
    author: 'John Smith',
    year: 2023,
    genres: ['Technology', 'Programming'],
  ));
  
  // ลงทะเบียนสมาชิก
  lib.registerMember(Member(id: 'M001', name: 'สมชาย ใจดี', type: MemberType.student));
  lib.registerMember(Member(id: 'M002', name: 'สมหญิง รักอ่าน', type: MemberType.premium));
  
  // ยืมหนังสือ
  lib.borrowBook('M001', '978-0-14-028329-7');
  lib.borrowBook('M002', '978-0-7432-7356-5');
  
  // แสดงหนังสือว่าง
  print('\n📚 หนังสือที่ว่าง:');
  for (var book in lib.getAvailableBooks()) {
    print('  • $book');
  }
  
  // คืนหนังสือ
  lib.returnBook('M001', '978-0-14-028329-7');
  
  // Report
  lib.printReport();
}
```

---

## ขั้นตอนที่ 161-180: สรุปและ Exercises

### Exercise: Design Patterns พื้นฐาน

```dart
// 1. Builder Pattern
class Pizza {
  final String size;
  final String crust;
  final List<String> toppings;
  final bool extraCheese;
  final bool spicy;
  
  Pizza._({
    required this.size,
    required this.crust,
    required this.toppings,
    required this.extraCheese,
    required this.spicy,
  });
  
  @override
  String toString() {
    return 'Pizza($size, $crust crust, '
        '${toppings.join("+")}${extraCheese ? ", extra cheese" : ""}'
        '${spicy ? ", spicy" : ""})';
  }
}

class PizzaBuilder {
  String _size = 'M';
  String _crust = 'thin';
  List<String> _toppings = [];
  bool _extraCheese = false;
  bool _spicy = false;
  
  PizzaBuilder size(String size) => this.._size = size;
  PizzaBuilder crust(String crust) => this.._crust = crust;
  PizzaBuilder topping(String topping) => this.._toppings.add(topping);
  PizzaBuilder withExtraCheese() => this.._extraCheese = true;
  PizzaBuilder withSpicy() => this.._spicy = true;
  
  Pizza build() => Pizza._(
    size: _size,
    crust: _crust,
    toppings: _toppings,
    extraCheese: _extraCheese,
    spicy: _spicy,
  );
}

// 2. Observer Pattern
abstract class Observer<T> {
  void update(T data);
}

class EventSubject<T> {
  final List<Observer<T>> _observers = [];
  
  void subscribe(Observer<T> observer) => _observers.add(observer);
  void unsubscribe(Observer<T> observer) => _observers.remove(observer);
  void notify(T data) => _observers.forEach((o) => o.update(data));
}

class StockPriceObserver implements Observer<Map<String, double>> {
  final String name;
  StockPriceObserver(this.name);
  
  @override
  void update(Map<String, double> data) {
    print('[$name] ราคาหุ้นเปลี่ยน: $data');
  }
}

void main() {
  // Builder Pattern
  Pizza pizza = PizzaBuilder()
      .size('L')
      .crust('thick')
      .topping('pepperoni')
      .topping('mushroom')
      .withExtraCheese()
      .build();
  
  print(pizza);
  
  // Observer Pattern
  EventSubject<Map<String, double>> stockFeed = EventSubject();
  
  stockFeed.subscribe(StockPriceObserver('Investor A'));
  stockFeed.subscribe(StockPriceObserver('Investor B'));
  
  stockFeed.notify({'AAPL': 180.50, 'GOOGL': 140.25});
  stockFeed.notify({'AAPL': 182.00, 'MSFT': 370.50});
}
```

### สรุป Part 06

```
✅ ขั้นตอนที่ 141: Class พื้นฐาน
✅ ขั้นตอนที่ 142: Constructors ทุกประเภท
✅ ขั้นตอนที่ 143: Encapsulation - Private Members
✅ ขั้นตอนที่ 144: Getters และ Setters
✅ ขั้นตอนที่ 145: Static Members
✅ ขั้นตอนที่ 146: Immutable Classes
✅ ขั้นตอนที่ 147: Operator Overloading
✅ ขั้นตอนที่ 148: Abstract Classes
✅ ขั้นตอนที่ 149: Interfaces
✅ ขั้นตอนที่ 150: Mixins
✅ ขั้นตอนที่ 151-160: โปรเจกต์ - Library System
✅ ขั้นตอนที่ 161-180: Exercises และ Design Patterns
```

---

**← [Part 05 - Collections](part-05-collections.md)**

**ต่อไป: [Part 07 - Inheritance และ Polymorphism →](part-07-inheritance-polymorphism.md)**
