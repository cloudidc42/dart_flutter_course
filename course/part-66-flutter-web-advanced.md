# Part 66: Flutter Web Advanced
## ขั้นตอนที่ 2521-2560

## 🎯 เป้าหมายของ Part นี้
- เรียนรู้การทำ SEO optimization สำหรับ Flutter Web
- ตั้งค่า Progressive Web App (PWA)
- ใช้งาน Web Workers
- Compile เป็น WebAssembly (WASM)
- กำหนดกลยุทธ์ Service Worker caching
- ตรวจสอบ platform ด้วย web-specific detection

---

## ขั้นตอนที่ 2521: SEO Optimization สำหรับ Flutter Web

Flutter Web ต้องการ SEO setup พิเศษ เพราะ content ถูก render ด้วย JavaScript

```dart
// lib/web/seo_helper.dart
import 'dart:html' as html;
import 'package:flutter/foundation.dart';

class SeoHelper {
  /// อัปเดต meta tags สำหรับ SEO
  static void updateMetaTags({
    required String title,
    required String description,
    String? keywords,
    String? ogImage,
    String? canonicalUrl,
  }) {
    if (!kIsWeb) return;

    // อัปเดต title
    html.document.title = title;

    // อัปเดต meta description
    _setMetaTag('description', description);

    // อัปเดต keywords
    if (keywords != null) {
      _setMetaTag('keywords', keywords);
    }

    // Open Graph tags
    _setMetaProperty('og:title', title);
    _setMetaProperty('og:description', description);
    if (ogImage != null) {
      _setMetaProperty('og:image', ogImage);
    }

    // Twitter Card tags
    _setMetaTag('twitter:card', 'summary_large_image');
    _setMetaTag('twitter:title', title);
    _setMetaTag('twitter:description', description);

    // Canonical URL
    if (canonicalUrl != null) {
      _setCanonicalUrl(canonicalUrl);
    }
  }

  static void _setMetaTag(String name, String content) {
    var element = html.document.querySelector('meta[name="$name"]');
    if (element == null) {
      element = html.MetaElement()
        ..name = name
        ..content = content;
      html.document.head!.append(element);
    } else {
      (element as html.MetaElement).content = content;
    }
  }

  static void _setMetaProperty(String property, String content) {
    var element = html.document.querySelector('meta[property="$property"]');
    if (element == null) {
      element = html.MetaElement()
        ..setAttribute('property', property)
        ..content = content;
      html.document.head!.append(element);
    } else {
      (element as html.MetaElement).content = content;
    }
  }

  static void _setCanonicalUrl(String url) {
    var link = html.document.querySelector('link[rel="canonical"]');
    if (link == null) {
      link = html.LinkElement()
        ..rel = 'canonical'
        ..href = url;
      html.document.head!.append(link);
    } else {
      (link as html.LinkElement).href = url;
    }
  }

  /// เพิ่ม JSON-LD structured data
  static void addStructuredData(Map<String, dynamic> jsonLd) {
    if (!kIsWeb) return;

    final script = html.ScriptElement()
      ..type = 'application/ld+json'
      ..text = _mapToJson(jsonLd);

    // ลบ script เดิมถ้ามี
    html.document
        .querySelectorAll('script[type="application/ld+json"]')
        .forEach((e) => e.remove());

    html.document.head!.append(script);
  }

  static String _mapToJson(Map<String, dynamic> map) {
    final buffer = StringBuffer('{');
    var first = true;
    map.forEach((key, value) {
      if (!first) buffer.write(',');
      first = false;
      buffer.write('"$key":');
      if (value is String) {
        buffer.write('"$value"');
      } else if (value is Map<String, dynamic>) {
        buffer.write(_mapToJson(value));
      } else if (value is List) {
        buffer.write('[');
        for (var i = 0; i < value.length; i++) {
          if (i > 0) buffer.write(',');
          if (value[i] is String) {
            buffer.write('"${value[i]}"');
          } else if (value[i] is Map<String, dynamic>) {
            buffer.write(_mapToJson(value[i] as Map<String, dynamic>));
          }
        }
        buffer.write(']');
      } else {
        buffer.write(value);
      }
    });
    buffer.write('}');
    return buffer.toString();
  }
}
```

