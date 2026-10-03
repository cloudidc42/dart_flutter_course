# Part 58: Design System
## ขั้นตอนที่ 2201-2240

## 🎯 เป้าหมายของ Part นี้
- สร้าง AppTokens (colors, spacing, radius, typography) เป็น design constants
- ตั้งค่า ThemeData จาก tokens เพื่อความสม่ำเสมอ
- สร้าง AppButton component พร้อม variants (primary/secondary/danger/ghost)
- สร้าง AppTextField พร้อม validation states
- สร้าง AppCard พร้อม elevation variants
- สร้าง AppSpacing และ AppTextStyle helper classes

---

## ขั้นตอนที่ 2201: Design Tokens - Colors

```dart
// lib/design_system/tokens/app_colors.dart
import 'package:flutter/material.dart';

class AppColors {
  AppColors._();

  // --- Brand ---
  static const Color primary = Color(0xFF6366F1);        // Indigo
  static const Color primaryLight = Color(0xFFA5B4FC);
  static const Color primaryDark = Color(0xFF4338CA);
  static const Color primaryContainer = Color(0xFFE0E7FF);

  static const Color secondary = Color(0xFF14B8A6);      // Teal
  static const Color secondaryLight = Color(0xFF5EEAD4);
  static const Color secondaryDark = Color(0xFF0F766E);
  static const Color secondaryContainer = Color(0xFFCCFBF1);

  // --- Semantic ---
  static const Color success = Color(0xFF22C55E);
  static const Color successLight = Color(0xFFBBF7D0);
  static const Color successDark = Color(0xFF15803D);

  static const Color warning = Color(0xFFF59E0B);
  static const Color warningLight = Color(0xFFFDE68A);
  static const Color warningDark = Color(0xFFB45309);

  static const Color danger = Color(0xFFEF4444);
  static const Color dangerLight = Color(0xFFFECACA);
  static const Color dangerDark = Color(0xFFB91C1C);

  static const Color info = Color(0xFF3B82F6);
  static const Color infoLight = Color(0xFFBFDBFE);
  static const Color infoDark = Color(0xFF1D4ED8);

  // --- Neutrals ---
  static const Color white = Color(0xFFFFFFFF);
  static const Color black = Color(0xFF000000);
  static const Color gray50 = Color(0xFFF9FAFB);
  static const Color gray100 = Color(0xFFF3F4F6);
  static const Color gray200 = Color(0xFFE5E7EB);
  static const Color gray300 = Color(0xFFD1D5DB);
  static const Color gray400 = Color(0xFF9CA3AF);
  static const Color gray500 = Color(0xFF6B7280);
  static const Color gray600 = Color(0xFF4B5563);
  static const Color gray700 = Color(0xFF374151);
  static const Color gray800 = Color(0xFF1F2937);
  static const Color gray900 = Color(0xFF111827);

  // --- Surfaces ---
  static const Color surface = white;
  static const Color surfaceVariant = gray50;
  static const Color surfaceOverlay = Color(0x0A000000);
  static const Color outline = gray200;
  static const Color outlineVariant = gray100;

  // --- Text ---
  static const Color textPrimary = gray900;
  static const Color textSecondary = gray600;
  static const Color textTertiary = gray400;
  static const Color textDisabled = gray300;
  static const Color textInverse = white;

  // --- Dark mode equivalents ---
  static const Color darkSurface = gray900;
  static const Color darkSurfaceVariant = gray800;
  static const Color darkOutline = gray700;
  static const Color darkTextPrimary = gray50;
  static const Color darkTextSecondary = gray400;
}
```

---

## ขั้นตอนที่ 2202: Design Tokens - Spacing & Radius

```dart
// lib/design_system/tokens/app_spacing.dart

/// Spacing scale based on 4px base unit
class AppSpacing {
  AppSpacing._();

  static const double px0 = 0;
  static const double px1 = 1;
  static const double px2 = 2;
  static const double px4 = 4;
  static const double px6 = 6;
  static const double px8 = 8;
  static const double px10 = 10;
  static const double px12 = 12;
  static const double px14 = 14;
  static const double px16 = 16;
  static const double px20 = 20;
  static const double px24 = 24;
  static const double px28 = 28;
  static const double px32 = 32;
  static const double px36 = 36;
  static const double px40 = 40;
  static const double px48 = 48;
  static const double px56 = 56;
  static const double px64 = 64;
  static const double px72 = 72;
  static const double px80 = 80;
  static const double px96 = 96;

  // Semantic aliases
  static const double none = px0;
  static const double xs = px4;
  static const double sm = px8;
  static const double md = px16;
  static const double lg = px24;
  static const double xl = px32;
  static const double xxl = px48;
  static const double xxxl = px64;

  // Page gutters
  static const double pageGutter = px16;
  static const double pagePaddingVertical = px24;

  // Component-specific
  static const double buttonPaddingH = px20;
  static const double buttonPaddingV = px12;
  static const double cardPadding = px16;
  static const double inputPaddingH = px16;
  static const double inputPaddingV = px14;
  static const double iconSize = px24;
  static const double iconSizeSm = px20;
  static const double iconSizeLg = px32;
}

// lib/design_system/tokens/app_radius.dart

/// Border radius tokens
class AppRadius {
  AppRadius._();

  static const double none = 0;
  static const double xs = 4;
  static const double sm = 6;
  static const double md = 8;
  static const double lg = 12;
  static const double xl = 16;
  static const double xxl = 24;
  static const double full = 9999;

  // Semantic
  static const double button = md;
  static const double input = md;
  static const double card = lg;
  static const double badge = full;
  static const double dialog = xl;
  static const double bottomSheet = xxl;
  static const double chip = full;
}
```

