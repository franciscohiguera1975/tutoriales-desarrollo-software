# Tutorial Flutter — Página 19
## Módulo 7 · Arquitectura
### Clean Architecture con Riverpod — proyecto integrador

---

## ¿Qué aprenderás?

```
┌────────────────────────────────┬────────────────────────────────────────┐
│  Sin arquitectura (pág. 11-17) │  Clean Architecture (esta página)      │
├────────────────────────────────┼────────────────────────────────────────┤
│ Todo en el Notifier            │ Notifier solo llama use cases          │
│ DTO mezclado con la entidad    │ DTO ≠ Entidad — separación estricta    │
│ HTTP directo en el notifier    │ HTTP solo en la capa Data              │
│ Sin interfaces de repositorio  │ Interfaz en Domain, impl. en Data      │
│ Sin use cases                  │ Cada operación = una clase             │
│ Difícil de testear (acoplado)  │ Cada capa se testea por separado       │
│ Cambiar API rompe la UI        │ Cambiar API no toca Domain ni UI       │
│ Lógica de negocio dispersa     │ Lógica de negocio en entidades puras   │
└────────────────────────────────┴────────────────────────────────────────┘
```

**La regla central:** las dependencias siempre apuntan hacia adentro.

```
┌──────────────────────────────────────────────┐
│  UI  (pantallas, notifiers, providers)       │
│       conoce Domain — no conoce Data         │
├──────────────────────────────────────────────┤
│  Domain  (entidades, interfaces, use cases)  │
│       SIN imports de Flutter, SIN HTTP       │
├──────────────────────────────────────────────┤
│  Data  (DTOs, repositorios, HTTP)            │
│       conoce Domain — implementa contratos   │
└──────────────────────────────────────────────┘
   Data conoce Domain. UI conoce Domain. Nadie conoce UI.
```

---

## Patrones clave

Los siguientes 6 patrones son los ladrillos de Clean Architecture.
Cada bloque sigue el ritmo: **concepto puro → ejemplo aplicado → mini-ejercicio**.

---

### Patrón 1 — Entidad de dominio (Dart puro)

**Concepto:** la entidad representa la lógica de negocio central de tu app.
Es Dart puro — sin imports de Flutter, sin `fromJson`, sin nada de HTTP.
Si puedes copiarla a un proyecto de consola y compilar, está bien.

**Ejemplo aplicado:**

```dart
// lib/domain/entities/producto.dart
// Entidad = solo lógica de negocio, sin imports de Flutter ni http

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

  // Lógica de negocio pura — getters que encapsulan reglas
  bool get disponible  => activo && stock > 0;
  bool get stockBajo   => stock > 0 && stock <= 5;

  String get nivelStock => switch (stock) {
    0    => 'Agotado',
    <= 5  => 'Bajo',
    <= 20 => 'Normal',
    _    => 'Alto',
  };

  @override
  String toString() => 'Producto($id: $nombre @ \$$precio)';

  @override
  bool operator ==(Object other) => other is Producto && other.id == id;

  @override
  int get hashCode => id.hashCode;
}

// Prueba rápida en main():
void main() {
  const p = Producto(
    id: 1, nombre: 'Teclado', precio: 89.99,
    categoria: 'Periféricos', activo: true, stock: 3,
  );
  print(p.disponible);   // true
  print(p.nivelStock);   // Bajo
  print(p.stockBajo);    // true
}
```

> **Mini-ejercicio ⏱ 5 min**
>
> Agrega a `Producto` el getter `precioConIva` (precio * 1.19) y el getter
> `etiquetaPrecio` que devuelva `"\$precio (IVA inc. \$precioConIva)"`.
> Pruébalos en un `main()` simple imprimiendo la etiqueta de tres productos distintos.
>
> ```dart
> // Resultado esperado:
> // $89.99 (IVA inc. $107.09)
> // $25.50 (IVA inc. $30.35)
> // $5.00  (IVA inc. $5.95)
> ```

---

### Patrón 2 — Interfaz del repositorio (contrato)

**Concepto:** en Domain defines QUÉ puede hacer el repositorio, nunca el CÓMO.
La UI conoce esta interfaz. La capa Data proporciona la implementación.
Cambiar de REST a GraphQL no modifica ni un carácter en Domain ni en UI.

**Ejemplo aplicado:**

```dart
// lib/domain/repositories/i_productos_repository.dart
// En domain/ — solo define QUÉ, nunca el CÓMO

import '../entities/producto.dart';

abstract interface class IProductosRepository {
  /// Obtener una página de productos.
  /// Devuelve un record con la lista y un flag de paginación.
  Future<({List<Producto> items, bool hayMas})> obtenerProductos({
    int    pagina        = 1,
    int    tamanioPagina = 10,
    String busqueda      = '',
  });

  /// Obtener un producto por ID. Devuelve null si no existe.
  Future<Producto?> obtenerPorId(int id);
}
```

Por qué `abstract interface class`: en Dart 3, `interface` garantiza que
cualquier clase que la implemente debe definir TODOS los métodos.

> **Mini-ejercicio ⏱ 5 min**
>
> Define `IFavoritosRepository` con tres métodos:
> - `Future<void> agregar(int id)`
> - `Future<void> eliminar(int id)`
> - `Future<List<int>> obtenerIds()`
>
> Luego escribe una clase `FavoritosRepositoryFake` que implemente la
> interfaz usando un `List<int>` en memoria. Pruébala en `main()`.

---

### Patrón 3 — DTO: JSON → Entidad

**Concepto:** el DTO (Data Transfer Object) vive en la capa Data y hace
el trabajo sucio de parsear JSON. La entidad de dominio nunca ve JSON.
Esto separa los cambios de la API (nombres de campos, tipos) del modelo de negocio.

**Ejemplo aplicado:**

```dart
// lib/data/dtos/producto_dto.dart
// En data/ — maneja JSON, convierte a entidad de dominio

import '../../domain/entities/producto.dart';

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

  factory ProductoDto.fromJson(Map<String, dynamic> json) => ProductoDto(
    id:           json['id']            as int,
    name:         json['name']          as String,
    price:        json['price']         as String,
    categoryName: json['category_name'] as String?,
    isActive:     json['is_active']     as bool? ?? false,
    stock:        json['stock']         as int?  ?? 0,
    urlImage:     json['url_image']     as String?,
  );

  // La magia ocurre aquí: snake_case → camelCase, String → double
  Producto toDomain() => Producto(
    id:        id,
    nombre:    name,
    precio:    double.tryParse(price) ?? 0.0,
    categoria: categoryName ?? 'Sin categoría',
    activo:    isActive,
    stock:     stock,
    imagenUrl: urlImage,
  );
}
```

> **Mini-ejercicio ⏱ 5 min**
>
> Dado el JSON:
> ```dart
> const json = {
>   'id': 5, 'name': 'Mouse', 'price': '25.50',
>   'is_active': true, 'stock': 15,
> };
> ```
> Crea el DTO con `ProductoDto.fromJson(json)`, conviértelo a entidad
> con `toDomain()` e imprime `nivelStock` y `disponible`.
>
> ```
> // Resultado esperado:
> // nivelStock: Normal
> // disponible: true
> ```

---

### Patrón 4 — Implementación del repositorio (con http)

**Concepto:** la implementación vive en Data y conoce todos los detalles
sucios: URLs, headers, códigos de estado, parseo de JSON.
La UI **nunca** importa esta clase — solo importa la interfaz.

**Ejemplo aplicado:**

```dart
// lib/data/repositories/productos_repository_impl.dart
// En data/ — implementa el contrato de domain/

import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

import '../../domain/entities/producto.dart';
import '../../domain/repositories/i_productos_repository.dart';
import '../dtos/producto_dto.dart';

class ProductosRepositoryImpl implements IProductosRepository {
  static const _baseUrl =
      'https://higuera-billing-api.desarrollo-software.xyz/api';

  final http.Client _client;
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

    final response = await _client
        .get(uri, headers: {'Accept': 'application/json'})
        .timeout(const Duration(seconds: 15));

    if (response.statusCode != 200) {
      throw Exception('HTTP ${response.statusCode}');
    }

    final json    = jsonDecode(response.body) as Map<String, dynamic>;
    final results = (json['results'] as List)
        .map((e) => ProductoDto.fromJson(e).toDomain())
        .toList();

    return (items: results, hayMas: json['next'] != null);
  }

  @override
  Future<Producto?> obtenerPorId(int id) async {
    final uri = Uri.parse('$_baseUrl/products/$id/');
    final response = await _client
        .get(uri, headers: {'Accept': 'application/json'})
        .timeout(const Duration(seconds: 15));

    if (response.statusCode == 404) return null;
    if (response.statusCode != 200) {
      throw Exception('HTTP ${response.statusCode}');
    }
    final json = jsonDecode(response.body) as Map<String, dynamic>;
    return ProductoDto.fromJson(json).toDomain();
  }
}
```

