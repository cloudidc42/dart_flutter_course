# Part 92: Capstone – Auth Module
## ขั้นตอนที่ 3561-3600

## 🎯 เป้าหมายของ Part นี้
- สร้าง Auth screens (Login, Register, Forgot Password)
- เชื่อมต่อ Email + Google + Apple Sign In
- Phone OTP verification
- JWT token management
- Auth state management ด้วย Riverpod

---

## ขั้นตอนที่ 3561: Auth Repository Interface & Use Cases

```dart
// lib/features/auth/domain/repositories/auth_repository.dart

import 'package:dartz/dartz.dart';

abstract class AuthRepository {
  Future<Either<Failure, UserEntity>> loginWithEmail({
    required String email,
    required String password,
  });

  Future<Either<Failure, UserEntity>> registerWithEmail({
    required String email,
    required String password,
    required String displayName,
  });

  Future<Either<Failure, UserEntity>> loginWithGoogle();

  Future<Either<Failure, UserEntity>> loginWithApple();

  Future<Either<Failure, String>> sendPhoneOtp(String phoneNumber);

  Future<Either<Failure, UserEntity>> verifyPhoneOtp({
    required String verificationId,
    required String otp,
  });

  Future<Either<Failure, void>> sendPasswordResetEmail(String email);

  Future<Either<Failure, void>> logout();

  Stream<UserEntity?> get authStateChanges;

  UserEntity? get currentUser;
}

// lib/features/auth/domain/usecases/login_usecase.dart

class LoginParams {
  final String email;
  final String password;
  const LoginParams({required this.email, required this.password});
}

class LoginUseCase {
  final AuthRepository repository;
  const LoginUseCase(this.repository);

  Future<Either<Failure, UserEntity>> call(LoginParams params) =>
      repository.loginWithEmail(
        email: params.email,
        password: params.password,
      );
}

class RegisterParams {
  final String email;
  final String password;
  final String displayName;
  const RegisterParams({
    required this.email,
    required this.password,
    required this.displayName,
  });
}

class RegisterUseCase {
  final AuthRepository repository;
  const RegisterUseCase(this.repository);

  Future<Either<Failure, UserEntity>> call(RegisterParams params) =>
      repository.registerWithEmail(
        email: params.email,
        password: params.password,
        displayName: params.displayName,
      );
}
```

---

## ขั้นตอนที่ 3562: Auth Repository Implementation

