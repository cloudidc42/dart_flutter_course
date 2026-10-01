# Part 01: แนะนำ Dart & Flutter และการติดตั้ง
## ขั้นตอนที่ 1-20

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจว่า Dart และ Flutter คืออะไร
- เข้าใจประวัติและที่มาของ Dart & Flutter
- ติดตั้ง Flutter SDK บนระบบปฏิบัติการต่างๆ
- ตั้งค่า IDE (VS Code หรือ Android Studio)
- สร้างและรันโปรเจกต์ Flutter แรกของคุณ
- เข้าใจโครงสร้างของโปรเจกต์ Flutter

---

## ขั้นตอนที่ 1: Dart คืออะไร?

**Dart** คือภาษาโปรแกรมมิ่งที่พัฒนาโดย **Google** โดยเปิดตัวครั้งแรกในปี 2011 มีลักษณะเด่นดังนี้:

### ลักษณะเฉพาะของ Dart

```
🔷 Strongly Typed Language
   - ทุกตัวแปรมีชนิดข้อมูลที่ชัดเจน
   - ตรวจสอบข้อผิดพลาดได้ตั้งแต่ตอน Compile

🔷 Object-Oriented Programming (OOP)
   - รองรับ Class, Object, Inheritance
   - รองรับ Mixins, Extension Methods

🔷 Null Safety
   - ป้องกัน Null Pointer Exception
   - ทำให้โค้ดปลอดภัยมากขึ้น

🔷 Asynchronous Programming
   - รองรับ async/await
   - รองรับ Futures และ Streams

🔷 Compile หลายรูปแบบ
   - Compile เป็น Native Code (ARM, x64)
   - Compile เป็น JavaScript
   - JIT (Just-In-Time) สำหรับ Development
   - AOT (Ahead-Of-Time) สำหรับ Production
```

### ตัวอย่างโค้ด Dart เบื้องต้น

```dart
// Hello World ใน Dart
void main() {
  print('สวัสดี Dart!');
  
  // การประกาศตัวแปร
  String name = 'Flutter Developer';
  int age = 25;
  double salary = 50000.50;
  bool isLearning = true;
  
  // การแสดงผล
  print('ชื่อ: $name');
  print('อายุ: $age ปี');
  print('เงินเดือน: $salary บาท');
  print('กำลังเรียน: $isLearning');
}
```

**ผลลัพธ์:**
```
สวัสดี Dart!
ชื่อ: Flutter Developer
อายุ: 25 ปี
เงินเดือน: 50000.5 บาท
กำลังเรียน: true
```

---

## ขั้นตอนที่ 2: Flutter คืออะไร?

**Flutter** คือ UI Framework สำหรับสร้างแอปพลิเคชัน Cross-Platform ที่พัฒนาโดย Google เปิดตัวในปี 2018

### Flutter รองรับแพลตฟอร์มอะไรบ้าง?

```
📱 Mobile
├── Android (Native ARM)
└── iOS (Native ARM)

🌐 Web
├── HTML/CSS/JavaScript
└── WebAssembly (ในอนาคต)

💻 Desktop
├── Windows
├── macOS
└── Linux

📺 Embedded
└── Google TV, Fuchsia OS และอื่นๆ
```

### ทำไมต้องใช้ Flutter?

| เหตุผล | รายละเอียด |
|--------|-----------|
| **Single Codebase** | เขียนครั้งเดียว รันได้ทุกแพลตฟอร์ม |
| **Hot Reload** | เห็นผลลัพธ์ทันทีขณะพัฒนา |
| **Performance** | ใกล้เคียง Native เพราะ Compile เป็น ARM |
| **Beautiful UI** | Widget ที่สวยงามและยืดหยุ่น |
| **Google Support** | พัฒนาและ Maintain โดย Google |
| **Community** | Community ขนาดใหญ่และเติบโตเร็ว |

---

## ขั้นตอนที่ 3: เปรียบเทียบ Flutter กับเทคโนโลยีอื่น

