# Tutorial Flutter — Página 13
## Módulo 4 · Calidad
### Testing: `flutter_test`, mocks con Mockito y providers con Riverpod

---

## ¿Por qué testear?

```
Sin tests                          Con tests
──────────────────────             ─────────────────────────────────────
Regressions al cambiar código      Los tests fallan antes del deploy
"Funciona en mi máquina"           Comportamiento verificado y reproducible
Miedo a refactorizar               Refactorización confiada
Bugs en producción                 Bugs detectados en CI/CD
```

Los 3 niveles de test en Flutter:

```
Unit test         → función, clase, provider — sin UI — más rápido
Widget test       → un widget aislado — con render, sin dispositivo real
Integration test  → app completa en emulador — más lento, más fiel
```

Este módulo cubre unit tests y widget tests. Usamos el mismo proyecto ServidorSSH del módulo 09.

---

## Patrones clave

### Patrón 1 — `test()` y `expect()`

```dart
// Anatomía de un test unitario
import 'package:flutter_test/flutter_test.dart';
import 'package:modulo13_tests/models/servidor.dart';

void main() {
  test('descripción clara del comportamiento esperado', () {
    // ARRANGE — preparar datos
    const servidor = ServidorSSH(
      id: 1, nombre: 'Producción', ip: '10.0.0.1',
      puerto: 22, os: 'Ubuntu', activo: true, cargaCpu: 0.95,
    );

    // ACT — ejecutar la función bajo test
    final nivel = servidor.nivelCarga;

    // ASSERT — verificar el resultado
    expect(nivel, equals('Crítica'));        // igualdad exacta
    expect(servidor.activo, isTrue);         // booleano
    expect(servidor.puerto, greaterThan(0)); // comparación
  });
}
```

**Mini-ejercicio ⏱ 5 min:**
Agrega un `test()` que verifique que un servidor con `cargaCpu: 0.0` tiene `nivelCarga == 'Normal'`.
Usa `expect(actual, equals('Normal'))`.

---

### Patrón 2 — `group()`, `setUp()`, `tearDown()`

```dart
void main() {
  // group() agrupa tests relacionados — aparecen con prefijo en la salida
  group('ServidorSSH — nivelCarga', () {

    // setUp() se ejecuta antes de CADA test del grupo
    // tearDown() se ejecuta después de CADA test del grupo
    late ServidorSSH servidor;

    setUp(() {
      servidor = const ServidorSSH(
        id: 1, nombre: 'Test', ip: '127.0.0.1',
        puerto: 22, os: 'Linux', activo: true, cargaCpu: 0.5,
      );
    });

    test('cargaCpu < 0.70 → Normal', () {
      expect(servidor.copyWith(cargaCpu: 0.69).nivelCarga, equals('Normal'));
    });

    test('cargaCpu >= 0.70 y < 0.90 → Alta', () {
      expect(servidor.copyWith(cargaCpu: 0.80).nivelCarga, equals('Alta'));
    });

    test('cargaCpu >= 0.90 → Crítica', () {
      expect(servidor.copyWith(cargaCpu: 0.91).nivelCarga, equals('Crítica'));
    });
  });
}
```

**Mini-ejercicio ⏱ 5 min:**
Añade un `group('ServidorSSH — disponible', ...)` con 2 tests:
- servidor activo con cualquier carga → `disponible` es `true`
- servidor inactivo → `disponible` es `false`

---

### Patrón 3 — `@GenerateMocks` y Mockito básico

```dart
// test/mocks/mocks.dart
import 'package:mockito/annotations.dart';
import 'package:modulo13_tests/repositories/servidores_repository.dart';

@GenerateMocks([ServidoresRepository])
void main() {}
// Genera: mocks.mocks.dart con MockServidoresRepository

// test/unit/repositorio_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import '../mocks/mocks.mocks.dart';

void main() {
  late MockServidoresRepository mockRepo;

  setUp(() => mockRepo = MockServidoresRepository());

  test('when + thenAnswer — mock devuelve lista', () async {
    // Configurar el comportamiento del mock
    when(mockRepo.obtenerServidores())
        .thenAnswer((_) async => [
              const ServidorSSH(id: 1, nombre: 'Web01', ip: '10.0.1.1',
                  puerto: 22, os: 'Ubuntu', activo: true, cargaCpu: 0.3),
            ]);

    // Ejecutar
    final lista = await mockRepo.obtenerServidores();

    // Verificar resultado
    expect(lista.length, equals(1));
    expect(lista.first.nombre, equals('Web01'));

    // Verificar que el mock fue llamado exactamente 1 vez
    verify(mockRepo.obtenerServidores()).called(1);
  });
}
```

**Mini-ejercicio ⏱ 5 min:**
Añade un test que use `when(...).thenThrow(Exception('Sin conexión'))` y verifique con `throwsException` que el repositorio lanza la excepción.

---

### Patrón 4 — `ProviderContainer` + `overrides` (test de providers)

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import '../mocks/mocks.mocks.dart';

void main() {
  late MockServidoresRepository mockRepo;
  late ProviderContainer container;

  setUp(() {
    mockRepo  = MockServidoresRepository();

    // ProviderContainer: permite leer/escribir providers sin UI
    container = ProviderContainer(
      overrides: [
        // Reemplazar la dependencia real por el mock
        servidoresRepositoryProvider.overrideWithValue(mockRepo),
      ],
    );

    // Configurar respuesta por defecto
    when(mockRepo.obtenerServidores()).thenAnswer((_) async => []);
  });

  // Liberar recursos después de cada test
  tearDown(() => container.dispose());

  test('estado inicial es AsyncLoading', () {
    // El primer read() retorna AsyncLoading antes de que build() termine
    final asyncValue = container.read(servidoresProvider);
    expect(asyncValue, isA<AsyncLoading>());
  });

  test('carga servidores y pasa a AsyncData', () async {
    when(mockRepo.obtenerServidores()).thenAnswer((_) async => [
          const ServidorSSH(id: 1, nombre: 'Web01', ip: '10.0.0.1',
              puerto: 22, os: 'Ubuntu', activo: true, cargaCpu: 0.5),
        ]);

    // .future espera a que AsyncNotifier.build() complete
    await container.read(servidoresProvider.future);

    final estado = container.read(servidoresProvider).valueOrNull!;
    expect(estado.servidores.length, equals(1));
    expect(estado.cargando, isFalse);
  });
}
```

**Mini-ejercicio ⏱ 5 min:**
Añade un test que configure el mock para lanzar `Exception('timeout')` y verifique que el estado tiene `error != null` tras llamar al notifier.

---

### Patrón 5 — `testWidgets`, `tester.pumpWidget`, `find`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:modulo13_tests/widgets/tarjeta_servidor.dart';

void main() {
  // Helper — envuelve el widget en MaterialApp para que tenga contexto
  Widget buildWidget(ServidorSSH servidor) {
    return MaterialApp(
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: Scaffold(body: TarjetaServidor(servidor: servidor)),
    );
  }

  const servidorActivo = ServidorSSH(
    id: 1, nombre: 'Producción', ip: '10.0.0.1',
    puerto: 22, os: 'Ubuntu', activo: true, cargaCpu: 0.3,
  );

  testWidgets('muestra nombre y IP del servidor', (tester) async {
    // pumpWidget monta el widget
    await tester.pumpWidget(buildWidget(servidorActivo));

    // find.text() busca texto en el árbol
    expect(find.text('Producción'), findsOneWidget);
    expect(find.text('10.0.0.1'),  findsOneWidget);
  });

  testWidgets('NO muestra badge Inactivo para servidores activos',
      (tester) async {
    await tester.pumpWidget(buildWidget(servidorActivo));
    expect(find.text('Inactivo'), findsNothing);
  });

  testWidgets('simula tap en botón de detalles', (tester) async {
    bool tapped = false;
    await tester.pumpWidget(MaterialApp(
      home: Scaffold(
        body: TarjetaServidor(
          servidor: servidorActivo,
          onTap: () => tapped = true,
        ),
      ),
    ));

    await tester.tap(find.byIcon(Icons.chevron_right));
    await tester.pump(); // procesar el evento de tap

    expect(tapped, isTrue);
  });
}
```

