# Flutter Shop App — Módulo 12
## Imágenes: foto de producto y avatar de usuario

> **Objetivo:** Integrar la carga y visualización de imágenes en la app Flutter. Mostrar la imagen de producto con efecto shimmer y caché, mostrar el avatar del usuario con fallback de iniciales, y permitir subir imágenes vía `multipart/form-data` desde la galería o cámara del dispositivo.
> **Checkpoint final:** visualizar la imagen de un producto, subir una imagen de producto como usuario staff, actualizar el avatar del perfil, y verificar que un archivo mayor a 2 MB muestra un mensaje de error adecuado.

---

## 12.1 Dependencias

Agregar las siguientes dependencias a `pubspec.yaml`:

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter

  # — ya existentes —
  flutter_riverpod: ^2.5.1
  go_router: ^13.2.0
  dio: ^5.4.3
  flutter_secure_storage: ^9.0.0
  shared_preferences: ^2.2.3

  # — nuevas para imágenes —
  cached_network_image: ^3.3.1   # Caché de imágenes de red con placeholder
  image_picker: ^1.1.2           # Seleccionar imagen de galería / cámara
  http: ^1.2.1                   # MultipartRequest para subir archivos
```

Ejecutar:

```bash
flutter pub get
```

### Permisos Android

En `android/app/src/main/AndroidManifest.xml`, dentro de `<manifest>`:

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<!-- Para Android 12 e inferior: -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />
<uses-permission android:name="android.permission.CAMERA" />
```

### Permisos iOS

En `ios/Runner/Info.plist`, agregar:

```xml
<!-- ios/Runner/Info.plist -->
<key>NSPhotoLibraryUsageDescription</key>
<string>Necesitamos acceso a tu galería para seleccionar imágenes.</string>
<key>NSCameraUsageDescription</key>
<string>Necesitamos acceso a la cámara para tomar fotos.</string>
```

---

## 12.2 Widget `ProductImage`

Archivo: `lib/presentation/widgets/product_image.dart`

Widget reutilizable que muestra la imagen de un producto con efecto shimmer mientras carga, y un icono de placeholder cuando `imageUrl` es `null` o la carga falla.

```dart
// lib/presentation/widgets/product_image.dart
import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter/material.dart';

/// Widget que muestra la imagen de un producto desde una URL remota.
///
/// - Si [imageUrl] es null, muestra un placeholder con icono.
/// - Mientras carga, muestra un shimmer animado.
/// - Si la carga falla, muestra un icono de error.
class ProductImage extends StatelessWidget {
  const ProductImage({
    super.key,
    required this.imageUrl,
    this.width,
    this.height = 220,
    this.fit = BoxFit.cover,
    this.borderRadius = const BorderRadius.all(Radius.circular(12)),
  });

  final String? imageUrl;
  final double? width;
  final double height;
  final BoxFit fit;
  final BorderRadius borderRadius;

  @override
  Widget build(BuildContext context) {
    final colorScheme = Theme.of(context).colorScheme;

    return ClipRRect(
      borderRadius: borderRadius,
      child: imageUrl != null
          ? CachedNetworkImage(
              imageUrl: imageUrl!,
              width: width,
              height: height,
              fit: fit,
              placeholder: (context, url) => _ShimmerBox(
                width: width,
                height: height,
              ),
              errorWidget: (context, url, error) => _PlaceholderBox(
                width: width,
                height: height,
                colorScheme: colorScheme,
                icon: Icons.broken_image_outlined,
              ),
            )
          : _PlaceholderBox(
              width: width,
              height: height,
              colorScheme: colorScheme,
              icon: Icons.image_outlined,
            ),
    );
  }
}

// ---------------------------------------------------------------------------
// Subwidgets privados
// ---------------------------------------------------------------------------

class _ShimmerBox extends StatefulWidget {
  const _ShimmerBox({this.width, required this.height});

  final double? width;
  final double height;

  @override
  State<_ShimmerBox> createState() => _ShimmerBoxState();
}

class _ShimmerBoxState extends State<_ShimmerBox>
    with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<double> _animation;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 1200),
    )..repeat(reverse: true);

    _animation = Tween<double>(begin: 0.4, end: 1.0).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final base = Theme.of(context).colorScheme.surfaceContainerHighest;
    return AnimatedBuilder(
      animation: _animation,
      builder: (_, __) => Container(
        width: widget.width,
        height: widget.height,
        color: base.withValues(alpha: _animation.value),
      ),
    );
  }
}

class _PlaceholderBox extends StatelessWidget {
  const _PlaceholderBox({
    this.width,
    required this.height,
    required this.colorScheme,
    required this.icon,
  });

  final double? width;
  final double height;
  final ColorScheme colorScheme;
  final IconData icon;

  @override
  Widget build(BuildContext context) {
    return Container(
      width: width,
      height: height,
      color: colorScheme.surfaceContainerHighest,
      child: Center(
        child: Icon(
          icon,
          size: 48,
          color: colorScheme.onSurfaceVariant.withValues(alpha: 0.5),
        ),
      ),
    );
  }
}
```

