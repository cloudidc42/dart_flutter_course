# Part 91: Capstone Project Planning – Food Delivery App
## ขั้นตอนที่ 3521-3560

## 🎯 เป้าหมายของ Part นี้
- วางแผนสถาปัตยกรรมของ Full-featured Food Delivery App
- ออกแบบ Data Models ครบทุก Entity
- กำหนด Database Schema สำหรับ Firestore
- ออกแบบ REST API endpoints
- สร้าง Dart model classes พร้อม serialization

---

## ขั้นตอนที่ 3521: Architecture Decision Document

```dart
// ============================================================
// ARCHITECTURE DECISION DOCUMENT
// Food Delivery App – "QuickBite"
// ============================================================
//
// TECH STACK
// ----------
// Frontend  : Flutter 3.x (Dart 3.x)
// State Mgmt: Riverpod 2.x
// Backend   : Firebase (Auth, Firestore, Storage, Functions)
// Maps      : Google Maps Flutter + Geolocator
// Payments  : Stripe SDK for Flutter
// Push Notif: Firebase Cloud Messaging
// Analytics : Firebase Analytics + Crashlytics
//
// CLEAN ARCHITECTURE LAYERS
// -------------------------
// Presentation  → Pages / Widgets / State Notifiers
// Domain        → Entities / Use Cases / Repository Interfaces
// Data          → Repository Implementations / Data Sources / DTOs
//
// FOLDER STRUCTURE
// ----------------
// lib/
//   core/
//     error/       → failures.dart, exceptions.dart
//     network/     → dio_client.dart, interceptors.dart
//     utils/       → validators.dart, formatters.dart
//     constants/   → app_colors.dart, app_routes.dart
//   features/
//     auth/
//       data/      → auth_repository_impl.dart, auth_remote_ds.dart
//       domain/    → auth_repository.dart, login_usecase.dart
//       presentation/ → login_page.dart, auth_notifier.dart
//     restaurant/
//     cart/
//     order/
//     profile/
//   shared/
//     widgets/     → custom_button.dart, loading_widget.dart
//     models/      → base_model.dart
//
// STATE MANAGEMENT PATTERN
// ------------------------
// AsyncNotifier  → for data-fetching notifiers
// StateNotifier  → for complex state machines (cart, order)
// Provider       → for simple derived state
//
// ============================================================

void main() {
  // Entry point – see lib/main.dart in the full project
  print('QuickBite Architecture defined.');
}
```

---

## ขั้นตอนที่ 3522: Core Failure & Exception Types

```dart
// lib/core/error/failures.dart

abstract class Failure {
  final String message;
  final int? code;
  const Failure({required this.message, this.code});

  @override
  String toString() => 'Failure(code: $code, message: $message)';
}

class ServerFailure extends Failure {
  const ServerFailure({required super.message, super.code});
}

class NetworkFailure extends Failure {
  const NetworkFailure({required super.message, super.code});
}

class CacheFailure extends Failure {
  const CacheFailure({required super.message, super.code});
}

class AuthFailure extends Failure {
  const AuthFailure({required super.message, super.code});
}

class NotFoundFailure extends Failure {
  const NotFoundFailure({required super.message, super.code});
}

class PermissionFailure extends Failure {
  const PermissionFailure({required super.message, super.code});
}

// lib/core/error/exceptions.dart

class ServerException implements Exception {
  final String message;
  final int? statusCode;
  const ServerException({required this.message, this.statusCode});
  @override
  String toString() => 'ServerException($statusCode): $message';
}

class CacheException implements Exception {
  final String message;
  const CacheException({required this.message});
}

class NetworkException implements Exception {
  final String message;
  const NetworkException({required this.message});
}
```

---

## ขั้นตอนที่ 3523: User Data Model

