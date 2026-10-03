# Part 56: Flutter Plugins & Platform Channels
## ขั้นตอนที่ 2121-2160

## 🎯 เป้าหมายของ Part นี้
- เข้าใจสถาปัตยกรรม Flutter Plugin และ Platform Channel
- สร้าง MethodChannel และ EventChannel อย่างมืออาชีพ
- เขียนโค้ดฝั่ง Kotlin (Android) และ Swift (iOS)
- กำหนดโครงสร้าง Plugin Project ที่ถูกต้อง
- เตรียม pubspec.yaml สำหรับ publish บน pub.dev

---

## ขั้นตอนที่ 2121: โครงสร้าง Flutter Plugin Project

```
battery_plus_custom/
├── lib/
│   ├── battery_plus_custom.dart          # Public API
│   └── src/
│       ├── battery_plus_custom_method_channel.dart
│       └── battery_plus_custom_platform_interface.dart
├── android/
│   └── src/main/kotlin/com/example/battery_plus_custom/
│       └── BatteryPlusCustomPlugin.kt
├── ios/
│   └── Classes/
│       ├── BatteryPlusCustomPlugin.swift
│       └── BatteryPlusCustomPlugin.h
├── example/
│   └── lib/
│       └── main.dart
├── test/
│   └── battery_plus_custom_test.dart
└── pubspec.yaml
```

---

## ขั้นตอนที่ 2122: pubspec.yaml สำหรับ Plugin

```yaml
# pubspec.yaml
name: battery_plus_custom
description: A Flutter plugin to get battery level and status on Android and iOS.
version: 1.0.0
homepage: https://github.com/yourname/battery_plus_custom

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

flutter:
  plugin:
    platforms:
      android:
        kotlinClass: com.example.battery_plus_custom.BatteryPlusCustomPlugin
        package: com.example.battery_plus_custom
      ios:
        classPrefix: ""
        pluginClass: BatteryPlusCustomPlugin

dependencies:
  flutter:
    sdk: flutter
  plugin_platform_interface: ^2.1.5

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
  mockito: ^5.4.2
  build_runner: ^2.4.6
```

---

## ขั้นตอนที่ 2123: Platform Interface (Abstract Contract)

```dart
// lib/src/battery_plus_custom_platform_interface.dart
import 'package:plugin_platform_interface/plugin_platform_interface.dart';

/// Battery status enumeration
enum BatteryStatus {
  charging,
  discharging,
  full,
  unknown,
}

/// Battery information data class
class BatteryInfo {
  final int level;
  final BatteryStatus status;
  final double? voltage;
  final double? temperature;

  const BatteryInfo({
    required this.level,
    required this.status,
    this.voltage,
    this.temperature,
  });

  factory BatteryInfo.fromMap(Map<dynamic, dynamic> map) {
    return BatteryInfo(
      level: map['level'] as int,
      status: _parseStatus(map['status'] as String),
      voltage: map['voltage'] != null ? (map['voltage'] as num).toDouble() : null,
      temperature: map['temperature'] != null
          ? (map['temperature'] as num).toDouble()
          : null,
    );
  }

  static BatteryStatus _parseStatus(String status) {
    switch (status) {
      case 'charging':
        return BatteryStatus.charging;
      case 'discharging':
        return BatteryStatus.discharging;
      case 'full':
        return BatteryStatus.full;
      default:
        return BatteryStatus.unknown;
    }
  }

  @override
  String toString() =>
      'BatteryInfo(level: $level, status: $status, voltage: $voltage, temperature: $temperature)';
}

/// Platform interface abstract class
abstract class BatteryPlusCustomPlatform extends PlatformInterface {
  BatteryPlusCustomPlatform() : super(token: _token);

  static final Object _token = Object();

  static BatteryPlusCustomPlatform _instance = MethodChannelBatteryPlusCustom();

  static BatteryPlusCustomPlatform get instance => _instance;

  static set instance(BatteryPlusCustomPlatform instance) {
    PlatformInterface.verifyToken(instance, _token);
    _instance = instance;
  }

  Future<int> getBatteryLevel();
  Future<BatteryInfo> getBatteryInfo();
  Stream<BatteryStatus> get onBatteryStatusChanged;
}

// Forward reference - implemented in method_channel file
class MethodChannelBatteryPlusCustom extends BatteryPlusCustomPlatform {
  @override
  Future<int> getBatteryLevel() => throw UnimplementedError();

  @override
  Future<BatteryInfo> getBatteryInfo() => throw UnimplementedError();

  @override
  Stream<BatteryStatus> get onBatteryStatusChanged => throw UnimplementedError();
}
```

