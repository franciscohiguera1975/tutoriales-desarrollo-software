# Tutorial Flutter — Página 14
## Módulo 5 · Almacenamiento
### `SharedPreferences`, `Hive` y persistencia de estado con Riverpod

---

## ¿Cuándo usar cada opción?

```
SharedPreferences   → preferencias simples: tema, idioma, token, flags
Hive                → objetos estructurados: historial, caché, colecciones
SQLite (sqflite)    → relaciones complejas, consultas SQL (no cubierto aquí)
Archivos            → binarios, imágenes, documentos grandes
```

---

## Dependencias

```yaml
# pubspec.yaml
dependencies:
  shared_preferences: ^2.3.4
  hive_flutter:       ^1.1.0     # incluye hive + path_provider

dev_dependencies:
  hive_generator:    ^2.0.1      # genera adaptadores de tipo
  build_runner:
```

```bash
# Generar adaptadores Hive
dart run build_runner build --delete-conflicting-outputs
```

---

## Parte 1 — `SharedPreferences`

`SharedPreferences` guarda pares clave-valor de tipos primitivos.
Es el equivalente a `DataStore Preferences` de Android:

```dart
import 'package:shared_preferences/shared_preferences.dart';

// ── Leer ─────────────────────────────────────────────────────────

Future<void> leerPreferencias() async {
  final prefs = await SharedPreferences.getInstance();

  // Métodos de lectura por tipo
  final modoOscuro  = prefs.getBool('modo_oscuro')     ?? false;
  final idioma      = prefs.getString('idioma')         ?? 'es';
  final tamanioFuente = prefs.getInt('tamanio_fuente') ?? 16;
  final volumen     = prefs.getDouble('volumen')        ?? 1.0;
  final etiquetas   = prefs.getStringList('etiquetas')  ?? [];
  final token       = prefs.getString('token');         // null si no existe

  print('Modo oscuro: $modoOscuro');
  print('Idioma: $idioma');
  print('Token: $token');
}

// ── Escribir ──────────────────────────────────────────────────────

Future<void> guardarPreferencias() async {
  final prefs = await SharedPreferences.getInstance();

  await prefs.setBool('modo_oscuro',    true);
  await prefs.setString('idioma',       'en');
  await prefs.setInt('tamanio_fuente',  18);
  await prefs.setDouble('volumen',      0.75);
  await prefs.setStringList('etiquetas', ['dart', 'flutter', 'riverpod']);
}

// ── Eliminar ──────────────────────────────────────────────────────

Future<void> limpiarPreferencias() async {
  final prefs = await SharedPreferences.getInstance();

  await prefs.remove('token');            // eliminar una clave
  await prefs.clear();                    // eliminar TODO

  // Verificar si existe
  final existe = prefs.containsKey('modo_oscuro');
  print('Existe modo_oscuro: $existe');
}
```

---

## Repositorio de preferencias

```dart
// lib/data/preferences/preferencias_repository.dart

class PreferenciasRepository {
  // Claves — constantes para evitar typos
  static const _kModoOscuro    = 'modo_oscuro';
  static const _kIdioma        = 'idioma';
  static const _kToken         = 'token';
  static const _kPrimeraVez    = 'primera_vez';
  static const _kBusquedasRecientes = 'busquedas_recientes';

  final SharedPreferences _prefs;

  const PreferenciasRepository(this._prefs);

  // ── Modo oscuro ──────────────────────────────────────────────
  bool get modoOscuro => _prefs.getBool(_kModoOscuro) ?? false;

  Future<void> setModoOscuro(bool valor) =>
      _prefs.setBool(_kModoOscuro, valor);

  // ── Idioma ───────────────────────────────────────────────────
  String get idioma => _prefs.getString(_kIdioma) ?? 'es';

  Future<void> setIdioma(String codigo) =>
      _prefs.setString(_kIdioma, codigo);

  // ── Sesión ───────────────────────────────────────────────────
  String? get token => _prefs.getString(_kToken);

  Future<void> guardarToken(String token) =>
      _prefs.setString(_kToken, token);

  Future<void> cerrarSesion() => _prefs.remove(_kToken);

  bool get hayToken => token != null && token!.isNotEmpty;

  // ── Primera vez ──────────────────────────────────────────────
  bool get esPrimeraVez => _prefs.getBool(_kPrimeraVez) ?? true;

  Future<void> marcarOnboardingCompletado() =>
      _prefs.setBool(_kPrimeraVez, false);

  // ── Historial de búsquedas ───────────────────────────────────
  List<String> get busquedasRecientes =>
      _prefs.getStringList(_kBusquedasRecientes) ?? [];

  Future<void> agregarBusqueda(String texto) async {
    if (texto.trim().isEmpty) return;

    final recientes = busquedasRecientes
        .where((b) => b != texto)    // eliminar duplicado si existe
        .toList();

    recientes.insert(0, texto);       // añadir al inicio

    final maximo = recientes.take(10).toList();   // máximo 10
    await _prefs.setStringList(_kBusquedasRecientes, maximo);
  }

  Future<void> limpiarHistorial() =>
      _prefs.remove(_kBusquedasRecientes);
}
```

