# Tutorial Flutter — Página 13
## Módulo 4 · Calidad
### Testing: `flutter_test`, `testWidgets`, Mockito y providers con Riverpod

---

## ¿Por qué testear?

```
Sin tests                          Con tests
──────────────────────             ─────────────────────────────────────
Regressions al cambiar código      Los tests fallan antes del deploy
"Funciona en mi máquina"           Comportamiento verificado y reproducible
Miedo a refactorizar               Refactorización confiada
Bugs en producción                 Bugs detectados en desarrollo
```

Dart/Flutter tiene tres niveles de test:

```
Unit test         → función, clase, provider — sin UI — más rápido
Widget test       → un widget aislado — con render, sin dispositivo
Integration test  → app completa en emulador — más lento
```

Este módulo cubre unit tests y widget tests — los más usados en el día a día.

---

## Dependencias

```yaml
# pubspec.yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  mockito:         ^5.4.4
  build_runner:              # genera mocks con @GenerateMocks
  riverpod_test: ^2.4.0     # utilidades para testear providers
```

```bash
# Generar mocks después de añadir @GenerateMocks
dart run build_runner build --delete-conflicting-outputs
```

---

## Parte 1 — Unit tests

Los unit tests verifican funciones y clases en aislamiento,
sin dependencias externas ni UI:

```dart
// test/unit/producto_test.dart
import 'package:flutter_test/flutter_test.dart';

void main() {

  // group() agrupa tests relacionados
  group('Producto', () {

    test('nivelStock es Agotado cuando stock es 0', () {
      const producto = Producto(
        id: 1, nombre: 'Teclado', precio: 89.99,
        categoria: 'Periféricos', activo: true, stock: 0,
      );

      expect(producto.nivelStock, equals('Agotado'));
      expect(producto.disponible, isFalse);
    });

    test('nivelStock es Bajo cuando stock está entre 1 y 5', () {
      for (final stock in [1, 3, 5]) {
        final producto = Producto(
          id: 1, nombre: 'Test', precio: 10.0,
          categoria: 'Cat', activo: true, stock: stock,
        );
        expect(producto.nivelStock, equals('Bajo'),
            reason: 'stock=$stock debe ser Bajo');
      }
    });

    test('disponible requiere activo Y stock > 0', () {
      // Activo pero sin stock
      const p1 = Producto(id: 1, nombre: 'P1', precio: 10.0,
          categoria: 'C', activo: true, stock: 0);
      expect(p1.disponible, isFalse);

      // Con stock pero inactivo
      const p2 = Producto(id: 2, nombre: 'P2', precio: 10.0,
          categoria: 'C', activo: false, stock: 10);
      expect(p2.disponible, isFalse);

      // Activo y con stock
      const p3 = Producto(id: 3, nombre: 'P3', precio: 10.0,
          categoria: 'C', activo: true, stock: 5);
      expect(p3.disponible, isTrue);
    });
  });

  // Tests del parser JSON
  group('ProductoDto.fromJson', () {
    const jsonValido = {
      'id': 42,
      'name': 'Monitor UHD',
      'price': '349.99',
      'category_name': 'Monitores',
      'is_active': true,
      'stock': 8,
      'url_image': 'https://img.ejemplo.com/monitor.jpg',
    };

    test('parsea todos los campos correctamente', () {
      final dto = ProductoDto.fromJson(jsonValido);

      expect(dto.id,           equals(42));
      expect(dto.name,         equals('Monitor UHD'));
      expect(dto.price,        equals('349.99'));
      expect(dto.categoryName, equals('Monitores'));
      expect(dto.isActive,     isTrue);
      expect(dto.stock,        equals(8));
      expect(dto.urlImage,     equals('https://img.ejemplo.com/monitor.jpg'));
    });

    test('toDomain() convierte price String a double', () {
      final domain = ProductoDto.fromJson(jsonValido).toDomain();

      expect(domain.precio, equals(349.99));
      expect(domain.nombre, equals('Monitor UHD'));
    });

    test('usa valores por defecto cuando campos son null', () {
      final jsonMinimo = {'id': 1, 'name': 'Test', 'price': '10.00'};
      final dto = ProductoDto.fromJson(jsonMinimo);

      expect(dto.categoryName, equals('Sin categoría'));
      expect(dto.isActive,     isFalse);
      expect(dto.stock,        equals(0));
      expect(dto.urlImage,     isNull);
    });

    test('lanza cuando id no está presente', () {
      expect(
        () => ProductoDto.fromJson({'name': 'Sin id', 'price': '10.0'}),
        throwsA(isA<TypeError>()),
      );
    });
  });

  // Tests del Result
  group('Result', () {
    test('Success contiene el dato', () {
      const result = Success(42);
      expect(result, isA<Success<int>>());
      expect(result.data, equals(42));
    });

    test('Failure contiene el error', () {
      const result = Failure<int>(ErrorNotFound());
      expect(result, isA<Failure<int>>());
      expect(result.error, isA<ErrorNotFound>());
    });

    test('switch exhaustivo sobre Result', () {
      Result<String> resultado = const Success('hola');
      final texto = switch (resultado) {
        Success(data: final d) => 'OK: $d',
        Failure(error: final e) => 'Error: ${e.mensaje}',
      };
      expect(texto, equals('OK: hola'));
    });
  });
}
```

