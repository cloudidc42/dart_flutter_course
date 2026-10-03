# Part 55: Internationalization (i18n)
## ขั้นตอนที่ 2081-2120

## 🎯 เป้าหมายของ Part นี้
- Setup flutter_localizations และ intl package
- สร้างและใช้งาน ARB files
- Pluralization และ Gender-based translations
- RTL (Right-to-Left) layout support
- Dynamic locale switching ขณะ runtime
- Date, Number และ Currency formatting

---

## ขั้นตอนที่ 2081: Setup pubspec.yaml และ l10n.yaml

```yaml
# pubspec.yaml
name: flutter_i18n_demo
description: Full internationalization demo

dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter
  intl: ^0.19.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0

flutter:
  uses-material-design: true
  generate: true  # สำคัญ! เปิดใช้ code generation สำหรับ localizations
```

```yaml
# l10n.yaml (วางที่ root ของ project)
arb-dir: lib/l10n
template-arb-file: app_en.arb
output-localization-file: app_localizations.dart
output-class: AppLocalizations
nullable-getter: false
```

---

## ขั้นตอนที่ 2082: ARB Files

```json
// lib/l10n/app_en.arb
{
  "@@locale": "en",

  "appTitle": "Flutter i18n Demo",
  "@appTitle": {
    "description": "The title of the application"
  },

  "welcomeMessage": "Welcome, {name}!",
  "@welcomeMessage": {
    "description": "Welcome message with user name",
    "placeholders": {
      "name": {
        "type": "String",
        "example": "Alice"
      }
    }
  },

  "itemCount": "{count, plural, =0{No items} =1{One item} other{{count} items}}",
  "@itemCount": {
    "description": "Item count with pluralization",
    "placeholders": {
      "count": {
        "type": "num",
        "format": "compact"
      }
    }
  },

  "unreadMessages": "{count, plural, =0{No unread messages} =1{1 unread message} other{{count} unread messages}}",
  "@unreadMessages": {
    "placeholders": {
      "count": {"type": "num"}
    }
  },

  "lastSeen": "Last seen {date}",
  "@lastSeen": {
    "placeholders": {
      "date": {
        "type": "DateTime",
        "format": "yMMMd",
        "isCustomDateFormat": false
      }
    }
  },

  "price": "Price: {amount}",
  "@price": {
    "placeholders": {
      "amount": {
        "type": "double",
        "format": "currency",
        "optionalParameters": {
          "symbol": "$",
          "decimalDigits": 2
        }
      }
    }
  },

  "loginButton": "Log In",
  "logoutButton": "Log Out",
  "signUpButton": "Sign Up",
  "cancelButton": "Cancel",
  "saveButton": "Save",
  "deleteButton": "Delete",
  "confirmButton": "Confirm",

  "homeTab": "Home",
  "profileTab": "Profile",
  "settingsTab": "Settings",
  "notificationsTab": "Notifications",

  "settingsTitle": "Settings",
  "languageLabel": "Language",
  "themeLabel": "Theme",
  "notificationsLabel": "Notifications",
  "privacyLabel": "Privacy",
  "aboutLabel": "About",

  "emailLabel": "Email",
  "passwordLabel": "Password",
  "emailHint": "Enter your email",
  "passwordHint": "Enter your password",
  "emailRequired": "Email is required",
  "invalidEmail": "Please enter a valid email",

  "errorGeneral": "Something went wrong. Please try again.",
  "errorNetwork": "Network error. Check your connection.",
  "successMessage": "Operation completed successfully!"
}
```

