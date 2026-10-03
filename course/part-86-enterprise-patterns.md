# Part 86: Enterprise Architecture Patterns
## ขั้นตอนที่ 3321-3360

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Multi-tenant architecture ใน Flutter
- สร้างระบบ Feature Flags ด้วย Remote Config
- ออกแบบและพัฒนา A/B Testing framework
- สร้าง Analytics event tracking system แบบครบวงจร
- ตั้งค่า Crash reporting พร้อม metadata ครบถ้วน
- นำ Enterprise patterns ไปใช้งานจริงได้

---

## ขั้นตอนที่ 3321: Multi-Tenant Architecture

Multi-tenant architecture ช่วยให้แอปเดียวกันรองรับลูกค้าหลายรายพร้อมกัน โดยแต่ละ tenant มี config และ data แยกจากกัน

```dart
// lib/core/tenant/tenant_config.dart

class TenantConfig {
  final String tenantId;
  final String tenantName;
  final String apiBaseUrl;
  final ThemeConfig theme;
  final FeatureSet features;
  final Map<String, dynamic> customSettings;

  const TenantConfig({
    required this.tenantId,
    required this.tenantName,
    required this.apiBaseUrl,
    required this.theme,
    required this.features,
    this.customSettings = const {},
  });

  factory TenantConfig.fromJson(Map<String, dynamic> json) {
    return TenantConfig(
      tenantId: json['tenant_id'] as String,
      tenantName: json['tenant_name'] as String,
      apiBaseUrl: json['api_base_url'] as String,
      theme: ThemeConfig.fromJson(json['theme'] as Map<String, dynamic>),
      features: FeatureSet.fromJson(json['features'] as Map<String, dynamic>),
      customSettings: json['custom_settings'] as Map<String, dynamic>? ?? {},
    );
  }

  Map<String, dynamic> toJson() => {
        'tenant_id': tenantId,
        'tenant_name': tenantName,
        'api_base_url': apiBaseUrl,
        'theme': theme.toJson(),
        'features': features.toJson(),
        'custom_settings': customSettings,
      };

  TenantConfig copyWith({
    String? tenantId,
    String? tenantName,
    String? apiBaseUrl,
    ThemeConfig? theme,
    FeatureSet? features,
    Map<String, dynamic>? customSettings,
  }) {
    return TenantConfig(
      tenantId: tenantId ?? this.tenantId,
      tenantName: tenantName ?? this.tenantName,
      apiBaseUrl: apiBaseUrl ?? this.apiBaseUrl,
      theme: theme ?? this.theme,
      features: features ?? this.features,
      customSettings: customSettings ?? this.customSettings,
    );
  }
}

class ThemeConfig {
  final String primaryColor;
  final String secondaryColor;
  final String logoUrl;
  final String fontFamily;

  const ThemeConfig({
    required this.primaryColor,
    required this.secondaryColor,
    required this.logoUrl,
    this.fontFamily = 'Roboto',
  });

  factory ThemeConfig.fromJson(Map<String, dynamic> json) {
    return ThemeConfig(
      primaryColor: json['primary_color'] as String? ?? '#2196F3',
      secondaryColor: json['secondary_color'] as String? ?? '#FFC107',
      logoUrl: json['logo_url'] as String? ?? '',
      fontFamily: json['font_family'] as String? ?? 'Roboto',
    );
  }

  Map<String, dynamic> toJson() => {
        'primary_color': primaryColor,
        'secondary_color': secondaryColor,
        'logo_url': logoUrl,
        'font_family': fontFamily,
      };
}

class FeatureSet {
  final bool enableChat;
  final bool enablePayments;
  final bool enableAnalytics;
  final bool enablePushNotifications;
  final bool enableOfflineMode;
  final Map<String, bool> experimentalFeatures;

  const FeatureSet({
    this.enableChat = false,
    this.enablePayments = false,
    this.enableAnalytics = true,
    this.enablePushNotifications = true,
    this.enableOfflineMode = false,
    this.experimentalFeatures = const {},
  });

  factory FeatureSet.fromJson(Map<String, dynamic> json) {
    return FeatureSet(
      enableChat: json['enable_chat'] as bool? ?? false,
      enablePayments: json['enable_payments'] as bool? ?? false,
      enableAnalytics: json['enable_analytics'] as bool? ?? true,
      enablePushNotifications:
          json['enable_push_notifications'] as bool? ?? true,
      enableOfflineMode: json['enable_offline_mode'] as bool? ?? false,
      experimentalFeatures: Map<String, bool>.from(
          json['experimental_features'] as Map? ?? {}),
    );
  }

  Map<String, dynamic> toJson() => {
        'enable_chat': enableChat,
        'enable_payments': enablePayments,
        'enable_analytics': enableAnalytics,
        'enable_push_notifications': enablePushNotifications,
        'enable_offline_mode': enableOfflineMode,
        'experimental_features': experimentalFeatures,
      };
}
```

