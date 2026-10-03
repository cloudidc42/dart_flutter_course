# Part 63: Real-World App — E-Commerce
## ขั้นตอนที่ 2401-2440

## 🎯 เป้าหมายของ Part นี้
- สร้าง e-commerce app ครบวงจร
- Product listing พร้อม filters และ sorting
- Shopping cart พร้อม quantity management
- Checkout flow พร้อม Stripe integration
- Order tracking screen
- Clean Architecture พร้อม Riverpod

---

## ขั้นตอนที่ 2401: pubspec.yaml

```yaml
# pubspec.yaml
name: flutter_ecommerce
description: Full-featured E-Commerce App

environment:
  sdk: ">=3.0.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.4.0
  riverpod_annotation: ^2.3.0
  go_router: ^12.1.1
  dio: ^5.3.3
  cached_network_image: ^3.3.0
  flutter_stripe: ^10.1.1
  equatable: ^2.0.5
  intl: ^0.18.1
  shimmer: ^3.0.0
  badges: ^3.1.2

dev_dependencies:
  flutter_test:
    sdk: flutter
  build_runner: ^2.4.6
  riverpod_generator: ^2.3.0
  flutter_lints: ^3.0.0
```

---

## ขั้นตอนที่ 2402: Domain Models

```dart
// lib/domain/models/product.dart
import 'package:equatable/equatable.dart';

class ProductImage extends Equatable {
  final String id;
  final String url;
  final bool isPrimary;

  const ProductImage({
    required this.id,
    required this.url,
    this.isPrimary = false,
  });

  @override
  List<Object?> get props => [id, url, isPrimary];
}

class ProductVariant extends Equatable {
  final String id;
  final String name;
  final String value;
  final double? priceModifier;
  final int stock;

  const ProductVariant({
    required this.id,
    required this.name,
    required this.value,
    this.priceModifier,
    required this.stock,
  });

  bool get isAvailable => stock > 0;

  @override
  List<Object?> get props => [id, name, value, priceModifier, stock];
}

class Product extends Equatable {
  final String id;
  final String name;
  final String description;
  final double price;
  final double? salePrice;
  final String category;
  final List<String> tags;
  final List<ProductImage> images;
  final List<ProductVariant> variants;
  final double rating;
  final int reviewCount;
  final int stock;
  final bool isFeatured;
  final bool isNew;
  final DateTime createdAt;

  const Product({
    required this.id,
    required this.name,
    required this.description,
    required this.price,
    this.salePrice,
    required this.category,
    this.tags = const [],
    this.images = const [],
    this.variants = const [],
    this.rating = 0.0,
    this.reviewCount = 0,
    required this.stock,
    this.isFeatured = false,
    this.isNew = false,
    required this.createdAt,
  });

  bool get isOnSale => salePrice != null && salePrice! < price;
  double get effectivePrice => salePrice ?? price;
  bool get isAvailable => stock > 0;
  double get discountPercentage =>
      isOnSale ? ((price - salePrice!) / price * 100).roundToDouble() : 0;

  String? get primaryImageUrl {
    final primary = images.where((img) => img.isPrimary).firstOrNull;
    return primary?.url ?? images.firstOrNull?.url;
  }

  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id: json['id'] as String,
      name: json['name'] as String,
      description: json['description'] as String,
      price: (json['price'] as num).toDouble(),
      salePrice: (json['sale_price'] as num?)?.toDouble(),
      category: json['category'] as String,
      tags: List<String>.from(json['tags'] as List? ?? []),
      images: (json['images'] as List? ?? [])
          .map((e) => ProductImage(
                id: e['id'] as String,
                url: e['url'] as String,
                isPrimary: e['is_primary'] as bool? ?? false,
              ))
          .toList(),
      rating: (json['rating'] as num? ?? 0).toDouble(),
      reviewCount: json['review_count'] as int? ?? 0,
      stock: json['stock'] as int? ?? 0,
      isFeatured: json['is_featured'] as bool? ?? false,
      isNew: json['is_new'] as bool? ?? false,
      createdAt: DateTime.parse(json['created_at'] as String),
    );
  }

  @override
  List<Object?> get props => [id, name, price, salePrice, stock];
}
```

```dart
// lib/domain/models/cart.dart
import 'package:equatable/equatable.dart';
import 'product.dart';

class CartItem extends Equatable {
  final String id;
  final Product product;
  final int quantity;
  final ProductVariant? selectedVariant;

  const CartItem({
    required this.id,
    required this.product,
    required this.quantity,
    this.selectedVariant,
  });

  double get unitPrice {
    final base = product.effectivePrice;
    final modifier = selectedVariant?.priceModifier ?? 0;
    return base + modifier;
  }

  double get subtotal => unitPrice * quantity;

  CartItem copyWith({
    int? quantity,
    ProductVariant? selectedVariant,
  }) {
    return CartItem(
      id: id,
      product: product,
      quantity: quantity ?? this.quantity,
      selectedVariant: selectedVariant ?? this.selectedVariant,
    );
  }

  @override
  List<Object?> get props => [id, product.id, quantity, selectedVariant?.id];
}

class Cart extends Equatable {
  final List<CartItem> items;
  final String? couponCode;
  final double discountAmount;

  const Cart({
    this.items = const [],
    this.couponCode,
    this.discountAmount = 0,
  });

  double get subtotal =>
      items.fold(0, (sum, item) => sum + item.subtotal);

  double get tax => subtotal * 0.08; // 8% tax

  double get shipping => subtotal >= 100 ? 0 : 9.99;

  double get total => subtotal + tax + shipping - discountAmount;

  int get totalItems => items.fold(0, (sum, item) => sum + item.quantity);

  bool get isFreeShipping => shipping == 0;

  double get amountToFreeShipping =>
      isFreeShipping ? 0 : 100 - subtotal;

  Cart copyWith({
    List<CartItem>? items,
    String? couponCode,
    double? discountAmount,
  }) {
    return Cart(
      items: items ?? this.items,
      couponCode: couponCode ?? this.couponCode,
      discountAmount: discountAmount ?? this.discountAmount,
    );
  }

  @override
  List<Object?> get props => [items, couponCode, discountAmount];
}
```

