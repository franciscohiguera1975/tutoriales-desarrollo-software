# Tutorial Flutter — Página 14
## Módulo 5 · Almacenamiento
### `SharedPreferences`, `Hive` y persistencia de estado con Riverpod

---

## ¿Por qué persistir datos localmente?

```
Sin persistencia                       Con persistencia
────────────────────────────────────   ─────────────────────────────────────────
Tema oscuro se resetea al cerrar       Preferencias sobreviven al reinicio
Historial desaparece en cada sesión   El usuario retoma donde lo dejó
Token de sesión se pierde              El usuario no tiene que volver a iniciar
Datos cargados de red en cada inicio  Caché local → inicio instantáneo
```

Opciones de almacenamiento local en Flutter:

```
SharedPreferences   → preferencias simples: tema, idioma, token, flags booleanos
Hive                → objetos estructurados: historial, caché, colecciones tipadas
SQLite (sqflite)    → relaciones complejas, consultas SQL (no cubierto aquí)
Archivos            → binarios, imágenes, documentos grandes
```

El proyecto de este módulo: **Gestor SSH** con modo oscuro persistente (SharedPreferences) e historial de conexiones (Hive).

---

## Patrones clave

### Patrón 1 — `SharedPreferences.getInstance()` + get/set básico

```dart
// Patrón: obtener instancia y leer/escribir valores primitivos
import 'package:shared_preferences/shared_preferences.dart';

Future<void> ejemploBasico() async {
  // Siempre async — accede al disco
  final prefs = await SharedPreferences.getInstance();

  // ── Escribir ────────────────────────────────────────────────────
  await prefs.setBool('modo_oscuro',   true);
  await prefs.setString('idioma',      'es');
  await prefs.setInt('tamanio_fuente', 18);
  await prefs.setDouble('volumen',     0.75);

  // ── Leer — con valor por defecto ────────────────────────────────
  final modoOscuro = prefs.getBool('modo_oscuro')   ?? false;
  final idioma     = prefs.getString('idioma')       ?? 'es';
  final fuente     = prefs.getInt('tamanio_fuente')  ?? 16;
  final volumen    = prefs.getDouble('volumen')       ?? 1.0;
  final token      = prefs.getString('token');        // null si no existe

  // ── Eliminar ────────────────────────────────────────────────────
  await prefs.remove('token');   // una clave
  await prefs.clear();           // TODO
  final existe = prefs.containsKey('modo_oscuro');
}
```

**Mini-ejercicio ⏱ 5 min:**
Dado el código anterior, añade una lectura de `getStringList('favoritos') ?? []` y escribe una lista `['ubuntu', 'debian', 'alpine']` con `setStringList`. Ejecuta el ejemplo y verifica que se guarda y recupera correctamente.

---

### Patrón 2 — Claves constantes + `getStringList` (historial)

```dart
// Patrón: repositorio de preferencias con claves tipadas
class PreferenciasRepository {
  // Constantes → evitan typos silenciosos
  static const _kModoOscuro = 'modo_oscuro';
  static const _kHistorial  = 'historial_busquedas';
  static const _kToken      = 'token';

  final SharedPreferences _prefs;
  const PreferenciasRepository(this._prefs);

  // ── Getters síncronos (SharedPreferences ya está en memoria) ────
  bool    get modoOscuro => _prefs.getBool(_kModoOscuro)       ?? false;
  String? get token      => _prefs.getString(_kToken);
  bool    get hayToken   => token != null && token!.isNotEmpty;

  List<String> get historial =>
      _prefs.getStringList(_kHistorial) ?? [];

  // ── Setters async ────────────────────────────────────────────────
  Future<void> setModoOscuro(bool v) => _prefs.setBool(_kModoOscuro, v);
  Future<void> guardarToken(String t) => _prefs.setString(_kToken, t);
  Future<void> cerrarSesion()         => _prefs.remove(_kToken);

  Future<void> agregarAlHistorial(String texto) async {
    if (texto.trim().isEmpty) return;

    // Eliminar duplicado + insertar al inicio + limitar a 10
    final actual = historial.where((h) => h != texto).toList();
    actual.insert(0, texto);
    await _prefs.setStringList(_kHistorial, actual.take(10).toList());
  }

  Future<void> limpiarHistorial() => _prefs.remove(_kHistorial);
}
```

**Mini-ejercicio ⏱ 5 min:**
Añade un método `Future<void> eliminarDelHistorial(String texto)` que elimine solo esa entrada del historial y guarde el resultado actualizado.

---

### Patrón 3 — Hive: `@HiveType`, `@HiveField` y `openBox`

```dart
// Patrón: definir un modelo persistible con Hive

// lib/data/models/hive/conexion_reciente.dart
import 'package:hive/hive.dart';

part 'conexion_reciente.g.dart';   // generado por build_runner

@HiveType(typeId: 0)               // typeId único por clase en la app
class ConexionReciente extends HiveObject {
  @HiveField(0) final String   hostname;
  @HiveField(1) final String   ip;
  @HiveField(2) final int      puerto;
  @HiveField(3) final String   usuario;
  @HiveField(4) final DateTime ultimaConexion;
  @HiveField(5) final bool     ssl;

  ConexionReciente({
    required this.hostname,
    required this.ip,
    required this.puerto,
    required this.usuario,
    required this.ultimaConexion,
    required this.ssl,
  });

  @override
  String toString() => '$usuario@$ip:$puerto';
}

// ── Registrar y abrir en main() ───────────────────────────────────
// void main() async {
//   WidgetsFlutterBinding.ensureInitialized();
//   await Hive.initFlutter();
//   Hive.registerAdapter(ConexionRecienteAdapter());   // clase generada
//   await Hive.openBox<ConexionReciente>('conexiones');
//   ...
// }
```

Generar el adaptador:
```bash
dart run build_runner build --delete-conflicting-outputs
# Crea: lib/data/models/hive/conexion_reciente.g.dart
```

**Mini-ejercicio ⏱ 5 min:**
Crea un segundo modelo `ConfiguracionSsh` con `typeId: 1` y los campos: `nombre` (String), `host` (String), `puerto` (int), `favorito` (bool). Añade `@HiveField` a cada campo y corre `build_runner`.

---

### Patrón 4 — `Box<T>`: operaciones CRUD

