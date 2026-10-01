# Part 15: Local Storage
## ขั้นตอนที่ 481-520

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ SharedPreferences สำหรับ key-value storage
- ใช้ SQLite ด้วย sqflite
- ใช้ Hive สำหรับ NoSQL storage
- เข้าใจเมื่อไหร่ควรใช้อะไร
- Database migrations

---

## ขั้นตอนที่ 481: SharedPreferences

```dart
// pubspec.yaml:
// dependencies:
//   shared_preferences: ^2.2.0

import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';

// SharedPreferences เหมาะกับ:
// - Settings (theme, language, notifications)
// - Auth token/userId
// - User preferences
// - Simple flags

class PreferencesService {
  static const String _keyTheme = 'theme_mode';
  static const String _keyLanguage = 'language';
  static const String _keyToken = 'auth_token';
  static const String _keyUserId = 'user_id';
  static const String _keyOnboarding = 'onboarding_done';
  static const String _keyFavorites = 'favorites';
  
  static SharedPreferences? _prefs;
  
  static Future<void> init() async {
    _prefs = await SharedPreferences.getInstance();
  }
  
  static SharedPreferences get _instance {
    if (_prefs == null) throw StateError('PreferencesService not initialized');
    return _prefs!;
  }
  
  // Theme
  static ThemeMode get themeMode {
    String? value = _instance.getString(_keyTheme);
    return switch (value) {
      'dark' => ThemeMode.dark,
      'light' => ThemeMode.light,
      _ => ThemeMode.system,
    };
  }
  
  static Future<void> setThemeMode(ThemeMode mode) async {
    await _instance.setString(_keyTheme, mode.name);
  }
  
  // Language
  static String get language => _instance.getString(_keyLanguage) ?? 'th';
  
  static Future<void> setLanguage(String lang) async {
    await _instance.setString(_keyLanguage, lang);
  }
  
  // Auth Token
  static String? get authToken => _instance.getString(_keyToken);
  
  static Future<void> setAuthToken(String token) async {
    await _instance.setString(_keyToken, token);
  }
  
  static Future<void> clearAuthToken() async {
    await _instance.remove(_keyToken);
  }
  
  // Onboarding
  static bool get isOnboardingDone => _instance.getBool(_keyOnboarding) ?? false;
  
  static Future<void> setOnboardingDone() async {
    await _instance.setBool(_keyOnboarding, true);
  }
  
  // Favorites (list of strings)
  static List<String> get favorites => _instance.getStringList(_keyFavorites) ?? [];
  
  static Future<void> addFavorite(String id) async {
    List<String> favs = favorites;
    if (!favs.contains(id)) {
      favs.add(id);
      await _instance.setStringList(_keyFavorites, favs);
    }
  }
  
  static Future<void> removeFavorite(String id) async {
    List<String> favs = favorites;
    favs.remove(id);
    await _instance.setStringList(_keyFavorites, favs);
  }
  
  static bool isFavorite(String id) => favorites.contains(id);
  
  // Clear all
  static Future<void> clearAll() async {
    await _instance.clear();
  }
}

// ─── Settings Screen ───
class SettingsScreen extends StatefulWidget {
  const SettingsScreen({super.key});
  
  @override
  State<SettingsScreen> createState() => _SettingsScreenState();
}

class _SettingsScreenState extends State<SettingsScreen> {
  ThemeMode _themeMode = ThemeMode.system;
  String _language = 'th';
  
  @override
  void initState() {
    super.initState();
    _themeMode = PreferencesService.themeMode;
    _language = PreferencesService.language;
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('การตั้งค่า')),
      body: ListView(
        children: [
          ListTile(
            leading: const Icon(Icons.dark_mode),
            title: const Text('ธีม'),
            trailing: DropdownButton<ThemeMode>(
              value: _themeMode,
              items: const [
                DropdownMenuItem(value: ThemeMode.system, child: Text('ระบบ')),
                DropdownMenuItem(value: ThemeMode.light, child: Text('สว่าง')),
                DropdownMenuItem(value: ThemeMode.dark, child: Text('มืด')),
              ],
              onChanged: (mode) async {
                if (mode != null) {
                  await PreferencesService.setThemeMode(mode);
                  setState(() => _themeMode = mode);
                }
              },
            ),
          ),
          ListTile(
            leading: const Icon(Icons.language),
            title: const Text('ภาษา'),
            trailing: DropdownButton<String>(
              value: _language,
              items: const [
                DropdownMenuItem(value: 'th', child: Text('ไทย')),
                DropdownMenuItem(value: 'en', child: Text('English')),
              ],
              onChanged: (lang) async {
                if (lang != null) {
                  await PreferencesService.setLanguage(lang);
                  setState(() => _language = lang);
                }
              },
            ),
          ),
          ListTile(
            leading: const Icon(Icons.delete_sweep, color: Colors.red),
            title: const Text('ล้างข้อมูลทั้งหมด', style: TextStyle(color: Colors.red)),
            onTap: () async {
              bool? confirm = await showDialog<bool>(
                context: context,
                builder: (_) => AlertDialog(
                  title: const Text('ยืนยัน'),
                  content: const Text('ล้างข้อมูลทั้งหมดใช่ไหม?'),
                  actions: [
                    TextButton(onPressed: () => Navigator.pop(context, false), child: const Text('ยกเลิก')),
                    ElevatedButton(onPressed: () => Navigator.pop(context, true), child: const Text('ยืนยัน')),
                  ],
                ),
              );
              
              if (confirm == true) {
                await PreferencesService.clearAll();
                if (mounted) {
                  ScaffoldMessenger.of(context).showSnackBar(
                    const SnackBar(content: Text('ล้างข้อมูลแล้ว')),
                  );
                }
              }
            },
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 482: SQLite กับ sqflite

```dart
// pubspec.yaml:
// dependencies:
//   sqflite: ^2.3.0
//   path: ^1.9.0