---

## ขั้นตอนที่ 2522: Structured Data สำหรับเว็บไซต์

```dart
// lib/web/structured_data.dart
import 'seo_helper.dart';

class StructuredDataHelper {
  /// เพิ่ม Organization structured data
  static void addOrganizationData({
    required String name,
    required String url,
    String? logo,
    String? description,
    List<String>? sameAs,
  }) {
    final data = <String, dynamic>{
      '@context': 'https://schema.org',
      '@type': 'Organization',
      'name': name,
      'url': url,
    };

    if (logo != null) data['logo'] = logo;
    if (description != null) data['description'] = description;
    if (sameAs != null) data['sameAs'] = sameAs;

    SeoHelper.addStructuredData(data);
  }

  /// เพิ่ม Article structured data
  static void addArticleData({
    required String headline,
    required String description,
    required String authorName,
    required String datePublished,
    String? dateModified,
    String? image,
    String? url,
  }) {
    final data = <String, dynamic>{
      '@context': 'https://schema.org',
      '@type': 'Article',
      'headline': headline,
      'description': description,
      'author': {
        '@type': 'Person',
        'name': authorName,
      },
      'datePublished': datePublished,
    };

    if (dateModified != null) data['dateModified'] = dateModified;
    if (image != null) data['image'] = image;
    if (url != null) data['url'] = url;

    SeoHelper.addStructuredData(data);
  }

  /// เพิ่ม Product structured data
  static void addProductData({
    required String name,
    required String description,
    required double price,
    required String currency,
    String? image,
    String? sku,
    String? brand,
    double? ratingValue,
    int? reviewCount,
  }) {
    final data = <String, dynamic>{
      '@context': 'https://schema.org',
      '@type': 'Product',
      'name': name,
      'description': description,
      'offers': {
        '@type': 'Offer',
        'price': price.toString(),
        'priceCurrency': currency,
        'availability': 'https://schema.org/InStock',
      },
    };

    if (image != null) data['image'] = image;
    if (sku != null) data['sku'] = sku;
    if (brand != null) {
      data['brand'] = {'@type': 'Brand', 'name': brand};
    }

    if (ratingValue != null && reviewCount != null) {
      data['aggregateRating'] = {
        '@type': 'AggregateRating',
        'ratingValue': ratingValue.toString(),
        'reviewCount': reviewCount.toString(),
      };
    }

    SeoHelper.addStructuredData(data);
  }

  /// เพิ่ม BreadcrumbList structured data
  static void addBreadcrumb(List<BreadcrumbItem> items) {
    final itemListElement = items.asMap().entries.map((entry) {
      return {
        '@type': 'ListItem',
        'position': (entry.key + 1).toString(),
        'name': entry.value.name,
        'item': entry.value.url,
      };
    }).toList();

    final data = <String, dynamic>{
      '@context': 'https://schema.org',
      '@type': 'BreadcrumbList',
      'itemListElement': itemListElement,
    };

    SeoHelper.addStructuredData(data);
  }
}

class BreadcrumbItem {
  final String name;
  final String url;

  const BreadcrumbItem({required this.name, required this.url});
}
```

---

## ขั้นตอนที่ 2523: Progressive Web App (PWA) Setup

