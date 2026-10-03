# Part 62: Security Best Practices
## ขั้นตอนที่ 2361-2400

## 🎯 เป้าหมายของ Part นี้
- Certificate pinning ด้วย dio
- Root/Jailbreak detection
- Code obfuscation
- Secure storage ด้วย flutter_secure_storage
- RASP (Runtime Application Self-Protection) patterns
- OWASP Mobile Top 10 mitigations

---

## ขั้นตอนที่ 2361: pubspec.yaml สำหรับ Security

```yaml
# pubspec.yaml
name: flutter_security_app
description: Security Best Practices in Flutter

environment:
  sdk: ">=3.0.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter
  dio: ^5.3.3
  flutter_secure_storage: ^9.0.0
  local_auth: ^2.1.7
  crypto: ^3.0.3
  pointycastle: ^3.7.3
  device_info_plus: ^9.1.0
  package_info_plus: ^5.0.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
```

---

## ขั้นตอนที่ 2362: Certificate Pinning with Dio

```dart
// lib/security/certificate_pinning.dart
import 'dart:io';
import 'package:dio/dio.dart';
import 'package:dio/io.dart';

/// Certificate pinning implementation using SHA-256 hash of certificate
/// Run: openssl s_client -connect api.example.com:443 | openssl x509 -pubkey -noout | openssl pkey -pubin -outform DER | openssl dgst -sha256 -binary | base64
const String _pinnedCertHash =
    'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA='; // Replace with real hash

class PinnedDioClient {
  late final Dio _dio;

  PinnedDioClient({required String baseUrl}) {
    _dio = Dio(
      BaseOptions(
        baseUrl: baseUrl,
        connectTimeout: const Duration(seconds: 30),
        receiveTimeout: const Duration(seconds: 30),
        headers: {
          'Content-Type': 'application/json',
          'Accept': 'application/json',
        },
      ),
    );

    _setupCertificatePinning();
    _setupInterceptors();
  }

  void _setupCertificatePinning() {
    (_dio.httpClientAdapter as IOHttpClientAdapter).createHttpClient = () {
      final client = HttpClient();
      client.badCertificateCallback =
          (X509Certificate cert, String host, int port) {
        // Verify certificate fingerprint
        final certBytes = cert.der;
        final hash = _computeSha256Base64(certBytes);
        
        if (hash != _pinnedCertHash) {
          // In production: log security event and reject
          return false;
        }
        return true;
      };
      return client;
    };
  }

  String _computeSha256Base64(List<int> bytes) {
    // Using dart:convert + dart:typed_data for SHA-256
    // In real app use crypto package: sha256.convert(bytes).bytes
    return _pinnedCertHash; // placeholder
  }

  void _setupInterceptors() {
    _dio.interceptors.add(
      InterceptorsWrapper(
        onRequest: (options, handler) {
          // Add security headers
          options.headers['X-Request-ID'] = _generateRequestId();
          options.headers['X-Client-Version'] = '1.0.0';
          handler.next(options);
        },
        onResponse: (response, handler) {
          // Validate response headers
          final serverHeader = response.headers.value('Server');
          if (serverHeader != null && serverHeader.contains('Microsoft-IIS')) {
            // Log unexpected server technology
          }
          handler.next(response);
        },
        onError: (error, handler) {
          if (error.type == DioExceptionType.badCertificate) {
            // Certificate pinning failed - possible MITM attack
            _handleSecurityViolation('Certificate pinning failed');
          }
          handler.next(error);
        },
      ),
    );
  }

  String _generateRequestId() {
    final timestamp = DateTime.now().millisecondsSinceEpoch;
    return '$timestamp-${timestamp.hashCode.abs()}';
  }

  void _handleSecurityViolation(String reason) {
    // Log to security monitoring service
    // Optionally terminate app
    throw SecurityException('Security violation: $reason');
  }

  Dio get client => _dio;
}

class SecurityException implements Exception {
  final String message;
  SecurityException(this.message);

  @override
  String toString() => 'SecurityException: $message';
}
```

---

## ขั้นตอนที่ 2363: Dio with Multiple Certificate Pins

```dart
// lib/security/multi_pin_client.dart
import 'dart:convert';
import 'dart:io';
import 'dart:typed_data';
import 'package:dio/dio.dart';
import 'package:dio/io.dart';

/// Support for multiple pins (backup pins for certificate rotation)
class MultiPinDioClient {
  static const List<String> _allowedPins = [
    'sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=',
    'sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=', // backup pin
  ];

  static Dio createSecureDio({
    required String baseUrl,
    List<String>? allowedPins,
  }) {
    final pins = allowedPins ?? _allowedPins;
    final dio = Dio(
      BaseOptions(
        baseUrl: baseUrl,
        connectTimeout: const Duration(seconds: 30),
      ),
    );

    (dio.httpClientAdapter as IOHttpClientAdapter).createHttpClient = () {
      final client = HttpClient();
      client.badCertificateCallback = (cert, host, port) {
        return _validateCertificate(cert, pins);
      };
      return client;
    };

    return dio;
  }

  static bool _validateCertificate(
    X509Certificate cert,
    List<String> allowedPins,
  ) {
    try {
      final certDer = cert.der;
      // Compute SHA-256 of SubjectPublicKeyInfo
      // In production: extract SPKI from DER, hash it
      final hashBase64 = base64.encode(certDer.sublist(0, 32)); // simplified
      final pinString = 'sha256/$hashBase64';
      return allowedPins.any((pin) => pin == pinString);
    } catch (e) {
      return false;
    }
  }
}
```

