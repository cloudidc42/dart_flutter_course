# Part 50: Performance Optimization in Flutter
## ขั้นตอนที่ 1881-1920

## 🎯 เป้าหมายของ Part นี้
- Optimize images ด้วย cached_network_image และ flutter_image_compress
- ใช้ Lazy Loading patterns สำหรับ lists และ data
- ย้ายงานหนักออกจาก main thread ด้วย compute()
- Optimize SliverList และ SliverGrid สำหรับ large datasets
- เข้าใจความแตกต่างระหว่าง debug / profile / release modes
- ลดขนาด app bundle

---

## ขั้นตอนที่ 1881: pubspec.yaml for Performance

```yaml
# pubspec.yaml
name: flutter_performance_demo
description: Flutter Performance Optimization Demo
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter
  cached_network_image: ^3.3.1
  flutter_image_compress: ^2.1.0
  image_picker: ^1.0.4
  visibility_detector: ^0.4.0+2
  scroll_to_index: ^3.0.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0

flutter:
  uses-material-design: true
  assets:
    - assets/images/placeholder.png
```

---

## ขั้นตอนที่ 1882: Image Optimization ด้วย cached_network_image

```dart
// lib/widgets/optimized_image.dart
import 'package:flutter/material.dart';
import 'package:cached_network_image/cached_network_image.dart';

class OptimizedNetworkImage extends StatelessWidget {
  final String imageUrl;
  final double? width;
  final double? height;
  final BoxFit fit;
  final BorderRadius? borderRadius;

  const OptimizedNetworkImage({
    super.key,
    required this.imageUrl,
    this.width,
    this.height,
    this.fit = BoxFit.cover,
    this.borderRadius,
  });

  @override
  Widget build(BuildContext context) {
    Widget image = CachedNetworkImage(
      imageUrl: imageUrl,
      width: width,
      height: height,
      fit: fit,
      // Memory cache: max 100 images, 50MB
      memCacheWidth: width?.toInt(),
      memCacheHeight: height?.toInt(),
      // Show placeholder while loading
      placeholder: (context, url) => Container(
        width: width,
        height: height,
        color: Colors.grey[300],
        child: const Center(
          child: CircularProgressIndicator(strokeWidth: 2),
        ),
      ),
      // Show error widget on failure
      errorWidget: (context, url, error) => Container(
        width: width,
        height: height,
        color: Colors.grey[200],
        child: const Icon(Icons.broken_image, color: Colors.grey),
      ),
      // Fade in animation
      fadeInDuration: const Duration(milliseconds: 300),
      fadeOutDuration: const Duration(milliseconds: 100),
    );

    if (borderRadius != null) {
      return ClipRRect(borderRadius: borderRadius!, child: image);
    }
    return image;
  }
}

// Custom cache manager with size limits
import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter_cache_manager/flutter_cache_manager.dart';

class AppCacheManager extends CacheManager with ImageCacheManager {
  static const key = 'appCacheManager';

  static final AppCacheManager _instance = AppCacheManager._();
  factory AppCacheManager() => _instance;

  AppCacheManager._()
      : super(Config(
          key,
          stalePeriod: const Duration(days: 7),
          maxNrOfCacheObjects: 200,
          repo: JsonCacheInfoRepository(databaseName: key),
          fileService: HttpFileService(),
        ));
}

// Use with custom cache manager:
class ProductImage extends StatelessWidget {
  final String url;
  const ProductImage({super.key, required this.url});

  @override
  Widget build(BuildContext context) {
    return CachedNetworkImage(
      imageUrl: url,
      cacheManager: AppCacheManager(),
      width: 200,
      height: 200,
      fit: BoxFit.cover,
      placeholder: (_, __) => const SizedBox(
        width: 200,
        height: 200,
        child: Center(child: CircularProgressIndicator()),
      ),
      errorWidget: (_, __, ___) => const Icon(Icons.error),
    );
  }
}
```

---

## ขั้นตอนที่ 1883: Image Compression