---

## ขั้นตอนที่ 2124: MethodChannel & EventChannel Implementation (Dart)

```dart
// lib/src/battery_plus_custom_method_channel.dart
import 'package:flutter/foundation.dart';
import 'package:flutter/services.dart';
import 'battery_plus_custom_platform_interface.dart';

/// Method Channel implementation
class MethodChannelBatteryPlusCustomImpl extends BatteryPlusCustomPlatform {
  static const String _channelName = 'com.example/battery_plus_custom';
  static const String _eventChannelName = 'com.example/battery_plus_custom_events';

  @visibleForTesting
  final MethodChannel methodChannel = const MethodChannel(_channelName);

  final EventChannel _eventChannel = const EventChannel(_eventChannelName);

  @override
  Future<int> getBatteryLevel() async {
    try {
      final int? level = await methodChannel.invokeMethod<int>('getBatteryLevel');
      if (level == null) throw PlatformException(code: 'NULL_RESULT');
      return level;
    } on PlatformException catch (e) {
      throw BatteryException(
        code: e.code,
        message: e.message ?? 'Failed to get battery level',
      );
    }
  }

  @override
  Future<BatteryInfo> getBatteryInfo() async {
    try {
      final Map<dynamic, dynamic>? result =
          await methodChannel.invokeMethod<Map<dynamic, dynamic>>('getBatteryInfo');
      if (result == null) throw PlatformException(code: 'NULL_RESULT');
      return BatteryInfo.fromMap(result);
    } on PlatformException catch (e) {
      throw BatteryException(
        code: e.code,
        message: e.message ?? 'Failed to get battery info',
      );
    }
  }

  @override
  Stream<BatteryStatus> get onBatteryStatusChanged {
    return _eventChannel.receiveBroadcastStream().map((dynamic event) {
      return BatteryInfo._parseStatus(event as String);
    });
  }
}

/// Custom exception for battery plugin errors
class BatteryException implements Exception {
  final String code;
  final String message;

  const BatteryException({required this.code, required this.message});

  @override
  String toString() => 'BatteryException(code: $code, message: $message)';
}
```

---

## ขั้นตอนที่ 2125: Public API (Main Library File)

```dart
// lib/battery_plus_custom.dart
library battery_plus_custom;

export 'src/battery_plus_custom_platform_interface.dart'
    show BatteryInfo, BatteryStatus;
export 'src/battery_plus_custom_method_channel.dart' show BatteryException;

import 'src/battery_plus_custom_method_channel.dart';
import 'src/battery_plus_custom_platform_interface.dart';

/// Main public API for the battery plugin
class BatteryPlusCustom {
  // Register the implementation
  static void _ensureRegistered() {
    BatteryPlusCustomPlatform.instance = MethodChannelBatteryPlusCustomImpl();
  }

  static final BatteryPlusCustom _instance = BatteryPlusCustom._();
  factory BatteryPlusCustom() => _instance;
  BatteryPlusCustom._() {
    _ensureRegistered();
  }

  /// Returns current battery level as percentage (0-100)
  Future<int> getBatteryLevel() {
    return BatteryPlusCustomPlatform.instance.getBatteryLevel();
  }

  /// Returns full battery information
  Future<BatteryInfo> getBatteryInfo() {
    return BatteryPlusCustomPlatform.instance.getBatteryInfo();
  }

  /// Stream of battery status changes
  Stream<BatteryStatus> get onBatteryStatusChanged {
    return BatteryPlusCustomPlatform.instance.onBatteryStatusChanged;
  }
}
```

---

## ขั้นตอนที่ 2126: Android Implementation (Kotlin)

