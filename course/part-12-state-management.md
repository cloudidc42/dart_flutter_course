# Part 12: State Management
## ขั้นตอนที่ 361-400

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ State Management คืออะไร
- ใช้ setState สำหรับ local state
- ใช้ InheritedWidget และ InheritedNotifier
- ใช้ Provider package
- ใช้ ChangeNotifier
- Scoped vs Global state

---

## ขั้นตอนที่ 361: State Management คืออะไร

```
State = ข้อมูลที่ UI ต้องการเพื่อ render

ประเภท State:
1. Ephemeral/Local State  → เฉพาะ widget นั้น (เช่น tab ที่กำลังเปิด)
   → ใช้ setState()

2. App/Global State      → ใช้ทั่วทั้งแอป (เช่น user login, cart)
   → ใช้ Provider, Riverpod, BLoC ฯลฯ

State Management Options:
├── Built-in
│   ├── setState (local)
│   ├── InheritedWidget (manual)
│   └── ValueNotifier/ChangeNotifier
├── Community Packages
│   ├── Provider (พื้นฐาน, ง่าย)
│   ├── Riverpod (ทันสมัย, type-safe)
│   ├── BLoC (enterprise, predictable)
│   └── GetX (all-in-one)
```

---

## ขั้นตอนที่ 362: setState - Local State

