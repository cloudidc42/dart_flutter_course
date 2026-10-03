# Part 49: CI/CD for Flutter
## ขั้นตอนที่ 1841-1880

## 🎯 เป้าหมายของ Part นี้
- สร้าง GitHub Actions workflow สำหรับ Flutter (test, build, deploy)
- ตั้งค่า Fastlane สำหรับ iOS และ Android
- ใช้งาน Firebase App Distribution
- ตั้งค่า Code Signing อย่างถูกต้อง
- ทำ Automated Versioning แบบ semantic

---

## ขั้นตอนที่ 1841: โครงสร้าง CI/CD Files

```
.github/
├── workflows/
│   ├── ci.yml              # Run tests on every PR
│   ├── build-android.yml   # Build Android APK/AAB
│   ├── build-ios.yml       # Build iOS IPA
│   └── deploy.yml          # Deploy to Firebase/stores
fastlane/
├── Fastfile
├── Appfile
├── Matchfile
└── Pluginfile
scripts/
├── bump_version.sh
├── generate_changelog.sh
└── upload_symbols.sh
```

---

## ขั้นตอนที่ 1842: GitHub Actions - CI Workflow (Test & Analyze)

```yaml
# .github/workflows/ci.yml
name: Flutter CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  FLUTTER_VERSION: '3.16.0'

jobs:
  # ─── Analyze & Format ────────────────────────────────────────────
  analyze:
    name: Analyze & Format
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: stable
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Verify formatting
        run: dart format --output=none --set-exit-if-changed .

      - name: Analyze code
        run: flutter analyze --no-fatal-infos

      - name: Run build_runner
        run: dart run build_runner build --delete-conflicting-outputs

  # ─── Unit & Widget Tests ─────────────────────────────────────────
  test:
    name: Unit & Widget Tests
    runs-on: ubuntu-latest
    needs: analyze
    steps:
      - uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: stable
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Run tests with coverage
        run: flutter test --coverage --reporter=github

      - name: Check coverage threshold
        run: |
          COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep -oP '\d+\.\d+(?=%)' | head -1)
          echo "Coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 70" | bc -l) )); then
            echo "❌ Coverage $COVERAGE% is below 70% threshold"
            exit 1
          fi
          echo "✅ Coverage $COVERAGE% passes threshold"

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: coverage/lcov.info
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: false

  # ─── Security Scan ───────────────────────────────────────────────
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run dependency vulnerability scan
        uses: dart-lang/setup-dart@v1

      - name: Check for known vulnerabilities
        run: dart pub outdated --no-dev-dependencies

      - name: Scan for secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
```

---

## ขั้นตอนที่ 1843: GitHub Actions - Android Build

```yaml
# .github/workflows/build-android.yml
name: Build Android

on:
  push:
    branches: [main]
    tags: ['v*.*.*']
  workflow_dispatch:
    inputs:
      build_type:
        description: 'Build type (debug/release)'
        required: true
        default: 'debug'
        type: choice
        options:
          - debug
          - release

env:
  FLUTTER_VERSION: '3.16.0'
  JAVA_VERSION: '17'

jobs:
  build-android:
    name: Build Android ${{ inputs.build_type || 'release' }}
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Need full history for versioning

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: ${{ env.JAVA_VERSION }}

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: stable
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Decode keystore
        if: inputs.build_type == 'release' || github.ref_type == 'tag'
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 --decode > android/app/keystore.jks

      - name: Create key.properties
        if: inputs.build_type == 'release' || github.ref_type == 'tag'
        run: |
          cat > android/key.properties << EOF
          storePassword=${{ secrets.ANDROID_KEY_STORE_PASSWORD }}
          keyPassword=${{ secrets.ANDROID_KEY_PASSWORD }}
          keyAlias=${{ secrets.ANDROID_KEY_ALIAS }}
          storeFile=keystore.jks
          EOF

      - name: Get version from pubspec
        id: version
        run: |
          VERSION=$(grep '^version:' pubspec.yaml | sed 's/version: //')
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "Building version: $VERSION"

      - name: Build APK (debug)
        if: inputs.build_type == 'debug'
        run: flutter build apk --debug --build-number=${{ github.run_number }}

      - name: Build App Bundle (release)
        if: inputs.build_type == 'release' || github.ref_type == 'tag'
        run: |
          flutter build appbundle \
            --release \
            --build-number=${{ github.run_number }} \
            --dart-define=ENVIRONMENT=production \
            --obfuscate \
            --split-debug-info=symbols/

      - name: Upload APK artifact
        if: inputs.build_type == 'debug'
        uses: actions/upload-artifact@v4
        with:
          name: android-debug-${{ steps.version.outputs.version }}
          path: build/app/outputs/flutter-apk/app-debug.apk
          retention-days: 7

      - name: Upload AAB artifact
        if: inputs.build_type == 'release' || github.ref_type == 'tag'
        uses: actions/upload-artifact@v4
        with:
          name: android-release-${{ steps.version.outputs.version }}
          path: build/app/outputs/bundle/release/app-release.aab
          retention-days: 30

      - name: Upload debug symbols
        if: inputs.build_type == 'release' || github.ref_type == 'tag'
        uses: actions/upload-artifact@v4
        with:
          name: android-symbols-${{ steps.version.outputs.version }}
          path: symbols/
          retention-days: 30
```

