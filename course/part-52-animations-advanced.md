# Part 52: Advanced Animations
## ขั้นตอนที่ 1961-2000

## 🎯 เป้าหมายของ Part นี้
- ทำ Hero animations พร้อม custom flight path
- สร้าง Staggered animations ด้วย Interval
- ใช้ Lottie animations ผ่าน lottie package
- TweenAnimationBuilder สำหรับ implicit animations
- AnimatedSwitcher กับ custom transitions
- Shared axis transitions จาก animations package

---

## ขั้นตอนที่ 1961: Hero Animation กับ Custom Flight Path

```dart
// lib/animations/hero_demo.dart
import 'package:flutter/material.dart';

class HeroDemoListPage extends StatelessWidget {
  const HeroDemoListPage({super.key});

  static const _items = [
    _HeroItem(id: '1', title: 'Mountain Sunrise', color: Color(0xFF1565C0), icon: Icons.landscape),
    _HeroItem(id: '2', title: 'Ocean Waves', color: Color(0xFF00695C), icon: Icons.water),
    _HeroItem(id: '3', title: 'Forest Path', color: Color(0xFF2E7D32), icon: Icons.park),
    _HeroItem(id: '4', title: 'Desert Dunes', color: Color(0xFFE65100), icon: Icons.terrain),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Hero Animations')),
      body: ListView.builder(
        padding: const EdgeInsets.all(16),
        itemCount: _items.length,
        itemBuilder: (ctx, i) {
          final item = _items[i];
          return GestureDetector(
            onTap: () => Navigator.push(
              ctx,
              PageRouteBuilder(
                transitionDuration: const Duration(milliseconds: 500),
                pageBuilder: (_, __, ___) => HeroDemoDetailPage(item: item),
              ),
            ),
            child: Padding(
              padding: const EdgeInsets.only(bottom: 12),
              child: Hero(
                tag: 'hero_${item.id}',
                flightShuttleBuilder: _customFlightShuttle,
                child: _ItemCard(item: item, isDetail: false),
              ),
            ),
          );
        },
      ),
    );
  }

  Widget _customFlightShuttle(
    BuildContext flightContext,
    Animation<double> animation,
    HeroFlightDirection direction,
    BuildContext fromHeroContext,
    BuildContext toHeroContext,
  ) {
    final tween = CurvedAnimation(parent: animation, curve: Curves.easeInOutCubic);
    return AnimatedBuilder(
      animation: tween,
      builder: (ctx, child) {
        return Material(
          color: Colors.transparent,
          child: ScaleTransition(
            scale: Tween<double>(begin: 1.0, end: 1.05).animate(
              CurvedAnimation(
                parent: animation,
                curve: const Interval(0.3, 0.7, curve: Curves.easeOut),
              ),
            ),
            child: child,
          ),
        );
      },
      child: toHeroContext.widget,
    );
  }
}

class HeroDemoDetailPage extends StatelessWidget {
  const HeroDemoDetailPage({super.key, required this.item});
  final _HeroItem item;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.grey.shade100,
      body: CustomScrollView(
        slivers: [
          SliverAppBar(
            expandedHeight: 300,
            pinned: true,
            flexibleSpace: FlexibleSpaceBar(
              title: Text(item.title),
              background: Hero(
                tag: 'hero_${item.id}',
                child: _ItemCard(item: item, isDetail: true),
              ),
            ),
          ),
          SliverPadding(
            padding: const EdgeInsets.all(16),
            sliver: SliverList(
              delegate: SliverChildListDelegate([
                _buildInfoCard('Description',
                    'A beautiful landscape that captures the essence of nature at its finest. '
                    'This scene showcases the natural world in all its glory.'),
                const SizedBox(height: 12),
                _buildInfoCard('Details', 'Location: Nature Reserve\nBest time: Early morning\nDifficulty: Moderate'),
              ]),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildInfoCard(String title, String body) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(title, style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            Text(body, style: const TextStyle(fontSize: 14, color: Colors.black54)),
          ],
        ),
      ),
    );
  }
}

class _ItemCard extends StatelessWidget {
  const _ItemCard({required this.item, required this.isDetail});
  final _HeroItem item;
  final bool isDetail;

  @override
  Widget build(BuildContext context) {
    return Container(
      height: isDetail ? double.infinity : 80,
      decoration: BoxDecoration(
        color: item.color,
        borderRadius: isDetail ? BorderRadius.zero : BorderRadius.circular(16),
      ),
      padding: const EdgeInsets.all(16),
      child: Row(
        children: [
          Icon(item.icon, color: Colors.white, size: isDetail ? 48 : 32),
          const SizedBox(width: 16),
          Expanded(
            child: Text(
              item.title,
              style: TextStyle(
                color: Colors.white,
                fontSize: isDetail ? 24 : 16,
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
        ],
      ),
    );
  }
}

class _HeroItem {
  const _HeroItem({
    required this.id,
    required this.title,
    required this.color,
    required this.icon,
  });
  final String id;
  final String title;
  final Color color;
  final IconData icon;
}
```

