# Part 96: Capstone - Admin Dashboard (Flutter Web)
## ขั้นตอนที่ 3721-3760

## 🎯 เป้าหมายของ Part นี้
- สร้าง Flutter Web Admin Dashboard สำหรับระบบ Food Delivery
- จัดการ Orders ด้วยตาราง พร้อม Filter และ Search
- CRUD สำหรับ Restaurant Management
- แสดง Real-time Analytics ด้วย fl_chart
- User Management พร้อม Role-based Access Control

---

## ขั้นตอนที่ 3721: Project Setup และ Dependencies

```yaml
# pubspec.yaml
name: food_admin_dashboard
description: Flutter Web Admin Dashboard for Food Delivery

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: '>=3.10.0'

dependencies:
  flutter:
    sdk: flutter
  flutter_web_plugins:
    sdk: flutter

  # State Management
  flutter_riverpod: ^2.4.9
  riverpod_annotation: ^2.3.3

  # Navigation
  go_router: ^12.1.3

  # Charts
  fl_chart: ^0.65.0

  # UI
  data_table_2: ^2.5.12
  flutter_animate: ^4.3.0
  shimmer: ^3.0.0
  cached_network_image: ^3.3.1

  # HTTP
  dio: ^5.4.0
  retrofit: ^4.1.0

  # Storage
  shared_preferences: ^2.2.2

  # Utils
  intl: ^0.18.1
  equatable: ^2.0.5
  freezed_annotation: ^2.4.1
  json_annotation: ^4.8.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  build_runner: ^2.4.7
  freezed: ^2.4.6
  json_serializable: ^6.7.1
  riverpod_generator: ^2.3.9
  retrofit_generator: ^8.1.0
  flutter_lints: ^3.0.1
  mockito: ^5.4.3

flutter:
  uses-material-design: true
  assets:
    - assets/images/
    - assets/icons/
```

## ขั้นตอนที่ 3722: App Entry Point และ Theme

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:food_admin_dashboard/core/router/app_router.dart';
import 'package:food_admin_dashboard/core/theme/app_theme.dart';

void main() {
  runApp(
    const ProviderScope(
      child: AdminApp(),
    ),
  );
}

class AdminApp extends ConsumerWidget {
  const AdminApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(appRouterProvider);
    final themeMode = ref.watch(themeModeProvider);

    return MaterialApp.router(
      title: 'Food Admin Dashboard',
      debugShowCheckedModeBanner: false,
      theme: AppTheme.lightTheme,
      darkTheme: AppTheme.darkTheme,
      themeMode: themeMode,
      routerConfig: router,
    );
  }
}
```

```dart
// lib/core/theme/app_theme.dart
import 'package:flutter/material.dart';
import 'package:google_fonts/google_fonts.dart';

class AppTheme {
  static const _primaryColor = Color(0xFF6C63FF);
  static const _secondaryColor = Color(0xFF03DAC6);
  static const _errorColor = Color(0xFFCF6679);

  static ThemeData get lightTheme => ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(
          seedColor: _primaryColor,
          secondary: _secondaryColor,
          error: _errorColor,
        ),
        textTheme: GoogleFonts.interTextTheme(),
        cardTheme: const CardTheme(
          elevation: 0,
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.all(Radius.circular(12)),
          ),
        ),
        appBarTheme: const AppBarTheme(
          elevation: 0,
          scrolledUnderElevation: 1,
        ),
      );

  static ThemeData get darkTheme => ThemeData(
        useMaterial3: true,
        brightness: Brightness.dark,
        colorScheme: ColorScheme.fromSeed(
          seedColor: _primaryColor,
          brightness: Brightness.dark,
          secondary: _secondaryColor,
          error: _errorColor,
        ),
        textTheme: GoogleFonts.interTextTheme(ThemeData.dark().textTheme),
      );
}
```

## ขั้นตอนที่ 3723: Router Configuration

```dart
// lib/core/router/app_router.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:food_admin_dashboard/features/dashboard/presentation/pages/dashboard_page.dart';
import 'package:food_admin_dashboard/features/orders/presentation/pages/orders_page.dart';
import 'package:food_admin_dashboard/features/restaurants/presentation/pages/restaurants_page.dart';
import 'package:food_admin_dashboard/features/users/presentation/pages/users_page.dart';
import 'package:food_admin_dashboard/features/analytics/presentation/pages/analytics_page.dart';
import 'package:food_admin_dashboard/features/auth/presentation/pages/login_page.dart';
import 'package:food_admin_dashboard/core/widgets/admin_scaffold.dart';

part 'app_router.g.dart';

@riverpod
GoRouter appRouter(AppRouterRef ref) {
  return GoRouter(
    initialLocation: '/dashboard',
    redirect: (context, state) {
      // Auth check logic here
      return null;
    },
    routes: [
      GoRoute(
        path: '/login',
        builder: (context, state) => const LoginPage(),
      ),
      ShellRoute(
        builder: (context, state, child) => AdminScaffold(child: child),
        routes: [
          GoRoute(
            path: '/dashboard',
            builder: (context, state) => const DashboardPage(),
          ),
          GoRoute(
            path: '/orders',
            builder: (context, state) => const OrdersPage(),
            routes: [
              GoRoute(
                path: ':id',
                builder: (context, state) => OrderDetailPage(
                  orderId: state.pathParameters['id']!,
                ),
              ),
            ],
          ),
          GoRoute(
            path: '/restaurants',
            builder: (context, state) => const RestaurantsPage(),
            routes: [
              GoRoute(
                path: 'new',
                builder: (context, state) => const RestaurantFormPage(),
              ),
              GoRoute(
                path: ':id/edit',
                builder: (context, state) => RestaurantFormPage(
                  restaurantId: state.pathParameters['id'],
                ),
              ),
            ],
          ),
          GoRoute(
            path: '/users',
            builder: (context, state) => const UsersPage(),
          ),
          GoRoute(
            path: '/analytics',
            builder: (context, state) => const AnalyticsPage(),
          ),
        ],
      ),
    ],
  );
}
```

## ขั้นตอนที่ 3724: Admin Scaffold Layout

```dart
// lib/core/widgets/admin_scaffold.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

class AdminScaffold extends ConsumerStatefulWidget {
  final Widget child;

  const AdminScaffold({super.key, required this.child});

  @override
  ConsumerState<AdminScaffold> createState() => _AdminScaffoldState();
}

class _AdminScaffoldState extends ConsumerState<AdminScaffold> {
  bool _isSidebarExpanded = true;

  static const _navItems = [
    _NavItem(icon: Icons.dashboard_rounded, label: 'Dashboard', path: '/dashboard'),
    _NavItem(icon: Icons.receipt_long_rounded, label: 'Orders', path: '/orders'),
    _NavItem(icon: Icons.restaurant_rounded, label: 'Restaurants', path: '/restaurants'),
    _NavItem(icon: Icons.people_rounded, label: 'Users', path: '/users'),
    _NavItem(icon: Icons.analytics_rounded, label: 'Analytics', path: '/analytics'),
  ];

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final screenWidth = MediaQuery.of(context).size.width;
    final isSmallScreen = screenWidth < 800;

    if (isSmallScreen) {
      return Scaffold(
        appBar: _buildAppBar(theme),
        drawer: _buildDrawer(theme),
        body: widget.child,
      );
    }

