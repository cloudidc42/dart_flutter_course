# Part 41: In-App Purchases
## ขั้นตอนที่ 1521-1560

---

## 🎯 เป้าหมายของ Part นี้

- in_app_purchase package
- One-time purchases
- Subscriptions
- Restore purchases
- Receipt validation

---

## ขั้นตอนที่ 1521: Setup

```yaml
# pubspec.yaml
dependencies:
  in_app_purchase: ^3.2.0
  in_app_purchase_android: ^0.3.5
  in_app_purchase_storekit: ^0.3.15
```

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="com.android.vending.BILLING" />
```

---

## ขั้นตอนที่ 1522: IAP Service

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:in_app_purchase/in_app_purchase.dart';

// ─── Product IDs ───
class IAPProducts {
  // Consumables (ซื้อได้หลายครั้ง)
  static const String coins100 = 'coins_100';
  static const String coins500 = 'coins_500';
  static const String coins1000 = 'coins_1000';

  // Non-consumables (ซื้อครั้งเดียว)
  static const String removeAds = 'remove_ads';
  static const String premiumTheme = 'premium_theme';

  // Subscriptions
  static const String monthlyPremium = 'monthly_premium';
  static const String yearlyPremium = 'yearly_premium';

  static const Set<String> all = {
    coins100, coins500, coins1000,
    removeAds, premiumTheme,
    monthlyPremium, yearlyPremium,
  };
}

// ─── IAP Service ───
class IAPService {
  static final IAPService _instance = IAPService._();
  factory IAPService() => _instance;
  IAPService._();

  final InAppPurchase _iap = InAppPurchase.instance;
  StreamSubscription<List<PurchaseDetails>>? _purchaseSubscription;

  List<ProductDetails> _products = [];
  List<ProductDetails> get products => _products;

  bool _available = false;
  bool get isAvailable => _available;

  final _purchaseController = StreamController<PurchaseDetails>.broadcast();
  Stream<PurchaseDetails> get purchaseStream => _purchaseController.stream;

  Future<void> initialize() async {
    _available = await _iap.isAvailable();
    if (!_available) return;

    // Listen for purchase updates
    _purchaseSubscription = _iap.purchaseStream.listen(
      _handlePurchases,
      onError: (error) => print('Purchase stream error: $error'),
    );

    // Load products
    await loadProducts();

    // Restore previous purchases
    await _iap.restorePurchases();
  }

  Future<void> loadProducts() async {
    ProductDetailsResponse response =
        await _iap.queryProductDetails(IAPProducts.all);

    if (response.error != null) {
      print('Product query error: ${response.error}');
      return;
    }

    _products = response.productDetails;
  }

  // ─── Buy product ───
  Future<void> buy(ProductDetails product, {bool isConsumable = false}) async {
    PurchaseParam param = PurchaseParam(productDetails: product);

    if (isConsumable) {
      await _iap.buyConsumable(purchaseParam: param);
    } else {
      await _iap.buyNonConsumable(purchaseParam: param);
    }
  }

  // ─── Restore purchases ───
  Future<void> restorePurchases() async {
    await _iap.restorePurchases();
  }

  // ─── Handle purchase updates ───
  Future<void> _handlePurchases(List<PurchaseDetails> purchases) async {
    for (PurchaseDetails purchase in purchases) {
      switch (purchase.status) {
        case PurchaseStatus.purchased:
        case PurchaseStatus.restored:
          await _verifyAndDeliver(purchase);
          break;
        case PurchaseStatus.pending:
          print('Purchase pending: ${purchase.productID}');
          break;
        case PurchaseStatus.error:
          print('Purchase error: ${purchase.error}');
          break;
        case PurchaseStatus.canceled:
          print('Purchase canceled: ${purchase.productID}');
          break;
      }

      // Complete the purchase
      if (purchase.pendingCompletePurchase) {
        await _iap.completePurchase(purchase);
      }

      _purchaseController.add(purchase);
    }
  }

  Future<void> _verifyAndDeliver(PurchaseDetails purchase) async {
    // Verify receipt with your backend
    bool valid = await _verifyReceipt(purchase);
    if (!valid) return;

    // Deliver the product
    await _deliverProduct(purchase);
  }

  Future<bool> _verifyReceipt(PurchaseDetails purchase) async {
    // Send receipt to backend for verification
    // POST /api/verify-purchase
    // { receipt: purchase.verificationData.serverVerificationData }
    return true; // Mock: always valid
  }

  Future<void> _deliverProduct(PurchaseDetails purchase) async {
    String productId = purchase.productID;
    print('Delivering: $productId');
    // Update user's account in DB
  }

  void dispose() {
    _purchaseSubscription?.cancel();
    _purchaseController.close();
  }
}

// ─── IAP Screen ───
class StorePage extends StatefulWidget {
  const StorePage({super.key});

  @override
  State<StorePage> createState() => _StorePageState();
}

class _StorePageState extends State<StorePage> {
  final IAPService _iap = IAPService();
  bool _isLoading = true;
  String? _purchasingId;

  @override
  void initState() {
    super.initState();
    _init();
  }

  Future<void> _init() async {
    await _iap.initialize();
    setState(() => _isLoading = false);

    // Listen for purchases
    _iap.purchaseStream.listen((purchase) {
      setState(() => _purchasingId = null);

      if (purchase.status == PurchaseStatus.purchased) {
        _showSuccessDialog(purchase.productID);
      }
    });
  }

  void _showSuccessDialog(String productId) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('ซื้อสำเร็จ!'),
        content: Text('คุณได้รับ $productId แล้ว'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('ตกลง'),
          ),
        ],
      ),
    );
  }

  Future<void> _purchase(ProductDetails product) async {
    setState(() => _purchasingId = product.id);

    try {
      bool isConsumable = [
        IAPProducts.coins100,
        IAPProducts.coins500,
        IAPProducts.coins1000,
      ].contains(product.id);

      await _iap.buy(product, isConsumable: isConsumable);
    } catch (e) {
      setState(() => _purchasingId = null);
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('เกิดข้อผิดพลาด: $e')),
        );
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return const Scaffold(body: Center(child: CircularProgressIndicator()));
    }

    if (!_iap.isAvailable) {
      return const Scaffold(
        body: Center(child: Text('In-App Purchase ไม่พร้อมใช้งาน')),
      );
    }

    // Group products by type
    List<ProductDetails> coins = _iap.products
        .where((p) => p.id.startsWith('coins_'))
        .toList();
    List<ProductDetails> oneTime = _iap.products
        .where((p) => [IAPProducts.removeAds, IAPProducts.premiumTheme].contains(p.id))
        .toList();
    List<ProductDetails> subs = _iap.products
        .where((p) => [IAPProducts.monthlyPremium, IAPProducts.yearlyPremium].contains(p.id))
        .toList();

    return Scaffold(
      appBar: AppBar(
        title: const Text('Store'),
        actions: [
          TextButton(
            onPressed: _iap.restorePurchases,
            child: const Text('Restore'),
          ),
        ],
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          // Subscriptions
          if (subs.isNotEmpty) ...[
            const _SectionHeader(title: 'Premium', icon: Icons.star),
            ...subs.map((p) => _ProductCard(
              product: p,
              type: 'subscription',
              isPurchasing: _purchasingId == p.id,
              onBuy: () => _purchase(p),
            )),
          ],

          // One-time
          if (oneTime.isNotEmpty) ...[
            const _SectionHeader(title: 'One-time Purchases', icon: Icons.shop),
            ...oneTime.map((p) => _ProductCard(
              product: p,
              type: 'one-time',
              isPurchasing: _purchasingId == p.id,
              onBuy: () => _purchase(p),
            )),
          ],

          // Coins
          if (coins.isNotEmpty) ...[
            const _SectionHeader(title: 'Coins', icon: Icons.monetization_on),
            ...coins.map((p) => _ProductCard(
              product: p,
              type: 'consumable',
              isPurchasing: _purchasingId == p.id,
              onBuy: () => _purchase(p),
            )),
          ],
        ],
      ),
    );
  }

  @override
  void dispose() {
    _iap.dispose();
    super.dispose();
  }
}

class _SectionHeader extends StatelessWidget {
  final String title;
  final IconData icon;

  const _SectionHeader({required this.title, required this.icon});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 12),
      child: Row(
        children: [
          Icon(icon, size: 20),
          const SizedBox(width: 8),
          Text(title, style: const TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
        ],
      ),
    );
  }
}

class _ProductCard extends StatelessWidget {
  final ProductDetails product;
  final String type;
  final bool isPurchasing;
  final VoidCallback onBuy;

  const _ProductCard({
    required this.product,
    required this.type,
    required this.isPurchasing,
    required this.onBuy,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Row(
          children: [
            // Product info
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(product.title, style: const TextStyle(fontWeight: FontWeight.bold)),
                  Text(product.description, style: TextStyle(color: Colors.grey[600], fontSize: 13)),
                  if (type == 'subscription')
                    const Text('ยกเลิกได้ทุกเมื่อ', style: TextStyle(fontSize: 11, color: Colors.green)),
                ],
              ),
            ),
            // Price & Buy button
            Column(
              crossAxisAlignment: CrossAxisAlignment.end,
              children: [
                Text(
                  product.price,
                  style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 16),
                ),
                const SizedBox(height: 8),
                isPurchasing
                    ? const SizedBox(
                        width: 80,
                        height: 36,
                        child: Center(child: CircularProgressIndicator(strokeWidth: 2)),
                      )
                    : ElevatedButton(
                        onPressed: onBuy,
                        child: const Text('ซื้อ'),
                      ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

---

**← [Part 40 - Biometric & Security](part-40-biometric-security.md)**

**ต่อไป: [Part 42 - Background Tasks →](part-42-background-tasks.md)**
