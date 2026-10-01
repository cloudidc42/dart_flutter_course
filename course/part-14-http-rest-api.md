# Part 14: HTTP/REST API
## ขั้นตอนที่ 441-480

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ http package สำหรับ HTTP requests
- GET, POST, PUT, PATCH, DELETE
- JSON Serialization/Deserialization
- Error handling สำหรับ network
- Interceptors และ Headers
- ใช้ Dio package

---

## ขั้นตอนที่ 441: HTTP GET Request

```dart
// pubspec.yaml:
// dependencies:
//   http: ^1.2.0

import 'dart:convert';
import 'package:http/http.dart' as http;

// ─── Simple GET ───
Future<Map<String, dynamic>> fetchPost(int id) async {
  Uri url = Uri.parse('https://jsonplaceholder.typicode.com/posts/$id');
  
  http.Response response = await http.get(url);
  
  if (response.statusCode == 200) {
    return jsonDecode(response.body) as Map<String, dynamic>;
  } else {
    throw Exception('HTTP ${response.statusCode}: ${response.reasonPhrase}');
  }
}

// ─── GET List ───
Future<List<Map<String, dynamic>>> fetchPosts({int limit = 10}) async {
  Uri url = Uri.parse('https://jsonplaceholder.typicode.com/posts').replace(
    queryParameters: {'_limit': limit.toString()},
  );
  
  http.Response response = await http.get(
    url,
    headers: {
      'Accept': 'application/json',
      'Authorization': 'Bearer token_here',
    },
  );
  
  if (response.statusCode == 200) {
    List<dynamic> data = jsonDecode(response.body);
    return data.cast<Map<String, dynamic>>();
  }
  
  throw HttpException(response.statusCode, response.body);
}

class HttpException implements Exception {
  final int statusCode;
  final String body;
  
  HttpException(this.statusCode, this.body);
  
  @override
  String toString() => 'HttpException[$statusCode]: $body';
}

void main() async {
  // Single post
  Map<String, dynamic> post = await fetchPost(1);
  print('Post: ${post['title']}');
  
  // Post list
  List<Map<String, dynamic>> posts = await fetchPosts(limit: 3);
  for (var p in posts) {
    print('- ${p['id']}: ${p['title']}');
  }
}
```

---

## ขั้นตอนที่ 442: JSON Serialization

```dart
import 'dart:convert';

// ─── Manual JSON ───
class User {
  final int id;
  final String name;
  final String email;
  final String? phone;
  final Address address;
  
  const User({
    required this.id,
    required this.name,
    required this.email,
    this.phone,
    required this.address,
  });
  
  // fromJson: JSON → Object
  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'] as int,
      name: json['name'] as String,
      email: json['email'] as String,
      phone: json['phone'] as String?,
      address: Address.fromJson(json['address'] as Map<String, dynamic>),
    );
  }
  
  // toJson: Object → JSON
  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'email': email,
    if (phone != null) 'phone': phone,
    'address': address.toJson(),
  };
  
  @override
  String toString() => 'User(id: $id, name: $name)';
}

class Address {
  final String street;
  final String city;
  final String zipcode;
  
  const Address({required this.street, required this.city, required this.zipcode});
  
  factory Address.fromJson(Map<String, dynamic> json) => Address(
    street: json['street'] as String,
    city: json['city'] as String,
    zipcode: json['zipcode'] as String,
  );
  
  Map<String, dynamic> toJson() => {
    'street': street,
    'city': city,
    'zipcode': zipcode,
  };
}

void main() {
  String jsonString = '''
  {
    "id": 1,
    "name": "Alice",
    "email": "alice@example.com",
    "phone": "081-234-5678",
    "address": {
      "street": "123 Sukhumvit",
      "city": "Bangkok",
      "zipcode": "10110"
    }
  }
  ''';
  
  Map<String, dynamic> json = jsonDecode(jsonString);
  User user = User.fromJson(json);
  print(user);
  print(user.address.city);
  
  // Convert back to JSON
  String backToJson = jsonEncode(user.toJson());
  print(backToJson);
  
  // List of users
  String listJson = '''
  [
    {"id": 1, "name": "Alice", "email": "a@e.com", "address": {"street": "1", "city": "BKK", "zipcode": "10000"}},
    {"id": 2, "name": "Bob", "email": "b@e.com", "address": {"street": "2", "city": "CNX", "zipcode": "50000"}}
  ]
  ''';
  
  List<User> users = (jsonDecode(listJson) as List)
      .map((j) => User.fromJson(j))
      .toList();
  
  users.forEach(print);
}
```

