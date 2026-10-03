# Part 34: Flutter Desktop
## ขั้นตอนที่ 1241-1280

---

## 🎯 เป้าหมายของ Part นี้

- Flutter Desktop (Windows/macOS/Linux) setup
- Desktop-specific layouts
- Menu bar และ system tray
- File system access
- Window management

---

## ขั้นตอนที่ 1241: Desktop Setup

```bash
# เปิดใช้งาน desktop platforms
flutter config --enable-windows-desktop
flutter config --enable-macos-desktop
flutter config --enable-linux-desktop

# สร้าง project ที่มี desktop support
flutter create --platforms=windows,macos,linux my_desktop_app

# Run
flutter run -d windows
flutter run -d macos
flutter run -d linux

# Build
flutter build windows --release
flutter build macos --release
flutter build linux --release
```

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  window_manager: ^0.3.8    # Window management
  menubar: ^0.2.0           # Native menu bar
  path_provider: ^2.1.2     # File system paths
  file_picker: ^8.0.0       # File picker dialog
  bitsdojo_window: ^0.1.6   # Custom title bar
  system_tray: ^2.0.3       # System tray icon
  hotkey_manager: ^0.2.2    # Global hotkeys
  screen_retriever: ^0.2.0  # Screen info
```

---

## ขั้นตอนที่ 1242: Window Management

```dart
import 'package:flutter/material.dart';
import 'package:window_manager/window_manager.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // ตั้งค่า window
  await windowManager.ensureInitialized();

  WindowOptions windowOptions = const WindowOptions(
    size: Size(1200, 800),
    minimumSize: Size(800, 600),
    center: true,
    backgroundColor: Colors.transparent,
    skipTaskbar: false,
    titleBarStyle: TitleBarStyle.normal,
    title: 'My Desktop App',
  );

  windowManager.waitUntilReadyToShow(windowOptions, () async {
    await windowManager.show();
    await windowManager.focus();
  });

  runApp(const MyApp());
}

// ─── Custom Window Controls ───
class CustomTitleBar extends StatelessWidget {
  const CustomTitleBar({super.key});

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onPanStart: (_) => windowManager.startDragging(),
      child: Container(
        height: 40,
        color: Theme.of(context).colorScheme.surface,
        child: Row(
          children: [
            // App icon
            const SizedBox(width: 12),
            const FlutterLogo(size: 20),
            const SizedBox(width: 8),
            const Text('My Desktop App', style: TextStyle(fontSize: 14)),
            const Spacer(),
            // Window controls
            WindowControlButton(
              icon: Icons.minimize,
              onTap: () => windowManager.minimize(),
            ),
            WindowControlButton(
              icon: Icons.crop_square,
              onTap: () async {
                bool isMaximized = await windowManager.isMaximized();
                if (isMaximized) {
                  windowManager.unmaximize();
                } else {
                  windowManager.maximize();
                }
              },
            ),
            WindowControlButton(
              icon: Icons.close,
              isClose: true,
              onTap: () => windowManager.close(),
            ),
          ],
        ),
      ),
    );
  }
}

class WindowControlButton extends StatefulWidget {
  final IconData icon;
  final VoidCallback onTap;
  final bool isClose;

  const WindowControlButton({
    super.key,
    required this.icon,
    required this.onTap,
    this.isClose = false,
  });

  @override
  State<WindowControlButton> createState() => _WindowControlButtonState();
}

class _WindowControlButtonState extends State<WindowControlButton> {
  bool _isHovered = false;

