# Part 03: การควบคุมการไหลของโปรแกรม (Control Flow)
## ขั้นตอนที่ 51-80

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ if/else statements ได้อย่างมีประสิทธิภาพ
- ใช้ switch/case และ switch expressions (Dart 3.0+)
- ใช้ for loops, while loops, do-while loops
- เข้าใจ break, continue, labels
- ใช้ Pattern Matching ใน control flow
- สร้างโปรแกรมแบบ conditional และ iterative

---

## ขั้นตอนที่ 51: if Statement พื้นฐาน

```dart
void main() {
  int score = 75;
  
  // ─────────────── if ───────────────
  if (score >= 60) {
    print('ผ่าน!');
  }
  
  // ─────────────── if-else ───────────────
  if (score >= 60) {
    print('ผ่าน!');
  } else {
    print('ไม่ผ่าน');
  }
  
  // ─────────────── if-else if-else ───────────────
  String grade;
  
  if (score >= 90) {
    grade = 'A';
  } else if (score >= 80) {
    grade = 'B';
  } else if (score >= 70) {
    grade = 'C';
  } else if (score >= 60) {
    grade = 'D';
  } else {
    grade = 'F';
  }
  
  print('เกรด: $grade');
  
  // ─────────────── Nested if ───────────────
  bool isStudent = true;
  bool hasScholarship = true;
  double tuitionFee = 30000;
  
  if (isStudent) {
    if (hasScholarship) {
      tuitionFee *= 0.5;  // ลด 50%
      print('ค่าเล่าเรียนหลังทุน: $tuitionFee บาท');
    } else {
      print('ค่าเล่าเรียนปกติ: $tuitionFee บาท');
    }
  } else {
    print('ไม่ใช่นักเรียน');
  }
}
```

---

## ขั้นตอนที่ 52: Conditional Expressions ขั้นสูง

```dart
void main() {
  // ─────────────── Ternary Operator ───────────────
  int age = 20;
  String status = age >= 18 ? 'ผู้ใหญ่' : 'เด็ก';
  print(status);
  
  // ─────────────── Null-Coalescing (??) ───────────────
  String? name;
  String displayName = name ?? 'ไม่ระบุชื่อ';
  print(displayName);
  
  // ─────────────── Null-conditional (?.) ───────────────
  String? text = null;
  int? length = text?.length;
  print(length);  // null
  
  // ─────────────── If as expression (ใน Flutter) ───────────────
  bool isLoggedIn = true;
  // ใช้ใน Widget tree
  // Widget button = isLoggedIn 
  //     ? LogoutButton() 
  //     : LoginButton();
  
  // ─────────────── Short-circuit evaluation ───────────────
  int? nullableValue;
  
  // && short-circuits: ถ้า false ด้านซ้าย จะไม่ประเมินด้านขวา
  if (nullableValue != null && nullableValue > 0) {
    print('มีค่าบวก');
  }
  
  // || short-circuits: ถ้า true ด้านซ้าย จะไม่ประเมินด้านขวา
  String? savedName;
  String finalName = savedName ?? (savedName = 'Default');
  print(finalName);
  
  // ─────────────── Cascade in Conditionals ───────────────
  bool debug = true;
  
  StringBuffer buffer = StringBuffer();
  buffer.write('Hello');
  if (debug) buffer.write(' [DEBUG]');
  print(buffer.toString());
}
```

---

## ขั้นตอนที่ 53: switch Statement แบบ Classic

