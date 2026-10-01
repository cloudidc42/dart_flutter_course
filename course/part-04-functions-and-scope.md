# Part 04: Functions และ Scope
## ขั้นตอนที่ 81-110

---

## 🎯 เป้าหมายของ Part นี้

- ประกาศและเรียกใช้ Functions
- เข้าใจ Parameter types ทั้งหมด
- ใช้ Arrow Functions และ Anonymous Functions
- เข้าใจ Closures และ Scope
- ใช้ Higher-Order Functions
- เข้าใจ Recursion
- ใช้ Extension Methods

---

## ขั้นตอนที่ 81: Function พื้นฐาน

```dart
// ─────────────── Function ไม่คืนค่า ───────────────
void sayHello() {
  print('สวัสดี!');
}

// ─────────────── Function คืนค่า ───────────────
int add(int a, int b) {
  return a + b;
}

// ─────────────── Arrow Function (ฟังก์ชันบรรทัดเดียว) ───────────────
int multiply(int a, int b) => a * b;
double square(double n) => n * n;
bool isEven(int n) => n % 2 == 0;
String greet(String name) => 'สวัสดี $name!';

// ─────────────── Function ที่คืนหลายค่า (ใช้ Record) ───────────────
(int, int) divmod(int a, int b) {
  return (a ~/ b, a % b);
}

void main() {
  // เรียกใช้
  sayHello();
  
  int result = add(5, 3);
  print('5 + 3 = $result');
  
  print('4 × 6 = ${multiply(4, 6)}');
  print('3² = ${square(3)}');
  print('isEven(4) = ${isEven(4)}');
  print(greet('Flutter'));
  
  var (quotient, remainder) = divmod(17, 5);
  print('17 ÷ 5 = $quotient เศษ $remainder');
}
```

---

## ขั้นตอนที่ 82: Parameters ชนิดต่างๆ

```dart
// ─────────────── Required Positional Parameters ───────────────
double calculateArea(double width, double height) {
  return width * height;
}

// ─────────────── Optional Positional Parameters ───────────────
// ใช้ [] สำหรับ optional
String buildAddress(String city, [String? district, String? street]) {
  String address = city;
  if (district != null) address = '$district, $address';
  if (street != null) address = '$street, $address';
  return address;
}

// ─────────────── Named Parameters ───────────────
// ใช้ {} สำหรับ named
void createUser({
  required String name,     // required named parameter
  required String email,
  int age = 0,               // optional named with default
  String role = 'user',
}) {
  print('ชื่อ: $name, Email: $email, อายุ: $age, Role: $role');
}

// ─────────────── Mixed Parameters ───────────────
String formatName(
  String firstName,           // required positional
  String lastName,            // required positional
  {
    String? title,            // optional named
    String suffix = '',       // optional named with default
  }
) {
  String result = '$firstName $lastName';
  if (title != null) result = '$title $result';
  if (suffix.isNotEmpty) result = '$result $suffix';
  return result;
}

void main() {
  // Required positional
  print(calculateArea(5, 3));   // 15.0
  
  // Optional positional
  print(buildAddress('กรุงเทพ'));
  print(buildAddress('กรุงเทพ', 'ลาดพร้าว'));
  print(buildAddress('กรุงเทพ', 'ลาดพร้าว', 'ถนนลาดพร้าว'));
  
  // Named parameters
  createUser(name: 'Alice', email: 'alice@example.com');
  createUser(name: 'Bob', email: 'bob@example.com', age: 25, role: 'admin');
  
  // Mixed
  print(formatName('สมชาย', 'ใจดี'));
  print(formatName('สมชาย', 'ใจดี', title: 'นาย', suffix: 'Ph.D.'));
}
```

---

## ขั้นตอนที่ 83: Default Parameter Values

