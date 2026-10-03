# Part 99: Career & Professional Flutter Development
## ขั้นตอนที่ 3841-3880

## 🎯 เป้าหมายของ Part นี้
- แนวทางการ Contribute ให้ Open Source Flutter Projects
- สร้าง Flutter Portfolio ที่โดดเด่น
- เผยแพร่ Flutter Package บน pub.dev (พร้อม pubspec.yaml ครบถ้วน)
- Code Review Best Practices
- เตรียมตัวสัมภาษณ์งาน Flutter (คำถาม + code ตอบได้)

---

## ขั้นตอนที่ 3841: Open Source Contribution Guidelines

```markdown
# Contributing to Flutter Open Source

## 1. Finding Projects to Contribute To

Search strategies:
- github.com/flutter/flutter (issues labeled 'good first issue')
- github.com/rrousselGit/riverpod
- github.com/felangel/bloc
- pub.dev (packages you use daily)

## 2. Contribution Workflow

1. Fork → Clone → Create Branch
2. Make focused, small PRs (< 400 lines)
3. Follow existing code style
4. Add tests for new functionality
5. Update documentation
6. Submit PR with clear description
```

```bash
# Setting up for OSS contribution
git clone https://github.com/flutter/packages.git
cd packages
git checkout -b fix/animation-controller-dispose

# Create a focused fix branch
git add -p  # Stage changes interactively (not git add .)
git commit -m "fix: properly dispose AnimationController in FadeTransition

AnimationController was not being disposed when the widget was removed
from the widget tree, causing memory leaks in long-running apps.

Fixes: flutter/flutter#12345"

git push origin fix/animation-controller-dispose
# Then open PR on GitHub
```

## ขั้นตอนที่ 3842: Building a Portfolio Project