```dart
void main() {
  // ─────────────── switch-case พื้นฐาน ───────────────
  int day = 3;
  String dayName;
  
  switch (day) {
    case 1:
      dayName = 'จันทร์';
      break;
    case 2:
      dayName = 'อังคาร';
      break;
    case 3:
      dayName = 'พุธ';
      break;
    case 4:
      dayName = 'พฤหัสบดี';
      break;
    case 5:
      dayName = 'ศุกร์';
      break;
    case 6:
      dayName = 'เสาร์';
      break;
    case 7:
      dayName = 'อาทิตย์';
      break;
    default:
      dayName = 'ไม่ถูกต้อง';
  }
  
  print('วัน$dayName');
  
  // ─────────────── Fallthrough (ไม่มี break) ───────────────
  String season;
  int month = 12;
  
  switch (month) {
    case 12:
    case 1:
    case 2:
      season = 'ฤดูหนาว';
      break;
    case 3:
    case 4:
    case 5:
      season = 'ฤดูร้อน';
      break;
    case 6:
    case 7:
    case 8:
      season = 'ฤดูฝน';
      break;
    case 9:
    case 10:
    case 11:
      season = 'ฤดูใบไม้ร่วง';
      break;
    default:
      season = 'ไม่ถูกต้อง';
  }
  
  print('เดือน $month: $season');
  
  // ─────────────── switch กับ String ───────────────
  String command = 'play';
  
  switch (command) {
    case 'play':
      print('▶ กำลังเล่น...');
      break;
    case 'pause':
      print('⏸ หยุดชั่วคราว');
      break;
    case 'stop':
      print('⏹ หยุด');
      break;
    default:
      print('❓ คำสั่งไม่รู้จัก: $command');
  }
}
```

---

## ขั้นตอนที่ 54: switch Expressions (Dart 3.0+)

```dart
void main() {
  // ─────────────── switch expression ───────────────
  // switch expression คืนค่า ไม่เหมือน switch statement
  
  int day = 3;
  String dayName = switch (day) {
    1 => 'จันทร์',
    2 => 'อังคาร',
    3 => 'พุธ',
    4 => 'พฤหัสบดี',
    5 => 'ศุกร์',
    6 => 'เสาร์',
    7 => 'อาทิตย์',
    _ => 'ไม่ถูกต้อง',  // default
  };
  print('วัน$dayName');
  
  // ─────────────── switch expression กับ Guard Clause ───────────────
  int score = 75;
  String grade = switch (score) {
    int s when s >= 90 => 'A',
    int s when s >= 80 => 'B',
    int s when s >= 70 => 'C',
    int s when s >= 60 => 'D',
    _ => 'F',
  };
  print('เกรด: $grade');
  
  // ─────────────── switch กับ Types ───────────────
  Object value = 3.14;
  String type = switch (value) {
    int() => 'จำนวนเต็ม: $value',
    double() => 'จำนวนทศนิยม: $value',
    String() => 'ข้อความ: $value',
    bool() => 'ค่าตรรกะ: $value',
    _ => 'ชนิดอื่น',
  };
  print(type);
  
  // ─────────────── switch กับ Records ───────────────
  (int, int) point = (3, -2);
  String quadrant = switch (point) {
    (int x, int y) when x > 0 && y > 0 => 'จตุภาคที่ 1',
    (int x, int y) when x < 0 && y > 0 => 'จตุภาคที่ 2',
    (int x, int y) when x < 0 && y < 0 => 'จตุภาคที่ 3',
    (int x, int y) when x > 0 && y < 0 => 'จตุภาคที่ 4',
    (0, _) => 'แกน Y',
    (_, 0) => 'แกน X',
    _ => 'จุดกำเนิด',
  };
  print('จุด $point อยู่ใน $quadrant');
  
  // ─────────────── Exhaustive switch ───────────────
  // Dart บังคับให้ครอบคลุมทุก case กับ sealed classes
  // (จะเรียนเพิ่มใน Part เรื่อง OOP)
}
```

---

## ขั้นตอนที่ 55: for Loop พื้นฐาน