```dart
// ─────────────── Default Values ───────────────
void printMessage(String message, {
  String prefix = '📢',
  String suffix = '',
  bool uppercase = false,
}) {
  String formatted = uppercase ? message.toUpperCase() : message;
  print('$prefix $formatted$suffix');
}

// ─────────────── Default กับ Null Safety ───────────────
String createSlug(String title, {String separator = '-'}) {
  return title
      .toLowerCase()
      .replaceAll(' ', separator)
      .replaceAll(RegExp(r'[^a-z0-9\-_]'), '');
}

// ─────────────── Default value จาก function ───────────────
DateTime getExpiryDate({
  DateTime? from,
  int days = 30,
}) {
  DateTime startDate = from ?? DateTime.now();
  return startDate.add(Duration(days: days));
}

void main() {
  printMessage('สวัสดีครับ');
  printMessage('สวัสดีครับ', prefix: '🔔');
  printMessage('สวัสดีครับ', uppercase: true, suffix: '!');
  
  print(createSlug('Hello World Flutter'));     // hello-world-flutter
  print(createSlug('Hello World Flutter', separator: '_')); // hello_world_flutter
  
  print('หมดอายุ: ${getExpiryDate()}');
  print('หมดอายุ: ${getExpiryDate(days: 7)}');
}
```

---

## ขั้นตอนที่ 84: Functions เป็น First-Class Objects

```dart
void main() {
  // ─────────────── Functions สามารถเก็บในตัวแปรได้ ───────────────
  Function greet = (String name) => 'สวัสดี $name!';
  print(greet('Alice'));  // สวัสดี Alice!
  
  // ─────────────── Function Type ───────────────
  int Function(int, int) add = (a, b) => a + b;
  int Function(int, int) subtract = (a, b) => a - b;
  
  print(add(5, 3));       // 8
  print(subtract(5, 3));  // 2
  
  // ─────────────── Functions ใน List ───────────────
  List<int Function(int)> transformations = [
    (n) => n * 2,
    (n) => n + 10,
    (n) => n * n,
  ];
  
  int value = 5;
  for (var transform in transformations) {
    print(transform(value));  // 10, 15, 25
  }
  
  // ─────────────── Functions เป็น Parameter ───────────────
  int applyTwice(int Function(int) fn, int value) {
    return fn(fn(value));
  }
  
  print(applyTwice((n) => n + 1, 5));   // 7
  print(applyTwice((n) => n * 2, 3));   // 12
  
  // ─────────────── Functions คืนค่า Function (Currying) ───────────────
  int Function(int) adder(int x) {
    return (y) => x + y;
  }
  
  var add5 = adder(5);
  var add10 = adder(10);
  
  print(add5(3));   // 8
  print(add10(3));  // 13
}
```

---

## ขั้นตอนที่ 85: Anonymous Functions และ Closures

```dart
void main() {
  // ─────────────── Anonymous Function ───────────────
  var multiply = (int a, int b) {
    return a * b;
  };
  
  print(multiply(4, 5));  // 20
  
  // ─────────────── Immediately Invoked Function ───────────────
  int result = ((int x) => x * x)(7);  // IIFE
  print(result);  // 49
  
  // ─────────────── Closure ───────────────
  // Function ที่ "จำ" ตัวแปรจาก scope ที่สร้างมัน
  
  int Function() makeCounter() {
    int count = 0;  // Captured variable
    return () {
      count++;
      return count;
    };
  }
  
  var counter1 = makeCounter();
  var counter2 = makeCounter();  // State แยกกัน
  
  print(counter1());  // 1
  print(counter1());  // 2
  print(counter1());  // 3
  print(counter2());  // 1 (counter แยกกัน)
  
  // ─────────────── Closure กับ Parameter ───────────────
  String Function(String) makeGreeter(String greeting) {
    return (String name) => '$greeting, $name!';
  }
  
  var sayHello = makeGreeter('สวัสดี');
  var sayHi = makeGreeter('Hi');
  
  print(sayHello('Alice'));  // สวัสดี, Alice!
  print(sayHi('Bob'));       // Hi, Bob!
  
  // ─────────────── Closure ใน Loop ───────────────
  // ระวัง! ปัญหา closure ใน loop
  List<Function> funcs = [];
  
  for (int i = 0; i < 3; i++) {
    int captured = i;  // capture ค่า i ใน local variable
    funcs.add(() => print(captured));
  }
  
  for (var f in funcs) {
    f();  // 0, 1, 2 (ไม่ใช่ 2, 2, 2)
  }
}
```

