# Part 79: Accessibility Advanced
## ขั้นตอนที่ 3041-3080

## 🎯 เป้าหมายของ Part นี้
- ทดสอบ Screen Reader (TalkBack บน Android, VoiceOver บน iOS)
- สร้าง Custom Semantics Actions
- จัดการ Focus สำหรับ Modals และ Dialogs
- ตรวจสอบ WCAG 2.1 AA compliance
- Accessibility audit อัตโนมัติ

---

## ขั้นตอนที่ 3041: Semantics Fundamentals

```dart
// lib/accessibility/semantics_basics.dart
import 'package:flutter/material.dart';
import 'package:flutter/semantics.dart';

/// Semantics ใน Flutter คือ metadata ที่ช่วยให้ screen readers
/// เข้าใจ UI และอ่านให้ผู้ใช้ที่มีความบกพร่องทางการมองเห็นฟัง

// 1. Basic Semantics Widget
class AccessibleCard extends StatelessWidget {
  const AccessibleCard({
    super.key,
    required this.title,
    required this.description,
    required this.onTap,
    this.imageUrl,
  });

  final String title;
  final String description;
  final VoidCallback onTap;
  final String? imageUrl;

  @override
  Widget build(BuildContext context) {
    return Semantics(
      // กำหนด semantic label ที่สมบูรณ์
      label: '$title. $description',
      hint: 'Double tap to open',
      button: true,
      onTap: onTap,
      child: GestureDetector(
        onTap: onTap,
        child: Card(
          child: Padding(
            padding: const EdgeInsets.all(16),
            child: Row(
              children: [
                if (imageUrl != null)
                  // ExcludeSemantics ป้องกัน screen reader อ่านซ้ำ
                  ExcludeSemantics(
                    child: CircleAvatar(
                      radius: 30,
                      backgroundImage: NetworkImage(imageUrl!),
                    ),
                  ),
                const SizedBox(width: 12),
                Expanded(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      // MergeSemantics รวม semantic children เป็น node เดียว
                      MergeSemantics(
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            Text(
                              title,
                              style: const TextStyle(
                                fontSize: 16,
                                fontWeight: FontWeight.bold,
                              ),
                            ),
                            Text(
                              description,
                              style: const TextStyle(
                                color: Colors.grey,
                              ),
                            ),
                          ],
                        ),
                      ),
                    ],
                  ),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}

// 2. Custom Semantics Actions
class AudioPlayerWidget extends StatefulWidget {
  const AudioPlayerWidget({super.key});

  @override
  State<AudioPlayerWidget> createState() => _AudioPlayerWidgetState();
}

class _AudioPlayerWidgetState extends State<AudioPlayerWidget> {
  bool _isPlaying = false;
  double _volume = 0.7;
  int _currentTrack = 0;

  final List<String> _tracks = [
    'Symphony No. 5 - Beethoven',
    'The Four Seasons - Vivaldi',
    'Moonlight Sonata - Beethoven',
    'Canon in D - Pachelbel',
  ];

  @override
  Widget build(BuildContext context) {
    return Semantics(
      // กำหนด custom semantic actions
      customSemanticsActions: {
        const CustomSemanticsAction(label: 'Play'): () {
          setState(() => _isPlaying = true);
        },
        const CustomSemanticsAction(label: 'Pause'): () {
          setState(() => _isPlaying = false);
        },
        const CustomSemanticsAction(label: 'Next Track'): () {
          setState(() {
            _currentTrack = (_currentTrack + 1) % _tracks.length;
          });
        },
        const CustomSemanticsAction(label: 'Previous Track'): () {
          setState(() {
            _currentTrack =
                (_currentTrack - 1 + _tracks.length) % _tracks.length;
          });
        },
        const CustomSemanticsAction(label: 'Volume Up'): () {
          setState(() {
            _volume = (_volume + 0.1).clamp(0.0, 1.0);
          });
        },
        const CustomSemanticsAction(label: 'Volume Down'): () {
          setState(() {
            _volume = (_volume - 0.1).clamp(0.0, 1.0);
          });
        },
      },
      label: 'Audio Player. Now playing: ${_tracks[_currentTrack]}. '
          '${_isPlaying ? 'Playing' : 'Paused'}. '
          'Volume: ${(_volume * 100).toInt()}%',
      child: Card(
        color: Colors.grey.shade900,
        child: Padding(
          padding: const EdgeInsets.all(20),
          child: Column(
            children: [
              // Album art placeholder
              Container(
                width: 200,
                height: 200,
                decoration: BoxDecoration(
                  color: Colors.grey.shade700,
                  borderRadius: BorderRadius.circular(12),
                ),
                child: ExcludeSemantics(
                  child: Icon(
                    Icons.music_note,
                    size: 80,
                    color: Colors.white.withOpacity(0.5),
                  ),
                ),
              ),
              const SizedBox(height: 20),
              Text(
                _tracks[_currentTrack],
                style: const TextStyle(
                  color: Colors.white,
                  fontSize: 18,
                  fontWeight: FontWeight.bold,
                ),
              ),
              const SizedBox(height: 16),
              // Playback controls
              Row(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Semantics(
                    label: 'Previous track',
                    button: true,
                    child: IconButton(
                      icon: const Icon(Icons.skip_previous, color: Colors.white),
                      iconSize: 36,
                      onPressed: () {
                        setState(() {
                          _currentTrack =
                              (_currentTrack - 1 + _tracks.length) %
                                  _tracks.length;
                        });
                      },
                    ),
                  ),
                  const SizedBox(width: 16),
                  Semantics(
                    label: _isPlaying ? 'Pause' : 'Play',
                    button: true,
                    child: FloatingActionButton(
                      onPressed: () {
                        setState(() => _isPlaying = !_isPlaying);
                      },
                      child: Icon(_isPlaying ? Icons.pause : Icons.play_arrow),
                    ),
                  ),
                  const SizedBox(width: 16),
                  Semantics(
                    label: 'Next track',
                    button: true,
                    child: IconButton(
                      icon: const Icon(Icons.skip_next, color: Colors.white),
                      iconSize: 36,
                      onPressed: () {
                        setState(() {
                          _currentTrack =
                              (_currentTrack + 1) % _tracks.length;
                        });
                      },
                    ),
                  ),
                ],
              ),
              const SizedBox(height: 16),
              // Volume slider
              Row(
                children: [
                  Semantics(
                    label: 'Volume',
                    child: const Icon(Icons.volume_down, color: Colors.white),
                  ),
                  Expanded(
                    child: Semantics(
                      label: 'Volume slider',
                      value: '${(_volume * 100).toInt()}%',
                      increasedValue: '${((_volume + 0.1).clamp(0.0, 1.0) * 100).toInt()}%',
                      decreasedValue: '${((_volume - 0.1).clamp(0.0, 1.0) * 100).toInt()}%',
                      onIncrease: () {
                        setState(() => _volume = (_volume + 0.1).clamp(0.0, 1.0));
                      },
                      onDecrease: () {
                        setState(() => _volume = (_volume - 0.1).clamp(0.0, 1.0));
                      },
                      child: Slider(
                        value: _volume,
                        onChanged: (v) => setState(() => _volume = v),
                        activeColor: Colors.white,
                        inactiveColor: Colors.white30,
                      ),
                    ),
                  ),
                  Semantics(
                    label: 'Volume high',
                    child: const Icon(Icons.volume_up, color: Colors.white),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3042: Focus Management

```dart
// lib/accessibility/focus_management.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

