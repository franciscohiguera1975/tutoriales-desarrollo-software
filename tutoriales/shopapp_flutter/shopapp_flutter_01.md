# Flutter Shop App — Módulo 1
## Setup + Estructura + Design System Material 3 + Modelos de dominio

> **Objetivo:** Proyecto Flutter configurado, estructura de carpetas con Clean Architecture, design system con Material 3 y modelos de dominio definidos por feature.
> **Checkpoint final:** `flutter run` muestra la pantalla de verificación con el tema oscuro y el accent dorado aplicado.

---

## 1.1 Crear el proyecto

```bash
flutter create flutter_shop_app --org com.flutter_shop_app --platforms android,ios
cd flutter_shop_app
```

Verificar versiones:

```bash
flutter --version   # >= 3.19
dart --version      # >= 3.3
```

---

## 1.2 Dependencias — `pubspec.yaml`

```yaml
# pubspec.yaml
name: flutter_shop_app
description: Flutter Shop App — Cliente móvil para ShopAPI
publish_to: none
version: 1.0.0+1

environment:
  sdk: ">=3.5.0 <4.0.0"

dependencies:
  flutter:
    sdk: flutter

  # ── Estado ────────────────────────────────────────────────
  flutter_riverpod: ^2.5.1
  riverpod_annotation: ^2.3.5

  # ── Navegación ────────────────────────────────────────────
  go_router: ^14.2.7

  # ── HTTP ─────────────────────────────────────────────────
  dio: ^5.7.0

  # ── Almacenamiento seguro ─────────────────────────────────
  flutter_secure_storage: ^9.2.2

  # ── Imágenes ─────────────────────────────────────────────
  cached_network_image: ^3.4.1

  # ── Modelos / serialización ───────────────────────────────
  freezed_annotation: ^2.4.4
  json_annotation: ^4.9.0

  # ── Configuración / entorno ───────────────────────────────
  flutter_dotenv: ^5.2.1

  # ── Utilidades ────────────────────────────────────────────
  intl: ^0.19.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0

  # ── Generadores de código ─────────────────────────────────
  build_runner: ^2.4.13
  freezed: ^2.5.7
  json_serializable: ^6.8.0
  riverpod_generator: ^2.4.3
  custom_lint: ^0.7.0
  riverpod_lint: ^2.3.13

flutter:
  uses-material-design: true
  assets:
    - .env
```

```bash
flutter pub get
```

---

## 1.3 Configuración base — `lib/core/config/app_config.dart`

Se usa `flutter_dotenv` para cargar variables desde un archivo `.env` en la raíz del proyecto. Cambiar de ambiente es solo cambiar el archivo, sin tocar código.

### Archivos `.env` por ambiente

Crear uno por ambiente en la raíz del proyecto — **ninguno se sube al repositorio**:

```env
# .env  ← producción (o el ambiente activo)
API_BASE_URL=https://higuera-shopapi.uaeftt-ute.site/api
```

```env
# .env.dev  ← desarrollo local
API_BASE_URL=http://10.0.2.2:8000/api
```

Agregar a `.gitignore`:

```
# Ambientes — nunca versionar
.env
.env.*
!.env.example
```

Crear plantilla para el equipo — **esta sí se versiona**:

```env
# .env.example
API_BASE_URL=https://higuera-shopapi.uaeftt-ute.site/api
```

### `lib/core/config/app_config.dart`

```dart
// lib/core/config/app_config.dart

import 'package:flutter_dotenv/flutter_dotenv.dart';

class AppConfig {
  static String get baseUrl =>
      dotenv.env['API_BASE_URL'] ?? 'http://10.0.2.2:8000/api';

  static const String appName = 'Flutter Shop App';
  static const double taxRate = 0.15; // IVA Ecuador 15 %
}
```

### Ejecutar la app

```bash
flutter run          # carga .env por defecto
```

Para cambiar de ambiente, renombra o copia el archivo correspondiente a `.env`:

```bash
cp .env.dev .env && flutter run   # desarrollo local
```

> ⚠️ **Importante:** el archivo `.env` se empaqueta como asset en el binario, por lo que puede ser extraído con herramientas de análisis. Para secrets críticos (pagos, datos sensibles) lo correcto es que el backend sea el intermediario y la app nunca reciba esas claves directamente.

---

## 1.4 Permisos Android — `android/app/src/main/AndroidManifest.xml`

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>

    <application
        android:label="Flutter Shop App"
        android:usesCleartextTraffic="true">  <!-- solo desarrollo HTTP -->
        ...
    </application>
