# Part 51: Custom Paint & Canvas
## ขั้นตอนที่ 1921-1960

## 🎯 เป้าหมายของ Part นี้
- เข้าใจการทำงานของ CustomPainter และ Canvas API
- วาดรูปทรงด้วย drawPath, drawCircle, drawLine, drawRect
- สร้าง AnimatedCustomPainter ที่ animate ด้วย Listenable
- ใช้ Bezier curves สร้าง shape ที่ซับซ้อน
- วาด Line Chart และ Bar Chart จาก scratch
- ใช้ Canvas clipping และ transforms

---

## ขั้นตอนที่ 1921: พื้นฐาน CustomPainter

```dart
// lib/custom_paint/basic_painter.dart
import 'package:flutter/material.dart';

/// CustomPainter พื้นฐาน — วาด circle, rect, line
class BasicShapesPainter extends CustomPainter {
  BasicShapesPainter({
    required this.color,
    required this.strokeWidth,
  });

  final Color color;
  final double strokeWidth;

  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = color
      ..strokeWidth = strokeWidth
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round;

    // วาดวงกลม
    canvas.drawCircle(
      Offset(size.width * 0.25, size.height * 0.3),
      40,
      paint..style = PaintingStyle.fill..color = Colors.blue.withOpacity(0.3),
    );
    canvas.drawCircle(
      Offset(size.width * 0.25, size.height * 0.3),
      40,
      paint..style = PaintingStyle.stroke..color = Colors.blue,
    );

    // วาด rectangle
    final rectPaint = Paint()
      ..color = Colors.green
      ..strokeWidth = 2
      ..style = PaintingStyle.stroke;
    canvas.drawRect(
      Rect.fromLTWH(size.width * 0.55, size.height * 0.1, 80, 60),
      rectPaint,
    );

    // วาด rounded rectangle
    final rrectPaint = Paint()
      ..color = Colors.orange
      ..style = PaintingStyle.fill;
    canvas.drawRRect(
      RRect.fromRectAndRadius(
        Rect.fromLTWH(size.width * 0.1, size.height * 0.55, 100, 50),
        const Radius.circular(16),
      ),
      rrectPaint,
    );

    // วาดเส้น
    final linePaint = Paint()
      ..color = Colors.red
      ..strokeWidth = 3
      ..strokeCap = StrokeCap.round;
    canvas.drawLine(
      Offset(size.width * 0.5, size.height * 0.6),
      Offset(size.width * 0.9, size.height * 0.8),
      linePaint,
    );

    // วาด oval
    final ovalPaint = Paint()
      ..color = Colors.purple.withOpacity(0.5)
      ..style = PaintingStyle.fill;
    canvas.drawOval(
      Rect.fromLTWH(size.width * 0.55, size.height * 0.6, 120, 60),
      ovalPaint,
    );
  }

  @override
  bool shouldRepaint(BasicShapesPainter oldDelegate) =>
      oldDelegate.color != color || oldDelegate.strokeWidth != strokeWidth;
}

class BasicShapesWidget extends StatelessWidget {
  const BasicShapesWidget({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Basic Shapes')),
      body: CustomPaint(
        painter: BasicShapesPainter(color: Colors.blue, strokeWidth: 2),
        child: const SizedBox.expand(),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1922: วาด Path และ drawPath

```dart
// lib/custom_paint/path_painter.dart
import 'package:flutter/material.dart';

class PathDemoPainter extends CustomPainter {
  PathDemoPainter({required this.progress}) : super(repaint: progress);

  final Animation<double> progress;

  @override
  void paint(Canvas canvas, Size size) {
    _drawStar(canvas, size);
    _drawArrow(canvas, size);
    _drawHeartOutline(canvas, size);
  }

  void _drawStar(Canvas canvas, Size size) {
    const cx = 80.0;
    const cy = 80.0;
    const outerR = 50.0;
    const innerR = 20.0;
    const points = 5;

    final path = Path();
    for (int i = 0; i < points * 2; i++) {
      final angle = (i * 3.14159265 / points) - 3.14159265 / 2;
      final r = i.isEven ? outerR : innerR;
      final x = cx + r * Math.cos(angle);
      final y = cy + r * Math.sin(angle);
      if (i == 0) {
        path.moveTo(x, y);
      } else {
        path.lineTo(x, y);
      }
    }
    path.close();

    canvas.drawPath(
      path,
      Paint()
        ..color = Colors.amber
        ..style = PaintingStyle.fill,
    );
    canvas.drawPath(
      path,
      Paint()
        ..color = Colors.orange
        ..style = PaintingStyle.stroke
        ..strokeWidth = 2,
    );
  }

