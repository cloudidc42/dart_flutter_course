# Part 05: Collections: List, Set, Map
## ขั้นตอนที่ 111-140

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ List, Set, Map ได้อย่างคล่องแคล่ว
- เข้าใจ Generics กับ Collections
- ใช้ Collection operations (map, filter, reduce)
- เข้าใจ Immutable vs Mutable collections
- ใช้ Collection If และ Collection For
- สร้าง Custom Iterables

---

## ขั้นตอนที่ 111: List พื้นฐาน

```dart
void main() {
  // ─────────────── สร้าง List ───────────────
  
  // วิธีที่ 1: List literal
  List<int> numbers = [1, 2, 3, 4, 5];
  List<String> names = ['Alice', 'Bob', 'Charlie'];
  
  // วิธีที่ 2: ใช้ constructor
  List<int> empty = [];
  List<String> fromLength = List.filled(5, 'default');  // 5 elements
  List<int> generated = List.generate(5, (i) => i * 2); // [0,2,4,6,8]
  
  // วิธีที่ 3: var (type inference)
  var mixed = [1, 'two', 3.0, true];  // List<Object>
  
  print(numbers);     // [1, 2, 3, 4, 5]
  print(fromLength);  // [default, default, default, default, default]
  print(generated);   // [0, 2, 4, 6, 8]
  
  // ─────────────── การเข้าถึงข้อมูล ───────────────
  print(numbers[0]);           // 1 (first)
  print(numbers[numbers.length - 1]);  // 5 (last)
  print(numbers.first);        // 1
  print(numbers.last);         // 5
  print(numbers.length);       // 5
  print(numbers.isEmpty);      // false
  print(numbers.isNotEmpty);   // true
  
  // ─────────────── การแก้ไข ───────────────
  numbers[0] = 10;      // เปลี่ยน element
  numbers.add(6);       // เพิ่มท้าย
  numbers.addAll([7, 8]); // เพิ่มหลาย
  numbers.insert(0, 0);   // แทรกที่ index
  numbers.insertAll(2, [100, 200]); // แทรกหลาย
  
  print(numbers);
  
  // ─────────────── การลบ ───────────────
  numbers.remove(100);         // ลบค่า 100
  numbers.removeAt(0);         // ลบที่ index 0
  numbers.removeLast();        // ลบตัวสุดท้าย
  numbers.removeRange(1, 3);   // ลบ index 1-2
  numbers.removeWhere((n) => n > 5);  // ลบตามเงื่อนไข
  numbers.clear();             // ล้างทั้งหมด
  
  // ─────────────── ค้นหา ───────────────
  List<String> fruits = ['Apple', 'Banana', 'Cherry', 'Apple'];
  print(fruits.contains('Banana'));     // true
  print(fruits.indexOf('Apple'));       // 0 (ตัวแรก)
  print(fruits.lastIndexOf('Apple'));   // 3 (ตัวสุดท้าย)
  print(fruits.indexWhere((f) => f.startsWith('C'))); // 2
  
  // ─────────────── การเรียงลำดับ ───────────────
  List<int> unsorted = [3, 1, 4, 1, 5, 9, 2, 6];
  unsorted.sort();  // sort in place
  print(unsorted);  // [1, 1, 2, 3, 4, 5, 6, 9]
  
  unsorted.sort((a, b) => b.compareTo(a));  // descending
  print(unsorted);  // [9, 6, 5, 4, 3, 2, 1, 1]
  
  List<String> words = ['banana', 'apple', 'cherry'];
  words.sort((a, b) => a.compareTo(b));
  print(words);  // [apple, banana, cherry]
}
```

---

## ขั้นตอนที่ 112: List Operations ขั้นสูง

