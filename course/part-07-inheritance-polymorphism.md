# Part 07: Inheritance และ Polymorphism
## ขั้นตอนที่ 181-210

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Inheritance และ extends keyword
- ใช้ super keyword
- เข้าใจ Method Overriding
- ใช้ Polymorphism อย่างมีประสิทธิภาพ
- เข้าใจ sealed classes
- ใช้ Generic Classes
- เข้าใจ Covariance และ Contravariance

---

## ขั้นตอนที่ 181: Inheritance พื้นฐาน

```dart
// ─────────────── Base Class (Parent/Super) ───────────────
class Animal {
  String name;
  int age;
  String? species;
  
  Animal(this.name, this.age, {this.species});
  
  void breathe() => print('$name กำลังหายใจ');
  
  void eat(String food) => print('$name กินอาหาร: $food');
  
  void makeSound() => print('$name ส่งเสียง...');
  
  @override
  String toString() => '$name (${species ?? 'Unknown'}, อายุ $age)';
}

// ─────────────── Derived Class (Child/Sub) ───────────────
class Dog extends Animal {
  String breed;
  
  // เรียก super constructor
  Dog(String name, int age, this.breed)
      : super(name, age, species: 'สุนัข');
  
  // Override method
  @override
  void makeSound() {
    print('$name: โฮ่ง โฮ่ง!');
  }
  
  // เพิ่ม method ใหม่
  void fetch(String item) {
    print('$name วิ่งไปเอา $item มา!');
  }
  
  @override
  String toString() => '$name ($breed, อายุ $age)';
}

class Cat extends Animal {
  bool isIndoor;
  
  Cat(String name, int age, {this.isIndoor = true})
      : super(name, age, species: 'แมว');
  
  @override
  void makeSound() {
    print('$name: เมี้ยว!');
  }
  
  void purr() {
    print('$name กำลัง purr...');
  }
}

class Bird extends Animal {
  double wingspan;
  bool canFly;
  
  Bird(String name, int age, this.wingspan, {this.canFly = true})
      : super(name, age, species: 'นก');
  
  @override
  void makeSound() {
    print('$name: จิ๊บ จิ๊บ!');
  }
  
  void fly() {
    if (canFly) {
      print('$name กำลังบิน (ปีกกว้าง ${wingspan}cm)');
    } else {
      print('$name บินไม่ได้');
    }
  }
}

void main() {
  Dog dog = Dog('บัดดี้', 3, 'Labrador');
  Cat cat = Cat('มิมิ', 5);
  Bird bird = Bird('ทวีตตี้', 2, 25.0);
  
  dog.breathe();
  dog.eat('กระดูก');
  dog.makeSound();
  dog.fetch('ลูกบอล');
  
  print('');
  
  cat.makeSound();
  cat.purr();
  cat.eat('ปลา');
  
  print('');
  
  bird.makeSound();
  bird.fly();
  
  // Polymorphism
  print('\n--- Polymorphism ---');
  List<Animal> animals = [dog, cat, bird];
  
  for (Animal animal in animals) {
    print(animal);
    animal.makeSound();
    print('');
  }
}
```

---

## ขั้นตอนที่ 182: super keyword

