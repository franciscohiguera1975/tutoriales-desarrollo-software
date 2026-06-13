# Tutorial Flutter — Página 12
## Módulo 4 · Conectividad
### API REST: `http`, modelos, repositorios y manejo de errores

---

## Dependencias

```yaml
# pubspec.yaml
dependencies:
  http: ^1.2.2           # cliente HTTP
  flutter_riverpod: ^3.3.0
```

La API de referencia del curso es la misma usada en el proyecto
integrador de Kotlin:

```
Base URL: https://higuera-billing-api.desarrollo-software.xyz/api
GET /products/?page=1&page_size=10
```

---

## Parte 1 — Modelo de datos

Antes de hacer la petición, definimos el modelo que mapea la respuesta:

```dart
// lib/data/models/producto_dto.dart

// DTO — Data Transfer Object — representa la respuesta de la API
// Los nombres coinciden exactamente con los campos del JSON
class ProductoDto {
  final int     id;
  final String  name;
  final String  price;           // la API devuelve String, no double
  final String  categoryName;
  final bool    isActive;
  final int     stock;
  final String? urlImage;

  const ProductoDto({
    required this.id,
    required this.name,
    required this.price,
    required this.categoryName,
    required this.isActive,
    required this.stock,
    this.urlImage,
  });

  // Construir desde JSON — factory constructor
  factory ProductoDto.fromJson(Map<String, dynamic> json) {
    return ProductoDto(
      id:           json['id'] as int,
      name:         json['name'] as String,
      price:        json['price'] as String,
      categoryName: json['category_name'] as String? ?? 'Sin categoría',
      isActive:     json['is_active'] as bool? ?? false,
      stock:        json['stock'] as int? ?? 0,
      urlImage:     json['url_image'] as String?,
    );
  }

  // Serializar a JSON
  Map<String, dynamic> toJson() => {
    'id':            id,
    'name':          name,
    'price':         price,
    'category_name': categoryName,
    'is_active':     isActive,
    'stock':         stock,
    'url_image':     urlImage,
  };

  @override
  String toString() => 'ProductoDto($id: $name @ \$$price)';
}

// Respuesta paginada — envuelve la lista de productos
class PaginaProductos {
  final int               count;
  final String?           next;
  final String?           previous;
  final List<ProductoDto> results;

  const PaginaProductos({
    required this.count,
    required this.next,
    required this.previous,
    required this.results,
  });

  factory PaginaProductos.fromJson(Map<String, dynamic> json) {
    return PaginaProductos(
      count:    json['count'] as int,
      next:     json['next'] as String?,
      previous: json['previous'] as String?,
      results:  (json['results'] as List)
          .map((e) => ProductoDto.fromJson(e as Map<String, dynamic>))
          .toList(),
    );
  }

  bool get hayMas     => next != null;
  bool get hayAnterior => previous != null;
}
```

### Modelo de dominio — separado del DTO

```dart
// lib/domain/models/producto.dart

// Modelo de dominio — limpio, sin dependencias de JSON
// Los tipos son correctos: price es double, no String
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

  // Nivel de stock como categoría
  String get nivelStock => switch (stock) {
    0           => 'Agotado',
    <= 5        => 'Bajo',
    <= 20       => 'Normal',
    _           => 'Alto',
  };

  bool get disponible => activo && stock > 0;

  @override
  String toString() => 'Producto($id: $nombre @ \$$precio)';
}

// Extensión — convierte DTO a modelo de dominio
extension ProductoDtoMapper on ProductoDto {
  Producto toDomain() => Producto(
    id:        id,
    nombre:    name,
    precio:    double.tryParse(price) ?? 0.0,
    categoria: categoryName,
    activo:    isActive,
    stock:     stock,
    imagenUrl: urlImage,
  );
}
```

---

## Parte 2 — Manejo de errores tipado