```
┌─────────────────┬─────────────┬────────────┬────────────┐
│ Feature         │   Flutter   │ React Native│  Xamarin   │
├─────────────────┼─────────────┼────────────┼────────────┤
│ ภาษา            │    Dart     │ JavaScript │    C#      │
│ Performance     │   ⭐⭐⭐⭐⭐   │   ⭐⭐⭐⭐    │   ⭐⭐⭐     │
│ Hot Reload      │     ✅      │     ✅      │    ✅      │
│ Native UI       │     ❌      │     ✅      │    ✅      │
│ Custom UI       │     ✅      │     ✅      │    ✅      │
│ Bundle Size     │   ปานกลาง   │   ปานกลาง  │    ใหญ่    │
│ Web Support     │     ✅      │     ✅      │    ✅      │
│ Desktop Support │     ✅      │   บางส่วน  │    ✅      │
└─────────────────┴─────────────┴────────────┴────────────┘
```

---

## ขั้นตอนที่ 4: ประวัติและพัฒนาการของ Dart & Flutter

### Timeline ที่สำคัญ

```
2011 ──► Dart เปิดตัวครั้งแรกโดย Google
         Lars Bak และ Kasper Lund เป็นผู้สร้าง

2013 ──► Dart 1.0 เปิดตัวอย่างเป็นทางการ

2018 ──► Flutter 1.0 (December)
         รองรับ Android และ iOS

2019 ──► Flutter 1.9
         เพิ่ม Web Support (Beta)

2020 ──► Flutter 1.17, 1.20, 1.22
         Null Safety เริ่มพัฒนา
         
2021 ──► Flutter 2.0 (March)
         รองรับ Web, Desktop อย่างเป็นทางการ
         Dart 2.12: Sound Null Safety

2022 ──► Flutter 3.0 (May)
         รองรับ macOS, Linux อย่างเป็นทางการ
         
2023 ──► Flutter 3.7, 3.10, 3.13
         Impeller Graphics Engine
         Dart 3.0: Records, Patterns
         
2024 ──► Flutter 3.16, 3.19, 3.22
         Material 3 เป็น Default
         WebAssembly Support

2025 ──► Flutter 3.24+
         AI Integration Tools
         Enhanced Performance
```

---

## ขั้นตอนที่ 5: โครงสร้างของ Flutter Framework

```
┌─────────────────────────────────────────────────────────┐
│                    Your Flutter App                      │
├─────────────────────────────────────────────────────────┤
│                   Dart Framework                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Flutter Widgets                      │   │
│  │  Material Design │ Cupertino │ Custom Widgets    │   │
│  ├─────────────────────────────────────────────────┤   │
│  │              Flutter Engine                       │   │
│  │   Skia/Impeller │ Dart Runtime │ Text Layout     │   │
│  └─────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────┤
│              Platform-Specific Layer                     │
│     Android │ iOS │ Web │ Windows │ macOS │ Linux       │
└─────────────────────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 6: ติดตั้ง Flutter SDK บน Windows

### ความต้องการของระบบ (Windows)

```
OS:     Windows 10/11 (64-bit)
RAM:    8 GB (แนะนำ 16 GB)
Storage: 5 GB
Tools:  Git for Windows, PowerShell 5.0+
```

### วิธีติดตั้ง

**วิธีที่ 1: ติดตั้งผ่าน Flutter official website**

```powershell
# ขั้นตอนที่ 1: ดาวน์โหลด Flutter SDK
# ไปที่ https://flutter.dev/docs/get-started/install/windows
# ดาวน์โหลด flutter_windows_x.x.x-stable.zip

# ขั้นตอนที่ 2: แตกไฟล์ไปที่ C:\
# แตกไฟล์ไปที่ C:\flutter (อย่าวางใน C:\Program Files)

# ขั้นตอนที่ 3: เพิ่ม Flutter ใน PATH
# เปิด System Properties > Environment Variables
# เพิ่ม C:\flutter\bin ใน PATH

# ขั้นตอนที่ 4: ตรวจสอบการติดตั้ง
flutter --version
flutter doctor
```

**วิธีที่ 2: ติดตั้งผ่าน Chocolatey (แนะนำ)**

```powershell
# ติดตั้ง Chocolatey ก่อน (รัน PowerShell เป็น Administrator)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# ติดตั้ง Flutter
choco install flutter

