# Part 11: Async/Await และ Futures
## ขั้นตอนที่ 321-360

---

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Asynchronous Programming
- ใช้ Future, async, await
- จัดการ async errors
- ใช้ Stream สำหรับ data streams
- ทำงานกับ Completer
- ใช้ FutureBuilder และ StreamBuilder ใน Flutter

---

## ขั้นตอนที่ 321: Future พื้นฐาน

```dart
import 'dart:async';

// Future คือ Promise ว่าจะมีค่าในอนาคต
// Future<T> = ค่าของ type T ที่ยังไม่เสร็จ

Future<String> fetchUserName(String id) async {
  // จำลอง network delay
  await Future.delayed(const Duration(seconds: 1));
  
  if (id == 'invalid') throw Exception('User not found');
  
  return 'User_$id';
}

Future<int> fetchUserAge(String name) async {
  await Future.delayed(const Duration(milliseconds: 500));
  return name.length * 3;  // จำลอง
}

void main() async {
  print('Start');
  
  // ─── await คือรอ Future ให้เสร็จ ───
  String name = await fetchUserName('alice');
  print('Name: $name');
  
  int age = await fetchUserAge(name);
  print('Age: $age');
  
  // ─── Sequential vs Parallel ───
  
  // Sequential (ช้า): รอทีละตัว
  print('\n--- Sequential ---');
  Stopwatch sw = Stopwatch()..start();
  String n1 = await fetchUserName('bob');
  String n2 = await fetchUserName('charlie');
  print('Sequential: ${sw.elapsedMilliseconds}ms');
  
  // Parallel (เร็ว): รอพร้อมกัน
  print('\n--- Parallel ---');
  sw.reset();
  List<String> names = await Future.wait([
    fetchUserName('bob'),
    fetchUserName('charlie'),
    fetchUserName('dave'),
  ]);
  print('Parallel: ${sw.elapsedMilliseconds}ms');
  print('Names: $names');
  
  // ─── Future.value และ Future.error ───
  Future<int> immediate = Future.value(42);
  Future<int> failed = Future.error(Exception('Something went wrong'));
  
  int v = await immediate;
  print('Immediate: $v');
  
  try {
    await failed;
  } catch (e) {
    print('Failed: $e');
  }
  
  print('End');
}
```

---

## ขั้นตอนที่ 322: async/await patterns

```dart
import 'dart:async';

// ─── Loading Pattern ───
class DataLoader {
  bool _isLoading = false;
  String? _data;
  String? _error;
  
  Future<void> loadData() async {
    _isLoading = true;
    _error = null;
    print('Loading...');
    
    try {
      await Future.delayed(const Duration(seconds: 1));
      _data = 'Loaded data at ${DateTime.now()}';
      print('Success: $_data');
    } catch (e) {
      _error = e.toString();
      print('Error: $_error');
    } finally {
      _isLoading = false;
      print('Done loading');
    }
  }
}

// ─── Retry Pattern ───
Future<T> withRetry<T>(
  Future<T> Function() operation, {
  int maxRetries = 3,
  Duration delay = const Duration(seconds: 1),
}) async {
  int attempt = 0;
  
  while (true) {
    try {
      return await operation();
    } catch (e) {
      attempt++;
      if (attempt >= maxRetries) rethrow;
      
      print('Attempt $attempt failed, retrying in $delay...');
      await Future.delayed(delay * attempt);
    }
  }
}

// ─── Timeout Pattern ───
Future<T> withTimeout<T>(
  Future<T> Function() operation,
  Duration timeout,
) async {
  try {
    return await operation().timeout(timeout);
  } on TimeoutException {
    throw TimeoutException('Operation timed out after $timeout');
  }
}

// ─── Debounce Pattern (ใช้บ่อยใน Search) ───
class Debouncer {
  final Duration delay;
  Timer? _timer;
  
  Debouncer({required this.delay});
  
  void run(void Function() action) {
    _timer?.cancel();
    _timer = Timer(delay, action);
  }
  
  void dispose() => _timer?.cancel();
}

// ─── Cache Pattern ───
class CachedFuture<T> {
  final Duration ttl;
  final Future<T> Function() fetcher;
  
  T? _cached;
  DateTime? _cachedAt;
  
  CachedFuture({required this.ttl, required this.fetcher});
  
  Future<T> get() async {
    if (_cached != null && _cachedAt != null) {
      if (DateTime.now().difference(_cachedAt!) < ttl) {
        print('Cache hit');
        return _cached!;
      }
    }
    
    print('Cache miss, fetching...');
    _cached = await fetcher();
    _cachedAt = DateTime.now();
    return _cached!;
  }
  
  void invalidate() {
    _cached = null;
    _cachedAt = null;
  }
}

void main() async {
  // Retry
  print('--- Retry ---');
  int attempts = 0;
  try {
    String result = await withRetry(
      () async {
        attempts++;
        if (attempts < 3) throw Exception('Temporary error');
        return 'Success after $attempts attempts';
      },
      maxRetries: 5,
      delay: const Duration(milliseconds: 100),
    );
    print(result);
  } catch (e) {
    print('All retries failed: $e');
  }
  
  // Cache
  print('\n--- Cache ---');
  CachedFuture<String> cache = CachedFuture(
    ttl: const Duration(seconds: 5),
    fetcher: () async {
      await Future.delayed(const Duration(milliseconds: 200));
      return 'data_${DateTime.now().millisecondsSinceEpoch}';
    },
  );
  
  String d1 = await cache.get();
  String d2 = await cache.get();  // cache hit
  print('d1: $d1');
  print('d2: $d2');
  print('Same: ${d1 == d2}');
}
```