**Mini-ejercicio ⏱ 5 min:**
Agrega un `testWidgets` que verifique que aparece el texto `'Inactivo'` cuando el servidor tiene `activo: false`.

---

### Patrón 6 — Widget test con Riverpod (`UncontrolledProviderScope`)

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import '../mocks/mocks.mocks.dart';

void main() {
  testWidgets('PantallaServidores muestra CircularProgressIndicator al cargar',
      (tester) async {
    final mockRepo = MockServidoresRepository();

    // Mock que tarda — para capturar el estado de carga
    when(mockRepo.obtenerServidores()).thenAnswer((_) async {
      await Future.delayed(const Duration(seconds: 2));
      return [];
    });

    final container = ProviderContainer(
      overrides: [
        servidoresRepositoryProvider.overrideWithValue(mockRepo),
      ],
    );

    // UncontrolledProviderScope usa un container externo
    await tester.pumpWidget(
      UncontrolledProviderScope(
        container: container,
        child: const MaterialApp(home: PantallaServidores()),
      ),
    );

    // pump() sin args = un solo frame — future aún no completó
    expect(find.byType(CircularProgressIndicator), findsOneWidget);

    container.dispose();
  });

  testWidgets('PantallaServidores muestra servidores tras carga exitosa',
      (tester) async {
    final mockRepo = MockServidoresRepository();

    when(mockRepo.obtenerServidores()).thenAnswer((_) async => [
          const ServidorSSH(id: 1, nombre: 'Web-Prod', ip: '10.0.0.1',
              puerto: 22, os: 'Ubuntu', activo: true, cargaCpu: 0.4),
        ]);

    final container = ProviderContainer(
      overrides: [
        servidoresRepositoryProvider.overrideWithValue(mockRepo),
      ],
    );

    await tester.pumpWidget(
      UncontrolledProviderScope(
        container: container,
        child: const MaterialApp(home: PantallaServidores()),
      ),
    );

    // pumpAndSettle espera a que todos los futures y animaciones terminen
    await tester.pumpAndSettle();

    expect(find.text('Web-Prod'), findsOneWidget);
    expect(find.text('10.0.0.1'), findsOneWidget);

    container.dispose();
  });
}
```

**Mini-ejercicio ⏱ 5 min:**
Añade un `testWidgets` que simule que el repositorio lanza un error y verifique que aparece un `Text` con el mensaje de error en la UI.

---

## Crea el proyecto

```bash
flutter create modulo13_tests --empty
cd modulo13_tests
```

**pubspec.yaml** (solo las partes relevantes):

```yaml
name: modulo13_tests
description: Testing en Flutter — módulo 13

environment:
  sdk: '>=3.4.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.6.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  mockito: ^5.4.4
  build_runner: ^2.4.9
```

```bash
flutter pub get
```

**Estructura de carpetas:**

```
modulo13_tests/
├── lib/
│   ├── main.dart
│   ├── models/
│   │   └── servidor.dart          ← modelo ServidorSSH
│   ├── repositories/
│   │   └── servidores_repository.dart ← abstract class + implementación
│   ├── providers/
│   │   └── servidores_provider.dart   ← Riverpod providers
│   ├── screens/
│   │   └── pantalla_servidores.dart   ← pantalla principal
│   └── widgets/
│       └── tarjeta_servidor.dart       ← widget para testear
└── test/
    ├── mocks/
    │   ├── mocks.dart              ← @GenerateMocks
    │   └── mocks.mocks.dart        ← generado por build_runner
    ├── paso1_modelo_test.dart
    ├── paso2_grupo_test.dart
    ├── paso3_mock_test.dart
    ├── paso4_provider_test.dart
    └── paso5_widget_test.dart
```

> **Selector de pasos** — En este módulo cada Paso es un archivo de test. Para ejecutar solo el paso actual:
>
> ```bash
> flutter test test/paso1_modelo_test.dart   # Paso 1
> flutter test test/paso2_grupo_test.dart    # Paso 2
> flutter test                               # Todos los pasos
> ```

---

## Paso 1 — Primer unit test: modelo ServidorSSH

Primero creamos el modelo bajo test:

**`lib/models/servidor.dart`**

```dart
// lib/models/servidor.dart

class ServidorSSH {
  final int    id;
  final String nombre;
  final String ip;
  final int    puerto;
  final String os;
  final bool   activo;
  final double cargaCpu; // 0.0 a 1.0

  const ServidorSSH({
    required this.id,
    required this.nombre,
    required this.ip,
    required this.puerto,
    required this.os,
    required this.activo,
    required this.cargaCpu,
  });

  /// Nivel de carga calculado a partir de cargaCpu
  String get nivelCarga {
    if (cargaCpu >= 0.90) return 'Crítica';
    if (cargaCpu >= 0.70) return 'Alta';
    return 'Normal';
  }

  /// Disponible si el servidor está activo
  bool get disponible => activo;

  /// Dirección SSH completa
  String get direccionSSH => '$nombre@$ip:$puerto';

