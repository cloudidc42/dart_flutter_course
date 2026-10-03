# Part 87: Flutter Hooks
## ขั้นตอนที่ 3361-3400

---

## 🎯 เป้าหมายของ Part นี้

- ติดตั้งและใช้งาน flutter_hooks package
- เข้าใจ useState, useEffect, useCallback, useMemoized
- สร้าง Custom hooks (useDebounce, useThrottle, usePrevious)
- ใช้ useAnimationController กับ animations
- แปลง StatefulWidgets เป็น Hooks widgets
- เขียนโค้ดที่สะอาดและ reusable ยิ่งขึ้น

---

## ขั้นตอนที่ 3361: ทำความเข้าใจ Flutter Hooks

Flutter Hooks นำแนวคิดจาก React Hooks มาใช้ใน Flutter ช่วยให้ลด boilerplate code และทำให้ stateful logic สามารถ reuse ได้

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_hooks: ^0.20.4
  hooks_riverpod: ^2.5.1
```

### ทำไมต้องใช้ Flutter Hooks?

```dart
// แบบดั้งเดิม: StatefulWidget ที่มี boilerplate มาก
class TraditionalCounterWidget extends StatefulWidget {
  const TraditionalCounterWidget({super.key});

  @override
  State<TraditionalCounterWidget> createState() =>
      _TraditionalCounterWidgetState();
}

class _TraditionalCounterWidgetState
    extends State<TraditionalCounterWidget> {
  int _count = 0;

  void _increment() {
    setState(() => _count++);
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('Count: $_count'),
        ElevatedButton(onPressed: _increment, child: const Text('+')),
      ],
    );
  }
}

// แบบ Hooks: สะอาดกว่ามาก
import 'package:flutter_hooks/flutter_hooks.dart';

class HookCounterWidget extends HookWidget {
  const HookCounterWidget({super.key});

  @override
  Widget build(BuildContext context) {
    final count = useState(0);

    return Column(
      children: [
        Text('Count: ${count.value}'),
        ElevatedButton(
          onPressed: () => count.value++,
          child: const Text('+'),
        ),
      ],
    );
  }
}
```

## ขั้นตอนที่ 3362: useState Hook

```dart
// lib/hooks/use_state_examples.dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';

// ตัวอย่าง 1: Counter
class UseStateCounterExample extends HookWidget {
  const UseStateCounterExample({super.key});

  @override
  Widget build(BuildContext context) {
    final count = useState(0);
    final text = useState('');
    final isLoading = useState(false);

    return Scaffold(
      appBar: AppBar(title: const Text('useState Example')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // Number state
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    Text(
                      'Count: ${count.value}',
                      style: Theme.of(context).textTheme.headlineMedium,
                    ),
                    Row(
                      children: [
                        IconButton(
                          icon: const Icon(Icons.remove),
                          onPressed: () => count.value--,
                        ),
                        IconButton(
                          icon: const Icon(Icons.add),
                          onPressed: () => count.value++,
                        ),
                      ],
                    ),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 16),

            // String state
            TextField(
              decoration: const InputDecoration(
                labelText: 'Enter text',
                border: OutlineInputBorder(),
              ),
              onChanged: (value) => text.value = value,
            ),
            const SizedBox(height: 8),
            Text('You typed: ${text.value}'),
            const SizedBox(height: 16),

            // Boolean state for loading
            ElevatedButton(
              onPressed: isLoading.value
                  ? null
                  : () async {
                      isLoading.value = true;
                      await Future.delayed(const Duration(seconds: 2));
                      isLoading.value = false;
                    },
              child: isLoading.value
                  ? const SizedBox(
                      width: 20,
                      height: 20,
                      child: CircularProgressIndicator(strokeWidth: 2),
                    )
                  : const Text('Simulate Loading'),
            ),
          ],
        ),
      ),
    );
  }
}

// ตัวอย่าง 2: List state
class UseStateListExample extends HookWidget {
  const UseStateListExample({super.key});

