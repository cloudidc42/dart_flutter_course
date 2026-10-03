# Part 90: Release Checklist & App Store Submission
## ขั้นตอนที่ 3481-3520

---

## 🎯 เป้าหมายของ Part นี้

- ทำ App Store (iOS) submission checklist ครบถ้วน
- ทำ Google Play (Android) submission checklist ครบถ้วน
- ตั้งค่า App Signing (keystore และ certificates)
- จัดการ Privacy policy และ permissions อย่างถูกต้อง
- วัด Performance benchmarks ก่อน release
- เขียน automation scripts สำหรับ release process

---

## ขั้นตอนที่ 3481: App Signing — Android Keystore

```bash
# สร้าง keystore สำหรับ signing APK/AAB
keytool -genkey -v \
  -keystore ~/my-release-key.jks \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -alias my-key-alias \
  -dname "CN=My Name, OU=My Unit, O=My Company, L=Bangkok, S=Bangkok, C=TH"

# ตรวจสอบ keystore
keytool -list -v -keystore ~/my-release-key.jks
```

```groovy
// android/app/build.gradle
android {
    // ...

    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

```properties
# android/key.properties  (DO NOT COMMIT TO GIT)
storePassword=your_keystore_password
keyPassword=your_key_password
keyAlias=my-key-alias
storeFile=/path/to/my-release-key.jks
```

```kotlin
// android/app/build.gradle.kts (Kotlin DSL version)
import java.util.Properties
import java.io.FileInputStream

val keystorePropertiesFile = rootProject.file("key.properties")
val keystoreProperties = Properties()
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(FileInputStream(keystorePropertiesFile))
}