```dart
void main() {
  // ─────────────── for loop พื้นฐาน ───────────────
  for (int i = 0; i < 5; i++) {
    print('ครั้งที่ $i');
  }
  
  // ─────────────── นับถอยหลัง ───────────────
  for (int i = 10; i >= 0; i--) {
    print(i);
  }
  print('หมดเวลา!');
  
  // ─────────────── ขยับทีละ 2 ───────────────
  print('\nเลขคู่ 0-20:');
  for (int i = 0; i <= 20; i += 2) {
    print(i);
  }
  
  // ─────────────── Nested for loops ───────────────
  print('\nตารางคูณ (3x3):');
  for (int i = 1; i <= 3; i++) {
    String row = '';
    for (int j = 1; j <= 3; j++) {
      row += '${i * j}'.padLeft(4);
    }
    print(row);
  }
  
  // ─────────────── for-in loop ───────────────
  List<String> fruits = ['แอปเปิ้ล', 'กล้วย', 'มะม่วง', 'ส้ม'];
  
  print('\nผลไม้:');
  for (String fruit in fruits) {
    print('  🍎 $fruit');
  }
  
  // ─────────────── for-in กับ index ───────────────
  print('\nผลไม้พร้อมหมายเลข:');
  for (int i = 0; i < fruits.length; i++) {
    print('  ${i + 1}. ${fruits[i]}');
  }
  
  // ─────────────── forEach method ───────────────
  fruits.forEach((fruit) {
    print('  🍊 $fruit');
  });
  
  // ─────────────── forEach กับ index ───────────────
  fruits.asMap().forEach((index, fruit) {
    print('  ${index + 1}. $fruit');
  });
}
```

---

## ขั้นตอนที่ 56: while และ do-while Loops

```dart
void main() {
  // ─────────────── while loop ───────────────
  // ตรวจสอบเงื่อนไขก่อนทำงาน
  int count = 0;
  
  while (count < 5) {
    print('count = $count');
    count++;
  }
  
  // ─────────────── while กับ User Input Simulation ───────────────
  // ในโปรแกรมจริง มักใช้กับ I/O
  List<int> numbers = [3, 7, 2, 9, 1, 8, 4];
  int index = 0;
  int sum = 0;
  
  while (index < numbers.length) {
    sum += numbers[index];
    index++;
  }
  print('ผลรวม: $sum');
  
  // ─────────────── do-while loop ───────────────
  // ทำงานก่อน แล้วค่อยตรวจสอบเงื่อนไข
  // ทำงานอย่างน้อย 1 ครั้งเสมอ
  int n = 1;
  
  do {
    print('n = $n');
    n++;
  } while (n <= 5);
  
  // ─────────────── ตัวอย่าง: หาตัวเลข Fibonacci ───────────────
  print('\nFibonacci numbers < 100:');
  int a = 0, b = 1;
  
  while (a < 100) {
    print(a);
    int temp = a + b;
    a = b;
    b = temp;
  }
  
  // ─────────────── ตัวอย่าง: Newton's Method หา Square Root ───────────────
  double findSqrt(double n) {
    double x = n;
    double epsilon = 0.0001;
    
    while ((x * x - n).abs() > epsilon) {
      x = (x + n / x) / 2;
    }
    
    return x;
  }
  
  print('\nSquare roots:');
  for (int i in [4, 9, 16, 25, 2, 3]) {
    print('√$i ≈ ${findSqrt(i.toDouble()).toStringAsFixed(6)}');
  }
}
```

---

## ขั้นตอนที่ 57: break และ continue

```dart
void main() {
  // ─────────────── break ───────────────
  // หยุด loop ทันที
  
  print('ค้นหาตัวเลขที่หารด้วย 7 ลงตัว ตั้งแต่ 1-100:');
  for (int i = 1; i <= 100; i++) {
    if (i % 7 == 0) {
      print('พบ: $i');
      break;  // หยุดหลังพบครั้งแรก
    }
  }
  
  // ─────────────── continue ───────────────
  // ข้ามการทำงานในรอบนี้ไปรอบถัดไป
  
  print('\nเลขคี่ 1-20:');
  for (int i = 1; i <= 20; i++) {
    if (i % 2 == 0) continue;  // ข้ามเลขคู่
    print(i);
  }
  
  // ─────────────── break ใน while ───────────────
  print('\nหาเฉพาะตัวเลขแรกที่มากกว่า 50 และหารด้วย 3 ลงตัว:');
  int n = 51;
  while (true) {
    if (n % 3 == 0) {
      print('พบ: $n');
      break;
    }
    n++;
  }
  
  // ─────────────── break ใน nested loop ───────────────
  // break จะหยุดเฉพาะ loop ชั้นใน
  print('\nค้นหาใน nested loop:');
  bool found = false;
  
  outer:
  for (int i = 0; i < 5; i++) {
    for (int j = 0; j < 5; j++) {
      if (i * j > 10) {
        print('พบ: i=$i, j=$j, product=${i * j}');
        found = true;
        break outer;  // ใช้ label ออกจาก outer loop
      }
    }
  }
  
  // ─────────────── Labels ───────────────
  print('\nการใช้ Labels:');
  
  search:
  for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
      if (i == 1 && j == 1) {
        print('พบที่ i=$i, j=$j');
        break search;  // ออกจาก outer loop
      }
      print('i=$i, j=$j');
    }
  }
}
```

