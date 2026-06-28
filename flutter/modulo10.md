# Tutorial Flutter — Página 10
## Módulo 3 · Estado
### Riverpod 3.0 — `ProviderScope`, `ConsumerWidget`, providers y `AsyncNotifier`

---

## ¿Por qué Riverpod?

`setState` funciona bien para estado local de un solo widget.
Pero cuando el estado necesita **compartirse entre pantallas** o
**mantenerse cuando el widget se destruye**, `setState` se vuelve
un laberinto de callbacks y prop-drilling.

Riverpod resuelve esto con **providers** — fuentes de estado
declarativas que cualquier widget puede leer y escuchar:

```
setState                     Riverpod
────────────────────         ────────────────────────────────────
Estado vive en el widget     Estado vive en el provider (fuera del árbol)
Prop-drilling entre widgets  Cualquier widget lee el provider directamente
Se destruye con el widget    Sobrevive al widget (configurable)
Difícil de testear           Fácil de testear — inyección limpia
```

### Dependencias

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^3.3.0
```

```bash
flutter pub get
```

---

## Patrones clave — de un vistazo

Referencia rápida de los patrones del módulo. Léelos ahora y vuelve
a consultarlos conforme avanzas en cada paso.

---

### `ProviderScope` y `ConsumerWidget`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() {
  runApp(
    // ProviderScope: contenedor global — SIEMPRE envuelve la app, solo una vez
    const ProviderScope(child: MiApp()),
  );
}

class MiApp extends StatelessWidget {
  const MiApp({super.key});
  @override
  Widget build(BuildContext context) {
    return MaterialApp(home: const PantallaInicio());
  }
}

// ConsumerWidget = StatelessWidget con acceso a providers
class PantallaInicio extends ConsumerWidget {
  const PantallaInicio({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ref.watch — suscribe el widget, se reconstruye al cambiar
    final contador = ref.watch(contadorProvider);

    return Scaffold(
      body: Center(child: Text('$contador')),
      floatingActionButton: FloatingActionButton(
        // ref.read — lee sin suscribir, solo en callbacks
        onPressed: () => ref.read(contadorProvider.notifier).state++,
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

> **Mini-ejercicio ⏱ 5 min** — Agrega un segundo `FloatingActionButton` con `Icons.remove` que decremente el contador. Añade una guardia para que no baje de `0`.

---

### `ref.watch` vs `ref.read`

```dart
// ✅ watch — en build() para reconstruir cuando cambia
final valor = ref.watch(miProvider);

// ✅ read — en callbacks (onPressed, onChanged, initState)
onPressed: () => ref.read(miProvider.notifier).state = nuevoValor,

// ❌ NUNCA watch dentro de un callback — no tiene efecto
onPressed: () => ref.watch(miProvider),

// ❌ NUNCA read en build() si necesitas reactividad
Widget build(ctx, ref) {
  final valor = ref.read(miProvider); // no se actualiza — incorrecto
}
```

> **Mini-ejercicio ⏱ 5 min** — En un `ConsumerWidget`, reemplaza `ref.watch` por `ref.read` en el `build()` y prueba pulsar el botón. Observa que la UI no se actualiza. Devuelve `ref.watch` y confirma que vuelve a funcionar.

---

### `StateProvider` — estado primitivo

```dart
// Para valores simples: bool, int, String, enum
final modoOscuroProvider = StateProvider<bool>((ref) => false);
final indiceTabProvider   = StateProvider<int>((ref) => 0);
final filtroProvider      = StateProvider<String>((ref) => 'todos');

// Leer en build()
final modoOscuro = ref.watch(modoOscuroProvider);

// Modificar en callback
ref.read(modoOscuroProvider.notifier).state = true;

// Actualizar en base al valor anterior
ref.read(indiceTabProvider.notifier).update((n) => n + 1);
```

> **Mini-ejercicio ⏱ 5 min** — Crea un `StateProvider<String>` llamado `filtroEstadoProvider` con valor inicial `'todos'`. En la UI muestra el valor actual y un `SegmentedButton` con opciones `'todos'`, `'ssl'`, `'favoritos'`. Actualiza el provider al seleccionar.

---

### `NotifierProvider` — estado complejo síncrono

```dart
// Para lógica de negocio con múltiples métodos — equivale al ViewModel
class ServidoresNotifier extends Notifier<List<ServidorSSH>> {
  // build() define el estado inicial
  @override
  List<ServidorSSH> build() => [];

  void agregar(ServidorSSH servidor) {
    state = [...state, servidor];
  }

  void eliminar(String id) {
    state = state.where((s) => s.id != id).toList();
  }

  void toggleFavorito(String id) {
    state = state.map((s) =>
        s.id == id
            ? ServidorSSH(id: s.id, nombre: s.nombre, ip: s.ip,
                          puerto: s.puerto, ssl: s.ssl,
                          favorito: !s.favorito)
            : s
    ).toList();
  }
}

final servidoresProvider =
    NotifierProvider<ServidoresNotifier, List<ServidorSSH>>(
  ServidoresNotifier.new,
);
```

> **Mini-ejercicio ⏱ 5 min** — Agrega al `ServidoresNotifier` un método `toggleSsl(String id)` que invierta el valor de `ssl`. Llámalo desde un `IconButton` con `Icons.lock_outline` / `Icons.lock`.

---

### `Provider` derivado

```dart
// Filtro de búsqueda — estado primitivo
final busquedaProvider = StateProvider<String>((ref) => '');

