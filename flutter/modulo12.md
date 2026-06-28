# Tutorial Flutter — Página 12
## Módulo 4 · Conectividad
### API REST: `http`, modelos, repositorios y manejo de errores

---

## 1. ¿Por qué esta arquitectura?

Cuando una app crece, hacer `http.get()` directamente en el widget mezcla
lógica de red, transformación de datos y UI en el mismo lugar. La arquitectura
DTO + Repositorio + Riverpod separa cada responsabilidad en su propia capa:

```
Sin arquitectura              Con DTO + Repositorio + Riverpod
───────────────────           ────────────────────────────────────
http.get() en el widget       HttpClient centralizado + timeout
JSON directo en la UI         DTO → Domain: tipos correctos
Sin manejo de errores         sealed class ApiError exhaustiva
Duplicación por pantalla      Repositorio reutilizable en toda la app
```

---

## 2. Dependencias

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.2
  flutter_riverpod: ^3.3.0
```

API base URL: `https://higuera-billing-api.desarrollo-software.xyz/api`

Endpoints usados en este módulo:
- `GET /products/?page=1&page_size=10`
- `GET /products/:id/`

---

## 3. Patrones clave — de un vistazo

Referencia rápida de los seis patrones del módulo. Léelos ahora y vuelve
a consultarlos conforme avanzas en cada paso.

---

### Bloque 1 — `http.get()` básico y `jsonDecode`

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

// GET simple — la forma más directa de llamar a una API
final uri      = Uri.parse('https://api.ejemplo.com/products/?page=1');
final response = await http.get(uri);

if (response.statusCode == 200) {
  final json = jsonDecode(response.body) as Map<String, dynamic>;
  print(json['count']); // total de registros devueltos por la API
} else {
  print('Error: ${response.statusCode}');
}
```

`Uri.parse` convierte el string en un objeto `Uri` tipado.
`jsonDecode` convierte el JSON en un `Map<String, dynamic>`.
`response.statusCode` es el código HTTP (200 = OK, 404 = no encontrado, etc.).

> **Mini-ejercicio ⏱ 5 min**
> Cambia la URL a `https://higuera-billing-api.desarrollo-software.xyz/api/products/?page=1&page_size=5`.
> ¿Qué devuelve `json['count']`? ¿Y `json['next']`? ¿Qué ocurre si cambias
> `page_size=5` a `page_size=100` siendo que hay menos de 100 productos?

---

### Bloque 2 — DTO con `fromJson`

```dart
// DTO — Data Transfer Object
// Sus campos coinciden exactamente con los del JSON de la API
class ProductoDto {
  final int    id;
  final String name;
  final String price;        // la API devuelve String, no double
  final String categoryName;
  final bool   isActive;
  final int    stock;

  const ProductoDto({
    required this.id,
    required this.name,
    required this.price,
    required this.categoryName,
    required this.isActive,
    required this.stock,
  });

  factory ProductoDto.fromJson(Map<String, dynamic> json) => ProductoDto(
    id:           json['id']            as int,
    name:         json['name']          as String,
    price:        json['price']         as String,
    categoryName: json['category_name'] as String? ?? 'Sin categoría',
    isActive:     json['is_active']     as bool?   ?? false,
    stock:        json['stock']         as int?    ?? 0,
  );
}

// Modelo de dominio — separado del DTO
// Los tipos son correctos: precio ya es double
class Producto {
  final int    id;
  final String nombre;
  final double precio;    // convertido a double en el mapper
  final String categoria;
  final bool   activo;
  final int    stock;

  const Producto({...});

  // Getters calculados — lógica de negocio en el modelo
  String get nivelStock => switch (stock) {
    0    => 'Agotado',
    <= 5  => 'Bajo',
    <= 20 => 'Normal',
    _     => 'Alto',
  };
}

// Extensión de conversión DTO → Domain
extension ProductoDtoMapper on ProductoDto {
  Producto toDomain() => Producto(
    id:        id,
    nombre:    name,
    precio:    double.tryParse(price) ?? 0.0, // "12.50" → 12.50
    categoria: categoryName,
    activo:    isActive,
    stock:     stock,
  );
}
```

> **Mini-ejercicio ⏱ 5 min**
> Añade al `Producto` un getter `bool disponible => activo && stock > 0`.
> ¿Cómo lo mostrarías en un `ListTile` con un ícono verde si disponible
> y gris si no? Pista: usa `Icon(Icons.circle, color: p.disponible ? Colors.green : Colors.grey)`.

---

### Bloque 3 — `sealed class ApiError` + `Result<T>`

```dart
// Sealed class — todos los posibles errores de la API
sealed class ApiError {
  const ApiError();
  String get mensaje;
}

class ErrorRed          extends ApiError { /* sin conexión */     }
class ErrorServidor     extends ApiError { final int codigo; ...  }
class ErrorNoAutorizado extends ApiError { /* 401 */              }
class ErrorNotFound     extends ApiError { /* 404 */              }
class ErrorDesconocido  extends ApiError { final Object error; .. }

// Result<T> — éxito o fallo, sin excepciones en el flujo normal
sealed class Result<T> { const Result(); }
class Success<T> extends Result<T> { final T data;      const Success(this.data); }
class Failure<T> extends Result<T> { final ApiError error; const Failure(this.error); }

// La UI usa switch exhaustivo — el compilador exige todos los casos
switch (resultado) {
  case Success(:final data):  mostrarDatos(data);
  case Failure(:final error): mostrarError(error.mensaje);
}
```

`sealed class` garantiza que el `switch` sea exhaustivo — si añades un nuevo
subtipo, el compilador señala todos los `switch` que hay que actualizar.

> **Mini-ejercicio ⏱ 5 min**
> Añade `class ErrorTimeout extends ApiError` a la sealed class con
> `String get mensaje => 'La petición tardó demasiado'`. Luego modifica
> el `HttpClient` del bloque siguiente para que atrape `TimeoutException`
> y devuelva `Failure(ErrorTimeout())` en lugar de `Failure(ErrorRed(...))`.

---

### Bloque 4 — `HttpClient` base

```dart
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

// Cliente HTTP centralizado — maneja errores, headers y timeout
class HttpClient {
  final String   baseUrl;
  final String?  token;
  final Duration timeout;

  const HttpClient({
    required this.baseUrl,
    this.token,
    this.timeout = const Duration(seconds: 15),
  });

  // Headers comunes a todas las peticiones
  Map<String, String> get _headers => {
    'Content-Type': 'application/json',
    'Accept':       'application/json',
    if (token != null) 'Authorization': 'Bearer $token',
  };

  Future<Result<Map<String, dynamic>>> get(String path,
      {Map<String, String>? queryParams}) async {
    try {
      final uri = Uri.parse('$baseUrl$path')
          .replace(queryParameters: queryParams);
      final response = await http.get(uri, headers: _headers).timeout(timeout);
      return _procesar(response);
    } on SocketException catch (e) {
      return Failure(ErrorRed(e.message));     // sin conexión
    } on TimeoutException {
      return Failure(const ErrorRed('Tiempo de espera agotado'));
    } catch (e) {
      return Failure(ErrorDesconocido(e));
    }
  }

  // Procesar la respuesta según el código HTTP
  Result<Map<String, dynamic>> _procesar(http.Response r) =>
      switch (r.statusCode) {
        200 || 201 => _ok(r.body),
        401        => const Failure(ErrorNoAutorizado()),
        404        => const Failure(ErrorNotFound()),
        >= 500     => Failure(ErrorServidor(r.statusCode, r.body)),
        _          => Failure(ErrorServidor(r.statusCode, r.body)),
      };

  Result<Map<String, dynamic>> _ok(String body) {
    try {
      return Success(jsonDecode(body) as Map<String, dynamic>);
    } catch (e) {
      return Failure(ErrorDesconocido('JSON inválido: $e'));
    }
  }
}
```