```dart
void main() {
  List<int> nums = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
  
  // ─────────────── Slicing ───────────────
  print(nums.sublist(2, 5));    // [3, 4, 5]
  print(nums.sublist(7));       // [8, 9, 10]
  
  // ─────────────── Range ───────────────
  print(nums.getRange(2, 5).toList()); // [3, 4, 5]
  
  // ─────────────── Spread ───────────────
  List<int> a = [1, 2, 3];
  List<int> b = [4, 5, 6];
  List<int> combined = [...a, ...b, 7, 8];
  print(combined);  // [1, 2, 3, 4, 5, 6, 7, 8]
  
  // ─────────────── Reversed ───────────────
  print(nums.reversed.toList());  // [10, 9, 8, 7, 6, 5, 4, 3, 2, 1]
  
  // ─────────────── Set operations ───────────────
  List<int> l1 = [1, 2, 3, 4, 5];
  List<int> l2 = [3, 4, 5, 6, 7];
  
  Set<int> intersection = l1.toSet().intersection(l2.toSet());
  Set<int> union = l1.toSet().union(l2.toSet());
  Set<int> difference = l1.toSet().difference(l2.toSet());
  
  print('Intersection: $intersection');  // {3, 4, 5}
  print('Union: $union');               // {1, 2, 3, 4, 5, 6, 7}
  print('Difference: $difference');     // {1, 2}
  
  // ─────────────── Statistics ───────────────
  print('Max: ${nums.reduce((a, b) => a > b ? a : b)}');  // 10
  print('Min: ${nums.reduce((a, b) => a < b ? a : b)}');  // 1
  print('Sum: ${nums.reduce((a, b) => a + b)}');          // 55
  print('Avg: ${nums.fold<double>(0, (a, b) => a + b) / nums.length}'); // 5.5
  
  // ─────────────── Flatten ───────────────
  List<List<int>> nested = [[1, 2], [3, 4], [5, 6]];
  List<int> flat = nested.expand((list) => list).toList();
  print('Flat: $flat');  // [1, 2, 3, 4, 5, 6]
  
  // ─────────────── Zip (combine two lists) ───────────────
  List<String> keys = ['a', 'b', 'c'];
  List<int> values = [1, 2, 3];
  Map<String, int> zipped = Map.fromIterables(keys, values);
  print('Zipped: $zipped');  // {a: 1, b: 2, c: 3}
  
  // ─────────────── Partition ───────────────
  var (evens, odds) = nums.fold<(List<int>, List<int>)>(
    ([], []),
    (acc, n) => n % 2 == 0 
        ? ([...acc.$1, n], acc.$2) 
        : (acc.$1, [...acc.$2, n]),
  );
  print('Evens: $evens, Odds: $odds');
}
```

---

## ขั้นตอนที่ 113: Collection If และ Collection For

```dart
void main() {
  bool showAdmin = true;
  bool isDarkMode = false;
  List<String> extraItems = ['Settings', 'About'];
  
  // ─────────────── Collection If ───────────────
  List<String> menuItems = [
    'Home',
    'Profile',
    if (showAdmin) 'Admin Panel',      // แสดงถ้า true
    if (isDarkMode) '🌙 Dark' else '☀️ Light', // if-else
    'Logout',
  ];
  
  print(menuItems);
  
  // ─────────────── Collection For ───────────────
  List<int> range = [
    for (int i = 1; i <= 5; i++) i,
  ];
  print(range);  // [1, 2, 3, 4, 5]
  
  // ─────────────── ผสม ───────────────
  List<String> fullMenu = [
    'Home',
    for (String item in extraItems) item,
    if (showAdmin) ...[   // spread ใน if
      'Admin',
      'Reports',
    ],
  ];
  print(fullMenu);
  
  // ─────────────── ใน Flutter Widgets ───────────────
  // นี่คือการใช้งานจริงใน Flutter
  /*
  Column(
    children: [
      Text('Header'),
      for (String item in items) ListTile(title: Text(item)),
      if (isLoggedIn) LogoutButton(),
    ],
  )
  */
  
  // ─────────────── Map Literal กับ Collection For ───────────────
  List<String> colors = ['red', 'green', 'blue'];
  Map<String, int> colorCodes = {
    for (int i = 0; i < colors.length; i++) colors[i]: i,
  };
  print(colorCodes);  // {red: 0, green: 1, blue: 2}
}
```

---

## ขั้นตอนที่ 114: Set