// Provider DERIVADO — se recalcula cuando cualquier dependencia cambia
final servidoresFiltradosProvider = Provider<List<ServidorSSH>>((ref) {
  final todos    = ref.watch(servidoresProvider);
  final busqueda = ref.watch(busquedaProvider);

  if (busqueda.isEmpty) return todos;

  final q = busqueda.toLowerCase();
  return todos.where((s) =>
      s.nombre.toLowerCase().contains(q) || s.ip.contains(q)
  ).toList();
});
```

> **Mini-ejercicio ⏱ 5 min** — Añade un `Provider<int>` llamado `totalFavoritosProvider` que cuente cuántos servidores tienen `favorito == true`. Muéstralo en un `Badge` en la `AppBar`.

---

### `AsyncNotifierProvider` — estado asíncrono

```dart
// Para datos de API, BD o cualquier Future — maneja loading/data/error
class MetricasNotifier extends AsyncNotifier<List<MetricaServidor>> {
  // build() puede ser async — es la carga inicial
  @override
  Future<List<MetricaServidor>> build() => _fetch();

  Future<List<MetricaServidor>> _fetch() async {
    await Future.delayed(const Duration(milliseconds: 800));
    return const [
      MetricaServidor(servidor: 'prod-web-01', cpu: 45.2, ram: 62.1, conexiones: 230),
    ];
  }

  Future<void> recargar() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(_fetch);
  }
}

final metricasProvider =
    AsyncNotifierProvider<MetricasNotifier, List<MetricaServidor>>(
  MetricasNotifier.new,
);
```

> **Mini-ejercicio ⏱ 5 min** — Agrega al `MetricasNotifier` un método `actualizarCpu(String servidor, double nuevaCpu)` que modifique el valor de CPU de un servidor en la lista sin recargar todo. Usa `state.whenData(...)` para actualizar.

---

### `AsyncValue.when()` — los tres estados

```dart
// AsyncValue tiene tres estados: AsyncLoading, AsyncData, AsyncError
body: metricasAsync.when(
  loading: () => const Center(child: CircularProgressIndicator()),
  error: (error, stack) => Center(
    child: Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        const Icon(Icons.error_outline, size: 56, color: Colors.red),
        const SizedBox(height: 12),
        Text('Error: $error'),
        const SizedBox(height: 12),
        FilledButton.icon(
          onPressed: () => ref.read(metricasProvider.notifier).recargar(),
          icon:  const Icon(Icons.refresh),
          label: const Text('Reintentar'),
        ),
      ],
    ),
  ),
  data: (metricas) => ListView.builder(
    itemCount:   metricas.length,
    itemBuilder: (_, i) => _TarjetaMetrica(metrica: metricas[i]),
  ),
),
```

> **Mini-ejercicio ⏱ 5 min** — Añade `skipLoadingOnRefresh: true` al método `when()`. Luego pulsa recargar y observa la diferencia: la pantalla ya no muestra el spinner al refrescar, solo al cargarse por primera vez.

---

### Providers que dependen entre sí

```dart
// Estadísticas derivadas — depende de servidoresProvider
final estadisticasProvider = Provider<Map<String, int>>((ref) {
  final servidores = ref.watch(servidoresProvider);
  return {
    'total':     servidores.length,
    'ssl':       servidores.where((s) => s.ssl).length,
    'favoritos': servidores.where((s) => s.favorito).length,
  };
});

// En la UI
final stats = ref.watch(estadisticasProvider);
Text('${stats['total']} servidores · ${stats['ssl']} SSL');
// Cuando servidoresProvider cambia, estadisticasProvider se recalcula
// y los widgets que lo escuchan se reconstruyen automáticamente.
```

> **Mini-ejercicio ⏱ 5 min** — Crea un `Provider<bool>` llamado `hayServidoresCriticosProvider` que retorne `true` si algún servidor tiene `ssl == false`. Úsalo para mostrar un `Banner` de advertencia en la parte superior de la lista.

---

## Crea el proyecto

```bash
flutter create modulo10_riverpod
cd modulo10_riverpod
```

Agrega la dependencia en `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^3.3.0
```

```bash
flutter pub get
```

Borra todo el contenido de `lib/main.dart` y déjalo vacío.
El Paso 1 parte desde cero en ese archivo.

```
modulo10_riverpod/
├── lib/
│   ├── main.dart              ← aquí trabajamos
│   ├── models/                ← se crea al llegar al Paso 2
│   ├── providers/             ← se crea al llegar al Paso 2
│   └── screens/               ← se crea al llegar al Paso 5
├── pubspec.yaml
└── ...
```

Para correr la app:
```bash
flutter run
```

---

## Paso 1 — `ProviderScope` + `StateProvider` básico

Escribe esto en `lib/main.dart`:

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// StateProvider — estado simple (int, bool, String, enum)
final contadorProvider = StateProvider<int>((ref) => 0);

void main() {
  runApp(
    // ProviderScope: contenedor global, siempre envuelve la app
    const ProviderScope(child: AppMonitoreo()),
  );
}

class AppMonitoreo extends StatelessWidget {
  const AppMonitoreo({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0D47A1)),
        useMaterial3: true,
      ),
      home: const _PantallaContador(),
    );
  }
}

// ConsumerWidget = StatelessWidget con acceso a providers
class _PantallaContador extends ConsumerWidget {
  const _PantallaContador();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ref.watch — lee y se suscribe (reconstruye al cambiar)
    final count = ref.watch(contadorProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Servidores conectados')),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Text('$count', style: Theme.of(context).textTheme.displayLarge),
            const Text('servidores activos'),
          ],
        ),
      ),
      floatingActionButton: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          FloatingActionButton(
            heroTag: 'add',
            // ref.read — lee SIN suscribirse (para callbacks)
            onPressed: () =>
                ref.read(contadorProvider.notifier).state++,
            child: const Icon(Icons.add),
          ),
          const SizedBox(height: 8),
          FloatingActionButton(
            heroTag: 'rem',
            onPressed: () {
              if (ref.read(contadorProvider) > 0) {
                ref.read(contadorProvider.notifier).state--;
              }
            },
            child: const Icon(Icons.remove),
          ),
        ],
      ),
    );
  }
}
```

