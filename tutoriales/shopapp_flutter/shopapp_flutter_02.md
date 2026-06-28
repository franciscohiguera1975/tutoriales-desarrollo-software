# Flutter Shop App — Módulo 2
## Dio + Interceptores JWT + SecureStorage + Repositorios base

> **Objetivo:** Capa de datos completa: cliente HTTP con renovación automática de tokens, SecureStorage para persistir la sesión y repositorios listos para ser consumidos por los providers.
> **Checkpoint final:** la pantalla de verificación llama a `GET /api/categories/` y muestra las categorías reales del backend.

---

## 2.1 SecureStorage — `lib/data/local/secure_storage.dart`

```dart
// lib/data/local/secure_storage.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecureStorage {
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
    iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock),
  );

  static const _keyAccess   = 'flutter_shop_app:access';
  static const _keyRefresh  = 'flutter_shop_app:refresh';
  static const _keyUserId   = 'flutter_shop_app:user_id';
  static const _keyUsername = 'flutter_shop_app:username';
  static const _keyEmail    = 'flutter_shop_app:email';
  static const _keyIsStaff  = 'flutter_shop_app:is_staff';

  // ── Tokens ────────────────────────────────────────────────
  Future<String?> getAccess()  => _storage.read(key: _keyAccess);
  Future<String?> getRefresh() => _storage.read(key: _keyRefresh);

  Future<void> saveTokens(String access, String refresh) async {
    await _storage.write(key: _keyAccess,  value: access);
    await _storage.write(key: _keyRefresh, value: refresh);
  }

  Future<void> saveAccessToken(String access) =>
      _storage.write(key: _keyAccess, value: access);

  // ── Usuario ───────────────────────────────────────────────
  Future<void> saveUser({
    required int    id,
    required String username,
    required String email,
    required bool   isStaff,
  }) async {
    await _storage.write(key: _keyUserId,   value: id.toString());
    await _storage.write(key: _keyUsername, value: username);
    await _storage.write(key: _keyEmail,    value: email);
    await _storage.write(key: _keyIsStaff,  value: isStaff.toString());
  }

  Future<Map<String, String>?> getUser() async {
    final id       = await _storage.read(key: _keyUserId);
    final username = await _storage.read(key: _keyUsername);
    final email    = await _storage.read(key: _keyEmail);
    final isStaff  = await _storage.read(key: _keyIsStaff);
    if (id == null || username == null) return null;
    return {
      'id':       id,
      'username': username,
      'email':    email ?? '',
      'is_staff': isStaff ?? 'false',
    };
  }

  Future<bool> isLoggedIn() async {
    final access = await getAccess();
    return access != null && access.isNotEmpty;
  }

  Future<void> clearSession() => _storage.deleteAll();
}

// Provider global de SecureStorage
final secureStorageProvider = Provider<SecureStorage>((_) => SecureStorage());
```

---

## 2.2 Manejo de errores API — `lib/core/error/api_exception.dart`

```dart
// lib/core/error/api_exception.dart

import 'package:dio/dio.dart';

class ApiException implements Exception {
  final String message;
  final int?   statusCode;
  final Map<String, dynamic>? fieldErrors;

  const ApiException(this.message, {this.statusCode, this.fieldErrors});

  factory ApiException.fromDioError(DioException error) {
    final response = error.response;
    final code     = response?.statusCode;

    if (response?.data == null) {
      return ApiException(
        error.message ?? 'Error de conexión',
        statusCode: code,
      );
    }

    final data = response!.data;

    // Mapa de errores de campo Django
    if (data is Map<String, dynamic>) {
      if (data.containsKey('detail')) {
        return ApiException(data['detail'].toString(), statusCode: code);
      }
      if (data.containsKey('error')) {
        return ApiException(data['error'].toString(), statusCode: code);
      }
      if (data.containsKey('non_field_errors')) {
        final errors = data['non_field_errors'];
        final msg    = errors is List ? errors.first.toString() : errors.toString();
        return ApiException(msg, statusCode: code);
      }
      // Errores por campo
      final fieldErrors = <String, dynamic>{};
      String? firstMessage;
      data.forEach((key, value) {
        final msg = value is List ? value.first.toString() : value.toString();
        fieldErrors[key] = msg;
        firstMessage ??= '$key: $msg';
      });
      return ApiException(
        firstMessage ?? 'Error de validación',
        statusCode:  code,
        fieldErrors: fieldErrors,
      );
    }

    return ApiException(data.toString(), statusCode: code);
  }

  String? fieldError(String field) => fieldErrors?[field]?.toString();

  @override
  String toString() => message;
}
```