```dart
void main() {
  // ─────────────── สร้าง Set ───────────────
  Set<int> s1 = {1, 2, 3, 4, 5};
  Set<String> names = {'Alice', 'Bob', 'Charlie'};
  
  // Set ไม่มี duplicates
  Set<int> withDups = {1, 2, 2, 3, 3, 3};
  print(withDups);  // {1, 2, 3}
  
  // Empty set ต้องระบุ type (ไม่เช่นนั้น Dart คิดว่าเป็น Map)
  Set<String> empty = {};
  Set<int> emptyTyped = <int>{};
  
  // ─────────────── การใช้งาน ───────────────
  s1.add(6);
  s1.addAll([7, 8, 9]);
  s1.remove(1);
  
  print(s1.contains(5));  // true
  print(s1.length);       // 8
  
  // ─────────────── Set Operations ───────────────
  Set<int> a = {1, 2, 3, 4, 5};
  Set<int> b = {3, 4, 5, 6, 7};
  
  // Union: ทุกอย่างรวมกัน
  Set<int> union = a.union(b);
  print('Union: $union');  // {1, 2, 3, 4, 5, 6, 7}
  
  // Intersection: ที่ซ้ำกัน
  Set<int> inter = a.intersection(b);
  print('Intersection: $inter');  // {3, 4, 5}
  
  // Difference: ใน a แต่ไม่ใน b
  Set<int> diff = a.difference(b);
  print('Difference: $diff');  // {1, 2}
  
  // isSubset
  Set<int> sub = {3, 4};
  print(sub.containsAll(a));      // false
  print(a.containsAll(sub));      // true (sub ⊆ a)
  
  // ─────────────── Set กับ Performance ───────────────
  // Set.contains() เร็วกว่า List.contains() มากสำหรับข้อมูลขนาดใหญ่
  // O(1) vs O(n)
  
  Set<String> largeSet = Set.generate(1000, (i) => 'item$i');
  print(largeSet.contains('item500'));  // true, O(1)
  
  // ─────────────── LinkedHashSet ───────────────
  // รักษาลำดับการ insert
  var linked = <String>{};
  linked.addAll(['banana', 'apple', 'cherry']);
  print(linked);  // {banana, apple, cherry} (รักษาลำดับ)
  
  // ─────────────── Sorted Set ───────────────
  var sorted = SplayTreeSet<int>((a, b) => a.compareTo(b));
  sorted.addAll([5, 3, 8, 1, 9, 2]);
  print(sorted);  // {1, 2, 3, 5, 8, 9}
}
```

---

## ขั้นตอนที่ 115: Map

```dart
void main() {
  // ─────────────── สร้าง Map ───────────────
  
  // Map literal
  Map<String, int> scores = {
    'Alice': 95,
    'Bob': 87,
    'Charlie': 72,
  };
  
  // Empty Map
  Map<String, dynamic> empty = {};
  Map<int, String> emptyTyped = <int, String>{};
  
  // fromIterables
  List<String> keys = ['a', 'b', 'c'];
  List<int> values = [1, 2, 3];
  Map<String, int> fromLists = Map.fromIterables(keys, values);
  
  // fromEntries
  List<MapEntry<String, int>> entries = [
    MapEntry('x', 10),
    MapEntry('y', 20),
  ];
  Map<String, int> fromEntries = Map.fromEntries(entries);
  
  print(scores);
  print(fromLists);
  print(fromEntries);
  
  // ─────────────── การเข้าถึงข้อมูล ───────────────
  print(scores['Alice']);        // 95
  print(scores['Unknown']);      // null (ไม่มี key)
  print(scores.containsKey('Bob'));   // true
  print(scores.containsValue(87));    // true
  print(scores.keys.toList());         // [Alice, Bob, Charlie]
  print(scores.values.toList());       // [95, 87, 72]
  print(scores.entries.toList());      // [Alice: 95, Bob: 87, Charlie: 72]
  print(scores.length);                // 3
  
  // ─────────────── การแก้ไข ───────────────
  scores['Diana'] = 90;          // เพิ่ม
  scores['Alice'] = 100;         // อัพเดต
  scores.remove('Charlie');      // ลบ
  scores.removeWhere((k, v) => v < 88);  // ลบตามเงื่อนไข
  scores.update('Bob', (v) => v + 5);     // อัพเดตค่า
  scores.updateAll((k, v) => v + 1);      // อัพเดตทุก entry
  
  print(scores);
  
  // ─────────────── putIfAbsent ───────────────
  scores.putIfAbsent('Eve', () => 85);   // เพิ่มถ้าไม่มี key
  scores.putIfAbsent('Bob', () => 100);  // ไม่เปลี่ยนเพราะมี Bob แล้ว
  
  // ─────────────── การวนซ้ำ ───────────────
  scores.forEach((name, score) {
    print('$name: $score');
  });
  
  for (var entry in scores.entries) {
    print('${entry.key}: ${entry.value}');
  }
  
  // ─────────────── Map transformations ───────────────
  Map<String, String> gradeMap = scores.map(
    (name, score) => MapEntry(name, score >= 90 ? 'A' : 'B'),
  );
  print(gradeMap);
  
  // ─────────────── Nested Maps ───────────────
  Map<String, Map<String, dynamic>> users = {
    'alice': {'age': 25, 'city': 'Bangkok'},
    'bob': {'age': 30, 'city': 'Chiang Mai'},
  };
  
  print(users['alice']?['city']);  // Bangkok
  print(users['unknown']?['age']); // null (safe access)
}
```

