# Part 39: WebSocket & Real-time
## ขั้นตอนที่ 1441-1480

---

## 🎯 เป้าหมายของ Part นี้

- WebSocket connection
- Real-time chat app
- Live data streaming
- Socket.IO
- Connection management & reconnect

---

## ขั้นตอนที่ 1441: WebSocket Service

```dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'package:flutter/material.dart';

// ─── WebSocket Service ───
class WebSocketService {
  static final WebSocketService _instance = WebSocketService._();
  factory WebSocketService() => _instance;
  WebSocketService._();

  WebSocket? _socket;
  bool _isConnected = false;
  final _messageController = StreamController<Map<String, dynamic>>.broadcast();
  final _connectionController = StreamController<bool>.broadcast();

  Stream<Map<String, dynamic>> get messages => _messageController.stream;
  Stream<bool> get connectionState => _connectionController.stream;
  bool get isConnected => _isConnected;

  Timer? _reconnectTimer;
  int _reconnectAttempts = 0;
  static const int _maxReconnectAttempts = 5;

  Future<void> connect(String url) async {
    try {
      _socket = await WebSocket.connect(url);
      _isConnected = true;
      _reconnectAttempts = 0;
      _connectionController.add(true);

      _socket!.listen(
        (data) {
          try {
            Map<String, dynamic> message = jsonDecode(data);
            _messageController.add(message);
          } catch (e) {
            print('Parse error: $e');
          }
        },
        onError: (error) {
          print('WebSocket error: $error');
          _onDisconnected(url);
        },
        onDone: () {
          print('WebSocket closed');
          _onDisconnected(url);
        },
      );
    } catch (e) {
      print('Connection failed: $e');
      _onDisconnected(url);
    }
  }

  void _onDisconnected(String url) {
    _isConnected = false;
    _connectionController.add(false);

    // Auto reconnect
    if (_reconnectAttempts < _maxReconnectAttempts) {
      _reconnectAttempts++;
      Duration delay = Duration(seconds: _reconnectAttempts * 2);
      print('Reconnecting in ${delay.inSeconds}s (attempt $_reconnectAttempts)');
      _reconnectTimer = Timer(delay, () => connect(url));
    }
  }

  void send(Map<String, dynamic> data) {
    if (_isConnected && _socket != null) {
      _socket!.add(jsonEncode(data));
    }
  }

  Future<void> disconnect() async {
    _reconnectTimer?.cancel();
    await _socket?.close();
    _isConnected = false;
    _connectionController.add(false);
  }

  void dispose() {
    disconnect();
    _messageController.close();
    _connectionController.close();
  }
}
```

---

## ขั้นตอนที่ 1442: Real-time Chat

