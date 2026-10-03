# Part 89: Payment Integration
## ขั้นตอนที่ 3441-3480

---

## 🎯 เป้าหมายของ Part นี้

- ผสาน Stripe Payment Sheet เข้ากับ Flutter
- ใช้ PayPal SDK สำหรับการชำระเงิน
- ตั้งค่า Apple Pay / Google Pay (NFC payments)
- จัดการ Webhooks สำหรับ payment events
- ทำความเข้าใจ PCI compliance ใน mobile apps
- เขียนโค้ด production-ready สำหรับ payment flows

---

## ขั้นตอนที่ 3441: Stripe Payment Setup

```yaml
# pubspec.yaml
dependencies:
  flutter_stripe: ^10.1.1
  http: ^1.2.1
  flutter_riverpod: ^2.5.1
  equatable: ^2.0.5
```

```dart
// lib/payments/stripe/stripe_service.dart
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:flutter_stripe/flutter_stripe.dart';
import 'package:http/http.dart' as http;

class StripeConfig {
  final String publishableKey;
  final String merchantId;
  final String serverUrl;

  const StripeConfig({
    required this.publishableKey,
    required this.merchantId,
    required this.serverUrl,
  });
}

class PaymentResult {
  final bool success;
  final String? paymentIntentId;
  final String? errorMessage;
  final PaymentMethod? paymentMethod;

  const PaymentResult({
    required this.success,
    this.paymentIntentId,
    this.errorMessage,
    this.paymentMethod,
  });

  factory PaymentResult.success({
    required String paymentIntentId,
    PaymentMethod? paymentMethod,
  }) {
    return PaymentResult(
      success: true,
      paymentIntentId: paymentIntentId,
      paymentMethod: paymentMethod,
    );
  }

  factory PaymentResult.failure(String message) {
    return PaymentResult(success: false, errorMessage: message);
  }
}

class StripeService {
  final StripeConfig config;
  final http.Client _httpClient;

  StripeService({required this.config, required http.Client httpClient})
      : _httpClient = httpClient;

  static Future<void> initialize(StripeConfig config) async {
    Stripe.publishableKey = config.publishableKey;
    Stripe.merchantIdentifier = config.merchantId;
    await Stripe.instance.applySettings();
  }

  /// Creates a PaymentIntent on your backend and presents the Payment Sheet
  Future<PaymentResult> presentPaymentSheet({
    required double amount, // in smallest currency unit (cents)
    required String currency,
    required String customerId,
    String? customerEmail,
    Map<String, String>? metadata,
  }) async {
    try {
      // 1. Create PaymentIntent on server
      final intentData = await _createPaymentIntent(
        amount: (amount * 100).toInt(), // convert to cents
        currency: currency,
        customerId: customerId,
        metadata: metadata,
      );

      // 2. Initialize the Payment Sheet
      await Stripe.instance.initPaymentSheet(
        paymentSheetParameters: SetupPaymentSheetParameters(
          paymentIntentClientSecret: intentData['client_secret'] as String,
          merchantDisplayName: 'My App Store',
          customerId: intentData['customer'] as String?,
          customerEphemeralKeySecret: intentData['ephemeral_key'] as String?,
          defaultBillingDetails: customerEmail != null
              ? BillingDetails(email: customerEmail)
              : null,
          applePay: const PaymentSheetApplePay(
            merchantCountryCode: 'TH',
          ),
          googlePay: const PaymentSheetGooglePay(
            merchantCountryCode: 'TH',
            testEnv: true,
          ),
          style: ThemeMode.system,
          appearance: const PaymentSheetAppearance(
            colors: PaymentSheetAppearanceColors(
              primary: Color(0xFF6366F1),
            ),
            shapes: PaymentSheetShape(
              borderRadius: 12,
              borderWidth: 1.5,
            ),
          ),
        ),
      );

      // 3. Present the Payment Sheet
      await Stripe.instance.presentPaymentSheet();

      return PaymentResult.success(
        paymentIntentId: intentData['id'] as String,
      );
    } on StripeException catch (e) {
      if (e.error.code == FailureCode.Canceled) {
        return PaymentResult.failure('Payment cancelled by user');
      }
      return PaymentResult.failure(e.error.localizedMessage ?? 'Payment failed');
    } catch (e) {
      return PaymentResult.failure('Unexpected error: $e');
    }
  }

  /// Creates a SetupIntent for saving a payment method without charging
  Future<PaymentResult> setupPaymentMethod({
    required String customerId,
  }) async {
    try {
      final setupData = await _createSetupIntent(customerId: customerId);

      await Stripe.instance.initPaymentSheet(
        paymentSheetParameters: SetupPaymentSheetParameters(
          setupIntentClientSecret: setupData['client_secret'] as String,
          merchantDisplayName: 'My App Store',
          customerId: customerId,
          customerEphemeralKeySecret: setupData['ephemeral_key'] as String?,
        ),
      );

      await Stripe.instance.presentPaymentSheet();
      return PaymentResult.success(paymentIntentId: setupData['id'] as String);
    } on StripeException catch (e) {
      return PaymentResult.failure(e.error.localizedMessage ?? 'Setup failed');
    }
  }

  /// Charge a saved payment method
  Future<PaymentResult> chargePaymentMethod({
    required String paymentMethodId,
    required double amount,
    required String currency,
    required String customerId,
  }) async {
    try {
      final response = await _httpClient.post(
        Uri.parse('${config.serverUrl}/charge'),
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode({
          'payment_method_id': paymentMethodId,
          'amount': (amount * 100).toInt(),
          'currency': currency,
          'customer_id': customerId,
        }),
      );

      if (response.statusCode == 200) {
        final data = jsonDecode(response.body) as Map<String, dynamic>;
        return PaymentResult.success(
          paymentIntentId: data['payment_intent_id'] as String,
        );
      }

      return PaymentResult.failure('Server error: ${response.statusCode}');
    } catch (e) {
      return PaymentResult.failure('Charge failed: $e');
    }
  }

  Future<Map<String, dynamic>> _createPaymentIntent({
    required int amount,
    required String currency,
    required String customerId,
    Map<String, String>? metadata,
  }) async {
    final response = await _httpClient.post(
      Uri.parse('${config.serverUrl}/payment-intent'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({
        'amount': amount,
        'currency': currency,
        'customer_id': customerId,
        if (metadata != null) 'metadata': metadata,
      }),
    );

    if (response.statusCode != 200) {
      throw Exception('Failed to create PaymentIntent: ${response.body}');
    }

    return jsonDecode(response.body) as Map<String, dynamic>;
  }

  Future<Map<String, dynamic>> _createSetupIntent({
    required String customerId,
  }) async {
    final response = await _httpClient.post(
      Uri.parse('${config.serverUrl}/setup-intent'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({'customer_id': customerId}),
    );

    if (response.statusCode != 200) {
      throw Exception('Failed to create SetupIntent: ${response.body}');
    }

    return jsonDecode(response.body) as Map<String, dynamic>;
  }
}
```

