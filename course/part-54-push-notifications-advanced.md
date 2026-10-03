# Part 54: Push Notifications Advanced
## ขั้นตอนที่ 2041-2080

## 🎯 เป้าหมายของ Part นี้
- Subscribe/Unsubscribe FCM topics
- สร้าง Notification Channels บน Android
- Rich notifications พร้อมรูปภาพ
- Notification Action Buttons
- In-app Notification Center
- Local Scheduled Notifications

---

## ขั้นตอนที่ 2041: FCM Setup และ Topic Subscription

```dart
// lib/notifications/fcm_service.dart
// pubspec.yaml:
//   firebase_messaging: ^15.1.3
//   flutter_local_notifications: ^17.2.2
//   firebase_core: ^3.6.0

import 'dart:io';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

// ── Background handler (ต้องเป็น top-level function) ─────────────────────────
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  print('[BG] Message id: ${message.messageId}');
  print('[BG] Data: ${message.data}');
  // Firebase.initializeApp() ต้อง call ก่อนถ้าใช้ Firebase ใน background handler
}

class FcmService {
  FcmService._();
  static final FcmService instance = FcmService._();

  final _fcm = FirebaseMessaging.instance;
  final _localNotif = FlutterLocalNotificationsPlugin();
  String? _token;

  String? get token => _token;

  // ── Initialization ─────────────────────────────────────────────────────────

  Future<void> initialize() async {
    // Register background handler
    FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);

    // Request permission (iOS / Web)
    final settings = await _fcm.requestPermission(
      alert: true,
      announcement: false,
      badge: true,
      carPlay: false,
      criticalAlert: false,
      provisional: false,
      sound: true,
    );
    print('Permission status: ${settings.authorizationStatus}');

    // Get FCM token
    _token = await _fcm.getToken();
    print('FCM Token: $_token');

    // Listen to token refresh
    _fcm.onTokenRefresh.listen((newToken) {
      _token = newToken;
      _onTokenRefresh(newToken);
    });

    // Initialize local notifications
    await _initLocalNotifications();

    // Handle foreground messages
    FirebaseMessaging.onMessage.listen(_handleForegroundMessage);

    // Handle notification tap when app is in background (but not terminated)
    FirebaseMessaging.onMessageOpenedApp.listen(_handleNotificationTap);

    // Handle notification that launched the app from terminated state
    final initialMessage = await _fcm.getInitialMessage();
    if (initialMessage != null) {
      _handleNotificationTap(initialMessage);
    }

    // สำหรับ iOS: ตั้ง foreground notification options
    await _fcm.setForegroundNotificationPresentationOptions(
      alert: true,
      badge: true,
      sound: true,
    );
  }

  // ── Local Notifications Setup ──────────────────────────────────────────────

  Future<void> _initLocalNotifications() async {
    const androidSettings = AndroidInitializationSettings('@mipmap/ic_launcher');
    const iosSettings = DarwinInitializationSettings(
      requestAlertPermission: false,
      requestBadgePermission: false,
      requestSoundPermission: false,
    );

    await _localNotif.initialize(
      const InitializationSettings(android: androidSettings, iOS: iosSettings),
      onDidReceiveNotificationResponse: (response) {
        print('Notification tapped: ${response.payload}');
        _onLocalNotificationTap(response);
      },
    );

    // Create Android notification channels
    await _createNotificationChannels();
  }

  Future<void> _createNotificationChannels() async {
    if (!Platform.isAndroid) return;

    final androidPlugin =
        _localNotif.resolvePlatformSpecificImplementation<AndroidFlutterLocalNotificationsPlugin>();
    if (androidPlugin == null) return;

    // Channel 1: General
    await androidPlugin.createNotificationChannel(
      const AndroidNotificationChannel(
        'general_channel',
        'General Notifications',
        description: 'General app notifications',
        importance: Importance.defaultImportance,
        playSound: true,
      ),
    );

    // Channel 2: High priority alerts
    await androidPlugin.createNotificationChannel(
      const AndroidNotificationChannel(
        'high_priority_channel',
        'Important Alerts',
        description: 'Critical alerts requiring immediate attention',
        importance: Importance.max,
        playSound: true,
        enableVibration: true,
        enableLights: true,
        ledColor: Color.fromARGB(255, 255, 0, 0),
      ),
    );

    // Channel 3: Promotions
    await androidPlugin.createNotificationChannel(
      const AndroidNotificationChannel(
        'promotions_channel',
        'Promotions & Offers',
        description: 'Deals and special offers',
        importance: Importance.low,
        playSound: false,
      ),
    );

    // Channel 4: Chat messages
    await androidPlugin.createNotificationChannel(
      const AndroidNotificationChannel(
        'chat_channel',
        'Chat Messages',
        description: 'New chat messages',
        importance: Importance.high,
        playSound: true,
        enableVibration: true,
      ),
    );
  }

  // ── Topic Subscription ─────────────────────────────────────────────────────

  Future<void> subscribeToTopic(String topic) async {
    await _fcm.subscribeToTopic(topic);
    print('Subscribed to topic: $topic');
  }

  Future<void> unsubscribeFromTopic(String topic) async {
    await _fcm.unsubscribeFromTopic(topic);
    print('Unsubscribed from topic: $topic');
  }

  Future<void> subscribeToMultipleTopics(List<String> topics) async {
    for (final topic in topics) {
      await subscribeToTopic(topic);
    }
  }

  // ── Show Local Notification ────────────────────────────────────────────────

  Future<void> showNotification({
    required int id,
    required String title,
    required String body,
    String? payload,
    String channelId = 'general_channel',
    String? imageUrl,
    List<AndroidNotificationAction>? actions,
  }) async {
    AndroidNotificationDetails androidDetails;

    if (imageUrl != null) {
      final bigPicture = imageUrl.startsWith('http')
          ? BigPictureStyleInformation(
              DrawableResourceAndroidBitmap('@mipmap/ic_launcher'),
              largeIcon: DrawableResourceAndroidBitmap('@mipmap/ic_launcher'),
              contentTitle: title,
              htmlFormatContentTitle: false,
              summaryText: body,
            )
          : BigPictureStyleInformation(
              FilePathAndroidBitmap(imageUrl),
              largeIcon: FilePathAndroidBitmap(imageUrl),
              contentTitle: title,
              summaryText: body,
            );

      androidDetails = AndroidNotificationDetails(
        channelId,
        channelId,
        styleInformation: bigPicture,
        actions: actions,
        importance: Importance.high,
        priority: Priority.high,
      );
    } else {
      androidDetails = AndroidNotificationDetails(
        channelId,
        channelId,
        actions: actions,
        importance: Importance.high,
        priority: Priority.high,
        styleInformation: BigTextStyleInformation(body),
      );
    }

    await _localNotif.show(
      id,
      title,
      body,
      NotificationDetails(
        android: androidDetails,
        iOS: const DarwinNotificationDetails(
          presentAlert: true,
          presentBadge: true,
          presentSound: true,
        ),
      ),
      payload: payload,
    );
  }

  // ── Notification with Action Buttons ──────────────────────────────────────

  Future<void> showNotificationWithActions({
    required int id,
    required String title,
    required String body,
    required String payload,
  }) async {
    const androidDetails = AndroidNotificationDetails(
      'chat_channel',
      'Chat Messages',
      importance: Importance.high,
      priority: Priority.high,
      actions: [
        AndroidNotificationAction(
          'reply_action',
          'Reply',
          icon: DrawableResourceAndroidBitmap('@drawable/ic_reply'),
          inputs: [
            AndroidNotificationActionInput(label: 'Type a reply...'),
          ],
          showsUserInterface: false,
          cancelNotification: false,
        ),
        AndroidNotificationAction(
          'mark_read_action',
          'Mark as Read',
          cancelNotification: true,
          showsUserInterface: false,
        ),
      ],
    );

    await _localNotif.show(
      id,
      title,
      body,
      const NotificationDetails(android: androidDetails),
      payload: payload,
    );
  }

  // ── Scheduled Notification ─────────────────────────────────────────────────

  Future<void> scheduleNotification({
    required int id,
    required String title,
    required String body,
    required DateTime scheduledTime,
    String? payload,
  }) async {
    final tzDateTime = tz.TZDateTime.from(scheduledTime, tz.local);

    await _localNotif.zonedSchedule(
      id,
      title,
      body,
      tzDateTime,
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'general_channel',
          'General Notifications',
          importance: Importance.high,
          priority: Priority.high,
        ),
        iOS: DarwinNotificationDetails(),
      ),
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      uiLocalNotificationDateInterpretation:
          UILocalNotificationDateInterpretation.absoluteTime,
      payload: payload,
    );
    print('Scheduled notification at $scheduledTime');
  }

  Future<void> cancelNotification(int id) => _localNotif.cancel(id);
  Future<void> cancelAllNotifications() => _localNotif.cancelAll();

  // ── Handlers ───────────────────────────────────────────────────────────────

  void _handleForegroundMessage(RemoteMessage message) {
    print('[FG] Message: ${message.notification?.title}');
    final notif = message.notification;
    if (notif != null) {
      showNotification(
        id: message.hashCode,
        title: notif.title ?? 'Notification',
        body: notif.body ?? '',
        payload: message.data.toString(),
      );
    }
  }

  void _handleNotificationTap(RemoteMessage message) {
    print('[TAP] Notification tapped: ${message.data}');
    // Navigate based on message.data
  }

  void _onLocalNotificationTap(NotificationResponse response) {
    print('[LOCAL TAP] payload: ${response.payload}');
    print('[LOCAL TAP] actionId: ${response.actionId}');
    print('[LOCAL TAP] input: ${response.input}');
  }

  void _onTokenRefresh(String token) {
    // Send new token to your backend
    print('[TOKEN] New token: $token');
  }
}

import 'dart:ui' show Color;
import 'package:flutter/material.dart' show Color;
import 'package:timezone/timezone.dart' as tz;
```