---

## ขั้นตอนที่ 2203: Design Tokens - Typography

```dart
// lib/design_system/tokens/app_typography.dart
import 'package:flutter/material.dart';
import 'app_colors.dart';

class AppTextStyle {
  AppTextStyle._();

  static const String _fontFamily = 'Inter';

  // --- Display ---
  static const TextStyle displayLarge = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 57,
    fontWeight: FontWeight.w400,
    letterSpacing: -0.25,
    height: 1.12,
    color: AppColors.textPrimary,
  );

  static const TextStyle displayMedium = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 45,
    fontWeight: FontWeight.w400,
    letterSpacing: 0,
    height: 1.16,
    color: AppColors.textPrimary,
  );

  // --- Headline ---
  static const TextStyle headlineLarge = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 32,
    fontWeight: FontWeight.w600,
    letterSpacing: 0,
    height: 1.25,
    color: AppColors.textPrimary,
  );

  static const TextStyle headlineMedium = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 28,
    fontWeight: FontWeight.w600,
    letterSpacing: 0,
    height: 1.29,
    color: AppColors.textPrimary,
  );

  static const TextStyle headlineSmall = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 24,
    fontWeight: FontWeight.w600,
    letterSpacing: 0,
    height: 1.33,
    color: AppColors.textPrimary,
  );

  // --- Title ---
  static const TextStyle titleLarge = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 22,
    fontWeight: FontWeight.w500,
    letterSpacing: 0,
    height: 1.27,
    color: AppColors.textPrimary,
  );

  static const TextStyle titleMedium = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 16,
    fontWeight: FontWeight.w600,
    letterSpacing: 0.15,
    height: 1.5,
    color: AppColors.textPrimary,
  );

  static const TextStyle titleSmall = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 14,
    fontWeight: FontWeight.w600,
    letterSpacing: 0.1,
    height: 1.43,
    color: AppColors.textPrimary,
  );

  // --- Body ---
  static const TextStyle bodyLarge = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 16,
    fontWeight: FontWeight.w400,
    letterSpacing: 0.5,
    height: 1.5,
    color: AppColors.textPrimary,
  );

  static const TextStyle bodyMedium = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 14,
    fontWeight: FontWeight.w400,
    letterSpacing: 0.25,
    height: 1.43,
    color: AppColors.textPrimary,
  );

  static const TextStyle bodySmall = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 12,
    fontWeight: FontWeight.w400,
    letterSpacing: 0.4,
    height: 1.33,
    color: AppColors.textSecondary,
  );

  // --- Label ---
  static const TextStyle labelLarge = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 14,
    fontWeight: FontWeight.w600,
    letterSpacing: 0.1,
    height: 1.43,
    color: AppColors.textPrimary,
  );

  static const TextStyle labelMedium = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 12,
    fontWeight: FontWeight.w600,
    letterSpacing: 0.5,
    height: 1.33,
    color: AppColors.textPrimary,
  );

  static const TextStyle labelSmall = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 11,
    fontWeight: FontWeight.w600,
    letterSpacing: 0.5,
    height: 1.45,
    color: AppColors.textSecondary,
  );

  // --- Button ---
  static const TextStyle buttonLarge = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 16,
    fontWeight: FontWeight.w600,
    letterSpacing: 0.1,
    height: 1.25,
  );

  static const TextStyle buttonMedium = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 14,
    fontWeight: FontWeight.w600,
    letterSpacing: 0.1,
    height: 1.25,
  );

  static const TextStyle buttonSmall = TextStyle(
    fontFamily: _fontFamily,
    fontSize: 12,
    fontWeight: FontWeight.w600,
    letterSpacing: 0.1,
    height: 1.25,
  );
}
```

---

## ขั้นตอนที่ 2204: ThemeData from Tokens