### Provider de SharedPreferences

```dart
// lib/providers/preferences_providers.dart

// SharedPreferences requiere async — se inicializa en main()
// y se pasa como override al ProviderScope
final sharedPreferencesProvider =
    Provider<SharedPreferences>((ref) => throw UnimplementedError());

final preferenciasRepositoryProvider =
    Provider<PreferenciasRepository>((ref) {
  return PreferenciasRepository(ref.watch(sharedPreferencesProvider));
});

// Providers reactivos — leen las preferencias y permiten modificarlas
class ModoOscuroNotifier extends Notifier<bool> {
  PreferenciasRepository get _repo =>
      ref.read(preferenciasRepositoryProvider);

  @override
  bool build() => ref.watch(preferenciasRepositoryProvider).modoOscuro;

  Future<void> toggle() async {
    await _repo.setModoOscuro(!state);
    state = !state;
  }

  Future<void> set(bool valor) async {
    await _repo.setModoOscuro(valor);
    state = valor;
  }
}

final modoOscuroProvider = NotifierProvider<ModoOscuroNotifier, bool>(
  ModoOscuroNotifier.new,
);

class BusquedasNotifier extends Notifier<List<String>> {
  PreferenciasRepository get _repo =>
      ref.read(preferenciasRepositoryProvider);

  @override
  List<String> build() =>
      ref.watch(preferenciasRepositoryProvider).busquedasRecientes;

  Future<void> agregar(String texto) async {
    await _repo.agregarBusqueda(texto);
    state = _repo.busquedasRecientes;
  }

  Future<void> limpiar() async {
    await _repo.limpiarHistorial();
    state = [];
  }
}

final busquedasProvider = NotifierProvider<BusquedasNotifier, List<String>>(
  BusquedasNotifier.new,
);
```

### Inicialización en `main()`

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();  // necesario antes de async

  // Inicializar SharedPreferences antes de crear la app
  final prefs = await SharedPreferences.getInstance();

  runApp(
    ProviderScope(
      overrides: [
        // Inyectar la instancia real
        sharedPreferencesProvider.overrideWithValue(prefs),
      ],
      child: const MiApp(),
    ),
  );
}
```

---

## Parte 2 — Hive

Hive almacena objetos Dart nativos en una base de datos key-value
de alto rendimiento. Es ideal para guardar listas, historial y caché:

### Inicialización

```dart
// main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Inicializar Hive con Flutter (usa path_provider automáticamente)
  await Hive.initFlutter();

  // Registrar adaptadores de tipo antes de abrir boxes
  Hive.registerAdapter(ConexionRecienteAdapter());   // generado por build_runner
  Hive.registerAdapter(ConfiguracionSshAdapter());

  // Abrir boxes (tablas)
  await Hive.openBox<ConexionReciente>('conexiones');
  await Hive.openBox<ConfiguracionSsh>('configuraciones');

  final prefs = await SharedPreferences.getInstance();

  runApp(
    ProviderScope(
      overrides: [sharedPreferencesProvider.overrideWithValue(prefs)],
      child: const MiApp(),
    ),
  );
}
```

### Definir modelos con anotaciones Hive

```dart
// lib/data/models/hive/conexion_reciente.dart
import 'package:hive/hive.dart';

