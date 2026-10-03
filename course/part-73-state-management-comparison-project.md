# Part 73: State Management Comparison Project
## ขั้นตอนที่ 2801-2840

## 🎯 เป้าหมายของ Part นี้
- สร้าง TODO App เดียวกันด้วย 4 วิธีต่างกัน
- เปรียบเทียบ setState vs Provider vs Riverpod vs BLoC
- เข้าใจข้อดีข้อเสียของแต่ละวิธี
- รู้ว่าควรใช้วิธีไหนในสถานการณ์ใด
- Code ที่ runnable ครบทั้ง 4 versions

---

## ขั้นตอนที่ 2801: Todo Model ที่ใช้ร่วมกัน

```dart
// lib/models/todo.dart
import 'package:flutter/foundation.dart';

@immutable
class Todo {
  final String id;
  final String title;
  final bool isCompleted;
  final DateTime createdAt;
  final String? category;

  const Todo({
    required this.id,
    required this.title,
    this.isCompleted = false,
    required this.createdAt,
    this.category,
  });

  Todo copyWith({
    String? id,
    String? title,
    bool? isCompleted,
    DateTime? createdAt,
    String? category,
  }) {
    return Todo(
      id: id ?? this.id,
      title: title ?? this.title,
      isCompleted: isCompleted ?? this.isCompleted,
      createdAt: createdAt ?? this.createdAt,
      category: category ?? this.category,
    );
  }

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is Todo && runtimeType == other.runtimeType && id == other.id;

  @override
  int get hashCode => id.hashCode;

  @override
  String toString() => 'Todo(id: $id, title: $title, done: $isCompleted)';

  Map<String, dynamic> toJson() => {
    'id': id,
    'title': title,
    'isCompleted': isCompleted,
    'createdAt': createdAt.toIso8601String(),
    'category': category,
  };

  factory Todo.fromJson(Map<String, dynamic> json) => Todo(
    id: json['id'] as String,
    title: json['title'] as String,
    isCompleted: json['isCompleted'] as bool,
    createdAt: DateTime.parse(json['createdAt'] as String),
    category: json['category'] as String?,
  );
}

// Filter options
enum TodoFilter { all, active, completed }

extension TodoFilterLabel on TodoFilter {
  String get label => switch (this) {
    TodoFilter.all => 'All',
    TodoFilter.active => 'Active',
    TodoFilter.completed => 'Completed',
  };
}
```

---

## ขั้นตอนที่ 2802: Version 1 - setState (Simplest Approach)