---

## ขั้นตอนที่ 1844: GitHub Actions - iOS Build

```yaml
# .github/workflows/build-ios.yml
name: Build iOS

on:
  push:
    branches: [main]
    tags: ['v*.*.*']
  workflow_dispatch:

env:
  FLUTTER_VERSION: '3.16.0'

jobs:
  build-ios:
    name: Build iOS
    runs-on: macos-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Flutter
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

      - name: Import certificates
        uses: apple-actions/import-codesign-certs@v2
        with:
          p12-file-base64: ${{ secrets.IOS_P12_BASE64 }}
          p12-password: ${{ secrets.IOS_P12_PASSWORD }}

      - name: Install provisioning profile
        run: |
          PROFILE_PATH="$HOME/Library/MobileDevice/Provisioning Profiles"
          mkdir -p "$PROFILE_PATH"
          echo "${{ secrets.IOS_PROVISIONING_PROFILE_BASE64 }}" | base64 --decode > \
            "$PROFILE_PATH/app_store_profile.mobileprovision"

      - name: Build IPA
        run: |
          flutter build ipa \
            --release \
            --build-number=${{ github.run_number }} \
            --dart-define=ENVIRONMENT=production \
            --obfuscate \
            --split-debug-info=symbols/ \
            --export-options-plist=ios/ExportOptions.plist

      - name: Upload IPA artifact
        uses: actions/upload-artifact@v4
        with:
          name: ios-release-ipa
          path: build/ios/ipa/*.ipa
          retention-days: 30
```

---

## ขั้นตอนที่ 1845: GitHub Actions - Deploy to Firebase

```yaml
# .github/workflows/deploy.yml
name: Deploy to Firebase App Distribution

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      release_notes:
        description: 'Release notes for this build'
        required: false
        default: 'New build from CI'

env:
  FLUTTER_VERSION: '3.16.0'

jobs:
  deploy-android:
    name: Deploy Android to Firebase
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - run: flutter pub get

      - name: Decode keystore
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 --decode > android/app/keystore.jks
          cat > android/key.properties << EOF
          storePassword=${{ secrets.ANDROID_KEY_STORE_PASSWORD }}
          keyPassword=${{ secrets.ANDROID_KEY_PASSWORD }}
          keyAlias=${{ secrets.ANDROID_KEY_ALIAS }}
          storeFile=keystore.jks
          EOF

      - name: Build APK for distribution
        run: |
          flutter build apk \
            --release \
            --build-number=${{ github.run_number }}

      - name: Generate release notes
        id: notes
        run: |
          if [ -n "${{ github.event.inputs.release_notes }}" ]; then
            NOTES="${{ github.event.inputs.release_notes }}"
          else
            NOTES="Build #${{ github.run_number }} from commit: $(git log --format='%s' -1)"
          fi
          echo "notes=$NOTES" >> $GITHUB_OUTPUT

      - name: Upload to Firebase App Distribution
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{ secrets.FIREBASE_ANDROID_APP_ID }}
          serviceCredentialsFileContent: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          groups: internal-testers
          releaseNotes: ${{ steps.notes.outputs.notes }}
          file: build/app/outputs/flutter-apk/app-release.apk

  deploy-ios:
    name: Deploy iOS to Firebase
    runs-on: macos-latest

    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          cache: true

      - run: flutter pub get
      - run: cd ios && pod install

      - uses: apple-actions/import-codesign-certs@v2
        with:
          p12-file-base64: ${{ secrets.IOS_P12_BASE64 }}
          p12-password: ${{ secrets.IOS_P12_PASSWORD }}

      - name: Install provisioning profile
        run: |
          PROFILE_PATH="$HOME/Library/MobileDevice/Provisioning Profiles"
          mkdir -p "$PROFILE_PATH"
          echo "${{ secrets.IOS_PROVISIONING_PROFILE_BASE64 }}" | base64 --decode > \
            "$PROFILE_PATH/app_store_profile.mobileprovision"

      - name: Build IPA for distribution
        run: |
          flutter build ipa \
            --release \
            --build-number=${{ github.run_number }} \
            --export-options-plist=ios/ExportOptions.plist

      - name: Upload to Firebase App Distribution
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{ secrets.FIREBASE_IOS_APP_ID }}
          serviceCredentialsFileContent: ${{ secrets.FIREBASE_SERVICE_ACCOUNT }}
          groups: internal-testers
          releaseNotes: "Build #${{ github.run_number }}"
          file: build/ios/ipa/*.ipa
```

