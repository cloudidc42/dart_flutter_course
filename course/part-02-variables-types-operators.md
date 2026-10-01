# Part 02: ตัวแปร ชนิดข้อมูล และ Operators
## ขั้นตอนที่ 21-50

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจการประกาศตัวแปรใน Dart
- รู้จักชนิดข้อมูลทั้งหมดใน Dart
- เข้าใจ Null Safety ซึ่งเป็นจุดเด่นของ Dart
- ใช้ Operators ต่างๆ ได้อย่างชำนาญ
- เข้าใจ Type Conversion และ Type Checking
- ใช้ String interpolation และ multiline strings

---

## ขั้นตอนที่ 21: การประกาศตัวแปร (Variable Declaration)

```dart
void main() {
  // ─────────────── วิธีที่ 1: ระบุชนิดข้อมูลชัดเจน ───────────────
  int age = 25;
  double salary = 50000.50;
  String name = 'สมชาย';
  bool isStudent = true;

  // ─────────────── วิธีที่ 2: ใช้ var (Dart อนุมานชนิดเอง) ───────────────
  var city = 'กรุงเทพ';        // Dart รู้ว่าเป็น String
  var population = 1000000;    // Dart รู้ว่าเป็น int
  var temperature = 36.5;      // Dart รู้ว่าเป็น double
  var isCapital = true;        // Dart รู้ว่าเป็น bool

  // ─────────────── วิธีที่ 3: ใช้ dynamic (ยืดหยุ่นแต่ไม่ปลอดภัย) ───────────────
  dynamic anything = 'ข้อความ';
  anything = 42;              // เปลี่ยนชนิดข้อมูลได้
  anything = true;            // เปลี่ยนได้อีก

  // ─────────────── วิธีที่ 4: ใช้ Object (Base class ทุกอย่าง) ───────────────
  Object something = 'ข้อความ';
  something = 100;
  something = [1, 2, 3];

  // แสดงผล
  print('อายุ: $age');
  print('เงินเดือน: $salary');
  print('ชื่อ: $name');
  print('เมือง: $city');
  print('dynamic: $anything');
}
```

---

## ขั้นตอนที่ 22: ค่าคงที่ (Constants)

```dart
void main() {
  // ─────────────── const: Compile-time constant ───────────────
  // ค่าต้องรู้ตอน Compile
  const double pi = 3.14159;
  const String appName = 'My Flutter App';
  const int maxScore = 100;

  // ─────────────── final: Runtime constant ───────────────
  // ค่าถูกกำหนดครั้งเดียวตอน Runtime
  final DateTime now = DateTime.now();
  final String greeting = 'สวัสดี $appName';

  // ความแตกต่าง const vs final
  // const: ต้องรู้ค่าตอน Compile → เร็วกว่า
  // final: รู้ค่าตอน Runtime → ยืดหยุ่นกว่า

  print('Pi = $pi');
  print('แอป: $appName');
  print('ตอนนี้: $now');
  print('$greeting');

  // ─────────────── Late final ───────────────
  // ประกาศก่อน กำหนดค่าทีหลัง
  late final String lazyValue;
  // ใช้งานจริงก่อนที่จะกำหนดค่าจะเกิด Error
  lazyValue = 'กำหนดค่าแล้ว';
  print(lazyValue);
}
```

---

## ขั้นตอนที่ 23: ชนิดข้อมูลตัวเลข (Numeric Types)

```dart
void main() {
  // ─────────────── int ───────────────
  int a = 42;
  int b = -100;
  int c = 0;
  int bigNumber = 9007199254740992;  // 2^53
  
  // ─────────────── double ───────────────
  double x = 3.14;
  double y = -2.718;
  double z = 1.0e10;    // Scientific notation
  double infinity = double.infinity;
  double nan = double.nan;
  
  // ─────────────── num (parent of int and double) ───────────────
  num number1 = 10;      // int
  num number2 = 10.5;    // double
  
  // ─────────────── การดำเนินการทางคณิตศาสตร์ ───────────────
  print('บวก: ${a + 10}');          // 52
  print('ลบ: ${a - 10}');           // 32
  print('คูณ: ${a * 2}');           // 84
  print('หาร: ${a / 5}');           // 8.4 (ได้ double)
  print('หารเอาเฉพาะจำนวนเต็ม: ${a ~/ 5}');  // 8 (ได้ int)
  print('หารเอาเศษ: ${a % 5}');     // 2
  
  // ─────────────── Properties และ Methods ───────────────
  print('int.maxValue ไม่มีใน Dart (ไม่มี overflow)');
  print('is NaN: ${nan.isNaN}');
  print('is Infinite: ${infinity.isInfinite}');
  print('is Finite: ${x.isFinite}');
  print('Absolute: ${b.abs()}');
  
  // ─────────────── การแปลงเป็น String ───────────────
  print(a.toString());           // "42"
  print(x.toStringAsFixed(2));   // "3.14"
  print(x.toStringAsPrecision(4)); // "3.140"
  
  // ─────────────── Parsing จาก String ───────────────
  int parsed = int.parse('100');
  double parsedDouble = double.parse('3.14');
  int? maybeNull = int.tryParse('abc'); // คืน null ถ้า parse ไม่ได้
  
  print('Parsed: $parsed');
  print('Parsed double: $parsedDouble');
  print('Try parse: $maybeNull');  // null
  
  // ─────────────── Bitwise Operations ───────────────
  print('AND: ${5 & 3}');    // 1
  print('OR: ${5 | 3}');     // 7
  print('XOR: ${5 ^ 3}');    // 6
  print('NOT: ${~5}');       // -6
  print('Left shift: ${5 << 1}');   // 10
  print('Right shift: ${5 >> 1}');  // 2
}
```

---

## ขั้นตอนที่ 24: String (ข้อความ)