  ServidorSSH copyWith({
    int?    id,
    String? nombre,
    String? ip,
    int?    puerto,
    String? os,
    bool?   activo,
    double? cargaCpu,
  }) {
    return ServidorSSH(
      id:       id       ?? this.id,
      nombre:   nombre   ?? this.nombre,
      ip:       ip       ?? this.ip,
      puerto:   puerto   ?? this.puerto,
      os:       os       ?? this.os,
      activo:   activo   ?? this.activo,
      cargaCpu: cargaCpu ?? this.cargaCpu,
    );
  }
}
```

Ahora el primer test:

**`test/paso1_modelo_test.dart`**

```dart
// test/paso1_modelo_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:modulo13_tests/models/servidor.dart';

void main() {

  // ── Servidor de referencia para los tests ──────────────────────
  const base = ServidorSSH(
    id:       1,
    nombre:   'Web-Prod',
    ip:       '10.0.0.1',
    puerto:   22,
    os:       'Ubuntu 24.04',
    activo:   true,
    cargaCpu: 0.50,
  );

  // ── test() — prueba individual ─────────────────────────────────
  test('direccionSSH combina nombre, IP y puerto', () {
    expect(base.direccionSSH, equals('Web-Prod@10.0.0.1:22'));
  });

  test('nivelCarga es Normal cuando cargaCpu < 0.70', () {
    expect(base.nivelCarga, equals('Normal'));
  });

  test('nivelCarga es Alta cuando cargaCpu está entre 0.70 y 0.89', () {
    final servidor = base.copyWith(cargaCpu: 0.80);
    expect(servidor.nivelCarga, equals('Alta'));
  });

  test('nivelCarga es Crítica cuando cargaCpu >= 0.90', () {
    final servidor = base.copyWith(cargaCpu: 0.90);
    expect(servidor.nivelCarga, equals('Crítica'));
  });

  test('disponible es true cuando activo es true', () {
    expect(base.disponible, isTrue);
  });

  test('disponible es false cuando activo es false', () {
    final inactivo = base.copyWith(activo: false);
    expect(inactivo.disponible, isFalse);
  });

  test('copyWith cambia solo los campos indicados', () {
    final modificado = base.copyWith(puerto: 2222, cargaCpu: 0.95);

    expect(modificado.puerto,   equals(2222));        // cambiado
    expect(modificado.cargaCpu, equals(0.95));        // cambiado
    expect(modificado.nombre,   equals(base.nombre)); // igual
    expect(modificado.ip,       equals(base.ip));     // igual
  });
}
```

**Prueba esto:**

```bash
flutter test test/paso1_modelo_test.dart --reporter expanded
```

Salida esperada:

```
✓ direccionSSH combina nombre, IP y puerto
✓ nivelCarga es Normal cuando cargaCpu < 0.70
✓ nivelCarga es Alta cuando cargaCpu está entre 0.70 y 0.89
✓ nivelCarga es Crítica cuando cargaCpu >= 0.90
✓ disponible es true cuando activo es true
✓ disponible es false cuando activo es false
✓ copyWith cambia solo los campos indicados

7 tests passed.
```

---

## Paso 2 — `group()` + `setUp()` + `tearDown()`

**`test/paso2_grupo_test.dart`**

```dart
// test/paso2_grupo_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:modulo13_tests/models/servidor.dart';

void main() {

  // ── group() agrupa tests relacionados ──────────────────────────
  group('ServidorSSH — nivelCarga', () {
    // setUp ejecuta ANTES de cada test dentro del group
    late ServidorSSH base;

    setUp(() {
      base = const ServidorSSH(
        id: 1, nombre: 'Test', ip: '127.0.0.1',
        puerto: 22, os: 'Linux', activo: true, cargaCpu: 0.0,
      );
    });

    test('0.00 → Normal', () => expect(
        base.copyWith(cargaCpu: 0.00).nivelCarga, equals('Normal')));

    test('0.69 → Normal', () => expect(
        base.copyWith(cargaCpu: 0.69).nivelCarga, equals('Normal')));

    test('0.70 → Alta', () => expect(
        base.copyWith(cargaCpu: 0.70).nivelCarga, equals('Alta')));

    test('0.89 → Alta', () => expect(
        base.copyWith(cargaCpu: 0.89).nivelCarga, equals('Alta')));

    test('0.90 → Crítica', () => expect(
        base.copyWith(cargaCpu: 0.90).nivelCarga, equals('Crítica')));

    test('1.00 → Crítica', () => expect(
        base.copyWith(cargaCpu: 1.00).nivelCarga, equals('Crítica')));
  });

  group('ServidorSSH — disponible', () {
    test('activo = true  → disponible es true', () {
      const s = ServidorSSH(id: 1, nombre: 'X', ip: '1.1.1.1',
          puerto: 22, os: 'OS', activo: true, cargaCpu: 0.5);
      expect(s.disponible, isTrue);
    });

    test('activo = false → disponible es false', () {
      const s = ServidorSSH(id: 1, nombre: 'X', ip: '1.1.1.1',
          puerto: 22, os: 'OS', activo: false, cargaCpu: 0.0);
      expect(s.disponible, isFalse);
    });
  });

  // ── Tests parametrizados — misma lógica, distintos valores ─────
  group('nivelCarga — tabla de valores límite', () {
    const casos = {
      0.00: 'Normal',
      0.50: 'Normal',
      0.69: 'Normal',
      0.70: 'Alta',
      0.85: 'Alta',
      0.89: 'Alta',
      0.90: 'Crítica',
      0.95: 'Crítica',
      1.00: 'Crítica',
    };

    for (final entry in casos.entries) {
      final carga    = entry.key;
      final esperado = entry.value;

      test('cargaCpu=${carga.toStringAsFixed(2)} → $esperado', () {
        final s = ServidorSSH(
          id: 0, nombre: '', ip: '', puerto: 22,
          os: '', activo: true, cargaCpu: carga,
        );
        expect(s.nivelCarga, equals(esperado));
      });
    }
  });
}
```

**Prueba esto:**

```bash
flutter test test/paso2_grupo_test.dart --reporter expanded
```

Salida esperada:

```
ServidorSSH — nivelCarga
  ✓ 0.00 → Normal
  ✓ 0.69 → Normal
  ✓ 0.70 → Alta
  ✓ 0.89 → Alta
  ✓ 0.90 → Crítica
  ✓ 1.00 → Crítica
ServidorSSH — disponible
  ✓ activo = true  → disponible es true
  ✓ activo = false → disponible es false
nivelCarga — tabla de valores límite
  ✓ cargaCpu=0.00 → Normal
  ✓ cargaCpu=0.50 → Normal
  ...

17 tests passed.
```

---

## Paso 3 — Mocks con Mockito

Primero el repositorio a mockear:

**`lib/repositories/servidores_repository.dart`**

```dart
// lib/repositories/servidores_repository.dart
import '../models/servidor.dart';

// Interfaz abstracta — la implementación real haría http.get()
abstract class ServidoresRepository {
  Future<List<ServidorSSH>> obtenerServidores();
  Future<ServidorSSH>       obtenerDetalle(int id);
  Future<void>               actualizarEstado(int id, bool activo);
}