```dart
// lib/data/errors/api_error.dart

// Sealed class para representar todos los posibles fallos
sealed class ApiError {
  const ApiError();

  String get mensaje;
}

class ErrorRed extends ApiError {
  final String detalle;
  const ErrorRed(this.detalle);

  @override
  String get mensaje => 'Sin conexión: $detalle';
}

class ErrorServidor extends ApiError {
  final int    codigo;
  final String cuerpo;
  const ErrorServidor(this.codigo, this.cuerpo);

  @override
  String get mensaje => 'Error del servidor ($codigo)';
}

class ErrorNoAutorizado extends ApiError {
  const ErrorNoAutorizado();

  @override
  String get mensaje => 'No autorizado — inicia sesión';
}

class ErrorNotFound extends ApiError {
  const ErrorNotFound();

  @override
  String get mensaje => 'Recurso no encontrado';
}

class ErrorDesconocido extends ApiError {
  final Object error;
  const ErrorDesconocido(this.error);

  @override
  String get mensaje => 'Error inesperado: $error';
}

// Tipo Result — éxito o fallo
// Equivale a Result<T> de Kotlin
sealed class Result<T> {
  const Result();
}

class Success<T> extends Result<T> {
  final T data;
  const Success(this.data);
}

class Failure<T> extends Result<T> {
  final ApiError error;
  const Failure(this.error);
}
```

---

## Parte 3 — Cliente HTTP base

```dart
// lib/data/network/http_client.dart

import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

class HttpClient {
  final String  baseUrl;
  final String? token;
  final Duration timeout;

  const HttpClient({
    required this.baseUrl,
    this.token,
    this.timeout = const Duration(seconds: 15),
  });

  // Construir headers comunes
  Map<String, String> get _headers => {
    'Content-Type': 'application/json',
    'Accept':       'application/json',
    if (token != null) 'Authorization': 'Bearer $token',
  };

  // GET
  Future<Result<Map<String, dynamic>>> get(String path,
      {Map<String, String>? queryParams}) async {
    try {
      final uri = Uri.parse('$baseUrl$path')
          .replace(queryParameters: queryParams);

      final response = await http
          .get(uri, headers: _headers)
          .timeout(timeout);

      return _procesarRespuesta(response);
    } on SocketException catch (e) {
      return Failure(ErrorRed(e.message));
    } on TimeoutException {
      return Failure(const ErrorRed('Tiempo de espera agotado'));
    } catch (e) {
      return Failure(ErrorDesconocido(e));
    }
  }

  // POST
  Future<Result<Map<String, dynamic>>> post(
      String path, Map<String, dynamic> body) async {
    try {
      final uri = Uri.parse('$baseUrl$path');
      final response = await http
          .post(uri, headers: _headers, body: jsonEncode(body))
          .timeout(timeout);

      return _procesarRespuesta(response);
    } on SocketException catch (e) {
      return Failure(ErrorRed(e.message));
    } on TimeoutException {
      return Failure(const ErrorRed('Tiempo de espera agotado'));
    } catch (e) {
      return Failure(ErrorDesconocido(e));
    }
  }

  // Procesar la respuesta según el código HTTP
  Result<Map<String, dynamic>> _procesarRespuesta(http.Response response) {
    return switch (response.statusCode) {
      200 || 201 => _parsearExito(response.body),
      401        => const Failure(ErrorNoAutorizado()),
      404        => const Failure(ErrorNotFound()),
      >= 500     => Failure(ErrorServidor(response.statusCode, response.body)),
      _          => Failure(ErrorServidor(response.statusCode, response.body)),
    };
  }

  Result<Map<String, dynamic>> _parsearExito(String body) {
    try {
      final json = jsonDecode(body) as Map<String, dynamic>;
      return Success(json);
    } catch (e) {
      return Failure(ErrorDesconocido('JSON inválido: $e'));
    }
  }
}
```

---

## Parte 4 — Repositorio

```dart
// lib/data/repositories/productos_repository.dart

class ProductosRepository {
  final HttpClient _client;

  const ProductosRepository(this._client);

  // Obtener página de productos
  Future<Result<PaginaProductos>> obtenerProductos({
    int pagina   = 1,
    int tamanioPagina = 10,
    String? busqueda,
    bool?   soloActivos,
  }) async {
    final params = <String, String>{
      'page':      '$pagina',
      'page_size': '$tamanioPagina',
      if (busqueda != null && busqueda.isNotEmpty) 'search': busqueda,
      if (soloActivos != null) 'is_active': '$soloActivos',
    };

    final resultado = await _client.get('/products/', queryParams: params);

    return switch (resultado) {
      Success(data: final json) => Success(PaginaProductos.fromJson(json)),
      Failure(error: final e)   => Failure(e),
    };
  }

  // Obtener un producto por ID
  Future<Result<Producto>> obtenerProducto(int id) async {
    final resultado = await _client.get('/products/$id/');

    return switch (resultado) {
      Success(data: final json) =>
          Success(ProductoDto.fromJson(json).toDomain()),
      Failure(error: final e)   => Failure(e),
    };
  }
}
```