```dart
// lib/features/auth/data/repositories/auth_repository_impl.dart

import 'package:firebase_auth/firebase_auth.dart';
import 'package:google_sign_in/google_sign_in.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:dartz/dartz.dart';
import 'package:sign_in_with_apple/sign_in_with_apple.dart';

class AuthRepositoryImpl implements AuthRepository {
  final FirebaseAuth _firebaseAuth;
  final GoogleSignIn _googleSignIn;
  final FirebaseFirestore _firestore;

  AuthRepositoryImpl({
    required FirebaseAuth firebaseAuth,
    required GoogleSignIn googleSignIn,
    required FirebaseFirestore firestore,
  })  : _firebaseAuth = firebaseAuth,
        _googleSignIn = googleSignIn,
        _firestore = firestore;

  @override
  Stream<UserEntity?> get authStateChanges {
    return _firebaseAuth.authStateChanges().asyncMap((user) async {
      if (user == null) return null;
      return _getUserFromFirestore(user.uid);
    });
  }

  @override
  UserEntity? get currentUser {
    final user = _firebaseAuth.currentUser;
    if (user == null) return null;
    // Return a minimal entity synchronously; full data loaded async
    return UserModel(
      id: user.uid,
      email: user.email ?? '',
      displayName: user.displayName ?? '',
      photoUrl: user.photoURL,
      phoneNumber: user.phoneNumber,
      isEmailVerified: user.emailVerified,
      isPhoneVerified: user.phoneNumber != null,
      createdAt: user.metadata.creationTime ?? DateTime.now(),
      updatedAt: DateTime.now(),
    );
  }

  @override
  Future<Either<Failure, UserEntity>> loginWithEmail({
    required String email,
    required String password,
  }) async {
    try {
      final credential = await _firebaseAuth.signInWithEmailAndPassword(
        email: email,
        password: password,
      );
      final user = await _getUserFromFirestore(credential.user!.uid);
      return Right(user);
    } on FirebaseAuthException catch (e) {
      return Left(AuthFailure(
        message: _mapFirebaseAuthError(e.code),
        code: int.tryParse(e.code),
      ));
    } catch (e) {
      return Left(ServerFailure(message: e.toString()));
    }
  }

  @override
  Future<Either<Failure, UserEntity>> registerWithEmail({
    required String email,
    required String password,
    required String displayName,
  }) async {
    try {
      final credential = await _firebaseAuth.createUserWithEmailAndPassword(
        email: email,
        password: password,
      );
      await credential.user!.updateDisplayName(displayName);
      await credential.user!.sendEmailVerification();

      final now = DateTime.now();
      final newUser = UserModel(
        id: credential.user!.uid,
        email: email,
        displayName: displayName,
        role: UserRole.customer,
        createdAt: now,
        updatedAt: now,
      );

      await _firestore
          .collection(FirestoreCollections.users)
          .doc(newUser.id)
          .set(newUser.toFirestore());

      return Right(newUser);
    } on FirebaseAuthException catch (e) {
      return Left(AuthFailure(message: _mapFirebaseAuthError(e.code)));
    } catch (e) {
      return Left(ServerFailure(message: e.toString()));
    }
  }

  @override
  Future<Either<Failure, UserEntity>> loginWithGoogle() async {
    try {
      final googleUser = await _googleSignIn.signIn();
      if (googleUser == null) {
        return const Left(AuthFailure(message: 'Google sign-in cancelled'));
      }
      final googleAuth = await googleUser.authentication;
      final credential = GoogleAuthProvider.credential(
        accessToken: googleAuth.accessToken,
        idToken: googleAuth.idToken,
      );
      final userCredential =
          await _firebaseAuth.signInWithCredential(credential);
      final user = userCredential.user!;

      final doc = await _firestore
          .collection(FirestoreCollections.users)
          .doc(user.uid)
          .get();

      if (!doc.exists) {
        final now = DateTime.now();
        final newUser = UserModel(
          id: user.uid,
          email: user.email ?? '',
          displayName: user.displayName ?? googleUser.displayName ?? '',
          photoUrl: user.photoURL,
          isEmailVerified: true,
          createdAt: now,
          updatedAt: now,
        );
        await _firestore
            .collection(FirestoreCollections.users)
            .doc(user.uid)
            .set(newUser.toFirestore());
        return Right(newUser);
      }

      final existingUser = await _getUserFromFirestore(user.uid);
      return Right(existingUser);
    } catch (e) {
      return Left(AuthFailure(message: e.toString()));
    }
  }

  @override
  Future<Either<Failure, UserEntity>> loginWithApple() async {
    try {
      final appleCredential = await SignInWithApple.getAppleIDCredential(
        scopes: [
          AppleIDAuthorizationScopes.email,
          AppleIDAuthorizationScopes.fullName,
        ],
      );
      final oauthCredential = OAuthProvider('apple.com').credential(
        idToken: appleCredential.identityToken,
        accessToken: appleCredential.authorizationCode,
      );
      final userCredential =
          await _firebaseAuth.signInWithCredential(oauthCredential);
      final user = userCredential.user!;
      final displayName =
          '${appleCredential.givenName ?? ''} ${appleCredential.familyName ?? ''}'
              .trim();

      final doc = await _firestore
          .collection(FirestoreCollections.users)
          .doc(user.uid)
          .get();

      if (!doc.exists) {
        final now = DateTime.now();
        final newUser = UserModel(
          id: user.uid,
          email: user.email ?? '',
          displayName: displayName.isNotEmpty ? displayName : 'Apple User',
          isEmailVerified: true,
          createdAt: now,
          updatedAt: now,
        );
        await _firestore
            .collection(FirestoreCollections.users)
            .doc(user.uid)
            .set(newUser.toFirestore());
        return Right(newUser);
      }

      return Right(await _getUserFromFirestore(user.uid));
    } catch (e) {
      return Left(AuthFailure(message: e.toString()));
    }
  }

  @override
  Future<Either<Failure, String>> sendPhoneOtp(String phoneNumber) async {
    try {
      String? verificationId;
      await _firebaseAuth.verifyPhoneNumber(
        phoneNumber: phoneNumber,
        verificationCompleted: (_) {},
        verificationFailed: (e) {
          throw FirebaseAuthException(code: e.code, message: e.message);
        },
        codeSent: (vId, _) {
          verificationId = vId;
        },
        codeAutoRetrievalTimeout: (_) {},
      );
      if (verificationId == null) {
        return const Left(AuthFailure(message: 'Failed to send OTP'));
      }
      return Right(verificationId!);
    } on FirebaseAuthException catch (e) {
      return Left(AuthFailure(message: _mapFirebaseAuthError(e.code)));
    }
  }

  @override
  Future<Either<Failure, UserEntity>> verifyPhoneOtp({
    required String verificationId,
    required String otp,
  }) async {
    try {
      final credential = PhoneAuthProvider.credential(
        verificationId: verificationId,
        smsCode: otp,
      );
      final userCredential =
          await _firebaseAuth.signInWithCredential(credential);
      final user = await _getUserFromFirestore(userCredential.user!.uid);
      return Right(user);
    } on FirebaseAuthException catch (e) {
      return Left(AuthFailure(message: _mapFirebaseAuthError(e.code)));
    }
  }

  @override
  Future<Either<Failure, void>> sendPasswordResetEmail(String email) async {
    try {
      await _firebaseAuth.sendPasswordResetEmail(email: email);
      return const Right(null);
    } on FirebaseAuthException catch (e) {
      return Left(AuthFailure(message: _mapFirebaseAuthError(e.code)));
    }
  }

  @override
  Future<Either<Failure, void>> logout() async {
    try {
      await Future.wait([
        _firebaseAuth.signOut(),
        _googleSignIn.signOut(),
      ]);
      return const Right(null);
    } catch (e) {
      return Left(AuthFailure(message: e.toString()));
    }
  }

  Future<UserEntity> _getUserFromFirestore(String uid) async {
    final doc = await _firestore
        .collection(FirestoreCollections.users)
        .doc(uid)
        .get();
    if (!doc.exists) {
      throw const ServerException(message: 'User not found in Firestore');
    }
    return UserModel.fromFirestore(doc.data()!, doc.id);
  }

  String _mapFirebaseAuthError(String code) {
    switch (code) {
      case 'user-not-found':
        return 'ไม่พบบัญชีผู้ใช้นี้';
      case 'wrong-password':
        return 'รหัสผ่านไม่ถูกต้อง';
      case 'email-already-in-use':
        return 'อีเมลนี้ถูกใช้งานแล้ว';
      case 'invalid-email':
        return 'รูปแบบอีเมลไม่ถูกต้อง';
      case 'weak-password':
        return 'รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร';
      case 'too-many-requests':
        return 'ลองมากเกินไป กรุณารอสักครู่';
      case 'invalid-verification-code':
        return 'รหัส OTP ไม่ถูกต้อง';
      default:
        return 'เกิดข้อผิดพลาด กรุณาลองใหม่';
    }
  }
}
```

