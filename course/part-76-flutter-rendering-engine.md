# Part 76: Flutter Rendering Engine
## ขั้นตอนที่ 2921-2960

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ RenderObject และ RenderBox lifecycle
- สร้าง Custom Layout ด้วย MultiChildRenderObjectWidget
- วาด Custom UI ด้วย RenderCustomPaint
- เข้าใจ Layer System และ Compositing
- ใช้ RepaintBoundary และ optimize rendering performance

---

## ขั้นตอนที่ 2921: RenderObject Fundamentals

```dart
// lib/rendering/render_object_basics.dart
import 'package:flutter/rendering.dart';
import 'package:flutter/widgets.dart';

/// RenderObject คือ low-level building block ของ Flutter rendering pipeline
/// ทุก widget ที่แสดงบนหน้าจอจะมี RenderObject อยู่เบื้องหลัง

// 1. Simple RenderBox ที่วาดสีพื้นหลัง
class RenderColoredBox extends RenderBox {
  RenderColoredBox({required Color color}) : _color = color;

  Color _color;
  Color get color => _color;
  set color(Color value) {
    if (_color == value) return;
    _color = value;
    markNeedsPaint(); // บอกให้ Flutter repaint
  }

  @override
  void performLayout() {
    // กำหนดขนาดของตัวเอง
    size = constraints.biggest; // ใช้พื้นที่ทั้งหมดที่ได้รับ
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    final paint = Paint()..color = _color;
    context.canvas.drawRect(offset & size, paint);
  }

  @override
  bool hitTestSelf(Offset position) => size.contains(position);
}

// 2. Widget wrapper สำหรับ RenderColoredBox
class ColoredBoxWidget extends LeafRenderObjectWidget {
  const ColoredBoxWidget({
    super.key,
    required this.color,
  });

  final Color color;

  @override
  RenderColoredBox createRenderObject(BuildContext context) {
    return RenderColoredBox(color: color);
  }

  @override
  void updateRenderObject(BuildContext context, RenderColoredBox renderObject) {
    renderObject.color = color;
  }
}

// 3. RenderBox พร้อม Child
class RenderPaddedBox extends RenderBox
    with RenderObjectWithChildMixin<RenderBox> {
  RenderPaddedBox({
    required EdgeInsets padding,
    RenderBox? child,
  }) : _padding = padding {
    this.child = child;
  }

  EdgeInsets _padding;
  EdgeInsets get padding => _padding;
  set padding(EdgeInsets value) {
    if (_padding == value) return;
    _padding = value;
    markNeedsLayout();
  }

  @override
  void performLayout() {
    final child = this.child;
    if (child == null) {
      size = constraints.smallest;
      return;
    }

    // คำนวณ constraints สำหรับ child หลังหัก padding
    final innerConstraints = constraints.deflate(_padding);
    child.layout(innerConstraints, parentUsesSize: true);

    // กำหนดขนาดตัวเองตาม child + padding
    size = constraints.constrain(
      Size(
        child.size.width + _padding.horizontal,
        child.size.height + _padding.vertical,
      ),
    );

    // กำหนดตำแหน่งของ child
    final childParentData = child.parentData as BoxParentData;
    childParentData.offset = Offset(_padding.left, _padding.top);
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    final child = this.child;
    if (child == null) return;

    final childParentData = child.parentData as BoxParentData;
    context.paintChild(child, offset + childParentData.offset);
  }

  @override
  void setupParentData(RenderObject child) {
    if (child.parentData is! BoxParentData) {
      child.parentData = BoxParentData();
    }
  }

  @override
  bool hitTestChildren(BoxHitTestResult result, {required Offset position}) {
    final child = this.child;
    if (child == null) return false;

    final childParentData = child.parentData as BoxParentData;
    return result.addWithPaintOffset(
      offset: childParentData.offset,
      position: position,
      hitTest: (BoxHitTestResult result, Offset transformed) {
        return child.hitTest(result, position: transformed);
      },
    );
  }
}
```

---

## ขั้นตอนที่ 2922: Custom Layout Widget

