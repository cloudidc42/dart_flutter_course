# Part 95: Capstone – Order Tracking
## ขั้นตอนที่ 3681-3720

## 🎯 เป้าหมายของ Part นี้
- Real-time order status ด้วย Firestore streams
- Google Maps tracking screen
- Driver location updates (Firestore + GeoFlutterFire)
- Push notifications สำหรับ order updates
- Order history
- Full working Flutter + Firebase code

---

## ขั้นตอนที่ 3681: Order Repository

```dart
// lib/features/order/data/repositories/order_repository_impl.dart

import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:dartz/dartz.dart';
import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class OrderRepositoryImpl {
  final FirebaseFirestore _firestore;

  OrderRepositoryImpl({required FirebaseFirestore firestore})
      : _firestore = firestore;

  /// Place a new order – writes the order document atomically
  Future<Either<Failure, OrderEntity>> placeOrder({
    required CartState cart,
    required String customerId,
    required PaymentMethod paymentMethod,
    required String? paymentIntentId,
  }) async {
    if (cart.deliveryAddress == null) {
      return const Left(
          ServerFailure(message: 'กรุณาเลือกที่อยู่จัดส่ง'));
    }

    try {
      final now = DateTime.now();
      final ref = _firestore.collection(FirestoreCollections.orders).doc();

      final order = OrderEntity(
        id: ref.id,
        customerId: customerId,
        restaurantId: cart.restaurantId!,
        restaurantName: cart.restaurantName ?? '',
        items: cart.items.map((i) => i.toOrderItem()).toList(),
        deliveryAddress: cart.deliveryAddress!,
        status: OrderStatus.pending,
        paymentMethod: paymentMethod,
        paymentStatus: paymentIntentId != null
            ? PaymentStatus.paid
            : PaymentStatus.pending,
        subtotal: cart.subtotal,
        deliveryFee: cart.deliveryFee,
        discount: cart.discount,
        tax: cart.tax,
        total: cart.total,
        promoCode: cart.appliedPromo?.code,
        estimatedDeliveryTime: now.add(const Duration(minutes: 40)),
        createdAt: now,
        updatedAt: now,
        statusHistory: [
          OrderStatusUpdate(
              status: OrderStatus.pending, timestamp: now),
        ],
      );

      await ref.set(_orderToFirestore(order));
      return Right(order);
    } catch (e) {
      return Left(ServerFailure(message: e.toString()));
    }
  }

  Map<String, dynamic> _orderToFirestore(OrderEntity order) {
    return {
      'customerId': order.customerId,
      'restaurantId': order.restaurantId,
      'restaurantName': order.restaurantName,
      'driverId': order.driverId,
      'items': order.items.map((i) => i.toJson()).toList(),
      'deliveryAddress': order.deliveryAddress.toJson(),
      'status': order.status.name,
      'paymentMethod': order.paymentMethod.name,
      'paymentStatus': order.paymentStatus.name,
      'subtotal': order.subtotal,
      'deliveryFee': order.deliveryFee,
      'discount': order.discount,
      'tax': order.tax,
      'total': order.total,
      'promoCode': order.promoCode,
      'specialInstructions': order.specialInstructions,
      'estimatedDeliveryTime': order.estimatedDeliveryTime,
      'createdAt': FieldValue.serverTimestamp(),
      'updatedAt': FieldValue.serverTimestamp(),
      'statusHistory': order.statusHistory
          .map((s) => s.toJson())
          .toList(),
    };
  }

  /// Real-time stream of a single order
  Stream<OrderEntity> watchOrder(String orderId) {
    return _firestore
        .collection(FirestoreCollections.orders)
        .doc(orderId)
        .snapshots()
        .map((snap) {
      if (!snap.exists) throw Exception('Order not found');
      return _orderFromFirestore(snap.data()!, snap.id);
    });
  }

  OrderEntity _orderFromFirestore(
      Map<String, dynamic> data, String docId) {
    return OrderEntity(
      id: docId,
      customerId: data['customerId'] as String,
      restaurantId: data['restaurantId'] as String,
      restaurantName: data['restaurantName'] as String? ?? '',
      driverId: data['driverId'] as String?,
      items: (data['items'] as List)
          .map((i) => OrderItem.fromJson(i as Map<String, dynamic>))
          .toList(),
      deliveryAddress:
          DeliveryAddress.fromJson(data['deliveryAddress'] as Map<String, dynamic>),
      status: OrderStatus.values.firstWhere(
        (s) => s.name == data['status'],
        orElse: () => OrderStatus.pending,
      ),
      paymentMethod: PaymentMethod.values.firstWhere(
        (p) => p.name == data['paymentMethod'],
        orElse: () => PaymentMethod.cash,
      ),
      paymentStatus: PaymentStatus.values.firstWhere(
        (p) => p.name == data['paymentStatus'],
        orElse: () => PaymentStatus.pending,
      ),
      subtotal: (data['subtotal'] as num).toDouble(),
      deliveryFee: (data['deliveryFee'] as num).toDouble(),
      discount: (data['discount'] as num?)?.toDouble() ?? 0,
      tax: (data['tax'] as num?)?.toDouble() ?? 0,
      total: (data['total'] as num).toDouble(),
      promoCode: data['promoCode'] as String?,
      estimatedDeliveryTime:
          (data['estimatedDeliveryTime'] as dynamic?)?.toDate()
              as DateTime?,
      createdAt: (data['createdAt'] as dynamic).toDate() as DateTime,
      updatedAt: (data['updatedAt'] as dynamic).toDate() as DateTime,
      statusHistory: (data['statusHistory'] as List? ?? [])
          .map((s) =>
              OrderStatusUpdate.fromJson(s as Map<String, dynamic>))
          .toList(),
    );
  }

  /// Real-time stream of driver location for a given order
  Stream<DriverLocation?> watchDriverLocation(String orderId) {
    return _firestore
        .collection(FirestoreCollections.orders)
        .doc(orderId)
        .snapshots()
        .asyncMap((orderSnap) async {
      final driverId = orderSnap.data()?['driverId'] as String?;
      if (driverId == null) return null;

      final driverDoc = await _firestore
          .collection(FirestoreCollections.drivers)
          .doc(driverId)
          .get();
      if (!driverDoc.exists) return null;

      final d = driverDoc.data()!;
      final loc = d['currentLocation'] as GeoPoint?;
      if (loc == null) return null;

      return DriverLocation(
        driverId: driverId,
        displayName: d['displayName'] as String? ?? '',
        photoUrl: d['photoUrl'] as String?,
        phoneNumber: d['phoneNumber'] as String? ?? '',
        latitude: loc.latitude,
        longitude: loc.longitude,
      );
    });
  }

  /// Get order history for a user
  Future<List<OrderEntity>> getOrderHistory(String userId) async {
    final snap = await _firestore
        .collection(FirestoreCollections.orders)
        .where('customerId', isEqualTo: userId)
        .orderBy('createdAt', descending: true)
        .limit(20)
        .get();

    return snap.docs
        .map((doc) => _orderFromFirestore(doc.data(), doc.id))
        .toList();
  }

  /// Rate a completed order and create a restaurant review
  Future<Either<Failure, void>> rateOrder({
    required String orderId,
    required String restaurantId,
    required String userId,
    required String userDisplayName,
    required double rating,
    required String comment,
  }) async {
    try {
      final batch = _firestore.batch();

      // Create review
      final reviewRef = _firestore
          .collection(FirestoreCollections.restaurants)
          .doc(restaurantId)
          .collection(FirestoreCollections.reviews)
          .doc();
      batch.set(reviewRef, {
        'userId': userId,
        'userDisplayName': userDisplayName,
        'restaurantId': restaurantId,
        'orderId': orderId,
        'rating': rating,
        'comment': comment,
        'images': [],
        'createdAt': FieldValue.serverTimestamp(),
      });

      // Update restaurant aggregate (in real app use Cloud Function)
      // Here we do a simple increment
      final restRef = _firestore
          .collection(FirestoreCollections.restaurants)
          .doc(restaurantId);
      batch.update(restRef, {
        'totalReviews': FieldValue.increment(1),
        // rating recalculation would be in a Cloud Function trigger
      });

      await batch.commit();
      return const Right(null);
    } catch (e) {
      return Left(ServerFailure(message: e.toString()));
    }
  }
}

class DriverLocation {
  final String driverId;
  final String displayName;
  final String? photoUrl;
  final String phoneNumber;
  final double latitude;
  final double longitude;

  const DriverLocation({
    required this.driverId,
    required this.displayName,
    this.photoUrl,
    required this.phoneNumber,
    required this.latitude,
    required this.longitude,
  });
}
```