```json
// lib/l10n/app_th.arb
{
  "@@locale": "th",

  "appTitle": "แอปสาธิต i18n",
  "welcomeMessage": "ยินดีต้อนรับ, {name}!",
  "@welcomeMessage": {
    "placeholders": {
      "name": {"type": "String"}
    }
  },

  "itemCount": "{count, plural, =0{ไม่มีรายการ} =1{หนึ่งรายการ} other{{count} รายการ}}",
  "@itemCount": {
    "placeholders": {
      "count": {"type": "num", "format": "compact"}
    }
  },

  "unreadMessages": "{count, plural, =0{ไม่มีข้อความที่ยังไม่ได้อ่าน} =1{ข้อความที่ยังไม่ได้อ่าน 1 ข้อความ} other{ข้อความที่ยังไม่ได้อ่าน {count} ข้อความ}}",
  "@unreadMessages": {
    "placeholders": {
      "count": {"type": "num"}
    }
  },

  "lastSeen": "เข้าใช้งานล่าสุด {date}",
  "@lastSeen": {
    "placeholders": {
      "date": {"type": "DateTime", "format": "yMMMd"}
    }
  },

  "price": "ราคา: {amount}",
  "@price": {
    "placeholders": {
      "amount": {
        "type": "double",
        "format": "currency",
        "optionalParameters": {"symbol": "฿", "decimalDigits": 2}
      }
    }
  },

  "loginButton": "เข้าสู่ระบบ",
  "logoutButton": "ออกจากระบบ",
  "signUpButton": "สมัครสมาชิก",
  "cancelButton": "ยกเลิก",
  "saveButton": "บันทึก",
  "deleteButton": "ลบ",
  "confirmButton": "ยืนยัน",

  "homeTab": "หน้าหลัก",
  "profileTab": "โปรไฟล์",
  "settingsTab": "การตั้งค่า",
  "notificationsTab": "การแจ้งเตือน",

  "settingsTitle": "การตั้งค่า",
  "languageLabel": "ภาษา",
  "themeLabel": "ธีม",
  "notificationsLabel": "การแจ้งเตือน",
  "privacyLabel": "ความเป็นส่วนตัว",
  "aboutLabel": "เกี่ยวกับ",

  "emailLabel": "อีเมล",
  "passwordLabel": "รหัสผ่าน",
  "emailHint": "กรอกอีเมลของคุณ",
  "passwordHint": "กรอกรหัสผ่านของคุณ",
  "emailRequired": "กรุณากรอกอีเมล",
  "invalidEmail": "กรุณากรอกอีเมลที่ถูกต้อง",

  "errorGeneral": "เกิดข้อผิดพลาด กรุณาลองอีกครั้ง",
  "errorNetwork": "ข้อผิดพลาดเครือข่าย ตรวจสอบการเชื่อมต่อ",
  "successMessage": "ดำเนินการสำเร็จ!"
}
```

```json
// lib/l10n/app_ar.arb
{
  "@@locale": "ar",

  "appTitle": "تطبيق i18n",
  "welcomeMessage": "مرحباً, {name}!",
  "@welcomeMessage": {
    "placeholders": {
      "name": {"type": "String"}
    }
  },

  "itemCount": "{count, plural, =0{لا توجد عناصر} =1{عنصر واحد} =2{عنصران} few{{count} عناصر} many{{count} عنصراً} other{{count} عنصر}}",
  "@itemCount": {
    "placeholders": {
      "count": {"type": "num", "format": "compact"}
    }
  },

  "unreadMessages": "{count, plural, =0{لا رسائل غير مقروءة} =1{رسالة غير مقروءة واحدة} other{{count} رسائل غير مقروءة}}",
  "@unreadMessages": {
    "placeholders": {
      "count": {"type": "num"}
    }
  },

  "lastSeen": "آخر ظهور {date}",
  "@lastSeen": {
    "placeholders": {
      "date": {"type": "DateTime", "format": "yMMMd"}
    }
  },

  "price": "السعر: {amount}",
  "@price": {
    "placeholders": {
      "amount": {
        "type": "double",
        "format": "currency",
        "optionalParameters": {"symbol": "ر.س", "decimalDigits": 2}
      }
    }
  },

  "loginButton": "تسجيل الدخول",
  "logoutButton": "تسجيل الخروج",
  "signUpButton": "إنشاء حساب",
  "cancelButton": "إلغاء",
  "saveButton": "حفظ",
  "deleteButton": "حذف",
  "confirmButton": "تأكيد",

  "homeTab": "الرئيسية",
  "profileTab": "الملف الشخصي",
  "settingsTab": "الإعدادات",
  "notificationsTab": "الإشعارات",

  "settingsTitle": "الإعدادات",
  "languageLabel": "اللغة",
  "themeLabel": "المظهر",
  "notificationsLabel": "الإشعارات",
  "privacyLabel": "الخصوصية",
  "aboutLabel": "حول التطبيق",

  "emailLabel": "البريد الإلكتروني",
  "passwordLabel": "كلمة المرور",
  "emailHint": "أدخل بريدك الإلكتروني",
  "passwordHint": "أدخل كلمة مرورك",
  "emailRequired": "البريد الإلكتروني مطلوب",
  "invalidEmail": "يرجى إدخال بريد إلكتروني صحيح",

  "errorGeneral": "حدث خطأ. يرجى المحاولة مرة أخرى.",
  "errorNetwork": "خطأ في الشبكة. تحقق من اتصالك.",
  "successMessage": "تمت العملية بنجاح!"
}
```

---

## ขั้นตอนที่ 2083: LocaleNotifier — Dynamic Locale Switching

