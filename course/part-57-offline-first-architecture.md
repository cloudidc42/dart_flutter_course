# Part 57: Offline-First Architecture
## ขั้นตอนที่ 2161-2200

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ Offline-First Pattern และทำไมถึงสำคัญ
- ใช้ Drift (SQLite) เป็น local database หลัก
- สร้าง Sync Queue สำหรับ offline operations
- จัดการ Conflict Resolution เมื่อข้อมูล sync
- ตรวจจับ network connectivity ด้วย connectivity_plus
- Background sync เมื่อ connectivity กลับมา

---

## ขั้นตอนที่ 2161: Dependencies Setup

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  drift: ^2.13.1
  drift_flutter: ^0.1.0
  sqlite3_flutter_libs: ^0.5.15
  connectivity_plus: ^5.0.2
  dio: ^5.3.3
  riverpod: ^2.4.9
  flutter_riverpod: ^2.4.9
  uuid: ^4.2.1
  json_annotation: ^4.8.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  drift_dev: ^2.13.1
  build_runner: ^2.4.6
  json_serializable: ^6.7.1
```

---

## ขั้นตอนที่ 2162: Database Schema with Drift

```dart
// lib/core/database/app_database.dart
import 'dart:io';
import 'package:drift/drift.dart';
import 'package:drift_flutter/drift_flutter.dart';
import 'package:uuid/uuid.dart';

part 'app_database.g.dart';

// --- Tables ---

class TodoItems extends Table {
  TextColumn get id => text().clientDefault(() => const Uuid().v4())();
  TextColumn get title => text().withLength(min: 1, max: 500)();
  TextColumn get description => text().nullable()();
  BoolColumn get isCompleted => boolean().withDefault(const Constant(false))();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
  DateTimeColumn get updatedAt => dateTime().withDefault(currentDateAndTime)();
  DateTimeColumn get syncedAt => dateTime().nullable()();
  IntColumn get version => integer().withDefault(const Constant(1))();

  @override
  Set<Column> get primaryKey => {id};
}

class SyncQueue extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get entityType => text()();         // e.g., 'todo'
  TextColumn get entityId => text()();           // entity's UUID
  TextColumn get operation => text()();          // 'CREATE', 'UPDATE', 'DELETE'
  TextColumn get payload => text()();            // JSON payload
  IntColumn get retryCount => integer().withDefault(const Constant(0))();
  IntColumn get maxRetries => integer().withDefault(const Constant(3))();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
  DateTimeColumn get nextRetryAt => dateTime().withDefault(currentDateAndTime)();
  BoolColumn get isFailed => boolean().withDefault(const Constant(false))();
  TextColumn get errorMessage => text().nullable()();
}

// --- Database ---

@DriftDatabase(tables: [TodoItems, SyncQueue])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());

  @override
  int get schemaVersion => 1;

  @override
  MigrationStrategy get migration => MigrationStrategy(
        onCreate: (m) async {
          await m.createAll();
        },
        onUpgrade: (m, from, to) async {
          // Handle migrations here
        },
      );

  static QueryExecutor _openConnection() {
    return driftDatabase(name: 'app_database');
  }
}
```

---

## ขั้นตอนที่ 2163: Todo Repository with Offline Support

```dart
// lib/features/todos/data/repositories/todo_repository.dart
import 'dart:convert';
import 'package:drift/drift.dart';
import 'package:uuid/uuid.dart';
import '../../../../core/database/app_database.dart';
import '../../../../core/sync/sync_service.dart';

class TodoRepository {
  final AppDatabase _db;
  final SyncService _syncService;

  TodoRepository(this._db, this._syncService);

  // --- Read Operations (always from local) ---

  Stream<List<TodoItem>> watchAllTodos() {
    return (_db.select(_db.todoItems)
          ..orderBy([
            (t) => OrderingTerm(
                  expression: t.createdAt,
                  mode: OrderingMode.desc,
                ),
          ]))
        .watch();
  }

  Future<TodoItem?> getTodoById(String id) {
    return (_db.select(_db.todoItems)
          ..where((t) => t.id.equals(id)))
        .getSingleOrNull();
  }

  Future<List<TodoItem>> getUnsyncedTodos() {
    return (_db.select(_db.todoItems)
          ..where((t) => t.syncedAt.isNull()))
        .get();
  }