---

## ขั้นตอนที่ 3682: Order Providers

```dart
// lib/features/order/presentation/providers/order_providers.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';

final orderRepositoryProvider = Provider<OrderRepositoryImpl>((ref) {
  return OrderRepositoryImpl(firestore: ref.watch(firestoreProvider));
});

final watchOrderProvider =
    StreamProvider.autoDispose.family<OrderEntity, String>((ref, orderId) {
  return ref.watch(orderRepositoryProvider).watchOrder(orderId);
});

final watchDriverLocationProvider = StreamProvider.autoDispose
    .family<DriverLocation?, String>((ref, orderId) {
  return ref.watch(orderRepositoryProvider).watchDriverLocation(orderId);
});

final orderHistoryProvider =
    FutureProvider.autoDispose<List<OrderEntity>>((ref) async {
  final user = ref.watch(authNotifierProvider).valueOrNull;
  if (user == null) return [];
  return ref.watch(orderRepositoryProvider).getOrderHistory(user.id);
});

// Place order notifier
final placeOrderNotifierProvider =
    AsyncNotifierProvider.autoDispose<PlaceOrderNotifier, OrderEntity?>(
        PlaceOrderNotifier.new);

class PlaceOrderNotifier extends AutoDisposeAsyncNotifier<OrderEntity?> {
  @override
  Future<OrderEntity?> build() async => null;

  Future<void> placeOrder({
    required PaymentMethod paymentMethod,
    String? paymentIntentId,
  }) async {
    state = const AsyncLoading();
    final cart = ref.read(cartNotifierProvider);
    final user = ref.read(authNotifierProvider).requireValue;

    if (user == null) {
      state = AsyncError(
          'กรุณาเข้าสู่ระบบก่อน', StackTrace.current);
      return;
    }

    final result = await ref.read(orderRepositoryProvider).placeOrder(
          cart: cart,
          customerId: user.id,
          paymentMethod: paymentMethod,
          paymentIntentId: paymentIntentId,
        );

    state = result.fold(
      (failure) => AsyncError(failure.message, StackTrace.current),
      (order) {
        // Clear cart after successful order
        ref.read(cartNotifierProvider.notifier).clearCart();
        return AsyncData(order);
      },
    );
  }
}
```