---

## ขั้นตอนที่ 2042: Notification Settings Screen

```dart
// lib/notifications/notification_settings_page.dart
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'fcm_service.dart';

class NotificationTopic {
  const NotificationTopic({
    required this.id,
    required this.name,
    required this.description,
    required this.icon,
  });
  final String id;
  final String name;
  final String description;
  final IconData icon;
}

class NotificationSettingsPage extends StatefulWidget {
  const NotificationSettingsPage({super.key});

  @override
  State<NotificationSettingsPage> createState() => _NotificationSettingsPageState();
}

class _NotificationSettingsPageState extends State<NotificationSettingsPage> {
  static const _topics = [
    NotificationTopic(
      id: 'news',
      name: 'News & Updates',
      description: 'Latest news and app updates',
      icon: Icons.newspaper,
    ),
    NotificationTopic(
      id: 'promotions',
      name: 'Promotions',
      description: 'Special offers and discounts',
      icon: Icons.local_offer,
    ),
    NotificationTopic(
      id: 'reminders',
      name: 'Reminders',
      description: 'Task and event reminders',
      icon: Icons.alarm,
    ),
    NotificationTopic(
      id: 'social',
      name: 'Social Activity',
      description: 'Likes, comments, follows',
      icon: Icons.people,
    ),
    NotificationTopic(
      id: 'security',
      name: 'Security Alerts',
      description: 'Login attempts and security notifications',
      icon: Icons.security,
    ),
  ];

  final Map<String, bool> _subscriptions = {};
  bool _allEnabled = true;
  bool _loading = true;

  @override
  void initState() {
    super.initState();
    _loadSettings();
  }

  Future<void> _loadSettings() async {
    final prefs = await SharedPreferences.getInstance();
    setState(() {
      _allEnabled = prefs.getBool('notif_all_enabled') ?? true;
      for (final t in _topics) {
        _subscriptions[t.id] = prefs.getBool('notif_topic_${t.id}') ?? true;
      }
      _loading = false;
    });
  }

  Future<void> _toggleAll(bool value) async {
    final prefs = await SharedPreferences.getInstance();
    setState(() => _allEnabled = value);
    await prefs.setBool('notif_all_enabled', value);
    if (value) {
      for (final t in _topics) {
        if (_subscriptions[t.id] == true) {
          await FcmService.instance.subscribeToTopic(t.id);
        }
      }
    } else {
      for (final t in _topics) {
        await FcmService.instance.unsubscribeFromTopic(t.id);
      }
    }
  }

  Future<void> _toggleTopic(String topicId, bool value) async {
    final prefs = await SharedPreferences.getInstance();
    setState(() => _subscriptions[topicId] = value);
    await prefs.setBool('notif_topic_$topicId', value);
    if (value && _allEnabled) {
      await FcmService.instance.subscribeToTopic(topicId);
    } else {
      await FcmService.instance.unsubscribeFromTopic(topicId);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Notification Settings')),
      body: _loading
          ? const Center(child: CircularProgressIndicator())
          : ListView(
              padding: const EdgeInsets.all(16),
              children: [
                _buildMasterSwitch(),
                const SizedBox(height: 16),
                const Text(
                  'TOPIC SUBSCRIPTIONS',
                  style: TextStyle(fontSize: 12, fontWeight: FontWeight.bold, color: Colors.grey),
                ),
                const SizedBox(height: 8),
                AnimatedOpacity(
                  duration: const Duration(milliseconds: 300),
                  opacity: _allEnabled ? 1.0 : 0.4,
                  child: Column(
                    children: _topics
                        .map((t) => _buildTopicTile(t))
                        .toList(),
                  ),
                ),
                const SizedBox(height: 24),
                _buildTestSection(),
              ],
            ),
    );
  }

  Widget _buildMasterSwitch() {
    return Card(
      child: SwitchListTile(
        value: _allEnabled,
        onChanged: _toggleAll,
        title: const Text('All Notifications', style: TextStyle(fontWeight: FontWeight.bold)),
        subtitle: const Text('Enable or disable all notifications'),
        secondary: Icon(
          _allEnabled ? Icons.notifications_active : Icons.notifications_off,
          color: _allEnabled ? Colors.blue : Colors.grey,
        ),
      ),
    );
  }

  Widget _buildTopicTile(NotificationTopic topic) {
    final isSubscribed = _subscriptions[topic.id] ?? true;
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: SwitchListTile(
        value: isSubscribed && _allEnabled,
        onChanged: _allEnabled ? (v) => _toggleTopic(topic.id, v) : null,
        title: Text(topic.name),
        subtitle: Text(topic.description, style: const TextStyle(fontSize: 12)),
        secondary: Icon(topic.icon),
      ),
    );
  }

  Widget _buildTestSection() {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.stretch,
      children: [
        const Text(
          'TEST NOTIFICATIONS',
          style: TextStyle(fontSize: 12, fontWeight: FontWeight.bold, color: Colors.grey),
        ),
        const SizedBox(height: 8),
        OutlinedButton.icon(
          onPressed: () => FcmService.instance.showNotification(
            id: 1,
            title: 'Test Notification',
            body: 'This is a test notification from Part 54!',
            channelId: 'general_channel',
          ),
          icon: const Icon(Icons.notifications),
          label: const Text('Show Test Notification'),
        ),
        const SizedBox(height: 8),
        OutlinedButton.icon(
          onPressed: () => FcmService.instance.showNotificationWithActions(
            id: 2,
            title: 'New Message from Alice',
            body: 'Hey, are you free this weekend?',
            payload: 'chat:alice:123',
          ),
          icon: const Icon(Icons.message),
          label: const Text('Show Notification with Actions'),
        ),
        const SizedBox(height: 8),
        OutlinedButton.icon(
          onPressed: () => FcmService.instance.scheduleNotification(
            id: 3,
            title: 'Scheduled Reminder',
            body: 'This was scheduled 10 seconds ago!',
            scheduledTime: DateTime.now().add(const Duration(seconds: 10)),
          ),
          icon: const Icon(Icons.schedule),
          label: const Text('Schedule (10s delay)'),
        ),
        const SizedBox(height: 8),
        OutlinedButton.icon(
          onPressed: FcmService.instance.cancelAllNotifications,
          icon: const Icon(Icons.clear_all),
          label: const Text('Cancel All'),
          style: OutlinedButton.styleFrom(foregroundColor: Colors.red),
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 2043: In-App Notification Center

```dart
// lib/notifications/notification_center.dart
import 'package:flutter/material.dart';

