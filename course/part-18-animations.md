# Part 18: Flutter Animations
## ขั้นตอนที่ 601-640

---

## 🎯 เป้าหมายของ Part นี้

- Implicit Animations
- Explicit Animations (AnimationController)
- Hero Animations
- Custom Animations
- Animated Widgets
- Lottie Animations

---

## ขั้นตอนที่ 601: Implicit Animations

```dart
import 'package:flutter/material.dart';

// ─── AnimatedContainer ───
class AnimatedBoxDemo extends StatefulWidget {
  const AnimatedBoxDemo({super.key});
  @override
  State<AnimatedBoxDemo> createState() => _AnimatedBoxDemoState();
}

class _AnimatedBoxDemoState extends State<AnimatedBoxDemo> {
  bool _isExpanded = false;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => setState(() => _isExpanded = !_isExpanded),
      child: AnimatedContainer(
        duration: const Duration(milliseconds: 400),
        curve: Curves.easeInOut,
        width: _isExpanded ? 300 : 150,
        height: _isExpanded ? 300 : 150,
        decoration: BoxDecoration(
          color: _isExpanded ? Colors.deepPurple : Colors.blue,
          borderRadius: BorderRadius.circular(_isExpanded ? 40 : 8),
          boxShadow: [
            BoxShadow(
              color: Colors.black26,
              blurRadius: _isExpanded ? 20 : 4,
              offset: const Offset(0, 4),
            ),
          ],
        ),
        child: Center(
          child: AnimatedDefaultTextStyle(
            duration: const Duration(milliseconds: 400),
            style: TextStyle(
              fontSize: _isExpanded ? 24 : 14,
              color: Colors.white,
              fontWeight: FontWeight.bold,
            ),
            child: const Text('แตะเพื่อขยาย'),
          ),
        ),
      ),
    );
  }
}

// ─── AnimatedOpacity, AnimatedPositioned, AnimatedCrossFade ───
class AnimatedWidgetsDemo extends StatefulWidget {
  const AnimatedWidgetsDemo({super.key});
  @override
  State<AnimatedWidgetsDemo> createState() => _AnimatedWidgetsDemoState();
}

class _AnimatedWidgetsDemoState extends State<AnimatedWidgetsDemo> {
  bool _visible = true;
  bool _showFirst = true;
  double _offset = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // AnimatedOpacity
        AnimatedOpacity(
          duration: const Duration(milliseconds: 500),
          opacity: _visible ? 1.0 : 0.0,
          child: const FlutterLogo(size: 80),
        ),
        ElevatedButton(
          onPressed: () => setState(() => _visible = !_visible),
          child: Text(_visible ? 'ซ่อน' : 'แสดง'),
        ),
        const SizedBox(height: 20),

        // AnimatedCrossFade
        AnimatedCrossFade(
          duration: const Duration(milliseconds: 400),
          crossFadeState: _showFirst
              ? CrossFadeState.showFirst
              : CrossFadeState.showSecond,
          firstChild: Container(
            padding: const EdgeInsets.all(16),
            color: Colors.blue.shade100,
            child: const Text('เนื้อหาแรก', style: TextStyle(fontSize: 18)),
          ),
          secondChild: Container(
            padding: const EdgeInsets.all(16),
            color: Colors.green.shade100,
            child: const Text('เนื้อหาที่สอง\nมีหลายบรรทัด', style: TextStyle(fontSize: 18)),
          ),
        ),
        ElevatedButton(
          onPressed: () => setState(() => _showFirst = !_showFirst),
          child: const Text('สลับ'),
        ),

        // AnimatedSlide
        AnimatedSlide(
          duration: const Duration(milliseconds: 500),
          offset: Offset(_offset, 0),
          child: const Card(
            child: Padding(padding: EdgeInsets.all(16), child: Text('Slide me!')),
          ),
        ),
        Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () => setState(() => _offset -= 0.3),
              child: const Icon(Icons.arrow_left),
            ),
            ElevatedButton(
              onPressed: () => setState(() => _offset += 0.3),
              child: const Icon(Icons.arrow_right),
            ),
          ],
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 602: AnimationController (Explicit Animations)

```dart
import 'package:flutter/material.dart';
import 'dart:math' as math;

class RotatingLogo extends StatefulWidget {
  const RotatingLogo({super.key});
  @override
  State<RotatingLogo> createState() => _RotatingLogoState();
}