### Uso básico

```dart
// Ejemplo de uso en cualquier pantalla:
ProductImage(
  imageUrl: product.imageUrl,   // String? — null muestra placeholder
  height: 260,
  fit: BoxFit.contain,
)
```

---

## 12.3 Widget `UserAvatar`

Archivo: `lib/presentation/widgets/user_avatar.dart`

Widget circular para el avatar del usuario. Si hay una URL válida, la muestra con caché. Si no hay imagen, usa las iniciales del nombre de usuario como fallback.

```dart
// lib/presentation/widgets/user_avatar.dart
import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter/material.dart';

/// Widget circular que muestra el avatar de un usuario.
///
/// Prioridad de visualización:
///   1. Imagen remota desde [avatarUrl] (si no es null).
///   2. Iniciales de [username] (si avatarUrl es null).
///   3. Icono de persona (si username también es null/vacío).
class UserAvatar extends StatelessWidget {
  const UserAvatar({
    super.key,
    this.avatarUrl,
    this.username,
    this.radius = 32,
    this.onTap,
  });

  final String? avatarUrl;
  final String? username;

  /// Radio del círculo en puntos lógicos.
  final double radius;

  /// Callback opcional para tap (p. ej. abrir selector de imagen).
  final VoidCallback? onTap;

  @override
  Widget build(BuildContext context) {
    final colorScheme = Theme.of(context).colorScheme;

    Widget avatar;

    if (avatarUrl != null) {
      avatar = CachedNetworkImage(
        imageUrl: avatarUrl!,
        imageBuilder: (context, imageProvider) => CircleAvatar(
          radius: radius,
          backgroundImage: imageProvider,
        ),
        placeholder: (context, url) => _InitialsAvatar(
          username: username,
          radius: radius,
          colorScheme: colorScheme,
        ),
        errorWidget: (context, url, error) => _InitialsAvatar(
          username: username,
          radius: radius,
          colorScheme: colorScheme,
        ),
      );
    } else {
      avatar = _InitialsAvatar(
        username: username,
        radius: radius,
        colorScheme: colorScheme,
      );
    }

    if (onTap != null) {
      return GestureDetector(
        onTap: onTap,
        child: Stack(
          children: [
            avatar,
            Positioned(
              bottom: 0,
              right: 0,
              child: Container(
                decoration: BoxDecoration(
                  color: colorScheme.primary,
                  shape: BoxShape.circle,
                ),
                padding: const EdgeInsets.all(4),
                child: Icon(
                  Icons.camera_alt,
                  size: radius * 0.5,
                  color: colorScheme.onPrimary,
                ),
              ),
            ),
          ],
        ),
      );
    }

    return avatar;
  }
}

// ---------------------------------------------------------------------------
// Subwidget privado: círculo con iniciales o icono
// ---------------------------------------------------------------------------

class _InitialsAvatar extends StatelessWidget {
  const _InitialsAvatar({
    this.username,
    required this.radius,
    required this.colorScheme,
  });

  final String? username;
  final double radius;
  final ColorScheme colorScheme;

  String get _initials {
    if (username == null || username!.isEmpty) return '';
    final parts = username!.trim().split(RegExp(r'\s+'));
    if (parts.length >= 2) {
      return '${parts[0][0]}${parts[1][0]}'.toUpperCase();
    }
    return username![0].toUpperCase();
  }

  @override
  Widget build(BuildContext context) {
    final initials = _initials;
    return CircleAvatar(
      radius: radius,
      backgroundColor: colorScheme.primaryContainer,
      child: initials.isNotEmpty
          ? Text(
              initials,
              style: TextStyle(
                fontSize: radius * 0.65,
                fontWeight: FontWeight.bold,
                color: colorScheme.onPrimaryContainer,
              ),
            )
          : Icon(
              Icons.person,
              size: radius,
              color: colorScheme.onPrimaryContainer,
            ),
    );
  }
}
```

### Uso básico