/// Focus Management สำคัญมากสำหรับ keyboard navigation
/// และ screen readers

// 1. Focus ใน Modal Dialog
class AccessibleModal extends StatefulWidget {
  const AccessibleModal({super.key});

  @override
  State<AccessibleModal> createState() => _AccessibleModalState();
}

class _AccessibleModalState extends State<AccessibleModal> {
  void _showAccessibleDialog(BuildContext context) {
    showDialog(
      context: context,
      builder: (context) => const _DeleteConfirmDialog(),
    );
  }

  void _showAccessibleBottomSheet(BuildContext context) {
    showModalBottomSheet(
      context: context,
      builder: (context) => const _AccessibleBottomSheet(),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Focus Management')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          ElevatedButton(
            onPressed: () => _showAccessibleDialog(context),
            child: const Text('Show Accessible Dialog'),
          ),
          const SizedBox(height: 16),
          ElevatedButton(
            onPressed: () => _showAccessibleBottomSheet(context),
            child: const Text('Show Accessible Bottom Sheet'),
          ),
          const SizedBox(height: 16),
          const _KeyboardNavigationDemo(),
        ],
      ),
    );
  }
}

class _DeleteConfirmDialog extends StatelessWidget {
  const _DeleteConfirmDialog();

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      // semanticLabel บอก screen reader ว่านี่คือ dialog อะไร
      semanticLabel: 'Delete confirmation dialog',
      title: const Text('Delete Item'),
      content: const Text(
        'Are you sure you want to delete this item? This action cannot be undone.',
      ),
      actions: [
        // Cancel button ควรเป็น focus แรก
        TextButton(
          autofocus: true, // focus แรกใน dialog
          onPressed: () => Navigator.pop(context),
          child: const Text('Cancel'),
        ),
        TextButton(
          onPressed: () {
            Navigator.pop(context);
            ScaffoldMessenger.of(context).showSnackBar(
              const SnackBar(content: Text('Item deleted')),
            );
          },
          style: TextButton.styleFrom(foregroundColor: Colors.red),
          child: const Text('Delete'),
        ),
      ],
    );
  }
}

