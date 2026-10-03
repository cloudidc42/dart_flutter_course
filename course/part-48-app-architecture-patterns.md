# Part 48: App Architecture Patterns
## ขั้นตอนที่ 1801-1840

## 🎯 เป้าหมายของ Part นี้
- ออกแบบ Clean Architecture แบ่งเป็น Domain / Data / Presentation layers
- ใช้ Repository Pattern ด้วย abstract base + implementation
- สร้าง UseCases / Interactors สำหรับ business logic
- ใช้ MVVM pattern ด้วย ChangeNotifier
- Dependency Injection ด้วย get_it + injectable

---

## ขั้นตอนที่ 1801: โครงสร้าง Clean Architecture

```
lib/
├── core/
│   ├── error/
│   │   ├── failures.dart
│   │   └── exceptions.dart
│   ├── usecases/
│   │   └── usecase.dart
│   └── utils/
│       └── input_converter.dart
├── features/
│   └── products/
│       ├── domain/
│       │   ├── entities/
│       │   │   └── product.dart
│       │   ├── repositories/
│       │   │   └── product_repository.dart
│       │   └── usecases/
│       │       ├── get_all_products.dart
│       │       ├── get_product_by_id.dart
│       │       └── create_product.dart
│       ├── data/
│       │   ├── models/
│       │   │   └── product_model.dart
│       │   ├── datasources/
│       │   │   ├── product_remote_datasource.dart
│       │   │   └── product_local_datasource.dart
│       │   └── repositories/
│       │       └── product_repository_impl.dart
│       └── presentation/
│           ├── viewmodels/
│           │   └── products_viewmodel.dart
│           ├── screens/
│           │   ├── products_screen.dart
│           │   └── product_detail_screen.dart
│           └── widgets/
│               └── product_card.dart
└── injection_container.dart
```

---

## ขั้นตอนที่ 1802: Core Error Types

```dart
// lib/core/error/failures.dart
abstract class Failure {
  final String message;
  const Failure(this.message);

  @override
  String toString() => '${runtimeType}: $message';
}

class ServerFailure extends Failure {
  final int? statusCode;
  const ServerFailure(super.message, {this.statusCode});
}

class NetworkFailure extends Failure {
  const NetworkFailure([super.message = 'No internet connection']);
}

class CacheFailure extends Failure {
  const CacheFailure([super.message = 'Cache error']);
}

class NotFoundFailure extends Failure {
  const NotFoundFailure([super.message = 'Resource not found']);
}

class ValidationFailure extends Failure {
  final Map<String, String> fieldErrors;
  const ValidationFailure(super.message, {this.fieldErrors = const {}});
}

// lib/core/error/exceptions.dart
class ServerException implements Exception {
  final String message;
  final int? statusCode;
  const ServerException(this.message, {this.statusCode});

  @override
  String toString() => 'ServerException($statusCode): $message';
}

class NetworkException implements Exception {
  final String message;
  const NetworkException([this.message = 'Network unavailable']);

  @override
  String toString() => 'NetworkException: $message';
}

class CacheException implements Exception {
  final String message;
  const CacheException([this.message = 'Cache operation failed']);

  @override
  String toString() => 'CacheException: $message';
}
```

---

## ขั้นตอนที่ 1803: Either Type (Result Pattern)

```dart
// lib/core/result/result.dart
// Simple Result type without external packages

sealed class Result<L, R> {
  const Result();

  bool get isSuccess => this is Success<L, R>;
  bool get isFailure => this is Failure2<L, R>;

  R? get valueOrNull => switch (this) {
        Success(value: final v) => v,
        Failure2() => null,
      };

  L? get failureOrNull => switch (this) {
        Success() => null,
        Failure2(failure: final f) => f,
      };

  T fold<T>({
    required T Function(L failure) onFailure,
    required T Function(R value) onSuccess,
  }) {
    return switch (this) {
      Success(value: final v) => onSuccess(v),
      Failure2(failure: final f) => onFailure(f),
    };
  }

  Result<L, R2> map<R2>(R2 Function(R value) transform) {
    return switch (this) {
      Success(value: final v) => Result.success(transform(v)),
      Failure2(failure: final f) => Result.failure(f),
    };
  }

  factory Result.success(R value) = Success<L, R>;
  factory Result.failure(L failure) = Failure2<L, R>;
}

class Success<L, R> extends Result<L, R> {
  final R value;
  const Success(this.value);
}

class Failure2<L, R> extends Result<L, R> {
  final L failure;
  const Failure2(this.failure);
}

typedef FailureOrResult<T> = Result<Failure, T>;
```

