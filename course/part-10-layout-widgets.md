# Part 10: Layout Widgets ใน Flutter
## ขั้นตอนที่ 281-320

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ Row, Column, Stack สำหรับจัด Layout
- เข้าใจ Flexible, Expanded, Spacer
- ใช้ ListView, GridView, SingleChildScrollView
- เข้าใจ Padding, Margin, Alignment
- ใช้ Wrap, Flow สำหรับ responsive layout
- สร้าง Complex layouts ด้วยการรวม widgets

---

## ขั้นตอนที่ 281: Row และ Column

```dart
import 'package:flutter/material.dart';

class RowColumnExample extends StatelessWidget {
  const RowColumnExample({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Row & Column')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text('Row - MainAxis: spaceAround', style: TextStyle(fontWeight: FontWeight.bold)),
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceAround,
              children: [
                _ColorBox(color: Colors.red, label: 'R'),
                _ColorBox(color: Colors.green, label: 'G'),
                _ColorBox(color: Colors.blue, label: 'B'),
              ],
            ),
            const SizedBox(height: 16),
            
            const Text('Row - MainAxis: spaceBetween', style: TextStyle(fontWeight: FontWeight.bold)),
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                _ColorBox(color: Colors.orange, label: '1'),
                _ColorBox(color: Colors.purple, label: '2'),
                _ColorBox(color: Colors.teal, label: '3'),
              ],
            ),
            const SizedBox(height: 16),
            
            const Text('Column - CrossAxis: center', style: TextStyle(fontWeight: FontWeight.bold)),
            Container(
              width: double.infinity,
              height: 150,
              color: Colors.grey[100],
              child: Column(
                mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                crossAxisAlignment: CrossAxisAlignment.center,
                children: [
                  Container(width: 200, height: 30, color: Colors.red.withOpacity(0.5)),
                  Container(width: 150, height: 30, color: Colors.green.withOpacity(0.5)),
                  Container(width: 100, height: 30, color: Colors.blue.withOpacity(0.5)),
                ],
              ),
            ),
            const SizedBox(height: 16),
            
            const Text('CrossAxis: stretch', style: TextStyle(fontWeight: FontWeight.bold)),
            Container(
              height: 100,
              color: Colors.grey[100],
              child: Row(
                crossAxisAlignment: CrossAxisAlignment.stretch,
                children: [
                  Container(width: 80, color: Colors.red.withOpacity(0.5), child: const Center(child: Text('R'))),
                  Container(width: 80, color: Colors.green.withOpacity(0.5), child: const Center(child: Text('G'))),
                  Container(width: 80, color: Colors.blue.withOpacity(0.5), child: const Center(child: Text('B'))),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _ColorBox extends StatelessWidget {
  final Color color;
  final String label;
  
  const _ColorBox({required this.color, required this.label});
  
  @override
  Widget build(BuildContext context) {
    return Container(
      width: 60,
      height: 60,
      color: color.withOpacity(0.7),
      child: Center(
        child: Text(label, style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 282: Flexible และ Expanded

```dart
import 'package:flutter/material.dart';

