# Part 94: Capstone – Cart & Checkout
## ขั้นตอนที่ 3641-3680

## 🎯 เป้าหมายของ Part นี้
- Cart management ด้วย Riverpod
- Address selection ด้วย Google Maps
- Delivery time selection
- Promo code / discount system
- Payment summary
- Full working Flutter code

---

## ขั้นตอนที่ 3641: Cart Item & State Models

```dart
// lib/features/cart/domain/entities/cart_item.dart

import 'package:equatable/equatable.dart';

class CartItem extends Equatable {
  final MenuItemEntity menuItem;
  final int quantity;
  final List<OrderItemCustomization> customizations;
  final String? specialInstructions;

  const CartItem({
    required this.menuItem,
    required this.quantity,
    this.customizations = const [],
    this.specialInstructions,
  });

  double get extraPrice => customizations.fold(0, (s, c) => s + c.extraPrice);
  double get unitPrice => menuItem.price + extraPrice;
  double get totalPrice => unitPrice * quantity;

  CartItem copyWith({
    int? quantity,
    List<OrderItemCustomization>? customizations,
    String? specialInstructions,
  }) {
    return CartItem(
      menuItem: menuItem,
      quantity: quantity ?? this.quantity,
      customizations: customizations ?? this.customizations,
      specialInstructions: specialInstructions ?? this.specialInstructions,
    );
  }

  // Convert to OrderItem for placing an order
  OrderItem toOrderItem() => OrderItem(
        menuItemId: menuItem.id,
        menuItemName: menuItem.name,
        menuItemImageUrl: menuItem.imageUrl,
        unitPrice: unitPrice,
        quantity: quantity,
        customizations: customizations,
        specialInstructions: specialInstructions,
        totalPrice: totalPrice,
      );

  @override
  List<Object?> get props =>
      [menuItem.id, quantity, customizations, specialInstructions];
}

// lib/features/cart/domain/entities/cart_state.dart

class CartState extends Equatable {
  final String? restaurantId;
  final String? restaurantName;
  final List<CartItem> items;
  final PromoCodeEntity? appliedPromo;
  final DeliveryAddress? deliveryAddress;
  final DateTime? scheduledTime;

  const CartState({
    this.restaurantId,
    this.restaurantName,
    this.items = const [],
    this.appliedPromo,
    this.deliveryAddress,
    this.scheduledTime,
  });

  bool get isEmpty => items.isEmpty;
  int get totalItemCount =>
      items.fold(0, (sum, item) => sum + item.quantity);

  double get subtotal =>
      items.fold(0, (sum, item) => sum + item.totalPrice);

  double get deliveryFee {
    // In real app, fetch from restaurant + distance calculation
    if (subtotal >= 300) return 0;
    return 40;
  }

  double get discount =>
      appliedPromo?.calculateDiscount(subtotal) ?? 0;

  double get tax => (subtotal - discount) * 0.07;

  double get total => subtotal + deliveryFee - discount + tax;

  CartState copyWith({
    String? restaurantId,
    String? restaurantName,
    List<CartItem>? items,
    PromoCodeEntity? appliedPromo,
    bool clearPromo = false,
    DeliveryAddress? deliveryAddress,
    DateTime? scheduledTime,
    bool clearScheduled = false,
  }) {
    return CartState(
      restaurantId: restaurantId ?? this.restaurantId,
      restaurantName: restaurantName ?? this.restaurantName,
      items: items ?? this.items,
      appliedPromo: clearPromo ? null : appliedPromo ?? this.appliedPromo,
      deliveryAddress: deliveryAddress ?? this.deliveryAddress,
      scheduledTime:
          clearScheduled ? null : scheduledTime ?? this.scheduledTime,
    );
  }

  @override
  List<Object?> get props =>
      [restaurantId, items, appliedPromo, deliveryAddress, scheduledTime];
}
```

---

## ขั้นตอนที่ 3642: Cart Notifier

