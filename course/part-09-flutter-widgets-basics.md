# Part 09: Flutter Widget เบื้องต้น
## ขั้นตอนที่ 241-280

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Widget คืออะไร และ Widget Tree
- รู้จัก StatelessWidget และ StatefulWidget
- ใช้ MaterialApp และ Scaffold
- จัดการ Text, Image, Icon, Button
- เข้าใจ BuildContext
- ใช้ setState สำหรับ UI update

---

## ขั้นตอนที่ 241: Widget คืออะไร

```dart
// ใน Flutter ทุกอย่างคือ Widget
// Widget เป็น immutable description ของ UI
// เมื่อ state เปลี่ยน Flutter สร้าง widget tree ใหม่

import 'package:flutter/material.dart';

// Widget Tree ตัวอย่าง:
//
// MaterialApp
//  └── Scaffold
//       ├── AppBar
//       │    └── Text('Title')
//       └── Center
//            └── Column
//                 ├── Text('Hello')
//                 └── ElevatedButton
//                      └── Text('Click me')

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'My First App',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      home: const HomeScreen(),
    );
  }
}

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Home'),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
      ),
      body: const Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(
              'ยินดีต้อนรับ!',
              style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
            ),
            SizedBox(height: 16),
            Text('นี่คือ Flutter App แรกของเรา'),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 242: StatelessWidget

```dart
import 'package:flutter/material.dart';

// StatelessWidget: UI ที่ไม่เปลี่ยนแปลง (ไม่มี state)
class ProfileCard extends StatelessWidget {
  final String name;
  final String role;
  final String avatarUrl;
  final VoidCallback? onTap;
  
  const ProfileCard({
    super.key,
    required this.name,
    required this.role,
    required this.avatarUrl,
    this.onTap,
  });
  
  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      child: Card(
        elevation: 4,
        margin: const EdgeInsets.all(8),
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Row(
            children: [
              CircleAvatar(
                radius: 30,
                backgroundImage: NetworkImage(avatarUrl),
                backgroundColor: Colors.grey[300],
              ),
              const SizedBox(width: 16),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      name,
                      style: const TextStyle(
                        fontSize: 18,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                    const SizedBox(height: 4),
                    Text(
                      role,
                      style: TextStyle(
                        fontSize: 14,
                        color: Colors.grey[600],
                      ),
                    ),
                  ],
                ),
              ),
              const Icon(Icons.arrow_forward_ios, size: 16),
            ],
          ),
        ),
      ),
    );
  }
}

// Custom Widget พร้อม default values
class RatingWidget extends StatelessWidget {
  final double rating;
  final int maxStars;
  final double size;
  final Color activeColor;
  
  const RatingWidget({
    super.key,
    required this.rating,
    this.maxStars = 5,
    this.size = 24,
    this.activeColor = Colors.amber,
  });
  
  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: List.generate(maxStars, (index) {
        double starValue = index + 1;
        IconData icon;
        
        if (rating >= starValue) {
          icon = Icons.star;
        } else if (rating >= starValue - 0.5) {
          icon = Icons.star_half;
        } else {
          icon = Icons.star_border;
        }
        
        return Icon(icon, size: size, color: activeColor);
      }),
    );
  }
}

