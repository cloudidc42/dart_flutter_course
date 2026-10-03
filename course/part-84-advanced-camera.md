# Part 84: Advanced Camera
## ขั้นตอนที่ 3241-3280

## 🎯 เป้าหมายของ Part นี้
- ใช้ camera package แบบ advanced
- สร้าง Custom Camera UI พร้อม exposure, zoom, และ flash
- QR/Barcode Scanner ด้วย mobile_scanner
- Document Scanner
- Video Recording พร้อม controls

---

## ขั้นตอนที่ 3241: ติดตั้งและตั้งค่า Camera Package

```yaml
# pubspec.yaml
name: advanced_camera_demo
description: Advanced Camera features demo

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: ">=3.10.0"

dependencies:
  flutter:
    sdk: flutter
  camera: ^0.10.5+9
  mobile_scanner: ^4.0.0
  permission_handler: ^11.1.0
  path_provider: ^2.1.1
  image_gallery_saver: ^2.0.3
  video_player: ^2.8.2
  gallery_saver: ^2.3.2
  image: ^4.1.3

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
```

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.CAMERA"/>
<uses-permission android:name="android.permission.RECORD_AUDIO"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
```

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'screens/camera_home_screen.dart';

void main() {
  runApp(const CameraDemoApp());
}

class CameraDemoApp extends StatelessWidget {
  const CameraDemoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Advanced Camera',
      theme: ThemeData.dark().copyWith(
        colorScheme: const ColorScheme.dark(primary: Colors.amber),
        useMaterial3: true,
      ),
      home: const CameraHomeScreen(),
    );
  }
}
```

---

## ขั้นตอนที่ 3242: Camera Home Screen

```dart
// lib/screens/camera_home_screen.dart
import 'package:flutter/material.dart';
import 'custom_camera_screen.dart';
import 'qr_scanner_screen.dart';
import 'document_scanner_screen.dart';
import 'video_recorder_screen.dart';

class CameraHomeScreen extends StatelessWidget {
  const CameraHomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      appBar: AppBar(
        title: const Text('Advanced Camera'),
        backgroundColor: Colors.black,
        foregroundColor: Colors.white,
      ),
      body: GridView.count(
        crossAxisCount: 2,
        padding: const EdgeInsets.all(16),
        mainAxisSpacing: 16,
        crossAxisSpacing: 16,
        children: [
          _CameraFeatureCard(
            title: 'Custom Camera',
            subtitle: 'Exposure, Zoom, Flash',
            icon: Icons.camera_alt,
            gradient: const LinearGradient(
              colors: [Colors.blue, Colors.purple],
            ),
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const CustomCameraScreen()),
            ),
          ),
          _CameraFeatureCard(
            title: 'QR Scanner',
            subtitle: 'Barcode & QR Code',
            icon: Icons.qr_code_scanner,
            gradient: const LinearGradient(
              colors: [Colors.green, Colors.teal],
            ),
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const QRScannerScreen()),
            ),
          ),
          _CameraFeatureCard(
            title: 'Doc Scanner',
            subtitle: 'Document Scanning',
            icon: Icons.document_scanner,
            gradient: const LinearGradient(
              colors: [Colors.orange, Colors.red],
            ),
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const DocumentScannerScreen()),
            ),
          ),
          _CameraFeatureCard(
            title: 'Video Recorder',
            subtitle: 'Record with controls',
            icon: Icons.videocam,
            gradient: const LinearGradient(
              colors: [Colors.pink, Colors.deepOrange],
            ),
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const VideoRecorderScreen()),
            ),
          ),
        ],
      ),
    );
  }
}

class _CameraFeatureCard extends StatelessWidget {
  final String title;
  final String subtitle;
  final IconData icon;
  final Gradient gradient;
  final VoidCallback onTap;

  const _CameraFeatureCard({
    required this.title,
    required this.subtitle,
    required this.icon,
    required this.gradient,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      child: Container(
        decoration: BoxDecoration(
          gradient: gradient,
          borderRadius: BorderRadius.circular(16),
          boxShadow: [
            BoxShadow(
              color: Colors.black.withOpacity(0.3),
              blurRadius: 8,
              offset: const Offset(0, 4),
            ),
          ],
        ),
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Icon(icon, size: 48, color: Colors.white),
              const SizedBox(height: 12),
              Text(
                title,
                style: const TextStyle(
                  color: Colors.white,
                  fontSize: 16,
                  fontWeight: FontWeight.bold,
                ),
              ),
              const SizedBox(height: 4),
              Text(
                subtitle,
                style: const TextStyle(
                  color: Colors.white70,
                  fontSize: 12,
                ),
                textAlign: TextAlign.center,
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

## ขั้นตอนที่ 3243: Custom Camera Screen พร้อม Advanced Controls

```dart
// lib/screens/custom_camera_screen.dart
import 'dart:async';
import 'dart:io';
import 'package:camera/camera.dart';
import 'package:flutter/material.dart';
import 'package:path_provider/path_provider.dart';
import 'package:permission_handler/permission_handler.dart';
import '../widgets/camera_controls_overlay.dart';
import '../widgets/camera_exposure_slider.dart';

class CustomCameraScreen extends StatefulWidget {
  const CustomCameraScreen({super.key});

  @override
  State<CustomCameraScreen> createState() => _CustomCameraScreenState();
}