part 'conexion_reciente.g.dart';   // archivo generado por build_runner

@HiveType(typeId: 0)              // typeId único por modelo (0, 1, 2...)
class ConexionReciente extends HiveObject {
  @HiveField(0)
  final String hostname;

  @HiveField(1)
  final String ip;

  @HiveField(2)
  final int puerto;

  @HiveField(3)
  final String usuario;

  @HiveField(4)
  final DateTime ultimaConexion;

  @HiveField(5)
  final bool ssl;

  ConexionReciente({
    required this.hostname,
    required this.ip,
    required this.puerto,
    required this.usuario,
    required this.ultimaConexion,
    required this.ssl,
  });

  @override
  String toString() => '$hostname ($ip:$puerto)';
}

// lib/data/models/hive/configuracion_ssh.dart
@HiveType(typeId: 1)
class ConfiguracionSsh extends HiveObject {
  @HiveField(0)
  final String nombre;

  @HiveField(1)
  final String host;

  @HiveField(2)
  final int puerto;

  @HiveField(3)
  final String usuario;

  @HiveField(4)
  bool favorito;

  @HiveField(5)
  final String? clavePath;     // ruta a la clave privada SSH

  ConfiguracionSsh({
    required this.nombre,
    required this.host,
    required this.puerto,
    required this.usuario,
    this.favorito  = false,
    this.clavePath,
  });
}
```

### Repositorio de Hive

```dart
// lib/data/repositories/conexiones_repository.dart

class ConexionesRepository {
  // Box<T> es la "tabla" de Hive para el tipo T
  Box<ConexionReciente> get _boxConexiones =>
      Hive.box<ConexionReciente>('conexiones');

  Box<ConfiguracionSsh> get _boxConfiguraciones =>
      Hive.box<ConfiguracionSsh>('configuraciones');

  // ── Conexiones recientes ─────────────────────────────────────

  List<ConexionReciente> obtenerConexionesRecientes({int limite = 20}) {
    return _boxConexiones.values
        .toList()
        ..sort((a, b) =>
            b.ultimaConexion.compareTo(a.ultimaConexion));
  }

  Future<void> registrarConexion(ConexionReciente conexion) async {
    // Eliminar si ya existía el mismo host
    final existente = _boxConexiones.values
        .where((c) => c.ip == conexion.ip && c.puerto == conexion.puerto)
        .toList();
    for (final c in existente) await c.delete();

    await _boxConexiones.add(conexion);

    // Mantener solo las últimas 20
    if (_boxConexiones.length > 20) {
      final todas = _boxConexiones.values.toList()
          ..sort((a, b) =>
              a.ultimaConexion.compareTo(b.ultimaConexion));
      await todas.first.delete();
    }
  }

  Future<void> eliminarConexion(ConexionReciente conexion) =>
      conexion.delete();

  Future<void> limpiarConexiones() => _boxConexiones.clear();

  // ── Configuraciones SSH guardadas ────────────────────────────

  List<ConfiguracionSsh> obtenerConfiguraciones() =>
      _boxConfiguraciones.values.toList();

  List<ConfiguracionSsh> obtenerFavoritos() =>
      _boxConfiguraciones.values
          .where((c) => c.favorito)
          .toList();

  Future<void> guardarConfiguracion(ConfiguracionSsh config) async {
    await _boxConfiguraciones.add(config);
  }

  Future<void> toggleFavorito(ConfiguracionSsh config) async {
    config.favorito = !config.favorito;
    await config.save();   // HiveObject.save() actualiza el registro
  }

  Future<void> eliminarConfiguracion(ConfiguracionSsh config) =>
      config.delete();

  // Escuchar cambios — devuelve un Stream de eventos Hive
  Stream<BoxEvent> escucharCambiosConexiones() =>
      _boxConexiones.watch();
}
```

### Providers de Hive

```dart
// Repositorio como singleton en Riverpod
final conexionesRepositoryProvider = Provider<ConexionesRepository>((ref) {
  return ConexionesRepository();
});

// Notifier de conexiones recientes
class ConexionesNotifier extends Notifier<List<ConexionReciente>> {
  ConexionesRepository get _repo =>
      ref.read(conexionesRepositoryProvider);

  @override
  List<ConexionReciente> build() => _repo.obtenerConexionesRecientes();