```dart
// lib/i18n/locale_notifier.dart
import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';

class LocaleNotifier extends ChangeNotifier {
  LocaleNotifier() {
    _loadSavedLocale();
  }

  Locale _locale = const Locale('en');
  Locale get locale => _locale;

  static const _key = 'app_locale';

  static const supportedLocales = [
    Locale('en'),
    Locale('th'),
    Locale('ar'),
    Locale('ja'),
    Locale('zh'),
    Locale('fr'),
    Locale('de'),
    Locale('es'),
  ];

  static const localeNames = {
    'en': 'English',
    'th': 'ภาษาไทย',
    'ar': 'العربية',
    'ja': '日本語',
    'zh': '中文',
    'fr': 'Français',
    'de': 'Deutsch',
    'es': 'Español',
  };

  static const localeFlags = {
    'en': '🇺🇸',
    'th': '🇹🇭',
    'ar': '🇸🇦',
    'ja': '🇯🇵',
    'zh': '🇨🇳',
    'fr': '🇫🇷',
    'de': '🇩🇪',
    'es': '🇪🇸',
  };

  Future<void> _loadSavedLocale() async {
    final prefs = await SharedPreferences.getInstance();
    final saved = prefs.getString(_key);
    if (saved != null) {
      _locale = Locale(saved);
      notifyListeners();
    }
  }

  Future<void> setLocale(Locale locale) async {
    if (_locale == locale) return;
    _locale = locale;
    notifyListeners();
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(_key, locale.languageCode);
  }

  String get localeName => localeNames[_locale.languageCode] ?? _locale.languageCode;
  String get localeFlag => localeFlags[_locale.languageCode] ?? '🌐';

  bool get isRtl => _isRtlLocale(_locale);

  static bool _isRtlLocale(Locale locale) {
    const rtlCodes = {'ar', 'he', 'fa', 'ur'};
    return rtlCodes.contains(locale.languageCode);
  }
}
```

---

## ขั้นตอนที่ 2084: Main App with Localization

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';
import 'i18n/locale_notifier.dart';
import 'screens/home_screen.dart';
import 'screens/settings_screen.dart';

void main() => runApp(const I18nDemoApp());

class I18nDemoApp extends StatefulWidget {
  const I18nDemoApp({super.key});

  @override
  State<I18nDemoApp> createState() => _I18nDemoAppState();
}

class _I18nDemoAppState extends State<I18nDemoApp> {
  final _localeNotifier = LocaleNotifier();

  @override
  void dispose() {
    _localeNotifier.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: _localeNotifier,
      builder: (ctx, _) => MaterialApp(
        debugShowCheckedModeBanner: false,

        // ── Localization delegates ────────────────────────────────────────
        localizationsDelegates: const [
          AppLocalizations.delegate,
          GlobalMaterialLocalizations.delegate,
          GlobalWidgetsLocalizations.delegate,
          GlobalCupertinoLocalizations.delegate,
        ],
        supportedLocales: LocaleNotifier.supportedLocales,

        // ── Active locale ─────────────────────────────────────────────────
        locale: _localeNotifier.locale,

        // ── RTL support ───────────────────────────────────────────────────
        // Flutter จัดการ RTL อัตโนมัติตาม locale

        title: 'i18n Demo',
        theme: ThemeData(
          colorSchemeSeed: Colors.indigo,
          useMaterial3: true,
        ),
        home: AppShell(localeNotifier: _localeNotifier),
      ),
    );
  }
}

class AppShell extends StatefulWidget {
  const AppShell({super.key, required this.localeNotifier});
  final LocaleNotifier localeNotifier;

  @override
  State<AppShell> createState() => _AppShellState();
}