import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';
import 'dart:async';

// ─── Model ───
class Contact {
  final int? id;
  final String name;
  final String email;
  final String? phone;
  final DateTime createdAt;
  
  const Contact({
    this.id,
    required this.name,
    required this.email,
    this.phone,
    required this.createdAt,
  });
  
  factory Contact.fromMap(Map<String, dynamic> map) => Contact(
    id: map['id'] as int?,
    name: map['name'] as String,
    email: map['email'] as String,
    phone: map['phone'] as String?,
    createdAt: DateTime.parse(map['created_at'] as String),
  );
  
  Map<String, dynamic> toMap() => {
    if (id != null) 'id': id,
    'name': name,
    'email': email,
    'phone': phone,
    'created_at': createdAt.toIso8601String(),
  };
  
  Contact copyWith({String? name, String? email, String? phone}) => Contact(
    id: id,
    name: name ?? this.name,
    email: email ?? this.email,
    phone: phone ?? this.phone,
    createdAt: createdAt,
  );
}

// ─── Database Helper ───
class DatabaseHelper {
  static DatabaseHelper? _instance;
  static Database? _database;
  
  DatabaseHelper._();
  
  static DatabaseHelper get instance {
    _instance ??= DatabaseHelper._();
    return _instance!;
  }
  
  Future<Database> get database async {
    _database ??= await _initDatabase();
    return _database!;
  }
  
  Future<Database> _initDatabase() async {
    String path = join(await getDatabasesPath(), 'contacts.db');
    
    return openDatabase(
      path,
      version: 2,
      onCreate: _onCreate,
      onUpgrade: _onUpgrade,
    );
  }
  