---

## ขั้นตอนที่ 2364: Root/Jailbreak Detection

```dart
// lib/security/device_security.dart
import 'dart:io';
import 'package:flutter/foundation.dart';
import 'package:device_info_plus/device_info_plus.dart';

enum SecurityThreatLevel { none, low, medium, high, critical }

class SecurityThreat {
  final String name;
  final String description;
  final SecurityThreatLevel level;
  final bool isMitigated;

  const SecurityThreat({
    required this.name,
    required this.description,
    required this.level,
    this.isMitigated = false,
  });
}

class DeviceSecurityCheck {
  final DeviceInfoPlugin _deviceInfo = DeviceInfoPlugin();

  Future<List<SecurityThreat>> runAllChecks() async {
    final threats = <SecurityThreat>[];

    if (Platform.isAndroid) {
      threats.addAll(await _checkAndroidSecurity());
    } else if (Platform.isIOS) {
      threats.addAll(await _checkIOSSecurity());
    }

    threats.addAll(_checkCommonThreats());

    return threats;
  }

  Future<List<SecurityThreat>> _checkAndroidSecurity() async {
    final threats = <SecurityThreat>[];

    // Check for root indicators
    if (await _isAndroidRooted()) {
      threats.add(const SecurityThreat(
        name: 'Root Access',
        description: 'Device appears to be rooted',
        level: SecurityThreatLevel.high,
      ));
    }

    // Check for emulator
    if (await _isEmulator()) {
      threats.add(const SecurityThreat(
        name: 'Emulator Detected',
        description: 'Running on an emulator or virtual device',
        level: SecurityThreatLevel.medium,
      ));
    }

    // Check USB debugging
    if (await _isUsbDebuggingEnabled()) {
      threats.add(const SecurityThreat(
        name: 'USB Debugging',
        description: 'USB debugging is enabled',
        level: SecurityThreatLevel.low,
      ));
    }

    return threats;
  }

  Future<List<SecurityThreat>> _checkIOSSecurity() async {
    final threats = <SecurityThreat>[];

    if (_isJailbroken()) {
      threats.add(const SecurityThreat(
        name: 'Jailbreak Detected',
        description: 'Device appears to be jailbroken',
        level: SecurityThreatLevel.high,
      ));
    }

    return threats;
  }

  List<SecurityThreat> _checkCommonThreats() {
    final threats = <SecurityThreat>[];

    if (kDebugMode) {
      threats.add(const SecurityThreat(
        name: 'Debug Mode',
        description: 'App running in debug mode',
        level: SecurityThreatLevel.low,
      ));
    }

    return threats;
  }

  Future<bool> _isAndroidRooted() async {
    // Check for su binary in common locations
    final suPaths = [
      '/system/bin/su',
      '/system/xbin/su',
      '/sbin/su',
      '/system/app/Superuser.apk',
      '/system/app/SuperSU.apk',
      '/data/local/xbin/su',
      '/data/local/bin/su',
    ];

    for (final path in suPaths) {
      if (await File(path).exists()) {
        return true;
      }
    }

    // Check for root management apps
    const rootApps = [
      '/system/app/Superuser',
      '/system/app/SuperSU',
      '/system/app/Magisk',
    ];

    for (final app in rootApps) {
      if (await Directory(app).exists()) {
        return true;
      }
    }

    return false;
  }

  bool _isJailbroken() {
    if (!Platform.isIOS) return false;

    // Check for common jailbreak files
    final jailbreakPaths = [
      '/Applications/Cydia.app',
      '/Library/MobileSubstrate/MobileSubstrate.dylib',
      '/bin/bash',
      '/usr/sbin/sshd',
      '/etc/apt',
      '/private/var/lib/apt/',
      '/private/var/mobile/Library/SBSettings/Themes',
    ];

    for (final path in jailbreakPaths) {
      if (File(path).existsSync() || Directory(path).existsSync()) {
        return true;
      }
    }

    return false;
  }

  Future<bool> _isEmulator() async {
    try {
      if (Platform.isAndroid) {
        final info = await _deviceInfo.androidInfo;
        return info.isPhysicalDevice == false ||
            info.fingerprint.contains('generic') ||
            info.model.contains('Emulator') ||
            info.brand == 'generic' ||
            info.manufacturer.contains('Genymotion');
      }
    } catch (_) {}
    return false;
  }

  Future<bool> _isUsbDebuggingEnabled() async {
    try {
      if (Platform.isAndroid) {
        final info = await _deviceInfo.androidInfo;
        return info.isPhysicalDevice && kDebugMode;
      }
    } catch (_) {}
    return false;
  }

  SecurityThreatLevel getOverallThreatLevel(List<SecurityThreat> threats) {
    if (threats.isEmpty) return SecurityThreatLevel.none;

    final highestLevel = threats
        .map((t) => t.level.index)
        .reduce((a, b) => a > b ? a : b);

    return SecurityThreatLevel.values[highestLevel];
  }
}
```