  @override
  Widget build(BuildContext context) {
    final items = useState<List<String>>([]);
    final inputController = useTextEditingController();

    void addItem() {
      if (inputController.text.isNotEmpty) {
        items.value = [...items.value, inputController.text];
        inputController.clear();
      }
    }

    void removeItem(int index) {
      final newList = List<String>.from(items.value);
      newList.removeAt(index);
      items.value = newList;
    }

    return Scaffold(
      appBar: AppBar(title: const Text('useState List Example')),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: Row(
              children: [
                Expanded(
                  child: TextField(
                    controller: inputController,
                    decoration: const InputDecoration(
                      labelText: 'Add item',
                      border: OutlineInputBorder(),
                    ),
                    onSubmitted: (_) => addItem(),
                  ),
                ),
                const SizedBox(width: 8),
                ElevatedButton(
                  onPressed: addItem,
                  child: const Text('Add'),
                ),
              ],
            ),
          ),
          Expanded(
            child: items.value.isEmpty
                ? const Center(child: Text('No items yet'))
                : ListView.builder(
                    itemCount: items.value.length,
                    itemBuilder: (context, index) {
                      return ListTile(
                        title: Text(items.value[index]),
                        trailing: IconButton(
                          icon: const Icon(Icons.delete),
                          onPressed: () => removeItem(index),
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
```

## ขั้นตอนที่ 3363: useEffect Hook

```dart
// lib/hooks/use_effect_examples.dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';
import 'package:http/http.dart' as http;
import 'dart:convert';

// ตัวอย่าง 1: Fetch data on mount
class UserProfileScreen extends HookWidget {
  final String userId;
  const UserProfileScreen({super.key, required this.userId});

  @override
  Widget build(BuildContext context) {
    final userData = useState<Map<String, dynamic>?>(null);
    final isLoading = useState(true);
    final error = useState<String?>(null);

    useEffect(() {
      bool cancelled = false;

      Future<void> fetchUser() async {
        try {
          final response = await http.get(
            Uri.parse('https://jsonplaceholder.typicode.com/users/$userId'),
          );
          if (!cancelled && response.statusCode == 200) {
            userData.value = jsonDecode(response.body) as Map<String, dynamic>;
          }
        } catch (e) {
          if (!cancelled) error.value = e.toString();
        } finally {
          if (!cancelled) isLoading.value = false;
        }
      }

      fetchUser();

      // Cleanup function: called when widget unmounts or userId changes
      return () {
        cancelled = true;
      };
    }, [userId]); // Re-run when userId changes

    if (isLoading.value) {
      return const Scaffold(
        body: Center(child: CircularProgressIndicator()),
      );
    }

    if (error.value != null) {
      return Scaffold(
        body: Center(child: Text('Error: ${error.value}')),
      );
    }

    final user = userData.value;
    return Scaffold(
      appBar: AppBar(title: Text(user?['name'] ?? 'User Profile')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            _InfoRow(label: 'Name', value: user?['name'] ?? ''),
            _InfoRow(label: 'Email', value: user?['email'] ?? ''),
            _InfoRow(label: 'Phone', value: user?['phone'] ?? ''),
            _InfoRow(label: 'Website', value: user?['website'] ?? ''),
          ],
        ),
      ),
    );
  }
}

class _InfoRow extends StatelessWidget {
  final String label;
  final String value;
  const _InfoRow({required this.label, required this.value});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 8),
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          SizedBox(
            width: 80,
            child: Text(
              '$label:',
              style: const TextStyle(fontWeight: FontWeight.bold),
            ),
          ),
          Expanded(child: Text(value)),
        ],
      ),
    );
  }
}

// ตัวอย่าง 2: Timer effect
class TimerWidget extends HookWidget {
  const TimerWidget({super.key});

  @override
  Widget build(BuildContext context) {
    final seconds = useState(0);
    final isRunning = useState(false);

    useEffect(() {
      if (!isRunning.value) return null;

      final timer = Stream.periodic(const Duration(seconds: 1))
          .listen((_) => seconds.value++);

      return timer.cancel; // Cleanup: cancel the timer
    }, [isRunning.value]);

    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Text(
          _formatTime(seconds.value),
          style: const TextStyle(fontSize: 48, fontWeight: FontWeight.bold),
        ),
        const SizedBox(height: 24),
        Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () => isRunning.value = !isRunning.value,
              child: Text(isRunning.value ? 'Pause' : 'Start'),
            ),
            const SizedBox(width: 16),
            OutlinedButton(
              onPressed: () {
                isRunning.value = false;
                seconds.value = 0;
              },
              child: const Text('Reset'),
            ),
          ],
        ),
      ],
    );
  }

  String _formatTime(int totalSeconds) {
    final minutes = totalSeconds ~/ 60;
    final secs = totalSeconds % 60;
    return '${minutes.toString().padLeft(2, '0')}:${secs.toString().padLeft(2, '0')}';
  }
}

