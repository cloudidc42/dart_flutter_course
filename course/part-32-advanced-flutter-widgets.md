# Part 32: Advanced Flutter Widgets - Slivers & CustomScrollView
## ขั้นตอนที่ 1161-1200

---

## 🎯 เป้าหมายของ Part นี้

- CustomScrollView กับ Sliver widgets
- SliverAppBar, SliverList, SliverGrid
- SliverPersistentHeader
- NestedScrollView
- Infinite scrolling กับ Slivers

---

## ขั้นตอนที่ 1161: Sliver Basics

```dart
import 'package:flutter/material.dart';

// ─── CustomScrollView พื้นฐาน ───
class SliverBasicDemo extends StatelessWidget {
  const SliverBasicDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          // SliverAppBar: collapse/expand ได้
          SliverAppBar(
            expandedHeight: 200,
            floating: false,
            pinned: true,
            flexibleSpace: FlexibleSpaceBar(
              title: const Text('Slivers Demo'),
              background: Image.network(
                'https://picsum.photos/400/200',
                fit: BoxFit.cover,
              ),
            ),
          ),

          // SliverToBoxAdapter: ใส่ widget ปกติ
          const SliverToBoxAdapter(
            child: Padding(
              padding: EdgeInsets.all(16),
              child: Text(
                'Featured Section',
                style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
              ),
            ),
          ),

          // SliverGrid
          SliverGrid(
            gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
              crossAxisCount: 2,
              crossAxisSpacing: 8,
              mainAxisSpacing: 8,
              childAspectRatio: 1.5,
            ),
            delegate: SliverChildBuilderDelegate(
              (context, index) => Card(
                color: Colors.primaries[index % Colors.primaries.length],
                child: Center(
                  child: Text(
                    'Item $index',
                    style: const TextStyle(color: Colors.white),
                  ),
                ),
              ),
              childCount: 6,
            ),
          ),

          // SliverToBoxAdapter: Header
          const SliverToBoxAdapter(
            child: Padding(
              padding: EdgeInsets.all(16),
              child: Text(
                'All Items',
                style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
              ),
            ),
          ),

          // SliverList
          SliverList(
            delegate: SliverChildBuilderDelegate(
              (context, index) => ListTile(
                leading: CircleAvatar(child: Text('$index')),
                title: Text('Item $index'),
                subtitle: Text('Subtitle $index'),
                trailing: const Icon(Icons.arrow_forward_ios, size: 16),
              ),
              childCount: 20,
            ),
          ),

          // Bottom padding
          const SliverPadding(padding: EdgeInsets.only(bottom: 80)),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1162: SliverPersistentHeader

```dart
// ─── Custom Persistent Header ───
class StickyHeaderDelegate extends SliverPersistentHeaderDelegate {
  final String title;
  final double height;
  final Color color;

  const StickyHeaderDelegate({
    required this.title,
    this.height = 48,
    this.color = Colors.blue,
  });

  @override
  Widget build(BuildContext context, double shrinkOffset, bool overlapsContent) {
    double opacity = 1.0 - shrinkOffset / maxExtent;
    return Container(
      color: color,
      child: Stack(
        fit: StackFit.expand,
        children: [
          // Background opacity เมื่อ scroll
          Opacity(
            opacity: opacity.clamp(0.0, 1.0),
            child: Container(color: color.withOpacity(0.3)),
          ),
          Align(
            alignment: Alignment.centerLeft,
            child: Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16),
              child: Text(
                title,
                style: const TextStyle(
                  color: Colors.white,
                  fontSize: 18,
                  fontWeight: FontWeight.bold,
                ),
              ),
            ),
          ),
        ],
      ),
    );
  }

  @override
  double get maxExtent => height;

  @override
  double get minExtent => height;

  @override
  bool shouldRebuild(StickyHeaderDelegate oldDelegate) =>
      title != oldDelegate.title || height != oldDelegate.height;
}

// ─── ใช้งาน Sticky Headers ───
class StickyHeadersDemo extends StatelessWidget {
  const StickyHeadersDemo({super.key});