```dart
// lib/features/cart/presentation/providers/cart_providers.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';

final cartNotifierProvider =
    StateNotifierProvider<CartNotifier, CartState>(
  (_) => CartNotifier(),
);

final cartItemCountProvider = Provider<int>((ref) {
  return ref.watch(cartNotifierProvider).totalItemCount;
});

final cartSubtotalProvider = Provider<double>((ref) {
  return ref.watch(cartNotifierProvider).subtotal;
});

class CartNotifier extends StateNotifier<CartState> {
  CartNotifier() : super(const CartState());

  /// Add or increment a cart item.
  /// If from a different restaurant, prompt and clear first (handled in UI).
  void addItem(CartItem newItem) {
    final items = List<CartItem>.from(state.items);

    // Find existing item with same menuItemId + same customizations
    final existingIndex = items.indexWhere((item) =>
        item.menuItem.id == newItem.menuItem.id &&
        _customizationsMatch(item.customizations, newItem.customizations));

    if (existingIndex >= 0) {
      items[existingIndex] = items[existingIndex].copyWith(
        quantity: items[existingIndex].quantity + newItem.quantity,
      );
    } else {
      items.add(newItem);
    }

    state = state.copyWith(
      restaurantId: newItem.menuItem.restaurantId,
      items: items,
    );
  }

  bool _customizationsMatch(
    List<OrderItemCustomization> a,
    List<OrderItemCustomization> b,
  ) {
    if (a.length != b.length) return false;
    for (int i = 0; i < a.length; i++) {
      if (a[i].optionId != b[i].optionId) return false;
      if (a[i].choiceIds.join(',') != b[i].choiceIds.join(',')) {
        return false;
      }
    }
    return true;
  }

  void removeItem(int index) {
    final items = List<CartItem>.from(state.items)..removeAt(index);
    if (items.isEmpty) {
      state = const CartState();
    } else {
      state = state.copyWith(items: items);
    }
  }

  void updateQuantity(int index, int quantity) {
    if (quantity <= 0) {
      removeItem(index);
      return;
    }
    final items = List<CartItem>.from(state.items);
    items[index] = items[index].copyWith(quantity: quantity);
    state = state.copyWith(items: items);
  }

  void applyPromoCode(PromoCodeEntity promo) {
    state = state.copyWith(appliedPromo: promo);
  }

  void removePromoCode() {
    state = state.copyWith(clearPromo: true);
  }

  void setDeliveryAddress(DeliveryAddress address) {
    state = state.copyWith(deliveryAddress: address);
  }

  void setScheduledTime(DateTime? time) {
    if (time == null) {
      state = state.copyWith(clearScheduled: true);
    } else {
      state = state.copyWith(scheduledTime: time);
    }
  }

  void clearCart() {
    state = const CartState();
  }
}
```

---

## ขั้นตอนที่ 3643: Cart Page UI

