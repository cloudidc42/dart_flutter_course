# Part 72: Advanced Animations & Physics ใน Flutter
## ขั้นตอนที่ 2761-2800

## 🎯 เป้าหมายของ Part นี้
- ใช้ SpringSimulation สำหรับ Spring Physics
- ใช้ FrictionSimulation สำหรับ Drag/Friction
- สร้าง Gravity Simulation
- กำหนด Custom Scroll Physics
- สร้าง Physics-based Card Flip และ Toss
- ครบ working Flutter code ทั้งหมด

---

## ขั้นตอนที่ 2761: พื้นฐาน Physics Simulation ใน Flutter

```dart
// main.dart
import 'package:flutter/material.dart';
import 'package:flutter/physics.dart';

void main() => runApp(const PhysicsAnimationsApp());

class PhysicsAnimationsApp extends StatelessWidget {
  const PhysicsAnimationsApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Physics Animations',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const PhysicsDemoHome(),
    );
  }
}

class PhysicsDemoHome extends StatelessWidget {
  const PhysicsDemoHome({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Flutter Physics Animations')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          _DemoCard(
            title: 'Spring Simulation',
            subtitle: 'Mass-spring system physics',
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const SpringDemo()),
            ),
          ),
          _DemoCard(
            title: 'Friction Simulation',
            subtitle: 'Drag and friction physics',
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const FrictionDemo()),
            ),
          ),
          _DemoCard(
            title: 'Gravity Simulation',
            subtitle: 'Gravity and bounce physics',
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const GravityDemo()),
            ),
          ),
          _DemoCard(
            title: 'Card Flip',
            subtitle: 'Physics-based card flip animation',
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const CardFlipDemo()),
            ),
          ),
          _DemoCard(
            title: 'Card Toss',
            subtitle: 'Swipe to toss cards',
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const CardTossDemo()),
            ),
          ),
          _DemoCard(
            title: 'Custom Scroll Physics',
            subtitle: 'Magnetic snap scroll physics',
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const CustomScrollPhysicsDemo()),
            ),
          ),
        ],
      ),
    );
  }
}

class _DemoCard extends StatelessWidget {
  final String title;
  final String subtitle;
  final VoidCallback onTap;

  const _DemoCard({
    required this.title,
    required this.subtitle,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.only(bottom: 12),
      child: ListTile(
        title: Text(title, style: const TextStyle(fontWeight: FontWeight.bold)),
        subtitle: Text(subtitle),
        trailing: const Icon(Icons.arrow_forward_ios),
        onTap: onTap,
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2762: Spring Simulation

```dart
// spring_demo.dart
import 'package:flutter/material.dart';
import 'package:flutter/physics.dart';

class SpringDemo extends StatefulWidget {
  const SpringDemo({super.key});

  @override
  State<SpringDemo> createState() => _SpringDemoState();
}

class _SpringDemoState extends State<SpringDemo>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<Offset> _animation;
  Offset _dragOffset = Offset.zero;
  Offset _dragVelocity = Offset.zero;

  // Spring parameters
  double _stiffness = 100.0;
  double _damping = 10.0;
  double _mass = 1.0;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController.unbounded(vsync: this);
    _animation = _controller.drive(
      Tween<Offset>(begin: Offset.zero, end: Offset.zero),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  void _runSpringAnimation(Offset pixelsPerSecond, Offset currentOffset) {
    _animation = _controller.drive(
      Tween<Offset>(begin: currentOffset, end: Offset.zero),
    );

    // Create spring description
    final spring = SpringDescription(
      mass: _mass,
      stiffness: _stiffness,
      damping: _damping,
    );

    // Calculate velocity magnitude
    final unitsPerSecondX = pixelsPerSecond.dx / 200;
    final unitsPerSecondY = pixelsPerSecond.dy / 200;

    final simulation = SpringSimulation(
      spring,
      1, // start at 1 (the current offset)
      0, // end at 0 (center)
      -unitsPerSecondX, // initial velocity X
    );

    _controller.animateWith(simulation);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Spring Simulation')),
      body: Column(
        children: [
          Expanded(
            child: GestureDetector(
              onPanUpdate: (details) {
                setState(() {
                  _dragOffset += details.delta;
                  _dragVelocity = details.velocity.pixelsPerSecond;
                });
              },
              onPanEnd: (details) {
                _runSpringAnimation(
                  details.velocity.pixelsPerSecond,
                  _dragOffset,
                );
              },
              child: Stack(
                children: [
                  Center(
                    child: AnimatedBuilder(
                      animation: _controller,
                      builder: (context, child) {
                        final offset = _controller.isAnimating
                            ? _animation.value * 200
                            : _dragOffset;
                        return Transform.translate(
                          offset: offset,
                          child: child,
                        );
                      },
                      child: _SpringBall(
                        isDragging: _controller.isAnimating == false,
                      ),
                    ),
                  ),
                  // Center indicator
                  const Center(
                    child: Icon(
                      Icons.add,
                      color: Colors.red,
                      size: 24,
                    ),
                  ),
                ],
              ),
            ),
          ),
          _SpringControls(
            stiffness: _stiffness,
            damping: _damping,
            mass: _mass,
            onStiffnessChanged: (v) => setState(() => _stiffness = v),
            onDampingChanged: (v) => setState(() => _damping = v),
            onMassChanged: (v) => setState(() => _mass = v),
          ),
        ],
      ),
    );
  }
}

