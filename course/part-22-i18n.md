# Part 22: Internationalization (i18n)
## ขั้นตอนที่ 761-800

---

## 🎯 เป้าหมายของ Part นี้

- Flutter Localization พื้นฐาน
- intl package
- ARB files สำหรับหลายภาษา
- Locale switching
- Number/Date formatting

---

## ขั้นตอนที่ 761: Setup Localization

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  intl: ^0.19.0

flutter:
  generate: true  # สำคัญมาก!
```

```yaml
# l10n.yaml (ที่ root project)
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
```

---

## ขั้นตอนที่ 762: ARB Files

```json
// lib/l10n/app_en.arb
{
  "@@locale": "en",
  
  "appTitle": "My App",
  "@appTitle": {
    "description": "The title of the application"
  },
  
  "helloUser": "Hello, {name}!",
  "@helloUser": {
    "description": "Greeting with user name",
    "placeholders": {
      "name": {
        "type": "String",
        "example": "Alice"
      }
    }
  },
  
  "itemCount": "{count, plural, =0{No items} =1{1 item} other{{count} items}}",
  "@itemCount": {
    "placeholders": {
      "count": {
        "type": "int"
      }
    }
  },
  
  "loginButton": "Login",
  "logoutButton": "Logout",
  "emailLabel": "Email",
  "passwordLabel": "Password",
  "saveButton": "Save",
  "cancelButton": "Cancel",
  "deleteButton": "Delete",
  "confirmDelete": "Are you sure you want to delete this?",
  "yes": "Yes",
  "no": "No"
}
```

```json
// lib/l10n/app_th.arb
{
  "@@locale": "th",
  
  "appTitle": "แอปของฉัน",
  "helloUser": "สวัสดี, {name}!",
  "itemCount": "{count, plural, =0{ไม่มีรายการ} =1{1 รายการ} other{{count} รายการ}}",
  "loginButton": "เข้าสู่ระบบ",
  "logoutButton": "ออกจากระบบ",
  "emailLabel": "อีเมล",
  "passwordLabel": "รหัสผ่าน",
  "saveButton": "บันทึก",
  "cancelButton": "ยกเลิก",
  "deleteButton": "ลบ",
  "confirmDelete": "คุณแน่ใจว่าต้องการลบรายการนี้?",
  "yes": "ใช่",
  "no": "ไม่"
}
```

```json
// lib/l10n/app_ja.arb
{
  "@@locale": "ja",
  
  "appTitle": "マイアプリ",
  "helloUser": "こんにちは、{name}さん!",
  "itemCount": "{count, plural, =0{アイテムなし} =1{1件} other{{count}件}}",
  "loginButton": "ログイン",
  "logoutButton": "ログアウト",
  "emailLabel": "メール",
  "passwordLabel": "パスワード",
  "saveButton": "保存",
  "cancelButton": "キャンセル",
  "deleteButton": "削除",
  "confirmDelete": "本当に削除しますか？",
  "yes": "はい",
  "no": "いいえ"
}
```

---

## ขั้นตอนที่ 763: Setup MaterialApp กับ Localizations

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';

void main() => runApp(const MyApp());

class MyApp extends StatefulWidget {
  const MyApp({super.key});
  
  // Static method เพื่อให้ child เปลี่ยน locale ได้
  static void setLocale(BuildContext context, Locale locale) {
    _MyAppState? state = context.findAncestorStateOfType<_MyAppState>();
    state?._setLocale(locale);
  }
  
  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  Locale _locale = const Locale('th');
  
  void _setLocale(Locale locale) {
    setState(() => _locale = locale);
  }
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Localized App',
      
      // Locale settings
      locale: _locale,
      supportedLocales: AppLocalizations.supportedLocales,
      
      localizationsDelegates: [
        AppLocalizations.delegate,
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      
      home: const HomePage(),
    );
  }
}

// ─── Usage in Widgets ───
class HomePage extends StatelessWidget {
  const HomePage({super.key});
  
  @override
  Widget build(BuildContext context) {
    AppLocalizations l10n = AppLocalizations.of(context)!;
    
    return Scaffold(
      appBar: AppBar(title: Text(l10n.appTitle)),
      body: Column(
        children: [
          Text(l10n.helloUser('สมชาย')),
          Text(l10n.itemCount(0)),
          Text(l10n.itemCount(1)),
          Text(l10n.itemCount(5)),
          
          // Language picker
          DropdownButton<Locale>(
            value: Localizations.localeOf(context),
            items: const [
              DropdownMenuItem(value: Locale('th'), child: Text('ไทย')),
              DropdownMenuItem(value: Locale('en'), child: Text('English')),
              DropdownMenuItem(value: Locale('ja'), child: Text('日本語')),
            ],
            onChanged: (locale) {
              if (locale != null) MyApp.setLocale(context, locale);
            },
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 764: Number & Date Formatting

```dart
import 'package:intl/intl.dart';