---

## ขั้นตอนที่ 3563: Auth State Notifier (Riverpod)

```dart
// lib/features/auth/presentation/providers/auth_providers.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:google_sign_in/google_sign_in.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

final firebaseAuthProvider =
    Provider<FirebaseAuth>((ref) => FirebaseAuth.instance);

final googleSignInProvider =
    Provider<GoogleSignIn>((ref) => GoogleSignIn());

final firestoreProvider =
    Provider<FirebaseFirestore>((ref) => FirebaseFirestore.instance);

final authRepositoryProvider = Provider<AuthRepository>((ref) {
  return AuthRepositoryImpl(
    firebaseAuth: ref.watch(firebaseAuthProvider),
    googleSignIn: ref.watch(googleSignInProvider),
    firestore: ref.watch(firestoreProvider),
  );
});

final loginUseCaseProvider =
    Provider((ref) => LoginUseCase(ref.watch(authRepositoryProvider)));

final registerUseCaseProvider =
    Provider((ref) => RegisterUseCase(ref.watch(authRepositoryProvider)));

// Auth State

sealed class AuthState {
  const AuthState();
}

class AuthInitial extends AuthState {
  const AuthInitial();
}

class AuthLoading extends AuthState {
  const AuthLoading();
}

class AuthAuthenticated extends AuthState {
  final UserEntity user;
  const AuthAuthenticated(this.user);
}

class AuthUnauthenticated extends AuthState {
  const AuthUnauthenticated();
}

class AuthError extends AuthState {
  final String message;
  const AuthError(this.message);
}

class AuthNotifier extends AsyncNotifier<UserEntity?> {
  @override
  Future<UserEntity?> build() async {
    final repo = ref.watch(authRepositoryProvider);
    // Listen to auth state stream and invalidate when it changes
    ref.listen(
      authStateStreamProvider,
      (_, next) {
        next.whenData((user) => state = AsyncData(user));
      },
    );
    return repo.currentUser;
  }

  Future<void> loginWithEmail({
    required String email,
    required String password,
  }) async {
    state = const AsyncLoading();
    final repo = ref.read(authRepositoryProvider);
    final result = await repo.loginWithEmail(
      email: email,
      password: password,
    );
    state = result.fold(
      (failure) => AsyncError(failure.message, StackTrace.current),
      (user) => AsyncData(user),
    );
  }

  Future<void> register({
    required String email,
    required String password,
    required String displayName,
  }) async {
    state = const AsyncLoading();
    final repo = ref.read(authRepositoryProvider);
    final result = await repo.registerWithEmail(
      email: email,
      password: password,
      displayName: displayName,
    );
    state = result.fold(
      (failure) => AsyncError(failure.message, StackTrace.current),
      (user) => AsyncData(user),
    );
  }

  Future<void> loginWithGoogle() async {
    state = const AsyncLoading();
    final repo = ref.read(authRepositoryProvider);
    final result = await repo.loginWithGoogle();
    state = result.fold(
      (failure) => AsyncError(failure.message, StackTrace.current),
      (user) => AsyncData(user),
    );
  }

  Future<void> loginWithApple() async {
    state = const AsyncLoading();
    final repo = ref.read(authRepositoryProvider);
    final result = await repo.loginWithApple();
    state = result.fold(
      (failure) => AsyncError(failure.message, StackTrace.current),
      (user) => AsyncData(user),
    );
  }

  Future<void> logout() async {
    final repo = ref.read(authRepositoryProvider);
    await repo.logout();
    state = const AsyncData(null);
  }
}

final authNotifierProvider =
    AsyncNotifierProvider<AuthNotifier, UserEntity?>(() => AuthNotifier());

final authStateStreamProvider = StreamProvider<UserEntity?>((ref) {
  return ref.watch(authRepositoryProvider).authStateChanges;
});

final isAuthenticatedProvider = Provider<bool>((ref) {
  return ref.watch(authNotifierProvider).valueOrNull != null;
});
```

