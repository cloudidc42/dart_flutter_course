# Part 68: Flutter ML with TensorFlow Lite
## ขั้นตอนที่ 2601-2640

## 🎯 เป้าหมายของ Part นี้
- ตั้งค่า tflite_flutter สำหรับ on-device ML
- ทำ Image Classification ด้วย MobileNet
- ทำ Object Detection ด้วย YOLO
- ทำ Text Classification
- เพิ่มประสิทธิภาพ on-device inference
- ผสาน Camera feed กับ real-time ML

---

## ขั้นตอนที่ 2601: Setup tflite_flutter

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  tflite_flutter: ^0.10.4
  tflite_flutter_helper: ^0.3.1
  camera: ^0.10.5+5
  image: ^4.1.7
  path_provider: ^2.1.1

flutter:
  assets:
    - assets/models/
    - assets/labels/
```

```dart
// lib/ml/interpreter_service.dart
import 'dart:typed_data';
import 'package:flutter/services.dart';
import 'package:tflite_flutter/tflite_flutter.dart';

/// Service สำหรับจัดการ TFLite interpreters
class InterpreterService {
  static final InterpreterService _instance = InterpreterService._internal();
  factory InterpreterService() => _instance;
  InterpreterService._internal();

  final Map<String, Interpreter> _interpreters = {};
  final Map<String, List<String>> _labels = {};

  /// โหลด model จาก assets
  Future<Interpreter> loadModel(
    String modelName, {
    int numThreads = 4,
    bool useGpu = false,
    bool useNnapi = false,
  }) async {
    if (_interpreters.containsKey(modelName)) {
      return _interpreters[modelName]!;
    }

    final options = InterpreterOptions()
      ..threads = numThreads
      ..useNnApiForAndroid = useNnapi;

    if (useGpu) {
      options.addDelegate(GpuDelegate());
    }

    final interpreter = await Interpreter.fromAsset(
      'assets/models/$modelName.tflite',
      options: options,
    );

    _interpreters[modelName] = interpreter;
    return interpreter;
  }

  /// โหลด labels จาก assets
  Future<List<String>> loadLabels(String labelsFile) async {
    if (_labels.containsKey(labelsFile)) {
      return _labels[labelsFile]!;
    }

    final labelsData =
        await rootBundle.loadString('assets/labels/$labelsFile');
    final labels = labelsData
        .split('\n')
        .where((line) => line.trim().isNotEmpty)
        .toList();

    _labels[labelsFile] = labels;
    return labels;
  }

  /// ดูข้อมูล model
  void inspectModel(String modelName) {
    final interpreter = _interpreters[modelName];
    if (interpreter == null) {
      print('Model $modelName not loaded');
      return;
    }

    print('=== Model: $modelName ===');
    print('Input tensors:');
    for (var i = 0; i < interpreter.getInputTensors().length; i++) {
      final tensor = interpreter.getInputTensor(i);
      print('  [$i] name: ${tensor.name}, shape: ${tensor.shape}, type: ${tensor.type}');
    }

    print('Output tensors:');
    for (var i = 0; i < interpreter.getOutputTensors().length; i++) {
      final tensor = interpreter.getOutputTensor(i);
      print('  [$i] name: ${tensor.name}, shape: ${tensor.shape}, type: ${tensor.type}');
    }
  }

  void dispose() {
    for (final interpreter in _interpreters.values) {
      interpreter.close();
    }
    _interpreters.clear();
    _labels.clear();
  }
}
```

---

## ขั้นตอนที่ 2602: Image Classification ด้วย MobileNet

```dart
// lib/ml/image_classifier.dart
import 'dart:typed_data';
import 'package:flutter/material.dart';
import 'package:image/image.dart' as img;
import 'package:tflite_flutter/tflite_flutter.dart';

class ClassificationResult {
  final String label;
  final double confidence;
  final int index;

  const ClassificationResult({
    required this.label,
    required this.confidence,
    required this.index,
  });

  @override
  String toString() =>
      'ClassificationResult(label: $label, confidence: ${(confidence * 100).toStringAsFixed(1)}%)';
}

class MobileNetClassifier {
  static const int INPUT_SIZE = 224;
  static const int NUM_CLASSES = 1001; // ImageNet classes

  Interpreter? _interpreter;
  List<String> _labels = [];

  bool get isInitialized => _interpreter != null;

  Future<void> initialize() async {
    final service = InterpreterService();
    _interpreter = await service.loadModel('mobilenet_v2');
    _labels = await service.loadLabels('imagenet_labels.txt');
  }