```dart
// lib/features/auth/domain/entities/user_entity.dart

import 'package:equatable/equatable.dart';

enum UserRole { customer, driver, restaurantOwner, admin }

class UserEntity extends Equatable {
  final String id;
  final String email;
  final String displayName;
  final String? photoUrl;
  final String? phoneNumber;
  final UserRole role;
  final bool isEmailVerified;
  final bool isPhoneVerified;
  final DateTime createdAt;
  final DateTime updatedAt;

  const UserEntity({
    required this.id,
    required this.email,
    required this.displayName,
    this.photoUrl,
    this.phoneNumber,
    this.role = UserRole.customer,
    this.isEmailVerified = false,
    this.isPhoneVerified = false,
    required this.createdAt,
    required this.updatedAt,
  });

  UserEntity copyWith({
    String? id,
    String? email,
    String? displayName,
    String? photoUrl,
    String? phoneNumber,
    UserRole? role,
    bool? isEmailVerified,
    bool? isPhoneVerified,
    DateTime? createdAt,
    DateTime? updatedAt,
  }) {
    return UserEntity(
      id: id ?? this.id,
      email: email ?? this.email,
      displayName: displayName ?? this.displayName,
      photoUrl: photoUrl ?? this.photoUrl,
      phoneNumber: phoneNumber ?? this.phoneNumber,
      role: role ?? this.role,
      isEmailVerified: isEmailVerified ?? this.isEmailVerified,
      isPhoneVerified: isPhoneVerified ?? this.isPhoneVerified,
      createdAt: createdAt ?? this.createdAt,
      updatedAt: updatedAt ?? this.updatedAt,
    );
  }

  @override
  List<Object?> get props => [
        id, email, displayName, photoUrl,
        phoneNumber, role, isEmailVerified,
        isPhoneVerified, createdAt, updatedAt,
      ];
}

// lib/features/auth/data/models/user_model.dart

class UserModel extends UserEntity {
  const UserModel({
    required super.id,
    required super.email,
    required super.displayName,
    super.photoUrl,
    super.phoneNumber,
    super.role,
    super.isEmailVerified,
    super.isPhoneVerified,
    required super.createdAt,
    required super.updatedAt,
  });

  factory UserModel.fromJson(Map<String, dynamic> json) {
    return UserModel(
      id: json['id'] as String,
      email: json['email'] as String,
      displayName: json['display_name'] as String,
      photoUrl: json['photo_url'] as String?,
      phoneNumber: json['phone_number'] as String?,
      role: UserRole.values.firstWhere(
        (r) => r.name == (json['role'] as String? ?? 'customer'),
        orElse: () => UserRole.customer,
      ),
      isEmailVerified: json['is_email_verified'] as bool? ?? false,
      isPhoneVerified: json['is_phone_verified'] as bool? ?? false,
      createdAt: DateTime.parse(json['created_at'] as String),
      updatedAt: DateTime.parse(json['updated_at'] as String),
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'email': email,
      'display_name': displayName,
      'photo_url': photoUrl,
      'phone_number': phoneNumber,
      'role': role.name,
      'is_email_verified': isEmailVerified,
      'is_phone_verified': isPhoneVerified,
      'created_at': createdAt.toIso8601String(),
      'updated_at': updatedAt.toIso8601String(),
    };
  }

  factory UserModel.fromFirestore(Map<String, dynamic> data, String docId) {
    return UserModel(
      id: docId,
      email: data['email'] as String,
      displayName: data['displayName'] as String,
      photoUrl: data['photoUrl'] as String?,
      phoneNumber: data['phoneNumber'] as String?,
      role: UserRole.values.firstWhere(
        (r) => r.name == (data['role'] as String? ?? 'customer'),
        orElse: () => UserRole.customer,
      ),
      isEmailVerified: data['isEmailVerified'] as bool? ?? false,
      isPhoneVerified: data['isPhoneVerified'] as bool? ?? false,
      createdAt: (data['createdAt'] as dynamic).toDate() as DateTime,
      updatedAt: (data['updatedAt'] as dynamic).toDate() as DateTime,
    );
  }

  Map<String, dynamic> toFirestore() {
    return {
      'email': email,
      'displayName': displayName,
      'photoUrl': photoUrl,
      'phoneNumber': phoneNumber,
      'role': role.name,
      'isEmailVerified': isEmailVerified,
      'isPhoneVerified': isPhoneVerified,
      'createdAt': createdAt,
      'updatedAt': updatedAt,
    };
  }
}
```

---

## ขั้นตอนที่ 3524: Restaurant Data Model

