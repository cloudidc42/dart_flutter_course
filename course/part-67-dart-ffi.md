# Part 67: Dart FFI (Foreign Function Interface)
## ขั้นตอนที่ 2561-2600

## 🎯 เป้าหมายของ Part นี้
- เรียนรู้ Dart FFI พื้นฐานในการเรียก C functions
- โหลด native libraries ด้วย DynamicLibrary
- ส่ง structs และ pointers ระหว่าง Dart กับ C
- รับ callbacks จาก native code กลับมา Dart
- ใช้ ffigen สร้าง bindings อัตโนมัติ

---

## ขั้นตอนที่ 2561: Dart FFI พื้นฐาน

```dart
// pubspec.yaml
// dependencies:
//   ffi: ^2.1.0
//
// dev_dependencies:
//   ffigen: ^10.0.0

// lib/ffi/ffi_basics.dart
import 'dart:ffi';
import 'dart:io';
import 'package:ffi/ffi.dart';

/// กำหนด C function signatures
/// int add(int a, int b)
typedef AddC = Int32 Function(Int32 a, Int32 b);
typedef AddDart = int Function(int a, int b);

/// double sqrt(double x)
typedef SqrtC = Double Function(Double x);
typedef SqrtDart = double Function(double x);

/// void* malloc(size_t size) - ตัวอย่าง pointer
typedef MallocC = Pointer<Void> Function(IntPtr size);
typedef MallocDart = Pointer<Void> Function(int size);

class FfiBasics {
  late final DynamicLibrary _lib;
  late final AddDart _add;
  late final SqrtDart _sqrt;

  void loadLibrary() {
    // โหลด math library จาก OS
    if (Platform.isLinux || Platform.isAndroid) {
      _lib = DynamicLibrary.open('libm.so.6');
    } else if (Platform.isMacOS || Platform.isIOS) {
      _lib = DynamicLibrary.process();
    } else if (Platform.isWindows) {
      _lib = DynamicLibrary.open('msvcrt.dll');
    } else {
      throw UnsupportedError('Unsupported platform');
    }

    // Lookup functions
    _sqrt = _lib
        .lookup<NativeFunction<SqrtC>>('sqrt')
        .asFunction<SqrtDart>();
  }

  double sqrt(double x) => _sqrt(x);
}

/// ตัวอย่าง C library ที่เราเขียนเอง
/// บันทึกเป็น mylib.c แล้ว compile:
/// gcc -shared -fPIC -o libmylib.so mylib.c   (Linux)
/// gcc -shared -o mylib.dll mylib.c           (Windows)

// mylib.c content:
// #include <stdio.h>
// #include <stdlib.h>
// #include <string.h>
//
// int add(int a, int b) {
//     return a + b;
// }
//
// double multiply(double a, double b) {
//     return a * b;
// }
//
// char* createString(const char* input) {
//     char* result = (char*)malloc(strlen(input) + 1);
//     strcpy(result, input);
//     return result;
// }
//
// void freeString(char* ptr) {
//     free(ptr);
// }
//
// int sumArray(int* arr, int length) {
//     int sum = 0;
//     for (int i = 0; i < length; i++) {
//         sum += arr[i];
//     }
//     return sum;
// }

class MyNativeLib {
  late final DynamicLibrary _lib;

  // Function signatures
  late final int Function(int, int) add;
  late final double Function(double, double) multiply;
  late final Pointer<Utf8> Function(Pointer<Utf8>) createString;
  late final void Function(Pointer<Utf8>) freeString;
  late final int Function(Pointer<Int32>, int) sumArray;

  void initialize() {
    final libPath = _getLibPath();
    _lib = DynamicLibrary.open(libPath);

    add = _lib
        .lookup<NativeFunction<Int32 Function(Int32, Int32)>>('add')
        .asFunction();

    multiply = _lib
        .lookup<NativeFunction<Double Function(Double, Double)>>('multiply')
        .asFunction();

    createString = _lib
        .lookup<NativeFunction<Pointer<Utf8> Function(Pointer<Utf8>)>>(
            'createString')
        .asFunction();

    freeString = _lib
        .lookup<NativeFunction<Void Function(Pointer<Utf8>)>>('freeString')
        .asFunction();

    sumArray = _lib
        .lookup<NativeFunction<Int32 Function(Pointer<Int32>, Int32)>>(
            'sumArray')
        .asFunction();
  }

  String _getLibPath() {
    if (Platform.isLinux) return './libmylib.so';
    if (Platform.isMacOS) return './libmylib.dylib';
    if (Platform.isWindows) return './mylib.dll';
    if (Platform.isAndroid) return 'libmylib.so';
    throw UnsupportedError('Unsupported platform: ${Platform.operatingSystem}');
  }
}
```