---

## ขั้นตอนที่ 3564: Login Page UI

```dart
// lib/features/auth/presentation/pages/login_page.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

class LoginPage extends ConsumerStatefulWidget {
  const LoginPage({super.key});

  @override
  ConsumerState<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends ConsumerState<LoginPage> {
  final _formKey = GlobalKey<FormState>();
  final _emailCtrl = TextEditingController();
  final _passCtrl = TextEditingController();
  bool _obscurePassword = true;

  @override
  void dispose() {
    _emailCtrl.dispose();
    _passCtrl.dispose();
    super.dispose();
  }

  Future<void> _onLogin() async {
    if (!_formKey.currentState!.validate()) return;
    await ref.read(authNotifierProvider.notifier).loginWithEmail(
          email: _emailCtrl.text.trim(),
          password: _passCtrl.text,
        );
  }

  @override
  Widget build(BuildContext context) {
    ref.listen(authNotifierProvider, (prev, next) {
      next.whenOrNull(
        data: (user) {
          if (user != null) context.go('/home');
        },
        error: (err, _) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(
              content: Text(err.toString()),
              backgroundColor: Colors.red,
            ),
          );
        },
      );
    });

    final authState = ref.watch(authNotifierProvider);
    final isLoading = authState.isLoading;

    return Scaffold(
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(24),
          child: Form(
            key: _formKey,
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                const SizedBox(height: 48),
                _buildLogo(),
                const SizedBox(height: 40),
                _buildTitle(),
                const SizedBox(height: 32),
                _buildEmailField(),
                const SizedBox(height: 16),
                _buildPasswordField(),
                const SizedBox(height: 8),
                _buildForgotPassword(context),
                const SizedBox(height: 24),
                _buildLoginButton(isLoading),
                const SizedBox(height: 24),
                _buildDivider(),
                const SizedBox(height: 24),
                _buildSocialButtons(isLoading),
                const SizedBox(height: 32),
                _buildRegisterLink(context),
              ],
            ),
          ),
        ),
      ),
    );
  }

  Widget _buildLogo() => Center(
        child: Container(
          width: 80,
          height: 80,
          decoration: BoxDecoration(
            color: Colors.orange,
            borderRadius: BorderRadius.circular(20),
          ),
          child: const Icon(Icons.fastfood, color: Colors.white, size: 48),
        ),
      );

  Widget _buildTitle() => Column(
        children: [
          Text(
            'ยินดีต้อนรับกลับ!',
            style: Theme.of(context).textTheme.headlineMedium?.copyWith(
                  fontWeight: FontWeight.bold,
                ),
            textAlign: TextAlign.center,
          ),
          const SizedBox(height: 8),
          Text(
            'เข้าสู่ระบบเพื่อสั่งอาหารอร่อยๆ',
            style: Theme.of(context)
                .textTheme
                .bodyMedium
                ?.copyWith(color: Colors.grey),
            textAlign: TextAlign.center,
          ),
        ],
      );

  Widget _buildEmailField() => TextFormField(
        controller: _emailCtrl,
        keyboardType: TextInputType.emailAddress,
        textInputAction: TextInputAction.next,
        decoration: const InputDecoration(
          labelText: 'อีเมล',
          prefixIcon: Icon(Icons.email_outlined),
          border: OutlineInputBorder(),
        ),
        validator: (v) {
          if (v == null || v.isEmpty) return 'กรุณากรอกอีเมล';
          if (!RegExp(r'^[\w-.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(v)) {
            return 'รูปแบบอีเมลไม่ถูกต้อง';
          }
          return null;
        },
      );

  Widget _buildPasswordField() => TextFormField(
        controller: _passCtrl,
        obscureText: _obscurePassword,
        textInputAction: TextInputAction.done,
        onFieldSubmitted: (_) => _onLogin(),
        decoration: InputDecoration(
          labelText: 'รหัสผ่าน',
          prefixIcon: const Icon(Icons.lock_outlined),
          suffixIcon: IconButton(
            icon: Icon(
              _obscurePassword ? Icons.visibility_off : Icons.visibility,
            ),
            onPressed: () =>
                setState(() => _obscurePassword = !_obscurePassword),
          ),
          border: const OutlineInputBorder(),
        ),
        validator: (v) {
          if (v == null || v.isEmpty) return 'กรุณากรอกรหัสผ่าน';
          if (v.length < 6) return 'รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร';
          return null;
        },
      );

  Widget _buildForgotPassword(BuildContext context) => Align(
        alignment: Alignment.centerRight,
        child: TextButton(
          onPressed: () => context.push('/forgot-password'),
          child: const Text('ลืมรหัสผ่าน?'),
        ),
      );

  Widget _buildLoginButton(bool isLoading) => ElevatedButton(
        onPressed: isLoading ? null : _onLogin,
        style: ElevatedButton.styleFrom(
          backgroundColor: Colors.orange,
          foregroundColor: Colors.white,
          minimumSize: const Size.fromHeight(52),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(12),
          ),
        ),
        child: isLoading
            ? const SizedBox(
                width: 24,
                height: 24,
                child: CircularProgressIndicator(
                  color: Colors.white,
                  strokeWidth: 2,
                ),
              )
            : const Text('เข้าสู่ระบบ',
                style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
      );

  Widget _buildDivider() => Row(
        children: [
          const Expanded(child: Divider()),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            child: Text('หรือ', style: TextStyle(color: Colors.grey[600])),
          ),
          const Expanded(child: Divider()),
        ],
      );

  Widget _buildSocialButtons(bool isLoading) => Column(
        children: [
          _SocialButton(
            label: 'เข้าสู่ระบบด้วย Google',
            iconAsset: 'assets/icons/google.png',
            onPressed: isLoading
                ? null
                : () => ref
                    .read(authNotifierProvider.notifier)
                    .loginWithGoogle(),
          ),
          const SizedBox(height: 12),
          _SocialButton(
            label: 'เข้าสู่ระบบด้วย Apple',
            iconWidget: const Icon(Icons.apple, size: 24),
            onPressed: isLoading
                ? null
                : () => ref
                    .read(authNotifierProvider.notifier)
                    .loginWithApple(),
          ),
        ],
      );

  Widget _buildRegisterLink(BuildContext context) => Row(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          const Text('ยังไม่มีบัญชี? '),
          TextButton(
            onPressed: () => context.push('/register'),
            child: const Text(
              'สมัครสมาชิก',
              style: TextStyle(
                color: Colors.orange,
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
        ],
      );
}

class _SocialButton extends StatelessWidget {
  final String label;
  final String? iconAsset;
  final Widget? iconWidget;
  final VoidCallback? onPressed;

  const _SocialButton({
    required this.label,
    this.iconAsset,
    this.iconWidget,
    this.onPressed,
  });

  @override
  Widget build(BuildContext context) {
    return OutlinedButton(
      onPressed: onPressed,
      style: OutlinedButton.styleFrom(
        minimumSize: const Size.fromHeight(52),
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(12),
        ),
        side: BorderSide(color: Colors.grey[300]!),
      ),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          if (iconAsset != null)
            Image.asset(iconAsset!, width: 24, height: 24)
          else if (iconWidget != null)
            iconWidget!,
          const SizedBox(width: 12),
          Text(label, style: const TextStyle(fontSize: 15)),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3565: Register Page

```dart
// lib/features/auth/presentation/pages/register_page.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