```dart
import 'dart:async';
import 'dart:convert';
import 'package:flutter/material.dart';

// ─── Models ───
class ChatMessage {
  final String id;
  final String content;
  final String senderId;
  final String senderName;
  final DateTime timestamp;
  final MessageType type;

  ChatMessage({
    required this.id,
    required this.content,
    required this.senderId,
    required this.senderName,
    required this.timestamp,
    this.type = MessageType.text,
  });

  factory ChatMessage.fromJson(Map<String, dynamic> json) => ChatMessage(
    id: json['id'],
    content: json['content'],
    senderId: json['senderId'],
    senderName: json['senderName'],
    timestamp: DateTime.parse(json['timestamp']),
    type: MessageType.values.firstWhere(
      (t) => t.name == json['type'],
      orElse: () => MessageType.text,
    ),
  );

  Map<String, dynamic> toJson() => {
    'id': id,
    'content': content,
    'senderId': senderId,
    'senderName': senderName,
    'timestamp': timestamp.toIso8601String(),
    'type': type.name,
  };
}

enum MessageType { text, image, system }

// ─── Chat Repository ───
class ChatRepository {
  final WebSocketService _wsService = WebSocketService();
  final List<ChatMessage> _messages = [];
  final _messagesController = StreamController<List<ChatMessage>>.broadcast();
  String? _currentUserId;
  String? _currentUserName;

  Stream<List<ChatMessage>> get messagesStream => _messagesController.stream;
  List<ChatMessage> get messages => List.unmodifiable(_messages);
  bool get isConnected => _wsService.isConnected;
  Stream<bool> get connectionState => _wsService.connectionState;

  Future<void> connect({
    required String serverUrl,
    required String userId,
    required String userName,
  }) async {
    _currentUserId = userId;
    _currentUserName = userName;

    await _wsService.connect(serverUrl);

    // Listen for messages
    _wsService.messages.listen((data) {
      String type = data['type'] ?? '';
      switch (type) {
        case 'message':
          ChatMessage msg = ChatMessage.fromJson(data['message']);
          _messages.add(msg);
          _messagesController.add(_messages.toList());
          break;
        case 'history':
          List messages = data['messages'] ?? [];
          _messages.addAll(messages.map((m) => ChatMessage.fromJson(m)));
          _messagesController.add(_messages.toList());
          break;
        case 'typing':
          // Handle typing indicator
          break;
      }
    });

    // Send join event
    _wsService.send({
      'type': 'join',
      'userId': userId,
      'userName': userName,
    });
  }

  void sendMessage(String content) {
    if (_currentUserId == null) return;

    ChatMessage message = ChatMessage(
      id: DateTime.now().millisecondsSinceEpoch.toString(),
      content: content,
      senderId: _currentUserId!,
      senderName: _currentUserName!,
      timestamp: DateTime.now(),
    );

    _wsService.send({
      'type': 'message',
      'message': message.toJson(),
    });

    // Optimistically add to local list
    _messages.add(message);
    _messagesController.add(_messages.toList());
  }

  void sendTyping(bool isTyping) {
    _wsService.send({
      'type': 'typing',
      'userId': _currentUserId,
      'isTyping': isTyping,
    });
  }

  void disconnect() {
    _wsService.send({'type': 'leave', 'userId': _currentUserId});
    _wsService.disconnect();
  }

  void dispose() {
    disconnect();
    _messagesController.close();
  }
}

// ─── Chat Screen ───
class ChatScreen extends StatefulWidget {
  final String roomName;
  final String userId;
  final String userName;

  const ChatScreen({
    super.key,
    required this.roomName,
    required this.userId,
    required this.userName,
  });

  @override
  State<ChatScreen> createState() => _ChatScreenState();
}

class _ChatScreenState extends State<ChatScreen> {
  final ChatRepository _chat = ChatRepository();
  final TextEditingController _inputController = TextEditingController();
  final ScrollController _scrollController = ScrollController();
  bool _isTyping = false;
  Timer? _typingTimer;

  @override
  void initState() {
    super.initState();
    _chat.connect(
      serverUrl: 'wss://your-chat-server.com/ws',
      userId: widget.userId,
      userName: widget.userName,
    );
  }

  void _sendMessage() {
    String text = _inputController.text.trim();
    if (text.isEmpty) return;

    _chat.sendMessage(text);
    _inputController.clear();
    _scrollToBottom();
  }

  void _onTypingChanged(String text) {
    if (!_isTyping && text.isNotEmpty) {
      _isTyping = true;
      _chat.sendTyping(true);
    }

    _typingTimer?.cancel();
    _typingTimer = Timer(const Duration(seconds: 2), () {
      _isTyping = false;
      _chat.sendTyping(false);
    });
  }

  void _scrollToBottom() {
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (_scrollController.hasClients) {
        _scrollController.animateTo(
          _scrollController.position.maxScrollExtent,
          duration: const Duration(milliseconds: 300),
          curve: Curves.easeOut,
        );
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(widget.roomName),
            StreamBuilder<bool>(
              stream: _chat.connectionState,
              builder: (context, snapshot) {
                bool isConnected = snapshot.data ?? false;
                return Text(
                  isConnected ? 'Online' : 'Connecting...',
                  style: TextStyle(
                    fontSize: 12,
                    color: isConnected ? Colors.greenAccent : Colors.orange,
                  ),
                );
              },
            ),
          ],
        ),
        actions: [
          StreamBuilder<bool>(
            stream: _chat.connectionState,
            builder: (context, snapshot) {
              return Icon(
                Icons.circle,
                color: (snapshot.data ?? false) ? Colors.green : Colors.grey,
                size: 12,
              );
            },
          ),
          const SizedBox(width: 12),
        ],
      ),
      body: Column(
        children: [
          // Messages list
          Expanded(
            child: StreamBuilder<List<ChatMessage>>(
              stream: _chat.messagesStream,
              initialData: _chat.messages,
              builder: (context, snapshot) {
                List<ChatMessage> messages = snapshot.data ?? [];

                return ListView.builder(
                  controller: _scrollController,
                  padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
                  itemCount: messages.length,
                  itemBuilder: (context, index) {
                    ChatMessage msg = messages[index];
                    bool isMe = msg.senderId == widget.userId;

                    if (msg.type == MessageType.system) {
                      return Center(
                        child: Padding(
                          padding: const EdgeInsets.symmetric(vertical: 4),
                          child: Text(
                            msg.content,
                            style: TextStyle(color: Colors.grey[600], fontSize: 12),
                          ),
                        ),
                      );
                    }

                    return _MessageBubble(message: msg, isMe: isMe);
                  },
                );
              },
            ),
          ),
          // Input area
          Container(
            padding: const EdgeInsets.all(8),
            decoration: BoxDecoration(
              color: Theme.of(context).colorScheme.surface,
              border: Border(top: BorderSide(color: Theme.of(context).dividerColor)),
            ),
            child: SafeArea(
              child: Row(
                children: [
                  Expanded(
                    child: TextField(
                      controller: _inputController,
                      onChanged: _onTypingChanged,
                      maxLines: null,
                      decoration: InputDecoration(
                        hintText: 'พิมพ์ข้อความ...',
                        border: OutlineInputBorder(
                          borderRadius: BorderRadius.circular(24),
                        ),
                        contentPadding: const EdgeInsets.symmetric(
                          horizontal: 16,
                          vertical: 10,
                        ),
                        isDense: true,
                      ),
                      textInputAction: TextInputAction.send,
                      onSubmitted: (_) => _sendMessage(),
                    ),
                  ),
                  const SizedBox(width: 8),
                  CircleAvatar(
                    child: IconButton(
                      icon: const Icon(Icons.send),
                      onPressed: _sendMessage,
                    ),
                  ),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }

  @override
  void dispose() {
    _typingTimer?.cancel();
    _inputController.dispose();
    _scrollController.dispose();
    _chat.dispose();
    super.dispose();
  }
}

// ─── Message Bubble ───
class _MessageBubble extends StatelessWidget {
  final ChatMessage message;
  final bool isMe;

  const _MessageBubble({required this.message, required this.isMe});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 4),
      child: Row(
        mainAxisAlignment: isMe ? MainAxisAlignment.end : MainAxisAlignment.start,
        crossAxisAlignment: CrossAxisAlignment.end,
        children: [
          if (!isMe) ...[
            CircleAvatar(
              radius: 16,
              child: Text(message.senderName[0].toUpperCase()),
            ),
            const SizedBox(width: 8),
          ],
          Flexible(
            child: Column(
              crossAxisAlignment: isMe ? CrossAxisAlignment.end : CrossAxisAlignment.start,
              children: [
                if (!isMe)
                  Padding(
                    padding: const EdgeInsets.only(left: 4, bottom: 2),
                    child: Text(
                      message.senderName,
                      style: TextStyle(fontSize: 11, color: Colors.grey[600]),
                    ),
                  ),
                Container(
                  constraints: BoxConstraints(
                    maxWidth: MediaQuery.of(context).size.width * 0.7,
                  ),
                  padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 10),
                  decoration: BoxDecoration(
                    color: isMe
                        ? Theme.of(context).colorScheme.primary
                        : Theme.of(context).colorScheme.surfaceContainerHighest,
                    borderRadius: BorderRadius.only(
                      topLeft: const Radius.circular(16),
                      topRight: const Radius.circular(16),
                      bottomLeft: isMe ? const Radius.circular(16) : const Radius.circular(4),
                      bottomRight: isMe ? const Radius.circular(4) : const Radius.circular(16),
                    ),
                  ),
                  child: Text(
                    message.content,
                    style: TextStyle(
                      color: isMe
                          ? Theme.of(context).colorScheme.onPrimary
                          : Theme.of(context).colorScheme.onSurface,
                    ),
                  ),
                ),
                Padding(
                  padding: const EdgeInsets.only(top: 2, left: 4, right: 4),
                  child: Text(
                    '${message.timestamp.hour}:${message.timestamp.minute.toString().padLeft(2, '0')}',
                    style: TextStyle(fontSize: 10, color: Colors.grey[500]),
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

## ขั้นตอนที่ 1443: Live Updates - Stock Ticker

```dart
import 'dart:async';
import 'dart:math';
import 'package:flutter/material.dart';

