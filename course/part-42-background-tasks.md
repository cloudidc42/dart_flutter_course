# Part 42: Background Tasks & Workmanager
## ขั้นตอนที่ 1561-1600

---

## 🎯 เป้าหมายของ Part นี้

- Workmanager (background jobs)
- Isolates
- Background fetch
- App lifecycle
- Periodic tasks

---

## ขั้นตอนที่ 1561: Workmanager Setup

```yaml
# pubspec.yaml
dependencies:
  workmanager: ^0.5.2
  flutter_background_service: ^5.0.5
  flutter_isolate: ^2.0.4
```

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
<uses-permission android:name="android.permission.WAKE_LOCK"/>

<application>
  <!-- Workmanager -->
  <service
    android:name="be.tramckrijte.workmanager.BackgroundWorker"
    android:permission="android.permission.BIND_JOB_SERVICE"
    android:exported="true"/>
</application>
```

---

## ขั้นตอนที่ 1562: Workmanager

```dart
import 'package:flutter/material.dart';
import 'package:workmanager/workmanager.dart';

// ─── Task names ───
class BackgroundTasks {
  static const String syncData = 'sync_data';
  static const String fetchNews = 'fetch_news';
  static const String cleanCache = 'clean_cache';
  static const String sendPendingRequests = 'send_pending_requests';
}

// ─── Background task dispatcher (top-level function) ───
@pragma('vm:entry-point')
void callbackDispatcher() {
  Workmanager().executeTask((taskName, inputData) async {
    print('Running task: $taskName, data: $inputData');

    try {
      switch (taskName) {
        case BackgroundTasks.syncData:
          await _syncData(inputData);
          break;
        case BackgroundTasks.fetchNews:
          await _fetchNews();
          break;
        case BackgroundTasks.cleanCache:
          await _cleanCache();
          break;
        case BackgroundTasks.sendPendingRequests:
          await _sendPendingRequests();
          break;
        default:
          print('Unknown task: $taskName');
      }
      return true; // success
    } catch (e) {
      print('Task failed: $e');
      return false; // retry
    }
  });
}

// Task implementations
Future<void> _syncData(Map<String, dynamic>? data) async {
  print('Syncing data...');
  await Future.delayed(const Duration(seconds: 2));
  print('Data synced!');
}

Future<void> _fetchNews() async {
  print('Fetching news...');
  await Future.delayed(const Duration(seconds: 3));
  print('News fetched!');
}

Future<void> _cleanCache() async {
  print('Cleaning cache...');
  await Future.delayed(const Duration(seconds: 1));
  print('Cache cleaned!');
}

Future<void> _sendPendingRequests() async {
  print('Sending pending requests...');
  await Future.delayed(const Duration(seconds: 2));
  print('Requests sent!');
}

// ─── Workmanager Service ───
class WorkmanagerService {
  static final WorkmanagerService _instance = WorkmanagerService._();
  factory WorkmanagerService() => _instance;
  WorkmanagerService._();

  Future<void> initialize() async {
    await Workmanager().initialize(
      callbackDispatcher,
      isInDebugMode: false,
    );
  }

  // One-time task
  Future<void> scheduleOneTimeTask({
    required String taskName,
    Map<String, dynamic>? inputData,
    Duration initialDelay = Duration.zero,
    BackoffPolicy backoffPolicy = BackoffPolicy.exponential,
  }) async {
    await Workmanager().registerOneOffTask(
      taskName,
      taskName,
      inputData: inputData,
      initialDelay: initialDelay,
      backoffPolicy: backoffPolicy,
      constraints: Constraints(
        networkType: NetworkType.connected,
        requiresBatteryNotLow: false,
        requiresCharging: false,
        requiresDeviceIdle: false,
      ),
    );
  }

  // Periodic task (minimum 15 minutes on Android)
  Future<void> schedulePeriodicTask({
    required String taskName,
    Duration frequency = const Duration(hours: 1),
    Map<String, dynamic>? inputData,
    bool requiresNetwork = false,
  }) async {
    await Workmanager().registerPeriodicTask(
      taskName,
      taskName,
      frequency: frequency,
      inputData: inputData,
      constraints: Constraints(
        networkType: requiresNetwork ? NetworkType.connected : NetworkType.not_required,
        requiresBatteryNotLow: false,
      ),
    );
  }

  Future<void> cancelTask(String taskName) async {
    await Workmanager().cancelByUniqueName(taskName);
  }