```dart
// lib/features/cart/presentation/pages/cart_page.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

class CartPage extends ConsumerWidget {
  const CartPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cart = ref.watch(cartNotifierProvider);

    if (cart.isEmpty) {
      return Scaffold(
        appBar: AppBar(title: const Text('ตะกร้าของฉัน')),
        body: const _EmptyCart(),
      );
    }

    return Scaffold(
      appBar: AppBar(
        title: Text('ตะกร้า (${cart.totalItemCount})'),
        actions: [
          TextButton(
            onPressed: () => _showClearConfirmation(context, ref),
            child:
                const Text('ล้างตะกร้า', style: TextStyle(color: Colors.red)),
          ),
        ],
      ),
      body: ListView(
        children: [
          _CartItemList(cart: cart),
          const Divider(height: 32),
          _DeliveryAddressSection(cart: cart),
          const Divider(height: 32),
          _DeliveryTimeSection(cart: cart),
          const Divider(height: 32),
          _PromoCodeSection(cart: cart),
          const Divider(height: 32),
          _OrderSummary(cart: cart),
          const SizedBox(height: 120),
        ],
      ),
      bottomNavigationBar: _CheckoutBar(cart: cart),
    );
  }

  Future<void> _showClearConfirmation(
      BuildContext context, WidgetRef ref) async {
    final confirm = await showDialog<bool>(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text('ล้างตะกร้า'),
        content: const Text('ต้องการลบสินค้าทั้งหมดออกจากตะกร้า?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('ยกเลิก'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context, true),
            child:
                const Text('ล้าง', style: TextStyle(color: Colors.red)),
          ),
        ],
      ),
    );
    if (confirm == true) {
      ref.read(cartNotifierProvider.notifier).clearCart();
    }
  }
}

class _EmptyCart extends StatelessWidget {
  const _EmptyCart();

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(Icons.shopping_cart_outlined,
              size: 80, color: Colors.grey[300]),
          const SizedBox(height: 16),
          const Text('ตะกร้าของคุณว่างเปล่า',
              style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
          const SizedBox(height: 8),
          const Text('เลือกอาหารที่คุณชอบแล้วเพิ่มลงตะกร้าได้เลย!',
              style: TextStyle(color: Colors.grey)),
          const SizedBox(height: 24),
          ElevatedButton(
            onPressed: () => context.go('/home'),
            style: ElevatedButton.styleFrom(
              backgroundColor: Colors.orange,
              shape: RoundedRectangleBorder(
                  borderRadius: BorderRadius.circular(12)),
            ),
            child: const Text('เลือกร้านอาหาร',
                style: TextStyle(color: Colors.white)),
          ),
        ],
      ),
    );
  }
}

class _CartItemList extends ConsumerWidget {
  final CartState cart;
  const _CartItemList({required this.cart});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Padding(
          padding: const EdgeInsets.all(16),
          child: Text(
            cart.restaurantName ?? 'ร้านอาหาร',
            style: const TextStyle(
                fontSize: 16, fontWeight: FontWeight.bold),
          ),
        ),
        ...cart.items.asMap().entries.map(
              (entry) => _CartItemTile(
                index: entry.key,
                item: entry.value,
                onIncrement: () => ref
                    .read(cartNotifierProvider.notifier)
                    .updateQuantity(
                        entry.key, entry.value.quantity + 1),
                onDecrement: () => ref
                    .read(cartNotifierProvider.notifier)
                    .updateQuantity(
                        entry.key, entry.value.quantity - 1),
              ),
            ),
      ],
    );
  }
}

class _CartItemTile extends StatelessWidget {
  final int index;
  final CartItem item;
  final VoidCallback onIncrement;
  final VoidCallback onDecrement;

  const _CartItemTile({
    required this.index,
    required this.item,
    required this.onIncrement,
    required this.onDecrement,
  });

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: ClipRRect(
        borderRadius: BorderRadius.circular(8),
        child: CachedNetworkImage(
          imageUrl: item.menuItem.imageUrl,
          width: 60,
          height: 60,
          fit: BoxFit.cover,
        ),
      ),
      title: Text(item.menuItem.name,
          style: const TextStyle(fontWeight: FontWeight.bold)),
      subtitle: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          if (item.customizations.isNotEmpty)
            Text(
              item.customizations
                  .map((c) => c.choiceNames.join(', '))
                  .join(' | '),
              style: const TextStyle(fontSize: 12, color: Colors.grey),
              maxLines: 1,
              overflow: TextOverflow.ellipsis,
            ),
          Text(
            '฿${item.totalPrice.toStringAsFixed(0)}',
            style: const TextStyle(
                color: Colors.orange, fontWeight: FontWeight.bold),
          ),
        ],
      ),
      trailing: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          _QuantityButton(
            icon: Icons.remove,
            onPressed: onDecrement,
          ),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 8),
            child: Text(
              '${item.quantity}',
              style: const TextStyle(
                  fontSize: 16, fontWeight: FontWeight.bold),
            ),
          ),
          _QuantityButton(
            icon: Icons.add,
            onPressed: onIncrement,
            isAdd: true,
          ),
        ],
      ),
    );
  }
}

class _QuantityButton extends StatelessWidget {
  final IconData icon;
  final VoidCallback onPressed;
  final bool isAdd;
  const _QuantityButton(
      {required this.icon, required this.onPressed, this.isAdd = false});

  @override
  Widget build(BuildContext context) {
    return Container(
      width: 32,
      height: 32,
      decoration: BoxDecoration(
        color: isAdd ? Colors.orange : Colors.grey.shade200,
        borderRadius: BorderRadius.circular(8),
      ),
      child: IconButton(
        icon: Icon(icon, size: 16),
        color: isAdd ? Colors.white : Colors.black,
        padding: EdgeInsets.zero,
        onPressed: onPressed,
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3644: Address Selection with Google Maps

```dart
// lib/features/cart/presentation/widgets/delivery_address_section.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:google_maps_flutter/google_maps_flutter.dart';
import 'package:geolocator/geolocator.dart';

class _DeliveryAddressSection extends ConsumerWidget {
  final CartState cart;
  const _DeliveryAddressSection({required this.cart});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const Text(
            'ที่อยู่จัดส่ง',
            style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
          ),
          const SizedBox(height: 12),
          if (cart.deliveryAddress != null)
            _SelectedAddress(
              address: cart.deliveryAddress!,
              onTap: () => _openAddressSelector(context, ref),
            )
          else
            OutlinedButton.icon(
              onPressed: () => _openAddressSelector(context, ref),
              icon: const Icon(Icons.add_location_alt_outlined,
                  color: Colors.orange),
              label: const Text(
                'เลือกที่อยู่จัดส่ง',
                style: TextStyle(color: Colors.orange),
              ),
              style: OutlinedButton.styleFrom(
                minimumSize: const Size.fromHeight(52),
                side: const BorderSide(color: Colors.orange),
                shape: RoundedRectangleBorder(
                    borderRadius: BorderRadius.circular(12)),
              ),
            ),
        ],
      ),
    );
  }

  Future<void> _openAddressSelector(
      BuildContext context, WidgetRef ref) async {
    final address = await showModalBottomSheet<DeliveryAddress>(
      context: context,
      isScrollControlled: true,
      shape: const RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
      ),
      builder: (_) => const _AddressSelectorSheet(),
    );
    if (address != null) {
      ref.read(cartNotifierProvider.notifier).setDeliveryAddress(address);
    }
  }
}

