# Part 78: Flutter State Restoration
## ขั้นตอนที่ 3001-3040

## 🎯 เป้าหมายของ Part นี้
- ใช้ RestorationMixin เพื่อเก็บ state ข้าม app restart
- ใช้ RestorableInt, RestorableString, RestorableBool
- Restore scroll position อัตโนมัติ
- Form state restoration
- Navigator state restoration

---

## ขั้นตอนที่ 3001: State Restoration คืออะไร?

```dart
// lib/restoration/restoration_intro.dart
import 'package:flutter/material.dart';

/// State Restoration ทำให้ app กลับมาอยู่ใน state เดิม
/// หลังจาก:
/// 1. OS kill app เพื่อประหยัด memory (iOS background kill)
/// 2. User กด home button แล้วกลับมา
/// 3. Screen rotation (บน Android)
/// 4. Multi-window switching (iPadOS)
///
/// ไม่ต้องใช้ shared_preferences หรือ database สำหรับ UI state

// การ setup ใน MaterialApp
class RestorationDemoApp extends StatelessWidget {
  const RestorationDemoApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'State Restoration Demo',
      // เปิดใช้ restorationScopeId ที่ root
      restorationScopeId: 'root',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const HomeScreen(),
    );
  }
}
```

---

## ขั้นตอนที่ 3002: RestorationMixin กับ Restorable Types

```dart
// lib/restoration/restorable_types_demo.dart
import 'package:flutter/material.dart';

/// Restorable types ที่ Flutter มีให้:
/// - RestorableInt
/// - RestorableDouble
/// - RestorableBool
/// - RestorableString
/// - RestorableTextEditingController
/// - RestorableDateTime
/// - RestorableListenable
/// - RestorableEnumN (custom)

class CounterScreen extends StatefulWidget {
  const CounterScreen({super.key});

  @override
  State<CounterScreen> createState() => _CounterScreenState();
}

class _CounterScreenState extends State<CounterScreen>
    with RestorationMixin {
  // ประกาศ restorable properties
  final RestorableInt _counter = RestorableInt(0);
  final RestorableBool _isDarkMode = RestorableBool(false);
  final RestorableString _username = RestorableString('');
  final RestorableDouble _sliderValue = RestorableDouble(0.5);

  @override
  String get restorationId => 'counter_screen';

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    // ลงทะเบียน properties ที่ต้องการ restore
    registerForRestoration(_counter, 'counter');
    registerForRestoration(_isDarkMode, 'is_dark_mode');
    registerForRestoration(_username, 'username');
    registerForRestoration(_sliderValue, 'slider_value');
  }

  @override
  void dispose() {
    _counter.dispose();
    _isDarkMode.dispose();
    _username.dispose();
    _sliderValue.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Counter with State Restoration'),
        backgroundColor: _isDarkMode.value ? Colors.grey.shade800 : null,
      ),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // Counter display
            Card(
              child: Padding(
                padding: const EdgeInsets.all(24),
                child: Column(
                  children: [
                    const Text(
                      'Counter Value',
                      style: TextStyle(fontSize: 16),
                    ),
                    Text(
                      '${_counter.value}',
                      style: const TextStyle(
                        fontSize: 64,
                        fontWeight: FontWeight.bold,
                        color: Colors.indigo,
                      ),
                    ),
                    Row(
                      mainAxisAlignment: MainAxisAlignment.center,
                      children: [
                        IconButton.filled(
                          onPressed: () => setState(() => _counter.value--),
                          icon: const Icon(Icons.remove),
                        ),
                        const SizedBox(width: 16),
                        IconButton.filled(
                          onPressed: () => setState(() => _counter.value++),
                          icon: const Icon(Icons.add),
                        ),
                      ],
                    ),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 16),
            // Dark mode toggle
            SwitchListTile(
              title: const Text('Dark Mode'),
              subtitle: const Text('This preference is restored'),
              value: _isDarkMode.value,
              onChanged: (value) {
                setState(() => _isDarkMode.value = value);
              },
            ),
            const SizedBox(height: 16),
            // Slider
            ListTile(
              title: const Text('Volume'),
              subtitle: Slider(
                value: _sliderValue.value,
                onChanged: (value) {
                  setState(() => _sliderValue.value = value);
                },
              ),
              trailing: Text('${(_sliderValue.value * 100).toInt()}%'),
            ),
            const SizedBox(height: 16),
            // Username
            if (_username.value.isNotEmpty)
              Text(
                'Welcome back, ${_username.value}!',
                style: const TextStyle(
                  fontSize: 18,
                  color: Colors.indigo,
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

## ขั้นตอนที่ 3003: Form State Restoration

```dart
// lib/restoration/form_state_restoration.dart
import 'package:flutter/material.dart';

