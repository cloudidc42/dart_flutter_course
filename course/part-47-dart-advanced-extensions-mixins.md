# Part 47: Dart Advanced Extensions & Mixins
## ขั้นตอนที่ 1761-1800

## 🎯 เป้าหมายของ Part นี้
- เขียน Extension Methods บน built-in types (String, List, DateTime, int)
- สร้าง Generic Extensions ที่ใช้งานได้กับหลาย types
- ออกแบบ Mixins สำหรับ Comparable, Serializable, Validatable
- แก้ไข Mixin conflict ด้วย on clause
- สร้าง Abstract Mixins พร้อม required methods

---

## ขั้นตอนที่ 1761: String Extensions

```dart
// lib/extensions/string_extensions.dart
extension StringExtensions on String {
  /// Converts to Title Case: "hello world" → "Hello World"
  String toTitleCase() {
    if (isEmpty) return this;
    return split(' ')
        .map((word) => word.isEmpty
            ? word
            : '${word[0].toUpperCase()}${word.substring(1).toLowerCase()}')
        .join(' ');
  }

  /// Converts to camelCase: "hello_world_test" → "helloWorldTest"
  String toCamelCase() {
    if (isEmpty) return this;
    final words = split(RegExp(r'[_\s-]+'));
    if (words.isEmpty) return this;
    return words.first.toLowerCase() +
        words.skip(1).map((w) => w.toTitleCase()).join('');
  }

  /// Converts to snake_case: "HelloWorld" → "hello_world"
  String toSnakeCase() {
    return replaceAllMapped(
      RegExp(r'[A-Z]'),
      (match) => '_${match.group(0)!.toLowerCase()}',
    ).replaceAll(RegExp(r'^_'), '');
  }

  /// Truncates string to maxLength and adds ellipsis
  String truncate(int maxLength, {String ellipsis = '...'}) {
    if (length <= maxLength) return this;
    return '${substring(0, maxLength - ellipsis.length)}$ellipsis';
  }

  /// Returns true if string is a valid email
  bool get isValidEmail {
    return RegExp(
      r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$',
    ).hasMatch(this);
  }

  /// Returns true if string is a valid URL
  bool get isValidUrl {
    try {
      final uri = Uri.parse(this);
      return uri.hasScheme && (uri.scheme == 'http' || uri.scheme == 'https');
    } catch (_) {
      return false;
    }
  }

  /// Returns true if string contains only digits
  bool get isNumeric => RegExp(r'^\d+$').hasMatch(this);

  /// Removes all whitespace characters
  String get removeWhitespace => replaceAll(RegExp(r'\s+'), '');

  /// Converts string to int safely, returns null if not parseable
  int? toIntOrNull() => int.tryParse(trim());

  /// Converts string to double safely, returns null if not parseable
  double? toDoubleOrNull() => double.tryParse(trim());

  /// Reverses the string
  String get reversed => split('').reversed.join('');

  /// Returns a masked version (for passwords, card numbers, etc.)
  String mask({int visibleChars = 4, String maskChar = '*'}) {
    if (length <= visibleChars) return this;
    final visible = substring(length - visibleChars);
    final masked = maskChar * (length - visibleChars);
    return '$masked$visible';
  }

  /// Counts occurrences of a substring
  int countOccurrences(String sub) {
    if (sub.isEmpty) return 0;
    int count = 0;
    int index = 0;
    while ((index = indexOf(sub, index)) != -1) {
      count++;
      index += sub.length;
    }
    return count;
  }

  /// Wraps text at given width
  String wordWrap(int width) {
    final words = split(' ');
    final lines = <String>[];
    var currentLine = '';

    for (final word in words) {
      if (currentLine.isEmpty) {
        currentLine = word;
      } else if (currentLine.length + 1 + word.length <= width) {
        currentLine += ' $word';
      } else {
        lines.add(currentLine);
        currentLine = word;
      }
    }
    if (currentLine.isNotEmpty) lines.add(currentLine);
    return lines.join('\n');
  }
}

void main() {
  // Test String extensions
  print('hello world'.toTitleCase()); // Hello World
  print('hello_world_test'.toCamelCase()); // helloWorldTest
  print('HelloWorldTest'.toSnakeCase()); // hello_world_test
  print('Long text that is too long'.truncate(15)); // Long text that...
  print('user@example.com'.isValidEmail); // true
  print('not-an-email'.isValidEmail); // false
  print('https://example.com'.isValidUrl); // true
  print('123'.isNumeric); // true
  print('abc123'.isNumeric); // false
  print('Hello'.reversed); // olleH
  print('4111111111111111'.mask()); // ************1111
  print('hello world hello'.countOccurrences('hello')); // 2

  print('--- camelCase ---');
  print('get user by id'.toCamelCase()); // getUserById
}
```