---

## ขั้นตอนที่ 86: Higher-Order Functions

```dart
void main() {
  List<int> numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  
  // ─────────────── map ───────────────
  // แปลงทุก element
  List<int> squared = numbers.map((n) => n * n).toList();
  print('Squared: $squared');
  
  // ─────────────── filter (where) ───────────────
  // กรอง element
  List<int> evens = numbers.where((n) => n % 2 == 0).toList();
  print('Evens: $evens');
  
  // ─────────────── reduce ───────────────
  // รวมเป็นค่าเดียว
  int sum = numbers.reduce((acc, n) => acc + n);
  print('Sum: $sum');
  
  int product = numbers.reduce((acc, n) => acc * n);
  print('Product: $product');
  
  // ─────────────── fold ───────────────
  // เหมือน reduce แต่มี initial value
  int sumFold = numbers.fold(0, (acc, n) => acc + n);
  Map<String, List<int>> grouped = numbers.fold(
    {'odd': [], 'even': []},
    (acc, n) {
      if (n % 2 == 0) {
        acc['even']!.add(n);
      } else {
        acc['odd']!.add(n);
      }
      return acc;
    }
  );
  print('Grouped: $grouped');
  
  // ─────────────── Custom Higher-Order Functions ───────────────
  
  // compose: รวม 2 functions
  T Function(T) compose<T>(T Function(T) f, T Function(T) g) {
    return (x) => f(g(x));
  }
  
  var doubleAndAddOne = compose((x) => x + 1, (x) => x * 2);
  print('compose(double, +1)(5) = ${doubleAndAddOne(5)}');  // 11
  
  // pipeline
  List<int> pipeline(List<int> list, List<List<int> Function(List<int>)> ops) {
    return ops.fold(list, (current, op) => op(current));
  }
  
  List<int> result = pipeline(
    [1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
    [
      (list) => list.where((n) => n % 2 == 0).toList(),
      (list) => list.map((n) => n * n).toList(),
      (list) => list.where((n) => n > 20).toList(),
    ],
  );
  print('Pipeline result: $result');  // [36, 64, 100]
  
  // ─────────────── memoize ───────────────
  // Cache ผลลัพธ์ของ function
  T Function(K) memoize<K, T>(T Function(K) fn) {
    Map<K, T> cache = {};
    return (key) => cache.putIfAbsent(key, () => fn(key));
  }
  
  int callCount = 0;
  int expensiveCalc(int n) {
    callCount++;
    return n * n;
  }
  
  var memoized = memoize(expensiveCalc);
  print(memoized(5));  // 25 (คำนวณ)
  print(memoized(5));  // 25 (จาก cache)
  print(memoized(6));  // 36 (คำนวณ)
  print('Call count: $callCount');  // 2 (ไม่ใช่ 3)
}
```

---

## ขั้นตอนที่ 87: Scope และ Variable Visibility

```dart
// ─────────────── Top-level variables ───────────────
String appName = 'My App';
const String version = '1.0.0';

void main() {
  // ─────────────── Function Scope ───────────────
  String localVar = 'ตัวแปรท้องถิ่น';
  print(localVar);
  print(appName);  // เข้าถึง top-level ได้
  
  // ─────────────── Block Scope ───────────────
  {
    String blockVar = 'ตัวแปรใน block';
    print(blockVar);  // OK
    print(localVar);  // OK (เข้าถึง outer scope ได้)
  }
  // print(blockVar);  // Error! ออก scope แล้ว
  
  // ─────────────── Shadowing ───────────────
  String appName = 'Local App';  // shadow top-level
  print(appName);  // Local App (local wins)
  
  // ─────────────── Loop Scope ───────────────
  for (int i = 0; i < 3; i++) {
    String loopVar = 'loop $i';
    print(loopVar);
  }
  // print(i);       // Error! i ออก scope แล้ว
  // print(loopVar); // Error! loopVar ออก scope แล้ว
}

// ─────────────── Lexical Scope ───────────────
int outer = 10;

void outerFunc() {
  int middle = 20;
  
  void innerFunc() {
    int inner = 30;
    print(outer);   // OK - เข้าถึง outer scope
    print(middle);  // OK - เข้าถึง middle scope
    print(inner);   // OK - local scope
  }
  
  innerFunc();
  // print(inner);  // Error! inner ออก scope แล้ว
}
```

