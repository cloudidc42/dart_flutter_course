# Part 74: Flutter Desktop Advanced
## ขั้นตอนที่ 2841-2880

## 🎯 เป้าหมายของ Part นี้
- ใช้ system_tray package สำหรับ System Tray Integration
- สร้าง Native Menus ด้วย menubar
- จัดการ Multiple Windows
- ทำ File Associations
- สร้าง Auto-Updater Pattern
- Full working Flutter desktop code

---

## ขั้นตอนที่ 2841: Setup Flutter Desktop Project

```yaml
# pubspec.yaml
name: flutter_desktop_app
description: Advanced Flutter Desktop Application
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter
  system_tray: ^2.0.3
  window_manager: ^0.3.8
  desktop_multi_window: ^0.6.0
  package_info_plus: ^8.0.0
  path_provider: ^2.1.3
  shared_preferences: ^2.3.2
  file_picker: ^8.1.2
  url_launcher: ^6.3.1
  http: ^1.2.2
  archive: ^3.6.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0

flutter:
  uses-material-design: true
  assets:
    - assets/icons/
    - assets/images/
```

---

## ขั้นตอนที่ 2842: Window Manager Setup

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:window_manager/window_manager.dart';
import 'package:desktop_multi_window/desktop_multi_window.dart';
import 'screens/main_screen.dart';
import 'services/system_tray_service.dart';
import 'services/window_service.dart';

void main(List<String> args) async {
  WidgetsFlutterBinding.ensureInitialized();

  // Handle multi-window sub-windows
  if (args.firstOrNull == 'multi_window') {
    final windowId = int.parse(args[1]);
    final argument = args[2].isEmpty ? const {} : <String, dynamic>{};
    runApp(
      SubWindowApp(
        windowId: windowId,
        arguments: argument,
      ),
    );
    return;
  }

  // Initialize window manager for main window
  await windowManager.ensureInitialized();

  const windowOptions = WindowOptions(
    size: Size(1200, 800),
    minimumSize: Size(800, 600),
    center: true,
    backgroundColor: Colors.transparent,
    skipTaskbar: false,
    titleBarStyle: TitleBarStyle.normal,
    title: 'Flutter Desktop App',
  );

  windowManager.waitUntilReadyToShow(windowOptions, () async {
    await windowManager.show();
    await windowManager.focus();
  });

  runApp(const DesktopApp());
}

class DesktopApp extends StatefulWidget {
  const DesktopApp({super.key});

  @override
  State<DesktopApp> createState() => _DesktopAppState();
}

class _DesktopAppState extends State<DesktopApp> with WindowListener {
  @override
  void initState() {
    super.initState();
    windowManager.addListener(this);
    _initServices();
  }

  @override
  void dispose() {
    windowManager.removeListener(this);
    SystemTrayService.instance.dispose();
    super.dispose();
  }

  Future<void> _initServices() async {
    await SystemTrayService.instance.initialize();
  }

  @override
  void onWindowClose() async {
    // Instead of closing, minimize to tray
    final bool isPreventClose = await windowManager.isPreventClose();
    if (isPreventClose) {
      await windowManager.hide();
    }
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Desktop',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: Colors.blue,
          brightness: Brightness.light,
        ),
        useMaterial3: true,
      ),
      darkTheme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: Colors.blue,
          brightness: Brightness.dark,
        ),
        useMaterial3: true,
      ),
      themeMode: ThemeMode.system,
      home: const MainScreen(),
    );
  }
}
```

---

## ขั้นตอนที่ 2843: System Tray Service

```dart
// lib/services/system_tray_service.dart
import 'package:flutter/material.dart';
import 'package:system_tray/system_tray.dart';
import 'package:window_manager/window_manager.dart';

class SystemTrayService {
  SystemTrayService._();
  static final SystemTrayService instance = SystemTrayService._();

  late SystemTray _systemTray;
  late AppWindow _appWindow;
  bool _isInitialized = false;

  Future<void> initialize() async {
    if (_isInitialized) return;

    _systemTray = SystemTray();
    _appWindow = AppWindow();

    // Platform-specific icon path
    final String iconPath = _getPlatformIconPath();

    // Initialize system tray
    await _systemTray.initSystemTray(
      title: 'Flutter App',
      iconPath: iconPath,
      toolTip: 'Flutter Desktop App',
    );

    // Build context menu
    await _buildContextMenu();

    // Handle tray icon clicks
    _systemTray.registerSystemTrayEventHandler((eventName) async {
      if (eventName == kSystemTrayEventClick) {
        await _toggleWindow();
      } else if (eventName == kSystemTrayEventRightClick) {
        await _systemTray.popUpContextMenu();
      }
    });

    _isInitialized = true;
    debugPrint('SystemTrayService: Initialized');
  }