  static const List<MapEntry<String, List<String>>> data = [
    MapEntry('Fruits', ['Apple', 'Banana', 'Cherry', 'Date']),
    MapEntry('Vegetables', ['Broccoli', 'Carrot', 'Daikon', 'Eggplant']),
    MapEntry('Proteins', ['Fish', 'Beef', 'Chicken', 'Tofu']),
    MapEntry('Grains', ['Rice', 'Wheat', 'Oat', 'Quinoa']),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Sticky Headers')),
      body: CustomScrollView(
        slivers: [
          for (var (i, entry) in data.indexed) ...[
            SliverPersistentHeader(
              pinned: true,
              delegate: StickyHeaderDelegate(
                title: entry.key,
                color: Colors.primaries[i * 2 % Colors.primaries.length],
              ),
            ),
            SliverList(
              delegate: SliverChildBuilderDelegate(
                (context, j) => ListTile(
                  title: Text(entry.value[j]),
                ),
                childCount: entry.value.length,
              ),
            ),
          ],
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1163: NestedScrollView

```dart
// ─── NestedScrollView กับ TabBar ───
class NestedScrollViewDemo extends StatelessWidget {
  const NestedScrollViewDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return DefaultTabController(
      length: 3,
      child: Scaffold(
        body: NestedScrollView(
          headerSliverBuilder: (context, innerBoxIsScrolled) => [
            SliverOverlapAbsorber(
              handle: NestedScrollView.sliverOverlapAbsorberHandleFor(context),
              sliver: SliverAppBar(
                title: const Text('Profile'),
                expandedHeight: 250,
                pinned: true,
                forceElevated: innerBoxIsScrolled,
                flexibleSpace: FlexibleSpaceBar(
                  background: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      const SizedBox(height: 60),
                      const CircleAvatar(
                        radius: 50,
                        backgroundImage: NetworkImage('https://picsum.photos/100'),
                      ),
                      const SizedBox(height: 12),
                      const Text(
                        'John Doe',
                        style: TextStyle(
                          color: Colors.white,
                          fontSize: 24,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      Text(
                        '@johndoe',
                        style: TextStyle(color: Colors.white.withOpacity(0.8)),
                      ),
                    ],
                  ),
                ),
                bottom: const TabBar(
                  tabs: [
                    Tab(text: 'Posts'),
                    Tab(text: 'Photos'),
                    Tab(text: 'Likes'),
                  ],
                ),
              ),
            ),
          ],
          body: TabBarView(
            children: [
              _buildPostsList(),
              _buildPhotosGrid(),
              _buildLikesList(),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildPostsList() {
    return Builder(
      builder: (context) => CustomScrollView(
        slivers: [
          SliverOverlapInjector(
            handle: NestedScrollView.sliverOverlapAbsorberHandleFor(context),
          ),
          SliverList(
            delegate: SliverChildBuilderDelegate(
              (context, index) => Card(
                margin: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
                child: Padding(
                  padding: const EdgeInsets.all(12),
                  child: Text('Post #${index + 1}: Lorem ipsum...'),
                ),
              ),
              childCount: 20,
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildPhotosGrid() {
    return Builder(
      builder: (context) => CustomScrollView(
        slivers: [
          SliverOverlapInjector(
            handle: NestedScrollView.sliverOverlapAbsorberHandleFor(context),
          ),
          SliverGrid(
            gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
              crossAxisCount: 3,
              crossAxisSpacing: 2,
              mainAxisSpacing: 2,
            ),
            delegate: SliverChildBuilderDelegate(
              (context, index) => Image.network(
                'https://picsum.photos/100?random=$index',
                fit: BoxFit.cover,
              ),
              childCount: 30,
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildLikesList() {
    return Builder(
      builder: (context) => CustomScrollView(
        slivers: [
          SliverOverlapInjector(
            handle: NestedScrollView.sliverOverlapAbsorberHandleFor(context),
          ),
          SliverList(
            delegate: SliverChildBuilderDelegate(
              (context, index) => ListTile(
                leading: CircleAvatar(
                  backgroundImage: NetworkImage('https://picsum.photos/50?random=$index'),
                ),
                title: Text('Liked item $index'),
                trailing: const Icon(Icons.favorite, color: Colors.red),
              ),
              childCount: 15,
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1164: Infinite Scroll กับ Slivers

```dart
import 'package:flutter/material.dart';

class InfiniteScrollPage extends StatefulWidget {
  const InfiniteScrollPage({super.key});

  @override
  State<InfiniteScrollPage> createState() => _InfiniteScrollPageState();
}

class _InfiniteScrollPageState extends State<InfiniteScrollPage> {
  final List<String> _items = [];
  bool _isLoading = false;
  bool _hasMore = true;
  int _page = 1;
  final ScrollController _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    _loadMore();
    _scrollController.addListener(_onScroll);
  }

  void _onScroll() {
    if (_scrollController.position.pixels >=
        _scrollController.position.maxScrollExtent - 200) {
      _loadMore();
    }
  }

  Future<void> _loadMore() async {
    if (_isLoading || !_hasMore) return;

    setState(() => _isLoading = true);

    // Simulate API call
    await Future.delayed(const Duration(milliseconds: 800));

    List<String> newItems = List.generate(
      20,
      (i) => 'Item ${(_page - 1) * 20 + i + 1}',
    );

    setState(() {
      _items.addAll(newItems);
      _page++;
      _isLoading = false;
      _hasMore = _page <= 5; // หยุดที่ 100 items
    });
  }

  Future<void> _refresh() async {
    setState(() {
      _items.clear();
      _page = 1;
      _hasMore = true;
    });
    await _loadMore();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: RefreshIndicator(
        onRefresh: _refresh,
        child: CustomScrollView(
          controller: _scrollController,
          slivers: [
            SliverAppBar(
              title: const Text('Infinite Scroll'),
              floating: true,
              snap: true,
            ),
            SliverList(
              delegate: SliverChildBuilderDelegate(
                (context, index) {
                  if (index == _items.length) {
                    return _isLoading
                        ? const Center(
                            child: Padding(
                              padding: EdgeInsets.all(16),
                              child: CircularProgressIndicator(),
                            ),
                          )
                        : !_hasMore
                            ? const Center(
                                child: Padding(
                                  padding: EdgeInsets.all(16),
                                  child: Text('ไม่มีข้อมูลเพิ่มเติม'),
                                ),
                              )
                            : const SizedBox.shrink();
                  }

                  return ListTile(
                    leading: CircleAvatar(
                      child: Text('${index + 1}'),
                    ),
                    title: Text(_items[index]),
                    subtitle: Text('Page: ${(index ~/ 20) + 1}'),
                  );
                },
                childCount: _items.length + 1,
              ),
            ),
          ],
        ),
      ),
    );
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }
}
```

---

## ขั้นตอนที่ 1165: SliverFillRemaining & Advanced

```dart
// ─── SliverFillRemaining: เติมพื้นที่ที่เหลือ ───
class SliverFillDemo extends StatelessWidget {
  const SliverFillDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        slivers: [
          SliverAppBar(
            title: const Text('Fill Remaining'),
            expandedHeight: 200,
            pinned: true,
            flexibleSpace: FlexibleSpaceBar(
              background: Container(
                decoration: const BoxDecoration(
                  gradient: LinearGradient(
                    begin: Alignment.topLeft,
                    end: Alignment.bottomRight,
                    colors: [Colors.purple, Colors.blue],
                  ),
                ),
              ),
            ),
          ),
          SliverList(
            delegate: SliverChildBuilderDelegate(
              (context, index) => ListTile(
                title: Text('Item $index'),
              ),
              childCount: 3,
            ),
          ),
          // เติมพื้นที่ที่เหลือ
          SliverFillRemaining(
            hasScrollBody: false,
            child: Container(
              color: Colors.grey[100],
              child: const Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Icon(Icons.inbox, size: 64, color: Colors.grey),
                    SizedBox(height: 16),
                    Text('ไม่มีข้อมูลเพิ่มเติม', style: TextStyle(color: Colors.grey)),
                  ],
                ),
              ),
            ),
          ),
        ],
      ),
    );
  }
}

// ─── SliverVariedExtentList: items ต่างขนาด ───
class SliverVariedDemo extends StatelessWidget {
  const SliverVariedDemo({super.key});

  static final List<double> itemHeights = [60, 100, 80, 120, 60, 90, 110, 70];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Varied Extent')),
      body: CustomScrollView(
        slivers: [
          SliverVariedExtentList(
            itemExtentBuilder: (index, dimensions) => itemHeights[index % itemHeights.length],
            delegate: SliverChildBuilderDelegate(
              (context, index) {
                double height = itemHeights[index % itemHeights.length];
                return Container(
                  color: Colors.primaries[index % Colors.primaries.length].withOpacity(0.3),
                  child: Center(
                    child: Text('Item $index (h=${height.toInt()}px)'),
                  ),
                );
              },
              childCount: 30,
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## pubspec.yaml สำหรับ Part นี้

```yaml
dependencies:
  flutter:
    sdk: flutter
  # ไม่ต้องการ dependencies พิเศษ - ใช้ Flutter built-in
```

---

**← [Part 31 - Dart 3.0 Records & Patterns](part-31-dart3-records-patterns.md)**

**ต่อไป: [Part 33 - Flutter Web →](part-33-flutter-web.md)**