```kotlin
// android/src/main/kotlin/com/example/battery_plus_custom/BatteryPlusCustomPlugin.kt
package com.example.battery_plus_custom

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.os.BatteryManager
import android.os.Build
import androidx.annotation.NonNull
import io.flutter.embedding.engine.plugins.FlutterPlugin
import io.flutter.plugin.common.EventChannel
import io.flutter.plugin.common.MethodCall
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugin.common.MethodChannel.MethodCallHandler
import io.flutter.plugin.common.MethodChannel.Result

class BatteryPlusCustomPlugin : FlutterPlugin, MethodCallHandler,
    EventChannel.StreamHandler {

    private lateinit var methodChannel: MethodChannel
    private lateinit var eventChannel: EventChannel
    private lateinit var context: Context
    private var chargingStateChangeReceiver: BroadcastReceiver? = null
    private var eventSink: EventChannel.EventSink? = null

    companion object {
        const val METHOD_CHANNEL = "com.example/battery_plus_custom"
        const val EVENT_CHANNEL = "com.example/battery_plus_custom_events"
    }

    override fun onAttachedToEngine(@NonNull flutterPluginBinding: FlutterPlugin.FlutterPluginBinding) {
        context = flutterPluginBinding.applicationContext

        methodChannel = MethodChannel(
            flutterPluginBinding.binaryMessenger,
            METHOD_CHANNEL
        )
        methodChannel.setMethodCallHandler(this)

        eventChannel = EventChannel(
            flutterPluginBinding.binaryMessenger,
            EVENT_CHANNEL
        )
        eventChannel.setStreamHandler(this)
    }

    override fun onMethodCall(@NonNull call: MethodCall, @NonNull result: Result) {
        when (call.method) {
            "getBatteryLevel" -> handleGetBatteryLevel(result)
            "getBatteryInfo" -> handleGetBatteryInfo(result)
            else -> result.notImplemented()
        }
    }

    private fun handleGetBatteryLevel(result: Result) {
        val batteryLevel = getBatteryLevel()
        if (batteryLevel == -1) {
            result.error("UNAVAILABLE", "Battery level not available.", null)
        } else {
            result.success(batteryLevel)
        }
    }

    private fun handleGetBatteryInfo(result: Result) {
        val batteryManager = context.getSystemService(Context.BATTERY_SERVICE) as BatteryManager
        val intentFilter = IntentFilter(Intent.ACTION_BATTERY_CHANGED)
        val batteryIntent = context.registerReceiver(null, intentFilter)

        if (batteryIntent == null) {
            result.error("UNAVAILABLE", "Battery information not available.", null)
            return
        }

        val level = batteryIntent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
        val scale = batteryIntent.getIntExtra(BatteryManager.EXTRA_SCALE, -1)
        val batteryPct = (level * 100 / scale.toFloat()).toInt()

        val statusInt = batteryIntent.getIntExtra(BatteryManager.EXTRA_STATUS, -1)
        val status = when (statusInt) {
            BatteryManager.BATTERY_STATUS_CHARGING -> "charging"
            BatteryManager.BATTERY_STATUS_FULL -> "full"
            BatteryManager.BATTERY_STATUS_DISCHARGING -> "discharging"
            BatteryManager.BATTERY_STATUS_NOT_CHARGING -> "discharging"
            else -> "unknown"
        }

        val voltage = batteryIntent.getIntExtra(BatteryManager.EXTRA_VOLTAGE, -1).toDouble() / 1000.0
        val temperature = batteryIntent.getIntExtra(BatteryManager.EXTRA_TEMPERATURE, -1).toDouble() / 10.0

        val info = mapOf(
            "level" to batteryPct,
            "status" to status,
            "voltage" to if (voltage > 0) voltage else null,
            "temperature" to if (temperature > 0) temperature else null
        )

        result.success(info)
    }

    private fun getBatteryLevel(): Int {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            val batteryManager = context.getSystemService(Context.BATTERY_SERVICE) as BatteryManager
            batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
        } else {
            val intent = context.registerReceiver(null,
                IntentFilter(Intent.ACTION_BATTERY_CHANGED))
            val level = intent?.getIntExtra(BatteryManager.EXTRA_LEVEL, -1) ?: -1
            val scale = intent?.getIntExtra(BatteryManager.EXTRA_SCALE, -1) ?: -1
            if (level == -1 || scale == -1) -1 else (level * 100 / scale)
        }
    }

    override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
        eventSink = events
        chargingStateChangeReceiver = createChargingStateChangeReceiver()
        context.registerReceiver(
            chargingStateChangeReceiver,
            IntentFilter(Intent.ACTION_BATTERY_CHANGED)
        )
    }

    override fun onCancel(arguments: Any?) {
        chargingStateChangeReceiver?.let {
            context.unregisterReceiver(it)
        }
        chargingStateChangeReceiver = null
        eventSink = null
    }

    private fun createChargingStateChangeReceiver(): BroadcastReceiver {
        return object : BroadcastReceiver() {
            override fun onReceive(context: Context, intent: Intent) {
                val status = intent.getIntExtra(BatteryManager.EXTRA_STATUS, -1)
                val statusString = when (status) {
                    BatteryManager.BATTERY_STATUS_CHARGING -> "charging"
                    BatteryManager.BATTERY_STATUS_FULL -> "full"
                    BatteryManager.BATTERY_STATUS_DISCHARGING,
                    BatteryManager.BATTERY_STATUS_NOT_CHARGING -> "discharging"
                    else -> "unknown"
                }
                eventSink?.success(statusString)
            }
        }
    }

    override fun onDetachedFromEngine(@NonNull binding: FlutterPlugin.FlutterPluginBinding) {
        methodChannel.setMethodCallHandler(null)
        eventChannel.setStreamHandler(null)
    }
}
```