  /// Classify image จาก Uint8List (raw image bytes)
  Future<List<ClassificationResult>> classify(
    Uint8List imageBytes, {
    int topK = 5,
  }) async {
    if (_interpreter == null) {
      throw StateError('Classifier not initialized');
    }

    // Decode image
    final image = img.decodeImage(imageBytes);
    if (image == null) throw ArgumentError('Invalid image data');

    // Preprocess image
    final input = _preprocessImage(image);

    // Prepare output tensor
    final output = [List<double>.filled(NUM_CLASSES, 0.0)];

    // Run inference
    final stopwatch = Stopwatch()..start();
    _interpreter!.run(input, output);
    stopwatch.stop();

    print('Inference time: ${stopwatch.elapsedMilliseconds}ms');

    // Post-process results
    return _processOutput(output[0], topK: topK);
  }

  /// Preprocess image สำหรับ MobileNet
  List<List<List<List<double>>>> _preprocessImage(img.Image image) {
    // Resize
    final resized = img.copyResize(
      image,
      width: INPUT_SIZE,
      height: INPUT_SIZE,
    );

    // Normalize pixels to [-1, 1] (MobileNet V2)
    return [
      List.generate(
        INPUT_SIZE,
        (y) => List.generate(
          INPUT_SIZE,
          (x) {
            final pixel = resized.getPixel(x, y);
            return [
              (pixel.r / 127.5) - 1.0,
              (pixel.g / 127.5) - 1.0,
              (pixel.b / 127.5) - 1.0,
            ];
          },
        ),
      ),
    ];
  }

  List<ClassificationResult> _processOutput(
    List<double> output, {
    int topK = 5,
  }) {
    // สร้าง list ของ (index, confidence)
    final indexed = output.asMap().entries.toList()
      ..sort((a, b) => b.value.compareTo(a.value));

    return indexed.take(topK).map((entry) {
      final label = entry.key < _labels.length
          ? _labels[entry.key]
          : 'Unknown (${entry.key})';
      return ClassificationResult(
        label: label,
        confidence: entry.value,
        index: entry.key,
      );
    }).toList();
  }

  void dispose() {
    _interpreter?.close();
    _interpreter = null;
  }
}
```

---

## ขั้นตอนที่ 2603: Object Detection ด้วย YOLO

```dart
// lib/ml/object_detector.dart
import 'dart:typed_data';
import 'package:flutter/material.dart';
import 'package:image/image.dart' as img;
import 'package:tflite_flutter/tflite_flutter.dart';

class DetectedObject {
  final String label;
  final double confidence;
  final Rect boundingBox; // normalized [0, 1]
  final int classIndex;

  const DetectedObject({
    required this.label,
    required this.confidence,
    required this.boundingBox,
    required this.classIndex,
  });

  @override
  String toString() => 'DetectedObject(label: $label, '
      'confidence: ${(confidence * 100).toStringAsFixed(1)}%, '
      'bbox: (${boundingBox.left.toStringAsFixed(2)}, '
      '${boundingBox.top.toStringAsFixed(2)}, '
      '${boundingBox.right.toStringAsFixed(2)}, '
      '${boundingBox.bottom.toStringAsFixed(2)}))';
}

class YoloDetector {
  static const int INPUT_SIZE = 416; // YOLOv5s input size
  static const int NUM_CLASSES = 80; // COCO classes
  static const double CONFIDENCE_THRESHOLD = 0.5;
  static const double IOU_THRESHOLD = 0.45;

  Interpreter? _interpreter;
  List<String> _labels = [];

  bool get isInitialized => _interpreter != null;

  Future<void> initialize() async {
    final service = InterpreterService();
    _interpreter = await service.loadModel('yolov5s_int8');
    _labels = await service.loadLabels('coco_labels.txt');
  }

  Future<List<DetectedObject>> detect(Uint8List imageBytes) async {
    if (_interpreter == null) {
      throw StateError('Detector not initialized');
    }

    final image = img.decodeImage(imageBytes);
    if (image == null) throw ArgumentError('Invalid image data');

    final originalWidth = image.width;
    final originalHeight = image.height;

    // Preprocess
    final input = _preprocessImage(image);

    // YOLOv5 output: [1, 25200, 85] for 80 classes + 5 (x, y, w, h, conf)
    final outputShape = _interpreter!.getOutputTensor(0).shape;
    final outputBuffer = [
      List.generate(
        outputShape[1],
        (_) => List<double>.filled(outputShape[2], 0.0),
      ),
    ];

    // Run inference
    _interpreter!.run(input, outputBuffer);

    // Post-process
    final rawDetections = _processYoloOutput(outputBuffer[0]);
    final afterNms = _nonMaxSuppression(rawDetections);

    return afterNms.map((det) {
      final label = det['classIndex'] < _labels.length
          ? _labels[det['classIndex'] as int]
          : 'Unknown';

      return DetectedObject(
        label: label,
        confidence: det['confidence'] as double,
        boundingBox: det['box'] as Rect,
        classIndex: det['classIndex'] as int,
      );
    }).toList();
  }