```dart
// lib/features/restaurant/domain/entities/restaurant_entity.dart

import 'package:equatable/equatable.dart';

class GeoPoint extends Equatable {
  final double latitude;
  final double longitude;
  const GeoPoint({required this.latitude, required this.longitude});

  @override
  List<Object?> get props => [latitude, longitude];
}

class OperatingHours extends Equatable {
  final String open;   // "08:00"
  final String close;  // "22:00"
  final bool isOpen;
  const OperatingHours({
    required this.open,
    required this.close,
    required this.isOpen,
  });

  factory OperatingHours.fromJson(Map<String, dynamic> json) {
    return OperatingHours(
      open: json['open'] as String,
      close: json['close'] as String,
      isOpen: json['is_open'] as bool? ?? true,
    );
  }

  Map<String, dynamic> toJson() =>
      {'open': open, 'close': close, 'is_open': isOpen};

  @override
  List<Object?> get props => [open, close, isOpen];
}

class RestaurantEntity extends Equatable {
  final String id;
  final String ownerId;
  final String name;
  final String description;
  final String imageUrl;
  final List<String> bannerUrls;
  final String address;
  final GeoPoint location;
  final List<String> cuisineTypes;
  final double rating;
  final int totalReviews;
  final double deliveryFee;
  final int deliveryTimeMinutes;
  final double minimumOrder;
  final bool isOpen;
  final bool isActive;
  final Map<String, OperatingHours> operatingHours; // key: 'mon','tue'…
  final DateTime createdAt;

  const RestaurantEntity({
    required this.id,
    required this.ownerId,
    required this.name,
    required this.description,
    required this.imageUrl,
    this.bannerUrls = const [],
    required this.address,
    required this.location,
    this.cuisineTypes = const [],
    this.rating = 0.0,
    this.totalReviews = 0,
    this.deliveryFee = 0.0,
    this.deliveryTimeMinutes = 30,
    this.minimumOrder = 0.0,
    this.isOpen = true,
    this.isActive = true,
    this.operatingHours = const {},
    required this.createdAt,
  });

  @override
  List<Object?> get props => [id, name, rating, isOpen];
}

// lib/features/restaurant/data/models/restaurant_model.dart

class RestaurantModel extends RestaurantEntity {
  const RestaurantModel({
    required super.id,
    required super.ownerId,
    required super.name,
    required super.description,
    required super.imageUrl,
    super.bannerUrls,
    required super.address,
    required super.location,
    super.cuisineTypes,
    super.rating,
    super.totalReviews,
    super.deliveryFee,
    super.deliveryTimeMinutes,
    super.minimumOrder,
    super.isOpen,
    super.isActive,
    super.operatingHours,
    required super.createdAt,
  });

  factory RestaurantModel.fromFirestore(
      Map<String, dynamic> data, String docId) {
    final loc = data['location'] as Map<String, dynamic>;
    final hours = <String, OperatingHours>{};
    if (data['operatingHours'] != null) {
      (data['operatingHours'] as Map<String, dynamic>).forEach((k, v) {
        hours[k] = OperatingHours.fromJson(v as Map<String, dynamic>);
      });
    }
    return RestaurantModel(
      id: docId,
      ownerId: data['ownerId'] as String,
      name: data['name'] as String,
      description: data['description'] as String? ?? '',
      imageUrl: data['imageUrl'] as String? ?? '',
      bannerUrls: List<String>.from(data['bannerUrls'] as List? ?? []),
      address: data['address'] as String,
      location: GeoPoint(
        latitude: (loc['latitude'] as num).toDouble(),
        longitude: (loc['longitude'] as num).toDouble(),
      ),
      cuisineTypes: List<String>.from(data['cuisineTypes'] as List? ?? []),
      rating: (data['rating'] as num?)?.toDouble() ?? 0.0,
      totalReviews: data['totalReviews'] as int? ?? 0,
      deliveryFee: (data['deliveryFee'] as num?)?.toDouble() ?? 0.0,
      deliveryTimeMinutes: data['deliveryTimeMinutes'] as int? ?? 30,
      minimumOrder: (data['minimumOrder'] as num?)?.toDouble() ?? 0.0,
      isOpen: data['isOpen'] as bool? ?? true,
      isActive: data['isActive'] as bool? ?? true,
      operatingHours: hours,
      createdAt: (data['createdAt'] as dynamic).toDate() as DateTime,
    );
  }

  Map<String, dynamic> toFirestore() {
    return {
      'ownerId': ownerId,
      'name': name,
      'description': description,
      'imageUrl': imageUrl,
      'bannerUrls': bannerUrls,
      'address': address,
      'location': {
        'latitude': location.latitude,
        'longitude': location.longitude,
      },
      'cuisineTypes': cuisineTypes,
      'rating': rating,
      'totalReviews': totalReviews,
      'deliveryFee': deliveryFee,
      'deliveryTimeMinutes': deliveryTimeMinutes,
      'minimumOrder': minimumOrder,
      'isOpen': isOpen,
      'isActive': isActive,
      'operatingHours': operatingHours.map(
        (k, v) => MapEntry(k, v.toJson()),
      ),
      'createdAt': createdAt,
    };
  }
}
```

---

## ขั้นตอนที่ 3525: Menu & Menu Item Models