---

## ขั้นตอนที่ 58: Iterators และ Iterables

```dart
void main() {
  // ─────────────── Iterator ───────────────
  List<int> numbers = [1, 2, 3, 4, 5];
  
  // ใช้ iterator โดยตรง
  Iterator<int> it = numbers.iterator;
  while (it.moveNext()) {
    print(it.current);
  }
  
  // ─────────────── Iterable methods ───────────────
  List<int> nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  
  // map: แปลงทุก element
  Iterable<int> doubled = nums.map((n) => n * 2);
  print('Doubled: $doubled');
  
  // where: กรอง
  Iterable<int> evens = nums.where((n) => n % 2 == 0);
  print('Evens: $evens');
  
  // reduce: รวมเป็นค่าเดียว
  int sum = nums.reduce((a, b) => a + b);
  print('Sum: $sum');
  
  // fold: เหมือน reduce แต่มี initial value
  int sumWithFold = nums.fold(0, (acc, n) => acc + n);
  print('Sum with fold: $sumWithFold');
  
  // any: มีอย่างน้อยหนึ่งที่ตรงเงื่อนไข
  bool hasEven = nums.any((n) => n % 2 == 0);
  print('Has even: $hasEven');
  
  // every: ทุกตัวตรงเงื่อนไข
  bool allPositive = nums.every((n) => n > 0);
  print('All positive: $allPositive');
  
  // take: เอา n ตัวแรก
  print('First 3: ${nums.take(3).toList()}');
  
  // skip: ข้าม n ตัวแรก
  print('Skip 7: ${nums.skip(7).toList()}');
  
  // takeWhile: เอาจนกว่าเงื่อนไขเป็น false
  print('Take while < 5: ${nums.takeWhile((n) => n < 5).toList()}');
  
  // skipWhile: ข้ามจนกว่าเงื่อนไขเป็น false
  print('Skip while < 5: ${nums.skipWhile((n) => n < 5).toList()}');
  
  // expand (flatMap): แปลงแต่ละ element เป็น Iterable แล้ว flatten
  List<List<int>> nested = [[1, 2], [3, 4], [5, 6]];
  Iterable<int> flat = nested.expand((list) => list);
  print('Flattened: $flat');
  
  // firstWhere / lastWhere: หาค่าที่ตรงเงื่อนไข
  int firstEven = nums.firstWhere((n) => n % 2 == 0);
  int lastEven = nums.lastWhere((n) => n % 2 == 0);
  print('First even: $firstEven, Last even: $lastEven');
  
  // ─────────────── Chaining ───────────────
  // รวม operations หลายอัน
  List<int> result = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
      .where((n) => n % 2 == 0)      // เอาเลขคู่
      .map((n) => n * n)              // ยกกำลังสอง
      .where((n) => n > 20)          // เฉพาะที่ > 20
      .toList();
  
  print('Result: $result');  // [36, 64, 100]
}
```

---

## ขั้นตอนที่ 59: Generator Functions