```dart
// Patrón: add, values, delete, save — operaciones sobre un Box<T>
import 'package:hive/hive.dart';

class ConexionesRepository {
  // box<T>() es síncrono después de openBox() en main()
  Box<ConexionReciente> get _box =>
      Hive.box<ConexionReciente>('conexiones');

  // ── Leer — lista ordenada ────────────────────────────────────────
  List<ConexionReciente> obtenerTodas() {
    return _box.values.toList()
      ..sort((a, b) => b.ultimaConexion.compareTo(a.ultimaConexion));
  }

  // ── Añadir — Hive asigna clave autoincremental ───────────────────
  Future<void> agregar(ConexionReciente conexion) =>
      _box.add(conexion);

  // ── Eliminar — HiveObject.delete() se borra a sí mismo ──────────
  Future<void> eliminar(ConexionReciente conexion) =>
      conexion.delete();

  // ── Actualizar — HiveObject.save() persiste los cambios ─────────
  Future<void> marcarFavorito(ConexionReciente conexion) async {
    // Hive no tiene campos mutables en este ejemplo; usa delete+add
    // o crea el modelo con campo mutable (sin 'final')
  }

  // ── Vaciar el box ────────────────────────────────────────────────
  Future<void> limpiarTodo() => _box.clear();

  // ── Escuchar cambios — Stream de eventos Hive ───────────────────
  Stream<BoxEvent> escucharCambios() => _box.watch();
}
```

**Mini-ejercicio ⏱ 5 min:**
Añade un método `Future<void> registrar(ConexionReciente nueva)` que:
1. Elimine cualquier conexión previa con el mismo `ip` y `puerto`.
2. Agregue `nueva` con `_box.add(nueva)`.
3. Si el box supera 20 elementos, elimine el más antiguo.

---

### Patrón 5 — Provider de SharedPreferences con `overrideWithValue` en `main()`

```dart
// Patrón: inicializar SharedPreferences async antes de runApp()
// y exponerlo vía Riverpod

// lib/providers/preferencias_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';

// Provider "placeholder" — se sobreescribe en main()
final sharedPreferencesProvider =
    Provider<SharedPreferences>((ref) => throw UnimplementedError());

// Repository provider — depende del anterior
final preferenciasRepositoryProvider =
    Provider<PreferenciasRepository>((ref) {
  return PreferenciasRepository(ref.watch(sharedPreferencesProvider));
});

// NotifierProvider reactivo para modo oscuro
class ModoOscuroNotifier extends Notifier<bool> {
  @override
  bool build() =>
      ref.watch(preferenciasRepositoryProvider).modoOscuro;

  Future<void> toggle() async {
    final repo = ref.read(preferenciasRepositoryProvider);
    await repo.setModoOscuro(!state);
    state = !state;
  }
}

final modoOscuroProvider =
    NotifierProvider<ModoOscuroNotifier, bool>(ModoOscuroNotifier.new);

// ── main() — inicializar antes de runApp() ────────────────────────
// void main() async {
//   WidgetsFlutterBinding.ensureInitialized();  // obligatorio antes de async
//   final prefs = await SharedPreferences.getInstance();
//   runApp(ProviderScope(
//     overrides: [sharedPreferencesProvider.overrideWithValue(prefs)],
//     child: const MiApp(),
//   ));
// }
```

**Mini-ejercicio ⏱ 5 min:**
Crea un `HistorialNotifier extends Notifier<List<String>>` que:
- `build()` retorne `ref.watch(preferenciasRepositoryProvider).historial`
- tenga `Future<void> agregar(String texto)` que llame al repo y actualice `state`
- tenga `Future<void> limpiar()` que vacíe el historial

---

### Patrón 6 — Provider de Hive + `box.watch()` reactivo

```dart
// Patrón: NotifierProvider que envuelve un Box<T> de Hive

// lib/providers/conexiones_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

final conexionesRepositoryProvider =
    Provider<ConexionesRepository>((ref) => ConexionesRepository());

class ConexionesNotifier extends Notifier<List<ConexionReciente>> {
  ConexionesRepository get _repo =>
      ref.read(conexionesRepositoryProvider);

  @override
  List<ConexionReciente> build() => _repo.obtenerTodas();

  Future<void> registrar({
    required String hostname,
    required String ip,
    required int    puerto,
    required String usuario,
    required bool   ssl,
  }) async {
    await _repo.registrar(ConexionReciente(
      hostname:       hostname,
      ip:             ip,
      puerto:         puerto,
      usuario:        usuario,
      ultimaConexion: DateTime.now(),
      ssl:            ssl,
    ));
    state = _repo.obtenerTodas();   // notifica a la UI
  }

  Future<void> eliminar(ConexionReciente c) async {
    await _repo.eliminar(c);
    state = _repo.obtenerTodas();
  }

  Future<void> limpiarTodo() async {
    await _repo.limpiarTodo();
    state = [];
  }
}

final conexionesProvider =
    NotifierProvider<ConexionesNotifier, List<ConexionReciente>>(
  ConexionesNotifier.new,
);

// ── StreamProvider reactivo con box.watch() ───────────────────────
// (alternativa a NotifierProvider cuando se quiere reactividad automática)
final conexionesStreamProvider =
    StreamProvider<List<ConexionReciente>>((ref) {
  final repo = ref.watch(conexionesRepositoryProvider);
  return repo.escucharCambios().map((_) => repo.obtenerTodas());
});
```

**Mini-ejercicio ⏱ 5 min:**
En `ConexionesNotifier`, añade `Future<void> reconectar(ConexionReciente c)` que:
1. Elimine la conexión existente.
2. La vuelva a agregar con `ultimaConexion: DateTime.now()`.
3. Actualice `state`.

---

## Crea el proyecto

```bash
flutter create modulo14_storage --empty
cd modulo14_storage
```

**`pubspec.yaml`:**
```yaml
name: modulo14_storage
description: Almacenamiento local — módulo 14

environment:
  sdk: '>=3.4.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.6.1
  shared_preferences: ^2.3.4
  hive_flutter:       ^1.1.0      # incluye hive + path_provider

dev_dependencies:
  flutter_test:
    sdk: flutter
  hive_generator: ^2.0.1
  build_runner:   ^2.4.9
```

```bash
flutter pub get
```

**Estructura del proyecto:**
```
modulo14_storage/
├── lib/
│   ├── main.dart
│   ├── data/
│   │   ├── models/
│   │   │   └── hive/
│   │   │       ├── conexion_reciente.dart      <- @HiveType(typeId: 0)
│   │   │       └── conexion_reciente.g.dart    <- generado
│   │   └── repositories/
│   │       ├── preferencias_repository.dart    <- SharedPreferences
│   │       └── conexiones_repository.dart      <- Hive Box<T>
│   ├── providers/
│   │   ├── preferencias_providers.dart         <- sharedPrefs + modoOscuro
│   │   └── conexiones_providers.dart           <- conexionesProvider
│   └── screens/
│       └── pantalla_inicio.dart                <- UI principal
```

> **Selector de pasos** — cambia `const int paso = N` en `main.dart`:
> ```bash
> # Paso 1: SharedPreferences básico — modo oscuro persistente
> # Paso 2: SharedPreferences — historial de búsquedas
> # Paso 3: Hive — modelo y box
> # Paso 4: Hive — NotifierProvider + CRUD
> # Paso 5: App integrada — modo oscuro + historial conexiones
> ```

