# Part 19: Riverpod State Management
## ขั้นตอนที่ 641-680

---

## 🎯 เป้าหมายของ Part นี้

- Riverpod 2.0 concepts
- Provider types (Provider, StateProvider, FutureProvider, StreamProvider)
- NotifierProvider และ AsyncNotifierProvider
- StateNotifier vs Notifier
- Code Generation กับ @riverpod
- Real-world application

---

## ขั้นตอนที่ 641: Setup Riverpod

```yaml
# pubspec.yaml
dependencies:
  flutter_riverpod: ^2.5.1
  riverpod_annotation: ^2.3.5

dev_dependencies:
  riverpod_generator: ^2.4.0
  build_runner: ^2.4.0
```

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() {
  runApp(
    // ProviderScope ต้องอยู่บน top ของ Widget tree
    const ProviderScope(
      child: MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Riverpod Demo',
      home: const CounterPage(),
    );
  }
}
```

---

## ขั้นตอนที่ 642: Provider types

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// ─── 1. Provider (read-only, computed) ───
final greetingProvider = Provider<String>((ref) {
  return 'สวัสดี Riverpod!';
});

// Provider ที่ depend on provider อื่น
final fullNameProvider = Provider<String>((ref) {
  String greeting = ref.watch(greetingProvider);
  return '$greeting - ยินดีต้อนรับ';
});

// ─── 2. StateProvider (simple mutable state) ───
final counterProvider = StateProvider<int>((ref) => 0);

final themeProvider = StateProvider<ThemeMode>((ref) => ThemeMode.system);

// ─── 3. FutureProvider (async data) ───
final userProvider = FutureProvider.autoDispose<Map<String, dynamic>>((ref) async {
  await Future.delayed(const Duration(seconds: 1));
  return {'id': 1, 'name': 'Alice', 'email': 'alice@example.com'};
});

// ─── 4. StreamProvider (reactive stream) ───
final timerProvider = StreamProvider.autoDispose<int>((ref) {
  return Stream.periodic(const Duration(seconds: 1), (i) => i);
});

// ─── Widget ที่ใช้ Providers ───
class CounterPage extends ConsumerWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // อ่าน state
    int count = ref.watch(counterProvider);
    String greeting = ref.watch(greetingProvider);

    return Scaffold(
      appBar: AppBar(title: Text(greeting)),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('Count: $count', style: const TextStyle(fontSize: 32)),
            const SizedBox(height: 16),

            // FutureProvider
            ref.watch(userProvider).when(
              data: (user) => Text('User: ${user['name']}'),
              loading: () => const CircularProgressIndicator(),
              error: (e, _) => Text('Error: $e'),
            ),

            // StreamProvider
            ref.watch(timerProvider).when(
              data: (t) => Text('Timer: ${t}s'),
              loading: () => const Text('Starting...'),
              error: (_, __) => const SizedBox(),
            ),
          ],
        ),
      ),
      floatingActionButton: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          FloatingActionButton(
            heroTag: 'dec',
            onPressed: () => ref.read(counterProvider.notifier).state--,
            child: const Icon(Icons.remove),
          ),
          const SizedBox(width: 8),
          FloatingActionButton(
            heroTag: 'inc',
            onPressed: () => ref.read(counterProvider.notifier).state++,
            child: const Icon(Icons.add),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 643: NotifierProvider

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

// ─── State class ───
class CartItem {
  final String id;
  final String name;
  final double price;
  final int quantity;

  const CartItem({
    required this.id,
    required this.name,
    required this.price,
    required this.quantity,
  });

  CartItem copyWith({int? quantity}) {
    return CartItem(
      id: id,
      name: name,
      price: price,
      quantity: quantity ?? this.quantity,
    );
  }

  double get subtotal => price * quantity;
}

class CartState {
  final List<CartItem> items;
  final double discountPercent;

  const CartState({this.items = const [], this.discountPercent = 0});

  double get subtotal => items.fold(0, (sum, i) => sum + i.subtotal);
  double get discount => subtotal * discountPercent / 100;
  double get total => subtotal - discount;
  int get itemCount => items.fold(0, (sum, i) => sum + i.quantity);

  CartState copyWith({
    List<CartItem>? items,
    double? discountPercent,
  }) {
    return CartState(
      items: items ?? this.items,
      discountPercent: discountPercent ?? this.discountPercent,
    );
  }
}

// ─── Notifier ───
class CartNotifier extends Notifier<CartState> {
  @override
  CartState build() => const CartState();

  void addItem(CartItem item) {
    List<CartItem> items = [...state.items];
    int idx = items.indexWhere((i) => i.id == item.id);

    if (idx >= 0) {
      items[idx] = items[idx].copyWith(quantity: items[idx].quantity + item.quantity);
    } else {
      items.add(item);
    }

    state = state.copyWith(items: items);
  }

  void removeItem(String id) {
    state = state.copyWith(
      items: state.items.where((i) => i.id != id).toList(),
    );
  }

  void updateQuantity(String id, int quantity) {
    if (quantity <= 0) {
      removeItem(id);
      return;
    }

    state = state.copyWith(
      items: state.items.map((i) => i.id == id ? i.copyWith(quantity: quantity) : i).toList(),
    );
  }

  void applyDiscount(double percent) {
    state = state.copyWith(discountPercent: percent.clamp(0, 100));
  }

  void clear() => state = const CartState();
}

// Provider
final cartProvider = NotifierProvider<CartNotifier, CartState>(CartNotifier.new);

// Derived providers
final cartItemCountProvider = Provider<int>((ref) {
  return ref.watch(cartProvider).itemCount;
});

final cartTotalProvider = Provider<double>((ref) {
  return ref.watch(cartProvider).total;
});
```

