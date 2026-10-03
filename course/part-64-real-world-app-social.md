# Part 64: Real-World App — Social Network
## ขั้นตอนที่ 2441-2480

## 🎯 เป้าหมายของ Part นี้
- Social feed พร้อม infinite scroll
- Post creation พร้อม image/video
- Like, comment, share system
- Follow/unfollow พร้อม count
- User profile screen
- Clean Architecture + Riverpod

---

## ขั้นตอนที่ 2441: pubspec.yaml

```yaml
# pubspec.yaml
name: flutter_social
description: Social Network App

environment:
  sdk: ">=3.0.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.4.0
  go_router: ^12.1.1
  cached_network_image: ^3.3.0
  image_picker: ^1.0.4
  video_player: ^2.7.2
  timeago: ^3.6.0
  equatable: ^2.0.5
  share_plus: ^7.2.1
  infinite_scroll_pagination: ^4.0.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
```

---

## ขั้นตอนที่ 2442: Domain Models

```dart
// lib/domain/models/user.dart
import 'package:equatable/equatable.dart';

class SocialUser extends Equatable {
  final String id;
  final String username;
  final String displayName;
  final String? bio;
  final String? avatarUrl;
  final String? coverUrl;
  final int followersCount;
  final int followingCount;
  final int postsCount;
  final bool isVerified;
  final bool isFollowedByMe;
  final bool isPrivate;
  final DateTime createdAt;

  const SocialUser({
    required this.id,
    required this.username,
    required this.displayName,
    this.bio,
    this.avatarUrl,
    this.coverUrl,
    this.followersCount = 0,
    this.followingCount = 0,
    this.postsCount = 0,
    this.isVerified = false,
    this.isFollowedByMe = false,
    this.isPrivate = false,
    required this.createdAt,
  });

  SocialUser copyWith({
    int? followersCount,
    int? followingCount,
    int? postsCount,
    bool? isFollowedByMe,
  }) {
    return SocialUser(
      id: id,
      username: username,
      displayName: displayName,
      bio: bio,
      avatarUrl: avatarUrl,
      coverUrl: coverUrl,
      followersCount: followersCount ?? this.followersCount,
      followingCount: followingCount ?? this.followingCount,
      postsCount: postsCount ?? this.postsCount,
      isVerified: isVerified,
      isFollowedByMe: isFollowedByMe ?? this.isFollowedByMe,
      isPrivate: isPrivate,
      createdAt: createdAt,
    );
  }

  @override
  List<Object?> get props => [id, username, isFollowedByMe, followersCount];
}
```

```dart
// lib/domain/models/post.dart
import 'package:equatable/equatable.dart';
import 'user.dart';

enum MediaType { image, video }

class PostMedia extends Equatable {
  final String id;
  final String url;
  final MediaType type;
  final double? aspectRatio;
  final String? thumbnailUrl;

  const PostMedia({
    required this.id,
    required this.url,
    required this.type,
    this.aspectRatio,
    this.thumbnailUrl,
  });

  @override
  List<Object?> get props => [id, url, type];
}

class Comment extends Equatable {
  final String id;
  final SocialUser author;
  final String content;
  final int likesCount;
  final bool isLikedByMe;
  final DateTime createdAt;
  final List<Comment> replies;

  const Comment({
    required this.id,
    required this.author,
    required this.content,
    this.likesCount = 0,
    this.isLikedByMe = false,
    required this.createdAt,
    this.replies = const [],
  });

  Comment copyWith({int? likesCount, bool? isLikedByMe}) {
    return Comment(
      id: id,
      author: author,
      content: content,
      likesCount: likesCount ?? this.likesCount,
      isLikedByMe: isLikedByMe ?? this.isLikedByMe,
      createdAt: createdAt,
      replies: replies,
    );
  }

  @override
  List<Object?> get props => [id, content, likesCount, isLikedByMe];
}

class Post extends Equatable {
  final String id;
  final SocialUser author;
  final String? caption;
  final List<PostMedia> media;
  final int likesCount;
  final int commentsCount;
  final int sharesCount;
  final bool isLikedByMe;
  final bool isSavedByMe;
  final List<String> hashtags;
  final List<SocialUser> taggedUsers;
  final DateTime createdAt;
  final String? location;

  const Post({
    required this.id,
    required this.author,
    this.caption,
    this.media = const [],
    this.likesCount = 0,
    this.commentsCount = 0,
    this.sharesCount = 0,
    this.isLikedByMe = false,
    this.isSavedByMe = false,
    this.hashtags = const [],
    this.taggedUsers = const [],
    required this.createdAt,
    this.location,
  });

  bool get hasMedia => media.isNotEmpty;
  bool get hasMultipleMedia => media.length > 1;

  Post copyWith({
    int? likesCount,
    int? commentsCount,
    int? sharesCount,
    bool? isLikedByMe,
    bool? isSavedByMe,
  }) {
    return Post(
      id: id,
      author: author,
      caption: caption,
      media: media,
      likesCount: likesCount ?? this.likesCount,
      commentsCount: commentsCount ?? this.commentsCount,
      sharesCount: sharesCount ?? this.sharesCount,
      isLikedByMe: isLikedByMe ?? this.isLikedByMe,
      isSavedByMe: isSavedByMe ?? this.isSavedByMe,
      hashtags: hashtags,
      taggedUsers: taggedUsers,
      createdAt: createdAt,
      location: location,
    );
  }

  @override
  List<Object?> get props => [id, likesCount, commentsCount, isLikedByMe];
}
```

---

## ขั้นตอนที่ 2443: Mock Data Service