---

## ขั้นตอนที่ 88: Recursion

```dart
void main() {
  // ─────────────── Factorial ───────────────
  int factorial(int n) {
    if (n <= 1) return 1;      // Base case
    return n * factorial(n - 1); // Recursive case
  }
  
  for (int i = 0; i <= 10; i++) {
    print('$i! = ${factorial(i)}');
  }
  
  // ─────────────── Fibonacci ───────────────
  int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
  }
  
  print('\nFibonacci:');
  for (int i = 0; i <= 10; i++) {
    print('F($i) = ${fibonacci(i)}');
  }
  
  // ─────────────── Fibonacci ด้วย Memoization (ดีกว่า) ───────────────
  Map<int, int> memo = {};
  
  int fibMemo(int n) {
    if (n <= 1) return n;
    return memo.putIfAbsent(n, () => fibMemo(n - 1) + fibMemo(n - 2));
  }
  
  print('\nFibonacci (มี memoization):');
  print('F(40) = ${fibMemo(40)}');
  
  // ─────────────── Binary Search ───────────────
  int binarySearch(List<int> sorted, int target, [int? low, int? high]) {
    low ??= 0;
    high ??= sorted.length - 1;
    
    if (low > high) return -1;  // ไม่พบ
    
    int mid = (low + high) ~/ 2;
    
    if (sorted[mid] == target) return mid;
    if (sorted[mid] < target) return binarySearch(sorted, target, mid + 1, high);
    return binarySearch(sorted, target, low, mid - 1);
  }
  
  List<int> sorted = [1, 3, 5, 7, 9, 11, 13, 15, 17, 19];
  print('\nBinary Search:');
  print('หา 7: index ${binarySearch(sorted, 7)}');   // 3
  print('หา 10: index ${binarySearch(sorted, 10)}'); // -1
  
  // ─────────────── Tower of Hanoi ───────────────
  int moves = 0;
  
  void hanoi(int n, String from, String to, String via) {
    if (n == 1) {
      print('  ย้าย disk 1 จาก $from ไป $to');
      moves++;
      return;
    }
    hanoi(n - 1, from, via, to);
    print('  ย้าย disk $n จาก $from ไป $to');
    moves++;
    hanoi(n - 1, via, to, from);
  }
  
  print('\nTower of Hanoi (3 disks):');
  hanoi(3, 'A', 'C', 'B');
  print('จำนวนการย้าย: $moves ครั้ง');
}
```

---

## ขั้นตอนที่ 89: Extension Methods