android {
    signingConfigs {
        create("release") {
            keyAlias = keystoreProperties["keyAlias"] as String? ?: ""
            keyPassword = keystoreProperties["keyPassword"] as String? ?: ""
            storeFile = keystoreProperties["storeFile"]?.let { file(it) }
            storePassword = keystoreProperties["storePassword"] as String? ?: ""
        }
    }

    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

## ขั้นตอนที่ 3482: Android ProGuard Rules

```proguard
# android/app/proguard-rules.pro

# Flutter rules
-keep class io.flutter.app.** { *; }
-keep class io.flutter.plugin.** { *; }
-keep class io.flutter.util.** { *; }
-keep class io.flutter.view.** { *; }
-keep class io.flutter.** { *; }
-keep class io.flutter.plugins.** { *; }

# Stripe
-keep class com.stripe.android.** { *; }
-dontwarn com.stripe.android.**

# Firebase
-keep class com.google.firebase.** { *; }
-dontwarn com.google.firebase.**

# Gson
-keepattributes Signature
-keepattributes *Annotation*
-dontwarn sun.misc.**
-keep class com.google.gson.** { *; }
-keep class * implements com.google.gson.TypeAdapterFactory
-keep class * implements com.google.gson.JsonSerializer
-keep class * implements com.google.gson.JsonDeserializer

# Model classes (adjust to your package)
-keep class com.example.myapp.models.** { *; }

# Prevent obfuscation of entry points
-keepclassmembers class * {
    @android.webkit.JavascriptInterface <methods>;
}
```

## ขั้นตอนที่ 3483: iOS App Signing & Certificates

```ruby
# Gemfile (for Fastlane)
source "https://rubygems.org"
gem "fastlane"
gem "cocoapods"
```

```ruby
# ios/Fastfile
default_platform(:ios)

platform :ios do
  desc "Push a new beta build to TestFlight"
  lane :beta do
    # Increment build number
    increment_build_number(
      build_number: latest_testflight_build_number + 1,
      xcodeproj: "Runner.xcodeproj"
    )

    # Build
    build_app(
      scheme: "Runner",
      workspace: "Runner.xcworkspace",
      export_method: "app-store",
      export_options: {
        provisioningProfiles: {
          "com.example.myapp" => "MyApp AppStore"
        }
      }
    )

    # Upload to TestFlight
    upload_to_testflight(
      skip_waiting_for_build_processing: true,
      changelog: "Bug fixes and performance improvements"
    )

    # Notify team
    slack(
      message: "New TestFlight build uploaded!",
      channel: "#releases"
    )
  end

  desc "Submit to App Store"
  lane :release do
    increment_build_number(
      build_number: latest_testflight_build_number + 1,
      xcodeproj: "Runner.xcodeproj"
    )

    build_app(
      scheme: "Runner",
      workspace: "Runner.xcworkspace",
      export_method: "app-store"
    )

    upload_to_app_store(
      force: true,
      reject_if_possible: true,
      skip_screenshots: false,
      skip_metadata: false,
      submit_for_review: true,
      automatic_release: false,
      submission_information: {
        add_id_info_uses_idfa: false,
        export_compliance_uses_encryption: false,
        content_rights_contains_third_party_content: false
      }
    )
  end
end
```

## ขั้นตอนที่ 3484: Android Fastlane Setup

```ruby
# android/Fastfile
default_platform(:android)

platform :android do
  desc "Build and upload to Google Play internal testing"
  lane :internal do
    # Increment version code
    android_set_version_code(
      version_code: google_play_track_version_codes(
        track: "internal"
      ).max + 1
    )

    # Build AAB
    gradle(
      task: "bundle",
      build_type: "Release",
      print_command: false,
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"],
      }
    )

    # Upload to internal track
    upload_to_play_store(
      track: "internal",
      aab: "app/build/outputs/bundle/release/app-release.aab",
      skip_upload_metadata: true,
      skip_upload_images: true,
      skip_upload_screenshots: true
    )
  end

  desc "Promote internal to production"
  lane :production do
    upload_to_play_store(
      track: "internal",
      track_promote_to: "production",
      rollout: "0.1"  # 10% rollout
    )
  end
end
```

## ขั้นตอนที่ 3485: Flutter Build Configuration

```dart
// lib/config/app_config.dart
enum BuildEnvironment { development, staging, production }

class AppConfig {
  final BuildEnvironment environment;
  final String apiBaseUrl;
  final String stripePublishableKey;
  final bool enableCrashReporting;
  final bool enableAnalytics;
  final bool enableLogging;
  final String appName;

  const AppConfig._({
    required this.environment,
    required this.apiBaseUrl,
    required this.stripePublishableKey,
    required this.enableCrashReporting,
    required this.enableAnalytics,
    required this.enableLogging,
    required this.appName,
  });

  static const development = AppConfig._(
    environment: BuildEnvironment.development,
    apiBaseUrl: 'https://dev-api.example.com',
    stripePublishableKey: 'pk_test_xxx',
    enableCrashReporting: false,
    enableAnalytics: false,
    enableLogging: true,
    appName: 'MyApp DEV',
  );

  static const staging = AppConfig._(
    environment: BuildEnvironment.staging,
    apiBaseUrl: 'https://staging-api.example.com',
    stripePublishableKey: 'pk_test_xxx',
    enableCrashReporting: true,
    enableAnalytics: true,
    enableLogging: true,
    appName: 'MyApp STAGING',
  );

  static const production = AppConfig._(
    environment: BuildEnvironment.production,
    apiBaseUrl: 'https://api.example.com',
    stripePublishableKey: 'pk_live_xxx',
    enableCrashReporting: true,
    enableAnalytics: true,
    enableLogging: false,
    appName: 'MyApp',
  );

  bool get isDevelopment => environment == BuildEnvironment.development;
  bool get isProduction => environment == BuildEnvironment.production;
}

// main_development.dart
void main() => runApp(AppWithConfig(config: AppConfig.development));

// main_staging.dart
void main() => runApp(AppWithConfig(config: AppConfig.staging));

// main_production.dart
void main() => runApp(AppWithConfig(config: AppConfig.production));

class AppWithConfig extends StatelessWidget {
  final AppConfig config;
  const AppWithConfig({super.key, required this.config});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: config.appName,
      builder: (context, child) {
        if (config.isDevelopment) {
          return Banner(
            message: 'DEV',
            location: BannerLocation.topStart,
            child: child!,
          );
        }
        return child!;
      },
      home: const Placeholder(),
    );
  }
}
```

```yaml
# flutter run / build with specific entry point
# Development
# flutter run -t lib/main_development.dart

# Staging
# flutter run -t lib/main_staging.dart --dart-define=ENV=staging

# Production
# flutter build appbundle -t lib/main_production.dart --release

# scripts/build_android.sh
#!/bin/bash
set -e

ENVIRONMENT=${1:-production}
VERSION=$(grep 'version:' pubspec.yaml | head -1 | awk '{print $2}')
echo "Building Android $ENVIRONMENT - v$VERSION"

flutter clean
flutter pub get

if [ "$ENVIRONMENT" = "production" ]; then
  flutter build appbundle \
    -t lib/main_production.dart \
    --release \
    --dart-define=ENV=production
else
  flutter build apk \
    -t lib/main_development.dart \
    --debug \
    --dart-define=ENV=development
fi

echo "Build complete!"
```

## ขั้นตอนที่ 3486: Privacy Policy & Permissions

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Network -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

    <!-- Camera (only if used) -->
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-feature android:name="android.hardware.camera" android:required="false" />

    <!-- Location (only if used) -->
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

    <!-- Storage (Android < 13) -->
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
        android:maxSdkVersion="32" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
        android:maxSdkVersion="29" />

    <!-- Media (Android 13+) -->
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
    <uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />

    <!-- Notifications -->
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />

    <!-- Biometric -->
    <uses-permission android:name="android.permission.USE_BIOMETRIC" />
    <uses-permission android:name="android.permission.USE_FINGERPRINT" />

    <application
        android:name="${applicationName}"
        android:label="@string/app_name"
        android:icon="@mipmap/ic_launcher"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:allowBackup="false"
        android:networkSecurityConfig="@xml/network_security_config"
        android:usesCleartextTraffic="false">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTop"
            android:theme="@style/LaunchTheme"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|smallestScreenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
            android:hardwareAccelerated="true"
            android:windowSoftInputMode="adjustResize">

            <intent-filter android:autoVerify="true">
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>

            <!-- Deep links -->
            <intent-filter android:autoVerify="true">
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />
                <data android:scheme="https"
                    android:host="example.com"
                    android:pathPrefix="/app" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

```xml
<!-- android/app/src/main/res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">example.com</domain>
        <!-- Certificate pinning -->
        <pin-set expiration="2027-01-01">
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <!-- Backup pin -->
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
        </pin-set>
    </domain-config>
    <!-- Allow debug traffic in development only -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

```xml
<!-- ios/Runner/Info.plist — Privacy usage descriptions (required by Apple) -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- Required: Camera -->
    <key>NSCameraUsageDescription</key>
    <string>We need camera access to let you take profile photos and scan documents.</string>

    <!-- Required: Photo Library -->
    <key>NSPhotoLibraryUsageDescription</key>
    <string>We need photo library access to let you upload images.</string>
    <key>NSPhotoLibraryAddUsageDescription</key>
    <string>We need permission to save photos to your library.</string>

    <!-- Required: Location -->
    <key>NSLocationWhenInUseUsageDescription</key>
    <string>We use your location to show nearby stores and provide delivery estimates.</string>
    <key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
    <string>We use your location for real-time delivery tracking.</string>

    <!-- Required: Microphone -->
    <key>NSMicrophoneUsageDescription</key>
    <string>We need microphone access for voice messages.</string>

    <!-- Required: Contacts -->
    <key>NSContactsUsageDescription</key>
    <string>We use contacts to make it easy to invite friends.</string>

    <!-- Required: Face ID -->
    <key>NSFaceIDUsageDescription</key>
    <string>Use Face ID to securely log in to your account.</string>

    <!-- App Transport Security -->
    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSAllowsArbitraryLoads</key>
        <false/>
        <key>NSExceptionDomains</key>
        <dict>
            <key>example.com</key>
            <dict>
                <key>NSIncludesSubdomains</key>
                <true/>
                <key>NSThirdPartyExceptionMinimumTLSVersion</key>
                <string>TLSv1.2</string>
            </dict>
        </dict>
    </dict>
</dict>
</plist>
```

## ขั้นตอนที่ 3487: Runtime Permission Handler

```dart
// lib/permissions/permission_handler.dart
import 'package:flutter/material.dart';
import 'package:permission_handler/permission_handler.dart';

enum AppPermission {
  camera,
  photos,
  location,
  notifications,
  microphone,
  contacts,
}

class PermissionResult {
  final AppPermission permission;
  final bool isGranted;
  final bool isPermanentlyDenied;

  const PermissionResult({
    required this.permission,
    required this.isGranted,
    this.isPermanentlyDenied = false,
  });
}

class PermissionManager {
  static Permission _toPermission(AppPermission appPermission) {
    switch (appPermission) {
      case AppPermission.camera:
        return Permission.camera;
      case AppPermission.photos:
        return Permission.photos;
      case AppPermission.location:
        return Permission.locationWhenInUse;
      case AppPermission.notifications:
        return Permission.notification;
      case AppPermission.microphone:
        return Permission.microphone;
      case AppPermission.contacts:
        return Permission.contacts;
    }
  }

  static Future<PermissionResult> request(AppPermission permission) async {
    final permObj = _toPermission(permission);
    final status = await permObj.request();

    return PermissionResult(
      permission: permission,
      isGranted: status.isGranted,
      isPermanentlyDenied: status.isPermanentlyDenied,
    );
  }

  static Future<PermissionResult> check(AppPermission permission) async {
    final permObj = _toPermission(permission);
    final status = await permObj.status;

    return PermissionResult(
      permission: permission,
      isGranted: status.isGranted,
      isPermanentlyDenied: status.isPermanentlyDenied,
    );
  }

  static Future<void> requestWithRationale({
    required BuildContext context,
    required AppPermission permission,
    required String title,
    required String rationale,
    required VoidCallback onGranted,
    VoidCallback? onDenied,
  }) async {
    final current = await check(permission);

    if (current.isGranted) {
      onGranted();
      return;
    }

    if (current.isPermanentlyDenied) {
      if (context.mounted) {
        await _showSettingsDialog(context, title, rationale);
      }
      return;
    }

    // Show rationale before requesting
    if (context.mounted) {
      final shouldRequest = await showDialog<bool>(
            context: context,
            builder: (ctx) => AlertDialog(
              title: Text(title),
              content: Text(rationale),
              actions: [
                TextButton(
                  onPressed: () => Navigator.of(ctx).pop(false),
                  child: const Text('Not Now'),
                ),
                ElevatedButton(
                  onPressed: () => Navigator.of(ctx).pop(true),
                  child: const Text('Allow'),
                ),
              ],
            ),
          ) ??
          false;

      if (!shouldRequest) {
        onDenied?.call();
        return;
      }
    }

    final result = await request(permission);
    if (result.isGranted) {
      onGranted();
    } else {
      onDenied?.call();
    }
  }

  static Future<void> _showSettingsDialog(
    BuildContext context,
    String title,
    String message,
  ) async {
    await showDialog<void>(
      context: context,
      builder: (ctx) => AlertDialog(
        title: Text(title),
        content: Text(
          '$message\n\nPlease enable this permission in Settings.',
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.of(ctx).pop(),
            child: const Text('Cancel'),
          ),
          ElevatedButton(
            onPressed: () {
              Navigator.of(ctx).pop();
              openAppSettings();
            },
            child: const Text('Open Settings'),
          ),
        ],
      ),
    );
  }
}
```

## ขั้นตอนที่ 3488: Performance Benchmarks

```dart
// lib/performance/performance_monitor.dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:flutter/scheduler.dart';