## ขั้นตอนที่ 3442: Stripe Payment Screen UI

```dart
// lib/payments/stripe/payment_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Product model for checkout
class CheckoutItem {
  final String id;
  final String name;
  final double price;
  final int quantity;
  final String? imageUrl;

  const CheckoutItem({
    required this.id,
    required this.name,
    required this.price,
    required this.quantity,
    this.imageUrl,
  });

  double get total => price * quantity;
}

class CheckoutScreen extends ConsumerStatefulWidget {
  final List<CheckoutItem> items;
  final String customerId;

  const CheckoutScreen({
    super.key,
    required this.items,
    required this.customerId,
  });

  @override
  ConsumerState<CheckoutScreen> createState() => _CheckoutScreenState();
}

class _CheckoutScreenState extends ConsumerState<CheckoutScreen> {
  bool _isProcessing = false;
  String? _lastError;

  double get subtotal =>
      widget.items.fold(0, (sum, item) => sum + item.total);
  double get tax => subtotal * 0.07; // 7% VAT
  double get total => subtotal + tax;

  Future<void> _processPayment(StripeService stripeService) async {
    setState(() {
      _isProcessing = true;
      _lastError = null;
    });

    try {
      final result = await stripeService.presentPaymentSheet(
        amount: total,
        currency: 'thb',
        customerId: widget.customerId,
        metadata: {
          'item_count': widget.items.length.toString(),
          'order_source': 'mobile_app',
        },
      );

      if (!mounted) return;

      if (result.success) {
        Navigator.of(context).pushReplacement(
          MaterialPageRoute(
            builder: (_) => PaymentSuccessScreen(
              paymentIntentId: result.paymentIntentId!,
              amount: total,
            ),
          ),
        );
      } else {
        setState(() => _lastError = result.errorMessage);
      }
    } finally {
      if (mounted) setState(() => _isProcessing = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Checkout'),
        centerTitle: true,
      ),
      body: Column(
        children: [
          Expanded(
            child: ListView(
              padding: const EdgeInsets.all(16),
              children: [
                // Order items
                Card(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      const Padding(
                        padding: EdgeInsets.all(16),
                        child: Text(
                          'Order Summary',
                          style: TextStyle(
                            fontWeight: FontWeight.bold,
                            fontSize: 16,
                          ),
                        ),
                      ),
                      const Divider(height: 1),
                      ...widget.items.map(
                        (item) => _OrderItemTile(item: item),
                      ),
                    ],
                  ),
                ),
                const SizedBox(height: 16),

                // Totals
                Card(
                  child: Padding(
                    padding: const EdgeInsets.all(16),
                    child: Column(
                      children: [
                        _TotalRow(
                          label: 'Subtotal',
                          amount: subtotal,
                        ),
                        _TotalRow(
                          label: 'VAT (7%)',
                          amount: tax,
                        ),
                        const Divider(),
                        _TotalRow(
                          label: 'Total',
                          amount: total,
                          isBold: true,
                        ),
                      ],
                    ),
                  ),
                ),
                if (_lastError != null) ...[
                  const SizedBox(height: 16),
                  Container(
                    padding: const EdgeInsets.all(12),
                    decoration: BoxDecoration(
                      color: Colors.red.shade50,
                      borderRadius: BorderRadius.circular(8),
                      border: Border.all(color: Colors.red.shade200),
                    ),
                    child: Row(
                      children: [
                        const Icon(Icons.error_outline, color: Colors.red),
                        const SizedBox(width: 8),
                        Expanded(
                          child: Text(
                            _lastError!,
                            style: const TextStyle(color: Colors.red),
                          ),
                        ),
                      ],
                    ),
                  ),
                ],
              ],
            ),
          ),

          // Payment buttons
          SafeArea(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                children: [
                  SizedBox(
                    width: double.infinity,
                    height: 56,
                    child: ElevatedButton(
                      style: ElevatedButton.styleFrom(
                        backgroundColor: const Color(0xFF6366F1),
                        foregroundColor: Colors.white,
                        shape: RoundedRectangleBorder(
                          borderRadius: BorderRadius.circular(12),
                        ),
                      ),
                      onPressed: _isProcessing
                          ? null
                          : () {
                              // In real app: ref.read(stripeServiceProvider)
                              // _processPayment(stripeService)
                            },
                      child: _isProcessing
                          ? const SizedBox(
                              width: 24,
                              height: 24,
                              child: CircularProgressIndicator(
                                color: Colors.white,
                                strokeWidth: 2,
                              ),
                            )
                          : Text(
                              'Pay ฿${total.toStringAsFixed(2)}',
                              style: const TextStyle(
                                fontSize: 18,
                                fontWeight: FontWeight.bold,
                              ),
                            ),
                    ),
                  ),
                  const SizedBox(height: 8),
                  const Row(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      Icon(Icons.lock_outline, size: 14, color: Colors.grey),
                      SizedBox(width: 4),
                      Text(
                        'Secured by Stripe',
                        style: TextStyle(color: Colors.grey, fontSize: 12),
                      ),
                    ],
                  ),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }
}

class _OrderItemTile extends StatelessWidget {
  final CheckoutItem item;
  const _OrderItemTile({required this.item});

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(item.name),
      subtitle: Text('฿${item.price.toStringAsFixed(2)} × ${item.quantity}'),
      trailing: Text(
        '฿${item.total.toStringAsFixed(2)}',
        style: const TextStyle(fontWeight: FontWeight.bold),
      ),
    );
  }
}

class _TotalRow extends StatelessWidget {
  final String label;
  final double amount;
  final bool isBold;

  const _TotalRow({
    required this.label,
    required this.amount,
    this.isBold = false,
  });

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        children: [
          Text(
            label,
            style: isBold
                ? const TextStyle(fontWeight: FontWeight.bold, fontSize: 16)
                : null,
          ),
          Text(
            '฿${amount.toStringAsFixed(2)}',
            style: isBold
                ? const TextStyle(fontWeight: FontWeight.bold, fontSize: 16)
                : null,
          ),
        ],
      ),
    );
  }
}

class PaymentSuccessScreen extends StatelessWidget {
  final String paymentIntentId;
  final double amount;

  const PaymentSuccessScreen({
    super.key,
    required this.paymentIntentId,
    required this.amount,
  });

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: Center(
          child: Padding(
            padding: const EdgeInsets.all(32),
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                Container(
                  width: 80,
                  height: 80,
                  decoration: BoxDecoration(
                    color: Colors.green.shade100,
                    shape: BoxShape.circle,
                  ),
                  child: const Icon(
                    Icons.check_rounded,
                    size: 48,
                    color: Colors.green,
                  ),
                ),
                const SizedBox(height: 24),
                const Text(
                  'Payment Successful!',
                  style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
                ),
                const SizedBox(height: 8),
                Text(
                  '฿${amount.toStringAsFixed(2)}',
                  style: TextStyle(
                    fontSize: 32,
                    fontWeight: FontWeight.bold,
                    color: Colors.green.shade700,
                  ),
                ),
                const SizedBox(height: 16),
                Text(
                  'Transaction ID:\n${paymentIntentId.substring(0, 20)}...',
                  textAlign: TextAlign.center,
                  style: const TextStyle(color: Colors.grey, fontSize: 12),
                ),
                const SizedBox(height: 32),
                ElevatedButton(
                  onPressed: () {
                    Navigator.of(context).popUntil((route) => route.isFirst);
                  },
                  child: const Text('Back to Home'),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

## ขั้นตอนที่ 3443: Apple Pay & Google Pay

```dart
// lib/payments/native_pay/native_pay_service.dart
import 'package:flutter/material.dart';
import 'package:flutter_stripe/flutter_stripe.dart';