---

## ขั้นตอนที่ 2365: Security Gate Widget

```dart
// lib/security/security_gate.dart
import 'package:flutter/material.dart';
import 'device_security.dart';

class SecurityGate extends StatefulWidget {
  final Widget child;
  final SecurityThreatLevel maxAllowedLevel;
  final Widget Function(List<SecurityThreat> threats)? blockedBuilder;

  const SecurityGate({
    super.key,
    required this.child,
    this.maxAllowedLevel = SecurityThreatLevel.low,
    this.blockedBuilder,
  });

  @override
  State<SecurityGate> createState() => _SecurityGateState();
}

class _SecurityGateState extends State<SecurityGate> {
  final DeviceSecurityCheck _securityCheck = DeviceSecurityCheck();
  List<SecurityThreat>? _threats;
  bool _isChecking = true;

  @override
  void initState() {
    super.initState();
    _runSecurityCheck();
  }

  Future<void> _runSecurityCheck() async {
    final threats = await _securityCheck.runAllChecks();
    if (mounted) {
      setState(() {
        _threats = threats;
        _isChecking = false;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    if (_isChecking) {
      return const Scaffold(
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              CircularProgressIndicator(),
              SizedBox(height: 16),
              Text('Checking device security...'),
            ],
          ),
        ),
      );
    }

    final threats = _threats ?? [];
    final overallLevel =
        _securityCheck.getOverallThreatLevel(threats);

    if (overallLevel.index > widget.maxAllowedLevel.index) {
      if (widget.blockedBuilder != null) {
        return widget.blockedBuilder!(threats);
      }
      return _DefaultBlockedScreen(threats: threats);
    }

    return widget.child;
  }
}

class _DefaultBlockedScreen extends StatelessWidget {
  final List<SecurityThreat> threats;

  const _DefaultBlockedScreen({required this.threats});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.red.shade900,
      body: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(
                Icons.security,
                size: 80,
                color: Colors.white,
              ),
              const SizedBox(height: 24),
              const Text(
                'Security Check Failed',
                style: TextStyle(
                  color: Colors.white,
                  fontSize: 24,
                  fontWeight: FontWeight.bold,
                ),
              ),
              const SizedBox(height: 16),
              const Text(
                'This app cannot run on a compromised device.',
                style: TextStyle(color: Colors.white70),
                textAlign: TextAlign.center,
              ),
              const SizedBox(height: 24),
              ...threats
                  .where((t) => t.level.index >= SecurityThreatLevel.medium.index)
                  .map(
                    (threat) => Card(
                      color: Colors.red.shade800,
                      child: ListTile(
                        leading: const Icon(Icons.warning, color: Colors.yellow),
                        title: Text(
                          threat.name,
                          style: const TextStyle(color: Colors.white),
                        ),
                        subtitle: Text(
                          threat.description,
                          style: const TextStyle(color: Colors.white70),
                        ),
                      ),
                    ),
                  ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2366: Secure Storage Service

```dart
// lib/security/secure_storage_service.dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'dart:convert';