```dart
// lib/services/image_compression_service.dart
import 'dart:io';
import 'dart:typed_data';
import 'package:flutter/foundation.dart';
import 'package:flutter_image_compress/flutter_image_compress.dart';
import 'package:image_picker/image_picker.dart';
import 'package:path_provider/path_provider.dart';
import 'package:path/path.dart' as path;

class ImageCompressionService {
  static const int _defaultQuality = 85;
  static const int _thumbnailQuality = 60;
  static const int _maxDimension = 1920;

  /// Compress image from XFile (image_picker) and return compressed bytes
  static Future<Uint8List?> compressFromXFile(
    XFile file, {
    int quality = _defaultQuality,
    int maxWidth = _maxDimension,
    int maxHeight = _maxDimension,
  }) async {
    final bytes = await file.readAsBytes();
    return compressBytes(
      bytes,
      quality: quality,
      maxWidth: maxWidth,
      maxHeight: maxHeight,
    );
  }

  /// Compress bytes directly
  static Future<Uint8List?> compressBytes(
    Uint8List bytes, {
    int quality = _defaultQuality,
    int maxWidth = _maxDimension,
    int maxHeight = _maxDimension,
  }) async {
    return FlutterImageCompress.compressWithList(
      bytes,
      quality: quality,
      minWidth: maxWidth,
      minHeight: maxHeight,
      rotate: 0,
      autoCorrectionAngle: true,
      format: CompressFormat.jpeg,
    );
  }

  /// Compress to specific file size (iterative)
  static Future<Uint8List?> compressToTargetSize(
    Uint8List bytes, {
    int targetSizeKB = 500,
    int minQuality = 30,
  }) async {
    int quality = _defaultQuality;
    Uint8List? compressed;

    while (quality >= minQuality) {
      compressed = await FlutterImageCompress.compressWithList(
        bytes,
        quality: quality,
      );

      if (compressed == null) break;

      final sizeKB = compressed.length / 1024;
      if (sizeKB <= targetSizeKB) break;

      quality -= 10;
    }

    return compressed;
  }

  /// Create thumbnail from bytes
  static Future<Uint8List?> createThumbnail(
    Uint8List bytes, {
    int size = 200,
  }) async {
    return FlutterImageCompress.compressWithList(
      bytes,
      quality: _thumbnailQuality,
      minWidth: size,
      minHeight: size,
    );
  }

  /// Compress file and save to temp directory
  static Future<File?> compressFile(
    File file, {
    int quality = _defaultQuality,
  }) async {
    final dir = await getTemporaryDirectory();
    final ext = path.extension(file.path).toLowerCase();
    final targetPath = path.join(
      dir.path,
      '${DateTime.now().millisecondsSinceEpoch}_compressed$ext',
    );

    final result = await FlutterImageCompress.compressAndGetFile(
      file.path,
      targetPath,
      quality: quality,
      autoCorrectionAngle: true,
    );

    return result != null ? File(result.path) : null;
  }

  /// Get file size in human-readable format
  static String formatFileSize(int bytes) {
    if (bytes < 1024) return '$bytes B';
    if (bytes < 1024 * 1024) return '${(bytes / 1024).toStringAsFixed(1)} KB';
    return '${(bytes / (1024 * 1024)).toStringAsFixed(2)} MB';
  }
}

// Example usage in a screen
class ImagePickerScreen extends StatefulWidget {
  const ImagePickerScreen({super.key});

  @override
  State<ImagePickerScreen> createState() => _ImagePickerScreenState();
}

class _ImagePickerScreenState extends State<ImagePickerScreen> {
  final ImagePicker _picker = ImagePicker();
  Uint8List? _originalBytes;
  Uint8List? _compressedBytes;
  bool _isCompressing = false;

  Future<void> _pickAndCompress() async {
    final XFile? image = await _picker.pickImage(source: ImageSource.gallery);
    if (image == null) return;

    final bytes = await image.readAsBytes();
    setState(() {
      _originalBytes = bytes;
      _isCompressing = true;
    });

    final compressed = await ImageCompressionService.compressToTargetSize(
      bytes,
      targetSizeKB: 200,
    );

    setState(() {
      _compressedBytes = compressed;
      _isCompressing = false;
    });

    if (mounted && compressed != null) {
      final origKB = bytes.length / 1024;
      final compKB = compressed.length / 1024;
      final reduction = ((1 - compKB / origKB) * 100).toStringAsFixed(1);
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text(
            'Compressed: ${origKB.toStringAsFixed(0)}KB → '
            '${compKB.toStringAsFixed(0)}KB ($reduction% smaller)',
          ),
        ),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Image Compression')),
      body: Column(
        children: [
          ElevatedButton.icon(
            onPressed: _isCompressing ? null : _pickAndCompress,
            icon: const Icon(Icons.image),
            label: const Text('Pick & Compress Image'),
          ),
          if (_isCompressing)
            const Padding(
              padding: EdgeInsets.all(16),
              child: CircularProgressIndicator(),
            ),
          if (_originalBytes != null && _compressedBytes != null)
            Row(
              children: [
                Expanded(
                  child: Column(
                    children: [
                      const Text('Original'),
                      Image.memory(_originalBytes!, height: 200, fit: BoxFit.contain),
                      Text(ImageCompressionService.formatFileSize(_originalBytes!.length)),
                    ],
                  ),
                ),
                Expanded(
                  child: Column(
                    children: [
                      const Text('Compressed'),
                      Image.memory(_compressedBytes!, height: 200, fit: BoxFit.contain),
                      Text(ImageCompressionService.formatFileSize(_compressedBytes!.length)),
                    ],
                  ),
                ),
              ],
            ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1884: Lazy Loading Patterns

```dart
// lib/widgets/lazy_loading_list.dart
import 'package:flutter/material.dart';