# ตรวจสอบ
flutter --version
```

**วิธีที่ 3: ติดตั้งผ่าน Windows Package Manager (winget)**

```powershell
# ติดตั้งผ่าน winget
winget install Google.Flutter

# ตรวจสอบ
flutter --version
flutter doctor
```

---

## ขั้นตอนที่ 7: ติดตั้ง Flutter SDK บน macOS

### ความต้องการของระบบ (macOS)

```
OS:     macOS 12 Monterey หรือใหม่กว่า
RAM:    8 GB (แนะนำ 16 GB)
Storage: 5 GB
Tools:  Xcode, CocoaPods, Homebrew
```

### วิธีติดตั้ง

**วิธีที่ 1: ติดตั้งผ่าน Homebrew (แนะนำ)**

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง Flutter
brew install --cask flutter

# ตรวจสอบ
flutter --version
flutter doctor

# ติดตั้ง Xcode (จาก App Store)
# จากนั้นรัน:
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -runFirstLaunch

# ติดตั้ง CocoaPods
sudo gem install cocoapods
```

**วิธีที่ 2: ติดตั้งด้วยตนเอง**

```bash
# ดาวน์โหลด Flutter SDK
# ไปที่ https://flutter.dev/docs/get-started/install/macos

# แตกไฟล์
cd ~/
unzip ~/Downloads/flutter_macos_x.x.x-stable.zip

# เพิ่ม PATH (เพิ่มใน ~/.zshrc หรือ ~/.bash_profile)
export PATH="$PATH:$HOME/flutter/bin"

# โหลด config ใหม่
source ~/.zshrc

# ตรวจสอบ
flutter --version
flutter doctor
```

---

## ขั้นตอนที่ 8: ติดตั้ง Flutter SDK บน Linux

### ความต้องการของระบบ (Linux)

```
OS:     Ubuntu 20.04 LTS หรือใหม่กว่า
RAM:    8 GB (แนะนำ 16 GB)
Storage: 5 GB
Tools:  git, curl, unzip, xz-utils, zip, libglu1-mesa
```

### วิธีติดตั้ง

```bash
# ขั้นตอนที่ 1: ติดตั้ง dependencies
sudo apt-get update
sudo apt-get install -y curl git unzip xz-utils zip libglu1-mesa

# ขั้นตอนที่ 2: ดาวน์โหลด Flutter SDK
cd ~
git clone https://github.com/flutter/flutter.git -b stable

# หรือดาวน์โหลดจาก website
# wget https://storage.googleapis.com/flutter_infra_release/releases/stable/linux/flutter_linux_x.x.x-stable.tar.xz
# tar xf flutter_linux_x.x.x-stable.tar.xz

# ขั้นตอนที่ 3: เพิ่ม PATH (เพิ่มใน ~/.bashrc หรือ ~/.zshrc)
echo 'export PATH="$PATH:$HOME/flutter/bin"' >> ~/.bashrc
source ~/.bashrc

# ขั้นตอนที่ 4: ตรวจสอบการติดตั้ง
flutter --version
flutter doctor

# สำหรับ Android Development
sudo apt-get install -y android-sdk

# ติดตั้ง Android Studio (แนะนำ)
sudo snap install android-studio --classic
```

---

## ขั้นตอนที่ 9: ทำความเข้าใจ flutter doctor

`flutter doctor` เป็นคำสั่งที่ตรวจสอบว่าระบบของคุณพร้อมสำหรับการพัฒนา Flutter หรือไม่

### ตัวอย่างผลลัพธ์ flutter doctor

```
Doctor summary (to see all details, run flutter doctor -v):
[✓] Flutter (Channel stable, 3.22.0, on macOS 14.0)
[✓] Android toolchain - develop for Android devices (Android SDK version 34.0.0)
[✓] Xcode - develop for iOS and macOS (Xcode 15.0)
[✓] Chrome - develop for the web
[✓] Android Studio (version 2023.1)
[✓] VS Code (version 1.85.0)
[✓] Connected device (3 available)
[✓] Network resources

• No issues found!
```

### วิธีแก้ปัญหาที่พบบ่อย

