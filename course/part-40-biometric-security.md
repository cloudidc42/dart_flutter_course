# Part 40: Biometric Authentication & App Security
## ขั้นตอนที่ 1481-1520

---

## 🎯 เป้าหมายของ Part นี้

- Face ID / Fingerprint authentication
- Secure storage
- SSL Pinning
- Code obfuscation
- App tamper detection

---

## ขั้นตอนที่ 1481: Biometric Authentication

```yaml
# pubspec.yaml
dependencies:
  local_auth: ^2.3.0
  flutter_secure_storage: ^9.2.0
  pointycastle: ^3.7.3
  ssl_pinning_plugin: ^2.0.0
```

```dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:local_auth/local_auth.dart';

class BiometricService {
  static final BiometricService _instance = BiometricService._();
  factory BiometricService() => _instance;
  BiometricService._();

  final LocalAuthentication _auth = LocalAuthentication();

  // ตรวจสอบว่า device รองรับ biometric
  Future<bool> isBiometricAvailable() async {
    try {
      return await _auth.canCheckBiometrics;
    } on PlatformException {
      return false;
    }
  }

  // ดู biometric types ที่รองรับ
  Future<List<BiometricType>> getAvailableBiometrics() async {
    try {
      return await _auth.getAvailableBiometrics();
    } on PlatformException {
      return [];
    }
  }

  // Authenticate
  Future<BiometricResult> authenticate({
    String reason = 'กรุณายืนยันตัวตนเพื่อเข้าใช้งาน',
    bool useErrorDialogs = true,
    bool stickyAuth = true,
  }) async {
    bool available = await isBiometricAvailable();
    if (!available) {
      return BiometricResult.notAvailable;
    }

    try {
      bool authenticated = await _auth.authenticate(
        localizedReason: reason,
        options: AuthenticationOptions(
          useErrorDialogs: useErrorDialogs,
          stickyAuth: stickyAuth,
          biometricOnly: false, // false = allow PIN/password fallback
        ),
      );

      return authenticated ? BiometricResult.success : BiometricResult.failed;
    } on PlatformException catch (e) {
      if (e.code == 'LockedOut') return BiometricResult.lockedOut;
      if (e.code == 'NotEnrolled') return BiometricResult.notEnrolled;
      return BiometricResult.error;
    }
  }

  Future<void> cancelAuthentication() async {
    await _auth.stopAuthentication();
  }
}

enum BiometricResult {
  success,
  failed,
  notAvailable,
  notEnrolled,
  lockedOut,
  error,
}

// ─── Biometric Auth Screen ───
class BiometricLockScreen extends StatefulWidget {
  final Widget child;
  const BiometricLockScreen({super.key, required this.child});

  @override
  State<BiometricLockScreen> createState() => _BiometricLockScreenState();
}

class _BiometricLockScreenState extends State<BiometricLockScreen>
    with WidgetsBindingObserver {
  final BiometricService _biometric = BiometricService();
  bool _isAuthenticated = false;
  bool _isAuthenticating = false;
  String? _errorMessage;
  DateTime? _backgroundedAt;

  static const Duration _lockAfter = Duration(minutes: 5);

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    _authenticate();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    switch (state) {
      case AppLifecycleState.paused:
        _backgroundedAt = DateTime.now();
        break;
      case AppLifecycleState.resumed:
        if (_backgroundedAt != null) {
          Duration elapsed = DateTime.now().difference(_backgroundedAt!);
          if (elapsed > _lockAfter) {
            setState(() => _isAuthenticated = false);
            _authenticate();
          }
        }
        break;
      default:
        break;
    }
  }

  Future<void> _authenticate() async {
    if (_isAuthenticating) return;
    setState(() {
      _isAuthenticating = true;
      _errorMessage = null;
    });

    BiometricResult result = await _biometric.authenticate(
      reason: 'กรุณายืนยันตัวตนเพื่อเข้าใช้แอป',
    );

    setState(() {
      _isAuthenticating = false;
      switch (result) {
        case BiometricResult.success:
          _isAuthenticated = true;
        case BiometricResult.failed:
          _errorMessage = 'ยืนยันตัวตนไม่สำเร็จ';
        case BiometricResult.notAvailable:
          _isAuthenticated = true; // fallback: ข้ามไป
        case BiometricResult.notEnrolled:
          _errorMessage = 'ยังไม่ได้ตั้งค่า biometric บน device';
          _isAuthenticated = true; // fallback
        case BiometricResult.lockedOut:
          _errorMessage = 'ลองใหม่ในอีก 30 วินาที';
        case BiometricResult.error:
          _errorMessage = 'เกิดข้อผิดพลาด';
          _isAuthenticated = true; // fallback
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    if (_isAuthenticated) return widget.child;

    return Scaffold(
      backgroundColor: Theme.of(context).colorScheme.surface,
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Icon(Icons.lock, size: 80, color: Colors.grey),
            const SizedBox(height: 24),
            const Text(
              'App Locked',
              style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            if (_errorMessage != null)
              Padding(
                padding: const EdgeInsets.all(16),
                child: Text(
                  _errorMessage!,
                  style: const TextStyle(color: Colors.red),
                  textAlign: TextAlign.center,
                ),
              ),
            const SizedBox(height: 16),
            if (_isAuthenticating)
              const CircularProgressIndicator()
            else
              ElevatedButton.icon(
                onPressed: _authenticate,
                icon: const Icon(Icons.fingerprint),
                label: const Text('ยืนยันตัวตน'),
              ),
          ],
        ),
      ),
    );
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }
}
```