```dart
// lib/rendering/custom_layout.dart
import 'package:flutter/rendering.dart';
import 'package:flutter/widgets.dart';

/// Custom ParentData สำหรับเก็บ position ของแต่ละ child
class CircularLayoutParentData extends ContainerBoxParentData<RenderBox> {
  double? angle; // มุมใน radians
}

/// RenderObject สำหรับจัดเรียง children เป็นวงกลม
class RenderCircularLayout extends RenderBox
    with
        ContainerRenderObjectMixin<RenderBox, CircularLayoutParentData>,
        RenderBoxContainerDefaultsMixin<RenderBox, CircularLayoutParentData> {
  RenderCircularLayout({
    double radius = 100.0,
    List<RenderBox>? children,
  }) : _radius = radius {
    addAll(children);
  }

  double _radius;
  double get radius => _radius;
  set radius(double value) {
    if (_radius == value) return;
    _radius = value;
    markNeedsLayout();
  }

  @override
  void setupParentData(RenderObject child) {
    if (child.parentData is! CircularLayoutParentData) {
      child.parentData = CircularLayoutParentData();
    }
  }

  @override
  void performLayout() {
    // กำหนดขนาดของ layout
    final diameter = _radius * 2;
    size = constraints.constrain(
      Size(diameter + 80, diameter + 80), // เพิ่มพื้นที่สำหรับ children
    );

    final center = Offset(size.width / 2, size.height / 2);
    final childConstraints = BoxConstraints.loose(const Size(60, 60));

    // จัดเรียง children เป็นวงกลม
    int childCount = 0;
    RenderBox? child = firstChild;
    while (child != null) {
      childCount++;
      child = childAfter(child);
    }

    if (childCount == 0) return;

    final angleStep = (2 * 3.14159265) / childCount;
    int index = 0;
    child = firstChild;

    while (child != null) {
      child.layout(childConstraints, parentUsesSize: true);

      final angle = angleStep * index - 3.14159265 / 2;
      final x = center.dx + _radius * _cos(angle) - child.size.width / 2;
      final y = center.dy + _radius * _sin(angle) - child.size.height / 2;

      final childParentData = child.parentData as CircularLayoutParentData;
      childParentData.offset = Offset(x, y);
      childParentData.angle = angle;

      index++;
      child = childAfter(child);
    }
  }

  double _cos(double angle) => _mathCos(angle);
  double _sin(double angle) => _mathSin(angle);

  // Simple cos/sin approximation (ใช้ dart:math ในโปรเจกต์จริง)
  double _mathCos(double x) {
    // ใช้ Taylor series approximation
    x = x % (2 * 3.14159265);
    double result = 1;
    double term = 1;
    for (int i = 1; i <= 8; i++) {
      term *= -x * x / ((2 * i - 1) * (2 * i));
      result += term;
    }
    return result;
  }

  double _mathSin(double x) {
    x = x % (2 * 3.14159265);
    double result = x;
    double term = x;
    for (int i = 1; i <= 8; i++) {
      term *= -x * x / ((2 * i) * (2 * i + 1));
      result += term;
    }
    return result;
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    defaultPaint(context, offset);
  }

  @override
  bool hitTestChildren(BoxHitTestResult result, {required Offset position}) {
    return defaultHitTestChildren(result, position: position);
  }
}

/// Widget wrapper สำหรับ RenderCircularLayout
class CircularLayout extends MultiChildRenderObjectWidget {
  const CircularLayout({
    super.key,
    required super.children,
    this.radius = 100.0,
  });

  final double radius;

  @override
  RenderCircularLayout createRenderObject(BuildContext context) {
    return RenderCircularLayout(radius: radius);
  }

  @override
  void updateRenderObject(
      BuildContext context, RenderCircularLayout renderObject) {
    renderObject.radius = radius;
  }
}

// Demo App ใช้งาน CircularLayout
class CircularLayoutDemo extends StatelessWidget {
  const CircularLayoutDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Circular Layout Demo')),
      body: Center(
        child: CircularLayout(
          radius: 120,
          children: List.generate(6, (index) {
            final colors = [
              const Color(0xFFE53935),
              const Color(0xFF8E24AA),
              const Color(0xFF1E88E5),
              const Color(0xFF00ACC1),
              const Color(0xFF43A047),
              const Color(0xFFFB8C00),
            ];
            return Container(
              width: 50,
              height: 50,
              decoration: BoxDecoration(
                color: colors[index],
                shape: BoxShape.circle,
              ),
              child: Center(
                child: Text(
                  '${index + 1}',
                  style: const TextStyle(
                    color: Colors.white,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ),
            );
          }),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2923: Custom Painting ด้วย CustomPainter

```dart
// lib/rendering/custom_painter_advanced.dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