```dart
import 'package:flutter/material.dart';

// setState เหมาะกับ:
// - ข้อมูลที่ใช้เฉพาะใน widget เดียว
// - Simple UI interactions
// - Toggle, Counter, Form input

class ShoppingCartItem {
  final String name;
  final double price;
  int quantity;
  
  ShoppingCartItem({required this.name, required this.price, this.quantity = 1});
  
  double get total => price * quantity;
}

class CartScreen extends StatefulWidget {
  const CartScreen({super.key});
  
  @override
  State<CartScreen> createState() => _CartScreenState();
}

class _CartScreenState extends State<CartScreen> {
  final List<ShoppingCartItem> _items = [
    ShoppingCartItem(name: 'Flutter Book', price: 599, quantity: 1),
    ShoppingCartItem(name: 'Dart Course', price: 1299, quantity: 1),
    ShoppingCartItem(name: 'VS Code Pro', price: 299, quantity: 2),
  ];
  
  double get _totalPrice => _items.fold(0, (sum, item) => sum + item.total);
  
  void _incrementQuantity(int index) {
    setState(() {
      _items[index].quantity++;
    });
  }
  
  void _decrementQuantity(int index) {
    setState(() {
      if (_items[index].quantity > 1) {
        _items[index].quantity--;
      }
    });
  }
  
  void _removeItem(int index) {
    setState(() {
      _items.removeAt(index);
    });
  }
  
  void _checkout() {
    showDialog(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('ยืนยันการสั่งซื้อ'),
        content: Text('ยอดรวม: ฿${_totalPrice.toStringAsFixed(2)}'),
        actions: [
          TextButton(onPressed: () => Navigator.pop(ctx), child: const Text('ยกเลิก')),
          ElevatedButton(
            onPressed: () {
              Navigator.pop(ctx);
              setState(() => _items.clear());
              ScaffoldMessenger.of(context).showSnackBar(
                const SnackBar(content: Text('สั่งซื้อสำเร็จ!')),
              );
            },
            child: const Text('ยืนยัน'),
          ),
        ],
      ),
    );
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('ตะกร้าสินค้า'),
        actions: [
          if (_items.isNotEmpty)
            Badge(
              label: Text('${_items.length}'),
              child: const IconButton(icon: Icon(Icons.shopping_cart), onPressed: null),
            ),
        ],
      ),
      body: _items.isEmpty
          ? const Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Icon(Icons.shopping_cart_outlined, size: 64, color: Colors.grey),
                  SizedBox(height: 16),
                  Text('ตะกร้าว่าง', style: TextStyle(color: Colors.grey, fontSize: 18)),
                ],
              ),
            )
          : Column(
              children: [
                Expanded(
                  child: ListView.builder(
                    padding: const EdgeInsets.all(16),
                    itemCount: _items.length,
                    itemBuilder: (context, index) {
                      ShoppingCartItem item = _items[index];
                      return Card(
                        margin: const EdgeInsets.only(bottom: 8),
                        child: Padding(
                          padding: const EdgeInsets.all(12),
                          child: Row(
                            children: [
                              Expanded(
                                child: Column(
                                  crossAxisAlignment: CrossAxisAlignment.start,
                                  children: [
                                    Text(item.name, style: const TextStyle(fontWeight: FontWeight.bold)),
                                    Text('฿${item.price.toStringAsFixed(0)} / ชิ้น',
                                        style: TextStyle(color: Colors.grey[600])),
                                  ],
                                ),
                              ),
                              Row(
                                children: [
                                  IconButton(
                                    onPressed: () => _decrementQuantity(index),
                                    icon: const Icon(Icons.remove_circle_outline),
                                    iconSize: 20,
                                  ),
                                  SizedBox(
                                    width: 32,
                                    child: Text(
                                      '${item.quantity}',
                                      textAlign: TextAlign.center,
                                      style: const TextStyle(fontWeight: FontWeight.bold),
                                    ),
                                  ),
                                  IconButton(
                                    onPressed: () => _incrementQuantity(index),
                                    icon: const Icon(Icons.add_circle_outline),
                                    iconSize: 20,
                                  ),
                                ],
                              ),
                              Column(
                                crossAxisAlignment: CrossAxisAlignment.end,
                                children: [
                                  Text('฿${item.total.toStringAsFixed(0)}',
                                      style: const TextStyle(fontWeight: FontWeight.bold, color: Colors.deepPurple)),
                                  IconButton(
                                    onPressed: () => _removeItem(index),
                                    icon: const Icon(Icons.delete_outline, color: Colors.red, size: 20),
                                    constraints: const BoxConstraints(),
                                    padding: EdgeInsets.zero,
                                  ),
                                ],
                              ),
                            ],
                          ),
                        ),
                      );
                    },
                  ),
                ),
                Container(
                  padding: const EdgeInsets.all(16),
                  decoration: BoxDecoration(
                    color: Colors.white,
                    boxShadow: [BoxShadow(color: Colors.black12, blurRadius: 8, offset: const Offset(0, -2))],
                  ),
                  child: Column(
                    children: [
                      Row(
                        mainAxisAlignment: MainAxisAlignment.spaceBetween,
                        children: [
                          const Text('ยอดรวม:', style: TextStyle(fontSize: 16)),
                          Text('฿${_totalPrice.toStringAsFixed(2)}',
                              style: const TextStyle(fontSize: 20, fontWeight: FontWeight.bold, color: Colors.deepPurple)),
                        ],
                      ),
                      const SizedBox(height: 12),
                      SizedBox(
                        width: double.infinity,
                        child: ElevatedButton(
                          onPressed: _checkout,
                          style: ElevatedButton.styleFrom(padding: const EdgeInsets.all(16)),
                          child: const Text('สั่งซื้อเลย', style: TextStyle(fontSize: 16)),
                        ),
                      ),
                    ],
                  ),
                ),
              ],
            ),
    );
  }
}
```

---

## ขั้นตอนที่ 363: ChangeNotifier

