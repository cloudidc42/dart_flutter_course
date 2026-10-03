# Part 69: Dart Isolates Advanced
## ขั้นตอนที่ 2641-2680

## 🎯 เป้าหมายของ Part นี้
- เรียนรู้ Isolate Groups และการทำงาน
- ใช้ SendPort และ ReceivePort patterns อย่างมีประสิทธิภาพ
- แชร์ memory ด้วย TransferableTypedData
- สร้าง parallel data processing pipelines
- จัดการ Isolate errors และ restart

---

## ขั้นตอนที่ 2641: Isolate พื้นฐาน

```dart
// lib/isolates/isolate_basics.dart
import 'dart:isolate';
import 'dart:async';

/// ตัวอย่าง Isolate พื้นฐาน
Future<void> basicIsolateExample() async {
  print('Main isolate: ${Isolate.current.debugName}');

  final receivePort = ReceivePort();

  // Spawn isolate
  final isolate = await Isolate.spawn(
    _isolateEntry,
    receivePort.sendPort,
    debugName: 'worker-1',
  );

  // รอรับข้อมูลจาก isolate
  final result = await receivePort.first;
  print('Received from isolate: $result');

  receivePort.close();
  isolate.kill();
}

/// Entry function สำหรับ isolate
void _isolateEntry(SendPort sendPort) {
  print('Worker isolate: ${Isolate.current.debugName}');

  // ทำงานหนัก
  final result = _heavyComputation(1000000);

  // ส่งผลลัพธ์กลับ
  sendPort.send(result);
}

int _heavyComputation(int n) {
  var sum = 0;
  for (var i = 0; i < n; i++) {
    sum += i;
  }
  return sum;
}

/// Isolate แบบ two-way communication
class TwoWayIsolate {
  late final Isolate _isolate;
  late final SendPort _sendPort;
  final _receivePort = ReceivePort();
  late final Stream _responses;

  Future<void> initialize() async {
    _responses = _receivePort.asBroadcastStream();

    // ส่ง ReceivePort ไปให้ isolate
    _isolate = await Isolate.spawn(
      _workerMain,
      _receivePort.sendPort,
      debugName: 'two-way-worker',
    );

    // รอรับ SendPort จาก isolate
    _sendPort = await _responses.first as SendPort;
    print('Two-way isolate initialized');
  }

  /// ส่งคำสั่งและรอรับผล
  Future<dynamic> send(Map<String, dynamic> message) async {
    final responsePort = ReceivePort();
    _sendPort.send({
      ...message,
      'responsePort': responsePort.sendPort,
    });
    final result = await responsePort.first;
    responsePort.close();
    return result;
  }

  void dispose() {
    _isolate.kill(priority: Isolate.immediate);
    _receivePort.close();
  }
}

void _workerMain(SendPort mainSendPort) {
  final workerReceivePort = ReceivePort();

  // ส่ง SendPort กลับไปยัง main
  mainSendPort.send(workerReceivePort.sendPort);

  // ฟัง messages
  workerReceivePort.listen((message) {
    if (message is Map<String, dynamic>) {
      final responsePort = message['responsePort'] as SendPort?;
      final action = message['action'] as String?;
      final data = message['data'];

      dynamic result;
      switch (action) {
        case 'compute':
          result = _processData(data);
          break;
        case 'transform':
          result = _transformData(data as List);
          break;
        default:
          result = {'error': 'Unknown action: $action'};
      }

      responsePort?.send(result);
    }
  });
}

dynamic _processData(dynamic data) {
  if (data is int) return data * data;
  if (data is String) return data.toUpperCase();
  return data;
}

List _transformData(List data) {
  return data.map((item) {
    if (item is int) return item * 2;
    if (item is String) return item.toLowerCase();
    return item;
  }).toList();
}
```

---

## ขั้นตอนที่ 2642: Isolate Groups

