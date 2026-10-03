# Part 88: Advanced Local Database
## ขั้นตอนที่ 3401-3440

---

## 🎯 เป้าหมายของ Part นี้

- เขียน Drift advanced queries (joins, aggregates, transactions)
- ใช้ Hive สำหรับ fast key-value storage
- ใช้ ObjectBox สำหรับ complex relational data
- ออกแบบ Database migration strategies
- เข้ารหัส local database ด้วย SQLCipher
- เลือก database ที่เหมาะกับ use case แต่ละแบบ

---

## ขั้นตอนที่ 3401: Drift Advanced Queries

```yaml
# pubspec.yaml
dependencies:
  drift: ^2.14.1
  sqlite3_flutter_libs: ^0.5.18
  path_provider: ^2.1.2
  path: ^1.9.0

dev_dependencies:
  drift_dev: ^2.14.1
  build_runner: ^2.4.7
```

```dart
// lib/database/app_database.dart
import 'dart:io';
import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:path_provider/path_provider.dart';
import 'package:path/path.dart' as p;

part 'app_database.g.dart';

// Table definitions
class Users extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text().withLength(min: 1, max: 100)();
  TextColumn get email => text().unique()();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
  BoolColumn get isActive => boolean().withDefault(const Constant(true))();
}

class Products extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text().withLength(min: 1, max: 200)();
  RealColumn get price => real()();
  IntColumn get categoryId => integer().references(Categories, #id)();
  IntColumn get stock => integer().withDefault(const Constant(0))();
  DateTimeColumn get updatedAt => dateTime().withDefault(currentDateAndTime)();
}

class Categories extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text().unique()();
  TextColumn get description => text().nullable()();
}

class Orders extends Table {
  IntColumn get id => integer().autoIncrement()();
  IntColumn get userId => integer().references(Users, #id)();
  DateTimeColumn get createdAt => dateTime().withDefault(currentDateAndTime)();
  TextColumn get status => text().withDefault(const Constant('pending'))();
  RealColumn get totalAmount => real().withDefault(const Constant(0))();
}

class OrderItems extends Table {
  IntColumn get id => integer().autoIncrement()();
  IntColumn get orderId => integer().references(Orders, #id)();
  IntColumn get productId => integer().references(Products, #id)();
  IntColumn get quantity => integer()();
  RealColumn get unitPrice => real()();

  @override
  Set<Column> get primaryKey => {orderId, productId};
}

@DriftDatabase(tables: [Users, Products, Categories, Orders, OrderItems])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());

  @override
  int get schemaVersion => 2;

  @override
  MigrationStrategy get migration => MigrationStrategy(
        onCreate: (m) => m.createAll(),
        onUpgrade: (m, from, to) async {
          if (from < 2) {
            await m.addColumn(products, products.updatedAt);
          }
        },
        beforeOpen: (details) async {
          await customStatement('PRAGMA foreign_keys = ON');
        },
      );

  // ======== User queries ========

  Future<int> insertUser(UsersCompanion user) => into(users).insert(user);

  Future<List<User>> getAllActiveUsers() =>
      (select(users)..where((u) => u.isActive.equals(true))).get();

  Stream<List<User>> watchAllUsers() => select(users).watch();

  Future<User?> getUserByEmail(String email) =>
      (select(users)..where((u) => u.email.equals(email)))
          .getSingleOrNull();

  Future<int> updateUser(int id, UsersCompanion data) =>
      (update(users)..where((u) => u.id.equals(id))).write(data);

  Future<int> deactivateUser(int id) =>
      (update(users)..where((u) => u.id.equals(id)))
          .write(const UsersCompanion(isActive: Value(false)));

  // ======== Product queries ========

  Future<List<Product>> getProductsByCategory(int categoryId) =>
      (select(products)
            ..where((p) => p.categoryId.equals(categoryId))
            ..orderBy([(p) => OrderingTerm.asc(p.name)]))
          .get();

  Future<List<Product>> searchProducts(String query) =>
      (select(products)
            ..where((p) => p.name.like('%$query%')))
          .get();

  Future<List<Product>> getLowStockProducts(int threshold) =>
      (select(products)
            ..where((p) => p.stock.isSmallerThanValue(threshold))
            ..orderBy([(p) => OrderingTerm.asc(p.stock)]))
          .get();

  // ======== Advanced JOIN queries ========

  Future<List<OrderWithUser>> getOrdersWithUsers() {
    final query = select(orders).join([
      innerJoin(users, users.id.equalsExp(orders.userId)),
    ]);

    return query.map((row) {
      return OrderWithUser(
        order: row.readTable(orders),
        user: row.readTable(users),
      );
    }).get();
  }

  Future<List<OrderItemWithProduct>> getOrderItems(int orderId) {
    final query = (select(orderItems)
          ..where((oi) => oi.orderId.equals(orderId)))
        .join([
      innerJoin(products, products.id.equalsExp(orderItems.productId)),
    ]);

    return query.map((row) {
      return OrderItemWithProduct(
        item: row.readTable(orderItems),
        product: row.readTable(products),
      );
    }).get();
  }

  // ======== Aggregate queries ========

  Future<Map<String, dynamic>> getOrderStats() async {
    final countQuery = selectOnly(orders)
      ..addColumns([orders.id.count(), orders.totalAmount.sum()]);

    final row = await countQuery.getSingle();
    return {
      'total_orders': row.read(orders.id.count()),
      'total_revenue': row.read(orders.totalAmount.sum()),
    };
  }

  Future<List<CategorySalesData>> getSalesByCategory() {
    final query = select(orderItems).join([
      innerJoin(products, products.id.equalsExp(orderItems.productId)),
      innerJoin(categories, categories.id.equalsExp(products.categoryId)),
    ]);

    query
      ..addColumns([
        categories.name,
        orderItems.quantity.sum(),
        orderItems.unitPrice.sum(),
      ])
      ..groupBy([categories.id]);

    return query.map((row) {
      return CategorySalesData(
        categoryName: row.read(categories.name)!,
        totalQuantity: row.read(orderItems.quantity.sum()) ?? 0,
        totalRevenue: row.read(orderItems.unitPrice.sum()) ?? 0,
      );
    }).get();
  }

  Future<List<TopCustomer>> getTopCustomers({int limit = 10}) {
    final totalSpent = orders.totalAmount.sum();
    final orderCount = orders.id.count();

    final query = select(orders).join([
      innerJoin(users, users.id.equalsExp(orders.userId)),
    ]);

    query
      ..addColumns([totalSpent, orderCount])
      ..groupBy([users.id])
      ..orderBy([OrderingTerm.desc(totalSpent)])
      ..limit(limit);

    return query.map((row) {
      return TopCustomer(
        user: row.readTable(users),
        totalSpent: row.read(totalSpent) ?? 0,
        orderCount: row.read(orderCount),
      );
    }).get();
  }

  // ======== Transaction example ========

  Future<Order> createOrder({
    required int userId,
    required List<({int productId, int quantity})> items,
  }) async {
    return transaction(() async {
      // Calculate total and validate stock
      double total = 0;
      final productDetails = <int, Product>{};

      for (final item in items) {
        final product = await (select(products)
              ..where((p) => p.id.equals(item.productId)))
            .getSingleOrNull();

        if (product == null) {
          throw Exception('Product ${item.productId} not found');
        }
        if (product.stock < item.quantity) {
          throw Exception('Insufficient stock for ${product.name}');
        }

        total += product.price * item.quantity;
        productDetails[item.productId] = product;
      }

      // Create order
      final orderId = await into(orders).insert(OrdersCompanion.insert(
        userId: userId,
        totalAmount: Value(total),
      ));

      // Insert order items
      for (final item in items) {
        final product = productDetails[item.productId]!;
        await into(orderItems).insert(OrderItemsCompanion.insert(
          orderId: orderId,
          productId: item.productId,
          quantity: item.quantity,
          unitPrice: product.price,
        ));

        // Update stock
        await (update(products)
              ..where((p) => p.id.equals(item.productId)))
            .write(ProductsCompanion(
          stock: Value(product.stock - item.quantity),
          updatedAt: Value(DateTime.now()),
        ));
      }

      return (select(orders)..where((o) => o.id.equals(orderId))).getSingle();
    });
  }
}

// Data classes for joined queries
class OrderWithUser {
  final Order order;
  final User user;
  const OrderWithUser({required this.order, required this.user});
}

class OrderItemWithProduct {
  final OrderItem item;
  final Product product;
  const OrderItemWithProduct({required this.item, required this.product});
}

class CategorySalesData {
  final String categoryName;
  final int totalQuantity;
  final double totalRevenue;
  const CategorySalesData({
    required this.categoryName,
    required this.totalQuantity,
    required this.totalRevenue,
  });
}

class TopCustomer {
  final User user;
  final double totalSpent;
  final int orderCount;
  const TopCustomer({
    required this.user,
    required this.totalSpent,
    required this.orderCount,
  });
}

LazyDatabase _openConnection() {
  return LazyDatabase(() async {
    final dbFolder = await getApplicationDocumentsDirectory();
    final file = File(p.join(dbFolder.path, 'app.db'));
    return NativeDatabase.createInBackground(file);
  });
}
```