    return Scaffold(
      body: Row(
        children: [
          _buildSidebar(theme),
          Expanded(
            child: Column(
              children: [
                _buildTopBar(theme),
                Expanded(child: widget.child),
              ],
            ),
          ),
        ],
      ),
    );
  }

  AppBar _buildAppBar(ThemeData theme) {
    return AppBar(
      title: Row(
        children: [
          Icon(Icons.local_dining, color: theme.colorScheme.primary),
          const SizedBox(width: 8),
          const Text('FoodAdmin'),
        ],
      ),
    );
  }

  Widget _buildSidebar(ThemeData theme) {
    final currentLocation = GoRouterState.of(context).matchedLocation;
    final sidebarWidth = _isSidebarExpanded ? 240.0 : 72.0;

    return AnimatedContainer(
      duration: const Duration(milliseconds: 200),
      width: sidebarWidth,
      decoration: BoxDecoration(
        color: theme.colorScheme.surface,
        border: Border(
          right: BorderSide(
            color: theme.colorScheme.outlineVariant,
            width: 1,
          ),
        ),
      ),
      child: Column(
        children: [
          _buildSidebarHeader(theme),
          const Divider(height: 1),
          Expanded(
            child: ListView.builder(
              padding: const EdgeInsets.symmetric(vertical: 8),
              itemCount: _navItems.length,
              itemBuilder: (context, index) {
                final item = _navItems[index];
                final isActive = currentLocation.startsWith(item.path);
                return _SidebarNavItem(
                  item: item,
                  isActive: isActive,
                  isExpanded: _isSidebarExpanded,
                  onTap: () => context.go(item.path),
                );
              },
            ),
          ),
          const Divider(height: 1),
          _buildSidebarFooter(theme),
        ],
      ),
    );
  }

  Widget _buildSidebarHeader(ThemeData theme) {
    return Container(
      height: 64,
      padding: const EdgeInsets.symmetric(horizontal: 16),
      child: Row(
        children: [
          Icon(
            Icons.local_dining,
            color: theme.colorScheme.primary,
            size: 28,
          ),
          if (_isSidebarExpanded) ...[
            const SizedBox(width: 12),
            Text(
              'FoodAdmin',
              style: theme.textTheme.titleLarge?.copyWith(
                fontWeight: FontWeight.bold,
                color: theme.colorScheme.primary,
              ),
            ),
            const Spacer(),
          ],
          IconButton(
            icon: Icon(
              _isSidebarExpanded
                  ? Icons.chevron_left_rounded
                  : Icons.chevron_right_rounded,
            ),
            onPressed: () {
              setState(() => _isSidebarExpanded = !_isSidebarExpanded);
            },
          ),
        ],
      ),
    );
  }

  Widget _buildSidebarFooter(ThemeData theme) {
    return Padding(
      padding: const EdgeInsets.all(8),
      child: ListTile(
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(8)),
        leading: const CircleAvatar(
          radius: 16,
          child: Icon(Icons.person, size: 20),
        ),
        title: _isSidebarExpanded ? const Text('Admin User') : null,
        subtitle: _isSidebarExpanded ? const Text('admin@food.app') : null,
        trailing: _isSidebarExpanded
            ? IconButton(
                icon: const Icon(Icons.logout_rounded, size: 20),
                onPressed: () {},
              )
            : null,
      ),
    );
  }

  Widget _buildTopBar(ThemeData theme) {
    return Container(
      height: 64,
      padding: const EdgeInsets.symmetric(horizontal: 24),
      decoration: BoxDecoration(
        color: theme.colorScheme.surface,
        border: Border(
          bottom: BorderSide(
            color: theme.colorScheme.outlineVariant,
          ),
        ),
      ),
      child: Row(
        children: [
          Expanded(
            child: SearchAnchor(
              builder: (context, controller) => SearchBar(
                controller: controller,
                padding: const MaterialStatePropertyAll(
                  EdgeInsets.symmetric(horizontal: 16),
                ),
                hintText: 'Search orders, restaurants, users...',
                leading: const Icon(Icons.search),
                constraints: const BoxConstraints(
                  maxWidth: 400,
                  minHeight: 40,
                  maxHeight: 40,
                ),
                onTap: () => controller.openView(),
                onChanged: (_) => controller.openView(),
              ),
              suggestionsBuilder: (context, controller) => [],
            ),
          ),
          const SizedBox(width: 16),
          IconButton(
            icon: const Icon(Icons.notifications_outlined),
            onPressed: () {},
          ),
          const SizedBox(width: 8),
          IconButton(
            icon: const Icon(Icons.brightness_6_outlined),
            onPressed: () {},
          ),
        ],
      ),
    );
  }

  Widget _buildDrawer(ThemeData theme) {
    return const Drawer(child: SizedBox());
  }
}

class _NavItem {
  final IconData icon;
  final String label;
  final String path;

  const _NavItem({
    required this.icon,
    required this.label,
    required this.path,
  });
}

class _SidebarNavItem extends StatelessWidget {
  final _NavItem item;
  final bool isActive;
  final bool isExpanded;
  final VoidCallback onTap;

  const _SidebarNavItem({
    required this.item,
    required this.isActive,
    required this.isExpanded,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
      child: ListTile(
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(8)),
        selected: isActive,
        selectedTileColor: theme.colorScheme.primaryContainer,
        leading: Icon(
          item.icon,
          color: isActive ? theme.colorScheme.primary : null,
        ),
        title: isExpanded
            ? Text(
                item.label,
                style: TextStyle(
                  color: isActive ? theme.colorScheme.primary : null,
                  fontWeight: isActive ? FontWeight.w600 : null,
                ),
              )
            : null,
        onTap: onTap,
        dense: true,
      ),
    );
  }
}
```

## ขั้นตอนที่ 3725: Dashboard Overview Page

```dart
// lib/features/dashboard/presentation/pages/dashboard_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:fl_chart/fl_chart.dart';
import 'package:food_admin_dashboard/features/dashboard/presentation/widgets/stat_card.dart';
import 'package:food_admin_dashboard/features/dashboard/presentation/widgets/revenue_chart.dart';
import 'package:food_admin_dashboard/features/dashboard/presentation/widgets/order_status_chart.dart';
import 'package:food_admin_dashboard/features/dashboard/presentation/widgets/recent_orders_widget.dart';

class DashboardPage extends ConsumerWidget {
  const DashboardPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      backgroundColor: Theme.of(context).colorScheme.background,
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(24),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            _buildHeader(context),
            const SizedBox(height: 24),
            _buildStatCards(context),
            const SizedBox(height: 24),
            _buildChartSection(context),
            const SizedBox(height: 24),
            const RecentOrdersWidget(),
          ],
        ),
      ),
    );
  }

  Widget _buildHeader(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceBetween,
      children: [
        Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Dashboard',
              style: Theme.of(context).textTheme.headlineMedium?.copyWith(
                    fontWeight: FontWeight.bold,
                  ),
            ),
            Text(
              'Welcome back, Admin! Here\'s what\'s happening.',
              style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                    color: Theme.of(context).colorScheme.onSurfaceVariant,
                  ),
            ),
          ],
        ),
        FilledButton.icon(
          onPressed: () {},
          icon: const Icon(Icons.download_rounded),
          label: const Text('Export Report'),
        ),
      ],
    );
  }

  Widget _buildStatCards(BuildContext context) {
    const stats = [
      _StatData(
        title: 'Total Revenue',
        value: '฿284,592',
        change: '+12.5%',
        isPositive: true,
        icon: Icons.attach_money_rounded,
        color: Color(0xFF6C63FF),
      ),
      _StatData(
        title: 'Total Orders',
        value: '3,842',
        change: '+8.1%',
        isPositive: true,
        icon: Icons.receipt_long_rounded,
        color: Color(0xFF03DAC6),
      ),
      _StatData(
        title: 'Active Restaurants',
        value: '128',
        change: '+3',
        isPositive: true,
        icon: Icons.restaurant_rounded,
        color: Color(0xFFFF6B6B),
      ),
      _StatData(
        title: 'New Users',
        value: '567',
        change: '-2.3%',
        isPositive: false,
        icon: Icons.people_rounded,
        color: Color(0xFFFFA726),
      ),
    ];

    return GridView.builder(
      shrinkWrap: true,
      physics: const NeverScrollableScrollPhysics(),
      gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
        maxCrossAxisExtent: 300,
        mainAxisExtent: 120,
        crossAxisSpacing: 16,
        mainAxisSpacing: 16,
      ),
      itemCount: stats.length,
      itemBuilder: (context, index) => StatCard(data: stats[index]),
    );
  }

  Widget _buildChartSection(BuildContext context) {
    return Row(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        const Expanded(flex: 2, child: RevenueChart()),
        const SizedBox(width: 16),
        Expanded(child: OrderStatusChart()),
      ],
    );
  }
}

class _StatData {
  final String title;
  final String value;
  final String change;
  final bool isPositive;
  final IconData icon;
  final Color color;

  const _StatData({
    required this.title,
    required this.value,
    required this.change,
    required this.isPositive,
    required this.icon,
    required this.color,
  });
}
```

## ขั้นตอนที่ 3726: Stat Card Widget

```dart
// lib/features/dashboard/presentation/widgets/stat_card.dart
import 'package:flutter/material.dart';

class StatCard extends StatelessWidget {
  final dynamic data;

