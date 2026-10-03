# Part 35: Accessibility (A11y)
## ขั้นตอนที่ 1281-1320

---

## 🎯 เป้าหมายของ Part นี้

- Semantics widget
- Screen reader support
- Color contrast
- Focus management
- Accessible forms

---

## ขั้นตอนที่ 1281: Semantics Widget

```dart
import 'package:flutter/material.dart';

// ─── Basic Semantics ───
class AccessibleButton extends StatelessWidget {
  final String label;
  final VoidCallback onPressed;
  final bool isLoading;

  const AccessibleButton({
    super.key,
    required this.label,
    required this.onPressed,
    this.isLoading = false,
  });

  @override
  Widget build(BuildContext context) {
    return Semantics(
      label: isLoading ? '$label กำลังโหลด' : label,
      button: true,
      enabled: !isLoading,
      child: ElevatedButton(
        onPressed: isLoading ? null : onPressed,
        child: isLoading
            ? Row(
                mainAxisSize: MainAxisSize.min,
                children: [
                  const SizedBox(
                    width: 16,
                    height: 16,
                    child: CircularProgressIndicator(strokeWidth: 2),
                  ),
                  const SizedBox(width: 8),
                  Text(label),
                ],
              )
            : Text(label),
      ),
    );
  }
}

// ─── Accessible Image ───
class AccessibleImage extends StatelessWidget {
  final String imageUrl;
  final String altText;
  final double? width;
  final double? height;

  const AccessibleImage({
    super.key,
    required this.imageUrl,
    required this.altText,
    this.width,
    this.height,
  });

  @override
  Widget build(BuildContext context) {
    return Semantics(
      label: altText,
      image: true,
      child: Image.network(
        imageUrl,
        width: width,
        height: height,
        fit: BoxFit.cover,
        semanticLabel: altText, // alt attribute
      ),
    );
  }
}

// ─── Accessible Icon Button ───
class AccessibleIconButton extends StatelessWidget {
  final IconData icon;
  final String label;
  final VoidCallback onPressed;

  const AccessibleIconButton({
    super.key,
    required this.icon,
    required this.label,
    required this.onPressed,
  });

  @override
  Widget build(BuildContext context) {
    return Semantics(
      label: label,
      button: true,
      child: IconButton(
        icon: Icon(icon),
        tooltip: label,
        onPressed: onPressed,
      ),
    );
  }
}

// ─── Accessible Card ───
class AccessibleProductCard extends StatelessWidget {
  final String name;
  final double price;
  final String imageUrl;
  final VoidCallback onAddToCart;

  const AccessibleProductCard({
    super.key,
    required this.name,
    required this.price,
    required this.imageUrl,
    required this.onAddToCart,
  });

  @override
  Widget build(BuildContext context) {
    return Semantics(
      label: '$name ราคา ฿${price.toStringAsFixed(2)}',
      child: Card(
        child: Column(
          children: [
            ExcludeSemantics(  // รูปไม่ต้องอ่าน ตอนอ่าน card แล้ว
              child: AccessibleImage(
                imageUrl: imageUrl,
                altText: 'ภาพสินค้า $name',
              ),
            ),
            Padding(
              padding: const EdgeInsets.all(8),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(name, style: const TextStyle(fontWeight: FontWeight.bold)),
                  Text('฿${price.toStringAsFixed(2)}'),
                  const SizedBox(height: 8),
                  AccessibleButton(
                    label: 'เพิ่ม $name ลงตะกร้า',
                    onPressed: onAddToCart,
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
```

---

## ขั้นตอนที่ 1282: Focus Management