```dart
// ─────────────── Extension บน String ───────────────
extension StringExtensions on String {
  // แปลงเป็น Title Case
  String toTitleCase() {
    return split(' ')
        .map((word) => word.isEmpty 
            ? word 
            : word[0].toUpperCase() + word.substring(1).toLowerCase())
        .join(' ');
  }
  
  // ตรวจสอบว่าเป็น email หรือไม่
  bool get isEmail {
    RegExp emailRegex = RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$');
    return emailRegex.hasMatch(this);
  }
  
  // แปลงเป็น Slug
  String toSlug() {
    return toLowerCase()
        .replaceAll(' ', '-')
        .replaceAll(RegExp(r'[^a-z0-9\-]'), '');
  }
  
  // นับคำ
  int get wordCount => trim().isEmpty ? 0 : trim().split(RegExp(r'\s+')).length;
  
  // ย้อนกลับ
  String get reversed => split('').reversed.join('');
  
  // ทำซ้ำ
  String repeat(int times) => this * times;
}

// ─────────────── Extension บน int ───────────────
extension IntExtensions on int {
  // ตรวจสอบเฉพาะ
  bool get isPrime {
    if (this < 2) return false;
    if (this == 2) return true;
    if (this % 2 == 0) return false;
    for (int i = 3; i * i <= this; i += 2) {
      if (this % i == 0) return false;
    }
    return true;
  }
  
  // Factorial
  int get factorial {
    if (this <= 1) return 1;
    return this * (this - 1).factorial;
  }
  
  // แปลงเป็น Thai Number Words
  String get inThai {
    const thaiNums = ['ศูนย์', 'หนึ่ง', 'สอง', 'สาม', 'สี่', 
                       'ห้า', 'หก', 'เจ็ด', 'แปด', 'เก้า'];
    if (this >= 0 && this <= 9) return thaiNums[this];
    return toString(); // สำหรับตัวเลขที่ซับซ้อนกว่า
  }
  
  // Range
  Iterable<int> to(int end) sync* {
    int step = this <= end ? 1 : -1;
    for (int i = this; step > 0 ? i <= end : i >= end; i += step) {
      yield i;
    }
  }
}

// ─────────────── Extension บน List ───────────────
extension ListExtensions<T> on List<T> {
  // แบ่งเป็น chunks
  List<List<T>> chunk(int size) {
    List<List<T>> chunks = [];
    for (int i = 0; i < length; i += size) {
      chunks.add(sublist(i, i + size > length ? length : i + size));
    }
    return chunks;
  }
  
  // สุ่ม
  T get random {
    if (isEmpty) throw StateError('List is empty');
    return this[DateTime.now().millisecondsSinceEpoch % length];
  }
  
  // Remove duplicates
  List<T> get unique => toSet().toList();
}

void main() {
  // String extensions
  print('hello world'.toTitleCase());  // Hello World
  print('test@email.com'.isEmail);     // true
  print('Hello World Flutter'.toSlug()); // hello-world-flutter
  print('hello world dart'.wordCount);  // 3
  print('Hello'.reversed);            // olleH
  print('abc'.repeat(3));             // abcabcabc
  
  print('\n');
  
  // int extensions
  print(17.isPrime);    // true
  print(5.factorial);   // 120
  print(7.inThai);      // เจ็ด
  
  for (int i in 1.to(5)) {
    print(i);  // 1, 2, 3, 4, 5
  }
  
  print('\n');
  
  // List extensions
  List<int> nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  print(nums.chunk(3));   // [[1,2,3],[4,5,6],[7,8,9],[10]]
  
  List<int> withDups = [1, 2, 2, 3, 3, 3, 4];
  print(withDups.unique);  // [1, 2, 3, 4]
}
```

---

## ขั้นตอนที่ 90: Typedef และ Function Types

```dart
// ─────────────── Typedef ───────────────
typedef Predicate<T> = bool Function(T);
typedef Transformer<T, R> = R Function(T);
typedef Comparator<T> = int Function(T, T);
typedef VoidCallback = void Function();
typedef OnError = void Function(String message);

// ─────────────── ใช้ Typedef ───────────────
List<T> filter<T>(List<T> list, Predicate<T> predicate) {
  return list.where(predicate).toList();
}

List<R> transform<T, R>(List<T> list, Transformer<T, R> transformer) {
  return list.map(transformer).toList();
}

void sortBy<T>(List<T> list, Comparator<T> comparator) {
  list.sort(comparator);
}

class EventEmitter {
  final Map<String, List<VoidCallback>> _listeners = {};
  
  void on(String event, VoidCallback callback) {
    _listeners.putIfAbsent(event, () => []).add(callback);
  }
  
  void emit(String event) {
    _listeners[event]?.forEach((callback) => callback());
  }
}

void main() {
  List<int> numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  
  // ใช้ Predicate
  Predicate<int> isEven = (n) => n % 2 == 0;
  Predicate<int> isGreaterThan5 = (n) => n > 5;
  
  print(filter(numbers, isEven));           // [2, 4, 6, 8, 10]
  print(filter(numbers, isGreaterThan5));   // [6, 7, 8, 9, 10]
  
  // ใช้ Transformer
  Transformer<int, String> toText = (n) => 'number $n';
  print(transform(numbers.take(3).toList(), toText));
  // [number 1, number 2, number 3]
  
  // ใช้ Comparator
  List<String> names = ['Charlie', 'Alice', 'Bob', 'Diana'];
  sortBy(names, (a, b) => a.compareTo(b));
  print(names);  // [Alice, Bob, Charlie, Diana]
  
  // EventEmitter
  var emitter = EventEmitter();
  emitter.on('click', () => print('คลิก!'));
  emitter.on('click', () => print('ถูกคลิกอีกครั้ง!'));
  emitter.on('hover', () => print('Hover!'));
  
  emitter.emit('click');  // คลิก! + ถูกคลิกอีกครั้ง!
  emitter.emit('hover');  // Hover!
}
```