  List<List<List<List<double>>>> _preprocessImage(img.Image image) {
    final resized = img.copyResize(
      image,
      width: INPUT_SIZE,
      height: INPUT_SIZE,
    );

    return [
      List.generate(
        INPUT_SIZE,
        (y) => List.generate(
          INPUT_SIZE,
          (x) {
            final pixel = resized.getPixel(x, y);
            return [
              pixel.r / 255.0,
              pixel.g / 255.0,
              pixel.b / 255.0,
            ];
          },
        ),
      ),
    ];
  }

  List<Map<String, dynamic>> _processYoloOutput(
    List<List<double>> output,
  ) {
    final detections = <Map<String, dynamic>>[];

    for (final detection in output) {
      final confidence = detection[4];
      if (confidence < CONFIDENCE_THRESHOLD) continue;

      // ค้นหา class ที่มีค่าสูงสุด
      var maxClassProb = 0.0;
      var classIndex = 0;
      for (var i = 5; i < detection.length; i++) {
        final prob = detection[i] * confidence;
        if (prob > maxClassProb) {
          maxClassProb = prob;
          classIndex = i - 5;
        }
      }

      if (maxClassProb < CONFIDENCE_THRESHOLD) continue;

      // แปลง xywh เป็น xyxy (normalized)
      final cx = detection[0] / INPUT_SIZE;
      final cy = detection[1] / INPUT_SIZE;
      final w = detection[2] / INPUT_SIZE;
      final h = detection[3] / INPUT_SIZE;

      final x1 = cx - w / 2;
      final y1 = cy - h / 2;
      final x2 = cx + w / 2;
      final y2 = cy + h / 2;

      detections.add({
        'confidence': maxClassProb,
        'classIndex': classIndex,
        'box': Rect.fromLTRB(x1, y1, x2, y2),
      });
    }

    return detections;
  }

  /// Non-Maximum Suppression
  List<Map<String, dynamic>> _nonMaxSuppression(
    List<Map<String, dynamic>> detections,
  ) {
    if (detections.isEmpty) return [];

    // Sort by confidence
    detections.sort((a, b) =>
        (b['confidence'] as double).compareTo(a['confidence'] as double));

    final keep = <Map<String, dynamic>>[];

    while (detections.isNotEmpty) {
      final best = detections.removeAt(0);
      keep.add(best);

      detections.removeWhere((det) {
        return _iou(
              det['box'] as Rect,
              best['box'] as Rect,
            ) >
            IOU_THRESHOLD;
      });
    }

    return keep;
  }

  double _iou(Rect box1, Rect box2) {
    final intersectLeft = box1.left > box2.left ? box1.left : box2.left;
    final intersectTop = box1.top > box2.top ? box1.top : box2.top;
    final intersectRight =
        box1.right < box2.right ? box1.right : box2.right;
    final intersectBottom =
        box1.bottom < box2.bottom ? box1.bottom : box2.bottom;

    if (intersectRight <= intersectLeft || intersectBottom <= intersectTop) {
      return 0.0;
    }

    final intersectArea =
        (intersectRight - intersectLeft) * (intersectBottom - intersectTop);
    final box1Area = box1.width * box1.height;
    final box2Area = box2.width * box2.height;

    return intersectArea / (box1Area + box2Area - intersectArea);
  }

  void dispose() {
    _interpreter?.close();
    _interpreter = null;
  }
}
```

---

## ขั้นตอนที่ 2604: Text Classification

```dart
// lib/ml/text_classifier.dart
import 'package:tflite_flutter/tflite_flutter.dart';
import 'package:flutter/services.dart';
import 'dart:convert';

class SentimentResult {
  final String sentiment; // 'positive', 'negative', 'neutral'
  final double confidence;
  final Map<String, double> probabilities;

  const SentimentResult({
    required this.sentiment,
    required this.confidence,
    required this.probabilities,
  });

  @override
  String toString() =>
      'SentimentResult($sentiment: ${(confidence * 100).toStringAsFixed(1)}%)';
}

class TextClassifier {
  static const int MAX_SEQ_LENGTH = 128;
  static const int VOCAB_SIZE = 30522; // BERT vocab size

  Interpreter? _interpreter;
  Map<String, int> _vocab = {};

  bool get isInitialized => _interpreter != null;

  Future<void> initialize() async {
    final service = InterpreterService();
    _interpreter = await service.loadModel('bert_sentiment');

    // โหลด vocabulary
    final vocabJson =
        await rootBundle.loadString('assets/models/vocab.json');
    _vocab = Map<String, int>.from(json.decode(vocabJson));
  }