---

## ขั้นตอนที่ 443: POST, PUT, DELETE

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

class ApiClient {
  static const String baseUrl = 'https://jsonplaceholder.typicode.com';
  
  static Map<String, String> get _headers => {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  };
  
  // POST
  static Future<Map<String, dynamic>> createPost({
    required String title,
    required String body,
    required int userId,
  }) async {
    Uri url = Uri.parse('$baseUrl/posts');
    
    http.Response response = await http.post(
      url,
      headers: _headers,
      body: jsonEncode({'title': title, 'body': body, 'userId': userId}),
    );
    
    if (response.statusCode == 201) {
      return jsonDecode(response.body);
    }
    throw Exception('Create failed: ${response.statusCode}');
  }
  
  // PUT (replace entire resource)
  static Future<Map<String, dynamic>> updatePost(
    int id, {
    required String title,
    required String body,
    required int userId,
  }) async {
    Uri url = Uri.parse('$baseUrl/posts/$id');
    
    http.Response response = await http.put(
      url,
      headers: _headers,
      body: jsonEncode({'id': id, 'title': title, 'body': body, 'userId': userId}),
    );
    
    if (response.statusCode == 200) {
      return jsonDecode(response.body);
    }
    throw Exception('Update failed: ${response.statusCode}');
  }
  
  // PATCH (partial update)
  static Future<Map<String, dynamic>> patchPost(
    int id,
    Map<String, dynamic> changes,
  ) async {
    Uri url = Uri.parse('$baseUrl/posts/$id');
    
    http.Response response = await http.patch(
      url,
      headers: _headers,
      body: jsonEncode(changes),
    );
    
    if (response.statusCode == 200) {
      return jsonDecode(response.body);
    }
    throw Exception('Patch failed: ${response.statusCode}');
  }
  
  // DELETE
  static Future<void> deletePost(int id) async {
    Uri url = Uri.parse('$baseUrl/posts/$id');
    
    http.Response response = await http.delete(url, headers: _headers);
    
    if (response.statusCode != 200) {
      throw Exception('Delete failed: ${response.statusCode}');
    }
  }
}

void main() async {
  // Create
  Map<String, dynamic> newPost = await ApiClient.createPost(
    title: 'Flutter Tutorial',
    body: 'Learning Flutter step by step',
    userId: 1,
  );
  print('Created: $newPost');
  
  // Update
  Map<String, dynamic> updated = await ApiClient.updatePost(
    1,
    title: 'Updated Title',
    body: 'Updated body',
    userId: 1,
  );
  print('Updated: $updated');
  
  // Patch
  Map<String, dynamic> patched = await ApiClient.patchPost(1, {'title': 'Patched Title'});
  print('Patched: $patched');
  
  // Delete
  await ApiClient.deletePost(1);
  print('Deleted post 1');
}
```

---

## ขั้นตอนที่ 444: Dio Package

```dart
// pubspec.yaml:
// dependencies:
//   dio: ^5.4.0

import 'package:dio/dio.dart';

class DioClient {
  static final Dio _dio = Dio(
    BaseOptions(
      baseUrl: 'https://jsonplaceholder.typicode.com',
      connectTimeout: const Duration(seconds: 30),
      receiveTimeout: const Duration(seconds: 30),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    ),
  );
  