> **Mini-ejercicio ⏱ 5 min**
>
> Agrega manejo de errores de red a `obtenerProductos`:
> - Si `SocketException` → lanza `Exception('Sin conexión a internet')`
> - Si timeout → lanza `Exception('La solicitud tardó demasiado')`
> - En cualquier otro error → relanza con `rethrow`
>
> Tip: envuelve el bloque en `try/catch` con tres `on` distintos.

---

### Patrón 5 — Use Case (una operación = una clase)

**Concepto:** cada use case encapsula una sola operación de negocio.
El truco de `operator call()` permite invocarlos como si fueran funciones:
`uc(pagina: 2)` en lugar de `uc.ejecutar(pagina: 2)`.

**Ejemplo aplicado:**

```dart
// lib/domain/usecases/obtener_productos_uc.dart
// En domain/ — encapsula UNA regla de negocio

import '../entities/producto.dart';
import '../repositories/i_productos_repository.dart';

class ObtenerProductosUc {
  final IProductosRepository _repo;
  const ObtenerProductosUc(this._repo);

  // operator call() → se invoca como función: uc(pagina: 2)
  Future<({List<Producto> items, bool hayMas})> call({
    int    pagina        = 1,
    int    tamanioPagina = 10,
    String busqueda      = '',
  }) => _repo.obtenerProductos(
    pagina:        pagina,
    tamanioPagina: tamanioPagina,
    busqueda:      busqueda,
  );
}

// lib/domain/usecases/buscar_productos_uc.dart

class BuscarProductosUc {
  final IProductosRepository _repo;
  const BuscarProductosUc(this._repo);

  Future<List<Producto>> call(String query) async {
    if (query.trim().length < 2) return [];   // regla de negocio
    final res = await _repo.obtenerProductos(
      busqueda:      query.trim(),
      tamanioPagina: 20,
    );
    return res.items;
  }
}
```

> **Mini-ejercicio ⏱ 5 min**
>
> Crea `FiltrarPorCategoriaUc` que reciba una categoría como `String` y
> devuelva `Future<List<Producto>>`.
>
> Discusión: ¿deberías llamar al repo con un parámetro `categoria` o
> traer todos los productos y filtrar en memoria?
> ¿Cuándo conviene cada enfoque?
>
> ```dart
> // Pista: si la API soporta filtro por categoría, úsala:
> // _repo.obtenerProductos(busqueda: categoria)
> // Si no, trae todos y filtra:
> // final todos = await _repo.obtenerProductos(tamanioPagina: 100);
> // return todos.items.where((p) => p.categoria == categoria).toList();
> ```

---

### Patrón 6 — Providers Riverpod: inyección de dependencias

**Concepto:** los providers de Riverpod forman una cadena de dependencias.
Cada provider declara explícitamente de qué depende. Riverpod los instancia
en el orden correcto. Para testear, solo se hace override del provider base.

**Ejemplo aplicado:**

```dart
// lib/ui/providers/productos_providers.dart
// En ui/ — conecta infraestructura con dominio

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:http/http.dart' as http;

import '../../data/repositories/productos_repository_impl.dart';
import '../../domain/repositories/i_productos_repository.dart';
import '../../domain/usecases/obtener_productos_uc.dart';
import '../../domain/usecases/buscar_productos_uc.dart';

// Infraestructura — base de la cadena
final httpClientProvider = Provider<http.Client>((ref) {
  final client = http.Client();
  ref.onDispose(client.close);   // cerrar al destruir el provider
  return client;
});

// Repositorio expuesto como INTERFAZ (no implementación)
// Para cambiar Data (ej: SQLite), solo cambia este provider
final productosRepositoryProvider = Provider<IProductosRepository>((ref) =>
  ProductosRepositoryImpl(ref.watch(httpClientProvider)));

// Use cases — reciben el repo automáticamente
final obtenerProductosUcProvider = Provider<ObtenerProductosUc>((ref) =>
  ObtenerProductosUc(ref.watch(productosRepositoryProvider)));

final buscarProductosUcProvider = Provider<BuscarProductosUc>((ref) =>
  BuscarProductosUc(ref.watch(productosRepositoryProvider)));

// Notifier — solo llama use cases, sin HTTP ni JSON
final catalogoProvider = NotifierProvider<CatalogoNotifier, CatalogoState>(
  CatalogoNotifier.new);
```

La cadena completa de dependencias:

```
httpClientProvider
    └── productosRepositoryProvider (IProductosRepository)
            ├── obtenerProductosUcProvider
            ├── buscarProductosUcProvider
            └── catalogoProvider (CatalogoNotifier)
```

> **Mini-ejercicio ⏱ 5 min**
>
> Agrega `productoPorIdProvider` como `FutureProvider.family<Producto?, int>`.
> ¿Qué use case necesitas crear primero?
>
> ```dart
> // Pista:
> final productoPorIdProvider =
>     FutureProvider.family<Producto?, int>((ref, id) async {
>   final repo = ref.watch(productosRepositoryProvider);
>   return repo.obtenerPorId(id);
>   // O si tienes ObtenerProductoPorIdUc:
>   // final uc = ref.watch(obtenerProductoPorIdUcProvider);
>   // return uc(id);
> });
> ```

---

## Crea el proyecto

```bash
flutter create productos_clean
cd productos_clean
```

Reemplaza el contenido de `pubspec.yaml`:

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
  flutter_riverpod: ^3.3.0
  http:             ^1.2.2

dev_dependencies:
  flutter_test:
    sdk: flutter
  mockito:      ^5.4.4
  build_runner:
  flutter_lints: ^5.0.0

flutter:
  uses-material-design: true
```

```bash
flutter pub get
```

Crea la estructura de carpetas:

```bash
mkdir -p lib/domain/entities
mkdir -p lib/domain/repositories
mkdir -p lib/domain/usecases
mkdir -p lib/data/dtos
mkdir -p lib/data/repositories
mkdir -p lib/ui/providers
mkdir -p lib/ui/notifiers
mkdir -p lib/ui/screens
mkdir -p test/domain
mkdir -p test/data
```

---

## Selector de pasos

Abre `lib/main.dart` y cambia este valor para avanzar paso a paso:

```dart
// Cambia entre 1 y 5 para activar cada paso
const int paso = 1;
```

Cada paso agrega una capa nueva. En el Paso 5 la app está completa.

---

## Paso 1 — App mínima + Entidad de dominio

Demuestra que la entidad funciona sin red, sin Riverpod, solo Dart puro.

**`lib/main.dart` — Paso 1:**

```dart
import 'package:flutter/material.dart';

// Entidad de dominio — Dart puro
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

  bool get disponible  => activo && stock > 0;
  bool get stockBajo   => stock > 0 && stock <= 5;
  double get precioConIva => precio * 1.19;

  String get nivelStock => switch (stock) {
    0    => 'Agotado',
    <= 5  => 'Bajo',
    <= 20 => 'Normal',
    _    => 'Alto',
  };
}

// Datos hardcodeados — sin HTTP
final _productos = [
  const Producto(
    id: 1, nombre: 'Teclado Mecánico', precio: 89.99,
    categoria: 'Periféricos', activo: true, stock: 3,
  ),
  const Producto(
    id: 2, nombre: 'Monitor 27"', precio: 349.99,
    categoria: 'Monitores', activo: true, stock: 25,
  ),
  const Producto(
    id: 3, nombre: 'Webcam HD', precio: 55.00,
    categoria: 'Accesorios', activo: false, stock: 0,
  ),
];