```dart
// lib/version1_setstate/todo_app_setstate.dart
import 'package:flutter/material.dart';
import '../models/todo.dart';

/// VERSION 1: setState
/// ✅ ข้อดี: ง่าย, ไม่ต้องติดตั้ง package เพิ่ม, เหมาะสำหรับ app เล็ก
/// ❌ ข้อเสีย: state กระจัดกระจาย, hard to share across widgets, 
///            rebuild ทั้งหมดเมื่อ state เปลี่ยน

void main() => runApp(const SetStateTodoApp());

class SetStateTodoApp extends StatelessWidget {
  const SetStateTodoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Todo App (setState)',
      theme: _buildTheme(),
      home: const SetStateTodoScreen(),
    );
  }
}

class SetStateTodoScreen extends StatefulWidget {
  const SetStateTodoScreen({super.key});

  @override
  State<SetStateTodoScreen> createState() => _SetStateTodoScreenState();
}

class _SetStateTodoScreenState extends State<SetStateTodoScreen> {
  // All state lives here
  final List<Todo> _todos = [];
  TodoFilter _filter = TodoFilter.all;
  final TextEditingController _textController = TextEditingController();
  bool _isEditing = false;
  String? _editingId;

  @override
  void dispose() {
    _textController.dispose();
    super.dispose();
  }

  // State mutation methods
  void _addTodo(String title) {
    if (title.trim().isEmpty) return;
    setState(() {
      _todos.add(Todo(
        id: DateTime.now().millisecondsSinceEpoch.toString(),
        title: title.trim(),
        createdAt: DateTime.now(),
      ));
    });
    _textController.clear();
  }

  void _toggleTodo(String id) {
    setState(() {
      final index = _todos.indexWhere((t) => t.id == id);
      if (index != -1) {
        _todos[index] = _todos[index].copyWith(
          isCompleted: !_todos[index].isCompleted,
        );
      }
    });
  }

  void _deleteTodo(String id) {
    setState(() {
      _todos.removeWhere((t) => t.id == id);
    });
  }

  void _updateTodo(String id, String newTitle) {
    if (newTitle.trim().isEmpty) return;
    setState(() {
      final index = _todos.indexWhere((t) => t.id == id);
      if (index != -1) {
        _todos[index] = _todos[index].copyWith(title: newTitle.trim());
      }
      _isEditing = false;
      _editingId = null;
    });
    _textController.clear();
  }

  void _clearCompleted() {
    setState(() {
      _todos.removeWhere((t) => t.isCompleted);
    });
  }

  List<Todo> get _filteredTodos => switch (_filter) {
    TodoFilter.all => List.from(_todos),
    TodoFilter.active => _todos.where((t) => !t.isCompleted).toList(),
    TodoFilter.completed => _todos.where((t) => t.isCompleted).toList(),
  };

  int get _activeCount => _todos.where((t) => !t.isCompleted).length;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Todo (setState)'),
        backgroundColor: Colors.indigo,
        foregroundColor: Colors.white,
        actions: [
          if (_todos.any((t) => t.isCompleted))
            TextButton(
              onPressed: _clearCompleted,
              child: const Text(
                'Clear Done',
                style: TextStyle(color: Colors.white70),
              ),
            ),
        ],
      ),
      body: Column(
        children: [
          _buildInputField(),
          _buildFilterBar(),
          _buildStats(),
          Expanded(child: _buildTodoList()),
        ],
      ),
    );
  }

  Widget _buildInputField() {
    return Container(
      padding: const EdgeInsets.all(16),
      color: Colors.indigo[50],
      child: Row(
        children: [
          Expanded(
            child: TextField(
              controller: _textController,
              decoration: InputDecoration(
                hintText: _isEditing ? 'Edit todo...' : 'Add new todo...',
                filled: true,
                fillColor: Colors.white,
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(12),
                  borderSide: BorderSide.none,
                ),
                contentPadding: const EdgeInsets.symmetric(
                  horizontal: 16,
                  vertical: 12,
                ),
              ),
              onSubmitted: (value) {
                if (_isEditing && _editingId != null) {
                  _updateTodo(_editingId!, value);
                } else {
                  _addTodo(value);
                }
              },
            ),
          ),
          const SizedBox(width: 8),
          IconButton.filled(
            onPressed: () {
              final text = _textController.text;
              if (_isEditing && _editingId != null) {
                _updateTodo(_editingId!, text);
              } else {
                _addTodo(text);
              }
            },
            icon: Icon(_isEditing ? Icons.check : Icons.add),
            style: IconButton.styleFrom(backgroundColor: Colors.indigo),
          ),
          if (_isEditing)
            IconButton(
              onPressed: () {
                setState(() {
                  _isEditing = false;
                  _editingId = null;
                });
                _textController.clear();
              },
              icon: const Icon(Icons.close),
            ),
        ],
      ),
    );
  }

  Widget _buildFilterBar() {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: SegmentedButton<TodoFilter>(
        segments: TodoFilter.values.map((f) => ButtonSegment<TodoFilter>(
          value: f,
          label: Text(f.label),
        )).toList(),
        selected: {_filter},
        onSelectionChanged: (selection) {
          setState(() => _filter = selection.first);
        },
      ),
    );
  }

  Widget _buildStats() {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
      child: Text(
        '$_activeCount items left',
        style: TextStyle(color: Colors.grey[600], fontSize: 13),
      ),
    );
  }

  Widget _buildTodoList() {
    final filtered = _filteredTodos;

    if (filtered.isEmpty) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(Icons.check_circle_outline, size: 64, color: Colors.grey[300]),
            const SizedBox(height: 16),
            Text(
              _filter == TodoFilter.completed
                  ? 'No completed tasks'
                  : 'No tasks yet!',
              style: TextStyle(color: Colors.grey[400], fontSize: 16),
            ),
          ],
        ),
      );
    }

    return ListView.builder(
      padding: const EdgeInsets.symmetric(vertical: 8),
      itemCount: filtered.length,
      itemBuilder: (context, index) {
        final todo = filtered[index];
        return _TodoTile(
          todo: todo,
          onToggle: () => _toggleTodo(todo.id),
          onDelete: () => _deleteTodo(todo.id),
          onEdit: () {
            setState(() {
              _isEditing = true;
              _editingId = todo.id;
              _textController.text = todo.title;
            });
          },
        );
      },
    );
  }
}

class _TodoTile extends StatelessWidget {
  final Todo todo;
  final VoidCallback onToggle;
  final VoidCallback onDelete;
  final VoidCallback onEdit;

  const _TodoTile({
    required this.todo,
    required this.onToggle,
    required this.onDelete,
    required this.onEdit,
  });

  @override
  Widget build(BuildContext context) {
    return Dismissible(
      key: ValueKey(todo.id),
      background: Container(
        color: Colors.red[100],
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 20),
        child: const Icon(Icons.delete, color: Colors.red),
      ),
      direction: DismissDirection.endToStart,
      onDismissed: (_) => onDelete(),
      child: Card(
        margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
        child: ListTile(
          leading: Checkbox(
            value: todo.isCompleted,
            onChanged: (_) => onToggle(),
            activeColor: Colors.indigo,
          ),
          title: Text(
            todo.title,
            style: TextStyle(
              decoration: todo.isCompleted ? TextDecoration.lineThrough : null,
              color: todo.isCompleted ? Colors.grey : null,
            ),
          ),
          subtitle: Text(
            todo.createdAt.toString().substring(0, 16),
            style: const TextStyle(fontSize: 11),
          ),
          trailing: IconButton(
            icon: const Icon(Icons.edit, size: 18),
            onPressed: onEdit,
          ),
        ),
      ),
    );
  }
}

ThemeData _buildTheme() => ThemeData(
  useMaterial3: true,
  colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
);
```

---

## ขั้นตอนที่ 2803: Version 2 - Provider

```yaml
# pubspec.yaml dependencies for Provider version
# provider: ^6.1.2
```

