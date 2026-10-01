# Part 25: Platform Channels
## ขั้นตอนที่ 881-920

---

## 🎯 เป้าหมายของ Part นี้

- MethodChannel (Dart ↔ Native)
- EventChannel (Native → Dart stream)
- BasicMessageChannel
- Android/iOS native code integration
- Plugins

---

## ขั้นตอนที่ 881: MethodChannel พื้นฐาน

```dart
// ─── Dart side ───
import 'package:flutter/services.dart';

class BatteryService {
  static const MethodChannel _channel = MethodChannel('com.example/battery');

  // เรียก native code
  Future<int> getBatteryLevel() async {
    try {
      int level = await _channel.invokeMethod('getBatteryLevel');
      return level;
    } on PlatformException catch (e) {
      throw Exception('ไม่สามารถดึงข้อมูล battery: ${e.message}');
    }
  }
}

// ─── Android (Kotlin) side ─── lib/main/kotlin/com/.../MainActivity.kt
/*
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel
import android.content.Intent
import android.content.IntentFilter
import android.os.BatteryManager

class MainActivity : FlutterActivity() {
    private val CHANNEL = "com.example/battery"

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL)
            .setMethodCallHandler { call, result ->
                if (call.method == "getBatteryLevel") {
                    val batteryLevel = getBatteryLevel()
                    if (batteryLevel != -1) {
                        result.success(batteryLevel)
                    } else {
                        result.error("UNAVAILABLE", "Battery level not available.", null)
                    }
                } else {
                    result.notImplemented()
                }
            }
    }

    private fun getBatteryLevel(): Int {
        val batteryIntent = registerReceiver(null, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
        return batteryIntent?.getIntExtra(BatteryManager.EXTRA_LEVEL, -1) ?: -1
    }
}
*/

// ─── iOS (Swift) side ─── ios/Runner/AppDelegate.swift
/*
import UIKit
import Flutter

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
    override func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        let controller = window?.rootViewController as! FlutterViewController
        let batteryChannel = FlutterMethodChannel(
            name: "com.example/battery",
            binaryMessenger: controller.binaryMessenger
        )
        
        batteryChannel.setMethodCallHandler { call, result in
            if call.method == "getBatteryLevel" {
                self.receiveBatteryLevel(result: result)
            } else {
                result(FlutterMethodNotImplemented)
            }
        }
        
        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }
    
    private func receiveBatteryLevel(result: FlutterResult) {
        UIDevice.current.isBatteryMonitoringEnabled = true
        if UIDevice.current.batteryState == UIDevice.BatteryState.unknown {
            result(FlutterError(
                code: "UNAVAILABLE",
                message: "Battery info unavailable",
                details: nil
            ))
        } else {
            result(Int(UIDevice.current.batteryLevel * 100))
        }
    }
}
*/
```

---

## ขั้นตอนที่ 882: EventChannel (Streaming ข้อมูล)

```dart
// ─── Dart side ───
import 'package:flutter/services.dart';

class SensorService {
  static const EventChannel _channel = EventChannel('com.example/sensors');

  // รับ stream ของ sensor data
  Stream<Map<String, double>> get accelerometerStream {
    return _channel.receiveBroadcastStream().map((event) {
      Map<String, dynamic> data = Map<String, dynamic>.from(event as Map);
      return {
        'x': (data['x'] as num).toDouble(),
        'y': (data['y'] as num).toDouble(),
        'z': (data['z'] as num).toDouble(),
      };
    });
  }
}

// ─── Widget ───
class SensorWidget extends StatelessWidget {
  const SensorWidget({super.key});

  @override
  Widget build(BuildContext context) {
    SensorService sensorService = SensorService();

    return StreamBuilder<Map<String, double>>(
      stream: sensorService.accelerometerStream,
      builder: (context, snapshot) {
        if (!snapshot.hasData) {
          return const CircularProgressIndicator();
        }

        Map<String, double> data = snapshot.data!;
        return Column(
          children: [
            Text('X: ${data['x']?.toStringAsFixed(2)}'),
            Text('Y: ${data['y']?.toStringAsFixed(2)}'),
            Text('Z: ${data['z']?.toStringAsFixed(2)}'),
          ],
        );
      },
    );
  }
}
```

---

## ขั้นตอนที่ 883: Device Info Plugin