`ProviderScope` es el contenedor global — debe ir en `main()` y envolver toda la app, solo una vez.
`ConsumerWidget` reemplaza `StatelessWidget` y añade `WidgetRef ref` al método `build`.
`ref.watch` suscribe el widget al provider: cada vez que `contadorProvider` cambia, Flutter reconstruye el widget.
`ref.read` lee sin suscribirse — úsalo SOLO dentro de callbacks (`onPressed`, `onChanged`), nunca en `build`.

### Prueba esto

- Ejecuta la app — el contador empieza en `0`
- Pulsa `+` varias veces — el número sube y la UI se actualiza sola sin `setState`
- Pulsa `-` — el número baja; lleva el contador a `0` y vuelve a pulsar — nada cambia (la guardia del `if`)
- Cambia el valor inicial de `contadorProvider` de `0` a `5` — al recargar hot-restart parte desde `5`
- Reemplaza `ref.watch` por `ref.read` en `build()` — ¿qué pasa cuando pulsas `+`? (la UI no se actualiza — este es el error clásico)

---

## Convierte `main.dart` en selector de pasos

Antes de avanzar al Paso 2, actualiza `main.dart` **una sola vez**.
A partir de aquí solo cambias el número de `paso` para navegar entre ejemplos
— el Paso 1 sigue funcionando igual con `paso = 1`:

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Importa las pantallas a medida que las crees en cada paso:
// import 'screens/pantalla_servidores.dart';
// import 'screens/pantalla_busqueda.dart';
// import 'screens/pantalla_metricas.dart';
// import 'screens/pantalla_dashboard.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1  ProviderScope + StateProvider básico (contador)     │
// │  2  Paso 2  NotifierProvider + lista de servidores              │
// │  3  Paso 3  Provider derivado + búsqueda filtrada               │
// │  4  Paso 4  AsyncNotifierProvider + métricas loading/error      │
// │  5  Paso 5  NavigationBar con dos tabs usando Riverpod          │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 1;

// StateProvider — estado simple del Paso 1
final contadorProvider = StateProvider<int>((ref) => 0);

void main() {
  runApp(const ProviderScope(child: AppMonitoreo()));
}

class AppMonitoreo extends StatelessWidget {
  const AppMonitoreo({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0D47A1)),
        useMaterial3: true,
      ),
      home: switch (paso) {
        1 => const _Paso1(),
        // 2 => const PantallaServidores(),
        // 3 => const PantallaBusqueda(),
        // 4 => const PantallaMetricas(),
        // 5 => const PantallaDashboard(),
        _ => Scaffold(
            body: Center(child: Text('Paso $paso: crea el widget primero'))),
      },
    );
  }
}

// ─── Paso 1 — vive en main.dart ─────────────────────────────────────────
class _Paso1 extends ConsumerWidget {
  const _Paso1();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(contadorProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Servidores conectados')),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Text('$count', style: Theme.of(context).textTheme.displayLarge),
            const Text('servidores activos'),
          ],
        ),
      ),
      floatingActionButton: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          FloatingActionButton(
            heroTag: 'add',
            onPressed: () => ref.read(contadorProvider.notifier).state++,
            child: const Icon(Icons.add),
          ),
          const SizedBox(height: 8),
          FloatingActionButton(
            heroTag: 'rem',
            onPressed: () {
              if (ref.read(contadorProvider) > 0) {
                ref.read(contadorProvider.notifier).state--;
              }
            },
            child: const Icon(Icons.remove),
          ),
        ],
      ),
    );
  }
}
```

---

## Paso 2 — `NotifierProvider` + lista de servidores

Crea `lib/models/servidor_ssh.dart`:

```dart
// lib/models/servidor_ssh.dart
class ServidorSSH {
  final String id;
  final String nombre;
  final String ip;
  final int    puerto;
  final bool   ssl;
  bool         favorito;

  ServidorSSH({
    required this.id,
    required this.nombre,
    required this.ip,
    required this.puerto,
    required this.ssl,
    this.favorito = false,
  });
}
```

Crea `lib/providers/servidores_provider.dart`:

```dart
// lib/providers/servidores_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/servidor_ssh.dart';

// NotifierProvider — estado complejo con métodos propios
class ServidoresNotifier extends Notifier<List<ServidorSSH>> {
  @override
  List<ServidorSSH> build() => [
    ServidorSSH(id:'1', nombre:'prod-web-01', ip:'10.0.2.10', puerto:22,   ssl:true,  favorito:true),
    ServidorSSH(id:'2', nombre:'prod-db-01',  ip:'10.0.2.20', puerto:22,   ssl:true),
    ServidorSSH(id:'3', nombre:'staging-api', ip:'10.0.3.10', puerto:2222, ssl:false),
  ];

  void toggleFavorito(String id) {
    state = state.map((s) =>
        s.id == id
          ? ServidorSSH(id:s.id, nombre:s.nombre, ip:s.ip,
                        puerto:s.puerto, ssl:s.ssl,
                        favorito:!s.favorito)
          : s
    ).toList();
  }

  void eliminar(String id) {
    state = state.where((s) => s.id != id).toList();
  }

  void agregar(ServidorSSH servidor) {
    state = [...state, servidor];
  }
}

final servidoresProvider =
    NotifierProvider<ServidoresNotifier, List<ServidorSSH>>(
  ServidoresNotifier.new,
);
```

Crea `lib/screens/pantalla_servidores.dart`:

```dart
// lib/screens/pantalla_servidores.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/servidores_provider.dart';