---

## ขั้นตอนที่ 1962: Staggered Animations

```dart
// lib/animations/staggered_animation.dart
import 'package:flutter/material.dart';

class StaggeredListItem {
  const StaggeredListItem({required this.title, required this.subtitle, required this.icon});
  final String title;
  final String subtitle;
  final IconData icon;
}

class StaggeredAnimationDemo extends StatefulWidget {
  const StaggeredAnimationDemo({super.key});

  @override
  State<StaggeredAnimationDemo> createState() => _StaggeredAnimationDemoState();
}

class _StaggeredAnimationDemoState extends State<StaggeredAnimationDemo>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;
  bool _isVisible = false;

  static const _items = [
    StaggeredListItem(title: 'Swift Performance', subtitle: 'Ultra-fast processing', icon: Icons.speed),
    StaggeredListItem(title: 'Beautiful UI', subtitle: 'Material Design 3', icon: Icons.palette),
    StaggeredListItem(title: 'Cross Platform', subtitle: 'iOS, Android & Web', icon: Icons.devices),
    StaggeredListItem(title: 'Hot Reload', subtitle: 'Instant preview', icon: Icons.refresh),
    StaggeredListItem(title: 'Rich Ecosystem', subtitle: 'pub.dev packages', icon: Icons.extension),
  ];

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 1800),
    );
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  void _toggle() {
    setState(() => _isVisible = !_isVisible);
    if (_isVisible) {
      _ctrl.forward();
    } else {
      _ctrl.reverse();
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Staggered Animations')),
      body: Column(
        children: [
          const SizedBox(height: 16),
          ElevatedButton.icon(
            onPressed: _toggle,
            icon: Icon(_isVisible ? Icons.visibility_off : Icons.visibility),
            label: Text(_isVisible ? 'Hide' : 'Show'),
          ),
          const SizedBox(height: 24),
          Expanded(
            child: ListView.builder(
              padding: const EdgeInsets.symmetric(horizontal: 16),
              itemCount: _items.length,
              itemBuilder: (ctx, i) {
                // Stagger each item 150ms apart, within the 1800ms total
                const itemDuration = 0.4; // fraction of total
                final start = i * 0.12;
                final end = (start + itemDuration).clamp(0.0, 1.0);

                final fadeAnim = CurvedAnimation(
                  parent: _ctrl,
                  curve: Interval(start, end, curve: Curves.easeOut),
                );
                final slideAnim = Tween<Offset>(
                  begin: const Offset(-0.5, 0),
                  end: Offset.zero,
                ).animate(CurvedAnimation(
                  parent: _ctrl,
                  curve: Interval(start, end, curve: Curves.easeOutCubic),
                ));

                return Padding(
                  padding: const EdgeInsets.only(bottom: 10),
                  child: FadeTransition(
                    opacity: fadeAnim,
                    child: SlideTransition(
                      position: slideAnim,
                      child: _StaggeredCard(item: _items[i], index: i),
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

class _StaggeredCard extends StatelessWidget {
  const _StaggeredCard({required this.item, required this.index});
  final StaggeredListItem item;
  final int index;

  static const _colors = [
    Colors.blue, Colors.purple, Colors.teal, Colors.orange, Colors.pink,
  ];

  @override
  Widget build(BuildContext context) {
    final color = _colors[index % _colors.length];
    return Card(
      child: ListTile(
        leading: CircleAvatar(
          backgroundColor: color.withOpacity(0.15),
          child: Icon(item.icon, color: color),
        ),
        title: Text(item.title, style: const TextStyle(fontWeight: FontWeight.w600)),
        subtitle: Text(item.subtitle),
        trailing: Icon(Icons.chevron_right, color: Colors.grey.shade400),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1963: Staggered Header Animation

```dart
// lib/animations/staggered_header.dart
import 'package:flutter/material.dart';