```dart
// lib/version2_provider/todo_provider.dart
import 'package:flutter/foundation.dart';
import '../models/todo.dart';

/// Todo state managed by ChangeNotifier (for Provider)
class TodoProvider extends ChangeNotifier {
  final List<Todo> _todos = [];
  TodoFilter _filter = TodoFilter.all;

  List<Todo> get todos => _filteredTodos;
  TodoFilter get filter => _filter;
  int get totalCount => _todos.length;
  int get activeCount => _todos.where((t) => !t.isCompleted).length;
  int get completedCount => _todos.where((t) => t.isCompleted).length;

  List<Todo> get _filteredTodos => switch (_filter) {
    TodoFilter.all => List.from(_todos),
    TodoFilter.active => _todos.where((t) => !t.isCompleted).toList(),
    TodoFilter.completed => _todos.where((t) => t.isCompleted).toList(),
  };

  void addTodo(String title) {
    if (title.trim().isEmpty) return;
    _todos.add(Todo(
      id: DateTime.now().millisecondsSinceEpoch.toString(),
      title: title.trim(),
      createdAt: DateTime.now(),
    ));
    notifyListeners();
  }

  void toggleTodo(String id) {
    final index = _todos.indexWhere((t) => t.id == id);
    if (index != -1) {
      _todos[index] = _todos[index].copyWith(
        isCompleted: !_todos[index].isCompleted,
      );
      notifyListeners();
    }
  }

  void deleteTodo(String id) {
    _todos.removeWhere((t) => t.id == id);
    notifyListeners();
  }

  void updateTodo(String id, String newTitle) {
    if (newTitle.trim().isEmpty) return;
    final index = _todos.indexWhere((t) => t.id == id);
    if (index != -1) {
      _todos[index] = _todos[index].copyWith(title: newTitle.trim());
      notifyListeners();
    }
  }

  void setFilter(TodoFilter filter) {
    if (_filter == filter) return;
    _filter = filter;
    notifyListeners();
  }

  void clearCompleted() {
    _todos.removeWhere((t) => t.isCompleted);
    notifyListeners();
  }

  void reorderTodos(int oldIndex, int newIndex) {
    if (oldIndex < newIndex) newIndex -= 1;
    final todo = _todos.removeAt(oldIndex);
    _todos.insert(newIndex, todo);
    notifyListeners();
  }
}
```

```dart
// lib/version2_provider/todo_app_provider.dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../models/todo.dart';
import 'todo_provider.dart';

void main() {
  runApp(
    ChangeNotifierProvider(
      create: (_) => TodoProvider(),
      child: const ProviderTodoApp(),
    ),
  );
}

class ProviderTodoApp extends StatelessWidget {
  const ProviderTodoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Todo App (Provider)',
      theme: _buildTheme(),
      home: const ProviderTodoScreen(),
    );
  }
}

class ProviderTodoScreen extends StatefulWidget {
  const ProviderTodoScreen({super.key});

  @override
  State<ProviderTodoScreen> createState() => _ProviderTodoScreenState();
}

class _ProviderTodoScreenState extends State<ProviderTodoScreen> {
  final TextEditingController _textController = TextEditingController();
  bool _isEditing = false;
  String? _editingId;

  @override
  void dispose() {
    _textController.dispose();
    super.dispose();
  }

  void _handleSubmit(BuildContext context) {
    final provider = context.read<TodoProvider>();
    final text = _textController.text;

    if (_isEditing && _editingId != null) {
      provider.updateTodo(_editingId!, text);
      setState(() {
        _isEditing = false;
        _editingId = null;
      });
    } else {
      provider.addTodo(text);
    }
    _textController.clear();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Todo (Provider)'),
        backgroundColor: Colors.teal,
        foregroundColor: Colors.white,
        actions: [
          // Using Consumer for selective rebuilds
          Consumer<TodoProvider>(
            builder: (context, provider, _) {
              if (provider.completedCount == 0) return const SizedBox.shrink();
              return TextButton(
                onPressed: provider.clearCompleted,
                child: const Text(
                  'Clear Done',
                  style: TextStyle(color: Colors.white70),
                ),
              );
            },
          ),
        ],
      ),
      body: Column(
        children: [
          _buildInputField(context),
          _buildFilterBar(),
          _buildStats(),
          Expanded(child: _buildTodoList()),
        ],
      ),
    );
  }

  Widget _buildInputField(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      color: Colors.teal[50],
      child: Row(
        children: [
          Expanded(
            child: TextField(
              controller: _textController,
              decoration: InputDecoration(
                hintText: _isEditing ? 'Edit todo...' : 'Add new todo...',
                filled: true,
                fillColor: Colors.white,
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(12),
                  borderSide: BorderSide.none,
                ),
                contentPadding: const EdgeInsets.symmetric(
                  horizontal: 16,
                  vertical: 12,
                ),
              ),
              onSubmitted: (_) => _handleSubmit(context),
            ),
          ),
          const SizedBox(width: 8),
          IconButton.filled(
            onPressed: () => _handleSubmit(context),
            icon: Icon(_isEditing ? Icons.check : Icons.add),
            style: IconButton.styleFrom(backgroundColor: Colors.teal),
          ),
          if (_isEditing)
            IconButton(
              onPressed: () {
                setState(() {
                  _isEditing = false;
                  _editingId = null;
                });
                _textController.clear();
              },
              icon: const Icon(Icons.close),
            ),
        ],
      ),
    );
  }

  Widget _buildFilterBar() {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: Consumer<TodoProvider>(
        builder: (context, provider, _) {
          return SegmentedButton<TodoFilter>(
            segments: TodoFilter.values.map((f) => ButtonSegment<TodoFilter>(
              value: f,
              label: Text(f.label),
            )).toList(),
            selected: {provider.filter},
            onSelectionChanged: (sel) => provider.setFilter(sel.first),
          );
        },
      ),
    );
  }

  Widget _buildStats() {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
      // Selector only rebuilds when activeCount changes
      child: Selector<TodoProvider, int>(
        selector: (_, provider) => provider.activeCount,
        builder: (_, count, __) => Text(
          '$count items left',
          style: TextStyle(color: Colors.grey[600], fontSize: 13),
        ),
      ),
    );
  }

  Widget _buildTodoList() {
    return Consumer<TodoProvider>(
      builder: (context, provider, _) {
        final todos = provider.todos;

        if (todos.isEmpty) {
          return Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                Icon(Icons.check_circle_outline, size: 64, color: Colors.grey[300]),
                const SizedBox(height: 16),
                Text(
                  'No tasks yet!',
                  style: TextStyle(color: Colors.grey[400], fontSize: 16),
                ),
              ],
            ),
          );
        }

        return ListView.builder(
          padding: const EdgeInsets.symmetric(vertical: 8),
          itemCount: todos.length,
          itemBuilder: (context, index) {
            final todo = todos[index];
            return _ProviderTodoTile(
              todo: todo,
              onToggle: () => provider.toggleTodo(todo.id),
              onDelete: () => provider.deleteTodo(todo.id),
              onEdit: () {
                setState(() {
                  _isEditing = true;
                  _editingId = todo.id;
                  _textController.text = todo.title;
                });
              },
            );
          },
        );
      },
    );
  }
}

class _ProviderTodoTile extends StatelessWidget {
  final Todo todo;
  final VoidCallback onToggle;
  final VoidCallback onDelete;
  final VoidCallback onEdit;

  const _ProviderTodoTile({
    required this.todo,
    required this.onToggle,
    required this.onDelete,
    required this.onEdit,
  });

  @override
  Widget build(BuildContext context) {
    return Dismissible(
      key: ValueKey(todo.id),
      background: Container(
        color: Colors.red[100],
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 20),
        child: const Icon(Icons.delete, color: Colors.red),
      ),
      direction: DismissDirection.endToStart,
      onDismissed: (_) => onDelete(),
      child: Card(
        margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
        child: ListTile(
          leading: Checkbox(
            value: todo.isCompleted,
            onChanged: (_) => onToggle(),
            activeColor: Colors.teal,
          ),
          title: Text(
            todo.title,
            style: TextStyle(
              decoration: todo.isCompleted ? TextDecoration.lineThrough : null,
              color: todo.isCompleted ? Colors.grey : null,
            ),
          ),
          trailing: IconButton(
            icon: const Icon(Icons.edit, size: 18),
            onPressed: onEdit,
          ),
        ),
      ),
    );
  }
}

ThemeData _buildTheme() => ThemeData(
  useMaterial3: true,
  colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
);
```