---

## Parte 5 — Providers de Riverpod

```dart
// lib/providers/productos_providers.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';

// Cliente HTTP — singleton
final httpClientProvider = Provider<HttpClient>((ref) {
  return const HttpClient(
    baseUrl: 'https://higuera-billing-api.desarrollo-software.xyz/api',
  );
});

// Repositorio — depende del cliente
final productosRepositoryProvider = Provider<ProductosRepository>((ref) {
  return ProductosRepository(ref.watch(httpClientProvider));
});

// Estado de la lista de productos
class ProductosState {
  final List<Producto> productos;
  final bool           cargando;
  final bool           cargandoMas;
  final ApiError?      error;
  final int            paginaActual;
  final bool           hayMas;
  final String         busqueda;

  const ProductosState({
    this.productos    = const [],
    this.cargando     = false,
    this.cargandoMas  = false,
    this.error,
    this.paginaActual = 1,
    this.hayMas       = true,
    this.busqueda     = '',
  });

  ProductosState copyWith({
    List<Producto>? productos,
    bool?           cargando,
    bool?           cargandoMas,
    ApiError?       error,
    int?            paginaActual,
    bool?           hayMas,
    String?         busqueda,
    bool            limpiarError = false,
  }) {
    return ProductosState(
      productos:    productos    ?? this.productos,
      cargando:     cargando     ?? this.cargando,
      cargandoMas:  cargandoMas  ?? this.cargandoMas,
      error:        limpiarError ? null : error ?? this.error,
      paginaActual: paginaActual ?? this.paginaActual,
      hayMas:       hayMas       ?? this.hayMas,
      busqueda:     busqueda     ?? this.busqueda,
    );
  }
}

class ProductosNotifier extends AsyncNotifier<ProductosState> {
  ProductosRepository get _repo => ref.read(productosRepositoryProvider);

  @override
  Future<ProductosState> build() async {
    return _cargarPagina(estado: const ProductosState());
  }

  // Cargar primera página
  Future<ProductosState> _cargarPagina({
    required ProductosState estado,
    int pagina = 1,
  }) async {
    final resultado = await _repo.obtenerProductos(
      pagina:        pagina,
      busqueda:      estado.busqueda,
    );

    return switch (resultado) {
      Success(data: final pagina_) => estado.copyWith(
          productos:    pagina_.results.map((d) => d.toDomain()).toList(),
          paginaActual: pagina,
          hayMas:       pagina_.hayMas,
          cargando:     false,
          limpiarError: true,
        ),
      Failure(error: final e) => estado.copyWith(
          cargando: false,
          error:    e,
        ),
    };
  }

  Future<void> recargar() async {
    final estadoActual = state.valueOrNull ?? const ProductosState();
    state = const AsyncLoading();
    state = AsyncData(await _cargarPagina(
      estado: estadoActual.copyWith(cargando: true),
    ));
  }

  Future<void> buscar(String texto) async {
    state = const AsyncLoading();
    final nuevoEstado = const ProductosState().copyWith(busqueda: texto);
    state = AsyncData(await _cargarPagina(estado: nuevoEstado));
  }

  Future<void> cargarMas() async {
    final actual = state.valueOrNull;
    if (actual == null || !actual.hayMas || actual.cargandoMas) return;

    // Actualizar estado para mostrar indicador de carga
    state = AsyncData(actual.copyWith(cargandoMas: true));

    final siguiente = actual.paginaActual + 1;
    final resultado = await _repo.obtenerProductos(
      pagina:   siguiente,
      busqueda: actual.busqueda,
    );

    final nuevo = switch (resultado) {
      Success(data: final pag) => actual.copyWith(
          // AÑADIR a la lista existente, no reemplazar
          productos:    [...actual.productos,
                          ...pag.results.map((d) => d.toDomain())],
          paginaActual: siguiente,
          hayMas:       pag.hayMas,
          cargandoMas:  false,
        ),
      Failure(error: final e) => actual.copyWith(
          cargandoMas: false,
          error:       e,
        ),
    };

    state = AsyncData(nuevo);
  }
}

final productosProvider =
    AsyncNotifierProvider<ProductosNotifier, ProductosState>(
  ProductosNotifier.new,
);
```

---

## Programa completo — catálogo de productos con paginación

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() => runApp(const ProviderScope(child: AppCatalogo()));