class PantallaServidores extends ConsumerWidget {
  const PantallaServidores({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final servidores = ref.watch(servidoresProvider);
    final cs         = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           Text('Servidores (${servidores.length})'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: servidores.isEmpty
          ? const Center(child: Text('Sin servidores'))
          : ListView.separated(
              itemCount:        servidores.length,
              separatorBuilder: (_, __) =>
                  const Divider(height: 1, indent: 72),
              itemBuilder: (context, i) {
                final s = servidores[i];
                return ListTile(
                  leading: CircleAvatar(
                    backgroundColor: s.ssl
                        ? Colors.green.shade50
                        : Colors.grey.shade100,
                    child: Icon(Icons.dns,
                        color: s.ssl ? Colors.green : Colors.grey),
                  ),
                  title:    Text(s.nombre,
                      style: const TextStyle(fontWeight: FontWeight.w600)),
                  subtitle: Text('${s.ip}:${s.puerto}'),
                  trailing: Row(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      IconButton(
                        icon: Icon(
                          s.favorito ? Icons.star : Icons.star_border,
                          color: s.favorito ? Colors.amber : null,
                        ),
                        onPressed: () => ref
                            .read(servidoresProvider.notifier)
                            .toggleFavorito(s.id),
                      ),
                      IconButton(
                        icon: const Icon(Icons.delete_outline,
                            color: Colors.red),
                        onPressed: () => ref
                            .read(servidoresProvider.notifier)
                            .eliminar(s.id),
                      ),
                    ],
                  ),
                );
              },
            ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          final id = DateTime.now().millisecondsSinceEpoch.toString();
          ref.read(servidoresProvider.notifier).agregar(
            ServidorSSH(
              id:     id,
              nombre: 'nuevo-srv-$id',
              ip:     '192.168.0.${servidores.length + 1}',
              puerto: 22,
              ssl:    true,
            ),
          );
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

### Agrega al `main.dart`

Descomenta el import y el `case` en el selector:

```dart
import 'screens/pantalla_servidores.dart';

// en el switch:
2 => const PantallaServidores(),
```

Cambia `const int paso = 2;` y guarda.

### Prueba esto

- Ejecuta la app — ves la lista con 3 servidores
- Pulsa la estrella de `prod-web-01` — el ícono cambia de vacío a lleno sin `setState`
- Pulsa el basurero de `staging-api` — desaparece de la lista
- Pulsa el `+` flotante varias veces — se agregan servidores nuevos
- Vuelve al Paso 1 (cambia `paso = 1`) — el contador independiente sigue en su valor

---

## Paso 3 — `Provider` derivado + búsqueda filtrada

Agrega al final de `lib/providers/servidores_provider.dart`:

```dart
// Filtro de búsqueda — estado primitivo
final busquedaProvider = StateProvider<String>((ref) => '');

// Provider DERIVADO — se recalcula cuando cualquiera de sus dependencias cambia
final servidoresFiltradosProvider = Provider<List<ServidorSSH>>((ref) {
  final todos    = ref.watch(servidoresProvider);
  final busqueda = ref.watch(busquedaProvider);

  if (busqueda.isEmpty) return todos;

  final q = busqueda.toLowerCase();
  return todos.where((s) =>
      s.nombre.toLowerCase().contains(q) || s.ip.contains(q)
  ).toList();
  // Cuando 'servidoresProvider' o 'busquedaProvider' cambian,
  // este provider se recalcula automáticamente.
});
```

Crea `lib/screens/pantalla_busqueda.dart`:

```dart
// lib/screens/pantalla_busqueda.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/servidores_provider.dart';

class PantallaBusqueda extends ConsumerWidget {
  const PantallaBusqueda({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final servidores = ref.watch(servidoresFiltradosProvider);
    final busqueda   = ref.watch(busquedaProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Buscar servidores')),
      body: Column(children: [
        Padding(
          padding: const EdgeInsets.all(12),
          child: SearchBar(
            hintText: 'Buscar por nombre o IP...',
            leading:  const Icon(Icons.search),
            trailing: busqueda.isNotEmpty
                ? [IconButton(
                    icon: const Icon(Icons.clear),
                    onPressed: () =>
                        ref.read(busquedaProvider.notifier).state = '',
                  )]
                : null,
            onChanged: (v) =>
                ref.read(busquedaProvider.notifier).state = v,
            padding: const WidgetStatePropertyAll(
              EdgeInsets.symmetric(horizontal: 16),
            ),
          ),
        ),
        Expanded(
          child: servidores.isEmpty
              ? const Center(child: Text('Sin resultados'))
              : ListView.builder(
                  itemCount:   servidores.length,
                  itemBuilder: (_, i) => ListTile(
                    leading: const Icon(Icons.dns),
                    title:    Text(servidores[i].nombre),
                    subtitle: Text(servidores[i].ip),
                  ),
                ),
        ),
      ]),
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'screens/pantalla_busqueda.dart';

// en el switch:
3 => const PantallaBusqueda(),
```

Cambia `const int paso = 3;` y guarda.

### Prueba esto

- Escribe `web` en el campo de búsqueda — solo aparecen servidores con "web" en el nombre
- Escribe `10.0` — filtra por IP
- Pulsa la X — limpia el filtro y vuelven todos
- Ve al Paso 2 (cambia `paso = 2`) y elimina un servidor — vuelve al 3, la lista filtrada también refleja el cambio (son el mismo provider)

---

## Paso 4 — `AsyncNotifierProvider` + métricas con `loading/error/data`

Crea `lib/models/metrica_servidor.dart`:

```dart
// lib/models/metrica_servidor.dart
class MetricaServidor {
  final String servidor;
  final double cpu;
  final double ram;
  final int    conexiones;

  const MetricaServidor({
    required this.servidor,
    required this.cpu,
    required this.ram,
    required this.conexiones,
  });
}
```

Crea `lib/providers/metricas_provider.dart`:

```dart
// lib/providers/metricas_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/metrica_servidor.dart';

class MetricasNotifier extends AsyncNotifier<List<MetricaServidor>> {
  // build() puede ser async — es la carga inicial
  @override
  Future<List<MetricaServidor>> build() => _fetch();

  Future<List<MetricaServidor>> _fetch() async {
    await Future.delayed(const Duration(milliseconds: 800));
    return const [
      MetricaServidor(servidor:'prod-web-01', cpu:45.2, ram:62.1, conexiones:230),
      MetricaServidor(servidor:'prod-db-01',  cpu:88.1, ram:91.2, conexiones:80),
      MetricaServidor(servidor:'staging-api', cpu:22.4, ram:41.0, conexiones:50),
    ];
  }

  Future<void> recargar() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(_fetch);
  }
}

final metricasProvider =
    AsyncNotifierProvider<MetricasNotifier, List<MetricaServidor>>(
  MetricasNotifier.new,
);
```

Crea `lib/screens/pantalla_metricas.dart`:

```dart
// lib/screens/pantalla_metricas.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/metrica_servidor.dart';
import '../providers/metricas_provider.dart';

class PantallaMetricas extends ConsumerWidget {
  const PantallaMetricas({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final metricasAsync = ref.watch(metricasProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Métricas de servidores'),
        actions: [
          IconButton(
            icon:    const Icon(Icons.refresh),
            tooltip: 'Recargar',
            onPressed: () =>
                ref.read(metricasProvider.notifier).recargar(),
          ),
        ],
      ),
      body: metricasAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.error_outline, size: 48, color: Colors.red),
              const SizedBox(height: 8),
              Text('Error: $e'),
              const SizedBox(height: 12),
              FilledButton.icon(
                onPressed: () =>
                    ref.read(metricasProvider.notifier).recargar(),
                icon:  const Icon(Icons.refresh),
                label: const Text('Reintentar'),
              ),
            ],
          ),
        ),
        data: (metricas) => ListView.builder(
          padding:     const EdgeInsets.all(12),
          itemCount:   metricas.length,
          itemBuilder: (_, i) => _TarjetaMetrica(metrica: metricas[i]),
        ),
      ),
    );
  }
}

class _TarjetaMetrica extends StatelessWidget {
  final MetricaServidor metrica;
  const _TarjetaMetrica({required this.metrica});

  @override
  Widget build(BuildContext context) {
    final cs         = Theme.of(context).colorScheme;
    final cpuCritica = metrica.cpu > 85;
    final ramCritica = metrica.ram > 90;
    final esCritico  = cpuCritica || ramCritica;

    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      color:  esCritico ? cs.errorContainer : null,
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(children: [
              Icon(Icons.dns, color: esCritico ? cs.error : cs.primary, size: 18),
              const SizedBox(width: 8),
              Text(metrica.servidor,
                  style: const TextStyle(fontWeight: FontWeight.bold)),
              const Spacer(),
              Text('${metrica.conexiones} conn',
                  style: TextStyle(fontSize: 12, color: cs.onSurfaceVariant)),
            ]),
            const SizedBox(height: 10),
            _Barra('CPU', metrica.cpu, cpuCritica),
            const SizedBox(height: 4),
            _Barra('RAM', metrica.ram, ramCritica),
          ],
        ),
      ),
    );
  }
}

class _Barra extends StatelessWidget {
  final String label;
  final double valor;
  final bool   critica;
  const _Barra(this.label, this.valor, this.critica);

  @override
  Widget build(BuildContext context) {
    final color = critica ? Colors.red : Colors.green;
    return Row(children: [
      SizedBox(width: 36, child: Text(label,
          style: const TextStyle(fontSize: 12))),
      Expanded(
        child: LinearProgressIndicator(
          value:           valor / 100,
          backgroundColor: Colors.grey.shade200,
          valueColor:      AlwaysStoppedAnimation(color),
        ),
      ),
      const SizedBox(width: 8),
      Text('${valor.toStringAsFixed(1)}%',
          style: TextStyle(fontSize: 12, color: color,
              fontWeight: FontWeight.w600)),
    ]);
  }
}
```

### Agrega al `main.dart`

```dart
import 'screens/pantalla_metricas.dart';

// en el switch:
4 => const PantallaMetricas(),
```

Cambia `const int paso = 4;` y guarda.

### Prueba esto

- Ejecuta — ves 800ms de `CircularProgressIndicator` y luego la lista de métricas
- Pulsa el ícono de recarga — la pantalla muestra el loading nuevamente y recarga
- Cambia `cpu: 88.1` de `prod-db-01` a `20.0` — en hot-restart la tarjeta ya no aparece en rojo
- Busca en el archivo el `Future.delayed` y cambia `800` a `2000` — el loading dura más

---

## Paso 5 — `NavigationBar` con dos tabs gestionada con Riverpod

En lugar de un `StatefulWidget` con `setState`, la pestaña activa vive en un `StateProvider`.

Crea `lib/screens/pantalla_dashboard.dart`:

```dart
// lib/screens/pantalla_dashboard.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'pantalla_servidores.dart';
import 'pantalla_metricas.dart';

final indiceTabProvider = StateProvider<int>((ref) => 0);

class PantallaDashboard extends ConsumerWidget {
  const PantallaDashboard({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final indice = ref.watch(indiceTabProvider);

    return Scaffold(
      body: switch (indice) {
        0 => const PantallaServidores(),
        1 => const PantallaMetricas(),
        _ => const PantallaServidores(),
      },
      bottomNavigationBar: NavigationBar(
        selectedIndex:         indice,
        onDestinationSelected: (i) =>
            ref.read(indiceTabProvider.notifier).state = i,
        destinations: const [
          NavigationDestination(
            icon:         Icon(Icons.dns_outlined),
            selectedIcon: Icon(Icons.dns),
            label:        'Servidores',
          ),
          NavigationDestination(
            icon:         Icon(Icons.bar_chart_outlined),
            selectedIcon: Icon(Icons.bar_chart),
            label:        'Métricas',
          ),
        ],
      ),
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'screens/pantalla_dashboard.dart';

// en el switch:
5 => const PantallaDashboard(),
```

Cambia `const int paso = 5;` y guarda.

### Prueba esto

- Ejecuta — aparece la `NavigationBar` con dos pestañas
- Cambia a Métricas — la carga asíncrona funciona desde la primera vez
- Regresa a Servidores — los datos siguen ahí (el estado sobrevive al cambio de tab)
- Añade un servidor en la pestaña Servidores y vuelve — sigue en la lista
- Compara con cómo lo harías con `StatefulWidget` — ¿cuántas líneas más necesitarías?

---

## `main.dart` completo — referencia

Con todos los pasos integrados (`paso = 5` seleccionado por defecto):

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'screens/pantalla_servidores.dart';
import 'screens/pantalla_busqueda.dart';
import 'screens/pantalla_metricas.dart';
import 'screens/pantalla_dashboard.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1  ProviderScope + StateProvider básico (contador)     │
// │  2  Paso 2  NotifierProvider + lista de servidores              │
// │  3  Paso 3  Provider derivado + búsqueda filtrada               │
// │  4  Paso 4  AsyncNotifierProvider + métricas loading/error      │
// │  5  Paso 5  NavigationBar con dos tabs usando Riverpod          │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 5;

// StateProvider del Paso 1
final contadorProvider = StateProvider<int>((ref) => 0);

void main() {
  runApp(const ProviderScope(child: AppMonitoreo()));
}

class AppMonitoreo extends StatelessWidget {
  const AppMonitoreo({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0D47A1)),
        useMaterial3: true,
      ),
      home: switch (paso) {
        1 => const _Paso1(),
        2 => const PantallaServidores(),
        3 => const PantallaBusqueda(),
        4 => const PantallaMetricas(),
        5 => const PantallaDashboard(),
        _ => Scaffold(
            body: Center(child: Text('Paso $paso: crea el widget primero'))),
      },
    );
  }
}

// ─── Paso 1 — vive en main.dart ─────────────────────────────────────────
class _Paso1 extends ConsumerWidget {
  const _Paso1();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(contadorProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('Servidores conectados')),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Text('$count', style: Theme.of(context).textTheme.displayLarge),
            const Text('servidores activos'),
          ],
        ),
      ),
      floatingActionButton: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          FloatingActionButton(
            heroTag: 'add',
            onPressed: () => ref.read(contadorProvider.notifier).state++,
            child: const Icon(Icons.add),
          ),
          const SizedBox(height: 8),
          FloatingActionButton(
            heroTag: 'rem',
            onPressed: () {
              if (ref.read(contadorProvider) > 0) {
                ref.read(contadorProvider.notifier).state--;
              }
            },
            child: const Icon(Icons.remove),
          ),
        ],
      ),
    );
  }
}
```

---

## Proyecto — Dashboard de Monitoreo de Servidores

App completa con `NavigationBar` de 2 tabs: **Servidores** (lista con favorito, eliminar y búsqueda) y **Métricas** (carga asíncrona con refresco). Todo el estado vive en providers Riverpod — cero `setState`.

```dart
// ─── Punto de entrada ────────────────────────────────────────────────────
// main() con ProviderScope y todos los providers definidos arriba
// (ver archivos individuales creados en los pasos anteriores)

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'models/servidor_ssh.dart';
import 'models/metrica_servidor.dart';