class NativePayService {
  static Future<bool> isApplePaySupported() async {
    return Stripe.instance.isApplePaySupported;
  }

  static Future<bool> isGooglePaySupported() async {
    try {
      return await Stripe.instance.isPlatformPaySupported(
        googlePayConfig: const IsGooglePaySupportedParams(
          testEnv: false,
          existingPaymentMethodRequired: false,
        ),
      );
    } catch (_) {
      return false;
    }
  }

  static Future<PaymentResult> presentApplePay({
    required double amount,
    required String currency,
    required String clientSecret,
    List<ApplePayCartSummaryItem>? cartItems,
  }) async {
    try {
      await Stripe.instance.presentApplePay(
        ApplePayPresentParams(
          cartItems: cartItems ??
              [
                ApplePayCartSummaryItem.immediate(
                  label: 'Total',
                  amount: amount.toStringAsFixed(2),
                ),
              ],
          country: 'TH',
          currency: currency.toUpperCase(),
          shippingContact: const ShippingContact(),
          requiredShippingAddressFields: [
            ApplePayContactFieldsType.postalAddress,
          ],
          requiredBillingContactFields: [
            ApplePayContactFieldsType.emailAddress,
          ],
        ),
      );

      await Stripe.instance.confirmApplePayPayment(clientSecret);
      return PaymentResult.success(paymentIntentId: clientSecret);
    } on StripeException catch (e) {
      return PaymentResult.failure(
          e.error.localizedMessage ?? 'Apple Pay failed');
    }
  }