> **Mini-ejercicio ⏱ 5 min**
> Añade un método `post(String path, Map<String, dynamic> body)` al `HttpClient`.
> Pista: usa `http.post(uri, headers: _headers, body: jsonEncode(body))`.
> ¿Qué diferencia hay entre `get` y `post` en los headers?

---

### Bloque 5 — Repositorio

```dart
// El repositorio conoce la API; los widgets no saben de HTTP
class ProductosRepository {
  final HttpClient _client;
  const ProductosRepository(this._client);

  Future<Result<PaginaProductos>> obtenerProductos({
    int pagina = 1,
    int tamanioPagina = 10,
    String? busqueda,
  }) async {
    final params = <String, String>{
      'page':      '$pagina',
      'page_size': '$tamanioPagina',
      if (busqueda != null && busqueda.isNotEmpty) 'search': busqueda,
    };
    final r = await _client.get('/products/', queryParams: params);
    return switch (r) {
      Success(:final data) => Success(PaginaProductos.fromJson(data)),
      Failure(:final error) => Failure(error),
    };
  }
}
```

> **Mini-ejercicio ⏱ 5 min**
> Añade `obtenerProducto(int id)` al repositorio que llame a `/products/$id/`
> y devuelva `Result<Producto>`. Convierte el DTO con `.toDomain()`.
> ¿Cuántas líneas necesita este método?

---

### Bloque 6 — Providers con Riverpod

```dart
// Cliente — singleton compartido en toda la app
final httpClientProvider = Provider<HttpClient>((ref) => const HttpClient(
  baseUrl: 'https://higuera-billing-api.desarrollo-software.xyz/api',
));

// Repositorio — depende del cliente vía ref.watch
final productosRepositoryProvider = Provider<ProductosRepository>((ref) =>
  ProductosRepository(ref.watch(httpClientProvider)),
);

// Notifier asíncrono — gestiona estado de la lista
class ProductosNotifier extends AsyncNotifier<List<Producto>> {
  @override
  Future<List<Producto>> build() => _cargar(pagina: 1);

  Future<List<Producto>> _cargar({int pagina = 1}) async {
    final repo      = ref.read(productosRepositoryProvider);
    final resultado = await repo.obtenerProductos(pagina: pagina);
    return switch (resultado) {
      Success(:final data)  => data.results.map((d) => d.toDomain()).toList(),
      Failure(:final error) => throw Exception(error.mensaje),
    };
  }

  Future<void> recargar() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() => _cargar(pagina: 1));
  }
}

final productosProvider =
    AsyncNotifierProvider<ProductosNotifier, List<Producto>>(
  ProductosNotifier.new,
);
```

> **Mini-ejercicio ⏱ 5 min**
> Agrega un `StateProvider<String> busquedaProvider` y un
> `Provider<List<Producto>> productosFiltradosProvider` que filtre
> la lista por nombre en el cliente (sin llamar a la API).
> Pista: usa `ref.watch(productosProvider).valueOrNull ?? []`.

---

## 4. Crea el proyecto

```bash
flutter create modulo12_api
cd modulo12_api
```

`pubspec.yaml` — añade las dependencias:
```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.2
  flutter_riverpod: ^3.3.0
```

```bash
flutter pub get
```

Estructura de carpetas:
```
modulo12_api/
├── lib/
│   ├── main.dart
│   ├── data/
│   │   ├── models/
│   │   │   └── producto_dto.dart         ← Paso 2
│   │   ├── network/
│   │   │   └── http_client.dart          ← Paso 3
│   │   ├── errors/
│   │   │   └── api_error.dart            ← Paso 3
│   │   └── repositories/
│   │       └── productos_repository.dart ← Paso 4
│   ├── domain/
│   │   └── models/
│   │       └── producto.dart             ← Paso 2
│   ├── providers/
│   │   └── productos_providers.dart      ← Paso 4
│   └── screens/
│       ├── pantalla_productos.dart       ← Paso 4
│       └── pantalla_detalle.dart         ← Paso 5
├── pubspec.yaml
└── ...
```

---

## 5. Paso 1 — Primera petición HTTP con `http`

**Objetivo:** hacer un GET real a la API y mostrar el JSON crudo en pantalla.

`lib/main.dart`:
```dart
// lib/main.dart
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;

void main() => runApp(const AppBilling());

class AppBilling extends StatelessWidget {
  const AppBilling({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF1A237E)),
        useMaterial3: true,
      ),
      home: const _PantallaHttp(),
    );
  }
}

class _PantallaHttp extends StatefulWidget {
  const _PantallaHttp();
  @override
  State<_PantallaHttp> createState() => _PantallaHttpState();
}

class _PantallaHttpState extends State<_PantallaHttp> {
  String _respuesta  = 'Pulsa el botón para hacer la petición';
  bool   _cargando   = false;
  int    _statusCode = 0;

  Future<void> _hacerPeticion() async {
    setState(() { _cargando = true; _respuesta = 'Cargando...'; });

    try {
      final uri = Uri.parse(
        'https://higuera-billing-api.desarrollo-software.xyz/api/products/'
        '?page=1&page_size=3',
      );

      final response = await http.get(uri).timeout(
        const Duration(seconds: 15),
      );

      final json = jsonDecode(response.body);

      setState(() {
        _statusCode = response.statusCode;
        _respuesta  = const JsonEncoder.withIndent('  ').convert(json);
      });
    } catch (e) {
      setState(() {
        _statusCode = 0;
        _respuesta  = 'Error: $e';
      });
    } finally {
      setState(() => _cargando = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Paso 1 — http básico'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: Column(children: [
        // Banner con el código HTTP
        if (_statusCode > 0)
          Container(
            width:   double.infinity,
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            color:   _statusCode == 200
                ? Colors.green.shade50
                : Colors.red.shade50,
            child: Text(
              'HTTP $_statusCode — ${_statusCode == 200 ? 'OK' : 'Error'}',
              style: TextStyle(
                fontWeight: FontWeight.bold,
                color: _statusCode == 200
                    ? Colors.green.shade800
                    : Colors.red.shade800,
              ),
            ),
          ),

        // JSON crudo formateado
        Expanded(
          child: SingleChildScrollView(
            padding: const EdgeInsets.all(16),
            child: SelectableText(
              _respuesta,
              style: const TextStyle(
                fontFamily: 'monospace',
                fontSize:   12,
              ),
            ),
          ),
        ),

        // Botón de llamada
        Padding(
          padding: const EdgeInsets.all(16),
          child: SizedBox(
            width: double.infinity,
            child: FilledButton.icon(
              onPressed: _cargando ? null : _hacerPeticion,
              icon:  _cargando
                  ? const SizedBox(
                      width: 18, height: 18,
                      child: CircularProgressIndicator(strokeWidth: 2),
                    )
                  : const Icon(Icons.cloud_download),
              label: Text(_cargando ? 'Cargando...' : 'Obtener productos (página 1)'),
            ),
          ),
        ),
      ]),
    );
  }
}
```

### Prueba esto

- Ejecuta la app y pulsa "Obtener productos" — el JSON completo aparece formateado
- Observa los campos: `count`, `next`, `previous`, `results`
- Fíjate en `results[0]` — `price` es un `String`, no un `double` (aquí viene el DTO)
- Cambia `page_size=3` a `page_size=1` — ¿qué cambia en `results`? ¿y en `next`?
- Desactiva el WiFi y pulsa el botón — aparece el mensaje de error de red
- Cambia el timeout a `const Duration(milliseconds: 100)` — ¿qué error aparece?

---

## 6. Convierte `main.dart` en selector de pasos

A partir de aquí, cada paso añade una pantalla nueva. El selector permite
cambiar de paso modificando solo una constante y guardando:

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1  http.get() crudo — ver JSON en pantalla             │
// │  2  Paso 2  DTO + fromJson + mostrar lista real                  │
// │  3  Paso 3  ApiError + Result<T> — errores tipados              │
// │  4  Paso 4  HttpClient + Repositorio + Riverpod                 │
// │  5  Paso 5  Paginación infinita + búsqueda                      │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 1;