class RegisterPage extends ConsumerStatefulWidget {
  const RegisterPage({super.key});

  @override
  ConsumerState<RegisterPage> createState() => _RegisterPageState();
}

class _RegisterPageState extends ConsumerState<RegisterPage> {
  final _formKey = GlobalKey<FormState>();
  final _nameCtrl = TextEditingController();
  final _emailCtrl = TextEditingController();
  final _passCtrl = TextEditingController();
  final _confirmPassCtrl = TextEditingController();
  bool _obscurePassword = true;
  bool _obscureConfirm = true;
  bool _acceptTerms = false;

  @override
  void dispose() {
    _nameCtrl.dispose();
    _emailCtrl.dispose();
    _passCtrl.dispose();
    _confirmPassCtrl.dispose();
    super.dispose();
  }

  Future<void> _onRegister() async {
    if (!_formKey.currentState!.validate()) return;
    if (!_acceptTerms) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('กรุณายอมรับเงื่อนไขการใช้งาน')),
      );
      return;
    }
    await ref.read(authNotifierProvider.notifier).register(
          email: _emailCtrl.text.trim(),
          password: _passCtrl.text,
          displayName: _nameCtrl.text.trim(),
        );
  }

  @override
  Widget build(BuildContext context) {
    ref.listen(authNotifierProvider, (_, next) {
      next.whenOrNull(
        data: (user) {
          if (user != null) {
            ScaffoldMessenger.of(context).showSnackBar(
              const SnackBar(
                content: Text(
                    'สมัครสมาชิกสำเร็จ! กรุณายืนยันอีเมลของคุณ'),
                backgroundColor: Colors.green,
              ),
            );
            context.go('/home');
          }
        },
        error: (err, _) {
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(
              content: Text(err.toString()),
              backgroundColor: Colors.red,
            ),
          );
        },
      );
    });

    final isLoading = ref.watch(authNotifierProvider).isLoading;

    return Scaffold(
      appBar: AppBar(
        title: const Text('สมัครสมาชิก'),
        backgroundColor: Colors.transparent,
        elevation: 0,
      ),
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(24),
          child: Form(
            key: _formKey,
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                _buildNameField(),
                const SizedBox(height: 16),
                _buildEmailField(),
                const SizedBox(height: 16),
                _buildPasswordField(),
                const SizedBox(height: 16),
                _buildConfirmPasswordField(),
                const SizedBox(height: 16),
                _buildTermsCheckbox(),
                const SizedBox(height: 24),
                _buildRegisterButton(isLoading),
                const SizedBox(height: 16),
                _buildLoginLink(context),
              ],
            ),
          ),
        ),
      ),
    );
  }

  Widget _buildNameField() => TextFormField(
        controller: _nameCtrl,
        textInputAction: TextInputAction.next,
        decoration: const InputDecoration(
          labelText: 'ชื่อ-นามสกุล',
          prefixIcon: Icon(Icons.person_outlined),
          border: OutlineInputBorder(),
        ),
        validator: (v) {
          if (v == null || v.trim().isEmpty) return 'กรุณากรอกชื่อ';
          if (v.trim().length < 2) return 'ชื่อต้องมีอย่างน้อย 2 ตัวอักษร';
          return null;
        },
      );

  Widget _buildEmailField() => TextFormField(
        controller: _emailCtrl,
        keyboardType: TextInputType.emailAddress,
        textInputAction: TextInputAction.next,
        decoration: const InputDecoration(
          labelText: 'อีเมล',
          prefixIcon: Icon(Icons.email_outlined),
          border: OutlineInputBorder(),
        ),
        validator: (v) {
          if (v == null || v.isEmpty) return 'กรุณากรอกอีเมล';
          if (!RegExp(r'^[\w-.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(v)) {
            return 'รูปแบบอีเมลไม่ถูกต้อง';
          }
          return null;
        },
      );

  Widget _buildPasswordField() => TextFormField(
        controller: _passCtrl,
        obscureText: _obscurePassword,
        textInputAction: TextInputAction.next,
        decoration: InputDecoration(
          labelText: 'รหัสผ่าน',
          prefixIcon: const Icon(Icons.lock_outlined),
          suffixIcon: IconButton(
            icon: Icon(
              _obscurePassword ? Icons.visibility_off : Icons.visibility,
            ),
            onPressed: () =>
                setState(() => _obscurePassword = !_obscurePassword),
          ),
          border: const OutlineInputBorder(),
        ),
        validator: (v) {
          if (v == null || v.isEmpty) return 'กรุณากรอกรหัสผ่าน';
          if (v.length < 6) return 'รหัสผ่านต้องมีอย่างน้อย 6 ตัวอักษร';
          if (!RegExp(r'(?=.*[A-Z])').hasMatch(v)) {
            return 'ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว';
          }
          return null;
        },
      );

  Widget _buildConfirmPasswordField() => TextFormField(
        controller: _confirmPassCtrl,
        obscureText: _obscureConfirm,
        textInputAction: TextInputAction.done,
        decoration: InputDecoration(
          labelText: 'ยืนยันรหัสผ่าน',
          prefixIcon: const Icon(Icons.lock_outlined),
          suffixIcon: IconButton(
            icon: Icon(
              _obscureConfirm ? Icons.visibility_off : Icons.visibility,
            ),
            onPressed: () =>
                setState(() => _obscureConfirm = !_obscureConfirm),
          ),
          border: const OutlineInputBorder(),
        ),
        validator: (v) {
          if (v != _passCtrl.text) return 'รหัสผ่านไม่ตรงกัน';
          return null;
        },
      );

  Widget _buildTermsCheckbox() => Row(
        children: [
          Checkbox(
            value: _acceptTerms,
            activeColor: Colors.orange,
            onChanged: (v) => setState(() => _acceptTerms = v ?? false),
          ),
          Expanded(
            child: GestureDetector(
              onTap: () => setState(() => _acceptTerms = !_acceptTerms),
              child: RichText(
                text: const TextSpan(
                  style: TextStyle(color: Colors.black87, fontSize: 13),
                  children: [
                    TextSpan(text: 'ฉันยอมรับ '),
                    TextSpan(
                      text: 'เงื่อนไขการใช้งาน',
                      style: TextStyle(
                          color: Colors.orange, fontWeight: FontWeight.bold),
                    ),
                    TextSpan(text: ' และ '),
                    TextSpan(
                      text: 'นโยบายความเป็นส่วนตัว',
                      style: TextStyle(
                          color: Colors.orange, fontWeight: FontWeight.bold),
                    ),
                  ],
                ),
              ),
            ),
          ),
        ],
      );

  Widget _buildRegisterButton(bool isLoading) => ElevatedButton(
        onPressed: isLoading ? null : _onRegister,
        style: ElevatedButton.styleFrom(
          backgroundColor: Colors.orange,
          foregroundColor: Colors.white,
          minimumSize: const Size.fromHeight(52),
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(12),
          ),
        ),
        child: isLoading
            ? const CircularProgressIndicator(color: Colors.white)
            : const Text('สมัครสมาชิก',
                style:
                    TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
      );

  Widget _buildLoginLink(BuildContext context) => Row(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          const Text('มีบัญชีอยู่แล้ว? '),
          TextButton(
            onPressed: () => context.pop(),
            child: const Text('เข้าสู่ระบบ',
                style: TextStyle(
                    color: Colors.orange, fontWeight: FontWeight.bold)),
          ),
        ],
      );
}
```

---

## ขั้นตอนที่ 3566: Phone OTP Page

```dart
// lib/features/auth/presentation/pages/phone_otp_page.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:pin_code_fields/pin_code_fields.dart';