## ขั้นตอนที่ 3322: TenantManager และ TenantRepository

```dart
// lib/core/tenant/tenant_manager.dart
import 'dart:convert';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:http/http.dart' as http;

class TenantManager {
  static const _tenantCacheKey = 'cached_tenant_config';
  static const _tenantIdKey = 'current_tenant_id';

  final SharedPreferences _prefs;
  final http.Client _httpClient;

  TenantConfig? _currentTenant;

  TenantManager({
    required SharedPreferences prefs,
    required http.Client httpClient,
  })  : _prefs = prefs,
        _httpClient = httpClient;

  TenantConfig? get currentTenant => _currentTenant;

  String? get currentTenantId => _prefs.getString(_tenantIdKey);

  Future<TenantConfig> loadTenant(String tenantId) async {
    // Try cache first
    final cached = _loadFromCache(tenantId);
    if (cached != null) {
      _currentTenant = cached;
      return cached;
    }

    // Fetch from remote
    final config = await _fetchTenantConfig(tenantId);
    await _saveToCache(tenantId, config);
    await _prefs.setString(_tenantIdKey, tenantId);
    _currentTenant = config;
    return config;
  }

  Future<void> clearTenant() async {
    _currentTenant = null;
    await _prefs.remove(_tenantIdKey);
    await _prefs.remove(_tenantCacheKey);
  }

  Future<void> refreshTenant() async {
    final tenantId = currentTenantId;
    if (tenantId == null) return;
    await _prefs.remove(_tenantCacheKey);
    await loadTenant(tenantId);
  }

  TenantConfig? _loadFromCache(String tenantId) {
    final cached = _prefs.getString(_tenantCacheKey);
    if (cached == null) return null;

    try {
      final json = jsonDecode(cached) as Map<String, dynamic>;
      final config = TenantConfig.fromJson(json);
      if (config.tenantId != tenantId) return null;
      return config;
    } catch (_) {
      return null;
    }
  }

  Future<void> _saveToCache(String tenantId, TenantConfig config) async {
    await _prefs.setString(_tenantCacheKey, jsonEncode(config.toJson()));
  }

  Future<TenantConfig> _fetchTenantConfig(String tenantId) async {
    final response = await _httpClient.get(
      Uri.parse('https://config.example.com/tenants/$tenantId'),
      headers: {'Accept': 'application/json'},
    );

    if (response.statusCode == 200) {
      final json = jsonDecode(response.body) as Map<String, dynamic>;
      return TenantConfig.fromJson(json);
    }

    // Default fallback config
    return TenantConfig(
      tenantId: tenantId,
      tenantName: 'Default',
      apiBaseUrl: 'https://api.example.com',
      theme: const ThemeConfig(
        primaryColor: '#2196F3',
        secondaryColor: '#FFC107',
        logoUrl: '',
      ),
      features: const FeatureSet(),
    );
  }
}

// Riverpod provider
final tenantManagerProvider = Provider<TenantManager>((ref) {
  throw UnimplementedError('Override in ProviderScope');
});

final currentTenantProvider = StateProvider<TenantConfig?>((ref) => null);
```

## ขั้นตอนที่ 3323: Multi-Tenant Flutter App Setup

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:http/http.dart' as http;

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  final prefs = await SharedPreferences.getInstance();
  final httpClient = http.Client();
  final tenantManager = TenantManager(
    prefs: prefs,
    httpClient: httpClient,
  );

  // Determine tenant from deep link or stored value
  const tenantId = 'tenant_001';
  final tenantConfig = await tenantManager.loadTenant(tenantId);

  runApp(
    ProviderScope(
      overrides: [
        tenantManagerProvider.overrideWithValue(tenantManager),
        currentTenantProvider.overrideWith((ref) => tenantConfig),
      ],
      child: const MyApp(),
    ),
  );
}

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final tenant = ref.watch(currentTenantProvider);

    return MaterialApp(
      title: tenant?.tenantName ?? 'App',
      theme: _buildTheme(tenant?.theme),
      home: const HomeScreen(),
    );
  }

  ThemeData _buildTheme(ThemeConfig? config) {
    if (config == null) return ThemeData.light();

    final primaryColor = _parseColor(config.primaryColor);
    final secondaryColor = _parseColor(config.secondaryColor);

    return ThemeData(
      colorScheme: ColorScheme.fromSeed(
        seedColor: primaryColor,
        secondary: secondaryColor,
      ),
      useMaterial3: true,
    );
  }

  Color _parseColor(String hex) {
    final cleaned = hex.replaceAll('#', '');
    return Color(int.parse('FF$cleaned', radix: 16));
  }
}