class FrameMetrics {
  final Duration totalSpan;
  final Duration buildDuration;
  final Duration rasterDuration;
  final int frameNumber;

  const FrameMetrics({
    required this.totalSpan,
    required this.buildDuration,
    required this.rasterDuration,
    required this.frameNumber,
  });

  bool get isJanky => totalSpan.inMilliseconds > 16; // Below 60fps
  double get fps => 1000 / totalSpan.inMilliseconds.clamp(1, 1000).toDouble();
}

class PerformanceMonitor {
  final List<FrameMetrics> _metrics = [];
  int _frameCount = 0;
  bool _isMonitoring = false;

  void startMonitoring() {
    if (_isMonitoring) return;
    _isMonitoring = true;

    SchedulerBinding.instance.addTimingsCallback(_onTimings);
  }

  void stopMonitoring() {
    if (!_isMonitoring) return;
    _isMonitoring = false;
    SchedulerBinding.instance.removeTimingsCallback(_onTimings);
  }

  void _onTimings(List<FrameTiming> timings) {
    for (final timing in timings) {
      _frameCount++;
      _metrics.add(FrameMetrics(
        totalSpan: timing.totalSpan,
        buildDuration: timing.buildDuration,
        rasterDuration: timing.rasterDuration,
        frameNumber: _frameCount,
      ));

      // Keep only last 300 frames (5 seconds at 60fps)
      if (_metrics.length > 300) {
        _metrics.removeAt(0);
      }
    }
  }