```dart
// Avatar sin callback de tap:
UserAvatar(
  avatarUrl: userProfile.avatarUrl,
  username: userProfile.username,
  radius: 40,
)

// Avatar con tap para subir imagen:
UserAvatar(
  avatarUrl: userProfile.avatarUrl,
  username: userProfile.username,
  radius: 48,
  onTap: () => ref.read(imageUploadProvider.notifier).pickAndUploadAvatar(),
)
```

---

## 12.4 Servicio de subida de imágenes

Archivo: `lib/data/remote/api/image_upload_service.dart`

Servicio que encapsula las peticiones `multipart/form-data` al API. Usa el paquete `http` (no Dio) para construir `MultipartRequest`, y lee el token Bearer desde `FlutterSecureStorage`.

```dart
// lib/data/remote/api/image_upload_service.dart
import 'dart:convert';
import 'dart:io';

import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:http/http.dart' as http;

/// URL base del API.
/// Usar 10.0.2.2 para el emulador Android (apunta al localhost del host).
/// Usar localhost para el simulador iOS o web.
const String _kBaseUrl = 'http://10.0.2.2:8000/api';

/// Excepción lanzada cuando la subida falla.
class ImageUploadException implements Exception {
  const ImageUploadException(this.message);
  final String message;

  @override
  String toString() => 'ImageUploadException: $message';
}

/// Servicio para subir imágenes al API mediante multipart/form-data.
class ImageUploadService {
  ImageUploadService({FlutterSecureStorage? storage})
      : _storage = storage ?? const FlutterSecureStorage();

  final FlutterSecureStorage _storage;

  // -------------------------------------------------------------------------
  // Privados
  // -------------------------------------------------------------------------

  Future<String?> _readToken() async {
    return _storage.read(key: 'access_token');
  }

  Map<String, String> _authHeaders(String token) => {
        'Authorization': 'Bearer $token',
      };

  /// Sube un archivo al [uri] usando el campo de formulario [fieldName].
  /// Devuelve el cuerpo de la respuesta decodificado como Map.
  Future<Map<String, dynamic>> _upload({
    required Uri uri,
    required String fieldName,
    required File file,
  }) async {
    final token = await _readToken();
    if (token == null) {
      throw const ImageUploadException('No autenticado. Inicia sesión primero.');
    }

    final mimeType = _mimeTypeFromPath(file.path);

    final request = http.MultipartRequest('PATCH', uri)
      ..headers.addAll(_authHeaders(token))
      ..files.add(
        await http.MultipartFile.fromPath(
          fieldName,
          file.path,
          contentType: mimeType,
        ),
      );

    final streamedResponse = await request.send().timeout(
      const Duration(seconds: 30),
      onTimeout: () {
        throw const ImageUploadException(
          'La solicitud tardó demasiado. Verifica tu conexión.',
        );
      },
    );

    final response = await http.Response.fromStream(streamedResponse);
    final body = jsonDecode(response.body) as Map<String, dynamic>;

    if (response.statusCode == 200 || response.statusCode == 201) {
      return body;
    }

    // Intentar extraer mensaje del API
    final detail = _extractError(body);
    throw ImageUploadException(detail);
  }

  String _extractError(Map<String, dynamic> body) {
    if (body.containsKey('detail')) return body['detail'].toString();
    if (body.containsKey('image')) {
      final v = body['image'];
      return v is List ? v.first.toString() : v.toString();
    }
    if (body.containsKey('avatar')) {
      final v = body['avatar'];
      return v is List ? v.first.toString() : v.toString();
    }
    return 'Error desconocido al subir la imagen.';
  }

  http.MediaType _mimeTypeFromPath(String path) {
    final ext = path.split('.').last.toLowerCase();
    switch (ext) {
      case 'jpg':
      case 'jpeg':
        return http.MediaType('image', 'jpeg');
      case 'png':
        return http.MediaType('image', 'png');
      case 'webp':
        return http.MediaType('image', 'webp');
      default:
        return http.MediaType('image', 'jpeg');
    }
  }

  // -------------------------------------------------------------------------
  // API pública
  // -------------------------------------------------------------------------

  /// Sube la imagen de un producto (requiere usuario staff).
  ///
  /// Endpoint: PATCH /api/products/{productId}/
  /// Campo:    `image`
  ///
  /// Devuelve la URL absoluta de la imagen o null.
  Future<String?> uploadProductImage({
    required int productId,
    required File file,
  }) async {
    final uri = Uri.parse('$_kBaseUrl/products/$productId/');
    final body = await _upload(uri: uri, fieldName: 'image', file: file);
    return body['image_url'] as String?;
  }

  /// Sube el avatar del usuario autenticado.
  ///
  /// Endpoint: PATCH /api/users/profile/
  /// Campo:    `avatar`
  ///
  /// Devuelve la URL absoluta del avatar o null.
  Future<String?> uploadAvatar({required File file}) async {
    final uri = Uri.parse('$_kBaseUrl/users/profile/');
    final body = await _upload(uri: uri, fieldName: 'avatar', file: file);
    return body['avatar_url'] as String?;
  }
}
```