class StaggeredHeaderAnimation extends StatefulWidget {
  const StaggeredHeaderAnimation({super.key});

  @override
  State<StaggeredHeaderAnimation> createState() => _StaggeredHeaderAnimationState();
}

class _StaggeredHeaderAnimationState extends State<StaggeredHeaderAnimation>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;

  late final Animation<double> _logoScale;
  late final Animation<double> _titleFade;
  late final Animation<Offset> _titleSlide;
  late final Animation<double> _subtitleFade;
  late final Animation<double> _buttonFade;
  late final Animation<Offset> _buttonSlide;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 2000),
    );

    _logoScale = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(parent: _ctrl, curve: const Interval(0.0, 0.4, curve: Curves.elasticOut)),
    );

    _titleFade = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(parent: _ctrl, curve: const Interval(0.3, 0.6, curve: Curves.easeIn)),
    );
    _titleSlide = Tween<Offset>(begin: const Offset(0, 0.5), end: Offset.zero).animate(
      CurvedAnimation(parent: _ctrl, curve: const Interval(0.3, 0.6, curve: Curves.easeOut)),
    );

    _subtitleFade = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(parent: _ctrl, curve: const Interval(0.5, 0.8, curve: Curves.easeIn)),
    );

    _buttonFade = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(parent: _ctrl, curve: const Interval(0.7, 1.0, curve: Curves.easeIn)),
    );
    _buttonSlide = Tween<Offset>(begin: const Offset(0, 1), end: Offset.zero).animate(
      CurvedAnimation(parent: _ctrl, curve: const Interval(0.7, 1.0, curve: Curves.easeOutCubic)),
    );

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
      backgroundColor: const Color(0xFF1A237E),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Logo
            ScaleTransition(
              scale: _logoScale,
              child: Container(
                width: 100,
                height: 100,
                decoration: BoxDecoration(
                  color: Colors.white,
                  borderRadius: BorderRadius.circular(24),
                ),
                child: const Icon(Icons.flutter_dash, size: 60, color: Color(0xFF1A237E)),
              ),
            ),
            const SizedBox(height: 32),
            // Title
            FadeTransition(
              opacity: _titleFade,
              child: SlideTransition(
                position: _titleSlide,
                child: const Text(
                  'Flutter Pro',
                  style: TextStyle(
                    color: Colors.white,
                    fontSize: 36,
                    fontWeight: FontWeight.bold,
                    letterSpacing: 2,
                  ),
                ),
              ),
            ),
            const SizedBox(height: 12),
            // Subtitle
            FadeTransition(
              opacity: _subtitleFade,
              child: const Text(
                'Build beautiful apps',
                style: TextStyle(color: Colors.white60, fontSize: 16),
              ),
            ),
            const SizedBox(height: 48),
            // Button
            FadeTransition(
              opacity: _buttonFade,
              child: SlideTransition(
                position: _buttonSlide,
                child: ElevatedButton(
                  onPressed: () {},
                  style: ElevatedButton.styleFrom(
                    padding: const EdgeInsets.symmetric(horizontal: 48, vertical: 16),
                    backgroundColor: Colors.white,
                    foregroundColor: const Color(0xFF1A237E),
                    shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(30)),
                  ),
                  child: const Text('Get Started', style: TextStyle(fontSize: 16)),
                ),
              ),
            ),
            const SizedBox(height: 16),
            TextButton(
              onPressed: () {
                _ctrl.reset();
                _ctrl.forward();
              },
              child: const Text('Replay', style: TextStyle(color: Colors.white54)),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1964: TweenAnimationBuilder

```dart
// lib/animations/tween_animation_builder_demo.dart
import 'package:flutter/material.dart';

class TweenBuilderDemo extends StatefulWidget {
  const TweenBuilderDemo({super.key});

  @override
  State<TweenBuilderDemo> createState() => _TweenBuilderDemoState();
}

class _TweenBuilderDemoState extends State<TweenBuilderDemo> {
  double _progress = 0;
  Color _targetColor = Colors.blue;
  double _rotation = 0;
  double _scale = 1.0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('TweenAnimationBuilder')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(24),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // Progress ring
            const Text('Circular Progress', style: TextStyle(fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            TweenAnimationBuilder<double>(
              tween: Tween(begin: 0, end: _progress),
              duration: const Duration(milliseconds: 800),
              curve: Curves.easeOutCubic,
              builder: (ctx, value, child) {
                return Center(
                  child: SizedBox(
                    width: 120,
                    height: 120,
                    child: Stack(
                      alignment: Alignment.center,
                      children: [
                        CircularProgressIndicator(
                          value: value,
                          strokeWidth: 10,
                          backgroundColor: Colors.grey.shade200,
                        ),
                        Text(
                          '${(value * 100).round()}%',
                          style: const TextStyle(
                            fontSize: 22,
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                      ],
                    ),
                  ),
                );
              },
            ),
            const SizedBox(height: 8),
            Slider(
              value: _progress,
              onChanged: (v) => setState(() => _progress = v),
            ),
            const Divider(height: 32),

            // Color transition
            const Text('Color Transition', style: TextStyle(fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            TweenAnimationBuilder<Color?>(
              tween: ColorTween(begin: Colors.blue, end: _targetColor),
              duration: const Duration(milliseconds: 600),
              builder: (ctx, color, child) {
                return AnimatedContainer(
                  duration: Duration.zero,
                  height: 80,
                  decoration: BoxDecoration(
                    color: color,
                    borderRadius: BorderRadius.circular(16),
                  ),
                  child: Center(
                    child: Text(
                      '#${color?.value.toRadixString(16).padLeft(8, '0').toUpperCase()}',
                      style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold),
                    ),
                  ),
                );
              },
            ),
            const SizedBox(height: 8),
            Wrap(
              spacing: 8,
              children: [Colors.blue, Colors.red, Colors.green, Colors.orange, Colors.purple]
                  .map((c) => GestureDetector(
                        onTap: () => setState(() => _targetColor = c),
                        child: CircleAvatar(backgroundColor: c, radius: 18),
                      ))
                  .toList(),
            ),
            const Divider(height: 32),

            // Rotation
            const Text('Rotation', style: TextStyle(fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            TweenAnimationBuilder<double>(
              tween: Tween(begin: 0, end: _rotation),
              duration: const Duration(milliseconds: 600),
              curve: Curves.easeOutBack,
              builder: (ctx, angle, child) {
                return Center(
                  child: Transform.rotate(
                    angle: angle,
                    child: child,
                  ),
                );
              },
              child: Container(
                width: 80,
                height: 80,
                decoration: BoxDecoration(
                  gradient: const LinearGradient(
                    colors: [Colors.blue, Colors.purple],
                    begin: Alignment.topLeft,
                    end: Alignment.bottomRight,
                  ),
                  borderRadius: BorderRadius.circular(16),
                ),
                child: const Icon(Icons.star, color: Colors.white, size: 40),
              ),
            ),
            const SizedBox(height: 8),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                ElevatedButton(
                  onPressed: () => setState(() => _rotation -= 3.14159 / 2),
                  child: const Icon(Icons.rotate_left),
                ),
                const SizedBox(width: 16),
                ElevatedButton(
                  onPressed: () => setState(() => _rotation += 3.14159 / 2),
                  child: const Icon(Icons.rotate_right),
                ),
              ],
            ),
            const Divider(height: 32),

            // Scale bounce
            const Text('Scale Bounce', style: TextStyle(fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            TweenAnimationBuilder<double>(
              tween: Tween(begin: 1.0, end: _scale),
              duration: const Duration(milliseconds: 500),
              curve: Curves.elasticOut,
              builder: (ctx, s, child) {
                return Center(
                  child: Transform.scale(scale: s, child: child),
                );
              },
              child: ElevatedButton(
                onPressed: () {
                  setState(() => _scale = _scale == 1.0 ? 1.5 : 1.0);
                },
                style: ElevatedButton.styleFrom(
                  padding: const EdgeInsets.symmetric(horizontal: 32, vertical: 16),
                ),
                child: const Text('Tap to Scale'),
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

## ขั้นตอนที่ 1965: AnimatedSwitcher กับ Custom Transitions

```dart
// lib/animations/animated_switcher_demo.dart
import 'package:flutter/material.dart';

class AnimatedSwitcherDemo extends StatefulWidget {
  const AnimatedSwitcherDemo({super.key});

  @override
  State<AnimatedSwitcherDemo> createState() => _AnimatedSwitcherDemoState();
}

class _AnimatedSwitcherDemoState extends State<AnimatedSwitcherDemo> {
  int _page = 0;
  _TransitionType _transitionType = _TransitionType.fade;

  static const _pages = [
    _PageContent(
      color: Color(0xFF1565C0),
      icon: Icons.home,
      label: 'Home',
    ),
    _PageContent(
      color: Color(0xFF2E7D32),
      icon: Icons.settings,
      label: 'Settings',
    ),
    _PageContent(
      color: Color(0xFF6A1B9A),
      icon: Icons.person,
      label: 'Profile',
    ),
  ];

  Widget _transitionBuilder(Widget child, Animation<double> animation) {
    switch (_transitionType) {
      case _TransitionType.fade:
        return FadeTransition(opacity: animation, child: child);

      case _TransitionType.scale:
        return ScaleTransition(
          scale: CurvedAnimation(parent: animation, curve: Curves.easeOut),
          child: FadeTransition(opacity: animation, child: child),
        );

      case _TransitionType.slide:
        return SlideTransition(
          position: Tween<Offset>(
            begin: const Offset(1.0, 0),
            end: Offset.zero,
          ).animate(CurvedAnimation(parent: animation, curve: Curves.easeOut)),
          child: child,
        );

      case _TransitionType.rotation:
        return RotationTransition(
          turns: Tween<double>(begin: 0.5, end: 1.0).animate(
            CurvedAnimation(parent: animation, curve: Curves.easeOut),
          ),
          child: FadeTransition(opacity: animation, child: child),
        );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('AnimatedSwitcher')),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: Wrap(
              spacing: 8,
              children: _TransitionType.values.map((t) {
                return ChoiceChip(
                  label: Text(t.name),
                  selected: _transitionType == t,
                  onSelected: (_) => setState(() => _transitionType = t),
                );
              }).toList(),
            ),
          ),
          Expanded(
            child: Center(
              child: AnimatedSwitcher(
                duration: const Duration(milliseconds: 500),
                transitionBuilder: _transitionBuilder,
                child: _PageView(
                  key: ValueKey(_page),
                  content: _pages[_page],
                ),
              ),
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(16),
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceEvenly,
              children: List.generate(
                _pages.length,
                (i) => ElevatedButton(
                  onPressed: () => setState(() => _page = i),
                  style: ElevatedButton.styleFrom(
                    backgroundColor: _page == i ? _pages[i].color : null,
                    foregroundColor: _page == i ? Colors.white : null,
                  ),
                  child: Icon(_pages[i].icon),
                ),
              ),
            ),
          ),
        ],
      ),
    );
  }
}