```dart
void main() {
  // ─────────────── การสร้าง String ───────────────
  String s1 = 'ใช้ single quote';
  String s2 = "ใช้ double quote";
  String s3 = 'I\'m learning Dart';     // escape character
  String s4 = "She said \"Hello\"";
  
  // ─────────────── Multiline String ───────────────
  String multiline = '''
บรรทัดที่ 1
บรรทัดที่ 2
บรรทัดที่ 3
''';

  String multiline2 = """
สวัสดี
โลก
""";

  // ─────────────── String Interpolation ───────────────
  String firstName = 'สมชาย';
  String lastName = 'ใจดี';
  int age = 25;
  
  // วิธีที่ 1: ใช้ $variable
  print('ชื่อ: $firstName');
  
  // วิธีที่ 2: ใช้ ${expression}
  print('ชื่อเต็ม: ${firstName + ' ' + lastName}');
  print('อายุ: ${age + 1} ปีหน้า');
  print('ตัวพิมพ์ใหญ่: ${firstName.toUpperCase()}');
  
  // ─────────────── String Operations ───────────────
  String text = 'Flutter คือสุดยอด!';
  
  // ความยาว
  print('ความยาว: ${text.length}');
  
  // การเข้าถึงตัวอักษร
  print('ตัวแรก: ${text[0]}');
  
  // substring
  print('Substring: ${text.substring(0, 7)}');
  
  // ค้นหา
  print('มี Flutter: ${text.contains('Flutter')}');
  print('ตำแหน่ง: ${text.indexOf('คือ')}');
  
  // แทนที่
  print('แทนที่: ${text.replaceAll('สุดยอด', 'เยี่ยม')}');
  
  // ตัด whitespace
  String padded = '  ตัดช่องว่าง  ';
  print('ตัดแล้ว: "${padded.trim()}"');
  print('ตัดซ้าย: "${padded.trimLeft()}"');
  print('ตัดขวา: "${padded.trimRight()}"');
  
  // แยกข้อความ
  String csv = 'แมว,สุนัข,กระต่าย';
  List<String> animals = csv.split(',');
  print('สัตว์: $animals');
  
  // ต่อ String
  String joined = animals.join(' และ ');
  print('ต่อกัน: $joined');
  
  // เปรียบเทียบ
  print('เท่ากัน: ${'abc' == 'abc'}');
  print('ไม่เท่ากัน: ${'abc' != 'xyz'}');
  
  // ตรวจสอบ prefix/suffix
  print('เริ่มด้วย: ${text.startsWith('Flutter')}');
  print('ลงท้ายด้วย: ${text.endsWith('!')}');
  
  // ตัวพิมพ์
  print('ตัวใหญ่: ${text.toUpperCase()}');
  print('ตัวเล็ก: ${text.toLowerCase()}');
  
  // ─────────────── Raw String ───────────────
  String rawString = r'ไม่แปล \n และ \t';
  print(rawString); // แสดง \n และ \t ตามตัวอักษร
  
  // ─────────────── String Buffer (สร้าง String ขนาดใหญ่) ───────────────
  StringBuffer buffer = StringBuffer();
  buffer.write('Hello');
  buffer.write(' ');
  buffer.write('World');
  buffer.writeln('!');
  buffer.writeAll(['1', '2', '3'], ', ');
  
  print(buffer.toString()); // Hello World!\n1, 2, 3
  
  // ─────────────── String Padding ───────────────
  String num = '42';
  print(num.padLeft(5));          // "   42"
  print(num.padRight(5));         // "42   "
  print(num.padLeft(5, '0'));     // "00042"
}
```

---

## ขั้นตอนที่ 25: Boolean (ค่าตรรกะ)

```dart
void main() {
  // ─────────────── การประกาศ bool ───────────────
  bool isTrue = true;
  bool isFalse = false;
  
  // ─────────────── Logical Operators ───────────────
  // AND (&&): true เมื่อทั้งคู่เป็น true
  print('true && true = ${true && true}');   // true
  print('true && false = ${true && false}'); // false
  
  // OR (||): true เมื่ออย่างน้อยหนึ่งตัวเป็น true
  print('false || true = ${false || true}'); // true
  print('false || false = ${false || false}'); // false
  
  // NOT (!): กลับค่า
  print('!true = ${!true}');    // false
  print('!false = ${!false}');  // true
  
  // ─────────────── การใช้งานทั่วไป ───────────────
  int age = 20;
  bool hasID = true;
  
  bool canDrink = age >= 18 && hasID;
  print('ดื่มแอลกอฮอล์ได้: $canDrink');
  
  bool isWeekend = true;
  bool isHoliday = false;
  bool canRelax = isWeekend || isHoliday;
  print('พักผ่อนได้: $canRelax');
  
  // ─────────────── Comparison Operators ───────────────
  int a = 10, b = 20;
  print('$a == $b: ${a == b}');   // false
  print('$a != $b: ${a != b}');   // true
  print('$a < $b: ${a < b}');     // true
  print('$a > $b: ${a > b}');     // false
  print('$a <= $b: ${a <= b}');   // true
  print('$a >= $b: ${a >= b}');   // false
}
```

---

## ขั้นตอนที่ 26: Null Safety ใน Dart

Null Safety เป็นหนึ่งในฟีเจอร์สำคัญที่สุดของ Dart ช่วยป้องกัน Null Pointer Exception