> **Nota sobre la URL base:** En el emulador de Android, `10.0.2.2` es la IP especial que apunta al `localhost` del equipo host. Para el simulador de iOS o para web, usar `localhost`. Se puede externalizar esta constante a un archivo de configuración de entorno para facilitar el cambio.

---

## 12.5 Provider de imágenes

Archivo: `lib/presentation/providers/image_upload_provider.dart`

`StateNotifier` que gestiona el ciclo de vida de la subida: selección de archivo con `ImagePicker`, llamada al servicio, y notificación del resultado.

```dart
// lib/presentation/providers/image_upload_provider.dart
import 'dart:io';

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:image_picker/image_picker.dart';

import '../../data/remote/api/image_upload_service.dart';

// ---------------------------------------------------------------------------
// Estado
// ---------------------------------------------------------------------------

/// Estado del proceso de subida de imágenes.
sealed class ImageUploadState {
  const ImageUploadState();
}

/// Estado inicial o tras resetear.
class ImageUploadIdle extends ImageUploadState {
  const ImageUploadIdle();
}

/// Subida en progreso.
class ImageUploadLoading extends ImageUploadState {
  const ImageUploadLoading();
}

/// Subida completada con éxito.
class ImageUploadSuccess extends ImageUploadState {
  const ImageUploadSuccess({required this.imageUrl});

  /// URL absoluta de la imagen subida (puede ser null si el API no la devuelve).
  final String? imageUrl;
}

/// Error en la subida.
class ImageUploadError extends ImageUploadState {
  const ImageUploadError({required this.message});
  final String message;
}

// ---------------------------------------------------------------------------
// Notifier
// ---------------------------------------------------------------------------

class ImageUploadNotifier extends StateNotifier<ImageUploadState> {
  ImageUploadNotifier({
    ImageUploadService? service,
    ImagePicker? picker,
  })  : _service = service ?? ImageUploadService(),
        _picker = picker ?? ImagePicker(),
        super(const ImageUploadIdle());

  final ImageUploadService _service;
  final ImagePicker _picker;

  // -------------------------------------------------------------------------
  // Privados
  // -------------------------------------------------------------------------

  /// Muestra el bottom sheet de selección de fuente y devuelve el archivo.
  /// Devuelve null si el usuario cancela.
  Future<File?> _pickImage() async {
    // Abrir galería por defecto (la fuente puede extenderse a cámara con
    // un dialog previo si se desea).
    final picked = await _picker.pickImage(
      source: ImageSource.gallery,
      imageQuality: 90,     // Compresión leve para reducir tamaño
      maxWidth: 1920,
      maxHeight: 1920,
    );

    if (picked == null) return null;
    return File(picked.path);
  }

  Future<void> _handleUpload(Future<String?> Function(File) upload) async {
    try {
      final file = await _pickImage();
      if (file == null) {
        // Usuario canceló la selección — no cambiar estado
        return;
      }

      state = const ImageUploadLoading();

      final imageUrl = await upload(file);
      state = ImageUploadSuccess(imageUrl: imageUrl);
    } on ImageUploadException catch (e) {
      state = ImageUploadError(message: e.message);
    } catch (e) {
      state = ImageUploadError(
        message: 'Error inesperado: ${e.toString()}',
      );
    }
  }

  // -------------------------------------------------------------------------
  // API pública
  // -------------------------------------------------------------------------

  /// Selecciona una imagen de la galería y la sube como imagen del producto.
  Future<void> pickAndUploadProductImage(int productId) async {
    await _handleUpload(
      (file) => _service.uploadProductImage(productId: productId, file: file),
    );
  }

  /// Selecciona una imagen de la galería y la sube como avatar del usuario.
  Future<void> pickAndUploadAvatar() async {
    await _handleUpload(
      (file) => _service.uploadAvatar(file: file),
    );
  }

  /// Vuelve al estado inicial (útil tras mostrar error o éxito).
  void reset() => state = const ImageUploadIdle();
}

// ---------------------------------------------------------------------------
// Provider
// ---------------------------------------------------------------------------

/// Provider global para subida de imágenes.
///
/// Se usa `autoDispose` para limpiar el estado cuando ya no hay listeners
/// (p. ej. al navegar fuera de la pantalla).
final imageUploadProvider =
    StateNotifierProvider.autoDispose<ImageUploadNotifier, ImageUploadState>(
  (ref) => ImageUploadNotifier(),
);
```