---

## ขั้นตอนที่ 323: Stream

```dart
import 'dart:async';

// Stream = ชุดของ events ที่เกิดขึ้นตามเวลา
// เหมาะกับ: real-time data, events, websocket

// ─── Single-subscription Stream ───
Stream<int> countDown(int from) async* {
  for (int i = from; i >= 0; i--) {
    yield i;
    await Future.delayed(const Duration(milliseconds: 500));
  }
}

// ─── Stream operations ───
Stream<String> userStream() async* {
  List<String> users = ['Alice', 'Bob', 'Charlie', 'Dave', 'Eve'];
  for (String user in users) {
    await Future.delayed(const Duration(milliseconds: 200));
    yield user;
  }
}

// ─── StreamController ───
class EventBus {
  final StreamController<Map<String, dynamic>> _controller =
      StreamController.broadcast();
  
  Stream<Map<String, dynamic>> get stream => _controller.stream;
  
  void emit(String event, dynamic data) {
    _controller.add({'event': event, 'data': data, 'time': DateTime.now()});
  }
  
  void dispose() => _controller.close();
}

void main() async {
  // Basic stream
  print('--- Countdown ---');
  await for (int n in countDown(5)) {
    print(n == 0 ? 'Go!' : '$n...');
  }
  
  // Stream operations
  print('\n--- Stream Operations ---');
  await userStream()
      .where((name) => name.length > 3)         // filter
      .map((name) => name.toUpperCase())         // transform
      .take(3)                                    // take first 3
      .forEach((name) => print(name));
  
  // Stream listen
  print('\n--- Stream Listen ---');
  StreamSubscription<int> sub = Stream.periodic(
    const Duration(milliseconds: 300),
    (i) => i,
  ).take(5).listen(
    (data) => print('Data: $data'),
    onError: (e) => print('Error: $e'),
    onDone: () => print('Stream done'),
  );
  
  await Future.delayed(const Duration(seconds: 2));
  
  // EventBus
  print('\n--- EventBus ---');
  EventBus bus = EventBus();
  
  bus.stream
      .where((e) => e['event'] == 'click')
      .listen((e) => print('Click: ${e['data']}'));
  
  bus.emit('click', {'x': 100, 'y': 200});
  bus.emit('hover', {'x': 150, 'y': 250});
  bus.emit('click', {'x': 300, 'y': 400});
  
  await Future.delayed(const Duration(milliseconds: 100));
  bus.dispose();
}
```