```dart
// portfolio/lib/main.dart
// A Flutter Web portfolio showcasing your skills
import 'package:flutter/material.dart';
import 'package:flutter_animate/flutter_animate.dart';
import 'package:url_launcher/url_launcher.dart';

void main() => runApp(const PortfolioApp());

class PortfolioApp extends StatelessWidget {
  const PortfolioApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Phutjirakul - Flutter Developer',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF6C63FF)),
        useMaterial3: true,
      ),
      home: const PortfolioHome(),
    );
  }
}

class PortfolioHome extends StatefulWidget {
  const PortfolioHome({super.key});

  @override
  State<PortfolioHome> createState() => _PortfolioHomeState();
}

class _PortfolioHomeState extends State<PortfolioHome> {
  final _scrollController = ScrollController();
  int _currentSection = 0;

  final _projects = [
    PortfolioProject(
      title: 'FoodDelivery App',
      description: 'Full-stack food delivery app with Flutter + Node.js. '
          'Features real-time order tracking, payment integration, '
          'and admin dashboard.',
      tags: ['Flutter', 'Riverpod', 'Clean Architecture', 'Firebase'],
      githubUrl: 'https://github.com/username/food-delivery',
      imageUrl: 'assets/projects/food_delivery.png',
      stars: 284,
    ),
    PortfolioProject(
      title: 'Flutter Chat SDK',
      description: 'Open-source chat SDK with WebSocket support, '
          'message encryption, and offline caching.',
      tags: ['Flutter', 'WebSocket', 'SQLite', 'BLoC'],
      githubUrl: 'https://github.com/username/flutter-chat-sdk',
      imageUrl: 'assets/projects/chat_sdk.png',
      stars: 156,
    ),
    PortfolioProject(
      title: 'pub.dev: animated_list_tile',
      description: 'Flutter package for animated list tiles with '
          '50+ animation presets. 1,200+ pub points.',
      tags: ['Dart Package', 'Animation', 'pub.dev'],
      githubUrl: 'https://github.com/username/animated_list_tile',
      imageUrl: 'assets/projects/package.png',
      stars: 92,
      pubUrl: 'https://pub.dev/packages/animated_list_tile',
    ),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        controller: _scrollController,
        slivers: [
          _buildHeroSection(context),
          _buildSkillsSection(context),
          _buildProjectsSection(context),
          _buildContactSection(context),
        ],
      ),
    );
  }

  Widget _buildHeroSection(BuildContext context) {
    return SliverToBoxAdapter(
      child: Container(
        height: MediaQuery.of(context).size.height,
        decoration: BoxDecoration(
          gradient: LinearGradient(
            begin: Alignment.topLeft,
            end: Alignment.bottomRight,
            colors: [
              Theme.of(context).colorScheme.primaryContainer,
              Theme.of(context).colorScheme.surface,
            ],
          ),
        ),
        child: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              CircleAvatar(
                radius: 60,
                backgroundColor: Theme.of(context).colorScheme.primary,
                child: Text(
                  'P',
                  style: Theme.of(context).textTheme.displayMedium?.copyWith(
                        color: Colors.white,
                        fontWeight: FontWeight.bold,
                      ),
                ),
              ).animate().fadeIn(duration: 600.ms).scale(),
              const SizedBox(height: 24),
              Text(
                'Flutter Developer',
                style: Theme.of(context).textTheme.headlineMedium?.copyWith(
                      fontWeight: FontWeight.bold,
                    ),
              ).animate().fadeIn(delay: 300.ms).slideY(begin: 0.3),
              const SizedBox(height: 12),
              Text(
                '5+ Years Building Beautiful, Performant Mobile & Web Apps',
                style: Theme.of(context).textTheme.titleMedium?.copyWith(
                      color: Theme.of(context).colorScheme.onSurfaceVariant,
                    ),
                textAlign: TextAlign.center,
              ).animate().fadeIn(delay: 600.ms),
              const SizedBox(height: 32),
              Wrap(
                spacing: 12,
                children: [
                  _HeroChip(icon: Icons.location_on, label: 'Bangkok, Thailand'),
                  _HeroChip(icon: Icons.work, label: 'Open to Remote'),
                  _HeroChip(icon: Icons.star, label: '500+ GitHub Stars'),
                ],
              ).animate().fadeIn(delay: 900.ms),
              const SizedBox(height: 32),
              Row(
                mainAxisSize: MainAxisSize.min,
                children: [
                  FilledButton.icon(
                    onPressed: () => launchUrl(Uri.parse('mailto:dev@example.com')),
                    icon: const Icon(Icons.email_rounded),
                    label: const Text('Contact Me'),
                  ),
                  const SizedBox(width: 12),
                  OutlinedButton.icon(
                    onPressed: () => launchUrl(
                        Uri.parse('https://github.com/username')),
                    icon: const Icon(Icons.code_rounded),
                    label: const Text('GitHub'),
                  ),
                ],
              ).animate().fadeIn(delay: 1200.ms),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildSkillsSection(BuildContext context) {
    final skills = {
      'Mobile': ['Flutter', 'Dart', 'Android (Kotlin)', 'iOS (Swift)'],
      'State Management': ['Riverpod', 'BLoC', 'Provider', 'GetX'],
      'Backend': ['Node.js', 'Firebase', 'Supabase', 'REST/GraphQL'],
      'Tools': ['Git', 'CI/CD', 'Docker', 'Figma'],
    };

    return SliverToBoxAdapter(
      child: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 48, vertical: 64),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Skills & Expertise',
              style: Theme.of(context).textTheme.headlineMedium?.copyWith(
                    fontWeight: FontWeight.bold,
                  ),
            ),
            const SizedBox(height: 32),
            Wrap(
              spacing: 24,
              runSpacing: 24,
              children: skills.entries.map((entry) {
                return SizedBox(
                  width: 200,
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(
                        entry.key,
                        style: Theme.of(context).textTheme.titleSmall?.copyWith(
                              fontWeight: FontWeight.bold,
                              color: Theme.of(context).colorScheme.primary,
                            ),
                      ),
                      const SizedBox(height: 8),
                      ...entry.value.map(
                        (skill) => Padding(
                          padding: const EdgeInsets.only(bottom: 4),
                          child: Row(
                            children: [
                              Icon(
                                Icons.check_circle_rounded,
                                size: 16,
                                color: Theme.of(context).colorScheme.primary,
                              ),
                              const SizedBox(width: 8),
                              Text(skill),
                            ],
                          ),
                        ),
                      ),
                    ],
                  ),
                );
              }).toList(),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildProjectsSection(BuildContext context) {
    return SliverToBoxAdapter(
      child: Container(
        color: Theme.of(context).colorScheme.surfaceVariant.withOpacity(0.3),
        padding: const EdgeInsets.symmetric(horizontal: 48, vertical: 64),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Featured Projects',
              style: Theme.of(context).textTheme.headlineMedium?.copyWith(
                    fontWeight: FontWeight.bold,
                  ),
            ),
            const SizedBox(height: 32),
            GridView.builder(
              shrinkWrap: true,
              physics: const NeverScrollableScrollPhysics(),
              gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
                maxCrossAxisExtent: 400,
                mainAxisExtent: 280,
                crossAxisSpacing: 24,
                mainAxisSpacing: 24,
              ),
              itemCount: _projects.length,
              itemBuilder: (context, index) => _ProjectCard(
                project: _projects[index],
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildContactSection(BuildContext context) {
    return SliverToBoxAdapter(
      child: Container(
        padding: const EdgeInsets.all(64),
        child: Column(
          children: [
            Text(
              'Let\'s Work Together',
              style: Theme.of(context).textTheme.headlineMedium?.copyWith(
                    fontWeight: FontWeight.bold,
                  ),
            ),
            const SizedBox(height: 16),
            Text(
              'Available for freelance, full-time, and consulting opportunities',
              style: Theme.of(context).textTheme.bodyLarge?.copyWith(
                    color: Theme.of(context).colorScheme.onSurfaceVariant,
                  ),
            ),
            const SizedBox(height: 32),
            FilledButton.icon(
              onPressed: () => launchUrl(Uri.parse('mailto:dev@example.com')),
              icon: const Icon(Icons.email_rounded),
              label: const Text('Send Message'),
              style: FilledButton.styleFrom(
                padding: const EdgeInsets.symmetric(horizontal: 32, vertical: 16),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _HeroChip extends StatelessWidget {
  final IconData icon;
  final String label;

  const _HeroChip({required this.icon, required this.label});

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
      decoration: BoxDecoration(
        color: Theme.of(context).colorScheme.primaryContainer,
        borderRadius: BorderRadius.circular(20),
      ),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(icon, size: 16, color: Theme.of(context).colorScheme.primary),
          const SizedBox(width: 6),
          Text(label, style: const TextStyle(fontSize: 13)),
        ],
      ),
    );
  }
}

class _ProjectCard extends StatelessWidget {
  final PortfolioProject project;

  const _ProjectCard({required this.project});

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Icon(
                  Icons.phone_android_rounded,
                  size: 32,
                  color: Theme.of(context).colorScheme.primary,
                ),
                Row(
                  children: [
                    const Icon(Icons.star_rounded, size: 16, color: Colors.amber),
                    const SizedBox(width: 4),
                    Text(project.stars.toString()),
                  ],
                ),
              ],
            ),
            const SizedBox(height: 12),
            Text(
              project.title,
              style: Theme.of(context).textTheme.titleMedium?.copyWith(
                    fontWeight: FontWeight.bold,
                  ),
            ),
            const SizedBox(height: 8),
            Expanded(
              child: Text(
                project.description,
                style: Theme.of(context).textTheme.bodySmall?.copyWith(
                      color: Theme.of(context).colorScheme.onSurfaceVariant,
                    ),
                overflow: TextOverflow.fade,
              ),
            ),
            const SizedBox(height: 12),
            Wrap(
              spacing: 6,
              runSpacing: 6,
              children: project.tags
                  .map(
                    (tag) => Container(
                      padding: const EdgeInsets.symmetric(
                          horizontal: 8, vertical: 4),
                      decoration: BoxDecoration(
                        color: Theme.of(context)
                            .colorScheme
                            .secondaryContainer,
                        borderRadius: BorderRadius.circular(8),
                      ),
                      child: Text(
                        tag,
                        style: const TextStyle(fontSize: 11),
                      ),
                    ),
                  )
                  .toList(),
            ),
            const SizedBox(height: 12),
            Row(
              children: [
                TextButton.icon(
                  onPressed: () => launchUrl(Uri.parse(project.githubUrl)),
                  icon: const Icon(Icons.code_rounded, size: 16),
                  label: const Text('Code'),
                  style: TextButton.styleFrom(
                    padding: EdgeInsets.zero,
                    visualDensity: VisualDensity.compact,
                  ),
                ),
                if (project.pubUrl != null) ...[
                  const SizedBox(width: 12),
                  TextButton.icon(
                    onPressed: () =>
                        launchUrl(Uri.parse(project.pubUrl!)),
                    icon: const Icon(Icons.open_in_new_rounded, size: 16),
                    label: const Text('pub.dev'),
                    style: TextButton.styleFrom(
                      padding: EdgeInsets.zero,
                      visualDensity: VisualDensity.compact,
                    ),
                  ),
                ],
              ],
            ),
          ],
        ),
      ),
    );
  }
}

class PortfolioProject {
  final String title;
  final String description;
  final List<String> tags;
  final String githubUrl;
  final String imageUrl;
  final int stars;
  final String? pubUrl;

  const PortfolioProject({
    required this.title,
    required this.description,
    required this.tags,
    required this.githubUrl,
    required this.imageUrl,
    required this.stars,
    this.pubUrl,
  });
}
```