// Implementación real (conectaría a una API)
class ServidoresRepositoryImpl implements ServidoresRepository {
  @override
  Future<List<ServidorSSH>> obtenerServidores() async {
    // En producción: await http.get(...)
    await Future.delayed(const Duration(milliseconds: 500));
    return [
      const ServidorSSH(id: 1, nombre: 'Web-01', ip: '10.0.0.1',
          puerto: 22, os: 'Ubuntu', activo: true, cargaCpu: 0.35),
    ];
  }

  @override
  Future<ServidorSSH> obtenerDetalle(int id) async {
    throw UnimplementedError();
  }

  @override
  Future<void> actualizarEstado(int id, bool activo) async {
    throw UnimplementedError();
  }
}
```

**`test/mocks/mocks.dart`** — archivo de anotaciones para build_runner:

```dart
// test/mocks/mocks.dart
import 'package:mockito/annotations.dart';
import 'package:modulo13_tests/repositories/servidores_repository.dart';

// Genera MockServidoresRepository en mocks.mocks.dart
@GenerateMocks([ServidoresRepository])
void main() {}
```

```bash
# Genera test/mocks/mocks.mocks.dart
dart run build_runner build --delete-conflicting-outputs
```

**`test/paso3_mock_test.dart`**

```dart
// test/paso3_mock_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:modulo13_tests/models/servidor.dart';
import 'mocks/mocks.mocks.dart';

void main() {
  late MockServidoresRepository mockRepo;

  setUp(() => mockRepo = MockServidoresRepository());

  // ── when + thenAnswer ─────────────────────────────────────────
  group('MockServidoresRepository — when/thenAnswer', () {

    test('obtenerServidores devuelve lista configurada', () async {
      // ARRANGE — configurar el mock
      when(mockRepo.obtenerServidores()).thenAnswer((_) async => [
            const ServidorSSH(id: 1, nombre: 'DB-Main', ip: '10.0.1.1',
                puerto: 5432, os: 'Debian', activo: true, cargaCpu: 0.6),
          ]);

      // ACT
      final lista = await mockRepo.obtenerServidores();

      // ASSERT
      expect(lista.length,         equals(1));
      expect(lista.first.nombre,   equals('DB-Main'));
      expect(lista.first.cargaCpu, greaterThan(0.5));
    });

    test('obtenerDetalle devuelve el servidor con el id correcto', () async {
      const esperado = ServidorSSH(id: 7, nombre: 'Cache-01', ip: '10.0.2.1',
          puerto: 6379, os: 'Alpine', activo: true, cargaCpu: 0.2);

      when(mockRepo.obtenerDetalle(7)).thenAnswer((_) async => esperado);

      final resultado = await mockRepo.obtenerDetalle(7);

      expect(resultado.id,     equals(7));
      expect(resultado.nombre, equals('Cache-01'));
    });
  });

  // ── verify — verificar que el mock fue llamado ─────────────────
  group('verify — verificaciones de llamadas', () {

    test('obtenerServidores es llamado exactamente 1 vez', () async {
      when(mockRepo.obtenerServidores()).thenAnswer((_) async => []);

      await mockRepo.obtenerServidores();

      verify(mockRepo.obtenerServidores()).called(1);
    });

    test('actualizarEstado es llamado con los parámetros correctos',
        () async {
      when(mockRepo.actualizarEstado(any, any)).thenAnswer((_) async {});

      await mockRepo.actualizarEstado(3, false);

      verify(mockRepo.actualizarEstado(3, false)).called(1);
      verifyNever(mockRepo.actualizarEstado(3, true));
    });
  });

  // ── thenThrow — simular errores ────────────────────────────────
  group('thenThrow — simular errores de red', () {

    test('obtenerServidores lanza Exception cuando hay error de red',
        () async {
      when(mockRepo.obtenerServidores())
          .thenThrow(Exception('Sin conexión'));

      expect(
        () async => mockRepo.obtenerServidores(),
        throwsException,
      );
    });

    test('obtenerDetalle lanza cuando el servidor no existe', () async {
      when(mockRepo.obtenerDetalle(999))
          .thenThrow(Exception('404 Not Found'));

      expect(
        () async => mockRepo.obtenerDetalle(999),
        throwsA(isA<Exception>()),
      );
    });
  });
}
```

**Prueba esto:**

```bash
flutter test test/paso3_mock_test.dart --reporter expanded
```

Salida esperada:

```
MockServidoresRepository — when/thenAnswer
  ✓ obtenerServidores devuelve lista configurada
  ✓ obtenerDetalle devuelve el servidor con el id correcto
verify — verificaciones de llamadas
  ✓ obtenerServidores es llamado exactamente 1 vez
  ✓ actualizarEstado es llamado con los parámetros correctos
thenThrow — simular errores de red
  ✓ obtenerServidores lanza Exception cuando hay error de red
  ✓ obtenerDetalle lanza cuando el servidor no existe

6 tests passed.
```

---

## Paso 4 — `ProviderContainer` + `overrides`

**`lib/providers/servidores_provider.dart`**

```dart
// lib/providers/servidores_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/servidor.dart';
import '../repositories/servidores_repository.dart';

// ── Provider del repositorio — reemplazable en tests ──────────────
final servidoresRepositoryProvider = Provider<ServidoresRepository>((ref) {
  return ServidoresRepositoryImpl();
});

// ── Estado del notifier ────────────────────────────────────────────
class ServidoresState {
  final List<ServidorSSH> servidores;
  final bool    cargando;
  final String? error;

  const ServidoresState({
    this.servidores = const [],
    this.cargando   = false,
    this.error,
  });

  ServidoresState copyWith({
    List<ServidorSSH>? servidores,
    bool?   cargando,
    String? error,
  }) {
    return ServidoresState(
      servidores: servidores ?? this.servidores,
      cargando:   cargando   ?? this.cargando,
      error:      error,               // null borra el error anterior
    );
  }
}

// ── AsyncNotifier ─────────────────────────────────────────────────
class ServidoresNotifier extends AsyncNotifier<ServidoresState> {
  @override
  Future<ServidoresState> build() async {
    final repo  = ref.read(servidoresRepositoryProvider);
    final lista = await repo.obtenerServidores();
    return ServidoresState(servidores: lista);
  }

  Future<void> recargar() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() async {
      final repo  = ref.read(servidoresRepositoryProvider);
      final lista = await repo.obtenerServidores();
      return ServidoresState(servidores: lista);
    });
  }

  Future<void> toggleActivo(int id) async {
    final actual = state.valueOrNull;
    if (actual == null) return;

    final idx = actual.servidores.indexWhere((s) => s.id == id);
    if (idx == -1) return;

    final servidorActual = actual.servidores[idx];
    final nuevaLista     = [...actual.servidores];
    nuevaLista[idx]      = servidorActual.copyWith(
      activo: !servidorActual.activo,
    );

    state = AsyncData(actual.copyWith(servidores: nuevaLista));

    // Sincronizar con el repositorio
    await ref.read(servidoresRepositoryProvider)
        .actualizarEstado(id, nuevaLista[idx].activo);
  }
}