```dart
import 'package:flutter/material.dart';

// ─── Focus Order ───
class AccessibleForm extends StatefulWidget {
  const AccessibleForm({super.key});

  @override
  State<AccessibleForm> createState() => _AccessibleFormState();
}

class _AccessibleFormState extends State<AccessibleForm> {
  final _formKey = GlobalKey<FormState>();
  final _nameFocus = FocusNode();
  final _emailFocus = FocusNode();
  final _phoneFocus = FocusNode();
  final _passwordFocus = FocusNode();

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      child: Column(
        children: [
          // ชื่อ - focus แรก
          FocusTraversalOrder(
            order: const NumericFocusOrder(1),
            child: TextFormField(
              focusNode: _nameFocus,
              decoration: const InputDecoration(
                labelText: 'ชื่อ-นามสกุล *',
                hintText: 'กรอกชื่อของคุณ',
                helperText: 'ใช้ชื่อจริงตามบัตรประชาชน',
              ),
              textInputAction: TextInputAction.next,
              onFieldSubmitted: (_) => _emailFocus.requestFocus(),
              validator: (v) => v!.isEmpty ? 'กรุณากรอกชื่อ' : null,
            ),
          ),
          const SizedBox(height: 16),

          // Email
          FocusTraversalOrder(
            order: const NumericFocusOrder(2),
            child: TextFormField(
              focusNode: _emailFocus,
              decoration: const InputDecoration(
                labelText: 'Email *',
                hintText: 'example@email.com',
              ),
              keyboardType: TextInputType.emailAddress,
              textInputAction: TextInputAction.next,
              onFieldSubmitted: (_) => _phoneFocus.requestFocus(),
              validator: (v) {
                if (v!.isEmpty) return 'กรุณากรอก email';
                if (!v.contains('@')) return 'email ไม่ถูกต้อง';
                return null;
              },
            ),
          ),
          const SizedBox(height: 16),

          // เบอร์โทร
          FocusTraversalOrder(
            order: const NumericFocusOrder(3),
            child: TextFormField(
              focusNode: _phoneFocus,
              decoration: const InputDecoration(
                labelText: 'เบอร์โทรศัพท์',
                hintText: '0812345678',
                helperText: 'ไม่จำเป็นต้องกรอก',
              ),
              keyboardType: TextInputType.phone,
              textInputAction: TextInputAction.next,
              onFieldSubmitted: (_) => _passwordFocus.requestFocus(),
            ),
          ),
          const SizedBox(height: 16),

          // Password
          FocusTraversalOrder(
            order: const NumericFocusOrder(4),
            child: TextFormField(
              focusNode: _passwordFocus,
              decoration: const InputDecoration(
                labelText: 'รหัสผ่าน *',
                hintText: 'อย่างน้อย 8 ตัวอักษร',
              ),
              obscureText: true,
              textInputAction: TextInputAction.done,
              onFieldSubmitted: (_) => _submit(),
              validator: (v) {
                if (v!.isEmpty) return 'กรุณากรอกรหัสผ่าน';
                if (v.length < 8) return 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
                return null;
              },
            ),
          ),
          const SizedBox(height: 24),

          FocusTraversalOrder(
            order: const NumericFocusOrder(5),
            child: SizedBox(
              width: double.infinity,
              child: ElevatedButton(
                onPressed: _submit,
                child: const Text('สมัครสมาชิก'),
              ),
            ),
          ),
        ],
      ),
    );
  }

  void _submit() {
    if (_formKey.currentState!.validate()) {
      // Process form
    }
  }

  @override
  void dispose() {
    _nameFocus.dispose();
    _emailFocus.dispose();
    _phoneFocus.dispose();
    _passwordFocus.dispose();
    super.dispose();
  }
}
```

---

## ขั้นตอนที่ 1283: Color Contrast

```dart
import 'package:flutter/material.dart';

// ─── WCAG Color Contrast ───
class ContrastChecker {
  // คำนวณ relative luminance
  static double _relativeLuminance(Color color) {
    double r = color.red / 255.0;
    double g = color.green / 255.0;
    double b = color.blue / 255.0;

    double linearize(double c) =>
        c <= 0.04045 ? c / 12.92 : ((c + 0.055) / 1.055).abs().pow(2.4);

    return 0.2126 * linearize(r) + 0.7152 * linearize(g) + 0.0722 * linearize(b);
  }

  // คำนวณ contrast ratio
  static double contrastRatio(Color foreground, Color background) {
    double l1 = _relativeLuminance(foreground);
    double l2 = _relativeLuminance(background);
    double lighter = l1 > l2 ? l1 : l2;
    double darker = l1 > l2 ? l2 : l1;
    return (lighter + 0.05) / (darker + 0.05);
  }

  // WCAG AA: 4.5:1 สำหรับ normal text, 3:1 สำหรับ large text
  static bool isAACompliant(Color fg, Color bg, {bool largeText = false}) {
    double ratio = contrastRatio(fg, bg);
    return largeText ? ratio >= 3.0 : ratio >= 4.5;
  }

  // WCAG AAA: 7:1 สำหรับ normal text, 4.5:1 สำหรับ large text
  static bool isAAACompliant(Color fg, Color bg, {bool largeText = false}) {
    double ratio = contrastRatio(fg, bg);
    return largeText ? ratio >= 4.5 : ratio >= 7.0;
  }
}

extension on double {
  double pow(double exponent) {
    double result = 1.0;
    for (int i = 0; i < exponent; i++) {
      result *= this;
    }
    return result;
  }
}

// ─── Accessible Color Scheme ───
class AccessibleTheme {
  // WCAG AA Compliant colors
  static const Color primaryBlue = Color(0xFF0057B7);   // contrast 7.1:1 on white
  static const Color successGreen = Color(0xFF2E7D32);  // contrast 5.9:1 on white
  static const Color errorRed = Color(0xFFC62828);      // contrast 6.3:1 on white
  static const Color warningOrange = Color(0xFFE65100); // contrast 4.6:1 on white
  static const Color textDark = Color(0xFF212121);      // contrast 16.1:1 on white

  static ThemeData get theme {
    return ThemeData(
      colorScheme: const ColorScheme.light(
        primary: primaryBlue,
        error: errorRed,
        onSurface: textDark,
      ),
      // ขนาด font ที่อ่านง่าย
      textTheme: const TextTheme(
        bodyMedium: TextStyle(fontSize: 16, height: 1.5),
        bodySmall: TextStyle(fontSize: 14, height: 1.5),
        labelMedium: TextStyle(fontSize: 14, fontWeight: FontWeight.w500),
      ),
      // Touch target ต้องมีขนาดอย่างน้อย 48x48
      iconButtonTheme: IconButtonThemeData(
        style: ButtonStyle(
          minimumSize: WidgetStateProperty.all(const Size(48, 48)),
        ),
      ),
    );
  }
}

// ─── Accessible Text Sizing ───
class AccessibleText extends StatelessWidget {
  final String text;
  final TextStyle? style;
  final bool allowScaling;  // ให้ผู้ใช้ scale font ได้

  const AccessibleText(
    this.text, {
    super.key,
    this.style,
    this.allowScaling = true,
  });

  @override
  Widget build(BuildContext context) {
    Widget textWidget = Text(
      text,
      style: style,
      textScaler: allowScaling
          ? MediaQuery.textScalerOf(context)
          : TextScaler.noScaling,
    );

    if (!allowScaling) return textWidget;

    // ให้ scale ได้สูงสุด 2x เพื่อ accessibility
    return MediaQuery(
      data: MediaQuery.of(context).copyWith(
        textScaler: TextScaler.linear(
          MediaQuery.of(context).textScaler.scale(1).clamp(1.0, 2.0),
        ),
      ),
      child: textWidget,
    );
  }
}
```

