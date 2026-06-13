


# Tutorial Flutter — Página 19
## Módulo 7 · Arquitectura
### Clean Architecture con Riverpod — proyecto integrador

---

## ¿Qué es Clean Architecture?

Clean Architecture divide la app en capas con una regla fundamental:
**las dependencias solo apuntan hacia adentro**. Las capas internas
no saben nada de las capas externas:

```
┌──────────────────────────────────────────────────┐
│  UI (Presentación)                               │
│  widgets, pantallas, notifiers                   │
├──────────────────────────────────────────────────┤
│  Domain (Negocio)                                │
│  entidades, repositorios (interfaces), use cases │
│  — SIN imports de Flutter, SIN imports de HTTP  │
├──────────────────────────────────────────────────┤
│  Data (Datos)                                    │
│  DTOs, repositorios (implementación), API, BD    │
└──────────────────────────────────────────────────┘

Regla: Data conoce Domain. UI conoce Domain. Nadie conoce UI.
```

### Comparativa con el enfoque anterior

```
Páginas 11–17 (sin arquitectura)   Esta página
────────────────────────────────   ─────────────────────────────────
Todo en el notifier                Notifier solo llama use cases
DTO mezclado con dominio           DTO ≠ Entidad de dominio
Repositorio acoplado al notifier   Interfaz de repositorio en domain
Sin use cases                      Cada operación = un use case
Difícil de testear                 Cada capa se testea por separado
```

---

## Proyecto: `productos_clean`

Construiremos la misma app de catálogo de la página 12,
pero con arquitectura limpia. La API es la misma:

```
GET https://higuera-billing-api.desarrollo-software.xyz/api/products/
    ?page=1&page_size=10
```

```bash
flutter create productos_clean
cd productos_clean
```

### `pubspec.yaml`

```yaml
name: productos_clean
description: "Catálogo con Clean Architecture"
publish_to: none
version: 1.0.0+1

environment:
  sdk: ^3.7.0

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod:  ^3.3.0
  http:              ^1.2.2

dev_dependencies:
  flutter_test:
    sdk: flutter
  mockito:           ^5.4.4
  build_runner:
  flutter_lints:     ^5.0.0

flutter:
  uses-material-design: true
```

```bash
flutter pub get
```

---

## Estructura completa del proyecto

```
lib/
├── main.dart
│
├── domain/                          ← CAPA DE DOMINIO
│   ├── entities/
│   │   └── producto.dart            ← entidad de negocio
│   ├── repositories/
│   │   └── i_productos_repository.dart  ← interfaz (contrato)
│   └── usecases/
│       ├── obtener_productos_uc.dart
│       └── buscar_productos_uc.dart
│
├── data/                            ← CAPA DE DATOS
│   ├── dtos/
│   │   └── producto_dto.dart        ← mapea JSON → entidad
│   └── repositories/
│       └── productos_repository_impl.dart  ← implementación real
│
└── ui/                              ← CAPA DE UI
    ├── providers/
    │   └── productos_providers.dart  ← providers Riverpod
    ├── notifiers/
    │   └── catalogo_notifier.dart    ← estado + lógica de UI
    └── screens/
        └── pantalla_catalogo.dart    ← widget
```

---

## Paso 1 — App mínima

**`lib/main.dart`**:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() {
  runApp(const ProviderScope(child: ProductosApp()));
}

class ProductosApp extends StatelessWidget {
  const ProductosApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Catálogo Clean',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const Scaffold(
        body: Center(child: Text('Paso 1 ✅')),
      ),
    );
  }
}
```

```bash
flutter run
```

---

## Paso 2 — Capa Domain

La capa de dominio no importa nada de Flutter ni de HTTP.
Es Dart puro — se puede reutilizar en cualquier plataforma.

### `lib/domain/entities/producto.dart`

```dart
/// Entidad de dominio — representa un Producto en la lógica de negocio.
/// Sin dependencias externas: ni Flutter, ni http, ni JSON.
class Producto {
  final int     id;
  final String  nombre;
  final double  precio;
  final String  categoria;
  final bool    activo;
  final int     stock;
  final String? imagenUrl;

  const Producto({
    required this.id,
    required this.nombre,
    required this.precio,
    required this.categoria,
    required this.activo,
    required this.stock,
    this.imagenUrl,
  });