---

## Parte 2 — Mocks con Mockito

Mockito permite reemplazar dependencias reales (API, BD)
con versiones controladas en los tests:

```dart
// test/mocks/mocks.dart
import 'package:mockito/annotations.dart';

// Anotación — genera el mock automáticamente con build_runner
@GenerateMocks([ProductosRepository, HttpClient])
void main() {}
// Tras correr build_runner genera: mocks.mocks.dart

// Importar en los tests:
// import 'mocks/mocks.mocks.dart';
```

### Tests del repositorio con mock

```dart
// test/unit/productos_repository_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import '../mocks/mocks.mocks.dart';

void main() {
  late MockHttpClient mockHttp;
  late ProductosRepository repo;

  // setUp se ejecuta antes de CADA test
  setUp(() {
    mockHttp = MockHttpClient();
    repo     = ProductosRepository(mockHttp);
  });

  group('ProductosRepository.obtenerProductos', () {

    test('devuelve lista de productos al recibir JSON válido', () async {
      // ARRANGE — configurar el mock
      final jsonRespuesta = {
        'count': 2,
        'next': null,
        'previous': null,
        'results': [
          {
            'id': 1, 'name': 'Teclado', 'price': '89.99',
            'category_name': 'Periféricos', 'is_active': true, 'stock': 15,
          },
          {
            'id': 2, 'name': 'Mouse', 'price': '29.99',
            'category_name': 'Periféricos', 'is_active': true, 'stock': 0,
          },
        ],
      };

      when(mockHttp.get(
        '/products/',
        queryParams: anyNamed('queryParams'),
      )).thenAnswer((_) async => Success(jsonRespuesta));

      // ACT — ejecutar la función bajo test
      final resultado = await repo.obtenerProductos();

      // ASSERT — verificar el resultado
      expect(resultado, isA<Success<PaginaProductos>>());
      final pagina = (resultado as Success<PaginaProductos>).data;
      expect(pagina.results.length, equals(2));
      expect(pagina.count, equals(2));
      expect(pagina.hayMas, isFalse);
    });

    test('propaga el error de red sin transformarlo', () async {
      // El mock devuelve un error
      when(mockHttp.get(any, queryParams: anyNamed('queryParams')))
          .thenAnswer((_) async =>
              const Failure(ErrorRed('Sin conexión')));

      final resultado = await repo.obtenerProductos();

      expect(resultado, isA<Failure>());
      final error = (resultado as Failure).error;
      expect(error, isA<ErrorRed>());
    });

    test('pasa los parámetros de búsqueda al cliente HTTP', () async {
      when(mockHttp.get(any, queryParams: anyNamed('queryParams')))
          .thenAnswer((_) async => Success({
                'count': 0, 'next': null, 'previous': null, 'results': [],
              }));

      await repo.obtenerProductos(
        pagina:   2,
        busqueda: 'teclado',
      );

      // Verificar que el mock fue llamado con los parámetros correctos
      verify(mockHttp.get(
        '/products/',
        queryParams: {
          'page':      '2',
          'page_size': '10',
          'search':    'teclado',
        },
      )).called(1);
    });

    test('no incluye parámetro search cuando busqueda está vacía', () async {
      when(mockHttp.get(any, queryParams: anyNamed('queryParams')))
          .thenAnswer((_) async => Success({
                'count': 0, 'next': null, 'previous': null, 'results': [],
              }));

      await repo.obtenerProductos(busqueda: '');

      final capturedCall = verify(mockHttp.get(
        any,
        queryParams: captureAnyNamed('queryParams'),
      )).captured.first as Map<String, String>;

      expect(capturedCall.containsKey('search'), isFalse);
    });
  });
}
```