class SecureStorageService {
  static const FlutterSecureStorage _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(
      encryptedSharedPreferences: true,
      sharedPreferencesName: 'secure_prefs',
      preferencesKeyPrefix: 'secure_',
    ),
    iOptions: IOSOptions(
      accessibility: KeychainAccessibility.first_unlock_this_device,
      synchronizable: false,
    ),
  );

  // Keys
  static const String _authTokenKey = 'auth_token';
  static const String _refreshTokenKey = 'refresh_token';
  static const String _userDataKey = 'user_data';
  static const String _biometricEnabledKey = 'biometric_enabled';
  static const String _pinHashKey = 'pin_hash';

  // Auth Token
  static Future<void> saveAuthToken(String token) async {
    await _storage.write(key: _authTokenKey, value: token);
  }

  static Future<String?> getAuthToken() async {
    return _storage.read(key: _authTokenKey);
  }

  static Future<void> deleteAuthToken() async {
    await _storage.delete(key: _authTokenKey);
  }

  // Refresh Token
  static Future<void> saveRefreshToken(String token) async {
    await _storage.write(key: _refreshTokenKey, value: token);
  }

  static Future<String?> getRefreshToken() async {
    return _storage.read(key: _refreshTokenKey);
  }

  // User Data (JSON)
  static Future<void> saveUserData(Map<String, dynamic> userData) async {
    final jsonStr = jsonEncode(userData);
    await _storage.write(key: _userDataKey, value: jsonStr);
  }

  static Future<Map<String, dynamic>?> getUserData() async {
    final jsonStr = await _storage.read(key: _userDataKey);
    if (jsonStr == null) return null;
    return jsonDecode(jsonStr) as Map<String, dynamic>;
  }

  // Biometric preference
  static Future<void> setBiometricEnabled(bool enabled) async {
    await _storage.write(
      key: _biometricEnabledKey,
      value: enabled.toString(),
    );
  }

  static Future<bool> isBiometricEnabled() async {
    final value = await _storage.read(key: _biometricEnabledKey);
    return value == 'true';
  }

  // PIN storage (store only the hash, never the plain PIN)
  static Future<void> savePinHash(String pinHash) async {
    await _storage.write(key: _pinHashKey, value: pinHash);
  }

  static Future<String?> getPinHash() async {
    return _storage.read(key: _pinHashKey);
  }

  static Future<bool> verifyPin(String inputPin, String salt) async {
    final storedHash = await getPinHash();
    if (storedHash == null) return false;
    final inputHash = _hashPin(inputPin, salt);
    return inputHash == storedHash;
  }

  static String _hashPin(String pin, String salt) {
    // In production: use bcrypt or PBKDF2 from pointycastle
    final bytes = utf8.encode('$salt:$pin');
    var hash = 0;
    for (final byte in bytes) {
      hash = (hash * 31 + byte) & 0xFFFFFFFF;
    }
    return hash.toRadixString(16).padLeft(8, '0');
  }

  // Clear all secure data
  static Future<void> clearAll() async {
    await _storage.deleteAll();
  }

  // Check if any auth data exists
  static Future<bool> hasAuthData() async {
    final token = await getAuthToken();
    return token != null;
  }
}
```

---

## ขั้นตอนที่ 2367: Biometric Authentication

```dart
// lib/security/biometric_auth.dart
import 'package:flutter/services.dart';
import 'package:local_auth/local_auth.dart';

enum BiometricAuthResult {
  success,
  failed,
  notAvailable,
  notEnrolled,
  lockedOut,
  cancelled,
}

class BiometricAuthService {
  final LocalAuthentication _localAuth = LocalAuthentication();

  Future<bool> isAvailable() async {
    try {
      return await _localAuth.canCheckBiometrics &&
          await _localAuth.isDeviceSupported();
    } on PlatformException {
      return false;
    }
  }

  Future<List<BiometricType>> getAvailableBiometrics() async {
    try {
      return await _localAuth.getAvailableBiometrics();
    } on PlatformException {
      return [];
    }
  }

  Future<BiometricAuthResult> authenticate({
    String reason = 'Please authenticate to continue',
    bool useStrongAuthentication = true,
  }) async {
    try {
      final isAvail = await isAvailable();
      if (!isAvail) return BiometricAuthResult.notAvailable;

      final authenticated = await _localAuth.authenticate(
        localizedReason: reason,
        options: AuthenticationOptions(
          biometricOnly: useStrongAuthentication,
          stickyAuth: true,
          sensitiveTransaction: true,
          useErrorDialogs: true,
        ),
      );

      return authenticated
          ? BiometricAuthResult.success
          : BiometricAuthResult.failed;
    } on PlatformException catch (e) {
      switch (e.code) {
        case 'NotAvailable':
          return BiometricAuthResult.notAvailable;
        case 'NotEnrolled':
          return BiometricAuthResult.notEnrolled;
        case 'LockedOut':
        case 'PermanentlyLockedOut':
          return BiometricAuthResult.lockedOut;
        default:
          return BiometricAuthResult.failed;
      }
    }
  }

  Future<void> stopAuthentication() async {
    await _localAuth.stopAuthentication();
  }
}
```

---

## ขั้นตอนที่ 2368: Network Security Config (Android)

```xml
<!-- android/app/src/main/res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- Block cleartext traffic globally -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>

    <!-- Domain-specific configuration for production -->
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">api.myapp.com</domain>
        <trust-anchors>
            <certificates src="system" />
            <!-- Add backup certificate pins -->
            <certificates src="@raw/my_certificate" />
        </trust-anchors>
        <pin-set expiration="2026-01-01">
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <!-- Backup pin for certificate rotation -->
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
        </pin-set>
    </domain-config>

    <!-- Allow cleartext only for debug builds -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

---

## ขั้นตอนที่ 2369: OWASP Mobile Top 10 - Input Validation