```dart
import 'package:flutter/material.dart';

// ChangeNotifier = Observable object ที่ notify listeners เมื่อ state เปลี่ยน
class CounterModel extends ChangeNotifier {
  int _count = 0;
  int get count => _count;
  
  void increment() {
    _count++;
    notifyListeners();  // แจ้ง listeners ว่ามีการเปลี่ยนแปลง
  }
  
  void decrement() {
    if (_count > 0) {
      _count--;
      notifyListeners();
    }
  }
  
  void reset() {
    _count = 0;
    notifyListeners();
  }
}

// ใช้ ValueNotifier สำหรับ single value
class ThemeModel extends ValueNotifier<ThemeMode> {
  ThemeModel() : super(ThemeMode.light);
  
  void toggle() {
    value = value == ThemeMode.light ? ThemeMode.dark : ThemeMode.light;
  }
}

// AnimatedBuilder ใช้ rebuild เมื่อ notifier เปลี่ยน
class CounterWithNotifier extends StatefulWidget {
  const CounterWithNotifier({super.key});
  
  @override
  State<CounterWithNotifier> createState() => _CounterWithNotifierState();
}

class _CounterWithNotifierState extends State<CounterWithNotifier> {
  final CounterModel _counter = CounterModel();
  
  @override
  void dispose() {
    _counter.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('ChangeNotifier')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            AnimatedBuilder(
              animation: _counter,
              builder: (context, child) {
                return Text(
                  '${_counter.count}',
                  style: const TextStyle(fontSize: 72, fontWeight: FontWeight.bold),
                );
              },
            ),
            const SizedBox(height: 24),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                ElevatedButton(onPressed: _counter.decrement, child: const Text('-')),
                const SizedBox(width: 16),
                ElevatedButton(onPressed: _counter.reset, child: const Text('Reset')),
                const SizedBox(width: 16),
                ElevatedButton(onPressed: _counter.increment, child: const Text('+')),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 364: Provider Package

```dart
// pubspec.yaml:
// dependencies:
//   provider: ^6.1.0

import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

// ─── Models ───
class Product {
  final String id;
  final String name;
  final double price;
  final String imageUrl;
  
  const Product({required this.id, required this.name, required this.price, required this.imageUrl});
}

class CartItem {
  final Product product;
  int quantity;
  
  CartItem({required this.product, this.quantity = 1});
  
  double get subtotal => product.price * quantity;
}

// ─── Cart Provider ───
class CartProvider extends ChangeNotifier {
  final Map<String, CartItem> _items = {};
  
  Map<String, CartItem> get items => Map.unmodifiable(_items);
  
  int get itemCount => _items.values.fold(0, (sum, item) => sum + item.quantity);
  
  double get totalPrice => _items.values.fold(0.0, (sum, item) => sum + item.subtotal);
  
  bool isInCart(String productId) => _items.containsKey(productId);
  
  void addToCart(Product product) {
    if (_items.containsKey(product.id)) {
      _items[product.id]!.quantity++;
    } else {
      _items[product.id] = CartItem(product: product);
    }
    notifyListeners();
  }
  
  void removeFromCart(String productId) {
    _items.remove(productId);
    notifyListeners();
  }
  
  void updateQuantity(String productId, int quantity) {
    if (quantity <= 0) {
      removeFromCart(productId);
    } else if (_items.containsKey(productId)) {
      _items[productId]!.quantity = quantity;
      notifyListeners();
    }
  }
  
  void clearCart() {
    _items.clear();
    notifyListeners();
  }
}

// ─── Auth Provider ───
class AuthProvider extends ChangeNotifier {
  String? _userId;
  String? _userName;
  bool _isLoading = false;
  
  bool get isAuthenticated => _userId != null;
  String? get userId => _userId;
  String? get userName => _userName;
  bool get isLoading => _isLoading;
  
  Future<bool> login(String email, String password) async {
    _isLoading = true;
    notifyListeners();
    
    await Future.delayed(const Duration(seconds: 1));
    
    if (email == 'user@example.com' && password == 'password') {
      _userId = 'user_001';
      _userName = 'Alice';
      _isLoading = false;
      notifyListeners();
      return true;
    }
    
    _isLoading = false;
    notifyListeners();
    return false;
  }
  
  void logout() {
    _userId = null;
    _userName = null;
    notifyListeners();
  }
}

// ─── App Setup ───
class EcommerceApp extends StatelessWidget {
  const EcommerceApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => CartProvider()),
        ChangeNotifierProvider(create: (_) => AuthProvider()),
      ],
      child: MaterialApp(
        title: 'E-Commerce',
        theme: ThemeData(
          colorScheme: ColorScheme.fromSeed(seedColor: Colors.orange),
          useMaterial3: true,
        ),
        home: const MainShopScreen(),
      ),
    );
  }
}