---

## Paso 1 — SharedPreferences: modo oscuro persistente

**`lib/data/repositories/preferencias_repository.dart`:**
```dart
// lib/data/repositories/preferencias_repository.dart
import 'package:shared_preferences/shared_preferences.dart';

class PreferenciasRepository {
  static const _kModoOscuro = 'modo_oscuro';

  final SharedPreferences _prefs;
  const PreferenciasRepository(this._prefs);

  bool get modoOscuro => _prefs.getBool(_kModoOscuro) ?? false;

  Future<void> setModoOscuro(bool valor) =>
      _prefs.setBool(_kModoOscuro, valor);
}
```

**`lib/providers/preferencias_providers.dart` (Paso 1):**
```dart
// lib/providers/preferencias_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';
import '../data/repositories/preferencias_repository.dart';

final sharedPreferencesProvider =
    Provider<SharedPreferences>((ref) => throw UnimplementedError());

final preferenciasRepositoryProvider =
    Provider<PreferenciasRepository>((ref) =>
        PreferenciasRepository(ref.watch(sharedPreferencesProvider)));

class ModoOscuroNotifier extends Notifier<bool> {
  @override
  bool build() =>
      ref.watch(preferenciasRepositoryProvider).modoOscuro;

  Future<void> toggle() async {
    await ref.read(preferenciasRepositoryProvider).setModoOscuro(!state);
    state = !state;
  }
}

final modoOscuroProvider =
    NotifierProvider<ModoOscuroNotifier, bool>(ModoOscuroNotifier.new);
```

**`lib/main.dart` (Paso 1):**
```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'providers/preferencias_providers.dart';

const int paso = 1;

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final prefs = await SharedPreferences.getInstance();

  runApp(ProviderScope(
    overrides: [sharedPreferencesProvider.overrideWithValue(prefs)],
    child: const AppGestorSsh(),
  ));
}

class AppGestorSsh extends ConsumerWidget {
  const AppGestorSsh({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final modoOscuro = ref.watch(modoOscuroProvider);

    return MaterialApp(
      title: 'SSH Manager',
      themeMode: modoOscuro ? ThemeMode.dark : ThemeMode.light,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF1B5E20),
          brightness: Brightness.light,
        ),
        useMaterial3: true,
      ),
      darkTheme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF1B5E20),
          brightness: Brightness.dark,
        ),
        useMaterial3: true,
      ),
      home: const _PasoPantalla(),
    );
  }
}

class _PasoPantalla extends ConsumerWidget {
  const _PasoPantalla();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final modoOscuro = ref.watch(modoOscuroProvider);
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: const Text('SSH Manager — Paso 1'),
        actions: [
          IconButton(
            icon: Icon(modoOscuro ? Icons.light_mode : Icons.dark_mode),
            onPressed: () => ref.read(modoOscuroProvider.notifier).toggle(),
            tooltip: modoOscuro ? 'Modo claro' : 'Modo oscuro',
          ),
        ],
      ),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(
              modoOscuro ? Icons.nights_stay : Icons.wb_sunny,
              size: 80,
              color: cs.primary,
            ),
            const SizedBox(height: 24),
            Text(
              modoOscuro ? 'Modo oscuro activado' : 'Modo claro activado',
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            const SizedBox(height: 8),
            Text(
              'Preferencia guardada en SharedPreferences',
              style: TextStyle(color: cs.onSurfaceVariant),
            ),
            const SizedBox(height: 32),
            FilledButton.icon(
              onPressed: () =>
                  ref.read(modoOscuroProvider.notifier).toggle(),
              icon: const Icon(Icons.swap_horiz),
              label: const Text('Cambiar tema'),
            ),
          ],
        ),
      ),
    );
  }
}
```

**Prueba esto:**
```bash
flutter run
```
- Pulsa el botón "Cambiar tema" — la app cambia entre claro y oscuro.
- Cierra la app y vuelve a abrirla — el tema persiste.

---

## Paso 2 — SharedPreferences: historial de búsquedas

Actualiza `PreferenciasRepository` añadiendo el historial:

**`lib/data/repositories/preferencias_repository.dart` (actualizado):**
```dart
// lib/data/repositories/preferencias_repository.dart
import 'package:shared_preferences/shared_preferences.dart';

class PreferenciasRepository {
  static const _kModoOscuro = 'modo_oscuro';
  static const _kHistorial  = 'historial_conexiones';

  final SharedPreferences _prefs;
  const PreferenciasRepository(this._prefs);

  // ── Modo oscuro ──────────────────────────────────────────────────
  bool get modoOscuro => _prefs.getBool(_kModoOscuro) ?? false;

  Future<void> setModoOscuro(bool valor) =>
      _prefs.setBool(_kModoOscuro, valor);

  // ── Historial de búsquedas (StringList) ─────────────────────────
  List<String> get historial =>
      _prefs.getStringList(_kHistorial) ?? [];

  Future<void> agregarAlHistorial(String texto) async {
    if (texto.trim().isEmpty) return;
    final actual = historial.where((h) => h != texto).toList();
    actual.insert(0, texto);
    await _prefs.setStringList(_kHistorial, actual.take(10).toList());
  }

  Future<void> limpiarHistorial() => _prefs.remove(_kHistorial);
}
```

Añade el notifier de historial en `preferencias_providers.dart`:

```dart
// Añadir al final de lib/providers/preferencias_providers.dart

class HistorialNotifier extends Notifier<List<String>> {
  PreferenciasRepository get _repo =>
      ref.read(preferenciasRepositoryProvider);

  @override
  List<String> build() =>
      ref.watch(preferenciasRepositoryProvider).historial;

  Future<void> agregar(String texto) async {
    await _repo.agregarAlHistorial(texto);
    state = _repo.historial;
  }

  Future<void> limpiar() async {
    await _repo.limpiarHistorial();
    state = [];
  }
}

final historialProvider =
    NotifierProvider<HistorialNotifier, List<String>>(HistorialNotifier.new);
```

En `main.dart`, cambia `const int paso = 2` y reemplaza `_PasoPantalla` con `_Paso2`:

```dart
// Pantalla del Paso 2 — historial de búsquedas persistente
class _Paso2 extends ConsumerStatefulWidget {
  const _Paso2();

  @override
  ConsumerState<_Paso2> createState() => _Paso2State();
}

class _Paso2State extends ConsumerState<_Paso2> {
  final _ctrl = TextEditingController();

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final historial  = ref.watch(historialProvider);
    final modoOscuro = ref.watch(modoOscuroProvider);
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: const Text('SSH Manager — Paso 2'),
        actions: [
          IconButton(
            icon: Icon(modoOscuro ? Icons.light_mode : Icons.dark_mode),
            onPressed: () => ref.read(modoOscuroProvider.notifier).toggle(),
          ),
        ],
      ),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(16),
            child: Row(
              children: [
                Expanded(
                  child: TextField(
                    controller: _ctrl,
                    decoration: const InputDecoration(
                      labelText: 'Hostname del servidor',
                      border: OutlineInputBorder(),
                      prefixIcon: Icon(Icons.dns),
                    ),
                    onSubmitted: (v) {
                      ref.read(historialProvider.notifier).agregar(v);
                      _ctrl.clear();
                    },
                  ),
                ),
                const SizedBox(width: 8),
                FilledButton(
                  onPressed: () {
                    ref.read(historialProvider.notifier).agregar(_ctrl.text);
                    _ctrl.clear();
                  },
                  child: const Text('Buscar'),
                ),
              ],
            ),
          ),
          if (historial.isNotEmpty) ...[
            Padding(
              padding: const EdgeInsets.fromLTRB(16, 0, 16, 8),
              child: Row(
                children: [
                  Text('Historial',
                      style: Theme.of(context).textTheme.labelLarge
                          ?.copyWith(color: cs.primary)),
                  const Spacer(),
                  TextButton(
                    onPressed: () =>
                        ref.read(historialProvider.notifier).limpiar(),
                    child: const Text('Limpiar'),
                  ),
                ],
              ),
            ),
            Expanded(
              child: ListView.builder(
                itemCount: historial.length,
                itemBuilder: (_, i) => ListTile(
                  leading: const Icon(Icons.history),
                  title:   Text(historial[i]),
                  onTap: () {
                    _ctrl.text = historial[i];
                    ref.read(historialProvider.notifier)
                        .agregar(historial[i]);
                  },
                ),
              ),
            ),
          ] else
            Expanded(
              child: Center(
                child: Text('Sin historial',
                    style: TextStyle(color: cs.onSurfaceVariant)),
              ),
            ),
        ],
      ),
    );
  }
}
```

**Prueba esto:**
```bash
flutter run
```
- Escribe `web-01.empresa.com` y presiona "Buscar".
- Agrega 3 hostnames distintos.
- Cierra la app y vuelve a abrir — el historial persiste.
- Pulsa "Limpiar" y verifica que desaparece.

---

## Paso 3 — Hive: modelo `ConexionReciente` + adaptador

**`lib/data/models/hive/conexion_reciente.dart`:**
```dart
// lib/data/models/hive/conexion_reciente.dart
import 'package:hive/hive.dart';

part 'conexion_reciente.g.dart';

@HiveType(typeId: 0)
class ConexionReciente extends HiveObject {
  @HiveField(0) final String   hostname;
  @HiveField(1) final String   ip;
  @HiveField(2) final int      puerto;
  @HiveField(3) final String   usuario;
  @HiveField(4) final DateTime ultimaConexion;
  @HiveField(5) final bool     ssl;

  ConexionReciente({
    required this.hostname,
    required this.ip,
    required this.puerto,
    required this.usuario,
    required this.ultimaConexion,
    required this.ssl,
  });

  String get direccion => '$usuario@$ip:$puerto';

  String formatearFecha() {
    final diff = DateTime.now().difference(ultimaConexion);
    if (diff.inMinutes < 1) return 'Ahora';
    if (diff.inHours   < 1) return 'Hace ${diff.inMinutes}m';
    if (diff.inDays    < 1) return 'Hace ${diff.inHours}h';
    if (diff.inDays    < 7) return 'Hace ${diff.inDays}d';
    return '${ultimaConexion.day}/${ultimaConexion.month}/${ultimaConexion.year}';
  }
}
```

Genera el adaptador:
```bash
dart run build_runner build --delete-conflicting-outputs
# Crea: lib/data/models/hive/conexion_reciente.g.dart
```

Actualiza `main.dart` para inicializar Hive (cambia `const int paso = 3`):

```dart
// main.dart — Paso 3 (sección void main)
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // ── Hive ─────────────────────────────────────────────────────────
  await Hive.initFlutter();
  Hive.registerAdapter(ConexionRecienteAdapter());
  await Hive.openBox<ConexionReciente>('conexiones');

  // ── SharedPreferences ────────────────────────────────────────────
  final prefs = await SharedPreferences.getInstance();

  runApp(ProviderScope(
    overrides: [sharedPreferencesProvider.overrideWithValue(prefs)],
    child: const AppGestorSsh(),
  ));
}
```

Pantalla del Paso 3 — muestra el box vacío y permite agregar una conexión manual:

```dart
class _Paso3 extends ConsumerStatefulWidget {
  const _Paso3();

  @override
  ConsumerState<_Paso3> createState() => _Paso3State();
}

class _Paso3State extends ConsumerState<_Paso3> {
  List<ConexionReciente> _conexiones = [];

  @override
  void initState() {
    super.initState();
    _cargar();
  }

  void _cargar() {
    final box = Hive.box<ConexionReciente>('conexiones');
    setState(() {
      _conexiones = box.values.toList()
        ..sort((a, b) => b.ultimaConexion.compareTo(a.ultimaConexion));
    });
  }

  Future<void> _agregar() async {
    final box = Hive.box<ConexionReciente>('conexiones');
    await box.add(ConexionReciente(
      hostname:       'web-${box.length + 1}.empresa.com',
      ip:             '10.0.0.${box.length + 1}',
      puerto:         22,
      usuario:        'ubuntu',
      ultimaConexion: DateTime.now(),
      ssl:            true,
    ));
    _cargar();
  }

  Future<void> _limpiar() async {
    await Hive.box<ConexionReciente>('conexiones').clear();
    _cargar();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('SSH Manager — Paso 3 (Hive)'),
        actions: [
          IconButton(
            icon: const Icon(Icons.delete_sweep),
            onPressed: _limpiar,
            tooltip: 'Limpiar box',
          ),
        ],
      ),
      body: _conexiones.isEmpty
          ? const Center(child: Text('Box vacío — pulsa + para agregar'))
          : ListView.builder(
              itemCount: _conexiones.length,
              itemBuilder: (_, i) {
                final c = _conexiones[i];
                return ListTile(
                  leading: const Icon(Icons.terminal),
                  title:   Text(c.hostname),
                  subtitle: Text('${c.direccion} · ${c.formatearFecha()}'),
                  trailing: c.ssl
                      ? const Chip(label: Text('SSL'))
                      : null,
                );
              },
            ),
      floatingActionButton: FloatingActionButton(
        onPressed: _agregar,
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

**Prueba esto:**
```bash
flutter run
```
- Pulsa `+` 3 veces — aparecen 3 conexiones de ejemplo.
- Cierra la app y vuelve a abrir — las conexiones persisten en Hive.
- Pulsa el icono de borrar — el box se vacía.

---

## Paso 4 — Hive: `NotifierProvider` + CRUD completo

**`lib/data/repositories/conexiones_repository.dart`:**
```dart
// lib/data/repositories/conexiones_repository.dart
import 'package:hive/hive.dart';
import '../models/hive/conexion_reciente.dart';