---

## 2.3 Cliente Dio — `lib/data/remote/api/dio_client.dart`

```dart
// lib/data/remote/api/dio_client.dart

import 'package:dio/dio.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../core/config/app_config.dart';
import '../local/secure_storage.dart';

// ── Interceptor de autenticación ──────────────────────────────
class _AuthInterceptor extends Interceptor {
  final SecureStorage _storage;
  final Dio           _dio;

  _AuthInterceptor(this._storage, this._dio);

  @override
  void onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final token = await _storage.getAccess();
    if (token != null && token.isNotEmpty) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    final response = err.response;

    // Solo actuar en 401 y no es el endpoint de refresh
    if (response?.statusCode != 401) {
      handler.next(err);
      return;
    }
    if (err.requestOptions.path.contains('/auth/token/refresh/')) {
      await _storage.clearSession();
      handler.next(err);
      return;
    }
    // Evitar bucle infinito
    if (err.requestOptions.extra['_retry'] == true) {
      await _storage.clearSession();
      handler.next(err);
      return;
    }

    // Intentar renovar el token
    final refresh = await _storage.getRefresh();
    if (refresh == null || refresh.isEmpty) {
      await _storage.clearSession();
      handler.next(err);
      return;
    }

    try {
      final refreshResponse = await _dio.post(
        '/auth/token/refresh/',
        data:    {'refresh': refresh},
        options: Options(extra: {'_retry': true}),
      );

      final newAccess  = refreshResponse.data['access']  as String;
      final newRefresh = refreshResponse.data['refresh']  as String?;

      await _storage.saveAccessToken(newAccess);
      if (newRefresh != null) {
        await _storage.saveTokens(newAccess, newRefresh);
      }

      // Reintentar la petición original
      final retryOptions = err.requestOptions;
      retryOptions.headers['Authorization'] = 'Bearer $newAccess';
      retryOptions.extra['_retry']          = true;

      final retryResponse = await _dio.fetch(retryOptions);
      handler.resolve(retryResponse);

    } on DioException {
      await _storage.clearSession();
      handler.next(err);
    }
  }
}

// ── Fábrica del cliente Dio ────────────────────────────────────
Dio createDioClient(SecureStorage storage) {
  final dio = Dio(
    BaseOptions(
      baseUrl:        '${AppConfig.baseUrl}/',
      connectTimeout: const Duration(seconds: 30),
      receiveTimeout: const Duration(seconds: 30),
      headers:        {'Content-Type': 'application/json'},
    ),
  );

  dio.interceptors.addAll([
    _AuthInterceptor(storage, dio),
    LogInterceptor(
      requestBody:  true,
      responseBody: true,
      logPrint:     (o) => debugPrint(o.toString()),
    ),
  ]);

  return dio;
}

// ── Provider global de Dio ────────────────────────────────────
final dioProvider = Provider<Dio>((ref) {
  final storage = ref.watch(secureStorageProvider);
  return createDioClient(storage);
});
```

---

## 2.4 Datasource de categorías — `data/remote/api/category_remote_datasource.dart`

```dart
// lib/data/remote/api/category_remote_datasource.dart

import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../core/error/api_exception.dart';
import 'dio_client.dart';
import '../../../domain/model/category.dart';

abstract class CategoryRemoteDatasource {
  Future<List<Category>> getCategories();
  Future<Category>       getCategory(int id);
  Future<Category>       createCategory(Map<String, dynamic> payload);
  Future<Category>       updateCategory(int id, Map<String, dynamic> payload);
  Future<void>           deleteCategory(int id);
  Future<Map<String, dynamic>> getStats();
}

class CategoryRemoteDatasourceImpl implements CategoryRemoteDatasource {
  final Dio _dio;
  CategoryRemoteDatasourceImpl(this._dio);

  @override
  Future<List<Category>> getCategories() async {
    try {
      final res = await _dio.get('/categories/');
      final data = res.data as Map<String, dynamic>;
      return (data['results'] as List)
          .map((e) => Category.fromJson(e as Map<String, dynamic>))
          .toList();
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Category> getCategory(int id) async {
    try {
      final res = await _dio.get('/categories/$id/');
      return Category.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Category> createCategory(Map<String, dynamic> payload) async {
    try {
      final res = await _dio.post('/categories/', data: payload);
      return Category.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Category> updateCategory(int id, Map<String, dynamic> payload) async {
    try {
      final res = await _dio.patch('/categories/$id/', data: payload);
      return Category.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<void> deleteCategory(int id) async {
    try {
      await _dio.delete('/categories/$id/');
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Map<String, dynamic>> getStats() async {
    try {
      final res = await _dio.get('/categories/stats/');
      return res.data as Map<String, dynamic>;
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }
}

final categoryDatasourceProvider = Provider<CategoryRemoteDatasource>((ref) {
  return CategoryRemoteDatasourceImpl(ref.watch(dioProvider));
});
```