class _SelectedAddress extends StatelessWidget {
  final DeliveryAddress address;
  final VoidCallback onTap;

  const _SelectedAddress({required this.address, required this.onTap});

  @override
  Widget build(BuildContext context) {
    return InkWell(
      onTap: onTap,
      borderRadius: BorderRadius.circular(12),
      child: Container(
        padding: const EdgeInsets.all(12),
        decoration: BoxDecoration(
          border: Border.all(color: Colors.orange.shade200),
          borderRadius: BorderRadius.circular(12),
          color: Colors.orange.shade50,
        ),
        child: Row(
          children: [
            const Icon(Icons.location_on, color: Colors.orange),
            const SizedBox(width: 12),
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    address.label,
                    style: const TextStyle(fontWeight: FontWeight.bold),
                  ),
                  Text(
                    address.fullAddress,
                    style: TextStyle(
                        color: Colors.grey[600], fontSize: 13),
                    maxLines: 2,
                    overflow: TextOverflow.ellipsis,
                  ),
                  if (address.instructions != null &&
                      address.instructions!.isNotEmpty)
                    Text(
                      address.instructions!,
                      style: const TextStyle(
                          fontSize: 12, color: Colors.grey),
                    ),
                ],
              ),
            ),
            const Icon(Icons.edit, size: 18, color: Colors.orange),
          ],
        ),
      ),
    );
  }
}

class _AddressSelectorSheet extends ConsumerStatefulWidget {
  const _AddressSelectorSheet();

  @override
  ConsumerState<_AddressSelectorSheet> createState() =>
      _AddressSelectorSheetState();
}

class _AddressSelectorSheetState
    extends ConsumerState<_AddressSelectorSheet> {
  GoogleMapController? _mapController;
  LatLng _selectedLocation = const LatLng(13.7563, 100.5018);
  String _addressText = '';
  bool _isLocating = false;

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      height: MediaQuery.of(context).size.height * 0.85,
      child: Column(
        children: [
          _buildHandle(),
          const Padding(
            padding: EdgeInsets.all(16),
            child: Text(
              'เลือกที่อยู่จัดส่ง',
              style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
            ),
          ),
          Expanded(
            child: Stack(
              children: [
                GoogleMap(
                  initialCameraPosition: CameraPosition(
                    target: _selectedLocation,
                    zoom: 15,
                  ),
                  onMapCreated: (ctrl) => _mapController = ctrl,
                  onCameraMove: (pos) {
                    setState(
                        () => _selectedLocation = pos.target);
                  },
                  myLocationEnabled: true,
                  myLocationButtonEnabled: false,
                ),
                const Center(
                  child: Icon(Icons.location_pin,
                      size: 48, color: Colors.orange),
                ),
                Positioned(
                  bottom: 16,
                  right: 16,
                  child: FloatingActionButton.small(
                    backgroundColor: Colors.white,
                    onPressed: _getCurrentLocation,
                    child: _isLocating
                        ? const SizedBox(
                            width: 18,
                            height: 18,
                            child: CircularProgressIndicator(strokeWidth: 2),
                          )
                        : const Icon(Icons.my_location,
                            color: Colors.orange),
                  ),
                ),
              ],
            ),
          ),
          _buildBottomSection(),
        ],
      ),
    );
  }

  Widget _buildHandle() => Center(
        child: Container(
          width: 40,
          height: 4,
          margin: const EdgeInsets.symmetric(vertical: 8),
          decoration: BoxDecoration(
            color: Colors.grey[300],
            borderRadius: BorderRadius.circular(2),
          ),
        ),
      );

  Widget _buildBottomSection() {
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: Colors.white,
        boxShadow: [
          BoxShadow(
            color: Colors.black.withOpacity(0.05),
            blurRadius: 10,
            offset: const Offset(0, -4),
          ),
        ],
      ),
      child: Column(
        children: [
          if (_addressText.isNotEmpty)
            ListTile(
              leading:
                  const Icon(Icons.location_on, color: Colors.orange),
              title: Text(_addressText),
              dense: true,
            ),
          ElevatedButton(
            onPressed: () {
              final address = DeliveryAddress(
                label: 'ที่อยู่ปัจจุบัน',
                fullAddress: _addressText.isNotEmpty
                    ? _addressText
                    : '${_selectedLocation.latitude.toStringAsFixed(4)}, '
                        '${_selectedLocation.longitude.toStringAsFixed(4)}',
                subdistrict: '',
                district: '',
                province: '',
                postalCode: '',
                latitude: _selectedLocation.latitude,
                longitude: _selectedLocation.longitude,
              );
              Navigator.pop(context, address);
            },
            style: ElevatedButton.styleFrom(
              backgroundColor: Colors.orange,
              minimumSize: const Size.fromHeight(52),
              shape: RoundedRectangleBorder(
                  borderRadius: BorderRadius.circular(12)),
            ),
            child: const Text('ยืนยันที่อยู่นี้',
                style: TextStyle(color: Colors.white, fontSize: 16)),
          ),
        ],
      ),
    );
  }

  Future<void> _getCurrentLocation() async {
    setState(() => _isLocating = true);
    try {
      final perm = await Geolocator.checkPermission();
      if (perm == LocationPermission.denied) {
        await Geolocator.requestPermission();
      }
      final pos = await Geolocator.getCurrentPosition();
      setState(() {
        _selectedLocation = LatLng(pos.latitude, pos.longitude);
      });
      _mapController?.animateCamera(
        CameraUpdate.newLatLng(_selectedLocation),
      );
    } catch (e) {
      // Handle error
    } finally {
      setState(() => _isLocating = false);
    }
  }
}
```

---

## ขั้นตอนที่ 3645: Delivery Time Selection

```dart
// lib/features/cart/presentation/widgets/delivery_time_section.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:intl/intl.dart';