---

## ขั้นตอนที่ 2804: Version 3 - Riverpod

```yaml
# pubspec.yaml
# flutter_riverpod: ^2.5.1
# riverpod_annotation: ^2.3.5
# build_runner: ^2.4.8 (dev)
# riverpod_generator: ^2.4.0 (dev)
```

```dart
// lib/version3_riverpod/todo_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/todo.dart';

// Filter provider
final todoFilterProvider = StateProvider<TodoFilter>(
  (ref) => TodoFilter.all,
);

// Todos state (using Notifier for complex state)
class TodosNotifier extends Notifier<List<Todo>> {
  @override
  List<Todo> build() => [];

  void addTodo(String title) {
    if (title.trim().isEmpty) return;
    state = [
      ...state,
      Todo(
        id: DateTime.now().millisecondsSinceEpoch.toString(),
        title: title.trim(),
        createdAt: DateTime.now(),
      ),
    ];
  }

  void toggleTodo(String id) {
    state = [
      for (final todo in state)
        if (todo.id == id) todo.copyWith(isCompleted: !todo.isCompleted) else todo,
    ];
  }

  void deleteTodo(String id) {
    state = state.where((t) => t.id != id).toList();
  }

  void updateTodo(String id, String newTitle) {
    if (newTitle.trim().isEmpty) return;
    state = [
      for (final todo in state)
        if (todo.id == id) todo.copyWith(title: newTitle.trim()) else todo,
    ];
  }

  void clearCompleted() {
    state = state.where((t) => !t.isCompleted).toList();
  }
}

final todosProvider = NotifierProvider<TodosNotifier, List<Todo>>(
  TodosNotifier.new,
);

// Derived/computed providers
final filteredTodosProvider = Provider<List<Todo>>((ref) {
  final todos = ref.watch(todosProvider);
  final filter = ref.watch(todoFilterProvider);

  return switch (filter) {
    TodoFilter.all => todos,
    TodoFilter.active => todos.where((t) => !t.isCompleted).toList(),
    TodoFilter.completed => todos.where((t) => t.isCompleted).toList(),
  };
});

final activeCountProvider = Provider<int>((ref) {
  final todos = ref.watch(todosProvider);
  return todos.where((t) => !t.isCompleted).length;
});

final completedCountProvider = Provider<int>((ref) {
  final todos = ref.watch(todosProvider);
  return todos.where((t) => t.isCompleted).length;
});
```