final servidoresProvider =
    AsyncNotifierProvider<ServidoresNotifier, ServidoresState>(
  ServidoresNotifier.new,
);
```

**`test/paso4_provider_test.dart`**

```dart
// test/paso4_provider_test.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:modulo13_tests/models/servidor.dart';
import 'package:modulo13_tests/providers/servidores_provider.dart';
import 'mocks/mocks.mocks.dart';

void main() {
  late MockServidoresRepository mockRepo;
  late ProviderContainer container;

  // ── Datos de prueba reutilizables ─────────────────────────────
  final servidoresEjemplo = [
    const ServidorSSH(id: 1, nombre: 'Web-01', ip: '10.0.0.1',
        puerto: 22,   os: 'Ubuntu', activo: true,  cargaCpu: 0.35),
    const ServidorSSH(id: 2, nombre: 'DB-Main', ip: '10.0.0.2',
        puerto: 5432, os: 'Debian', activo: true,  cargaCpu: 0.72),
    const ServidorSSH(id: 3, nombre: 'Cache-01', ip: '10.0.0.3',
        puerto: 6379, os: 'Alpine', activo: false, cargaCpu: 0.10),
  ];

  setUp(() {
    mockRepo = MockServidoresRepository();

    container = ProviderContainer(
      overrides: [
        servidoresRepositoryProvider.overrideWithValue(mockRepo),
      ],
    );

    // Respuesta por defecto
    when(mockRepo.obtenerServidores())
        .thenAnswer((_) async => servidoresEjemplo);
    when(mockRepo.actualizarEstado(any, any))
        .thenAnswer((_) async {});
  });

  tearDown(() => container.dispose());

  // ── Tests del estado inicial ───────────────────────────────────
  group('ServidoresNotifier — estado inicial', () {

    test('estado inicial es AsyncLoading', () {
      final asyncValue = container.read(servidoresProvider);
      expect(asyncValue, isA<AsyncLoading>());
    });

    test('tras build() el estado es AsyncData con los servidores', () async {
      await container.read(servidoresProvider.future);

      final estado = container.read(servidoresProvider).valueOrNull!;
      expect(estado.servidores.length, equals(3));
      expect(estado.cargando,          isFalse);
      expect(estado.error,             isNull);
    });

    test('los servidores tienen los nombres correctos', () async {
      await container.read(servidoresProvider.future);
      final estado = container.read(servidoresProvider).valueOrNull!;

      expect(estado.servidores[0].nombre, equals('Web-01'));
      expect(estado.servidores[1].nombre, equals('DB-Main'));
      expect(estado.servidores[2].nombre, equals('Cache-01'));
    });
  });

  // ── Tests de acciones ─────────────────────────────────────────
  group('ServidoresNotifier — recargar', () {

    test('recargar llama al repositorio por segunda vez', () async {
      await container.read(servidoresProvider.future);
      await container.read(servidoresProvider.notifier).recargar();

      verify(mockRepo.obtenerServidores()).called(2);
    });

    test('tras recargar el estado sigue siendo AsyncData', () async {
      await container.read(servidoresProvider.future);
      await container.read(servidoresProvider.notifier).recargar();

      expect(container.read(servidoresProvider), isA<AsyncData>());
    });
  });

  group('ServidoresNotifier — toggleActivo', () {

    test('toggleActivo invierte el campo activo del servidor', () async {
      await container.read(servidoresProvider.future);

      // El servidor id=1 empieza activo: true
      final antes = container.read(servidoresProvider)
          .valueOrNull!.servidores.firstWhere((s) => s.id == 1);
      expect(antes.activo, isTrue);

      await container.read(servidoresProvider.notifier).toggleActivo(1);

      final despues = container.read(servidoresProvider)
          .valueOrNull!.servidores.firstWhere((s) => s.id == 1);
      expect(despues.activo, isFalse);
    });

    test('toggleActivo llama a actualizarEstado con el nuevo valor',
        () async {
      await container.read(servidoresProvider.future);
      await container.read(servidoresProvider.notifier).toggleActivo(1);

      verify(mockRepo.actualizarEstado(1, false)).called(1);
    });
  });

  // ── Test de error ─────────────────────────────────────────────
  group('ServidoresNotifier — manejo de errores', () {

    test('recargar con error de red pasa a AsyncError', () async {
      // Primera carga exitosa
      await container.read(servidoresProvider.future);

      // La segunda llamada falla
      when(mockRepo.obtenerServidores())
          .thenThrow(Exception('timeout'));

      await container.read(servidoresProvider.notifier).recargar();

      // AsyncValue.guard convierte la excepción en AsyncError
      final asyncVal = container.read(servidoresProvider);
      expect(asyncVal, isA<AsyncError>());
    });
  });
}
```

**Prueba esto:**

```bash
flutter test test/paso4_provider_test.dart --reporter expanded
```

Salida esperada:

```
ServidoresNotifier — estado inicial
  ✓ estado inicial es AsyncLoading
  ✓ tras build() el estado es AsyncData con los servidores
  ✓ los servidores tienen los nombres correctos
ServidoresNotifier — recargar
  ✓ recargar llama al repositorio por segunda vez
  ✓ tras recargar el estado sigue siendo AsyncData
ServidoresNotifier — toggleActivo
  ✓ toggleActivo invierte el campo activo del servidor
  ✓ toggleActivo llama a actualizarEstado con el nuevo valor
ServidoresNotifier — manejo de errores
  ✓ recargar con error de red pasa a AsyncError

8 tests passed.
```

---

## Paso 5 — `testWidgets` + widget tests con Riverpod

**`lib/widgets/tarjeta_servidor.dart`**

```dart
// lib/widgets/tarjeta_servidor.dart
import 'package:flutter/material.dart';
import '../models/servidor.dart';

class TarjetaServidor extends StatelessWidget {
  final ServidorSSH   servidor;
  final VoidCallback? onTap;

  const TarjetaServidor({super.key, required this.servidor, this.onTap});

