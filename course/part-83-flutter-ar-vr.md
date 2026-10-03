# Part 83: Flutter AR/VR
## ขั้นตอนที่ 3201-3240

## 🎯 เป้าหมายของ Part นี้
- ใช้งาน ARCore/ARKit ด้วย ar_flutter_plugin
- วาง 3D objects ใน Augmented Reality
- ใช้ AR Model Viewer
- จัดการ AR tracking และ anchors
- สร้าง Simple VR viewer สำหรับ 360-degree images

---

## ขั้นตอนที่ 3201: ติดตั้งและตั้งค่า AR Package

```yaml
# pubspec.yaml
name: flutter_ar_demo
description: Flutter AR/VR Demo with ARCore and ARKit

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: ">=3.10.0"

dependencies:
  flutter:
    sdk: flutter
  ar_flutter_plugin: ^0.7.3
  vector_math: ^2.1.4
  http: ^1.1.0
  path_provider: ^2.1.1
  permission_handler: ^11.1.0
  sensors_plus: ^4.0.2
  flutter_gl: ^0.0.9

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0

flutter:
  assets:
    - assets/models/
    - assets/images/
    - assets/textures/
```

```xml
<!-- android/app/src/main/AndroidManifest.xml - add these -->
<uses-permission android:name="android.permission.CAMERA"/>
<uses-feature android:name="android.hardware.camera.ar" android:required="true"/>

<application>
  <!-- ... -->
  <meta-data
    android:name="com.google.ar.core"
    android:value="required"/>
</application>
```

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'screens/ar_home_screen.dart';

void main() {
  runApp(const ARDemoApp());
}

class ARDemoApp extends StatelessWidget {
  const ARDemoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter AR Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepOrange),
        useMaterial3: true,
      ),
      home: const ARHomeScreen(),
    );
  }
}
```

---

## ขั้นตอนที่ 3202: AR Home Screen

```dart
// lib/screens/ar_home_screen.dart
import 'package:flutter/material.dart';
import 'ar_object_placement_screen.dart';
import 'ar_model_viewer_screen.dart';
import 'ar_measurement_screen.dart';
import 'vr_360_viewer_screen.dart';

class ARHomeScreen extends StatelessWidget {
  const ARHomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Flutter AR/VR Demo'),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
      ),
      body: GridView.count(
        crossAxisCount: 2,
        padding: const EdgeInsets.all(16),
        mainAxisSpacing: 16,
        crossAxisSpacing: 16,
        children: [
          _ARFeatureCard(
            title: 'Object Placement',
            description: 'Place 3D objects in your space',
            icon: Icons.view_in_ar,
            color: Colors.blue,
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const ARObjectPlacementScreen()),
            ),
          ),
          _ARFeatureCard(
            title: 'Model Viewer',
            description: 'View 3D models in AR',
            icon: Icons.threed_rotation,
            color: Colors.green,
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const ARModelViewerScreen()),
            ),
          ),
          _ARFeatureCard(
            title: 'Measurement',
            description: 'Measure distances with AR',
            icon: Icons.straighten,
            color: Colors.orange,
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const ARMeasurementScreen()),
            ),
          ),
          _ARFeatureCard(
            title: '360° VR Viewer',
            description: 'View 360 panoramic images',
            icon: Icons.panorama,
            color: Colors.purple,
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const VR360ViewerScreen()),
            ),
          ),
        ],
      ),
    );
  }
}

class _ARFeatureCard extends StatelessWidget {
  final String title;
  final String description;
  final IconData icon;
  final Color color;
  final VoidCallback onTap;