---

## ขั้นตอนที่ 1804: Domain Entities

```dart
// lib/features/products/domain/entities/product.dart
class Product {
  final int id;
  final String name;
  final String description;
  final double price;
  final int stock;
  final String category;
  final String? imageUrl;
  final DateTime createdAt;

  const Product({
    required this.id,
    required this.name,
    required this.description,
    required this.price,
    required this.stock,
    required this.category,
    this.imageUrl,
    required this.createdAt,
  });

  bool get isInStock => stock > 0;
  bool get isLowStock => stock > 0 && stock <= 5;
  bool get isOutOfStock => stock == 0;

  String get priceFormatted {
    if (price >= 1000000) {
      return '฿${(price / 1000000).toStringAsFixed(1)}M';
    } else if (price >= 1000) {
      return '฿${(price / 1000).toStringAsFixed(1)}K';
    }
    return '฿${price.toStringAsFixed(2)}';
  }

  Product copyWith({
    int? id,
    String? name,
    String? description,
    double? price,
    int? stock,
    String? category,
    String? imageUrl,
    DateTime? createdAt,
  }) {
    return Product(
      id: id ?? this.id,
      name: name ?? this.name,
      description: description ?? this.description,
      price: price ?? this.price,
      stock: stock ?? this.stock,
      category: category ?? this.category,
      imageUrl: imageUrl ?? this.imageUrl,
      createdAt: createdAt ?? this.createdAt,
    );
  }

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    return other is Product && other.id == id;
  }

  @override
  int get hashCode => id.hashCode;

  @override
  String toString() => 'Product(id: $id, name: $name, price: $price)';
}
```

---

## ขั้นตอนที่ 1805: Domain Repository Interface

```dart
// lib/features/products/domain/repositories/product_repository.dart
import '../../domain/entities/product.dart';
import '../../../../core/result/result.dart';
import '../../../../core/error/failures.dart';

abstract class ProductRepository {
  Future<FailureOrResult<List<Product>>> getAllProducts();
  Future<FailureOrResult<Product>> getProductById(int id);
  Future<FailureOrResult<List<Product>>> getProductsByCategory(String category);
  Future<FailureOrResult<Product>> createProduct({
    required String name,
    required String description,
    required double price,
    required int stock,
    required String category,
    String? imageUrl,
  });
  Future<FailureOrResult<Product>> updateProduct(Product product);
  Future<FailureOrResult<void>> deleteProduct(int id);
  Future<FailureOrResult<List<Product>>> searchProducts(String query);
}
```

---

## ขั้นตอนที่ 1806: UseCase Base Classes

```dart
// lib/core/usecases/usecase.dart

/// Base class for use cases that take parameters
abstract class UseCase<Type, Params> {
  Future<Type> call(Params params);
}

/// Base class for use cases with no parameters
abstract class UseCaseNoParams<Type> {
  Future<Type> call();
}

/// Marker class for use cases with no parameters
class NoParams {
  const NoParams();
}
```

---

## ขั้นตอนที่ 1807: Domain UseCases