class FlexibleExpandedExample extends StatelessWidget {
  const FlexibleExpandedExample({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Flexible & Expanded')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Expanded: ใช้พื้นที่ที่เหลือทั้งหมด
            const Text('Expanded (1:1:1)', style: TextStyle(fontWeight: FontWeight.bold)),
            SizedBox(
              height: 60,
              child: Row(
                children: [
                  Expanded(child: Container(color: Colors.red.withOpacity(0.7), child: const Center(child: Text('1')))),
                  Expanded(child: Container(color: Colors.green.withOpacity(0.7), child: const Center(child: Text('1')))),
                  Expanded(child: Container(color: Colors.blue.withOpacity(0.7), child: const Center(child: Text('1')))),
                ],
              ),
            ),
            const SizedBox(height: 16),
            
            // Expanded กับ flex factor
            const Text('Expanded (1:2:1)', style: TextStyle(fontWeight: FontWeight.bold)),
            SizedBox(
              height: 60,
              child: Row(
                children: [
                  Expanded(flex: 1, child: Container(color: Colors.red.withOpacity(0.7), child: const Center(child: Text('1')))),
                  Expanded(flex: 2, child: Container(color: Colors.green.withOpacity(0.7), child: const Center(child: Text('2')))),
                  Expanded(flex: 1, child: Container(color: Colors.blue.withOpacity(0.7), child: const Center(child: Text('1')))),
                ],
              ),
            ),
            const SizedBox(height: 16),
            
            // Flexible: ใช้พื้นที่ไม่เกินที่กำหนด (shrink ได้)
            const Text('Flexible', style: TextStyle(fontWeight: FontWeight.bold)),
            Row(
              children: [
                const FlutterLogo(size: 48),
                const SizedBox(width: 8),
                Flexible(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      const Text('Title', style: TextStyle(fontWeight: FontWeight.bold)),
                      Text(
                        'ข้อความที่ยาวมากและสามารถ wrap ได้ เพราะใช้ Flexible',
                        style: TextStyle(color: Colors.grey[600]),
                      ),
                    ],
                  ),
                ),
              ],
            ),
            const SizedBox(height: 16),
            