---

## ขั้นตอนที่ 1482: Secure Storage

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'dart:convert';

// ─── Secure Storage Service ───
class SecureStorageService {
  static final SecureStorageService _instance = SecureStorageService._();
  factory SecureStorageService() => _instance;
  SecureStorageService._();

  final FlutterSecureStorage _storage = const FlutterSecureStorage(
    aOptions: AndroidOptions(
      encryptedSharedPreferences: true,
    ),
    iOptions: IOSOptions(
      accessibility: KeychainAccessibility.first_unlock_this_device,
    ),
  );

  // Keys
  static const String _authTokenKey = 'auth_token';
  static const String _refreshTokenKey = 'refresh_token';
  static const String _userIdKey = 'user_id';
  static const String _pinKey = 'app_pin';
  static const String _biometricEnabledKey = 'biometric_enabled';

  // ─── Token management ───
  Future<void> saveTokens({
    required String authToken,
    required String refreshToken,
  }) async {
    await Future.wait([
      _storage.write(key: _authTokenKey, value: authToken),
      _storage.write(key: _refreshTokenKey, value: refreshToken),
    ]);
  }

  Future<String?> getAuthToken() => _storage.read(key: _authTokenKey);
  Future<String?> getRefreshToken() => _storage.read(key: _refreshTokenKey);

  Future<void> clearTokens() async {
    await Future.wait([
      _storage.delete(key: _authTokenKey),
      _storage.delete(key: _refreshTokenKey),
    ]);
  }

  // ─── User info ───
  Future<void> saveUserId(String userId) =>
      _storage.write(key: _userIdKey, value: userId);
  Future<String?> getUserId() => _storage.read(key: _userIdKey);

  // ─── PIN management ───
  Future<void> setPin(String pin) async {
    // Hash PIN before storing
    String hashedPin = _hashPin(pin);
    await _storage.write(key: _pinKey, value: hashedPin);
  }

  Future<bool> verifyPin(String pin) async {
    String? storedHash = await _storage.read(key: _pinKey);
    if (storedHash == null) return false;
    return _hashPin(pin) == storedHash;
  }

  String _hashPin(String pin) {
    // ใน production ควรใช้ bcrypt หรือ scrypt
    List<int> bytes = utf8.encode(pin + 'salt_secret');
    return base64.encode(bytes);
  }

  Future<bool> hasPin() async => (await _storage.read(key: _pinKey)) != null;
  Future<void> removePin() => _storage.delete(key: _pinKey);

  // ─── Biometric settings ───
  Future<void> setBiometricEnabled(bool enabled) =>
      _storage.write(key: _biometricEnabledKey, value: enabled.toString());

  Future<bool> isBiometricEnabled() async {
    String? value = await _storage.read(key: _biometricEnabledKey);
    return value == 'true';
  }

  // ─── Clear all ───
  Future<void> clearAll() => _storage.deleteAll();

  // ─── Custom key-value ───
  Future<void> write(String key, String value) => _storage.write(key: key, value: value);
  Future<String?> read(String key) => _storage.read(key: key);
  Future<void> delete(String key) => _storage.delete(key: key);
}
```

---

## ขั้นตอนที่ 1483: PIN Setup Screen

```dart
import 'package:flutter/material.dart';

class PinSetupScreen extends StatefulWidget {
  final bool isNewPin; // true = ตั้งค่าใหม่, false = ยืนยัน
  final Function(String pin)? onPinSet;

  const PinSetupScreen({super.key, this.isNewPin = true, this.onPinSet});

  @override
  State<PinSetupScreen> createState() => _PinSetupScreenState();
}

class _PinSetupScreenState extends State<PinSetupScreen> {
  String _pin = '';
  String? _firstPin;
  String _statusText = 'กรอก PIN 6 หลัก';
  bool _isError = false;

  void _onKeyPress(String digit) {
    if (_pin.length >= 6) return;

    setState(() {
      _pin += digit;
      _isError = false;
    });

    if (_pin.length == 6) {
      _onPinComplete();
    }
  }

  void _onDelete() {
    if (_pin.isEmpty) return;
    setState(() => _pin = _pin.substring(0, _pin.length - 1));
  }

