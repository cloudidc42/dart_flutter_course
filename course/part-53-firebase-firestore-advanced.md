# Part 53: Firebase Firestore Advanced
## ขั้นตอนที่ 2001-2040

## 🎯 เป้าหมายของ Part นี้
- เขียน Firestore Security Rules ที่ถูกต้อง
- ใช้ StreamBuilder กับ Real-time listeners
- ตั้งค่า Offline Persistence
- Batch writes และ Transactions
- Firestore Pagination ด้วย startAfter
- Firebase Storage upload พร้อม progress tracking

---

## ขั้นตอนที่ 2001: Firestore Security Rules

```
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Helper functions
    function isAuthenticated() {
      return request.auth != null;
    }

    function isOwner(userId) {
      return isAuthenticated() && request.auth.uid == userId;
    }

    function isValidUser() {
      return isAuthenticated()
        && request.auth.token.email_verified == true;
    }

    function hasRole(role) {
      return isAuthenticated()
        && get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == role;
    }

    // Users collection
    match /users/{userId} {
      allow read: if isAuthenticated();
      allow create: if isOwner(userId)
        && request.resource.data.keys().hasAll(['email', 'displayName', 'createdAt'])
        && request.resource.data.email == request.auth.token.email;
      allow update: if isOwner(userId)
        && !request.resource.data.diff(resource.data).affectedKeys().hasAny(['email', 'createdAt', 'role']);
      allow delete: if false; // ห้ามลบ user document
    }

    // Posts collection
    match /posts/{postId} {
      allow read: if resource.data.visibility == 'public'
        || (isAuthenticated() && resource.data.visibility == 'private' && resource.data.authorId == request.auth.uid);
      allow create: if isAuthenticated()
        && request.resource.data.authorId == request.auth.uid
        && request.resource.data.content.size() <= 5000
        && request.resource.data.keys().hasAll(['title', 'content', 'authorId', 'createdAt', 'visibility']);
      allow update: if isOwner(resource.data.authorId)
        && !request.resource.data.diff(resource.data).affectedKeys().hasAny(['authorId', 'createdAt']);
      allow delete: if isOwner(resource.data.authorId) || hasRole('admin');

      // Comments sub-collection
      match /comments/{commentId} {
        allow read: if true;
        allow create: if isAuthenticated()
          && request.resource.data.authorId == request.auth.uid;
        allow update, delete: if isOwner(resource.data.authorId);
      }
    }

    // Admin-only collection
    match /admin/{document=**} {
      allow read, write: if hasRole('admin');
    }
  }
}
```

---

## ขั้นตอนที่ 2002: Firestore Service Layer

