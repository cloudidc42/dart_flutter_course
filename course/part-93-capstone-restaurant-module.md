# Part 93: Capstone – Restaurant Module
## ขั้นตอนที่ 3601-3640

## 🎯 เป้าหมายของ Part นี้
- Restaurant listing พร้อม search และ filters
- Restaurant detail page พร้อม menu categories
- Menu item detail พร้อม customization options
- Favorites system
- Restaurant rating และ reviews
- Full working Flutter + Firestore code

---

## ขั้นตอนที่ 3601: Restaurant Repository Implementation

```dart
// lib/features/restaurant/data/repositories/restaurant_repository_impl.dart

import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:dartz/dartz.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class RestaurantRepositoryImpl implements RestaurantRepository {
  final FirebaseFirestore _firestore;

  RestaurantRepositoryImpl({required FirebaseFirestore firestore})
      : _firestore = firestore;

  @override
  Future<List<RestaurantEntity>> getNearbyRestaurants({
    required double latitude,
    required double longitude,
    double radiusKm = 10,
    String? cuisineFilter,
  }) async {
    try {
      Query<Map<String, dynamic>> query = _firestore
          .collection(FirestoreCollections.restaurants)
          .where('isActive', isEqualTo: true)
          .orderBy('rating', descending: true)
          .limit(30);

      if (cuisineFilter != null && cuisineFilter.isNotEmpty) {
        query =
            query.where('cuisineTypes', arrayContains: cuisineFilter);
      }

      final snapshot = await query.get();
      final restaurants = snapshot.docs
          .map((doc) => RestaurantModel.fromFirestore(doc.data(), doc.id))
          .toList();

      // Client-side distance filter (real app: use GeoFlutterFire)
      return restaurants.where((r) {
        final dist = _calculateDistance(
          latitude, longitude,
          r.location.latitude, r.location.longitude,
        );
        return dist <= radiusKm;
      }).toList();
    } catch (e) {
      throw ServerException(message: e.toString());
    }
  }

  double _calculateDistance(
    double lat1, double lon1,
    double lat2, double lon2,
  ) {
    const earthRadius = 6371.0;
    final dLat = _toRad(lat2 - lat1);
    final dLon = _toRad(lon2 - lon1);
    final a = (dLat / 2).abs() * (dLat / 2).abs() +
        (dLon / 2).abs() * (dLon / 2).abs();
    final c = 2 * (a < 1 ? a : 1);
    return earthRadius * c;
  }

  double _toRad(double deg) => deg * 3.14159265358979323846 / 180;

  @override
  Future<RestaurantEntity> getRestaurantById(String id) async {
    final doc = await _firestore
        .collection(FirestoreCollections.restaurants)
        .doc(id)
        .get();
    if (!doc.exists) {
      throw const ServerException(message: 'Restaurant not found');
    }
    return RestaurantModel.fromFirestore(doc.data()!, doc.id);
  }

  @override
  Future<List<MenuCategoryEntity>> getRestaurantMenu(
      String restaurantId) async {
    final categoriesSnap = await _firestore
        .collection(FirestoreCollections.restaurants)
        .doc(restaurantId)
        .collection(FirestoreCollections.categories)
        .orderBy('sortOrder')
        .get();

    final categories = <MenuCategoryEntity>[];

    for (final catDoc in categoriesSnap.docs) {
      final itemsSnap = await _firestore
          .collection(FirestoreCollections.restaurants)
          .doc(restaurantId)
          .collection(FirestoreCollections.menuItems)
          .where('categoryId', isEqualTo: catDoc.id)
          .where('isAvailable', isEqualTo: true)
          .orderBy('sortOrder')
          .get();

      final items = itemsSnap.docs.map((itemDoc) {
        final data = itemDoc.data();
        return MenuItemEntity(
          id: itemDoc.id,
          restaurantId: restaurantId,
          categoryId: catDoc.id,
          name: data['name'] as String,
          description: data['description'] as String? ?? '',
          imageUrl: data['imageUrl'] as String? ?? '',
          price: (data['price'] as num).toDouble(),
          isAvailable: data['isAvailable'] as bool? ?? true,
          isPopular: data['isPopular'] as bool? ?? false,
          isVegetarian: data['isVegetarian'] as bool? ?? false,
          isSpicy: data['isSpicy'] as bool? ?? false,
          calories: data['calories'] as int? ?? 0,
          options: (data['options'] as List? ?? [])
              .map((o) => MenuOption.fromJson(o as Map<String, dynamic>))
              .toList(),
          sortOrder: data['sortOrder'] as int? ?? 0,
        );
      }).toList();

      final catData = catDoc.data();
      categories.add(MenuCategoryEntity(
        id: catDoc.id,
        restaurantId: restaurantId,
        name: catData['name'] as String,
        imageUrl: catData['imageUrl'] as String?,
        sortOrder: catData['sortOrder'] as int? ?? 0,
        items: items,
      ));
    }
    return categories;
  }

  @override
  Future<List<RestaurantEntity>> searchRestaurants(String query) async {
    if (query.isEmpty) return [];
    final snapshot = await _firestore
        .collection(FirestoreCollections.restaurants)
        .where('isActive', isEqualTo: true)
        .get();

    final lowerQuery = query.toLowerCase();
    return snapshot.docs
        .map((doc) => RestaurantModel.fromFirestore(doc.data(), doc.id))
        .where((r) =>
            r.name.toLowerCase().contains(lowerQuery) ||
            r.cuisineTypes
                .any((c) => c.toLowerCase().contains(lowerQuery)))
        .toList();
  }

  @override
  Future<void> toggleFavorite(String userId, String restaurantId) async {
    final ref = _firestore
        .collection(FirestoreCollections.users)
        .doc(userId)
        .collection('favorites')
        .doc(restaurantId);

    final doc = await ref.get();
    if (doc.exists) {
      await ref.delete();
    } else {
      await ref.set({'restaurantId': restaurantId, 'addedAt': FieldValue.serverTimestamp()});
    }
  }

  @override
  Future<List<String>> getFavoriteIds(String userId) async {
    final snap = await _firestore
        .collection(FirestoreCollections.users)
        .doc(userId)
        .collection('favorites')
        .get();
    return snap.docs.map((d) => d.id).toList();
  }

  @override
  Stream<RestaurantEntity> watchRestaurant(String restaurantId) {
    return _firestore
        .collection(FirestoreCollections.restaurants)
        .doc(restaurantId)
        .snapshots()
        .map((snap) => RestaurantModel.fromFirestore(snap.data()!, snap.id));
  }
}
```