class HomeScreen extends ConsumerWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final tenant = ref.watch(currentTenantProvider);

    return Scaffold(
      appBar: AppBar(
        title: Text(tenant?.tenantName ?? 'Enterprise App'),
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          _TenantInfoCard(tenant: tenant),
          const SizedBox(height: 16),
          if (tenant?.features.enableChat ?? false) const _ChatFeatureCard(),
          if (tenant?.features.enablePayments ?? false)
            const _PaymentsFeatureCard(),
        ],
      ),
    );
  }
}

class _TenantInfoCard extends StatelessWidget {
  final TenantConfig? tenant;
  const _TenantInfoCard({required this.tenant});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Tenant: ${tenant?.tenantName ?? "Unknown"}',
              style: Theme.of(context).textTheme.titleLarge,
            ),
            Text('ID: ${tenant?.tenantId ?? "N/A"}'),
            Text('API: ${tenant?.apiBaseUrl ?? "N/A"}'),
          ],
        ),
      ),
    );
  }
}

class _ChatFeatureCard extends StatelessWidget {
  const _ChatFeatureCard();

  @override
  Widget build(BuildContext context) {
    return Card(
      color: Colors.blue.shade50,
      child: const ListTile(
        leading: Icon(Icons.chat, color: Colors.blue),
        title: Text('Chat Enabled'),
        subtitle: Text('Real-time messaging is available'),
      ),
    );
  }
}

class _PaymentsFeatureCard extends StatelessWidget {
  const _PaymentsFeatureCard();

  @override
  Widget build(BuildContext context) {
    return Card(
      color: Colors.green.shade50,
      child: const ListTile(
        leading: Icon(Icons.payment, color: Colors.green),
        title: Text('Payments Enabled'),
        subtitle: Text('Payment processing is available'),
      ),
    );
  }
}
```

## ขั้นตอนที่ 3324: Feature Flags ด้วย Remote Config

```dart
// lib/core/feature_flags/feature_flag_service.dart
import 'dart:async';
import 'dart:convert';
import 'package:flutter/foundation.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:http/http.dart' as http;

enum FlagType { boolean, string, number, json }

class FeatureFlag {
  final String key;
  final FlagType type;
  final dynamic defaultValue;
  final dynamic currentValue;
  final DateTime? expiresAt;

  const FeatureFlag({
    required this.key,
    required this.type,
    required this.defaultValue,
    this.currentValue,
    this.expiresAt,
  });

  bool get isExpired =>
      expiresAt != null && DateTime.now().isAfter(expiresAt!);

  dynamic get effectiveValue {
    if (isExpired) return defaultValue;
    return currentValue ?? defaultValue;
  }

  bool get asBool => effectiveValue as bool? ?? false;
  String get asString => effectiveValue?.toString() ?? '';
  num get asNumber => effectiveValue as num? ?? 0;
  Map<String, dynamic> get asJson =>
      effectiveValue as Map<String, dynamic>? ?? {};

  factory FeatureFlag.fromJson(Map<String, dynamic> json) {
    final typeStr = json['type'] as String? ?? 'boolean';
    final type = FlagType.values.firstWhere(
      (t) => t.name == typeStr,
      orElse: () => FlagType.boolean,
    );

    return FeatureFlag(
      key: json['key'] as String,
      type: type,
      defaultValue: json['default_value'],
      currentValue: json['current_value'],
      expiresAt: json['expires_at'] != null
          ? DateTime.parse(json['expires_at'] as String)
          : null,
    );
  }
}

class FeatureFlagService extends ChangeNotifier {
  final SharedPreferences _prefs;
  final http.Client _httpClient;
  final String _configUrl;

  static const _cacheKey = 'feature_flags_cache';
  static const _fetchIntervalMinutes = 30;

  Map<String, FeatureFlag> _flags = {};
  DateTime? _lastFetchTime;
  Timer? _refreshTimer;

  FeatureFlagService({
    required SharedPreferences prefs,
    required http.Client httpClient,
    required String configUrl,
  })  : _prefs = prefs,
        _httpClient = httpClient,
        _configUrl = configUrl;

  bool isEnabled(String key, {bool defaultValue = false}) {
    final flag = _flags[key];
    if (flag == null) return defaultValue;
    return flag.asBool;
  }

  String getString(String key, {String defaultValue = ''}) {
    return _flags[key]?.asString ?? defaultValue;
  }

  num getNumber(String key, {num defaultValue = 0}) {
    return _flags[key]?.asNumber ?? defaultValue;
  }

  Map<String, dynamic> getJson(String key) {
    return _flags[key]?.asJson ?? {};
  }

  Future<void> initialize() async {
    _loadFromCache();
    await fetchFlags();
    _startRefreshTimer();
  }

  Future<void> fetchFlags() async {
    try {
      final response = await _httpClient.get(
        Uri.parse(_configUrl),
        headers: {'Accept': 'application/json'},
      );

      if (response.statusCode == 200) {
        final data = jsonDecode(response.body) as Map<String, dynamic>;
        final flagsList = data['flags'] as List? ?? [];

        _flags = {
          for (final flagJson in flagsList)
            (flagJson as Map<String, dynamic>)['key'] as String:
                FeatureFlag.fromJson(flagJson)
        };

        _lastFetchTime = DateTime.now();
        await _saveToCache();
        notifyListeners();
      }
    } catch (e) {
      debugPrint('FeatureFlagService: Failed to fetch flags: $e');
    }
  }