```dart
// lib/isolates/isolate_groups.dart
import 'dart:isolate';
import 'dart:async';

/// Isolate Group ช่วยให้ isolates แชร์ heap memory ได้
/// (รองรับใน Dart 2.15+)
class IsolateGroupManager {
  final int groupSize;
  final List<_WorkerInfo> _workers = [];
  int _nextWorker = 0;

  IsolateGroupManager({this.groupSize = 4});

  Future<void> initialize() async {
    for (var i = 0; i < groupSize; i++) {
      final worker = await _createWorker(i);
      _workers.add(worker);
    }
    print('Isolate group initialized with $groupSize workers');
  }

  Future<_WorkerInfo> _createWorker(int id) async {
    final receivePort = ReceivePort();
    final responses = receivePort.asBroadcastStream();

    final isolate = await Isolate.spawn(
      _groupWorkerMain,
      _WorkerConfig(
        id: id,
        sendPort: receivePort.sendPort,
      ),
      debugName: 'group-worker-$id',
    );

    final sendPort = await responses.first as SendPort;

    return _WorkerInfo(
      id: id,
      isolate: isolate,
      sendPort: sendPort,
      receivePort: receivePort,
      responses: responses,
    );
  }

  /// ส่งงานไปยัง worker โดยใช้ round-robin
  Future<dynamic> dispatch(String action, dynamic data) async {
    if (_workers.isEmpty) throw StateError('No workers available');

    final worker = _workers[_nextWorker % _workers.length];
    _nextWorker++;

    return worker.call(action, data);
  }

  /// ส่งงานไปยัง worker ทั้งหมดพร้อมกัน (broadcast)
  Future<List<dynamic>> broadcast(String action, dynamic data) async {
    return Future.wait(
      _workers.map((w) => w.call(action, data)),
    );
  }

  void dispose() {
    for (final worker in _workers) {
      worker.dispose();
    }
    _workers.clear();
  }
}

class _WorkerConfig {
  final int id;
  final SendPort sendPort;

  _WorkerConfig({required this.id, required this.sendPort});
}

class _WorkerInfo {
  final int id;
  final Isolate isolate;
  final SendPort sendPort;
  final ReceivePort receivePort;
  final Stream responses;

  _WorkerInfo({
    required this.id,
    required this.isolate,
    required this.sendPort,
    required this.receivePort,
    required this.responses,
  });

  Future<dynamic> call(String action, dynamic data) async {
    final responsePort = ReceivePort();
    sendPort.send({
      'action': action,
      'data': data,
      'responsePort': responsePort.sendPort,
    });
    final result = await responsePort.first;
    responsePort.close();
    return result;
  }

  void dispose() {
    isolate.kill();
    receivePort.close();
  }
}

void _groupWorkerMain(_WorkerConfig config) {
  final workerPort = ReceivePort();
  config.sendPort.send(workerPort.sendPort);

  print('Worker ${config.id} started');

  workerPort.listen((message) {
    if (message is! Map<String, dynamic>) return;

    final action = message['action'] as String;
    final data = message['data'];
    final responsePort = message['responsePort'] as SendPort;

    try {
      final result = _handleAction(action, data, config.id);
      responsePort.send(result);
    } catch (e) {
      responsePort.send({'error': e.toString()});
    }
  });
}

dynamic _handleAction(String action, dynamic data, int workerId) {
  switch (action) {
    case 'sort':
      final list = List<int>.from(data as List);
      list.sort();
      return list;
    case 'sum':
      final list = data as List;
      return list.fold<num>(0, (acc, val) => acc + (val as num));
    case 'filter_even':
      final list = data as List;
      return list.where((x) => (x as int).isEven).toList();
    case 'worker_id':
      return workerId;
    default:
      throw ArgumentError('Unknown action: $action');
  }
}
```

---

## ขั้นตอนที่ 2643: TransferableTypedData สำหรับ Shared Memory

