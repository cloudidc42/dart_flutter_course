# Part 70: Flutter Game Development with Flame
## ขั้นตอนที่ 2681-2720

## 🎯 เป้าหมายของ Part นี้
- เรียนรู้ Flame engine พื้นฐาน
- เข้าใจ Game loop และ component lifecycle
- สร้าง Sprite animations
- ทำ Collision detection
- สร้าง 2D Breakout game ที่สมบูรณ์

---

## ขั้นตอนที่ 2681: Flame Engine Setup

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  flame: ^1.17.0
  flame_audio: ^2.10.0

flutter:
  assets:
    - assets/images/
    - assets/audio/
    - assets/sprites/
```

```dart
// lib/main_game.dart
import 'package:flutter/material.dart';
import 'package:flame/game.dart';
import 'games/breakout_game.dart';

void main() {
  runApp(const GameApp());
}

class GameApp extends StatelessWidget {
  const GameApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Game',
      debugShowCheckedModeBanner: false,
      home: GameWidget.controlled(
        gameFactory: BreakoutGame.new,
        overlayBuilderMap: {
          'MainMenu': (ctx, game) => MainMenuOverlay(game: game as BreakoutGame),
          'GameOver': (ctx, game) => GameOverOverlay(game: game as BreakoutGame),
          'PauseMenu': (ctx, game) => PauseMenuOverlay(game: game as BreakoutGame),
        },
        initialActiveOverlays: const ['MainMenu'],
      ),
    );
  }
}

class MainMenuOverlay extends StatelessWidget {
  final BreakoutGame game;

  const MainMenuOverlay({super.key, required this.game});

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Container(
        padding: const EdgeInsets.all(32),
        decoration: BoxDecoration(
          color: Colors.black87,
          borderRadius: BorderRadius.circular(16),
        ),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const Text(
              'BREAKOUT',
              style: TextStyle(
                color: Colors.white,
                fontSize: 48,
                fontWeight: FontWeight.bold,
                letterSpacing: 4,
              ),
            ),
            const SizedBox(height: 32),
            ElevatedButton(
              onPressed: () {
                game.overlays.remove('MainMenu');
                game.startGame();
              },
              style: ElevatedButton.styleFrom(
                backgroundColor: Colors.orange,
                padding: const EdgeInsets.symmetric(
                  horizontal: 48, vertical: 16,
                ),
              ),
              child: const Text(
                'PLAY',
                style: TextStyle(fontSize: 24, color: Colors.white),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class GameOverOverlay extends StatelessWidget {
  final BreakoutGame game;

  const GameOverOverlay({super.key, required this.game});

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Container(
        padding: const EdgeInsets.all(32),
        decoration: BoxDecoration(
          color: Colors.black87,
          borderRadius: BorderRadius.circular(16),
        ),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const Text(
              'GAME OVER',
              style: TextStyle(
                color: Colors.red,
                fontSize: 40,
                fontWeight: FontWeight.bold,
              ),
            ),
            const SizedBox(height: 16),
            Text(
              'Score: ${game.score}',
              style: const TextStyle(color: Colors.white, fontSize: 24),
            ),
            const SizedBox(height: 32),
            ElevatedButton(
              onPressed: () {
                game.overlays.remove('GameOver');
                game.resetGame();
              },
              child: const Text('PLAY AGAIN'),
            ),
          ],
        ),
      ),
    );
  }
}

class PauseMenuOverlay extends StatelessWidget {
  final BreakoutGame game;