// ─── Shop Screen ───
final List<Product> products = [
  const Product(id: '1', name: 'Laptop Pro', price: 35000, imageUrl: 'https://via.placeholder.com/200'),
  const Product(id: '2', name: 'Wireless Mouse', price: 890, imageUrl: 'https://via.placeholder.com/200'),
  const Product(id: '3', name: 'Mechanical Keyboard', price: 3500, imageUrl: 'https://via.placeholder.com/200'),
  const Product(id: '4', name: 'USB Hub', price: 650, imageUrl: 'https://via.placeholder.com/200'),
];

class MainShopScreen extends StatelessWidget {
  const MainShopScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    // consumer ดูค่าจาก provider และ rebuild เมื่อเปลี่ยน
    return Scaffold(
      appBar: AppBar(
        title: const Text('ร้านค้า'),
        actions: [
          Consumer<CartProvider>(
            builder: (context, cart, child) {
              return Badge(
                label: Text('${cart.itemCount}'),
                isLabelVisible: cart.itemCount > 0,
                child: IconButton(
                  icon: const Icon(Icons.shopping_cart),
                  onPressed: () {
                    Navigator.push(context, MaterialPageRoute(builder: (_) => const CartScreen()));
                  },
                ),
              );
            },
          ),
        ],
      ),
      body: GridView.builder(
        padding: const EdgeInsets.all(16),
        gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
          crossAxisCount: 2,
          mainAxisSpacing: 12,
          crossAxisSpacing: 12,
          childAspectRatio: 0.75,
        ),
        itemCount: products.length,
        itemBuilder: (context, index) => ProductCard(product: products[index]),
      ),
    );
  }
}

class ProductCard extends StatelessWidget {
  final Product product;
  
  const ProductCard({super.key, required this.product});
  