class _DeliveryTimeSection extends ConsumerWidget {
  final CartState cart;
  const _DeliveryTimeSection({required this.cart});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const Text(
            'เวลาจัดส่ง',
            style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
          ),
          const SizedBox(height: 12),
          Row(
            children: [
              Expanded(
                child: _TimeOptionCard(
                  title: 'จัดส่งเดี๋ยวนี้',
                  subtitle: 'ประมาณ 30-45 นาที',
                  icon: Icons.flash_on,
                  isSelected: cart.scheduledTime == null,
                  onTap: () => ref
                      .read(cartNotifierProvider.notifier)
                      .setScheduledTime(null),
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: _TimeOptionCard(
                  title: 'กำหนดเวลา',
                  subtitle: cart.scheduledTime != null
                      ? DateFormat('dd/MM HH:mm')
                          .format(cart.scheduledTime!)
                      : 'เลือกวันและเวลา',
                  icon: Icons.schedule,
                  isSelected: cart.scheduledTime != null,
                  onTap: () =>
                      _showTimePicker(context, ref),
                ),
              ),
            ],
          ),
        ],
      ),
    );
  }

  Future<void> _showTimePicker(BuildContext context, WidgetRef ref) async {
    final now = DateTime.now();
    final minTime = now.add(const Duration(hours: 1));

    // Show custom time picker
    final slots = _generateTimeSlots(minTime);

    final selected = await showModalBottomSheet<DateTime>(
      context: context,
      shape: const RoundedRectangleBorder(
        borderRadius: BorderRadius.vertical(top: Radius.circular(20)),
      ),
      builder: (_) => _TimeSlotPicker(slots: slots),
    );

    if (selected != null) {
      ref.read(cartNotifierProvider.notifier).setScheduledTime(selected);
    }
  }

  List<DateTime> _generateTimeSlots(DateTime from) {
    final slots = <DateTime>[];
    // Round to next 30-minute interval
    final start = DateTime(
      from.year, from.month, from.day,
      from.hour, from.minute >= 30 ? 30 : 0,
    ).add(Duration(minutes: from.minute >= 30 ? 30 : 30));

    for (int i = 0; i < 24; i++) {
      final slot = start.add(Duration(minutes: 30 * i));
      if (slot.hour >= 8 && slot.hour < 22) {
        slots.add(slot);
      }
    }
    return slots;
  }
}

class _TimeOptionCard extends StatelessWidget {
  final String title;
  final String subtitle;
  final IconData icon;
  final bool isSelected;
  final VoidCallback onTap;