  PerformanceReport generateReport() {
    if (_metrics.isEmpty) {
      return const PerformanceReport(
        averageFps: 0,
        minFps: 0,
        maxFps: 0,
        jankyFrameCount: 0,
        jankyFramePercentage: 0,
        averageBuildTime: Duration.zero,
        averageRasterTime: Duration.zero,
        totalFrames: 0,
      );
    }

    final fpsList = _metrics.map((m) => m.fps).toList();
    final avgFps = fpsList.reduce((a, b) => a + b) / fpsList.length;
    final minFps = fpsList.reduce((a, b) => a < b ? a : b);
    final maxFps = fpsList.reduce((a, b) => a > b ? a : b);

    final jankyFrames = _metrics.where((m) => m.isJanky).length;

    final avgBuild = Duration(
      microseconds: _metrics
              .map((m) => m.buildDuration.inMicroseconds)
              .reduce((a, b) => a + b) ~/
          _metrics.length,
    );

    final avgRaster = Duration(
      microseconds: _metrics
              .map((m) => m.rasterDuration.inMicroseconds)
              .reduce((a, b) => a + b) ~/
          _metrics.length,
    );

    return PerformanceReport(
      averageFps: avgFps,
      minFps: minFps,
      maxFps: maxFps,
      jankyFrameCount: jankyFrames,
      jankyFramePercentage: jankyFrames / _metrics.length * 100,
      averageBuildTime: avgBuild,
      averageRasterTime: avgRaster,
      totalFrames: _metrics.length,
    );
  }
}

class PerformanceReport {
  final double averageFps;
  final double minFps;
  final double maxFps;
  final int jankyFrameCount;
  final double jankyFramePercentage;
  final Duration averageBuildTime;
  final Duration averageRasterTime;
  final int totalFrames;

  const PerformanceReport({
    required this.averageFps,
    required this.minFps,
    required this.maxFps,
    required this.jankyFrameCount,
    required this.jankyFramePercentage,
    required this.averageBuildTime,
    required this.averageRasterTime,
    required this.totalFrames,
  });

  bool get passesProductionThreshold =>
      averageFps >= 55 && jankyFramePercentage < 5;

