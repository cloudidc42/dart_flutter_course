# Part 85: Dart Concurrency Advanced
## ขั้นตอนที่ 3281-3320

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ Dart Concurrency Model อย่างลึกซึ้ง
- ใช้งาน Zone และ Error Handling ขั้นสูง
- สร้าง Async Generators (async*, yield*)
- สร้าง Stream Transformers
- ใช้ RxDart สำหรับ Reactive Programming

---

## ขั้นตอนที่ 3281: Dart Concurrency Model

```dart
// bin/01_concurrency_model.dart
import 'dart:async';
import 'dart:isolate';

/// Dart's concurrency model:
/// - Single-threaded event loop
/// - Isolates for true parallelism
/// - Futures and Streams for async operations
/// - No shared memory between isolates (message passing)

void main() async {
  print('=== Dart Concurrency Model ===\n');
  
  await demonstrateEventLoop();
  await demonstrateIsolates();
  await demonstrateMicrotasks();
}

/// Demonstrates the event loop behavior
Future<void> demonstrateEventLoop() async {
  print('--- Event Loop Demo ---');
  
  // Synchronous code runs first
  print('1. Synchronous code');
  
  // Microtasks run before macrotasks
  scheduleMicrotask(() => print('3. Microtask'));
  
  // Futures are macrotasks
  Future(() => print('5. Future (macrotask)'));
  
  // Timer is a macrotask
  Timer(Duration.zero, () => print('6. Timer (macrotask)'));
  
  // Async/await uses microtask queue internally
  await Future.microtask(() => print('4. Future.microtask'));
  
  print('2. Synchronous code after scheduling');
  
  // Give macrotasks time to run
  await Future.delayed(const Duration(milliseconds: 10));
  print('');
}

/// Demonstrates Isolate communication
Future<void> demonstrateIsolates() async {
  print('--- Isolate Demo ---');
  
  // Simple isolate with compute
  final result = await Isolate.run(() {
    // This runs in a separate isolate
    var sum = 0;
    for (var i = 0; i < 1000000; i++) {
      sum += i;
    }
    return sum;
  });
  print('Sum computed in isolate: $result');
  
  // Two-way communication with ReceivePort
  final receivePort = ReceivePort();
  final isolate = await Isolate.spawn(
    _isolateWorker,
    receivePort.sendPort,
  );
  
  // Listen for messages
  final messages = <String>[];
  await for (final message in receivePort) {
    if (message is String) {
      messages.add(message);
      if (message == 'DONE') break;
    }
  }
  
  print('Messages from isolate: $messages');
  isolate.kill();
  receivePort.close();
  print('');
}

void _isolateWorker(SendPort sendPort) {
  sendPort.send('Hello from isolate!');
  sendPort.send('Processing...');
  
  // Simulate work
  var result = 0;
  for (var i = 0; i < 100; i++) result += i;
  
  sendPort.send('Result: $result');
  sendPort.send('DONE');
}

/// Demonstrates microtask scheduling priority
Future<void> demonstrateMicrotasks() async {
  print('--- Microtask Priority Demo ---');
  
  final order = <int>[];
  
  // Future completes in microtask queue
  final future1 = Future.value(1);
  final future2 = Future.value(2);
  
  future1.then((v) => order.add(v));
  future2.then((v) => order.add(v));
  
  scheduleMicrotask(() => order.add(0)); // Runs before futures!
  
  await Future.delayed(Duration.zero);
  print('Order of execution: $order'); // [0, 1, 2]
  print('');
}
```

---

## ขั้นตอนที่ 3282: Zone dan Error Handling

```dart
// bin/02_zones.dart
import 'dart:async';

void main() async {
  print('=== Zones and Error Handling ===\n');
  
  await demonstrateBasicZone();
  await demonstrateZoneErrorHandling();
  await demonstrateZoneValues();
  demonstrateRunZonedGuarded();
}

/// Basic zone usage
Future<void> demonstrateBasicZone() async {
  print('--- Basic Zone ---');
  
  // Zones intercept async operations
  await runZoned(
    () async {
      print('Inside zone');
      
      // All async operations in this zone are tracked
      await Future.delayed(const Duration(milliseconds: 10));
      Timer(const Duration(milliseconds: 5), () {
        print('Timer in zone');
      });
      
      await Future.delayed(const Duration(milliseconds: 20));
    },
    zoneSpecification: ZoneSpecification(
      scheduleMicrotask: (self, parent, zone, microtask) {
        print('Microtask scheduled in zone');
        parent.scheduleMicrotask(zone, microtask);
      },
      createTimer: (self, parent, zone, duration, callback) {
        print('Timer created in zone: ${duration.inMilliseconds}ms');
        return parent.createTimer(zone, duration, callback);
      },
    ),
  );
  
  print('');
}

/// Zone-based error handling
Future<void> demonstrateZoneErrorHandling() async {
  print('--- Zone Error Handling ---');
  
  final errors = <Object>[];
  
  await runZoned(
    () async {
      // This error will be caught by the zone
      scheduleMicrotask(() => throw Exception('Microtask error'));
      
      // Timer error
      Timer(const Duration(milliseconds: 5), () {
        throw Exception('Timer error');
      });
      
      await Future.delayed(const Duration(milliseconds: 50));
    },
    onError: (error, stackTrace) {
      errors.add(error);
      print('Caught in zone: $error');
    },
  );
  
  print('Total errors caught: ${errors.length}');
  print('');
}

/// Zone values (zone-local storage)
Future<void> demonstrateZoneValues() async {
  print('--- Zone Values ---');
  
  // Zone values act like thread-local storage
  const requestIdKey = #requestId;
  const userKey = #currentUser;
  
  await runZoned(
    () async {
      final requestId = Zone.current[requestIdKey] as String;
      final user = Zone.current[userKey] as String;
      print('Processing request $requestId for user $user');
      
      // Nested zone inherits values
      await runZoned(
        () async {
          final inheritedId = Zone.current[requestIdKey] as String;
          print('Nested zone sees request: $inheritedId');
          
          // Override for nested zone
          await runZoned(
            () async {
              final overridden = Zone.current[requestIdKey] as String;
              print('Overridden request: $overridden');
            },
            zoneValues: {requestIdKey: 'req_999'},
          );
        },
      );
    },
    zoneValues: {
      requestIdKey: 'req_123',
      userKey: 'alice@example.com',
    },
  );
  
  print('');
}

/// runZonedGuarded for global error handling
void demonstrateRunZonedGuarded() {
  print('--- runZonedGuarded ---');
  
  runZonedGuarded(
    () {
      // This simulates an application entry point
      _runApp();
    },
    (error, stackTrace) {
      // Global error handler - log to crash reporting service
      print('CRITICAL ERROR: $error');
      // In production: crashlytics.recordError(error, stackTrace)
    },
  );
}

void _runApp() {
  print('App started in protected zone');
  
  // Simulate unhandled async error
  Future.delayed(const Duration(milliseconds: 10), () {
    throw Exception('Unhandled app error');
  });
}
```

