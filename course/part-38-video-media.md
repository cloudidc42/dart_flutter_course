# Part 38: Video Player & Media
## ขั้นตอนที่ 1401-1440

---

## 🎯 เป้าหมายของ Part นี้

- video_player package
- Chewie player (UI controls)
- Audio playback
- Image/video picker
- Media compression

---

## ขั้นตอนที่ 1401: Setup

```yaml
# pubspec.yaml
dependencies:
  video_player: ^2.9.0
  chewie: ^1.8.0
  just_audio: ^0.9.39
  audio_session: ^0.1.21
  image_picker: ^1.1.0
  image_cropper: ^7.0.0
  flutter_image_compress: ^2.2.0
  cached_network_image: ^3.4.0
```

---

## ขั้นตอนที่ 1402: Video Player

```dart
import 'package:flutter/material.dart';
import 'package:video_player/video_player.dart';
import 'package:chewie/chewie.dart';

// ─── Basic Video Player ───
class VideoPlayerWidget extends StatefulWidget {
  final String videoUrl;
  final bool autoPlay;
  final bool looping;

  const VideoPlayerWidget({
    super.key,
    required this.videoUrl,
    this.autoPlay = false,
    this.looping = false,
  });

  @override
  State<VideoPlayerWidget> createState() => _VideoPlayerWidgetState();
}

class _VideoPlayerWidgetState extends State<VideoPlayerWidget> {
  late VideoPlayerController _videoController;
  ChewieController? _chewieController;
  bool _isInitialized = false;
  String? _error;

  @override
  void initState() {
    super.initState();
    _initializePlayer();
  }

  Future<void> _initializePlayer() async {
    try {
      _videoController = VideoPlayerController.networkUrl(
        Uri.parse(widget.videoUrl),
      );

      await _videoController.initialize();

      _chewieController = ChewieController(
        videoPlayerController: _videoController,
        autoPlay: widget.autoPlay,
        looping: widget.looping,
        aspectRatio: _videoController.value.aspectRatio,
        // Custom controls
        showControlsOnInitialize: true,
        placeholder: Container(color: Colors.black),
        errorBuilder: (context, errorMessage) => Center(
          child: Text(errorMessage, style: const TextStyle(color: Colors.white)),
        ),
        // Subtitle support
        subtitleBuilder: (context, subtitle) => Padding(
          padding: const EdgeInsets.only(bottom: 40),
          child: Container(
            padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
            decoration: BoxDecoration(
              color: Colors.black54,
              borderRadius: BorderRadius.circular(4),
            ),
            child: Text(
              subtitle,
              style: const TextStyle(color: Colors.white, fontSize: 16),
              textAlign: TextAlign.center,
            ),
          ),
        ),
      );

      setState(() => _isInitialized = true);
    } catch (e) {
      setState(() => _error = 'ไม่สามารถโหลดวิดีโอได้: $e');
    }
  }

  @override
  Widget build(BuildContext context) {
    if (_error != null) {
      return AspectRatio(
        aspectRatio: 16 / 9,
        child: Container(
          color: Colors.black,
          child: Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                const Icon(Icons.error_outline, color: Colors.white, size: 48),
                const SizedBox(height: 8),
                Text(_error!, style: const TextStyle(color: Colors.white)),
                TextButton(
                  onPressed: () {
                    setState(() { _error = null; _isInitialized = false; });
                    _initializePlayer();
                  },
                  child: const Text('ลองใหม่'),
                ),
              ],
            ),
          ),
        ),
      );
    }

    if (!_isInitialized) {
      return const AspectRatio(
        aspectRatio: 16 / 9,
        child: ColoredBox(
          color: Colors.black,
          child: Center(child: CircularProgressIndicator(color: Colors.white)),
        ),
      );
    }

    return AspectRatio(
      aspectRatio: _videoController.value.aspectRatio,
      child: Chewie(controller: _chewieController!),
    );
  }

  @override
  void dispose() {
    _chewieController?.dispose();
    _videoController.dispose();
    super.dispose();
  }
}

// ─── Video List Player ───
class VideoFeedPage extends StatefulWidget {
  const VideoFeedPage({super.key});

  @override
  State<VideoFeedPage> createState() => _VideoFeedPageState();
}

class _VideoFeedPageState extends State<VideoFeedPage> {
  final PageController _pageController = PageController();
  int _currentIndex = 0;

  static const List<Map<String, String>> _videos = [
    {
      'url': 'https://flutter.github.io/assets-for-api-docs/assets/videos/bee.mp4',
      'title': 'Bee Video',
      'description': 'A bee collecting nectar',
    },
    {
      'url': 'https://flutter.github.io/assets-for-api-docs/assets/videos/butterfly.mp4',
      'title': 'Butterfly',
      'description': 'Beautiful butterfly',
    },
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.black,
      body: PageView.builder(
        scrollDirection: Axis.vertical,
        controller: _pageController,
        itemCount: _videos.length,
        onPageChanged: (index) => setState(() => _currentIndex = index),
        itemBuilder: (context, index) {
          bool isActive = index == _currentIndex;
          return _VideoFeedItem(
            video: _videos[index],
            isActive: isActive,
          );
        },
      ),
    );
  }

  @override
  void dispose() {
    _pageController.dispose();
    super.dispose();
  }
}

class _VideoFeedItem extends StatefulWidget {
  final Map<String, String> video;
  final bool isActive;

  const _VideoFeedItem({required this.video, required this.isActive});

  @override
  State<_VideoFeedItem> createState() => _VideoFeedItemState();
}

class _VideoFeedItemState extends State<_VideoFeedItem> {
  late VideoPlayerController _controller;
  bool _initialized = false;

  @override
  void initState() {
    super.initState();
    _init();
  }

  Future<void> _init() async {
    _controller = VideoPlayerController.networkUrl(
      Uri.parse(widget.video['url']!),
    );
    await _controller.initialize();
    _controller.setLooping(true);
    setState(() => _initialized = true);
    if (widget.isActive) _controller.play();
  }

  @override
  void didUpdateWidget(_VideoFeedItem oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.isActive && _initialized) {
      _controller.play();
    } else {
      _controller.pause();
    }
  }

  @override
  Widget build(BuildContext context) {
    return Stack(
      fit: StackFit.expand,
      children: [
        // Video
        if (_initialized)
          FittedBox(
            fit: BoxFit.cover,
            child: SizedBox(
              width: _controller.value.size.width,
              height: _controller.value.size.height,
              child: VideoPlayer(_controller),
            ),
          )
        else
          const Center(child: CircularProgressIndicator(color: Colors.white)),

        // Overlay
        Positioned(
          bottom: 80,
          left: 16,
          right: 60,
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                widget.video['title']!,
                style: const TextStyle(
                  color: Colors.white,
                  fontSize: 20,
                  fontWeight: FontWeight.bold,
                ),
              ),
              Text(
                widget.video['description']!,
                style: const TextStyle(color: Colors.white70),
              ),
            ],
          ),
        ),
      ],
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

---

## ขั้นตอนที่ 1403: Audio Player

```dart
import 'package:flutter/material.dart';
import 'package:just_audio/just_audio.dart';