```dart
// lib/security/input_validator.dart
import 'dart:convert';

class InputValidator {
  // M1: Improper Platform Usage - Validate all inputs
  static String? validateEmail(String? value) {
    if (value == null || value.isEmpty) return 'Email is required';
    final emailRegex = RegExp(
      r'^[a-zA-Z0-9.]+@[a-zA-Z0-9]+\.[a-zA-Z]+',
    );
    if (!emailRegex.hasMatch(value)) return 'Invalid email format';
    if (value.length > 254) return 'Email too long';
    return null;
  }

  static String? validatePassword(String? value) {
    if (value == null || value.isEmpty) return 'Password is required';
    if (value.length < 8) return 'Password must be at least 8 characters';
    if (value.length > 128) return 'Password too long';
    if (!RegExp(r'[A-Z]').hasMatch(value)) {
      return 'Must contain uppercase letter';
    }
    if (!RegExp(r'[a-z]').hasMatch(value)) {
      return 'Must contain lowercase letter';
    }
    if (!RegExp(r'[0-9]').hasMatch(value)) {
      return 'Must contain a number';
    }
    if (!RegExp(r'[!@#$%^&*(),.?":{}|<>]').hasMatch(value)) {
      return 'Must contain a special character';
    }
    return null;
  }

  // M7: Client Code Quality - Prevent SQL injection in local DB
  static String sanitizeForSql(String input) {
    return input
        .replaceAll("'", "''")
        .replaceAll(';', '')
        .replaceAll('--', '')
        .replaceAll('/*', '')
        .replaceAll('*/', '');
  }

  // Prevent path traversal
  static String sanitizeFilename(String filename) {
    return filename
        .replaceAll(RegExp(r'[/\\]'), '_')
        .replaceAll('..', '_')
        .replaceAll(RegExp(r'[^a-zA-Z0-9._-]'), '_');
  }

  // Validate URL scheme to prevent open redirects
  static bool isValidRedirectUrl(String url, List<String> allowedHosts) {
    try {
      final uri = Uri.parse(url);
      if (!['https', 'http'].contains(uri.scheme)) return false;
      return allowedHosts.contains(uri.host);
    } catch (_) {
      return false;
    }
  }

  // M8: Security Decisions via Untrusted Inputs
  static String sanitizeHtml(String html) {
    // Remove script tags and event handlers
    var sanitized = html
        .replaceAll(RegExp(r'<script[^>]*>.*?</script>', dotAll: true), '')
        .replaceAll(RegExp(r' on\w+="[^"]*"'), '')
        .replaceAll(RegExp(r' on\w+=\'[^\']*\''), '')
        .replaceAll(RegExp(r'javascript:', caseSensitive: false), '');
    return sanitized;
  }

  // Encode output to prevent XSS in WebViews
  static String encodeForHtml(String input) {
    return input
        .replaceAll('&', '&amp;')
        .replaceAll('<', '&lt;')
        .replaceAll('>', '&gt;')
        .replaceAll('"', '&quot;')
        .replaceAll("'", '&#x27;');
  }
}
```

---

## ขั้นตอนที่ 2370: RASP - Runtime Application Self-Protection

```dart
// lib/security/rasp.dart
import 'dart:async';
import 'dart:io';
import 'package:flutter/foundation.dart';

typedef SecurityViolationCallback = void Function(
  String violationType,
  String details,
);

class RASP {
  static RASP? _instance;
  static RASP get instance => _instance ??= RASP._();

  RASP._();

  SecurityViolationCallback? _onViolation;
  Timer? _integrityTimer;
  bool _isActive = false;

  final List<String> _violations = [];
  int _violationCount = 0;
  static const int _maxViolations = 3;

  void initialize({
    required SecurityViolationCallback onViolation,
    bool enableContinuousMonitoring = true,
  }) {
    _onViolation = onViolation;
    _isActive = true;

    _runInitialChecks();

    if (enableContinuousMonitoring) {
      _startContinuousMonitoring();
    }
  }

  void _runInitialChecks() {
    _checkDebugger();
    _checkTampering();
    if (Platform.isAndroid) _checkAndroidIntegrity();
    if (Platform.isIOS) _checkIOSIntegrity();
  }

  void _startContinuousMonitoring() {
    _integrityTimer = Timer.periodic(
      const Duration(seconds: 30),
      (_) => _runInitialChecks(),
    );
  }

  void _checkDebugger() {
    if (kDebugMode) {
      _reportViolation(
        'debug_mode',
        'Application is running in debug mode',
      );
    }

    // Check for debugger attachment
    // In production, use native code for reliable detection
  }

  void _checkTampering() {
    // Check if app signature matches expected value
    // This requires native code in production
    // Example: compare current signature with stored hash

    if (Platform.isAndroid) {
      _checkAndroidSignature();
    }
  }

  void _checkAndroidSignature() {
    // In production: use MethodChannel to get APK signature
    // and compare against stored expected signature hash
    const expectedSignatureHash =
        'expected_sha256_hash_of_certificate'; // Replace with real value
    // final actualHash = await _getNativeSignatureHash();
    // if (actualHash != expectedSignatureHash) {
    //   _reportViolation('signature_mismatch', 'APK signature verification failed');
    // }
  }

  void _checkAndroidIntegrity() {
    // Check for Android emulator
    // Check for debuggable flag
    // Check for test-keys build
  }

  void _checkIOSIntegrity() {
    // Check for jailbreak
    // Check for debugger
    // Check code signing
  }

  void _reportViolation(String type, String details) {
    if (!_isActive) return;

    _violations.add('$type: $details');
    _violationCount++;

    _onViolation?.call(type, details);

    if (_violationCount >= _maxViolations) {
      _terminate();
    }
  }

  void _terminate() {
    // Wipe sensitive data from memory
    _clearSensitiveData();

    // Exit app
    // In production: use a more graceful shutdown
    if (Platform.isIOS || Platform.isAndroid) {
      exit(0);
    }
  }

  void _clearSensitiveData() {
    // Clear in-memory secrets
    // This should also clear SecureStorage in production
  }

  List<String> get violations => List.unmodifiable(_violations);

  void dispose() {
    _integrityTimer?.cancel();
    _isActive = false;
  }
}
```