void main() => runApp(
  const ProviderScope(child: AppBilling(paso: paso)),
);

class AppBilling extends StatelessWidget {
  final int paso;
  const AppBilling({super.key, required this.paso});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF1A237E)),
        useMaterial3: true,
      ),
      home: switch (paso) {
        1 => const _PasoCrudo(),
        2 => const _PasoDto(),
        3 => const _PasoErrores(),
        4 => const PantallaProductos(),
        5 => const PantallaProductosPaginados(),
        _ => Scaffold(body: Center(child: Text('Paso $paso no definido'))),
      },
    );
  }
}
```

---

## 7. Paso 2 — DTO + `fromJson` + mostrar lista real

**Objetivo:** parsear el JSON con un DTO y mostrar la lista de productos con datos reales.

Crea `lib/data/models/producto_dto.dart`:

```dart
// lib/data/models/producto_dto.dart

class ProductoDto {
  final int     id;
  final String  name;
  final String  price;
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

  factory ProductoDto.fromJson(Map<String, dynamic> json) => ProductoDto(
    id:           json['id']            as int,
    name:         json['name']          as String,
    price:        json['price']         as String,
    categoryName: json['category_name'] as String? ?? 'Sin categoría',
    isActive:     json['is_active']     as bool?   ?? false,
    stock:        json['stock']         as int?    ?? 0,
    urlImage:     json['url_image']     as String?,
  );
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

  factory PaginaProductos.fromJson(Map<String, dynamic> json) => PaginaProductos(
    count:    json['count']    as int,
    next:     json['next']     as String?,
    previous: json['previous'] as String?,
    results:  (json['results'] as List)
        .map((e) => ProductoDto.fromJson(e as Map<String, dynamic>))
        .toList(),
  );

  bool get hayMas      => next     != null;
  bool get hayAnterior => previous != null;
}
```

Crea `lib/domain/models/producto.dart`:

```dart
// lib/domain/models/producto.dart
import 'package:flutter/material.dart';
import '../data/models/producto_dto.dart';

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

  // Categoría de stock — lógica de negocio en el modelo de dominio
  String get nivelStock => switch (stock) {
    0    => 'Agotado',
    <= 5  => 'Bajo',
    <= 20 => 'Normal',
    _     => 'Alto',
  };

  bool get disponible => activo && stock > 0;

  // Color para la UI según el nivel de stock
  Color get colorStock => switch (stock) {
    0    => const Color(0xFFB71C1C), // rojo oscuro
    <= 5  => const Color(0xFFE65100), // naranja
    <= 20 => const Color(0xFF1B5E20), // verde oscuro
    _     => const Color(0xFF0D47A1), // azul oscuro
  };

  @override
  String toString() => 'Producto($id: $nombre @ \$$precio)';
}

// Extensión — convierte DTO a modelo de dominio
extension ProductoDtoMapper on ProductoDto {
  Producto toDomain() => Producto(
    id:        id,
    nombre:    name,
    precio:    double.tryParse(price) ?? 0.0, // "12.50" → 12.50
    categoria: categoryName,
    activo:    isActive,
    stock:     stock,
    imagenUrl: urlImage,
  );
}
```

El widget `_PasoDto` en `main.dart` hace la petición, parsea con `PaginaProductos.fromJson`
y muestra la lista con `ListView.builder`:

```dart
// Añadir en lib/main.dart — importar dart:convert, http y los modelos
class _PasoDto extends StatefulWidget {
  const _PasoDto();
  @override
  State<_PasoDto> createState() => _PasoDtoState();
}

class _PasoDtoState extends State<_PasoDto> {
  List<Producto> _productos = [];
  bool   _cargando = false;
  String _error    = '';

  @override
  void initState() {
    super.initState();
    _cargar(); // carga automática al abrir la pantalla
  }

  Future<void> _cargar() async {
    setState(() { _cargando = true; _error = ''; });
    try {
      final uri = Uri.parse(
        'https://higuera-billing-api.desarrollo-software.xyz/api/products/'
        '?page=1&page_size=10',
      );
      final response = await http.get(uri).timeout(const Duration(seconds: 15));

      if (response.statusCode == 200) {
        final json   = jsonDecode(response.body) as Map<String, dynamic>;
        final pagina = PaginaProductos.fromJson(json);
        setState(() {
          _productos = pagina.results.map((d) => d.toDomain()).toList();
        });
      } else {
        setState(() => _error = 'Error HTTP ${response.statusCode}');
      }
    } catch (e) {
      setState(() => _error = 'Error de red: $e');
    } finally {
      setState(() => _cargando = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Paso 2 — DTO + fromJson'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: _cargando ? null : _cargar,
          ),
        ],
      ),
      body: _cargando
          ? const Center(child: CircularProgressIndicator())
          : _error.isNotEmpty
              ? Center(child: Text(_error, style: TextStyle(color: cs.error)))
              : ListView.separated(
                  itemCount:        _productos.length,
                  separatorBuilder: (_, __) => const Divider(height: 1),
                  itemBuilder: (context, i) {
                    final p = _productos[i];
                    return ListTile(
                      leading: CircleAvatar(
                        backgroundColor: p.disponible
                            ? Colors.green.shade50
                            : Colors.grey.shade100,
                        child: Text(
                          '${p.stock}',
                          style: TextStyle(
                            fontSize:   11,
                            fontWeight: FontWeight.bold,
                            color: p.disponible ? Colors.green : Colors.grey,
                          ),
                        ),
                      ),
                      title:    Text(p.nombre,
                          style: const TextStyle(fontWeight: FontWeight.w600)),
                      subtitle: Text('${p.categoria} — \$${p.precio.toStringAsFixed(2)}'),
                      trailing: Text(
                        p.nivelStock,
                        style: TextStyle(
                          color:      p.colorStock,
                          fontWeight: FontWeight.w600,
                          fontSize:   12,
                        ),
                      ),
                    );
                  },
                ),
    );
  }
}
```

### Agrega al `main.dart`

Añade los imports de `producto_dto.dart` y `producto.dart`, y cambia `paso = 2`.

### Prueba esto

- Ejecuta — carga automáticamente la lista de productos reales de la API
- Observa `nivelStock` y `disponible` — datos calculados en el modelo de dominio
- Abre `producto_dto.dart` y cambia `json['category_name']` a `json['categoria']` — ¿qué pasa? (devuelve `'Sin categoría'` porque el campo no existe)
- En el DTO, `price` es `String`. En `Producto`, `precio` es `double`. El mapper hace la conversión con `double.tryParse(price)`. ¿Qué pasa si la API devuelve `"N/A"` para el precio? (devuelve `0.0`)

---

## 8. Paso 3 — `ApiError` tipado + `Result<T>`

**Objetivo:** reemplazar el `try/catch` manual con un sistema de errores tipados.

Crea `lib/data/errors/api_error.dart`:

```dart
// lib/data/errors/api_error.dart

// Sealed class — todos los posibles fallos, exhaustivos en switch
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

// Result<T> — valor tipado: éxito o fallo
// Los errores son valores, no excepciones inesperadas
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

El widget `_PasoErrores` en `main.dart` usa un helper que devuelve `Result<PaginaProductos>`:

```dart
// Añadir en lib/main.dart — importar dart:io para SocketException
Future<Result<PaginaProductos>> _fetchProductos() async {
  try {
    final uri = Uri.parse(
      'https://higuera-billing-api.desarrollo-software.xyz/api/products/'
      '?page=1&page_size=10',
    );
    final response = await http.get(uri).timeout(const Duration(seconds: 15));

    return switch (response.statusCode) {
      200 => () {
        final json   = jsonDecode(response.body) as Map<String, dynamic>;
        final pagina = PaginaProductos.fromJson(json);
        return Success(pagina);
      }(),
      401 => const Failure(ErrorNoAutorizado()),
      404 => const Failure(ErrorNotFound()),
      _   => Failure(ErrorServidor(response.statusCode, response.body)),
    };
  } on SocketException catch (e) {
    return Failure(ErrorRed(e.message));
  } on TimeoutException {
    return Failure(const ErrorRed('Tiempo agotado'));
  } catch (e) {
    return Failure(ErrorDesconocido(e));
  }
}
```