  @override
  Widget build(BuildContext context) {
    final colorCarga = switch (servidor.nivelCarga) {
      'Crítica' => Colors.red,
      'Alta'    => Colors.orange,
      _         => Colors.green,
    };

    return Card(
      child: ListTile(
        leading: Icon(Icons.computer, color: colorCarga),
        title:   Text(servidor.nombre),
        subtitle: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(servidor.ip),
            Text('Puerto: ${servidor.puerto}'),
          ],
        ),
        trailing: Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            if (!servidor.activo)
              const Chip(label: Text('Inactivo')),
            IconButton(
              icon: const Icon(Icons.chevron_right),
              onPressed: onTap,
            ),
          ],
        ),
      ),
    );
  }
}
```

**`lib/screens/pantalla_servidores.dart`** (widget a testear con Riverpod):

```dart
// lib/screens/pantalla_servidores.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/servidores_provider.dart';
import '../widgets/tarjeta_servidor.dart';

class PantallaServidores extends ConsumerWidget {
  const PantallaServidores({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final asyncServidores = ref.watch(servidoresProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Servidores SSH')),
      body: asyncServidores.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error:   (e, _) => Center(
          child: Text('Error: $e', key: const Key('error-text')),
        ),
        data: (estado) => ListView.builder(
          itemCount: estado.servidores.length,
          itemBuilder: (context, i) => TarjetaServidor(
            servidor: estado.servidores[i],
          ),
        ),
      ),
    );
  }
}
```

**`test/paso5_widget_test.dart`**

```dart
// test/paso5_widget_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:modulo13_tests/models/servidor.dart';
import 'package:modulo13_tests/providers/servidores_provider.dart';
import 'package:modulo13_tests/screens/pantalla_servidores.dart';
import 'package:modulo13_tests/widgets/tarjeta_servidor.dart';
import 'mocks/mocks.mocks.dart';

void main() {

  // ── Helper ────────────────────────────────────────────────────
  Widget buildConProvider({required MockServidoresRepository mockRepo}) {
    final container = ProviderContainer(
      overrides: [
        servidoresRepositoryProvider.overrideWithValue(mockRepo),
      ],
    );

    return UncontrolledProviderScope(
      container: container,
      child: const MaterialApp(home: PantallaServidores()),
    );
  }

  // ── Tests de TarjetaServidor (sin Riverpod) ───────────────────
  group('TarjetaServidor', () {
    const activo = ServidorSSH(id: 1, nombre: 'Web-01', ip: '10.0.0.1',
        puerto: 22, os: 'Ubuntu', activo: true,  cargaCpu: 0.3);
    const inactivo = ServidorSSH(id: 2, nombre: 'Backup', ip: '10.0.0.9',
        puerto: 22, os: 'Debian', activo: false, cargaCpu: 0.0);

    Widget buildTarjeta(ServidorSSH s) => MaterialApp(
          home: Scaffold(body: TarjetaServidor(servidor: s)),
        );

    testWidgets('muestra nombre del servidor', (tester) async {
      await tester.pumpWidget(buildTarjeta(activo));
      expect(find.text('Web-01'), findsOneWidget);
    });

    testWidgets('muestra IP del servidor', (tester) async {
      await tester.pumpWidget(buildTarjeta(activo));
      expect(find.text('10.0.0.1'), findsOneWidget);
    });

    testWidgets('NO muestra chip Inactivo para servidor activo',
        (tester) async {
      await tester.pumpWidget(buildTarjeta(activo));
      expect(find.text('Inactivo'), findsNothing);
    });

    testWidgets('muestra chip Inactivo para servidor inactivo',
        (tester) async {
      await tester.pumpWidget(buildTarjeta(inactivo));
      expect(find.text('Inactivo'), findsOneWidget);
    });

    testWidgets('onTap se dispara al presionar el ícono', (tester) async {
      bool tapped = false;
      await tester.pumpWidget(MaterialApp(
        home: Scaffold(
          body: TarjetaServidor(
            servidor: activo,
            onTap:    () => tapped = true,
          ),
        ),
      ));

      await tester.tap(find.byIcon(Icons.chevron_right));
      await tester.pump();

      expect(tapped, isTrue);
    });
  });

  // ── Tests de PantallaServidores (con Riverpod) ────────────────
  group('PantallaServidores', () {

    testWidgets('muestra CircularProgressIndicator mientras carga',
        (tester) async {
      final mockRepo = MockServidoresRepository();

      when(mockRepo.obtenerServidores()).thenAnswer((_) async {
        await Future.delayed(const Duration(seconds: 2));
        return [];
      });

      await tester.pumpWidget(buildConProvider(mockRepo: mockRepo));

      // Un frame — future aún no completó
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
    });

    testWidgets('muestra los servidores tras carga exitosa',
        (tester) async {
      final mockRepo = MockServidoresRepository();

      when(mockRepo.obtenerServidores()).thenAnswer((_) async => [
            const ServidorSSH(id: 1, nombre: 'Web-Prod', ip: '10.0.0.1',
                puerto: 22,   os: 'Ubuntu', activo: true, cargaCpu: 0.4),
            const ServidorSSH(id: 2, nombre: 'DB-Main',  ip: '10.0.0.2',
                puerto: 5432, os: 'Debian', activo: true, cargaCpu: 0.7),
          ]);

      await tester.pumpWidget(buildConProvider(mockRepo: mockRepo));

      // pumpAndSettle espera a que todos los futures completen
      await tester.pumpAndSettle();

      expect(find.text('Web-Prod'), findsOneWidget);
      expect(find.text('DB-Main'),  findsOneWidget);
      expect(find.byType(TarjetaServidor), findsNWidgets(2));
    });

    testWidgets('muestra mensaje de error cuando la carga falla',
        (tester) async {
      final mockRepo = MockServidoresRepository();

      when(mockRepo.obtenerServidores())
          .thenThrow(Exception('Sin conexión'));

      await tester.pumpWidget(buildConProvider(mockRepo: mockRepo));
      await tester.pumpAndSettle();

      // Verificar que aparece el widget de error
      expect(find.byKey(const Key('error-text')), findsOneWidget);
      expect(find.textContaining('Error'),        findsOneWidget);
    });

    testWidgets('find.byType localiza widgets por tipo', (tester) async {
      final mockRepo = MockServidoresRepository();
      when(mockRepo.obtenerServidores()).thenAnswer((_) async => [
            const ServidorSSH(id: 1, nombre: 'S1', ip: '1.1.1.1',
                puerto: 22, os: 'OS', activo: true,  cargaCpu: 0.5),
            const ServidorSSH(id: 2, nombre: 'S2', ip: '2.2.2.2',
                puerto: 22, os: 'OS', activo: false, cargaCpu: 0.2),
          ]);

      await tester.pumpWidget(buildConProvider(mockRepo: mockRepo));
      await tester.pumpAndSettle();

      // Hay exactamente 2 TarjetaServidor en la pantalla
      expect(find.byType(TarjetaServidor), findsNWidgets(2));
      // Solo 1 chip Inactivo (solo S2 es inactivo)
      expect(find.text('Inactivo'), findsOneWidget);
    });
  });
}
```

**Prueba esto:**

```bash
# Ejecutar solo widgets
flutter test test/paso5_widget_test.dart --reporter expanded