/// CustomPainter สำหรับวาด Animated Wave
class WavePainter extends CustomPainter {
  WavePainter({
    required this.animation,
    required this.color,
    this.waveCount = 3,
    this.amplitude = 20.0,
  }) : super(repaint: animation);

  final Animation<double> animation;
  final Color color;
  final int waveCount;
  final double amplitude;

  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = color
      ..style = PaintingStyle.fill;

    final path = Path();
    final progress = animation.value;

    path.moveTo(0, size.height);

    // วาด wave path
    for (double x = 0; x <= size.width; x++) {
      final y = size.height * 0.5 +
          amplitude *
              math.sin(
                (x / size.width) * waveCount * 2 * math.pi +
                    progress * 2 * math.pi,
              );
      path.lineTo(x, y);
    }

    path.lineTo(size.width, size.height);
    path.close();

    canvas.drawPath(path, paint);
  }

  @override
  bool shouldRepaint(WavePainter oldDelegate) {
    return oldDelegate.color != color ||
        oldDelegate.waveCount != waveCount ||
        oldDelegate.amplitude != amplitude;
  }
}

/// CustomPainter สำหรับวาด Radar Chart
class RadarChartPainter extends CustomPainter {
  RadarChartPainter({
    required this.data,
    required this.labels,
    required this.fillColor,
    required this.strokeColor,
    this.gridColor = const Color(0xFFBBBBBB),
    this.gridLevels = 5,
  });

  final List<double> data; // ค่า 0.0 - 1.0
  final List<String> labels;
  final Color fillColor;
  final Color strokeColor;
  final Color gridColor;
  final int gridLevels;

  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final radius = math.min(size.width, size.height) / 2 - 30;
    final sides = data.length;
    final angleStep = (2 * math.pi) / sides;

    // วาด grid
    _drawGrid(canvas, center, radius, sides, angleStep);

    // วาด data
    _drawData(canvas, center, radius, sides, angleStep);