```dart
// ─────────────── sync* Generator ───────────────
Iterable<int> count(int start, int end) sync* {
  for (int i = start; i <= end; i++) {
    yield i;  // ส่งค่าและ pause
  }
}

// Generator แบบ recursive
Iterable<int> fibonacci() sync* {
  int a = 0, b = 1;
  while (true) {
    yield a;
    int temp = a + b;
    a = b;
    b = temp;
  }
}

// yield* (yield from another Iterable)
Iterable<int> countBoth() sync* {
  yield* count(1, 5);   // yield ค่าทั้งหมดจาก count(1,5)
  yield* count(10, 15);
}

// ─────────────── async* Generator (Stream) ───────────────
Stream<int> asyncCount(int start, int end) async* {
  for (int i = start; i <= end; i++) {
    await Future.delayed(Duration(milliseconds: 100));
    yield i;
  }
}

void main() async {
  // ใช้ sync* generator
  print('นับ 1-5:');
  for (int n in count(1, 5)) {
    print(n);
  }
  
  // เอา Fibonacci 10 ตัวแรก
  print('\nFibonacci 10 ตัวแรก:');
  for (int n in fibonacci().take(10)) {
    print(n);
  }
  
  print('\nนับ 1-5 และ 10-15:');
  for (int n in countBoth()) {
    print(n);
  }
  
  // ใช้ async* generator (Stream)
  print('\nAsync count:');
  await for (int n in asyncCount(1, 5)) {
    print('Got: $n');
  }
}
```

---

## ขั้นตอนที่ 60: Pattern Matching ขั้นสูงใน Control Flow

```dart
void main() {
  // ─────────────── if-case ───────────────
  Object value = [1, 2, 3];
  
  if (value case List<int> numbers when numbers.length > 2) {
    print('List ที่มีมากกว่า 2 ตัว: $numbers');
  }
  
  // ─────────────── switch กับ sealed classes ───────────────
  // (ดูตัวอย่างเพิ่มเติมใน Part OOP)
  
  // ─────────────── Destructuring ใน for loop ───────────────
  List<(String, int)> scores = [
    ('Alice', 95),
    ('Bob', 72),
    ('Charlie', 88),
    ('Diana', 60),
  ];
  
  print('\nผลการสอบ:');
  for (var (name, score) in scores) {
    String grade = switch (score) {
      int s when s >= 90 => 'A',
      int s when s >= 80 => 'B',
      int s when s >= 70 => 'C',
      _ => 'D',
    };
    print('$name: $score คะแนน (เกรด $grade)');
  }
  
  // ─────────────── Map destructuring ───────────────
  List<Map<String, dynamic>> users = [
    {'name': 'Alice', 'age': 25, 'active': true},
    {'name': 'Bob', 'age': 17, 'active': false},
    {'name': 'Charlie', 'age': 30, 'active': true},
  ];
  
  print('\nผู้ใช้ที่ active และอายุ >= 18:');
  for (var user in users) {
    if (user case {'name': String name, 'age': int age, 'active': true}
        when age >= 18) {
      print('  $name (อายุ $age)');
    }
  }
}
```

---

## ขั้นตอนที่ 61-65: โปรแกรมตัวอย่าง - เกมทายตัวเลข

```dart
import 'dart:math';

void main() {
  playGuessGame();
}

void playGuessGame() {
  final Random random = Random();
  int secretNumber = random.nextInt(100) + 1;  // 1-100
  int attempts = 0;
  int maxAttempts = 7;
  bool won = false;
  
  print('🎮 เกมทายตัวเลข 1-100');
  print('คุณมี $maxAttempts ครั้งในการทาย\n');
  
  // จำลองการเดา (ในโปรแกรมจริงจะรับ input จากผู้ใช้)
  List<int> guesses = [50, 75, 60, 65, secretNumber]; // จำลอง
  
  for (int guess in guesses) {
    if (attempts >= maxAttempts) {
      print('❌ หมดครั้งแล้ว!');
      break;
    }
    
    attempts++;
    print('ครั้งที่ $attempts: ทาย $guess');
    
    if (guess == secretNumber) {
      won = true;
      print('🎉 ถูกต้อง! ตอบถูกใน $attempts ครั้ง');
      break;
    } else if (guess < secretNumber) {
      print('   ⬆ มากกว่านี้!');
    } else {
      print('   ⬇ น้อยกว่านี้!');
    }
  }
  
  if (!won) {
    print('😞 คำตอบคือ $secretNumber');
  }
  
  // แสดงสถิติ
  print('\n📊 สถิติ:');
  print('  ตัวเลขที่ซ่อน: $secretNumber');
  print('  จำนวนครั้งที่ทาย: $attempts/$maxAttempts');
  print('  ผล: ${won ? "ชนะ 🏆" : "แพ้ 😢"}');
}
```