---

## ขั้นตอนที่ 3683: Order Tracking Page with Google Maps

```dart
// lib/features/order/presentation/pages/order_tracking_page.dart

import 'dart:async';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:google_maps_flutter/google_maps_flutter.dart';
import 'package:go_router/go_router.dart';
import 'package:url_launcher/url_launcher.dart';

class OrderTrackingPage extends ConsumerStatefulWidget {
  final String orderId;
  const OrderTrackingPage({super.key, required this.orderId});

  @override
  ConsumerState<OrderTrackingPage> createState() =>
      _OrderTrackingPageState();
}

class _OrderTrackingPageState extends ConsumerState<OrderTrackingPage> {
  GoogleMapController? _mapController;
  final Set<Marker> _markers = {};
  final Set<Polyline> _polylines = {};

  @override
  Widget build(BuildContext context) {
    final orderAsync = ref.watch(watchOrderProvider(widget.orderId));
    final driverAsync =
        ref.watch(watchDriverLocationProvider(widget.orderId));

    // Update map when driver location changes
    ref.listen(watchDriverLocationProvider(widget.orderId), (_, next) {
      next.whenData((driver) => _updateDriverMarker(driver));
    });

    return Scaffold(
      body: orderAsync.when(
        data: (order) => _buildTrackingView(order, driverAsync),
        loading: () =>
            const Center(child: CircularProgressIndicator()),
        error: (e, _) =>
            Center(child: Text('ไม่สามารถโหลดข้อมูลได้: $e')),
      ),
    );
  }

  Widget _buildTrackingView(
      OrderEntity order, AsyncValue<DriverLocation?> driverAsync) {
    return Stack(
      children: [
        GoogleMap(
          initialCameraPosition: CameraPosition(
            target: LatLng(
              order.deliveryAddress.latitude,
              order.deliveryAddress.longitude,
            ),
            zoom: 14,
          ),
          onMapCreated: (ctrl) {
            _mapController = ctrl;
            _addDeliveryMarker(order.deliveryAddress);
          },
          markers: _markers,
          polylines: _polylines,
          myLocationEnabled: false,
          zoomControlsEnabled: false,
        ),
        DraggableScrollableSheet(
          initialChildSize: 0.4,
          minChildSize: 0.2,
          maxChildSize: 0.85,
          builder: (_, ctrl) => _OrderStatusSheet(
            order: order,
            driver: driverAsync.valueOrNull,
            scrollController: ctrl,
          ),
        ),
        _buildBackButton(context),
      ],
    );
  }

  Widget _buildBackButton(BuildContext context) => SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: CircleAvatar(
            backgroundColor: Colors.white,
            child: IconButton(
              icon: const Icon(Icons.arrow_back, color: Colors.black),
              onPressed: () => context.pop(),
            ),
          ),
        ),
      );

  void _addDeliveryMarker(DeliveryAddress address) {
    setState(() {
      _markers.add(
        Marker(
          markerId: const MarkerId('delivery'),
          position: LatLng(address.latitude, address.longitude),
          icon: BitmapDescriptor.defaultMarkerWithHue(
              BitmapDescriptor.hueOrange),
          infoWindow: InfoWindow(title: 'ที่อยู่จัดส่ง'),
        ),
      );
    });
  }

  void _updateDriverMarker(DriverLocation? driver) {
    if (driver == null) return;
    final pos = LatLng(driver.latitude, driver.longitude);
    setState(() {
      _markers.removeWhere(
          (m) => m.markerId == const MarkerId('driver'));
      _markers.add(
        Marker(
          markerId: const MarkerId('driver'),
          position: pos,
          icon: BitmapDescriptor.defaultMarkerWithHue(
              BitmapDescriptor.hueBlue),
          infoWindow: InfoWindow(title: driver.displayName),
        ),
      );
    });
    _mapController?.animateCamera(
      CameraUpdate.newLatLng(pos),
    );
  }
}

class _OrderStatusSheet extends StatelessWidget {
  final OrderEntity order;
  final DriverLocation? driver;
  final ScrollController scrollController;

  const _OrderStatusSheet({
    required this.order,
    required this.driver,
    required this.scrollController,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      decoration: const BoxDecoration(
        color: Colors.white,
        borderRadius: BorderRadius.vertical(top: Radius.circular(24)),
        boxShadow: [
          BoxShadow(
              color: Colors.black26, blurRadius: 16, offset: Offset(0, -4))
        ],
      ),
      child: ListView(
        controller: scrollController,
        padding: const EdgeInsets.all(20),
        children: [
          _buildHandle(),
          _buildOrderStatus(),
          const SizedBox(height: 16),
          _buildStatusTimeline(),
          if (driver != null) ...[
            const Divider(height: 32),
            _DriverInfo(driver: driver!),
          ],
          const Divider(height: 32),
          _buildOrderItems(),
          const Divider(height: 32),
          _buildOrderTotal(),
        ],
      ),
    );
  }

  Widget _buildHandle() => Center(
        child: Container(
          width: 40,
          height: 4,
          margin: const EdgeInsets.only(bottom: 16),
          decoration: BoxDecoration(
            color: Colors.grey[300],
            borderRadius: BorderRadius.circular(2),
          ),
        ),
      );

  Widget _buildOrderStatus() {
    final statusInfo = _getStatusInfo(order.status);
    return Row(
      children: [
        Container(
          width: 52,
          height: 52,
          decoration: BoxDecoration(
            color: statusInfo['color'] as Color,
            shape: BoxShape.circle,
          ),
          child: Icon(
            statusInfo['icon'] as IconData,
            color: Colors.white,
            size: 28,
          ),
        ),
        const SizedBox(width: 12),
        Expanded(
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                statusInfo['title'] as String,
                style: const TextStyle(
                    fontSize: 18, fontWeight: FontWeight.bold),
              ),
              Text(
                statusInfo['subtitle'] as String,
                style: TextStyle(color: Colors.grey[600]),
              ),
              if (order.estimatedDeliveryTime != null)
                Text(
                  'คาดว่าจะถึง: ${_formatTime(order.estimatedDeliveryTime!)}',
                  style: const TextStyle(
                      color: Colors.orange, fontWeight: FontWeight.bold),
                ),
            ],
          ),
        ),
      ],
    );
  }

  Map<String, dynamic> _getStatusInfo(OrderStatus status) {
    switch (status) {
      case OrderStatus.pending:
        return {
          'title': 'รอการยืนยัน',
          'subtitle': 'ร้านอาหารกำลังรับคำสั่ง',
          'icon': Icons.hourglass_empty,
          'color': Colors.orange,
        };
      case OrderStatus.confirmed:
        return {
          'title': 'ยืนยันแล้ว',
          'subtitle': 'ร้านอาหารรับออเดอร์แล้ว',
          'icon': Icons.check_circle,
          'color': Colors.blue,
        };
      case OrderStatus.preparing:
        return {
          'title': 'กำลังเตรียมอาหาร',
          'subtitle': 'ร้านอาหารกำลังปรุงอาหาร',
          'icon': Icons.restaurant,
          'color': Colors.purple,
        };
      case OrderStatus.driverAssigned:
        return {
          'title': 'หาคนขับแล้ว',
          'subtitle': 'คนขับกำลังเดินทางไปร้าน',
          'icon': Icons.delivery_dining,
          'color': Colors.indigo,
        };
      case OrderStatus.pickedUp:
      case OrderStatus.onTheWay:
        return {
          'title': 'กำลังจัดส่ง',
          'subtitle': 'คนขับกำลังนำอาหารมาหาคุณ',
          'icon': Icons.directions_bike,
          'color': Colors.teal,
        };
      case OrderStatus.delivered:
        return {
          'title': 'จัดส่งสำเร็จ',
          'subtitle': 'อาหารถึงแล้ว! อร่อยนะคะ',
          'icon': Icons.check_circle,
          'color': Colors.green,
        };
      case OrderStatus.cancelled:
        return {
          'title': 'ยกเลิกออเดอร์',
          'subtitle': 'ออเดอร์ถูกยกเลิก',
          'icon': Icons.cancel,
          'color': Colors.red,
        };
      default:
        return {
          'title': 'กำลังดำเนินการ',
          'subtitle': '',
          'icon': Icons.info,
          'color': Colors.grey,
        };
    }
  }

  Widget _buildStatusTimeline() {
    final steps = [
      (OrderStatus.pending, 'รับออเดอร์'),
      (OrderStatus.confirmed, 'ยืนยัน'),
      (OrderStatus.preparing, 'เตรียมอาหาร'),
      (OrderStatus.onTheWay, 'กำลังส่ง'),
      (OrderStatus.delivered, 'ส่งสำเร็จ'),
    ];

    final currentIndex = steps.indexWhere(
        (s) => s.$1 == order.status);

    return Row(
      children: steps.asMap().entries.map((entry) {
        final i = entry.key;
        final step = entry.value;
        final isDone = currentIndex >= i;
        final isLast = i == steps.length - 1;

        return Expanded(
          child: Row(
            children: [
              Column(
                children: [
                  CircleAvatar(
                    radius: 12,
                    backgroundColor:
                        isDone ? Colors.orange : Colors.grey.shade300,
                    child: isDone
                        ? const Icon(Icons.check,
                            size: 14, color: Colors.white)
                        : null,
                  ),
                  const SizedBox(height: 4),
                  Text(
                    step.$2,
                    style: TextStyle(
                      fontSize: 9,
                      color: isDone ? Colors.orange : Colors.grey,
                      fontWeight: isDone
                          ? FontWeight.bold
                          : FontWeight.normal,
                    ),
                    textAlign: TextAlign.center,
                  ),
                ],
              ),
              if (!isLast)
                Expanded(
                  child: Container(
                    height: 2,
                    color: isDone && currentIndex > i
                        ? Colors.orange
                        : Colors.grey.shade300,
                  ),
                ),
            ],
          ),
        );
      }).toList(),
    );
  }

  Widget _buildOrderItems() => Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const Text('รายการอาหาร',
              style:
                  TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
          const SizedBox(height: 8),
          ...order.items.map((item) => Padding(
                padding: const EdgeInsets.symmetric(vertical: 4),
                child: Row(
                  children: [
                    Text('x${item.quantity} ',
                        style: const TextStyle(
                            fontWeight: FontWeight.bold,
                            color: Colors.orange)),
                    Expanded(child: Text(item.menuItemName)),
                    Text('฿${item.totalPrice.toStringAsFixed(0)}'),
                  ],
                ),
              )),
        ],
      );

  Widget _buildOrderTotal() => Column(
        children: [
          _TotalRow(
              label: 'ยอดรวม',
              value: '฿${order.subtotal.toStringAsFixed(0)}'),
          _TotalRow(
              label: 'ค่าจัดส่ง',
              value: '฿${order.deliveryFee.toStringAsFixed(0)}'),
          if (order.discount > 0)
            _TotalRow(
                label: 'ส่วนลด',
                value: '-฿${order.discount.toStringAsFixed(0)}',
                valueColor: Colors.green),
          _TotalRow(
              label: 'รวมทั้งสิ้น',
              value: '฿${order.total.toStringAsFixed(0)}',
              isBold: true),
        ],
      );

  String _formatTime(DateTime dt) =>
      '${dt.hour.toString().padLeft(2, '0')}:${dt.minute.toString().padLeft(2, '0')} น.';
}

class _TotalRow extends StatelessWidget {
  final String label;
  final String value;
  final bool isBold;
  final Color? valueColor;
  const _TotalRow({
    required this.label,
    required this.value,
    this.isBold = false,
    this.valueColor,
  });

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(label,
              style: TextStyle(
                fontWeight:
                    isBold ? FontWeight.bold : FontWeight.normal,
                fontSize: isBold ? 16 : 14,
              )),
          Text(value,
              style: TextStyle(
                fontWeight:
                    isBold ? FontWeight.bold : FontWeight.normal,
                fontSize: isBold ? 16 : 14,
                color: valueColor ?? Colors.black87,
              )),
        ],
      ),
    );
  }
}

class _DriverInfo extends StatelessWidget {
  final DriverLocation driver;
  const _DriverInfo({required this.driver});

  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        CircleAvatar(
          radius: 28,
          backgroundImage: driver.photoUrl != null
              ? CachedNetworkImageProvider(driver.photoUrl!)
              : null,
          child: driver.photoUrl == null
              ? Text(
                  driver.displayName.isNotEmpty
                      ? driver.displayName[0].toUpperCase()
                      : 'D',
                  style: const TextStyle(fontSize: 20),
                )
              : null,
        ),
        const SizedBox(width: 12),
        Expanded(
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              const Text('คนขับของคุณ',
                  style: TextStyle(color: Colors.grey, fontSize: 12)),
              Text(
                driver.displayName,
                style: const TextStyle(
                    fontWeight: FontWeight.bold, fontSize: 16),
              ),
            ],
          ),
        ),
        IconButton(
          icon: const CircleAvatar(
            backgroundColor: Colors.green,
            child: Icon(Icons.phone, color: Colors.white, size: 20),
          ),
          onPressed: () => launchUrl(
              Uri.parse('tel:${driver.phoneNumber}')),
        ),
        IconButton(
          icon: const CircleAvatar(
            backgroundColor: Colors.orange,
            child: Icon(Icons.chat, color: Colors.white, size: 20),
          ),
          onPressed: () {},
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 3684: Push Notifications with FCM

```dart
// lib/core/services/notification_service.dart

