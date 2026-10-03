# Part 75: Monetization Strategies ใน Flutter
## ขั้นตอนที่ 2881-2920

## 🎯 เป้าหมายของ Part นี้
- สร้าง Freemium Model Implementation
- Integrate Google Mobile Ads (Banner, Interstitial, Rewarded)
- ทำ Ad Frequency Capping
- รวม Ads + IAP เข้าด้วยกัน
- Full working Flutter code สำหรับทุก monetization strategy

---

## ขั้นตอนที่ 2881: Setup Monetization Dependencies

```yaml
# pubspec.yaml
name: monetized_flutter_app
description: Flutter app with ads and in-app purchases
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter
  google_mobile_ads: ^5.1.0
  in_app_purchase: ^3.2.0
  shared_preferences: ^2.3.2
  provider: ^6.1.2
  intl: ^0.19.0
  flutter_riverpod: ^2.5.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0

flutter:
  uses-material-design: true
```

---

## ขั้นตอนที่ 2882: Subscription & Freemium Model

```dart
// lib/models/subscription.dart

enum SubscriptionTier {
  free,
  basic,
  premium,
  lifetime,
}

extension SubscriptionTierInfo on SubscriptionTier {
  String get displayName => switch (this) {
    SubscriptionTier.free => 'Free',
    SubscriptionTier.basic => 'Basic',
    SubscriptionTier.premium => 'Premium',
    SubscriptionTier.lifetime => 'Lifetime',
  };

  String get price => switch (this) {
    SubscriptionTier.free => 'Free',
    SubscriptionTier.basic => '฿59/month',
    SubscriptionTier.premium => '฿149/month',
    SubscriptionTier.lifetime => '฿999',
  };

  List<String> get features => switch (this) {
    SubscriptionTier.free => [
      '5 projects',
      'Basic features',
      'Ads supported',
      'Community support',
    ],
    SubscriptionTier.basic => [
      '20 projects',
      'All basic features',
      'No ads',
      'Email support',
      'Export to PDF',
    ],
    SubscriptionTier.premium => [
      'Unlimited projects',
      'All premium features',
      'No ads',
      'Priority support',
      'All export formats',
      'Advanced analytics',
      'Team collaboration',
    ],
    SubscriptionTier.lifetime => [
      'Everything in Premium',
      'Lifetime access',
      'Future updates included',
      'Dedicated support',
    ],
  };

  int get projectLimit => switch (this) {
    SubscriptionTier.free => 5,
    SubscriptionTier.basic => 20,
    SubscriptionTier.premium => 999999,
    SubscriptionTier.lifetime => 999999,
  };

  bool get hasAds => this == SubscriptionTier.free;
  bool get hasAdvancedFeatures =>
      this == SubscriptionTier.premium || this == SubscriptionTier.lifetime;
}

class UserSubscription {
  final SubscriptionTier tier;
  final DateTime? expiresAt;
  final bool isActive;
  final String? purchaseToken;

  const UserSubscription({
    required this.tier,
    this.expiresAt,
    required this.isActive,
    this.purchaseToken,
  });

  const UserSubscription.free()
      : tier = SubscriptionTier.free,
        expiresAt = null,
        isActive = true,
        purchaseToken = null;

  bool get isExpired {
    if (tier == SubscriptionTier.free || tier == SubscriptionTier.lifetime) {
      return false;
    }
    return expiresAt?.isBefore(DateTime.now()) ?? true;
  }

  bool get canUseFeature(String featureName) {
    // Check if current subscription allows the feature
    return isActive && !isExpired;
  }

  UserSubscription copyWith({
    SubscriptionTier? tier,
    DateTime? expiresAt,
    bool? isActive,
    String? purchaseToken,
  }) {
    return UserSubscription(
      tier: tier ?? this.tier,
      expiresAt: expiresAt ?? this.expiresAt,
      isActive: isActive ?? this.isActive,
      purchaseToken: purchaseToken ?? this.purchaseToken,
    );
  }
}
```

---

## ขั้นตอนที่ 2883: Subscription Service