class AppCatalogo extends StatelessWidget {
  const AppCatalogo({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Catálogo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const PantallaCatalogo(),
    );
  }
}

class PantallaCatalogo extends ConsumerStatefulWidget {
  const PantallaCatalogo({super.key});

  @override
  ConsumerState<PantallaCatalogo> createState() => _PantallaCatalogoState();
}

class _PantallaCatalogoState extends ConsumerState<PantallaCatalogo> {
  final _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    // Detectar cuando llegar al final de la lista
    _scrollController.addListener(_onScroll);
  }

  @override
  void dispose() {
    _scrollController.removeListener(_onScroll);
    _scrollController.dispose();
    super.dispose();
  }

  void _onScroll() {
    final pos = _scrollController.position;
    // Si quedan menos de 200px para el final, cargar más
    if (pos.pixels >= pos.maxScrollExtent - 200) {
      ref.read(productosProvider.notifier).cargarMas();
    }
  }

  @override
  Widget build(BuildContext context) {
    final asyncState = ref.watch(productosProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Catálogo de productos'),
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: () =>
                ref.read(productosProvider.notifier).recargar(),
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
                  ref.read(productosProvider.notifier).buscar(v),
              padding: const WidgetStatePropertyAll(
                EdgeInsets.symmetric(horizontal: 16),
              ),
            ),
          ),

          // Contenido
          Expanded(
            child: asyncState.when(
              loading: () =>
                  const Center(child: CircularProgressIndicator()),

              error: (e, _) => _PantallaError(
                mensaje: e.toString(),
                onReintentar: () =>
                    ref.read(productosProvider.notifier).recargar(),
              ),

              data: (estado) {
                if (estado.cargando) {
                  return const Center(child: CircularProgressIndicator());
                }

                if (estado.productos.isEmpty) {
                  return _EstadoVacio(busqueda: estado.busqueda);
                }

                return Stack(
                  children: [
                    ListView.builder(
                      controller: _scrollController,
                      padding:    const EdgeInsets.all(12),
                      itemCount:  estado.productos.length + 1,
                      itemBuilder: (context, i) {
                        // Último elemento — indicador de carga o fin
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

                    // Error no bloqueante (al cargar más)
                    if (estado.error != null)
                      Positioned(
                        bottom: 0,
                        left:   0,
                        right:  0,
                        child: _BannerError(
                          mensaje: estado.error!.mensaje,
                        ),
                      ),
                  ],
                );
              },
            ),
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
      margin:  const EdgeInsets.only(bottom: 10),
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
                      width:  72,
                      height: 72,
                      fit:    BoxFit.cover,
                      errorBuilder: (_, __, ___) =>
                          _PlaceholderImagen(size: 72),
                    )
                  : _PlaceholderImagen(size: 72),
            ),
            const SizedBox(width: 12),

            // Datos
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  // Nombre + badge activo/inactivo
                  Row(
                    children: [
                      Expanded(
                        child: Text(
                          producto.nombre,
                          style: const TextStyle(
                            fontWeight: FontWeight.bold,
                            fontSize:   15,
                          ),
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
                          child: Text(
                            'Inactivo',
                            style: TextStyle(
                              fontSize: 10,
                              color:    cs.onErrorContainer,
                            ),
                          ),
                        ),
                    ],
                  ),

                  Text(
                    producto.categoria,
                    style: TextStyle(
                      fontSize: 12,
                      color:    cs.onSurfaceVariant,
                    ),
                  ),

                  const SizedBox(height: 8),

                  Row(
                    children: [
                      Text(
                        '\$${producto.precio.toStringAsFixed(2)}',
                        style: TextStyle(
                          fontSize:   17,
                          fontWeight: FontWeight.bold,
                          color:      cs.primary,
                        ),
                      ),
                      const Spacer(),
                      _BadgeStock(nivel: producto.nivelStock, stock: producto.stock),
                    ],
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

class _PlaceholderImagen extends StatelessWidget {
  final double size;
  const _PlaceholderImagen({required this.size});

  @override
  Widget build(BuildContext context) => Container(
    width:  size,
    height: size,
    color:  Colors.grey.shade100,
    child:  Icon(Icons.image_outlined,
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
  Widget build(BuildContext context) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        Icon(Icons.inventory_2_outlined, size: 14, color: _color),
        const SizedBox(width: 4),
        Text(
          '$stock · $nivel',
          style: TextStyle(
            fontSize:   12,
            color:      _color,
            fontWeight: FontWeight.w600,
          ),
        ),
      ],
    );
  }
}

class _PieDeLista extends StatelessWidget {
  final bool cargandoMas;
  final bool hayMas;
  const _PieDeLista({required this.cargandoMas, required this.hayMas});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 16),
      child: Center(
        child: cargandoMas
            ? const CircularProgressIndicator()
            : hayMas
                ? null
                : Text(
                    'No hay más productos',
                    style: TextStyle(
                      color:    Colors.grey.shade500,
                      fontSize: 13,
                    ),
                  ),
      ),
    );
  }
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
          const SizedBox(height: 16),
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
  final String   mensaje;
  final VoidCallback onReintentar;
  const _PantallaError({required this.mensaje, required this.onReintentar});

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(32),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            const Icon(Icons.cloud_off, size: 64, color: Colors.red),
            const SizedBox(height: 16),
            Text(
              mensaje,
              textAlign: TextAlign.center,
              style: const TextStyle(color: Colors.red),
            ),
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
}