  Future<void> _onCreate(Database db, int version) async {
    await db.execute('''
      CREATE TABLE contacts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        phone TEXT,
        created_at TEXT NOT NULL
      )
    ''');
    
    if (version >= 2) {
      await db.execute('''
        CREATE TABLE contact_groups (
          id INTEGER PRIMARY KEY AUTOINCREMENT,
          name TEXT NOT NULL
        )
      ''');
      
      await db.execute('''
        CREATE TABLE contact_group_members (
          contact_id INTEGER,
          group_id INTEGER,
          PRIMARY KEY (contact_id, group_id),
          FOREIGN KEY (contact_id) REFERENCES contacts(id),
          FOREIGN KEY (group_id) REFERENCES contact_groups(id)
        )
      ''');
    }
  }
  
  Future<void> _onUpgrade(Database db, int oldVersion, int newVersion) async {
    if (oldVersion < 2) {
      // Migration: เพิ่ม groups
      await db.execute('''
        CREATE TABLE IF NOT EXISTS contact_groups (
          id INTEGER PRIMARY KEY AUTOINCREMENT,
          name TEXT NOT NULL
        )
      ''');
    }
  }
  
  // CRUD Operations
  Future<int> insertContact(Contact contact) async {
    Database db = await database;
    return db.insert('contacts', contact.toMap(),
        conflictAlgorithm: ConflictAlgorithm.replace);
  }
  
  Future<List<Contact>> getContacts({String? search, String orderBy = 'name'}) async {
    Database db = await database;
    
    String? where;
    List<dynamic>? whereArgs;
    
    if (search != null && search.isNotEmpty) {
      where = 'name LIKE ? OR email LIKE ? OR phone LIKE ?';
      whereArgs = ['%$search%', '%$search%', '%$search%'];
    }
    
    List<Map<String, dynamic>> maps = await db.query(
      'contacts',
      where: where,
      whereArgs: whereArgs,
      orderBy: orderBy,
    );
    
    return maps.map((m) => Contact.fromMap(m)).toList();
  }
  
  Future<Contact?> getContactById(int id) async {
    Database db = await database;
    List<Map<String, dynamic>> maps = await db.query(
      'contacts',
      where: 'id = ?',
      whereArgs: [id],
      limit: 1,
    );
    
    if (maps.isEmpty) return null;
    return Contact.fromMap(maps.first);
  }
  
  Future<int> updateContact(Contact contact) async {
    Database db = await database;
    return db.update(
      'contacts',
      contact.toMap(),
      where: 'id = ?',
      whereArgs: [contact.id],
    );
  }
  
  Future<int> deleteContact(int id) async {
    Database db = await database;
    return db.delete('contacts', where: 'id = ?', whereArgs: [id]);
  }
  
  Future<int> getContactCount() async {
    Database db = await database;
    int? count = Sqflite.firstIntValue(
      await db.rawQuery('SELECT COUNT(*) FROM contacts'),
    );
    return count ?? 0;
  }
  
  // Batch insert
  Future<void> insertContacts(List<Contact> contacts) async {
    Database db = await database;
    Batch batch = db.batch();
    
    for (Contact c in contacts) {
      batch.insert('contacts', c.toMap());
    }
    
    await batch.commit(noResult: true);
  }
  
  Future<void> close() async {
    if (_database != null) {
      await _database!.close();
      _database = null;
    }
  }
}

// ─── Repository ───
class ContactRepository {
  final DatabaseHelper _db = DatabaseHelper.instance;
  
  Future<Contact> create(String name, String email, {String? phone}) async {
    Contact contact = Contact(
      name: name,
      email: email,
      phone: phone,
      createdAt: DateTime.now(),
    );
    
    int id = await _db.insertContact(contact);
    return Contact(id: id, name: name, email: email, phone: phone, createdAt: contact.createdAt);
  }
  
  Future<List<Contact>> getAll({String? search}) => _db.getContacts(search: search);
  
  Future<Contact?> getById(int id) => _db.getContactById(id);
  
  Future<bool> update(Contact contact) async {
    int rowsAffected = await _db.updateContact(contact);
    return rowsAffected > 0;
  }
  