class _RotatingLogoState extends State<RotatingLogo>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _rotation;
  late Animation<double> _scale;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 2),
    );

    _rotation = Tween<double>(begin: 0, end: 2 * math.pi).animate(
      CurvedAnimation(parent: _controller, curve: Curves.linear),
    );

    _scale = TweenSequence<double>([
      TweenSequenceItem(tween: Tween(begin: 1.0, end: 1.5), weight: 50),
      TweenSequenceItem(tween: Tween(begin: 1.5, end: 1.0), weight: 50),
    ]).animate(_controller);

    _controller.repeat();
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
        return Transform.scale(
          scale: _scale.value,
          child: Transform.rotate(
            angle: _rotation.value,
            child: child,
          ),
        );
      },
      child: const FlutterLogo(size: 100),
    );
  }
}

// ─── Custom Bounce Animation ───
class BounceButton extends StatefulWidget {
  final VoidCallback? onPressed;
  final Widget child;

  const BounceButton({super.key, this.onPressed, required this.child});

  @override
  State<BounceButton> createState() => _BounceButtonState();
}

class _BounceButtonState extends State<BounceButton>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _scale;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 150),
    );

    _scale = Tween<double>(begin: 1.0, end: 0.9).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  void _onTapDown(TapDownDetails _) => _controller.forward();
  void _onTapUp(TapUpDetails _) {
    _controller.reverse();
    widget.onPressed?.call();
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTapDown: _onTapDown,
      onTapUp: _onTapUp,
      onTapCancel: () => _controller.reverse(),
      child: ScaleTransition(
        scale: _scale,
        child: widget.child,
      ),
    );
  }
}

// ─── Progress Ring Animation ───
class AnimatedProgressRing extends StatefulWidget {
  final double progress;
  final Color color;
  final double size;

  const AnimatedProgressRing({
    super.key,
    required this.progress,
    this.color = Colors.blue,
    this.size = 120,
  });

  @override
  State<AnimatedProgressRing> createState() => _AnimatedProgressRingState();
}

class _AnimatedProgressRingState extends State<AnimatedProgressRing>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 800),
    );
    _animation = Tween<double>(begin: 0, end: widget.progress)
        .animate(CurvedAnimation(parent: _controller, curve: Curves.easeOut));
    _controller.forward();
  }

  @override
  void didUpdateWidget(AnimatedProgressRing oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (oldWidget.progress != widget.progress) {
      _animation = Tween<double>(
        begin: _animation.value,
        end: widget.progress,
      ).animate(CurvedAnimation(parent: _controller, curve: Curves.easeOut));
      _controller
        ..reset()
        ..forward();
    }
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (_, __) => SizedBox(
        width: widget.size,
        height: widget.size,
        child: Stack(
          alignment: Alignment.center,
          children: [
            CustomPaint(
              size: Size.square(widget.size),
              painter: _RingPainter(
                progress: _animation.value,
                color: widget.color,
              ),
            ),
            Text(
              '${(_animation.value * 100).toInt()}%',
              style: TextStyle(
                fontSize: widget.size * 0.2,
                fontWeight: FontWeight.bold,
                color: widget.color,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _RingPainter extends CustomPainter {
  final double progress;
  final Color color;

  _RingPainter({required this.progress, required this.color});

  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final radius = size.width / 2 - 8;
    final paint = Paint()
      ..style = PaintingStyle.stroke
      ..strokeWidth = 8
      ..strokeCap = StrokeCap.round;

    // Background track
    canvas.drawCircle(center, radius, paint..color = Colors.grey.shade200);

    // Progress arc
    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      -math.pi / 2,                    // start from top
      2 * math.pi * progress,           // sweep angle
      false,
      paint..color = color,
    );
  }

  @override
  bool shouldRepaint(_RingPainter old) => old.progress != progress;
}
```

---

## ขั้นตอนที่ 603: Hero Animation

```dart
import 'package:flutter/material.dart';

// ─── Product List ─── (Screen 1)
class ProductListScreen extends StatelessWidget {
  final List<Map<String, dynamic>> products = const [
    {'id': '1', 'name': 'iPhone 15', 'price': 39900, 'color': Color(0xFF1A73E8)},
    {'id': '2', 'name': 'MacBook Pro', 'price': 79900, 'color': Color(0xFF34A853)},
    {'id': '3', 'name': 'AirPods Pro', 'price': 9900, 'color': Color(0xFFEA4335)},
  ];

  const ProductListScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('สินค้า')),
      body: GridView.builder(
        padding: const EdgeInsets.all(16),
        gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
          crossAxisCount: 2,
          childAspectRatio: 0.8,
          crossAxisSpacing: 12,
          mainAxisSpacing: 12,
        ),
        itemCount: products.length,
        itemBuilder: (context, i) {
          final p = products[i];
          return GestureDetector(
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(
                builder: (_) => ProductDetailScreen(product: p),
              ),
            ),
            child: Card(
              child: Column(
                children: [
                  // Hero tag must be unique
                  Hero(
                    tag: 'product-${p['id']}',
                    child: Container(
                      height: 120,
                      color: p['color'] as Color,
                      child: const Center(
                        child: Icon(Icons.phone_iphone, size: 60, color: Colors.white),
                      ),
                    ),
                  ),
                  Padding(
                    padding: const EdgeInsets.all(8),
                    child: Text(
                      p['name'] as String,
                      style: const TextStyle(fontWeight: FontWeight.bold),
                    ),
                  ),
                  Text('฿${p['price']}'),
                ],
              ),
            ),
          );
        },
      ),
    );
  }
}