  // --- Write Operations (write locally + queue for sync) ---

  Future<TodoItem> createTodo({
    required String title,
    String? description,
  }) async {
    final id = const Uuid().v4();
    final now = DateTime.now();

    final todo = TodoItemsCompanion.insert(
      id: Value(id),
      title: title,
      description: Value(description),
      isCompleted: const Value(false),
      createdAt: Value(now),
      updatedAt: Value(now),
    );

    await _db.into(_db.todoItems).insert(todo);

    // Queue for sync
    await _syncService.enqueue(
      entityType: 'todo',
      entityId: id,
      operation: SyncOperation.create,
      payload: {
        'id': id,
        'title': title,
        'description': description,
        'isCompleted': false,
        'createdAt': now.toIso8601String(),
        'updatedAt': now.toIso8601String(),
      },
    );

    return (await getTodoById(id))!;
  }

  Future<TodoItem> updateTodo({
    required String id,
    String? title,
    String? description,
    bool? isCompleted,
  }) async {
    final existing = await getTodoById(id);
    if (existing == null) throw Exception('Todo $id not found');

    final now = DateTime.now();
    final newVersion = existing.version + 1;

    await (_db.update(_db.todoItems)..where((t) => t.id.equals(id))).write(
      TodoItemsCompanion(
        title: title != null ? Value(title) : const Value.absent(),
        description:
            description != null ? Value(description) : const Value.absent(),
        isCompleted:
            isCompleted != null ? Value(isCompleted) : const Value.absent(),
        updatedAt: Value(now),
        syncedAt: const Value(null), // mark as unsynced
        version: Value(newVersion),
      ),
    );

    // Queue for sync with version for conflict detection
    await _syncService.enqueue(
      entityType: 'todo',
      entityId: id,
      operation: SyncOperation.update,
      payload: {
        'id': id,
        if (title != null) 'title': title,
        if (description != null) 'description': description,
        if (isCompleted != null) 'isCompleted': isCompleted,
        'updatedAt': now.toIso8601String(),
        'version': newVersion,
      },
    );

    return (await getTodoById(id))!;
  }

  Future<void> deleteTodo(String id) async {
    await (_db.delete(_db.todoItems)..where((t) => t.id.equals(id))).go();

    await _syncService.enqueue(
      entityType: 'todo',
      entityId: id,
      operation: SyncOperation.delete,
      payload: {'id': id},
    );
  }

  // --- Sync operations ---

  Future<void> markAsSynced(String id) async {
    await (_db.update(_db.todoItems)..where((t) => t.id.equals(id))).write(
      TodoItemsCompanion(syncedAt: Value(DateTime.now())),
    );
  }

  Future<void> applyServerUpdate(Map<String, dynamic> serverData) async {
    final id = serverData['id'] as String;
    final serverVersion = serverData['version'] as int;

    final existing = await getTodoById(id);

    if (existing == null) {
      // New item from server
      await _db.into(_db.todoItems).insertOnConflictUpdate(
            TodoItemsCompanion.insert(
              id: Value(id),
              title: serverData['title'] as String,
              description: Value(serverData['description'] as String?),
              isCompleted: Value(serverData['isCompleted'] as bool),
              createdAt: Value(DateTime.parse(serverData['createdAt'])),
              updatedAt: Value(DateTime.parse(serverData['updatedAt'])),
              syncedAt: Value(DateTime.now()),
              version: Value(serverVersion),
            ),
          );
    } else {
      // Conflict resolution: server wins if server version is higher
      if (serverVersion >= existing.version) {
        await (_db.update(_db.todoItems)..where((t) => t.id.equals(id))).write(
          TodoItemsCompanion(
            title: Value(serverData['title'] as String),
            description: Value(serverData['description'] as String?),
            isCompleted: Value(serverData['isCompleted'] as bool),
            updatedAt: Value(DateTime.parse(serverData['updatedAt'])),
            syncedAt: Value(DateTime.now()),
            version: Value(serverVersion),
          ),
        );
      }
      // else: keep local version (local wins strategy)
    }
  }
}
```

---

## ขั้นตอนที่ 2164: Sync Service with Queue Management

```dart
// lib/core/sync/sync_service.dart
import 'dart:convert';
import 'dart:math';
import 'package:drift/drift.dart';
import '../database/app_database.dart';
import '../network/api_client.dart';