```dart
// lib/data/mock_social_data.dart
import '../domain/models/user.dart';
import '../domain/models/post.dart';

class MockSocialData {
  static final List<SocialUser> users = [
    SocialUser(
      id: 'u1',
      username: 'alice_dev',
      displayName: 'Alice Johnson',
      bio: 'Flutter Developer | Coffee Lover ☕ | Building awesome apps',
      avatarUrl: 'https://i.pravatar.cc/150?img=1',
      coverUrl: 'https://picsum.photos/seed/alice/800/300',
      followersCount: 1250,
      followingCount: 320,
      postsCount: 48,
      isVerified: true,
      createdAt: DateTime(2022, 3, 15),
    ),
    SocialUser(
      id: 'u2',
      username: 'bob_codes',
      displayName: 'Bob Smith',
      bio: 'iOS & Android | Dart enthusiast | Open source contributor',
      avatarUrl: 'https://i.pravatar.cc/150?img=2',
      followersCount: 856,
      followingCount: 210,
      postsCount: 32,
      createdAt: DateTime(2022, 6, 20),
    ),
    SocialUser(
      id: 'u3',
      username: 'charlie_ui',
      displayName: 'Charlie Brown',
      bio: 'UI/UX Designer turned Flutter dev',
      avatarUrl: 'https://i.pravatar.cc/150?img=3',
      followersCount: 3200,
      followingCount: 540,
      postsCount: 125,
      isVerified: true,
      isFollowedByMe: true,
      createdAt: DateTime(2021, 10, 5),
    ),
  ];

  static List<Post> generatePosts() {
    final results = <Post>[];

    final captions = [
      'Just shipped a new Flutter feature! 🚀 #flutter #dart #mobiledev',
      'Learning about Riverpod state management today. Mind blown! 🤯',
      'Beautiful sunset from my workspace window 🌅',
      'Working on a new open source project. Stay tuned! #opensource #coding',
      'Coffee + Code = Perfect morning ☕💻 #developer #productivity',
      'New blog post about Flutter animations is live! Link in bio 🎯',
    ];

    for (var i = 0; i < 20; i++) {
      final user = users[i % users.length];
      final hasImage = i % 3 != 2;
      final caption = captions[i % captions.length];

      results.add(Post(
        id: 'post_$i',
        author: user,
        caption: caption,
        media: hasImage
            ? [
                PostMedia(
                  id: 'media_$i',
                  url: 'https://picsum.photos/seed/post$i/600/400',
                  type: MediaType.image,
                  aspectRatio: 1.5,
                ),
              ]
            : [],
        likesCount: (i + 1) * 15 + (i * 7),
        commentsCount: (i + 1) * 3,
        sharesCount: i * 2 + 1,
        isLikedByMe: i % 4 == 0,
        isSavedByMe: i % 6 == 0,
        hashtags: ['flutter', 'dart', 'coding'],
        createdAt: DateTime.now().subtract(Duration(hours: i * 3 + 1)),
        location: i % 5 == 0 ? 'Bangkok, Thailand' : null,
      ));
    }

    return results;
  }
}
```

---

## ขั้นตอนที่ 2444: Social Repository

```dart
// lib/data/repositories/social_repository.dart
import '../../domain/models/user.dart';
import '../../domain/models/post.dart';
import '../mock_social_data.dart';

abstract class SocialRepository {
  Future<List<Post>> getFeedPosts({int page, int pageSize});
  Future<List<Post>> getUserPosts(String userId);
  Future<SocialUser> getUser(String userId);
  Future<SocialUser> getCurrentUser();
  Future<bool> toggleLike(String postId);
  Future<bool> toggleFollow(String userId);
  Future<bool> toggleSave(String postId);
  Future<Comment> addComment(String postId, String content);
  Future<List<Comment>> getComments(String postId);
  Future<void> deletePost(String postId);
  Future<Post> createPost({String? caption, List<String>? mediaUrls});
}

class MockSocialRepository implements SocialRepository {
  final List<Post> _posts = MockSocialData.generatePosts();
  final List<SocialUser> _users = MockSocialData.users;
  final Map<String, bool> _likedPosts = {};
  final Map<String, bool> _savedPosts = {};
  final Map<String, bool> _followedUsers = {};
  final Map<String, List<Comment>> _comments = {};

  final SocialUser _currentUser = SocialUser(
    id: 'current_user',
    username: 'me',
    displayName: 'You',
    avatarUrl: 'https://i.pravatar.cc/150?img=10',
    followersCount: 100,
    followingCount: 50,
    postsCount: 10,
    createdAt: DateTime(2023, 1, 1),
  );

  @override
  Future<List<Post>> getFeedPosts({int page = 1, int pageSize = 10}) async {
    await Future.delayed(const Duration(milliseconds: 500));
    final start = (page - 1) * pageSize;
    final end = (start + pageSize).clamp(0, _posts.length);
    if (start >= _posts.length) return [];
    return _posts.sublist(start, end).map((post) {
      return post.copyWith(
        isLikedByMe: _likedPosts[post.id] ?? post.isLikedByMe,
        isSavedByMe: _savedPosts[post.id] ?? post.isSavedByMe,
      );
    }).toList();
  }

  @override
  Future<List<Post>> getUserPosts(String userId) async {
    await Future.delayed(const Duration(milliseconds: 300));
    return _posts.where((p) => p.author.id == userId).toList();
  }

  @override
  Future<SocialUser> getUser(String userId) async {
    await Future.delayed(const Duration(milliseconds: 200));
    return _users.firstWhere(
      (u) => u.id == userId,
      orElse: () => throw Exception('User not found'),
    );
  }

  @override
  Future<SocialUser> getCurrentUser() async {
    return _currentUser;
  }

  @override
  Future<bool> toggleLike(String postId) async {
    await Future.delayed(const Duration(milliseconds: 100));
    final current = _likedPosts[postId] ??
        _posts.firstWhere((p) => p.id == postId).isLikedByMe;
    _likedPosts[postId] = !current;
    return !current;
  }

  @override
  Future<bool> toggleFollow(String userId) async {
    await Future.delayed(const Duration(milliseconds: 200));
    final current = _followedUsers[userId] ?? false;
    _followedUsers[userId] = !current;
    return !current;
  }

  @override
  Future<bool> toggleSave(String postId) async {
    await Future.delayed(const Duration(milliseconds: 100));
    final current = _savedPosts[postId] ?? false;
    _savedPosts[postId] = !current;
    return !current;
  }

  @override
  Future<Comment> addComment(String postId, String content) async {
    await Future.delayed(const Duration(milliseconds: 300));
    final comment = Comment(
      id: 'comment_${DateTime.now().millisecondsSinceEpoch}',
      author: _currentUser,
      content: content,
      createdAt: DateTime.now(),
    );
    _comments[postId] = [...(_comments[postId] ?? []), comment];
    return comment;
  }

  @override
  Future<List<Comment>> getComments(String postId) async {
    await Future.delayed(const Duration(milliseconds: 300));

    if (!_comments.containsKey(postId)) {
      _comments[postId] = List.generate(
        3,
        (i) => Comment(
          id: 'c${postId}_$i',
          author: _users[i % _users.length],
          content: 'Great post! Comment number $i 🔥',
          likesCount: i * 3,
          createdAt: DateTime.now().subtract(Duration(minutes: i * 10 + 5)),
        ),
      );
    }

    return _comments[postId]!;
  }

  @override
  Future<void> deletePost(String postId) async {
    await Future.delayed(const Duration(milliseconds: 200));
    _posts.removeWhere((p) => p.id == postId);
  }

  @override
  Future<Post> createPost({String? caption, List<String>? mediaUrls}) async {
    await Future.delayed(const Duration(milliseconds: 500));
    final newPost = Post(
      id: 'post_${DateTime.now().millisecondsSinceEpoch}',
      author: _currentUser,
      caption: caption,
      media: mediaUrls
              ?.map((url) => PostMedia(
                    id: 'media_${url.hashCode}',
                    url: url,
                    type: MediaType.image,
                    aspectRatio: 1.0,
                  ))
              .toList() ??
          [],
      createdAt: DateTime.now(),
    );
    _posts.insert(0, newPost);
    return newPost;
  }
}
```