```dart
// lib/services/subscription_service.dart
import 'dart:async';
import 'package:flutter/foundation.dart';
import 'package:in_app_purchase/in_app_purchase.dart';
import 'package:shared_preferences/shared_preferences.dart';
import '../models/subscription.dart';

class SubscriptionService extends ChangeNotifier {
  SubscriptionService._();
  static final SubscriptionService instance = SubscriptionService._();

  // Product IDs from App Store / Play Store
  static const String basicMonthlyId = 'basic_monthly';
  static const String premiumMonthlyId = 'premium_monthly';
  static const String lifetimeId = 'lifetime_purchase';

  static const Set<String> _productIds = {
    basicMonthlyId,
    premiumMonthlyId,
    lifetimeId,
  };

  final InAppPurchase _inAppPurchase = InAppPurchase.instance;
  StreamSubscription<List<PurchaseDetails>>? _subscription;

  UserSubscription _subscription_ = const UserSubscription.free();
  List<ProductDetails> _products = [];
  bool _isLoading = false;
  String? _errorMessage;

  UserSubscription get subscription => _subscription_;
  List<ProductDetails> get products => _products;
  bool get isLoading => _isLoading;
  String? get errorMessage => _errorMessage;
  SubscriptionTier get tier => _subscription_.tier;

  Future<void> initialize() async {
    // Load saved subscription
    await _loadSavedSubscription();

    // Check if IAP is available
    final bool available = await _inAppPurchase.isAvailable();
    if (!available) {
      debugPrint('SubscriptionService: IAP not available');
      return;
    }

    // Listen to purchase stream
    final purchaseUpdated = _inAppPurchase.purchaseStream;
    _subscription = purchaseUpdated.listen(
      _handlePurchaseUpdate,
      onDone: () => _subscription?.cancel(),
      onError: (error) => debugPrint('Purchase stream error: $error'),
    );

    // Load products
    await _loadProducts();
  }

  Future<void> _loadProducts() async {
    try {
      final ProductDetailsResponse response =
          await _inAppPurchase.queryProductDetails(_productIds);

      if (response.error != null) {
        _errorMessage = response.error!.message;
        notifyListeners();
        return;
      }

      _products = response.productDetails;
      notifyListeners();
      debugPrint('SubscriptionService: Loaded ${_products.length} products');
    } catch (e) {
      _errorMessage = e.toString();
      notifyListeners();
    }
  }

  Future<bool> purchaseProduct(String productId) async {
    final product = _products.firstWhere(
      (p) => p.id == productId,
      orElse: () => throw StateError('Product $productId not found'),
    );

    _isLoading = true;
    notifyListeners();

    try {
      final PurchaseParam purchaseParam = PurchaseParam(
        productDetails: product,
      );

      if (productId == lifetimeId) {
        return await _inAppPurchase.buyNonConsumable(
          purchaseParam: purchaseParam,
        );
      } else {
        return await _inAppPurchase.buyNonConsumable(
          purchaseParam: purchaseParam,
        );
      }
    } catch (e) {
      _errorMessage = e.toString();
      _isLoading = false;
      notifyListeners();
      return false;
    }
  }

  Future<void> restorePurchases() async {
    _isLoading = true;
    notifyListeners();

    try {
      await _inAppPurchase.restorePurchases();
    } catch (e) {
      _errorMessage = e.toString();
    } finally {
      _isLoading = false;
      notifyListeners();
    }
  }

  void _handlePurchaseUpdate(List<PurchaseDetails> purchaseDetailsList) async {
    for (final purchase in purchaseDetailsList) {
      switch (purchase.status) {
        case PurchaseStatus.pending:
          _isLoading = true;

        case PurchaseStatus.purchased:
        case PurchaseStatus.restored:
          await _verifyAndActivate(purchase);

        case PurchaseStatus.error:
          _errorMessage = purchase.error?.message;
          _isLoading = false;

        case PurchaseStatus.canceled:
          _isLoading = false;
      }

      if (purchase.pendingCompletePurchase) {
        await _inAppPurchase.completePurchase(purchase);
      }
    }
    notifyListeners();
  }

  Future<void> _verifyAndActivate(PurchaseDetails purchase) async {
    // In production: verify receipt with your server
    // Here we just activate based on product ID
    SubscriptionTier newTier;

    switch (purchase.productID) {
      case basicMonthlyId:
        newTier = SubscriptionTier.basic;
      case premiumMonthlyId:
        newTier = SubscriptionTier.premium;
      case lifetimeId:
        newTier = SubscriptionTier.lifetime;
      default:
        return;
    }

    _subscription_ = UserSubscription(
      tier: newTier,
      expiresAt: newTier == SubscriptionTier.lifetime
          ? null
          : DateTime.now().add(const Duration(days: 30)),
      isActive: true,
      purchaseToken: purchase.verificationData.localVerificationData,
    );

    await _saveSubscription();
    _isLoading = false;
    notifyListeners();
  }

  Future<void> _saveSubscription() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('subscription_tier', _subscription_.tier.name);
    if (_subscription_.expiresAt != null) {
      await prefs.setString(
        'subscription_expires',
        _subscription_.expiresAt!.toIso8601String(),
      );
    }
  }

  Future<void> _loadSavedSubscription() async {
    final prefs = await SharedPreferences.getInstance();
    final tierName = prefs.getString('subscription_tier');
    if (tierName == null) return;

    try {
      final tier = SubscriptionTier.values.byName(tierName);
      final expiresStr = prefs.getString('subscription_expires');
      final expiresAt = expiresStr != null ? DateTime.parse(expiresStr) : null;

      _subscription_ = UserSubscription(
        tier: tier,
        expiresAt: expiresAt,
        isActive: true,
      );
      notifyListeners();
    } catch (e) {
      debugPrint('Error loading subscription: $e');
    }
  }

  @override
  void dispose() {
    _subscription?.cancel();
    super.dispose();
  }
}
```

---

## ขั้นตอนที่ 2884: Google Mobile Ads Service