```dart
// lib/domain/models/order.dart
import 'package:equatable/equatable.dart';
import 'cart.dart';

enum OrderStatus {
  pending,
  confirmed,
  processing,
  shipped,
  outForDelivery,
  delivered,
  cancelled,
  refunded,
}

class Address extends Equatable {
  final String id;
  final String fullName;
  final String addressLine1;
  final String? addressLine2;
  final String city;
  final String state;
  final String zipCode;
  final String country;
  final String phoneNumber;
  final bool isDefault;

  const Address({
    required this.id,
    required this.fullName,
    required this.addressLine1,
    this.addressLine2,
    required this.city,
    required this.state,
    required this.zipCode,
    required this.country,
    required this.phoneNumber,
    this.isDefault = false,
  });

  String get formatted =>
      '$addressLine1${addressLine2 != null ? ', $addressLine2' : ''}, $city, $state $zipCode, $country';

  @override
  List<Object?> get props => [id, addressLine1, city, country];
}

class OrderItem extends Equatable {
  final String productId;
  final String productName;
  final String? imageUrl;
  final int quantity;
  final double unitPrice;
  final double subtotal;

  const OrderItem({
    required this.productId,
    required this.productName,
    this.imageUrl,
    required this.quantity,
    required this.unitPrice,
    required this.subtotal,
  });

  factory OrderItem.fromCartItem(CartItem cartItem) {
    return OrderItem(
      productId: cartItem.product.id,
      productName: cartItem.product.name,
      imageUrl: cartItem.product.primaryImageUrl,
      quantity: cartItem.quantity,
      unitPrice: cartItem.unitPrice,
      subtotal: cartItem.subtotal,
    );
  }

  @override
  List<Object?> get props => [productId, quantity, unitPrice];
}

class TrackingEvent extends Equatable {
  final String status;
  final String description;
  final String location;
  final DateTime timestamp;

  const TrackingEvent({
    required this.status,
    required this.description,
    required this.location,
    required this.timestamp,
  });

  @override
  List<Object?> get props => [status, timestamp];
}

class Order extends Equatable {
  final String id;
  final String userId;
  final List<OrderItem> items;
  final Address shippingAddress;
  final double subtotal;
  final double tax;
  final double shipping;
  final double discount;
  final double total;
  final OrderStatus status;
  final String? trackingNumber;
  final List<TrackingEvent> trackingEvents;
  final String paymentIntentId;
  final DateTime createdAt;
  final DateTime? deliveredAt;

  const Order({
    required this.id,
    required this.userId,
    required this.items,
    required this.shippingAddress,
    required this.subtotal,
    required this.tax,
    required this.shipping,
    this.discount = 0,
    required this.total,
    required this.status,
    this.trackingNumber,
    this.trackingEvents = const [],
    required this.paymentIntentId,
    required this.createdAt,
    this.deliveredAt,
  });

  String get statusLabel => switch (status) {
        OrderStatus.pending => 'Pending',
        OrderStatus.confirmed => 'Confirmed',
        OrderStatus.processing => 'Processing',
        OrderStatus.shipped => 'Shipped',
        OrderStatus.outForDelivery => 'Out for Delivery',
        OrderStatus.delivered => 'Delivered',
        OrderStatus.cancelled => 'Cancelled',
        OrderStatus.refunded => 'Refunded',
      };

  @override
  List<Object?> get props => [id, status, total];
}
```

---

## ขั้นตอนที่ 2403: Product Repository