```dart
// lib/isolates/transferable_data.dart
import 'dart:isolate';
import 'dart:typed_data';

/// TransferableTypedData ช่วยส่ง typed data โดยไม่ต้อง copy
class SharedMemoryExample {
  /// ตัวอย่างการส่ง large buffer โดยใช้ TransferableTypedData
  static Future<void> demonstrateTransfer() async {
    // สร้าง large buffer
    const bufferSize = 1024 * 1024; // 1MB
    final buffer = Uint8List(bufferSize);
    for (var i = 0; i < bufferSize; i++) {
      buffer[i] = i % 256;
    }

    print('Original buffer size: ${buffer.lengthInBytes} bytes');

    final receivePort = ReceivePort();

    // Wrap ด้วย TransferableTypedData
    final transferable = TransferableTypedData.fromList([buffer]);

    final stopwatch = Stopwatch()..start();

    await Isolate.spawn(
      _receiveBuffer,
      _BufferMessage(
        transferable: transferable,
        sendPort: receivePort.sendPort,
      ),
    );

    final result = await receivePort.first as Map;
    stopwatch.stop();

    print('Transfer completed in ${stopwatch.elapsedMicroseconds}µs');
    print('Checksum from isolate: ${result['checksum']}');

    receivePort.close();
  }

  /// เปรียบเทียบ regular copy vs TransferableTypedData
  static Future<void> compareCopyVsTransfer() async {
    const size = 10 * 1024 * 1024; // 10MB
    final data = Uint8List(size);

    print('\n=== Comparison: Copy vs Transfer ===');

    // Method 1: Regular copy (ช้ากว่า)
    {
      final port = ReceivePort();
      final sw = Stopwatch()..start();

      await Isolate.spawn(
        _processRegularBuffer,
        _RegularMessage(data: data, sendPort: port.sendPort),
      );

      await port.first;
      sw.stop();
      print('Regular copy: ${sw.elapsedMilliseconds}ms');
      port.close();
    }

    // Method 2: TransferableTypedData (เร็วกว่า)
    {
      final port = ReceivePort();
      final transferable = TransferableTypedData.fromList([data]);
      final sw = Stopwatch()..start();

      await Isolate.spawn(
        _processTransferableBuffer,
        _TransferableMessage(
          transferable: transferable,
          sendPort: port.sendPort,
        ),
      );

      await port.first;
      sw.stop();
      print('TransferableTypedData: ${sw.elapsedMilliseconds}ms');
      port.close();
    }
  }
}

class _BufferMessage {
  final TransferableTypedData transferable;
  final SendPort sendPort;

  _BufferMessage({
    required this.transferable,
    required this.sendPort,
  });
}

class _RegularMessage {
  final Uint8List data;
  final SendPort sendPort;

  _RegularMessage({required this.data, required this.sendPort});
}

class _TransferableMessage {
  final TransferableTypedData transferable;
  final SendPort sendPort;

  _TransferableMessage({
    required this.transferable,
    required this.sendPort,
  });
}

void _receiveBuffer(_BufferMessage message) {
  // Materialize จาก TransferableTypedData
  final buffer =
      message.transferable.materialize().asUint8List();

  // คำนวณ checksum
  var checksum = 0;
  for (final byte in buffer) {
    checksum ^= byte;
  }

  message.sendPort.send({'checksum': checksum, 'size': buffer.length});
}

void _processRegularBuffer(_RegularMessage message) {
  // กระทำบาง processing
  var sum = 0;
  for (final byte in message.data) {
    sum += byte;
  }
  message.sendPort.send(sum);
}

void _processTransferableBuffer(_TransferableMessage message) {
  final data = message.transferable.materialize().asUint8List();
  var sum = 0;
  for (final byte in data) {
    sum += byte;
  }
  message.sendPort.send(sum);
}
```

---

## ขั้นตอนที่ 2644: Parallel Data Processing Pipelines