```dart
// pubspec.yaml:
// device_info_plus: ^10.1.0
// package_info_plus: ^8.0.0

import 'package:device_info_plus/device_info_plus.dart';
import 'package:package_info_plus/package_info_plus.dart';
import 'dart:io';

class DeviceInfo {
  final String deviceName;
  final String osVersion;
  final String manufacturer;
  final bool isPhysicalDevice;

  const DeviceInfo({
    required this.deviceName,
    required this.osVersion,
    required this.manufacturer,
    required this.isPhysicalDevice,
  });
}

class DeviceInfoService {
  final DeviceInfoPlugin _deviceInfoPlugin = DeviceInfoPlugin();

  Future<DeviceInfo> getDeviceInfo() async {
    if (Platform.isAndroid) {
      AndroidDeviceInfo info = await _deviceInfoPlugin.androidInfo;
      return DeviceInfo(
        deviceName: '${info.brand} ${info.model}',
        osVersion: 'Android ${info.version.release}',
        manufacturer: info.manufacturer,
        isPhysicalDevice: info.isPhysicalDevice,
      );
    } else if (Platform.isIOS) {
      IosDeviceInfo info = await _deviceInfoPlugin.iosInfo;
      return DeviceInfo(
        deviceName: info.name,
        osVersion: '${info.systemName} ${info.systemVersion}',
        manufacturer: 'Apple',
        isPhysicalDevice: info.isPhysicalDevice,
      );
    }
    return const DeviceInfo(
      deviceName: 'Unknown',
      osVersion: 'Unknown',
      manufacturer: 'Unknown',
      isPhysicalDevice: false,
    );
  }

  Future<Map<String, String>> getAppInfo() async {
    PackageInfo packageInfo = await PackageInfo.fromPlatform();
    return {
      'appName': packageInfo.appName,
      'packageName': packageInfo.packageName,
      'version': packageInfo.version,
      'buildNumber': packageInfo.buildNumber,
    };
  }
}

// ─── Widget ───
class DeviceInfoScreen extends StatelessWidget {
  const DeviceInfoScreen({super.key});

  @override
  Widget build(BuildContext context) {
    DeviceInfoService service = DeviceInfoService();

    return Scaffold(
      appBar: AppBar(title: const Text('Device Info')),
      body: FutureBuilder<DeviceInfo>(
        future: service.getDeviceInfo(),
        builder: (context, snapshot) {
          if (!snapshot.hasData) {
            return const Center(child: CircularProgressIndicator());
          }

          DeviceInfo info = snapshot.data!;
          return ListView(
            children: [
              ListTile(
                leading: const Icon(Icons.phone_android),
                title: const Text('Device'),
                subtitle: Text(info.deviceName),
              ),
              ListTile(
                leading: const Icon(Icons.android),
                title: const Text('OS'),
                subtitle: Text(info.osVersion),
              ),
              ListTile(
                leading: const Icon(Icons.business),
                title: const Text('Manufacturer'),
                subtitle: Text(info.manufacturer),
              ),
              ListTile(
                leading: Icon(
                  info.isPhysicalDevice ? Icons.smartphone : Icons.computer,
                ),
                title: const Text('Device Type'),
                subtitle: Text(info.isPhysicalDevice ? 'Physical' : 'Emulator'),
              ),
            ],
          );
        },
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 884: URL Launcher และ Share

```dart
// pubspec.yaml:
// url_launcher: ^6.3.0
// share_plus: ^10.0.0

import 'package:flutter/material.dart';
import 'package:url_launcher/url_launcher.dart';
import 'package:share_plus/share_plus.dart';

class DeepLinkService {
  // เปิด URL ใน browser
  Future<void> openUrl(String url) async {
    Uri uri = Uri.parse(url);
    if (!await launchUrl(uri, mode: LaunchMode.externalApplication)) {
      throw Exception('Cannot open URL: $url');
    }
  }

  // โทรศัพท์
  Future<void> makePhoneCall(String phone) async {
    Uri uri = Uri(scheme: 'tel', path: phone);
    await launchUrl(uri);
  }

  // ส่ง Email
  Future<void> sendEmail({
    required String to,
    String? subject,
    String? body,
  }) async {
    String params = '?subject=${Uri.encodeComponent(subject ?? '')}'
        '&body=${Uri.encodeComponent(body ?? '')}';
    Uri uri = Uri.parse('mailto:$to$params');
    await launchUrl(uri);
  }

  // SMS
  Future<void> sendSms(String phone, {String? message}) async {
    Uri uri = Uri(scheme: 'sms', path: phone, queryParameters: {
      if (message != null) 'body': message,
    });
    await launchUrl(uri);
  }

  // Map
  Future<void> openMap(double lat, double lng, {String? label}) async {
    Uri uri = Platform.isIOS
        ? Uri.parse('https://maps.apple.com/?q=$lat,$lng')
        : Uri.parse('geo:$lat,$lng?q=$lat,$lng(${Uri.encodeComponent(label ?? '')})');
    await launchUrl(uri, mode: LaunchMode.externalApplication);
  }
}

class ShareService {
  // Share text
  Future<void> shareText(String text) async {
    await Share.share(text);
  }