import 'package:firebase_messaging/firebase_messaging.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

/// Background message handler – must be top-level function
@pragma('vm:entry-point')
Future<void> firebaseMessagingBackgroundHandler(
    RemoteMessage message) async {
  // Handle background message
  print('Background message: ${message.messageId}');
}

final notificationServiceProvider =
    Provider<NotificationService>((ref) {
  return NotificationService(
    messaging: FirebaseMessaging.instance,
  );
});

class NotificationService {
  final FirebaseMessaging _messaging;
  final FlutterLocalNotificationsPlugin _localNotifications =
      FlutterLocalNotificationsPlugin();

  static const AndroidNotificationChannel _channel =
      AndroidNotificationChannel(
    'order_updates',
    'Order Updates',
    description: 'Notifications about your food order status',
    importance: Importance.high,
  );

  NotificationService({required FirebaseMessaging messaging})
      : _messaging = messaging;

  Future<void> initialize(GoRouter router) async {
    // Register background handler
    FirebaseMessaging.onBackgroundMessage(
        firebaseMessagingBackgroundHandler);

    // Request permissions
    await _messaging.requestPermission(
      alert: true,
      badge: true,
      sound: true,
    );

    // Initialize local notifications
    const initSettings = InitializationSettings(
      android: AndroidInitializationSettings('@mipmap/ic_launcher'),
      iOS: DarwinInitializationSettings(),
    );
    await _localNotifications.initialize(
      initSettings,
      onDidReceiveNotificationResponse: (response) {
        _handleNotificationTap(response.payload, router);
      },
    );

    // Create high-importance channel for Android
    await _localNotifications
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.createNotificationChannel(_channel);

    // Handle foreground messages
    FirebaseMessaging.onMessage.listen((message) {
      _showLocalNotification(message);
    });

    // Handle notification tap when app is in background
    FirebaseMessaging.onMessageOpenedApp.listen((message) {
      _handleNotificationTap(
          message.data['orderId'] as String?, router);
    });

    // Handle notification tap when app was terminated
    final initial = await _messaging.getInitialMessage();
    if (initial != null) {
      _handleNotificationTap(
          initial.data['orderId'] as String?, router);
    }
  }