class _SpringBall extends StatelessWidget {
  final bool isDragging;

  const _SpringBall({required this.isDragging});

  @override
  Widget build(BuildContext context) {
    return AnimatedContainer(
      duration: const Duration(milliseconds: 100),
      width: isDragging ? 90 : 80,
      height: isDragging ? 90 : 80,
      decoration: BoxDecoration(
        shape: BoxShape.circle,
        gradient: RadialGradient(
          colors: [
            Colors.deepPurple[300]!,
            Colors.deepPurple[700]!,
          ],
        ),
        boxShadow: [
          BoxShadow(
            color: Colors.deepPurple.withOpacity(0.4),
            blurRadius: isDragging ? 20 : 10,
            spreadRadius: isDragging ? 4 : 2,
          ),
        ],
      ),
      child: const Icon(Icons.circle, color: Colors.white, size: 32),
    );
  }
}

class _SpringControls extends StatelessWidget {
  final double stiffness;
  final double damping;
  final double mass;
  final ValueChanged<double> onStiffnessChanged;
  final ValueChanged<double> onDampingChanged;
  final ValueChanged<double> onMassChanged;

  const _SpringControls({
    required this.stiffness,
    required this.damping,
    required this.mass,
    required this.onStiffnessChanged,
    required this.onDampingChanged,
    required this.onMassChanged,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: Colors.grey[100],
        borderRadius: const BorderRadius.vertical(top: Radius.circular(16)),
      ),
      child: Column(
        children: [
          _SliderRow(
            label: 'Stiffness',
            value: stiffness,
            min: 10,
            max: 500,
            onChanged: onStiffnessChanged,
          ),
          _SliderRow(
            label: 'Damping',
            value: damping,
            min: 1,
            max: 50,
            onChanged: onDampingChanged,
          ),
          _SliderRow(
            label: 'Mass',
            value: mass,
            min: 0.1,
            max: 5.0,
            onChanged: onMassChanged,
          ),
        ],
      ),
    );
  }
}

class _SliderRow extends StatelessWidget {
  final String label;
  final double value;
  final double min;
  final double max;
  final ValueChanged<double> onChanged;

