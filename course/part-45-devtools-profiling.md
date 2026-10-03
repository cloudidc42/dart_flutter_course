# Part 45: Flutter DevTools & Performance Profiling
## ขั้นตอนที่ 1681-1720

---

## 🎯 เป้าหมายของ Part นี้

- Flutter DevTools
- Performance profiling
- Memory leaks detection
- Widget rebuilds optimization
- Network inspection

---

## ขั้นตอนที่ 1681: Flutter DevTools

```bash
# เปิด DevTools
flutter pub global activate devtools
dart devtools

# หรือผ่าน VS Code: F5 -> "Open DevTools"
# หรือผ่าน terminal
flutter run --track-widget-creation
# จาก URL ที่ได้เปิด browser

# Profile mode
flutter run --profile

# ดู performance
flutter run --profile --trace-skia
```

---

## ขั้นตอนที่ 1682: Performance Tracking

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';

// ─── Enable performance overlay ───
class PerformanceApp extends StatelessWidget {
  const PerformanceApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      // แสดง performance overlay (FPS, Raster)
      showPerformanceOverlay: kProfileMode,
      // แสดง widget bounds
      debugShowMaterialGrid: false,
      // Semantic debugger
      debugShowCheckedModeBanner: false,
      home: const HomePage(),
    );
  }
}

// ─── Timeline Events ───
class PerformanceTracer {
  static void trace(String name, void Function() fn) {
    Timeline.startSync(name);
    try {
      fn();
    } finally {
      Timeline.finishSync();
    }
  }

  static Future<T> traceAsync<T>(String name, Future<T> Function() fn) async {
    Timeline.startSync(name);
    try {
      return await fn();
    } finally {
      Timeline.finishSync();
    }
  }
}

// ─── Track Widget Rebuilds ───
class RebuildTracker extends StatefulWidget {
  final String name;
  final Widget child;
  const RebuildTracker({super.key, required this.name, required this.child});

  @override
  State<RebuildTracker> createState() => _RebuildTrackerState();
}

class _RebuildTrackerState extends State<RebuildTracker> {
  int _rebuilds = 0;

  @override
  Widget build(BuildContext context) {
    _rebuilds++;
    if (kDebugMode) {
      print('${widget.name} rebuilt: $_rebuilds times');
    }
    return widget.child;
  }
}

// ─── Rebuild Counter Overlay ───
class RebuildCounterOverlay extends StatefulWidget {
  final Widget child;
  const RebuildCounterOverlay({super.key, required this.child});

  @override
  State<RebuildCounterOverlay> createState() => _RebuildCounterOverlayState();
}