// ตัวอย่าง 3: Event listener effect
class WindowSizeWidget extends HookWidget {
  const WindowSizeWidget({super.key});

  @override
  Widget build(BuildContext context) {
    final screenSize = useState(MediaQuery.of(context).size);

    useEffect(() {
      void updateSize() {
        screenSize.value = MediaQuery.of(context).size;
      }

      // Normally you'd add a real listener here
      // This is a conceptual example
      updateSize();

      return null; // No cleanup needed
    }, []);

    return Center(
      child: Text(
        'Screen: ${screenSize.value.width.toInt()} x ${screenSize.value.height.toInt()}',
        style: const TextStyle(fontSize: 20),
      ),
    );
  }
}
```

## ขั้นตอนที่ 3364: useCallback และ useMemoized

```dart
// lib/hooks/use_callback_memoized.dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';

// ตัวอย่าง useMemoized: expensive computation
class ExpensiveComputationWidget extends HookWidget {
  final List<int> numbers;
  const ExpensiveComputationWidget({super.key, required this.numbers});

  @override
  Widget build(BuildContext context) {
    final multiplier = useState(2);

    // useMemoized caches the result and only recomputes when dependencies change
    final result = useMemoized(
      () {
        debugPrint('Computing expensive result...');
        return numbers
            .map((n) => n * multiplier.value)
            .fold<int>(0, (sum, n) => sum + n);
      },
      [numbers, multiplier.value],
    );

    final average = useMemoized(
      () => numbers.isEmpty ? 0.0 : numbers.reduce((a, b) => a + b) / numbers.length,
      [numbers],
    );

    return Scaffold(
      appBar: AppBar(title: const Text('useMemoized Example')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Numbers: $numbers'),
            const SizedBox(height: 16),
            Text('Average: ${average.toStringAsFixed(2)}'),
            Text('Sum × ${multiplier.value}: $result'),
            const SizedBox(height: 16),
            Text('Multiplier: ${multiplier.value}'),
            Slider(
              min: 1,
              max: 10,
              divisions: 9,
              value: multiplier.value.toDouble(),
              onChanged: (v) => multiplier.value = v.toInt(),
            ),
          ],
        ),
      ),
    );
  }
}

// ตัวอย่าง useCallback: stable function reference
class SearchWidget extends HookWidget {
  final Function(String) onSearch;
  const SearchWidget({super.key, required this.onSearch});

  @override
  Widget build(BuildContext context) {
    final query = useState('');
    final controller = useTextEditingController();

    // useCallback returns a stable function reference
    // Only recreates when dependencies change
    final handleSearch = useCallback(
      () {
        if (query.value.isNotEmpty) {
          onSearch(query.value);
        }
      },
      [query.value, onSearch],
    );

    final handleClear = useCallback(
      () {
        query.value = '';
        controller.clear();
      },
      [controller],
    );

    return Column(
      children: [
        TextField(
          controller: controller,
          decoration: InputDecoration(
            hintText: 'Search...',
            prefixIcon: const Icon(Icons.search),
            suffixIcon: query.value.isNotEmpty
                ? IconButton(
                    icon: const Icon(Icons.clear),
                    onPressed: handleClear,
                  )
                : null,
            border: const OutlineInputBorder(),
          ),
          onChanged: (v) => query.value = v,
          onSubmitted: (_) => handleSearch(),
        ),
        const SizedBox(height: 8),
        ElevatedButton(
          onPressed: handleSearch,
          child: const Text('Search'),
        ),
      ],
    );
  }
}

// ตัวอย่าง useRef: persistent reference without rebuilds
class FocusTrackingWidget extends HookWidget {
  const FocusTrackingWidget({super.key});