class _CustomCameraScreenState extends State<CustomCameraScreen>
    with WidgetsBindingObserver {
  CameraController? _controller;
  List<CameraDescription> _cameras = [];
  int _selectedCameraIndex = 0;
  bool _isInitialized = false;
  bool _isTakingPhoto = false;

  // Camera settings
  FlashMode _flashMode = FlashMode.off;
  double _currentZoom = 1.0;
  double _minZoom = 1.0;
  double _maxZoom = 1.0;
  double _exposureOffset = 0.0;
  double _minExposure = -2.0;
  double _maxExposure = 2.0;
  bool _showExposureSlider = false;
  bool _showZoomSlider = false;
  ResolutionPreset _resolution = ResolutionPreset.high;

  // Focus and exposure point
  Offset? _focusPoint;
  Timer? _focusTimer;

  // Photo gallery
  final List<String> _capturedPhotos = [];

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    _initCamera();
  }

  Future<void> _initCamera() async {
    final status = await Permission.camera.request();
    if (status.isDenied) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Camera permission required')),
        );
      }
      return;
    }

    _cameras = await availableCameras();
    if (_cameras.isEmpty) return;

    await _setupCamera(_cameras[_selectedCameraIndex]);
  }

  Future<void> _setupCamera(CameraDescription camera) async {
    _controller?.dispose();

    final controller = CameraController(
      camera,
      _resolution,
      enableAudio: false,
      imageFormatGroup: ImageFormatGroup.jpeg,
    );

    _controller = controller;

    try {
      await controller.initialize();

      // Get zoom range
      _minZoom = await controller.getMinZoomLevel();
      _maxZoom = await controller.getMaxZoomLevel();

      // Get exposure range
      _minExposure = await controller.getMinExposureOffset();
      _maxExposure = await controller.getMaxExposureOffset();

      // Set initial flash
      await controller.setFlashMode(_flashMode);

      if (mounted) {
        setState(() {
          _isInitialized = true;
          _currentZoom = _minZoom;
        });
      }
    } on CameraException catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Camera error: ${e.description}')),
        );
      }
    }
  }

  Future<void> _switchCamera() async {
    if (_cameras.length < 2) return;

    setState(() {
      _isInitialized = false;
      _selectedCameraIndex = (_selectedCameraIndex + 1) % _cameras.length;
    });

    await _setupCamera(_cameras[_selectedCameraIndex]);
  }

  Future<void> _setFlashMode(FlashMode mode) async {
    await _controller?.setFlashMode(mode);
    setState(() => _flashMode = mode);
  }

  Future<void> _setZoom(double zoom) async {
    final clampedZoom = zoom.clamp(_minZoom, _maxZoom);
    await _controller?.setZoomLevel(clampedZoom);
    setState(() => _currentZoom = clampedZoom);
  }

  Future<void> _setExposure(double offset) async {
    final clampedOffset = offset.clamp(_minExposure, _maxExposure);
    await _controller?.setExposureOffset(clampedOffset);
    setState(() => _exposureOffset = clampedOffset);
  }

  Future<void> _handleTap(TapDownDetails details) async {
    if (_controller == null || !_controller!.value.isInitialized) return;

    final renderBox = context.findRenderObject() as RenderBox?;
    if (renderBox == null) return;

    final offset = details.localPosition;
    final size = renderBox.size;

    final x = offset.dx / size.width;
    final y = offset.dy / size.height;

    setState(() {
      _focusPoint = offset;
      _showExposureSlider = false;
      _showZoomSlider = false;
    });

    await _controller!.setFocusPoint(Offset(x, y));
    await _controller!.setExposurePoint(Offset(x, y));

    // Hide focus indicator after delay
    _focusTimer?.cancel();
    _focusTimer = Timer(const Duration(seconds: 2), () {
      if (mounted) setState(() => _focusPoint = null);
    });
  }

  Future<void> _takePicture() async {
    if (_controller == null || !_controller!.value.isInitialized || _isTakingPhoto) {
      return;
    }

    setState(() => _isTakingPhoto = true);

    try {
      final XFile file = await _controller!.takePicture();

      // Save to a more accessible location
      final directory = await getTemporaryDirectory();
      final timestamp = DateTime.now().millisecondsSinceEpoch;
      final path = '${directory.path}/photo_$timestamp.jpg';

      await File(file.path).copy(path);

      setState(() => _capturedPhotos.insert(0, path));

      if (mounted) {
        _showCaptureAnimation();
      }
    } on CameraException catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Failed to take photo: ${e.description}')),
        );
      }
    } finally {
      if (mounted) setState(() => _isTakingPhoto = false);
    }
  }

  void _showCaptureAnimation() {
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(
        content: Row(
          children: [
            Icon(Icons.check_circle, color: Colors.green),
            SizedBox(width: 8),
            Text('Photo captured!'),
          ],
        ),
        duration: Duration(seconds: 1),
        behavior: SnackBarBehavior.floating,
      ),
    );
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (_controller == null || !_controller!.value.isInitialized) return;

    if (state == AppLifecycleState.inactive) {
      _controller?.dispose();
    } else if (state == AppLifecycleState.resumed) {
      _setupCamera(_cameras[_selectedCameraIndex]);
    }
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    _focusTimer?.cancel();
    _controller?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      body: SafeArea(
        child: !_isInitialized || _controller == null
            ? const Center(child: CircularProgressIndicator(color: Colors.white))
            : Stack(
                children: [
                  // Camera Preview
                  GestureDetector(
                    onTapDown: _handleTap,
                    onScaleUpdate: (details) {
                      final newZoom = _currentZoom * details.scale;
                      _setZoom(newZoom);
                    },
                    child: SizedBox.expand(
                      child: CameraPreview(_controller!),
                    ),
                  ),

                  // Focus indicator
                  if (_focusPoint != null)
                    Positioned(
                      left: _focusPoint!.dx - 30,
                      top: _focusPoint!.dy - 30,
                      child: Container(
                        width: 60,
                        height: 60,
                        decoration: BoxDecoration(
                          border: Border.all(color: Colors.yellow, width: 2),
                          borderRadius: BorderRadius.circular(4),
                        ),
                      ),
                    ),

                  // Top controls
                  Positioned(
                    top: 0,
                    left: 0,
                    right: 0,
                    child: _TopControls(
                      flashMode: _flashMode,
                      onFlashChanged: _setFlashMode,
                      onSwitchCamera: _switchCamera,
                      hasMultipleCameras: _cameras.length > 1,
                      onClose: () => Navigator.pop(context),
                    ),
                  ),

                  // Zoom slider
                  if (_showZoomSlider)
                    Positioned(
                      left: 16,
                      top: 100,
                      bottom: 100,
                      child: _ZoomSlider(
                        currentZoom: _currentZoom,
                        minZoom: _minZoom,
                        maxZoom: _maxZoom,
                        onChanged: _setZoom,
                      ),
                    ),

                  // Exposure slider
                  if (_showExposureSlider)
                    Positioned(
                      right: 16,
                      top: 100,
                      bottom: 100,
                      child: CameraExposureSlider(
                        currentExposure: _exposureOffset,
                        minExposure: _minExposure,
                        maxExposure: _maxExposure,
                        onChanged: _setExposure,
                      ),
                    ),

                  // Bottom controls
                  Positioned(
                    bottom: 0,
                    left: 0,
                    right: 0,
                    child: _BottomControls(
                      onCapture: _takePicture,
                      isTakingPhoto: _isTakingPhoto,
                      lastPhotoPath: _capturedPhotos.isNotEmpty
                          ? _capturedPhotos.first
                          : null,
                      onToggleZoom: () => setState(() {
                        _showZoomSlider = !_showZoomSlider;
                        _showExposureSlider = false;
                      }),
                      onToggleExposure: () => setState(() {
                        _showExposureSlider = !_showExposureSlider;
                        _showZoomSlider = false;
                      }),
                      currentZoom: _currentZoom,
                      isZoomActive: _showZoomSlider,
                      isExposureActive: _showExposureSlider,
                    ),
                  ),

                  // Photo count badge
                  if (_capturedPhotos.isNotEmpty)
                    Positioned(
                      bottom: 110,
                      right: 24,
                      child: Container(
                        padding: const EdgeInsets.symmetric(
                          horizontal: 8,
                          vertical: 4,
                        ),
                        decoration: BoxDecoration(
                          color: Colors.black54,
                          borderRadius: BorderRadius.circular(12),
                        ),
                        child: Text(
                          '${_capturedPhotos.length} photos',
                          style: const TextStyle(
                            color: Colors.white,
                            fontSize: 12,
                          ),
                        ),
                      ),
                    ),
                ],
              ),
      ),
    );
  }
}