```bash
# ถ้า Android toolchain มีปัญหา
flutter doctor --android-licenses
# กด y ทุกครั้งที่ถาม

# ถ้า Xcode มีปัญหา (macOS)
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -runFirstLaunch

# ถ้า Flutter เวอร์ชันเก่า
flutter upgrade

# ดูข้อมูลเพิ่มเติม
flutter doctor -v
```

---

## ขั้นตอนที่ 10: ติดตั้งและตั้งค่า VS Code

**VS Code** เป็น IDE ที่แนะนำสำหรับการพัฒนา Flutter เพราะเบา รวดเร็ว และมี Extension ที่ดีมาก

### การติดตั้ง VS Code

```bash
# macOS
brew install --cask visual-studio-code

# Ubuntu/Debian
sudo apt-get install -y software-properties-common apt-transport-https wget
wget -q https://packages.microsoft.com/keys/microsoft.asc -O- | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://packages.microsoft.com/repos/vscode stable main"
sudo apt-get update
sudo apt-get install code

# Windows
# ดาวน์โหลดจาก https://code.visualstudio.com/
```

### Extensions ที่จำเป็นสำหรับ Flutter

```
1. Flutter (dart-code.flutter)
   - Flutter support สำหรับ VS Code
   - ติดตั้งอัตโนมัติ Dart extension ด้วย

2. Dart (dart-code.dart-code)
   - Dart language support
   - Code completion, syntax highlighting

3. Flutter Widget Snippets
   - Snippets สำหรับ Flutter Widgets

4. Awesome Flutter Snippets
   - Shortcuts สำหรับ Flutter code

5. Error Lens (usernamehw.errorlens)
   - แสดง error inline ในโค้ด

6. Bracket Pair Colorizer (แนะนำ)
   - ทำให้อ่านโค้ดง่ายขึ้น

7. GitLens
   - Git integration ขั้นสูง
```

### วิธีติดตั้ง Extensions

```
1. เปิด VS Code
2. กด Ctrl+P (Windows/Linux) หรือ Cmd+P (macOS)
3. พิมพ์: ext install dart-code.flutter
4. กด Enter
```

### การตั้งค่า VS Code สำหรับ Flutter

```json
// settings.json (Ctrl+Shift+P > Open Settings JSON)
{
  "editor.formatOnSave": true,
  "editor.formatOnType": true,
  "dart.lineLength": 80,
  "dart.warnAboutAllPackageVersions": true,
  "dart.completeFunctionCalls": true,
  "[dart]": {
    "editor.defaultFormatter": "Dart-Code.dart-code",
    "editor.formatOnSave": true,
    "editor.formatOnPaste": true,
    "editor.selectionHighlight": false,
    "editor.suggest.snippetsPreventQuickSuggestions": false,
    "editor.suggestSelection": "first",
    "editor.tabCompletion": "onlySnippets",
    "editor.wordBasedSuggestions": "off"
  }
}
```

---

## ขั้นตอนที่ 11: ติดตั้งและตั้งค่า Android Studio

**Android Studio** มีความสามารถมากกว่า VS Code แต่ใช้ทรัพยากรมากกว่า เหมาะกับการพัฒนา Android โดยเฉพาะ

### การติดตั้ง Android Studio

```bash
# macOS
brew install --cask android-studio

# Ubuntu
sudo snap install android-studio --classic

# Windows
# ดาวน์โหลดจาก https://developer.android.com/studio
```

### การตั้งค่า Android Studio สำหรับ Flutter

```
1. เปิด Android Studio
2. ไปที่ Plugins (File > Settings > Plugins บน Windows/Linux)
                 (Android Studio > Preferences > Plugins บน macOS)
3. ค้นหา "Flutter" และติดตั้ง
4. ค้นหา "Dart" และติดตั้ง
5. Restart Android Studio
```

### ตั้งค่า Android SDK

```
1. ไปที่ SDK Manager (Tools > SDK Manager)
2. เลือก Android SDK Location
3. ติดตั้ง Android SDK Platform:
   - Android 14 (API 34) - แนะนำ
   - Android 13 (API 33)
4. ติดตั้ง Android SDK Build-Tools
5. ติดตั้ง Android Emulator
```