class LazyLoadingList<T> extends StatefulWidget {
  final Future<List<T>> Function(int page, int pageSize) fetchPage;
  final Widget Function(BuildContext context, T item, int index) itemBuilder;
  final int pageSize;
  final Widget? loadingWidget;
  final Widget? emptyWidget;
  final Widget Function(String error, VoidCallback retry)? errorBuilder;

  const LazyLoadingList({
    super.key,
    required this.fetchPage,
    required this.itemBuilder,
    this.pageSize = 20,
    this.loadingWidget,
    this.emptyWidget,
    this.errorBuilder,
  });

  @override
  State<LazyLoadingList<T>> createState() => _LazyLoadingListState<T>();
}

class _LazyLoadingListState<T> extends State<LazyLoadingList<T>> {
  final List<T> _items = [];
  final ScrollController _scrollController = ScrollController();
  int _currentPage = 0;
  bool _isLoading = false;
  bool _hasMore = true;
  String? _error;

  @override
  void initState() {
    super.initState();
    _loadNextPage();
    _scrollController.addListener(_onScroll);
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  void _onScroll() {
    if (_scrollController.position.pixels >=
        _scrollController.position.maxScrollExtent - 300) {
      _loadNextPage();
    }
  }

  Future<void> _loadNextPage() async {
    if (_isLoading || !_hasMore) return;

    setState(() {
      _isLoading = true;
      _error = null;
    });

    try {
      final newItems = await widget.fetchPage(_currentPage, widget.pageSize);
      setState(() {
        _items.addAll(newItems);
        _currentPage++;
        _isLoading = false;
        _hasMore = newItems.length >= widget.pageSize;
      });
    } catch (e) {
      setState(() {
        _isLoading = false;
        _error = e.toString();
      });
    }
  }

  Future<void> _refresh() async {
    setState(() {
      _items.clear();
      _currentPage = 0;
      _hasMore = true;
      _error = null;
    });
    await _loadNextPage();
  }

  @override
  Widget build(BuildContext context) {
    if (_items.isEmpty && _isLoading) {
      return widget.loadingWidget ??
          const Center(child: CircularProgressIndicator());
    }

    if (_items.isEmpty && _error != null) {
      return widget.errorBuilder?.call(_error!, _refresh) ??
          Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                Text('Error: $_error'),
                ElevatedButton(onPressed: _refresh, child: const Text('Retry')),
              ],
            ),
          );
    }

    if (_items.isEmpty) {
      return widget.emptyWidget ?? const Center(child: Text('No items'));
    }

    return RefreshIndicator(
      onRefresh: _refresh,
      child: ListView.builder(
        controller: _scrollController,
        itemCount: _items.length + (_hasMore ? 1 : 0),
        itemBuilder: (context, index) {
          if (index == _items.length) {
            return _buildLoadingOrError();
          }
          return widget.itemBuilder(context, _items[index], index);
        },
      ),
    );
  }

  Widget _buildLoadingOrError() {
    if (_error != null) {
      return Center(
        child: TextButton(
          onPressed: _loadNextPage,
          child: const Text('Retry loading more'),
        ),
      );
    }
    return const Padding(
      padding: EdgeInsets.all(16),
      child: Center(child: CircularProgressIndicator()),
    );
  }
}

// Usage example
class ProductsPage extends StatelessWidget {
  const ProductsPage({super.key});