  String _getPlatformIconPath() {
    // Different icon for each platform
    if (Theme.of(_getContext()).platform == TargetPlatform.windows) {
      return 'assets/icons/app_icon.ico';
    } else if (Theme.of(_getContext()).platform == TargetPlatform.macOS) {
      return 'assets/icons/app_icon.png';
    } else {
      return 'assets/icons/app_icon.png'; // Linux
    }
  }

  BuildContext _getContext() {
    // In practice, pass context from app or use a global key
    throw UnimplementedError('Override with actual context');
  }

  Future<void> _buildContextMenu() async {
    final menu = Menu();
    await menu.buildFrom([
      MenuItemLabel(
        label: 'Show Window',
        onClicked: (_) async => await _showWindow(),
      ),
      MenuItemLabel(
        label: 'Hide Window',
        onClicked: (_) async => await windowManager.hide(),
      ),
      MenuSeparator(),
      MenuItemLabel(
        label: 'New Window',
        onClicked: (_) async => await WindowService.instance.openNewWindow(),
      ),
      MenuSeparator(),
      SubMenu(
        label: 'Settings',
        children: [
          MenuItemLabel(
            label: 'Preferences',
            onClicked: (_) async {
              await _showWindow();
              // Navigate to settings
            },
          ),
          MenuItemLabel(
            label: 'Check for Updates',
            onClicked: (_) async => await AutoUpdater.instance.checkForUpdates(),
          ),
        ],
      ),
      MenuSeparator(),
      MenuItemLabel(
        label: 'Quit',
        onClicked: (_) async {
          await _systemTray.destroy();
          await _appWindow.close();
        },
      ),
    ]);

    await _systemTray.setContextMenu(menu);
  }

  Future<void> _showWindow() async {
    await windowManager.show();
    await windowManager.focus();
  }

  Future<void> _toggleWindow() async {
    final isVisible = await windowManager.isVisible();
    if (isVisible) {
      await windowManager.hide();
    } else {
      await _showWindow();
    }
  }

  Future<void> updateTrayTitle(String title) async {
    await _systemTray.setTitle(title);
  }

  Future<void> showNotification(String title, String message) async {
    // System tray balloon/notification
    await _systemTray.setTitle(title);
    // In real app, use platform-specific notification packages
    debugPrint('Tray notification: $title - $message');
  }

  void dispose() {
    if (_isInitialized) {
      _systemTray.destroy();
      _isInitialized = false;
    }
  }
}
```

---

## ขั้นตอนที่ 2844: Native Menu Bar

```dart
// lib/widgets/native_menu_bar.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

/// Native-style menu bar for desktop applications
class AppMenuBar extends StatelessWidget {
  final Widget child;
  final VoidCallback? onNewFile;
  final VoidCallback? onOpenFile;
  final VoidCallback? onSaveFile;
  final VoidCallback? onQuit;
  final VoidCallback? onNewWindow;
  final VoidCallback? onAbout;
  final VoidCallback? onCheckUpdates;

  const AppMenuBar({
    super.key,
    required this.child,
    this.onNewFile,
    this.onOpenFile,
    this.onSaveFile,
    this.onQuit,
    this.onNewWindow,
    this.onAbout,
    this.onCheckUpdates,
  });