            // Spacer
            const Text('Spacer', style: TextStyle(fontWeight: FontWeight.bold)),
            Container(
              color: Colors.grey[100],
              child: Row(
                children: [
                  Container(width: 60, height: 40, color: Colors.red.withOpacity(0.7), child: const Center(child: Text('Left'))),
                  const Spacer(),
                  Container(width: 60, height: 40, color: Colors.blue.withOpacity(0.7), child: const Center(child: Text('Right'))),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 283: Stack

```dart
import 'package:flutter/material.dart';

class StackExample extends StatelessWidget {
  const StackExample({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Stack')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Basic Stack
            const Text('Basic Stack', style: TextStyle(fontWeight: FontWeight.bold)),
            SizedBox(
              height: 150,
              child: Stack(
                children: [
                  Container(color: Colors.blue.withOpacity(0.7), width: 150, height: 150),
                  Positioned(
                    top: 20,
                    left: 20,
                    child: Container(color: Colors.green.withOpacity(0.7), width: 100, height: 100),
                  ),
                  Positioned(
                    top: 40,
                    left: 40,
                    child: Container(color: Colors.red.withOpacity(0.7), width: 50, height: 50),
                  ),
                ],
              ),
            ),
            const SizedBox(height: 16),
            
            // Profile card with Stack
            const Text('Profile Card', style: TextStyle(fontWeight: FontWeight.bold)),
            SizedBox(
              height: 200,
              child: Stack(
                children: [
                  // Background
                  Positioned.fill(
                    child: Container(
                      decoration: BoxDecoration(
                        gradient: const LinearGradient(
                          colors: [Colors.deepPurple, Colors.purple],
                          begin: Alignment.topLeft,
                          end: Alignment.bottomRight,
                        ),
                        borderRadius: BorderRadius.circular(16),
                      ),
                    ),
                  ),
                  
                  // Decorative circle
                  Positioned(
                    top: -30,
                    right: -30,
                    child: Container(
                      width: 150,
                      height: 150,
                      decoration: BoxDecoration(
                        color: Colors.white.withOpacity(0.1),
                        shape: BoxShape.circle,
                      ),
                    ),
                  ),
                  
                  // Content
                  Padding(
                    padding: const EdgeInsets.all(20),
                    child: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        const CircleAvatar(
                          radius: 30,
                          backgroundColor: Colors.white24,
                          child: Icon(Icons.person, color: Colors.white, size: 36),
                        ),
                        const SizedBox(height: 12),
                        const Text(
                          'Alice Developer',
                          style: TextStyle(color: Colors.white, fontSize: 20, fontWeight: FontWeight.bold),
                        ),
                        Text(
                          'Flutter Engineer',
                          style: TextStyle(color: Colors.white.withOpacity(0.8)),
                        ),
                      ],
                    ),
                  ),
                  
                  // Badge
                  Positioned(
                    top: 12,
                    right: 12,
                    child: Container(
                      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
                      decoration: BoxDecoration(
                        color: Colors.green,
                        borderRadius: BorderRadius.circular(12),
                      ),
                      child: const Text('Online', style: TextStyle(color: Colors.white, fontSize: 12)),
                    ),
                  ),
                ],
              ),
            ),
            const SizedBox(height: 16),
            
            // Image with overlay
            const Text('Image Overlay', style: TextStyle(fontWeight: FontWeight.bold)),
            ClipRRect(
              borderRadius: BorderRadius.circular(12),
              child: SizedBox(
                height: 200,
                child: Stack(
                  fit: StackFit.expand,
                  children: [
                    Image.network(
                      'https://via.placeholder.com/400x200',
                      fit: BoxFit.cover,
                    ),
                    DecoratedBox(
                      decoration: BoxDecoration(
                        gradient: LinearGradient(
                          colors: [Colors.transparent, Colors.black.withOpacity(0.8)],
                          begin: Alignment.topCenter,
                          end: Alignment.bottomCenter,
                        ),
                      ),
                    ),
                    const Positioned(
                      bottom: 16,
                      left: 16,
                      right: 16,
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Text('หัวข้อบทความ', style: TextStyle(color: Colors.white, fontSize: 18, fontWeight: FontWeight.bold)),
                          Text('21 ก.ย. 2024', style: TextStyle(color: Colors.white70, fontSize: 12)),
                        ],
                      ),
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
```

---

## ขั้นตอนที่ 284: ListView

```dart
import 'package:flutter/material.dart';

class ListViewExamples extends StatelessWidget {
  const ListViewExamples({super.key});
  
  @override
  Widget build(BuildContext context) {
    return DefaultTabController(
      length: 3,
      child: Scaffold(
        appBar: AppBar(
          title: const Text('ListView'),
          bottom: const TabBar(
            tabs: [
              Tab(text: 'Basic'),
              Tab(text: 'Builder'),
              Tab(text: 'Separated'),
            ],
          ),
        ),
        body: TabBarView(
          children: [
            // ListView basic
            ListView(
              padding: const EdgeInsets.all(8),
              children: List.generate(
                5,
                (i) => ListTile(
                  leading: CircleAvatar(child: Text('${i + 1}')),
                  title: Text('Item ${i + 1}'),
                  subtitle: Text('Subtitle ${i + 1}'),
                  trailing: const Icon(Icons.arrow_forward_ios, size: 14),
                  onTap: () {},
                ),
              ),
            ),
            
            // ListView.builder - สำหรับ list ยาว (lazy load)
            ListView.builder(
              itemCount: 100,
              itemBuilder: (context, index) {
                return ListTile(
                  leading: Icon(
                    Icons.circle,
                    color: HSLColor.fromAHSL(1, index * 3.6, 0.7, 0.5).toColor(),
                  ),
                  title: Text('Item $index'),
                  subtitle: Text('Description for item $index'),
                );
              },
            ),
            
            // ListView.separated - มี divider
            ListView.separated(
              padding: const EdgeInsets.all(8),
              itemCount: 20,
              separatorBuilder: (context, index) => const Divider(),
              itemBuilder: (context, index) {
                return ListTile(
                  leading: const Icon(Icons.book),
                  title: Text('Book ${index + 1}'),
                  subtitle: Text('Author ${index + 1}'),
                  trailing: const Icon(Icons.bookmark_border),
                );
              },
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 285: GridView

```dart
import 'package:flutter/material.dart';

class GridViewExamples extends StatelessWidget {
  const GridViewExamples({super.key});
  
  @override
  Widget build(BuildContext context) {
    return DefaultTabController(
      length: 2,
      child: Scaffold(
        appBar: AppBar(
          title: const Text('GridView'),
          bottom: const TabBar(
            tabs: [
              Tab(text: 'Fixed Count'),
              Tab(text: 'Extent'),
            ],
          ),
        ),
        body: TabBarView(
          children: [
            // GridView.count - กำหนดจำนวนคอลัมน์
            GridView.count(
              crossAxisCount: 3,
              padding: const EdgeInsets.all(8),
              crossAxisSpacing: 8,
              mainAxisSpacing: 8,
              children: List.generate(30, (index) {
                return Card(
                  color: HSLColor.fromAHSL(1, index * 12.0, 0.6, 0.7).toColor(),
                  child: Center(
                    child: Text(
                      '${index + 1}',
                      style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold),
                    ),
                  ),
                );
              }),
            ),
            
            // GridView.extent - กำหนดขนาดสูงสุดของ item
            GridView.extent(
              maxCrossAxisExtent: 180,
              padding: const EdgeInsets.all(8),
              crossAxisSpacing: 8,
              mainAxisSpacing: 8,
              childAspectRatio: 1.2,
              children: List.generate(20, (index) {
                return Card(
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      Icon(
                        Icons.category,
                        size: 48,
                        color: HSLColor.fromAHSL(1, index * 18.0, 0.6, 0.5).toColor(),
                      ),
                      const SizedBox(height: 4),
                      Text('Category ${index + 1}', style: const TextStyle(fontSize: 12)),
                    ],
                  ),
                );
              }),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 286: Padding, Margin, Alignment

```dart
import 'package:flutter/material.dart';

class SpacingAlignmentExample extends StatelessWidget {
  const SpacingAlignmentExample({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Padding, Margin, Alignment')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // EdgeInsets
            const Text('EdgeInsets', style: TextStyle(fontWeight: FontWeight.bold)),
            Container(
              color: Colors.grey[200],
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Container(color: Colors.blue, height: 40),
              ),
            ),
            const SizedBox(height: 8),
            Container(
              color: Colors.grey[200],
              child: Padding(
                padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 8),
                child: Container(color: Colors.green, height: 40),
              ),
            ),
            const SizedBox(height: 8),
            Container(
              color: Colors.grey[200],
              child: Padding(
                padding: const EdgeInsets.only(top: 4, left: 8, right: 16, bottom: 4),
                child: Container(color: Colors.orange, height: 40),
              ),
            ),
            const SizedBox(height: 16),
            
            // Alignment
            const Text('Alignment', style: TextStyle(fontWeight: FontWeight.bold)),
            Container(
              height: 100,
              color: Colors.grey[100],
              child: Align(
                alignment: Alignment.topRight,
                child: Container(width: 60, height: 30, color: Colors.red, child: const Center(child: Text('TR'))),
              ),
            ),
            const SizedBox(height: 8),
            Container(
              height: 100,
              color: Colors.grey[100],
              child: const Center(
                child: Text('Center', style: TextStyle(fontWeight: FontWeight.bold)),
              ),
            ),
            const SizedBox(height: 16),
            
            // FractionallySizedBox
            const Text('FractionallySizedBox', style: TextStyle(fontWeight: FontWeight.bold)),
            Container(
              height: 60,
              color: Colors.grey[200],
              child: FractionallySizedBox(
                widthFactor: 0.5,  // 50% ของ parent
                heightFactor: 0.8,
                alignment: Alignment.centerLeft,
                child: Container(color: Colors.blue, child: const Center(child: Text('50% width'))),
              ),
            ),
            const SizedBox(height: 16),
            
            // SizedBox สำหรับ fixed size
            const Text('SizedBox', style: TextStyle(fontWeight: FontWeight.bold)),
            Row(
              children: [
                SizedBox(
                  width: 80,
                  height: 80,
                  child: Container(color: Colors.purple, child: const Center(child: Text('80x80'))),
                ),
                const SizedBox(width: 8),  // spacing
                const Expanded(
                  child: SizedBox(
                    height: 80,
                    child: Placeholder(),
                  ),
                ),
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

## ขั้นตอนที่ 287: Wrap และ Flow

```dart
import 'package:flutter/material.dart';

class WrapExample extends StatelessWidget {
  const WrapExample({super.key});
  
  final List<String> tags = const [
    'Flutter', 'Dart', 'Mobile', 'Web', 'Desktop',
    'iOS', 'Android', 'Firebase', 'Riverpod', 'BLoC',
    'GetX', 'Provider', 'Animation', 'Testing',
  ];
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Wrap')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Chip tags with Wrap
            const Text('Tags', style: TextStyle(fontWeight: FontWeight.bold)),
            Wrap(
              spacing: 8,
              runSpacing: 8,
              children: tags.map((tag) {
                return Chip(
                  label: Text(tag),
                  backgroundColor: Colors.blue.withOpacity(0.1),
                  side: BorderSide(color: Colors.blue.withOpacity(0.3)),
                );
              }).toList(),
            ),
            const SizedBox(height: 16),
            
            // Filter chips
            const Text('Categories', style: TextStyle(fontWeight: FontWeight.bold)),
            _FilterChipGroup(
              options: const ['ทั้งหมด', 'Frontend', 'Backend', 'Mobile', 'DevOps'],
            ),
            const SizedBox(height: 16),
            
            // Icon wrap
            const Text('Icons', style: TextStyle(fontWeight: FontWeight.bold)),
            Wrap(
              spacing: 12,
              runSpacing: 12,
              children: [
                Icons.home, Icons.work, Icons.school, Icons.sports,
                Icons.music_note, Icons.camera, Icons.code, Icons.travel_explore,
              ].map((icon) => Container(
                width: 48,
                height: 48,
                decoration: BoxDecoration(
                  color: Colors.blue.withOpacity(0.1),
                  borderRadius: BorderRadius.circular(8),
                ),
                child: Icon(icon, color: Colors.blue),
              )).toList(),
            ),
          ],
        ),
      ),
    );
  }
}

class _FilterChipGroup extends StatefulWidget {
  final List<String> options;
  
  const _FilterChipGroup({required this.options});
  
  @override
  State<_FilterChipGroup> createState() => _FilterChipGroupState();
}

class _FilterChipGroupState extends State<_FilterChipGroup> {
  String _selected = 'ทั้งหมด';
  
  @override
  Widget build(BuildContext context) {
    return Wrap(
      spacing: 8,
      children: widget.options.map((option) {
        bool isSelected = _selected == option;
        return FilterChip(
          label: Text(option),
          selected: isSelected,
          onSelected: (_) => setState(() => _selected = option),
          selectedColor: Colors.blue.withOpacity(0.2),
        );
      }).toList(),
    );
  }
}
```

---

## ขั้นตอนที่ 288-310: โปรเจกต์ - News App Layout

```dart
// news_app.dart
import 'package:flutter/material.dart';

void main() {
  runApp(const NewsApp());
}

class NewsApp extends StatelessWidget {
  const NewsApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'News App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const NewsScreen(),
    );
  }
}

class Article {
  final String id;
  final String title;
  final String summary;
  final String category;
  final String imageUrl;
  final String author;
  final DateTime publishedAt;
  final int readMinutes;
  