```dart
// lib/services/ads_service.dart
import 'dart:async';
import 'package:flutter/foundation.dart';
import 'package:google_mobile_ads/google_mobile_ads.dart';
import 'package:shared_preferences/shared_preferences.dart';

/// Manages all ad lifecycle and frequency capping
class AdsService {
  AdsService._();
  static final AdsService instance = AdsService._();

  // Test Ad Unit IDs (replace with real ones in production)
  static const String _bannerAdUnitIdAndroid =
      'ca-app-pub-3940256099942544/6300978111';
  static const String _bannerAdUnitIdIOS =
      'ca-app-pub-3940256099942544/2934735716';
  static const String _interstitialAdUnitIdAndroid =
      'ca-app-pub-3940256099942544/1033173712';
  static const String _interstitialAdUnitIdIOS =
      'ca-app-pub-3940256099942544/4411468910';
  static const String _rewardedAdUnitIdAndroid =
      'ca-app-pub-3940256099942544/5224354917';
  static const String _rewardedAdUnitIdIOS =
      'ca-app-pub-3940256099942544/1712485313';

  InterstitialAd? _interstitialAd;
  RewardedAd? _rewardedAd;

  bool _isInitialized = false;
  bool _adsEnabled = true;

  // Frequency capping
  static const int _maxInterstitialsPerHour = 3;
  static const Duration _minInterstitialInterval = Duration(minutes: 5);
  final List<DateTime> _interstitialShowTimes = [];
  DateTime? _lastInterstitialTime;

  // Callbacks
  Function(RewardItem reward)? onRewardEarned;

  bool get isInitialized => _isInitialized;
  bool get adsEnabled => _adsEnabled;

  Future<void> initialize() async {
    if (_isInitialized) return;

    await MobileAds.instance.initialize();

    // Request tracking authorization on iOS (required for personalized ads)
    if (defaultTargetPlatform == TargetPlatform.iOS) {
      // UMP (User Messaging Platform) for GDPR compliance
      await _requestConsentInfo();
    }

    _isInitialized = true;
    debugPrint('AdsService: Initialized');

    // Pre-load ads
    await Future.wait([
      _loadInterstitialAd(),
      _loadRewardedAd(),
    ]);
  }

  void setAdsEnabled(bool enabled) {
    _adsEnabled = enabled;
    if (!enabled) {
      _interstitialAd?.dispose();
      _interstitialAd = null;
      _rewardedAd?.dispose();
      _rewardedAd = null;
    }
  }

  // ─── Banner Ad ───────────────────────────────────────────────────────────

  BannerAd createBannerAd() {
    return BannerAd(
      adUnitId: defaultTargetPlatform == TargetPlatform.android
          ? _bannerAdUnitIdAndroid
          : _bannerAdUnitIdIOS,
      size: AdSize.banner,
      request: const AdRequest(),
      listener: BannerAdListener(
        onAdLoaded: (ad) => debugPrint('Banner ad loaded'),
        onAdFailedToLoad: (ad, error) {
          debugPrint('Banner ad failed: $error');
          ad.dispose();
        },
        onAdOpened: (ad) => debugPrint('Banner ad opened'),
        onAdClosed: (ad) => debugPrint('Banner ad closed'),
      ),
    );
  }

  // ─── Interstitial Ad ──────────────────────────────────────────────────────

  Future<void> _loadInterstitialAd() async {
    if (!_adsEnabled) return;

    await InterstitialAd.load(
      adUnitId: defaultTargetPlatform == TargetPlatform.android
          ? _interstitialAdUnitIdAndroid
          : _interstitialAdUnitIdIOS,
      request: const AdRequest(),
      adLoadCallback: InterstitialAdLoadCallback(
        onAdLoaded: (ad) {
          _interstitialAd = ad;
          ad.setImmersiveMode(true);
          debugPrint('Interstitial ad loaded');
        },
        onAdFailedToLoad: (error) {
          debugPrint('Interstitial failed to load: $error');
          _interstitialAd = null;
        },
      ),
    );
  }

  /// Show interstitial ad if frequency cap allows
  Future<bool> showInterstitialAd() async {
    if (!_adsEnabled || _interstitialAd == null) return false;
    if (!_canShowInterstitial()) return false;

    final completer = Completer<bool>();

    _interstitialAd!.fullScreenContentCallback = FullScreenContentCallback(
      onAdShowedFullScreenContent: (ad) {
        _recordInterstitialShown();
        debugPrint('Interstitial ad shown');
      },
      onAdDismissedFullScreenContent: (ad) {
        ad.dispose();
        _interstitialAd = null;
        _loadInterstitialAd(); // Preload next one
        completer.complete(true);
      },
      onAdFailedToShowFullScreenContent: (ad, error) {
        ad.dispose();
        _interstitialAd = null;
        _loadInterstitialAd();
        completer.complete(false);
      },
    );

    await _interstitialAd!.show();
    return completer.future;
  }

  bool _canShowInterstitial() {
    final now = DateTime.now();

    // Min interval between ads
    if (_lastInterstitialTime != null) {
      final elapsed = now.difference(_lastInterstitialTime!);
      if (elapsed < _minInterstitialInterval) return false;
    }

    // Max per hour
    final oneHourAgo = now.subtract(const Duration(hours: 1));
    final recentShows = _interstitialShowTimes
        .where((t) => t.isAfter(oneHourAgo))
        .length;
    return recentShows < _maxInterstitialsPerHour;
  }

  void _recordInterstitialShown() {
    final now = DateTime.now();
    _lastInterstitialTime = now;
    _interstitialShowTimes.add(now);

    // Keep only last 24 hours
    final yesterday = now.subtract(const Duration(hours: 24));
    _interstitialShowTimes.removeWhere((t) => t.isBefore(yesterday));
  }

  // ─── Rewarded Ad ──────────────────────────────────────────────────────────

  Future<void> _loadRewardedAd() async {
    if (!_adsEnabled) return;

    await RewardedAd.load(
      adUnitId: defaultTargetPlatform == TargetPlatform.android
          ? _rewardedAdUnitIdAndroid
          : _rewardedAdUnitIdIOS,
      request: const AdRequest(),
      rewardedAdLoadCallback: RewardedAdLoadCallback(
        onAdLoaded: (ad) {
          _rewardedAd = ad;
          debugPrint('Rewarded ad loaded');
        },
        onAdFailedToLoad: (error) {
          debugPrint('Rewarded ad failed to load: $error');
          _rewardedAd = null;
        },
      ),
    );
  }

  Future<RewardItem?> showRewardedAd() async {
    if (!_adsEnabled || _rewardedAd == null) return null;

    final completer = Completer<RewardItem?>();
    RewardItem? earnedReward;

    _rewardedAd!.fullScreenContentCallback = FullScreenContentCallback(
      onAdShowedFullScreenContent: (ad) {
        debugPrint('Rewarded ad shown');
      },
      onAdDismissedFullScreenContent: (ad) {
        ad.dispose();
        _rewardedAd = null;
        _loadRewardedAd(); // Preload next
        completer.complete(earnedReward);
      },
      onAdFailedToShowFullScreenContent: (ad, error) {
        ad.dispose();
        _rewardedAd = null;
        _loadRewardedAd();
        completer.complete(null);
      },
    );

    await _rewardedAd!.show(
      onUserEarnedReward: (ad, reward) {
        earnedReward = reward;
        onRewardEarned?.call(reward);
        debugPrint('Earned reward: ${reward.amount} ${reward.type}');
      },
    );

    return completer.future;
  }

  bool get isRewardedAdReady => _rewardedAd != null;
  bool get isInterstitialAdReady => _interstitialAd != null;

  Future<void> _requestConsentInfo() async {
    // Placeholder for UMP SDK integration
    debugPrint('AdsService: Requesting consent info (iOS)');
  }

  void dispose() {
    _interstitialAd?.dispose();
    _rewardedAd?.dispose();
  }
}
```