```dart
// lib/version3_riverpod/todo_app_riverpod.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/todo.dart';
import 'todo_providers.dart';

void main() {
  runApp(
    const ProviderScope(
      child: RiverpodTodoApp(),
    ),
  );
}

class RiverpodTodoApp extends StatelessWidget {
  const RiverpodTodoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Todo App (Riverpod)',
      theme: ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepOrange),
      ),
      home: const RiverpodTodoScreen(),
    );
  }
}

class RiverpodTodoScreen extends ConsumerStatefulWidget {
  const RiverpodTodoScreen({super.key});

  @override
  ConsumerState<RiverpodTodoScreen> createState() => _RiverpodTodoScreenState();
}

class _RiverpodTodoScreenState extends ConsumerState<RiverpodTodoScreen> {
  final TextEditingController _textController = TextEditingController();
  bool _isEditing = false;
  String? _editingId;

  @override
  void dispose() {
    _textController.dispose();
    super.dispose();
  }

  void _handleSubmit() {
    final text = _textController.text;
    if (_isEditing && _editingId != null) {
      ref.read(todosProvider.notifier).updateTodo(_editingId!, text);
      setState(() {
        _isEditing = false;
        _editingId = null;
      });
    } else {
      ref.read(todosProvider.notifier).addTodo(text);
    }
    _textController.clear();
  }

  @override
  Widget build(BuildContext context) {
    // Watch derived providers - only rebuilds when these change
    final activeCount = ref.watch(activeCountProvider);
    final completedCount = ref.watch(completedCountProvider);
    final currentFilter = ref.watch(todoFilterProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Todo (Riverpod)'),
        backgroundColor: Colors.deepOrange,
        foregroundColor: Colors.white,
        actions: [
          if (completedCount > 0)
            TextButton(
              onPressed: () =>
                  ref.read(todosProvider.notifier).clearCompleted(),
              child: const Text(
                'Clear Done',
                style: TextStyle(color: Colors.white70),
              ),
            ),
        ],
      ),
      body: Column(
        children: [
          _buildInputField(),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: SegmentedButton<TodoFilter>(
              segments: TodoFilter.values.map((f) => ButtonSegment<TodoFilter>(
                value: f,
                label: Text(f.label),
              )).toList(),
              selected: {currentFilter},
              onSelectionChanged: (sel) {
                ref.read(todoFilterProvider.notifier).state = sel.first;
              },
            ),
          ),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
            child: Text(
              '$activeCount items left',
              style: TextStyle(color: Colors.grey[600], fontSize: 13),
            ),
          ),
          const Expanded(child: _RiverpodTodoList()),
        ],
      ),
    );
  }

  Widget _buildInputField() {
    return Container(
      padding: const EdgeInsets.all(16),
      color: Colors.deepOrange[50],
      child: Row(
        children: [
          Expanded(
            child: TextField(
              controller: _textController,
              decoration: InputDecoration(
                hintText: _isEditing ? 'Edit todo...' : 'Add new todo...',
                filled: true,
                fillColor: Colors.white,
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(12),
                  borderSide: BorderSide.none,
                ),
                contentPadding: const EdgeInsets.symmetric(
                  horizontal: 16,
                  vertical: 12,
                ),
              ),
              onSubmitted: (_) => _handleSubmit(),
            ),
          ),
          const SizedBox(width: 8),
          IconButton.filled(
            onPressed: _handleSubmit,
            icon: Icon(_isEditing ? Icons.check : Icons.add),
            style: IconButton.styleFrom(backgroundColor: Colors.deepOrange),
          ),
          if (_isEditing)
            IconButton(
              onPressed: () {
                setState(() {
                  _isEditing = false;
                  _editingId = null;
                });
                _textController.clear();
              },
              icon: const Icon(Icons.close),
            ),
        ],
      ),
    );
  }
}

// Separate widget = only rebuilds when filteredTodos changes
class _RiverpodTodoList extends ConsumerWidget {
  const _RiverpodTodoList();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final todos = ref.watch(filteredTodosProvider);

    if (todos.isEmpty) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(Icons.check_circle_outline, size: 64, color: Colors.grey[300]),
            const SizedBox(height: 16),
            Text(
              'No tasks yet!',
              style: TextStyle(color: Colors.grey[400], fontSize: 16),
            ),
          ],
        ),
      );
    }

    return ListView.builder(
      padding: const EdgeInsets.symmetric(vertical: 8),
      itemCount: todos.length,
      itemBuilder: (context, index) {
        final todo = todos[index];
        return _RiverpodTodoTile(todo: todo);
      },
    );
  }
}

class _RiverpodTodoTile extends ConsumerWidget {
  final Todo todo;

  const _RiverpodTodoTile({required this.todo});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Dismissible(
      key: ValueKey(todo.id),
      background: Container(
        color: Colors.red[100],
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 20),
        child: const Icon(Icons.delete, color: Colors.red),
      ),
      direction: DismissDirection.endToStart,
      onDismissed: (_) =>
          ref.read(todosProvider.notifier).deleteTodo(todo.id),
      child: Card(
        margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
        child: ListTile(
          leading: Checkbox(
            value: todo.isCompleted,
            onChanged: (_) =>
                ref.read(todosProvider.notifier).toggleTodo(todo.id),
            activeColor: Colors.deepOrange,
          ),
          title: Text(
            todo.title,
            style: TextStyle(
              decoration: todo.isCompleted ? TextDecoration.lineThrough : null,
              color: todo.isCompleted ? Colors.grey : null,
            ),
          ),
          subtitle: Text(
            todo.createdAt.toString().substring(0, 16),
            style: const TextStyle(fontSize: 11),
          ),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2805: Version 4 - BLoC Pattern

```yaml
# pubspec.yaml
# flutter_bloc: ^8.1.6
# equatable: ^2.0.5
```

```dart
// lib/version4_bloc/todo_event.dart
import 'package:equatable/equatable.dart';
import '../models/todo.dart';