  const _TimeOptionCard({
    required this.title,
    required this.subtitle,
    required this.icon,
    required this.isSelected,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      child: Container(
        padding: const EdgeInsets.all(12),
        decoration: BoxDecoration(
          color: isSelected ? Colors.orange.shade50 : Colors.white,
          border: Border.all(
            color: isSelected ? Colors.orange : Colors.grey.shade300,
            width: isSelected ? 2 : 1,
          ),
          borderRadius: BorderRadius.circular(12),
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Icon(icon,
                color: isSelected ? Colors.orange : Colors.grey),
            const SizedBox(height: 8),
            Text(
              title,
              style: TextStyle(
                fontWeight: FontWeight.bold,
                color: isSelected ? Colors.orange : Colors.black,
              ),
            ),
            Text(
              subtitle,
              style: TextStyle(
                  fontSize: 12,
                  color: isSelected
                      ? Colors.orange.shade700
                      : Colors.grey),
            ),
          ],
        ),
      ),
    );
  }
}

class _TimeSlotPicker extends StatefulWidget {
  final List<DateTime> slots;
  const _TimeSlotPicker({required this.slots});

  @override
  State<_TimeSlotPicker> createState() => _TimeSlotPickerState();
}

class _TimeSlotPickerState extends State<_TimeSlotPicker> {
  DateTime? _selected;