---

## ขั้นตอนที่ 116: Collections ขั้นสูง

```dart
import 'dart:collection';

void main() {
  // ─────────────── Queue ───────────────
  Queue<int> queue = Queue();
  queue.add(1);        // เพิ่มหลัง
  queue.add(2);
  queue.add(3);
  queue.addFirst(0);   // เพิ่มหน้า
  
  print(queue);             // {0, 1, 2, 3}
  print(queue.removeFirst()); // 0 (FIFO)
  print(queue.removeLast()); // 3
  print(queue);             // {1, 2}
  
  // ─────────────── Stack (ใช้ List) ───────────────
  List<int> stack = [];
  stack.add(1);     // push
  stack.add(2);
  stack.add(3);
  
  print(stack.removeLast());  // 3 (LIFO) - pop
  print(stack.last);          // 2 - peek
  
  // ─────────────── LinkedHashMap ───────────────
  // รักษาลำดับการ insert
  LinkedHashMap<String, int> ordered = LinkedHashMap();
  ordered['banana'] = 2;
  ordered['apple'] = 1;
  ordered['cherry'] = 3;
  print(ordered);  // {banana: 2, apple: 1, cherry: 3}
  
  // ─────────────── SplayTreeMap ───────────────
  // เรียงลำดับ key อัตโนมัติ
  SplayTreeMap<String, int> sorted = SplayTreeMap();
  sorted['banana'] = 2;
  sorted['apple'] = 1;
  sorted['cherry'] = 3;
  print(sorted);  // {apple: 1, banana: 2, cherry: 3}
  
  // ─────────────── UnmodifiableListView ───────────────
  List<int> mutable = [1, 2, 3];
  var immutable = List<int>.unmodifiable(mutable);
  // immutable.add(4);  // Error!
  print(immutable);
  
  // ─────────────── Lazy Evaluation ───────────────
  // Iterable ถูก evaluate แบบ lazy (เมื่อถูกเรียกใช้จริง)
  Iterable<int> lazy = Iterable.generate(1000000, (i) => i * 2);
  
  // เฉพาะ 5 ตัวแรกที่ถูก evaluate จริง
  print(lazy.take(5).toList());  // [0, 2, 4, 6, 8]
}
```

---

## ขั้นตอนที่ 117: Collections กับ Null Safety

```dart
void main() {
  // ─────────────── Nullable Collections ───────────────
  List<int>? nullableList;
  print(nullableList?.length);  // null
  print(nullableList?.isEmpty ?? true);  // true
  
  // ─────────────── Nullable Elements ───────────────
  List<String?> withNulls = ['Alice', null, 'Bob', null, 'Charlie'];
  
  // กรอง null ออก
  List<String> noNulls = withNulls
      .where((s) => s != null)
      .cast<String>()
      .toList();
  print(noNulls);  // [Alice, Bob, Charlie]
  
  // หรือใช้ whereType
  List<String> noNulls2 = withNulls.whereType<String>().toList();
  print(noNulls2);  // [Alice, Bob, Charlie]
  
  // ─────────────── Map กับ Null Safety ───────────────
  Map<String, int?> scores = {
    'Alice': 95,
    'Bob': null,  // ยังไม่ได้คะแนน
    'Charlie': 80,
  };
  
  // เข้าถึงค่าที่อาจเป็น null
  int? aliceScore = scores['Alice'];
  int? unknownScore = scores['Unknown'];
  
  print(aliceScore);    // 95
  print(unknownScore);  // null (ไม่มี key)
  
  // Default value
  int aliceOrDefault = scores['Alice'] ?? 0;
  int unknownOrDefault = scores['Unknown'] ?? 0;
  
  // ─────────────── Spread กับ Null ───────────────
  List<int>? maybeList = [1, 2, 3];
  List<int>? noList;
  
  List<int> merged = [
    ...?maybeList,  // ปลอดภัย ถ้า null จะไม่เพิ่ม element
    ...?noList,
    4, 5,
  ];
  print(merged);  // [1, 2, 3, 4, 5]
}
```