void main() => runApp(const App());

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Paso 1 — Entidad',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: Scaffold(
        appBar: AppBar(
          title: const Text('Paso 1 — Entidad de dominio'),
          backgroundColor: Colors.teal.shade100,
        ),
        body: ListView.builder(
          padding: const EdgeInsets.all(12),
          itemCount: _productos.length,
          itemBuilder: (_, i) {
            final p = _productos[i];
            return Card(
              margin: const EdgeInsets.only(bottom: 10),
              child: ListTile(
                title:    Text(p.nombre),
                subtitle: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(p.categoria),
                    Text('Stock: ${p.stock} — ${p.nivelStock}'),
                    Text('Precio: \$${p.precio} '
                         '(IVA: \$${p.precioConIva.toStringAsFixed(2)})'),
                  ],
                ),
                trailing: Chip(
                  label: Text(
                    p.disponible ? 'Disponible' : 'No disp.',
                    style: const TextStyle(fontSize: 11),
                  ),
                  backgroundColor: p.disponible
                      ? Colors.green.shade100
                      : Colors.red.shade100,
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

```bash
flutter run
```

**Prueba esto:**
- Verifica que el Teclado muestra "Bajo" (stock = 3) y "Disponible"
- Verifica que la Webcam muestra "Agotado" y "No disp." (activo = false, stock = 0)
- Verifica que el Monitor muestra "Alto" (stock = 25)
- El precio con IVA del Monitor debe ser `$416.49`

**Salida esperada:**

```
Teclado Mecánico  | Stock: 3 — Bajo   | Disponible
Monitor 27"       | Stock: 25 — Alto  | Disponible
Webcam HD         | Stock: 0 — Agotado | No disp.
```

---

## Paso 2 — Capa Data: DTO + Repositorio fake

Agrega la capa Data con un repositorio falso (sin HTTP).
Perfecta para desarrollar UI sin depender de la API real.

Crea los siguientes archivos:

**`lib/domain/entities/producto.dart`** — mueve la entidad de `main.dart` aquí:

```dart
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

  bool   get disponible  => activo && stock > 0;
  bool   get stockBajo   => stock > 0 && stock <= 5;
  double get precioConIva => precio * 1.19;

  String get nivelStock => switch (stock) {
    0    => 'Agotado',
    <= 5  => 'Bajo',
    <= 20 => 'Normal',
    _    => 'Alto',
  };

  @override
  bool operator ==(Object other) => other is Producto && other.id == id;

  @override
  int get hashCode => id.hashCode;
}
```

**`lib/domain/repositories/i_productos_repository.dart`**:

```dart
import '../entities/producto.dart';

abstract interface class IProductosRepository {
  Future<({List<Producto> items, bool hayMas})> obtenerProductos({
    int    pagina        = 1,
    int    tamanioPagina = 10,
    String busqueda      = '',
  });
  Future<Producto?> obtenerPorId(int id);
}
```

**`lib/data/dtos/producto_dto.dart`**:

```dart
import '../../domain/entities/producto.dart';

class ProductoDto {
  final int     id;
  final String  name;
  final String  price;
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

  factory ProductoDto.fromJson(Map<String, dynamic> json) => ProductoDto(
    id:           json['id']            as int,
    name:         json['name']          as String,
    price:        json['price']         as String,
    categoryName: json['category_name'] as String?,
    isActive:     json['is_active']     as bool? ?? false,
    stock:        json['stock']         as int?  ?? 0,
    urlImage:     json['url_image']     as String?,
  );

  Producto toDomain() => Producto(
    id:        id,
    nombre:    name,
    precio:    double.tryParse(price) ?? 0.0,
    categoria: categoryName ?? 'Sin categoría',
    activo:    isActive,
    stock:     stock,
    imagenUrl: urlImage,
  );
}
```

**`lib/data/repositories/productos_repository_fake.dart`**:

```dart
import '../../domain/entities/producto.dart';
import '../../domain/repositories/i_productos_repository.dart';

// Fake — implementa la interfaz con datos en memoria
// Útil para desarrollo sin API y para tests
class ProductosRepositoryFake implements IProductosRepository {
  static final _datos = [
    const Producto(id: 1, nombre: 'Teclado', precio: 89.99,
        categoria: 'Periféricos', activo: true, stock: 3),
    const Producto(id: 2, nombre: 'Monitor', precio: 349.99,
        categoria: 'Monitores', activo: true, stock: 25),
    const Producto(id: 3, nombre: 'Webcam', precio: 55.00,
        categoria: 'Accesorios', activo: false, stock: 0),
    const Producto(id: 4, nombre: 'Mouse', precio: 35.00,
        categoria: 'Periféricos', activo: true, stock: 12),
    const Producto(id: 5, nombre: 'Auriculares', precio: 120.00,
        categoria: 'Audio', activo: true, stock: 5),
  ];

  @override
  Future<({List<Producto> items, bool hayMas})> obtenerProductos({
    int pagina = 1, int tamanioPagina = 10, String busqueda = '',
  }) async {
    await Future.delayed(const Duration(milliseconds: 400)); // simula red
    var lista = _datos;
    if (busqueda.isNotEmpty) {
      lista = _datos
          .where((p) => p.nombre.toLowerCase()
              .contains(busqueda.toLowerCase()))
          .toList();
    }
    return (items: lista, hayMas: false);
  }

  @override
  Future<Producto?> obtenerPorId(int id) async {
    await Future.delayed(const Duration(milliseconds: 200));
    try {
      return _datos.firstWhere((p) => p.id == id);
    } catch (_) {
      return null;
    }
  }
}
```

**`lib/main.dart` — Paso 2** (usa fake repo con `FutureBuilder`):

```dart
import 'package:flutter/material.dart';
import 'domain/entities/producto.dart';
import 'domain/repositories/i_productos_repository.dart';
import 'data/repositories/productos_repository_fake.dart';

// Flag para desarrollo sin API
const bool kUsarFake = true;

void main() => runApp(const App());

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Paso 2 — DTO + Fake',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const PantallaLista(),
    );
  }
}

class PantallaLista extends StatefulWidget {
  const PantallaLista({super.key});

  @override
  State<PantallaLista> createState() => _PantallaListaState();
}

class _PantallaListaState extends State<PantallaLista> {
  // La UI conoce la interfaz, no la implementación
  final IProductosRepository _repo = ProductosRepositoryFake();
  late final Future<List<Producto>> _futuro;

  @override
  void initState() {
    super.initState();
    _futuro = _repo.obtenerProductos()
        .then((r) => r.items);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Paso 2 — DTO + Fake Repo'),
        backgroundColor: Colors.teal.shade100,
      ),
      body: FutureBuilder<List<Producto>>(
        future: _futuro,
        builder: (ctx, snap) {
          if (snap.connectionState == ConnectionState.waiting) {
            return const Center(child: CircularProgressIndicator());
          }
          if (snap.hasError) {
            return Center(child: Text('Error: ${snap.error}'));
          }
          final productos = snap.data!;
          return ListView.builder(
            padding: const EdgeInsets.all(12),
            itemCount: productos.length,
            itemBuilder: (_, i) {
              final p = productos[i];
              return Card(
                margin: const EdgeInsets.only(bottom: 8),
                child: ListTile(
                  title:    Text(p.nombre),
                  subtitle: Text('${p.categoria} · \$${p.precio}'),
                  trailing: Text(p.nivelStock),
                ),
              );
            },
          );
        },
      ),
    );
  }
}
```

```bash
flutter run
```

**Prueba esto:**
- La app muestra los 5 productos del fake repo tras un breve delay
- Cambia `kUsarFake = false` (aún no hay impl real, compilará pero no funcionará)
- Verifica que la UI solo conoce `IProductosRepository`, nunca `ProductosRepositoryFake`

---

## Paso 3 — Use Cases + Repositorio real

Agrega los use cases y conecta la API real.

**`lib/domain/usecases/obtener_productos_uc.dart`**:

```dart
import '../entities/producto.dart';
import '../repositories/i_productos_repository.dart';

class ObtenerProductosUc {
  final IProductosRepository _repo;
  const ObtenerProductosUc(this._repo);

  Future<({List<Producto> items, bool hayMas})> call({
    int    pagina        = 1,
    int    tamanioPagina = 10,
    String busqueda      = '',
  }) => _repo.obtenerProductos(
    pagina:        pagina,
    tamanioPagina: tamanioPagina,
    busqueda:      busqueda,
  );
}
```

**`lib/domain/usecases/buscar_productos_uc.dart`**:

```dart
import '../entities/producto.dart';
import '../repositories/i_productos_repository.dart';

class BuscarProductosUc {
  final IProductosRepository _repo;
  const BuscarProductosUc(this._repo);

  Future<List<Producto>> call(String query) async {
    if (query.trim().length < 2) return [];
    final res = await _repo.obtenerProductos(
      busqueda:      query.trim(),
      tamanioPagina: 20,
    );
    return res.items;
  }
}
```

**`lib/data/repositories/productos_repository_impl.dart`**:

```dart
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

import '../../domain/entities/producto.dart';
import '../../domain/repositories/i_productos_repository.dart';
import '../dtos/producto_dto.dart';

class ProductosRepositoryImpl implements IProductosRepository {
  static const _baseUrl =
      'https://higuera-billing-api.desarrollo-software.xyz/api';

  final http.Client _client;
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

      if (response.statusCode != 200) {
        throw HttpException('HTTP ${response.statusCode}');
      }
      final json    = jsonDecode(response.body) as Map<String, dynamic>;
      final results = (json['results'] as List)
          .map((e) => ProductoDto.fromJson(e).toDomain())
          .toList();
      return (items: results, hayMas: json['next'] != null);
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
      if (response.statusCode == 404) return null;
      if (response.statusCode != 200) {
        throw HttpException('HTTP ${response.statusCode}');
      }
      final json = jsonDecode(response.body) as Map<String, dynamic>;
      return ProductoDto.fromJson(json).toDomain();
    } on SocketException {
      throw const SocketException('Sin conexión a internet');
    } catch (e) {
      throw Exception('Error inesperado: $e');
    }
  }
}
```

**`lib/main.dart` — Paso 3** (use case + impl real + FutureBuilder):

```dart
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;

import 'domain/entities/producto.dart';
import 'domain/usecases/obtener_productos_uc.dart';
import 'data/repositories/productos_repository_impl.dart';

void main() => runApp(const App());

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Paso 3 — Use Cases + API real',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const PantallaLista(),
    );
  }
}

class PantallaLista extends StatefulWidget {
  const PantallaLista({super.key});

  @override
  State<PantallaLista> createState() => _PantallaListaState();
}

class _PantallaListaState extends State<PantallaLista> {
  // El use case recibe el repo — la UI no conoce ProductosRepositoryImpl
  final _uc = ObtenerProductosUc(
    ProductosRepositoryImpl(http.Client()),
  );
  late Future<List<Producto>> _futuro;

  @override
  void initState() {
    super.initState();
    _cargar();
  }

  void _cargar() {
    setState(() {
      _futuro = _uc().then((r) => r.items);
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Paso 3 — API real'),
        backgroundColor: Colors.teal.shade100,
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: _cargar,
          ),
        ],
      ),
      body: FutureBuilder<List<Producto>>(
        future: _futuro,
        builder: (ctx, snap) {
          if (snap.connectionState == ConnectionState.waiting) {
            return const Center(child: CircularProgressIndicator());
          }
          if (snap.hasError) {
            return Center(
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  const Icon(Icons.cloud_off, size: 48, color: Colors.red),
                  const SizedBox(height: 12),
                  Text(snap.error.toString(), textAlign: TextAlign.center),
                  const SizedBox(height: 16),
                  ElevatedButton(
                    onPressed: _cargar,
                    child: const Text('Reintentar'),
                  ),
                ],
              ),
            );
          }
          final productos = snap.data!;
          return ListView.builder(
            padding: const EdgeInsets.all(12),
            itemCount: productos.length,
            itemBuilder: (_, i) {
              final p = productos[i];
              return Card(
                margin: const EdgeInsets.only(bottom: 8),
                child: ListTile(
                  title:    Text(p.nombre),
                  subtitle: Text('${p.categoria} · \$${p.precio}'),
                  trailing: Text(
                    p.nivelStock,
                    style: TextStyle(
                      color: p.nivelStock == 'Agotado'
                          ? Colors.red
                          : Colors.green,
                    ),
                  ),
                ),
              );
            },
          );
        },
      ),
    );
  }
}
```

```bash
flutter run
```

**Prueba esto:**
- La app carga productos reales desde la API
- Desconecta el wifi: debe aparecer el mensaje "Sin conexión a internet"
- El botón Reintentar vuelve a intentar la carga
- El use case `uc()` se invoca como función gracias a `operator call()`

---

## Paso 4 — Providers Riverpod + CatalogoNotifier

Conecta todo con Riverpod. Estado reactivo, inyección automática.

**`lib/ui/providers/productos_providers.dart`**:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:http/http.dart' as http;

import '../../data/repositories/productos_repository_impl.dart';
import '../../domain/repositories/i_productos_repository.dart';
import '../../domain/usecases/obtener_productos_uc.dart';
import '../../domain/usecases/buscar_productos_uc.dart';
import '../notifiers/catalogo_notifier.dart';

// Infraestructura
final httpClientProvider = Provider<http.Client>((ref) {
  final client = http.Client();
  ref.onDispose(client.close);
  return client;
});

// Repositorio — expuesto como interfaz, no implementación
final productosRepositoryProvider =
    Provider<IProductosRepository>((ref) =>
  ProductosRepositoryImpl(ref.watch(httpClientProvider)));

// Use cases — inyectan el repo automáticamente
final obtenerProductosUcProvider =
    Provider<ObtenerProductosUc>((ref) =>
  ObtenerProductosUc(ref.watch(productosRepositoryProvider)));

final buscarProductosUcProvider =
    Provider<BuscarProductosUc>((ref) =>
  BuscarProductosUc(ref.watch(productosRepositoryProvider)));

// Notifier del catálogo
final catalogoProvider =
    NotifierProvider<CatalogoNotifier, CatalogoState>(
  CatalogoNotifier.new);
```

**`lib/ui/notifiers/catalogo_notifier.dart`**:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../domain/entities/producto.dart';
import '../providers/productos_providers.dart';

// Estado inmutable del catálogo
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

  // Derivados — se calculan del estado base
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
  }) => CatalogoState(
    productos:    productos    ?? this.productos,
    cargando:     cargando     ?? this.cargando,
    cargandoMas:  cargandoMas  ?? this.cargandoMas,
    error:        limpiarError ? null : (error ?? this.error),
    paginaActual: paginaActual ?? this.paginaActual,
    hayMas:       hayMas       ?? this.hayMas,
    busqueda:     busqueda     ?? this.busqueda,
  );
}

// Notifier — solo llama use cases, no sabe de HTTP ni JSON
class CatalogoNotifier extends Notifier<CatalogoState> {
  @override
  CatalogoState build() {
    Future.microtask(cargar);
    return const CatalogoState();
  }

  Future<void> cargar() async {
    state = state.copyWith(cargando: true, limpiarError: true);
    try {
      final uc = ref.read(obtenerProductosUcProvider);
      final resultado = await uc(busqueda: state.busqueda);
      state = state.copyWith(
        productos:    resultado.items,
        hayMas:       resultado.hayMas,
        paginaActual: 1,
        cargando:     false,
      );
    } catch (e) {
      state = state.copyWith(cargando: false, error: e.toString());
    }
  }

  Future<void> buscar(String query) async {
    state = state.copyWith(
        busqueda: query, cargando: true, limpiarError: true);
    try {
      if (query.trim().isEmpty) {
        final uc = ref.read(obtenerProductosUcProvider);
        final resultado = await uc();
        state = state.copyWith(
          productos:    resultado.items,
          hayMas:       resultado.hayMas,
          paginaActual: 1,
          cargando:     false,
        );
      } else {
        final uc = ref.read(buscarProductosUcProvider);
        final items = await uc(query);
        state = state.copyWith(
          productos:    items,
          hayMas:       false,
          paginaActual: 1,
          cargando:     false,
        );
      }
    } catch (e) {
      state = state.copyWith(cargando: false, error: e.toString());
    }
  }

  Future<void> cargarMas() async {
    if (!state.hayMas || state.cargandoMas || state.busqueda.isNotEmpty) {
      return;
    }
    state = state.copyWith(cargandoMas: true);
    try {
      final siguiente = state.paginaActual + 1;
      final uc        = ref.read(obtenerProductosUcProvider);
      final resultado = await uc(pagina: siguiente);
      state = state.copyWith(
        productos:    [...state.productos, ...resultado.items],
        hayMas:       resultado.hayMas,
        paginaActual: siguiente,
        cargandoMas:  false,
      );
    } catch (e) {
      state = state.copyWith(cargandoMas: false, error: e.toString());
    }
  }
}
```

**`lib/main.dart` — Paso 4** (Riverpod conectado, switch en UI):

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'ui/notifiers/catalogo_notifier.dart';
import 'ui/providers/productos_providers.dart';

void main() => runApp(const ProviderScope(child: App()));

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Paso 4 — Riverpod',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const PantallaRiverpod(),
    );
  }
}