  // Lógica de negocio pura — sin dependencias
  bool   get disponible   => activo && stock > 0;
  bool   get stockBajo    => stock > 0 && stock <= 5;
  String get nivelStock   => switch (stock) {
    0       => 'Agotado',
    <= 5    => 'Bajo',
    <= 20   => 'Normal',
    _       => 'Alto',
  };

  @override
  String toString() => 'Producto($id: $nombre @ \$$precio)';

  @override
  bool operator ==(Object other) =>
      other is Producto && other.id == id;

  @override
  int get hashCode => id.hashCode;
}
```

### `lib/domain/repositories/i_productos_repository.dart`

```dart
import '../entities/producto.dart';

/// Contrato del repositorio — define QUÉ puede hacer,
/// no CÓMO lo hace. La implementación vive en la capa Data.
///
/// La capa UI solo conoce esta interfaz — nunca la implementación.
abstract interface class IProductosRepository {
  /// Obtener una página de productos.
  /// Devuelve [({List<Producto> items, bool hayMas})] o lanza excepción.
  Future<({List<Producto> items, bool hayMas})> obtenerProductos({
    int    pagina        = 1,
    int    tamanioPagina = 10,
    String busqueda      = '',
  });

  /// Obtener un producto por su ID.
  /// Devuelve null si no existe.
  Future<Producto?> obtenerPorId(int id);
}
```

### `lib/domain/usecases/obtener_productos_uc.dart`

```dart
import '../entities/producto.dart';
import '../repositories/i_productos_repository.dart';

/// Use case: obtener la lista de productos.
///
/// Un use case encapsula UNA operación de negocio.
/// Recibe el repositorio por constructor — fácil de testear.
class ObtenerProductosUc {
  final IProductosRepository _repo;

  const ObtenerProductosUc(this._repo);

  /// Ejecutar el use case.
  /// El operador call() permite invocar como función: uc(pagina: 1)
  Future<({List<Producto> items, bool hayMas})> call({
    int    pagina        = 1,
    int    tamanioPagina = 10,
    String busqueda      = '',
  }) {
    return _repo.obtenerProductos(
      pagina:        pagina,
      tamanioPagina: tamanioPagina,
      busqueda:      busqueda,
    );
  }
}
```

### `lib/domain/usecases/buscar_productos_uc.dart`

```dart
import '../entities/producto.dart';
import '../repositories/i_productos_repository.dart';

/// Use case: buscar productos por texto.
/// Centraliza la lógica de búsqueda — validación incluida.
class BuscarProductosUc {
  final IProductosRepository _repo;

  const BuscarProductosUc(this._repo);

  Future<List<Producto>> call(String query) async {
    // Regla de negocio: no buscar si el query es demasiado corto
    if (query.trim().length < 2) return [];

    final resultado = await _repo.obtenerProductos(
      busqueda:      query.trim(),
      tamanioPagina: 20,
    );
    return resultado.items;
  }
}
```

```bash
flutter analyze
# La capa domain no debe tener errores
# Nota: aún no hay implementación — el compile falla si hay imports externos
```

---

## Paso 3 — Capa Data

La capa Data implementa los contratos definidos en Domain.
Aquí viven los DTOs, el cliente HTTP y el repositorio real.

### `lib/data/dtos/producto_dto.dart`

```dart
import '../../domain/entities/producto.dart';

/// DTO — Data Transfer Object.
/// Representa la estructura exacta del JSON que devuelve la API.
/// Los nombres de campos coinciden con el JSON (snake_case).
class ProductoDto {
  final int     id;
  final String  name;
  final String  price;          // La API devuelve String, no double
  final String? categoryName;
  final bool    isActive;
  final int     stock;
  final String? urlImage;

  const ProductoDto({
    required this.id,
    required this.name,
    required this.price,
    this.categoryName,
    required this.isActive,
    required this.stock,
    this.urlImage,
  });

  /// Construir desde el Map que devuelve jsonDecode()
  factory ProductoDto.fromJson(Map<String, dynamic> json) {
    return ProductoDto(
      id:           json['id']            as int,
      name:         json['name']          as String,
      price:        json['price']         as String,
      categoryName: json['category_name'] as String?,
      isActive:     json['is_active']     as bool? ?? false,
      stock:        json['stock']         as int?  ?? 0,
      urlImage:     json['url_image']     as String?,
    );
  }

  /// Convertir a entidad de dominio.
  /// Aquí hacemos la transformación: String → double, snake → camelCase.
  Producto toDomain() {
    return Producto(
      id:        id,
      nombre:    name,
      precio:    double.tryParse(price) ?? 0.0,
      categoria: categoryName ?? 'Sin categoría',
      activo:    isActive,
      stock:     stock,
      imagenUrl: urlImage,
    );
  }
}