  @override
  String toString() => '''
Performance Report:
  Average FPS: ${averageFps.toStringAsFixed(1)}
  Min FPS: ${minFps.toStringAsFixed(1)}
  Max FPS: ${maxFps.toStringAsFixed(1)}
  Janky frames: $jankyFrameCount (${jankyFramePercentage.toStringAsFixed(1)}%)
  Avg build time: ${averageBuildTime.inMicroseconds}μs
  Avg raster time: ${averageRasterTime.inMicroseconds}μs
  Total frames: $totalFrames
  Passes production threshold: $passesProductionThreshold
''';
}

// Performance overlay widget for debug builds
class PerformanceOverlay extends StatefulWidget {
  final Widget child;
  const PerformanceOverlay({super.key, required this.child});

  @override
  State<PerformanceOverlay> createState() => _PerformanceOverlayState();
}

class _PerformanceOverlayState extends State<PerformanceOverlay> {
  final _monitor = PerformanceMonitor();
  Timer? _updateTimer;
  PerformanceReport _report = const PerformanceReport(
    averageFps: 0, minFps: 0, maxFps: 0,
    jankyFrameCount: 0, jankyFramePercentage: 0,
    averageBuildTime: Duration.zero, averageRasterTime: Duration.zero,
    totalFrames: 0,
  );

  @override
  void initState() {
    super.initState();
    _monitor.startMonitoring();
    _updateTimer = Timer.periodic(
      const Duration(seconds: 1),
      (_) => setState(() => _report = _monitor.generateReport()),
    );
  }

  @override
  void dispose() {
    _monitor.stopMonitoring();
    _updateTimer?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        widget.child,
        Positioned(
          top: 48,
          right: 8,
          child: Container(
            padding: const EdgeInsets.all(8),
            decoration: BoxDecoration(
              color: _report.passesProductionThreshold
                  ? Colors.green.withOpacity(0.8)
                  : Colors.red.withOpacity(0.8),
              borderRadius: BorderRadius.circular(8),
            ),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.end,
              children: [
                Text(
                  '${_report.averageFps.toStringAsFixed(0)} FPS',
                  style: const TextStyle(
                    color: Colors.white,
                    fontWeight: FontWeight.bold,
                    fontSize: 16,
                  ),
                ),
                Text(
                  'Janky: ${_report.jankyFramePercentage.toStringAsFixed(1)}%',
                  style: const TextStyle(color: Colors.white, fontSize: 12),
                ),
              ],
            ),
          ),
        ),
      ],
    );
  }
}
```

## ขั้นตอนที่ 3489: Release Automation Script

```dart
// scripts/check_release_readiness.dart
// Run with: dart scripts/check_release_readiness.dart

import 'dart:io';

class ReleaseCheck {
  final String name;
  final String description;
  final Future<bool> Function() check;

  const ReleaseCheck({
    required this.name,
    required this.description,
    required this.check,
  });
}

class ReleaseChecker {
  final List<ReleaseCheck> _checks = [];
  int _passed = 0;
  int _failed = 0;

  void addCheck(ReleaseCheck check) => _checks.add(check);

  Future<bool> runAll() async {
    print('=== Release Readiness Check ===\n');

    for (final check in _checks) {
      stdout.write('  Checking: ${check.name}... ');
      try {
        final result = await check.check();
        if (result) {
          print('✓ PASS');
          _passed++;
        } else {
          print('✗ FAIL — ${check.description}');
          _failed++;
        }
      } catch (e) {
        print('✗ ERROR: $e');
        _failed++;
      }
    }

    print('\n=== Results ===');
    print('Passed: $_passed');
    print('Failed: $_failed');
    print('Total: ${_checks.length}');

    return _failed == 0;
  }
}

