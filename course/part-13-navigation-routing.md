# Part 13: Navigation และ Routing
## ขั้นตอนที่ 401-440

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ Navigator.push/pop
- Named routes
- GoRouter package สำหรับ declarative routing
- Deep linking
- การส่งข้อมูลระหว่าง screens
- Bottom navigation และ Drawer

---

## ขั้นตอนที่ 401: Navigator.push และ pop

```dart
import 'package:flutter/material.dart';

// ─── Screen A ─── ส่งข้อมูลไป Screen B
class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Home')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () {
                // Push screen ใหม่
                Navigator.push(
                  context,
                  MaterialPageRoute(
                    builder: (_) => const DetailScreen(productId: '123', productName: 'Flutter Book'),
                  ),
                );
              },
              child: const Text('ไปหน้า Detail'),
            ),
            const SizedBox(height: 8),
            ElevatedButton(
              onPressed: () async {
                // Push และรอ result กลับมา
                final result = await Navigator.push<String>(
                  context,
                  MaterialPageRoute(builder: (_) => const InputScreen()),
                );
                
                if (result != null && context.mounted) {
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text('ได้รับ: $result')),
                  );
                }
              },
              child: const Text('รับข้อมูลจาก Screen'),
            ),
          ],
        ),
      ),
    );
  }
}

// ─── Screen B ─── รับข้อมูลจาก Screen A
class DetailScreen extends StatelessWidget {
  final String productId;
  final String productName;
  
  const DetailScreen({super.key, required this.productId, required this.productName});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(productName)),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('Product ID: $productId'),
            Text('Product Name: $productName'),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () => Navigator.pop(context),
              child: const Text('กลับ'),
            ),
          ],
        ),
      ),
    );
  }
}

// ─── Screen ที่ return ค่า ───
class InputScreen extends StatefulWidget {
  const InputScreen({super.key});
  
  @override
  State<InputScreen> createState() => _InputScreenState();
}

class _InputScreenState extends State<InputScreen> {
  final _controller = TextEditingController();
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Input')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            TextField(
              controller: _controller,
              decoration: const InputDecoration(labelText: 'พิมพ์อะไรสักอย่าง'),
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () {
                // ส่งค่ากลับ
                Navigator.pop(context, _controller.text);
              },
              child: const Text('ส่งกลับ'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 402: Named Routes

```dart
import 'package:flutter/material.dart';

// ─── Route names ───
class AppRoutes {
  static const home = '/';
  static const product = '/product';
  static const cart = '/cart';
  static const profile = '/profile';
  static const settings = '/settings';
}

// ─── App setup กับ Named Routes ───
class NamedRoutesApp extends StatelessWidget {
  const NamedRoutesApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Named Routes',
      initialRoute: AppRoutes.home,
      routes: {
        AppRoutes.home: (_) => const HomeNamedScreen(),
        AppRoutes.cart: (_) => const CartNamedScreen(),
        AppRoutes.profile: (_) => const ProfileScreen(),
        AppRoutes.settings: (_) => const SettingsScreen(),
      },
      // onGenerateRoute สำหรับ dynamic routes
      onGenerateRoute: (settings) {
        if (settings.name == AppRoutes.product) {
          final args = settings.arguments as Map<String, dynamic>?;
          return MaterialPageRoute(
            builder: (_) => ProductScreen(
              id: args?['id'] ?? '',
              name: args?['name'] ?? '',
            ),
          );
        }
        return null;
      },
      // หน้า 404
      onUnknownRoute: (settings) => MaterialPageRoute(
        builder: (_) => const NotFoundScreen(),
      ),
    );
  }
}

class HomeNamedScreen extends StatelessWidget {
  const HomeNamedScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Home')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () => Navigator.pushNamed(
                context,
                AppRoutes.product,
                arguments: {'id': 'p001', 'name': 'Flutter Book'},
              ),
              child: const Text('ไปหน้า Product'),
            ),
            ElevatedButton(
              onPressed: () => Navigator.pushNamed(context, AppRoutes.cart),
              child: const Text('ไปหน้า Cart'),
            ),
          ],
        ),
      ),
    );
  }
}