---

## ขั้นตอนที่ 118: Sorting และ Comparing

```dart
import 'package:collection/collection.dart'; // ถ้าต้องการ advanced

void main() {
  // ─────────────── Sort พื้นฐาน ───────────────
  List<int> nums = [5, 2, 8, 1, 9, 3];
  nums.sort();  // ascending
  print('Ascending: $nums');
  
  nums.sort((a, b) => b.compareTo(a));  // descending
  print('Descending: $nums');
  
  // ─────────────── Sort Objects ───────────────
  List<Map<String, dynamic>> people = [
    {'name': 'Charlie', 'age': 30},
    {'name': 'Alice', 'age': 25},
    {'name': 'Bob', 'age': 35},
    {'name': 'Diana', 'age': 25},
  ];
  
  // เรียงตามอายุ
  people.sort((a, b) => (a['age'] as int).compareTo(b['age'] as int));
  print('By age: ${people.map((p) => "${p['name']}:${p['age']}").toList()}');
  
  // เรียงตามอายุ แล้วตามชื่อ
  people.sort((a, b) {
    int ageCompare = (a['age'] as int).compareTo(b['age'] as int);
    if (ageCompare != 0) return ageCompare;
    return (a['name'] as String).compareTo(b['name'] as String);
  });
  print('By age+name: ${people.map((p) => "${p['name']}:${p['age']}").toList()}');
  
  // ─────────────── Stable Sort ───────────────
  // List.sort() ใน Dart เป็น stable sort
  
  // ─────────────── Binary Search ───────────────
  List<int> sorted = [1, 3, 5, 7, 9, 11, 13];
  
  // หา index ด้วย binary search
  int findIndex(List<int> list, int target) {
    int low = 0, high = list.length - 1;
    while (low <= high) {
      int mid = (low + high) ~/ 2;
      if (list[mid] == target) return mid;
      if (list[mid] < target) low = mid + 1;
      else high = mid - 1;
    }
    return -1;
  }
  
  print(findIndex(sorted, 7));   // 3
  print(findIndex(sorted, 4));   // -1
}
```

---

## ขั้นตอนที่ 119: Collections ขั้นสูง - Grouping และ Aggregation

```dart
void main() {
  List<Map<String, dynamic>> students = [
    {'name': 'Alice', 'grade': 'A', 'score': 95},
    {'name': 'Bob', 'grade': 'B', 'score': 82},
    {'name': 'Charlie', 'grade': 'A', 'score': 91},
    {'name': 'Diana', 'grade': 'C', 'score': 67},
    {'name': 'Eve', 'grade': 'B', 'score': 78},
    {'name': 'Frank', 'grade': 'A', 'score': 88},
  ];
  
  // ─────────────── Group By ───────────────
  Map<String, List<Map<String, dynamic>>> grouped = {};
  
  for (var student in students) {
    String grade = student['grade'] as String;
    grouped.putIfAbsent(grade, () => []).add(student);
  }
  
  print('Grouped by grade:');
  grouped.forEach((grade, gradeStudents) {
    print('  Grade $grade:');
    for (var s in gradeStudents) {
      print('    - ${s['name']}: ${s['score']}');
    }
  });
  
  // ─────────────── Aggregate ───────────────
  print('\nStatistics per grade:');
  grouped.forEach((grade, gradeStudents) {
    List<int> scores = gradeStudents.map((s) => s['score'] as int).toList();
    double avg = scores.fold(0, (sum, s) => sum + s) / scores.length;
    int max = scores.reduce((a, b) => a > b ? a : b);
    int min = scores.reduce((a, b) => a < b ? a : b);
    
    print('  Grade $grade: avg=${avg.toStringAsFixed(1)}, max=$max, min=$min');
  });
  
  // ─────────────── Sort by grouped ───────────────
  var sortedGrades = grouped.entries.toList()
    ..sort((a, b) => a.key.compareTo(b.key));
  
  print('\nSorted by grade:');
  for (var entry in sortedGrades) {
    print('  ${entry.key}: ${entry.value.map((s) => s['name']).toList()}');
  }
  
  // ─────────────── Top N ───────────────
  List<Map<String, dynamic>> top3 = List.from(students)
    ..sort((a, b) => (b['score'] as int).compareTo(a['score'] as int));
  
  print('\nTop 3 students:');
  for (int i = 0; i < 3; i++) {
    print('  ${i + 1}. ${top3[i]['name']}: ${top3[i]['score']}');
  }
}
```