---

## ขั้นตอนที่ 2885: Banner Ad Widget

```dart
// lib/widgets/ad_banner_widget.dart
import 'package:flutter/material.dart';
import 'package:google_mobile_ads/google_mobile_ads.dart';
import '../services/ads_service.dart';
import '../services/subscription_service.dart';

class AdBannerWidget extends StatefulWidget {
  const AdBannerWidget({super.key});

  @override
  State<AdBannerWidget> createState() => _AdBannerWidgetState();
}

class _AdBannerWidgetState extends State<AdBannerWidget> {
  BannerAd? _bannerAd;
  bool _isAdLoaded = false;

  @override
  void initState() {
    super.initState();
    _loadAd();
  }

  @override
  void dispose() {
    _bannerAd?.dispose();
    super.dispose();
  }

  void _loadAd() {
    // Don't show ads to paid users
    final subscription = SubscriptionService.instance.subscription;
    if (!subscription.tier.hasAds) return;

    final ad = AdsService.instance.createBannerAd();
    ad.load().then((_) {
      if (mounted) {
        setState(() {
          _bannerAd = ad;
          _isAdLoaded = true;
        });
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    // Hide for premium users
    final subscription = SubscriptionService.instance.subscription;
    if (!subscription.tier.hasAds) return const SizedBox.shrink();

    if (!_isAdLoaded || _bannerAd == null) {
      return Container(
        height: 50,
        color: Colors.grey[100],
        child: const Center(
          child: Text(
            'Advertisement',
            style: TextStyle(color: Colors.grey, fontSize: 12),
          ),
        ),
      );
    }

    return Container(
      alignment: Alignment.center,
      width: _bannerAd!.size.width.toDouble(),
      height: _bannerAd!.size.height.toDouble(),
      child: AdWidget(ad: _bannerAd!),
    );
  }
}

/// Smart banner that adjusts size to screen width
class SmartBannerWidget extends StatefulWidget {
  const SmartBannerWidget({super.key});

  @override
  State<SmartBannerWidget> createState() => _SmartBannerWidgetState();
}

class _SmartBannerWidgetState extends State<SmartBannerWidget> {
  BannerAd? _bannerAd;
  bool _isAdLoaded = false;

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    _loadSmartBannerAd();
  }

  @override
  void dispose() {
    _bannerAd?.dispose();
    super.dispose();
  }

  Future<void> _loadSmartBannerAd() async {
    if (!SubscriptionService.instance.subscription.tier.hasAds) return;

    final adSize = await AdSize.getCurrentOrientationAnchoredAdaptiveBannerAdSize(
      MediaQuery.sizeOf(context).width.truncate(),
    );

    if (adSize == null) return;

    final bannerAd = BannerAd(
      adUnitId: 'ca-app-pub-3940256099942544/6300978111',
      size: adSize,
      request: const AdRequest(),
      listener: BannerAdListener(
        onAdLoaded: (ad) {
          if (!mounted) {
            ad.dispose();
            return;
          }
          setState(() {
            _bannerAd = ad as BannerAd;
            _isAdLoaded = true;
          });
        },
        onAdFailedToLoad: (ad, error) {
          ad.dispose();
          debugPrint('Banner failed: $error');
        },
      ),
    );

    await bannerAd.load();
  }

  @override
  Widget build(BuildContext context) {
    if (!SubscriptionService.instance.subscription.tier.hasAds) {
      return const SizedBox.shrink();
    }

    if (!_isAdLoaded || _bannerAd == null) return const SizedBox(height: 50);

    return SizedBox(
      width: _bannerAd!.size.width.toDouble(),
      height: _bannerAd!.size.height.toDouble(),
      child: AdWidget(ad: _bannerAd!),
    );
  }
}
```

---

## ขั้นตอนที่ 2886: Rewarded Ad Integration with Game/Content