</manifest>
```

---

## 1.5 Estructura de carpetas objetivo

```
lib/
├── main.dart
│
├── data/
│   ├── remote/
│   │   ├── api/
│   │   │   └── dio_client.dart         ← instancia Dio + interceptores
│   │   ├── dto/
│   │   │   ├── auth_dto.dart
│   │   │   ├── category_dto.dart
│   │   │   ├── product_dto.dart
│   │   │   └── order_dto.dart
│   │   └── interceptor/
│   │       └── auth_interceptor.dart
│   ├── local/
│   │   └── secure_storage.dart         ← wrapper de FlutterSecureStorage
│   └── repository/
│       ├── auth_repository_impl.dart
│       ├── catalog_repository_impl.dart
│       ├── order_repository_impl.dart
│       └── admin_repository_impl.dart
│
├── domain/
│   ├── model/
│   │   ├── auth_models.dart
│   │   ├── category.dart
│   │   ├── product.dart
│   │   ├── order.dart
│   │   └── user.dart
│   └── repository/
│       ├── auth_repository.dart
│       ├── catalog_repository.dart
│       ├── order_repository.dart
│       └── admin_repository.dart
│
├── presentation/
│   ├── navigation/
│   │   └── app_router.dart             ← GoRouter con guards
│   ├── screens/
│   │   ├── auth/
│   │   │   ├── login_screen.dart
│   │   │   └── register_screen.dart
│   │   ├── catalog/
│   │   │   ├── catalog_screen.dart
│   │   │   └── product_detail_screen.dart
│   │   ├── cart/
│   │   │   └── cart_screen.dart
│   │   ├── orders/
│   │   │   ├── orders_screen.dart
│   │   │   └── order_detail_screen.dart
│   │   ├── profile/
│   │   │   └── profile_screen.dart
│   │   └── admin/
│   │       ├── admin_products_screen.dart
│   │       ├── admin_categories_screen.dart
│   │       ├── admin_orders_screen.dart
│   │       └── admin_users_screen.dart
│   ├── providers/
│   │   ├── auth_provider.dart
│   │   ├── catalog_provider.dart
│   │   ├── cart_provider.dart
│   │   ├── order_provider.dart
│   │   └── admin_provider.dart
│   └── widgets/
│       ├── product_card.dart
│       ├── category_chip.dart
│       ├── order_status_badge.dart
│       └── loading_overlay.dart
│
├── theme/
│   ├── app_colors.dart
│   ├── app_text_styles.dart
│   └── app_theme.dart
│
└── core/
    ├── config/
    │   └── app_config.dart
    ├── error/
    │   └── api_exception.dart      ← excepción compartida entre datasources
    └── utils/
        ├── formatters.dart
        └── validators.dart