# Ejecutar TODOS los pasos
flutter test --reporter expanded
```

Salida esperada (todos los pasos):

```
TarjetaServidor
  ✓ muestra nombre del servidor
  ✓ muestra IP del servidor
  ✓ NO muestra chip Inactivo para servidor activo
  ✓ muestra chip Inactivo para servidor inactivo
  ✓ onTap se dispara al presionar el ícono
PantallaServidores
  ✓ muestra CircularProgressIndicator mientras carga
  ✓ muestra los servidores tras carga exitosa
  ✓ muestra mensaje de error cuando la carga falla
  ✓ find.byType localiza widgets por tipo

All tests passed.
```

---

## main.dart completo — referencia del proyecto

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'screens/pantalla_servidores.dart';

void main() {
  runApp(
    const ProviderScope(
      child: AppMonitoreo(),
    ),
  );
}

class AppMonitoreo extends StatelessWidget {
  const AppMonitoreo({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Monitor SSH — Tests',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const PantallaServidores(),
    );
  }
}
```

> Los tests se ejecutan con `flutter test`, no con `flutter run`.
> El `main.dart` es la app real; los tests viven todos en `test/`.

---

## Proyecto final — suite completa del Monitor SSH

Suite unificada que cubre todos los niveles:

**`test/suite_monitor_test.dart`**

```dart
// test/suite_monitor_test.dart
// Suite completa — ejecutar con: flutter test test/suite_monitor_test.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:modulo13_tests/models/servidor.dart';
import 'package:modulo13_tests/providers/servidores_provider.dart';
import 'package:modulo13_tests/screens/pantalla_servidores.dart';
import 'package:modulo13_tests/widgets/tarjeta_servidor.dart';
import 'mocks/mocks.mocks.dart';

void main() {

  // ╔══════════════════════════════════════════════════════════════╗
  // ║  NIVEL 1 — Unit tests del modelo                           ║
  // ╚══════════════════════════════════════════════════════════════╝

  group('Modelo ServidorSSH', () {
    const base = ServidorSSH(id: 1, nombre: 'Test', ip: '1.1.1.1',
        puerto: 22, os: 'OS', activo: true, cargaCpu: 0.5);

    test('nivelCarga — tabla de valores límite', () {
      final casos = <double, String>{
        0.00: 'Normal', 0.69: 'Normal',
        0.70: 'Alta',   0.89: 'Alta',
        0.90: 'Crítica', 1.00: 'Crítica',
      };
      for (final e in casos.entries) {
        expect(base.copyWith(cargaCpu: e.key).nivelCarga, equals(e.value),
            reason: 'cargaCpu=${e.key}');
      }
    });

    test('disponible — true solo cuando activo es true', () {
      expect(base.disponible,                        isTrue);
      expect(base.copyWith(activo: false).disponible, isFalse);
    });

    test('direccionSSH — formato correcto', () {
      expect(
        base.copyWith(nombre: 'admin', ip: '10.0.0.1', puerto: 2222)
            .direccionSSH,
        equals('admin@10.0.0.1:2222'),
      );
    });
  });

  // ╔══════════════════════════════════════════════════════════════╗
  // ║  NIVEL 2 — Unit tests del repositorio con mock             ║
  // ╚══════════════════════════════════════════════════════════════╝

  group('MockServidoresRepository', () {
    late MockServidoresRepository mockRepo;

    setUp(() => mockRepo = MockServidoresRepository());

    test('devuelve la lista configurada con when/thenAnswer', () async {
      final esperados = [
        const ServidorSSH(id: 10, nombre: 'LoadBalancer', ip: '10.0.0.10',
            puerto: 80, os: 'HAProxy', activo: true, cargaCpu: 0.55),
      ];
      when(mockRepo.obtenerServidores()).thenAnswer((_) async => esperados);

      final resultado = await mockRepo.obtenerServidores();

      expect(resultado.first.nombre, equals('LoadBalancer'));
      verify(mockRepo.obtenerServidores()).called(1);
    });

    test('lanza excepción con thenThrow', () {
      when(mockRepo.obtenerServidores())
          .thenThrow(Exception('403 Forbidden'));

      expect(() async => mockRepo.obtenerServidores(), throwsException);
    });
  });

  // ╔══════════════════════════════════════════════════════════════╗
  // ║  NIVEL 3 — Tests de providers con ProviderContainer        ║
  // ╚══════════════════════════════════════════════════════════════╝

  group('ServidoresNotifier', () {
    late MockServidoresRepository mockRepo;
    late ProviderContainer container;

    final ejemplos = [
      const ServidorSSH(id: 1, nombre: 'Web-01', ip: '10.0.0.1',
          puerto: 22,   os: 'Ubuntu', activo: true,  cargaCpu: 0.35),
      const ServidorSSH(id: 2, nombre: 'DB-Main', ip: '10.0.0.2',
          puerto: 5432, os: 'Debian', activo: false, cargaCpu: 0.72),
    ];

    setUp(() {
      mockRepo  = MockServidoresRepository();
      container = ProviderContainer(overrides: [
        servidoresRepositoryProvider.overrideWithValue(mockRepo),
      ]);
      when(mockRepo.obtenerServidores()).thenAnswer((_) async => ejemplos);
      when(mockRepo.actualizarEstado(any, any)).thenAnswer((_) async {});
    });

    tearDown(() => container.dispose());

    test('build() carga servidores y pasa a AsyncData', () async {
      await container.read(servidoresProvider.future);
      final estado = container.read(servidoresProvider).valueOrNull!;
      expect(estado.servidores.length, equals(2));
    });

    test('toggleActivo invierte el estado del servidor', () async {
      await container.read(servidoresProvider.future);
      // id=2 empieza inactivo
      await container.read(servidoresProvider.notifier).toggleActivo(2);
      final s2 = container.read(servidoresProvider)
          .valueOrNull!.servidores.firstWhere((s) => s.id == 2);
      expect(s2.activo, isTrue);
    });

    test('recargar vuelve a llamar al repositorio', () async {
      await container.read(servidoresProvider.future);
      await container.read(servidoresProvider.notifier).recargar();
      verify(mockRepo.obtenerServidores()).called(2);
    });
  });

  // ╔══════════════════════════════════════════════════════════════╗
  // ║  NIVEL 4 — Widget tests                                    ║
  // ╚══════════════════════════════════════════════════════════════╝

  group('PantallaServidores', () {
    Widget buildApp(MockServidoresRepository mockRepo) {
      return UncontrolledProviderScope(
        container: ProviderContainer(overrides: [
          servidoresRepositoryProvider.overrideWithValue(mockRepo),
        ]),
        child: const MaterialApp(home: PantallaServidores()),
      );
    }

    testWidgets('estado de carga muestra CircularProgressIndicator',
        (tester) async {
      final mockRepo = MockServidoresRepository();
      when(mockRepo.obtenerServidores()).thenAnswer((_) async {
        await Future.delayed(const Duration(seconds: 5));
        return [];
      });

      await tester.pumpWidget(buildApp(mockRepo));
      expect(find.byType(CircularProgressIndicator), findsOneWidget);
    });

    testWidgets('estado de datos muestra tarjetas de servidores',
        (tester) async {
      final mockRepo = MockServidoresRepository();
      when(mockRepo.obtenerServidores()).thenAnswer((_) async => [
            const ServidorSSH(id: 1, nombre: 'Alpha', ip: '10.0.0.1',
                puerto: 22, os: 'Ubuntu', activo: true, cargaCpu: 0.2),
            const ServidorSSH(id: 2, nombre: 'Beta',  ip: '10.0.0.2',
                puerto: 22, os: 'Debian', activo: true, cargaCpu: 0.8),
          ]);

      await tester.pumpWidget(buildApp(mockRepo));
      await tester.pumpAndSettle();

      expect(find.text('Alpha'), findsOneWidget);
      expect(find.text('Beta'),  findsOneWidget);
      expect(find.byType(TarjetaServidor), findsNWidgets(2));
    });

    testWidgets('estado de error muestra mensaje', (tester) async {
      final mockRepo = MockServidoresRepository();
      when(mockRepo.obtenerServidores())
          .thenThrow(Exception('timeout'));

      await tester.pumpWidget(buildApp(mockRepo));
      await tester.pumpAndSettle();

      expect(find.byKey(const Key('error-text')), findsOneWidget);
    });
  });
}
```