  void _showLocalNotification(RemoteMessage message) {
    final notification = message.notification;
    if (notification == null) return;

    _localNotifications.show(
      notification.hashCode,
      notification.title,
      notification.body,
      NotificationDetails(
        android: AndroidNotificationDetails(
          _channel.id,
          _channel.name,
          channelDescription: _channel.description,
          importance: Importance.high,
          priority: Priority.high,
          icon: '@mipmap/ic_launcher',
        ),
        iOS: const DarwinNotificationDetails(
          presentAlert: true,
          presentBadge: true,
          presentSound: true,
        ),
      ),
      payload: message.data['orderId'] as String?,
    );
  }

  void _handleNotificationTap(String? orderId, GoRouter router) {
    if (orderId != null) {
      router.push('/order/$orderId');
    }
  }

  Future<String?> getToken() => _messaging.getToken();

  Future<void> subscribeToTopic(String topic) =>
      _messaging.subscribeToTopic(topic);

  Future<void> unsubscribeFromTopic(String topic) =>
      _messaging.unsubscribeFromTopic(topic);

  /// Save FCM token to Firestore for the current user
  Future<void> saveFcmToken(
      FirebaseFirestore firestore, String userId) async {
    final token = await getToken();
    if (token == null) return;
    await firestore
        .collection(FirestoreCollections.users)
        .doc(userId)
        .update({'fcmToken': token});
  }
}
```

---

## ขั้นตอนที่ 3685: Order History Page

```dart
// lib/features/order/presentation/pages/order_history_page.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:intl/intl.dart';