```dart
// lib/features/restaurant/domain/entities/menu_entity.dart

import 'package:equatable/equatable.dart';

class MenuOptionChoice extends Equatable {
  final String id;
  final String name;
  final double extraPrice;
  const MenuOptionChoice({
    required this.id,
    required this.name,
    this.extraPrice = 0.0,
  });

  factory MenuOptionChoice.fromJson(Map<String, dynamic> json) =>
      MenuOptionChoice(
        id: json['id'] as String,
        name: json['name'] as String,
        extraPrice: (json['extra_price'] as num?)?.toDouble() ?? 0.0,
      );

  Map<String, dynamic> toJson() =>
      {'id': id, 'name': name, 'extra_price': extraPrice};

  @override
  List<Object?> get props => [id, name, extraPrice];
}

class MenuOption extends Equatable {
  final String id;
  final String title;
  final bool isRequired;
  final bool isMultiSelect;
  final int maxSelections;
  final List<MenuOptionChoice> choices;

  const MenuOption({
    required this.id,
    required this.title,
    this.isRequired = false,
    this.isMultiSelect = false,
    this.maxSelections = 1,
    required this.choices,
  });

  factory MenuOption.fromJson(Map<String, dynamic> json) => MenuOption(
        id: json['id'] as String,
        title: json['title'] as String,
        isRequired: json['is_required'] as bool? ?? false,
        isMultiSelect: json['is_multi_select'] as bool? ?? false,
        maxSelections: json['max_selections'] as int? ?? 1,
        choices: (json['choices'] as List)
            .map((c) => MenuOptionChoice.fromJson(c as Map<String, dynamic>))
            .toList(),
      );

  Map<String, dynamic> toJson() => {
        'id': id,
        'title': title,
        'is_required': isRequired,
        'is_multi_select': isMultiSelect,
        'max_selections': maxSelections,
        'choices': choices.map((c) => c.toJson()).toList(),
      };

  @override
  List<Object?> get props => [id, title, choices];
}

class MenuItemEntity extends Equatable {
  final String id;
  final String restaurantId;
  final String categoryId;
  final String name;
  final String description;
  final String imageUrl;
  final double price;
  final bool isAvailable;
  final bool isPopular;
  final bool isVegetarian;
  final bool isSpicy;
  final int calories;
  final List<MenuOption> options;
  final int sortOrder;

  const MenuItemEntity({
    required this.id,
    required this.restaurantId,
    required this.categoryId,
    required this.name,
    required this.description,
    required this.imageUrl,
    required this.price,
    this.isAvailable = true,
    this.isPopular = false,
    this.isVegetarian = false,
    this.isSpicy = false,
    this.calories = 0,
    this.options = const [],
    this.sortOrder = 0,
  });

  @override
  List<Object?> get props => [id, name, price, isAvailable];
}

class MenuCategoryEntity extends Equatable {
  final String id;
  final String restaurantId;
  final String name;
  final String? imageUrl;
  final int sortOrder;
  final List<MenuItemEntity> items;

  const MenuCategoryEntity({
    required this.id,
    required this.restaurantId,
    required this.name,
    this.imageUrl,
    this.sortOrder = 0,
    this.items = const [],
  });

  @override
  List<Object?> get props => [id, name, restaurantId];
}
```

---

## ขั้นตอนที่ 3526: Order Data Model