class ProductScreen extends StatelessWidget {
  final String id;
  final String name;
  
  const ProductScreen({super.key, required this.id, required this.name});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(name)),
      body: Center(child: Text('Product: $id - $name')),
    );
  }
}

class CartNamedScreen extends StatelessWidget {
  const CartNamedScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Cart')),
      body: const Center(child: Text('Cart Screen')),
    );
  }
}

class ProfileScreen extends StatelessWidget {
  const ProfileScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Profile')),
      body: const Center(child: Text('Profile Screen')),
    );
  }
}

class SettingsScreen extends StatelessWidget {
  const SettingsScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Settings')),
      body: const Center(child: Text('Settings Screen')),
    );
  }
}

class NotFoundScreen extends StatelessWidget {
  const NotFoundScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('ไม่พบหน้านี้')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Text('404 - Page Not Found', style: TextStyle(fontSize: 24)),
            ElevatedButton(
              onPressed: () => Navigator.pushReplacementNamed(context, AppRoutes.home),
              child: const Text('กลับหน้าแรก'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 403: GoRouter

```dart
// pubspec.yaml:
// dependencies:
//   go_router: ^13.0.0

import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

// ─── GoRouter setup ───
final GoRouter _router = GoRouter(
  initialLocation: '/',
  debugLogDiagnostics: true,
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const GoHomeScreen(),
      routes: [
        GoRoute(
          path: 'product/:id',
          builder: (context, state) {
            String productId = state.pathParameters['id']!;
            String? name = state.uri.queryParameters['name'];
            return GoProductScreen(productId: productId, productName: name ?? 'Product');
          },
        ),
      ],
    ),
    GoRoute(
      path: '/cart',
      builder: (context, state) => const GoCartScreen(),
    ),
    GoRoute(
      path: '/profile/:userId',
      builder: (context, state) {
        return GoProfileScreen(userId: state.pathParameters['userId']!);
      },
    ),
    ShellRoute(
      builder: (context, state, child) => MainShell(child: child),
      routes: [
        GoRoute(path: '/home', builder: (ctx, s) => const Tab1Screen()),
        GoRoute(path: '/explore', builder: (ctx, s) => const Tab2Screen()),
        GoRoute(path: '/saved', builder: (ctx, s) => const Tab3Screen()),
      ],
    ),
  ],
  errorBuilder: (context, state) => Scaffold(
    body: Center(child: Text('Error: ${state.error}')),
  ),
);

class GoRouterApp extends StatelessWidget {
  const GoRouterApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: _router,
      title: 'GoRouter App',
      theme: ThemeData(colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue)),
    );
  }
}

class GoHomeScreen extends StatelessWidget {
  const GoHomeScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('GoRouter Home')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () => context.go('/product/123?name=Flutter+Book'),
              child: const Text('ไปหน้า Product (go)'),
            ),
            ElevatedButton(
              onPressed: () => context.push('/product/456?name=Dart+Guide'),
              child: const Text('Push Product'),
            ),
            ElevatedButton(
              onPressed: () => context.go('/cart'),
              child: const Text('ไปหน้า Cart'),
            ),
          ],
        ),
      ),
    );
  }
}

class GoProductScreen extends StatelessWidget {
  final String productId;
  final String productName;
  
  const GoProductScreen({super.key, required this.productId, required this.productName});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(productName),
        leading: IconButton(
          icon: const Icon(Icons.arrow_back),
          onPressed: () => context.pop(),
        ),
      ),
      body: Center(child: Text('Product: $productId')),
    );
  }
}

class GoCartScreen extends StatelessWidget {
  const GoCartScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Cart')),
      body: Center(
        child: ElevatedButton(
          onPressed: () => context.go('/'),
          child: const Text('กลับหน้าแรก'),
        ),
      ),
    );
  }
}

class GoProfileScreen extends StatelessWidget {
  final String userId;
  
  const GoProfileScreen({super.key, required this.userId});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Profile: $userId')),
      body: Center(child: Text('User ID: $userId')),
    );
  }
}

// ─── ShellRoute สำหรับ Bottom Navigation ───
class MainShell extends StatelessWidget {
  final Widget child;
  
  const MainShell({super.key, required this.child});
  
  int _getCurrentIndex(BuildContext context) {
    String location = GoRouterState.of(context).uri.path;
    if (location.startsWith('/home')) return 0;
    if (location.startsWith('/explore')) return 1;
    if (location.startsWith('/saved')) return 2;
    return 0;
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: child,
      bottomNavigationBar: NavigationBar(
        selectedIndex: _getCurrentIndex(context),
        onDestinationSelected: (index) {
          switch (index) {
            case 0: context.go('/home');
            case 1: context.go('/explore');
            case 2: context.go('/saved');
          }
        },
        destinations: const [
          NavigationDestination(icon: Icon(Icons.home_outlined), selectedIcon: Icon(Icons.home), label: 'หน้าแรก'),
          NavigationDestination(icon: Icon(Icons.explore_outlined), selectedIcon: Icon(Icons.explore), label: 'สำรวจ'),
          NavigationDestination(icon: Icon(Icons.bookmark_outline), selectedIcon: Icon(Icons.bookmark), label: 'บันทึก'),
        ],
      ),
    );
  }
}

class Tab1Screen extends StatelessWidget {
  const Tab1Screen({super.key});
  @override
  Widget build(BuildContext context) => const Center(child: Text('Home Tab'));
}

class Tab2Screen extends StatelessWidget {
  const Tab2Screen({super.key});
  @override
  Widget build(BuildContext context) => const Center(child: Text('Explore Tab'));
}

class Tab3Screen extends StatelessWidget {
  const Tab3Screen({super.key});
  @override
  Widget build(BuildContext context) => const Center(child: Text('Saved Tab'));
}
```

---

## ขั้นตอนที่ 404: Navigation Transitions

```dart
import 'package:flutter/material.dart';

// ─── Custom Transitions ───
class SlideRoute extends PageRouteBuilder {
  final Widget page;
  final AxisDirection direction;
  
  SlideRoute({required this.page, this.direction = AxisDirection.left})
      : super(
          pageBuilder: (context, animation, secondaryAnimation) => page,
          transitionsBuilder: (context, animation, secondaryAnimation, child) {
            Offset begin = switch (direction) {
              AxisDirection.left => const Offset(1.0, 0.0),
              AxisDirection.right => const Offset(-1.0, 0.0),
              AxisDirection.up => const Offset(0.0, 1.0),
              AxisDirection.down => const Offset(0.0, -1.0),
            };
            
            Curve curve = Curves.easeInOut;
            Animatable<Offset> tween = Tween(begin: begin, end: Offset.zero).chain(CurveTween(curve: curve));
            
            return SlideTransition(
              position: animation.drive(tween),
              child: child,
            );
          },
          transitionDuration: const Duration(milliseconds: 300),
        );
}

class FadeRoute extends PageRouteBuilder {
  final Widget page;
  
  FadeRoute({required this.page})
      : super(
          pageBuilder: (context, animation, secondaryAnimation) => page,
          transitionsBuilder: (context, animation, secondaryAnimation, child) {
            return FadeTransition(opacity: animation, child: child);
          },
          transitionDuration: const Duration(milliseconds: 400),
        );
}

class ScaleRoute extends PageRouteBuilder {
  final Widget page;
  
  ScaleRoute({required this.page})
      : super(
          pageBuilder: (context, animation, secondaryAnimation) => page,
          transitionsBuilder: (context, animation, secondaryAnimation, child) {
            Curve curve = Curves.bounceOut;
            Animatable<double> tween = Tween(begin: 0.0, end: 1.0).chain(CurveTween(curve: curve));
            return ScaleTransition(scale: animation.drive(tween), child: child);
          },
          transitionDuration: const Duration(milliseconds: 500),
        );
}

class TransitionDemo extends StatelessWidget {
  const TransitionDemo({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Custom Transitions')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () => Navigator.push(context, SlideRoute(page: const TargetScreen())),
              child: const Text('Slide Transition'),
            ),
            ElevatedButton(
              onPressed: () => Navigator.push(context, FadeRoute(page: const TargetScreen())),
              child: const Text('Fade Transition'),
            ),
            ElevatedButton(
              onPressed: () => Navigator.push(context, ScaleRoute(page: const TargetScreen())),
              child: const Text('Scale Transition'),
            ),
          ],
        ),
      ),
    );
  }
}

class TargetScreen extends StatelessWidget {
  const TargetScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Target Screen')),
      body: Center(
        child: ElevatedButton(
          onPressed: () => Navigator.pop(context),
          child: const Text('กลับ'),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 405-420: Drawer และ Bottom Sheet

```dart
import 'package:flutter/material.dart';

class DrawerBottomSheetDemo extends StatefulWidget {
  const DrawerBottomSheetDemo({super.key});
  
  @override
  State<DrawerBottomSheetDemo> createState() => _DrawerBottomSheetDemoState();
}

class _DrawerBottomSheetDemoState extends State<DrawerBottomSheetDemo> {
  int _selectedIndex = 0;
  final List<String> _pages = ['หน้าแรก', 'ค้นหา', 'โปรไฟล์'];
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(_pages[_selectedIndex])),
      
      drawer: NavigationDrawer(
        selectedIndex: _selectedIndex,
        onDestinationSelected: (index) {
          setState(() => _selectedIndex = index);
          Navigator.pop(context);
        },
        children: [
          // Drawer Header
          Padding(
            padding: const EdgeInsets.fromLTRB(28, 16, 16, 10),
            child: Row(
              children: [
                const CircleAvatar(
                  radius: 28,
                  backgroundColor: Colors.blue,
                  child: Text('A', style: TextStyle(color: Colors.white, fontSize: 24)),
                ),
                const SizedBox(width: 12),
                const Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text('Alice Developer', style: TextStyle(fontWeight: FontWeight.bold)),
                    Text('alice@example.com', style: TextStyle(fontSize: 12, color: Colors.grey)),
                  ],
                ),
              ],
            ),
          ),
          const Divider(indent: 28, endIndent: 28),
          
          // Navigation items
          const NavigationDrawerDestination(
            icon: Icon(Icons.home_outlined),
            selectedIcon: Icon(Icons.home),
            label: Text('หน้าแรก'),
          ),
          const NavigationDrawerDestination(
            icon: Icon(Icons.search_outlined),
            selectedIcon: Icon(Icons.search),
            label: Text('ค้นหา'),
          ),
          const NavigationDrawerDestination(
            icon: Icon(Icons.person_outline),
            selectedIcon: Icon(Icons.person),
            label: Text('โปรไฟล์'),
          ),
          
          const Divider(indent: 28, endIndent: 28),
          
          // Settings (non-navigation)
          ListTile(
            leading: const Icon(Icons.settings_outlined),
            title: const Text('การตั้งค่า'),
            onTap: () {},
          ),
          ListTile(
            leading: const Icon(Icons.logout),
            title: const Text('ออกจากระบบ'),
            onTap: () {},
          ),
        ],
      ),
      