  Future<SentimentResult> classify(String text) async {
    if (_interpreter == null) {
      throw StateError('TextClassifier not initialized');
    }

    // Tokenize
    final tokens = _tokenize(text);
    final inputIds = _tokensToIds(tokens);
    final attentionMask =
        List<int>.filled(inputIds.length, 1);

    // Pad sequences
    final paddedIds = _padSequence(inputIds, MAX_SEQ_LENGTH, 0);
    final paddedMask =
        _padSequence(attentionMask, MAX_SEQ_LENGTH, 0);
    final tokenTypeIds =
        List<int>.filled(MAX_SEQ_LENGTH, 0);

    // Prepare inputs
    final inputs = [
      [paddedIds],
      [paddedMask],
      [tokenTypeIds],
    ];

    // Output: [1, 3] for 3 classes
    final output = [List<double>.filled(3, 0.0)];

    // Run inference
    _interpreter!.runForMultipleInputs(inputs, {0: output});

    return _processOutput(output[0]);
  }

  List<String> _tokenize(String text) {
    // Simple whitespace tokenizer (ใน production ใช้ BERT tokenizer)
    final cleaned = text.toLowerCase()
        .replaceAll(RegExp(r'[^\w\s]'), ' ')
        .trim();
    return ['[CLS]', ...cleaned.split(RegExp(r'\s+')), '[SEP]'];
  }

  List<int> _tokensToIds(List<String> tokens) {
    return tokens.map((token) => _vocab[token] ?? _vocab['[UNK]'] ?? 100).toList();
  }

  List<int> _padSequence(List<int> sequence, int maxLen, int padValue) {
    if (sequence.length >= maxLen) {
      return sequence.take(maxLen).toList();
    }
    return [...sequence, ...List.filled(maxLen - sequence.length, padValue)];
  }

  SentimentResult _processOutput(List<double> output) {
    // Softmax
    final expValues = output.map((v) => _exp(v)).toList();
    final sumExp = expValues.reduce((a, b) => a + b);
    final probabilities = expValues.map((e) => e / sumExp).toList();

    final labels = ['negative', 'neutral', 'positive'];
    var maxProb = probabilities[0];
    var maxIndex = 0;

    for (var i = 1; i < probabilities.length; i++) {
      if (probabilities[i] > maxProb) {
        maxProb = probabilities[i];
        maxIndex = i;
      }
    }

    return SentimentResult(
      sentiment: labels[maxIndex],
      confidence: maxProb,
      probabilities: {
        for (var i = 0; i < labels.length; i++) labels[i]: probabilities[i],
      },
    );
  }

  double _exp(double x) {
    // Prevent overflow
    final clamped = x.clamp(-88.0, 88.0);
    return clamped >= 0
        ? _expPositive(clamped)
        : 1.0 / _expPositive(-clamped);
  }

  double _expPositive(double x) {
    // Taylor series approximation for positive x
    var result = 1.0;
    var term = 1.0;
    for (var i = 1; i <= 20; i++) {
      term *= x / i;
      result += term;
    }
    return result;
  }

  void dispose() {
    _interpreter?.close();
    _interpreter = null;
  }
}
```

---

## ขั้นตอนที่ 2605: Camera Feed + Real-time ML

```dart
// lib/ml/realtime_detector.dart
import 'dart:async';
import 'package:camera/camera.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'object_detector.dart';

class RealtimeDetector extends StatefulWidget {
  const RealtimeDetector({super.key});

  @override
  State<RealtimeDetector> createState() => _RealtimeDetectorState();
}

class _RealtimeDetectorState extends State<RealtimeDetector> {
  CameraController? _cameraController;
  List<CameraDescription> _cameras = [];
  final YoloDetector _detector = YoloDetector();
  List<DetectedObject> _detections = [];
  bool _isProcessing = false;
  int _frameCount = 0;
  double _fps = 0;
  final _stopwatch = Stopwatch();

  @override
  void initState() {
    super.initState();
    _initializeCamera();
    _initializeDetector();
  }

  Future<void> _initializeCamera() async {
    _cameras = await availableCameras();
    if (_cameras.isEmpty) return;

    _cameraController = CameraController(
      _cameras[0],
      ResolutionPreset.medium,
      enableAudio: false,
      imageFormatGroup: ImageFormatGroup.yuv420,
    );

    await _cameraController!.initialize();

    _cameraController!.startImageStream(_processFrame);

    if (mounted) setState(() {});
  }

  Future<void> _initializeDetector() async {
    await _detector.initialize();
  }

  void _processFrame(CameraImage cameraImage) async {
    if (_isProcessing) return;

    _frameCount++;
    if (_frameCount % 3 != 0) return; // Process every 3rd frame

    _isProcessing = true;

    try {
      final imageBytes = _convertCameraImageToBytes(cameraImage);
      final detections = await _detector.detect(imageBytes);

      // Calculate FPS
      if (!_stopwatch.isRunning) {
        _stopwatch.start();
      } else if (_stopwatch.elapsedMilliseconds > 1000) {
        _fps = _frameCount / (_stopwatch.elapsedMilliseconds / 1000);
        _frameCount = 0;
        _stopwatch.reset();
        _stopwatch.start();
      }

      if (mounted) {
        setState(() {
          _detections = detections;
        });
      }
    } catch (e) {
      debugPrint('Detection error: $e');
    } finally {
      _isProcessing = false;
    }
  }