```dart
class Vehicle {
  String brand;
  int year;
  double engineSize;
  
  Vehicle(this.brand, this.year, this.engineSize);
  
  String get description => '$brand ($year), ${engineSize}L engine';
  
  void start() => print('$brand เครื่องยนต์สตาร์ท...');
  void stop() => print('$brand หยุด');
  
  double calculateFuelCost(double km) {
    return km * 0.08 * 40; // 8L/100km at 40 บาท/L
  }
}

class Car extends Vehicle {
  int doors;
  String bodyType;
  
  Car(String brand, int year, double engineSize, this.doors, this.bodyType)
      : super(brand, year, engineSize);
  
  @override
  String get description => 
      '${super.description}, $doors doors, $bodyType';
  
  @override
  void start() {
    super.start();  // เรียก parent method
    print('$brand พร้อมขับ');
  }
  
  @override
  double calculateFuelCost(double km) {
    return super.calculateFuelCost(km) * 0.9;  // รถยนต์ประหยัดกว่า
  }
}

class ElectricCar extends Car {
  double batteryCapacity;
  int range;
  
  ElectricCar(
    String brand,
    int year,
    this.batteryCapacity,
    this.range,
    int doors,
    String bodyType,
  ) : super(brand, year, 0, doors, bodyType);
  
  @override
  String get description =>
      '$brand ($year), EV, ${batteryCapacity}kWh battery, '
      '${range}km range, $doors doors';
  
  @override
  void start() {
    print('$brand ระบบไฟฟ้าพร้อมทำงาน... 🔋');
  }
  
  @override
  double calculateFuelCost(double km) {
    // ค่าไฟแทนน้ำมัน
    return (km / range) * batteryCapacity * 5; // 5 บาท/kWh
  }
  
  void charge() {
    print('$brand กำลังชาร์จ...');
  }
}

void main() {
  var civic = Car('Honda Civic', 2023, 1.5, 4, 'Sedan');
  var ev = ElectricCar('Tesla Model 3', 2024, 75, 500, 4, 'Sedan');
  
  print(civic.description);
  civic.start();
  print('ค่าน้ำมัน 100 กม.: ฿${civic.calculateFuelCost(100).toStringAsFixed(2)}');
  
  print('');
  
  print(ev.description);
  ev.start();
  print('ค่าไฟ 100 กม.: ฿${ev.calculateFuelCost(100).toStringAsFixed(2)}');
  ev.charge();
}
```

---

## ขั้นตอนที่ 183: Method Overriding และ @override

```dart
class Shape {
  // ─────────────── เมธอดที่ Subclass ควร Override ───────────────
  double get area => 0;
  double get perimeter => 0;
  String get name => 'Shape';
  
  // ─────────────── Template method ───────────────
  void printInfo() {
    print('$name:');
    print('  Area: ${area.toStringAsFixed(4)}');
    print('  Perimeter: ${perimeter.toStringAsFixed(4)}');
  }
  
  // ─────────────── final: ห้าม Override ───────────────
  final String type = 'Geometric Shape';
  
  void describe() {
    printInfo();  // เรียก method ที่อาจถูก override
  }
}

class Circle extends Shape {
  final double radius;
  Circle(this.radius);
  
  @override
  double get area => 3.14159 * radius * radius;
  
  @override
  double get perimeter => 2 * 3.14159 * radius;
  
  @override
  String get name => 'Circle (r=$radius)';
}

class Square extends Shape {
  final double side;
  Square(this.side);
  
  @override
  double get area => side * side;
  
  @override
  double get perimeter => 4 * side;
  
  @override
  String get name => 'Square (s=$side)';
}

// ─────────────── Late binding / Dynamic dispatch ───────────────
void main() {
  List<Shape> shapes = [
    Circle(5),
    Square(4),
    Circle(3),
    Square(7),
  ];
  
  // Polymorphic method call
  for (Shape s in shapes) {
    s.describe();  // เรียก method ที่ถูก override โดยอัตโนมัติ
    print('');
  }
  
  // Sort by area (Polymorphism ใน action)
  shapes.sort((a, b) => a.area.compareTo(b.area));
  
  print('เรียงตามพื้นที่:');
  for (Shape s in shapes) {
    print('  ${s.name}: ${s.area.toStringAsFixed(2)}');
  }
}
```

---

## ขั้นตอนที่ 184: Sealed Classes (Dart 3.0+)