```dart
// lib/design_system/app_theme.dart
import 'package:flutter/material.dart';
import 'tokens/app_colors.dart';
import 'tokens/app_radius.dart';
import 'tokens/app_typography.dart';

class AppTheme {
  AppTheme._();

  static ThemeData get light => _buildTheme(Brightness.light);
  static ThemeData get dark => _buildTheme(Brightness.dark);

  static ThemeData _buildTheme(Brightness brightness) {
    final isDark = brightness == Brightness.dark;
    final colorScheme = isDark ? _darkColorScheme : _lightColorScheme;

    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      brightness: brightness,
      fontFamily: 'Inter',

      // Text theme
      textTheme: TextTheme(
        displayLarge: AppTextStyle.displayLarge,
        displayMedium: AppTextStyle.displayMedium,
        headlineLarge: AppTextStyle.headlineLarge,
        headlineMedium: AppTextStyle.headlineMedium,
        headlineSmall: AppTextStyle.headlineSmall,
        titleLarge: AppTextStyle.titleLarge,
        titleMedium: AppTextStyle.titleMedium,
        titleSmall: AppTextStyle.titleSmall,
        bodyLarge: AppTextStyle.bodyLarge,
        bodyMedium: AppTextStyle.bodyMedium,
        bodySmall: AppTextStyle.bodySmall,
        labelLarge: AppTextStyle.labelLarge,
        labelMedium: AppTextStyle.labelMedium,
        labelSmall: AppTextStyle.labelSmall,
      ),

      // Card theme
      cardTheme: CardTheme(
        elevation: 0,
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(AppRadius.card),
          side: BorderSide(
            color: isDark ? AppColors.darkOutline : AppColors.outline,
          ),
        ),
        color: isDark ? AppColors.darkSurface : AppColors.surface,
        margin: EdgeInsets.zero,
      ),

      // Input decoration theme
      inputDecorationTheme: InputDecorationTheme(
        filled: true,
        fillColor: isDark
            ? AppColors.darkSurfaceVariant
            : AppColors.surfaceVariant,
        border: OutlineInputBorder(
          borderRadius: BorderRadius.circular(AppRadius.input),
          borderSide: BorderSide(
            color: isDark ? AppColors.darkOutline : AppColors.outline,
          ),
        ),
        enabledBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(AppRadius.input),
          borderSide: BorderSide(
            color: isDark ? AppColors.darkOutline : AppColors.outline,
          ),
        ),
        focusedBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(AppRadius.input),
          borderSide: const BorderSide(color: AppColors.primary, width: 2),
        ),
        errorBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(AppRadius.input),
          borderSide: const BorderSide(color: AppColors.danger),
        ),
        focusedErrorBorder: OutlineInputBorder(
          borderRadius: BorderRadius.circular(AppRadius.input),
          borderSide: const BorderSide(color: AppColors.danger, width: 2),
        ),
        contentPadding: const EdgeInsets.symmetric(
          horizontal: 16,
          vertical: 14,
        ),
        labelStyle: AppTextStyle.bodyMedium.copyWith(
          color: isDark ? AppColors.darkTextSecondary : AppColors.textSecondary,
        ),
        hintStyle: AppTextStyle.bodyMedium.copyWith(
          color: AppColors.textTertiary,
        ),
        errorStyle: AppTextStyle.bodySmall.copyWith(color: AppColors.danger),
      ),

      // Elevated button theme
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          backgroundColor: AppColors.primary,
          foregroundColor: AppColors.white,
          elevation: 0,
          padding: const EdgeInsets.symmetric(horizontal: 20, vertical: 12),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(AppRadius.button),
          ),
          textStyle: AppTextStyle.buttonMedium,
        ),
      ),

      // Chip theme
      chipTheme: ChipThemeData(
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(AppRadius.chip),
        ),
      ),

      // Divider theme
      dividerTheme: const DividerThemeData(
        color: AppColors.outline,
        thickness: 1,
        space: 1,
      ),

      // App bar theme
      appBarTheme: AppBarTheme(
        elevation: 0,
        scrolledUnderElevation: 1,
        backgroundColor: isDark ? AppColors.darkSurface : AppColors.surface,
        foregroundColor:
            isDark ? AppColors.darkTextPrimary : AppColors.textPrimary,
        titleTextStyle: AppTextStyle.titleLarge.copyWith(
          color: isDark ? AppColors.darkTextPrimary : AppColors.textPrimary,
        ),
        surfaceTintColor: Colors.transparent,
      ),
    );
  }

  static const ColorScheme _lightColorScheme = ColorScheme(
    brightness: Brightness.light,
    primary: AppColors.primary,
    onPrimary: AppColors.white,
    primaryContainer: AppColors.primaryContainer,
    onPrimaryContainer: AppColors.primaryDark,
    secondary: AppColors.secondary,
    onSecondary: AppColors.white,
    secondaryContainer: AppColors.secondaryContainer,
    onSecondaryContainer: AppColors.secondaryDark,
    error: AppColors.danger,
    onError: AppColors.white,
    surface: AppColors.surface,
    onSurface: AppColors.textPrimary,
    surfaceContainerHighest: AppColors.surfaceVariant,
    outline: AppColors.outline,
    outlineVariant: AppColors.outlineVariant,
  );

  static const ColorScheme _darkColorScheme = ColorScheme(
    brightness: Brightness.dark,
    primary: AppColors.primaryLight,
    onPrimary: AppColors.primaryDark,
    primaryContainer: AppColors.primaryDark,
    onPrimaryContainer: AppColors.primaryLight,
    secondary: AppColors.secondaryLight,
    onSecondary: AppColors.secondaryDark,
    secondaryContainer: AppColors.secondaryDark,
    onSecondaryContainer: AppColors.secondaryLight,
    error: AppColors.dangerLight,
    onError: AppColors.dangerDark,
    surface: AppColors.darkSurface,
    onSurface: AppColors.darkTextPrimary,
    surfaceContainerHighest: AppColors.darkSurfaceVariant,
    outline: AppColors.darkOutline,
    outlineVariant: AppColors.gray800,
  );
}
```

