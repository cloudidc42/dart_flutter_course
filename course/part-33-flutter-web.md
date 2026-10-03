# Part 33: Flutter Web
## ขั้นตอนที่ 1201-1240

---

## 🎯 เป้าหมายของ Part นี้

- Flutter Web setup และ deployment
- Web-specific widgets และ patterns
- Responsive layouts
- SEO considerations
- Web API integration

---

## ขั้นตอนที่ 1201: Flutter Web Setup

```bash
# สร้าง Flutter Web project
flutter create --platforms=web my_web_app
cd my_web_app

# Run บน web
flutter run -d chrome
flutter run -d web-server --web-port=8080

# Build สำหรับ production
flutter build web --release

# Build พร้อม renderer options
flutter build web --web-renderer canvaskit  # ดีสำหรับ graphics
flutter build web --web-renderer html        # ดีสำหรับ text/performance
flutter build web --web-renderer auto        # auto-select (default)

# output: build/web/
```

```yaml
# pubspec.yaml
name: my_web_app
environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  flutter_web_plugins:
    sdk: flutter
  url_strategy: ^0.2.0  # Clean URLs (ไม่มี /#/)
  go_router: ^13.0.0
  responsive_framework: ^1.1.1
```

---

## ขั้นตอนที่ 1202: Responsive Web Layout