**Ejecutar la suite completa:**

```bash
flutter test test/suite_monitor_test.dart --reporter expanded

# Con reporte de cobertura
flutter test --coverage
genhtml coverage/lcov.info -o coverage/html
# Abrir coverage/html/index.html en el navegador
```

---

## Guía rápida de imports

```dart
// Unit tests + widget tests
import 'package:flutter_test/flutter_test.dart';

// Riverpod en tests (ProviderContainer, UncontrolledProviderScope)
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Mockito (when, verify, thenAnswer, thenThrow)
import 'package:mockito/mockito.dart';

// Anotación para generar mocks
import 'package:mockito/annotations.dart';

// Widget específico siendo testeado
import 'package:mi_app/widgets/tarjeta_servidor.dart';

// Mock generado por build_runner
import 'mocks/mocks.mocks.dart';
```

---

## Cuándo usar qué

```
Situación                                    Herramienta recomendada
──────────────────────────────────────────────────────────────────────────────
Testear cálculos, getters, lógica pura       test() + expect()
Agrupar tests relacionados                   group() + setUp() + tearDown()
Reemplazar dependencias externas (API)       Mockito @GenerateMocks + when/verify
Testear providers sin levantar la UI         ProviderContainer + overrides
Testear que un widget renderiza bien         testWidgets + find + pumpWidget
Testear widget que depende de Riverpod       UncontrolledProviderScope + container
Esperar a que un Future complete en UI       tester.pumpAndSettle()
Simular escritura del usuario                tester.enterText()
Simular tap del usuario                      tester.tap() + tester.pump()
Verificar que el mock fue llamado N veces    verify(mock.metodo()).called(N)
Verificar que el mock NO fue llamado         verifyNever(mock.metodo())
```

---

## Ejercicios propuestos

**1. Tests de `AuthNotifier`**

Crea una suite completa para el `AuthNotifier` de la página 11. Verifica:
- Estado inicial es `SinSesion`.
- Login con credenciales correctas da `Autenticado`.
- Login incorrecto da `ErrorAuth`.
- Logout desde `Autenticado` da `SinSesion`.

Usa `ProviderContainer` sin mocks (la lógica es interna).

**2. Tests parametrizados con tabla**

En `paso2_grupo_test.dart` ya usaste un `Map<double, String>` para cubrir todos los rangos de `nivelCarga`. Ahora haz lo mismo con `direccionSSH`: crea una tabla de 5 combinaciones de nombre/IP/puerto y verifica el formato de salida esperado para cada una.

**3. Golden tests**

Instala `golden_toolkit` e implementa un golden test para `TarjetaServidor`. Crea capturas de referencia para los tres estados: servidor activo con carga Normal, activo con carga Crítica e inactivo. Verifica que el diseño no cambia involuntariamente con `matchesGoldenFile`.

**4. Coverage al 80%**

Corre `flutter test --coverage` en el proyecto. Abre el reporte HTML generado con `genhtml`. Identifica las funciones sin cobertura y escribe los tests necesarios para llevar el total a mínimo 80%.

---

## Resumen

- Los tests en Flutter tienen tres niveles: **unit** (más rápido, sin UI), **widget** (un componente aislado con render) y **integration** (app completa en emulador). Este módulo cubre los dos primeros.
- `test('descripción', () { ... })` define un test. `expect(actual, matcher)` verifica el resultado. Matchers comunes: `equals()`, `isTrue/isFalse`, `isA<Tipo>()`, `findsOneWidget`, `findsNothing`.
- `group()` agrupa tests relacionados bajo un prefijo común en la salida. `setUp()` se ejecuta antes de cada test del grupo; `tearDown()` después. Ambos garantizan aislamiento entre tests.
- `@GenerateMocks([Clase])` + `dart run build_runner build` genera `MockClase` automáticamente. `when(mock.metodo()).thenAnswer(...)` configura la respuesta; `when(...).thenThrow(...)` simula errores.
- `verify(mock.metodo()).called(n)` verifica que el mock fue llamado exactamente `n` veces. `verifyNever(mock.metodo())` verifica que nunca fue llamado.
- `ProviderContainer` permite testear providers sin UI. `overrides: [provider.overrideWithValue(mock)]` reemplaza dependencias reales por mocks. Siempre llama `container.dispose()` en `tearDown()`.
- `testWidgets('...', (tester) async { ... })` monta y prueba widgets. `tester.pumpWidget()` monta el árbol. `tester.pumpAndSettle()` espera futures y animaciones. `tester.tap()` + `tester.enterText()` simulan interacciones del usuario.
- `find.text()`, `find.byType()`, `find.byKey()` localizan widgets en el árbol. `findsOneWidget`, `findsNothing`, `findsNWidgets(n)` son los matchers de presencia.
- `UncontrolledProviderScope(container: container, child: ...)` permite usar un `ProviderContainer` personalizado en widget tests con Riverpod, desacoplando el árbol de providers del árbol de widgets.

---

> **Siguiente página →** Página 14: Almacenamiento local —
> `SharedPreferences`, `Hive` y persistencia de estado con Riverpod.