// ── Providers ─────────────────────────────────────────────────────────────

class ServidoresNotifier extends Notifier<List<ServidorSSH>> {
  @override
  List<ServidorSSH> build() => [
    ServidorSSH(id:'1', nombre:'prod-web-01', ip:'10.0.2.10', puerto:22,   ssl:true,  favorito:true),
    ServidorSSH(id:'2', nombre:'prod-db-01',  ip:'10.0.2.20', puerto:22,   ssl:true),
    ServidorSSH(id:'3', nombre:'staging-api', ip:'10.0.3.10', puerto:2222, ssl:false),
  ];

  void toggleFavorito(String id) {
    state = state.map((s) =>
        s.id == id
            ? ServidorSSH(id:s.id, nombre:s.nombre, ip:s.ip,
                          puerto:s.puerto, ssl:s.ssl,
                          favorito:!s.favorito)
            : s
    ).toList();
  }

  void eliminar(String id) {
    state = state.where((s) => s.id != id).toList();
  }

  void agregar(ServidorSSH servidor) {
    state = [...state, servidor];
  }
}

final servidoresProvider =
    NotifierProvider<ServidoresNotifier, List<ServidorSSH>>(
  ServidoresNotifier.new,
);

final busquedaProvider            = StateProvider<String>((ref) => '');
final indiceTabProvider           = StateProvider<int>((ref) => 0);