```dart
// lib/firebase/firestore_service.dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';

class Post {
  Post({
    required this.id,
    required this.title,
    required this.content,
    required this.authorId,
    required this.authorName,
    required this.createdAt,
    required this.visibility,
    this.likeCount = 0,
    this.tags = const [],
  });

  final String id;
  final String title;
  final String content;
  final String authorId;
  final String authorName;
  final DateTime createdAt;
  final String visibility;
  final int likeCount;
  final List<String> tags;

  factory Post.fromDoc(DocumentSnapshot doc) {
    final d = doc.data() as Map<String, dynamic>;
    return Post(
      id: doc.id,
      title: d['title'] as String? ?? '',
      content: d['content'] as String? ?? '',
      authorId: d['authorId'] as String? ?? '',
      authorName: d['authorName'] as String? ?? '',
      createdAt: (d['createdAt'] as Timestamp?)?.toDate() ?? DateTime.now(),
      visibility: d['visibility'] as String? ?? 'public',
      likeCount: d['likeCount'] as int? ?? 0,
      tags: List<String>.from(d['tags'] ?? []),
    );
  }

  Map<String, dynamic> toMap() => {
        'title': title,
        'content': content,
        'authorId': authorId,
        'authorName': authorName,
        'createdAt': Timestamp.fromDate(createdAt),
        'visibility': visibility,
        'likeCount': likeCount,
        'tags': tags,
      };
}

class FirestoreService {
  FirestoreService()
      : _db = FirebaseFirestore.instance,
        _auth = FirebaseAuth.instance;

  final FirebaseFirestore _db;
  final FirebaseAuth _auth;

  // ─── Real-time Stream ───────────────────────────────────────────────────────

  /// Stream ของ posts ทั้งหมดที่ visibility = public, เรียงตาม createdAt
  Stream<List<Post>> watchPublicPosts() {
    return _db
        .collection('posts')
        .where('visibility', isEqualTo: 'public')
        .orderBy('createdAt', descending: true)
        .limit(50)
        .snapshots()
        .map((snap) => snap.docs.map(Post.fromDoc).toList());
  }

  /// Stream ของ posts ของ user ปัจจุบัน
  Stream<List<Post>> watchMyPosts() {
    final uid = _auth.currentUser?.uid;
    if (uid == null) return const Stream.empty();
    return _db
        .collection('posts')
        .where('authorId', isEqualTo: uid)
        .orderBy('createdAt', descending: true)
        .snapshots()
        .map((snap) => snap.docs.map(Post.fromDoc).toList());
  }

  // ─── CRUD ───────────────────────────────────────────────────────────────────

  Future<String> createPost({
    required String title,
    required String content,
    String visibility = 'public',
    List<String> tags = const [],
  }) async {
    final user = _auth.currentUser!;
    final ref = _db.collection('posts').doc();
    await ref.set(Post(
      id: ref.id,
      title: title,
      content: content,
      authorId: user.uid,
      authorName: user.displayName ?? 'Anonymous',
      createdAt: DateTime.now(),
      visibility: visibility,
      tags: tags,
    ).toMap());
    return ref.id;
  }

  Future<void> updatePost(String postId, {String? title, String? content}) async {
    final updates = <String, dynamic>{};
    if (title != null) updates['title'] = title;
    if (content != null) updates['content'] = content;
    if (updates.isEmpty) return;
    updates['updatedAt'] = FieldValue.serverTimestamp();
    await _db.collection('posts').doc(postId).update(updates);
  }

  Future<void> deletePost(String postId) async {
    await _db.collection('posts').doc(postId).delete();
  }

  // ─── Transactions ───────────────────────────────────────────────────────────

  /// Like a post — atomic increment
  Future<void> likePost(String postId) async {
    final uid = _auth.currentUser?.uid;
    if (uid == null) throw Exception('Not authenticated');

    await _db.runTransaction((tx) async {
      final postRef = _db.collection('posts').doc(postId);
      final likeRef = _db.collection('posts').doc(postId).collection('likes').doc(uid);

      final postSnap = await tx.get(postRef);
      final likeSnap = await tx.get(likeRef);

      if (!postSnap.exists) throw Exception('Post not found');

      if (likeSnap.exists) {
        // Unlike
        tx.delete(likeRef);
        tx.update(postRef, {'likeCount': FieldValue.increment(-1)});
      } else {
        // Like
        tx.set(likeRef, {'userId': uid, 'createdAt': FieldValue.serverTimestamp()});
        tx.update(postRef, {'likeCount': FieldValue.increment(1)});
      }
    });
  }

  // ─── Batch Writes ───────────────────────────────────────────────────────────

  /// สร้าง posts หลายอันพร้อมกันใน batch
  Future<void> createPostsBatch(List<Map<String, dynamic>> postsData) async {
    final batch = _db.batch();
    for (final data in postsData) {
      final ref = _db.collection('posts').doc();
      batch.set(ref, {
        ...data,
        'createdAt': FieldValue.serverTimestamp(),
        'likeCount': 0,
      });
    }
    await batch.commit();
  }

  /// ลบหลาย posts พร้อมกัน (batch delete)
  Future<void> deletePostsBatch(List<String> postIds) async {
    // Firestore batch limit = 500 operations
    const chunkSize = 400;
    for (int i = 0; i < postIds.length; i += chunkSize) {
      final chunk = postIds.skip(i).take(chunkSize).toList();
      final batch = _db.batch();
      for (final id in chunk) {
        batch.delete(_db.collection('posts').doc(id));
      }
      await batch.commit();
    }
  }
}
```

---

## ขั้นตอนที่ 2003: Real-time StreamBuilder Widget