  void _drawArrow(Canvas canvas, Size size) {
    final path = Path()
      ..moveTo(size.width * 0.5, 40)
      ..lineTo(size.width * 0.7, 80)
      ..lineTo(size.width * 0.62, 80)
      ..lineTo(size.width * 0.62, 130)
      ..lineTo(size.width * 0.58, 130)
      ..lineTo(size.width * 0.58, 80)
      ..lineTo(size.width * 0.5, 80)
      ..close();

    canvas.drawPath(
      path,
      Paint()
        ..color = Colors.teal
        ..style = PaintingStyle.fill,
    );
  }

  void _drawHeartOutline(Canvas canvas, Size size) {
    final cx = size.width * 0.75;
    const cy = 80.0;
    const w = 50.0;

    final path = Path();
    path.moveTo(cx, cy + 15);
    path.cubicTo(cx, cy, cx - w / 2, cy - 20, cx - w / 2, cy + 5);
    path.cubicTo(cx - w / 2, cy + 25, cx, cy + 40, cx, cy + 45);
    path.cubicTo(cx, cy + 40, cx + w / 2, cy + 25, cx + w / 2, cy + 5);
    path.cubicTo(cx + w / 2, cy - 20, cx, cy, cx, cy + 15);

    canvas.drawPath(
      path,
      Paint()
        ..color = Colors.red
        ..style = PaintingStyle.fill,
    );
  }

  @override
  bool shouldRepaint(PathDemoPainter oldDelegate) => false;
}

// dart:math ใช้ตรง ๆ ไม่ได้ ใช้ class นี้แทน
class Math {
  static double cos(double x) => _cos(x);
  static double sin(double x) => _sin(x);

  static double _cos(double x) {
    double result = 1;
    double term = 1;
    for (int i = 1; i <= 10; i++) {
      term *= -x * x / ((2 * i - 1) * (2 * i));
      result += term;
    }
    return result;
  }

  static double _sin(double x) {
    double result = x;
    double term = x;
    for (int i = 1; i <= 10; i++) {
      term *= -x * x / ((2 * i) * (2 * i + 1));
      result += term;
    }
    return result;
  }
}

class PathDemoWidget extends StatefulWidget {
  const PathDemoWidget({super.key});

  @override
  State<PathDemoWidget> createState() => _PathDemoWidgetState();
}