La UI usa `switch` exhaustivo sobre el resultado:

```dart
// En _PasoErroresState._cargar()
final resultado = await _fetchProductos();
switch (resultado) {
  case Success(:final data):
    setState(() =>
        _productos = data.results.map((d) => d.toDomain()).toList());
  case Failure(:final error):
    setState(() => _errorMsg = error.mensaje);
}
```

### Prueba esto

- Desactiva el WiFi — aparece `ErrorRed` con el mensaje de red
- Cambia la URL a `https://higuera-billing-api.desarrollo-software.xyz/api/products/99999/` — aparece `ErrorNotFound`
- Cambia la URL base a una que no exista — ¿qué tipo de `ApiError` lanza?
- El `switch` sobre `Result<T>` es **exhaustivo** — prueba a quitar el `case Failure` y observa el error de compilación

---

## 9. Paso 4 — `HttpClient` + `ProductosRepository` + Riverpod

**Objetivo:** arquitectura limpia — la UI solo habla con el provider, no con HTTP.

Crea `lib/data/network/http_client.dart`:

```dart
// lib/data/network/http_client.dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;
import '../errors/api_error.dart';

class HttpClient {
  final String   baseUrl;
  final String?  token;
  final Duration timeout;

  const HttpClient({
    required this.baseUrl,
    this.token,
    this.timeout = const Duration(seconds: 15),
  });

  Map<String, String> get _headers => {
    'Content-Type': 'application/json',
    'Accept':       'application/json',
    if (token != null) 'Authorization': 'Bearer $token',
  };

  // GET con query params opcionales
  Future<Result<Map<String, dynamic>>> get(String path,
      {Map<String, String>? queryParams}) async {
    try {
      final uri = Uri.parse('$baseUrl$path')
          .replace(queryParameters: queryParams);
      final response =
          await http.get(uri, headers: _headers).timeout(timeout);
      return _procesarRespuesta(response);
    } on SocketException catch (e) {
      return Failure(ErrorRed(e.message));
    } on TimeoutException {
      return Failure(const ErrorRed('Tiempo de espera agotado'));
    } catch (e) {
      return Failure(ErrorDesconocido(e));
    }
  }

  // POST con cuerpo JSON
  Future<Result<Map<String, dynamic>>> post(
      String path, Map<String, dynamic> body) async {
    try {
      final uri      = Uri.parse('$baseUrl$path');
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

  // Despacha según el código HTTP — switch exhaustivo
  Result<Map<String, dynamic>> _procesarRespuesta(http.Response r) =>
      switch (r.statusCode) {
        200 || 201 => _parsearExito(r.body),
        401        => const Failure(ErrorNoAutorizado()),
        404        => const Failure(ErrorNotFound()),
        >= 500     => Failure(ErrorServidor(r.statusCode, r.body)),
        _          => Failure(ErrorServidor(r.statusCode, r.body)),
      };

  Result<Map<String, dynamic>> _parsearExito(String body) {
    try {
      return Success(jsonDecode(body) as Map<String, dynamic>);
    } catch (e) {
      return Failure(ErrorDesconocido('JSON inválido: $e'));
    }
  }
}
```

Crea `lib/data/repositories/productos_repository.dart`:

```dart
// lib/data/repositories/productos_repository.dart
import '../errors/api_error.dart';
import '../models/producto_dto.dart';
import '../network/http_client.dart';
import '../../domain/models/producto.dart';

class ProductosRepository {
  final HttpClient _client;
  const ProductosRepository(this._client);

  // Lista paginada con búsqueda opcional
  Future<Result<PaginaProductos>> obtenerProductos({
    int     pagina        = 1,
    int     tamanioPagina = 10,
    String? busqueda,
  }) async {
    final params = <String, String>{
      'page':      '$pagina',
      'page_size': '$tamanioPagina',
      if (busqueda != null && busqueda.isNotEmpty) 'search': busqueda,
    };
    final r = await _client.get('/products/', queryParams: params);
    return switch (r) {
      Success(:final data)  => Success(PaginaProductos.fromJson(data)),
      Failure(:final error) => Failure(error),
    };
  }

  // Detalle de un producto por ID
  Future<Result<Producto>> obtenerProducto(int id) async {
    final r = await _client.get('/products/$id/');
    return switch (r) {
      Success(:final data)  => Success(ProductoDto.fromJson(data).toDomain()),
      Failure(:final error) => Failure(error),
    };
  }
}
```

Crea `lib/providers/productos_providers.dart`:

```dart
// lib/providers/productos_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../data/errors/api_error.dart';
import '../data/models/producto_dto.dart';
import '../data/network/http_client.dart';
import '../data/repositories/productos_repository.dart';
import '../domain/models/producto.dart';

// Cliente HTTP — singleton compartido
final httpClientProvider = Provider<HttpClient>((ref) => const HttpClient(
  baseUrl: 'https://higuera-billing-api.desarrollo-software.xyz/api',
));

// Repositorio — depende del cliente
final productosRepositoryProvider = Provider<ProductosRepository>((ref) =>
  ProductosRepository(ref.watch(httpClientProvider)),
);

// Notifier que gestiona la lista de productos
class ProductosNotifier extends AsyncNotifier<List<Producto>> {
  @override
  Future<List<Producto>> build() => _cargar(pagina: 1);

  Future<List<Producto>> _cargar({int pagina = 1}) async {
    final repo      = ref.read(productosRepositoryProvider);
    final resultado = await repo.obtenerProductos(pagina: pagina);

    return switch (resultado) {
      Success(:final data)  =>
          data.results.map((d) => d.toDomain()).toList(),
      Failure(:final error) => throw Exception(error.mensaje),
    };
  }

  // Vuelve a cargar desde la primera página
  Future<void> recargar() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() => _cargar(pagina: 1));
  }
}

final productosProvider =
    AsyncNotifierProvider<ProductosNotifier, List<Producto>>(
  ProductosNotifier.new,
);
```

Crea `lib/screens/pantalla_productos.dart`:

```dart
// lib/screens/pantalla_productos.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/productos_providers.dart';
import '../domain/models/producto.dart';

class PantallaProductos extends ConsumerWidget {
  const PantallaProductos({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final productosAsync = ref.watch(productosProvider);
    final cs             = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Productos'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: () =>
                ref.read(productosProvider.notifier).recargar(),
          ),
        ],
      ),
      body: productosAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error:   (e, _) => Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Icon(Icons.cloud_off, size: 48, color: cs.error),
              const SizedBox(height: 8),
              Text('$e', style: TextStyle(color: cs.error)),
              const SizedBox(height: 12),
              FilledButton.icon(
                onPressed: () =>
                    ref.read(productosProvider.notifier).recargar(),
                icon:  const Icon(Icons.refresh),
                label: const Text('Reintentar'),
              ),
            ],
          ),
        ),
        data: (productos) => ListView.separated(
          itemCount:        productos.length,
          separatorBuilder: (_, __) => const Divider(height: 1),
          itemBuilder: (context, i) {
            final p = productos[i];
            return ListTile(
              leading: CircleAvatar(
                backgroundColor: p.activo
                    ? cs.primaryContainer
                    : cs.surfaceContainerHighest,
                child: Text(
                  '${p.stock}',
                  style: TextStyle(
                    fontSize:   11,
                    fontWeight: FontWeight.bold,
                    color: p.activo
                        ? cs.onPrimaryContainer
                        : cs.onSurfaceVariant,
                  ),
                ),
              ),
              title:    Text(p.nombre,
                  style: const TextStyle(fontWeight: FontWeight.w600)),
              subtitle: Text('${p.categoria} · \$${p.precio.toStringAsFixed(2)}'),
              trailing: Container(
                padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 3),
                decoration: BoxDecoration(
                  color:        p.colorStock.withOpacity(0.12),
                  borderRadius: BorderRadius.circular(12),
                  border:       Border.all(color: p.colorStock.withOpacity(0.3)),
                ),
                child: Text(
                  p.nivelStock,
                  style: TextStyle(
                    color:      p.colorStock,
                    fontSize:   11,
                    fontWeight: FontWeight.w600,
                  ),
                ),
              ),
            );
          },
        ),
      ),
    );
  }
}
```