    // วาด labels
    _drawLabels(canvas, center, radius, sides, angleStep);
  }

  void _drawGrid(Canvas canvas, Offset center, double radius, int sides,
      double angleStep) {
    final gridPaint = Paint()
      ..color = gridColor
      ..style = PaintingStyle.stroke
      ..strokeWidth = 1.0;

    // วาด grid lines จาก center ไปยัง vertices
    for (int i = 0; i < sides; i++) {
      final angle = angleStep * i - math.pi / 2;
      final x = center.dx + radius * math.cos(angle);
      final y = center.dy + radius * math.sin(angle);
      canvas.drawLine(center, Offset(x, y), gridPaint);
    }

    // วาด concentric polygons
    for (int level = 1; level <= gridLevels; level++) {
      final levelRadius = radius * level / gridLevels;
      final path = Path();

      for (int i = 0; i < sides; i++) {
        final angle = angleStep * i - math.pi / 2;
        final x = center.dx + levelRadius * math.cos(angle);
        final y = center.dy + levelRadius * math.sin(angle);

        if (i == 0) {
          path.moveTo(x, y);
        } else {
          path.lineTo(x, y);
        }
      }
      path.close();
      canvas.drawPath(path, gridPaint);
    }
  }

  void _drawData(Canvas canvas, Offset center, double radius, int sides,
      double angleStep) {
    final fillPaint = Paint()
      ..color = fillColor.withOpacity(0.3)
      ..style = PaintingStyle.fill;

    final strokePaint = Paint()
      ..color = strokeColor
      ..style = PaintingStyle.stroke
      ..strokeWidth = 2.0;

    final path = Path();

    for (int i = 0; i < sides; i++) {
      final angle = angleStep * i - math.pi / 2;
      final value = data[i].clamp(0.0, 1.0);
      final x = center.dx + radius * value * math.cos(angle);
      final y = center.dy + radius * value * math.sin(angle);

      if (i == 0) {
        path.moveTo(x, y);
      } else {
        path.lineTo(x, y);
      }
    }
    path.close();

    canvas.drawPath(path, fillPaint);
    canvas.drawPath(path, strokePaint);

    // วาด data points
    final pointPaint = Paint()
      ..color = strokeColor
      ..style = PaintingStyle.fill;

    for (int i = 0; i < sides; i++) {
      final angle = angleStep * i - math.pi / 2;
      final value = data[i].clamp(0.0, 1.0);
      final x = center.dx + radius * value * math.cos(angle);
      final y = center.dy + radius * value * math.sin(angle);
      canvas.drawCircle(Offset(x, y), 4, pointPaint);
    }
  }

  void _drawLabels(Canvas canvas, Offset center, double radius, int sides,
      double angleStep) {
    for (int i = 0; i < sides; i++) {
      final angle = angleStep * i - math.pi / 2;
      final labelRadius = radius + 20;
      final x = center.dx + labelRadius * math.cos(angle);
      final y = center.dy + labelRadius * math.sin(angle);

      final textPainter = TextPainter(
        text: TextSpan(
          text: labels[i],
          style: const TextStyle(
            color: Color(0xFF333333),
            fontSize: 12,
          ),
        ),
        textDirection: TextDirection.ltr,
      );
      textPainter.layout();
      textPainter.paint(
        canvas,
        Offset(x - textPainter.width / 2, y - textPainter.height / 2),
      );
    }
  }

  @override
  bool shouldRepaint(RadarChartPainter oldDelegate) {
    return oldDelegate.data != data || oldDelegate.fillColor != fillColor;
  }
}

/// Widget ใช้งาน RadarChart
class RadarChartWidget extends StatelessWidget {
  const RadarChartWidget({super.key});

  @override
  Widget build(BuildContext context) {
    return CustomPaint(
      size: const Size(300, 300),
      painter: RadarChartPainter(
        data: [0.8, 0.6, 0.9, 0.7, 0.5, 0.85],
        labels: ['Speed', 'Power', 'Tech', 'Design', 'UX', 'Performance'],
        fillColor: Colors.blue,
        strokeColor: Colors.blue,
      ),
    );
  }
}

/// Animated Wave Widget
class AnimatedWaveWidget extends StatefulWidget {
  const AnimatedWaveWidget({super.key});

  @override
  State<AnimatedWaveWidget> createState() => _AnimatedWaveWidgetState();
}