---

## ขั้นตอนที่ 3283: Async Generators (async*, yield, yield*)

```dart
// bin/03_async_generators.dart
import 'dart:async';
import 'dart:math';

void main() async {
  print('=== Async Generators ===\n');
  
  await demonstrateAsyncGenerator();
  await demonstrateYieldStar();
  await demonstrateSyncGenerator();
  await demonstratePaginatedStream();
  await demonstrateInfiniteStream();
}

/// Basic async* generator
Stream<int> countUp(int from, int to) async* {
  for (var i = from; i <= to; i++) {
    await Future.delayed(const Duration(milliseconds: 50));
    yield i;
  }
}

/// Async generator with error handling
Stream<String> processItems(List<String> items) async* {
  for (final item in items) {
    try {
      await Future.delayed(const Duration(milliseconds: 20));
      if (item.startsWith('bad_')) {
        throw Exception('Bad item: $item');
      }
      yield 'Processed: $item';
    } catch (e) {
      yield 'Error: $e';
    }
  }
}

/// Demonstrates basic async* usage
Future<void> demonstrateAsyncGenerator() async {
  print('--- Basic async* Generator ---');
  
  await for (final value in countUp(1, 5)) {
    print('Received: $value');
  }
  
  print('');
  
  // Process with error handling
  final items = ['item1', 'item2', 'bad_item3', 'item4'];
  await for (final result in processItems(items)) {
    print(result);
  }
  
  print('');
}

/// yield* delegates to another stream/iterable
Stream<int> fibonacciStream(int count) async* {
  var a = 0, b = 1;
  for (var i = 0; i < count; i++) {
    yield a;
    final temp = b;
    b = a + b;
    a = temp;
    await Future.delayed(const Duration(milliseconds: 10));
  }
}

Stream<String> decoratedFibonacci(int count) async* {
  yield 'Starting Fibonacci series:';
  
  // yield* delegates all values from another stream
  yield* fibonacciStream(count).map((n) => '  F = $n');
  
  yield 'Fibonacci series complete!';
}

Future<void> demonstrateYieldStar() async {
  print('--- yield* Delegation ---');
  
  await for (final msg in decoratedFibonacci(8)) {
    print(msg);
  }
  
  print('');
}

/// Synchronous generator with yield
Iterable<int> range(int start, int end, {int step = 1}) sync* {
  for (var i = start; i < end; i += step) {
    yield i;
  }
}

Iterable<List<T>> chunked<T>(Iterable<T> source, int size) sync* {
  final iter = source.iterator;
  while (true) {
    final chunk = <T>[];
    for (var i = 0; i < size; i++) {
      if (!iter.moveNext()) {
        if (chunk.isNotEmpty) yield chunk;
        return;
      }
      chunk.add(iter.current);
    }
    yield chunk;
  }
}

Future<void> demonstrateSyncGenerator() async {
  print('--- sync* Generator ---');
  
  // Range generator
  print('Range(0, 10, step: 2): ${range(0, 10, step: 2).toList()}');
  
  // Chunked generator
  final numbers = range(1, 11).toList();
  for (final chunk in chunked(numbers, 3)) {
    print('Chunk: $chunk');
  }
  
  // Lazy evaluation - only generates needed values
  final lazyRange = range(0, 1000000);
  print('First 5 of 1M: ${lazyRange.take(5).toList()}');
  
  print('');
}

/// Simulated paginated API stream
Stream<List<Map<String, dynamic>>> paginatedStream({
  required int totalItems,
  int pageSize = 5,
}) async* {
  var page = 0;
  var fetched = 0;

  while (fetched < totalItems) {
    // Simulate network request
    await Future.delayed(const Duration(milliseconds: 100));
    
    final count = min(pageSize, totalItems - fetched);
    final items = List.generate(count, (i) => {
      'id': fetched + i + 1,
      'name': 'Item ${fetched + i + 1}',
      'page': page + 1,
    });
    
    fetched += count;
    page++;
    
    yield items;
    
    if (fetched >= totalItems) break;
  }
}

Future<void> demonstratePaginatedStream() async {
  print('--- Paginated Stream ---');
  
  var allItems = <Map<String, dynamic>>[];
  var pageCount = 0;
  
  await for (final page in paginatedStream(totalItems: 13, pageSize: 5)) {
    pageCount++;
    allItems.addAll(page);
    print('Page $pageCount: ${page.map((i) => i['id']).toList()}');
  }
  
  print('Total items fetched: ${allItems.length}');
  print('');
}

/// Infinite stream with cancellation
Stream<double> sensorStream() async* {
  final random = Random();
  while (true) {
    await Future.delayed(const Duration(milliseconds: 100));
    // Simulate sensor reading (temperature)
    yield 20.0 + random.nextDouble() * 10;
  }
}

Future<void> demonstrateInfiniteStream() async {
  print('--- Infinite Stream with Cancellation ---');
  
  // Take only first 5 readings
  final readings = await sensorStream().take(5).toList();
  print('Sensor readings: ${readings.map((r) => r.toStringAsFixed(2)).join(', ')}');
  
  // Process with timeout
  try {
    await for (final reading in sensorStream().timeout(
      const Duration(milliseconds: 350),
    )) {
      print('Live reading: ${reading.toStringAsFixed(2)}°C');
    }
  } on TimeoutException {
    print('Stream timed out (expected)');
  }
  
  print('');
}
```

---

## ขั้นตอนที่ 3284: Stream Transformers