```dart
// lib/features/order/domain/entities/order_entity.dart

import 'package:equatable/equatable.dart';

enum OrderStatus {
  pending,
  confirmed,
  preparing,
  readyForPickup,
  driverAssigned,
  pickedUp,
  onTheWay,
  delivered,
  cancelled,
  refunded,
}

enum PaymentMethod { creditCard, debitCard, promptPay, cash, wallet }

enum PaymentStatus { pending, paid, failed, refunded }

class OrderItemCustomization extends Equatable {
  final String optionId;
  final String optionTitle;
  final List<String> choiceIds;
  final List<String> choiceNames;
  final double extraPrice;

  const OrderItemCustomization({
    required this.optionId,
    required this.optionTitle,
    required this.choiceIds,
    required this.choiceNames,
    required this.extraPrice,
  });

  factory OrderItemCustomization.fromJson(Map<String, dynamic> json) {
    return OrderItemCustomization(
      optionId: json['option_id'] as String,
      optionTitle: json['option_title'] as String,
      choiceIds: List<String>.from(json['choice_ids'] as List),
      choiceNames: List<String>.from(json['choice_names'] as List),
      extraPrice: (json['extra_price'] as num).toDouble(),
    );
  }

  Map<String, dynamic> toJson() => {
        'option_id': optionId,
        'option_title': optionTitle,
        'choice_ids': choiceIds,
        'choice_names': choiceNames,
        'extra_price': extraPrice,
      };

  @override
  List<Object?> get props => [optionId, choiceIds];
}

class OrderItem extends Equatable {
  final String menuItemId;
  final String menuItemName;
  final String menuItemImageUrl;
  final double unitPrice;
  final int quantity;
  final List<OrderItemCustomization> customizations;
  final String? specialInstructions;
  final double totalPrice;

  const OrderItem({
    required this.menuItemId,
    required this.menuItemName,
    required this.menuItemImageUrl,
    required this.unitPrice,
    required this.quantity,
    this.customizations = const [],
    this.specialInstructions,
    required this.totalPrice,
  });

  factory OrderItem.fromJson(Map<String, dynamic> json) {
    return OrderItem(
      menuItemId: json['menu_item_id'] as String,
      menuItemName: json['menu_item_name'] as String,
      menuItemImageUrl: json['menu_item_image_url'] as String? ?? '',
      unitPrice: (json['unit_price'] as num).toDouble(),
      quantity: json['quantity'] as int,
      customizations: (json['customizations'] as List? ?? [])
          .map((c) =>
              OrderItemCustomization.fromJson(c as Map<String, dynamic>))
          .toList(),
      specialInstructions: json['special_instructions'] as String?,
      totalPrice: (json['total_price'] as num).toDouble(),
    );
  }

  Map<String, dynamic> toJson() => {
        'menu_item_id': menuItemId,
        'menu_item_name': menuItemName,
        'menu_item_image_url': menuItemImageUrl,
        'unit_price': unitPrice,
        'quantity': quantity,
        'customizations': customizations.map((c) => c.toJson()).toList(),
        'special_instructions': specialInstructions,
        'total_price': totalPrice,
      };

  @override
  List<Object?> get props => [menuItemId, quantity, customizations];
}

class DeliveryAddress extends Equatable {
  final String label;        // "Home", "Work", "Other"
  final String fullAddress;
  final String subdistrict;
  final String district;
  final String province;
  final String postalCode;
  final double latitude;
  final double longitude;
  final String? instructions; // "Ring the bell"

  const DeliveryAddress({
    required this.label,
    required this.fullAddress,
    required this.subdistrict,
    required this.district,
    required this.province,
    required this.postalCode,
    required this.latitude,
    required this.longitude,
    this.instructions,
  });

  factory DeliveryAddress.fromJson(Map<String, dynamic> json) {
    return DeliveryAddress(
      label: json['label'] as String,
      fullAddress: json['full_address'] as String,
      subdistrict: json['subdistrict'] as String,
      district: json['district'] as String,
      province: json['province'] as String,
      postalCode: json['postal_code'] as String,
      latitude: (json['latitude'] as num).toDouble(),
      longitude: (json['longitude'] as num).toDouble(),
      instructions: json['instructions'] as String?,
    );
  }

  Map<String, dynamic> toJson() => {
        'label': label,
        'full_address': fullAddress,
        'subdistrict': subdistrict,
        'district': district,
        'province': province,
        'postal_code': postalCode,
        'latitude': latitude,
        'longitude': longitude,
        'instructions': instructions,
      };

  @override
  List<Object?> get props => [fullAddress, latitude, longitude];
}

class OrderEntity extends Equatable {
  final String id;
  final String customerId;
  final String restaurantId;
  final String restaurantName;
  final String? driverId;
  final List<OrderItem> items;
  final DeliveryAddress deliveryAddress;
  final OrderStatus status;
  final PaymentMethod paymentMethod;
  final PaymentStatus paymentStatus;
  final double subtotal;
  final double deliveryFee;
  final double discount;
  final double tax;
  final double total;
  final String? promoCode;
  final String? specialInstructions;
  final DateTime? estimatedDeliveryTime;
  final DateTime createdAt;
  final DateTime updatedAt;
  final List<OrderStatusUpdate> statusHistory;

  const OrderEntity({
    required this.id,
    required this.customerId,
    required this.restaurantId,
    required this.restaurantName,
    this.driverId,
    required this.items,
    required this.deliveryAddress,
    required this.status,
    required this.paymentMethod,
    required this.paymentStatus,
    required this.subtotal,
    required this.deliveryFee,
    this.discount = 0,
    this.tax = 0,
    required this.total,
    this.promoCode,
    this.specialInstructions,
    this.estimatedDeliveryTime,
    required this.createdAt,
    required this.updatedAt,
    this.statusHistory = const [],
  });

  @override
  List<Object?> get props => [id, status, customerId];
}

class OrderStatusUpdate extends Equatable {
  final OrderStatus status;
  final DateTime timestamp;
  final String? note;

  const OrderStatusUpdate({
    required this.status,
    required this.timestamp,
    this.note,
  });

  factory OrderStatusUpdate.fromJson(Map<String, dynamic> json) {
    return OrderStatusUpdate(
      status: OrderStatus.values.firstWhere(
        (s) => s.name == json['status'],
        orElse: () => OrderStatus.pending,
      ),
      timestamp: DateTime.parse(json['timestamp'] as String),
      note: json['note'] as String?,
    );
  }

  Map<String, dynamic> toJson() => {
        'status': status.name,
        'timestamp': timestamp.toIso8601String(),
        'note': note,
      };

  @override
  List<Object?> get props => [status, timestamp];
}
```