enum NotifType { info, success, warning, error, message }

class AppNotification {
  AppNotification({
    required this.id,
    required this.title,
    required this.body,
    required this.type,
    required this.createdAt,
    this.isRead = false,
    this.payload,
    this.imageUrl,
  });

  final String id;
  final String title;
  final String body;
  final NotifType type;
  final DateTime createdAt;
  bool isRead;
  final String? payload;
  final String? imageUrl;
}

class NotificationStore extends ChangeNotifier {
  final List<AppNotification> _notifications = [];

  List<AppNotification> get all => List.unmodifiable(_notifications);

  List<AppNotification> get unread =>
      _notifications.where((n) => !n.isRead).toList();

  int get unreadCount => unread.length;

  void add(AppNotification notif) {
    _notifications.insert(0, notif);
    notifyListeners();
  }

  void markAsRead(String id) {
    final idx = _notifications.indexWhere((n) => n.id == id);
    if (idx >= 0) {
      _notifications[idx].isRead = true;
      notifyListeners();
    }
  }

  void markAllAsRead() {
    for (final n in _notifications) {
      n.isRead = true;
    }
    notifyListeners();
  }

  void remove(String id) {
    _notifications.removeWhere((n) => n.id == id);
    notifyListeners();
  }

  void clear() {
    _notifications.clear();
    notifyListeners();
  }