```

### Crear directorios base

```bash
mkdir -p lib/data/{remote/{api,dto,interceptor},local,repository}
mkdir -p lib/domain/{model,repository}
mkdir -p lib/presentation/{navigation,screens/{auth,catalog,cart,orders,profile,admin},providers,widgets}
mkdir -p lib/theme
mkdir -p lib/core/{config,error,utils}
```

O en un solo comando:

```bash
mkdir -p lib/{data/{remote/{api,dto,interceptor},local,repository},domain/{model,repository},presentation/{navigation,screens/{auth,catalog,cart,orders,profile,admin},providers,widgets},theme,core/{config,error,utils}}
```

### Tabla de imports por capa — referencia rápida

Cada `..` sube un nivel de directorio desde donde vive el archivo.

| Archivo origen | Destino | Import relativo |
|---|---|---|
| `data/remote/api/*.dart` | `core/error/` | `'../../../core/error/api_exception.dart'` |
| `data/remote/api/*.dart` | `core/config/` | `'../../../core/config/app_config.dart'` |
| `data/remote/api/*.dart` | `data/local/` | `'../../local/secure_storage.dart'` |
| `data/remote/api/*.dart` | `domain/model/` | `'../../../domain/model/xxx.dart'` |
| `data/repository/*.dart` | `core/error/` | `'../../core/error/api_exception.dart'` |
| `data/repository/*.dart` | `domain/model/` | `'../../domain/model/xxx.dart'` |
| `data/repository/*.dart` | `domain/repository/` | `'../../domain/repository/xxx.dart'` |
| `data/repository/*.dart` | `data/remote/api/` | `'../remote/api/xxx.dart'` |
| `presentation/providers/*.dart` | `core/error/` | `'../../core/error/api_exception.dart'` |
| `presentation/providers/*.dart` | `domain/model/` | `'../../domain/model/xxx.dart'` |
| `presentation/providers/*.dart` | `domain/repository/` | `'../../domain/repository/xxx.dart'` |
| `presentation/screens/auth/*.dart` | `core/error/` | `'../../../core/error/api_exception.dart'` |
| `presentation/screens/auth/*.dart` | `domain/model/` | `'../../../domain/model/xxx.dart'` |

---

## 1.6 Design System — colores

```dart
// lib/theme/app_colors.dart

import 'package:flutter/material.dart';

class AppColors {
  // ── Fondos ────────────────────────────────────────────────
  static const Color background  = Color(0xFF0A0A0F);
  static const Color surface     = Color(0xFF111118);
  static const Color surface2    = Color(0xFF1A1A24);
  static const Color border      = Color(0xFF2A2A38);
  static const Color borderLight = Color(0xFF1E1E2A);

  // ── Texto ─────────────────────────────────────────────────
  static const Color textPrimary   = Color(0xFFF0F0F8);
  static const Color textSecondary = Color(0xFF8888AA);
  static const Color textFaint     = Color(0xFF44445A);

  // ── Accent dorado ─────────────────────────────────────────
  static const Color accent      = Color(0xFFD4A843);
  static const Color accentLight = Color(0xFFF0C96E);
  static const Color accentDark  = Color(0xFFA07820);
  static const Color onAccent    = Color(0xFF0A0A0F);

  // ── Semánticos ────────────────────────────────────────────
  static const Color success = Color(0xFF22C55E);
  static const Color warning = Color(0xFFF59E0B);
  static const Color error   = Color(0xFFEF4444);
  static const Color info    = Color(0xFF3B82F6);

  // ── Estado de pedido ──────────────────────────────────────
  static const Color statusPending   = Color(0xFFF59E0B);
  static const Color statusConfirmed = Color(0xFF3B82F6);
  static const Color statusShipped   = Color(0xFF8B5CF6);
  static const Color statusDelivered = Color(0xFF22C55E);
  static const Color statusCancelled = Color(0xFFEF4444);

  AppColors._();
}
```

---

## 1.7 Design System — tema Material 3

```dart
// lib/theme/app_theme.dart

import 'package:flutter/material.dart';
import 'app_colors.dart';

class AppTheme {
  static ThemeData get dark => ThemeData(
    useMaterial3:  true,
    brightness:    Brightness.dark,
    colorScheme:   _colorScheme,
    scaffoldBackgroundColor: AppColors.background,
    appBarTheme:   _appBarTheme,
    cardTheme:     _cardTheme,
    inputDecorationTheme: _inputDecorationTheme,
    elevatedButtonTheme:  _elevatedButtonTheme,
    outlinedButtonTheme:  _outlinedButtonTheme,
    textButtonTheme:      _textButtonTheme,
    chipTheme:     _chipTheme,
    dividerTheme:  _dividerTheme,
    bottomNavigationBarTheme: _bottomNavTheme,
    navigationDrawerTheme:    _drawerTheme,
    snackBarTheme: _snackBarTheme,
    textTheme:     _textTheme,
    fontFamily:    'Roboto',
  );

  static const ColorScheme _colorScheme = ColorScheme.dark(
    primary:          AppColors.accent,
    onPrimary:        AppColors.onAccent,
    secondary:        AppColors.accentLight,
    onSecondary:      AppColors.onAccent,
    surface:          AppColors.surface,
    onSurface:        AppColors.textPrimary,
    surfaceContainer: AppColors.surface2,
    error:            AppColors.error,
    onError:          Colors.white,
    outline:          AppColors.border,
    outlineVariant:   AppColors.borderLight,
  );

  static const AppBarTheme _appBarTheme = AppBarTheme(
    backgroundColor: AppColors.surface,
    foregroundColor: AppColors.textPrimary,
    elevation:       0,
    scrolledUnderElevation: 0,
    centerTitle:     false,
    titleTextStyle:  TextStyle(
      color:      AppColors.textPrimary,
      fontSize:   18,
      fontWeight: FontWeight.bold,
    ),
  );

  static const CardThemeData _cardTheme = CardThemeData(
    color:        AppColors.surface,
    elevation:    0,
    shape:        RoundedRectangleBorder(
      borderRadius: BorderRadius.all(Radius.circular(16)),
    ),
    margin: EdgeInsets.zero,
  );

  static const InputDecorationTheme _inputDecorationTheme = InputDecorationTheme(
    filled:           true,
    fillColor:        AppColors.surface2,
    border:           OutlineInputBorder(
      borderRadius: BorderRadius.all(Radius.circular(12)),
      borderSide:   BorderSide(color: AppColors.border),
    ),
    enabledBorder:    OutlineInputBorder(
      borderRadius: BorderRadius.all(Radius.circular(12)),
      borderSide:   BorderSide(color: AppColors.border),
    ),
    focusedBorder:    OutlineInputBorder(
      borderRadius: BorderRadius.all(Radius.circular(12)),
      borderSide:   BorderSide(color: AppColors.accent, width: 2),
    ),
    errorBorder:      OutlineInputBorder(
      borderRadius: BorderRadius.all(Radius.circular(12)),
      borderSide:   BorderSide(color: AppColors.error),
    ),
    focusedErrorBorder: OutlineInputBorder(
      borderRadius: BorderRadius.all(Radius.circular(12)),
      borderSide:   BorderSide(color: AppColors.error, width: 2),
    ),
    labelStyle:       TextStyle(color: AppColors.textSecondary),
    hintStyle:        TextStyle(color: AppColors.textFaint),
    prefixIconColor:  AppColors.textSecondary,
    suffixIconColor:  AppColors.textSecondary,
  );

  static final ElevatedButtonThemeData _elevatedButtonTheme = ElevatedButtonThemeData(
    style: ElevatedButton.styleFrom(
      backgroundColor: AppColors.accent,
      foregroundColor: AppColors.onAccent,
      minimumSize:     const Size(double.infinity, 52),
      shape:           const RoundedRectangleBorder(
        borderRadius: BorderRadius.all(Radius.circular(12)),
      ),
      textStyle: const TextStyle(fontWeight: FontWeight.bold, fontSize: 15),
      elevation:  0,
    ),
  );

  static final OutlinedButtonThemeData _outlinedButtonTheme = OutlinedButtonThemeData(
    style: OutlinedButton.styleFrom(
      foregroundColor: AppColors.textPrimary,
      minimumSize:     const Size(double.infinity, 52),
      shape:           const RoundedRectangleBorder(
        borderRadius: BorderRadius.all(Radius.circular(12)),
      ),
      side:      const BorderSide(color: AppColors.border),
      textStyle: const TextStyle(fontWeight: FontWeight.w600, fontSize: 15),
    ),
  );

  static final TextButtonThemeData _textButtonTheme = TextButtonThemeData(
    style: TextButton.styleFrom(
      foregroundColor: AppColors.accent,
      textStyle: const TextStyle(fontWeight: FontWeight.w600),
    ),
  );

  static const ChipThemeData _chipTheme = ChipThemeData(
    backgroundColor:       AppColors.surface2,
    selectedColor:         AppColors.accent,
    labelStyle:            TextStyle(color: AppColors.textSecondary, fontSize: 12),
    secondaryLabelStyle:   TextStyle(color: AppColors.onAccent, fontSize: 12),
    side:                  BorderSide(color: AppColors.border),
    shape:                 StadiumBorder(),
    padding:               EdgeInsets.symmetric(horizontal: 8, vertical: 0),
  );

  static const DividerThemeData _dividerTheme = DividerThemeData(
    color:     AppColors.border,
    thickness: 0.5,
    space:     0,
  );

  static const BottomNavigationBarThemeData _bottomNavTheme =
      BottomNavigationBarThemeData(
    backgroundColor:          AppColors.surface,
    selectedItemColor:        AppColors.accent,
    unselectedItemColor:      AppColors.textSecondary,
    selectedLabelStyle:       TextStyle(fontSize: 11, fontWeight: FontWeight.bold),
    unselectedLabelStyle:     TextStyle(fontSize: 11),
    elevation:                0,
  );

  static const NavigationDrawerThemeData _drawerTheme = NavigationDrawerThemeData(
    backgroundColor:     AppColors.surface,
    indicatorColor:      Color(0x1FD4A843),
    surfaceTintColor:    Colors.transparent,
  );

  static const SnackBarThemeData _snackBarTheme = SnackBarThemeData(
    backgroundColor:   AppColors.surface2,
    contentTextStyle:  TextStyle(color: AppColors.textPrimary),
    behavior:          SnackBarBehavior.floating,
    shape:             RoundedRectangleBorder(
      borderRadius: BorderRadius.all(Radius.circular(12)),
    ),
  );

  static const TextTheme _textTheme = TextTheme(
    displayLarge:  TextStyle(color: AppColors.textPrimary,   fontWeight: FontWeight.bold),
    displayMedium: TextStyle(color: AppColors.textPrimary,   fontWeight: FontWeight.bold),
    headlineLarge: TextStyle(color: AppColors.textPrimary,   fontWeight: FontWeight.bold),
    headlineMedium:TextStyle(color: AppColors.textPrimary,   fontWeight: FontWeight.bold),
    titleLarge:    TextStyle(color: AppColors.textPrimary,   fontWeight: FontWeight.bold),
    titleMedium:   TextStyle(color: AppColors.textPrimary,   fontWeight: FontWeight.w600),
    titleSmall:    TextStyle(color: AppColors.textPrimary,   fontWeight: FontWeight.w600),
    bodyLarge:     TextStyle(color: AppColors.textPrimary),
    bodyMedium:    TextStyle(color: AppColors.textSecondary),
    bodySmall:     TextStyle(color: AppColors.textSecondary, fontSize: 12),
    labelLarge:    TextStyle(color: AppColors.textPrimary,   fontWeight: FontWeight.bold),
    labelSmall:    TextStyle(color: AppColors.textSecondary, fontSize: 11),
  );

  AppTheme._();
}
```

---

## 1.8 Modelos de dominio — `domain/model/`

### `domain/model/auth_models.dart`

```dart
// lib/domain/model/auth_models.dart

class AuthTokens {
  final String access;
  final String refresh;
  const AuthTokens({required this.access, required this.refresh});
}

class LoggedUser {
  final int    id;
  final String username;
  final String email;
  final bool   isStaff;

  const LoggedUser({
    required this.id,
    required this.username,
    required this.email,
    required this.isStaff,
  });

  factory LoggedUser.fromMap(Map<String, dynamic> map) => LoggedUser(
    id:       map['user_id'] as int,
    username: map['username'] as String,
    email:    map['email']    as String,
    isStaff:  map['is_staff'] as bool,
  );
}
```

### `domain/model/category.dart`

```dart
// lib/domain/model/category.dart

class Category {
  final int    id;
  final String name;
  final String slug;
  final String description;
  final bool   isActive;
  final int    totalProducts;
  final String createdAt;

  const Category({
    required this.id,
    required this.name,
    required this.slug,
    required this.description,
    required this.isActive,
    required this.totalProducts,
    required this.createdAt,
  });

  factory Category.fromJson(Map<String, dynamic> j) => Category(
    id:            j['id']             as int,
    name:          j['name']           as String,
    slug:          j['slug']           as String,
    description:   j['description']    as String,
    isActive:      j['is_active']      as bool,
    totalProducts: j['total_products'] as int,
    createdAt:     j['created_at']     as String,
  );

  Map<String, dynamic> toJson() => {
    'name':        name,
    'slug':        slug,
    'description': description,
    'is_active':   isActive,
  };

  Category copyWith({bool? isActive, String? name, String? slug, String? description}) =>
    Category(
      id:            id,
      name:          name          ?? this.name,
      slug:          slug          ?? this.slug,
      description:   description   ?? this.description,
      isActive:      isActive      ?? this.isActive,
      totalProducts: totalProducts,
      createdAt:     createdAt,
    );
}
```

### `domain/model/product.dart`

```dart
// lib/domain/model/product.dart

class ProductCategory {
  final int    id;
  final String name;
  const ProductCategory({required this.id, required this.name});

  factory ProductCategory.fromJson(Map<String, dynamic> j) =>
      ProductCategory(id: j['id'] as int, name: j['name'] as String);
}

class Product {
  final int              id;
  final String           name;
  final String           description;
  final double           price;
  final double           priceWithTax;
  final int              stock;
  final bool             inStock;
  final bool             isActive;
  final String?          imageUrl;
  final ProductCategory? category;
  final String           createdAt;
  final String           updatedAt;

  const Product({
    required this.id,
    required this.name,
    required this.description,
    required this.price,
    required this.priceWithTax,
    required this.stock,
    required this.inStock,
    required this.isActive,
    this.imageUrl,
    this.category,
    required this.createdAt,
    required this.updatedAt,
  });

  factory Product.fromJson(Map<String, dynamic> j) => Product(
    id:           j['id']                                  as int,
    name:         j['name']                                as String,
    description:  j['description']                         as String,
    price:        double.parse(j['price'].toString()),
    priceWithTax: (j['price_with_tax'] as num).toDouble(),
    stock:        j['stock']                               as int,
    inStock:      j['in_stock']                            as bool,
    isActive:     j['is_active']                           as bool,
    imageUrl:     j['image_url']                           as String?,
    category:     j['category'] != null
                  ? ProductCategory.fromJson(j['category'] as Map<String, dynamic>)
                  : null,
    createdAt:    j['created_at']                          as String,
    updatedAt:    j['updated_at']                          as String,
  );

  Product copyWith({bool? isActive, int? stock}) => Product(
    id:           id,
    name:         name,
    description:  description,
    price:        price,
    priceWithTax: priceWithTax,
    stock:        stock       ?? this.stock,
    inStock:      (stock ?? this.stock) > 0,
    isActive:     isActive    ?? this.isActive,
    imageUrl:     imageUrl,
    category:     category,
    createdAt:    createdAt,
    updatedAt:    updatedAt,
  );
}

class PaginatedProducts {
  final int            count;
  final String?        next;
  final List<Product>  results;

  const PaginatedProducts({
    required this.count,
    required this.next,
    required this.results,
  });

  factory PaginatedProducts.fromJson(Map<String, dynamic> j) => PaginatedProducts(
    count:   j['count']   as int,
    next:    j['next']    as String?,
    results: (j['results'] as List)
        .map((e) => Product.fromJson(e as Map<String, dynamic>))
        .toList(),
  );
}
```

### `domain/model/order.dart`

```dart
// lib/domain/model/order.dart

import 'package:flutter/material.dart';

enum OrderStatus {
  pending  ('pending',   'Pendiente'),
  confirmed('confirmed', 'Confirmado'),
  shipped  ('shipped',   'Enviado'),
  delivered('delivered', 'Entregado'),
  cancelled('cancelled', 'Cancelado');

  const OrderStatus(this.value, this.label);
  final String value;
  final String label;

  static OrderStatus fromValue(String v) =>
      OrderStatus.values.firstWhere((s) => s.value == v, orElse: () => OrderStatus.pending);
}

class OrderItem {
  final int    id;
  final int    productId;
  final String productName;
  final int    quantity;
  final double unitPrice;
  final double subtotal;

  const OrderItem({
    required this.id,
    required this.productId,
    required this.productName,
    required this.quantity,
    required this.unitPrice,
    required this.subtotal,
  });

  factory OrderItem.fromJson(Map<String, dynamic> j) => OrderItem(
    id:          j['id']                                       as int,
    productId:   (j['product'] as Map<String, dynamic>)['id'] as int,
    productName: (j['product'] as Map<String, dynamic>)['name'] as String,
    quantity:    j['quantity']                                 as int,
    unitPrice:   double.parse(j['unit_price'].toString()),
    subtotal:    (j['subtotal'] as num).toDouble(),
  );
}

class Order {
  final int         id;
  final String      username;
  final OrderStatus status;
  final double      total;
  final int         numItems;
  final List<OrderItem> items;
  final String      createdAt;
  final String      updatedAt;

  const Order({
    required this.id,
    required this.username,
    required this.status,
    required this.total,
    required this.numItems,
    required this.items,
    required this.createdAt,
    required this.updatedAt,
  });

  factory Order.fromJson(Map<String, dynamic> j) => Order(
    id:        j['id']                                 as int,
    username:  j['username']                           as String,
    status:    OrderStatus.fromValue(j['status']       as String),
    total:     double.parse(j['total'].toString()),
    numItems:  j['num_items']                          as int,
    items:     (j['items'] as List)
               .map((e) => OrderItem.fromJson(e as Map<String, dynamic>))
               .toList(),
    createdAt: j['created_at']                         as String,
    updatedAt: j['updated_at']                         as String,
  );

  Order copyWith({OrderStatus? status}) => Order(
    id:        id,
    username:  username,
    status:    status ?? this.status,
    total:     total,
    numItems:  numItems,
    items:     items,
    createdAt: createdAt,
    updatedAt: updatedAt,
  );
}

class PaginatedOrders {
  final int         count;
  final String?     next;
  final List<Order> results;

  const PaginatedOrders({
    required this.count,
    required this.next,
    required this.results,
  });

  factory PaginatedOrders.fromJson(Map<String, dynamic> j) => PaginatedOrders(
    count:   j['count']   as int,
    next:    j['next']    as String?,
    results: (j['results'] as List)
        .map((e) => Order.fromJson(e as Map<String, dynamic>))
        .toList(),
  );
}
```

### `domain/model/user.dart`

```dart
// lib/domain/model/user.dart

class User {
  final int    id;
  final String username;
  final String email;
  final String firstName;
  final String lastName;
  final bool   isStaff;
  final bool   isActive;
  final String dateJoined;
  final int    numOrders;

  const User({
    required this.id,
    required this.username,
    required this.email,
    required this.firstName,
    required this.lastName,
    required this.isStaff,
    required this.isActive,
    required this.dateJoined,
    required this.numOrders,
  });

  factory User.fromJson(Map<String, dynamic> j) => User(
    id:         j['id']          as int,
    username:   j['username']    as String,
    email:      j['email']       as String,
    firstName:  j['first_name']  as String,
    lastName:   j['last_name']   as String,
    isStaff:    j['is_staff']    as bool,
    isActive:   j['is_active']   as bool,
    dateJoined: j['date_joined'] as String,
    numOrders:  j['num_orders']  as int,
  );

  Map<String, dynamic> toJson() => {
    'username':   username,
    'email':      email,
    'first_name': firstName,
    'last_name':  lastName,
    'is_staff':   isStaff,
    'is_active':  isActive,
  };

  User copyWith({bool? isStaff, bool? isActive}) => User(
    id:         id,
    username:   username,
    email:      email,
    firstName:  firstName,
    lastName:   lastName,
    isStaff:    isStaff  ?? this.isStaff,
    isActive:   isActive ?? this.isActive,
    dateJoined: dateJoined,
    numOrders:  numOrders,
  );
}
```

---

## 1.9 Utilidades base

```dart
// lib/core/utils/formatters.dart

import 'package:flutter/material.dart';
import 'package:intl/intl.dart';
import '../../domain/models/order.dart';

String formatPrice(double value, {String currency = '\$'}) =>
    '$currency${value.toStringAsFixed(2)}';

String formatPriceStr(String value, {String currency = '\$'}) {
  final num = double.tryParse(value) ?? 0.0;
  return formatPrice(num, currency: currency);
}

String formatDate(String iso) {
  try {
    final dt = DateTime.parse(iso).toLocal();
    return DateFormat('dd MMM yyyy', 'es').format(dt);
  } catch (_) {
    return iso.substring(0, 10);
  }
}

String formatDateTime(String iso) {
  try {
    final dt = DateTime.parse(iso).toLocal();
    return DateFormat('dd MMM yyyy · HH:mm', 'es').format(dt);
  } catch (_) {
    return iso.substring(0, 16);
  }
}

String truncate(String text, int max) =>
    text.length <= max ? text : '${text.substring(0, max).trimRight()}…';

Color orderStatusColor(OrderStatus status) {
  switch (status) {
    case OrderStatus.pending:   return const Color(0xFFF59E0B);
    case OrderStatus.confirmed: return const Color(0xFF3B82F6);
    case OrderStatus.shipped:   return const Color(0xFF8B5CF6);
    case OrderStatus.delivered: return const Color(0xFF22C55E);
    case OrderStatus.cancelled: return const Color(0xFFEF4444);
  }
}
```

```dart
// lib/core/utils/validators.dart

String? validateUsername(String? value) {
  if (value == null || value.trim().length < 3) {
    return 'Mínimo 3 caracteres';
  }
  return null;
}

String? validateEmail(String? value) {
  if (value == null || !value.contains('@')) {
    return 'Email inválido';
  }
  return null;
}

String? validatePassword(String? value) {
  if (value == null || value.length < 8) {
    return 'Mínimo 8 caracteres';
  }
  return null;
}

String? validateRequired(String? value, [String field = 'Campo']) {
  if (value == null || value.trim().isEmpty) return '$field es obligatorio';
  return null;
}

String? validatePositiveNumber(String? value, String field) {
  final num = double.tryParse(value ?? '');
  if (num == null || num <= 0) return '$field inválido';
  return null;
}

String? validateNonNegativeInt(String? value, String field) {
  final num = int.tryParse(value ?? '');
  if (num == null || num < 0) return '$field inválido';
  return null;
}
```

---

## 1.10 Entry point — `lib/main.dart`

```dart
// lib/main.dart

import 'package:flutter/material.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'core/config/app_config.dart';
import 'theme/app_theme.dart';
import 'theme/app_colors.dart';

Future<void> main() async {
  await dotenv.load(fileName: '.env');
  runApp(
    const ProviderScope(
      child: FlutterShopApp(),
    ),
  );
}

class FlutterShopApp extends StatelessWidget {
  const FlutterShopApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title:        AppConfig.appName,
      debugShowCheckedModeBanner: false,
      theme:        AppTheme.dark,
      home:         const VerificationScreen(),
    );
  }
}

// ── Pantalla de verificación ──────────────────────────────────

class VerificationScreen extends StatelessWidget {
  const VerificationScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final tt = Theme.of(context).textTheme;

    return Scaffold(
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(24),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              const SizedBox(height: 40),

              // Logo
              Center(
                child: Column(
                  children: [
                    Text(
                      AppConfig.appName,
                      style: tt.displayMedium?.copyWith(color: AppColors.accent),
                    ),
                    const SizedBox(height: 8),
                    Text(
                      'Módulo 1 · Flutter + Material 3',
                      style: tt.bodyMedium,
                    ),
                  ],
                ),
              ),
              const SizedBox(height: 40),

              // Card de entorno
              Container(
                width:  double.infinity,
                padding: const EdgeInsets.all(20),
                decoration: BoxDecoration(
                  color:        AppColors.surface,
                  borderRadius: BorderRadius.circular(16),
                  border:       Border.all(color: AppColors.border),
                ),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      'ESTADO DEL ENTORNO',
                      style: tt.labelSmall?.copyWith(
                        letterSpacing: 1.2,
                        color: AppColors.textSecondary,
                      ),
                    ),
                    const SizedBox(height: 16),
                    ...[
                      ('Flutter',     '3.x'),
                      ('Dart',        '3.x'),
                      ('Riverpod',    '2.x'),
                      ('GoRouter',    '14.x'),
                      ('Dio',         '5.x'),
                      ('API URL',     AppConfig.baseUrl),
                    ].map((item) => _EnvRow(label: item.$1, value: item.$2)),
                  ],
                ),
              ),
              const SizedBox(height: 24),

              // Paleta de colores
              Text(
                'DESIGN SYSTEM',
                style: tt.labelSmall?.copyWith(
                  letterSpacing: 1.2,
                  color: AppColors.textSecondary,
                ),
              ),
              const SizedBox(height: 12),
              Row(
                children: [
                  ('Accent',  AppColors.accent),
                  ('Success', AppColors.success),
                  ('Warning', AppColors.warning),
                  ('Error',   AppColors.error),
                  ('Info',    AppColors.info),
                ].map((item) => Expanded(
                  child: Column(
                    children: [
                      Container(
                        height:       44,
                        margin:       const EdgeInsets.symmetric(horizontal: 4),
                        decoration:   BoxDecoration(
                          color:        item.$2,
                          borderRadius: BorderRadius.circular(8),
                        ),
                      ),
                      const SizedBox(height: 4),
                      Text(
                        item.$1,
                        style: const TextStyle(fontSize: 9, color: AppColors.textFaint),
                        textAlign: TextAlign.center,
                      ),
                    ],
                  ),
                )).toList(),
              ),
              const SizedBox(height: 24),

              // Modelos de dominio
              Text(
                'MODELOS DE DOMINIO',
                style: tt.labelSmall?.copyWith(
                  letterSpacing: 1.2,
                  color: AppColors.textSecondary,
                ),
              ),
              const SizedBox(height: 12),
              ...[
                'auth_models.dart',
                'category.dart',
                'product.dart',
                'order.dart',
                'user.dart',
              ].map((f) => Padding(
                padding: const EdgeInsets.symmetric(vertical: 4),
                child: Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    Text(
                      'domain/model/$f',
                      style: const TextStyle(
                        color:      AppColors.accent,
                        fontSize:   11,
                        fontFamily: 'monospace',
                      ),
                    ),
                    const Text(
                      '✓',
                      style: TextStyle(color: AppColors.success, fontWeight: FontWeight.bold),
                    ),
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

class _EnvRow extends StatelessWidget {
  final String label;
  final String value;
  const _EnvRow({required this.label, required this.value});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 8),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceBetween,
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(label, style: const TextStyle(color: AppColors.textSecondary, fontSize: 13)),
          const SizedBox(width: 12),
          Flexible(
            child: Text(
              value,
              textAlign: TextAlign.end,
              style: const TextStyle(
                color:      AppColors.textPrimary,
                fontWeight: FontWeight.w600,
                fontSize:   13,
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## ✅ Checkpoint Módulo 1

```bash
flutter run
```

| Verificación | Resultado esperado |
|---|---|
| App compila sin errores | 0 errores en consola |
| Fondo oscuro `#0A0A0F` | Pantalla con tema dark |
| Título "Flutter Shop App" en dorado | Color `#D4A843` |
| Card de entorno | 6 filas con datos |
| Paleta de 5 colores | Chips coloreados |
| Modelos de dominio | 5 archivos con ✓ verde |

### Problemas comunes

```
Error: SDK constraint not satisfied
→ flutter upgrade

Error: No internet en emulador Android
→ Verificar android:usesCleartextTraffic="true" en AndroidManifest.xml

Error: CocoaPods not installed (iOS)
→ sudo gem install cocoapods && pod install
```

---

## Resumen

| Elemento | Estado |
|---|---|
| Proyecto Flutter 3 + Dart 3 creado | ✅ |
| `pubspec.yaml` con Riverpod, GoRouter, Dio, etc. | ✅ |
| `AppConfig` con `API_BASE_URL` via `--dart-define` | ✅ |
| `theme/app_colors.dart` — paleta dark + accent dorado | ✅ |
| `theme/app_theme.dart` — Material 3 completo | ✅ |
| `core/utils/formatters.dart` — precio, fecha, orderStatusColor | ✅ |
| `core/utils/validators.dart` — username, email, password, campos | ✅ |
| `domain/model/auth_models.dart` — AuthTokens, LoggedUser | ✅ |
| `domain/model/category.dart` — Category con copyWith | ✅ |
| `domain/model/product.dart` — Product, PaginatedProducts | ✅ |
| `domain/model/order.dart` — OrderStatus enum, OrderItem, Order | ✅ |
| `domain/model/user.dart` — User admin con copyWith | ✅ |
| `VerificationScreen` con pantalla de verificación | ✅ |

**Siguiente módulo →** M2: Dio + interceptores JWT + SecureStorage + repositorios base