  Future<void> registrar({
    required String hostname,
    required String ip,
    required int    puerto,
    required String usuario,
    required bool   ssl,
  }) async {
    await _repo.registrarConexion(ConexionReciente(
      hostname:       hostname,
      ip:             ip,
      puerto:         puerto,
      usuario:        usuario,
      ultimaConexion: DateTime.now(),
      ssl:            ssl,
    ));
    state = _repo.obtenerConexionesRecientes();
  }

  Future<void> eliminar(ConexionReciente conexion) async {
    await _repo.eliminarConexion(conexion);
    state = _repo.obtenerConexionesRecientes();
  }

  Future<void> limpiarTodo() async {
    await _repo.limpiarConexiones();
    state = [];
  }
}

final conexionesProvider =
    NotifierProvider<ConexionesNotifier, List<ConexionReciente>>(
  ConexionesNotifier.new,
);

// Notifier de configuraciones
class ConfiguracionesNotifier extends Notifier<List<ConfiguracionSsh>> {
  ConexionesRepository get _repo =>
      ref.read(conexionesRepositoryProvider);

  @override
  List<ConfiguracionSsh> build() => _repo.obtenerConfiguraciones();

  Future<void> agregar(ConfiguracionSsh config) async {
    await _repo.guardarConfiguracion(config);
    state = _repo.obtenerConfiguraciones();
  }

  Future<void> toggleFavorito(ConfiguracionSsh config) async {
    await _repo.toggleFavorito(config);
    state = _repo.obtenerConfiguraciones();
  }

  Future<void> eliminar(ConfiguracionSsh config) async {
    await _repo.eliminarConfiguracion(config);
    state = _repo.obtenerConfiguraciones();
  }
}