---

## 2.5 Datasource de productos — `data/remote/api/product_remote_datasource.dart`

```dart
// lib/data/remote/api/product_remote_datasource.dart

import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../core/error/api_exception.dart';
import 'dio_client.dart';
import '../../../domain/model/product.dart';

abstract class ProductRemoteDatasource {
  Future<PaginatedProducts> getProducts({
    String?  search,
    int?     category,
    double?  priceMin,
    double?  priceMax,
    int?     stockMin,
    bool?    isActive,
    String?  ordering,
    int      page,
    int      pageSize,
  });
  Future<Product>             getProduct(int id);
  Future<Product>             createProduct(Map<String, dynamic> payload);
  Future<Product>             updateProduct(int id, Map<String, dynamic> payload);
  Future<void>                deleteProduct(int id);
  Future<Map<String, dynamic>> restock(int id, int quantity);
  Future<Map<String, dynamic>> getStats();
}

class ProductRemoteDatasourceImpl implements ProductRemoteDatasource {
  final Dio _dio;
  ProductRemoteDatasourceImpl(this._dio);

  @override
  Future<PaginatedProducts> getProducts({
    String?  search,
    int?     category,
    double?  priceMin,
    double?  priceMax,
    int?     stockMin,
    bool?    isActive,
    String?  ordering,
    int      page     = 1,
    int      pageSize = 12,
  }) async {
    try {
      final params = <String, dynamic>{
        'page':      page,
        'page_size': pageSize,
        if (search   != null) 'search':    search,
        if (category != null) 'category':  category,
        if (priceMin != null) 'price_min': priceMin,
        if (priceMax != null) 'price_max': priceMax,
        if (stockMin != null) 'stock_min': stockMin,
        if (isActive != null) 'is_active': isActive,
        if (ordering != null) 'ordering':  ordering,
      };
      final res = await _dio.get('/products/', queryParameters: params);
      return PaginatedProducts.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Product> getProduct(int id) async {
    try {
      final res = await _dio.get('/products/$id/');
      return Product.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Product> createProduct(Map<String, dynamic> payload) async {
    try {
      final res = await _dio.post('/products/', data: payload);
      return Product.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Product> updateProduct(int id, Map<String, dynamic> payload) async {
    try {
      final res = await _dio.patch('/products/$id/', data: payload);
      return Product.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<void> deleteProduct(int id) async {
    try {
      await _dio.delete('/products/$id/');
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Map<String, dynamic>> restock(int id, int quantity) async {
    try {
      final res = await _dio.post('/products/$id/restock/', data: {'quantity': quantity});
      return res.data as Map<String, dynamic>;
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Map<String, dynamic>> getStats() async {
    try {
      final res = await _dio.get('/products/stats/');
      return res.data as Map<String, dynamic>;
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }
}

final productDatasourceProvider = Provider<ProductRemoteDatasource>((ref) {
  return ProductRemoteDatasourceImpl(ref.watch(dioProvider));
});
```

---

## 2.6 Datasource de pedidos — `data/remote/api/order_remote_datasource.dart`