abstract class TodoEvent extends Equatable {
  const TodoEvent();

  @override
  List<Object?> get props => [];
}

class AddTodoEvent extends TodoEvent {
  final String title;
  const AddTodoEvent(this.title);

  @override
  List<Object?> get props => [title];
}

class ToggleTodoEvent extends TodoEvent {
  final String id;
  const ToggleTodoEvent(this.id);

  @override
  List<Object?> get props => [id];
}

class DeleteTodoEvent extends TodoEvent {
  final String id;
  const DeleteTodoEvent(this.id);

  @override
  List<Object?> get props => [id];
}

class UpdateTodoEvent extends TodoEvent {
  final String id;
  final String newTitle;
  const UpdateTodoEvent(this.id, this.newTitle);

  @override
  List<Object?> get props => [id, newTitle];
}

class SetFilterEvent extends TodoEvent {
  final TodoFilter filter;
  const SetFilterEvent(this.filter);

  @override
  List<Object?> get props => [filter];
}

class ClearCompletedEvent extends TodoEvent {
  const ClearCompletedEvent();
}
```

```dart
// lib/version4_bloc/todo_state.dart
import 'package:equatable/equatable.dart';
import '../models/todo.dart';

class TodoState extends Equatable {
  final List<Todo> todos;
  final TodoFilter filter;

  const TodoState({
    this.todos = const [],
    this.filter = TodoFilter.all,
  });

  List<Todo> get filteredTodos => switch (filter) {
    TodoFilter.all => todos,
    TodoFilter.active => todos.where((t) => !t.isCompleted).toList(),
    TodoFilter.completed => todos.where((t) => t.isCompleted).toList(),
  };

  int get activeCount => todos.where((t) => !t.isCompleted).length;
  int get completedCount => todos.where((t) => t.isCompleted).length;

  TodoState copyWith({List<Todo>? todos, TodoFilter? filter}) {
    return TodoState(
      todos: todos ?? this.todos,
      filter: filter ?? this.filter,
    );
  }

  @override
  List<Object?> get props => [todos, filter];
}
```

```dart
// lib/version4_bloc/todo_bloc.dart
import 'package:flutter_bloc/flutter_bloc.dart';
import '../models/todo.dart';
import 'todo_event.dart';
import 'todo_state.dart';

class TodoBloc extends Bloc<TodoEvent, TodoState> {
  TodoBloc() : super(const TodoState()) {
    on<AddTodoEvent>(_onAddTodo);
    on<ToggleTodoEvent>(_onToggleTodo);
    on<DeleteTodoEvent>(_onDeleteTodo);
    on<UpdateTodoEvent>(_onUpdateTodo);
    on<SetFilterEvent>(_onSetFilter);
    on<ClearCompletedEvent>(_onClearCompleted);
  }

  void _onAddTodo(AddTodoEvent event, Emitter<TodoState> emit) {
    if (event.title.trim().isEmpty) return;
    final newTodo = Todo(
      id: DateTime.now().millisecondsSinceEpoch.toString(),
      title: event.title.trim(),
      createdAt: DateTime.now(),
    );
    emit(state.copyWith(todos: [...state.todos, newTodo]));
  }

  void _onToggleTodo(ToggleTodoEvent event, Emitter<TodoState> emit) {
    emit(state.copyWith(
      todos: [
        for (final todo in state.todos)
          if (todo.id == event.id)
            todo.copyWith(isCompleted: !todo.isCompleted)
          else
            todo,
      ],
    ));
  }

  void _onDeleteTodo(DeleteTodoEvent event, Emitter<TodoState> emit) {
    emit(state.copyWith(
      todos: state.todos.where((t) => t.id != event.id).toList(),
    ));
  }

  void _onUpdateTodo(UpdateTodoEvent event, Emitter<TodoState> emit) {
    if (event.newTitle.trim().isEmpty) return;
    emit(state.copyWith(
      todos: [
        for (final todo in state.todos)
          if (todo.id == event.id)
            todo.copyWith(title: event.newTitle.trim())
          else
            todo,
      ],
    ));
  }

  void _onSetFilter(SetFilterEvent event, Emitter<TodoState> emit) {
    emit(state.copyWith(filter: event.filter));
  }

  void _onClearCompleted(ClearCompletedEvent event, Emitter<TodoState> emit) {
    emit(state.copyWith(
      todos: state.todos.where((t) => !t.isCompleted).toList(),
    ));
  }
}
```

```dart
// lib/version4_bloc/todo_app_bloc.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import '../models/todo.dart';
import 'todo_bloc.dart';
import 'todo_event.dart';
import 'todo_state.dart';

void main() {
  runApp(const BlocTodoApp());
}

class BlocTodoApp extends StatelessWidget {
  const BlocTodoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => TodoBloc(),
      child: MaterialApp(
        title: 'Todo App (BLoC)',
        theme: ThemeData(
          useMaterial3: true,
          colorScheme: ColorScheme.fromSeed(seedColor: Colors.purple),
        ),
        home: const BlocTodoScreen(),
      ),
    );
  }
}

class BlocTodoScreen extends StatefulWidget {
  const BlocTodoScreen({super.key});

  @override
  State<BlocTodoScreen> createState() => _BlocTodoScreenState();
}

class _BlocTodoScreenState extends State<BlocTodoScreen> {
  final TextEditingController _textController = TextEditingController();
  bool _isEditing = false;
  String? _editingId;