---

## ขั้นตอนที่ 3602: Restaurant Providers (Riverpod)

```dart
// lib/features/restaurant/presentation/providers/restaurant_providers.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';

final restaurantRepositoryProvider = Provider<RestaurantRepository>((ref) {
  return RestaurantRepositoryImpl(
    firestore: ref.watch(firestoreProvider),
  );
});

final restaurantFiltersProvider = StateProvider<RestaurantFilters>(
    (_) => const RestaurantFilters());

class RestaurantFilters {
  final String searchQuery;
  final String? cuisineFilter;
  final String sortBy; // 'rating', 'distance', 'delivery_time'
  final double? maxDeliveryFee;
  final bool openNowOnly;

  const RestaurantFilters({
    this.searchQuery = '',
    this.cuisineFilter,
    this.sortBy = 'rating',
    this.maxDeliveryFee,
    this.openNowOnly = false,
  });

  RestaurantFilters copyWith({
    String? searchQuery,
    String? cuisineFilter,
    String? sortBy,
    double? maxDeliveryFee,
    bool? openNowOnly,
  }) {
    return RestaurantFilters(
      searchQuery: searchQuery ?? this.searchQuery,
      cuisineFilter: cuisineFilter ?? this.cuisineFilter,
      sortBy: sortBy ?? this.sortBy,
      maxDeliveryFee: maxDeliveryFee ?? this.maxDeliveryFee,
      openNowOnly: openNowOnly ?? this.openNowOnly,
    );
  }
}

final nearbyRestaurantsProvider =
    FutureProvider.autoDispose<List<RestaurantEntity>>((ref) async {
  // In a real app, get location from a location provider
  const lat = 13.7563;
  const lon = 100.5018;
  final filters = ref.watch(restaurantFiltersProvider);

  List<RestaurantEntity> restaurants;

  if (filters.searchQuery.isNotEmpty) {
    restaurants = await ref
        .watch(restaurantRepositoryProvider)
        .searchRestaurants(filters.searchQuery);
  } else {
    restaurants = await ref
        .watch(restaurantRepositoryProvider)
        .getNearbyRestaurants(
          latitude: lat,
          longitude: lon,
          cuisineFilter: filters.cuisineFilter,
        );
  }

  if (filters.openNowOnly) {
    restaurants = restaurants.where((r) => r.isOpen).toList();
  }

  if (filters.maxDeliveryFee != null) {
    restaurants = restaurants
        .where((r) => r.deliveryFee <= filters.maxDeliveryFee!)
        .toList();
  }

  switch (filters.sortBy) {
    case 'delivery_time':
      restaurants.sort((a, b) =>
          a.deliveryTimeMinutes.compareTo(b.deliveryTimeMinutes));
    case 'distance':
      // Would sort by distance in a real implementation
      break;
    default:
      restaurants.sort((a, b) => b.rating.compareTo(a.rating));
  }

  return restaurants;
});

final restaurantDetailProvider =
    FutureProvider.autoDispose.family<RestaurantEntity, String>((ref, id) {
  return ref.watch(restaurantRepositoryProvider).getRestaurantById(id);
});

final restaurantMenuProvider = FutureProvider.autoDispose
    .family<List<MenuCategoryEntity>, String>((ref, restaurantId) {
  return ref
      .watch(restaurantRepositoryProvider)
      .getRestaurantMenu(restaurantId);
});

final favoritesProvider =
    FutureProvider.autoDispose<List<String>>((ref) async {
  final user = ref.watch(authNotifierProvider).valueOrNull;
  if (user == null) return [];
  return ref
      .watch(restaurantRepositoryProvider)
      .getFavoriteIds(user.id);
});

final isFavoriteProvider =
    Provider.family<bool, String>((ref, restaurantId) {
  final favs = ref.watch(favoritesProvider).valueOrNull ?? [];
  return favs.contains(restaurantId);
});
```