---

## ขั้นตอนที่ 12: สร้างโปรเจกต์ Flutter แรก

### วิธีที่ 1: สร้างผ่าน Command Line (แนะนำ)

```bash
# ไปที่ directory ที่ต้องการสร้างโปรเจกต์
cd ~/Documents/projects

# สร้างโปรเจกต์ใหม่
flutter create my_first_app

# หรือสร้างพร้อมระบุ options
flutter create \
  --org com.yourcompany \
  --project-name my_first_app \
  --description "My First Flutter App" \
  --platforms android,ios,web \
  my_first_app

# เข้าไปในโปรเจกต์
cd my_first_app

# รันแอป
flutter run
```

### วิธีที่ 2: สร้างผ่าน VS Code

```
1. เปิด VS Code
2. กด Ctrl+Shift+P (Windows/Linux) หรือ Cmd+Shift+P (macOS)
3. พิมพ์: Flutter: New Project
4. เลือก: Application
5. เลือก directory
6. ตั้งชื่อโปรเจกต์: my_first_app
7. กด Enter
```

### วิธีที่ 3: สร้างผ่าน Android Studio

```
1. เปิด Android Studio
2. คลิก "New Flutter Project"
3. เลือก Flutter Application
4. ตั้งค่า:
   - Project name: my_first_app
   - Flutter SDK path: /path/to/flutter
   - Project location: /path/to/projects
   - Description: My First Flutter App
5. คลิก Finish
```

---

## ขั้นตอนที่ 13: โครงสร้างโปรเจกต์ Flutter

หลังจากสร้างโปรเจกต์ คุณจะเห็นโครงสร้างไฟล์ดังนี้:

```
my_first_app/
├── android/                    # Android-specific code
│   ├── app/
│   │   ├── src/
│   │   │   └── main/
│   │   │       ├── AndroidManifest.xml
│   │   │       └── kotlin/
│   │   └── build.gradle
│   └── gradle/
├── ios/                        # iOS-specific code
│   ├── Runner/
│   │   ├── AppDelegate.swift
│   │   └── Info.plist
│   └── Podfile
├── lib/                        # ✨ โค้ด Dart หลัก (ส่วนสำคัญที่สุด)
│   └── main.dart               # Entry point ของแอป
├── test/                       # Unit tests
│   └── widget_test.dart
├── web/                        # Web-specific code
│   ├── index.html
│   └── manifest.json
├── windows/                    # Windows-specific code
├── linux/                      # Linux-specific code
├── macos/                      # macOS-specific code
├── pubspec.yaml                # ✨ Project configuration & dependencies
├── pubspec.lock                # Lock file สำหรับ dependencies
├── analysis_options.yaml       # Dart analysis settings
└── README.md                   # Project documentation
```

### ไฟล์สำคัญที่ต้องรู้จัก

**1. lib/main.dart** - จุดเริ่มต้นของแอป

```dart
import 'package:flutter/material.dart';

// Entry point ของแอป Flutter
void main() {
  runApp(const MyApp());
}

// Root Widget ของแอป
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const MyHomePage(title: 'Flutter Demo Home Page'),
    );
  }
}
```

**2. pubspec.yaml** - การตั้งค่าโปรเจกต์

```yaml
name: my_first_app
description: "My First Flutter Application"
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.6

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0

flutter:
  uses-material-design: true
  
  # ประกาศ assets (รูปภาพ, fonts, ฯลฯ)
  # assets:
  #   - images/a_dot_burr.jpeg
  
  # ประกาศ custom fonts
  # fonts:
  #   - family: Schyler
  #     fonts:
  #       - asset: fonts/Schyler-Regular.ttf
```

---

## ขั้นตอนที่ 14: รันแอป Flutter บนอุปกรณ์ต่างๆ

### รันบน Android Emulator

```bash
# สร้าง Android Virtual Device (AVD)
# เปิด Android Studio > AVD Manager > Create Virtual Device

# ตรวจสอบ devices ที่มีอยู่
flutter devices

# รันบน Emulator
flutter run

# รันบน device เฉพาะ
flutter run -d emulator-5554
```

