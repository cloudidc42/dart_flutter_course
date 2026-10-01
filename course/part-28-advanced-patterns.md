# Part 28: Advanced Flutter Patterns
## ขั้นตอนที่ 1001-1040

---

## 🎯 เป้าหมายของ Part นี้

- Reactive Programming patterns
- Observer pattern กับ Streams
- Dependency Injection advanced
- Feature flags
- A/B Testing
- Real-world enterprise patterns

---

## ขั้นตอนที่ 1001: Event Bus Pattern

```dart
import 'dart:async';

// ─── Event Bus ───
class EventBus {
  static final EventBus _instance = EventBus._internal();
  factory EventBus() => _instance;
  EventBus._internal();

  final _controller = StreamController.broadcast();

  Stream<T> on<T>() => _controller.stream.where((e) => e is T).cast<T>();
  void fire(dynamic event) => _controller.add(event);
  void dispose() => _controller.close();
}

// ─── Events ───
class UserLoggedInEvent {
  final String userId;
  const UserLoggedInEvent(this.userId);
}

class UserLoggedOutEvent {
  const UserLoggedOutEvent();
}

class CartUpdatedEvent {
  final int itemCount;
  const CartUpdatedEvent(this.itemCount);
}

class NetworkStatusChangedEvent {
  final bool isConnected;
  const NetworkStatusChangedEvent(this.isConnected);
}

// ─── Usage ───
class AppEventHandler {
  final EventBus _bus = EventBus();
  final List<StreamSubscription> _subs = [];

  void init() {
    _subs.add(_bus.on<UserLoggedInEvent>().listen(_onUserLoggedIn));
    _subs.add(_bus.on<UserLoggedOutEvent>().listen(_onUserLoggedOut));
    _subs.add(_bus.on<CartUpdatedEvent>().listen(_onCartUpdated));
  }

  void _onUserLoggedIn(UserLoggedInEvent event) {
    print('User logged in: ${event.userId}');
    // Update analytics, show welcome, etc.
  }

  void _onUserLoggedOut(UserLoggedOutEvent event) {
    print('User logged out');
    // Clear data, navigate to login, etc.
  }

  void _onCartUpdated(CartUpdatedEvent event) {
    print('Cart now has ${event.itemCount} items');
  }

  void dispose() {
    for (StreamSubscription sub in _subs) {
      sub.cancel();
    }
  }
}
```

---

## ขั้นตอนที่ 1002: Repository with Cache Strategy

```dart
import 'dart:async';

enum CacheStrategy { networkFirst, cacheFirst, networkOnly, cacheOnly }

class CachedResult<T> {
  final T data;
  final DateTime fetchedAt;
  final bool fromCache;

  const CachedResult({
    required this.data,
    required this.fetchedAt,
    this.fromCache = false,
  });

  bool get isStale {
    return DateTime.now().difference(fetchedAt) > const Duration(minutes: 5);
  }
}

abstract class CacheableRepository<T, ID> {
  // Network operations
  Future<T> fetchFromNetwork(ID id);
  Future<List<T>> fetchAllFromNetwork();

  // Cache operations
  Future<T?> getFromCache(ID id);
  Future<void> saveToCache(T data);
  Future<void> clearCache();

  // Composite operations
  Future<CachedResult<T>> get(
    ID id, {
    CacheStrategy strategy = CacheStrategy.cacheFirst,
  }) async {
    switch (strategy) {
      case CacheStrategy.networkFirst:
        return _networkFirst(id);
      case CacheStrategy.cacheFirst:
        return _cacheFirst(id);
      case CacheStrategy.networkOnly:
        T data = await fetchFromNetwork(id);
        return CachedResult(data: data, fetchedAt: DateTime.now());
      case CacheStrategy.cacheOnly:
        T? cached = await getFromCache(id);
        if (cached == null) throw Exception('Not in cache');
        return CachedResult(data: cached, fetchedAt: DateTime.now(), fromCache: true);
    }
  }

  Future<CachedResult<T>> _cacheFirst(ID id) async {
    T? cached = await getFromCache(id);
    if (cached != null) {
      return CachedResult(data: cached, fetchedAt: DateTime.now(), fromCache: true);
    }
    T data = await fetchFromNetwork(id);
    await saveToCache(data);
    return CachedResult(data: data, fetchedAt: DateTime.now());
  }

  Future<CachedResult<T>> _networkFirst(ID id) async {
    try {
      T data = await fetchFromNetwork(id).timeout(const Duration(seconds: 5));
      await saveToCache(data);
      return CachedResult(data: data, fetchedAt: DateTime.now());
    } catch (_) {
      T? cached = await getFromCache(id);
      if (cached != null) {
        return CachedResult(data: cached, fetchedAt: DateTime.now(), fromCache: true);
      }
      rethrow;
    }
  }
}
```