  @override
  Widget build(BuildContext context) {
    // context.watch<T>() = rebuild เมื่อ T เปลี่ยน
    // context.read<T>() = ไม่ rebuild (ใช้สำหรับ call methods)
    // context.select<T, R>() = rebuild เมื่อ field R เปลี่ยนเท่านั้น
    
    bool inCart = context.select<CartProvider, bool>(
      (cart) => cart.isInCart(product.id),
    );
    
    return Card(
      clipBehavior: Clip.antiAlias,
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Expanded(
            child: Image.network(
              product.imageUrl,
              width: double.infinity,
              fit: BoxFit.cover,
              errorBuilder: (c, e, s) => Container(
                color: Colors.grey[200],
                child: const Icon(Icons.image, size: 48),
              ),
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(8),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(product.name,
                    style: const TextStyle(fontWeight: FontWeight.bold),
                    maxLines: 1, overflow: TextOverflow.ellipsis),
                Text('฿${product.price.toStringAsFixed(0)}',
                    style: const TextStyle(color: Colors.orange, fontWeight: FontWeight.bold)),
                const SizedBox(height: 8),
                SizedBox(
                  width: double.infinity,
                  child: ElevatedButton(
                    onPressed: () {
                      if (inCart) {
                        context.read<CartProvider>().removeFromCart(product.id);
                      } else {
                        context.read<CartProvider>().addToCart(product);
                      }
                    },
                    style: ElevatedButton.styleFrom(
                      backgroundColor: inCart ? Colors.red : null,
                      foregroundColor: inCart ? Colors.white : null,
                    ),
                    child: Text(inCart ? 'ลบออก' : 'เพิ่มในตะกร้า', style: const TextStyle(fontSize: 12)),
                  ),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

class CartScreen extends StatelessWidget {
  const CartScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('ตะกร้าสินค้า')),
      body: Consumer<CartProvider>(
        builder: (context, cart, child) {
          if (cart.items.isEmpty) {
            return const Center(child: Text('ตะกร้าว่าง'));
          }
          
          return Column(
            children: [
              Expanded(
                child: ListView.builder(
                  itemCount: cart.items.length,
                  itemBuilder: (context, index) {
                    CartItem item = cart.items.values.elementAt(index);
                    return ListTile(
                      leading: Text(item.product.name[0], style: const TextStyle(fontSize: 24)),
                      title: Text(item.product.name),
                      subtitle: Text('฿${item.product.price} x ${item.quantity}'),
                      trailing: Row(
                        mainAxisSize: MainAxisSize.min,
                        children: [
                          Text('฿${item.subtotal.toStringAsFixed(0)}',
                              style: const TextStyle(fontWeight: FontWeight.bold)),
                          const SizedBox(width: 8),
                          IconButton(
                            icon: const Icon(Icons.delete_outline, color: Colors.red),
                            onPressed: () => cart.removeFromCart(item.product.id),
                          ),
                        ],
                      ),
                    );
                  },
                ),
              ),
              Container(
                padding: const EdgeInsets.all(16),
                child: Column(
                  children: [
                    Row(
                      mainAxisAlignment: MainAxisAlignment.spaceBetween,
                      children: [
                        const Text('ยอดรวม', style: TextStyle(fontSize: 18)),
                        Text('฿${cart.totalPrice.toStringAsFixed(2)}',
                            style: const TextStyle(fontSize: 22, fontWeight: FontWeight.bold, color: Colors.orange)),
                      ],
                    ),
                    const SizedBox(height: 12),
                    Row(
                      children: [
                        Expanded(
                          child: OutlinedButton(
                            onPressed: () => cart.clearCart(),
                            child: const Text('ล้างตะกร้า'),
                          ),
                        ),
                        const SizedBox(width: 12),
                        Expanded(
                          child: ElevatedButton(
                            onPressed: () {},
                            child: const Text('สั่งซื้อ'),
                          ),
                        ),
                      ],
                    ),
                  ],
                ),
              ),
            ],
          );
        },
      ),
    );
  }
}

void main() {
  runApp(const EcommerceApp());
}
```

---

## ขั้นตอนที่ 365-380: ProxyProvider และ Consumer patterns

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

// ─── ProxyProvider: provider ที่ depend on provider อื่น ───
class ApiService {
  final String baseUrl;
  ApiService({required this.baseUrl});
  
  Future<List<String>> fetchItems() async {
    await Future.delayed(const Duration(milliseconds: 500));
    return ['Item A', 'Item B', 'Item C'];
  }
}

class ItemRepository {
  final ApiService _api;
  ItemRepository(this._api);
  
  Future<List<String>> getItems() => _api.fetchItems();
}

// ─── Selector: เลือก rebuild เฉพาะส่วนที่สนใจ ───
class UserState extends ChangeNotifier {
  String _name = 'Guest';
  String _email = '';
  int _points = 0;
  
  String get name => _name;
  String get email => _email;
  int get points => _points;
  
  void setName(String name) {
    _name = name;
    notifyListeners();
  }
  
  void addPoints(int amount) {
    _points += amount;
    notifyListeners();
  }
}

class SelectorExample extends StatelessWidget {
  const SelectorExample({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Selector')),
      body: Column(
        children: [
          // Selector: rebuild เฉพาะเมื่อ name เปลี่ยน (ไม่ rebuild เมื่อ points เปลี่ยน)
          Selector<UserState, String>(
            selector: (_, user) => user.name,
            builder: (_, name, __) {
              print('NameWidget rebuilt');  // rebuild น้อยลง
              return Text('Name: $name', style: const TextStyle(fontSize: 18));
            },
          ),
          
          // Selector: rebuild เฉพาะเมื่อ points เปลี่ยน
          Selector<UserState, int>(
            selector: (_, user) => user.points,
            builder: (_, points, __) {
              print('PointsWidget rebuilt');
              return Text('Points: $points', style: const TextStyle(fontSize: 18));
            },
          ),
          
          ElevatedButton(
            onPressed: () => context.read<UserState>().addPoints(10),
            child: const Text('+10 Points'),
          ),
          ElevatedButton(
            onPressed: () => context.read<UserState>().setName('Alice'),
            child: const Text('Set Name'),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 381-400: โปรเจกต์ - Todo App

```dart
// todo_app.dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

void main() {
  runApp(
    ChangeNotifierProvider(
      create: (_) => TodoProvider(),
      child: const TodoApp(),
    ),
  );
}

// ─── Model ───
class Todo {
  final String id;
  String title;
  String? description;
  bool isDone;
  final DateTime createdAt;
  String category;
  
  Todo({
    required this.id,
    required this.title,
    this.description,
    this.isDone = false,
    String? category,
  })  : createdAt = DateTime.now(),
        category = category ?? 'General';
  
  Todo copyWith({String? title, String? description, bool? isDone, String? category}) {
    return Todo(
      id: id,
      title: title ?? this.title,
      description: description ?? this.description,
      isDone: isDone ?? this.isDone,
      category: category ?? this.category,
    );
  }
}

enum TodoFilter { all, active, completed }

// ─── Provider ───
class TodoProvider extends ChangeNotifier {
  final List<Todo> _todos = [
    Todo(id: '1', title: 'เรียน Flutter', description: 'ศึกษา State Management', category: 'Learning'),
    Todo(id: '2', title: 'สร้าง Portfolio App', category: 'Project'),
    Todo(id: '3', title: 'อ่านหนังสือ Clean Architecture', category: 'Learning', isDone: true),
  ];
  
  TodoFilter _filter = TodoFilter.all;
  String _searchQuery = '';
  String _sortBy = 'created';
  
  List<Todo> get todos {
    List<Todo> result = List.from(_todos);
    
    // Filter
    result = switch (_filter) {
      TodoFilter.active => result.where((t) => !t.isDone).toList(),
      TodoFilter.completed => result.where((t) => t.isDone).toList(),
      TodoFilter.all => result,
    };
    
    // Search
    if (_searchQuery.isNotEmpty) {
      result = result.where((t) =>
          t.title.toLowerCase().contains(_searchQuery.toLowerCase()) ||
          (t.description?.toLowerCase().contains(_searchQuery.toLowerCase()) ?? false)
      ).toList();
    }
    
    // Sort
    result.sort((a, b) => switch (_sortBy) {
      'title' => a.title.compareTo(b.title),
      'category' => a.category.compareTo(b.category),
      _ => b.createdAt.compareTo(a.createdAt),  // newest first
    });
    
    return result;
  }
  
  int get totalCount => _todos.length;
  int get completedCount => _todos.where((t) => t.isDone).length;
  int get activeCount => _todos.where((t) => !t.isDone).length;
  TodoFilter get filter => _filter;
  String get searchQuery => _searchQuery;
  
  List<String> get categories {
    return _todos.map((t) => t.category).toSet().toList()..sort();
  }
  
  void addTodo(String title, {String? description, String? category}) {
    _todos.add(Todo(
      id: DateTime.now().millisecondsSinceEpoch.toString(),
      title: title,
      description: description,
      category: category,
    ));
    notifyListeners();
  }
  
  void toggleTodo(String id) {
    int index = _todos.indexWhere((t) => t.id == id);
    if (index != -1) {
      _todos[index] = _todos[index].copyWith(isDone: !_todos[index].isDone);
      notifyListeners();
    }
  }
  
  void deleteTodo(String id) {
    _todos.removeWhere((t) => t.id == id);
    notifyListeners();
  }
  
  void updateTodo(String id, {String? title, String? description, String? category}) {
    int index = _todos.indexWhere((t) => t.id == id);
    if (index != -1) {
      _todos[index] = _todos[index].copyWith(
        title: title,
        description: description,
        category: category,
      );
      notifyListeners();
    }
  }
  
  void setFilter(TodoFilter filter) {
    _filter = filter;
    notifyListeners();
  }
  
  void setSearch(String query) {
    _searchQuery = query;
    notifyListeners();
  }
  
  void setSort(String sortBy) {
    _sortBy = sortBy;
    notifyListeners();
  }
  
  void clearCompleted() {
    _todos.removeWhere((t) => t.isDone);
    notifyListeners();
  }
}

// ─── App ───
class TodoApp extends StatelessWidget {
  const TodoApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Todo App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const TodoScreen(),
    );
  }
}

// ─── Main Screen ───
class TodoScreen extends StatelessWidget {
  const TodoScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Todo List'),
        actions: [
          Consumer<TodoProvider>(
            builder: (ctx, provider, _) => PopupMenuButton<String>(
              onSelected: provider.setSort,
              itemBuilder: (_) => const [
                PopupMenuItem(value: 'created', child: Text('เรียงตามเวลา')),
                PopupMenuItem(value: 'title', child: Text('เรียงตามชื่อ')),
                PopupMenuItem(value: 'category', child: Text('เรียงตาม Category')),
              ],
              icon: const Icon(Icons.sort),
            ),
          ),
          Consumer<TodoProvider>(
            builder: (ctx, provider, _) => provider.completedCount > 0
                ? IconButton(
                    icon: const Icon(Icons.cleaning_services),
                    tooltip: 'ลบที่เสร็จแล้ว',
                    onPressed: provider.clearCompleted,
                  )
                : const SizedBox.shrink(),
          ),
        ],
      ),
      body: Column(
        children: [
          // Stats
          Consumer<TodoProvider>(
            builder: (ctx, provider, _) => Padding(
              padding: const EdgeInsets.all(16),
              child: Row(
                children: [
                  _StatChip(label: 'ทั้งหมด', count: provider.totalCount, color: Colors.teal),
                  const SizedBox(width: 8),
                  _StatChip(label: 'กำลังทำ', count: provider.activeCount, color: Colors.orange),
                  const SizedBox(width: 8),
                  _StatChip(label: 'เสร็จแล้ว', count: provider.completedCount, color: Colors.green),
                ],
              ),
            ),
          ),
          
          // Search
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            child: Consumer<TodoProvider>(
              builder: (ctx, provider, _) => SearchBar(
                hintText: 'ค้นหา...',
                leading: const Icon(Icons.search),
                onChanged: provider.setSearch,
              ),
            ),
          ),
          const SizedBox(height: 8),
          
          // Filter tabs
          Consumer<TodoProvider>(
            builder: (ctx, provider, _) => Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16),
              child: SegmentedButton<TodoFilter>(
                segments: const [
                  ButtonSegment(value: TodoFilter.all, label: Text('ทั้งหมด')),
                  ButtonSegment(value: TodoFilter.active, label: Text('กำลังทำ')),
                  ButtonSegment(value: TodoFilter.completed, label: Text('เสร็จ')),
                ],
                selected: {provider.filter},
                onSelectionChanged: (selected) => provider.setFilter(selected.first),
              ),
            ),
          ),
          const SizedBox(height: 8),
          
          // Todo List
          Expanded(
            child: Consumer<TodoProvider>(
              builder: (ctx, provider, _) {
                List<Todo> todos = provider.todos;
                
                if (todos.isEmpty) {
                  return Center(
                    child: Column(
                      mainAxisAlignment: MainAxisAlignment.center,
                      children: [
                        const Icon(Icons.check_circle_outline, size: 64, color: Colors.grey),
                        const SizedBox(height: 16),
                        Text(
                          provider.searchQuery.isNotEmpty ? 'ไม่พบรายการ' : 'ยังไม่มีรายการ',
                          style: TextStyle(color: Colors.grey[600], fontSize: 16),
                        ),
                      ],
                    ),
                  );
                }
                
                return ListView.builder(
                  padding: const EdgeInsets.symmetric(horizontal: 16),
                  itemCount: todos.length,
                  itemBuilder: (context, index) => TodoItem(todo: todos[index]),
                );
              },
            ),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showAddDialog(context),
        child: const Icon(Icons.add),
      ),
    );
  }
  
  void _showAddDialog(BuildContext context) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      builder: (ctx) => const AddTodoSheet(),
    );
  }
}