## ขั้นตอนที่ 3843: Flutter Package Publishing Guide

```yaml
# pubspec.yaml - ตัวอย่าง Package สมบูรณ์
name: animated_list_tile
description: >-
  A Flutter package that provides beautifully animated list tiles with 50+
  animation presets. Supports enter, exit, and idle animations with full
  customization support.
version: 1.2.0
repository: https://github.com/username/animated_list_tile
homepage: https://username.github.io/animated_list_tile
issue_tracker: https://github.com/username/animated_list_tile/issues

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.1

topics:
  - animation
  - list
  - ui
  - widgets
  - material

screenshots:
  - description: "Fade animation preset"
    path: screenshots/fade.png
  - description: "Slide animation preset"
    path: screenshots/slide.png
  - description: "Bounce animation preset"
    path: screenshots/bounce.png

funding:
  - url: https://github.com/sponsors/username
```

```dart
// lib/animated_list_tile.dart
/// A Flutter package for beautifully animated list tiles.
///
/// ## Usage
///
/// ```dart
/// AnimatedListTile(
///   animation: ListTileAnimation.slideRight,
///   child: ListTile(
///     title: Text('Hello World'),
///   ),
/// )
/// ```
library animated_list_tile;

export 'src/animated_list_tile.dart';
export 'src/animations/list_tile_animation.dart';
export 'src/animations/animation_presets.dart';
```

```dart
// lib/src/animated_list_tile.dart
import 'package:flutter/material.dart';
import 'package:animated_list_tile/src/animations/list_tile_animation.dart';