```dart
// lib/widgets/rewarded_ad_gate.dart
import 'package:flutter/material.dart';
import '../services/ads_service.dart';
import '../services/subscription_service.dart';

/// Gate premium content behind rewarded ad OR subscription
class RewardedAdGate extends StatefulWidget {
  final Widget lockedContent;
  final Widget Function(BuildContext context) unlockedContent;
  final String rewardDescription;

  const RewardedAdGate({
    super.key,
    required this.lockedContent,
    required this.unlockedContent,
    this.rewardDescription = 'Watch an ad to unlock this content',
  });

  @override
  State<RewardedAdGate> createState() => _RewardedAdGateState();
}

class _RewardedAdGateState extends State<RewardedAdGate> {
  bool _isUnlocked = false;
  bool _isWatchingAd = false;

  bool get _hasSubscription =>
      !SubscriptionService.instance.subscription.tier.hasAds;

  Future<void> _watchAdForReward() async {
    setState(() => _isWatchingAd = true);

    final reward = await AdsService.instance.showRewardedAd();

    if (reward != null) {
      setState(() {
        _isUnlocked = true;
        _isWatchingAd = false;
      });

      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            content: Text(
              'Reward earned: ${reward.amount.toInt()} ${reward.type}!',
            ),
            backgroundColor: Colors.green,
          ),
        );
      }
    } else {
      setState(() => _isWatchingAd = false);
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('Ad not available. Try again later.'),
          ),
        );
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    // Premium users always see content
    if (_hasSubscription || _isUnlocked) {
      return widget.unlockedContent(context);
    }

    return Stack(
      children: [
        // Blurred/locked content preview
        Opacity(opacity: 0.3, child: widget.lockedContent),

        // Overlay
        Container(
          decoration: BoxDecoration(
            color: Colors.black.withOpacity(0.5),
            borderRadius: BorderRadius.circular(12),
          ),
          child: Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                const Icon(Icons.lock, size: 48, color: Colors.white),
                const SizedBox(height: 12),
                Padding(
                  padding: const EdgeInsets.symmetric(horizontal: 24),
                  child: Text(
                    widget.rewardDescription,
                    textAlign: TextAlign.center,
                    style: const TextStyle(color: Colors.white, fontSize: 14),
                  ),
                ),
                const SizedBox(height: 20),
                if (_isWatchingAd)
                  const CircularProgressIndicator(color: Colors.white)
                else ...[
                  ElevatedButton.icon(
                    onPressed: AdsService.instance.isRewardedAdReady
                        ? _watchAdForReward
                        : null,
                    icon: const Icon(Icons.play_circle),
                    label: const Text('Watch Ad (Free)'),
                    style: ElevatedButton.styleFrom(
                      backgroundColor: Colors.orange,
                      foregroundColor: Colors.white,
                    ),
                  ),
                  const SizedBox(height: 8),
                  TextButton(
                    onPressed: () => _showUpgradeDialog(),
                    child: const Text(
                      'Or Upgrade to Premium',
                      style: TextStyle(color: Colors.white70),
                    ),
                  ),
                ],
              ],
            ),
          ),
        ),
      ],
    );
  }

  void _showUpgradeDialog() {
    showDialog(
      context: context,
      builder: (_) => const UpgradeDialog(),
    );
  }
}
```

---

## ขั้นตอนที่ 2887: Upgrade/Paywall Screen