---

## ขั้นตอนที่ 324: FutureBuilder ใน Flutter

```dart
import 'package:flutter/material.dart';
import 'dart:async';

// จำลอง API call
Future<List<Map<String, dynamic>>> fetchPosts() async {
  await Future.delayed(const Duration(seconds: 2));
  
  // จำลอง error (ลอง uncomment เพื่อดู error state)
  // throw Exception('Network Error');
  
  return List.generate(10, (i) => {
    'id': i + 1,
    'title': 'โพสต์ที่ ${i + 1}',
    'body': 'เนื้อหาของโพสต์ที่ ${i + 1} มีรายละเอียดมากมาย',
    'author': 'User ${i % 3 + 1}',
  });
}

class FutureBuilderExample extends StatefulWidget {
  const FutureBuilderExample({super.key});
  
  @override
  State<FutureBuilderExample> createState() => _FutureBuilderExampleState();
}

class _FutureBuilderExampleState extends State<FutureBuilderExample> {
  late Future<List<Map<String, dynamic>>> _postsFuture;
  
  @override
  void initState() {
    super.initState();
    _postsFuture = fetchPosts();
  }
  
  void _refresh() {
    setState(() {
      _postsFuture = fetchPosts();
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('FutureBuilder'),
        actions: [
          IconButton(icon: const Icon(Icons.refresh), onPressed: _refresh),
        ],
      ),
      body: FutureBuilder<List<Map<String, dynamic>>>(
        future: _postsFuture,
        builder: (context, snapshot) {
          // Loading state
          if (snapshot.connectionState == ConnectionState.waiting) {
            return const Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  CircularProgressIndicator(),
                  SizedBox(height: 16),
                  Text('กำลังโหลด...'),
                ],
              ),
            );
          }
          
          // Error state
          if (snapshot.hasError) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Icon(Icons.error_outline, size: 64, color: Colors.red),
                  const SizedBox(height: 16),
                  Text('เกิดข้อผิดพลาด: ${snapshot.error}'),
                  const SizedBox(height: 16),
                  ElevatedButton(
                    onPressed: _refresh,
                    child: const Text('ลองใหม่'),
                  ),
                ],
              ),
            );
          }
          
          // Success state
          List<Map<String, dynamic>> posts = snapshot.data ?? [];
          
          if (posts.isEmpty) {
            return const Center(child: Text('ไม่มีข้อมูล'));
          }
          
          return ListView.builder(
            itemCount: posts.length,
            itemBuilder: (context, index) {
              Map<String, dynamic> post = posts[index];
              return ListTile(
                leading: CircleAvatar(child: Text('${post['id']}')),
                title: Text(post['title'] as String),
                subtitle: Text(post['body'] as String, maxLines: 1, overflow: TextOverflow.ellipsis),
                trailing: Text(post['author'] as String, style: const TextStyle(fontSize: 12)),
              );
            },
          );
        },
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 325: StreamBuilder ใน Flutter

```dart
import 'package:flutter/material.dart';
import 'dart:async';

// จำลอง real-time stock prices
Stream<Map<String, double>> stockPriceStream() async* {
  Map<String, double> prices = {
    'AAPL': 180.0,
    'GOOG': 140.0,
    'MSFT': 370.0,
    'TSLA': 250.0,
  };
  
  while (true) {
    await Future.delayed(const Duration(seconds: 1));
    
    // Random price change
    prices = prices.map((key, value) {
      double change = (value * 0.02 * (DateTime.now().millisecond % 3 - 1));
      return MapEntry(key, (value + change).clamp(1.0, 10000.0));
    });
    
    yield Map.from(prices);
  }
}

class StockTickerScreen extends StatefulWidget {
  const StockTickerScreen({super.key});
  
  @override
  State<StockTickerScreen> createState() => _StockTickerScreenState();
}

class _StockTickerScreenState extends State<StockTickerScreen> {
  late Stream<Map<String, double>> _stream;
  Map<String, double>? _previousPrices;
  