---

## ขั้นตอนที่ 1284: Live Region & Announcements

```dart
import 'package:flutter/material.dart';
import 'package:flutter/semantics.dart';

// ─── Announce to screen reader ───
void announceToScreenReader(BuildContext context, String message) {
  SemanticsService.announce(message, TextDirection.ltr);
}

// ─── Live Region (updates ที่ screen reader ต้องอ่าน) ───
class LiveRegion extends StatelessWidget {
  final String message;
  const LiveRegion({super.key, required this.message});

  @override
  Widget build(BuildContext context) {
    return Semantics(
      liveRegion: true,
      child: Text(message),
    );
  }
}

// ─── Loading State กับ Accessibility ───
class AccessibleLoadingScreen extends StatefulWidget {
  final Future<List<String>> Function() loadData;
  const AccessibleLoadingScreen({super.key, required this.loadData});

  @override
  State<AccessibleLoadingScreen> createState() => _AccessibleLoadingScreenState();
}

class _AccessibleLoadingScreenState extends State<AccessibleLoadingScreen> {
  bool _isLoading = true;
  List<String> _items = [];
  String? _error;

  @override
  void initState() {
    super.initState();
    _load();
  }

  Future<void> _load() async {
    setState(() {
      _isLoading = true;
      _error = null;
    });
    try {
      List<String> items = await widget.loadData();
      setState(() {
        _items = items;
        _isLoading = false;
      });
      // แจ้ง screen reader
      if (mounted) {
        announceToScreenReader(context, 'โหลดข้อมูลสำเร็จ ${items.length} รายการ');
      }
    } catch (e) {
      setState(() {
        _error = 'เกิดข้อผิดพลาด: $e';
        _isLoading = false;
      });
      if (mounted) {
        announceToScreenReader(context, 'เกิดข้อผิดพลาดในการโหลดข้อมูล');
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return Semantics(
        label: 'กำลังโหลดข้อมูล',
        child: const Center(child: CircularProgressIndicator()),
      );
    }

    if (_error != null) {
      return Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Semantics(
              label: _error!,
              child: Text(_error!, style: const TextStyle(color: Colors.red)),
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: _load,
              child: const Text('ลองใหม่'),
            ),
          ],
        ),
      );
    }

    return ListView.builder(
      itemCount: _items.length,
      itemBuilder: (context, index) => ListTile(
        title: Text(_items[index]),
      ),
    );
  }
}

// ─── Accessible Dialog ───
Future<bool?> showAccessibleConfirmDialog(
  BuildContext context, {
  required String title,
  required String content,
}) {
  return showDialog<bool>(
    context: context,
    builder: (context) => AlertDialog(
      title: Semantics(
        header: true,
        child: Text(title),
      ),
      content: Text(content),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context, false),
          child: const Text('ยกเลิก'),
        ),
        ElevatedButton(
          onPressed: () => Navigator.pop(context, true),
          // autofocus: true ทำให้ screen reader focus ที่ปุ่มนี้เมื่อเปิด dialog
          autofocus: true,
          child: const Text('ยืนยัน'),
        ),
      ],
    ),
  );
}
```

---

**← [Part 34 - Flutter Desktop](part-34-flutter-desktop.md)**

**ต่อไป: [Part 36 - Deep Linking & Push Notifications →](part-36-deep-linking-notifications.md)**