Future<void> main() async {
  final checker = ReleaseChecker();

  // 1. Version check
  checker.addCheck(ReleaseCheck(
    name: 'Version in pubspec.yaml',
    description: 'pubspec.yaml must have a valid version like 1.0.0+1',
    check: () async {
      final content = File('pubspec.yaml').readAsStringSync();
      final match = RegExp(r'version:\s+\d+\.\d+\.\d+\+\d+').hasMatch(content);
      return match;
    },
  ));

  // 2. No debug prints
  checker.addCheck(ReleaseCheck(
    name: 'No debug print statements in lib/',
    description: 'Remove or wrap print() and debugPrint() calls',
    check: () async {
      final result = Process.runSync('grep', [
        '-rn',
        '--include=*.dart',
        r'^\s*print(',
        'lib/',
      ]);
      return (result.stdout as String).trim().isEmpty;
    },
  ));

  // 3. No TODO in production code
  checker.addCheck(ReleaseCheck(
    name: 'No TODO/FIXME/HACK comments',
    description: 'Resolve all TODO, FIXME, HACK comments before release',
    check: () async {
      final result = Process.runSync('grep', [
        '-rn',
        '--include=*.dart',
        r'TODO\|FIXME\|HACK',
        'lib/',
      ]);
      return (result.stdout as String).trim().isEmpty;
    },
  ));

  // 4. Tests pass
  checker.addCheck(ReleaseCheck(
    name: 'All tests pass',
    description: 'Run flutter test and ensure all tests pass',
    check: () async {
      final result = Process.runSync('flutter', ['test', '--reporter', 'compact']);
      return result.exitCode == 0;
    },
  ));

  // 5. pubspec.yaml dependencies
  checker.addCheck(ReleaseCheck(
    name: 'No path: dependencies in pubspec.yaml',
    description: 'Replace path: dependencies with published packages',
    check: () async {
      final content = File('pubspec.yaml').readAsStringSync();
      return !content.contains('path:');
    },
  ));

  // 6. Android minSdkVersion
  checker.addCheck(ReleaseCheck(
    name: 'Android minSdkVersion >= 21',
    description: 'Minimum SDK version should be at least 21',
    check: () async {
      final content =
          File('android/app/build.gradle').readAsStringSync();
      final match = RegExp(r'minSdkVersion\s+(\d+)').firstMatch(content);
      if (match == null) return false;
      return int.parse(match.group(1)!) >= 21;
    },
  ));

  // 7. iOS deployment target
  checker.addCheck(ReleaseCheck(
    name: 'iOS deployment target >= 12.0',
    description: 'iOS deployment target should be at least 12.0',
    check: () async {
      final podfile = File('ios/Podfile');
      if (!podfile.existsSync()) return false;
      final content = podfile.readAsStringSync();
      final match =
          RegExp(r"platform :ios, '(\d+\.\d+)'").firstMatch(content);
      if (match == null) return false;
      return double.parse(match.group(1)!) >= 12.0;
    },
  ));

  // 8. App icon exists
  checker.addCheck(ReleaseCheck(
    name: 'App icon configured',
    description: 'Ensure app icon assets exist',
    check: () async {
      return Directory('android/app/src/main/res/mipmap-xxhdpi').existsSync();
    },
  ));

  // 9. Splash screen configured
  checker.addCheck(ReleaseCheck(
    name: 'Splash screen configured',
    description: 'Ensure splash screen is properly set up',
    check: () async {
      final pubspec = File('pubspec.yaml').readAsStringSync();
      return pubspec.contains('flutter_native_splash') ||
          pubspec.contains('flutter_launcher_icons');
    },
  ));

  // 10. No hardcoded secrets
  checker.addCheck(ReleaseCheck(
    name: 'No hardcoded API keys',
    description: 'Check for hardcoded secrets in source code',
    check: () async {
      final result = Process.runSync('grep', [
        '-rn',
        '--include=*.dart',
        r'sk_live_\|pk_live_\|AIzaSy\|AAAA[A-Za-z0-9_-]',
        'lib/',
      ]);
      return (result.stdout as String).trim().isEmpty;
    },
  ));

  final allPassed = await checker.runAll();

  if (allPassed) {
    print('\n✅ App is ready for release!');
  } else {
    print('\n❌ Please fix the issues above before releasing.');
    exit(1);
  }
}
```

## ขั้นตอนที่ 3490: CI/CD Pipeline for Release

```yaml
# .github/workflows/release.yml
name: Release Pipeline

on:
  push:
    tags:
      - 'v*.*.*'

env:
  FLUTTER_VERSION: '3.19.0'

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: stable
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Run analyzer
        run: flutter analyze --no-fatal-infos

      - name: Run tests with coverage
        run: flutter test --coverage --reporter github

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: coverage/lcov.info

  build-android:
    name: Build Android
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Set up Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: stable
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Decode keystore
        env:
          KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
        run: |
          echo $KEYSTORE_BASE64 | base64 --decode > android/app/release.keystore

      - name: Create key.properties
        env:
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
          STORE_PASSWORD: ${{ secrets.STORE_PASSWORD }}
        run: |
          cat > android/key.properties << EOF
          storePassword=$STORE_PASSWORD
          keyPassword=$KEY_PASSWORD
          keyAlias=$KEY_ALIAS
          storeFile=release.keystore
          EOF

      - name: Build AAB
        run: |
          flutter build appbundle \
            -t lib/main_production.dart \
            --release \
            --dart-define=ENV=production

      - name: Upload AAB artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-release.aab
          path: build/app/outputs/bundle/release/app-release.aab

  build-ios:
    name: Build iOS
    runs-on: macos-latest
    needs: test
    steps:
      - uses: actions/checkout@v4

      - name: Set up Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: stable
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Install CocoaPods
        run: |
          cd ios
          pod install

      - name: Import certificate
        env:
          CERTIFICATE_BASE64: ${{ secrets.CERTIFICATE_BASE64 }}
          CERTIFICATE_PASSWORD: ${{ secrets.CERTIFICATE_PASSWORD }}
          PROVISION_PROFILE_BASE64: ${{ secrets.PROVISION_PROFILE_BASE64 }}
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
        run: |
          # Create keychain
          security create-keychain -p "$KEYCHAIN_PASSWORD" build.keychain
          security default-keychain -s build.keychain
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" build.keychain

          # Import certificate
          echo $CERTIFICATE_BASE64 | base64 --decode > certificate.p12
          security import certificate.p12 \
            -k build.keychain \
            -P "$CERTIFICATE_PASSWORD" \
            -T /usr/bin/codesign

          # Import provisioning profile
          echo $PROVISION_PROFILE_BASE64 | base64 --decode > profile.mobileprovision
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          cp profile.mobileprovision ~/Library/MobileDevice/Provisioning\ Profiles/

      - name: Build IPA
        run: |
          flutter build ipa \
            -t lib/main_production.dart \
            --release \
            --dart-define=ENV=production \
            --export-options-plist=ios/ExportOptions.plist

      - name: Upload IPA artifact
        uses: actions/upload-artifact@v4
        with:
          name: Runner.ipa
          path: build/ios/ipa/Runner.ipa

  deploy-android:
    name: Deploy to Google Play
    runs-on: ubuntu-latest
    needs: build-android
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Download AAB
        uses: actions/download-artifact@v4
        with:
          name: app-release.aab
          path: build/

      - name: Upload to Play Store
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_STORE_SERVICE_ACCOUNT }}
          packageName: com.example.myapp
          releaseFiles: build/app-release.aab
          track: production
          rollout: 0.1
          status: inProgress

  deploy-ios:
    name: Deploy to App Store
    runs-on: macos-latest
    needs: build-ios
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Download IPA
        uses: actions/download-artifact@v4
        with:
          name: Runner.ipa
          path: build/

      - name: Upload to App Store Connect
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.ASC_API_KEY_ID }}
          APP_STORE_CONNECT_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY: ${{ secrets.ASC_API_KEY }}
        run: |
          xcrun altool --upload-app \
            --type ios \
            --file build/Runner.ipa \
            --apiKey $APP_STORE_CONNECT_API_KEY_ID \
            --apiIssuer $APP_STORE_CONNECT_ISSUER_ID