---

## ขั้นตอนที่ 120: Collections Performance

```dart
void main() {
  // ─────────────── List vs Set Performance ───────────────
  
  // List.contains: O(n)
  // Set.contains: O(1)
  
  List<int> largeList = List.generate(100000, (i) => i);
  Set<int> largeSet = Set.generate(100000, (i) => i);
  
  // สำหรับ lookup บ่อยๆ ใช้ Set หรือ Map
  
  // ─────────────── Map vs List of Objects ───────────────
  // Map lookup: O(1)
  // List search: O(n)
  
  List<Map<String, dynamic>> userList = List.generate(
    10000,
    (i) => {'id': i, 'name': 'User $i'},
  );
  
  Map<int, Map<String, dynamic>> userMap = {
    for (var user in userList) user['id'] as int: user,
  };
  
  // Map lookup เร็วกว่า
  print(userMap[5000]?['name']);  // User 5000
  
  // ─────────────── Memory efficient collections ───────────────
  
  // ใช้ Iterable แทน List เมื่อไม่จำเป็นต้องเก็บทั้งหมด
  Iterable<int> lazyRange = Iterable.generate(1000000, (i) => i);
  
  // ใช้เฉพาะส่วนที่ต้องการ
  int sum = lazyRange.take(100).fold(0, (a, b) => a + b);
  print('Sum of first 100: $sum');  // 4950
  
  // ─────────────── const collections ───────────────
  // const lists/sets/maps ไม่สามารถ modify ได้ และ compile-time constant
  const List<String> constList = ['a', 'b', 'c'];
  const Map<String, int> constMap = {'one': 1, 'two': 2};
  const Set<int> constSet = {1, 2, 3};
  
  // constList.add('d');  // Error!
  print(constList);
}
```

---

## ขั้นตอนที่ 121-130: โปรเจกต์ - Inventory System

