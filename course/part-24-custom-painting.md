# Part 24: Custom Painting
## ขั้นตอนที่ 841-880

---

## 🎯 เป้าหมายของ Part นี้

- CustomPainter พื้นฐาน
- Canvas API (drawLine, drawRect, drawCircle, drawPath)
- Paint styles (stroke, fill, gradient)
- Clip widgets
- Custom shapes
- Chart widgets

---

## ขั้นตอนที่ 841: CustomPainter พื้นฐาน

```dart
import 'package:flutter/material.dart';
import 'dart:math' as math;

// ─── พื้นฐาน Canvas ───
class BasicShapesPainter extends CustomPainter {
  @override
  void paint(Canvas canvas, Size size) {
    // ─── Paint ───
    Paint paint = Paint()
      ..color = Colors.blue
      ..strokeWidth = 3
      ..style = PaintingStyle.stroke;

    // Rectangle
    canvas.drawRect(
      Rect.fromLTWH(20, 20, 100, 80),
      paint..color = Colors.red,
    );

    // Circle
    canvas.drawCircle(
      Offset(200, 60),
      40,
      paint..color = Colors.green,
    );

    // Line
    canvas.drawLine(
      const Offset(20, 150),
      const Offset(300, 150),
      paint..color = Colors.purple..strokeWidth = 2,
    );

    // Path (triangle)
    Path triangle = Path()
      ..moveTo(160, 180)
      ..lineTo(120, 260)
      ..lineTo(200, 260)
      ..close();

    canvas.drawPath(
      triangle,
      paint..color = Colors.orange..style = PaintingStyle.fill,
    );

    // RRect (rounded rect)
    canvas.drawRRect(
      RRect.fromRectAndRadius(
        const Rect.fromLTWH(20, 280, 120, 60),
        const Radius.circular(16),
      ),
      paint..color = Colors.teal..style = PaintingStyle.stroke,
    );

    // Oval
    canvas.drawOval(
      const Rect.fromLTWH(160, 280, 150, 60),
      paint..color = Colors.indigo..style = PaintingStyle.fill,
    );
  }

  @override
  bool shouldRepaint(BasicShapesPainter old) => false;
}

// ─── Widget ───
class ShapesDemo extends StatelessWidget {
  const ShapesDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return CustomPaint(
      size: const Size(double.infinity, 400),
      painter: BasicShapesPainter(),
    );
  }
}
```

---

## ขั้นตอนที่ 842: Gradient และ Shadow

```dart
import 'package:flutter/material.dart';
import 'dart:math' as math;

class GradientPainter extends CustomPainter {
  @override
  void paint(Canvas canvas, Size size) {
    // Linear gradient
    Paint linearGrad = Paint()
      ..shader = LinearGradient(
        colors: [Colors.purple, Colors.blue, Colors.cyan],
        begin: Alignment.topLeft,
        end: Alignment.bottomRight,
      ).createShader(Rect.fromLTWH(0, 0, size.width / 2, 150));

    canvas.drawRect(
      Rect.fromLTWH(20, 20, size.width / 2 - 40, 150),
      linearGrad,
    );

    // Radial gradient
    Paint radialGrad = Paint()
      ..shader = RadialGradient(
        colors: [Colors.yellow, Colors.orange, Colors.red],
        radius: 0.5,
      ).createShader(Rect.fromCenter(
        center: Offset(size.width * 0.75, 95),
        width: 120,
        height: 120,
      ));

    canvas.drawCircle(
      Offset(size.width * 0.75, 95),
      60,
      radialGrad,
    );

    // Shadow effect
    canvas.drawShadow(
      Path()
        ..addRRect(RRect.fromRectAndRadius(
          Rect.fromLTWH(20, 200, 200, 100),
          const Radius.circular(20),
        )),
      Colors.black,
      8,
      true,
    );

    canvas.drawRRect(
      RRect.fromRectAndRadius(
        const Rect.fromLTWH(20, 200, 200, 100),
        const Radius.circular(20),
      ),
      Paint()..color = Colors.white,
    );
  }

  @override
  bool shouldRepaint(GradientPainter old) => false;
}
```

---

## ขั้นตอนที่ 843: Bar Chart