  Uint8List _convertCameraImageToBytes(CameraImage image) {
    // Convert YUV420 to RGB bytes
    final y = image.planes[0].bytes;
    final u = image.planes[1].bytes;
    final v = image.planes[2].bytes;

    final width = image.width;
    final height = image.height;
    final rgbBytes = Uint8List(width * height * 3);

    final uvRowStride = image.planes[1].bytesPerRow;
    final uvPixelStride = image.planes[1].bytesPerPixel!;

    for (var h = 0; h < height; h++) {
      for (var w = 0; w < width; w++) {
        final uvIndex =
            uvPixelStride * (w ~/ 2) + uvRowStride * (h ~/ 2);
        final index = h * width + w;

        final yValue = y[index];
        final uValue = u[uvIndex];
        final vValue = v[uvIndex];

        // YUV to RGB conversion
        final r = (yValue + 1.402 * (vValue - 128)).clamp(0, 255).toInt();
        final g = (yValue - 0.344136 * (uValue - 128) - 0.714136 * (vValue - 128))
            .clamp(0, 255)
            .toInt();
        final b = (yValue + 1.772 * (uValue - 128)).clamp(0, 255).toInt();

        final rgbIndex = index * 3;
        rgbBytes[rgbIndex] = r;
        rgbBytes[rgbIndex + 1] = g;
        rgbBytes[rgbIndex + 2] = b;
      }
    }

    return rgbBytes;
  }