  const StatCard({super.key, required this.data});

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          mainAxisAlignment: MainAxisAlignment.spaceBetween,
          children: [
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text(
                  data.title,
                  style: theme.textTheme.bodySmall?.copyWith(
                    color: theme.colorScheme.onSurfaceVariant,
                  ),
                ),
                Container(
                  padding: const EdgeInsets.all(8),
                  decoration: BoxDecoration(
                    color: data.color.withOpacity(0.12),
                    borderRadius: BorderRadius.circular(8),
                  ),
                  child: Icon(data.icon, color: data.color, size: 18),
                ),
              ],
            ),
            Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  data.value,
                  style: theme.textTheme.headlineSmall?.copyWith(
                    fontWeight: FontWeight.bold,
                  ),
                ),
                const SizedBox(height: 4),
                Row(
                  children: [
                    Icon(
                      data.isPositive
                          ? Icons.trending_up_rounded
                          : Icons.trending_down_rounded,
                      size: 14,
                      color: data.isPositive ? Colors.green : Colors.red,
                    ),
                    const SizedBox(width: 4),
                    Text(
                      data.change,
                      style: theme.textTheme.bodySmall?.copyWith(
                        color: data.isPositive ? Colors.green : Colors.red,
                        fontWeight: FontWeight.w500,
                      ),
                    ),
                    const SizedBox(width: 4),
                    Text(
                      'vs last month',
                      style: theme.textTheme.bodySmall?.copyWith(
                        color: theme.colorScheme.onSurfaceVariant,
                      ),
                    ),
                  ],
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

## ขั้นตอนที่ 3727: Revenue Chart with fl_chart

```dart
// lib/features/dashboard/presentation/widgets/revenue_chart.dart
import 'package:flutter/material.dart';
import 'package:fl_chart/fl_chart.dart';

class RevenueChart extends StatefulWidget {
  const RevenueChart({super.key});

  @override
  State<RevenueChart> createState() => _RevenueChartState();
}

class _RevenueChartState extends State<RevenueChart> {
  int _touchedIndex = -1;
  String _selectedPeriod = '7D';

  final _periods = ['7D', '1M', '3M', '1Y'];

  final _weeklyData = [
    FlSpot(0, 12000),
    FlSpot(1, 18500),
    FlSpot(2, 14200),
    FlSpot(3, 22800),
    FlSpot(4, 19600),
    FlSpot(5, 28400),
    FlSpot(6, 31200),
  ];

  final _monthLabels = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'];

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      'Revenue Overview',
                      style: theme.textTheme.titleMedium?.copyWith(
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                    Text(
                      'Total: ฿284,592',
                      style: theme.textTheme.bodySmall?.copyWith(
                        color: theme.colorScheme.onSurfaceVariant,
                      ),
                    ),
                  ],
                ),
                SegmentedButton<String>(
                  segments: _periods
                      .map((p) => ButtonSegment(value: p, label: Text(p)))
                      .toList(),
                  selected: {_selectedPeriod},
                  onSelectionChanged: (selection) {
                    setState(() => _selectedPeriod = selection.first);
                  },
                  style: const ButtonStyle(
                    visualDensity: VisualDensity.compact,
                  ),
                ),
              ],
            ),
            const SizedBox(height: 20),
            SizedBox(
              height: 200,
              child: LineChart(
                LineChartData(
                  gridData: FlGridData(
                    show: true,
                    drawVerticalLine: false,
                    horizontalInterval: 10000,
                    getDrawingHorizontalLine: (value) => FlLine(
                      color: theme.colorScheme.outlineVariant,
                      strokeWidth: 1,
                      dashArray: [5, 5],
                    ),
                  ),
                  titlesData: FlTitlesData(
                    bottomTitles: AxisTitles(
                      sideTitles: SideTitles(
                        showTitles: true,
                        getTitlesWidget: (value, meta) {
                          final index = value.toInt();
                          if (index < 0 || index >= _monthLabels.length) {
                            return const SizedBox();
                          }
                          return Padding(
                            padding: const EdgeInsets.only(top: 8),
                            child: Text(
                              _monthLabels[index],
                              style: theme.textTheme.bodySmall,
                            ),
                          );
                        },
                        reservedSize: 32,
                      ),
                    ),
                    leftTitles: AxisTitles(
                      sideTitles: SideTitles(
                        showTitles: true,
                        reservedSize: 48,
                        getTitlesWidget: (value, meta) {
                          if (value == 0) return const SizedBox();
                          return Text(
                            '${(value / 1000).toStringAsFixed(0)}K',
                            style: theme.textTheme.bodySmall,
                          );
                        },
                      ),
                    ),
                    topTitles: const AxisTitles(
                      sideTitles: SideTitles(showTitles: false),
                    ),
                    rightTitles: const AxisTitles(
                      sideTitles: SideTitles(showTitles: false),
                    ),
                  ),
                  borderData: FlBorderData(show: false),
                  lineBarsData: [
                    LineChartBarData(
                      spots: _weeklyData,
                      isCurved: true,
                      color: theme.colorScheme.primary,
                      barWidth: 3,
                      isStrokeCapRound: true,
                      dotData: FlDotData(
                        show: true,
                        getDotPainter: (spot, percent, barData, index) =>
                            FlDotCirclePainter(
                          radius: 4,
                          color: theme.colorScheme.primary,
                          strokeWidth: 2,
                          strokeColor: theme.colorScheme.surface,
                        ),
                      ),
                      belowBarData: BarAreaData(
                        show: true,
                        gradient: LinearGradient(
                          colors: [
                            theme.colorScheme.primary.withOpacity(0.3),
                            theme.colorScheme.primary.withOpacity(0.0),
                          ],
                          begin: Alignment.topCenter,
                          end: Alignment.bottomCenter,
                        ),
                      ),
                    ),
                  ],
                  lineTouchData: LineTouchData(
                    touchTooltipData: LineTouchTooltipData(
                      tooltipBgColor: theme.colorScheme.inverseSurface,
                      getTooltipItems: (touchedSpots) {
                        return touchedSpots.map((spot) {
                          return LineTooltipItem(
                            '฿${(spot.y / 1000).toStringAsFixed(1)}K',
                            TextStyle(
                              color: theme.colorScheme.onInverseSurface,
                              fontWeight: FontWeight.bold,
                            ),
                          );
                        }).toList();
                      },
                    ),
                  ),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

## ขั้นตอนที่ 3728: Order Status Pie Chart

```dart
// lib/features/dashboard/presentation/widgets/order_status_chart.dart
import 'package:flutter/material.dart';
import 'package:fl_chart/fl_chart.dart';

class OrderStatusChart extends StatefulWidget {
  const OrderStatusChart({super.key});

  @override
  State<OrderStatusChart> createState() => _OrderStatusChartState();
}

class _OrderStatusChartState extends State<OrderStatusChart> {
  int _touchedIndex = -1;

  final _statusData = [
    _StatusData('Delivered', 1842, Color(0xFF4CAF50)),
    _StatusData('Processing', 634, Color(0xFF2196F3)),
    _StatusData('Cancelled', 248, Color(0xFFF44336)),
    _StatusData('Pending', 421, Color(0xFFFF9800)),
  ];

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final total = _statusData.fold<int>(0, (sum, d) => sum + d.count);

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Order Status',
              style: theme.textTheme.titleMedium?.copyWith(
                fontWeight: FontWeight.bold,
              ),
            ),
            Text(
              'Total: $total orders',
              style: theme.textTheme.bodySmall?.copyWith(
                color: theme.colorScheme.onSurfaceVariant,
              ),
            ),
            const SizedBox(height: 20),
            SizedBox(
              height: 160,
              child: PieChart(
                PieChartData(
                  sections: _statusData.asMap().entries.map((entry) {
                    final index = entry.key;
                    final data = entry.value;
                    final isTouched = _touchedIndex == index;
                    return PieChartSectionData(
                      color: data.color,
                      value: data.count.toDouble(),
                      title: '${(data.count / total * 100).toStringAsFixed(0)}%',
                      radius: isTouched ? 65 : 55,
                      titleStyle: TextStyle(
                        fontSize: isTouched ? 13 : 11,
                        fontWeight: FontWeight.bold,
                        color: Colors.white,
                      ),
                    );
                  }).toList(),
                  pieTouchData: PieTouchData(
                    touchCallback: (event, pieTouchResponse) {
                      setState(() {
                        if (!event.isInterestedForInteractions ||
                            pieTouchResponse == null ||
                            pieTouchResponse.touchedSection == null) {
                          _touchedIndex = -1;
                          return;
                        }
                        _touchedIndex = pieTouchResponse
                            .touchedSection!.touchedSectionIndex;
                      });
                    },
                  ),
                  sectionsSpace: 2,
                  centerSpaceRadius: 32,
                ),
              ),
            ),
            const SizedBox(height: 16),
            ..._statusData.map(
              (data) => Padding(
                padding: const EdgeInsets.symmetric(vertical: 4),
                child: Row(
                  children: [
                    Container(
                      width: 12,
                      height: 12,
                      decoration: BoxDecoration(
                        color: data.color,
                        shape: BoxShape.circle,
                      ),
                    ),
                    const SizedBox(width: 8),
                    Expanded(
                      child: Text(
                        data.status,
                        style: theme.textTheme.bodySmall,
                      ),
                    ),
                    Text(
                      data.count.toString(),
                      style: theme.textTheme.bodySmall?.copyWith(
                        fontWeight: FontWeight.w600,
                      ),
                    ),
                  ],
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _StatusData {
  final String status;
  final int count;
  final Color color;

  const _StatusData(this.status, this.count, this.color);
}
```

## ขั้นตอนที่ 3729: Orders Management Page with DataTable2

```dart
// lib/features/orders/presentation/pages/orders_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:data_table_2/data_table_2.dart';
import 'package:intl/intl.dart';
import 'package:food_admin_dashboard/features/orders/domain/entities/order.dart';
import 'package:food_admin_dashboard/features/orders/presentation/providers/orders_provider.dart';

class OrdersPage extends ConsumerStatefulWidget {
  const OrdersPage({super.key});

  @override
  ConsumerState<OrdersPage> createState() => _OrdersPageState();
}

class _OrdersPageState extends ConsumerState<OrdersPage> {
  String _statusFilter = 'All';
  String _searchQuery = '';
  DateTimeRange? _dateRange;
  int _sortColumnIndex = 0;
  bool _sortAscending = false;
  final _searchController = TextEditingController();

  static const _statusOptions = ['All', 'Pending', 'Processing', 'Delivered', 'Cancelled'];

  @override
  void dispose() {
    _searchController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final ordersAsync = ref.watch(filteredOrdersProvider(
      status: _statusFilter == 'All' ? null : _statusFilter,
      search: _searchQuery.isEmpty ? null : _searchQuery,
    ));

    return Padding(
      padding: const EdgeInsets.all(24),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          _buildPageHeader(theme),
          const SizedBox(height: 16),
          _buildFiltersRow(theme),
          const SizedBox(height: 16),
          Expanded(
            child: Card(
              margin: EdgeInsets.zero,
              child: ordersAsync.when(
                data: (orders) => _buildOrdersTable(context, orders),
                loading: () => const Center(child: CircularProgressIndicator()),
                error: (err, stack) => Center(child: Text('Error: $err')),
              ),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildPageHeader(ThemeData theme) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceBetween,
      children: [
        Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Order Management',
              style: theme.textTheme.headlineSmall?.copyWith(
                fontWeight: FontWeight.bold,
              ),
            ),
            Text(
              'Track and manage all customer orders',
              style: theme.textTheme.bodyMedium?.copyWith(
                color: theme.colorScheme.onSurfaceVariant,
              ),
            ),
          ],
        ),
        Row(
          children: [
            OutlinedButton.icon(
              onPressed: _showDateRangePicker,
              icon: const Icon(Icons.calendar_today_rounded, size: 16),
              label: Text(
                _dateRange == null
                    ? 'Date Range'
                    : '${DateFormat('MMM dd').format(_dateRange!.start)} - '
                        '${DateFormat('MMM dd').format(_dateRange!.end)}',
              ),
            ),
            const SizedBox(width: 8),
            FilledButton.icon(
              onPressed: () {},
              icon: const Icon(Icons.download_rounded, size: 16),
              label: const Text('Export CSV'),
            ),
          ],
        ),
      ],
    );
  }

  Widget _buildFiltersRow(ThemeData theme) {
    return Row(
      children: [
        Expanded(
          flex: 2,
          child: TextField(
            controller: _searchController,
            decoration: InputDecoration(
              hintText: 'Search by order ID, customer name...',
              prefixIcon: const Icon(Icons.search_rounded),
              border: OutlineInputBorder(
                borderRadius: BorderRadius.circular(8),
              ),
              contentPadding: const EdgeInsets.symmetric(vertical: 8, horizontal: 12),
              isDense: true,
              suffixIcon: _searchQuery.isNotEmpty
                  ? IconButton(
                      icon: const Icon(Icons.clear_rounded, size: 18),
                      onPressed: () {
                        _searchController.clear();
                        setState(() => _searchQuery = '');
                      },
                    )
                  : null,
            ),
            onChanged: (value) => setState(() => _searchQuery = value),
          ),
        ),
        const SizedBox(width: 12),
        Expanded(
          child: DropdownButtonFormField<String>(
            value: _statusFilter,
            decoration: InputDecoration(
              labelText: 'Status',
              border: OutlineInputBorder(
                borderRadius: BorderRadius.circular(8),
              ),
              contentPadding: const EdgeInsets.symmetric(vertical: 8, horizontal: 12),
              isDense: true,
            ),
            items: _statusOptions
                .map((s) => DropdownMenuItem(value: s, child: Text(s)))
                .toList(),
            onChanged: (value) => setState(() => _statusFilter = value ?? 'All'),
          ),
        ),
        const SizedBox(width: 12),
        TextButton.icon(
          onPressed: _resetFilters,
          icon: const Icon(Icons.refresh_rounded, size: 16),
          label: const Text('Reset'),
        ),
      ],
    );
  }

  Widget _buildOrdersTable(BuildContext context, List<Order> orders) {
    final theme = Theme.of(context);
    return DataTable2(
      columnSpacing: 12,
      horizontalMargin: 16,
      minWidth: 800,
      sortColumnIndex: _sortColumnIndex,
      sortAscending: _sortAscending,
      headingRowColor: MaterialStatePropertyAll(
        theme.colorScheme.surfaceVariant.withOpacity(0.5),
      ),
      columns: [
        DataColumn2(
          label: const Text('Order ID'),
          size: ColumnSize.S,
          onSort: (index, asc) => setState(() {
            _sortColumnIndex = index;
            _sortAscending = asc;
          }),
        ),
        DataColumn2(
          label: const Text('Customer'),
          size: ColumnSize.M,
          onSort: (index, asc) => setState(() {
            _sortColumnIndex = index;
            _sortAscending = asc;
          }),
        ),
        DataColumn2(
          label: const Text('Restaurant'),
          size: ColumnSize.M,
        ),
        DataColumn2(
          label: const Text('Amount'),
          numeric: true,
          size: ColumnSize.S,
          onSort: (index, asc) => setState(() {
            _sortColumnIndex = index;
            _sortAscending = asc;
          }),
        ),
        DataColumn2(
          label: const Text('Status'),
          size: ColumnSize.S,
        ),
        DataColumn2(
          label: const Text('Date'),
          size: ColumnSize.M,
          onSort: (index, asc) => setState(() {
            _sortColumnIndex = index;
            _sortAscending = asc;
          }),
        ),
        const DataColumn2(
          label: Text('Actions'),
          size: ColumnSize.S,
        ),
      ],
      rows: orders.map((order) => _buildOrderRow(context, order)).toList(),
      empty: const Center(
        child: Padding(
          padding: EdgeInsets.all(32),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Icon(Icons.inbox_rounded, size: 48, color: Colors.grey),
              SizedBox(height: 12),
              Text('No orders found'),
            ],
          ),
        ),
      ),
    );
  }

  DataRow2 _buildOrderRow(BuildContext context, Order order) {
    final theme = Theme.of(context);
    return DataRow2(
      onTap: () {},
      cells: [
        DataCell(Text(
          '#${order.id.substring(0, 8).toUpperCase()}',
          style: const TextStyle(fontWeight: FontWeight.w500),
        )),
        DataCell(Row(
          children: [
            CircleAvatar(
              radius: 14,
              backgroundColor: theme.colorScheme.primaryContainer,
              child: Text(
                order.customerName.substring(0, 1).toUpperCase(),
                style: TextStyle(
                  fontSize: 12,
                  color: theme.colorScheme.primary,
                ),
              ),
            ),
            const SizedBox(width: 8),
            Expanded(
              child: Text(
                order.customerName,
                overflow: TextOverflow.ellipsis,
              ),
            ),
          ],
        )),
        DataCell(Text(order.restaurantName, overflow: TextOverflow.ellipsis)),
        DataCell(Text(
          '฿${NumberFormat('#,##0.00').format(order.totalAmount)}',
          style: const TextStyle(fontWeight: FontWeight.w600),
        )),
        DataCell(_buildStatusChip(order.status)),
        DataCell(Text(
          DateFormat('MMM dd, HH:mm').format(order.createdAt),
          style: TextStyle(color: theme.colorScheme.onSurfaceVariant),
        )),
        DataCell(Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            IconButton(
              icon: const Icon(Icons.visibility_rounded, size: 18),
              onPressed: () {},
              visualDensity: VisualDensity.compact,
            ),
            IconButton(
              icon: const Icon(Icons.edit_rounded, size: 18),
              onPressed: () {},
              visualDensity: VisualDensity.compact,
            ),
          ],
        )),
      ],
    );
  }

  Widget _buildStatusChip(String status) {
    final (color, bgColor) = switch (status) {
      'Delivered' => (Colors.green.shade700, Colors.green.shade50),
      'Processing' => (Colors.blue.shade700, Colors.blue.shade50),
      'Pending' => (Colors.orange.shade700, Colors.orange.shade50),
      'Cancelled' => (Colors.red.shade700, Colors.red.shade50),
      _ => (Colors.grey.shade700, Colors.grey.shade50),
    };

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
      decoration: BoxDecoration(
        color: bgColor,
        borderRadius: BorderRadius.circular(12),
      ),
      child: Text(
        status,
        style: TextStyle(
          color: color,
          fontSize: 12,
          fontWeight: FontWeight.w500,
        ),
      ),
    );
  }

  Future<void> _showDateRangePicker() async {
    final range = await showDateRangePicker(
      context: context,
      firstDate: DateTime(2023),
      lastDate: DateTime.now(),
      initialDateRange: _dateRange,
    );
    if (range != null) setState(() => _dateRange = range);
  }

  void _resetFilters() {
    _searchController.clear();
    setState(() {
      _statusFilter = 'All';
      _searchQuery = '';
      _dateRange = null;
    });
  }
}
```

## ขั้นตอนที่ 3730: Restaurant Management CRUD

```dart
// lib/features/restaurants/presentation/pages/restaurants_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:food_admin_dashboard/features/restaurants/domain/entities/restaurant.dart';
import 'package:food_admin_dashboard/features/restaurants/presentation/providers/restaurants_provider.dart';