```dart
// bin/04_stream_transformers.dart
import 'dart:async';

void main() async {
  print('=== Stream Transformers ===\n');
  
  await demonstrateBuiltInTransformers();
  await demonstrateCustomTransformer();
  await demonstrateThrottleTransformer();
  await demonstrateRetryTransformer();
  await demonstrateScanTransformer();
}

/// Built-in stream transformers
Future<void> demonstrateBuiltInTransformers() async {
  print('--- Built-in Transformers ---');
  
  final stream = Stream.fromIterable([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);
  
  // map + filter + take
  final result = await stream
      .map((x) => x * x)                    // Square each
      .where((x) => x % 2 == 0)             // Only even squares
      .take(3)                               // Take first 3
      .toList();
  
  print('map+filter+take: $result');
  
  // expand (flatMap for synchronous)
  final expanded = await Stream.fromIterable([1, 2, 3])
      .expand((x) => [x, x * 10])
      .toList();
  print('expand: $expanded');
  
  // asyncMap for async operations
  final asyncMapped = await Stream.fromIterable([1, 2, 3])
      .asyncMap((x) async {
        await Future.delayed(const Duration(milliseconds: 10));
        return x * 100;
      })
      .toList();
  print('asyncMap: $asyncMapped');
  
  // asyncExpand for async flatMap
  final asyncExpanded = await Stream.fromIterable([1, 2, 3])
      .asyncExpand((x) async* {
        yield x;
        await Future.delayed(const Duration(milliseconds: 5));
        yield x * 10;
      })
      .toList();
  print('asyncExpand: $asyncExpanded');
  
  // distinct - remove consecutive duplicates
  final withDuplicates = Stream.fromIterable([1, 1, 2, 2, 2, 3, 1, 1]);
  final distinct = await withDuplicates.distinct().toList();
  print('distinct: $distinct');
  
  print('');
}

/// Custom StreamTransformer
class BatchTransformer<T> extends StreamTransformerBase<T, List<T>> {
  final int batchSize;
  final Duration maxWait;

  const BatchTransformer({
    required this.batchSize,
    this.maxWait = const Duration(milliseconds: 100),
  });

  @override
  Stream<List<T>> bind(Stream<T> stream) {
    late StreamController<List<T>> controller;
    late StreamSubscription<T> subscription;
    final batch = <T>[];
    Timer? timer;

    void flush() {
      if (batch.isNotEmpty) {
        controller.add(List.from(batch));
        batch.clear();
      }
      timer?.cancel();
      timer = null;
    }

    controller = StreamController<List<T>>(
      onListen: () {
        subscription = stream.listen(
          (event) {
            batch.add(event);

            if (batch.length >= batchSize) {
              flush();
            } else {
              // Start/reset the timeout
              timer?.cancel();
              timer = Timer(maxWait, flush);
            }
          },
          onError: controller.addError,
          onDone: () {
            flush();
            controller.close();
          },
        );
      },
      onCancel: () {
        timer?.cancel();
        subscription.cancel();
      },
    );

    return controller.stream;
  }
}

Future<void> demonstrateCustomTransformer() async {
  print('--- Custom Batch Transformer ---');
  
  // Create a stream that emits items slowly
  Stream<int> slowStream() async* {
    for (var i = 1; i <= 12; i++) {
      yield i;
      await Future.delayed(const Duration(milliseconds: 30));
    }
  }
  
  final batches = await slowStream()
      .transform(BatchTransformer(batchSize: 3))
      .toList();
  
  print('Batches of 3: $batches');
  print('');
}

/// Throttle transformer - only emit most recent within time window
class ThrottleTransformer<T> extends StreamTransformerBase<T, T> {
  final Duration duration;
  final bool trailing; // emit last value at end of window

  const ThrottleTransformer(this.duration, {this.trailing = false});

  @override
  Stream<T> bind(Stream<T> stream) {
    late StreamController<T> controller;
    late StreamSubscription<T> subscription;
    
    T? lastValue;
    bool hasValue = false;
    bool throttled = false;
    Timer? trailingTimer;

    controller = StreamController<T>(
      onListen: () {
        subscription = stream.listen(
          (event) {
            lastValue = event;
            hasValue = true;

            if (!throttled) {
              throttled = true;
              controller.add(event);

              Timer(duration, () {
                throttled = false;
                if (trailing && hasValue) {
                  controller.add(lastValue as T);
                  hasValue = false;
                }
              });
            }
          },
          onError: controller.addError,
          onDone: () {
            trailingTimer?.cancel();
            controller.close();
          },
        );
      },
      onCancel: () {
        trailingTimer?.cancel();
        subscription.cancel();
      },
    );

    return controller.stream;
  }
}

/// Debounce transformer - only emit after silence period
class DebounceTransformer<T> extends StreamTransformerBase<T, T> {
  final Duration duration;

  const DebounceTransformer(this.duration);

  @override
  Stream<T> bind(Stream<T> stream) {
    late StreamController<T> controller;
    late StreamSubscription<T> subscription;
    Timer? debounceTimer;

    controller = StreamController<T>(
      onListen: () {
        subscription = stream.listen(
          (event) {
            debounceTimer?.cancel();
            debounceTimer = Timer(duration, () => controller.add(event));
          },
          onError: controller.addError,
          onDone: () {
            debounceTimer?.cancel();
            controller.close();
          },
        );
      },
      onCancel: () {
        debounceTimer?.cancel();
        subscription.cancel();
      },
    );

    return controller.stream;
  }
}

Future<void> demonstrateThrottleTransformer() async {
  print('--- Throttle & Debounce Transformers ---');
  
  // Simulate rapid events (like user typing)
  Stream<String> typeStream() async* {
    final keys = ['H', 'He', 'Hel', 'Hell', 'Hello', 'Hello ', 'Hello W',
                  'Hello Wo', 'Hello Wor', 'Hello Worl', 'Hello World'];
    for (final key in keys) {
      yield key;
      await Future.delayed(const Duration(milliseconds: 50));
    }
  }
  
  // Throttle: emit at most once per 200ms
  final throttled = <String>[];
  await typeStream()
      .transform(ThrottleTransformer(const Duration(milliseconds: 200)))
      .forEach(throttled.add);
  print('Throttled emissions: $throttled');
  
  // Debounce: only emit after 200ms silence
  final debounced = <String>[];
  await typeStream()
      .transform(DebounceTransformer(const Duration(milliseconds: 200)))
      .forEach(debounced.add);
  print('Debounced emissions: $debounced');
  
  print('');
}

/// Retry transformer
class RetryTransformer<T> extends StreamTransformerBase<T, T> {
  final int maxRetries;
  final Duration delay;

  const RetryTransformer({
    this.maxRetries = 3,
    this.delay = const Duration(seconds: 1),
  });

  @override
  Stream<T> bind(Stream<T> stream) {
    return _retryStream(stream, maxRetries, delay);
  }

  Stream<T> _retryStream(Stream<T> source, int retries, Duration delay) async* {
    var attempt = 0;
    while (true) {
      try {
        await for (final event in source) {
          yield event;
        }
        return; // Completed successfully
      } catch (e) {
        attempt++;
        if (attempt > retries) rethrow;

        print('Attempt $attempt failed: $e. Retrying in ${delay.inMilliseconds}ms...');
        await Future.delayed(delay);
      }
    }
  }
}

Future<void> demonstrateRetryTransformer() async {
  print('--- Retry Transformer ---');
  
  var callCount = 0;
  
  Stream<String> unreliableStream() async* {
    callCount++;
    if (callCount < 3) {
      throw Exception('Simulated failure (attempt $callCount)');
    }
    yield 'Success on attempt $callCount!';
  }
  
  try {
    await for (final msg in unreliableStream()
        .transform(RetryTransformer(maxRetries: 3, delay: const Duration(milliseconds: 100)))) {
      print('Result: $msg');
    }
  } catch (e) {
    print('Final error: $e');
  }
  
  print('');
}

/// Scan transformer (like reduce but emits intermediate values)
extension StreamExtensions<T> on Stream<T> {
  Stream<R> scan<R>(R initialValue, R Function(R accumulator, T element) combine) async* {
    var accumulator = initialValue;
    yield accumulator;
    await for (final element in this) {
      accumulator = combine(accumulator, element);
      yield accumulator;
    }
  }
}

Future<void> demonstrateScanTransformer() async {
  print('--- Scan Transformer ---');
  
  // Running sum
  final runningSum = await Stream.fromIterable([1, 2, 3, 4, 5])
      .scan(0, (acc, x) => acc + x)
      .toList();
  print('Running sum: $runningSum');
  
  // Running maximum
  final runningMax = await Stream.fromIterable([3, 1, 4, 1, 5, 9, 2, 6])
      .scan(0, (acc, x) => x > acc ? x : acc)
      .toList();
  print('Running max: $runningMax');
  
  // Cumulative product
  final cumulativeProduct = await Stream.fromIterable([1, 2, 3, 4, 5])
      .scan(1, (acc, x) => acc * x)
      .toList();
  print('Cumulative product: $cumulativeProduct');
  
  print('');
}
```