  const _ARFeatureCard({
    required this.title,
    required this.description,
    required this.icon,
    required this.color,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      elevation: 4,
      clipBehavior: Clip.antiAlias,
      child: InkWell(
        onTap: onTap,
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Container(
              width: 64,
              height: 64,
              decoration: BoxDecoration(
                color: color.withOpacity(0.15),
                shape: BoxShape.circle,
              ),
              child: Icon(icon, size: 36, color: color),
            ),
            const SizedBox(height: 12),
            Text(
              title,
              style: const TextStyle(
                fontWeight: FontWeight.bold,
                fontSize: 14,
              ),
              textAlign: TextAlign.center,
            ),
            const SizedBox(height: 4),
            Padding(
              padding: const EdgeInsets.symmetric(horizontal: 8),
              child: Text(
                description,
                style: TextStyle(
                  fontSize: 11,
                  color: Colors.grey.shade600,
                ),
                textAlign: TextAlign.center,
                maxLines: 2,
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

## ขั้นตอนที่ 3203: AR Object Placement Screen

```dart
// lib/screens/ar_object_placement_screen.dart
import 'dart:math';
import 'package:flutter/material.dart';
import 'package:ar_flutter_plugin/ar_flutter_plugin.dart';
import 'package:ar_flutter_plugin/datatypes/config_planedetection.dart';
import 'package:ar_flutter_plugin/datatypes/hittest_result_types.dart';
import 'package:ar_flutter_plugin/datatypes/node_types.dart';
import 'package:ar_flutter_plugin/managers/ar_anchor_manager.dart';
import 'package:ar_flutter_plugin/managers/ar_location_manager.dart';
import 'package:ar_flutter_plugin/managers/ar_object_manager.dart';
import 'package:ar_flutter_plugin/managers/ar_session_manager.dart';
import 'package:ar_flutter_plugin/models/ar_anchor.dart';
import 'package:ar_flutter_plugin/models/ar_hittest_result.dart';
import 'package:ar_flutter_plugin/models/ar_node.dart';
import 'package:vector_math/vector_math_64.dart' as vector_math;

/// Available 3D shapes to place in AR
enum ARShape { cube, sphere, cylinder, cone }

class ARObjectPlacementScreen extends StatefulWidget {
  const ARObjectPlacementScreen({super.key});

  @override
  State<ARObjectPlacementScreen> createState() =>
      _ARObjectPlacementScreenState();
}

class _ARObjectPlacementScreenState extends State<ARObjectPlacementScreen> {
  ARSessionManager? _arSessionManager;
  ARObjectManager? _arObjectManager;
  ARAnchorManager? _arAnchorManager;

  final List<ARNode> _nodes = [];
  final List<ARAnchor> _anchors = [];

  ARShape _selectedShape = ARShape.cube;
  Color _selectedColor = Colors.blue;
  double _selectedScale = 0.2;
  bool _isTracking = false;
  bool _planeDetected = false;
  int _objectCount = 0;

  final List<Color> _colors = [
    Colors.blue,
    Colors.red,
    Colors.green,
    Colors.yellow,
    Colors.purple,
    Colors.orange,
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('AR Object Placement'),
        backgroundColor: Colors.black,
        foregroundColor: Colors.white,
        actions: [
          IconButton(
            icon: const Icon(Icons.delete_sweep),
            onPressed: _clearAllObjects,
            tooltip: 'Clear All',
          ),
        ],
      ),
      body: Stack(
        children: [
          // AR View
          ARView(
            onARViewCreated: _onARViewCreated,
            planeDetectionConfig:
                PlaneDetectionConfig.horizontalAndVertical,
          ),

          // Status overlay
          _ARStatusOverlay(
            isTracking: _isTracking,
            planeDetected: _planeDetected,
            objectCount: _objectCount,
          ),

          // Controls panel at bottom
          Positioned(
            bottom: 0,
            left: 0,
            right: 0,
            child: _ARControlsPanel(
              selectedShape: _selectedShape,
              selectedColor: _selectedColor,
              selectedScale: _selectedScale,
              availableColors: _colors,
              onShapeChanged: (shape) => setState(() => _selectedShape = shape),
              onColorChanged: (color) => setState(() => _selectedColor = color),
              onScaleChanged: (scale) => setState(() => _selectedScale = scale),
            ),
          ),

          // Crosshair
          const Center(
            child: Icon(Icons.add, color: Colors.white, size: 32),
          ),
        ],
      ),
    );
  }

  void _onARViewCreated(
    ARSessionManager sessionManager,
    ARObjectManager objectManager,
    ARAnchorManager anchorManager,
    ARLocationManager locationManager,
  ) {
    _arSessionManager = sessionManager;
    _arObjectManager = objectManager;
    _arAnchorManager = anchorManager;

    _arSessionManager!.onInitialize(
      showAnimatedGuide: true,
      showFeaturePoints: false,
      showPlanes: true,
      customPlaneTexturePath: null,
      showWorldOrigin: false,
      handleRotation: true,
      handlePans: true,
      handleTaps: true,
    );

    _arObjectManager!.onInitialize();

    _arSessionManager!.onTrackingChanged = (isTracking) {
      if (mounted) setState(() => _isTracking = isTracking);
    };

    _arObjectManager!.onNodeTap = _handleNodeTap;

    // Listen for plane detection
    _arSessionManager!.onPlaneDetected = () {
      if (mounted && !_planeDetected) {
        setState(() => _planeDetected = true);
      }
    };
  }

  void _handleNodeTap(List<String> nodes) {
    if (nodes.isEmpty) return;

    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text('Object Action'),
        content: Text('Object ID: ${nodes.first}'),
        actions: [
          TextButton(
            onPressed: () {
              Navigator.pop(context);
              _deleteNode(nodes.first);
            },
            child: const Text('Delete', style: TextStyle(color: Colors.red)),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('Cancel'),
          ),
        ],
      ),
    );
  }

  void _deleteNode(String nodeId) {
    final node = _nodes.firstWhere(
      (n) => n.name == nodeId,
      orElse: () => _nodes.first,
    );
    _arObjectManager?.removeNode(node);
    _nodes.remove(node);
    if (mounted) setState(() => _objectCount = _nodes.length);
  }

  Future<void> _placeObjectAtCenter() async {
    if (_arObjectManager == null || !_planeDetected) return;

    // Create AR node based on selected shape
    final nodeUri = _getShapeUri(_selectedShape);
    final colorHex = '#${_selectedColor.value.toRadixString(16).substring(2)}';

    final newNode = ARNode(
      type: NodeType.webGLB,
      uri: nodeUri,
      scale: vector_math.Vector3(
        _selectedScale,
        _selectedScale,
        _selectedScale,
      ),
      position: vector_math.Vector3(0, 0, -1.0),
      rotation: vector_math.Vector4(0, 0, 0, 1),
      name: 'object_${DateTime.now().millisecondsSinceEpoch}',
    );

    final didAddNode = await _arObjectManager!.addNode(newNode);
    if (didAddNode == true) {
      _nodes.add(newNode);
      if (mounted) setState(() => _objectCount = _nodes.length);
    }
  }

  String _getShapeUri(ARShape shape) {
    switch (shape) {
      case ARShape.cube:
        return 'https://github.com/google/model-viewer/raw/master/packages/shared-assets/models/Astronaut.glb';
      case ARShape.sphere:
        return 'https://github.com/google/model-viewer/raw/master/packages/shared-assets/models/reflective-sphere.glb';
      case ARShape.cylinder:
        return 'https://github.com/google/model-viewer/raw/master/packages/shared-assets/models/Horse.glb';
      case ARShape.cone:
        return 'https://github.com/google/model-viewer/raw/master/packages/shared-assets/models/MaterialsVariantsShoe.glb';
    }
  }

  Future<void> _clearAllObjects() async {
    for (final node in _nodes) {
      await _arObjectManager?.removeNode(node);
    }
    for (final anchor in _anchors) {
      await _arAnchorManager?.removeAnchor(anchor);
    }
    _nodes.clear();
    _anchors.clear();
    if (mounted) setState(() => _objectCount = 0);
  }

  @override
  void dispose() {
    _arSessionManager?.dispose();
    super.dispose();
  }
}

class _ARStatusOverlay extends StatelessWidget {
  final bool isTracking;
  final bool planeDetected;
  final int objectCount;

  const _ARStatusOverlay({
    required this.isTracking,
    required this.planeDetected,
    required this.objectCount,
  });

  @override
  Widget build(BuildContext context) {
    return Positioned(
      top: 16,
      left: 16,
      right: 16,
      child: Container(
        padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
        decoration: BoxDecoration(
          color: Colors.black.withOpacity(0.6),
          borderRadius: BorderRadius.circular(8),
        ),
        child: Row(
          children: [
            _StatusDot(
              color: isTracking ? Colors.green : Colors.yellow,
              label: isTracking ? 'Tracking' : 'Initializing',
            ),
            const SizedBox(width: 16),
            _StatusDot(
              color: planeDetected ? Colors.blue : Colors.grey,
              label: planeDetected ? 'Plane Found' : 'Scanning...',
            ),
            const Spacer(),
            Text(
              '$objectCount objects',
              style: const TextStyle(color: Colors.white, fontSize: 12),
            ),
          ],
        ),
      ),
    );
  }
}

class _StatusDot extends StatelessWidget {
  final Color color;
  final String label;

  const _StatusDot({required this.color, required this.label});

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        Container(
          width: 8,
          height: 8,
          decoration: BoxDecoration(color: color, shape: BoxShape.circle),
        ),
        const SizedBox(width: 4),
        Text(label, style: const TextStyle(color: Colors.white, fontSize: 11)),
      ],
    );
  }
}

class _ARControlsPanel extends StatelessWidget {
  final ARShape selectedShape;
  final Color selectedColor;
  final double selectedScale;
  final List<Color> availableColors;
  final ValueChanged<ARShape> onShapeChanged;
  final ValueChanged<Color> onColorChanged;
  final ValueChanged<double> onScaleChanged;

  const _ARControlsPanel({
    required this.selectedShape,
    required this.selectedColor,
    required this.selectedScale,
    required this.availableColors,
    required this.onShapeChanged,
    required this.onColorChanged,
    required this.onScaleChanged,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      decoration: BoxDecoration(
        color: Colors.black.withOpacity(0.8),
        borderRadius: const BorderRadius.vertical(top: Radius.circular(16)),
      ),
      padding: const EdgeInsets.all(16),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          // Shape selector
          const Text('Shape', style: TextStyle(color: Colors.white, fontSize: 12)),
          const SizedBox(height: 8),
          SingleChildScrollView(
            scrollDirection: Axis.horizontal,
            child: Row(
              children: ARShape.values.map((shape) {
                final isSelected = shape == selectedShape;
                return GestureDetector(
                  onTap: () => onShapeChanged(shape),
                  child: AnimatedContainer(
                    duration: const Duration(milliseconds: 200),
                    margin: const EdgeInsets.only(right: 8),
                    padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
                    decoration: BoxDecoration(
                      color: isSelected ? Colors.blue : Colors.white.withOpacity(0.2),
                      borderRadius: BorderRadius.circular(16),
                    ),
                    child: Text(
                      shape.name.toUpperCase(),
                      style: const TextStyle(color: Colors.white, fontSize: 12),
                    ),
                  ),
                );
              }).toList(),
            ),
          ),

          const SizedBox(height: 12),

          // Color selector
          const Text('Color', style: TextStyle(color: Colors.white, fontSize: 12)),
          const SizedBox(height: 8),
          Row(
            children: availableColors.map((color) {
              final isSelected = color == selectedColor;
              return GestureDetector(
                onTap: () => onColorChanged(color),
                child: AnimatedContainer(
                  duration: const Duration(milliseconds: 200),
                  width: 36,
                  height: 36,
                  margin: const EdgeInsets.only(right: 8),
                  decoration: BoxDecoration(
                    color: color,
                    shape: BoxShape.circle,
                    border: isSelected
                        ? Border.all(color: Colors.white, width: 3)
                        : null,
                  ),
                ),
              );
            }).toList(),
          ),

          const SizedBox(height: 12),

          // Scale slider
          Row(
            children: [
              const Text('Scale:', style: TextStyle(color: Colors.white, fontSize: 12)),
              Expanded(
                child: Slider(
                  value: selectedScale,
                  min: 0.05,
                  max: 0.5,
                  divisions: 9,
                  onChanged: onScaleChanged,
                  activeColor: Colors.blue,
                ),
              ),
              Text(
                '${selectedScale.toStringAsFixed(2)}m',
                style: const TextStyle(color: Colors.white, fontSize: 12),
              ),
            ],
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3204: AR Model Viewer

```dart
// lib/screens/ar_model_viewer_screen.dart
import 'package:flutter/material.dart';
import 'package:ar_flutter_plugin/ar_flutter_plugin.dart';
import 'package:ar_flutter_plugin/datatypes/config_planedetection.dart';
import 'package:ar_flutter_plugin/datatypes/node_types.dart';
import 'package:ar_flutter_plugin/managers/ar_anchor_manager.dart';
import 'package:ar_flutter_plugin/managers/ar_location_manager.dart';
import 'package:ar_flutter_plugin/managers/ar_object_manager.dart';
import 'package:ar_flutter_plugin/managers/ar_session_manager.dart';
import 'package:ar_flutter_plugin/models/ar_node.dart';
import 'package:vector_math/vector_math_64.dart' as vector_math;

class ARModel {
  final String name;
  final String description;
  final String uri;
  final double defaultScale;
  final String thumbnailUrl;

  const ARModel({
    required this.name,
    required this.description,
    required this.uri,
    required this.defaultScale,
    required this.thumbnailUrl,
  });
}

class ARModelViewerScreen extends StatefulWidget {
  const ARModelViewerScreen({super.key});

  @override
  State<ARModelViewerScreen> createState() => _ARModelViewerScreenState();
}

class _ARModelViewerScreenState extends State<ARModelViewerScreen> {
  ARSessionManager? _arSessionManager;
  ARObjectManager? _arObjectManager;

  ARNode? _currentNode;
  ARModel? _selectedModel;
  bool _isPlaced = false;
  bool _showModelList = true;

  final List<ARModel> _models = [
    const ARModel(
      name: 'Astronaut',
      description: 'NASA Astronaut in space suit',
      uri: 'https://github.com/google/model-viewer/raw/master/packages/shared-assets/models/Astronaut.glb',
      defaultScale: 0.5,
      thumbnailUrl: '',
    ),
    const ARModel(
      name: 'Horse',
      description: 'Animated horse model',
      uri: 'https://github.com/google/model-viewer/raw/master/packages/shared-assets/models/Horse.glb',
      defaultScale: 0.3,
      thumbnailUrl: '',
    ),
    const ARModel(
      name: 'Reflective Sphere',
      description: 'Metallic reflective sphere',
      uri: 'https://github.com/google/model-viewer/raw/master/packages/shared-assets/models/reflective-sphere.glb',
      defaultScale: 0.2,
      thumbnailUrl: '',
    ),
    const ARModel(
      name: 'Shoe',
      description: 'Material variants shoe',
      uri: 'https://github.com/google/model-viewer/raw/master/packages/shared-assets/models/MaterialsVariantsShoe.glb',
      defaultScale: 0.3,
      thumbnailUrl: '',
    ),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(_selectedModel?.name ?? 'AR Model Viewer'),
        backgroundColor: Colors.black,
        foregroundColor: Colors.white,
        actions: [
          if (_isPlaced)
            IconButton(
              icon: const Icon(Icons.refresh),
              onPressed: _removeCurrentModel,
              tooltip: 'Remove Model',
            ),
          IconButton(
            icon: Icon(_showModelList ? Icons.visibility_off : Icons.list),
            onPressed: () => setState(() => _showModelList = !_showModelList),
            tooltip: 'Toggle Model List',
          ),
        ],
      ),
      body: Stack(
        children: [
          ARView(
            onARViewCreated: _onARViewCreated,
            planeDetectionConfig: PlaneDetectionConfig.horizontal,
          ),

          // Instructions
          if (!_isPlaced)
            Positioned(
              top: 20,
              left: 20,
              right: 20,
              child: _InstructionBanner(
                message: _selectedModel == null
                    ? 'Select a model from the list below'
                    : 'Point at a surface and tap to place',
              ),
            ),

          // Model list
          if (_showModelList)
            Positioned(
              bottom: 0,
              left: 0,
              right: 0,
              child: _ModelListPanel(
                models: _models,
                selectedModel: _selectedModel,
                onModelSelected: (model) {
                  setState(() {
                    _selectedModel = model;
                    _showModelList = false;
                  });
                },
              ),
            ),

          // AR Controls
          if (_isPlaced)
            Positioned(
              bottom: 20,
              left: 0,
              right: 0,
              child: _ARRotationControls(
                onRotateLeft: () => _rotateModel(-0.2),
                onRotateRight: () => _rotateModel(0.2),
                onScaleUp: () => _scaleModel(1.1),
                onScaleDown: () => _scaleModel(0.9),
              ),
            ),
        ],
      ),
    );
  }

  void _onARViewCreated(
    ARSessionManager sessionManager,
    ARObjectManager objectManager,
    ARAnchorManager anchorManager,
    ARLocationManager locationManager,
  ) {
    _arSessionManager = sessionManager;
    _arObjectManager = objectManager;

    _arSessionManager!.onInitialize(
      showAnimatedGuide: true,
      showFeaturePoints: false,
      showPlanes: true,
      showWorldOrigin: false,
      handleRotation: true,
      handlePans: true,
      handleTaps: true,
    );

    _arObjectManager!.onInitialize();
    _arObjectManager!.onNodeTap = _handleTap;
  }

  void _handleTap(List<String> nodes) async {
    if (_selectedModel == null || _currentNode != null) return;
    await _placeModel();
  }

  Future<void> _placeModel() async {
    if (_selectedModel == null || _arObjectManager == null) return;

    final scale = _selectedModel!.defaultScale;
    final node = ARNode(
      type: NodeType.webGLB,
      uri: _selectedModel!.uri,
      scale: vector_math.Vector3(scale, scale, scale),
      position: vector_math.Vector3(0, 0, -1.5),
      rotation: vector_math.Vector4(0, 0, 0, 1),
      name: 'current_model',
    );

    final result = await _arObjectManager!.addNode(node);
    if (result == true) {
      setState(() {
        _currentNode = node;
        _isPlaced = true;
      });
    }
  }

  Future<void> _removeCurrentModel() async {
    if (_currentNode == null) return;
    await _arObjectManager?.removeNode(_currentNode!);
    setState(() {
      _currentNode = null;
      _isPlaced = false;
    });
  }

  void _rotateModel(double angle) {
    if (_currentNode == null) return;
    // Update rotation - simplified
    final currentRotation = _currentNode!.rotation;
    _currentNode!.rotation = vector_math.Vector4(
      currentRotation.x,
      angle,
      currentRotation.z,
      currentRotation.w,
    );
  }

  void _scaleModel(double factor) {
    if (_currentNode == null) return;
    final current = _currentNode!.scale;
    _currentNode!.scale = vector_math.Vector3(
      current.x * factor,
      current.y * factor,
      current.z * factor,
    );
  }

  @override
  void dispose() {
    _arSessionManager?.dispose();
    super.dispose();
  }
}

class _InstructionBanner extends StatelessWidget {
  final String message;

  const _InstructionBanner({required this.message});

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(12),
      decoration: BoxDecoration(
        color: Colors.black.withOpacity(0.6),
        borderRadius: BorderRadius.circular(8),
      ),
      child: Row(
        children: [
          const Icon(Icons.info_outline, color: Colors.white, size: 16),
          const SizedBox(width: 8),
          Expanded(
            child: Text(
              message,
              style: const TextStyle(color: Colors.white, fontSize: 13),
            ),
          ),
        ],
      ),
    );
  }
}

class _ModelListPanel extends StatelessWidget {
  final List<ARModel> models;
  final ARModel? selectedModel;
  final ValueChanged<ARModel> onModelSelected;

  const _ModelListPanel({
    required this.models,
    required this.selectedModel,
    required this.onModelSelected,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      height: 140,
      decoration: BoxDecoration(
        color: Colors.black.withOpacity(0.85),
        borderRadius: const BorderRadius.vertical(top: Radius.circular(16)),
      ),
      child: Column(
        children: [
          const Padding(
            padding: EdgeInsets.all(8),
            child: Text(
              'Select a Model',
              style: TextStyle(
                color: Colors.white,
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
          Expanded(
            child: ListView.builder(
              scrollDirection: Axis.horizontal,
              padding: const EdgeInsets.symmetric(horizontal: 8),
              itemCount: models.length,
              itemBuilder: (context, index) {
                final model = models[index];
                final isSelected = model == selectedModel;

                return GestureDetector(
                  onTap: () => onModelSelected(model),
                  child: AnimatedContainer(
                    duration: const Duration(milliseconds: 200),
                    width: 90,
                    margin: const EdgeInsets.only(right: 8),
                    decoration: BoxDecoration(
                      color: isSelected
                          ? Colors.blue.withOpacity(0.3)
                          : Colors.white.withOpacity(0.1),
                      borderRadius: BorderRadius.circular(8),
                      border: isSelected
                          ? Border.all(color: Colors.blue, width: 2)
                          : null,
                    ),
                    child: Column(
                      mainAxisAlignment: MainAxisAlignment.center,
                      children: [
                        const Icon(
                          Icons.threed_rotation,
                          color: Colors.white,
                          size: 32,
                        ),
                        const SizedBox(height: 4),
                        Text(
                          model.name,
                          style: const TextStyle(
                            color: Colors.white,
                            fontSize: 11,
                          ),
                          textAlign: TextAlign.center,
                          maxLines: 2,
                          overflow: TextOverflow.ellipsis,
                        ),
                      ],
                    ),
                  ),
                );
              },
            ),
          ),
        ],
      ),
    );
  }
}

class _ARRotationControls extends StatelessWidget {
  final VoidCallback onRotateLeft;
  final VoidCallback onRotateRight;
  final VoidCallback onScaleUp;
  final VoidCallback onScaleDown;

  const _ARRotationControls({
    required this.onRotateLeft,
    required this.onRotateRight,
    required this.onScaleUp,
    required this.onScaleDown,
  });

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        _ControlButton(icon: Icons.rotate_left, onPressed: onRotateLeft),
        const SizedBox(width: 8),
        _ControlButton(icon: Icons.zoom_out, onPressed: onScaleDown),
        const SizedBox(width: 8),
        _ControlButton(icon: Icons.zoom_in, onPressed: onScaleUp),
        const SizedBox(width: 8),
        _ControlButton(icon: Icons.rotate_right, onPressed: onRotateRight),
      ],
    );
  }
}

class _ControlButton extends StatelessWidget {
  final IconData icon;
  final VoidCallback onPressed;

  const _ControlButton({required this.icon, required this.onPressed});

  @override
  Widget build(BuildContext context) {
    return Material(
      color: Colors.black.withOpacity(0.6),
      shape: const CircleBorder(),
      child: InkWell(
        onTap: onPressed,
        customBorder: const CircleBorder(),
        child: Padding(
          padding: const EdgeInsets.all(12),
          child: Icon(icon, color: Colors.white, size: 24),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3205: AR Distance Measurement

```dart
// lib/screens/ar_measurement_screen.dart
import 'dart:math';
import 'package:flutter/material.dart';
import 'package:ar_flutter_plugin/ar_flutter_plugin.dart';
import 'package:ar_flutter_plugin/datatypes/config_planedetection.dart';
import 'package:ar_flutter_plugin/datatypes/node_types.dart';
import 'package:ar_flutter_plugin/managers/ar_anchor_manager.dart';
import 'package:ar_flutter_plugin/managers/ar_location_manager.dart';
import 'package:ar_flutter_plugin/managers/ar_object_manager.dart';
import 'package:ar_flutter_plugin/managers/ar_session_manager.dart';
import 'package:ar_flutter_plugin/models/ar_node.dart';
import 'package:vector_math/vector_math_64.dart' as vector_math;

class MeasurementPoint {
  final String id;
  final vector_math.Vector3 position;
  final ARNode node;

  const MeasurementPoint({
    required this.id,
    required this.position,
    required this.node,
  });
}

class ARMeasurementScreen extends StatefulWidget {
  const ARMeasurementScreen({super.key});

  @override
  State<ARMeasurementScreen> createState() => _ARMeasurementScreenState();
}

class _ARMeasurementScreenState extends State<ARMeasurementScreen> {
  ARSessionManager? _arSessionManager;
  ARObjectManager? _arObjectManager;

  final List<MeasurementPoint> _measurementPoints = [];
  double? _measuredDistance;
  int _tapCount = 0;

  @override
  void dispose() {
    _arSessionManager?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('AR Measurement'),
        backgroundColor: Colors.black,
        foregroundColor: Colors.white,
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: _reset,
            tooltip: 'Reset',
          ),
        ],
      ),
      body: Stack(
        children: [
          ARView(
            onARViewCreated: _onARViewCreated,
            planeDetectionConfig: PlaneDetectionConfig.horizontal,
          ),

          // Instruction overlay
          Positioned(
            top: 16,
            left: 16,
            right: 16,
            child: _MeasurementInstructions(tapCount: _tapCount),
          ),

          // Crosshair
          const Center(
            child: Icon(Icons.add_circle_outline, color: Colors.white, size: 40),
          ),

          // Result display
          if (_measuredDistance != null)
            Positioned(
              bottom: 100,
              left: 0,
              right: 0,
              child: _DistanceDisplay(distance: _measuredDistance!),
            ),

          // Tap button
          Positioned(
            bottom: 30,
            left: 0,
            right: 0,
            child: Center(
              child: ElevatedButton.icon(
                onPressed: _tapCount < 2 ? _placeMeasurementPoint : _reset,
                icon: Icon(_tapCount < 2 ? Icons.add_location : Icons.refresh),
                label: Text(
                  _tapCount == 0
                      ? 'Place Start Point'
                      : _tapCount == 1
                          ? 'Place End Point'
                          : 'Measure Again',
                ),
                style: ElevatedButton.styleFrom(
                  backgroundColor: Colors.blue,
                  foregroundColor: Colors.white,
                  padding: const EdgeInsets.symmetric(
                    horizontal: 24,
                    vertical: 12,
                  ),
                ),
              ),
            ),
          ),
        ],
      ),
    );
  }

  void _onARViewCreated(
    ARSessionManager sessionManager,
    ARObjectManager objectManager,
    ARAnchorManager anchorManager,
    ARLocationManager locationManager,
  ) {
    _arSessionManager = sessionManager;
    _arObjectManager = objectManager;

    _arSessionManager!.onInitialize(
      showAnimatedGuide: true,
      showFeaturePoints: true,
      showPlanes: true,
      showWorldOrigin: false,
      handleRotation: false,
      handlePans: false,
      handleTaps: true,
    );

    _arObjectManager!.onInitialize();
  }

  Future<void> _placeMeasurementPoint() async {
    if (_arObjectManager == null || _tapCount >= 2) return;

    final pointId = 'point_$_tapCount';
    final color = _tapCount == 0 ? 'green' : 'red';
    final position = vector_math.Vector3(
      (_tapCount == 0 ? -0.3 : 0.3),
      0,
      -1.0,
    );

    final node = ARNode(
      type: NodeType.sphere,
      scale: vector_math.Vector3(0.05, 0.05, 0.05),
      position: position,
      rotation: vector_math.Vector4(0, 0, 0, 1),
      name: pointId,
    );

    final result = await _arObjectManager!.addNode(node);
    if (result == true) {
      _measurementPoints.add(MeasurementPoint(
        id: pointId,
        position: position,
        node: node,
      ));

      setState(() => _tapCount++);

      if (_tapCount == 2) {
        _calculateDistance();
      }
    }
  }

  void _calculateDistance() {
    if (_measurementPoints.length < 2) return;

    final p1 = _measurementPoints[0].position;
    final p2 = _measurementPoints[1].position;

    final distance = sqrt(
      pow(p2.x - p1.x, 2) +
          pow(p2.y - p1.y, 2) +
          pow(p2.z - p1.z, 2),
    );

    setState(() => _measuredDistance = distance);
  }

  Future<void> _reset() async {
    for (final point in _measurementPoints) {
      await _arObjectManager?.removeNode(point.node);
    }
    _measurementPoints.clear();
    setState(() {
      _tapCount = 0;
      _measuredDistance = null;
    });
  }
}

class _MeasurementInstructions extends StatelessWidget {
  final int tapCount;

  const _MeasurementInstructions({required this.tapCount});

  @override
  Widget build(BuildContext context) {
    final message = tapCount == 0
        ? 'Tap "Place Start Point" to set the starting position'
        : tapCount == 1
            ? 'Now tap "Place End Point" to complete measurement'
            : 'Measurement complete! Tap "Measure Again" to restart';

    return Container(
      padding: const EdgeInsets.all(12),
      decoration: BoxDecoration(
        color: Colors.black.withOpacity(0.6),
        borderRadius: BorderRadius.circular(8),
      ),
      child: Text(
        message,
        style: const TextStyle(color: Colors.white, fontSize: 13),
        textAlign: TextAlign.center,
      ),
    );
  }
}

class _DistanceDisplay extends StatelessWidget {
  final double distance;

  const _DistanceDisplay({required this.distance});

  @override
  Widget build(BuildContext context) {
    final distanceCm = (distance * 100).toStringAsFixed(1);
    final distanceM = distance.toStringAsFixed(3);

    return Center(
      child: Container(
        padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 16),
        decoration: BoxDecoration(
          color: Colors.blue.withOpacity(0.9),
          borderRadius: BorderRadius.circular(12),
          boxShadow: [
            BoxShadow(
              color: Colors.black.withOpacity(0.3),
              blurRadius: 8,
              offset: const Offset(0, 4),
            ),
          ],
        ),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const Text(
              'Distance',
              style: TextStyle(color: Colors.white70, fontSize: 12),
            ),
            Text(
              '$distanceCm cm',
              style: const TextStyle(
                color: Colors.white,
                fontSize: 32,
                fontWeight: FontWeight.bold,
              ),
            ),
            Text(
              '$distanceM m',
              style: const TextStyle(color: Colors.white70, fontSize: 14),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3206: VR 360-Degree Image Viewer

```dart
// lib/screens/vr_360_viewer_screen.dart
import 'dart:math';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:sensors_plus/sensors_plus.dart';

class VR360Image {
  final String title;
  final String url;
  final String description;

  const VR360Image({
    required this.title,
    required this.url,
    required this.description,
  });
}

class VR360ViewerScreen extends StatefulWidget {
  const VR360ViewerScreen({super.key});

  @override
  State<VR360ViewerScreen> createState() => _VR360ViewerScreenState();
}

class _VR360ViewerScreenState extends State<VR360ViewerScreen>
    with TickerProviderStateMixin {
  final List<VR360Image> _images = [
    const VR360Image(
      title: 'Mountain Panorama',
      url: 'https://upload.wikimedia.org/wikipedia/commons/thumb/a/af/All_Gizah_Pyramids.jpg/2560px-All_Gizah_Pyramids.jpg',
      description: 'Beautiful mountain panoramic view',
    ),
    const VR360Image(
      title: 'City Skyline',
      url: 'https://upload.wikimedia.org/wikipedia/commons/thumb/9/9d/Night_Sky_%28Unsplash%29.jpg/2560px-Night_Sky_%28Unsplash%29.jpg',
      description: 'Urban city panoramic view',
    ),
    const VR360Image(
      title: 'Ocean View',
      url: 'https://upload.wikimedia.org/wikipedia/commons/thumb/1/1e/Ocean_City%2C_Maryland.jpg/2560px-Ocean_City%2C_Maryland.jpg',
      description: 'Peaceful ocean panoramic view',
    ),
  ];

  int _currentImageIndex = 0;
  double _horizontalAngle = 0.0;
  double _verticalAngle = 0.0;
  bool _isGyroscopeMode = false;
  bool _isFullscreen = false;

  late final AnimationController _transitionController;
  double _scale = 1.0;
  Offset _lastFocalPoint = Offset.zero;
  double _lastScale = 1.0;

  VR360Image get _currentImage => _images[_currentImageIndex];

  @override
  void initState() {
    super.initState();
    _transitionController = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 500),
    );

    if (_isGyroscopeMode) {
      _setupGyroscope();
    }
  }

  void _setupGyroscope() {
    gyroscopeEventStream().listen((GyroscopeEvent event) {
      if (!mounted || !_isGyroscopeMode) return;
      setState(() {
        _horizontalAngle += event.z * 0.02;
        _verticalAngle = (_verticalAngle + event.x * 0.02).clamp(-0.5, 0.5);
      });
    });
  }

  void _toggleFullscreen() {
    setState(() => _isFullscreen = !_isFullscreen);
    if (_isFullscreen) {
      SystemChrome.setEnabledSystemUIMode(SystemUiMode.immersiveSticky);
      SystemChrome.setPreferredOrientations([
        DeviceOrientation.landscapeLeft,
        DeviceOrientation.landscapeRight,
      ]);
    } else {
      SystemChrome.setEnabledSystemUIMode(SystemUiMode.edgeToEdge);
      SystemChrome.setPreferredOrientations([DeviceOrientation.portraitUp]);
    }
  }

  @override
  void dispose() {
    _transitionController.dispose();
    SystemChrome.setEnabledSystemUIMode(SystemUiMode.edgeToEdge);
    SystemChrome.setPreferredOrientations([DeviceOrientation.portraitUp]);
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      appBar: _isFullscreen
          ? null
          : AppBar(
              title: Text(_currentImage.title),
              backgroundColor: Colors.black,
              foregroundColor: Colors.white,
              actions: [
                IconButton(
                  icon: Icon(_isGyroscopeMode ? Icons.screen_rotation : Icons.touch_app),
                  onPressed: () {
                    setState(() => _isGyroscopeMode = !_isGyroscopeMode);
                    if (_isGyroscopeMode) _setupGyroscope();
                  },
                  tooltip: _isGyroscopeMode ? 'Touch Mode' : 'Gyroscope Mode',
                ),
                IconButton(
                  icon: const Icon(Icons.fullscreen),
                  onPressed: _toggleFullscreen,
                ),
              ],
            ),
      body: Stack(
        children: [
          // 360 Panoramic Viewer
          _Panorama360Widget(
            imageUrl: _currentImage.url,
            horizontalAngle: _horizontalAngle,
            verticalAngle: _verticalAngle,
            scale: _scale,
            isGyroscopeMode: _isGyroscopeMode,
            onDragUpdate: (dx, dy) {
              if (!_isGyroscopeMode) {
                setState(() {
                  _horizontalAngle += dx * 0.005;
                  _verticalAngle = (_verticalAngle - dy * 0.005).clamp(-0.5, 0.5);
                });
              }
            },
          ),

          // Image gallery dots
          if (!_isFullscreen)
            Positioned(
              bottom: 100,
              left: 0,
              right: 0,
              child: Row(
                mainAxisAlignment: MainAxisAlignment.center,
                children: List.generate(_images.length, (index) {
                  return GestureDetector(
                    onTap: () => setState(() {
                      _currentImageIndex = index;
                      _horizontalAngle = 0;
                      _verticalAngle = 0;
                    }),
                    child: AnimatedContainer(
                      duration: const Duration(milliseconds: 200),
                      width: _currentImageIndex == index ? 20 : 8,
                      height: 8,
                      margin: const EdgeInsets.symmetric(horizontal: 4),
                      decoration: BoxDecoration(
                        color: _currentImageIndex == index
                            ? Colors.white
                            : Colors.white38,
                        borderRadius: BorderRadius.circular(4),
                      ),
                    ),
                  );
                }),
              ),
            ),

          // Navigation arrows
          if (!_isFullscreen) ...[
            Positioned(
              left: 8,
              top: 0,
              bottom: 0,
              child: Center(
                child: _NavArrow(
                  icon: Icons.chevron_left,
                  onPressed: _currentImageIndex > 0
                      ? () => setState(() {
                            _currentImageIndex--;
                            _horizontalAngle = 0;
                            _verticalAngle = 0;
                          })
                      : null,
                ),
              ),
            ),
            Positioned(
              right: 8,
              top: 0,
              bottom: 0,
              child: Center(
                child: _NavArrow(
                  icon: Icons.chevron_right,
                  onPressed: _currentImageIndex < _images.length - 1
                      ? () => setState(() {
                            _currentImageIndex++;
                            _horizontalAngle = 0;
                            _verticalAngle = 0;
                          })
                      : null,
                ),
              ),
            ),
          ],

          // Angle indicator
          Positioned(
            top: _isFullscreen ? 16 : 16,
            right: 16,
            child: Container(
              padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
              decoration: BoxDecoration(
                color: Colors.black54,
                borderRadius: BorderRadius.circular(4),
              ),
              child: Text(
                '${(_horizontalAngle * 180 / pi).toInt() % 360}°',
                style: const TextStyle(color: Colors.white, fontSize: 12),
              ),
            ),
          ),

          // Fullscreen toggle button when in fullscreen
          if (_isFullscreen)
            Positioned(
              top: 16,
              left: 16,
              child: IconButton(
                icon: const Icon(Icons.fullscreen_exit, color: Colors.white),
                onPressed: _toggleFullscreen,
              ),
            ),
        ],
      ),
    );
  }
}

class _Panorama360Widget extends StatelessWidget {
  final String imageUrl;
  final double horizontalAngle;
  final double verticalAngle;
  final double scale;
  final bool isGyroscopeMode;
  final void Function(double dx, double dy) onDragUpdate;

  const _Panorama360Widget({
    required this.imageUrl,
    required this.horizontalAngle,
    required this.verticalAngle,
    required this.scale,
    required this.isGyroscopeMode,
    required this.onDragUpdate,
  });

  @override
  Widget build(BuildContext context) {
    final size = MediaQuery.of(context).size;

    return GestureDetector(
      onPanUpdate: isGyroscopeMode
          ? null
          : (details) => onDragUpdate(details.delta.dx, details.delta.dy),
      child: ClipRect(
        child: OverflowBox(
          maxWidth: double.infinity,
          maxHeight: double.infinity,
          child: Transform(
            transform: Matrix4.identity()
              ..translate(
                -horizontalAngle * size.width * 0.5,
                verticalAngle * size.height,
              ),
            child: Image.network(
              imageUrl,
              width: size.width * 3,
              height: size.height,
              fit: BoxFit.cover,
              loadingBuilder: (context, child, loadingProgress) {
                if (loadingProgress == null) return child;
                return Center(
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      CircularProgressIndicator(
                        value: loadingProgress.expectedTotalBytes != null
                            ? loadingProgress.cumulativeBytesLoaded /
                                loadingProgress.expectedTotalBytes!
                            : null,
                        color: Colors.white,
                      ),
                      const SizedBox(height: 16),
                      const Text(
                        'Loading panorama...',
                        style: TextStyle(color: Colors.white),
                      ),
                    ],
                  ),
                );
              },
              errorBuilder: (context, error, stackTrace) {
                return const Center(
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      Icon(Icons.panorama, color: Colors.white54, size: 64),
                      SizedBox(height: 16),
                      Text(
                        'Failed to load panorama',
                        style: TextStyle(color: Colors.white54),
                      ),
                    ],
                  ),
                );
              },
            ),
          ),
        ),
      ),
    );
  }
}