---

## ขั้นตอนที่ 1003: Feature Flags

```dart
import 'dart:async';

enum FeatureFlag {
  newCheckoutFlow,
  darkModeV2,
  aiRecommendations,
  betaFeature,
}

class FeatureFlagService {
  static final FeatureFlagService _instance = FeatureFlagService._();
  factory FeatureFlagService() => _instance;
  FeatureFlagService._();

  final Map<FeatureFlag, bool> _flags = {
    FeatureFlag.newCheckoutFlow: false,
    FeatureFlag.darkModeV2: true,
    FeatureFlag.aiRecommendations: false,
    FeatureFlag.betaFeature: false,
  };

  final StreamController<FeatureFlag> _changesController =
      StreamController.broadcast();

  Stream<FeatureFlag> get changes => _changesController.stream;

  bool isEnabled(FeatureFlag flag) => _flags[flag] ?? false;

  void enable(FeatureFlag flag) {
    _flags[flag] = true;
    _changesController.add(flag);
  }

  void disable(FeatureFlag flag) {
    _flags[flag] = false;
    _changesController.add(flag);
  }

  void toggle(FeatureFlag flag) {
    _flags[flag] = !isEnabled(flag);
    _changesController.add(flag);
  }

  // Fetch from remote config
  Future<void> fetchRemoteFlags() async {
    await Future.delayed(const Duration(seconds: 1));
    // In real app: fetch from Firebase Remote Config, LaunchDarkly, etc.
    Map<String, bool> remoteFlags = {
      'newCheckoutFlow': true,
      'aiRecommendations': true,
    };

    remoteFlags.forEach((key, value) {
      FeatureFlag? flag = FeatureFlag.values.where((f) => f.name == key).firstOrNull;
      if (flag != null) {
        _flags[flag] = value;
        _changesController.add(flag);
      }
    });
  }

  void dispose() => _changesController.close();
}

// ─── Feature Gate Widget ───
import 'package:flutter/material.dart';

class FeatureGate extends StatelessWidget {
  final FeatureFlag feature;
  final Widget child;
  final Widget? fallback;

  const FeatureGate({
    super.key,
    required this.feature,
    required this.child,
    this.fallback,
  });

  @override
  Widget build(BuildContext context) {
    bool isEnabled = FeatureFlagService().isEnabled(feature);
    if (isEnabled) return child;
    return fallback ?? const SizedBox.shrink();
  }
}

// ─── Usage ───
class CheckoutScreen extends StatelessWidget {
  const CheckoutScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // ใช้ feature ใหม่ถ้า flag เปิดอยู่
        FeatureGate(
          feature: FeatureFlag.newCheckoutFlow,
          child: const NewCheckoutWidget(),
          fallback: const OldCheckoutWidget(),
        ),

        // FeatureGate สำหรับ AI
        FeatureGate(
          feature: FeatureFlag.aiRecommendations,
          child: const AIRecommendationsWidget(),
        ),
      ],
    );
  }
}

class NewCheckoutWidget extends StatelessWidget {
  const NewCheckoutWidget({super.key});
  @override
  Widget build(BuildContext context) => const Text('New Checkout (2.0)');
}

class OldCheckoutWidget extends StatelessWidget {
  const OldCheckoutWidget({super.key});
  @override
  Widget build(BuildContext context) => const Text('Old Checkout (1.0)');
}

class AIRecommendationsWidget extends StatelessWidget {
  const AIRecommendationsWidget({super.key});
  @override
  Widget build(BuildContext context) => const Text('AI Recommendations');
}
```

---

## ขั้นตอนที่ 1004: Command Pattern