```dart
// lib/data/repositories/product_repository.dart
import 'package:dio/dio.dart';
import '../../domain/models/product.dart';

enum SortOption { newest, priceLowToHigh, priceHighToLow, topRated, mostReviewed }

class ProductFilter {
  final String? category;
  final double? minPrice;
  final double? maxPrice;
  final double? minRating;
  final bool? onSaleOnly;
  final bool? inStockOnly;
  final List<String> tags;
  final SortOption sortBy;

  const ProductFilter({
    this.category,
    this.minPrice,
    this.maxPrice,
    this.minRating,
    this.onSaleOnly,
    this.inStockOnly,
    this.tags = const [],
    this.sortBy = SortOption.newest,
  });

  ProductFilter copyWith({
    String? category,
    double? minPrice,
    double? maxPrice,
    double? minRating,
    bool? onSaleOnly,
    bool? inStockOnly,
    List<String>? tags,
    SortOption? sortBy,
  }) {
    return ProductFilter(
      category: category ?? this.category,
      minPrice: minPrice ?? this.minPrice,
      maxPrice: maxPrice ?? this.maxPrice,
      minRating: minRating ?? this.minRating,
      onSaleOnly: onSaleOnly ?? this.onSaleOnly,
      inStockOnly: inStockOnly ?? this.inStockOnly,
      tags: tags ?? this.tags,
      sortBy: sortBy ?? this.sortBy,
    );
  }
}

abstract class ProductRepository {
  Future<List<Product>> getProducts({
    ProductFilter? filter,
    int page,
    int pageSize,
  });
  Future<Product> getProductById(String id);
  Future<List<Product>> getFeaturedProducts();
  Future<List<Product>> searchProducts(String query);
  Future<List<String>> getCategories();
}

class MockProductRepository implements ProductRepository {
  // Generate mock products
  static List<Product> _generateMockProducts() {
    final categories = ['Electronics', 'Clothing', 'Books', 'Sports', 'Home'];
    return List.generate(50, (index) {
      final category = categories[index % categories.length];
      final price = 10.0 + (index * 7.5);
      final isOnSale = index % 3 == 0;
      return Product(
        id: 'prod_${index + 1}',
        name: '$category Item ${index + 1}',
        description:
            'High quality $category product. Perfect for everyday use. This is a detailed description of the product.',
        price: price,
        salePrice: isOnSale ? price * 0.8 : null,
        category: category,
        tags: [category.toLowerCase(), 'trending'],
        images: [
          ProductImage(
            id: 'img_$index',
            url: 'https://picsum.photos/seed/$index/400/400',
            isPrimary: true,
          ),
        ],
        rating: 3.0 + (index % 10) * 0.2,
        reviewCount: index * 3 + 5,
        stock: index % 5 == 0 ? 0 : (index % 20 + 1),
        isFeatured: index % 7 == 0,
        isNew: index < 10,
        createdAt: DateTime.now().subtract(Duration(days: index)),
      );
    });
  }

  final List<Product> _products = _generateMockProducts();

  @override
  Future<List<Product>> getProducts({
    ProductFilter? filter,
    int page = 1,
    int pageSize = 20,
  }) async {
    await Future.delayed(const Duration(milliseconds: 300));

    var filtered = List<Product>.from(_products);

    if (filter != null) {
      if (filter.category != null) {
        filtered = filtered
            .where((p) => p.category == filter.category)
            .toList();
      }
      if (filter.minPrice != null) {
        filtered = filtered
            .where((p) => p.effectivePrice >= filter.minPrice!)
            .toList();
      }
      if (filter.maxPrice != null) {
        filtered = filtered
            .where((p) => p.effectivePrice <= filter.maxPrice!)
            .toList();
      }
      if (filter.minRating != null) {
        filtered = filtered
            .where((p) => p.rating >= filter.minRating!)
            .toList();
      }
      if (filter.onSaleOnly == true) {
        filtered = filtered.where((p) => p.isOnSale).toList();
      }
      if (filter.inStockOnly == true) {
        filtered = filtered.where((p) => p.isAvailable).toList();
      }

      // Sort
      switch (filter.sortBy) {
        case SortOption.newest:
          filtered.sort((a, b) => b.createdAt.compareTo(a.createdAt));
        case SortOption.priceLowToHigh:
          filtered.sort(
              (a, b) => a.effectivePrice.compareTo(b.effectivePrice));
        case SortOption.priceHighToLow:
          filtered.sort(
              (a, b) => b.effectivePrice.compareTo(a.effectivePrice));
        case SortOption.topRated:
          filtered.sort((a, b) => b.rating.compareTo(a.rating));
        case SortOption.mostReviewed:
          filtered.sort((a, b) => b.reviewCount.compareTo(a.reviewCount));
      }
    }

    final start = (page - 1) * pageSize;
    final end = (start + pageSize).clamp(0, filtered.length);
    return filtered.sublist(start, end);
  }

  @override
  Future<Product> getProductById(String id) async {
    await Future.delayed(const Duration(milliseconds: 200));
    return _products.firstWhere(
      (p) => p.id == id,
      orElse: () => throw Exception('Product not found: $id'),
    );
  }

  @override
  Future<List<Product>> getFeaturedProducts() async {
    await Future.delayed(const Duration(milliseconds: 200));
    return _products.where((p) => p.isFeatured).take(10).toList();
  }

  @override
  Future<List<Product>> searchProducts(String query) async {
    await Future.delayed(const Duration(milliseconds: 300));
    final lowerQuery = query.toLowerCase();
    return _products
        .where((p) =>
            p.name.toLowerCase().contains(lowerQuery) ||
            p.description.toLowerCase().contains(lowerQuery) ||
            p.category.toLowerCase().contains(lowerQuery))
        .toList();
  }

  @override
  Future<List<String>> getCategories() async {
    return _products.map((p) => p.category).toSet().toList();
  }
}
```

---

## ขั้นตอนที่ 2404: Riverpod Providers

```dart
// lib/providers/product_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../data/repositories/product_repository.dart';
import '../domain/models/product.dart';

final productRepositoryProvider = Provider<ProductRepository>(
  (ref) => MockProductRepository(),
);

final productFilterProvider =
    StateNotifierProvider<ProductFilterNotifier, ProductFilter>(
  (ref) => ProductFilterNotifier(),
);

class ProductFilterNotifier extends StateNotifier<ProductFilter> {
  ProductFilterNotifier() : super(const ProductFilter());

  void setCategory(String? category) =>
      state = state.copyWith(category: category);

  void setPriceRange(double? min, double? max) =>
      state = state.copyWith(minPrice: min, maxPrice: max);

  void setSortOption(SortOption sort) =>
      state = state.copyWith(sortBy: sort);

  void setOnSaleOnly(bool? value) =>
      state = state.copyWith(onSaleOnly: value);

  void setInStockOnly(bool? value) =>
      state = state.copyWith(inStockOnly: value);

  void reset() => state = const ProductFilter();
}

final productsProvider =
    FutureProvider.autoDispose<List<Product>>((ref) async {
  final repository = ref.watch(productRepositoryProvider);
  final filter = ref.watch(productFilterProvider);
  return repository.getProducts(filter: filter);
});

final featuredProductsProvider =
    FutureProvider.autoDispose<List<Product>>((ref) async {
  final repository = ref.watch(productRepositoryProvider);
  return repository.getFeaturedProducts();
});

final productDetailProvider =
    FutureProvider.autoDispose.family<Product, String>((ref, id) async {
  final repository = ref.watch(productRepositoryProvider);
  return repository.getProductById(id);
});

final searchQueryProvider = StateProvider<String>((ref) => '');

final searchResultsProvider =
    FutureProvider.autoDispose<List<Product>>((ref) async {
  final query = ref.watch(searchQueryProvider);
  if (query.isEmpty) return [];
  final repository = ref.watch(productRepositoryProvider);
  return repository.searchProducts(query);
});

final categoriesProvider = FutureProvider<List<String>>((ref) async {
  final repository = ref.watch(productRepositoryProvider);
  return repository.getCategories();
});
```

