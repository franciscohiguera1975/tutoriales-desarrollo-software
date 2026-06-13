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
  flutter_riverpod: ^3.3.0
  riverpod_annotation: ^4.0.2   # para la sintaxis moderna con @riverpod

dev_dependencies:
  build_runner:                  # genera código de los providers
  riverpod_generator: ^4.0.3
```

```bash
# Generar código (cada vez que cambies un provider anotado)
dart run build_runner watch --delete-conflicting-outputs
```

---

## Configuración inicial

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() {
  runApp(
    // ProviderScope es el contenedor global de todos los providers
    // DEBE envolver toda la app — solo una vez en main()
    const ProviderScope(
      child: MiApp(),
    ),
  );
}

class MiApp extends StatelessWidget {
  const MiApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Mi App con Riverpod',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const PantallaInicio(),
    );
  }
}
```

---

## `ConsumerWidget` — leer providers en la UI

Para leer providers, el widget cambia de `StatelessWidget`
a `ConsumerWidget`. La diferencia es el parámetro `WidgetRef ref`
en `build()`:

```dart
// StatelessWidget                  ConsumerWidget
// ─────────────────                ─────────────────────────────────
// Widget build(BuildContext ctx)   Widget build(BuildContext ctx, WidgetRef ref)
//                                                                    ↑
//                                                        acceso a los providers

class PantallaInicio extends ConsumerWidget {
  const PantallaInicio({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ref.watch() — lee el valor Y se suscribe a cambios
    // El widget se reconstruye cuando el provider cambia
    final contador = ref.watch(contadorProvider);

    return Scaffold(
      body: Center(child: Text('$contador')),
      floatingActionButton: FloatingActionButton(
        // ref.read() — lee el valor SIN suscribirse
        // Solo para usar en callbacks como onPressed
        onPressed: () => ref.read(contadorProvider.notifier).incrementar(),
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

### Regla de oro: `watch` vs `read`

```dart
// ✅ watch — en build() para reconstruir cuando cambia
final valor = ref.watch(miProvider);

// ✅ read — en callbacks (onPressed, onChanged, initState)
onPressed: () => ref.read(miProvider.notifier).hacerAlgo(),

// ❌ NUNCA watch dentro de un callback — no tiene efecto
onPressed: () => ref.watch(miProvider),  // incorrecto

// ❌ NUNCA read en build() si necesitas reactividad
Widget build(ctx, ref) {
  final valor = ref.read(miProvider);  // no se actualiza — incorrecto
}
```

---

## Tipos de provider

### `Provider` — valor inmutable

```dart
// Para valores que no cambian o se calculan una vez
final bienvenidaProvider = Provider<String>((ref) {
  return 'Sistema de Monitoreo v2.4.1';
});

// Provider que depende de otro provider
final resumenProvider = Provider<String>((ref) {
  final servidores = ref.watch(servidoresProvider);
  final activos    = servidores.where((s) => s.activo).length;
  return '$activos/${servidores.length} servidores activos';
});
```

### `StateProvider` — estado primitivo simple

```dart
// Para valores simples: bool, int, String, enum
final modoOscuroProvider = StateProvider<bool>((ref) => false);
final indiceTabProvider   = StateProvider<int>((ref) => 0);
final filtroProvider      = StateProvider<String>((ref) => 'todos');

// Leer
final modoOscuro = ref.watch(modoOscuroProvider);

// Modificar
ref.read(modoOscuroProvider.notifier).state = true;
// o usando update para depender del valor anterior
ref.read(indiceTabProvider.notifier).update((n) => n + 1);
```

### `NotifierProvider` — estado complejo síncrono

```dart
// Para lógica de negocio compleja con múltiples métodos
// Equivale al ViewModel de Kotlin

class ServidoresFiltroNotifier extends Notifier<List<ServidorSSH>> {
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
    state = state.map((s) {
      return s.id == id
          ? ServidorSSH(
              id:       s.id,
              nombre:   s.nombre,
              ip:       s.ip,
              puerto:   s.puerto,
              usuario:  s.usuario,
              so:       s.so,
              ssl:      s.ssl,
              favorito: !s.favorito,
            )
          : s;
    }).toList();
  }

  List<ServidorSSH> filtrarPor(String texto) =>
      state.where((s) =>
          s.nombre.toLowerCase().contains(texto.toLowerCase()) ||
          s.ip.contains(texto)).toList();
}