---

## ขั้นตอนที่ 3285: RxDart สำหรับ Reactive Programming

```dart
// bin/05_rxdart.dart
import 'dart:async';
// NOTE: Add rxdart: ^0.27.7 to pubspec.yaml
// import 'package:rxdart/rxdart.dart';

/// Since we can't import rxdart here, we'll implement
/// similar functionality to demonstrate RxDart concepts

void main() async {
  print('=== RxDart-Style Reactive Programming ===\n');
  
  await demonstrateBehaviorSubject();
  await demonstratePublishSubject();
  await demonstrateReplaySubject();
  await demonstrateCombineLatest();
  await demonstrateMergeStreams();
  await demonstrateZipStreams();
}

/// BehaviorSubject: emits current value to new subscribers
class BehaviorSubject<T> extends Stream<T> implements Sink<T> {
  T? _latestValue;
  bool _hasValue = false;
  final StreamController<T> _controller = StreamController.broadcast();

  BehaviorSubject([T? seed]) {
    if (seed != null) {
      _latestValue = seed;
      _hasValue = true;
    }
  }

  T get value {
    if (!_hasValue) throw StateError('No value yet');
    return _latestValue as T;
  }

  bool get hasValue => _hasValue;

  @override
  void add(T event) {
    _latestValue = event;
    _hasValue = true;
    _controller.add(event);
  }

  @override
  void close() => _controller.close();

  @override
  StreamSubscription<T> listen(
    void Function(T event)? onData, {
    Function? onError,
    void Function()? onDone,
    bool? cancelOnError,
  }) {
    final subscription = _controller.stream.listen(
      onData,
      onError: onError,
      onDone: onDone,
      cancelOnError: cancelOnError,
    );

    // Emit current value to new subscriber
    if (_hasValue && onData != null) {
      onData(_latestValue as T);
    }

    return subscription;
  }
}

Future<void> demonstrateBehaviorSubject() async {
  print('--- BehaviorSubject ---');
  
  final subject = BehaviorSubject<int>(0);
  
  // First subscriber
  subject.listen((v) => print('Subscriber 1: $v'));
  
  subject.add(1);
  subject.add(2);
  subject.add(3);
  
  // Second subscriber - receives current value immediately
  await Future.delayed(const Duration(milliseconds: 10));
  subject.listen((v) => print('Subscriber 2 (late): $v'));
  
  subject.add(4);
  
  await Future.delayed(const Duration(milliseconds: 10));
  print('Current value: ${subject.value}');
  subject.close();
  
  print('');
}

/// PublishSubject: only emits to active subscribers
class PublishSubject<T> extends Stream<T> implements Sink<T> {
  final StreamController<T> _controller = StreamController.broadcast();

  @override
  void add(T event) => _controller.add(event);

  @override
  void close() => _controller.close();

  @override
  StreamSubscription<T> listen(
    void Function(T event)? onData, {
    Function? onError,
    void Function()? onDone,
    bool? cancelOnError,
  }) {
    return _controller.stream.listen(
      onData,
      onError: onError,
      onDone: onDone,
      cancelOnError: cancelOnError,
    );
  }
}

Future<void> demonstratePublishSubject() async {
  print('--- PublishSubject ---');
  
  final subject = PublishSubject<String>();
  
  subject.add('Before subscriber'); // Not received by anyone
  
  final sub1 = subject.listen((v) => print('Sub 1: $v'));
  
  subject.add('Event 1'); // Received by sub1
  subject.add('Event 2'); // Received by sub1
  
  final sub2 = subject.listen((v) => print('Sub 2: $v'));
  
  subject.add('Event 3'); // Received by both
  
  sub1.cancel();
  subject.add('Event 4'); // Only received by sub2
  
  await Future.delayed(const Duration(milliseconds: 10));
  sub2.cancel();
  subject.close();
  
  print('');
}

/// ReplaySubject: replays last N events to new subscribers
class ReplaySubject<T> extends Stream<T> implements Sink<T> {
  final int maxSize;
  final List<T> _buffer = [];
  final StreamController<T> _controller = StreamController.broadcast();

  ReplaySubject({this.maxSize = 0}); // 0 = unlimited

  @override
  void add(T event) {
    _buffer.add(event);
    if (maxSize > 0 && _buffer.length > maxSize) {
      _buffer.removeAt(0);
    }
    _controller.add(event);
  }

  @override
  void close() => _controller.close();

  @override
  StreamSubscription<T> listen(
    void Function(T event)? onData, {
    Function? onError,
    void Function()? onDone,
    bool? cancelOnError,
  }) {
    final subscription = _controller.stream.listen(
      onData,
      onError: onError,
      onDone: onDone,
      cancelOnError: cancelOnError,
    );

    // Replay buffered events
    if (onData != null) {
      for (final event in List.from(_buffer)) {
        onData(event);
      }
    }

    return subscription;
  }
}

Future<void> demonstrateReplaySubject() async {
  print('--- ReplaySubject ---');
  
  final subject = ReplaySubject<int>(maxSize: 3);
  
  subject.add(1);
  subject.add(2);
  subject.add(3);
  subject.add(4);
  subject.add(5);
  
  // Late subscriber gets last 3 events
  subject.listen((v) => print('Late subscriber: $v'));
  
  await Future.delayed(const Duration(milliseconds: 10));
  subject.close();
  
  print('');
}

/// CombineLatest: combines latest values from multiple streams
Stream<List<T>> combineLatest<T>(List<Stream<T>> streams) async* {
  final latestValues = List<T?>.filled(streams.length, null);
  final hasValues = List<bool>.filled(streams.length, false);
  
  final controller = StreamController<List<T>>();
  
  final subscriptions = <StreamSubscription<T>>[];
  
  for (var i = 0; i < streams.length; i++) {
    final index = i;
    subscriptions.add(streams[i].listen(
      (value) {
        latestValues[index] = value;
        hasValues[index] = true;
        
        if (hasValues.every((h) => h)) {
          controller.add(latestValues.cast<T>().toList());
        }
      },
      onError: controller.addError,
    ));
  }
  
  Future.wait(subscriptions.map((s) => s.asFuture())).then((_) {
    controller.close();
  });
  
  yield* controller.stream;
}

Future<void> demonstrateCombineLatest() async {
  print('--- CombineLatest ---');
  
  Stream<int> stream1() async* {
    yield 1;
    await Future.delayed(const Duration(milliseconds: 50));
    yield 2;
    await Future.delayed(const Duration(milliseconds: 50));
    yield 3;
  }
  
  Stream<String> stream2() async* {
    await Future.delayed(const Duration(milliseconds: 25));
    yield 'a';
    await Future.delayed(const Duration(milliseconds: 50));
    yield 'b';
  }
  
  // Type must be same for this simple implementation
  // In real RxDart, CombineLatest supports different types
  await for (final values in combineLatest<dynamic>([stream1(), stream2()])) {
    print('CombineLatest: $values');
  }
  
  print('');
}

/// Merge streams
Stream<T> mergeStreams<T>(List<Stream<T>> streams) async* {
  final controller = StreamController<T>();
  var activeStreams = streams.length;
  
  for (final stream in streams) {
    stream.listen(
      controller.add,
      onError: controller.addError,
      onDone: () {
        activeStreams--;
        if (activeStreams == 0) controller.close();
      },
    );
  }
  
  yield* controller.stream;
}

Future<void> demonstrateMergeStreams() async {
  print('--- Merge Streams ---');
  
  Stream<String> stream1() async* {
    yield '1-A';
    await Future.delayed(const Duration(milliseconds: 30));
    yield '1-B';
    await Future.delayed(const Duration(milliseconds: 30));
    yield '1-C';
  }
  
  Stream<String> stream2() async* {
    await Future.delayed(const Duration(milliseconds: 15));
    yield '2-X';
    await Future.delayed(const Duration(milliseconds: 30));
    yield '2-Y';
  }
  
  await for (final event in mergeStreams([stream1(), stream2()])) {
    print('Merged: $event');
  }
  
  print('');
}

/// Zip streams
Stream<List<T>> zipStreams<T>(List<Stream<T>> streams) async* {
  final iterators = streams.map((s) => StreamIterator(s)).toList();
  
  while (true) {
    final nexts = await Future.wait(iterators.map((i) => i.moveNext()));
    if (nexts.any((n) => !n)) break;
    
    yield iterators.map((i) => i.current).toList();
  }
  
  for (final iterator in iterators) {
    await iterator.cancel();
  }
}

Future<void> demonstrateZipStreams() async {
  print('--- Zip Streams ---');
  
  final stream1 = Stream.fromIterable([1, 2, 3]);
  final stream2 = Stream.fromIterable(['a', 'b', 'c', 'd']); // Extra 'd' ignored
  final stream3 = Stream.fromIterable([true, false, true]);
  
  await for (final zipped in zipStreams<dynamic>([stream1, stream2, stream3])) {
    print('Zipped: $zipped');
  }
  
  print('');
}
```