class OrderHistoryPage extends ConsumerWidget {
  const OrderHistoryPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final historyAsync = ref.watch(orderHistoryProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('ประวัติการสั่งอาหาร')),
      body: historyAsync.when(
        data: (orders) {
          if (orders.isEmpty) {
            return const _EmptyHistory();
          }
          return ListView.separated(
            padding: const EdgeInsets.all(16),
            itemCount: orders.length,
            separatorBuilder: (_, __) => const SizedBox(height: 12),
            itemBuilder: (_, index) =>
                _OrderHistoryCard(order: orders[index]),
          );
        },
        loading: () =>
            const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text('เกิดข้อผิดพลาด: $e')),
      ),
    );
  }
}

class _EmptyHistory extends StatelessWidget {
  const _EmptyHistory();

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(Icons.receipt_long_outlined,
              size: 80, color: Colors.grey[300]),
          const SizedBox(height: 16),
          const Text(
            'ยังไม่มีประวัติการสั่งอาหาร',
            style:
                TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
          ),
          const SizedBox(height: 8),
          const Text('เริ่มสั่งอาหารอร่อยๆ ได้เลย!',
              style: TextStyle(color: Colors.grey)),
          const SizedBox(height: 24),
          ElevatedButton(
            onPressed: () => context.go('/home'),
            style: ElevatedButton.styleFrom(
              backgroundColor: Colors.orange,
              shape: RoundedRectangleBorder(
                  borderRadius: BorderRadius.circular(12)),
            ),
            child: const Text('สั่งอาหาร',
                style: TextStyle(color: Colors.white)),
          ),
        ],
      ),
    );
  }
}

