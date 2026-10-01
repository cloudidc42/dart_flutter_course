# Part 30: World-Class App - E-Commerce Complete
## ขั้นตอนที่ 1081-1120

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ทุก concept ที่เรียนมาสร้าง E-Commerce app ระดับ Production
- Clean Architecture + BLoC + Riverpod
- Offline-first
- Real-time notifications
- Advanced animations
- Performance-optimized

---

## ขั้นตอนที่ 1081: Project Structure

```
lib/
├── core/
│   ├── constants/
│   │   ├── app_constants.dart
│   │   └── api_constants.dart
│   ├── di/
│   │   └── injection_container.dart
│   ├── error/
│   │   ├── exceptions.dart
│   │   └── failures.dart
│   ├── network/
│   │   ├── api_client.dart
│   │   └── network_info.dart
│   ├── utils/
│   │   ├── validators.dart
│   │   └── formatters.dart
│   └── widgets/
│       ├── loading_widget.dart
│       └── error_widget.dart
│
├── features/
│   ├── auth/
│   ├── catalog/
│   ├── cart/
│   ├── checkout/
│   ├── orders/
│   └── profile/
│
└── app/
    ├── app.dart
    ├── router.dart
    └── theme.dart
```

---

## ขั้นตอนที่ 1082: Product Catalog Feature

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// ─── Domain ───
class Product {
  final String id;
  final String name;
  final String description;
  final double price;
  final double? salePrice;
  final List<String> imageUrls;
  final String category;
  final double rating;
  final int reviewCount;
  final int stock;
  final List<String> tags;

  const Product({
    required this.id,
    required this.name,
    required this.description,
    required this.price,
    this.salePrice,
    required this.imageUrls,
    required this.category,
    this.rating = 0,
    this.reviewCount = 0,
    required this.stock,
    this.tags = const [],
  });

  bool get isOnSale => salePrice != null && salePrice! < price;
  bool get inStock => stock > 0;
  double get effectivePrice => salePrice ?? price;
  double get discountPercent => isOnSale ? (price - salePrice!) / price * 100 : 0;
}

// ─── State ───
class CatalogState {
  final List<Product> products;
  final List<Product> featured;
  final List<String> categories;
  final String selectedCategory;
  final String searchQuery;
  final String sortBy;
  final bool isLoading;
  final String? error;
  final int page;
  final bool hasMore;

  const CatalogState({
    this.products = const [],
    this.featured = const [],
    this.categories = const [],
    this.selectedCategory = 'all',
    this.searchQuery = '',
    this.sortBy = 'popular',
    this.isLoading = false,
    this.error,
    this.page = 1,
    this.hasMore = true,
  });

  CatalogState copyWith({
    List<Product>? products,
    List<Product>? featured,
    List<String>? categories,
    String? selectedCategory,
    String? searchQuery,
    String? sortBy,
    bool? isLoading,
    String? error,
    int? page,
    bool? hasMore,
  }) {
    return CatalogState(
      products: products ?? this.products,
      featured: featured ?? this.featured,
      categories: categories ?? this.categories,
      selectedCategory: selectedCategory ?? this.selectedCategory,
      searchQuery: searchQuery ?? this.searchQuery,
      sortBy: sortBy ?? this.sortBy,
      isLoading: isLoading ?? this.isLoading,
      error: error,
      page: page ?? this.page,
      hasMore: hasMore ?? this.hasMore,
    );
  }
}

// ─── Notifier ───
class CatalogNotifier extends AsyncNotifier<CatalogState> {
  @override
  Future<CatalogState> build() async {
    List<Product> products = await _fetchProducts();
    return CatalogState(
      products: products,
      categories: _extractCategories(products),
    );
  }

