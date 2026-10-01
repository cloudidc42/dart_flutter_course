# Part 20: BLoC Pattern
## ขั้นตอนที่ 681-720

---

## 🎯 เป้าหมายของ Part นี้

- BLoC (Business Logic Component) pattern
- flutter_bloc package
- Cubit (simplified BLoC)
- BLoC Events & States
- BlocBuilder, BlocListener, BlocConsumer
- Real-world BLoC application

---

## ขั้นตอนที่ 681: Setup BLoC

```yaml
# pubspec.yaml
dependencies:
  flutter_bloc: ^8.1.5
  equatable: ^2.0.5
```

---

## ขั้นตอนที่ 682: Cubit (Simple State Management)

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

// ─── Counter Cubit ───
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);

  void increment() => emit(state + 1);
  void decrement() => emit(state - 1);
  void reset() => emit(0);
}

// ─── Theme Cubit ───
class ThemeCubit extends Cubit<ThemeMode> {
  ThemeCubit() : super(ThemeMode.system);

  void toggle() {
    emit(state == ThemeMode.light ? ThemeMode.dark : ThemeMode.light);
  }

  void setTheme(ThemeMode mode) => emit(mode);
}

// ─── Widgets ───
class CounterPage extends StatelessWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => CounterCubit(),
      child: const _CounterView(),
    );
  }
}