### Agrega al `main.dart`

Añade el import de `pantalla_productos.dart` y cambia `paso = 4`.

### Prueba esto

- Ejecuta — la lista carga con `CircularProgressIndicator` y luego aparecen los productos
- Pulsa el ícono de recarga — vuelve a mostrar el loading y recarga
- Desactiva el WiFi y recarga — aparece el panel de error con botón "Reintentar"
- Reactiva el WiFi, pulsa "Reintentar" — vuelve a cargar correctamente
- Añade `print('Construyendo PantallaProductos')` en `build()` — ¿cuántas veces se llama al hacer recargar?

---

## 10. Paso 5 — Paginación infinita + búsqueda

**Objetivo:** cargar más productos al llegar al final + filtrar por nombre desde la UI.

Añade a `lib/providers/productos_providers.dart`:

```dart
// Estado con soporte para paginación incremental
class ProductosState {
  final List<Producto> productos;
  final bool           cargando;      // primera carga
  final bool           cargandoMas;  // carga siguiente página
  final bool           hayMas;
  final int            pagina;
  final String?        error;
  final String?        errorMas;

  const ProductosState({
    this.productos    = const [],
    this.cargando     = false,
    this.cargandoMas  = false,
    this.hayMas       = true,
    this.pagina       = 1,
    this.error,
    this.errorMas,
  });

  ProductosState copyWith({
    List<Producto>? productos,
    bool?           cargando,
    bool?           cargandoMas,
    bool?           hayMas,
    int?            pagina,
    String?         error,
    String?         errorMas,
  }) => ProductosState(
    productos:   productos   ?? this.productos,
    cargando:    cargando    ?? this.cargando,
    cargandoMas: cargandoMas ?? this.cargandoMas,
    hayMas:      hayMas      ?? this.hayMas,
    pagina:      pagina      ?? this.pagina,
    error:       error,     // null limpia el error anterior
    errorMas:    errorMas,
  );
}

class ProductosPaginadosNotifier extends Notifier<ProductosState> {
  static const int _tamanioPagina = 10;

  @override
  ProductosState build() {
    Future.microtask(cargar); // carga inicial sin bloquear build
    return const ProductosState(cargando: true);
  }

  // Carga desde la primera página — descarta los datos anteriores
  Future<void> cargar() async {
    state = const ProductosState(cargando: true);
    final repo = ref.read(productosRepositoryProvider);
    final r    = await repo.obtenerProductos(
        pagina: 1, tamanioPagina: _tamanioPagina);

    state = switch (r) {
      Success(:final data) => ProductosState(
          productos: data.results.map((d) => d.toDomain()).toList(),
          hayMas:    data.hayMas,
          pagina:    1,
        ),
      Failure(:final error) => ProductosState(error: error.mensaje),
    };
  }

  // Carga la siguiente página — AÑADE a los datos existentes
  Future<void> cargarMas() async {
    if (!state.hayMas || state.cargandoMas) return;
    state = state.copyWith(cargandoMas: true);

    final repo            = ref.read(productosRepositoryProvider);
    final siguientePagina = state.pagina + 1;
    final r = await repo.obtenerProductos(
        pagina: siguientePagina, tamanioPagina: _tamanioPagina);

    state = switch (r) {
      Success(:final data) => state.copyWith(
          // Operador spread — acumula páginas sin reemplazar
          productos:   [...state.productos,
                         ...data.results.map((d) => d.toDomain())],
          hayMas:      data.hayMas,
          pagina:      siguientePagina,
          cargandoMas: false,
        ),
      Failure(:final error) => state.copyWith(
          cargandoMas: false,
          errorMas:    error.mensaje,
        ),
    };
  }
}

final productosPaginadosProvider =
    NotifierProvider<ProductosPaginadosNotifier, ProductosState>(
  ProductosPaginadosNotifier.new,
);

// Búsqueda local sobre la lista ya cargada — sin llamar a la API
final busquedaProductoProvider = StateProvider<String>((ref) => '');

final productosFiltradosProvider = Provider<List<Producto>>((ref) {
  final todos    = ref.watch(productosPaginadosProvider).productos;
  final busqueda = ref.watch(busquedaProductoProvider);
  if (busqueda.isEmpty) return todos;
  final q = busqueda.toLowerCase();
  return todos
      .where((p) =>
          p.nombre.toLowerCase().contains(q) ||
          p.categoria.toLowerCase().contains(q))
      .toList();
});
```

Crea `lib/screens/pantalla_productos_paginados.dart`:

```dart
// lib/screens/pantalla_productos_paginados.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/productos_providers.dart';
import '../domain/models/producto.dart';

class PantallaProductosPaginados extends ConsumerStatefulWidget {
  const PantallaProductosPaginados({super.key});

  @override
  ConsumerState<PantallaProductosPaginados> createState() =>
      _PantallaProductosPaginadosState();
}

class _PantallaProductosPaginadosState
    extends ConsumerState<PantallaProductosPaginados> {
  final _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(_onScroll);
  }

  @override
  void dispose() {
    _scrollController.removeListener(_onScroll);
    _scrollController.dispose();
    super.dispose();
  }

  // Dispara cargarMas() cuando quedan menos de 200px para el final
  void _onScroll() {
    final pos = _scrollController.position;
    if (pos.pixels >= pos.maxScrollExtent - 200) {
      ref.read(productosPaginadosProvider.notifier).cargarMas();
    }
  }

  @override
  Widget build(BuildContext context) {
    final estado    = ref.watch(productosPaginadosProvider);
    final productos = ref.watch(productosFiltradosProvider);
    final cs        = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Productos — Paginados'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: () =>
                ref.read(productosPaginadosProvider.notifier).cargar(),
          ),
        ],
      ),
      body: Column(children: [
        // Barra de búsqueda local
        Padding(
          padding: const EdgeInsets.fromLTRB(12, 8, 12, 4),
          child: SearchBar(
            hintText: 'Buscar por nombre o categoría...',
            leading:  const Icon(Icons.search),
            onChanged: (v) =>
                ref.read(busquedaProductoProvider.notifier).state = v,
            padding: const WidgetStatePropertyAll(
              EdgeInsets.symmetric(horizontal: 16),
            ),
          ),
        ),

        // Cuerpo principal
        Expanded(
          child: estado.cargando
              ? const Center(child: CircularProgressIndicator())
              : estado.error != null && productos.isEmpty
                  ? _PantallaError(
                      mensaje:      estado.error!,
                      onReintentar: () =>
                          ref.read(productosPaginadosProvider.notifier).cargar(),
                    )
                  : Stack(children: [
                      RefreshIndicator(
                        onRefresh: () =>
                            ref.read(productosPaginadosProvider.notifier).cargar(),
                        child: ListView.builder(
                          controller: _scrollController,
                          padding:    const EdgeInsets.all(12),
                          // +1 para el pie de lista (loader / fin)
                          itemCount: productos.length + 1,
                          itemBuilder: (context, i) {
                            if (i == productos.length) {
                              return _PieDeLista(
                                cargandoMas: estado.cargandoMas,
                                hayMas:      estado.hayMas,
                              );
                            }
                            return _TarjetaProducto(producto: productos[i]);
                          },
                        ),
                      ),

                      // Banner de error no bloqueante — errores de "cargar más"
                      if (estado.errorMas != null)
                        Positioned(
                          bottom: 0,
                          left:   0,
                          right:  0,
                          child: _BannerError(mensaje: estado.errorMas!),
                        ),
                    ]),
        ),
      ]),
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
      child: ListTile(
        contentPadding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
        leading: CircleAvatar(
          backgroundColor: producto.activo
              ? cs.primaryContainer
              : cs.surfaceContainerHighest,
          child: Text(
            '${producto.stock}',
            style: TextStyle(
              fontSize:   11,
              fontWeight: FontWeight.bold,
              color: producto.activo
                  ? cs.onPrimaryContainer
                  : cs.onSurfaceVariant,
            ),
          ),
        ),
        title:    Text(producto.nombre,
            style: const TextStyle(fontWeight: FontWeight.w600)),
        subtitle: Text(
          '${producto.categoria} · \$${producto.precio.toStringAsFixed(2)}',
        ),
        trailing: Container(
          padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 3),
          decoration: BoxDecoration(
            color:        producto.colorStock.withOpacity(0.12),
            borderRadius: BorderRadius.circular(12),
            border:       Border.all(color: producto.colorStock.withOpacity(0.3)),
          ),
          child: Text(
            producto.nivelStock,
            style: TextStyle(
              color:      producto.colorStock,
              fontSize:   11,
              fontWeight: FontWeight.w600,
            ),
          ),
        ),
      ),
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

class _PantallaError extends StatelessWidget {
  final String       mensaje;
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
    final cs = Theme.of(context).colorScheme;
    return Material(
      elevation: 4,
      child: Container(
        width:   double.infinity,
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
        color:   cs.errorContainer,
        child: Row(children: [
          Icon(Icons.warning_amber, color: cs.error, size: 18),
          const SizedBox(width: 8),
          Expanded(
            child: Text(
              mensaje,
              style: TextStyle(color: cs.onErrorContainer, fontSize: 13),
            ),
          ),
        ]),
      ),
    );
  }
}
```

