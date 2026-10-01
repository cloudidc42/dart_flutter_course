# Part 21: Custom Widgets & Themes
## ขั้นตอนที่ 721-760

---

## 🎯 เป้าหมายของ Part นี้

- Custom Reusable Widgets
- Theme system (ThemeData, ColorScheme)
- Dark/Light mode
- Custom Typography
- Design System

---

## ขั้นตอนที่ 721: Custom Widget Design

```dart
import 'package:flutter/material.dart';

// ─── Custom Button ───
enum AppButtonVariant { primary, secondary, danger, ghost }

class AppButton extends StatelessWidget {
  final String label;
  final VoidCallback? onPressed;
  final AppButtonVariant variant;
  final bool isLoading;
  final bool isFullWidth;
  final IconData? prefixIcon;
  final double? width;

  const AppButton({
    super.key,
    required this.label,
    this.onPressed,
    this.variant = AppButtonVariant.primary,
    this.isLoading = false,
    this.isFullWidth = false,
    this.prefixIcon,
    this.width,
  });

  @override
  Widget build(BuildContext context) {
    ColorScheme cs = Theme.of(context).colorScheme;

    Color bgColor;
    Color textColor;
    Color borderColor;

    switch (variant) {
      case AppButtonVariant.primary:
        bgColor = cs.primary;
        textColor = cs.onPrimary;
        borderColor = Colors.transparent;
        break;
      case AppButtonVariant.secondary:
        bgColor = cs.secondary;
        textColor = cs.onSecondary;
        borderColor = Colors.transparent;
        break;
      case AppButtonVariant.danger:
        bgColor = cs.error;
        textColor = cs.onError;
        borderColor = Colors.transparent;
        break;
      case AppButtonVariant.ghost:
        bgColor = Colors.transparent;
        textColor = cs.primary;
        borderColor = cs.primary;
        break;
    }

    Widget content = isLoading
        ? SizedBox(
            height: 20,
            width: 20,
            child: CircularProgressIndicator(strokeWidth: 2, color: textColor),
          )
        : Row(
            mainAxisSize: MainAxisSize.min,
            children: [
              if (prefixIcon != null) ...[
                Icon(prefixIcon, size: 18, color: textColor),
                const SizedBox(width: 8),
              ],
              Text(label, style: TextStyle(color: textColor, fontWeight: FontWeight.w600)),
            ],
          );

    Widget button = ElevatedButton(
      onPressed: isLoading ? null : onPressed,
      style: ElevatedButton.styleFrom(
        backgroundColor: bgColor,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(12),
          side: BorderSide(color: borderColor),
        ),
        padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 14),
      ),
      child: content,
    );

    if (isFullWidth) {
      return SizedBox(width: double.infinity, child: button);
    }

    if (width != null) {
      return SizedBox(width: width, child: button);
    }

    return button;
  }
}

// ─── Custom Input Field ───
class AppTextField extends StatefulWidget {
  final String label;
  final String? hint;
  final TextEditingController? controller;
  final String? Function(String?)? validator;
  final bool obscureText;
  final IconData? prefixIcon;
  final TextInputType keyboardType;
  final int maxLines;
  final void Function(String)? onChanged;

  const AppTextField({
    super.key,
    required this.label,
    this.hint,
    this.controller,
    this.validator,
    this.obscureText = false,
    this.prefixIcon,
    this.keyboardType = TextInputType.text,
    this.maxLines = 1,
    this.onChanged,
  });

  @override
  State<AppTextField> createState() => _AppTextFieldState();
}

class _AppTextFieldState extends State<AppTextField> {
  bool _showPassword = false;

  @override
  Widget build(BuildContext context) {
    return TextFormField(
      controller: widget.controller,
      validator: widget.validator,
      obscureText: widget.obscureText && !_showPassword,
      keyboardType: widget.keyboardType,
      maxLines: widget.obscureText ? 1 : widget.maxLines,
      onChanged: widget.onChanged,
      decoration: InputDecoration(
        labelText: widget.label,
        hintText: widget.hint,
        prefixIcon: widget.prefixIcon != null ? Icon(widget.prefixIcon) : null,
        suffixIcon: widget.obscureText
            ? IconButton(
                icon: Icon(_showPassword ? Icons.visibility_off : Icons.visibility),
                onPressed: () => setState(() => _showPassword = !_showPassword),
              )
            : null,
        border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
        filled: true,
      ),
    );
  }
}

// ─── Status Badge ───
enum StatusType { success, warning, error, info }

class StatusBadge extends StatelessWidget {
  final String label;
  final StatusType type;
  final bool isDot;

  const StatusBadge({
    super.key,
    required this.label,
    this.type = StatusType.info,
    this.isDot = false,
  });

  Color _getColor() {
    switch (type) {
      case StatusType.success: return Colors.green;
      case StatusType.warning: return Colors.orange;
      case StatusType.error: return Colors.red;
      case StatusType.info: return Colors.blue;
    }
  }

  @override
  Widget build(BuildContext context) {
    Color color = _getColor();

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 4),
      decoration: BoxDecoration(
        color: color.withOpacity(0.1),
        borderRadius: BorderRadius.circular(20),
        border: Border.all(color: color.withOpacity(0.3)),
      ),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          if (isDot) ...[
            Container(
              width: 8,
              height: 8,
              decoration: BoxDecoration(color: color, shape: BoxShape.circle),
            ),
            const SizedBox(width: 6),
          ],
          Text(
            label,
            style: TextStyle(
              color: color,
              fontSize: 12,
              fontWeight: FontWeight.w600,
            ),
          ),
        ],
      ),
    );
  }
}

// ─── Card with Elevation ───
class AppCard extends StatelessWidget {
  final Widget child;
  final EdgeInsetsGeometry? padding;
  final VoidCallback? onTap;
  final double elevation;
  final BorderRadius? borderRadius;

  const AppCard({
    super.key,
    required this.child,
    this.padding,
    this.onTap,
    this.elevation = 2,
    this.borderRadius,
  });

  @override
  Widget build(BuildContext context) {
    return Material(
      elevation: elevation,
      borderRadius: borderRadius ?? BorderRadius.circular(16),
      child: InkWell(
        onTap: onTap,
        borderRadius: borderRadius ?? BorderRadius.circular(16),
        child: Padding(
          padding: padding ?? const EdgeInsets.all(16),
          child: child,
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 722: Theme System

```dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';