---

## ขั้นตอนที่ 2445: Riverpod Providers for Social

```dart
// lib/providers/social_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../data/repositories/social_repository.dart';
import '../domain/models/post.dart';
import '../domain/models/user.dart';

final socialRepositoryProvider = Provider<SocialRepository>(
  (ref) => MockSocialRepository(),
);

final currentUserProvider = FutureProvider<SocialUser>((ref) async {
  return ref.watch(socialRepositoryProvider).getCurrentUser();
});

// Feed with pagination
class FeedNotifier extends StateNotifier<AsyncValue<List<Post>>> {
  final SocialRepository _repository;
  int _currentPage = 1;
  bool _hasMore = true;
  bool _isFetching = false;

  FeedNotifier(this._repository) : super(const AsyncValue.loading()) {
    loadInitial();
  }

  Future<void> loadInitial() async {
    state = const AsyncValue.loading();
    _currentPage = 1;
    _hasMore = true;
    try {
      final posts = await _repository.getFeedPosts(page: 1);
      _currentPage = 2;
      _hasMore = posts.length >= 10;
      state = AsyncValue.data(posts);
    } catch (e, st) {
      state = AsyncValue.error(e, st);
    }
  }

  Future<void> loadMore() async {
    if (!_hasMore || _isFetching) return;
    final currentPosts = state.valueOrNull ?? [];
    _isFetching = true;
    try {
      final newPosts =
          await _repository.getFeedPosts(page: _currentPage);
      _currentPage++;
      _hasMore = newPosts.length >= 10;
      state = AsyncValue.data([...currentPosts, ...newPosts]);
    } catch (_) {
      // Keep existing posts on error
    } finally {
      _isFetching = false;
    }
  }

  Future<void> toggleLike(String postId) async {
    final posts = state.valueOrNull ?? [];
    final index = posts.indexWhere((p) => p.id == postId);
    if (index < 0) return;

    final post = posts[index];
    final wasLiked = post.isLikedByMe;

    // Optimistic update
    final updatedPosts = List<Post>.from(posts);
    updatedPosts[index] = post.copyWith(
      isLikedByMe: !wasLiked,
      likesCount: wasLiked ? post.likesCount - 1 : post.likesCount + 1,
    );
    state = AsyncValue.data(updatedPosts);

    try {
      await _repository.toggleLike(postId);
    } catch (_) {
      // Revert on error
      final revertedPosts = List<Post>.from(updatedPosts);
      revertedPosts[index] = post;
      state = AsyncValue.data(revertedPosts);
    }
  }

  Future<void> toggleSave(String postId) async {
    final posts = state.valueOrNull ?? [];
    final index = posts.indexWhere((p) => p.id == postId);
    if (index < 0) return;

    final post = posts[index];
    final updatedPosts = List<Post>.from(posts);
    updatedPosts[index] = post.copyWith(isSavedByMe: !post.isSavedByMe);
    state = AsyncValue.data(updatedPosts);

    try {
      await _repository.toggleSave(postId);
    } catch (_) {
      final revertedPosts = List<Post>.from(updatedPosts);
      revertedPosts[index] = post;
      state = AsyncValue.data(revertedPosts);
    }
  }

  void addPost(Post post) {
    final posts = state.valueOrNull ?? [];
    state = AsyncValue.data([post, ...posts]);
  }
}

final feedProvider =
    StateNotifierProvider<FeedNotifier, AsyncValue<List<Post>>>(
  (ref) => FeedNotifier(ref.watch(socialRepositoryProvider)),
);

// User profile
final userProfileProvider =
    FutureProvider.family<SocialUser, String>((ref, userId) async {
  return ref.watch(socialRepositoryProvider).getUser(userId);
});

final userPostsProvider =
    FutureProvider.family<List<Post>, String>((ref, userId) async {
  return ref.watch(socialRepositoryProvider).getUserPosts(userId);
});

// Comments
final commentsProvider =
    FutureProvider.family<List<Comment>, String>((ref, postId) async {
  return ref.watch(socialRepositoryProvider).getComments(postId);
});
```