class _NavArrow extends StatelessWidget {
  final IconData icon;
  final VoidCallback? onPressed;

  const _NavArrow({required this.icon, this.onPressed});

  @override
  Widget build(BuildContext context) {
    return Material(
      color: Colors.black38,
      shape: const CircleBorder(),
      child: InkWell(
        onTap: onPressed,
        customBorder: const CircleBorder(),
        child: Padding(
          padding: const EdgeInsets.all(8),
          child: Icon(
            icon,
            color: onPressed != null ? Colors.white : Colors.white30,
            size: 32,
          ),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3207: AR Tracking และ Anchor Management

```dart
// lib/services/ar_anchor_service.dart
import 'package:ar_flutter_plugin/managers/ar_anchor_manager.dart';
import 'package:ar_flutter_plugin/managers/ar_object_manager.dart';
import 'package:ar_flutter_plugin/models/ar_anchor.dart';
import 'package:ar_flutter_plugin/models/ar_hittest_result.dart';
import 'package:ar_flutter_plugin/models/ar_node.dart';
import 'package:ar_flutter_plugin/datatypes/node_types.dart';
import 'package:vector_math/vector_math_64.dart' as vector_math;

class ARAnchorService {
  final ARAnchorManager anchorManager;
  final ARObjectManager objectManager;

  final Map<String, ARAnchor> _anchors = {};
  final Map<String, List<ARNode>> _anchorNodes = {};

  ARAnchorService({
    required this.anchorManager,
    required this.objectManager,
  });

  /// Add an anchor at a hit test result position
  Future<ARAnchor?> addAnchor(ARHitTestResult hitResult) async {
    final anchor = ARPlaneAnchor(transformation: hitResult.worldTransform);
    final didAddAnchor = await anchorManager.addAnchor(anchor);

    if (didAddAnchor == true) {
      _anchors[anchor.name] = anchor;
      _anchorNodes[anchor.name] = [];
      return anchor;
    }

    return null;
  }

  /// Attach a node to an anchor
  Future<ARNode?> attachNodeToAnchor(
    ARAnchor anchor, {
    required String uri,
    double scale = 0.2,
  }) async {
    final node = ARNode(
      type: NodeType.webGLB,
      uri: uri,
      scale: vector_math.Vector3(scale, scale, scale),
      rotation: vector_math.Vector4(0, 0, 0, 1),
      position: vector_math.Vector3(0, 0, 0),
    );

    final didAddNode = await objectManager.addNode(node, planeAnchor: anchor as ARPlaneAnchor);

    if (didAddNode == true) {
      _anchorNodes[anchor.name]?.add(node);
      return node;
    }

    return null;
  }

  /// Remove all nodes from an anchor
  Future<void> clearAnchor(String anchorName) async {
    final nodes = _anchorNodes[anchorName] ?? [];
    for (final node in nodes) {
      await objectManager.removeNode(node);
    }
    _anchorNodes[anchorName]?.clear();
  }

  /// Remove anchor and all its nodes
  Future<void> removeAnchor(String anchorName) async {
    await clearAnchor(anchorName);

    final anchor = _anchors[anchorName];
    if (anchor != null) {
      await anchorManager.removeAnchor(anchor);
      _anchors.remove(anchorName);
      _anchorNodes.remove(anchorName);
    }
  }

  /// Remove all anchors
  Future<void> removeAllAnchors() async {
    for (final name in List.from(_anchors.keys)) {
      await removeAnchor(name);
    }
  }

  /// Get all anchor names
  List<String> get anchorNames => _anchors.keys.toList();

  /// Check if any anchors exist
  bool get hasAnchors => _anchors.isNotEmpty;
}
```

---

**← [Part 82](part-82-real-time-collaboration.md)**
**ต่อไป: [Part 84 →](part-84-advanced-camera.md)**