---

## ขั้นตอนที่ 2371: Obfuscation Build Commands

```bash
# Build with obfuscation for Android
flutter build apk --obfuscate \
  --split-debug-info=build/debug-info/android \
  --release

# Build with obfuscation for iOS  
flutter build ios --obfuscate \
  --split-debug-info=build/debug-info/ios \
  --release

# Build App Bundle with obfuscation
flutter build appbundle --obfuscate \
  --split-debug-info=build/debug-info/android \
  --release

# Symbolize crash stack trace using debug-info
flutter symbolize \
  --debug-info=build/debug-info/android/app.android-arm64.symbols \
  --input=stack_trace.txt

# Additional ProGuard rules for Android (android/app/proguard-rules.pro)
# -keep class io.flutter.** { *; }
# -keep class io.flutter.embedding.** { *; }
# -dontwarn io.flutter.**
# -keep class your.package.name.** { *; }
```

---

## ขั้นตอนที่ 2372: Secure API Client - Complete Implementation

```dart
// lib/security/secure_api_client.dart
import 'dart:convert';
import 'package:crypto/crypto.dart';
import 'package:dio/dio.dart';
import 'secure_storage_service.dart';

class SecureApiClient {
  late final Dio _dio;
  final String _apiSecret;
  int _requestCount = 0;

  SecureApiClient({
    required String baseUrl,
    required String apiSecret,
  }) : _apiSecret = apiSecret {
    _dio = Dio(
      BaseOptions(
        baseUrl: baseUrl,
        connectTimeout: const Duration(seconds: 30),
        receiveTimeout: const Duration(seconds: 30),
      ),
    );

    _setupInterceptors();
  }

  void _setupInterceptors() {
    // HMAC signing interceptor
    _dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) async {
        await _signRequest(options);
        handler.next(options);
      },
      onResponse: (response, handler) {
        _validateResponse(response);
        handler.next(response);
      },
      onError: (error, handler) async {
        if (error.response?.statusCode == 401) {
          // Try token refresh
          final refreshed = await _refreshToken();
          if (refreshed) {
            // Retry original request
            final response = await _dio.fetch(error.requestOptions);
            handler.resolve(response);
            return;
          }
        }
        handler.next(error);
      },
    ));

    // Rate limiting
    _dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) {
        _requestCount++;
        if (_requestCount > 100) {
          handler.reject(
            DioException(
              requestOptions: options,
              message: 'Rate limit exceeded',
            ),
          );
          return;
        }
        handler.next(options);
      },
    ));
  }

  Future<void> _signRequest(RequestOptions options) async {
    final token = await SecureStorageService.getAuthToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }

    // HMAC signature for request integrity
    final timestamp = DateTime.now().millisecondsSinceEpoch.toString();
    final nonce = _generateNonce();
    final method = options.method.toUpperCase();
    final path = options.path;

    final message = '$timestamp$nonce$method$path';
    final signature = _computeHmac(message, _apiSecret);

    options.headers['X-Timestamp'] = timestamp;
    options.headers['X-Nonce'] = nonce;
    options.headers['X-Signature'] = signature;
  }

  void _validateResponse(Response response) {
    // Validate server signature if present
    final serverSignature = response.headers.value('X-Server-Signature');
    if (serverSignature != null) {
      // Verify signature
    }

    // Check for replay attacks
    final timestamp = response.headers.value('X-Timestamp');
    if (timestamp != null) {
      final serverTime = int.tryParse(timestamp) ?? 0;
      final now = DateTime.now().millisecondsSinceEpoch;
      const maxSkewMs = 5 * 60 * 1000; // 5 minutes
      if ((now - serverTime).abs() > maxSkewMs) {
        throw Exception('Response timestamp skew too large - possible replay attack');
      }
    }
  }

  Future<bool> _refreshToken() async {
    try {
      final refreshToken = await SecureStorageService.getRefreshToken();
      if (refreshToken == null) return false;

      final response = await _dio.post(
        '/auth/refresh',
        data: {'refresh_token': refreshToken},
        options: Options(headers: {'Authorization': null}),
      );

      final newToken = response.data['access_token'] as String?;
      if (newToken != null) {
        await SecureStorageService.saveAuthToken(newToken);
        return true;
      }
    } catch (_) {}
    return false;
  }

  String _computeHmac(String message, String secret) {
    final key = utf8.encode(secret);
    final bytes = utf8.encode(message);
    final hmacSha256 = Hmac(sha256, key);
    final digest = hmacSha256.convert(bytes);
    return digest.toString();
  }

  String _generateNonce() {
    final timestamp = DateTime.now().microsecondsSinceEpoch;
    return sha256.convert(utf8.encode(timestamp.toString())).toString().substring(0, 16);
  }

  Future<Response<T>> get<T>(String path, {Map<String, dynamic>? params}) {
    return _dio.get<T>(path, queryParameters: params);
  }

  Future<Response<T>> post<T>(String path, {dynamic data}) {
    return _dio.post<T>(path, data: data);
  }
}
```