---

## ขั้นตอนที่ 66-70: โปรแกรมตัวอย่าง - Menu System

```dart
void main() {
  runMenuSystem();
}

void runMenuSystem() {
  print('🏪 ระบบ POS (Point of Sale)');
  print('================================\n');
  
  List<Map<String, dynamic>> menu = [
    {'name': 'กาแฟดำ', 'price': 35.0, 'category': 'เครื่องดื่ม'},
    {'name': 'ลาเต้', 'price': 55.0, 'category': 'เครื่องดื่ม'},
    {'name': 'ชาเขียว', 'price': 45.0, 'category': 'เครื่องดื่ม'},
    {'name': 'แซนวิช', 'price': 75.0, 'category': 'อาหาร'},
    {'name': 'เค้ก', 'price': 65.0, 'category': 'ของหวาน'},
  ];
  
  // แสดงเมนู
  printMenu(menu);
  
  // จำลองการสั่งออเดอร์
  List<int> order = [1, 3, 1, 5]; // index ของเมนู (เริ่มที่ 1)
  List<Map<String, dynamic>> cart = [];
  
  print('\n📝 รายการสั่ง:');
  for (int itemNum in order) {
    if (itemNum < 1 || itemNum > menu.length) {
      print('  ❌ หมายเลข $itemNum ไม่มีในเมนู');
      continue;
    }
    
    var item = menu[itemNum - 1];
    
    // ตรวจสอบว่ามีในตะกร้าแล้วหรือไม่
    int existingIndex = cart.indexWhere((i) => i['name'] == item['name']);
    
    if (existingIndex >= 0) {
      cart[existingIndex]['qty'] = (cart[existingIndex]['qty'] as int) + 1;
    } else {
      cart.add({...item, 'qty': 1});
    }
    
    print('  + ${item['name']}');
  }
  
  // แสดงใบเสร็จ
  print('\n');
  printReceipt(cart);
}

void printMenu(List<Map<String, dynamic>> menu) {
  String currentCategory = '';
  
  for (int i = 0; i < menu.length; i++) {
    var item = menu[i];
    
    if (item['category'] != currentCategory) {
      currentCategory = item['category'] as String;
      print('\n📂 $currentCategory');
      print('─' * 35);
    }
    
    String price = (item['price'] as double).toStringAsFixed(0);
    print('  ${(i + 1).toString().padLeft(2)}. ${item['name'].padRight(15)} ฿$price');
  }
}

void printReceipt(List<Map<String, dynamic>> cart) {
  if (cart.isEmpty) {
    print('ตะกร้าว่าง');
    return;
  }
  
  print('🧾 ใบเสร็จ');
  print('═' * 40);
  
  double subtotal = 0;
  
  for (var item in cart) {
    double itemTotal = (item['price'] as double) * (item['qty'] as int);
    subtotal += itemTotal;
    
    String line = '${item['name']} x${item['qty']}';
    String price = '฿${itemTotal.toStringAsFixed(0)}';
    print('${line.padRight(30)}${price.padLeft(8)}');
  }
  
  double vat = subtotal * 0.07;
  double total = subtotal + vat;
  
  print('─' * 40);
  print('${'ราคาสินค้า'.padRight(30)}${'฿${subtotal.toStringAsFixed(2)}'.padLeft(8)}');
  print('${'ภาษีมูลค่าเพิ่ม 7%'.padRight(30)}${'฿${vat.toStringAsFixed(2)}'.padLeft(8)}');
  print('═' * 40);
  print('${'รวมทั้งหมด'.padRight(30)}${'฿${total.toStringAsFixed(2)}'.padLeft(8)}');
  print('═' * 40);
  print('\nขอบคุณที่ใช้บริการ 🙏');
}
```