```dart
// lib/data/remote/api/order_remote_datasource.dart

import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../core/error/api_exception.dart';
import 'dio_client.dart';
import '../../../domain/model/order.dart';

abstract class OrderRemoteDatasource {
  Future<PaginatedOrders>      getOrders({int? page, String? status});
  Future<Order>                getOrder(int id);
  Future<Order>                createOrder();
  Future<Order>                addItem(int orderId, int productId, int quantity);
  Future<Order>                confirmOrder(int orderId);
  Future<Order>                updateStatus(int orderId, String status);
  Future<Map<String, dynamic>> getStats();
}

class OrderRemoteDatasourceImpl implements OrderRemoteDatasource {
  final Dio _dio;
  OrderRemoteDatasourceImpl(this._dio);

  @override
  Future<PaginatedOrders> getOrders({int? page, String? status}) async {
    try {
      final params = <String, dynamic>{
        if (page   != null) 'page':   page,
        if (status != null) 'status': status,
      };
      final res = await _dio.get('/orders/', queryParameters: params);
      return PaginatedOrders.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Order> getOrder(int id) async {
    try {
      final res = await _dio.get('/orders/$id/');
      return Order.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Order> createOrder() async {
    try {
      final res = await _dio.post('/orders/', data: {});
      return Order.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Order> addItem(int orderId, int productId, int quantity) async {
    try {
      final res = await _dio.post(
        '/orders/$orderId/add-item/',
        data: {'product_id': productId, 'quantity': quantity},
      );
      return Order.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Order> confirmOrder(int orderId) async {
    try {
      final res = await _dio.post('/orders/$orderId/confirm/');
      return Order.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Order> updateStatus(int orderId, String status) async {
    try {
      final res = await _dio.post(
        '/orders/$orderId/update-status/',
        data: {'status': status},
      );
      return Order.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Map<String, dynamic>> getStats() async {
    try {
      final res = await _dio.get('/orders/stats/');
      return res.data as Map<String, dynamic>;
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }
}

final orderDatasourceProvider = Provider<OrderRemoteDatasource>((ref) {
  return OrderRemoteDatasourceImpl(ref.watch(dioProvider));
});
```

---

## 2.7 Datasource de usuarios — `data/remote/api/user_remote_datasource.dart`

```dart
// lib/data/remote/api/user_remote_datasource.dart

import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../core/error/api_exception.dart';
import 'dio_client.dart';
import '../../../domain/model/user.dart';

class PaginatedUsers {
  final int        count;
  final String?    next;
  final List<User> results;
  const PaginatedUsers({required this.count, required this.next, required this.results});

  factory PaginatedUsers.fromJson(Map<String, dynamic> j) => PaginatedUsers(
    count:   j['count']   as int,
    next:    j['next']    as String?,
    results: (j['results'] as List)
        .map((e) => User.fromJson(e as Map<String, dynamic>))
        .toList(),
  );
}

abstract class UserRemoteDatasource {
  Future<PaginatedUsers>       getUsers({String? search, bool? isStaff, bool? isActive});
  Future<User>                 createUser(Map<String, dynamic> payload);
  Future<User>                 updateUser(int id, Map<String, dynamic> payload);
  Future<void>                 deleteUser(int id);
  Future<bool>                 toggleActive(int id);
  Future<Map<String, dynamic>> getStats();
}

class UserRemoteDatasourceImpl implements UserRemoteDatasource {
  final Dio _dio;
  UserRemoteDatasourceImpl(this._dio);

  @override
  Future<PaginatedUsers> getUsers({String? search, bool? isStaff, bool? isActive}) async {
    try {
      final params = <String, dynamic>{
        if (search   != null) 'search':    search,
        if (isStaff  != null) 'is_staff':  isStaff,
        if (isActive != null) 'is_active': isActive,
      };
      final res = await _dio.get('/users/', queryParameters: params);
      return PaginatedUsers.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<User> createUser(Map<String, dynamic> payload) async {
    try {
      final res = await _dio.post('/users/', data: payload);
      return User.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<User> updateUser(int id, Map<String, dynamic> payload) async {
    try {
      final res = await _dio.patch('/users/$id/', data: payload);
      return User.fromJson(res.data as Map<String, dynamic>);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<void> deleteUser(int id) async {
    try {
      await _dio.delete('/users/$id/');
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<bool> toggleActive(int id) async {
    try {
      final res = await _dio.post('/users/$id/toggle-active/');
      return res.data['is_active'] as bool;
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<Map<String, dynamic>> getStats() async {
    try {
      final res = await _dio.get('/users/stats/');
      return res.data as Map<String, dynamic>;
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }
}

final userDatasourceProvider = Provider<UserRemoteDatasource>((ref) {
  return UserRemoteDatasourceImpl(ref.watch(dioProvider));
});
```

---

## 2.8 Repositorio de categorías

### 2.8.1 Contrato — `lib/domain/repository/category_repository.dart`

> La interfaz vive en `domain` porque es una regla de negocio: la capa de datos la implementa, pero no la define.

