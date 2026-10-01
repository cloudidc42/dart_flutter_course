# Part 16: Firebase
## ขั้นตอนที่ 521-560

---

## 🎯 เป้าหมายของ Part นี้

- Setup Firebase Project
- Firebase Authentication
- Cloud Firestore
- Firebase Storage
- Firebase Realtime Database
- Firebase Cloud Messaging (Push Notifications)

---

## ขั้นตอนที่ 521: Firebase Setup

```bash
# 1. ติดตั้ง Firebase CLI
npm install -g firebase-tools

# 2. Login
firebase login

# 3. ติดตั้ง FlutterFire CLI
dart pub global activate flutterfire_cli

# 4. Configure Firebase สำหรับ Flutter project
flutterfire configure

# pubspec.yaml:
# dependencies:
#   firebase_core: ^2.24.0
#   firebase_auth: ^4.16.0
#   cloud_firestore: ^4.14.0
#   firebase_storage: ^11.6.0
#   firebase_messaging: ^14.7.10
```

```dart
// main.dart
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/material.dart';
import 'firebase_options.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  runApp(const MyApp());
}
```

---

## ขั้นตอนที่ 522: Firebase Authentication

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/material.dart';

class AuthService {
  final FirebaseAuth _auth = FirebaseAuth.instance;
  
  // Stream of auth state changes
  Stream<User?> get authStateChanges => _auth.authStateChanges();
  
  User? get currentUser => _auth.currentUser;
  bool get isLoggedIn => currentUser != null;
  
  // Register with Email/Password
  Future<UserCredential> register({
    required String email,
    required String password,
    required String displayName,
  }) async {
    UserCredential credential = await _auth.createUserWithEmailAndPassword(
      email: email,
      password: password,
    );
    
    await credential.user?.updateDisplayName(displayName);
    await credential.user?.sendEmailVerification();
    
    return credential;
  }
  
  // Sign In with Email/Password
  Future<UserCredential> signIn({
    required String email,
    required String password,
  }) async {
    return _auth.signInWithEmailAndPassword(
      email: email,
      password: password,
    );
  }
  
  // Sign Out
  Future<void> signOut() => _auth.signOut();
  
  // Password Reset
  Future<void> resetPassword(String email) {
    return _auth.sendPasswordResetEmail(email: email);
  }
  
  // Update Profile
  Future<void> updateProfile({String? displayName, String? photoUrl}) async {
    await currentUser?.updateDisplayName(displayName);
    await currentUser?.updatePhotoURL(photoUrl);
  }
  
  // Handle Firebase Auth Errors
  String getErrorMessage(FirebaseAuthException e) {
    return switch (e.code) {
      'user-not-found' => 'ไม่พบ Email นี้',
      'wrong-password' => 'Password ไม่ถูกต้อง',
      'email-already-in-use' => 'Email นี้ถูกใช้แล้ว',
      'weak-password' => 'Password ไม่ปลอดภัยพอ',
      'invalid-email' => 'Email ไม่ถูกต้อง',
      'user-disabled' => 'บัญชีถูกระงับ',
      'too-many-requests' => 'ลองอีกครั้งในภายหลัง',
      _ => 'เกิดข้อผิดพลาด: ${e.message}',
    };
  }
}

// ─── Auth Screen ───
class AuthScreen extends StatefulWidget {
  const AuthScreen({super.key});
  
  @override
  State<AuthScreen> createState() => _AuthScreenState();
}

class _AuthScreenState extends State<AuthScreen> {
  final AuthService _auth = AuthService();
  final _formKey = GlobalKey<FormState>();
  final _emailCtrl = TextEditingController();
  final _passwordCtrl = TextEditingController();
  final _nameCtrl = TextEditingController();
  
  bool _isLogin = true;
  bool _isLoading = false;
  bool _obscurePassword = true;
  
  @override
  void dispose() {
    _emailCtrl.dispose();
    _passwordCtrl.dispose();
    _nameCtrl.dispose();
    super.dispose();
  }
  