// ─── Product Detail ─── (Screen 2)
class ProductDetailScreen extends StatelessWidget {
  final Map<String, dynamic> product;

  const ProductDetailScreen({super.key, required this.product});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(product['name'] as String)),
      body: Column(
        children: [
          // Same Hero tag → จะ animate เมื่อ navigate
          Hero(
            tag: 'product-${product['id']}',
            child: Container(
              width: double.infinity,
              height: 300,
              color: product['color'] as Color,
              child: const Center(
                child: Icon(Icons.phone_iphone, size: 150, color: Colors.white),
              ),
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  product['name'] as String,
                  style: const TextStyle(fontSize: 28, fontWeight: FontWeight.bold),
                ),
                const SizedBox(height: 8),
                Text(
                  '฿${product['price']}',
                  style: const TextStyle(fontSize: 22, color: Colors.green),
                ),
                const SizedBox(height: 16),
                ElevatedButton(
                  onPressed: () {},
                  child: const Text('เพิ่มในตะกร้า'),
                ),
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

## ขั้นตอนที่ 604: Custom Page Transitions

```dart
import 'package:flutter/material.dart';

// ─── Slide Transition ───
class SlidePageRoute<T> extends PageRouteBuilder<T> {
  final Widget page;
  final AxisDirection direction;

  SlidePageRoute({required this.page, this.direction = AxisDirection.left})
      : super(
          pageBuilder: (_, __, ___) => page,
          transitionsBuilder: (_, animation, __, child) {
            Offset begin;
            switch (direction) {
              case AxisDirection.left:
                begin = const Offset(1.0, 0.0);
                break;
              case AxisDirection.right:
                begin = const Offset(-1.0, 0.0);
                break;
              case AxisDirection.up:
                begin = const Offset(0.0, 1.0);
                break;
              case AxisDirection.down:
                begin = const Offset(0.0, -1.0);
                break;
            }

            return SlideTransition(
              position: Tween(begin: begin, end: Offset.zero)
                  .animate(CurvedAnimation(parent: animation, curve: Curves.easeOut)),
              child: child,
            );
          },
          transitionDuration: const Duration(milliseconds: 300),
        );
}

// ─── Fade Scale Transition ───
class FadeScaleRoute<T> extends PageRouteBuilder<T> {
  final Widget page;

  FadeScaleRoute({required this.page})
      : super(
          pageBuilder: (_, __, ___) => page,
          transitionsBuilder: (_, animation, __, child) {
            return FadeTransition(
              opacity: animation,
              child: ScaleTransition(
                scale: Tween(begin: 0.8, end: 1.0).animate(
                  CurvedAnimation(parent: animation, curve: Curves.easeOut),
                ),
                child: child,
              ),
            );
          },
        );
}

// ─── Flip Transition ───
class FlipPageRoute<T> extends PageRouteBuilder<T> {
  final Widget page;

  FlipPageRoute({required this.page})
      : super(
          pageBuilder: (_, __, ___) => page,
          transitionsBuilder: (_, animation, __, child) {
            final pi = 3.14159;
            return AnimatedBuilder(
              animation: animation,
              builder: (_, __) {
                double rotationY = (1 - animation.value) * pi;
                bool showFront = animation.value < 0.5;

                return Transform(
                  transform: Matrix4.identity()
                    ..setEntry(3, 2, 0.001)
                    ..rotateY(showFront ? rotationY : rotationY - pi),
                  alignment: Alignment.center,
                  child: showFront ? const SizedBox() : child,
                );
              },
            );
          },
          transitionDuration: const Duration(milliseconds: 600),
        );
}

// ─── Usage ───
class TransitionDemo extends StatelessWidget {
  const TransitionDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        ElevatedButton(
          onPressed: () => Navigator.push(
            context,
            SlidePageRoute(page: const _NextPage()),
          ),
          child: const Text('Slide Transition'),
        ),
        ElevatedButton(
          onPressed: () => Navigator.push(
            context,
            FadeScaleRoute(page: const _NextPage()),
          ),
          child: const Text('Fade Scale Transition'),
        ),
      ],
    );
  }
}

class _NextPage extends StatelessWidget {
  const _NextPage();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('หน้าถัดไป')),
      body: const Center(child: Text('หน้าถัดไป!')),
    );
  }
}
```

---

## ขั้นตอนที่ 605: Staggered Animations

```dart
import 'package:flutter/material.dart';

class StaggeredListAnimation extends StatefulWidget {
  final List<String> items;

  const StaggeredListAnimation({super.key, required this.items});

  @override
  State<StaggeredListAnimation> createState() => _StaggeredListAnimationState();
}

class _StaggeredListAnimationState extends State<StaggeredListAnimation>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: Duration(milliseconds: 100 * widget.items.length + 400),
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
    return ListView.builder(
      itemCount: widget.items.length,
      itemBuilder: (context, i) {
        // Each item animates with staggered delay
        double start = i * 0.1;
        double end = start + 0.4;

        Animation<double> opacity = Tween(begin: 0.0, end: 1.0).animate(
          CurvedAnimation(
            parent: _controller,
            curve: Interval(start, end, curve: Curves.easeOut),
          ),
        );

        Animation<Offset> slide = Tween(
          begin: const Offset(-0.3, 0),
          end: Offset.zero,
        ).animate(
          CurvedAnimation(
            parent: _controller,
            curve: Interval(start, end, curve: Curves.easeOut),
          ),
        );

        return AnimatedBuilder(
          animation: _controller,
          builder: (_, child) => FadeTransition(
            opacity: opacity,
            child: SlideTransition(position: slide, child: child),
          ),
          child: ListTile(
            leading: CircleAvatar(child: Text('${i + 1}')),
            title: Text(widget.items[i]),
          ),
        );
      },
    );
  }
}

// ─── Shimmer Loading ───
class ShimmerLoading extends StatefulWidget {
  final double width;
  final double height;
  final BorderRadius? borderRadius;

  const ShimmerLoading({
    super.key,
    required this.width,
    required this.height,
    this.borderRadius,
  });

  @override
  State<ShimmerLoading> createState() => _ShimmerLoadingState();
}

class _ShimmerLoadingState extends State<ShimmerLoading>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _shimmer;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 1200),
    )..repeat();

    _shimmer = Tween<double>(begin: -1.0, end: 2.0).animate(
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
    return AnimatedBuilder(
      animation: _shimmer,
      builder: (_, __) => Container(
        width: widget.width,
        height: widget.height,
        decoration: BoxDecoration(
          borderRadius: widget.borderRadius,
          gradient: LinearGradient(
            begin: Alignment.centerLeft,
            end: Alignment.centerRight,
            stops: [
              (_shimmer.value - 0.3).clamp(0, 1),
              _shimmer.value.clamp(0, 1),
              (_shimmer.value + 0.3).clamp(0, 1),
            ],
            colors: const [
              Color(0xFFEBEBF4),
              Color(0xFFF4F4FB),
              Color(0xFFEBEBF4),
            ],
          ),
        ),
      ),
    );
  }
}

// ─── Shimmer Card ───
class ShimmerCard extends StatelessWidget {
  const ShimmerCard({super.key});

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.all(16),
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                const ShimmerLoading(
                  width: 50,
                  height: 50,
                  borderRadius: BorderRadius.all(Radius.circular(25)),
                ),
                const SizedBox(width: 12),
                Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    ShimmerLoading(
                      width: 120,
                      height: 14,
                      borderRadius: BorderRadius.circular(7),
                    ),
                    const SizedBox(height: 6),
                    ShimmerLoading(
                      width: 80,
                      height: 12,
                      borderRadius: BorderRadius.circular(6),
                    ),
                  ],
                ),
              ],
            ),
            const SizedBox(height: 16),
            ShimmerLoading(
              width: double.infinity,
              height: 120,
              borderRadius: BorderRadius.circular(8),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 606: Lottie Animations

```dart
// pubspec.yaml:
// dependencies:
//   lottie: ^2.7.0

import 'package:flutter/material.dart';
import 'package:lottie/lottie.dart';

// ─── Lottie Basic Usage ───
class LottieDemo extends StatefulWidget {
  const LottieDemo({super.key});
  @override
  State<LottieDemo> createState() => _LottieDemoState();
}

class _LottieDemoState extends State<LottieDemo>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this);
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // จาก assets/animations/loading.json
        Lottie.asset(
          'assets/animations/loading.json',
          width: 200,
          height: 200,
          repeat: true,
        ),

        // จาก network URL
        Lottie.network(
          'https://assets.lottiefiles.com/packages/lf20_success.json',
          width: 150,
          height: 150,
          repeat: false,
        ),

        // Control animation manually
        Lottie.asset(
          'assets/animations/like.json',
          controller: _controller,
          onLoaded: (composition) {
            _controller.duration = composition.duration;
          },
          width: 100,
          height: 100,
        ),
        Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () => _controller.forward(from: 0),
              child: const Text('Play'),
            ),
            ElevatedButton(
              onPressed: () => _controller.stop(),
              child: const Text('Stop'),
            ),
          ],
        ),
      ],
    );
  }
}