```dart
// lib/isolates/pipeline.dart
import 'dart:isolate';
import 'dart:async';
import 'dart:typed_data';

/// Pipeline stage
typedef PipelineStage<T, R> = Future<R> Function(T input);

/// Parallel pipeline processor
class ParallelPipeline<T, R> {
  final int parallelism;
  final PipelineStage<T, R> processor;
  final _inputQueue = StreamController<_PipelineTask<T, R>>();
  final List<_PipelineWorker<T, R>> _workers = [];
  bool _initialized = false;

  ParallelPipeline({
    required this.processor,
    this.parallelism = 4,
  });

  Future<void> initialize() async {
    for (var i = 0; i < parallelism; i++) {
      final worker = _PipelineWorker<T, R>(processor);
      _workers.add(worker);
    }
    _initialized = true;
  }

  /// Process item และรอผลลัพธ์
  Future<R> process(T input) {
    if (!_initialized) throw StateError('Pipeline not initialized');

    final completer = Completer<R>();
    final task = _PipelineTask<T, R>(input, completer);

    // หา worker ที่ว่าง
    final worker = _getAvailableWorker();
    worker.execute(task);

    return completer.future;
  }

  /// Process ข้อมูลหลายรายการพร้อมกัน
  Future<List<R>> processAll(List<T> inputs) async {
    if (!_initialized) throw StateError('Pipeline not initialized');

    final futures = inputs.map(process).toList();
    return Future.wait(futures);
  }

  /// Stream processing
  Stream<R> processStream(Stream<T> inputStream) async* {
    final buffer = <T>[];
    const batchSize = 10;

    await for (final item in inputStream) {
      buffer.add(item);

      if (buffer.length >= batchSize) {
        final batch = List<T>.from(buffer);
        buffer.clear();
        final results = await processAll(batch);
        for (final result in results) {
          yield result;
        }
      }
    }

    // Process remaining items
    if (buffer.isNotEmpty) {
      final results = await processAll(buffer);
      for (final result in results) {
        yield result;
      }
    }
  }

  _PipelineWorker<T, R> _getAvailableWorker() {
    // Simple round-robin
    return _workers[DateTime.now().millisecond % _workers.length];
  }
}

class _PipelineTask<T, R> {
  final T input;
  final Completer<R> completer;

  _PipelineTask(this.input, this.completer);
}

class _PipelineWorker<T, R> {
  final PipelineStage<T, R> processor;

  _PipelineWorker(this.processor);

  Future<void> execute(_PipelineTask<T, R> task) async {
    try {
      final result = await processor(task.input);
      task.completer.complete(result);
    } catch (e) {
      task.completer.completeError(e);
    }
  }
}

/// ตัวอย่าง: Image Processing Pipeline ด้วย Isolates
class ImageProcessingPipeline {
  static Future<List<ProcessedImage>> processImages(
    List<RawImage> images,
  ) async {
    const parallelism = 4;
    final chunkSize = (images.length / parallelism).ceil();

    // แบ่งงานออกเป็น chunks
    final chunks = <List<RawImage>>[];
    for (var i = 0; i < images.length; i += chunkSize) {
      final end =
          (i + chunkSize < images.length) ? i + chunkSize : images.length;
      chunks.add(images.sublist(i, end));
    }

    // Process แต่ละ chunk ใน isolate แยกกัน
    final futures = chunks.map((chunk) {
      return Isolate.run(
        () => chunk.map(_processImage).toList(),
      );
    }).toList();

    final results = await Future.wait(futures);
    return results.expand((list) => list).toList();
  }

  static ProcessedImage _processImage(RawImage raw) {
    // Simulate image processing
    final processed = Uint8List(raw.data.length);

    // Apply grayscale filter
    for (var i = 0; i < raw.data.length; i += 4) {
      final r = raw.data[i];
      final g = raw.data[i + 1];
      final b = raw.data[i + 2];
      final gray = (0.299 * r + 0.587 * g + 0.114 * b).round();
      processed[i] = gray;
      processed[i + 1] = gray;
      processed[i + 2] = gray;
      processed[i + 3] = raw.data[i + 3];
    }

    return ProcessedImage(
      id: raw.id,
      data: processed,
      width: raw.width,
      height: raw.height,
      processedAt: DateTime.now(),
    );
  }
}

class RawImage {
  final String id;
  final Uint8List data;
  final int width;
  final int height;

  RawImage({
    required this.id,
    required this.data,
    required this.width,
    required this.height,
  });
}

class ProcessedImage {
  final String id;
  final Uint8List data;
  final int width;
  final int height;
  final DateTime processedAt;

  ProcessedImage({
    required this.id,
    required this.data,
    required this.width,
    required this.height,
    required this.processedAt,
  });
}
```

---

## ขั้นตอนที่ 2645: Isolate Error Handling และ Restart