```dart
// inventory_system.dart

enum Category { food, electronics, clothing, books }

class Product {
  final String id;
  final String name;
  final double price;
  final int quantity;
  final Category category;
  
  const Product({
    required this.id,
    required this.name,
    required this.price,
    required this.quantity,
    required this.category,
  });
  
  Product copyWith({
    String? id,
    String? name,
    double? price,
    int? quantity,
    Category? category,
  }) {
    return Product(
      id: id ?? this.id,
      name: name ?? this.name,
      price: price ?? this.price,
      quantity: quantity ?? this.quantity,
      category: category ?? this.category,
    );
  }
  
  @override
  String toString() => '$name (฿${price.toStringAsFixed(2)}) x$quantity';
}

class Inventory {
  final Map<String, Product> _products = {};
  
  void addProduct(Product product) {
    _products[product.id] = product;
  }
  
  void removeProduct(String id) {
    _products.remove(id);
  }
  
  void updateQuantity(String id, int delta) {
    final product = _products[id];
    if (product == null) throw ArgumentError('Product not found: $id');
    
    int newQty = product.quantity + delta;
    if (newQty < 0) throw StateError('Insufficient stock');
    
    _products[id] = product.copyWith(quantity: newQty);
  }
  
  Product? getProduct(String id) => _products[id];
  
  List<Product> get allProducts => _products.values.toList();
  
  List<Product> getByCategory(Category category) {
    return _products.values
        .where((p) => p.category == category)
        .toList();
  }
  
  List<Product> getLowStock({int threshold = 5}) {
    return _products.values
        .where((p) => p.quantity <= threshold)
        .toList()
      ..sort((a, b) => a.quantity.compareTo(b.quantity));
  }
  
  Map<Category, List<Product>> groupByCategory() {
    Map<Category, List<Product>> grouped = {};
    for (var product in _products.values) {
      grouped.putIfAbsent(product.category, () => []).add(product);
    }
    return grouped;
  }
  
  double get totalValue {
    return _products.values
        .fold(0, (sum, p) => sum + p.price * p.quantity);
  }
  
  List<Product> searchByName(String query) {
    String q = query.toLowerCase();
    return _products.values
        .where((p) => p.name.toLowerCase().contains(q))
        .toList();
  }
  
  List<Product> sortBy(String field, {bool ascending = true}) {
    List<Product> sorted = List.from(_products.values);
    
    sorted.sort((a, b) {
      int compare = switch (field) {
        'name' => a.name.compareTo(b.name),
        'price' => a.price.compareTo(b.price),
        'quantity' => a.quantity.compareTo(b.quantity),
        _ => 0,
      };
      return ascending ? compare : -compare;
    });
    
    return sorted;
  }
  
  Map<String, dynamic> getStats() {
    if (_products.isEmpty) return {};
    
    List<Product> products = allProducts;
    List<double> prices = products.map((p) => p.price).toList();
    
    return {
      'total_products': products.length,
      'total_value': totalValue,
      'avg_price': prices.fold(0.0, (sum, p) => sum + p) / prices.length,
      'max_price': prices.reduce((a, b) => a > b ? a : b),
      'min_price': prices.reduce((a, b) => a < b ? a : b),
      'low_stock_count': getLowStock().length,
    };
  }
}

void main() {
  Inventory inventory = Inventory();
  
  // เพิ่มสินค้า
  List<Product> products = [
    Product(id: 'F001', name: 'ข้าวสาร 5 กก.', price: 120.0, quantity: 50, category: Category.food),
    Product(id: 'F002', name: 'น้ำตาลทราย', price: 45.0, quantity: 30, category: Category.food),
    Product(id: 'F003', name: 'น้ำมันพืช', price: 89.0, quantity: 3, category: Category.food),
    Product(id: 'E001', name: 'หูฟัง Bluetooth', price: 1500.0, quantity: 15, category: Category.electronics),
    Product(id: 'E002', name: 'สายชาร์จ USB-C', price: 299.0, quantity: 2, category: Category.electronics),
    Product(id: 'C001', name: 'เสื้อยืด M', price: 199.0, quantity: 25, category: Category.clothing),
    Product(id: 'B001', name: 'Dart Programming', price: 450.0, quantity: 8, category: Category.books),
  ];
  
  for (var product in products) {
    inventory.addProduct(product);
  }
  
  // แสดงรายการทั้งหมด
  print('📦 สินค้าทั้งหมด:');
  print('─' * 50);
  for (var p in inventory.sortBy('name')) {
    print('  [${p.id}] ${p.toString()}');
  }
  
  // สินค้าที่ใกล้หมด
  print('\n⚠️ สินค้าที่ใกล้หมด (≤5):');
  for (var p in inventory.getLowStock()) {
    print('  ❗ ${p.name}: เหลือ ${p.quantity} ชิ้น');
  }
  
  // แยกตาม category
  print('\n📂 แยกตามประเภท:');
  inventory.groupByCategory().forEach((category, products) {
    print('  ${category.name.toUpperCase()}:');
    for (var p in products) {
      print('    - ${p.name}: ฿${p.price.toStringAsFixed(0)}');
    }
  });
  
  // สถิติ
  print('\n📊 สรุปสต็อก:');
  var stats = inventory.getStats();
  print('  จำนวนสินค้า: ${stats['total_products']}');
  print('  มูลค่ารวม: ฿${(stats['total_value'] as double).toStringAsFixed(2)}');
  print('  ราคาเฉลี่ย: ฿${(stats['avg_price'] as double).toStringAsFixed(2)}');
  print('  สินค้าใกล้หมด: ${stats['low_stock_count']} รายการ');
  
  // อัพเดตสต็อก
  print('\n📝 อัพเดตสต็อก...');
  inventory.updateQuantity('E002', 10);  // เพิ่ม 10 ชิ้น
  inventory.updateQuantity('F003', 20);  // เพิ่ม 20 ชิ้น
  
  print('สต็อกใหม่: ${inventory.getProduct('E002')}');
  print('สต็อกใหม่: ${inventory.getProduct('F003')}');
  
  // ค้นหา
  print('\n🔍 ค้นหา "น้ำ":');
  for (var p in inventory.searchByName('น้ำ')) {
    print('  ${p.name}');
  }
}
```