class RestaurantsPage extends ConsumerStatefulWidget {
  const RestaurantsPage({super.key});

  @override
  ConsumerState<RestaurantsPage> createState() => _RestaurantsPageState();
}

class _RestaurantsPageState extends ConsumerState<RestaurantsPage> {
  String _searchQuery = '';
  String _categoryFilter = 'All';

  @override
  Widget build(BuildContext context) {
    final restaurantsAsync = ref.watch(restaurantsListProvider);
    final theme = Theme.of(context);

    return Padding(
      padding: const EdgeInsets.all(24),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          _buildHeader(context),
          const SizedBox(height: 16),
          _buildFilters(context),
          const SizedBox(height: 16),
          Expanded(
            child: restaurantsAsync.when(
              data: (restaurants) => _buildGrid(context, restaurants),
              loading: () => const Center(child: CircularProgressIndicator()),
              error: (e, _) => Center(child: Text('Error: $e')),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildHeader(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceBetween,
      children: [
        Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Restaurant Management',
              style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                    fontWeight: FontWeight.bold,
                  ),
            ),
            Text(
              'Manage all partner restaurants',
              style: Theme.of(context).textTheme.bodyMedium?.copyWith(
                    color: Theme.of(context).colorScheme.onSurfaceVariant,
                  ),
            ),
          ],
        ),
        FilledButton.icon(
          onPressed: () => context.go('/restaurants/new'),
          icon: const Icon(Icons.add_rounded),
          label: const Text('Add Restaurant'),
        ),
      ],
    );
  }

  Widget _buildFilters(BuildContext context) {
    return Row(
      children: [
        Expanded(
          child: TextField(
            decoration: InputDecoration(
              hintText: 'Search restaurants...',
              prefixIcon: const Icon(Icons.search_rounded),
              border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
              isDense: true,
              contentPadding: const EdgeInsets.symmetric(vertical: 10, horizontal: 12),
            ),
            onChanged: (value) => setState(() => _searchQuery = value),
          ),
        ),
        const SizedBox(width: 12),
        FilterChip.elevated(
          label: const Text('All'),
          selected: _categoryFilter == 'All',
          onSelected: (_) => setState(() => _categoryFilter = 'All'),
        ),
        const SizedBox(width: 8),
        ...['Thai', 'Japanese', 'Italian', 'Fast Food'].map(
          (cat) => Padding(
            padding: const EdgeInsets.only(right: 8),
            child: FilterChip.elevated(
              label: Text(cat),
              selected: _categoryFilter == cat,
              onSelected: (_) => setState(() => _categoryFilter = cat),
            ),
          ),
        ),
      ],
    );
  }

  Widget _buildGrid(BuildContext context, List<Restaurant> restaurants) {
    final filtered = restaurants.where((r) {
      final matchSearch = _searchQuery.isEmpty ||
          r.name.toLowerCase().contains(_searchQuery.toLowerCase());
      final matchCategory =
          _categoryFilter == 'All' || r.category == _categoryFilter;
      return matchSearch && matchCategory;
    }).toList();

    return GridView.builder(
      gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
        maxCrossAxisExtent: 360,
        mainAxisExtent: 220,
        crossAxisSpacing: 16,
        mainAxisSpacing: 16,
      ),
      itemCount: filtered.length,
      itemBuilder: (context, index) =>
          _RestaurantCard(restaurant: filtered[index]),
    );
  }
}