class AudioPlayerWidget extends StatefulWidget {
  final String audioUrl;
  final String title;
  final String artist;
  final String? albumArtUrl;

  const AudioPlayerWidget({
    super.key,
    required this.audioUrl,
    required this.title,
    required this.artist,
    this.albumArtUrl,
  });

  @override
  State<AudioPlayerWidget> createState() => _AudioPlayerWidgetState();
}

class _AudioPlayerWidgetState extends State<AudioPlayerWidget> {
  final AudioPlayer _player = AudioPlayer();

  @override
  void initState() {
    super.initState();
    _initPlayer();
  }

  Future<void> _initPlayer() async {
    await _player.setUrl(widget.audioUrl);
  }

  String _formatDuration(Duration? d) {
    if (d == null) return '0:00';
    String minutes = d.inMinutes.toString();
    String seconds = (d.inSeconds % 60).toString().padLeft(2, '0');
    return '$minutes:$seconds';
  }

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            // Album art
            if (widget.albumArtUrl != null)
              ClipRRect(
                borderRadius: BorderRadius.circular(8),
                child: Image.network(
                  widget.albumArtUrl!,
                  width: 200,
                  height: 200,
                  fit: BoxFit.cover,
                ),
              )
            else
              Container(
                width: 200,
                height: 200,
                decoration: BoxDecoration(
                  color: Colors.grey[300],
                  borderRadius: BorderRadius.circular(8),
                ),
                child: const Icon(Icons.music_note, size: 80),
              ),
            const SizedBox(height: 16),