// ─── Todo Item ───
class TodoItem extends StatelessWidget {
  final Todo todo;
  
  const TodoItem({super.key, required this.todo});
  
  @override
  Widget build(BuildContext context) {
    return Dismissible(
      key: Key(todo.id),
      direction: DismissDirection.endToStart,
      background: Container(
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 16),
        color: Colors.red,
        child: const Icon(Icons.delete, color: Colors.white),
      ),
      onDismissed: (_) => context.read<TodoProvider>().deleteTodo(todo.id),
      child: Card(
        margin: const EdgeInsets.only(bottom: 8),
        child: ListTile(
          leading: Checkbox(
            value: todo.isDone,
            onChanged: (_) => context.read<TodoProvider>().toggleTodo(todo.id),
          ),
          title: Text(
            todo.title,
            style: TextStyle(
              decoration: todo.isDone ? TextDecoration.lineThrough : null,
              color: todo.isDone ? Colors.grey : null,
            ),
          ),
          subtitle: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              if (todo.description != null) Text(todo.description!, style: const TextStyle(fontSize: 12)),
              Chip(
                label: Text(todo.category, style: const TextStyle(fontSize: 10)),
                padding: EdgeInsets.zero,
                materialTapTargetSize: MaterialTapTargetSize.shrinkWrap,
                visualDensity: VisualDensity.compact,
              ),
            ],
          ),
          isThreeLine: todo.description != null,
        ),
      ),
    );
  }
}