// ─── Mock Real-time Stock Data ───
class StockTicker {
  final String symbol;
  double price;
  double change;
  double changePercent;

  StockTicker({
    required this.symbol,
    required this.price,
    required this.change,
    required this.changePercent,
  });
}

class StockService {
  final _random = Random();
  final _stocksController = StreamController<List<StockTicker>>.broadcast();
  Timer? _timer;

  Stream<List<StockTicker>> get stocks => _stocksController.stream;

  final List<StockTicker> _tickers = [
    StockTicker(symbol: 'AAPL', price: 193.50, change: 2.30, changePercent: 1.20),
    StockTicker(symbol: 'GOOGL', price: 141.20, change: -0.80, changePercent: -0.56),
    StockTicker(symbol: 'MSFT', price: 420.15, change: 5.40, changePercent: 1.30),
    StockTicker(symbol: 'AMZN', price: 185.60, change: -1.20, changePercent: -0.64),
    StockTicker(symbol: 'TSLA', price: 248.90, change: 8.70, changePercent: 3.62),
  ];

  void startUpdates() {
    _timer = Timer.periodic(const Duration(seconds: 2), (_) {
      for (StockTicker ticker in _tickers) {
        double delta = (_random.nextDouble() - 0.5) * 2;
        ticker.change = delta;
        ticker.changePercent = delta / ticker.price * 100;
        ticker.price += delta;
      }
      _stocksController.add(_tickers.toList());
    });
  }