```dart
// lib/domain/repository/category_repository.dart

import '../model/category.dart';

abstract class CategoryRepository {
  Future<List<Category>> getCategories();
  Future<Category>       getCategory(int id);
  Future<Category>       createCategory(Map<String, dynamic> payload);
  Future<Category>       updateCategory(int id, Map<String, dynamic> payload);
  Future<void>           deleteCategory(int id);
  Future<Map<String, dynamic>> getStats();
}
```

### 2.8.2 Implementación — `lib/data/repository/category_repository_impl.dart`

```dart
// lib/data/repository/category_repository_impl.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../domain/model/category.dart';
import '../../domain/repository/category_repository.dart';
import '../remote/api/category_remote_datasource.dart';

class CategoryRepositoryImpl implements CategoryRepository {
  final CategoryRemoteDatasource _datasource;
  CategoryRepositoryImpl(this._datasource);

  @override
  Future<List<Category>> getCategories() => _datasource.getCategories();

  @override
  Future<Category> getCategory(int id) => _datasource.getCategory(id);

  @override
  Future<Category> createCategory(Map<String, dynamic> payload) =>
      _datasource.createCategory(payload);

  @override
  Future<Category> updateCategory(int id, Map<String, dynamic> payload) =>
      _datasource.updateCategory(id, payload);

  @override
  Future<void> deleteCategory(int id) => _datasource.deleteCategory(id);

  @override
  Future<Map<String, dynamic>> getStats() => _datasource.getStats();
}

final categoryRepositoryProvider = Provider<CategoryRepository>((ref) {
  return CategoryRepositoryImpl(ref.watch(categoryDatasourceProvider));
});
```

---

## 2.9 Actualizar `main.dart` — verificar conexión real

```dart
// lib/main.dart — actualizar VerificationScreen con llamada real al backend

import 'package:flutter/material.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'core/config/app_config.dart';
import 'theme/app_theme.dart';
import 'theme/app_colors.dart';
import 'data/repository/category_repository_impl.dart';
import 'domain/model/category.dart';

Future<void> main() async {
  await dotenv.load(fileName: '.env');
  runApp(const ProviderScope(child: FlutterShopApp()));
}

class FlutterShopApp extends StatelessWidget {
  const FlutterShopApp({super.key});

  @override
  Widget build(BuildContext context) => MaterialApp(
    title:            AppConfig.appName,
    debugShowCheckedModeBanner: false,
    theme:            AppTheme.dark,
    home:             const VerificationScreen(),
  );
}

// Provider de verificación
final _categoriesVerifyProvider = FutureProvider<List<Category>>((ref) {
  return ref.watch(categoryRepositoryProvider).getCategories();
});

class VerificationScreen extends ConsumerWidget {
  const VerificationScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final categoriesAsync = ref.watch(_categoriesVerifyProvider);
    final tt              = Theme.of(context).textTheme;

    return Scaffold(
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(24),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              const SizedBox(height: 32),

              Center(
                child: Column(
                  children: [
                    Text(AppConfig.appName, style: tt.displayMedium?.copyWith(color: AppColors.accent)),
                    const SizedBox(height: 6),
                    Text('Módulo 2 · Conexión real con Django', style: tt.bodyMedium),
                  ],
                ),
              ),
              const SizedBox(height: 32),

              // Estado de la conexión
              Container(
                width: double.infinity,
                padding: const EdgeInsets.all(16),
                decoration: BoxDecoration(
                  color:        AppColors.surface,
                  borderRadius: BorderRadius.circular(16),
                  border:       Border.all(color: AppColors.border),
                ),
                child: categoriesAsync.when(
                  loading: () => const Row(
                    children: [
                      SizedBox(
                        width: 18, height: 18,
                        child: CircularProgressIndicator(
                          strokeWidth: 2,
                          color:       AppColors.accent,
                        ),
                      ),
                      SizedBox(width: 12),
                      Text('Conectando con Django...', style: TextStyle(color: AppColors.textSecondary)),
                    ],
                  ),
                  error: (err, _) => Row(
                    children: [
                      const Text('❌ ', style: TextStyle(fontSize: 20)),
                      Expanded(
                        child: Text(
                          err.toString(),
                          style: const TextStyle(color: AppColors.error, fontSize: 12),
                        ),
                      ),
                    ],
                  ),
                  data: (cats) => Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(
                        '✅ ${cats.length} categorías del backend',
                        style: const TextStyle(
                          color:      AppColors.success,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      const SizedBox(height: 12),
                      ...cats.take(3).map((c) => Padding(
                        padding: const EdgeInsets.symmetric(vertical: 4),
                        child: Row(
                          mainAxisAlignment: MainAxisAlignment.spaceBetween,
                          children: [
                            Text(c.name, style: const TextStyle(color: AppColors.textPrimary, fontSize: 13)),
                            Text(
                              '${c.totalProducts} prod.',
                              style: const TextStyle(color: AppColors.accent, fontSize: 12, fontWeight: FontWeight.bold),
                            ),
                          ],
                        ),
                      )),
                    ],
                  ),
                ),
              ),
              const SizedBox(height: 20),

              // Capa de datos
              Text(
                'CAPA DE DATOS',
                style: tt.labelSmall?.copyWith(letterSpacing: 1.2, color: AppColors.textSecondary),
              ),
              const SizedBox(height: 12),
              ...[
                'data/local/secure_storage.dart',
                'data/remote/api/dio_client.dart',
                'core/error/api_exception.dart',
                'data/remote/api/category_remote_datasource.dart',
                'data/remote/api/product_remote_datasource.dart',
                'data/remote/api/order_remote_datasource.dart',
                'data/remote/api/user_remote_datasource.dart',
              ].map((f) => Padding(
                padding: const EdgeInsets.symmetric(vertical: 3),
                child: Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    Expanded(
                      child: Text(
                        'lib/$f',
                        style: const TextStyle(color: AppColors.accent, fontSize: 11, fontFamily: 'monospace'),
                        overflow: TextOverflow.ellipsis,
                      ),
                    ),
                    const Text('✓', style: TextStyle(color: AppColors.success, fontWeight: FontWeight.bold)),
                  ],
                ),
              )),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## 2.10 Actualizar test base — `test/widget_test.dart`

> Al reemplazar el `main.dart` inicial, el test generado por Flutter puede seguir apuntando a `MyApp`. Reemplázalo por este bloque para que `flutter analyze` y `flutter test` pasen sin cambios extra.

```dart
// test/widget_test.dart