final servidoresProvider = NotifierProvider<ServidoresFiltroNotifier, List<ServidorSSH>>(
  ServidoresFiltroNotifier.new,
);
```

### `AsyncNotifierProvider` — estado asíncrono

```dart
// Para datos que vienen de una API, base de datos, etc.
// Maneja automáticamente los estados: loading, data, error

class MetricasNotifier extends AsyncNotifier<List<MetricaServidor>> {
  // build() puede ser async — carga los datos iniciales
  @override
  Future<List<MetricaServidor>> build() async {
    return _cargarMetricas();
  }

  Future<List<MetricaServidor>> _cargarMetricas() async {
    // Simular llamada a API
    await Future.delayed(const Duration(milliseconds: 800));
    return [
      MetricaServidor(servidor: 'web-01',  cpu: 45.2, ram: 62.1, conexiones: 230),
      MetricaServidor(servidor: 'web-02',  cpu: 38.8, ram: 55.0, conexiones: 180),
      MetricaServidor(servidor: 'api-01',  cpu: 72.3, ram: 78.4, conexiones: 450),
      MetricaServidor(servidor: 'db-01',   cpu: 88.1, ram: 91.2, conexiones: 80),
    ];
  }

  // Método para recargar
  Future<void> recargar() async {
    // state = const AsyncLoading();  // opcional — muestra loading
    state = await AsyncValue.guard(_cargarMetricas);
  }

  // Método para actualizar un servidor específico
  void actualizarServidor(String nombre, double cpu, double ram) {
    state.whenData((lista) {
      state = AsyncData(lista.map((m) =>
          m.servidor == nombre
              ? MetricaServidor(servidor: nombre, cpu: cpu, ram: ram, conexiones: m.conexiones)
              : m,
      ).toList());
    });
  }
}

final metricasProvider =
    AsyncNotifierProvider<MetricasNotifier, List<MetricaServidor>>(
  MetricasNotifier.new,
);
```

---

## `AsyncValue` — manejar loading/data/error

```dart
// AsyncValue tiene tres estados:
// AsyncLoading() — cargando
// AsyncData(valor) — datos disponibles
// AsyncError(error, stackTrace) — falló

class PantallaMetricas extends ConsumerWidget {
  const PantallaMetricas({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final metricasAsync = ref.watch(metricasProvider);

    // when() — la forma más limpia de manejar los tres estados
    return Scaffold(
      appBar: AppBar(
        title: const Text('Métricas en tiempo real'),
        actions: [
          IconButton(
            icon:      const Icon(Icons.refresh),
            onPressed: () => ref.read(metricasProvider.notifier).recargar(),
          ),
        ],
      ),
      body: metricasAsync.when(
        // Estado: cargando
        loading: () => const Center(child: CircularProgressIndicator()),

        // Estado: error
        error: (error, stack) => Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.error_outline, size: 56, color: Colors.red),
              const SizedBox(height: 12),
              Text('Error al cargar: $error'),
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

        // Estado: datos disponibles
        data: (metricas) => ListView.builder(
          padding:     const EdgeInsets.all(12),
          itemCount:   metricas.length,
          itemBuilder: (context, i) => _TarjetaMetrica(metrica: metricas[i]),
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
    final cs = Theme.of(context).colorScheme;
    final esCritico = metrica.cpu > 85 || metrica.ram > 90;

    return Card(
      margin:  const EdgeInsets.only(bottom: 8),
      color:   esCritico ? cs.errorContainer : null,
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(children: [
              Icon(
                Icons.dns,
                color: esCritico ? cs.error : cs.primary,
                size: 20,
              ),
              const SizedBox(width: 8),
              Text(
                metrica.servidor,
                style: const TextStyle(fontWeight: FontWeight.bold),
              ),
              const Spacer(),
              if (esCritico)
                Container(
                  padding: const EdgeInsets.symmetric(
                      horizontal: 8, vertical: 2),
                  decoration: BoxDecoration(
                    color: cs.error,
                    borderRadius: BorderRadius.circular(12),
                  ),
                  child: Text('CRÍTICO',
                      style: TextStyle(
                          color: cs.onError,
                          fontSize: 10,
                          fontWeight: FontWeight.bold)),
                ),
            ]),
            const SizedBox(height: 10),
            _FilaMetrica('CPU', metrica.cpu, esCritico: metrica.cpu > 85),
            const SizedBox(height: 4),
            _FilaMetrica('RAM', metrica.ram, esCritico: metrica.ram > 90),
          ],
        ),
      ),
    );
  }
}