```dart
import 'package:flutter/material.dart';
import 'package:responsive_framework/responsive_framework.dart';

// ─── Responsive Breakpoints ───
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      builder: (context, child) => ResponsiveBreakpoints.builder(
        child: child!,
        breakpoints: [
          const Breakpoint(start: 0, end: 450, name: MOBILE),
          const Breakpoint(start: 451, end: 800, name: TABLET),
          const Breakpoint(start: 801, end: 1200, name: DESKTOP),
          const Breakpoint(start: 1201, end: double.infinity, name: '4K'),
        ],
      ),
      home: const HomePage(),
    );
  }
}

// ─── Responsive Home Page ───
class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    bool isMobile = ResponsiveBreakpoints.of(context).isMobile;
    bool isTablet = ResponsiveBreakpoints.of(context).isTablet;
    bool isDesktop = ResponsiveBreakpoints.of(context).isDesktop;

    return Scaffold(
      // Drawer สำหรับ mobile, ใช้ Sidebar สำหรับ desktop
      drawer: isMobile ? const AppDrawer() : null,
      body: Row(
        children: [
          // Sidebar สำหรับ Desktop/Tablet
          if (!isMobile)
            SizedBox(
              width: isDesktop ? 250 : 200,
              child: const AppSidebar(),
            ),
          // Main content
          Expanded(
            child: Column(
              children: [
                // Top App Bar
                AppBar(
                  title: const Text('My Web App'),
                  automaticallyImplyLeading: isMobile,
                ),
                // Content area
                Expanded(
                  child: _buildContent(context, isMobile, isTablet, isDesktop),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildContent(BuildContext context, bool isMobile, bool isTablet, bool isDesktop) {
    int crossAxisCount = isMobile ? 1 : isTablet ? 2 : isDesktop ? 3 : 4;

    return GridView.builder(
      padding: EdgeInsets.all(isMobile ? 8 : 16),
      gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: crossAxisCount,
        crossAxisSpacing: 12,
        mainAxisSpacing: 12,
        childAspectRatio: 1.5,
      ),
      itemBuilder: (context, index) => Card(
        child: Center(child: Text('Item $index')),
      ),
      itemCount: 20,
    );
  }
}

class AppDrawer extends StatelessWidget {
  const AppDrawer({super.key});

  @override
  Widget build(BuildContext context) {
    return Drawer(
      child: ListView(
        children: const [
          DrawerHeader(
            decoration: BoxDecoration(color: Colors.blue),
            child: Text('Menu', style: TextStyle(color: Colors.white, fontSize: 24)),
          ),
          ListTile(leading: Icon(Icons.home), title: Text('Home')),
          ListTile(leading: Icon(Icons.person), title: Text('Profile')),
          ListTile(leading: Icon(Icons.settings), title: Text('Settings')),
        ],
      ),
    );
  }
}

class AppSidebar extends StatelessWidget {
  const AppSidebar({super.key});

  @override
  Widget build(BuildContext context) {
    return Container(
      decoration: BoxDecoration(
        color: Theme.of(context).colorScheme.surface,
        border: Border(
          right: BorderSide(color: Theme.of(context).dividerColor),
        ),
      ),
      child: Column(
        children: [
          const SizedBox(height: 24),
          const FlutterLogo(size: 48),
          const SizedBox(height: 24),
          ...[
            ('Home', Icons.home),
            ('Profile', Icons.person),
            ('Settings', Icons.settings),
            ('Help', Icons.help),
          ].map((item) => ListTile(
            leading: Icon(item.$2),
            title: Text(item.$1),
          )),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1203: Web URL Strategy

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import 'package:url_strategy/url_strategy.dart';

// ─── Clean URLs ───
void main() {
  // ลบ # ออกจาก URL
  usePathUrlStrategy();
  runApp(const MyApp());
}

// ─── GoRouter สำหรับ Web ───
final _router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomePage(),
    ),
    GoRoute(
      path: '/products',
      builder: (context, state) => const ProductListPage(),
      routes: [
        GoRoute(
          path: ':id',
          builder: (context, state) => ProductDetailPage(
            id: state.pathParameters['id']!,
          ),
        ),
      ],
    ),
    GoRoute(
      path: '/about',
      builder: (context, state) => const AboutPage(),
    ),
    // 404 page
    GoRoute(
      path: '/404',
      builder: (context, state) => const NotFoundPage(),
    ),
  ],
  errorBuilder: (context, state) => const NotFoundPage(),
  redirect: (context, state) {
    // Auth redirect
    bool isLoggedIn = AuthService.instance.isLoggedIn;
    bool goingToLogin = state.matchedLocation == '/login';

    if (!isLoggedIn && !goingToLogin) return '/login';
    if (isLoggedIn && goingToLogin) return '/';
    return null;
  },
);

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      title: 'Flutter Web App',
      routerConfig: _router,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
    );
  }
}

class ProductDetailPage extends StatelessWidget {
  final String id;
  const ProductDetailPage({super.key, required this.id});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Product $id')),
      body: Center(child: Text('Product ID: $id')),
    );
  }
}

class ProductListPage extends StatelessWidget {
  const ProductListPage({super.key});
  @override
  Widget build(BuildContext context) => const Scaffold(body: Center(child: Text('Products')));
}

class AboutPage extends StatelessWidget {
  const AboutPage({super.key});
  @override
  Widget build(BuildContext context) => const Scaffold(body: Center(child: Text('About')));
}

class NotFoundPage extends StatelessWidget {
  const NotFoundPage({super.key});
  @override
  Widget build(BuildContext context) => const Scaffold(body: Center(child: Text('404')));
}

class AuthService {
  static final AuthService instance = AuthService._();
  AuthService._();
  bool isLoggedIn = false;
}
```

---

## ขั้นตอนที่ 1204: Web-specific Features