---

## ขั้นตอนที่ 1846: Fastlane Setup

```ruby
# fastlane/Fastfile
default_platform(:android)

FLUTTER_BUILD_NUMBER = ENV["BUILD_NUMBER"] || "1"
APP_VERSION = ENV["APP_VERSION"] || "1.0.0"

# ─── Android Lanes ──────────────────────────────────────────────────
platform :android do
  desc "Run all Android tests"
  lane :test do
    Dir.chdir("..") do
      sh("flutter test --coverage")
    end
  end

  desc "Build Android debug APK"
  lane :build_debug do
    Dir.chdir("..") do
      sh("flutter build apk --debug --build-number=#{FLUTTER_BUILD_NUMBER}")
    end
  end

  desc "Build Android release AAB"
  lane :build_release do
    Dir.chdir("..") do
      sh(
        "flutter build appbundle " \
        "--release " \
        "--build-number=#{FLUTTER_BUILD_NUMBER} " \
        "--build-name=#{APP_VERSION} " \
        "--dart-define=ENVIRONMENT=production " \
        "--obfuscate " \
        "--split-debug-info=symbols/"
      )
    end
  end

  desc "Deploy to Firebase App Distribution (internal)"
  lane :deploy_firebase_internal do
    build_release
    firebase_app_distribution(
      app: ENV["FIREBASE_ANDROID_APP_ID"],
      service_credentials_file: "fastlane/service-account.json",
      groups: "internal-testers",
      release_notes: "Build #{FLUTTER_BUILD_NUMBER}",
      android_artifact_type: "AAB",
      android_artifact_path: "build/app/outputs/bundle/release/app-release.aab"
    )
  end

  desc "Deploy to Google Play (internal track)"
  lane :deploy_play_internal do
    build_release
    upload_to_play_store(
      track: "internal",
      aab: "build/app/outputs/bundle/release/app-release.aab",
      skip_upload_metadata: true,
      skip_upload_images: true,
      skip_upload_screenshots: true
    )
  end

  desc "Promote from internal to production on Play Store"
  lane :promote_to_production do
    upload_to_play_store(
      track: "internal",
      track_promote_to: "production",
      skip_upload_apk: true,
      skip_upload_aab: true,
      skip_upload_metadata: false,
      rollout: "0.1"   # 10% rollout
    )
  end
end

# ─── iOS Lanes ──────────────────────────────────────────────────────
platform :ios do
  desc "Sync certificates and provisioning profiles"
  lane :certificates do
    match(
      type: "appstore",
      app_identifier: "com.example.myapp",
      readonly: true
    )
  end

  desc "Build iOS release IPA"
  lane :build_release do
    certificates
    Dir.chdir("..") do
      sh(
        "flutter build ipa " \
        "--release " \
        "--build-number=#{FLUTTER_BUILD_NUMBER} " \
        "--build-name=#{APP_VERSION} " \
        "--dart-define=ENVIRONMENT=production " \
        "--export-options-plist=ios/ExportOptions.plist"
      )
    end
  end

  desc "Deploy to Firebase App Distribution"
  lane :deploy_firebase_internal do
    build_release
    firebase_app_distribution(
      app: ENV["FIREBASE_IOS_APP_ID"],
      service_credentials_file: "fastlane/service-account.json",
      groups: "internal-testers",
      release_notes: "Build #{FLUTTER_BUILD_NUMBER}",
      ipa_path: "build/ios/ipa/Runner.ipa"
    )
  end

  desc "Deploy to TestFlight"
  lane :deploy_testflight do
    build_release
    upload_to_testflight(
      ipa: "build/ios/ipa/Runner.ipa",
      skip_waiting_for_build_processing: false,
      distribute_external: false
    )
  end

  desc "Deploy to App Store"
  lane :deploy_appstore do
    deploy_testflight
    deliver(
      submit_for_review: true,
      automatic_release: false,
      force: true,
      skip_binary_upload: true,
      skip_screenshots: false,
      skip_metadata: false
    )
  end
end
```