  Future<void> _submit() async {
    if (!_formKey.currentState!.validate()) return;
    
    setState(() => _isLoading = true);
    
    try {
      if (_isLogin) {
        await _auth.signIn(email: _emailCtrl.text.trim(), password: _passwordCtrl.text);
      } else {
        await _auth.register(
          email: _emailCtrl.text.trim(),
          password: _passwordCtrl.text,
          displayName: _nameCtrl.text.trim(),
        );
      }
    } on FirebaseAuthException catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(_auth.getErrorMessage(e))),
        );
      }
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(24),
          child: Form(
            key: _formKey,
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.stretch,
              children: [
                const SizedBox(height: 40),
                const Icon(Icons.lock_outlined, size: 80, color: Colors.blue),
                const SizedBox(height: 24),
                Text(
                  _isLogin ? 'เข้าสู่ระบบ' : 'สมัครสมาชิก',
                  textAlign: TextAlign.center,
                  style: const TextStyle(fontSize: 28, fontWeight: FontWeight.bold),
                ),
                const SizedBox(height: 32),
                
                if (!_isLogin)
                  TextFormField(
                    controller: _nameCtrl,
                    decoration: const InputDecoration(
                      labelText: 'ชื่อ',
                      prefixIcon: Icon(Icons.person),
                      border: OutlineInputBorder(),
                    ),
                    validator: (v) => v?.trim().isEmpty == true ? 'กรุณาใส่ชื่อ' : null,
                  ),
                
                if (!_isLogin) const SizedBox(height: 16),
                
                TextFormField(
                  controller: _emailCtrl,
                  decoration: const InputDecoration(
                    labelText: 'Email',
                    prefixIcon: Icon(Icons.email),
                    border: OutlineInputBorder(),
                  ),
                  keyboardType: TextInputType.emailAddress,
                  validator: (v) {
                    if (v?.trim().isEmpty == true) return 'กรุณาใส่ Email';
                    if (!RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(v!)) {
                      return 'Email ไม่ถูกต้อง';
                    }
                    return null;
                  },
                ),
                const SizedBox(height: 16),
                
                TextFormField(
                  controller: _passwordCtrl,
                  decoration: InputDecoration(
                    labelText: 'Password',
                    prefixIcon: const Icon(Icons.lock),
                    border: const OutlineInputBorder(),
                    suffixIcon: IconButton(
                      icon: Icon(_obscurePassword ? Icons.visibility : Icons.visibility_off),
                      onPressed: () => setState(() => _obscurePassword = !_obscurePassword),
                    ),
                  ),
                  obscureText: _obscurePassword,
                  validator: (v) {
                    if (v?.isEmpty == true) return 'กรุณาใส่ Password';
                    if (!_isLogin && v!.length < 8) return 'Password ต้องมีอย่างน้อย 8 ตัว';
                    return null;
                  },
                ),
                const SizedBox(height: 24),
                
                if (_isLoading)
                  const Center(child: CircularProgressIndicator())
                else
                  ElevatedButton(
                    onPressed: _submit,
                    style: ElevatedButton.styleFrom(
                      padding: const EdgeInsets.all(16),
                    ),
                    child: Text(_isLogin ? 'เข้าสู่ระบบ' : 'สมัครสมาชิก', style: const TextStyle(fontSize: 16)),
                  ),
                
                const SizedBox(height: 16),
                
                if (_isLogin)
                  TextButton(
                    onPressed: () {
                      if (_emailCtrl.text.isNotEmpty) {
                        _auth.resetPassword(_emailCtrl.text.trim());
                        ScaffoldMessenger.of(context).showSnackBar(
                          const SnackBar(content: Text('ส่ง Email รีเซ็ต Password แล้ว')),
                        );
                      }
                    },
                    child: const Text('ลืม Password?'),
                  ),
                
                TextButton(
                  onPressed: () => setState(() => _isLogin = !_isLogin),
                  child: Text(_isLogin ? 'ยังไม่มีบัญชี? สมัครสมาชิก' : 'มีบัญชีแล้ว? เข้าสู่ระบบ'),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}

// ─── Auth Wrapper ───
class AuthWrapper extends StatelessWidget {
  final Widget homeScreen;
  final Widget authScreen;
  
  const AuthWrapper({super.key, required this.homeScreen, required this.authScreen});
  
  @override
  Widget build(BuildContext context) {
    return StreamBuilder<User?>(
      stream: FirebaseAuth.instance.authStateChanges(),
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.waiting) {
          return const Scaffold(body: Center(child: CircularProgressIndicator()));
        }
        
        if (snapshot.hasData) {
          return homeScreen;
        }
        
        return authScreen;
      },
    );
  }
}
```

---

## ขั้นตอนที่ 523: Cloud Firestore

```dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';