class _AccessibleBottomSheet extends StatelessWidget {
  const _AccessibleBottomSheet();

  @override
  Widget build(BuildContext context) {
    return Semantics(
      label: 'Share options menu',
      child: SafeArea(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            // Drag handle
            ExcludeSemantics(
              child: Container(
                margin: const EdgeInsets.symmetric(vertical: 8),
                width: 40,
                height: 4,
                decoration: BoxDecoration(
                  color: Colors.grey.shade300,
                  borderRadius: BorderRadius.circular(2),
                ),
              ),
            ),
            const ListTile(
              title: Text(
                'Share',
                style: TextStyle(
                  fontWeight: FontWeight.bold,
                  fontSize: 18,
                ),
              ),
            ),
            const Divider(),
            // First item gets autofocus
            ListTile(
              autofocus: true,
              leading: const Icon(Icons.share),
              title: const Text('Share via...'),
              onTap: () => Navigator.pop(context),
            ),
            ListTile(
              leading: const Icon(Icons.copy),
              title: const Text('Copy Link'),
              onTap: () => Navigator.pop(context),
            ),
            ListTile(
              leading: const Icon(Icons.email),
              title: const Text('Send Email'),
              onTap: () => Navigator.pop(context),
            ),
            ListTile(
              leading: const Icon(Icons.close),
              title: const Text('Cancel'),
              onTap: () => Navigator.pop(context),
            ),
            const SizedBox(height: 8),
          ],
        ),
      ),
    );
  }
}

// 2. Keyboard Navigation
class _KeyboardNavigationDemo extends StatefulWidget {
  const _KeyboardNavigationDemo();

  @override
  State<_KeyboardNavigationDemo> createState() =>
      _KeyboardNavigationDemoState();
}

class _KeyboardNavigationDemoState extends State<_KeyboardNavigationDemo> {
  final FocusNode _firstFocus = FocusNode();
  final FocusNode _secondFocus = FocusNode();
  final FocusNode _thirdFocus = FocusNode();
  String _lastFocused = 'None';

  @override
  void initState() {
    super.initState();
    for (final node in [_firstFocus, _secondFocus, _thirdFocus]) {
      node.addListener(() {
        if (node.hasFocus) {
          setState(() {
            _lastFocused = node == _firstFocus
                ? 'First'
                : node == _secondFocus
                    ? 'Second'
                    : 'Third';
          });
        }
      });
    }
  }