            // Title & Artist
            Text(widget.title, style: const TextStyle(fontSize: 20, fontWeight: FontWeight.bold)),
            Text(widget.artist, style: TextStyle(color: Colors.grey[600])),
            const SizedBox(height: 16),

            // Progress slider
            StreamBuilder<Duration>(
              stream: _player.positionStream,
              builder: (context, positionSnapshot) {
                return StreamBuilder<Duration?>(
                  stream: _player.durationStream,
                  builder: (context, durationSnapshot) {
                    Duration position = positionSnapshot.data ?? Duration.zero;
                    Duration duration = durationSnapshot.data ?? Duration.zero;
                    double progress = duration.inMilliseconds > 0
                        ? position.inMilliseconds / duration.inMilliseconds
                        : 0;

                    return Column(
                      children: [
                        Slider(
                          value: progress.clamp(0, 1),
                          onChanged: (value) {
                            Duration seekTo = duration * value;
                            _player.seek(seekTo);
                          },
                        ),
                        Row(
                          mainAxisAlignment: MainAxisAlignment.spaceBetween,
                          children: [
                            Text(_formatDuration(position)),
                            Text(_formatDuration(duration)),
                          ],
                        ),
                      ],
                    );
                  },
                );
              },
            ),

            // Controls
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                IconButton(
                  icon: const Icon(Icons.skip_previous, size: 36),
                  onPressed: () => _player.seek(Duration.zero),
                ),
                StreamBuilder<PlayerState>(
                  stream: _player.playerStateStream,
                  builder: (context, snapshot) {
                    PlayerState? state = snapshot.data;
                    bool isPlaying = state?.playing ?? false;
                    bool isLoading = state?.processingState == ProcessingState.loading ||
                        state?.processingState == ProcessingState.buffering;

                    return isLoading
                        ? const CircularProgressIndicator()
                        : IconButton(
                            icon: Icon(
                              isPlaying ? Icons.pause_circle : Icons.play_circle,
                              size: 64,
                            ),
                            onPressed: isPlaying ? _player.pause : _player.play,
                          );
                  },
                ),
                IconButton(
                  icon: const Icon(Icons.skip_next, size: 36),
                  onPressed: () => _player.seek(_player.duration ?? Duration.zero),
                ),
              ],
            ),

            // Volume
            StreamBuilder<double>(
              stream: _player.volumeStream,
              builder: (context, snapshot) {
                double volume = snapshot.data ?? 1.0;
                return Row(
                  children: [
                    const Icon(Icons.volume_down),
                    Expanded(
                      child: Slider(
                        value: volume,
                        onChanged: _player.setVolume,
                      ),
                    ),
                    const Icon(Icons.volume_up),
                  ],
                );
              },
            ),
          ],
        ),
      ),
    );
  }

  @override
  void dispose() {
    _player.dispose();
    super.dispose();
  }
}
```

---

## ขั้นตอนที่ 1404: Image/Video Picker

```dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:image_picker/image_picker.dart';
import 'package:image_cropper/image_cropper.dart';
import 'package:flutter_image_compress/flutter_image_compress.dart';

class MediaPickerService {
  final ImagePicker _picker = ImagePicker();

  // ─── Pick Image ───
  Future<File?> pickImage({
    required ImageSource source,
    int? maxWidth,
    int? maxHeight,
    int imageQuality = 85,
  }) async {
    XFile? picked = await _picker.pickImage(
      source: source,
      maxWidth: maxWidth?.toDouble(),
      maxHeight: maxHeight?.toDouble(),
      imageQuality: imageQuality,
    );

    return picked != null ? File(picked.path) : null;
  }