---

## ขั้นตอนที่ 3286: Advanced RxDart Patterns

```dart
// bin/06_rxdart_patterns.dart
import 'dart:async';

void main() async {
  print('=== Advanced Reactive Patterns ===\n');
  
  await demonstrateSwitchMap();
  await demonstrateFlatMap();
  await demonstrateBufferAndWindow();
  await demonstrateShareStream();
  await demonstrateRealWorldExample();
}

/// SwitchMap: cancels previous inner stream on new outer event
Stream<String> switchMap<T>(
  Stream<T> source,
  Stream<String> Function(T) mapper,
) async* {
  final controller = StreamController<String>();
  StreamSubscription<String>? innerSub;
  
  source.listen(
    (event) {
      innerSub?.cancel();
      innerSub = mapper(event).listen(
        controller.add,
        onError: controller.addError,
      );
    },
    onError: controller.addError,
    onDone: () async {
      await innerSub?.asFuture();
      controller.close();
    },
  );
  
  yield* controller.stream;
}

Future<void> demonstrateSwitchMap() async {
  print('--- SwitchMap (like search autocomplete) ---');
  
  // Simulate search queries
  Stream<String> searchQueries() async* {
    yield 'd';
    await Future.delayed(const Duration(milliseconds: 50));
    yield 'da';
    await Future.delayed(const Duration(milliseconds: 50));
    yield 'dar'; // Previous search cancelled
    await Future.delayed(const Duration(milliseconds: 100));
    yield 'dart';
  }
  
  // Simulate API search - "dar" search will be cancelled before completing
  Stream<String> searchAPI(String query) async* {
    print('  Starting search for: "$query"');
    await Future.delayed(const Duration(milliseconds: 80));
    yield '  Result for "$query": ${query.toUpperCase()}_RESULT';
  }
  
  await for (final result in switchMap(searchQueries(), searchAPI)) {
    print(result);
  }
  
  print('');
}

/// FlatMap (concatMap): maintains order, waits for each inner stream
Stream<R> flatMapConcurrent<T, R>(
  Stream<T> source,
  Stream<R> Function(T) mapper,
  int concurrency,
) async* {
  final activeStreams = <Stream<R>>[];
  final buffer = <T>[];
  var sourceComplete = false;

  final controller = StreamController<R>();
  
  source.listen(
    (event) {
      if (activeStreams.length < concurrency) {
        _subscribeToInner(mapper(event), activeStreams, controller, () {
          if (buffer.isNotEmpty) {
            final next = buffer.removeAt(0);
            _subscribeToInner(mapper(next), activeStreams, controller, () {
              if (sourceComplete && activeStreams.isEmpty && buffer.isEmpty) {
                controller.close();
              }
            });
          } else if (sourceComplete && activeStreams.isEmpty) {
            controller.close();
          }
        });
      } else {
        buffer.add(event);
      }
    },
    onDone: () {
      sourceComplete = true;
      if (activeStreams.isEmpty && buffer.isEmpty) controller.close();
    },
  );
  
  yield* controller.stream;
}

void _subscribeToInner<T>(
  Stream<T> stream,
  List<Stream<T>> activeStreams,
  StreamController<T> controller,
  VoidCallback onDone,
) {
  activeStreams.add(stream);
  stream.listen(
    controller.add,
    onDone: () {
      activeStreams.remove(stream);
      onDone();
    },
  );
}

typedef VoidCallback = void Function();

Future<void> demonstrateFlatMap() async {
  print('--- FlatMap / MergeMap ---');
  
  Stream<int> slowStream(int n) async* {
    await Future.delayed(Duration(milliseconds: 50 * n));
    yield n * 10;
    yield n * 20;
  }
  
  // Without concurrency limit - all streams run parallel
  final results = <int>[];
  final streams = [1, 2, 3].map((n) => slowStream(n)).toList();
  
  await Future.wait(streams.map((s) async {
    await for (final v in s) results.add(v);
  }));
  
  print('Parallel merge results: $results');
  print('');
}

/// Buffer: collect events and emit as batch
Stream<List<T>> buffer<T>(Stream<T> source, Duration window) async* {
  var batch = <T>[];
  Timer? timer;
  final controller = StreamController<List<T>>();
  
  source.listen(
    (event) {
      batch.add(event);
      timer ??= Timer(window, () {
        if (batch.isNotEmpty) {
          controller.add(List.from(batch));
          batch.clear();
        }
        timer = null;
      });
    },
    onDone: () {
      timer?.cancel();
      if (batch.isNotEmpty) controller.add(batch);
      controller.close();
    },
  );
  
  yield* controller.stream;
}

Future<void> demonstrateBufferAndWindow() async {
  print('--- Buffer (time window) ---');
  
  Stream<int> events() async* {
    for (var i = 1; i <= 10; i++) {
      yield i;
      await Future.delayed(const Duration(milliseconds: 40));
    }
  }
  
  // Buffer every 150ms
  await for (final batch in buffer(events(), const Duration(milliseconds: 150))) {
    print('Buffer: $batch');
  }
  
  print('');
}

/// Share stream (multicast)
class SharedStream<T> {
  final Stream<T> _source;
  final List<StreamController<T>> _controllers = [];
  StreamSubscription<T>? _subscription;
  
  SharedStream(this._source);
  
  Stream<T> share() {
    final controller = StreamController<T>(
      onCancel: () {
        _controllers.remove(_controllers.firstWhere(
          (c) => !c.hasListener,
        ));
        if (_controllers.isEmpty) {
          _subscription?.cancel();
          _subscription = null;
        }
      },
    );
    
    _controllers.add(controller);
    
    _subscription ??= _source.listen(
      (event) {
        for (final c in _controllers) {
          c.add(event);
        }
      },
      onError: (error) {
        for (final c in _controllers) {
          c.addError(error);
        }
      },
      onDone: () {
        for (final c in _controllers) {
          c.close();
        }
      },
    );
    
    return controller.stream;
  }
}

Future<void> demonstrateShareStream() async {
  print('--- Share (Multicast) Stream ---');
  
  var emitCount = 0;
  
  // Expensive source - should only execute once even with multiple subscribers
  Stream<int> expensiveSource() async* {
    for (var i = 1; i <= 5; i++) {
      emitCount++;
      yield i;
      await Future.delayed(const Duration(milliseconds: 20));
    }
  }
  
  final shared = SharedStream(expensiveSource());
  
  final results1 = <int>[];
  final results2 = <int>[];
  
  // Subscribe simultaneously
  final sub1 = shared.share().listen(results1.add);
  final sub2 = shared.share().listen(results2.add);
  
  await Future.delayed(const Duration(milliseconds: 200));
  
  await sub1.cancel();
  await sub2.cancel();
  
  print('Results 1: $results1');
  print('Results 2: $results2');
  print('Source emitted $emitCount times (should be 5 for shared)');
  
  print('');
}

/// Real-world example: autocomplete search
Future<void> demonstrateRealWorldExample() async {
  print('--- Real-World: Autocomplete Search ---');
  
  // Simulate user typing
  final searchInput = StreamController<String>();
  
  // Process: debounce + switchMap to API call
  final results = _autocompleteStream(searchInput.stream);
  
  final subscription = results.listen(
    (result) => print('Search result: $result'),
    onError: (e) => print('Error: $e'),
  );
  
  // Simulate typing
  searchInput.add('f');
  await Future.delayed(const Duration(milliseconds: 50));
  searchInput.add('fl');
  await Future.delayed(const Duration(milliseconds: 50));
  searchInput.add('flu');
  await Future.delayed(const Duration(milliseconds: 50));
  searchInput.add('flut');
  await Future.delayed(const Duration(milliseconds: 300)); // Wait for debounce
  searchInput.add('flutt');
  await Future.delayed(const Duration(milliseconds: 300)); // Wait for debounce
  
  await Future.delayed(const Duration(milliseconds: 200));
  subscription.cancel();
  searchInput.close();
  
  print('');
}

Stream<String> _autocompleteStream(Stream<String> input) async* {
  final controller = StreamController<String>();
  StreamSubscription<String>? currentSearch;
  Timer? debounceTimer;
  
  input.listen(
    (query) {
      // Debounce
      debounceTimer?.cancel();
      debounceTimer = Timer(const Duration(milliseconds: 200), () async {
        if (query.length < 2) return;
        
        // Cancel previous search (switchMap behavior)
        currentSearch?.cancel();
        
        currentSearch = _searchAPI(query).listen(
          controller.add,
          onError: controller.addError,
        );
      });
    },
    onDone: () {
      debounceTimer?.cancel();
      currentSearch?.cancel();
      controller.close();
    },
  );
  
  yield* controller.stream;
}

Stream<String> _searchAPI(String query) async* {
  // Simulate network delay
  await Future.delayed(const Duration(milliseconds: 150));
  
  // Simulate results
  final results = ['Flutter', 'Dart', 'Firebase', 'FlutterBloc', 'FlutterRx']
      .where((r) => r.toLowerCase().startsWith(query.toLowerCase()))
      .toList();
  
  yield '[${results.join(', ')}]';
}
```