  Future<bool> delete(int id) async {
    int rowsAffected = await _db.deleteContact(id);
    return rowsAffected > 0;
  }
  
  Future<int> count() => _db.getContactCount();
}

void main() async {
  ContactRepository repo = ContactRepository();
  
  // Create
  Contact alice = await repo.create('Alice', 'alice@example.com', phone: '081-234-5678');
  Contact bob = await repo.create('Bob', 'bob@example.com');
  print('Created: ${alice.id} - ${alice.name}');
  
  // Read all
  List<Contact> all = await repo.getAll();
  print('All contacts: ${all.map((c) => c.name).join(', ')}');
  
  // Search
  List<Contact> searched = await repo.getAll(search: 'alice');
  print('Search "alice": ${searched.length} results');
  
  // Update
  Contact updated = alice.copyWith(phone: '089-999-9999');
  bool success = await repo.update(updated);
  print('Updated: $success');
  
  // Delete
  bool deleted = await repo.delete(bob.id!);
  print('Deleted Bob: $deleted');
  
  // Count
  int count = await repo.count();
  print('Total contacts: $count');
}
```

---

## ขั้นตอนที่ 483: Hive

```dart
// pubspec.yaml:
// dependencies:
//   hive: ^2.2.3
//   hive_flutter: ^1.1.0
// dev_dependencies:
//   hive_generator: ^2.0.1
//   build_runner: ^2.4.0

import 'package:hive_flutter/hive_flutter.dart';
import 'package:flutter/material.dart';

// ─── Model with Hive Adapter ───
part 'task.g.dart';  // generated file

@HiveType(typeId: 0)
class Task extends HiveObject {
  @HiveField(0)
  late String id;
  
  @HiveField(1)
  late String title;
  
  @HiveField(2)
  late String? description;
  
  @HiveField(3)
  late bool isDone;
  
  @HiveField(4)
  late DateTime createdAt;
  
  @HiveField(5)
  late String priority;
  
  Task({
    required this.id,
    required this.title,
    this.description,
    this.isDone = false,
    required this.createdAt,
    this.priority = 'normal',
  });
}

// ─── Hive Service ───
class HiveService {
  static const String _boxName = 'tasks';
  
  static Future<void> init() async {
    await Hive.initFlutter();
    Hive.registerAdapter(TaskAdapter());
    await Hive.openBox<Task>(_boxName);
  }
  
  static Box<Task> get _box => Hive.box<Task>(_boxName);
  
  static List<Task> getAll() => _box.values.toList();
  
  static Task? getById(String id) {
    try {
      return _box.values.firstWhere((t) => t.id == id);
    } catch (_) {
      return null;
    }
  }
  
  static Future<void> save(Task task) async {
    await _box.put(task.id, task);
  }
  
  static Future<void> delete(String id) async {
    await _box.delete(id);
  }
  
  static Future<void> deleteAll() async {
    await _box.clear();
  }
  
  // ValueListenable สำหรับ reactive UI
  static ValueListenable<Box<Task>> get listenable => _box.listenable();
}

// ─── Hive App ───
class HiveTaskApp extends StatelessWidget {
  const HiveTaskApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Hive Tasks',
      home: const HiveTaskScreen(),
    );
  }
}