```dart
void main() {
  // ─────────────── Non-nullable (ค่าเริ่มต้น) ───────────────
  // ตัวแปรปกติต้องมีค่าเสมอ ไม่ใช่ null ได้
  String name = 'สมชาย';
  int age = 25;
  
  // ผิด! ทำไม่ได้
  // String error = null; // Compile Error!
  
  // ─────────────── Nullable Types (ใช้ ?) ───────────────
  // เพิ่ม ? เพื่อให้รับค่า null ได้
  String? nullableName = null;
  int? nullableAge;     // ค่าเริ่มต้นคือ null
  double? maybePrice = 100.0;
  
  print('Nullable name: $nullableName');  // null
  print('Nullable age: $nullableAge');    // null
  print('Maybe price: $maybePrice');      // 100.0
  
  // ─────────────── Null-aware Operators ───────────────
  
  // ?? (Null coalescing): ถ้า null ให้ใช้ค่าที่ระบุ
  String displayName = nullableName ?? 'ไม่ระบุชื่อ';
  print('Display: $displayName');  // ไม่ระบุชื่อ
  
  // ??= (Null assignment): กำหนดค่าถ้าเป็น null
  nullableAge ??= 0;
  print('Age: $nullableAge');  // 0
  
  // ?. (Null-aware access): เรียก method ถ้าไม่ใช่ null
  String? maybeText = 'Hello';
  print(maybeText?.length);  // 5
  maybeText = null;
  print(maybeText?.length);  // null (ไม่ crash)
  
  // !. (Null assertion): บอกว่าไม่ใช่ null (ระวัง! อาจ crash)
  String? definitelyNotNull = 'มีค่าแน่ๆ';
  print(definitelyNotNull!.length);  // 7
  
  // ─────────────── Null Checks ───────────────
  String? value = 'Hello';
  
  // วิธีที่ 1: ตรวจสอบก่อนใช้
  if (value != null) {
    print(value.length);  // ปลอดภัย
  }
  
  // วิธีที่ 2: ใช้ ?. 
  print(value?.toUpperCase());  // ปลอดภัย
  
  // วิธีที่ 3: ใช้ late
  late String lateValue;
  lateValue = 'กำหนดทีหลัง';
  print(lateValue);  // ปลอดภัย เพราะกำหนดค่าแล้ว
}
```

---

## ขั้นตอนที่ 27: ชนิดข้อมูลพิเศษ

```dart
void main() {
  // ─────────────── Symbol ───────────────
  Symbol symbol = #mySymbol;
  print(symbol);  // Symbol("mySymbol")
  
  // ─────────────── Runes (Unicode) ───────────────
  Runes runes = Runes('♥ \u{1F601}');  // ♥ 😁
  print(String.fromCharCodes(runes));
  
  // รับ runes จาก string
  'Hello'.runes.forEach((int r) {
    print(String.fromCharCode(r));
  });
  
  // ─────────────── Type Inference ───────────────
  var list = [1, 2, 3];              // List<int>
  var map = {'key': 'value'};        // Map<String, String>
  var set = {1, 2, 3};               // Set<int>
  var mixed = [1, 'two', 3.0];       // List<Object>
  
  // ตรวจสอบชนิด
  print(list.runtimeType);    // List<int>
  print(map.runtimeType);     // _InternalLinkedHashMap<String, String>
  
  // ─────────────── Type Checking (is) ───────────────
  Object someValue = 42;
  
  if (someValue is int) {
    print('เป็น int: ${someValue + 1}');  // smart cast ใน block
  }
  
  if (someValue is! String) {
    print('ไม่ใช่ String');
  }
  
  // ─────────────── Type Casting (as) ───────────────
  Object anotherValue = 'Hello';
  String casted = anotherValue as String;
  print(casted.toUpperCase());  // HELLO
  
  // ระวัง! ถ้า cast ผิดชนิดจะ throw CastError
  // int wrongCast = anotherValue as int; // Error!
  
  // ─────────────── Type Conversion ───────────────
  // int <-> double
  int i = 10;
  double d = i.toDouble();   // 10.0
  int back = d.toInt();      // 10 (ตัดทศนิยม)
  int rounded = d.round();   // ปัดเศษ
  
  // String <-> int
  String numStr = '42';
  int fromStr = int.parse(numStr);       // 42
  String toStr = fromStr.toString();    // "42"
  
  // String <-> double
  double fromStr2 = double.parse('3.14');
  String toStr2 = fromStr2.toStringAsFixed(2);  // "3.14"
  
  print('int: $i, double: $d, back: $back');
  print('fromStr: $fromStr, toStr: $toStr');
}
```

---

## ขั้นตอนที่ 28: Operators ทั้งหมดใน Dart