class _PathDemoWidgetState extends State<PathDemoWidget>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 2),
    )..repeat();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Path Drawing')),
      body: CustomPaint(
        painter: PathDemoPainter(progress: _controller),
        child: const SizedBox.expand(),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1923: Animated CustomPainter ด้วย Listenable

```dart
// lib/custom_paint/animated_painter.dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

/// WavesPainter — วาดคลื่น sine ที่ animate ด้วย AnimationController
class WavesPainter extends CustomPainter {
  WavesPainter({
    required this.animation,
    required this.waveColor,
    this.amplitude = 20.0,
    this.frequency = 2.0,
  }) : super(repaint: animation);

  final Animation<double> animation;
  final Color waveColor;
  final double amplitude;
  final double frequency;

  @override
  void paint(Canvas canvas, Size size) {
    final phase = animation.value * 2 * math.pi;

    // วาด background gradient
    final bgPaint = Paint()
      ..shader = LinearGradient(
        begin: Alignment.topCenter,
        end: Alignment.bottomCenter,
        colors: [Colors.blue.shade900, Colors.blue.shade400],
      ).createShader(Rect.fromLTWH(0, 0, size.width, size.height));
    canvas.drawRect(Rect.fromLTWH(0, 0, size.width, size.height), bgPaint);

    // วาด 3 layer ของคลื่น
    _drawWave(canvas, size, phase, waveColor.withOpacity(0.4), 0.65);
    _drawWave(canvas, size, phase + math.pi * 0.5, waveColor.withOpacity(0.6), 0.7);
    _drawWave(canvas, size, phase + math.pi, waveColor.withOpacity(0.8), 0.75);
  }

  void _drawWave(
      Canvas canvas, Size size, double phase, Color color, double heightFraction) {
    final path = Path();
    final midY = size.height * heightFraction;

    path.moveTo(0, midY);

    for (double x = 0; x <= size.width; x++) {
      final y =
          midY + amplitude * math.sin((x / size.width * frequency * 2 * math.pi) + phase);
      path.lineTo(x, y);
    }

    path.lineTo(size.width, size.height);
    path.lineTo(0, size.height);
    path.close();

    canvas.drawPath(path, Paint()..color = color);
  }

  @override
  bool shouldRepaint(WavesPainter oldDelegate) =>
      oldDelegate.waveColor != waveColor ||
      oldDelegate.amplitude != amplitude ||
      oldDelegate.frequency != frequency;
}

class AnimatedWavesWidget extends StatefulWidget {
  const AnimatedWavesWidget({super.key});

  @override
  State<AnimatedWavesWidget> createState() => _AnimatedWavesWidgetState();
}

class _AnimatedWavesWidgetState extends State<AnimatedWavesWidget>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 3),
    )..repeat();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Animated Waves'),
        backgroundColor: Colors.blue.shade900,
      ),
      body: CustomPaint(
        painter: WavesPainter(
          animation: _controller,
          waveColor: Colors.white,
          amplitude: 25,
          frequency: 1.5,
        ),
        child: const SizedBox.expand(),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1924: Bezier Curves และ Complex Shapes

```dart
// lib/custom_paint/bezier_painter.dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

class BezierDemoPainter extends CustomPainter {
  const BezierDemoPainter({required this.t});

  final double t; // 0.0 - 1.0 for animation progress

  @override
  void paint(Canvas canvas, Size size) {
    _drawQuadraticBezier(canvas, size);
    _drawCubicBezier(canvas, size);
    _drawBlobShape(canvas, size);
    _drawControlPoints(canvas, size);
  }

  void _drawQuadraticBezier(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = Colors.blue
      ..strokeWidth = 3
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round;

    final path = Path();
    path.moveTo(20, size.height * 0.2);
    path.quadraticBezierTo(
      size.width * 0.5, 20, // control point
      size.width - 20, size.height * 0.2, // end point
    );

    canvas.drawPath(path, paint);

    // Label
    final tp = TextPainter(
      text: const TextSpan(
        text: 'Quadratic Bézier',
        style: TextStyle(color: Colors.blue, fontSize: 12),
      ),
      textDirection: TextDirection.ltr,
    )..layout();
    tp.paint(canvas, Offset(size.width * 0.3, size.height * 0.22));
  }

  void _drawCubicBezier(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = Colors.red
      ..strokeWidth = 3
      ..style = PaintingStyle.stroke;

    final path = Path();
    path.moveTo(20, size.height * 0.45);
    path.cubicTo(
      size.width * 0.25, size.height * 0.3, // cp1
      size.width * 0.75, size.height * 0.6, // cp2
      size.width - 20, size.height * 0.45, // end
    );

    canvas.drawPath(path, paint);

    final tp = TextPainter(
      text: const TextSpan(
        text: 'Cubic Bézier',
        style: TextStyle(color: Colors.red, fontSize: 12),
      ),
      textDirection: TextDirection.ltr,
    )..layout();
    tp.paint(canvas, Offset(size.width * 0.35, size.height * 0.47));
  }

  void _drawBlobShape(Canvas canvas, Size size) {
    final cx = size.width * 0.5;
    final cy = size.height * 0.75;
    const r = 50.0;

    final path = Path();
    final offsets = <Offset>[
      Offset(cx, cy - r),
      Offset(cx + r * 0.9, cy - r * 0.2),
      Offset(cx + r * 0.7, cy + r * 0.7),
      Offset(cx, cy + r * 0.9),
      Offset(cx - r * 0.7, cy + r * 0.7),
      Offset(cx - r * 0.9, cy - r * 0.2),
    ];

    path.moveTo(offsets[0].dx, offsets[0].dy);
    for (int i = 0; i < offsets.length; i++) {
      final current = offsets[i];
      final next = offsets[(i + 1) % offsets.length];
      final cp1 = Offset(
        current.dx + (next.dx - offsets[(i - 1 + offsets.length) % offsets.length].dx) * 0.2,
        current.dy + (next.dy - offsets[(i - 1 + offsets.length) % offsets.length].dy) * 0.2,
      );
      final cp2 = Offset(
        next.dx - (offsets[(i + 2) % offsets.length].dx - current.dx) * 0.2,
        next.dy - (offsets[(i + 2) % offsets.length].dy - current.dy) * 0.2,
      );
      path.cubicTo(cp1.dx, cp1.dy, cp2.dx, cp2.dy, next.dx, next.dy);
    }
    path.close();

    canvas.drawPath(
      path,
      Paint()
        ..shader = RadialGradient(
          colors: [Colors.purple.shade300, Colors.purple.shade800],
        ).createShader(Rect.fromCircle(center: Offset(cx, cy), radius: r)),
    );
  }

  void _drawControlPoints(Canvas canvas, Size size) {
    final dotPaint = Paint()..color = Colors.grey.withOpacity(0.5);
    final positions = [
      Offset(size.width * 0.5, 20),
      Offset(size.width * 0.25, size.height * 0.3),
      Offset(size.width * 0.75, size.height * 0.6),
    ];
    for (final p in positions) {
      canvas.drawCircle(p, 5, dotPaint);
    }
  }

  @override
  bool shouldRepaint(BezierDemoPainter oldDelegate) => oldDelegate.t != t;
}

class BezierDemoWidget extends StatefulWidget {
  const BezierDemoWidget({super.key});

  @override
  State<BezierDemoWidget> createState() => _BezierDemoWidgetState();
}

class _BezierDemoWidgetState extends State<BezierDemoWidget>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;
  late final Animation<double> _anim;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(vsync: this, duration: const Duration(seconds: 4))
      ..repeat(reverse: true);
    _anim = CurvedAnimation(parent: _ctrl, curve: Curves.easeInOut);
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Bézier Curves')),
      body: AnimatedBuilder(
        animation: _anim,
        builder: (context, _) => CustomPaint(
          painter: BezierDemoPainter(t: _anim.value),
          child: const SizedBox.expand(),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1925: Line Chart จาก Scratch

```dart
// lib/custom_paint/line_chart_painter.dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

class LineChartData {
  const LineChartData({
    required this.label,
    required this.values,
    required this.color,
  });
  final String label;
  final List<double> values;
  final Color color;
}

class LineChartPainter extends CustomPainter {
  LineChartPainter({
    required this.datasets,
    required this.xLabels,
    required this.animationValue,
    this.showGrid = true,
    this.showDots = true,
  });

  final List<LineChartData> datasets;
  final List<String> xLabels;
  final double animationValue;
  final bool showGrid;
  final bool showDots;

  static const _paddingLeft = 48.0;
  static const _paddingBottom = 36.0;
  static const _paddingTop = 16.0;
  static const _paddingRight = 16.0;

  @override
  void paint(Canvas canvas, Size size) {
    final chartW = size.width - _paddingLeft - _paddingRight;
    final chartH = size.height - _paddingBottom - _paddingTop;
    final chartRect = Rect.fromLTWH(_paddingLeft, _paddingTop, chartW, chartH);

    // หา min/max
    double minVal = double.infinity;
    double maxVal = double.negativeInfinity;
    for (final ds in datasets) {
      for (final v in ds.values) {
        if (v < minVal) minVal = v;
        if (v > maxVal) maxVal = v;
      }
    }
    minVal = (minVal * 0.9).floorToDouble();
    maxVal = (maxVal * 1.1).ceilToDouble();

    if (showGrid) _drawGrid(canvas, chartRect, minVal, maxVal);
    _drawAxes(canvas, size, chartRect, minVal, maxVal);
    _drawLines(canvas, chartRect, minVal, maxVal);
    _drawLegend(canvas, size);
  }

  void _drawGrid(Canvas canvas, Rect r, double minVal, double maxVal) {
    final gridPaint = Paint()
      ..color = Colors.grey.withOpacity(0.2)
      ..strokeWidth = 1;
    const gridLines = 5;
    for (int i = 0; i <= gridLines; i++) {
      final y = r.top + r.height * (1 - i / gridLines);
      canvas.drawLine(Offset(r.left, y), Offset(r.right, y), gridPaint);
    }
  }

  void _drawAxes(Canvas canvas, Size size, Rect r, double minVal, double maxVal) {
    final axisPaint = Paint()
      ..color = Colors.grey.shade700
      ..strokeWidth = 1.5;
    // Y axis
    canvas.drawLine(Offset(r.left, r.top), Offset(r.left, r.bottom), axisPaint);
    // X axis
    canvas.drawLine(Offset(r.left, r.bottom), Offset(r.right, r.bottom), axisPaint);

    final textStyle = TextStyle(color: Colors.grey.shade600, fontSize: 10);
    const gridLines = 5;

    // Y labels
    for (int i = 0; i <= gridLines; i++) {
      final value = minVal + (maxVal - minVal) * i / gridLines;
      final y = r.bottom - r.height * i / gridLines;
      final tp = TextPainter(
        text: TextSpan(text: value.toStringAsFixed(0), style: textStyle),
        textDirection: TextDirection.ltr,
      )..layout(maxWidth: 40);
      tp.paint(canvas, Offset(r.left - tp.width - 4, y - tp.height / 2));
    }

    // X labels
    if (xLabels.isNotEmpty) {
      final step = r.width / (xLabels.length - 1);
      for (int i = 0; i < xLabels.length; i++) {
        final x = r.left + step * i;
        final tp = TextPainter(
          text: TextSpan(text: xLabels[i], style: textStyle),
          textDirection: TextDirection.ltr,
        )..layout();
        tp.paint(canvas, Offset(x - tp.width / 2, r.bottom + 6));
      }
    }
  }

  void _drawLines(Canvas canvas, Rect r, double minVal, double maxVal) {
    for (final ds in datasets) {
      if (ds.values.isEmpty) continue;
      final step = r.width / (ds.values.length - 1);
      final visibleCount = (ds.values.length * animationValue).round().clamp(2, ds.values.length);

      // Shadow
      final shadowPath = Path();
      for (int i = 0; i < visibleCount; i++) {
        final x = r.left + step * i;
        final y = r.bottom - r.height * (ds.values[i] - minVal) / (maxVal - minVal);
        if (i == 0) shadowPath.moveTo(x, y); else shadowPath.lineTo(x, y);
      }
      shadowPath.lineTo(r.left + step * (visibleCount - 1), r.bottom);
      shadowPath.lineTo(r.left, r.bottom);
      shadowPath.close();
      canvas.drawPath(
        shadowPath,
        Paint()
          ..shader = LinearGradient(
            begin: Alignment.topCenter,
            end: Alignment.bottomCenter,
            colors: [ds.color.withOpacity(0.3), ds.color.withOpacity(0.0)],
          ).createShader(r),
      );

      // Line
      final linePath = Path();
      final linePaint = Paint()
        ..color = ds.color
        ..strokeWidth = 2.5
        ..style = PaintingStyle.stroke
        ..strokeCap = StrokeCap.round
        ..strokeJoin = StrokeJoin.round;

      for (int i = 0; i < visibleCount; i++) {
        final x = r.left + step * i;
        final y = r.bottom - r.height * (ds.values[i] - minVal) / (maxVal - minVal);
        if (i == 0) linePath.moveTo(x, y); else linePath.lineTo(x, y);
      }
      canvas.drawPath(linePath, linePaint);

      // Dots
      if (showDots) {
        for (int i = 0; i < visibleCount; i++) {
          final x = r.left + step * i;
          final y = r.bottom - r.height * (ds.values[i] - minVal) / (maxVal - minVal);
          canvas.drawCircle(Offset(x, y), 4, Paint()..color = ds.color);
          canvas.drawCircle(
              Offset(x, y), 3, Paint()..color = Colors.white);
        }
      }
    }
  }

  void _drawLegend(Canvas canvas, Size size) {
    double x = 60;
    const y = 8.0;
    for (final ds in datasets) {
      canvas.drawRRect(
        RRect.fromRectAndRadius(
          Rect.fromLTWH(x, y, 20, 10),
          const Radius.circular(3),
        ),
        Paint()..color = ds.color,
      );
      final tp = TextPainter(
        text: TextSpan(
          text: ds.label,
          style: const TextStyle(fontSize: 10, color: Colors.black87),
        ),
        textDirection: TextDirection.ltr,
      )..layout();
      tp.paint(canvas, Offset(x + 24, y));
      x += 80;
    }
  }

  @override
  bool shouldRepaint(LineChartPainter oldDelegate) =>
      oldDelegate.animationValue != animationValue;
}

class LineChartWidget extends StatefulWidget {
  const LineChartWidget({super.key});

  @override
  State<LineChartWidget> createState() => _LineChartWidgetState();
}

class _LineChartWidgetState extends State<LineChartWidget>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;
  late final Animation<double> _anim;

  final datasets = const [
    LineChartData(
      label: 'Revenue',
      values: [120, 145, 132, 167, 180, 155, 200, 185, 220, 240, 210, 260],
      color: Colors.blue,
    ),
    LineChartData(
      label: 'Expense',
      values: [80, 90, 85, 100, 110, 95, 120, 115, 130, 145, 125, 150],
      color: Colors.red,
    ),
  ];

  final xLabels = const ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(vsync: this, duration: const Duration(milliseconds: 1500));
    _anim = CurvedAnimation(parent: _ctrl, curve: Curves.easeOutCubic);
    _ctrl.forward();
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Line Chart')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Card(
          child: Padding(
            padding: const EdgeInsets.all(8),
            child: AnimatedBuilder(
              animation: _anim,
              builder: (context, _) => CustomPaint(
                painter: LineChartPainter(
                  datasets: datasets,
                  xLabels: xLabels,
                  animationValue: _anim.value,
                ),
                child: const SizedBox(height: 250),
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1926: Bar Chart จาก Scratch

```dart
// lib/custom_paint/bar_chart_painter.dart
import 'package:flutter/material.dart';

class BarChartItem {
  const BarChartItem({
    required this.label,
    required this.value,
    required this.color,
  });
  final String label;
  final double value;
  final Color color;
}

class BarChartPainter extends CustomPainter {
  const BarChartPainter({
    required this.items,
    required this.animationValue,
    this.barRadius = 6.0,
  });

  final List<BarChartItem> items;
  final double animationValue;
  final double barRadius;

  static const _paddingLeft = 50.0;
  static const _paddingBottom = 40.0;
  static const _paddingTop = 20.0;
  static const _paddingRight = 16.0;
  static const _barGap = 12.0;

  @override
  void paint(Canvas canvas, Size size) {
    final chartW = size.width - _paddingLeft - _paddingRight;
    final chartH = size.height - _paddingBottom - _paddingTop;
    final chartRect = Rect.fromLTWH(_paddingLeft, _paddingTop, chartW, chartH);

    final maxVal = items.fold<double>(0, (m, e) => e.value > m ? e.value : m) * 1.1;
    final barW = (chartW - _barGap * (items.length + 1)) / items.length;

    _drawGridAndAxes(canvas, chartRect, maxVal);
    _drawBars(canvas, chartRect, maxVal, barW);
  }

  void _drawGridAndAxes(Canvas canvas, Rect r, double maxVal) {
    final gridPaint = Paint()
      ..color = Colors.grey.withOpacity(0.2)
      ..strokeWidth = 1;
    final axisPaint = Paint()
      ..color = Colors.grey.shade600
      ..strokeWidth = 1.5;
    final labelStyle = TextStyle(color: Colors.grey.shade600, fontSize: 10);

    const gridLines = 5;
    for (int i = 0; i <= gridLines; i++) {
      final y = r.top + r.height * (1 - i / gridLines);
      canvas.drawLine(Offset(r.left, y), Offset(r.right, y), gridPaint);
      final val = maxVal * i / gridLines;
      final tp = TextPainter(
        text: TextSpan(text: val.toStringAsFixed(0), style: labelStyle),
        textDirection: TextDirection.ltr,
      )..layout(maxWidth: 44);
      tp.paint(canvas, Offset(r.left - tp.width - 4, y - tp.height / 2));
    }
    canvas.drawLine(Offset(r.left, r.top), Offset(r.left, r.bottom), axisPaint);
    canvas.drawLine(Offset(r.left, r.bottom), Offset(r.right, r.bottom), axisPaint);
  }

  void _drawBars(Canvas canvas, Rect r, double maxVal, double barW) {
    for (int i = 0; i < items.length; i++) {
      final item = items[i];
      final barH = r.height * (item.value / maxVal) * animationValue;
      final x = r.left + _barGap * (i + 1) + barW * i;
      final y = r.bottom - barH;

      // Bar shadow
      canvas.drawRRect(
        RRect.fromRectAndRadius(
          Rect.fromLTWH(x + 2, y + 2, barW, barH),
          Radius.circular(barRadius),
        ),
        Paint()..color = Colors.black12,
      );

      // Bar fill with gradient
      canvas.drawRRect(
        RRect.fromRectAndRadius(
          Rect.fromLTWH(x, y, barW, barH),
          Radius.circular(barRadius),
        ),
        Paint()
          ..shader = LinearGradient(
            begin: Alignment.topCenter,
            end: Alignment.bottomCenter,
            colors: [item.color, item.color.withOpacity(0.6)],
          ).createShader(Rect.fromLTWH(x, y, barW, barH)),
      );

      // Value label on top of bar
      if (animationValue > 0.8) {
        final tp = TextPainter(
          text: TextSpan(
            text: item.value.toStringAsFixed(0),
            style: TextStyle(
              color: item.color,
              fontSize: 10,
              fontWeight: FontWeight.bold,
            ),
          ),
          textDirection: TextDirection.ltr,
        )..layout();
        tp.paint(canvas, Offset(x + barW / 2 - tp.width / 2, y - 14));
      }

      // X label
      final tp = TextPainter(
        text: TextSpan(
          text: item.label,
          style: TextStyle(color: Colors.grey.shade600, fontSize: 10),
        ),
        textDirection: TextDirection.ltr,
      )..layout(maxWidth: barW + 4);
      tp.paint(canvas, Offset(x + barW / 2 - tp.width / 2, r.bottom + 6));
    }
  }

  @override
  bool shouldRepaint(BarChartPainter oldDelegate) =>
      oldDelegate.animationValue != animationValue;
}

class BarChartWidget extends StatefulWidget {
  const BarChartWidget({super.key});

  @override
  State<BarChartWidget> createState() => _BarChartWidgetState();
}

class _BarChartWidgetState extends State<BarChartWidget>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;
  late final Animation<double> _anim;

  final items = const [
    BarChartItem(label: 'Mon', value: 42, color: Colors.blue),
    BarChartItem(label: 'Tue', value: 78, color: Colors.green),
    BarChartItem(label: 'Wed', value: 55, color: Colors.orange),
    BarChartItem(label: 'Thu', value: 90, color: Colors.purple),
    BarChartItem(label: 'Fri', value: 65, color: Colors.red),
    BarChartItem(label: 'Sat', value: 48, color: Colors.teal),
    BarChartItem(label: 'Sun', value: 30, color: Colors.pink),
  ];

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(vsync: this, duration: const Duration(milliseconds: 1200));
    _anim = CurvedAnimation(parent: _ctrl, curve: Curves.easeOutBack);
    _ctrl.forward();
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Bar Chart')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            Card(
              child: Padding(
                padding: const EdgeInsets.all(8),
                child: AnimatedBuilder(
                  animation: _anim,
                  builder: (ctx, _) => CustomPaint(
                    painter: BarChartPainter(
                      items: items,
                      animationValue: _anim.value,
                    ),
                    child: const SizedBox(height: 220),
                  ),
                ),
              ),
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () {
                _ctrl.reset();
                _ctrl.forward();
              },
              child: const Text('Replay'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1927: Canvas Clipping และ Transforms

```dart
// lib/custom_paint/clip_transform_painter.dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

class ClipTransformPainter extends CustomPainter {
  const ClipTransformPainter({required this.rotation});
  final double rotation;

  @override
  void paint(Canvas canvas, Size size) {
    final cx = size.width / 2;
    final cy = size.height / 2;

    // Demo 1: clipRect — วาดใน rectangle เท่านั้น
    canvas.save();
    canvas.clipRect(Rect.fromLTWH(20, 20, size.width * 0.4, 100));
    _drawCheckerboard(canvas, size);
    canvas.restore();

    // Demo 2: clipPath — วาดใน circle เท่านั้น
    canvas.save();
    final circleClipPath = Path()
      ..addOval(Rect.fromCircle(center: Offset(size.width * 0.7, 80), radius: 60));
    canvas.clipPath(circleClipPath);
    _drawRainbow(canvas, Offset(size.width * 0.7, 80), 60);
    canvas.restore();

    // Demo 3: rotate transform
    canvas.save();
    canvas.translate(cx, cy);
    canvas.rotate(rotation);
    _drawCross(canvas);
    canvas.restore();

    // Demo 4: scale transform
    canvas.save();
    canvas.translate(cx - 80, size.height * 0.7);
    canvas.scale(0.5 + math.sin(rotation) * 0.3, 1.0);
    canvas.drawRect(
      Rect.fromLTWH(-40, -20, 80, 40),
      Paint()..color = Colors.teal,
    );
    canvas.restore();

    // Demo 5: skew transform
    canvas.save();
    canvas.translate(cx + 60, size.height * 0.75);
    canvas.skew(math.sin(rotation) * 0.5, 0);
    canvas.drawRect(
      Rect.fromLTWH(-30, -25, 60, 50),
      Paint()..color = Colors.deepOrange,
    );
    canvas.restore();
  }

  void _drawCheckerboard(Canvas canvas, Size size) {
    const cellSize = 20.0;
    for (int row = 0; row * cellSize < 200; row++) {
      for (int col = 0; col * cellSize < size.width; col++) {
        if ((row + col) % 2 == 0) {
          canvas.drawRect(
            Rect.fromLTWH(col * cellSize, row * cellSize, cellSize, cellSize),
            Paint()..color = Colors.indigo.withOpacity(0.6),
          );
        }
      }
    }
  }

  void _drawRainbow(Canvas canvas, Offset center, double radius) {
    final colors = [
      Colors.red, Colors.orange, Colors.yellow,
      Colors.green, Colors.blue, Colors.indigo, Colors.purple,
    ];
    for (int i = colors.length - 1; i >= 0; i--) {
      canvas.drawCircle(
        center,
        radius * (i + 1) / colors.length,
        Paint()..color = colors[i],
      );
    }
  }

  void _drawCross(Canvas canvas) {
    final paint = Paint()
      ..color = Colors.green
      ..strokeWidth = 8
      ..strokeCap = StrokeCap.round;
    canvas.drawLine(const Offset(-40, 0), const Offset(40, 0), paint);
    canvas.drawLine(const Offset(0, -40), const Offset(0, 40), paint);
    paint.color = Colors.greenAccent;
    canvas.drawCircle(Offset.zero, 12, paint);
  }

  @override
  bool shouldRepaint(ClipTransformPainter oldDelegate) =>
      oldDelegate.rotation != rotation;
}

class ClipTransformWidget extends StatefulWidget {
  const ClipTransformWidget({super.key});

  @override
  State<ClipTransformWidget> createState() => _ClipTransformWidgetState();
}

class _ClipTransformWidgetState extends State<ClipTransformWidget>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 4),
    )..repeat();
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Clip & Transform')),
      body: AnimatedBuilder(
        animation: _ctrl,
        builder: (ctx, _) => CustomPaint(
          painter: ClipTransformPainter(rotation: _ctrl.value * 2 * math.pi),
          child: const SizedBox.expand(),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1928: Full Canvas Demo App

```dart
// lib/main_canvas_demo.dart
import 'package:flutter/material.dart';
import 'custom_paint/basic_painter.dart';
import 'custom_paint/path_painter.dart';
import 'custom_paint/animated_painter.dart';
import 'custom_paint/bezier_painter.dart';
import 'custom_paint/line_chart_painter.dart';
import 'custom_paint/bar_chart_painter.dart';
import 'custom_paint/clip_transform_painter.dart';

void main() => runApp(const CanvasDemoApp());

class CanvasDemoApp extends StatelessWidget {
  const CanvasDemoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Canvas Demo',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorSchemeSeed: Colors.blue,
        useMaterial3: true,
      ),
      home: const CanvasDemoHome(),
    );
  }
}

class CanvasDemoHome extends StatelessWidget {
  const CanvasDemoHome({super.key});

  static final _demos = <_Demo>[
    _Demo('Basic Shapes', Icons.category, () => const BasicShapesWidget()),
    _Demo('Bézier Curves', Icons.gesture, () => const BezierDemoWidget()),
    _Demo('Animated Waves', Icons.waves, () => const AnimatedWavesWidget()),
    _Demo('Line Chart', Icons.show_chart, () => const LineChartWidget()),
    _Demo('Bar Chart', Icons.bar_chart, () => const BarChartWidget()),
    _Demo('Clip & Transform', Icons.transform, () => const ClipTransformWidget()),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Part 51 — Custom Paint')),
      body: ListView.separated(
        padding: const EdgeInsets.all(16),
        itemCount: _demos.length,
        separatorBuilder: (_, __) => const SizedBox(height: 8),
        itemBuilder: (ctx, i) {
          final demo = _demos[i];
          return ListTile(
            leading: Icon(demo.icon, color: Colors.blue),
            title: Text(demo.title),
            trailing: const Icon(Icons.chevron_right),
            shape: RoundedRectangleBorder(
              borderRadius: BorderRadius.circular(12),
              side: BorderSide(color: Colors.grey.shade200),
            ),
            onTap: () => Navigator.push(
              ctx,
              MaterialPageRoute(builder: (_) => demo.builder()),
            ),
          );
        },
      ),
    );
  }
}

class _Demo {
  const _Demo(this.title, this.icon, this.builder);
  final String title;
  final IconData icon;
  final Widget Function() builder;
}
```

---

**← [Part 50](part-50-platform-channels.md)**
**ต่อไป: [Part 52 →](part-52-animations-advanced.md)**