```dart
// lib/firebase/realtime_posts_page.dart
import 'package:flutter/material.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:intl/intl.dart';
import 'firestore_service.dart';

class RealtimePostsPage extends StatefulWidget {
  const RealtimePostsPage({super.key});

  @override
  State<RealtimePostsPage> createState() => _RealtimePostsPageState();
}

class _RealtimePostsPageState extends State<RealtimePostsPage> {
  final _service = FirestoreService();
  bool _showMine = false;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Live Posts'),
        actions: [
          Switch(
            value: _showMine,
            onChanged: (v) => setState(() => _showMine = v),
          ),
          const SizedBox(width: 8),
          Text(_showMine ? 'Mine' : 'All', style: const TextStyle(fontSize: 12)),
          const SizedBox(width: 8),
        ],
      ),
      body: StreamBuilder<List<Post>>(
        stream: _showMine ? _service.watchMyPosts() : _service.watchPublicPosts(),
        builder: (ctx, snap) {
          if (snap.connectionState == ConnectionState.waiting) {
            return const Center(child: CircularProgressIndicator());
          }
          if (snap.hasError) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Icon(Icons.error_outline, size: 48, color: Colors.red),
                  const SizedBox(height: 8),
                  Text('Error: ${snap.error}', textAlign: TextAlign.center),
                ],
              ),
            );
          }
          final posts = snap.data ?? [];
          if (posts.isEmpty) {
            return const Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Icon(Icons.article_outlined, size: 64, color: Colors.grey),
                  SizedBox(height: 16),
                  Text('No posts yet', style: TextStyle(color: Colors.grey)),
                ],
              ),
            );
          }
          return ListView.separated(
            padding: const EdgeInsets.all(16),
            itemCount: posts.length,
            separatorBuilder: (_, __) => const SizedBox(height: 12),
            itemBuilder: (ctx, i) => _PostCard(
              post: posts[i],
              onLike: () => _service.likePost(posts[i].id),
              onDelete: () => _service.deletePost(posts[i].id),
            ),
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _showCreateDialog(context),
        child: const Icon(Icons.add),
      ),
    );
  }

  void _showCreateDialog(BuildContext context) {
    final titleCtrl = TextEditingController();
    final contentCtrl = TextEditingController();
    showDialog(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('New Post'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(
              controller: titleCtrl,
              decoration: const InputDecoration(labelText: 'Title', border: OutlineInputBorder()),
            ),
            const SizedBox(height: 12),
            TextField(
              controller: contentCtrl,
              maxLines: 3,
              decoration: const InputDecoration(labelText: 'Content', border: OutlineInputBorder()),
            ),
          ],
        ),
        actions: [
          TextButton(onPressed: () => Navigator.pop(ctx), child: const Text('Cancel')),
          ElevatedButton(
            onPressed: () async {
              if (titleCtrl.text.isNotEmpty) {
                await _service.createPost(
                  title: titleCtrl.text,
                  content: contentCtrl.text,
                );
                if (ctx.mounted) Navigator.pop(ctx);
              }
            },
            child: const Text('Post'),
          ),
        ],
      ),
    );
  }
}

class _PostCard extends StatelessWidget {
  const _PostCard({required this.post, required this.onLike, required this.onDelete});
  final Post post;
  final VoidCallback onLike;
  final VoidCallback onDelete;

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          ListTile(
            leading: CircleAvatar(
              child: Text(post.authorName.isNotEmpty ? post.authorName[0] : '?'),
            ),
            title: Text(post.title, style: const TextStyle(fontWeight: FontWeight.w600)),
            subtitle: Text(
              DateFormat('dd MMM yyyy, HH:mm').format(post.createdAt),
              style: const TextStyle(fontSize: 12),
            ),
            trailing: IconButton(
              icon: const Icon(Icons.delete_outline),
              onPressed: onDelete,
            ),
          ),
          if (post.content.isNotEmpty)
            Padding(
              padding: const EdgeInsets.fromLTRB(16, 0, 16, 8),
              child: Text(post.content),
            ),
          if (post.tags.isNotEmpty)
            Padding(
              padding: const EdgeInsets.fromLTRB(16, 0, 16, 8),
              child: Wrap(
                spacing: 6,
                children: post.tags
                    .map((t) => Chip(label: Text(t), materialTapTargetSize: MaterialTapTargetSize.shrinkWrap))
                    .toList(),
              ),
            ),
          Row(
            children: [
              TextButton.icon(
                onPressed: onLike,
                icon: const Icon(Icons.favorite_outline, size: 18),
                label: Text('${post.likeCount}'),
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

## ขั้นตอนที่ 2004: Offline Persistence

```dart
// lib/firebase/firebase_init.dart
import 'package:firebase_core/firebase_core.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:flutter/foundation.dart';

Future<void> initFirebase() async {
  await Firebase.initializeApp();
  _configureFirestore();
}