  @override
  Widget build(BuildContext context) {
    final focusNode = useFocusNode();
    final isFocused = useState(false);
    final controller = useTextEditingController();
    // useRef holds value that persists across rebuilds but doesn't trigger rebuild
    final tapCount = useRef(0);

    useEffect(() {
      void listener() => isFocused.value = focusNode.hasFocus;
      focusNode.addListener(listener);
      return () => focusNode.removeListener(listener);
    }, [focusNode]);

    return GestureDetector(
      onTap: () => tapCount.value++,
      child: AnimatedContainer(
        duration: const Duration(milliseconds: 200),
        padding: const EdgeInsets.all(16),
        decoration: BoxDecoration(
          border: Border.all(
            color: isFocused.value ? Colors.blue : Colors.grey,
            width: isFocused.value ? 2 : 1,
          ),
          borderRadius: BorderRadius.circular(8),
        ),
        child: TextField(
          controller: controller,
          focusNode: focusNode,
          decoration: InputDecoration(
            hintText: isFocused.value ? 'Typing...' : 'Tap to focus',
            border: InputBorder.none,
          ),
        ),
      ),
    );
  }
}
```

## ขั้นตอนที่ 3365: Custom Hooks

```dart
// lib/hooks/custom_hooks.dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';

/// useDebounce: delays updating a value until after a delay
T useDebounce<T>(T value, Duration delay) {
  final debouncedValue = useState<T>(value);

  useEffect(() {
    final timer = Timer(delay, () {
      debouncedValue.value = value;
    });
    return timer.cancel;
  }, [value]);

  return debouncedValue.value;
}

/// useThrottle: limits how often a callback can be called
VoidCallback useThrottle(VoidCallback callback, Duration duration) {
  final lastCall = useRef<DateTime?>(null);
  final mounted = useIsMounted();

  return useCallback(() {
    final now = DateTime.now();
    final last = lastCall.value;

    if (last == null || now.difference(last) >= duration) {
      lastCall.value = now;
      if (mounted()) callback();
    }
  }, [callback, duration]);
}

/// usePrevious: returns the previous value of a state
T? usePrevious<T>(T value) {
  final prevRef = useRef<T?>(null);

  useEffect(() {
    // The effect runs AFTER render, so prevRef still has old value during render
    return null;
  }, [value]);

  final previousValue = prevRef.value;

  useEffect(() {
    prevRef.value = value;
    return null;
  });

  return previousValue;
}

/// useCounter: encapsulates counter logic
class CounterHookResult {
  final int count;
  final void Function() increment;
  final void Function() decrement;
  final void Function(int) set;
  final void Function() reset;

  const CounterHookResult({
    required this.count,
    required this.increment,
    required this.decrement,
    required this.set,
    required this.reset,
  });
}

CounterHookResult useCounter({
  int initialValue = 0,
  int? min,
  int? max,
}) {
  final count = useState(initialValue);

  void safeSet(int value) {
    if (min != null && value < min) return;
    if (max != null && value > max) return;
    count.value = value;
  }

  return CounterHookResult(
    count: count.value,
    increment: () => safeSet(count.value + 1),
    decrement: () => safeSet(count.value - 1),
    set: safeSet,
    reset: () => safeSet(initialValue),
  );
}

/// useLocalStorage: persists state to SharedPreferences
// Note: For a real implementation, you'd inject SharedPreferences
// This is a simplified in-memory version for demonstration
String useLocalStorage(String key, String defaultValue) {
  final value = useState(defaultValue);
  // In real usage, load from SharedPreferences in useEffect
  return value.value;
}

/// useAsync: handles async operations with loading/error state
class AsyncState<T> {
  final bool isLoading;
  final T? data;
  final String? error;

  const AsyncState({
    this.isLoading = false,
    this.data,
    this.error,
  });

  bool get hasData => data != null;
  bool get hasError => error != null;
}

AsyncState<T> useAsync<T>(Future<T> Function() asyncFn, List<Object?> keys) {
  final state = useState(const AsyncState<Object>());

  useEffect(() {
    bool cancelled = false;
    state.value = const AsyncState(isLoading: true);

    asyncFn().then((data) {
      if (!cancelled) {
        state.value = AsyncState(data: data, isLoading: false);
      }
    }).catchError((error) {
      if (!cancelled) {
        state.value = AsyncState(
          error: error.toString(),
          isLoading: false,
        );
      }
    });

    return () {
      cancelled = true;
    };
  }, keys);

  return AsyncState<T>(
    isLoading: state.value.isLoading,
    data: state.value.data as T?,
    error: state.value.error,
  );
}

// Demo Widget using all custom hooks
class CustomHooksDemo extends HookWidget {
  const CustomHooksDemo({super.key});