```dart
void main() {
  // ─────────────── Arithmetic Operators ───────────────
  int a = 15, b = 4;
  
  print('บวก: ${a + b}');      // 19
  print('ลบ: ${a - b}');       // 11
  print('คูณ: ${a * b}');      // 60
  print('หาร: ${a / b}');      // 3.75 (double)
  print('หารจำนวนเต็ม: ${a ~/ b}');  // 3
  print('หารเศษ: ${a % b}');   // 3
  
  // Unary operators
  int x = 5;
  print('ลบ: ${-x}');          // -5
  
  // ─────────────── Increment / Decrement ───────────────
  int count = 0;
  
  count++;        // เพิ่มหลังใช้
  print(count);   // 1
  
  ++count;        // เพิ่มก่อนใช้
  print(count);   // 2
  
  print(count++); // แสดง 2 แล้วค่อยเพิ่ม
  print(count);   // 3
  
  print(++count); // เพิ่มก่อนแล้วแสดง 4
  
  count--;        // ลด
  count--;
  print(count);   // 2
  
  // ─────────────── Assignment Operators ───────────────
  int n = 10;
  
  n += 5;    // n = n + 5 = 15
  n -= 3;    // n = n - 3 = 12
  n *= 2;    // n = n * 2 = 24
  n ~/= 5;   // n = n ~/ 5 = 4
  n %= 3;    // n = n % 3 = 1
  
  print('n = $n');  // 1
  
  // Null-aware assignment
  String? str;
  str ??= 'default';  // กำหนดค่าถ้าเป็น null
  print(str);  // default
  
  // ─────────────── Comparison Operators ───────────────
  print('5 == 5: ${5 == 5}');    // true
  print('5 != 4: ${5 != 4}');    // true
  print('5 > 3: ${5 > 3}');      // true
  print('5 < 3: ${5 < 3}');      // false
  print('5 >= 5: ${5 >= 5}');    // true
  print('5 <= 4: ${5 <= 4}');    // false
  
  // ─────────────── Logical Operators ───────────────
  bool p = true, q = false;
  
  print('AND: ${p && q}');   // false
  print('OR: ${p || q}');    // true
  print('NOT p: ${!p}');     // false
  
  // Short-circuit evaluation
  // ถ้า && ด้านซ้ายเป็น false จะไม่ประเมินด้านขวา
  // ถ้า || ด้านซ้ายเป็น true จะไม่ประเมินด้านขวา
  
  // ─────────────── Conditional (Ternary) Operator ───────────────
  int score = 75;
  String grade = score >= 60 ? 'ผ่าน' : 'ไม่ผ่าน';
  print('เกรด: $grade');  // ผ่าน
  
  // Nested ternary (ไม่แนะนำถ้าซับซ้อนมาก)
  String level = score >= 80 ? 'A' : score >= 70 ? 'B' : score >= 60 ? 'C' : 'F';
  print('ระดับ: $level');  // B
  
  // ─────────────── Cascade Operator (..) ───────────────
  // ใช้เรียก methods หลายอันต่อกัน
  StringBuffer buffer = StringBuffer()
    ..write('สวัสดี ')
    ..write('โลก')
    ..writeln('!')
    ..write('Dart');
  print(buffer.toString());
  
  // ─────────────── Spread Operator (...) ───────────────
  List<int> list1 = [1, 2, 3];
  List<int> list2 = [4, 5, 6];
  List<int> combined = [...list1, ...list2];
  print(combined);  // [1, 2, 3, 4, 5, 6]
  
  // Null-aware spread
  List<int>? maybeList;
  List<int> safe = [0, ...?maybeList, 7];
  print(safe);  // [0, 7]
  
  // ─────────────── Type Test Operators ───────────────
  Object obj = 'Hello';
  print(obj is String);    // true
  print(obj is! int);      // true
}
```

---

## ขั้นตอนที่ 29: Operators ขั้นสูง

```dart
void main() {
  // ─────────────── Bitwise Operators ───────────────
  int a = 5;  // 0101 ในฐาน 2
  int b = 3;  // 0011 ในฐาน 2
  
  print('AND: ${a & b}');    // 0001 = 1
  print('OR: ${a | b}');     // 0111 = 7
  print('XOR: ${a ^ b}');    // 0110 = 6
  print('NOT: ${~a}');       // -(a+1) = -6
  print('Left shift: ${a << 1}');    // 1010 = 10
  print('Right shift: ${a >> 1}');   // 0010 = 2
  print('Unsigned right: ${-1 >>> 1}'); // ขยับขวาโดยใส่ 0
  
  // ─────────────── Operator Precedence ───────────────
  // ลำดับความสำคัญ (สูง → ต่ำ)
  // 1. () : วงเล็บ
  // 2. [] ?[] . ?. ! : Member access
  // 3. -(unary) !(unary) ~
  // 4. * / ~/ %
  // 5. + -
  // 6. << >> >>>
  // 7. &
  // 8. ^
  // 9. |
  // 10. < > <= >= as is is!
  // 11. == !=
  // 12. &&
  // 13. ||
  // 14. ??
  // 15. ? : (ternary)
  // 16. = *= += ...
  
  // ตัวอย่าง
  int result = 2 + 3 * 4;           // 2 + 12 = 14 (ไม่ใช่ 20)
  int withParen = (2 + 3) * 4;       // 5 * 4 = 20
  bool complex = 1 + 2 > 2 || false; // (1+2) > 2 || false = true
  
  print('result: $result');       // 14
  print('withParen: $withParen'); // 20
  print('complex: $complex');     // true
  
  // ─────────────── Index Operator ([]) ───────────────
  List<String> fruits = ['แอปเปิ้ล', 'กล้วย', 'มะม่วง'];
  print(fruits[0]);   // แอปเปิ้ล
  print(fruits[2]);   // มะม่วง
  fruits[1] = 'ส้ม';  // เปลี่ยนค่า
  print(fruits);      // [แอปเปิ้ล, ส้ม, มะม่วง]
  
  // ─────────────── Member Access ───────────────
  String text = 'Hello World';
  print(text.length);           // 11 (. operator)
  print(text?.toUpperCase());   // HELLO WORLD (?. operator)
  
  String? nullText;
  print(nullText?.length);      // null (ปลอดภัย)
  // print(nullText!.length);   // Error! (! บอกว่าไม่ null แต่เป็น null)
}
```

---

## ขั้นตอนที่ 30: String Formatting ขั้นสูง