  static void setup() {
    // Logging interceptor
    _dio.interceptors.add(LogInterceptor(
      requestHeader: true,
      requestBody: true,
      responseHeader: false,
      responseBody: true,
    ));
    
    // Auth interceptor
    _dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) {
        // เพิ่ม auth token
        String? token = _getToken();
        if (token != null) {
          options.headers['Authorization'] = 'Bearer $token';
        }
        handler.next(options);
      },
      onResponse: (response, handler) {
        // บันทึก response time
        print('Response time: ${DateTime.now()}');
        handler.next(response);
      },
      onError: (error, handler) async {
        if (error.response?.statusCode == 401) {
          // Token หมดอายุ - ลอง refresh
          try {
            String newToken = await _refreshToken();
            error.requestOptions.headers['Authorization'] = 'Bearer $newToken';
            
            // Retry request
            Response response = await _dio.fetch(error.requestOptions);
            handler.resolve(response);
            return;
          } catch (e) {
            // Refresh failed - logout user
            print('Token refresh failed: $e');
          }
        }
        handler.next(error);
      },
    ));
    
    // Retry interceptor (manual implementation)
    _dio.interceptors.add(InterceptorsWrapper(
      onError: (error, handler) async {
        if (error.type == DioExceptionType.connectionTimeout ||
            error.type == DioExceptionType.receiveTimeout) {
          int retryCount = error.requestOptions.extra['retryCount'] ?? 0;
          
          if (retryCount < 3) {
            await Future.delayed(Duration(seconds: retryCount + 1));
            error.requestOptions.extra['retryCount'] = retryCount + 1;
            
            try {
              Response response = await _dio.fetch(error.requestOptions);
              handler.resolve(response);
              return;
            } catch (e) {
              // Retry failed
            }
          }
        }
        handler.next(error);
      },
    ));
  }
  
  static String? _getToken() => 'stored_token_here';
  
  static Future<String> _refreshToken() async {
    await Future.delayed(const Duration(milliseconds: 500));
    return 'new_token_here';
  }
  
  // GET
  static Future<T> get<T>(
    String path, {
    Map<String, dynamic>? queryParams,
    T Function(dynamic)? fromJson,
  }) async {
    try {
      Response response = await _dio.get(path, queryParameters: queryParams);
      
      if (fromJson != null) {
        return fromJson(response.data);
      }
      return response.data as T;
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }
  
  // POST
  static Future<T> post<T>(
    String path, {
    dynamic data,
    T Function(dynamic)? fromJson,
  }) async {
    try {
      Response response = await _dio.post(path, data: data);
      
      if (fromJson != null) {
        return fromJson(response.data);
      }
      return response.data as T;
    } on DioException catch (e) {
      throw _handleError(e);
    }
  }
  
  // Upload File
  static Future<Map<String, dynamic>> uploadFile(
    String path,
    String filePath,
    String fieldName,
  ) async {
    FormData formData = FormData.fromMap({
      fieldName: await MultipartFile.fromFile(filePath, filename: filePath.split('/').last),
    });
    
    Response response = await _dio.post(path, data: formData,
      onSendProgress: (sent, total) {
        if (total > 0) print('Upload: ${(sent / total * 100).toInt()}%');
      },
    );
    
    return response.data;
  }
  
  // Download File
  static Future<void> downloadFile(String url, String savePath) async {
    await _dio.download(
      url,
      savePath,
      onReceiveProgress: (received, total) {
        if (total > 0) print('Download: ${(received / total * 100).toInt()}%');
      },
    );
  }
  
  static Exception _handleError(DioException e) {
    return switch (e.type) {
      DioExceptionType.connectionTimeout => Exception('Connection timeout'),
      DioExceptionType.receiveTimeout => Exception('Receive timeout'),
      DioExceptionType.badResponse => Exception('HTTP ${e.response?.statusCode}: ${e.message}'),
      DioExceptionType.cancel => Exception('Request cancelled'),
      _ => Exception('Network error: ${e.message}'),
    };
  }
}