  const Article({
    required this.id,
    required this.title,
    required this.summary,
    required this.category,
    required this.imageUrl,
    required this.author,
    required this.publishedAt,
    required this.readMinutes,
  });
}

final List<Article> articles = [
  Article(
    id: '1',
    title: 'Flutter 4.0 เปิดตัวแล้ว พร้อมฟีเจอร์ใหม่มากมาย',
    summary: 'Google ประกาศเปิดตัว Flutter 4.0 พร้อมการปรับปรุงประสิทธิภาพและฟีเจอร์ใหม่ที่นักพัฒนารอคอยมานาน',
    category: 'Technology',
    imageUrl: 'https://via.placeholder.com/800x400',
    author: 'Alice Dev',
    publishedAt: DateTime.now().subtract(const Duration(hours: 2)),
    readMinutes: 5,
  ),
  Article(
    id: '2',
    title: 'Dart 4 มาพร้อม Pattern Matching ที่ทรงพลังยิ่งขึ้น',
    summary: 'ภาษา Dart เวอร์ชัน 4 มาพร้อมกับการปรับปรุง type system และ pattern matching ที่ทำให้โค้ดอ่านง่ายขึ้น',
    category: 'Programming',
    imageUrl: 'https://via.placeholder.com/800x400',
    author: 'Bob Code',
    publishedAt: DateTime.now().subtract(const Duration(hours: 5)),
    readMinutes: 8,
  ),
  Article(
    id: '3',
    title: 'แนวทางการพัฒนา Mobile App ในปี 2025',
    summary: 'ผู้เชี่ยวชาญในวงการเทคโนโลยีแชร์แนวทางการพัฒนา Mobile Application สำหรับปี 2025',
    category: 'Career',
    imageUrl: 'https://via.placeholder.com/800x400',
    author: 'Charlie Pro',
    publishedAt: DateTime.now().subtract(const Duration(days: 1)),
    readMinutes: 10,
  ),
];

class NewsScreen extends StatefulWidget {
  const NewsScreen({super.key});
  