class PantallaRiverpod extends ConsumerWidget {
  const PantallaRiverpod({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final estado = ref.watch(catalogoProvider);
    final cs     = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Paso 4 — Riverpod'),
        backgroundColor: cs.primaryContainer,
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: () =>
                ref.read(catalogoProvider.notifier).cargar(),
          ),
        ],
      ),
      // switch en la UI según el estado
      body: switch ((estado.cargando, estado.error, estado.estaVacio)) {
        // Cargando por primera vez
        (true, _, _) => const Center(
            child: CircularProgressIndicator()),

        // Error sin datos previos
        (_, String e, _) when !estado.tieneDatos => Center(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                const Icon(Icons.cloud_off, size: 64, color: Colors.red),
                const SizedBox(height: 12),
                Text(e, textAlign: TextAlign.center),
                const SizedBox(height: 16),
                FilledButton.icon(
                  onPressed: () =>
                      ref.read(catalogoProvider.notifier).cargar(),
                  icon:  const Icon(Icons.refresh),
                  label: const Text('Reintentar'),
                ),
              ],
            ),
          ),

        // Sin resultados
        (_, _, true) => const Center(
            child: Text('No hay productos')),

        // Lista
        _ => ListView.builder(
            padding:   const EdgeInsets.all(12),
            itemCount: estado.productos.length,
            itemBuilder: (_, i) {
              final p = estado.productos[i];
              return Card(
                margin: const EdgeInsets.only(bottom: 8),
                child: ListTile(
                  title:    Text(p.nombre),
                  subtitle: Text('${p.categoria} · \$${p.precio}'),
                  trailing: Text(p.nivelStock),
                ),
              );
            },
          ),
      },
    );
  }
}
```

```bash
flutter run
```

**Prueba esto:**
- La app usa Riverpod: el estado es reactivo y se reconstruye al cambiar
- El botón Refresh llama a `cargar()` en el notifier
- El notifier NO importa nada de HTTP — solo llama use cases
- Observa la cadena: `catalogoProvider` → `obtenerProductosUcProvider` → `productosRepositoryProvider` → `httpClientProvider`

---

## Paso 5 — Pantalla completa con búsqueda y paginación infinita

La pantalla final con `SearchBar`, paginación infinita y tarjetas de producto.

**`lib/ui/screens/pantalla_catalogo.dart`**:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import '../../domain/entities/producto.dart';
import '../notifiers/catalogo_notifier.dart';
import '../providers/productos_providers.dart';

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
    // Disparar paginación cuando quedan 200px para el final
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
        title:           const Text('Catálogo'),
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

          // Contenido principal
          Expanded(
            child: switch ((
              estado.cargando,
              estado.error,
              estado.estaVacio,
            )) {

              // Primera carga
              (true, _, _) => const Center(
                  child: CircularProgressIndicator()),

              // Error sin datos previos
              (_, String e, _) when !estado.tieneDatos =>
                _PantallaError(
                  mensaje:      e,
                  onReintentar: () =>
                      ref.read(catalogoProvider.notifier).cargar(),
                ),

              // Lista vacía (sin resultados de búsqueda)
              (_, _, true) => _EstadoVacio(busqueda: estado.busqueda),

              // Lista con datos
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
                          child: Row(
                            children: [
                              Icon(Icons.warning_amber,
                                  color: cs.onErrorContainer),
                              const SizedBox(width: 8),
                              Expanded(
                                child: Text(
                                  estado.error!,
                                  style: TextStyle(
                                      color: cs.onErrorContainer),
                                ),
                              ),
                            ],
                          ),
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

// ── Widgets auxiliares ─────────────────────────────────────────────

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
                          const _PlaceholderImagen(size: 72),
                    )
                  : const _PlaceholderImagen(size: 72),
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
                            fontSize:   15),
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
                                color:    cs.onErrorContainer)),
                      ),
                  ]),

                  Text(
                    producto.categoria,
                    style: TextStyle(
                        fontSize: 12,
                        color:    cs.onSurfaceVariant),
                  ),

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
                      stock: producto.stock,
                    ),
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
    child: Icon(
      Icons.image_outlined,
      size:  size * 0.5,
      color: Colors.grey.shade400,
    ),
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
      Text(
        '$stock · $nivel',
        style: TextStyle(
            fontSize:   12,
            color:      _color,
            fontWeight: FontWeight.w600),
      ),
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
  const _PantallaError({required this.mensaje, required this.onReintentar});

  @override
  Widget build(BuildContext context) => Center(
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
            style:     const TextStyle(color: Colors.red),
          ),
          const SizedBox(height: 24),
          FilledButton.icon(
            onPressed: onReintentar,
            icon:      const Icon(Icons.refresh),
            label:     const Text('Reintentar'),
          ),
        ],
      ),
    ),
  );
}
```