void main() async {
  DioClient.setup();
  
  // GET
  List<dynamic> posts = await DioClient.get<List<dynamic>>(
    '/posts',
    queryParams: {'_limit': '5'},
  );
  print('Posts count: ${posts.length}');
  
  // POST
  Map<String, dynamic> newPost = await DioClient.post<Map<String, dynamic>>(
    '/posts',
    data: {'title': 'Test', 'body': 'Test body', 'userId': 1},
  );
  print('Created: $newPost');
}
```

---

## ขั้นตอนที่ 445-460: Repository Pattern

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

// ─── Model ───
class Post {
  final int id;
  final int userId;
  final String title;
  final String body;
  
  const Post({required this.id, required this.userId, required this.title, required this.body});
  
  factory Post.fromJson(Map<String, dynamic> json) => Post(
    id: json['id'] as int,
    userId: json['userId'] as int,
    title: json['title'] as String,
    body: json['body'] as String,
  );
  
  Map<String, dynamic> toJson() => {
    'id': id,
    'userId': userId,
    'title': title,
    'body': body,
  };
}

// ─── Abstract Repository ───
abstract class PostRepository {
  Future<List<Post>> getPosts({int limit = 10, int offset = 0});
  Future<Post> getPost(int id);
  Future<Post> createPost(String title, String body, int userId);
  Future<Post> updatePost(Post post);
  Future<void> deletePost(int id);
}

// ─── Implementation ───
class PostApiRepository implements PostRepository {
  final String _baseUrl;
  final http.Client _client;
  
  PostApiRepository({
    String? baseUrl,
    http.Client? client,
  })  : _baseUrl = baseUrl ?? 'https://jsonplaceholder.typicode.com',
        _client = client ?? http.Client();
  
  @override
  Future<List<Post>> getPosts({int limit = 10, int offset = 0}) async {
    Uri url = Uri.parse('$_baseUrl/posts').replace(
      queryParameters: {'_limit': '$limit', '_start': '$offset'},
    );
    
    http.Response response = await _client.get(url);
    _checkStatus(response);
    
    List<dynamic> data = jsonDecode(response.body);
    return data.map((j) => Post.fromJson(j)).toList();
  }
  
  @override
  Future<Post> getPost(int id) async {
    http.Response response = await _client.get(Uri.parse('$_baseUrl/posts/$id'));
    _checkStatus(response);
    return Post.fromJson(jsonDecode(response.body));
  }
  
  @override
  Future<Post> createPost(String title, String body, int userId) async {
    http.Response response = await _client.post(
      Uri.parse('$_baseUrl/posts'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({'title': title, 'body': body, 'userId': userId}),
    );
    _checkStatus(response, expectedStatus: 201);
    return Post.fromJson(jsonDecode(response.body));
  }
  
  @override
  Future<Post> updatePost(Post post) async {
    http.Response response = await _client.put(
      Uri.parse('$_baseUrl/posts/${post.id}'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode(post.toJson()),
    );
    _checkStatus(response);
    return Post.fromJson(jsonDecode(response.body));
  }
  
  @override
  Future<void> deletePost(int id) async {
    http.Response response = await _client.delete(Uri.parse('$_baseUrl/posts/$id'));
    _checkStatus(response);
  }
  
  void _checkStatus(http.Response response, {int expectedStatus = 200}) {
    if (response.statusCode != expectedStatus) {
      throw Exception('HTTP ${response.statusCode}: ${response.reasonPhrase}');
    }
  }
}

// ─── Cached Repository (Decorator Pattern) ───
class CachedPostRepository implements PostRepository {
  final PostRepository _delegate;
  final Map<int, Post> _cache = {};
  final Map<String, List<Post>> _listCache = {};
  final Duration _ttl;
  final Map<String, DateTime> _cacheTime = {};
  
  CachedPostRepository(this._delegate, {Duration? ttl})
      : _ttl = ttl ?? const Duration(minutes: 5);
  
  bool _isExpired(String key) {
    DateTime? time = _cacheTime[key];
    if (time == null) return true;
    return DateTime.now().difference(time) > _ttl;
  }
  
  @override
  Future<List<Post>> getPosts({int limit = 10, int offset = 0}) async {
    String key = 'posts_${limit}_$offset';
    
    if (!_isExpired(key) && _listCache.containsKey(key)) {
      print('Cache hit for $key');
      return _listCache[key]!;
    }
    
    List<Post> posts = await _delegate.getPosts(limit: limit, offset: offset);
    _listCache[key] = posts;
    _cacheTime[key] = DateTime.now();
    
    for (Post post in posts) {
      _cache[post.id] = post;
    }
    
    return posts;
  }
  
  @override
  Future<Post> getPost(int id) async {
    String key = 'post_$id';
    
    if (!_isExpired(key) && _cache.containsKey(id)) {
      print('Cache hit for post $id');
      return _cache[id]!;
    }
    
    Post post = await _delegate.getPost(id);
    _cache[id] = post;
    _cacheTime[key] = DateTime.now();
    return post;
  }
  
  @override
  Future<Post> createPost(String title, String body, int userId) async {
    Post post = await _delegate.createPost(title, body, userId);
    _listCache.clear();  // Invalidate list cache
    return post;
  }
  
  @override
  Future<Post> updatePost(Post post) async {
    Post updated = await _delegate.updatePost(post);
    _cache[updated.id] = updated;
    _listCache.clear();
    return updated;
  }
  
  @override
  Future<void> deletePost(int id) async {
    await _delegate.deletePost(id);
    _cache.remove(id);
    _listCache.clear();
  }
}

void main() async {
  PostRepository repo = CachedPostRepository(PostApiRepository());
  
  // First call - fetches from API
  List<Post> posts = await repo.getPosts(limit: 3);
  print('Posts: ${posts.map((p) => p.title).join(', ')}');
  
  // Second call - returns from cache
  List<Post> cachedPosts = await repo.getPosts(limit: 3);
  print('Cached posts count: ${cachedPosts.length}');
  
  // Single post
  Post post = await repo.getPost(1);
  print('Post: ${post.title}');
}
```