  Future<List<Map<String, dynamic>>> _fetchProducts(
      int page, int pageSize) async {
    await Future.delayed(const Duration(milliseconds: 500));
    return List.generate(
      pageSize,
      (i) => {
        'id': page * pageSize + i,
        'name': 'Product ${page * pageSize + i + 1}',
        'price': (page * pageSize + i + 1) * 9.99,
      },
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Products (Lazy Loading)')),
      body: LazyLoadingList<Map<String, dynamic>>(
        fetchPage: _fetchProducts,
        itemBuilder: (context, product, index) => ListTile(
          leading: CircleAvatar(child: Text('${product['id']}')),
          title: Text(product['name'] as String),
          subtitle: Text('\$${(product['price'] as double).toStringAsFixed(2)}'),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1885: compute() สำหรับ Heavy Operations

```dart
// lib/services/compute_service.dart
import 'dart:convert';
import 'dart:isolate';
import 'package:flutter/foundation.dart';

// ─── Simple compute() examples ────────────────────────────────────

/// Parse large JSON list in background isolate
Future<List<Map<String, dynamic>>> parseJsonInBackground(
    String jsonString) async {
  return compute(_parseJson, jsonString);
}

List<Map<String, dynamic>> _parseJson(String jsonString) {
  final List<dynamic> data = json.decode(jsonString);
  return data.cast<Map<String, dynamic>>();
}

/// Sort large list in background
Future<List<T>> sortInBackground<T extends Comparable<T>>(
    List<T> items) async {
  return compute(_sortList, items);
}

List<T> _sortList<T extends Comparable<T>>(List<T> items) {
  final sorted = List<T>.from(items);
  sorted.sort();
  return sorted;
}

/// Heavy computation: find primes up to n
Future<List<int>> findPrimesInBackground(int limit) async {
  return compute(_sieveOfEratosthenes, limit);
}

List<int> _sieveOfEratosthenes(int limit) {
  if (limit < 2) return [];
  final sieve = List<bool>.filled(limit + 1, true);
  sieve[0] = false;
  sieve[1] = false;

  for (int i = 2; i * i <= limit; i++) {
    if (sieve[i]) {
      for (int j = i * i; j <= limit; j += i) {
        sieve[j] = false;
      }
    }
  }

  return [for (int i = 2; i <= limit; i++) if (sieve[i]) i];
}

// ─── Advanced: Custom Isolate with bidirectional communication ─────

class _ProcessingMessage {
  final List<int> data;
  final SendPort sendPort;

  _ProcessingMessage({required this.data, required this.sendPort});
}

/// Process data with progress reporting
Future<Map<String, dynamic>> processDataWithProgress({
  required List<int> data,
  required void Function(double progress) onProgress,
}) async {
  final receivePort = ReceivePort();

  await Isolate.spawn(
    _processDataIsolate,
    _ProcessingMessage(data: data, sendPort: receivePort.sendPort),
  );

  Map<String, dynamic>? result;

  await for (final message in receivePort) {
    if (message is double) {
      onProgress(message); // Progress update
    } else if (message is Map<String, dynamic>) {
      result = message;
      break;
    }
  }

  receivePort.close();
  return result ?? {};
}

void _processDataIsolate(_ProcessingMessage message) {
  final data = message.data;
  final sendPort = message.sendPort;

  int sum = 0;
  int min = data.isEmpty ? 0 : data[0];
  int max = data.isEmpty ? 0 : data[0];

  for (int i = 0; i < data.length; i++) {
    sum += data[i];
    if (data[i] < min) min = data[i];
    if (data[i] > max) max = data[i];

    // Report progress every 10%
    if (i % (data.length ~/ 10) == 0) {
      sendPort.send(i / data.length);
    }
  }

  sendPort.send({
    'sum': sum,
    'min': min,
    'max': max,
    'avg': data.isEmpty ? 0 : sum / data.length,
    'count': data.length,
  });
}

// Example screen using compute()
class HeavyComputationScreen extends StatefulWidget {
  const HeavyComputationScreen({super.key});

  @override
  State<HeavyComputationScreen> createState() => _HeavyComputationScreenState();
}

class _HeavyComputationScreenState extends State<HeavyComputationScreen> {
  List<int>? _primes;
  double _progress = 0;
  bool _isProcessing = false;
  Map<String, dynamic>? _stats;

  Future<void> _findPrimes() async {
    setState(() => _isProcessing = true);
    final stopwatch = Stopwatch()..start();
    final primes = await findPrimesInBackground(100000);
    stopwatch.stop();
    setState(() {
      _primes = primes;
      _isProcessing = false;
    });
    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(
        content: Text(
          'Found ${primes.length} primes in ${stopwatch.elapsedMilliseconds}ms',
        ),
      ));
    }
  }

  Future<void> _processLargeData() async {
    setState(() {
      _isProcessing = true;
      _progress = 0;
    });

    final data = List.generate(1000000, (i) => i);
    final result = await processDataWithProgress(
      data: data,
      onProgress: (p) => setState(() => _progress = p),
    );

    setState(() {
      _stats = result;
      _isProcessing = false;
      _progress = 1.0;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Heavy Computation')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            ElevatedButton(
              onPressed: _isProcessing ? null : _findPrimes,
              child: const Text('Find Primes (0-100,000)'),
            ),
            if (_primes != null)
              Text('Found ${_primes!.length} primes. Last: ${_primes!.last}'),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: _isProcessing ? null : _processLargeData,
              child: const Text('Process 1M numbers with progress'),
            ),
            if (_isProcessing) ...[
              const SizedBox(height: 8),
              LinearProgressIndicator(value: _progress),
              Text('${(_progress * 100).toStringAsFixed(0)}%'),
            ],
            if (_stats != null) ...[
              const SizedBox(height: 8),
              Card(
                child: Padding(
                  padding: const EdgeInsets.all(8),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text('Count: ${_stats!['count']}'),
                      Text('Sum: ${_stats!['sum']}'),
                      Text('Min: ${_stats!['min']}'),
                      Text('Max: ${_stats!['max']}'),
                      Text('Avg: ${(_stats!['avg'] as double).toStringAsFixed(2)}'),
                    ],
                  ),
                ),
              ),
            ],
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1886: SliverList Optimization

```dart
// lib/widgets/optimized_sliver_list.dart
import 'package:flutter/material.dart';

class OptimizedSliverPage extends StatefulWidget {
  const OptimizedSliverPage({super.key});

  @override
  State<OptimizedSliverPage> createState() => _OptimizedSliverPageState();
}

class _OptimizedSliverPageState extends State<OptimizedSliverPage> {
  // Large dataset
  late final List<Map<String, dynamic>> _items;

  @override
  void initState() {
    super.initState();
    _items = List.generate(
      10000,
      (i) => {
        'id': i,
        'name': 'Item ${i + 1}',
        'subtitle': 'Description for item ${i + 1}',
        'value': (i + 1) * 1.5,
      },
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: CustomScrollView(
        // Key for better performance with large lists
        cacheExtent: 500,
        slivers: [
          // Collapsible app bar
          SliverAppBar(
            expandedHeight: 200,
            pinned: true,
            flexibleSpace: FlexibleSpaceBar(
              title: const Text('Optimized List'),
              background: Container(
                decoration: BoxDecoration(
                  gradient: LinearGradient(
                    colors: [Colors.blue.shade800, Colors.blue.shade400],
                    begin: Alignment.topLeft,
                    end: Alignment.bottomRight,
                  ),
                ),
                child: const Center(
                  child: Icon(Icons.list, size: 80, color: Colors.white54),
                ),
              ),
            ),
          ),

          // Stats header
          SliverToBoxAdapter(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Row(
                mainAxisAlignment: MainAxisAlignment.spaceAround,
                children: [
                  _StatChip(label: 'Total', value: '${_items.length}'),
                  _StatChip(
                    label: 'Avg Value',
                    value:
                        (_items.fold(0.0, (s, i) => s + (i['value'] as double)) /
                                _items.length)
                            .toStringAsFixed(1),
                  ),
                ],
              ),
            ),
          ),

          // Section header
          const SliverPersistentHeader(
            pinned: true,
            delegate: _SectionHeaderDelegate(title: 'All Items'),
          ),

          // Highly optimized SliverList with builder
          // Uses lazy construction - only builds visible items
          SliverList.builder(
            itemCount: _items.length,
            itemBuilder: (context, index) {
              final item = _items[index];
              return _OptimizedListItem(
                key: ValueKey(item['id']),
                item: item,
                index: index,
              );
            },
          ),

          // Footer padding
          const SliverPadding(padding: EdgeInsets.only(bottom: 32)),
        ],
      ),
    );
  }
}

// RepaintBoundary prevents unnecessary repaints
class _OptimizedListItem extends StatelessWidget {
  final Map<String, dynamic> item;
  final int index;

  const _OptimizedListItem({super.key, required this.item, required this.index});

  @override
  Widget build(BuildContext context) {
    return RepaintBoundary(
      child: ListTile(
        leading: CircleAvatar(
          backgroundColor: index.isEven ? Colors.blue : Colors.green,
          child: Text('${index + 1}', style: const TextStyle(fontSize: 12)),
        ),
        title: Text(item['name'] as String),
        subtitle: Text(item['subtitle'] as String),
        trailing: Text(
          (item['value'] as double).toStringAsFixed(1),
          style: const TextStyle(fontWeight: FontWeight.bold),
        ),
      ),
    );
  }
}

class _StatChip extends StatelessWidget {
  final String label;
  final String value;

  const _StatChip({required this.label, required this.value});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text(value,
            style: const TextStyle(fontSize: 24, fontWeight: FontWeight.bold)),
        Text(label, style: TextStyle(color: Colors.grey[600])),
      ],
    );
  }
}

class _SectionHeaderDelegate extends SliverPersistentHeaderDelegate {
  final String title;
  const _SectionHeaderDelegate({required this.title});