  void _loadFromCache() {
    final cached = _prefs.getString(_cacheKey);
    if (cached == null) return;

    try {
      final data = jsonDecode(cached) as Map<String, dynamic>;
      final flagsData = data['flags'] as Map<String, dynamic>? ?? {};
      _flags = flagsData.map(
        (k, v) => MapEntry(k, FeatureFlag.fromJson(v as Map<String, dynamic>)),
      );
    } catch (e) {
      debugPrint('FeatureFlagService: Cache load failed: $e');
    }
  }

  Future<void> _saveToCache() async {
    final data = {
      'flags': _flags.map((k, v) => MapEntry(k, {
            'key': v.key,
            'type': v.type.name,
            'default_value': v.defaultValue,
            'current_value': v.currentValue,
            'expires_at': v.expiresAt?.toIso8601String(),
          })),
      'saved_at': DateTime.now().toIso8601String(),
    };
    await _prefs.setString(_cacheKey, jsonEncode(data));
  }

  void _startRefreshTimer() {
    _refreshTimer?.cancel();
    _refreshTimer = Timer.periodic(
      Duration(minutes: _fetchIntervalMinutes),
      (_) => fetchFlags(),
    );
  }

  @override
  void dispose() {
    _refreshTimer?.cancel();
    super.dispose();
  }
}
```

## ขั้นตอนที่ 3325: A/B Testing Implementation

```dart
// lib/core/ab_testing/ab_test_service.dart
import 'dart:convert';
import 'dart:math';
import 'package:crypto/crypto.dart';
import 'package:shared_preferences/shared_preferences.dart';

class Experiment {
  final String id;
  final String name;
  final List<Variant> variants;
  final bool isActive;
  final double trafficAllocation; // 0.0 - 1.0

  const Experiment({
    required this.id,
    required this.name,
    required this.variants,
    this.isActive = true,
    this.trafficAllocation = 1.0,
  });

  factory Experiment.fromJson(Map<String, dynamic> json) {
    return Experiment(
      id: json['id'] as String,
      name: json['name'] as String,
      variants: (json['variants'] as List)
          .map((v) => Variant.fromJson(v as Map<String, dynamic>))
          .toList(),
      isActive: json['is_active'] as bool? ?? true,
      trafficAllocation: (json['traffic_allocation'] as num?)?.toDouble() ?? 1.0,
    );
  }
}

class Variant {
  final String id;
  final String name;
  final double weight; // Relative weight for traffic split
  final Map<String, dynamic> config;

  const Variant({
    required this.id,
    required this.name,
    this.weight = 1.0,
    this.config = const {},
  });

  factory Variant.fromJson(Map<String, dynamic> json) {
    return Variant(
      id: json['id'] as String,
      name: json['name'] as String,
      weight: (json['weight'] as num?)?.toDouble() ?? 1.0,
      config: json['config'] as Map<String, dynamic>? ?? {},
    );
  }
}

class ABTestAssignment {
  final String experimentId;
  final String variantId;
  final DateTime assignedAt;

  const ABTestAssignment({
    required this.experimentId,
    required this.variantId,
    required this.assignedAt,
  });

  factory ABTestAssignment.fromJson(Map<String, dynamic> json) {
    return ABTestAssignment(
      experimentId: json['experiment_id'] as String,
      variantId: json['variant_id'] as String,
      assignedAt: DateTime.parse(json['assigned_at'] as String),
    );
  }

  Map<String, dynamic> toJson() => {
        'experiment_id': experimentId,
        'variant_id': variantId,
        'assigned_at': assignedAt.toIso8601String(),
      };
}

class ABTestService {
  final SharedPreferences _prefs;
  final String _userId;

  static const _assignmentsKey = 'ab_test_assignments';

  Map<String, ABTestAssignment> _assignments = {};
  final List<Experiment> _experiments = [];

  ABTestService({
    required SharedPreferences prefs,
    required String userId,
  })  : _prefs = prefs,
        _userId = userId {
    _loadAssignments();
  }

  void registerExperiment(Experiment experiment) {
    _experiments.removeWhere((e) => e.id == experiment.id);
    _experiments.add(experiment);
  }

  Variant? getVariant(String experimentId) {
    final experiment = _experiments.cast<Experiment?>().firstWhere(
          (e) => e?.id == experimentId,
          orElse: () => null,
        );

    if (experiment == null || !experiment.isActive) return null;

    // Check if user is in traffic allocation
    if (!_isUserInTraffic(experimentId, experiment.trafficAllocation)) {
      return null;
    }

    // Return cached assignment
    final existing = _assignments[experimentId];
    if (existing != null) {
      return experiment.variants
          .cast<Variant?>()
          .firstWhere((v) => v?.id == existing.variantId, orElse: () => null);
    }

    // Assign variant deterministically based on user ID
    final variant = _assignVariant(experimentId, experiment.variants);
    _saveAssignment(experimentId, variant.id);
    return variant;
  }