class _AppShellState extends State<AppShell> {
  int _selectedIndex = 0;

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context);
    final pages = [
      HomeScreen(localeNotifier: widget.localeNotifier),
      SettingsScreen(localeNotifier: widget.localeNotifier),
    ];

    return Scaffold(
      body: pages[_selectedIndex],
      bottomNavigationBar: NavigationBar(
        selectedIndex: _selectedIndex,
        onDestinationSelected: (i) => setState(() => _selectedIndex = i),
        destinations: [
          NavigationDestination(
            icon: const Icon(Icons.home_outlined),
            selectedIcon: const Icon(Icons.home),
            label: l10n.homeTab,
          ),
          NavigationDestination(
            icon: const Icon(Icons.settings_outlined),
            selectedIcon: const Icon(Icons.settings),
            label: l10n.settingsTab,
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2085: Home Screen — แสดงการใช้งาน Localizations

```dart
// lib/screens/home_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';
import '../i18n/locale_notifier.dart';

class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key, required this.localeNotifier});
  final LocaleNotifier localeNotifier;

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  int _itemCount = 0;
  int _messageCount = 0;
  final String _userName = 'Alice';

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context);

    return Scaffold(
      appBar: AppBar(title: Text(l10n.appTitle)),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // Welcome card
            _WelcomeCard(name: _userName),
            const SizedBox(height: 16),

            // Pluralization demo
            _SectionHeader(l10n.itemCount(_itemCount)),
            Card(
              child: Column(
                children: [
                  ListTile(
                    title: Text(l10n.itemCount(_itemCount)),
                    trailing: Row(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        IconButton(
                          icon: const Icon(Icons.remove),
                          onPressed: _itemCount > 0
                              ? () => setState(() => _itemCount--)
                              : null,
                        ),
                        Text('$_itemCount', style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
                        IconButton(
                          icon: const Icon(Icons.add),
                          onPressed: () => setState(() => _itemCount++),
                        ),
                      ],
                    ),
                  ),
                  ListTile(
                    title: Text(l10n.unreadMessages(_messageCount)),
                    trailing: Row(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        IconButton(
                          icon: const Icon(Icons.remove),
                          onPressed: _messageCount > 0
                              ? () => setState(() => _messageCount--)
                              : null,
                        ),
                        Text('$_messageCount', style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
                        IconButton(
                          icon: const Icon(Icons.add),
                          onPressed: () => setState(() => _messageCount++),
                        ),
                      ],
                    ),
                  ),
                ],
              ),
            ),
            const SizedBox(height: 16),

            // Date formatting
            _SectionHeader('Date & Time Formatting'),
            _DateFormatCard(),
            const SizedBox(height: 16),

            // Number formatting
            _SectionHeader('Number & Currency Formatting'),
            _NumberFormatCard(),
            const SizedBox(height: 16),

            // Buttons
            _SectionHeader('Localized Buttons'),
            Wrap(
              spacing: 8,
              runSpacing: 8,
              children: [
                ElevatedButton(onPressed: () {}, child: Text(l10n.loginButton)),
                ElevatedButton(onPressed: () {}, child: Text(l10n.signUpButton)),
                OutlinedButton(onPressed: () {}, child: Text(l10n.cancelButton)),
                FilledButton(onPressed: () {}, child: Text(l10n.saveButton)),
              ],
            ),
            const SizedBox(height: 16),

            // RTL indicator
            _RTLIndicatorCard(localeNotifier: widget.localeNotifier),
          ],
        ),
      ),
    );
  }
}

class _WelcomeCard extends StatelessWidget {
  const _WelcomeCard({required this.name});
  final String name;

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context);
    return Card(
      color: Theme.of(context).colorScheme.primaryContainer,
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Row(
          children: [
            CircleAvatar(
              radius: 28,
              backgroundColor: Theme.of(context).colorScheme.primary,
              child: Text(
                name[0],
                style: TextStyle(
                  fontSize: 24,
                  color: Theme.of(context).colorScheme.onPrimary,
                  fontWeight: FontWeight.bold,
                ),
              ),
            ),
            const SizedBox(width: 16),
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    l10n.welcomeMessage(name),
                    style: const TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
                  ),
                  Text(
                    l10n.lastSeen(DateTime.now()),
                    style: const TextStyle(fontSize: 13),
                  ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _SectionHeader extends StatelessWidget {
  const _SectionHeader(this.title);
  final String title;

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.only(bottom: 8),
      child: Text(
        title.toUpperCase(),
        style: TextStyle(
          fontSize: 11,
          fontWeight: FontWeight.bold,
          color: Colors.grey.shade600,
          letterSpacing: 1.2,
        ),
      ),
    );
  }
}

class _DateFormatCard extends StatelessWidget {
  const _DateFormatCard();

  @override
  Widget build(BuildContext context) {
    final locale = Localizations.localeOf(context).languageCode;
    final now = DateTime.now();

    // ใช้ intl package สำหรับ formatting
    final formats = {
      'yMMMd': _formatDate(now, 'yMMMd', locale),
      'yMd': _formatDate(now, 'yMd', locale),
      'EEEE': _formatDate(now, 'EEEE', locale),
      'yMMMMEEEEd': _formatDate(now, 'yMMMMEEEEd', locale),
      'Hm': _formatDate(now, 'Hm', locale),
      'jm': _formatDate(now, 'jm', locale),
    };

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Column(
          children: formats.entries
              .map((e) => Padding(
                    padding: const EdgeInsets.symmetric(vertical: 4),
                    child: Row(
                      children: [
                        Container(
                          padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
                          decoration: BoxDecoration(
                            color: Colors.grey.shade100,
                            borderRadius: BorderRadius.circular(4),
                          ),
                          child: Text(e.key, style: const TextStyle(fontSize: 11, fontFamily: 'monospace')),
                        ),
                        const Spacer(),
                        Text(e.value, style: const TextStyle(fontWeight: FontWeight.w500)),
                      ],
                    ),
                  ))
              .toList(),
        ),
      ),
    );
  }

  String _formatDate(DateTime dt, String format, String locale) {
    try {
      // จำลองการ format (ใน production จะใช้ intl.DateFormat)
      switch (format) {
        case 'yMMMd':
          final months = const {
            'en': ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'],
            'th': ['ม.ค.', 'ก.พ.', 'มี.ค.', 'เม.ย.', 'พ.ค.', 'มิ.ย.', 'ก.ค.', 'ส.ค.', 'ก.ย.', 'ต.ค.', 'พ.ย.', 'ธ.ค.'],
          };
          final m = (months[locale] ?? months['en']!)[(dt.month - 1)];
          return '$m ${dt.day}, ${dt.year}';
        case 'yMd':
          return '${dt.month}/${dt.day}/${dt.year}';
        case 'EEEE':
          final days = const {
            'en': ['Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday'],
            'th': ['จันทร์', 'อังคาร', 'พุธ', 'พฤหัสบดี', 'ศุกร์', 'เสาร์', 'อาทิตย์'],
          };
          return (days[locale] ?? days['en']!)[(dt.weekday - 1)];
        case 'Hm':
          return '${dt.hour.toString().padLeft(2, '0')}:${dt.minute.toString().padLeft(2, '0')}';
        default:
          return dt.toString().substring(0, 16);
      }
    } catch (_) {
      return dt.toString();
    }
  }
}

class _NumberFormatCard extends StatelessWidget {
  const _NumberFormatCard();

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context);
    final value = 1234567.89;

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Column(
          children: [
            _Row('Currency', l10n.price(value)),
            _Row('Integer', _formatInt(1234567)),
            _Row('Decimal', _formatDecimal(value)),
            _Row('Percent', _formatPercent(0.756)),
            _Row('Compact', _formatCompact(1500000)),
          ],
        ),
      ),
    );
  }

  Widget _Row(String label, String value) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        children: [
          Text(label, style: const TextStyle(color: Colors.grey)),
          const Spacer(),
          Text(value, style: const TextStyle(fontWeight: FontWeight.w600)),
        ],
      ),
    );
  }

  String _formatInt(int n) => n.toString().replaceAllMapped(
      RegExp(r'(\d{1,3})(?=(\d{3})+(?!\d))'), (m) => '${m[1]},');

  String _formatDecimal(double n) => n.toStringAsFixed(2)
      .replaceAllMapped(RegExp(r'(\d{1,3})(?=(\d{3})+\.)'), (m) => '${m[1]},');

  String _formatPercent(double n) => '${(n * 100).toStringAsFixed(1)}%';

  String _formatCompact(num n) {
    if (n >= 1000000) return '${(n / 1000000).toStringAsFixed(1)}M';
    if (n >= 1000) return '${(n / 1000).toStringAsFixed(1)}K';
    return n.toString();
  }
}

class _RTLIndicatorCard extends StatelessWidget {
  const _RTLIndicatorCard({required this.localeNotifier});
  final LocaleNotifier localeNotifier;

  @override
  Widget build(BuildContext context) {
    final isRtl = Directionality.of(context) == TextDirection.rtl;
    return Card(
      color: isRtl ? Colors.purple.shade50 : Colors.blue.shade50,
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Row(
          children: [
            Icon(isRtl ? Icons.format_textdirection_r_to_l : Icons.format_textdirection_l_to_r,
                color: isRtl ? Colors.purple : Colors.blue),
            const SizedBox(width: 12),
            Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  isRtl ? 'RTL Layout Active' : 'LTR Layout Active',
                  style: TextStyle(
                    fontWeight: FontWeight.bold,
                    color: isRtl ? Colors.purple : Colors.blue,
                  ),
                ),
                Text(
                  'Direction: ${isRtl ? 'Right → Left' : 'Left → Right'}',
                  style: const TextStyle(fontSize: 12),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2086: Settings Screen — Language Switcher

```dart
// lib/screens/settings_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';
import '../i18n/locale_notifier.dart';

class SettingsScreen extends StatelessWidget {
  const SettingsScreen({super.key, required this.localeNotifier});
  final LocaleNotifier localeNotifier;

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context);

    return Scaffold(
      appBar: AppBar(title: Text(l10n.settingsTitle)),
      body: ListView(
        children: [
          // Language picker section
          _SectionHeader(l10n.languageLabel),
          _LanguageSelector(localeNotifier: localeNotifier),
          const Divider(height: 1),

          // Other settings
          _SectionHeader(l10n.themeLabel),
          ListTile(
            leading: const Icon(Icons.dark_mode_outlined),
            title: Text(l10n.themeLabel),
            trailing: Switch(value: false, onChanged: (_) {}),
          ),
          const Divider(height: 1),
          _SectionHeader(l10n.notificationsLabel),
          ListTile(
            leading: const Icon(Icons.notifications_outlined),
            title: Text(l10n.notificationsLabel),
            trailing: Switch(value: true, onChanged: (_) {}),
          ),
          const Divider(height: 1),
          _SectionHeader(l10n.aboutLabel),
          ListTile(
            leading: const Icon(Icons.info_outlined),
            title: Text(l10n.aboutLabel),
            trailing: const Icon(Icons.chevron_right),
            onTap: () {},
          ),
        ],
      ),
    );
  }
}

class _SectionHeader extends StatelessWidget {
  const _SectionHeader(this.title);
  final String title;

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.fromLTRB(16, 16, 16, 4),
      child: Text(
        title.toUpperCase(),
        style: TextStyle(
          fontSize: 11,
          fontWeight: FontWeight.bold,
          color: Theme.of(context).colorScheme.primary,
          letterSpacing: 1.2,
        ),
      ),
    );
  }
}