---

## ขั้นตอนที่ 1847: Fastlane Appfile

```ruby
# fastlane/Appfile
app_identifier("com.example.myapp")

# Apple credentials
apple_id("developer@example.com")
team_id("XXXXXXXXXX")
itc_team_id("XXXXXXXXXX")  # App Store Connect Team ID

# Android credentials
json_key_file("fastlane/play-store-service-account.json")
package_name("com.example.myapp")
```

---

## ขั้นตอนที่ 1848: Fastlane Matchfile (Code Signing)

```ruby
# fastlane/Matchfile
git_url("https://github.com/your-org/certificates")
git_branch("main")

storage_mode("git")
type("appstore")

app_identifier(["com.example.myapp"])
username("developer@example.com")
team_id("XXXXXXXXXX")

# Readonly in CI
readonly(true) if ENV["CI"]
```

---

## ขั้นตอนที่ 1849: iOS ExportOptions.plist

```xml
<!-- ios/ExportOptions.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>app-store</string>
  <key>teamID</key>
  <string>XXXXXXXXXX</string>
  <key>uploadBitcode</key>
  <false/>
  <key>uploadSymbols</key>
  <true/>
  <key>compileBitcode</key>
  <false/>
  <key>destination</key>
  <string>export</string>
  <key>signingStyle</key>
  <string>manual</string>
  <key>provisioningProfiles</key>
  <dict>
    <key>com.example.myapp</key>
    <string>match AppStore com.example.myapp</string>
  </dict>
  <key>stripSwiftSymbols</key>
  <true/>
  <key>thinning</key>
  <string>&lt;none&gt;</string>
</dict>
</plist>
```

---

## ขั้นตอนที่ 1850: Android Signing Configuration

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

    // Load signing config from key.properties
    def keystoreProperties = new Properties()
    def keystorePropertiesFile = rootProject.file('key.properties')
    if (keystorePropertiesFile.exists()) {
        keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
    }

    signingConfigs {
        debug {
            storeFile file('debug.keystore')
            storePassword 'android'
            keyAlias 'androiddebugkey'
            keyPassword 'android'
        }
        release {
            if (keystorePropertiesFile.exists()) {
                keyAlias keystoreProperties['keyAlias']
                keyPassword keystoreProperties['keyPassword']
                storeFile file(keystoreProperties['storeFile'])
                storePassword keystoreProperties['storePassword']
            }
        }
    }

    buildTypes {
        debug {
            signingConfig signingConfigs.debug
            debuggable true
            applicationIdSuffix '.debug'
            versionNameSuffix '-debug'
        }
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'),
                         'proguard-rules.pro'
        }
    }

    // Build flavors
    flavorDimensions "env"
    productFlavors {
        development {
            dimension "env"
            applicationIdSuffix ".dev"
            versionNameSuffix "-dev"
            resValue "string", "app_name", "MyApp Dev"
        }
        staging {
            dimension "env"
            applicationIdSuffix ".staging"
            versionNameSuffix "-staging"
            resValue "string", "app_name", "MyApp Staging"
        }
        production {
            dimension "env"
            resValue "string", "app_name", "MyApp"
        }
    }
}
```

---

## ขั้นตอนที่ 1851: Automated Versioning Script

```bash
#!/bin/bash
# scripts/bump_version.sh
# Usage: ./scripts/bump_version.sh [major|minor|patch]

set -e

PUBSPEC="pubspec.yaml"
BUMP_TYPE=${1:-patch}

# Extract current version
CURRENT=$(grep '^version:' $PUBSPEC | sed 's/version: //')
VERSION_NAME=$(echo $CURRENT | cut -d'+' -f1)
BUILD_NUM=$(echo $CURRENT | cut -d'+' -f2)