  bool isInVariant(String experimentId, String variantId) {
    return getVariant(experimentId)?.id == variantId;
  }

  Map<String, dynamic>? getVariantConfig(String experimentId) {
    return getVariant(experimentId)?.config;
  }

  bool _isUserInTraffic(String experimentId, double allocation) {
    final hash = _computeHash('$_userId:$experimentId:traffic');
    return hash < allocation;
  }

  Variant _assignVariant(String experimentId, List<Variant> variants) {
    final totalWeight = variants.fold<double>(0, (sum, v) => sum + v.weight);
    final hash = _computeHash('$_userId:$experimentId:assignment');
    final targetWeight = hash * totalWeight;

    double cumulative = 0;
    for (final variant in variants) {
      cumulative += variant.weight;
      if (targetWeight <= cumulative) return variant;
    }

    return variants.last;
  }

  double _computeHash(String input) {
    final bytes = utf8.encode(input);
    final digest = sha256.convert(bytes);
    // Use first 4 bytes as a uint32 and normalize to 0..1
    final value = (digest.bytes[0] << 24) |
        (digest.bytes[1] << 16) |
        (digest.bytes[2] << 8) |
        digest.bytes[3];
    return value / 0xFFFFFFFF;
  }

  void _saveAssignment(String experimentId, String variantId) {
    _assignments[experimentId] = ABTestAssignment(
      experimentId: experimentId,
      variantId: variantId,
      assignedAt: DateTime.now(),
    );
    _persistAssignments();
  }

  void _loadAssignments() {
    final stored = _prefs.getString(_assignmentsKey);
    if (stored == null) return;

    try {
      final data = jsonDecode(stored) as Map<String, dynamic>;
      _assignments = data.map(
        (k, v) => MapEntry(
            k, ABTestAssignment.fromJson(v as Map<String, dynamic>)),
      );
    } catch (_) {}
  }

  void _persistAssignments() {
    _prefs.setString(
      _assignmentsKey,
      jsonEncode(_assignments.map((k, v) => MapEntry(k, v.toJson()))),
    );
  }

  List<Map<String, String>> getActiveAssignments() {
    return _assignments.entries
        .map((e) => {
              'experiment_id': e.key,
              'variant_id': e.value.variantId,
            })
        .toList();
  }
}
```

## ขั้นตอนที่ 3326: Analytics Event Tracking System

```dart
// lib/core/analytics/analytics_event.dart

enum EventCategory {
  userAction,
  screenView,
  conversion,
  error,
  performance,
  custom,
}

class AnalyticsEvent {
  final String name;
  final EventCategory category;
  final Map<String, dynamic> properties;
  final DateTime timestamp;
  final String? userId;
  final String? sessionId;
  final Map<String, String> abTestAssignments;

  AnalyticsEvent({
    required this.name,
    required this.category,
    this.properties = const {},
    DateTime? timestamp,
    this.userId,
    this.sessionId,
    this.abTestAssignments = const {},
  }) : timestamp = timestamp ?? DateTime.now();

  Map<String, dynamic> toJson() => {
        'name': name,
        'category': category.name,
        'properties': properties,
        'timestamp': timestamp.toIso8601String(),
        if (userId != null) 'user_id': userId,
        if (sessionId != null) 'session_id': sessionId,
        'ab_tests': abTestAssignments,
      };

  // Named constructors for common events
  factory AnalyticsEvent.screenView(
    String screenName, {
    Map<String, dynamic> properties = const {},
  }) {
    return AnalyticsEvent(
      name: 'screen_view',
      category: EventCategory.screenView,
      properties: {'screen_name': screenName, ...properties},
    );
  }

  factory AnalyticsEvent.buttonTap(
    String buttonName, {
    String? screen,
    Map<String, dynamic> extra = const {},
  }) {
    return AnalyticsEvent(
      name: 'button_tap',
      category: EventCategory.userAction,
      properties: {
        'button_name': buttonName,
        if (screen != null) 'screen': screen,
        ...extra,
      },
    );
  }

  factory AnalyticsEvent.purchase(
    String productId,
    double amount,
    String currency,
  ) {
    return AnalyticsEvent(
      name: 'purchase',
      category: EventCategory.conversion,
      properties: {
        'product_id': productId,
        'amount': amount,
        'currency': currency,
      },
    );
  }
}

// lib/core/analytics/analytics_service.dart
import 'dart:async';
import 'dart:convert';
import 'package:flutter/foundation.dart';
import 'package:http/http.dart' as http;