// ─── Add Todo Sheet ───
class AddTodoSheet extends StatefulWidget {
  const AddTodoSheet({super.key});
  
  @override
  State<AddTodoSheet> createState() => _AddTodoSheetState();
}

class _AddTodoSheetState extends State<AddTodoSheet> {
  final _titleController = TextEditingController();
  final _descController = TextEditingController();
  String _category = 'General';
  
  @override
  void dispose() {
    _titleController.dispose();
    _descController.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.only(
        left: 16,
        right: 16,
        top: 16,
        bottom: MediaQuery.of(context).viewInsets.bottom + 16,
      ),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const Text('เพิ่ม Todo', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          const SizedBox(height: 16),
          TextField(
            controller: _titleController,
            decoration: const InputDecoration(labelText: 'ชื่อ Todo', border: OutlineInputBorder()),
            autofocus: true,
          ),
          const SizedBox(height: 8),
          TextField(
            controller: _descController,
            decoration: const InputDecoration(labelText: 'รายละเอียด (ถ้ามี)', border: OutlineInputBorder()),
            maxLines: 2,
          ),
          const SizedBox(height: 8),
          DropdownButtonFormField<String>(
            value: _category,
            decoration: const InputDecoration(labelText: 'Category', border: OutlineInputBorder()),
            items: ['General', 'Work', 'Learning', 'Project', 'Personal']
                .map((c) => DropdownMenuItem(value: c, child: Text(c)))
                .toList(),
            onChanged: (v) => setState(() => _category = v!),
          ),
          const SizedBox(height: 16),
          SizedBox(
            width: double.infinity,
            child: ElevatedButton(
              onPressed: () {
                if (_titleController.text.isNotEmpty) {
                  context.read<TodoProvider>().addTodo(
                    _titleController.text,
                    description: _descController.text.isEmpty ? null : _descController.text,
                    category: _category,
                  );
                  Navigator.pop(context);
                }
              },
              child: const Text('เพิ่ม'),
            ),
          ),
        ],
      ),
    );
  }
}

class _StatChip extends StatelessWidget {
  final String label;
  final int count;
  final Color color;
  
  const _StatChip({required this.label, required this.count, required this.color});
  
  @override
  Widget build(BuildContext context) {
    return Expanded(
      child: Container(
        padding: const EdgeInsets.all(8),
        decoration: BoxDecoration(
          color: color.withOpacity(0.1),
          borderRadius: BorderRadius.circular(8),
          border: Border.all(color: color.withOpacity(0.3)),
        ),
        child: Column(
          children: [
            Text('$count', style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold, color: color)),
            Text(label, style: TextStyle(fontSize: 11, color: color)),
          ],
        ),
      ),
    );
  }
}
```

---

**← [Part 11 - Async/Await](part-11-async-await-futures.md)**

**ต่อไป: [Part 13 - Navigation และ Routing →](part-13-navigation-routing.md)**