class _TopControls extends StatelessWidget {
  final FlashMode flashMode;
  final ValueChanged<FlashMode> onFlashChanged;
  final VoidCallback onSwitchCamera;
  final bool hasMultipleCameras;
  final VoidCallback onClose;

  const _TopControls({
    required this.flashMode,
    required this.onFlashChanged,
    required this.onSwitchCamera,
    required this.hasMultipleCameras,
    required this.onClose,
  });

  IconData get _flashIcon {
    switch (flashMode) {
      case FlashMode.off: return Icons.flash_off;
      case FlashMode.auto: return Icons.flash_auto;
      case FlashMode.always: return Icons.flash_on;
      case FlashMode.torch: return Icons.highlight;
    }
  }

  FlashMode get _nextFlashMode {
    switch (flashMode) {
      case FlashMode.off: return FlashMode.auto;
      case FlashMode.auto: return FlashMode.always;
      case FlashMode.always: return FlashMode.torch;
      case FlashMode.torch: return FlashMode.off;
    }
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      decoration: BoxDecoration(
        gradient: LinearGradient(
          begin: Alignment.topCenter,
          end: Alignment.bottomCenter,
          colors: [Colors.black.withOpacity(0.6), Colors.transparent],
        ),
      ),
      child: Row(
        children: [
          IconButton(
            icon: const Icon(Icons.close, color: Colors.white),
            onPressed: onClose,
          ),
          const Spacer(),
          IconButton(
            icon: Icon(_flashIcon, color: Colors.white),
            onPressed: () => onFlashChanged(_nextFlashMode),
          ),
          if (hasMultipleCameras)
            IconButton(
              icon: const Icon(Icons.cameraswitch, color: Colors.white),
              onPressed: onSwitchCamera,
            ),
        ],
      ),
    );
  }
}

class _BottomControls extends StatelessWidget {
  final VoidCallback onCapture;
  final bool isTakingPhoto;
  final String? lastPhotoPath;
  final VoidCallback onToggleZoom;
  final VoidCallback onToggleExposure;
  final double currentZoom;
  final bool isZoomActive;
  final bool isExposureActive;

  const _BottomControls({
    required this.onCapture,
    required this.isTakingPhoto,
    this.lastPhotoPath,
    required this.onToggleZoom,
    required this.onToggleExposure,
    required this.currentZoom,
    required this.isZoomActive,
    required this.isExposureActive,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.fromLTRB(24, 16, 24, 32),
      decoration: BoxDecoration(
        gradient: LinearGradient(
          begin: Alignment.bottomCenter,
          end: Alignment.topCenter,
          colors: [Colors.black.withOpacity(0.8), Colors.transparent],
        ),
      ),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          // Secondary controls row
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceEvenly,
            children: [
              _ControlChip(
                icon: Icons.zoom_in,
                label: '${currentZoom.toStringAsFixed(1)}x',
                isActive: isZoomActive,
                onTap: onToggleZoom,
              ),
              _ControlChip(
                icon: Icons.exposure,
                label: 'EV',
                isActive: isExposureActive,
                onTap: onToggleExposure,
              ),
            ],
          ),
          const SizedBox(height: 16),

          // Main controls row
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceEvenly,
            crossAxisAlignment: CrossAxisAlignment.center,
            children: [
              // Last photo thumbnail
              if (lastPhotoPath != null)
                GestureDetector(
                  onTap: () {/* Open gallery */},
                  child: ClipRRect(
                    borderRadius: BorderRadius.circular(8),
                    child: Image.file(
                      File(lastPhotoPath!),
                      width: 52,
                      height: 52,
                      fit: BoxFit.cover,
                    ),
                  ),
                )
              else
                const SizedBox(width: 52),

              // Capture button
              GestureDetector(
                onTap: isTakingPhoto ? null : onCapture,
                child: Container(
                  width: 72,
                  height: 72,
                  decoration: BoxDecoration(
                    shape: BoxShape.circle,
                    border: Border.all(color: Colors.white, width: 4),
                  ),
                  child: AnimatedContainer(
                    duration: const Duration(milliseconds: 100),
                    margin: const EdgeInsets.all(4),
                    decoration: BoxDecoration(
                      color: isTakingPhoto ? Colors.grey : Colors.white,
                      shape: BoxShape.circle,
                    ),
                  ),
                ),
              ),

              const SizedBox(width: 52),
            ],
          ),
        ],
      ),
    );
  }
}

class _ControlChip extends StatelessWidget {
  final IconData icon;
  final String label;
  final bool isActive;
  final VoidCallback onTap;