class _PageView extends StatelessWidget {
  const _PageView({super.key, required this.content});
  final _PageContent content;

  @override
  Widget build(BuildContext context) {
    return Container(
      width: 240,
      height: 240,
      decoration: BoxDecoration(
        color: content.color,
        borderRadius: BorderRadius.circular(24),
        boxShadow: [
          BoxShadow(
            color: content.color.withOpacity(0.4),
            blurRadius: 20,
            offset: const Offset(0, 8),
          ),
        ],
      ),
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(content.icon, color: Colors.white, size: 64),
          const SizedBox(height: 16),
          Text(
            content.label,
            style: const TextStyle(color: Colors.white, fontSize: 24, fontWeight: FontWeight.bold),
          ),
        ],
      ),
    );
  }
}

class _PageContent {
  const _PageContent({required this.color, required this.icon, required this.label});
  final Color color;
  final IconData icon;
  final String label;
}

enum _TransitionType { fade, scale, slide, rotation }
```

---

## ขั้นตอนที่ 1966: Lottie Animation Integration

```dart
// lib/animations/lottie_demo.dart
// pubspec.yaml ต้องมี:
//   lottie: ^3.1.0
//
// ตัวอย่าง Lottie animation URL (ใช้ network):
//   https://assets5.lottiefiles.com/packages/lf20_jhu1lpit.json