```dart
// lib/screens/upgrade_screen.dart
import 'package:flutter/material.dart';
import '../models/subscription.dart';
import '../services/subscription_service.dart';

class UpgradeScreen extends StatefulWidget {
  const UpgradeScreen({super.key});

  @override
  State<UpgradeScreen> createState() => _UpgradeScreenState();
}

class _UpgradeScreenState extends State<UpgradeScreen> {
  SubscriptionTier _selectedTier = SubscriptionTier.premium;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Upgrade'),
        leading: IconButton(
          icon: const Icon(Icons.close),
          onPressed: () => Navigator.pop(context),
        ),
      ),
      body: SingleChildScrollView(
        child: Column(
          children: [
            _buildHeader(),
            _buildTierCards(),
            _buildFeatureComparison(),
            _buildPurchaseButton(),
            _buildRestoreButton(),
            _buildLegalText(),
          ],
        ),
      ),
    );
  }

  Widget _buildHeader() {
    return Container(
      width: double.infinity,
      padding: const EdgeInsets.all(32),
      decoration: const BoxDecoration(
        gradient: LinearGradient(
          begin: Alignment.topLeft,
          end: Alignment.bottomRight,
          colors: [Color(0xFF6200EE), Color(0xFF3700B3)],
        ),
      ),
      child: Column(
        children: [
          const Icon(Icons.workspace_premium, size: 64, color: Colors.amber),
          const SizedBox(height: 16),
          Text(
            'Unlock Full Access',
            style: Theme.of(context).textTheme.headlineMedium?.copyWith(
              color: Colors.white,
              fontWeight: FontWeight.bold,
            ),
          ),
          const SizedBox(height: 8),
          const Text(
            'Remove ads and unlock all premium features',
            style: TextStyle(color: Colors.white70),
            textAlign: TextAlign.center,
          ),
        ],
      ),
    );
  }

  Widget _buildTierCards() {
    return Padding(
      padding: const EdgeInsets.all(16),
      child: Row(
        children: [
          Expanded(
            child: _TierCard(
              tier: SubscriptionTier.basic,
              isSelected: _selectedTier == SubscriptionTier.basic,
              onTap: () => setState(() => _selectedTier = SubscriptionTier.basic),
            ),
          ),
          const SizedBox(width: 12),
          Expanded(
            child: _TierCard(
              tier: SubscriptionTier.premium,
              isSelected: _selectedTier == SubscriptionTier.premium,
              isPopular: true,
              onTap: () => setState(() => _selectedTier = SubscriptionTier.premium),
            ),
          ),
          const SizedBox(width: 12),
          Expanded(
            child: _TierCard(
              tier: SubscriptionTier.lifetime,
              isSelected: _selectedTier == SubscriptionTier.lifetime,
              onTap: () => setState(() => _selectedTier = SubscriptionTier.lifetime),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildFeatureComparison() {
    final allFeatures = [
      'No Ads',
      'Unlimited Projects',
      'Export Features',
      'Priority Support',
      'Advanced Analytics',
      'Team Collaboration',
    ];

    final tierSupports = {
      SubscriptionTier.basic: [true, false, true, false, false, false],
      SubscriptionTier.premium: [true, true, true, true, true, true],
      SubscriptionTier.lifetime: [true, true, true, true, true, true],
    };

    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16),
      child: Card(
        child: Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              const Text(
                'Feature Comparison',
                style: TextStyle(fontWeight: FontWeight.bold, fontSize: 16),
              ),
              const SizedBox(height: 12),
              ...allFeatures.asMap().entries.map((entry) {
                final feature = entry.value;
                final index = entry.key;
                final isIncluded =
                    tierSupports[_selectedTier]?[index] ?? false;

                return Padding(
                  padding: const EdgeInsets.symmetric(vertical: 4),
                  child: Row(
                    children: [
                      Icon(
                        isIncluded ? Icons.check_circle : Icons.cancel,
                        color: isIncluded ? Colors.green : Colors.grey[300],
                        size: 20,
                      ),
                      const SizedBox(width: 12),
                      Text(feature),
                    ],
                  ),
                );
              }),
            ],
          ),
        ),
      ),
    );
  }

  Widget _buildPurchaseButton() {
    final service = SubscriptionService.instance;
    final isLoading = service.isLoading;

    String productId;
    switch (_selectedTier) {
      case SubscriptionTier.basic:
        productId = SubscriptionService.basicMonthlyId;
      case SubscriptionTier.premium:
        productId = SubscriptionService.premiumMonthlyId;
      case SubscriptionTier.lifetime:
        productId = SubscriptionService.lifetimeId;
      case SubscriptionTier.free:
        return const SizedBox.shrink();
    }

    return Padding(
      padding: const EdgeInsets.all(16),
      child: SizedBox(
        width: double.infinity,
        height: 56,
        child: ElevatedButton(
          onPressed: isLoading
              ? null
              : () async {
                  final success = await service.purchaseProduct(productId);
                  if (success && mounted) {
                    Navigator.pop(context);
                  }
                },
          style: ElevatedButton.styleFrom(
            backgroundColor: const Color(0xFF6200EE),
            foregroundColor: Colors.white,
            shape: RoundedRectangleBorder(
              borderRadius: BorderRadius.circular(12),
            ),
          ),
          child: isLoading
              ? const CircularProgressIndicator(color: Colors.white)
              : Text(
                  'Subscribe - ${_selectedTier.price}',
                  style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
                ),
        ),
      ),
    );
  }

  Widget _buildRestoreButton() {
    return TextButton(
      onPressed: () async {
        await SubscriptionService.instance.restorePurchases();
        if (mounted) {
          ScaffoldMessenger.of(context).showSnackBar(
            const SnackBar(content: Text('Purchases restored')),
          );
        }
      },
      child: const Text('Restore Purchases'),
    );
  }

  Widget _buildLegalText() {
    return Padding(
      padding: const EdgeInsets.all(16),
      child: Text(
        'Subscriptions will be charged to your account at the price above. '
        'Subscriptions automatically renew unless auto-renew is turned off at least '
        '24 hours before the end of the current period. '
        'Cancel anytime in your account settings.',
        style: TextStyle(
          fontSize: 10,
          color: Colors.grey[500],
        ),
        textAlign: TextAlign.center,
      ),
    );
  }
}

class _TierCard extends StatelessWidget {
  final SubscriptionTier tier;
  final bool isSelected;
  final bool isPopular;
  final VoidCallback onTap;

  const _TierCard({
    required this.tier,
    required this.isSelected,
    this.isPopular = false,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      child: Stack(
        clipBehavior: Clip.none,
        children: [
          AnimatedContainer(
            duration: const Duration(milliseconds: 200),
            padding: const EdgeInsets.all(12),
            decoration: BoxDecoration(
              borderRadius: BorderRadius.circular(12),
              border: Border.all(
                color: isSelected
                    ? const Color(0xFF6200EE)
                    : Colors.grey[300]!,
                width: isSelected ? 2 : 1,
              ),
              color: isSelected
                  ? const Color(0xFF6200EE).withOpacity(0.05)
                  : Colors.white,
            ),
            child: Column(
              children: [
                Text(
                  tier.displayName,
                  style: TextStyle(
                    fontWeight: FontWeight.bold,
                    color: isSelected ? const Color(0xFF6200EE) : null,
                  ),
                ),
                const SizedBox(height: 4),
                Text(
                  tier.price,
                  style: TextStyle(
                    fontSize: 12,
                    color: Colors.grey[600],
                  ),
                ),
              ],
            ),
          ),
          if (isPopular)
            Positioned(
              top: -10,
              left: 0,
              right: 0,
              child: Center(
                child: Container(
                  padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
                  decoration: BoxDecoration(
                    color: Colors.orange,
                    borderRadius: BorderRadius.circular(10),
                  ),
                  child: const Text(
                    'POPULAR',
                    style: TextStyle(
                      color: Colors.white,
                      fontSize: 9,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                ),
              ),
            ),
        ],
      ),
    );
  }
}

class UpgradeDialog extends StatelessWidget {
  const UpgradeDialog({super.key});

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: const Text('Upgrade to Premium'),
      content: const Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(Icons.workspace_premium, size: 48, color: Colors.amber),
          SizedBox(height: 8),
          Text('Get unlimited access and remove all ads!'),
        ],
      ),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context),
          child: const Text('Not Now'),
        ),
        ElevatedButton(
          onPressed: () {
            Navigator.pop(context);
            Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const UpgradeScreen()),
            );
          },
          child: const Text('View Plans'),
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 2888: Combining Ads + IAP - Main App

```dart
// lib/main.dart - Full monetized app
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'services/ads_service.dart';
import 'services/subscription_service.dart';
import 'screens/home_screen.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Initialize services
  await AdsService.instance.initialize();
  await SubscriptionService.instance.initialize();

  runApp(
    MultiProvider(
      providers: [
        ChangeNotifierProvider.value(value: SubscriptionService.instance),
        ChangeNotifierProvider.value(value: AutoUpdater.instance),
      ],
      child: const MonetizedApp(),
    ),
  );
}

