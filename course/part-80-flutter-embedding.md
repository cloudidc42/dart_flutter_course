# Part 80: Flutter Embedding (Add-to-App)
## ขั้นตอนที่ 3081-3120

## 🎯 เป้าหมายของ Part นี้
- Embed Flutter ใน Native Android ด้วย FlutterActivity/FlutterFragment
- Embed Flutter ใน Native iOS ด้วย FlutterViewController
- สื่อสารระหว่าง Flutter และ Native ด้วย MethodChannel/EventChannel
- รัน Multiple Flutter Engines
- Best practices สำหรับ Add-to-App

---

## ขั้นตอนที่ 3081: Add-to-App Overview

```
Flutter Add-to-App Architecture:
┌─────────────────────────────────────────┐
│          Native App (Host)               │
│  ┌─────────────────────────────────┐    │
│  │   Native Screen (Activity/VC)    │    │
│  │                                  │    │
│  │  ┌────────────────────────────┐  │    │
│  │  │   Flutter Fragment/View    │  │    │
│  │  │                            │  │    │
│  │  │   Flutter Engine           │  │    │
│  │  │   ┌──────────────────────┐ │  │    │
│  │  │   │  Flutter Widget Tree  │ │  │    │
│  │  │   └──────────────────────┘ │  │    │
│  │  └────────────────────────────┘  │    │
│  └─────────────────────────────────┘    │
│                                         │
│  MethodChannel ←→ MethodChannel         │
│  EventChannel  ←→ EventChannel          │
└─────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 3082: Flutter Module Setup

```yaml
# flutter_module/pubspec.yaml
name: flutter_module
description: Flutter module for embedding in native apps
publish_to: 'none'

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: ">=3.10.0"

dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.6

flutter:
  module:
    androidX: true
    androidPackage: com.example.flutter_module
    iosBundleIdentifier: com.example.flutterModule

  uses-material-design: true
```

---

## ขั้นตอนที่ 3083: Flutter Module - Main Entry Point

```dart
// flutter_module/lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

// Entry point หลัก
@pragma('vm:entry-point')
void main() => runApp(const FlutterModuleApp());

// Entry point สำหรับ product catalog feature
@pragma('vm:entry-point')
void productCatalog() => runApp(const ProductCatalogApp());

// Entry point สำหรับ checkout feature
@pragma('vm:entry-point')
void checkout() => runApp(const CheckoutApp());

class FlutterModuleApp extends StatelessWidget {
  const FlutterModuleApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Module',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      home: const MainFlutterScreen(),
    );
  }
}

class MainFlutterScreen extends StatefulWidget {
  const MainFlutterScreen({super.key});

  @override
  State<MainFlutterScreen> createState() => _MainFlutterScreenState();
}

class _MainFlutterScreenState extends State<MainFlutterScreen> {
  // MethodChannel สำหรับ communicate กับ native
  static const MethodChannel _methodChannel =
      MethodChannel('com.example.flutter_module/main');

  // EventChannel สำหรับรับ events จาก native
  static const EventChannel _eventChannel =
      EventChannel('com.example.flutter_module/events');

  String _nativeMessage = 'Waiting for native message...';
  String _nativeData = '';
  int _nativeCounter = 0;

  @override
  void initState() {
    super.initState();
    _setupChannels();
  }

  void _setupChannels() {
    // ฟัง method calls จาก native
    _methodChannel.setMethodCallHandler(_handleMethodCall);

    // Subscribe to native events
    _eventChannel
        .receiveBroadcastStream()
        .listen(_handleNativeEvent, onError: _handleEventError);
  }

  Future<dynamic> _handleMethodCall(MethodCall call) async {
    switch (call.method) {
      case 'updateMessage':
        setState(() {
          _nativeMessage = call.arguments as String;
        });
        return 'Message received: ${call.arguments}';

      case 'updateData':
        final data = Map<String, dynamic>.from(call.arguments as Map);
        setState(() {
          _nativeData = data.entries
              .map((e) => '${e.key}: ${e.value}')
              .join('\n');
        });
        return true;

      default:
        throw PlatformException(
          code: 'NOT_IMPLEMENTED',
          message: 'Method ${call.method} not implemented',
        );
    }
  }

  void _handleNativeEvent(dynamic event) {
    if (event is Map) {
      final eventType = event['type'] as String?;
      final eventData = event['data'];
      setState(() {
        _nativeCounter++;
        _nativeMessage = 'Event #$_nativeCounter: $eventType - $eventData';
      });
    }
  }