---

## ขั้นตอนที่ 2127: iOS Implementation (Swift)

```swift
// ios/Classes/BatteryPlusCustomPlugin.swift
import Flutter
import UIKit

public class BatteryPlusCustomPlugin: NSObject, FlutterPlugin, FlutterStreamHandler {
    
    private static let methodChannelName = "com.example/battery_plus_custom"
    private static let eventChannelName = "com.example/battery_plus_custom_events"
    
    private var eventSink: FlutterEventSink?
    
    public static func register(with registrar: FlutterPluginRegistrar) {
        let instance = BatteryPlusCustomPlugin()
        
        let methodChannel = FlutterMethodChannel(
            name: methodChannelName,
            binaryMessenger: registrar.messenger()
        )
        registrar.addMethodCallDelegate(instance, channel: methodChannel)
        
        let eventChannel = FlutterEventChannel(
            name: eventChannelName,
            binaryMessenger: registrar.messenger()
        )
        eventChannel.setStreamHandler(instance)
    }
    
    public func handle(_ call: FlutterMethodCall, result: @escaping FlutterResult) {
        switch call.method {
        case "getBatteryLevel":
            handleGetBatteryLevel(result: result)
        case "getBatteryInfo":
            handleGetBatteryInfo(result: result)
        default:
            result(FlutterMethodNotImplemented)
        }
    }
    
    private func handleGetBatteryLevel(result: FlutterResult) {
        UIDevice.current.isBatteryMonitoringEnabled = true
        let batteryLevel = UIDevice.current.batteryLevel
        
        if batteryLevel < 0 {
            result(FlutterError(
                code: "UNAVAILABLE",
                message: "Battery info unavailable",
                details: nil
            ))
        } else {
            result(Int(batteryLevel * 100))
        }
    }
    
    private func handleGetBatteryInfo(result: FlutterResult) {
        UIDevice.current.isBatteryMonitoringEnabled = true
        
        let batteryLevel = UIDevice.current.batteryLevel
        let batteryState = UIDevice.current.batteryState
        
        let statusString: String
        switch batteryState {
        case .charging:
            statusString = "charging"
        case .full:
            statusString = "full"
        case .unplugged:
            statusString = "discharging"
        default:
            statusString = "unknown"
        }
        
        let info: [String: Any?] = [
            "level": batteryLevel < 0 ? 0 : Int(batteryLevel * 100),
            "status": statusString,
            "voltage": nil,       // iOS doesn't expose voltage
            "temperature": nil    // iOS doesn't expose temperature
        ]
        
        result(info.compactMapValues { $0 })
    }
    
    // MARK: - FlutterStreamHandler
    
    public func onListen(
        withArguments arguments: Any?,
        eventSink events: @escaping FlutterEventSink
    ) -> FlutterError? {
        self.eventSink = events
        UIDevice.current.isBatteryMonitoringEnabled = true
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(onBatteryStateDidChange),
            name: UIDevice.batteryStateDidChangeNotification,
            object: nil
        )
        return nil
    }
    
    public func onCancel(withArguments arguments: Any?) -> FlutterError? {
        NotificationCenter.default.removeObserver(self)
        eventSink = nil
        UIDevice.current.isBatteryMonitoringEnabled = false
        return nil
    }
    
    @objc private func onBatteryStateDidChange(_ notification: Notification) {
        guard let sink = eventSink else { return }
        
        switch UIDevice.current.batteryState {
        case .charging:
            sink("charging")
        case .full:
            sink("full")
        case .unplugged:
            sink("discharging")
        default:
            sink("unknown")
        }
    }
}
```