  // ── Demo data ───────────────────────────────────────────────────────────────
  void addSampleNotifications() {
    final samples = [
      AppNotification(
        id: '1',
        title: 'Welcome!',
        body: 'Thank you for using our app. Explore all features.',
        type: NotifType.info,
        createdAt: DateTime.now().subtract(const Duration(minutes: 5)),
      ),
      AppNotification(
        id: '2',
        title: 'Payment Successful',
        body: 'Your order #1234 has been confirmed. Total: \$49.99',
        type: NotifType.success,
        createdAt: DateTime.now().subtract(const Duration(hours: 1)),
      ),
      AppNotification(
        id: '3',
        title: 'Storage Almost Full',
        body: 'You are using 90% of your storage. Upgrade now.',
        type: NotifType.warning,
        createdAt: DateTime.now().subtract(const Duration(hours: 3)),
      ),
      AppNotification(
        id: '4',
        title: 'New Message',
        body: 'Alice: Hey, are you free this weekend?',
        type: NotifType.message,
        createdAt: DateTime.now().subtract(const Duration(hours: 5)),
        isRead: true,
      ),
      AppNotification(
        id: '5',
        title: 'Login from new device',
        body: 'A login was detected from iPhone 15 in Bangkok, TH.',
        type: NotifType.error,
        createdAt: DateTime.now().subtract(const Duration(days: 1)),
      ),
    ];
    for (final s in samples) {
      _notifications.add(s);
    }
    notifyListeners();
  }
}

