# Part 77: Dart Meta-Programming
## ขั้นตอนที่ 2961-3000

## 🎯 เป้าหมายของ Part นี้
- ใช้ Reflection ด้วย dart:mirrors (server-side)
- สร้าง Custom Annotations
- ใช้ build_runner และ source_gen สร้าง code generation
- เจาะลึก json_serializable สำหรับ JSON serialization
- สร้าง Custom Code Generator ของตัวเอง

---

## ขั้นตอนที่ 2961: Custom Annotations

```dart
// lib/annotations/annotations.dart

/// Annotation สำหรับ JSON serialization
class JsonSerializable {
  const JsonSerializable({
    this.explicitToJson = false,
    this.includeIfNull = true,
    this.fieldRename = FieldRename.none,
  });

  final bool explicitToJson;
  final bool includeIfNull;
  final FieldRename fieldRename;
}

enum FieldRename {
  none,
  snake,
  kebab,
  pascal,
}

/// Annotation สำหรับแต่ละ field
class JsonKey {
  const JsonKey({
    this.name,
    this.required = false,
    this.ignore = false,
    this.defaultValue,
    this.fromJson,
    this.toJson,
  });

  final String? name;
  final bool required;
  final bool ignore;
  final dynamic defaultValue;
  final Function? fromJson;
  final Function? toJson;
}

/// Annotation สำหรับ validation
class Validate {
  const Validate({
    this.minLength,
    this.maxLength,
    this.pattern,
    this.min,
    this.max,
    this.required = false,
  });

  final int? minLength;
  final int? maxLength;
  final String? pattern;
  final num? min;
  final num? max;
  final bool required;
}

/// Annotation สำหรับ API endpoint
class ApiEndpoint {
  const ApiEndpoint({
    required this.path,
    this.method = 'GET',
    this.auth = false,
  });

  final String path;
  final String method;
  final bool auth;
}

/// Annotation สำหรับ Singleton
class Singleton {
  const Singleton();
}

/// Annotation สำหรับ Injectable (Dependency Injection)
class Injectable {
  const Injectable({this.as});
  final Type? as;
}

/// Annotation สำหรับ Repository
class Repository {
  const Repository({required this.entity});
  final Type entity;
}
```

---

## ขั้นตอนที่ 2962: Models ที่ใช้ Annotations