  @override
  void initState() {
    super.initState();
    _stream = stockPriceStream().asBroadcastStream();
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Stock Ticker'),
        actions: [
          Container(
            margin: const EdgeInsets.symmetric(vertical: 12, horizontal: 8),
            padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
            decoration: BoxDecoration(
              color: Colors.green.withOpacity(0.2),
              borderRadius: BorderRadius.circular(4),
            ),
            child: const Row(
              children: [
                Icon(Icons.circle, color: Colors.green, size: 8),
                SizedBox(width: 4),
                Text('Live', style: TextStyle(color: Colors.green)),
              ],
            ),
          ),
        ],
      ),
      body: StreamBuilder<Map<String, double>>(
        stream: _stream,
        builder: (context, snapshot) {
          if (!snapshot.hasData) {
            return const Center(child: CircularProgressIndicator());
          }
          
          Map<String, double> prices = snapshot.data!;
          
          return ListView.builder(
            padding: const EdgeInsets.all(16),
            itemCount: prices.length,
            itemBuilder: (context, index) {
              String symbol = prices.keys.elementAt(index);
              double price = prices[symbol]!;
              double? prevPrice = _previousPrices?[symbol];
              
              Color priceColor = prevPrice == null
                  ? Colors.black
                  : price > prevPrice
                      ? Colors.green
                      : price < prevPrice
                          ? Colors.red
                          : Colors.black;
              
              IconData priceIcon = prevPrice == null
                  ? Icons.remove
                  : price > prevPrice
                      ? Icons.arrow_upward
                      : price < prevPrice
                          ? Icons.arrow_downward
                          : Icons.remove;
              
              return Card(
                margin: const EdgeInsets.only(bottom: 8),
                child: ListTile(
                  title: Text(symbol, style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 18)),
                  subtitle: Text(
                    prevPrice != null
                        ? '${price > prevPrice ? '+' : ''}${(price - prevPrice).toStringAsFixed(2)}'
                        : 'Real-time price',
                    style: TextStyle(color: priceColor),
                  ),
                  trailing: Row(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      Icon(priceIcon, color: priceColor, size: 16),
                      const SizedBox(width: 4),
                      Text(
                        '\$${price.toStringAsFixed(2)}',
                        style: TextStyle(
                          color: priceColor,
                          fontSize: 20,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                    ],
                  ),
                ),
              );
            },
          );
        },
      ),
    );
  }
  
  @override
  void dispose() {
    super.dispose();
  }
}
```

---

## ขั้นตอนที่ 326-340: Completer และ Advanced Patterns

```dart
import 'dart:async';

// ─── Completer: สร้าง Future ด้วยตัวเอง ───
class DialogResult {
  static Completer<bool?>? _completer;
  
  static Future<bool?> show() {
    _completer = Completer<bool?>();
    
    // จำลอง user clicking dialog after 1 second
    Future.delayed(const Duration(seconds: 1), () {
      _completer?.complete(true);  // User clicked OK
    });
    
    return _completer!.future;
  }
  
  static void cancel() {
    _completer?.complete(null);
  }
}

// ─── Isolate-safe operations ───
// Dart ทำงานแบบ single-threaded ด้วย event loop
// หากต้องการ CPU-intensive work ใช้ compute() ใน Flutter

// ─── Zone ───
// Zone ใช้สำหรับ error handling ระดับ global
void runWithZone() {
  runZonedGuarded(
    () async {
      throw Exception('Uncaught error in zone');
    },
    (error, stack) {
      print('Zone caught: $error');
    },
  );
}

// ─── Future Chain ───
Future<Map<String, dynamic>> buildUserProfile(String userId) {
  return Future.value(userId)
      .then((id) async {
        await Future.delayed(const Duration(milliseconds: 100));
        return {'id': id, 'name': 'User_$id'};
      })
      .then((user) async {
        await Future.delayed(const Duration(milliseconds: 50));
        user['email'] = '${user['id']}@example.com';
        return user;
      })
      .then((user) async {
        user['avatar'] = 'https://avatar.example.com/${user['id']}';
        return user;
      });
}