final phoneOtpStateProvider = StateNotifierProvider.autoDispose<
    PhoneOtpNotifier, PhoneOtpState>((ref) {
  return PhoneOtpNotifier(ref.watch(authRepositoryProvider));
});

class PhoneOtpState {
  final String? verificationId;
  final bool isLoading;
  final bool isSending;
  final String? error;
  final int resendCountdown;

  const PhoneOtpState({
    this.verificationId,
    this.isLoading = false,
    this.isSending = false,
    this.error,
    this.resendCountdown = 0,
  });

  PhoneOtpState copyWith({
    String? verificationId,
    bool? isLoading,
    bool? isSending,
    String? error,
    int? resendCountdown,
  }) {
    return PhoneOtpState(
      verificationId: verificationId ?? this.verificationId,
      isLoading: isLoading ?? this.isLoading,
      isSending: isSending ?? this.isSending,
      error: error,
      resendCountdown: resendCountdown ?? this.resendCountdown,
    );
  }
}

class PhoneOtpNotifier extends StateNotifier<PhoneOtpState> {
  final AuthRepository _repository;

  PhoneOtpNotifier(this._repository) : super(const PhoneOtpState());

  Future<void> sendOtp(String phoneNumber) async {
    state = state.copyWith(isSending: true, error: null);
    final result = await _repository.sendPhoneOtp(phoneNumber);
    result.fold(
      (failure) => state = state.copyWith(
        isSending: false,
        error: failure.message,
      ),
      (verificationId) => state = state.copyWith(
        isSending: false,
        verificationId: verificationId,
        resendCountdown: 60,
      ),
    );
  }