```dart
import 'package:flutter/material.dart';

class BarChart extends StatelessWidget {
  final List<double> data;
  final List<String> labels;
  final Color barColor;
  final String title;

  const BarChart({
    super.key,
    required this.data,
    required this.labels,
    this.barColor = Colors.blue,
    this.title = '',
  });

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        if (title.isNotEmpty)
          Padding(
            padding: const EdgeInsets.only(bottom: 8),
            child: Text(title, style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
          ),
        CustomPaint(
          size: const Size(double.infinity, 200),
          painter: _BarChartPainter(data: data, labels: labels, barColor: barColor),
        ),
      ],
    );
  }
}

class _BarChartPainter extends CustomPainter {
  final List<double> data;
  final List<String> labels;
  final Color barColor;

  _BarChartPainter({required this.data, required this.labels, required this.barColor});

  @override
  void paint(Canvas canvas, Size size) {
    if (data.isEmpty) return;

    double maxValue = data.reduce(math.max);
    double padding = 40;
    double chartWidth = size.width - padding * 2;
    double chartHeight = size.height - padding * 2;
    double barWidth = chartWidth / data.length * 0.6;
    double barSpacing = chartWidth / data.length;

    Paint gridPaint = Paint()
      ..color = Colors.grey.shade200
      ..strokeWidth = 1;

    Paint barPaint = Paint()..color = barColor;

    // Draw grid lines
    for (int i = 0; i <= 4; i++) {
      double y = padding + chartHeight - (chartHeight / 4 * i);
      canvas.drawLine(
        Offset(padding, y),
        Offset(size.width - padding, y),
        gridPaint,
      );

      // Y-axis labels
      double value = maxValue / 4 * i;
      TextPainter tp = TextPainter(
        text: TextSpan(
          text: value.toInt().toString(),
          style: TextStyle(color: Colors.grey.shade600, fontSize: 10),
        ),
        textDirection: TextDirection.ltr,
      )..layout();
      tp.paint(canvas, Offset(0, y - 6));
    }

    // Draw bars
    for (int i = 0; i < data.length; i++) {
      double barHeight = (data[i] / maxValue) * chartHeight;
      double x = padding + barSpacing * i + (barSpacing - barWidth) / 2;
      double y = padding + chartHeight - barHeight;

      // Bar with gradient
      barPaint.shader = LinearGradient(
        colors: [barColor.withOpacity(0.7), barColor],
        begin: Alignment.topCenter,
        end: Alignment.bottomCenter,
      ).createShader(Rect.fromLTWH(x, y, barWidth, barHeight));

      canvas.drawRRect(
        RRect.fromRectAndCorners(
          Rect.fromLTWH(x, y, barWidth, barHeight),
          topLeft: const Radius.circular(4),
          topRight: const Radius.circular(4),
        ),
        barPaint,
      );

      // Value label
      TextPainter valueTp = TextPainter(
        text: TextSpan(
          text: data[i].toInt().toString(),
          style: const TextStyle(color: Colors.black, fontSize: 10, fontWeight: FontWeight.bold),
        ),
        textDirection: TextDirection.ltr,
      )..layout();
      valueTp.paint(canvas, Offset(x + barWidth / 2 - valueTp.width / 2, y - 16));

      // X-axis label
      if (i < labels.length) {
        TextPainter labelTp = TextPainter(
          text: TextSpan(
            text: labels[i],
            style: TextStyle(color: Colors.grey.shade700, fontSize: 10),
          ),
          textDirection: TextDirection.ltr,
        )..layout();
        labelTp.paint(canvas, Offset(
          x + barWidth / 2 - labelTp.width / 2,
          padding + chartHeight + 4,
        ));
      }
    }
  }

  @override
  bool shouldRepaint(_BarChartPainter old) {
    return old.data != data;
  }
}

// ─── Pie Chart ───
class PieChart extends StatelessWidget {
  final List<PieSegment> segments;
  final double size;
  final double holeRadius;

  const PieChart({
    super.key,
    required this.segments,
    this.size = 200,
    this.holeRadius = 0,
  });

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      width: size,
      height: size,
      child: CustomPaint(
        painter: _PieChartPainter(segments: segments, holeRadius: holeRadius),
      ),
    );
  }
}

class PieSegment {
  final String label;
  final double value;
  final Color color;

  const PieSegment({required this.label, required this.value, required this.color});
}

class _PieChartPainter extends CustomPainter {
  final List<PieSegment> segments;
  final double holeRadius;

  _PieChartPainter({required this.segments, required this.holeRadius});

  @override
  void paint(Canvas canvas, Size size) {
    Offset center = size.center(Offset.zero);
    double radius = size.width / 2;
    double total = segments.fold(0, (sum, s) => sum + s.value);

    double startAngle = -math.pi / 2;

    for (PieSegment segment in segments) {
      double sweepAngle = (segment.value / total) * 2 * math.pi;

      canvas.drawArc(
        Rect.fromCircle(center: center, radius: radius),
        startAngle,
        sweepAngle,
        true,
        Paint()..color = segment.color,
      );

      startAngle += sweepAngle;
    }

    // Donut hole
    if (holeRadius > 0) {
      canvas.drawCircle(center, holeRadius, Paint()..color = Colors.white);
    }
  }

  @override
  bool shouldRepaint(_PieChartPainter old) => old.segments != segments;
}

// ─── Usage ───
class ChartsDemo extends StatelessWidget {
  const ChartsDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child: Column(
        children: [
          BarChart(
            title: 'Monthly Sales',
            data: const [120, 180, 90, 240, 160, 200],
            labels: const ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun'],
            barColor: Colors.blue,
          ),
          const SizedBox(height: 32),
          const PieChart(
            segments: [
              PieSegment(label: 'Flutter', value: 40, color: Colors.blue),
              PieSegment(label: 'React Native', value: 25, color: Colors.cyan),
              PieSegment(label: 'SwiftUI', value: 20, color: Colors.orange),
              PieSegment(label: 'Jetpack', value: 15, color: Colors.green),
            ],
            holeRadius: 60,
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 844: ClipPath Custom Shapes

```dart
import 'package:flutter/material.dart';

