# Part 36: Deep Linking & Push Notifications (FCM)
## ขั้นตอนที่ 1321-1360

---

## 🎯 เป้าหมายของ Part นี้

- App Links / Universal Links
- Deep link handling
- Firebase Cloud Messaging (FCM)
- Local notifications
- Notification handling

---

## ขั้นตอนที่ 1321: Deep Linking Setup

```yaml
# pubspec.yaml
dependencies:
  go_router: ^13.0.0
  firebase_messaging: ^15.0.0
  flutter_local_notifications: ^17.0.0
  firebase_core: ^3.0.0
```

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<activity android:name=".MainActivity">
  <!-- Deep Links -->
  <intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <category android:name="android.intent.category.BROWSABLE"/>
    <!-- https://yourapp.com/products/123 -->
    <data android:scheme="https"
          android:host="yourapp.com"
          android:pathPrefix="/products"/>
  </intent-filter>
  
  <!-- Custom Scheme Links -->
  <intent-filter>
    <action android:name="android.intent.action.VIEW"/>
    <category android:name="android.intent.category.DEFAULT"/>
    <category android:name="android.intent.category.BROWSABLE"/>
    <!-- myapp://products/123 -->
    <data android:scheme="myapp"/>
  </intent-filter>
</activity>
```

```xml
<!-- ios/Runner/Info.plist -->
<!-- Custom URL scheme -->
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>myapp</string>
    </array>
  </dict>
</array>

<!-- Associated Domains (Universal Links) -->
<key>com.apple.developer.associated-domains</key>
<array>
  <string>applinks:yourapp.com</string>
</array>
```

---

## ขั้นตอนที่ 1322: Deep Link Handler

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

// ─── Router กับ Deep Links ───
class AppRouter {
  static final GoRouter router = GoRouter(
    initialLocation: '/',
    routes: [
      GoRoute(path: '/', builder: (_, __) => const HomeScreen()),
      GoRoute(
        path: '/products',
        builder: (_, __) => const ProductListScreen(),
        routes: [
          GoRoute(
            path: ':productId',
            builder: (context, state) => ProductDetailScreen(
              productId: state.pathParameters['productId']!,
            ),
          ),
        ],
      ),
      GoRoute(
        path: '/orders/:orderId',
        builder: (context, state) => OrderDetailScreen(
          orderId: state.pathParameters['orderId']!,
        ),
      ),
      GoRoute(
        path: '/promo/:code',
        builder: (context, state) => PromoScreen(
          code: state.pathParameters['code']!,
        ),
      ),
      GoRoute(
        path: '/profile',
        builder: (_, __) => const ProfileScreen(),
      ),
    ],
    // Custom redirect logic
    redirect: (context, state) async {
      // Handle auth state
      bool isLoggedIn = await AuthManager.isLoggedIn();
      bool requiresAuth = _requiresAuth(state.matchedLocation);

      if (requiresAuth && !isLoggedIn) {
        return '/login?redirect=${Uri.encodeComponent(state.uri.toString())}';
      }
      return null;
    },
  );

  static bool _requiresAuth(String path) {
    const protectedPaths = ['/profile', '/orders'];
    return protectedPaths.any((p) => path.startsWith(p));
  }
}

class AuthManager {
  static Future<bool> isLoggedIn() async => true;
}

// Screens
class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});
  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Home')),
    body: ListView(
      children: [
        ListTile(title: const Text('Products'), onTap: () => context.go('/products')),
        ListTile(title: const Text('My Orders'), onTap: () => context.go('/orders/123')),
        ListTile(title: const Text('Promo'), onTap: () => context.go('/promo/SAVE20')),
      ],
    ),
  );
}

class ProductListScreen extends StatelessWidget {
  const ProductListScreen({super.key});
  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Products')),
    body: ListView.builder(
      itemCount: 5,
      itemBuilder: (context, i) => ListTile(
        title: Text('Product $i'),
        onTap: () => context.go('/products/$i'),
      ),
    ),
  );
}

class ProductDetailScreen extends StatelessWidget {
  final String productId;
  const ProductDetailScreen({super.key, required this.productId});
  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: Text('Product $productId')),
    body: Center(child: Text('Product ID: $productId')),
  );
}

class OrderDetailScreen extends StatelessWidget {
  final String orderId;
  const OrderDetailScreen({super.key, required this.orderId});
  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: Text('Order $orderId')),
    body: Center(child: Text('Order: $orderId')),
  );
}

class PromoScreen extends StatelessWidget {
  final String code;
  const PromoScreen({super.key, required this.code});
  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: Text('Promo: $code')),
    body: Center(child: Text('Code: $code')),
  );
}

class ProfileScreen extends StatelessWidget {
  const ProfileScreen({super.key});
  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Profile')),
    body: const Center(child: Text('Profile')),
  );
}
```