class _RestaurantCard extends ConsumerWidget {
  final Restaurant restaurant;

  const _RestaurantCard({required this.restaurant});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final theme = Theme.of(context);
    return Card(
      clipBehavior: Clip.antiAlias,
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Container(
            height: 100,
            color: theme.colorScheme.primaryContainer,
            child: Stack(
              children: [
                Center(
                  child: Icon(
                    Icons.restaurant_rounded,
                    size: 48,
                    color: theme.colorScheme.primary.withOpacity(0.3),
                  ),
                ),
                Positioned(
                  top: 8,
                  right: 8,
                  child: _buildStatusBadge(restaurant.isActive),
                ),
              ],
            ),
          ),
          Expanded(
            child: Padding(
              padding: const EdgeInsets.all(12),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Row(
                    children: [
                      Expanded(
                        child: Text(
                          restaurant.name,
                          style: theme.textTheme.titleSmall?.copyWith(
                            fontWeight: FontWeight.bold,
                          ),
                          overflow: TextOverflow.ellipsis,
                        ),
                      ),
                      _buildActionMenu(context, ref),
                    ],
                  ),
                  const SizedBox(height: 4),
                  Text(
                    restaurant.category,
                    style: theme.textTheme.bodySmall?.copyWith(
                      color: theme.colorScheme.onSurfaceVariant,
                    ),
                  ),
                  const Spacer(),
                  Row(
                    children: [
                      Icon(Icons.star_rounded,
                          size: 14, color: Colors.amber.shade600),
                      const SizedBox(width: 4),
                      Text(
                        restaurant.rating.toStringAsFixed(1),
                        style: theme.textTheme.bodySmall,
                      ),
                      const SizedBox(width: 12),
                      Icon(Icons.receipt_long_rounded,
                          size: 14,
                          color: theme.colorScheme.onSurfaceVariant),
                      const SizedBox(width: 4),
                      Text(
                        '${restaurant.totalOrders} orders',
                        style: theme.textTheme.bodySmall,
                      ),
                    ],
                  ),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildStatusBadge(bool isActive) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
      decoration: BoxDecoration(
        color: isActive ? Colors.green : Colors.red,
        borderRadius: BorderRadius.circular(12),
      ),
      child: Text(
        isActive ? 'Active' : 'Inactive',
        style: const TextStyle(
          color: Colors.white,
          fontSize: 11,
          fontWeight: FontWeight.w500,
        ),
      ),
    );
  }

  Widget _buildActionMenu(BuildContext context, WidgetRef ref) {
    return PopupMenuButton<String>(
      padding: EdgeInsets.zero,
      iconSize: 18,
      itemBuilder: (context) => [
        const PopupMenuItem(value: 'edit', child: Text('Edit')),
        const PopupMenuItem(value: 'toggle', child: Text('Toggle Status')),
        const PopupMenuItem(
          value: 'delete',
          child: Text('Delete', style: TextStyle(color: Colors.red)),
        ),
      ],
      onSelected: (value) async {
        switch (value) {
          case 'edit':
            context.go('/restaurants/${restaurant.id}/edit');
          case 'toggle':
            await ref
                .read(restaurantsListProvider.notifier)
                .toggleStatus(restaurant.id);
          case 'delete':
            _showDeleteDialog(context, ref);
        }
      },
    );
  }

  void _showDeleteDialog(BuildContext context, WidgetRef ref) {
    showDialog(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('Delete Restaurant'),
        content: Text(
          'Are you sure you want to delete "${restaurant.name}"? '
          'This action cannot be undone.',
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(ctx),
            child: const Text('Cancel'),
          ),
          FilledButton(
            style: FilledButton.styleFrom(backgroundColor: Colors.red),
            onPressed: () async {
              Navigator.pop(ctx);
              await ref
                  .read(restaurantsListProvider.notifier)
                  .deleteRestaurant(restaurant.id);
            },
            child: const Text('Delete'),
          ),
        ],
      ),
    );
  }
}
```

## ขั้นตอนที่ 3731: User Management with Role-based Access

```dart
// lib/features/users/presentation/pages/users_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:data_table_2/data_table_2.dart';
import 'package:intl/intl.dart';
import 'package:food_admin_dashboard/features/users/domain/entities/app_user.dart';
import 'package:food_admin_dashboard/features/users/presentation/providers/users_provider.dart';

class UsersPage extends ConsumerStatefulWidget {
  const UsersPage({super.key});

  @override
  ConsumerState<UsersPage> createState() => _UsersPageState();
}

class _UsersPageState extends ConsumerState<UsersPage> {
  String _roleFilter = 'All';
  String _searchQuery = '';
  final _searchController = TextEditingController();