```dart
// lib/features/products/domain/usecases/get_all_products.dart
import '../entities/product.dart';
import '../repositories/product_repository.dart';
import '../../../../core/usecases/usecase.dart';
import '../../../../core/result/result.dart';
import '../../../../core/error/failures.dart';

class GetAllProducts extends UseCaseNoParams<FailureOrResult<List<Product>>> {
  final ProductRepository repository;

  const GetAllProducts({required this.repository});

  @override
  Future<FailureOrResult<List<Product>>> call() async {
    final result = await repository.getAllProducts();
    // Apply business logic: sort by name by default
    return result.map((products) {
      final sorted = List<Product>.from(products)
        ..sort((a, b) => a.name.compareTo(b.name));
      return sorted;
    });
  }
}

// lib/features/products/domain/usecases/get_product_by_id.dart
class GetProductByIdParams {
  final int id;
  const GetProductByIdParams({required this.id});
}

class GetProductById
    extends UseCase<FailureOrResult<Product>, GetProductByIdParams> {
  final ProductRepository repository;

  const GetProductById({required this.repository});

  @override
  Future<FailureOrResult<Product>> call(GetProductByIdParams params) async {
    if (params.id <= 0) {
      return Result.failure(
        const ValidationFailure('Product ID must be positive'),
      );
    }
    return repository.getProductById(params.id);
  }
}

// lib/features/products/domain/usecases/create_product.dart
class CreateProductParams {
  final String name;
  final String description;
  final double price;
  final int stock;
  final String category;
  final String? imageUrl;

  const CreateProductParams({
    required this.name,
    required this.description,
    required this.price,
    required this.stock,
    required this.category,
    this.imageUrl,
  });
}

class CreateProduct
    extends UseCase<FailureOrResult<Product>, CreateProductParams> {
  final ProductRepository repository;

  const CreateProduct({required this.repository});

  @override
  Future<FailureOrResult<Product>> call(CreateProductParams params) async {
    // Validate business rules
    if (params.name.trim().isEmpty) {
      return Result.failure(
        const ValidationFailure('Product name cannot be empty'),
      );
    }
    if (params.price < 0) {
      return Result.failure(
        const ValidationFailure('Price cannot be negative'),
      );
    }
    if (params.stock < 0) {
      return Result.failure(
        const ValidationFailure('Stock cannot be negative'),
      );
    }

    return repository.createProduct(
      name: params.name.trim(),
      description: params.description.trim(),
      price: params.price,
      stock: params.stock,
      category: params.category,
      imageUrl: params.imageUrl,
    );
  }
}
```

---

## ขั้นตอนที่ 1808: Data Layer Models

```dart
// lib/features/products/data/models/product_model.dart
import '../../domain/entities/product.dart';

class ProductModel extends Product {
  const ProductModel({
    required super.id,
    required super.name,
    required super.description,
    required super.price,
    required super.stock,
    required super.category,
    super.imageUrl,
    required super.createdAt,
  });

  factory ProductModel.fromJson(Map<String, dynamic> json) {
    return ProductModel(
      id: json['id'] as int,
      name: json['name'] as String,
      description: json['description'] as String? ?? '',
      price: (json['price'] as num).toDouble(),
      stock: json['stock'] as int? ?? 0,
      category: json['category'] as String? ?? 'Uncategorized',
      imageUrl: json['image_url'] as String?,
      createdAt: json['created_at'] != null
          ? DateTime.parse(json['created_at'] as String)
          : DateTime.now(),
    );
  }

  Map<String, dynamic> toJson() => {
        'id': id,
        'name': name,
        'description': description,
        'price': price,
        'stock': stock,
        'category': category,
        'image_url': imageUrl,
        'created_at': createdAt.toIso8601String(),
      };

  factory ProductModel.fromEntity(Product product) {
    return ProductModel(
      id: product.id,
      name: product.name,
      description: product.description,
      price: product.price,
      stock: product.stock,
      category: product.category,
      imageUrl: product.imageUrl,
      createdAt: product.createdAt,
    );
  }
}
```

---

## ขั้นตอนที่ 1809: Data Sources