      body: Center(child: Text('${_pages[_selectedIndex]} Content')),
      
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showBottomSheet(context),
        child: const Icon(Icons.add),
      ),
    );
  }
  
  void _showBottomSheet(BuildContext context) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      useRootNavigator: true,
      builder: (ctx) => DraggableScrollableSheet(
        initialChildSize: 0.5,
        minChildSize: 0.25,
        maxChildSize: 0.9,
        expand: false,
        builder: (_, scrollController) => Container(
          decoration: const BoxDecoration(
            borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
          ),
          child: Column(
            children: [
              const SizedBox(height: 8),
              Container(
                width: 40,
                height: 4,
                decoration: BoxDecoration(
                  color: Colors.grey[400],
                  borderRadius: BorderRadius.circular(2),
                ),
              ),
              const SizedBox(height: 16),
              const Text('Bottom Sheet', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
              Expanded(
                child: ListView.builder(
                  controller: scrollController,
                  itemCount: 20,
                  itemBuilder: (ctx, i) => ListTile(
                    leading: const Icon(Icons.item_a, size: 20),
                    title: Text('Item ${i + 1}'),
                  ),
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

## ขั้นตอนที่ 421-440: โปรเจกต์ - Multi-Screen App

```dart
// social_app.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

void main() => runApp(const SocialApp());

// ─── Router ───
final _router = GoRouter(
  initialLocation: '/feed',
  routes: [
    ShellRoute(
      builder: (ctx, state, child) => AppShell(child: child),
      routes: [
        GoRoute(path: '/feed', builder: (c, s) => const FeedScreen()),
        GoRoute(path: '/search', builder: (c, s) => const SearchScreen()),
        GoRoute(path: '/create', builder: (c, s) => const CreatePostScreen()),
        GoRoute(path: '/notifications', builder: (c, s) => const NotificationsScreen()),
        GoRoute(path: '/profile', builder: (c, s) => const UserProfileScreen(userId: 'me')),
      ],
    ),
    GoRoute(
      path: '/post/:id',
      builder: (ctx, state) => PostDetailScreen(postId: state.pathParameters['id']!),
    ),
    GoRoute(
      path: '/user/:id',
      builder: (ctx, state) => UserProfileScreen(userId: state.pathParameters['id']!),
    ),
  ],
);

class SocialApp extends StatelessWidget {
  const SocialApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: _router,
      title: 'Social App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.pink),
        useMaterial3: true,
      ),
    );
  }
}

// ─── Shell ───
class AppShell extends StatelessWidget {
  final Widget child;
  
  const AppShell({super.key, required this.child});
  
  int _indexFromLocation(String location) {
    if (location.startsWith('/feed')) return 0;
    if (location.startsWith('/search')) return 1;
    if (location.startsWith('/create')) return 2;
    if (location.startsWith('/notifications')) return 3;
    if (location.startsWith('/profile')) return 4;
    return 0;
  }
  
  @override
  Widget build(BuildContext context) {
    String location = GoRouterState.of(context).uri.path;
    int index = _indexFromLocation(location);
    
    return Scaffold(
      body: child,
      bottomNavigationBar: NavigationBar(
        selectedIndex: index,
        onDestinationSelected: (i) {
          switch (i) {
            case 0: context.go('/feed');
            case 1: context.go('/search');
            case 2: context.go('/create');
            case 3: context.go('/notifications');
            case 4: context.go('/profile');
          }
        },
        destinations: const [
          NavigationDestination(icon: Icon(Icons.home_outlined), selectedIcon: Icon(Icons.home), label: 'Feed'),
          NavigationDestination(icon: Icon(Icons.search), label: 'Search'),
          NavigationDestination(icon: Icon(Icons.add_box_outlined), selectedIcon: Icon(Icons.add_box), label: 'Create'),
          NavigationDestination(icon: Icon(Icons.notifications_outlined), selectedIcon: Icon(Icons.notifications), label: 'Activity'),
          NavigationDestination(icon: Icon(Icons.person_outline), selectedIcon: Icon(Icons.person), label: 'Profile'),
        ],
      ),
    );
  }
}

// ─── Screens ───
class Post {
  final String id;
  final String username;
  final String content;
  final String imageUrl;
  final int likes;
  final int comments;
  
  const Post({
    required this.id, required this.username, required this.content,
    required this.imageUrl, required this.likes, required this.comments,
  });
}

final posts = [
  const Post(id: '1', username: '@alice_dev', content: 'เพิ่งเรียน Flutter เสร็จ! 🚀 #flutter #dart', imageUrl: 'https://via.placeholder.com/400', likes: 124, comments: 18),
  const Post(id: '2', username: '@bob_code', content: 'Dart 3 มีฟีเจอร์เจ๋งมากเลย pattern matching ช่วยได้เยอะ #dart', imageUrl: 'https://via.placeholder.com/400', likes: 89, comments: 12),
  const Post(id: '3', username: '@charlie_pro', content: 'สร้าง app แรกสำเร็จแล้ว! 🎉 รู้สึกดีมาก', imageUrl: 'https://via.placeholder.com/400', likes: 256, comments: 34),
];

class FeedScreen extends StatelessWidget {
  const FeedScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Feed'),
        actions: [
          IconButton(icon: const Icon(Icons.message_outlined), onPressed: () {}),
        ],
      ),
      body: ListView.builder(
        itemCount: posts.length,
        itemBuilder: (ctx, i) => PostCard(post: posts[i]),
      ),
    );
  }
}

class PostCard extends StatelessWidget {
  final Post post;
  
  const PostCard({super.key, required this.post});
  
  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        ListTile(
          leading: GestureDetector(
            onTap: () => context.push('/user/${post.username}'),
            child: CircleAvatar(child: Text(post.username[1].toUpperCase())),
          ),
          title: GestureDetector(
            onTap: () => context.push('/user/${post.username}'),
            child: Text(post.username, style: const TextStyle(fontWeight: FontWeight.bold)),
          ),
          subtitle: const Text('2 ชั่วโมงที่แล้ว'),
          trailing: IconButton(icon: const Icon(Icons.more_vert), onPressed: () {}),
        ),
        GestureDetector(
          onTap: () => context.push('/post/${post.id}'),
          child: Image.network(
            post.imageUrl,
            width: double.infinity,
            height: 300,
            fit: BoxFit.cover,
            errorBuilder: (c, e, s) => Container(height: 200, color: Colors.grey[200]),
          ),
        ),
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 8),
          child: Row(
            children: [
              IconButton(icon: const Icon(Icons.favorite_border), onPressed: () {}),
              Text('${post.likes}'),
              const SizedBox(width: 8),
              IconButton(icon: const Icon(Icons.chat_bubble_outline), onPressed: () {}),
              Text('${post.comments}'),
              const Spacer(),
              IconButton(icon: const Icon(Icons.bookmark_border), onPressed: () {}),
            ],
          ),
        ),
        Padding(
          padding: const EdgeInsets.fromLTRB(16, 0, 16, 12),
          child: Text(post.content),
        ),
        const Divider(),
      ],
    );
  }
}

class SearchScreen extends StatelessWidget {
  const SearchScreen({super.key});
  
  @override
  Widget build(BuildContext context) => const Scaffold(
    body: Center(child: Text('Search Screen')),
  );
}

class CreatePostScreen extends StatelessWidget {
  const CreatePostScreen({super.key});
  
  @override
  Widget build(BuildContext context) => const Scaffold(
    body: Center(child: Text('Create Post Screen')),
  );
}

class NotificationsScreen extends StatelessWidget {
  const NotificationsScreen({super.key});
  
  @override
  Widget build(BuildContext context) => const Scaffold(
    body: Center(child: Text('Notifications Screen')),
  );
}

class UserProfileScreen extends StatelessWidget {
  final String userId;
  
  const UserProfileScreen({super.key, required this.userId});
  
  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: Text('Profile: $userId')),
    body: Center(child: Text('User: $userId')),
  );
}

class PostDetailScreen extends StatelessWidget {
  final String postId;
  
  const PostDetailScreen({super.key, required this.postId});
  
  @override
  Widget build(BuildContext context) {
    Post? post = posts.where((p) => p.id == postId).firstOrNull;
    
    if (post == null) {
      return Scaffold(
        appBar: AppBar(title: const Text('ไม่พบโพสต์')),
        body: const Center(child: Text('Post not found')),
      );
    }
    
    return Scaffold(
      appBar: AppBar(title: Text(post.username)),
      body: Column(
        children: [
          Image.network(
            post.imageUrl,
            width: double.infinity,
            height: 300,
            fit: BoxFit.cover,
          ),
          Padding(
            padding: const EdgeInsets.all(16),
            child: Text(post.content, style: const TextStyle(fontSize: 16)),
          ),
        ],
      ),
    );
  }
}
```

---

**← [Part 12 - State Management](part-12-state-management.md)**

**ต่อไป: [Part 14 - HTTP/REST API →](part-14-http-rest-api.md)**