class ConexionesRepository {
  Box<ConexionReciente> get _box =>
      Hive.box<ConexionReciente>('conexiones');

  List<ConexionReciente> obtenerTodas() {
    return _box.values.toList()
      ..sort((a, b) => b.ultimaConexion.compareTo(a.ultimaConexion));
  }

  Future<void> registrar(ConexionReciente nueva) async {
    // Eliminar duplicado (mismo ip:puerto)
    final duplicados = _box.values
        .where((c) => c.ip == nueva.ip && c.puerto == nueva.puerto)
        .toList();
    for (final d in duplicados) await d.delete();

    await _box.add(nueva);

    // Máximo 20 conexiones
    if (_box.length > 20) {
      final todas = _box.values.toList()
        ..sort((a, b) => a.ultimaConexion.compareTo(b.ultimaConexion));
      await todas.first.delete();
    }
  }

  Future<void> eliminar(ConexionReciente c) => c.delete();

  Future<void> limpiarTodo() => _box.clear();

  Stream<BoxEvent> escucharCambios() => _box.watch();
}
```

**`lib/providers/conexiones_providers.dart`:**
```dart
// lib/providers/conexiones_providers.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../data/models/hive/conexion_reciente.dart';
import '../data/repositories/conexiones_repository.dart';

final conexionesRepositoryProvider =
    Provider<ConexionesRepository>((ref) => ConexionesRepository());

class ConexionesNotifier extends Notifier<List<ConexionReciente>> {
  ConexionesRepository get _repo =>
      ref.read(conexionesRepositoryProvider);

  @override
  List<ConexionReciente> build() => _repo.obtenerTodas();

  Future<void> registrar({
    required String hostname,
    required String ip,
    required int    puerto,
    required String usuario,
    required bool   ssl,
  }) async {
    await _repo.registrar(ConexionReciente(
      hostname:       hostname,
      ip:             ip,
      puerto:         puerto,
      usuario:        usuario,
      ultimaConexion: DateTime.now(),
      ssl:            ssl,
    ));
    state = _repo.obtenerTodas();
  }

  Future<void> reconectar(ConexionReciente c) async {
    // Actualiza el timestamp registrando de nuevo (elimina el anterior)
    await registrar(
      hostname: c.hostname,
      ip:       c.ip,
      puerto:   c.puerto,
      usuario:  c.usuario,
      ssl:      c.ssl,
    );
  }

  Future<void> eliminar(ConexionReciente c) async {
    await _repo.eliminar(c);
    state = _repo.obtenerTodas();
  }

  Future<void> limpiarTodo() async {
    await _repo.limpiarTodo();
    state = [];
  }
}