```dart
// lib/isolates/resilient_isolate.dart
import 'dart:isolate';
import 'dart:async';

/// Isolate ที่ restart ตัวเองเมื่อ error
class ResilientIsolate {
  final String name;
  final void Function(SendPort) entryPoint;
  final int maxRestarts;
  final Duration restartDelay;

  Isolate? _isolate;
  SendPort? _workerSendPort;
  ReceivePort? _receivePort;
  int _restartCount = 0;
  bool _shouldRun = true;

  final _onRestart = StreamController<int>.broadcast();
  Stream<int> get onRestart => _onRestart.stream;

  final _onError = StreamController<Object>.broadcast();
  Stream<Object> get onError => _onError.stream;

  ResilientIsolate({
    required this.name,
    required this.entryPoint,
    this.maxRestarts = 3,
    this.restartDelay = const Duration(seconds: 1),
  });

  Future<void> start() async {
    await _spawn();
  }

  Future<void> _spawn() async {
    _receivePort?.close();
    _receivePort = ReceivePort();

    final errorPort = ReceivePort();
    final exitPort = ReceivePort();

    try {
      _isolate = await Isolate.spawn(
        entryPoint,
        _receivePort!.sendPort,
        debugName: name,
        onError: errorPort.sendPort,
        onExit: exitPort.sendPort,
        errorsAreFatal: false,
      );

      // รับ SendPort จาก worker
      final portOrError =
          await Future.any<dynamic>([
        _receivePort!.first,
        errorPort.first.then((err) => throw Exception(err)),
      ]);

      if (portOrError is SendPort) {
        _workerSendPort = portOrError;
      }

      // ฟัง errors และ exits
      errorPort.listen(_handleError);
      exitPort.listen((_) => _handleExit());

      print('$name: Started (restart count: $_restartCount)');
    } catch (e) {
      errorPort.close();
      exitPort.close();
      _handleError(e);
    }
  }

  void _handleError(dynamic error) {
    print('$name: Error - $error');
    _onError.add(error is Object ? error : Exception(error));

    if (_shouldRun && _restartCount < maxRestarts) {
      _scheduleRestart();
    } else if (_restartCount >= maxRestarts) {
      print('$name: Max restarts reached, giving up');
      _shouldRun = false;
    }
  }

  void _handleExit() {
    print('$name: Exited');

    if (_shouldRun && _restartCount < maxRestarts) {
      _scheduleRestart();
    }
  }

  void _scheduleRestart() {
    _restartCount++;
    _onRestart.add(_restartCount);

    Future.delayed(
      restartDelay * _restartCount, // Exponential backoff
      () {
        if (_shouldRun) {
          print('$name: Restarting (attempt $_restartCount)...');
          _spawn();
        }
      },
    );
  }

  Future<dynamic> call(
    String action,
    dynamic data, {
    Duration timeout = const Duration(seconds: 10),
  }) async {
    if (_workerSendPort == null) {
      throw StateError('Worker not ready');
    }

    final responsePort = ReceivePort();
    _workerSendPort!.send({
      'action': action,
      'data': data,
      'responsePort': responsePort.sendPort,
    });

    try {
      return await responsePort.first.timeout(timeout);
    } finally {
      responsePort.close();
    }
  }

  Future<void> stop() async {
    _shouldRun = false;
    _isolate?.kill(priority: Isolate.immediate);
    _receivePort?.close();
    await _onRestart.close();
    await _onError.close();
  }
}

/// Worker ที่มีโอกาส error
void faultyWorker(SendPort sendPort) {
  final receivePort = ReceivePort();
  sendPort.send(receivePort.sendPort);

  var requestCount = 0;

  receivePort.listen((message) {
    if (message is! Map<String, dynamic>) return;

    requestCount++;
    final action = message['action'] as String;
    final data = message['data'];
    final responsePort = message['responsePort'] as SendPort;

    // จำลอง error ทุก 5 requests
    if (requestCount % 5 == 0) {
      throw Exception('Simulated worker crash!');
    }

    switch (action) {
      case 'process':
        responsePort.send({'result': data.toString().toUpperCase()});
        break;
      default:
        responsePort.send({'error': 'Unknown action'});
    }
  });
}

/// ตัวอย่างการใช้งาน
Future<void> demonstrateResilientIsolate() async {
  final worker = ResilientIsolate(
    name: 'fault-tolerant-worker',
    entryPoint: faultyWorker,
    maxRestarts: 3,
    restartDelay: const Duration(milliseconds: 500),
  );

  // ฟัง restart events
  worker.onRestart.listen((count) {
    print('Worker restarted (count: $count)');
  });

  worker.onError.listen((error) {
    print('Worker error: $error');
  });

  await worker.start();

  // ส่งงานหลายๆ ครั้ง
  for (var i = 0; i < 10; i++) {
    try {
      final result = await worker.call('process', 'data-$i');
      print('Result $i: $result');
    } catch (e) {
      print('Request $i failed: $e');
    }
    await Future.delayed(const Duration(milliseconds: 200));
  }

  await worker.stop();
}
```

---

## ขั้นตอนที่ 2646: Worker Pool Pattern