/// A widget that wraps any widget with beautiful animations.
///
/// Provides 50+ animation presets for list items, cards, and tiles.
///
/// Example:
/// ```dart
/// AnimatedListTile(
///   animation: ListTileAnimation.slideRight,
///   delay: Duration(milliseconds: 100),
///   child: Card(
///     child: ListTile(title: Text('Item')),
///   ),
/// )
/// ```
class AnimatedListTile extends StatefulWidget {
  /// The widget to animate
  final Widget child;

  /// The animation preset to use
  final ListTileAnimation animation;

  /// Delay before animation starts
  final Duration delay;

  /// Duration of the animation
  final Duration duration;

  /// Animation curve
  final Curve curve;

  /// Whether to reverse the animation on exit
  final bool reverseOnExit;

  const AnimatedListTile({
    super.key,
    required this.child,
    this.animation = ListTileAnimation.fadeIn,
    this.delay = Duration.zero,
    this.duration = const Duration(milliseconds: 300),
    this.curve = Curves.easeOut,
    this.reverseOnExit = false,
  });

  @override
  State<AnimatedListTile> createState() => _AnimatedListTileState();
}

class _AnimatedListTileState extends State<AnimatedListTile>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: widget.duration,
    );
    _animation = CurvedAnimation(
      parent: _controller,
      curve: widget.curve,
    );

    // Start with delay
    if (widget.delay == Duration.zero) {
      _controller.forward();
    } else {
      Future.delayed(widget.delay, () {
        if (mounted) _controller.forward();
      });
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
      builder: (context, child) {
        return switch (widget.animation) {
          ListTileAnimation.fadeIn => Opacity(
              opacity: _animation.value,
              child: child,
            ),
          ListTileAnimation.slideRight => Transform.translate(
              offset: Offset(-50 * (1 - _animation.value), 0),
              child: Opacity(opacity: _animation.value, child: child),
            ),
          ListTileAnimation.slideUp => Transform.translate(
              offset: Offset(0, 30 * (1 - _animation.value)),
              child: Opacity(opacity: _animation.value, child: child),
            ),
          ListTileAnimation.scale => Transform.scale(
              scale: 0.8 + 0.2 * _animation.value,
              child: Opacity(opacity: _animation.value, child: child),
            ),
          ListTileAnimation.bounce => Transform.scale(
              scale: _buildBounceValue(_animation.value),
              child: child,
            ),
          ListTileAnimation.flip => Transform(
              transform: Matrix4.identity()
                ..rotateX((1 - _animation.value) * 3.14159 / 2),
              alignment: Alignment.center,
              child: child,
            ),
        };
      },
      child: widget.child,
    );
  }

  double _buildBounceValue(double t) {
    if (t < 0.6) return t / 0.6;
    if (t < 0.8) return 1.0 + 0.2 * ((t - 0.6) / 0.2);
    if (t < 0.9) return 1.2 - 0.2 * ((t - 0.8) / 0.1);
    return 1.0;
  }
}
```

```dart
// lib/src/animations/list_tile_animation.dart

/// Available animation presets for AnimatedListTile
enum ListTileAnimation {
  /// Simple fade in from transparent to opaque
  fadeIn,

  /// Slide in from the left with fade
  slideRight,

  /// Slide in from below with fade
  slideUp,

  /// Scale up from 80% to 100% with fade
  scale,

  /// Bounce effect - overshoots slightly then settles
  bounce,

  /// 3D flip effect on X axis
  flip,
}
```

## ขั้นตอนที่ 3844: Code Review Best Practices

```dart
// BAD - What NOT to do in code reviews:
// lib/features/home/bad_example.dart

class BadHomeWidget extends StatefulWidget {
  @override
  _BadHomeWidgetState createState() => _BadHomeWidgetState();
}

class _BadHomeWidgetState extends State<BadHomeWidget> {
  List data = []; // BAD: untyped list
  
  @override
  void initState() {
    super.initState();
    // BAD: async in initState without proper error handling
    Future.delayed(Duration.zero).then((_) async {
      final res = await http.get(Uri.parse('https://api.example.com/data'));
      // BAD: direct JSON parsing without try/catch
      setState(() => data = jsonDecode(res.body));
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: data.map((item) => Text(item['name'])).toList(), // BAD: type cast
    );
  }
}
```

```dart
// GOOD - Code review approved patterns:
// lib/features/home/good_example.dart

// ✅ Typed, immutable data
// ✅ Error handling
// ✅ Loading states
// ✅ Proper async handling
// ✅ Using Riverpod providers
// ✅ Clean separation of concerns

@riverpod
Future<List<HomeItem>> homeItems(HomeItemsRef ref) async {
  final repository = ref.read(homeRepositoryProvider);
  final result = await repository.getHomeItems();
  return result.fold(
    (failure) => throw failure,
    (items) => items,
  );
}

class GoodHomeWidget extends ConsumerWidget {
  const GoodHomeWidget({super.key}); // ✅ const constructor

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final itemsAsync = ref.watch(homeItemsProvider);

    return itemsAsync.when(
      data: (items) => _ItemList(items: items),  // ✅ extracted widget
      loading: () => const _LoadingState(),
      error: (err, stack) => _ErrorState(
        error: err,
        onRetry: () => ref.invalidate(homeItemsProvider),
      ),
    );
  }
}