class _RebuildCounterOverlayState extends State<RebuildCounterOverlay> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    _count++;
    return Stack(
      children: [
        widget.child,
        if (kDebugMode)
          Positioned(
            top: 0,
            right: 0,
            child: Container(
              color: Colors.red.withOpacity(0.8),
              padding: const EdgeInsets.all(4),
              child: Text(
                'Rebuilds: $_count',
                style: const TextStyle(color: Colors.white, fontSize: 10),
              ),
            ),
          ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 1683: Memory Leak Detection

```dart
import 'dart:developer';

// ─── Memory monitoring ───
class MemoryMonitor {
  static Timer? _timer;

  static void startMonitoring({Duration interval = const Duration(seconds: 5)}) {
    _timer = Timer.periodic(interval, (_) {
      _checkMemory();
    });
  }

  static void stopMonitoring() {
    _timer?.cancel();
  }

  static void _checkMemory() {
    // ใน production ใช้ dart:developer เพื่อดู memory
    NativeMemory memory = NativeMemory();
    debugPrint(
      'Memory: ${memory.total ~/ 1024 ~/ 1024}MB total, '
      '${memory.used ~/ 1024 ~/ 1024}MB used',
    );
  }
}

// Mock NativeMemory for demo
class NativeMemory {
  int get total => 512 * 1024 * 1024;
  int get used => 128 * 1024 * 1024;
}

// ─── Prevent common memory leaks ───

// ❌ Bad: StreamSubscription ไม่ cancel
class BadWidget extends StatefulWidget {
  const BadWidget({super.key});
  @override
  State<BadWidget> createState() => _BadWidgetState();
}

class _BadWidgetState extends State<BadWidget> {
  // LEAK: subscription ไม่ถูก cancel
  final Stream<int> _stream = Stream.periodic(const Duration(seconds: 1), (i) => i);
  late final StreamSubscription _sub;

  @override
  void initState() {
    super.initState();
    _sub = _stream.listen((value) => setState(() {}));
  }

  // FORGOT dispose! -> memory leak

  @override
  Widget build(BuildContext context) => const Placeholder();
}

// ✅ Good: Cancel subscription ใน dispose
class GoodWidget extends StatefulWidget {
  const GoodWidget({super.key});
  @override
  State<GoodWidget> createState() => _GoodWidgetState();
}

class _GoodWidgetState extends State<GoodWidget> {
  final Stream<int> _stream = Stream.periodic(const Duration(seconds: 1), (i) => i);
  StreamSubscription? _sub;

  @override
  void initState() {
    super.initState();
    _sub = _stream.listen((value) {
      if (mounted) setState(() {});
    });
  }

  @override
  void dispose() {
    _sub?.cancel(); // ✅ Always cancel!
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => const Placeholder();
}

// ─── AnimationController leak prevention ───
class AnimationWidget extends StatefulWidget {
  const AnimationWidget({super.key});
  @override
  State<AnimationWidget> createState() => _AnimationWidgetState();
}

class _AnimationWidgetState extends State<AnimationWidget>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 2),
    )..repeat();
  }

  @override
  void dispose() {
    _controller.dispose(); // ✅ Always dispose!
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) => RotationTransition(
        turns: _controller,
        child: const Icon(Icons.refresh, size: 48),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1684: Widget Optimization Techniques

```dart
import 'package:flutter/material.dart';

// ─── 1. const widgets ───
class OptimizedWidgets extends StatelessWidget {
  const OptimizedWidgets({super.key});

  @override
  Widget build(BuildContext context) {
    // ✅ const = ไม่ rebuild
    return const Column(
      children: [
        Icon(Icons.star, size: 48), // const
        Text('Hello', style: TextStyle(fontSize: 24)), // const
        SizedBox(height: 16), // const
      ],
    );
  }
}

// ─── 2. Extract widgets ───
// ❌ Bad: รวมทุกอย่างในที่เดียว
class BigBadWidget extends StatefulWidget {
  const BigBadWidget({super.key});
  @override
  State<BigBadWidget> createState() => _BigBadWidgetState();
}

class _BigBadWidgetState extends State<BigBadWidget> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // นี้ rebuild ทุกครั้งที่ counter เปลี่ยน แม้มันไม่เปลี่ยน
        Container(
          padding: const EdgeInsets.all(32),
          child: const FlutterLogo(size: 100), // ไม่ควร rebuild
        ),
        Text('Count: $_count'),
        ElevatedButton(
          onPressed: () => setState(() => _count++),
          child: const Text('Increment'),
        ),
      ],
    );
  }
}

// ✅ Good: แยก widget ที่ไม่เปลี่ยน
class BigGoodWidget extends StatefulWidget {
  const BigGoodWidget({super.key});
  @override
  State<BigGoodWidget> createState() => _BigGoodWidgetState();
}

class _BigGoodWidgetState extends State<BigGoodWidget> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        const _StaticHeader(), // ไม่ rebuild เพราะเป็น const class
        Text('Count: $_count'), // rebuild เฉพาะนี้
        ElevatedButton(
          onPressed: () => setState(() => _count++),
          child: const Text('Increment'),
        ),
      ],
    );
  }
}

class _StaticHeader extends StatelessWidget {
  const _StaticHeader(); // const constructor

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(32),
      child: const FlutterLogo(size: 100),
    );
  }
}

// ─── 3. ListView.builder ───
class EfficientList extends StatelessWidget {
  final List<String> items;
  const EfficientList({super.key, required this.items});

  @override
  Widget build(BuildContext context) {
    // ✅ builder = lazy loading, ไม่ render ทุก item
    return ListView.builder(
      itemCount: items.length,
      // ✅ itemExtent ทำให้ scroll calculation เร็วขึ้น
      itemExtent: 60,
      itemBuilder: (context, index) => ListTile(
        title: Text(items[index]),
      ),
    );
  }
}

// ─── 4. RepaintBoundary ───
class ComplexAnimation extends StatelessWidget {
  const ComplexAnimation({super.key});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // ✅ RepaintBoundary กั้นไม่ให้ animation ทำให้ parent repaint
        RepaintBoundary(
          child: const _HeavyAnimatedWidget(),
        ),
        // parent ส่วนนี้จะไม่ repaint เมื่อ animation เล่น
        const Text('Static Content'),
        const Text('More Static Content'),
      ],
    );
  }
}