---

## ขั้นตอนที่ 3287: Practical RxDart in Flutter

```dart
// lib/main.dart - Flutter app using reactive patterns
import 'dart:async';
import 'package:flutter/material.dart';
// In real app: import 'package:rxdart/rxdart.dart';

void main() {
  runApp(const ReactiveApp());
}

class ReactiveApp extends StatelessWidget {
  const ReactiveApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Reactive Patterns',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const ReactiveCounterScreen(),
    );
  }
}

/// Simple state store using streams
class CounterStore {
  final _countSubject = _BehaviorSubjectLite(0);
  final _loadingSubject = _BehaviorSubjectLite(false);
  final _historySubject = _BehaviorSubjectLite(<int>[]);

  Stream<int> get count => _countSubject.stream;
  Stream<bool> get isLoading => _loadingSubject.stream;
  Stream<List<int>> get history => _historySubject.stream;

  int get currentCount => _countSubject.value;

  Stream<String> get countFormatted =>
      _countSubject.stream.map((c) => 'Count: $c');

  Stream<bool> get canDecrement =>
      _countSubject.stream.map((c) => c > 0);

  void increment() {
    final newCount = _countSubject.value + 1;
    _countSubject.add(newCount);
    _historySubject.add([..._historySubject.value, newCount]);
  }

  void decrement() {
    if (_countSubject.value > 0) {
      final newCount = _countSubject.value - 1;
      _countSubject.add(newCount);
      _historySubject.add([..._historySubject.value, newCount]);
    }
  }

  Future<void> fetchFromServer() async {
    _loadingSubject.add(true);
    await Future.delayed(const Duration(seconds: 2));
    _countSubject.add(42);
    _loadingSubject.add(false);
  }

  void reset() {
    _countSubject.add(0);
    _historySubject.add([]);
  }

  void dispose() {
    _countSubject.close();
    _loadingSubject.close();
    _historySubject.close();
  }
}

/// Simple BehaviorSubject-like implementation
class _BehaviorSubjectLite<T> {
  T _value;
  final _controller = StreamController<T>.broadcast();

  _BehaviorSubjectLite(this._value);

  T get value => _value;

  Stream<T> get stream async* {
    yield _value;
    yield* _controller.stream;
  }

  void add(T value) {
    _value = value;
    _controller.add(value);
  }

  void close() => _controller.close();
}

class ReactiveCounterScreen extends StatefulWidget {
  const ReactiveCounterScreen({super.key});

  @override
  State<ReactiveCounterScreen> createState() => _ReactiveCounterScreenState();
}

class _ReactiveCounterScreenState extends State<ReactiveCounterScreen> {
  final _store = CounterStore();

  @override
  void dispose() {
    _store.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Reactive Counter'),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: _store.reset,
            tooltip: 'Reset',
          ),
        ],
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Count display
            StreamBuilder<String>(
              stream: _store.countFormatted,
              builder: (context, snapshot) {
                return Text(
                  snapshot.data ?? 'Count: 0',
                  style: Theme.of(context).textTheme.displayMedium,
                );
              },
            ),

            const SizedBox(height: 32),

            // Loading state
            StreamBuilder<bool>(
              stream: _store.isLoading,
              builder: (context, snapshot) {
                final isLoading = snapshot.data ?? false;
                return Column(
                  children: [
                    if (isLoading)
                      const CircularProgressIndicator()
                    else
                      ElevatedButton.icon(
                        onPressed: _store.fetchFromServer,
                        icon: const Icon(Icons.cloud_download),
                        label: const Text('Fetch from Server'),
                      ),
                  ],
                );
              },
            ),

            const SizedBox(height: 32),

            // Increment/Decrement buttons
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                StreamBuilder<bool>(
                  stream: _store.canDecrement,
                  builder: (context, snapshot) {
                    return IconButton(
                      icon: const Icon(Icons.remove_circle_outline),
                      iconSize: 48,
                      onPressed: (snapshot.data ?? false) ? _store.decrement : null,
                    );
                  },
                ),
                const SizedBox(width: 24),
                IconButton(
                  icon: const Icon(Icons.add_circle_outline),
                  iconSize: 48,
                  onPressed: _store.increment,
                ),
              ],
            ),

            const SizedBox(height: 32),

            // History
            StreamBuilder<List<int>>(
              stream: _store.history,
              builder: (context, snapshot) {
                final history = snapshot.data ?? [];
                if (history.isEmpty) return const SizedBox.shrink();

                return Column(
                  children: [
                    const Text(
                      'History',
                      style: TextStyle(fontWeight: FontWeight.bold),
                    ),
                    const SizedBox(height: 8),
                    SizedBox(
                      height: 40,
                      child: ListView.builder(
                        scrollDirection: Axis.horizontal,
                        itemCount: history.length,
                        reverse: true,
                        itemBuilder: (context, index) {
                          final actualIndex = history.length - 1 - index;
                          return Container(
                            margin: const EdgeInsets.symmetric(horizontal: 4),
                            padding: const EdgeInsets.symmetric(
                              horizontal: 12,
                              vertical: 4,
                            ),
                            decoration: BoxDecoration(
                              color: Theme.of(context)
                                  .colorScheme
                                  .primaryContainer,
                              borderRadius: BorderRadius.circular(16),
                            ),
                            child: Text('${history[actualIndex]}'),
                          );
                        },
                      ),
                    ),
                  ],
                );
              },
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3288: Testing Async Code

```dart
// test/async_test.dart
import 'dart:async';
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('Stream Tests', () {
    test('countUp stream emits correct values', () async {
      // Test async generator
      Stream<int> countUp(int from, int to) async* {
        for (var i = from; i <= to; i++) {
          yield i;
        }
      }

      final values = await countUp(1, 5).toList();
      expect(values, [1, 2, 3, 4, 5]);
    });

    test('stream transformer works correctly', () async {
      final source = Stream.fromIterable([1, 2, 3, 4, 5, 6]);
      final result = await source
          .map((x) => x * 2)
          .where((x) => x > 4)
          .toList();

      expect(result, [6, 8, 10, 12]);
    });

    test('debounce only emits after silence period', () async {
      final controller = StreamController<int>();
      final results = <int>[];

      // Simple debounce
      Timer? debounce;
      controller.stream.listen((event) {
        debounce?.cancel();
        debounce = Timer(const Duration(milliseconds: 100), () {
          results.add(event);
        });
      });

      controller.add(1);
      await Future.delayed(const Duration(milliseconds: 30));
      controller.add(2);
      await Future.delayed(const Duration(milliseconds: 30));
      controller.add(3); // Only this should be debounced through
      await Future.delayed(const Duration(milliseconds: 200));

      expect(results, [3]);
      controller.close();
    });

    test('zone catches async errors', () async {
      final errors = <String>[];

      await runZoned(
        () async {
          scheduleMicrotask(() => throw Exception('Test error'));
          await Future.delayed(const Duration(milliseconds: 50));
        },
        onError: (error, _) => errors.add(error.toString()),
      );

      expect(errors.length, 1);
      expect(errors.first, contains('Test error'));
    });

    test('BehaviorSubject replays latest value', () async {
      final controller = StreamController<int>.broadcast();
      int? latestValue;

      // Simulate BehaviorSubject by keeping track of latest
      int? savedLatest;

      controller.stream.listen((v) => savedLatest = v);
      controller.add(1);
      controller.add(2);
      controller.add(3);

      await Future.delayed(Duration.zero);

      // Late subscriber would receive 3
      expect(savedLatest, 3);
      controller.close();
    });

    test('isolate computes result correctly', () async {
      final result = await Isolate.run(() {
        var sum = 0;
        for (var i = 1; i <= 100; i++) sum += i;
        return sum;
      });

      expect(result, 5050); // Sum of 1 to 100
    });

    test('async generator handles errors', () async {
      Stream<int> errorStream() async* {
        yield 1;
        throw Exception('Stream error');
        yield 2; // Never reached
      }

      int? received;
      Object? caughtError;

      await for (final value in errorStream().handleError(
        (error) => caughtError = error,
      )) {
        received = value;
      }

      expect(received, 1);
      expect(caughtError, isNotNull);
    });

    test('zip streams combines values correctly', () async {
      final s1 = Stream.fromIterable([1, 2, 3]);
      final s2 = Stream.fromIterable(['a', 'b', 'c']);

      final combined = <List<dynamic>>[];

      final iter1 = StreamIterator(s1);
      final iter2 = StreamIterator(s2);

      while (await iter1.moveNext() && await iter2.moveNext()) {
        combined.add([iter1.current, iter2.current]);
      }

      expect(combined, [
        [1, 'a'],
        [2, 'b'],
        [3, 'c'],
      ]);
    });
  });

  group('Concurrency Tests', () {
    test('futures run concurrently with Future.wait', () async {
      final stopwatch = Stopwatch()..start();

      // These should run concurrently
      await Future.wait([
        Future.delayed(const Duration(milliseconds: 100)),
        Future.delayed(const Duration(milliseconds: 100)),
        Future.delayed(const Duration(milliseconds: 100)),
      ]);

      stopwatch.stop();

      // Should take ~100ms, not 300ms
      expect(stopwatch.elapsedMilliseconds, lessThan(250));
    });

    test('microtasks run before futures', () async {
      final order = <int>[];

      // Schedule in reverse priority order
      Future(() => order.add(3));  // Lowest priority
      scheduleMicrotask(() => order.add(2));  // Higher priority
      order.add(1);  // Synchronous - runs first

      await Future.delayed(Duration.zero);

      expect(order, [1, 2, 3]);
    });
  });
}
```

---

## ขั้นตอนที่ 3289: Isolate Pool Pattern

```dart
// lib/concurrency/isolate_pool.dart
import 'dart:async';
import 'dart:isolate';
import 'dart:collection';