// ─── Wave Clipper ───
class WaveClipper extends CustomClipper<Path> {
  final bool isBottom;

  WaveClipper({this.isBottom = false});

  @override
  Path getClip(Size size) {
    Path path = Path();

    if (isBottom) {
      path.lineTo(0, size.height - 40);
      path.quadraticBezierTo(size.width / 4, size.height, size.width / 2, size.height - 40);
      path.quadraticBezierTo(size.width * 3 / 4, size.height - 80, size.width, size.height - 40);
      path.lineTo(size.width, 0);
    } else {
      path.moveTo(0, 40);
      path.quadraticBezierTo(size.width / 4, 0, size.width / 2, 40);
      path.quadraticBezierTo(size.width * 3 / 4, 80, size.width, 40);
      path.lineTo(size.width, size.height);
      path.lineTo(0, size.height);
    }

    path.close();
    return path;
  }

  @override
  bool shouldReclip(WaveClipper old) => false;
}

// ─── Diagonal Clipper ───
class DiagonalClipper extends CustomClipper<Path> {
  @override
  Path getClip(Size size) {
    return Path()
      ..lineTo(0, size.height - 60)
      ..lineTo(size.width, size.height)
      ..lineTo(size.width, 0)
      ..close();
  }

  @override
  bool shouldReclip(DiagonalClipper old) => false;
}

// ─── Usage ───
class ClipDemo extends StatelessWidget {
  const ClipDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // Wave header
        ClipPath(
          clipper: WaveClipper(),
          child: Container(
            height: 200,
            decoration: const BoxDecoration(
              gradient: LinearGradient(
                colors: [Color(0xFF6C63FF), Color(0xFF3498DB)],
              ),
            ),
            child: const Center(
              child: Text(
                'Wave Header',
                style: TextStyle(color: Colors.white, fontSize: 24),
              ),
            ),
          ),
        ),

        const SizedBox(height: 20),

        // Diagonal image
        ClipPath(
          clipper: DiagonalClipper(),
          child: Container(
            height: 200,
            color: Colors.orange.shade200,
            child: const Center(child: Text('Diagonal Clip')),
          ),
        ),

        const SizedBox(height: 20),

        // ClipOval
        ClipOval(
          child: Image.network(
            'https://picsum.photos/200/200',
            width: 120,
            height: 120,
            fit: BoxFit.cover,
          ),
        ),
      ],
    );
  }
}
```

---

**← [Part 23 - Performance](part-23-performance.md)**

**ต่อไป: [Part 25 - Platform Channels →](part-25-platform-channels.md)**