```dart
// lib/models/user_model.dart
import '../annotations/annotations.dart';

// ก่อน code generation - นี่คือ model ที่เราเขียน
@JsonSerializable(
  explicitToJson: true,
  fieldRename: FieldRename.snake,
)
class User {
  const User({
    required this.id,
    required this.firstName,
    required this.lastName,
    required this.email,
    this.avatar,
    required this.createdAt,
    this.roles = const [],
  });

  @JsonKey(name: 'id')
  final int id;

  @JsonKey(name: 'first_name')
  @Validate(minLength: 2, maxLength: 50, required: true)
  final String firstName;

  @JsonKey(name: 'last_name')
  @Validate(minLength: 2, maxLength: 50, required: true)
  final String lastName;

  @JsonKey(name: 'email')
  @Validate(
    required: true,
    pattern: r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$',
  )
  final String email;

  @JsonKey(name: 'avatar_url')
  final String? avatar;

  @JsonKey(name: 'created_at')
  final DateTime createdAt;

  @JsonKey(name: 'roles', defaultValue: [])
  final List<String> roles;

  // factory constructor จะถูก generate โดย json_serializable
  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);

  Map<String, dynamic> toJson() => _$UserToJson(this);

  // copyWith สำหรับ immutable updates
  User copyWith({
    int? id,
    String? firstName,
    String? lastName,
    String? email,
    String? avatar,
    DateTime? createdAt,
    List<String>? roles,
  }) {
    return User(
      id: id ?? this.id,
      firstName: firstName ?? this.firstName,
      lastName: lastName ?? this.lastName,
      email: email ?? this.email,
      avatar: avatar ?? this.avatar,
      createdAt: createdAt ?? this.createdAt,
      roles: roles ?? this.roles,
    );
  }

  @override
  String toString() =>
      'User(id: $id, name: $firstName $lastName, email: $email)';

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is User &&
          runtimeType == other.runtimeType &&
          id == other.id &&
          email == other.email;

  @override
  int get hashCode => id.hashCode ^ email.hashCode;
}

// lib/models/product_model.dart
@JsonSerializable(explicitToJson: true)
class Product {
  const Product({
    required this.id,
    required this.name,
    required this.price,
    required this.category,
    this.description,
    this.imageUrl,
    this.stock = 0,
    this.tags = const [],
  });

  final String id;
  final String name;
  final double price;
  final String category;
  final String? description;

  @JsonKey(name: 'image_url')
  final String? imageUrl;

  final int stock;
  final List<String> tags;

  bool get isInStock => stock > 0;

  factory Product.fromJson(Map<String, dynamic> json) =>
      _$ProductFromJson(json);

  Map<String, dynamic> toJson() => _$ProductToJson(this);
}

// lib/models/order_model.dart
@JsonSerializable(explicitToJson: true)
class Order {
  const Order({
    required this.id,
    required this.userId,
    required this.items,
    required this.status,
    required this.createdAt,
    this.totalAmount,
  });

  final String id;

  @JsonKey(name: 'user_id')
  final int userId;

  final List<OrderItem> items;

  @JsonKey(fromJson: _orderStatusFromJson, toJson: _orderStatusToJson)
  final OrderStatus status;

  @JsonKey(name: 'created_at')
  final DateTime createdAt;

  @JsonKey(name: 'total_amount')
  final double? totalAmount;

  double get calculatedTotal =>
      items.fold(0, (sum, item) => sum + item.total);

  factory Order.fromJson(Map<String, dynamic> json) => _$OrderFromJson(json);
  Map<String, dynamic> toJson() => _$OrderToJson(this);

  static OrderStatus _orderStatusFromJson(String value) =>
      OrderStatus.values.firstWhere((e) => e.name == value);

  static String _orderStatusToJson(OrderStatus status) => status.name;
}

enum OrderStatus { pending, confirmed, shipped, delivered, cancelled }

@JsonSerializable()
class OrderItem {
  const OrderItem({
    required this.productId,
    required this.quantity,
    required this.unitPrice,
  });

  @JsonKey(name: 'product_id')
  final String productId;

  final int quantity;

  @JsonKey(name: 'unit_price')
  final double unitPrice;

  double get total => quantity * unitPrice;

  factory OrderItem.fromJson(Map<String, dynamic> json) =>
      _$OrderItemFromJson(json);
  Map<String, dynamic> toJson() => _$OrderItemToJson(this);
}
```

---

## ขั้นตอนที่ 2963: Generated Code (ตัวอย่างที่ build_runner สร้าง)

```dart
// lib/models/user_model.g.dart
// GENERATED CODE - DO NOT MODIFY BY HAND
// ignore_for_file: type=lint
// **************************************************************************
// JsonSerializableGenerator
// **************************************************************************

User _$UserFromJson(Map<String, dynamic> json) => User(
      id: (json['id'] as num).toInt(),
      firstName: json['first_name'] as String,
      lastName: json['last_name'] as String,
      email: json['email'] as String,
      avatar: json['avatar_url'] as String?,
      createdAt: DateTime.parse(json['created_at'] as String),
      roles: (json['roles'] as List<dynamic>?)
              ?.map((e) => e as String)
              .toList() ??
          const [],
    );

Map<String, dynamic> _$UserToJson(User instance) => <String, dynamic>{
      'id': instance.id,
      'first_name': instance.firstName,
      'last_name': instance.lastName,
      'email': instance.email,
      'avatar_url': instance.avatar,
      'created_at': instance.createdAt.toIso8601String(),
      'roles': instance.roles,
    };

// lib/models/product_model.g.dart
Product _$ProductFromJson(Map<String, dynamic> json) => Product(
      id: json['id'] as String,
      name: json['name'] as String,
      price: (json['price'] as num).toDouble(),
      category: json['category'] as String,
      description: json['description'] as String?,
      imageUrl: json['image_url'] as String?,
      stock: (json['stock'] as num?)?.toInt() ?? 0,
      tags: (json['tags'] as List<dynamic>?)
              ?.map((e) => e as String)
              .toList() ??
          const [],
    );

Map<String, dynamic> _$ProductToJson(Product instance) => <String, dynamic>{
      'id': instance.id,
      'name': instance.name,
      'price': instance.price,
      'category': instance.category,
      'description': instance.description,
      'image_url': instance.imageUrl,
      'stock': instance.stock,
      'tags': instance.tags,
    };

// lib/models/order_model.g.dart
Order _$OrderFromJson(Map<String, dynamic> json) => Order(
      id: json['id'] as String,
      userId: (json['user_id'] as num).toInt(),
      items: (json['items'] as List<dynamic>)
          .map((e) => OrderItem.fromJson(e as Map<String, dynamic>))
          .toList(),
      status: Order._orderStatusFromJson(json['status'] as String),
      createdAt: DateTime.parse(json['created_at'] as String),
      totalAmount: (json['total_amount'] as num?)?.toDouble(),
    );

Map<String, dynamic> _$OrderToJson(Order instance) => <String, dynamic>{
      'id': instance.id,
      'user_id': instance.userId,
      'items': instance.items.map((e) => e.toJson()).toList(),
      'status': Order._orderStatusToJson(instance.status),
      'created_at': instance.createdAt.toIso8601String(),
      'total_amount': instance.totalAmount,
    };

OrderItem _$OrderItemFromJson(Map<String, dynamic> json) => OrderItem(
      productId: json['product_id'] as String,
      quantity: (json['quantity'] as num).toInt(),
      unitPrice: (json['unit_price'] as num).toDouble(),
    );

Map<String, dynamic> _$OrderItemToJson(OrderItem instance) =>
    <String, dynamic>{
      'product_id': instance.productId,
      'quantity': instance.quantity,
      'unit_price': instance.unitPrice,
    };
```