class NotificationCenterPage extends StatefulWidget {
  const NotificationCenterPage({super.key});

  @override
  State<NotificationCenterPage> createState() => _NotificationCenterPageState();
}

class _NotificationCenterPageState extends State<NotificationCenterPage>
    with SingleTickerProviderStateMixin {
  late final NotificationStore _store;
  late final TabController _tabCtrl;

  @override
  void initState() {
    super.initState();
    _store = NotificationStore()..addSampleNotifications();
    _tabCtrl = TabController(length: 2, vsync: this);
  }

  @override
  void dispose() {
    _store.dispose();
    _tabCtrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Notifications'),
        bottom: TabBar(
          controller: _tabCtrl,
          tabs: [
            Tab(
              child: AnimatedBuilder(
                animation: _store,
                builder: (ctx, _) => Row(
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    const Text('All'),
                    if (_store.unreadCount > 0) ...[
                      const SizedBox(width: 6),
                      CircleAvatar(
                        radius: 9,
                        backgroundColor: Colors.red,
                        child: Text(
                          '${_store.unreadCount}',
                          style: const TextStyle(fontSize: 10, color: Colors.white),
                        ),
                      ),
                    ],
                  ],
                ),
              ),
            ),
            const Tab(text: 'Unread'),
          ],
        ),
        actions: [
          AnimatedBuilder(
            animation: _store,
            builder: (ctx, _) => _store.unreadCount > 0
                ? TextButton(
                    onPressed: _store.markAllAsRead,
                    child: const Text('Mark all read'),
                  )
                : const SizedBox.shrink(),
          ),
          PopupMenuButton<String>(
            onSelected: (v) {
              if (v == 'clear') _store.clear();
            },
            itemBuilder: (_) => [
              const PopupMenuItem(value: 'clear', child: Text('Clear all')),
            ],
          ),
        ],
      ),
      body: AnimatedBuilder(
        animation: _store,
        builder: (ctx, _) => TabBarView(
          controller: _tabCtrl,
          children: [
            _NotificationList(notifications: _store.all, store: _store),
            _NotificationList(notifications: _store.unread, store: _store),
          ],
        ),
      ),
    );
  }
}