---

## ขั้นตอนที่ 3527: Driver Data Model

```dart
// lib/features/driver/domain/entities/driver_entity.dart

import 'package:equatable/equatable.dart';

enum DriverStatus { offline, available, busy }

enum VehicleType { bicycle, motorbike, car }

class VehicleInfo extends Equatable {
  final VehicleType type;
  final String licensePlate;
  final String brand;
  final String model;
  final String color;

  const VehicleInfo({
    required this.type,
    required this.licensePlate,
    required this.brand,
    required this.model,
    required this.color,
  });

  factory VehicleInfo.fromJson(Map<String, dynamic> json) => VehicleInfo(
        type: VehicleType.values.firstWhere(
          (t) => t.name == json['type'],
          orElse: () => VehicleType.motorbike,
        ),
        licensePlate: json['license_plate'] as String,
        brand: json['brand'] as String,
        model: json['model'] as String,
        color: json['color'] as String,
      );

  Map<String, dynamic> toJson() => {
        'type': type.name,
        'license_plate': licensePlate,
        'brand': brand,
        'model': model,
        'color': color,
      };

  @override
  List<Object?> get props => [licensePlate, type];
}

class DriverEntity extends Equatable {
  final String id;
  final String userId;
  final String displayName;
  final String phoneNumber;
  final String? photoUrl;
  final VehicleInfo vehicle;
  final DriverStatus status;
  final double currentLatitude;
  final double currentLongitude;
  final double rating;
  final int totalDeliveries;
  final bool isVerified;
  final DateTime createdAt;

  const DriverEntity({
    required this.id,
    required this.userId,
    required this.displayName,
    required this.phoneNumber,
    this.photoUrl,
    required this.vehicle,
    this.status = DriverStatus.offline,
    this.currentLatitude = 0.0,
    this.currentLongitude = 0.0,
    this.rating = 5.0,
    this.totalDeliveries = 0,
    this.isVerified = false,
    required this.createdAt,
  });

  DriverEntity copyWith({
    DriverStatus? status,
    double? currentLatitude,
    double? currentLongitude,
  }) {
    return DriverEntity(
      id: id,
      userId: userId,
      displayName: displayName,
      phoneNumber: phoneNumber,
      photoUrl: photoUrl,
      vehicle: vehicle,
      status: status ?? this.status,
      currentLatitude: currentLatitude ?? this.currentLatitude,
      currentLongitude: currentLongitude ?? this.currentLongitude,
      rating: rating,
      totalDeliveries: totalDeliveries,
      isVerified: isVerified,
      createdAt: createdAt,
    );
  }

  @override
  List<Object?> get props => [id, userId, status];
}
```

---

## ขั้นตอนที่ 3528: Firestore Database Schema