class _OrderHistoryCard extends StatelessWidget {
  final OrderEntity order;
  const _OrderHistoryCard({required this.order});

  @override
  Widget build(BuildContext context) {
    return Card(
      elevation: 1,
      shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(12)),
      child: InkWell(
        onTap: () {
          if (order.status == OrderStatus.onTheWay ||
              order.status == OrderStatus.pickedUp) {
            context.push('/order/${order.id}');
          } else {
            context.push('/order-detail/${order.id}');
          }
        },
        borderRadius: BorderRadius.circular(12),
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  Text(
                    order.restaurantName,
                    style: const TextStyle(
                        fontWeight: FontWeight.bold, fontSize: 16),
                  ),
                  _StatusBadge(status: order.status),
                ],
              ),
              const SizedBox(height: 8),
              Text(
                order.items.take(2).map((i) => i.menuItemName).join(', ') +
                    (order.items.length > 2
                        ? ' +${order.items.length - 2}'
                        : ''),
                style:
                    TextStyle(color: Colors.grey[600], fontSize: 13),
              ),
              const SizedBox(height: 8),
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  Text(
                    DateFormat('dd MMM yyyy, HH:mm')
                        .format(order.createdAt),
                    style: TextStyle(
                        color: Colors.grey[500], fontSize: 12),
                  ),
                  Text(
                    '฿${order.total.toStringAsFixed(0)}',
                    style: const TextStyle(
                        fontWeight: FontWeight.bold,
                        color: Colors.orange,
                        fontSize: 15),
                  ),
                ],
              ),
              if (order.status == OrderStatus.delivered)
                Padding(
                  padding: const EdgeInsets.only(top: 12),
                  child: Row(
                    children: [
                      Expanded(
                        child: OutlinedButton.icon(
                          onPressed: () {},
                          icon: const Icon(Icons.star_outline,
                              size: 18),
                          label: const Text('ให้คะแนน'),
                          style: OutlinedButton.styleFrom(
                            foregroundColor: Colors.orange,
                            side: const BorderSide(color: Colors.orange),
                            shape: RoundedRectangleBorder(
                                borderRadius: BorderRadius.circular(8)),
                          ),
                        ),
                      ),
                      const SizedBox(width: 8),
                      Expanded(
                        child: ElevatedButton.icon(
                          onPressed: () {},
                          icon: const Icon(Icons.repeat, size: 18),
                          label: const Text('สั่งซ้ำ'),
                          style: ElevatedButton.styleFrom(
                            backgroundColor: Colors.orange,
                            foregroundColor: Colors.white,
                            shape: RoundedRectangleBorder(
                                borderRadius: BorderRadius.circular(8)),
                          ),
                        ),
                      ),
                    ],
                  ),
                ),
            ],
          ),
        ),
      ),
    );
  }
}

class _StatusBadge extends StatelessWidget {
  final OrderStatus status;
  const _StatusBadge({required this.status});