---

## ขั้นตอนที่ 1762: List Extensions

```dart
// lib/extensions/list_extensions.dart
extension ListExtensions<T> on List<T> {
  /// Splits list into chunks of given size
  List<List<T>> chunked(int chunkSize) {
    if (chunkSize <= 0) throw ArgumentError('chunkSize must be > 0');
    final chunks = <List<T>>[];
    for (int i = 0; i < length; i += chunkSize) {
      chunks.add(sublist(i, (i + chunkSize).clamp(0, length)));
    }
    return chunks;
  }

  /// Returns distinct elements preserving order
  List<T> get distinct {
    final seen = <T>{};
    return where(seen.add).toList();
  }

  /// Interleaves another list: [1,2,3].interleave([a,b,c]) → [1,a,2,b,3,c]
  List<T> interleave(List<T> other) {
    final result = <T>[];
    final len = length > other.length ? length : other.length;
    for (int i = 0; i < len; i++) {
      if (i < length) result.add(this[i]);
      if (i < other.length) result.add(other[i]);
    }
    return result;
  }

  /// Returns element at index or null if out of bounds
  T? getOrNull(int index) {
    if (index < 0 || index >= length) return null;
    return this[index];
  }

  /// Groups elements by a key
  Map<K, List<T>> groupBy<K>(K Function(T) keySelector) {
    final map = <K, List<T>>{};
    for (final item in this) {
      final key = keySelector(item);
      map.putIfAbsent(key, () => []).add(item);
    }
    return map;
  }

  /// Flattens one level deep (for List<List<T>>)
  List<R> flatMap<R>(List<R> Function(T) transform) {
    return expand(transform).toList();
  }

  /// Returns the sum if T is num
  T? sum() {
    if (isEmpty) return null;
    if (this is List<int>) {
      return (this as List<int>).fold(0, (a, b) => a + b) as T;
    }
    if (this is List<double>) {
      return (this as List<double>).fold(0.0, (a, b) => a + b) as T;
    }
    throw UnsupportedError('sum() only works on List<int> or List<double>');
  }

  /// Rotate list by n positions
  List<T> rotate(int n) {
    if (isEmpty) return this;
    final offset = n % length;
    if (offset == 0) return List.from(this);
    return [...sublist(offset), ...sublist(0, offset)];
  }

  /// Returns second element or null
  T? get secondOrNull => length > 1 ? this[1] : null;

  /// Returns last element or null (extension on non-nullable)
  T? get lastOrNull => isEmpty ? null : last;

  /// Zip two lists together
  List<(T, R)> zipWith<R>(List<R> other) {
    final minLen = length < other.length ? length : other.length;
    return List.generate(minLen, (i) => (this[i], other[i]));
  }
}

extension NullableListExtensions<T> on List<T?> {
  /// Filters out null values
  List<T> get nonNulls => whereType<T>().toList();
}

void main() {
  final numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9];

  print(numbers.chunked(3)); // [[1,2,3],[4,5,6],[7,8,9]]
  print([1, 2, 2, 3, 1, 4].distinct); // [1, 2, 3, 4]
  print([1, 2, 3].interleave(['a', 'b', 'c'])); // [1,a,2,b,3,c]
  print(numbers.getOrNull(100)); // null
  print(numbers.rotate(3)); // [4,5,6,7,8,9,1,2,3]

  final people = [
    {'name': 'Alice', 'dept': 'Engineering'},
    {'name': 'Bob', 'dept': 'Marketing'},
    {'name': 'Charlie', 'dept': 'Engineering'},
  ];
  final grouped = people.groupBy((p) => p['dept']!);
  print(grouped.keys.toList()); // [Engineering, Marketing]

  final names = ['Alice', 'Bob', 'Charlie'];
  final ages = [30, 25, 35];
  print(names.zipWith(ages)); // [(Alice, 30), (Bob, 25), (Charlie, 35)]

  final withNulls = [1, null, 2, null, 3];
  print(withNulls.nonNulls); // [1, 2, 3]
}
```