# Split version name
MAJOR=$(echo $VERSION_NAME | cut -d'.' -f1)
MINOR=$(echo $VERSION_NAME | cut -d'.' -f2)
PATCH=$(echo $VERSION_NAME | cut -d'.' -f3)

# Bump version
case $BUMP_TYPE in
  major)
    MAJOR=$((MAJOR + 1))
    MINOR=0
    PATCH=0
    ;;
  minor)
    MINOR=$((MINOR + 1))
    PATCH=0
    ;;
  patch)
    PATCH=$((PATCH + 1))
    ;;
  *)
    echo "Usage: $0 [major|minor|patch]"
    exit 1
    ;;
esac

NEW_BUILD=$((BUILD_NUM + 1))
NEW_VERSION="${MAJOR}.${MINOR}.${PATCH}+${NEW_BUILD}"

# Update pubspec.yaml
sed -i "s/^version: .*/version: ${NEW_VERSION}/" $PUBSPEC

echo "Bumped: $CURRENT → $NEW_VERSION"

# Create git commit and tag
git add $PUBSPEC
git commit -m "chore: bump version to $NEW_VERSION"
git tag "v${MAJOR}.${MINOR}.${PATCH}"

echo "✅ Version bumped to $NEW_VERSION and tagged v${MAJOR}.${MINOR}.${PATCH}"
```

---

## ขั้นตอนที่ 1852: Changelog Generator

```bash
#!/bin/bash
# scripts/generate_changelog.sh
# Generates changelog from git commits since last tag

set -e

LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
OUTPUT_FILE="CHANGELOG.md"

echo "# Changelog" > $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

if [ -z "$LAST_TAG" ]; then
  COMMIT_RANGE="HEAD"
  echo "## Unreleased" >> $OUTPUT_FILE
else
  COMMIT_RANGE="${LAST_TAG}..HEAD"
  echo "## $(git describe --tags --abbrev=0)" >> $OUTPUT_FILE
fi

echo "" >> $OUTPUT_FILE
echo "**Released:** $(date '+%Y-%m-%d')" >> $OUTPUT_FILE
echo "" >> $OUTPUT_FILE

# Features
FEATURES=$(git log $COMMIT_RANGE --format="%s" | grep "^feat:" || true)
if [ -n "$FEATURES" ]; then
  echo "### ✨ Features" >> $OUTPUT_FILE
  echo "$FEATURES" | sed 's/^feat: /- /' >> $OUTPUT_FILE
  echo "" >> $OUTPUT_FILE
fi

# Bug Fixes
FIXES=$(git log $COMMIT_RANGE --format="%s" | grep "^fix:" || true)
if [ -n "$FIXES" ]; then
  echo "### 🐛 Bug Fixes" >> $OUTPUT_FILE
  echo "$FIXES" | sed 's/^fix: /- /' >> $OUTPUT_FILE
  echo "" >> $OUTPUT_FILE
fi

# Breaking Changes
BREAKING=$(git log $COMMIT_RANGE --format="%s" | grep "BREAKING" || true)
if [ -n "$BREAKING" ]; then
  echo "### ⚠️ Breaking Changes" >> $OUTPUT_FILE
  echo "$BREAKING" | sed 's/^/- /' >> $OUTPUT_FILE
  echo "" >> $OUTPUT_FILE
fi

echo "✅ Changelog written to $OUTPUT_FILE"
cat $OUTPUT_FILE
```

---

## ขั้นตอนที่ 1853: Firebase App Distribution in Flutter (Dart script)

```dart
// scripts/distribute_build.dart
// Dart script to automate Firebase App Distribution via CLI

import 'dart:io';

Future<void> main(List<String> args) async {
  if (args.isEmpty) {
    print('Usage: dart scripts/distribute_build.dart [android|ios]');
    exit(1);
  }

  final platform = args[0];
  final buildNumber = Platform.environment['BUILD_NUMBER'] ?? '1';
  final releaseNotes = Platform.environment['RELEASE_NOTES'] ??
      'Build #$buildNumber';

  print('📦 Distributing $platform build #$buildNumber...');

  switch (platform) {
    case 'android':
      await distributeAndroid(releaseNotes, buildNumber);
    case 'ios':
      await distributeIos(releaseNotes, buildNumber);
    default:
      print('Unknown platform: $platform');
      exit(1);
  }
}