import 'package:flutter/material.dart';
import 'package:lottie/lottie.dart';

class LottieDemoPage extends StatefulWidget {
  const LottieDemoPage({super.key});

  @override
  State<LottieDemoPage> createState() => _LottieDemoPageState();
}

class _LottieDemoPageState extends State<LottieDemoPage>
    with TickerProviderStateMixin {
  late final AnimationController _successCtrl;
  late final AnimationController _loadingCtrl;
  bool _showSuccess = false;
  bool _isLoading = false;

  @override
  void initState() {
    super.initState();
    _successCtrl = AnimationController(vsync: this);
    _loadingCtrl = AnimationController(vsync: this);
  }

  @override
  void dispose() {
    _successCtrl.dispose();
    _loadingCtrl.dispose();
    super.dispose();
  }

  Future<void> _simulateAction() async {
    setState(() => _isLoading = true);
    _loadingCtrl.repeat();
    await Future.delayed(const Duration(seconds: 2));
    _loadingCtrl.stop();
    setState(() {
      _isLoading = false;
      _showSuccess = true;
    });
    await _successCtrl.forward();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Lottie Animations')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            if (_isLoading)
              // Loading Lottie (network or asset)
              Lottie.network(
                'https://assets9.lottiefiles.com/packages/lf20_x62chJ.json',
                controller: _loadingCtrl,
                width: 150,
                height: 150,
                onLoaded: (composition) {
                  _loadingCtrl.duration = composition.duration;
                },
                errorBuilder: (ctx, err, stack) => const SizedBox(
                  width: 150,
                  height: 150,
                  child: CircularProgressIndicator(),
                ),
              )
            else if (_showSuccess)
              // Success Lottie
              Lottie.network(
                'https://assets4.lottiefiles.com/packages/lf20_lk80fpsm.json',
                controller: _successCtrl,
                width: 200,
                height: 200,
                repeat: false,
                onLoaded: (composition) {
                  _successCtrl.duration = composition.duration;
                },
                errorBuilder: (ctx, err, stack) => const Icon(
                  Icons.check_circle,
                  size: 80,
                  color: Colors.green,
                ),
              )
            else
              const Icon(Icons.play_circle_outline, size: 80, color: Colors.grey),
            const SizedBox(height: 32),
            if (!_isLoading && !_showSuccess)
              ElevatedButton.icon(
                onPressed: _simulateAction,
                icon: const Icon(Icons.send),
                label: const Text('Submit'),
              ),
            if (_showSuccess) ...[
              const Text('Success!', style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold, color: Colors.green)),
              const SizedBox(height: 16),
              OutlinedButton(
                onPressed: () {
                  _successCtrl.reset();
                  setState(() => _showSuccess = false);
                },
                child: const Text('Reset'),
              ),
            ],
          ],
        ),
      ),
    );
  }
}