---

## ขั้นตอนที่ 2964: Custom Code Generator ด้วย source_gen

```dart
// tool/generator/validator_generator.dart
// (รันใน Dart VM - ไม่ใช่ Flutter)

// นี่คือ Generator ที่สร้างขึ้นเอง
// pubspec.yaml (สำหรับ generator package):
// dependencies:
//   source_gen: ^1.4.0
//   analyzer: ^6.0.0
//   build: ^2.4.0

import 'package:analyzer/dart/element/element.dart';
import 'package:build/build.dart';
import 'package:source_gen/source_gen.dart';

// Annotation class (ต้องอยู่ใน package ที่ import ได้)
class GenerateValidator {
  const GenerateValidator();
}

// Generator
class ValidatorGenerator extends GeneratorForAnnotation<GenerateValidator> {
  @override
  String generateForAnnotatedElement(
    Element element,
    ConstantReader annotation,
    BuildStep buildStep,
  ) {
    if (element is! ClassElement) {
      throw InvalidGenerationSourceError(
        'GenerateValidator can only be applied to classes.',
        element: element,
      );
    }

    final className = element.name;
    final buffer = StringBuffer();

    buffer.writeln('// Generated validator for $className');
    buffer.writeln('class ${className}Validator {');

    // Generate validation methods for each field
    for (final field in element.fields) {
      final validateAnnotation = field.metadata
          .where((m) => m.element?.enclosingElement?.name == 'Validate')
          .firstOrNull;

      if (validateAnnotation != null) {
        _generateFieldValidator(buffer, field, validateAnnotation);
      }
    }

    // Generate validate all method
    buffer.writeln('  static Map<String, String?> validateAll($className model) {');
    buffer.writeln('    final errors = <String, String?>{};');
    for (final field in element.fields) {
      final hasValidation = field.metadata
          .any((m) => m.element?.enclosingElement?.name == 'Validate');
      if (hasValidation) {
        buffer.writeln(
            '    errors[\'${field.name}\'] = validate${_capitalize(field.name)}(model.${field.name});');
      }
    }
    buffer.writeln('    errors.removeWhere((key, value) => value == null);');
    buffer.writeln('    return errors;');
    buffer.writeln('  }');

    buffer.writeln('}');
    return buffer.toString();
  }

  void _generateFieldValidator(
    StringBuffer buffer,
    FieldElement field,
    dynamic annotation,
  ) {
    final fieldName = field.name;
    final typeName = field.type.toString();

    buffer.writeln(
        '  static String? validate${_capitalize(fieldName)}($typeName value) {');

    // ตรวจ required
    buffer.writeln('    // Required check');
    buffer.writeln('    if (value == null || value.toString().isEmpty) {');
    buffer.writeln("      return '$fieldName is required';");
    buffer.writeln('    }');

    // ตรวจ minLength สำหรับ String
    if (typeName == 'String' || typeName == 'String?') {
      buffer.writeln('    // Length validation');
      buffer.writeln('    if (value.length < 2) {');
      buffer.writeln("      return '$fieldName must be at least 2 characters';");
      buffer.writeln('    }');
    }

    buffer.writeln('    return null;');
    buffer.writeln('  }');
    buffer.writeln();
  }

  String _capitalize(String s) =>
      s.isEmpty ? s : s[0].toUpperCase() + s.substring(1);
}

// Builder factory function
Builder validatorBuilder(BuilderOptions options) =>
    SharedPartBuilder([ValidatorGenerator()], 'validator');
```