```dart
// lib/features/products/data/datasources/product_remote_datasource.dart
import 'dart:convert';
import 'package:http/http.dart' as http;
import '../models/product_model.dart';
import '../../../../core/error/exceptions.dart';

abstract class ProductRemoteDataSource {
  Future<List<ProductModel>> getAllProducts();
  Future<ProductModel> getProductById(int id);
  Future<ProductModel> createProduct(Map<String, dynamic> data);
  Future<ProductModel> updateProduct(int id, Map<String, dynamic> data);
  Future<void> deleteProduct(int id);
}

class ProductRemoteDataSourceImpl implements ProductRemoteDataSource {
  final http.Client client;
  final String baseUrl;

  static const _defaultHeaders = {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  };

  const ProductRemoteDataSourceImpl({
    required this.client,
    this.baseUrl = 'https://api.example.com/v1',
  });

  @override
  Future<List<ProductModel>> getAllProducts() async {
    final response = await client.get(
      Uri.parse('$baseUrl/products'),
      headers: _defaultHeaders,
    );
    _checkResponse(response);
    final List<dynamic> data = json.decode(response.body);
    return data.map((json) => ProductModel.fromJson(json)).toList();
  }

  @override
  Future<ProductModel> getProductById(int id) async {
    final response = await client.get(
      Uri.parse('$baseUrl/products/$id'),
      headers: _defaultHeaders,
    );
    _checkResponse(response);
    return ProductModel.fromJson(json.decode(response.body));
  }

  @override
  Future<ProductModel> createProduct(Map<String, dynamic> data) async {
    final response = await client.post(
      Uri.parse('$baseUrl/products'),
      headers: _defaultHeaders,
      body: json.encode(data),
    );
    _checkResponse(response, expectedStatus: 201);
    return ProductModel.fromJson(json.decode(response.body));
  }

  @override
  Future<ProductModel> updateProduct(int id, Map<String, dynamic> data) async {
    final response = await client.put(
      Uri.parse('$baseUrl/products/$id'),
      headers: _defaultHeaders,
      body: json.encode(data),
    );
    _checkResponse(response);
    return ProductModel.fromJson(json.decode(response.body));
  }

  @override
  Future<void> deleteProduct(int id) async {
    final response = await client.delete(
      Uri.parse('$baseUrl/products/$id'),
      headers: _defaultHeaders,
    );
    _checkResponse(response);
  }

  void _checkResponse(http.Response response, {int expectedStatus = 200}) {
    if (response.statusCode == 404) {
      throw const ServerException('Resource not found', statusCode: 404);
    }
    if (response.statusCode >= 400) {
      throw ServerException(
        'Server error: ${response.body}',
        statusCode: response.statusCode,
      );
    }
    if (response.statusCode != expectedStatus &&
        response.statusCode != 200 &&
        response.statusCode != 201) {
      throw ServerException(
        'Unexpected status: ${response.statusCode}',
        statusCode: response.statusCode,
      );
    }
  }
}
```

---

## ขั้นตอนที่ 1810: Repository Implementation