/// DTO para la respuesta paginada de la API.
class PaginaDto {
  final int              count;
  final String?          next;
  final String?          previous;
  final List<ProductoDto> results;

  const PaginaDto({
    required this.count,
    required this.next,
    required this.previous,
    required this.results,
  });

  factory PaginaDto.fromJson(Map<String, dynamic> json) {
    return PaginaDto(
      count:    json['count']    as int,
      next:     json['next']     as String?,
      previous: json['previous'] as String?,
      results:  (json['results'] as List)
          .map((e) => ProductoDto.fromJson(e as Map<String, dynamic>))
          .toList(),
    );
  }

  bool get hayMas => next != null;
}
```

### `lib/data/repositories/productos_repository_impl.dart`

```dart
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

import '../../domain/entities/producto.dart';
import '../../domain/repositories/i_productos_repository.dart';
import '../dtos/producto_dto.dart';

/// Implementación real del repositorio.
/// Hace las llamadas HTTP y convierte DTOs a entidades de dominio.
///
/// La UI NUNCA importa esta clase — solo conoce IProductosRepository.
class ProductosRepositoryImpl implements IProductosRepository {
  static const _baseUrl =
      'https://higuera-billing-api.desarrollo-software.xyz/api';

  final http.Client _client;

  // El cliente HTTP se inyecta — facilita el testing con mocks
  const ProductosRepositoryImpl(this._client);

  @override
  Future<({List<Producto> items, bool hayMas})> obtenerProductos({
    int    pagina        = 1,
    int    tamanioPagina = 10,
    String busqueda      = '',
  }) async {
    final params = {
      'page':      '$pagina',
      'page_size': '$tamanioPagina',
      if (busqueda.isNotEmpty) 'search': busqueda,
    };

    final uri = Uri.parse('$_baseUrl/products/')
        .replace(queryParameters: params);

    try {
      final response = await _client
          .get(uri, headers: {'Accept': 'application/json'})
          .timeout(const Duration(seconds: 15));

      if (response.statusCode == 200) {
        final json  = jsonDecode(response.body) as Map<String, dynamic>;
        final pagina_ = PaginaDto.fromJson(json);

        return (
          items:  pagina_.results.map((d) => d.toDomain()).toList(),
          hayMas: pagina_.hayMas,
        );
      } else {
        throw HttpException(
            'Error ${response.statusCode}: ${response.body}');
      }
    } on SocketException {
      throw const SocketException('Sin conexión a internet');
    } on HttpException {
      rethrow;
    } catch (e) {
      throw Exception('Error inesperado: $e');
    }
  }

  @override
  Future<Producto?> obtenerPorId(int id) async {
    final uri = Uri.parse('$_baseUrl/products/$id/');

    try {
      final response = await _client
          .get(uri, headers: {'Accept': 'application/json'})
          .timeout(const Duration(seconds: 15));

      if (response.statusCode == 200) {
        final json = jsonDecode(response.body) as Map<String, dynamic>;
        return ProductoDto.fromJson(json).toDomain();
      }
      if (response.statusCode == 404) return null;

      throw HttpException('Error ${response.statusCode}');
    } on SocketException {
      throw const SocketException('Sin conexión a internet');
    } catch (e) {
      throw Exception('Error inesperado: $e');
    }
  }
}
```

```bash
flutter analyze
# Las tres capas deben estar libres de errores antes de continuar
```

---

## Paso 4 — Capa UI: providers y notifier

### `lib/ui/providers/productos_providers.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:http/http.dart' as http;

import '../../data/repositories/productos_repository_impl.dart';
import '../../domain/repositories/i_productos_repository.dart';
import '../../domain/usecases/obtener_productos_uc.dart';
import '../../domain/usecases/buscar_productos_uc.dart';

// ── Infraestructura ───────────────────────────────────────────────

/// Cliente HTTP — singleton reutilizable en toda la app
final httpClientProvider = Provider<http.Client>((ref) {
  final client = http.Client();
  ref.onDispose(client.close);   // cerrar al destruir el provider
  return client;
});

// ── Repositorio ───────────────────────────────────────────────────

/// El repositorio se expone como la INTERFAZ, no como la implementación.
/// Para cambiar la fuente de datos (ej: SQLite), solo cambia este provider.
final productosRepositoryProvider =
    Provider<IProductosRepository>((ref) {
  return ProductosRepositoryImpl(ref.watch(httpClientProvider));
});