  Future<List<Product>> _fetchProducts({int page = 1, String? category}) async {
    await Future.delayed(const Duration(milliseconds: 800));
    return List.generate(
      20,
      (i) => Product(
        id: 'product_${page * 100 + i}',
        name: 'สินค้า ${page * 100 + i}',
        description: 'รายละเอียดของสินค้า ${page * 100 + i}',
        price: (i + 1) * 100.0,
        salePrice: i % 3 == 0 ? (i + 1) * 80.0 : null,
        imageUrls: ['https://picsum.photos/400/400?random=${page * 100 + i}'],
        category: ['Electronics', 'Fashion', 'Home', 'Beauty'][i % 4],
        rating: 3.5 + (i % 3) * 0.5,
        reviewCount: (i + 1) * 10,
        stock: i % 5 == 0 ? 0 : (i + 1) * 5,
        tags: ['new', 'popular'].sublist(0, i % 2 + 1),
      ),
    );
  }

  List<String> _extractCategories(List<Product> products) {
    Set<String> cats = products.map((p) => p.category).toSet();
    return ['all', ...cats];
  }

  void filterByCategory(String category) {
    state.whenData((s) {
      state = AsyncData(s.copyWith(selectedCategory: category, page: 1));
    });
  }

  void search(String query) {
    state.whenData((s) {
      state = AsyncData(s.copyWith(searchQuery: query, page: 1));
    });
  }

  void sortBy(String sort) {
    state.whenData((s) {
      state = AsyncData(s.copyWith(sortBy: sort));
    });
  }

  Future<void> loadMore() async {
    CatalogState? current = state.valueOrNull;
    if (current == null || !current.hasMore || current.isLoading) return;

    state = AsyncData(current.copyWith(isLoading: true));

    List<Product> more = await _fetchProducts(page: current.page + 1);
    state = AsyncData(current.copyWith(
      products: [...current.products, ...more],
      page: current.page + 1,
      isLoading: false,
      hasMore: more.length == 20,
    ));
  }

  Future<void> refresh() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      List<Product> products = await _fetchProducts();
      return CatalogState(
        products: products,
        categories: _extractCategories(products),
      );
    });
  }
}

final catalogProvider = AsyncNotifierProvider<CatalogNotifier, CatalogState>(
  CatalogNotifier.new,
);

