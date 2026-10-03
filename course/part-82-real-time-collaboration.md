# Part 82: Real-Time Collaboration
## ขั้นตอนที่ 3161-3200

## 🎯 เป้าหมายของ Part นี้
- เข้าใจ Operational Transform basics และ CRDT
- สร้าง Collaborative text editing อย่างง่าย
- แสดง Presence indicators (ผู้ที่กำลัง online/editing)
- จัดการ Conflict-free editing ด้วย Firestore
- แชร์ cursor position แบบ real-time

---

## ขั้นตอนที่ 3161: ติดตั้งและตั้งค่า Firebase + Dependencies

```yaml
# pubspec.yaml
name: collab_editor
description: Real-time collaborative text editor

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: ">=3.10.0"

dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^2.24.2
  cloud_firestore: ^4.14.0
  firebase_auth: ^4.16.0
  uuid: ^4.3.3
  intl: ^0.19.0
  collection: ^1.18.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
```

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'screens/document_list_screen.dart';
import 'screens/auth_screen.dart';
import 'services/auth_service.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(const CollabEditorApp());
}

class CollabEditorApp extends StatelessWidget {
  const CollabEditorApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Collab Editor',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      home: StreamBuilder(
        stream: AuthService.instance.authStateChanges,
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return const Scaffold(
              body: Center(child: CircularProgressIndicator()),
            );
          }
          if (snapshot.hasData) {
            return const DocumentListScreen();
          }
          return const AuthScreen();
        },
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3162: CRDT (Conflict-free Replicated Data Type) Implementation

```dart
// lib/crdt/crdt_text.dart
import 'package:uuid/uuid.dart';

/// A simple CRDT character that can be inserted/deleted without conflicts
class CrdtChar {
  final String id;        // Unique ID for this character
  final String char;      // The character itself
  final String authorId;  // Who inserted this
  final int timestamp;    // Logical timestamp (Lamport clock)
  bool isDeleted;         // Soft delete (tombstone)

  CrdtChar({
    required this.id,
    required this.char,
    required this.authorId,
    required this.timestamp,
    this.isDeleted = false,
  });

  Map<String, dynamic> toMap() => {
    'id': id,
    'char': char,
    'authorId': authorId,
    'timestamp': timestamp,
    'isDeleted': isDeleted,
  };

  factory CrdtChar.fromMap(Map<String, dynamic> map) => CrdtChar(
    id: map['id'] as String,
    char: map['char'] as String,
    authorId: map['authorId'] as String,
    timestamp: map['timestamp'] as int,
    isDeleted: map['isDeleted'] as bool? ?? false,
  );

  @override
  String toString() => 'CrdtChar($id: "$char", deleted: $isDeleted)';
}

/// Operation types for the CRDT
enum OperationType { insert, delete }

class CrdtOperation {
  final OperationType type;
  final CrdtChar character;
  final String? afterId;  // Insert after this character ID (null = beginning)

  CrdtOperation({
    required this.type,
    required this.character,
    this.afterId,
  });

  Map<String, dynamic> toMap() => {
    'type': type.name,
    'character': character.toMap(),
    'afterId': afterId,
  };

  factory CrdtOperation.fromMap(Map<String, dynamic> map) => CrdtOperation(
    type: OperationType.values.firstWhere((e) => e.name == map['type']),
    character: CrdtChar.fromMap(map['character'] as Map<String, dynamic>),
    afterId: map['afterId'] as String?,
  );
}

/// CRDT-based document that handles concurrent edits
class CrdtDocument {
  final _uuid = const Uuid();
  final List<CrdtChar> _chars = [];
  int _clock = 0;

  /// Get visible text (excluding deleted chars)
  String get text => _chars
      .where((c) => !c.isDeleted)
      .map((c) => c.char)
      .join();

  /// Get all characters including deleted (for CRDT merging)
  List<CrdtChar> get allChars => List.unmodifiable(_chars);

  /// Insert a character at position
  CrdtOperation insertAt(int position, String char, String authorId) {
    _clock++;

    // Find the ID of the character before this position
    final visibleChars = _chars.where((c) => !c.isDeleted).toList();
    final afterId = position > 0 ? visibleChars[position - 1].id : null;

    final newChar = CrdtChar(
      id: _uuid.v4(),
      char: char,
      authorId: authorId,
      timestamp: _clock,
    );

    _applyInsert(newChar, afterId);

    return CrdtOperation(
      type: OperationType.insert,
      character: newChar,
      afterId: afterId,
    );
  }

  /// Delete character at position
  CrdtOperation deleteAt(int position, String authorId) {
    final visibleChars = _chars.where((c) => !c.isDeleted).toList();
    if (position < 0 || position >= visibleChars.length) {
      throw RangeError('Position $position out of range');
    }

    final charToDelete = visibleChars[position];
    charToDelete.isDeleted = true;

    return CrdtOperation(
      type: OperationType.delete,
      character: charToDelete,
    );
  }

  /// Apply a remote operation (for merging)
  void applyOperation(CrdtOperation op) {
    _clock = _clock > op.character.timestamp ? _clock : op.character.timestamp;
    _clock++;

    if (op.type == OperationType.insert) {
      // Check if we already have this char (idempotent)
      if (_chars.any((c) => c.id == op.character.id)) return;
      _applyInsert(op.character, op.afterId);
    } else if (op.type == OperationType.delete) {
      final idx = _chars.indexWhere((c) => c.id == op.character.id);
      if (idx >= 0) _chars[idx].isDeleted = true;
    }
  }

  void _applyInsert(CrdtChar newChar, String? afterId) {
    if (afterId == null) {
      // Insert at beginning
      int insertIdx = 0;
      // Handle concurrent insertions at same position by timestamp+id ordering
      while (insertIdx < _chars.length &&
          _chars[insertIdx].id != afterId &&
          _shouldInsertAfter(_chars[insertIdx], newChar)) {
        insertIdx++;
      }
      _chars.insert(0, newChar);
      return;
    }

    final afterIdx = _chars.indexWhere((c) => c.id == afterId);
    if (afterIdx < 0) {
      // afterId not found, append at end
      _chars.add(newChar);
      return;
    }

    // Insert after the specified character
    int insertIdx = afterIdx + 1;
    // Resolve concurrent insertions: higher timestamp or lexicographically larger ID wins
    while (insertIdx < _chars.length &&
        _shouldInsertAfter(_chars[insertIdx], newChar)) {
      insertIdx++;
    }

    _chars.insert(insertIdx, newChar);
  }

  bool _shouldInsertAfter(CrdtChar existing, CrdtChar newChar) {
    if (existing.timestamp != newChar.timestamp) {
      return existing.timestamp > newChar.timestamp;
    }
    return existing.id.compareTo(newChar.id) > 0;
  }

  /// Merge another document's characters into this one
  void merge(List<CrdtChar> remoteChars) {
    for (final remoteChar in remoteChars) {
      final existing = _chars.firstWhere(
        (c) => c.id == remoteChar.id,
        orElse: () => CrdtChar(id: '', char: '', authorId: '', timestamp: -1),
      );

      if (existing.id.isEmpty) {
        // New character we don't have
        _chars.add(remoteChar);
      } else {
        // Update deletion state (deletion is permanent)
        if (remoteChar.isDeleted) existing.isDeleted = true;
      }
    }
  }

  /// Initialize from a list of characters
  void loadChars(List<CrdtChar> chars) {
    _chars.clear();
    _chars.addAll(chars);
    if (chars.isNotEmpty) {
      _clock = chars.map((c) => c.timestamp).reduce((a, b) => a > b ? a : b);
    }
  }
}
```