  void stop() {
    _timer?.cancel();
    _stocksController.close();
  }
}

class StockTickerScreen extends StatefulWidget {
  const StockTickerScreen({super.key});

  @override
  State<StockTickerScreen> createState() => _StockTickerScreenState();
}

class _StockTickerScreenState extends State<StockTickerScreen> {
  final StockService _service = StockService();

  @override
  void initState() {
    super.initState();
    _service.startUpdates();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Live Stocks'),
        actions: [
          Container(
            margin: const EdgeInsets.all(8),
            padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
            decoration: BoxDecoration(
              color: Colors.green.withOpacity(0.2),
              borderRadius: BorderRadius.circular(12),
            ),
            child: Row(
              children: [
                Container(width: 8, height: 8, decoration: const BoxDecoration(
                  color: Colors.green,
                  shape: BoxShape.circle,
                )),
                const SizedBox(width: 4),
                const Text('LIVE', style: TextStyle(fontSize: 12, color: Colors.green)),
              ],
            ),
          ),
        ],
      ),
      body: StreamBuilder<List<StockTicker>>(
        stream: _service.stocks,
        builder: (context, snapshot) {
          if (!snapshot.hasData) {
            return const Center(child: CircularProgressIndicator());
          }

          return ListView.builder(
            itemCount: snapshot.data!.length,
            itemBuilder: (context, index) {
              StockTicker ticker = snapshot.data![index];
              bool isUp = ticker.change >= 0;

              return AnimatedContainer(
                duration: const Duration(milliseconds: 300),
                child: ListTile(
                  leading: CircleAvatar(
                    backgroundColor: Colors.grey[200],
                    child: Text(
                      ticker.symbol[0],
                      style: const TextStyle(fontWeight: FontWeight.bold),
                    ),
                  ),
                  title: Text(ticker.symbol, style: const TextStyle(fontWeight: FontWeight.bold)),
                  subtitle: Text('฿${ticker.price.toStringAsFixed(2)}'),
                  trailing: Container(
                    padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
                    decoration: BoxDecoration(
                      color: (isUp ? Colors.green : Colors.red).withOpacity(0.1),
                      borderRadius: BorderRadius.circular(8),
                    ),
                    child: Column(
                      mainAxisSize: MainAxisSize.min,
                      crossAxisAlignment: CrossAxisAlignment.end,
                      children: [
                        Text(
                          '${isUp ? '+' : ''}${ticker.change.toStringAsFixed(2)}',
                          style: TextStyle(
                            color: isUp ? Colors.green : Colors.red,
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                        Text(
                          '${isUp ? '+' : ''}${ticker.changePercent.toStringAsFixed(2)}%',
                          style: TextStyle(
                            color: isUp ? Colors.green : Colors.red,
                            fontSize: 12,
                          ),
                        ),
                      ],
                    ),
                  ),
                ),
              );
            },
          );
        },
      ),
    );
  }

  @override
  void dispose() {
    _service.stop();
    super.dispose();
  }
}
```

---

**← [Part 38 - Video Player & Media](part-38-video-media.md)**

**ต่อไป: [Part 40 - Biometric & Security →](part-40-biometric-security.md)**