  Future<void> _onPinComplete() async {
    if (widget.isNewPin) {
      if (_firstPin == null) {
        // รอบแรก: บันทึก PIN แล้วให้กรอกอีกครั้ง
        setState(() {
          _firstPin = _pin;
          _pin = '';
          _statusText = 'กรอก PIN อีกครั้งเพื่อยืนยัน';
        });
      } else {
        // รอบสอง: ตรวจสอบว่าตรงกัน
        if (_pin == _firstPin) {
          await SecureStorageService().setPin(_pin);
          widget.onPinSet?.call(_pin);
          if (mounted) Navigator.pop(context, true);
        } else {
          setState(() {
            _pin = '';
            _firstPin = null;
            _statusText = 'PIN ไม่ตรงกัน ลองใหม่อีกครั้ง';
            _isError = true;
          });
        }
      }
    } else {
      // Verify PIN mode
      bool isCorrect = await SecureStorageService().verifyPin(_pin);
      if (isCorrect) {
        if (mounted) Navigator.pop(context, true);
      } else {
        setState(() {
          _pin = '';
          _statusText = 'PIN ไม่ถูกต้อง';
          _isError = true;
        });
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.isNewPin ? 'ตั้งค่า PIN' : 'ใส่ PIN'),
      ),
      body: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          // Status text
          Text(
            _statusText,
            style: TextStyle(
              fontSize: 18,
              color: _isError ? Colors.red : null,
            ),
          ),
          const SizedBox(height: 32),

          // PIN dots
          Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: List.generate(6, (index) {
              bool filled = index < _pin.length;
              return Container(
                width: 16,
                height: 16,
                margin: const EdgeInsets.symmetric(horizontal: 8),
                decoration: BoxDecoration(
                  shape: BoxShape.circle,
                  color: filled ? Colors.blue : Colors.transparent,
                  border: Border.all(
                    color: _isError ? Colors.red : Colors.blue,
                    width: 2,
                  ),
                ),
              );
            }),
          ),
          const SizedBox(height: 48),

          // Numpad
          ...List.generate(3, (row) => Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: List.generate(3, (col) {
              int num = row * 3 + col + 1;
              return _NumPadButton(
                label: '$num',
                onPress: () => _onKeyPress('$num'),
              );
            }),
          )),

          // Bottom row
          Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const SizedBox(width: 80, height: 80),
              _NumPadButton(label: '0', onPress: () => _onKeyPress('0')),
              _NumPadButton(
                icon: Icons.backspace_outlined,
                onPress: _onDelete,
              ),
            ],
          ),
        ],
      ),
    );
  }
}

class _NumPadButton extends StatelessWidget {
  final String? label;
  final IconData? icon;
  final VoidCallback onPress;

  const _NumPadButton({this.label, this.icon, required this.onPress});

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      width: 80,
      height: 80,
      child: TextButton(
        onPressed: onPress,
        child: label != null
            ? Text(label!, style: const TextStyle(fontSize: 28))
            : Icon(icon, size: 28),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1484: SSL Pinning

```dart
import 'dart:io';
import 'package:dio/dio.dart';
import 'package:dio/io.dart';
import 'package:flutter/services.dart';

// ─── SSL Pinning กับ Dio ───
class SecureHttpClient {
  static Dio createWithPinning() {
    Dio dio = Dio(BaseOptions(
      baseUrl: 'https://api.yourapp.com',
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 30),
    ));

    (dio.httpClientAdapter as IOHttpClientAdapter).createHttpClient = () {
      HttpClient client = HttpClient();

      client.badCertificateCallback = (X509Certificate cert, String host, int port) {
        // ตรวจสอบ certificate fingerprint
        String fingerprint = cert.sha256.map((b) => b.toRadixString(16).padLeft(2, '0')).join(':');

        const List<String> trustedFingerprints = [
          'AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99',
        ];

        return trustedFingerprints.contains(fingerprint.toUpperCase());
      };

      return client;
    };

    return dio;
  }
}

// ─── Certificate pinning ผ่าน asset ───
class CertificatePinningService {
  static Future<SecurityContext> createSecurityContext() async {
    // Load certificate จาก assets
    ByteData certData = await rootBundle.load('assets/certificates/server.crt');
    SecurityContext context = SecurityContext(withTrustedRoots: false);
    context.setTrustedCertificatesBytes(certData.buffer.asUint8List());
    return context;
  }
}

// ─── App Security Checklist ───
class AppSecurityChecker {
  // ตรวจสอบว่า device ถูก root/jailbreak
  static Future<bool> isDeviceCompromised() async {
    // ใช้ flutter_jailbreak_detection package
    // ใน production ควรใช้ package จริง
    return false;
  }

  // ตรวจสอบว่า app ถูก tamper
  static bool isSignatureValid() {
    // ตรวจสอบ app signature
    // ใน production ใช้ Play Integrity API (Android)
    // หรือ App Attest (iOS)
    return true;
  }

  // ตรวจสอบว่า run ใน emulator
  static bool isEmulator() {
    return false; // ใช้ device_info_plus ตรวจสอบ
  }
}
```

---

**← [Part 39 - WebSocket & Real-time](part-39-websocket-realtime.md)**

**ต่อไป: [Part 41 - In-App Purchases →](part-41-in-app-purchases.md)**