  @override
  void dispose() {
    _searchController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final usersAsync = ref.watch(usersListProvider);

    return Padding(
      padding: const EdgeInsets.all(24),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          _buildHeader(context),
          const SizedBox(height: 16),
          _buildFilters(context),
          const SizedBox(height: 16),
          Expanded(
            child: Card(
              margin: EdgeInsets.zero,
              child: usersAsync.when(
                data: (users) {
                  final filtered = _filterUsers(users);
                  return _buildUsersTable(context, filtered);
                },
                loading: () =>
                    const Center(child: CircularProgressIndicator()),
                error: (e, _) => Center(child: Text('Error: $e')),
              ),
            ),
          ),
        ],
      ),
    );
  }

  List<AppUser> _filterUsers(List<AppUser> users) {
    return users.where((user) {
      final matchRole = _roleFilter == 'All' || user.role == _roleFilter;
      final matchSearch = _searchQuery.isEmpty ||
          user.name.toLowerCase().contains(_searchQuery.toLowerCase()) ||
          user.email.toLowerCase().contains(_searchQuery.toLowerCase());
      return matchRole && matchSearch;
    }).toList();
  }

  Widget _buildHeader(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceBetween,
      children: [
        Text(
          'User Management',
          style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                fontWeight: FontWeight.bold,
              ),
        ),
        FilledButton.icon(
          onPressed: () => _showAddUserDialog(context),
          icon: const Icon(Icons.person_add_rounded),
          label: const Text('Add User'),
        ),
      ],
    );
  }

  Widget _buildFilters(BuildContext context) {
    final roles = ['All', 'Customer', 'Restaurant Owner', 'Admin', 'Driver'];
    return Row(
      children: [
        Expanded(
          flex: 2,
          child: TextField(
            controller: _searchController,
            decoration: InputDecoration(
              hintText: 'Search users by name or email...',
              prefixIcon: const Icon(Icons.search_rounded),
              border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
              isDense: true,
              contentPadding:
                  const EdgeInsets.symmetric(vertical: 10, horizontal: 12),
            ),
            onChanged: (v) => setState(() => _searchQuery = v),
          ),
        ),
        const SizedBox(width: 12),
        Expanded(
          child: DropdownButtonFormField<String>(
            value: _roleFilter,
            decoration: InputDecoration(
              labelText: 'Role',
              border: OutlineInputBorder(borderRadius: BorderRadius.circular(8)),
              isDense: true,
              contentPadding:
                  const EdgeInsets.symmetric(vertical: 10, horizontal: 12),
            ),
            items:
                roles.map((r) => DropdownMenuItem(value: r, child: Text(r))).toList(),
            onChanged: (v) => setState(() => _roleFilter = v ?? 'All'),
          ),
        ),
      ],
    );
  }

  Widget _buildUsersTable(BuildContext context, List<AppUser> users) {
    final theme = Theme.of(context);
    return DataTable2(
      columnSpacing: 12,
      horizontalMargin: 16,
      minWidth: 700,
      columns: const [
        DataColumn2(label: Text('User'), size: ColumnSize.L),
        DataColumn2(label: Text('Role'), size: ColumnSize.M),
        DataColumn2(label: Text('Status'), size: ColumnSize.S),
        DataColumn2(label: Text('Joined'), size: ColumnSize.M),
        DataColumn2(label: Text('Orders'), numeric: true, size: ColumnSize.S),
        DataColumn2(label: Text('Actions'), size: ColumnSize.S),
      ],
      rows: users.map((user) {
        return DataRow2(
          cells: [
            DataCell(Row(
              children: [
                CircleAvatar(
                  radius: 16,
                  backgroundColor: theme.colorScheme.primaryContainer,
                  child: Text(
                    user.name.substring(0, 1).toUpperCase(),
                    style: TextStyle(
                      color: theme.colorScheme.primary,
                      fontSize: 14,
                    ),
                  ),
                ),
                const SizedBox(width: 10),
                Expanded(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      Text(
                        user.name,
                        style: const TextStyle(fontWeight: FontWeight.w500),
                        overflow: TextOverflow.ellipsis,
                      ),
                      Text(
                        user.email,
                        style: theme.textTheme.bodySmall?.copyWith(
                          color: theme.colorScheme.onSurfaceVariant,
                        ),
                        overflow: TextOverflow.ellipsis,
                      ),
                    ],
                  ),
                ),
              ],
            )),
            DataCell(_buildRoleChip(user.role)),
            DataCell(_buildActiveIndicator(user.isActive)),
            DataCell(Text(
              DateFormat('MMM dd, yyyy').format(user.createdAt),
              style: theme.textTheme.bodySmall,
            )),
            DataCell(Text(user.totalOrders.toString())),
            DataCell(Row(
              mainAxisSize: MainAxisSize.min,
              children: [
                IconButton(
                  icon: const Icon(Icons.edit_rounded, size: 18),
                  onPressed: () => _showEditRoleDialog(context, user),
                  visualDensity: VisualDensity.compact,
                ),
                IconButton(
                  icon: Icon(
                    user.isActive ? Icons.block_rounded : Icons.check_circle_rounded,
                    size: 18,
                    color: user.isActive ? Colors.red : Colors.green,
                  ),
                  onPressed: () => ref
                      .read(usersListProvider.notifier)
                      .toggleUserStatus(user.id),
                  visualDensity: VisualDensity.compact,
                ),
              ],
            )),
          ],
        );
      }).toList(),
    );
  }

  Widget _buildRoleChip(String role) {
    final (color, bg) = switch (role) {
      'Admin' => (Colors.purple.shade700, Colors.purple.shade50),
      'Restaurant Owner' => (Colors.blue.shade700, Colors.blue.shade50),
      'Driver' => (Colors.orange.shade700, Colors.orange.shade50),
      _ => (Colors.grey.shade700, Colors.grey.shade100),
    };
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
      decoration: BoxDecoration(
        color: bg,
        borderRadius: BorderRadius.circular(12),
      ),
      child: Text(
        role,
        style: TextStyle(color: color, fontSize: 12, fontWeight: FontWeight.w500),
      ),
    );
  }

  Widget _buildActiveIndicator(bool isActive) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        Container(
          width: 8,
          height: 8,
          decoration: BoxDecoration(
            color: isActive ? Colors.green : Colors.grey,
            shape: BoxShape.circle,
          ),
        ),
        const SizedBox(width: 6),
        Text(isActive ? 'Active' : 'Inactive',
            style: const TextStyle(fontSize: 13)),
      ],
    );
  }

  void _showAddUserDialog(BuildContext context) {
    showDialog(
      context: context,
      builder: (ctx) => _UserFormDialog(),
    );
  }

  void _showEditRoleDialog(BuildContext context, AppUser user) {
    showDialog(
      context: context,
      builder: (ctx) => _UserFormDialog(user: user),
    );
  }
}

class _UserFormDialog extends ConsumerStatefulWidget {
  final AppUser? user;

  const _UserFormDialog({this.user});

  @override
  ConsumerState<_UserFormDialog> createState() => _UserFormDialogState();
}

class _UserFormDialogState extends ConsumerState<_UserFormDialog> {
  final _formKey = GlobalKey<FormState>();
  late final TextEditingController _nameController;
  late final TextEditingController _emailController;
  String _selectedRole = 'Customer';

  @override
  void initState() {
    super.initState();
    _nameController = TextEditingController(text: widget.user?.name);
    _emailController = TextEditingController(text: widget.user?.email);
    _selectedRole = widget.user?.role ?? 'Customer';
  }

  @override
  void dispose() {
    _nameController.dispose();
    _emailController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: Text(widget.user == null ? 'Add New User' : 'Edit User'),
      content: Form(
        key: _formKey,
        child: SizedBox(
          width: 400,
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              TextFormField(
                controller: _nameController,
                decoration: const InputDecoration(labelText: 'Full Name'),
                validator: (v) =>
                    v == null || v.isEmpty ? 'Name is required' : null,
              ),
              const SizedBox(height: 12),
              TextFormField(
                controller: _emailController,
                decoration: const InputDecoration(labelText: 'Email'),
                validator: (v) =>
                    v == null || !v.contains('@') ? 'Valid email required' : null,
              ),
              const SizedBox(height: 12),
              DropdownButtonFormField<String>(
                value: _selectedRole,
                decoration: const InputDecoration(labelText: 'Role'),
                items: ['Customer', 'Restaurant Owner', 'Admin', 'Driver']
                    .map((r) => DropdownMenuItem(value: r, child: Text(r)))
                    .toList(),
                onChanged: (v) => setState(() => _selectedRole = v ?? 'Customer'),
              ),
            ],
          ),
        ),
      ),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context),
          child: const Text('Cancel'),
        ),
        FilledButton(
          onPressed: _submit,
          child: Text(widget.user == null ? 'Add' : 'Save'),
        ),
      ],
    );
  }

  void _submit() {
    if (!_formKey.currentState!.validate()) return;
    // Save logic here
    Navigator.pop(context);
  }
}
```

## ขั้นตอนที่ 3732: Analytics Page with Multiple Charts

```dart
// lib/features/analytics/presentation/pages/analytics_page.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:fl_chart/fl_chart.dart';
import 'package:intl/intl.dart';

class AnalyticsPage extends ConsumerWidget {
  const AnalyticsPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return SingleChildScrollView(
      padding: const EdgeInsets.all(24),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(
            'Analytics',
            style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                  fontWeight: FontWeight.bold,
                ),
          ),
          const SizedBox(height: 24),
          const Row(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Expanded(flex: 2, child: _TopRestaurantsChart()),
              SizedBox(width: 16),
              Expanded(child: _HourlyOrdersChart()),
            ],
          ),
          const SizedBox(height: 16),
          const Row(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Expanded(child: _CategoryRevenueChart()),
              SizedBox(width: 16),
              Expanded(child: _CustomerGrowthChart()),
            ],
          ),
        ],
      ),
    );
  }
}