---

## ขั้นตอนที่ 1763: DateTime Extensions

```dart
// lib/extensions/datetime_extensions.dart
extension DateTimeExtensions on DateTime {
  /// Returns true if date is today
  bool get isToday {
    final now = DateTime.now();
    return year == now.year && month == now.month && day == now.day;
  }

  /// Returns true if date is yesterday
  bool get isYesterday {
    final yesterday = DateTime.now().subtract(const Duration(days: 1));
    return year == yesterday.year &&
        month == yesterday.month &&
        day == yesterday.day;
  }

  /// Returns true if date is tomorrow
  bool get isTomorrow {
    final tomorrow = DateTime.now().add(const Duration(days: 1));
    return year == tomorrow.year &&
        month == tomorrow.month &&
        day == tomorrow.day;
  }

  /// Returns true if the date is in the past
  bool get isPast => isBefore(DateTime.now());

  /// Returns true if the date is in the future
  bool get isFuture => isAfter(DateTime.now());

  /// Returns start of day (midnight)
  DateTime get startOfDay => DateTime(year, month, day);

  /// Returns end of day (23:59:59.999)
  DateTime get endOfDay =>
      DateTime(year, month, day, 23, 59, 59, 999);

  /// Returns start of week (Monday)
  DateTime get startOfWeek {
    final daysFromMonday = weekday - 1;
    return startOfDay.subtract(Duration(days: daysFromMonday));
  }

  /// Returns end of week (Sunday)
  DateTime get endOfWeek => startOfWeek.add(const Duration(days: 6)).endOfDay;

  /// Returns start of month
  DateTime get startOfMonth => DateTime(year, month);

  /// Returns end of month
  DateTime get endOfMonth => DateTime(year, month + 1, 0, 23, 59, 59, 999);

  /// Adds business days (skipping Saturday/Sunday)
  DateTime addBusinessDays(int days) {
    var result = this;
    int remaining = days;
    while (remaining > 0) {
      result = result.add(const Duration(days: 1));
      if (result.weekday != DateTime.saturday &&
          result.weekday != DateTime.sunday) {
        remaining--;
      }
    }
    return result;
  }

  /// Returns human-readable relative time
  String toRelativeString() {
    final now = DateTime.now();
    final diff = now.difference(this);

    if (diff.inSeconds < 60) return 'just now';
    if (diff.inMinutes < 60) return '${diff.inMinutes} minutes ago';
    if (diff.inHours < 24) return '${diff.inHours} hours ago';
    if (diff.inDays < 7) return '${diff.inDays} days ago';
    if (diff.inDays < 30) return '${(diff.inDays / 7).floor()} weeks ago';
    if (diff.inDays < 365) return '${(diff.inDays / 30).floor()} months ago';
    return '${(diff.inDays / 365).floor()} years ago';
  }

  /// Format as ISO date string "yyyy-MM-dd"
  String toDateString() =>
      '$year-${month.toString().padLeft(2, '0')}-${day.toString().padLeft(2, '0')}';

  /// Format as Thai date (BE year)
  String toThaiDateString() {
    const thaiMonths = [
      'มกราคม', 'กุมภาพันธ์', 'มีนาคม', 'เมษายน',
      'พฤษภาคม', 'มิถุนายน', 'กรกฎาคม', 'สิงหาคม',
      'กันยายน', 'ตุลาคม', 'พฤศจิกายน', 'ธันวาคม',
    ];
    final beYear = year + 543;
    return '$day ${thaiMonths[month - 1]} $beYear';
  }

  /// Returns age in years from this date
  int get ageInYears {
    final now = DateTime.now();
    int age = now.year - year;
    if (now.month < month || (now.month == month && now.day < day)) {
      age--;
    }
    return age;
  }

  /// Returns number of days in this month
  int get daysInMonth => DateTime(year, month + 1, 0).day;

  /// Returns true if this year is a leap year
  bool get isLeapYear =>
      (year % 4 == 0 && year % 100 != 0) || (year % 400 == 0);
}

void main() {
  final now = DateTime.now();
  final past = DateTime(1990, 6, 15);
  final birthday = DateTime(1990, 5, 10);

  print(now.isToday); // true
  print(past.isPast); // true
  print(now.startOfDay); // 2026-10-03 00:00:00
  print(now.endOfMonth); // Last day of current month
  print(past.toRelativeString()); // "36 years ago"
  print(now.toThaiDateString()); // "3 ตุลาคม 2569"
  print(birthday.ageInYears); // Age from 1990-05-10
  print(DateTime(2024, 1, 1).isLeapYear); // true (2024 is leap year)
  print(DateTime(2025, 1, 1).isLeapYear); // false
  print(now.addBusinessDays(3)); // 3 business days from now
}
```