---

## ขั้นตอนที่ 3603: Restaurant List Page

```dart
// lib/features/restaurant/presentation/pages/home_page.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

class HomePage extends ConsumerWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final restaurantsAsync = ref.watch(nearbyRestaurantsProvider);
    final filters = ref.watch(restaurantFiltersProvider);

    return Scaffold(
      body: CustomScrollView(
        slivers: [
          _buildSliverAppBar(context, ref, filters),
          _buildCategoryFilter(ref, filters),
          _buildFeaturedBanner(),
          restaurantsAsync.when(
            data: (restaurants) => _buildRestaurantList(context, restaurants),
            loading: () => const SliverFillRemaining(
              child: Center(child: CircularProgressIndicator()),
            ),
            error: (err, _) => SliverFillRemaining(
              child: Center(child: Text('เกิดข้อผิดพลาด: $err')),
            ),
          ),
        ],
      ),
    );
  }

  SliverAppBar _buildSliverAppBar(
    BuildContext context,
    WidgetRef ref,
    RestaurantFilters filters,
  ) {
    return SliverAppBar(
      floating: true,
      snap: true,
      title: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        mainAxisSize: MainAxisSize.min,
        children: [
          Row(
            children: [
              const Icon(Icons.location_on, color: Colors.orange, size: 20),
              const SizedBox(width: 4),
              Text(
                'กรุงเทพมหานคร',
                style: Theme.of(context).textTheme.titleMedium?.copyWith(
                      fontWeight: FontWeight.bold,
                    ),
              ),
              const Icon(Icons.arrow_drop_down, color: Colors.orange),
            ],
          ),
        ],
      ),
      bottom: PreferredSize(
        preferredSize: const Size.fromHeight(60),
        child: Padding(
          padding: const EdgeInsets.fromLTRB(16, 0, 16, 12),
          child: _SearchBar(
            initial: filters.searchQuery,
            onChanged: (query) {
              ref.read(restaurantFiltersProvider.notifier).update(
                    (s) => s.copyWith(searchQuery: query),
                  );
            },
          ),
        ),
      ),
    );
  }

  Widget _buildCategoryFilter(WidgetRef ref, RestaurantFilters filters) {
    final cuisines = [
      null, 'Thai', 'Japanese', 'Italian', 'Chinese',
      'American', 'Indian', 'Korean', 'Dessert',
    ];
    return SliverToBoxAdapter(
      child: SizedBox(
        height: 48,
        child: ListView.separated(
          scrollDirection: Axis.horizontal,
          padding: const EdgeInsets.symmetric(horizontal: 16),
          itemCount: cuisines.length,
          separatorBuilder: (_, __) => const SizedBox(width: 8),
          itemBuilder: (_, index) {
            final cuisine = cuisines[index];
            final label = cuisine ?? 'ทั้งหมด';
            final isSelected = filters.cuisineFilter == cuisine;
            return FilterChip(
              label: Text(label),
              selected: isSelected,
              onSelected: (_) {
                ref.read(restaurantFiltersProvider.notifier).update(
                      (s) => s.copyWith(cuisineFilter: cuisine),
                    );
              },
              selectedColor: Colors.orange.shade100,
              checkmarkColor: Colors.orange,
            );
          },
        ),
      ),
    );
  }

  Widget _buildFeaturedBanner() {
    return SliverToBoxAdapter(
      child: Container(
        height: 160,
        margin: const EdgeInsets.all(16),
        decoration: BoxDecoration(
          borderRadius: BorderRadius.circular(16),
          gradient: const LinearGradient(
            colors: [Color(0xFFFF6B35), Color(0xFFFF8E53)],
            begin: Alignment.topLeft,
            end: Alignment.bottomRight,
          ),
        ),
        child: Stack(
          children: [
            Positioned(
              left: 20,
              top: 20,
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const Text(
                    'ลด 30% วันนี้!',
                    style: TextStyle(
                      color: Colors.white,
                      fontSize: 24,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                  const SizedBox(height: 8),
                  const Text(
                    'สั่งอาหารออนไลน์ครั้งแรก',
                    style: TextStyle(color: Colors.white70, fontSize: 14),
                  ),
                  const SizedBox(height: 16),
                  ElevatedButton(
                    onPressed: () {},
                    style: ElevatedButton.styleFrom(
                      backgroundColor: Colors.white,
                      foregroundColor: Colors.orange,
                    ),
                    child: const Text('สั่งเลย'),
                  ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildRestaurantList(
      BuildContext context, List<RestaurantEntity> restaurants) {
    if (restaurants.isEmpty) {
      return const SliverFillRemaining(
        child: Center(child: Text('ไม่พบร้านอาหารในพื้นที่')),
      );
    }
    return SliverPadding(
      padding: const EdgeInsets.all(16),
      sliver: SliverList(
        delegate: SliverChildBuilderDelegate(
          (context, index) {
            final restaurant = restaurants[index];
            return Padding(
              padding: const EdgeInsets.only(bottom: 16),
              child: RestaurantCard(
                restaurant: restaurant,
                onTap: () => context.push('/restaurant/${restaurant.id}'),
              ),
            );
          },
          childCount: restaurants.length,
        ),
      ),
    );
  }
}

class _SearchBar extends StatefulWidget {
  final String initial;
  final ValueChanged<String> onChanged;

  const _SearchBar({required this.initial, required this.onChanged});

  @override
  State<_SearchBar> createState() => _SearchBarState();
}

class _SearchBarState extends State<_SearchBar> {
  late final TextEditingController _ctrl;

  @override
  void initState() {
    super.initState();
    _ctrl = TextEditingController(text: widget.initial);
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return TextField(
      controller: _ctrl,
      onChanged: widget.onChanged,
      decoration: InputDecoration(
        hintText: 'ค้นหาร้านอาหาร...',
        prefixIcon: const Icon(Icons.search),
        suffixIcon: _ctrl.text.isNotEmpty
            ? IconButton(
                icon: const Icon(Icons.clear),
                onPressed: () {
                  _ctrl.clear();
                  widget.onChanged('');
                },
              )
            : null,
        filled: true,
        fillColor: Colors.grey.shade100,
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(12),
          borderSide: BorderSide.none,
        ),
        contentPadding: const EdgeInsets.symmetric(horizontal: 16),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3604: Restaurant Card Widget

```dart
// lib/features/restaurant/presentation/widgets/restaurant_card.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:cached_network_image/cached_network_image.dart';