class _HeavyAnimatedWidget extends StatefulWidget {
  const _HeavyAnimatedWidget();

  @override
  State<_HeavyAnimatedWidget> createState() => _HeavyAnimatedWidgetState();
}

class _HeavyAnimatedWidgetState extends State<_HeavyAnimatedWidget>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this, duration: const Duration(seconds: 2))..repeat();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return RotationTransition(
      turns: _controller,
      child: Container(
        width: 100,
        height: 100,
        color: Colors.blue,
        child: const Center(child: Text('Spinning', style: TextStyle(color: Colors.white))),
      ),
    );
  }
}

// ─── 5. Avoid rebuilds with ValueNotifier ───
class EfficientCounter extends StatelessWidget {
  const EfficientCounter({super.key});

  static final ValueNotifier<int> _counter = ValueNotifier(0);

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // ✅ ValueListenableBuilder: rebuild เฉพาะ Text
        ValueListenableBuilder<int>(
          valueListenable: _counter,
          builder: (context, count, child) => Text('Count: $count'),
        ),
        // ✅ child ไม่ rebuild (pass ผ่าน child parameter)
        ElevatedButton(
          onPressed: () => _counter.value++,
          child: const Text('Increment'),
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 1685: Network Inspector

```dart
import 'package:dio/dio.dart';
import 'dart:developer';

// ─── Network Logging Interceptor ───
class NetworkInspector extends Interceptor {
  final bool logRequestHeaders;
  final bool logResponseHeaders;
  final bool logRequestBody;
  final bool logResponseBody;
  final int maxBodyLogLength;

  const NetworkInspector({
    this.logRequestHeaders = false,
    this.logResponseHeaders = false,
    this.logRequestBody = true,
    this.logResponseBody = true,
    this.maxBodyLogLength = 1000,
  });

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    if (kDebugMode) {
      StringBuffer log = StringBuffer();
      log.writeln('┌─────────────────────────────────────');
      log.writeln('│ 📤 REQUEST');
      log.writeln('│ Method: ${options.method}');
      log.writeln('│ URL: ${options.uri}');

      if (logRequestHeaders) {
        log.writeln('│ Headers: ${options.headers}');
      }

      if (logRequestBody && options.data != null) {
        String body = options.data.toString();
        if (body.length > maxBodyLogLength) {
          body = '${body.substring(0, maxBodyLogLength)}... (truncated)';
        }
        log.writeln('│ Body: $body');
      }

      log.write('└─────────────────────────────────────');
      debugPrint(log.toString());
    }
    handler.next(options);
  }

  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    if (kDebugMode) {
      StringBuffer log = StringBuffer();
      log.writeln('┌─────────────────────────────────────');
      log.writeln('│ 📥 RESPONSE');
      log.writeln('│ Status: ${response.statusCode}');
      log.writeln('│ URL: ${response.requestOptions.uri}');
      log.writeln('│ Duration: ${response.requestOptions.extra['startTime'] != null
          ? '${DateTime.now().difference(response.requestOptions.extra['startTime']).inMilliseconds}ms'
          : 'N/A'}');

      if (logResponseBody) {
        String body = response.data.toString();
        if (body.length > maxBodyLogLength) {
          body = '${body.substring(0, maxBodyLogLength)}... (truncated)';
        }
        log.writeln('│ Body: $body');
      }

      log.write('└─────────────────────────────────────');
      debugPrint(log.toString());
    }
    handler.next(response);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    if (kDebugMode) {
      debugPrint('❌ REQUEST FAILED: ${err.requestOptions.uri}');
      debugPrint('   Error: ${err.message}');
      debugPrint('   Status: ${err.response?.statusCode}');
    }
    handler.next(err);
  }
}

// ─── Usage ───
Dio createDioWithInspector() {
  Dio dio = Dio();

  if (kDebugMode) {
    dio.interceptors.add(const NetworkInspector(
      logRequestBody: true,
      logResponseBody: true,
    ));
  }

  return dio;
}
```

---

**← [Part 44 - GraphQL](part-44-graphql.md)**

**ต่อไป: [Part 46 - Advanced Testing →](part-46-advanced-testing.md)**