---

## ขั้นตอนที่ 1764: Int และ Num Extensions

```dart
// lib/extensions/num_extensions.dart
extension IntExtensions on int {
  /// Returns true if number is even
  bool get isEven => this % 2 == 0;

  /// Returns true if number is odd
  bool get isOdd => this % 2 != 0;

  /// Returns true if number is prime
  bool get isPrime {
    if (this < 2) return false;
    if (this == 2) return true;
    if (this % 2 == 0) return false;
    for (int i = 3; i * i <= this; i += 2) {
      if (this % i == 0) return false;
    }
    return true;
  }

  /// Returns factorial
  int get factorial {
    if (this < 0) throw ArgumentError('Factorial not defined for negative numbers');
    if (this <= 1) return 1;
    int result = 1;
    for (int i = 2; i <= this; i++) {
      result *= i;
    }
    return result;
  }

  /// Converts seconds to Duration
  Duration get seconds => Duration(seconds: this);

  /// Converts milliseconds to Duration
  Duration get milliseconds => Duration(milliseconds: this);

  /// Converts minutes to Duration
  Duration get minutes => Duration(minutes: this);

  /// Converts hours to Duration
  Duration get hours => Duration(hours: this);

  /// Converts days to Duration
  Duration get days => Duration(days: this);

  /// Formats number with thousands separator
  String get withCommas {
    final str = abs().toString();
    final result = StringBuffer();
    for (int i = 0; i < str.length; i++) {
      if (i > 0 && (str.length - i) % 3 == 0) result.write(',');
      result.write(str[i]);
    }
    return this < 0 ? '-$result' : result.toString();
  }

  /// Converts to human-readable file size
  String toFileSize() {
    const units = ['B', 'KB', 'MB', 'GB', 'TB'];
    double size = toDouble();
    int unitIndex = 0;
    while (size >= 1024 && unitIndex < units.length - 1) {
      size /= 1024;
      unitIndex++;
    }
    return '${size.toStringAsFixed(2)} ${units[unitIndex]}';
  }

  /// Executes a function n times
  void times(void Function(int index) action) {
    for (int i = 0; i < this; i++) {
      action(i);
    }
  }

  /// Returns range from 0 to this (exclusive)
  Iterable<int> get range => Iterable.generate(this);

  /// Clamps to range
  int clampTo(int min, int max) => clamp(min, max) as int;

  /// GCD (Greatest Common Divisor)
  int gcd(int other) {
    int a = abs();
    int b = other.abs();
    while (b != 0) {
      final t = b;
      b = a % b;
      a = t;
    }
    return a;
  }
}

extension DoubleExtensions on double {
  /// Rounds to n decimal places
  double roundTo(int decimalPlaces) {
    final factor = _pow10(decimalPlaces);
    return (this * factor).round() / factor;
  }

  double _pow10(int n) {
    double result = 1.0;
    for (int i = 0; i < n; i++) result *= 10;
    return result;
  }

  /// Converts degrees to radians
  double get toRadians => this * 3.141592653589793 / 180;

  /// Converts radians to degrees
  double get toDegrees => this * 180 / 3.141592653589793;

  /// Formats as currency string
  String toCurrency({String symbol = '฿', int decimals = 2}) {
    final formatted = toStringAsFixed(decimals);
    final parts = formatted.split('.');
    final intPart = int.parse(parts[0]).withCommas;
    return '$symbol$intPart.${parts[1]}';
  }
}

void main() {
  print(7.isPrime); // true
  print(6.isPrime); // false
  print(5.factorial); // 120
  print(1000000.withCommas); // 1,000,000
  print(1536000.toFileSize()); // 1.46 MB
  print(12.gcd(8)); // 4

  3.times((i) => print('Iteration $i'));
  // Iteration 0, Iteration 1, Iteration 2

  print(3.14159.roundTo(2)); // 3.14
  print(180.0.toRadians); // 3.14159...
  print(1234567.89.toCurrency()); // ฿1,234,567.89
}
```