  void _handleEventError(dynamic error) {
    setState(() {
      _nativeMessage = 'Event error: $error';
    });
  }

  Future<void> _callNativeMethod(String methodName,
      [dynamic arguments]) async {
    try {
      final result = await _methodChannel.invokeMethod(methodName, arguments);
      setState(() {
        _nativeMessage = 'Native returned: $result';
      });
    } on PlatformException catch (e) {
      setState(() {
        _nativeMessage = 'Platform error: ${e.message}';
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: const Color(0xFFF5F7FA),
      appBar: AppBar(
        title: const Text('Flutter Module'),
        backgroundColor: Colors.blue,
        foregroundColor: Colors.white,
        actions: [
          IconButton(
            icon: const Icon(Icons.close),
            onPressed: () {
              // บอก native ให้ dismiss flutter
              _callNativeMethod('closeFlutter');
            },
          ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    const Text(
                      'Flutter → Native',
                      style: TextStyle(
                        fontSize: 16,
                        fontWeight: FontWeight.bold,
                        color: Colors.blue,
                      ),
                    ),
                    const SizedBox(height: 12),
                    ElevatedButton(
                      onPressed: () =>
                          _callNativeMethod('showNativeToast', 'Hello from Flutter!'),
                      child: const Text('Show Native Toast'),
                    ),
                    const SizedBox(height: 8),
                    ElevatedButton(
                      onPressed: () =>
                          _callNativeMethod('openNativeCamera'),
                      child: const Text('Open Native Camera'),
                    ),
                    const SizedBox(height: 8),
                    ElevatedButton(
                      onPressed: () => _callNativeMethod('getBatteryLevel'),
                      child: const Text('Get Battery Level'),
                    ),
                    const SizedBox(height: 8),
                    ElevatedButton(
                      onPressed: () => _callNativeMethod('getUserData'),
                      child: const Text('Get Native User Data'),
                    ),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 16),
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    const Text(
                      'Native → Flutter',
                      style: TextStyle(
                        fontSize: 16,
                        fontWeight: FontWeight.bold,
                        color: Colors.green,
                      ),
                    ),
                    const SizedBox(height: 12),
                    Container(
                      padding: const EdgeInsets.all(12),
                      decoration: BoxDecoration(
                        color: Colors.grey.shade100,
                        borderRadius: BorderRadius.circular(8),
                        border: Border.all(color: Colors.grey.shade300),
                      ),
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Text(
                            _nativeMessage,
                            style: const TextStyle(fontSize: 14),
                          ),
                          if (_nativeData.isNotEmpty) ...[
                            const Divider(),
                            Text(
                              _nativeData,
                              style: TextStyle(
                                fontSize: 12,
                                color: Colors.grey.shade700,
                              ),
                            ),
                          ],
                        ],
                      ),
                    ),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 16),
            const _ProductListDemo(),
          ],
        ),
      ),
    );
  }
}

