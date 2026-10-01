# Part 23: Performance Optimization
## ขั้นตอนที่ 801-840

---

## 🎯 เป้าหมายของ Part นี้

- Flutter Performance tools
- Widget rebuilds optimization
- ListView.builder / GridView.builder
- Image caching และ optimization
- Isolates สำหรับ heavy computation
- Memory management

---

## ขั้นตอนที่ 801: ลด Widget Rebuilds

```dart
import 'package:flutter/material.dart';

// ❌ ปัญหา: Widget rebuild ทั้งหมดเมื่อ counter เปลี่ยน
class BadCounter extends StatefulWidget {
  const BadCounter({super.key});

  @override
  State<BadCounter> createState() => _BadCounterState();
}

class _BadCounterState extends State<BadCounter> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      // Header สร้างใหม่ทุกครั้ง แม้ข้อมูลไม่เปลี่ยน
      appBar: AppBar(title: const Text('Counter')),
      body: Column(
        children: [
          const _ExpensiveWidget(),  // rebuild โดยไม่จำเป็น
          Text('Count: $count', style: const TextStyle(fontSize: 32)),
          ElevatedButton(
            onPressed: () => setState(() => count++),
            child: const Text('Increment'),
          ),
        ],
      ),
    );
  }
}

// ✅ ดีกว่า: แยก state ออกมาเป็น widget เล็กๆ
class GoodCounter extends StatefulWidget {
  const GoodCounter({super.key});

  @override
  State<GoodCounter> createState() => _GoodCounterState();
}

class _GoodCounterState extends State<GoodCounter> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Counter')),
      body: Column(
        children: [
          const _ExpensiveWidget(),  // เป็น const → ไม่ rebuild
          _CountDisplay(count: count),  // rebuild เฉพาะส่วนนี้
          ElevatedButton(
            onPressed: () => setState(() => count++),
            child: const Text('Increment'),
          ),
        ],
      ),
    );
  }
}

// Widget แยก (StatelessWidget) เพื่อควบคุม rebuild
class _CountDisplay extends StatelessWidget {
  final int count;
  const _CountDisplay({required this.count});

  @override
  Widget build(BuildContext context) {
    return Text('Count: $count', style: const TextStyle(fontSize: 32));
  }
}

class _ExpensiveWidget extends StatelessWidget {
  const _ExpensiveWidget();

  @override
  Widget build(BuildContext context) {
    // Expensive build operation
    return const Card(
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Text('Expensive Content'),
      ),
    );
  }
}

// ─── const widgets ───
// const widgets ถูก cache และไม่ rebuild
class ConstDemo extends StatelessWidget {
  const ConstDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return const Column(
      children: [
        // ✅ const: สร้างครั้งเดียว
        Icon(Icons.star),
        Text('Static Text'),
        SizedBox(height: 16),
        Divider(),
        // ❌ ไม่ควร: ทุกครั้งที่ build จะสร้างใหม่
        // Icon(Icons.star),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 802: Keys สำหรับ Widget Identity

```dart
import 'package:flutter/material.dart';

// Keys ช่วยให้ Flutter รู้ว่า widget ตัวไหนคือตัวไหน
// มีประโยชน์มากเมื่อ list มีการเรียงลำดับใหม่

class KeyDemo extends StatefulWidget {
  const KeyDemo({super.key});

  @override
  State<KeyDemo> createState() => _KeyDemoState();
}

class _KeyDemoState extends State<KeyDemo> {
  List<String> items = ['A', 'B', 'C'];

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // ✅ ใช้ Key เพื่อให้ Flutter track widget ได้ถูก
        ...items.map((item) => _ColorBox(key: ValueKey(item), label: item)),
        ElevatedButton(
          onPressed: () => setState(() => items.shuffle()),
          child: const Text('Shuffle'),
        ),
      ],
    );
  }
}

class _ColorBox extends StatefulWidget {
  final String label;
  const _ColorBox({super.key, required this.label});

  @override
  State<_ColorBox> createState() => _ColorBoxState();
}

class _ColorBoxState extends State<_ColorBox> {
  // State ที่จะถูก preserve เมื่อใช้ Key
  final Color color = Colors.primaries[DateTime.now().millisecond % Colors.primaries.length];