// ─── async* generator ───
Stream<Map<String, dynamic>> processUsers(List<String> ids) async* {
  for (String id in ids) {
    await Future.delayed(const Duration(milliseconds: 100));
    yield await buildUserProfile(id);
  }
}

void main() async {
  // Completer
  print('--- Completer ---');
  bool? result = await DialogResult.show();
  print('Dialog result: $result');
  
  // Future chain
  print('\n--- Future Chain ---');
  Map<String, dynamic> profile = await buildUserProfile('alice');
  print('Profile: $profile');
  
  // async* generator
  print('\n--- Process Users ---');
  await for (Map<String, dynamic> user in processUsers(['alice', 'bob', 'charlie'])) {
    print('Processed: ${user['name']} (${user['email']})');
  }
  
  // Zone error handling
  print('\n--- Zone ---');
  runWithZone();
  await Future.delayed(const Duration(milliseconds: 100));
}
```

---

## ขั้นตอนที่ 341-360: โปรเจกต์ - Weather App

```dart
// weather_app.dart
import 'package:flutter/material.dart';
import 'dart:async';

void main() {
  runApp(const WeatherApp());
}

class WeatherApp extends StatelessWidget {
  const WeatherApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Weather App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      home: const WeatherScreen(),
    );
  }
}

// ─── Model ───
class WeatherData {
  final String city;
  final double temperature;
  final double feelsLike;
  final String condition;
  final int humidity;
  final double windSpeed;
  final List<HourlyForecast> hourly;
  final List<DailyForecast> daily;
  
  const WeatherData({
    required this.city,
    required this.temperature,
    required this.feelsLike,
    required this.condition,
    required this.humidity,
    required this.windSpeed,
    required this.hourly,
    required this.daily,
  });
}

class HourlyForecast {
  final String time;
  final double temp;
  final String icon;
  const HourlyForecast(this.time, this.temp, this.icon);
}

class DailyForecast {
  final String day;
  final double high;
  final double low;
  final String condition;
  const DailyForecast(this.day, this.high, this.low, this.condition);
}

// ─── Fake Weather Service ───
class WeatherService {
  Future<WeatherData> fetchWeather(String city) async {
    await Future.delayed(const Duration(seconds: 1));
    
    if (city.toLowerCase() == 'unknown') {
      throw Exception('City not found: $city');
    }
    
    return WeatherData(
      city: city,
      temperature: 28 + (city.length % 5).toDouble(),
      feelsLike: 31,
      condition: 'Partly Cloudy',
      humidity: 75,
      windSpeed: 12.5,
      hourly: List.generate(8, (i) => HourlyForecast(
        '${(DateTime.now().hour + i) % 24}:00',
        28 + (i % 3) - 1,
        i % 2 == 0 ? '🌤' : '⛅',
      )),
      daily: [
        const DailyForecast('วันนี้', 31, 24, '🌤 Partly Cloudy'),
        const DailyForecast('พรุ่งนี้', 29, 22, '🌧 Rain'),
        const DailyForecast('มะรืน', 33, 25, '☀️ Sunny'),
        const DailyForecast('ว.พ.', 30, 23, '⛅ Cloudy'),
        const DailyForecast('ว.พฤ.', 28, 22, '🌧 Showers'),
        const DailyForecast('ว.ศ.', 32, 25, '☀️ Sunny'),
        const DailyForecast('ว.ส.', 31, 24, '🌤 Partly Cloudy'),
      ],
    );
  }
  
  Stream<WeatherData> liveWeather(String city) async* {
    while (true) {
      yield await fetchWeather(city);
      await Future.delayed(const Duration(seconds: 30));
    }
  }
}

// ─── Screen ───
class WeatherScreen extends StatefulWidget {
  const WeatherScreen({super.key});
  
  @override
  State<WeatherScreen> createState() => _WeatherScreenState();
}

class _WeatherScreenState extends State<WeatherScreen> {
  final WeatherService _service = WeatherService();
  String _city = 'Bangkok';
  late Future<WeatherData> _weatherFuture;
  