enum SyncOperation { create, update, delete }

class SyncService {
  final AppDatabase _db;
  final ApiClient _apiClient;

  SyncService(this._db, this._apiClient);

  Future<void> enqueue({
    required String entityType,
    required String entityId,
    required SyncOperation operation,
    required Map<String, dynamic> payload,
  }) async {
    final now = DateTime.now();
    await _db.into(_db.syncQueue).insert(
          SyncQueueCompanion.insert(
            entityType: entityType,
            entityId: entityId,
            operation: operation.name.toUpperCase(),
            payload: jsonEncode(payload),
            createdAt: Value(now),
            nextRetryAt: Value(now),
          ),
        );
  }

  Future<List<SyncQueueData>> getPendingItems() {
    return (_db.select(_db.syncQueue)
          ..where((q) =>
              q.isFailed.equals(false) &
              q.retryCount.isSmallerOrEqualValue(q.maxRetries) &
              q.nextRetryAt.isSmallerOrEqualValue(DateTime.now()))
          ..orderBy([
            (q) => OrderingTerm(expression: q.createdAt),
          ]))
        .get();
  }

  Future<void> processPendingItems() async {
    final items = await getPendingItems();

    for (final item in items) {
      await _processItem(item);
    }
  }

  Future<void> _processItem(SyncQueueData item) async {
    try {
      final payload = jsonDecode(item.payload) as Map<String, dynamic>;

      await _apiClient.sync(
        entityType: item.entityType,
        entityId: item.entityId,
        operation: item.operation,
        payload: payload,
      );

      // Success: remove from queue
      await (_db.delete(_db.syncQueue)
            ..where((q) => q.id.equals(item.id)))
          .go();
    } catch (e) {
      final newRetryCount = item.retryCount + 1;
      final maxRetries = item.maxRetries;

      if (newRetryCount >= maxRetries) {
        // Mark as permanently failed
        await (_db.update(_db.syncQueue)
              ..where((q) => q.id.equals(item.id)))
            .write(SyncQueueCompanion(
          isFailed: const Value(true),
          errorMessage: Value(e.toString()),
          retryCount: Value(newRetryCount),
        ));
      } else {
        // Exponential backoff: 2^retryCount * base_delay seconds
        final backoffSeconds = pow(2, newRetryCount).toInt() * 5;
        final nextRetry = DateTime.now().add(Duration(seconds: backoffSeconds));

        await (_db.update(_db.syncQueue)
              ..where((q) => q.id.equals(item.id)))
            .write(SyncQueueCompanion(
          retryCount: Value(newRetryCount),
          nextRetryAt: Value(nextRetry),
          errorMessage: Value(e.toString()),
        ));
      }
    }
  }

  Future<void> retryFailedItems() async {
    await (_db.update(_db.syncQueue)
          ..where((q) => q.isFailed.equals(true)))
        .write(SyncQueueCompanion(
      isFailed: const Value(false),
      retryCount: const Value(0),
      nextRetryAt: Value(DateTime.now()),
      errorMessage: const Value(null),
    ));
  }

  Future<int> getPendingCount() async {
    final count = await (_db.select(_db.syncQueue)
          ..where((q) => q.isFailed.equals(false)))
        .get();
    return count.length;
  }

  Stream<int> watchPendingCount() {
    return (_db.select(_db.syncQueue)
          ..where((q) => q.isFailed.equals(false)))
        .watch()
        .map((items) => items.length);
  }
}
```

---

## ขั้นตอนที่ 2165: Connectivity Monitor

```dart
// lib/core/network/connectivity_monitor.dart
import 'dart:async';
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:flutter/foundation.dart';

enum NetworkStatus { online, offline }

class ConnectivityMonitor {
  final Connectivity _connectivity;
  final StreamController<NetworkStatus> _controller =
      StreamController<NetworkStatus>.broadcast();

  StreamSubscription<List<ConnectivityResult>>? _subscription;
  NetworkStatus _currentStatus = NetworkStatus.offline;

  ConnectivityMonitor(this._connectivity);

  NetworkStatus get currentStatus => _currentStatus;
  bool get isOnline => _currentStatus == NetworkStatus.online;