---

## Parte 3 — Tests de providers con Riverpod

```dart
// test/unit/productos_notifier_test.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import '../mocks/mocks.mocks.dart';

void main() {
  late MockProductosRepository mockRepo;
  late ProviderContainer container;

  // Datos de prueba reutilizables
  final productosEjemplo = [
    const Producto(id: 1, nombre: 'Teclado', precio: 89.99,
        categoria: 'Periféricos', activo: true, stock: 15),
    const Producto(id: 2, nombre: 'Mouse',   precio: 29.99,
        categoria: 'Periféricos', activo: true, stock: 0),
    const Producto(id: 3, nombre: 'Monitor', precio: 349.99,
        categoria: 'Monitores',   activo: false, stock: 5),
  ];

  PaginaProductos paginaConProductos(List<Producto> productos) {
    return PaginaProductos(
      count:    productos.length,
      next:     null,
      previous: null,
      results:  productos.map((p) => ProductoDto(
        id:           p.id,
        name:         p.nombre,
        price:        p.precio.toString(),
        categoryName: p.categoria,
        isActive:     p.activo,
        stock:        p.stock,
      )).toList(),
    );
  }

  setUp(() {
    mockRepo = MockProductosRepository();

    // ProviderContainer permite testear providers sin UI
    container = ProviderContainer(
      overrides: [
        // Reemplazar el repositorio real por el mock
        productosRepositoryProvider.overrideWithValue(mockRepo),
      ],
    );
  });

  // tearDown se ejecuta después de CADA test
  tearDown(() => container.dispose());

  group('ProductosNotifier', () {

    test('carga los productos al inicializar', () async {
      // Configurar mock
      when(mockRepo.obtenerProductos(
        pagina:        anyNamed('pagina'),
        tamanioPagina: anyNamed('tamanioPagina'),
        busqueda:      anyNamed('busqueda'),
        soloActivos:   anyNamed('soloActivos'),
      )).thenAnswer((_) async =>
          Success(paginaConProductos(productosEjemplo)));

      // Leer el provider — dispara build()
      final notifier = container.read(productosProvider.notifier);

      // Esperar a que el estado sea AsyncData
      await container.read(productosProvider.future);

      final estado = container.read(productosProvider).valueOrNull!;
      expect(estado.productos.length, equals(3));
      expect(estado.cargando,         isFalse);
      expect(estado.error,            isNull);
    });

    test('buscar actualiza la lista con los resultados filtrados', () async {
      // Primera carga — todos
      when(mockRepo.obtenerProductos(
        pagina:        anyNamed('pagina'),
        tamanioPagina: anyNamed('tamanioPagina'),
        busqueda:      '',
        soloActivos:   anyNamed('soloActivos'),
      )).thenAnswer((_) async =>
          Success(paginaConProductos(productosEjemplo)));

      // Búsqueda — solo monitores
      when(mockRepo.obtenerProductos(
        pagina:        anyNamed('pagina'),
        tamanioPagina: anyNamed('tamanioPagina'),
        busqueda:      'monitor',
        soloActivos:   anyNamed('soloActivos'),
      )).thenAnswer((_) async => Success(paginaConProductos(
            productosEjemplo.where((p) => p.nombre == 'Monitor').toList())));

      // Esperar carga inicial
      await container.read(productosProvider.future);

      // Ejecutar búsqueda
      await container.read(productosProvider.notifier).buscar('monitor');

      final estado = container.read(productosProvider).valueOrNull!;
      expect(estado.productos.length, equals(1));
      expect(estado.productos.first.nombre, equals('Monitor'));
      expect(estado.busqueda, equals('monitor'));
    });

    test('cargarMas añade productos a la lista existente', () async {
      // Página 1 con hayMas = true
      when(mockRepo.obtenerProductos(
        pagina:        1,
        tamanioPagina: anyNamed('tamanioPagina'),
        busqueda:      anyNamed('busqueda'),
        soloActivos:   anyNamed('soloActivos'),
      )).thenAnswer((_) async => const Success(PaginaProductos(
            count: 5, next: 'url-pag-2', previous: null,
            results: [/* 3 items */])));

      final masProductos = [
        const Producto(id: 4, nombre: 'Auriculares', precio: 149.99,
            categoria: 'Audio', activo: true, stock: 3),
        const Producto(id: 5, nombre: 'Webcam', precio: 59.99,
            categoria: 'Video', activo: true, stock: 12),
      ];

      // Página 2 — más productos
      when(mockRepo.obtenerProductos(
        pagina:        2,
        tamanioPagina: anyNamed('tamanioPagina'),
        busqueda:      anyNamed('busqueda'),
        soloActivos:   anyNamed('soloActivos'),
      )).thenAnswer((_) async =>
          Success(paginaConProductos(masProductos)));

      await container.read(productosProvider.future);

      final antes = container.read(productosProvider).valueOrNull!;
      final cantidadAntes = antes.productos.length;

      await container.read(productosProvider.notifier).cargarMas();

      final despues = container.read(productosProvider).valueOrNull!;
      expect(despues.productos.length, greaterThan(cantidadAntes));
      expect(despues.paginaActual, equals(2));
    });

    test('error de red actualiza el campo error sin borrar los datos', () async {
      // Carga inicial exitosa
      when(mockRepo.obtenerProductos(
        pagina:        1,
        tamanioPagina: anyNamed('tamanioPagina'),
        busqueda:      anyNamed('busqueda'),
        soloActivos:   anyNamed('soloActivos'),
      )).thenAnswer((_) async =>
          Success(paginaConProductos(productosEjemplo)));

      await container.read(productosProvider.future);

      // Configurar el mock para que falle en la siguiente llamada
      when(mockRepo.obtenerProductos(
        pagina:        anyNamed('pagina'),
        tamanioPagina: anyNamed('tamanioPagina'),
        busqueda:      anyNamed('busqueda'),
        soloActivos:   anyNamed('soloActivos'),
      )).thenAnswer((_) async =>
          const Failure(ErrorRed('Sin conexión')));

      await container.read(productosProvider.notifier).recargar();

      final estado = container.read(productosProvider).valueOrNull!;
      // Los datos anteriores se mantienen
      expect(estado.productos.length, equals(3));
      // Y el error se muestra
      expect(estado.error, isA<ErrorRed>());
    });
  });
}
```