---

## ขั้นตอนที่ 131-140: สรุปและ Challenge

### Challenge: Data Analysis

```dart
void main() {
  List<Map<String, dynamic>> salesData = [
    {'date': '2024-01', 'product': 'A', 'amount': 50000.0, 'region': 'North'},
    {'date': '2024-01', 'product': 'B', 'amount': 30000.0, 'region': 'South'},
    {'date': '2024-01', 'product': 'A', 'amount': 45000.0, 'region': 'East'},
    {'date': '2024-02', 'product': 'B', 'amount': 60000.0, 'region': 'North'},
    {'date': '2024-02', 'product': 'C', 'amount': 25000.0, 'region': 'South'},
    {'date': '2024-02', 'product': 'A', 'amount': 55000.0, 'region': 'West'},
    {'date': '2024-03', 'product': 'C', 'amount': 40000.0, 'region': 'North'},
    {'date': '2024-03', 'product': 'A', 'amount': 70000.0, 'region': 'East'},
    {'date': '2024-03', 'product': 'B', 'amount': 35000.0, 'region': 'West'},
  ];
  
  // 1. ยอดขายรวมต่อเดือน
  Map<String, double> monthlyTotal = {};
  for (var sale in salesData) {
    String month = sale['date'] as String;
    monthlyTotal[month] = (monthlyTotal[month] ?? 0) + (sale['amount'] as double);
  }
  
  print('ยอดขายรายเดือน:');
  monthlyTotal.entries.toList()
    ..sort((a, b) => a.key.compareTo(b.key))
    ..forEach((e) => print('  ${e.key}: ฿${e.value.toStringAsFixed(0)}'));
  
  // 2. สินค้าขายดีที่สุด
  Map<String, double> productTotal = {};
  for (var sale in salesData) {
    String product = sale['product'] as String;
    productTotal[product] = (productTotal[product] ?? 0) + (sale['amount'] as double);
  }
  
  var topProduct = productTotal.entries
      .reduce((a, b) => a.value > b.value ? a : b);
  
  print('\nสินค้าขายดีที่สุด: ${topProduct.key} (฿${topProduct.value.toStringAsFixed(0)})');
  
  // 3. ยอดขายต่อ region
  Map<String, double> regionTotal = {};
  for (var sale in salesData) {
    String region = sale['region'] as String;
    regionTotal[region] = (regionTotal[region] ?? 0) + (sale['amount'] as double);
  }
  
  print('\nยอดขายต่อ Region:');
  regionTotal.entries.toList()
    ..sort((a, b) => b.value.compareTo(a.value))
    ..forEach((e) => print('  ${e.key}: ฿${e.value.toStringAsFixed(0)}'));
  
  // 4. Growth rate
  List<String> months = monthlyTotal.keys.toList()..sort();
  print('\nอัตราการเติบโต:');
  for (int i = 1; i < months.length; i++) {
    double prev = monthlyTotal[months[i-1]]!;
    double curr = monthlyTotal[months[i]]!;
    double growth = (curr - prev) / prev * 100;
    print('  ${months[i-1]} → ${months[i]}: ${growth >= 0 ? '+' : ''}${growth.toStringAsFixed(1)}%');
  }
}
```

### สรุป Part 05

```
✅ ขั้นตอนที่ 111: List พื้นฐาน
✅ ขั้นตอนที่ 112: List Operations ขั้นสูง
✅ ขั้นตอนที่ 113: Collection If และ Collection For
✅ ขั้นตอนที่ 114: Set
✅ ขั้นตอนที่ 115: Map
✅ ขั้นตอนที่ 116: Collections ขั้นสูง (Queue, Stack, LinkedHashMap)
✅ ขั้นตอนที่ 117: Collections กับ Null Safety
✅ ขั้นตอนที่ 118: Sorting และ Comparing
✅ ขั้นตอนที่ 119: Grouping และ Aggregation
✅ ขั้นตอนที่ 120: Performance
✅ ขั้นตอนที่ 121-130: โปรเจกต์ - Inventory System
✅ ขั้นตอนที่ 131-140: Challenge - Data Analysis
```

---

**← [Part 04 - Functions และ Scope](part-04-functions-and-scope.md)**

**ต่อไป: [Part 06 - Object-Oriented Programming →](part-06-oop-basics.md)**