  static Future<PaymentResult> presentGooglePay({
    required double amount,
    required String currency,
    required String clientSecret,
  }) async {
    try {
      await Stripe.instance.confirmPlatformPayPayment(
        clientSecret,
        confirmParams: PlatformPayConfirmParams.googlePay(
          googlePayParams: GooglePayParams(
            testEnv: false,
            merchantName: 'My Store',
            merchantCountryCode: 'TH',
            currencyCode: currency.toUpperCase(),
            isEmailRequired: true,
          ),
        ),
      );

      return PaymentResult.success(paymentIntentId: clientSecret);
    } on StripeException catch (e) {
      return PaymentResult.failure(
          e.error.localizedMessage ?? 'Google Pay failed');
    }
  }
}

// Native Pay Button Widget
class NativePayButton extends StatefulWidget {
  final double amount;
  final String currency;
  final String clientSecret;
  final Function(PaymentResult) onPaymentComplete;

  const NativePayButton({
    super.key,
    required this.amount,
    required this.currency,
    required this.clientSecret,
    required this.onPaymentComplete,
  });

  @override
  State<NativePayButton> createState() => _NativePayButtonState();
}

class _NativePayButtonState extends State<NativePayButton> {
  bool _isApplePayAvailable = false;
  bool _isGooglePayAvailable = false;
  bool _isLoading = false;