```dart
// ─────────────── sealed class ───────────────
// ทุก subclass ต้องอยู่ใน file เดียวกัน
// ทำให้ switch exhaustive (ต้องครอบคลุมทุก case)

sealed class Shape2 {}

class Circle2 extends Shape2 {
  final double radius;
  Circle2(this.radius);
}

class Rectangle2 extends Shape2 {
  final double width;
  final double height;
  Rectangle2(this.width, this.height);
}

class Triangle2 extends Shape2 {
  final double base;
  final double height;
  Triangle2(this.base, this.height);
}

// ─────────────── Exhaustive switch กับ sealed ───────────────
double calculateArea(Shape2 shape) {
  return switch (shape) {
    Circle2(:var radius) => 3.14159 * radius * radius,
    Rectangle2(:var width, :var height) => width * height,
    Triangle2(:var base, :var height) => 0.5 * base * height,
    // ไม่ต้องมี default! Dart รู้ว่า Shape2 มีแค่ 3 subclasses
  };
}

String describeShape(Shape2 shape) {
  return switch (shape) {
    Circle2(radius: var r) => 'วงกลม รัศมี $r',
    Rectangle2(width: var w, height: var h) => 'สี่เหลี่ยม ${w}x$h',
    Triangle2(base: var b, height: var h) => 'สามเหลี่ยม base=$b, h=$h',
  };
}

// ─────────────── sealed กับ Union Types ───────────────
sealed class Result<T> {}

class Ok<T> extends Result<T> {
  final T value;
  Ok(this.value);
}

class Err<T> extends Result<T> {
  final String message;
  Err(this.message);
}

Result<int> divide(int a, int b) {
  if (b == 0) return Err('หารด้วยศูนย์ไม่ได้');
  return Ok(a ~/ b);
}

void main() {
  List<Shape2> shapes = [
    Circle2(5),
    Rectangle2(4, 6),
    Triangle2(3, 4),
  ];
  
  for (var shape in shapes) {
    print('${describeShape(shape)}: area = ${calculateArea(shape).toStringAsFixed(2)}');
  }
  
  // Result type
  print('\nDivision results:');
  var results = [divide(10, 3), divide(10, 0), divide(15, 5)];
  
  for (var result in results) {
    switch (result) {
      case Ok(:var value):
        print('  ✅ = $value');
      case Err(:var message):
        print('  ❌ Error: $message');
    }
  }
}
```

---

## ขั้นตอนที่ 185: Generics

```dart
// ─────────────── Generic Class ───────────────
class Box<T> {
  T? _content;
  
  void put(T item) {
    _content = item;
    print('วางใน Box: $item');
  }
  
  T? take() {
    T? item = _content;
    _content = null;
    return item;
  }
  
  bool get isEmpty => _content == null;
  
  @override
  String toString() => 'Box<${T.toString()}>(${_content ?? 'empty'})';
}

// ─────────────── Generic กับ Constraint (extends) ───────────────
class NumberBox<T extends num> {
  final T value;
  NumberBox(this.value);
  
  T doubled() => (value * 2) as T;
  bool isPositive() => value > 0;
}

// ─────────────── Generic Stack ───────────────
class Stack<T> {
  final List<T> _items = [];
  
  void push(T item) => _items.add(item);
  
  T pop() {
    if (isEmpty) throw StateError('Stack is empty');
    return _items.removeLast();
  }
  
  T get peek {
    if (isEmpty) throw StateError('Stack is empty');
    return _items.last;
  }
  
  bool get isEmpty => _items.isEmpty;
  int get length => _items.length;
  
  @override
  String toString() => 'Stack($_items)';
}

// ─────────────── Generic Pair ───────────────
class Pair<A, B> {
  final A first;
  final B second;
  
  const Pair(this.first, this.second);
  
  Pair<B, A> swap() => Pair(second, first);
  
  @override
  String toString() => '($first, $second)';
}

// ─────────────── Generic Functions ───────────────
T identity<T>(T value) => value;

List<T> repeat<T>(T value, int times) {
  return List.generate(times, (_) => value);
}

Pair<T, R> zip<T, R>(T a, R b) => Pair(a, b);

// ─────────────── Bounded Generics ───────────────
class SortedList<T extends Comparable<T>> {
  final List<T> _items = [];
  
  void add(T item) {
    _items.add(item);
    _items.sort();
  }
  
  T get min => _items.first;
  T get max => _items.last;
  
  @override
  String toString() => _items.toString();
}

void main() {
  // Box
  var intBox = Box<int>();
  var strBox = Box<String>();
  
  intBox.put(42);
  strBox.put('Hello');
  
  print(intBox);
  print(strBox);
  print('Int: ${intBox.take()}');
  
  // NumberBox
  var numBox = NumberBox<int>(5);
  var doubleBox = NumberBox<double>(3.14);
  
  print('doubled: ${numBox.doubled()}');
  print('isPositive: ${doubleBox.isPositive()}');
  
  // Stack
  Stack<String> stack = Stack();
  stack.push('A');
  stack.push('B');
  stack.push('C');
  
  print('Stack: $stack');
  print('Pop: ${stack.pop()}');
  print('Peek: ${stack.peek}');
  
  // Pair
  Pair<String, int> pair = Pair('age', 25);
  print(pair);
  print(pair.swap());
  
  // Generic functions
  print(identity<String>('Hello'));
  print(repeat(0, 5));
  print(zip('Alice', 95));
  
  // SortedList
  SortedList<int> sorted = SortedList();
  sorted.add(5);
  sorted.add(2);
  sorted.add(8);
  sorted.add(1);
  
  print('Sorted: $sorted');
  print('Min: ${sorted.min}, Max: ${sorted.max}');
}
```