class _LanguageSelector extends StatelessWidget {
  const _LanguageSelector({required this.localeNotifier});
  final LocaleNotifier localeNotifier;

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: localeNotifier,
      builder: (ctx, _) {
        return Column(
          children: LocaleNotifier.supportedLocales.map((locale) {
            final code = locale.languageCode;
            final name = LocaleNotifier.localeNames[code] ?? code;
            final flag = LocaleNotifier.localeFlags[code] ?? '🌐';
            final isSelected = localeNotifier.locale == locale;

            return ListTile(
              leading: Text(flag, style: const TextStyle(fontSize: 24)),
              title: Text(name),
              subtitle: Text(code.toUpperCase(), style: const TextStyle(fontSize: 11)),
              trailing: isSelected
                  ? Icon(Icons.check_circle, color: Theme.of(ctx).colorScheme.primary)
                  : null,
              selected: isSelected,
              onTap: () => localeNotifier.setLocale(locale),
            );
          }).toList(),
        );
      },
    );
  }
}
```

---

## ขั้นตอนที่ 2087: RTL Layout Patterns

```dart
// lib/screens/rtl_demo_screen.dart
import 'package:flutter/material.dart';

/// หน้านี้แสดง pattern ที่ถูกต้องสำหรับ RTL + LTR
class RTLDemoScreen extends StatelessWidget {
  const RTLDemoScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final isRtl = Directionality.of(context) == TextDirection.rtl;