  @override
  void dispose() {
    _firstFocus.dispose();
    _secondFocus.dispose();
    _thirdFocus.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return FocusTraversalGroup(
      policy: OrderedTraversalPolicy(),
      child: Card(
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              const Text(
                'Keyboard Navigation (Tab/Shift+Tab)',
                style: TextStyle(fontWeight: FontWeight.bold),
              ),
              Text('Last focused: $_lastFocused'),
              const SizedBox(height: 16),
              Row(
                children: [
                  Expanded(
                    child: FocusTraversalOrder(
                      order: const NumericFocusOrder(1),
                      child: ElevatedButton(
                        focusNode: _firstFocus,
                        onPressed: () {},
                        child: const Text('First (Tab 1)'),
                      ),
                    ),
                  ),
                  const SizedBox(width: 8),
                  Expanded(
                    child: FocusTraversalOrder(
                      order: const NumericFocusOrder(3),
                      child: ElevatedButton(
                        focusNode: _thirdFocus,
                        onPressed: () {},
                        child: const Text('Third (Tab 3)'),
                      ),
                    ),
                  ),
                ],
              ),
              const SizedBox(height: 8),
              FocusTraversalOrder(
                order: const NumericFocusOrder(2),
                child: OutlinedButton(
                  focusNode: _secondFocus,
                  onPressed: () {},
                  child: const Text('Second (Tab 2) - Out of visual order'),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3043: Color Contrast และ WCAG 2.1

```dart
// lib/accessibility/wcag_compliance.dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

/// WCAG 2.1 AA ต้องการ:
/// - Text: contrast ratio >= 4.5:1 (ปกติ) หรือ 3:1 (text ใหญ่ >= 18px)
/// - UI Components: contrast ratio >= 3:1
/// - Focus indicator: contrast ratio >= 3:1

class ColorContrastChecker {
  /// คำนวณ relative luminance ตาม WCAG
  static double relativeLuminance(Color color) {
    double r = color.r;
    double g = color.g;
    double b = color.b;

    r = r <= 0.04045 ? r / 12.92 : math.pow((r + 0.055) / 1.055, 2.4).toDouble();
    g = g <= 0.04045 ? g / 12.92 : math.pow((g + 0.055) / 1.055, 2.4).toDouble();
    b = b <= 0.04045 ? b / 12.92 : math.pow((b + 0.055) / 1.055, 2.4).toDouble();

    return 0.2126 * r + 0.7152 * g + 0.0722 * b;
  }

  /// คำนวณ contrast ratio ระหว่าง 2 สี
  static double contrastRatio(Color color1, Color color2) {
    final l1 = relativeLuminance(color1);
    final l2 = relativeLuminance(color2);
    final lighter = math.max(l1, l2);
    final darker = math.min(l1, l2);
    return (lighter + 0.05) / (darker + 0.05);
  }

  /// ตรวจสอบว่าผ่าน WCAG AA สำหรับ normal text (4.5:1)
  static bool passesAA(Color foreground, Color background) {
    return contrastRatio(foreground, background) >= 4.5;
  }

  /// ตรวจสอบว่าผ่าน WCAG AA สำหรับ large text (3:1)
  static bool passesAALargeText(Color foreground, Color background) {
    return contrastRatio(foreground, background) >= 3.0;
  }

  /// ตรวจสอบว่าผ่าน WCAG AAA (7:1)
  static bool passesAAA(Color foreground, Color background) {
    return contrastRatio(foreground, background) >= 7.0;
  }

  /// หาสีที่ contrast สูงสุดระหว่าง white และ black
  static Color getAccessibleTextColor(Color background) {
    final whiteContrast = contrastRatio(Colors.white, background);
    final blackContrast = contrastRatio(Colors.black, background);
    return whiteContrast > blackContrast ? Colors.white : Colors.black;
  }
}

// Widget ที่แสดง contrast checker
class ContrastCheckerWidget extends StatefulWidget {
  const ContrastCheckerWidget({super.key});

  @override
  State<ContrastCheckerWidget> createState() => _ContrastCheckerWidgetState();
}

class _ContrastCheckerWidgetState extends State<ContrastCheckerWidget> {
  Color _foreground = Colors.black;
  Color _background = Colors.white;

  static const List<Color> _presetColors = [
    Colors.white,
    Colors.black,
    Colors.blue,
    Colors.red,
    Colors.green,
    Colors.yellow,
    Colors.orange,
    Colors.purple,
    Color(0xFF2196F3),
    Color(0xFF4CAF50),
    Color(0xFFF44336),
    Color(0xFFFF9800),
  ];

  @override
  Widget build(BuildContext context) {
    final ratio = ColorContrastChecker.contrastRatio(_foreground, _background);
    final passesAA = ColorContrastChecker.passesAA(_foreground, _background);
    final passesAALarge =
        ColorContrastChecker.passesAALargeText(_foreground, _background);
    final passesAAA = ColorContrastChecker.passesAAA(_foreground, _background);

    return Scaffold(
      appBar: AppBar(title: const Text('WCAG Contrast Checker')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          // Preview
          Container(
            height: 120,
            decoration: BoxDecoration(
              color: _background,
              borderRadius: BorderRadius.circular(12),
              border: Border.all(color: Colors.grey.shade300),
            ),
            child: Center(
              child: Text(
                'Sample Text',
                style: TextStyle(
                  color: _foreground,
                  fontSize: 24,
                  fontWeight: FontWeight.bold,
                ),
              ),
            ),
          ),
          const SizedBox(height: 20),
          // Ratio display
          Card(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                children: [
                  Text(
                    'Contrast Ratio: ${ratio.toStringAsFixed(2)}:1',
                    style: const TextStyle(
                      fontSize: 24,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                  const SizedBox(height: 16),
                  _buildCheckRow('AA Normal Text (4.5:1)', passesAA),
                  _buildCheckRow('AA Large Text (3:1)', passesAALarge),
                  _buildCheckRow('AAA Normal Text (7:1)', passesAAA),
                ],
              ),
            ),
          ),
          const SizedBox(height: 16),
          // Color pickers
          _buildColorSection(
            'Foreground Color',
            _foreground,
            (color) => setState(() => _foreground = color),
          ),
          const SizedBox(height: 16),
          _buildColorSection(
            'Background Color',
            _background,
            (color) => setState(() => _background = color),
          ),
        ],
      ),
    );
  }

  Widget _buildCheckRow(String label, bool passes) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        children: [
          Icon(
            passes ? Icons.check_circle : Icons.cancel,
            color: passes ? Colors.green : Colors.red,
          ),
          const SizedBox(width: 8),
          Text(label),
          const Spacer(),
          Container(
            padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
            decoration: BoxDecoration(
              color: passes
                  ? Colors.green.shade100
                  : Colors.red.shade100,
              borderRadius: BorderRadius.circular(4),
            ),
            child: Text(
              passes ? 'PASS' : 'FAIL',
              style: TextStyle(
                color: passes ? Colors.green.shade800 : Colors.red.shade800,
                fontWeight: FontWeight.bold,
                fontSize: 12,
              ),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildColorSection(
    String title,
    Color selectedColor,
    void Function(Color) onChanged,
  ) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              title,
              style: const TextStyle(fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            Wrap(
              spacing: 8,
              runSpacing: 8,
              children: _presetColors.map((color) {
                final isSelected = selectedColor == color;
                return GestureDetector(
                  onTap: () => onChanged(color),
                  child: Semantics(
                    label: 'Select color ${color.toString()}',
                    selected: isSelected,
                    button: true,
                    child: Container(
                      width: 44,
                      height: 44,
                      decoration: BoxDecoration(
                        color: color,
                        borderRadius: BorderRadius.circular(8),
                        border: Border.all(
                          color: isSelected ? Colors.blue : Colors.grey.shade300,
                          width: isSelected ? 3 : 1,
                        ),
                      ),
                      child: isSelected
                          ? Icon(
                              Icons.check,
                              color: ColorContrastChecker.getAccessibleTextColor(
                                  color),
                            )
                          : null,
                    ),
                  ),
                );
              }).toList(),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3044: Accessibility Audit Tool

```dart
// lib/accessibility/accessibility_audit.dart
import 'package:flutter/material.dart';
import 'package:flutter/semantics.dart';

/// Accessibility issues ที่พบบ่อย
enum AccessibilityIssueType {
  missingLabel,
  lowContrast,
  smallTouchTarget,
  missingRole,
  missingHint,
}

class AccessibilityIssue {
  const AccessibilityIssue({
    required this.type,
    required this.description,
    required this.recommendation,
    this.severity = IssueSeverity.warning,
  });

  final AccessibilityIssueType type;
  final String description;
  final String recommendation;
  final IssueSeverity severity;
}

enum IssueSeverity { error, warning, info }

/// WCAG 2.1 AA Checklist
class WCAGChecklist {
  static const List<WCAGItem> items = [
    // Perceivable
    WCAGItem(
      criterion: '1.1.1',
      level: 'A',
      title: 'Non-text Content',
      description: 'All images have meaningful alt text',
      category: 'Perceivable',
    ),
    WCAGItem(
      criterion: '1.3.1',
      level: 'A',
      title: 'Info and Relationships',
      description: 'Semantic structure communicates meaning',
      category: 'Perceivable',
    ),
    WCAGItem(
      criterion: '1.4.1',
      level: 'A',
      title: 'Use of Color',
      description: 'Color is not the only visual means of conveying information',
      category: 'Perceivable',
    ),
    WCAGItem(
      criterion: '1.4.3',
      level: 'AA',
      title: 'Contrast (Minimum)',
      description: 'Text has contrast ratio of at least 4.5:1',
      category: 'Perceivable',
    ),
    WCAGItem(
      criterion: '1.4.4',
      level: 'AA',
      title: 'Resize Text',
      description: 'Text can be resized up to 200% without loss of content',
      category: 'Perceivable',
    ),
    // Operable
    WCAGItem(
      criterion: '2.1.1',
      level: 'A',
      title: 'Keyboard',
      description: 'All functionality available from keyboard',
      category: 'Operable',
    ),
    WCAGItem(
      criterion: '2.4.3',
      level: 'A',
      title: 'Focus Order',
      description: 'Focus order preserves meaning and operability',
      category: 'Operable',
    ),
    WCAGItem(
      criterion: '2.4.7',
      level: 'AA',
      title: 'Focus Visible',
      description: 'Keyboard focus indicator is visible',
      category: 'Operable',
    ),
    WCAGItem(
      criterion: '2.5.3',
      level: 'A',
      title: 'Label in Name',
      description: 'UI components labeled with accessible name that includes visible text',
      category: 'Operable',
    ),
    // Understandable
    WCAGItem(
      criterion: '3.1.1',
      level: 'A',
      title: 'Language of Page',
      description: 'Default human language can be determined programmatically',
      category: 'Understandable',
    ),
    WCAGItem(
      criterion: '3.2.2',
      level: 'A',
      title: 'On Input',
      description: 'Changing setting of UI component does not automatically cause context change',
      category: 'Understandable',
    ),
    WCAGItem(
      criterion: '3.3.1',
      level: 'A',
      title: 'Error Identification',
      description: 'Input errors are identified and described to the user',
      category: 'Understandable',
    ),
    WCAGItem(
      criterion: '3.3.2',
      level: 'A',
      title: 'Labels or Instructions',
      description: 'Labels or instructions provided for user input',
      category: 'Understandable',
    ),
    // Robust
    WCAGItem(
      criterion: '4.1.2',
      level: 'A',
      title: 'Name, Role, Value',
      description: 'UI components have name, role, and values that can be determined by assistive technologies',
      category: 'Robust',
    ),
    WCAGItem(
      criterion: '4.1.3',
      level: 'AA',
      title: 'Status Messages',
      description: 'Status messages can be determined programmatically without receiving focus',
      category: 'Robust',
    ),
  ];
}

class WCAGItem {
  const WCAGItem({
    required this.criterion,
    required this.level,
    required this.title,
    required this.description,
    required this.category,
  });

  final String criterion;
  final String level;
  final String title;
  final String description;
  final String category;
}

// WCAG Checklist Widget
class WCAGChecklistWidget extends StatefulWidget {
  const WCAGChecklistWidget({super.key});

  @override
  State<WCAGChecklistWidget> createState() => _WCAGChecklistWidgetState();
}

class _WCAGChecklistWidgetState extends State<WCAGChecklistWidget> {
  final Map<String, bool> _checked = {};
  String _filter = 'All';

  final List<String> _categories = [
    'All',
    'Perceivable',
    'Operable',
    'Understandable',
    'Robust',
  ];

  List<WCAGItem> get _filteredItems {
    if (_filter == 'All') return WCAGChecklist.items;
    return WCAGChecklist.items.where((item) => item.category == _filter).toList();
  }

  int get _checkedCount =>
      _checked.values.where((v) => v).length;

  int get _totalCount => WCAGChecklist.items.length;

  double get _progress =>
      _totalCount > 0 ? _checkedCount / _totalCount : 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('WCAG 2.1 AA Checklist'),
        actions: [
          Center(
            child: Padding(
              padding: const EdgeInsets.only(right: 16),
              child: Semantics(
                label: '$_checkedCount of $_totalCount items checked',
                child: Text(
                  '$_checkedCount/$_totalCount',
                  style: const TextStyle(fontSize: 16),
                ),
              ),
            ),
          ),
        ],
      ),
      body: Column(
        children: [
          // Progress bar
          Semantics(
            label: 'Progress: ${(_progress * 100).toInt()}% complete',
            value: '${(_progress * 100).toInt()}%',
            child: LinearProgressIndicator(
              value: _progress,
              minHeight: 8,
              backgroundColor: Colors.grey.shade200,
              valueColor: AlwaysStoppedAnimation<Color>(
                _progress == 1.0 ? Colors.green : Colors.blue,
              ),
            ),
          ),
          // Filter chips
          Padding(
            padding: const EdgeInsets.all(8),
            child: Semantics(
              label: 'Filter by category',
              child: SingleChildScrollView(
                scrollDirection: Axis.horizontal,
                child: Row(
                  children: _categories.map((cat) {
                    final isSelected = _filter == cat;
                    return Padding(
                      padding: const EdgeInsets.only(right: 8),
                      child: FilterChip(
                        label: Text(cat),
                        selected: isSelected,
                        onSelected: (_) {
                          setState(() => _filter = cat);
                        },
                      ),
                    );
                  }).toList(),
                ),
              ),
            ),
          ),
          // Checklist
          Expanded(
            child: ListView.builder(
              restorationId: 'wcag_checklist',
              itemCount: _filteredItems.length,
              itemBuilder: (context, index) {
                final item = _filteredItems[index];
                final isChecked = _checked[item.criterion] ?? false;

                return Semantics(
                  checked: isChecked,
                  label: 'WCAG ${item.criterion} ${item.title}. '
                      'Level ${item.level}. '
                      '${item.description}. '
                      '${isChecked ? 'Checked' : 'Not checked'}.',
                  child: CheckboxListTile(
                    value: isChecked,
                    onChanged: (value) {
                      setState(() => _checked[item.criterion] = value ?? false);
                    },
                    title: Row(
                      children: [
                        Text(
                          '${item.criterion} ',
                          style: const TextStyle(
                            fontWeight: FontWeight.bold,
                            color: Colors.blue,
                          ),
                        ),
                        Container(
                          padding: const EdgeInsets.symmetric(
                            horizontal: 6,
                            vertical: 2,
                          ),
                          decoration: BoxDecoration(
                            color: item.level == 'A'
                                ? Colors.blue.shade100
                                : Colors.orange.shade100,
                            borderRadius: BorderRadius.circular(4),
                          ),
                          child: Text(
                            item.level,
                            style: TextStyle(
                              fontSize: 11,
                              fontWeight: FontWeight.bold,
                              color: item.level == 'A'
                                  ? Colors.blue.shade800
                                  : Colors.orange.shade800,
                            ),
                          ),
                        ),
                      ],
                    ),
                    subtitle: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        Text(
                          item.title,
                          style: const TextStyle(fontWeight: FontWeight.w500),
                        ),
                        Text(
                          item.description,
                          style: TextStyle(
                            fontSize: 12,
                            color: Colors.grey.shade600,
                          ),
                        ),
                      ],
                    ),
                    secondary: isChecked
                        ? const Icon(Icons.check_circle, color: Colors.green)
                        : null,
                  ),
                );
              },
            ),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: () {
          final unchecked = _filteredItems
              .where((item) => !(_checked[item.criterion] ?? false))
              .length;
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(
              content: Text(
                unchecked == 0
                    ? 'All items checked! Great accessibility!'
                    : '$unchecked items remaining',
              ),
            ),
          );
        },
        icon: const Icon(Icons.assessment),
        label: const Text('Check Progress'),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3045: Accessible Forms และ Error Messages

```dart
// lib/accessibility/accessible_forms.dart
import 'package:flutter/material.dart';

/// Accessible Form ที่ทำตาม WCAG 3.3.1 และ 3.3.2

class AccessibleLoginForm extends StatefulWidget {
  const AccessibleLoginForm({super.key});

  @override
  State<AccessibleLoginForm> createState() => _AccessibleLoginFormState();
}

class _AccessibleLoginFormState extends State<AccessibleLoginForm> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  final _emailFocus = FocusNode();
  final _passwordFocus = FocusNode();

  bool _isLoading = false;
  bool _obscurePassword = true;
  String? _emailError;
  String? _passwordError;

  // สำหรับ screen reader announcements
  final _semanticsKey = GlobalKey();

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    _emailFocus.dispose();
    _passwordFocus.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    setState(() {
      _emailError = null;
      _passwordError = null;
    });

    // Validate
    final emailValue = _emailController.text;
    final passwordValue = _passwordController.text;

    bool hasError = false;

    if (emailValue.isEmpty) {
      setState(() => _emailError = 'Email is required');
      hasError = true;
    } else if (!emailValue.contains('@')) {
      setState(() => _emailError = 'Please enter a valid email address');
      hasError = true;
    }

    if (passwordValue.isEmpty) {
      setState(() => _passwordError = 'Password is required');
      hasError = true;
    } else if (passwordValue.length < 6) {
      setState(() =>
          _passwordError = 'Password must be at least 6 characters');
      hasError = true;
    }

    if (hasError) {
      // Announce errors to screen reader
      SemanticsService.announce(
        'Form has errors. '
        '${_emailError != null ? 'Email: $_emailError. ' : ''}'
        '${_passwordError != null ? 'Password: $_passwordError. ' : ''}',
        TextDirection.ltr,
      );
      // Focus first error field
      if (_emailError != null) {
        _emailFocus.requestFocus();
      } else if (_passwordError != null) {
        _passwordFocus.requestFocus();
      }
      return;
    }

    setState(() => _isLoading = true);

    // Simulate API call
    await Future.delayed(const Duration(seconds: 2));

    if (mounted) {
      setState(() => _isLoading = false);

      // Announce success
      SemanticsService.announce(
        'Login successful. Welcome back!',
        TextDirection.ltr,
      );

      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(
          content: Text('Login successful!'),
          backgroundColor: Colors.green,
        ),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Accessible Login')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(24),
        child: AutofillGroup(
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              // Page description for screen readers
              Semantics(
                header: true,
                child: const Text(
                  'Sign In',
                  style: TextStyle(
                    fontSize: 28,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ),
              const SizedBox(height: 8),
              const Text(
                'Enter your credentials to access your account',
                style: TextStyle(color: Colors.grey),
              ),
              const SizedBox(height: 32),
              // Email field
              _buildAccessibleTextField(
                controller: _emailController,
                focusNode: _emailFocus,
                label: 'Email Address',
                hint: 'Enter your email',
                errorText: _emailError,
                keyboardType: TextInputType.emailAddress,
                autofillHints: const [AutofillHints.email],
                prefixIcon: Icons.email_outlined,
                onChanged: (_) {
                  if (_emailError != null) {
                    setState(() => _emailError = null);
                  }
                },
                textInputAction: TextInputAction.next,
                onFieldSubmitted: (_) => _passwordFocus.requestFocus(),
              ),
              const SizedBox(height: 20),
              // Password field
              _buildAccessibleTextField(
                controller: _passwordController,
                focusNode: _passwordFocus,
                label: 'Password',
                hint: 'Enter your password (minimum 6 characters)',
                errorText: _passwordError,
                obscureText: _obscurePassword,
                autofillHints: const [AutofillHints.password],
                prefixIcon: Icons.lock_outlined,
                suffixIcon: Semantics(
                  label: _obscurePassword ? 'Show password' : 'Hide password',
                  child: IconButton(
                    icon: Icon(
                      _obscurePassword
                          ? Icons.visibility_outlined
                          : Icons.visibility_off_outlined,
                    ),
                    onPressed: () {
                      setState(() => _obscurePassword = !_obscurePassword);
                    },
                  ),
                ),
                onChanged: (_) {
                  if (_passwordError != null) {
                    setState(() => _passwordError = null);
                  }
                },
                textInputAction: TextInputAction.done,
                onFieldSubmitted: (_) => _submit(),
              ),
              const SizedBox(height: 8),
              Align(
                alignment: Alignment.centerRight,
                child: TextButton(
                  onPressed: () {},
                  child: const Text('Forgot Password?'),
                ),
              ),
              const SizedBox(height: 24),
              // Submit button
              Semantics(
                label: _isLoading ? 'Signing in, please wait' : 'Sign In',
                button: true,
                enabled: !_isLoading,
                child: ElevatedButton(
                  onPressed: _isLoading ? null : _submit,
                  style: ElevatedButton.styleFrom(
                    padding: const EdgeInsets.symmetric(vertical: 16),
                    minimumSize: const Size(double.infinity, 56), // WCAG 2.5.5: 44x44 minimum
                  ),
                  child: _isLoading
                      ? const SizedBox(
                          width: 20,
                          height: 20,
                          child: CircularProgressIndicator(
                            strokeWidth: 2,
                            color: Colors.white,
                          ),
                        )
                      : const Text(
                          'Sign In',
                          style: TextStyle(fontSize: 16),
                        ),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildAccessibleTextField({
    required TextEditingController controller,
    required FocusNode focusNode,
    required String label,
    required String hint,
    String? errorText,
    TextInputType keyboardType = TextInputType.text,
    List<String>? autofillHints,
    bool obscureText = false,
    IconData? prefixIcon,
    Widget? suffixIcon,
    void Function(String)? onChanged,
    TextInputAction textInputAction = TextInputAction.next,
    void Function(String)? onFieldSubmitted,
  }) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        // Label (ทำ explicit label เพราะ accessible กว่า floatingLabel)
        Text(
          label,
          style: const TextStyle(
            fontWeight: FontWeight.w500,
            fontSize: 14,
          ),
        ),
        const SizedBox(height: 6),
        TextFormField(
          controller: controller,
          focusNode: focusNode,
          keyboardType: keyboardType,
          autofillHints: autofillHints,
          obscureText: obscureText,
          textInputAction: textInputAction,
          onFieldSubmitted: onFieldSubmitted,
          onChanged: onChanged,
          decoration: InputDecoration(
            hintText: hint,
            errorText: errorText,
            prefixIcon: prefixIcon != null ? Icon(prefixIcon) : null,
            suffix: suffixIcon,
            border: const OutlineInputBorder(),
            // Error message สำหรับ screen reader จะอ่านอัตโนมัติ
            errorStyle: const TextStyle(fontSize: 13),
          ),
        ),
      ],
    );
  }
}

void main() {
  runApp(
    MaterialApp(
      title: 'Accessibility Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const WCAGChecklistWidget(),
    ),
  );
}
```

---

**← [Part 78](part-78-flutter-state-restoration.md)**
**ต่อไป: [Part 80 →](part-80-flutter-embedding.md)**