void _configureFirestore() {
  final db = FirebaseFirestore.instance;

  // Enable offline persistence
  // บน Mobile (iOS/Android) — enabled by default
  // บน Web ต้องเปิดเองอย่างชัดเจน:
  if (kIsWeb) {
    db.enablePersistence(const PersistenceSettings(synchronizeTabs: true));
  }

  // กำหนด cache size (bytes) — default 100MB
  db.settings = const Settings(
    persistenceEnabled: true,
    cacheSizeBytes: Settings.CACHE_SIZE_UNLIMITED, // ไม่จำกัด
  );
}

/// ตรวจสอบว่า document มาจาก cache หรือ server
void checkDocumentSource(DocumentSnapshot doc) {
  if (doc.metadata.isFromCache) {
    print('Data came from CACHE (offline)');
  } else {
    print('Data came from SERVER');
  }
  if (doc.metadata.hasPendingWrites) {
    print('Document has pending writes (not yet synced)');
  }
}

/// ดึงข้อมูลจาก cache ก่อน แล้วค่อย sync กับ server
Future<List<Post>> getPostsWithCacheFirst(FirebaseFirestore db) async {
  try {
    // พยายามดึงจาก cache ก่อน
    final cacheSnap = await db
        .collection('posts')
        .orderBy('createdAt', descending: true)
        .get(const GetOptions(source: Source.cache));
    if (cacheSnap.docs.isNotEmpty) {
      print('Got ${cacheSnap.docs.length} posts from cache');
      return cacheSnap.docs.map(Post.fromDoc).toList();
    }
  } catch (_) {
    // cache ว่าง — ไป server
  }
  // ถ้า cache ว่าง ดึงจาก server
  final serverSnap = await db
      .collection('posts')
      .orderBy('createdAt', descending: true)
      .get(const GetOptions(source: Source.server));
  return serverSnap.docs.map(Post.fromDoc).toList();
}

// Import ที่ต้องการ (จากไฟล์ firestore_service.dart)
import 'firestore_service.dart';
```

---

## ขั้นตอนที่ 2005: Pagination ด้วย startAfter

```dart
// lib/firebase/paginated_posts.dart
import 'package:flutter/material.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'firestore_service.dart';

class PaginatedPostsNotifier extends ChangeNotifier {
  PaginatedPostsNotifier() : _db = FirebaseFirestore.instance;

  final FirebaseFirestore _db;
  final List<Post> _posts = [];
  DocumentSnapshot? _lastDoc;
  bool _hasMore = true;
  bool _isLoading = false;
  Object? _error;

  List<Post> get posts => List.unmodifiable(_posts);
  bool get hasMore => _hasMore;
  bool get isLoading => _isLoading;
  Object? get error => _error;

  static const _pageSize = 10;

  Future<void> fetchFirstPage() async {
    _posts.clear();
    _lastDoc = null;
    _hasMore = true;
    _error = null;
    await _fetchNextPage();
  }

  Future<void> fetchNextPage() async {
    if (_isLoading || !_hasMore) return;
    await _fetchNextPage();
  }

  Future<void> _fetchNextPage() async {
    _isLoading = true;
    notifyListeners();
    try {
      Query<Map<String, dynamic>> query = _db
          .collection('posts')
          .where('visibility', isEqualTo: 'public')
          .orderBy('createdAt', descending: true)
          .limit(_pageSize);

      if (_lastDoc != null) {
        query = query.startAfterDocument(_lastDoc!);
      }

      final snap = await query.get();
      if (snap.docs.isEmpty || snap.docs.length < _pageSize) {
        _hasMore = false;
      }
      if (snap.docs.isNotEmpty) {
        _lastDoc = snap.docs.last;
        _posts.addAll(snap.docs.map(Post.fromDoc));
      }
    } catch (e) {
      _error = e;
    } finally {
      _isLoading = false;
      notifyListeners();
    }
  }
}

class PaginatedPostsPage extends StatefulWidget {
  const PaginatedPostsPage({super.key});

  @override
  State<PaginatedPostsPage> createState() => _PaginatedPostsPageState();
}

class _PaginatedPostsPageState extends State<PaginatedPostsPage> {
  late final PaginatedPostsNotifier _notifier;
  final _scrollCtrl = ScrollController();

  @override
  void initState() {
    super.initState();
    _notifier = PaginatedPostsNotifier();
    _notifier.fetchFirstPage();
    _scrollCtrl.addListener(_onScroll);
  }