  @override
  Widget build(
      BuildContext context, double shrinkOffset, bool overlapsContent) {
    return Container(
      color: Theme.of(context).scaffoldBackgroundColor,
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      child: Text(
        title,
        style: Theme.of(context).textTheme.titleSmall?.copyWith(
              color: Colors.grey[600],
              letterSpacing: 1.2,
            ),
      ),
    );
  }

  @override
  double get maxExtent => 40;

  @override
  double get minExtent => 40;

  @override
  bool shouldRebuild(_SectionHeaderDelegate oldDelegate) =>
      title != oldDelegate.title;
}

// SliverGrid for photo gallery
class PhotoGallery extends StatelessWidget {
  final List<String> imageUrls;

  const PhotoGallery({super.key, required this.imageUrls});

  @override
  Widget build(BuildContext context) {
    return CustomScrollView(
      slivers: [
        SliverPadding(
          padding: const EdgeInsets.all(8),
          sliver: SliverGrid(
            gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
              maxCrossAxisExtent: 150,
              mainAxisSpacing: 8,
              crossAxisSpacing: 8,
              childAspectRatio: 1,
            ),
            delegate: SliverChildBuilderDelegate(
              (context, index) {
                return RepaintBoundary(
                  child: OptimizedNetworkImage(
                    imageUrl: imageUrls[index],
                    width: 150,
                    height: 150,
                    borderRadius: BorderRadius.circular(8),
                  ),
                );
              },
              childCount: imageUrls.length,
            ),
          ),
        ),
      ],
    );
  }
}