Future<void> distributeAndroid(
    String releaseNotes, String buildNumber) async {
  // Build APK
  print('Building Android release...');
  await runProcess('flutter', [
    'build',
    'apk',
    '--release',
    '--build-number=$buildNumber',
  ]);

  // Upload to Firebase
  final appId = Platform.environment['FIREBASE_ANDROID_APP_ID']!;
  await runProcess('firebase', [
    'appdistribution:distribute',
    'build/app/outputs/flutter-apk/app-release.apk',
    '--app', appId,
    '--groups', 'internal-testers',
    '--release-notes', releaseNotes,
  ]);

  print('✅ Android distributed successfully');
}

Future<void> distributeIos(String releaseNotes, String buildNumber) async {
  // Build IPA
  print('Building iOS release...');
  await runProcess('flutter', [
    'build',
    'ipa',
    '--release',
    '--build-number=$buildNumber',
    '--export-options-plist=ios/ExportOptions.plist',
  ]);

  // Upload to Firebase
  final appId = Platform.environment['FIREBASE_IOS_APP_ID']!;
  await runProcess('firebase', [
    'appdistribution:distribute',
    'build/ios/ipa/Runner.ipa',
    '--app', appId,
    '--groups', 'internal-testers',
    '--release-notes', releaseNotes,
  ]);

  print('✅ iOS distributed successfully');
}

Future<void> runProcess(String command, List<String> args) async {
  print('Running: $command ${args.join(' ')}');
  final result = await Process.run(command, args);

  if (result.exitCode != 0) {
    print('STDOUT: ${result.stdout}');
    print('STDERR: ${result.stderr}');
    throw Exception('Command failed with exit code ${result.exitCode}');
  }

  print(result.stdout);
}
```

---

## ขั้นตอนที่ 1854: GitHub Secrets Required

```bash
# Required GitHub Secrets for CI/CD

# Android
ANDROID_KEYSTORE_BASE64        # base64 encoded .jks keystore file
ANDROID_KEY_STORE_PASSWORD     # keystore password
ANDROID_KEY_PASSWORD           # key password
ANDROID_KEY_ALIAS              # key alias

# iOS
IOS_P12_BASE64                 # base64 encoded .p12 certificate
IOS_P12_PASSWORD               # p12 password
IOS_PROVISIONING_PROFILE_BASE64 # base64 encoded .mobileprovision file

# Firebase
FIREBASE_ANDROID_APP_ID        # Firebase Android App ID
FIREBASE_IOS_APP_ID            # Firebase iOS App ID
FIREBASE_SERVICE_ACCOUNT       # Firebase service account JSON (base64)

# Codecov
CODECOV_TOKEN                  # Codecov.io token

# App Store Connect
APPLE_ID                       # Apple ID email
APP_STORE_CONNECT_API_KEY_ID   # API Key ID
APP_STORE_CONNECT_ISSUER_ID    # Issuer ID
APP_STORE_CONNECT_API_KEY      # base64 encoded .p8 key file

# How to encode files as base64:
# macOS/Linux: base64 -i keystore.jks | pbcopy
# Windows:     certutil -encode keystore.jks keystore.b64
```

---

## ขั้นตอนที่ 1855: Complete CI Pipeline Summary

```yaml
# .github/workflows/complete-pipeline.yml
name: Complete CI/CD Pipeline

on:
  push:
    branches: [main]
    tags: ['v*.*.*']

jobs:
  test:
    uses: ./.github/workflows/ci.yml

  build-android:
    needs: test
    uses: ./.github/workflows/build-android.yml
    with:
      build_type: release
    secrets: inherit

  build-ios:
    needs: test
    uses: ./.github/workflows/build-ios.yml
    secrets: inherit

  deploy:
    needs: [build-android, build-ios]
    if: github.ref_type == 'tag'
    uses: ./.github/workflows/deploy.yml
    secrets: inherit

  notify:
    needs: [deploy]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Notify Slack
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          channel: '#deployments'
          text: |
            Deploy ${{ job.status }} for ${{ github.ref }}
            Build: ${{ github.run_number }}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

**← [Part 48 - App Architecture Patterns](part-48-app-architecture-patterns.md)**
**ต่อไป: [Part 50 - Performance Optimization →](part-50-performance-optimization.md)**