---

## ขั้นตอนที่ 186-190: โปรเจกต์ - Zoo Management System

```dart
// zoo_management.dart

// ─────────────── Enums ───────────────
enum Diet { carnivore, herbivore, omnivore }
enum ConservationStatus { leastConcern, nearThreatened, vulnerable, endangered, criticallyEndangered }

// ─────────────── Base Animal Class ───────────────
abstract class ZooAnimal {
  final String id;
  final String name;
  final String species;
  final int age;
  final double weight;
  final Diet diet;
  final ConservationStatus status;
  
  ZooAnimal({
    required this.id,
    required this.name,
    required this.species,
    required this.age,
    required this.weight,
    required this.diet,
    required this.status,
  });
  
  // Abstract methods
  String get sound;
  String get habitat;
  double get dailyFoodKg;
  
  // Template methods
  void makeSound() => print('$name: $sound');
  
  void eat() {
    print('$name กิน ${dailyFoodKg.toStringAsFixed(1)} กก. (${diet.name})');
  }
  
  String get statusEmoji {
    return switch (status) {
      ConservationStatus.leastConcern => '🟢',
      ConservationStatus.nearThreatened => '🟡',
      ConservationStatus.vulnerable => '🟠',
      ConservationStatus.endangered => '🔴',
      ConservationStatus.criticallyEndangered => '⚫',
    };
  }
  
  @override
  String toString() {
    return '$statusEmoji $name ($species, ${age}ปี, ${weight}กก.)';
  }
}

// ─────────────── Specific Animals ───────────────
class Lion extends ZooAnimal {
  final String pride;
  
  Lion({
    required String id,
    required String name,
    required int age,
    required double weight,
    required this.pride,
  }) : super(
    id: id,
    name: name,
    species: 'Panthera leo',
    age: age,
    weight: weight,
    diet: Diet.carnivore,
    status: ConservationStatus.vulnerable,
  );
  
  @override
  String get sound => 'ROARRRR!!! 🦁';
  
  @override
  String get habitat => 'African Savanna';
  
  @override
  double get dailyFoodKg => weight * 0.05; // 5% of body weight
  
  void hunt() => print('$name กำลังล่า...');
}

class Elephant extends ZooAnimal {
  final String herd;
  final double tuskLength;
  
  Elephant({
    required String id,
    required String name,
    required int age,
    required double weight,
    required this.herd,
    this.tuskLength = 0,
  }) : super(
    id: id,
    name: name,
    species: 'Loxodonta africana',
    age: age,
    weight: weight,
    diet: Diet.herbivore,
    status: ConservationStatus.vulnerable,
  );
  
  @override
  String get sound => 'TRUMPETTTTT! 🐘';
  
  @override
  String get habitat => 'African Savanna';
  
  @override
  double get dailyFoodKg => weight * 0.04; // 4% of body weight
  
  void bathe() => print('$name กำลังอาบน้ำ 🛁');
}

class Penguin extends ZooAnimal {
  final String colony;
  final bool canFly = false;
  
  Penguin({
    required String id,
    required String name,
    required int age,
    required double weight,
    required this.colony,
  }) : super(
    id: id,
    name: name,
    species: 'Spheniscidae',
    age: age,
    weight: weight,
    diet: Diet.carnivore,
    status: ConservationStatus.leastConcern,
  );
  
  @override
  String get sound => 'Squeak squeak! 🐧';
  
  @override
  String get habitat => 'Antarctic';
  
  @override
  double get dailyFoodKg => 0.5;
  
  void swim() => print('$name ว่ายน้ำได้เร็วมาก!');
}

// ─────────────── Zoo Enclosure ───────────────
class Enclosure<T extends ZooAnimal> {
  final String id;
  final String name;
  final int maxCapacity;
  final List<T> _animals = [];
  
  Enclosure({
    required this.id,
    required this.name,
    required this.maxCapacity,
  });
  
  bool get isFull => _animals.length >= maxCapacity;
  int get occupancy => _animals.length;
  List<T> get animals => List.unmodifiable(_animals);
  
  void addAnimal(T animal) {
    if (isFull) throw StateError('$name เต็มแล้ว!');
    _animals.add(animal);
    print('✅ เพิ่ม ${animal.name} เข้า $name');
  }
  
  void removeAnimal(String id) {
    _animals.removeWhere((a) => a.id == id);
  }
  
  void feedAll() {
    print('\n🍖 เวลาให้อาหาร $name:');
    for (var animal in _animals) {
      animal.eat();
    }
  }
  
  double get totalDailyFood {
    return _animals.fold(0, (sum, a) => sum + a.dailyFoodKg);
  }
}

// ─────────────── Zoo Class ───────────────
class Zoo {
  final String name;
  final Map<String, Enclosure> _enclosures = {};
  
  Zoo(this.name);
  
  void addEnclosure(Enclosure enclosure) {
    _enclosures[enclosure.id] = enclosure;
  }
  
  Enclosure? getEnclosure(String id) => _enclosures[id];
  
  int get totalAnimals {
    return _enclosures.values.fold(0, (sum, e) => sum + e.occupancy);
  }
  
  double get totalDailyFood {
    return _enclosures.values.fold(0, (sum, e) => sum + e.totalDailyFood);
  }
  
  void feedAll() {
    print('\n🍽️ ======= เวลาให้อาหารทั่วสวนสัตว์ =======');
    for (var enclosure in _enclosures.values) {
      enclosure.feedAll();
    }
  }
  
  void printReport() {
    print('\n🦁 ======= รายงาน ${name} =======');
    print('จำนวนสัตว์ทั้งหมด: $totalAnimals ตัว');
    print('อาหารที่ต้องการต่อวัน: ${totalDailyFood.toStringAsFixed(1)} กก.');
    
    for (var enclosure in _enclosures.values) {
      print('\n📍 ${enclosure.name} (${enclosure.occupancy}/${enclosure.maxCapacity}):');
      for (var animal in enclosure.animals) {
        print('  $animal');
      }
    }
  }
}

void main() {
  Zoo zoo = Zoo('สวนสัตว์กรุงเทพ');
  
  // สร้าง Enclosures
  Enclosure<Lion> lionEnclosure = Enclosure(
    id: 'ENC001',
    name: 'อาณาจักรสิงโต',
    maxCapacity: 5,
  );
  
  Enclosure<Elephant> elephantEnclosure = Enclosure(
    id: 'ENC002',
    name: 'บ้านช้าง',
    maxCapacity: 4,
  );
  
  Enclosure<Penguin> penguinEnclosure = Enclosure(
    id: 'ENC003',
    name: 'โลกเพนกวิน',
    maxCapacity: 20,
  );
  
  zoo.addEnclosure(lionEnclosure);
  zoo.addEnclosure(elephantEnclosure);
  zoo.addEnclosure(penguinEnclosure);
  
  // เพิ่มสัตว์
  lionEnclosure.addAnimal(Lion(
    id: 'L001', name: 'ซิมบา', age: 5, weight: 190, pride: 'Alpha',
  ));
  lionEnclosure.addAnimal(Lion(
    id: 'L002', name: 'นาลา', age: 4, weight: 130, pride: 'Alpha',
  ));
  
  elephantEnclosure.addAnimal(Elephant(
    id: 'E001', name: 'จัมโบ้', age: 15, weight: 5000, herd: 'Main', tuskLength: 120,
  ));
  
  penguinEnclosure.addAnimal(Penguin(id: 'P001', name: 'พิต้า', age: 2, weight: 4, colony: 'Alpha'));
  penguinEnclosure.addAnimal(Penguin(id: 'P002', name: 'โพโล', age: 3, weight: 5, colony: 'Alpha'));
  penguinEnclosure.addAnimal(Penguin(id: 'P003', name: 'เพิ้ล', age: 1, weight: 3, colony: 'Beta'));
  
  // แสดงรายงาน
  zoo.printReport();
  
  // ให้อาหาร
  zoo.feedAll();
  
  // Polymorphism
  print('\n🎵 เสียงสัตว์:');
  List<ZooAnimal> allAnimals = [
    ...lionEnclosure.animals,
    ...elephantEnclosure.animals,
    ...penguinEnclosure.animals,
  ];
  
  for (ZooAnimal animal in allAnimals) {
    animal.makeSound();
  }
}
```