---

## Parte 4 — Widget tests

Los widget tests verifican la UI de un widget aislado:

```dart
// test/widget/tarjeta_producto_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  // Producto de prueba — activo con stock normal
  const productoActivo = Producto(
    id: 1, nombre: 'Teclado Mecánico RGB', precio: 89.99,
    categoria: 'Periféricos', activo: true, stock: 15,
  );

  const productoAgotado = Producto(
    id: 2, nombre: 'Monitor UHD 27"', precio: 349.99,
    categoria: 'Monitores', activo: true, stock: 0,
  );

  const productoInactivo = Producto(
    id: 3, nombre: 'Webcam Antigua', precio: 19.99,
    categoria: 'Video', activo: false, stock: 5,
  );

  // Helper — construye el widget con MaterialApp envolvente
  Widget buildWidget(Producto producto) {
    return MaterialApp(
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: Scaffold(
        body: _TarjetaProducto(producto: producto),
      ),
    );
  }

  group('_TarjetaProducto', () {

    testWidgets('muestra nombre y precio del producto', (tester) async {
      await tester.pumpWidget(buildWidget(productoActivo));

      // Buscar texto en el árbol de widgets
      expect(find.text('Teclado Mecánico RGB'), findsOneWidget);
      expect(find.text('\$89.99'), findsOneWidget);
    });

    testWidgets('muestra badge Inactivo cuando activo es false',
        (tester) async {
      await tester.pumpWidget(buildWidget(productoInactivo));

      expect(find.text('Inactivo'), findsOneWidget);
    });

    testWidgets('NO muestra badge Inactivo para productos activos',
        (tester) async {
      await tester.pumpWidget(buildWidget(productoActivo));

      expect(find.text('Inactivo'), findsNothing);
    });

    testWidgets('muestra nivel de stock correcto', (tester) async {
      await tester.pumpWidget(buildWidget(productoAgotado));

      expect(find.text(contains('Agotado')), findsOneWidget);
    });

    testWidgets('muestra la categoría del producto', (tester) async {
      await tester.pumpWidget(buildWidget(productoActivo));

      expect(find.text('Periféricos'), findsOneWidget);
    });
  });

  // Test de la pantalla completa con Riverpod
  group('PantallaCatalogo', () {

    testWidgets('muestra CircularProgressIndicator al cargar',
        (tester) async {
      // Mock del provider que tarda en cargar
      final container = ProviderContainer(
        overrides: [
          productosRepositoryProvider.overrideWith((ref) {
            final mock = MockProductosRepository();
            when(mock.obtenerProductos(
              pagina:        anyNamed('pagina'),
              tamanioPagina: anyNamed('tamanioPagina'),
              busqueda:      anyNamed('busqueda'),
              soloActivos:   anyNamed('soloActivos'),
            )).thenAnswer((_) async {
              await Future.delayed(const Duration(seconds: 1));
              return const Failure(ErrorRed('timeout'));
            });
            return mock;
          }),
        ],
      );

      await tester.pumpWidget(
        UncontrolledProviderScope(
          container: container,
          child: const MaterialApp(home: PantallaCatalogo()),
        ),
      );

      // pump() sin argumento = un frame
      // pumpAndSettle() espera hasta que no haya animaciones
      expect(find.byType(CircularProgressIndicator), findsOneWidget);

      container.dispose();
    });

    testWidgets('muestra los productos cuando carga exitosamente',
        (tester) async {
      final container = ProviderContainer(
        overrides: [
          productosRepositoryProvider.overrideWith((ref) {
            final mock = MockProductosRepository();
            when(mock.obtenerProductos(
              pagina:        anyNamed('pagina'),
              tamanioPagina: anyNamed('tamanioPagina'),
              busqueda:      anyNamed('busqueda'),
              soloActivos:   anyNamed('soloActivos'),
            )).thenAnswer((_) async => const Success(PaginaProductos(
                  count: 1, next: null, previous: null,
                  results: [
                    ProductoDto(
                      id: 1, name: 'Teclado Test', price: '89.99',
                      categoryName: 'Test', isActive: true, stock: 5,
                    ),
                  ],
                )));
            return mock;
          }),
        ],
      );

      await tester.pumpWidget(
        UncontrolledProviderScope(
          container: container,
          child: const MaterialApp(home: PantallaCatalogo()),
        ),
      );

      // pumpAndSettle espera que todos los futures completen
      await tester.pumpAndSettle();

      expect(find.text('Teclado Test'), findsOneWidget);
      expect(find.text('\$89.99'),      findsOneWidget);

      container.dispose();
    });

    testWidgets('buscador filtra productos', (tester) async {
      // ...similar al anterior pero con interacción
      // enterText simula escribir en el campo de búsqueda
      await tester.pumpWidget(const MaterialApp(
        home: Scaffold(
          body: SearchBar(hintText: 'Buscar...'),
        ),
      ));

      await tester.enterText(find.byType(SearchBar), 'monitor');
      await tester.pumpAndSettle();

      expect(find.text('monitor'), findsOneWidget);
    });
  });
}
```