abstract class AnalyticsDestination {
  String get name;
  Future<void> send(List<AnalyticsEvent> events);
}

class ConsoleDestination implements AnalyticsDestination {
  @override
  String get name => 'console';

  @override
  Future<void> send(List<AnalyticsEvent> events) async {
    for (final event in events) {
      debugPrint('[Analytics] ${event.name}: ${jsonEncode(event.toJson())}');
    }
  }
}

class HttpDestination implements AnalyticsDestination {
  final String endpoint;
  final http.Client _client;
  final Map<String, String> headers;

  HttpDestination({
    required this.endpoint,
    required http.Client client,
    this.headers = const {},
  }) : _client = client;

  @override
  String get name => 'http';

  @override
  Future<void> send(List<AnalyticsEvent> events) async {
    try {
      final body = jsonEncode({
        'events': events.map((e) => e.toJson()).toList(),
        'sent_at': DateTime.now().toIso8601String(),
      });

      await _client.post(
        Uri.parse(endpoint),
        headers: {
          'Content-Type': 'application/json',
          ...headers,
        },
        body: body,
      );
    } catch (e) {
      debugPrint('HttpDestination: Failed to send events: $e');
    }
  }
}

class AnalyticsService {
  final List<AnalyticsDestination> _destinations;
  final List<AnalyticsEvent> _queue = [];
  Timer? _flushTimer;

  static const _batchSize = 20;
  static const _flushIntervalSeconds = 30;

  String? _userId;
  String? _sessionId;
  Map<String, String> _defaultProperties = {};

  AnalyticsService({required List<AnalyticsDestination> destinations})
      : _destinations = destinations;

  void initialize({
    required String userId,
    required String sessionId,
    Map<String, String> defaultProperties = const {},
  }) {
    _userId = userId;
    _sessionId = sessionId;
    _defaultProperties = defaultProperties;
    _startFlushTimer();
  }

  void track(AnalyticsEvent event) {
    final enriched = AnalyticsEvent(
      name: event.name,
      category: event.category,
      properties: {..._defaultProperties, ...event.properties},
      timestamp: event.timestamp,
      userId: _userId,
      sessionId: _sessionId,
      abTestAssignments: event.abTestAssignments,
    );

    _queue.add(enriched);

    if (_queue.length >= _batchSize) {
      flush();
    }
  }

  void trackScreenView(String screenName) {
    track(AnalyticsEvent.screenView(screenName));
  }

  void trackButtonTap(String buttonName, {String? screen}) {
    track(AnalyticsEvent.buttonTap(buttonName, screen: screen));
  }

  void trackPurchase(String productId, double amount, String currency) {
    track(AnalyticsEvent.purchase(productId, amount, currency));
  }

  Future<void> flush() async {
    if (_queue.isEmpty) return;

    final batch = List<AnalyticsEvent>.from(_queue);
    _queue.clear();

    await Future.wait(
      _destinations.map((d) => d.send(batch)),
    );
  }

  void _startFlushTimer() {
    _flushTimer?.cancel();
    _flushTimer = Timer.periodic(
      Duration(seconds: _flushIntervalSeconds),
      (_) => flush(),
    );
  }

  Future<void> dispose() async {
    _flushTimer?.cancel();
    await flush();
  }
}
```

## ขั้นตอนที่ 3327: Crash Reporting with Metadata

```dart
// lib/core/crash_reporting/crash_reporter.dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'package:flutter/foundation.dart';
import 'package:device_info_plus/device_info_plus.dart';
import 'package:package_info_plus/package_info_plus.dart';
import 'package:http/http.dart' as http;

class DeviceMetadata {
  final String platform;
  final String osVersion;
  final String model;
  final String? manufacturer;
  final bool isPhysicalDevice;
  final String appVersion;
  final String buildNumber;
  final String packageName;

  const DeviceMetadata({
    required this.platform,
    required this.osVersion,
    required this.model,
    this.manufacturer,
    required this.isPhysicalDevice,
    required this.appVersion,
    required this.buildNumber,
    required this.packageName,
  });

  Map<String, dynamic> toJson() => {
        'platform': platform,
        'os_version': osVersion,
        'model': model,
        if (manufacturer != null) 'manufacturer': manufacturer,
        'is_physical_device': isPhysicalDevice,
        'app_version': appVersion,
        'build_number': buildNumber,
        'package_name': packageName,
      };
}

class CrashReport {
  final String id;
  final String error;
  final String stackTrace;
  final DeviceMetadata device;
  final DateTime timestamp;
  final Map<String, dynamic> customData;
  final String? userId;
  final List<String> breadcrumbs;
  final Map<String, dynamic> abTestAssignments;

  CrashReport({
    required this.id,
    required this.error,
    required this.stackTrace,
    required this.device,
    required this.timestamp,
    this.customData = const {},
    this.userId,
    this.breadcrumbs = const [],
    this.abTestAssignments = const {},
  });