  @override
  void dispose() {
    _textController.dispose();
    super.dispose();
  }

  void _handleSubmit(BuildContext context) {
    final text = _textController.text;
    if (_isEditing && _editingId != null) {
      context.read<TodoBloc>().add(UpdateTodoEvent(_editingId!, text));
      setState(() {
        _isEditing = false;
        _editingId = null;
      });
    } else {
      context.read<TodoBloc>().add(AddTodoEvent(text));
    }
    _textController.clear();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Todo (BLoC)'),
        backgroundColor: Colors.purple,
        foregroundColor: Colors.white,
        actions: [
          BlocBuilder<TodoBloc, TodoState>(
            buildWhen: (prev, curr) =>
                prev.completedCount != curr.completedCount,
            builder: (context, state) {
              if (state.completedCount == 0) return const SizedBox.shrink();
              return TextButton(
                onPressed: () =>
                    context.read<TodoBloc>().add(const ClearCompletedEvent()),
                child: const Text(
                  'Clear Done',
                  style: TextStyle(color: Colors.white70),
                ),
              );
            },
          ),
        ],
      ),
      body: Column(
        children: [
          _buildInputField(context),
          BlocBuilder<TodoBloc, TodoState>(
            buildWhen: (prev, curr) => prev.filter != curr.filter,
            builder: (context, state) => Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
              child: SegmentedButton<TodoFilter>(
                segments: TodoFilter.values
                    .map((f) => ButtonSegment<TodoFilter>(
                          value: f,
                          label: Text(f.label),
                        ))
                    .toList(),
                selected: {state.filter},
                onSelectionChanged: (sel) => context
                    .read<TodoBloc>()
                    .add(SetFilterEvent(sel.first)),
              ),
            ),
          ),
          BlocBuilder<TodoBloc, TodoState>(
            buildWhen: (prev, curr) => prev.activeCount != curr.activeCount,
            builder: (context, state) => Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
              child: Text(
                '${state.activeCount} items left',
                style: TextStyle(color: Colors.grey[600], fontSize: 13),
              ),
            ),
          ),
          Expanded(
            child: BlocBuilder<TodoBloc, TodoState>(
              builder: (context, state) {
                final todos = state.filteredTodos;
                if (todos.isEmpty) {
                  return Center(
                    child: Text(
                      'No tasks yet!',
                      style: TextStyle(color: Colors.grey[400], fontSize: 16),
                    ),
                  );
                }
                return ListView.builder(
                  padding: const EdgeInsets.symmetric(vertical: 8),
                  itemCount: todos.length,
                  itemBuilder: (context, index) {
                    final todo = todos[index];
                    return _BlocTodoTile(
                      todo: todo,
                      onEdit: () {
                        setState(() {
                          _isEditing = true;
                          _editingId = todo.id;
                          _textController.text = todo.title;
                        });
                      },
                    );
                  },
                );
              },
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildInputField(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      color: Colors.purple[50],
      child: Row(
        children: [
          Expanded(
            child: TextField(
              controller: _textController,
              decoration: InputDecoration(
                hintText: _isEditing ? 'Edit todo...' : 'Add new todo...',
                filled: true,
                fillColor: Colors.white,
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(12),
                  borderSide: BorderSide.none,
                ),
                contentPadding: const EdgeInsets.symmetric(
                  horizontal: 16,
                  vertical: 12,
                ),
              ),
              onSubmitted: (_) => _handleSubmit(context),
            ),
          ),
          const SizedBox(width: 8),
          IconButton.filled(
            onPressed: () => _handleSubmit(context),
            icon: Icon(_isEditing ? Icons.check : Icons.add),
            style: IconButton.styleFrom(backgroundColor: Colors.purple),
          ),
          if (_isEditing)
            IconButton(
              onPressed: () {
                setState(() {
                  _isEditing = false;
                  _editingId = null;
                });
                _textController.clear();
              },
              icon: const Icon(Icons.close),
            ),
        ],
      ),
    );
  }
}

class _BlocTodoTile extends StatelessWidget {
  final Todo todo;
  final VoidCallback onEdit;

  const _BlocTodoTile({required this.todo, required this.onEdit});