**`lib/main.dart` — Paso 5** (app completa):

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'ui/screens/pantalla_catalogo.dart';

void main() => runApp(const ProviderScope(child: App()));

class App extends StatelessWidget {
  const App({super.key});

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
```

**Prueba esto:**
- Escribe en el buscador: los resultados se filtran en tiempo real
- Borra el texto: la lista vuelve a la vista completa
- Llega al final: aparece el spinner y carga la siguiente página
- Desconecta wifi mientras cargas más: aparece el banner naranja en la parte inferior (sin perder los datos ya cargados)
- El botón de refresh en la AppBar recarga desde la página 1

**Comportamiento esperado:**

```
[ SearchBar: "Buscar producto..." ]
┌─────────────────────────────────────────┐
│ [img] Teclado Mecánico                  │
│       Periféricos              3 · Bajo │
│       $89.99                            │
├─────────────────────────────────────────┤
│ [img] Monitor 27"                       │
│       Monitores              25 · Alto  │
│       $349.99                           │
├─────────────────────────────────────────┤
│ [img] ...                               │
│              [CircularProgressIndicator]│ ← al llegar al final
└─────────────────────────────────────────┘
```

---

## `main.dart` completo — referencia

Este archivo único activa cada paso cambiando `const int paso = 5`:

```dart
// lib/main.dart — versión de referencia con selector de pasos
// Cambia el valor de [paso] entre 1 y 5 para probar cada etapa

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:http/http.dart' as http;

import 'domain/entities/producto.dart';
import 'domain/usecases/obtener_productos_uc.dart';
import 'data/repositories/productos_repository_fake.dart';
import 'data/repositories/productos_repository_impl.dart';
import 'ui/screens/pantalla_catalogo.dart';
import 'ui/providers/productos_providers.dart';
import 'ui/notifiers/catalogo_notifier.dart';

// ────────────────────────────────────────────────────────────────
// SELECTOR DE PASOS — cambia este valor entre 1 y 5
const int paso = 5;
// ────────────────────────────────────────────────────────────────

void main() {
  if (paso >= 4) {
    runApp(const ProviderScope(child: _AppRiverpod()));
  } else {
    runApp(const _AppSimple());
  }
}

// ── Paso 1-3: sin Riverpod ────────────────────────────────────────

class _AppSimple extends StatelessWidget {
  const _AppSimple();

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title:                    'Catálogo — Paso $paso',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: paso == 1
          ? const _Paso1()
          : paso == 2
              ? const _Paso2()
              : const _Paso3(),
    );
  }
}

// Paso 1: entidad hardcodeada
class _Paso1 extends StatelessWidget {
  const _Paso1();

  static final _productos = [
    const Producto(id: 1, nombre: 'Teclado', precio: 89.99,
        categoria: 'Periféricos', activo: true,  stock: 3),
    const Producto(id: 2, nombre: 'Monitor', precio: 349.99,
        categoria: 'Monitores',  activo: true,  stock: 25),
    const Producto(id: 3, nombre: 'Webcam',  precio: 55.00,
        categoria: 'Accesorios', activo: false, stock: 0),
  ];

  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Paso 1 — Entidad')),
    body: ListView.builder(
      padding:   const EdgeInsets.all(12),
      itemCount: _productos.length,
      itemBuilder: (_, i) {
        final p = _productos[i];
        return Card(
          margin: const EdgeInsets.only(bottom: 8),
          child:  ListTile(
            title:    Text(p.nombre),
            subtitle: Text(
                '${p.categoria} · \$${p.precio} · ${p.nivelStock}'),
            trailing: Icon(
              p.disponible ? Icons.check_circle : Icons.cancel,
              color: p.disponible ? Colors.green : Colors.red,
            ),
          ),
        );
      },
    ),
  );
}

// Paso 2: fake repo con FutureBuilder
class _Paso2 extends StatefulWidget {
  const _Paso2();
  @override
  State<_Paso2> createState() => _Paso2State();
}

class _Paso2State extends State<_Paso2> {
  final _repo = ProductosRepositoryFake();
  late final Future<List<Producto>> _futuro;

  @override
  void initState() {
    super.initState();
    _futuro = _repo.obtenerProductos().then((r) => r.items);
  }

  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Paso 2 — Fake Repo')),
    body: FutureBuilder<List<Producto>>(
      future: _futuro,
      builder: (_, snap) {
        if (snap.connectionState == ConnectionState.waiting) {
          return const Center(child: CircularProgressIndicator());
        }
        if (snap.hasError) {
          return Center(child: Text('Error: ${snap.error}'));
        }
        return ListView.builder(
          padding:   const EdgeInsets.all(12),
          itemCount: snap.data!.length,
          itemBuilder: (_, i) {
            final p = snap.data![i];
            return Card(
              margin: const EdgeInsets.only(bottom: 8),
              child:  ListTile(
                title:    Text(p.nombre),
                subtitle: Text('${p.categoria} · ${p.nivelStock}'),
              ),
            );
          },
        );
      },
    ),
  );
}

// Paso 3: repo real + use case
class _Paso3 extends StatefulWidget {
  const _Paso3();
  @override
  State<_Paso3> createState() => _Paso3State();
}

class _Paso3State extends State<_Paso3> {
  final _uc = ObtenerProductosUc(
      ProductosRepositoryImpl(http.Client()));
  late Future<List<Producto>> _futuro;

  @override
  void initState() {
    super.initState();
    _cargar();
  }