// ─── Success Screen with Lottie ───
class SuccessScreen extends StatelessWidget {
  final String message;
  final VoidCallback onContinue;

  const SuccessScreen({
    super.key,
    required this.message,
    required this.onContinue,
  });

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Lottie.asset(
              'assets/animations/success.json',
              width: 200,
              height: 200,
              repeat: false,
            ),
            const SizedBox(height: 24),
            Text(
              message,
              style: const TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
              textAlign: TextAlign.center,
            ),
            const SizedBox(height: 32),
            ElevatedButton(
              onPressed: onContinue,
              style: ElevatedButton.styleFrom(
                padding: const EdgeInsets.symmetric(horizontal: 48, vertical: 16),
              ),
              child: const Text('ดำเนินการต่อ'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 607: Animation Project - Onboarding Screen

```dart
import 'package:flutter/material.dart';
import 'dart:math' as math;

class OnboardingScreen extends StatefulWidget {
  final VoidCallback onComplete;

  const OnboardingScreen({super.key, required this.onComplete});

  @override
  State<OnboardingScreen> createState() => _OnboardingScreenState();
}

class _OnboardingScreenState extends State<OnboardingScreen>
    with TickerProviderStateMixin {
  late PageController _pageController;
  late AnimationController _progressController;
  int _currentPage = 0;

  final List<_OnboardingPage> _pages = const [
    _OnboardingPage(
      icon: Icons.rocket_launch,
      title: 'เริ่มต้นง่าย',
      description: 'ติดตั้งและเริ่มใช้งานได้ทันที ไม่ต้องมีความรู้เดิม',
      color: Color(0xFF6C63FF),
    ),
    _OnboardingPage(
      icon: Icons.speed,
      title: 'รวดเร็วทันใจ',
      description: 'ประสิทธิภาพสูง ตอบสนองทันทีทุกการกระทำ',
      color: Color(0xFF3ECFCF),
    ),
    _OnboardingPage(
      icon: Icons.security,
      title: 'ปลอดภัยมั่นใจ',
      description: 'ข้อมูลของคุณถูกเข้ารหัสและปลอดภัยตลอดเวลา',
      color: Color(0xFFFF6584),
    ),
  ];

  @override
  void initState() {
    super.initState();
    _pageController = PageController();
    _progressController = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 300),
      value: 0,
    );
  }

  @override
  void dispose() {
    _pageController.dispose();
    _progressController.dispose();
    super.dispose();
  }

  void _nextPage() {
    if (_currentPage < _pages.length - 1) {
      _pageController.nextPage(
        duration: const Duration(milliseconds: 400),
        curve: Curves.easeOut,
      );
    } else {
      widget.onComplete();
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          Expanded(
            child: PageView.builder(
              controller: _pageController,
              onPageChanged: (i) {
                setState(() => _currentPage = i);
                _progressController.animateTo(
                  (i + 1) / _pages.length,
                  duration: const Duration(milliseconds: 300),
                );
              },
              itemCount: _pages.length,
              itemBuilder: (context, i) {
                return _OnboardingPageView(page: _pages[i]);
              },
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(24),
            child: Column(
              children: [
                // Page indicators
                Row(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: List.generate(
                    _pages.length,
                    (i) => AnimatedContainer(
                      duration: const Duration(milliseconds: 300),
                      margin: const EdgeInsets.symmetric(horizontal: 4),
                      width: i == _currentPage ? 24 : 8,
                      height: 8,
                      decoration: BoxDecoration(
                        borderRadius: BorderRadius.circular(4),
                        color: i == _currentPage
                            ? _pages[i].color
                            : Colors.grey.shade300,
                      ),
                    ),
                  ),
                ),
                const SizedBox(height: 32),
                // Next / Get Started button
                SizedBox(
                  width: double.infinity,
                  child: AnimatedContainer(
                    duration: const Duration(milliseconds: 300),
                    child: ElevatedButton(
                      onPressed: _nextPage,
                      style: ElevatedButton.styleFrom(
                        backgroundColor: _pages[_currentPage].color,
                        padding: const EdgeInsets.symmetric(vertical: 16),
                        shape: RoundedRectangleBorder(
                          borderRadius: BorderRadius.circular(30),
                        ),
                      ),
                      child: AnimatedSwitcher(
                        duration: const Duration(milliseconds: 200),
                        child: Text(
                          _currentPage < _pages.length - 1 ? 'ถัดไป' : 'เริ่มเลย!',
                          key: ValueKey(_currentPage),
                          style: const TextStyle(fontSize: 18, color: Colors.white),
                        ),
                      ),
                    ),
                  ),
                ),
                if (_currentPage < _pages.length - 1)
                  TextButton(
                    onPressed: widget.onComplete,
                    child: const Text('ข้าม'),
                  ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

class _OnboardingPage {
  final IconData icon;
  final String title;
  final String description;
  final Color color;

  const _OnboardingPage({
    required this.icon,
    required this.title,
    required this.description,
    required this.color,
  });
}

class _OnboardingPageView extends StatefulWidget {
  final _OnboardingPage page;

  const _OnboardingPageView({required this.page});

  @override
  State<_OnboardingPageView> createState() => _OnboardingPageViewState();
}

class _OnboardingPageViewState extends State<_OnboardingPageView>
    with TickerProviderStateMixin {
  late AnimationController _floatController;
  late Animation<double> _floatAnimation;

  @override
  void initState() {
    super.initState();
    _floatController = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 2),
    )..repeat(reverse: true);

    _floatAnimation = Tween<double>(begin: -10, end: 10).animate(
      CurvedAnimation(parent: _floatController, curve: Curves.easeInOut),
    );
  }

  @override
  void dispose() {
    _floatController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        // Floating icon
        AnimatedBuilder(
          animation: _floatAnimation,
          builder: (_, child) => Transform.translate(
            offset: Offset(0, _floatAnimation.value),
            child: child,
          ),
          child: Container(
            width: 160,
            height: 160,
            decoration: BoxDecoration(
              shape: BoxShape.circle,
              color: widget.page.color.withOpacity(0.15),
            ),
            child: Icon(widget.page.icon, size: 80, color: widget.page.color),
          ),
        ),
        const SizedBox(height: 48),
        Text(
          widget.page.title,
          style: const TextStyle(fontSize: 28, fontWeight: FontWeight.bold),
        ),
        const SizedBox(height: 16),
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 32),
          child: Text(
            widget.page.description,
            textAlign: TextAlign.center,
            style: TextStyle(fontSize: 16, color: Colors.grey.shade600),
          ),
        ),
      ],
    );
  }
}
```

---

**← [Part 17 - Testing](part-17-testing.md)**

**ต่อไป: [Part 19 - Riverpod →](part-19-riverpod.md)**