## ขั้นตอนที่ 3402: Hive for Fast Key-Value Storage

```yaml
# pubspec.yaml additions
dependencies:
  hive: ^2.2.3
  hive_flutter: ^1.1.0

dev_dependencies:
  hive_generator: ^2.0.1
```

```dart
// lib/database/hive/user_preferences.dart
import 'package:hive_flutter/hive_flutter.dart';

part 'user_preferences.g.dart';

@HiveType(typeId: 0)
class UserPreferences extends HiveObject {
  @HiveField(0)
  late String theme;

  @HiveField(1)
  late String language;

  @HiveField(2)
  late bool notificationsEnabled;

  @HiveField(3)
  late int fontSize;

  @HiveField(4)
  late List<String> recentSearches;

  UserPreferences({
    this.theme = 'light',
    this.language = 'en',
    this.notificationsEnabled = true,
    this.fontSize = 14,
    List<String>? recentSearches,
  }) : recentSearches = recentSearches ?? [];
}

@HiveType(typeId: 1)
class CachedProduct extends HiveObject {
  @HiveField(0)
  late int id;

  @HiveField(1)
  late String name;

  @HiveField(2)
  late double price;

  @HiveField(3)
  late String imageUrl;

  @HiveField(4)
  late DateTime cachedAt;

  CachedProduct({
    required this.id,
    required this.name,
    required this.price,
    required this.imageUrl,
    DateTime? cachedAt,
  }) : cachedAt = cachedAt ?? DateTime.now();

  bool get isExpired =>
      DateTime.now().difference(cachedAt).inHours > 24;
}

// lib/database/hive/hive_service.dart
import 'package:hive_flutter/hive_flutter.dart';

class HiveService {
  static const _prefsBoxName = 'user_preferences';
  static const _productCacheBoxName = 'product_cache';
  static const _sessionBoxName = 'session';

  static Future<void> initialize() async {
    await Hive.initFlutter();

    // Register adapters
    Hive.registerAdapter(UserPreferencesAdapter());
    Hive.registerAdapter(CachedProductAdapter());

    // Open boxes
    await Hive.openBox<UserPreferences>(_prefsBoxName);
    await Hive.openBox<CachedProduct>(_productCacheBoxName);
    await Hive.openBox<dynamic>(_sessionBoxName);
  }

  // ---- Preferences ----

  Box<UserPreferences> get _prefsBox =>
      Hive.box<UserPreferences>(_prefsBoxName);

  UserPreferences getPreferences() {
    return _prefsBox.get('prefs') ?? UserPreferences();
  }

  Future<void> savePreferences(UserPreferences prefs) async {
    await _prefsBox.put('prefs', prefs);
  }

  Future<void> updateTheme(String theme) async {
    final prefs = getPreferences();
    prefs.theme = theme;
    await prefs.save();
  }

  Future<void> addRecentSearch(String query) async {
    final prefs = getPreferences();
    prefs.recentSearches.remove(query);
    prefs.recentSearches.insert(0, query);
    if (prefs.recentSearches.length > 10) {
      prefs.recentSearches.removeLast();
    }
    await prefs.save();
  }

  // ---- Product cache ----

  Box<CachedProduct> get _cacheBox =>
      Hive.box<CachedProduct>(_productCacheBoxName);

  Future<void> cacheProduct(CachedProduct product) async {
    await _cacheBox.put('product_${product.id}', product);
  }

  CachedProduct? getCachedProduct(int id) {
    final product = _cacheBox.get('product_$id');
    if (product == null || product.isExpired) {
      _cacheBox.delete('product_$id');
      return null;
    }
    return product;
  }

  Future<void> cacheProducts(List<CachedProduct> products) async {
    final entries = {
      for (final p in products) 'product_${p.id}': p,
    };
    await _cacheBox.putAll(entries);
  }

  Future<void> clearExpiredCache() async {
    final expiredKeys = _cacheBox.keys
        .where((k) => _cacheBox.get(k)?.isExpired ?? true)
        .toList();
    await _cacheBox.deleteAll(expiredKeys);
  }

  // ---- Session data ----

  Box<dynamic> get _sessionBox => Hive.box<dynamic>(_sessionBoxName);

  Future<void> setSessionValue(String key, dynamic value) async {
    await _sessionBox.put(key, value);
  }

  T? getSessionValue<T>(String key) {
    return _sessionBox.get(key) as T?;
  }

  Future<void> clearSession() async {
    await _sessionBox.clear();
  }

  // Watch for changes
  Stream<BoxEvent> watchPreferences() => _prefsBox.watch();
}

// Usage example
class HiveDemoScreen extends StatelessWidget {
  final HiveService hiveService;
  const HiveDemoScreen({super.key, required this.hiveService});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Hive Demo')),
      body: ValueListenableBuilder(
        valueListenable: Hive.box<UserPreferences>('user_preferences').listenable(),
        builder: (context, box, _) {
          final prefs = box.get('prefs') ?? UserPreferences();
          return ListView(
            padding: const EdgeInsets.all(16),
            children: [
              ListTile(
                title: const Text('Theme'),
                trailing: DropdownButton<String>(
                  value: prefs.theme,
                  items: ['light', 'dark', 'system']
                      .map((t) => DropdownMenuItem(value: t, child: Text(t)))
                      .toList(),
                  onChanged: (v) {
                    if (v != null) hiveService.updateTheme(v);
                  },
                ),
              ),
              SwitchListTile(
                title: const Text('Notifications'),
                value: prefs.notificationsEnabled,
                onChanged: (v) async {
                  prefs.notificationsEnabled = v;
                  await prefs.save();
                },
              ),
              ListTile(
                title: const Text('Recent Searches'),
                subtitle: Text(prefs.recentSearches.join(', ')),
              ),
            ],
          );
        },
      ),
    );
  }
}
```