class _NotificationList extends StatelessWidget {
  const _NotificationList({required this.notifications, required this.store});
  final List<AppNotification> notifications;
  final NotificationStore store;

  @override
  Widget build(BuildContext context) {
    if (notifications.isEmpty) {
      return const Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(Icons.notifications_none, size: 64, color: Colors.grey),
            SizedBox(height: 16),
            Text('No notifications', style: TextStyle(color: Colors.grey)),
          ],
        ),
      );
    }
    return ListView.builder(
      padding: const EdgeInsets.symmetric(vertical: 8),
      itemCount: notifications.length,
      itemBuilder: (ctx, i) => _NotifTile(
        notif: notifications[i],
        onTap: () => store.markAsRead(notifications[i].id),
        onDismiss: () => store.remove(notifications[i].id),
      ),
    );
  }
}

class _NotifTile extends StatelessWidget {
  const _NotifTile({required this.notif, required this.onTap, required this.onDismiss});
  final AppNotification notif;
  final VoidCallback onTap;
  final VoidCallback onDismiss;

  static const _typeConfig = {
    NotifType.info: (Icons.info_outline, Colors.blue),
    NotifType.success: (Icons.check_circle_outline, Colors.green),
    NotifType.warning: (Icons.warning_amber_outlined, Colors.orange),
    NotifType.error: (Icons.error_outline, Colors.red),
    NotifType.message: (Icons.message_outlined, Colors.purple),
  };

  @override
  Widget build(BuildContext context) {
    final config = _typeConfig[notif.type]!;
    final (icon, color) = config;

    return Dismissible(
      key: Key(notif.id),
      direction: DismissDirection.endToStart,
      onDismissed: (_) => onDismiss(),
      background: Container(
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 16),
        color: Colors.red,
        child: const Icon(Icons.delete, color: Colors.white),
      ),
      child: InkWell(
        onTap: onTap,
        child: Container(
          color: notif.isRead ? null : color.withOpacity(0.05),
          child: ListTile(
            leading: Container(
              width: 44,
              height: 44,
              decoration: BoxDecoration(
                color: color.withOpacity(0.15),
                shape: BoxShape.circle,
              ),
              child: Icon(icon, color: color, size: 22),
            ),
            title: Text(
              notif.title,
              style: TextStyle(
                fontWeight: notif.isRead ? FontWeight.normal : FontWeight.bold,
              ),
            ),
            subtitle: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(notif.body, maxLines: 2, overflow: TextOverflow.ellipsis),
                const SizedBox(height: 2),
                Text(
                  _formatRelativeTime(notif.createdAt),
                  style: TextStyle(fontSize: 11, color: Colors.grey.shade500),
                ),
              ],
            ),
            trailing: notif.isRead
                ? null
                : Container(
                    width: 8,
                    height: 8,
                    decoration: BoxDecoration(color: color, shape: BoxShape.circle),
                  ),
            isThreeLine: true,
          ),
        ),
      ),
    );
  }

  String _formatRelativeTime(DateTime dt) {
    final diff = DateTime.now().difference(dt);
    if (diff.inMinutes < 1) return 'Just now';
    if (diff.inMinutes < 60) return '${diff.inMinutes}m ago';
    if (diff.inHours < 24) return '${diff.inHours}h ago';
    if (diff.inDays < 7) return '${diff.inDays}d ago';
    return '${dt.day}/${dt.month}/${dt.year}';
  }
}
```

---

## ขั้นตอนที่ 2044: Local Scheduled Notifications Manager

```dart
// lib/notifications/scheduled_notifications_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:timezone/data/latest_all.dart' as tz;
import 'package:timezone/timezone.dart' as tz_lib;
import 'fcm_service.dart';