class MonetizedApp extends StatelessWidget {
  const MonetizedApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Monetized Flutter App',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF6200EE)),
        useMaterial3: true,
      ),
      home: const HomeScreen(),
    );
  }
}
```

```dart
// lib/screens/home_screen.dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../services/ads_service.dart';
import '../services/subscription_service.dart';
import '../widgets/ad_banner_widget.dart';
import '../widgets/rewarded_ad_gate.dart';
import '../screens/upgrade_screen.dart';
import '../models/subscription.dart';

class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  int _articleCount = 0;

  Future<void> _onArticleRead() async {
    _articleCount++;

    // Show interstitial after every 3 article reads (free users only)
    final subscription = SubscriptionService.instance.subscription;
    if (subscription.tier.hasAds && _articleCount % 3 == 0) {
      final shown = await AdsService.instance.showInterstitialAd();
      if (!shown) {
        debugPrint('Interstitial not shown (frequency cap or not loaded)');
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Flutter News'),
        actions: [
          Consumer<SubscriptionService>(
            builder: (context, service, _) {
              if (!service.subscription.tier.hasAds) {
                return const Padding(
                  padding: EdgeInsets.symmetric(horizontal: 8),
                  child: Icon(Icons.workspace_premium, color: Colors.amber),
                );
              }
              return TextButton(
                onPressed: () => Navigator.push(
                  context,
                  MaterialPageRoute(builder: (_) => const UpgradeScreen()),
                ),
                child: const Text(
                  'Upgrade',
                  style: TextStyle(color: Colors.amber),
                ),
              );
            },
          ),
        ],
      ),
      body: Column(
        children: [
          // Top banner ad (free users only)
          const SmartBannerWidget(),

          // Content
          Expanded(
            child: ListView.builder(
              itemCount: 20,
              itemBuilder: (context, index) {
                return _ArticleCard(
                  index: index,
                  onRead: _onArticleRead,
                );
              },
            ),
          ),

          // Bottom banner ad (free users only)
          const AdBannerWidget(),
        ],
      ),
      floatingActionButton: Consumer<SubscriptionService>(
        builder: (context, service, _) {
          if (!service.subscription.tier.hasAds) return const SizedBox.shrink();

          return FloatingActionButton.extended(
            onPressed: () async {
              final reward = await AdsService.instance.showRewardedAd();
              if (reward != null && context.mounted) {
                ScaffoldMessenger.of(context).showSnackBar(
                  SnackBar(
                    content: Text(
                      'You earned ${reward.amount.toInt()} coins!',
                    ),
                    backgroundColor: Colors.green,
                  ),
                );
              }
            },
            icon: const Icon(Icons.play_arrow),
            label: const Text('Watch Ad for Coins'),
            backgroundColor: Colors.orange,
          );
        },
      ),
    );
  }
}

class _ArticleCard extends StatelessWidget {
  final int index;
  final VoidCallback onRead;

  const _ArticleCard({required this.index, required this.onRead});

  @override
  Widget build(BuildContext context) {
    final isPremiumContent = index % 5 == 4; // Every 5th article is premium

    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 6),
      child: InkWell(
        onTap: isPremiumContent ? null : onRead,
        borderRadius: BorderRadius.circular(12),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Container(
              height: 120,
              decoration: BoxDecoration(
                color: Colors.primaries[index % Colors.primaries.length],
                borderRadius: const BorderRadius.vertical(top: Radius.circular(12)),
              ),
              child: Stack(
                children: [
                  Center(
                    child: Icon(
                      Icons.article,
                      size: 48,
                      color: Colors.white.withOpacity(0.5),
                    ),
                  ),
                  if (isPremiumContent)
                    Positioned(
                      top: 8,
                      right: 8,
                      child: Container(
                        padding: const EdgeInsets.symmetric(
                          horizontal: 8,
                          vertical: 4,
                        ),
                        decoration: BoxDecoration(
                          color: Colors.amber,
                          borderRadius: BorderRadius.circular(4),
                        ),
                        child: const Row(
                          mainAxisSize: MainAxisSize.min,
                          children: [
                            Icon(Icons.workspace_premium, size: 12, color: Colors.white),
                            SizedBox(width: 4),
                            Text(
                              'PREMIUM',
                              style: TextStyle(
                                color: Colors.white,
                                fontSize: 10,
                                fontWeight: FontWeight.bold,
                              ),
                            ),
                          ],
                        ),
                      ),
                    ),
                ],
              ),
            ),
            if (isPremiumContent)
              RewardedAdGate(
                rewardDescription: 'Watch a short ad to read this premium article',
                lockedContent: _ArticlePreview(index: index),
                unlockedContent: (_) => _ArticleContent(index: index),
              )
            else
              _ArticleContent(index: index),
          ],
        ),
      ),
    );
  }
}

class _ArticlePreview extends StatelessWidget {
  final int index;
  const _ArticlePreview({required this.index});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(12),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(
            'Premium Article ${index + 1}',
            style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 16),
          ),
          const SizedBox(height: 4),
          const Text(
            'This is premium content. Unlock by watching an ad...',
            style: TextStyle(color: Colors.grey, fontSize: 13),
          ),
        ],
      ),
    );
  }
}