  Stream<NetworkStatus> get onStatusChanged => _controller.stream;

  Future<void> initialize() async {
    final results = await _connectivity.checkConnectivity();
    _currentStatus = _resultToStatus(results);

    _subscription = _connectivity.onConnectivityChanged.listen((results) {
      final status = _resultToStatus(results);
      if (status != _currentStatus) {
        _currentStatus = status;
        _controller.add(status);
        debugPrint('[ConnectivityMonitor] Status changed: $status');
      }
    });
  }

  NetworkStatus _resultToStatus(List<ConnectivityResult> results) {
    if (results.contains(ConnectivityResult.none) || results.isEmpty) {
      return NetworkStatus.offline;
    }
    return NetworkStatus.online;
  }

  void dispose() {
    _subscription?.cancel();
    _controller.close();
  }
}
```

---

## ขั้นตอนที่ 2166: Background Sync Coordinator

```dart
// lib/core/sync/sync_coordinator.dart
import 'dart:async';
import 'package:flutter/foundation.dart';
import '../network/connectivity_monitor.dart';
import 'sync_service.dart';

class SyncCoordinator {
  final SyncService _syncService;
  final ConnectivityMonitor _connectivityMonitor;

  StreamSubscription<NetworkStatus>? _connectivitySubscription;
  Timer? _periodicSyncTimer;
  bool _isSyncing = false;

  static const Duration _periodicSyncInterval = Duration(minutes: 5);

  SyncCoordinator(this._syncService, this._connectivityMonitor);

  void start() {
    // Listen for connectivity changes
    _connectivitySubscription =
        _connectivityMonitor.onStatusChanged.listen((status) {
      if (status == NetworkStatus.online) {
        debugPrint('[SyncCoordinator] Back online - triggering sync');
        _triggerSync();
      }
    });

    // Also start periodic sync
    _periodicSyncTimer = Timer.periodic(_periodicSyncInterval, (_) {
      if (_connectivityMonitor.isOnline) {
        _triggerSync();
      }
    });

    // Immediate sync if online
    if (_connectivityMonitor.isOnline) {
      _triggerSync();
    }
  }

  Future<void> _triggerSync() async {
    if (_isSyncing) return;
    _isSyncing = true;

    try {
      await _syncService.processPendingItems();
      debugPrint('[SyncCoordinator] Sync completed');
    } catch (e) {
      debugPrint('[SyncCoordinator] Sync error: $e');
    } finally {
      _isSyncing = false;
    }
  }

  Future<void> forceSyncNow() => _triggerSync();

  void stop() {
    _connectivitySubscription?.cancel();
    _periodicSyncTimer?.cancel();
  }

  void dispose() {
    stop();
  }
}
```

---

## ขั้นตอนที่ 2167: API Client for Sync

```dart
// lib/core/network/api_client.dart
import 'package:dio/dio.dart';

class ApiClient {
  final Dio _dio;

  ApiClient(this._dio);

  static ApiClient create({required String baseUrl}) {
    final dio = Dio(BaseOptions(
      baseUrl: baseUrl,
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 30),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    ));

    dio.interceptors.addAll([
      LogInterceptor(requestBody: true, responseBody: true),
    ]);

    return ApiClient(dio);
  }

  Future<void> sync({
    required String entityType,
    required String entityId,
    required String operation,
    required Map<String, dynamic> payload,
  }) async {
    await _dio.post('/sync', data: {
      'entityType': entityType,
      'entityId': entityId,
      'operation': operation,
      'payload': payload,
      'clientTimestamp': DateTime.now().toIso8601String(),
    });
  }

  Future<List<Map<String, dynamic>>> fetchServerChanges({
    required DateTime since,
  }) async {
    final response = await _dio.get('/changes', queryParameters: {
      'since': since.toIso8601String(),
    });

    return (response.data as List)
        .cast<Map<String, dynamic>>();
  }
}
```

---

## ขั้นตอนที่ 2168: Riverpod Providers for Offline-First

```dart
// lib/core/providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:dio/dio.dart';
import 'database/app_database.dart';
import 'network/api_client.dart';
import 'network/connectivity_monitor.dart';
import 'sync/sync_coordinator.dart';
import 'sync/sync_service.dart';
import '../features/todos/data/repositories/todo_repository.dart';