  @override
  void initState() {
    super.initState();
    _checkAvailability();
  }

  Future<void> _checkAvailability() async {
    final apple = await NativePayService.isApplePaySupported();
    final google = await NativePayService.isGooglePaySupported();
    if (mounted) {
      setState(() {
        _isApplePayAvailable = apple;
        _isGooglePayAvailable = google;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    if (!_isApplePayAvailable && !_isGooglePayAvailable) {
      return const SizedBox.shrink();
    }

    return Column(
      children: [
        if (_isApplePayAvailable)
          Container(
            margin: const EdgeInsets.symmetric(vertical: 8),
            width: double.infinity,
            height: 48,
            child: ElevatedButton.icon(
              style: ElevatedButton.styleFrom(
                backgroundColor: Colors.black,
                foregroundColor: Colors.white,
                shape: RoundedRectangleBorder(
                  borderRadius: BorderRadius.circular(8),
                ),
              ),
              onPressed: _isLoading ? null : _handleApplePay,
              icon: const Icon(Icons.apple),
              label: Text('Pay ฿${widget.amount.toStringAsFixed(2)}'),
            ),
          ),
        if (_isGooglePayAvailable)
          Container(
            margin: const EdgeInsets.symmetric(vertical: 8),
            width: double.infinity,
            height: 48,
            child: OutlinedButton.icon(
              style: OutlinedButton.styleFrom(
                shape: RoundedRectangleBorder(
                  borderRadius: BorderRadius.circular(8),
                ),
                side: const BorderSide(color: Colors.grey),
              ),
              onPressed: _isLoading ? null : _handleGooglePay,
              icon: const Icon(Icons.g_mobiledata, color: Colors.blue),
              label: Text('Google Pay ฿${widget.amount.toStringAsFixed(2)}'),
            ),
          ),
      ],
    );
  }

  Future<void> _handleApplePay() async {
    setState(() => _isLoading = true);
    final result = await NativePayService.presentApplePay(
      amount: widget.amount,
      currency: widget.currency,
      clientSecret: widget.clientSecret,
    );
    setState(() => _isLoading = false);
    widget.onPaymentComplete(result);
  }

  Future<void> _handleGooglePay() async {
    setState(() => _isLoading = true);
    final result = await NativePayService.presentGooglePay(
      amount: widget.amount,
      currency: widget.currency,
      clientSecret: widget.clientSecret,
    );
    setState(() => _isLoading = false);
    widget.onPaymentComplete(result);
  }
}
```

## ขั้นตอนที่ 3444: Webhook Handling

```dart
// lib/payments/webhooks/webhook_handler.dart
import 'dart:convert';
import 'package:flutter/foundation.dart';
import 'package:http/http.dart' as http;
import 'package:crypto/crypto.dart';

enum WebhookEventType {
  paymentIntentSucceeded,
  paymentIntentFailed,
  paymentIntentCanceled,
  chargeRefunded,
  subscriptionCreated,
  subscriptionUpdated,
  subscriptionDeleted,
  invoicePaid,
  invoicePaymentFailed,
  unknown,
}

class WebhookEvent {
  final String id;
  final WebhookEventType type;
  final Map<String, dynamic> data;
  final DateTime createdAt;
  final bool livemode;

  const WebhookEvent({
    required this.id,
    required this.type,
    required this.data,
    required this.createdAt,
    required this.livemode,
  });

  factory WebhookEvent.fromJson(Map<String, dynamic> json) {
    final typeStr = json['type'] as String? ?? '';
    final type = _parseEventType(typeStr);

    return WebhookEvent(
      id: json['id'] as String,
      type: type,
      data: json['data'] as Map<String, dynamic>? ?? {},
      createdAt: DateTime.fromMillisecondsSinceEpoch(
        ((json['created'] as int?) ?? 0) * 1000,
      ),
      livemode: json['livemode'] as bool? ?? false,
    );
  }

  static WebhookEventType _parseEventType(String type) {
    switch (type) {
      case 'payment_intent.succeeded':
        return WebhookEventType.paymentIntentSucceeded;
      case 'payment_intent.payment_failed':
        return WebhookEventType.paymentIntentFailed;
      case 'payment_intent.canceled':
        return WebhookEventType.paymentIntentCanceled;
      case 'charge.refunded':
        return WebhookEventType.chargeRefunded;
      case 'customer.subscription.created':
        return WebhookEventType.subscriptionCreated;
      case 'customer.subscription.updated':
        return WebhookEventType.subscriptionUpdated;
      case 'customer.subscription.deleted':
        return WebhookEventType.subscriptionDeleted;
      case 'invoice.paid':
        return WebhookEventType.invoicePaid;
      case 'invoice.payment_failed':
        return WebhookEventType.invoicePaymentFailed;
      default:
        return WebhookEventType.unknown;
    }
  }
}

/// Server-side webhook handler (Dart backend / Cloud Function)
class StripeWebhookHandler {
  final String webhookSecret;
  final Map<WebhookEventType, Future<void> Function(WebhookEvent)> _handlers;

  StripeWebhookHandler({
    required this.webhookSecret,
  }) : _handlers = {};

  void on(
    WebhookEventType eventType,
    Future<void> Function(WebhookEvent event) handler,
  ) {
    _handlers[eventType] = handler;
  }

  Future<http.Response> handleRequest(http.Request request) async {
    final signature = request.headers['stripe-signature'];
    if (signature == null) {
      return http.Response('Missing signature', 400);
    }

    final body = await request.readAsString();

    if (!_verifySignature(body, signature)) {
      return http.Response('Invalid signature', 400);
    }

    try {
      final json = jsonDecode(body) as Map<String, dynamic>;
      final event = WebhookEvent.fromJson(json);

      final handler = _handlers[event.type];
      if (handler != null) {
        await handler(event);
      }

      return http.Response(jsonEncode({'received': true}), 200);
    } catch (e) {
      debugPrint('Webhook processing error: $e');
      return http.Response('Processing error', 500);
    }
  }

  bool _verifySignature(String payload, String signatureHeader) {
    // Parse the signature header: t=timestamp,v1=signature
    final parts = signatureHeader.split(',');
    String? timestamp;
    String? signature;

    for (final part in parts) {
      if (part.startsWith('t=')) timestamp = part.substring(2);
      if (part.startsWith('v1=')) signature = part.substring(3);
    }

    if (timestamp == null || signature == null) return false;

    // Verify timestamp is within 5 minutes
    final ts = int.tryParse(timestamp) ?? 0;
    final eventTime = DateTime.fromMillisecondsSinceEpoch(ts * 1000);
    if (DateTime.now().difference(eventTime).abs().inMinutes > 5) {
      return false;
    }

    // Compute expected signature
    final signedPayload = '$timestamp.$payload';
    final hmac = Hmac(sha256, utf8.encode(webhookSecret));
    final digest = hmac.convert(utf8.encode(signedPayload));
    final expectedSignature = digest.toString();

    return signature == expectedSignature;
  }
}

// Example backend usage
void setupWebhookHandlers(StripeWebhookHandler handler) {
  handler.on(WebhookEventType.paymentIntentSucceeded, (event) async {
    final paymentIntent = event.data['object'] as Map<String, dynamic>;
    final paymentIntentId = paymentIntent['id'] as String;
    final amount = paymentIntent['amount'] as int;
    final metadata = paymentIntent['metadata'] as Map<String, dynamic>?;

    debugPrint('Payment succeeded: $paymentIntentId, amount: $amount');
    // Update order status in database
    // Send confirmation email
    // Trigger fulfillment
  });

  handler.on(WebhookEventType.paymentIntentFailed, (event) async {
    final paymentIntent = event.data['object'] as Map<String, dynamic>;
    debugPrint('Payment failed: ${paymentIntent['id']}');
    // Notify user, update order status
  });

  handler.on(WebhookEventType.chargeRefunded, (event) async {
    final charge = event.data['object'] as Map<String, dynamic>;
    debugPrint('Charge refunded: ${charge['id']}');
    // Process refund in your system
  });
}
```

## ขั้นตอนที่ 3445: PCI Compliance in Mobile

```dart
// lib/payments/security/pci_compliance.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

/// PCI DSS Compliance Guidelines for Mobile Flutter Apps
///
/// Key rules:
/// 1. NEVER store raw card numbers (PAN) on device
/// 2. NEVER log card data
/// 3. Use tokenization (Stripe/Braintree handles this)
/// 4. Secure network transmission (HTTPS only)
/// 5. Protect the app from reverse engineering

class PCICompliantCardInput extends StatefulWidget {
  final Function(String token) onTokenized;

  const PCICompliantCardInput({super.key, required this.onTokenized});

  @override
  State<PCICompliantCardInput> createState() => _PCICompliantCardInputState();
}

class _PCICompliantCardInputState extends State<PCICompliantCardInput> {
  final _cardController = CardEditController();
  bool _isComplete = false;

  @override
  void initState() {
    super.initState();
    _cardController.addListener(() {
      setState(() => _isComplete = _cardController.complete);
    });
  }

  @override
  void dispose() {
    _cardController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.stretch,
      children: [
        // Use Stripe's pre-built card widget — card numbers NEVER touch your code
        CardField(
          controller: _cardController,
          decoration: const InputDecoration(
            border: OutlineInputBorder(),
            labelText: 'Card Details',
          ),
          onCardChanged: (card) {
            // card.complete — do NOT log card.number!
            setState(() => _isComplete = card?.complete ?? false);
          },
        ),
        const SizedBox(height: 8),
        const _PCINotice(),
        const SizedBox(height: 16),
        ElevatedButton(
          onPressed: _isComplete ? _tokenize : null,
          child: const Text('Add Card'),
        ),
      ],
    );
  }

  Future<void> _tokenize() async {
    try {
      // Create token — Stripe SDK handles this, card data goes directly to Stripe
      // Your server never sees raw card data
      final paymentMethod = await Stripe.instance.createPaymentMethod(
        params: const PaymentMethodParams.card(
          paymentMethodData: PaymentMethodData(),
        ),
      );

      // Only the token/payment method ID is passed to your backend
      widget.onTokenized(paymentMethod.id);
    } on StripeException catch (e) {
      debugPrint('Tokenization failed: ${e.error.localizedMessage}');
    }
  }
}

class _PCINotice extends StatelessWidget {
  const _PCINotice();

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(12),
      decoration: BoxDecoration(
        color: Colors.blue.shade50,
        borderRadius: BorderRadius.circular(8),
      ),
      child: const Row(
        children: [
          Icon(Icons.security, size: 16, color: Colors.blue),
          SizedBox(width: 8),
          Expanded(
            child: Text(
              'Your card is encrypted and securely processed by Stripe. We never store your full card number.',
              style: TextStyle(fontSize: 12, color: Colors.blue),
            ),
          ),
        ],
      ),
    );
  }
}