```dart
// lib/providers/cart_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../domain/models/cart.dart';
import '../domain/models/product.dart';

class CartNotifier extends StateNotifier<Cart> {
  CartNotifier() : super(const Cart());

  void addProduct(Product product, {int quantity = 1, ProductVariant? variant}) {
    final existingIndex = state.items.indexWhere(
      (item) =>
          item.product.id == product.id &&
          item.selectedVariant?.id == variant?.id,
    );

    if (existingIndex >= 0) {
      final updatedItems = List<CartItem>.from(state.items);
      final existingItem = updatedItems[existingIndex];
      updatedItems[existingIndex] =
          existingItem.copyWith(quantity: existingItem.quantity + quantity);
      state = state.copyWith(items: updatedItems);
    } else {
      final newItem = CartItem(
        id: '${product.id}_${variant?.id ?? "default"}',
        product: product,
        quantity: quantity,
        selectedVariant: variant,
      );
      state = state.copyWith(items: [...state.items, newItem]);
    }
  }

  void removeItem(String itemId) {
    state = state.copyWith(
      items: state.items.where((item) => item.id != itemId).toList(),
    );
  }

  void updateQuantity(String itemId, int quantity) {
    if (quantity <= 0) {
      removeItem(itemId);
      return;
    }
    state = state.copyWith(
      items: state.items.map((item) {
        return item.id == itemId ? item.copyWith(quantity: quantity) : item;
      }).toList(),
    );
  }

  void applyCoupon(String code, double discount) {
    state = state.copyWith(couponCode: code, discountAmount: discount);
  }

  void removeCoupon() {
    state = state.copyWith(couponCode: null, discountAmount: 0);
  }

  void clearCart() {
    state = const Cart();
  }
}

final cartProvider = StateNotifierProvider<CartNotifier, Cart>(
  (ref) => CartNotifier(),
);

final cartItemCountProvider = Provider<int>((ref) {
  return ref.watch(cartProvider).totalItems;
});
```

---

## ขั้นตอนที่ 2405: Product Listing Screen

