# Part 27: CI/CD and Deployment
## ขั้นตอนที่ 961-1000

---

## 🎯 เป้าหมายของ Part นี้

- GitHub Actions สำหรับ Flutter
- Automated testing
- Build และ deploy ไปยัง App Store / Play Store
- Fastlane integration
- Versioning

---

## ขั้นตอนที่ 961: GitHub Actions Workflow

```yaml
# .github/workflows/flutter.yml
name: Flutter CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # ─── Test ───
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.0'
          channel: 'stable'
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Generate code
        run: flutter pub run build_runner build --delete-conflicting-outputs

      - name: Run analyzer
        run: flutter analyze

      - name: Run tests with coverage
        run: flutter test --coverage

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          file: coverage/lcov.info

  # ─── Build Android ───
  build-android:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.0'
          channel: 'stable'
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Decode keystore
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE }}" | base64 -d > android/app/keystore.jks

      - name: Build APK
        run: |
          flutter build apk --release \
            --build-number=${{ github.run_number }} \
            --dart-define=API_URL=${{ secrets.API_URL }}
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}

      - name: Build App Bundle
        run: flutter build appbundle --release

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: android-release
          path: |
            build/app/outputs/flutter-apk/app-release.apk
            build/app/outputs/bundle/release/app-release.aab

  # ─── Build iOS ───
  build-ios:
    needs: test
    runs-on: macos-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.22.0'
          channel: 'stable'
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Install CocoaPods
        run: |
          cd ios
          pod install

      - name: Build iOS
        run: |
          flutter build ios --release --no-codesign \
            --build-number=${{ github.run_number }}

      - name: Archive app
        run: |
          xcodebuild -workspace ios/Runner.xcworkspace \
            -scheme Runner \
            -configuration Release \
            -archivePath build/Runner.xcarchive \
            archive

      - name: Export IPA
        run: |
          xcodebuild -exportArchive \
            -archivePath build/Runner.xcarchive \
            -exportOptionsPlist ios/ExportOptions.plist \
            -exportPath build/ios
```

---

## ขั้นตอนที่ 962: Android Signing

```groovy
// android/app/build.gradle
android {
    compileSdkVersion 34

    defaultConfig {
        applicationId "com.example.myapp"
        minSdkVersion 21
        targetSdkVersion 34
        versionCode flutterVersionCode.toInteger()
        versionName flutterVersionName
    }

    signingConfigs {
        release {
            storeFile file(System.getenv("KEYSTORE_PATH") ?: "keystore.jks")
            storePassword System.getenv("KEYSTORE_PASSWORD") ?: ""
            keyAlias System.getenv("KEY_ALIAS") ?: ""
            keyPassword System.getenv("KEY_PASSWORD") ?: ""
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }

    flavorDimensions "environment"
    productFlavors {
        dev {
            dimension "environment"
            applicationIdSuffix ".dev"
            versionNameSuffix "-dev"
            resValue "string", "app_name", "MyApp Dev"
        }
        staging {
            dimension "environment"
            applicationIdSuffix ".staging"
            versionNameSuffix "-staging"
            resValue "string", "app_name", "MyApp Staging"
        }
        prod {
            dimension "environment"
            resValue "string", "app_name", "MyApp"
        }
    }
}
```

---

## ขั้นตอนที่ 963: Flutter Flavors / Environments

```dart
// lib/core/config/app_config.dart
enum Environment { dev, staging, prod }

class AppConfig {
  static Environment _environment = Environment.dev;
  static Environment get environment => _environment;

  static late String _apiUrl;
  static late String _apiKey;
  static late bool _enableAnalytics;
  static late bool _showDebugInfo;

  static String get apiUrl => _apiUrl;
  static String get apiKey => _apiKey;
  static bool get enableAnalytics => _enableAnalytics;
  static bool get showDebugInfo => _showDebugInfo;

  static void initialize(Environment env) {
    _environment = env;

    switch (env) {
      case Environment.dev:
        _apiUrl = 'https://dev-api.example.com';
        _apiKey = 'dev-key';
        _enableAnalytics = false;
        _showDebugInfo = true;
        break;
      case Environment.staging:
        _apiUrl = 'https://staging-api.example.com';
        _apiKey = 'staging-key';
        _enableAnalytics = false;
        _showDebugInfo = true;
        break;
      case Environment.prod:
        _apiUrl = 'https://api.example.com';
        _apiKey = const String.fromEnvironment('API_KEY');
        _enableAnalytics = true;
        _showDebugInfo = false;
        break;
    }
  }

  static bool get isDev => _environment == Environment.dev;
  static bool get isStaging => _environment == Environment.staging;
  static bool get isProd => _environment == Environment.prod;
}

// ─── Entry points ───
// lib/main_dev.dart
void main() {
  AppConfig.initialize(Environment.dev);
  runApp(const MyApp());
}

// lib/main_staging.dart
void main() {
  AppConfig.initialize(Environment.staging);
  runApp(const MyApp());
}

// lib/main_prod.dart
void main() {
  AppConfig.initialize(Environment.prod);
  runApp(const MyApp());
}
```

---

## ขั้นตอนที่ 964: Fastlane สำหรับ iOS