---

## ขั้นตอนที่ 644: AsyncNotifierProvider (Async State)

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

class Post {
  final int id;
  final String title;
  final String body;
  bool isFavorite;

  Post({required this.id, required this.title, required this.body, this.isFavorite = false});

  factory Post.fromJson(Map<String, dynamic> json) => Post(
    id: json['id'],
    title: json['title'],
    body: json['body'],
  );
}

// Simulated API
class PostApi {
  Future<List<Post>> fetchPosts() async {
    await Future.delayed(const Duration(seconds: 1));
    return List.generate(
      10,
      (i) => Post(id: i + 1, title: 'Post ${i + 1}', body: 'Content for post ${i + 1}'),
    );
  }

  Future<void> deletePost(int id) async {
    await Future.delayed(const Duration(milliseconds: 500));
  }
}

final postApiProvider = Provider<PostApi>((ref) => PostApi());

// ─── AsyncNotifier ───
class PostsNotifier extends AsyncNotifier<List<Post>> {
  @override
  Future<List<Post>> build() async {
    return ref.read(postApiProvider).fetchPosts();
  }

  Future<void> refresh() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() => ref.read(postApiProvider).fetchPosts());
  }

  Future<void> delete(int postId) async {
    List<Post> current = state.requireValue;

    // Optimistic update
    state = AsyncData(current.where((p) => p.id != postId).toList());

    try {
      await ref.read(postApiProvider).deletePost(postId);
    } catch (e) {
      // Rollback on error
      state = AsyncData(current);
      rethrow;
    }
  }

  void toggleFavorite(int postId) {
    state.whenData((posts) {
      state = AsyncData(
        posts.map((p) {
          return p.id == postId
              ? (Post(id: p.id, title: p.title, body: p.body)..isFavorite = !p.isFavorite)
              : p;
        }).toList(),
      );
    });
  }
}

final postsProvider = AsyncNotifierProvider<PostsNotifier, List<Post>>(PostsNotifier.new);