---

## ขั้นตอนที่ 2562: การทำงานกับ Strings และ Memory

```dart
// lib/ffi/string_operations.dart
import 'dart:ffi';
import 'package:ffi/ffi.dart';

class FfiStringHelper {
  /// แปลง Dart String เป็น C String (null-terminated)
  static Pointer<Utf8> toCString(String dartString) {
    return dartString.toNativeUtf8();
  }

  /// แปลง C String กลับเป็น Dart String
  static String fromCString(Pointer<Utf8> cString) {
    return cString.toDartString();
  }

  /// ตัวอย่างการใช้งาน string กับ FFI
  static void demonstrateStringUsage(DynamicLibrary lib) {
    // สร้าง function pointer
    final greet = lib
        .lookup<NativeFunction<Pointer<Utf8> Function(Pointer<Utf8>)>>('greet')
        .asFunction<Pointer<Utf8> Function(Pointer<Utf8>)>();

    // สร้าง C string
    final namePtr = 'World'.toNativeUtf8();

    try {
      // เรียก C function
      final resultPtr = greet(namePtr);
      final result = resultPtr.toDartString();
      print('Result: $result');

      // Free memory (ถ้า C code allocate memory เอง)
      // malloc.free(resultPtr);
    } finally {
      // Free the input string
      malloc.free(namePtr);
    }
  }

  /// จัดการ UTF-16 strings (Windows)
  static Pointer<Uint16> toUtf16(String dartString) {
    final units = dartString.codeUnits;
    final result = malloc<Uint16>(units.length + 1);
    for (var i = 0; i < units.length; i++) {
      result[i] = units[i];
    }
    result[units.length] = 0; // null terminator
    return result;
  }

  static String fromUtf16(Pointer<Uint16> ptr) {
    final buffer = StringBuffer();
    var i = 0;
    while (ptr[i] != 0) {
      buffer.writeCharCode(ptr[i]);
      i++;
    }
    return buffer.toString();
  }
}
```

---

## ขั้นตอนที่ 2563: Structs ใน Dart FFI

```dart
// lib/ffi/structs.dart
import 'dart:ffi';
import 'package:ffi/ffi.dart';

/// ตรงกับ C struct:
/// typedef struct {
///   int x;
///   int y;
/// } Point;
final class Point extends Struct {
  @Int32()
  external int x;

  @Int32()
  external int y;

  @override
  String toString() => 'Point($x, $y)';
}

/// ตรงกับ C struct:
/// typedef struct {
///   double real;
///   double imaginary;
/// } Complex;
final class Complex extends Struct {
  @Double()
  external double real;

  @Double()
  external double imaginary;

  double get magnitude =>
      (real * real + imaginary * imaginary).toDouble();

  @override
  String toString() => '$real + ${imaginary}i';
}

/// Nested struct:
/// typedef struct {
///   Point position;
///   Point velocity;
///   float mass;
/// } Particle;
final class Particle extends Struct {
  external Point position;
  external Point velocity;

  @Float()
  external double mass;
}

/// Array ใน struct:
/// typedef struct {
///   int data[10];
///   int size;
/// } IntArray;
final class IntArray extends Struct {
  @Array(10)
  external Array<Int32> data;

  @Int32()
  external int size;
}

class StructUsageExample {
  /// ตัวอย่างการสร้างและใช้งาน struct
  static void demonstrateStructs() {
    // สร้าง Point
    final point = malloc<Point>();
    point.ref.x = 10;
    point.ref.y = 20;
    print('Point: ${point.ref}');
    malloc.free(point);

    // สร้าง Complex number
    final complex = malloc<Complex>();
    complex.ref.real = 3.0;
    complex.ref.imaginary = 4.0;
    print('Complex: ${complex.ref}');
    print('Magnitude: ${complex.ref.magnitude}');
    malloc.free(complex);

    // ใช้ IntArray
    final intArray = malloc<IntArray>();
    intArray.ref.size = 5;
    for (var i = 0; i < 5; i++) {
      intArray.ref.data[i] = (i + 1) * 10;
    }
    print('Array elements:');
    for (var i = 0; i < intArray.ref.size; i++) {
      print('  [$i] = ${intArray.ref.data[i]}');
    }
    malloc.free(intArray);
  }

  /// ส่ง struct ไปยัง C function
  static void useStructWithCFunction(DynamicLibrary lib) {
    // C function: double distance(Point* p1, Point* p2)
    final distance = lib
        .lookup<NativeFunction<Double Function(Pointer<Point>, Pointer<Point>)>>(
            'distance')
        .asFunction<double Function(Pointer<Point>, Pointer<Point>)>();

    final p1 = malloc<Point>();
    final p2 = malloc<Point>();

    p1.ref.x = 0;
    p1.ref.y = 0;
    p2.ref.x = 3;
    p2.ref.y = 4;

    final dist = distance(p1, p2);
    print('Distance: $dist'); // Expected: 5.0

    malloc.free(p1);
    malloc.free(p2);
  }
}
```