### Agrega al `main.dart`

Añade el import de `pantalla_productos_paginados.dart` y cambia `paso = 5`.

### Prueba esto

- Ejecuta y desplázate hasta el final de la lista — aparece el loader y se cargan más productos
- ¿Qué pasa cuando llegaste a la última página? (el scroll no intenta cargar más)
- Escribe un término en la búsqueda — filtra la lista en memoria sin llamar a la API
- Borra el texto — vuelven todos los productos cargados hasta ese momento
- Haz pull-to-refresh desde la parte superior — recarga desde la primera página

---

## 11. `main.dart` completo — referencia

```dart
// lib/main.dart
import 'dart:async';
import 'dart:convert';
import 'dart:io';

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:http/http.dart' as http;

import 'data/errors/api_error.dart';
import 'data/models/producto_dto.dart';
import 'domain/models/producto.dart';
import 'screens/pantalla_productos.dart';
import 'screens/pantalla_productos_paginados.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  http.get() crudo — ver JSON en pantalla                     │
// │  2  DTO + fromJson + mostrar lista real                          │
// │  3  ApiError + Result<T> — errores tipados                      │
// │  4  HttpClient + Repositorio + Riverpod                         │
// │  5  Paginación infinita + búsqueda                              │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 5;

void main() => runApp(
  const ProviderScope(child: AppBilling(paso: paso)),
);

class AppBilling extends StatelessWidget {
  final int paso;
  const AppBilling({super.key, required this.paso});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF1A237E)),
        useMaterial3: true,
      ),
      home: switch (paso) {
        1 => const _PasoCrudo(),
        2 => const _PasoDto(),
        3 => const _PasoErrores(),
        4 => const PantallaProductos(),
        5 => const PantallaProductosPaginados(),
        _ => Scaffold(body: Center(child: Text('Paso $paso no definido'))),
      },
    );
  }
}

// ─── Paso 1: http.get() crudo ────────────────────────────────────

class _PasoCrudo extends StatefulWidget {
  const _PasoCrudo();
  @override
  State<_PasoCrudo> createState() => _PasoCrudoState();
}

class _PasoCrudoState extends State<_PasoCrudo> {
  String _respuesta  = 'Pulsa el botón para hacer la petición';
  bool   _cargando   = false;
  int    _statusCode = 0;

  Future<void> _hacerPeticion() async {
    setState(() { _cargando = true; _respuesta = 'Cargando...'; });
    try {
      final uri = Uri.parse(
        'https://higuera-billing-api.desarrollo-software.xyz/api/products/'
        '?page=1&page_size=3',
      );
      final response = await http.get(uri).timeout(const Duration(seconds: 15));
      final json     = jsonDecode(response.body);
      setState(() {
        _statusCode = response.statusCode;
        _respuesta  = const JsonEncoder.withIndent('  ').convert(json);
      });
    } catch (e) {
      setState(() { _statusCode = 0; _respuesta = 'Error: $e'; });
    } finally {
      setState(() => _cargando = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    return Scaffold(
      appBar: AppBar(
        title:           const Text('Paso 1 — http básico'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: Column(children: [
        if (_statusCode > 0)
          Container(
            width:   double.infinity,
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            color: _statusCode == 200 ? Colors.green.shade50 : Colors.red.shade50,
            child: Text(
              'HTTP $_statusCode — ${_statusCode == 200 ? 'OK' : 'Error'}',
              style: TextStyle(
                fontWeight: FontWeight.bold,
                color: _statusCode == 200
                    ? Colors.green.shade800
                    : Colors.red.shade800,
              ),
            ),
          ),
        Expanded(
          child: SingleChildScrollView(
            padding: const EdgeInsets.all(16),
            child: SelectableText(
              _respuesta,
              style: const TextStyle(fontFamily: 'monospace', fontSize: 12),
            ),
          ),
        ),
        Padding(
          padding: const EdgeInsets.all(16),
          child: SizedBox(
            width: double.infinity,
            child: FilledButton.icon(
              onPressed: _cargando ? null : _hacerPeticion,
              icon: _cargando
                  ? const SizedBox(
                      width: 18, height: 18,
                      child: CircularProgressIndicator(strokeWidth: 2),
                    )
                  : const Icon(Icons.cloud_download),
              label: Text(_cargando ? 'Cargando...' : 'Obtener productos (página 1)'),
            ),
          ),
        ),
      ]),
    );
  }
}

// ─── Paso 2: DTO + fromJson ───────────────────────────────────────

class _PasoDto extends StatefulWidget {
  const _PasoDto();
  @override
  State<_PasoDto> createState() => _PasoDtoState();
}

class _PasoDtoState extends State<_PasoDto> {
  List<Producto> _productos = [];
  bool   _cargando = false;
  String _error    = '';

  @override
  void initState() {
    super.initState();
    _cargar();
  }

  Future<void> _cargar() async {
    setState(() { _cargando = true; _error = ''; });
    try {
      final uri = Uri.parse(
        'https://higuera-billing-api.desarrollo-software.xyz/api/products/'
        '?page=1&page_size=10',
      );
      final response = await http.get(uri).timeout(const Duration(seconds: 15));
      if (response.statusCode == 200) {
        final json   = jsonDecode(response.body) as Map<String, dynamic>;
        final pagina = PaginaProductos.fromJson(json);
        setState(() {
          _productos = pagina.results.map((d) => d.toDomain()).toList();
        });
      } else {
        setState(() => _error = 'Error HTTP ${response.statusCode}');
      }
    } catch (e) {
      setState(() => _error = 'Error de red: $e');
    } finally {
      setState(() => _cargando = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    return Scaffold(
      appBar: AppBar(
        title:           const Text('Paso 2 — DTO + fromJson'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: _cargando ? null : _cargar,
          ),
        ],
      ),
      body: _cargando
          ? const Center(child: CircularProgressIndicator())
          : _error.isNotEmpty
              ? Center(child: Text(_error, style: TextStyle(color: cs.error)))
              : ListView.separated(
                  itemCount:        _productos.length,
                  separatorBuilder: (_, __) => const Divider(height: 1),
                  itemBuilder: (context, i) {
                    final p = _productos[i];
                    return ListTile(
                      leading: CircleAvatar(
                        backgroundColor: p.disponible
                            ? Colors.green.shade50
                            : Colors.grey.shade100,
                        child: Text(
                          '${p.stock}',
                          style: TextStyle(
                            fontSize:   11,
                            fontWeight: FontWeight.bold,
                            color: p.disponible ? Colors.green : Colors.grey,
                          ),
                        ),
                      ),
                      title:    Text(p.nombre,
                          style: const TextStyle(fontWeight: FontWeight.w600)),
                      subtitle: Text(
                          '${p.categoria} — \$${p.precio.toStringAsFixed(2)}'),
                      trailing: Text(
                        p.nivelStock,
                        style: TextStyle(
                          color:      p.colorStock,
                          fontWeight: FontWeight.w600,
                          fontSize:   12,
                        ),
                      ),
                    );
                  },
                ),
    );
  }
}

// ─── Paso 3: ApiError + Result<T> ────────────────────────────────

class _PasoErrores extends StatefulWidget {
  const _PasoErrores();
  @override
  State<_PasoErrores> createState() => _PasoErroresState();
}

class _PasoErroresState extends State<_PasoErrores> {
  List<Producto> _productos = [];
  bool   _cargando = false;
  String _errorMsg = '';

  @override
  void initState() {
    super.initState();
    _cargar();
  }

  Future<Result<PaginaProductos>> _fetchProductos() async {
    try {
      final uri = Uri.parse(
        'https://higuera-billing-api.desarrollo-software.xyz/api/products/'
        '?page=1&page_size=10',
      );
      final response = await http.get(uri).timeout(const Duration(seconds: 15));
      return switch (response.statusCode) {
        200 => () {
          final json   = jsonDecode(response.body) as Map<String, dynamic>;
          final pagina = PaginaProductos.fromJson(json);
          return Success(pagina);
        }(),
        401 => const Failure(ErrorNoAutorizado()),
        404 => const Failure(ErrorNotFound()),
        _   => Failure(ErrorServidor(response.statusCode, response.body)),
      };
    } on SocketException catch (e) {
      return Failure(ErrorRed(e.message));
    } on TimeoutException {
      return Failure(const ErrorRed('Tiempo agotado'));
    } catch (e) {
      return Failure(ErrorDesconocido(e));
    }
  }

  Future<void> _cargar() async {
    setState(() { _cargando = true; _errorMsg = ''; });
    final resultado = await _fetchProductos();
    switch (resultado) {
      case Success(:final data):
        setState(() =>
            _productos = data.results.map((d) => d.toDomain()).toList());
      case Failure(:final error):
        setState(() => _errorMsg = error.mensaje);
    }
    setState(() => _cargando = false);
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    return Scaffold(
      appBar: AppBar(
        title:           const Text('Paso 3 — ApiError + Result<T>'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: _cargando ? null : _cargar,
          ),
        ],
      ),
      body: _cargando
          ? const Center(child: CircularProgressIndicator())
          : _errorMsg.isNotEmpty
              ? Center(
                  child: Column(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      Icon(Icons.cloud_off, size: 48, color: cs.error),
                      const SizedBox(height: 8),
                      Text(_errorMsg, style: TextStyle(color: cs.error)),
                      const SizedBox(height: 12),
                      FilledButton.icon(
                        onPressed: _cargar,
                        icon:  const Icon(Icons.refresh),
                        label: const Text('Reintentar'),
                      ),
                    ],
                  ),
                )
              : ListView.separated(
                  itemCount:        _productos.length,
                  separatorBuilder: (_, __) => const Divider(height: 1),
                  itemBuilder: (context, i) {
                    final p = _productos[i];
                    return ListTile(
                      title:    Text(p.nombre,
                          style: const TextStyle(fontWeight: FontWeight.w600)),
                      subtitle: Text(p.categoria),
                      trailing: Text(
                        '\$${p.precio.toStringAsFixed(2)}',
                        style: TextStyle(
                            color: cs.primary, fontWeight: FontWeight.bold),
                      ),
                    );
                  },
                ),
    );
  }
}
```