class ScheduledNotificationsPage extends StatefulWidget {
  const ScheduledNotificationsPage({super.key});

  @override
  State<ScheduledNotificationsPage> createState() => _ScheduledNotificationsPageState();
}

class _ScheduledNotificationsPageState extends State<ScheduledNotificationsPage> {
  final _localNotif = FlutterLocalNotificationsPlugin();
  List<PendingNotificationRequest> _pending = [];

  @override
  void initState() {
    super.initState();
    tz.initializeTimeZones();
    _loadPending();
  }

  Future<void> _loadPending() async {
    final pending = await _localNotif.pendingNotificationRequests();
    setState(() => _pending = pending);
  }

  Future<void> _scheduleRepeat() async {
    await _localNotif.periodicallyShow(
      10,
      'Daily Reminder',
      "Don't forget to check your tasks!",
      RepeatInterval.daily,
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'general_channel',
          'General Notifications',
          importance: Importance.high,
        ),
      ),
    );
    await _loadPending();
  }

  Future<void> _scheduleMorningReminder() async {
    final now = tz_lib.TZDateTime.now(tz_lib.local);
    var scheduledDate = tz_lib.TZDateTime(
      tz_lib.local, now.year, now.month, now.day, 9, 0, 0,
    );
    if (scheduledDate.isBefore(now)) {
      scheduledDate = scheduledDate.add(const Duration(days: 1));
    }

    await _localNotif.zonedSchedule(
      20,
      'Good Morning!',
      'Start your day with Flutter development.',
      scheduledDate,
      const NotificationDetails(
        android: AndroidNotificationDetails(
          'general_channel',
          'General Notifications',
          importance: Importance.high,
        ),
      ),
      androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
      uiLocalNotificationDateInterpretation:
          UILocalNotificationDateInterpretation.absoluteTime,
      matchDateTimeComponents: DateTimeComponents.time, // repeat daily at 9am
    );
    await _loadPending();
    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Scheduled for tomorrow 9:00 AM (${scheduledDate.toString().substring(0, 16)})')),
      );
    }
  }

  Future<void> _cancelPending(int id) async {
    await _localNotif.cancel(id);
    await _loadPending();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Scheduled Notifications')),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                FilledButton.icon(
                  onPressed: () => FcmService.instance.scheduleNotification(
                    id: 100,
                    title: 'In 10 seconds',
                    body: 'This was scheduled 10 seconds ago!',
                    scheduledTime: DateTime.now().add(const Duration(seconds: 10)),
                  ).then((_) => _loadPending()),
                  icon: const Icon(Icons.timer),
                  label: const Text('Schedule in 10 seconds'),
                ),
                const SizedBox(height: 8),
                FilledButton.tonal(
                  onPressed: _scheduleMorningReminder,
                  child: const Text('Schedule Daily 9 AM Reminder'),
                ),
                const SizedBox(height: 8),
                OutlinedButton(
                  onPressed: _localNotif.cancelAll().then((_) => _loadPending()) as void Function()?,
                  child: const Text('Cancel All Pending'),
                ),
              ],
            ),
          ),
          const Divider(),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text('Pending: ${_pending.length}', style: const TextStyle(fontWeight: FontWeight.bold)),
                IconButton(icon: const Icon(Icons.refresh), onPressed: _loadPending),
              ],
            ),
          ),
          Expanded(
            child: _pending.isEmpty
                ? const Center(child: Text('No pending notifications'))
                : ListView.builder(
                    itemCount: _pending.length,
                    itemBuilder: (ctx, i) {
                      final n = _pending[i];
                      return ListTile(
                        leading: const Icon(Icons.notifications_outlined),
                        title: Text(n.title ?? 'No title'),
                        subtitle: Text(n.body ?? ''),
                        trailing: IconButton(
                          icon: const Icon(Icons.cancel_outlined, color: Colors.red),
                          onPressed: () => _cancelPending(n.id),
                        ),
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2045: Full Notifications App

```dart
// lib/main_notifications_demo.dart
import 'package:flutter/material.dart';
import 'notifications/fcm_service.dart';
import 'notifications/notification_settings_page.dart';
import 'notifications/notification_center.dart';
import 'notifications/scheduled_notifications_page.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  // Firebase.initializeApp() — ต้องทำก่อนใน production app
  // await Firebase.initializeApp();
  // await FcmService.instance.initialize();
  runApp(const NotificationsDemoApp());
}

class NotificationsDemoApp extends StatelessWidget {
  const NotificationsDemoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Notifications Demo',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(colorSchemeSeed: Colors.deepPurple, useMaterial3: true),
      home: const NotificationsDemoHome(),
    );
  }
}

class NotificationsDemoHome extends StatefulWidget {
  const NotificationsDemoHome({super.key});

  @override
  State<NotificationsDemoHome> createState() => _NotificationsDemoHomeState();
}

class _NotificationsDemoHomeState extends State<NotificationsDemoHome> {
  final _store = NotificationStore()..addSampleNotifications();
  int _selectedIndex = 0;

  late final List<Widget> _pages;

  @override
  void initState() {
    super.initState();
    _pages = [
      const _HomeOverview(),
      NotificationCenterPage(),
      const NotificationSettingsPage(),
      const ScheduledNotificationsPage(),
    ];
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: _pages[_selectedIndex],
      bottomNavigationBar: NavigationBar(
        selectedIndex: _selectedIndex,
        onDestinationSelected: (i) => setState(() => _selectedIndex = i),
        destinations: [
          const NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
          NavigationDestination(
            icon: AnimatedBuilder(
              animation: _store,
              builder: (ctx, _) => Badge(
                isLabelVisible: _store.unreadCount > 0,
                label: Text('${_store.unreadCount}'),
                child: const Icon(Icons.notifications),
              ),
            ),
            label: 'Center',
          ),
          const NavigationDestination(icon: Icon(Icons.tune), label: 'Settings'),
          const NavigationDestination(icon: Icon(Icons.schedule), label: 'Scheduled'),
        ],
      ),
    );
  }
}

class _HomeOverview extends StatelessWidget {
  const _HomeOverview();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Part 54 — Push Notifications')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: const [
          _FeatureCard(
            icon: Icons.topic,
            title: 'FCM Topics',
            description: 'Subscribe/unsubscribe to notification topics',
          ),
          SizedBox(height: 12),
          _FeatureCard(
            icon: Icons.layers,
            title: 'Notification Channels',
            description: 'Android notification channels with different priorities',
          ),
          SizedBox(height: 12),
          _FeatureCard(
            icon: Icons.image,
            title: 'Rich Notifications',
            description: 'Notifications with images and expanded text',
          ),
          SizedBox(height: 12),
          _FeatureCard(
            icon: Icons.touch_app,
            title: 'Action Buttons',
            description: 'Reply and action buttons on notifications',
          ),
          SizedBox(height: 12),
          _FeatureCard(
            icon: Icons.inbox,
            title: 'Notification Center',
            description: 'In-app notification inbox with read/unread state',
          ),
          SizedBox(height: 12),
          _FeatureCard(
            icon: Icons.alarm,
            title: 'Scheduled Notifications',
            description: 'Local notifications scheduled for future times',
          ),
        ],
      ),
    );
  }
}

class _FeatureCard extends StatelessWidget {
  const _FeatureCard({required this.icon, required this.title, required this.description});
  final IconData icon;
  final String title;
  final String description;

  @override
  Widget build(BuildContext context) {
    return Card(
      child: ListTile(
        leading: CircleAvatar(
          backgroundColor: Colors.deepPurple.withOpacity(0.1),
          child: Icon(icon, color: Colors.deepPurple),
        ),
        title: Text(title, style: const TextStyle(fontWeight: FontWeight.w600)),
        subtitle: Text(description),
      ),
    );
  }
}
```

---

**← [Part 53](part-53-firebase-firestore-advanced.md)**
**ต่อไป: [Part 55 →](part-55-internationalization.md)**