---

## Parte 5 — Ejecutar los tests

```bash
# Todos los tests
flutter test

# Un archivo específico
flutter test test/unit/producto_test.dart

# Con coverage
flutter test --coverage
genhtml coverage/lcov.info -o coverage/html
open coverage/html/index.html

# Solo tests que coincidan con la descripción
flutter test --name "ProductoDto"

# Tests en modo verbose (detallado)
flutter test --reporter expanded
```

---

## Programa completo — suite de tests del catálogo

```dart
// test/suite_catalogo_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';

void main() {

  // ── 1. Tests del modelo de dominio ─────────────────────────────
  group('Producto — modelo de dominio', () {
    test('precio formateado a 2 decimales', () {
      const p = Producto(id: 1, nombre: 'Test', precio: 89.9,
          categoria: 'C', activo: true, stock: 1);
      expect(p.precio.toStringAsFixed(2), equals('89.90'));
    });

    test('nivelStock cubre todos los rangos', () {
      final casos = {
        0:  'Agotado',
        1:  'Bajo',
        5:  'Bajo',
        6:  'Normal',
        20: 'Normal',
        21: 'Alto',
        99: 'Alto',
      };
      for (final entry in casos.entries) {
        final p = Producto(id: 1, nombre: 'T', precio: 1.0,
            categoria: 'C', activo: true, stock: entry.key);
        expect(p.nivelStock, equals(entry.value),
            reason: 'stock=${entry.key} debe ser ${entry.value}');
      }
    });
  });

  // ── 2. Tests del parser JSON ────────────────────────────────────
  group('ProductoDto.fromJson', () {
    test('roundtrip: fromJson → toJson preserva todos los campos', () {
      const original = {
        'id': 42, 'name': 'Monitor', 'price': '349.99',
        'category_name': 'Monitores', 'is_active': true,
        'stock': 8, 'url_image': 'https://img.test/monitor.jpg',
      };
      final dto      = ProductoDto.fromJson(original);
      final serializado = dto.toJson();

      expect(serializado['id'],            equals(original['id']));
      expect(serializado['name'],          equals(original['name']));
      expect(serializado['price'],         equals(original['price']));
      expect(serializado['category_name'], equals(original['category_name']));
      expect(serializado['is_active'],     equals(original['is_active']));
      expect(serializado['stock'],         equals(original['stock']));
      expect(serializado['url_image'],     equals(original['url_image']));
    });
  });

  // ── 3. Tests del resultado tipado ───────────────────────────────
  group('Result<T>', () {
    test('puede guardarse en lista polimórfica', () {
      final resultados = <Result<int>>[
        const Success(1),
        const Failure(ErrorNotFound()),
        const Success(3),
      ];

      final exitosos = resultados.whereType<Success<int>>().toList();
      final fallidos = resultados.whereType<Failure<int>>().toList();

      expect(exitosos.length, equals(2));
      expect(fallidos.length, equals(1));
    });

    test('switch exhaustivo no requiere default', () {
      const Result<String> res = Success('OK');
      final texto = switch (res) {
        Success(data: final d) => d,
        Failure(error: final e) => e.mensaje,
      };
      expect(texto, equals('OK'));
    });
  });

  // ── 4. Tests del repositorio con mock ───────────────────────────
  group('ProductosRepository', () {
    late MockHttpClient mockHttp;
    late ProductosRepository repo;

    setUp(() {
      mockHttp = MockHttpClient();
      repo     = ProductosRepository(mockHttp);
    });

    test('convierte JSON de la API a PaginaProductos', () async {
      when(mockHttp.get(any, queryParams: anyNamed('queryParams')))
          .thenAnswer((_) async => const Success({
                'count': 1, 'next': null, 'previous': null,
                'results': [{
                  'id': 7, 'name': 'Headset', 'price': '79.99',
                  'category_name': 'Audio', 'is_active': true, 'stock': 20,
                }],
              }));

      final result = await repo.obtenerProductos();
      final pagina = (result as Success<PaginaProductos>).data;

      expect(pagina.results.first.id,   equals(7));
      expect(pagina.results.first.name, equals('Headset'));
    });

    test('llama a la API exactamente una vez por obtenerProductos', () async {
      when(mockHttp.get(any, queryParams: anyNamed('queryParams')))
          .thenAnswer((_) async => const Success({
                'count': 0, 'next': null, 'previous': null, 'results': [],
              }));

      await repo.obtenerProductos();

      verify(mockHttp.get(any, queryParams: anyNamed('queryParams')))
          .called(1);
    });
  });

  // ── 5. Tests del notifier ───────────────────────────────────────
  group('ProductosNotifier', () {
    late MockProductosRepository mockRepo;
    late ProviderContainer container;

    final ejemplos = const [
      Producto(id: 1, nombre: 'Teclado', precio: 89.99,
          categoria: 'Periféricos', activo: true, stock: 10),
    ];

    setUp(() {
      mockRepo  = MockProductosRepository();
      container = ProviderContainer(
        overrides: [
          productosRepositoryProvider.overrideWithValue(mockRepo),
        ],
      );

      when(mockRepo.obtenerProductos(
        pagina:        anyNamed('pagina'),
        tamanioPagina: anyNamed('tamanioPagina'),
        busqueda:      anyNamed('busqueda'),
        soloActivos:   anyNamed('soloActivos'),
      )).thenAnswer((_) async => Success(PaginaProductos(
            count: ejemplos.length, next: null, previous: null,
            results: ejemplos.map((p) => ProductoDto(
              id: p.id, name: p.nombre, price: p.precio.toString(),
              categoryName: p.categoria, isActive: p.activo, stock: p.stock,
            )).toList(),
          )));
    });

    tearDown(() => container.dispose());

    test('estado inicial es AsyncLoading luego AsyncData', () async {
      // El primer acceso devuelve AsyncLoading
      final asyncValue = container.read(productosProvider);
      expect(asyncValue, isA<AsyncLoading>());

      // Tras esperar, es AsyncData
      await container.read(productosProvider.future);
      expect(container.read(productosProvider), isA<AsyncData>());
    });

    test('recargar actualiza los productos', () async {
      await container.read(productosProvider.future);
      await container.read(productosProvider.notifier).recargar();

      // Se llamó al repositorio 2 veces (build + recargar)
      verify(mockRepo.obtenerProductos(
        pagina:        anyNamed('pagina'),
        tamanioPagina: anyNamed('tamanioPagina'),
        busqueda:      anyNamed('busqueda'),
        soloActivos:   anyNamed('soloActivos'),
      )).called(2);
    });
  });
}
```