final databaseProvider = Provider<AppDatabase>((ref) {
  final db = AppDatabase();
  ref.onDispose(db.close);
  return db;
});

final apiClientProvider = Provider<ApiClient>((ref) {
  return ApiClient.create(baseUrl: 'https://api.example.com');
});

final connectivityMonitorProvider = Provider<ConnectivityMonitor>((ref) {
  final monitor = ConnectivityMonitor(Connectivity());
  ref.onDispose(monitor.dispose);
  return monitor;
});

final syncServiceProvider = Provider<SyncService>((ref) {
  return SyncService(
    ref.watch(databaseProvider),
    ref.watch(apiClientProvider),
  );
});

final syncCoordinatorProvider = Provider<SyncCoordinator>((ref) {
  final coordinator = SyncCoordinator(
    ref.watch(syncServiceProvider),
    ref.watch(connectivityMonitorProvider),
  );
  ref.onDispose(coordinator.dispose);
  return coordinator;
});

final todoRepositoryProvider = Provider<TodoRepository>((ref) {
  return TodoRepository(
    ref.watch(databaseProvider),
    ref.watch(syncServiceProvider),
  );
});

final todosProvider = StreamProvider<List<TodoItem>>((ref) {
  return ref.watch(todoRepositoryProvider).watchAllTodos();
});

final networkStatusProvider = StreamProvider<NetworkStatus>((ref) {
  return ref.watch(connectivityMonitorProvider).onStatusChanged;
});

final pendingSyncCountProvider = StreamProvider<int>((ref) {
  return ref.watch(syncServiceProvider).watchPendingCount();
});
```

---

## ขั้นตอนที่ 2169: Full UI with Offline Indicator

```dart
// lib/features/todos/presentation/pages/todos_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../../core/providers.dart';
import '../../../../core/network/connectivity_monitor.dart';

class TodosPage extends ConsumerStatefulWidget {
  const TodosPage({super.key});

  @override
  ConsumerState<TodosPage> createState() => _TodosPageState();
}

class _TodosPageState extends ConsumerState<TodosPage> {
  @override
  void initState() {
    super.initState();
    // Initialize connectivity and start sync
    WidgetsBinding.instance.addPostFrameCallback((_) async {
      final monitor = ref.read(connectivityMonitorProvider);
      await monitor.initialize();
      ref.read(syncCoordinatorProvider).start();
    });
  }