/// ถ้าใช้ asset Lottie แทน network:
/// 1. เพิ่มไฟล์ assets/animations/loading.json
/// 2. ใน pubspec.yaml:
///    flutter:
///      assets:
///        - assets/animations/
/// 3. ใช้ Lottie.asset('assets/animations/loading.json', ...)
class LottieAssetDemo extends StatelessWidget {
  const LottieAssetDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Lottie Asset')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // ใช้ asset file
            Lottie.asset(
              'assets/animations/loading.json',
              width: 200,
              height: 200,
              // fit: BoxFit.contain,
              errorBuilder: (ctx, err, stack) {
                return Container(
                  width: 200,
                  height: 200,
                  decoration: BoxDecoration(
                    color: Colors.grey.shade100,
                    borderRadius: BorderRadius.circular(16),
                  ),
                  child: const Center(
                    child: Text('Add Lottie JSON\nto assets/', textAlign: TextAlign.center),
                  ),
                );
              },
            ),
            const SizedBox(height: 24),
            const Text('Lottie from Asset File'),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1967: Shared Axis Transition (animations package)

```dart
// lib/animations/shared_axis_demo.dart
// pubspec.yaml ต้องมี:
//   animations: ^2.0.11

import 'package:flutter/material.dart';
import 'package:animations/animations.dart';

class SharedAxisTransitionDemo extends StatefulWidget {
  const SharedAxisTransitionDemo({super.key});

  @override
  State<SharedAxisTransitionDemo> createState() => _SharedAxisTransitionDemoState();
}

class _SharedAxisTransitionDemoState extends State<SharedAxisTransitionDemo> {
  int _selectedIndex = 0;
  SharedAxisTransitionType _transitionType = SharedAxisTransitionType.horizontal;

  final _pages = const [
    _SAPage(title: 'Dashboard', icon: Icons.dashboard, color: Colors.blue),
    _SAPage(title: 'Analytics', icon: Icons.bar_chart, color: Colors.purple),
    _SAPage(title: 'Messages', icon: Icons.message, color: Colors.teal),
    _SAPage(title: 'Profile', icon: Icons.person, color: Colors.orange),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Shared Axis'),
        actions: [
          PopupMenuButton<SharedAxisTransitionType>(
            onSelected: (t) => setState(() => _transitionType = t),
            itemBuilder: (_) => [
              const PopupMenuItem(value: SharedAxisTransitionType.horizontal, child: Text('Horizontal')),
              const PopupMenuItem(value: SharedAxisTransitionType.vertical, child: Text('Vertical')),
              const PopupMenuItem(value: SharedAxisTransitionType.scaled, child: Text('Scaled')),
            ],
          ),
        ],
      ),
      body: PageTransitionSwitcher(
        duration: const Duration(milliseconds: 400),
        reverse: _selectedIndex < (_selectedIndex),
        transitionBuilder: (child, animation, secondaryAnimation) {
          return SharedAxisTransition(
            animation: animation,
            secondaryAnimation: secondaryAnimation,
            transitionType: _transitionType,
            child: child,
          );
        },
        child: _SAPageView(
          key: ValueKey(_selectedIndex),
          page: _pages[_selectedIndex],
        ),
      ),
      bottomNavigationBar: NavigationBar(
        selectedIndex: _selectedIndex,
        onDestinationSelected: (i) => setState(() => _selectedIndex = i),
        destinations: _pages
            .map((p) => NavigationDestination(icon: Icon(p.icon), label: p.title))
            .toList(),
      ),
    );
  }
}

class _SAPageView extends StatelessWidget {
  const _SAPageView({super.key, required this.page});
  final _SAPage page;

  @override
  Widget build(BuildContext context) {
    return Container(
      color: Colors.grey.shade50,
      child: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Container(
              width: 100,
              height: 100,
              decoration: BoxDecoration(
                color: page.color.withOpacity(0.15),
                borderRadius: BorderRadius.circular(24),
              ),
              child: Icon(page.icon, size: 48, color: page.color),
            ),
            const SizedBox(height: 16),
            Text(
              page.title,
              style: TextStyle(
                fontSize: 28,
                fontWeight: FontWeight.bold,
                color: page.color,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _SAPage {
  const _SAPage({required this.title, required this.icon, required this.color});
  final String title;
  final IconData icon;
  final Color color;
}

/// Container Transform — OpenContainer
class OpenContainerDemo extends StatelessWidget {
  const OpenContainerDemo({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Container Transform')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Wrap(
          spacing: 16,
          runSpacing: 16,
          children: List.generate(6, (i) {
            return OpenContainer(
              closedShape: RoundedRectangleBorder(
                borderRadius: BorderRadius.circular(16),
              ),
              closedColor: Colors.primaries[i % Colors.primaries.length].shade100,
              closedBuilder: (ctx, openContainer) {
                return InkWell(
                  onTap: openContainer,
                  child: SizedBox(
                    width: 140,
                    height: 100,
                    child: Center(
                      child: Column(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          Icon(
                            Icons.photo,
                            size: 32,
                            color: Colors.primaries[i % Colors.primaries.length],
                          ),
                          const SizedBox(height: 8),
                          Text('Item ${i + 1}'),
                        ],
                      ),
                    ),
                  ),
                );
              },
              openBuilder: (ctx, _) {
                return Scaffold(
                  appBar: AppBar(title: Text('Item ${i + 1}')),
                  backgroundColor: Colors.primaries[i % Colors.primaries.length].shade50,
                  body: Center(
                    child: Icon(
                      Icons.photo,
                      size: 120,
                      color: Colors.primaries[i % Colors.primaries.length],
                    ),
                  ),
                );
              },
            );
          }),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1968: Fade Through Transition

```dart
// lib/animations/fade_through_demo.dart
import 'package:flutter/material.dart';
import 'package:animations/animations.dart';

class FadeThroughDemo extends StatefulWidget {
  const FadeThroughDemo({super.key});