---

## ขั้นตอนที่ 1765: Generic Extensions

```dart
// lib/extensions/generic_extensions.dart

/// Extension on nullable T
extension NullableExtension<T> on T? {
  /// Returns value or throws with custom message
  T orThrow([String? message]) {
    if (this == null) {
      throw StateError(message ?? 'Value is null');
    }
    return this!;
  }

  /// Returns value or default
  T orDefault(T defaultValue) => this ?? defaultValue;

  /// Applies function if not null
  R? let<R>(R Function(T) transform) {
    if (this == null) return null;
    return transform(this as T);
  }

  /// Runs side effect if not null
  T? also(void Function(T) effect) {
    if (this != null) effect(this as T);
    return this;
  }
}

/// Extension on any type T
extension AnyExtension<T> on T {
  /// Applies transform and returns result
  R let<R>(R Function(T) transform) => transform(this);

  /// Runs side effect and returns self
  T also(void Function(T) effect) {
    effect(this);
    return this;
  }

  /// Returns this if predicate is true, else null
  T? takeIf(bool Function(T) predicate) => predicate(this) ? this : null;

  /// Returns this if predicate is false, else null
  T? takeUnless(bool Function(T) predicate) =>
      predicate(this) ? null : this;
}

/// Extension on Future<T>
extension FutureExtensions<T> on Future<T> {
  /// Maps the future value
  Future<R> mapResult<R>(R Function(T) transform) => then(transform);

  /// Returns null on error instead of throwing
  Future<T?> orNull() async {
    try {
      return await this;
    } catch (_) {
      return null;
    }
  }

  /// Provides a fallback value on error
  Future<T> orElse(T Function(Object error) fallback) async {
    try {
      return await this;
    } catch (e) {
      return fallback(e);
    }
  }
}

/// Extension on Map<K, V>
extension MapExtensions<K, V> on Map<K, V> {
  /// Returns map without specified keys
  Map<K, V> withoutKeys(Set<K> keys) {
    return Map.fromEntries(entries.where((e) => !keys.contains(e.key)));
  }

  /// Returns map with only specified keys
  Map<K, V> withOnlyKeys(Set<K> keys) {
    return Map.fromEntries(entries.where((e) => keys.contains(e.key)));
  }

  /// Maps both keys and values
  Map<K2, V2> mapEntries<K2, V2>(
    MapEntry<K2, V2> Function(K key, V value) transform,
  ) {
    return Map.fromEntries(entries.map((e) => transform(e.key, e.value)));
  }

  /// Merges with another map, preferring this map's values
  Map<K, V> mergeWith(Map<K, V> other) {
    return {...other, ...this};
  }
}

void main() {
  // Nullable extensions
  String? name;
  print(name.orDefault('Anonymous')); // Anonymous
  print(name.let((n) => n.toUpperCase())); // null

  name = 'alice';
  print(name.let((n) => n.toUpperCase())); // ALICE

  // AnyExtension
  final result = 'hello'
      .also((s) => print('Value: $s')) // prints "Value: hello"
      .let((s) => s.toUpperCase())     // "HELLO"
      .takeIf((s) => s.length > 3);   // "HELLO" (length 5 > 3)

  print(result); // HELLO

  final short = 'hi'
      .takeIf((s) => s.length > 5); // null (length 2 not > 5)
  print(short); // null

  // Future extensions
  Future<int> fetchNumber() async => 42;
  Future<int> failingFetch() async => throw Exception('Failed');

  fetchNumber()
      .mapResult((n) => n * 2)
      .then(print); // 84

  failingFetch()
      .orElse((_) => -1)
      .then(print); // -1

  // Map extensions
  final data = {'a': 1, 'b': 2, 'c': 3, 'd': 4};
  print(data.withoutKeys({'a', 'c'})); // {b: 2, d: 4}
  print(data.withOnlyKeys({'a', 'd'})); // {a: 1, d: 4}
}
```

---

## ขั้นตอนที่ 1766: Comparable Mixin