  Map<String, dynamic> toJson() => {
        'id': id,
        'error': error,
        'stack_trace': stackTrace,
        'device': device.toJson(),
        'timestamp': timestamp.toIso8601String(),
        'custom_data': customData,
        if (userId != null) 'user_id': userId,
        'breadcrumbs': breadcrumbs,
        'ab_tests': abTestAssignments,
      };
}

class CrashReporter {
  final String _endpoint;
  final http.Client _httpClient;

  DeviceMetadata? _deviceMetadata;
  String? _userId;
  final List<String> _breadcrumbs = [];
  final Map<String, dynamic> _customData = {};
  int _breadcrumbLimit = 50;

  CrashReporter({
    required String endpoint,
    required http.Client httpClient,
  })  : _endpoint = endpoint,
        _httpClient = httpClient;

  Future<void> initialize() async {
    _deviceMetadata = await _collectDeviceMetadata();
    _setupFlutterErrorHandler();
    _setupZoneErrorHandler();
  }

  void setUser(String userId) {
    _userId = userId;
  }

  void setCustomData(String key, dynamic value) {
    _customData[key] = value;
  }

  void addBreadcrumb(String message, {Map<String, dynamic>? data}) {
    final entry = data != null
        ? '$message: ${jsonEncode(data)}'
        : message;

    _breadcrumbs.add('[${DateTime.now().toIso8601String()}] $entry');

    if (_breadcrumbs.length > _breadcrumbLimit) {
      _breadcrumbs.removeAt(0);
    }
  }

  Future<void> reportError(
    dynamic error,
    StackTrace stackTrace, {
    Map<String, dynamic> extra = const {},
  }) async {
    if (_deviceMetadata == null) return;

    final report = CrashReport(
      id: _generateId(),
      error: error.toString(),
      stackTrace: stackTrace.toString(),
      device: _deviceMetadata!,
      timestamp: DateTime.now(),
      customData: {..._customData, ...extra},
      userId: _userId,
      breadcrumbs: List.from(_breadcrumbs),
    );

    await _sendReport(report);
  }

  void _setupFlutterErrorHandler() {
    FlutterError.onError = (details) {
      FlutterError.presentError(details);
      reportError(
        details.exception,
        details.stack ?? StackTrace.current,
        extra: {'flutter_error_context': details.context?.toString()},
      );
    };
  }

  void _setupZoneErrorHandler() {
    // Call this in runZonedGuarded in main.dart
  }

  Future<void> _sendReport(CrashReport report) async {
    try {
      await _httpClient.post(
        Uri.parse('$_endpoint/crashes'),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode(report.toJson()),
      );
      debugPrint('CrashReporter: Report sent: ${report.id}');
    } catch (e) {
      debugPrint('CrashReporter: Failed to send report: $e');
    }
  }

  Future<DeviceMetadata> _collectDeviceMetadata() async {
    final packageInfo = await PackageInfo.fromPlatform();
    final deviceInfo = DeviceInfoPlugin();

    if (Platform.isAndroid) {
      final android = await deviceInfo.androidInfo;
      return DeviceMetadata(
        platform: 'android',
        osVersion: android.version.release,
        model: android.model,
        manufacturer: android.manufacturer,
        isPhysicalDevice: android.isPhysicalDevice,
        appVersion: packageInfo.version,
        buildNumber: packageInfo.buildNumber,
        packageName: packageInfo.packageName,
      );
    } else if (Platform.isIOS) {
      final ios = await deviceInfo.iosInfo;
      return DeviceMetadata(
        platform: 'ios',
        osVersion: ios.systemVersion,
        model: ios.model,
        isPhysicalDevice: ios.isPhysicalDevice,
        appVersion: packageInfo.version,
        buildNumber: packageInfo.buildNumber,
        packageName: packageInfo.packageName,
      );
    }

    return DeviceMetadata(
      platform: defaultTargetPlatform.name,
      osVersion: 'unknown',
      model: 'unknown',
      isPhysicalDevice: true,
      appVersion: packageInfo.version,
      buildNumber: packageInfo.buildNumber,
      packageName: packageInfo.packageName,
    );
  }

  String _generateId() {
    final now = DateTime.now().millisecondsSinceEpoch;
    final random = now.hashCode.abs();
    return '${now.toRadixString(16)}-${random.toRadixString(16)}';
  }
}

// main.dart integration example
Future<void> mainWithCrashReporting() async {
  WidgetsFlutterBinding.ensureInitialized();

  final crashReporter = CrashReporter(
    endpoint: 'https://crashes.example.com',
    httpClient: http.Client(),
  );
  await crashReporter.initialize();

  runZonedGuarded(
    () {
      runApp(const Placeholder());
    },
    (error, stackTrace) {
      crashReporter.reportError(error, stackTrace);
    },
  );
}
```

## ขั้นตอนที่ 3328: Enterprise App with All Patterns Combined

```dart
// lib/screens/enterprise_demo_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Providers
final featureFlagServiceProvider = Provider<FeatureFlagService>((ref) {
  throw UnimplementedError();
});