```dart
void main() {
  // ─────────────── String Interpolation ───────────────
  String name = 'Alice';
  int age = 30;
  double height = 165.5;
  
  // พื้นฐาน
  print('สวัสดี $name คุณอายุ $age ปี');
  
  // Expression
  print('อีก ${30 - age} ปีจะอายุ 60');
  
  // Method call
  print('ชื่อตัวพิมพ์ใหญ่: ${name.toUpperCase()}');
  
  // ─────────────── Number Formatting ───────────────
  double price = 1234567.89;
  
  // Fixed decimal places
  print(price.toStringAsFixed(2));    // 1234567.89
  print(price.toStringAsFixed(0));    // 1234568
  
  // Precision
  print(price.toStringAsPrecision(4)); // 1.235e+6
  
  // Exponential
  print(price.toStringAsExponential(2)); // 1.23e+6
  
  // ─────────────── ใช้ intl Package สำหรับ Formatting ───────────────
  // (ต้องเพิ่ม intl: ^0.19.0 ใน pubspec.yaml)
  // import 'package:intl/intl.dart';
  //
  // NumberFormat formatter = NumberFormat('#,##0.00', 'th_TH');
  // print(formatter.format(price));  // 1,234,567.89
  //
  // DateFormat dateFormatter = DateFormat('dd/MM/yyyy', 'th_TH');
  // print(dateFormatter.format(DateTime.now()));
  
  // ─────────────── String Padding และ Alignment ───────────────
  String padded = '42'.padLeft(8);        // "      42"
  String paddedZero = '42'.padLeft(8, '0'); // "00000042"
  String paddedRight = '42'.padRight(8, '-'); // "42------"
  
  print('"$padded"');
  print('"$paddedZero"');
  print('"$paddedRight"');
  
  // ─────────────── String Template แบบซับซ้อน ───────────────
  List<String> items = ['สินค้า A', 'สินค้า B', 'สินค้า C'];
  List<double> prices = [100.0, 200.0, 150.0];
  
  // สร้างตารางแบบง่าย
  print('─' * 30);
  print('${'รายการ'.padRight(15)}${'ราคา'.padLeft(10)}');
  print('─' * 30);
  
  for (int i = 0; i < items.length; i++) {
    String priceStr = '${prices[i].toStringAsFixed(2)} บาท';
    print('${items[i].padRight(15)}${priceStr.padLeft(15)}');
  }
  
  double total = prices.fold(0, (sum, p) => sum + p);
  print('─' * 30);
  print('${'รวม:'.padRight(15)}${'${total.toStringAsFixed(2)} บาท'.padLeft(15)}');
}
```

---

## ขั้นตอนที่ 31: ชนิดข้อมูลขั้นสูง - Records (Dart 3.0+)

```dart
void main() {
  // ─────────────── Records เบื้องต้น ───────────────
  // Records คือ data structure ที่รวมค่าหลายอย่างไว้ด้วยกัน
  
  // Positional record
  (String, int) person = ('สมชาย', 25);
  print(person.$1);  // สมชาย
  print(person.$2);  // 25
  
  // Named record
  ({String name, int age}) namedPerson = (name: 'สมหญิง', age: 30);
  print(namedPerson.name);  // สมหญิง
  print(namedPerson.age);   // 30
  
  // ผสม positional และ named
  (String, {int age, String city}) mixed = ('มานะ', age: 28, city: 'กรุงเทพ');
  print(mixed.$1);      // มานะ
  print(mixed.age);     // 28
  print(mixed.city);    // กรุงเทพ
  
  // ─────────────── Records ใน Functions ───────────────
  // คืนค่าหลายค่าจาก function
  (String, double) divide(int a, int b) {
    return ('$a / $b', a / b);
  }
  
  var result = divide(10, 3);
  print('${result.$1} = ${result.$2}');  // 10 / 3 = 3.3333...
  
  // ─────────────── Pattern Matching กับ Records ───────────────
  var point = (x: 3, y: 4);
  var (x: px, y: py) = point;  // Destructuring
  print('x=$px, y=$py');  // x=3, y=4
  
  // ─────────────── Records เปรียบเทียบ ───────────────
  var r1 = ('a', 1);
  var r2 = ('a', 1);
  var r3 = ('b', 2);
  
  print(r1 == r2);  // true (เปรียบค่า)
  print(r1 == r3);  // false
}
```

---

## ขั้นตอนที่ 32: Patterns และ Pattern Matching (Dart 3.0+)

```dart
void main() {
  // ─────────────── Pattern Matching พื้นฐาน ───────────────
  
  // Variable patterns
  var [a, b, c] = [1, 2, 3];
  print('$a, $b, $c');  // 1, 2, 3
  
  // Object patterns
  var (x, y) = (10, 20);
  print('x=$x, y=$y');  // x=10, y=20
  
  // ─────────────── Switch Expressions กับ Patterns ───────────────
  int value = 42;
  String description = switch (value) {
    0 => 'ศูนย์',
    1 => 'หนึ่ง',
    int n when n < 0 => 'ลบ',
    int n when n > 100 => 'มากกว่า 100',
    _ => 'ตัวเลขอื่น',
  };
  print(description);  // ตัวเลขอื่น
  
  // ─────────────── Pattern กับ Collections ───────────────
  List<int> numbers = [1, 2, 3, 4, 5];
  
  var [first, second, ...rest] = numbers;
  print('first=$first, second=$second, rest=$rest');
  // first=1, second=2, rest=[3, 4, 5]
  
  // ─────────────── Map Pattern ───────────────
  Map<String, dynamic> user = {
    'name': 'สมชาย',
    'age': 25,
    'city': 'กรุงเทพ',
  };
  
  var {'name': String userName, 'age': int userAge} = user;
  print('$userName, $userAge');  // สมชาย, 25
  
  // ─────────────── Guard clause ───────────────
  List<(String, int)> scores = [
    ('Alice', 90),
    ('Bob', 45),
    ('Charlie', 75),
  ];
  
  for (var (name, score) in scores) {
    switch (score) {
      case int s when s >= 80:
        print('$name: ยอดเยี่ยม');
      case int s when s >= 60:
        print('$name: ผ่าน');
      default:
        print('$name: ไม่ผ่าน');
    }
  }
}
```

---

## ขั้นตอนที่ 33: Type System ขั้นสูง