  // Share with title
  Future<void> shareWithSubject(String text, {String? subject}) async {
    await Share.share(text, subject: subject);
  }

  // Share file
  Future<void> shareFile(String filePath, {String? text}) async {
    await Share.shareXFiles(
      [XFile(filePath)],
      text: text,
    );
  }
}

// ─── UI ───
class SocialLinks extends StatelessWidget {
  const SocialLinks({super.key});

  @override
  Widget build(BuildContext context) {
    DeepLinkService links = DeepLinkService();
    ShareService share = ShareService();

    return Column(
      children: [
        ListTile(
          leading: const Icon(Icons.language, color: Colors.blue),
          title: const Text('เปิดเว็บไซต์'),
          onTap: () => links.openUrl('https://flutter.dev'),
        ),
        ListTile(
          leading: const Icon(Icons.phone, color: Colors.green),
          title: const Text('โทร 1234'),
          onTap: () => links.makePhoneCall('1234'),
        ),
        ListTile(
          leading: const Icon(Icons.email, color: Colors.orange),
          title: const Text('ส่ง Email'),
          onTap: () => links.sendEmail(
            to: 'hello@example.com',
            subject: 'Hello from Flutter',
          ),
        ),
        ListTile(
          leading: const Icon(Icons.share, color: Colors.purple),
          title: const Text('แชร์'),
          onTap: () => share.shareText('ดาวน์โหลดแอปนี้ที่ https://example.com'),
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 885: Camera & Image Picker

```dart
// pubspec.yaml:
// image_picker: ^1.1.0
// camera: ^0.11.0

import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';
import 'dart:io';

class ImagePickerService {
  final ImagePicker _picker = ImagePicker();

  Future<File?> pickFromGallery({int? maxWidth, int? maxHeight}) async {
    XFile? file = await _picker.pickImage(
      source: ImageSource.gallery,
      maxWidth: maxWidth?.toDouble(),
      maxHeight: maxHeight?.toDouble(),
      imageQuality: 85,
    );
    return file != null ? File(file.path) : null;
  }

  Future<File?> pickFromCamera() async {
    XFile? file = await _picker.pickImage(
      source: ImageSource.camera,
      imageQuality: 85,
    );
    return file != null ? File(file.path) : null;
  }

  Future<List<File>> pickMultiple({int? limit}) async {
    List<XFile> files = await _picker.pickMultiImage(
      imageQuality: 85,
      limit: limit,
    );
    return files.map((f) => File(f.path)).toList();
  }

  Future<File?> pickVideo() async {
    XFile? file = await _picker.pickVideo(source: ImageSource.gallery);
    return file != null ? File(file.path) : null;
  }
}

// ─── Profile Photo Picker ───
class ProfilePhotoPicker extends StatefulWidget {
  final void Function(File file) onImageSelected;

  const ProfilePhotoPicker({super.key, required this.onImageSelected});

  @override
  State<ProfilePhotoPicker> createState() => _ProfilePhotoPickerState();
}

class _ProfilePhotoPickerState extends State<ProfilePhotoPicker> {
  File? _image;
  final ImagePickerService _service = ImagePickerService();

  Future<void> _showPickOptions() async {
    showModalBottomSheet(
      context: context,
      builder: (context) => Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          ListTile(
            leading: const Icon(Icons.photo_library),
            title: const Text('เลือกจากคลัง'),
            onTap: () async {
              Navigator.pop(context);
              File? file = await _service.pickFromGallery(maxWidth: 500, maxHeight: 500);
              if (file != null) {
                setState(() => _image = file);
                widget.onImageSelected(file);
              }
            },
          ),
          ListTile(
            leading: const Icon(Icons.camera_alt),
            title: const Text('ถ่ายรูป'),
            onTap: () async {
              Navigator.pop(context);
              File? file = await _service.pickFromCamera();
              if (file != null) {
                setState(() => _image = file);
                widget.onImageSelected(file);
              }
            },
          ),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _showPickOptions,
      child: Stack(
        children: [
          CircleAvatar(
            radius: 50,
            backgroundImage: _image != null ? FileImage(_image!) : null,
            child: _image == null
                ? const Icon(Icons.person, size: 50)
                : null,
          ),
          Positioned(
            bottom: 0,
            right: 0,
            child: Container(
              padding: const EdgeInsets.all(4),
              decoration: const BoxDecoration(
                color: Colors.blue,
                shape: BoxShape.circle,
              ),
              child: const Icon(Icons.camera_alt, size: 16, color: Colors.white),
            ),
          ),
        ],
      ),
    );
  }
}
```

---

**← [Part 24 - Custom Painting](part-24-custom-painting.md)**

**ต่อไป: [Part 26 - Clean Architecture →](part-26-clean-architecture.md)**