```dart
// ============================================================
// FIRESTORE COLLECTIONS SCHEMA
// ============================================================
//
// /users/{userId}
//   - email: string
//   - displayName: string
//   - photoUrl: string?
//   - phoneNumber: string?
//   - role: string ('customer'|'driver'|'restaurantOwner'|'admin')
//   - isEmailVerified: boolean
//   - isPhoneVerified: boolean
//   - fcmToken: string?
//   - createdAt: timestamp
//   - updatedAt: timestamp
//
//   /users/{userId}/addresses/{addressId}
//     - label: string
//     - fullAddress: string
//     - subdistrict: string
//     - district: string
//     - province: string
//     - postalCode: string
//     - latitude: number
//     - longitude: number
//     - instructions: string?
//     - isDefault: boolean
//
// /restaurants/{restaurantId}
//   - ownerId: string (ref → users)
//   - name: string
//   - description: string
//   - imageUrl: string
//   - bannerUrls: array<string>
//   - address: string
//   - location: geopoint
//   - cuisineTypes: array<string>
//   - rating: number
//   - totalReviews: number
//   - deliveryFee: number
//   - deliveryTimeMinutes: number
//   - minimumOrder: number
//   - isOpen: boolean
//   - isActive: boolean
//   - operatingHours: map
//   - createdAt: timestamp
//
//   /restaurants/{restaurantId}/categories/{categoryId}
//     - name: string
//     - imageUrl: string?
//     - sortOrder: number
//     - isActive: boolean
//
//   /restaurants/{restaurantId}/menuItems/{itemId}
//     - categoryId: string
//     - name: string
//     - description: string
//     - imageUrl: string
//     - price: number
//     - isAvailable: boolean
//     - isPopular: boolean
//     - isVegetarian: boolean
//     - isSpicy: boolean
//     - calories: number
//     - options: array<MenuOption>
//     - sortOrder: number
//
//   /restaurants/{restaurantId}/reviews/{reviewId}
//     - userId: string
//     - orderId: string
//     - rating: number (1-5)
//     - comment: string
//     - images: array<string>
//     - createdAt: timestamp
//
// /orders/{orderId}
//   - customerId: string
//   - restaurantId: string
//   - restaurantName: string
//   - driverId: string?
//   - items: array<OrderItem>
//   - deliveryAddress: map
//   - status: string
//   - paymentMethod: string
//   - paymentStatus: string
//   - subtotal: number
//   - deliveryFee: number
//   - discount: number
//   - tax: number
//   - total: number
//   - promoCode: string?
//   - specialInstructions: string?
//   - estimatedDeliveryTime: timestamp?
//   - createdAt: timestamp
//   - updatedAt: timestamp
//   - statusHistory: array<StatusUpdate>
//
// /drivers/{driverId}
//   - userId: string
//   - displayName: string
//   - phoneNumber: string
//   - photoUrl: string?
//   - vehicle: map
//   - status: string ('offline'|'available'|'busy')
//   - currentLocation: geopoint
//   - rating: number
//   - totalDeliveries: number
//   - isVerified: boolean
//   - createdAt: timestamp
//
// /promoCodes/{code}
//   - type: string ('percentage'|'fixed')
//   - value: number
//   - minOrderAmount: number
//   - maxDiscountAmount: number?
//   - usageLimit: number
//   - usedCount: number
//   - validFrom: timestamp
//   - validUntil: timestamp
//   - isActive: boolean
// ============================================================

class FirestoreCollections {
  static const String users = 'users';
  static const String addresses = 'addresses';
  static const String restaurants = 'restaurants';
  static const String categories = 'categories';
  static const String menuItems = 'menuItems';
  static const String reviews = 'reviews';
  static const String orders = 'orders';
  static const String drivers = 'drivers';
  static const String promoCodes = 'promoCodes';
  static const String notifications = 'notifications';
}
```

---

## ขั้นตอนที่ 3529: API Design (REST Endpoints)

```dart
// lib/core/network/api_endpoints.dart

class ApiEndpoints {
  static const String baseUrl = 'https://api.quickbite.app/v1';

  // Auth
  static const String register        = '/auth/register';
  static const String login           = '/auth/login';
  static const String googleLogin     = '/auth/google';
  static const String appleLogin      = '/auth/apple';
  static const String phoneOtp        = '/auth/phone/send-otp';
  static const String verifyOtp       = '/auth/phone/verify';
  static const String forgotPassword  = '/auth/forgot-password';
  static const String refreshToken    = '/auth/refresh';
  static const String logout          = '/auth/logout';

  // Users
  static const String profile         = '/users/me';
  static const String updateProfile   = '/users/me';
  static const String userAddresses   = '/users/me/addresses';
  static String userAddress(String id)  => '/users/me/addresses/$id';

  // Restaurants
  static const String restaurants     = '/restaurants';
  static String restaurant(String id) => '/restaurants/$id';
  static String restaurantMenu(String id) => '/restaurants/$id/menu';
  static String restaurantReviews(String id) => '/restaurants/$id/reviews';
  static const String nearbyRestaurants = '/restaurants/nearby';
  static const String featuredRestaurants = '/restaurants/featured';

  // Menu
  static String menuItem(String restId, String itemId) =>
      '/restaurants/$restId/menu/$itemId';

  // Cart (client-side only, no API needed)

  // Orders
  static const String orders          = '/orders';
  static String order(String id)      => '/orders/$id';
  static String cancelOrder(String id) => '/orders/$id/cancel';
  static String rateOrder(String id)  => '/orders/$id/rate';
  static const String orderHistory    = '/orders/history';

  // Promo
  static const String validatePromo   = '/promo/validate';

  // Payments
  static const String createPaymentIntent = '/payments/intent';
  static const String confirmPayment  = '/payments/confirm';

  // Notifications
  static const String fcmToken        = '/notifications/fcm-token';
  static const String notificationList = '/notifications';
}
```