final configuracionesProvider =
    NotifierProvider<ConfiguracionesNotifier, List<ConfiguracionSsh>>(
  ConfiguracionesNotifier.new,
);
```

---

## Programa completo — gestor SSH con historial persistente

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:hive_flutter/hive_flutter.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await Hive.initFlutter();
  Hive.registerAdapter(ConexionRecienteAdapter());
  await Hive.openBox<ConexionReciente>('conexiones');

  final prefs = await SharedPreferences.getInstance();

  runApp(ProviderScope(
    overrides: [
      sharedPreferencesProvider.overrideWithValue(prefs),
    ],
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
      theme:     ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor:  const Color(0xFF1B5E20),
          brightness: Brightness.light,
        ),
        useMaterial3: true,
      ),
      darkTheme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor:  const Color(0xFF1B5E20),
          brightness: Brightness.dark,
        ),
        useMaterial3: true,
      ),
      home: const PantallaInicio(),
    );
  }
}

class PantallaInicio extends ConsumerWidget {
  const PantallaInicio({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final conexiones  = ref.watch(conexionesProvider);
    final modoOscuro  = ref.watch(modoOscuroProvider);
    final busquedas   = ref.watch(busquedasProvider);
    final cs          = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: const Text('SSH Manager'),
        actions: [
          // Toggle modo oscuro — persiste en SharedPreferences
          IconButton(
            icon: Icon(modoOscuro ? Icons.light_mode : Icons.dark_mode),
            onPressed: () =>
                ref.read(modoOscuroProvider.notifier).toggle(),
            tooltip: modoOscuro ? 'Modo claro' : 'Modo oscuro',
          ),
        ],
      ),
      body: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [

          // ── Búsquedas recientes (SharedPreferences) ──────────
          if (busquedas.isNotEmpty) ...[
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
                        ref.read(busquedasProvider.notifier).limpiar(),
                    child: const Text('Limpiar'),
                  ),
                ],
              ),
            ),
            SizedBox(
              height: 36,
              child: ListView.separated(
                scrollDirection:  Axis.horizontal,
                padding:          const EdgeInsets.symmetric(horizontal: 16),
                itemCount:        busquedas.length,
                separatorBuilder: (_, __) => const SizedBox(width: 8),
                itemBuilder: (context, i) => ActionChip(
                  label:    Text(busquedas[i]),
                  avatar:   const Icon(Icons.history, size: 14),
                  onPressed: () {/* navegar a búsqueda */},
                ),
              ),
            ),
            const SizedBox(height: 8),
          ],

          // ── Conexiones recientes (Hive) ───────────────────────
          Padding(
            padding: const EdgeInsets.fromLTRB(16, 8, 16, 4),
            child: Row(
              children: [
                Text('Conexiones recientes',
                    style: Theme.of(context).textTheme.labelLarge
                        ?.copyWith(color: cs.primary)),
                const Spacer(),
                if (conexiones.isNotEmpty)
                  TextButton(
                    onPressed: () =>
                        ref.read(conexionesProvider.notifier).limpiarTodo(),
                    child: const Text('Limpiar'),
                  ),
              ],
            ),
          ),

          Expanded(
            child: conexiones.isEmpty
                ? Center(
                    child: Column(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        Icon(Icons.history, size: 56,
                            color: cs.onSurfaceVariant),
                        const SizedBox(height: 12),
                        Text('Sin conexiones recientes',
                            style: TextStyle(color: cs.onSurfaceVariant)),
                      ],
                    ),
                  )
                : ListView.builder(
                    itemCount:   conexiones.length,
                    itemBuilder: (context, i) {
                      final c = conexiones[i];
                      return _ItemConexionReciente(
                        conexion:  c,
                        onEliminar: () => ref
                            .read(conexionesProvider.notifier)
                            .eliminar(c),
                        onConectar: () {
                          // Simular conexión — registra de nuevo para actualizar timestamp
                          ref.read(conexionesProvider.notifier).registrar(
                            hostname: c.hostname,
                            ip:       c.ip,
                            puerto:   c.puerto,
                            usuario:  c.usuario,
                            ssl:      c.ssl,
                          );
                          ScaffoldMessenger.of(context).showSnackBar(
                            SnackBar(
                              content: Text('Conectando a ${c.hostname}...'),
                              behavior: SnackBarBehavior.floating,
                            ),
                          );
                        },
                      );
                    },
                  ),
          ),
        ],
      ),

      floatingActionButton: FloatingActionButton.extended(
        onPressed: () => _mostrarFormularioConexion(context, ref),
        icon:  const Icon(Icons.add),
        label: const Text('Nueva conexión'),
      ),
    );
  }

  void _mostrarFormularioConexion(BuildContext context, WidgetRef ref) {
    final ctrlHost    = TextEditingController();
    final ctrlIp      = TextEditingController();
    final ctrlUsuario = TextEditingController(text: 'ubuntu');
    var   puerto      = 22;
    var   ssl         = true;

    showModalBottomSheet(
      context:   context,
      isScrollControlled: true,
      builder:   (ctx) => StatefulBuilder(
        builder: (ctx, setModalState) => Padding(
          padding: EdgeInsets.only(
            left:   16,
            right:  16,
            top:    16,
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
              const SizedBox(height: 8),
              SwitchListTile(
                title:    const Text('SSL/TLS'),
                value:    ssl,
                onChanged: (v) => setModalState(() => ssl = v),
              ),
              const SizedBox(height: 12),
              SizedBox(
                width: double.infinity,
                child: FilledButton.icon(
                  onPressed: () {
                    if (ctrlHost.text.isEmpty || ctrlIp.text.isEmpty) return;

                    ref.read(conexionesProvider.notifier).registrar(
                      hostname: ctrlHost.text,
                      ip:       ctrlIp.text,
                      puerto:   puerto,
                      usuario:  ctrlUsuario.text,
                      ssl:      ssl,
                    );

                    // Guardar hostname en búsquedas recientes
                    ref.read(busquedasProvider.notifier)
                        .agregar(ctrlHost.text);

                    Navigator.pop(ctx);
                    ScaffoldMessenger.of(context).showSnackBar(
                      SnackBar(
                        content: Text('${ctrlHost.text} guardado'),
                        behavior: SnackBarBehavior.floating,
                      ),
                    );
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

class _ItemConexionReciente extends StatelessWidget {
  final ConexionReciente conexion;
  final VoidCallback     onEliminar;
  final VoidCallback     onConectar;

  const _ItemConexionReciente({
    required this.conexion,
    required this.onEliminar,
    required this.onConectar,
  });

  String _formatearFecha(DateTime dt) {
    final diff = DateTime.now().difference(dt);
    if (diff.inMinutes < 1)  return 'Ahora';
    if (diff.inHours   < 1)  return 'Hace ${diff.inMinutes}m';
    if (diff.inDays    < 1)  return 'Hace ${diff.inHours}h';
    if (diff.inDays    < 7)  return 'Hace ${diff.inDays}d';
    return '${dt.day}/${dt.month}/${dt.year}';
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return ListTile(
      leading: CircleAvatar(
        backgroundColor: conexion.ssl
            ? Colors.green.shade50
            : Colors.grey.shade100,
        child: Icon(
          Icons.terminal,
          color: conexion.ssl ? Colors.green.shade700 : Colors.grey,
          size: 20,
        ),
      ),
      title: Text(
        conexion.hostname,
        style: const TextStyle(fontWeight: FontWeight.w600),
      ),
      subtitle: Text(
        '${conexion.usuario}@${conexion.ip}:${conexion.puerto} · '
        '${_formatearFecha(conexion.ultimaConexion)}',
        style: TextStyle(fontSize: 12, color: cs.onSurfaceVariant),
      ),
      trailing: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          IconButton(
            icon: const Icon(Icons.login, size: 20),
            onPressed: onConectar,
            tooltip: 'Conectar',
          ),
          IconButton(
            icon: const Icon(Icons.delete_outline,
                size: 20, color: Colors.red),
            onPressed: onEliminar,
            tooltip: 'Eliminar',
          ),
        ],
      ),
    );
  }
}
```