class AppTheme {
  AppTheme._();

  // Color palette
  static const Color primaryColor = Color(0xFF6C63FF);
  static const Color secondaryColor = Color(0xFF03DAC6);
  static const Color errorColor = Color(0xFFCF6679);

  static ThemeData get lightTheme {
    ColorScheme colorScheme = ColorScheme.fromSeed(
      seedColor: primaryColor,
      brightness: Brightness.light,
    );

    return ThemeData(
      colorScheme: colorScheme,
      useMaterial3: true,

      // Typography
      textTheme: GoogleFonts.notoSansThaiTextTheme().copyWith(
        displayLarge: const TextStyle(fontSize: 57, fontWeight: FontWeight.bold),
        headlineLarge: const TextStyle(fontSize: 32, fontWeight: FontWeight.bold),
        titleLarge: const TextStyle(fontSize: 22, fontWeight: FontWeight.w600),
        bodyLarge: const TextStyle(fontSize: 16),
        bodyMedium: const TextStyle(fontSize: 14),
      ),

      // AppBar
      appBarTheme: AppBarTheme(
        backgroundColor: colorScheme.surface,
        foregroundColor: colorScheme.onSurface,
        elevation: 0,
        centerTitle: false,
        titleTextStyle: TextStyle(
          color: colorScheme.onSurface,
          fontSize: 20,
          fontWeight: FontWeight.bold,
        ),
      ),

      // Card
      cardTheme: CardTheme(
        elevation: 2,
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
        margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      ),

      // ElevatedButton
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 14),
          textStyle: const TextStyle(fontSize: 16, fontWeight: FontWeight.w600),
        ),
      ),

      // Input
      inputDecorationTheme: InputDecorationTheme(
        border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
        filled: true,
        fillColor: colorScheme.surfaceVariant.withOpacity(0.3),
        contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 14),
      ),

      // FloatingActionButton
      floatingActionButtonTheme: FloatingActionButtonThemeData(
        backgroundColor: colorScheme.primaryContainer,
        foregroundColor: colorScheme.onPrimaryContainer,
        shape: const CircleBorder(),
      ),

      // BottomNavigationBar
      bottomNavigationBarTheme: BottomNavigationBarThemeData(
        selectedItemColor: colorScheme.primary,
        unselectedItemColor: colorScheme.onSurface.withOpacity(0.5),
        type: BottomNavigationBarType.fixed,
        elevation: 8,
      ),

      // Divider
      dividerTheme: const DividerThemeData(space: 1, thickness: 1),
    );
  }

  static ThemeData get darkTheme {
    ColorScheme colorScheme = ColorScheme.fromSeed(
      seedColor: primaryColor,
      brightness: Brightness.dark,
    );

    return ThemeData(
      colorScheme: colorScheme,
      useMaterial3: true,

      textTheme: GoogleFonts.notoSansThaiTextTheme(
        ThemeData(brightness: Brightness.dark).textTheme,
      ),

      appBarTheme: AppBarTheme(
        backgroundColor: colorScheme.surface,
        elevation: 0,
        titleTextStyle: TextStyle(
          color: colorScheme.onSurface,
          fontSize: 20,
          fontWeight: FontWeight.bold,
        ),
      ),

      cardTheme: CardTheme(
        elevation: 4,
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
      ),

      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 14),
        ),
      ),

      inputDecorationTheme: InputDecorationTheme(
        border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
        filled: true,
      ),
    );
  }
}