---

## ขั้นตอนที่ 2128: Example App - Full Working Demo

```dart
// example/lib/main.dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:battery_plus_custom/battery_plus_custom.dart';

void main() {
  runApp(const BatteryExampleApp());
}

class BatteryExampleApp extends StatelessWidget {
  const BatteryExampleApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Battery Plugin Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.green),
        useMaterial3: true,
      ),
      home: const BatteryHomePage(),
    );
  }
}

class BatteryHomePage extends StatefulWidget {
  const BatteryHomePage({super.key});

  @override
  State<BatteryHomePage> createState() => _BatteryHomePageState();
}

class _BatteryHomePageState extends State<BatteryHomePage> {
  final _battery = BatteryPlusCustom();
  int? _batteryLevel;
  BatteryInfo? _batteryInfo;
  BatteryStatus? _currentStatus;
  StreamSubscription<BatteryStatus>? _statusSubscription;
  String? _error;
  bool _loading = false;

  @override
  void initState() {
    super.initState();
    _listenToStatusChanges();
    _loadBatteryInfo();
  }

  void _listenToStatusChanges() {
    _statusSubscription = _battery.onBatteryStatusChanged.listen(
      (status) => setState(() => _currentStatus = status),
      onError: (e) => setState(() => _error = e.toString()),
    );
  }

  Future<void> _loadBatteryInfo() async {
    setState(() => _loading = true);
    try {
      final info = await _battery.getBatteryInfo();
      setState(() {
        _batteryInfo = info;
        _batteryLevel = info.level;
        _error = null;
      });
    } on BatteryException catch (e) {
      setState(() => _error = '${e.code}: ${e.message}');
    } finally {
      setState(() => _loading = false);
    }
  }

  @override
  void dispose() {
    _statusSubscription?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Battery Plugin Demo'),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            _buildBatteryCard(),
            const SizedBox(height: 16),
            if (_error != null)
              Card(
                color: Colors.red.shade100,
                child: Padding(
                  padding: const EdgeInsets.all(16.0),
                  child: Text(
                    'Error: $_error',
                    style: TextStyle(color: Colors.red.shade800),
                  ),
                ),
              ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: _loading ? null : _loadBatteryInfo,
              child: _loading
                  ? const SizedBox(
                      height: 20,
                      width: 20,
                      child: CircularProgressIndicator(strokeWidth: 2),
                    )
                  : const Text('Refresh'),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildBatteryCard() {
    return Card(
      elevation: 4,
      child: Padding(
        padding: const EdgeInsets.all(20.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Icon(
                  _getBatteryIcon(),
                  size: 48,
                  color: _getBatteryColor(),
                ),
                const SizedBox(width: 16),
                Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      '${_batteryLevel ?? '--'}%',
                      style: Theme.of(context).textTheme.headlineLarge?.copyWith(
                            color: _getBatteryColor(),
                            fontWeight: FontWeight.bold,
                          ),
                    ),
                    Text(
                      _batteryInfo?.status.name.toUpperCase() ?? 'Unknown',
                      style: Theme.of(context).textTheme.bodyMedium,
                    ),
                  ],
                ),
              ],
            ),
            if (_batteryInfo != null) ...[
              const Divider(height: 24),
              _buildInfoRow('Voltage',
                  _batteryInfo!.voltage != null
                      ? '${_batteryInfo!.voltage!.toStringAsFixed(2)}V'
                      : 'N/A'),
              _buildInfoRow('Temperature',
                  _batteryInfo!.temperature != null
                      ? '${_batteryInfo!.temperature!.toStringAsFixed(1)}°C'
                      : 'N/A'),
              _buildInfoRow('Live Status',
                  _currentStatus?.name.toUpperCase() ?? 'Listening...'),
            ],
          ],
        ),
      ),
    );
  }

  Widget _buildInfoRow(String label, String value) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(label, style: const TextStyle(color: Colors.grey)),
          Text(value, style: const TextStyle(fontWeight: FontWeight.w500)),
        ],
      ),
    );
  }

  IconData _getBatteryIcon() {
    final level = _batteryLevel ?? 100;
    final status = _batteryInfo?.status ?? BatteryStatus.unknown;

    if (status == BatteryStatus.charging) return Icons.battery_charging_full;
    if (level > 80) return Icons.battery_full;
    if (level > 60) return Icons.battery_5_bar;
    if (level > 40) return Icons.battery_4_bar;
    if (level > 20) return Icons.battery_2_bar;
    return Icons.battery_alert;
  }

  Color _getBatteryColor() {
    final level = _batteryLevel ?? 100;
    if (level > 60) return Colors.green;
    if (level > 30) return Colors.orange;
    return Colors.red;
  }
}
```