final abTestServiceProvider = Provider<ABTestService>((ref) {
  throw UnimplementedError();
});

final analyticsServiceProvider = Provider<AnalyticsService>((ref) {
  throw UnimplementedError();
});

class EnterpriseDemoScreen extends ConsumerStatefulWidget {
  const EnterpriseDemoScreen({super.key});

  @override
  ConsumerState<EnterpriseDemoScreen> createState() =>
      _EnterpriseDemoScreenState();
}

class _EnterpriseDemoScreenState
    extends ConsumerState<EnterpriseDemoScreen> {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addPostFrameCallback((_) {
      ref.read(analyticsServiceProvider).trackScreenView('enterprise_demo');
    });
  }

  @override
  Widget build(BuildContext context) {
    final flags = ref.watch(featureFlagServiceProvider);
    final abTest = ref.watch(abTestServiceProvider);
    final analytics = ref.watch(analyticsServiceProvider);

    final showNewCheckout = abTest.isInVariant('checkout_flow', 'new_design');
    final ctaText = flags.getString('cta_button_text', defaultValue: 'Get Started');
    final showPromo = flags.isEnabled('show_promo_banner');

    return Scaffold(
      appBar: AppBar(
        title: const Text('Enterprise Demo'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: () async {
              analytics.trackButtonTap('refresh_flags');
              await flags.fetchFlags();
              if (mounted) {
                ScaffoldMessenger.of(context).showSnackBar(
                  const SnackBar(content: Text('Flags refreshed')),
                );
              }
            },
          ),
        ],
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          if (showPromo) const _PromoBanner(),
          const SizedBox(height: 16),
          _ABTestVariantCard(
            isNewDesign: showNewCheckout,
            onTap: () => analytics.trackButtonTap(
              'checkout_cta',
              screen: 'enterprise_demo',
            ),
            ctaText: ctaText,
          ),
          const SizedBox(height: 16),
          const _FeatureFlagList(),
          const SizedBox(height: 16),
          const _AnalyticsDebugPanel(),
        ],
      ),
    );
  }
}

class _PromoBanner extends StatelessWidget {
  const _PromoBanner();

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(12),
      decoration: BoxDecoration(
        gradient: LinearGradient(
          colors: [Colors.purple.shade400, Colors.blue.shade400],
        ),
        borderRadius: BorderRadius.circular(12),
      ),
      child: const Text(
        '🎉 Special Promo Active!',
        style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold),
        textAlign: TextAlign.center,
      ),
    );
  }
}

class _ABTestVariantCard extends StatelessWidget {
  final bool isNewDesign;
  final VoidCallback onTap;
  final String ctaText;

  const _ABTestVariantCard({
    required this.isNewDesign,
    required this.onTap,
    required this.ctaText,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            Text(
              isNewDesign ? 'New Checkout Design (Variant B)' : 'Classic Checkout (Control A)',
              style: Theme.of(context).textTheme.titleMedium,
            ),
            const SizedBox(height: 12),
            SizedBox(
              width: double.infinity,
              child: ElevatedButton(
                style: isNewDesign
                    ? ElevatedButton.styleFrom(backgroundColor: Colors.purple)
                    : null,
                onPressed: onTap,
                child: Text(ctaText),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _FeatureFlagList extends StatelessWidget {
  const _FeatureFlagList();

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Feature Flags',
              style: Theme.of(context).textTheme.titleMedium,
            ),
            const Divider(),
            const _FlagRow(name: 'show_promo_banner', value: 'Active'),
            const _FlagRow(name: 'cta_button_text', value: 'Get Started'),
            const _FlagRow(name: 'enable_new_feature', value: 'Disabled'),
          ],
        ),
      ),
    );
  }
}

class _FlagRow extends StatelessWidget {
  final String name;
  final String value;
  const _FlagRow({required this.name, required this.value});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(name, style: const TextStyle(fontFamily: 'monospace')),
          Chip(
            label: Text(value),
            backgroundColor: Colors.blue.shade100,
          ),
        ],
      ),
    );
  }
}

class _AnalyticsDebugPanel extends StatelessWidget {
  const _AnalyticsDebugPanel();

  @override
  Widget build(BuildContext context) {
    return Card(
      color: Colors.grey.shade100,
      child: const Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Analytics Debug',
              style: TextStyle(fontWeight: FontWeight.bold),
            ),
            SizedBox(height: 8),
            Text('Events queued: 3'),
            Text('Last flush: 30s ago'),
            Text('Destinations: console, http'),
          ],
        ),
      ),
    );
  }
}
```

---

**← [Part 85](part-85-advanced-animations.md)**
**ต่อไป: [Part 87 →](part-87-flutter-hooks.md)**