class RestaurantCard extends ConsumerWidget {
  final RestaurantEntity restaurant;
  final VoidCallback onTap;

  const RestaurantCard({
    super.key,
    required this.restaurant,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final isFav = ref.watch(isFavoriteProvider(restaurant.id));
    final user = ref.watch(authNotifierProvider).valueOrNull;

    return GestureDetector(
      onTap: onTap,
      child: Card(
        elevation: 2,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(16),
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            _buildImage(context, isFav, user, ref),
            _buildInfo(context),
          ],
        ),
      ),
    );
  }

  Widget _buildImage(
    BuildContext context,
    bool isFav,
    UserEntity? user,
    WidgetRef ref,
  ) {
    return Stack(
      children: [
        ClipRRect(
          borderRadius: const BorderRadius.only(
            topLeft: Radius.circular(16),
            topRight: Radius.circular(16),
          ),
          child: CachedNetworkImage(
            imageUrl: restaurant.imageUrl,
            height: 160,
            width: double.infinity,
            fit: BoxFit.cover,
            placeholder: (_, __) => Container(
              color: Colors.grey.shade200,
              child: const Center(child: CircularProgressIndicator()),
            ),
            errorWidget: (_, __, ___) => Container(
              color: Colors.grey.shade200,
              child: const Icon(Icons.restaurant, size: 48, color: Colors.grey),
            ),
          ),
        ),
        Positioned(
          top: 8,
          right: 8,
          child: CircleAvatar(
            backgroundColor: Colors.white,
            radius: 18,
            child: IconButton(
              icon: Icon(
                isFav ? Icons.favorite : Icons.favorite_border,
                color: isFav ? Colors.red : Colors.grey,
                size: 18,
              ),
              padding: EdgeInsets.zero,
              onPressed: user == null
                  ? null
                  : () => ref
                      .read(restaurantRepositoryProvider)
                      .toggleFavorite(user.id, restaurant.id),
            ),
          ),
        ),
        if (!restaurant.isOpen)
          Positioned.fill(
            child: ClipRRect(
              borderRadius: const BorderRadius.only(
                topLeft: Radius.circular(16),
                topRight: Radius.circular(16),
              ),
              child: Container(
                color: Colors.black54,
                child: const Center(
                  child: Text(
                    'ปิดอยู่',
                    style: TextStyle(
                      color: Colors.white,
                      fontSize: 20,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                ),
              ),
            ),
          ),
      ],
    );
  }

  Widget _buildInfo(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(12),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              Expanded(
                child: Text(
                  restaurant.name,
                  style: Theme.of(context).textTheme.titleMedium?.copyWith(
                        fontWeight: FontWeight.bold,
                      ),
                  maxLines: 1,
                  overflow: TextOverflow.ellipsis,
                ),
              ),
              Row(
                children: [
                  const Icon(Icons.star, color: Colors.amber, size: 16),
                  const SizedBox(width: 4),
                  Text(
                    restaurant.rating.toStringAsFixed(1),
                    style: const TextStyle(fontWeight: FontWeight.bold),
                  ),
                  Text(
                    ' (${restaurant.totalReviews})',
                    style: TextStyle(color: Colors.grey[600], fontSize: 12),
                  ),
                ],
              ),
            ],
          ),
          const SizedBox(height: 4),
          Text(
            restaurant.cuisineTypes.take(3).join(' · '),
            style: TextStyle(color: Colors.grey[600], fontSize: 13),
          ),
          const SizedBox(height: 8),
          Row(
            children: [
              _InfoChip(
                icon: Icons.access_time,
                label: '${restaurant.deliveryTimeMinutes} นาที',
              ),
              const SizedBox(width: 8),
              _InfoChip(
                icon: Icons.delivery_dining,
                label: restaurant.deliveryFee == 0
                    ? 'ส่งฟรี'
                    : '฿${restaurant.deliveryFee.toStringAsFixed(0)}',
                color:
                    restaurant.deliveryFee == 0 ? Colors.green : null,
              ),
              const SizedBox(width: 8),
              _InfoChip(
                icon: Icons.shopping_bag_outlined,
                label:
                    'ขั้นต่ำ ฿${restaurant.minimumOrder.toStringAsFixed(0)}',
              ),
            ],
          ),
        ],
      ),
    );
  }
}