  const _ControlChip({
    required this.icon,
    required this.label,
    required this.isActive,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      child: AnimatedContainer(
        duration: const Duration(milliseconds: 200),
        padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
        decoration: BoxDecoration(
          color: isActive
              ? Colors.amber.withOpacity(0.3)
              : Colors.white.withOpacity(0.1),
          borderRadius: BorderRadius.circular(16),
          border: isActive ? Border.all(color: Colors.amber) : null,
        ),
        child: Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(icon, color: isActive ? Colors.amber : Colors.white, size: 16),
            const SizedBox(width: 4),
            Text(
              label,
              style: TextStyle(
                color: isActive ? Colors.amber : Colors.white,
                fontSize: 12,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _ZoomSlider extends StatelessWidget {
  final double currentZoom;
  final double minZoom;
  final double maxZoom;
  final ValueChanged<double> onChanged;

  const _ZoomSlider({
    required this.currentZoom,
    required this.minZoom,
    required this.maxZoom,
    required this.onChanged,
  });

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        const Text('ZOOM', style: TextStyle(color: Colors.white, fontSize: 10)),
        const SizedBox(height: 8),
        Expanded(
          child: RotatedBox(
            quarterTurns: -1,
            child: Slider(
              value: currentZoom.clamp(minZoom, maxZoom),
              min: minZoom,
              max: maxZoom,
              divisions: ((maxZoom - minZoom) * 4).toInt(),
              onChanged: onChanged,
              activeColor: Colors.amber,
            ),
          ),
        ),
        Text(
          '${currentZoom.toStringAsFixed(1)}x',
          style: const TextStyle(color: Colors.white, fontSize: 12),
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 3244: Camera Exposure Slider Widget

```dart
// lib/widgets/camera_exposure_slider.dart
import 'package:flutter/material.dart';

class CameraExposureSlider extends StatelessWidget {
  final double currentExposure;
  final double minExposure;
  final double maxExposure;
  final ValueChanged<double> onChanged;

  const CameraExposureSlider({
    super.key,
    required this.currentExposure,
    required this.minExposure,
    required this.maxExposure,
    required this.onChanged,
  });

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Icon(
          currentExposure > 0 ? Icons.brightness_high : Icons.brightness_low,
          color: Colors.yellow,
          size: 20,
        ),
        const SizedBox(height: 4),
        Expanded(
          child: RotatedBox(
            quarterTurns: -1,
            child: SliderTheme(
              data: const SliderThemeData(
                trackHeight: 2,
                thumbShape: RoundSliderThumbShape(enabledThumbRadius: 6),
              ),
              child: Slider(
                value: currentExposure.clamp(minExposure, maxExposure),
                min: minExposure,
                max: maxExposure,
                divisions: 20,
                onChanged: onChanged,
                activeColor: Colors.yellow,
                inactiveColor: Colors.white38,
              ),
            ),
          ),
        ),
        Text(
          currentExposure >= 0
              ? '+${currentExposure.toStringAsFixed(1)}'
              : currentExposure.toStringAsFixed(1),
          style: const TextStyle(color: Colors.yellow, fontSize: 12),
        ),
        const SizedBox(height: 4),
        Icon(
          currentExposure > 0 ? Icons.brightness_low : Icons.brightness_high,
          color: Colors.yellow,
          size: 20,
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 3245: QR/Barcode Scanner

```dart
// lib/screens/qr_scanner_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:mobile_scanner/mobile_scanner.dart';

class ScanResult {
  final String value;
  final BarcodeType type;
  final DateTime scannedAt;

  const ScanResult({
    required this.value,
    required this.type,
    required this.scannedAt,
  });
}

class QRScannerScreen extends StatefulWidget {
  const QRScannerScreen({super.key});

  @override
  State<QRScannerScreen> createState() => _QRScannerScreenState();
}

class _QRScannerScreenState extends State<QRScannerScreen> {
  final MobileScannerController _controller = MobileScannerController(
    facing: CameraFacing.back,
    torchEnabled: false,
  );

  final List<ScanResult> _scanHistory = [];
  ScanResult? _lastScan;
  bool _isScanning = true;
  bool _isTorchOn = false;

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  void _onDetect(BarcodeCapture capture) {
    if (!_isScanning) return;

    final List<Barcode> barcodes = capture.barcodes;
    for (final barcode in barcodes) {
      if (barcode.rawValue != null) {
        final result = ScanResult(
          value: barcode.rawValue!,
          type: barcode.type,
          scannedAt: DateTime.now(),
        );

        setState(() {
          _lastScan = result;
          _isScanning = false;

          // Avoid duplicates in history
          if (_scanHistory.isEmpty ||
              _scanHistory.first.value != result.value) {
            _scanHistory.insert(0, result);
          }
        });

        HapticFeedback.mediumImpact();
        _showResultDialog(result);
        break;
      }
    }
  }

  void _showResultDialog(ScanResult result) {
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      backgroundColor: Colors.transparent,
      builder: (context) => _ScanResultSheet(
        result: result,
        onRescan: () {
          Navigator.pop(context);
          setState(() => _isScanning = true);
        },
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      appBar: AppBar(
        title: const Text('QR / Barcode Scanner'),
        backgroundColor: Colors.black,
        foregroundColor: Colors.white,
        actions: [
          IconButton(
            icon: Icon(
              _isTorchOn ? Icons.flash_on : Icons.flash_off,
              color: _isTorchOn ? Colors.yellow : Colors.white,
            ),
            onPressed: () {
              _controller.toggleTorch();
              setState(() => _isTorchOn = !_isTorchOn);
            },
          ),
          IconButton(
            icon: const Icon(Icons.cameraswitch),
            onPressed: () => _controller.switchCamera(),
          ),
          if (_scanHistory.isNotEmpty)
            IconButton(
              icon: const Icon(Icons.history),
              onPressed: () => _showHistorySheet(),
            ),
        ],
      ),
      body: Stack(
        children: [
          // Camera preview
          MobileScanner(
            controller: _controller,
            onDetect: _onDetect,
          ),

          // Scanning overlay
          _ScannerOverlay(isScanning: _isScanning),

          // Status text
          Positioned(
            bottom: 40,
            left: 0,
            right: 0,
            child: Center(
              child: Container(
                padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
                decoration: BoxDecoration(
                  color: Colors.black.withOpacity(0.6),
                  borderRadius: BorderRadius.circular(8),
                ),
                child: Text(
                  _isScanning
                      ? 'Point camera at QR code or barcode'
                      : 'Tap screen or use Rescan button',
                  style: const TextStyle(color: Colors.white),
                ),
              ),
            ),
          ),

          // Last scan info
          if (_lastScan != null && !_isScanning)
            Positioned(
              top: 16,
              left: 16,
              right: 16,
              child: _LastScanBanner(
                result: _lastScan!,
                onRescan: () => setState(() => _isScanning = true),
              ),
            ),
        ],
      ),
    );
  }

  void _showHistorySheet() {
    showModalBottomSheet(
      context: context,
      backgroundColor: Colors.grey.shade900,
      builder: (context) => _ScanHistorySheet(history: _scanHistory),
    );
  }
}

class _ScannerOverlay extends StatelessWidget {
  final bool isScanning;

  const _ScannerOverlay({required this.isScanning});

  @override
  Widget build(BuildContext context) {
    return CustomPaint(
      size: Size.infinite,
      painter: _ScannerFramePainter(isScanning: isScanning),
    );
  }
}

class _ScannerFramePainter extends CustomPainter {
  final bool isScanning;

  _ScannerFramePainter({required this.isScanning});

  @override
  void paint(Canvas canvas, Size size) {
    final centerX = size.width / 2;
    final centerY = size.height / 2;
    const frameSize = 250.0;
    const cornerLength = 30.0;
    const cornerRadius = 4.0;

    final left = centerX - frameSize / 2;
    final top = centerY - frameSize / 2;
    final right = centerX + frameSize / 2;
    final bottom = centerY + frameSize / 2;

    // Darken outside the frame
    final darkPaint = Paint()..color = Colors.black.withOpacity(0.5);
    canvas.drawPath(
      Path.combine(
        PathOperation.difference,
        Path()..addRect(Rect.fromLTWH(0, 0, size.width, size.height)),
        Path()..addRRect(RRect.fromLTRBR(
          left, top, right, bottom,
          const Radius.circular(8),
        )),
      ),
      darkPaint,
    );

    // Draw corner brackets
    final cornerPaint = Paint()
      ..color = isScanning ? Colors.green : Colors.amber
      ..strokeWidth = 3
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round;

    // Top-left corner
    canvas.drawPath(
      Path()
        ..moveTo(left, top + cornerLength)
        ..lineTo(left, top + cornerRadius)
        ..arcToPoint(Offset(left + cornerRadius, top), radius: const Radius.circular(cornerRadius))
        ..lineTo(left + cornerLength, top),
      cornerPaint,
    );

    // Top-right corner
    canvas.drawPath(
      Path()
        ..moveTo(right - cornerLength, top)
        ..lineTo(right - cornerRadius, top)
        ..arcToPoint(Offset(right, top + cornerRadius), radius: const Radius.circular(cornerRadius))
        ..lineTo(right, top + cornerLength),
      cornerPaint,
    );

    // Bottom-left corner
    canvas.drawPath(
      Path()
        ..moveTo(left, bottom - cornerLength)
        ..lineTo(left, bottom - cornerRadius)
        ..arcToPoint(Offset(left + cornerRadius, bottom), radius: const Radius.circular(cornerRadius), clockwise: false)
        ..lineTo(left + cornerLength, bottom),
      cornerPaint,
    );

    // Bottom-right corner
    canvas.drawPath(
      Path()
        ..moveTo(right - cornerLength, bottom)
        ..lineTo(right - cornerRadius, bottom)
        ..arcToPoint(Offset(right, bottom - cornerRadius), radius: const Radius.circular(cornerRadius), clockwise: false)
        ..lineTo(right, bottom - cornerLength),
      cornerPaint,
    );
  }

  @override
  bool shouldRepaint(covariant _ScannerFramePainter old) =>
      old.isScanning != isScanning;
}

class _ScanResultSheet extends StatelessWidget {
  final ScanResult result;
  final VoidCallback onRescan;

  const _ScanResultSheet({required this.result, required this.onRescan});

  IconData get _typeIcon {
    switch (result.type) {
      case BarcodeType.url: return Icons.link;
      case BarcodeType.email: return Icons.email;
      case BarcodeType.phone: return Icons.phone;
      case BarcodeType.sms: return Icons.sms;
      case BarcodeType.wifi: return Icons.wifi;
      case BarcodeType.contactInfo: return Icons.contact_page;
      default: return Icons.qr_code;
    }
  }

  String get _typeLabel {
    switch (result.type) {
      case BarcodeType.url: return 'URL';
      case BarcodeType.email: return 'Email';
      case BarcodeType.phone: return 'Phone';
      case BarcodeType.sms: return 'SMS';
      case BarcodeType.wifi: return 'WiFi';
      case BarcodeType.contactInfo: return 'Contact';
      default: return 'Text';
    }
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(24),
      decoration: const BoxDecoration(
        color: Colors.white,
        borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
      ),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(
            children: [
              Icon(_typeIcon, color: Colors.blue),
              const SizedBox(width: 8),
              Text(
                _typeLabel,
                style: const TextStyle(
                  fontWeight: FontWeight.bold,
                  fontSize: 18,
                ),
              ),
              const Spacer(),
              IconButton(
                icon: const Icon(Icons.copy),
                onPressed: () {
                  Clipboard.setData(ClipboardData(text: result.value));
                  ScaffoldMessenger.of(context).showSnackBar(
                    const SnackBar(content: Text('Copied to clipboard')),
                  );
                },
              ),
            ],
          ),
          const Divider(),
          const SizedBox(height: 8),
          SelectableText(
            result.value,
            style: const TextStyle(fontSize: 16),
          ),
          const SizedBox(height: 24),
          SizedBox(
            width: double.infinity,
            child: ElevatedButton(
              onPressed: onRescan,
              style: ElevatedButton.styleFrom(
                padding: const EdgeInsets.all(12),
              ),
              child: const Text('Scan Again'),
            ),
          ),
        ],
      ),
    );
  }
}

class _LastScanBanner extends StatelessWidget {
  final ScanResult result;
  final VoidCallback onRescan;

  const _LastScanBanner({required this.result, required this.onRescan});

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(12),
      decoration: BoxDecoration(
        color: Colors.green.withOpacity(0.9),
        borderRadius: BorderRadius.circular(8),
      ),
      child: Row(
        children: [
          const Icon(Icons.check_circle, color: Colors.white),
          const SizedBox(width: 8),
          Expanded(
            child: Text(
              result.value,
              style: const TextStyle(color: Colors.white),
              maxLines: 1,
              overflow: TextOverflow.ellipsis,
            ),
          ),
          TextButton(
            onPressed: onRescan,
            child: const Text('RESCAN', style: TextStyle(color: Colors.white)),
          ),
        ],
      ),
    );
  }
}

class _ScanHistorySheet extends StatelessWidget {
  final List<ScanResult> history;

  const _ScanHistorySheet({required this.history});

  @override
  Widget build(BuildContext context) {
    return DraggableScrollableSheet(
      initialChildSize: 0.7,
      maxChildSize: 0.9,
      minChildSize: 0.4,
      expand: false,
      builder: (context, scrollController) {
        return Column(
          children: [
            Padding(
              padding: const EdgeInsets.all(16),
              child: Text(
                'Scan History (${history.length})',
                style: const TextStyle(
                  color: Colors.white,
                  fontSize: 18,
                  fontWeight: FontWeight.bold,
                ),
              ),
            ),
            Expanded(
              child: ListView.builder(
                controller: scrollController,
                itemCount: history.length,
                itemBuilder: (context, index) {
                  final item = history[index];
                  return ListTile(
                    leading: const Icon(Icons.qr_code, color: Colors.white),
                    title: Text(
                      item.value,
                      style: const TextStyle(color: Colors.white),
                      maxLines: 1,
                      overflow: TextOverflow.ellipsis,
                    ),
                    subtitle: Text(
                      item.scannedAt.toString().split('.').first,
                      style: const TextStyle(color: Colors.grey),
                    ),
                    trailing: IconButton(
                      icon: const Icon(Icons.copy, color: Colors.white70),
                      onPressed: () {
                        Clipboard.setData(ClipboardData(text: item.value));
                        ScaffoldMessenger.of(context).showSnackBar(
                          const SnackBar(content: Text('Copied!')),
                        );
                      },
                    ),
                  );
                },
              ),
            ),
          ],
        );
      },
    );
  }
}
```

---

## ขั้นตอนที่ 3246: Video Recorder Screen

```dart
// lib/screens/video_recorder_screen.dart
import 'dart:io';
import 'dart:async';
import 'package:camera/camera.dart';
import 'package:flutter/material.dart';
import 'package:permission_handler/permission_handler.dart';
import 'package:video_player/video_player.dart';
import 'package:path_provider/path_provider.dart';

class VideoRecorderScreen extends StatefulWidget {
  const VideoRecorderScreen({super.key});

  @override
  State<VideoRecorderScreen> createState() => _VideoRecorderScreenState();
}

class _VideoRecorderScreenState extends State<VideoRecorderScreen> {
  CameraController? _cameraController;
  VideoPlayerController? _videoController;
  List<CameraDescription> _cameras = [];

  bool _isInitialized = false;
  bool _isRecording = false;
  bool _isPaused = false;
  bool _isPlayingPreview = false;

  String? _recordedVideoPath;
  Duration _recordingDuration = Duration.zero;
  Timer? _durationTimer;

  int _cameraIndex = 0;
  FlashMode _flashMode = FlashMode.off;
  double _currentZoom = 1.0;

  @override
  void initState() {
    super.initState();
    _initCamera();
  }

  Future<void> _initCamera() async {
    final cameraStatus = await Permission.camera.request();
    final micStatus = await Permission.microphone.request();

    if (cameraStatus.isDenied || micStatus.isDenied) return;

    _cameras = await availableCameras();
    if (_cameras.isEmpty) return;

    await _setupCamera(_cameras[_cameraIndex]);
  }

  Future<void> _setupCamera(CameraDescription camera) async {
    _cameraController?.dispose();
    _videoController?.dispose();
    _videoController = null;
    _recordedVideoPath = null;

    final controller = CameraController(
      camera,
      ResolutionPreset.high,
      enableAudio: true,
    );

    _cameraController = controller;

    await controller.initialize();

    if (mounted) {
      setState(() {
        _isInitialized = true;
        _currentZoom = 1.0;
      });
    }
  }

  Future<void> _startRecording() async {
    if (_cameraController == null || !_cameraController!.value.isInitialized) {
      return;
    }

    try {
      await _cameraController!.startVideoRecording();

      setState(() {
        _isRecording = true;
        _isPaused = false;
        _recordingDuration = Duration.zero;
      });

      _durationTimer = Timer.periodic(const Duration(seconds: 1), (_) {
        if (!_isPaused) {
          setState(() {
            _recordingDuration += const Duration(seconds: 1);
          });
        }
      });
    } on CameraException catch (e) {
      _showError('Start recording failed: ${e.description}');
    }
  }

  Future<void> _stopRecording() async {
    if (!_isRecording) return;

    _durationTimer?.cancel();

    try {
      final XFile file = await _cameraController!.stopVideoRecording();

      final directory = await getTemporaryDirectory();
      final timestamp = DateTime.now().millisecondsSinceEpoch;
      final path = '${directory.path}/video_$timestamp.mp4';

      await File(file.path).copy(path);

      setState(() {
        _isRecording = false;
        _isPaused = false;
        _recordedVideoPath = path;
        _recordingDuration = Duration.zero;
      });

      await _setupVideoPreview(path);
    } on CameraException catch (e) {
      _showError('Stop recording failed: ${e.description}');
    }
  }

  Future<void> _pauseRecording() async {
    if (!_isRecording || _isPaused) return;

    await _cameraController!.pauseVideoRecording();
    setState(() => _isPaused = true);
  }

  Future<void> _resumeRecording() async {
    if (!_isRecording || !_isPaused) return;

    await _cameraController!.resumeVideoRecording();
    setState(() => _isPaused = false);
  }

  Future<void> _setupVideoPreview(String path) async {
    _videoController = VideoPlayerController.file(File(path));
    await _videoController!.initialize();
    await _videoController!.setLooping(true);

    setState(() {});
  }

  Future<void> _togglePlayback() async {
    if (_videoController == null) return;

    if (_videoController!.value.isPlaying) {
      await _videoController!.pause();
      setState(() => _isPlayingPreview = false);
    } else {
      await _videoController!.play();
      setState(() => _isPlayingPreview = true);
    }
  }

  void _discardRecording() {
    _videoController?.dispose();
    _videoController = null;

    setState(() {
      _recordedVideoPath = null;
      _isPlayingPreview = false;
    });
  }

  void _showError(String message) {
    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(message), backgroundColor: Colors.red),
      );
    }
  }