  @override
  Widget build(BuildContext context) {
    return MouseRegion(
      onEnter: (_) => setState(() => _isHovered = true),
      onExit: (_) => setState(() => _isHovered = false),
      child: GestureDetector(
        onTap: widget.onTap,
        child: AnimatedContainer(
          duration: const Duration(milliseconds: 100),
          width: 46,
          height: 40,
          color: _isHovered
              ? (widget.isClose ? Colors.red : Colors.grey.withOpacity(0.2))
              : Colors.transparent,
          child: Icon(
            widget.icon,
            size: 16,
            color: _isHovered && widget.isClose ? Colors.white : null,
          ),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1243: Desktop Layout

```dart
import 'package:flutter/material.dart';

// ─── Three-panel Desktop Layout ───
class DesktopLayout extends StatefulWidget {
  const DesktopLayout({super.key});

  @override
  State<DesktopLayout> createState() => _DesktopLayoutState();
}

class _DesktopLayoutState extends State<DesktopLayout> {
  double _leftPanelWidth = 250;
  double _rightPanelWidth = 300;
  int _selectedIndex = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          // Title bar
          if (Theme.of(context).platform != TargetPlatform.macOS)
            const CustomTitleBar(),
          // Main content
          Expanded(
            child: Row(
              children: [
                // Left panel (sidebar)
                SizedBox(
                  width: _leftPanelWidth,
                  child: _buildSidebar(),
                ),
                // Resize handle left
                MouseRegion(
                  cursor: SystemMouseCursors.resizeColumn,
                  child: GestureDetector(
                    onPanUpdate: (details) {
                      setState(() {
                        _leftPanelWidth = (_leftPanelWidth + details.delta.dx)
                            .clamp(150, 400);
                      });
                    },
                    child: Container(
                      width: 4,
                      color: Theme.of(context).dividerColor,
                      child: Center(
                        child: Container(
                          width: 2,
                          height: 40,
                          color: Theme.of(context).colorScheme.outline,
                        ),
                      ),
                    ),
                  ),
                ),
                // Main content
                Expanded(child: _buildMainContent()),
                // Resize handle right
                MouseRegion(
                  cursor: SystemMouseCursors.resizeColumn,
                  child: GestureDetector(
                    onPanUpdate: (details) {
                      setState(() {
                        _rightPanelWidth = (_rightPanelWidth - details.delta.dx)
                            .clamp(200, 500);
                      });
                    },
                    child: Container(
                      width: 4,
                      color: Theme.of(context).dividerColor,
                    ),
                  ),
                ),
                // Right panel (details/inspector)
                SizedBox(
                  width: _rightPanelWidth,
                  child: _buildRightPanel(),
                ),
              ],
            ),
          ),
          // Status bar
          _buildStatusBar(),
        ],
      ),
    );
  }

  Widget _buildSidebar() {
    return Container(
      color: Theme.of(context).colorScheme.surface,
      child: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(8),
            child: TextField(
              decoration: InputDecoration(
                hintText: 'Search...',
                prefixIcon: const Icon(Icons.search, size: 18),
                isDense: true,
                border: OutlineInputBorder(borderRadius: BorderRadius.circular(6)),
              ),
            ),
          ),
          Expanded(
            child: ListView.builder(
              itemCount: 20,
              itemBuilder: (context, index) => ListTile(
                dense: true,
                leading: const Icon(Icons.folder, size: 18),
                title: Text('Project $index', style: const TextStyle(fontSize: 13)),
                selected: _selectedIndex == index,
                selectedColor: Theme.of(context).colorScheme.primary,
                selectedTileColor:
                    Theme.of(context).colorScheme.primaryContainer.withOpacity(0.3),
                onTap: () => setState(() => _selectedIndex = index),
              ),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildMainContent() {
    return Container(
      color: Theme.of(context).colorScheme.background,
      child: Column(
        children: [
          // Toolbar
          Container(
            height: 40,
            padding: const EdgeInsets.symmetric(horizontal: 8),
            decoration: BoxDecoration(
              color: Theme.of(context).colorScheme.surface,
              border: Border(bottom: BorderSide(color: Theme.of(context).dividerColor)),
            ),
            child: Row(
              children: [
                IconButton(icon: const Icon(Icons.add), onPressed: () {}, iconSize: 18),
                IconButton(icon: const Icon(Icons.edit), onPressed: () {}, iconSize: 18),
                IconButton(icon: const Icon(Icons.delete), onPressed: () {}, iconSize: 18),
              ],
            ),
          ),
          // Content
          Expanded(
            child: Center(
              child: Text(
                'Selected: Project $_selectedIndex',
                style: Theme.of(context).textTheme.headlineMedium,
              ),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildRightPanel() {
    return Container(
      color: Theme.of(context).colorScheme.surface,
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text('Inspector', style: Theme.of(context).textTheme.titleMedium),
          const Divider(),
          const _PropertyRow(label: 'Name', value: 'Project'),
          const _PropertyRow(label: 'Size', value: '2.4 MB'),
          const _PropertyRow(label: 'Created', value: '2024-01-15'),
          const _PropertyRow(label: 'Modified', value: '2024-03-20'),
        ],
      ),
    );
  }

  Widget _buildStatusBar() {
    return Container(
      height: 24,
      padding: const EdgeInsets.symmetric(horizontal: 12),
      color: Theme.of(context).colorScheme.inverseSurface,
      child: Row(
        children: [
          Text(
            'Ready',
            style: TextStyle(
              fontSize: 12,
              color: Theme.of(context).colorScheme.onInverseSurface,
            ),
          ),
          const Spacer(),
          Text(
            '${DateTime.now().day}/${DateTime.now().month}/${DateTime.now().year}',
            style: TextStyle(
              fontSize: 12,
              color: Theme.of(context).colorScheme.onInverseSurface,
            ),
          ),
        ],
      ),
    );
  }
}

class _PropertyRow extends StatelessWidget {
  final String label;
  final String value;
  const _PropertyRow({required this.label, required this.value});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        children: [
          SizedBox(
            width: 80,
            child: Text(label, style: const TextStyle(fontSize: 12, color: Colors.grey)),
          ),
          Expanded(
            child: Text(value, style: const TextStyle(fontSize: 12)),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1244: File System Access

```dart
import 'dart:io';
import 'package:path_provider/path_provider.dart';
import 'package:file_picker/file_picker.dart';

// ─── File Service ───
class FileService {
  // เปิด file picker
  Future<String?> pickFile({List<String>? allowedExtensions}) async {
    FilePickerResult? result = await FilePicker.platform.pickFiles(
      type: allowedExtensions != null ? FileType.custom : FileType.any,
      allowedExtensions: allowedExtensions,
    );

    return result?.files.single.path;
  }

  // เปิด multiple files
  Future<List<String>> pickMultipleFiles() async {
    FilePickerResult? result = await FilePicker.platform.pickFiles(
      allowMultiple: true,
    );

    return result?.files.map((f) => f.path!).toList() ?? [];
  }

  // เลือก directory
  Future<String?> pickDirectory() async {
    return FilePicker.platform.getDirectoryPath();
  }

  // Save dialog
  Future<String?> saveFile({
    required String fileName,
    String? initialDirectory,
    List<String>? allowedExtensions,
  }) async {
    return FilePicker.platform.saveFile(
      dialogTitle: 'Save File',
      fileName: fileName,
      initialDirectory: initialDirectory,
      allowedExtensions: allowedExtensions,
    );
  }

  // Read file
  Future<String> readFile(String path) async {
    return File(path).readAsString();
  }

  // Write file
  Future<void> writeFile(String path, String content) async {
    await File(path).writeAsString(content);
  }

  // Get app documents directory
  Future<Directory> getDocumentsDir() async {
    return getApplicationDocumentsDirectory();
  }

  // List files in directory
  Future<List<FileSystemEntity>> listFiles(String dirPath) async {
    Directory dir = Directory(dirPath);
    return dir.listSync().toList();
  }
}

// ─── File Editor Widget ───
class FileEditorWidget extends StatefulWidget {
  const FileEditorWidget({super.key});

  @override
  State<FileEditorWidget> createState() => _FileEditorWidgetState();
}

class _FileEditorWidgetState extends State<FileEditorWidget> {
  final FileService _fileService = FileService();
  final TextEditingController _controller = TextEditingController();
  String? _currentFilePath;
  bool _hasUnsavedChanges = false;

  Future<void> _openFile() async {
    String? path = await _fileService.pickFile(allowedExtensions: ['txt', 'md', 'dart']);
    if (path == null) return;

    String content = await _fileService.readFile(path);
    setState(() {
      _currentFilePath = path;
      _controller.text = content;
      _hasUnsavedChanges = false;
    });
  }

  Future<void> _saveFile() async {
    if (_currentFilePath == null) {
      await _saveAs();
      return;
    }

    await _fileService.writeFile(_currentFilePath!, _controller.text);
    setState(() => _hasUnsavedChanges = false);
    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Saved!')),
      );
    }
  }

  Future<void> _saveAs() async {
    String? path = await _fileService.saveFile(
      fileName: 'document.txt',
      allowedExtensions: ['txt', 'md'],
    );

    if (path == null) return;

    await _fileService.writeFile(path, _controller.text);
    setState(() {
      _currentFilePath = path;
      _hasUnsavedChanges = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // Toolbar
        ToolBar(
          title: _currentFilePath != null
              ? '${_currentFilePath!.split('/').last}${_hasUnsavedChanges ? ' •' : ''}'
              : 'Untitled',
          onOpen: _openFile,
          onSave: _saveFile,
          onSaveAs: _saveAs,
        ),
        // Editor
        Expanded(
          child: TextField(
            controller: _controller,
            maxLines: null,
            expands: true,
            style: const TextStyle(fontFamily: 'monospace', fontSize: 14),
            decoration: const InputDecoration(
              contentPadding: EdgeInsets.all(16),
              border: InputBorder.none,
            ),
            onChanged: (_) => setState(() => _hasUnsavedChanges = true),
          ),
        ),
      ],
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}

class ToolBar extends StatelessWidget {
  final String title;
  final VoidCallback onOpen;
  final VoidCallback onSave;
  final VoidCallback onSaveAs;

  const ToolBar({
    super.key,
    required this.title,
    required this.onOpen,
    required this.onSave,
    required this.onSaveAs,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      height: 40,
      color: Theme.of(context).colorScheme.surface,
      child: Row(
        children: [
          TextButton(onPressed: onOpen, child: const Text('Open')),
          TextButton(onPressed: onSave, child: const Text('Save')),
          TextButton(onPressed: onSaveAs, child: const Text('Save As...')),
          const VerticalDivider(),
          Text(title),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1245: Keyboard Shortcuts

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

// ─── Keyboard Shortcuts ───
class KeyboardShortcutsApp extends StatelessWidget {
  const KeyboardShortcutsApp({super.key});

  @override
  Widget build(BuildContext context) {
    return Shortcuts(
      shortcuts: {
        // Ctrl+N: New
        const SingleActivator(LogicalKeyboardKey.keyN, control: true):
            const NewIntent(),
        // Ctrl+O: Open
        const SingleActivator(LogicalKeyboardKey.keyO, control: true):
            const OpenIntent(),
        // Ctrl+S: Save
        const SingleActivator(LogicalKeyboardKey.keyS, control: true):
            const SaveIntent(),
        // Ctrl+Z: Undo
        const SingleActivator(LogicalKeyboardKey.keyZ, control: true):
            const UndoIntent(),
        // Ctrl+Shift+Z: Redo
        const SingleActivator(
          LogicalKeyboardKey.keyZ,
          control: true,
          shift: true,
        ): const RedoIntent(),
        // F5: Refresh
        const SingleActivator(LogicalKeyboardKey.f5): const RefreshIntent(),
      },
      child: Actions(
        actions: {
          NewIntent: CallbackAction<NewIntent>(onInvoke: (_) => _handleNew()),
          OpenIntent: CallbackAction<OpenIntent>(onInvoke: (_) => _handleOpen()),
          SaveIntent: CallbackAction<SaveIntent>(onInvoke: (_) => _handleSave()),
          UndoIntent: CallbackAction<UndoIntent>(onInvoke: (_) => _handleUndo()),
          RedoIntent: CallbackAction<RedoIntent>(onInvoke: (_) => _handleRedo()),
          RefreshIntent: CallbackAction<RefreshIntent>(onInvoke: (_) => _handleRefresh()),
        },
        child: const HomePage(),
      ),
    );
  }

  Object? _handleNew() { print('New'); return null; }
  Object? _handleOpen() { print('Open'); return null; }
  Object? _handleSave() { print('Save'); return null; }
  Object? _handleUndo() { print('Undo'); return null; }
  Object? _handleRedo() { print('Redo'); return null; }
  Object? _handleRefresh() { print('Refresh'); return null; }
}

class NewIntent extends Intent { const NewIntent(); }
class OpenIntent extends Intent { const OpenIntent(); }
class SaveIntent extends Intent { const SaveIntent(); }
class UndoIntent extends Intent { const UndoIntent(); }
class RedoIntent extends Intent { const RedoIntent(); }
class RefreshIntent extends Intent { const RefreshIntent(); }

// ─── Context Menu (Right-click) ───
class ContextMenuWidget extends StatelessWidget {
  final Widget child;
  const ContextMenuWidget({super.key, required this.child});

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onSecondaryTapDown: (details) {
        _showContextMenu(context, details.globalPosition);
      },
      child: child,
    );
  }

  void _showContextMenu(BuildContext context, Offset position) {
    showMenu(
      context: context,
      position: RelativeRect.fromLTRB(
        position.dx,
        position.dy,
        position.dx + 200,
        position.dy + 200,
      ),
      items: [
        const PopupMenuItem(value: 'cut', child: ListTile(leading: Icon(Icons.cut), title: Text('Cut'))),
        const PopupMenuItem(value: 'copy', child: ListTile(leading: Icon(Icons.copy), title: Text('Copy'))),
        const PopupMenuItem(value: 'paste', child: ListTile(leading: Icon(Icons.paste), title: Text('Paste'))),
        const PopupMenuDivider(),
        const PopupMenuItem(value: 'delete', child: ListTile(leading: Icon(Icons.delete), title: Text('Delete'))),
      ],
    );
  }
}
```

---

**← [Part 33 - Flutter Web](part-33-flutter-web.md)**

**ต่อไป: [Part 35 - Accessibility →](part-35-accessibility.md)**