  Future<bool> verifyOtp(String otp) async {
    if (state.verificationId == null) return false;
    state = state.copyWith(isLoading: true, error: null);
    final result = await _repository.verifyPhoneOtp(
      verificationId: state.verificationId!,
      otp: otp,
    );
    return result.fold(
      (failure) {
        state = state.copyWith(isLoading: false, error: failure.message);
        return false;
      },
      (_) {
        state = state.copyWith(isLoading: false);
        return true;
      },
    );
  }
}

class PhoneOtpPage extends ConsumerStatefulWidget {
  final String phoneNumber;
  const PhoneOtpPage({super.key, required this.phoneNumber});

  @override
  ConsumerState<PhoneOtpPage> createState() => _PhoneOtpPageState();
}

class _PhoneOtpPageState extends ConsumerState<PhoneOtpPage> {
  String _otp = '';

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addPostFrameCallback((_) {
      ref.read(phoneOtpStateProvider.notifier).sendOtp(widget.phoneNumber);
    });
  }

  @override
  Widget build(BuildContext context) {
    final otpState = ref.watch(phoneOtpStateProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('ยืนยัน OTP')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.center,
          children: [
            const SizedBox(height: 32),
            const Icon(Icons.sms_outlined, size: 64, color: Colors.orange),
            const SizedBox(height: 24),
            Text(
              'กรอกรหัส OTP ที่ส่งไปยัง',
              style: Theme.of(context).textTheme.titleMedium,
            ),
            Text(
              widget.phoneNumber,
              style: Theme.of(context).textTheme.titleLarge?.copyWith(
                    fontWeight: FontWeight.bold,
                  ),
            ),
            const SizedBox(height: 32),
            PinCodeTextField(
              appContext: context,
              length: 6,
              onChanged: (value) => _otp = value,
              onCompleted: (value) async {
                _otp = value;
                final success = await ref
                    .read(phoneOtpStateProvider.notifier)
                    .verifyOtp(value);
                if (success && mounted) context.go('/home');
              },
              pinTheme: PinTheme(
                shape: PinCodeFieldShape.box,
                borderRadius: BorderRadius.circular(8),
                fieldHeight: 56,
                fieldWidth: 48,
                activeFillColor: Colors.orange.shade50,
                selectedFillColor: Colors.orange.shade100,
                inactiveFillColor: Colors.grey.shade100,
                activeColor: Colors.orange,
                selectedColor: Colors.orange,
                inactiveColor: Colors.grey.shade300,
              ),
              enableActiveFill: true,
              keyboardType: TextInputType.number,
            ),
            if (otpState.error != null) ...[
              const SizedBox(height: 12),
              Text(
                otpState.error!,
                style: const TextStyle(color: Colors.red),
              ),
            ],
            const SizedBox(height: 24),
            if (otpState.isLoading)
              const CircularProgressIndicator(color: Colors.orange)
            else
              ElevatedButton(
                onPressed: _otp.length == 6
                    ? () async {
                        final success = await ref
                            .read(phoneOtpStateProvider.notifier)
                            .verifyOtp(_otp);
                        if (success && mounted) context.go('/home');
                      }
                    : null,
                style: ElevatedButton.styleFrom(
                  backgroundColor: Colors.orange,
                  minimumSize: const Size(200, 52),
                  shape: RoundedRectangleBorder(
                    borderRadius: BorderRadius.circular(12),
                  ),
                ),
                child: const Text('ยืนยัน OTP'),
              ),
            const SizedBox(height: 16),
            if (otpState.isSending)
              const Text('กำลังส่ง OTP...')
            else
              TextButton(
                onPressed: otpState.resendCountdown > 0
                    ? null
                    : () => ref
                        .read(phoneOtpStateProvider.notifier)
                        .sendOtp(widget.phoneNumber),
                child: Text(
                  otpState.resendCountdown > 0
                      ? 'ส่งอีกครั้งใน ${otpState.resendCountdown}s'
                      : 'ส่ง OTP อีกครั้ง',
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

## ขั้นตอนที่ 3567: App Router with Auth Guard

```dart
// lib/core/router/app_router.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

final appRouterProvider = Provider<GoRouter>((ref) {
  final authNotifier = ref.watch(authNotifierProvider.notifier);

  return GoRouter(
    initialLocation: '/splash',
    refreshListenable: GoRouterRefreshStream(
      ref.watch(authStateStreamProvider.stream),
    ),
    redirect: (context, state) {
      final isAuthenticated = ref.read(isAuthenticatedProvider);
      final isSplash = state.matchedLocation == '/splash';
      final isAuthRoute = state.matchedLocation.startsWith('/auth');

      if (isSplash) return null;

      if (!isAuthenticated && !isAuthRoute) {
        return '/auth/login';
      }

      if (isAuthenticated && isAuthRoute) {
        return '/home';
      }

      return null;
    },
    routes: [
      GoRoute(
        path: '/splash',
        builder: (_, __) => const SplashPage(),
      ),
      GoRoute(
        path: '/auth/login',
        builder: (_, __) => const LoginPage(),
      ),
      GoRoute(
        path: '/auth/register',
        builder: (_, __) => const RegisterPage(),
      ),
      GoRoute(
        path: '/auth/forgot-password',
        builder: (_, __) => const ForgotPasswordPage(),
      ),
      GoRoute(
        path: '/auth/phone-otp',
        builder: (_, state) {
          final phone = state.uri.queryParameters['phone'] ?? '';
          return PhoneOtpPage(phoneNumber: phone);
        },
      ),
      ShellRoute(
        builder: (_, __, child) => MainShell(child: child),
        routes: [
          GoRoute(path: '/home', builder: (_, __) => const HomePage()),
          GoRoute(
            path: '/restaurant/:id',
            builder: (_, state) =>
                RestaurantDetailPage(id: state.pathParameters['id']!),
          ),
          GoRoute(path: '/cart', builder: (_, __) => const CartPage()),
          GoRoute(
            path: '/order/:id',
            builder: (_, state) =>
                OrderTrackingPage(orderId: state.pathParameters['id']!),
          ),
          GoRoute(path: '/profile', builder: (_, __) => const ProfilePage()),
        ],
      ),
    ],
  );
});

class GoRouterRefreshStream extends ChangeNotifier {
  GoRouterRefreshStream(Stream<dynamic> stream) {
    notifyListeners();
    _subscription = stream.asBroadcastStream().listen((_) => notifyListeners());
  }

  late final dynamic _subscription;

  @override
  void dispose() {
    (_subscription as dynamic).cancel();
    super.dispose();
  }
}
```

---

**← [Part 91](part-91-capstone-project-planning.md)**
**ต่อไป: [Part 93 →](part-93-capstone-restaurant-module.md)**