// ─── App with Theme ───
class MyApp extends StatefulWidget {
  const MyApp({super.key});

  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  ThemeMode _themeMode = ThemeMode.system;

  void _toggleTheme() {
    setState(() {
      _themeMode = _themeMode == ThemeMode.light ? ThemeMode.dark : ThemeMode.light;
    });
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'App with Theme',
      theme: AppTheme.lightTheme,
      darkTheme: AppTheme.darkTheme,
      themeMode: _themeMode,
      home: ThemeDemo(onToggle: _toggleTheme, themeMode: _themeMode),
    );
  }
}

class ThemeDemo extends StatelessWidget {
  final VoidCallback onToggle;
  final ThemeMode themeMode;

  const ThemeDemo({super.key, required this.onToggle, required this.themeMode});

  @override
  Widget build(BuildContext context) {
    ThemeData theme = Theme.of(context);
    ColorScheme cs = theme.colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Design System'),
        actions: [
          IconButton(
            icon: Icon(themeMode == ThemeMode.dark ? Icons.light_mode : Icons.dark_mode),
            onPressed: onToggle,
          ),
        ],
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Colors palette
            Text('Colors', style: theme.textTheme.titleLarge),
            const SizedBox(height: 8),
            Wrap(
              spacing: 8,
              runSpacing: 8,
              children: [
                _ColorChip('Primary', cs.primary),
                _ColorChip('Secondary', cs.secondary),
                _ColorChip('Surface', cs.surface),
                _ColorChip('Error', cs.error),
              ],
            ),

            const SizedBox(height: 24),

            // Typography
            Text('Typography', style: theme.textTheme.titleLarge),
            const SizedBox(height: 8),
            Text('Display Large', style: theme.textTheme.displayLarge?.copyWith(fontSize: 24)),
            Text('Headline Large', style: theme.textTheme.headlineLarge?.copyWith(fontSize: 20)),
            Text('Title Large', style: theme.textTheme.titleLarge),
            Text('Body Large', style: theme.textTheme.bodyLarge),
            Text('Body Medium', style: theme.textTheme.bodyMedium),

            const SizedBox(height: 24),

            // Buttons
            Text('Buttons', style: theme.textTheme.titleLarge),
            const SizedBox(height: 8),
            Wrap(
              spacing: 8,
              runSpacing: 8,
              children: [
                AppButton(label: 'Primary', onPressed: () {}),
                AppButton(label: 'Secondary', variant: AppButtonVariant.secondary, onPressed: () {}),
                AppButton(label: 'Danger', variant: AppButtonVariant.danger, onPressed: () {}),
                AppButton(label: 'Ghost', variant: AppButtonVariant.ghost, onPressed: () {}),
                AppButton(label: 'Loading', isLoading: true, onPressed: () {}),
                AppButton(label: 'With Icon', prefixIcon: Icons.add, onPressed: () {}),
              ],
            ),

            const SizedBox(height: 24),

            // Badges
            Text('Status Badges', style: theme.textTheme.titleLarge),
            const SizedBox(height: 8),
            Wrap(
              spacing: 8,
              runSpacing: 8,
              children: [
                const StatusBadge(label: 'Active', type: StatusType.success, isDot: true),
                const StatusBadge(label: 'Pending', type: StatusType.warning, isDot: true),
                const StatusBadge(label: 'Error', type: StatusType.error),
                const StatusBadge(label: 'Info', type: StatusType.info),
              ],
            ),
          ],
        ),
      ),
    );
  }
}

class _ColorChip extends StatelessWidget {
  final String name;
  final Color color;

  const _ColorChip(this.name, this.color);

  @override
  Widget build(BuildContext context) {
    bool isDark = color.computeLuminance() < 0.5;
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
      decoration: BoxDecoration(
        color: color,
        borderRadius: BorderRadius.circular(8),
      ),
      child: Text(
        name,
        style: TextStyle(color: isDark ? Colors.white : Colors.black, fontSize: 12),
      ),
    );
  }
}
```

---

**← [Part 20 - BLoC](part-20-bloc.md)**

**ต่อไป: [Part 22 - Internationalization →](part-22-i18n.md)**