```dart
// web/manifest.json (สร้างไฟล์นี้ใน web/ folder)
// {
//   "name": "Flutter PWA App",
//   "short_name": "FlutterPWA",
//   "start_url": ".",
//   "display": "standalone",
//   "background_color": "#ffffff",
//   "theme_color": "#2196F3",
//   "description": "A Progressive Web App built with Flutter",
//   "orientation": "portrait-primary",
//   "prefer_related_applications": false,
//   "icons": [
//     {
//       "src": "icons/Icon-192.png",
//       "sizes": "192x192",
//       "type": "image/png"
//     },
//     {
//       "src": "icons/Icon-512.png",
//       "sizes": "512x512",
//       "type": "image/png"
//     },
//     {
//       "src": "icons/Icon-maskable-192.png",
//       "sizes": "192x192",
//       "type": "image/png",
//       "purpose": "maskable"
//     },
//     {
//       "src": "icons/Icon-maskable-512.png",
//       "sizes": "512x512",
//       "type": "image/png",
//       "purpose": "maskable"
//     }
//   ]
// }

// lib/web/pwa_helper.dart
import 'dart:html' as html;
import 'package:flutter/foundation.dart';

class PwaHelper {
  static bool _isInstalled = false;
  static html.EventTarget? _deferredPrompt;

  /// ตรวจสอบว่า PWA ถูก install แล้วหรือยัง
  static bool get isInstalled => _isInstalled;

  /// ฟัง event สำหรับ install prompt
  static void listenForInstallPrompt() {
    if (!kIsWeb) return;

    html.window.addEventListener('beforeinstallprompt', (event) {
      event.preventDefault();
      _deferredPrompt = event;
    });

    html.window.addEventListener('appinstalled', (_) {
      _isInstalled = true;
      _deferredPrompt = null;
    });
  }

  /// แสดง install prompt
  static Future<bool> showInstallPrompt() async {
    if (_deferredPrompt == null) return false;

    (_deferredPrompt as dynamic).prompt();

    final outcome = await (_deferredPrompt as dynamic).userChoice;
    _deferredPrompt = null;

    return outcome['outcome'] == 'accepted';
  }

  /// ตรวจสอบว่าทำงานใน standalone mode หรือไม่
  static bool isRunningStandalone() {
    if (!kIsWeb) return false;

    return html.window.matchMedia('(display-mode: standalone)').matches ||
        (html.window.navigator as dynamic).standalone == true;
  }

  /// ตรวจสอบ network status
  static bool isOnline() {
    if (!kIsWeb) return true;
    return html.window.navigator.onLine ?? true;
  }

  /// ฟัง network changes
  static Stream<bool> networkStatusStream() {
    if (!kIsWeb) return Stream.value(true);

    final controller = StreamController<bool>.broadcast();

    html.window.addEventListener('online', (_) => controller.add(true));
    html.window.addEventListener('offline', (_) => controller.add(false));

    return controller.stream;
  }

  /// ลงทะเบียน service worker
  static Future<void> registerServiceWorker() async {
    if (!kIsWeb) return;

    if ('serviceWorker' in html.window.navigator) {
      try {
        await html.window.navigator.serviceWorker!
            .register('flutter_service_worker.js');
        print('Service Worker registered successfully');
      } catch (e) {
        print('Service Worker registration failed: $e');
      }
    }
  }
}

// Workaround for 'in' operator in Dart
extension _Contains on html.Navigator {
  bool operator [](String key) => js_util.hasProperty(this, key) as bool;
}
```

---

## ขั้นตอนที่ 2524: Service Worker Caching Strategies

```javascript
// web/flutter_service_worker.js
// Cache-First Strategy สำหรับ static assets

const CACHE_NAME = 'flutter-app-v1';
const RUNTIME_CACHE = 'flutter-runtime-v1';

const PRECACHE_ASSETS = [
  '/',
  '/index.html',
  '/main.dart.js',
  '/flutter.js',
  '/manifest.json',
  '/icons/Icon-192.png',
  '/icons/Icon-512.png',
];

// Install event - precache assets
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(PRECACHE_ASSETS);
    })
  );
  self.skipWaiting();
});

// Activate event - cleanup old caches
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_NAME && name !== RUNTIME_CACHE)
          .map((name) => caches.delete(name))
      );
    })
  );
  self.clients.claim();
});

// Fetch event - serve from cache or network
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);

  // API calls - Network First
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirst(request));
    return;
  }

  // Static assets - Cache First
  if (
    request.destination === 'image' ||
    request.destination === 'script' ||
    request.destination === 'style'
  ) {
    event.respondWith(cacheFirst(request));
    return;
  }

  // HTML pages - Stale While Revalidate
  if (request.mode === 'navigate') {
    event.respondWith(staleWhileRevalidate(request));
    return;
  }

  // Default - Network First
  event.respondWith(networkFirst(request));
});

async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;
  
  const response = await fetch(request);
  const cache = await caches.open(RUNTIME_CACHE);
  cache.put(request, response.clone());
  return response;
}

async function networkFirst(request) {
  try {
    const response = await fetch(request);
    const cache = await caches.open(RUNTIME_CACHE);
    cache.put(request, response.clone());
    return response;
  } catch (error) {
    const cached = await caches.match(request);
    if (cached) return cached;
    throw error;
  }
}

async function staleWhileRevalidate(request) {
  const cached = await caches.match(request);
  const networkPromise = fetch(request).then((response) => {
    caches.open(CACHE_NAME).then((cache) => cache.put(request, response.clone()));
    return response;
  });
  return cached || networkPromise;
}
```