  @override
  Widget build(BuildContext context) {
    return Container(
      margin: const EdgeInsets.all(8),
      padding: const EdgeInsets.all(16),
      color: color,
      child: Text(widget.label, style: const TextStyle(color: Colors.white, fontSize: 24)),
    );
  }
}
```

---

## ขั้นตอนที่ 803: ListView Optimization

```dart
import 'package:flutter/material.dart';

// ─── Large List Optimization ───
class OptimizedList extends StatelessWidget {
  final List<Map<String, dynamic>> items;

  const OptimizedList({super.key, required this.items});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      // addAutomaticKeepAlives: false → ประหยัด memory
      addAutomaticKeepAlives: false,
      // addRepaintBoundaries: true → แต่ละ item ใน repaint boundary แยก
      addRepaintBoundaries: true,
      
      itemCount: items.length,
      itemExtent: 80,  // กำหนด height คงที่ → เร็วกว่า dynamic height
      
      itemBuilder: (context, i) {
        Map<String, dynamic> item = items[i];
        return ListTile(
          key: ValueKey(item['id']),
          leading: CircleAvatar(child: Text('${i + 1}')),
          title: Text(item['title'].toString()),
          subtitle: Text(item['subtitle'].toString()),
        );
      },
    );
  }
}

// ─── Lazy Loading List ───
class InfiniteList extends StatefulWidget {
  const InfiniteList({super.key});

  @override
  State<InfiniteList> createState() => _InfiniteListState();
}

class _InfiniteListState extends State<InfiniteList> {
  final List<String> _items = List.generate(20, (i) => 'Item ${i + 1}');
  bool _isLoading = false;
  bool _hasMore = true;
  final ScrollController _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(_onScroll);
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
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

    await Future.delayed(const Duration(seconds: 1));

    int newCount = _items.length;
    List<String> newItems = List.generate(
      20,
      (i) => 'Item ${newCount + i + 1}',
    );