## ขั้นตอนที่ 3403: Database Migration Strategies

```dart
// lib/database/migration/migration_manager.dart
import 'package:drift/drift.dart';

/// VersionedMigration wraps each migration step
class VersionedMigration {
  final int version;
  final String description;
  final Future<void> Function(Migrator m, AppDatabase db) migrate;

  const VersionedMigration({
    required this.version,
    required this.description,
    required this.migrate,
  });
}

class MigrationManager {
  static final List<VersionedMigration> migrations = [
    VersionedMigration(
      version: 2,
      description: 'Add updated_at to products',
      migrate: (m, db) async {
        await m.addColumn(db.products, db.products.updatedAt);
      },
    ),
    VersionedMigration(
      version: 3,
      description: 'Add full-text search index',
      migrate: (m, db) async {
        await db.customStatement(
          'CREATE VIRTUAL TABLE IF NOT EXISTS products_fts '
          'USING fts5(name, content=products, content_rowid=id)',
        );
      },
    ),
    VersionedMigration(
      version: 4,
      description: 'Add archived flag to orders',
      migrate: (m, db) async {
        await m.addColumn(db.orders, db.orders.status);
        await db.customStatement(
          "UPDATE orders SET status = 'completed' WHERE status = 'done'",
        );
      },
    ),
  ];

  static MigrationStrategy buildStrategy(AppDatabase db) {
    return MigrationStrategy(
      onCreate: (m) => m.createAll(),
      onUpgrade: (m, from, to) async {
        for (final migration in migrations) {
          if (migration.version > from && migration.version <= to) {
            try {
              await migration.migrate(m, db);
            } catch (e) {
              throw MigrationException(
                'Migration v${migration.version} failed: ${migration.description}\n$e',
              );
            }
          }
        }
      },
      beforeOpen: (details) async {
        await db.customStatement('PRAGMA foreign_keys = ON');
        await db.customStatement('PRAGMA journal_mode = WAL');
        await db.customStatement('PRAGMA synchronous = NORMAL');
      },
    );
  }
}

class MigrationException implements Exception {
  final String message;
  MigrationException(this.message);

  @override
  String toString() => 'MigrationException: $message';
}
```