  Future<void> cancelAllTasks() async {
    await Workmanager().cancelAll();
  }
}

// ─── Usage ───
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  WorkmanagerService service = WorkmanagerService();
  await service.initialize();

  // Schedule background sync every hour
  await service.schedulePeriodicTask(
    taskName: BackgroundTasks.syncData,
    frequency: const Duration(hours: 1),
    requiresNetwork: true,
  );

  // Schedule cache cleanup daily
  await service.schedulePeriodicTask(
    taskName: BackgroundTasks.cleanCache,
    frequency: const Duration(hours: 24),
  );

  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(home: BackgroundTasksDemo());
  }
}

class BackgroundTasksDemo extends StatelessWidget {
  const BackgroundTasksDemo({super.key});

  @override
  Widget build(BuildContext context) {
    WorkmanagerService service = WorkmanagerService();
    return Scaffold(
      appBar: AppBar(title: const Text('Background Tasks')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          ElevatedButton(
            onPressed: () => service.scheduleOneTimeTask(
              taskName: BackgroundTasks.fetchNews,
              initialDelay: const Duration(seconds: 5),
            ),
            child: const Text('Schedule News Fetch (5s delay)'),
          ),
          const SizedBox(height: 8),
          ElevatedButton(
            onPressed: () => service.scheduleOneTimeTask(
              taskName: BackgroundTasks.sendPendingRequests,
            ),
            child: const Text('Send Pending Requests Now'),
          ),
          const SizedBox(height: 8),
          ElevatedButton(
            onPressed: service.cancelAllTasks,
            style: ElevatedButton.styleFrom(backgroundColor: Colors.red),
            child: const Text('Cancel All Tasks'),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1563: Isolates สำหรับ Heavy Computation

```dart
import 'dart:async';
import 'dart:isolate';
import 'package:flutter/foundation.dart';

// ─── Simple compute() ───
Future<int> computeSum(int n) async {
  return compute(_sumWorker, n);
}

int _sumWorker(int n) {
  int sum = 0;
  for (int i = 1; i <= n; i++) {
    sum += i;
  }
  return sum;
}

// ─── Manual Isolate ───
class IsolateWorker {
  Isolate? _isolate;
  ReceivePort? _receivePort;
  SendPort? _sendPort;

  Future<void> start() async {
    _receivePort = ReceivePort();

    _isolate = await Isolate.spawn(
      _workerMain,
      _receivePort!.sendPort,
    );

    // รอรับ SendPort จาก worker
    _sendPort = await _receivePort!.first;
  }

  Future<dynamic> sendTask(dynamic task) async {
    if (_sendPort == null) throw StateError('Worker not started');

    ReceivePort replyPort = ReceivePort();
    _sendPort!.send([replyPort.sendPort, task]);
    return replyPort.first;
  }

  void dispose() {
    _isolate?.kill(priority: Isolate.immediate);
    _receivePort?.close();
  }

  static void _workerMain(SendPort mainSendPort) {
    ReceivePort workerReceivePort = ReceivePort();
    mainSendPort.send(workerReceivePort.sendPort);

    workerReceivePort.listen((message) {
      SendPort replyTo = message[0];
      dynamic task = message[1];

      // Process task
      dynamic result = _processTask(task);
      replyTo.send(result);
    });
  }

  static dynamic _processTask(dynamic task) {
    if (task is Map) {
      String type = task['type'] ?? '';
      switch (type) {
        case 'sum':
          int n = task['n'] ?? 0;
          int sum = 0;
          for (int i = 1; i <= n; i++) sum += i;
          return sum;
        case 'fibonacci':
          int n = task['n'] ?? 0;
          return _fib(n);
        case 'sort':
          List<int> list = List<int>.from(task['list'] ?? []);
          list.sort();
          return list;
      }
    }
    return null;
  }

  static int _fib(int n) {
    if (n <= 1) return n;
    return _fib(n - 1) + _fib(n - 2);
  }
}

// ─── Isolate Pool ───
class IsolatePool {
  final int size;
  final List<IsolateWorker> _workers = [];
  int _nextWorker = 0;

  IsolatePool({this.size = 4});

  Future<void> start() async {
    for (int i = 0; i < size; i++) {
      IsolateWorker worker = IsolateWorker();
      await worker.start();
      _workers.add(worker);
    }
  }

  Future<dynamic> execute(dynamic task) {
    // Round-robin task distribution
    IsolateWorker worker = _workers[_nextWorker % _workers.length];
    _nextWorker++;
    return worker.sendTask(task);
  }

  void dispose() {
    for (IsolateWorker worker in _workers) {
      worker.dispose();
    }
  }
}

// ─── Usage in Flutter ───
class HeavyComputationWidget extends StatefulWidget {
  const HeavyComputationWidget({super.key});

  @override
  State<HeavyComputationWidget> createState() => _HeavyComputationWidgetState();
}

class _HeavyComputationWidgetState extends State<HeavyComputationWidget> {
  final IsolatePool _pool = IsolatePool(size: 2);
  bool _poolReady = false;
  String _result = '';
  bool _computing = false;

  @override
  void initState() {
    super.initState();
    _pool.start().then((_) => setState(() => _poolReady = true));
  }

  Future<void> _computeFibonacci() async {
    setState(() => _computing = true);

    DateTime start = DateTime.now();
    dynamic result = await _pool.execute({'type': 'fibonacci', 'n': 40});
    Duration elapsed = DateTime.now().difference(start);

    setState(() {
      _result = 'fib(40) = $result (${elapsed.inMilliseconds}ms)';
      _computing = false;
    });
  }

  Future<void> _sortLargeList() async {
    setState(() => _computing = true);

    // Generate large random list
    List<int> list = List.generate(100000, (i) => 100000 - i);

    DateTime start = DateTime.now();
    dynamic sorted = await _pool.execute({'type': 'sort', 'list': list});
    Duration elapsed = DateTime.now().difference(start);

    setState(() {
      _result = 'Sorted ${(sorted as List).length} items in ${elapsed.inMilliseconds}ms';
      _computing = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Isolates Demo')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            if (!_poolReady)
              const CircularProgressIndicator()
            else ...[
              ElevatedButton(
                onPressed: _computing ? null : _computeFibonacci,
                child: const Text('Compute fib(40)'),
              ),
              const SizedBox(height: 8),
              ElevatedButton(
                onPressed: _computing ? null : _sortLargeList,
                child: const Text('Sort 100,000 items'),
              ),
            ],
            const SizedBox(height: 16),
            if (_computing)
              const Row(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  SizedBox(width: 20, height: 20, child: CircularProgressIndicator(strokeWidth: 2)),
                  SizedBox(width: 8),
                  Text('Computing in background...'),
                ],
              )
            else if (_result.isNotEmpty)
              Text(_result, style: const TextStyle(fontSize: 16)),
          ],
        ),
      ),
    );
  }

  @override
  void dispose() {
    _pool.dispose();
    super.dispose();
  }
}
```

---

## ขั้นตอนที่ 1564: App Lifecycle Manager

```dart
import 'package:flutter/material.dart';

class AppLifecycleManager extends StatefulWidget {
  final Widget child;
  const AppLifecycleManager({super.key, required this.child});

  @override
  State<AppLifecycleManager> createState() => _AppLifecycleManagerState();
}

class _AppLifecycleManagerState extends State<AppLifecycleManager>
    with WidgetsBindingObserver {
  AppLifecycleState _lastState = AppLifecycleState.resumed;

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    print('App lifecycle: $_lastState -> $state');

    switch (state) {
      case AppLifecycleState.resumed:
        _onResumed();
        break;
      case AppLifecycleState.inactive:
        _onInactive();
        break;
      case AppLifecycleState.paused:
        _onPaused();
        break;
      case AppLifecycleState.detached:
        _onDetached();
        break;
      case AppLifecycleState.hidden:
        _onHidden();
        break;
    }

    _lastState = state;
  }

  void _onResumed() {
    print('App resumed - sync data, refresh UI');
    // SyncService.sync();
    // refreshAccessToken();
  }

  void _onInactive() {
    print('App inactive (e.g., phone call)');
  }

  void _onPaused() {
    print('App backgrounded - save state');
    // saveAppState();
    // pauseAnimations();
  }

  void _onDetached() {
    print('App about to be killed');
    // final cleanup
  }

  void _onHidden() {
    print('App hidden');
  }

  @override
  Widget build(BuildContext context) => widget.child;

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }
}
```

---

**← [Part 41 - In-App Purchases](part-41-in-app-purchases.md)**

**ต่อไป: [Part 43 - State Management Advanced →](part-43-state-management-advanced.md)**