// ─── Widget ───
class PostsPage extends ConsumerWidget {
  const PostsPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    AsyncValue<List<Post>> postsAsync = ref.watch(postsProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Posts'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: () => ref.read(postsProvider.notifier).refresh(),
          ),
        ],
      ),
      body: postsAsync.when(
        data: (posts) => RefreshIndicator(
          onRefresh: () => ref.read(postsProvider.notifier).refresh(),
          child: ListView.builder(
            itemCount: posts.length,
            itemBuilder: (context, i) {
              Post post = posts[i];
              return ListTile(
                title: Text(post.title),
                subtitle: Text(post.body),
                trailing: Row(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    IconButton(
                      icon: Icon(
                        post.isFavorite ? Icons.favorite : Icons.favorite_border,
                        color: post.isFavorite ? Colors.red : null,
                      ),
                      onPressed: () => ref
                          .read(postsProvider.notifier)
                          .toggleFavorite(post.id),
                    ),
                    IconButton(
                      icon: const Icon(Icons.delete),
                      onPressed: () => ref
                          .read(postsProvider.notifier)
                          .delete(post.id),
                    ),
                  ],
                ),
              );
            },
          ),
        ),
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Text('Error: $e'),
              ElevatedButton(
                onPressed: () => ref.read(postsProvider.notifier).refresh(),
                child: const Text('Retry'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 645: Family Modifier (Parameterized Providers)

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Family: Provider ที่รับ parameter
final userByIdProvider = FutureProvider.family<Map<String, dynamic>, int>((ref, userId) async {
  await Future.delayed(const Duration(milliseconds: 500));
  return {'id': userId, 'name': 'User $userId'};
});

// Family + AutoDispose
final productProvider = FutureProvider.autoDispose.family<Map<String, dynamic>, String>(
  (ref, productId) async {
    await Future.delayed(const Duration(milliseconds: 300));
    return {'id': productId, 'name': 'Product $productId', 'price': 999};
  },
);

// Usage in widget
class UserCard extends ConsumerWidget {
  final int userId;
  const UserCard({super.key, required this.userId});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    AsyncValue<Map<String, dynamic>> user = ref.watch(userByIdProvider(userId));

    return user.when(
      data: (u) => ListTile(
        title: Text(u['name'].toString()),
        subtitle: Text('ID: ${u['id']}'),
      ),
      loading: () => const ListTile(title: LinearProgressIndicator()),
      error: (e, _) => ListTile(title: Text('Error: $e')),
    );
  }
}
```

---

## ขั้นตอนที่ 646: Code Generation กับ @riverpod

```dart
// lib/providers/todo_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'todo_provider.g.dart';

class Todo {
  final String id;
  final String title;
  final bool isDone;

  const Todo({required this.id, required this.title, this.isDone = false});

  Todo copyWith({String? title, bool? isDone}) {
    return Todo(id: id, title: title ?? this.title, isDone: isDone ?? this.isDone);
  }
}

// @riverpod สร้าง todoProvider อัตโนมัติ
@riverpod
class TodoList extends _$TodoList {
  @override
  List<Todo> build() => [];

  void add(String title) {
    state = [
      ...state,
      Todo(id: DateTime.now().millisecondsSinceEpoch.toString(), title: title),
    ];
  }

  void toggle(String id) {
    state = state.map((t) {
      return t.id == id ? t.copyWith(isDone: !t.isDone) : t;
    }).toList();
  }

  void remove(String id) {
    state = state.where((t) => t.id != id).toList();
  }
}

// Simple providers with @riverpod
@riverpod
String greeting(GreetingRef ref) => 'สวัสดี Riverpod!';

@riverpod
Future<List<String>> fetchTags(FetchTagsRef ref) async {
  await Future.delayed(const Duration(seconds: 1));
  return ['Flutter', 'Dart', 'Riverpod'];
}

// Run: dart run build_runner build
// This generates todo_provider.g.dart
```

---

## ขั้นตอนที่ 647: Todo App กับ Riverpod

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

enum TodoFilter { all, active, done }

// Providers
final todoFilterProvider = StateProvider<TodoFilter>((ref) => TodoFilter.all);

final filteredTodosProvider = Provider<List<Todo>>((ref) {
  List<Todo> todos = ref.watch(todoListProvider);
  TodoFilter filter = ref.watch(todoFilterProvider);

  switch (filter) {
    case TodoFilter.active: return todos.where((t) => !t.isDone).toList();
    case TodoFilter.done: return todos.where((t) => t.isDone).toList();
    case TodoFilter.all: return todos;
  }
});

final completedCountProvider = Provider<int>((ref) {
  return ref.watch(todoListProvider).where((t) => t.isDone).length;
});

// ─── Main Screen ───
class TodoScreen extends ConsumerStatefulWidget {
  const TodoScreen({super.key});

  @override
  ConsumerState<TodoScreen> createState() => _TodoScreenState();
}

class _TodoScreenState extends ConsumerState<TodoScreen> {
  final TextEditingController _textController = TextEditingController();

  @override
  void dispose() {
    _textController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    List<Todo> todos = ref.watch(filteredTodosProvider);
    int completedCount = ref.watch(completedCountProvider);
    int totalCount = ref.watch(todoListProvider).length;

    return Scaffold(
      appBar: AppBar(
        title: Text('Todo ($completedCount/$totalCount)'),
        actions: [
          PopupMenuButton<TodoFilter>(
            initialValue: ref.watch(todoFilterProvider),
            onSelected: (f) => ref.read(todoFilterProvider.notifier).state = f,
            itemBuilder: (_) => [
              const PopupMenuItem(value: TodoFilter.all, child: Text('ทั้งหมด')),
              const PopupMenuItem(value: TodoFilter.active, child: Text('ยังไม่เสร็จ')),
              const PopupMenuItem(value: TodoFilter.done, child: Text('เสร็จแล้ว')),
            ],
          ),
        ],
      ),
      body: Column(
        children: [
          // Add todo
          Padding(
            padding: const EdgeInsets.all(16),
            child: Row(
              children: [
                Expanded(
                  child: TextField(
                    controller: _textController,
                    decoration: InputDecoration(
                      hintText: 'เพิ่ม todo...',
                      border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
                    ),
                    onSubmitted: (text) {
                      if (text.isNotEmpty) {
                        ref.read(todoListProvider.notifier).add(text);
                        _textController.clear();
                      }
                    },
                  ),
                ),
                const SizedBox(width: 8),
                ElevatedButton(
                  onPressed: () {
                    if (_textController.text.isNotEmpty) {
                      ref.read(todoListProvider.notifier).add(_textController.text);
                      _textController.clear();
                    }
                  },
                  child: const Text('เพิ่ม'),
                ),
              ],
            ),
          ),
          // Todo list
          Expanded(
            child: todos.isEmpty
                ? const Center(child: Text('ไม่มี Todo'))
                : ListView.builder(
                    itemCount: todos.length,
                    itemBuilder: (_, i) {
                      Todo todo = todos[i];
                      return ListTile(
                        leading: Checkbox(
                          value: todo.isDone,
                          onChanged: (_) => ref.read(todoListProvider.notifier).toggle(todo.id),
                        ),
                        title: Text(
                          todo.title,
                          style: TextStyle(
                            decoration: todo.isDone ? TextDecoration.lineThrough : null,
                            color: todo.isDone ? Colors.grey : null,
                          ),
                        ),
                        trailing: IconButton(
                          icon: const Icon(Icons.delete, color: Colors.red),
                          onPressed: () => ref.read(todoListProvider.notifier).remove(todo.id),
                        ),
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }
}
```

---

**← [Part 18 - Animations](part-18-animations.md)**

**ต่อไป: [Part 20 - BLoC Pattern →](part-20-bloc.md)**