```dart
// lib/mixins/comparable_mixin.dart

/// Mixin that provides comparison operators based on compareTo()
mixin ComparableMixin<T> implements Comparable<T> {
  bool operator <(T other) => compareTo(other) < 0;
  bool operator <=(T other) => compareTo(other) <= 0;
  bool operator >(T other) => compareTo(other) > 0;
  bool operator >=(T other) => compareTo(other) >= 0;

  T max(T other) => compareTo(other) >= 0 ? this as T : other;
  T min(T other) => compareTo(other) <= 0 ? this as T : other;

  bool isBetween(T lower, T upper) =>
      compareTo(lower) >= 0 && compareTo(upper) <= 0;
}

// Product class using ComparableMixin
class Product with ComparableMixin<Product> {
  final String name;
  final double price;
  final int stock;

  const Product({
    required this.name,
    required this.price,
    required this.stock,
  });

  @override
  int compareTo(Product other) => price.compareTo(other.price);

  @override
  String toString() => 'Product($name, \$$price)';
}

// Temperature class using ComparableMixin
class Temperature with ComparableMixin<Temperature> {
  final double celsius;

  const Temperature(this.celsius);

  double get fahrenheit => celsius * 9 / 5 + 32;
  double get kelvin => celsius + 273.15;

  @override
  int compareTo(Temperature other) => celsius.compareTo(other.celsius);

  @override
  String toString() => '${celsius.toStringAsFixed(1)}°C';
}

void main() {
  final cheap = Product(name: 'Widget', price: 9.99, stock: 100);
  final expensive = Product(name: 'Gadget', price: 99.99, stock: 10);
  final mid = Product(name: 'Doohickey', price: 49.99, stock: 50);

  print(cheap < expensive); // true
  print(expensive > mid); // true
  print(mid.isBetween(cheap, expensive)); // true
  print(expensive.max(cheap)); // Product(Gadget, $99.99)

  final products = [expensive, cheap, mid];
  products.sort();
  print(products.map((p) => p.name)); // (Widget, Doohickey, Gadget)

  final boiling = Temperature(100);
  final freezing = Temperature(0);
  final body = Temperature(37);

  print(boiling > freezing); // true
  print(body.isBetween(freezing, boiling)); // true
  print(body.min(boiling)); // 37.0°C
}
```

---

## ขั้นตอนที่ 1767: Serializable Mixin

```dart
// lib/mixins/serializable_mixin.dart
import 'dart:convert';

/// Mixin that provides JSON serialization/deserialization support
mixin SerializableMixin {
  /// Subclasses must implement this to provide JSON representation
  Map<String, dynamic> toJson();

  /// Returns JSON string
  String toJsonString({bool pretty = false}) {
    return pretty
        ? const JsonEncoder.withIndent('  ').convert(toJson())
        : json.encode(toJson());
  }

  /// Returns a deep copy via JSON roundtrip
  @override
  String toString() => toJsonString(pretty: true);
}

/// Abstract base for entities with fromJson factory
abstract class JsonSerializable with SerializableMixin {
  const JsonSerializable();
}

// Address class
class Address extends JsonSerializable {
  final String street;
  final String city;
  final String country;
  final String postalCode;

  const Address({
    required this.street,
    required this.city,
    required this.country,
    required this.postalCode,
  });

  factory Address.fromJson(Map<String, dynamic> json) {
    return Address(
      street: json['street'] as String,
      city: json['city'] as String,
      country: json['country'] as String,
      postalCode: json['postal_code'] as String,
    );
  }

  @override
  Map<String, dynamic> toJson() => {
        'street': street,
        'city': city,
        'country': country,
        'postal_code': postalCode,
      };
}

// Employee class using SerializableMixin
class Employee extends JsonSerializable {
  final int id;
  final String firstName;
  final String lastName;
  final double salary;
  final Address address;
  final List<String> skills;
  final DateTime hireDate;

  const Employee({
    required this.id,
    required this.firstName,
    required this.lastName,
    required this.salary,
    required this.address,
    required this.skills,
    required this.hireDate,
  });

  factory Employee.fromJson(Map<String, dynamic> json) {
    return Employee(
      id: json['id'] as int,
      firstName: json['first_name'] as String,
      lastName: json['last_name'] as String,
      salary: (json['salary'] as num).toDouble(),
      address: Address.fromJson(json['address'] as Map<String, dynamic>),
      skills: List<String>.from(json['skills'] as List),
      hireDate: DateTime.parse(json['hire_date'] as String),
    );
  }

  String get fullName => '$firstName $lastName';

  @override
  Map<String, dynamic> toJson() => {
        'id': id,
        'first_name': firstName,
        'last_name': lastName,
        'salary': salary,
        'address': address.toJson(),
        'skills': skills,
        'hire_date': hireDate.toIso8601String(),
      };
}

void main() {
  final emp = Employee(
    id: 1,
    firstName: 'Alice',
    lastName: 'Smith',
    salary: 75000.0,
    address: const Address(
      street: '123 Main St',
      city: 'Bangkok',
      country: 'Thailand',
      postalCode: '10100',
    ),
    skills: ['Flutter', 'Dart', 'Firebase'],
    hireDate: DateTime(2022, 1, 15),
  );

  // Serialize to JSON
  print(emp.toJsonString(pretty: true));

  // Deserialize from JSON
  final jsonStr = emp.toJsonString();
  final decoded = Employee.fromJson(json.decode(jsonStr));
  print(decoded.fullName); // Alice Smith
  print(decoded.skills); // [Flutter, Dart, Firebase]
  print(decoded.address.city); // Bangkok
}
```