  @override
  Widget build(BuildContext context) {
    if (_cameraController == null ||
        !_cameraController!.value.isInitialized) {
      return const Scaffold(
        body: Center(child: CircularProgressIndicator()),
      );
    }

    return Scaffold(
      appBar: AppBar(
        title: Text('Real-time Detection (${_fps.toStringAsFixed(1)} FPS)'),
      ),
      body: Stack(
        children: [
          // Camera preview
          SizedBox(
            width: double.infinity,
            height: double.infinity,
            child: CameraPreview(_cameraController!),
          ),

          // Detection overlays
          CustomPaint(
            size: Size.infinite,
            painter: DetectionPainter(
              detections: _detections,
              imageSize: Size(
                _cameraController!.value.previewSize!.height,
                _cameraController!.value.previewSize!.width,
              ),
            ),
          ),

          // Detection list
          Positioned(
            bottom: 0,
            left: 0,
            right: 0,
            child: Container(
              color: Colors.black54,
              padding: const EdgeInsets.all(8),
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  if (_detections.isEmpty)
                    const Text(
                      'ไม่พบวัตถุ',
                      style: TextStyle(color: Colors.white),
                    )
                  else
                    ..._detections.take(3).map(
                          (d) => Text(
                            '${d.label}: ${(d.confidence * 100).toStringAsFixed(1)}%',
                            style: const TextStyle(color: Colors.white),
                          ),
                        ),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }

  @override
  void dispose() {
    _cameraController?.dispose();
    _detector.dispose();
    super.dispose();
  }
}

/// Painter สำหรับวาด bounding boxes
class DetectionPainter extends CustomPainter {
  final List<DetectedObject> detections;
  final Size imageSize;

  static const Map<int, Color> _classColors = {
    0: Colors.red,
    1: Colors.blue,
    2: Colors.green,
    3: Colors.yellow,
    4: Colors.purple,
  };

  DetectionPainter({
    required this.detections,
    required this.imageSize,
  });

  @override
  void paint(Canvas canvas, Size size) {
    final scaleX = size.width / imageSize.width;
    final scaleY = size.height / imageSize.height;

    for (final detection in detections) {
      final color =
          _classColors[detection.classIndex % _classColors.length] ??
              Colors.orange;

      final boxPaint = Paint()
        ..color = color.withOpacity(0.3)
        ..style = PaintingStyle.fill;

      final borderPaint = Paint()
        ..color = color
        ..style = PaintingStyle.stroke
        ..strokeWidth = 2.0;

      final rect = Rect.fromLTRB(
        detection.boundingBox.left * size.width,
        detection.boundingBox.top * size.height,
        detection.boundingBox.right * size.width,
        detection.boundingBox.bottom * size.height,
      );

      // วาด box
      canvas.drawRect(rect, boxPaint);
      canvas.drawRect(rect, borderPaint);

      // วาด label
      final textSpan = TextSpan(
        text:
            '${detection.label} ${(detection.confidence * 100).toStringAsFixed(0)}%',
        style: TextStyle(
          color: Colors.white,
          backgroundColor: color,
          fontSize: 12,
        ),
      );

      final textPainter = TextPainter(
        text: textSpan,
        textDirection: TextDirection.ltr,
      )..layout();

      textPainter.paint(
        canvas,
        Offset(rect.left, rect.top - textPainter.height),
      );
    }
  }

  @override
  bool shouldRepaint(DetectionPainter oldDelegate) =>
      detections != oldDelegate.detections;
}
```

---

## ขั้นตอนที่ 2606: On-Device Inference Optimization

```dart
// lib/ml/inference_optimizer.dart
import 'dart:async';
import 'dart:isolate';
import 'dart:typed_data';
import 'package:tflite_flutter/tflite_flutter.dart';

/// Queue สำหรับจัดคิว inference requests
class InferenceQueue {
  final int maxConcurrent;
  int _running = 0;
  final _queue = <_InferenceTask>[];

  InferenceQueue({this.maxConcurrent = 1});

  Future<T> enqueue<T>(Future<T> Function() task) {
    final completer = Completer<T>();
    _queue.add(_InferenceTask<T>(task, completer));
    _processQueue();
    return completer.future;
  }

  void _processQueue() {
    while (_running < maxConcurrent && _queue.isNotEmpty) {
      final task = _queue.removeAt(0);
      _running++;
      task.run().then((_) {
        _running--;
        _processQueue();
      });
    }
  }
}

class _InferenceTask<T> {
  final Future<T> Function() task;
  final Completer<T> completer;

  _InferenceTask(this.task, this.completer);

  Future<void> run() async {
    try {
      final result = await task();
      completer.complete(result);
    } catch (e) {
      completer.completeError(e);
    }
  }
}

/// Cache สำหรับ inference results
class InferenceCache<K, V> {
  final int maxSize;
  final Duration ttl;
  final _cache = <K, _CacheEntry<V>>{};
  final _accessOrder = <K>[];

  InferenceCache({
    this.maxSize = 100,
    this.ttl = const Duration(minutes: 5),
  });

  V? get(K key) {
    final entry = _cache[key];
    if (entry == null) return null;

    if (DateTime.now().difference(entry.createdAt) > ttl) {
      _cache.remove(key);
      _accessOrder.remove(key);
      return null;
    }

    // LRU: move to end
    _accessOrder.remove(key);
    _accessOrder.add(key);

    return entry.value;
  }

  void put(K key, V value) {
    if (_cache.containsKey(key)) {
      _accessOrder.remove(key);
    } else if (_cache.length >= maxSize) {
      // Evict least recently used
      final lruKey = _accessOrder.removeAt(0);
      _cache.remove(lruKey);
    }

    _cache[key] = _CacheEntry(value);
    _accessOrder.add(key);
  }

  void clear() {
    _cache.clear();
    _accessOrder.clear();
  }

  int get size => _cache.length;
}

class _CacheEntry<V> {
  final V value;
  final DateTime createdAt;

  _CacheEntry(this.value) : createdAt = DateTime.now();
}

/// Batching inference requests
class BatchInferenceProcessor {
  final Interpreter interpreter;
  final int batchSize;
  final Duration maxWait;

  final _pending = <_BatchItem>[];
  Timer? _timer;

  BatchInferenceProcessor({
    required this.interpreter,
    this.batchSize = 8,
    this.maxWait = const Duration(milliseconds: 50),
  });

  Future<List<double>> process(List<double> input) {
    final completer = Completer<List<double>>();
    _pending.add(_BatchItem(input, completer));

    if (_pending.length >= batchSize) {
      _processBatch();
    } else {
      _timer ??= Timer(maxWait, _processBatch);
    }

    return completer.future;
  }

  void _processBatch() {
    _timer?.cancel();
    _timer = null;

    if (_pending.isEmpty) return;

    final batch = List<_BatchItem>.from(_pending);
    _pending.clear();

    // Stack inputs into a batch
    final batchInput = batch.map((item) => item.input).toList();

    // Run batched inference
    final outputs = _runBatchedInference(batchInput);

    // Distribute results
    for (var i = 0; i < batch.length; i++) {
      batch[i].completer.complete(outputs[i]);
    }
  }

  List<List<double>> _runBatchedInference(List<List<double>> inputs) {
    // Implementation depends on model structure
    // This is a simplified example
    final results = <List<double>>[];
    for (final input in inputs) {
      final output = [List<double>.filled(10, 0.0)];
      interpreter.run([input], output);
      results.add(output[0]);
    }
    return results;
  }

  void dispose() {
    _timer?.cancel();
    for (final item in _pending) {
      item.completer.completeError(
        StateError('BatchInferenceProcessor disposed'),
      );
    }
    _pending.clear();
  }
}

class _BatchItem {
  final List<double> input;
  final Completer<List<double>> completer;

  _BatchItem(this.input, this.completer);
}

/// Profiler สำหรับวัด inference performance
class InferenceProfiler {
  final _timings = <String, List<int>>{};

  void record(String name, int microseconds) {
    _timings.putIfAbsent(name, () => []).add(microseconds);
  }

  Map<String, Map<String, double>> getStats() {
    return {
      for (final entry in _timings.entries)
        entry.key: {
          'count': entry.value.length.toDouble(),
          'mean': _mean(entry.value),
          'min': entry.value.reduce((a, b) => a < b ? a : b).toDouble(),
          'max': entry.value.reduce((a, b) => a > b ? a : b).toDouble(),
          'p95': _percentile(entry.value, 0.95),
        },
    };
  }

  double _mean(List<int> values) {
    return values.isEmpty
        ? 0
        : values.reduce((a, b) => a + b) / values.length;
  }

  double _percentile(List<int> values, double p) {
    if (values.isEmpty) return 0;
    final sorted = List<int>.from(values)..sort();
    final index = (p * (sorted.length - 1)).round();
    return sorted[index].toDouble();
  }

  void reset() => _timings.clear();

  void printReport() {
    print('=== Inference Performance Report ===');
    for (final entry in getStats().entries) {
      final stats = entry.value;
      print('${entry.key}:');
      print('  Count: ${stats['count']!.toInt()}');
      print('  Mean: ${stats['mean']!.toStringAsFixed(2)}µs');
      print('  Min: ${stats['min']!.toInt()}µs');
      print('  Max: ${stats['max']!.toInt()}µs');
      print('  P95: ${stats['p95']!.toStringAsFixed(2)}µs');
    }
  }
}
```

---

## ขั้นตอนที่ 2607: ML Flutter App ที่สมบูรณ์

```dart
// lib/main_ml.dart
import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';
import 'dart:io';
import 'dart:typed_data';
import 'ml/image_classifier.dart';
import 'ml/text_classifier.dart';

void main() => runApp(const MlDemoApp());

class MlDemoApp extends StatelessWidget {
  const MlDemoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'ML Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const MlDemoPage(),
    );
  }
}

class MlDemoPage extends StatefulWidget {
  const MlDemoPage({super.key});

  @override
  State<MlDemoPage> createState() => _MlDemoPageState();
}

class _MlDemoPageState extends State<MlDemoPage>
    with SingleTickerProviderStateMixin {
  late TabController _tabController;
  final _imagePicker = ImagePicker();
  final _imageClassifier = MobileNetClassifier();
  final _textClassifier = TextClassifier();

  bool _classifierReady = false;
  bool _textClassifierReady = false;
  File? _selectedImage;
  List<ClassificationResult> _classifications = [];
  SentimentResult? _sentimentResult;
  bool _isProcessing = false;
  final _textController = TextEditingController();

  @override
  void initState() {
    super.initState();
    _tabController = TabController(length: 2, vsync: this);
    _initModels();
  }

  Future<void> _initModels() async {
    try {
      await Future.wait([
        _imageClassifier.initialize(),
        _textClassifier.initialize(),
      ]);
      setState(() {
        _classifierReady = true;
        _textClassifierReady = true;
      });
    } catch (e) {
      debugPrint('Error initializing models: $e');
      // Demo mode - show UI without actual models
      setState(() {
        _classifierReady = true;
        _textClassifierReady = true;
      });
    }
  }

  Future<void> _pickAndClassify() async {
    final picked = await _imagePicker.pickImage(source: ImageSource.gallery);
    if (picked == null) return;

    setState(() {
      _selectedImage = File(picked.path);
      _isProcessing = true;
    });

    try {
      final bytes = await _selectedImage!.readAsBytes();
      final results = await _imageClassifier.classify(bytes, topK: 5);
      setState(() => _classifications = results);
    } catch (e) {
      debugPrint('Classification error: $e');
      // Demo results
      setState(() {
        _classifications = [
          const ClassificationResult(
              label: 'cat', confidence: 0.92, index: 281),
          const ClassificationResult(
              label: 'kitten', confidence: 0.05, index: 282),
        ];
      });
    } finally {
      setState(() => _isProcessing = false);
    }
  }

  Future<void> _analyzeSentiment() async {
    final text = _textController.text.trim();
    if (text.isEmpty) return;

    setState(() => _isProcessing = true);

    try {
      final result = await _textClassifier.classify(text);
      setState(() => _sentimentResult = result);
    } catch (e) {
      debugPrint('Sentiment error: $e');
    } finally {
      setState(() => _isProcessing = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('ML Demo'),
        bottom: TabBar(
          controller: _tabController,
          tabs: const [
            Tab(icon: Icon(Icons.image), text: 'Image'),
            Tab(icon: Icon(Icons.text_fields), text: 'Text'),
          ],
        ),
      ),
      body: TabBarView(
        controller: _tabController,
        children: [
          _buildImageTab(),
          _buildTextTab(),
        ],
      ),
    );
  }

  Widget _buildImageTab() {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child: Column(
        children: [
          if (_selectedImage != null)
            Container(
              height: 200,
              width: double.infinity,
              decoration: BoxDecoration(
                border: Border.all(color: Colors.grey),
                borderRadius: BorderRadius.circular(8),
              ),
              child: ClipRRect(
                borderRadius: BorderRadius.circular(8),
                child: Image.file(_selectedImage!, fit: BoxFit.cover),
              ),
            )
          else
            Container(
              height: 200,
              width: double.infinity,
              decoration: BoxDecoration(
                border: Border.all(color: Colors.grey),
                borderRadius: BorderRadius.circular(8),
                color: Colors.grey[100],
              ),
              child: const Icon(Icons.add_photo_alternate,
                  size: 64, color: Colors.grey),
            ),
          const SizedBox(height: 16),
          ElevatedButton.icon(
            onPressed: _classifierReady && !_isProcessing
                ? _pickAndClassify
                : null,
            icon: _isProcessing
                ? const SizedBox(
                    width: 16,
                    height: 16,
                    child: CircularProgressIndicator(strokeWidth: 2),
                  )
                : const Icon(Icons.image_search),
            label: Text(_isProcessing ? 'Processing...' : 'Select & Classify'),
          ),
          const SizedBox(height: 24),
          if (_classifications.isNotEmpty) ...[
            const Text(
              'Results',
              style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            ..._classifications.map((result) => Card(
                  child: ListTile(
                    title: Text(result.label),
                    trailing: Text(
                      '${(result.confidence * 100).toStringAsFixed(1)}%',
                      style: const TextStyle(
                        fontWeight: FontWeight.bold,
                        color: Colors.blue,
                      ),
                    ),
                    subtitle: LinearProgressIndicator(
                      value: result.confidence,
                      backgroundColor: Colors.grey[200],
                    ),
                  ),
                )),
          ],
        ],
      ),
    );
  }

  Widget _buildTextTab() {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.stretch,
        children: [
          TextField(
            controller: _textController,
            maxLines: 4,
            decoration: const InputDecoration(
              border: OutlineInputBorder(),
              labelText: 'Enter text to analyze',
              hintText: 'Type something here...',
            ),
          ),
          const SizedBox(height: 16),
          ElevatedButton.icon(
            onPressed: _textClassifierReady && !_isProcessing
                ? _analyzeSentiment
                : null,
            icon: _isProcessing
                ? const SizedBox(
                    width: 16,
                    height: 16,
                    child: CircularProgressIndicator(strokeWidth: 2),
                  )
                : const Icon(Icons.psychology),
            label: Text(_isProcessing ? 'Analyzing...' : 'Analyze Sentiment'),
          ),
          const SizedBox(height: 24),
          if (_sentimentResult != null) ...[
            Card(
              color: _getSentimentColor(_sentimentResult!.sentiment),
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Column(
                  children: [
                    Icon(
                      _getSentimentIcon(_sentimentResult!.sentiment),
                      size: 48,
                      color: Colors.white,
                    ),
                    const SizedBox(height: 8),
                    Text(
                      _sentimentResult!.sentiment.toUpperCase(),
                      style: const TextStyle(
                        color: Colors.white,
                        fontSize: 24,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                    Text(
                      '${(_sentimentResult!.confidence * 100).toStringAsFixed(1)}% confident',
                      style: const TextStyle(color: Colors.white70),
                    ),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 16),
            ..._sentimentResult!.probabilities.entries.map(
              (entry) => Padding(
                padding: const EdgeInsets.symmetric(vertical: 4),
                child: Row(
                  children: [
                    SizedBox(
                      width: 80,
                      child: Text(entry.key),
                    ),
                    Expanded(
                      child: LinearProgressIndicator(
                        value: entry.value,
                        backgroundColor: Colors.grey[200],
                      ),
                    ),
                    const SizedBox(width: 8),
                    Text('${(entry.value * 100).toStringAsFixed(1)}%'),
                  ],
                ),
              ),
            ),
          ],
        ],
      ),
    );
  }

  Color _getSentimentColor(String sentiment) {
    switch (sentiment) {
      case 'positive':
        return Colors.green[600]!;
      case 'negative':
        return Colors.red[600]!;
      default:
        return Colors.grey[600]!;
    }
  }

  IconData _getSentimentIcon(String sentiment) {
    switch (sentiment) {
      case 'positive':
        return Icons.sentiment_very_satisfied;
      case 'negative':
        return Icons.sentiment_very_dissatisfied;
      default:
        return Icons.sentiment_neutral;
    }
  }

  @override
  void dispose() {
    _tabController.dispose();
    _textController.dispose();
    _imageClassifier.dispose();
    _textClassifier.dispose();
    super.dispose();
  }
}
```

---

**← [Part 67](part-67-dart-ffi.md)**
**ต่อไป: [Part 69 →](part-69-dart-isolate-advanced.md)**