```dart
// lib/features/products/data/repositories/product_repository_impl.dart
import '../../domain/entities/product.dart';
import '../../domain/repositories/product_repository.dart';
import '../datasources/product_remote_datasource.dart';
import '../datasources/product_local_datasource.dart';
import '../models/product_model.dart';
import '../../../../core/error/exceptions.dart';
import '../../../../core/error/failures.dart';
import '../../../../core/result/result.dart';

class ProductRepositoryImpl implements ProductRepository {
  final ProductRemoteDataSource remoteDataSource;
  final ProductLocalDataSource localDataSource;

  const ProductRepositoryImpl({
    required this.remoteDataSource,
    required this.localDataSource,
  });

  @override
  Future<FailureOrResult<List<Product>>> getAllProducts() async {
    try {
      final products = await remoteDataSource.getAllProducts();
      // Cache locally
      await localDataSource.cacheProducts(products);
      return Result.success(products);
    } on ServerException catch (e) {
      // Try cache on server failure
      try {
        final cached = await localDataSource.getCachedProducts();
        return Result.success(cached);
      } on CacheException {
        return Result.failure(
          ServerFailure(e.message, statusCode: e.statusCode),
        );
      }
    } on NetworkException {
      try {
        final cached = await localDataSource.getCachedProducts();
        return Result.success(cached);
      } on CacheException {
        return const Result.failure(NetworkFailure());
      }
    }
  }

  @override
  Future<FailureOrResult<Product>> getProductById(int id) async {
    try {
      final product = await remoteDataSource.getProductById(id);
      return Result.success(product);
    } on ServerException catch (e) {
      if (e.statusCode == 404) {
        return Result.failure(NotFoundFailure('Product $id not found'));
      }
      return Result.failure(ServerFailure(e.message, statusCode: e.statusCode));
    } on NetworkException {
      return const Result.failure(NetworkFailure());
    }
  }

  @override
  Future<FailureOrResult<Product>> createProduct({
    required String name,
    required String description,
    required double price,
    required int stock,
    required String category,
    String? imageUrl,
  }) async {
    try {
      final product = await remoteDataSource.createProduct({
        'name': name,
        'description': description,
        'price': price,
        'stock': stock,
        'category': category,
        if (imageUrl != null) 'image_url': imageUrl,
      });
      return Result.success(product);
    } on ServerException catch (e) {
      return Result.failure(ServerFailure(e.message, statusCode: e.statusCode));
    }
  }

  @override
  Future<FailureOrResult<Product>> updateProduct(Product product) async {
    try {
      final model = ProductModel.fromEntity(product);
      final updated = await remoteDataSource.updateProduct(product.id, model.toJson());
      return Result.success(updated);
    } on ServerException catch (e) {
      return Result.failure(ServerFailure(e.message, statusCode: e.statusCode));
    }
  }

  @override
  Future<FailureOrResult<void>> deleteProduct(int id) async {
    try {
      await remoteDataSource.deleteProduct(id);
      return const Result.success(null);
    } on ServerException catch (e) {
      return Result.failure(ServerFailure(e.message, statusCode: e.statusCode));
    }
  }

  @override
  Future<FailureOrResult<List<Product>>> getProductsByCategory(
      String category) async {
    final result = await getAllProducts();
    return result.map(
      (products) => products.where((p) => p.category == category).toList(),
    );
  }

  @override
  Future<FailureOrResult<List<Product>>> searchProducts(String query) async {
    final result = await getAllProducts();
    return result.map((products) => products
        .where((p) =>
            p.name.toLowerCase().contains(query.toLowerCase()) ||
            p.description.toLowerCase().contains(query.toLowerCase()))
        .toList());
  }
}

// lib/features/products/data/datasources/product_local_datasource.dart
import 'dart:convert';
import '../models/product_model.dart';
import '../../../../core/error/exceptions.dart';

abstract class ProductLocalDataSource {
  Future<List<ProductModel>> getCachedProducts();
  Future<void> cacheProducts(List<ProductModel> products);
}

class ProductLocalDataSourceImpl implements ProductLocalDataSource {
  // In a real app, use SharedPreferences or Hive
  final Map<String, String> _prefs;

  const ProductLocalDataSourceImpl({required Map<String, String> prefs})
      : _prefs = prefs;

  static const _cacheKey = 'CACHED_PRODUCTS';

  @override
  Future<List<ProductModel>> getCachedProducts() async {
    final jsonStr = _prefs[_cacheKey];
    if (jsonStr == null) {
      throw const CacheException('No cached products found');
    }
    try {
      final List<dynamic> jsonList = json.decode(jsonStr);
      return jsonList.map((j) => ProductModel.fromJson(j)).toList();
    } catch (e) {
      throw CacheException('Failed to parse cache: $e');
    }
  }

  @override
  Future<void> cacheProducts(List<ProductModel> products) async {
    _prefs[_cacheKey] = json.encode(products.map((p) => p.toJson()).toList());
  }
}
```

---