### รันบน iOS Simulator (macOS เท่านั้น)

```bash
# เปิด Simulator
open -a Simulator

# หรือรันผ่าน Flutter
flutter run

# เลือก iOS Simulator
flutter run -d "iPhone 15 Pro"
```

### รันบน Web Browser

```bash
# รันบน Chrome
flutter run -d chrome

# รันบน Web Server
flutter run -d web-server

# Build สำหรับ Production
flutter build web
```

### รันบน Desktop

```bash
# Windows
flutter run -d windows

# macOS
flutter run -d macos

# Linux
flutter run -d linux
```

### รันบนอุปกรณ์จริง

```bash
# Android
# 1. เปิด Developer Options บนโทรศัพท์
# 2. เปิด USB Debugging
# 3. เชื่อมต่อ USB

# ตรวจสอบว่าเห็น device
flutter devices

# รัน
flutter run

# iOS (ต้องมี Apple Developer Account)
# 1. เชื่อมต่อ iPhone
# 2. Trust computer บนโทรศัพท์
# 3. รัน
flutter run
```

---

## ขั้นตอนที่ 15: Hot Reload และ Hot Restart

หนึ่งในฟีเจอร์ที่ดีที่สุดของ Flutter คือ **Hot Reload** ซึ่งช่วยให้เห็นการเปลี่ยนแปลงทันที

### Hot Reload (r)

```bash
# ขณะที่แอปกำลังรัน กด r ใน terminal
# หรือ กด Ctrl+S ใน VS Code (เซฟไฟล์)

# Hot Reload จะ:
# ✅ อัพเดต UI ทันที
# ✅ รักษา State ของแอปไว้
# ✅ เร็วมาก (ประมาณ 1 วินาที)
# ❌ ไม่ reset state
# ❌ ไม่ reload main()
```

### Hot Restart (R - capital R)

```bash
# ขณะที่แอปกำลังรัน กด R (capital) ใน terminal
# หรือ กดปุ่ม restart ใน IDE

# Hot Restart จะ:
# ✅ Restart แอปทั้งหมด
# ✅ Reset state ทั้งหมด
# ✅ เร็วกว่า Full Restart
# ❌ ช้ากว่า Hot Reload
```

### เมื่อไหรต้องใช้ Full Restart?

```bash
# Full Restart ใช้เมื่อ:
# - เปลี่ยน main() function
# - เปลี่ยน pubspec.yaml
# - เพิ่ม Native code
# - เปลี่ยน Assets

# วิธี Full Restart
# หยุดแอป (Ctrl+C) แล้วรันใหม่
flutter run
```

---

## ขั้นตอนที่ 16: ทำความเข้าใจ Flutter App ตัวอย่าง

มาทำความเข้าใจโค้ดตัวอย่างที่ Flutter สร้างให้:

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(const MyApp());
}

// StatelessWidget: Widget ที่ไม่มี State (ไม่เปลี่ยนแปลง)
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    // MaterialApp: Root widget สำหรับ Material Design
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const MyHomePage(title: 'Flutter Demo Home Page'),
    );
  }
}

// StatefulWidget: Widget ที่มี State (เปลี่ยนแปลงได้)
class MyHomePage extends StatefulWidget {
  const MyHomePage({super.key, required this.title});
  final String title;

  @override
  State<MyHomePage> createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;  // State variable