### Escuchar el estado en un widget

```dart
// Ejemplo de uso en una pantalla con ConsumerWidget:
ref.listen<ImageUploadState>(imageUploadProvider, (previous, next) {
  if (next is ImageUploadSuccess) {
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('Imagen subida correctamente.')),
    );
    // Recargar datos si es necesario
    ref.invalidate(productDetailProvider(productId));
  } else if (next is ImageUploadError) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text(next.message),
        backgroundColor: Theme.of(context).colorScheme.error,
      ),
    );
    ref.read(imageUploadProvider.notifier).reset();
  }
});
```

---

## 12.6 Actualizar `ProductDetailScreen`

Archivo: `lib/presentation/screens/catalog/product_detail_screen.dart`

Integrar `ProductImage` en la parte superior de la pantalla. Si el usuario autenticado es staff, mostrar un FAB para subir una nueva imagen.

```dart
// lib/presentation/screens/catalog/product_detail_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../widgets/product_image.dart';
import '../../providers/auth_provider.dart';         // provider de sesión
import '../../providers/image_upload_provider.dart';
import '../../providers/product_detail_provider.dart';       // AsyncNotifierProvider

class ProductDetailScreen extends ConsumerWidget {
  const ProductDetailScreen({super.key, required this.productId});

  final int productId;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final productAsync = ref.watch(productDetailProvider(productId));
    final uploadState = ref.watch(imageUploadProvider);
    final authState   = ref.watch(authProvider);

    // Determinar si el usuario es staff
    final isStaff = authState.valueOrNull?.isStaff ?? false;

    // Escuchar resultado de la subida
    ref.listen<ImageUploadState>(imageUploadProvider, (_, next) {
      if (next is ImageUploadSuccess) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Imagen del producto actualizada.')),
        );
        ref.invalidate(productDetailProvider(productId));
        ref.read(imageUploadProvider.notifier).reset();
      } else if (next is ImageUploadError) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            content: Text(next.message),
            backgroundColor: Theme.of(context).colorScheme.error,
          ),
        );
        ref.read(imageUploadProvider.notifier).reset();
      }
    });

    return Scaffold(
      appBar: AppBar(
        title: productAsync.whenOrNull(
              data: (p) => Text(p.name),
            ) ??
            const Text('Detalle de producto'),
      ),
      floatingActionButton: isStaff
          ? FloatingActionButton.extended(
              onPressed: uploadState is ImageUploadLoading
                  ? null
                  : () => ref
                      .read(imageUploadProvider.notifier)
                      .pickAndUploadProductImage(productId),
              icon: uploadState is ImageUploadLoading
                  ? const SizedBox(
                      width: 20,
                      height: 20,
                      child: CircularProgressIndicator(strokeWidth: 2),
                    )
                  : const Icon(Icons.photo_camera),
              label: const Text('Cambiar imagen'),
            )
          : null,
      body: productAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.error_outline, size: 48),
              const SizedBox(height: 12),
              Text('Error al cargar el producto: $e'),
              const SizedBox(height: 12),
              ElevatedButton(
                onPressed: () => ref.invalidate(productDetailProvider(productId)),
                child: const Text('Reintentar'),
              ),
            ],
          ),
        ),
        data: (product) => SingleChildScrollView(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              // Imagen del producto con caché y shimmer
              ProductImage(
                imageUrl: product.imageUrl,
                width: double.infinity,
                height: 260,
                borderRadius: const BorderRadius.all(Radius.circular(16)),
              ),
              const SizedBox(height: 20),

              // Nombre y precio
              Text(
                product.name,
                style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                      fontWeight: FontWeight.bold,
                    ),
              ),
              const SizedBox(height: 8),
              Row(
                children: [
                  Text(
                    '\$${product.price.toStringAsFixed(2)}',
                    style: Theme.of(context).textTheme.titleLarge?.copyWith(
                          color: Theme.of(context).colorScheme.primary,
                          fontWeight: FontWeight.w600,
                        ),
                  ),
                  const SizedBox(width: 12),
                  Text(
                    'con imp.: \$${product.priceWithTax.toStringAsFixed(2)}',
                    style: Theme.of(context).textTheme.bodySmall?.copyWith(
                          color: Theme.of(context).colorScheme.onSurfaceVariant,
                        ),
                  ),
                ],
              ),
              const SizedBox(height: 12),

              // Estado de stock
              Chip(
                avatar: Icon(
                  product.inStock
                      ? Icons.check_circle_outline
                      : Icons.cancel_outlined,
                  size: 18,
                ),
                label: Text(product.inStock ? 'En stock' : 'Sin stock'),
                backgroundColor: product.inStock
                    ? Colors.green.shade50
                    : Colors.red.shade50,
              ),
              const SizedBox(height: 16),

              // Descripción
              if (product.description.isNotEmpty) ...[
                Text(
                  'Descripción',
                  style: Theme.of(context).textTheme.titleMedium?.copyWith(
                        fontWeight: FontWeight.bold,
                      ),
                ),
                const SizedBox(height: 6),
                Text(
                  product.description,
                  style: Theme.of(context).textTheme.bodyMedium,
                ),
                const SizedBox(height: 16),
              ],

              // Categoría
              Row(
                children: [
                  const Icon(Icons.category_outlined, size: 18),
                  const SizedBox(width: 6),
                  Text(
                    product.category.name,
                    style: Theme.of(context).textTheme.bodyMedium,
                  ),
                ],
              ),
              // Espacio extra para que el FAB no tape contenido
              const SizedBox(height: 80),
            ],
          ),
        ),
      ),
    );
  }
}
```