---

## ขั้นตอนที่ 2446: Social Feed Screen

```dart
// lib/screens/feed_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:cached_network_image/cached_network_image.dart';
import 'package:timeago/timeago.dart' as timeago;
import '../domain/models/post.dart';
import '../domain/models/user.dart';
import '../providers/social_providers.dart';

class FeedScreen extends ConsumerStatefulWidget {
  const FeedScreen({super.key});

  @override
  ConsumerState<FeedScreen> createState() => _FeedScreenState();
}

class _FeedScreenState extends ConsumerState<FeedScreen> {
  final ScrollController _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(_onScroll);
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  void _onScroll() {
    if (_scrollController.position.pixels >=
        _scrollController.position.maxScrollExtent - 200) {
      ref.read(feedProvider.notifier).loadMore();
    }
  }

  @override
  Widget build(BuildContext context) {
    final feedState = ref.watch(feedProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Feed'),
        actions: [
          IconButton(
            icon: const Icon(Icons.add_box_outlined),
            onPressed: () => Navigator.pushNamed(context, '/create-post'),
          ),
          IconButton(
            icon: const Icon(Icons.notifications_outlined),
            onPressed: () {},
          ),
        ],
      ),
      body: RefreshIndicator(
        onRefresh: () => ref.read(feedProvider.notifier).loadInitial(),
        child: feedState.when(
          loading: () => const Center(child: CircularProgressIndicator()),
          error: (e, _) => Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                Text('Error: $e'),
                ElevatedButton(
                  onPressed: () =>
                      ref.read(feedProvider.notifier).loadInitial(),
                  child: const Text('Retry'),
                ),
              ],
            ),
          ),
          data: (posts) => ListView.builder(
            controller: _scrollController,
            itemCount: posts.length + 1,
            itemBuilder: (context, index) {
              if (index == posts.length) {
                return const Padding(
                  padding: EdgeInsets.all(16),
                  child: Center(child: CircularProgressIndicator()),
                );
              }
              return PostCard(post: posts[index]);
            },
          ),
        ),
      ),
    );
  }
}

class PostCard extends ConsumerWidget {
  final Post post;

  const PostCard({super.key, required this.post});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Card(
      margin: const EdgeInsets.symmetric(vertical: 4),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          _buildHeader(context),
          if (post.hasMedia) _buildMedia(),
          _buildActions(context, ref),
          _buildCaption(context),
          _buildCommentPreview(context),
          _buildTimestamp(),
        ],
      ),
    );
  }

  Widget _buildHeader(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.all(12),
      child: Row(
        children: [
          GestureDetector(
            onTap: () =>
                Navigator.pushNamed(context, '/profile/${post.author.id}'),
            child: CircleAvatar(
              radius: 20,
              backgroundImage: post.author.avatarUrl != null
                  ? CachedNetworkImageProvider(post.author.avatarUrl!)
                  : null,
              child: post.author.avatarUrl == null
                  ? Text(post.author.displayName[0])
                  : null,
            ),
          ),
          const SizedBox(width: 12),
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Row(
                  children: [
                    Text(
                      post.author.displayName,
                      style: const TextStyle(fontWeight: FontWeight.bold),
                    ),
                    if (post.author.isVerified) ...[
                      const SizedBox(width: 4),
                      const Icon(Icons.verified, size: 16, color: Colors.blue),
                    ],
                  ],
                ),
                if (post.location != null)
                  Text(
                    post.location!,
                    style: TextStyle(
                        color: Colors.grey.shade600, fontSize: 12),
                  ),
              ],
            ),
          ),
          IconButton(
            icon: const Icon(Icons.more_horiz),
            onPressed: () => _showPostOptions(context),
          ),
        ],
      ),
    );
  }

  Widget _buildMedia() {
    if (post.media.isEmpty) return const SizedBox.shrink();
    final media = post.media.first;

    return AspectRatio(
      aspectRatio: media.aspectRatio ?? 1.0,
      child: CachedNetworkImage(
        imageUrl: media.url,
        fit: BoxFit.cover,
        placeholder: (_, __) => Container(color: Colors.grey.shade200),
        errorWidget: (_, __, ___) => Container(
          color: Colors.grey.shade300,
          child: const Icon(Icons.image_not_supported),
        ),
      ),
    );
  }

  Widget _buildActions(BuildContext context, WidgetRef ref) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
      child: Row(
        children: [
          _ActionButton(
            icon: post.isLikedByMe ? Icons.favorite : Icons.favorite_border,
            color: post.isLikedByMe ? Colors.red : null,
            count: post.likesCount,
            onTap: () =>
                ref.read(feedProvider.notifier).toggleLike(post.id),
          ),
          _ActionButton(
            icon: Icons.chat_bubble_outline,
            count: post.commentsCount,
            onTap: () =>
                Navigator.pushNamed(context, '/post/${post.id}/comments'),
          ),
          _ActionButton(
            icon: Icons.share_outlined,
            count: post.sharesCount,
            onTap: () {},
          ),
          const Spacer(),
          IconButton(
            icon: Icon(
              post.isSavedByMe ? Icons.bookmark : Icons.bookmark_border,
            ),
            onPressed: () =>
                ref.read(feedProvider.notifier).toggleSave(post.id),
          ),
        ],
      ),
    );
  }

  Widget _buildCaption(BuildContext context) {
    if (post.caption == null || post.caption!.isEmpty) {
      return const SizedBox.shrink();
    }
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 4),
      child: RichText(
        text: TextSpan(
          style: DefaultTextStyle.of(context).style,
          children: _buildCaptionSpans(post.caption!, context),
        ),
      ),
    );
  }

  List<TextSpan> _buildCaptionSpans(String caption, BuildContext context) {
    final spans = <TextSpan>[];
    final words = caption.split(' ');

    for (final word in words) {
      if (word.startsWith('#') || word.startsWith('@')) {
        spans.add(TextSpan(
          text: '$word ',
          style: const TextStyle(
            color: Colors.blue,
            fontWeight: FontWeight.w500,
          ),
        ));
      } else {
        spans.add(TextSpan(text: '$word '));
      }
    }

    return spans;
  }

  Widget _buildCommentPreview(BuildContext context) {
    if (post.commentsCount == 0) return const SizedBox.shrink();
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 12),
      child: TextButton(
        onPressed: () =>
            Navigator.pushNamed(context, '/post/${post.id}/comments'),
        style: TextButton.styleFrom(
          padding: EdgeInsets.zero,
          minimumSize: Size.zero,
        ),
        child: Text(
          'View all ${post.commentsCount} comments',
          style: TextStyle(color: Colors.grey.shade600, fontSize: 14),
        ),
      ),
    );
  }

  Widget _buildTimestamp() {
    return Padding(
      padding: const EdgeInsets.only(left: 12, right: 12, bottom: 12),
      child: Text(
        timeago.format(post.createdAt),
        style: TextStyle(color: Colors.grey.shade500, fontSize: 12),
      ),
    );
  }

  void _showPostOptions(BuildContext context) {
    showModalBottomSheet(
      context: context,
      builder: (_) => Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          ListTile(
            leading: const Icon(Icons.link),
            title: const Text('Copy Link'),
            onTap: () => Navigator.pop(context),
          ),
          ListTile(
            leading: const Icon(Icons.share),
            title: const Text('Share'),
            onTap: () => Navigator.pop(context),
          ),
          ListTile(
            leading: const Icon(Icons.report_outlined),
            title: const Text('Report'),
            onTap: () => Navigator.pop(context),
          ),
        ],
      ),
    );
  }
}

class _ActionButton extends StatelessWidget {
  final IconData icon;
  final Color? color;
  final int count;
  final VoidCallback onTap;

  const _ActionButton({
    required this.icon,
    this.color,
    required this.count,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return InkWell(
      onTap: onTap,
      borderRadius: BorderRadius.circular(8),
      child: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
        child: Row(
          children: [
            Icon(icon, size: 24, color: color),
            const SizedBox(width: 4),
            Text('$count', style: TextStyle(color: Colors.grey.shade700)),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2447: User Profile Screen

```dart
// lib/screens/profile_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:cached_network_image/cached_network_image.dart';
import '../domain/models/user.dart';
import '../domain/models/post.dart';
import '../providers/social_providers.dart';