  void _incrementCounter() {
    setState(() {        // บอกให้ Flutter รู้ว่า State เปลี่ยนแปลง
      _counter++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            const Text('You have pushed the button this many times:'),
            Text(
              '$_counter',
              style: Theme.of(context).textTheme.headlineMedium,
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _incrementCounter,
        tooltip: 'Increment',
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

### อธิบายโค้ด

```
🔷 main()
   - Entry point ของแอป
   - เรียก runApp() เพื่อเริ่มแอป

🔷 MyApp (StatelessWidget)
   - Widget ที่ไม่มี State
   - ไม่เปลี่ยนแปลงหลังจาก build ครั้งแรก

🔷 MaterialApp
   - Root Widget สำหรับ Material Design
   - ตั้งค่า Theme, Navigation, Locale

🔷 MyHomePage (StatefulWidget)
   - Widget ที่มี State
   - สามารถเปลี่ยนแปลงได้เมื่อ setState() ถูกเรียก

🔷 _MyHomePageState
   - Class ที่เก็บ State ของ MyHomePage
   - มี _counter เป็น State variable

🔷 Scaffold
   - โครงสร้างพื้นฐานของหน้าจอ
   - มี AppBar, Body, FloatingActionButton
```

---

## ขั้นตอนที่ 17: แก้ไขแอปตัวอย่างครั้งแรก

ลองแก้ไขแอปให้แสดงข้อความภาษาไทย:

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'แอปแรกของฉัน',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
        fontFamily: 'Sarabun', // font ภาษาไทย (ต้องเพิ่มใน pubspec.yaml)
      ),
      home: const MyHomePage(title: 'สวัสดี Flutter!'),
    );
  }
}

class MyHomePage extends StatefulWidget {
  const MyHomePage({super.key, required this.title});
  final String title;

  @override
  State<MyHomePage> createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;