---

## ขั้นตอนที่ 461-480: โปรเจกต์ - News API App

```dart
// news_api_app.dart
import 'package:flutter/material.dart';
import 'dart:convert';
import 'package:http/http.dart' as http;

void main() => runApp(const NewsApiApp());

// ─── Model ───
class Article {
  final String title;
  final String? description;
  final String? author;
  final String? urlToImage;
  final String url;
  final DateTime? publishedAt;
  final String sourceName;
  
  const Article({
    required this.title,
    this.description,
    this.author,
    this.urlToImage,
    required this.url,
    this.publishedAt,
    required this.sourceName,
  });
  
  factory Article.fromJson(Map<String, dynamic> json) => Article(
    title: json['title'] ?? 'No Title',
    description: json['description'],
    author: json['author'],
    urlToImage: json['urlToImage'],
    url: json['url'] ?? '',
    publishedAt: json['publishedAt'] != null
        ? DateTime.tryParse(json['publishedAt'])
        : null,
    sourceName: (json['source'] as Map<String, dynamic>?)?['name'] ?? 'Unknown',
  );
  
  String get timeAgo {
    if (publishedAt == null) return '';
    Duration diff = DateTime.now().difference(publishedAt!);
    if (diff.inDays > 0) return '${diff.inDays} วันที่แล้ว';
    if (diff.inHours > 0) return '${diff.inHours} ชั่วโมงที่แล้ว';
    return '${diff.inMinutes} นาทีที่แล้ว';
  }
}

// ─── Repository ───
class NewsRepository {
  static const String _baseUrl = 'https://newsapi.org/v2';
  static const String _apiKey = 'YOUR_API_KEY';  // ใส่ API key จริง
  
  // จำลองข้อมูล (เนื่องจาก API key ต้องสมัคร)
  static List<Article> _mockArticles() => List.generate(10, (i) => Article(
    title: 'ข่าวเทคโนโลยี ${i + 1}: Flutter 4.0 เปิดตัวแล้ว',
    description: 'รายละเอียดข่าวที่ ${i + 1} เกี่ยวกับการพัฒนาเทคโนโลยีใหม่ๆ ที่น่าสนใจ',
    author: 'ผู้สื่อข่าว ${i % 3 + 1}',
    urlToImage: 'https://via.placeholder.com/400x200',
    url: 'https://example.com/news/${i + 1}',
    publishedAt: DateTime.now().subtract(Duration(hours: i * 2)),
    sourceName: ['TechCrunch', 'The Verge', 'Wired'][i % 3],
  ));
  
  Future<List<Article>> getTopHeadlines({String? category}) async {
    // จำลอง API delay
    await Future.delayed(const Duration(seconds: 1));
    return _mockArticles();
    
    // Real API call (ต้องใช้ API key จริง):
    // Uri url = Uri.parse('$_baseUrl/top-headlines').replace(
    //   queryParameters: {
    //     'apiKey': _apiKey,
    //     'country': 'th',
    //     if (category != null) 'category': category,
    //   },
    // );
    // http.Response response = await http.get(url);
    // if (response.statusCode == 200) {
    //   Map<String, dynamic> data = jsonDecode(response.body);
    //   return (data['articles'] as List).map((j) => Article.fromJson(j)).toList();
    // }
    // throw Exception('Failed to load news');
  }
  
  Future<List<Article>> searchArticles(String query) async {
    await Future.delayed(const Duration(milliseconds: 800));
    return _mockArticles()
        .where((a) => a.title.toLowerCase().contains(query.toLowerCase()))
        .toList();
  }
}

// ─── App ───
class NewsApiApp extends StatelessWidget {
  const NewsApiApp({super.key});
  
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'News App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue), useMaterial3: true),
      home: const NewsListScreen(),
    );
  }
}

class NewsListScreen extends StatefulWidget {
  const NewsListScreen({super.key});
  
  @override
  State<NewsListScreen> createState() => _NewsListScreenState();
}

class _NewsListScreenState extends State<NewsListScreen> {
  final NewsRepository _repo = NewsRepository();
  final TextEditingController _searchController = TextEditingController();
  
  late Future<List<Article>> _articlesFuture;
  bool _isSearching = false;
  String _selectedCategory = 'ทั้งหมด';
  
  final List<String> _categories = ['ทั้งหมด', 'Technology', 'Business', 'Sports', 'Entertainment'];
  
  @override
  void initState() {
    super.initState();
    _articlesFuture = _repo.getTopHeadlines();
  }
  
  @override
  void dispose() {
    _searchController.dispose();
    super.dispose();
  }
  
  void _search(String query) {
    if (query.isEmpty) {
      setState(() => _articlesFuture = _repo.getTopHeadlines());
    } else {
      setState(() => _articlesFuture = _repo.searchArticles(query));
    }
  }
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: _isSearching
            ? TextField(
                controller: _searchController,
                autofocus: true,
                decoration: const InputDecoration(hintText: 'ค้นหา...', border: InputBorder.none),
                onChanged: _search,
              )
            : const Text('ข่าวสาร'),
        actions: [
          IconButton(
            icon: Icon(_isSearching ? Icons.close : Icons.search),
            onPressed: () {
              setState(() {
                _isSearching = !_isSearching;
                if (!_isSearching) {
                  _searchController.clear();
                  _articlesFuture = _repo.getTopHeadlines();
                }
              });
            },
          ),
        ],
      ),
      body: Column(
        children: [
          // Category chips
          SizedBox(
            height: 48,
            child: ListView.builder(
              scrollDirection: Axis.horizontal,
              padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
              itemCount: _categories.length,
              itemBuilder: (ctx, i) => Padding(
                padding: const EdgeInsets.only(right: 8),
                child: FilterChip(
                  label: Text(_categories[i]),
                  selected: _selectedCategory == _categories[i],
                  onSelected: (_) {
                    setState(() {
                      _selectedCategory = _categories[i];
                      _articlesFuture = _repo.getTopHeadlines(
                        category: _categories[i] == 'ทั้งหมด' ? null : _categories[i].toLowerCase(),
                      );
                    });
                  },
                ),
              ),
            ),
          ),
          
          // News list
          Expanded(
            child: FutureBuilder<List<Article>>(
              future: _articlesFuture,
              builder: (ctx, snapshot) {
                if (snapshot.connectionState == ConnectionState.waiting) {
                  return const Center(child: CircularProgressIndicator());
                }
                
                if (snapshot.hasError) {
                  return Center(
                    child: Column(
                      mainAxisAlignment: MainAxisAlignment.center,
                      children: [
                        const Icon(Icons.error_outline, size: 64, color: Colors.red),
                        const SizedBox(height: 16),
                        Text('Error: ${snapshot.error}'),
                        const SizedBox(height: 16),
                        ElevatedButton(
                          onPressed: () => setState(() {
                            _articlesFuture = _repo.getTopHeadlines();
                          }),
                          child: const Text('ลองใหม่'),
                        ),
                      ],
                    ),
                  );
                }
                
                List<Article> articles = snapshot.data ?? [];
                
                if (articles.isEmpty) {
                  return const Center(child: Text('ไม่พบข่าว'));
                }
                
                return RefreshIndicator(
                  onRefresh: () async {
                    setState(() {
                      _articlesFuture = _repo.getTopHeadlines();
                    });
                    await _articlesFuture;
                  },
                  child: ListView.builder(
                    itemCount: articles.length,
                    itemBuilder: (ctx, i) => ArticleListTile(article: articles[i]),
                  ),
                );
              },
            ),
          ),
        ],
      ),
    );
  }
}

class ArticleListTile extends StatelessWidget {
  final Article article;
  
  const ArticleListTile({super.key, required this.article});
  
  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
      child: InkWell(
        onTap: () {},
        borderRadius: BorderRadius.circular(12),
        child: Padding(
          padding: const EdgeInsets.all(12),
          child: Row(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              ClipRRect(
                borderRadius: BorderRadius.circular(8),
                child: Image.network(
                  article.urlToImage ?? 'https://via.placeholder.com/80',
                  width: 80,
                  height: 80,
                  fit: BoxFit.cover,
                  errorBuilder: (c, e, s) => Container(
                    width: 80,
                    height: 80,
                    color: Colors.grey[200],
                    child: const Icon(Icons.image, size: 32),
                  ),
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      article.sourceName,
                      style: TextStyle(color: Theme.of(context).colorScheme.primary, fontSize: 11, fontWeight: FontWeight.bold),
                    ),
                    const SizedBox(height: 4),
                    Text(
                      article.title,
                      style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 14),
                      maxLines: 2,
                      overflow: TextOverflow.ellipsis,
                    ),
                    const SizedBox(height: 4),
                    Row(
                      children: [
                        if (article.author != null) ...[
                          Text(article.author!, style: TextStyle(color: Colors.grey[600], fontSize: 11)),
                          const Text(' • ', style: TextStyle(color: Colors.grey)),
                        ],
                        Text(article.timeAgo, style: TextStyle(color: Colors.grey[600], fontSize: 11)),
                      ],
                    ),
                  ],
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

**← [Part 13 - Navigation](part-13-navigation-routing.md)**

**ต่อไป: [Part 15 - Local Storage →](part-15-local-storage.md)**