> **Nota:** El provider `productDetailProvider` y `authProvider` deben existir de módulos anteriores. El modelo `Product` debe tener los campos `imageUrl`, `priceWithTax` e `inStock` mapeados desde el JSON del API.

### Modelo `Product` — campos de imagen

Verificar que el modelo de dominio incluya `imageUrl`:

```dart
// lib/domain/model/product.dart  (extracto relevante)
class Product {
  const Product({
    required this.id,
    required this.name,
    required this.description,
    required this.price,
    required this.priceWithTax,
    required this.stock,
    required this.inStock,
    required this.isActive,
    required this.category,
    this.imageUrl,            // <-- campo nuevo, nullable
  });

  final int id;
  final String name;
  final String description;
  final double price;
  final double priceWithTax;
  final int stock;
  final bool inStock;
  final bool isActive;
  final Category category;
  final String? imageUrl;     // URL absoluta o null

  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id:           json['id'] as int,
      name:         json['name'] as String,
      description:  json['description'] as String? ?? '',
      price:        double.parse(json['price'].toString()),
      priceWithTax: double.parse(json['price_with_tax'].toString()),
      stock:        json['stock'] as int,
      inStock:      json['in_stock'] as bool,
      isActive:     json['is_active'] as bool,
      category:     Category.fromJson(json['category'] as Map<String, dynamic>),
      imageUrl:     json['image_url'] as String?,   // <-- mapear image_url
    );
  }
}
```

---

## 12.7 Actualizar `ProfileScreen`

Archivo: `lib/presentation/screens/profile/profile_screen.dart`

Mostrar el `UserAvatar` con soporte de tap para subir avatar. Escuchar el estado del upload y actualizar el perfil al finalizar.