---

## ขั้นตอนที่ 2564: Pointers และ Memory Management

```dart
// lib/ffi/pointers.dart
import 'dart:ffi';
import 'package:ffi/ffi.dart';

class FfiPointerDemo {
  /// ตัวอย่าง pointer arithmetic
  static void demonstratePointers() {
    // Allocate array ของ integers
    const length = 5;
    final ptr = malloc<Int32>(length);

    // กำหนดค่า
    for (var i = 0; i < length; i++) {
      ptr[i] = i * i; // 0, 1, 4, 9, 16
    }

    // อ่านค่าแบบ pointer arithmetic
    print('Array values:');
    for (var i = 0; i < length; i++) {
      final element = ptr.elementAt(i);
      print('  ptr[$i] = ${element.value}');
    }

    // ใช้ asTypedList สำหรับประสิทธิภาพที่ดีกว่า
    final list = ptr.asTypedList(length);
    print('As typed list: $list');

    malloc.free(ptr);
  }

  /// Double pointer (pointer to pointer)
  static void demonstrateDoublePointer(DynamicLibrary lib) {
    // C function: void allocateArray(int** arr, int size)
    final allocateArray = lib
        .lookup<NativeFunction<Void Function(Pointer<Pointer<Int32>>, Int32)>>(
            'allocateArray')
        .asFunction<void Function(Pointer<Pointer<Int32>>, int)>();

    final ptrToPtr = malloc<Pointer<Int32>>();
    allocateArray(ptrToPtr, 5);

    final arr = ptrToPtr.value;
    for (var i = 0; i < 5; i++) {
      print('arr[$i] = ${arr[i]}');
    }

    // Free allocated memory
    malloc.free(arr);
    malloc.free(ptrToPtr);
  }

  /// ตัวอย่าง Arena allocator
  static void demonstrateArena() {
    // Arena จัดการ memory แบบ batch
    using((arena) {
      final p1 = arena<Point>();
      p1.ref.x = 1;
      p1.ref.y = 2;

      final p2 = arena<Point>();
      p2.ref.x = 3;
      p2.ref.y = 4;

      final str = 'Hello FFI'.toNativeUtf8(allocator: arena);

      print('p1: (${p1.ref.x}, ${p1.ref.y})');
      print('p2: (${p2.ref.x}, ${p2.ref.y})');
      print('str: ${str.toDartString()}');
      // Arena จะ free memory ทั้งหมดเมื่อ block จบ
    });
  }
}

// Reuse Point struct from previous file
final class Point extends Struct {
  @Int32()
  external int x;

  @Int32()
  external int y;
}
```

---

## ขั้นตอนที่ 2565: Callbacks จาก Native Code ไปยัง Dart