class _FilaMetrica extends StatelessWidget {
  final String label;
  final double valor;
  final bool   esCritico;

  const _FilaMetrica(this.label, this.valor, {this.esCritico = false});

  @override
  Widget build(BuildContext context) {
    final color = esCritico ? Colors.red : Colors.green;
    return Row(children: [
      SizedBox(width: 36, child: Text(label,
          style: const TextStyle(fontSize: 12))),
      Expanded(
        child: LinearProgressIndicator(
          value:           valor / 100,
          backgroundColor: Colors.grey.shade200,
          valueColor:      AlwaysStoppedAnimation<Color>(color),
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

---

## Providers que dependen entre sí

```dart
// El filtro es un StateProvider simple
final textoBusquedaProvider = StateProvider<String>((ref) => '');

// El resultado es un Provider que depende del filtro y de los servidores
final servidoresFiltradosProvider = Provider<List<ServidorSSH>>((ref) {
  final servidores = ref.watch(servidoresProvider);  // lista completa
  final busqueda   = ref.watch(textoBusquedaProvider); // texto del filtro

  if (busqueda.isEmpty) return servidores;

  return servidores.where((s) =>
      s.nombre.toLowerCase().contains(busqueda.toLowerCase()) ||
      s.ip.contains(busqueda)
  ).toList();
  // Cuando cualquiera de los dos cambia, este provider se recalcula
});

// En la UI — la pantalla solo escucha el provider filtrado
class PantallaServidores extends ConsumerWidget {
  const PantallaServidores({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final servidores = ref.watch(servidoresFiltradosProvider);
    final busqueda   = ref.watch(textoBusquedaProvider);

    return Column(children: [
      Padding(
        padding: const EdgeInsets.all(12),
        child: SearchBar(
          hintText: 'Buscar servidor...',
          leading:  const Icon(Icons.search),
          onChanged: (v) =>
              ref.read(textoBusquedaProvider.notifier).state = v,
        ),
      ),
      Expanded(
        child: ListView.builder(
          itemCount:   servidores.length,
          itemBuilder: (_, i) => ListTile(
            title:    Text(servidores[i].nombre),
            subtitle: Text(servidores[i].ip),
          ),
        ),
      ),
    ]);
  }
}
```

---

## Programa completo — dashboard con Riverpod

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// ── Modelos ───────────────────────────────────────────────────────

class ServidorSSH {
  final String id;
  final String nombre;
  final String ip;
  final int    puerto;
  final String usuario;
  final String so;
  final bool   ssl;
  final bool   favorito;

  const ServidorSSH({
    required this.id,
    required this.nombre,
    required this.ip,
    required this.puerto,
    required this.usuario,
    required this.so,
    required this.ssl,
    this.favorito = false,
  });

  ServidorSSH copyWith({bool? favorito}) => ServidorSSH(
    id:       id,
    nombre:   nombre,
    ip:       ip,
    puerto:   puerto,
    usuario:  usuario,
    so:       so,
    ssl:      ssl,
    favorito: favorito ?? this.favorito,
  );
}

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

// ── Providers ─────────────────────────────────────────────────────

// Lista de servidores — fuente de verdad
class ServidoresNotifier extends Notifier<List<ServidorSSH>> {
  @override
  List<ServidorSSH> build() => [
    const ServidorSSH(id: '1', nombre: 'prod-web-01', ip: '10.0.2.10', puerto: 22, usuario: 'deploy', so: 'Ubuntu 24.04', ssl: true, favorito: true),
    const ServidorSSH(id: '2', nombre: 'prod-db-01',  ip: '10.0.2.20', puerto: 22, usuario: 'postgres', so: 'Debian 12',  ssl: true),
    const ServidorSSH(id: '3', nombre: 'staging-api', ip: '10.0.3.10', puerto: 2222, usuario: 'ubuntu', so: 'Ubuntu 24.04', ssl: false),
  ];

  void toggleFavorito(String id) {
    state = state.map((s) =>
        s.id == id ? s.copyWith(favorito: !s.favorito) : s
    ).toList();
  }

  void eliminar(String id) {
    state = state.where((s) => s.id != id).toList();
  }
}

final servidoresProvider =
    NotifierProvider<ServidoresNotifier, List<ServidorSSH>>(
  ServidoresNotifier.new,
);

// Filtro de búsqueda
final busquedaProvider = StateProvider<String>((ref) => '');

// Servidores filtrados — depende de ambos
final servidoresFiltradosProvider = Provider<List<ServidorSSH>>((ref) {
  final todos     = ref.watch(servidoresProvider);
  final busqueda  = ref.watch(busquedaProvider);
  if (busqueda.isEmpty) return todos;
  final q = busqueda.toLowerCase();
  return todos.where((s) =>
      s.nombre.toLowerCase().contains(q) || s.ip.contains(q)).toList();
});

// Estadísticas derivadas
final estadisticasProvider = Provider<Map<String, int>>((ref) {
  final servidores = ref.watch(servidoresProvider);
  return {
    'total':     servidores.length,
    'ssl':       servidores.where((s) => s.ssl).length,
    'favoritos': servidores.where((s) => s.favorito).length,
  };
});

// Métricas — asíncrono
class MetricasNotifier extends AsyncNotifier<List<MetricaServidor>> {
  @override
  Future<List<MetricaServidor>> build() => _fetch();

  Future<List<MetricaServidor>> _fetch() async {
    await Future.delayed(const Duration(milliseconds: 600));
    return const [
      MetricaServidor(servidor: 'prod-web-01', cpu: 45.2, ram: 62.1, conexiones: 230),
      MetricaServidor(servidor: 'prod-db-01',  cpu: 88.1, ram: 91.2, conexiones: 80),
      MetricaServidor(servidor: 'staging-api', cpu: 22.4, ram: 41.0, conexiones: 50),
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

// ── App ───────────────────────────────────────────────────────────

void main() {
  runApp(const ProviderScope(child: AppMonitoreo()));
}

class AppMonitoreo extends StatelessWidget {
  const AppMonitoreo({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Monitoreo Riverpod',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
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

final indiceTabProvider = StateProvider<int>((ref) => 0);

// ── Tab Servidores ────────────────────────────────────────────────

class _TabServidores extends ConsumerWidget {
  const _TabServidores();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final servidores = ref.watch(servidoresFiltradosProvider);
    final stats      = ref.watch(estadisticasProvider);
    final cs         = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: Text('Servidores (${stats['total']})'),
        backgroundColor: cs.primaryContainer,
      ),
      body: Column(children: [
        // Resumen
        Padding(
          padding: const EdgeInsets.all(12),
          child: Row(children: [
            _ChipStat(label: '${stats['ssl']} SSL',       color: Colors.green),
            const SizedBox(width: 8),
            _ChipStat(label: '${stats['favoritos']} ★', color: Colors.amber),
          ]),
        ),
        // Búsqueda
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 12),
          child: SearchBar(
            hintText: 'Buscar...',
            leading: const Icon(Icons.search),
            onChanged: (v) =>
                ref.read(busquedaProvider.notifier).state = v,
            padding: const WidgetStatePropertyAll(
                EdgeInsets.symmetric(horizontal: 16)),
          ),
        ),
        const SizedBox(height: 8),
        // Lista
        Expanded(
          child: ListView.separated(
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
                subtitle: Text('${s.usuario}@${s.ip}:${s.puerto}'),
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
    );
  }
}

// ── Tab Métricas ──────────────────────────────────────────────────

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
        error:   (e, _) => Center(
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
                  style: TextStyle(
                      fontSize: 12, color: cs.onSurfaceVariant)),
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
          style: TextStyle(fontSize: 12,
              color: color, fontWeight: FontWeight.w600)),
    ]);
  }
}

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