class _InfoChip extends StatelessWidget {
  final IconData icon;
  final String label;
  final Color? color;

  const _InfoChip({required this.icon, required this.label, this.color});

  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        Icon(icon, size: 14, color: color ?? Colors.grey[600]),
        const SizedBox(width: 2),
        Text(
          label,
          style: TextStyle(
            fontSize: 12,
            color: color ?? Colors.grey[600],
            fontWeight:
                color != null ? FontWeight.bold : FontWeight.normal,
          ),
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 3605: Restaurant Detail Page

```dart
// lib/features/restaurant/presentation/pages/restaurant_detail_page.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

class RestaurantDetailPage extends ConsumerStatefulWidget {
  final String id;
  const RestaurantDetailPage({super.key, required this.id});

  @override
  ConsumerState<RestaurantDetailPage> createState() =>
      _RestaurantDetailPageState();
}

class _RestaurantDetailPageState
    extends ConsumerState<RestaurantDetailPage>
    with SingleTickerProviderStateMixin {
  late TabController _tabController;
  int _currentTab = 0;

  @override
  void dispose() {
    _tabController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final restaurantAsync =
        ref.watch(restaurantDetailProvider(widget.id));
    final menuAsync = ref.watch(restaurantMenuProvider(widget.id));

    return restaurantAsync.when(
      data: (restaurant) {
        final categories = menuAsync.valueOrNull ?? [];

        if (_tabController.length != categories.length &&
            categories.isNotEmpty) {
          _tabController = TabController(
            length: categories.length,
            vsync: this,
          );
        }

        return Scaffold(
          body: NestedScrollView(
            headerSliverBuilder: (context, _) => [
              _buildSliverHeader(restaurant),
              _buildTabBar(categories),
            ],
            body: menuAsync.when(
              data: (cats) => TabBarView(
                controller: _tabController,
                children: cats
                    .map((cat) => _MenuCategoryTab(
                          category: cat,
                          restaurantId: widget.id,
                        ))
                    .toList(),
              ),
              loading: () =>
                  const Center(child: CircularProgressIndicator()),
              error: (e, _) => Center(child: Text('Error: $e')),
            ),
          ),
          floatingActionButton: menuAsync.valueOrNull != null
              ? _buildViewCartFab(context)
              : null,
        );
      },
      loading: () =>
          const Scaffold(body: Center(child: CircularProgressIndicator())),
      error: (e, _) =>
          Scaffold(body: Center(child: Text('Error: $e'))),
    );
  }

  SliverAppBar _buildSliverHeader(RestaurantEntity restaurant) {
    return SliverAppBar(
      expandedHeight: 220,
      pinned: true,
      flexibleSpace: FlexibleSpaceBar(
        background: Stack(
          fit: StackFit.expand,
          children: [
            CachedNetworkImage(
              imageUrl: restaurant.imageUrl,
              fit: BoxFit.cover,
            ),
            Container(
              decoration: BoxDecoration(
                gradient: LinearGradient(
                  begin: Alignment.topCenter,
                  end: Alignment.bottomCenter,
                  colors: [Colors.transparent, Colors.black54],
                ),
              ),
            ),
            Positioned(
              bottom: 16,
              left: 16,
              right: 16,
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    restaurant.name,
                    style: const TextStyle(
                      color: Colors.white,
                      fontSize: 24,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                  Text(
                    restaurant.cuisineTypes.join(' · '),
                    style: const TextStyle(
                      color: Colors.white70,
                      fontSize: 14,
                    ),
                  ),
                ],
              ),
            ),
          ],
        ),
      ),
      actions: [
        IconButton(
          icon: const Icon(Icons.share),
          onPressed: () {},
        ),
        IconButton(
          icon: Icon(
            ref.watch(isFavoriteProvider(restaurant.id))
                ? Icons.favorite
                : Icons.favorite_border,
            color: Colors.red,
          ),
          onPressed: () {
            final user = ref.read(authNotifierProvider).valueOrNull;
            if (user != null) {
              ref
                  .read(restaurantRepositoryProvider)
                  .toggleFavorite(user.id, restaurant.id);
            }
          },
        ),
      ],
    );
  }

  SliverPersistentHeader _buildTabBar(List<MenuCategoryEntity> categories) {
    if (categories.isEmpty) {
      _tabController = TabController(length: 1, vsync: this);
    } else if (_tabController.length != categories.length) {
      _tabController =
          TabController(length: categories.length, vsync: this);
    }

    return SliverPersistentHeader(
      pinned: true,
      delegate: _StickyTabBarDelegate(
        TabBar(
          controller: _tabController,
          isScrollable: true,
          labelColor: Colors.orange,
          unselectedLabelColor: Colors.grey,
          indicatorColor: Colors.orange,
          tabs: categories.map((c) => Tab(text: c.name)).toList(),
        ),
      ),
    );
  }

  Widget _buildViewCartFab(BuildContext context) {
    final cartCount = ref.watch(cartItemCountProvider);
    if (cartCount == 0) return const SizedBox.shrink();
    return FloatingActionButton.extended(
      backgroundColor: Colors.orange,
      onPressed: () => context.push('/cart'),
      icon: const Icon(Icons.shopping_cart, color: Colors.white),
      label: Text(
        'ดูตะกร้า ($cartCount)',
        style: const TextStyle(color: Colors.white),
      ),
    );
  }
}

class _StickyTabBarDelegate extends SliverPersistentHeaderDelegate {
  final TabBar tabBar;
  const _StickyTabBarDelegate(this.tabBar);

  @override
  double get minExtent => tabBar.preferredSize.height;
  @override
  double get maxExtent => tabBar.preferredSize.height;

  @override
  Widget build(
      BuildContext context, double shrinkOffset, bool overlapsContent) {
    return Container(
      color: Theme.of(context).scaffoldBackgroundColor,
      child: tabBar,
    );
  }

  @override
  bool shouldRebuild(_StickyTabBarDelegate old) => false;
}

class _MenuCategoryTab extends StatelessWidget {
  final MenuCategoryEntity category;
  final String restaurantId;

  const _MenuCategoryTab({
    required this.category,
    required this.restaurantId,
  });

  @override
  Widget build(BuildContext context) {
    return ListView.separated(
      padding: const EdgeInsets.all(16),
      itemCount: category.items.length,
      separatorBuilder: (_, __) => const Divider(),
      itemBuilder: (_, index) {
        final item = category.items[index];
        return MenuItemTile(
          item: item,
          onTap: () => showModalBottomSheet(
            context: context,
            isScrollControlled: true,
            shape: const RoundedRectangleBorder(
              borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
            ),
            builder: (_) => MenuItemDetailSheet(item: item),
          ),
        );
      },
    );
  }
}
```

---

## ขั้นตอนที่ 3606: Menu Item Detail Sheet

```dart
// lib/features/restaurant/presentation/widgets/menu_item_detail_sheet.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class MenuItemDetailSheet extends ConsumerStatefulWidget {
  final MenuItemEntity item;
  const MenuItemDetailSheet({super.key, required this.item});

  @override
  ConsumerState<MenuItemDetailSheet> createState() =>
      _MenuItemDetailSheetState();
}

class _MenuItemDetailSheetState extends ConsumerState<MenuItemDetailSheet> {
  int _quantity = 1;
  final Map<String, List<String>> _selectedChoices = {};
  final _instructionsCtrl = TextEditingController();
  double _extraCost = 0;

  @override
  void dispose() {
    _instructionsCtrl.dispose();
    super.dispose();
  }

  void _toggleChoice(MenuOption option, MenuOptionChoice choice) {
    setState(() {
      final current = _selectedChoices[option.id] ?? [];

      if (option.isMultiSelect) {
        if (current.contains(choice.id)) {
          current.remove(choice.id);
        } else if (current.length < option.maxSelections) {
          current.add(choice.id);
        }
      } else {
        _selectedChoices[option.id] = [choice.id];
      }

      _recalculateExtra();
    });
  }

  void _recalculateExtra() {
    double extra = 0;
    for (final option in widget.item.options) {
      final selected = _selectedChoices[option.id] ?? [];
      for (final choiceId in selected) {
        final choice = option.choices.firstWhere(
          (c) => c.id == choiceId,
          orElse: () =>
              const MenuOptionChoice(id: '', name: '', extraPrice: 0),
        );
        extra += choice.extraPrice;
      }
    }
    _extraCost = extra;
  }

  bool get _canAddToCart {
    for (final option in widget.item.options) {
      if (option.isRequired) {
        final selected = _selectedChoices[option.id];
        if (selected == null || selected.isEmpty) return false;
      }
    }
    return true;
  }

  double get _totalPrice =>
      (widget.item.price + _extraCost) * _quantity;

  void _addToCart() {
    final customizations = widget.item.options.map((option) {
      final selectedIds = _selectedChoices[option.id] ?? [];
      final selectedChoices = option.choices
          .where((c) => selectedIds.contains(c.id))
          .toList();
      return OrderItemCustomization(
        optionId: option.id,
        optionTitle: option.title,
        choiceIds: selectedIds,
        choiceNames: selectedChoices.map((c) => c.name).toList(),
        extraPrice: selectedChoices.fold(0, (s, c) => s + c.extraPrice),
      );
    }).toList();

    ref.read(cartNotifierProvider.notifier).addItem(
          CartItem(
            menuItem: widget.item,
            quantity: _quantity,
            customizations: customizations,
            specialInstructions: _instructionsCtrl.text.trim(),
          ),
        );

    Navigator.of(context).pop();
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text('เพิ่ม ${widget.item.name} x$_quantity ลงตะกร้าแล้ว'),
        backgroundColor: Colors.orange,
        duration: const Duration(seconds: 2),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return DraggableScrollableSheet(
      expand: false,
      initialChildSize: 0.85,
      minChildSize: 0.5,
      maxChildSize: 0.95,
      builder: (_, scrollCtrl) => SingleChildScrollView(
        controller: scrollCtrl,
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            _buildHandle(),
            _buildImage(),
            _buildItemInfo(),
            ...widget.item.options.map(_buildOptionSection),
            _buildSpecialInstructions(),
            _buildQuantityAndAdd(),
          ],
        ),
      ),
    );
  }

  Widget _buildHandle() => Center(
        child: Container(
          width: 40,
          height: 4,
          margin: const EdgeInsets.symmetric(vertical: 8),
          decoration: BoxDecoration(
            color: Colors.grey[300],
            borderRadius: BorderRadius.circular(2),
          ),
        ),
      );

  Widget _buildImage() => CachedNetworkImage(
        imageUrl: widget.item.imageUrl,
        height: 200,
        width: double.infinity,
        fit: BoxFit.cover,
      );

  Widget _buildItemInfo() => Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Expanded(
                  child: Text(
                    widget.item.name,
                    style: const TextStyle(
                        fontSize: 22, fontWeight: FontWeight.bold),
                  ),
                ),
                if (widget.item.isVegetarian)
                  const _Badge(label: 'มังสวิรัติ', color: Colors.green),
                if (widget.item.isSpicy)
                  const _Badge(label: '🌶 เผ็ด', color: Colors.red),
              ],
            ),
            const SizedBox(height: 8),
            Text(widget.item.description,
                style: TextStyle(color: Colors.grey[600])),
            const SizedBox(height: 8),
            Text(
              '฿${widget.item.price.toStringAsFixed(0)}',
              style: const TextStyle(
                  fontSize: 20,
                  fontWeight: FontWeight.bold,
                  color: Colors.orange),
            ),
          ],
        ),
      );

  Widget _buildOptionSection(MenuOption option) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Container(
          padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
          color: Colors.grey.shade100,
          child: Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              Text(
                option.title,
                style: const TextStyle(
                    fontWeight: FontWeight.bold, fontSize: 16),
              ),
              if (option.isRequired)
                Container(
                  padding: const EdgeInsets.symmetric(
                      horizontal: 8, vertical: 2),
                  decoration: BoxDecoration(
                    color: Colors.red.shade100,
                    borderRadius: BorderRadius.circular(8),
                  ),
                  child: const Text(
                    'จำเป็น',
                    style: TextStyle(color: Colors.red, fontSize: 12),
                  ),
                ),
            ],
          ),
        ),
        ...option.choices.map((choice) {
          final isSelected =
              (_selectedChoices[option.id] ?? []).contains(choice.id);
          return ListTile(
            title: Text(choice.name),
            trailing: Row(
              mainAxisSize: MainAxisSize.min,
              children: [
                if (choice.extraPrice > 0)
                  Text(
                    '+฿${choice.extraPrice.toStringAsFixed(0)}',
                    style: const TextStyle(color: Colors.orange),
                  ),
                const SizedBox(width: 8),
                option.isMultiSelect
                    ? Checkbox(
                        value: isSelected,
                        activeColor: Colors.orange,
                        onChanged: (_) =>
                            _toggleChoice(option, choice),
                      )
                    : Radio<String>(
                        value: choice.id,
                        groupValue: (_selectedChoices[option.id] ?? [])
                            .firstOrNull,
                        activeColor: Colors.orange,
                        onChanged: (_) =>
                            _toggleChoice(option, choice),
                      ),
              ],
            ),
            onTap: () => _toggleChoice(option, choice),
          );
        }),
      ],
    );
  }

  Widget _buildSpecialInstructions() => Padding(
        padding: const EdgeInsets.all(16),
        child: TextField(
          controller: _instructionsCtrl,
          maxLines: 2,
          decoration: const InputDecoration(
            labelText: 'คำแนะนำพิเศษ (ไม่บังคับ)',
            hintText: 'เช่น ไม่ใส่น้ำตาล, แยกซอส',
            border: OutlineInputBorder(),
          ),
        ),
      );

  Widget _buildQuantityAndAdd() => Container(
        padding: const EdgeInsets.all(16),
        child: Row(
          children: [
            Container(
              decoration: BoxDecoration(
                border: Border.all(color: Colors.grey.shade300),
                borderRadius: BorderRadius.circular(8),
              ),
              child: Row(
                children: [
                  IconButton(
                    icon: const Icon(Icons.remove),
                    onPressed: _quantity > 1
                        ? () => setState(() => _quantity--)
                        : null,
                  ),
                  Text(
                    '$_quantity',
                    style: const TextStyle(
                        fontSize: 18, fontWeight: FontWeight.bold),
                  ),
                  IconButton(
                    icon: const Icon(Icons.add),
                    onPressed: () => setState(() => _quantity++),
                  ),
                ],
              ),
            ),
            const SizedBox(width: 16),
            Expanded(
              child: ElevatedButton(
                onPressed: _canAddToCart ? _addToCart : null,
                style: ElevatedButton.styleFrom(
                  backgroundColor: Colors.orange,
                  foregroundColor: Colors.white,
                  minimumSize: const Size.fromHeight(52),
                  shape: RoundedRectangleBorder(
                    borderRadius: BorderRadius.circular(12),
                  ),
                ),
                child: Text(
                  'เพิ่มลงตะกร้า • ฿${_totalPrice.toStringAsFixed(0)}',
                  style: const TextStyle(fontSize: 16),
                ),
              ),
            ),
          ],
        ),
      );
}

class _Badge extends StatelessWidget {
  final String label;
  final Color color;
  const _Badge({required this.label, required this.color});

  @override
  Widget build(BuildContext context) {
    return Container(
      margin: const EdgeInsets.only(left: 4),
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
      decoration: BoxDecoration(
        color: color.withOpacity(0.15),
        borderRadius: BorderRadius.circular(8),
      ),
      child:
          Text(label, style: TextStyle(color: color, fontSize: 12)),
    );
  }
}
```

---

## ขั้นตอนที่ 3607: Reviews & Rating

```dart
// lib/features/restaurant/data/models/review_model.dart

class ReviewModel {
  final String id;
  final String userId;
  final String userDisplayName;
  final String? userPhotoUrl;
  final String restaurantId;
  final String orderId;
  final double rating;
  final String comment;
  final List<String> images;
  final DateTime createdAt;

  const ReviewModel({
    required this.id,
    required this.userId,
    required this.userDisplayName,
    this.userPhotoUrl,
    required this.restaurantId,
    required this.orderId,
    required this.rating,
    required this.comment,
    this.images = const [],
    required this.createdAt,
  });

  factory ReviewModel.fromFirestore(Map<String, dynamic> data, String docId) {
    return ReviewModel(
      id: docId,
      userId: data['userId'] as String,
      userDisplayName: data['userDisplayName'] as String,
      userPhotoUrl: data['userPhotoUrl'] as String?,
      restaurantId: data['restaurantId'] as String,
      orderId: data['orderId'] as String,
      rating: (data['rating'] as num).toDouble(),
      comment: data['comment'] as String? ?? '',
      images: List<String>.from(data['images'] as List? ?? []),
      createdAt: (data['createdAt'] as dynamic).toDate() as DateTime,
    );
  }

  Map<String, dynamic> toFirestore() => {
        'userId': userId,
        'userDisplayName': userDisplayName,
        'userPhotoUrl': userPhotoUrl,
        'restaurantId': restaurantId,
        'orderId': orderId,
        'rating': rating,
        'comment': comment,
        'images': images,
        'createdAt': createdAt,
      };
}

// Review widget
class ReviewTile extends StatelessWidget {
  final ReviewModel review;
  const ReviewTile({super.key, required this.review});

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: CircleAvatar(
        backgroundImage: review.userPhotoUrl != null
            ? CachedNetworkImageProvider(review.userPhotoUrl!)
            : null,
        child: review.userPhotoUrl == null
            ? Text(review.userDisplayName[0].toUpperCase())
            : null,
      ),
      title: Row(
        children: [
          Text(review.userDisplayName,
              style: const TextStyle(fontWeight: FontWeight.bold)),
          const Spacer(),
          ...List.generate(
              5,
              (i) => Icon(
                    Icons.star,
                    size: 14,
                    color: i < review.rating
                        ? Colors.amber
                        : Colors.grey.shade300,
                  )),
        ],
      ),
      subtitle: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          if (review.comment.isNotEmpty)
            Text(review.comment),
          Text(
            _formatDate(review.createdAt),
            style: TextStyle(color: Colors.grey[500], fontSize: 12),
          ),
        ],
      ),
      isThreeLine: true,
    );
  }

  String _formatDate(DateTime date) {
    return '${date.day}/${date.month}/${date.year}';
  }
}
```

---

**← [Part 92](part-92-capstone-auth-module.md)**
**ต่อไป: [Part 94 →](part-94-capstone-cart-checkout.md)**