class _ArticleContent extends StatelessWidget {
  final int index;
  const _ArticleContent({required this.index});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(12),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(
            'Article ${index + 1}: Flutter Development Tips',
            style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 16),
          ),
          const SizedBox(height: 4),
          Text(
            'Lorem ipsum dolor sit amet, consectetur adipiscing elit. '
            'Sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.',
            style: TextStyle(color: Colors.grey[600], fontSize: 13),
          ),
          const SizedBox(height: 8),
          Row(
            children: [
              Icon(Icons.access_time, size: 14, color: Colors.grey[400]),
              const SizedBox(width: 4),
              Text(
                '5 min read',
                style: TextStyle(color: Colors.grey[400], fontSize: 12),
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

## ขั้นตอนที่ 2889: Ad Frequency Manager (Advanced)

```dart
// lib/services/ad_frequency_manager.dart
import 'package:shared_preferences/shared_preferences.dart';

/// Advanced frequency capping across sessions
class AdFrequencyManager {
  AdFrequencyManager._();
  static final AdFrequencyManager instance = AdFrequencyManager._();

  static const String _keyInterstitialTimes = 'interstitial_times';
  static const String _keySessionAdCount = 'session_ad_count';

  // Configurable limits
  int maxInterstitialsPerSession = 5;
  int maxInterstitialsPerHour = 3;
  Duration minTimeBetweenInterstitials = const Duration(minutes: 5);

  int _sessionAdCount = 0;

  Future<bool> canShowInterstitial() async {
    // Check session limit
    if (_sessionAdCount >= maxInterstitialsPerSession) return false;

    // Check hourly limit
    final prefs = await SharedPreferences.getInstance();
    final timesJson = prefs.getStringList(_keyInterstitialTimes) ?? [];
    final times = timesJson
        .map((s) => DateTime.parse(s))
        .toList();

    final now = DateTime.now();
    final oneHourAgo = now.subtract(const Duration(hours: 1));

    // Count ads in last hour
    final recentCount = times.where((t) => t.isAfter(oneHourAgo)).length;
    if (recentCount >= maxInterstitialsPerHour) return false;

    // Check minimum interval
    if (times.isNotEmpty) {
      final lastTime = times.last;
      if (now.difference(lastTime) < minTimeBetweenInterstitials) return false;
    }

    return true;
  }

  Future<void> recordInterstitialShown() async {
    _sessionAdCount++;

    final prefs = await SharedPreferences.getInstance();
    final timesJson = prefs.getStringList(_keyInterstitialTimes) ?? [];

    final now = DateTime.now();
    timesJson.add(now.toIso8601String());

    // Keep only last 24 hours
    final yesterday = now.subtract(const Duration(hours: 24));
    final filtered = timesJson
        .where((s) => DateTime.parse(s).isAfter(yesterday))
        .toList();

    await prefs.setStringList(_keyInterstitialTimes, filtered);
  }

  Future<AdFrequencyStats> getStats() async {
    final prefs = await SharedPreferences.getInstance();
    final timesJson = prefs.getStringList(_keyInterstitialTimes) ?? [];
    final times = timesJson.map((s) => DateTime.parse(s)).toList();
    final now = DateTime.now();

    final lastHour = times
        .where((t) => t.isAfter(now.subtract(const Duration(hours: 1))))
        .length;
    final lastDay = times
        .where((t) => t.isAfter(now.subtract(const Duration(hours: 24))))
        .length;

    return AdFrequencyStats(
      sessionCount: _sessionAdCount,
      lastHourCount: lastHour,
      lastDayCount: lastDay,
      totalCount: times.length,
    );
  }
}

class AdFrequencyStats {
  final int sessionCount;
  final int lastHourCount;
  final int lastDayCount;
  final int totalCount;

  AdFrequencyStats({
    required this.sessionCount,
    required this.lastHourCount,
    required this.lastDayCount,
    required this.totalCount,
  });

  @override
  String toString() =>
      'AdStats(session: $sessionCount, hour: $lastHourCount, '
      'day: $lastDayCount, total: $totalCount)';
}
```

---

## ขั้นตอนที่ 2890: สรุป Monetization Strategy

```dart
/*
MONETIZATION DECISION MATRIX
══════════════════════════════════════════════════════════════════════
App Type           | Best Strategy                | Why
──────────────────────────────────────────────────────────────────────
Games              | Rewarded Ads + IAP coins     | Users opt-in for reward
News/Content       | Banner + Freemium            | Natural reading flow
Tools/Productivity | Freemium + Subscription      | Clear value proposition
Social App         | Freemium + Cosmetics IAP     | FOMO drives purchases
Professional Tool  | Free Trial + Subscription    | Value before payment
──────────────────────────────────────────────────────────────────────

AD REVENUE FORMULAS:
  Daily Revenue = DAU × CTR × CPC × Fill Rate
  Interstitial eCPM: ฿50-200 (varies by region)
  Banner eCPM: ฿5-30
  Rewarded eCPM: ฿100-400

FREQUENCY CAP BEST PRACTICES:
  - Interstitials: max 3/hour, min 5min apart
  - Never show on every screen transition
  - Let user finish action before showing
  - Never break user flow mid-task

FREEMIUM CONVERSION RATES:
  - Average: 2-5% of free users convert
  - Good: 5-8%
  - Excellent: 8%+
  - Key: clear feature differentiation

COMBINING ADS + IAP:
  ✅ Offer "remove ads" as IAP
  ✅ Use rewarded ads for virtual currency
  ✅ Show upgrade CTA near ads (not over them)
  ✅ Track LTV (Lifetime Value) per user segment
  ❌ Never auto-play audio ads
  ❌ Never show ads on onboarding
  ❌ Never interrupt payments with ads
*/

// Example: Track revenue metrics
class RevenueTracker {
  static void trackAdImpression(String adType) {
    // Log to Firebase Analytics
    print('Ad impression: $adType');
  }

  static void trackAdClick(String adType) {
    print('Ad click: $adType');
  }

  static void trackSubscriptionStart(String tier, double amount) {
    print('Subscription: $tier, amount: $amount');
  }

  static void trackSubscriptionCancel(String tier, String reason) {
    print('Cancellation: $tier, reason: $reason');
  }

  static void trackConversionFunnel(String step) {
    // paywall_view → plan_selected → purchase_started → purchase_complete
    print('Funnel: $step');
  }
}
```

---

**← [Part 74](part-74-flutter-desktop-advanced.md)**
**ต่อไป: [Part 76 →](part-76-advanced-testing.md)**