```dart
// lib/isolates/worker_pool.dart
import 'dart:isolate';
import 'dart:async';
import 'dart:collection';

/// Worker Pool สำหรับจัดการ Isolates อย่างมีประสิทธิภาพ
class IsolateWorkerPool {
  final int minWorkers;
  final int maxWorkers;
  final Duration workerTimeout;
  final void Function(SendPort) workerEntryPoint;

  final _availableWorkers = Queue<_Worker>();
  final _busyWorkers = <_Worker>{};
  final _pendingTasks = Queue<_Task>();

  bool _closed = false;
  int _totalWorkers = 0;

  IsolateWorkerPool({
    required this.workerEntryPoint,
    this.minWorkers = 2,
    this.maxWorkers = 8,
    this.workerTimeout = const Duration(minutes: 5),
  });

  Future<void> initialize() async {
    for (var i = 0; i < minWorkers; i++) {
      final worker = await _createWorker();
      _availableWorkers.add(worker);
    }
    print('Worker pool initialized with $minWorkers workers');
  }

  Future<_Worker> _createWorker() async {
    final receivePort = ReceivePort();

    final isolate = await Isolate.spawn(
      workerEntryPoint,
      receivePort.sendPort,
      debugName: 'pool-worker-${_totalWorkers++}',
    );

    final sendPort = await receivePort.first as SendPort;

    return _Worker(
      isolate: isolate,
      sendPort: sendPort,
      receivePort: receivePort,
      createdAt: DateTime.now(),
    );
  }

  Future<R> submit<R>(Future<R> Function(SendPort) task) async {
    if (_closed) throw StateError('Worker pool is closed');

    final completer = Completer<R>();
    final wrappedTask = _Task(
      execute: (sendPort) async {
        final result = await task(sendPort);
        completer.complete(result as dynamic);
      },
      onError: completer.completeError,
    );

    _pendingTasks.add(wrappedTask);
    _processNext();

    return completer.future;
  }

  void _processNext() {
    while (_pendingTasks.isNotEmpty && _availableWorkers.isNotEmpty) {
      final task = _pendingTasks.removeFirst();
      final worker = _availableWorkers.removeFirst();
      _busyWorkers.add(worker);

      worker.execute(task).then((_) {
        _busyWorkers.remove(worker);

        // ตรวจสอบว่า worker หมดอายุหรือยัง
        final age = DateTime.now().difference(worker.createdAt);
        if (age > workerTimeout && _totalWorkers > minWorkers) {
          worker.dispose();
          _totalWorkers--;
        } else {
          _availableWorkers.add(worker);
        }

        _processNext();
      });
    }

    // Scale up ถ้าจำเป็น
    if (_pendingTasks.isNotEmpty && _totalWorkers < maxWorkers) {
      _createWorker().then((worker) {
        _availableWorkers.add(worker);
        _processNext();
      });
    }
  }

  /// ข้อมูลสถานะ pool
  Map<String, int> get stats => {
        'total_workers': _totalWorkers,
        'available': _availableWorkers.length,
        'busy': _busyWorkers.length,
        'pending_tasks': _pendingTasks.length,
      };

  Future<void> close() async {
    _closed = true;

    // รอให้งานที่กำลังทำอยู่เสร็จ
    while (_busyWorkers.isNotEmpty) {
      await Future.delayed(const Duration(milliseconds: 100));
    }

    // ปิด workers ทั้งหมด
    for (final worker in _availableWorkers) {
      worker.dispose();
    }
    for (final worker in _busyWorkers) {
      worker.dispose();
    }

    _availableWorkers.clear();
    _busyWorkers.clear();
  }
}

class _Worker {
  final Isolate isolate;
  final SendPort sendPort;
  final ReceivePort receivePort;
  final DateTime createdAt;

  _Worker({
    required this.isolate,
    required this.sendPort,
    required this.receivePort,
    required this.createdAt,
  });

  Future<void> execute(_Task task) async {
    try {
      await task.execute(sendPort);
    } catch (e) {
      task.onError(e);
    }
  }

  void dispose() {
    isolate.kill();
    receivePort.close();
  }
}

class _Task {
  final Future<void> Function(SendPort) execute;
  final void Function(Object) onError;

  _Task({required this.execute, required this.onError});
}

/// ตัวอย่าง worker entry point สำหรับ pool
void poolWorkerEntry(SendPort mainSendPort) {
  final receivePort = ReceivePort();
  mainSendPort.send(receivePort.sendPort);

  receivePort.listen((message) {
    if (message is! Map<String, dynamic>) return;

    final action = message['action'] as String;
    final data = message['data'];
    final responsePort = message['responsePort'] as SendPort;

    try {
      final result = _handlePoolTask(action, data);
      responsePort.send({'success': true, 'result': result});
    } catch (e) {
      responsePort.send({'success': false, 'error': e.toString()});
    }
  });
}

dynamic _handlePoolTask(String action, dynamic data) {
  switch (action) {
    case 'fibonacci':
      return _fibonacci(data as int);
    case 'sort':
      final list = List<int>.from(data as List);
      list.sort();
      return list;
    case 'primes':
      return _sieve(data as int);
    default:
      throw ArgumentError('Unknown action: $action');
  }
}

int _fibonacci(int n) {
  if (n <= 1) return n;
  return _fibonacci(n - 1) + _fibonacci(n - 2);
}

List<int> _sieve(int limit) {
  final sieve = List<bool>.filled(limit + 1, true);
  sieve[0] = sieve[1] = false;

  for (var i = 2; i * i <= limit; i++) {
    if (sieve[i]) {
      for (var j = i * i; j <= limit; j += i) {
        sieve[j] = false;
      }
    }
  }

  return [
    for (var i = 2; i <= limit; i++)
      if (sieve[i]) i
  ];
}
```