class RegistrationForm extends StatefulWidget {
  const RegistrationForm({super.key});

  @override
  State<RegistrationForm> createState() => _RegistrationFormState();
}

class _RegistrationFormState extends State<RegistrationForm>
    with RestorationMixin {
  // TextEditingController พร้อม restoration
  final RestorableTextEditingController _nameController =
      RestorableTextEditingController();
  final RestorableTextEditingController _emailController =
      RestorableTextEditingController();
  final RestorableTextEditingController _passwordController =
      RestorableTextEditingController();
  final RestorableTextEditingController _phoneController =
      RestorableTextEditingController();

  // Other form state
  final RestorableBool _agreeToTerms = RestorableBool(false);
  final RestorableInt _selectedRoleIndex = RestorableInt(0);
  final RestorableBool _obscurePassword = RestorableBool(true);

  final _formKey = GlobalKey<FormState>();

  static const List<String> _roles = [
    'User',
    'Developer',
    'Designer',
    'Manager',
  ];

  @override
  String get restorationId => 'registration_form';

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    registerForRestoration(_nameController, 'name_controller');
    registerForRestoration(_emailController, 'email_controller');
    registerForRestoration(_passwordController, 'password_controller');
    registerForRestoration(_phoneController, 'phone_controller');
    registerForRestoration(_agreeToTerms, 'agree_to_terms');
    registerForRestoration(_selectedRoleIndex, 'selected_role');
    registerForRestoration(_obscurePassword, 'obscure_password');
  }

  @override
  void dispose() {
    _nameController.dispose();
    _emailController.dispose();
    _passwordController.dispose();
    _phoneController.dispose();
    _agreeToTerms.dispose();
    _selectedRoleIndex.dispose();
    _obscurePassword.dispose();
    super.dispose();
  }

  void _submitForm() {
    if (_formKey.currentState!.validate()) {
      if (!_agreeToTerms.value) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('Please agree to the terms and conditions'),
            backgroundColor: Colors.red,
          ),
        );
        return;
      }

      final data = {
        'name': _nameController.value.text,
        'email': _emailController.value.text,
        'phone': _phoneController.value.text,
        'role': _roles[_selectedRoleIndex.value],
      };

      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text('Registered: ${data['name']} as ${data['role']}'),
          backgroundColor: Colors.green,
        ),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Registration Form'),
        actions: [
          TextButton(
            onPressed: () {
              _nameController.value.clear();
              _emailController.value.clear();
              _passwordController.value.clear();
              _phoneController.value.clear();
              setState(() {
                _agreeToTerms.value = false;
                _selectedRoleIndex.value = 0;
              });
            },
            child: const Text('Clear'),
          ),
        ],
      ),
      body: Form(
        key: _formKey,
        child: ListView(
          padding: const EdgeInsets.all(16),
          children: [
            const Text(
              'Your form data is preserved even if the app is killed!',
              style: TextStyle(
                color: Colors.indigo,
                fontStyle: FontStyle.italic,
              ),
            ),
            const SizedBox(height: 16),
            // Name field
            TextFormField(
              controller: _nameController.value,
              decoration: const InputDecoration(
                labelText: 'Full Name *',
                prefixIcon: Icon(Icons.person),
                border: OutlineInputBorder(),
              ),
              validator: (value) {
                if (value == null || value.isEmpty) {
                  return 'Please enter your name';
                }
                if (value.length < 2) {
                  return 'Name must be at least 2 characters';
                }
                return null;
              },
              textInputAction: TextInputAction.next,
            ),
            const SizedBox(height: 16),
            // Email field
            TextFormField(
              controller: _emailController.value,
              decoration: const InputDecoration(
                labelText: 'Email *',
                prefixIcon: Icon(Icons.email),
                border: OutlineInputBorder(),
              ),
              keyboardType: TextInputType.emailAddress,
              validator: (value) {
                if (value == null || value.isEmpty) {
                  return 'Please enter your email';
                }
                final emailRegex = RegExp(
                  r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$',
                );
                if (!emailRegex.hasMatch(value)) {
                  return 'Please enter a valid email';
                }
                return null;
              },
              textInputAction: TextInputAction.next,
            ),
            const SizedBox(height: 16),
            // Password field
            TextFormField(
              controller: _passwordController.value,
              decoration: InputDecoration(
                labelText: 'Password *',
                prefixIcon: const Icon(Icons.lock),
                border: const OutlineInputBorder(),
                suffixIcon: IconButton(
                  icon: Icon(
                    _obscurePassword.value
                        ? Icons.visibility
                        : Icons.visibility_off,
                  ),
                  onPressed: () {
                    setState(() {
                      _obscurePassword.value = !_obscurePassword.value;
                    });
                  },
                ),
              ),
              obscureText: _obscurePassword.value,
              validator: (value) {
                if (value == null || value.isEmpty) {
                  return 'Please enter a password';
                }
                if (value.length < 8) {
                  return 'Password must be at least 8 characters';
                }
                return null;
              },
              textInputAction: TextInputAction.next,
            ),
            const SizedBox(height: 16),
            // Phone field
            TextFormField(
              controller: _phoneController.value,
              decoration: const InputDecoration(
                labelText: 'Phone Number',
                prefixIcon: Icon(Icons.phone),
                border: OutlineInputBorder(),
              ),
              keyboardType: TextInputType.phone,
              textInputAction: TextInputAction.done,
            ),
            const SizedBox(height: 16),
            // Role selection
            DropdownButtonFormField<int>(
              value: _selectedRoleIndex.value,
              decoration: const InputDecoration(
                labelText: 'Role',
                prefixIcon: Icon(Icons.work),
                border: OutlineInputBorder(),
              ),
              items: List.generate(
                _roles.length,
                (index) => DropdownMenuItem(
                  value: index,
                  child: Text(_roles[index]),
                ),
              ),
              onChanged: (value) {
                setState(() {
                  _selectedRoleIndex.value = value ?? 0;
                });
              },
            ),
            const SizedBox(height: 16),
            // Terms checkbox
            CheckboxListTile(
              title: const Text('I agree to the Terms and Conditions'),
              value: _agreeToTerms.value,
              onChanged: (value) {
                setState(() {
                  _agreeToTerms.value = value ?? false;
                });
              },
              controlAffinity: ListTileControlAffinity.leading,
            ),
            const SizedBox(height: 24),
            // Submit button
            ElevatedButton(
              onPressed: _submitForm,
              style: ElevatedButton.styleFrom(
                padding: const EdgeInsets.symmetric(vertical: 16),
                backgroundColor: Colors.indigo,
                foregroundColor: Colors.white,
              ),
              child: const Text(
                'Register',
                style: TextStyle(fontSize: 16),
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

## ขั้นตอนที่ 3004: Scroll Position Restoration

```dart
// lib/restoration/scroll_restoration.dart
import 'package:flutter/material.dart';

class ScrollableListScreen extends StatefulWidget {
  const ScrollableListScreen({super.key});

  @override
  State<ScrollableListScreen> createState() => _ScrollableListScreenState();
}

class _ScrollableListScreenState extends State<ScrollableListScreen>
    with RestorationMixin {
  final RestorableScrollController _scrollController =
      RestorableScrollController();

  @override
  String get restorationId => 'scrollable_list_screen';

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    registerForRestoration(_scrollController, 'scroll_controller');
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Scroll Position Restored'),
        actions: [
          IconButton(
            icon: const Icon(Icons.arrow_upward),
            onPressed: () {
              _scrollController.value.animateTo(
                0,
                duration: const Duration(milliseconds: 500),
                curve: Curves.easeOut,
              );
            },
            tooltip: 'Scroll to top',
          ),
        ],
      ),
      body: ListView.separated(
        controller: _scrollController.value,
        // restorationId บน ListView ก็ restore scroll position ได้เช่นกัน
        restorationId: 'scroll_position',
        itemCount: 100,
        separatorBuilder: (_, __) => const Divider(height: 1),
        itemBuilder: (context, index) {
          return ListTile(
            leading: CircleAvatar(
              backgroundColor: Colors.indigo.shade100,
              child: Text(
                '${index + 1}',
                style: const TextStyle(
                  color: Colors.indigo,
                  fontWeight: FontWeight.bold,
                ),
              ),
            ),
            title: Text('Item ${index + 1}'),
            subtitle: Text('Scroll position is preserved for item $index'),
            trailing: const Icon(Icons.chevron_right),
            onTap: () {
              ScaffoldMessenger.of(context).showSnackBar(
                SnackBar(content: Text('Tapped item ${index + 1}')),
              );
            },
          );
        },
      ),
    );
  }
}

// GridView ที่ restore scroll position
class RestoredGridView extends StatefulWidget {
  const RestoredGridView({super.key});

  @override
  State<RestoredGridView> createState() => _RestoredGridViewState();
}

class _RestoredGridViewState extends State<RestoredGridView>
    with RestorationMixin {
  @override
  String get restorationId => 'restored_grid_view';

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {}

  @override
  Widget build(BuildContext context) {
    return GridView.builder(
      // restorationId ทำให้ Flutter เก็บ scroll position อัตโนมัติ
      restorationId: 'grid_scroll',
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 3,
        crossAxisSpacing: 8,
        mainAxisSpacing: 8,
      ),
      itemCount: 200,
      itemBuilder: (context, index) {
        final colors = [
          Colors.red,
          Colors.blue,
          Colors.green,
          Colors.orange,
          Colors.purple,
        ];
        return Container(
          decoration: BoxDecoration(
            color: colors[index % colors.length].withOpacity(0.3),
            borderRadius: BorderRadius.circular(8),
          ),
          child: Center(
            child: Text(
              '${index + 1}',
              style: const TextStyle(
                fontWeight: FontWeight.bold,
                fontSize: 18,
              ),
            ),
          ),
        );
      },
    );
  }
}
```

---

## ขั้นตอนที่ 3005: Navigator State Restoration

```dart
// lib/restoration/navigator_restoration.dart
import 'package:flutter/material.dart';

/// Navigator State Restoration จะจำ route history ไว้
/// เมื่อ app กลับมา จะ restore navigation stack เดิม

class NavigationRestorationApp extends StatelessWidget {
  const NavigationRestorationApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Navigation Restoration',
      restorationScopeId: 'app',
      // ใช้ onGenerateRoute เพื่อ support restoration
      onGenerateRoute: (settings) {
        switch (settings.name) {
          case '/':
            return MaterialPageRoute(
              settings: settings,
              builder: (_) => const HomeScreen(),
            );
          case '/products':
            return MaterialPageRoute(
              settings: settings,
              builder: (_) => const ProductListScreen(),
            );
          case '/product-detail':
            final productId = settings.arguments as String? ?? 'unknown';
            return MaterialPageRoute(
              settings: settings,
              builder: (_) => ProductDetailScreen(productId: productId),
            );
          case '/profile':
            return MaterialPageRoute(
              settings: settings,
              builder: (_) => const ProfileScreen(),
            );
          default:
            return MaterialPageRoute(
              builder: (_) => const HomeScreen(),
            );
        }
      },
    );
  }
}

class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});

  @override
  State<HomeScreen> createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> with RestorationMixin {
  final RestorableInt _selectedTab = RestorableInt(0);

  @override
  String get restorationId => 'home_screen';

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    registerForRestoration(_selectedTab, 'selected_tab');
  }

  @override
  void dispose() {
    _selectedTab.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Home'),
      ),
      body: IndexedStack(
        index: _selectedTab.value,
        children: [
          _buildDashboard(context),
          _buildProducts(context),
          _buildProfile(context),
        ],
      ),
      bottomNavigationBar: NavigationBar(
        selectedIndex: _selectedTab.value,
        onDestinationSelected: (index) {
          setState(() => _selectedTab.value = index);
        },
        destinations: const [
          NavigationDestination(
            icon: Icon(Icons.home_outlined),
            selectedIcon: Icon(Icons.home),
            label: 'Home',
          ),
          NavigationDestination(
            icon: Icon(Icons.shopping_bag_outlined),
            selectedIcon: Icon(Icons.shopping_bag),
            label: 'Products',
          ),
          NavigationDestination(
            icon: Icon(Icons.person_outlined),
            selectedIcon: Icon(Icons.person),
            label: 'Profile',
          ),
        ],
      ),
    );
  }

  Widget _buildDashboard(BuildContext context) {
    return ListView(
      padding: const EdgeInsets.all(16),
      children: [
        Card(
          color: Colors.indigo.shade50,
          child: const Padding(
            padding: EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  'State Restoration Demo',
                  style: TextStyle(
                    fontSize: 20,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                SizedBox(height: 8),
                Text(
                  'Kill the app and reopen it. The selected tab, '
                  'scroll position, and form data will be restored.',
                ),
              ],
            ),
          ),
        ),
        const SizedBox(height: 16),
        ElevatedButton.icon(
          onPressed: () {
            Navigator.pushNamed(context, '/products');
          },
          icon: const Icon(Icons.shopping_bag),
          label: const Text('Go to Products'),
        ),
        const SizedBox(height: 8),
        ElevatedButton.icon(
          onPressed: () {
            Navigator.pushNamed(
              context,
              '/product-detail',
              arguments: 'prod-123',
            );
          },
          icon: const Icon(Icons.info),
          label: const Text('Product Detail'),
        ),
      ],
    );
  }

  Widget _buildProducts(BuildContext context) {
    return const ProductListScreen();
  }

  Widget _buildProfile(BuildContext context) {
    return const ProfileScreen();
  }
}

class ProductListScreen extends StatefulWidget {
  const ProductListScreen({super.key});

  @override
  State<ProductListScreen> createState() => _ProductListScreenState();
}

class _ProductListScreenState extends State<ProductListScreen>
    with RestorationMixin {
  final RestorableTextEditingController _searchController =
      RestorableTextEditingController();
  final RestorableString _selectedFilter = RestorableString('All');

  final List<String> _filters = ['All', 'Electronics', 'Clothing', 'Food'];

  @override
  String get restorationId => 'product_list_screen';

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    registerForRestoration(_searchController, 'search');
    registerForRestoration(_selectedFilter, 'filter');
  }

  @override
  void dispose() {
    _searchController.dispose();
    _selectedFilter.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Padding(
          padding: const EdgeInsets.all(16),
          child: Column(
            children: [
              TextField(
                controller: _searchController.value,
                decoration: const InputDecoration(
                  hintText: 'Search products...',
                  prefixIcon: Icon(Icons.search),
                  border: OutlineInputBorder(),
                ),
              ),
              const SizedBox(height: 8),
              SingleChildScrollView(
                scrollDirection: Axis.horizontal,
                child: Row(
                  children: _filters.map((filter) {
                    final isSelected = _selectedFilter.value == filter;
                    return Padding(
                      padding: const EdgeInsets.only(right: 8),
                      child: FilterChip(
                        label: Text(filter),
                        selected: isSelected,
                        onSelected: (_) {
                          setState(() => _selectedFilter.value = filter);
                        },
                      ),
                    );
                  }).toList(),
                ),
              ),
            ],
          ),
        ),
        Expanded(
          child: ListView.builder(
            restorationId: 'products_list',
            itemCount: 30,
            itemBuilder: (context, index) {
              return ListTile(
                leading: const Icon(Icons.shopping_bag),
                title: Text('Product ${index + 1}'),
                subtitle: Text(_filters[index % _filters.length]),
                trailing: Text('\$${(index + 1) * 9.99}'),
                onTap: () {
                  Navigator.pushNamed(
                    context,
                    '/product-detail',
                    arguments: 'prod-$index',
                  );
                },
              );
            },
          ),
        ),
      ],
    );
  }
}

class ProductDetailScreen extends StatelessWidget {
  const ProductDetailScreen({super.key, required this.productId});

  final String productId;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Product $productId')),
      body: Center(
        child: Text(
          'Product ID: $productId',
          style: const TextStyle(fontSize: 24),
        ),
      ),
    );
  }
}

class ProfileScreen extends StatefulWidget {
  const ProfileScreen({super.key});

  @override
  State<ProfileScreen> createState() => _ProfileScreenState();
}

class _ProfileScreenState extends State<ProfileScreen>
    with RestorationMixin {
  final RestorableBool _notificationsEnabled = RestorableBool(true);
  final RestorableBool _biometricEnabled = RestorableBool(false);
  final RestorableInt _themeIndex = RestorableInt(0);
  final RestorableDouble _fontSize = RestorableDouble(14.0);

  @override
  String get restorationId => 'profile_screen';

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    registerForRestoration(_notificationsEnabled, 'notifications');
    registerForRestoration(_biometricEnabled, 'biometric');
    registerForRestoration(_themeIndex, 'theme');
    registerForRestoration(_fontSize, 'font_size');
  }

  @override
  void dispose() {
    _notificationsEnabled.dispose();
    _biometricEnabled.dispose();
    _themeIndex.dispose();
    _fontSize.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ListView(
      padding: const EdgeInsets.all(16),
      children: [
        const ListTile(
          leading: CircleAvatar(
            radius: 30,
            child: Icon(Icons.person, size: 30),
          ),
          title: Text(
            'John Doe',
            style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
          ),
          subtitle: Text('john.doe@example.com'),
        ),
        const Divider(),
        const Padding(
          padding: EdgeInsets.all(8),
          child: Text(
            'Settings (restored after app kill)',
            style: TextStyle(
              fontWeight: FontWeight.bold,
              color: Colors.indigo,
            ),
          ),
        ),
        SwitchListTile(
          title: const Text('Notifications'),
          value: _notificationsEnabled.value,
          onChanged: (v) => setState(() => _notificationsEnabled.value = v),
        ),
        SwitchListTile(
          title: const Text('Biometric Authentication'),
          value: _biometricEnabled.value,
          onChanged: (v) => setState(() => _biometricEnabled.value = v),
        ),
        ListTile(
          title: const Text('Font Size'),
          subtitle: Slider(
            value: _fontSize.value,
            min: 10,
            max: 22,
            divisions: 12,
            label: '${_fontSize.value.toInt()}px',
            onChanged: (v) => setState(() => _fontSize.value = v),
          ),
          trailing: Text('${_fontSize.value.toInt()}px'),
        ),
        const Divider(),
        ListTile(
          leading: const Icon(Icons.edit),
          title: const Text('Edit Profile (Form Restoration)'),
          trailing: const Icon(Icons.chevron_right),
          onTap: () {
            Navigator.push(
              context,
              MaterialPageRoute(
                builder: (_) => const RegistrationForm(),
              ),
            );
          },
        ),
      ],
    );
  }
}
```

---

## ขั้นตอนที่ 3006: Custom Restorable Type

```dart
// lib/restoration/custom_restorable.dart
import 'package:flutter/material.dart';

/// Custom RestorableProperty สำหรับ enum
class RestorableEnum<T extends Enum> extends RestorableValue<T> {
  RestorableEnum(this._defaultValue, {required this.values});

  final T _defaultValue;
  final List<T> values;

  @override
  T createDefaultValue() => _defaultValue;

  @override
  void didUpdateValue(T? oldValue) {
    notifyListeners();
  }

  @override
  T fromPrimitives(Object? data) {
    if (data is int && data >= 0 && data < values.length) {
      return values[data];
    }
    return _defaultValue;
  }

  @override
  Object? toPrimitives() => value.index;
}

// Custom RestorableProperty สำหรับ List<String>
class RestorableStringList extends RestorableValue<List<String>> {
  RestorableStringList([List<String>? defaultValue])
      : _defaultValue = defaultValue ?? [];

  final List<String> _defaultValue;

  @override
  List<String> createDefaultValue() => List.from(_defaultValue);

  @override
  void didUpdateValue(List<String>? oldValue) {
    notifyListeners();
  }

  @override
  List<String> fromPrimitives(Object? data) {
    if (data is List) {
      return data.cast<String>();
    }
    return List.from(_defaultValue);
  }

  @override
  Object? toPrimitives() => value;
}

// Custom RestorableProperty สำหรับ Set<int>
class RestorableIntSet extends RestorableValue<Set<int>> {
  RestorableIntSet([Set<int>? defaultValue])
      : _defaultValue = defaultValue ?? {};

  final Set<int> _defaultValue;

  @override
  Set<int> createDefaultValue() => Set.from(_defaultValue);

  @override
  void didUpdateValue(Set<int>? oldValue) {
    notifyListeners();
  }

  @override
  Set<int> fromPrimitives(Object? data) {
    if (data is List) {
      return data.cast<int>().toSet();
    }
    return Set.from(_defaultValue);
  }

  @override
  Object? toPrimitives() => value.toList();
}

// Demo ใช้งาน custom restorable types
enum SortOrder { ascending, descending }

class ShoppingCartScreen extends StatefulWidget {
  const ShoppingCartScreen({super.key});

  @override
  State<ShoppingCartScreen> createState() => _ShoppingCartScreenState();
}

class _ShoppingCartScreenState extends State<ShoppingCartScreen>
    with RestorationMixin {
  late final RestorableEnum<SortOrder> _sortOrder;
  final RestorableStringList _selectedItems = RestorableStringList();
  final RestorableIntSet _favoriteIds = RestorableIntSet();
  final RestorableInt _itemCount = RestorableInt(0);

  @override
  void initState() {
    super.initState();
    _sortOrder = RestorableEnum<SortOrder>(
      SortOrder.ascending,
      values: SortOrder.values,
    );
  }

  @override
  String get restorationId => 'shopping_cart';

  @override
  void restoreState(RestorationBucket? oldBucket, bool initialRestore) {
    registerForRestoration(_sortOrder, 'sort_order');
    registerForRestoration(_selectedItems, 'selected_items');
    registerForRestoration(_favoriteIds, 'favorite_ids');
    registerForRestoration(_itemCount, 'item_count');
  }

  @override
  void dispose() {
    _sortOrder.dispose();
    _selectedItems.dispose();
    _favoriteIds.dispose();
    _itemCount.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Shopping Cart'),
        actions: [
          IconButton(
            icon: Icon(
              _sortOrder.value == SortOrder.ascending
                  ? Icons.arrow_upward
                  : Icons.arrow_downward,
            ),
            onPressed: () {
              setState(() {
                _sortOrder.value = _sortOrder.value == SortOrder.ascending
                    ? SortOrder.descending
                    : SortOrder.ascending;
              });
            },
          ),
        ],
      ),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text('Sort: ${_sortOrder.value.name}'),
                Text('Selected: ${_selectedItems.value.join(', ')}'),
                Text('Favorites: ${_favoriteIds.value.join(', ')}'),
                Text('Count: ${_itemCount.value}'),
              ],
            ),
          ),
          Expanded(
            child: ListView.builder(
              restorationId: 'cart_list',
              itemCount: 20,
              itemBuilder: (context, index) {
                final isSelected =
                    _selectedItems.value.contains('item-$index');
                final isFavorite = _favoriteIds.value.contains(index);
                return ListTile(
                  title: Text('Product ${index + 1}'),
                  leading: Checkbox(
                    value: isSelected,
                    onChanged: (_) {
                      setState(() {
                        final items = List<String>.from(_selectedItems.value);
                        if (isSelected) {
                          items.remove('item-$index');
                        } else {
                          items.add('item-$index');
                          _itemCount.value++;
                        }
                        _selectedItems.value = items;
                      });
                    },
                  ),
                  trailing: IconButton(
                    icon: Icon(
                      isFavorite ? Icons.favorite : Icons.favorite_border,
                      color: isFavorite ? Colors.red : null,
                    ),
                    onPressed: () {
                      setState(() {
                        final favs = Set<int>.from(_favoriteIds.value);
                        if (isFavorite) {
                          favs.remove(index);
                        } else {
                          favs.add(index);
                        }
                        _favoriteIds.value = favs;
                      });
                    },
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

void main() {
  runApp(const RestorationDemoApp());
}
```

---

**← [Part 77](part-77-dart-meta-programming.md)**
**ต่อไป: [Part 79 →](part-79-accessibility-advanced.md)**
