# Part 26: Clean Architecture
## ขั้นตอนที่ 921-960

---

## 🎯 เป้าหมายของ Part นี้

- Clean Architecture layers
- Domain, Data, Presentation layers
- Use Cases pattern
- Dependency Injection
- Feature-first folder structure

---

## ขั้นตอนที่ 921: Clean Architecture Overview

```
lib/
├── core/
│   ├── error/
│   │   ├── exceptions.dart
│   │   └── failures.dart
│   ├── network/
│   │   └── network_info.dart
│   └── usecases/
│       └── usecase.dart
│
└── features/
    └── user/
        ├── data/
        │   ├── datasources/
        │   │   ├── user_remote_data_source.dart
        │   │   └── user_local_data_source.dart
        │   ├── models/
        │   │   └── user_model.dart
        │   └── repositories/
        │       └── user_repository_impl.dart
        ├── domain/
        │   ├── entities/
        │   │   └── user.dart
        │   ├── repositories/
        │   │   └── user_repository.dart
        │   └── usecases/
        │       ├── get_user.dart
        │       └── update_user.dart
        └── presentation/
            ├── bloc/
            │   ├── user_bloc.dart
            │   ├── user_event.dart
            │   └── user_state.dart
            ├── pages/
            │   └── user_page.dart
            └── widgets/
                └── user_card.dart
```

---

## ขั้นตอนที่ 922: Domain Layer

```dart
// ─── core/error/failures.dart ───
abstract class Failure {
  final String message;
  const Failure(this.message);
}

class ServerFailure extends Failure {
  const ServerFailure([String message = 'เกิดข้อผิดพลาดกับ server']) : super(message);
}

class CacheFailure extends Failure {
  const CacheFailure([String message = 'ไม่พบข้อมูลใน cache']) : super(message);
}

class NetworkFailure extends Failure {
  const NetworkFailure([String message = 'ไม่มีการเชื่อมต่ออินเทอร์เน็ต']) : super(message);
}

class NotFoundFailure extends Failure {
  const NotFoundFailure([String message = 'ไม่พบข้อมูล']) : super(message);
}

// ─── core/usecases/usecase.dart ───
import 'package:dartz/dartz.dart';

abstract class UseCase<Type, Params> {
  Future<Either<Failure, Type>> call(Params params);
}

abstract class NoParamsUseCase<Type> {
  Future<Either<Failure, Type>> call();
}

class NoParams {}

// ─── features/user/domain/entities/user.dart ───
class User {
  final String id;
  final String name;
  final String email;
  final String? avatarUrl;
  final DateTime createdAt;

  const User({
    required this.id,
    required this.name,
    required this.email,
    this.avatarUrl,
    required this.createdAt,
  });

  @override
  bool operator ==(Object other) => other is User && other.id == id;

  @override
  int get hashCode => id.hashCode;
}

// ─── features/user/domain/repositories/user_repository.dart ───
import 'package:dartz/dartz.dart';

abstract class UserRepository {
  Future<Either<Failure, User>> getUserById(String id);
  Future<Either<Failure, List<User>>> getUsers({int page, int limit});
  Future<Either<Failure, User>> createUser(User user);
  Future<Either<Failure, User>> updateUser(User user);
  Future<Either<Failure, void>> deleteUser(String id);
}

// ─── features/user/domain/usecases/get_user.dart ───
import 'package:dartz/dartz.dart';

class GetUser implements UseCase<User, String> {
  final UserRepository _repository;

  const GetUser(this._repository);

  @override
  Future<Either<Failure, User>> call(String userId) {
    return _repository.getUserById(userId);
  }
}

// ─── features/user/domain/usecases/get_users.dart ───
class GetUsersParams {
  final int page;
  final int limit;

  const GetUsersParams({this.page = 1, this.limit = 20});
}

class GetUsers implements UseCase<List<User>, GetUsersParams> {
  final UserRepository _repository;

  const GetUsers(this._repository);

  @override
  Future<Either<Failure, List<User>>> call(GetUsersParams params) {
    return _repository.getUsers(page: params.page, limit: params.limit);
  }
}

// ─── features/user/domain/usecases/update_user.dart ───
class UpdateUser implements UseCase<User, User> {
  final UserRepository _repository;

  const UpdateUser(this._repository);

  @override
  Future<Either<Failure, User>> call(User user) {
    return _repository.updateUser(user);
  }
}
```