```

## ขั้นตอนที่ 3491: Final Release Checklist Widget

```dart
// lib/debug/release_checklist_screen.dart
// Use this screen internally to review before submission

import 'package:flutter/material.dart';

class ChecklistItem {
  final String category;
  final String title;
  final String description;
  bool isChecked;

  ChecklistItem({
    required this.category,
    required this.title,
    required this.description,
    this.isChecked = false,
  });
}

class ReleaseChecklistScreen extends StatefulWidget {
  const ReleaseChecklistScreen({super.key});

  @override
  State<ReleaseChecklistScreen> createState() => _ReleaseChecklistScreenState();
}

class _ReleaseChecklistScreenState extends State<ReleaseChecklistScreen> {
  late List<ChecklistItem> _items;

  @override
  void initState() {
    super.initState();
    _items = [
      // App Store
      ChecklistItem(category: 'App Store', title: 'App screenshots (6.5", 5.5", iPad)', description: 'All required screenshot sizes uploaded'),
      ChecklistItem(category: 'App Store', title: 'App preview video', description: 'Optional but recommended 30-second preview'),
      ChecklistItem(category: 'App Store', title: 'App description & keywords', description: 'Optimized for ASO'),
      ChecklistItem(category: 'App Store', title: 'Privacy policy URL', description: 'Must be publicly accessible'),
      ChecklistItem(category: 'App Store', title: 'Age rating questionnaire', description: 'Completed in App Store Connect'),
      ChecklistItem(category: 'App Store', title: 'Export compliance', description: 'Encryption declaration completed'),
      ChecklistItem(category: 'App Store', title: 'Review contact info', description: 'Valid email and phone for review team'),
      ChecklistItem(category: 'App Store', title: 'Demo account credentials', description: 'If app requires login'),

      // Google Play
      ChecklistItem(category: 'Google Play', title: 'Feature graphic (1024×500)', description: 'Required for Play Store listing'),
      ChecklistItem(category: 'Google Play', title: 'Phone screenshots (min 2)', description: 'At least 2 screenshots required'),
      ChecklistItem(category: 'Google Play', title: 'App content rating', description: 'IARC questionnaire completed'),
      ChecklistItem(category: 'Google Play', title: 'Data safety form', description: 'Declare what data is collected'),
      ChecklistItem(category: 'Google Play', title: 'Target audience', description: 'Age group and content settings'),
      ChecklistItem(category: 'Google Play', title: 'App category', description: 'Correct category selected'),

      // Technical
      ChecklistItem(category: 'Technical', title: 'All tests passing', description: 'flutter test shows 0 failures'),
      ChecklistItem(category: 'Technical', title: 'No analyzer warnings', description: 'flutter analyze shows no issues'),
      ChecklistItem(category: 'Technical', title: 'Performance tested', description: 'Average FPS >= 55, janky frames < 5%'),
      ChecklistItem(category: 'Technical', title: 'Memory leaks checked', description: 'No growing memory in DevTools'),
      ChecklistItem(category: 'Technical', title: 'Network timeout handling', description: 'Graceful handling of offline state'),
      ChecklistItem(category: 'Technical', title: 'Deep links tested', description: 'All deep link routes work correctly'),
      ChecklistItem(category: 'Technical', title: 'Push notifications tested', description: 'Foreground, background, killed state'),
      ChecklistItem(category: 'Technical', title: 'Dark mode support', description: 'UI looks correct in both themes'),
      ChecklistItem(category: 'Technical', title: 'Accessibility check', description: 'TalkBack/VoiceOver navigation works'),
      ChecklistItem(category: 'Technical', title: 'Tablet layout tested', description: 'UI adapts properly to larger screens'),

      // Security
      ChecklistItem(category: 'Security', title: 'No hardcoded secrets', description: 'No API keys in source code'),
      ChecklistItem(category: 'Security', title: 'Certificate pinning', description: 'Pins are valid and not expiring soon'),
      ChecklistItem(category: 'Security', title: 'ProGuard/R8 configured', description: 'Android code obfuscation enabled'),
      ChecklistItem(category: 'Security', title: 'Keystore backup exists', description: 'Keystore stored securely, backup made'),
      ChecklistItem(category: 'Security', title: 'Sensitive screens secured', description: 'Screenshots disabled on payment screens'),
    ];
  }