    setState(() {
      _items.addAll(newItems);
      _isLoading = false;
      if (_items.length >= 100) _hasMore = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Items: ${_items.length}')),
      body: ListView.builder(
        controller: _scrollController,
        itemCount: _items.length + (_hasMore ? 1 : 0),
        itemBuilder: (context, i) {
          if (i == _items.length) {
            return const Padding(
              padding: EdgeInsets.all(16),
              child: Center(child: CircularProgressIndicator()),
            );
          }
          return ListTile(title: Text(_items[i]));
        },
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 804: Isolates สำหรับ Heavy Computation

```dart
import 'dart:isolate';
import 'package:flutter/foundation.dart';

// ─── ปัญหา: การคำนวณหนักทำให้ UI freeze ───
int _computePrimesSync(int max) {
  List<int> primes = [];
  for (int n = 2; n <= max; n++) {
    bool isPrime = true;
    for (int i = 2; i * i <= n; i++) {
      if (n % i == 0) { isPrime = false; break; }
    }
    if (isPrime) primes.add(n);
  }
  return primes.length;
}

// ─── แก้ปัญหา: ใช้ compute() เพื่อรันใน Isolate ───
Future<int> computePrimesAsync(int max) async {
  return compute(_computePrimesSync, max);
}

// ─── Manual Isolate (สำหรับ task ที่ซับซ้อนกว่า) ───
Future<List<int>> sortLargeListInIsolate(List<int> data) async {
  ReceivePort receivePort = ReceivePort();
  
  await Isolate.spawn(_sortIsolateEntry, {
    'data': data,
    'sendPort': receivePort.sendPort,
  });
  
  return await receivePort.first as List<int>;
}

void _sortIsolateEntry(Map<String, dynamic> message) {
  List<int> data = List<int>.from(message['data'] as List);
  SendPort sendPort = message['sendPort'] as SendPort;
  
  data.sort();
  sendPort.send(data);
}

// ─── Usage in Widget ───
class IsolateDemo extends StatefulWidget {
  const IsolateDemo({super.key});

  @override
  State<IsolateDemo> createState() => _IsolateDemoState();
}

class _IsolateDemoState extends State<IsolateDemo> {
  String _result = '';
  bool _computing = false;

  Future<void> _compute() async {
    setState(() {
      _computing = true;
      _result = 'Computing...';
    });

    Stopwatch sw = Stopwatch()..start();
    int count = await computePrimesAsync(100000);
    sw.stop();

    setState(() {
      _result = 'Found $count primes in ${sw.elapsedMilliseconds}ms';
      _computing = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Isolate Demo')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(_result, textAlign: TextAlign.center),
            const SizedBox(height: 24),
            ElevatedButton(
              onPressed: _computing ? null : _compute,
              child: _computing
                  ? const CircularProgressIndicator()
                  : const Text('คำนวณ (ไม่บล็อก UI)'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 805: Image Optimization

```dart
import 'package:flutter/material.dart';
import 'package:cached_network_image/cached_network_image.dart';

// pubspec.yaml: cached_network_image: ^3.3.1

class OptimizedImageList extends StatelessWidget {
  const OptimizedImageList({super.key});

  static const List<String> _imageUrls = [
    'https://picsum.photos/400/300?random=1',
    'https://picsum.photos/400/300?random=2',
    'https://picsum.photos/400/300?random=3',
  ];

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: _imageUrls.length,
      itemBuilder: (context, i) {
        return Card(
          margin: const EdgeInsets.all(8),
          child: Column(
            children: [
              // ✅ CachedNetworkImage: caches อัตโนมัติ
              CachedNetworkImage(
                imageUrl: _imageUrls[i],
                height: 200,
                width: double.infinity,
                fit: BoxFit.cover,
                placeholder: (_, __) => const SizedBox(
                  height: 200,
                  child: Center(child: CircularProgressIndicator()),
                ),
                errorWidget: (_, __, ___) => const SizedBox(
                  height: 200,
                  child: Center(child: Icon(Icons.broken_image, size: 60)),
                ),
              ),
              Padding(
                padding: const EdgeInsets.all(8),
                child: Text('Image ${i + 1}'),
              ),
            ],
          ),
        );
      },
    );
  }
}

// ─── Responsive Image ───
class ResponsiveImage extends StatelessWidget {
  final String url;

  const ResponsiveImage({super.key, required this.url});

  @override
  Widget build(BuildContext context) {
    double screenWidth = MediaQuery.of(context).size.width;
    
    // เลือก size ตาม screen
    String optimizedUrl = screenWidth > 600
        ? url.replaceAll('400/300', '800/600')  // Tablet
        : url;                                    // Phone

    return CachedNetworkImage(
      imageUrl: optimizedUrl,
      memCacheWidth: screenWidth.toInt(),  // Cache ขนาดที่เหมาะสม
      fit: BoxFit.cover,
    );
  }
}
```

---

## ขั้นตอนที่ 806: RepaintBoundary

```dart
import 'package:flutter/material.dart';

// RepaintBoundary ช่วยให้เฉพาะส่วนที่เปลี่ยนถูก repaint
class RepaintBoundaryDemo extends StatefulWidget {
  const RepaintBoundaryDemo({super.key});

  @override
  State<RepaintBoundaryDemo> createState() => _RepaintBoundaryDemoState();
}

class _RepaintBoundaryDemoState extends State<RepaintBoundaryDemo> {
  int _counter = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          // ✅ ส่วนนี้จะไม่ถูก repaint เมื่อ counter เปลี่ยน
          const RepaintBoundary(
            child: _StaticHeavyWidget(),
          ),

          // ✅ เฉพาะส่วนนี้จะ repaint
          RepaintBoundary(
            child: Text('Count: $_counter', style: const TextStyle(fontSize: 36)),
          ),

          ElevatedButton(
            onPressed: () => setState(() => _counter++),
            child: const Text('Increment'),
          ),
        ],
      ),
    );
  }
}

class _StaticHeavyWidget extends StatelessWidget {
  const _StaticHeavyWidget();

  @override
  Widget build(BuildContext context) {
    return Container(
      height: 200,
      color: Colors.blue.shade50,
      child: const Center(child: Text('Heavy Static Widget')),
    );
  }
}
```

---

**← [Part 22 - Internationalization](part-22-i18n.md)**

**ต่อไป: [Part 24 - Custom Painting →](part-24-custom-painting.md)**