  // ─── Pick Multiple Images ───
  Future<List<File>> pickMultipleImages() async {
    List<XFile> picked = await _picker.pickMultiImage(imageQuality: 85);
    return picked.map((f) => File(f.path)).toList();
  }

  // ─── Pick Video ───
  Future<File?> pickVideo({ImageSource source = ImageSource.gallery}) async {
    XFile? picked = await _picker.pickVideo(
      source: source,
      maxDuration: const Duration(minutes: 5),
    );

    return picked != null ? File(picked.path) : null;
  }

  // ─── Crop Image ───
  Future<File?> cropImage(File imageFile) async {
    CroppedFile? cropped = await ImageCropper().cropImage(
      sourcePath: imageFile.path,
      uiSettings: [
        AndroidUiSettings(
          toolbarTitle: 'Crop Image',
          toolbarColor: Colors.blue,
          toolbarWidgetColor: Colors.white,
          lockAspectRatio: false,
        ),
        IOSUiSettings(title: 'Crop Image'),
      ],
    );

    return cropped != null ? File(cropped.path) : null;
  }

  // ─── Compress Image ───
  Future<File?> compressImage(File file, {int quality = 70}) async {
    String outputPath = '${file.parent.path}/compressed_${file.uri.pathSegments.last}';

    XFile? result = await FlutterImageCompress.compressAndGetFile(
      file.absolute.path,
      outputPath,
      quality: quality,
      format: CompressFormat.jpeg,
    );

    return result != null ? File(result.path) : null;
  }
}

// ─── Profile Photo Picker Widget ───
class ProfilePhotoPicker extends StatefulWidget {
  final Function(File) onPhotoSelected;
  const ProfilePhotoPicker({super.key, required this.onPhotoSelected});

  @override
  State<ProfilePhotoPicker> createState() => _ProfilePhotoPickerState();
}

class _ProfilePhotoPickerState extends State<ProfilePhotoPicker> {
  File? _selectedImage;
  final MediaPickerService _service = MediaPickerService();

  Future<void> _pickImage(ImageSource source) async {
    File? image = await _service.pickImage(source: source, maxWidth: 512, maxHeight: 512);
    if (image == null) return;

    File? cropped = await _service.cropImage(image);
    if (cropped == null) return;

    File? compressed = await _service.compressImage(cropped);
    File finalImage = compressed ?? cropped;

    setState(() => _selectedImage = finalImage);
    widget.onPhotoSelected(finalImage);
  }

  void _showPickerModal() {
    showModalBottomSheet(
      context: context,
      builder: (context) => SafeArea(
        child: Wrap(
          children: [
            ListTile(
              leading: const Icon(Icons.camera_alt),
              title: const Text('ถ่ายรูป'),
              onTap: () {
                Navigator.pop(context);
                _pickImage(ImageSource.camera);
              },
            ),
            ListTile(
              leading: const Icon(Icons.photo_library),
              title: const Text('เลือกจากคลัง'),
              onTap: () {
                Navigator.pop(context);
                _pickImage(ImageSource.gallery);
              },
            ),
            if (_selectedImage != null)
              ListTile(
                leading: const Icon(Icons.delete, color: Colors.red),
                title: const Text('ลบรูป', style: TextStyle(color: Colors.red)),
                onTap: () {
                  Navigator.pop(context);
                  setState(() => _selectedImage = null);
                },
              ),
          ],
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _showPickerModal,
      child: Stack(
        children: [
          CircleAvatar(
            radius: 50,
            backgroundImage: _selectedImage != null
                ? FileImage(_selectedImage!)
                : null,
            child: _selectedImage == null
                ? const Icon(Icons.person, size: 50)
                : null,
          ),
          Positioned(
            bottom: 0,
            right: 0,
            child: Container(
              decoration: BoxDecoration(
                color: Theme.of(context).colorScheme.primary,
                shape: BoxShape.circle,
              ),
              child: const Icon(Icons.camera_alt, color: Colors.white, size: 20),
            ),
          ),
        ],
      ),
    );
  }
}
```

---

**← [Part 37 - Maps & Location](part-37-maps-location.md)**

**ต่อไป: [Part 39 - WebSocket & Real-time →](part-39-websocket-realtime.md)**