  @override
  Widget build(BuildContext context) {
    return PlatformMenuBar(
      menus: [
        PlatformMenu(
          label: 'File',
          menus: [
            PlatformMenuItem(
              label: 'New',
              shortcut: const SingleActivator(LogicalKeyboardKey.keyN, meta: true),
              onSelected: onNewFile,
            ),
            PlatformMenuItem(
              label: 'Open...',
              shortcut: const SingleActivator(LogicalKeyboardKey.keyO, meta: true),
              onSelected: onOpenFile,
            ),
            PlatformMenuItemGroup(
              members: [
                PlatformMenuItem(
                  label: 'Save',
                  shortcut: const SingleActivator(LogicalKeyboardKey.keyS, meta: true),
                  onSelected: onSaveFile,
                ),
              ],
            ),
            PlatformMenuItemGroup(
              members: [
                PlatformMenuItem(
                  label: 'Quit',
                  shortcut: const SingleActivator(LogicalKeyboardKey.keyQ, meta: true),
                  onSelected: onQuit,
                ),
              ],
            ),
          ],
        ),
        PlatformMenu(
          label: 'Edit',
          menus: [
            PlatformMenuItem(
              label: 'Undo',
              shortcut: const SingleActivator(LogicalKeyboardKey.keyZ, meta: true),
              onSelected: () {},
            ),
            PlatformMenuItem(
              label: 'Redo',
              shortcut: const SingleActivator(
                LogicalKeyboardKey.keyZ,
                meta: true,
                shift: true,
              ),
              onSelected: () {},
            ),
            PlatformMenuItemGroup(
              members: [
                PlatformMenuItem(
                  label: 'Cut',
                  shortcut: const SingleActivator(LogicalKeyboardKey.keyX, meta: true),
                  onSelected: () {},
                ),
                PlatformMenuItem(
                  label: 'Copy',
                  shortcut: const SingleActivator(LogicalKeyboardKey.keyC, meta: true),
                  onSelected: () {},
                ),
                PlatformMenuItem(
                  label: 'Paste',
                  shortcut: const SingleActivator(LogicalKeyboardKey.keyV, meta: true),
                  onSelected: () {},
                ),
              ],
            ),
          ],
        ),
        PlatformMenu(
          label: 'Window',
          menus: [
            PlatformMenuItem(
              label: 'New Window',
              shortcut: const SingleActivator(
                LogicalKeyboardKey.keyN,
                meta: true,
                shift: true,
              ),
              onSelected: onNewWindow,
            ),
          ],
        ),
        PlatformMenu(
          label: 'Help',
          menus: [
            PlatformMenuItem(
              label: 'Check for Updates...',
              onSelected: onCheckUpdates,
            ),
            PlatformMenuItemGroup(
              members: [
                PlatformMenuItem(
                  label: 'About',
                  onSelected: onAbout,
                ),
              ],
            ),
          ],
        ),
      ],
      child: child,
    );
  }
}
```

---

## ขั้นตอนที่ 2845: Multiple Windows Management

```dart
// lib/services/window_service.dart
import 'package:flutter/material.dart';
import 'package:desktop_multi_window/desktop_multi_window.dart';
import 'package:window_manager/window_manager.dart';

class WindowInfo {
  final int id;
  final String title;
  final Size size;
  final WindowController controller;

  WindowInfo({
    required this.id,
    required this.title,
    required this.size,
    required this.controller,
  });
}

class WindowService {
  WindowService._();
  static final WindowService instance = WindowService._();

  final Map<int, WindowInfo> _windows = {};
  int _windowCounter = 0;

  Future<WindowController> openNewWindow({
    String title = 'New Window',
    Size size = const Size(800, 600),
    Map<String, dynamic> args = const {},
  }) async {
    _windowCounter++;
    final windowId = _windowCounter;

    final controller = await DesktopMultiWindow.createWindow(
      jsonEncode({
        'window_id': windowId,
        'title': title,
        ...args,
      }),
    );

    controller
      ..setFrame(const Offset(100, 100) & size)
      ..center()
      ..setTitle(title)
      ..show();

    _windows[windowId] = WindowInfo(
      id: windowId,
      title: title,
      size: size,
      controller: controller,
    );

    debugPrint('WindowService: Opened window $windowId "$title"');
    return controller;
  }

  Future<WindowController> openDocumentWindow(String filePath) async {
    return openNewWindow(
      title: filePath.split('/').last,
      size: const Size(900, 700),
      args: {'file_path': filePath},
    );
  }

  Future<WindowController> openSettingsWindow() async {
    return openNewWindow(
      title: 'Settings',
      size: const Size(600, 500),
      args: {'window_type': 'settings'},
    );
  }

  Future<void> closeWindow(int windowId) async {
    final info = _windows[windowId];
    if (info != null) {
      await info.controller.close();
      _windows.remove(windowId);
      debugPrint('WindowService: Closed window $windowId');
    }
  }

  Future<void> focusWindow(int windowId) async {
    final info = _windows[windowId];
    if (info != null) {
      await info.controller.show();
    }
  }

  Future<void> closeAllWindows() async {
    for (final info in _windows.values) {
      await info.controller.close();
    }
    _windows.clear();
  }

  List<WindowInfo> get openWindows => _windows.values.toList();
  int get windowCount => _windows.length;
}

// Sub-window app widget
class SubWindowApp extends StatelessWidget {
  final int windowId;
  final Map<String, dynamic> arguments;

  const SubWindowApp({
    super.key,
    required this.windowId,
    required this.arguments,
  });

  @override
  Widget build(BuildContext context) {
    final windowType = arguments['window_type'] as String?;

    return MaterialApp(
      title: arguments['title'] as String? ?? 'Window $windowId',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      home: switch (windowType) {
        'settings' => const SettingsSubWindow(),
        _ => GenericSubWindow(windowId: windowId, arguments: arguments),
      },
    );
  }
}

class SettingsSubWindow extends StatelessWidget {
  const SettingsSubWindow({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Settings')),
      body: const _SettingsContent(),
    );
  }
}

class _SettingsContent extends StatelessWidget {
  const _SettingsContent();