final conexionesProvider =
    NotifierProvider<ConexionesNotifier, List<ConexionReciente>>(
  ConexionesNotifier.new,
);
```

Pantalla Paso 4 — lista de conexiones con swipe-to-delete:

```dart
class _Paso4 extends ConsumerWidget {
  const _Paso4();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final conexiones = ref.watch(conexionesProvider);
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: const Text('SSH Manager — Paso 4'),
        actions: [
          if (conexiones.isNotEmpty)
            IconButton(
              icon: const Icon(Icons.delete_sweep),
              onPressed: () =>
                  ref.read(conexionesProvider.notifier).limpiarTodo(),
              tooltip: 'Limpiar todo',
            ),
        ],
      ),
      body: conexiones.isEmpty
          ? Center(
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Icon(Icons.history, size: 64, color: cs.onSurfaceVariant),
                  const SizedBox(height: 12),
                  Text('Sin conexiones guardadas',
                      style: TextStyle(color: cs.onSurfaceVariant)),
                ],
              ),
            )
          : ListView.builder(
              itemCount: conexiones.length,
              itemBuilder: (_, i) {
                final c = conexiones[i];
                return Dismissible(
                  key: Key('${c.ip}:${c.puerto}'),
                  direction:  DismissDirection.endToStart,
                  background: Container(
                    color:     Colors.red,
                    alignment: Alignment.centerRight,
                    padding:   const EdgeInsets.only(right: 16),
                    child:     const Icon(Icons.delete, color: Colors.white),
                  ),
                  onDismissed: (_) =>
                      ref.read(conexionesProvider.notifier).eliminar(c),
                  child: ListTile(
                    leading: CircleAvatar(
                      backgroundColor: c.ssl
                          ? Colors.green.shade50
                          : Colors.grey.shade100,
                      child: Icon(Icons.terminal,
                          color: c.ssl
                              ? Colors.green.shade700
                              : Colors.grey,
                          size: 20),
                    ),
                    title: Text(c.hostname,
                        style: const TextStyle(fontWeight: FontWeight.w600)),
                    subtitle: Text(
                        '${c.direccion} · ${c.formatearFecha()}',
                        style: TextStyle(
                            fontSize: 12, color: cs.onSurfaceVariant)),
                    trailing: IconButton(
                      icon:      const Icon(Icons.login, size: 20),
                      onPressed: () => ref
                          .read(conexionesProvider.notifier)
                          .reconectar(c),
                      tooltip: 'Reconectar',
                    ),
                  ),
                );
              },
            ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: () => _mostrarFormulario(context, ref),
        icon:  const Icon(Icons.add),
        label: const Text('Nueva conexión'),
      ),
    );
  }

  void _mostrarFormulario(BuildContext context, WidgetRef ref) {
    final ctrlHost    = TextEditingController();
    final ctrlIp      = TextEditingController();
    final ctrlUsuario = TextEditingController(text: 'ubuntu');
    bool ssl          = true;

    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      builder: (ctx) => StatefulBuilder(
        builder: (ctx, set) => Padding(
          padding: EdgeInsets.only(
            left: 16, right: 16, top: 16,
            bottom: MediaQuery.viewInsetsOf(ctx).bottom + 16,
          ),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Text('Nueva conexión SSH',
                  style: Theme.of(ctx).textTheme.titleLarge),
              const SizedBox(height: 16),
              TextField(
                controller: ctrlHost,
                decoration: const InputDecoration(
                  labelText: 'Hostname', border: OutlineInputBorder(),
                  prefixIcon: Icon(Icons.dns),
                ),
              ),
              const SizedBox(height: 12),
              TextField(
                controller: ctrlIp,
                decoration: const InputDecoration(
                  labelText: 'Dirección IP', border: OutlineInputBorder(),
                  prefixIcon: Icon(Icons.router),
                ),
                keyboardType: TextInputType.number,
              ),
              const SizedBox(height: 12),
              TextField(
                controller: ctrlUsuario,
                decoration: const InputDecoration(
                  labelText: 'Usuario', border: OutlineInputBorder(),
                  prefixIcon: Icon(Icons.person_outline),
                ),
              ),
              SwitchListTile(
                title:     const Text('SSL/TLS'),
                value:     ssl,
                onChanged: (v) => set(() => ssl = v),
              ),
              const SizedBox(height: 8),
              SizedBox(
                width: double.infinity,
                child: FilledButton.icon(
                  onPressed: () {
                    if (ctrlHost.text.isEmpty || ctrlIp.text.isEmpty) return;
                    ref.read(conexionesProvider.notifier).registrar(
                      hostname: ctrlHost.text,
                      ip:       ctrlIp.text,
                      puerto:   22,
                      usuario:  ctrlUsuario.text,
                      ssl:      ssl,
                    );
                    Navigator.pop(ctx);
                    ScaffoldMessenger.of(context).showSnackBar(SnackBar(
                      content: Text('${ctrlHost.text} guardado'),
                      behavior: SnackBarBehavior.floating,
                    ));
                  },
                  icon:  const Icon(Icons.save),
                  label: const Text('Guardar conexión'),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

**Prueba esto:**
```bash
flutter run
```
- Añade 3 conexiones con el botón `+`.
- Desliza a la izquierda para eliminar — desaparece al instante.
- Pulsa el ícono de reconectar — sube al inicio con timestamp actualizado.
- Cierra y vuelve a abrir — todo persiste en Hive.

---

## Paso 5 — App integrada: modo oscuro (SharedPrefs) + conexiones (Hive)

App completa con dos secciones en `NavigationBar`:
- **Pestaña 0:** Historial de conexiones SSH (Hive)
- **Pestaña 1:** Preferencias de usuario (SharedPreferences)

```dart
// Añadir al final de lib/providers/preferencias_providers.dart

final tabIndexProvider = StateProvider<int>((ref) => 0);
```

```dart
// Pantalla integrada del Paso 5
class _Paso5 extends ConsumerWidget {
  const _Paso5();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final tabIndex   = ref.watch(tabIndexProvider);
    final modoOscuro = ref.watch(modoOscuroProvider);

    final paginas = [
      const _TabConexiones(),
      const _TabPreferencias(),
    ];

    return Scaffold(
      appBar: AppBar(
        title: const Text('SSH Manager'),
        actions: [
          IconButton(
            icon: Icon(modoOscuro ? Icons.light_mode : Icons.dark_mode),
            onPressed: () => ref.read(modoOscuroProvider.notifier).toggle(),
          ),
        ],
      ),
      body: paginas[tabIndex],
      floatingActionButton: tabIndex == 0
          ? FloatingActionButton.extended(
              onPressed: () => _mostrarFormulario(context, ref),
              icon:  const Icon(Icons.add),
              label: const Text('Nueva conexión'),
            )
          : null,
      bottomNavigationBar: NavigationBar(
        selectedIndex: tabIndex,
        onDestinationSelected: (i) =>
            ref.read(tabIndexProvider.notifier).state = i,
        destinations: const [
          NavigationDestination(
            icon:  Icon(Icons.history),
            label: 'Conexiones',
          ),
          NavigationDestination(
            icon:  Icon(Icons.settings),
            label: 'Preferencias',
          ),
        ],
      ),
    );
  }

  void _mostrarFormulario(BuildContext context, WidgetRef ref) {
    final ctrlHost    = TextEditingController();
    final ctrlIp      = TextEditingController();
    final ctrlUsuario = TextEditingController(text: 'ubuntu');
    bool ssl          = true;

    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      builder: (ctx) => StatefulBuilder(
        builder: (ctx, set) => Padding(
          padding: EdgeInsets.only(
            left: 16, right: 16, top: 16,
            bottom: MediaQuery.viewInsetsOf(ctx).bottom + 16,
          ),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Text('Nueva conexión SSH',
                  style: Theme.of(ctx).textTheme.titleLarge),
              const SizedBox(height: 16),
              TextField(
                controller: ctrlHost,
                decoration: const InputDecoration(
                  labelText: 'Hostname', border: OutlineInputBorder(),
                  prefixIcon: Icon(Icons.dns),
                ),
              ),
              const SizedBox(height: 12),
              TextField(
                controller: ctrlIp,
                decoration: const InputDecoration(
                  labelText: 'Dirección IP', border: OutlineInputBorder(),
                  prefixIcon: Icon(Icons.router),
                ),
                keyboardType: TextInputType.number,
              ),
              const SizedBox(height: 12),
              TextField(
                controller: ctrlUsuario,
                decoration: const InputDecoration(
                  labelText: 'Usuario', border: OutlineInputBorder(),
                  prefixIcon: Icon(Icons.person_outline),
                ),
              ),
              SwitchListTile(
                title:     const Text('SSL/TLS'),
                value:     ssl,
                onChanged: (v) => set(() => ssl = v),
              ),
              const SizedBox(height: 8),
              SizedBox(
                width: double.infinity,
                child: FilledButton.icon(
                  onPressed: () {
                    if (ctrlHost.text.isEmpty || ctrlIp.text.isEmpty) return;

                    ref.read(conexionesProvider.notifier).registrar(
                      hostname: ctrlHost.text,
                      ip:       ctrlIp.text,
                      puerto:   22,
                      usuario:  ctrlUsuario.text,
                      ssl:      ssl,
                    );

                    // También guarda en historial de SharedPreferences
                    ref.read(historialProvider.notifier)
                        .agregar(ctrlHost.text);

                    Navigator.pop(ctx);
                    ScaffoldMessenger.of(context).showSnackBar(SnackBar(
                      content: Text('${ctrlHost.text} guardado'),
                      behavior: SnackBarBehavior.floating,
                    ));
                  },
                  icon:  const Icon(Icons.save),
                  label: const Text('Guardar y conectar'),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

class _TabConexiones extends ConsumerWidget {
  const _TabConexiones();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final conexiones = ref.watch(conexionesProvider);
    final historial  = ref.watch(historialProvider);
    final cs = Theme.of(context).colorScheme;

    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [

        // ── Historial de hostnames (SharedPreferences) ────────────
        if (historial.isNotEmpty) ...[
          Padding(
            padding: const EdgeInsets.fromLTRB(16, 12, 16, 4),
            child: Row(
              children: [
                Text('Búsquedas recientes',
                    style: Theme.of(context).textTheme.labelLarge
                        ?.copyWith(color: cs.primary)),
                const Spacer(),
                TextButton(
                  onPressed: () =>
                      ref.read(historialProvider.notifier).limpiar(),
                  child: const Text('Limpiar'),
                ),
              ],
            ),
          ),
          SizedBox(
            height: 38,
            child: ListView.separated(
              scrollDirection: Axis.horizontal,
              padding:         const EdgeInsets.symmetric(horizontal: 16),
              itemCount:       historial.length,
              separatorBuilder: (_, __) => const SizedBox(width: 8),
              itemBuilder: (_, i) => ActionChip(
                label:  Text(historial[i]),
                avatar: const Icon(Icons.history, size: 14),
                onPressed: () {/* buscar */},
              ),
            ),
          ),
          const SizedBox(height: 4),
        ],

        // ── Conexiones recientes (Hive) ───────────────────────────
        Padding(
          padding: const EdgeInsets.fromLTRB(16, 8, 16, 4),
          child: Text('Conexiones guardadas',
              style: Theme.of(context).textTheme.labelLarge
                  ?.copyWith(color: cs.primary)),
        ),

        Expanded(
          child: conexiones.isEmpty
              ? Center(
                  child: Column(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      Icon(Icons.terminal, size: 64,
                          color: cs.onSurfaceVariant),
                      const SizedBox(height: 12),
                      Text('Sin conexiones guardadas',
                          style: TextStyle(color: cs.onSurfaceVariant)),
                    ],
                  ),
                )
              : ListView.builder(
                  itemCount: conexiones.length,
                  itemBuilder: (_, i) {
                    final c = conexiones[i];
                    return ListTile(
                      leading: CircleAvatar(
                        backgroundColor: c.ssl
                            ? Colors.green.shade50
                            : Colors.grey.shade100,
                        child: Icon(Icons.terminal,
                            color: c.ssl
                                ? Colors.green.shade700
                                : Colors.grey,
                            size: 20),
                      ),
                      title: Text(c.hostname,
                          style: const TextStyle(fontWeight: FontWeight.w600)),
                      subtitle: Text(
                          '${c.direccion} · ${c.formatearFecha()}',
                          style: TextStyle(
                              fontSize: 12, color: cs.onSurfaceVariant)),
                      trailing: Row(
                        mainAxisSize: MainAxisSize.min,
                        children: [
                          IconButton(
                            icon: const Icon(Icons.login, size: 20),
                            onPressed: () => ref
                                .read(conexionesProvider.notifier)
                                .reconectar(c),
                          ),
                          IconButton(
                            icon: const Icon(Icons.delete_outline,
                                size: 20, color: Colors.red),
                            onPressed: () => ref
                                .read(conexionesProvider.notifier)
                                .eliminar(c),
                          ),
                        ],
                      ),
                    );
                  },
                ),
        ),
      ],
    );
  }
}

class _TabPreferencias extends ConsumerWidget {
  const _TabPreferencias();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final modoOscuro = ref.watch(modoOscuroProvider);
    final historial  = ref.watch(historialProvider);
    final conexiones = ref.watch(conexionesProvider);
    final cs = Theme.of(context).colorScheme;

    return ListView(
      padding: const EdgeInsets.all(16),
      children: [
        // ── Apariencia ────────────────────────────────────────────
        Text('Apariencia', style: Theme.of(context).textTheme.labelLarge
            ?.copyWith(color: cs.primary)),
        Card(
          child: SwitchListTile(
            secondary: const Icon(Icons.dark_mode),
            title:     const Text('Modo oscuro'),
            subtitle:  const Text('Guardado en SharedPreferences'),
            value:     modoOscuro,
            onChanged: (_) =>
                ref.read(modoOscuroProvider.notifier).toggle(),
          ),
        ),
        const SizedBox(height: 16),

        // ── Estadísticas ──────────────────────────────────────────
        Text('Estadísticas', style: Theme.of(context).textTheme.labelLarge
            ?.copyWith(color: cs.primary)),
        Card(
          child: Column(
            children: [
              ListTile(
                leading:  const Icon(Icons.dns),
                title:    const Text('Conexiones guardadas'),
                trailing: Chip(label: Text('${conexiones.length}')),
              ),
              ListTile(
                leading:  const Icon(Icons.history),
                title:    const Text('Búsquedas recientes'),
                trailing: Chip(label: Text('${historial.length}')),
              ),
            ],
          ),
        ),
        const SizedBox(height: 16),

        // ── Gestión ───────────────────────────────────────────────
        Text('Gestión de datos', style: Theme.of(context).textTheme.labelLarge
            ?.copyWith(color: cs.primary)),
        Card(
          child: Column(
            children: [
              ListTile(
                leading:  const Icon(Icons.history, color: Colors.orange),
                title:    const Text('Limpiar historial de búsquedas'),
                subtitle: const Text('SharedPreferences'),
                trailing: const Icon(Icons.chevron_right),
                onTap: () =>
                    ref.read(historialProvider.notifier).limpiar(),
              ),
              const Divider(height: 0),
              ListTile(
                leading:  const Icon(Icons.delete_sweep, color: Colors.red),
                title:    const Text('Eliminar todas las conexiones'),
                subtitle: const Text('Hive'),
                trailing: const Icon(Icons.chevron_right),
                onTap: () =>
                    ref.read(conexionesProvider.notifier).limpiarTodo(),
              ),
            ],
          ),
        ),
      ],
    );
  }
}
```

**Prueba esto:**
```bash
flutter run
```
- Agrega 3 conexiones desde la pestaña "Conexiones".
- Cambia a "Preferencias" — activa modo oscuro — toda la app cambia.
- Cierra la app y vuelve — modo oscuro y conexiones persisten.
- Elimina una conexión desde "Preferencias" → "Eliminar todas" — lista vacía.

---

## main.dart completo — referencia (Paso 5)

```dart
// lib/main.dart — versión completa Paso 5
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'data/models/hive/conexion_reciente.dart';
import 'providers/preferencias_providers.dart';
import 'providers/conexiones_providers.dart';

const int paso = 5;

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // ── Hive ─────────────────────────────────────────────────────────
  await Hive.initFlutter();
  Hive.registerAdapter(ConexionRecienteAdapter());
  await Hive.openBox<ConexionReciente>('conexiones');

  // ── SharedPreferences ────────────────────────────────────────────
  final prefs = await SharedPreferences.getInstance();

  runApp(ProviderScope(
    overrides: [sharedPreferencesProvider.overrideWithValue(prefs)],
    child: const AppGestorSsh(),
  ));
}

class AppGestorSsh extends ConsumerWidget {
  const AppGestorSsh({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final modoOscuro = ref.watch(modoOscuroProvider);

    return MaterialApp(
      title:     'SSH Manager',
      themeMode: modoOscuro ? ThemeMode.dark : ThemeMode.light,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF1B5E20),
          brightness: Brightness.light,
        ),
        useMaterial3: true,
      ),
      darkTheme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF1B5E20),
          brightness: Brightness.dark,
        ),
        useMaterial3: true,
      ),
      home: const _Paso5(),
    );
  }
}
```

---

## Proyecto final — AppGestorSsh con NavigationBar + FAB

El proyecto final es la app del Paso 5 con la estructura completa integrada. Resumen de archivos que deben existir al terminar:

```
lib/
├── main.dart                                    <- AppGestorSsh + _Paso5 + _TabConexiones + _TabPreferencias
├── data/
│   ├── models/
│   │   └── hive/
│   │       ├── conexion_reciente.dart           <- @HiveType(typeId: 0), HiveObject
│   │       └── conexion_reciente.g.dart         <- GENERADO por build_runner
│   └── repositories/
│       ├── preferencias_repository.dart         <- SharedPreferences wrapper
│       └── conexiones_repository.dart           <- Hive Box<T> wrapper
└── providers/
    ├── preferencias_providers.dart              <- sharedPreferencesProvider, modoOscuroProvider,
    │                                               historialProvider, tabIndexProvider
    └── conexiones_providers.dart                <- conexionesRepositoryProvider, conexionesProvider
```

Flujo de datos completo de la app:

```
main() async
  ├── Hive.initFlutter()
  ├── Hive.registerAdapter(ConexionRecienteAdapter())
  ├── Hive.openBox<ConexionReciente>('conexiones')
  └── SharedPreferences.getInstance()
        └── ProviderScope(overrides: [sharedPreferencesProvider.overrideWithValue(prefs)])
              └── AppGestorSsh
                    └── MaterialApp (themeMode reactivo via modoOscuroProvider)
                          └── _Paso5
                                ├── _TabConexiones  → ref.watch(conexionesProvider)
                                │                     ref.watch(historialProvider)
                                └── _TabPreferencias → ref.watch(modoOscuroProvider)
                                                       ref.watch(conexionesProvider)
                                                       ref.watch(historialProvider)
```

---

## Guía rápida de imports

```dart
// SharedPreferences
import 'package:shared_preferences/shared_preferences.dart';

// Hive — en archivos de repository y main
import 'package:hive/hive.dart';
import 'package:hive_flutter/hive_flutter.dart';   // initFlutter()

// Hive — en el archivo del modelo (junto a las anotaciones)
import 'package:hive/hive.dart';
part 'conexion_reciente.g.dart';                   // adaptador generado

// Riverpod
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Secuencia de inicialización en main()
// WidgetsFlutterBinding.ensureInitialized();
// await Hive.initFlutter();
// Hive.registerAdapter(ConexionRecienteAdapter());
// await Hive.openBox<ConexionReciente>('conexiones');
// final prefs = await SharedPreferences.getInstance();
// runApp(ProviderScope(overrides: [...], child: ...));
```

---

## Cuándo usar qué

```
Situación                                              Herramienta
───────────────────────────────────────────────────────────────────────────────────
Guardar bool/String/int/double primitivo               SharedPreferences.setBool/setString/...
Guardar lista de strings (historial, tags)             SharedPreferences.setStringList
Guardar objeto Dart complejo (1 campo = @HiveField)    Hive Box<T>
Guardar colección de objetos tipados                   Hive Box<T>.add() / .values
Persistir preferencias de UI (tema, idioma)            SharedPreferences
Persistir historial, caché, favoritos                  Hive
Consultas complejas con JOIN o WHERE                   sqflite (SQLite)
Archivo binario / imagen / PDF                         path_provider + dart:io
Escuchar cambios en tiempo real                        box.watch() → StreamProvider
Reemplazar SharedPreferences en Riverpod tests         Provider.overrideWithValue(FakePrefs)
```

---

## Ejercicios propuestos

**1. Configuraciones SSH favoritas**

Crea un modelo Hive `ConfiguracionSsh` (`typeId: 1`) con los campos: `nombre`, `host`, `puerto`, `usuario`, `clavePath` (String nullable), `favorito` (bool mutable — sin `final`). Implementa la pantalla de configuraciones: `ListView` con las configs, `IconButton` de estrella para `toggleFavorito` usando `config.favorito = !config.favorito; await config.save()`, y un FAB para agregar nuevas. Persiste todo en un box `'configuraciones'`.

**2. Migración de SharedPreferences a Hive**

Crea una función `Future<void> migrarHistorial(SharedPreferences prefs)` que:
1. Lea `getStringList('historial_conexiones') ?? []`.
2. Abra un box Hive `'historial_v2'` y guarde cada string como `HistorialItem(texto: h, fecha: DateTime.now())`.
3. Llame `prefs.remove('historial_conexiones')`.

Llámala en `main()` solo si `prefs.containsKey('historial_conexiones')`.

**3. Box encriptado para tokens**

Usa `HiveAesCipher` para encriptar el box de tokens. Genera la clave con:
```dart
final llave = Hive.generateSecureKey();
```
Guárdala de forma segura con el paquete `flutter_secure_storage`. Al abrir el box, usa:
```dart
await Hive.openBox('tokens',
    encryptionCipher: HiveAesCipher(llave));
```
Guarda y recupera el token de sesión.

**4. `StreamProvider` reactivo con `box.watch()`**

Crea un `StreamProvider<List<ConexionReciente>>` que use `box.watch()` para emitir la lista actualizada automáticamente cada vez que Hive cambia. Conecta este stream a la UI con `ref.watch(conexionesStreamProvider)` y el método `.when()`. La UI debe actualizarse sin necesidad de llamar manualmente a `state = ...` en el notifier.

---

## Resumen

- `SharedPreferences` guarda pares clave-valor de tipos primitivos (`bool`, `String`, `int`, `double`, `List<String>`). Usar para configuraciones simples; no para objetos complejos.
- Las claves de `SharedPreferences` son `String`. Definirlas como `static const` evita typos silenciosos y facilita el renombrado.
- `SharedPreferences` requiere inicialización async en `main()`. Se expone a Riverpod con `Provider.overrideWithValue(prefs)` en el `ProviderScope`.
- `WidgetsFlutterBinding.ensureInitialized()` es obligatorio antes de cualquier operación async en `main()` que use APIs de Flutter (plugins).
- `Hive` almacena objetos Dart tipados en `Box<T>`. Los adaptadores se generan con `@HiveType`/`@HiveField` y `dart run build_runner build`.
- `HiveObject.save()` actualiza un registro directamente en su box. `HiveObject.delete()` lo elimina. No hay que buscar por clave manualmente.
- `Hive.box<T>('nombre')` es síncrono después de `await Hive.openBox<T>('nombre')` en `main()`. Nunca llamar antes de `openBox`.
- `box.watch()` devuelve un `Stream<BoxEvent>` que permite reactividad sin polling — ideal para `StreamProvider`.

---

> **Siguiente página ->** Página 15: Ciclo de vida, permisos y acceso a hardware — cámara, galería y GPS en Flutter.