typedef IsolateTask<T> = Future<T> Function();

/// A pool of isolates for parallel processing
class IsolatePool {
  final int size;
  final List<_IsolateWorker> _workers = [];
  final Queue<_PendingTask> _taskQueue = Queue();

  IsolatePool({this.size = 4});

  Future<void> initialize() async {
    for (var i = 0; i < size; i++) {
      final worker = _IsolateWorker(id: i);
      await worker.initialize();
      _workers.add(worker);
    }
  }

  Future<T> submit<T>(Object Function(Object?) task, Object? argument) async {
    // Find a free worker
    final freeWorker = _workers.where((w) => !w.isBusy).firstOrNull;

    if (freeWorker != null) {
      return freeWorker.execute<T>(task, argument);
    }

    // Queue the task if all workers are busy
    final completer = Completer<T>();
    _taskQueue.add(_PendingTask(
      task: task,
      argument: argument,
      completer: completer as Completer<Object?>,
    ));
    return completer.future;
  }

  Future<List<T>> submitBatch<T>(
    Object Function(Object?) task,
    List<Object?> arguments,
  ) async {
    return Future.wait(
      arguments.map((arg) => submit<T>(task, arg)),
    );
  }

  Future<void> dispose() async {
    for (final worker in _workers) {
      worker.dispose();
    }
    _workers.clear();
  }
}