  @override
  Widget build(BuildContext context) {
    return ListView(
      padding: const EdgeInsets.all(16),
      children: const [
        _SettingsSection(
          title: 'General',
          children: [
            _SettingsTile(
              title: 'Start on Login',
              subtitle: 'Launch app when system starts',
            ),
            _SettingsTile(
              title: 'Minimize to Tray',
              subtitle: 'Keep running in background',
            ),
          ],
        ),
        _SettingsSection(
          title: 'Appearance',
          children: [
            _SettingsTile(
              title: 'Theme',
              subtitle: 'System / Light / Dark',
            ),
          ],
        ),
      ],
    );
  }
}

class _SettingsSection extends StatelessWidget {
  final String title;
  final List<Widget> children;

  const _SettingsSection({required this.title, required this.children});

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Padding(
          padding: const EdgeInsets.symmetric(vertical: 8),
          child: Text(
            title,
            style: Theme.of(context).textTheme.titleMedium?.copyWith(
              fontWeight: FontWeight.bold,
              color: Theme.of(context).colorScheme.primary,
            ),
          ),
        ),
        ...children,
        const Divider(),
      ],
    );
  }
}

class _SettingsTile extends StatefulWidget {
  final String title;
  final String subtitle;

  const _SettingsTile({required this.title, required this.subtitle});

  @override
  State<_SettingsTile> createState() => _SettingsTileState();
}

class _SettingsTileState extends State<_SettingsTile> {
  bool _value = false;

  @override
  Widget build(BuildContext context) {
    return SwitchListTile(
      title: Text(widget.title),
      subtitle: Text(widget.subtitle),
      value: _value,
      onChanged: (v) => setState(() => _value = v),
    );
  }
}

class GenericSubWindow extends StatelessWidget {
  final int windowId;
  final Map<String, dynamic> arguments;