---

## ขั้นตอนที่ 2205: AppButton Component

```dart
// lib/design_system/components/app_button.dart
import 'package:flutter/material.dart';
import '../tokens/app_colors.dart';
import '../tokens/app_radius.dart';
import '../tokens/app_spacing.dart';
import '../tokens/app_typography.dart';

enum AppButtonVariant { primary, secondary, danger, ghost, outline }
enum AppButtonSize { small, medium, large }

class AppButton extends StatefulWidget {
  final String label;
  final VoidCallback? onPressed;
  final AppButtonVariant variant;
  final AppButtonSize size;
  final Widget? leading;
  final Widget? trailing;
  final bool isLoading;
  final bool isFullWidth;

  const AppButton({
    super.key,
    required this.label,
    required this.onPressed,
    this.variant = AppButtonVariant.primary,
    this.size = AppButtonSize.medium,
    this.leading,
    this.trailing,
    this.isLoading = false,
    this.isFullWidth = false,
  });

  const AppButton.primary({
    super.key,
    required this.label,
    required this.onPressed,
    this.size = AppButtonSize.medium,
    this.leading,
    this.trailing,
    this.isLoading = false,
    this.isFullWidth = false,
  }) : variant = AppButtonVariant.primary;

  const AppButton.secondary({
    super.key,
    required this.label,
    required this.onPressed,
    this.size = AppButtonSize.medium,
    this.leading,
    this.trailing,
    this.isLoading = false,
    this.isFullWidth = false,
  }) : variant = AppButtonVariant.secondary;

  const AppButton.danger({
    super.key,
    required this.label,
    required this.onPressed,
    this.size = AppButtonSize.medium,
    this.leading,
    this.trailing,
    this.isLoading = false,
    this.isFullWidth = false,
  }) : variant = AppButtonVariant.danger;

  const AppButton.ghost({
    super.key,
    required this.label,
    required this.onPressed,
    this.size = AppButtonSize.medium,
    this.leading,
    this.trailing,
    this.isLoading = false,
    this.isFullWidth = false,
  }) : variant = AppButtonVariant.ghost;

  @override
  State<AppButton> createState() => _AppButtonState();
}

class _AppButtonState extends State<AppButton> {
  bool _isPressed = false;

  _ButtonStyle get _style => _ButtonStyle.fromVariant(widget.variant);
  _ButtonDimensions get _dims => _ButtonDimensions.fromSize(widget.size);

  @override
  Widget build(BuildContext context) {
    final isDisabled = widget.onPressed == null || widget.isLoading;

    return GestureDetector(
      onTapDown: isDisabled ? null : (_) => setState(() => _isPressed = true),
      onTapUp: isDisabled ? null : (_) => setState(() => _isPressed = false),
      onTapCancel: isDisabled ? null : () => setState(() => _isPressed = false),
      onTap: isDisabled ? null : widget.onPressed,
      child: AnimatedOpacity(
        opacity: isDisabled ? 0.5 : 1.0,
        duration: const Duration(milliseconds: 150),
        child: AnimatedScale(
          scale: _isPressed ? 0.97 : 1.0,
          duration: const Duration(milliseconds: 100),
          child: Container(
            width: widget.isFullWidth ? double.infinity : null,
            height: _dims.height,
            decoration: BoxDecoration(
              color: _style.backgroundColor,
              borderRadius: BorderRadius.circular(AppRadius.button),
              border: _style.border,
              boxShadow: _style.shadow,
            ),
            padding: EdgeInsets.symmetric(
              horizontal: _dims.paddingH,
              vertical: _dims.paddingV,
            ),
            child: Row(
              mainAxisSize:
                  widget.isFullWidth ? MainAxisSize.max : MainAxisSize.min,
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                if (widget.isLoading) ...[
                  SizedBox(
                    width: _dims.iconSize,
                    height: _dims.iconSize,
                    child: CircularProgressIndicator(
                      strokeWidth: 2,
                      color: _style.foregroundColor,
                    ),
                  ),
                  const SizedBox(width: AppSpacing.sm),
                ] else if (widget.leading != null) ...[
                  IconTheme(
                    data: IconThemeData(
                        color: _style.foregroundColor, size: _dims.iconSize),
                    child: widget.leading!,
                  ),
                  const SizedBox(width: AppSpacing.xs),
                ],
                Text(
                  widget.label,
                  style: _dims.textStyle.copyWith(color: _style.foregroundColor),
                ),
                if (widget.trailing != null && !widget.isLoading) ...[
                  const SizedBox(width: AppSpacing.xs),
                  IconTheme(
                    data: IconThemeData(
                        color: _style.foregroundColor, size: _dims.iconSize),
                    child: widget.trailing!,
                  ),
                ],
              ],
            ),
          ),
        ),
      ),
    );
  }
}

class _ButtonStyle {
  final Color backgroundColor;
  final Color foregroundColor;
  final Border? border;
  final List<BoxShadow>? shadow;

  const _ButtonStyle({
    required this.backgroundColor,
    required this.foregroundColor,
    this.border,
    this.shadow,
  });

  factory _ButtonStyle.fromVariant(AppButtonVariant variant) {
    switch (variant) {
      case AppButtonVariant.primary:
        return const _ButtonStyle(
          backgroundColor: AppColors.primary,
          foregroundColor: AppColors.white,
          shadow: [
            BoxShadow(
              color: Color(0x266366F1),
              blurRadius: 8,
              offset: Offset(0, 2),
            ),
          ],
        );
      case AppButtonVariant.secondary:
        return const _ButtonStyle(
          backgroundColor: AppColors.primaryContainer,
          foregroundColor: AppColors.primaryDark,
        );
      case AppButtonVariant.danger:
        return const _ButtonStyle(
          backgroundColor: AppColors.danger,
          foregroundColor: AppColors.white,
          shadow: [
            BoxShadow(
              color: Color(0x26EF4444),
              blurRadius: 8,
              offset: Offset(0, 2),
            ),
          ],
        );
      case AppButtonVariant.ghost:
        return const _ButtonStyle(
          backgroundColor: Colors.transparent,
          foregroundColor: AppColors.primary,
        );
      case AppButtonVariant.outline:
        return _ButtonStyle(
          backgroundColor: Colors.transparent,
          foregroundColor: AppColors.primary,
          border: Border.all(color: AppColors.primary, width: 1.5),
        );
    }
  }
}

class _ButtonDimensions {
  final double height;
  final double paddingH;
  final double paddingV;
  final double iconSize;
  final TextStyle textStyle;

  const _ButtonDimensions({
    required this.height,
    required this.paddingH,
    required this.paddingV,
    required this.iconSize,
    required this.textStyle,
  });

  factory _ButtonDimensions.fromSize(AppButtonSize size) {
    switch (size) {
      case AppButtonSize.small:
        return const _ButtonDimensions(
          height: 32,
          paddingH: 12,
          paddingV: 6,
          iconSize: 16,
          textStyle: AppTextStyle.buttonSmall,
        );
      case AppButtonSize.medium:
        return const _ButtonDimensions(
          height: 44,
          paddingH: 20,
          paddingV: 12,
          iconSize: 20,
          textStyle: AppTextStyle.buttonMedium,
        );
      case AppButtonSize.large:
        return const _ButtonDimensions(
          height: 52,
          paddingH: 24,
          paddingV: 14,
          iconSize: 24,
          textStyle: AppTextStyle.buttonLarge,
        );
    }
  }
}
```