---

## ขั้นตอนที่ 2965: build.yaml Configuration

```yaml
# build.yaml (ใน root ของ project)
targets:
  $default:
    builders:
      json_serializable:
        options:
          # Global options สำหรับ json_serializable
          explicit_to_json: true
          include_if_null: false
          field_rename: snake
      
      # Custom builder ของเรา
      my_validator_generator|validator:
        enabled: true
        generate_for:
          include:
            - lib/models/**.dart

builders:
  json_serializable:
    import: "package:json_serializable/builder.dart"
    builder_factories: ["jsonSerializable"]
    build_extensions: {".dart": [".g.dart"]}
    auto_apply: dependents
    build_to: cache
    applies_builders: ["source_gen|combining_builder"]
```

---

## ขั้นตอนที่ 2966: Freezed Code Generation

```dart
// lib/models/freezed_models.dart
// pubspec.yaml:
// dependencies:
//   freezed_annotation: ^2.4.1
// dev_dependencies:
//   freezed: ^2.4.5
//   build_runner: ^2.4.7

import 'package:freezed_annotation/freezed_annotation.dart';

part 'freezed_models.freezed.dart';
part 'freezed_models.g.dart';

/// Freezed สร้าง:
/// - immutable classes
/// - copyWith
/// - == และ hashCode
/// - toString
/// - Union types (sealed classes)

// 1. Simple immutable model
@freezed
class AppConfig with _$AppConfig {
  const factory AppConfig({
    required String apiBaseUrl,
    required String apiKey,
    @Default(30) int timeoutSeconds,
    @Default(false) bool debugMode,
    @Default([]) List<String> allowedOrigins,
  }) = _AppConfig;

  factory AppConfig.fromJson(Map<String, dynamic> json) =>
      _$AppConfigFromJson(json);

  // Custom methods (ต้องมี private constructor)
  const AppConfig._();

  bool get isProduction => !debugMode;
  String get fullApiUrl => '$apiBaseUrl/api/v1';
}

// 2. Union type สำหรับ API Response
@freezed
class ApiResponse<T> with _$ApiResponse<T> {
  const factory ApiResponse.success({
    required T data,
    String? message,
  }) = ApiSuccess<T>;

  const factory ApiResponse.error({
    required String message,
    int? statusCode,
    Map<String, dynamic>? details,
  }) = ApiError<T>;

  const factory ApiResponse.loading() = ApiLoading<T>;
}

// 3. เรียกใช้ Union type
void handleApiResponse(ApiResponse<User> response) {
  // Pattern matching ด้วย when
  response.when(
    success: (data, message) {
      print('Got user: ${data.firstName}');
    },
    error: (message, statusCode, details) {
      print('Error $statusCode: $message');
    },
    loading: () {
      print('Loading...');
    },
  );

  // Pattern matching ด้วย map
  final result = response.map(
    success: (s) => 'Success: ${s.data}',
    error: (e) => 'Error: ${e.message}',
    loading: (_) => 'Loading...',
  );

  // maybeWhen สำหรับ partial handling
  response.maybeWhen(
    success: (data, message) => print('Got user'),
    orElse: () => print('Not success'),
  );
}

// 4. Nested Freezed models
@freezed
class Address with _$Address {
  const factory Address({
    required String street,
    required String city,
    required String country,
    String? zipCode,
  }) = _Address;

  factory Address.fromJson(Map<String, dynamic> json) =>
      _$AddressFromJson(json);
}

@freezed
class UserProfile with _$UserProfile {
  const factory UserProfile({
    required User user,
    required Address address,
    @Default([]) List<String> preferences,
  }) = _UserProfile;

  factory UserProfile.fromJson(Map<String, dynamic> json) =>
      _$UserProfileFromJson(json);
}

// 5. Sealed classes (Dart 3.0+)
// ใช้ sealed keyword แทน freezed สำหรับ simple union types
sealed class AuthState {}

final class Authenticated extends AuthState {
  Authenticated({required this.user, required this.token});
  final User user;
  final String token;
}

final class Unauthenticated extends AuthState {}

final class AuthLoading extends AuthState {}

final class AuthError extends AuthState {
  AuthError({required this.message});
  final String message;
}

// Pattern matching กับ sealed classes
Widget buildAuthWidget(AuthState state) {
  return switch (state) {
    Authenticated(user: final user) => Text('Welcome, ${user.firstName}!'),
    Unauthenticated() => const LoginPage(),
    AuthLoading() => const CircularProgressIndicator(),
    AuthError(message: final msg) => Text('Error: $msg'),
  };
}
```