// ─── Model ───
class UserProfile {
  final String uid;
  final String displayName;
  final String email;
  final String? photoUrl;
  final DateTime createdAt;
  final Map<String, dynamic>? preferences;
  
  const UserProfile({
    required this.uid,
    required this.displayName,
    required this.email,
    this.photoUrl,
    required this.createdAt,
    this.preferences,
  });
  
  factory UserProfile.fromFirestore(DocumentSnapshot<Map<String, dynamic>> doc) {
    Map<String, dynamic> data = doc.data()!;
    return UserProfile(
      uid: doc.id,
      displayName: data['displayName'] ?? '',
      email: data['email'] ?? '',
      photoUrl: data['photoUrl'],
      createdAt: (data['createdAt'] as Timestamp).toDate(),
      preferences: data['preferences'],
    );
  }
  
  Map<String, dynamic> toFirestore() => {
    'displayName': displayName,
    'email': email,
    if (photoUrl != null) 'photoUrl': photoUrl,
    'createdAt': Timestamp.fromDate(createdAt),
    if (preferences != null) 'preferences': preferences,
  };
}

class Post {
  final String? id;
  final String userId;
  final String content;
  final List<String> imageUrls;
  final int likes;
  final DateTime createdAt;
  final GeoPoint? location;
  
  const Post({
    this.id,
    required this.userId,
    required this.content,
    this.imageUrls = const [],
    this.likes = 0,
    required this.createdAt,
    this.location,
  });
  
  factory Post.fromFirestore(DocumentSnapshot<Map<String, dynamic>> doc) {
    Map<String, dynamic> data = doc.data()!;
    return Post(
      id: doc.id,
      userId: data['userId'] ?? '',
      content: data['content'] ?? '',
      imageUrls: List<String>.from(data['imageUrls'] ?? []),
      likes: data['likes'] ?? 0,
      createdAt: (data['createdAt'] as Timestamp).toDate(),
      location: data['location'],
    );
  }
  
  Map<String, dynamic> toFirestore() => {
    'userId': userId,
    'content': content,
    'imageUrls': imageUrls,
    'likes': likes,
    'createdAt': FieldValue.serverTimestamp(),
    if (location != null) 'location': location,
  };
}

// ─── Firestore Service ───
class FirestoreService {
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  final FirebaseAuth _auth = FirebaseAuth.instance;
  
  String? get _userId => _auth.currentUser?.uid;
  
  // ─── Users ───
  CollectionReference<Map<String, dynamic>> get _users => _db.collection('users');
  
  Future<void> createUserProfile(UserProfile profile) async {
    await _users.doc(profile.uid).set(profile.toFirestore());
  }
  
  Future<UserProfile?> getUserProfile(String uid) async {
    DocumentSnapshot<Map<String, dynamic>> doc = await _users.doc(uid).get();
    if (!doc.exists) return null;
    return UserProfile.fromFirestore(doc);
  }
  