```dart
// lib/web/offline_support.dart
import 'dart:html' as html;
import 'package:flutter/material.dart';
import 'package:flutter/foundation.dart';

class OfflineAwarePage extends StatefulWidget {
  final Widget child;
  final Widget? offlineWidget;

  const OfflineAwarePage({
    super.key,
    required this.child,
    this.offlineWidget,
  });

  @override
  State<OfflineAwarePage> createState() => _OfflineAwarePageState();
}

class _OfflineAwarePageState extends State<OfflineAwarePage> {
  bool _isOnline = true;

  @override
  void initState() {
    super.initState();
    if (kIsWeb) {
      _isOnline = html.window.navigator.onLine ?? true;
      html.window.addEventListener('online', _onOnline);
      html.window.addEventListener('offline', _onOffline);
    }
  }

  void _onOnline(html.Event _) => setState(() => _isOnline = true);
  void _onOffline(html.Event _) => setState(() => _isOnline = false);

  @override
  void dispose() {
    if (kIsWeb) {
      html.window.removeEventListener('online', _onOnline);
      html.window.removeEventListener('offline', _onOffline);
    }
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (!_isOnline && widget.offlineWidget != null) {
      return widget.offlineWidget!;
    }

    return Stack(
      children: [
        widget.child,
        if (!_isOnline)
          Positioned(
            top: 0,
            left: 0,
            right: 0,
            child: Container(
              color: Colors.red[700],
              padding: const EdgeInsets.symmetric(vertical: 4),
              child: const Text(
                'คุณกำลังออฟไลน์',
                textAlign: TextAlign.center,
                style: TextStyle(color: Colors.white, fontSize: 12),
              ),
            ),
          ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 2525: Web-Specific Platform Detection

```dart
// lib/web/platform_detector.dart
import 'package:flutter/foundation.dart';
import 'dart:html' as html show window;

enum WebBrowser { chrome, firefox, safari, edge, opera, unknown }
enum DeviceType { mobile, tablet, desktop }

class WebPlatformDetector {
  static final WebPlatformDetector _instance =
      WebPlatformDetector._internal();
  factory WebPlatformDetector() => _instance;
  WebPlatformDetector._internal();

  String get _userAgent {
    if (!kIsWeb) return '';
    return html.window.navigator.userAgent.toLowerCase();
  }

  /// ตรวจสอบ browser
  WebBrowser get browser {
    if (!kIsWeb) return WebBrowser.unknown;
    final ua = _userAgent;

    if (ua.contains('edg/')) return WebBrowser.edge;
    if (ua.contains('opr/') || ua.contains('opera')) return WebBrowser.opera;
    if (ua.contains('chrome') || ua.contains('chromium')) {
      return WebBrowser.chrome;
    }
    if (ua.contains('firefox')) return WebBrowser.firefox;
    if (ua.contains('safari')) return WebBrowser.safari;
    return WebBrowser.unknown;
  }

  /// ตรวจสอบ OS
  String get operatingSystem {
    if (!kIsWeb) return 'unknown';
    final ua = _userAgent;

    if (ua.contains('windows nt')) return 'Windows';
    if (ua.contains('mac os x') || ua.contains('macos')) return 'macOS';
    if (ua.contains('linux')) return 'Linux';
    if (ua.contains('android')) return 'Android';
    if (ua.contains('iphone') || ua.contains('ipad')) return 'iOS';
    return 'unknown';
  }