  void _cargar() => setState(() {
    _futuro = _uc().then((r) => r.items);
  });

  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(
      title: const Text('Paso 3 — API real'),
      actions: [
        IconButton(icon: const Icon(Icons.refresh), onPressed: _cargar),
      ],
    ),
    body: FutureBuilder<List<Producto>>(
      future: _futuro,
      builder: (_, snap) {
        if (snap.connectionState == ConnectionState.waiting) {
          return const Center(child: CircularProgressIndicator());
        }
        if (snap.hasError) {
          return Center(
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                Text('Error: ${snap.error}'),
                const SizedBox(height: 12),
                ElevatedButton(
                  onPressed: _cargar,
                  child: const Text('Reintentar'),
                ),
              ],
            ),
          );
        }
        return ListView.builder(
          padding:   const EdgeInsets.all(12),
          itemCount: snap.data!.length,
          itemBuilder: (_, i) {
            final p = snap.data![i];
            return Card(
              margin: const EdgeInsets.only(bottom: 8),
              child:  ListTile(
                title:    Text(p.nombre),
                subtitle: Text('${p.categoria} · \$${p.precio}'),
                trailing: Text(p.nivelStock),
              ),
            );
          },
        );
      },
    ),
  );
}

// ── Paso 4-5: con Riverpod ────────────────────────────────────────

class _AppRiverpod extends StatelessWidget {
  const _AppRiverpod();

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title:                    'Catálogo Clean — Paso $paso',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: paso == 4
          ? const _Paso4()
          : const PantallaCatalogo(),
    );
  }
}

// Paso 4: Riverpod básico con switch en UI
class _Paso4 extends ConsumerWidget {
  const _Paso4();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final estado = ref.watch(catalogoProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Paso 4 — Riverpod'),
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: () =>
                ref.read(catalogoProvider.notifier).cargar(),
          ),
        ],
      ),
      body: switch ((
        estado.cargando,
        estado.error,
        estado.estaVacio,
      )) {
        (true, _, _)  => const Center(child: CircularProgressIndicator()),
        (_, String e, _) when !estado.tieneDatos => Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Text('Error: $e'),
              FilledButton(
                onPressed: () =>
                    ref.read(catalogoProvider.notifier).cargar(),
                child: const Text('Reintentar'),
              ),
            ],
          ),
        ),
        (_, _, true) => const Center(child: Text('Sin productos')),
        _ => ListView.builder(
          padding:   const EdgeInsets.all(12),
          itemCount: estado.productos.length,
          itemBuilder: (_, i) {
            final p = estado.productos[i];
            return Card(
              margin: const EdgeInsets.only(bottom: 8),
              child:  ListTile(
                title:    Text(p.nombre),
                subtitle: Text('${p.categoria} · \$${p.precio}'),
                trailing: Text(p.nivelStock),
              ),
            );
          },
        ),
      },
    );
  }
}
```

---

## Proyecto final

### Estructura completa

```
productos_clean/
├── pubspec.yaml
├── lib/
│   ├── main.dart
│   ├── domain/
│   │   ├── entities/
│   │   │   └── producto.dart
│   │   ├── repositories/
│   │   │   └── i_productos_repository.dart
│   │   └── usecases/
│   │       ├── obtener_productos_uc.dart
│   │       └── buscar_productos_uc.dart
│   ├── data/
│   │   ├── dtos/
│   │   │   └── producto_dto.dart
│   │   └── repositories/
│   │       ├── productos_repository_impl.dart
│   │       └── productos_repository_fake.dart
│   └── ui/
│       ├── providers/
│       │   └── productos_providers.dart
│       ├── notifiers/
│       │   └── catalogo_notifier.dart
│       └── screens/
│           └── pantalla_catalogo.dart
└── test/
    ├── domain/
    │   ├── producto_test.dart
    │   └── buscar_productos_uc_test.dart
    └── data/
        └── producto_dto_test.dart
```

---

### `lib/domain/entities/producto.dart`

```dart
/// Entidad de dominio — Dart puro, sin imports externos.
/// Toda la lógica de negocio de Producto vive aquí.
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

  bool   get disponible   => activo && stock > 0;
  bool   get stockBajo    => stock > 0 && stock <= 5;
  double get precioConIva => precio * 1.19;

  String get nivelStock => switch (stock) {
    0    => 'Agotado',
    <= 5  => 'Bajo',
    <= 20 => 'Normal',
    _    => 'Alto',
  };

  String get etiquetaPrecio =>
      '\$$precio (IVA inc. \$${precioConIva.toStringAsFixed(2)})';

  @override
  String toString() => 'Producto($id: $nombre @ \$$precio)';

  @override
  bool operator ==(Object other) => other is Producto && other.id == id;

  @override
  int get hashCode => id.hashCode;
}
```

---

### `lib/domain/repositories/i_productos_repository.dart`

```dart
import '../entities/producto.dart';

/// Contrato del repositorio de productos.
/// La capa UI solo conoce esta interfaz — nunca la implementación.
abstract interface class IProductosRepository {
  Future<({List<Producto> items, bool hayMas})> obtenerProductos({
    int    pagina        = 1,
    int    tamanioPagina = 10,
    String busqueda      = '',
  });

  Future<Producto?> obtenerPorId(int id);
}
```

---

### `lib/domain/usecases/obtener_productos_uc.dart`

```dart
import '../entities/producto.dart';
import '../repositories/i_productos_repository.dart';

/// Use case: obtener una página de productos.
/// Una clase = una operación de negocio.
class ObtenerProductosUc {
  final IProductosRepository _repo;
  const ObtenerProductosUc(this._repo);

  Future<({List<Producto> items, bool hayMas})> call({
    int    pagina        = 1,
    int    tamanioPagina = 10,
    String busqueda      = '',
  }) => _repo.obtenerProductos(
    pagina:        pagina,
    tamanioPagina: tamanioPagina,
    busqueda:      busqueda,
  );
}
```

---

### `lib/domain/usecases/buscar_productos_uc.dart`

```dart
import '../entities/producto.dart';
import '../repositories/i_productos_repository.dart';

/// Use case: buscar productos por texto.
/// Aplica la regla de negocio: no buscar con menos de 2 caracteres.
class BuscarProductosUc {
  final IProductosRepository _repo;
  const BuscarProductosUc(this._repo);

  Future<List<Producto>> call(String query) async {
    if (query.trim().length < 2) return [];
    final res = await _repo.obtenerProductos(
      busqueda:      query.trim(),
      tamanioPagina: 20,
    );
    return res.items;
  }
}
```

---

### `lib/data/dtos/producto_dto.dart`

```dart
import '../../domain/entities/producto.dart';

/// DTO — representa la estructura del JSON que devuelve la API.
/// snake_case igual que el JSON. Nunca sale de la capa Data.
class ProductoDto {
  final int     id;
  final String  name;
  final String  price;          // API devuelve String, no double
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

  factory ProductoDto.fromJson(Map<String, dynamic> json) => ProductoDto(
    id:           json['id']            as int,
    name:         json['name']          as String,
    price:        json['price']         as String,
    categoryName: json['category_name'] as String?,
    isActive:     json['is_active']     as bool? ?? false,
    stock:        json['stock']         as int?  ?? 0,
    urlImage:     json['url_image']     as String?,
  );

  /// Convertir a entidad de dominio.
  /// Aquí ocurren todas las transformaciones: tipo, nombre de campo.
  Producto toDomain() => Producto(
    id:        id,
    nombre:    name,
    precio:    double.tryParse(price) ?? 0.0,
    categoria: categoryName ?? 'Sin categoría',
    activo:    isActive,
    stock:     stock,
    imagenUrl: urlImage,
  );
}
```

---

### `lib/data/repositories/productos_repository_impl.dart`

```dart
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

import '../../domain/entities/producto.dart';
import '../../domain/repositories/i_productos_repository.dart';
import '../dtos/producto_dto.dart';

/// Implementación real — HTTP + JSON.
/// La UI NUNCA importa esta clase directamente.
class ProductosRepositoryImpl implements IProductosRepository {
  static const _baseUrl =
      'https://higuera-billing-api.desarrollo-software.xyz/api';

  final http.Client _client;
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

      if (response.statusCode != 200) {
        throw HttpException('HTTP ${response.statusCode}');
      }
      final json    = jsonDecode(response.body) as Map<String, dynamic>;
      final results = (json['results'] as List)
          .map((e) => ProductoDto.fromJson(e).toDomain())
          .toList();
      return (items: results, hayMas: json['next'] != null);
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
      if (response.statusCode == 404) return null;
      if (response.statusCode != 200) {
        throw HttpException('HTTP ${response.statusCode}');
      }
      final json = jsonDecode(response.body) as Map<String, dynamic>;
      return ProductoDto.fromJson(json).toDomain();
    } on SocketException {
      throw const SocketException('Sin conexión a internet');
    } catch (e) {
      throw Exception('Error inesperado: $e');
    }
  }
}
```

---

### `lib/data/repositories/productos_repository_fake.dart`

```dart
import '../../domain/entities/producto.dart';
import '../../domain/repositories/i_productos_repository.dart';