---

## ขั้นตอนที่ 91-100: Workshop Projects

### โปรเจกต์ 1: Functional Programming Utils

```dart
// functional_utils.dart

// ─────────────── Maybe Monad ───────────────
class Maybe<T> {
  final T? _value;
  
  const Maybe._(this._value);
  
  factory Maybe.just(T value) => Maybe._(value);
  factory Maybe.nothing() => Maybe._(null);
  
  bool get hasValue => _value != null;
  T get value => _value!;
  
  Maybe<R> map<R>(R Function(T) fn) {
    if (hasValue) return Maybe.just(fn(_value as T));
    return Maybe.nothing();
  }
  
  Maybe<R> flatMap<R>(Maybe<R> Function(T) fn) {
    if (hasValue) return fn(_value as T);
    return Maybe.nothing();
  }
  
  T getOrElse(T defaultValue) => _value ?? defaultValue;
  
  @override
  String toString() => hasValue ? 'Just($_value)' : 'Nothing';
}

// ─────────────── Result Type ───────────────
sealed class Result<T> {}

class Success<T> extends Result<T> {
  final T value;
  Success(this.value);
  @override
  String toString() => 'Success($value)';
}

class Failure<T> extends Result<T> {
  final String error;
  Failure(this.error);
  @override
  String toString() => 'Failure($error)';
}

Result<int> parsePositiveInt(String s) {
  try {
    int n = int.parse(s);
    if (n <= 0) return Failure('ต้องเป็นจำนวนบวก');
    return Success(n);
  } catch (e) {
    return Failure('ไม่สามารถแปลง "$s" เป็นตัวเลขได้');
  }
}

void main() {
  // Maybe monad
  Maybe<String> userName = Maybe.just('Alice');
  Maybe<String> noUser = Maybe.nothing();
  
  print(userName.map((s) => s.toUpperCase()));  // Just(ALICE)
  print(noUser.map((s) => s.toUpperCase()));    // Nothing
  print(noUser.getOrElse('Guest'));             // Guest
  
  // Result type
  List<String> inputs = ['42', '-5', 'abc', '100'];
  
  for (String input in inputs) {
    Result<int> result = parsePositiveInt(input);
    switch (result) {
      case Success(:var value):
        print('✅ "$input" → $value');
      case Failure(:var error):
        print('❌ "$input" → $error');
    }
  }
}
```

### โปรเจกต์ 2: Task Pipeline