```dart
// lib/screens/product_listing_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:cached_network_image/cached_network_image.dart';
import 'package:intl/intl.dart';
import '../domain/models/product.dart';
import '../providers/product_providers.dart';
import '../providers/cart_providers.dart';

class ProductListingScreen extends ConsumerStatefulWidget {
  const ProductListingScreen({super.key});

  @override
  ConsumerState<ProductListingScreen> createState() =>
      _ProductListingScreenState();
}

class _ProductListingScreenState
    extends ConsumerState<ProductListingScreen> {
  bool _isGridView = true;

  @override
  Widget build(BuildContext context) {
    final productsAsync = ref.watch(productsProvider);
    final filter = ref.watch(productFilterProvider);
    final cartCount = ref.watch(cartItemCountProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Products'),
        actions: [
          Stack(
            children: [
              IconButton(
                icon: const Icon(Icons.shopping_cart),
                onPressed: () => Navigator.pushNamed(context, '/cart'),
              ),
              if (cartCount > 0)
                Positioned(
                  right: 8,
                  top: 8,
                  child: Container(
                    padding: const EdgeInsets.all(2),
                    decoration: BoxDecoration(
                      color: Colors.red,
                      borderRadius: BorderRadius.circular(10),
                    ),
                    constraints: const BoxConstraints(minWidth: 16, minHeight: 16),
                    child: Text(
                      '$cartCount',
                      style: const TextStyle(color: Colors.white, fontSize: 10),
                      textAlign: TextAlign.center,
                    ),
                  ),
                ),
            ],
          ),
          IconButton(
            icon: Icon(_isGridView ? Icons.list : Icons.grid_view),
            onPressed: () => setState(() => _isGridView = !_isGridView),
          ),
          IconButton(
            icon: const Icon(Icons.filter_list),
            onPressed: () => _showFilterSheet(context),
          ),
        ],
      ),
      body: Column(
        children: [
          _buildSearchBar(),
          _buildSortBar(filter),
          Expanded(
            child: productsAsync.when(
              loading: () => _buildLoadingGrid(),
              error: (e, _) => Center(child: Text('Error: $e')),
              data: (products) => products.isEmpty
                  ? const Center(child: Text('No products found'))
                  : _isGridView
                      ? _buildProductGrid(products)
                      : _buildProductList(products),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildSearchBar() {
    return Padding(
      padding: const EdgeInsets.all(8.0),
      child: TextField(
        decoration: InputDecoration(
          hintText: 'Search products...',
          prefixIcon: const Icon(Icons.search),
          border: OutlineInputBorder(
            borderRadius: BorderRadius.circular(12),
          ),
          contentPadding: const EdgeInsets.symmetric(horizontal: 16),
        ),
        onChanged: (value) =>
            ref.read(searchQueryProvider.notifier).state = value,
      ),
    );
  }

  Widget _buildSortBar(ProductFilter filter) {
    return SizedBox(
      height: 40,
      child: ListView(
        scrollDirection: Axis.horizontal,
        padding: const EdgeInsets.symmetric(horizontal: 8),
        children: SortOption.values.map((option) {
          final label = switch (option) {
            SortOption.newest => 'Newest',
            SortOption.priceLowToHigh => 'Price ↑',
            SortOption.priceHighToLow => 'Price ↓',
            SortOption.topRated => 'Top Rated',
            SortOption.mostReviewed => 'Most Reviewed',
          };
          return Padding(
            padding: const EdgeInsets.only(right: 8),
            child: ChoiceChip(
              label: Text(label),
              selected: filter.sortBy == option,
              onSelected: (_) => ref
                  .read(productFilterProvider.notifier)
                  .setSortOption(option),
            ),
          );
        }).toList(),
      ),
    );
  }

  Widget _buildProductGrid(List<Product> products) {
    return GridView.builder(
      padding: const EdgeInsets.all(8),
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 2,
        childAspectRatio: 0.72,
        crossAxisSpacing: 8,
        mainAxisSpacing: 8,
      ),
      itemCount: products.length,
      itemBuilder: (context, index) => ProductCard(product: products[index]),
    );
  }

  Widget _buildProductList(List<Product> products) {
    return ListView.builder(
      padding: const EdgeInsets.all(8),
      itemCount: products.length,
      itemBuilder: (context, index) =>
          ProductListTile(product: products[index]),
    );
  }

  Widget _buildLoadingGrid() {
    return GridView.builder(
      padding: const EdgeInsets.all(8),
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 2,
        childAspectRatio: 0.72,
        crossAxisSpacing: 8,
        mainAxisSpacing: 8,
      ),
      itemCount: 6,
      itemBuilder: (_, __) => const _ShimmerCard(),
    );
  }

  void _showFilterSheet(BuildContext context) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      shape: const RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
      ),
      builder: (_) => const ProductFilterSheet(),
    );
  }
}

class ProductCard extends ConsumerWidget {
  final Product product;

  const ProductCard({super.key, required this.product});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final formatter = NumberFormat.currency(symbol: '\$');

    return Card(
      clipBehavior: Clip.antiAlias,
      child: InkWell(
        onTap: () => Navigator.pushNamed(
          context,
          '/product/${product.id}',
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Expanded(
              flex: 3,
              child: Stack(
                fit: StackFit.expand,
                children: [
                  CachedNetworkImage(
                    imageUrl: product.primaryImageUrl ??
                        'https://picsum.photos/400/400',
                    fit: BoxFit.cover,
                    placeholder: (_, __) =>
                        Container(color: Colors.grey.shade200),
                    errorWidget: (_, __, ___) =>
                        const Icon(Icons.image_not_supported),
                  ),
                  if (product.isOnSale)
                    Positioned(
                      top: 8,
                      left: 8,
                      child: Container(
                        padding: const EdgeInsets.symmetric(
                          horizontal: 8,
                          vertical: 4,
                        ),
                        decoration: BoxDecoration(
                          color: Colors.red,
                          borderRadius: BorderRadius.circular(4),
                        ),
                        child: Text(
                          '-${product.discountPercentage.toInt()}%',
                          style: const TextStyle(
                            color: Colors.white,
                            fontSize: 12,
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                      ),
                    ),
                  if (!product.isAvailable)
                    Container(
                      color: Colors.black54,
                      child: const Center(
                        child: Text(
                          'Out of Stock',
                          style: TextStyle(color: Colors.white),
                        ),
                      ),
                    ),
                ],
              ),
            ),
            Expanded(
              flex: 2,
              child: Padding(
                padding: const EdgeInsets.all(8),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      product.name,
                      maxLines: 2,
                      overflow: TextOverflow.ellipsis,
                      style: const TextStyle(fontWeight: FontWeight.w500),
                    ),
                    const Spacer(),
                    Row(
                      children: [
                        const Icon(Icons.star, size: 14, color: Colors.amber),
                        Text(
                          ' ${product.rating.toStringAsFixed(1)}',
                          style: const TextStyle(fontSize: 12),
                        ),
                      ],
                    ),
                    Row(
                      mainAxisAlignment: MainAxisAlignment.spaceBetween,
                      children: [
                        Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            if (product.isOnSale)
                              Text(
                                formatter.format(product.price),
                                style: const TextStyle(
                                  decoration: TextDecoration.lineThrough,
                                  color: Colors.grey,
                                  fontSize: 12,
                                ),
                              ),
                            Text(
                              formatter.format(product.effectivePrice),
                              style: TextStyle(
                                fontWeight: FontWeight.bold,
                                color: product.isOnSale
                                    ? Colors.red
                                    : Colors.black87,
                              ),
                            ),
                          ],
                        ),
                        IconButton(
                          icon: const Icon(Icons.add_shopping_cart),
                          onPressed: product.isAvailable
                              ? () {
                                  ref
                                      .read(cartProvider.notifier)
                                      .addProduct(product);
                                  ScaffoldMessenger.of(context).showSnackBar(
                                    SnackBar(
                                      content: Text(
                                          '${product.name} added to cart'),
                                      duration: const Duration(seconds: 2),
                                    ),
                                  );
                                }
                              : null,
                          iconSize: 20,
                        ),
                      ],
                    ),
                  ],
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class ProductListTile extends ConsumerWidget {
  final Product product;

  const ProductListTile({super.key, required this.product});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final formatter = NumberFormat.currency(symbol: '\$');
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: ListTile(
        leading: ClipRRect(
          borderRadius: BorderRadius.circular(8),
          child: CachedNetworkImage(
            imageUrl:
                product.primaryImageUrl ?? 'https://picsum.photos/80/80',
            width: 60,
            height: 60,
            fit: BoxFit.cover,
          ),
        ),
        title: Text(product.name),
        subtitle: Text(
          formatter.format(product.effectivePrice),
          style: const TextStyle(fontWeight: FontWeight.bold),
        ),
        trailing: IconButton(
          icon: const Icon(Icons.add_shopping_cart),
          onPressed: () =>
              ref.read(cartProvider.notifier).addProduct(product),
        ),
        onTap: () =>
            Navigator.pushNamed(context, '/product/${product.id}'),
      ),
    );
  }
}

class _ShimmerCard extends StatelessWidget {
  const _ShimmerCard();

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        children: [
          Expanded(
            flex: 3,
            child: Container(color: Colors.grey.shade300),
          ),
          Expanded(
            flex: 2,
            child: Padding(
              padding: const EdgeInsets.all(8),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Container(
                    height: 14,
                    width: double.infinity,
                    color: Colors.grey.shade300,
                  ),
                  const SizedBox(height: 8),
                  Container(
                    height: 14,
                    width: 80,
                    color: Colors.grey.shade300,
                  ),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }
}

class ProductFilterSheet extends ConsumerWidget {
  const ProductFilterSheet({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final filter = ref.watch(productFilterProvider);
    final categoriesAsync = ref.watch(categoriesProvider);

    return DraggableScrollableSheet(
      initialChildSize: 0.6,
      maxChildSize: 0.9,
      minChildSize: 0.4,
      expand: false,
      builder: (_, controller) {
        return Column(
          children: [
            Container(
              width: 40,
              height: 4,
              margin: const EdgeInsets.symmetric(vertical: 12),
              decoration: BoxDecoration(
                color: Colors.grey.shade400,
                borderRadius: BorderRadius.circular(2),
              ),
            ),
            Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16),
              child: Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  const Text(
                    'Filters',
                    style: TextStyle(
                        fontSize: 18, fontWeight: FontWeight.bold),
                  ),
                  TextButton(
                    onPressed: () {
                      ref.read(productFilterProvider.notifier).reset();
                      Navigator.pop(context);
                    },
                    child: const Text('Reset'),
                  ),
                ],
              ),
            ),
            Expanded(
              child: ListView(
                controller: controller,
                padding: const EdgeInsets.symmetric(horizontal: 16),
                children: [
                  const Text('Category',
                      style: TextStyle(fontWeight: FontWeight.bold)),
                  const SizedBox(height: 8),
                  categoriesAsync.when(
                    loading: () => const CircularProgressIndicator(),
                    error: (_, __) => const Text('Failed to load categories'),
                    data: (categories) => Wrap(
                      spacing: 8,
                      children: [
                        ChoiceChip(
                          label: const Text('All'),
                          selected: filter.category == null,
                          onSelected: (_) => ref
                              .read(productFilterProvider.notifier)
                              .setCategory(null),
                        ),
                        ...categories.map(
                          (cat) => ChoiceChip(
                            label: Text(cat),
                            selected: filter.category == cat,
                            onSelected: (_) => ref
                                .read(productFilterProvider.notifier)
                                .setCategory(cat),
                          ),
                        ),
                      ],
                    ),
                  ),
                  const SizedBox(height: 16),
                  const Text('Options',
                      style: TextStyle(fontWeight: FontWeight.bold)),
                  SwitchListTile(
                    title: const Text('On Sale Only'),
                    value: filter.onSaleOnly ?? false,
                    onChanged: (v) => ref
                        .read(productFilterProvider.notifier)
                        .setOnSaleOnly(v),
                  ),
                  SwitchListTile(
                    title: const Text('In Stock Only'),
                    value: filter.inStockOnly ?? false,
                    onChanged: (v) => ref
                        .read(productFilterProvider.notifier)
                        .setInStockOnly(v),
                  ),
                  const SizedBox(height: 16),
                  ElevatedButton(
                    onPressed: () => Navigator.pop(context),
                    child: const Text('Apply Filters'),
                  ),
                  const SizedBox(height: 16),
                ],
              ),
            ),
          ],
        );
      },
    );
  }
}
```