class _TopRestaurantsChart extends StatelessWidget {
  const _TopRestaurantsChart();

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final data = [
      _RestaurantData('Somtam House', 48000, Colors.blue),
      _RestaurantData('Ramen Ichiban', 42000, Colors.green),
      _RestaurantData('Pizza Roma', 38000, Colors.orange),
      _RestaurantData('Pad Thai King', 35000, Colors.purple),
      _RestaurantData('Sushi Bar 88', 29000, Colors.red),
    ];

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Top Restaurants by Revenue',
              style: theme.textTheme.titleMedium
                  ?.copyWith(fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 20),
            SizedBox(
              height: 220,
              child: BarChart(
                BarChartData(
                  alignment: BarChartAlignment.spaceAround,
                  maxY: 55000,
                  gridData: FlGridData(
                    drawVerticalLine: false,
                    getDrawingHorizontalLine: (v) => FlLine(
                      color: theme.colorScheme.outlineVariant,
                      strokeWidth: 1,
                      dashArray: [4, 4],
                    ),
                  ),
                  titlesData: FlTitlesData(
                    leftTitles: AxisTitles(
                      sideTitles: SideTitles(
                        showTitles: true,
                        reservedSize: 48,
                        getTitlesWidget: (v, _) => Text(
                          '${(v / 1000).toInt()}K',
                          style: theme.textTheme.bodySmall,
                        ),
                      ),
                    ),
                    bottomTitles: AxisTitles(
                      sideTitles: SideTitles(
                        showTitles: true,
                        reservedSize: 40,
                        getTitlesWidget: (v, _) {
                          final idx = v.toInt();
                          if (idx < 0 || idx >= data.length) return const SizedBox();
                          return Padding(
                            padding: const EdgeInsets.only(top: 8),
                            child: Text(
                              data[idx].name.split(' ').first,
                              style: theme.textTheme.bodySmall,
                            ),
                          );
                        },
                      ),
                    ),
                    topTitles: const AxisTitles(
                        sideTitles: SideTitles(showTitles: false)),
                    rightTitles: const AxisTitles(
                        sideTitles: SideTitles(showTitles: false)),
                  ),
                  borderData: FlBorderData(show: false),
                  barGroups: data.asMap().entries.map((entry) {
                    return BarChartGroupData(
                      x: entry.key,
                      barRods: [
                        BarChartRodData(
                          toY: entry.value.revenue.toDouble(),
                          color: entry.value.color,
                          width: 32,
                          borderRadius: const BorderRadius.only(
                            topLeft: Radius.circular(4),
                            topRight: Radius.circular(4),
                          ),
                        ),
                      ],
                    );
                  }).toList(),
                  barTouchData: BarTouchData(
                    touchTooltipData: BarTouchTooltipData(
                      tooltipBgColor: theme.colorScheme.inverseSurface,
                      getTooltipItem: (group, groupIndex, rod, rodIndex) {
                        return BarTooltipItem(
                          '${data[group.x].name}\n'
                          '฿${NumberFormat('#,##0').format(rod.toY.toInt())}',
                          TextStyle(
                            color: theme.colorScheme.onInverseSurface,
                            fontSize: 12,
                          ),
                        );
                      },
                    ),
                  ),
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _RestaurantData {
  final String name;
  final int revenue;
  final Color color;

  const _RestaurantData(this.name, this.revenue, this.color);
}

class _HourlyOrdersChart extends StatelessWidget {
  const _HourlyOrdersChart();

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final hourlyData = [2, 1, 0, 0, 1, 3, 8, 15, 22, 28, 32, 38, 45, 42, 35, 30, 36, 48, 55, 52, 44, 35, 20, 8];

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Hourly Order Distribution',
              style: theme.textTheme.titleMedium?.copyWith(fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 20),
            SizedBox(
              height: 220,
              child: LineChart(
                LineChartData(
                  gridData: FlGridData(show: false),
                  borderData: FlBorderData(show: false),
                  titlesData: FlTitlesData(
                    bottomTitles: AxisTitles(
                      sideTitles: SideTitles(
                        showTitles: true,
                        interval: 6,
                        getTitlesWidget: (v, _) {
                          final hour = v.toInt();
                          return Text(
                            '${hour.toString().padLeft(2, '0')}:00',
                            style: theme.textTheme.bodySmall,
                          );
                        },
                        reservedSize: 24,
                      ),
                    ),
                    leftTitles: const AxisTitles(
                        sideTitles: SideTitles(showTitles: false)),
                    topTitles: const AxisTitles(
                        sideTitles: SideTitles(showTitles: false)),
                    rightTitles: const AxisTitles(
                        sideTitles: SideTitles(showTitles: false)),
                  ),
                  lineBarsData: [
                    LineChartBarData(
                      spots: hourlyData.asMap().entries.map((e) =>
                          FlSpot(e.key.toDouble(), e.value.toDouble())).toList(),
                      isCurved: true,
                      color: theme.colorScheme.secondary,
                      barWidth: 2,
                      dotData: const FlDotData(show: false),
                      belowBarData: BarAreaData(
                        show: true,
                        gradient: LinearGradient(
                          colors: [
                            theme.colorScheme.secondary.withOpacity(0.3),
                            theme.colorScheme.secondary.withOpacity(0.0),
                          ],
                          begin: Alignment.topCenter,
                          end: Alignment.bottomCenter,
                        ),
                      ),
                    ),
                  ],
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _CategoryRevenueChart extends StatefulWidget {
  const _CategoryRevenueChart();

  @override
  State<_CategoryRevenueChart> createState() => _CategoryRevenueChartState();
}

class _CategoryRevenueChartState extends State<_CategoryRevenueChart> {
  int _touchedIndex = -1;

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final data = [
      _CategoryData('Thai Food', 35, const Color(0xFF6C63FF)),
      _CategoryData('Japanese', 25, const Color(0xFF03DAC6)),
      _CategoryData('Fast Food', 20, const Color(0xFFFF6B6B)),
      _CategoryData('Italian', 12, const Color(0xFFFFA726)),
      _CategoryData('Others', 8, const Color(0xFF78909C)),
    ];

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Revenue by Category',
              style: theme.textTheme.titleMedium?.copyWith(fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 20),
            Row(
              children: [
                Expanded(
                  child: SizedBox(
                    height: 160,
                    child: PieChart(
                      PieChartData(
                        sectionsSpace: 2,
                        centerSpaceRadius: 40,
                        sections: data.asMap().entries.map((entry) {
                          final isTouched = _touchedIndex == entry.key;
                          return PieChartSectionData(
                            color: entry.value.color,
                            value: entry.value.percentage.toDouble(),
                            title: '${entry.value.percentage}%',
                            radius: isTouched ? 60 : 50,
                            titleStyle: const TextStyle(
                              fontSize: 11,
                              fontWeight: FontWeight.bold,
                              color: Colors.white,
                            ),
                          );
                        }).toList(),
                        pieTouchData: PieTouchData(
                          touchCallback: (event, response) {
                            setState(() {
                              if (response?.touchedSection != null &&
                                  event.isInterestedForInteractions) {
                                _touchedIndex = response!
                                    .touchedSection!.touchedSectionIndex;
                              } else {
                                _touchedIndex = -1;
                              }
                            });
                          },
                        ),
                      ),
                    ),
                  ),
                ),
                const SizedBox(width: 16),
                Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: data.map((d) => Padding(
                    padding: const EdgeInsets.symmetric(vertical: 4),
                    child: Row(
                      children: [
                        Container(
                          width: 12,
                          height: 12,
                          decoration: BoxDecoration(
                            color: d.color,
                            shape: BoxShape.circle,
                          ),
                        ),
                        const SizedBox(width: 8),
                        Text(d.category, style: theme.textTheme.bodySmall),
                        const SizedBox(width: 4),
                        Text(
                          '${d.percentage}%',
                          style: theme.textTheme.bodySmall?.copyWith(
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                      ],
                    ),
                  )).toList(),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}

class _CategoryData {
  final String category;
  final int percentage;
  final Color color;

  const _CategoryData(this.category, this.percentage, this.color);
}

class _CustomerGrowthChart extends StatelessWidget {
  const _CustomerGrowthChart();

  @override
  Widget build(BuildContext context) {
    final theme = Theme.of(context);
    final months = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun'];
    final newUsers = [120, 185, 210, 280, 350, 420];
    final returningUsers = [80, 130, 165, 220, 290, 380];

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Customer Growth',
              style: theme.textTheme.titleMedium?.copyWith(fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            Row(
              children: [
                _LegendDot(color: theme.colorScheme.primary, label: 'New Users'),
                const SizedBox(width: 16),
                _LegendDot(color: theme.colorScheme.secondary, label: 'Returning'),
              ],
            ),
            const SizedBox(height: 16),
            SizedBox(
              height: 160,
              child: LineChart(
                LineChartData(
                  gridData: FlGridData(
                    drawVerticalLine: false,
                    getDrawingHorizontalLine: (_) => FlLine(
                      color: theme.colorScheme.outlineVariant,
                      strokeWidth: 1,
                      dashArray: [4, 4],
                    ),
                  ),
                  titlesData: FlTitlesData(
                    bottomTitles: AxisTitles(
                      sideTitles: SideTitles(
                        showTitles: true,
                        getTitlesWidget: (v, _) {
                          final i = v.toInt();
                          if (i < 0 || i >= months.length) return const SizedBox();
                          return Padding(
                            padding: const EdgeInsets.only(top: 8),
                            child: Text(months[i], style: theme.textTheme.bodySmall),
                          );
                        },
                        reservedSize: 28,
                      ),
                    ),
                    leftTitles: AxisTitles(
                      sideTitles: SideTitles(
                        showTitles: true,
                        reservedSize: 40,
                        getTitlesWidget: (v, _) => Text(
                          v.toInt().toString(),
                          style: theme.textTheme.bodySmall,
                        ),
                      ),
                    ),
                    topTitles: const AxisTitles(sideTitles: SideTitles(showTitles: false)),
                    rightTitles: const AxisTitles(sideTitles: SideTitles(showTitles: false)),
                  ),
                  borderData: FlBorderData(show: false),
                  lineBarsData: [
                    LineChartBarData(
                      spots: newUsers.asMap().entries.map((e) =>
                          FlSpot(e.key.toDouble(), e.value.toDouble())).toList(),
                      isCurved: true,
                      color: theme.colorScheme.primary,
                      barWidth: 2,
                      dotData: const FlDotData(show: false),
                    ),
                    LineChartBarData(
                      spots: returningUsers.asMap().entries.map((e) =>
                          FlSpot(e.key.toDouble(), e.value.toDouble())).toList(),
                      isCurved: true,
                      color: theme.colorScheme.secondary,
                      barWidth: 2,
                      dotData: const FlDotData(show: false),
                    ),
                  ],
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _LegendDot extends StatelessWidget {
  final Color color;
  final String label;

  const _LegendDot({required this.color, required this.label});

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        Container(
          width: 12,
          height: 12,
          decoration: BoxDecoration(color: color, shape: BoxShape.circle),
        ),
        const SizedBox(width: 6),
        Text(label, style: Theme.of(context).textTheme.bodySmall),
      ],
    );
  }
}
```

## ขั้นตอนที่ 3733: Domain Entities

```dart
// lib/features/orders/domain/entities/order.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'order.freezed.dart';
part 'order.g.dart';

enum OrderStatus { pending, processing, delivered, cancelled }

@freezed
class Order with _$Order {
  const factory Order({
    required String id,
    required String customerId,
    required String customerName,
    required String customerPhone,
    required String restaurantId,
    required String restaurantName,
    required List<OrderItem> items,
    required double subtotal,
    required double deliveryFee,
    required double discount,
    required double totalAmount,
    required String status,
    required String deliveryAddress,
    required DateTime createdAt,
    required DateTime updatedAt,
    String? riderId,
    String? riderName,
    DateTime? pickedUpAt,
    DateTime? deliveredAt,
    String? cancelReason,
    String? notes,
  }) = _Order;

  factory Order.fromJson(Map<String, dynamic> json) => _$OrderFromJson(json);
}

@freezed
class OrderItem with _$OrderItem {
  const factory OrderItem({
    required String menuItemId,
    required String name,
    required double price,
    required int quantity,
    List<String>? addons,
    String? notes,
  }) = _OrderItem;

  factory OrderItem.fromJson(Map<String, dynamic> json) =>
      _$OrderItemFromJson(json);
}
```

```dart
// lib/features/restaurants/domain/entities/restaurant.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'restaurant.freezed.dart';
part 'restaurant.g.dart';

@freezed
class Restaurant with _$Restaurant {
  const factory Restaurant({
    required String id,
    required String name,
    required String category,
    required String address,
    required String phone,
    required double rating,
    required int totalOrders,
    required double totalRevenue,
    required bool isActive,
    required bool isOpen,
    required DateTime createdAt,
    String? imageUrl,
    String? description,
    double? latitude,
    double? longitude,
    String? ownerId,
  }) = _Restaurant;

  factory Restaurant.fromJson(Map<String, dynamic> json) =>
      _$RestaurantFromJson(json);
}
```

```dart
// lib/features/users/domain/entities/app_user.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'app_user.freezed.dart';
part 'app_user.g.dart';

@freezed
class AppUser with _$AppUser {
  const factory AppUser({
    required String id,
    required String name,
    required String email,
    required String role,
    required bool isActive,
    required int totalOrders,
    required DateTime createdAt,
    String? phone,
    String? avatarUrl,
    double? totalSpent,
    String? address,
  }) = _AppUser;

  factory AppUser.fromJson(Map<String, dynamic> json) =>
      _$AppUserFromJson(json);
}
```

## ขั้นตอนที่ 3734: Providers

```dart
// lib/features/orders/presentation/providers/orders_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:food_admin_dashboard/features/orders/domain/entities/order.dart';

part 'orders_provider.g.dart';

@riverpod
Future<List<Order>> filteredOrders(
  FilteredOrdersRef ref, {
  String? status,
  String? search,
}) async {
  // Simulated API call with mock data
  await Future.delayed(const Duration(milliseconds: 300));
  
  final mockOrders = List.generate(50, (i) => Order(
    id: 'order-${i.toString().padLeft(4, '0')}',
    customerId: 'user-$i',
    customerName: 'Customer ${i + 1}',
    customerPhone: '08${i.toString().padLeft(8, '0')}',
    restaurantId: 'rest-${i % 10}',
    restaurantName: ['Somtam House', 'Ramen Ichiban', 'Pizza Roma', 'Sushi Bar'][i % 4],
    items: [
      OrderItem(menuItemId: 'menu-1', name: 'Item 1', price: 150, quantity: 2),
    ],
    subtotal: 300,
    deliveryFee: 40,
    discount: 0,
    totalAmount: 340,
    status: ['Pending', 'Processing', 'Delivered', 'Cancelled'][i % 4],
    deliveryAddress: 'Address $i, Bangkok',
    createdAt: DateTime.now().subtract(Duration(hours: i)),
    updatedAt: DateTime.now().subtract(Duration(hours: i ~/ 2)),
  ));

  var filtered = mockOrders;
  if (status != null) {
    filtered = filtered.where((o) => o.status == status).toList();
  }
  if (search != null && search.isNotEmpty) {
    final q = search.toLowerCase();
    filtered = filtered.where((o) =>
        o.customerName.toLowerCase().contains(q) ||
        o.id.toLowerCase().contains(q)).toList();
  }
  return filtered;
}
```

```dart
// lib/features/restaurants/presentation/providers/restaurants_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:food_admin_dashboard/features/restaurants/domain/entities/restaurant.dart';

part 'restaurants_provider.g.dart';

@riverpod
class RestaurantsList extends _$RestaurantsList {
  @override
  Future<List<Restaurant>> build() async {
    await Future.delayed(const Duration(milliseconds: 400));
    return List.generate(20, (i) => Restaurant(
      id: 'rest-$i',
      name: ['Somtam House', 'Ramen Ichiban', 'Pizza Roma', 'Sushi Bar 88',
             'Pad Thai King'][i % 5],
      category: ['Thai', 'Japanese', 'Italian', 'Fast Food'][i % 4],
      address: 'Address $i, Bangkok 10${i.toString().padLeft(3, '0')}',
      phone: '02-${(1000000 + i).toString()}',
      rating: 3.5 + (i % 15) * 0.1,
      totalOrders: 100 + i * 20,
      totalRevenue: 50000.0 + i * 5000,
      isActive: i % 5 != 0,
      isOpen: i % 3 != 0,
      createdAt: DateTime.now().subtract(Duration(days: i * 10)),
    ));
  }

  Future<void> toggleStatus(String id) async {
    final currentList = await future;
    state = AsyncData(currentList.map((r) {
      if (r.id == id) return r.copyWith(isActive: !r.isActive);
      return r;
    }).toList());
  }

  Future<void> deleteRestaurant(String id) async {
    final currentList = await future;
    state = AsyncData(currentList.where((r) => r.id != id).toList());
  }
}
```

```dart
// lib/features/users/presentation/providers/users_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';
import 'package:food_admin_dashboard/features/users/domain/entities/app_user.dart';

part 'users_provider.g.dart';

@riverpod
class UsersList extends _$UsersList {
  @override
  Future<List<AppUser>> build() async {
    await Future.delayed(const Duration(milliseconds: 350));
    return List.generate(30, (i) => AppUser(
      id: 'user-$i',
      name: 'User ${i + 1} Surname',
      email: 'user${i + 1}@example.com',
      role: ['Customer', 'Restaurant Owner', 'Admin', 'Driver'][i % 4],
      isActive: i % 7 != 0,
      totalOrders: i * 5,
      createdAt: DateTime.now().subtract(Duration(days: i * 5)),
      phone: '08${(10000000 + i).toString()}',
    ));
  }

  Future<void> toggleUserStatus(String id) async {
    final current = await future;
    state = AsyncData(current.map((u) {
      if (u.id == id) return u.copyWith(isActive: !u.isActive);
      return u;
    }).toList());
  }
}

@riverpod
StateController<ThemeMode> themeMode(ThemeModeRef ref) =>
    StateController(ThemeMode.system);
```

---

**← [Part 95 - Capstone Backend](part-95-capstone-backend.md)**
**ต่อไป: [Part 97 →](part-97-capstone-testing.md)**