  @override
  Widget build(BuildContext context) {
    final grouped = <String, List<DateTime>>{};
    for (final s in widget.slots) {
      final key = DateFormat('EEE d MMM', 'th').format(s);
      grouped.putIfAbsent(key, () => []).add(s);
    }

    return Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        Container(
          width: 40,
          height: 4,
          margin: const EdgeInsets.symmetric(vertical: 8),
          decoration: BoxDecoration(
            color: Colors.grey[300],
            borderRadius: BorderRadius.circular(2),
          ),
        ),
        const Padding(
          padding: EdgeInsets.all(16),
          child: Text('เลือกเวลาจัดส่ง',
              style:
                  TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
        ),
        SizedBox(
          height: 300,
          child: ListView(
            children: grouped.entries.map((entry) {
              return Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Padding(
                    padding: const EdgeInsets.fromLTRB(16, 12, 16, 4),
                    child: Text(
                      entry.key,
                      style: const TextStyle(
                          fontWeight: FontWeight.bold,
                          color: Colors.grey),
                    ),
                  ),
                  Wrap(
                    spacing: 8,
                    runSpacing: 4,
                    children: entry.value.map((slot) {
                      final isSelected = _selected == slot;
                      return Padding(
                        padding: const EdgeInsets.only(left: 16),
                        child: ChoiceChip(
                          label: Text(DateFormat('HH:mm').format(slot)),
                          selected: isSelected,
                          selectedColor: Colors.orange.shade100,
                          onSelected: (_) =>
                              setState(() => _selected = slot),
                        ),
                      );
                    }).toList(),
                  ),
                ],
              );
            }).toList(),
          ),
        ),
        Padding(
          padding: const EdgeInsets.all(16),
          child: ElevatedButton(
            onPressed: _selected != null
                ? () => Navigator.pop(context, _selected)
                : null,
            style: ElevatedButton.styleFrom(
              backgroundColor: Colors.orange,
              minimumSize: const Size.fromHeight(52),
              shape: RoundedRectangleBorder(
                  borderRadius: BorderRadius.circular(12)),
            ),
            child: const Text('ยืนยัน',
                style: TextStyle(color: Colors.white)),
          ),
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 3646: Promo Code Section

```dart
// lib/features/cart/presentation/widgets/promo_code_section.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

final promoValidationProvider = StateNotifierProvider.autoDispose<
    PromoValidationNotifier, AsyncValue<PromoCodeEntity?>>((ref) {
  return PromoValidationNotifier(ref.watch(firestoreProvider));
});

class PromoValidationNotifier
    extends StateNotifier<AsyncValue<PromoCodeEntity?>> {
  final FirebaseFirestore _firestore;
  PromoValidationNotifier(this._firestore)
      : super(const AsyncData(null));

  Future<void> validate(String code, double orderAmount) async {
    if (code.isEmpty) return;
    state = const AsyncLoading();
    try {
      final doc = await _firestore
          .collection(FirestoreCollections.promoCodes)
          .doc(code.toUpperCase())
          .get();

      if (!doc.exists) {
        state = AsyncError('ไม่พบรหัสส่วนลด', StackTrace.current);
        return;
      }

      final data = doc.data()!;
      final promo = PromoCodeEntity(
        code: doc.id,
        type: data['type'] == 'percentage'
            ? PromoType.percentage
            : PromoType.fixed,
        value: (data['value'] as num).toDouble(),
        minOrderAmount: (data['minOrderAmount'] as num).toDouble(),
        maxDiscountAmount:
            (data['maxDiscountAmount'] as num?)?.toDouble(),
        usageLimit: data['usageLimit'] as int,
        usedCount: data['usedCount'] as int,
        validFrom:
            (data['validFrom'] as dynamic).toDate() as DateTime,
        validUntil:
            (data['validUntil'] as dynamic).toDate() as DateTime,
        isActive: data['isActive'] as bool,
      );

      if (!promo.isValid) {
        state = AsyncError('รหัสส่วนลดหมดอายุหรือใช้ครบแล้ว',
            StackTrace.current);
        return;
      }

      if (orderAmount < promo.minOrderAmount) {
        state = AsyncError(
          'ต้องสั่งขั้นต่ำ ฿${promo.minOrderAmount.toStringAsFixed(0)}',
          StackTrace.current,
        );
        return;
      }

      state = AsyncData(promo);
    } catch (e, st) {
      state = AsyncError(e.toString(), st);
    }
  }

  void clear() => state = const AsyncData(null);
}

class _PromoCodeSection extends ConsumerStatefulWidget {
  final CartState cart;
  const _PromoCodeSection({required this.cart});

  @override
  ConsumerState<_PromoCodeSection> createState() =>
      _PromoCodeSectionState();
}

class _PromoCodeSectionState
    extends ConsumerState<_PromoCodeSection> {
  final _ctrl = TextEditingController();

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final promoState = ref.watch(promoValidationProvider);
    final appliedPromo = widget.cart.appliedPromo;

    return Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const Text('โปรโมชัน / รหัสส่วนลด',
              style:
                  TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
          const SizedBox(height: 12),
          if (appliedPromo != null)
            _AppliedPromoChip(
              promo: appliedPromo,
              discount: appliedPromo.calculateDiscount(widget.cart.subtotal),
              onRemove: () {
                ref.read(cartNotifierProvider.notifier).removePromoCode();
                ref.read(promoValidationProvider.notifier).clear();
                _ctrl.clear();
              },
            )
          else
            Row(
              children: [
                Expanded(
                  child: TextField(
                    controller: _ctrl,
                    textCapitalization: TextCapitalization.characters,
                    decoration: InputDecoration(
                      hintText: 'กรอกรหัสส่วนลด',
                      border: const OutlineInputBorder(),
                      contentPadding: const EdgeInsets.symmetric(
                          horizontal: 16, vertical: 12),
                      errorText: promoState.hasError
                          ? promoState.error.toString()
                          : null,
                    ),
                  ),
                ),
                const SizedBox(width: 12),
                ElevatedButton(
                  onPressed: promoState.isLoading
                      ? null
                      : () async {
                          await ref
                              .read(promoValidationProvider.notifier)
                              .validate(
                                _ctrl.text.trim(),
                                widget.cart.subtotal,
                              );
                          final promo =
                              ref.read(promoValidationProvider).valueOrNull;
                          if (promo != null) {
                            ref
                                .read(cartNotifierProvider.notifier)
                                .applyPromoCode(promo);
                          }
                        },
                  style: ElevatedButton.styleFrom(
                    backgroundColor: Colors.orange,
                    padding: const EdgeInsets.symmetric(
                        horizontal: 20, vertical: 14),
                  ),
                  child: promoState.isLoading
                      ? const SizedBox(
                          width: 18,
                          height: 18,
                          child: CircularProgressIndicator(
                              color: Colors.white, strokeWidth: 2),
                        )
                      : const Text('ใช้',
                          style: TextStyle(color: Colors.white)),
                ),
              ],
            ),
        ],
      ),
    );
  }
}

class _AppliedPromoChip extends StatelessWidget {
  final PromoCodeEntity promo;
  final double discount;
  final VoidCallback onRemove;

  const _AppliedPromoChip({
    required this.promo,
    required this.discount,
    required this.onRemove,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(12),
      decoration: BoxDecoration(
        color: Colors.green.shade50,
        border: Border.all(color: Colors.green.shade200),
        borderRadius: BorderRadius.circular(12),
      ),
      child: Row(
        children: [
          const Icon(Icons.local_offer, color: Colors.green),
          const SizedBox(width: 8),
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  promo.code,
                  style: const TextStyle(
                      fontWeight: FontWeight.bold, color: Colors.green),
                ),
                Text(
                  'ลด ฿${discount.toStringAsFixed(0)}',
                  style: TextStyle(
                      color: Colors.green.shade700, fontSize: 13),
                ),
              ],
            ),
          ),
          IconButton(
            icon: const Icon(Icons.close, color: Colors.grey),
            onPressed: onRemove,
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3647: Order Summary & Checkout

```dart
// lib/features/cart/presentation/widgets/order_summary.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

class _OrderSummary extends StatelessWidget {
  final CartState cart;
  const _OrderSummary({required this.cart});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          const Text('สรุปรายการ',
              style:
                  TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
          const SizedBox(height: 12),
          _SummaryRow(
              label: 'ยอดรวม',
              value: '฿${cart.subtotal.toStringAsFixed(2)}'),
          _SummaryRow(
              label: 'ค่าจัดส่ง',
              value: cart.deliveryFee == 0
                  ? 'ฟรี'
                  : '฿${cart.deliveryFee.toStringAsFixed(2)}',
              valueColor:
                  cart.deliveryFee == 0 ? Colors.green : null),
          if (cart.discount > 0)
            _SummaryRow(
                label: 'ส่วนลด',
                value: '-฿${cart.discount.toStringAsFixed(2)}',
                valueColor: Colors.green),
          _SummaryRow(
              label: 'ภาษีมูลค่าเพิ่ม (7%)',
              value: '฿${cart.tax.toStringAsFixed(2)}'),
          const Divider(height: 24),
          _SummaryRow(
            label: 'รวมทั้งสิ้น',
            value: '฿${cart.total.toStringAsFixed(2)}',
            isBold: true,
            labelColor: Colors.black,
            valueColor: Colors.orange,
          ),
        ],
      ),
    );
  }
}