---

## ขั้นตอนที่ 2373: OWASP Checklist Implementation

```dart
// lib/security/owasp_checklist.dart

/// OWASP Mobile Top 10 (2023) Mitigations
/// M1: Improper Credential Usage
class CredentialManager {
  // Never store credentials in plain text
  // Never hardcode credentials in source code
  // Use secure storage for all sensitive data
  
  static void validateNoHardcodedSecrets() {
    // This would be enforced by static analysis tools
    // like Semgrep or custom lint rules
  }
}

/// M2: Inadequate Supply Chain Security  
class SupplyChainSecurity {
  // Pin dependency versions in pubspec.lock
  // Verify package signatures
  // Use trusted package sources only
  
  static List<String> getTrustedSources() => [
    'pub.dev',
    'dart.dev',
  ];
}

/// M3: Insecure Authentication/Authorization
class AuthorizationChecker {
  final Map<String, Set<String>> _rolePermissions = {
    'admin': {'read', 'write', 'delete', 'manage_users'},
    'editor': {'read', 'write'},
    'viewer': {'read'},
  };

  bool hasPermission(String role, String permission) {
    return _rolePermissions[role]?.contains(permission) ?? false;
  }

  bool canAccessResource(String userRole, String resource, String action) {
    // Always validate on server side; this is client-side UX only
    return hasPermission(userRole, action);
  }
}

/// M4: Insufficient Input/Output Validation
class IoValidation {
  static String sanitizeOutput(String data) {
    return data
        .replaceAll('<', '&lt;')
        .replaceAll('>', '&gt;')
        .replaceAll('"', '&quot;');
  }

  static bool isValidUrl(String url) {
    try {
      final uri = Uri.parse(url);
      return uri.hasScheme && ['http', 'https'].contains(uri.scheme);
    } catch (_) {
      return false;
    }
  }
}

/// M5: Insecure Communication
class CommunicationSecurity {
  static bool isSecureEndpoint(String url) {
    return url.startsWith('https://');
  }

  static Map<String, String> getSecurityHeaders() {
    return {
      'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
      'X-Content-Type-Options': 'nosniff',
      'X-Frame-Options': 'DENY',
      'Cache-Control': 'no-store',
    };
  }
}

/// M6: Inadequate Privacy Controls
class PrivacyManager {
  static const List<String> sensitiveFields = [
    'password', 'token', 'secret', 'card_number', 'cvv', 'ssn',
  ];

  static Map<String, dynamic> maskSensitiveData(Map<String, dynamic> data) {
    final masked = Map<String, dynamic>.from(data);
    for (final field in sensitiveFields) {
      if (masked.containsKey(field)) {
        masked[field] = '***REDACTED***';
      }
    }
    return masked;
  }

  static String maskEmail(String email) {
    final parts = email.split('@');
    if (parts.length != 2) return email;
    final name = parts[0];
    final domain = parts[1];
    final maskedName = name.length <= 2
        ? '*' * name.length
        : '${name[0]}${'*' * (name.length - 2)}${name[name.length - 1]}';
    return '$maskedName@$domain';
  }
}
```

---

## ขั้นตอนที่ 2374: Security Screen UI