  /// ตรวจสอบ device type
  DeviceType get deviceType {
    if (!kIsWeb) return DeviceType.desktop;
    final ua = _userAgent;

    if (ua.contains('ipad') ||
        (ua.contains('tablet') && !ua.contains('mobile'))) {
      return DeviceType.tablet;
    }

    if (ua.contains('mobile') ||
        ua.contains('iphone') ||
        ua.contains('android')) {
      return DeviceType.mobile;
    }

    return DeviceType.desktop;
  }

  /// ตรวจสอบ touch support
  bool get hasTouchScreen {
    if (!kIsWeb) return false;
    return html.window.navigator.maxTouchPoints > 0;
  }

  /// ตรวจสอบ language
  String get preferredLanguage {
    if (!kIsWeb) return 'en';
    return html.window.navigator.language ?? 'en';
  }

  /// ตรวจสอบ screen resolution
  Map<String, int> get screenResolution {
    if (!kIsWeb) return {'width': 0, 'height': 0};
    return {
      'width': html.window.screen!.width!.toInt(),
      'height': html.window.screen!.height!.toInt(),
    };
  }

  /// ตรวจสอบ dark mode preference
  bool get prefersDarkMode {
    if (!kIsWeb) return false;
    return html.window
        .matchMedia('(prefers-color-scheme: dark)')
        .matches;
  }

  /// สรุปข้อมูล platform
  Map<String, dynamic> get summary => {
        'browser': browser.name,
        'os': operatingSystem,
        'deviceType': deviceType.name,
        'hasTouchScreen': hasTouchScreen,
        'language': preferredLanguage,
        'screenResolution': screenResolution,
        'prefersDarkMode': prefersDarkMode,
        'isWebPlatform': kIsWeb,
      };
}
```

---

## ขั้นตอนที่ 2526: WebAssembly (WASM) Integration

```dart
// lib/web/wasm_helper.dart
import 'dart:js_interop';
import 'package:flutter/foundation.dart';

// การ compile เป็น WASM ใช้คำสั่ง:
// flutter build web --wasm
// 
// หมายเหตุ: Flutter Web รองรับ WASM ตั้งแต่ Flutter 3.22

@JS('WebAssembly.instantiateStreaming')
external JSPromise<JSObject> _instantiateStreaming(
  JSObject response,
  JSObject imports,
);

@JS('WebAssembly.Instance')
external JSObject get _wasmInstance;

class WasmModule {
  JSObject? _instance;
  JSObject? _exports;

  /// โหลด WASM module
  Future<void> load(String wasmUrl) async {
    if (!kIsWeb) return;

    try {
      // ใน production ใช้ fetch API
      final response = await _fetchWasm(wasmUrl);
      final result = await _instantiateStreaming(response, {}.jsify() as JSObject)
          .toDart;
      _instance = result;
      print('WASM module loaded successfully');
    } catch (e) {
      print('Failed to load WASM: $e');
    }
  }

  JSObject _fetchWasm(String url) {
    // Mock implementation สำหรับ example
    throw UnimplementedError('Use fetch API in real implementation');
  }

  /// เรียกใช้ฟังก์ชันจาก WASM
  dynamic callFunction(String name, List<dynamic> args) {
    if (_exports == null) throw StateError('WASM module not loaded');
    // Call exported WASM function
    return null;
  }
}

// Example: การใช้ dart compile wasm
// dart compile wasm my_program.dart -o my_program.wasm
//
// จากนั้นโหลดใน HTML:
// <script>
//   const { instance } = await WebAssembly.instantiateStreaming(
//     fetch('my_program.wasm'),
//     {}
//   );
//   const result = instance.exports.myFunction(42);
// </script>