  Stream<UserProfile?> userProfileStream(String uid) {
    return _users.doc(uid)
        .withConverter<UserProfile?>(
          fromFirestore: (snap, _) => snap.exists ? UserProfile.fromFirestore(snap) : null,
          toFirestore: (profile, _) => profile?.toFirestore() ?? {},
        )
        .snapshots()
        .map((snap) => snap.data());
  }
  
  Future<void> updateUserProfile(String uid, Map<String, dynamic> updates) async {
    await _users.doc(uid).update(updates);
  }
  
  // ─── Posts ───
  CollectionReference<Map<String, dynamic>> get _posts => _db.collection('posts');
  
  Future<DocumentReference> createPost(Post post) async {
    return _posts.add(post.toFirestore());
  }
  
  Stream<List<Post>> postsStream({int limit = 20}) {
    return _posts
        .orderBy('createdAt', descending: true)
        .limit(limit)
        .snapshots()
        .map((snapshot) => snapshot.docs.map(Post.fromFirestore).toList());
  }
  
  Stream<List<Post>> userPostsStream(String userId) {
    return _posts
        .where('userId', isEqualTo: userId)
        .orderBy('createdAt', descending: true)
        .snapshots()
        .map((snapshot) => snapshot.docs.map(Post.fromFirestore).toList());
  }
  
  Future<void> likePost(String postId) async {
    await _posts.doc(postId).update({
      'likes': FieldValue.increment(1),
    });
    
    // บันทึกว่า user นี้ like แล้ว
    await _db.collection('likes').doc('${_userId}_$postId').set({
      'userId': _userId,
      'postId': postId,
      'likedAt': FieldValue.serverTimestamp(),
    });
  }
  
  Future<bool> hasLiked(String postId) async {
    DocumentSnapshot doc = await _db
        .collection('likes')
        .doc('${_userId}_$postId')
        .get();
    return doc.exists;
  }
  
  Future<void> deletePost(String postId) async {
    // Delete post and its subcollections
    await _posts.doc(postId).delete();
  }
  
  // ─── Query with pagination ───
  Future<List<Post>> getPostsPaginated({
    int limit = 10,
    DocumentSnapshot? lastDoc,
  }) async {
    Query<Map<String, dynamic>> query = _posts
        .orderBy('createdAt', descending: true)
        .limit(limit);
    
    if (lastDoc != null) {
      query = query.startAfterDocument(lastDoc);
    }
    
    QuerySnapshot<Map<String, dynamic>> snapshot = await query.get();
    return snapshot.docs.map(Post.fromFirestore).toList();
  }
  
  // ─── Transactions ───
  Future<void> transferPoints(String fromUserId, String toUserId, int points) async {
    await _db.runTransaction((transaction) async {
      DocumentReference fromRef = _users.doc(fromUserId);
      DocumentReference toRef = _users.doc(toUserId);
      
      DocumentSnapshot fromSnap = await transaction.get(fromRef);
      DocumentSnapshot toSnap = await transaction.get(toRef);
      
      int fromPoints = (fromSnap.data() as Map)['points'] ?? 0;
      int toPoints = (toSnap.data() as Map)['points'] ?? 0;
      
      if (fromPoints < points) {
        throw Exception('คะแนนไม่เพียงพอ');
      }
      
      transaction.update(fromRef, {'points': fromPoints - points});
      transaction.update(toRef, {'points': toPoints + points});
    });
  }
  
  // ─── Batch writes ───
  Future<void> deleteUserData(String userId) async {
    WriteBatch batch = _db.batch();
    
    // Delete posts
    QuerySnapshot posts = await _posts.where('userId', isEqualTo: userId).get();
    for (DocumentSnapshot doc in posts.docs) {
      batch.delete(doc.reference);
    }
    
    // Delete profile
    batch.delete(_users.doc(userId));
    
    await batch.commit();
  }
}
```

---

## ขั้นตอนที่ 524: Firebase Storage

```dart
import 'package:firebase_storage/firebase_storage.dart';
import 'dart:io';