```dart
void main() {
  // ─────────────── Type Hierarchy ───────────────
  // Object: Root ของทุกชนิดข้อมูล
  // ├── bool
  // ├── num
  // │   ├── int
  // │   └── double
  // ├── String
  // ├── List
  // ├── Set
  // ├── Map
  // ├── Function
  // └── ...
  //
  // Null: มีแค่ null value
  // Never: Bottom type (ไม่มีค่า, ใช้ใน function ที่ throw เสมอ)
  
  // ─────────────── Bottom Type: Never ───────────────
  Never throwAlways() {
    throw Exception('ไม่มีทางคืนค่าได้');
  }
  
  // ─────────────── Dynamic vs Object ───────────────
  // dynamic: ปิด type checking
  dynamic d = 'text';
  d = 42;           // OK
  d.anything();     // ไม่มี compile error แต่อาจ runtime error
  
  // Object?: เปิด type checking, ต้อง cast ก่อนใช้ method เฉพาะ
  Object? o = 'text';
  o = 42;           // OK
  // o.toUpperCase(); // Compile error! ต้อง cast ก่อน
  if (o is String) {
    print(o.toUpperCase()); // OK หลัง type check
  }
  
  // ─────────────── Generics พื้นฐาน ───────────────
  // List<T>: List ที่เก็บชนิดข้อมูล T
  List<int> intList = [1, 2, 3];
  List<String> strList = ['a', 'b', 'c'];
  List<dynamic> mixedList = [1, 'two', 3.0];
  
  // ─────────────── Type Alias ───────────────
  typedef StringList = List<String>;
  typedef Predicate<T> = bool Function(T value);
  
  StringList names = ['Alice', 'Bob'];
  Predicate<int> isEven = (n) => n % 2 == 0;
  
  print(names);           // [Alice, Bob]
  print(isEven(4));       // true
  print(isEven(3));       // false
}
```

---

## ขั้นตอนที่ 34: สรุป Type System และ Operators

```dart
// ─────────────── Quick Reference ───────────────

// ชนิดข้อมูลหลัก
void quickReference() {
  // Numbers
  int i = 42;
  double d = 3.14;
  num n = 10; // int หรือ double
  
  // Text
  String s = 'text';
  
  // Boolean
  bool b = true;
  
  // Collections (จะเรียนเพิ่มใน Part 05)
  List<int> list = [1, 2, 3];
  Set<String> set = {'a', 'b', 'c'};
  Map<String, int> map = {'one': 1, 'two': 2};
  
  // Special
  var v = 'inferred'; // type inference
  dynamic dy = 'dynamic';
  Object? obj = null; // nullable Object
  
  // Null safety
  String? nullable = null;
  String nonNull = nullable ?? 'default';
  String? chained = nullable?.toUpperCase();
}

// ─────────────── Operators Quick Reference ───────────────
/*
Arithmetic:  + - * / ~/ % 
Comparison:  == != < > <= >=
Logical:     && || !
Assignment:  = += -= *= /= ~/= %= &&= ||= ??=
Bitwise:     & | ^ ~ << >> >>>
Increment:   ++ --
Ternary:     condition ? true : false
Null-aware:  ?? ??= ?. ?[]
Type:        is is! as
Cascade:     ..
Spread:      ...  ...?
*/
```

---

## ขั้นตอนที่ 35-40: Workshop - แบบฝึกหัดปฏิบัติ

```dart
// แบบฝึกหัดที่ 1: ตัวแปรและชนิดข้อมูล
void exercise1() {
  // TODO: สร้างตัวแปรต่อไปนี้
  // - ชื่อนักเรียน (String)
  // - อายุ (int)
  // - คะแนนเฉลี่ย (double)
  // - เป็นนักเรียนใหม่หรือไม่ (bool)
  // - ชื่อกลาง (nullable String)
  
  String studentName = 'สมชาย ใจดี';
  int studentAge = 18;
  double gpa = 3.75;
  bool isNewStudent = true;
  String? middleName = null;
  
  // แสดงข้อมูลทั้งหมด
  print('ชื่อ: $studentName');
  print('อายุ: $studentAge ปี');
  print('GPA: ${gpa.toStringAsFixed(2)}');
  print('นักเรียนใหม่: $isNewStudent');
  print('ชื่อกลาง: ${middleName ?? 'ไม่มี'}');
}

// แบบฝึกหัดที่ 2: Null Safety
void exercise2() {
  List<String?> names = ['Alice', null, 'Bob', null, 'Charlie'];
  
  // แสดงชื่อที่ไม่ใช่ null พร้อมหมายเลข
  int count = 0;
  for (int i = 0; i < names.length; i++) {
    String? name = names[i];
    if (name != null) {
      count++;
      print('$count. $name');
    }
  }
  
  print('พบ $count ชื่อ');
}

// แบบฝึกหัดที่ 3: String Operations
void exercise3() {
  String text = "  สวัสดี Flutter! ยินดีต้อนรับ  ";
  
  // ตัดช่องว่าง
  String trimmed = text.trim();
  print(trimmed);
  
  // เปลี่ยนเป็นตัวพิมพ์ใหญ่
  print(trimmed.toUpperCase());
  
  // ค้นหาคำ
  print('มีคำ Flutter: ${trimmed.contains('Flutter')}');
  print('ตำแหน่ง: ${trimmed.indexOf('Flutter')}');
  
  // แทนที่
  print(trimmed.replaceAll('Flutter', 'Dart'));
  
  // แยกคำ
  List<String> words = trimmed.split(' ');
  print('จำนวนคำ: ${words.length}');
  print(words);
}

// แบบฝึกหัดที่ 4: การคำนวณ
void exercise4() {
  // คำนวณ BMI
  double weight = 70.0; // กิโลกรัม
  double height = 1.75; // เมตร
  
  double bmi = weight / (height * height);
  
  String category = bmi < 18.5 ? 'น้ำหนักน้อย' :
                    bmi < 25.0 ? 'ปกติ' :
                    bmi < 30.0 ? 'น้ำหนักเกิน' : 'อ้วน';
  
  print('น้ำหนัก: $weight kg');
  print('ส่วนสูง: $height m');
  print('BMI: ${bmi.toStringAsFixed(2)}');
  print('สถานะ: $category');
}

// แบบฝึกหัดที่ 5: Records (Dart 3.0+)
({String name, double gpa, String grade}) calculateGrade(
  String name, double score
) {
  String grade = score >= 80 ? 'A' :
                 score >= 70 ? 'B' :
                 score >= 60 ? 'C' :
                 score >= 50 ? 'D' : 'F';
  
  double gpa = score >= 80 ? 4.0 :
               score >= 70 ? 3.0 :
               score >= 60 ? 2.0 :
               score >= 50 ? 1.0 : 0.0;
  
  return (name: name, gpa: gpa, grade: grade);
}

void exercise5() {
  var result = calculateGrade('สมชาย', 75.5);
  print('ชื่อ: ${result.name}');
  print('เกรด: ${result.grade}');
  print('GPA: ${result.gpa}');
}

void main() {
  print('═' * 40);
  print('Exercise 1: ตัวแปรและชนิดข้อมูล');
  print('═' * 40);
  exercise1();
  
  print('\n' + '═' * 40);
  print('Exercise 2: Null Safety');
  print('═' * 40);
  exercise2();
  
  print('\n' + '═' * 40);
  print('Exercise 3: String Operations');
  print('═' * 40);
  exercise3();
  
  print('\n' + '═' * 40);
  print('Exercise 4: การคำนวณ BMI');
  print('═' * 40);
  exercise4();
  
  print('\n' + '═' * 40);
  print('Exercise 5: Records');
  print('═' * 40);
  exercise5();
}
```