---

## Ejercicios propuestos

1. **Configuraciones favoritas** — Implementa la pantalla de configuraciones
   SSH guardadas usando `configuracionesProvider`. Muestra un `ListView`
   con las configuraciones, con un `IconButton` de estrella para `toggleFavorito`
   y un `FloatingActionButton` para agregar nuevas. Persiste todo en Hive.

2. **Migración de SharedPreferences a Hive** — Crea una función
   `migrarPreferencias()` que lea los valores existentes de `SharedPreferences`
   y los copie a un `Box<String>` de Hive, luego llame `prefs.clear()`.
   Llámala en `main()` solo si existe la clave `'necesita_migracion'`.

3. **Box encriptado** — Usa `HiveAesCipher` para encriptar el box de tokens.
   Genera la clave con `Hive.generateSecureKey()` y guárdala en
   `FlutterSecureStorage`. Al abrir el box, usa
   `Hive.openBox('tokens', encryptionCipher: HiveAesCipher(llave))`.

4. **Watchbox reactivo** — Hive permite escuchar cambios con `box.watch()`.
   Crea un `StreamProvider<List<ConexionReciente>>` que use
   `_boxConexiones.watch()` para emitir la lista actualizada cada vez que
   cambia el box. Conecta este stream a la UI con `ref.watch` para que
   se actualice en tiempo real sin llamar a `setState`.

---

## Resumen de la página 14

- `SharedPreferences` guarda primitivos (bool, String, int, double, List\<String\>). Solo usar para configuraciones simples — no para objetos complejos.
- Las claves de `SharedPreferences` son strings. Definirlas como constantes (`static const`) evita typos silenciosos.
- `SharedPreferences` requiere `async` — inicializar en `main()` y pasar como override al `ProviderScope` para que esté disponible de forma síncrona en el resto de la app.
- `Hive` almacena objetos Dart tipados en `Box<T>`. Los adaptadores se generan automáticamente con `@HiveType`/`@HiveField` y `build_runner`.
- `HiveObject.save()` actualiza un registro en su box directamente. `HiveObject.delete()` lo elimina. No hay que buscar y reemplazar manualmente.
- `Hive.box<T>('nombre')` obtiene un box ya abierto de forma síncrona — solo es válido después de `await Hive.openBox<T>('nombre')` en `main()`.
- `box.watch()` devuelve un `Stream<BoxEvent>` — permite hacer la UI reactiva sin polling.
- `WidgetsFlutterBinding.ensureInitialized()` es obligatorio antes de cualquier operación async en `main()` que use APIs de Flutter.

---

> **Siguiente página →** Página 15: Ciclo de vida, permisos y acceso
> a hardware — cámara, galería y GPS en Flutter.