  const PauseMenuOverlay({super.key, required this.game});

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Container(
        padding: const EdgeInsets.all(24),
        decoration: BoxDecoration(
          color: Colors.black87,
          borderRadius: BorderRadius.circular(12),
        ),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const Text(
              'PAUSED',
              style: TextStyle(
                color: Colors.white,
                fontSize: 32,
                fontWeight: FontWeight.bold,
              ),
            ),
            const SizedBox(height: 24),
            ElevatedButton(
              onPressed: () {
                game.overlays.remove('PauseMenu');
                game.resumeEngine();
              },
              child: const Text('RESUME'),
            ),
            const SizedBox(height: 8),
            TextButton(
              onPressed: () {
                game.overlays.remove('PauseMenu');
                game.overlays.add('MainMenu');
                game.resetGame();
              },
              child: const Text(
                'MAIN MENU',
                style: TextStyle(color: Colors.white70),
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

## ขั้นตอนที่ 2682: Game Loop และ Component Lifecycle

```dart
// lib/games/breakout_game.dart
import 'package:flame/game.dart';
import 'package:flame/components.dart';
import 'package:flame/events.dart';
import 'package:flame/input.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'components/paddle.dart';
import 'components/ball.dart';
import 'components/brick.dart';
import 'components/score_text.dart';
import 'components/lives_display.dart';

class BreakoutGame extends FlameGame
    with HasCollisionDetection, KeyboardEvents, TapCallbacks {
  // Game state
  int score = 0;
  int lives = 3;
  int level = 1;
  bool isPlaying = false;

  // Components
  late final Paddle paddle;
  late final Ball ball;
  late final ScoreText scoreText;
  late final LivesDisplay livesDisplay;

  // Game dimensions
  static const double gameWidth = 400;
  static const double gameHeight = 700;

  @override
  Color backgroundColor() => const Color(0xFF0A0A1A);

  @override
  Future<void> onLoad() async {
    // Set fixed viewport
    camera.viewport = FixedResolutionViewport(
      resolution: Vector2(gameWidth, gameHeight),
    );

    // Load assets
    await loadAssets();

    // Setup world boundaries
    _addWorldBoundaries();

    // Add HUD components
    scoreText = ScoreText();
    livesDisplay = LivesDisplay(lives: lives);
    add(scoreText);
    add(livesDisplay);

    // Add paddle
    paddle = Paddle(
      position: Vector2(gameWidth / 2, gameHeight - 60),
    );
    add(paddle);

    // Add ball
    ball = Ball(
      position: Vector2(gameWidth / 2, gameHeight / 2),
    );
    add(ball);

    // Generate bricks for level 1
    _generateBricks(level);

    print('BreakoutGame loaded');
  }

  Future<void> loadAssets() async {
    // ถ้ามี assets จริงให้ load ที่นี่
    // await images.loadAll(['ball.png', 'paddle.png', 'brick.png']);
  }

  void _addWorldBoundaries() {
    // Top wall
    add(ScreenHitbox());

    // Side walls and top using PositionComponent + rectangles
    addAll([
      // Left wall
      RectangleComponent(
        position: Vector2(-10, 0),
        size: Vector2(10, gameHeight),
        paint: Paint()..color = Colors.transparent,
      )..add(RectangleHitbox()),

      // Right wall
      RectangleComponent(
        position: Vector2(gameWidth, 0),
        size: Vector2(10, gameHeight),
        paint: Paint()..color = Colors.transparent,
      )..add(RectangleHitbox()),
    ]);
  }

  void _generateBricks(int level) {
    const brickRows = 5;
    const brickCols = 8;
    const brickWidth = 44.0;
    const brickHeight = 20.0;
    const brickPadding = 4.0;
    const topOffset = 80.0;
    const leftOffset = 12.0;

    final levelColors = [
      [Colors.red, Colors.orange, Colors.yellow, Colors.green, Colors.blue],
      [Colors.purple, Colors.teal, Colors.indigo, Colors.pink, Colors.cyan],
    ];

    final colors = levelColors[(level - 1) % levelColors.length];

    for (var row = 0; row < brickRows; row++) {
      for (var col = 0; col < brickCols; col++) {
        final brick = Brick(
          position: Vector2(
            leftOffset + col * (brickWidth + brickPadding),
            topOffset + row * (brickHeight + brickPadding),
          ),
          size: Vector2(brickWidth, brickHeight),
          color: colors[row % colors.length],
          points: (brickRows - row) * 10,
          hitPoints: row < 2 ? 2 : 1, // top rows need 2 hits
        );
        add(brick);
      }
    }
  }

  void startGame() {
    isPlaying = true;
    ball.launch();
    resumeEngine();
  }

  void resetGame() {
    score = 0;
    lives = 3;
    level = 1;
    isPlaying = false;

    // Remove existing bricks
    children.whereType<Brick>().forEach((b) => b.removeFromParent());

    // Reset ball and paddle
    ball.reset();
    paddle.reset();

    // Update HUD
    scoreText.updateScore(score);
    livesDisplay.updateLives(lives);

    // Generate new bricks
    _generateBricks(level);
  }

  void addScore(int points) {
    score += points;
    scoreText.updateScore(score);
  }

  void loseLife() {
    lives--;
    livesDisplay.updateLives(lives);

    if (lives <= 0) {
      _gameOver();
    } else {
      ball.reset();
      isPlaying = false;
    }
  }

  void _gameOver() {
    isPlaying = false;
    pauseEngine();
    overlays.add('GameOver');
  }

  void nextLevel() {
    level++;
    children.whereType<Brick>().forEach((b) => b.removeFromParent());
    ball.reset();
    ball.increaseSpeed(level);
    isPlaying = false;
    _generateBricks(level);

    // Show level up animation
    add(LevelUpText(level: level));
  }

  void checkLevelComplete() {
    final bricksRemaining =
        children.whereType<Brick>().where((b) => !b.isDestroyed).length;
    if (bricksRemaining == 0) {
      nextLevel();
    }
  }

  // Keyboard controls
  @override
  KeyEventResult onKeyEvent(
    KeyEvent event,
    Set<LogicalKeyboardKey> keysPressed,
  ) {
    if (event is KeyDownEvent) {
      if (event.logicalKey == LogicalKeyboardKey.escape) {
        if (!overlays.isActive('PauseMenu')) {
          pauseEngine();
          overlays.add('PauseMenu');
        }
        return KeyEventResult.handled;
      }

      if (event.logicalKey == LogicalKeyboardKey.space && !isPlaying) {
        startGame();
        return KeyEventResult.handled;
      }
    }

    paddle.handleKeyInput(keysPressed);
    return KeyEventResult.handled;
  }

  // Touch controls
  @override
  void onTapDown(TapDownEvent event) {
    paddle.moveTo(event.canvasPosition.x);
  }
}

/// Temporary level up text
class LevelUpText extends TextComponent with TimerComponent {
  final int level;

  LevelUpText({required this.level})
      : super(
          text: 'LEVEL $level',
          textRenderer: TextPaint(
            style: const TextStyle(
              color: Colors.yellow,
              fontSize: 36,
              fontWeight: FontWeight.bold,
            ),
          ),
          anchor: Anchor.center,
          position: Vector2(200, 350),
        );

  @override
  Future<void> onLoad() async {
    await super.onLoad();
    Future.delayed(const Duration(seconds: 2), removeFromParent);
  }

  @override
  void update(double dt) {
    super.update(dt);
    // Fade out effect
    if (textRenderer is TextPaint) {
      // Fade implementation
    }
  }
}
```

---

## ขั้นตอนที่ 2683: Paddle Component

```dart
// lib/games/components/paddle.dart
import 'package:flame/components.dart';
import 'package:flame/collisions.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

class Paddle extends RectangleComponent with CollisionCallbacks {
  static const double defaultWidth = 80;
  static const double defaultHeight = 15;
  static const double speed = 300; // pixels per second

  double _targetX = 0;
  bool _movingLeft = false;
  bool _movingRight = false;

  Paddle({required Vector2 position})
      : super(
          position: position,
          size: Vector2(defaultWidth, defaultHeight),
          anchor: Anchor.center,
          paint: Paint()..color = Colors.lightBlue,
        );

  @override
  Future<void> onLoad() async {
    add(RectangleHitbox());
    _targetX = position.x;

    // Add gradient effect
    paint = Paint()
      ..shader = LinearGradient(
        begin: Alignment.topCenter,
        end: Alignment.bottomCenter,
        colors: [
          Colors.lightBlue[300]!,
          Colors.lightBlue[700]!,
        ],
      ).createShader(
        Rect.fromLTWH(0, 0, defaultWidth, defaultHeight),
      );
  }

  @override
  void update(double dt) {
    super.update(dt);

    if (_movingLeft) {
      position.x -= speed * dt;
    } else if (_movingRight) {
      position.x += speed * dt;
    } else if (_targetX != position.x) {
      // Smooth movement towards target
      final diff = _targetX - position.x;
      final movement = diff.sign * speed * dt;
      if (diff.abs() <= movement.abs()) {
        position.x = _targetX;
      } else {
        position.x += movement;
      }
    }

    // Clamp to screen bounds
    final halfWidth = size.x / 2;
    position.x = position.x.clamp(
      halfWidth,
      400 - halfWidth, // gameWidth
    );
  }

  void handleKeyInput(Set<LogicalKeyboardKey> keysPressed) {
    _movingLeft = keysPressed.contains(LogicalKeyboardKey.arrowLeft) ||
        keysPressed.contains(LogicalKeyboardKey.keyA);
    _movingRight = keysPressed.contains(LogicalKeyboardKey.arrowRight) ||
        keysPressed.contains(LogicalKeyboardKey.keyD);
  }

  void moveTo(double x) {
    _targetX = x;
    _movingLeft = false;
    _movingRight = false;
  }

  void reset() {
    position.x = 200; // center
    _targetX = 200;
    _movingLeft = false;
    _movingRight = false;
  }
}
```

---

## ขั้นตอนที่ 2684: Ball Component with Physics

```dart
// lib/games/components/ball.dart
import 'package:flame/components.dart';
import 'package:flame/collisions.dart';
import 'package:flame/game.dart';
import 'package:flutter/material.dart';
import 'dart:math';
import '../breakout_game.dart';
import 'paddle.dart';
import 'brick.dart';

class Ball extends CircleComponent
    with CollisionCallbacks, HasGameRef<BreakoutGame> {
  static const double radius = 10;
  static const double baseSpeed = 300;

  Vector2 _velocity = Vector2.zero();
  bool _isActive = false;
  double _speed = baseSpeed;
  final _random = Random();

  // Trail effect
  final List<Vector2> _trail = [];
  static const int maxTrailLength = 10;

  Ball({required Vector2 position})
      : super(
          position: position,
          radius: radius,
          anchor: Anchor.center,
          paint: Paint()..color = Colors.white,
        );

  @override
  Future<void> onLoad() async {
    add(CircleHitbox());
  }

  @override
  void update(double dt) {
    super.update(dt);

    if (!_isActive) return;

    // Update trail
    _trail.add(position.clone());
    if (_trail.length > maxTrailLength) {
      _trail.removeAt(0);
    }

    // Move ball
    position += _velocity * dt;

    // Bounce off top wall
    if (position.y - radius <= 0) {
      position.y = radius;
      _velocity.y = _velocity.y.abs();
    }

    // Bounce off side walls
    if (position.x - radius <= 0) {
      position.x = radius;
      _velocity.x = _velocity.x.abs();
    } else if (position.x + radius >= 400) {
      position.x = 400 - radius;
      _velocity.x = -_velocity.x.abs();
    }

    // Ball falls below paddle
    if (position.y > 700 + radius) {
      _isActive = false;
      gameRef.loseLife();
    }
  }

  @override
  void render(Canvas canvas) {
    // Draw trail
    for (var i = 0; i < _trail.length; i++) {
      final alpha = (i / _trail.length * 0.5);
      final trailPaint = Paint()
        ..color = Colors.white.withOpacity(alpha)
        ..style = PaintingStyle.fill;

      final trailRadius = radius * (i / _trail.length);
      canvas.drawCircle(
        (_trail[i] - position).toOffset(),
        trailRadius,
        trailPaint,
      );
    }

    super.render(canvas);

    // Glow effect
    final glowPaint = Paint()
      ..color = Colors.white.withOpacity(0.3)
      ..maskFilter = const MaskFilter.blur(BlurStyle.normal, 8);
    canvas.drawCircle(Offset.zero, radius * 1.5, glowPaint);
  }

  @override
  void onCollisionStart(
    Set<Vector2> intersectionPoints,
    PositionComponent other,
  ) {
    super.onCollisionStart(intersectionPoints, other);

    if (other is Paddle) {
      _bounceOffPaddle(other);
    } else if (other is Brick) {
      _bounceOffBrick(other, intersectionPoints);
      other.hit(gameRef);
    } else if (other is ScreenHitbox) {
      _bounceOffWall(intersectionPoints);
    }
  }

  void _bounceOffPaddle(Paddle paddle) {
    // คำนวณมุม bounce ตาม position บน paddle
    final paddleCenter = paddle.position.x;
    final ballX = position.x;
    final hitOffset = (ballX - paddleCenter) / (paddle.size.x / 2);

    // -1 = ขอบซ้าย, 0 = กลาง, 1 = ขอบขวา
    final angle = hitOffset.clamp(-0.8, 0.8) * (pi / 3);

    _velocity = Vector2(
      _speed * sin(angle),
      -_speed * cos(angle).abs(),
    );
  }

  void _bounceOffBrick(Brick brick, Set<Vector2> points) {
    if (points.isEmpty) return;

    final hitPoint = points.first;
    final brickCenter = brick.position + brick.size / 2;
    final dx = hitPoint.x - brickCenter.x;
    final dy = hitPoint.y - brickCenter.y;

    // ตัดสินใจว่า bounce แนวไหน
    if (dx.abs() > dy.abs()) {
      _velocity.x *= -1;
    } else {
      _velocity.y *= -1;
    }
  }

  void _bounceOffWall(Set<Vector2> intersectionPoints) {
    if (position.y <= 0) {
      _velocity.y = _velocity.y.abs();
    }
    if (position.x <= 0) {
      _velocity.x = _velocity.x.abs();
    }
    if (position.x >= 400) {
      _velocity.x = -_velocity.x.abs();
    }
  }

  void launch() {
    _isActive = true;
    // Launch at random upward angle
    final angle = (_random.nextDouble() - 0.5) * pi / 3; // -60 to 60 degrees
    _velocity = Vector2(
      _speed * sin(angle),
      -_speed * cos(angle),
    );
  }

  void reset() {
    _isActive = false;
    _trail.clear();
    position = Vector2(200, 350); // center of screen
    _velocity = Vector2.zero();
    _speed = baseSpeed;
  }

  void increaseSpeed(int level) {
    _speed = baseSpeed + (level - 1) * 30;
  }
}
```

---

## ขั้นตอนที่ 2685: Brick Component with Animations

```dart
// lib/games/components/brick.dart
import 'package:flame/components.dart';
import 'package:flame/collisions.dart';
import 'package:flame/particles.dart';
import 'package:flutter/material.dart';
import 'dart:math';
import '../breakout_game.dart';

class Brick extends RectangleComponent with CollisionCallbacks {
  final Color color;
  final int points;
  int _hitPoints;
  bool isDestroyed = false;
  final _random = Random();

  Brick({
    required Vector2 position,
    required Vector2 size,
    required this.color,
    required this.points,
    int hitPoints = 1,
  })  : _hitPoints = hitPoints,
        super(
          position: position,
          size: size,
          anchor: Anchor.topLeft,
        );

  @override
  Future<void> onLoad() async {
    _updatePaint();
    add(RectangleHitbox());
  }

  void _updatePaint() {
    // สี ตาม hit points
    final opacity = _hitPoints > 1 ? 1.0 : 0.9;
    paint = Paint()
      ..color = color.withOpacity(opacity)
      ..style = PaintingStyle.fill;
  }

  @override
  void render(Canvas canvas) {
    super.render(canvas);

    // Border
    final borderPaint = Paint()
      ..color = Colors.white.withOpacity(0.3)
      ..style = PaintingStyle.stroke
      ..strokeWidth = 1;
    canvas.drawRect(size.toRect(), borderPaint);

    // Hit points indicator
    if (_hitPoints > 1) {
      final cracks = TextPaint(
        style: TextStyle(
          color: Colors.white.withOpacity(0.7),
          fontSize: 8,
        ),
      );
      cracks.render(canvas, '×$_hitPoints', Vector2(4, 4));
    }
  }

  void hit(BreakoutGame game) {
    if (isDestroyed) return;

    _hitPoints--;

    if (_hitPoints <= 0) {
      _destroy(game);
    } else {
      // Flash effect
      _updatePaint();
      _flashEffect();
    }
  }

  void _flashEffect() {
    paint = Paint()..color = Colors.white;
    Future.delayed(const Duration(milliseconds: 50), () {
      if (!isDestroyed) _updatePaint();
    });
  }

  void _destroy(BreakoutGame game) {
    isDestroyed = true;

    // Add score
    game.addScore(points);

    // Explosion particles
    _spawnParticles(game);

    removeFromParent();
    game.checkLevelComplete();
  }

  void _spawnParticles(BreakoutGame game) {
    final center = position + size / 2;

    game.add(
      ParticleSystemComponent(
        position: center,
        particle: Particle.generate(
          count: 15,
          lifespan: 0.8,
          generator: (i) => AcceleratedParticle(
            acceleration: Vector2(
              (_random.nextDouble() - 0.5) * 200,
              _random.nextDouble() * 200 + 50,
            ),
            speed: Vector2(
              (_random.nextDouble() - 0.5) * 150,
              -_random.nextDouble() * 100 - 50,
            ),
            child: CircleParticle(
              radius: _random.nextDouble() * 4 + 2,
              paint: Paint()
                ..color = color.withOpacity(0.8),
            ),
          ),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2686: Sprite Animations

```dart
// lib/games/components/animated_sprite_demo.dart
import 'package:flame/components.dart';
import 'package:flame/game.dart';
import 'package:flutter/material.dart';

/// ตัวอย่างการใช้ SpriteAnimation
class AnimatedCharacter extends SpriteAnimationComponent
    with HasGameRef<FlameGame> {
  late SpriteAnimation _idleAnimation;
  late SpriteAnimation _walkAnimation;
  late SpriteAnimation _jumpAnimation;

  CharacterState _state = CharacterState.idle;

  @override
  Future<void> onLoad() async {
    // ใน production โหลด sprite sheet จริง
    // final spriteSheet = await gameRef.images.load('character.png');
    // _idleAnimation = SpriteAnimation.fromFrameData(
    //   spriteSheet,
    //   SpriteAnimationData.sequenced(
    //     amount: 4,
    //     stepTime: 0.15,
    //     textureSize: Vector2(64, 64),
    //   ),
    // );

    // สำหรับ demo ใช้ simple colored rectangles
    _idleAnimation = _createColorAnimation(Colors.blue, 4, 0.2);
    _walkAnimation = _createColorAnimation(Colors.green, 6, 0.1);
    _jumpAnimation = _createColorAnimation(Colors.orange, 3, 0.15);

    animation = _idleAnimation;
    size = Vector2(64, 64);
    anchor = Anchor.center;
  }

  SpriteAnimation _createColorAnimation(
    Color color,
    int frameCount,
    double stepTime,
  ) {
    // สร้าง simple sprite animation จาก color
    final sprites = List.generate(frameCount, (i) {
      // ใน production ใช้ Sprite จาก image
      return Sprite(
        gameRef.images.fromCache('sprite.png'),
        srcPosition: Vector2(i * 64, 0),
        srcSize: Vector2(64, 64),
      );
    });

    return SpriteAnimation.spriteList(sprites, stepTime: stepTime);
  }

  void setIdle() {
    if (_state == CharacterState.idle) return;
    _state = CharacterState.idle;
    animation = _idleAnimation;
  }

  void setWalking() {
    if (_state == CharacterState.walking) return;
    _state = CharacterState.walking;
    animation = _walkAnimation;
  }

  void setJumping() {
    if (_state == CharacterState.jumping) return;
    _state = CharacterState.jumping;
    animation = _jumpAnimation;
  }
}

enum CharacterState { idle, walking, jumping }

/// Simple Frame-based Animator ไม่ต้องใช้ sprite sheet
class SimpleAnimator extends RectangleComponent {
  final List<Color> frames;
  final double frameDuration;
  int _currentFrame = 0;
  double _elapsed = 0;

  SimpleAnimator({
    required Vector2 position,
    required Vector2 size,
    required this.frames,
    this.frameDuration = 0.2,
  }) : super(
          position: position,
          size: size,
          anchor: Anchor.center,
        );

  @override
  void update(double dt) {
    super.update(dt);
    _elapsed += dt;

    if (_elapsed >= frameDuration) {
      _elapsed -= frameDuration;
      _currentFrame = (_currentFrame + 1) % frames.length;
      paint = Paint()..color = frames[_currentFrame];
    }
  }
}
```

---

## ขั้นตอนที่ 2687: HUD Components

```dart
// lib/games/components/score_text.dart
import 'package:flame/components.dart';
import 'package:flame/game.dart';
import 'package:flutter/material.dart';

class ScoreText extends TextComponent with HasGameRef<FlameGame> {
  int _score = 0;

  ScoreText()
      : super(
          text: 'Score: 0',
          position: Vector2(10, 10),
          textRenderer: TextPaint(
            style: const TextStyle(
              color: Colors.white,
              fontSize: 18,
              fontWeight: FontWeight.bold,
            ),
          ),
        );

  void updateScore(int score) {
    _score = score;
    text = 'Score: $_score';
  }
}

// lib/games/components/lives_display.dart
class LivesDisplay extends PositionComponent with HasGameRef<FlameGame> {
  int _lives;
  final List<CircleComponent> _lifeIcons = [];

  LivesDisplay({required int lives})
      : _lives = lives,
        super(
          position: Vector2(300, 15),
        );

  @override
  Future<void> onLoad() async {
    _buildIcons();
  }

  void _buildIcons() {
    for (final icon in _lifeIcons) {
      icon.removeFromParent();
    }
    _lifeIcons.clear();

    for (var i = 0; i < _lives; i++) {
      final icon = CircleComponent(
        radius: 8,
        position: Vector2(i * 22.0, 0),
        paint: Paint()..color = Colors.red,
      );
      _lifeIcons.add(icon);
      add(icon);
    }
  }

  void updateLives(int lives) {
    _lives = lives;
    _buildIcons();
  }
}
```

---

## ขั้นตอนที่ 2688: Collision Detection ขั้นสูง

```dart
// lib/games/components/collision_utils.dart
import 'package:flame/components.dart';
import 'package:flame/collisions.dart';
import 'dart:math';

/// Helper สำหรับ collision detection
class CollisionHelper {
  /// Circle-Rectangle collision
  static bool circleRect(
    Vector2 circlePos,
    double radius,
    Vector2 rectPos,
    Vector2 rectSize,
  ) {
    // หาจุดที่ใกล้สุดบน rectangle
    final closestX = circlePos.x.clamp(rectPos.x, rectPos.x + rectSize.x);
    final closestY = circlePos.y.clamp(rectPos.y, rectPos.y + rectSize.y);

    final dx = circlePos.x - closestX;
    final dy = circlePos.y - closestY;

    return (dx * dx + dy * dy) <= radius * radius;
  }

  /// คำนวณ reflection vector
  static Vector2 reflect(Vector2 velocity, Vector2 normal) {
    final dot = velocity.dot(normal);
    return velocity - normal * (2 * dot);
  }

  /// คำนวณ collision side
  static CollisionSide getCollisionSide(
    Vector2 ballPos,
    double ballRadius,
    Vector2 brickPos,
    Vector2 brickSize,
  ) {
    final ballLeft = ballPos.x - ballRadius;
    final ballRight = ballPos.x + ballRadius;
    final ballTop = ballPos.y - ballRadius;
    final ballBottom = ballPos.y + ballRadius;

    final brickRight = brickPos.x + brickSize.x;
    final brickBottom = brickPos.y + brickSize.y;

    // Calculate overlaps on each side
    final overlapLeft = ballRight - brickPos.x;
    final overlapRight = brickRight - ballLeft;
    final overlapTop = ballBottom - brickPos.y;
    final overlapBottom = brickBottom - ballTop;

    final minOverlap = [overlapLeft, overlapRight, overlapTop, overlapBottom]
        .reduce(min);

    if (minOverlap == overlapLeft) return CollisionSide.left;
    if (minOverlap == overlapRight) return CollisionSide.right;
    if (minOverlap == overlapTop) return CollisionSide.top;
    return CollisionSide.bottom;
  }
}

enum CollisionSide { left, right, top, bottom }

/// Zone-based collision system
class CollisionZone extends PositionComponent {
  final String zoneId;
  final void Function(PositionComponent) onEnter;
  final void Function(PositionComponent) onExit;

  final Set<PositionComponent> _inside = {};

  CollisionZone({
    required Vector2 position,
    required Vector2 size,
    required this.zoneId,
    required this.onEnter,
    required this.onExit,
  }) : super(position: position, size: size);

  void checkComponent(PositionComponent component) {
    final componentBounds = component.toAbsoluteRect();
    final zoneBounds = toAbsoluteRect();

    final isInside = zoneBounds.overlaps(componentBounds);
    final wasInside = _inside.contains(component);

    if (isInside && !wasInside) {
      _inside.add(component);
      onEnter(component);
    } else if (!isInside && wasInside) {
      _inside.remove(component);
      onExit(component);
    }
  }
}
```

---

## ขั้นตอนที่ 2689: Power-ups System

```dart
// lib/games/components/powerup.dart
import 'package:flame/components.dart';
import 'package:flame/collisions.dart';
import 'package:flutter/material.dart';
import 'dart:math';
import '../breakout_game.dart';
import 'paddle.dart';

enum PowerUpType {
  expandPaddle,
  shrinkPaddle,
  extraLife,
  multiball,
  slowBall,
  fastBall,
}

class PowerUp extends CircleComponent with CollisionCallbacks {
  static const double radius = 12;
  static const double fallSpeed = 120;

  final PowerUpType type;
  final _random = Random();

  PowerUp({
    required Vector2 position,
    required this.type,
  }) : super(
          position: position,
          radius: radius,
          anchor: Anchor.center,
        );

  @override
  Future<void> onLoad() async {
    paint = Paint()..color = _getColor();
    add(CircleHitbox());
  }

  @override
  void update(double dt) {
    super.update(dt);
    position.y += fallSpeed * dt;

    // Rotation effect
    angle += dt * 2;

    // Remove if off screen
    if (position.y > 720) {
      removeFromParent();
    }
  }

  @override
  void render(Canvas canvas) {
    super.render(canvas);

    // Draw icon
    final textPaint = TextPaint(
      style: const TextStyle(
        color: Colors.white,
        fontSize: 12,
        fontWeight: FontWeight.bold,
      ),
    );
    textPaint.render(canvas, _getIcon(), Vector2(-5, -6));
  }

  @override
  void onCollisionStart(
    Set<Vector2> intersectionPoints,
    PositionComponent other,
  ) {
    if (other is Paddle) {
      _applyEffect(other);
      removeFromParent();
    }
  }

  void _applyEffect(Paddle paddle) {
    final game = findGame() as BreakoutGame?;
    if (game == null) return;

    switch (type) {
      case PowerUpType.expandPaddle:
        paddle.size.x = (paddle.size.x + 20).clamp(40, 150);
        _scheduleRevert(() => paddle.size.x -= 20, 10);
        break;
      case PowerUpType.shrinkPaddle:
        paddle.size.x = (paddle.size.x - 20).clamp(40, 150);
        _scheduleRevert(() => paddle.size.x += 20, 8);
        break;
      case PowerUpType.extraLife:
        game.lives++;
        game.livesDisplay.updateLives(game.lives);
        break;
      case PowerUpType.slowBall:
        // Handled by ball component
        break;
      case PowerUpType.fastBall:
        // Handled by ball component
        break;
      case PowerUpType.multiball:
        // Spawn additional balls
        break;
    }
  }

  void _scheduleRevert(VoidCallback action, int seconds) {
    Future.delayed(Duration(seconds: seconds), action);
  }

  Color _getColor() {
    switch (type) {
      case PowerUpType.expandPaddle:
        return Colors.green;
      case PowerUpType.shrinkPaddle:
        return Colors.red;
      case PowerUpType.extraLife:
        return Colors.pink;
      case PowerUpType.multiball:
        return Colors.orange;
      case PowerUpType.slowBall:
        return Colors.blue;
      case PowerUpType.fastBall:
        return Colors.purple;
    }
  }

  String _getIcon() {
    switch (type) {
      case PowerUpType.expandPaddle: return '↔';
      case PowerUpType.shrinkPaddle: return '→←';
      case PowerUpType.extraLife: return '♥';
      case PowerUpType.multiball: return '●●';
      case PowerUpType.slowBall: return '↓';
      case PowerUpType.fastBall: return '↑';
    }
  }
}

/// Factory สำหรับสร้าง random power-up
class PowerUpFactory {
  static final _random = Random();

  static PowerUp? tryCreate(Vector2 position, {double chance = 0.15}) {
    if (_random.nextDouble() > chance) return null;

    final types = PowerUpType.values;
    final type = types[_random.nextInt(types.length)];

    return PowerUp(position: position, type: type);
  }
}
```

---

## ขั้นตอนที่ 2690: ตัวอย่าง Game สมบูรณ์พร้อม Effects

```dart
// lib/games/components/effects.dart
import 'package:flame/components.dart';
import 'package:flame/effects.dart';
import 'package:flame/particles.dart';
import 'package:flutter/material.dart';
import 'dart:math';

class GameEffects {
  static final _random = Random();

  /// Explosion effect
  static ParticleSystemComponent explosion({
    required Vector2 position,
    required Color color,
    int particleCount = 20,
  }) {
    return ParticleSystemComponent(
      position: position,
      particle: Particle.generate(
        count: particleCount,
        lifespan: 1.0,
        generator: (i) {
          final angle = _random.nextDouble() * 2 * pi;
          final speed = _random.nextDouble() * 200 + 50;

          return AcceleratedParticle(
            acceleration: Vector2(0, 150), // gravity
            speed: Vector2(
              cos(angle) * speed,
              sin(angle) * speed,
            ),
            child: ComputedParticle(
              renderer: (canvas, particle) {
                final paint = Paint()
                  ..color = color.withOpacity(1 - particle.progress);
                final radius = (1 - particle.progress) * 5 + 2;
                canvas.drawCircle(Offset.zero, radius, paint);
              },
            ),
          );
        },
      ),
    );
  }

  /// Screen flash effect
  static void screenFlash(HasChildren parent, Color color) {
    final flash = RectangleComponent(
      size: Vector2(400, 700),
      paint: Paint()..color = color.withOpacity(0.5),
    );

    parent.add(flash);

    flash.add(
      OpacityEffect.fadeOut(
        EffectController(duration: 0.3),
        onComplete: () => flash.removeFromParent(),
      ),
    );
  }

  /// Score popup effect
  static TextComponent scorePopup({
    required Vector2 position,
    required int points,
    required HasChildren parent,
  }) {
    final text = TextComponent(
      text: '+$points',
      position: position,
      textRenderer: TextPaint(
        style: TextStyle(
          color: points >= 50 ? Colors.yellow : Colors.white,
          fontSize: 16,
          fontWeight: FontWeight.bold,
        ),
      ),
      anchor: Anchor.center,
    );

    text.add(
      SequenceEffect([
        MoveEffect.by(
          Vector2(0, -40),
          EffectController(duration: 0.8),
        ),
        OpacityEffect.fadeOut(
          EffectController(duration: 0.3),
          onComplete: () => text.removeFromParent(),
        ),
      ]),
    );

    parent.add(text);
    return text;
  }

  /// Shake effect (for losing a life)
  static void screenShake(CameraComponent camera, {int intensity = 5}) {
    camera.add(
      MoveEffect.by(
        Vector2(intensity.toDouble(), 0),
        EffectController(
          duration: 0.05,
          reverseDuration: 0.05,
          repeatCount: 4,
        ),
      ),
    );
  }
}

/// Animated background
class StarBackground extends Component with HasGameRef<FlameGame> {
  final List<_Star> _stars = [];
  final _random = Random();

  @override
  Future<void> onLoad() async {
    for (var i = 0; i < 80; i++) {
      _stars.add(_Star(
        position: Vector2(
          _random.nextDouble() * 400,
          _random.nextDouble() * 700,
        ),
        speed: _random.nextDouble() * 20 + 5,
        radius: _random.nextDouble() * 1.5 + 0.5,
        brightness: _random.nextDouble() * 0.8 + 0.2,
      ));
    }
  }

  @override
  void update(double dt) {
    for (final star in _stars) {
      star.position.y += star.speed * dt;
      if (star.position.y > 700) {
        star.position.y = 0;
        star.position.x = _random.nextDouble() * 400;
      }
    }
  }

  @override
  void render(Canvas canvas) {
    for (final star in _stars) {
      final paint = Paint()
        ..color = Colors.white.withOpacity(star.brightness);
      canvas.drawCircle(star.position.toOffset(), star.radius, paint);
    }
  }
}

class _Star {
  Vector2 position;
  final double speed;
  final double radius;
  final double brightness;

  _Star({
    required this.position,
    required this.speed,
    required this.radius,
    required this.brightness,
  });
}

/// ตัวอย่างเรียกใช้งาน game ด้วยทุก features
// void main() {
//   runApp(
//     MaterialApp(
//       debugShowCheckedModeBanner: false,
//       home: GameWidget.controlled(
//         gameFactory: BreakoutGame.new,
//         overlayBuilderMap: {
//           'MainMenu': (ctx, game) =>
//               MainMenuOverlay(game: game as BreakoutGame),
//           'GameOver': (ctx, game) =>
//               GameOverOverlay(game: game as BreakoutGame),
//           'PauseMenu': (ctx, game) =>
//               PauseMenuOverlay(game: game as BreakoutGame),
//         },
//         initialActiveOverlays: const ['MainMenu'],
//       ),
//     ),
//   );
// }
```

---

## ขั้นตอนที่ 2691: Testing the Game

```dart
// test/game_test.dart
import 'package:flame/game.dart';
import 'package:flame_test/flame_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter/material.dart';
import '../lib/games/breakout_game.dart';
import '../lib/games/components/ball.dart';
import '../lib/games/components/paddle.dart';
import '../lib/games/components/brick.dart';

void main() {
  final gameTester = FlameTester(BreakoutGame.new);

  group('BreakoutGame Tests', () {
    gameTester.test(
      'game loads successfully',
      (game) async {
        expect(game.score, equals(0));
        expect(game.lives, equals(3));
        expect(game.level, equals(1));
      },
    );

    gameTester.test(
      'paddle exists',
      (game) async {
        expect(
          game.children.whereType<Paddle>().isNotEmpty,
          isTrue,
        );
      },
    );

    gameTester.test(
      'ball exists',
      (game) async {
        expect(
          game.children.whereType<Ball>().isNotEmpty,
          isTrue,
        );
      },
    );

    gameTester.test(
      'bricks are generated',
      (game) async {
        final bricks = game.children.whereType<Brick>();
        expect(bricks.length, equals(40)); // 5 rows * 8 cols
      },
    );

    gameTester.test(
      'score increases when brick is destroyed',
      (game) async {
        final initialScore = game.score;
        game.addScore(100);
        expect(game.score, equals(initialScore + 100));
      },
    );

    gameTester.test(
      'lives decrease when ball is lost',
      (game) async {
        final initialLives = game.lives;
        game.loseLife();
        expect(game.lives, equals(initialLives - 1));
      },
    );

    gameTester.test(
      'game over when no lives left',
      (game) async {
        game.lives = 1;
        game.loseLife();
        expect(game.lives, equals(0));
        expect(game.isPlaying, isFalse);
      },
    );

    gameTester.test(
      'level advances when all bricks destroyed',
      (game) async {
        final initialLevel = game.level;
        // Remove all bricks
        game.children.whereType<Brick>().toList()
            .forEach((b) => b.removeFromParent());
        await game.ready();
        game.checkLevelComplete();
        expect(game.level, equals(initialLevel + 1));
      },
    );
  });

  group('Ball Tests', () {
    gameTester.test(
      'ball resets to center position',
      (game) async {
        final ball = game.children.whereType<Ball>().first;
        ball.reset();
        expect(ball.position.x, closeTo(200, 0.1));
        expect(ball.position.y, closeTo(350, 0.1));
      },
    );
  });

  group('Collision Helper Tests', () {
    test('circleRect detects collision correctly', () {
      // Circle at (5, 5) radius 5, touching rect at (8, 0) size (10, 10)
      final isColliding = CollisionHelper.circleRect(
        Vector2(5, 5),
        5,
        Vector2(8, 0),
        Vector2(10, 10),
      );
      expect(isColliding, isTrue);
    });

    test('circleRect no collision when apart', () {
      final isColliding = CollisionHelper.circleRect(
        Vector2(0, 0),
        5,
        Vector2(20, 20),
        Vector2(10, 10),
      );
      expect(isColliding, isFalse);
    });
  });
}
```

---

**← [Part 69](part-69-dart-isolate-advanced.md)**
**ต่อไป: [Part 71 →](part-71-flutter-architecture.md)**