---

## ขั้นตอนที่ 41-45: Project - เครื่องคิดเลขพื้นฐาน

```dart
// project: basic_calculator.dart

void main() {
  Calculator calc = Calculator();
  
  print('=== เครื่องคิดเลข Dart ===\n');
  
  // การคำนวณพื้นฐาน
  print('การคำนวณพื้นฐาน:');
  print('10 + 5 = ${calc.add(10, 5)}');
  print('10 - 3 = ${calc.subtract(10, 3)}');
  print('4 × 6 = ${calc.multiply(4, 6)}');
  print('15 ÷ 4 = ${calc.divide(15, 4).toStringAsFixed(4)}');
  print('15 ÷ 4 = ${calc.intDivide(15, 4)} เศษ ${calc.modulo(15, 4)}');
  
  print('\nการคำนวณขั้นสูง:');
  print('2^8 = ${calc.power(2, 8)}');
  print('√144 = ${calc.sqrt(144)}');
  print('|−42| = ${calc.abs(-42)}');
  
  print('\nการแปลงเลข:');
  print('255 (10) = ${calc.toHex(255)} (16)');
  print('255 (10) = ${calc.toBinary(255)} (2)');
  print('255 (10) = ${calc.toOctal(255)} (8)');
  
  print('\nประวัติการคำนวณ:');
  for (String history in calc.history) {
    print('  $history');
  }
}

class Calculator {
  final List<String> history = [];
  
  void _addHistory(String operation) {
    history.add(operation);
  }
  
  double add(double a, double b) {
    double result = a + b;
    _addHistory('$a + $b = $result');
    return result;
  }
  
  double subtract(double a, double b) {
    double result = a - b;
    _addHistory('$a - $b = $result');
    return result;
  }
  
  double multiply(double a, double b) {
    double result = a * b;
    _addHistory('$a × $b = $result');
    return result;
  }
  
  double divide(double a, double b) {
    if (b == 0) {
      _addHistory('$a ÷ 0 = Error (หารด้วยศูนย์)');
      throw ArgumentError('ไม่สามารถหารด้วยศูนย์ได้');
    }
    double result = a / b;
    _addHistory('$a ÷ $b = $result');
    return result;
  }
  
  int intDivide(int a, int b) {
    if (b == 0) throw ArgumentError('ไม่สามารถหารด้วยศูนย์ได้');
    return a ~/ b;
  }
  
  int modulo(int a, int b) {
    if (b == 0) throw ArgumentError('ไม่สามารถหารด้วยศูนย์ได้');
    return a % b;
  }
  
  double power(double base, double exponent) {
    double result = 1;
    for (int i = 0; i < exponent; i++) {
      result *= base;
    }
    _addHistory('$base^$exponent = $result');
    return result;
  }
  
  double sqrt(double value) {
    if (value < 0) throw ArgumentError('ไม่สามารถหาราก√ของจำนวนลบได้');
    // Newton's method
    double x = value;
    double y = (x + value / x) / 2;
    while ((x - y).abs() > 0.0001) {
      x = y;
      y = (x + value / x) / 2;
    }
    _addHistory('√$value = $y');
    return y;
  }
  
  double abs(double value) {
    return value < 0 ? -value : value;
  }
  
  String toHex(int value) => value.toRadixString(16).toUpperCase();
  String toBinary(int value) => value.toRadixString(2);
  String toOctal(int value) => value.toRadixString(8);
}
```

---

## ขั้นตอนที่ 46-50: สรุป Part 02 และ Exercise สุดท้าย

### สรุปสิ่งที่เรียนใน Part 02