// ✅ Extracted, reusable, testable
class _ItemList extends StatelessWidget {
  final List<HomeItem> items;

  const _ItemList({required this.items});

  @override
  Widget build(BuildContext context) {
    if (items.isEmpty) {
      return const _EmptyState();
    }
    return ListView.separated(
      itemCount: items.length,
      separatorBuilder: (_, __) => const Divider(height: 1),
      itemBuilder: (context, index) => HomeItemTile(item: items[index]),
    );
  }
}
```

```dart
// Code Review Checklist as executable tests
// test/code_quality/architecture_test.dart

/// These tests enforce architectural rules
void main() {
  group('Architecture Rules', () {
    test('domain entities should not import flutter', () {
      // Run: dart run custom_lint
      // Verify no Flutter imports in domain layer
      // This is enforced by custom_lint rules
    });

    test('repositories should return Either, not throw', () {
      // All repository methods should return Either<Failure, T>
      // Verified by type checking at compile time
    });

    test('widgets should not call repositories directly', () {
      // Enforced by AvoidDirectApiCallRule custom lint
    });
  });
}
```

## ขั้นตอนที่ 3845: Technical Interview Prep - Core Concepts

```dart
// INTERVIEW QUESTION 1: Explain StatelessWidget vs StatefulWidget
// with a practical example

/// StatelessWidget: Immutable, no internal state
class PriceTag extends StatelessWidget {
  final double price;
  final String currency;

  const PriceTag({
    super.key,
    required this.price,
    this.currency = '฿',
  });

  @override
  Widget build(BuildContext context) {
    return Text(
      '$currency${price.toStringAsFixed(2)}',
      style: Theme.of(context).textTheme.titleLarge?.copyWith(
            color: Theme.of(context).colorScheme.primary,
            fontWeight: FontWeight.bold,
          ),
    );
  }
}

/// StatefulWidget: Has mutable internal state
class CounterButton extends StatefulWidget {
  final int initialCount;
  final ValueChanged<int>? onCountChanged;

  const CounterButton({
    super.key,
    this.initialCount = 0,
    this.onCountChanged,
  });

  @override
  State<CounterButton> createState() => _CounterButtonState();
}

class _CounterButtonState extends State<CounterButton> {
  late int _count;

  @override
  void initState() {
    super.initState();
    _count = widget.initialCount;
  }

  void _increment() {
    setState(() => _count++);
    widget.onCountChanged?.call(_count);
  }

  void _decrement() {
    if (_count > 0) {
      setState(() => _count--);
      widget.onCountChanged?.call(_count);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        IconButton(
          onPressed: _count > 0 ? _decrement : null,
          icon: const Icon(Icons.remove_rounded),
          style: IconButton.styleFrom(
            backgroundColor: Theme.of(context).colorScheme.surfaceVariant,
          ),
        ),
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 12),
          child: Text(
            '$_count',
            style: Theme.of(context).textTheme.titleMedium,
          ),
        ),
        IconButton(
          onPressed: _increment,
          icon: const Icon(Icons.add_rounded),
          style: IconButton.styleFrom(
            backgroundColor: Theme.of(context).colorScheme.primary,
            foregroundColor: Colors.white,
          ),
        ),
      ],
    );
  }
}
```

## ขั้นตอนที่ 3846: Interview - Keys and Widget Rebuild

```dart
// INTERVIEW QUESTION 2: When and why use Keys in Flutter?

// Problem WITHOUT keys: items don't animate correctly
Widget badList() {
  return StatefulBuilder(
    builder: (context, setState) {
      final items = ['Apple', 'Banana', 'Cherry'];
      return Column(
        // BAD: No keys - Flutter can't track which widget is which
        children: items.map((item) => _AnimatedItem(name: item)).toList(),
      );
    },
  );
}

// Solution WITH keys: proper widget identity tracking
Widget goodList() {
  return StatefulBuilder(
    builder: (context, setState) {
      final items = ['Apple', 'Banana', 'Cherry'];
      return Column(
        // GOOD: ValueKey ensures proper widget tracking
        children: items
            .map((item) => _AnimatedItem(key: ValueKey(item), name: item))
            .toList(),
      );
    },
  );
}

class _AnimatedItem extends StatefulWidget {
  final String name;

  const _AnimatedItem({super.key, required this.name});

  @override
  State<_AnimatedItem> createState() => _AnimatedItemState();
}

class _AnimatedItemState extends State<_AnimatedItem>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 300),
    )..forward();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return FadeTransition(
      opacity: _controller,
      child: ListTile(title: Text(widget.name)),
    );
  }
}

// GlobalKey: access State from outside widget tree
class ParentWidget extends StatelessWidget {
  final GlobalKey<_ChildWithStateState> _childKey = GlobalKey();

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        _ChildWithState(key: _childKey),
        ElevatedButton(
          onPressed: () => _childKey.currentState?.doSomething(),
          child: const Text('Trigger Child Action'),
        ),
      ],
    );
  }
}

class _ChildWithState extends StatefulWidget {
  const _ChildWithState({super.key});

  @override
  State<_ChildWithState> createState() => _ChildWithStateState();
}