// ── Use cases ─────────────────────────────────────────────────────

final obtenerProductosUcProvider =
    Provider<ObtenerProductosUc>((ref) {
  return ObtenerProductosUc(ref.watch(productosRepositoryProvider));
});

final buscarProductosUcProvider =
    Provider<BuscarProductosUc>((ref) {
  return BuscarProductosUc(ref.watch(productosRepositoryProvider));
});
```

### `lib/ui/notifiers/catalogo_notifier.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../domain/entities/producto.dart';
import '../../domain/usecases/obtener_productos_uc.dart';
import '../../domain/usecases/buscar_productos_uc.dart';
import '../providers/productos_providers.dart';

/// Estado inmutable del catálogo
class CatalogoState {
  final List<Producto> productos;
  final bool           cargando;
  final bool           cargandoMas;
  final String?        error;
  final int            paginaActual;
  final bool           hayMas;
  final String         busqueda;

  const CatalogoState({
    this.productos    = const [],
    this.cargando     = true,
    this.cargandoMas  = false,
    this.error,
    this.paginaActual = 1,
    this.hayMas       = true,
    this.busqueda     = '',
  });

  bool get estaVacio  => !cargando && productos.isEmpty;
  bool get tieneDatos => productos.isNotEmpty;

  CatalogoState copyWith({
    List<Producto>? productos,
    bool?           cargando,
    bool?           cargandoMas,
    String?         error,
    int?            paginaActual,
    bool?           hayMas,
    String?         busqueda,
    bool            limpiarError = false,
  }) {
    return CatalogoState(
      productos:    productos    ?? this.productos,
      cargando:     cargando     ?? this.cargando,
      cargandoMas:  cargandoMas  ?? this.cargandoMas,
      error:        limpiarError ? null : (error ?? this.error),
      paginaActual: paginaActual ?? this.paginaActual,
      hayMas:       hayMas       ?? this.hayMas,
      busqueda:     busqueda     ?? this.busqueda,
    );
  }
}

/// Notifier del catálogo — solo llama use cases, no sabe de HTTP ni JSON.
class CatalogoNotifier extends Notifier<CatalogoState> {
  ObtenerProductosUc get _obtenerUc =>
      ref.read(obtenerProductosUcProvider);

  BuscarProductosUc get _buscarUc =>
      ref.read(buscarProductosUcProvider);

  @override
  CatalogoState build() {
    // Cargar la primera página al inicializar
    Future.microtask(cargar);
    return const CatalogoState();
  }

  Future<void> cargar() async {
    state = state.copyWith(
        cargando: true, limpiarError: true);
    try {
      final resultado = await _obtenerUc(busqueda: state.busqueda);
      state = state.copyWith(
        productos:    resultado.items,
        hayMas:       resultado.hayMas,
        paginaActual: 1,
        cargando:     false,
      );
    } catch (e) {
      state = state.copyWith(
          cargando: false, error: e.toString());
    }
  }

  Future<void> buscar(String query) async {
    state = state.copyWith(
        busqueda: query, cargando: true, limpiarError: true);
    try {
      if (query.trim().isEmpty) {
        // Sin query: volver a la lista completa
        final resultado = await _obtenerUc();
        state = state.copyWith(
          productos:    resultado.items,
          hayMas:       resultado.hayMas,
          paginaActual: 1,
          cargando:     false,
        );
      } else {
        // Con query: usar el use case de búsqueda
        final items = await _buscarUc(query);
        state = state.copyWith(
          productos:    items,
          hayMas:       false,
          paginaActual: 1,
          cargando:     false,
        );
      }
    } catch (e) {
      state = state.copyWith(
          cargando: false, error: e.toString());
    }
  }

  Future<void> cargarMas() async {
    if (!state.hayMas || state.cargandoMas || state.busqueda.isNotEmpty) {
      return;
    }
    state = state.copyWith(cargandoMas: true);
    try {
      final siguiente = state.paginaActual + 1;
      final resultado = await _obtenerUc(pagina: siguiente);
      state = state.copyWith(
        productos:    [...state.productos, ...resultado.items],
        hayMas:       resultado.hayMas,
        paginaActual: siguiente,
        cargandoMas:  false,
      );
    } catch (e) {
      state = state.copyWith(
          cargandoMas: false, error: e.toString());
    }
  }
}