---

## ขั้นตอนที่ 923: Data Layer

```dart
// ─── features/user/data/models/user_model.dart ───
class UserModel extends User {
  const UserModel({
    required String id,
    required String name,
    required String email,
    String? avatarUrl,
    required DateTime createdAt,
  }) : super(
    id: id,
    name: name,
    email: email,
    avatarUrl: avatarUrl,
    createdAt: createdAt,
  );

  factory UserModel.fromJson(Map<String, dynamic> json) => UserModel(
    id: json['id'].toString(),
    name: json['name'],
    email: json['email'],
    avatarUrl: json['avatar_url'],
    createdAt: DateTime.parse(json['created_at']),
  );

  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'email': email,
    'avatar_url': avatarUrl,
    'created_at': createdAt.toIso8601String(),
  };

  factory UserModel.fromEntity(User user) => UserModel(
    id: user.id,
    name: user.name,
    email: user.email,
    avatarUrl: user.avatarUrl,
    createdAt: user.createdAt,
  );
}

// ─── features/user/data/datasources/user_remote_data_source.dart ───
import 'package:dio/dio.dart';

abstract class UserRemoteDataSource {
  Future<UserModel> getUserById(String id);
  Future<List<UserModel>> getUsers({int page, int limit});
  Future<UserModel> createUser(Map<String, dynamic> data);
  Future<UserModel> updateUser(String id, Map<String, dynamic> data);
  Future<void> deleteUser(String id);
}

class UserRemoteDataSourceImpl implements UserRemoteDataSource {
  final Dio _dio;

  const UserRemoteDataSourceImpl(this._dio);

  @override
  Future<UserModel> getUserById(String id) async {
    try {
      Response res = await _dio.get('/users/$id');
      return UserModel.fromJson(res.data);
    } on DioException catch (e) {
      if (e.response?.statusCode == 404) {
        throw const NotFoundFailure();
      }
      throw ServerFailure(e.message ?? 'Server error');
    }
  }

  @override
  Future<List<UserModel>> getUsers({int page = 1, int limit = 20}) async {
    Response res = await _dio.get('/users', queryParameters: {
      'page': page,
      'limit': limit,
    });
    return (res.data as List).map((j) => UserModel.fromJson(j)).toList();
  }

  @override
  Future<UserModel> createUser(Map<String, dynamic> data) async {
    Response res = await _dio.post('/users', data: data);
    return UserModel.fromJson(res.data);
  }

  @override
  Future<UserModel> updateUser(String id, Map<String, dynamic> data) async {
    Response res = await _dio.put('/users/$id', data: data);
    return UserModel.fromJson(res.data);
  }

  @override
  Future<void> deleteUser(String id) => _dio.delete('/users/$id');
}

// ─── features/user/data/datasources/user_local_data_source.dart ───
import 'package:shared_preferences/shared_preferences.dart';
import 'dart:convert';

abstract class UserLocalDataSource {
  Future<UserModel?> getCachedUser(String id);
  Future<void> cacheUser(UserModel user);
  Future<void> clearCache();
}

class UserLocalDataSourceImpl implements UserLocalDataSource {
  final SharedPreferences _prefs;
  static const String _prefix = 'user_';

  const UserLocalDataSourceImpl(this._prefs);

  @override
  Future<UserModel?> getCachedUser(String id) async {
    String? json = _prefs.getString('$_prefix$id');
    if (json == null) return null;
    return UserModel.fromJson(jsonDecode(json));
  }

  @override
  Future<void> cacheUser(UserModel user) async {
    await _prefs.setString(
      '$_prefix${user.id}',
      jsonEncode(user.toJson()),
    );
  }

  @override
  Future<void> clearCache() async {
    Set<String> keys = _prefs.getKeys().where((k) => k.startsWith(_prefix)).toSet();
    for (String key in keys) {
      await _prefs.remove(key);
    }
  }
}

// ─── features/user/data/repositories/user_repository_impl.dart ───
import 'package:dartz/dartz.dart';

class UserRepositoryImpl implements UserRepository {
  final UserRemoteDataSource _remote;
  final UserLocalDataSource _local;

  const UserRepositoryImpl(this._remote, this._local);

  @override
  Future<Either<Failure, User>> getUserById(String id) async {
    // Try cache first
    UserModel? cached = await _local.getCachedUser(id);
    if (cached != null) return Right(cached);

    try {
      UserModel user = await _remote.getUserById(id);
      await _local.cacheUser(user);
      return Right(user);
    } on Failure catch (f) {
      return Left(f);
    } catch (e) {
      return Left(ServerFailure(e.toString()));
    }
  }

  @override
  Future<Either<Failure, List<User>>> getUsers({
    int page = 1,
    int limit = 20,
  }) async {
    try {
      List<UserModel> users = await _remote.getUsers(page: page, limit: limit);
      return Right(users);
    } on Failure catch (f) {
      return Left(f);
    } catch (e) {
      return Left(ServerFailure(e.toString()));
    }
  }

  @override
  Future<Either<Failure, User>> createUser(User user) async {
    try {
      UserModel created = await _remote.createUser(UserModel.fromEntity(user).toJson());
      return Right(created);
    } on Failure catch (f) {
      return Left(f);
    } catch (e) {
      return Left(ServerFailure(e.toString()));
    }
  }

  @override
  Future<Either<Failure, User>> updateUser(User user) async {
    try {
      UserModel updated = await _remote.updateUser(
        user.id,
        UserModel.fromEntity(user).toJson(),
      );
      await _local.cacheUser(updated);
      return Right(updated);
    } on Failure catch (f) {
      return Left(f);
    } catch (e) {
      return Left(ServerFailure(e.toString()));
    }
  }

  @override
  Future<Either<Failure, void>> deleteUser(String id) async {
    try {
      await _remote.deleteUser(id);
      return const Right(null);
    } on Failure catch (f) {
      return Left(f);
    } catch (e) {
      return Left(ServerFailure(e.toString()));
    }
  }
}
```