  Map<String, List<ChecklistItem>> get _groupedItems {
    final groups = <String, List<ChecklistItem>>{};
    for (final item in _items) {
      groups.putIfAbsent(item.category, () => []).add(item);
    }
    return groups;
  }

  int get _checkedCount => _items.where((i) => i.isChecked).length;
  double get _progress => _items.isEmpty ? 0 : _checkedCount / _items.length;
  bool get _isReadyForRelease => _checkedCount == _items.length;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Release Checklist'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: () => setState(() {
              for (final item in _items) item.isChecked = false;
            }),
            tooltip: 'Reset all',
          ),
        ],
      ),
      body: Column(
        children: [
          // Progress header
          Container(
            padding: const EdgeInsets.all(16),
            color: _isReadyForRelease ? Colors.green.shade50 : Colors.grey.shade100,
            child: Column(
              children: [
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    Text(
                      '$_checkedCount/${_items.length} completed',
                      style: const TextStyle(fontWeight: FontWeight.bold),
                    ),
                    if (_isReadyForRelease)
                      const Chip(
                        label: Text('Ready to Release! 🚀'),
                        backgroundColor: Colors.green,
                        labelStyle: TextStyle(color: Colors.white),
                      ),
                  ],
                ),
                const SizedBox(height: 8),
                LinearProgressIndicator(
                  value: _progress,
                  backgroundColor: Colors.grey.shade300,
                  color: _isReadyForRelease ? Colors.green : Colors.blue,
                  minHeight: 8,
                  borderRadius: BorderRadius.circular(4),
                ),
              ],
            ),
          ),

          // Checklist items
          Expanded(
            child: ListView(
              children: _groupedItems.entries.map((entry) {
                return _CategorySection(
                  category: entry.key,
                  items: entry.value,
                  onChanged: (item, checked) {
                    setState(() => item.isChecked = checked);
                  },
                );
              }).toList(),
            ),
          ),
        ],
      ),
    );
  }
}

class _CategorySection extends StatelessWidget {
  final String category;
  final List<ChecklistItem> items;
  final Function(ChecklistItem, bool) onChanged;

  const _CategorySection({
    required this.category,
    required this.items,
    required this.onChanged,
  });

  @override
  Widget build(BuildContext context) {
    final allChecked = items.every((i) => i.isChecked);

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Container(
          padding: const EdgeInsets.fromLTRB(16, 16, 16, 8),
          child: Row(
            children: [
              Icon(
                allChecked ? Icons.check_circle : Icons.circle_outlined,
                color: allChecked ? Colors.green : Colors.grey,
                size: 20,
              ),
              const SizedBox(width: 8),
              Text(
                category,
                style: TextStyle(
                  fontWeight: FontWeight.bold,
                  fontSize: 16,
                  color: allChecked ? Colors.green : Colors.black87,
                ),
              ),
              const Spacer(),
              Text(
                '${items.where((i) => i.isChecked).length}/${items.length}',
                style: const TextStyle(color: Colors.grey, fontSize: 12),
              ),
            ],
          ),
        ),
        ...items.map(
          (item) => CheckboxListTile(
            title: Text(
              item.title,
              style: TextStyle(
                decoration: item.isChecked ? TextDecoration.lineThrough : null,
                color: item.isChecked ? Colors.grey : null,
              ),
            ),
            subtitle: Text(
              item.description,
              style: const TextStyle(fontSize: 12),
            ),
            value: item.isChecked,
            onChanged: (v) => onChanged(item, v ?? false),
            activeColor: Colors.green,
          ),
        ),
        const Divider(height: 1),
      ],
    );
  }
}
```

---

**← [Part 89](part-89-payment-integration.md)**
**ต่อไป: [Part 91 →](part-91-advanced-topics.md)**