```dart
// lib/presentation/screens/profile/profile_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../widgets/user_avatar.dart';
import '../../providers/auth_provider.dart';
import '../../providers/image_upload_provider.dart';
import '../../providers/profile_provider.dart';   // AsyncNotifierProvider del perfil

class ProfileScreen extends ConsumerWidget {
  const ProfileScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final profileAsync = ref.watch(profileProvider);
    final uploadState  = ref.watch(imageUploadProvider);

    // Escuchar resultado de la subida de avatar
    ref.listen<ImageUploadState>(imageUploadProvider, (_, next) {
      if (next is ImageUploadSuccess) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Avatar actualizado correctamente.')),
        );
        ref.invalidate(profileProvider);
        ref.read(imageUploadProvider.notifier).reset();
      } else if (next is ImageUploadError) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            content: Text(next.message),
            backgroundColor: Theme.of(context).colorScheme.error,
          ),
        );
        ref.read(imageUploadProvider.notifier).reset();
      }
    });

    return Scaffold(
      appBar: AppBar(title: const Text('Mi perfil')),
      body: profileAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.error_outline, size: 48),
              const SizedBox(height: 12),
              Text('Error al cargar el perfil: $e'),
              const SizedBox(height: 12),
              ElevatedButton(
                onPressed: () => ref.invalidate(profileProvider),
                child: const Text('Reintentar'),
              ),
            ],
          ),
        ),
        data: (profile) => SingleChildScrollView(
          padding: const EdgeInsets.symmetric(horizontal: 24, vertical: 32),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.center,
            children: [
              // Avatar con tap para cambiar
              Center(
                child: Stack(
                  alignment: Alignment.center,
                  children: [
                    UserAvatar(
                      avatarUrl: profile.avatarUrl,
                      username: profile.username,
                      radius: 52,
                      onTap: uploadState is ImageUploadLoading
                          ? null
                          : () => ref
                              .read(imageUploadProvider.notifier)
                              .pickAndUploadAvatar(),
                    ),
                    if (uploadState is ImageUploadLoading)
                      const CircularProgressIndicator(),
                  ],
                ),
              ),
              const SizedBox(height: 8),
              Text(
                'Toca el avatar para cambiarlo',
                style: Theme.of(context).textTheme.bodySmall?.copyWith(
                      color: Theme.of(context).colorScheme.onSurfaceVariant,
                    ),
              ),
              const SizedBox(height: 28),

              // Información del perfil
              _ProfileInfoCard(profile: profile),
            ],
          ),
        ),
      ),
    );
  }
}

// ---------------------------------------------------------------------------
// Subwidget: tarjeta de información del perfil
// ---------------------------------------------------------------------------

class _ProfileInfoCard extends StatelessWidget {
  const _ProfileInfoCard({required this.profile});

  final UserProfile profile; // modelo de dominio del perfil

  @override
  Widget build(BuildContext context) {
    final textTheme   = Theme.of(context).textTheme;
    final colorScheme = Theme.of(context).colorScheme;

    return Card(
      elevation: 0,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(16),
        side: BorderSide(color: colorScheme.outlineVariant),
      ),
      child: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 20, vertical: 16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            _InfoRow(
              icon: Icons.person_outline,
              label: 'Usuario',
              value: profile.username,
            ),
            const Divider(height: 24),
            _InfoRow(
              icon: Icons.email_outlined,
              label: 'Email',
              value: profile.email,
            ),
            if (profile.firstName.isNotEmpty || profile.lastName.isNotEmpty) ...[
              const Divider(height: 24),
              _InfoRow(
                icon: Icons.badge_outlined,
                label: 'Nombre',
                value: '${profile.firstName} ${profile.lastName}'.trim(),
              ),
            ],
            if (profile.isStaff) ...[
              const Divider(height: 24),
              Row(
                children: [
                  Icon(Icons.admin_panel_settings_outlined,
                      size: 20, color: colorScheme.primary),
                  const SizedBox(width: 12),
                  Text(
                    'Usuario staff',
                    style: textTheme.bodyMedium?.copyWith(
                      color: colorScheme.primary,
                      fontWeight: FontWeight.w600,
                    ),
                  ),
                ],
              ),
            ],
          ],
        ),
      ),
    );
  }
}

class _InfoRow extends StatelessWidget {
  const _InfoRow({
    required this.icon,
    required this.label,
    required this.value,
  });

  final IconData icon;
  final String label;
  final String value;

  @override
  Widget build(BuildContext context) {
    final textTheme   = Theme.of(context).textTheme;
    final colorScheme = Theme.of(context).colorScheme;

    return Row(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Icon(icon, size: 20, color: colorScheme.onSurfaceVariant),
        const SizedBox(width: 12),
        Expanded(
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(
                label,
                style: textTheme.labelSmall?.copyWith(
                  color: colorScheme.onSurfaceVariant,
                  letterSpacing: 0.8,
                ),
              ),
              const SizedBox(height: 2),
              Text(value, style: textTheme.bodyMedium),
            ],
          ),
        ),
      ],
    );
  }
}
```

### Modelo `UserProfile` — campos de imagen

Verificar que el modelo de dominio incluya `avatarUrl`:

```dart
// lib/domain/model/user_profile.dart  (extracto relevante)
class UserProfile {
  const UserProfile({
    required this.id,
    required this.username,
    required this.email,
    required this.firstName,
    required this.lastName,
    required this.isStaff,
    this.avatarUrl,           // <-- campo nuevo, nullable
  });

  final int id;
  final String username;
  final String email;
  final String firstName;
  final String lastName;
  final bool isStaff;
  final String? avatarUrl;    // URL absoluta o null

  factory UserProfile.fromJson(Map<String, dynamic> json) {
    return UserProfile(
      id:         json['id'] as int,
      username:   json['username'] as String,
      email:      json['email'] as String,
      firstName:  json['first_name'] as String? ?? '',
      lastName:   json['last_name'] as String? ?? '',
      isStaff:    json['is_staff'] as bool? ?? false,
      avatarUrl:  json['avatar_url'] as String?,  // <-- mapear avatar_url
    );
  }
}
```