```dart
import 'dart:html' as html;
import 'dart:ui_web' as ui;
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';

// ─── Platform check ───
bool get isWeb => kIsWeb;

// ─── Download file บน Web ───
void downloadFile(String content, String filename) {
  if (!kIsWeb) return;

  final bytes = html.Blob([content]);
  final url = html.Url.createObjectUrlFromBlob(bytes);
  final anchor = html.AnchorElement(href: url)
    ..setAttribute('download', filename)
    ..click();
  html.Url.revokeObjectUrl(url);
}

// ─── Open URL ───
void openUrl(String url) {
  if (kIsWeb) {
    html.window.open(url, '_blank');
  }
}

// ─── Copy to Clipboard ───
Future<void> copyToClipboard(String text) async {
  if (kIsWeb) {
    await html.window.navigator.clipboard?.writeText(text);
  }
}

// ─── Local Storage บน Web ───
class WebStorage {
  static String? get(String key) {
    if (!kIsWeb) return null;
    return html.window.localStorage[key];
  }

  static void set(String key, String value) {
    if (!kIsWeb) return;
    html.window.localStorage[key] = value;
  }

  static void remove(String key) {
    if (!kIsWeb) return;
    html.window.localStorage.remove(key);
  }
}

// ─── HtmlElementView: ฝัง HTML element ───
class EmbeddedMapView extends StatelessWidget {
  final String mapSrc;
  const EmbeddedMapView({super.key, required this.mapSrc});

  @override
  Widget build(BuildContext context) {
    if (!kIsWeb) {
      return const Center(child: Text('Maps only on web'));
    }

    // Register the view factory
    ui.platformViewRegistry.registerViewFactory(
      'map-view',
      (int viewId) {
        final iframe = html.IFrameElement()
          ..src = mapSrc
          ..style.width = '100%'
          ..style.height = '100%'
          ..style.border = 'none';
        return iframe;
      },
    );

    return const SizedBox(
      height: 300,
      child: HtmlElementView(viewType: 'map-view'),
    );
  }
}

// ─── Web Meta Tags (index.html) ───
/*
<!-- web/index.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="description" content="My Flutter Web App - Shop for products">
  <meta name="keywords" content="flutter, web, shop">
  <meta property="og:title" content="My Shop">
  <meta property="og:description" content="Best products online">
  <meta property="og:image" content="https://mysite.com/thumbnail.jpg">
  <title>My Flutter Web App</title>
  <link rel="icon" type="image/png" href="favicon.png"/>
</head>
<body>
  <script src="flutter.js" defer></script>
</body>
</html>
*/

// ─── Firebase Hosting Deploy ───
/*
# firebase.json
{
  "hosting": {
    "public": "build/web",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ],
    "headers": [
      {
        "source": "**/*.@(js|css)",
        "headers": [{"key": "Cache-Control", "value": "max-age=31536000"}]
      }
    ]
  }
}

# Deploy commands
flutter build web --release
firebase deploy --only hosting
*/
```

---

## ขั้นตอนที่ 1205: Web Performance

```dart
// ─── Lazy Loading Images ───
class LazyImage extends StatelessWidget {
  final String url;
  final double? width;
  final double? height;

  const LazyImage({
    super.key,
    required this.url,
    this.width,
    this.height,
  });

  @override
  Widget build(BuildContext context) {
    return Image.network(
      url,
      width: width,
      height: height,
      fit: BoxFit.cover,
      frameBuilder: (context, child, frame, wasSynchronouslyLoaded) {
        if (wasSynchronouslyLoaded) return child;
        return AnimatedSwitcher(
          duration: const Duration(milliseconds: 300),
          child: frame != null
              ? child
              : Container(
                  width: width,
                  height: height,
                  color: Colors.grey[200],
                  child: const Center(child: CircularProgressIndicator()),
                ),
        );
      },
      errorBuilder: (context, error, stackTrace) => Container(
        width: width,
        height: height,
        color: Colors.grey[200],
        child: const Icon(Icons.broken_image),
      ),
    );
  }
}

// ─── Code splitting / Deferred loading ───
// ใช้ dart:js_interop สำหรับ feature ที่ load แยก
import 'package:flutter/material.dart';

// Heavy widget ที่ load ช้า
class HeavyChartWidget extends StatelessWidget {
  const HeavyChartWidget({super.key});

  @override
  Widget build(BuildContext context) {
    return FutureBuilder(
      future: _loadChartLibrary(),
      builder: (context, snapshot) {
        if (snapshot.connectionState != ConnectionState.done) {
          return const Center(child: CircularProgressIndicator());
        }
        return const Text('Chart loaded!');
      },
    );
  }

  Future<void> _loadChartLibrary() async {
    // Simulate loading heavy library
    await Future.delayed(const Duration(milliseconds: 500));
  }
}
```

---

**← [Part 32 - Advanced Widgets](part-32-advanced-flutter-widgets.md)**

**ต่อไป: [Part 34 - Flutter Desktop →](part-34-flutter-desktop.md)**