---

## ขั้นตอนที่ 2406: Cart Screen

```dart
// lib/screens/cart_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:cached_network_image/cached_network_image.dart';
import 'package:intl/intl.dart';
import '../domain/models/cart.dart';
import '../providers/cart_providers.dart';

class CartScreen extends ConsumerWidget {
  const CartScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cart = ref.watch(cartProvider);
    final formatter = NumberFormat.currency(symbol: '\$');

    if (cart.items.isEmpty) {
      return Scaffold(
        appBar: AppBar(title: const Text('My Cart')),
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(Icons.shopping_cart_outlined, size: 80, color: Colors.grey),
              const SizedBox(height: 16),
              const Text('Your cart is empty'),
              const SizedBox(height: 16),
              ElevatedButton(
                onPressed: () => Navigator.pop(context),
                child: const Text('Continue Shopping'),
              ),
            ],
          ),
        ),
      );
    }

    return Scaffold(
      appBar: AppBar(
        title: Text('Cart (${cart.totalItems})'),
        actions: [
          TextButton(
            onPressed: () => ref.read(cartProvider.notifier).clearCart(),
            child: const Text('Clear'),
          ),
        ],
      ),
      body: Column(
        children: [
          if (!cart.isFreeShipping)
            Container(
              padding: const EdgeInsets.all(12),
              color: Colors.blue.shade50,
              child: Row(
                children: [
                  const Icon(Icons.local_shipping, color: Colors.blue),
                  const SizedBox(width: 8),
                  Text(
                    'Add ${formatter.format(cart.amountToFreeShipping)} more for FREE shipping!',
                    style: const TextStyle(color: Colors.blue),
                  ),
                ],
              ),
            ),
          Expanded(
            child: ListView.builder(
              padding: const EdgeInsets.all(8),
              itemCount: cart.items.length,
              itemBuilder: (context, index) =>
                  CartItemTile(item: cart.items[index]),
            ),
          ),
          _buildOrderSummary(context, cart, formatter),
        ],
      ),
    );
  }

  Widget _buildOrderSummary(
      BuildContext context, Cart cart, NumberFormat formatter) {
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: Colors.white,
        boxShadow: [
          BoxShadow(
            color: Colors.grey.shade200,
            blurRadius: 10,
            offset: const Offset(0, -5),
          ),
        ],
      ),
      child: Column(
        children: [
          _summaryRow('Subtotal', formatter.format(cart.subtotal)),
          _summaryRow(
            'Shipping',
            cart.isFreeShipping ? 'FREE' : formatter.format(cart.shipping),
          ),
          _summaryRow('Tax (8%)', formatter.format(cart.tax)),
          if (cart.discountAmount > 0)
            _summaryRow(
              'Discount',
              '-${formatter.format(cart.discountAmount)}',
              color: Colors.green,
            ),
          const Divider(),
          _summaryRow(
            'Total',
            formatter.format(cart.total),
            isBold: true,
          ),
          const SizedBox(height: 12),
          SizedBox(
            width: double.infinity,
            child: ElevatedButton(
              onPressed: () =>
                  Navigator.pushNamed(context, '/checkout'),
              style: ElevatedButton.styleFrom(
                padding: const EdgeInsets.symmetric(vertical: 16),
              ),
              child: Text(
                'Checkout • ${formatter.format(cart.total)}',
                style: const TextStyle(fontSize: 16),
              ),
            ),
          ),
        ],
      ),
    );
  }

  Widget _summaryRow(String label, String value, {bool isBold = false, Color? color}) {
    final style = TextStyle(
      fontWeight: isBold ? FontWeight.bold : FontWeight.normal,
      fontSize: isBold ? 16 : 14,
      color: color,
    );
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(label, style: style),
          Text(value, style: style),
        ],
      ),
    );
  }
}

class CartItemTile extends ConsumerWidget {
  final CartItem item;

  const CartItemTile({super.key, required this.item});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final formatter = NumberFormat.currency(symbol: '\$');

    return Dismissible(
      key: Key(item.id),
      direction: DismissDirection.endToStart,
      background: Container(
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 16),
        color: Colors.red,
        child: const Icon(Icons.delete, color: Colors.white),
      ),
      onDismissed: (_) =>
          ref.read(cartProvider.notifier).removeItem(item.id),
      child: Card(
        margin: const EdgeInsets.only(bottom: 8),
        child: Padding(
          padding: const EdgeInsets.all(12),
          child: Row(
            children: [
              ClipRRect(
                borderRadius: BorderRadius.circular(8),
                child: CachedNetworkImage(
                  imageUrl: item.product.primaryImageUrl ??
                      'https://picsum.photos/80/80',
                  width: 72,
                  height: 72,
                  fit: BoxFit.cover,
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      item.product.name,
                      style: const TextStyle(fontWeight: FontWeight.w500),
                    ),
                    if (item.selectedVariant != null)
                      Text(
                        '${item.selectedVariant!.name}: ${item.selectedVariant!.value}',
                        style: TextStyle(
                          color: Colors.grey.shade600,
                          fontSize: 12,
                        ),
                      ),
                    Text(
                      formatter.format(item.unitPrice),
                      style: const TextStyle(
                        color: Colors.blue,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                  ],
                ),
              ),
              Column(
                children: [
                  Text(
                    formatter.format(item.subtotal),
                    style: const TextStyle(fontWeight: FontWeight.bold),
                  ),
                  const SizedBox(height: 8),
                  Row(
                    children: [
                      _quantityButton(
                        icon: Icons.remove,
                        onTap: () => ref
                            .read(cartProvider.notifier)
                            .updateQuantity(item.id, item.quantity - 1),
                      ),
                      Padding(
                        padding: const EdgeInsets.symmetric(horizontal: 8),
                        child: Text(
                          '${item.quantity}',
                          style: const TextStyle(
                            fontSize: 16,
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                      ),
                      _quantityButton(
                        icon: Icons.add,
                        onTap: () => ref
                            .read(cartProvider.notifier)
                            .updateQuantity(item.id, item.quantity + 1),
                      ),
                    ],
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }

  Widget _quantityButton({required IconData icon, required VoidCallback onTap}) {
    return GestureDetector(
      onTap: onTap,
      child: Container(
        width: 28,
        height: 28,
        decoration: BoxDecoration(
          border: Border.all(color: Colors.grey.shade400),
          borderRadius: BorderRadius.circular(6),
        ),
        child: Icon(icon, size: 16),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2407: Checkout & Order Tracking

```dart
// lib/screens/checkout_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:intl/intl.dart';
import '../domain/models/order.dart';
import '../providers/cart_providers.dart';