  @override
  Widget build(BuildContext context) {
    final (label, color) = switch (status) {
      OrderStatus.pending => ('รอยืนยัน', Colors.orange),
      OrderStatus.confirmed => ('ยืนยันแล้ว', Colors.blue),
      OrderStatus.preparing => ('กำลังเตรียม', Colors.purple),
      OrderStatus.driverAssigned => ('หาคนขับแล้ว', Colors.indigo),
      OrderStatus.pickedUp ||
      OrderStatus.onTheWay =>
        ('กำลังส่ง', Colors.teal),
      OrderStatus.delivered => ('ส่งสำเร็จ', Colors.green),
      OrderStatus.cancelled => ('ยกเลิก', Colors.red),
      OrderStatus.refunded => ('คืนเงินแล้ว', Colors.blueGrey),
      OrderStatus.readyForPickup => ('รอรับ', Colors.cyan),
    };

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 4),
      decoration: BoxDecoration(
        color: color.withOpacity(0.15),
        borderRadius: BorderRadius.circular(12),
      ),
      child: Text(
        label,
        style: TextStyle(
            color: color, fontSize: 12, fontWeight: FontWeight.bold),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3686: Rate Order Dialog

```dart
// lib/features/order/presentation/widgets/rate_order_dialog.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class RateOrderDialog extends ConsumerStatefulWidget {
  final OrderEntity order;
  const RateOrderDialog({super.key, required this.order});

  @override
  ConsumerState<RateOrderDialog> createState() =>
      _RateOrderDialogState();
}

class _RateOrderDialogState extends ConsumerState<RateOrderDialog> {
  double _rating = 5;
  final _commentCtrl = TextEditingController();
  bool _isSubmitting = false;

  @override
  void dispose() {
    _commentCtrl.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    setState(() => _isSubmitting = true);
    final user = ref.read(authNotifierProvider).valueOrNull;
    if (user == null) return;

    await ref.read(orderRepositoryProvider).rateOrder(
          orderId: widget.order.id,
          restaurantId: widget.order.restaurantId,
          userId: user.id,
          userDisplayName: user.displayName,
          rating: _rating,
          comment: _commentCtrl.text.trim(),
        );

    if (mounted) Navigator.pop(context);
  }

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
      title: const Text('ให้คะแนนร้านอาหาร'),
      content: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text(
            widget.order.restaurantName,
            style: const TextStyle(fontWeight: FontWeight.bold),
          ),
          const SizedBox(height: 16),
          Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: List.generate(5, (i) {
              return IconButton(
                icon: Icon(
                  i < _rating ? Icons.star : Icons.star_border,
                  size: 36,
                  color: Colors.amber,
                ),
                onPressed: () => setState(() => _rating = i + 1),
              );
            }),
          ),
          const SizedBox(height: 16),
          TextField(
            controller: _commentCtrl,
            maxLines: 3,
            decoration: const InputDecoration(
              hintText: 'บอกความรู้สึกของคุณ (ไม่บังคับ)',
              border: OutlineInputBorder(),
            ),
          ),
        ],
      ),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context),
          child: const Text('ข้าม'),
        ),
        ElevatedButton(
          onPressed: _isSubmitting ? null : _submit,
          style: ElevatedButton.styleFrom(
            backgroundColor: Colors.orange,
            foregroundColor: Colors.white,
            shape: RoundedRectangleBorder(
                borderRadius: BorderRadius.circular(8)),
          ),
          child: _isSubmitting
              ? const SizedBox(
                  width: 18,
                  height: 18,
                  child: CircularProgressIndicator(
                      color: Colors.white, strokeWidth: 2),
                )
              : const Text('ส่งรีวิว'),
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 3687: Main App Entry Point

```dart
// lib/main.dart

import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(const ProviderScope(child: QuickBiteApp()));
}

class QuickBiteApp extends ConsumerWidget {
  const QuickBiteApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(appRouterProvider);

    return MaterialApp.router(
      title: 'QuickBite',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(
          seedColor: Colors.orange,
          primary: Colors.orange,
        ),
        appBarTheme: const AppBarTheme(
          backgroundColor: Colors.white,
          foregroundColor: Colors.black,
          elevation: 0,
          centerTitle: true,
        ),
        elevatedButtonTheme: ElevatedButtonThemeData(
          style: ElevatedButton.styleFrom(
            backgroundColor: Colors.orange,
            foregroundColor: Colors.white,
            shape: RoundedRectangleBorder(
              borderRadius: BorderRadius.circular(12),
            ),
          ),
        ),
        inputDecorationTheme: InputDecorationTheme(
          border: OutlineInputBorder(
            borderRadius: BorderRadius.circular(12),
          ),
          focusedBorder: OutlineInputBorder(
            borderRadius: BorderRadius.circular(12),
            borderSide: const BorderSide(color: Colors.orange, width: 2),
          ),
        ),
      ),
      routerConfig: router,
    );
  }
}

// pubspec.yaml dependencies summary:
// flutter_riverpod: ^2.5.1
// go_router: ^13.0.0
// firebase_core: ^2.30.0
// firebase_auth: ^4.19.0
// cloud_firestore: ^4.17.0
// firebase_storage: ^11.7.0
// firebase_messaging: ^14.9.0
// google_sign_in: ^6.2.1
// sign_in_with_apple: ^6.1.0
// google_maps_flutter: ^2.6.0
// geolocator: ^11.0.0
// cached_network_image: ^3.3.1
// flutter_local_notifications: ^17.1.0
// pin_code_fields: ^8.0.1
// dartz: ^0.10.1
// equatable: ^2.0.5
// intl: ^0.19.0
// url_launcher: ^6.2.6
```

---

**← [Part 94](part-94-capstone-cart-checkout.md)**
**ต่อไป: Part 96 (Coming Soon)**