## ขั้นตอนที่ 3404: Encrypted Database with SQLCipher

```dart
// lib/database/encrypted/encrypted_database.dart
import 'dart:io';
import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:path_provider/path_provider.dart';
import 'package:path/path.dart' as p;
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:sqlcipher_flutter_libs/sqlcipher_flutter_libs.dart';
import 'package:sqlite3/open.dart';

/// Secure key management
class DatabaseKeyManager {
  static const _keyStorageKey = 'db_encryption_key';
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
    iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock),
  );

  static Future<String> getOrCreateKey() async {
    var key = await _storage.read(key: _keyStorageKey);
    if (key == null) {
      key = _generateSecureKey();
      await _storage.write(key: _keyStorageKey, value: key);
    }
    return key;
  }

  static String _generateSecureKey() {
    // In production, use a cryptographically secure random generator
    final random = DateTime.now().millisecondsSinceEpoch;
    return random.toRadixString(16).padLeft(64, '0');
  }

  static Future<void> rotateKey(String newKey) async {
    await _storage.write(key: _keyStorageKey, value: newKey);
  }
}

/// Opens an encrypted SQLite database using SQLCipher
LazyDatabase openEncryptedConnection() {
  return LazyDatabase(() async {
    // Load SQLCipher instead of standard SQLite
    open.overrideForAll(applyWorkaroundToOpenSqlcipher);

    final dbFolder = await getApplicationDocumentsDirectory();
    final file = File(p.join(dbFolder.path, 'secure_app.db'));
    final encryptionKey = await DatabaseKeyManager.getOrCreateKey();

    return NativeDatabase.createInBackground(
      file,
      setup: (database) {
        // Set encryption key via PRAGMA
        database.execute("PRAGMA key = '$encryptionKey'");
        database.execute('PRAGMA cipher_page_size = 4096');
        database.execute('PRAGMA kdf_iter = 256000');
        database.execute("PRAGMA cipher_hmac_algorithm = HMAC_SHA512");
        database.execute("PRAGMA cipher_kdf_algorithm = PBKDF2_HMAC_SHA512");
      },
    );
  });
}

/// Sensitive data models
class SensitiveNote extends Insertable<SensitiveNote> {
  final int id;
  final String title;
  final String content;
  final DateTime createdAt;

  const SensitiveNote({
    required this.id,
    required this.title,
    required this.content,
    required this.createdAt,
  });

  @override
  Map<String, Expression> toColumns(bool nullToAbsent) => {
        'id': Variable(id),
        'title': Variable(title),
        'content': Variable(content),
        'created_at': Variable(createdAt),
      };
}

// Utility: check if encryption is working
Future<bool> verifyEncryption(String dbPath) async {
  try {
    final file = File(dbPath);
    if (!file.existsSync()) return false;

    final bytes = await file.readAsBytes();
    // Encrypted SQLite files start with "SQLite format 3\000"
    // but the first bytes are encrypted with SQLCipher
    // A properly encrypted file won't have the SQLite magic bytes
    final magicBytes = [0x53, 0x51, 0x4C, 0x69, 0x74, 0x65]; // "SQLite"
    for (var i = 0; i < magicBytes.length; i++) {
      if (bytes[i] == magicBytes[i]) return false; // Not encrypted!
    }
    return true;
  } catch (_) {
    return false;
  }
}
```