## ขั้นตอนที่ 1811: MVVM ViewModel with ChangeNotifier

```dart
// lib/features/products/presentation/viewmodels/products_viewmodel.dart
import 'package:flutter/foundation.dart';
import '../../domain/entities/product.dart';
import '../../domain/usecases/get_all_products.dart';
import '../../domain/usecases/create_product.dart';
import '../../domain/usecases/get_product_by_id.dart';
import '../../../../core/error/failures.dart';

enum ViewState { initial, loading, success, failure }

class ProductsViewModel extends ChangeNotifier {
  final GetAllProducts getAllProducts;
  final GetProductById getProductById;
  final CreateProduct createProduct;

  ProductsViewModel({
    required this.getAllProducts,
    required this.getProductById,
    required this.createProduct,
  });

  ViewState _state = ViewState.initial;
  List<Product> _products = [];
  String _errorMessage = '';
  String _searchQuery = '';
  String? _selectedCategory;
  bool _isCreating = false;

  ViewState get state => _state;
  List<Product> get products => _filteredProducts;
  String get errorMessage => _errorMessage;
  bool get isLoading => _state == ViewState.loading;
  bool get hasError => _state == ViewState.failure;
  bool get isCreating => _isCreating;

  List<Product> get _filteredProducts {
    var result = _products;

    if (_searchQuery.isNotEmpty) {
      result = result
          .where((p) =>
              p.name.toLowerCase().contains(_searchQuery.toLowerCase()) ||
              p.category.toLowerCase().contains(_searchQuery.toLowerCase()))
          .toList();
    }

    if (_selectedCategory != null) {
      result = result.where((p) => p.category == _selectedCategory).toList();
    }

    return result;
  }

  List<String> get categories {
    final cats = _products.map((p) => p.category).toSet().toList();
    cats.sort();
    return cats;
  }

  Future<void> loadProducts() async {
    _setState(ViewState.loading);

    final result = await getAllProducts();

    result.fold(
      onFailure: (failure) {
        _errorMessage = _mapFailureToMessage(failure);
        _setState(ViewState.failure);
      },
      onSuccess: (products) {
        _products = products;
        _setState(ViewState.success);
      },
    );
  }

  Future<bool> addProduct({
    required String name,
    required String description,
    required double price,
    required int stock,
    required String category,
  }) async {
    _isCreating = true;
    notifyListeners();

    final result = await createProduct(CreateProductParams(
      name: name,
      description: description,
      price: price,
      stock: stock,
      category: category,
    ));

    _isCreating = false;

    return result.fold(
      onFailure: (failure) {
        _errorMessage = _mapFailureToMessage(failure);
        notifyListeners();
        return false;
      },
      onSuccess: (product) {
        _products = [..._products, product];
        notifyListeners();
        return true;
      },
    );
  }

  void setSearchQuery(String query) {
    _searchQuery = query;
    notifyListeners();
  }

  void setCategory(String? category) {
    _selectedCategory = category;
    notifyListeners();
  }

  void clearFilters() {
    _searchQuery = '';
    _selectedCategory = null;
    notifyListeners();
  }

  void _setState(ViewState newState) {
    _state = newState;
    notifyListeners();
  }

  String _mapFailureToMessage(Failure failure) {
    return switch (failure) {
      ServerFailure() => 'Server error: ${failure.message}',
      NetworkFailure() => 'No internet connection. Check your network.',
      CacheFailure() => 'Failed to load cached data.',
      NotFoundFailure() => 'Product not found.',
      ValidationFailure() => failure.message,
      _ => 'An unexpected error occurred.',
    };
  }
}
```

---

## ขั้นตอนที่ 1812: Dependency Injection with get_it