class ExampleScreen extends StatelessWidget {
  const ExampleScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('StatelessWidget')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          ProfileCard(
            name: 'Alice Developer',
            role: 'Flutter Engineer',
            avatarUrl: 'https://via.placeholder.com/60',
            onTap: () => ScaffoldMessenger.of(context).showSnackBar(
              const SnackBar(content: Text('Tapped Alice!')),
            ),
          ),
          const SizedBox(height: 16),
          const Center(
            child: Column(
              children: [
                Text('Rating: 4.5'),
                SizedBox(height: 8),
                RatingWidget(rating: 4.5),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 243: StatefulWidget

```dart
import 'package:flutter/material.dart';

// StatefulWidget: Widget ที่มี state ที่เปลี่ยนแปลงได้
class CounterWidget extends StatefulWidget {
  final int initialValue;
  final String label;
  
  const CounterWidget({
    super.key,
    this.initialValue = 0,
    this.label = 'Count',
  });
  
  @override
  State<CounterWidget> createState() => _CounterWidgetState();
}

class _CounterWidgetState extends State<CounterWidget> {
  late int _count;
  
  @override
  void initState() {
    super.initState();
    _count = widget.initialValue;  // widget ใช้ reference กลับไปที่ StatefulWidget
  }
  
  void _increment() {
    setState(() {
      _count++;
    });
  }
  
  void _decrement() {
    setState(() {
      if (_count > 0) _count--;
    });
  }
  
  void _reset() {
    setState(() {
      _count = widget.initialValue;
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.all(8),
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            Text(
              widget.label,
              style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            Text(
              '$_count',
              style: const TextStyle(fontSize: 48, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                IconButton(
                  onPressed: _decrement,
                  icon: const Icon(Icons.remove_circle),
                  iconSize: 32,
                  color: Colors.red,
                ),
                const SizedBox(width: 16),
                ElevatedButton(
                  onPressed: _reset,
                  child: const Text('Reset'),
                ),
                const SizedBox(width: 16),
                IconButton(
                  onPressed: _increment,
                  icon: const Icon(Icons.add_circle),
                  iconSize: 32,
                  color: Colors.green,
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
  
  @override
  void dispose() {
    // cleanup resources ที่นี่ (Controller, Stream subscription, etc.)
    super.dispose();
  }
}

// TextField พร้อม State
class SearchField extends StatefulWidget {
  final String hint;
  final void Function(String)? onSearch;
  
  const SearchField({
    super.key,
    this.hint = 'ค้นหา...',
    this.onSearch,
  });
  
  @override
  State<SearchField> createState() => _SearchFieldState();
}

class _SearchFieldState extends State<SearchField> {
  final TextEditingController _controller = TextEditingController();
  String _query = '';
  
  @override
  void initState() {
    super.initState();
    _controller.addListener(() {
      setState(() => _query = _controller.text);
    });
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        TextField(
          controller: _controller,
          decoration: InputDecoration(
            hintText: widget.hint,
            prefixIcon: const Icon(Icons.search),
            suffixIcon: _query.isNotEmpty
                ? IconButton(
                    icon: const Icon(Icons.clear),
                    onPressed: () {
                      _controller.clear();
                      widget.onSearch?.call('');
                    },
                  )
                : null,
            border: const OutlineInputBorder(),
          ),
          onSubmitted: widget.onSearch,
        ),
        if (_query.isNotEmpty)
          Padding(
            padding: const EdgeInsets.only(top: 8),
            child: Text(
              'กำลังค้นหา: "$_query"',
              style: TextStyle(color: Colors.grey[600]),
            ),
          ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 244: Text Widget

```dart
import 'package:flutter/material.dart';

class TextExamples extends StatelessWidget {
  const TextExamples({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Text Widget')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Basic Text
            const Text('ข้อความธรรมดา'),
            
            // Styled Text
            const Text(
              'ข้อความมี Style',
              style: TextStyle(
                fontSize: 20,
                fontWeight: FontWeight.bold,
                color: Colors.blue,
                letterSpacing: 1.5,
              ),
            ),
            
            // Text with overflow
            const Text(
              'ข้อความที่ยาวมากๆ และอาจล้น ต้องการการจัดการ overflow ที่เหมาะสม',
              maxLines: 2,
              overflow: TextOverflow.ellipsis,
            ),
            
            // RichText
            RichText(
              text: TextSpan(
                style: DefaultTextStyle.of(context).style,
                children: const [
                  TextSpan(text: 'ราคา: '),
                  TextSpan(
                    text: '฿ 1,299',
                    style: TextStyle(
                      color: Colors.red,
                      fontWeight: FontWeight.bold,
                      fontSize: 18,
                    ),
                  ),
                  TextSpan(text: ' (ลด 30%)'),
                ],
              ),
            ),
            
            // SelectableText
            const SelectableText(
              'ข้อความนี้สามารถเลือก copy ได้',
              style: TextStyle(fontStyle: FontStyle.italic),
            ),
            
            // Text Themes
            const SizedBox(height: 16),
            const Text('Display Large', style: TextStyle(fontSize: 36, fontWeight: FontWeight.w300)),
            const Text('Headline Medium', style: TextStyle(fontSize: 28)),
            const Text('Title Large', style: TextStyle(fontSize: 22, fontWeight: FontWeight.w500)),
            const Text('Body Large', style: TextStyle(fontSize: 16)),
            const Text('Label Small', style: TextStyle(fontSize: 11, letterSpacing: 0.5)),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 245: Button Widgets

```dart
import 'package:flutter/material.dart';

class ButtonExamples extends StatelessWidget {
  const ButtonExamples({super.key});
  
  void _showSnackBar(BuildContext context, String message) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(message)),
    );
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Button Widgets')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // ElevatedButton
            ElevatedButton(
              onPressed: () => _showSnackBar(context, 'ElevatedButton pressed'),
              child: const Text('ElevatedButton'),
            ),
            const SizedBox(height: 8),
            
            // ElevatedButton with icon
            ElevatedButton.icon(
              onPressed: () {},
              icon: const Icon(Icons.add),
              label: const Text('Add Item'),
            ),
            const SizedBox(height: 8),
            
            // OutlinedButton
            OutlinedButton(
              onPressed: () => _showSnackBar(context, 'OutlinedButton pressed'),
              child: const Text('OutlinedButton'),
            ),
            const SizedBox(height: 8),
            
            // TextButton
            TextButton(
              onPressed: () => _showSnackBar(context, 'TextButton pressed'),
              child: const Text('TextButton'),
            ),
            const SizedBox(height: 8),
            
            // FilledButton (Material 3)
            FilledButton(
              onPressed: () {},
              child: const Text('FilledButton'),
            ),
            const SizedBox(height: 8),
            
            // IconButton
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                IconButton(
                  onPressed: () {},
                  icon: const Icon(Icons.favorite_border),
                  tooltip: 'Favorite',
                ),
                IconButton.filled(
                  onPressed: () {},
                  icon: const Icon(Icons.share),
                ),
                IconButton.outlined(
                  onPressed: () {},
                  icon: const Icon(Icons.bookmark_border),
                ),
              ],
            ),
            const SizedBox(height: 8),
            
            // Disabled button
            ElevatedButton(
              onPressed: null,  // null = disabled
              child: const Text('Disabled Button'),
            ),
            const SizedBox(height: 8),
            
            // Custom styled button
            ElevatedButton(
              onPressed: () {},
              style: ElevatedButton.styleFrom(
                backgroundColor: Colors.green,
                foregroundColor: Colors.white,
                padding: const EdgeInsets.symmetric(vertical: 16),
                shape: RoundedRectangleBorder(
                  borderRadius: BorderRadius.circular(8),
                ),
              ),
              child: const Text('Custom Green Button', style: TextStyle(fontSize: 16)),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 246: Image และ Icon

```dart
import 'package:flutter/material.dart';

class ImageIconExamples extends StatelessWidget {
  const ImageIconExamples({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Image & Icon')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            // Network Image
            Image.network(
              'https://via.placeholder.com/200x100',
              width: 200,
              height: 100,
              fit: BoxFit.cover,
              loadingBuilder: (context, child, loadingProgress) {
                if (loadingProgress == null) return child;
                return Center(
                  child: CircularProgressIndicator(
                    value: loadingProgress.expectedTotalBytes != null
                        ? loadingProgress.cumulativeBytesLoaded /
                            loadingProgress.expectedTotalBytes!
                        : null,
                  ),
                );
              },
              errorBuilder: (context, error, stackTrace) {
                return Container(
                  width: 200,
                  height: 100,
                  color: Colors.grey[300],
                  child: const Icon(Icons.broken_image),
                );
              },
            ),
            const SizedBox(height: 16),
            
            // Asset Image (ต้องเพิ่มใน pubspec.yaml)
            // Image.asset('assets/images/logo.png'),
            
            // CircleAvatar
            const CircleAvatar(
              radius: 40,
              backgroundColor: Colors.blue,
              child: Text('AB', style: TextStyle(fontSize: 24, color: Colors.white)),
            ),
            const SizedBox(height: 16),
            
            // Icons
            Wrap(
              spacing: 8,
              runSpacing: 8,
              alignment: WrapAlignment.center,
              children: [
                Icons.home,
                Icons.person,
                Icons.settings,
                Icons.favorite,
                Icons.search,
                Icons.notifications,
              ].map((icon) => Icon(icon, size: 32, color: Colors.blue)).toList(),
            ),
            const SizedBox(height: 16),
            
            // Icon with color and size
            const Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                Icon(Icons.star, color: Colors.amber, size: 48),
                Icon(Icons.star, color: Colors.amber, size: 48),
                Icon(Icons.star, color: Colors.amber, size: 48),
                Icon(Icons.star_half, color: Colors.amber, size: 48),
                Icon(Icons.star_border, color: Colors.amber, size: 48),
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

## ขั้นตอนที่ 247: Container และ Decoration

```dart
import 'package:flutter/material.dart';

class ContainerExamples extends StatelessWidget {
  const ContainerExamples({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Container & Decoration')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Basic Container
            Container(
              width: 100,
              height: 100,
              color: Colors.blue,
              child: const Center(child: Text('Blue Box', style: TextStyle(color: Colors.white))),
            ),
            const SizedBox(height: 16),
            
            // Container with BoxDecoration
            Container(
              width: double.infinity,
              padding: const EdgeInsets.all(16),
              decoration: BoxDecoration(
                gradient: const LinearGradient(
                  colors: [Colors.purple, Colors.blue],
                  begin: Alignment.topLeft,
                  end: Alignment.bottomRight,
                ),
                borderRadius: BorderRadius.circular(12),
                boxShadow: const [
                  BoxShadow(
                    color: Colors.black26,
                    blurRadius: 8,
                    offset: Offset(0, 4),
                  ),
                ],
              ),
              child: const Text(
                'Gradient Container',
                style: TextStyle(color: Colors.white, fontSize: 18),
                textAlign: TextAlign.center,
              ),
            ),
            const SizedBox(height: 16),
            
            // Container with border
            Container(
              padding: const EdgeInsets.all(12),
              decoration: BoxDecoration(
                border: Border.all(color: Colors.blue, width: 2),
                borderRadius: BorderRadius.circular(8),
              ),
              child: const Text('Container with Border'),
            ),
            const SizedBox(height: 16),
            
            // Container with image
            Container(
              width: double.infinity,
              height: 150,
              decoration: BoxDecoration(
                borderRadius: BorderRadius.circular(12),
                image: const DecorationImage(
                  image: NetworkImage('https://via.placeholder.com/400x150'),
                  fit: BoxFit.cover,
                ),
              ),
              child: Container(
                decoration: BoxDecoration(
                  borderRadius: BorderRadius.circular(12),
                  gradient: LinearGradient(
                    colors: [Colors.transparent, Colors.black.withOpacity(0.7)],
                    begin: Alignment.topCenter,
                    end: Alignment.bottomCenter,
                  ),
                ),
                padding: const EdgeInsets.all(12),
                alignment: Alignment.bottomLeft,
                child: const Text(
                  'Image Caption',
                  style: TextStyle(color: Colors.white, fontSize: 16),
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

## ขั้นตอนที่ 248: BuildContext

```dart
import 'package:flutter/material.dart';

// BuildContext คือ handle ไปยังตำแหน่งของ Widget ใน Widget Tree
// ใช้หา Theme, MediaQuery, Navigator, Scaffold ฯลฯ

class BuildContextExamples extends StatelessWidget {
  const BuildContextExamples({super.key});
  
  @override
  Widget build(BuildContext context) {
    // ── MediaQuery ──
    double screenWidth = MediaQuery.of(context).size.width;
    double screenHeight = MediaQuery.of(context).size.height;
    double pixelRatio = MediaQuery.of(context).devicePixelRatio;
    EdgeInsets padding = MediaQuery.of(context).padding;
    
    // ── Theme ──
    ThemeData theme = Theme.of(context);
    ColorScheme colors = theme.colorScheme;
    TextTheme textTheme = theme.textTheme;
    
    return Scaffold(
      appBar: AppBar(title: const Text('BuildContext')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Screen info
            Card(
              child: Padding(
                padding: const EdgeInsets.all(12),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    const Text('Screen Info', style: TextStyle(fontWeight: FontWeight.bold)),
                    Text('Width: ${screenWidth.toStringAsFixed(1)}'),
                    Text('Height: ${screenHeight.toStringAsFixed(1)}'),
                    Text('Pixel Ratio: $pixelRatio'),
                    Text('Top padding: ${padding.top}'),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 16),
            
            // Theme colors
            Card(
              child: Padding(
                padding: const EdgeInsets.all(12),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    const Text('Theme Colors', style: TextStyle(fontWeight: FontWeight.bold)),
                    Row(
                      children: [
                        _ColorBox(color: colors.primary, label: 'Primary'),
                        _ColorBox(color: colors.secondary, label: 'Secondary'),
                        _ColorBox(color: colors.tertiary, label: 'Tertiary'),
                        _ColorBox(color: colors.error, label: 'Error'),
                      ],
                    ),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 16),
            
            // Responsive layout using context
            screenWidth > 600
                ? const Row(
                    children: [
                      Expanded(child: Placeholder(fallbackHeight: 100)),
                      SizedBox(width: 16),
                      Expanded(child: Placeholder(fallbackHeight: 100)),
                    ],
                  )
                : const Placeholder(fallbackHeight: 100),
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
    return Expanded(
      child: Column(
        children: [
          Container(
            height: 40,
            color: color,
          ),
          Text(label, style: const TextStyle(fontSize: 10)),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 249-260: โปรเจกต์ - Product Card App

```dart
// main.dart
import 'package:flutter/material.dart';

void main() {
  runApp(const ProductApp());
}

class ProductApp extends StatelessWidget {
  const ProductApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Product App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const ProductListScreen(),
    );
  }
}

// ─── Model ───
class Product {
  final String id;
  final String name;
  final String description;
  final double price;
  final String imageUrl;
  final double rating;
  final int reviews;
  final bool inStock;
  
  const Product({
    required this.id,
    required this.name,
    required this.description,
    required this.price,
    required this.imageUrl,
    required this.rating,
    required this.reviews,
    required this.inStock,
  });
}

// ─── Sample Data ───
final List<Product> sampleProducts = [
  const Product(
    id: '1',
    name: 'Flutter Developer Book',
    description: 'หนังสือเรียน Flutter ฉบับสมบูรณ์ สำหรับทุกระดับ',
    price: 599,
    imageUrl: 'https://via.placeholder.com/300x200',
    rating: 4.8,
    reviews: 1250,
    inStock: true,
  ),
  const Product(
    id: '2',
    name: 'Dart Programming Guide',
    description: 'คู่มือการเขียนโปรแกรม Dart แบบครบวงจร',
    price: 499,
    imageUrl: 'https://via.placeholder.com/300x200',
    rating: 4.5,
    reviews: 890,
    inStock: true,
  ),
  const Product(
    id: '3',
    name: 'Mobile Dev Course',
    description: 'คอร์สสอนพัฒนาแอปมือถือด้วย Flutter',
    price: 1299,
    imageUrl: 'https://via.placeholder.com/300x200',
    rating: 4.9,
    reviews: 2340,
    inStock: false,
  ),
];

// ─── Product List Screen ───
class ProductListScreen extends StatefulWidget {
  const ProductListScreen({super.key});
  
  @override
  State<ProductListScreen> createState() => _ProductListScreenState();
}

class _ProductListScreenState extends State<ProductListScreen> {
  String _searchQuery = '';
  Set<String> _favorites = {};
  
  List<Product> get _filteredProducts => sampleProducts
      .where((p) =>
          p.name.toLowerCase().contains(_searchQuery.toLowerCase()) ||
          p.description.toLowerCase().contains(_searchQuery.toLowerCase()))
      .toList();
  
  void _toggleFavorite(String productId) {
    setState(() {
      if (_favorites.contains(productId)) {
        _favorites.remove(productId);
      } else {
        _favorites.add(productId);
      }
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('สินค้า'),
        actions: [
          Badge(
            label: Text('${_favorites.length}'),
            isLabelVisible: _favorites.isNotEmpty,
            child: IconButton(
              icon: const Icon(Icons.favorite),
              onPressed: () {},
            ),
          ),
          IconButton(
            icon: const Icon(Icons.shopping_cart),
            onPressed: () {},
          ),
        ],
      ),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: SearchBar(
              hintText: 'ค้นหาสินค้า...',
              leading: const Icon(Icons.search),
              onChanged: (value) => setState(() => _searchQuery = value),
            ),
          ),
          Expanded(
            child: _filteredProducts.isEmpty
                ? Center(
                    child: Column(
                      mainAxisAlignment: MainAxisAlignment.center,
                      children: [
                        const Icon(Icons.search_off, size: 64, color: Colors.grey),
                        const SizedBox(height: 16),
                        Text(
                          'ไม่พบสินค้า "$_searchQuery"',
                          style: TextStyle(color: Colors.grey[600]),
                        ),
                      ],
                    ),
                  )
                : ListView.builder(
                    padding: const EdgeInsets.symmetric(horizontal: 16),
                    itemCount: _filteredProducts.length,
                    itemBuilder: (context, index) {
                      Product product = _filteredProducts[index];
                      bool isFavorite = _favorites.contains(product.id);
                      
                      return ProductCard(
                        product: product,
                        isFavorite: isFavorite,
                        onFavoriteToggle: () => _toggleFavorite(product.id),
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }
}

// ─── Product Card Widget ───
class ProductCard extends StatelessWidget {
  final Product product;
  final bool isFavorite;
  final VoidCallback onFavoriteToggle;
  
  const ProductCard({
    super.key,
    required this.product,
    required this.isFavorite,
    required this.onFavoriteToggle,
  });
  
  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.only(bottom: 12),
      clipBehavior: Clip.antiAlias,
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          // Image section
          Stack(
            children: [
              Image.network(
                product.imageUrl,
                width: double.infinity,
                height: 150,
                fit: BoxFit.cover,
                errorBuilder: (context, error, stack) => Container(
                  height: 150,
                  color: Colors.grey[300],
                  child: const Icon(Icons.image, size: 48),
                ),
              ),
              if (!product.inStock)
                Positioned.fill(
                  child: Container(
                    color: Colors.black45,
                    child: const Center(
                      child: Text(
                        'หมดสต็อก',
                        style: TextStyle(color: Colors.white, fontSize: 18, fontWeight: FontWeight.bold),
                      ),
                    ),
                  ),
                ),
              Positioned(
                top: 8,
                right: 8,
                child: IconButton.filled(
                  onPressed: onFavoriteToggle,
                  icon: Icon(isFavorite ? Icons.favorite : Icons.favorite_border),
                  style: IconButton.styleFrom(
                    backgroundColor: Colors.white.withOpacity(0.9),
                    foregroundColor: isFavorite ? Colors.red : Colors.grey,
                  ),
                ),
              ),
            ],
          ),
          
          // Content section
          Padding(
            padding: const EdgeInsets.all(12),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  product.name,
                  style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
                ),
                const SizedBox(height: 4),
                Text(
                  product.description,
                  style: TextStyle(fontSize: 13, color: Colors.grey[600]),
                  maxLines: 2,
                  overflow: TextOverflow.ellipsis,
                ),
                const SizedBox(height: 8),
                Row(
                  children: [
                    const Icon(Icons.star, color: Colors.amber, size: 16),
                    const SizedBox(width: 4),
                    Text(
                      product.rating.toString(),
                      style: const TextStyle(fontWeight: FontWeight.bold),
                    ),
                    const SizedBox(width: 4),
                    Text(
                      '(${product.reviews} รีวิว)',
                      style: TextStyle(color: Colors.grey[600], fontSize: 12),
                    ),
                    const Spacer(),
                    Text(
                      '฿${product.price.toStringAsFixed(0)}',
                      style: const TextStyle(
                        fontSize: 18,
                        fontWeight: FontWeight.bold,
                        color: Colors.deepPurple,
                      ),
                    ),
                  ],
                ),
                const SizedBox(height: 8),
                SizedBox(
                  width: double.infinity,
                  child: ElevatedButton(
                    onPressed: product.inStock
                        ? () {
                            ScaffoldMessenger.of(context).showSnackBar(
                              SnackBar(
                                content: Text('เพิ่ม "${product.name}" ในตะกร้าแล้ว'),
                                action: SnackBarAction(label: 'ดูตะกร้า', onPressed: () {}),
                              ),
                            );
                          }
                        : null,
                    child: Text(product.inStock ? 'เพิ่มในตะกร้า' : 'หมดสต็อก'),
                  ),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 261-280: สรุป Widget พื้นฐาน

### Widget Lifecycle

```
StatelessWidget:
  build() → Widget Tree

StatefulWidget:
  createState() → State
  State.initState()    ← เรียกครั้งเดียวตอนสร้าง
  State.build()        ← เรียกทุกครั้งที่ setState หรือ rebuild
  State.didUpdateWidget() ← เมื่อ parent rebuild ส่ง widget ใหม่
  State.dispose()      ← ตอน widget ถูกลบออกจาก tree
```

### Best Practices

```dart
// ✅ ดี: const constructors ลด rebuild
const MyWidget();

// ✅ ดี: keys สำหรับ list items
ListView.builder(
  itemBuilder: (ctx, i) => ProductCard(key: ValueKey(products[i].id), product: products[i]),
)

// ✅ ดี: จัดการ dispose
@override
void dispose() {
  controller.dispose();  // ลบ memory leak
  subscription.cancel();
  super.dispose();
}

// ❌ ไม่ดี: rebuild ทั้ง tree เมื่อไม่จำเป็น
// ใช้ const และ split widget แทน

// ✅ ดี: ใช้ const สำหรับ widget ที่ไม่เปลี่ยน
class _Header extends StatelessWidget {
  const _Header();
  // ...
}
```

---

**← [Part 08 - Error Handling](part-08-error-handling.md)**

**ต่อไป: [Part 10 - Layout Widgets →](part-10-layout-widgets.md)**