final catalogoProvider =
    NotifierProvider<CatalogoNotifier, CatalogoState>(
  CatalogoNotifier.new,
);
```

---

## Paso 5 — Capa UI: pantalla

### `lib/ui/screens/pantalla_catalogo.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../domain/entities/producto.dart';
import '../notifiers/catalogo_notifier.dart';

class PantallaCatalogo extends ConsumerStatefulWidget {
  const PantallaCatalogo({super.key});

  @override
  ConsumerState<PantallaCatalogo> createState() =>
      _PantallaCatalogoState();
}

class _PantallaCatalogoState
    extends ConsumerState<PantallaCatalogo> {
  final _scrollCtrl = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollCtrl.addListener(_onScroll);
  }

  @override
  void dispose() {
    _scrollCtrl.removeListener(_onScroll);
    _scrollCtrl.dispose();
    super.dispose();
  }

  void _onScroll() {
    final pos = _scrollCtrl.position;
    if (pos.pixels >= pos.maxScrollExtent - 200) {
      ref.read(catalogoProvider.notifier).cargarMas();
    }
  }

  @override
  Widget build(BuildContext context) {
    final estado = ref.watch(catalogoProvider);
    final cs     = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Catálogo'),
        backgroundColor: cs.primaryContainer,
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: () =>
                ref.read(catalogoProvider.notifier).cargar(),
          ),
        ],
      ),
      body: Column(
        children: [

          // Buscador
          Padding(
            padding: const EdgeInsets.fromLTRB(12, 8, 12, 4),
            child: SearchBar(
              hintText: 'Buscar producto...',
              leading:  const Icon(Icons.search),
              onChanged: (v) =>
                  ref.read(catalogoProvider.notifier).buscar(v),
              padding: const WidgetStatePropertyAll(
                EdgeInsets.symmetric(horizontal: 16),
              ),
            ),
          ),

          // Contenido
          Expanded(
            child: switch ((estado.cargando, estado.error,
                estado.estaVacio)) {

              // Cargando por primera vez
              (true, _, _) => const Center(
                  child: CircularProgressIndicator()),

              // Error sin datos
              (_, String e, _) when !estado.tieneDatos => _PantallaError(
                  mensaje:      e,
                  onReintentar: () =>
                      ref.read(catalogoProvider.notifier).cargar(),
                ),

              // Vacío (sin resultados de búsqueda)
              (_, _, true) => _EstadoVacio(
                  busqueda: estado.busqueda),

              // Lista de productos
              _ => Stack(
                  children: [
                    ListView.builder(
                      controller: _scrollCtrl,
                      padding:    const EdgeInsets.all(12),
                      itemCount:  estado.productos.length + 1,
                      itemBuilder: (_, i) {
                        if (i == estado.productos.length) {
                          return _PieDeLista(
                            cargandoMas: estado.cargandoMas,
                            hayMas:      estado.hayMas,
                          );
                        }
                        return _TarjetaProducto(
                          producto: estado.productos[i],
                        );
                      },
                    ),

                    // Banner de error no bloqueante (al cargar más)
                    if (estado.error != null && estado.tieneDatos)
                      Positioned(
                        bottom: 0, left: 0, right: 0,
                        child: Container(
                          padding: const EdgeInsets.all(12),
                          color:   cs.errorContainer,
                          child: Text(estado.error!,
                              style: TextStyle(
                                  color: cs.onErrorContainer)),
                        ),
                      ),
                  ],
                ),
            },
          ),
        ],
      ),
    );
  }
}

// ── Widgets auxiliares ────────────────────────────────────────────