```
✅ ขั้นตอนที่ 21: การประกาศตัวแปร (var, type, dynamic, Object)
✅ ขั้นตอนที่ 22: ค่าคงที่ (const, final, late)
✅ ขั้นตอนที่ 23: ชนิดข้อมูลตัวเลข (int, double, num)
✅ ขั้นตอนที่ 24: String และ String Operations
✅ ขั้นตอนที่ 25: Boolean และ Logical Operators
✅ ขั้นตอนที่ 26: Null Safety (?, ??, ??=, ?., !)
✅ ขั้นตอนที่ 27: ชนิดข้อมูลพิเศษ (Symbol, Runes)
✅ ขั้นตอนที่ 28: Operators ทั้งหมด
✅ ขั้นตอนที่ 29: Operators ขั้นสูง (Bitwise, Cascade, Spread)
✅ ขั้นตอนที่ 30: String Formatting ขั้นสูง
✅ ขั้นตอนที่ 31: Records (Dart 3.0+)
✅ ขั้นตอนที่ 32: Pattern Matching (Dart 3.0+)
✅ ขั้นตอนที่ 33: Type System ขั้นสูง
✅ ขั้นตอนที่ 34: สรุป Type System
✅ ขั้นตอนที่ 35-40: Workshop - แบบฝึกหัดปฏิบัติ
✅ ขั้นตอนที่ 41-45: Project - เครื่องคิดเลข
```

### Challenge Exercise: Currency Converter

```dart
// challenge: currency_converter.dart
// สร้างโปรแกรมแปลงสกุลเงิน

void main() {
  // อัตราแลกเปลี่ยน (ณ วันที่สร้าง)
  final Map<String, double> rates = {
    'USD': 1.0,       // US Dollar (base)
    'THB': 35.50,     // Thai Baht
    'EUR': 0.92,      // Euro
    'GBP': 0.79,      // British Pound
    'JPY': 149.50,    // Japanese Yen
    'CNY': 7.24,      // Chinese Yuan
    'KRW': 1330.0,    // Korean Won
    'SGD': 1.34,      // Singapore Dollar
  };
  
  // แปลงเงิน
  double convert(double amount, String from, String to) {
    double? fromRate = rates[from];
    double? toRate = rates[to];
    
    if (fromRate == null || toRate == null) {
      throw ArgumentError('ไม่รู้จักสกุลเงิน');
    }
    
    // แปลงเป็น USD ก่อน แล้วค่อยแปลงเป็น target
    double inUSD = amount / fromRate;
    return inUSD * toRate;
  }
  
  // ทดสอบ
  print('=== Currency Converter ===\n');
  
  List<(double, String, String)> conversions = [
    (1000.0, 'THB', 'USD'),
    (100.0, 'USD', 'THB'),
    (50.0, 'EUR', 'JPY'),
    (10000.0, 'JPY', 'THB'),
    (1.0, 'GBP', 'EUR'),
  ];
  
  for (var (amount, from, to) in conversions) {
    double result = convert(amount, from, to);
    print('${amount.toStringAsFixed(2)} $from = ${result.toStringAsFixed(2)} $to');
  }
  
  print('\n=== ตาราง Exchange Rate (เทียบกับ USD) ===');
  print('${'สกุลเงิน'.padRight(12)}${'อัตรา'.padLeft(10)}');
  print('─' * 22);
  
  rates.forEach((currency, rate) {
    print('${currency.padRight(12)}${rate.toString().padLeft(10)}');
  });
}
```

### แบบฝึกหัดสุดท้าย: สร้างแอป Profile Card

```dart
// exercise_final: profile_card.dart

void main() {
  // สร้าง Profile ของตัวเอง
  var profile = (
    name: 'ชื่อของคุณ',
    age: 0, // อายุของคุณ
    profession: 'Flutter Developer',
    skills: ['Dart', 'Flutter', 'Firebase'],
    contact: (
      email: 'your@email.com',
      github: 'yourusername',
      linkedin: 'yourprofile',
    ),
    bio: 'เรียนรู้ Dart และ Flutter เพื่อพัฒนาแอปพลิเคชัน',
  );
  
  // แสดง Profile Card
  print('╔' + '═' * 38 + '╗');
  print('║${" PROFILE CARD ".padLeft(28).padRight(38)}║');
  print('╠' + '═' * 38 + '╣');
  print('║ ชื่อ: ${profile.name.padRight(30)}║');
  print('║ อายุ: ${profile.age.toString().padRight(30)}║');
  print('║ อาชีพ: ${profile.profession.padRight(29)}║');
  print('╠' + '═' * 38 + '╣');
  print('║ ทักษะ:${' ' * 31}║');
  for (String skill in profile.skills) {
    print('║   • ${skill.padRight(33)}║');
  }
  print('╠' + '═' * 38 + '╣');
  print('║ ติดต่อ:${' ' * 30}║');
  print('║   📧 ${profile.contact.email.padRight(32)}║');
  print('║   🐙 github: ${profile.contact.github.padRight(25)}║');
  print('╠' + '═' * 38 + '╣');
  print('║ Bio:${' ' * 34}║');
  
  // Word wrap bio
  String bio = profile.bio;
  while (bio.length > 36) {
    int breakAt = bio.lastIndexOf(' ', 36);
    if (breakAt < 0) breakAt = 36;
    print('║ ${bio.substring(0, breakAt).padRight(36)}║');
    bio = bio.substring(breakAt + 1);
  }
  if (bio.isNotEmpty) {
    print('║ ${bio.padRight(36)}║');
  }
  
  print('╚' + '═' * 38 + '╝');
}
```

---

## 🔗 แหล่งข้อมูลเพิ่มเติม

- [Dart Type System](https://dart.dev/language/type-system)
- [Dart Null Safety](https://dart.dev/null-safety)
- [Dart Operators](https://dart.dev/language/operators)
- [Dart Records](https://dart.dev/language/records)
- [DartPad - ทดลองโค้ดออนไลน์](https://dartpad.dev)

---

**← [Part 01 - แนะนำ Dart & Flutter และการติดตั้ง](part-01-introduction-and-setup.md)**

**ต่อไป: [Part 03 - การควบคุมการไหลของโปรแกรม →](part-03-control-flow.md)**