  @override
  void initState() {
    super.initState();
    _weatherFuture = _service.fetchWeather(_city);
  }
  
  void _changeCity(String city) {
    setState(() {
      _city = city;
      _weatherFuture = _service.fetchWeather(city);
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: FutureBuilder<WeatherData>(
        future: _weatherFuture,
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return Container(
              decoration: const BoxDecoration(
                gradient: LinearGradient(
                  colors: [Colors.blue, Colors.lightBlue],
                  begin: Alignment.topCenter,
                  end: Alignment.bottomCenter,
                ),
              ),
              child: const Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    CircularProgressIndicator(color: Colors.white),
                    SizedBox(height: 16),
                    Text('กำลังดึงข้อมูลสภาพอากาศ...', style: TextStyle(color: Colors.white)),
                  ],
                ),
              ),
            );
          }
          
          if (snapshot.hasError) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Icon(Icons.cloud_off, size: 64, color: Colors.grey),
                  const SizedBox(height: 16),
                  Text('ไม่สามารถดึงข้อมูลได้: ${snapshot.error}'),
                  const SizedBox(height: 16),
                  ElevatedButton(
                    onPressed: () => _changeCity(_city),
                    child: const Text('ลองใหม่'),
                  ),
                ],
              ),
            );
          }
          
          WeatherData weather = snapshot.data!;
          
          return _WeatherView(
            weather: weather,
            onCityChange: _changeCity,
          );
        },
      ),
    );
  }
}

class _WeatherView extends StatelessWidget {
  final WeatherData weather;
  final void Function(String) onCityChange;
  
  const _WeatherView({required this.weather, required this.onCityChange});
  