---

## ขั้นตอนที่ 924: Presentation Layer กับ BLoC

```dart
// ─── features/user/presentation/bloc/user_event.dart ───
import 'package:equatable/equatable.dart';

abstract class UserEvent extends Equatable {
  @override
  List<Object?> get props => [];
}

class UsersLoadRequested extends UserEvent {
  final int page;
  UsersLoadRequested({this.page = 1});
  @override
  List<Object?> get props => [page];
}

class UserLoadRequested extends UserEvent {
  final String id;
  UserLoadRequested(this.id);
  @override
  List<Object?> get props => [id];
}

class UserUpdateRequested extends UserEvent {
  final User user;
  UserUpdateRequested(this.user);
  @override
  List<Object?> get props => [user];
}

// ─── features/user/presentation/bloc/user_state.dart ───
abstract class UserState extends Equatable {
  @override
  List<Object?> get props => [];
}

class UserInitial extends UserState {}
class UserLoading extends UserState {}

class UsersLoaded extends UserState {
  final List<User> users;
  const UsersLoaded(this.users);
  @override
  List<Object?> get props => [users];
}

class UserLoaded extends UserState {
  final User user;
  const UserLoaded(this.user);
  @override
  List<Object?> get props => [user];
}

class UserUpdated extends UserState {
  final User user;
  const UserUpdated(this.user);
  @override
  List<Object?> get props => [user];
}

class UserError extends UserState {
  final String message;
  const UserError(this.message);
  @override
  List<Object?> get props => [message];
}

// ─── features/user/presentation/bloc/user_bloc.dart ───
import 'package:dartz/dartz.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class UserBloc extends Bloc<UserEvent, UserState> {
  final GetUser _getUser;
  final GetUsers _getUsers;
  final UpdateUser _updateUser;

  UserBloc({
    required GetUser getUser,
    required GetUsers getUsers,
    required UpdateUser updateUser,
  })  : _getUser = getUser,
        _getUsers = getUsers,
        _updateUser = updateUser,
        super(UserInitial()) {
    on<UsersLoadRequested>(_onUsersLoadRequested);
    on<UserLoadRequested>(_onUserLoadRequested);
    on<UserUpdateRequested>(_onUserUpdateRequested);
  }

  Future<void> _onUsersLoadRequested(
    UsersLoadRequested event,
    Emitter<UserState> emit,
  ) async {
    emit(UserLoading());
    Either<Failure, List<User>> result = await _getUsers(
      GetUsersParams(page: event.page),
    );
    result.fold(
      (failure) => emit(UserError(failure.message)),
      (users) => emit(UsersLoaded(users)),
    );
  }

  Future<void> _onUserLoadRequested(
    UserLoadRequested event,
    Emitter<UserState> emit,
  ) async {
    emit(UserLoading());
    Either<Failure, User> result = await _getUser(event.id);
    result.fold(
      (failure) => emit(UserError(failure.message)),
      (user) => emit(UserLoaded(user)),
    );
  }

  Future<void> _onUserUpdateRequested(
    UserUpdateRequested event,
    Emitter<UserState> emit,
  ) async {
    emit(UserLoading());
    Either<Failure, User> result = await _updateUser(event.user);
    result.fold(
      (failure) => emit(UserError(failure.message)),
      (user) => emit(UserUpdated(user)),
    );
  }
}
```