    return Scaffold(
      appBar: AppBar(
        title: const Text('RTL Layout Demo'),
        // leading icon จะสลับอัตโนมัติ
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          // ✅ ถูกต้อง: ใช้ start/end แทน left/right
          const _DemoCard(
            title: 'EdgeInsets.symmetric (recommended)',
            good: true,
            child: Card(
              child: Padding(
                // ✅ symmetric ทำงานได้ทั้ง LTR และ RTL
                padding: EdgeInsets.symmetric(horizontal: 16, vertical: 12),
                child: Text('Padding that works in both directions'),
              ),
            ),
          ),
          const SizedBox(height: 16),

          // ✅ ใช้ EdgeInsetsDirectional สำหรับ asymmetric padding
          _DemoCard(
            title: 'EdgeInsetsDirectional.fromSTEB',
            good: true,
            child: Card(
              child: Padding(
                // start = left ใน LTR, right ใน RTL
                padding: const EdgeInsetsDirectional.fromSTEB(24, 12, 8, 12),
                child: Row(
                  children: [
                    const Icon(Icons.star, color: Colors.amber),
                    const SizedBox(width: 8),
                    Text(isRtl ? 'بدء التطبيق' : 'Start of content'),
                  ],
                ),
              ),
            ),
          ),
          const SizedBox(height: 16),

          // ✅ Icons สลับอัตโนมัติเมื่อ RTL
          _DemoCard(
            title: 'Auto-mirrored Icons',
            good: true,
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceAround,
              children: [
                Column(
                  children: [
                    const Icon(Icons.arrow_back, size: 32),
                    Text(isRtl ? '← يعكس' : 'Auto ↓'),
                  ],
                ),
                Column(
                  children: [
                    const Icon(Icons.arrow_forward, size: 32),
                    Text(isRtl ? 'يعكس →' : 'Auto ↓'),
                  ],
                ),
                Column(
                  children: [
                    const Icon(Icons.format_list_bulleted, size: 32),
                    const Text('Auto'),
                  ],
                ),
              ],
            ),
          ),
          const SizedBox(height: 16),

          // ✅ AlignmentDirectional
          _DemoCard(
            title: 'AlignmentDirectional',
            good: true,
            child: Container(
              height: 80,
              decoration: BoxDecoration(
                color: Colors.indigo.shade50,
                borderRadius: BorderRadius.circular(8),
              ),
              child: const Align(
                // start = left ใน LTR, right ใน RTL
                alignment: AlignmentDirectional.centerStart,
                child: Padding(
                  padding: EdgeInsetsDirectional.only(start: 16),
                  child: Text('Aligned to start'),
                ),
              ),
            ),
          ),
          const SizedBox(height: 16),

          // ✅ TextAlign.start
          _DemoCard(
            title: 'TextAlign.start',
            good: true,
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                Container(
                  padding: const EdgeInsets.all(8),
                  color: Colors.green.shade50,
                  child: const Text(
                    'Text aligned to start (LTR: left, RTL: right)',
                    textAlign: TextAlign.start,
                  ),
                ),
                Container(
                  padding: const EdgeInsets.all(8),
                  color: Colors.blue.shade50,
                  child: const Text(
                    'Text aligned to end (LTR: right, RTL: left)',
                    textAlign: TextAlign.end,
                  ),
                ),
              ],
            ),
          ),
          const SizedBox(height: 16),

          // Wrap with explicit Directionality to force LTR (for phone numbers, codes)
          _DemoCard(
            title: 'Force LTR for codes',
            good: true,
            child: Column(
              children: [
                const Text('Phone: ', textAlign: TextAlign.start),
                Directionality(
                  textDirection: TextDirection.ltr, // force LTR for numbers
                  child: Container(
                    padding: const EdgeInsets.all(8),
                    decoration: BoxDecoration(
                      border: Border.all(color: Colors.grey.shade300),
                      borderRadius: BorderRadius.circular(8),
                    ),
                    child: const Text('+66 81 234 5678'),
                  ),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

class _DemoCard extends StatelessWidget {
  const _DemoCard({required this.title, required this.child, this.good = true});
  final String title;
  final Widget child;
  final bool good;

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.stretch,
      children: [
        Row(
          children: [
            Icon(
              good ? Icons.check_circle : Icons.cancel,
              color: good ? Colors.green : Colors.red,
              size: 16,
            ),
            const SizedBox(width: 4),
            Text(
              title,
              style: TextStyle(
                fontSize: 12,
                fontWeight: FontWeight.bold,
                color: good ? Colors.green.shade700 : Colors.red.shade700,
              ),
            ),
          ],
        ),
        const SizedBox(height: 4),
        child,
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 2088: Full i18n App Runner

```dart
// lib/main_i18n_full.dart
import 'package:flutter/material.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'package:flutter_gen/gen_l10n/app_localizations.dart';
import 'i18n/locale_notifier.dart';
import 'screens/home_screen.dart';
import 'screens/settings_screen.dart';
import 'screens/rtl_demo_screen.dart';

void main() => runApp(const FullI18nApp());

class FullI18nApp extends StatefulWidget {
  const FullI18nApp({super.key});

  @override
  State<FullI18nApp> createState() => _FullI18nAppState();
}

class _FullI18nAppState extends State<FullI18nApp> {
  final _localeNotifier = LocaleNotifier();

  @override
  void dispose() {
    _localeNotifier.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: _localeNotifier,
      builder: (ctx, _) {
        return MaterialApp(
          debugShowCheckedModeBanner: false,
          localizationsDelegates: const [
            AppLocalizations.delegate,
            GlobalMaterialLocalizations.delegate,
            GlobalWidgetsLocalizations.delegate,
            GlobalCupertinoLocalizations.delegate,
          ],
          supportedLocales: LocaleNotifier.supportedLocales,
          locale: _localeNotifier.locale,
          theme: ThemeData(colorSchemeSeed: Colors.teal, useMaterial3: true),
          home: FullAppShell(localeNotifier: _localeNotifier),
        );
      },
    );
  }
}

class FullAppShell extends StatefulWidget {
  const FullAppShell({super.key, required this.localeNotifier});
  final LocaleNotifier localeNotifier;

  @override
  State<FullAppShell> createState() => _FullAppShellState();
}

class _FullAppShellState extends State<FullAppShell> {
  int _index = 0;

  @override
  Widget build(BuildContext context) {
    final l10n = AppLocalizations.of(context);
    final pages = [
      HomeScreen(localeNotifier: widget.localeNotifier),
      const RTLDemoScreen(),
      SettingsScreen(localeNotifier: widget.localeNotifier),
    ];

    return Scaffold(
      body: pages[_index],
      bottomNavigationBar: NavigationBar(
        selectedIndex: _index,
        onDestinationSelected: (i) => setState(() => _index = i),
        destinations: [
          NavigationDestination(
            icon: const Icon(Icons.home_outlined),
            selectedIcon: const Icon(Icons.home),
            label: l10n.homeTab,
          ),
          const NavigationDestination(
            icon: Icon(Icons.format_textdirection_l_to_r),
            label: 'RTL Demo',
          ),
          NavigationDestination(
            icon: const Icon(Icons.settings_outlined),
            selectedIcon: const Icon(Icons.settings),
            label: l10n.settingsTab,
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2089: สรุป i18n Best Practices

```dart
// lib/i18n/best_practices.dart
// Best Practices สำหรับ Flutter Internationalization

// ✅ 1. ใช้ AppLocalizations ผ่าน context เสมอ
//    แทน: Text('Hello')
//    ใช้:  Text(AppLocalizations.of(context).hello)

// ✅ 2. ใช้ EdgeInsetsDirectional สำหรับ asymmetric padding
//    แทน: EdgeInsets.only(left: 16)
//    ใช้:  EdgeInsetsDirectional.only(start: 16)

// ✅ 3. ใช้ AlignmentDirectional
//    แทน: Alignment.centerLeft
//    ใช้:  AlignmentDirectional.centerStart

// ✅ 4. ใช้ TextAlign.start และ .end
//    แทน: TextAlign.left / TextAlign.right
//    ใช้:  TextAlign.start / TextAlign.end

// ✅ 5. Icons ที่ต้องการให้ mirror ใน RTL ให้ใช้ icon ที่ auto-mirror
//    เช่น: Icons.arrow_back, Icons.arrow_forward, Icons.chevron_left/right

// ✅ 6. ใช้ Directionality widget เมื่อต้องการ force direction
//    สำหรับ phone numbers, codes, emails ที่ควรเป็น LTR เสมอ:
//    Directionality(textDirection: TextDirection.ltr, child: Text(phoneNumber))

// ✅ 7. ใช้ CrossAxisAlignment.start ใน Column (ไม่ใช่ .end เว้นจำเป็น)

// ✅ 8. Test ด้วยการเปลี่ยน locale ใน debug mode:
//    flutter run --dart-define=LOCALE=ar

// ✅ 9. ตั้ง textScaleFactor ใน test เพื่อเช็ค text overflow:
//    MediaQuery.withNoTextScaling(child: ...)

// ✅ 10. ใช้ semanticsLabel ใน Icon สำหรับ accessibility:
//    Icon(Icons.star, semanticLabel: 'Favorite')

/// ตัวอย่าง LocalizedText widget ที่ safe สำหรับ RTL
class LocalizedText extends StatelessWidget {
  const LocalizedText(this.text, {super.key, this.style, this.maxLines});
  final String text;
  final TextStyle? style;
  final int? maxLines;

  @override
  Widget build(BuildContext context) {
    return Text(
      text,
      style: style,
      maxLines: maxLines,
      overflow: maxLines != null ? TextOverflow.ellipsis : null,
      textAlign: TextAlign.start, // ✅ start ทำงานถูกต้องทั้ง LTR และ RTL
    );
  }
}

/// Widget สำหรับ content ที่ต้อง force LTR เสมอ (เช่น URL, phone, code)
class LtrText extends StatelessWidget {
  const LtrText(this.text, {super.key, this.style});
  final String text;
  final TextStyle? style;

  @override
  Widget build(BuildContext context) {
    return Directionality(
      textDirection: TextDirection.ltr,
      child: Text(text, style: style),
    );
  }
}
```

---

**← [Part 54](part-54-push-notifications-advanced.md)**
**ต่อไป: [Part 56 →](part-56-state-management-advanced.md)**