  String _formatDuration(Duration d) {
    final minutes = d.inMinutes.remainder(60).toString().padLeft(2, '0');
    final seconds = d.inSeconds.remainder(60).toString().padLeft(2, '0');
    return '$minutes:$seconds';
  }

  @override
  void dispose() {
    _durationTimer?.cancel();
    _cameraController?.dispose();
    _videoController?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      appBar: AppBar(
        title: const Text('Video Recorder'),
        backgroundColor: Colors.black,
        foregroundColor: Colors.white,
        actions: [
          if (_cameraController != null && _isInitialized)
            IconButton(
              icon: Icon(
                _flashMode == FlashMode.off ? Icons.flash_off : Icons.flash_on,
                color: _flashMode == FlashMode.torch ? Colors.yellow : Colors.white,
              ),
              onPressed: () async {
                final next = _flashMode == FlashMode.off
                    ? FlashMode.torch
                    : FlashMode.off;
                await _cameraController!.setFlashMode(next);
                setState(() => _flashMode = next);
              },
            ),
          if (_cameras.length > 1)
            IconButton(
              icon: const Icon(Icons.cameraswitch),
              onPressed: () async {
                setState(() {
                  _cameraIndex = (_cameraIndex + 1) % _cameras.length;
                  _isInitialized = false;
                });
                await _setupCamera(_cameras[_cameraIndex]);
              },
            ),
        ],
      ),
      body: !_isInitialized
          ? const Center(child: CircularProgressIndicator(color: Colors.white))
          : Column(
              children: [
                // Camera / Preview
                Expanded(
                  child: _recordedVideoPath != null && _videoController != null
                      ? Stack(
                          alignment: Alignment.center,
                          children: [
                            AspectRatio(
                              aspectRatio: _videoController!.value.aspectRatio,
                              child: VideoPlayer(_videoController!),
                            ),
                            GestureDetector(
                              onTap: _togglePlayback,
                              child: AnimatedOpacity(
                                opacity: _isPlayingPreview ? 0.0 : 1.0,
                                duration: const Duration(milliseconds: 300),
                                child: Container(
                                  width: 64,
                                  height: 64,
                                  decoration: BoxDecoration(
                                    color: Colors.black54,
                                    shape: BoxShape.circle,
                                  ),
                                  child: const Icon(
                                    Icons.play_arrow,
                                    color: Colors.white,
                                    size: 40,
                                  ),
                                ),
                              ),
                            ),
                          ],
                        )
                      : Stack(
                          children: [
                            SizedBox.expand(
                              child: CameraPreview(_cameraController!),
                            ),
                            if (_isRecording)
                              Positioned(
                                top: 16,
                                left: 16,
                                right: 16,
                                child: _RecordingIndicator(
                                  duration: _recordingDuration,
                                  isPaused: _isPaused,
                                ),
                              ),
                          ],
                        ),
                ),

                // Controls
                Container(
                  padding: const EdgeInsets.symmetric(
                    horizontal: 24,
                    vertical: 20,
                  ),
                  color: Colors.black,
                  child: _recordedVideoPath != null
                      ? _VideoPreviewControls(
                          isPlaying: _isPlayingPreview,
                          onPlay: _togglePlayback,
                          onDiscard: _discardRecording,
                          onSave: () {
                            ScaffoldMessenger.of(context).showSnackBar(
                              const SnackBar(content: Text('Video saved!')),
                            );
                          },
                        )
                      : _RecordingControls(
                          isRecording: _isRecording,
                          isPaused: _isPaused,
                          onStart: _startRecording,
                          onStop: _stopRecording,
                          onPause: _pauseRecording,
                          onResume: _resumeRecording,
                        ),
                ),
              ],
            ),
    );
  }
}