```dart
// lib/ffi/callbacks.dart
import 'dart:ffi';
import 'package:ffi/ffi.dart';

/// ประเภทของ callback function
typedef NativeCallback = Void Function(Int32 value);
typedef DartCallback = void Function(int value);

/// Callback ที่มี return value
typedef NativeFilterCallback = Int32 Function(Int32 value);
typedef DartFilterCallback = int Function(int value);

class FfiCallbacks {
  static NativeCallable<NativeCallback>? _callbackHandle;
  static NativeCallable<NativeFilterCallback>? _filterHandle;

  /// ลงทะเบียน callback สำหรับ C code
  static void registerCallback(
    DynamicLibrary lib,
    void Function(int) dartCallback,
  ) {
    // C function: void registerListener(void (*callback)(int))
    final registerListener = lib
        .lookup<NativeFunction<Void Function(Pointer<NativeFunction<NativeCallback>>)>>(
            'registerListener')
        .asFunction<void Function(Pointer<NativeFunction<NativeCallback>>)>();

    // สร้าง NativeCallable จาก Dart function
    _callbackHandle =
        NativeCallable<NativeCallback>.listener(dartCallback);

    registerListener(_callbackHandle!.nativeFunction);
  }

  /// Callback แบบ synchronous
  static void useCallbackSync(
    DynamicLibrary lib,
    List<int> data,
    void Function(int) onEach,
  ) {
    // C function: void forEach(int* arr, int len, void (*callback)(int))
    final forEach = lib
        .lookup<NativeFunction<
            Void Function(
              Pointer<Int32>,
              Int32,
              Pointer<NativeFunction<NativeCallback>>,
            )>>('forEach')
        .asFunction<
            void Function(
              Pointer<Int32>,
              int,
              Pointer<NativeFunction<NativeCallback>>,
            )>();

    final arr = malloc<Int32>(data.length);
    for (var i = 0; i < data.length; i++) {
      arr[i] = data[i];
    }

    final callback =
        NativeCallable<NativeCallback>.isolateLocal(onEach);

    try {
      forEach(arr, data.length, callback.nativeFunction);
    } finally {
      callback.close();
      malloc.free(arr);
    }
  }

  /// Filter data ด้วย callback
  static List<int> filterWithCallback(
    DynamicLibrary lib,
    List<int> data,
    bool Function(int) predicate,
  ) {
    // C function: int filter(int* in, int* out, int len, int (*pred)(int))
    final filterFn = lib
        .lookup<NativeFunction<
            Int32 Function(
              Pointer<Int32>,
              Pointer<Int32>,
              Int32,
              Pointer<NativeFunction<NativeFilterCallback>>,
            )>>('filter')
        .asFunction<
            int Function(
              Pointer<Int32>,
              Pointer<Int32>,
              int,
              Pointer<NativeFunction<NativeFilterCallback>>,
            )>();

    final inputArr = malloc<Int32>(data.length);
    final outputArr = malloc<Int32>(data.length);

    for (var i = 0; i < data.length; i++) {
      inputArr[i] = data[i];
    }

    final dartPredicate = (int value) => predicate(value) ? 1 : 0;
    final callbackHandle =
        NativeCallable<NativeFilterCallback>.isolateLocal(dartPredicate);

    int resultCount;
    try {
      resultCount = filterFn(
        inputArr,
        outputArr,
        data.length,
        callbackHandle.nativeFunction,
      );
    } finally {
      callbackHandle.close();
    }

    final result = <int>[];
    for (var i = 0; i < resultCount; i++) {
      result.add(outputArr[i]);
    }

    malloc.free(inputArr);
    malloc.free(outputArr);

    return result;
  }

  static void cleanup() {
    _callbackHandle?.close();
    _callbackHandle = null;
    _filterHandle?.close();
    _filterHandle = null;
  }
}
```

---

## ขั้นตอนที่ 2566: ffigen สำหรับ Auto-generated Bindings

```yaml
# ffigen.yaml - configuration สำหรับ ffigen
# รันด้วย: dart run ffigen --config ffigen.yaml

output: 'lib/ffi/generated_bindings.dart'
name: 'MyLibBindings'
description: 'Auto-generated bindings for MyLib'

headers:
  entry-points:
    - 'native/mylib.h'
  include-directives:
    - 'native/mylib.h'

preamble: |
  // AUTO-GENERATED FILE - DO NOT EDIT
  // Generated by ffigen

functions:
  include:
    - 'ml_.*'
    - 'compute_.*'
  rename:
    'ml_(.*)': '$1'

structs:
  include:
    - 'ML.*'
    - 'Tensor.*'

enums:
  include:
    - 'MLError.*'
    - 'DataType.*'

typedef-map:
  'int32_t': 'Int32'
  'int64_t': 'Int64'
  'float': 'Float'
  'double': 'Double'
```