import 'package:flutter_test/flutter_test.dart';

import 'package:flutter_shop_app/main.dart';

void main() {
  test('FlutterShopApp can be constructed', () {
    expect(const FlutterShopApp(), isA<FlutterShopApp>());
  });
}
```

---

## ✅ Checkpoint Módulo 2

### Django corriendo

```bash
uv run python manage.py runserver 0.0.0.0:8000
```

### Flutter corriendo

```bash
flutter run
```

### Verificación local

```bash
flutter analyze
flutter test
```

| Verificación | Resultado esperado |
|---|---|
| `✅ N categorías del backend` | Los datos reales de Django aparecen |
| Django apagado → error visible | `❌ Error de conexión` en rojo |
| Logs de Dio en consola | `GET http://10.0.2.2:8000/api/categories/` |
| Lista de 7 archivos con ✓ | Toda la capa de datos creada |
| `flutter analyze` | `No issues found!` |
| `flutter test` | `All tests passed!` |

### Poblar datos de prueba

```bash
uv run python manage.py shell -c "
from store.models import Category, Product
cat1 = Category.objects.get_or_create(name='Electrónica', slug='electronica', defaults={'description':'Dispositivos', 'is_active':True})[0]
cat2 = Category.objects.get_or_create(name='Ropa', slug='ropa', defaults={'description':'Moda', 'is_active':True})[0]
Product.objects.get_or_create(name='Laptop', defaults={'price':999,'stock':5,'category':cat1})
Product.objects.get_or_create(name='Camiseta', defaults={'price':25,'stock':20,'category':cat2})
print('Datos OK')
"
```

---

## Resumen

| Elemento | Estado |
|---|---|
| `SecureStorage` — wrapper de FlutterSecureStorage | ✅ |
| `ApiException` — normaliza errores Django | ✅ |
| `DioClient` con interceptor de renovación JWT | ✅ |
| Renovación automática de token en 401 | ✅ |
| `CategoryRemoteDatasource` | ✅ |
| `ProductRemoteDatasource` con filtros | ✅ |
| `OrderRemoteDatasource` con flujo de checkout | ✅ |
| `UserRemoteDatasource` con stats | ✅ |
| `CategoryRepository` con Provider de Riverpod | ✅ |
| `FutureProvider` de verificación en `main.dart` | ✅ |
| `widget_test.dart` actualizado para `FlutterShopApp` | ✅ |
| Conexión real verificada con datos del backend | ✅ |

**Siguiente módulo →** M3: Auth — Login, Registro, AuthNotifier y persistencia de sesión