class FormatUtils {
  // ─── Currency ───
  static String formatCurrency(double amount, {String locale = 'th_TH'}) {
    NumberFormat formatter = NumberFormat.currency(
      locale: locale,
      symbol: locale == 'th_TH' ? '฿' : '\$',
      decimalDigits: 0,
    );
    return formatter.format(amount);
  }
  
  static String formatCompact(double amount, {String locale = 'th_TH'}) {
    return NumberFormat.compact(locale: locale).format(amount);
  }
  
  static String formatPercent(double value, {String locale = 'th_TH'}) {
    return NumberFormat.percentPattern(locale).format(value);
  }
  
  // ─── Date ───
  static String formatDate(DateTime date, {String locale = 'th'}) {
    return DateFormat.yMMMMd(locale).format(date);
  }
  
  static String formatShortDate(DateTime date, {String locale = 'th'}) {
    return DateFormat.yMd(locale).format(date);
  }
  
  static String formatTime(DateTime date, {String locale = 'th'}) {
    return DateFormat.jm(locale).format(date);
  }
  
  static String formatDateTime(DateTime date, {String locale = 'th'}) {
    return DateFormat.yMMMd(locale).add_jm().format(date);
  }
  
  static String formatRelative(DateTime date) {
    Duration diff = DateTime.now().difference(date);
    
    if (diff.inDays > 365) return '${diff.inDays ~/ 365} ปีที่แล้ว';
    if (diff.inDays > 30) return '${diff.inDays ~/ 30} เดือนที่แล้ว';
    if (diff.inDays > 0) return '${diff.inDays} วันที่แล้ว';
    if (diff.inHours > 0) return '${diff.inHours} ชั่วโมงที่แล้ว';
    if (diff.inMinutes > 0) return '${diff.inMinutes} นาทีที่แล้ว';
    return 'เมื่อกี้';
  }
}

// ─── Usage ───
void main() {
  print(FormatUtils.formatCurrency(39900));          // ฿39,900
  print(FormatUtils.formatCurrency(1500.50, locale: 'en_US'));  // $1,501
  print(FormatUtils.formatCompact(1500000));          // 1.5M
  print(FormatUtils.formatPercent(0.75));             // 75%
  
  DateTime now = DateTime.now();
  print(FormatUtils.formatDate(now));                 // 1 ตุลาคม 2026
  print(FormatUtils.formatTime(now));                 // 10:30 AM
  
  DateTime yesterday = now.subtract(const Duration(days: 1));
  print(FormatUtils.formatRelative(yesterday));       // 1 วันที่แล้ว
}
```

---

## ขั้นตอนที่ 765: Language Preference Persistence

```dart
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';

class LocaleProvider extends ChangeNotifier {
  static const String _key = 'app_locale';
  
  Locale _locale = const Locale('th');
  Locale get locale => _locale;
  
  Future<void> init() async {
    SharedPreferences prefs = await SharedPreferences.getInstance();
    String? savedLocale = prefs.getString(_key);
    if (savedLocale != null) {
      _locale = Locale(savedLocale);
      notifyListeners();
    }
  }
  
  Future<void> setLocale(Locale locale) async {
    _locale = locale;
    SharedPreferences prefs = await SharedPreferences.getInstance();
    await prefs.setString(_key, locale.languageCode);
    notifyListeners();
  }
  
  static List<Map<String, dynamic>> get supportedLanguages => [
    {'code': 'th', 'name': 'ไทย', 'flag': '🇹🇭'},
    {'code': 'en', 'name': 'English', 'flag': '🇺🇸'},
    {'code': 'ja', 'name': '日本語', 'flag': '🇯🇵'},
    {'code': 'zh', 'name': '中文', 'flag': '🇨🇳'},
  ];
}

// ─── Language Selector ───
class LanguageSelector extends StatelessWidget {
  const LanguageSelector({super.key});
  
  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: const Icon(Icons.language),
      title: const Text('ภาษา'),
      trailing: const Icon(Icons.chevron_right),
      onTap: () => showModalBottomSheet(
        context: context,
        builder: (_) => const _LanguageSheet(),
      ),
    );
  }
}

class _LanguageSheet extends StatelessWidget {
  const _LanguageSheet();
  
  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        const Padding(
          padding: EdgeInsets.all(16),
          child: Text('เลือกภาษา', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
        ),
        ...LocaleProvider.supportedLanguages.map((lang) => ListTile(
          leading: Text(lang['flag']!, style: const TextStyle(fontSize: 24)),
          title: Text(lang['name']!),
          onTap: () {
            Navigator.pop(context);
            // context.read<LocaleProvider>().setLocale(Locale(lang['code']!));
          },
        )),
      ],
    );
  }
}
```

---

**← [Part 21 - Custom Widgets](part-21-custom-widgets-themes.md)**

**ต่อไป: [Part 23 - Performance Optimization →](part-23-performance.md)**