  @override
  State<NewsScreen> createState() => _NewsScreenState();
}

class _NewsScreenState extends State<NewsScreen> {
  int _selectedIndex = 0;
  String _selectedCategory = 'ทั้งหมด';
  
  final List<String> categories = ['ทั้งหมด', 'Technology', 'Programming', 'Career', 'Design'];
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('News'),
        actions: [
          IconButton(icon: const Icon(Icons.search), onPressed: () {}),
          IconButton(icon: const Icon(Icons.notifications_outlined), onPressed: () {}),
        ],
      ),
      body: CustomScrollView(
        slivers: [
          // Category Filter
          SliverToBoxAdapter(
            child: SizedBox(
              height: 48,
              child: ListView.builder(
                scrollDirection: Axis.horizontal,
                padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
                itemCount: categories.length,
                itemBuilder: (context, index) {
                  bool isSelected = _selectedCategory == categories[index];
                  return Padding(
                    padding: const EdgeInsets.only(right: 8),
                    child: FilterChip(
                      label: Text(categories[index]),
                      selected: isSelected,
                      onSelected: (_) => setState(() => _selectedCategory = categories[index]),
                    ),
                  );
                },
              ),
            ),
          ),
          
          // Featured Article
          SliverToBoxAdapter(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: FeaturedArticleCard(article: articles.first),
            ),
          ),
          
          // Section Header
          const SliverToBoxAdapter(
            child: Padding(
              padding: EdgeInsets.symmetric(horizontal: 16, vertical: 8),
              child: Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  Text('ล่าสุด', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
                  TextButton(onPressed: null, child: Text('ดูทั้งหมด')),
                ],
              ),
            ),
          ),
          