class StorageService {
  final FirebaseStorage _storage = FirebaseStorage.instance;
  
  // Upload image
  Future<String> uploadImage(File file, String path) async {
    Reference ref = _storage.ref(path);
    
    UploadTask task = ref.putFile(
      file,
      SettableMetadata(contentType: 'image/jpeg'),
    );
    
    // Listen to upload progress
    task.snapshotEvents.listen((TaskSnapshot snapshot) {
      double progress = snapshot.bytesTransferred / snapshot.totalBytes;
      print('Upload progress: ${(progress * 100).toInt()}%');
    });
    
    TaskSnapshot snapshot = await task;
    return snapshot.ref.getDownloadURL();
  }
  
  // Upload with progress stream
  Stream<double> uploadImageWithProgress(File file, String path) async* {
    Reference ref = _storage.ref(path);
    UploadTask task = ref.putFile(file);
    
    await for (TaskSnapshot snapshot in task.snapshotEvents) {
      if (snapshot.totalBytes > 0) {
        yield snapshot.bytesTransferred / snapshot.totalBytes;
      }
      if (snapshot.state == TaskState.success) break;
    }
  }
  
  // Delete file
  Future<void> deleteFile(String url) async {
    Reference ref = _storage.refFromURL(url);
    await ref.delete();
  }
  
  // List files in a folder
  Future<List<Reference>> listFiles(String folder) async {
    ListResult result = await _storage.ref(folder).listAll();
    return result.items;
  }
  
  // Get download URL
  Future<String> getDownloadUrl(String path) async {
    return _storage.ref(path).getDownloadURL();
  }
}

// ─── Image Upload Widget ───
import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';

// pubspec.yaml:
// dependencies:
//   image_picker: ^1.0.7

class ImageUploadWidget extends StatefulWidget {
  final void Function(String url) onUploaded;
  
  const ImageUploadWidget({super.key, required this.onUploaded});
  
  @override
  State<ImageUploadWidget> createState() => _ImageUploadWidgetState();
}

class _ImageUploadWidgetState extends State<ImageUploadWidget> {
  final StorageService _storage = StorageService();
  final ImagePicker _picker = ImagePicker();
  
  String? _imageUrl;
  double? _uploadProgress;
  bool _isUploading = false;
  
  Future<void> _pickAndUpload() async {
    XFile? file = await _picker.pickImage(source: ImageSource.gallery);
    if (file == null) return;
    
    setState(() {
      _isUploading = true;
      _uploadProgress = 0;
    });
    
    String path = 'uploads/${DateTime.now().millisecondsSinceEpoch}_${file.name}';
    
    try {
      String url = await _storage.uploadImage(File(file.path), path);
      
      setState(() {
        _imageUrl = url;
        _isUploading = false;
        _uploadProgress = null;
      });
      
      widget.onUploaded(url);
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Upload failed: $e')),
        );
      }
      setState(() {
        _isUploading = false;
        _uploadProgress = null;
      });
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _isUploading ? null : _pickAndUpload,
      child: Container(
        width: 150,
        height: 150,
        decoration: BoxDecoration(
          border: Border.all(color: Colors.grey),
          borderRadius: BorderRadius.circular(8),
        ),
        child: _isUploading
            ? Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  CircularProgressIndicator(value: _uploadProgress),
                  const SizedBox(height: 8),
                  Text('${((_uploadProgress ?? 0) * 100).toInt()}%'),
                ],
              )
            : _imageUrl != null
                ? ClipRRect(
                    borderRadius: BorderRadius.circular(8),
                    child: Image.network(_imageUrl!, fit: BoxFit.cover),
                  )
                : const Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      Icon(Icons.add_photo_alternate, size: 48, color: Colors.grey),
                      Text('เลือกรูปภาพ', style: TextStyle(color: Colors.grey)),
                    ],
                  ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 525-540: Firebase Realtime Database