---

## ขั้นตอนที่ 1768: Validatable Mixin

```dart
// lib/mixins/validatable_mixin.dart

/// Represents a validation error
class ValidationError {
  final String field;
  final String message;

  const ValidationError({required this.field, required this.message});

  @override
  String toString() => '[$field]: $message';
}

/// Result of a validation operation
class ValidationResult {
  final List<ValidationError> errors;

  const ValidationResult._(this.errors);

  factory ValidationResult.valid() => const ValidationResult._([]);
  factory ValidationResult.invalid(List<ValidationError> errors) =>
      ValidationResult._(errors);

  bool get isValid => errors.isEmpty;
  bool get isInvalid => errors.isNotEmpty;

  @override
  String toString() => isValid
      ? 'ValidationResult: valid'
      : 'ValidationResult: invalid\n${errors.join('\n')}';
}

/// Mixin for validation
mixin ValidatableMixin {
  /// Subclasses define their validation rules
  List<ValidationError> validate();

  ValidationResult get validationResult {
    final errors = validate();
    return errors.isEmpty
        ? ValidationResult.valid()
        : ValidationResult.invalid(errors);
  }

  bool get isValid => validate().isEmpty;

  /// Throws if validation fails
  void ensureValid() {
    final result = validationResult;
    if (result.isInvalid) {
      throw ValidationException(result.errors);
    }
  }
}

class ValidationException implements Exception {
  final List<ValidationError> errors;
  const ValidationException(this.errors);

  @override
  String toString() =>
      'ValidationException: ${errors.map((e) => e.toString()).join(', ')}';
}

// RegistrationForm using ValidatableMixin
class RegistrationForm with ValidatableMixin {
  final String username;
  final String email;
  final String password;
  final String confirmPassword;
  final int age;

  const RegistrationForm({
    required this.username,
    required this.email,
    required this.password,
    required this.confirmPassword,
    required this.age,
  });

  @override
  List<ValidationError> validate() {
    final errors = <ValidationError>[];

    if (username.length < 3) {
      errors.add(const ValidationError(
        field: 'username',
        message: 'Username must be at least 3 characters',
      ));
    }
    if (username.length > 20) {
      errors.add(const ValidationError(
        field: 'username',
        message: 'Username must be at most 20 characters',
      ));
    }

    if (!RegExp(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
        .hasMatch(email)) {
      errors.add(const ValidationError(
        field: 'email',
        message: 'Invalid email format',
      ));
    }

    if (password.length < 8) {
      errors.add(const ValidationError(
        field: 'password',
        message: 'Password must be at least 8 characters',
      ));
    }

    if (password != confirmPassword) {
      errors.add(const ValidationError(
        field: 'confirmPassword',
        message: 'Passwords do not match',
      ));
    }

    if (age < 13) {
      errors.add(const ValidationError(
        field: 'age',
        message: 'Must be at least 13 years old',
      ));
    }

    return errors;
  }
}

void main() {
  final validForm = RegistrationForm(
    username: 'alice_dev',
    email: 'alice@example.com',
    password: 'secure123',
    confirmPassword: 'secure123',
    age: 25,
  );

  print(validForm.isValid); // true
  print(validForm.validationResult); // ValidationResult: valid

  final invalidForm = RegistrationForm(
    username: 'ab', // too short
    email: 'not-an-email',
    password: 'short',
    confirmPassword: 'different',
    age: 10, // too young
  );

  final result = invalidForm.validationResult;
  print(result.isInvalid); // true
  for (final error in result.errors) {
    print(error); // prints all validation errors
  }

  // Try ensureValid
  try {
    invalidForm.ensureValid();
  } catch (e) {
    print('Caught: $e');
  }
}
```