---

## ขั้นตอนที่ 3163: Presence Service (Who's Online)

```dart
// lib/services/presence_service.dart
import 'package:cloud_firestore/cloud_firestore.dart';

class UserPresence {
  final String userId;
  final String displayName;
  final String color;
  final int cursorPosition;
  final int selectionStart;
  final int selectionEnd;
  final DateTime lastSeen;
  final bool isOnline;

  const UserPresence({
    required this.userId,
    required this.displayName,
    required this.color,
    required this.cursorPosition,
    required this.selectionStart,
    required this.selectionEnd,
    required this.lastSeen,
    required this.isOnline,
  });

  Map<String, dynamic> toMap() => {
    'userId': userId,
    'displayName': displayName,
    'color': color,
    'cursorPosition': cursorPosition,
    'selectionStart': selectionStart,
    'selectionEnd': selectionEnd,
    'lastSeen': Timestamp.fromDate(lastSeen),
    'isOnline': isOnline,
  };

  factory UserPresence.fromMap(Map<String, dynamic> map) => UserPresence(
    userId: map['userId'] as String,
    displayName: map['displayName'] as String,
    color: map['color'] as String,
    cursorPosition: map['cursorPosition'] as int? ?? 0,
    selectionStart: map['selectionStart'] as int? ?? 0,
    selectionEnd: map['selectionEnd'] as int? ?? 0,
    lastSeen: (map['lastSeen'] as Timestamp).toDate(),
    isOnline: map['isOnline'] as bool? ?? false,
  );

  UserPresence copyWith({
    int? cursorPosition,
    int? selectionStart,
    int? selectionEnd,
    DateTime? lastSeen,
    bool? isOnline,
  }) {
    return UserPresence(
      userId: userId,
      displayName: displayName,
      color: color,
      cursorPosition: cursorPosition ?? this.cursorPosition,
      selectionStart: selectionStart ?? this.selectionStart,
      selectionEnd: selectionEnd ?? this.selectionEnd,
      lastSeen: lastSeen ?? this.lastSeen,
      isOnline: isOnline ?? this.isOnline,
    );
  }
}

/// Manages user presence in a document
class PresenceService {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  static const List<String> _userColors = [
    '#FF5733', '#33FF57', '#3357FF', '#FF33A8',
    '#A833FF', '#33FFF5', '#FF8C33', '#8CFF33',
  ];

  String _assignColor(int index) => _userColors[index % _userColors.length];

  /// Join a document session
  Future<void> joinDocument({
    required String documentId,
    required String userId,
    required String displayName,
  }) async {
    final presenceRef = _firestore
        .collection('documents')
        .doc(documentId)
        .collection('presence')
        .doc(userId);

    // Count existing users for color assignment
    final snapshot = await _firestore
        .collection('documents')
        .doc(documentId)
        .collection('presence')
        .count()
        .get();

    final colorIndex = snapshot.count ?? 0;

    await presenceRef.set({
      'userId': userId,
      'displayName': displayName,
      'color': _assignColor(colorIndex),
      'cursorPosition': 0,
      'selectionStart': 0,
      'selectionEnd': 0,
      'lastSeen': FieldValue.serverTimestamp(),
      'isOnline': true,
    });
  }

  /// Leave a document session
  Future<void> leaveDocument({
    required String documentId,
    required String userId,
  }) async {
    await _firestore
        .collection('documents')
        .doc(documentId)
        .collection('presence')
        .doc(userId)
        .update({
      'isOnline': false,
      'lastSeen': FieldValue.serverTimestamp(),
    });
  }

  /// Update cursor position
  Future<void> updateCursor({
    required String documentId,
    required String userId,
    required int cursorPosition,
    int selectionStart = 0,
    int selectionEnd = 0,
  }) async {
    await _firestore
        .collection('documents')
        .doc(documentId)
        .collection('presence')
        .doc(userId)
        .update({
      'cursorPosition': cursorPosition,
      'selectionStart': selectionStart,
      'selectionEnd': selectionEnd,
      'lastSeen': FieldValue.serverTimestamp(),
    });
  }

  /// Stream of all online users in a document
  Stream<List<UserPresence>> watchPresence(String documentId) {
    return _firestore
        .collection('documents')
        .doc(documentId)
        .collection('presence')
        .where('isOnline', isEqualTo: true)
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map((doc) => UserPresence.fromMap(doc.data()))
            .toList());
  }
}
```