---

## ขั้นตอนที่ 2129: Unit Tests for the Plugin

```dart
// test/battery_plus_custom_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:battery_plus_custom/battery_plus_custom.dart';
import 'package:battery_plus_custom/src/battery_plus_custom_method_channel.dart';
import 'package:battery_plus_custom/src/battery_plus_custom_platform_interface.dart';
import 'package:flutter/services.dart';

void main() {
  TestWidgetsFlutterBinding.ensureInitialized();

  group('BatteryPlusCustom', () {
    late MethodChannelBatteryPlusCustomImpl platform;
    const MethodChannel channel = MethodChannel('com.example/battery_plus_custom');

    setUp(() {
      platform = MethodChannelBatteryPlusCustomImpl();
      BatteryPlusCustomPlatform.instance = platform;
    });

    tearDown(() {
      TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
          .setMockMethodCallHandler(channel, null);
    });

    test('getBatteryLevel returns correct level', () async {
      TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
          .setMockMethodCallHandler(channel, (MethodCall call) async {
        if (call.method == 'getBatteryLevel') return 85;
        return null;
      });

      final battery = BatteryPlusCustom();
      final level = await battery.getBatteryLevel();
      expect(level, equals(85));
    });

    test('getBatteryInfo returns complete info', () async {
      TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
          .setMockMethodCallHandler(channel, (MethodCall call) async {
        if (call.method == 'getBatteryInfo') {
          return {
            'level': 75,
            'status': 'charging',
            'voltage': 4.2,
            'temperature': 28.5,
          };
        }
        return null;
      });

      final battery = BatteryPlusCustom();
      final info = await battery.getBatteryInfo();

      expect(info.level, equals(75));
      expect(info.status, equals(BatteryStatus.charging));
      expect(info.voltage, closeTo(4.2, 0.01));
      expect(info.temperature, closeTo(28.5, 0.01));
    });

    test('getBatteryLevel throws BatteryException on platform error', () async {
      TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
          .setMockMethodCallHandler(channel, (MethodCall call) async {
        throw PlatformException(
          code: 'UNAVAILABLE',
          message: 'Battery unavailable',
        );
      });

      final battery = BatteryPlusCustom();
      expect(
        () => battery.getBatteryLevel(),
        throwsA(isA<BatteryException>()),
      );
    });

    test('BatteryInfo.fromMap parses all statuses correctly', () {
      final statuses = ['charging', 'discharging', 'full', 'unknown'];
      final expected = [
        BatteryStatus.charging,
        BatteryStatus.discharging,
        BatteryStatus.full,
        BatteryStatus.unknown,
      ];

      for (int i = 0; i < statuses.length; i++) {
        final info = BatteryInfo.fromMap({
          'level': 50,
          'status': statuses[i],
        });
        expect(info.status, equals(expected[i]));
      }
    });
  });
}
```

---

## ขั้นตอนที่ 2130: Publishing Checklist & CHANGELOG.md

```markdown
# CHANGELOG.md

## 1.0.0

- Initial release
- MethodChannel support for getBatteryLevel and getBatteryInfo
- EventChannel support for real-time battery status changes
- Android (Kotlin) implementation
- iOS (Swift) implementation
- Null-safe API

---

# Publishing Steps:
# 1. dart pub publish --dry-run
# 2. Ensure all license headers are present
# 3. dart analyze
# 4. flutter test
# 5. dart pub publish
```

```yaml
# analysis_options.yaml
include: package:flutter_lints/flutter.yaml

linter:
  rules:
    - always_declare_return_types
    - avoid_print
    - prefer_const_constructors
    - prefer_final_fields
    - use_key_in_widget_constructors

analyzer:
  errors:
    missing_required_param: error
    missing_return: error
```

---

**← [Part 55](part-55-advanced-animations.md)**
**ต่อไป: [Part 57 →](part-57-offline-first-architecture.md)**