// Placeholder - will be imported from optimized_image.dart in real usage
class OptimizedNetworkImage extends StatelessWidget {
  final String imageUrl;
  final double? width;
  final double? height;
  final BoxFit fit;
  final BorderRadius? borderRadius;

  const OptimizedNetworkImage({
    super.key,
    required this.imageUrl,
    this.width,
    this.height,
    this.fit = BoxFit.cover,
    this.borderRadius,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      width: width,
      height: height,
      decoration: BoxDecoration(
        color: Colors.grey[200],
        borderRadius: borderRadius,
      ),
      child: Icon(Icons.image, color: Colors.grey[400]),
    );
  }
}
```

---

## ขั้นตอนที่ 1887: Build Mode Differences

```dart
// lib/core/build_config.dart

/// Utility class to check build mode and configure accordingly
class BuildConfig {
  BuildConfig._();

  /// Returns true in debug mode (flutter run without --release)
  static bool get isDebug => kDebugMode;

  /// Returns true in profile mode (flutter run --profile)
  static bool get isProfile => kProfileMode;

  /// Returns true in release mode (flutter build apk)
  static bool get isRelease => kReleaseMode;

  /// Base URL based on build mode
  static String get apiBaseUrl {
    if (isDebug) return 'http://localhost:3000/api';
    if (isProfile) return 'https://staging.example.com/api';
    return 'https://api.example.com';
  }

  /// Log level based on build mode
  static void log(String message, {String tag = 'App'}) {
    if (isDebug) {
      debugPrint('[$tag] $message');
    }
    // In release mode, send to crash reporting instead
  }

  /// Feature flags
  static bool get enableAnalytics => isRelease;
  static bool get enableCrashReporting => !isDebug;
  static bool get showDebugBanner => isDebug;
  static bool get enablePerformanceOverlay => isProfile;
}

// In main.dart, use build-specific configurations
import 'package:flutter/material.dart';

void main() {
  // Disable debug assertions in profile/release
  if (!BuildConfig.isDebug) {
    // Configure error handling for production
    FlutterError.onError = (details) {
      // Send to crash reporting service
      BuildConfig.log('Flutter Error: ${details.exception}', tag: 'Error');
    };
  }

  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter App',
      debugShowCheckedModeBanner: BuildConfig.showDebugBanner,
      showPerformanceOverlay: BuildConfig.enablePerformanceOverlay,
      home: const Placeholder(),
    );
  }
}
```

---

## ขั้นตอนที่ 1888: App Size Reduction Techniques

```bash
# Build with tree shaking (removes unused code/assets)
flutter build apk --release --tree-shake-icons

# Analyze app size
flutter build apk --analyze-size
flutter build appbundle --analyze-size

# Split APK by ABI (reduces download size ~50%)
flutter build apk --release --split-per-abi

# Output:
# build/app/outputs/flutter-apk/app-armeabi-v7a-release.apk
# build/app/outputs/flutter-apk/app-arm64-v8a-release.apk
# build/app/outputs/flutter-apk/app-x86_64-release.apk

# Obfuscate Dart code
flutter build apk --release \
  --obfuscate \
  --split-debug-info=build/debug-info/

# Build IPA with bitcode disabled (smaller upload)
flutter build ipa --release --no-codesign
```

```yaml
# android/app/build.gradle - Enable R8 full mode
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            // Full R8 optimization
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'),
                         'proguard-rules.pro'
        }
    }
}
```

```
# android/app/proguard-rules.pro
# Flutter-specific rules
-keep class io.flutter.** { *; }
-keep class io.flutter.embedding.** { *; }

# Dart VM rules
-dontwarn io.flutter.**