class _SummaryRow extends StatelessWidget {
  final String label;
  final String value;
  final bool isBold;
  final Color? labelColor;
  final Color? valueColor;

  const _SummaryRow({
    required this.label,
    required this.value,
    this.isBold = false,
    this.labelColor,
    this.valueColor,
  });

  @override
  Widget build(BuildContext context) {
    final textStyle = TextStyle(
      fontSize: isBold ? 18 : 14,
      fontWeight: isBold ? FontWeight.bold : FontWeight.normal,
    );

    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(
            label,
            style: textStyle.copyWith(
                color: labelColor ?? Colors.grey[700]),
          ),
          Text(
            value,
            style: textStyle.copyWith(
                color: valueColor ?? Colors.black87),
          ),
        ],
      ),
    );
  }
}

class _CheckoutBar extends ConsumerWidget {
  final CartState cart;
  const _CheckoutBar({required this.cart});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final canCheckout = !cart.isEmpty && cart.deliveryAddress != null;

    return SafeArea(
      child: Container(
        padding: const EdgeInsets.all(16),
        decoration: BoxDecoration(
          color: Colors.white,
          boxShadow: [
            BoxShadow(
              color: Colors.black.withOpacity(0.08),
              blurRadius: 16,
              offset: const Offset(0, -4),
            ),
          ],
        ),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            if (cart.deliveryAddress == null)
              const Padding(
                padding: EdgeInsets.only(bottom: 8),
                child: Row(
                  children: [
                    Icon(Icons.info_outline,
                        size: 16, color: Colors.orange),
                    SizedBox(width: 8),
                    Text(
                      'กรุณาเลือกที่อยู่จัดส่ง',
                      style: TextStyle(color: Colors.orange),
                    ),
                  ],
                ),
              ),
            ElevatedButton(
              onPressed: canCheckout
                  ? () => context.push('/checkout')
                  : null,
              style: ElevatedButton.styleFrom(
                backgroundColor: Colors.orange,
                disabledBackgroundColor: Colors.grey.shade300,
                minimumSize: const Size.fromHeight(56),
                shape: RoundedRectangleBorder(
                    borderRadius: BorderRadius.circular(14)),
              ),
              child: Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  const Text(
                    'ดำเนินการชำระเงิน',
                    style: TextStyle(
                        color: Colors.white,
                        fontSize: 16,
                        fontWeight: FontWeight.bold),
                  ),
                  Text(
                    '฿${cart.total.toStringAsFixed(0)}',
                    style: const TextStyle(
                        color: Colors.white,
                        fontSize: 16,
                        fontWeight: FontWeight.bold),
                  ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

**← [Part 93](part-93-capstone-restaurant-module.md)**
**ต่อไป: [Part 95 →](part-95-capstone-order-tracking.md)**