// Derived providers
final filteredProductsProvider = Provider<List<Product>>((ref) {
  CatalogState? state = ref.watch(catalogProvider).valueOrNull;
  if (state == null) return [];

  List<Product> products = state.products;

  if (state.selectedCategory != 'all') {
    products = products.where((p) => p.category == state.selectedCategory).toList();
  }

  if (state.searchQuery.isNotEmpty) {
    String q = state.searchQuery.toLowerCase();
    products = products.where((p) =>
      p.name.toLowerCase().contains(q) ||
      p.description.toLowerCase().contains(q)
    ).toList();
  }

  switch (state.sortBy) {
    case 'price_asc':
      products.sort((a, b) => a.effectivePrice.compareTo(b.effectivePrice));
      break;
    case 'price_desc':
      products.sort((a, b) => b.effectivePrice.compareTo(a.effectivePrice));
      break;
    case 'rating':
      products.sort((a, b) => b.rating.compareTo(a.rating));
      break;
  }

  return products;
});
```

---

## ขั้นตอนที่ 1083: Catalog UI

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class CatalogPage extends ConsumerStatefulWidget {
  const CatalogPage({super.key});

  @override
  ConsumerState<CatalogPage> createState() => _CatalogPageState();
}

class _CatalogPageState extends ConsumerState<CatalogPage> {
  final TextEditingController _searchController = TextEditingController();
  final ScrollController _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(_onScroll);
  }

  @override
  void dispose() {
    _searchController.dispose();
    _scrollController.dispose();
    super.dispose();
  }

  void _onScroll() {
    if (_scrollController.position.pixels >=
        _scrollController.position.maxScrollExtent - 300) {
      ref.read(catalogProvider.notifier).loadMore();
    }
  }

  @override
  Widget build(BuildContext context) {
    AsyncValue<CatalogState> catalogAsync = ref.watch(catalogProvider);
    List<Product> filtered = ref.watch(filteredProductsProvider);

    return Scaffold(
      body: NestedScrollView(
        headerSliverBuilder: (context, _) => [
          // Search App Bar
          SliverAppBar(
            floating: true,
            snap: true,
            title: TextField(
              controller: _searchController,
              decoration: InputDecoration(
                hintText: 'ค้นหาสินค้า...',
                prefixIcon: const Icon(Icons.search),
                suffixIcon: _searchController.text.isNotEmpty
                    ? IconButton(
                        icon: const Icon(Icons.clear),
                        onPressed: () {
                          _searchController.clear();
                          ref.read(catalogProvider.notifier).search('');
                        },
                      )
                    : null,
                filled: true,
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(30),
                  borderSide: BorderSide.none,
                ),
                contentPadding: const EdgeInsets.symmetric(horizontal: 16),
              ),
              onChanged: (q) => ref.read(catalogProvider.notifier).search(q),
            ),
            actions: [
              IconButton(
                icon: const Icon(Icons.filter_list),
                onPressed: () => _showFilterSheet(context),
              ),
            ],
          ),

          // Category Bar
          SliverToBoxAdapter(
            child: catalogAsync.whenOrNull(
              data: (state) => SizedBox(
                height: 50,
                child: ListView.builder(
                  scrollDirection: Axis.horizontal,
                  padding: const EdgeInsets.symmetric(horizontal: 16),
                  itemCount: state.categories.length,
                  itemBuilder: (_, i) {
                    String cat = state.categories[i];
                    bool isSelected = cat == state.selectedCategory;
                    return Padding(
                      padding: const EdgeInsets.only(right: 8, top: 8, bottom: 8),
                      child: FilterChip(
                        label: Text(cat == 'all' ? 'ทั้งหมด' : cat),
                        selected: isSelected,
                        onSelected: (_) =>
                            ref.read(catalogProvider.notifier).filterByCategory(cat),
                      ),
                    );
                  },
                ),
              ),
            ),
          ),
        ],
        body: catalogAsync.when(
          data: (_) => RefreshIndicator(
            onRefresh: () => ref.read(catalogProvider.notifier).refresh(),
            child: filtered.isEmpty
                ? const Center(child: Text('ไม่พบสินค้า'))
                : GridView.builder(
                    controller: _scrollController,
                    padding: const EdgeInsets.all(12),
                    gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                      crossAxisCount: 2,
                      childAspectRatio: 0.7,
                      crossAxisSpacing: 12,
                      mainAxisSpacing: 12,
                    ),
                    itemCount: filtered.length,
                    itemBuilder: (_, i) => ProductCard(product: filtered[i]),
                  ),
          ),
          loading: () => const Center(child: CircularProgressIndicator()),
          error: (e, _) => Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                const Icon(Icons.error_outline, size: 60, color: Colors.red),
                Text('เกิดข้อผิดพลาด: $e'),
                ElevatedButton(
                  onPressed: () => ref.read(catalogProvider.notifier).refresh(),
                  child: const Text('ลองใหม่'),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }

  void _showFilterSheet(BuildContext context) {
    showModalBottomSheet(
      context: context,
      builder: (context) => Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          const Padding(
            padding: EdgeInsets.all(16),
            child: Text('เรียงลำดับ', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          ),
          ...[
            ('popular', 'ยอดนิยม'),
            ('price_asc', 'ราคาต่ำไปสูง'),
            ('price_desc', 'ราคาสูงไปต่ำ'),
            ('rating', 'คะแนนสูงสุด'),
          ].map((sort) => ListTile(
            title: Text(sort.$2),
            onTap: () {
              ref.read(catalogProvider.notifier).sortBy(sort.$1);
              Navigator.pop(context);
            },
          )),
        ],
      ),
    );
  }
}

// ─── Product Card ───
class ProductCard extends ConsumerWidget {
  final Product product;

  const ProductCard({super.key, required this.product});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Card(
      clipBehavior: Clip.antiAlias,
      child: InkWell(
        onTap: () => Navigator.push(
          context,
          MaterialPageRoute(builder: (_) => ProductDetailPage(product: product)),
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Image
            Expanded(
              child: Stack(
                children: [
                  Hero(
                    tag: 'product-${product.id}',
                    child: Image.network(
                      product.imageUrls.first,
                      width: double.infinity,
                      fit: BoxFit.cover,
                      errorBuilder: (_, __, ___) => Container(
                        color: Colors.grey.shade200,
                        child: const Icon(Icons.image, size: 60),
                      ),
                    ),
                  ),
                  if (product.isOnSale)
                    Positioned(
                      top: 8,
                      left: 8,
                      child: Container(
                        padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
                        decoration: BoxDecoration(
                          color: Colors.red,
                          borderRadius: BorderRadius.circular(12),
                        ),
                        child: Text(
                          '-${product.discountPercent.toInt()}%',
                          style: const TextStyle(color: Colors.white, fontSize: 11, fontWeight: FontWeight.bold),
                        ),
                      ),
                    ),
                  if (!product.inStock)
                    Container(
                      color: Colors.black45,
                      child: const Center(
                        child: Text('หมด', style: TextStyle(color: Colors.white, fontSize: 18, fontWeight: FontWeight.bold)),
                      ),
                    ),
                ],
              ),
            ),
            // Info
            Padding(
              padding: const EdgeInsets.all(8),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    product.name,
                    maxLines: 2,
                    overflow: TextOverflow.ellipsis,
                    style: const TextStyle(fontWeight: FontWeight.w600, fontSize: 13),
                  ),
                  const SizedBox(height: 4),
                  Row(
                    children: [
                      const Icon(Icons.star, size: 12, color: Colors.amber),
                      Text(' ${product.rating}', style: const TextStyle(fontSize: 11)),
                      Text(' (${product.reviewCount})', style: TextStyle(fontSize: 11, color: Colors.grey.shade600)),
                    ],
                  ),
                  const SizedBox(height: 4),
                  if (product.isOnSale) ...[
                    Text(
                      '฿${product.price.toStringAsFixed(0)}',
                      style: TextStyle(
                        fontSize: 11,
                        decoration: TextDecoration.lineThrough,
                        color: Colors.grey.shade500,
                      ),
                    ),
                    Text(
                      '฿${product.salePrice!.toStringAsFixed(0)}',
                      style: const TextStyle(
                        color: Colors.red,
                        fontWeight: FontWeight.bold,
                        fontSize: 16,
                      ),
                    ),
                  ] else
                    Text(
                      '฿${product.price.toStringAsFixed(0)}',
                      style: TextStyle(
                        color: Theme.of(context).colorScheme.primary,
                        fontWeight: FontWeight.bold,
                        fontSize: 16,
                      ),
                    ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}

// Placeholder for detail page
class ProductDetailPage extends StatelessWidget {
  final Product product;
  const ProductDetailPage({super.key, required this.product});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(product.name)),
      body: Center(child: Text('Detail: ${product.name}')),
    );
  }
}
```