---

## ขั้นตอนที่ 3164: Collaborative Document Service

```dart
// lib/services/document_service.dart
import 'dart:async';
import 'package:cloud_firestore/cloud_firestore.dart';
import '../crdt/crdt_text.dart';

class CollaborativeDocument {
  final String id;
  final String title;
  final String ownerId;
  final DateTime createdAt;
  final DateTime updatedAt;
  final int version;

  const CollaborativeDocument({
    required this.id,
    required this.title,
    required this.ownerId,
    required this.createdAt,
    required this.updatedAt,
    required this.version,
  });

  factory CollaborativeDocument.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return CollaborativeDocument(
      id: doc.id,
      title: data['title'] as String,
      ownerId: data['ownerId'] as String,
      createdAt: (data['createdAt'] as Timestamp).toDate(),
      updatedAt: (data['updatedAt'] as Timestamp).toDate(),
      version: data['version'] as int? ?? 0,
    );
  }
}

class DocumentService {
  final FirebaseFirestore _firestore = FirebaseFirestore.instance;

  /// Create a new document
  Future<String> createDocument({
    required String title,
    required String ownerId,
  }) async {
    final docRef = await _firestore.collection('documents').add({
      'title': title,
      'ownerId': ownerId,
      'createdAt': FieldValue.serverTimestamp(),
      'updatedAt': FieldValue.serverTimestamp(),
      'version': 0,
      'content': '',
    });

    return docRef.id;
  }

  /// Get documents for a user
  Stream<List<CollaborativeDocument>> getDocuments(String userId) {
    return _firestore
        .collection('documents')
        .where('ownerId', isEqualTo: userId)
        .orderBy('updatedAt', descending: true)
        .snapshots()
        .map((snapshot) => snapshot.docs
            .map(CollaborativeDocument.fromFirestore)
            .toList());
  }

  /// Save a CRDT operation to Firestore
  Future<void> saveOperation({
    required String documentId,
    required CrdtOperation operation,
    required String authorId,
  }) async {
    await _firestore
        .collection('documents')
        .doc(documentId)
        .collection('operations')
        .add({
      ...operation.toMap(),
      'authorId': authorId,
      'timestamp': FieldValue.serverTimestamp(),
    });

    // Update document metadata
    await _firestore.collection('documents').doc(documentId).update({
      'updatedAt': FieldValue.serverTimestamp(),
      'version': FieldValue.increment(1),
    });
  }

  /// Stream of operations for a document
  Stream<List<CrdtOperation>> watchOperations(
    String documentId, {
    int? afterVersion,
  }) {
    Query<Map<String, dynamic>> query = _firestore
        .collection('documents')
        .doc(documentId)
        .collection('operations')
        .orderBy('timestamp');

    return query.snapshots().map((snapshot) => snapshot.docs
        .map((doc) => CrdtOperation.fromMap(doc.data()))
        .toList());
  }

  /// Save full document state (snapshot)
  Future<void> saveSnapshot({
    required String documentId,
    required List<CrdtChar> chars,
  }) async {
    final batch = _firestore.batch();

    final snapshotRef = _firestore
        .collection('documents')
        .doc(documentId)
        .collection('snapshots')
        .doc('latest');

    batch.set(snapshotRef, {
      'chars': chars.map((c) => c.toMap()).toList(),
      'savedAt': FieldValue.serverTimestamp(),
    });

    await batch.commit();
  }

  /// Load latest snapshot
  Future<List<CrdtChar>?> loadSnapshot(String documentId) async {
    final doc = await _firestore
        .collection('documents')
        .doc(documentId)
        .collection('snapshots')
        .doc('latest')
        .get();

    if (!doc.exists) return null;

    final data = doc.data()!;
    final charsList = data['chars'] as List<dynamic>;
    return charsList
        .map((c) => CrdtChar.fromMap(c as Map<String, dynamic>))
        .toList();
  }

  /// Delete a document
  Future<void> deleteDocument(String documentId) async {
    await _firestore.collection('documents').doc(documentId).delete();
  }
}
```