```dart
// task_pipeline.dart

typedef Task<T> = Future<T> Function();
typedef Middleware<T> = Future<T> Function(T, Future<T> Function(T));

class Pipeline<T> {
  final List<Middleware<T>> _middlewares = [];
  
  Pipeline<T> use(Middleware<T> middleware) {
    _middlewares.add(middleware);
    return this;
  }
  
  Future<T> execute(T initialValue, Future<T> Function(T) handler) async {
    Future<T> Function(T) chain = handler;
    
    for (var middleware in _middlewares.reversed) {
      final next = chain;
      final m = middleware;
      chain = (value) => m(value, next);
    }
    
    return chain(initialValue);
  }
}

void main() async {
  Pipeline<Map<String, dynamic>> pipeline = Pipeline();
  
  // Logging middleware
  pipeline.use((data, next) async {
    print('📥 Input: ${data['value']}');
    var result = await next(data);
    print('📤 Output: ${result['value']}');
    return result;
  });
  
  // Validation middleware
  pipeline.use((data, next) async {
    int value = data['value'] as int;
    if (value < 0) throw ArgumentError('ค่าต้องไม่ติดลบ');
    return next(data);
  });
  
  // Transform middleware
  pipeline.use((data, next) async {
    data['value'] = (data['value'] as int) * 2;
    return next(data);
  });
  
  // Handler
  var result = await pipeline.execute(
    {'value': 5},
    (data) async => {...data, 'processed': true},
  );
  
  print('ผลลัพธ์: $result');
}
```

---

## ขั้นตอนที่ 101-110: สรุปและ Challenge

### Challenge 1: Functional Calculator

```dart
void main() {
  // สร้าง calculator แบบ functional
  num Function(num) add(num n) => (x) => x + n;
  num Function(num) multiply(num n) => (x) => x * n;
  num Function(num) divide(num n) => (x) => x / n;
  num Function(num) power(num n) => (x) => x * x;  // simplified

  // compose functions
  num Function(num) compose(List<num Function(num)> fns) {
    return (x) => fns.fold(x, (acc, fn) => fn(acc));
  }
  
  // ตัวอย่าง: ((5 + 3) * 2) / 4
  var calc = compose([add(3), multiply(2), divide(4)]);
  print('((5+3)*2)/4 = ${calc(5)}');  // 4.0
}
```

### Challenge 2: Event System

```dart
class EventBus {
  final Map<Type, List<Function>> _handlers = {};
  
  void on<T>(void Function(T) handler) {
    _handlers.putIfAbsent(T, () => []).add(handler);
  }
  
  void emit<T>(T event) {
    _handlers[T]?.forEach((handler) => handler(event));
  }
  
  void off<T>(void Function(T) handler) {
    _handlers[T]?.remove(handler);
  }
}

class UserCreatedEvent {
  final String name;
  final String email;
  UserCreatedEvent(this.name, this.email);
}

class OrderPlacedEvent {
  final String userId;
  final double amount;
  OrderPlacedEvent(this.userId, this.amount);
}

void main() {
  EventBus bus = EventBus();
  
  bus.on<UserCreatedEvent>((event) {
    print('👤 User created: ${event.name} (${event.email})');
  });
  
  bus.on<UserCreatedEvent>((event) {
    print('📧 ส่ง welcome email ไปที่ ${event.email}');
  });
  
  bus.on<OrderPlacedEvent>((event) {
    print('🛒 Order placed: user=${event.userId}, amount=฿${event.amount}');
  });
  
  // emit events
  bus.emit(UserCreatedEvent('Alice', 'alice@example.com'));
  bus.emit(UserCreatedEvent('Bob', 'bob@example.com'));
  bus.emit(OrderPlacedEvent('alice', 299.0));
}
```

### สรุป Part 04

```
✅ ขั้นตอนที่ 81: Function พื้นฐาน
✅ ขั้นตอนที่ 82: Parameters ชนิดต่างๆ
✅ ขั้นตอนที่ 83: Default Parameter Values
✅ ขั้นตอนที่ 84: Functions เป็น First-Class Objects
✅ ขั้นตอนที่ 85: Anonymous Functions และ Closures
✅ ขั้นตอนที่ 86: Higher-Order Functions
✅ ขั้นตอนที่ 87: Scope และ Variable Visibility
✅ ขั้นตอนที่ 88: Recursion
✅ ขั้นตอนที่ 89: Extension Methods
✅ ขั้นตอนที่ 90: Typedef และ Function Types
✅ ขั้นตอนที่ 91-100: Workshop Projects
✅ ขั้นตอนที่ 101-110: Challenge
```

---

**← [Part 03 - Control Flow](part-03-control-flow.md)**

**ต่อไป: [Part 05 - Collections: List, Set, Map →](part-05-collections.md)**