/// Security utilities for payment flows
class PaymentSecurityUtils {
  /// Mask a card number for display (show only last 4 digits)
  static String maskCardNumber(String last4) => '•••• •••• •••• $last4';

  /// Validate that no card data is being sent to analytics or logs
  static void assertNoPANInData(Map<String, dynamic> data) {
    final dataStr = data.toString();
    // Basic regex to detect what looks like a card number
    final cardPattern = RegExp(r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b');
    assert(
      !cardPattern.hasMatch(dataStr),
      'PCI VIOLATION: Card number detected in data payload!',
    );
  }

  /// Check that the app uses certificate pinning (conceptual)
  static bool verifyCertificatePinning() {
    // Implement with http_certificate_pinning package
    // Return false if pinning fails
    return true;
  }

  /// Prevent screenshots on payment screens (Android)
  static Future<void> secureScreen(bool secure) async {
    try {
      await SystemChannels.platform.invokeMethod(
        'SystemNavigator.setSecure',
        secure,
      );
    } catch (_) {
      // Platform may not support this
    }
  }
}

// Secure Payment Wrapper Widget
class SecurePaymentWrapper extends StatefulWidget {
  final Widget child;

  const SecurePaymentWrapper({super.key, required this.child});

  @override
  State<SecurePaymentWrapper> createState() => _SecurePaymentWrapperState();
}

class _SecurePaymentWrapperState extends State<SecurePaymentWrapper> {
  @override
  void initState() {
    super.initState();
    PaymentSecurityUtils.secureScreen(true);
  }

  @override
  void dispose() {
    PaymentSecurityUtils.secureScreen(false);
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return widget.child;
  }
}
```

---

**← [Part 88](part-88-advanced-database.md)**
**ต่อไป: [Part 90 →](part-90-release-checklist.md)**