class _RecordingIndicator extends StatefulWidget {
  final Duration duration;
  final bool isPaused;

  const _RecordingIndicator({required this.duration, required this.isPaused});

  @override
  State<_RecordingIndicator> createState() => _RecordingIndicatorState();
}

class _RecordingIndicatorState extends State<_RecordingIndicator>
    with SingleTickerProviderStateMixin {
  late final AnimationController _blinkController;

  @override
  void initState() {
    super.initState();
    _blinkController = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 800),
    )..repeat(reverse: true);
  }

  @override
  void dispose() {
    _blinkController.dispose();
    super.dispose();
  }

  String _formatDuration(Duration d) {
    final minutes = d.inMinutes.remainder(60).toString().padLeft(2, '0');
    final seconds = d.inSeconds.remainder(60).toString().padLeft(2, '0');
    return '$minutes:$seconds';
  }

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Container(
        padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
        decoration: BoxDecoration(
          color: Colors.black.withOpacity(0.6),
          borderRadius: BorderRadius.circular(20),
        ),
        child: Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            AnimatedBuilder(
              animation: _blinkController,
              builder: (context, child) => Opacity(
                opacity: widget.isPaused ? 0.5 : _blinkController.value,
                child: const Icon(Icons.circle, color: Colors.red, size: 12),
              ),
            ),
            const SizedBox(width: 6),
            Text(
              widget.isPaused
                  ? 'PAUSED'
                  : 'REC ${_formatDuration(widget.duration)}',
              style: const TextStyle(
                color: Colors.white,
                fontWeight: FontWeight.bold,
                fontSize: 14,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _RecordingControls extends StatelessWidget {
  final bool isRecording;
  final bool isPaused;
  final VoidCallback onStart;
  final VoidCallback onStop;
  final VoidCallback onPause;
  final VoidCallback onResume;

  const _RecordingControls({
    required this.isRecording,
    required this.isPaused,
    required this.onStart,
    required this.onStop,
    required this.onPause,
    required this.onResume,
  });

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
      children: [
        // Pause/Resume button
        if (isRecording)
          IconButton(
            icon: Icon(
              isPaused ? Icons.play_arrow : Icons.pause,
              color: Colors.white,
              size: 32,
            ),
            onPressed: isPaused ? onResume : onPause,
          )
        else
          const SizedBox(width: 48),

        // Record/Stop button
        GestureDetector(
          onTap: isRecording ? onStop : onStart,
          child: AnimatedContainer(
            duration: const Duration(milliseconds: 200),
            width: 72,
            height: 72,
            decoration: BoxDecoration(
              shape: BoxShape.circle,
              border: Border.all(color: Colors.white, width: 3),
              color: isRecording ? Colors.transparent : Colors.transparent,
            ),
            child: Center(
              child: AnimatedContainer(
                duration: const Duration(milliseconds: 200),
                width: isRecording ? 28 : 56,
                height: isRecording ? 28 : 56,
                decoration: BoxDecoration(
                  color: Colors.red,
                  borderRadius: BorderRadius.circular(isRecording ? 4 : 28),
                ),
              ),
            ),
          ),
        ),

        const SizedBox(width: 48),
      ],
    );
  }
}

class _VideoPreviewControls extends StatelessWidget {
  final bool isPlaying;
  final VoidCallback onPlay;
  final VoidCallback onDiscard;
  final VoidCallback onSave;

  const _VideoPreviewControls({
    required this.isPlaying,
    required this.onPlay,
    required this.onDiscard,
    required this.onSave,
  });

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
      children: [
        ElevatedButton.icon(
          onPressed: onDiscard,
          icon: const Icon(Icons.delete),
          label: const Text('Discard'),
          style: ElevatedButton.styleFrom(
            backgroundColor: Colors.red.shade900,
          ),
        ),
        IconButton(
          icon: Icon(
            isPlaying ? Icons.pause_circle : Icons.play_circle,
            color: Colors.white,
            size: 48,
          ),
          onPressed: onPlay,
        ),
        ElevatedButton.icon(
          onPressed: onSave,
          icon: const Icon(Icons.save),
          label: const Text('Save'),
          style: ElevatedButton.styleFrom(
            backgroundColor: Colors.green.shade900,
          ),
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 3247: Document Scanner Screen

```dart
// lib/screens/document_scanner_screen.dart
import 'dart:io';
import 'dart:math';
import 'package:camera/camera.dart';
import 'package:flutter/material.dart';
import 'package:path_provider/path_provider.dart';
import 'package:permission_handler/permission_handler.dart';

class DocumentScannerScreen extends StatefulWidget {
  const DocumentScannerScreen({super.key});

  @override
  State<DocumentScannerScreen> createState() => _DocumentScannerScreenState();
}

class _DocumentScannerScreenState extends State<DocumentScannerScreen> {
  CameraController? _cameraController;
  bool _isInitialized = false;
  bool _isCapturing = false;
  String? _capturedImagePath;

  // Simulated document detection rectangle
  final ValueNotifier<Rect?> _detectedRect = ValueNotifier(null);

  @override
  void initState() {
    super.initState();
    _initCamera();
    _simulateDocumentDetection();
  }

  Future<void> _initCamera() async {
    await Permission.camera.request();
    final cameras = await availableCameras();
    if (cameras.isEmpty) return;

    _cameraController = CameraController(
      cameras.first,
      ResolutionPreset.high,
      enableAudio: false,
    );

    await _cameraController!.initialize();
    if (mounted) setState(() => _isInitialized = true);
  }

  void _simulateDocumentDetection() {
    // Simulate document edge detection with a pulsing rectangle
    Future.delayed(const Duration(seconds: 1), () {
      if (mounted) {
        _detectedRect.value = const Rect.fromLTWH(40, 100, 300, 420);

        // Update every second to simulate tracking
        Timer.periodic(const Duration(milliseconds: 500), (timer) {
          if (!mounted) {
            timer.cancel();
            return;
          }
          final random = Random();
          final jitter = random.nextDouble() * 4 - 2;
          _detectedRect.value = Rect.fromLTWH(
            40 + jitter,
            100 + jitter,
            300 - jitter,
            420 - jitter,
          );
        });
      }
    });
  }

  Future<void> _captureDocument() async {
    if (_cameraController == null || _isCapturing) return;

    setState(() => _isCapturing = true);

    try {
      final XFile file = await _cameraController!.takePicture();
      final directory = await getTemporaryDirectory();
      final path = '${directory.path}/doc_${DateTime.now().millisecondsSinceEpoch}.jpg';
      await File(file.path).copy(path);

      setState(() => _capturedImagePath = path);
    } finally {
      if (mounted) setState(() => _isCapturing = false);
    }
  }

  @override
  void dispose() {
    _detectedRect.dispose();
    _cameraController?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      appBar: AppBar(
        title: const Text('Document Scanner'),
        backgroundColor: Colors.black,
        foregroundColor: Colors.white,
        actions: [
          if (_capturedImagePath != null)
            IconButton(
              icon: const Icon(Icons.refresh),
              onPressed: () => setState(() => _capturedImagePath = null),
            ),
        ],
      ),
      body: !_isInitialized
          ? const Center(child: CircularProgressIndicator(color: Colors.white))
          : _capturedImagePath != null
              ? _DocumentPreview(
                  imagePath: _capturedImagePath!,
                  onRetake: () => setState(() => _capturedImagePath = null),
                  onSave: () {
                    ScaffoldMessenger.of(context).showSnackBar(
                      const SnackBar(content: Text('Document saved!')),
                    );
                  },
                )
              : Stack(
                  children: [
                    // Camera preview
                    CameraPreview(_cameraController!),

                    // Document detection overlay
                    ValueListenableBuilder<Rect?>(
                      valueListenable: _detectedRect,
                      builder: (context, rect, _) {
                        return CustomPaint(
                          size: Size.infinite,
                          painter: _DocumentDetectionOverlay(
                            detectedRect: rect,
                            isGoodCapture: rect != null,
                          ),
                        );
                      },
                    ),

                    // Instructions
                    Positioned(
                      top: 16,
                      left: 16,
                      right: 16,
                      child: ValueListenableBuilder<Rect?>(
                        valueListenable: _detectedRect,
                        builder: (context, rect, _) {
                          return Container(
                            padding: const EdgeInsets.all(8),
                            decoration: BoxDecoration(
                              color: Colors.black54,
                              borderRadius: BorderRadius.circular(8),
                            ),
                            child: Text(
                              rect != null
                                  ? 'Document detected! Tap to capture.'
                                  : 'Position document in frame...',
                              style: TextStyle(
                                color: rect != null ? Colors.green : Colors.white,
                              ),
                              textAlign: TextAlign.center,
                            ),
                          );
                        },
                      ),
                    ),

                    // Capture button
                    Positioned(
                      bottom: 40,
                      left: 0,
                      right: 0,
                      child: Center(
                        child: GestureDetector(
                          onTap: _captureDocument,
                          child: Container(
                            width: 72,
                            height: 72,
                            decoration: BoxDecoration(
                              shape: BoxShape.circle,
                              color: Colors.white,
                              boxShadow: [
                                BoxShadow(
                                  color: Colors.white.withOpacity(0.3),
                                  blurRadius: 16,
                                  spreadRadius: 4,
                                ),
                              ],
                            ),
                            child: const Icon(
                              Icons.document_scanner,
                              color: Colors.black,
                              size: 36,
                            ),
                          ),
                        ),
                      ),
                    ),
                  ],
                ),
    );
  }
}

class _DocumentDetectionOverlay extends CustomPainter {
  final Rect? detectedRect;
  final bool isGoodCapture;

  _DocumentDetectionOverlay({
    required this.detectedRect,
    required this.isGoodCapture,
  });

  @override
  void paint(Canvas canvas, Size size) {
    if (detectedRect == null) return;

    final color = isGoodCapture ? Colors.green : Colors.orange;
    final paint = Paint()
      ..color = color.withOpacity(0.3)
      ..style = PaintingStyle.fill;

    final borderPaint = Paint()
      ..color = color
      ..style = PaintingStyle.stroke
      ..strokeWidth = 3;

    // Draw semi-transparent fill
    canvas.drawRect(detectedRect!, paint);

    // Draw border
    canvas.drawRect(detectedRect!, borderPaint);

    // Draw corner markers
    const cornerSize = 20.0;
    final corners = [
      Offset(detectedRect!.left, detectedRect!.top),
      Offset(detectedRect!.right, detectedRect!.top),
      Offset(detectedRect!.right, detectedRect!.bottom),
      Offset(detectedRect!.left, detectedRect!.bottom),
    ];

    final cornerPaint = Paint()
      ..color = color
      ..style = PaintingStyle.stroke
      ..strokeWidth = 4
      ..strokeCap = StrokeCap.round;

    for (final corner in corners) {
      final isLeft = corner.dx == detectedRect!.left;
      final isTop = corner.dy == detectedRect!.top;

      final dx = isLeft ? cornerSize : -cornerSize;
      final dy = isTop ? cornerSize : -cornerSize;

      canvas.drawLine(corner, corner.translate(dx, 0), cornerPaint);
      canvas.drawLine(corner, corner.translate(0, dy), cornerPaint);
    }
  }

  @override
  bool shouldRepaint(covariant _DocumentDetectionOverlay old) =>
      old.detectedRect != detectedRect || old.isGoodCapture != isGoodCapture;
}

class _DocumentPreview extends StatelessWidget {
  final String imagePath;
  final VoidCallback onRetake;
  final VoidCallback onSave;

  const _DocumentPreview({
    required this.imagePath,
    required this.onRetake,
    required this.onSave,
  });

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Expanded(
          child: Container(
            margin: const EdgeInsets.all(16),
            decoration: BoxDecoration(
              borderRadius: BorderRadius.circular(8),
              boxShadow: [
                BoxShadow(
                  color: Colors.white.withOpacity(0.1),
                  blurRadius: 8,
                ),
              ],
            ),
            clipBehavior: Clip.antiAlias,
            child: Image.file(
              File(imagePath),
              fit: BoxFit.contain,
            ),
          ),
        ),
        Padding(
          padding: const EdgeInsets.all(24),
          child: Row(
            children: [
              Expanded(
                child: OutlinedButton.icon(
                  onPressed: onRetake,
                  icon: const Icon(Icons.camera_alt),
                  label: const Text('Retake'),
                  style: OutlinedButton.styleFrom(
                    foregroundColor: Colors.white,
                    side: const BorderSide(color: Colors.white),
                    padding: const EdgeInsets.all(12),
                  ),
                ),
              ),
              const SizedBox(width: 16),
              Expanded(
                child: ElevatedButton.icon(
                  onPressed: onSave,
                  icon: const Icon(Icons.save),
                  label: const Text('Save'),
                  style: ElevatedButton.styleFrom(
                    padding: const EdgeInsets.all(12),
                  ),
                ),
              ),
            ],
          ),
        ),
      ],
    );
  }
}

// Add Timer import at top of document_scanner_screen.dart
import 'dart:async';
```

---

**← [Part 83](part-83-flutter-ar-vr.md)**
**ต่อไป: [Part 85 →](part-85-dart-concurrency-advanced.md)**