// ตัวอย่าง Flutter UI ที่แสดงใน native app
class _ProductListDemo extends StatelessWidget {
  const _ProductListDemo();

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text(
              'Flutter Product List',
              style: TextStyle(fontSize: 16, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            ...List.generate(
              3,
              (i) => ListTile(
                dense: true,
                leading: Container(
                  width: 40,
                  height: 40,
                  decoration: BoxDecoration(
                    color: Colors.blue.shade100,
                    borderRadius: BorderRadius.circular(8),
                  ),
                  child: Icon(Icons.shopping_bag, color: Colors.blue.shade700),
                ),
                title: Text('Product ${i + 1}'),
                subtitle: Text('\$${(i + 1) * 29.99}'),
                trailing: const Icon(Icons.add_shopping_cart),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class ProductCatalogApp extends StatelessWidget {
  const ProductCatalogApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      home: Scaffold(
        appBar: AppBar(title: const Text('Flutter Product Catalog')),
        body: const Center(child: Text('Product Catalog Feature')),
      ),
    );
  }
}

class CheckoutApp extends StatelessWidget {
  const CheckoutApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      home: Scaffold(
        appBar: AppBar(title: const Text('Flutter Checkout')),
        body: const Center(child: Text('Checkout Feature')),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3084: Android - FlutterActivity

```kotlin
// android/app/src/main/kotlin/com/example/myapp/MainActivity.kt
package com.example.myapp

import android.content.Intent
import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.android.FlutterActivityLaunchConfigs
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.embedding.engine.FlutterEngineCache
import io.flutter.embedding.engine.dart.DartExecutor
import io.flutter.plugin.common.MethodChannel

class MainActivity : AppCompatActivity() {

    companion object {
        const val FLUTTER_ENGINE_ID = "my_flutter_engine"
        const val METHOD_CHANNEL = "com.example.flutter_module/main"
    }

    private lateinit var flutterEngine: FlutterEngine
    private lateinit var methodChannel: MethodChannel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        setupFlutterEngine()
        setupButtons()
    }

    private fun setupFlutterEngine() {
        // สร้าง Flutter Engine และ cache ไว้ใช้ซ้ำ
        flutterEngine = FlutterEngine(this).apply {
            // ระบุ Dart entry point
            dartExecutor.executeDartEntrypoint(
                DartExecutor.DartEntrypoint.createDefault()
            )
        }

        // Cache engine เพื่อใช้ซ้ำ (ไม่ต้อง warm up ใหม่)
        FlutterEngineCache.getInstance().put(FLUTTER_ENGINE_ID, flutterEngine)

        // Setup MethodChannel
        methodChannel = MethodChannel(
            flutterEngine.dartExecutor.binaryMessenger,
            METHOD_CHANNEL
        )

        // Handle calls FROM Flutter
        methodChannel.setMethodCallHandler { call, result ->
            when (call.method) {
                "showNativeToast" -> {
                    val message = call.argument<String>("message") ?: call.arguments as? String ?: ""
                    Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
                    result.success(null)
                }
                "openNativeCamera" -> {
                    // Open camera
                    result.success("Camera opened")
                }
                "getBatteryLevel" -> {
                    val batteryLevel = getBatteryLevel()
                    result.success(batteryLevel)
                }
                "getUserData" -> {
                    val userData = mapOf(
                        "userId" to "user123",
                        "name" to "John Doe",
                        "email" to "john@example.com",
                        "plan" to "premium"
                    )
                    result.success(userData)
                }
                "closeFlutter" -> {
                    // Flutter is asking to close itself
                    result.success(null)
                }
                else -> result.notImplemented()
            }
        }
    }

    private fun getBatteryLevel(): Int {
        // Simplified - ใน production ใช้ BatteryManager
        return 85
    }

    private fun setupButtons() {
        // Launch full Flutter screen
        findViewById<Button>(R.id.btnFlutterFullScreen)?.setOnClickListener {
            launchFlutterFullScreen()
        }

        // Launch Flutter in fragment
        findViewById<Button>(R.id.btnFlutterFragment)?.setOnClickListener {
            launchFlutterFragment()
        }

        // Send message to Flutter
        findViewById<Button>(R.id.btnSendToFlutter)?.setOnClickListener {
            sendMessageToFlutter("Hello from Android!")
        }

        // Send data to Flutter
        findViewById<Button>(R.id.btnSendDataToFlutter)?.setOnClickListener {
            sendDataToFlutter()
        }
    }

    private fun launchFlutterFullScreen() {
        // Option 1: ใช้ cached engine
        startActivity(
            FlutterActivity
                .withCachedEngine(FLUTTER_ENGINE_ID)
                .build(this)
        )
    }

    private fun launchFlutterFragment() {
        startActivity(
            Intent(this, FlutterFragmentActivity::class.java)
        )
    }

    private fun sendMessageToFlutter(message: String) {
        methodChannel.invokeMethod("updateMessage", message) { result ->
            // Handle result from Flutter
            when (result) {
                is String -> Toast.makeText(this, "Flutter: $result", Toast.LENGTH_SHORT).show()
                is FlutterException -> Toast.makeText(this, "Error: ${result.message}", Toast.LENGTH_SHORT).show()
            }
        }
    }

    private fun sendDataToFlutter() {
        val data = mapOf(
            "userId" to 123,
            "username" to "john_doe",
            "preferences" to listOf("dark_mode", "notifications"),
            "timestamp" to System.currentTimeMillis()
        )
        methodChannel.invokeMethod("updateData", data)
    }

    override fun onDestroy() {
        super.onDestroy()
        // Clean up Flutter engine
        FlutterEngineCache.getInstance().remove(FLUTTER_ENGINE_ID)
        flutterEngine.destroy()
    }
}
```

---

## ขั้นตอนที่ 3085: Android - FlutterFragment

```kotlin
// android/app/src/main/kotlin/com/example/myapp/FlutterFragmentActivity.kt
package com.example.myapp

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import androidx.fragment.app.Fragment
import androidx.fragment.app.FragmentManager
import io.flutter.embedding.android.FlutterFragment

class FlutterFragmentActivity : AppCompatActivity() {

    companion object {
        private const val TAG_FLUTTER_FRAGMENT = "flutter_fragment"
        private const val FRAGMENT_CONTAINER_ID = android.R.id.content
    }

    private var flutterFragment: FlutterFragment? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_flutter_fragment)

        // Try to restore existing fragment
        val fragmentManager: FragmentManager = supportFragmentManager
        flutterFragment = fragmentManager
            .findFragmentByTag(TAG_FLUTTER_FRAGMENT) as FlutterFragment?

        if (flutterFragment == null) {
            // Create and add fragment
            flutterFragment = FlutterFragment
                .withCachedEngine(MainActivity.FLUTTER_ENGINE_ID)
                .shouldAttachEngineToActivity(true)
                .build()

            fragmentManager
                .beginTransaction()
                .add(
                    R.id.flutter_container,
                    flutterFragment!!,
                    TAG_FLUTTER_FRAGMENT
                )
                .commit()
        }
    }

    override fun onPostResume() {
        super.onPostResume()
        flutterFragment?.onPostResume()
    }

    override fun onNewIntent(intent: android.content.Intent) {
        super.onNewIntent(intent)
        flutterFragment?.onNewIntent(intent)
    }

    override fun onBackPressed() {
        flutterFragment?.onBackPressed()
    }

    override fun onRequestPermissionsResult(
        requestCode: Int,
        permissions: Array<String>,
        grantResults: IntArray
    ) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        flutterFragment?.onRequestPermissionsResult(
            requestCode, permissions, grantResults
        )
    }

    override fun onUserLeaveHint() {
        flutterFragment?.onUserLeaveHint()
    }

    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)
        flutterFragment?.onTrimMemory(level)
    }
}

// android/app/src/main/res/layout/activity_flutter_fragment.xml
/*
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <!-- Native header -->
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="#2196F3"
        android:padding="16dp">
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Native + Flutter Fragment"
            android:textColor="#FFFFFF"
            android:textSize="18sp"/>
    </LinearLayout>

    <!-- Flutter Fragment container -->
    <FrameLayout
        android:id="@+id/flutter_container"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"/>

    <!-- Native footer -->
    <Button
        android:id="@+id/btnClose"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Close Flutter"
        android:margin="8dp"/>

</LinearLayout>
*/
```

---

## ขั้นตอนที่ 3086: iOS - FlutterViewController (Swift)

```swift
// ios/Runner/AppDelegate.swift
import UIKit
import Flutter
import FlutterPluginRegistrant

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
    
    // Pre-warm Flutter engine
    lazy var flutterEngine = FlutterEngine(name: "my_flutter_engine")
    
    override func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        
        // Pre-warm Flutter engine ก่อนที่จะแสดง Flutter view
        // เพื่อให้ transition เร็วขึ้น
        flutterEngine.run()
        
        // Register generated plugins
        GeneratedPluginRegistrant.register(with: self.flutterEngine)
        
        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }
}

// ios/Runner/NativeViewController.swift
import UIKit
import Flutter

class NativeViewController: UIViewController {
    
    var flutterEngine: FlutterEngine? {
        return (UIApplication.shared.delegate as? AppDelegate)?.flutterEngine
    }
    
    // MethodChannel
    var methodChannel: FlutterMethodChannel?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .white
        title = "Native View"
        
        setupUI()
        setupMethodChannel()
    }
    
    func setupMethodChannel() {
        guard let engine = flutterEngine else { return }
        
        methodChannel = FlutterMethodChannel(
            name: "com.example.flutter_module/main",
            binaryMessenger: engine.binaryMessenger
        )
        
        // Handle calls FROM Flutter
        methodChannel?.setMethodCallHandler { [weak self] call, result in
            switch call.method {
            case "showNativeToast":
                let message = call.arguments as? String ?? ""
                self?.showToast(message: message)
                result(nil)
                
            case "openNativeCamera":
                self?.openCamera()
                result("Camera opened")
                
            case "getBatteryLevel":
                let level = self?.getBatteryLevel() ?? -1
                result(level)
                
            case "getUserData":
                let userData: [String: Any] = [
                    "userId": "user456",
                    "name": "Jane Smith",
                    "email": "jane@example.com",
                    "plan": "premium"
                ]
                result(userData)
                
            case "closeFlutter":
                self?.dismissFlutter()
                result(nil)
                
            default:
                result(FlutterMethodNotImplemented)
            }
        }
    }
    
    func setupUI() {
        let stackView = UIStackView()
        stackView.axis = .vertical
        stackView.spacing = 16
        stackView.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(stackView)
        
        NSLayoutConstraint.activate([
            stackView.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            stackView.centerYAnchor.constraint(equalTo: view.centerYAnchor),
            stackView.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 20),
            stackView.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -20)
        ])
        
        // Full screen Flutter button
        let fullScreenBtn = makeButton(
            title: "Open Flutter Full Screen",
            action: #selector(openFlutterFullScreen)
        )
        stackView.addArrangedSubview(fullScreenBtn)
        
        // Flutter in container button
        let containerBtn = makeButton(
            title: "Flutter in Native Container",
            action: #selector(openFlutterInContainer)
        )
        stackView.addArrangedSubview(containerBtn)
        
        // Send message button
        let messageBtn = makeButton(
            title: "Send Message to Flutter",
            action: #selector(sendMessageToFlutter)
        )
        stackView.addArrangedSubview(messageBtn)
    }
    
    func makeButton(title: String, action: Selector) -> UIButton {
        let button = UIButton(type: .system)
        button.setTitle(title, for: .normal)
        button.addTarget(self, action: action, for: .touchUpInside)
        button.backgroundColor = .systemBlue
        button.setTitleColor(.white, for: .normal)
        button.layer.cornerRadius = 8
        button.heightAnchor.constraint(equalToConstant: 50).isActive = true
        return button
    }
    
    @objc func openFlutterFullScreen() {
        guard let engine = flutterEngine else { return }
        
        let flutterVC = FlutterViewController(
            engine: engine,
            nibName: nil,
            bundle: nil
        )
        flutterVC.modalPresentationStyle = .fullScreen
        present(flutterVC, animated: true)
    }
    
    @objc func openFlutterInContainer() {
        let containerVC = FlutterContainerViewController()
        navigationController?.pushViewController(containerVC, animated: true)
    }
    
    @objc func sendMessageToFlutter() {
        methodChannel?.invokeMethod("updateMessage", arguments: "Hello from iOS!") { result in
            if let response = result as? String {
                print("Flutter responded: \(response)")
            }
        }
    }
    
    func showToast(message: String) {
        // Simple toast implementation
        let alert = UIAlertController(title: nil, message: message, preferredStyle: .alert)
        present(alert, animated: true)
        DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
            alert.dismiss(animated: true)
        }
    }
    
    func openCamera() {
        let imagePicker = UIImagePickerController()
        imagePicker.sourceType = .camera
        present(imagePicker, animated: true)
    }
    
    func getBatteryLevel() -> Int {
        UIDevice.current.isBatteryMonitoringEnabled = true
        let level = UIDevice.current.batteryLevel
        return Int(level * 100)
    }
    
    func dismissFlutter() {
        dismiss(animated: true)
    }
}

// Flutter embedded in a container view (ไม่ fullscreen)
class FlutterContainerViewController: UIViewController {
    
    var flutterViewController: FlutterViewController?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground
        title = "Embedded Flutter"
        
        setupNativeContent()
        embedFlutter()
    }
    
    func setupNativeContent() {
        // Native header label
        let headerLabel = UILabel()
        headerLabel.text = "Native Header (UIKit)"
        headerLabel.textAlignment = .center
        headerLabel.backgroundColor = .systemOrange
        headerLabel.textColor = .white
        headerLabel.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(headerLabel)
        
        NSLayoutConstraint.activate([
            headerLabel.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
            headerLabel.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            headerLabel.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            headerLabel.heightAnchor.constraint(equalToConstant: 50)
        ])
    }
    
    func embedFlutter() {
        guard let engine = (UIApplication.shared.delegate as? AppDelegate)?.flutterEngine else {
            return
        }
        
        flutterViewController = FlutterViewController(
            engine: engine,
            nibName: nil,
            bundle: nil
        )
        
        guard let flutterVC = flutterViewController else { return }
        
        // Add as child view controller
        addChild(flutterVC)
        view.addSubview(flutterVC.view)
        flutterVC.view.translatesAutoresizingMaskIntoConstraints = false
        
        // Position Flutter below native header, above native footer
        NSLayoutConstraint.activate([
            flutterVC.view.topAnchor.constraint(
                equalTo: view.safeAreaLayoutGuide.topAnchor,
                constant: 50 // header height
            ),
            flutterVC.view.bottomAnchor.constraint(
                equalTo: view.safeAreaLayoutGuide.bottomAnchor,
                constant: -60 // footer height
            ),
            flutterVC.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            flutterVC.view.trailingAnchor.constraint(equalTo: view.trailingAnchor)
        ])
        
        flutterVC.didMove(toParent: self)
        
        // Native footer button
        let footerButton = UIButton(type: .system)
        footerButton.setTitle("Native Footer Button", for: .normal)
        footerButton.backgroundColor = .systemGreen
        footerButton.setTitleColor(.white, for: .normal)
        footerButton.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(footerButton)
        
        NSLayoutConstraint.activate([
            footerButton.bottomAnchor.constraint(equalTo: view.safeAreaLayoutGuide.bottomAnchor),
            footerButton.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            footerButton.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            footerButton.heightAnchor.constraint(equalToConstant: 50)
        ])
    }
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        // Notify Flutter engine
        flutterViewController?.flutterEngine?.lifecycleChannel.sendAppIsInactive()
    }
}
```

---

## ขั้นตอนที่ 3087: EventChannel - Streaming Data

```dart
// flutter_module/lib/channels/event_channel_demo.dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

/// EventChannel ใช้สำหรับ streaming data จาก Native → Flutter
/// เหมาะสำหรับ:
/// - Sensor data (accelerometer, GPS)
/// - Battery level changes
/// - Network status changes
/// - Real-time native events

class SensorDataWidget extends StatefulWidget {
  const SensorDataWidget({super.key});

  @override
  State<SensorDataWidget> createState() => _SensorDataWidgetState();
}

class _SensorDataWidgetState extends State<SensorDataWidget> {
  // EventChannel สำหรับ accelerometer
  static const EventChannel _accelerometerChannel =
      EventChannel('com.example.flutter_module/accelerometer');

  // EventChannel สำหรับ network status
  static const EventChannel _networkChannel =
      EventChannel('com.example.flutter_module/network');

  // MethodChannel สำหรับ battery
  static const MethodChannel _batteryChannel =
      MethodChannel('com.example.flutter_module/battery');

  StreamSubscription? _accelerometerSub;
  StreamSubscription? _networkSub;

  Map<String, double> _accelerometerData = {'x': 0, 'y': 0, 'z': 0};
  String _networkStatus = 'Unknown';
  int _batteryLevel = -1;
  Timer? _batteryTimer;

  @override
  void initState() {
    super.initState();
    _startListening();
    _startBatteryPolling();
  }

  void _startListening() {
    // Listen to accelerometer stream
    _accelerometerSub = _accelerometerChannel
        .receiveBroadcastStream()
        .listen(
      (event) {
        if (event is Map) {
          setState(() {
            _accelerometerData = {
              'x': (event['x'] as num?)?.toDouble() ?? 0,
              'y': (event['y'] as num?)?.toDouble() ?? 0,
              'z': (event['z'] as num?)?.toDouble() ?? 0,
            };
          });
        }
      },
      onError: (error) {
        debugPrint('Accelerometer error: $error');
      },
    );

    // Listen to network status stream
    _networkSub = _networkChannel.receiveBroadcastStream().listen(
      (event) {
        setState(() {
          _networkStatus = event as String? ?? 'Unknown';
        });
      },
      onError: (error) {
        setState(() => _networkStatus = 'Error: $error');
      },
    );
  }

  void _startBatteryPolling() {
    _batteryTimer = Timer.periodic(const Duration(seconds: 30), (_) {
      _getBatteryLevel();
    });
    _getBatteryLevel(); // Initial read
  }

  Future<void> _getBatteryLevel() async {
    try {
      final level =
          await _batteryChannel.invokeMethod<int>('getBatteryLevel');
      if (mounted) {
        setState(() => _batteryLevel = level ?? -1);
      }
    } on PlatformException catch (e) {
      debugPrint('Battery error: ${e.message}');
    }
  }

  @override
  void dispose() {
    _accelerometerSub?.cancel();
    _networkSub?.cancel();
    _batteryTimer?.cancel();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Native Sensors Demo')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          // Accelerometer
          Card(
            child: Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  const Row(
                    children: [
                      Icon(Icons.sensors, color: Colors.blue),
                      SizedBox(width: 8),
                      Text(
                        'Accelerometer',
                        style: TextStyle(
                          fontSize: 16,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                    ],
                  ),
                  const SizedBox(height: 16),
                  _buildSensorBar(
                      'X', _accelerometerData['x']!, Colors.red),
                  _buildSensorBar(
                      'Y', _accelerometerData['y']!, Colors.green),
                  _buildSensorBar(
                      'Z', _accelerometerData['z']!, Colors.blue),
                ],
              ),
            ),
          ),
          const SizedBox(height: 16),
          // Network Status
          Card(
            child: ListTile(
              leading: Icon(
                _networkStatus.contains('wifi')
                    ? Icons.wifi
                    : _networkStatus.contains('mobile')
                        ? Icons.signal_cellular_alt
                        : Icons.signal_wifi_off,
                color: _networkStatus == 'none' ? Colors.red : Colors.green,
              ),
              title: const Text('Network Status'),
              subtitle: Text(_networkStatus),
            ),
          ),
          const SizedBox(height: 16),
          // Battery
          Card(
            child: ListTile(
              leading: Icon(
                _batteryLevel > 20
                    ? Icons.battery_full
                    : Icons.battery_alert,
                color: _batteryLevel > 20 ? Colors.green : Colors.red,
              ),
              title: const Text('Battery Level'),
              subtitle: Text(
                _batteryLevel >= 0 ? '$_batteryLevel%' : 'Unknown',
              ),
              trailing: _batteryLevel >= 0
                  ? SizedBox(
                      width: 60,
                      child: LinearProgressIndicator(
                        value: _batteryLevel / 100,
                        backgroundColor: Colors.grey.shade200,
                        color: _batteryLevel > 20 ? Colors.green : Colors.red,
                      ),
                    )
                  : null,
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildSensorBar(String axis, double value, Color color) {
    // Normalize value from -10 to 10 to 0-1
    final normalized = ((value + 10) / 20).clamp(0.0, 1.0);

    return Padding(
      padding: const EdgeInsets.only(bottom: 8),
      child: Row(
        children: [
          SizedBox(
            width: 24,
            child: Text(
              axis,
              style: TextStyle(fontWeight: FontWeight.bold, color: color),
            ),
          ),
          const SizedBox(width: 8),
          Expanded(
            child: ClipRRect(
              borderRadius: BorderRadius.circular(4),
              child: LinearProgressIndicator(
                value: normalized,
                backgroundColor: Colors.grey.shade200,
                color: color,
                minHeight: 12,
              ),
            ),
          ),
          const SizedBox(width: 8),
          SizedBox(
            width: 50,
            child: Text(
              value.toStringAsFixed(2),
              style: const TextStyle(fontSize: 12),
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3088: Multiple Flutter Engines

```dart
// flutter_module/lib/multi_engine_demo.dart
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

/// Multiple Flutter instances ใน Native App
/// ใช้ FlutterEngineGroup เพื่อ share resources ระหว่าง engines

// Native side code (Kotlin) - หลักการ:
/*
// Android - Multiple engines
val engineGroup = FlutterEngineGroup(context)

// Engine 1 - Checkout feature
val checkoutEngine = engineGroup.createAndRunDefaultEngine(context).also { engine ->
    DartExecutor.DartEntrypoint(
        FlutterInjector.instance().flutterLoader().findAppBundlePath(),
        "checkout"
    ).let { engine.dartExecutor.executeDartEntrypoint(it) }
}

// Engine 2 - Product catalog feature
val catalogEngine = engineGroup.createAndRunDefaultEngine(context).also { engine ->
    DartExecutor.DartEntrypoint(
        FlutterInjector.instance().flutterLoader().findAppBundlePath(),
        "productCatalog"
    ).let { engine.dartExecutor.executeDartEntrypoint(it) }
}
*/

// Flutter side - รองรับหลาย entry points
@pragma('vm:entry-point')
void checkoutEntryPoint() {
  runApp(const CheckoutFeatureApp());
}

@pragma('vm:entry-point')
void catalogEntryPoint() {
  runApp(const CatalogFeatureApp());
}

class CheckoutFeatureApp extends StatefulWidget {
  const CheckoutFeatureApp({super.key});

  @override
  State<CheckoutFeatureApp> createState() => _CheckoutFeatureAppState();
}

class _CheckoutFeatureAppState extends State<CheckoutFeatureApp> {
  static const MethodChannel _channel =
      MethodChannel('com.example.app/checkout');

  List<CartItem> _cartItems = [];
  double _total = 0;

  @override
  void initState() {
    super.initState();
    _channel.setMethodCallHandler(_handleNativeCall);
    _loadCartFromNative();
  }

  Future<dynamic> _handleNativeCall(MethodCall call) async {
    switch (call.method) {
      case 'updateCart':
        final items = (call.arguments as List)
            .map((item) => CartItem.fromMap(Map<String, dynamic>.from(item)))
            .toList();
        setState(() {
          _cartItems = items;
          _total = _cartItems.fold(0, (sum, item) => sum + item.price * item.quantity);
        });
        return true;
      default:
        throw PlatformException(code: 'NOT_IMPLEMENTED');
    }
  }

  Future<void> _loadCartFromNative() async {
    try {
      final cartData =
          await _channel.invokeMethod<List<dynamic>>('getCart');
      if (cartData != null) {
        setState(() {
          _cartItems = cartData
              .map((item) =>
                  CartItem.fromMap(Map<String, dynamic>.from(item as Map)))
              .toList();
          _total = _cartItems.fold(
              0, (sum, item) => sum + item.price * item.quantity);
        });
      }
    } catch (e) {
      debugPrint('Error loading cart: $e');
    }
  }

  Future<void> _processCheckout() async {
    try {
      final result = await _channel.invokeMethod('processPayment', {
        'items': _cartItems.map((i) => i.toMap()).toList(),
        'total': _total,
      });
      if (result == true) {
        await _channel.invokeMethod('checkoutComplete');
      }
    } catch (e) {
      debugPrint('Checkout error: $e');
    }
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.green),
        useMaterial3: true,
      ),
      home: Scaffold(
        appBar: AppBar(
          title: const Text('Checkout'),
          backgroundColor: Colors.green,
          foregroundColor: Colors.white,
        ),
        body: Column(
          children: [
            Expanded(
              child: _cartItems.isEmpty
                  ? const Center(child: Text('Your cart is empty'))
                  : ListView.builder(
                      itemCount: _cartItems.length,
                      itemBuilder: (context, index) {
                        final item = _cartItems[index];
                        return ListTile(
                          title: Text(item.name),
                          subtitle: Text('\$${item.price} x ${item.quantity}'),
                          trailing: Text(
                            '\$${(item.price * item.quantity).toStringAsFixed(2)}',
                            style: const TextStyle(fontWeight: FontWeight.bold),
                          ),
                        );
                      },
                    ),
            ),
            Container(
              padding: const EdgeInsets.all(16),
              decoration: BoxDecoration(
                border: Border(top: BorderSide(color: Colors.grey.shade200)),
              ),
              child: Column(
                children: [
                  Row(
                    mainAxisAlignment: MainAxisAlignment.spaceBetween,
                    children: [
                      const Text(
                        'Total',
                        style: TextStyle(
                          fontSize: 18,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      Text(
                        '\$${_total.toStringAsFixed(2)}',
                        style: const TextStyle(
                          fontSize: 18,
                          fontWeight: FontWeight.bold,
                          color: Colors.green,
                        ),
                      ),
                    ],
                  ),
                  const SizedBox(height: 16),
                  ElevatedButton(
                    onPressed: _cartItems.isEmpty ? null : _processCheckout,
                    style: ElevatedButton.styleFrom(
                      backgroundColor: Colors.green,
                      foregroundColor: Colors.white,
                      minimumSize: const Size(double.infinity, 56),
                    ),
                    child: const Text(
                      'Proceed to Payment',
                      style: TextStyle(fontSize: 16),
                    ),
                  ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class CatalogFeatureApp extends StatelessWidget {
  const CatalogFeatureApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      home: Scaffold(
        appBar: AppBar(title: const Text('Product Catalog')),
        body: GridView.builder(
          padding: const EdgeInsets.all(16),
          gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: 2,
            crossAxisSpacing: 12,
            mainAxisSpacing: 12,
            childAspectRatio: 0.75,
          ),
          itemCount: 20,
          itemBuilder: (context, index) {
            return Card(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Expanded(
                    child: Container(
                      color: Colors.grey.shade200,
                      child: const Center(child: Icon(Icons.image, size: 50)),
                    ),
                  ),
                  Padding(
                    padding: const EdgeInsets.all(8),
                    child: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        Text('Product ${index + 1}'),
                        Text(
                          '\$${(index + 1) * 9.99}',
                          style: const TextStyle(
                            fontWeight: FontWeight.bold,
                            color: Colors.green,
                          ),
                        ),
                      ],
                    ),
                  ),
                ],
              ),
            );
          },
        ),
      ),
    );
  }
}

// Data model
class CartItem {
  const CartItem({
    required this.id,
    required this.name,
    required this.price,
    required this.quantity,
  });

  final String id;
  final String name;
  final double price;
  final int quantity;

  factory CartItem.fromMap(Map<String, dynamic> map) => CartItem(
        id: map['id'] as String,
        name: map['name'] as String,
        price: (map['price'] as num).toDouble(),
        quantity: map['quantity'] as int,
      );

  Map<String, dynamic> toMap() => {
        'id': id,
        'name': name,
        'price': price,
        'quantity': quantity,
      };
}

void main() => runApp(const MainFlutterScreen());
```

---

**← [Part 79](part-79-accessibility-advanced.md)**
**ต่อไป: [Part 81 →](part-81-flutter-desktop.md)**