---

## ขั้นตอนที่ 1769: Mixin Conflict Resolution

```dart
// lib/mixins/mixin_conflicts.dart

// When two mixins have methods with the same name,
// the LAST mixin in the with clause wins.

mixin LoggingMixin {
  String get prefix => '[LOG]';

  void log(String message) {
    print('$prefix $message');
  }
}

mixin TimestampMixin {
  String get prefix => '[TIMESTAMP]'; // Conflicts with LoggingMixin.prefix!

  void log(String message) {
    final ts = DateTime.now().toIso8601String();
    print('[$ts] $message');
  }
}

// TimestampMixin wins because it comes LAST
class Service with LoggingMixin, TimestampMixin {
  void doWork() {
    log('Doing work'); // Uses TimestampMixin.log
    print(prefix); // Uses TimestampMixin.prefix = '[TIMESTAMP]'
  }
}

// LoggingMixin wins because it comes LAST
class OtherService with TimestampMixin, LoggingMixin {
  void doWork() {
    log('Doing work'); // Uses LoggingMixin.log
    print(prefix); // Uses LoggingMixin.prefix = '[LOG]'
  }
}

// Using 'on' clause to restrict mixin usage
mixin DatabaseMixin on Repository {
  Future<void> saveToDatabase(Map<String, dynamic> data) async {
    // Can safely call Repository methods here
    final existing = await findById(data['id'] as int);
    if (existing != null) {
      print('Updating existing record');
    } else {
      print('Creating new record');
    }
  }
}

abstract class Repository {
  Future<Map<String, dynamic>?> findById(int id);
  Future<void> save(Map<String, dynamic> data);
}

// This works: UserRepository extends Repository
class UserRepository extends Repository with DatabaseMixin {
  final Map<int, Map<String, dynamic>> _store = {};

  @override
  Future<Map<String, dynamic>?> findById(int id) async => _store[id];

  @override
  Future<void> save(Map<String, dynamic> data) async {
    _store[data['id'] as int] = data;
  }
}

// Abstract mixin with required implementations
mixin CacheMixin {
  // Abstract - subclasses must implement cache store access
  Map<String, dynamic> get cacheStore;

  void cacheSet(String key, dynamic value) {
    cacheStore[key] = value;
    print('Cached: $key');
  }

  dynamic cacheGet(String key) {
    return cacheStore[key];
  }

  void cacheInvalidate(String key) {
    cacheStore.remove(key);
    print('Invalidated: $key');
  }

  void cacheClear() {
    cacheStore.clear();
    print('Cache cleared');
  }
}

class ApiService with CacheMixin {
  final Map<String, dynamic> _cache = {};

  @override
  Map<String, dynamic> get cacheStore => _cache;

  Future<String> fetchData(String endpoint) async {
    final cached = cacheGet(endpoint);
    if (cached != null) {
      print('Cache hit for $endpoint');
      return cached as String;
    }

    // Simulated network call
    await Future.delayed(const Duration(milliseconds: 100));
    final data = 'Data from $endpoint';
    cacheSet(endpoint, data);
    return data;
  }
}

void main() async {
  // Mixin conflict demo
  final service = Service();
  service.doWork();
  // Uses TimestampMixin (last in with clause)

  final otherService = OtherService();
  otherService.doWork();
  // Uses LoggingMixin (last in with clause)

  // Restricted mixin demo
  final userRepo = UserRepository();
  await userRepo.saveToDatabase({'id': 1, 'name': 'Alice'});

  // Cache mixin demo
  final api = ApiService();
  final data1 = await api.fetchData('/users'); // Network call
  final data2 = await api.fetchData('/users'); // Cache hit
  print(data1 == data2); // true

  api.cacheInvalidate('/users');
  final data3 = await api.fetchData('/users'); // Network call again
  print(data3); // Data from /users
}
```

---

**← [Part 46 - Advanced Testing](part-46-advanced-testing.md)**
**ต่อไป: [Part 48 - App Architecture Patterns →](part-48-app-architecture-patterns.md)**