---

## ขั้นตอนที่ 1323: Firebase Cloud Messaging (FCM)

```dart
import 'dart:convert';
import 'dart:io';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter/material.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

// ─── Background Message Handler (top-level function) ───
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp();
  print('Background message: ${message.messageId}');
}

// ─── Notification Service ───
class NotificationService {
  static final NotificationService _instance = NotificationService._();
  factory NotificationService() => _instance;
  NotificationService._();

  final FirebaseMessaging _messaging = FirebaseMessaging.instance;
  final FlutterLocalNotificationsPlugin _localNotifications =
      FlutterLocalNotificationsPlugin();

  String? fcmToken;
  final StreamController<Map<String, dynamic>> _notificationStream =
      StreamController.broadcast();
  Stream<Map<String, dynamic>> get notifications => _notificationStream.stream;

  Future<void> initialize() async {
    // Background handler
    FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);

    // Request permissions
    await _requestPermissions();

    // Setup local notifications
    await _setupLocalNotifications();

    // Get FCM token
    fcmToken = await _messaging.getToken();
    print('FCM Token: $fcmToken');

    // Token refresh
    _messaging.onTokenRefresh.listen((token) {
      fcmToken = token;
      // Send to backend
      _sendTokenToServer(token);
    });

    // Foreground messages
    FirebaseMessaging.onMessage.listen(_handleForegroundMessage);

    // App opened from notification (background)
    FirebaseMessaging.onMessageOpenedApp.listen(_handleNotificationOpen);

    // App opened from terminated state
    RemoteMessage? initialMessage = await _messaging.getInitialMessage();
    if (initialMessage != null) {
      _handleNotificationOpen(initialMessage);
    }
  }

  Future<void> _requestPermissions() async {
    NotificationSettings settings = await _messaging.requestPermission(
      alert: true,
      announcement: false,
      badge: true,
      carPlay: false,
      criticalAlert: false,
      provisional: false,
      sound: true,
    );

    print('Permission status: ${settings.authorizationStatus}');
  }

  Future<void> _setupLocalNotifications() async {
    // Android channel
    const AndroidNotificationChannel channel = AndroidNotificationChannel(
      'high_importance',
      'High Importance Notifications',
      description: 'สำหรับ notifications สำคัญ',
      importance: Importance.high,
    );

    await _localNotifications
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(channel);

    // iOS
    await _localNotifications
        .resolvePlatformSpecificImplementation<
            IOSFlutterLocalNotificationsPlugin>()
        ?.requestPermissions(alert: true, badge: true, sound: true);

    // Initialize
    const InitializationSettings initSettings = InitializationSettings(
      android: AndroidInitializationSettings('@mipmap/ic_launcher'),
      iOS: DarwinInitializationSettings(),
    );

    await _localNotifications.initialize(
      initSettings,
      onDidReceiveNotificationResponse: (details) {
        if (details.payload != null) {
          Map<String, dynamic> data = jsonDecode(details.payload!);
          _notificationStream.add(data);
        }
      },
    );
  }

  Future<void> _handleForegroundMessage(RemoteMessage message) async {
    RemoteNotification? notification = message.notification;
    if (notification == null) return;

    // แสดง local notification เมื่อ app อยู่ foreground
    await _localNotifications.show(
      notification.hashCode,
      notification.title,
      notification.body,
      NotificationDetails(
        android: AndroidNotificationDetails(
          'high_importance',
          'High Importance Notifications',
          channelDescription: 'สำหรับ notifications สำคัญ',
          importance: Importance.high,
          priority: Priority.high,
          icon: '@mipmap/ic_launcher',
        ),
        iOS: const DarwinNotificationDetails(
          presentAlert: true,
          presentBadge: true,
          presentSound: true,
        ),
      ),
      payload: jsonEncode(message.data),
    );
  }

  void _handleNotificationOpen(RemoteMessage message) {
    _notificationStream.add(message.data);
  }

  void _sendTokenToServer(String token) {
    // POST token to your backend
    print('Sending token to server: $token');
  }

  // Subscribe to topic
  Future<void> subscribeToTopic(String topic) async {
    await _messaging.subscribeToTopic(topic);
  }

  Future<void> unsubscribeFromTopic(String topic) async {
    await _messaging.unsubscribeFromTopic(topic);
  }

  void dispose() {
    _notificationStream.close();
  }
}

// ─── main.dart ───
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  await NotificationService().initialize();
  runApp(const MyApp());
}

class MyApp extends StatefulWidget {
  const MyApp({super.key});

  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  @override
  void initState() {
    super.initState();
    // Listen for notification taps
    NotificationService().notifications.listen((data) {
      _handleNotificationNavigation(data);
    });
  }

  void _handleNotificationNavigation(Map<String, dynamic> data) {
    String? type = data['type'];
    String? id = data['id'];

    switch (type) {
      case 'product':
        AppRouter.router.go('/products/$id');
        break;
      case 'order':
        AppRouter.router.go('/orders/$id');
        break;
      case 'promo':
        AppRouter.router.go('/promo/$id');
        break;
    }
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: AppRouter.router,
      title: 'FCM Demo',
    );
  }
}
```