```dart
// Command Pattern: undo/redo operations
abstract class Command {
  Future<void> execute();
  Future<void> undo();
  String get description;
}

class CommandHistory {
  final List<Command> _history = [];
  int _cursor = -1;

  bool get canUndo => _cursor >= 0;
  bool get canRedo => _cursor < _history.length - 1;

  Future<void> execute(Command command) async {
    // Remove redo history
    if (_cursor < _history.length - 1) {
      _history.removeRange(_cursor + 1, _history.length);
    }

    await command.execute();
    _history.add(command);
    _cursor++;
  }

  Future<void> undo() async {
    if (!canUndo) return;
    await _history[_cursor].undo();
    _cursor--;
  }

  Future<void> redo() async {
    if (!canRedo) return;
    _cursor++;
    await _history[_cursor].execute();
  }

  List<String> get historyDescriptions =>
      _history.sublist(0, _cursor + 1).map((c) => c.description).toList();
}

// ─── Concrete Commands ───
class AddItemCommand implements Command {
  final List<String> _list;
  final String _item;

  AddItemCommand(this._list, this._item);

  @override
  Future<void> execute() async => _list.add(_item);

  @override
  Future<void> undo() async => _list.remove(_item);

  @override
  String get description => 'Add "$_item"';
}

class RemoveItemCommand implements Command {
  final List<String> _list;
  final String _item;
  int _removedIndex = -1;

  RemoveItemCommand(this._list, this._item);

  @override
  Future<void> execute() async {
    _removedIndex = _list.indexOf(_item);
    _list.remove(_item);
  }

  @override
  Future<void> undo() async {
    if (_removedIndex >= 0) {
      _list.insert(_removedIndex, _item);
    }
  }

  @override
  String get description => 'Remove "$_item"';
}

// ─── Usage ───
class UndoRedoDemo extends StatefulWidget {
  const UndoRedoDemo({super.key});

  @override
  State<UndoRedoDemo> createState() => _UndoRedoDemoState();
}

class _UndoRedoDemoState extends State<UndoRedoDemo> {
  final List<String> _items = [];
  final CommandHistory _history = CommandHistory();

  Future<void> _addItem(String item) async {
    await _history.execute(AddItemCommand(_items, item));
    setState(() {});
  }

  Future<void> _removeItem(String item) async {
    await _history.execute(RemoveItemCommand(_items, item));
    setState(() {});
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Row(
          children: [
            ElevatedButton(
              onPressed: _history.canUndo
                  ? () async {
                      await _history.undo();
                      setState(() {});
                    }
                  : null,
              child: const Text('Undo'),
            ),
            ElevatedButton(
              onPressed: _history.canRedo
                  ? () async {
                      await _history.redo();
                      setState(() {});
                    }
                  : null,
              child: const Text('Redo'),
            ),
          ],
        ),
        ...(_items.map((item) => ListTile(
          title: Text(item),
          trailing: IconButton(
            icon: const Icon(Icons.delete),
            onPressed: () => _removeItem(item),
          ),
        ))),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 1005: Interceptor Chain Pattern

```dart
abstract class Interceptor<T> {
  Future<T?> intercept(T request, Future<T?> Function(T) next);
}

class InterceptorChain<T> {
  final List<Interceptor<T>> _interceptors = [];

  void add(Interceptor<T> interceptor) => _interceptors.add(interceptor);

  Future<T?> execute(T request) {
    Future<T?> chain(int index, T req) {
      if (index >= _interceptors.length) return Future.value(null);
      return _interceptors[index].intercept(req, (r) => chain(index + 1, r));
    }
    return chain(0, request);
  }
}

// ─── HTTP Request ───
class HttpRequest {
  final String url;
  final Map<String, String> headers;
  final Map<String, dynamic>? body;

  HttpRequest({required this.url, this.headers = const {}, this.body});

  HttpRequest copyWith({Map<String, String>? headers}) {
    return HttpRequest(
      url: url,
      headers: headers ?? this.headers,
      body: body,
    );
  }
}

// ─── Interceptors ───
class LoggingInterceptor implements Interceptor<HttpRequest> {
  @override
  Future<HttpRequest?> intercept(
    HttpRequest request,
    Future<HttpRequest?> Function(HttpRequest) next,
  ) async {
    print('→ ${request.url}');
    HttpRequest? result = await next(request);
    print('← ${request.url}');
    return result;
  }
}

class AuthInterceptor implements Interceptor<HttpRequest> {
  final String token;
  AuthInterceptor(this.token);

  @override
  Future<HttpRequest?> intercept(
    HttpRequest request,
    Future<HttpRequest?> Function(HttpRequest) next,
  ) async {
    HttpRequest withAuth = request.copyWith(headers: {
      ...request.headers,
      'Authorization': 'Bearer $token',
    });
    return next(withAuth);
  }
}
```

---

**← [Part 27 - CI/CD](part-27-cicd.md)**

**ต่อไป: [Part 29 - Enterprise Architecture →](part-29-enterprise.md)**