---

## ขั้นตอนที่ 2967: JSON Serialization Deep Dive

```dart
// lib/serialization/json_serialization_advanced.dart
import 'dart:convert';

/// Advanced JSON Serialization patterns

// 1. Custom DateTime serialization
class DateTimeConverter {
  static DateTime fromJson(String value) {
    // Handle multiple date formats
    try {
      return DateTime.parse(value);
    } catch (_) {
      // Try Unix timestamp
      final timestamp = int.tryParse(value);
      if (timestamp != null) {
        return DateTime.fromMillisecondsSinceEpoch(timestamp * 1000);
      }
      throw FormatException('Cannot parse date: $value');
    }
  }

  static String toJson(DateTime dateTime) {
    return dateTime.toIso8601String();
  }
}

// 2. Polymorphic deserialization
abstract class Shape {
  const Shape({required this.color});

  final String color;

  factory Shape.fromJson(Map<String, dynamic> json) {
    switch (json['type'] as String) {
      case 'circle':
        return Circle.fromJson(json);
      case 'rectangle':
        return Rectangle.fromJson(json);
      case 'triangle':
        return Triangle.fromJson(json);
      default:
        throw ArgumentError('Unknown shape type: ${json['type']}');
    }
  }

  Map<String, dynamic> toJson();
  double get area;
}

class Circle extends Shape {
  const Circle({required super.color, required this.radius});

  final double radius;

  factory Circle.fromJson(Map<String, dynamic> json) => Circle(
        color: json['color'] as String,
        radius: (json['radius'] as num).toDouble(),
      );

  @override
  Map<String, dynamic> toJson() => {
        'type': 'circle',
        'color': color,
        'radius': radius,
      };

  @override
  double get area => 3.14159265 * radius * radius;
}

class Rectangle extends Shape {
  const Rectangle({
    required super.color,
    required this.width,
    required this.height,
  });

  final double width;
  final double height;

  factory Rectangle.fromJson(Map<String, dynamic> json) => Rectangle(
        color: json['color'] as String,
        width: (json['width'] as num).toDouble(),
        height: (json['height'] as num).toDouble(),
      );

  @override
  Map<String, dynamic> toJson() => {
        'type': 'rectangle',
        'color': color,
        'width': width,
        'height': height,
      };

  @override
  double get area => width * height;
}

class Triangle extends Shape {
  const Triangle({
    required super.color,
    required this.base,
    required this.height,
  });

  final double base;
  final double height;

  factory Triangle.fromJson(Map<String, dynamic> json) => Triangle(
        color: json['color'] as String,
        base: (json['base'] as num).toDouble(),
        height: (json['height'] as num).toDouble(),
      );

  @override
  Map<String, dynamic> toJson() => {
        'type': 'triangle',
        'color': color,
        'base': base,
        'height': height,
      };

  @override
  double get area => 0.5 * base * height;
}

// 3. JSON Transformation Pipeline
class JsonTransformer {
  static Map<String, dynamic> snakeToCamel(Map<String, dynamic> json) {
    return json.map((key, value) {
      final camelKey = _snakeToCamelCase(key);
      final transformedValue = value is Map<String, dynamic>
          ? snakeToCamel(value)
          : value is List
              ? value
                  .map((e) =>
                      e is Map<String, dynamic> ? snakeToCamel(e) : e)
                  .toList()
              : value;
      return MapEntry(camelKey, transformedValue);
    });
  }

  static String _snakeToCamelCase(String snake) {
    final parts = snake.split('_');
    return parts.first +
        parts.skip(1).map((p) => p.isEmpty ? '' : p[0].toUpperCase() + p.substring(1)).join('');
  }

  static Map<String, dynamic> camelToSnake(Map<String, dynamic> json) {
    return json.map((key, value) {
      final snakeKey = _camelToSnakeCase(key);
      final transformedValue = value is Map<String, dynamic>
          ? camelToSnake(value)
          : value is List
              ? value
                  .map((e) =>
                      e is Map<String, dynamic> ? camelToSnake(e) : e)
                  .toList()
              : value;
      return MapEntry(snakeKey, transformedValue);
    });
  }

  static String _camelToSnakeCase(String camel) {
    return camel.replaceAllMapped(
      RegExp(r'[A-Z]'),
      (m) => '_${m.group(0)!.toLowerCase()}',
    );
  }
}

// 4. Type-safe JSON parsing
extension JsonExtensions on Map<String, dynamic> {
  T get<T>(String key, {T? defaultValue}) {
    final value = this[key];
    if (value == null) {
      if (defaultValue != null) return defaultValue;
      throw ArgumentError('Key "$key" not found in JSON');
    }
    if (value is T) return value;
    
    // Try type coercion
    if (T == double && value is int) return value.toDouble() as T;
    if (T == int && value is double) return value.toInt() as T;
    if (T == String) return value.toString() as T;
    
    throw TypeError();
  }

  T? tryGet<T>(String key) {
    try {
      return get<T>(key);
    } catch (_) {
      return null;
    }
  }

  List<T> getList<T>(String key) {
    final value = this[key];
    if (value == null) return [];
    if (value is List) {
      return value.cast<T>();
    }
    return [];
  }
}

// 5. Demo
void demonstrateJsonSerialization() {
  // Polymorphic deserialization
  final jsonStr = '''
  [
    {"type": "circle", "color": "red", "radius": 5.0},
    {"type": "rectangle", "color": "blue", "width": 10.0, "height": 5.0},
    {"type": "triangle", "color": "green", "base": 8.0, "height": 6.0}
  ]
  ''';

  final shapes = (jsonDecode(jsonStr) as List)
      .map((json) => Shape.fromJson(json as Map<String, dynamic>))
      .toList();

  for (final shape in shapes) {
    print('${shape.runtimeType}: area = ${shape.area.toStringAsFixed(2)}');
  }

  // JSON transformation
  final snakeJson = {'first_name': 'John', 'last_name': 'Doe', 'user_id': 1};
  final camelJson = JsonTransformer.snakeToCamel(snakeJson);
  print('Camel: $camelJson'); // {firstName: John, lastName: Doe, userId: 1}

  // Serialization roundtrip
  final user = User(
    id: 1,
    firstName: 'Alice',
    lastName: 'Smith',
    email: 'alice@example.com',
    createdAt: DateTime.now(),
  );

  final userJson = user.toJson();
  final restoredUser = User.fromJson(userJson);
  print('Original: $user');
  print('Restored: $restoredUser');
  print('Equal: ${user == restoredUser}');
}

// pubspec.yaml สำหรับ code generation:
// dependencies:
//   json_annotation: ^4.8.1
//   freezed_annotation: ^2.4.1
//
// dev_dependencies:
//   build_runner: ^2.4.7
//   json_serializable: ^6.7.1
//   freezed: ^2.4.5
//
// คำสั่ง:
// dart run build_runner build --delete-conflicting-outputs
// dart run build_runner watch  (สำหรับ development)
```