  const GenericSubWindow({
    super.key,
    required this.windowId,
    required this.arguments,
  });

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(arguments['title'] as String? ?? 'Window $windowId'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text('Window ID: $windowId'),
            const SizedBox(height: 8),
            Text('Arguments: $arguments'),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2846: File Associations

```dart
// lib/services/file_association_service.dart
import 'dart:io';
import 'package:file_picker/file_picker.dart';
import 'package:path/path.dart' as path;

/// Handles file type associations for the desktop app
class FileAssociationService {
  FileAssociationService._();
  static final FileAssociationService instance = FileAssociationService._();

  // Supported file extensions
  static const List<String> supportedExtensions = [
    'flutter',
    'dart',
    'json',
    'txt',
    'md',
  ];

  /// Open a file with the correct handler
  Future<FileOpenResult?> openFile(String filePath) async {
    final ext = path.extension(filePath).toLowerCase().replaceAll('.', '');

    if (!supportedExtensions.contains(ext)) {
      return FileOpenResult.unsupported(filePath, ext);
    }

    try {
      final file = File(filePath);
      if (!await file.exists()) {
        return FileOpenResult.notFound(filePath);
      }

      final content = await file.readAsString();
      return FileOpenResult.success(
        filePath: filePath,
        extension: ext,
        content: content,
        size: await file.length(),
      );
    } catch (e) {
      return FileOpenResult.error(filePath, e.toString());
    }
  }

  /// Pick a file via native file dialog
  Future<FileOpenResult?> pickAndOpenFile() async {
    final result = await FilePicker.platform.pickFiles(
      type: FileType.custom,
      allowedExtensions: supportedExtensions,
    );

    if (result == null || result.files.isEmpty) return null;

    final file = result.files.first;
    if (file.path == null) return null;

    return openFile(file.path!);
  }

  /// Save content to a file
  Future<String?> saveFile({
    String? defaultPath,
    String? content,
    List<String>? allowedExtensions,
  }) async {
    final savePath = await FilePicker.platform.saveFile(
      dialogTitle: 'Save File',
      fileName: defaultPath,
      type: FileType.custom,
      allowedExtensions: allowedExtensions ?? supportedExtensions,
    );

    if (savePath == null) return null;
    if (content != null) {
      await File(savePath).writeAsString(content);
    }
    return savePath;
  }

  /// Register file associations (macOS/Windows/Linux specific)
  /// This is typically done through platform-specific installer code
  Future<void> registerFileAssociations() async {
    // macOS: Done via Info.plist CFBundleDocumentTypes
    // Windows: Done via Registry (installer)
    // Linux: Done via .desktop file and mime.types
    // These are set at install time, not runtime
    print('File associations registered at install time');
  }
}

class FileOpenResult {
  final String filePath;
  final String? extension;
  final String? content;
  final int? size;
  final FileOpenStatus status;
  final String? errorMessage;

  FileOpenResult._({
    required this.filePath,
    this.extension,
    this.content,
    this.size,
    required this.status,
    this.errorMessage,
  });

  factory FileOpenResult.success({
    required String filePath,
    required String extension,
    required String content,
    required int size,
  }) => FileOpenResult._(
    filePath: filePath,
    extension: extension,
    content: content,
    size: size,
    status: FileOpenStatus.success,
  );

  factory FileOpenResult.unsupported(String filePath, String ext) =>
      FileOpenResult._(
        filePath: filePath,
        extension: ext,
        status: FileOpenStatus.unsupported,
        errorMessage: 'File type .$ext is not supported',
      );

  factory FileOpenResult.notFound(String filePath) => FileOpenResult._(
    filePath: filePath,
    status: FileOpenStatus.notFound,
    errorMessage: 'File not found: $filePath',
  );

  factory FileOpenResult.error(String filePath, String message) =>
      FileOpenResult._(
        filePath: filePath,
        status: FileOpenStatus.error,
        errorMessage: message,
      );

  bool get isSuccess => status == FileOpenStatus.success;
  String get fileName => path.basename(filePath);
}

enum FileOpenStatus { success, unsupported, notFound, error }
```

---

## ขั้นตอนที่ 2847: Auto-Updater Pattern

```dart
// lib/services/auto_updater.dart
import 'dart:convert';
import 'dart:io';
import 'package:flutter/foundation.dart';
import 'package:http/http.dart' as http;
import 'package:package_info_plus/package_info_plus.dart';
import 'package:path_provider/path_provider.dart';

class UpdateInfo {
  final String version;
  final String downloadUrl;
  final String releaseNotes;
  final int fileSize;
  final DateTime releaseDate;
  final bool isMandatory;

  UpdateInfo({
    required this.version,
    required this.downloadUrl,
    required this.releaseNotes,
    required this.fileSize,
    required this.releaseDate,
    required this.isMandatory,
  });

  factory UpdateInfo.fromJson(Map<String, dynamic> json) => UpdateInfo(
    version: json['version'] as String,
    downloadUrl: json['download_url'] as String,
    releaseNotes: json['release_notes'] as String,
    fileSize: json['file_size'] as int,
    releaseDate: DateTime.parse(json['release_date'] as String),
    isMandatory: json['is_mandatory'] as bool? ?? false,
  );
}

enum UpdateStatus { checking, upToDate, available, downloading, installing, error }

class AutoUpdater extends ChangeNotifier {
  AutoUpdater._();
  static final AutoUpdater instance = AutoUpdater._();

  // Replace with your actual update server URL
  static const String _updateServerUrl =
      'https://your-app.com/api/updates/latest';

  UpdateStatus _status = UpdateStatus.upToDate;
  UpdateInfo? _latestUpdate;
  double _downloadProgress = 0;
  String? _errorMessage;

  UpdateStatus get status => _status;
  UpdateInfo? get latestUpdate => _latestUpdate;
  double get downloadProgress => _downloadProgress;
  String? get errorMessage => _errorMessage;
  bool get hasUpdate => _latestUpdate != null;

  Future<UpdateInfo?> checkForUpdates() async {
    _setStatus(UpdateStatus.checking);

    try {
      final packageInfo = await PackageInfo.fromPlatform();
      final currentVersion = packageInfo.version;

      final response = await http.get(
        Uri.parse(_updateServerUrl),
        headers: {
          'Content-Type': 'application/json',
          'X-App-Version': currentVersion,
          'X-Platform': Platform.operatingSystem,
        },
      ).timeout(const Duration(seconds: 10));

      if (response.statusCode != 200) {
        throw HttpException('Server error: ${response.statusCode}');
      }

      final data = jsonDecode(response.body) as Map<String, dynamic>;
      final latestVersion = data['version'] as String;

      if (_isNewerVersion(latestVersion, currentVersion)) {
        _latestUpdate = UpdateInfo.fromJson(data);
        _setStatus(UpdateStatus.available);
        debugPrint('Update available: $latestVersion (current: $currentVersion)');
        return _latestUpdate;
      } else {
        _setStatus(UpdateStatus.upToDate);
        debugPrint('App is up to date: $currentVersion');
        return null;
      }
    } catch (e) {
      _errorMessage = e.toString();
      _setStatus(UpdateStatus.error);
      debugPrint('Update check failed: $e');
      return null;
    }
  }

  Future<bool> downloadAndInstall() async {
    if (_latestUpdate == null) return false;

    try {
      _setStatus(UpdateStatus.downloading);
      _downloadProgress = 0;
      notifyListeners();

      // Download the update package
      final downloadDir = await getTemporaryDirectory();
      final fileName = _latestUpdate!.downloadUrl.split('/').last;
      final savePath = '${downloadDir.path}/$fileName';

      final request = http.Request('GET', Uri.parse(_latestUpdate!.downloadUrl));
      final response = await request.send();

      final total = response.contentLength ?? _latestUpdate!.fileSize;
      int received = 0;

      final file = File(savePath);
      final sink = file.openWrite();

      await response.stream.forEach((chunk) {
        received += chunk.length;
        sink.add(chunk);
        _downloadProgress = received / total;
        notifyListeners();
      });

      await sink.close();

      _setStatus(UpdateStatus.installing);
      await _installUpdate(savePath);
      return true;
    } catch (e) {
      _errorMessage = e.toString();
      _setStatus(UpdateStatus.error);
      debugPrint('Download failed: $e');
      return false;
    }
  }

  Future<void> _installUpdate(String packagePath) async {
    // Platform-specific installation
    if (Platform.isWindows) {
      await Process.run(packagePath, ['/SILENT', '/NORESTART']);
    } else if (Platform.isMacOS) {
      // Open the DMG/PKG installer
      await Process.run('open', [packagePath]);
    } else if (Platform.isLinux) {
      await Process.run('xdg-open', [packagePath]);
    }
  }

  bool _isNewerVersion(String latest, String current) {
    final latestParts = latest.split('.').map(int.parse).toList();
    final currentParts = current.split('.').map(int.parse).toList();

    for (int i = 0; i < 3; i++) {
      final l = i < latestParts.length ? latestParts[i] : 0;
      final c = i < currentParts.length ? currentParts[i] : 0;
      if (l > c) return true;
      if (l < c) return false;
    }
    return false;
  }

  void _setStatus(UpdateStatus status) {
    _status = status;
    notifyListeners();
  }
}
```

---

## ขั้นตอนที่ 2848: Update Dialog Widget

```dart
// lib/widgets/update_dialog.dart
import 'package:flutter/material.dart';
import '../services/auto_updater.dart';

class UpdateDialog extends StatelessWidget {
  final UpdateInfo updateInfo;

  const UpdateDialog({super.key, required this.updateInfo});

  static Future<bool?> show(BuildContext context, UpdateInfo updateInfo) {
    return showDialog<bool>(
      context: context,
      barrierDismissible: false,
      builder: (_) => UpdateDialog(updateInfo: updateInfo),
    );
  }

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: const Row(
        children: [
          Icon(Icons.system_update, color: Colors.blue),
          SizedBox(width: 8),
          Text('Update Available'),
        ],
      ),
      content: SizedBox(
        width: 400,
        child: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Version ${updateInfo.version} is available',
              style: const TextStyle(fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            Text(
              'Released: ${_formatDate(updateInfo.releaseDate)}',
              style: const TextStyle(color: Colors.grey, fontSize: 13),
            ),
            const SizedBox(height: 16),
            const Text(
              'Release Notes:',
              style: TextStyle(fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            Container(
              padding: const EdgeInsets.all(12),
              decoration: BoxDecoration(
                color: Colors.grey[100],
                borderRadius: BorderRadius.circular(8),
              ),
              child: Text(
                updateInfo.releaseNotes,
                style: const TextStyle(fontSize: 13),
              ),
            ),
            if (updateInfo.isMandatory) ...[
              const SizedBox(height: 12),
              const Row(
                children: [
                  Icon(Icons.warning, color: Colors.orange, size: 16),
                  SizedBox(width: 4),
                  Text(
                    'This update is required to continue',
                    style: TextStyle(color: Colors.orange, fontSize: 12),
                  ),
                ],
              ),
            ],
          ],
        ),
      ),
      actions: [
        if (!updateInfo.isMandatory)
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('Later'),
          ),
        ElevatedButton(
          onPressed: () => Navigator.pop(context, true),
          child: const Text('Update Now'),
        ),
      ],
    );
  }

  String _formatDate(DateTime date) {
    return '${date.year}-${date.month.toString().padLeft(2, '0')}-${date.day.toString().padLeft(2, '0')}';
  }
}

class UpdateProgressDialog extends StatelessWidget {
  const UpdateProgressDialog({super.key});

  static void show(BuildContext context) {
    showDialog(
      context: context,
      barrierDismissible: false,
      builder: (_) => const UpdateProgressDialog(),
    );
  }

  @override
  Widget build(BuildContext context) {
    return ListenableBuilder(
      listenable: AutoUpdater.instance,
      builder: (context, _) {
        final updater = AutoUpdater.instance;

        return AlertDialog(
          title: const Text('Updating...'),
          content: SizedBox(
            width: 350,
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                if (updater.status == UpdateStatus.downloading) ...[
                  Text(
                    '${(updater.downloadProgress * 100).toStringAsFixed(1)}%',
                    style: const TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
                  ),
                  const SizedBox(height: 8),
                  LinearProgressIndicator(value: updater.downloadProgress),
                  const SizedBox(height: 8),
                  const Text('Downloading update...'),
                ] else if (updater.status == UpdateStatus.installing) ...[
                  const CircularProgressIndicator(),
                  const SizedBox(height: 16),
                  const Text('Installing update...'),
                ] else if (updater.status == UpdateStatus.error) ...[
                  const Icon(Icons.error, color: Colors.red, size: 48),
                  const SizedBox(height: 8),
                  Text(
                    updater.errorMessage ?? 'Update failed',
                    style: const TextStyle(color: Colors.red),
                  ),
                ],
              ],
            ),
          ),
          actions: [
            if (updater.status == UpdateStatus.error)
              ElevatedButton(
                onPressed: () => Navigator.pop(context),
                child: const Text('Close'),
              ),
          ],
        );
      },
    );
  }
}
```

---

## ขั้นตอนที่ 2849: Main Desktop Screen

```dart
// lib/screens/main_screen.dart
import 'package:flutter/material.dart';
import 'package:window_manager/window_manager.dart';
import '../services/file_association_service.dart';
import '../services/window_service.dart';
import '../services/auto_updater.dart';
import '../widgets/native_menu_bar.dart';
import '../widgets/update_dialog.dart';

class MainScreen extends StatefulWidget {
  const MainScreen({super.key});

  @override
  State<MainScreen> createState() => _MainScreenState();
}

class _MainScreenState extends State<MainScreen> with WindowListener {
  String _statusMessage = 'Ready';
  List<String> _recentFiles = [];
  int _selectedTab = 0;

  @override
  void initState() {
    super.initState();
    windowManager.addListener(this);
    _checkForUpdatesOnStartup();
  }

  @override
  void dispose() {
    windowManager.removeListener(this);
    super.dispose();
  }

  Future<void> _checkForUpdatesOnStartup() async {
    await Future.delayed(const Duration(seconds: 3));
    final update = await AutoUpdater.instance.checkForUpdates();
    if (update != null && mounted) {
      final shouldUpdate = await UpdateDialog.show(context, update);
      if (shouldUpdate == true) {
        UpdateProgressDialog.show(context);
        await AutoUpdater.instance.downloadAndInstall();
      }
    }
  }

  Future<void> _openFile() async {
    final result = await FileAssociationService.instance.pickAndOpenFile();
    if (result?.isSuccess == true) {
      setState(() {
        _statusMessage = 'Opened: ${result!.fileName}';
        _recentFiles = [result.filePath, ..._recentFiles.take(9)];
      });
    }
  }

  Future<void> _newWindow() async {
    await WindowService.instance.openNewWindow(
      title: 'Document ${WindowService.instance.windowCount + 1}',
    );
  }

  @override
  Widget build(BuildContext context) {
    return AppMenuBar(
      onNewFile: () => setState(() => _statusMessage = 'New file created'),
      onOpenFile: _openFile,
      onSaveFile: () => setState(() => _statusMessage = 'File saved'),
      onQuit: () async {
        await windowManager.destroy();
      },
      onNewWindow: _newWindow,
      onAbout: () => _showAboutDialog(),
      onCheckUpdates: () async {
        final update = await AutoUpdater.instance.checkForUpdates();
        if (mounted) {
          if (update != null) {
            await UpdateDialog.show(context, update);
          } else {
            ScaffoldMessenger.of(context).showSnackBar(
              const SnackBar(content: Text('You have the latest version!')),
            );
          }
        }
      },
      child: Scaffold(
        body: Column(
          children: [
            _buildToolbar(),
            Expanded(
              child: Row(
                children: [
                  _buildSidebar(),
                  const VerticalDivider(width: 1),
                  Expanded(child: _buildContent()),
                ],
              ),
            ),
            _buildStatusBar(),
          ],
        ),
      ),
    );
  }

  Widget _buildToolbar() {
    return Container(
      height: 48,
      padding: const EdgeInsets.symmetric(horizontal: 8),
      decoration: BoxDecoration(
        color: Theme.of(context).colorScheme.surface,
        border: Border(
          bottom: BorderSide(color: Theme.of(context).dividerColor),
        ),
      ),
      child: Row(
        children: [
          IconButton(
            icon: const Icon(Icons.add),
            tooltip: 'New File',
            onPressed: () => setState(() => _statusMessage = 'New file'),
          ),
          IconButton(
            icon: const Icon(Icons.folder_open),
            tooltip: 'Open File',
            onPressed: _openFile,
          ),
          IconButton(
            icon: const Icon(Icons.save),
            tooltip: 'Save',
            onPressed: () => setState(() => _statusMessage = 'Saved'),
          ),
          const VerticalDivider(height: 32),
          IconButton(
            icon: const Icon(Icons.open_in_new),
            tooltip: 'New Window',
            onPressed: _newWindow,
          ),
          IconButton(
            icon: const Icon(Icons.settings),
            tooltip: 'Settings',
            onPressed: () => WindowService.instance.openSettingsWindow(),
          ),
          const Spacer(),
          // Window controls indicator
          Text(
            '${WindowService.instance.windowCount + 1} window(s)',
            style: TextStyle(
              color: Theme.of(context).colorScheme.onSurfaceVariant,
              fontSize: 12,
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildSidebar() {
    return SizedBox(
      width: 200,
      child: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(8),
            child: Text(
              'Recent Files',
              style: Theme.of(context).textTheme.labelMedium?.copyWith(
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
          Expanded(
            child: _recentFiles.isEmpty
                ? const Center(
                    child: Text(
                      'No recent files',
                      style: TextStyle(color: Colors.grey),
                    ),
                  )
                : ListView.builder(
                    itemCount: _recentFiles.length,
                    itemBuilder: (context, index) {
                      final file = _recentFiles[index];
                      return ListTile(
                        dense: true,
                        leading: const Icon(Icons.insert_drive_file, size: 16),
                        title: Text(
                          file.split('/').last,
                          style: const TextStyle(fontSize: 12),
                          overflow: TextOverflow.ellipsis,
                        ),
                        onTap: () async {
                          final result = await FileAssociationService.instance
                              .openFile(file);
                          if (result?.isSuccess == true) {
                            setState(() => _statusMessage = 'Opened: ${result!.fileName}');
                          }
                        },
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }

  Widget _buildContent() {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(
            Icons.flutter_dash,
            size: 80,
            color: Theme.of(context).colorScheme.primary,
          ),
          const SizedBox(height: 16),
          Text(
            'Flutter Desktop App',
            style: Theme.of(context).textTheme.headlineMedium,
          ),
          const SizedBox(height: 8),
          const Text(
            'Open a file or create a new one to get started',
            style: TextStyle(color: Colors.grey),
          ),
          const SizedBox(height: 24),
          Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              ElevatedButton.icon(
                onPressed: () => setState(() => _statusMessage = 'New file created'),
                icon: const Icon(Icons.add),
                label: const Text('New File'),
              ),
              const SizedBox(width: 12),
              OutlinedButton.icon(
                onPressed: _openFile,
                icon: const Icon(Icons.folder_open),
                label: const Text('Open File'),
              ),
            ],
          ),
        ],
      ),
    );
  }

  Widget _buildStatusBar() {
    return Container(
      height: 24,
      padding: const EdgeInsets.symmetric(horizontal: 8),
      decoration: BoxDecoration(
        color: Theme.of(context).colorScheme.surfaceContainerHighest,
        border: Border(
          top: BorderSide(color: Theme.of(context).dividerColor),
        ),
      ),
      child: Row(
        children: [
          Text(
            _statusMessage,
            style: TextStyle(
              fontSize: 11,
              color: Theme.of(context).colorScheme.onSurfaceVariant,
            ),
          ),
          const Spacer(),
          ListenableBuilder(
            listenable: AutoUpdater.instance,
            builder: (_, __) {
              if (AutoUpdater.instance.status != UpdateStatus.available) {
                return const SizedBox.shrink();
              }
              return GestureDetector(
                onTap: () => UpdateDialog.show(
                  context,
                  AutoUpdater.instance.latestUpdate!,
                ),
                child: const Row(
                  children: [
                    Icon(Icons.system_update, size: 12, color: Colors.blue),
                    SizedBox(width: 4),
                    Text(
                      'Update available',
                      style: TextStyle(fontSize: 11, color: Colors.blue),
                    ),
                  ],
                ),
              );
            },
          ),
        ],
      ),
    );
  }

  void _showAboutDialog() {
    showAboutDialog(
      context: context,
      applicationName: 'Flutter Desktop App',
      applicationVersion: '1.0.0',
      applicationIcon: const Icon(Icons.flutter_dash, size: 48),
      children: const [
        Text('A demo Flutter desktop application showcasing advanced features.'),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 2850: macOS entitlements และ Platform Setup

```xml
<!-- macos/Runner/DebugProfile.entitlements -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.app-sandbox</key>
    <true/>
    <key>com.apple.security.cs.allow-jit</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
    <key>com.apple.security.files.downloads.read-write</key>
    <true/>
</dict>
</plist>
```

```xml
<!-- macos/Runner/Info.plist - File associations -->
<key>CFBundleDocumentTypes</key>
<array>
    <dict>
        <key>CFBundleTypeName</key>
        <string>Flutter Project File</string>
        <key>CFBundleTypeExtensions</key>
        <array>
            <string>flutter</string>
        </array>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>LSIsAppleDefaultForType</key>
        <true/>
    </dict>
    <dict>
        <key>CFBundleTypeName</key>
        <string>Dart File</string>
        <key>CFBundleTypeExtensions</key>
        <array>
            <string>dart</string>
        </array>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
    </dict>
</array>
```

---

**← [Part 73](part-73-state-management-comparison-project.md)**
**ต่อไป: [Part 75 →](part-75-monetization-strategies.md)**