---

## ขั้นตอนที่ 3165: Collaborative Editor Screen

```dart
// lib/screens/collaborative_editor_screen.dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import '../crdt/crdt_text.dart';
import '../services/document_service.dart';
import '../services/presence_service.dart';
import '../widgets/presence_indicator.dart';
import '../widgets/cursor_overlay.dart';

class CollaborativeEditorScreen extends StatefulWidget {
  final String documentId;
  final String title;

  const CollaborativeEditorScreen({
    super.key,
    required this.documentId,
    required this.title,
  });

  @override
  State<CollaborativeEditorScreen> createState() =>
      _CollaborativeEditorScreenState();
}

class _CollaborativeEditorScreenState extends State<CollaborativeEditorScreen>
    with WidgetsBindingObserver {
  final _textController = TextEditingController();
  final _focusNode = FocusNode();
  final _documentService = DocumentService();
  final _presenceService = PresenceService();
  final _crdtDoc = CrdtDocument();

  late final StreamSubscription<List<CrdtOperation>> _operationSub;
  late final StreamSubscription<List<UserPresence>> _presenceSub;

  String get _userId => FirebaseAuth.instance.currentUser!.uid;
  String get _displayName =>
      FirebaseAuth.instance.currentUser!.displayName ?? 'Anonymous';

  List<UserPresence> _otherUsers = [];
  bool _isLoading = true;
  bool _isApplyingRemote = false;

  Timer? _presenceTimer;
  Timer? _snapshotTimer;

  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
    _initialize();
  }

  Future<void> _initialize() async {
    // Load existing document state
    final snapshot = await _documentService.loadSnapshot(widget.documentId);
    if (snapshot != null) {
      _crdtDoc.loadChars(snapshot);
      _textController.text = _crdtDoc.text;
    }

    // Join presence
    await _presenceService.joinDocument(
      documentId: widget.documentId,
      userId: _userId,
      displayName: _displayName,
    );

    // Subscribe to remote operations
    _operationSub = _documentService
        .watchOperations(widget.documentId)
        .listen(_handleRemoteOperations);

    // Subscribe to presence updates
    _presenceSub = _presenceService
        .watchPresence(widget.documentId)
        .listen((users) {
      setState(() {
        _otherUsers = users.where((u) => u.userId != _userId).toList();
      });
    });

    // Setup presence heartbeat
    _presenceTimer = Timer.periodic(const Duration(seconds: 30), (_) {
      _presenceService.updateCursor(
        documentId: widget.documentId,
        userId: _userId,
        cursorPosition: _textController.selection.baseOffset,
      );
    });

    // Auto-save snapshot periodically
    _snapshotTimer = Timer.periodic(const Duration(minutes: 5), (_) {
      _saveSnapshot();
    });

    // Listen for text changes
    _textController.addListener(_handleTextChange);

    setState(() => _isLoading = false);
  }

  void _handleRemoteOperations(List<CrdtOperation> operations) {
    if (_isApplyingRemote) return;

    _isApplyingRemote = true;

    for (final op in operations) {
      if (op.character.authorId == _userId) continue;
      _crdtDoc.applyOperation(op);
    }

    final newText = _crdtDoc.text;
    if (newText != _textController.text) {
      final currentOffset = _textController.selection.baseOffset;
      _textController.value = TextEditingValue(
        text: newText,
        selection: TextSelection.collapsed(
          offset: currentOffset.clamp(0, newText.length),
        ),
      );
    }

    _isApplyingRemote = false;
  }

  void _handleTextChange() {
    if (_isApplyingRemote) return;

    // Update cursor presence
    final cursorPos = _textController.selection.baseOffset;
    if (cursorPos >= 0) {
      _presenceService.updateCursor(
        documentId: widget.documentId,
        userId: _userId,
        cursorPosition: cursorPos,
        selectionStart: _textController.selection.start,
        selectionEnd: _textController.selection.end,
      );
    }
  }

  Future<void> _onTextChanged(String newText) async {
    if (_isApplyingRemote) return;

    final oldText = _crdtDoc.text;
    if (newText == oldText) return;

    // Simple diff: find changes
    final ops = _computeDiff(oldText, newText);

    for (final op in ops) {
      await _documentService.saveOperation(
        documentId: widget.documentId,
        operation: op,
        authorId: _userId,
      );
    }
  }

  List<CrdtOperation> _computeDiff(String oldText, String newText) {
    final ops = <CrdtOperation>[];

    // Find common prefix length
    int prefixLen = 0;
    while (prefixLen < oldText.length &&
        prefixLen < newText.length &&
        oldText[prefixLen] == newText[prefixLen]) {
      prefixLen++;
    }

    // Find common suffix length
    int suffixLen = 0;
    while (suffixLen < oldText.length - prefixLen &&
        suffixLen < newText.length - prefixLen &&
        oldText[oldText.length - 1 - suffixLen] ==
            newText[newText.length - 1 - suffixLen]) {
      suffixLen++;
    }

    // Delete removed characters
    final deleteCount = oldText.length - prefixLen - suffixLen;
    for (int i = 0; i < deleteCount; i++) {
      try {
        final op = _crdtDoc.deleteAt(prefixLen, _userId);
        ops.add(op);
      } catch (_) {}
    }

    // Insert new characters
    final insertStr = newText.substring(
        prefixLen, newText.length - suffixLen);
    for (int i = 0; i < insertStr.length; i++) {
      final op = _crdtDoc.insertAt(prefixLen + i, insertStr[i], _userId);
      ops.add(op);
    }

    return ops;
  }

  Future<void> _saveSnapshot() async {
    await _documentService.saveSnapshot(
      documentId: widget.documentId,
      chars: _crdtDoc.allChars.toList(),
    );
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.paused ||
        state == AppLifecycleState.detached) {
      _presenceService.leaveDocument(
        documentId: widget.documentId,
        userId: _userId,
      );
    } else if (state == AppLifecycleState.resumed) {
      _presenceService.joinDocument(
        documentId: widget.documentId,
        userId: _userId,
        displayName: _displayName,
      );
    }
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    _operationSub.cancel();
    _presenceSub.cancel();
    _presenceTimer?.cancel();
    _snapshotTimer?.cancel();
    _presenceService.leaveDocument(
      documentId: widget.documentId,
      userId: _userId,
    );
    _saveSnapshot();
    _textController.dispose();
    _focusNode.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return const Scaffold(body: Center(child: CircularProgressIndicator()));
    }

    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
        actions: [
          // Presence indicators in app bar
          PresenceIndicatorBar(users: _otherUsers),
          IconButton(
            icon: const Icon(Icons.save),
            onPressed: _saveSnapshot,
            tooltip: 'Save',
          ),
        ],
      ),
      body: Column(
        children: [
          // Online users banner
          if (_otherUsers.isNotEmpty)
            _OnlineUsersBanner(users: _otherUsers),

          // Editor
          Expanded(
            child: Stack(
              children: [
                Padding(
                  padding: const EdgeInsets.all(16),
                  child: TextField(
                    controller: _textController,
                    focusNode: _focusNode,
                    maxLines: null,
                    expands: true,
                    keyboardType: TextInputType.multiline,
                    textAlignVertical: TextAlignVertical.top,
                    onChanged: _onTextChanged,
                    style: const TextStyle(
                      fontSize: 16,
                      height: 1.6,
                      fontFamily: 'monospace',
                    ),
                    decoration: const InputDecoration(
                      border: InputBorder.none,
                      hintText: 'Start typing...',
                    ),
                  ),
                ),

                // Cursor overlays for other users
                CursorOverlay(
                  users: _otherUsers,
                  textController: _textController,
                ),
              ],
            ),
          ),

          // Status bar
          _StatusBar(
            docId: widget.documentId,
            userCount: _otherUsers.length + 1,
            cursorPosition: _textController.selection.baseOffset,
            textLength: _textController.text.length,
          ),
        ],
      ),
    );
  }
}

class _OnlineUsersBanner extends StatelessWidget {
  final List<UserPresence> users;

  const _OnlineUsersBanner({required this.users});

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
      color: Theme.of(context).colorScheme.primaryContainer.withOpacity(0.5),
      child: Row(
        children: [
          const Icon(Icons.group, size: 16),
          const SizedBox(width: 8),
          Text(
            '${users.length + 1} people editing',
            style: const TextStyle(fontSize: 12),
          ),
          const SizedBox(width: 12),
          Expanded(
            child: SingleChildScrollView(
              scrollDirection: Axis.horizontal,
              child: Row(
                children: users.map((user) => Padding(
                  padding: const EdgeInsets.only(right: 8),
                  child: _UserChip(user: user),
                )).toList(),
              ),
            ),
          ),
        ],
      ),
    );
  }
}

class _UserChip extends StatelessWidget {
  final UserPresence user;

  const _UserChip({required this.user});

  @override
  Widget build(BuildContext context) {
    final color = Color(int.parse(user.color.replaceAll('#', '0xFF')));

    return Chip(
      avatar: CircleAvatar(
        backgroundColor: color,
        child: Text(
          user.displayName.isNotEmpty ? user.displayName[0].toUpperCase() : '?',
          style: const TextStyle(color: Colors.white, fontSize: 12),
        ),
      ),
      label: Text(user.displayName, style: const TextStyle(fontSize: 12)),
      backgroundColor: color.withOpacity(0.1),
      side: BorderSide(color: color),
      padding: const EdgeInsets.symmetric(horizontal: 4),
      visualDensity: VisualDensity.compact,
    );
  }
}

class _StatusBar extends StatelessWidget {
  final String docId;
  final int userCount;
  final int cursorPosition;
  final int textLength;

  const _StatusBar({
    required this.docId,
    required this.userCount,
    required this.cursorPosition,
    required this.textLength,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
      color: Theme.of(context).colorScheme.surfaceVariant,
      child: Row(
        children: [
          Icon(Icons.circle, size: 8, color: Colors.green.shade600),
          const SizedBox(width: 4),
          Text('$userCount online', style: const TextStyle(fontSize: 11)),
          const Spacer(),
          Text(
            'Pos: $cursorPosition | Chars: $textLength',
            style: const TextStyle(fontSize: 11),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3166: Presence Indicator Widget

```dart
// lib/widgets/presence_indicator.dart
import 'package:flutter/material.dart';
import '../services/presence_service.dart';