---

**← [Part 29 - Enterprise Architecture](part-29-enterprise.md)**

---

## 🎓 หลักสูตรครบสมบูรณ์

ยินดีด้วยที่เรียนครบหลักสูตรระดับ World-Class Dart & Flutter!

**สิ่งที่คุณเรียนรู้ไปแล้ว:**
- ✅ Dart fundamentals (Variables, OOP, Generics, Dart 3.0)
- ✅ Flutter Widgets & Layout
- ✅ State Management (setState, Provider, Riverpod, BLoC)
- ✅ Navigation (GoRouter, deep linking)
- ✅ HTTP/REST APIs (Dio, Repository pattern)
- ✅ Local Storage (SharedPreferences, SQLite, Hive)
- ✅ Firebase (Auth, Firestore, Storage, RTDB)
- ✅ Testing (Unit, Widget, Integration, TDD)
- ✅ Animations (Implicit, Explicit, Hero, Lottie)
- ✅ Custom Painting & Charts
- ✅ Platform Channels (Native code)
- ✅ Clean Architecture & Design Patterns
- ✅ CI/CD & Deployment
- ✅ Enterprise Patterns
- ✅ Performance Optimization
- ✅ Internationalization

**ก้าวต่อไป:**
1. Build project จริงๆ ที่ใช้ความรู้ทั้งหมด
2. Contribute to open source Flutter packages
3. เรียน Flutter Web & Desktop
4. Advanced AI/ML integration กับ Flutter