class _TarjetaProducto extends StatelessWidget {
  final Producto producto;
  const _TarjetaProducto({required this.producto});

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Card(
      margin: const EdgeInsets.only(bottom: 10),
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Row(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [

            // Imagen o placeholder
            ClipRRect(
              borderRadius: BorderRadius.circular(8),
              child: producto.imagenUrl != null
                  ? Image.network(
                      producto.imagenUrl!,
                      width: 72, height: 72,
                      fit: BoxFit.cover,
                      errorBuilder: (_, __, ___) =>
                          _PlaceholderImagen(size: 72),
                    )
                  : _PlaceholderImagen(size: 72),
            ),

            const SizedBox(width: 12),

            // Datos del producto
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [

                  Row(children: [
                    Expanded(
                      child: Text(
                        producto.nombre,
                        style: const TextStyle(
                            fontWeight: FontWeight.bold,
                            fontSize: 15),
                      ),
                    ),
                    if (!producto.activo)
                      Container(
                        padding: const EdgeInsets.symmetric(
                            horizontal: 6, vertical: 2),
                        decoration: BoxDecoration(
                          color:        cs.errorContainer,
                          borderRadius: BorderRadius.circular(4),
                        ),
                        child: Text('Inactivo',
                            style: TextStyle(
                                fontSize: 10,
                                color: cs.onErrorContainer)),
                      ),
                  ]),

                  Text(producto.categoria,
                      style: TextStyle(
                          fontSize: 12,
                          color:    cs.onSurfaceVariant)),

                  const SizedBox(height: 8),

                  Row(children: [
                    Text(
                      '\$${producto.precio.toStringAsFixed(2)}',
                      style: TextStyle(
                          fontSize:   17,
                          fontWeight: FontWeight.bold,
                          color:      cs.primary),
                    ),
                    const Spacer(),
                    _BadgeStock(
                        nivel: producto.nivelStock,
                        stock: producto.stock),
                  ]),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _PlaceholderImagen extends StatelessWidget {
  final double size;
  const _PlaceholderImagen({required this.size});

  @override
  Widget build(BuildContext context) => Container(
    width: size, height: size,
    color: Colors.grey.shade100,
    child: Icon(Icons.image_outlined,
        size: size * 0.5, color: Colors.grey.shade400),
  );
}

class _BadgeStock extends StatelessWidget {
  final String nivel;
  final int    stock;
  const _BadgeStock({required this.nivel, required this.stock});

  Color get _color => switch (nivel) {
    'Agotado' => Colors.red,
    'Bajo'    => Colors.orange,
    'Normal'  => Colors.blue,
    _         => Colors.green,
  };

  @override
  Widget build(BuildContext context) => Row(
    mainAxisSize: MainAxisSize.min,
    children: [
      Icon(Icons.inventory_2_outlined, size: 13, color: _color),
      const SizedBox(width: 3),
      Text('$stock · $nivel',
          style: TextStyle(
              fontSize: 12, color: _color,
              fontWeight: FontWeight.w600)),
    ],
  );
}

class _PieDeLista extends StatelessWidget {
  final bool cargandoMas;
  final bool hayMas;
  const _PieDeLista({required this.cargandoMas, required this.hayMas});

  @override
  Widget build(BuildContext context) => Padding(
    padding: const EdgeInsets.symmetric(vertical: 16),
    child: Center(
      child: cargandoMas
          ? const CircularProgressIndicator()
          : !hayMas
              ? Text('Fin del catálogo',
                  style: TextStyle(color: Colors.grey.shade500))
              : null,
    ),
  );
}

class _EstadoVacio extends StatelessWidget {
  final String busqueda;
  const _EstadoVacio({required this.busqueda});

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    return Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Icon(Icons.search_off, size: 64, color: cs.onSurfaceVariant),
          const SizedBox(height: 12),
          Text(
            busqueda.isEmpty
                ? 'No hay productos disponibles'
                : 'Sin resultados para "$busqueda"',
            style: TextStyle(color: cs.onSurfaceVariant),
          ),
        ],
      ),
    );
  }
}

class _PantallaError extends StatelessWidget {
  final String       mensaje;
  final VoidCallback onReintentar;
  const _PantallaError({
    required this.mensaje,
    required this.onReintentar,
  });

  @override
  Widget build(BuildContext context) => Center(
    child: Padding(
      padding: const EdgeInsets.all(32),
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          const Icon(Icons.cloud_off, size: 64, color: Colors.red),
          const SizedBox(height: 16),
          Text(mensaje, textAlign: TextAlign.center,
              style: const TextStyle(color: Colors.red)),
          const SizedBox(height: 24),
          FilledButton.icon(
            onPressed: onReintentar,
            icon:  const Icon(Icons.refresh),
            label: const Text('Reintentar'),
          ),
        ],
      ),
    ),
  );
}
```

Actualiza `main.dart` para usar la pantalla real:

```dart
// lib/main.dart — final

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'ui/screens/pantalla_catalogo.dart';

void main() {
  runApp(const ProviderScope(child: ProductosApp()));
}

class ProductosApp extends StatelessWidget {
  const ProductosApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Catálogo Clean',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const PantallaCatalogo(),
    );
  }
}
```

```bash
flutter run
# La app carga el catálogo desde la API real
# El buscador filtra en tiempo real
# Al llegar al final de la lista, carga la siguiente página automáticamente
```

---

## Paso 6 — Tests de la arquitectura

La ventaja real de Clean Architecture es la testeabilidad.
Cada capa se testea por separado sin la UI:

```dart
// test/domain/producto_test.dart