---

## ขั้นตอนที่ 2206: AppTextField Component

```dart
// lib/design_system/components/app_text_field.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import '../tokens/app_colors.dart';
import '../tokens/app_typography.dart';

class AppTextField extends StatefulWidget {
  final String? label;
  final String? hint;
  final String? helperText;
  final String? errorText;
  final TextEditingController? controller;
  final FocusNode? focusNode;
  final TextInputType keyboardType;
  final TextInputAction textInputAction;
  final bool obscureText;
  final bool readOnly;
  final bool enabled;
  final int? maxLines;
  final int? maxLength;
  final ValueChanged<String>? onChanged;
  final VoidCallback? onTap;
  final ValueChanged<String>? onSubmitted;
  final FormFieldValidator<String>? validator;
  final List<TextInputFormatter>? inputFormatters;
  final Widget? prefix;
  final Widget? suffix;
  final String? prefixText;

  const AppTextField({
    super.key,
    this.label,
    this.hint,
    this.helperText,
    this.errorText,
    this.controller,
    this.focusNode,
    this.keyboardType = TextInputType.text,
    this.textInputAction = TextInputAction.done,
    this.obscureText = false,
    this.readOnly = false,
    this.enabled = true,
    this.maxLines = 1,
    this.maxLength,
    this.onChanged,
    this.onTap,
    this.onSubmitted,
    this.validator,
    this.inputFormatters,
    this.prefix,
    this.suffix,
    this.prefixText,
  });

  @override
  State<AppTextField> createState() => _AppTextFieldState();
}

class _AppTextFieldState extends State<AppTextField> {
  late bool _obscureText;
  bool _isFocused = false;
  late final FocusNode _focusNode;

  @override
  void initState() {
    super.initState();
    _obscureText = widget.obscureText;
    _focusNode = widget.focusNode ?? FocusNode();
    _focusNode.addListener(() {
      setState(() => _isFocused = _focusNode.hasFocus);
    });
  }

  @override
  void dispose() {
    if (widget.focusNode == null) _focusNode.dispose();
    super.dispose();
  }

  bool get _hasError => widget.errorText != null;

  Color get _borderColor {
    if (_hasError) return AppColors.danger;
    if (_isFocused) return AppColors.primary;
    return AppColors.outline;
  }

  double get _borderWidth {
    if (_isFocused || _hasError) return 2;
    return 1;
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      mainAxisSize: MainAxisSize.min,
      children: [
        if (widget.label != null) ...[
          Text(
            widget.label!,
            style: AppTextStyle.labelMedium.copyWith(
              color: _hasError ? AppColors.danger : AppColors.textPrimary,
            ),
          ),
          const SizedBox(height: 6),
        ],
        AnimatedContainer(
          duration: const Duration(milliseconds: 150),
          decoration: BoxDecoration(
            borderRadius: BorderRadius.circular(8),
            border: Border.all(
              color: _borderColor,
              width: _borderWidth,
            ),
            color: widget.enabled
                ? AppColors.surfaceVariant
                : AppColors.gray100,
          ),
          child: TextFormField(
            controller: widget.controller,
            focusNode: _focusNode,
            keyboardType: widget.keyboardType,
            textInputAction: widget.textInputAction,
            obscureText: _obscureText,
            readOnly: widget.readOnly,
            enabled: widget.enabled,
            maxLines: widget.obscureText ? 1 : widget.maxLines,
            maxLength: widget.maxLength,
            onChanged: widget.onChanged,
            onTap: widget.onTap,
            onFieldSubmitted: widget.onSubmitted,
            validator: widget.validator,
            inputFormatters: widget.inputFormatters,
            style: AppTextStyle.bodyMedium,
            decoration: InputDecoration(
              hintText: widget.hint,
              hintStyle: AppTextStyle.bodyMedium.copyWith(
                color: AppColors.textTertiary,
              ),
              prefixText: widget.prefixText,
              prefixStyle: AppTextStyle.bodyMedium,
              prefix: widget.prefix,
              suffix: widget.obscureText
                  ? GestureDetector(
                      onTap: () =>
                          setState(() => _obscureText = !_obscureText),
                      child: Icon(
                        _obscureText
                            ? Icons.visibility_off_outlined
                            : Icons.visibility_outlined,
                        size: 20,
                        color: AppColors.textSecondary,
                      ),
                    )
                  : widget.suffix,
              border: InputBorder.none,
              contentPadding: const EdgeInsets.symmetric(
                horizontal: 16,
                vertical: 14,
              ),
              counterText: '',
              errorStyle: const TextStyle(height: 0),
            ),
          ),
        ),
        if (_hasError) ...[
          const SizedBox(height: 4),
          Row(
            children: [
              const Icon(Icons.error_outline,
                  size: 14, color: AppColors.danger),
              const SizedBox(width: 4),
              Expanded(
                child: Text(
                  widget.errorText!,
                  style: AppTextStyle.bodySmall.copyWith(
                    color: AppColors.danger,
                  ),
                ),
              ),
            ],
          ),
        ] else if (widget.helperText != null) ...[
          const SizedBox(height: 4),
          Text(
            widget.helperText!,
            style: AppTextStyle.bodySmall,
          ),
        ],
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 2207: AppCard Component

```dart
// lib/design_system/components/app_card.dart
import 'package:flutter/material.dart';
import '../tokens/app_colors.dart';
import '../tokens/app_radius.dart';
import '../tokens/app_spacing.dart';