---

## ขั้นตอนที่ 71-75: โปรแกรมตัวอย่าง - Grade Calculator

```dart
void main() {
  List<Map<String, dynamic>> students = [
    {
      'name': 'สมชาย ใจดี',
      'scores': {
        'คณิต': 85.0,
        'ภาษาไทย': 90.0,
        'อังกฤษ': 75.0,
        'วิทยาศาสตร์': 88.0,
        'สังคมศึกษา': 72.0,
      }
    },
    {
      'name': 'สมหญิง รักเรียน',
      'scores': {
        'คณิต': 92.0,
        'ภาษาไทย': 88.0,
        'อังกฤษ': 95.0,
        'วิทยาศาสตร์': 80.0,
        'สังคมศึกษา': 85.0,
      }
    },
    {
      'name': 'มานะ ตั้งใจ',
      'scores': {
        'คณิต': 55.0,
        'ภาษาไทย': 60.0,
        'อังกฤษ': 45.0,
        'วิทยาศาสตร์': 70.0,
        'สังคมศึกษา': 58.0,
      }
    },
  ];
  
  print('📊 รายงานผลการเรียน');
  print('═' * 60);
  
  for (var student in students) {
    print('\n👤 ${student['name']}');
    print('─' * 50);
    
    Map<String, double> scores = 
        Map<String, double>.from(student['scores'] as Map);
    
    double total = 0;
    int passed = 0;
    int failed = 0;
    
    for (var entry in scores.entries) {
      double score = entry.value;
      String status = score >= 60 ? '✅ ผ่าน' : '❌ ไม่ผ่าน';
      String grade = getGrade(score);
      
      total += score;
      if (score >= 60) {
        passed++;
      } else {
        failed++;
      }
      
      print('  ${entry.key.padRight(15)}: '
            '${score.toStringAsFixed(1).padLeft(6)} '
            '(${grade}) $status');
    }
    
    double average = total / scores.length;
    String overallGrade = getGrade(average);
    String overallStatus = average >= 60 ? 'ผ่าน' : 'ไม่ผ่าน';
    
    print('─' * 50);
    print('  คะแนนเฉลี่ย: ${average.toStringAsFixed(2)} ($overallGrade)');
    print('  ผ่าน: $passed วิชา | ไม่ผ่าน: $failed วิชา');
    print('  สถานะ: $overallStatus');
  }
  
  print('\n' + '═' * 60);
  
  // อันดับในห้อง
  print('\n🏆 อันดับในห้อง:');
  
  List<Map<String, dynamic>> ranking = students.map((s) {
    Map<String, double> scores = Map<String, double>.from(s['scores'] as Map);
    double avg = scores.values.fold(0.0, (sum, s) => sum + s) / scores.length;
    return {'name': s['name'], 'average': avg};
  }).toList();
  
  ranking.sort((a, b) => 
      (b['average'] as double).compareTo(a['average'] as double));
  
  for (int i = 0; i < ranking.length; i++) {
    String medal = i == 0 ? '🥇' : i == 1 ? '🥈' : '🥉';
    print('  $medal อันดับ ${i + 1}: ${ranking[i]['name']} '
          '(${(ranking[i]['average'] as double).toStringAsFixed(2)})');
  }
}

String getGrade(double score) {
  return switch (score) {
    double s when s >= 90 => 'A',
    double s when s >= 80 => 'B',
    double s when s >= 70 => 'C',
    double s when s >= 60 => 'D',
    _ => 'F',
  };
}
```

---

## ขั้นตอนที่ 76-80: สรุปและ Exercise

### Exercise 1: FizzBuzz ขั้นสูง

```dart
void fizzBuzzAdvanced(int start, int end) {
  for (int i = start; i <= end; i++) {
    String result = '';
    
    if (i % 3 == 0) result += 'Fizz';
    if (i % 5 == 0) result += 'Buzz';
    if (i % 7 == 0) result += 'Bazz';
    
    print(result.isEmpty ? i : result);
  }
}

void main() {
  fizzBuzzAdvanced(1, 50);
}
```