---

## 12. Proyecto — Catálogo de Productos con paginación

App completa autocontenida (sin selector de pasos) que integra
`ProductosPaginadosNotifier`, `SearchBar`, paginación infinita,
`RefreshIndicator`, `pantalla_detalle.dart` con `FutureProvider.family`
y banner de error no bloqueante:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() => runApp(const ProviderScope(child: AppCatalogo()));

class AppCatalogo extends StatelessWidget {
  const AppCatalogo({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title:        'Catálogo Billing',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const PantallaCatalogo(),
    );
  }
}

// ─── Pantalla principal ───────────────────────────────────────────

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
    _scrollController.addListener(_onScroll);
  }

  @override
  void dispose() {
    _scrollController.removeListener(_onScroll);
    _scrollController.dispose();
    super.dispose();
  }

  // Carga la siguiente página cuando queda poco scroll
  void _onScroll() {
    final pos = _scrollController.position;
    if (pos.pixels >= pos.maxScrollExtent - 200) {
      ref.read(productosPaginadosProvider.notifier).cargarMas();
    }
  }

  @override
  Widget build(BuildContext context) {
    final estado    = ref.watch(productosPaginadosProvider);
    final productos = ref.watch(productosFiltradosProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Catálogo de productos'),
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: () =>
                ref.read(productosPaginadosProvider.notifier).cargar(),
          ),
        ],
      ),
      body: Column(children: [
        // Buscador — filtra en memoria, sin llamar a la API
        Padding(
          padding: const EdgeInsets.fromLTRB(12, 8, 12, 4),
          child: SearchBar(
            hintText: 'Buscar producto...',
            leading:  const Icon(Icons.search),
            onChanged: (v) =>
                ref.read(busquedaProductoProvider.notifier).state = v,
            padding: const WidgetStatePropertyAll(
              EdgeInsets.symmetric(horizontal: 16),
            ),
          ),
        ),

        Expanded(
          child: estado.cargando
              ? const Center(child: CircularProgressIndicator())
              : estado.error != null && productos.isEmpty
                  ? _PantallaError(
                      mensaje: estado.error!,
                      onReintentar: () =>
                          ref.read(productosPaginadosProvider.notifier).cargar(),
                    )
                  : Stack(children: [
                      RefreshIndicator(
                        onRefresh: () =>
                            ref.read(productosPaginadosProvider.notifier).cargar(),
                        child: ListView.builder(
                          controller: _scrollController,
                          padding:    const EdgeInsets.all(12),
                          itemCount:  productos.length + 1,
                          itemBuilder: (context, i) {
                            // Último elemento — loader o mensaje de fin
                            if (i == productos.length) {
                              return _PieDeLista(
                                cargandoMas: estado.cargandoMas,
                                hayMas:      estado.hayMas,
                              );
                            }
                            return _TarjetaProducto(
                              producto:  productos[i],
                              onTap: () => Navigator.push(
                                context,
                                MaterialPageRoute(
                                  builder: (_) => PantallaDetalle(
                                      idProducto: productos[i].id),
                                ),
                              ),
                            );
                          },
                        ),
                      ),

                      // Error no bloqueante para errores de "cargar más"
                      if (estado.errorMas != null)
                        Positioned(
                          bottom: 0, left: 0, right: 0,
                          child: _BannerError(mensaje: estado.errorMas!),
                        ),
                    ]),
        ),
      ]),
    );
  }
}

// ─── Pantalla de detalle ──────────────────────────────────────────