class ProfileScreen extends ConsumerWidget {
  final String userId;

  const ProfileScreen({super.key, required this.userId});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final userAsync = ref.watch(userProfileProvider(userId));
    final postsAsync = ref.watch(userPostsProvider(userId));

    return Scaffold(
      body: userAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(child: Text('Error: $e')),
        data: (user) => CustomScrollView(
          slivers: [
            _buildAppBar(context, user),
            SliverToBoxAdapter(
              child: _buildProfileInfo(context, ref, user),
            ),
            _buildPostsGrid(postsAsync),
          ],
        ),
      ),
    );
  }

  SliverAppBar _buildAppBar(BuildContext context, SocialUser user) {
    return SliverAppBar(
      expandedHeight: 200,
      pinned: true,
      flexibleSpace: FlexibleSpaceBar(
        background: user.coverUrl != null
            ? CachedNetworkImage(
                imageUrl: user.coverUrl!,
                fit: BoxFit.cover,
              )
            : Container(
                decoration: BoxDecoration(
                  gradient: LinearGradient(
                    colors: [
                      Colors.purple.shade400,
                      Colors.blue.shade400,
                    ],
                  ),
                ),
              ),
      ),
      actions: [
        IconButton(
          icon: const Icon(Icons.more_vert, color: Colors.white),
          onPressed: () {},
        ),
      ],
    );
  }

  Widget _buildProfileInfo(
      BuildContext context, WidgetRef ref, SocialUser user) {
    return Padding(
      padding: const EdgeInsets.all(16),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(
            children: [
              CircleAvatar(
                radius: 40,
                backgroundImage: user.avatarUrl != null
                    ? CachedNetworkImageProvider(user.avatarUrl!)
                    : null,
                child: user.avatarUrl == null
                    ? Text(
                        user.displayName[0],
                        style: const TextStyle(fontSize: 32),
                      )
                    : null,
              ),
              const Spacer(),
              _FollowButton(user: user),
            ],
          ),
          const SizedBox(height: 12),
          Row(
            children: [
              Text(
                user.displayName,
                style: const TextStyle(
                    fontSize: 20, fontWeight: FontWeight.bold),
              ),
              if (user.isVerified) ...[
                const SizedBox(width: 4),
                const Icon(Icons.verified, color: Colors.blue, size: 20),
              ],
            ],
          ),
          Text(
            '@${user.username}',
            style: TextStyle(color: Colors.grey.shade600),
          ),
          if (user.bio != null) ...[
            const SizedBox(height: 8),
            Text(user.bio!),
          ],
          const SizedBox(height: 16),
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceAround,
            children: [
              _StatColumn(
                count: user.postsCount,
                label: 'Posts',
              ),
              GestureDetector(
                onTap: () =>
                    Navigator.pushNamed(context, '/followers/${user.id}'),
                child: _StatColumn(
                  count: user.followersCount,
                  label: 'Followers',
                ),
              ),
              GestureDetector(
                onTap: () =>
                    Navigator.pushNamed(context, '/following/${user.id}'),
                child: _StatColumn(
                  count: user.followingCount,
                  label: 'Following',
                ),
              ),
            ],
          ),
          const SizedBox(height: 16),
          Row(
            children: [
              Expanded(
                child: OutlinedButton(
                  onPressed: () {},
                  child: const Text('Message'),
                ),
              ),
            ],
          ),
        ],
      ),
    );
  }

  Widget _buildPostsGrid(AsyncValue<List<Post>> postsAsync) {
    return postsAsync.when(
      loading: () => const SliverToBoxAdapter(
        child: Center(child: CircularProgressIndicator()),
      ),
      error: (e, _) => SliverToBoxAdapter(
        child: Center(child: Text('Error loading posts: $e')),
      ),
      data: (posts) => SliverGrid(
        delegate: SliverChildBuilderDelegate(
          (context, index) {
            final post = posts[index];
            return GestureDetector(
              onTap: () => Navigator.pushNamed(
                context,
                '/post/${post.id}',
              ),
              child: Stack(
                fit: StackFit.expand,
                children: [
                  if (post.hasMedia)
                    CachedNetworkImage(
                      imageUrl: post.media.first.url,
                      fit: BoxFit.cover,
                    )
                  else
                    Container(
                      color: Colors.grey.shade200,
                      child: const Icon(Icons.text_snippet),
                    ),
                  if (post.hasMultipleMedia)
                    const Positioned(
                      top: 8,
                      right: 8,
                      child: Icon(
                        Icons.collections,
                        color: Colors.white,
                        shadows: [Shadow(blurRadius: 4)],
                      ),
                    ),
                ],
              ),
            );
          },
          childCount: posts.length,
        ),
        gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
          crossAxisCount: 3,
          crossAxisSpacing: 2,
          mainAxisSpacing: 2,
        ),
      ),
    );
  }
}