---

## ขั้นตอนที่ 191-210: สรุปและ Advanced Topics

### Advanced: Covariant

```dart
class AnimalShelter {
  List<Animal> _animals = [];
  
  // covariant อนุญาตให้ subclass ใช้ subtype ของ parameter
  void add(covariant Animal animal) {
    _animals.add(animal);
  }
  
  Animal? findByName(String name) {
    try {
      return _animals.firstWhere((a) => a.name == name);
    } catch (_) {
      return null;
    }
  }
}

class DogShelter extends AnimalShelter {
  // Override ด้วย narrower type
  @override
  void add(Dog dog) {  // Dog แทน Animal (ได้ เพราะ covariant)
    super.add(dog);
    print('เพิ่มหมา: ${dog.name}');
  }
}

class Animal {
  String name;
  Animal(this.name);
}

class Dog extends Animal {
  String breed;
  Dog(String name, this.breed) : super(name);
}

void main() {
  DogShelter shelter = DogShelter();
  shelter.add(Dog('Buddy', 'Labrador'));
  shelter.add(Dog('Rex', 'German Shepherd'));
  
  var found = shelter.findByName('Buddy');
  if (found != null) {
    print('พบ: ${found.name}');
  }
}
```

### สรุป Part 07

```
✅ ขั้นตอนที่ 181: Inheritance พื้นฐาน
✅ ขั้นตอนที่ 182: super keyword
✅ ขั้นตอนที่ 183: Method Overriding
✅ ขั้นตอนที่ 184: Sealed Classes (Dart 3.0+)
✅ ขั้นตอนที่ 185: Generics
✅ ขั้นตอนที่ 186-190: โปรเจกต์ - Zoo Management
✅ ขั้นตอนที่ 191-210: Advanced Topics
```

---

**← [Part 06 - OOP Basics](part-06-oop-basics.md)**

**ต่อไป: [Part 08 - Error Handling →](part-08-error-handling.md)**