class _ChildWithStateState extends State<_ChildWithState> {
  String _message = 'Idle';

  void doSomething() {
    setState(() => _message = 'Action triggered at ${DateTime.now()}');
  }

  @override
  Widget build(BuildContext context) => Text(_message);
}
```

## ขั้นตอนที่ 3847: Interview - BuildContext and InheritedWidget

```dart
// INTERVIEW QUESTION 3: How does BuildContext work?
// Explain InheritedWidget and how Theme.of(context) works

/// Custom InheritedWidget - the foundation of Flutter's context system
class AppSettings extends InheritedWidget {
  final String language;
  final bool isDarkMode;
  final String currency;

  const AppSettings({
    super.key,
    required this.language,
    required this.isDarkMode,
    required this.currency,
    required super.child,
  });

  /// Static accessor - this is how Theme.of(context) works under the hood
  static AppSettings of(BuildContext context) {
    final settings = context.dependOnInheritedWidgetOfExactType<AppSettings>();
    assert(settings != null, 'No AppSettings found in context');
    return settings!;
  }

  /// Only rebuild dependents when relevant data changes
  @override
  bool updateShouldNotify(AppSettings oldWidget) =>
      language != oldWidget.language ||
      isDarkMode != oldWidget.isDarkMode ||
      currency != oldWidget.currency;
}

/// Usage example
class PriceDisplay extends StatelessWidget {
  final double amount;

  const PriceDisplay({super.key, required this.amount});

  @override
  Widget build(BuildContext context) {
    // This widget will rebuild ONLY when AppSettings.currency changes
    final currency = AppSettings.of(context).currency;
    return Text('$currency ${amount.toStringAsFixed(2)}');
  }
}

// Context tree example
void contextTreeExample(BuildContext context) {
  // Each of these traverses the widget tree upward
  final theme = Theme.of(context);          // Finds ThemeData
  final mediaQuery = MediaQuery.of(context); // Finds MediaQueryData
  final navigator = Navigator.of(context);   // Finds NavigatorState
  final scaffold = Scaffold.of(context);     // Finds ScaffoldState

  // context.mounted: Always check before async gaps
  Future.delayed(const Duration(seconds: 1), () {
    if (context.mounted) {
      // Safe to use context here
      Navigator.of(context).pop();
    }
  });
}
```

## ขั้นตอนที่ 3848: Interview - Async & Streams

```dart
// INTERVIEW QUESTION 4: Explain Futures vs Streams in Dart

// Future: single async value
Future<String> fetchUserName(String userId) async {
  final response = await http.get(
    Uri.parse('https://api.example.com/users/$userId'),
  );
  if (response.statusCode == 200) {
    return jsonDecode(response.body)['name'] as String;
  }
  throw Exception('User not found');
}

// Stream: multiple async values over time
Stream<OrderStatus> trackOrderStatus(String orderId) async* {
  // Simulate real-time updates
  yield OrderStatus.pending;
  await Future.delayed(const Duration(seconds: 2));
  yield OrderStatus.processing;
  await Future.delayed(const Duration(seconds: 3));
  yield OrderStatus.pickedUp;
  await Future.delayed(const Duration(seconds: 5));
  yield OrderStatus.delivered;
}

// Real-world Stream with WebSocket
Stream<ChatMessage> chatMessages(String roomId) {
  final controller = StreamController<ChatMessage>();

  final socket = WebSocketChannel.connect(
    Uri.parse('wss://api.example.com/chat/$roomId'),
  );

  socket.stream.listen(
    (data) {
      final message = ChatMessage.fromJson(jsonDecode(data as String));
      controller.add(message);
    },
    onError: controller.addError,
    onDone: controller.close,
  );

  // Cancel subscription when stream is no longer listened to
  controller.onCancel = () => socket.sink.close();

  return controller.stream;
}

// StreamBuilder in Flutter
class LiveOrderTracker extends StatelessWidget {
  final String orderId;

  const LiveOrderTracker({super.key, required this.orderId});

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<OrderStatus>(
      stream: trackOrderStatus(orderId),
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.waiting) {
          return const CircularProgressIndicator();
        }
        if (snapshot.hasError) {
          return Text('Error: ${snapshot.error}');
        }
        if (!snapshot.hasData) {
          return const Text('No data');
        }

        final status = snapshot.data!;
        return OrderStatusDisplay(status: status);
      },
    );
  }
}

// Future.wait for concurrent operations
Future<void> initializeApp() async {
  // Run 3 operations concurrently, not sequentially
  final results = await Future.wait([
    fetchUserProfile(),      // ~500ms
    fetchRecentOrders(),     // ~300ms
    fetchRestaurantList(),   // ~400ms
  ]);
  // Total: ~500ms (max), not 1200ms (sum)
}
```

## ขั้นตอนที่ 3849: Interview - Performance Optimization

```dart
// INTERVIEW QUESTION 5: How do you optimize Flutter performance?

// 1. const constructors - avoid unnecessary rebuilds
class OptimizedWidget extends StatelessWidget {
  final String data;