```dart
// pubspec.yaml additions:
# get_it: ^7.6.4
# injectable: ^2.3.2

// lib/injection_container.dart
import 'package:get_it/get_it.dart';
import 'package:http/http.dart' as http;
import 'features/products/data/datasources/product_remote_datasource.dart';
import 'features/products/data/datasources/product_local_datasource.dart';
import 'features/products/data/repositories/product_repository_impl.dart';
import 'features/products/domain/repositories/product_repository.dart';
import 'features/products/domain/usecases/get_all_products.dart';
import 'features/products/domain/usecases/get_product_by_id.dart';
import 'features/products/domain/usecases/create_product.dart';
import 'features/products/presentation/viewmodels/products_viewmodel.dart';

final sl = GetIt.instance; // Service Locator

Future<void> configureDependencies() async {
  // External dependencies
  sl.registerLazySingleton<http.Client>(() => http.Client());

  // Data Sources
  sl.registerLazySingleton<ProductRemoteDataSource>(
    () => ProductRemoteDataSourceImpl(
      client: sl(),
      baseUrl: 'https://api.example.com/v1',
    ),
  );

  sl.registerLazySingleton<ProductLocalDataSource>(
    () => ProductLocalDataSourceImpl(prefs: {}), // Use real SharedPreferences in production
  );

  // Repositories
  sl.registerLazySingleton<ProductRepository>(
    () => ProductRepositoryImpl(
      remoteDataSource: sl(),
      localDataSource: sl(),
    ),
  );

  // Use Cases
  sl.registerLazySingleton(
    () => GetAllProducts(repository: sl()),
  );
  sl.registerLazySingleton(
    () => GetProductById(repository: sl()),
  );
  sl.registerLazySingleton(
    () => CreateProduct(repository: sl()),
  );

  // ViewModels (factory so a new instance is created each time)
  sl.registerFactory(
    () => ProductsViewModel(
      getAllProducts: sl(),
      getProductById: sl(),
      createProduct: sl(),
    ),
  );
}
```

---

## ขั้นตอนที่ 1813: Presentation Screen