/// ตัวอย่าง Dart code ที่ compile เป็น WASM
/// dart compile wasm computation.dart -o computation.wasm
class ComputationModule {
  /// ฟังก์ชัน heavy computation ที่ WASM จะทำงานได้เร็วกว่า
  static int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
  }

  static List<int> sieveOfEratosthenes(int limit) {
    final sieve = List<bool>.filled(limit + 1, true);
    sieve[0] = sieve[1] = false;

    for (var i = 2; i * i <= limit; i++) {
      if (sieve[i]) {
        for (var j = i * i; j <= limit; j += i) {
          sieve[j] = false;
        }
      }
    }

    return [
      for (var i = 2; i <= limit; i++)
        if (sieve[i]) i
    ];
  }

  static double matrixMultiply(
    List<List<double>> a,
    List<List<double>> b,
  ) {
    final n = a.length;
    var sum = 0.0;

    for (var i = 0; i < n; i++) {
      for (var j = 0; j < n; j++) {
        for (var k = 0; k < n; k++) {
          sum += a[i][k] * b[k][j];
        }
      }
    }

    return sum;
  }
}
```

---

## ขั้นตอนที่ 2527: Web Workers ด้วย flutter_web_workers

```dart
// pubspec.yaml dependencies:
// flutter_web_workers: ^1.0.0

// lib/web/web_worker_service.dart
import 'dart:html' as html;
import 'dart:async';
import 'package:flutter/foundation.dart';

class WebWorkerService {
  html.Worker? _worker;
  final _messageController =
      StreamController<Map<String, dynamic>>.broadcast();
  int _messageId = 0;
  final Map<int, Completer<dynamic>> _pendingRequests = {};

  /// สร้าง Web Worker
  void initialize(String workerScriptUrl) {
    if (!kIsWeb) return;

    _worker = html.Worker(workerScriptUrl);

    _worker!.onMessage.listen((event) {
      final data = event.data as Map<String, dynamic>;
      final id = data['id'] as int?;

      if (id != null && _pendingRequests.containsKey(id)) {
        _pendingRequests[id]!.complete(data['result']);
        _pendingRequests.remove(id);
      } else {
        _messageController.add(data);
      }
    });

    _worker!.onError.listen((event) {
      print('Worker error: ${event.message}');
      for (final completer in _pendingRequests.values) {
        completer.completeError(event.message ?? 'Worker error');
      }
      _pendingRequests.clear();
    });
  }

  /// ส่ง message ไปยัง worker และรอ response
  Future<dynamic> sendRequest(
    String action,
    Map<String, dynamic> params,
  ) {
    if (_worker == null) {
      throw StateError('Worker not initialized');
    }

    final id = _messageId++;
    final completer = Completer<dynamic>();
    _pendingRequests[id] = completer;

    _worker!.postMessage({
      'id': id,
      'action': action,
      'params': params,
    });

    return completer.future.timeout(
      const Duration(seconds: 30),
      onTimeout: () {
        _pendingRequests.remove(id);
        throw TimeoutException('Worker request timed out', const Duration(seconds: 30));
      },
    );
  }

  /// รับ stream ของ messages จาก worker
  Stream<Map<String, dynamic>> get messages => _messageController.stream;

  /// ยกเลิก worker
  void terminate() {
    _worker?.terminate();
    _worker = null;
    _messageController.close();
    _pendingRequests.clear();
  }
}