class _FollowButton extends ConsumerStatefulWidget {
  final SocialUser user;

  const _FollowButton({required this.user});

  @override
  ConsumerState<_FollowButton> createState() => _FollowButtonState();
}

class _FollowButtonState extends ConsumerState<_FollowButton> {
  late bool _isFollowing;
  bool _isLoading = false;

  @override
  void initState() {
    super.initState();
    _isFollowing = widget.user.isFollowedByMe;
  }

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      width: 120,
      child: ElevatedButton(
        onPressed: _isLoading ? null : _toggleFollow,
        style: ElevatedButton.styleFrom(
          backgroundColor: _isFollowing ? Colors.grey : Colors.blue,
          foregroundColor: Colors.white,
        ),
        child: _isLoading
            ? const SizedBox(
                width: 20,
                height: 20,
                child: CircularProgressIndicator(
                  strokeWidth: 2,
                  color: Colors.white,
                ),
              )
            : Text(_isFollowing ? 'Following' : 'Follow'),
      ),
    );
  }

  Future<void> _toggleFollow() async {
    setState(() => _isLoading = true);
    try {
      final result = await ref
          .read(socialRepositoryProvider)
          .toggleFollow(widget.user.id);
      setState(() => _isFollowing = result);
    } finally {
      setState(() => _isLoading = false);
    }
  }
}

class _StatColumn extends StatelessWidget {
  final int count;
  final String label;

  const _StatColumn({required this.count, required this.label});

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text(
          _formatCount(count),
          style: const TextStyle(
              fontSize: 20, fontWeight: FontWeight.bold),
        ),
        Text(
          label,
          style: TextStyle(color: Colors.grey.shade600, fontSize: 13),
        ),
      ],
    );
  }

  String _formatCount(int count) {
    if (count >= 1000000) {
      return '${(count / 1000000).toStringAsFixed(1)}M';
    } else if (count >= 1000) {
      return '${(count / 1000).toStringAsFixed(1)}K';
    }
    return '$count';
  }
}
```

---

## ขั้นตอนที่ 2448: Create Post Screen

```dart
// lib/screens/create_post_screen.dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:image_picker/image_picker.dart';
import '../providers/social_providers.dart';

class CreatePostScreen extends ConsumerStatefulWidget {
  const CreatePostScreen({super.key});

  @override
  ConsumerState<CreatePostScreen> createState() =>
      _CreatePostScreenState();
}

class _CreatePostScreenState extends ConsumerState<CreatePostScreen> {
  final _captionController = TextEditingController();
  final List<File> _selectedMedia = [];
  bool _isPosting = false;
  final ImagePicker _picker = ImagePicker();