```dart
import 'package:firebase_database/firebase_database.dart';

class RealtimeDbService {
  final FirebaseDatabase _db = FirebaseDatabase.instance;
  
  // ─── Chat example ───
  DatabaseReference get _chatRef => _db.ref('chats');
  DatabaseReference get _presenceRef => _db.ref('presence');
  
  // Send message
  Future<void> sendMessage(String chatId, String userId, String content) async {
    await _chatRef.child(chatId).child('messages').push().set({
      'userId': userId,
      'content': content,
      'timestamp': ServerValue.timestamp,
    });
    
    // Update last message
    await _chatRef.child(chatId).update({
      'lastMessage': content,
      'lastMessageTime': ServerValue.timestamp,
      'lastSenderId': userId,
    });
  }
  
  // Listen to messages
  Stream<List<Map<String, dynamic>>> messagesStream(String chatId) {
    return _chatRef
        .child(chatId)
        .child('messages')
        .orderByChild('timestamp')
        .limitToLast(50)
        .onValue
        .map((event) {
          if (event.snapshot.value == null) return [];
          
          Map<dynamic, dynamic> data = event.snapshot.value as Map;
          return data.entries
              .map((e) => {
                    'id': e.key,
                    ...Map<String, dynamic>.from(e.value as Map),
                  })
              .toList()
            ..sort((a, b) => (a['timestamp'] ?? 0).compareTo(b['timestamp'] ?? 0));
        });
  }
  
  // Online Presence
  void setOnline(String userId) {
    DatabaseReference userPresenceRef = _presenceRef.child(userId);
    
    userPresenceRef.set({
      'status': 'online',
      'lastSeen': ServerValue.timestamp,
    });
    
    // Set offline when disconnected
    userPresenceRef.onDisconnect().set({
      'status': 'offline',
      'lastSeen': ServerValue.timestamp,
    });
  }
  
  Stream<bool> isOnlineStream(String userId) {
    return _presenceRef
        .child(userId)
        .child('status')
        .onValue
        .map((event) => event.snapshot.value == 'online');
  }
  
  // Counter (atomic increment)
  Future<void> incrementCounter(String key) async {
    DatabaseReference ref = _db.ref('counters/$key');
    
    await ref.runTransaction((Object? current) {
      int count = (current as int?) ?? 0;
      return Transaction.success(count + 1);
    });
  }
}
```

---

## ขั้นตอนที่ 541-560: โปรเจกต์ - Chat App