  void _onScroll() {
    if (_scrollCtrl.position.pixels >= _scrollCtrl.position.maxScrollExtent - 200) {
      _notifier.fetchNextPage();
    }
  }

  @override
  void dispose() {
    _notifier.dispose();
    _scrollCtrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paginated Posts')),
      body: AnimatedBuilder(
        animation: _notifier,
        builder: (ctx, _) {
          if (_notifier.posts.isEmpty && _notifier.isLoading) {
            return const Center(child: CircularProgressIndicator());
          }
          if (_notifier.error != null && _notifier.posts.isEmpty) {
            return Center(child: Text('Error: ${_notifier.error}'));
          }
          return RefreshIndicator(
            onRefresh: _notifier.fetchFirstPage,
            child: ListView.builder(
              controller: _scrollCtrl,
              padding: const EdgeInsets.all(16),
              itemCount: _notifier.posts.length + (_notifier.hasMore ? 1 : 0),
              itemBuilder: (ctx, i) {
                if (i == _notifier.posts.length) {
                  return Padding(
                    padding: const EdgeInsets.all(16),
                    child: Center(
                      child: _notifier.isLoading
                          ? const CircularProgressIndicator()
                          : TextButton(
                              onPressed: _notifier.fetchNextPage,
                              child: const Text('Load more'),
                            ),
                    ),
                  );
                }
                final post = _notifier.posts[i];
                return Card(
                  margin: const EdgeInsets.only(bottom: 12),
                  child: ListTile(
                    title: Text(post.title, style: const TextStyle(fontWeight: FontWeight.w600)),
                    subtitle: Text(post.content, maxLines: 2, overflow: TextOverflow.ellipsis),
                    trailing: Text('❤️ ${post.likeCount}'),
                  ),
                );
              },
            ),
          );
        },
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2006: Firebase Storage Upload กับ Progress

```dart
// lib/firebase/storage_upload_page.dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:firebase_storage/firebase_storage.dart';
import 'package:image_picker/image_picker.dart';
import 'package:path/path.dart' as p;

class UploadTask {
  UploadTask({
    required this.file,
    required this.uploadTask,
  });
  final File file;
  final UploadTask uploadTask;
  String? downloadUrl;
  Object? error;
}

class StorageUploadPage extends StatefulWidget {
  const StorageUploadPage({super.key});

  @override
  State<StorageUploadPage> createState() => _StorageUploadPageState();
}

class _StorageUploadPageState extends State<StorageUploadPage> {
  final _storage = FirebaseStorage.instance;
  final _picker = ImagePicker();
  final List<_UploadEntry> _uploads = [];

  Future<void> _pickAndUpload() async {
    final files = await _picker.pickMultiImage(imageQuality: 85);
    if (files.isEmpty) return;

    for (final xFile in files) {
      final file = File(xFile.path);
      final fileName = '${DateTime.now().millisecondsSinceEpoch}_${p.basename(xFile.path)}';
      final ref = _storage.ref().child('uploads/$fileName');

      final task = ref.putFile(
        file,
        SettableMetadata(
          contentType: 'image/jpeg',
          customMetadata: {'uploadedBy': 'user', 'originalName': xFile.name},
        ),
      );

      final entry = _UploadEntry(
        fileName: xFile.name,
        file: file,
        task: task,
      );

      setState(() => _uploads.insert(0, entry));

      task.snapshotEvents.listen((snap) {
        setState(() => entry.snapshot = snap);
      });

      try {
        await task.whenComplete(() async {
          final url = await ref.getDownloadURL();
          setState(() => entry.downloadUrl = url);
        });
      } catch (e) {
        setState(() => entry.error = e);
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Firebase Storage Upload'),
        actions: [
          IconButton(icon: const Icon(Icons.add_photo_alternate), onPressed: _pickAndUpload),
        ],
      ),
      body: _uploads.isEmpty
          ? Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Icon(Icons.cloud_upload_outlined, size: 80, color: Colors.grey),
                  const SizedBox(height: 16),
                  const Text('No uploads yet'),
                  const SizedBox(height: 16),
                  ElevatedButton.icon(
                    onPressed: _pickAndUpload,
                    icon: const Icon(Icons.upload),
                    label: const Text('Select Images'),
                  ),
                ],
              ),
            )
          : ListView.builder(
              padding: const EdgeInsets.all(16),
              itemCount: _uploads.length,
              itemBuilder: (ctx, i) => _UploadCard(entry: _uploads[i]),
            ),
    );
  }
}

class _UploadEntry {
  _UploadEntry({required this.fileName, required this.file, required this.task});

  final String fileName;
  final File file;
  final firebase_storage.UploadTask task;
  firebase_storage.TaskSnapshot? snapshot;
  String? downloadUrl;
  Object? error;

  double get progress {
    final s = snapshot;
    if (s == null) return 0;
    return s.bytesTransferred / s.totalBytes;
  }

  bool get isDone => downloadUrl != null;
  bool get hasError => error != null;
  bool get isRunning => !isDone && !hasError;
}

// pubspec.yaml ต้องมี:
//   firebase_storage: ^12.3.0

// ignore: library_prefixes
import 'package:firebase_storage/firebase_storage.dart' as firebase_storage;

class _UploadCard extends StatefulWidget {
  const _UploadCard({required this.entry});
  final _UploadEntry entry;

  @override
  State<_UploadCard> createState() => _UploadCardState();
}

class _UploadCardState extends State<_UploadCard> {
  @override
  void initState() {
    super.initState();
    widget.entry.task.snapshotEvents.listen((_) {
      if (mounted) setState(() {});
    });
  }

  @override
  Widget build(BuildContext context) {
    final e = widget.entry;
    return Card(
      margin: const EdgeInsets.only(bottom: 12),
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                ClipRRect(
                  borderRadius: BorderRadius.circular(8),
                  child: Image.file(e.file, width: 60, height: 60, fit: BoxFit.cover),
                ),
                const SizedBox(width: 12),
                Expanded(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(e.fileName, style: const TextStyle(fontWeight: FontWeight.w600),
                          maxLines: 1, overflow: TextOverflow.ellipsis),
                      const SizedBox(height: 4),
                      if (e.hasError)
                        Text('Error: ${e.error}', style: const TextStyle(color: Colors.red, fontSize: 12))
                      else if (e.isDone)
                        const Text('Upload complete ✓', style: TextStyle(color: Colors.green, fontSize: 12))
                      else
                        Text('${(e.progress * 100).round()}%', style: const TextStyle(fontSize: 12)),
                    ],
                  ),
                ),
                if (e.isRunning)
                  IconButton(
                    icon: const Icon(Icons.cancel_outlined),
                    onPressed: () => e.task.cancel(),
                  ),
              ],
            ),
            if (e.isRunning) ...[
              const SizedBox(height: 8),
              LinearProgressIndicator(
                value: e.progress,
                backgroundColor: Colors.grey.shade200,
              ),
              const SizedBox(height: 4),
              if (e.snapshot != null)
                Text(
                  '${_formatBytes(e.snapshot!.bytesTransferred)} / ${_formatBytes(e.snapshot!.totalBytes)}',
                  style: TextStyle(fontSize: 11, color: Colors.grey.shade600),
                ),
            ],
            if (e.isDone && e.downloadUrl != null) ...[
              const SizedBox(height: 8),
              SelectableText(
                e.downloadUrl!,
                style: TextStyle(fontSize: 11, color: Colors.blue.shade700),
                maxLines: 2,
              ),
            ],
          ],
        ),
      ),
    );
  }

  String _formatBytes(int bytes) {
    if (bytes < 1024) return '$bytes B';
    if (bytes < 1024 * 1024) return '${(bytes / 1024).toStringAsFixed(1)} KB';
    return '${(bytes / (1024 * 1024)).toStringAsFixed(1)} MB';
  }
}
```

---

## ขั้นตอนที่ 2007: Compound Queries & Indexes

```dart
// lib/firebase/compound_queries.dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'firestore_service.dart';

class CompoundQueryService {
  final _db = FirebaseFirestore.instance;

  // Query ที่ต้องการ composite index:
  // Collection: posts
  // Fields: tags (array-contains), createdAt (descending)
  // สร้าง index ผ่าน Firebase console หรือ firebase.indexes.json

  /// Posts ที่มี tag หนึ่งๆ เรียงตามเวลา
  Future<List<Post>> getPostsByTag(String tag, {int limit = 20}) async {
    final snap = await _db
        .collection('posts')
        .where('tags', arrayContains: tag)
        .where('visibility', isEqualTo: 'public')
        .orderBy('createdAt', descending: true)
        .limit(limit)
        .get();
    return snap.docs.map(Post.fromDoc).toList();
  }

  /// Posts ที่มีหลาย tags (array-contains-any — max 10 values)
  Future<List<Post>> getPostsByAnyTag(List<String> tags) async {
    if (tags.isEmpty) return [];
    final snap = await _db
        .collection('posts')
        .where('tags', arrayContainsAny: tags.take(10).toList())
        .orderBy('createdAt', descending: true)
        .limit(20)
        .get();
    return snap.docs.map(Post.fromDoc).toList();
  }

  /// Posts ที่มี likeCount มากกว่า threshold
  Future<List<Post>> getPopularPosts({int minLikes = 10}) async {
    final snap = await _db
        .collection('posts')
        .where('visibility', isEqualTo: 'public')
        .where('likeCount', isGreaterThanOrEqualTo: minLikes)
        .orderBy('likeCount', descending: true)
        .limit(20)
        .get();
    return snap.docs.map(Post.fromDoc).toList();
  }

  /// Posts ในช่วงเวลา
  Future<List<Post>> getPostsInDateRange(DateTime start, DateTime end) async {
    final snap = await _db
        .collection('posts')
        .where('createdAt', isGreaterThanOrEqualTo: Timestamp.fromDate(start))
        .where('createdAt', isLessThanOrEqualTo: Timestamp.fromDate(end))
        .orderBy('createdAt', descending: true)
        .get();
    return snap.docs.map(Post.fromDoc).toList();
  }

  /// Full-text search แบบ prefix (ต้องเก็บ searchTokens ในเอกสาร)
  Future<List<Post>> searchByTitle(String query) async {
    if (query.isEmpty) return [];
    final lower = query.toLowerCase();
    final snap = await _db
        .collection('posts')
        .where('searchTokens', arrayContains: lower)
        .limit(20)
        .get();
    return snap.docs.map(Post.fromDoc).toList();
  }
}

// firebase.indexes.json
// {
//   "indexes": [
//     {
//       "collectionGroup": "posts",
//       "queryScope": "COLLECTION",
//       "fields": [
//         {"fieldPath": "tags", "arrayConfig": "CONTAINS"},
//         {"fieldPath": "visibility", "order": "ASCENDING"},
//         {"fieldPath": "createdAt", "order": "DESCENDING"}
//       ]
//     }
//   ]
// }
```

---

## ขั้นตอนที่ 2008: Full Firestore App

```dart
// lib/main_firestore_demo.dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'firebase/firebase_init.dart';
import 'firebase/realtime_posts_page.dart';
import 'firebase/paginated_posts.dart';
import 'firebase/storage_upload_page.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await initFirebase();
  runApp(const FirestoreDemoApp());
}

class FirestoreDemoApp extends StatelessWidget {
  const FirestoreDemoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Firestore Demo',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(colorSchemeSeed: Colors.orange, useMaterial3: true),
      home: const FirestoreDemoHome(),
    );
  }
}

class FirestoreDemoHome extends StatelessWidget {
  const FirestoreDemoHome({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Part 53 — Firestore Advanced')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          _DemoTile(
            title: 'Real-time Posts',
            icon: Icons.stream,
            color: Colors.orange,
            builder: () => const RealtimePostsPage(),
          ),
          const SizedBox(height: 8),
          _DemoTile(
            title: 'Paginated Posts',
            icon: Icons.pages,
            color: Colors.blue,
            builder: () => const PaginatedPostsPage(),
          ),
          const SizedBox(height: 8),
          _DemoTile(
            title: 'Storage Upload',
            icon: Icons.cloud_upload,
            color: Colors.green,
            builder: () => const StorageUploadPage(),
          ),
        ],
      ),
    );
  }
}

class _DemoTile extends StatelessWidget {
  const _DemoTile({
    required this.title,
    required this.icon,
    required this.color,
    required this.builder,
  });
  final String title;
  final IconData icon;
  final Color color;
  final Widget Function() builder;

  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: CircleAvatar(
        backgroundColor: color.withOpacity(0.15),
        child: Icon(icon, color: color),
      ),
      title: Text(title),
      trailing: const Icon(Icons.chevron_right),
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
        side: BorderSide(color: Colors.grey.shade200),
      ),
      onTap: () => Navigator.push(context, MaterialPageRoute(builder: (_) => builder())),
    );
  }
}
```

---

**← [Part 52](part-52-animations-advanced.md)**
**ต่อไป: [Part 54 →](part-54-push-notifications-advanced.md)**