---

## ขั้นตอนที่ 925: Dependency Injection กับ get_it

```dart
// pubspec.yaml: get_it: ^8.0.0

import 'package:get_it/get_it.dart';
import 'package:dio/dio.dart';
import 'package:shared_preferences/shared_preferences.dart';

final GetIt sl = GetIt.instance;

Future<void> setupDependencies() async {
  // ─── External ───
  SharedPreferences prefs = await SharedPreferences.getInstance();
  sl.registerLazySingleton<SharedPreferences>(() => prefs);

  Dio dio = Dio(BaseOptions(baseUrl: 'https://api.example.com'));
  sl.registerLazySingleton<Dio>(() => dio);

  // ─── Data Sources ───
  sl.registerLazySingleton<UserRemoteDataSource>(
    () => UserRemoteDataSourceImpl(sl()),
  );
  sl.registerLazySingleton<UserLocalDataSource>(
    () => UserLocalDataSourceImpl(sl()),
  );

  // ─── Repositories ───
  sl.registerLazySingleton<UserRepository>(
    () => UserRepositoryImpl(sl(), sl()),
  );

  // ─── Use Cases ───
  sl.registerLazySingleton(() => GetUser(sl()));
  sl.registerLazySingleton(() => GetUsers(sl()));
  sl.registerLazySingleton(() => UpdateUser(sl()));

  // ─── BLoC ───
  sl.registerFactory(() => UserBloc(
    getUser: sl(),
    getUsers: sl(),
    updateUser: sl(),
  ));
}

// ─── main.dart ───
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await setupDependencies();
  runApp(const MyApp());
}

// ─── Widget ───
class UserListPage extends StatelessWidget {
  const UserListPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => sl<UserBloc>()..add(UsersLoadRequested()),
      child: Scaffold(
        appBar: AppBar(title: const Text('Users')),
        body: BlocBuilder<UserBloc, UserState>(
          builder: (context, state) {
            if (state is UserLoading) {
              return const Center(child: CircularProgressIndicator());
            }
            if (state is UsersLoaded) {
              return ListView.builder(
                itemCount: state.users.length,
                itemBuilder: (_, i) => ListTile(
                  title: Text(state.users[i].name),
                  subtitle: Text(state.users[i].email),
                ),
              );
            }
            if (state is UserError) {
              return Center(child: Text(state.message));
            }
            return const SizedBox();
          },
        ),
      ),
    );
  }
}
```

---

**← [Part 25 - Platform Channels](part-25-platform-channels.md)**

**ต่อไป: [Part 27 - CI/CD and Deployment →](part-27-cicd.md)**