import 'package:flutter_test/flutter_test.dart';
import 'package:productos_clean/domain/entities/producto.dart';

void main() {
  group('Producto — entidad de dominio', () {

    const base = Producto(
      id: 1, nombre: 'Teclado', precio: 89.99,
      categoria: 'Periféricos', activo: true, stock: 10,
    );

    test('disponible cuando activo y stock > 0', () {
      expect(base.disponible, isTrue);
    });

    test('no disponible cuando inactivo', () {
      const p = Producto(
        id: 1, nombre: 'T', precio: 1, categoria: 'C',
        activo: false, stock: 5,
      );
      expect(p.disponible, isFalse);
    });

    test('no disponible cuando sin stock', () {
      const p = Producto(
        id: 1, nombre: 'T', precio: 1, categoria: 'C',
        activo: true, stock: 0,
      );
      expect(p.disponible, isFalse);
    });

    test('nivelStock cubre todos los rangos', () {
      final casos = {
        0: 'Agotado',
        1: 'Bajo',
        5: 'Bajo',
        6: 'Normal',
        20: 'Normal',
        21: 'Alto',
      };
      for (final e in casos.entries) {
        final p = Producto(
          id: 1, nombre: 'T', precio: 1, categoria: 'C',
          activo: true, stock: e.key,
        );
        expect(p.nivelStock, e.value,
            reason: 'stock=${e.key}');
      }
    });

    test('igualdad basada en id', () {
      const p1 = Producto(
          id: 1, nombre: 'A', precio: 1, categoria: 'C',
          activo: true, stock: 1);
      const p2 = Producto(
          id: 1, nombre: 'B', precio: 2, categoria: 'D',
          activo: false, stock: 0);
      expect(p1, equals(p2));  // mismo id → iguales
    });
  });
}
```

```dart
// test/data/producto_dto_test.dart

import 'package:flutter_test/flutter_test.dart';
import 'package:productos_clean/data/dtos/producto_dto.dart';

void main() {
  group('ProductoDto', () {

    const jsonCompleto = {
      'id': 42, 'name': 'Monitor UHD',
      'price': '349.99', 'category_name': 'Monitores',
      'is_active': true, 'stock': 8,
      'url_image': 'https://img.test/monitor.jpg',
    };

    test('fromJson parsea todos los campos', () {
      final dto = ProductoDto.fromJson(jsonCompleto);
      expect(dto.id,           42);
      expect(dto.name,         'Monitor UHD');
      expect(dto.categoryName, 'Monitores');
      expect(dto.isActive,     isTrue);
      expect(dto.stock,        8);
    });

    test('toDomain convierte precio String → double', () {
      final entidad = ProductoDto.fromJson(jsonCompleto).toDomain();
      expect(entidad.precio,  349.99);
      expect(entidad.nombre,  'Monitor UHD');
    });

    test('usa valores por defecto para campos opcionales', () {
      final dto = ProductoDto.fromJson(
          {'id': 1, 'name': 'Test', 'price': '10.00'});
      expect(dto.categoryName, isNull);
      final entidad = dto.toDomain();
      expect(entidad.categoria, 'Sin categoría');
    });
  });
}
```

```dart
// test/domain/buscar_productos_uc_test.dart

import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/annotations.dart';
import 'package:mockito/mockito.dart';
import 'package:productos_clean/domain/entities/producto.dart';
import 'package:productos_clean/domain/repositories/i_productos_repository.dart';
import 'package:productos_clean/domain/usecases/buscar_productos_uc.dart';

@GenerateMocks([IProductosRepository])
import 'buscar_productos_uc_test.mocks.dart';