# Keep model classes (adjust to your package)
-keep class com.example.myapp.models.** { *; }

# Keep Gson model classes
-keepattributes Signature
-keepattributes *Annotation*
```

---

## ขั้นตอนที่ 1889: Const Constructors and Widget Caching

```dart
// lib/widgets/performance_patterns.dart
import 'package:flutter/material.dart';

// BAD: Creates new widget every build
class BadWidget extends StatelessWidget {
  const BadWidget({super.key});

  @override
  Widget build(BuildContext context) {
    // These are recreated every time parent rebuilds
    final icon = Icon(Icons.star, color: Colors.yellow[700]);
    final text = Text('Rating', style: TextStyle(color: Colors.grey[600]));
    return Row(children: [icon, const SizedBox(width: 4), text]);
  }
}

// GOOD: Use const constructors to reuse widget instances
class GoodWidget extends StatelessWidget {
  const GoodWidget({super.key});

  @override
  Widget build(BuildContext context) {
    return const Row(
      children: [
        Icon(Icons.star, color: Colors.amber),
        SizedBox(width: 4),
        Text('Rating'),
      ],
    );
  }
}

// GOOD: Extract static child to avoid rebuilds
class ParentWidget extends StatefulWidget {
  const ParentWidget({super.key});

  @override
  State<ParentWidget> createState() => _ParentWidgetState();
}

class _ParentWidgetState extends State<ParentWidget> {
  int _counter = 0;

  // Static child never rebuilds regardless of parent state changes
  static const _staticChild = ExpensiveStaticWidget();

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Count: $_counter'),
        ElevatedButton(
          onPressed: () => setState(() => _counter++),
          child: const Text('Increment'),
        ),
        _staticChild, // ← never rebuilds
      ],
    );
  }
}

class ExpensiveStaticWidget extends StatelessWidget {
  const ExpensiveStaticWidget({super.key});

  @override
  Widget build(BuildContext context) {
    // Expensive rendering but only called once
    return Container(
      height: 100,
      color: Colors.blue[50],
      child: const Center(child: Text('I never rebuild!')),
    );
  }
}

// GOOD: Use RepaintBoundary for complex isolated widgets
class AnimatedCounter extends StatelessWidget {
  final int value;
  const AnimatedCounter({super.key, required this.value});

  @override
  Widget build(BuildContext context) {
    return RepaintBoundary(
      // Isolates this subtree from parent repaints
      child: AnimatedSwitcher(
        duration: const Duration(milliseconds: 300),
        transitionBuilder: (child, animation) => ScaleTransition(
          scale: animation,
          child: child,
        ),
        child: Text(
          '$value',
          key: ValueKey(value),
          style: const TextStyle(fontSize: 48, fontWeight: FontWeight.bold),
        ),
      ),
    );
  }
}

// GOOD: IndexedStack for tab-based navigation (preserves state)
class TabsWithPreservation extends StatefulWidget {
  const TabsWithPreservation({super.key});

  @override
  State<TabsWithPreservation> createState() => _TabsWithPreservationState();
}

class _TabsWithPreservationState extends State<TabsWithPreservation> {
  int _currentIndex = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: IndexedStack(
        index: _currentIndex,
        children: const [
          _Tab(label: 'Home', color: Colors.blue),
          _Tab(label: 'Search', color: Colors.green),
          _Tab(label: 'Profile', color: Colors.orange),
        ],
      ),
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _currentIndex,
        onTap: (i) => setState(() => _currentIndex = i),
        items: const [
          BottomNavigationBarItem(icon: Icon(Icons.home), label: 'Home'),
          BottomNavigationBarItem(icon: Icon(Icons.search), label: 'Search'),
          BottomNavigationBarItem(icon: Icon(Icons.person), label: 'Profile'),
        ],
      ),
    );
  }
}

class _Tab extends StatefulWidget {
  final String label;
  final Color color;
  const _Tab({required this.label, required this.color});

  @override
  State<_Tab> createState() => _TabState();
}

class _TabState extends State<_Tab> {
  int _localCounter = 0; // This state is PRESERVED with IndexedStack