final servidoresFiltradosProvider = Provider<List<ServidorSSH>>((ref) {
  final todos    = ref.watch(servidoresProvider);
  final busqueda = ref.watch(busquedaProvider);
  if (busqueda.isEmpty) return todos;
  final q = busqueda.toLowerCase();
  return todos.where((s) =>
      s.nombre.toLowerCase().contains(q) || s.ip.contains(q)).toList();
});

final estadisticasProvider = Provider<Map<String, int>>((ref) {
  final servidores = ref.watch(servidoresProvider);
  return {
    'total':     servidores.length,
    'ssl':       servidores.where((s) => s.ssl).length,
    'favoritos': servidores.where((s) => s.favorito).length,
  };
});

class MetricasNotifier extends AsyncNotifier<List<MetricaServidor>> {
  @override
  Future<List<MetricaServidor>> build() => _fetch();

  Future<List<MetricaServidor>> _fetch() async {
    await Future.delayed(const Duration(milliseconds: 600));
    return const [
      MetricaServidor(servidor:'prod-web-01', cpu:45.2, ram:62.1, conexiones:230),
      MetricaServidor(servidor:'prod-db-01',  cpu:88.1, ram:91.2, conexiones:80),
      MetricaServidor(servidor:'staging-api', cpu:22.4, ram:41.0, conexiones:50),
    ];
  }

  Future<void> recargar() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(_fetch);
  }
}