  @override
  Widget build(BuildContext context) {
    final searchText = useState('');
    final debouncedSearch = useDebounce(searchText.value, const Duration(milliseconds: 500));

    final counter = useCounter(initialValue: 0, min: 0, max: 10);
    final prevCount = usePrevious(counter.count);

    final heavyData = useAsync(
      () async {
        await Future.delayed(const Duration(seconds: 1));
        return List.generate(5, (i) => 'Item ${i + 1}');
      },
      [],
    );

    return Scaffold(
      appBar: AppBar(title: const Text('Custom Hooks Demo')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          // Debounce
          Card(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const Text('useDebounce', style: TextStyle(fontWeight: FontWeight.bold)),
                  TextField(
                    decoration: const InputDecoration(hintText: 'Type to search...'),
                    onChanged: (v) => searchText.value = v,
                  ),
                  Text('Live: ${searchText.value}'),
                  Text('Debounced (500ms): $debouncedSearch'),
                ],
              ),
            ),
          ),
          const SizedBox(height: 16),

          // Counter with previous
          Card(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const Text('useCounter + usePrevious', style: TextStyle(fontWeight: FontWeight.bold)),
                  Text('Current: ${counter.count}'),
                  Text('Previous: ${prevCount ?? "none"}'),
                  Text('Range: 0 - 10'),
                  Row(
                    children: [
                      IconButton(icon: const Icon(Icons.remove), onPressed: counter.decrement),
                      IconButton(icon: const Icon(Icons.add), onPressed: counter.increment),
                      TextButton(onPressed: counter.reset, child: const Text('Reset')),
                    ],
                  ),
                ],
              ),
            ),
          ),
          const SizedBox(height: 16),

          // Async data
          Card(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const Text('useAsync', style: TextStyle(fontWeight: FontWeight.bold)),
                  if (heavyData.isLoading)
                    const CircularProgressIndicator()
                  else if (heavyData.hasError)
                    Text('Error: ${heavyData.error}', style: const TextStyle(color: Colors.red))
                  else
                    ...?heavyData.data?.map((item) => Text('• $item')),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

## ขั้นตอนที่ 3366: useAnimationController

```dart
// lib/hooks/animation_hooks.dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';

// ตัวอย่าง 1: Basic animation with useAnimationController
class FadeInWidget extends HookWidget {
  final Widget child;
  final Duration duration;

  const FadeInWidget({
    super.key,
    required this.child,
    this.duration = const Duration(milliseconds: 600),
  });

  @override
  Widget build(BuildContext context) {
    final controller = useAnimationController(
      duration: duration,
      initialValue: 0,
    );

    final fadeAnimation = useMemoized(
      () => CurvedAnimation(parent: controller, curve: Curves.easeIn),
      [controller],
    );

    useEffect(() {
      controller.forward();
      return null;
    }, []);

    return FadeTransition(
      opacity: fadeAnimation,
      child: child,
    );
  }
}

// ตัวอย่าง 2: Pulse animation
class PulseButton extends HookWidget {
  final String label;
  final VoidCallback onPressed;

  const PulseButton({
    super.key,
    required this.label,
    required this.onPressed,
  });

  @override
  Widget build(BuildContext context) {
    final controller = useAnimationController(
      duration: const Duration(milliseconds: 800),
    );

    final scaleAnimation = useMemoized(
      () => Tween<double>(begin: 1.0, end: 1.08).animate(
        CurvedAnimation(parent: controller, curve: Curves.easeInOut),
      ),
      [controller],
    );

    useEffect(() {
      controller.repeat(reverse: true);
      return controller.stop;
    }, []);

    return AnimatedBuilder(
      animation: scaleAnimation,
      builder: (context, child) {
        return Transform.scale(
          scale: scaleAnimation.value,
          child: ElevatedButton(
            onPressed: onPressed,
            child: Text(label),
          ),
        );
      },
    );
  }
}

// ตัวอย่าง 3: Slide transition
class SlideInScreen extends HookWidget {
  const SlideInScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final controller = useAnimationController(
      duration: const Duration(milliseconds: 400),
    );

    final slideAnimation = useMemoized(
      () => Tween<Offset>(
        begin: const Offset(1, 0),
        end: Offset.zero,
      ).animate(CurvedAnimation(
        parent: controller,
        curve: Curves.easeOutCubic,
      )),
      [controller],
    );

    useEffect(() {
      controller.forward();
      return null;
    }, []);

    return Scaffold(
      appBar: AppBar(title: const Text('Slide In')),
      body: SlideTransition(
        position: slideAnimation,
        child: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(Icons.check_circle, size: 80, color: Colors.green),
              const SizedBox(height: 16),
              const Text(
                'Action Completed!',
                style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
              ),
              const SizedBox(height: 8),
              OutlinedButton(
                onPressed: () => controller.reverse().then(
                      (_) => Navigator.of(context).pop(),
                    ),
                child: const Text('Go Back'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

// ตัวอย่าง 4: Staggered animations
class StaggeredListWidget extends HookWidget {
  final List<String> items;

  const StaggeredListWidget({super.key, required this.items});

  @override
  Widget build(BuildContext context) {
    final controller = useAnimationController(
      duration: Duration(milliseconds: 300 * items.length),
    );

    useEffect(() {
      controller.forward();
      return null;
    }, []);

    return ListView.builder(
      itemCount: items.length,
      itemBuilder: (context, index) {
        final start = index / items.length;
        final end = (index + 1) / items.length;

        final itemAnimation = CurvedAnimation(
          parent: controller,
          curve: Interval(start, end, curve: Curves.easeOut),
        );

        return AnimatedBuilder(
          animation: itemAnimation,
          builder: (context, child) {
            return Transform.translate(
              offset: Offset(0, 50 * (1 - itemAnimation.value)),
              child: Opacity(
                opacity: itemAnimation.value,
                child: child,
              ),
            );
          },
          child: ListTile(
            leading: CircleAvatar(child: Text('${index + 1}')),
            title: Text(items[index]),
          ),
        );
      },
    );
  }
}
```

## ขั้นตอนที่ 3367: Converting StatefulWidgets to Hooks

```dart
// lib/hooks/conversion_examples.dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';

// BEFORE: StatefulWidget TabBarView
class OldTabScreen extends StatefulWidget {
  const OldTabScreen({super.key});

  @override
  State<OldTabScreen> createState() => _OldTabScreenState();
}

class _OldTabScreenState extends State<OldTabScreen>
    with SingleTickerProviderStateMixin {
  late TabController _tabController;
  int _selectedIndex = 0;

  @override
  void initState() {
    super.initState();
    _tabController = TabController(length: 3, vsync: this);
    _tabController.addListener(() {
      setState(() {
        _selectedIndex = _tabController.index;
      });
    });
  }

  @override
  void dispose() {
    _tabController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Tab $_selectedIndex'),
        bottom: TabBar(
          controller: _tabController,
          tabs: const [
            Tab(text: 'Tab 1'),
            Tab(text: 'Tab 2'),
            Tab(text: 'Tab 3'),
          ],
        ),
      ),
      body: TabBarView(
        controller: _tabController,
        children: const [
          Center(child: Text('Content 1')),
          Center(child: Text('Content 2')),
          Center(child: Text('Content 3')),
        ],
      ),
    );
  }
}

// AFTER: Hooks version — much cleaner
class NewTabScreen extends HookWidget {
  const NewTabScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final tabController = useTabController(initialLength: 3);
    final selectedIndex = useState(0);

    useEffect(() {
      void listener() => selectedIndex.value = tabController.index;
      tabController.addListener(listener);
      return () => tabController.removeListener(listener);
    }, [tabController]);

    return Scaffold(
      appBar: AppBar(
        title: Text('Tab ${selectedIndex.value}'),
        bottom: TabBar(
          controller: tabController,
          tabs: const [
            Tab(text: 'Tab 1'),
            Tab(text: 'Tab 2'),
            Tab(text: 'Tab 3'),
          ],
        ),
      ),
      body: TabBarView(
        controller: tabController,
        children: const [
          Center(child: Text('Content 1')),
          Center(child: Text('Content 2')),
          Center(child: Text('Content 3')),
        ],
      ),
    );
  }
}

// BEFORE: Form with validation StatefulWidget
class OldLoginForm extends StatefulWidget {
  const OldLoginForm({super.key});

  @override
  State<OldLoginForm> createState() => _OldLoginFormState();
}

class _OldLoginFormState extends State<OldLoginForm> {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  bool _isLoading = false;
  bool _obscurePassword = true;

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    if (!_formKey.currentState!.validate()) return;
    setState(() => _isLoading = true);
    await Future.delayed(const Duration(seconds: 2));
    setState(() => _isLoading = false);
  }

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      child: Column(
        children: [
          TextFormField(
            controller: _emailController,
            decoration: const InputDecoration(labelText: 'Email'),
            validator: (v) => v!.contains('@') ? null : 'Invalid email',
          ),
          TextFormField(
            controller: _passwordController,
            obscureText: _obscurePassword,
            decoration: InputDecoration(
              labelText: 'Password',
              suffixIcon: IconButton(
                icon: Icon(_obscurePassword ? Icons.visibility : Icons.visibility_off),
                onPressed: () => setState(() => _obscurePassword = !_obscurePassword),
              ),
            ),
            validator: (v) => v!.length >= 6 ? null : 'Too short',
          ),
          ElevatedButton(
            onPressed: _isLoading ? null : _submit,
            child: _isLoading ? const CircularProgressIndicator() : const Text('Login'),
          ),
        ],
      ),
    );
  }
}

// AFTER: Hooks version
class NewLoginForm extends HookWidget {
  const NewLoginForm({super.key});

  @override
  Widget build(BuildContext context) {
    final formKey = useMemoized(GlobalKey<FormState>.new);
    final emailController = useTextEditingController();
    final passwordController = useTextEditingController();
    final isLoading = useState(false);
    final obscurePassword = useState(true);

    Future<void> submit() async {
      if (!formKey.currentState!.validate()) return;
      isLoading.value = true;
      await Future.delayed(const Duration(seconds: 2));
      isLoading.value = false;
    }

    return Form(
      key: formKey,
      child: Column(
        children: [
          TextFormField(
            controller: emailController,
            decoration: const InputDecoration(
              labelText: 'Email',
              border: OutlineInputBorder(),
            ),
            validator: (v) => (v ?? '').contains('@') ? null : 'Invalid email',
          ),
          const SizedBox(height: 16),
          TextFormField(
            controller: passwordController,
            obscureText: obscurePassword.value,
            decoration: InputDecoration(
              labelText: 'Password',
              border: const OutlineInputBorder(),
              suffixIcon: IconButton(
                icon: Icon(
                  obscurePassword.value ? Icons.visibility : Icons.visibility_off,
                ),
                onPressed: () => obscurePassword.value = !obscurePassword.value,
              ),
            ),
            validator: (v) => (v ?? '').length >= 6 ? null : 'Too short',
          ),
          const SizedBox(height: 24),
          SizedBox(
            width: double.infinity,
            child: ElevatedButton(
              onPressed: isLoading.value ? null : submit,
              child: isLoading.value
                  ? const SizedBox(
                      width: 20,
                      height: 20,
                      child: CircularProgressIndicator(strokeWidth: 2),
                    )
                  : const Text('Login'),
            ),
          ),
        ],
      ),
    );
  }
}
```

## ขั้นตอนที่ 3368: Complete Hooks Demo App

```dart
// lib/main_hooks.dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';

void main() {
  runApp(const HooksDemoApp());
}

class HooksDemoApp extends StatelessWidget {
  const HooksDemoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Hooks Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const HooksDemoHome(),
    );
  }
}

class HooksDemoHome extends HookWidget {
  const HooksDemoHome({super.key});

  @override
  Widget build(BuildContext context) {
    final selectedDemo = useState(0);

    final demos = [
      const UseStateCounterExample(),
      const UseStateListExample(),
      const TimerWidget(),
      const CustomHooksDemo(),
      const NewTabScreen(),
    ];

    final demoNames = [
      'useState Counter',
      'useState List',
      'useEffect Timer',
      'Custom Hooks',
      'useTabController',
    ];

    return Scaffold(
      appBar: AppBar(
        title: const Text('Flutter Hooks Demo'),
        backgroundColor: Colors.deepPurple,
        foregroundColor: Colors.white,
      ),
      drawer: Drawer(
        child: ListView(
          children: [
            const DrawerHeader(
              decoration: BoxDecoration(color: Colors.deepPurple),
              child: Text(
                'Hooks Examples',
                style: TextStyle(color: Colors.white, fontSize: 20),
              ),
            ),
            ...List.generate(
              demoNames.length,
              (i) => ListTile(
                title: Text(demoNames[i]),
                selected: selectedDemo.value == i,
                selectedColor: Colors.deepPurple,
                onTap: () {
                  selectedDemo.value = i;
                  Navigator.of(context).pop();
                },
              ),
            ),
          ],
        ),
      ),
      body: IndexedStack(
        index: selectedDemo.value,
        children: demos,
      ),
    );
  }
}
```

---

**← [Part 86](part-86-enterprise-patterns.md)**
**ต่อไป: [Part 88 →](part-88-advanced-database.md)**