// FutureProvider.family — carga el detalle de un producto por ID
final productoDetalleProvider =
    FutureProvider.family<Producto, int>((ref, id) async {
  final repo      = ref.read(productosRepositoryProvider);
  final resultado = await repo.obtenerProducto(id);
  return switch (resultado) {
    Success(:final data)  => data,
    Failure(:final error) => throw Exception(error.mensaje),
  };
});

class PantallaDetalle extends ConsumerWidget {
  final int idProducto;
  const PantallaDetalle({super.key, required this.idProducto});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final productoAsync = ref.watch(productoDetalleProvider(idProducto));
    final cs            = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Detalle del producto'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: productoAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error:   (e, _) => Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Icon(Icons.error_outline, size: 48, color: cs.error),
              const SizedBox(height: 8),
              Text('$e', style: TextStyle(color: cs.error)),
              const SizedBox(height: 12),
              FilledButton.icon(
                onPressed: () =>
                    ref.invalidate(productoDetalleProvider(idProducto)),
                icon:  const Icon(Icons.refresh),
                label: const Text('Reintentar'),
              ),
            ],
          ),
        ),
        data: (p) => SingleChildScrollView(
          padding: const EdgeInsets.all(24),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              // Imagen o placeholder
              ClipRRect(
                borderRadius: BorderRadius.circular(12),
                child: p.imagenUrl != null
                    ? Image.network(
                        p.imagenUrl!,
                        width:  double.infinity,
                        height: 200,
                        fit:    BoxFit.cover,
                        errorBuilder: (_, __, ___) =>
                            _PlaceholderImagen(height: 200),
                      )
                    : _PlaceholderImagen(height: 200),
              ),
              const SizedBox(height: 20),

              // Nombre
              Text(p.nombre,
                  style: Theme.of(context).textTheme.headlineSmall?.copyWith(
                        fontWeight: FontWeight.bold,
                      )),
              const SizedBox(height: 4),

              // Categoría
              Text(p.categoria,
                  style: TextStyle(color: cs.onSurfaceVariant, fontSize: 14)),
              const SizedBox(height: 16),

              // Precio
              Text(
                '\$${p.precio.toStringAsFixed(2)}',
                style: TextStyle(
                  fontSize:   28,
                  fontWeight: FontWeight.bold,
                  color:      cs.primary,
                ),
              ),
              const SizedBox(height: 16),

              // Nivel de stock con color
              Row(children: [
                Container(
                  padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
                  decoration: BoxDecoration(
                    color:        p.colorStock.withOpacity(0.12),
                    borderRadius: BorderRadius.circular(20),
                    border:       Border.all(color: p.colorStock.withOpacity(0.4)),
                  ),
                  child: Text(
                    '${p.nivelStock} · ${p.stock} unidades',
                    style: TextStyle(
                      color:      p.colorStock,
                      fontWeight: FontWeight.w600,
                    ),
                  ),
                ),
                const SizedBox(width: 8),
                if (!p.activo)
                  Container(
                    padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 6),
                    decoration: BoxDecoration(
                      color:        cs.errorContainer,
                      borderRadius: BorderRadius.circular(20),
                    ),
                    child: Text(
                      'Inactivo',
                      style: TextStyle(
                        color:      cs.onErrorContainer,
                        fontWeight: FontWeight.w600,
                        fontSize:   12,
                      ),
                    ),
                  ),
              ]),
            ],
          ),
        ),
      ),
    );
  }
}

// ── Widgets auxiliares ────────────────────────────────────────────

class _TarjetaProducto extends StatelessWidget {
  final Producto   producto;
  final VoidCallback? onTap;
  const _TarjetaProducto({required this.producto, this.onTap});

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Card(
      margin: const EdgeInsets.only(bottom: 10),
      child: InkWell(
        onTap:        onTap,
        borderRadius: BorderRadius.circular(12),
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
                            _PlaceholderImagen(height: 72, width: 72),
                      )
                    : _PlaceholderImagen(height: 72, width: 72),
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
                                fontSize: 10, color: cs.onErrorContainer),
                          ),
                        ),
                    ]),
                    Text(
                      producto.categoria,
                      style: TextStyle(
                          fontSize: 12, color: cs.onSurfaceVariant),
                    ),
                    const SizedBox(height: 8),
                    Row(children: [
                      Text(
                        '\$${producto.precio.toStringAsFixed(2)}',
                        style: TextStyle(
                          fontSize:   17,
                          fontWeight: FontWeight.bold,
                          color:      cs.primary,
                        ),
                      ),
                      const Spacer(),
                      _BadgeStock(
                        nivel: producto.nivelStock,
                        color: producto.colorStock,
                        stock: producto.stock,
                      ),
                    ]),
                  ],
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

class _PlaceholderImagen extends StatelessWidget {
  final double  height;
  final double? width;
  const _PlaceholderImagen({required this.height, this.width});

  @override
  Widget build(BuildContext context) => Container(
    width:  width ?? double.infinity,
    height: height,
    color:  Colors.grey.shade100,
    child:  Icon(Icons.image_outlined,
        size: height * 0.5, color: Colors.grey.shade400),
  );
}

class _BadgeStock extends StatelessWidget {
  final String nivel;
  final Color  color;
  final int    stock;
  const _BadgeStock({
    required this.nivel,
    required this.color,
    required this.stock,
  });

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        Icon(Icons.inventory_2_outlined, size: 14, color: color),
        const SizedBox(width: 4),
        Text(
          '$stock · $nivel',
          style: TextStyle(
            fontSize:   12,
            color:      color,
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
                        color: Colors.grey.shade500, fontSize: 13),
                  ),
      ),
    );
  }
}

class _PantallaError extends StatelessWidget {
  final String       mensaje;
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
    final cs = Theme.of(context).colorScheme;
    return Material(
      elevation: 4,
      child: Container(
        width:   double.infinity,
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
        color:   cs.errorContainer,
        child: Row(children: [
          Icon(Icons.warning_amber, color: cs.error, size: 18),
          const SizedBox(width: 8),
          Expanded(
            child: Text(
              mensaje,
              style: TextStyle(
                  color: cs.onErrorContainer, fontSize: 13),
            ),
          ),
        ]),
      ),
    );
  }
}
```

---

## 13. Guía rápida de imports

```dart
import 'dart:convert';                          // jsonDecode, jsonEncode
import 'dart:io';                               // SocketException
import 'dart:async';                            // TimeoutException
import 'package:http/http.dart' as http;        // http.get, http.post
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Hacer GET:
final uri      = Uri.parse('https://api.ejemplo.com/ruta');
final response = await http.get(uri);
final json     = jsonDecode(response.body) as Map<String, dynamic>;

// GET con query params:
final uri = Uri.parse('https://api.ejemplo.com/ruta')
    .replace(queryParameters: {'page': '1', 'page_size': '10'});

// POST con cuerpo JSON:
final response = await http.post(
  Uri.parse('https://api.ejemplo.com/ruta'),
  headers: {'Content-Type': 'application/json'},
  body:    jsonEncode({'campo': 'valor'}),
);
```

---

## 14. Cuándo usar `http` vs `dio`

```
Aspecto              http (este módulo)           dio (alternativa popular)
───────────────────  ──────────────────────────   ───────────────────────────
Setup                pubspec + import             pubspec + import
Interceptores        Manual (en HttpClient)       Automáticos (clase Interceptor)
Timeouts             .timeout(Duration)           options.connectTimeout
Subida de archivos   FormData manual              FormData integrado
Cancelación          No (nativo)                  CancelToken
Tamaño de pkg        Pequeño                      Mayor
Ideal para           Apps simples / aprendizaje   Apps complejas / producción
```

`http` es el paquete oficial de Dart y suficiente para la mayoría de proyectos.
`dio` añade una capa de abstracción más potente pero con mayor complejidad de
configuración. El `HttpClient` que construimos en este módulo puede reemplazarse
internamente por `dio` sin cambiar el contrato del repositorio.

---

## 15. Ejercicios propuestos

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

## 16. Resumen de la página 12

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