---

## ขั้นตอนที่ 3530: PromoCode Model & Repository Interface

```dart
// lib/features/promo/domain/entities/promo_code_entity.dart

import 'package:equatable/equatable.dart';

enum PromoType { percentage, fixed }

class PromoCodeEntity extends Equatable {
  final String code;
  final PromoType type;
  final double value;
  final double minOrderAmount;
  final double? maxDiscountAmount;
  final int usageLimit;
  final int usedCount;
  final DateTime validFrom;
  final DateTime validUntil;
  final bool isActive;

  const PromoCodeEntity({
    required this.code,
    required this.type,
    required this.value,
    required this.minOrderAmount,
    this.maxDiscountAmount,
    required this.usageLimit,
    required this.usedCount,
    required this.validFrom,
    required this.validUntil,
    required this.isActive,
  });

  bool get isValid {
    final now = DateTime.now();
    return isActive &&
        usedCount < usageLimit &&
        now.isAfter(validFrom) &&
        now.isBefore(validUntil);
  }

  double calculateDiscount(double orderAmount) {
    if (!isValid || orderAmount < minOrderAmount) return 0;
    double discount = type == PromoType.percentage
        ? orderAmount * value / 100
        : value;
    if (maxDiscountAmount != null) {
      discount = discount.clamp(0, maxDiscountAmount!);
    }
    return discount;
  }

  @override
  List<Object?> get props => [code, type, value, isActive];
}

// lib/features/restaurant/domain/repositories/restaurant_repository.dart
// (interface only – implementation in data layer)

abstract class RestaurantRepository {
  Future<List<RestaurantEntity>> getNearbyRestaurants({
    required double latitude,
    required double longitude,
    double radiusKm = 10,
    String? cuisineFilter,
  });

  Future<RestaurantEntity> getRestaurantById(String id);

  Future<List<MenuCategoryEntity>> getRestaurantMenu(String restaurantId);

  Future<List<RestaurantEntity>> searchRestaurants(String query);

  Future<void> toggleFavorite(String userId, String restaurantId);

  Future<List<String>> getFavoriteIds(String userId);

  Stream<RestaurantEntity> watchRestaurant(String restaurantId);
}
```

---

## ขั้นตอนที่ 3531: Notification Model

```dart
// lib/features/notification/domain/entities/notification_entity.dart

import 'package:equatable/equatable.dart';

enum NotificationType {
  orderConfirmed,
  orderPreparing,
  driverAssigned,
  orderPickedUp,
  orderDelivered,
  orderCancelled,
  promoOffer,
  systemMessage,
}

class NotificationEntity extends Equatable {
  final String id;
  final String userId;
  final NotificationType type;
  final String title;
  final String body;
  final Map<String, dynamic> data;
  final bool isRead;
  final DateTime createdAt;

  const NotificationEntity({
    required this.id,
    required this.userId,
    required this.type,
    required this.title,
    required this.body,
    this.data = const {},
    this.isRead = false,
    required this.createdAt,
  });

  NotificationEntity copyWith({bool? isRead}) => NotificationEntity(
        id: id,
        userId: userId,
        type: type,
        title: title,
        body: body,
        data: data,
        isRead: isRead ?? this.isRead,
        createdAt: createdAt,
      );

  factory NotificationEntity.fromFirestore(
      Map<String, dynamic> data, String docId) {
    return NotificationEntity(
      id: docId,
      userId: data['userId'] as String,
      type: NotificationType.values.firstWhere(
        (t) => t.name == data['type'],
        orElse: () => NotificationType.systemMessage,
      ),
      title: data['title'] as String,
      body: data['body'] as String,
      data: Map<String, dynamic>.from(data['data'] as Map? ?? {}),
      isRead: data['isRead'] as bool? ?? false,
      createdAt: (data['createdAt'] as dynamic).toDate() as DateTime,
    );
  }

  Map<String, dynamic> toFirestore() => {
        'userId': userId,
        'type': type.name,
        'title': title,
        'body': body,
        'data': data,
        'isRead': isRead,
        'createdAt': createdAt,
      };

  @override
  List<Object?> get props => [id, type, isRead];
}
```

---

**← [Part 90](part-90-advanced-firebase.md)**
**ต่อไป: [Part 92 →](part-92-capstone-auth-module.md)**