          // Article List
          SliverList(
            delegate: SliverChildBuilderDelegate(
              (context, index) {
                return Padding(
                  padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
                  child: ArticleCard(article: articles[index % articles.length]),
                );
              },
              childCount: 10,
            ),
          ),
          
          const SliverToBoxAdapter(child: SizedBox(height: 16)),
        ],
      ),
      bottomNavigationBar: NavigationBar(
        selectedIndex: _selectedIndex,
        onDestinationSelected: (index) => setState(() => _selectedIndex = index),
        destinations: const [
          NavigationDestination(icon: Icon(Icons.home_outlined), selectedIcon: Icon(Icons.home), label: 'หน้าแรก'),
          NavigationDestination(icon: Icon(Icons.explore_outlined), selectedIcon: Icon(Icons.explore), label: 'สำรวจ'),
          NavigationDestination(icon: Icon(Icons.bookmark_outline), selectedIcon: Icon(Icons.bookmark), label: 'บันทึก'),
          NavigationDestination(icon: Icon(Icons.person_outline), selectedIcon: Icon(Icons.person), label: 'โปรไฟล์'),
        ],
      ),
    );
  }
}

class FeaturedArticleCard extends StatelessWidget {
  final Article article;
  
  const FeaturedArticleCard({super.key, required this.article});
  