  @override
  Widget build(BuildContext context) {
    return Dismissible(
      key: ValueKey(todo.id),
      background: Container(
        color: Colors.red[100],
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 20),
        child: const Icon(Icons.delete, color: Colors.red),
      ),
      direction: DismissDirection.endToStart,
      onDismissed: (_) =>
          context.read<TodoBloc>().add(DeleteTodoEvent(todo.id)),
      child: Card(
        margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
        child: ListTile(
          leading: Checkbox(
            value: todo.isCompleted,
            onChanged: (_) =>
                context.read<TodoBloc>().add(ToggleTodoEvent(todo.id)),
            activeColor: Colors.purple,
          ),
          title: Text(
            todo.title,
            style: TextStyle(
              decoration: todo.isCompleted ? TextDecoration.lineThrough : null,
              color: todo.isCompleted ? Colors.grey : null,
            ),
          ),
          subtitle: Text(
            todo.createdAt.toString().substring(0, 16),
            style: const TextStyle(fontSize: 11),
          ),
          trailing: IconButton(
            icon: const Icon(Icons.edit, size: 18),
            onPressed: onEdit,
          ),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2806: เปรียบเทียบและสรุปแต่ละ Approach

```dart
// lib/comparison_screen.dart - แสดงการเปรียบเทียบ
import 'package:flutter/material.dart';

class ComparisonScreen extends StatelessWidget {
  const ComparisonScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('State Management Comparison')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: const [
          _ComparisonCard(
            method: 'setState',
            color: Colors.indigo,
            emoji: '🔵',
            pros: [
              'No dependencies needed',
              'Simple to understand',
              'Good for local state',
              'Built into Flutter',
            ],
            cons: [
              'State scoped to one widget',
              'Hard to share across widgets',
              'Rebuilds entire widget subtree',
              'Prop drilling needed',
            ],
            useWhen: [
              'Small apps',
              'Local UI state (form, toggle)',
              'Prototypes',
            ],
            lines: '~100 lines',
            complexity: 1,
          ),
          _ComparisonCard(
            method: 'Provider',
            color: Colors.teal,
            emoji: '🟢',
            pros: [
              'Official Flutter recommendation',
              'Simple API (ChangeNotifier)',
              'Good performance with Selector',
              'Easy testing',
            ],
            cons: [
              'Boilerplate ChangeNotifier code',
              'Mutable state model',
              'No built-in immutability',
              'Manual notifyListeners()',
            ],
            useWhen: [
              'Medium apps',
              'Shared state across routes',
              'Team familiar with OOP',
            ],
            lines: '~150 lines',
            complexity: 2,
          ),
          _ComparisonCard(
            method: 'Riverpod',
            color: Colors.deepOrange,
            emoji: '🟠',
            pros: [
              'Compile-time safe',
              'Immutable state by default',
              'Powerful derived state (Provider)',
              'Easy async handling',
              'DevTools support',
            ],
            cons: [
              'New concepts to learn',
              'Heavier dependency',
              'ConsumerWidget boilerplate',
              'Code generation optional but recommended',
            ],
            useWhen: [
              'Medium to large apps',
              'Complex derived state',
              'Teams wanting type-safety',
            ],
            lines: '~130 lines',
            complexity: 3,
          ),
          _ComparisonCard(
            method: 'BLoC',
            color: Colors.purple,
            emoji: '🟣',
            pros: [
              'Clear separation of concerns',
              'Fully testable',
              'Predictable state flow',
              'Enterprise-ready',
              'DevTools + logging built-in',
            ],
            cons: [
              'Most boilerplate',
              'Steep learning curve',
              'Over-engineering for small apps',
              'Requires Equatable for equality',
            ],
            useWhen: [
              'Large enterprise apps',
              'Complex state/business logic',
              'Team needs clear contracts',
              'High test coverage required',
            ],
            lines: '~200+ lines',
            complexity: 4,
          ),
        ],
      ),
    );
  }
}

class _ComparisonCard extends StatelessWidget {
  final String method;
  final Color color;
  final String emoji;
  final List<String> pros;
  final List<String> cons;
  final List<String> useWhen;
  final String lines;
  final int complexity;

  const _ComparisonCard({
    required this.method,
    required this.color,
    required this.emoji,
    required this.pros,
    required this.cons,
    required this.useWhen,
    required this.lines,
    required this.complexity,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.only(bottom: 16),
      child: ExpansionTile(
        leading: CircleAvatar(
          backgroundColor: color,
          child: Text(emoji, style: const TextStyle(fontSize: 20)),
        ),
        title: Text(
          method,
          style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 18),
        ),
        subtitle: Row(
          children: [
            Text(lines, style: TextStyle(color: Colors.grey[600])),
            const SizedBox(width: 8),
            Row(
              children: List.generate(
                4,
                (i) => Icon(
                  Icons.circle,
                  size: 8,
                  color: i < complexity ? color : Colors.grey[300],
                ),
              ),
            ),
            const SizedBox(width: 4),
            Text(
              'Complexity',
              style: TextStyle(color: Colors.grey[600], fontSize: 11),
            ),
          ],
        ),
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                _Section(title: 'Pros', items: pros, color: Colors.green),
                const SizedBox(height: 12),
                _Section(title: 'Cons', items: cons, color: Colors.red),
                const SizedBox(height: 12),
                _Section(title: 'Use When', items: useWhen, color: color),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

class _Section extends StatelessWidget {
  final String title;
  final List<String> items;
  final Color color;

  const _Section({
    required this.title,
    required this.items,
    required this.color,
  });

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(
          title,
          style: TextStyle(
            fontWeight: FontWeight.bold,
            color: color,
          ),
        ),
        const SizedBox(height: 4),
        ...items.map((item) => Padding(
          padding: const EdgeInsets.only(left: 8, top: 2),
          child: Row(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Icon(Icons.fiber_manual_record, size: 8, color: color),
              const SizedBox(width: 8),
              Expanded(child: Text(item, style: const TextStyle(fontSize: 13))),
            ],
          ),
        )),
      ],
    );
  }
}

// Decision helper
/*
DECISION GUIDE:
─────────────────────────────────────────────────────
App Size | Recommended    | Why
─────────────────────────────────────────────────────
Tiny     | setState       | No overhead, quick to build
Small    | setState/Provider | Simple, less boilerplate
Medium   | Provider/Riverpod | Balance of simplicity + power
Large    | Riverpod/BLoC  | Testability, scalability
Enterprise | BLoC         | Clear contracts, full testing
─────────────────────────────────────────────────────
*/
```

---

**← [Part 72](part-72-advanced-animations-physics.md)**
**ต่อไป: [Part 74 →](part-74-flutter-desktop-advanced.md)**