---

## Ejercicios propuestos

1. **Tests de `AuthNotifier`** — Crea una suite completa para el `AuthNotifier`
   de la página 11. Verifica: estado inicial es `SinSesion`, login con credenciales
   correctas da `Autenticado`, login incorrecto da `ErrorAuth`, logout desde
   `Autenticado` da `SinSesion`. Usa `ProviderContainer` sin mocks
   (la lógica es interna).

2. **Golden tests** — Instala `golden_toolkit` e implementa un golden test
   para `_TarjetaProducto`. Crea capturas de referencia para los tres estados:
   producto activo, agotado e inactivo. Verifica que el diseño no cambia
   involuntariamente con `matchesGoldenFile`.

3. **Test de integración de widgets** — Escribe un `testWidgets` completo para
   `PantallaCatalogo` que: pumpe la pantalla con datos mockeados, verifique
   los productos en pantalla, escriba en el `SearchBar`, verifique que los
   resultados cambian, haga tap en un producto y verifique que navega a detalle.

4. **Coverage al 80%** — Corre `flutter test --coverage` en el proyecto.
   Abre el reporte HTML generado. Identifica las funciones sin cobertura y
   escribe los tests necesarios para llevar el total a mínimo 80%.

---

## Resumen de la página 13

- Los tests en Flutter tienen tres niveles: unit (más rápido, sin UI), widget (un componente aislado) y integration (app completa en emulador).
- `group()` agrupa tests relacionados. `setUp()` y `tearDown()` se ejecutan antes y después de cada test. `expect(actual, matcher)` verifica el resultado.
- `@GenerateMocks([Clase])` + `build_runner` genera `MockClase` automáticamente. `when(mock.metodo()).thenAnswer()` configura el comportamiento del mock.
- `verify(mock.metodo()).called(n)` verifica que el mock fue llamado exactamente `n` veces. `captureAnyNamed()` captura los argumentos para inspeccionarlos.
- `ProviderContainer` permite testear providers sin UI. `overrides:` reemplaza dependencias reales por mocks. Siempre llama `container.dispose()` en `tearDown()`.
- `tester.pumpWidget()` monta el widget. `tester.pumpAndSettle()` espera que todos los futures y animaciones completen. `tester.enterText()` simula escritura.
- `find.text()`, `find.byType()`, `find.byKey()` localizan widgets. `findsOneWidget`, `findsNothing`, `findsWidgets` son los matchers de presencia.
- `UncontrolledProviderScope` permite usar un `ProviderContainer` personalizado en widget tests con Riverpod.

---

> **Siguiente página →** Página 14: Almacenamiento local —
> `SharedPreferences`, `Hive` y persistencia de estado con Riverpod.