void main() {
  late MockIProductosRepository mockRepo;
  late BuscarProductosUc uc;

  setUp(() {
    mockRepo = MockIProductosRepository();
    uc       = BuscarProductosUc(mockRepo);
  });

  group('BuscarProductosUc', () {

    test('devuelve lista vacía si query < 2 caracteres', () async {
      // No debe llamar al repositorio
      final resultado = await uc('a');
      expect(resultado, isEmpty);
      verifyNever(mockRepo.obtenerProductos());
    });

    test('llama al repositorio con query válido', () async {
      when(mockRepo.obtenerProductos(
        busqueda:      'teclado',
        tamanioPagina: 20,
        pagina:        anyNamed('pagina'),
      )).thenAnswer((_) async => (
        items: [
          const Producto(
            id: 1, nombre: 'Teclado', precio: 89.99,
            categoria: 'Periféricos', activo: true, stock: 5,
          ),
        ],
        hayMas: false,
      ));

      final resultado = await uc('teclado');
      expect(resultado.length, 1);
      expect(resultado.first.nombre, 'Teclado');
    });

    test('no busca con espacios solamente', () async {
      final resultado = await uc('   ');
      expect(resultado, isEmpty);
    });
  });
}
```

```bash
# Generar mocks
dart run build_runner build --delete-conflicting-outputs

# Ejecutar todos los tests
flutter test

# Ejecutar solo tests de dominio
flutter test test/domain/
```

---

## Resumen de la arquitectura implementada

```
main.dart
    │
    └── ProviderScope
            │
    ui/providers/productos_providers.dart
            │
            ├── httpClientProvider        → http.Client
            ├── productosRepositoryProvider → IProductosRepository
            │                                  ↑ implementado por
            │                              ProductosRepositoryImpl
            ├── obtenerProductosUcProvider → ObtenerProductosUc
            └── buscarProductosUcProvider  → BuscarProductosUc
                        │
    ui/notifiers/catalogo_notifier.dart
                        │ llama use cases
    domain/usecases/    │
        ├── ObtenerProductosUc(repo)
        └── BuscarProductosUc(repo)
                        │ llama la interfaz
    domain/repositories/IProductosRepository
                        │ implementado por
    data/repositories/ProductosRepositoryImpl
                        │ parsea con
    data/dtos/ProductoDto.fromJson().toDomain()
```

---

## Ejercicios propuestos

1. **Use case de detalle** — Crea `ObtenerProductoPorIdUc` en
   `lib/domain/usecases/`. Agrega un `FutureProvider.family<Producto?, int>`
   en `productos_providers.dart` que lo use. Crea `lib/ui/screens/pantalla_detalle.dart`
   que reciba un ID, use el provider y muestre todos los campos del producto.

2. **Repositorio con caché** — Crea `ProductosRepositoryConCache` que
   implemente `IProductosRepository`. Internamente usa un `Map<String, List<Producto>>`
   como caché por búsqueda. En el provider, decide cuál implementación inyectar
   según una variable de entorno o un flag de configuración.

3. **Tests del notifier** — Usa `ProviderContainer` con override de
   `productosRepositoryProvider` para testear `CatalogoNotifier` sin HTTP.
   Verifica: estado inicial es cargando, al completar hay productos, al buscar
   filtra, al cargar más añade a la lista existente.

4. **Fake repository para desarrollo** — Crea `ProductosRepositoryFake`
   con datos hardcodeados (sin HTTP). En `main.dart`, cuando
   `const bool kUsarFake = true`, inyecta el fake en lugar de la implementación
   real. Esto permite desarrollar la UI sin depender de la API.

---

## Resumen de la página 19

- Clean Architecture divide la app en tres capas: **Domain** (reglas de negocio, sin dependencias), **Data** (fuentes de datos, implementa contratos de Domain) y **UI** (widgets, notifiers, solo conoce Domain).
- La **entidad** de dominio es Dart puro — sin `fromJson`, sin imports de Flutter. La conversión JSON → entidad ocurre en el DTO de la capa Data.
- La **interfaz** `IProductosRepository` en Domain define el contrato. La implementación `ProductosRepositoryImpl` en Data lo cumple. La UI solo importa la interfaz.
- Los **use cases** encapsulan una sola operación de negocio. `operator call()` permite invocarlos como funciones: `uc(pagina: 1)`.
- El **notifier** solo llama use cases — no sabe de HTTP ni de JSON. Si mañana cambias la API por GraphQL, el notifier no cambia.
- Los **providers de Riverpod** inyectan las dependencias: `httpClientProvider` → `repositoryProvider` → `useCaseProvider` → `notifierProvider`.
- Cada capa es **testeable de forma independiente**: domain con Dart puro, data con mocks del cliente HTTP, UI con mocks del repositorio.
- `@GenerateMocks([IProductosRepository])` genera un mock de la interfaz — no de la implementación. Así los tests no conocen los detalles de HTTP.

---

> **Siguiente página →** Página 20: Publicación — `flutter build apk/appbundle/ipa`,
> firma, Google Play Console, App Store Connect y CI/CD con GitHub Actions.