class PresenceIndicatorBar extends StatelessWidget {
  final List<UserPresence> users;

  const PresenceIndicatorBar({super.key, required this.users});

  @override
  Widget build(BuildContext context) {
    if (users.isEmpty) return const SizedBox.shrink();

    return Padding(
      padding: const EdgeInsets.only(right: 8),
      child: Row(
        children: [
          // Show up to 3 avatars
          ...users.take(3).toList().asMap().entries.map((entry) {
            final user = entry.value;
            final color = Color(
              int.parse(user.color.replaceAll('#', '0xFF')),
            );

            return Transform.translate(
              offset: Offset(-entry.key * 8.0, 0),
              child: CircleAvatar(
                radius: 14,
                backgroundColor: color,
                child: Text(
                  user.displayName.isNotEmpty
                      ? user.displayName[0].toUpperCase()
                      : '?',
                  style: const TextStyle(
                    color: Colors.white,
                    fontSize: 12,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ),
            );
          }),

          // Show count if more than 3
          if (users.length > 3)
            Transform.translate(
              offset: const Offset(-24, 0),
              child: CircleAvatar(
                radius: 14,
                backgroundColor: Colors.grey,
                child: Text(
                  '+${users.length - 3}',
                  style: const TextStyle(color: Colors.white, fontSize: 10),
                ),
              ),
            ),
        ],
      ),
    );
  }
}

class PresenceAvatar extends StatelessWidget {
  final UserPresence user;
  final double radius;

  const PresenceAvatar({
    super.key,
    required this.user,
    this.radius = 20,
  });

  @override
  Widget build(BuildContext context) {
    final color = Color(int.parse(user.color.replaceAll('#', '0xFF')));

    return Stack(
      children: [
        CircleAvatar(
          radius: radius,
          backgroundColor: color,
          child: Text(
            user.displayName.isNotEmpty
                ? user.displayName[0].toUpperCase()
                : '?',
            style: TextStyle(
              color: Colors.white,
              fontSize: radius * 0.7,
              fontWeight: FontWeight.bold,
            ),
          ),
        ),
        Positioned(
          right: 0,
          bottom: 0,
          child: Container(
            width: radius * 0.5,
            height: radius * 0.5,
            decoration: BoxDecoration(
              color: user.isOnline ? Colors.green : Colors.grey,
              shape: BoxShape.circle,
              border: Border.all(color: Colors.white, width: 1.5),
            ),
          ),
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 3167: Cursor Overlay Widget

```dart
// lib/widgets/cursor_overlay.dart
import 'package:flutter/material.dart';
import '../services/presence_service.dart';

class CursorOverlay extends StatelessWidget {
  final List<UserPresence> users;
  final TextEditingController textController;

  const CursorOverlay({
    super.key,
    required this.users,
    required this.textController,
  });

  @override
  Widget build(BuildContext context) {
    // In a production app, you would calculate actual pixel positions
    // based on the text layout. This is a simplified visualization.
    return IgnorePointer(
      child: Stack(
        children: users.map((user) {
          final color = Color(
            int.parse(user.color.replaceAll('#', '0xFF')),
          );

          return _RemoteCursor(user: user, color: color);
        }).toList(),
      ),
    );
  }
}

class _RemoteCursor extends StatefulWidget {
  final UserPresence user;
  final Color color;

  const _RemoteCursor({required this.user, required this.color});

  @override
  State<_RemoteCursor> createState() => _RemoteCursorState();
}

class _RemoteCursorState extends State<_RemoteCursor>
    with SingleTickerProviderStateMixin {
  late final AnimationController _blinkController;
  late final Animation<double> _opacityAnimation;

  @override
  void initState() {
    super.initState();
    _blinkController = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 600),
    )..repeat(reverse: true);

    _opacityAnimation = Tween<double>(begin: 1.0, end: 0.3)
        .animate(_blinkController);
  }

  @override
  void dispose() {
    _blinkController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    // Simplified position - in production, use TextPainter to calculate
    final estimatedTop = (widget.user.cursorPosition / 40.0) * 26.0 + 16;

    return Positioned(
      top: estimatedTop.clamp(0, double.infinity),
      left: 16,
      child: AnimatedBuilder(
        animation: _opacityAnimation,
        builder: (context, child) => Opacity(
          opacity: _opacityAnimation.value,
          child: child,
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          mainAxisSize: MainAxisSize.min,
          children: [
            // User name label
            Container(
              padding: const EdgeInsets.symmetric(horizontal: 4, vertical: 2),
              decoration: BoxDecoration(
                color: widget.color,
                borderRadius: const BorderRadius.only(
                  topLeft: Radius.circular(4),
                  topRight: Radius.circular(4),
                ),
              ),
              child: Text(
                widget.user.displayName,
                style: const TextStyle(
                  color: Colors.white,
                  fontSize: 10,
                  fontWeight: FontWeight.bold,
                ),
              ),
            ),
            // Cursor line
            Container(
              width: 2,
              height: 20,
              color: widget.color,
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3168: Document List Screen

```dart
// lib/screens/document_list_screen.dart
import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:intl/intl.dart';
import '../services/document_service.dart';
import '../services/auth_service.dart';
import 'collaborative_editor_screen.dart';

class DocumentListScreen extends StatelessWidget {
  const DocumentListScreen({super.key});

  final _documentService = const DocumentService();

  String get _userId => FirebaseAuth.instance.currentUser!.uid;

  Future<void> _createDocument(BuildContext context) async {
    final titleController = TextEditingController();

    final title = await showDialog<String>(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('New Document'),
        content: TextField(
          controller: titleController,
          autofocus: true,
          decoration: const InputDecoration(
            labelText: 'Document Title',
            border: OutlineInputBorder(),
          ),
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(ctx),
            child: const Text('Cancel'),
          ),
          ElevatedButton(
            onPressed: () => Navigator.pop(ctx, titleController.text),
            child: const Text('Create'),
          ),
        ],
      ),
    );

    if (title != null && title.isNotEmpty && context.mounted) {
      final docId = await _documentService.createDocument(
        title: title,
        ownerId: _userId,
      );

      if (context.mounted) {
        Navigator.push(
          context,
          MaterialPageRoute(
            builder: (_) => CollaborativeEditorScreen(
              documentId: docId,
              title: title,
            ),
          ),
        );
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('My Documents'),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
        actions: [
          IconButton(
            icon: const Icon(Icons.logout),
            onPressed: () => AuthService.instance.signOut(),
          ),
        ],
      ),
      body: StreamBuilder<List<CollaborativeDocument>>(
        stream: _documentService.getDocuments(_userId),
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return const Center(child: CircularProgressIndicator());
          }

          if (snapshot.hasError) {
            return Center(child: Text('Error: ${snapshot.error}'));
          }

          final docs = snapshot.data ?? [];

          if (docs.isEmpty) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  const Icon(Icons.article_outlined, size: 64, color: Colors.grey),
                  const SizedBox(height: 16),
                  const Text(
                    'No documents yet',
                    style: TextStyle(fontSize: 18, color: Colors.grey),
                  ),
                  const SizedBox(height: 8),
                  ElevatedButton.icon(
                    onPressed: () => _createDocument(context),
                    icon: const Icon(Icons.add),
                    label: const Text('Create First Document'),
                  ),
                ],
              ),
            );
          }

          return ListView.builder(
            padding: const EdgeInsets.all(16),
            itemCount: docs.length,
            itemBuilder: (context, index) {
              final doc = docs[index];
              return Card(
                margin: const EdgeInsets.only(bottom: 8),
                child: ListTile(
                  leading: const Icon(Icons.article, size: 40),
                  title: Text(
                    doc.title,
                    style: const TextStyle(fontWeight: FontWeight.bold),
                  ),
                  subtitle: Text(
                    'Updated: ${DateFormat('MMM dd, yyyy HH:mm').format(doc.updatedAt)}',
                  ),
                  trailing: Row(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      Text('v${doc.version}', style: const TextStyle(fontSize: 12)),
                      const SizedBox(width: 8),
                      const Icon(Icons.arrow_forward_ios, size: 16),
                    ],
                  ),
                  onTap: () => Navigator.push(
                    context,
                    MaterialPageRoute(
                      builder: (_) => CollaborativeEditorScreen(
                        documentId: doc.id,
                        title: doc.title,
                      ),
                    ),
                  ),
                ),
              );
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: () => _createDocument(context),
        icon: const Icon(Icons.add),
        label: const Text('New Document'),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3169: Auth Service

```dart
// lib/services/auth_service.dart
import 'package:firebase_auth/firebase_auth.dart';

class AuthService {
  static final AuthService instance = AuthService._();
  final FirebaseAuth _auth = FirebaseAuth.instance;

  AuthService._();

  Stream<User?> get authStateChanges => _auth.authStateChanges();
  User? get currentUser => _auth.currentUser;

  Future<UserCredential> signInAnonymously() async {
    return await _auth.signInAnonymously();
  }

  Future<UserCredential> signInWithEmail(String email, String password) async {
    return await _auth.signInWithEmailAndPassword(
      email: email,
      password: password,
    );
  }

  Future<UserCredential> register(
    String email,
    String password,
    String displayName,
  ) async {
    final cred = await _auth.createUserWithEmailAndPassword(
      email: email,
      password: password,
    );
    await cred.user?.updateDisplayName(displayName);
    return cred;
  }

  Future<void> signOut() async => await _auth.signOut();
}
```

---

## ขั้นตอนที่ 3170: Auth Screen

```dart
// lib/screens/auth_screen.dart
import 'package:flutter/material.dart';
import '../services/auth_service.dart';

class AuthScreen extends StatefulWidget {
  const AuthScreen({super.key});

  @override
  State<AuthScreen> createState() => _AuthScreenState();
}

class _AuthScreenState extends State<AuthScreen>
    with SingleTickerProviderStateMixin {
  late final TabController _tabController;
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  final _nameController = TextEditingController();
  bool _isLoading = false;

  @override
  void initState() {
    super.initState();
    _tabController = TabController(length: 2, vsync: this);
  }

  @override
  void dispose() {
    _tabController.dispose();
    _emailController.dispose();
    _passwordController.dispose();
    _nameController.dispose();
    super.dispose();
  }

  Future<void> _signIn() async {
    setState(() => _isLoading = true);
    try {
      await AuthService.instance.signInWithEmail(
        _emailController.text.trim(),
        _passwordController.text,
      );
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Sign in failed: $e')),
        );
      }
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }

  Future<void> _register() async {
    setState(() => _isLoading = true);
    try {
      await AuthService.instance.register(
        _emailController.text.trim(),
        _passwordController.text,
        _nameController.text.trim(),
      );
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Registration failed: $e')),
        );
      }
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }

  Future<void> _signInAnonymously() async {
    setState(() => _isLoading = true);
    try {
      await AuthService.instance.signInAnonymously();
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Error: $e')),
        );
      }
    } finally {
      if (mounted) setState(() => _isLoading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Collab Editor'),
        bottom: TabBar(
          controller: _tabController,
          tabs: const [
            Tab(text: 'Sign In'),
            Tab(text: 'Register'),
          ],
        ),
      ),
      body: TabBarView(
        controller: _tabController,
        children: [
          // Sign In Tab
          Padding(
            padding: const EdgeInsets.all(24),
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                TextField(
                  controller: _emailController,
                  keyboardType: TextInputType.emailAddress,
                  decoration: const InputDecoration(
                    labelText: 'Email',
                    border: OutlineInputBorder(),
                    prefixIcon: Icon(Icons.email),
                  ),
                ),
                const SizedBox(height: 16),
                TextField(
                  controller: _passwordController,
                  obscureText: true,
                  decoration: const InputDecoration(
                    labelText: 'Password',
                    border: OutlineInputBorder(),
                    prefixIcon: Icon(Icons.lock),
                  ),
                ),
                const SizedBox(height: 24),
                SizedBox(
                  width: double.infinity,
                  child: ElevatedButton(
                    onPressed: _isLoading ? null : _signIn,
                    style: ElevatedButton.styleFrom(padding: const EdgeInsets.all(16)),
                    child: _isLoading
                        ? const CircularProgressIndicator()
                        : const Text('Sign In'),
                  ),
                ),
                const SizedBox(height: 12),
                TextButton(
                  onPressed: _isLoading ? null : _signInAnonymously,
                  child: const Text('Continue Anonymously'),
                ),
              ],
            ),
          ),

          // Register Tab
          Padding(
            padding: const EdgeInsets.all(24),
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                TextField(
                  controller: _nameController,
                  decoration: const InputDecoration(
                    labelText: 'Display Name',
                    border: OutlineInputBorder(),
                    prefixIcon: Icon(Icons.person),
                  ),
                ),
                const SizedBox(height: 16),
                TextField(
                  controller: _emailController,
                  keyboardType: TextInputType.emailAddress,
                  decoration: const InputDecoration(
                    labelText: 'Email',
                    border: OutlineInputBorder(),
                    prefixIcon: Icon(Icons.email),
                  ),
                ),
                const SizedBox(height: 16),
                TextField(
                  controller: _passwordController,
                  obscureText: true,
                  decoration: const InputDecoration(
                    labelText: 'Password',
                    border: OutlineInputBorder(),
                    prefixIcon: Icon(Icons.lock),
                  ),
                ),
                const SizedBox(height: 24),
                SizedBox(
                  width: double.infinity,
                  child: ElevatedButton(
                    onPressed: _isLoading ? null : _register,
                    style: ElevatedButton.styleFrom(padding: const EdgeInsets.all(16)),
                    child: _isLoading
                        ? const CircularProgressIndicator()
                        : const Text('Create Account'),
                  ),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

---

**← [Part 81](part-81-advanced-forms.md)**
**ต่อไป: [Part 83 →](part-83-flutter-ar-vr.md)**