class HiveTaskScreen extends StatelessWidget {
  const HiveTaskScreen({super.key});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Tasks (Hive)')),
      body: ValueListenableBuilder<Box<Task>>(
        valueListenable: HiveService.listenable,
        builder: (context, box, _) {
          List<Task> tasks = box.values.toList();
          
          if (tasks.isEmpty) {
            return const Center(child: Text('ไม่มี Task'));
          }
          
          return ListView.builder(
            itemCount: tasks.length,
            itemBuilder: (ctx, i) {
              Task task = tasks[i];
              return ListTile(
                leading: Checkbox(
                  value: task.isDone,
                  onChanged: (_) async {
                    task.isDone = !task.isDone;
                    await task.save();  // HiveObject.save() ง่ายมาก!
                  },
                ),
                title: Text(
                  task.title,
                  style: TextStyle(
                    decoration: task.isDone ? TextDecoration.lineThrough : null,
                  ),
                ),
                trailing: IconButton(
                  icon: const Icon(Icons.delete),
                  onPressed: () => HiveService.delete(task.id),
                ),
              );
            },
          );
        },
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          await HiveService.save(Task(
            id: DateTime.now().millisecondsSinceEpoch.toString(),
            title: 'Task ${DateTime.now().second}',
            createdAt: DateTime.now(),
          ));
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 484-520: โปรเจกต์ - Note App พร้อม SQLite

```dart
// note_app.dart
import 'package:flutter/material.dart';
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';
import 'dart:async';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await NoteDatabase.instance.database;
  runApp(const NoteApp());
}

// ─── Model ───
class Note {
  final int? id;
  final String title;
  final String content;
  final String color;
  final DateTime updatedAt;
  final bool isPinned;
  
  const Note({
    this.id,
    required this.title,
    required this.content,
    this.color = '#FFFFFF',
    required this.updatedAt,
    this.isPinned = false,
  });
  
  factory Note.fromMap(Map<String, dynamic> map) => Note(
    id: map['id'],
    title: map['title'],
    content: map['content'],
    color: map['color'] ?? '#FFFFFF',
    updatedAt: DateTime.parse(map['updated_at']),
    isPinned: (map['is_pinned'] as int) == 1,
  );
  
  Map<String, dynamic> toMap() => {
    if (id != null) 'id': id,
    'title': title,
    'content': content,
    'color': color,
    'updated_at': updatedAt.toIso8601String(),
    'is_pinned': isPinned ? 1 : 0,
  };
  
  Note copyWith({String? title, String? content, String? color, bool? isPinned}) => Note(
    id: id,
    title: title ?? this.title,
    content: content ?? this.content,
    color: color ?? this.color,
    updatedAt: DateTime.now(),
    isPinned: isPinned ?? this.isPinned,
  );
}

// ─── Database ───
class NoteDatabase {
  static final NoteDatabase instance = NoteDatabase._init();
  static Database? _database;
  
  NoteDatabase._init();
  
  Future<Database> get database async {
    _database ??= await _initDatabase('notes.db');
    return _database!;
  }
  
  Future<Database> _initDatabase(String filePath) async {
    String dbPath = join(await getDatabasesPath(), filePath);
    return openDatabase(dbPath, version: 1, onCreate: _createDB);
  }
  
  Future _createDB(Database db, int version) async {
    await db.execute('''
      CREATE TABLE notes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        content TEXT NOT NULL,
        color TEXT DEFAULT '#FFFFFF',
        updated_at TEXT NOT NULL,
        is_pinned INTEGER NOT NULL DEFAULT 0
      )
    ''');
    
    // Sample data
    Batch batch = db.batch();
    for (Map<String, dynamic> note in _sampleNotes) {
      batch.insert('notes', note);
    }
    await batch.commit();
  }
  
  static List<Map<String, dynamic>> get _sampleNotes => [
    {'title': 'ยินดีต้อนรับ', 'content': 'นี่คือแอปจดโน้ต ใช้งานง่าย', 'color': '#FFF9C4', 'updated_at': DateTime.now().toIso8601String(), 'is_pinned': 1},
    {'title': 'Todo ประจำวัน', 'content': '- ออกกำลังกาย\n- อ่านหนังสือ\n- เรียน Flutter', 'color': '#E8F5E9', 'updated_at': DateTime.now().subtract(const Duration(hours: 1)).toIso8601String(), 'is_pinned': 0},
    {'title': 'ความคิดใหม่', 'content': 'ไอเดียแอปใหม่: แอปติดตามค่าใช้จ่าย', 'color': '#E3F2FD', 'updated_at': DateTime.now().subtract(const Duration(days: 1)).toIso8601String(), 'is_pinned': 0},
  ];
  
  Future<Note> create(Note note) async {
    Database db = await instance.database;
    int id = await db.insert('notes', note.toMap());
    return note.copyWith().._copy(id);
  }
  
  Future<Note?> readNote(int id) async {
    Database db = await instance.database;
    List<Map<String, dynamic>> maps = await db.query(
      'notes',
      where: 'id = ?',
      whereArgs: [id],
    );
    if (maps.isEmpty) return null;
    return Note.fromMap(maps.first);
  }
  
  Future<List<Note>> readAllNotes({String? search}) async {
    Database db = await instance.database;
    
    String? where;
    List<dynamic>? whereArgs;
    
    if (search != null && search.isNotEmpty) {
      where = 'title LIKE ? OR content LIKE ?';
      whereArgs = ['%$search%', '%$search%'];
    }
    
    List<Map<String, dynamic>> maps = await db.query(
      'notes',
      where: where,
      whereArgs: whereArgs,
      orderBy: 'is_pinned DESC, updated_at DESC',
    );
    
    return maps.map(Note.fromMap).toList();
  }
  
  Future<int> update(Note note) async {
    Database db = await instance.database;
    return db.update('notes', note.toMap(), where: 'id = ?', whereArgs: [note.id]);
  }
  
  Future<int> delete(int id) async {
    Database db = await instance.database;
    return db.delete('notes', where: 'id = ?', whereArgs: [id]);
  }
  
  Future close() async {
    Database db = await instance.database;
    db.close();
  }
}

extension on Note {
  void _copy(int newId) {}  // dummy
}

// ─── App ───
class NoteApp extends StatelessWidget {
  const NoteApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Notes',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(colorScheme: ColorScheme.fromSeed(seedColor: Colors.yellow), useMaterial3: true),
      home: const NoteListScreen(),
    );
  }
}

// ─── Note List Screen ───
class NoteListScreen extends StatefulWidget {
  const NoteListScreen({super.key});
  
  @override
  State<NoteListScreen> createState() => _NoteListScreenState();
}

class _NoteListScreenState extends State<NoteListScreen> {
  List<Note> _notes = [];
  bool _isLoading = true;
  String _search = '';
  bool _isGrid = true;
  
  @override
  void initState() {
    super.initState();
    _loadNotes();
  }
  
  Future<void> _loadNotes() async {
    setState(() => _isLoading = true);
    List<Note> notes = await NoteDatabase.instance.readAllNotes(search: _search.isEmpty ? null : _search);
    setState(() {
      _notes = notes;
      _isLoading = false;
    });
  }
  
  Future<void> _deleteNote(Note note) async {
    await NoteDatabase.instance.delete(note.id!);
    _loadNotes();
    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: const Text('ลบโน้ตแล้ว'),
          action: SnackBarAction(label: 'เลิกทำ', onPressed: () async {
            await NoteDatabase.instance.create(note);
            _loadNotes();
          }),
        ),
      );
    }
  }
  
  Future<void> _togglePin(Note note) async {
    await NoteDatabase.instance.update(note.copyWith(isPinned: !note.isPinned));
    _loadNotes();
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('โน้ต'),
        actions: [
          IconButton(
            icon: Icon(_isGrid ? Icons.view_list : Icons.grid_view),
            onPressed: () => setState(() => _isGrid = !_isGrid),
          ),
        ],
      ),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: SearchBar(
              hintText: 'ค้นหาโน้ต...',
              leading: const Icon(Icons.search),
              onChanged: (v) {
                setState(() => _search = v);
                _loadNotes();
              },
            ),
          ),
          Expanded(
            child: _isLoading
                ? const Center(child: CircularProgressIndicator())
                : _notes.isEmpty
                    ? Center(
                        child: Column(
                          mainAxisAlignment: MainAxisAlignment.center,
                          children: [
                            const Icon(Icons.note_outlined, size: 64, color: Colors.grey),
                            const SizedBox(height: 16),
                            Text(_search.isEmpty ? 'ยังไม่มีโน้ต' : 'ไม่พบโน้ต "$_search"',
                                style: const TextStyle(color: Colors.grey)),
                          ],
                        ),
                      )
                    : _isGrid
                        ? GridView.builder(
                            padding: const EdgeInsets.symmetric(horizontal: 16),
                            gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                              crossAxisCount: 2,
                              crossAxisSpacing: 12,
                              mainAxisSpacing: 12,
                            ),
                            itemCount: _notes.length,
                            itemBuilder: (ctx, i) => NoteCard(
                              note: _notes[i],
                              onTap: () => _openNote(_notes[i]),
                              onDelete: () => _deleteNote(_notes[i]),
                              onPin: () => _togglePin(_notes[i]),
                            ),
                          )
                        : ListView.builder(
                            padding: const EdgeInsets.symmetric(horizontal: 16),
                            itemCount: _notes.length,
                            itemBuilder: (ctx, i) => NoteListItem(
                              note: _notes[i],
                              onTap: () => _openNote(_notes[i]),
                              onDelete: () => _deleteNote(_notes[i]),
                              onPin: () => _togglePin(_notes[i]),
                            ),
                          ),
          ),
        ],
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _openNote(null),
        child: const Icon(Icons.add),
      ),
    );
  }
  
  void _openNote(Note? note) async {
    await Navigator.push(
      context,
      MaterialPageRoute(builder: (_) => NoteEditScreen(note: note)),
    );
    _loadNotes();
  }
}

class NoteCard extends StatelessWidget {
  final Note note;
  final VoidCallback onTap;
  final VoidCallback onDelete;
  final VoidCallback onPin;
  
  const NoteCard({super.key, required this.note, required this.onTap, required this.onDelete, required this.onPin});
  
  Color get _bgColor {
    try {
      String hex = note.color.replaceFirst('#', '');
      return Color(int.parse('FF$hex', radix: 16));
    } catch (_) {
      return Colors.white;
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      child: Container(
        padding: const EdgeInsets.all(12),
        decoration: BoxDecoration(
          color: _bgColor,
          borderRadius: BorderRadius.circular(12),
          border: Border.all(color: Colors.grey.withOpacity(0.3)),
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Expanded(
                  child: Text(note.title,
                      style: const TextStyle(fontWeight: FontWeight.bold),
                      maxLines: 1, overflow: TextOverflow.ellipsis),
                ),
                if (note.isPinned) const Icon(Icons.push_pin, size: 16, color: Colors.orange),
              ],
            ),
            const SizedBox(height: 8),
            Expanded(
              child: Text(note.content,
                  style: const TextStyle(fontSize: 13),
                  overflow: TextOverflow.fade),
            ),
            const SizedBox(height: 8),
            Text(
              _formatDate(note.updatedAt),
              style: TextStyle(color: Colors.grey[600], fontSize: 11),
            ),
          ],
        ),
      ),
    );
  }
  
  String _formatDate(DateTime dt) {
    Duration diff = DateTime.now().difference(dt);
    if (diff.inDays > 0) return '${diff.inDays}d ago';
    if (diff.inHours > 0) return '${diff.inHours}h ago';
    return '${diff.inMinutes}m ago';
  }
}

class NoteListItem extends StatelessWidget {
  final Note note;
  final VoidCallback onTap;
  final VoidCallback onDelete;
  final VoidCallback onPin;
  
  const NoteListItem({super.key, required this.note, required this.onTap, required this.onDelete, required this.onPin});
  
  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      child: ListTile(
        title: Text(note.title, style: const TextStyle(fontWeight: FontWeight.bold)),
        subtitle: Text(note.content, maxLines: 2, overflow: TextOverflow.ellipsis),
        trailing: IconButton(icon: Icon(note.isPinned ? Icons.push_pin : Icons.push_pin_outlined), onPressed: onPin),
        onTap: onTap,
      ),
    );
  }
}

// ─── Note Edit Screen ───
class NoteEditScreen extends StatefulWidget {
  final Note? note;
  
  const NoteEditScreen({super.key, this.note});
  
  @override
  State<NoteEditScreen> createState() => _NoteEditScreenState();
}

class _NoteEditScreenState extends State<NoteEditScreen> {
  late TextEditingController _titleCtrl;
  late TextEditingController _contentCtrl;
  late String _color;
  bool _hasChanges = false;
  
  final List<String> _colors = ['#FFFFFF', '#FFF9C4', '#E8F5E9', '#E3F2FD', '#FCE4EC', '#F3E5F5'];
  
  @override
  void initState() {
    super.initState();
    _titleCtrl = TextEditingController(text: widget.note?.title ?? '');
    _contentCtrl = TextEditingController(text: widget.note?.content ?? '');
    _color = widget.note?.color ?? '#FFFFFF';
    
    _titleCtrl.addListener(() => _hasChanges = true);
    _contentCtrl.addListener(() => _hasChanges = true);
  }
  
  @override
  void dispose() {
    _titleCtrl.dispose();
    _contentCtrl.dispose();
    super.dispose();
  }
  
  Future<void> _save() async {
    if (_titleCtrl.text.trim().isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('กรุณาใส่หัวข้อ')));
      return;
    }
    
    Note note = Note(
      id: widget.note?.id,
      title: _titleCtrl.text.trim(),
      content: _contentCtrl.text.trim(),
      color: _color,
      updatedAt: DateTime.now(),
      isPinned: widget.note?.isPinned ?? false,
    );
    
    if (widget.note == null) {
      await NoteDatabase.instance.create(note);
    } else {
      await NoteDatabase.instance.update(note);
    }
    
    if (mounted) Navigator.pop(context);
  }
  
  @override
  Widget build(BuildContext context) {
    Color bgColor;
    try {
      String hex = _color.replaceFirst('#', '');
      bgColor = Color(int.parse('FF$hex', radix: 16));
    } catch (_) {
      bgColor = Colors.white;
    }
    
    return Scaffold(
      backgroundColor: bgColor,
      appBar: AppBar(
        backgroundColor: bgColor,
        elevation: 0,
        actions: [
          // Color picker
          PopupMenuButton<String>(
            icon: const Icon(Icons.palette_outlined),
            itemBuilder: (_) => _colors.map((c) {
              Color color;
              try {
                String hex = c.replaceFirst('#', '');
                color = Color(int.parse('FF$hex', radix: 16));
              } catch (_) {
                color = Colors.white;
              }
              return PopupMenuItem(
                value: c,
                child: Container(width: 24, height: 24, color: color),
              );
            }).toList(),
            onSelected: (c) => setState(() => _color = c),
          ),
          if (widget.note != null)
            IconButton(
              icon: const Icon(Icons.delete_outline, color: Colors.red),
              onPressed: () async {
                await NoteDatabase.instance.delete(widget.note!.id!);
                if (mounted) Navigator.pop(context);
              },
            ),
          IconButton(icon: const Icon(Icons.check), onPressed: _save),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            TextField(
              controller: _titleCtrl,
              style: const TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
              decoration: const InputDecoration(hintText: 'หัวข้อ', border: InputBorder.none),
            ),
            const Divider(),
            Expanded(
              child: TextField(
                controller: _contentCtrl,
                style: const TextStyle(fontSize: 16),
                decoration: const InputDecoration(hintText: 'เขียนโน้ต...', border: InputBorder.none),
                maxLines: null,
                expands: true,
                textAlignVertical: TextAlignVertical.top,
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

**← [Part 14 - HTTP/REST API](part-14-http-rest-api.md)**

**ต่อไป: [Part 16 - Firebase →](part-16-firebase.md)**