### Exercise 2: Pattern Printer

```dart
void main() {
  // สร้าง patterns ต่างๆ
  printTriangle(5);
  print('');
  printDiamond(5);
  print('');
  printChessboard(6);
}

void printTriangle(int rows) {
  print('Triangle:');
  for (int i = 1; i <= rows; i++) {
    String row = ' ' * (rows - i) + '*' * (2 * i - 1);
    print(row);
  }
}

void printDiamond(int rows) {
  print('Diamond:');
  // บน
  for (int i = 1; i <= rows; i++) {
    print(' ' * (rows - i) + '*' * (2 * i - 1));
  }
  // ล่าง
  for (int i = rows - 1; i >= 1; i--) {
    print(' ' * (rows - i) + '*' * (2 * i - 1));
  }
}

void printChessboard(int size) {
  print('Chessboard:');
  for (int row = 0; row < size; row++) {
    String line = '';
    for (int col = 0; col < size; col++) {
      line += (row + col) % 2 == 0 ? '██' : '  ';
    }
    print(line);
  }
}
```

### Exercise 3: Prime Numbers

```dart
bool isPrime(int n) {
  if (n < 2) return false;
  if (n == 2) return true;
  if (n % 2 == 0) return false;
  
  for (int i = 3; i * i <= n; i += 2) {
    if (n % i == 0) return false;
  }
  
  return true;
}

List<int> sieveOfEratosthenes(int limit) {
  List<bool> isPrime = List.filled(limit + 1, true);
  isPrime[0] = isPrime[1] = false;
  
  for (int i = 2; i * i <= limit; i++) {
    if (isPrime[i]) {
      for (int j = i * i; j <= limit; j += i) {
        isPrime[j] = false;
      }
    }
  }
  
  List<int> primes = [];
  for (int i = 2; i <= limit; i++) {
    if (isPrime[i]) primes.add(i);
  }
  
  return primes;
}

void main() {
  // ตรวจสอบเฉพาะ
  print('ตัวเลขเฉพาะตรวจสอบ:');
  for (int n in [2, 3, 4, 5, 17, 25, 97, 100]) {
    print('  $n: ${isPrime(n) ? "เฉพาะ" : "ไม่ใช่เฉพาะ"}');
  }
  
  // Sieve of Eratosthenes
  print('\nตัวเลขเฉพาะ 1-100:');
  List<int> primes = sieveOfEratosthenes(100);
  print(primes);
  print('จำนวน: ${primes.length} ตัว');
}
```

### สรุป Part 03

```
✅ ขั้นตอนที่ 51: if Statement พื้นฐาน
✅ ขั้นตอนที่ 52: Conditional Expressions ขั้นสูง
✅ ขั้นตอนที่ 53: switch Statement แบบ Classic
✅ ขั้นตอนที่ 54: switch Expressions (Dart 3.0+)
✅ ขั้นตอนที่ 55: for Loop พื้นฐาน
✅ ขั้นตอนที่ 56: while และ do-while Loops
✅ ขั้นตอนที่ 57: break และ continue
✅ ขั้นตอนที่ 58: Iterators และ Iterables
✅ ขั้นตอนที่ 59: Generator Functions
✅ ขั้นตอนที่ 60: Pattern Matching ขั้นสูงใน Control Flow
✅ ขั้นตอนที่ 61-65: โปรแกรมตัวอย่าง - เกมทายตัวเลข
✅ ขั้นตอนที่ 66-70: โปรแกรมตัวอย่าง - Menu System
✅ ขั้นตอนที่ 71-75: โปรแกรมตัวอย่าง - Grade Calculator
✅ ขั้นตอนที่ 76-80: สรุปและ Exercise
```

---

**← [Part 02 - ตัวแปร ชนิดข้อมูล และ Operators](part-02-variables-types-operators.md)**

**ต่อไป: [Part 04 - Functions และ Scope →](part-04-functions-and-scope.md)**