  void _incrementCounter() {
    setState(() {
      _counter++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        backgroundColor: Colors.blue,
        foregroundColor: Colors.white,
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            const Icon(
              Icons.touch_app,
              size: 80,
              color: Colors.blue,
            ),
            const SizedBox(height: 20),
            const Text(
              'คุณกดปุ่มไปแล้ว:',
              style: TextStyle(fontSize: 18),
            ),
            Text(
              '$_counter ครั้ง',
              style: Theme.of(context).textTheme.headlineLarge?.copyWith(
                color: Colors.blue,
                fontWeight: FontWeight.bold,
              ),
            ),
            const SizedBox(height: 20),
            if (_counter > 0)
              Text(
                'เยี่ยมมาก! 🎉',
                style: TextStyle(
                  fontSize: 24,
                  color: Colors.green[700],
                ),
              ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: _incrementCounter,
        label: const Text('กดฉัน'),
        icon: const Icon(Icons.add),
        backgroundColor: Colors.blue,
        foregroundColor: Colors.white,
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 18: เรียนรู้ Flutter DevTools

**Flutter DevTools** เป็นชุดเครื่องมือสำหรับ Debug และ Profile แอป Flutter

### เปิด DevTools

```bash
# วิธีที่ 1: เปิดจาก terminal
flutter pub global activate devtools
flutter pub global run devtools

# วิธีที่ 2: เปิดขณะแอปกำลังรัน
# ขณะรัน flutter run จะเห็น URL เช่น:
# An Observatory debugger and profiler is available at: http://127.0.0.1:59813/

# วิธีที่ 3: เปิดจาก VS Code
# กด Ctrl+Shift+P > Flutter: Open DevTools
```

### เครื่องมือใน DevTools

```
🔧 Widget Inspector
   - ดู Widget tree ทั้งหมด
   - ตรวจสอบ Properties ของแต่ละ Widget
   - Debug Layout ปัญหา

⚡ Performance
   - วัด FPS (Frames Per Second)
   - ค้นหา Jank (การ lag)
   - วิเคราะห์ Rendering performance

🧠 Memory
   - ตรวจสอบการใช้ Memory
   - ค้นหา Memory Leak

📡 Network
   - ตรวจสอบ HTTP requests
   - ดู Response data

📝 Logging
   - ดู logs ทั้งหมด
   - Filter logs

🐞 Debugger
   - Set Breakpoints
   - Step through code
```

---

## ขั้นตอนที่ 19: คำสั่ง Flutter ที่ใช้บ่อย

```bash
# ─────────────────── Project Management ───────────────────
# สร้างโปรเจกต์ใหม่
flutter create <project_name>

# สร้างโปรเจกต์พร้อม options
flutter create --org com.company --platforms android,ios,web <project_name>

# ─────────────────── Running ───────────────────
# รันแอป
flutter run

# รันบน device เฉพาะ
flutter run -d <device_id>

# รันใน release mode
flutter run --release

# ─────────────────── Building ───────────────────
# Build APK (Android)
flutter build apk
flutter build apk --split-per-abi  # แยก APK ตาม architecture

# Build App Bundle (Android - สำหรับ Play Store)
flutter build appbundle

# Build IPA (iOS)
flutter build ipa

# Build Web
flutter build web

# Build Desktop
flutter build windows
flutter build macos
flutter build linux

# ─────────────────── Dependencies ───────────────────
# ดึง packages ตาม pubspec.yaml
flutter pub get

# อัพเดต packages
flutter pub upgrade

# เพิ่ม package
flutter pub add <package_name>

# ─────────────────── Testing ───────────────────
# รัน tests ทั้งหมด
flutter test

# รัน test เฉพาะไฟล์
flutter test test/widget_test.dart

# ─────────────────── Code Quality ───────────────────
# Format code
dart format lib/

# Analyze code
flutter analyze

# Fix issues อัตโนมัติ
dart fix --apply

# ─────────────────── Devices ───────────────────
# แสดง devices ที่เชื่อมต่อ
flutter devices

# แสดง emulators ที่มีอยู่
flutter emulators

# เปิด emulator
flutter emulators --launch <emulator_id>

# ─────────────────── Info ───────────────────
# ตรวจสอบ Flutter version
flutter --version

# ตรวจสอบ environment
flutter doctor

# อัพเดต Flutter
flutter upgrade

# เปลี่ยน channel
flutter channel stable
flutter channel beta
flutter channel master
```

---

## ขั้นตอนที่ 20: สรุปและ Exercise

### สรุป Part 01

ในส่วนนี้คุณได้เรียนรู้:

- ✅ Dart คืออะไรและมีลักษณะเฉพาะอย่างไร
- ✅ Flutter คืออะไรและรองรับแพลตฟอร์มอะไรบ้าง
- ✅ วิธีติดตั้ง Flutter SDK บน Windows, macOS, Linux
- ✅ การตั้งค่า VS Code และ Android Studio
- ✅ การสร้างและรันโปรเจกต์ Flutter แรก
- ✅ โครงสร้างโปรเจกต์ Flutter
- ✅ Hot Reload และ Hot Restart
- ✅ Flutter DevTools เบื้องต้น
- ✅ คำสั่ง Flutter ที่ใช้บ่อย

### Exercise

**แบบฝึกหัดที่ 1**: ติดตั้ง Flutter และรันแอปตัวอย่าง
```
1. ติดตั้ง Flutter SDK บนคอมพิวเตอร์ของคุณ
2. รัน flutter doctor ให้ผ่านทุกหัวข้อ
3. สร้างโปรเจกต์ใหม่ชื่อ "hello_flutter"
4. รันแอปบน Emulator หรือ Browser
5. ทดสอบ Hot Reload โดยเปลี่ยนข้อความ
```

**แบบฝึกหัดที่ 2**: แก้ไขแอปตัวอย่าง
```
1. เปลี่ยนสีธีมของแอปเป็นสีที่คุณชอบ
2. เปลี่ยนข้อความทั้งหมดเป็นภาษาไทย
3. เพิ่ม Icon ใน AppBar
4. เปลี่ยนจาก FAB เป็นปุ่มปกติ
5. เพิ่มข้อความแสดงผลเมื่อกดครบ 10 ครั้ง
```

**แบบฝึกหัดที่ 3 (ท้าทาย)**: สร้างแอปนับถอยหลัง
```
1. สร้างแอปที่เริ่มต้นที่เลข 10
2. มีปุ่มลดจำนวน
3. เมื่อถึง 0 ให้แสดงข้อความ "หมดเวลา!"
4. มีปุ่ม Reset กลับไปที่ 10
```

---

## 🔗 แหล่งข้อมูลเพิ่มเติม

- [Flutter Official Documentation](https://flutter.dev/docs)
- [Dart Official Documentation](https://dart.dev/guides)
- [Flutter Widget Catalog](https://flutter.dev/widgets)
- [pub.dev - Flutter Packages](https://pub.dev)
- [Flutter YouTube Channel](https://www.youtube.com/@flutterdev)
- [DartPad - Online Dart Editor](https://dartpad.dev)

---

**ต่อไป: [Part 02 - ตัวแปร ชนิดข้อมูล และ Operators →](part-02-variables-types-operators.md)**