## ขั้นตอนที่ 3405: Database Repository Pattern

```dart
// lib/database/repositories/product_repository.dart
import 'package:flutter/foundation.dart';

abstract class ProductRepository {
  Future<List<Product>> getAll();
  Future<Product?> getById(int id);
  Future<List<Product>> search(String query);
  Future<int> insert(ProductsCompanion product);
  Future<void> update(int id, ProductsCompanion data);
  Future<void> delete(int id);
  Stream<List<Product>> watchAll();
  Future<void> batchInsert(List<ProductsCompanion> products);
}

class DriftProductRepository implements ProductRepository {
  final AppDatabase _db;
  final HiveService _cache;

  DriftProductRepository({
    required AppDatabase db,
    required HiveService cache,
  })  : _db = db,
        _cache = cache;

  @override
  Future<List<Product>> getAll() => _db.select(_db.products).get();

  @override
  Future<Product?> getById(int id) async {
    // Check cache first
    final cached = _cache.getCachedProduct(id);
    if (cached != null) {
      return Product(
        id: cached.id,
        name: cached.name,
        price: cached.price,
        categoryId: 0,
        stock: 0,
        updatedAt: cached.cachedAt,
      );
    }

    return (_db.select(_db.products)..where((p) => p.id.equals(id)))
        .getSingleOrNull();
  }

  @override
  Future<List<Product>> search(String query) =>
      _db.searchProducts(query);

  @override
  Future<int> insert(ProductsCompanion product) =>
      _db.into(_db.products).insert(product);

  @override
  Future<void> update(int id, ProductsCompanion data) async {
    await (_db.update(_db.products)..where((p) => p.id.equals(id))).write(data);
  }

  @override
  Future<void> delete(int id) async {
    await (_db.delete(_db.products)..where((p) => p.id.equals(id))).go();
  }

  @override
  Stream<List<Product>> watchAll() =>
      (_db.select(_db.products)
            ..orderBy([(p) => OrderingTerm.asc(p.name)]))
          .watch();

  @override
  Future<void> batchInsert(List<ProductsCompanion> products) async {
    await _db.batch((batch) {
      batch.insertAll(_db.products, products, mode: InsertMode.insertOrReplace);
    });
  }

  Future<List<CategorySalesData>> getSalesByCategory() =>
      _db.getSalesByCategory();
}

// lib/database/database_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

final productRepositoryProvider = Provider<DriftProductRepository>((ref) {
  throw UnimplementedError();
});

final allProductsProvider = StreamProvider<List<Product>>((ref) {
  return ref.watch(productRepositoryProvider).watchAll();
});

class DatabaseDemoScreen extends ConsumerWidget {
  const DatabaseDemoScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final productsAsync = ref.watch(allProductsProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Drift Database Demo')),
      body: productsAsync.when(
        data: (products) => products.isEmpty
            ? const Center(child: Text('No products. Add some!'))
            : ListView.builder(
                itemCount: products.length,
                itemBuilder: (context, index) {
                  final product = products[index];
                  return ListTile(
                    title: Text(product.name),
                    subtitle: Text('\$${product.price.toStringAsFixed(2)}'),
                    trailing: Text('Stock: ${product.stock}'),
                  );
                },
              ),
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text('Error: $e')),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          await ref.read(productRepositoryProvider).insert(
                ProductsCompanion.insert(
                  name: 'Sample Product ${DateTime.now().millisecond}',
                  price: 29.99,
                  categoryId: 1,
                ),
              );
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

---

**← [Part 87](part-87-flutter-hooks.md)**
**ต่อไป: [Part 89 →](part-89-payment-integration.md)**