  @override
  State<FadeThroughDemo> createState() => _FadeThroughDemoState();
}

class _FadeThroughDemoState extends State<FadeThroughDemo> {
  int _index = 0;
  bool _reverse = false;
  int _prevIndex = 0;

  void _navigate(int newIndex) {
    setState(() {
      _reverse = newIndex < _index;
      _prevIndex = _index;
      _index = newIndex;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Fade Through')),
      body: PageTransitionSwitcher(
        reverse: _reverse,
        duration: const Duration(milliseconds: 350),
        transitionBuilder: (child, animation, secondaryAnimation) {
          return FadeThroughTransition(
            animation: animation,
            secondaryAnimation: secondaryAnimation,
            child: child,
          );
        },
        child: _buildPage(_index),
      ),
      bottomNavigationBar: NavigationBar(
        selectedIndex: _index,
        onDestinationSelected: _navigate,
        destinations: const [
          NavigationDestination(icon: Icon(Icons.inbox), label: 'Inbox'),
          NavigationDestination(icon: Icon(Icons.article), label: 'Articles'),
          NavigationDestination(icon: Icon(Icons.bookmark), label: 'Saved'),
        ],
      ),
    );
  }

  Widget _buildPage(int index) {
    final titles = ['Inbox', 'Articles', 'Saved'];
    final icons = [Icons.inbox, Icons.article, Icons.bookmark];
    final colors = [Colors.indigo, Colors.teal, Colors.amber];
    return Container(
      key: ValueKey(index),
      color: Colors.white,
      child: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(icons[index], size: 80, color: colors[index]),
            const SizedBox(height: 16),
            Text(
              titles[index],
              style: TextStyle(
                fontSize: 32,
                fontWeight: FontWeight.bold,
                color: colors[index],
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

## ขั้นตอนที่ 1969: Full Advanced Animation App

```dart
// lib/main_animations_demo.dart
import 'package:flutter/material.dart';
import 'animations/hero_demo.dart';
import 'animations/staggered_animation.dart';
import 'animations/staggered_header.dart';
import 'animations/tween_animation_builder_demo.dart';
import 'animations/animated_switcher_demo.dart';
import 'animations/lottie_demo.dart';
import 'animations/shared_axis_demo.dart';
import 'animations/fade_through_demo.dart';

void main() => runApp(const AnimationsDemoApp());

class AnimationsDemoApp extends StatelessWidget {
  const AnimationsDemoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Advanced Animations',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(colorSchemeSeed: Colors.indigo, useMaterial3: true),
      home: const AnimationsDemoHome(),
    );
  }
}

class AnimationsDemoHome extends StatelessWidget {
  const AnimationsDemoHome({super.key});

  static final _demos = <_Demo>[
    _Demo('Hero Animation', Icons.flight, () => const HeroDemoListPage()),
    _Demo('Staggered List', Icons.format_list_bulleted, () => const StaggeredAnimationDemo()),
    _Demo('Staggered Header', Icons.title, () => const StaggeredHeaderAnimation()),
    _Demo('TweenAnimationBuilder', Icons.tune, () => const TweenBuilderDemo()),
    _Demo('AnimatedSwitcher', Icons.swap_horiz, () => const AnimatedSwitcherDemo()),
    _Demo('Lottie Animations', Icons.animation, () => const LottieDemoPage()),
    _Demo('Shared Axis', Icons.swap_calls, () => const SharedAxisTransitionDemo()),
    _Demo('Container Transform', Icons.open_in_new, () => const OpenContainerDemo()),
    _Demo('Fade Through', Icons.blur_on, () => const FadeThroughDemo()),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Part 52 — Advanced Animations')),
      body: ListView.separated(
        padding: const EdgeInsets.all(16),
        itemCount: _demos.length,
        separatorBuilder: (_, __) => const SizedBox(height: 8),
        itemBuilder: (ctx, i) {
          final demo = _demos[i];
          return ListTile(
            leading: Icon(demo.icon, color: Colors.indigo),
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

**← [Part 51](part-51-custom-paint-canvas.md)**
**ต่อไป: [Part 53 →](part-53-firebase-firestore-advanced.md)**