```dart
// firebase_chat_app.dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(const ChatApp());
}

class ChatApp extends StatelessWidget {
  const ChatApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Chat App',
      theme: ThemeData(colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal), useMaterial3: true),
      home: StreamBuilder<User?>(
        stream: FirebaseAuth.instance.authStateChanges(),
        builder: (ctx, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return const Scaffold(body: Center(child: CircularProgressIndicator()));
          }
          if (snapshot.hasData) return const ChatRoomScreen();
          return const LoginScreen();
        },
      ),
    );
  }
}

// ─── Message Model ───
class Message {
  final String id;
  final String senderId;
  final String senderName;
  final String content;
  final DateTime timestamp;
  final bool isMe;
  
  Message({
    required this.id,
    required this.senderId,
    required this.senderName,
    required this.content,
    required this.timestamp,
    required this.isMe,
  });
  
  factory Message.fromFirestore(DocumentSnapshot<Map<String, dynamic>> doc, String currentUserId) {
    Map<String, dynamic> data = doc.data()!;
    return Message(
      id: doc.id,
      senderId: data['senderId'] ?? '',
      senderName: data['senderName'] ?? 'Unknown',
      content: data['content'] ?? '',
      timestamp: (data['timestamp'] as Timestamp?)?.toDate() ?? DateTime.now(),
      isMe: data['senderId'] == currentUserId,
    );
  }
}

// ─── Chat Room ───
class ChatRoomScreen extends StatefulWidget {
  const ChatRoomScreen({super.key});
  
  @override
  State<ChatRoomScreen> createState() => _ChatRoomScreenState();
}

class _ChatRoomScreenState extends State<ChatRoomScreen> {
  final TextEditingController _messageCtrl = TextEditingController();
  final ScrollController _scrollCtrl = ScrollController();
  final FirebaseFirestore _db = FirebaseFirestore.instance;
  final FirebaseAuth _auth = FirebaseAuth.instance;
  
  User? get _user => _auth.currentUser;
  
  CollectionReference<Map<String, dynamic>> get _messages =>
      _db.collection('chat_rooms').doc('global').collection('messages');
  
  Future<void> _sendMessage() async {
    String content = _messageCtrl.text.trim();
    if (content.isEmpty || _user == null) return;
    
    _messageCtrl.clear();
    
    await _messages.add({
      'senderId': _user!.uid,
      'senderName': _user!.displayName ?? 'Anonymous',
      'content': content,
      'timestamp': FieldValue.serverTimestamp(),
    });
    
    // Scroll to bottom
    await Future.delayed(const Duration(milliseconds: 100));
    if (_scrollCtrl.hasClients) {
      _scrollCtrl.animateTo(
        _scrollCtrl.position.maxScrollExtent,
        duration: const Duration(milliseconds: 300),
        curve: Curves.easeOut,
      );
    }
  }
  
  @override
  void dispose() {
    _messageCtrl.dispose();
    _scrollCtrl.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Row(
          children: [
            CircleAvatar(radius: 16, child: Icon(Icons.people, size: 18)),
            SizedBox(width: 8),
            Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text('Global Chat', style: TextStyle(fontSize: 16)),
                Text('Online', style: TextStyle(fontSize: 12, color: Colors.green)),
              ],
            ),
          ],
        ),
        actions: [
          IconButton(
            icon: const Icon(Icons.logout),
            onPressed: () => _auth.signOut(),
          ),
        ],
      ),
      body: Column(
        children: [
          Expanded(
            child: StreamBuilder<QuerySnapshot<Map<String, dynamic>>>(
              stream: _messages
                  .orderBy('timestamp', descending: false)
                  .limitToLast(100)
                  .snapshots(),
              builder: (ctx, snapshot) {
                if (!snapshot.hasData) {
                  return const Center(child: CircularProgressIndicator());
                }
                
                List<Message> msgs = snapshot.data!.docs
                    .map((doc) => Message.fromFirestore(doc, _user?.uid ?? ''))
                    .toList();
                
                if (msgs.isEmpty) {
                  return const Center(child: Text('ยังไม่มีข้อความ เริ่มต้นการสนทนา!'));
                }
                
                WidgetsBinding.instance.addPostFrameCallback((_) {
                  if (_scrollCtrl.hasClients) {
                    _scrollCtrl.jumpTo(_scrollCtrl.position.maxScrollExtent);
                  }
                });
                
                return ListView.builder(
                  controller: _scrollCtrl,
                  padding: const EdgeInsets.all(8),
                  itemCount: msgs.length,
                  itemBuilder: (ctx, i) => MessageBubble(message: msgs[i]),
                );
              },
            ),
          ),
          
          // Input
          Container(
            padding: const EdgeInsets.all(8),
            decoration: BoxDecoration(
              color: Theme.of(context).cardColor,
              boxShadow: const [BoxShadow(color: Colors.black12, blurRadius: 4, offset: Offset(0, -2))],
            ),
            child: SafeArea(
              child: Row(
                children: [
                  Expanded(
                    child: TextField(
                      controller: _messageCtrl,
                      decoration: InputDecoration(
                        hintText: 'พิมพ์ข้อความ...',
                        border: OutlineInputBorder(borderRadius: BorderRadius.circular(24)),
                        contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
                      ),
                      onSubmitted: (_) => _sendMessage(),
                    ),
                  ),
                  const SizedBox(width: 8),
                  IconButton.filled(
                    onPressed: _sendMessage,
                    icon: const Icon(Icons.send),
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

class MessageBubble extends StatelessWidget {
  final Message message;
  
  const MessageBubble({super.key, required this.message});
  
  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: message.isMe ? MainAxisAlignment.end : MainAxisAlignment.start,
        crossAxisAlignment: CrossAxisAlignment.end,
        children: [
          if (!message.isMe) ...[
            CircleAvatar(
              radius: 16,
              backgroundColor: Theme.of(context).colorScheme.primaryContainer,
              child: Text(
                message.senderName.isNotEmpty ? message.senderName[0].toUpperCase() : '?',
                style: const TextStyle(fontSize: 14),
              ),
            ),
            const SizedBox(width: 8),
          ],
          Flexible(
            child: Container(
              padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
              decoration: BoxDecoration(
                color: message.isMe
                    ? Theme.of(context).colorScheme.primary
                    : Theme.of(context).colorScheme.surfaceVariant,
                borderRadius: BorderRadius.only(
                  topLeft: const Radius.circular(16),
                  topRight: const Radius.circular(16),
                  bottomLeft: Radius.circular(message.isMe ? 16 : 4),
                  bottomRight: Radius.circular(message.isMe ? 4 : 16),
                ),
              ),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  if (!message.isMe)
                    Text(
                      message.senderName,
                      style: TextStyle(
                        fontSize: 12,
                        fontWeight: FontWeight.bold,
                        color: message.isMe
                            ? Colors.white70
                            : Theme.of(context).colorScheme.primary,
                      ),
                    ),
                  Text(
                    message.content,
                    style: TextStyle(
                      color: message.isMe ? Colors.white : null,
                    ),
                  ),
                  Text(
                    '${message.timestamp.hour.toString().padLeft(2, '0')}:${message.timestamp.minute.toString().padLeft(2, '0')}',
                    style: TextStyle(
                      fontSize: 10,
                      color: message.isMe ? Colors.white70 : Colors.grey,
                    ),
                  ),
                ],
              ),
            ),
          ),
          if (message.isMe) const SizedBox(width: 8),
        ],
      ),
    );
  }
}

// ─── Login Screen ───
class LoginScreen extends StatefulWidget {
  const LoginScreen({super.key});
  
  @override
  State<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _nameCtrl = TextEditingController();
  bool _isLoading = false;
  
  @override
  void dispose() {
    _nameCtrl.dispose();
    super.dispose();
  }
  
  Future<void> _loginAnonymously() async {
    if (_nameCtrl.text.trim().isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('กรุณาใส่ชื่อ')));
      return;
    }
    
    setState(() => _isLoading = true);
    
    try {
      UserCredential cred = await FirebaseAuth.instance.signInAnonymously();
      await cred.user?.updateDisplayName(_nameCtrl.text.trim());
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Error: $e')));
      }
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Padding(
          padding: const EdgeInsets.all(32),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Icon(Icons.chat_bubble, size: 80, color: Colors.teal),
              const SizedBox(height: 24),
              const Text('Flutter Chat', style: TextStyle(fontSize: 28, fontWeight: FontWeight.bold)),
              const SizedBox(height: 32),
              TextField(
                controller: _nameCtrl,
                decoration: const InputDecoration(
                  labelText: 'ชื่อของคุณ',
                  prefixIcon: Icon(Icons.person),
                  border: OutlineInputBorder(),
                ),
              ),
              const SizedBox(height: 16),
              if (_isLoading)
                const CircularProgressIndicator()
              else
                SizedBox(
                  width: double.infinity,
                  child: ElevatedButton(
                    onPressed: _loginAnonymously,
                    style: ElevatedButton.styleFrom(padding: const EdgeInsets.all(16)),
                    child: const Text('เข้าร่วมแชท'),
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

**← [Part 15 - Local Storage](part-15-local-storage.md)**

**ต่อไป: [Part 17 - Testing →](part-17-testing.md)**