  @override
  void dispose() {
    _captionController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('New Post'),
        actions: [
          TextButton(
            onPressed: (_isPosting || (_captionController.text.isEmpty && _selectedMedia.isEmpty))
                ? null
                : _createPost,
            child: _isPosting
                ? const SizedBox(
                    width: 20,
                    height: 20,
                    child: CircularProgressIndicator(strokeWidth: 2),
                  )
                : const Text(
                    'Post',
                    style: TextStyle(fontWeight: FontWeight.bold, fontSize: 16),
                  ),
          ),
        ],
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            _buildMediaSection(),
            const SizedBox(height: 16),
            TextField(
              controller: _captionController,
              maxLines: 5,
              maxLength: 2200,
              decoration: const InputDecoration(
                hintText: 'Write a caption...',
                border: OutlineInputBorder(),
              ),
              onChanged: (_) => setState(() {}),
            ),
            const SizedBox(height: 16),
            _buildMediaButtons(),
            const SizedBox(height: 16),
            _buildTagsSection(),
          ],
        ),
      ),
    );
  }

  Widget _buildMediaSection() {
    if (_selectedMedia.isEmpty) {
      return GestureDetector(
        onTap: _pickImage,
        child: Container(
          height: 200,
          decoration: BoxDecoration(
            color: Colors.grey.shade100,
            borderRadius: BorderRadius.circular(12),
            border: Border.all(color: Colors.grey.shade300, style: BorderStyle.solid),
          ),
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Icon(Icons.add_photo_alternate,
                  size: 60, color: Colors.grey.shade500),
              const SizedBox(height: 8),
              Text(
                'Add photos or videos',
                style: TextStyle(color: Colors.grey.shade600),
              ),
            ],
          ),
        ),
      );
    }

    return SizedBox(
      height: 200,
      child: ListView.builder(
        scrollDirection: Axis.horizontal,
        itemCount: _selectedMedia.length + 1,
        itemBuilder: (context, index) {
          if (index == _selectedMedia.length) {
            return GestureDetector(
              onTap: _pickImage,
              child: Container(
                width: 150,
                margin: const EdgeInsets.only(left: 8),
                decoration: BoxDecoration(
                  color: Colors.grey.shade200,
                  borderRadius: BorderRadius.circular(8),
                ),
                child: const Icon(Icons.add, size: 40),
              ),
            );
          }

          return Stack(
            children: [
              ClipRRect(
                borderRadius: BorderRadius.circular(8),
                child: Image.file(
                  _selectedMedia[index],
                  width: 150,
                  height: 200,
                  fit: BoxFit.cover,
                ),
              ),
              Positioned(
                top: 4,
                right: 4,
                child: GestureDetector(
                  onTap: () => setState(
                      () => _selectedMedia.removeAt(index)),
                  child: Container(
                    padding: const EdgeInsets.all(4),
                    decoration: const BoxDecoration(
                      color: Colors.black54,
                      shape: BoxShape.circle,
                    ),
                    child: const Icon(Icons.close,
                        color: Colors.white, size: 16),
                  ),
                ),
              ),
            ],
          );
        },
      ),
    );
  }

  Widget _buildMediaButtons() {
    return Row(
      children: [
        _MediaButton(
          icon: Icons.photo_library,
          label: 'Gallery',
          onTap: _pickImage,
        ),
        const SizedBox(width: 16),
        _MediaButton(
          icon: Icons.camera_alt,
          label: 'Camera',
          onTap: _takePhoto,
        ),
        const SizedBox(width: 16),
        _MediaButton(
          icon: Icons.videocam,
          label: 'Video',
          onTap: _pickVideo,
        ),
      ],
    );
  }

  Widget _buildTagsSection() {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        ListTile(
          contentPadding: EdgeInsets.zero,
          leading: const Icon(Icons.location_on_outlined),
          title: const Text('Add Location'),
          trailing: const Icon(Icons.chevron_right),
          onTap: () {},
        ),
        ListTile(
          contentPadding: EdgeInsets.zero,
          leading: const Icon(Icons.person_add_outlined),
          title: const Text('Tag People'),
          trailing: const Icon(Icons.chevron_right),
          onTap: () {},
        ),
      ],
    );
  }

  Future<void> _pickImage() async {
    final files = await _picker.pickMultiImage(
      imageQuality: 85,
      maxWidth: 1080,
    );
    setState(() {
      _selectedMedia.addAll(files.map((f) => File(f.path)));
    });
  }

  Future<void> _takePhoto() async {
    final file = await _picker.pickImage(
      source: ImageSource.camera,
      imageQuality: 85,
    );
    if (file != null) {
      setState(() => _selectedMedia.add(File(file.path)));
    }
  }

  Future<void> _pickVideo() async {
    final file = await _picker.pickVideo(source: ImageSource.gallery);
    if (file != null) {
      setState(() => _selectedMedia.add(File(file.path)));
    }
  }

  Future<void> _createPost() async {
    setState(() => _isPosting = true);
    try {
      final post = await ref
          .read(socialRepositoryProvider)
          .createPost(
            caption: _captionController.text.isNotEmpty
                ? _captionController.text
                : null,
            mediaUrls: _selectedMedia.isNotEmpty
                ? _selectedMedia
                    .map((f) => 'https://picsum.photos/600/400')
                    .toList()
                : null,
          );

      ref.read(feedProvider.notifier).addPost(post);

      if (mounted) {
        Navigator.pop(context);
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Post created successfully!')),
        );
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Failed to post: $e')),
        );
      }
    } finally {
      if (mounted) setState(() => _isPosting = false);
    }
  }
}

class _MediaButton extends StatelessWidget {
  final IconData icon;
  final String label;
  final VoidCallback onTap;