  @override
  Widget build(BuildContext context) {
    return ClipRRect(
      borderRadius: BorderRadius.circular(16),
      child: SizedBox(
        height: 220,
        child: Stack(
          fit: StackFit.expand,
          children: [
            Image.network(article.imageUrl, fit: BoxFit.cover,
              errorBuilder: (c, e, s) => Container(color: Colors.grey[300])),
            DecoratedBox(
              decoration: BoxDecoration(
                gradient: LinearGradient(
                  colors: [Colors.transparent, Colors.black.withOpacity(0.8)],
                  begin: Alignment.topCenter,
                  end: Alignment.bottomCenter,
                ),
              ),
            ),
            Positioned(
              top: 12,
              left: 12,
              child: Container(
                padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
                decoration: BoxDecoration(
                  color: Colors.indigo,
                  borderRadius: BorderRadius.circular(8),
                ),
                child: Text(article.category, style: const TextStyle(color: Colors.white, fontSize: 12)),
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
                    article.title,
                    style: const TextStyle(color: Colors.white, fontSize: 16, fontWeight: FontWeight.bold),
                    maxLines: 2,
                    overflow: TextOverflow.ellipsis,
                  ),
                  const SizedBox(height: 8),
                  Row(
                    children: [
                      Text(article.author, style: const TextStyle(color: Colors.white70, fontSize: 12)),
                      const Spacer(),
                      Icon(Icons.access_time, size: 12, color: Colors.white70),
                      const SizedBox(width: 4),
                      Text('${article.readMinutes} นาที', style: const TextStyle(color: Colors.white70, fontSize: 12)),
                    ],
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

class ArticleCard extends StatelessWidget {
  final Article article;
  
  const ArticleCard({super.key, required this.article});
  
  @override
  Widget build(BuildContext context) {
    return Card(
      child: InkWell(
        onTap: () {},
        borderRadius: BorderRadius.circular(12),
        child: Padding(
          padding: const EdgeInsets.all(12),
          child: Row(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              ClipRRect(
                borderRadius: BorderRadius.circular(8),
                child: Image.network(
                  article.imageUrl,
                  width: 80,
                  height: 80,
                  fit: BoxFit.cover,
                  errorBuilder: (c, e, s) => Container(
                    width: 80,
                    height: 80,
                    color: Colors.grey[300],
                    child: const Icon(Icons.image),
                  ),
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Container(
                      padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
                      decoration: BoxDecoration(
                        color: Colors.indigo.withOpacity(0.1),
                        borderRadius: BorderRadius.circular(4),
                      ),
                      child: Text(
                        article.category,
                        style: const TextStyle(color: Colors.indigo, fontSize: 11),
                      ),
                    ),
                    const SizedBox(height: 4),
                    Text(
                      article.title,
                      style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 14),
                      maxLines: 2,
                      overflow: TextOverflow.ellipsis,
                    ),
                    const SizedBox(height: 4),
                    Row(
                      children: [
                        Text(
                          article.author,
                          style: TextStyle(color: Colors.grey[600], fontSize: 11),
                        ),
                        const Spacer(),
                        Text(
                          '${article.readMinutes} นาที',
                          style: TextStyle(color: Colors.grey[600], fontSize: 11),
                        ),
                      ],
                    ),
                  ],
                ),
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

## ขั้นตอนที่ 311-320: สรุป Layout

### Layout Decision Guide

```
ต้องการจัด widgets ในแนวนอน?
  → Row

ต้องการจัด widgets ในแนวตั้ง?
  → Column

ต้องการ overlap widgets?
  → Stack + Positioned

ต้องการ scroll แนวตั้ง?
  → ListView / SingleChildScrollView

ต้องการ scroll แนวนอน?
  → ListView(scrollDirection: Axis.horizontal)

ต้องการ grid?
  → GridView.count / GridView.extent

ต้องการ wrap เมื่อล้น?
  → Wrap

ต้องการ responsive?
  → LayoutBuilder / MediaQuery
```

### LayoutBuilder สำหรับ Responsive

```dart
LayoutBuilder(
  builder: (context, constraints) {
    if (constraints.maxWidth > 600) {
      // Tablet/Desktop layout
      return Row(
        children: [
          SizedBox(width: 250, child: Sidebar()),
          Expanded(child: MainContent()),
        ],
      );
    } else {
      // Phone layout
      return MainContent();
    }
  },
)
```

---

**← [Part 09 - Flutter Widget เบื้องต้น](part-09-flutter-widgets-basics.md)**

**ต่อไป: [Part 11 - Async/Await และ Future →](part-11-async-await-futures.md)**