final metricasProvider =
    AsyncNotifierProvider<MetricasNotifier, List<MetricaServidor>>(
  MetricasNotifier.new,
);

// ── App ───────────────────────────────────────────────────────────────────

void main() {
  runApp(const ProviderScope(child: AppMonitoreo()));
}

class AppMonitoreo extends StatelessWidget {
  const AppMonitoreo({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'Dashboard SSH',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0D47A1)),
        useMaterial3: true,
      ),
      home: const _PantallaRaiz(),
    );
  }
}

class _PantallaRaiz extends ConsumerWidget {
  const _PantallaRaiz();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final indice = ref.watch(indiceTabProvider);

    return Scaffold(
      body: switch (indice) {
        0 => const _TabServidores(),
        1 => const _TabMetricas(),
        _ => const _TabServidores(),
      },
      bottomNavigationBar: NavigationBar(
        selectedIndex:         indice,
        onDestinationSelected: (i) =>
            ref.read(indiceTabProvider.notifier).state = i,
        destinations: const [
          NavigationDestination(
            icon:         Icon(Icons.dns_outlined),
            selectedIcon: Icon(Icons.dns),
            label:        'Servidores',
          ),
          NavigationDestination(
            icon:         Icon(Icons.bar_chart_outlined),
            selectedIcon: Icon(Icons.bar_chart),
            label:        'Métricas',
          ),
        ],
      ),
    );
  }
}

// ── Tab Servidores ────────────────────────────────────────────────────────

class _TabServidores extends ConsumerWidget {
  const _TabServidores();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final servidores = ref.watch(servidoresFiltradosProvider);
    final stats      = ref.watch(estadisticasProvider);
    final cs         = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           Text('Servidores (${stats['total']})'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: Column(children: [
        // Estadísticas rápidas
        Padding(
          padding: const EdgeInsets.all(12),
          child: Row(children: [
            _ChipStat(label: '${stats['ssl']} SSL',       color: Colors.green),
            const SizedBox(width: 8),
            _ChipStat(label: '${stats['favoritos']} fav', color: Colors.amber),
          ]),
        ),
        // Búsqueda
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 12),
          child: SearchBar(
            hintText: 'Buscar...',
            leading:  const Icon(Icons.search),
            onChanged: (v) =>
                ref.read(busquedaProvider.notifier).state = v,
            padding: const WidgetStatePropertyAll(
                EdgeInsets.symmetric(horizontal: 16)),
          ),
        ),
        const SizedBox(height: 8),
        // Lista
        Expanded(
          child: servidores.isEmpty
              ? const Center(child: Text('Sin resultados'))
              : ListView.separated(
                  itemCount:        servidores.length,
                  separatorBuilder: (_, __) =>
                      const Divider(height: 1, indent: 72),
                  itemBuilder: (context, i) {
                    final s = servidores[i];
                    return ListTile(
                      leading: CircleAvatar(
                        backgroundColor: s.ssl
                            ? Colors.green.shade50
                            : Colors.grey.shade100,
                        child: Icon(Icons.dns,
                            color: s.ssl
                                ? Colors.green.shade700
                                : Colors.grey),
                      ),
                      title:    Text(s.nombre,
                          style: const TextStyle(fontWeight: FontWeight.w600)),
                      subtitle: Text('${s.ip}:${s.puerto}'),
                      trailing: Row(
                        mainAxisSize: MainAxisSize.min,
                        children: [
                          IconButton(
                            icon: Icon(
                              s.favorito ? Icons.star : Icons.star_border,
                              color: s.favorito ? Colors.amber : null,
                            ),
                            visualDensity: VisualDensity.compact,
                            onPressed: () => ref
                                .read(servidoresProvider.notifier)
                                .toggleFavorito(s.id),
                          ),
                          IconButton(
                            icon: const Icon(Icons.delete_outline,
                                color: Colors.red),
                            visualDensity: VisualDensity.compact,
                            onPressed: () => ref
                                .read(servidoresProvider.notifier)
                                .eliminar(s.id),
                          ),
                        ],
                      ),
                    );
                  },
                ),
        ),
      ]),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          final id = DateTime.now().millisecondsSinceEpoch.toString();
          ref.read(servidoresProvider.notifier).agregar(
            ServidorSSH(
              id:     id,
              nombre: 'nuevo-srv-${servidores.length + 1}',
              ip:     '192.168.0.${servidores.length + 1}',
              puerto: 22,
              ssl:    true,
            ),
          );
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}

// ── Tab Métricas ──────────────────────────────────────────────────────────

class _TabMetricas extends ConsumerWidget {
  const _TabMetricas();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final metricasAsync = ref.watch(metricasProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Métricas'),
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: () =>
                ref.read(metricasProvider.notifier).recargar(),
          ),
        ],
      ),
      body: metricasAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, _) => Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.error_outline, size: 48, color: Colors.red),
              const SizedBox(height: 8),
              Text('$e'),
              const SizedBox(height: 12),
              FilledButton.icon(
                onPressed: () =>
                    ref.read(metricasProvider.notifier).recargar(),
                icon:  const Icon(Icons.refresh),
                label: const Text('Reintentar'),
              ),
            ],
          ),
        ),
        data: (metricas) => ListView.builder(
          padding:     const EdgeInsets.all(12),
          itemCount:   metricas.length,
          itemBuilder: (_, i) => _TarjetaMetrica(metrica: metricas[i]),
        ),
      ),
    );
  }
}

class _TarjetaMetrica extends StatelessWidget {
  final MetricaServidor metrica;
  const _TarjetaMetrica({required this.metrica});