  const _MediaButton({
    required this.icon,
    required this.label,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: onTap,
      child: Column(
        children: [
          Container(
            padding: const EdgeInsets.all(12),
            decoration: BoxDecoration(
              color: Colors.grey.shade100,
              borderRadius: BorderRadius.circular(8),
            ),
            child: Icon(icon),
          ),
          const SizedBox(height: 4),
          Text(label, style: const TextStyle(fontSize: 12)),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 2449: Comments Screen

```dart
// lib/screens/comments_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:cached_network_image/cached_network_image.dart';
import 'package:timeago/timeago.dart' as timeago;
import '../domain/models/post.dart';
import '../providers/social_providers.dart';

class CommentsScreen extends ConsumerStatefulWidget {
  final String postId;

  const CommentsScreen({super.key, required this.postId});

  @override
  ConsumerState<CommentsScreen> createState() => _CommentsScreenState();
}

class _CommentsScreenState extends ConsumerState<CommentsScreen> {
  final _commentController = TextEditingController();
  final _scrollController = ScrollController();
  bool _isPosting = false;

  @override
  void dispose() {
    _commentController.dispose();
    _scrollController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final commentsAsync = ref.watch(commentsProvider(widget.postId));

    return Scaffold(
      appBar: AppBar(title: const Text('Comments')),
      body: Column(
        children: [
          Expanded(
            child: commentsAsync.when(
              loading: () => const Center(child: CircularProgressIndicator()),
              error: (e, _) => Center(child: Text('Error: $e')),
              data: (comments) => comments.isEmpty
                  ? const Center(
                      child: Column(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          Icon(Icons.chat_bubble_outline,
                              size: 60, color: Colors.grey),
                          SizedBox(height: 16),
                          Text('No comments yet. Be the first!'),
                        ],
                      ),
                    )
                  : ListView.builder(
                      controller: _scrollController,
                      padding: const EdgeInsets.all(8),
                      itemCount: comments.length,
                      itemBuilder: (context, index) =>
                          CommentTile(comment: comments[index]),
                    ),
            ),
          ),
          _buildCommentInput(),
        ],
      ),
    );
  }

  Widget _buildCommentInput() {
    return Container(
      padding: EdgeInsets.only(
        left: 16,
        right: 8,
        top: 8,
        bottom: MediaQuery.of(context).viewInsets.bottom + 8,
      ),
      decoration: BoxDecoration(
        color: Colors.white,
        boxShadow: [
          BoxShadow(
            color: Colors.grey.shade200,
            blurRadius: 10,
            offset: const Offset(0, -4),
          ),
        ],
      ),
      child: Row(
        children: [
          Expanded(
            child: TextField(
              controller: _commentController,
              decoration: InputDecoration(
                hintText: 'Add a comment...',
                border: OutlineInputBorder(
                  borderRadius: BorderRadius.circular(24),
                  borderSide: BorderSide.none,
                ),
                filled: true,
                fillColor: Colors.grey.shade100,
                contentPadding: const EdgeInsets.symmetric(
                  horizontal: 16,
                  vertical: 8,
                ),
              ),
              maxLines: null,
              textInputAction: TextInputAction.send,
              onSubmitted: (_) => _postComment(),
            ),
          ),
          const SizedBox(width: 8),
          _isPosting
              ? const SizedBox(
                  width: 40,
                  height: 40,
                  child: CircularProgressIndicator(strokeWidth: 2),
                )
              : IconButton(
                  icon: const Icon(Icons.send, color: Colors.blue),
                  onPressed: _commentController.text.isEmpty
                      ? null
                      : _postComment,
                ),
        ],
      ),
    );
  }

  Future<void> _postComment() async {
    final content = _commentController.text.trim();
    if (content.isEmpty) return;

    setState(() => _isPosting = true);
    try {
      await ref
          .read(socialRepositoryProvider)
          .addComment(widget.postId, content);

      _commentController.clear();
      ref.refresh(commentsProvider(widget.postId));

      // Scroll to bottom
      await Future.delayed(const Duration(milliseconds: 300));
      if (_scrollController.hasClients) {
        _scrollController.animateTo(
          _scrollController.position.maxScrollExtent,
          duration: const Duration(milliseconds: 300),
          curve: Curves.easeOut,
        );
      }
    } finally {
      if (mounted) setState(() => _isPosting = false);
    }
  }
}

class CommentTile extends StatelessWidget {
  final Comment comment;

  const CommentTile({super.key, required this.comment});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 8),
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          CircleAvatar(
            radius: 18,
            backgroundImage: comment.author.avatarUrl != null
                ? CachedNetworkImageProvider(comment.author.avatarUrl!)
                : null,
            child: comment.author.avatarUrl == null
                ? Text(comment.author.displayName[0])
                : null,
          ),
          const SizedBox(width: 12),
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                RichText(
                  text: TextSpan(
                    style: DefaultTextStyle.of(context).style,
                    children: [
                      TextSpan(
                        text: '${comment.author.username} ',
                        style: const TextStyle(fontWeight: FontWeight.bold),
                      ),
                      TextSpan(text: comment.content),
                    ],
                  ),
                ),
                const SizedBox(height: 4),
                Row(
                  children: [
                    Text(
                      timeago.format(comment.createdAt),
                      style: TextStyle(
                          color: Colors.grey.shade500, fontSize: 12),
                    ),
                    const SizedBox(width: 16),
                    Text(
                      '${comment.likesCount} likes',
                      style: TextStyle(
                          color: Colors.grey.shade500, fontSize: 12),
                    ),
                    const SizedBox(width: 16),
                    GestureDetector(
                      onTap: () {},
                      child: Text(
                        'Reply',
                        style: TextStyle(
                          color: Colors.grey.shade500,
                          fontSize: 12,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                    ),
                  ],
                ),
              ],
            ),
          ),
          Column(
            children: [
              GestureDetector(
                onTap: () {},
                child: Icon(
                  comment.isLikedByMe
                      ? Icons.favorite
                      : Icons.favorite_border,
                  size: 16,
                  color: comment.isLikedByMe ? Colors.red : Colors.grey,
                ),
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

**← [Part 63](part-63-real-world-app-ecommerce.md)**
**ต่อไป: [Part 65 →](part-65-real-world-app-chat.md)**