```ruby
# ios/fastlane/Fastfile
default_platform(:ios)

platform :ios do
  before_all do
    setup_ci if is_ci
  end

  desc "Run all tests"
  lane :test do
    run_tests(
      workspace: "Runner.xcworkspace",
      devices: ["iPhone 15"],
      scheme: "Runner"
    )
  end

  desc "Deploy to TestFlight"
  lane :beta do
    # Increment build number
    increment_build_number(
      build_number: ENV["BUILD_NUMBER"]
    )

    # Code signing
    match(
      type: "appstore",
      readonly: is_ci
    )

    # Build
    build_app(
      workspace: "Runner.xcworkspace",
      scheme: "Runner",
      configuration: "Release"
    )

    # Upload to TestFlight
    upload_to_testflight(
      api_key_path: "fastlane/api_key.json",
      skip_waiting_for_build_processing: true
    )

    # Notify Slack
    slack(
      message: "✅ New iOS build uploaded to TestFlight",
      channel: "#releases",
      slack_url: ENV["SLACK_URL"]
    )
  end

  desc "Deploy to App Store"
  lane :release do
    # Screenshot
    capture_screenshots

    # Build
    build_app(
      workspace: "Runner.xcworkspace",
      scheme: "Runner",
      configuration: "Release"
    )

    # Submit to App Store
    deliver(
      submit_for_review: true,
      automatic_release: false,
      force: true
    )
  end
end
```

---

## ขั้นตอนที่ 965: Fastlane สำหรับ Android

```ruby
# android/fastlane/Fastfile
default_platform(:android)

platform :android do
  desc "Run tests"
  lane :test do
    gradle(
      task: "test",
      build_type: "Release"
    )
  end

  desc "Build and deploy to Play Console (Internal)"
  lane :internal do
    # Build AAB
    gradle(
      task: "bundle",
      build_type: "Release",
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"]
      }
    )

    # Upload to Play Store Internal track
    upload_to_play_store(
      track: "internal",
      aab: "../build/app/outputs/bundle/release/app-release.aab",
      json_key: "google-play-key.json"
    )
  end

  desc "Promote to Production"
  lane :production do
    upload_to_play_store(
      track: "production",
      track_promote_to: "production",
      rollout: "0.1"  # 10% rollout
    )
  end
end
```

---

## ขั้นตอนที่ 966: Semantic Versioning

```dart
// lib/core/versioning/version_manager.dart
class VersionManager {
  final int major;
  final int minor;
  final int patch;
  final String? preRelease;
  final int buildNumber;

  const VersionManager({
    required this.major,
    required this.minor,
    required this.patch,
    this.preRelease,
    required this.buildNumber,
  });

  factory VersionManager.parse(String version, {int buildNumber = 0}) {
    List<String> parts = version.split('.');
    String patchPart = parts[2];
    String? pre;

    if (patchPart.contains('-')) {
      List<String> patchSplit = patchPart.split('-');
      patchPart = patchSplit[0];
      pre = patchSplit[1];
    }

    return VersionManager(
      major: int.parse(parts[0]),
      minor: int.parse(parts[1]),
      patch: int.parse(patchPart),
      preRelease: pre,
      buildNumber: buildNumber,
    );
  }

  String get version => preRelease != null
      ? '$major.$minor.$patch-$preRelease'
      : '$major.$minor.$patch';

  String get fullVersion => '$version+$buildNumber';

  bool isNewerThan(VersionManager other) {
    if (major != other.major) return major > other.major;
    if (minor != other.minor) return minor > other.minor;
    return patch > other.patch;
  }

  VersionManager bumpMajor() => VersionManager(
    major: major + 1, minor: 0, patch: 0,
    buildNumber: buildNumber + 1,
  );

  VersionManager bumpMinor() => VersionManager(
    major: major, minor: minor + 1, patch: 0,
    buildNumber: buildNumber + 1,
  );

  VersionManager bumpPatch() => VersionManager(
    major: major, minor: minor, patch: patch + 1,
    buildNumber: buildNumber + 1,
  );

  @override
  String toString() => fullVersion;
}

// ─── Usage ───
void main() {
  VersionManager v1 = VersionManager.parse('1.2.3', buildNumber: 45);
  print(v1.version);     // 1.2.3
  print(v1.fullVersion); // 1.2.3+45

  VersionManager v2 = v1.bumpMinor();
  print(v2.version);     // 1.3.0

  print(v2.isNewerThan(v1));  // true
}
```

---

## ขั้นตอนที่ 967: Release Checklist

```dart
// Pre-release Checklist
class ReleaseChecklist {
  static const List<String> checks = [
    // ─── Code Quality ───
    '✅ flutter analyze: ไม่มี error/warning',
    '✅ flutter test: ทุก test ผ่าน',
    '✅ Coverage >= 80%',
    '✅ Code review ผ่านแล้ว',

    // ─── Performance ───
    '✅ Profile mode: FPS >= 60',
    '✅ ไม่มี memory leak',
    '✅ Image ถูก optimize',
    '✅ Bundle size ไม่ใหญ่เกิน',

    // ─── Security ───
    '✅ ไม่มี hardcoded secrets/API keys',
    '✅ Network calls ใช้ HTTPS',
    '✅ User data ถูก encrypt',
    '✅ Obfuscation เปิดใช้งาน (Android)',

    // ─── UX ───
    '✅ Loading states ทำงานถูกต้อง',
    '✅ Error messages ชัดเจน',
    '✅ Test บน devices จริงทั้ง Android/iOS',
    '✅ Accessibility ผ่าน',

    // ─── Store Requirements ───
    '✅ App icons ครบทุกขนาด',
    '✅ Screenshots ครบ',
    '✅ Privacy policy URL',
    '✅ Version number ถูกต้อง',
    '✅ Release notes เขียนแล้ว',
  ];
}
```

---

**← [Part 26 - Clean Architecture](part-26-clean-architecture.md)**

**ต่อไป: [Part 28 - Advanced Flutter Patterns →](part-28-advanced-patterns.md)**