```dart
// lib/screens/security_dashboard_screen.dart
import 'package:flutter/material.dart';
import '../security/device_security.dart';
import '../security/biometric_auth.dart';
import '../security/secure_storage_service.dart';

class SecurityDashboardScreen extends StatefulWidget {
  const SecurityDashboardScreen({super.key});

  @override
  State<SecurityDashboardScreen> createState() =>
      _SecurityDashboardScreenState();
}

class _SecurityDashboardScreenState extends State<SecurityDashboardScreen> {
  final DeviceSecurityCheck _securityCheck = DeviceSecurityCheck();
  final BiometricAuthService _biometricAuth = BiometricAuthService();

  List<SecurityThreat> _threats = [];
  bool _biometricAvailable = false;
  bool _biometricEnabled = false;
  bool _isLoading = true;

  @override
  void initState() {
    super.initState();
    _loadSecurityInfo();
  }

  Future<void> _loadSecurityInfo() async {
    setState(() => _isLoading = true);

    final results = await Future.wait([
      _securityCheck.runAllChecks(),
      _biometricAuth.isAvailable(),
      SecureStorageService.isBiometricEnabled(),
    ]);

    setState(() {
      _threats = results[0] as List<SecurityThreat>;
      _biometricAvailable = results[1] as bool;
      _biometricEnabled = results[2] as bool;
      _isLoading = false;
    });
  }

  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return const Scaffold(
        body: Center(child: CircularProgressIndicator()),
      );
    }

    final threatLevel = _securityCheck.getOverallThreatLevel(_threats);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Security Dashboard'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: _loadSecurityInfo,
          ),
        ],
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          _buildThreatLevelCard(threatLevel),
          const SizedBox(height: 16),
          _buildBiometricCard(),
          const SizedBox(height: 16),
          _buildThreatsSection(),
        ],
      ),
    );
  }

  Widget _buildThreatLevelCard(SecurityThreatLevel level) {
    final (color, icon, label) = switch (level) {
      SecurityThreatLevel.none => (Colors.green, Icons.check_circle, 'Secure'),
      SecurityThreatLevel.low => (Colors.yellow, Icons.warning, 'Low Risk'),
      SecurityThreatLevel.medium => (Colors.orange, Icons.warning, 'Medium Risk'),
      SecurityThreatLevel.high => (Colors.red, Icons.dangerous, 'High Risk'),
      SecurityThreatLevel.critical => (Colors.red.shade900, Icons.gpp_bad, 'Critical'),
    };

    return Card(
      color: color.withOpacity(0.1),
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Row(
          children: [
            Icon(icon, color: color, size: 48),
            const SizedBox(width: 16),
            Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                const Text(
                  'Overall Security Status',
                  style: TextStyle(fontWeight: FontWeight.bold),
                ),
                Text(
                  label,
                  style: TextStyle(
                    color: color,
                    fontSize: 20,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                Text('${_threats.length} threat(s) detected'),
              ],
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildBiometricCard() {
    return Card(
      child: ListTile(
        leading: Icon(
          Icons.fingerprint,
          color: _biometricAvailable ? Colors.blue : Colors.grey,
          size: 36,
        ),
        title: const Text('Biometric Authentication'),
        subtitle: Text(
          _biometricAvailable
              ? 'Fingerprint/Face ID available'
              : 'Not available on this device',
        ),
        trailing: _biometricAvailable
            ? Switch(
                value: _biometricEnabled,
                onChanged: (value) async {
                  if (value) {
                    final result = await _biometricAuth.authenticate(
                      reason: 'Enable biometric authentication',
                    );
                    if (result == BiometricAuthResult.success) {
                      await SecureStorageService.setBiometricEnabled(true);
                      setState(() => _biometricEnabled = true);
                    }
                  } else {
                    await SecureStorageService.setBiometricEnabled(false);
                    setState(() => _biometricEnabled = false);
                  }
                },
              )
            : null,
      ),
    );
  }

  Widget _buildThreatsSection() {
    if (_threats.isEmpty) {
      return const Card(
        child: ListTile(
          leading: Icon(Icons.check, color: Colors.green),
          title: Text('No security threats detected'),
        ),
      );
    }

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(
          'Security Threats',
          style: Theme.of(context).textTheme.titleLarge,
        ),
        const SizedBox(height: 8),
        ..._threats.map(
          (threat) => Card(
            child: ListTile(
              leading: Icon(
                Icons.warning,
                color: threat.level == SecurityThreatLevel.high
                    ? Colors.red
                    : Colors.orange,
              ),
              title: Text(threat.name),
              subtitle: Text(threat.description),
              trailing: Chip(
                label: Text(threat.level.name),
                backgroundColor: threat.level == SecurityThreatLevel.high
                    ? Colors.red.shade100
                    : Colors.orange.shade100,
              ),
            ),
          ),
        ),
      ],
    );
  }
}
```

---

**← [Part 61](part-61-flutter-test-advanced.md)**
**ต่อไป: [Part 63 →](part-63-real-world-app-ecommerce.md)**