---

## ขั้นตอนที่ 2968: Reflection ด้วย dart:mirrors

```dart
// lib/reflection/mirrors_demo.dart
// หมายเหตุ: dart:mirrors ใช้ได้เฉพาะ Dart VM (ไม่รองรับ Flutter web/mobile)
// ใช้สำหรับ server-side Dart หรือ testing tools

// import 'dart:mirrors';

// ตัวอย่างการใช้ mirrors (pseudo-code สำหรับ server-side)
/*
import 'dart:mirrors';

// 1. Instance inspection
void inspectObject(Object obj) {
  final mirror = reflect(obj);
  final classMirror = mirror.type;
  
  print('Class: ${MirrorSystem.getName(classMirror.simpleName)}');
  
  // ดู fields
  classMirror.declarations.forEach((symbol, declaration) {
    if (declaration is VariableMirror) {
      final name = MirrorSystem.getName(symbol);
      final value = mirror.getField(symbol).reflectee;
      final type = MirrorSystem.getName(declaration.type.simpleName);
      print('  Field $name: $type = $value');
    }
  });
  
  // ดู methods
  classMirror.declarations.forEach((symbol, declaration) {
    if (declaration is MethodMirror && !declaration.isConstructor) {
      final name = MirrorSystem.getName(symbol);
      print('  Method: $name');
    }
  });
}

// 2. Dynamic method invocation
dynamic callMethod(Object obj, String methodName, List args) {
  final mirror = reflect(obj);
  return mirror.invoke(Symbol(methodName), args).reflectee;
}

// 3. Annotation reading
List<InstanceMirror> getAnnotations(ClassMirror classMirror, Type annotationType) {
  final typeMirror = reflectClass(annotationType);
  return classMirror.metadata
    .where((m) => m.type == typeMirror)
    .toList();
}
*/

// แทนที่ dart:mirrors ด้วย code generation (แนะนำ)
// ตัวอย่าง: simple dependency injection container

class ServiceLocator {
  static final ServiceLocator _instance = ServiceLocator._();
  factory ServiceLocator() => _instance;
  ServiceLocator._();

  final Map<Type, dynamic> _services = {};
  final Map<Type, dynamic Function()> _factories = {};

  // Register singleton
  void register<T>(T service) {
    _services[T] = service;
  }

  // Register factory
  void registerFactory<T>(T Function() factory) {
    _factories[T] = factory;
  }

  // Resolve
  T resolve<T>() {
    if (_services.containsKey(T)) {
      return _services[T] as T;
    }
    if (_factories.containsKey(T)) {
      return _factories[T]!() as T;
    }
    throw StateError('Service ${T.toString()} not registered');
  }

  void reset() {
    _services.clear();
    _factories.clear();
  }
}

// Services
abstract class Logger {
  void log(String message);
  void error(String message, [Object? error]);
}

class ConsoleLogger implements Logger {
  @override
  void log(String message) {
    print('[LOG] $message');
  }

  @override
  void error(String message, [Object? error]) {
    print('[ERROR] $message${error != null ? ': $error' : ''}');
  }
}

abstract class UserRepository {
  Future<User?> findById(int id);
  Future<List<User>> findAll();
  Future<User> save(User user);
}

class InMemoryUserRepository implements UserRepository {
  final Map<int, User> _store = {};
  int _nextId = 1;

  @override
  Future<User?> findById(int id) async => _store[id];

  @override
  Future<List<User>> findAll() async => _store.values.toList();

  @override
  Future<User> save(User user) async {
    final savedUser = user.copyWith(id: user.id == 0 ? _nextId++ : user.id);
    _store[savedUser.id] = savedUser;
    return savedUser;
  }
}

class UserService {
  UserService({
    required this.repository,
    required this.logger,
  });

  final UserRepository repository;
  final Logger logger;

  Future<User?> getUserById(int id) async {
    logger.log('Getting user by id: $id');
    final user = await repository.findById(id);
    if (user == null) {
      logger.error('User not found', id);
    }
    return user;
  }

  Future<User> createUser({
    required String firstName,
    required String lastName,
    required String email,
  }) async {
    logger.log('Creating user: $email');
    final user = User(
      id: 0,
      firstName: firstName,
      lastName: lastName,
      email: email,
      createdAt: DateTime.now(),
    );
    return repository.save(user);
  }
}

// Setup DI
void setupServices() {
  final locator = ServiceLocator();

  locator.register<Logger>(ConsoleLogger());
  locator.register<UserRepository>(InMemoryUserRepository());
  locator.registerFactory<UserService>(() => UserService(
        repository: locator.resolve<UserRepository>(),
        logger: locator.resolve<Logger>(),
      ));
}

// ใช้งาน
Future<void> runMetaProgrammingDemo() async {
  setupServices();

  final userService = ServiceLocator().resolve<UserService>();

  final user = await userService.createUser(
    firstName: 'Alice',
    lastName: 'Smith',
    email: 'alice@example.com',
  );
  print('Created: $user');

  final found = await userService.getUserById(user.id);
  print('Found: $found');

  demonstrateJsonSerialization();
}

void main() async {
  await runMetaProgrammingDemo();
}
```

---

**← [Part 76](part-76-flutter-rendering-engine.md)**
**ต่อไป: [Part 78 →](part-78-flutter-state-restoration.md)**