/// Repositorio falso — sin HTTP, ideal para tests y desarrollo sin red.
/// Para activarlo: cambia productosRepositoryProvider en productos_providers.dart
class ProductosRepositoryFake implements IProductosRepository {
  static final _datos = [
    const Producto(id: 1, nombre: 'Teclado Mecánico', precio: 89.99,
        categoria: 'Periféricos', activo: true,  stock: 3),
    const Producto(id: 2, nombre: 'Monitor 27"',      precio: 349.99,
        categoria: 'Monitores',  activo: true,  stock: 25),
    const Producto(id: 3, nombre: 'Webcam HD',        precio: 55.00,
        categoria: 'Accesorios', activo: false, stock: 0),
    const Producto(id: 4, nombre: 'Mouse Inalámbrico', precio: 35.00,
        categoria: 'Periféricos', activo: true, stock: 12),
    const Producto(id: 5, nombre: 'Auriculares BT',   precio: 120.00,
        categoria: 'Audio',      activo: true,  stock: 5),
  ];

  @override
  Future<({List<Producto> items, bool hayMas})> obtenerProductos({
    int pagina = 1, int tamanioPagina = 10, String busqueda = '',
  }) async {
    await Future.delayed(const Duration(milliseconds: 400));
    var lista = _datos;
    if (busqueda.isNotEmpty) {
      lista = _datos
          .where((p) => p.nombre.toLowerCase()
              .contains(busqueda.toLowerCase()))
          .toList();
    }
    return (items: lista, hayMas: false);
  }

  @override
  Future<Producto?> obtenerPorId(int id) async {
    await Future.delayed(const Duration(milliseconds: 200));
    try {
      return _datos.firstWhere((p) => p.id == id);
    } catch (_) {
      return null;
    }
  }
}
```

---

### `lib/ui/providers/productos_providers.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:http/http.dart' as http;

import '../../data/repositories/productos_repository_impl.dart';
import '../../domain/repositories/i_productos_repository.dart';
import '../../domain/usecases/obtener_productos_uc.dart';
import '../../domain/usecases/buscar_productos_uc.dart';
import '../notifiers/catalogo_notifier.dart';

// ── Infraestructura ────────────────────────────────────────────────

/// Cliente HTTP — singleton con cierre automático al destruir el provider
final httpClientProvider = Provider<http.Client>((ref) {
  final client = http.Client();
  ref.onDispose(client.close);
  return client;
});

// ── Repositorio ────────────────────────────────────────────────────

/// Expuesto como interfaz — cambiar la implementación: solo aquí
final productosRepositoryProvider =
    Provider<IProductosRepository>((ref) =>
  ProductosRepositoryImpl(ref.watch(httpClientProvider)));

// Para usar el fake en desarrollo, descomenta:
// Provider<IProductosRepository>((ref) => ProductosRepositoryFake());

// ── Use cases ──────────────────────────────────────────────────────

final obtenerProductosUcProvider =
    Provider<ObtenerProductosUc>((ref) =>
  ObtenerProductosUc(ref.watch(productosRepositoryProvider)));

final buscarProductosUcProvider =
    Provider<BuscarProductosUc>((ref) =>
  BuscarProductosUc(ref.watch(productosRepositoryProvider)));

// ── Notifier ───────────────────────────────────────────────────────

final catalogoProvider =
    NotifierProvider<CatalogoNotifier, CatalogoState>(
  CatalogoNotifier.new);
```

---

### `lib/ui/notifiers/catalogo_notifier.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../domain/entities/producto.dart';
import '../providers/productos_providers.dart';

/// Estado inmutable del catálogo de productos
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
  }) => CatalogoState(
    productos:    productos    ?? this.productos,
    cargando:     cargando     ?? this.cargando,
    cargandoMas:  cargandoMas  ?? this.cargandoMas,
    error:        limpiarError ? null : (error ?? this.error),
    paginaActual: paginaActual ?? this.paginaActual,
    hayMas:       hayMas       ?? this.hayMas,
    busqueda:     busqueda     ?? this.busqueda,
  );
}

/// Notifier — orquesta las operaciones del catálogo.
/// Solo llama use cases. No importa http ni jsonDecode.
class CatalogoNotifier extends Notifier<CatalogoState> {
  @override
  CatalogoState build() {
    Future.microtask(cargar);
    return const CatalogoState();
  }

  Future<void> cargar() async {
    state = state.copyWith(cargando: true, limpiarError: true);
    try {
      final uc        = ref.read(obtenerProductosUcProvider);
      final resultado = await uc(busqueda: state.busqueda);
      state = state.copyWith(
        productos:    resultado.items,
        hayMas:       resultado.hayMas,
        paginaActual: 1,
        cargando:     false,
      );
    } catch (e) {
      state = state.copyWith(cargando: false, error: e.toString());
    }
  }

  Future<void> buscar(String query) async {
    state = state.copyWith(
        busqueda: query, cargando: true, limpiarError: true);
    try {
      if (query.trim().isEmpty) {
        final uc        = ref.read(obtenerProductosUcProvider);
        final resultado = await uc();
        state = state.copyWith(
          productos:    resultado.items,
          hayMas:       resultado.hayMas,
          paginaActual: 1,
          cargando:     false,
        );
      } else {
        final uc    = ref.read(buscarProductosUcProvider);
        final items = await uc(query);
        state = state.copyWith(
          productos:    items,
          hayMas:       false,
          paginaActual: 1,
          cargando:     false,
        );
      }
    } catch (e) {
      state = state.copyWith(cargando: false, error: e.toString());
    }
  }

  Future<void> cargarMas() async {
    if (!state.hayMas || state.cargandoMas || state.busqueda.isNotEmpty) {
      return;
    }
    state = state.copyWith(cargandoMas: true);
    try {
      final siguiente = state.paginaActual + 1;
      final uc        = ref.read(obtenerProductosUcProvider);
      final resultado = await uc(pagina: siguiente);
      state = state.copyWith(
        productos:    [...state.productos, ...resultado.items],
        hayMas:       resultado.hayMas,
        paginaActual: siguiente,
        cargandoMas:  false,
      );
    } catch (e) {
      state = state.copyWith(cargandoMas: false, error: e.toString());
    }
  }
}
```

---

### `lib/ui/screens/pantalla_catalogo.dart`

*(Ver código completo en la sección "Paso 5" arriba — es el mismo archivo)*

---

### Tests

**`test/domain/producto_test.dart`**:

```dart
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

    test('no disponible cuando stock = 0', () {
      const p = Producto(
        id: 1, nombre: 'T', precio: 1, categoria: 'C',
        activo: true, stock: 0,
      );
      expect(p.disponible, isFalse);
    });

    test('nivelStock cubre todos los rangos', () {
      final casos = {
        0: 'Agotado', 1: 'Bajo', 5: 'Bajo',
        6: 'Normal', 20: 'Normal', 21: 'Alto',
      };
      for (final e in casos.entries) {
        final p = Producto(
          id: 1, nombre: 'T', precio: 1, categoria: 'C',
          activo: true, stock: e.key,
        );
        expect(p.nivelStock, e.value, reason: 'stock=${e.key}');
      }
    });

    test('igualdad basada en id', () {
      const p1 = Producto(
          id: 1, nombre: 'A', precio: 1, categoria: 'C',
          activo: true, stock: 1);
      const p2 = Producto(
          id: 1, nombre: 'B', precio: 99, categoria: 'D',
          activo: false, stock: 0);
      expect(p1, equals(p2)); // mismo id → iguales
    });

    test('precioConIva aplica 19%', () {
      const p = Producto(
        id: 1, nombre: 'T', precio: 100.0, categoria: 'C',
        activo: true, stock: 1,
      );
      expect(p.precioConIva, closeTo(119.0, 0.01));
    });
  });
}
```

**`test/data/producto_dto_test.dart`**:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:productos_clean/data/dtos/producto_dto.dart';

void main() {
  group('ProductoDto', () {

    const jsonCompleto = {
      'id': 42,    'name': 'Monitor UHD',
      'price': '349.99', 'category_name': 'Monitores',
      'is_active': true,  'stock': 8,
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
      expect(entidad.precio, 349.99);
      expect(entidad.nombre, 'Monitor UHD');
    });

    test('usa valores por defecto para campos opcionales', () {
      final dto = ProductoDto.fromJson(
          {'id': 1, 'name': 'Test', 'price': '10.00'});
      final entidad = dto.toDomain();
      expect(entidad.categoria, 'Sin categoría');
      expect(entidad.activo,    isFalse);
      expect(entidad.stock,     0);
    });

    test('precio inválido devuelve 0.0', () {
      final dto = ProductoDto.fromJson(
          {'id': 1, 'name': 'X', 'price': 'N/A'});
      expect(dto.toDomain().precio, 0.0);
    });
  });
}
```

**`test/domain/buscar_productos_uc_test.dart`**:

```dart
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
      final resultado = await uc('a');
      expect(resultado, isEmpty);
      verifyNever(mockRepo.obtenerProductos());
    });

    test('devuelve lista vacía con solo espacios', () async {
      final resultado = await uc('   ');
      expect(resultado, isEmpty);
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
      expect(resultado.length,       1);
      expect(resultado.first.nombre, 'Teclado');
    });

    test('trim del query antes de buscar', () async {
      when(mockRepo.obtenerProductos(
        busqueda:      'mouse',
        tamanioPagina: 20,
        pagina:        anyNamed('pagina'),
      )).thenAnswer((_) async => (
        items: <Producto>[],
        hayMas: false,
      ));

      await uc('  mouse  ');
      verify(mockRepo.obtenerProductos(
        busqueda:      'mouse',
        tamanioPagina: 20,
        pagina:        anyNamed('pagina'),
      )).called(1);
    });
  });
}
```

```bash
# Generar mocks de mockito
dart run build_runner build --delete-conflicting-outputs

# Ejecutar todos los tests
flutter test

# Solo tests de dominio
flutter test test/domain/

# Solo tests de data
flutter test test/data/
```

---

## Guía rápida de imports

| Qué necesitas | Import |
|---|---|
| Riverpod (Provider, Notifier...) | `package:flutter_riverpod/flutter_riverpod.dart` |
| http.Client, http.get | `package:http/http.dart` as http |
| jsonDecode, jsonEncode | `dart:convert` |
| SocketException, HttpException | `dart:io` |
| IProductosRepository | `../../domain/repositories/i_productos_repository.dart` |
| ProductoDto | `../../data/dtos/producto_dto.dart` |
| ProductosRepositoryImpl | `../../data/repositories/productos_repository_impl.dart` |
| ObtenerProductosUc | `../../domain/usecases/obtener_productos_uc.dart` |
| BuscarProductosUc | `../../domain/usecases/buscar_productos_uc.dart` |
| CatalogoState / CatalogoNotifier | `../notifiers/catalogo_notifier.dart` |
| catalogoProvider | `../providers/productos_providers.dart` |
| PantallaCatalogo | `../screens/pantalla_catalogo.dart` |

---

## Cuándo usar qué

```
┌──────────────────────────────┬───────────┬────────────────────────────────────────────┐
│ Elemento                     │ Capa      │ Razón                                      │
├──────────────────────────────┼───────────┼────────────────────────────────────────────┤
│ Entidad Producto             │ Domain    │ Lógica de negocio pura, sin dependencias   │
│ IProductosRepository         │ Domain    │ Contrato que UI conoce, no implementación  │
│ ObtenerProductosUc           │ Domain    │ Una operación, testeable sin mocks HTTP     │
│ BuscarProductosUc            │ Domain    │ Encapsula la regla "mínimo 2 caracteres"   │
│ ProductoDto                  │ Data      │ Mapea JSON → entidad; no contamina Domain  │
│ ProductosRepositoryImpl      │ Data      │ HTTP real; la UI NUNCA la importa directo  │
│ ProductosRepositoryFake      │ Data      │ Desarrollo/tests sin red, mismo contrato   │
│ httpClientProvider           │ UI/Infra  │ Singleton con ref.onDispose(client.close)  │
│ productosRepositoryProvider  │ UI/Infra  │ Expone la interfaz; cambiar Data: aquí     │
│ obtenerProductosUcProvider   │ UI/Infra  │ Riverpod inyecta el repo automáticamente   │
│ CatalogoState                │ UI        │ Estado inmutable con copyWith; derivados   │
│ CatalogoNotifier             │ UI        │ Solo llama use cases; no conoce HTTP       │
│ PantallaCatalogo             │ UI        │ ConsumerStatefulWidget para scroll infinito │
└──────────────────────────────┴───────────┴────────────────────────────────────────────┘
```

---

## Ejercicios propuestos

**1. Pantalla de detalle con `FutureProvider.family`**

Crea `ObtenerProductoPorIdUc` en `lib/domain/usecases/`.
Agrega en `productos_providers.dart`:

```dart
final productoPorIdProvider =
    FutureProvider.family<Producto?, int>((ref, id) {
  final uc = ref.watch(obtenerProductoPorIdUcProvider);
  return uc(id);
});
```

Crea `lib/ui/screens/pantalla_detalle.dart` que reciba un `id`,
observe `productoPorIdProvider(id)` y muestre imagen grande, nombre,
categoría, precio con IVA, nivel de stock con colores y un botón "Volver".

---

**2. Repositorio con caché en memoria**

Crea `ProductosRepositoryConCache` que implemente `IProductosRepository`.
Internamente mantiene un `Map<String, List<Producto>>` donde la clave es
`'$busqueda-$pagina'`. Si la clave existe, devuelve la caché en lugar de
llamar a la API. Agrega un método `limpiarCache()`.

Para activarlo, cambia solo la línea del provider:

```dart
// Antes:
Provider<IProductosRepository>((ref) =>
  ProductosRepositoryImpl(ref.watch(httpClientProvider)));

// Después:
Provider<IProductosRepository>((ref) =>
  ProductosRepositoryConCache(
    ProductosRepositoryImpl(ref.watch(httpClientProvider))));
```

---

**3. Tests del `CatalogoNotifier` con `ProviderContainer`**

```dart
// test/ui/catalogo_notifier_test.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
// ...imports...

void main() {
  test('estado inicial: cargando = true', () {
    final container = ProviderContainer(
      overrides: [
        productosRepositoryProvider.overrideWithValue(MockRepo()),
      ],
    );
    addTearDown(container.dispose);
    final estado = container.read(catalogoProvider);
    expect(estado.cargando, isTrue);
  });

  test('cargar() actualiza productos', () async {
    final mockRepo = MockIProductosRepository();
    when(mockRepo.obtenerProductos()).thenAnswer((_) async => (
      items: [const Producto(id: 1, nombre: 'T', precio: 1,
          categoria: 'C', activo: true, stock: 1)],
      hayMas: false,
    ));

    final container = ProviderContainer(
      overrides: [
        productosRepositoryProvider.overrideWithValue(mockRepo),
      ],
    );
    addTearDown(container.dispose);

    await container.read(catalogoProvider.notifier).cargar();
    expect(container.read(catalogoProvider).productos.length, 1);
    expect(container.read(catalogoProvider).cargando,         isFalse);
  });
}
```

Verifica también: `buscar('')` vuelve a la lista completa, `cargarMas()`
añade a la lista existente, `cargarMas()` no llama al repo si `hayMas = false`.

---

**4. Flag `kUsarFake` para desarrollo sin API**

En `lib/main.dart`:

```dart
// Cambia a true para desarrollar sin API
const bool kUsarFake = false;
```

En `lib/ui/providers/productos_providers.dart`:

```dart
import 'package:productos_clean/main.dart' show kUsarFake;

final productosRepositoryProvider =
    Provider<IProductosRepository>((ref) {
  if (kUsarFake) return ProductosRepositoryFake();
  return ProductosRepositoryImpl(ref.watch(httpClientProvider));
});
```

Con `kUsarFake = true` la app funciona sin red — util para demos y UI testing.
Con `kUsarFake = false` usa la API real. La UI no cambia nada.

---

## Resumen

- **Diagrama de capas:** Data conoce Domain. UI conoce Domain. Nadie conoce UI. Las dependencias siempre apuntan hacia adentro.

```
  UI  ──────────→  Domain  ←──────────  Data
  (conoce Domain)           (implementa Domain)
```

- **Entidad = Dart puro:** sin `fromJson`, sin imports de Flutter ni HTTP. Lógica de negocio en getters: `disponible`, `nivelStock`, `precioConIva`. Copiable a cualquier proyecto Dart.

- **DTO en Data:** hace el trabajo sucio de parsear JSON. `fromJson` vive en el DTO, no en la entidad. `toDomain()` convierte: `String → double`, `snake_case → camelCase`, nulos → valores por defecto.

- **`abstract interface class`:** define QUÉ puede hacer el repositorio, nunca el CÓMO. La UI importa la interfaz — cambiar de REST a GraphQL no toca la UI.

- **Use case:** una clase = una operación de negocio. `operator call()` permite invocarlos como funciones: `uc(pagina: 2)`. Testeable con un mock de la interfaz, sin necesidad de HTTP real.

- **Cadena de providers Riverpod:** `httpClient → repository → useCase → notifier`. Cada provider declara su dependencia explícitamente. Para testear, se hace override del provider base.

- **Notifier solo llama use cases:** no importa `http`, no importa `jsonDecode`. Si mañana cambias la API por GraphQL o por WebSocket, el notifier no cambia ni una línea.

- **Testeabilidad por capas:** domain con Dart puro (sin mocks), data con `MockIProductosRepository` (mockito), UI con `ProviderContainer` y override del repositoryProvider.

---

> **Siguiente página →** Página 20: Publicación — `flutter build apk/appbundle/ipa`, firma, Google Play Console, App Store Connect y CI/CD con GitHub Actions.