  const OptimizedWidget({super.key, required this.data}); // ✅ const

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        const _StaticHeader(),  // ✅ const - never rebuilds
        Text(data),             // Only this rebuilds when data changes
        const _StaticFooter(),  // ✅ const - never rebuilds
      ],
    );
  }
}

// 2. RepaintBoundary - isolate repaints
class ChatMessageList extends StatelessWidget {
  const ChatMessageList({super.key});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemBuilder: (context, index) => RepaintBoundary(
        // Each message is isolated - typing indicator at top
        // won't repaint all messages
        child: ChatMessage(index: index),
      ),
    );
  }
}

// 3. ListView.builder vs ListView for large lists
Widget buildEfficientList(List<String> items) {
  // BAD: builds ALL items at once
  // return ListView(children: items.map((i) => Text(i)).toList());

  // GOOD: builds only visible items
  return ListView.builder(
    itemCount: items.length,
    // addRepaintBoundaries: true (default - don't disable)
    itemBuilder: (context, index) => ListTile(
      title: Text(items[index]),
    ),
  );
}

// 4. Use Selector to avoid over-rebuilding
class PriceWidget extends ConsumerWidget {
  const PriceWidget({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // BAD: rebuilds when ANY cart field changes
    // final cart = ref.watch(cartProvider);
    // return Text('Total: ${cart?.total}');

    // GOOD: only rebuilds when total changes
    final total = ref.watch(
      cartProvider.select((cart) => cart?.subtotal ?? 0.0),
    );
    return Text('Total: ฿${total.toStringAsFixed(2)}');
  }
}

// 5. Image optimization
Widget buildOptimizedImage(String url) {
  return CachedNetworkImage(
    imageUrl: url,
    // Cache decoded image - avoids re-decoding on scroll
    memCacheWidth: 400,
    memCacheHeight: 400,
    // Show shimmer while loading
    placeholder: (context, url) => const ShimmerBox(width: 400, height: 400),
    fadeInDuration: const Duration(milliseconds: 200),
  );
}

// 6. Avoid heavy computation in build()
class CorrectComputationWidget extends StatefulWidget {
  final List<Product> products;

  const CorrectComputationWidget({super.key, required this.products});

  @override
  State<CorrectComputationWidget> createState() =>
      _CorrectComputationWidgetState();
}

class _CorrectComputationWidgetState
    extends State<CorrectComputationWidget> {
  late List<Product> _sortedProducts;

  @override
  void initState() {
    super.initState();
    // GOOD: compute once in initState, not in build()
    _sortedProducts = _sortProducts(widget.products);
  }

  @override
  void didUpdateWidget(CorrectComputationWidget oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (oldWidget.products != widget.products) {
      _sortedProducts = _sortProducts(widget.products);
    }
  }

  List<Product> _sortProducts(List<Product> products) {
    return [...products]..sort((a, b) => b.rating.compareTo(a.rating));
  }

  @override
  Widget build(BuildContext context) {
    // BAD was: final sorted = products.sort(...); // runs every build!
    return ListView.builder(
      itemCount: _sortedProducts.length,
      itemBuilder: (context, index) => ProductCard(
        product: _sortedProducts[index],
      ),
    );
  }
}
```

## ขั้นตอนที่ 3850: Interview - State Management

```dart
// INTERVIEW QUESTION 6: Compare state management approaches

// 1. setState - Simple local state
class SimpleCounter extends StatefulWidget {
  const SimpleCounter({super.key});

  @override
  State<SimpleCounter> createState() => _SimpleCounterState();
}

class _SimpleCounterState extends State<SimpleCounter> {
  int _count = 0;

  @override
  Widget build(BuildContext context) => Column(
        children: [
          Text('Count: $_count'),
          FloatingActionButton(
            onPressed: () => setState(() => _count++),
            child: const Icon(Icons.add),
          ),
        ],
      );
}

// 2. Provider/Riverpod - App-wide state
@riverpod
class CartNotifier extends _$CartNotifier {
  @override
  Cart? build() => null;

  void addItem(CartItem item) {
    final current = state;
    if (current == null) {
      state = Cart(items: [item]);
    } else {
      final existing = current.items.indexWhere(
        (i) => i.menuItemId == item.menuItemId,
      );
      if (existing >= 0) {
        final updated = [...current.items];
        updated[existing] = updated[existing].copyWith(
          quantity: updated[existing].quantity + item.quantity,
        );
        state = current.copyWith(items: updated);
      } else {
        state = current.copyWith(items: [...current.items, item]);
      }
    }
  }

  void removeItem(String menuItemId) {
    state = state?.copyWith(
      items: state!.items.where((i) => i.menuItemId != menuItemId).toList(),
    );
  }

  void clear() => state = null;
}

// 3. BLoC - Complex business logic
abstract class OrderEvent {}
class PlaceOrderEvent extends OrderEvent {
  final CreateOrderParams params;
  PlaceOrderEvent(this.params);
}
class CancelOrderEvent extends OrderEvent {
  final String orderId;
  CancelOrderEvent(this.orderId);
}

abstract class OrderState {}
class OrderInitial extends OrderState {}
class OrderLoading extends OrderState {}
class OrderSuccess extends OrderState {
  final Order order;
  OrderSuccess(this.order);
}
class OrderFailure extends OrderState {
  final String message;
  OrderFailure(this.message);
}

class OrderBloc extends Bloc<OrderEvent, OrderState> {
  final OrderRepository _repository;