class _PendingTask {
  final Object Function(Object?) task;
  final Object? argument;
  final Completer<Object?> completer;

  _PendingTask({
    required this.task,
    required this.argument,
    required this.completer,
  });
}

class _IsolateWorker {
  final int id;
  Isolate? _isolate;
  late SendPort _sendPort;
  bool _isBusy = false;

  _IsolateWorker({required this.id});

  bool get isBusy => _isBusy;

  Future<void> initialize() async {
    final receivePort = ReceivePort();
    _isolate = await Isolate.spawn(_workerEntry, receivePort.sendPort);
    _sendPort = await receivePort.first as SendPort;
    receivePort.close();
  }

  Future<T> execute<T>(Object Function(Object?) task, Object? argument) async {
    _isBusy = true;
    final completer = Completer<T>();

    final responsePort = ReceivePort();
    _sendPort.send([responsePort.sendPort, task, argument]);

    final result = await responsePort.first;
    responsePort.close();

    if (result is _ErrorResult) {
      completer.completeError(result.error, result.stackTrace);
    } else {
      completer.complete(result as T);
    }

    _isBusy = false;
    return completer.future;
  }

  void dispose() => _isolate?.kill();
}

class _ErrorResult {
  final Object error;
  final StackTrace stackTrace;

  _ErrorResult(this.error, this.stackTrace);
}

void _workerEntry(SendPort mainSendPort) async {
  final receivePort = ReceivePort();
  mainSendPort.send(receivePort.sendPort);

  await for (final message in receivePort) {
    if (message is List) {
      final responseSendPort = message[0] as SendPort;
      final task = message[1] as Object Function(Object?);
      final argument = message[2];

      try {
        final result = task(argument);
        responseSendPort.send(result);
      } catch (e, st) {
        responseSendPort.send(_ErrorResult(e, st));
      }
    }
  }
}

// Usage example
void isolatePoolExample() async {
  final pool = IsolatePool(size: 4);
  await pool.initialize();

  // Process many items in parallel
  final numbers = List.generate(20, (i) => i + 1);

  final results = await pool.submitBatch<int>(
    (n) => (n as int) * (n as int), // Square each number
    numbers.cast<Object?>(),
  );

  print('Squared: $results');

  await pool.dispose();
}
```

---

**← [Part 84](part-84-advanced-camera.md)**
**ต่อไป: [Part 86 →](part-86-flutter-plugins.md)**