class CheckoutScreen extends ConsumerStatefulWidget {
  const CheckoutScreen({super.key});

  @override
  ConsumerState<CheckoutScreen> createState() => _CheckoutScreenState();
}

class _CheckoutScreenState extends ConsumerState<CheckoutScreen> {
  int _currentStep = 0;
  final _formKey = GlobalKey<FormState>();
  bool _isProcessing = false;

  // Form fields
  final _nameController = TextEditingController();
  final _addressController = TextEditingController();
  final _cityController = TextEditingController();
  final _zipController = TextEditingController();
  final _phoneController = TextEditingController();

  String _selectedPaymentMethod = 'card';

  @override
  void dispose() {
    _nameController.dispose();
    _addressController.dispose();
    _cityController.dispose();
    _zipController.dispose();
    _phoneController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final cart = ref.watch(cartProvider);
    final formatter = NumberFormat.currency(symbol: '\$');

    return Scaffold(
      appBar: AppBar(title: const Text('Checkout')),
      body: Stepper(
        currentStep: _currentStep,
        onStepContinue: _onStepContinue,
        onStepCancel: _currentStep > 0
            ? () => setState(() => _currentStep--)
            : null,
        steps: [
          Step(
            title: const Text('Shipping Address'),
            isActive: _currentStep >= 0,
            state: _currentStep > 0
                ? StepState.complete
                : StepState.indexed,
            content: _buildAddressForm(),
          ),
          Step(
            title: const Text('Payment'),
            isActive: _currentStep >= 1,
            state: _currentStep > 1
                ? StepState.complete
                : StepState.indexed,
            content: _buildPaymentForm(),
          ),
          Step(
            title: const Text('Review Order'),
            isActive: _currentStep >= 2,
            content: _buildOrderReview(cart, formatter),
          ),
        ],
      ),
    );
  }

  Widget _buildAddressForm() {
    return Form(
      key: _formKey,
      child: Column(
        children: [
          TextFormField(
            controller: _nameController,
            decoration: const InputDecoration(labelText: 'Full Name'),
            validator: (v) =>
                v?.isEmpty == true ? 'Name is required' : null,
          ),
          const SizedBox(height: 12),
          TextFormField(
            controller: _addressController,
            decoration: const InputDecoration(labelText: 'Address Line 1'),
            validator: (v) =>
                v?.isEmpty == true ? 'Address is required' : null,
          ),
          const SizedBox(height: 12),
          Row(
            children: [
              Expanded(
                child: TextFormField(
                  controller: _cityController,
                  decoration: const InputDecoration(labelText: 'City'),
                  validator: (v) =>
                      v?.isEmpty == true ? 'City is required' : null,
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: TextFormField(
                  controller: _zipController,
                  decoration: const InputDecoration(labelText: 'ZIP Code'),
                  keyboardType: TextInputType.number,
                  validator: (v) =>
                      v?.isEmpty == true ? 'ZIP is required' : null,
                ),
              ),
            ],
          ),
          const SizedBox(height: 12),
          TextFormField(
            controller: _phoneController,
            decoration: const InputDecoration(labelText: 'Phone Number'),
            keyboardType: TextInputType.phone,
            validator: (v) =>
                v?.isEmpty == true ? 'Phone is required' : null,
          ),
        ],
      ),
    );
  }

  Widget _buildPaymentForm() {
    return Column(
      children: [
        RadioListTile<String>(
          title: const Row(
            children: [
              Icon(Icons.credit_card),
              SizedBox(width: 8),
              Text('Credit/Debit Card'),
            ],
          ),
          value: 'card',
          groupValue: _selectedPaymentMethod,
          onChanged: (v) =>
              setState(() => _selectedPaymentMethod = v!),
        ),
        RadioListTile<String>(
          title: const Row(
            children: [
              Icon(Icons.account_balance_wallet),
              SizedBox(width: 8),
              Text('Digital Wallet'),
            ],
          ),
          value: 'wallet',
          groupValue: _selectedPaymentMethod,
          onChanged: (v) =>
              setState(() => _selectedPaymentMethod = v!),
        ),
        if (_selectedPaymentMethod == 'card') ...[
          const SizedBox(height: 16),
          TextFormField(
            decoration: const InputDecoration(
              labelText: 'Card Number',
              hintText: '4242 4242 4242 4242',
            ),
            keyboardType: TextInputType.number,
          ),
          const SizedBox(height: 12),
          Row(
            children: [
              Expanded(
                child: TextFormField(
                  decoration: const InputDecoration(labelText: 'MM/YY'),
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: TextFormField(
                  decoration: const InputDecoration(labelText: 'CVV'),
                  keyboardType: TextInputType.number,
                ),
              ),
            ],
          ),
        ],
      ],
    );
  }

  Widget _buildOrderReview(cart, NumberFormat formatter) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        const Text('Order Items',
            style: TextStyle(fontWeight: FontWeight.bold)),
        const SizedBox(height: 8),
        ...cart.items.map(
          (item) => ListTile(
            contentPadding: EdgeInsets.zero,
            title: Text(item.product.name),
            subtitle: Text('Qty: ${item.quantity}'),
            trailing: Text(formatter.format(item.subtotal)),
          ),
        ),
        const Divider(),
        Row(
          mainAxisAlignment: MainAxisAlignment.spaceBetween,
          children: [
            const Text('Total',
                style: TextStyle(fontWeight: FontWeight.bold, fontSize: 16)),
            Text(
              formatter.format(cart.total),
              style: const TextStyle(
                  fontWeight: FontWeight.bold, fontSize: 16),
            ),
          ],
        ),
        const SizedBox(height: 16),
        if (_isProcessing)
          const Center(child: CircularProgressIndicator())
        else
          SizedBox(
            width: double.infinity,
            child: ElevatedButton(
              onPressed: _placeOrder,
              style: ElevatedButton.styleFrom(
                padding: const EdgeInsets.symmetric(vertical: 16),
                backgroundColor: Colors.green,
              ),
              child: Text(
                'Place Order • ${formatter.format(cart.total)}',
                style: const TextStyle(
                    fontSize: 16, color: Colors.white),
              ),
            ),
          ),
      ],
    );
  }

  void _onStepContinue() {
    if (_currentStep == 0) {
      if (_formKey.currentState?.validate() == true) {
        setState(() => _currentStep++);
      }
    } else if (_currentStep < 2) {
      setState(() => _currentStep++);
    }
  }

  Future<void> _placeOrder() async {
    setState(() => _isProcessing = true);

    try {
      // In production: create Stripe PaymentIntent and confirm
      await Future.delayed(const Duration(seconds: 2));

      ref.read(cartProvider.notifier).clearCart();

      if (mounted) {
        Navigator.pushReplacementNamed(
          context,
          '/order-confirmation',
          arguments: 'ORD-${DateTime.now().millisecondsSinceEpoch}',
        );
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Payment failed: $e')),
        );
      }
    } finally {
      if (mounted) setState(() => _isProcessing = false);
    }
  }
}