  @override
  Widget build(BuildContext context) {
    return Container(
      decoration: const BoxDecoration(
        gradient: LinearGradient(
          colors: [Color(0xFF1565C0), Color(0xFF42A5F5)],
          begin: Alignment.topCenter,
          end: Alignment.bottomCenter,
        ),
      ),
      child: SafeArea(
        child: CustomScrollView(
          slivers: [
            // City selector
            SliverToBoxAdapter(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Row(
                  children: [
                    Expanded(
                      child: Text(
                        weather.city,
                        style: const TextStyle(color: Colors.white, fontSize: 24, fontWeight: FontWeight.bold),
                      ),
                    ),
                    IconButton(
                      icon: const Icon(Icons.search, color: Colors.white),
                      onPressed: () {
                        showDialog(
                          context: context,
                          builder: (ctx) => _CityDialog(
                            onSelect: (city) {
                              Navigator.pop(ctx);
                              onCityChange(city);
                            },
                          ),
                        );
                      },
                    ),
                  ],
                ),
              ),
            ),
            
            // Main temperature
            SliverToBoxAdapter(
              child: Padding(
                padding: const EdgeInsets.symmetric(vertical: 24),
                child: Column(
                  children: [
                    const Text('☁️', style: TextStyle(fontSize: 72)),
                    Text(
                      '${weather.temperature.toInt()}°',
                      style: const TextStyle(color: Colors.white, fontSize: 80, fontWeight: FontWeight.w200),
                    ),
                    Text(
                      weather.condition,
                      style: const TextStyle(color: Colors.white70, fontSize: 20),
                    ),
                    const SizedBox(height: 8),
                    Text(
                      'รู้สึกเหมือน ${weather.feelsLike.toInt()}°',
                      style: const TextStyle(color: Colors.white60),
                    ),
                  ],
                ),
              ),
            ),
            
            // Stats
            SliverToBoxAdapter(
              child: Padding(
                padding: const EdgeInsets.symmetric(horizontal: 16),
                child: Row(
                  mainAxisAlignment: MainAxisAlignment.spaceAround,
                  children: [
                    _StatItem(icon: Icons.water_drop, label: 'ความชื้น', value: '${weather.humidity}%'),
                    _StatItem(icon: Icons.air, label: 'ลม', value: '${weather.windSpeed} km/h'),
                    _StatItem(icon: Icons.visibility, label: 'ทัศนวิสัย', value: '10 km'),
                  ],
                ),
              ),
            ),
            
            const SliverToBoxAdapter(child: SizedBox(height: 16)),
            
            // Hourly forecast
            SliverToBoxAdapter(
              child: Container(
                margin: const EdgeInsets.all(16),
                padding: const EdgeInsets.all(16),
                decoration: BoxDecoration(
                  color: Colors.white.withOpacity(0.15),
                  borderRadius: BorderRadius.circular(16),
                ),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    const Text('พยากรณ์รายชั่วโมง', style: TextStyle(color: Colors.white70, fontSize: 12)),
                    const SizedBox(height: 12),
                    SizedBox(
                      height: 80,
                      child: ListView.builder(
                        scrollDirection: Axis.horizontal,
                        itemCount: weather.hourly.length,
                        itemBuilder: (ctx, i) {
                          HourlyForecast h = weather.hourly[i];
                          return Padding(
                            padding: const EdgeInsets.symmetric(horizontal: 12),
                            child: Column(
                              children: [
                                Text(h.time, style: const TextStyle(color: Colors.white70, fontSize: 12)),
                                const SizedBox(height: 4),
                                Text(h.icon, style: const TextStyle(fontSize: 24)),
                                const SizedBox(height: 4),
                                Text('${h.temp.toInt()}°', style: const TextStyle(color: Colors.white)),
                              ],
                            ),
                          );
                        },
                      ),
                    ),
                  ],
                ),
              ),
            ),
            
            // Daily forecast
            SliverToBoxAdapter(
              child: Container(
                margin: const EdgeInsets.fromLTRB(16, 0, 16, 16),
                padding: const EdgeInsets.all(16),
                decoration: BoxDecoration(
                  color: Colors.white.withOpacity(0.15),
                  borderRadius: BorderRadius.circular(16),
                ),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    const Text('7 วันข้างหน้า', style: TextStyle(color: Colors.white70, fontSize: 12)),
                    const SizedBox(height: 12),
                    ...weather.daily.map((d) => Padding(
                      padding: const EdgeInsets.symmetric(vertical: 6),
                      child: Row(
                        children: [
                          SizedBox(width: 80, child: Text(d.day, style: const TextStyle(color: Colors.white))),
                          Text(d.condition.split(' ')[0], style: const TextStyle(fontSize: 20)),
                          const Spacer(),
                          Text('${d.low.toInt()}°', style: const TextStyle(color: Colors.white60)),
                          const SizedBox(width: 12),
                          Text('${d.high.toInt()}°', style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
                        ],
                      ),
                    )),
                  ],
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _StatItem extends StatelessWidget {
  final IconData icon;
  final String label;
  final String value;
  
  const _StatItem({required this.icon, required this.label, required this.value});
  
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Icon(icon, color: Colors.white70, size: 24),
        const SizedBox(height: 4),
        Text(value, style: const TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
        Text(label, style: const TextStyle(color: Colors.white60, fontSize: 12)),
      ],
    );
  }
}

class _CityDialog extends StatefulWidget {
  final void Function(String) onSelect;
  
  const _CityDialog({required this.onSelect});
  
  @override
  State<_CityDialog> createState() => _CityDialogState();
}

class _CityDialogState extends State<_CityDialog> {
  final List<String> cities = ['Bangkok', 'Chiang Mai', 'Phuket', 'Pattaya', 'Hua Hin'];
  
  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: const Text('เลือกเมือง'),
      content: Column(
        mainAxisSize: MainAxisSize.min,
        children: cities.map((city) => ListTile(
          title: Text(city),
          onTap: () => widget.onSelect(city),
        )).toList(),
      ),
    );
  }
}

void main() {
  runApp(const WeatherApp());
}
```

---

**← [Part 10 - Layout Widgets](part-10-layout-widgets.md)**

**ต่อไป: [Part 12 - State Management (setState & Provider) →](part-12-state-management.md)**