// web/workers/computation_worker.js
// self.onmessage = function(event) {
//   const { id, action, params } = event.data;
//
//   let result;
//   switch (action) {
//     case 'fibonacci':
//       result = fibonacci(params.n);
//       break;
//     case 'processData':
//       result = processLargeDataset(params.data);
//       break;
//     default:
//       result = null;
//   }
//
//   self.postMessage({ id, result });
// };
//
// function fibonacci(n) {
//   if (n <= 1) return n;
//   return fibonacci(n - 1) + fibonacci(n - 2);
// }
//
// function processLargeDataset(data) {
//   return data.map(x => x * 2).filter(x => x > 10).reduce((a, b) => a + b, 0);
// }
```

---

## ขั้นตอนที่ 2528: ตัวอย่าง Flutter Web App ที่สมบูรณ์

```dart
// lib/main_web.dart
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'web/seo_helper.dart';
import 'web/structured_data.dart';
import 'web/pwa_helper.dart';
import 'web/platform_detector.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  if (kIsWeb) {
    // Setup SEO
    SeoHelper.updateMetaTags(
      title: 'Flutter Web App - Professional Course',
      description: 'Learn advanced Flutter Web development',
      keywords: 'flutter, web, dart, pwa, wasm',
      canonicalUrl: 'https://myapp.com',
    );

    // Add structured data
    StructuredDataHelper.addOrganizationData(
      name: 'Flutter Course',
      url: 'https://myapp.com',
      description: 'Professional Flutter Development Course',
    );

    // Register service worker
    await PwaHelper.registerServiceWorker();
    PwaHelper.listenForInstallPrompt();
  }

  runApp(const FlutterWebAdvancedApp());
}

class FlutterWebAdvancedApp extends StatelessWidget {
  const FlutterWebAdvancedApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Web Advanced',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      darkTheme: ThemeData.dark(useMaterial3: true),
      themeMode: WebPlatformDetector().prefersDarkMode
          ? ThemeMode.dark
          : ThemeMode.light,
      home: const WebDashboardPage(),
    );
  }
}

class WebDashboardPage extends StatefulWidget {
  const WebDashboardPage({super.key});

  @override
  State<WebDashboardPage> createState() => _WebDashboardPageState();
}

class _WebDashboardPageState extends State<WebDashboardPage> {
  final _detector = WebPlatformDetector();
  bool _canInstallPwa = false;

  @override
  void initState() {
    super.initState();
    _updateSeo();
  }

  void _updateSeo() {
    SeoHelper.updateMetaTags(
      title: 'Dashboard - Flutter Web App',
      description: 'Your personal Flutter Web dashboard',
    );
    StructuredDataHelper.addBreadcrumb([
      const BreadcrumbItem(name: 'Home', url: 'https://myapp.com'),
      const BreadcrumbItem(
          name: 'Dashboard', url: 'https://myapp.com/dashboard'),
    ]);
  }

  @override
  Widget build(BuildContext context) {
    final platformInfo = _detector.summary;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Flutter Web Advanced'),
        actions: [
          if (_canInstallPwa)
            IconButton(
              icon: const Icon(Icons.install_desktop),
              onPressed: () async {
                final installed = await PwaHelper.showInstallPrompt();
                if (installed) {
                  setState(() => _canInstallPwa = false);
                }
              },
              tooltip: 'Install App',
            ),
        ],
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text(
              'Platform Information',
              style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 16),
            ...platformInfo.entries.map(
              (e) => Card(
                child: ListTile(
                  leading: const Icon(Icons.info),
                  title: Text(e.key),
                  trailing: Text(
                    e.value.toString(),
                    style: const TextStyle(fontWeight: FontWeight.bold),
                  ),
                ),
              ),
            ),
            const SizedBox(height: 24),
            const Text(
              'PWA Status',
              style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
            ),
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Column(
                  children: [
                    ListTile(
                      leading: Icon(
                        PwaHelper.isRunningStandalone()
                            ? Icons.check_circle
                            : Icons.info,
                        color: PwaHelper.isRunningStandalone()
                            ? Colors.green
                            : Colors.orange,
                      ),
                      title: const Text('Standalone Mode'),
                      subtitle: Text(
                        PwaHelper.isRunningStandalone()
                            ? 'Running as PWA'
                            : 'Running in browser',
                      ),
                    ),
                    ListTile(
                      leading: Icon(
                        PwaHelper.isOnline()
                            ? Icons.wifi
                            : Icons.wifi_off,
                        color: PwaHelper.isOnline()
                            ? Colors.green
                            : Colors.red,
                      ),
                      title: const Text('Network Status'),
                      subtitle:
                          Text(PwaHelper.isOnline() ? 'Online' : 'Offline'),
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

**← [Part 65](part-65-flutter-desktop.md)**
**ต่อไป: [Part 67 →](part-67-dart-ffi.md)**