  @override
  Widget build(BuildContext context) {
    return Container(
      color: widget.color.withOpacity(0.1),
      child: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(widget.label, style: const TextStyle(fontSize: 24)),
            Text('Tab counter: $_localCounter'),
            ElevatedButton(
              onPressed: () => setState(() => _localCounter++),
              child: const Text('Increment'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1890: Performance Profiling Tips

```dart
// lib/utils/performance_monitor.dart
import 'dart:developer' as developer;
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';

/// Enable performance profiling in debug mode
class PerformanceMonitor {
  static void enable() {
    if (!kDebugMode) return;

    // Show performance overlay
    debugPaintSizeEnabled = false;        // Show layout bounds
    debugPaintPointersEnabled = false;    // Show touch events
    debugRepaintRainbowEnabled = false;   // Flash on repaint
    debugPaintLayerBordersEnabled = false;// Show layer borders

    // Enable timeline events
    developer.Timeline.startSync('App Start');
  }

  /// Measure execution time of a function
  static Future<T> measure<T>(
    String operationName,
    Future<T> Function() operation,
  ) async {
    final stopwatch = Stopwatch()..start();
    developer.Timeline.startSync(operationName);

    try {
      final result = await operation();
      stopwatch.stop();
      developer.Timeline.finishSync();

      if (kDebugMode) {
        debugPrint(
          '⏱️ $operationName: ${stopwatch.elapsedMilliseconds}ms',
        );
      }

      return result;
    } catch (e) {
      stopwatch.stop();
      developer.Timeline.finishSync();
      rethrow;
    }
  }

  /// Synchronous version
  static T measureSync<T>(String name, T Function() operation) {
    final sw = Stopwatch()..start();
    developer.Timeline.startSync(name);
    try {
      final result = operation();
      sw.stop();
      developer.Timeline.finishSync();
      if (kDebugMode) debugPrint('⏱️ $name: ${sw.elapsedMilliseconds}ms');
      return result;
    } catch (e) {
      sw.stop();
      developer.Timeline.finishSync();
      rethrow;
    }
  }
}

// Usage
void exampleUsage() async {
  final result = await PerformanceMonitor.measure(
    'Load Products',
    () => Future.delayed(const Duration(milliseconds: 100), () => [1, 2, 3]),
  );
  debugPrint('Got ${result.length} items');
}
```

---

## ขั้นตอนที่ 1891: Memory Leak Prevention

```dart
// lib/widgets/lifecycle_aware_widget.dart
import 'package:flutter/material.dart';

class LifecycleAwareWidget extends StatefulWidget {
  const LifecycleAwareWidget({super.key});

  @override
  State<LifecycleAwareWidget> createState() => _LifecycleAwareWidgetState();
}

class _LifecycleAwareWidgetState extends State<LifecycleAwareWidget>
    with WidgetsBindingObserver {
  final List<StreamSubscription> _subscriptions = [];
  Timer? _refreshTimer;
  final ScrollController _scrollController = ScrollController();
  late AnimationController _animationController;

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);

    // Subscribe to streams - MUST cancel in dispose
    _subscriptions.add(
      Stream.periodic(const Duration(seconds: 5))
          .listen((_) => _onPeriodicEvent()),
    );

    // Timer - MUST cancel in dispose
    _refreshTimer = Timer.periodic(
      const Duration(minutes: 1),
      (_) => _refreshData(),
    );
  }

  @override
  void dispose() {
    // Cancel all subscriptions
    for (final sub in _subscriptions) {
      sub.cancel();
    }
    _subscriptions.clear();

    // Cancel timer
    _refreshTimer?.cancel();

    // Dispose controllers
    _scrollController.dispose();

    // Remove observer
    WidgetsBinding.instance.removeObserver(this);

    super.dispose();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    switch (state) {
      case AppLifecycleState.resumed:
        _refreshData();
      case AppLifecycleState.paused:
        _pauseOperations();
      case AppLifecycleState.detached:
        _cleanupResources();
      default:
        break;
    }
  }

  void _onPeriodicEvent() {
    if (!mounted) return; // Guard against disposed widget
    setState(() {/* update state */});
  }

  void _refreshData() {
    if (!mounted) return;
    // Fetch fresh data
  }

  void _pauseOperations() {
    // Pause battery-draining operations
  }

  void _cleanupResources() {
    // Release heavy resources
  }

  @override
  Widget build(BuildContext context) {
    return const Scaffold(
      body: Center(child: Text('Lifecycle-aware widget')),
    );
  }
}

// Missing import - add this to avoid compile errors in standalone file
import 'dart:async';
```

---

**← [Part 49 - CI/CD for Flutter](part-49-ci-cd-flutter.md)**

---

## 🎓 หลักสูตรสำเร็จแล้ว!

ยินดีด้วย! คุณได้เรียนจบหลักสูตร **Dart & Flutter Professional** ครบทั้ง 50 Parts แล้ว!

### สิ่งที่คุณเรียนรู้ใน Parts 46-50:
- **Part 46**: Advanced Testing - Unit, Widget, Integration, Golden Tests
- **Part 47**: Extensions & Mixins - ขยายความสามารถของ built-in types
- **Part 48**: Clean Architecture - Domain/Data/Presentation layers
- **Part 49**: CI/CD - Automated build, test, and deploy pipelines
- **Part 50**: Performance - Image optimization, lazy loading, isolates