---

## ✅ Checkpoint

Pasos de verificación a ejecutar en orden:

| # | Acción | Dónde | Resultado esperado |
|---|--------|-------|-------------------|
| 1 | Navegar al detalle de cualquier producto | `ProductDetailScreen` | Se muestra el shimmer mientras carga; si el producto tiene imagen, se muestra con caché; si no tiene, se muestra el ícono placeholder |
| 2 | Navegar al detalle de un producto sin imagen | `ProductDetailScreen` | Se muestra `Icons.image_outlined` centrado en el área de imagen |
| 3 | Iniciar sesión como usuario **no staff** | Login → `ProductDetailScreen` | El FAB de "Cambiar imagen" **no** se muestra |
| 4 | Iniciar sesión como usuario **staff** | Login → `ProductDetailScreen` | El FAB de "Cambiar imagen" **sí** se muestra |
| 5 | Pulsar FAB "Cambiar imagen" → seleccionar JPEG < 2 MB | Galería del dispositivo | La imagen se sube; aparece SnackBar "Imagen del producto actualizada."; la pantalla se recarga con la nueva imagen |
| 6 | Pulsar FAB "Cambiar imagen" → seleccionar archivo > 2 MB | Galería del dispositivo | Aparece SnackBar con el mensaje de error del API: "Image size must not exceed 2 MB." |
| 7 | Navegar a `ProfileScreen` sin avatar | Perfil | Se muestra el círculo con las iniciales del nombre de usuario |
| 8 | Tocar el avatar → seleccionar JPEG < 2 MB | Galería del dispositivo | El avatar se sube; aparece SnackBar "Avatar actualizado correctamente."; la pantalla se recarga con el nuevo avatar |
| 9 | Tocar el avatar → seleccionar PNG > 2 MB | Galería del dispositivo | Aparece SnackBar con error: "Image size must not exceed 2 MB." |
| 10 | Cancelar la selección de imagen en la galería | Galería del dispositivo | No ocurre ningún cambio de estado; no aparece SnackBar |
| 11 | Simular pérdida de conexión durante la subida | Ajustes del emulador | Aparece SnackBar con mensaje "La solicitud tardó demasiado. Verifica tu conexión." |
| 12 | Reiniciar la app y navegar al producto ya actualizado | `ProductDetailScreen` | La imagen se carga desde la caché de `CachedNetworkImage` sin nueva petición de red |

---

## Resumen

| Elemento | Archivo | Estado |
|---|---|---|
| Dependencias `cached_network_image`, `image_picker`, `http` | `pubspec.yaml` | ✅ |
| Permisos Android para galería y cámara | `AndroidManifest.xml` | ✅ |
| Permisos iOS para galería y cámara | `Info.plist` | ✅ |
| Widget `ProductImage` con shimmer y placeholder | `lib/presentation/widgets/product_image.dart` | ✅ |
| Widget `UserAvatar` con iniciales y tap-to-upload | `lib/presentation/widgets/user_avatar.dart` | ✅ |
| `ImageUploadService` con `uploadProductImage` y `uploadAvatar` | `lib/data/remote/api/image_upload_service.dart` | ✅ |
| `ImageUploadNotifier` con estados idle/loading/success/error | `lib/presentation/providers/image_upload_provider.dart` | ✅ |
| `ProductDetailScreen` con `ProductImage` y FAB para staff | `lib/presentation/screens/catalog/product_detail_screen.dart` | ✅ |
| `ProfileScreen` con `UserAvatar` tap-to-upload | `lib/presentation/screens/profile/profile_screen.dart` | ✅ |
| Modelo `Product` con campo `imageUrl` | `lib/domain/model/product.dart` | ✅ |
| Modelo `UserProfile` con campo `avatarUrl` | `lib/domain/model/user_profile.dart` | ✅ |
| Validación de error API (tamaño y tipo) mostrada en SnackBar | `ImageUploadNotifier._extractError` | ✅ |
| Timeout de 30 segundos en peticiones multipart | `ImageUploadService._upload` | ✅ |

**Siguiente módulo →** Módulo 13: Gestión del carrito de compras y proceso de pago