class _BannerError extends StatelessWidget {
  final String mensaje;
  const _BannerError({required this.mensaje});

  @override
  Widget build(BuildContext context) {
    return Material(
      elevation: 4,
      child: Container(
        width:   double.infinity,
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
        color:   Theme.of(context).colorScheme.errorContainer,
        child: Row(
          children: [
            Icon(Icons.warning_amber,
                color: Theme.of(context).colorScheme.error,
                size: 18),
            const SizedBox(width: 8),
            Expanded(
              child: Text(
                mensaje,
                style: TextStyle(
                  color:    Theme.of(context).colorScheme.onErrorContainer,
                  fontSize: 13,
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

---

## Ejercicios propuestos

1. **Detalle de producto** — Crea `FutureProvider.family<Result<Producto>, int>`
   que llame a `productosRepository.obtenerProducto(id)`. En la pantalla de
   detalle, usa `ref.watch(productoProvider(id)).when(...)` para mostrar
   todos los campos con una imagen grande, precio formateado y el nivel de
   stock con color.

2. **Caché local** — Extiende `ProductosNotifier` para guardar los resultados
   en memoria entre búsquedas. Añade un `Map<String, List<Producto>> _cache`
   privado donde la clave es la búsqueda. Antes de llamar la API, revisa
   si ya hay datos en caché y devuélvelos instantáneamente.

3. **Interceptor de errores** — Añade a `HttpClient` un mecanismo de reintento
   automático. Si la petición falla con `ErrorRed`, reintenta hasta 2 veces
   con 500ms de espera. Si falla con `ErrorServidor` (5xx), reintenta 1 vez.
   No reintentes `ErrorNoAutorizado` ni `ErrorNotFound`.

4. **Paginación con RefreshIndicator** — Envuelve la `ListView` en un
   `RefreshIndicator`. Al hacer pull-to-refresh, llama a
   `ref.read(productosProvider.notifier).recargar()`. Asegúrate de que
   el `RefreshIndicator` desaparece cuando la carga termina.

---

## Resumen de la página 12

- El patrón DTO → Domain separa la representación de red (nombres del JSON, tipos de la API) del modelo de dominio limpio que usa la UI.
- `factory Clase.fromJson(Map<String, dynamic> json)` es el constructor estándar para parsear JSON. El operador `as` hace casting y lanza si el tipo no coincide.
- La `sealed class ApiError` tipifica todos los posibles fallos — la UI puede usar `switch` exhaustivo para manejar cada caso diferente.
- `Result<T>` con `Success<T>` y `Failure<T>` elimina las excepciones del flujo normal — los errores son valores, no excepciones inesperadas.
- El `HttpClient` centraliza el manejo de errores de red, timeouts y códigos HTTP. Los repositorios no necesitan saber de `try/catch`.
- La paginación infinita detecta el scroll con `ScrollController` y llama `cargarMas()` cuando queda poco para el final.
- `[...lista1, ...lista2]` añade los nuevos elementos a los existentes en la paginación — es el patrón correcto para "cargar más".
- El error no bloqueante (banner en `Positioned`) permite mostrar datos existentes mientras informa del error al cargar más páginas.

---

> **Siguiente página →** Página 13: Testing — `flutter_test`,
> `testWidgets`, mocks con Mockito y tests de providers con Riverpod.