class _CounterView extends StatelessWidget {
  const _CounterView();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Counter with Cubit')),
      body: Center(
        child: BlocBuilder<CounterCubit, int>(
          builder: (context, count) => Text(
            '$count',
            style: const TextStyle(fontSize: 60, fontWeight: FontWeight.bold),
          ),
        ),
      ),
      floatingActionButton: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          FloatingActionButton(
            heroTag: 'inc',
            onPressed: () => context.read<CounterCubit>().increment(),
            child: const Icon(Icons.add),
          ),
          const SizedBox(height: 8),
          FloatingActionButton(
            heroTag: 'dec',
            onPressed: () => context.read<CounterCubit>().decrement(),
            child: const Icon(Icons.remove),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 683: BLoC Events & States

```dart
import 'package:equatable/equatable.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

// ─── States ───
abstract class AuthState extends Equatable {
  @override
  List<Object?> get props => [];
}

class AuthInitial extends AuthState {}

class AuthLoading extends AuthState {}

class AuthAuthenticated extends AuthState {
  final Map<String, dynamic> user;
  AuthAuthenticated(this.user);

  @override
  List<Object?> get props => [user];
}

class AuthUnauthenticated extends AuthState {}

class AuthError extends AuthState {
  final String message;
  AuthError(this.message);

  @override
  List<Object?> get props => [message];
}

// ─── Events ───
abstract class AuthEvent extends Equatable {
  @override
  List<Object?> get props => [];
}

class AuthLoginRequested extends AuthEvent {
  final String email;
  final String password;
  AuthLoginRequested({required this.email, required this.password});

  @override
  List<Object?> get props => [email, password];
}

class AuthLogoutRequested extends AuthEvent {}

class AuthCheckRequested extends AuthEvent {}

// ─── BLoC ───
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  AuthBloc() : super(AuthInitial()) {
    on<AuthCheckRequested>(_onCheckRequested);
    on<AuthLoginRequested>(_onLoginRequested);
    on<AuthLogoutRequested>(_onLogoutRequested);
  }

  Future<void> _onCheckRequested(
    AuthCheckRequested event,
    Emitter<AuthState> emit,
  ) async {
    emit(AuthLoading());
    await Future.delayed(const Duration(seconds: 1));
    // Check stored token
    emit(AuthUnauthenticated());
  }

  Future<void> _onLoginRequested(
    AuthLoginRequested event,
    Emitter<AuthState> emit,
  ) async {
    emit(AuthLoading());
    try {
      await Future.delayed(const Duration(seconds: 1));

      if (event.email == 'test@test.com' && event.password == '123456') {
        emit(AuthAuthenticated({'email': event.email, 'name': 'Test User'}));
      } else {
        emit(AuthError('Email หรือ password ไม่ถูกต้อง'));
      }
    } catch (e) {
      emit(AuthError(e.toString()));
    }
  }

  Future<void> _onLogoutRequested(
    AuthLogoutRequested event,
    Emitter<AuthState> emit,
  ) async {
    emit(AuthLoading());
    await Future.delayed(const Duration(milliseconds: 500));
    emit(AuthUnauthenticated());
  }
}
```

---

## ขั้นตอนที่ 684: BlocBuilder, BlocListener, BlocConsumer

```dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class LoginPage extends StatefulWidget {
  const LoginPage({super.key});

  @override
  State<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> {
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  final _formKey = GlobalKey<FormState>();

  @override
  void dispose() {
    _emailController.dispose();
    _passwordController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('เข้าสู่ระบบ')),
      body: BlocConsumer<AuthBloc, AuthState>(
        // listener: รัน side effects เช่น navigation, snackbar
        listener: (context, state) {
          if (state is AuthAuthenticated) {
            Navigator.pushReplacementNamed(context, '/home');
          }
          if (state is AuthError) {
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(
                content: Text(state.message),
                backgroundColor: Colors.red,
              ),
            );
          }
        },
        // builder: สร้าง UI จาก state
        builder: (context, state) {
          return Padding(
            padding: const EdgeInsets.all(24),
            child: Form(
              key: _formKey,
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  TextFormField(
                    controller: _emailController,
                    decoration: const InputDecoration(
                      labelText: 'Email',
                      prefixIcon: Icon(Icons.email),
                      border: OutlineInputBorder(),
                    ),
                    validator: (v) => v?.isEmpty == true ? 'กรุณาใส่ email' : null,
                    enabled: state is! AuthLoading,
                  ),
                  const SizedBox(height: 16),
                  TextFormField(
                    controller: _passwordController,
                    decoration: const InputDecoration(
                      labelText: 'Password',
                      prefixIcon: Icon(Icons.lock),
                      border: OutlineInputBorder(),
                    ),
                    obscureText: true,
                    validator: (v) => v?.isEmpty == true ? 'กรุณาใส่ password' : null,
                    enabled: state is! AuthLoading,
                  ),
                  const SizedBox(height: 24),
                  if (state is AuthLoading)
                    const CircularProgressIndicator()
                  else
                    SizedBox(
                      width: double.infinity,
                      child: ElevatedButton(
                        onPressed: () {
                          if (_formKey.currentState!.validate()) {
                            context.read<AuthBloc>().add(
                              AuthLoginRequested(
                                email: _emailController.text,
                                password: _passwordController.text,
                              ),
                            );
                          }
                        },
                        style: ElevatedButton.styleFrom(
                          padding: const EdgeInsets.symmetric(vertical: 16),
                        ),
                        child: const Text('เข้าสู่ระบบ', style: TextStyle(fontSize: 18)),
                      ),
                    ),
                ],
              ),
            ),
          );
        },
      ),
    );
  }
}

// ─── BlocBuilder (build only) ───
class AuthStatusWidget extends StatelessWidget {
  const AuthStatusWidget({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<AuthBloc, AuthState>(
      // buildWhen: รีเบิลด์เฉพาะเมื่อ condition เป็น true
      buildWhen: (previous, current) => previous.runtimeType != current.runtimeType,
      builder: (context, state) {
        if (state is AuthAuthenticated) {
          return Chip(
            avatar: const Icon(Icons.person, size: 16, color: Colors.white),
            label: Text(state.user['email'].toString()),
            backgroundColor: Colors.green,
          );
        }
        return const Chip(label: Text('ไม่ได้เข้าสู่ระบบ'));
      },
    );
  }
}
```

---

## ขั้นตอนที่ 685: Shopping App กับ BLoC

```dart
import 'package:equatable/equatable.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter/material.dart';

// ─── Domain ───
class Product extends Equatable {
  final String id;
  final String name;
  final double price;
  final String imageUrl;
  final String category;

  const Product({
    required this.id,
    required this.name,
    required this.price,
    required this.imageUrl,
    required this.category,
  });

  @override
  List<Object?> get props => [id];
}

// ─── Product States ───
abstract class ProductState extends Equatable {
  @override
  List<Object?> get props => [];
}

class ProductInitial extends ProductState {}
class ProductLoading extends ProductState {}
class ProductLoaded extends ProductState {
  final List<Product> products;
  final String selectedCategory;
  final String searchQuery;

  List<Product> get filtered {
    List<Product> result = products;

    if (selectedCategory.isNotEmpty) {
      result = result.where((p) => p.category == selectedCategory).toList();
    }

    if (searchQuery.isNotEmpty) {
      result = result.where((p) =>
        p.name.toLowerCase().contains(searchQuery.toLowerCase())
      ).toList();
    }

    return result;
  }

  const ProductLoaded({
    required this.products,
    this.selectedCategory = '',
    this.searchQuery = '',
  });

  ProductLoaded copyWith({
    List<Product>? products,
    String? selectedCategory,
    String? searchQuery,
  }) {
    return ProductLoaded(
      products: products ?? this.products,
      selectedCategory: selectedCategory ?? this.selectedCategory,
      searchQuery: searchQuery ?? this.searchQuery,
    );
  }

  @override
  List<Object?> get props => [products, selectedCategory, searchQuery];
}
class ProductError extends ProductState {
  final String message;
  ProductError(this.message);
  @override
  List<Object?> get props => [message];
}

// ─── Product Events ───
abstract class ProductEvent extends Equatable {
  @override
  List<Object?> get props => [];
}

class ProductLoadRequested extends ProductEvent {}
class ProductCategorySelected extends ProductEvent {
  final String category;
  ProductCategorySelected(this.category);
  @override
  List<Object?> get props => [category];
}
class ProductSearched extends ProductEvent {
  final String query;
  ProductSearched(this.query);
  @override
  List<Object?> get props => [query];
}

// ─── Product BLoC ───
class ProductBloc extends Bloc<ProductEvent, ProductState> {
  ProductBloc() : super(ProductInitial()) {
    on<ProductLoadRequested>(_onLoadRequested);
    on<ProductCategorySelected>(_onCategorySelected);
    on<ProductSearched>(_onSearched);
  }

  Future<void> _onLoadRequested(
    ProductLoadRequested event,
    Emitter<ProductState> emit,
  ) async {
    emit(ProductLoading());
    try {
      await Future.delayed(const Duration(seconds: 1));
      emit(ProductLoaded(products: _fakeProducts));
    } catch (e) {
      emit(ProductError(e.toString()));
    }
  }

  void _onCategorySelected(
    ProductCategorySelected event,
    Emitter<ProductState> emit,
  ) {
    if (state is ProductLoaded) {
      emit((state as ProductLoaded).copyWith(selectedCategory: event.category));
    }
  }

  void _onSearched(
    ProductSearched event,
    Emitter<ProductState> emit,
  ) {
    if (state is ProductLoaded) {
      emit((state as ProductLoaded).copyWith(searchQuery: event.query));
    }
  }

  static final List<Product> _fakeProducts = [
    const Product(id: '1', name: 'iPhone 15', price: 39900, imageUrl: '', category: 'Phone'),
    const Product(id: '2', name: 'Samsung S24', price: 32900, imageUrl: '', category: 'Phone'),
    const Product(id: '3', name: 'MacBook Pro', price: 79900, imageUrl: '', category: 'Laptop'),
    const Product(id: '4', name: 'iPad Pro', price: 29900, imageUrl: '', category: 'Tablet'),
  ];
}

// ─── Product Screen ───
class ProductScreen extends StatelessWidget {
  const ProductScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => ProductBloc()..add(ProductLoadRequested()),
      child: const _ProductView(),
    );
  }
}

class _ProductView extends StatelessWidget {
  const _ProductView();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('สินค้า'),
        bottom: PreferredSize(
          preferredSize: const Size.fromHeight(48),
          child: Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
            child: TextField(
              onChanged: (q) => context.read<ProductBloc>().add(ProductSearched(q)),
              decoration: const InputDecoration(
                hintText: 'ค้นหาสินค้า...',
                prefixIcon: Icon(Icons.search),
                fillColor: Colors.white,
                filled: true,
                border: OutlineInputBorder(),
                isDense: true,
                contentPadding: EdgeInsets.symmetric(vertical: 8, horizontal: 12),
              ),
            ),
          ),
        ),
      ),
      body: BlocBuilder<ProductBloc, ProductState>(
        builder: (context, state) {
          if (state is ProductLoading) {
            return const Center(child: CircularProgressIndicator());
          }

          if (state is ProductError) {
            return Center(
              child: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text(state.message),
                  ElevatedButton(
                    onPressed: () => context.read<ProductBloc>().add(ProductLoadRequested()),
                    child: const Text('ลองใหม่'),
                  ),
                ],
              ),
            );
          }

          if (state is ProductLoaded) {
            List<String> categories = ['', ...state.products.map((p) => p.category).toSet()];

            return Column(
              children: [
                // Categories
                SizedBox(
                  height: 48,
                  child: ListView.builder(
                    scrollDirection: Axis.horizontal,
                    padding: const EdgeInsets.symmetric(horizontal: 12),
                    itemCount: categories.length,
                    itemBuilder: (_, i) {
                      bool isSelected = categories[i] == state.selectedCategory;
                      return Padding(
                        padding: const EdgeInsets.symmetric(horizontal: 4, vertical: 8),
                        child: FilterChip(
                          label: Text(categories[i].isEmpty ? 'ทั้งหมด' : categories[i]),
                          selected: isSelected,
                          onSelected: (_) => context.read<ProductBloc>().add(
                            ProductCategorySelected(categories[i]),
                          ),
                        ),
                      );
                    },
                  ),
                ),
                // Products
                Expanded(
                  child: GridView.builder(
                    padding: const EdgeInsets.all(12),
                    gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                      crossAxisCount: 2,
                      childAspectRatio: 0.8,
                      crossAxisSpacing: 12,
                      mainAxisSpacing: 12,
                    ),
                    itemCount: state.filtered.length,
                    itemBuilder: (_, i) {
                      Product p = state.filtered[i];
                      return Card(
                        child: Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            Expanded(
                              child: Container(
                                color: Colors.blue.shade50,
                                child: const Center(child: Icon(Icons.phone_iphone, size: 60)),
                              ),
                            ),
                            Padding(
                              padding: const EdgeInsets.all(8),
                              child: Column(
                                crossAxisAlignment: CrossAxisAlignment.start,
                                children: [
                                  Text(p.name, style: const TextStyle(fontWeight: FontWeight.bold)),
                                  Text('฿${p.price.toStringAsFixed(0)}',
                                      style: const TextStyle(color: Colors.green)),
                                ],
                              ),
                            ),
                          ],
                        ),
                      );
                    },
                  ),
                ),
              ],
            );
          }

          return const SizedBox();
        },
      ),
    );
  }
}
```

---

**← [Part 19 - Riverpod](part-19-riverpod.md)**

**ต่อไป: [Part 21 - Custom Widgets and Themes →](part-21-custom-widgets-themes.md)**