---

## ขั้นตอนที่ 2647: Compute Function Utility

```dart
// lib/isolates/compute_utils.dart
import 'dart:isolate';
import 'dart:async';
import 'dart:typed_data';

/// Utility function คล้าย Flutter's compute() แต่ flexible กว่า
Future<R> compute<Q, R>(R Function(Q) callback, Q message) async {
  final receivePort = ReceivePort();
  await Isolate.spawn(
    _isolateCallback<Q, R>,
    _IsolateCallbackMessage(
      callback: callback,
      message: message,
      sendPort: receivePort.sendPort,
    ),
  );
  return await receivePort.first as R;
}

class _IsolateCallbackMessage<Q, R> {
  final R Function(Q) callback;
  final Q message;
  final SendPort sendPort;

  _IsolateCallbackMessage({
    required this.callback,
    required this.message,
    required this.sendPort,
  });
}

void _isolateCallback<Q, R>(_IsolateCallbackMessage<Q, R> config) {
  final result = config.callback(config.message);
  config.sendPort.send(result);
}

/// Async compute
Future<R> computeAsync<Q, R>(
  Future<R> Function(Q) callback,
  Q message,
) async {
  final receivePort = ReceivePort();
  await Isolate.spawn(
    _asyncIsolateCallback<Q, R>,
    _AsyncCallbackMessage(
      callback: callback,
      message: message,
      sendPort: receivePort.sendPort,
    ),
  );
  return await receivePort.first as R;
}

class _AsyncCallbackMessage<Q, R> {
  final Future<R> Function(Q) callback;
  final Q message;
  final SendPort sendPort;

  _AsyncCallbackMessage({
    required this.callback,
    required this.message,
    required this.sendPort,
  });
}

void _asyncIsolateCallback<Q, R>(
    _AsyncCallbackMessage<Q, R> config) async {
  final result = await config.callback(config.message);
  config.sendPort.send(result);
}

/// ตัวอย่างการใช้งาน
class DataProcessor {
  /// Process large dataset ใน isolate
  static Future<ProcessingResult> processDataset(
    List<Map<String, dynamic>> data,
  ) async {
    return compute(_processDataset, data);
  }

  static ProcessingResult _processDataset(
    List<Map<String, dynamic>> data,
  ) {
    var total = 0.0;
    var count = 0;
    var min = double.infinity;
    var max = double.negativeInfinity;
    final categories = <String, int>{};

    for (final item in data) {
      final value = (item['value'] as num?)?.toDouble() ?? 0;
      total += value;
      count++;

      if (value < min) min = value;
      if (value > max) max = value;

      final category = item['category'] as String? ?? 'unknown';
      categories[category] = (categories[category] ?? 0) + 1;
    }

    return ProcessingResult(
      count: count,
      mean: count > 0 ? total / count : 0,
      min: min.isInfinite ? 0 : min,
      max: max.isInfinite ? 0 : max,
      categories: categories,
    );
  }

  /// Compress data ใน isolate
  static Future<Uint8List> compressData(Uint8List data) async {
    return compute(_compressData, data);
  }

  static Uint8List _compressData(Uint8List data) {
    // Simple RLE compression
    if (data.isEmpty) return data;

    final compressed = <int>[];
    var i = 0;

    while (i < data.length) {
      final current = data[i];
      var count = 1;

      while (i + count < data.length &&
          data[i + count] == current &&
          count < 255) {
        count++;
      }

      compressed.add(count);
      compressed.add(current);
      i += count;
    }

    return Uint8List.fromList(compressed);
  }

  static Uint8List decompressData(Uint8List compressed) {
    final decompressed = <int>[];

    for (var i = 0; i < compressed.length - 1; i += 2) {
      final count = compressed[i];
      final value = compressed[i + 1];
      decompressed.addAll(List.filled(count, value));
    }

    return Uint8List.fromList(decompressed);
  }
}

class ProcessingResult {
  final int count;
  final double mean;
  final double min;
  final double max;
  final Map<String, int> categories;

  const ProcessingResult({
    required this.count,
    required this.mean,
    required this.min,
    required this.max,
    required this.categories,
  });

  @override
  String toString() => 'ProcessingResult(count: $count, '
      'mean: ${mean.toStringAsFixed(2)}, '
      'min: $min, max: $max, '
      'categories: $categories)';
}

/// ทดสอบทุกอย่าง
void main() async {
  print('=== Dart Isolates Advanced Demo ===\n');

  // Test 1: Basic two-way
  print('1. Two-Way Isolate:');
  final twoWay = TwoWayIsolate();
  await twoWay.initialize();
  final computeResult = await twoWay.send({'action': 'compute', 'data': 42});
  print('Compute 42 -> $computeResult');
  twoWay.dispose();

  // Test 2: Parallel processing
  print('\n2. Parallel Processing:');
  final data = List.generate(
    100,
    (i) => {'value': (i * 7.3).toDouble(), 'category': 'cat${i % 3}'},
  );
  final result = await DataProcessor.processDataset(data);
  print('Dataset result: $result');

  // Test 3: Data compression
  print('\n3. Data Compression:');
  final original = Uint8List.fromList(
    List.generate(1000, (i) => i % 10),
  );
  final compressed = await DataProcessor.compressData(original);
  print(
      'Original: ${original.length} bytes, Compressed: ${compressed.length} bytes');

  // Test 4: Resilient isolate
  print('\n4. Resilient Isolate:');
  await demonstrateResilientIsolate();

  print('\n=== Demo Complete ===');
}

// Import ที่จำเป็น (จาก files ก่อนหน้า)
class TwoWayIsolate {
  late final Isolate _isolate;
  late final SendPort _sendPort;
  final _receivePort = ReceivePort();
  late final Stream _responses;

  Future<void> initialize() async {
    _responses = _receivePort.asBroadcastStream();
    _isolate = await Isolate.spawn(_workerMain, _receivePort.sendPort);
    _sendPort = await _responses.first as SendPort;
  }

  Future<dynamic> send(Map<String, dynamic> message) async {
    final responsePort = ReceivePort();
    _sendPort.send({...message, 'responsePort': responsePort.sendPort});
    final result = await responsePort.first;
    responsePort.close();
    return result;
  }

  void dispose() {
    _isolate.kill(priority: Isolate.immediate);
    _receivePort.close();
  }
}

void _workerMain(SendPort mainSendPort) {
  final workerReceivePort = ReceivePort();
  mainSendPort.send(workerReceivePort.sendPort);

  workerReceivePort.listen((message) {
    if (message is! Map<String, dynamic>) return;
    final responsePort = message['responsePort'] as SendPort?;
    final data = message['data'];
    responsePort?.send(data is int ? data * data : data.toString().toUpperCase());
  });
}

Future<void> demonstrateResilientIsolate() async {
  print('Resilient isolate demo...');
  final worker = ResilientIsolate(
    name: 'demo-worker',
    entryPoint: faultyWorker,
    maxRestarts: 2,
    restartDelay: const Duration(milliseconds: 200),
  );

  worker.onRestart.listen((count) => print('  Restarted: $count'));
  await worker.start();

  for (var i = 0; i < 6; i++) {
    try {
      final result = await worker.call('process', 'item-$i');
      print('  Result $i: $result');
    } catch (e) {
      print('  Error $i: $e');
    }
    await Future.delayed(const Duration(milliseconds: 100));
  }

  await worker.stop();
}
```

---

**← [Part 68](part-68-flutter-ml-tflite.md)**
**ต่อไป: [Part 70 →](part-70-flutter-game-development.md)**