// lib/screens/order_tracking_screen.dart
class OrderTrackingScreen extends StatelessWidget {
  final String orderId;

  const OrderTrackingScreen({super.key, required this.orderId});

  @override
  Widget build(BuildContext context) {
    // Mock tracking events
    final trackingEvents = [
      TrackingEvent(
        status: 'Order Placed',
        description: 'Your order has been placed successfully',
        location: 'Online',
        timestamp: DateTime.now().subtract(const Duration(hours: 24)),
      ),
      TrackingEvent(
        status: 'Processing',
        description: 'Your order is being prepared',
        location: 'Warehouse, Bangkok',
        timestamp: DateTime.now().subtract(const Duration(hours: 20)),
      ),
      TrackingEvent(
        status: 'Shipped',
        description: 'Package handed to courier',
        location: 'Distribution Center, Bangkok',
        timestamp: DateTime.now().subtract(const Duration(hours: 12)),
      ),
    ];

    return Scaffold(
      appBar: AppBar(title: Text('Order $orderId')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          Card(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const Text(
                    'Tracking Number',
                    style: TextStyle(color: Colors.grey),
                  ),
                  const SizedBox(height: 4),
                  const Text(
                    'TH-1234567890',
                    style: TextStyle(
                        fontSize: 18, fontWeight: FontWeight.bold),
                  ),
                  const SizedBox(height: 12),
                  Container(
                    padding: const EdgeInsets.symmetric(
                        horizontal: 12, vertical: 6),
                    decoration: BoxDecoration(
                      color: Colors.orange.shade100,
                      borderRadius: BorderRadius.circular(20),
                    ),
                    child: const Text(
                      'In Transit',
                      style: TextStyle(color: Colors.orange),
                    ),
                  ),
                ],
              ),
            ),
          ),
          const SizedBox(height: 16),
          const Text(
            'Tracking History',
            style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
          ),
          const SizedBox(height: 8),
          ...trackingEvents.asMap().entries.map(
            (entry) {
              final event = entry.value;
              final isFirst = entry.key == 0;
              final isLast = entry.key == trackingEvents.length - 1;

              return IntrinsicHeight(
                child: Row(
                  children: [
                    Column(
                      children: [
                        Container(
                          width: 12,
                          height: 12,
                          decoration: BoxDecoration(
                            color: isFirst ? Colors.blue : Colors.grey,
                            shape: BoxShape.circle,
                          ),
                        ),
                        if (!isLast)
                          Expanded(
                            child: Container(
                              width: 2,
                              color: Colors.grey.shade300,
                            ),
                          ),
                      ],
                    ),
                    const SizedBox(width: 16),
                    Expanded(
                      child: Padding(
                        padding: const EdgeInsets.only(bottom: 16),
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            Text(
                              event.status,
                              style: TextStyle(
                                fontWeight: FontWeight.bold,
                                color: isFirst ? Colors.blue : Colors.black,
                              ),
                            ),
                            Text(
                              event.description,
                              style: const TextStyle(color: Colors.grey),
                            ),
                            Text(
                              event.location,
                              style: const TextStyle(fontSize: 12),
                            ),
                            Text(
                              '${event.timestamp.day}/${event.timestamp.month}/${event.timestamp.year} ${event.timestamp.hour}:${event.timestamp.minute.toString().padLeft(2, '0')}',
                              style: TextStyle(
                                fontSize: 12,
                                color: Colors.grey.shade500,
                              ),
                            ),
                          ],
                        ),
                      ),
                    ),
                  ],
                ),
              );
            },
          ),
        ],
      ),
    );
  }
}
```

---

**← [Part 62](part-62-security-best-practices.md)**
**ต่อไป: [Part 64 →](part-64-real-world-app-social.md)**