```c
// native/mylib.h - ตัวอย่าง C header file

#ifndef MYLIB_H
#define MYLIB_H

#include <stdint.h>

// Error codes
typedef enum {
    ML_SUCCESS = 0,
    ML_ERROR_INVALID_INPUT = -1,
    ML_ERROR_OUT_OF_MEMORY = -2,
    ML_ERROR_NOT_INITIALIZED = -3,
} MLError;

// Data types
typedef enum {
    DATA_TYPE_INT32,
    DATA_TYPE_INT64,
    DATA_TYPE_FLOAT32,
    DATA_TYPE_FLOAT64,
} DataType;

// Tensor struct
typedef struct {
    void* data;
    int32_t* shape;
    int32_t ndim;
    DataType dtype;
    int64_t total_elements;
} Tensor;

// Compute context
typedef struct {
    int32_t num_threads;
    int32_t device_id;
    void* device_context;
} MLContext;

// Function declarations
MLContext* ml_create_context(int32_t num_threads);
void ml_destroy_context(MLContext* ctx);

Tensor* ml_create_tensor(MLContext* ctx, int32_t* shape, int32_t ndim, DataType dtype);
void ml_destroy_tensor(Tensor* tensor);

MLError ml_fill_tensor(Tensor* tensor, float value);
MLError ml_compute_matmul(MLContext* ctx, Tensor* a, Tensor* b, Tensor* result);
MLError ml_compute_relu(MLContext* ctx, Tensor* input, Tensor* output);
MLError ml_compute_softmax(MLContext* ctx, Tensor* input, Tensor* output);

float ml_tensor_get_float(Tensor* tensor, int32_t* indices);
MLError ml_tensor_set_float(Tensor* tensor, int32_t* indices, float value);

int32_t ml_get_version_major(void);
int32_t ml_get_version_minor(void);
const char* ml_get_error_string(MLError error);

#endif // MYLIB_H
```

```dart
// lib/ffi/ml_wrapper.dart
// ตัวอย่าง wrapper class หลังจาก ffigen สร้าง bindings แล้ว

import 'dart:ffi';
import 'package:ffi/ffi.dart';
// import 'generated_bindings.dart'; // จะ import จาก generated file

class MlContext {
  // final MLContext _context; // จาก generated bindings
  // final MyLibBindings _bindings;

  MlContext._();

  static MlContext create({int numThreads = 4}) {
    final ctx = MlContext._();
    // ctx._context = ctx._bindings.ml_create_context(numThreads);
    return ctx;
  }

  void dispose() {
    // _bindings.ml_destroy_context(_context);
  }
}

class MlTensor {
  final List<int> shape;
  final int ndim;
  // final Tensor _tensor; // จาก generated bindings

  MlTensor._(this.shape, this.ndim);

  static MlTensor create(MlContext ctx, List<int> shape) {
    final tensor = MlTensor._(shape, shape.length);
    // tensor._tensor = bindings.ml_create_tensor(ctx._context, shapePtr, shape.length, DATA_TYPE_FLOAT32);
    return tensor;
  }

  void fill(double value) {
    // _bindings.ml_fill_tensor(_tensor, value);
  }

  void dispose() {
    // _bindings.ml_destroy_tensor(_tensor);
  }
}

/// ตัวอย่างการใช้งาน ML wrapper
class MlExample {
  static void runMatMulExample() {
    final ctx = MlContext.create(numThreads: 4);

    final a = MlTensor.create(ctx, [2, 3]);
    final b = MlTensor.create(ctx, [3, 2]);
    final result = MlTensor.create(ctx, [2, 2]);

    a.fill(1.0);
    b.fill(2.0);

    // _bindings.ml_compute_matmul(ctx._context, a._tensor, b._tensor, result._tensor);

    print('Matrix multiplication completed');
    print('Output shape: ${result.shape}');

    result.dispose();
    b.dispose();
    a.dispose();
    ctx.dispose();
  }
}
```