enum AppCardVariant { flat, elevated, outlined, filled }

class AppCard extends StatelessWidget {
  final Widget child;
  final AppCardVariant variant;
  final EdgeInsetsGeometry? padding;
  final VoidCallback? onTap;
  final Color? backgroundColor;
  final double? width;

  const AppCard({
    super.key,
    required this.child,
    this.variant = AppCardVariant.flat,
    this.padding,
    this.onTap,
    this.backgroundColor,
    this.width,
  });

  const AppCard.elevated({
    super.key,
    required this.child,
    this.padding,
    this.onTap,
    this.backgroundColor,
    this.width,
  }) : variant = AppCardVariant.elevated;

  const AppCard.outlined({
    super.key,
    required this.child,
    this.padding,
    this.onTap,
    this.backgroundColor,
    this.width,
  }) : variant = AppCardVariant.outlined;

  const AppCard.filled({
    super.key,
    required this.child,
    this.padding,
    this.onTap,
    this.backgroundColor,
    this.width,
  }) : variant = AppCardVariant.filled;

  _CardStyle get _style => _CardStyle.fromVariant(variant);

  @override
  Widget build(BuildContext context) {
    Widget card = Container(
      width: width,
      decoration: BoxDecoration(
        color: backgroundColor ?? _style.backgroundColor,
        borderRadius: BorderRadius.circular(AppRadius.card),
        border: _style.border,
        boxShadow: _style.shadow,
      ),
      child: ClipRRect(
        borderRadius: BorderRadius.circular(AppRadius.card),
        child: Padding(
          padding: padding ?? const EdgeInsets.all(AppSpacing.cardPadding),
          child: child,
        ),
      ),
    );

    if (onTap != null) {
      return Material(
        color: Colors.transparent,
        child: InkWell(
          borderRadius: BorderRadius.circular(AppRadius.card),
          onTap: onTap,
          child: card,
        ),
      );
    }

    return card;
  }
}