  OrderBloc(this._repository) : super(OrderInitial()) {
    on<PlaceOrderEvent>(_onPlaceOrder);
    on<CancelOrderEvent>(_onCancelOrder);
  }

  Future<void> _onPlaceOrder(
    PlaceOrderEvent event,
    Emitter<OrderState> emit,
  ) async {
    emit(OrderLoading());
    final result = await _repository.createOrder(params: event.params);
    result.fold(
      (failure) => emit(OrderFailure(failure.message)),
      (order) => emit(OrderSuccess(order)),
    );
  }

  Future<void> _onCancelOrder(
    CancelOrderEvent event,
    Emitter<OrderState> emit,
  ) async {
    emit(OrderLoading());
    final result = await _repository.cancelOrder(
      orderId: event.orderId,
      reason: 'User requested cancellation',
    );
    result.fold(
      (failure) => emit(OrderFailure(failure.message)),
      (order) => emit(OrderSuccess(order)),
    );
  }
}
```

## ขั้นตอนที่ 3851: Interview - Clean Architecture

```dart
// INTERVIEW QUESTION 7: Explain Clean Architecture in Flutter

// Layer 1: Domain (no Flutter dependencies)
// lib/features/auth/domain/entities/user.dart
class User {
  final String id;
  final String name;
  final String email;
  final UserRole role;
  
  const User({
    required this.id,
    required this.name,
    required this.email,
    required this.role,
  });
}

enum UserRole { customer, restaurantOwner, rider, admin }

// lib/features/auth/domain/repositories/auth_repository.dart
abstract class AuthRepository {
  Future<Either<Failure, User>> login({
    required String email,
    required String password,
  });
  Future<Either<Failure, void>> logout();
  Future<Either<Failure, User>> getCurrentUser();
  Stream<User?> authStateChanges();
}

// lib/features/auth/domain/use_cases/login_use_case.dart
class LoginUseCase {
  final AuthRepository _repository;

  LoginUseCase(this._repository);

  Future<Either<Failure, User>> call(LoginParams params) {
    return _repository.login(
      email: params.email,
      password: params.password,
    );
  }
}

class LoginParams {
  final String email;
  final String password;

  const LoginParams({required this.email, required this.password});
}

// Layer 2: Data
// lib/features/auth/data/datasources/auth_remote_datasource.dart
abstract class AuthRemoteDataSource {
  Future<({User user, String token})> login({
    required String email,
    required String password,
  });
  Future<void> logout();
}

class AuthRemoteDataSourceImpl implements AuthRemoteDataSource {
  final Dio _dio;

  AuthRemoteDataSourceImpl(this._dio);

  @override
  Future<({User user, String token})> login({
    required String email,
    required String password,
  }) async {
    final response = await _dio.post(
      '/auth/login',
      data: {'email': email, 'password': password},
    );
    return (
      user: User.fromJson(response.data['user']),
      token: response.data['token'] as String,
    );
  }

  @override
  Future<void> logout() async {
    await _dio.post('/auth/logout');
  }
}

// lib/features/auth/data/repositories/auth_repository_impl.dart
class AuthRepositoryImpl implements AuthRepository {
  final AuthRemoteDataSource _remoteDataSource;
  final AuthLocalDataSource _localDataSource;

  AuthRepositoryImpl({
    required AuthRemoteDataSource remoteDataSource,
    required AuthLocalDataSource localDataSource,
  })  : _remoteDataSource = remoteDataSource,
        _localDataSource = localDataSource;

  @override
  Future<Either<Failure, User>> login({
    required String email,
    required String password,
  }) async {
    try {
      final result = await _remoteDataSource.login(
        email: email,
        password: password,
      );
      await _localDataSource.cacheToken(result.token);
      await _localDataSource.cacheUser(result.user);
      return Right(result.user);
    } on NetworkException catch (e) {
      return Left(NetworkFailure(message: e.message));
    } on ServerException catch (e) {
      return Left(ServerFailure(message: e.message));
    } catch (e) {
      return Left(UnexpectedFailure(message: e.toString()));
    }
  }

  @override
  Future<Either<Failure, void>> logout() async {
    try {
      await _remoteDataSource.logout();
      await _localDataSource.clearToken();
      await _localDataSource.clearUser();
      return const Right(null);
    } catch (e) {
      return Left(UnexpectedFailure(message: e.toString()));
    }
  }

  @override
  Future<Either<Failure, User>> getCurrentUser() async {
    try {
      final user = await _localDataSource.getCachedUser();
      if (user == null) return Left(CacheFailure(message: 'No cached user'));
      return Right(user);
    } on CacheException catch (e) {
      return Left(CacheFailure(message: e.message));
    }
  }

  @override
  Stream<User?> authStateChanges() => _localDataSource.userStream();
}
```

---

**← [Part 98 - World-Class Patterns](part-98-world-class-patterns.md)**
**ต่อไป: [Part 100 →](part-100-course-completion.md)**