---

## ขั้นตอนที่ 2567: ตัวอย่าง FFI ที่ใช้งานจริง - Image Processing

```dart
// lib/ffi/image_processor.dart
import 'dart:ffi';
import 'dart:io';
import 'dart:typed_data';
import 'package:ffi/ffi.dart';

// C functions สำหรับ image processing:
// void applyGrayscale(uint8_t* pixels, int width, int height);
// void applyBlur(uint8_t* pixels, int width, int height, int radius);
// float calculateBrightness(uint8_t* pixels, int width, int height);
// void applyThreshold(uint8_t* pixels, int width, int height, uint8_t threshold);

typedef GrayscaleC = Void Function(Pointer<Uint8>, Int32, Int32);
typedef GrayscaleDart = void Function(Pointer<Uint8>, int, int);

typedef BlurC = Void Function(Pointer<Uint8>, Int32, Int32, Int32);
typedef BlurDart = void Function(Pointer<Uint8>, int, int, int);

typedef BrightnessC = Float Function(Pointer<Uint8>, Int32, Int32);
typedef BrightnessDart = double Function(Pointer<Uint8>, int, int);

typedef ThresholdC = Void Function(Pointer<Uint8>, Int32, Int32, Uint8);
typedef ThresholdDart = void Function(Pointer<Uint8>, int, int, int);

class FfiImageProcessor {
  late final GrayscaleDart _applyGrayscale;
  late final BlurDart _applyBlur;
  late final BrightnessDart _calculateBrightness;
  late final ThresholdDart _applyThreshold;

  bool _initialized = false;

  void initialize() {
    final libPath = _getLibPath();
    final lib = DynamicLibrary.open(libPath);

    _applyGrayscale = lib
        .lookup<NativeFunction<GrayscaleC>>('applyGrayscale')
        .asFunction();

    _applyBlur = lib
        .lookup<NativeFunction<BlurC>>('applyBlur')
        .asFunction();

    _calculateBrightness = lib
        .lookup<NativeFunction<BrightnessC>>('calculateBrightness')
        .asFunction();

    _applyThreshold = lib
        .lookup<NativeFunction<ThresholdC>>('applyThreshold')
        .asFunction();

    _initialized = true;
  }

  String _getLibPath() {
    if (Platform.isLinux) return './libimageproc.so';
    if (Platform.isMacOS) return './libimageproc.dylib';
    if (Platform.isWindows) return './imageproc.dll';
    throw UnsupportedError('Unsupported platform');
  }

  Uint8List applyGrayscale(Uint8List pixels, int width, int height) {
    _checkInitialized();

    final ptr = _pixelsToPointer(pixels);
    try {
      _applyGrayscale(ptr, width, height);
      return _pointerToPixels(ptr, pixels.length);
    } finally {
      malloc.free(ptr);
    }
  }

  Uint8List applyBlur(Uint8List pixels, int width, int height,
      {int radius = 3}) {
    _checkInitialized();

    final ptr = _pixelsToPointer(pixels);
    try {
      _applyBlur(ptr, width, height, radius);
      return _pointerToPixels(ptr, pixels.length);
    } finally {
      malloc.free(ptr);
    }
  }

  double calculateBrightness(Uint8List pixels, int width, int height) {
    _checkInitialized();

    final ptr = _pixelsToPointer(pixels);
    try {
      return _calculateBrightness(ptr, width, height);
    } finally {
      malloc.free(ptr);
    }
  }

  Uint8List applyThreshold(Uint8List pixels, int width, int height,
      {int threshold = 128}) {
    _checkInitialized();

    final ptr = _pixelsToPointer(pixels);
    try {
      _applyThreshold(ptr, width, height, threshold);
      return _pointerToPixels(ptr, pixels.length);
    } finally {
      malloc.free(ptr);
    }
  }

  Pointer<Uint8> _pixelsToPointer(Uint8List pixels) {
    final ptr = malloc<Uint8>(pixels.length);
    final byteList = ptr.asTypedList(pixels.length);
    byteList.setAll(0, pixels);
    return ptr;
  }

  Uint8List _pointerToPixels(Pointer<Uint8> ptr, int length) {
    return Uint8List.fromList(ptr.asTypedList(length));
  }

  void _checkInitialized() {
    if (!_initialized) {
      throw StateError('FfiImageProcessor not initialized. Call initialize() first.');
    }
  }
}

/// Pure Dart implementation สำหรับ testing (ไม่ต้องใช้ FFI)
class PureDartImageProcessor {
  static Uint8List applyGrayscale(Uint8List pixels, int width, int height) {
    final result = Uint8List(pixels.length);

    for (var i = 0; i < pixels.length; i += 4) {
      final r = pixels[i];
      final g = pixels[i + 1];
      final b = pixels[i + 2];
      final a = pixels[i + 3];

      // Luminance formula
      final gray = (0.299 * r + 0.587 * g + 0.114 * b).round();
      result[i] = gray;
      result[i + 1] = gray;
      result[i + 2] = gray;
      result[i + 3] = a;
    }

    return result;
  }

  static double calculateBrightness(Uint8List pixels) {
    var totalLuminance = 0.0;
    final pixelCount = pixels.length ~/ 4;

    for (var i = 0; i < pixels.length; i += 4) {
      final r = pixels[i];
      final g = pixels[i + 1];
      final b = pixels[i + 2];
      totalLuminance += 0.299 * r + 0.587 * g + 0.114 * b;
    }

    return totalLuminance / pixelCount / 255.0;
  }
}
```