class _AnimatedWaveWidgetState extends State<AnimatedWaveWidget>
    with TickerProviderStateMixin {
  late AnimationController _controller1;
  late AnimationController _controller2;
  late Animation<double> _animation1;
  late Animation<double> _animation2;

  @override
  void initState() {
    super.initState();
    _controller1 = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 2),
    )..repeat();

    _controller2 = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 1500),
    )..repeat();

    _animation1 = Tween<double>(begin: 0, end: 1).animate(_controller1);
    _animation2 = Tween<double>(begin: 0.5, end: 1.5).animate(_controller2);
  }

  @override
  void dispose() {
    _controller1.dispose();
    _controller2.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      height: 200,
      child: Stack(
        children: [
          CustomPaint(
            size: const Size(double.infinity, 200),
            painter: WavePainter(
              animation: _animation1,
              color: Colors.blue.withOpacity(0.5),
              amplitude: 25,
              waveCount: 2,
            ),
          ),
          CustomPaint(
            size: const Size(double.infinity, 200),
            painter: WavePainter(
              animation: _animation2,
              color: Colors.lightBlue.withOpacity(0.4),
              amplitude: 15,
              waveCount: 3,
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2924: Layer System ใน Flutter

```dart
// lib/rendering/layer_system.dart
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';

/// Layer System ใน Flutter มี 3 ระดับหลัก:
/// 1. PictureLayer - เก็บ drawing commands
/// 2. ContainerLayer - เก็บ child layers
/// 3. CompositorLayer - ส่งไปยัง Skia/Impeller

// 1. RepaintBoundary - สร้าง layer boundary ใหม่
class OptimizedListItem extends StatelessWidget {
  const OptimizedListItem({
    super.key,
    required this.title,
    required this.subtitle,
    required this.onTap,
  });

  final String title;
  final String subtitle;
  final VoidCallback onTap;

  @override
  Widget build(BuildContext context) {
    // RepaintBoundary ป้องกัน repaint เมื่อ parent เปลี่ยน
    return RepaintBoundary(
      child: GestureDetector(
        onTap: onTap,
        child: Container(
          padding: const EdgeInsets.all(16),
          decoration: BoxDecoration(
            border: Border(
              bottom: BorderSide(color: Colors.grey.shade200),
            ),
          ),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                title,
                style: const TextStyle(
                  fontWeight: FontWeight.bold,
                  fontSize: 16,
                ),
              ),
              const SizedBox(height: 4),
              Text(
                subtitle,
                style: TextStyle(color: Colors.grey.shade600),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

// 2. Custom RenderObject พร้อม Layer Management
class RenderLayerDemo extends RenderBox {
  RenderLayerDemo({
    required Color topColor,
    required Color bottomColor,
  })  : _topColor = topColor,
        _bottomColor = bottomColor;

  Color _topColor;
  Color _bottomColor;

  set topColor(Color value) {
    if (_topColor == value) return;
    _topColor = value;
    markNeedsPaint();
  }

  set bottomColor(Color value) {
    if (_bottomColor == value) return;
    _bottomColor = value;
    markNeedsPaint();
  }

  @override
  void performLayout() {
    size = constraints.biggest;
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    final canvas = context.canvas;

    // วาด top half
    canvas.drawRect(
      Rect.fromLTWH(offset.dx, offset.dy, size.width, size.height / 2),
      Paint()..color = _topColor,
    );

    // วาด bottom half
    canvas.drawRect(
      Rect.fromLTWH(
          offset.dx, offset.dy + size.height / 2, size.width, size.height / 2),
      Paint()..color = _bottomColor,
    );

    // วาด overlay text
    final textPainter = TextPainter(
      text: TextSpan(
        text: 'Layer Demo',
        style: const TextStyle(
          color: Colors.white,
          fontSize: 24,
          fontWeight: FontWeight.bold,
        ),
      ),
      textDirection: TextDirection.ltr,
    );
    textPainter.layout();
    textPainter.paint(
      canvas,
      offset +
          Offset(
            (size.width - textPainter.width) / 2,
            (size.height - textPainter.height) / 2,
          ),
    );
  }
}

// 3. Compositing Example - ใช้ opacity layer
class CompositingExample extends StatefulWidget {
  const CompositingExample({super.key});

  @override
  State<CompositingExample> createState() => _CompositingExampleState();
}

class _CompositingExampleState extends State<CompositingExample>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _opacityAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 2),
    )..repeat(reverse: true);

    _opacityAnimation = Tween<double>(begin: 0.2, end: 1.0).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Layer & Compositing Demo')),
      body: Column(
        children: [
          // FadeTransition ใช้ Opacity Layer (GPU compositing)
          FadeTransition(
            opacity: _opacityAnimation,
            child: Container(
              height: 100,
              color: Colors.blue,
              child: const Center(
                child: Text(
                  'GPU Composited Opacity',
                  style: TextStyle(color: Colors.white, fontSize: 18),
                ),
              ),
            ),
          ),
          const SizedBox(height: 20),
          // AnimatedOpacity ใช้ CPU painting (ไม่ใช้ layer)
          AnimatedOpacity(
            opacity: _opacityAnimation.value,
            duration: const Duration(milliseconds: 100),
            child: Container(
              height: 100,
              color: Colors.green,
              child: const Center(
                child: Text(
                  'CPU Painted Opacity',
                  style: TextStyle(color: Colors.white, fontSize: 18),
                ),
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2925: RenderCustomPaint และ Performance

```dart
// lib/rendering/render_custom_paint.dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

/// High-Performance Particle System ด้วย CustomPainter
class Particle {
  Particle({
    required this.position,
    required this.velocity,
    required this.color,
    required this.radius,
    required this.life,
  });

  Offset position;
  Offset velocity;
  Color color;
  double radius;
  double life; // 0.0 - 1.0

  void update(double dt) {
    position += velocity * dt;
    velocity = Offset(velocity.dx * 0.99, velocity.dy + 98 * dt); // gravity
    life -= dt * 0.5;
    radius = radius * (0.99);
  }

  bool get isDead => life <= 0;
}

class ParticleSystemPainter extends CustomPainter {
  ParticleSystemPainter({required this.particles});

  final List<Particle> particles;

  @override
  void paint(Canvas canvas, Size size) {
    for (final particle in particles) {
      if (particle.isDead) continue;

      final paint = Paint()
        ..color = particle.color.withOpacity(particle.life)
        ..style = PaintingStyle.fill;

      canvas.drawCircle(particle.position, particle.radius, paint);
    }
  }

  @override
  bool shouldRepaint(ParticleSystemPainter oldDelegate) => true;
}

class ParticleSystemWidget extends StatefulWidget {
  const ParticleSystemWidget({super.key});

  @override
  State<ParticleSystemWidget> createState() => _ParticleSystemWidgetState();
}

class _ParticleSystemWidgetState extends State<ParticleSystemWidget>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  final List<Particle> _particles = [];
  final math.Random _random = math.Random();
  Offset _emitterPosition = const Offset(150, 300);
  double _lastTime = 0;

  final List<Color> _colors = [
    Colors.red,
    Colors.orange,
    Colors.yellow,
    Colors.green,
    Colors.blue,
    Colors.purple,
  ];

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(hours: 1),
    )..forward();

    _controller.addListener(_update);
  }

  void _update() {
    final currentTime = _controller.value * 3600;
    final dt = currentTime - _lastTime;
    _lastTime = currentTime;

    // Emit new particles
    for (int i = 0; i < 3; i++) {
      _spawnParticle();
    }

    // Update existing particles
    _particles.removeWhere((p) => p.isDead);
    for (final particle in _particles) {
      particle.update(dt * 0.01);
    }

    if (mounted) setState(() {});
  }

  void _spawnParticle() {
    final angle = _random.nextDouble() * math.pi - math.pi / 2;
    final speed = 50 + _random.nextDouble() * 100;

    _particles.add(Particle(
      position: _emitterPosition,
      velocity: Offset(
        math.cos(angle) * speed,
        math.sin(angle) * speed,
      ),
      color: _colors[_random.nextInt(_colors.length)],
      radius: 3 + _random.nextDouble() * 5,
      life: 1.0,
    ));
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      appBar: AppBar(
        backgroundColor: Colors.black,
        title:
            const Text('Particle System', style: TextStyle(color: Colors.white)),
      ),
      body: GestureDetector(
        onPanUpdate: (details) {
          setState(() {
            _emitterPosition = details.localPosition;
          });
        },
        child: CustomPaint(
          painter: ParticleSystemPainter(particles: List.from(_particles)),
          size: Size.infinite,
          child: Center(
            child: Text(
              'Particles: ${_particles.length}',
              style: const TextStyle(color: Colors.white70, fontSize: 12),
            ),
          ),
        ),
      ),
    );
  }
}

/// Clock Painter - ตัวอย่าง CustomPainter ที่ซับซ้อน
class ClockPainter extends CustomPainter {
  ClockPainter({required this.dateTime});

  final DateTime dateTime;

  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final radius = math.min(size.width, size.height) / 2 - 10;

    // วาด clock face
    _drawFace(canvas, center, radius);

    // วาด tick marks
    _drawTicks(canvas, center, radius);

    // วาด numbers
    _drawNumbers(canvas, center, radius);

    // วาด hands
    _drawHourHand(canvas, center, radius);
    _drawMinuteHand(canvas, center, radius);
    _drawSecondHand(canvas, center, radius);

    // วาด center dot
    canvas.drawCircle(center, 8, Paint()..color = Colors.red);
  }

  void _drawFace(Canvas canvas, Offset center, double radius) {
    // Shadow
    canvas.drawCircle(
      center + const Offset(2, 2),
      radius,
      Paint()..color = Colors.black26,
    );

    // Face
    canvas.drawCircle(
      center,
      radius,
      Paint()
        ..color = Colors.white
        ..style = PaintingStyle.fill,
    );

    // Border
    canvas.drawCircle(
      center,
      radius,
      Paint()
        ..color = const Color(0xFF333333)
        ..style = PaintingStyle.stroke
        ..strokeWidth = 3,
    );
  }

  void _drawTicks(Canvas canvas, Offset center, double radius) {
    final paint = Paint()..color = const Color(0xFF333333);

    for (int i = 0; i < 60; i++) {
      final angle = (i * 6 - 90) * math.pi / 180;
      final isHour = i % 5 == 0;

      final inner = isHour ? radius * 0.85 : radius * 0.92;
      final outer = radius * 0.98;

      canvas.drawLine(
        Offset(
          center.dx + inner * math.cos(angle),
          center.dy + inner * math.sin(angle),
        ),
        Offset(
          center.dx + outer * math.cos(angle),
          center.dy + outer * math.sin(angle),
        ),
        paint
          ..strokeWidth = isHour ? 2.5 : 1.0
          ..strokeCap = StrokeCap.round,
      );
    }
  }

  void _drawNumbers(Canvas canvas, Offset center, double radius) {
    for (int i = 1; i <= 12; i++) {
      final angle = (i * 30 - 90) * math.pi / 180;
      final x = center.dx + radius * 0.75 * math.cos(angle);
      final y = center.dy + radius * 0.75 * math.sin(angle);

      final textPainter = TextPainter(
        text: TextSpan(
          text: '$i',
          style: const TextStyle(
            color: Color(0xFF333333),
            fontSize: 14,
            fontWeight: FontWeight.bold,
          ),
        ),
        textDirection: TextDirection.ltr,
      );
      textPainter.layout();
      textPainter.paint(
        canvas,
        Offset(x - textPainter.width / 2, y - textPainter.height / 2),
      );
    }
  }

  void _drawHourHand(Canvas canvas, Offset center, double radius) {
    final hours = dateTime.hour % 12;
    final minutes = dateTime.minute;
    final angle =
        ((hours * 30 + minutes * 0.5 - 90) * math.pi / 180);

    _drawHand(
      canvas,
      center,
      angle,
      radius * 0.55,
      6,
      const Color(0xFF333333),
    );
  }

  void _drawMinuteHand(Canvas canvas, Offset center, double radius) {
    final minutes = dateTime.minute;
    final seconds = dateTime.second;
    final angle = ((minutes * 6 + seconds * 0.1 - 90) * math.pi / 180);

    _drawHand(
      canvas,
      center,
      angle,
      radius * 0.75,
      4,
      const Color(0xFF555555),
    );
  }

  void _drawSecondHand(Canvas canvas, Offset center, double radius) {
    final seconds = dateTime.second;
    final angle = (seconds * 6 - 90) * math.pi / 180;

    _drawHand(canvas, center, angle, radius * 0.85, 2, Colors.red);
  }

  void _drawHand(Canvas canvas, Offset center, double angle, double length,
      double width, Color color) {
    canvas.drawLine(
      center,
      Offset(
        center.dx + length * math.cos(angle),
        center.dy + length * math.sin(angle),
      ),
      Paint()
        ..color = color
        ..strokeWidth = width
        ..strokeCap = StrokeCap.round,
    );
  }

  @override
  bool shouldRepaint(ClockPainter oldDelegate) {
    return oldDelegate.dateTime != dateTime;
  }
}