  @override
  Widget build(BuildContext context) {
    final cs         = Theme.of(context).colorScheme;
    final cpuCritica = metrica.cpu > 85;
    final ramCritica = metrica.ram > 90;
    final esCritico  = cpuCritica || ramCritica;

    return Card(
      margin: const EdgeInsets.only(bottom: 8),
      color:  esCritico ? cs.errorContainer : null,
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(children: [
              Icon(Icons.dns,
                  color: esCritico ? cs.error : cs.primary, size: 18),
              const SizedBox(width: 8),
              Text(metrica.servidor,
                  style: const TextStyle(fontWeight: FontWeight.bold)),
              const Spacer(),
              Text('${metrica.conexiones} conn',
                  style: TextStyle(fontSize: 12, color: cs.onSurfaceVariant)),
            ]),
            const SizedBox(height: 10),
            _Barra('CPU', metrica.cpu, cpuCritica),
            const SizedBox(height: 4),
            _Barra('RAM', metrica.ram, ramCritica),
          ],
        ),
      ),
    );
  }
}

class _Barra extends StatelessWidget {
  final String label;
  final double valor;
  final bool   critica;
  const _Barra(this.label, this.valor, this.critica);

  @override
  Widget build(BuildContext context) {
    final color = critica ? Colors.red : Colors.green;
    return Row(children: [
      SizedBox(width: 36, child: Text(label,
          style: const TextStyle(fontSize: 12))),
      Expanded(
        child: LinearProgressIndicator(
          value:           valor / 100,
          backgroundColor: Colors.grey.shade200,
          valueColor:      AlwaysStoppedAnimation(color),
        ),
      ),
      const SizedBox(width: 8),
      Text('${valor.toStringAsFixed(1)}%',
          style: TextStyle(fontSize: 12, color: color,
              fontWeight: FontWeight.w600)),
    ]);
  }
}

// ── Helpers ───────────────────────────────────────────────────────────────

class _ChipStat extends StatelessWidget {
  final String label;
  final Color  color;
  const _ChipStat({required this.label, required this.color});

  @override
  Widget build(BuildContext context) => Chip(
    label:           Text(label,
        style: TextStyle(color: color, fontWeight: FontWeight.w600,
            fontSize: 12)),
    backgroundColor: color.withOpacity(0.1),
    side:            BorderSide(color: color.withOpacity(0.3)),
    padding:         EdgeInsets.zero,
    visualDensity:   VisualDensity.compact,
  );
}
```

---

## Guía rápida de imports

```dart
// Siempre necesario con Riverpod
import 'package:flutter_riverpod/flutter_riverpod.dart';

// StateProvider / NotifierProvider / AsyncNotifierProvider — todos en flutter_riverpod
// No hay imports separados por tipo de provider

// Clases disponibles tras el import:
// ProviderScope, ConsumerWidget, ConsumerStatefulWidget
// StateProvider, Provider, NotifierProvider, AsyncNotifierProvider
// Notifier, AsyncNotifier
// AsyncValue, AsyncData, AsyncLoading, AsyncError
// WidgetRef
```

---

## Cuándo usar cada provider

| Necesito... | Provider a usar |
|---|---|
| Un valor que nunca cambia / se calcula una sola vez | `Provider` |
| Un bool, int, String o enum que cambia | `StateProvider` |
| Lógica compleja con varios métodos (ej: lista con CRUD) | `NotifierProvider` |
| Datos que vienen de una API / son asíncronos | `AsyncNotifierProvider` |
| Un valor derivado de otros providers | `Provider` que hace `ref.watch` |

---

## Ejercicios propuestos

1. **Provider de configuración** — Crea un `NotifierProvider<ConfigNotifier, ConfigApp>`
   con campos `tema`, `idioma` y `notificaciones`. Implementa métodos
   `setTema`, `setIdioma` y `toggleNotificaciones`. Conéctalo a la
   `MaterialApp` para que el modo oscuro cambie en tiempo real.

2. **`AsyncNotifier` con reintentos** — Extiende `MetricasNotifier` para
   que al fallar la carga, reintente automáticamente hasta 3 veces con
   500ms de espera entre intentos. Muestra en la UI el número de intento
   actual mientras carga.

3. **Provider derivado de filtros múltiples** — Añade al programa completo
   un `StateProvider<bool> soloSslProvider` y un
   `StateProvider<bool> soloFavoritosProvider`. Crea un provider
   `servidoresFiltradosProvider` que combine búsqueda + soloSSL +
   soloFavoritos. Añade `FilterChip` en la UI para activarlos.

4. **`ConsumerStatefulWidget`** — Convierte `_TabServidores` en un
   `ConsumerStatefulWidget`. En `initState`, usa `ref.read` para registrar
   una acción de log. En `dispose`, usa `ref.read` para otro log.
   Explica por qué `ref.watch` no puede usarse en `initState`.

---

## Resumen de la página 10

- `ProviderScope` envuelve toda la app en `main()` — es el contenedor global de todos los providers. Solo se pone una vez.
- `ConsumerWidget` reemplaza `StatelessWidget`. `ConsumerStatefulWidget` reemplaza `StatefulWidget`. Ambos reciben `WidgetRef ref` en `build()`.
- `ref.watch()` se usa en `build()` — suscribe el widget a cambios y lo reconstruye. `ref.read()` se usa en callbacks — lee sin suscribir.
- `Provider` para valores calculados. `StateProvider` para estado primitivo simple. `NotifierProvider` para lógica compleja con métodos.
- `AsyncNotifierProvider` maneja automáticamente los estados loading/data/error. `build()` puede ser `async`.
- `AsyncValue.when(loading, error, data)` es la forma idiomática de renderizar los tres estados posibles de un `AsyncNotifierProvider`.
- `AsyncValue.guard(fn)` ejecuta una función async y captura automáticamente los errores en `AsyncError`.
- Los providers pueden depender entre sí con `ref.watch()` dentro de otro provider — se recalculan automáticamente cuando cualquier dependencia cambia.

---

> **Siguiente página →** Página 11: Navegación con GoRouter —
> rutas, `go()`, `push()`, parámetros y guards de autenticación.