class _CardStyle {
  final Color backgroundColor;
  final Border? border;
  final List<BoxShadow>? shadow;

  const _CardStyle({
    required this.backgroundColor,
    this.border,
    this.shadow,
  });

  factory _CardStyle.fromVariant(AppCardVariant variant) {
    switch (variant) {
      case AppCardVariant.flat:
        return const _CardStyle(
          backgroundColor: AppColors.surface,
          border: Border.fromBorderSide(
            BorderSide(color: AppColors.outline),
          ),
        );
      case AppCardVariant.elevated:
        return const _CardStyle(
          backgroundColor: AppColors.surface,
          shadow: [
            BoxShadow(
              color: Color(0x0F000000),
              blurRadius: 8,
              spreadRadius: 0,
              offset: Offset(0, 2),
            ),
            BoxShadow(
              color: Color(0x0A000000),
              blurRadius: 24,
              spreadRadius: 0,
              offset: Offset(0, 8),
            ),
          ],
        );
      case AppCardVariant.outlined:
        return const _CardStyle(
          backgroundColor: AppColors.surface,
          border: Border.fromBorderSide(
            BorderSide(color: AppColors.outline, width: 1.5),
          ),
        );
      case AppCardVariant.filled:
        return const _CardStyle(
          backgroundColor: AppColors.surfaceVariant,
        );
    }
  }
}
```

---

## ขั้นตอนที่ 2208: Design System Showcase App

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'design_system/app_theme.dart';
import 'design_system/components/app_button.dart';
import 'design_system/components/app_card.dart';
import 'design_system/components/app_text_field.dart';
import 'design_system/tokens/app_colors.dart';
import 'design_system/tokens/app_spacing.dart';
import 'design_system/tokens/app_typography.dart';

void main() {
  runApp(const DesignSystemApp());
}

class DesignSystemApp extends StatelessWidget {
  const DesignSystemApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Design System',
      theme: AppTheme.light,
      darkTheme: AppTheme.dark,
      home: const DesignSystemShowcase(),
    );
  }
}

class DesignSystemShowcase extends StatefulWidget {
  const DesignSystemShowcase({super.key});

  @override
  State<DesignSystemShowcase> createState() => _DesignSystemShowcaseState();
}

class _DesignSystemShowcaseState extends State<DesignSystemShowcase> {
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  String? _emailError;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Design System')),
      body: ListView(
        padding: const EdgeInsets.all(AppSpacing.pageGutter),
        children: [
          _buildSection('Typography', _buildTypographySection()),
          _buildSection('Buttons', _buildButtonsSection()),
          _buildSection('Text Fields', _buildTextFieldsSection()),
          _buildSection('Cards', _buildCardsSection()),
          _buildSection('Colors', _buildColorsSection()),
        ],
      ),
    );
  }

  Widget _buildSection(String title, Widget content) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Padding(
          padding: const EdgeInsets.symmetric(vertical: AppSpacing.md),
          child: Text(title, style: AppTextStyle.headlineSmall),
        ),
        content,
        const Divider(height: 32),
      ],
    );
  }

  Widget _buildTypographySection() {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text('Headline Large', style: AppTextStyle.headlineLarge),
        const SizedBox(height: 8),
        Text('Headline Medium', style: AppTextStyle.headlineMedium),
        const SizedBox(height: 8),
        Text('Title Large', style: AppTextStyle.titleLarge),
        const SizedBox(height: 8),
        Text('Body Large — The quick brown fox jumps over the lazy dog.',
            style: AppTextStyle.bodyLarge),
        const SizedBox(height: 8),
        Text('Body Medium — The quick brown fox jumps over the lazy dog.',
            style: AppTextStyle.bodyMedium),
        const SizedBox(height: 8),
        Text('Label Medium', style: AppTextStyle.labelMedium),
      ],
    );
  }

  Widget _buildButtonsSection() {
    return Wrap(
      spacing: AppSpacing.sm,
      runSpacing: AppSpacing.sm,
      children: [
        AppButton.primary(label: 'Primary', onPressed: () {}),
        AppButton.secondary(label: 'Secondary', onPressed: () {}),
        AppButton.danger(label: 'Danger', onPressed: () {}),
        AppButton.ghost(label: 'Ghost', onPressed: () {}),
        AppButton(
          label: 'With Icon',
          variant: AppButtonVariant.primary,
          onPressed: () {},
          leading: const Icon(Icons.add),
        ),
        const AppButton(
          label: 'Loading',
          variant: AppButtonVariant.primary,
          onPressed: null,
          isLoading: true,
        ),
        const AppButton(
          label: 'Disabled',
          variant: AppButtonVariant.primary,
          onPressed: null,
        ),
        AppButton(
          label: 'Full Width',
          variant: AppButtonVariant.primary,
          onPressed: () {},
          isFullWidth: true,
        ),
      ],
    );
  }

  Widget _buildTextFieldsSection() {
    return Column(
      children: [
        AppTextField(
          label: 'Email',
          hint: 'you@example.com',
          controller: _emailController,
          keyboardType: TextInputType.emailAddress,
          errorText: _emailError,
          prefix: const Icon(Icons.email_outlined, size: 20),
          onChanged: (v) {
            setState(() {
              _emailError = v.contains('@') ? null : 'Invalid email';
            });
          },
        ),
        const SizedBox(height: 16),
        AppTextField(
          label: 'Password',
          hint: 'Enter your password',
          controller: _passwordController,
          obscureText: true,
          helperText: 'Must be at least 8 characters',
        ),
        const SizedBox(height: 16),
        const AppTextField(
          label: 'Disabled',
          hint: 'Cannot edit this',
          enabled: false,
        ),
      ],
    );
  }

  Widget _buildCardsSection() {
    return Column(
      children: [
        AppCard(
          child: Text('Flat Card', style: AppTextStyle.titleMedium),
        ),
        const SizedBox(height: 12),
        AppCard.elevated(
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text('Elevated Card', style: AppTextStyle.titleMedium),
              const SizedBox(height: 8),
              Text('With shadow elevation', style: AppTextStyle.bodySmall),
            ],
          ),
        ),
        const SizedBox(height: 12),
        AppCard.outlined(
          onTap: () {},
          child: Row(
            children: [
              const Icon(Icons.touch_app, color: AppColors.primary),
              const SizedBox(width: 12),
              Text('Tappable Outlined Card',
                  style: AppTextStyle.titleMedium),
            ],
          ),
        ),
        const SizedBox(height: 12),
        AppCard.filled(
          backgroundColor: AppColors.primaryContainer,
          child: Text('Filled Card (custom color)',
              style: AppTextStyle.titleMedium.copyWith(
                color: AppColors.primaryDark,
              )),
        ),
      ],
    );
  }

  Widget _buildColorsSection() {
    final colors = [
      (AppColors.primary, 'Primary'),
      (AppColors.secondary, 'Secondary'),
      (AppColors.success, 'Success'),
      (AppColors.warning, 'Warning'),
      (AppColors.danger, 'Danger'),
      (AppColors.info, 'Info'),
    ];

    return Wrap(
      spacing: 8,
      runSpacing: 8,
      children: colors
          .map((e) => Column(
                children: [
                  Container(
                    width: 48,
                    height: 48,
                    decoration: BoxDecoration(
                      color: e.$1,
                      borderRadius: BorderRadius.circular(8),
                    ),
                  ),
                  const SizedBox(height: 4),
                  Text(e.$2, style: AppTextStyle.labelSmall),
                ],
              ))
          .toList(),
    );
  }
}
```

---

**← [Part 57](part-57-offline-first-architecture.md)**
**ต่อไป: [Part 59 →](part-59-advanced-navigation.md)**