---

## ขั้นตอนที่ 1324: Local Notifications

```dart
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/timezone.dart' as tz;
import 'package:timezone/data/latest.dart' as tz;

class LocalNotificationService {
  static final LocalNotificationService _instance = LocalNotificationService._();
  factory LocalNotificationService() => _instance;
  LocalNotificationService._();

  final FlutterLocalNotificationsPlugin _plugin = FlutterLocalNotificationsPlugin();

  Future<void> init() async {
    tz.initializeTimeZones();

    const initSettings = InitializationSettings(
      android: AndroidInitializationSettings('@mipmap/ic_launcher'),
      iOS: DarwinInitializationSettings(
        requestAlertPermission: true,
        requestBadgePermission: true,
        requestSoundPermission: true,
      ),
    );

    await _plugin.initialize(initSettings);
  }

  // ─── Show immediate notification ───
  Future<void> showNotification({
    required int id,
    required String title,
    required String body,
    String? payload,
  }) async {
    await _plugin.show(
      id,
      title,
      body,
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'default',
          'Default',
          importance: Importance.high,
          priority: Priority.high,
        ),
        iOS: DarwinNotificationDetails(),
      ),
      payload: payload,
    );
  }

  // ─── Scheduled notification ───
  Future<void> scheduleNotification({
    required int id,
    required String title,
    required String body,
    required DateTime scheduledDate,
    String? payload,
  }) async {
    await _plugin.zonedSchedule(
      id,
      title,
      body,
      tz.TZDateTime.from(scheduledDate, tz.local),
      const NotificationDetails(
        android: AndroidNotificationDetails('scheduled', 'Scheduled'),
        iOS: DarwinNotificationDetails(),
      ),
      payload: payload,
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      uiLocalNotificationDateInterpretation:
          UILocalNotificationDateInterpretation.absoluteTime,
    );
  }

  // ─── Periodic notification ───
  Future<void> showDailyNotification({
    required int id,
    required String title,
    required String body,
    required Time time, // flutter_local_notifications Time
  }) async {
    await _plugin.periodicallyShowWithDuration(
      id,
      title,
      body,
      const Duration(days: 1),
      const NotificationDetails(
        android: AndroidNotificationDetails('daily', 'Daily'),
        iOS: DarwinNotificationDetails(),
      ),
    );
  }

  // ─── Progress notification (Android) ───
  Future<void> showProgressNotification({
    required int id,
    required String title,
    required int progress,
    required int maxProgress,
  }) async {
    await _plugin.show(
      id,
      title,
      '$progress / $maxProgress',
      NotificationDetails(
        android: AndroidNotificationDetails(
          'progress',
          'Progress',
          channelShowBadge: false,
          progress: progress,
          maxProgress: maxProgress,
          showProgress: true,
          onlyAlertOnce: true,
        ),
      ),
    );
  }

  Future<void> cancelNotification(int id) async => _plugin.cancel(id);
  Future<void> cancelAllNotifications() async => _plugin.cancelAll();
}
```

---

**← [Part 35 - Accessibility](part-35-accessibility.md)**

**ต่อไป: [Part 37 - Maps & Location →](part-37-maps-location.md)**