  const _SliderRow({
    required this.label,
    required this.value,
    required this.min,
    required this.max,
    required this.onChanged,
  });

  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        SizedBox(
          width: 80,
          child: Text('$label:', style: const TextStyle(fontSize: 12)),
        ),
        Expanded(
          child: Slider(value: value, min: min, max: max, onChanged: onChanged),
        ),
        SizedBox(
          width: 50,
          child: Text(value.toStringAsFixed(1), style: const TextStyle(fontSize: 12)),
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 2763: Friction Simulation

```dart
// friction_demo.dart
import 'package:flutter/material.dart';
import 'package:flutter/physics.dart';

class FrictionDemo extends StatefulWidget {
  const FrictionDemo({super.key});

  @override
  State<FrictionDemo> createState() => _FrictionDemoState();
}

class _FrictionDemoState extends State<FrictionDemo>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  double _position = 0.5; // normalized 0-1
  double _friction = 0.3;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController.unbounded(
      vsync: this,
      value: _position,
    );
    _controller.addListener(() {
      setState(() {
        _position = _controller.value.clamp(0.0, 1.0);
      });
    });
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  void _onHorizontalDragEnd(DragEndDetails details, double maxWidth) {
    final velocity = details.velocity.pixelsPerSecond.dx / maxWidth;
    final simulation = FrictionSimulation(
      _friction,           // friction coefficient (0-1, higher = more friction)
      _position,           // current position
      velocity,            // initial velocity
    );
    _controller.animateWith(simulation);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Friction Simulation')),
      body: Column(
        children: [
          Expanded(
            child: LayoutBuilder(
              builder: (context, constraints) {
                return GestureDetector(
                  onHorizontalDragUpdate: (details) {
                    setState(() {
                      _position =
                          (_position + details.delta.dx / constraints.maxWidth)
                              .clamp(0.0, 1.0);
                      _controller.value = _position;
                    });
                  },
                  onHorizontalDragEnd: (details) =>
                      _onHorizontalDragEnd(details, constraints.maxWidth),
                  child: Container(
                    color: Colors.transparent,
                    child: Stack(
                      children: [
                        // Track
                        Center(
                          child: Container(
                            height: 4,
                            margin: const EdgeInsets.symmetric(horizontal: 40),
                            decoration: BoxDecoration(
                              color: Colors.grey[300],
                              borderRadius: BorderRadius.circular(2),
                            ),
                          ),
                        ),
                        // Draggable ball
                        Positioned(
                          left: constraints.maxWidth * _position - 30,
                          top: constraints.maxHeight / 2 - 30,
                          child: GestureDetector(
                            child: Container(
                              width: 60,
                              height: 60,
                              decoration: const BoxDecoration(
                                shape: BoxShape.circle,
                                color: Colors.orange,
                              ),
                              child: const Icon(
                                Icons.drag_indicator,
                                color: Colors.white,
                              ),
                            ),
                          ),
                        ),
                        // Info overlay
                        Positioned(
                          top: 20,
                          left: 0,
                          right: 0,
                          child: Center(
                            child: Container(
                              padding: const EdgeInsets.symmetric(
                                horizontal: 16,
                                vertical: 8,
                              ),
                              decoration: BoxDecoration(
                                color: Colors.black.withOpacity(0.7),
                                borderRadius: BorderRadius.circular(8),
                              ),
                              child: Text(
                                'Position: ${(_position * 100).toStringAsFixed(1)}%',
                                style: const TextStyle(color: Colors.white),
                              ),
                            ),
                          ),
                        ),
                      ],
                    ),
                  ),
                );
              },
            ),
          ),
          Container(
            padding: const EdgeInsets.all(16),
            color: Colors.grey[100],
            child: Column(
              children: [
                const Text(
                  'Friction Coefficient',
                  style: TextStyle(fontWeight: FontWeight.bold),
                ),
                const SizedBox(height: 4),
                const Text(
                  '0 = no friction (slides forever)\n1 = maximum friction (stops immediately)',
                  textAlign: TextAlign.center,
                  style: TextStyle(fontSize: 12, color: Colors.grey),
                ),
                Slider(
                  value: _friction,
                  min: 0.01,
                  max: 1.0,
                  divisions: 99,
                  label: _friction.toStringAsFixed(2),
                  onChanged: (v) => setState(() => _friction = v),
                ),
                Text('Friction: ${_friction.toStringAsFixed(2)}'),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2764: Gravity Simulation พร้อม Bounce

```dart
// gravity_demo.dart
import 'package:flutter/material.dart';
import 'package:flutter/physics.dart';

class GravityDemo extends StatefulWidget {
  const GravityDemo({super.key});

  @override
  State<GravityDemo> createState() => _GravityDemoState();
}

class _GravityDemoState extends State<GravityDemo>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  double _ballY = 0.1; // normalized (0 = top, 1 = bottom)
  double _gravity = 9.8;
  double _bounciness = 0.7;
  bool _isSimulating = false;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController.unbounded(vsync: this, value: _ballY);
    _controller.addListener(() {
      setState(() {
        _ballY = _controller.value.clamp(0.0, 1.0);

        // Check if ball hits bottom and bounce
        if (_ballY >= 1.0 && _controller.velocity > 0) {
          final bounceVelocity = -_controller.velocity * _bounciness;
          if (bounceVelocity.abs() < 0.01) {
            _controller.stop();
            _isSimulating = false;
          } else {
            _controller.value = 1.0;
            _runGravitySimulation(
              startPosition: 1.0,
              startVelocity: bounceVelocity,
            );
          }
        }
      });
    });
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  void _runGravitySimulation({
    required double startPosition,
    required double startVelocity,
  }) {
    final simulation = GravitySimulation(
      _gravity * 0.5, // acceleration (scaled)
      startPosition,
      1.0,            // end position (floor)
      startVelocity,
    );

    _controller.animateWith(simulation);
    _isSimulating = true;
  }

  void _dropBall() {
    if (_isSimulating) {
      _controller.stop();
      _isSimulating = false;
    } else {
      setState(() => _ballY = 0.1);
      _controller.value = 0.1;
      _runGravitySimulation(startPosition: 0.1, startVelocity: 0.0);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Gravity Simulation')),
      body: Column(
        children: [
          Expanded(
            child: LayoutBuilder(
              builder: (context, constraints) {
                final ballSize = 60.0;
                final maxY = constraints.maxHeight - ballSize;
                final ballTop = _ballY * maxY;

                return GestureDetector(
                  onTapDown: (details) {
                    // Drop ball from tap position
                    setState(() {
                      _ballY = (details.localPosition.dy / constraints.maxHeight)
                          .clamp(0.0, 0.9);
                      _controller.value = _ballY;
                    });
                    _runGravitySimulation(
                      startPosition: _ballY,
                      startVelocity: 0.0,
                    );
                  },
                  child: Stack(
                    children: [
                      // Background grid
                      CustomPaint(
                        size: Size(constraints.maxWidth, constraints.maxHeight),
                        painter: _GridPainter(),
                      ),
                      // Floor
                      Positioned(
                        bottom: 0,
                        left: 0,
                        right: 0,
                        child: Container(
                          height: 4,
                          color: Colors.brown[700],
                        ),
                      ),
                      // Ball
                      Positioned(
                        top: ballTop,
                        left: constraints.maxWidth / 2 - ballSize / 2,
                        child: Container(
                          width: ballSize,
                          height: ballSize,
                          decoration: const BoxDecoration(
                            shape: BoxShape.circle,
                            gradient: RadialGradient(
                              center: Alignment(-0.3, -0.3),
                              colors: [Colors.red, Colors.red, Colors.brown],
                            ),
                          ),
                        ),
                      ),
                      // Velocity indicator
                      Positioned(
                        top: 16,
                        left: 16,
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            Text(
                              'Y: ${(_ballY * 100).toStringAsFixed(1)}%',
                              style: const TextStyle(
                                backgroundColor: Colors.white70,
                              ),
                            ),
                            Text(
                              'V: ${_controller.velocity.toStringAsFixed(2)}',
                              style: const TextStyle(
                                backgroundColor: Colors.white70,
                              ),
                            ),
                          ],
                        ),
                      ),
                    ],
                  ),
                );
              },
            ),
          ),
          Container(
            padding: const EdgeInsets.all(16),
            color: Colors.grey[100],
            child: Column(
              children: [
                _SliderRow(
                  label: 'Gravity',
                  value: _gravity,
                  min: 1.0,
                  max: 50.0,
                  onChanged: (v) => setState(() => _gravity = v),
                ),
                _SliderRow(
                  label: 'Bounce',
                  value: _bounciness,
                  min: 0.0,
                  max: 1.0,
                  onChanged: (v) => setState(() => _bounciness = v),
                ),
                const SizedBox(height: 8),
                ElevatedButton.icon(
                  onPressed: _dropBall,
                  icon: Icon(_isSimulating ? Icons.stop : Icons.play_arrow),
                  label: Text(_isSimulating ? 'Stop' : 'Drop Ball'),
                ),
                const SizedBox(height: 4),
                const Text(
                  'Tap anywhere to drop from that height',
                  style: TextStyle(fontSize: 12, color: Colors.grey),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

class _GridPainter extends CustomPainter {
  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = Colors.grey[200]!
      ..strokeWidth = 1;

    for (double y = 0; y < size.height; y += 50) {
      canvas.drawLine(Offset(0, y), Offset(size.width, y), paint);
    }
    for (double x = 0; x < size.width; x += 50) {
      canvas.drawLine(Offset(x, 0), Offset(x, size.height), paint);
    }
  }

  @override
  bool shouldRepaint(_GridPainter oldDelegate) => false;
}
```

---

## ขั้นตอนที่ 2765: Physics-Based Card Flip

```dart
// card_flip_demo.dart
import 'dart:math' as math;
import 'package:flutter/material.dart';
import 'package:flutter/physics.dart';

class CardFlipDemo extends StatefulWidget {
  const CardFlipDemo({super.key});

  @override
  State<CardFlipDemo> createState() => _CardFlipDemoState();
}

class _CardFlipDemoState extends State<CardFlipDemo>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  bool _showFront = true;
  double _dragStartX = 0;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      value: 0.0,
      lowerBound: 0.0,
      upperBound: 1.0,
    );
    _controller.addListener(() {
      // Flip the visual at the halfway point
      if (_controller.value >= 0.5 && _showFront) {
        setState(() => _showFront = false);
      } else if (_controller.value < 0.5 && !_showFront) {
        setState(() => _showFront = true);
      }
    });
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  void _onPanStart(DragStartDetails details) {
    _dragStartX = details.localPosition.dx;
  }

  void _onPanUpdate(DragUpdateDetails details, double width) {
    final dx = details.localPosition.dx - _dragStartX;
    final normalizedDx = dx / width;
    _controller.value = (_controller.value + normalizedDx * 0.01).clamp(0.0, 1.0);
    _dragStartX = details.localPosition.dx;
  }

  void _onPanEnd(DragEndDetails details) {
    final velocity = details.velocity.pixelsPerSecond.dx;

    // Snap to nearest face using spring
    final targetValue = velocity > 0
        ? (_controller.value < 0.5 ? 0.0 : 1.0)
        : (_controller.value > 0.5 ? 1.0 : 0.0);

    final simulation = SpringSimulation(
      const SpringDescription(mass: 1, stiffness: 200, damping: 20),
      _controller.value,
      targetValue,
      velocity / 1000,
    );

    _controller.animateWith(simulation);
  }

  void _flipCard() {
    final target = _controller.value < 0.5 ? 1.0 : 0.0;
    final simulation = SpringSimulation(
      const SpringDescription(mass: 1, stiffness: 150, damping: 15),
      _controller.value,
      target,
      0,
    );
    _controller.animateWith(simulation);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Card Flip (Physics-Based)')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Text(
              'Drag or tap to flip',
              style: TextStyle(color: Colors.grey),
            ),
            const SizedBox(height: 24),
            LayoutBuilder(
              builder: (context, constraints) {
                return GestureDetector(
                  onPanStart: _onPanStart,
                  onPanUpdate: (d) => _onPanUpdate(d, constraints.maxWidth),
                  onPanEnd: _onPanEnd,
                  onTap: _flipCard,
                  child: AnimatedBuilder(
                    animation: _controller,
                    builder: (context, _) {
                      // Calculate rotation angle
                      final angle = _controller.value * math.pi;
                      final isBack = angle > math.pi / 2;

                      return Transform(
                        transform: Matrix4.identity()
                          ..setEntry(3, 2, 0.001) // perspective
                          ..rotateY(isBack ? math.pi - angle : angle),
                        alignment: Alignment.center,
                        child: isBack
                            ? _CardBack()
                            : _CardFront(),
                      );
                    },
                  ),
                );
              },
            ),
            const SizedBox(height: 32),
            AnimatedBuilder(
              animation: _controller,
              builder: (context, _) => Text(
                _showFront ? '🎴 Front Side' : '🃏 Back Side',
                style: const TextStyle(fontSize: 20),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _CardFront extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Container(
      width: 220,
      height: 320,
      decoration: BoxDecoration(
        borderRadius: BorderRadius.circular(16),
        gradient: const LinearGradient(
          begin: Alignment.topLeft,
          end: Alignment.bottomRight,
          colors: [Color(0xFF1a237e), Color(0xFF283593)],
        ),
        boxShadow: [
          BoxShadow(
            color: Colors.black.withOpacity(0.3),
            blurRadius: 20,
            offset: const Offset(0, 10),
          ),
        ],
      ),
      child: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                const Text(
                  'A',
                  style: TextStyle(
                    color: Colors.white,
                    fontSize: 32,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                Container(
                  width: 40,
                  height: 40,
                  decoration: const BoxDecoration(
                    shape: BoxShape.circle,
                    color: Colors.white24,
                  ),
                  child: const Icon(Icons.credit_card, color: Colors.white),
                ),
              ],
            ),
            const Spacer(),
            const Text(
              '•••• •••• •••• 4242',
              style: TextStyle(
                color: Colors.white,
                fontSize: 18,
                letterSpacing: 2,
              ),
            ),
            const SizedBox(height: 12),
            const Text(
              'CARD HOLDER',
              style: TextStyle(color: Colors.white54, fontSize: 10),
            ),
            const Text(
              'John Doe',
              style: TextStyle(color: Colors.white, fontSize: 14),
            ),
          ],
        ),
      ),
    );
  }
}

class _CardBack extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Transform(
      transform: Matrix4.identity()..rotateY(math.pi),
      alignment: Alignment.center,
      child: Container(
        width: 220,
        height: 320,
        decoration: BoxDecoration(
          borderRadius: BorderRadius.circular(16),
          gradient: const LinearGradient(
            begin: Alignment.topLeft,
            end: Alignment.bottomRight,
            colors: [Color(0xFF880E4F), Color(0xFFAD1457)],
          ),
          boxShadow: [
            BoxShadow(
              color: Colors.black.withOpacity(0.3),
              blurRadius: 20,
              offset: const Offset(0, 10),
            ),
          ],
        ),
        child: Column(
          children: [
            const SizedBox(height: 40),
            Container(height: 50, color: Colors.black87),
            const SizedBox(height: 20),
            Padding(
              padding: const EdgeInsets.symmetric(horizontal: 20),
              child: Container(
                height: 40,
                decoration: BoxDecoration(
                  color: Colors.white,
                  borderRadius: BorderRadius.circular(4),
                ),
                alignment: Alignment.centerRight,
                padding: const EdgeInsets.only(right: 12),
                child: const Text(
                  '737',
                  style: TextStyle(
                    fontStyle: FontStyle.italic,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ),
            ),
            const SizedBox(height: 8),
            const Text(
              'CVV',
              style: TextStyle(color: Colors.white70, fontSize: 12),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2766: Card Toss / Swipe Physics

```dart
// card_toss_demo.dart
import 'package:flutter/material.dart';
import 'package:flutter/physics.dart';

class CardTossDemo extends StatefulWidget {
  const CardTossDemo({super.key});

  @override
  State<CardTossDemo> createState() => _CardTossDemoState();
}

class _CardTossDemoState extends State<CardTossDemo>
    with TickerProviderStateMixin {
  final List<_CardData> _cards = [
    _CardData(color: Colors.red, label: 'Card 1'),
    _CardData(color: Colors.blue, label: 'Card 2'),
    _CardData(color: Colors.green, label: 'Card 3'),
    _CardData(color: Colors.orange, label: 'Card 4'),
    _CardData(color: Colors.purple, label: 'Card 5'),
  ];

  int _currentIndex = 0;
  Offset _dragOffset = Offset.zero;
  bool _isDragging = false;
  final List<_TossedCard> _tossedCards = [];

  void _onDragUpdate(DragUpdateDetails details) {
    setState(() {
      _dragOffset += details.delta;
      _isDragging = true;
    });
  }

  void _onDragEnd(DragEndDetails details, Size screenSize) {
    final velocity = details.velocity.pixelsPerSecond;
    final speed = velocity.distance;

    // Determine if card should be tossed away
    final threshold = screenSize.width * 0.3;
    final shouldToss = _dragOffset.dx.abs() > threshold || speed > 800;

    if (shouldToss) {
      _tossCard(velocity, screenSize);
    } else {
      // Snap back with spring
      setState(() {
        _dragOffset = Offset.zero;
        _isDragging = false;
      });
    }
  }

  void _tossCard(Offset velocity, Size screenSize) {
    if (_currentIndex >= _cards.length) return;

    final card = _cards[_currentIndex];
    final direction = _dragOffset.dx > 0 ? 1.0 : -1.0;

    _tossedCards.add(_TossedCard(
      data: card,
      initialOffset: _dragOffset,
      velocity: velocity,
      direction: direction,
    ));

    setState(() {
      _currentIndex++;
      _dragOffset = Offset.zero;
      _isDragging = false;
    });

    // Clean up tossed cards after animation
    Future.delayed(const Duration(milliseconds: 600), () {
      if (mounted) setState(() => _tossedCards.clear());
    });
  }

  void _reset() {
    setState(() {
      _currentIndex = 0;
      _dragOffset = Offset.zero;
      _isDragging = false;
      _tossedCards.clear();
    });
  }

  double get _rotationAngle => (_dragOffset.dx / 400).clamp(-0.5, 0.5);

  @override
  Widget build(BuildContext context) {
    final screenSize = MediaQuery.of(context).size;

    return Scaffold(
      appBar: AppBar(title: const Text('Card Toss Physics')),
      body: Column(
        children: [
          Expanded(
            child: Stack(
              alignment: Alignment.center,
              children: [
                // Background cards
                for (int i = _currentIndex + 2; i >= _currentIndex + 1; i--)
                  if (i < _cards.length)
                    Transform.scale(
                      scale: 1.0 - (i - _currentIndex) * 0.05,
                      child: Transform.translate(
                        offset: Offset(0, (i - _currentIndex) * 8.0),
                        child: _buildCard(_cards[i], false),
                      ),
                    ),

                // Tossed cards (animating away)
                for (final tossed in _tossedCards)
                  _TossedCardWidget(tossedCard: tossed),

                // Current card (draggable)
                if (_currentIndex < _cards.length)
                  GestureDetector(
                    onPanUpdate: _onDragUpdate,
                    onPanEnd: (d) => _onDragEnd(d, screenSize),
                    child: Transform.translate(
                      offset: _dragOffset,
                      child: Transform.rotate(
                        angle: _rotationAngle,
                        child: Stack(
                          alignment: Alignment.topCenter,
                          children: [
                            _buildCard(_cards[_currentIndex], true),
                            if (_isDragging)
                              Positioned(
                                top: 20,
                                child: Container(
                                  padding: const EdgeInsets.symmetric(
                                    horizontal: 16,
                                    vertical: 8,
                                  ),
                                  decoration: BoxDecoration(
                                    color: _dragOffset.dx > 0
                                        ? Colors.green.withOpacity(0.8)
                                        : Colors.red.withOpacity(0.8),
                                    borderRadius: BorderRadius.circular(8),
                                  ),
                                  child: Text(
                                    _dragOffset.dx > 0 ? 'LIKE ❤️' : 'NOPE 👎',
                                    style: const TextStyle(
                                      color: Colors.white,
                                      fontWeight: FontWeight.bold,
                                    ),
                                  ),
                                ),
                              ),
                          ],
                        ),
                      ),
                    ),
                  )
                else
                  Center(
                    child: Column(
                      mainAxisAlignment: MainAxisAlignment.center,
                      children: [
                        const Text(
                          'No more cards!',
                          style: TextStyle(fontSize: 24),
                        ),
                        const SizedBox(height: 16),
                        ElevatedButton(
                          onPressed: _reset,
                          child: const Text('Reset'),
                        ),
                      ],
                    ),
                  ),
              ],
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(16),
            child: Text(
              '${_cards.length - _currentIndex} cards remaining — Swipe left/right',
              style: const TextStyle(color: Colors.grey),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildCard(_CardData card, bool isActive) {
    return Container(
      width: 280,
      height: 380,
      decoration: BoxDecoration(
        color: card.color,
        borderRadius: BorderRadius.circular(20),
        boxShadow: isActive
            ? [
                BoxShadow(
                  color: card.color.withOpacity(0.5),
                  blurRadius: 20,
                  offset: const Offset(0, 10),
                ),
              ]
            : [],
      ),
      child: Center(
        child: Text(
          card.label,
          style: const TextStyle(
            color: Colors.white,
            fontSize: 32,
            fontWeight: FontWeight.bold,
          ),
        ),
      ),
    );
  }
}

class _CardData {
  final Color color;
  final String label;
  _CardData({required this.color, required this.label});
}

class _TossedCard {
  final _CardData data;
  final Offset initialOffset;
  final Offset velocity;
  final double direction;
  _TossedCard({
    required this.data,
    required this.initialOffset,
    required this.velocity,
    required this.direction,
  });
}

class _TossedCardWidget extends StatefulWidget {
  final _TossedCard tossedCard;
  const _TossedCardWidget({required this.tossedCard});

  @override
  State<_TossedCardWidget> createState() => _TossedCardWidgetState();
}

class _TossedCardWidgetState extends State<_TossedCardWidget>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<Offset> _offsetAnimation;
  late Animation<double> _opacityAnimation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 500),
    );

    final endOffset = Offset(
      widget.tossedCard.direction * 600,
      -200,
    );

    _offsetAnimation = Tween<Offset>(
      begin: widget.tossedCard.initialOffset,
      end: endOffset,
    ).animate(CurvedAnimation(parent: _controller, curve: Curves.easeOut));

    _opacityAnimation = Tween<double>(begin: 1.0, end: 0.0).animate(
      CurvedAnimation(parent: _controller, curve: const Interval(0.5, 1.0)),
    );

    _controller.forward();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return Opacity(
          opacity: _opacityAnimation.value,
          child: Transform.translate(
            offset: _offsetAnimation.value,
            child: Container(
              width: 280,
              height: 380,
              decoration: BoxDecoration(
                color: widget.tossedCard.data.color,
                borderRadius: BorderRadius.circular(20),
              ),
              child: Center(
                child: Text(
                  widget.tossedCard.data.label,
                  style: const TextStyle(
                    color: Colors.white,
                    fontSize: 32,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ),
            ),
          ),
        );
      },
    );
  }
}
```

---

## ขั้นตอนที่ 2767: Custom Scroll Physics - Magnetic Snap

```dart
// custom_scroll_physics_demo.dart
import 'package:flutter/material.dart';
import 'package:flutter/physics.dart';

/// Custom scroll physics that snaps to item boundaries
class SnapScrollPhysics extends ScrollPhysics {
  final double itemExtent;

  const SnapScrollPhysics({
    required this.itemExtent,
    super.parent,
  });

  @override
  SnapScrollPhysics applyTo(ScrollPhysics? ancestor) {
    return SnapScrollPhysics(
      itemExtent: itemExtent,
      parent: buildParent(ancestor),
    );
  }

  double _getTargetPixels(
    ScrollPosition position,
    Tolerance tolerance,
    double velocity,
  ) {
    double page = position.pixels / itemExtent;

    if (velocity < -tolerance.velocity) {
      page -= 0.5;
    } else if (velocity > tolerance.velocity) {
      page += 0.5;
    }

    return page.roundToDouble() * itemExtent;
  }

  @override
  Simulation? createBallisticSimulation(
    ScrollMetrics position,
    double velocity,
  ) {
    if ((velocity <= 0.0 && position.pixels <= position.minScrollExtent) ||
        (velocity >= 0.0 && position.pixels >= position.maxScrollExtent)) {
      return super.createBallisticSimulation(position, velocity);
    }

    final tolerance = toleranceFor(position);
    final target = _getTargetPixels(
      position as ScrollPosition,
      tolerance,
      velocity,
    );

    if (target == position.pixels) return null;

    return SpringSimulation(
      const SpringDescription(mass: 0.5, stiffness: 100, damping: 15),
      position.pixels,
      target,
      velocity,
      tolerance: tolerance,
    );
  }

  @override
  bool get allowImplicitScrolling => false;
}

class CustomScrollPhysicsDemo extends StatelessWidget {
  const CustomScrollPhysicsDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return DefaultTabController(
      length: 3,
      child: Scaffold(
        appBar: AppBar(
          title: const Text('Custom Scroll Physics'),
          bottom: const TabBar(
            tabs: [
              Tab(text: 'Snap Scroll'),
              Tab(text: 'Bouncy Scroll'),
              Tab(text: 'Page Snap'),
            ],
          ),
        ),
        body: const TabBarView(
          children: [
            _SnapScrollDemo(),
            _BouncyScrollDemo(),
            _PageSnapDemo(),
          ],
        ),
      ),
    );
  }
}

class _SnapScrollDemo extends StatelessWidget {
  const _SnapScrollDemo();

  @override
  Widget build(BuildContext context) {
    const itemHeight = 120.0;

    return ListView.builder(
      physics: const SnapScrollPhysics(itemExtent: itemHeight),
      itemExtent: itemHeight,
      itemCount: 20,
      itemBuilder: (context, index) {
        return Container(
          height: itemHeight,
          margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
          decoration: BoxDecoration(
            color: Colors.primaries[index % Colors.primaries.length],
            borderRadius: BorderRadius.circular(12),
          ),
          child: Center(
            child: Text(
              'Item ${index + 1}',
              style: const TextStyle(
                color: Colors.white,
                fontSize: 20,
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
        );
      },
    );
  }
}

class _BouncyScrollDemo extends StatelessWidget {
  const _BouncyScrollDemo();

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      physics: const BouncingScrollPhysics(
        decelerationRate: ScrollDecelerationRate.fast,
      ),
      itemCount: 20,
      itemBuilder: (context, index) {
        return Card(
          margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
          child: ListTile(
            leading: CircleAvatar(
              backgroundColor: Colors.primaries[index % Colors.primaries.length],
              child: Text('${index + 1}'),
            ),
            title: Text('Bouncy Item ${index + 1}'),
            subtitle: const Text('Scroll past the edges!'),
          ),
        );
      },
    );
  }
}

class _PageSnapDemo extends StatelessWidget {
  const _PageSnapDemo();

  @override
  Widget build(BuildContext context) {
    return PageView.builder(
      physics: const _SmoothPageScrollPhysics(),
      itemCount: 5,
      itemBuilder: (context, index) {
        return Container(
          margin: const EdgeInsets.all(16),
          decoration: BoxDecoration(
            color: Colors.primaries[index * 3 % Colors.primaries.length],
            borderRadius: BorderRadius.circular(20),
          ),
          child: Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                Text(
                  'Page ${index + 1}',
                  style: const TextStyle(
                    color: Colors.white,
                    fontSize: 48,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                const SizedBox(height: 16),
                const Text(
                  'Swipe to next page',
                  style: TextStyle(color: Colors.white70, fontSize: 16),
                ),
              ],
            ),
          ),
        );
      },
    );
  }
}

class _SmoothPageScrollPhysics extends PageScrollPhysics {
  const _SmoothPageScrollPhysics({super.parent});

  @override
  _SmoothPageScrollPhysics applyTo(ScrollPhysics? ancestor) {
    return _SmoothPageScrollPhysics(parent: buildParent(ancestor));
  }

  @override
  SpringDescription get spring => const SpringDescription(
    mass: 0.8,
    stiffness: 100,
    damping: 12,
  );
}
```

---

## ขั้นตอนที่ 2768: Combining Physics Simulations

```dart
// combined_physics_demo.dart - Physics pinball-like demo
import 'dart:math' as math;
import 'package:flutter/material.dart';
import 'package:flutter/scheduler.dart';

class PinballBall {
  Offset position;
  Offset velocity;
  final double radius;

  PinballBall({
    required this.position,
    required this.velocity,
    this.radius = 20,
  });

  void update(double dt, Size bounds) {
    // Apply gravity
    velocity = Offset(velocity.dx, velocity.dy + 500 * dt);

    // Apply air friction
    velocity = velocity * 0.999;

    // Update position
    position = Offset(
      position.dx + velocity.dx * dt,
      position.dy + velocity.dy * dt,
    );

    // Bounce off walls
    if (position.dx - radius < 0) {
      position = Offset(radius, position.dy);
      velocity = Offset(-velocity.dx * 0.8, velocity.dy);
    }
    if (position.dx + radius > bounds.width) {
      position = Offset(bounds.width - radius, position.dy);
      velocity = Offset(-velocity.dx * 0.8, velocity.dy);
    }

    // Bounce off floor
    if (position.dy + radius > bounds.height) {
      position = Offset(position.dx, bounds.height - radius);
      velocity = Offset(velocity.dx, -velocity.dy * 0.7);
    }

    // Bounce off ceiling
    if (position.dy - radius < 0) {
      position = Offset(position.dx, radius);
      velocity = Offset(velocity.dx, -velocity.dy * 0.8);
    }
  }
}

class CombinedPhysicsDemo extends StatefulWidget {
  const CombinedPhysicsDemo({super.key});

  @override
  State<CombinedPhysicsDemo> createState() => _CombinedPhysicsDemoState();
}

class _CombinedPhysicsDemoState extends State<CombinedPhysicsDemo>
    with SingleTickerProviderStateMixin {
  late Ticker _ticker;
  late DateTime _lastTime;
  final List<PinballBall> _balls = [];
  Size _size = Size.zero;
  final math.Random _random = math.Random();

  @override
  void initState() {
    super.initState();
    _lastTime = DateTime.now();
    _ticker = createTicker(_tick)..start();
  }

  void _tick(Duration elapsed) {
    if (_size == Size.zero) return;

    final now = DateTime.now();
    final dt = now.difference(_lastTime).inMicroseconds / 1000000.0;
    _lastTime = now;

    setState(() {
      for (final ball in _balls) {
        ball.update(dt.clamp(0.0, 0.05), _size);
      }
    });
  }

  @override
  void dispose() {
    _ticker.dispose();
    super.dispose();
  }

  void _addBall(Offset position) {
    _balls.add(PinballBall(
      position: position,
      velocity: Offset(
        (_random.nextDouble() - 0.5) * 400,
        -300 - _random.nextDouble() * 200,
      ),
    ));
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Combined Physics Demo'),
        actions: [
          IconButton(
            icon: const Icon(Icons.clear),
            onPressed: () => setState(() => _balls.clear()),
          ),
        ],
      ),
      body: LayoutBuilder(
        builder: (context, constraints) {
          _size = Size(constraints.maxWidth, constraints.maxHeight);

          return GestureDetector(
            onTapDown: (details) => _addBall(details.localPosition),
            child: CustomPaint(
              size: _size,
              painter: _PhysicsPainter(balls: _balls),
              child: Container(
                alignment: Alignment.center,
                child: Text(
                  _balls.isEmpty ? 'Tap to add balls!' : '',
                  style: const TextStyle(
                    color: Colors.grey,
                    fontSize: 20,
                  ),
                ),
              ),
            ),
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          // Add 5 balls at random positions
          for (int i = 0; i < 5; i++) {
            _addBall(Offset(
              _random.nextDouble() * _size.width,
              _size.height * 0.3,
            ));
          }
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}

class _PhysicsPainter extends CustomPainter {
  final List<PinballBall> balls;

  _PhysicsPainter({required this.balls});

  @override
  void paint(Canvas canvas, Size size) {
    // Draw floor
    final floorPaint = Paint()
      ..color = Colors.brown[700]!
      ..strokeWidth = 4;
    canvas.drawLine(
      Offset(0, size.height),
      Offset(size.width, size.height),
      floorPaint,
    );

    // Draw balls
    for (final ball in balls) {
      // Shadow
      final shadowPaint = Paint()
        ..color = Colors.black.withOpacity(0.2)
        ..maskFilter = const MaskFilter.blur(BlurStyle.normal, 8);
      canvas.drawCircle(
        Offset(ball.position.dx + 4, ball.position.dy + 4),
        ball.radius,
        shadowPaint,
      );

      // Ball gradient
      final ballPaint = Paint()
        ..shader = RadialGradient(
          center: const Alignment(-0.3, -0.3),
          colors: [Colors.orange[300]!, Colors.deepOrange],
        ).createShader(
          Rect.fromCircle(center: ball.position, radius: ball.radius),
        );
      canvas.drawCircle(ball.position, ball.radius, ballPaint);

      // Speed indicator
      final speed = ball.velocity.distance;
      final indicatorPaint = Paint()
        ..color = Colors.white.withOpacity(0.6)
        ..style = PaintingStyle.fill;
      canvas.drawCircle(
        Offset(ball.position.dx - ball.radius * 0.3, ball.position.dy - ball.radius * 0.3),
        ball.radius * 0.15,
        indicatorPaint,
      );
    }
  }

  @override
  bool shouldRepaint(_PhysicsPainter oldDelegate) => true;
}
```

---

## ขั้นตอนที่ 2769: ประกอบทุก Physics Demos เป็น App เดียว

```dart
// full_physics_app.dart - Complete runnable app
import 'package:flutter/material.dart';

void main() {
  runApp(const FullPhysicsApp());
}

class FullPhysicsApp extends StatelessWidget {
  const FullPhysicsApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Physics',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      // All the demos from above would be registered here
      routes: {
        '/': (context) => const PhysicsDemoHome(),
        '/spring': (context) => const SpringDemo(),
        '/friction': (context) => const FrictionDemo(),
        '/gravity': (context) => const GravityDemo(),
        '/card-flip': (context) => const CardFlipDemo(),
        '/card-toss': (context) => const CardTossDemo(),
        '/scroll': (context) => const CustomScrollPhysicsDemo(),
        '/combined': (context) => const CombinedPhysicsDemo(),
      },
    );
  }
}

// Re-export all needed for convenience
// (SpringDemo, FrictionDemo, GravityDemo, CardFlipDemo,
//  CardTossDemo, CustomScrollPhysicsDemo, CombinedPhysicsDemo
//  are defined in the sections above)
```

---

**← [Part 71](part-71-micro-frontend-architecture.md)**
**ต่อไป: [Part 73 →](part-73-state-management-comparison-project.md)**