  @override
  Widget build(BuildContext context) {
    final todosAsync = ref.watch(todosProvider);
    final networkAsync = ref.watch(networkStatusProvider);
    final pendingAsync = ref.watch(pendingSyncCountProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Todos (Offline-First)'),
        actions: [
          pendingAsync.when(
            data: (count) => count > 0
                ? Padding(
                    padding: const EdgeInsets.only(right: 8),
                    child: Chip(
                      label: Text('$count pending'),
                      backgroundColor: Colors.orange.shade100,
                    ),
                  )
                : const SizedBox.shrink(),
            loading: () => const SizedBox.shrink(),
            error: (_, __) => const SizedBox.shrink(),
          ),
        ],
      ),
      body: Column(
        children: [
          // Offline Banner
          networkAsync.when(
            data: (status) => status == NetworkStatus.offline
                ? Container(
                    width: double.infinity,
                    color: Colors.red.shade700,
                    padding: const EdgeInsets.symmetric(
                        vertical: 6, horizontal: 16),
                    child: const Row(
                      children: [
                        Icon(Icons.wifi_off, color: Colors.white, size: 16),
                        SizedBox(width: 8),
                        Text(
                          'You are offline. Changes will sync when connected.',
                          style: TextStyle(color: Colors.white, fontSize: 12),
                        ),
                      ],
                    ),
                  )
                : const SizedBox.shrink(),
            loading: () => const SizedBox.shrink(),
            error: (_, __) => const SizedBox.shrink(),
          ),
          // Todo List
          Expanded(
            child: todosAsync.when(
              data: (todos) => todos.isEmpty
                  ? const Center(child: Text('No todos yet. Add one!'))
                  : ListView.builder(
                      itemCount: todos.length,
                      itemBuilder: (context, index) {
                        final todo = todos[index];
                        return _TodoTile(todo: todo);
                      },
                    ),
              loading: () =>
                  const Center(child: CircularProgressIndicator()),
              error: (e, _) => Center(child: Text('Error: $e')),
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
    final controller = TextEditingController();
    showDialog(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('New Todo'),
        content: TextField(
          controller: controller,
          decoration: const InputDecoration(hintText: 'Todo title'),
          autofocus: true,
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(ctx),
            child: const Text('Cancel'),
          ),
          ElevatedButton(
            onPressed: () async {
              if (controller.text.isNotEmpty) {
                await ref.read(todoRepositoryProvider).createTodo(
                      title: controller.text,
                    );
                if (ctx.mounted) Navigator.pop(ctx);
              }
            },
            child: const Text('Add'),
          ),
        ],
      ),
    );
  }
}

class _TodoTile extends ConsumerWidget {
  final TodoItem todo;
  const _TodoTile({required this.todo});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ListTile(
      leading: Checkbox(
        value: todo.isCompleted,
        onChanged: (val) => ref.read(todoRepositoryProvider).updateTodo(
              id: todo.id,
              isCompleted: val,
            ),
      ),
      title: Text(
        todo.title,
        style: TextStyle(
          decoration:
              todo.isCompleted ? TextDecoration.lineThrough : null,
        ),
      ),
      subtitle: todo.syncedAt == null
          ? const Text(
              'Not synced',
              style: TextStyle(color: Colors.orange, fontSize: 11),
            )
          : null,
      trailing: IconButton(
        icon: const Icon(Icons.delete_outline),
        onPressed: () =>
            ref.read(todoRepositoryProvider).deleteTodo(todo.id),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2170: Conflict Resolution Strategies

```dart
// lib/core/sync/conflict_resolver.dart

/// Conflict resolution strategy enum
enum ConflictStrategy {
  /// Server always wins
  serverWins,
  /// Client always wins
  clientWins,
  /// The most recently updated version wins
  lastWriteWins,
  /// Custom merge logic
  merge,
}

/// Conflict data holder
class SyncConflict<T> {
  final T localVersion;
  final T serverVersion;
  final DateTime localUpdatedAt;
  final DateTime serverUpdatedAt;

  const SyncConflict({
    required this.localVersion,
    required this.serverVersion,
    required this.localUpdatedAt,
    required this.serverUpdatedAt,
  });
}

/// Generic conflict resolver
class ConflictResolver<T> {
  final ConflictStrategy strategy;
  final T Function(T local, T server)? customMerge;

  ConflictResolver({
    this.strategy = ConflictStrategy.lastWriteWins,
    this.customMerge,
  });

  T resolve(SyncConflict<T> conflict) {
    switch (strategy) {
      case ConflictStrategy.serverWins:
        return conflict.serverVersion;

      case ConflictStrategy.clientWins:
        return conflict.localVersion;

      case ConflictStrategy.lastWriteWins:
        return conflict.serverUpdatedAt.isAfter(conflict.localUpdatedAt)
            ? conflict.serverVersion
            : conflict.localVersion;

      case ConflictStrategy.merge:
        if (customMerge == null) {
          throw StateError('customMerge must be provided for merge strategy');
        }
        return customMerge!(conflict.localVersion, conflict.serverVersion);
    }
  }
}

/// Example: Todo-specific conflict resolver with merge
class TodoConflictResolver {
  static Map<String, dynamic> merge(
    Map<String, dynamic> local,
    Map<String, dynamic> server,
  ) {
    // Custom merge: combine non-conflicting fields
    return {
      'id': local['id'],
      // If titles differ, prefer server but append local note
      'title': local['title'] != server['title']
          ? '${server['title']} [local: ${local['title']}]'
          : server['title'],
      // Prefer whichever marks completed as true
      'isCompleted': (local['isCompleted'] as bool) ||
          (server['isCompleted'] as bool),
      // Take the latest updatedAt
      'updatedAt': DateTime.parse(local['updatedAt'])
              .isAfter(DateTime.parse(server['updatedAt']))
          ? local['updatedAt']
          : server['updatedAt'],
      // Keep highest version
      'version': (local['version'] as int) > (server['version'] as int)
          ? local['version']
          : server['version'],
    };
  }
}
```

---

**← [Part 56](part-56-flutter-plugins.md)**
**ต่อไป: [Part 58 →](part-58-design-system.md)**