/// Animated Clock Widget
class AnalogClockWidget extends StatefulWidget {
  const AnalogClockWidget({super.key});

  @override
  State<AnalogClockWidget> createState() => _AnalogClockWidgetState();
}

class _AnalogClockWidgetState extends State<AnalogClockWidget> {
  late DateTime _currentTime;

  @override
  void initState() {
    super.initState();
    _currentTime = DateTime.now();
    _startTimer();
  }

  void _startTimer() {
    Future.delayed(const Duration(seconds: 1), () {
      if (mounted) {
        setState(() {
          _currentTime = DateTime.now();
        });
        _startTimer();
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return CustomPaint(
      size: const Size(250, 250),
      painter: ClockPainter(dateTime: _currentTime),
    );
  }
}
```

---

## ขั้นตอนที่ 2926: Complete Rendering Demo App

```dart
// lib/main_rendering_demo.dart
import 'package:flutter/material.dart';

// import files จากด้านบน (ในโปรเจกต์จริง)

void main() {
  runApp(const RenderingDemoApp());
}

class RenderingDemoApp extends StatelessWidget {
  const RenderingDemoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Rendering Engine Demo',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      home: const RenderingHomePage(),
    );
  }
}

class RenderingHomePage extends StatelessWidget {
  const RenderingHomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Flutter Rendering Engine'),
        backgroundColor: Colors.blue,
        foregroundColor: Colors.white,
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          _buildSection(
            context,
            title: '1. Custom RenderObject',
            description: 'RenderBox with custom layout and painting',
            color: Colors.blue.shade100,
            child: ColoredBoxWidget(color: Colors.blue.shade300),
          ),
          _buildSection(
            context,
            title: '2. Circular Layout',
            description: 'MultiChildRenderObjectWidget',
            color: Colors.green.shade100,
            child: const SizedBox(
              height: 300,
              child: CircularLayoutDemo(),
            ),
          ),
          _buildSection(
            context,
            title: '3. Radar Chart',
            description: 'Complex CustomPainter',
            color: Colors.purple.shade100,
            child: const Center(child: RadarChartWidget()),
          ),
          _buildSection(
            context,
            title: '4. Analog Clock',
            description: 'Animated CustomPainter',
            color: Colors.orange.shade100,
            child: const Center(child: AnalogClockWidget()),
          ),
          _buildSection(
            context,
            title: '5. Animated Waves',
            description: 'Multi-layer animation',
            color: Colors.cyan.shade100,
            child: const AnimatedWaveWidget(),
          ),
          ElevatedButton(
            onPressed: () {
              Navigator.push(
                context,
                MaterialPageRoute(
                  builder: (_) => const ParticleSystemWidget(),
                ),
              );
            },
            child: const Text('Open Particle System Demo'),
          ),
        ],
      ),
    );
  }

  Widget _buildSection(
    BuildContext context, {
    required String title,
    required String description,
    required Color color,
    required Widget child,
  }) {
    return Card(
      margin: const EdgeInsets.only(bottom: 16),
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              title,
              style: const TextStyle(
                fontSize: 18,
                fontWeight: FontWeight.bold,
              ),
            ),
            Text(
              description,
              style: TextStyle(color: Colors.grey.shade600),
            ),
            const SizedBox(height: 12),
            Container(
              padding: const EdgeInsets.all(8),
              decoration: BoxDecoration(
                color: color,
                borderRadius: BorderRadius.circular(8),
              ),
              child: child,
            ),
          ],
        ),
      ),
    );
  }
}

// pubspec.yaml dependencies:
// dependencies:
//   flutter:
//     sdk: flutter
// 
// dev_dependencies:
//   flutter_test:
//     sdk: flutter
```

---

**← [Part 75](part-75-flutter-performance-optimization.md)**
**ต่อไป: [Part 77 →](part-77-dart-meta-programming.md)**