---

## ขั้นตอนที่ 2568: ทดสอบ FFI Code

```dart
// test/ffi_test.dart
import 'package:test/test.dart';
import '../lib/ffi/image_processor.dart';
import 'dart:typed_data';

void main() {
  group('PureDartImageProcessor', () {
    test('applyGrayscale converts colors correctly', () {
      // RGBA pixel: red
      final pixels = Uint8List.fromList([255, 0, 0, 255]);

      final result = PureDartImageProcessor.applyGrayscale(pixels, 1, 1);

      // สี gray ของ red = 0.299 * 255 ≈ 76
      expect(result[0], closeTo(76, 1));
      expect(result[1], closeTo(76, 1));
      expect(result[2], closeTo(76, 1));
      expect(result[3], equals(255)); // alpha ไม่เปลี่ยน
    });

    test('calculateBrightness returns correct value', () {
      // White pixel = brightness 1.0
      final whitePixel = Uint8List.fromList([255, 255, 255, 255]);
      expect(
        PureDartImageProcessor.calculateBrightness(whitePixel),
        closeTo(1.0, 0.01),
      );

      // Black pixel = brightness 0.0
      final blackPixel = Uint8List.fromList([0, 0, 0, 255]);
      expect(
        PureDartImageProcessor.calculateBrightness(blackPixel),
        closeTo(0.0, 0.01),
      );
    });

    test('applyGrayscale preserves alpha channel', () {
      // Semi-transparent pixel
      final pixels = Uint8List.fromList([100, 150, 200, 128]);
      final result = PureDartImageProcessor.applyGrayscale(pixels, 1, 1);
      expect(result[3], equals(128)); // alpha preserved
    });
  });
}

// bin/run_ffi_example.dart
// import 'package:my_app/ffi/structs.dart';
//
// void main() {
//   print('=== Dart FFI Demo ===\n');
//
//   print('1. Struct Demo:');
//   StructUsageExample.demonstrateStructs();
//
//   print('\n2. Pointer Demo:');
//   FfiPointerDemo.demonstratePointers();
//
//   print('\n3. Pure Dart Image Processing:');
//   final pixels = Uint8List.fromList([
//     255, 0, 0, 255,    // Red pixel
//     0, 255, 0, 255,    // Green pixel
//     0, 0, 255, 255,    // Blue pixel
//     255, 255, 255, 255 // White pixel
//   ]);
//   final gray = PureDartImageProcessor.applyGrayscale(pixels, 2, 2);
//   print('Grayscale values: ${gray.take(12).toList()}');
// }
```

---

**← [Part 66](part-66-flutter-web-advanced.md)**
**ต่อไป: [Part 68 →](part-68-flutter-ml-tflite.md)**