```dart
// lib/features/products/presentation/screens/products_screen.dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../viewmodels/products_viewmodel.dart';
import '../widgets/product_card.dart';
import '../../../../injection_container.dart';

class ProductsScreen extends StatefulWidget {
  const ProductsScreen({super.key});

  @override
  State<ProductsScreen> createState() => _ProductsScreenState();
}

class _ProductsScreenState extends State<ProductsScreen> {
  late final ProductsViewModel _viewModel;
  final _searchController = TextEditingController();

  @override
  void initState() {
    super.initState();
    _viewModel = sl<ProductsViewModel>();
    WidgetsBinding.instance.addPostFrameCallback((_) {
      _viewModel.loadProducts();
    });
  }

  @override
  void dispose() {
    _searchController.dispose();
    _viewModel.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider.value(
      value: _viewModel,
      child: Scaffold(
        appBar: AppBar(
          title: const Text('Products'),
          actions: [
            Consumer<ProductsViewModel>(
              builder: (_, vm, __) => vm.categories.isNotEmpty
                  ? PopupMenuButton<String?>(
                      onSelected: vm.setCategory,
                      itemBuilder: (_) => [
                        const PopupMenuItem(value: null, child: Text('All')),
                        ...vm.categories.map(
                          (c) => PopupMenuItem(value: c, child: Text(c)),
                        ),
                      ],
                    )
                  : const SizedBox.shrink(),
            ),
          ],
          bottom: PreferredSize(
            preferredSize: const Size.fromHeight(60),
            child: Padding(
              padding: const EdgeInsets.all(8),
              child: TextField(
                controller: _searchController,
                decoration: InputDecoration(
                  hintText: 'Search products...',
                  prefixIcon: const Icon(Icons.search),
                  filled: true,
                  fillColor: Colors.white,
                  border: OutlineInputBorder(
                    borderRadius: BorderRadius.circular(8),
                    borderSide: BorderSide.none,
                  ),
                ),
                onChanged: _viewModel.setSearchQuery,
              ),
            ),
          ),
        ),
        body: Consumer<ProductsViewModel>(
          builder: (context, vm, _) {
            if (vm.isLoading) {
              return const Center(child: CircularProgressIndicator());
            }

            if (vm.hasError) {
              return Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    const Icon(Icons.error_outline, size: 48, color: Colors.red),
                    const SizedBox(height: 16),
                    Text(vm.errorMessage, textAlign: TextAlign.center),
                    const SizedBox(height: 16),
                    ElevatedButton(
                      onPressed: vm.loadProducts,
                      child: const Text('Retry'),
                    ),
                  ],
                ),
              );
            }

            if (vm.products.isEmpty) {
              return const Center(child: Text('No products found'));
            }

            return RefreshIndicator(
              onRefresh: vm.loadProducts,
              child: ListView.builder(
                padding: const EdgeInsets.all(8),
                itemCount: vm.products.length,
                itemBuilder: (_, i) => ProductCard(product: vm.products[i]),
              ),
            );
          },
        ),
        floatingActionButton: FloatingActionButton(
          onPressed: () => _showAddProductDialog(context),
          child: const Icon(Icons.add),
        ),
      ),
    );
  }

  Future<void> _showAddProductDialog(BuildContext context) async {
    final nameController = TextEditingController();
    final priceController = TextEditingController();

    await showDialog(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('Add Product'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(
              controller: nameController,
              decoration: const InputDecoration(labelText: 'Name'),
            ),
            TextField(
              controller: priceController,
              decoration: const InputDecoration(labelText: 'Price'),
              keyboardType: TextInputType.number,
            ),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(ctx),
            child: const Text('Cancel'),
          ),
          ElevatedButton(
            onPressed: () async {
              Navigator.pop(ctx);
              final success = await _viewModel.addProduct(
                name: nameController.text,
                description: '',
                price: double.tryParse(priceController.text) ?? 0,
                stock: 0,
                category: 'General',
              );
              if (context.mounted) {
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(
                    content: Text(success ? 'Product added!' : _viewModel.errorMessage),
                    backgroundColor: success ? Colors.green : Colors.red,
                  ),
                );
              }
            },
            child: const Text('Add'),
          ),
        ],
      ),
    );
  }
}

// lib/features/products/presentation/widgets/product_card.dart
import 'package:flutter/material.dart';
import '../../domain/entities/product.dart';

class ProductCard extends StatelessWidget {
  final Product product;
  final VoidCallback? onTap;

  const ProductCard({super.key, required this.product, this.onTap});

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 4, vertical: 4),
      child: ListTile(
        leading: CircleAvatar(
          backgroundColor: Theme.of(context).colorScheme.primaryContainer,
          child: Text(product.name[0].toUpperCase()),
        ),
        title: Text(product.name),
        subtitle: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(product.category),
            Row(
              children: [
                Text(
                  product.priceFormatted,
                  style: TextStyle(
                    color: Theme.of(context).colorScheme.primary,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                const SizedBox(width: 8),
                Container(
                  padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
                  decoration: BoxDecoration(
                    color: product.isInStock ? Colors.green : Colors.red,
                    borderRadius: BorderRadius.circular(4),
                  ),
                  child: Text(
                    product.isInStock ? 'In Stock (${product.stock})' : 'Out of Stock',
                    style: const TextStyle(color: Colors.white, fontSize: 10),
                  ),
                ),
              ],
            ),
          ],
        ),
        trailing: const Icon(Icons.arrow_forward_ios, size: 16),
        onTap: onTap,
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1814: Main App Entry Point

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'injection_container.dart';
import 'features/products/presentation/screens/products_screen.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await configureDependencies();
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Clean Architecture Demo',
      theme: ThemeData(
        colorSchemeSeed: Colors.blue,
        useMaterial3: true,
      ),
      home: const ProductsScreen(),
    );
  }
}
```

---

**← [Part 47 - Advanced Extensions & Mixins](part-47-dart-advanced-extensions-mixins.md)**
**ต่อไป: [Part 49 - CI/CD for Flutter →](part-49-ci-cd-flutter.md)**
