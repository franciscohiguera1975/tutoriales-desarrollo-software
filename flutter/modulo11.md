# Tutorial Flutter — Página 11
## Módulo 3 · Estado y Navegación
### GoRouter — rutas, parámetros, guards y navegación anidada

---

## ¿Por qué GoRouter?

Flutter tiene un sistema de navegación nativo (`Navigator`),
pero para apps con múltiples pantallas se vuelve verboso.
GoRouter añade rutas declarativas con URL, deep links y guards:

```
Navigator nativo                  GoRouter
──────────────────────            ─────────────────────────────────────
Navigator.push(context, route)    context.go('/detalle/42')
Sin soporte URL real              URL reales — deep linking
Guards manuales y dispersos       redirect() centralizado
Parámetros como constructores     Parámetros en la URL /items/:id
Sin historial de navegación web   Historial completo en web
```

### Tabla comparativa: Navigator vs GoRouter

```
Característica              Navigator nativo           GoRouter
─────────────────────────   ────────────────────────   ──────────────────────────
Definición de rutas         Imperativa (push/pop)      Declarativa (lista de GoRoute)
Deep linking                Manual, complejo           Incorporado
URLs en web                 No                         Sí
Guards / redirecciones      Dispersos en widgets       redirect() centralizado
Parámetros de ruta          Constructor del widget     state.pathParameters['id']
Query params                Manual                     state.uri.queryParameters
Shell / tab persistente     Muy difícil                ShellRoute
Integración con Riverpod    Compleja                   ref.read() en redirect
```

### Dependencia

```yaml
# pubspec.yaml
dependencies:
  go_router: ^14.6.2
  flutter_riverpod: ^3.3.0
```

---

## Patrones clave — de un vistazo

Referencia rápida de los patrones del módulo. Léelos ahora y vuelve
a consultarlos conforme avanzas en cada paso.

---

### Bloque 1 — Configuración básica + `MaterialApp.router`

```dart
// lib/router/app_router.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

final appRouter = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(
      path:    '/',
      builder: (context, state) => const PantallaInicio(),
    ),
    GoRoute(
      path:    '/servidores',
      builder: (context, state) => const PantallaServidores(),
    ),
  ],
);

// En main():
MaterialApp.router(          // .router en lugar de home:
  routerConfig: appRouter,
  theme: ThemeData(useMaterial3: true),
)
```

> **Mini-ejercicio ⏱ 5 min**
> Agrega una tercera ruta `/metricas` que muestre un `Scaffold` con
> `AppBar(title: Text('Métricas'))`. Navega a ella desde `PantallaInicio`
> con un `FilledButton` y `context.go('/metricas')`.

---

### Bloque 2 — `context.go` vs `context.push` vs `context.pop`

```dart
// go() — reemplaza la pantalla (sin botón de retroceso)
context.go('/servidores');

// push() — apila la pantalla (con botón de retroceso)
context.push('/servidores/42');

// pop() — retrocede a la pantalla anterior
context.pop();

// pop con valor de retorno
context.pop({'resultado': 'guardado'});
final resultado = await context.push<Map>('/editar/42');
```

> **Mini-ejercicio ⏱ 5 min**
> Desde `PantallaServidores`, navega a `/` con `context.go` — ¿hay botón
> de retroceso? Ahora prueba con `context.push('/')` — ¿qué cambia?

---

### Bloque 3 — Parámetros de ruta y query

```dart
// Path parameter: /servidores/:id
GoRoute(
  path:    '/servidores/:id',
  builder: (context, state) {
    final id = state.pathParameters['id']!;
    return PantallaDetalle(id: id);
  },
),

// Query parameters: /servidores?filtro=ssl&pagina=2
final filtro = state.uri.queryParameters['filtro'];
final pagina = int.tryParse(state.uri.queryParameters['pagina'] ?? '1') ?? 1;

// Extras — objeto Dart completo (no se serializa en la URL)
final servidor = state.extra as ServidorSSH?;
```

> **Mini-ejercicio ⏱ 5 min**
> Agrega al router una ruta `/servidores/:id/logs` que muestre un `Scaffold`
> con título `'Logs de $id'`. Navega a ella con
> `context.push('/servidores/prod-web-01/logs')`.

---

### Bloque 4 — Rutas anidadas con `ShellRoute`

```dart
ShellRoute(
  builder: (context, state, child) => ScaffoldConNav(child: child),
  routes: [
    GoRoute(path: '/servidores', builder: (_, __) => const PantallaServidores()),
    GoRoute(path: '/metricas',   builder: (_, __) => const PantallaMetricas()),
    GoRoute(path: '/ajustes',    builder: (_, __) => const PantallaAjustes()),
  ],
),

// ScaffoldConNav detecta la ruta activa
int _indiceActivo(BuildContext context) {
  final loc = GoRouterState.of(context).uri.path;
  if (loc.startsWith('/metricas')) return 1;
  if (loc.startsWith('/ajustes'))  return 2;
  return 0;
}
```

> **Mini-ejercicio ⏱ 5 min**
> Agrega una cuarta tab `/alertas` al `ShellRoute`. Actualiza
> `_indiceActivo` y añade la `NavigationDestination` correspondiente
> con `Icons.notifications_outlined`.

---

### Bloque 5 — Guards con `redirect`

```dart
final appRouter = GoRouter(
  initialLocation: '/servidores',
  redirect: (context, state) {
    // Se llama antes de cada navegación
    final estaAutenticado = /* leer del provider */;
    final enLogin = state.matchedLocation == '/login';

    if (!estaAutenticado && !enLogin) return '/login';
    if (estaAutenticado && enLogin)   return '/servidores';
    return null; // null = no redirigir
  },
  routes: [ /* ... */ ],
);
```

> **Mini-ejercicio ⏱ 5 min**
> Cambia el guard para que también permita acceder a `/registro` sin
> autenticar. Pista: `final enPaginaPublica = enLogin || state.matchedLocation == '/registro';`.

---

### Bloque 6 — GoRouter con Riverpod

```dart
// El router escucha el provider de auth
final routerProvider = Provider<GoRouter>((ref) {
  final authState = ref.watch(authProvider);

  return GoRouter(
    initialLocation: '/servidores',
    refreshListenable: ...,   // no necesario con Riverpod
    redirect: (context, state) {
      final autenticado = authState is Autenticado;
      if (!autenticado && state.matchedLocation != '/login') return '/login';
      if (autenticado && state.matchedLocation == '/login') return '/servidores';
      return null;
    },
    routes: [ /* ... */ ],
  );
});

// En la UI:
final router = ref.watch(routerProvider);
return MaterialApp.router(routerConfig: router);
```

> **Mini-ejercicio ⏱ 5 min**
> ¿Qué pasa si no usas `ref.watch` y usas `ref.read` para el `authState`
> en el `redirect`? Prueba el cambio y observa qué ocurre con la navegación
> al hacer logout.

---

## Crea el proyecto

```bash
flutter create modulo11_gorouter
cd modulo11_gorouter
```

`pubspec.yaml`:
```yaml
dependencies:
  flutter:
    sdk: flutter
  go_router: ^14.6.2
  flutter_riverpod: ^3.3.0
```

```bash
flutter pub get
```

Estructura de carpetas:
```
modulo11_gorouter/
├── lib/
│   ├── main.dart
│   ├── router/
│   │   └── app_router.dart      ← Paso 1
│   ├── models/                  ← Paso 2
│   ├── providers/               ← Paso 4
│   └── screens/
│       ├── pantalla_inicio.dart     ← Paso 1
│       ├── pantalla_servidores.dart ← Paso 2
│       ├── pantalla_detalle.dart    ← Paso 2
│       ├── pantalla_metricas.dart   ← Paso 3
│       ├── pantalla_ajustes.dart    ← Paso 3
│       └── pantalla_login.dart      ← Paso 4
```

```bash
flutter run
```

---

## Paso 1 — Configuración básica + navegación con `context.go`

**Objetivo:** arrancar la app con GoRouter. Dos pantallas, sin parámetros.

Crea `lib/router/app_router.dart`:
```dart
// lib/router/app_router.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import '../screens/pantalla_inicio.dart';
import '../screens/pantalla_servidores.dart';

final appRouter = GoRouter(
  initialLocation: '/',
  debugLogDiagnostics: true,  // imprime cada navegación en la consola
  routes: [
    GoRoute(
      path:    '/',
      name:    'inicio',
      builder: (context, state) => const PantallaInicio(),
    ),
    GoRoute(
      path:    '/servidores',
      name:    'servidores',
      builder: (context, state) => const PantallaServidores(),
    ),
  ],
);
```

Crea `lib/screens/pantalla_inicio.dart`:
```dart
// lib/screens/pantalla_inicio.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

class PantallaInicio extends StatelessWidget {
  const PantallaInicio({super.key});

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Monitor SSH'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(Icons.terminal, size: 64, color: cs.primary),
            const SizedBox(height: 16),
            const Text('Dashboard de Monitoreo',
                style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            Text('Gestiona tus servidores SSH',
                style: TextStyle(color: cs.onSurfaceVariant)),
            const SizedBox(height: 32),
            FilledButton.icon(
              // context.go() — navega SIN apilar (no hay botón "atrás")
              onPressed: () => context.go('/servidores'),
              icon:  const Icon(Icons.dns),
              label: const Text('Ver servidores'),
            ),
          ],
        ),
      ),
    );
  }
}
```

Crea `lib/screens/pantalla_servidores.dart` (simple por ahora):
```dart
// lib/screens/pantalla_servidores.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

class PantallaServidores extends StatelessWidget {
  const PantallaServidores({super.key});

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    final servidores = ['prod-web-01', 'prod-db-01', 'staging-api'];

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Servidores'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: ListView.builder(
        itemCount:   servidores.length,
        itemBuilder: (context, i) => ListTile(
          leading: const Icon(Icons.dns),
          title:   Text(servidores[i]),
          onTap: () {
            // context.push() — apila la pantalla (aparece botón "atrás")
            context.push('/servidores/${servidores[i]}');
          },
        ),
      ),
    );
  }
}
```

`lib/main.dart`:
```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'router/app_router.dart';

void main() {
  runApp(const AppMonitoreo());
}

class AppMonitoreo extends StatelessWidget {
  const AppMonitoreo({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      title:        'Monitor SSH',
      debugShowCheckedModeBanner: false,
      routerConfig: appRouter,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0D47A1)),
        useMaterial3: true,
      ),
    );
  }
}
```

**Nota:** `debugLogDiagnostics: true` imprime en la consola cada navegación — muy útil para depurar.

### Prueba esto

- Ejecuta la app — aparece la `PantallaInicio`
- Pulsa "Ver servidores" — navega a `/servidores` con `context.go`: **no aparece botón de retroceso** (go reemplaza el stack)
- Observa la consola — `debugLogDiagnostics` imprime `Routing to /servidores`
- Cambia `context.go` por `context.push` — ahora **sí aparece botón de retroceso**
- Vuelve a `context.go` — el comportamiento correcto para la navegación principal

---

## Convierte `main.dart` en selector de pasos

Igual que en los módulos 08, 09, 10: variable `const int paso = 1` con un switch.

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'router/app_router.dart';
import 'router/app_router_paso2.dart';
import 'router/app_router_paso3.dart';
import 'router/app_router_paso4.dart';
import 'router/app_router_paso5.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1  Rutas básicas + context.go / push / pop             │
// │  2  Paso 2  pathParameters + pantalla de detalle                │
// │  3  Paso 3  queryParameters + extras + ShellRoute               │
// │  4  Paso 4  ShellRoute completo + NavigationBar persistente     │
// │  5  Paso 5  Guard redirect + pantalla de login + Riverpod       │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 1;

void main() {
  runApp(
    ProviderScope(
      child: AppMonitoreo(paso: paso),
    ),
  );
}

class AppMonitoreo extends StatelessWidget {
  final int paso;
  const AppMonitoreo({super.key, required this.paso});

  @override
  Widget build(BuildContext context) {
    final router = switch (paso) {
      1 => appRouter,
      2 => appRouterPaso2,
      3 => appRouterPaso3,
      4 => appRouterPaso4,
      5 => appRouterPaso5(context),
      _ => appRouter,
    };

    return MaterialApp.router(
      title:        'Monitor SSH',
      debugShowCheckedModeBanner: false,
      routerConfig: router,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0D47A1)),
        useMaterial3: true,
      ),
    );
  }
}
```

**Nota:** A diferencia de los módulos anteriores, aquí cada paso tiene su propio archivo de router (`app_router_pasoN.dart`) porque GoRouter vive fuera del árbol de widgets. El `main.dart` selecciona cuál usar.

---

## Paso 2 — `pathParameters` + pantalla de detalle

**Objetivo:** navegar con `/servidores/:id` y mostrar el detalle del servidor.

Crea `lib/models/servidor_ssh.dart` (igual que módulos anteriores):
```dart
class ServidorSSH {
  final String id;
  final String nombre;
  final String ip;
  final int    puerto;
  final bool   ssl;

  const ServidorSSH({
    required this.id,
    required this.nombre,
    required this.ip,
    required this.puerto,
    required this.ssl,
  });
}

// Lista simulada — en una app real vendría de un provider
const servidoresSimulados = [
  ServidorSSH(id: '1', nombre: 'prod-web-01', ip: '10.0.2.10',   puerto: 22,   ssl: true),
  ServidorSSH(id: '2', nombre: 'prod-db-01',  ip: '10.0.2.20',   puerto: 22,   ssl: true),
  ServidorSSH(id: '3', nombre: 'staging-api', ip: '10.0.3.10',   puerto: 2222, ssl: false),
];
```

Crea `lib/screens/pantalla_detalle.dart`:
```dart
// lib/screens/pantalla_detalle.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import '../models/servidor_ssh.dart';

class PantallaDetalle extends StatelessWidget {
  final String      id;
  final ServidorSSH? servidor; // puede venir por extras

  const PantallaDetalle({super.key, required this.id, this.servidor});

  @override
  Widget build(BuildContext context) {
    // Si no viene por extras, buscar en la lista simulada
    final srv = servidor ??
        servidoresSimulados.where((s) => s.id == id).firstOrNull;

    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           Text('Detalle: ${srv?.nombre ?? id}'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
      ),
      body: srv == null
          ? Center(child: Text('Servidor $id no encontrado'))
          : Padding(
              padding: const EdgeInsets.all(16),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  _Fila('ID',       srv.id),
                  _Fila('Nombre',   srv.nombre),
                  _Fila('IP',       srv.ip),
                  _Fila('Puerto',   srv.puerto.toString()),
                  _Fila('SSL',      srv.ssl ? 'Activo' : 'Inactivo'),
                  const SizedBox(height: 24),
                  Row(children: [
                    OutlinedButton.icon(
                      onPressed: () => context.pop(),
                      icon:  const Icon(Icons.arrow_back),
                      label: const Text('Volver'),
                    ),
                    const SizedBox(width: 12),
                    FilledButton.icon(
                      onPressed: () => context.push('/servidores/${srv.id}/logs'),
                      icon:  const Icon(Icons.list_alt),
                      label: const Text('Ver logs'),
                    ),
                  ]),
                ],
              ),
            ),
    );
  }
}

class _Fila extends StatelessWidget {
  final String label;
  final String valor;
  const _Fila(this.label, this.valor);

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 6),
      child: Row(children: [
        SizedBox(
          width: 70,
          child: Text(label,
              style: TextStyle(color: cs.onSurfaceVariant,
                  fontWeight: FontWeight.w600, fontSize: 12)),
        ),
        Text(valor, style: const TextStyle(fontSize: 15)),
      ]),
    );
  }
}
```

Crea `lib/router/app_router_paso2.dart`:
```dart
// lib/router/app_router_paso2.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import '../screens/pantalla_inicio.dart';
import '../screens/pantalla_servidores.dart';
import '../screens/pantalla_detalle.dart';
import '../models/servidor_ssh.dart';

final appRouterPaso2 = GoRouter(
  initialLocation: '/',
  debugLogDiagnostics: true,
  routes: [
    GoRoute(
      path:    '/',
      builder: (context, state) => const PantallaInicio(),
    ),
    GoRoute(
      path:    '/servidores',
      builder: (context, state) => const PantallaServidores(),
      routes: [
        // Ruta hija: /servidores/:id
        GoRoute(
          path:    ':id',   // relativa — ruta completa: /servidores/:id
          builder: (context, state) {
            final id       = state.pathParameters['id']!;
            final servidor = state.extra as ServidorSSH?;
            return PantallaDetalle(id: id, servidor: servidor);
          },
        ),
        // Ruta hija: /servidores/:id/logs
        GoRoute(
          path:    ':id/logs',
          builder: (context, state) {
            final id = state.pathParameters['id']!;
            return Scaffold(
              appBar: AppBar(title: Text('Logs de $id')),
              body:   Center(child: Text('Logs del servidor $id')),
            );
          },
        ),
      ],
    ),
  ],
);
```

Actualiza `lib/screens/pantalla_servidores.dart` para navegar con extras:
```dart
// Navegar con extras — pasa el objeto completo evitando una segunda búsqueda
context.push(
  '/servidores/${s.id}',
  extra: s,   // ServidorSSH completo
);
```

### Agrega al `main.dart`

Cambia `const int paso = 2;` y guarda.

### Prueba esto

- En la lista, toca un servidor — navega a `/servidores/1` y aparece el detalle
- Observa la consola: `GoRouter: Routing to /servidores/1`
- Toca "Ver logs" — navega a `/servidores/1/logs` (ruta anidada)
- Toca "Volver" en detalle — `context.pop()` regresa a la lista
- Toca el botón nativo "atrás" del sistema — también funciona
- Cambia `context.push('/servidores/${s.id}', extra: s)` por `context.go(...)` — ¿qué pasa con el botón de retroceso en detalle?

---

## Paso 3 — `queryParameters` + `extras`

**Objetivo:** añadir filtrado por query params en la lista y pasar objetos completos con extras.

Crea `lib/router/app_router_paso3.dart`:
```dart
// lib/router/app_router_paso3.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import '../screens/pantalla_inicio.dart';
import '../screens/pantalla_servidores_filtro.dart';
import '../screens/pantalla_detalle.dart';
import '../models/servidor_ssh.dart';

final appRouterPaso3 = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(
      path:    '/',
      builder: (context, state) => const PantallaInicio(),
    ),
    GoRoute(
      path:    '/servidores',
      builder: (context, state) {
        // Query parameters — /servidores?soloSSL=true
        final soloSSL = state.uri.queryParameters['soloSSL'] == 'true';
        return PantallaServidoresFiltro(soloSSL: soloSSL);
      },
    ),
    GoRoute(
      path:    '/servidores/:id',
      builder: (context, state) {
        final id       = state.pathParameters['id']!;
        final servidor = state.extra as ServidorSSH?;
        return PantallaDetalle(id: id, servidor: servidor);
      },
    ),
  ],
);
```

Crea `lib/screens/pantalla_servidores_filtro.dart`:
```dart
// lib/screens/pantalla_servidores_filtro.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import '../models/servidor_ssh.dart';

class PantallaServidoresFiltro extends StatelessWidget {
  final bool soloSSL;
  const PantallaServidoresFiltro({super.key, this.soloSSL = false});

  @override
  Widget build(BuildContext context) {
    final filtrados = soloSSL
        ? servidoresSimulados.where((s) => s.ssl).toList()
        : servidoresSimulados;

    return Scaffold(
      appBar: AppBar(
        title:   Text('Servidores${soloSSL ? ' (SSL)' : ''}'),
        actions: [
          // Toggle filtro SSL — cambia la URL con query param
          IconButton(
            icon:    Icon(soloSSL ? Icons.lock : Icons.lock_open),
            tooltip: soloSSL ? 'Ver todos' : 'Solo SSL',
            onPressed: () => soloSSL
                ? context.go('/servidores')
                : context.go('/servidores?soloSSL=true'),
          ),
        ],
      ),
      body: ListView.builder(
        itemCount:   filtrados.length,
        itemBuilder: (context, i) {
          final s = filtrados[i];
          return ListTile(
            leading: Icon(Icons.dns, color: s.ssl ? Colors.green : Colors.grey),
            title:   Text(s.nombre),
            subtitle: Text(s.ip),
            onTap: () => context.push(
              '/servidores/${s.id}',
              extra: s,   // pasa el objeto completo
            ),
          );
        },
      ),
    );
  }
}
```

### Agrega al `main.dart`

Cambia `const int paso = 3;` y guarda.

### Prueba esto

- Ejecuta — aparece la lista completa (sin filtro)
- Pulsa el icono de candado — navega a `/servidores?soloSSL=true`: solo aparecen los SSL
- Pulsa el candado de nuevo — regresa a `/servidores`: aparecen todos
- Toca un servidor y observa que el detalle se carga sin una segunda búsqueda (lo recibe por `extras`)
- En la consola, compara las URLs impresas por `debugLogDiagnostics`

---

## Paso 4 — `ShellRoute` + `NavigationBar` persistente

**Objetivo:** NavigationBar que no se reconstruye al cambiar de tab.

Crea `lib/screens/scaffold_con_nav.dart`:
```dart
// lib/screens/scaffold_con_nav.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

class ScaffoldConNav extends StatelessWidget {
  final Widget child;
  const ScaffoldConNav({super.key, required this.child});

  // Detecta la ruta activa para resaltar la tab correcta
  int _indiceActivo(BuildContext context) {
    final loc = GoRouterState.of(context).uri.path;
    if (loc.startsWith('/metricas')) return 1;
    if (loc.startsWith('/ajustes'))  return 2;
    return 0; // /servidores
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: child,    // child cambia, el Scaffold NO se reconstruye
      bottomNavigationBar: NavigationBar(
        selectedIndex:         _indiceActivo(context),
        onDestinationSelected: (i) {
          switch (i) {
            case 0: context.go('/servidores');
            case 1: context.go('/metricas');
            case 2: context.go('/ajustes');
          }
        },
        destinations: const [
          NavigationDestination(
            icon: Icon(Icons.dns_outlined), selectedIcon: Icon(Icons.dns),
            label: 'Servidores',
          ),
          NavigationDestination(
            icon: Icon(Icons.bar_chart_outlined), selectedIcon: Icon(Icons.bar_chart),
            label: 'Métricas',
          ),
          NavigationDestination(
            icon: Icon(Icons.settings_outlined), selectedIcon: Icon(Icons.settings),
            label: 'Ajustes',
          ),
        ],
      ),
    );
  }
}
```

Crea `lib/screens/pantalla_metricas.dart` y `lib/screens/pantalla_ajustes.dart` simples:
```dart
// pantalla_metricas.dart
import 'package:flutter/material.dart';

class PantallaMetricas extends StatelessWidget {
  const PantallaMetricas({super.key});

  @override
  Widget build(BuildContext context) => Scaffold(
    body: const Center(child: Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        Icon(Icons.bar_chart, size: 56),
        SizedBox(height: 8),
        Text('Métricas de servidores', style: TextStyle(fontSize: 18)),
      ],
    )),
  );
}
```

```dart
// pantalla_ajustes.dart
import 'package:flutter/material.dart';

class PantallaAjustes extends StatelessWidget {
  const PantallaAjustes({super.key});

  @override
  Widget build(BuildContext context) => Scaffold(
    body: const Center(child: Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        Icon(Icons.settings, size: 56),
        SizedBox(height: 8),
        Text('Ajustes de la app', style: TextStyle(fontSize: 18)),
      ],
    )),
  );
}
```

Crea `lib/router/app_router_paso4.dart`:
```dart
// lib/router/app_router_paso4.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import '../screens/scaffold_con_nav.dart';
import '../screens/pantalla_servidores.dart';
import '../screens/pantalla_detalle.dart';
import '../screens/pantalla_metricas.dart';
import '../screens/pantalla_ajustes.dart';
import '../models/servidor_ssh.dart';

final appRouterPaso4 = GoRouter(
  initialLocation: '/servidores',
  debugLogDiagnostics: true,
  routes: [
    // ShellRoute — mantiene ScaffoldConNav vivo entre rutas hijas
    ShellRoute(
      builder: (context, state, child) => ScaffoldConNav(child: child),
      routes: [
        GoRoute(
          path:    '/servidores',
          builder: (_, __) => const PantallaServidores(),
          routes: [
            GoRoute(
              path:    ':id',
              builder: (context, state) {
                final id       = state.pathParameters['id']!;
                final servidor = state.extra as ServidorSSH?;
                return PantallaDetalle(id: id, servidor: servidor);
              },
            ),
          ],
        ),
        GoRoute(
          path:    '/metricas',
          builder: (_, __) => const PantallaMetricas(),
        ),
        GoRoute(
          path:    '/ajustes',
          builder: (_, __) => const PantallaAjustes(),
        ),
      ],
    ),
  ],
);
```

### Agrega al `main.dart`

Cambia `const int paso = 4;` y guarda.

### Prueba esto

- Ejecuta — aparece la app con `NavigationBar` de 3 tabs
- Navega entre Servidores / Métricas / Ajustes — la `NavigationBar` **no parpadea** (ShellRoute la mantiene)
- Toca un servidor — navega al detalle; el botón de retroceso vuelve a `/servidores`
- Compara: sin `ShellRoute` cada tab recrearía el `Scaffold` entero. Con `ShellRoute`, solo el `child` cambia
- Añade un `print('construyendo ScaffoldConNav')` en `build()` de `ScaffoldConNav` — comprueba que solo se llama una vez al inicio

---

## Paso 5 — Guard `redirect` + login + Riverpod

**Objetivo:** redirigir a `/login` si no está autenticado; volver al app tras login.

Crea `lib/providers/auth_provider.dart`:
```dart
// lib/providers/auth_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Estado de autenticación con sealed class
sealed class AuthState { const AuthState(); }
class SinSesion   extends AuthState { const SinSesion(); }
class Cargando    extends AuthState { const Cargando(); }
class Autenticado extends AuthState {
  final String usuario;
  const Autenticado(this.usuario);
}
class ErrorAuth   extends AuthState {
  final String mensaje;
  const ErrorAuth(this.mensaje);
}

class AuthNotifier extends Notifier<AuthState> {
  @override
  AuthState build() => const SinSesion();

  Future<void> login(String usuario, String password) async {
    state = const Cargando();
    await Future.delayed(const Duration(seconds: 1));

    if (usuario == 'admin' && password == 'admin123') {
      state = Autenticado(usuario);
    } else {
      state = const ErrorAuth('Usuario o contraseña incorrectos');
      await Future.delayed(const Duration(seconds: 2));
      state = const SinSesion();
    }
  }

  void logout() => state = const SinSesion();
}

final authProvider =
    NotifierProvider<AuthNotifier, AuthState>(AuthNotifier.new);
```

Crea `lib/screens/pantalla_login.dart`:
```dart
// lib/screens/pantalla_login.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../providers/auth_provider.dart';

class PantallaLogin extends ConsumerStatefulWidget {
  const PantallaLogin({super.key});

  @override
  ConsumerState<PantallaLogin> createState() => _PantallaLoginState();
}

class _PantallaLoginState extends ConsumerState<PantallaLogin> {
  final _formKey  = GlobalKey<FormState>();
  final _ctrlUser = TextEditingController();
  final _ctrlPass = TextEditingController();

  @override
  void dispose() {
    _ctrlUser.dispose();
    _ctrlPass.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final authState = ref.watch(authProvider);
    final cs        = Theme.of(context).colorScheme;

    return Scaffold(
      body: Center(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(32),
          child: Form(
            key: _formKey,
            child: Column(
              mainAxisSize: MainAxisSize.min,
              children: [
                Icon(Icons.terminal, size: 64, color: cs.primary),
                const SizedBox(height: 16),
                const Text('Monitor SSH',
                    style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold)),
                const SizedBox(height: 32),

                TextFormField(
                  controller: _ctrlUser,
                  decoration: const InputDecoration(
                    labelText:  'Usuario',
                    prefixIcon: Icon(Icons.person_outline),
                    border:     OutlineInputBorder(),
                  ),
                  validator: (v) => v == null || v.isEmpty ? 'Requerido' : null,
                ),
                const SizedBox(height: 12),

                TextFormField(
                  controller:  _ctrlPass,
                  obscureText: true,
                  decoration: const InputDecoration(
                    labelText:  'Contrasena',
                    prefixIcon: Icon(Icons.lock_outline),
                    border:     OutlineInputBorder(),
                  ),
                  validator: (v) => v == null || v.isEmpty ? 'Requerida' : null,
                ),
                const SizedBox(height: 8),

                if (authState is ErrorAuth)
                  Padding(
                    padding: const EdgeInsets.only(bottom: 8),
                    child: Text(authState.mensaje,
                        style: TextStyle(color: cs.error)),
                  ),
                const SizedBox(height: 8),

                SizedBox(
                  width: double.infinity,
                  child: FilledButton(
                    onPressed: authState is Cargando
                        ? null
                        : () {
                            if (!_formKey.currentState!.validate()) return;
                            ref.read(authProvider.notifier).login(
                                _ctrlUser.text, _ctrlPass.text);
                          },
                    child: authState is Cargando
                        ? const SizedBox(
                            width: 20, height: 20,
                            child: CircularProgressIndicator(strokeWidth: 2),
                          )
                        : const Text('Iniciar sesion'),
                  ),
                ),
                const SizedBox(height: 8),
                Text('Pista: admin / admin123',
                    style: TextStyle(fontSize: 11, color: cs.onSurfaceVariant)),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

Crea `lib/router/app_router_paso5.dart` — el router recibe `WidgetRef`:
```dart
// lib/router/app_router_paso5.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../providers/auth_provider.dart';
import '../screens/scaffold_con_nav.dart';
import '../screens/pantalla_servidores.dart';
import '../screens/pantalla_detalle.dart';
import '../screens/pantalla_metricas.dart';
import '../screens/pantalla_ajustes.dart';
import '../screens/pantalla_login.dart';
import '../models/servidor_ssh.dart';

// Función que crea el router con acceso al WidgetRef (para el guard)
GoRouter appRouterPaso5(WidgetRef ref) => GoRouter(
  initialLocation: '/servidores',
  debugLogDiagnostics: true,
  redirect: (context, state) {
    final authState     = ref.read(authProvider);
    final autenticado   = authState is Autenticado;
    final enLogin       = state.matchedLocation == '/login';

    // No autenticado y no está en /login → ir al login
    if (!autenticado && !enLogin) return '/login';
    // Autenticado y está en /login → ir a la app
    if (autenticado && enLogin)   return '/servidores';
    // Sin redirección
    return null;
  },
  routes: [
    ShellRoute(
      builder: (context, state, child) => ScaffoldConNav(child: child),
      routes: [
        GoRoute(
          path:    '/servidores',
          builder: (_, __) => const PantallaServidores(),
          routes: [
            GoRoute(
              path:    ':id',
              builder: (context, state) => PantallaDetalle(
                id:       state.pathParameters['id']!,
                servidor: state.extra as ServidorSSH?,
              ),
            ),
          ],
        ),
        GoRoute(path: '/metricas', builder: (_, __) => const PantallaMetricas()),
        GoRoute(path: '/ajustes',  builder: (_, __) => const PantallaAjustes()),
      ],
    ),
    GoRoute(
      path:    '/login',
      builder: (_, __) => const PantallaLogin(),
    ),
  ],
);
```

Actualiza `lib/main.dart` para el paso 5 — el `AppMonitoreo` debe ser `ConsumerWidget`:
```dart
// Para el paso 5, AppMonitoreo es ConsumerWidget (accede al ref para el router)
class AppMonitoreo extends ConsumerWidget {
  final int paso;
  const AppMonitoreo({super.key, required this.paso});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Observamos el estado de auth para que el router se regenere al cambiar
    ref.watch(authProvider);

    final router = switch (paso) {
      1 => appRouter,
      2 => appRouterPaso2,
      3 => appRouterPaso3,
      4 => appRouterPaso4,
      5 => appRouterPaso5(ref),
      _ => appRouter,
    };

    return MaterialApp.router(
      title:        'Monitor SSH',
      debugShowCheckedModeBanner: false,
      routerConfig: router,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0D47A1)),
        useMaterial3: true,
      ),
    );
  }
}
```

**Nota técnica:** `ref.watch(authProvider)` en `AppMonitoreo` hace que el widget se reconstruya cuando cambia el estado de auth, lo que provoca que `appRouterPaso5(ref)` se llame de nuevo, re-evaluando el `redirect`. Así el guard funciona reactivamente.

### Agrega al `main.dart`

Cambia `const int paso = 5;` y guarda.

### Prueba esto

- Ejecuta — el guard redirige inmediatamente a `/login`
- Escribe credenciales incorrectas — aparece el mensaje de error en rojo
- Espera 2 segundos — el formulario regresa a estado inicial
- Escribe `admin` / `admin123` — el guard detecta el cambio y redirige a `/servidores`
- Ve a Ajustes y agrega un botón "Cerrar sesión" que llame a `ref.read(authProvider.notifier).logout()` — el guard redirige a `/login` automáticamente

---

## `main.dart` completo — referencia

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'providers/auth_provider.dart';
import 'router/app_router.dart';
import 'router/app_router_paso2.dart';
import 'router/app_router_paso3.dart';
import 'router/app_router_paso4.dart';
import 'router/app_router_paso5.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1  Rutas básicas + context.go / push / pop             │
// │  2  Paso 2  pathParameters + pantalla de detalle                │
// │  3  Paso 3  queryParameters + extras + filtro SSL               │
// │  4  Paso 4  ShellRoute + NavigationBar persistente              │
// │  5  Paso 5  Guard redirect + login + Riverpod                   │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 5;

void main() => runApp(const ProviderScope(child: AppMonitoreo(paso: paso)));

class AppMonitoreo extends ConsumerWidget {
  final int paso;
  const AppMonitoreo({super.key, required this.paso});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    ref.watch(authProvider); // reaccionar a cambios de auth

    final router = switch (paso) {
      1 => appRouter,
      2 => appRouterPaso2,
      3 => appRouterPaso3,
      4 => appRouterPaso4,
      5 => appRouterPaso5(ref),
      _ => appRouter,
    };

    return MaterialApp.router(
      title:        'Monitor SSH',
      debugShowCheckedModeBanner: false,
      routerConfig: router,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0D47A1)),
        useMaterial3: true,
      ),
    );
  }
}
```

---

## Proyecto — App de Monitoreo con GoRouter + Riverpod

**Descripción:** app completa que combina todo lo aprendido en la página 11 con lo del módulo anterior (Riverpod). El proyecto final no usa el selector de pasos — es una app independiente.

**Características:**
- Login con `AuthNotifier` (Riverpod)
- `ShellRoute` con 3 tabs: Servidores / Métricas / Ajustes (con logout en Ajustes)
- `/servidores/:id` con detalle pasando objeto por extras
- Guard en el router que redirige al login si no está autenticado
- `ScaffoldConNav` persistente entre tabs

---

### `lib/main.dart` (proyecto)

```dart
// lib/main.dart — proyecto final (sin selector de pasos)
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'providers/auth_provider.dart';
import 'router/app_router.dart';

void main() => runApp(const ProviderScope(child: AppMonitoreo()));

class AppMonitoreo extends ConsumerWidget {
  const AppMonitoreo({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    ref.watch(authProvider); // reaccionar a cambios de auth
    final router = ref.watch(routerProvider);

    return MaterialApp.router(
      title:        'Monitor SSH',
      debugShowCheckedModeBanner: false,
      routerConfig: router,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: const Color(0xFF0D47A1)),
        useMaterial3: true,
      ),
    );
  }
}
```

---

### `lib/providers/auth_provider.dart`

```dart
// lib/providers/auth_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

sealed class AuthState { const AuthState(); }
class SinSesion   extends AuthState { const SinSesion(); }
class Cargando    extends AuthState { const Cargando(); }
class Autenticado extends AuthState {
  final String usuario;
  const Autenticado(this.usuario);
}
class ErrorAuth extends AuthState {
  final String mensaje;
  const ErrorAuth(this.mensaje);
}

class AuthNotifier extends Notifier<AuthState> {
  @override
  AuthState build() => const SinSesion();

  Future<void> login(String usuario, String password) async {
    state = const Cargando();
    await Future.delayed(const Duration(milliseconds: 800));
    if (usuario == 'admin' && password == 'admin123') {
      state = Autenticado(usuario);
    } else {
      state = const ErrorAuth('Usuario o contrasena incorrectos');
      await Future.delayed(const Duration(seconds: 2));
      state = const SinSesion();
    }
  }

  void logout() => state = const SinSesion();
}

final authProvider =
    NotifierProvider<AuthNotifier, AuthState>(AuthNotifier.new);
```

---

### `lib/models/servidor_ssh.dart`

```dart
// lib/models/servidor_ssh.dart
class ServidorSSH {
  final String id;
  final String nombre;
  final String ip;
  final int    puerto;
  final bool   ssl;

  const ServidorSSH({
    required this.id,
    required this.nombre,
    required this.ip,
    required this.puerto,
    required this.ssl,
  });
}

const servidoresSimulados = [
  ServidorSSH(id: '1', nombre: 'prod-web-01', ip: '10.0.2.10',   puerto: 22,   ssl: true),
  ServidorSSH(id: '2', nombre: 'prod-db-01',  ip: '10.0.2.20',   puerto: 22,   ssl: true),
  ServidorSSH(id: '3', nombre: 'staging-api', ip: '10.0.3.10',   puerto: 2222, ssl: false),
];
```

---

### `lib/router/app_router.dart` (proyecto — con guard y Riverpod)

```dart
// lib/router/app_router.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../providers/auth_provider.dart';
import '../models/servidor_ssh.dart';
import '../screens/scaffold_con_nav.dart';
import '../screens/pantalla_login.dart';
import '../screens/pantalla_servidores.dart';
import '../screens/pantalla_detalle.dart';
import '../screens/pantalla_metricas.dart';
import '../screens/pantalla_ajustes.dart';

// Provider del router — se reconstruye con ref.watch(authProvider) en AppMonitoreo
final routerProvider = Provider<GoRouter>((ref) {
  return GoRouter(
    initialLocation: '/servidores',
    debugLogDiagnostics: true,
    redirect: (context, state) {
      final authState   = ref.read(authProvider);
      final autenticado = authState is Autenticado;
      final enLogin     = state.matchedLocation == '/login';

      if (!autenticado && !enLogin) return '/login';
      if (autenticado && enLogin)   return '/servidores';
      return null;
    },
    routes: [
      GoRoute(
        path:    '/login',
        builder: (_, __) => const PantallaLogin(),
      ),
      ShellRoute(
        builder: (context, state, child) => ScaffoldConNav(child: child),
        routes: [
          GoRoute(
            path:    '/servidores',
            builder: (_, __) => const PantallaServidores(),
            routes: [
              GoRoute(
                path:    ':id',
                builder: (context, state) => PantallaDetalle(
                  id:       state.pathParameters['id']!,
                  servidor: state.extra as ServidorSSH?,
                ),
              ),
            ],
          ),
          GoRoute(
            path:    '/metricas',
            builder: (_, __) => const PantallaMetricas(),
          ),
          GoRoute(
            path:    '/ajustes',
            builder: (_, __) => const PantallaAjustes(),
          ),
        ],
      ),
    ],
  );
});
```

---

### Pantallas principales (proyecto)

**`lib/screens/pantalla_servidores.dart`** — lista con Riverpod:

```dart
// lib/screens/pantalla_servidores.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../models/servidor_ssh.dart';
import '../providers/auth_provider.dart';

class PantallaServidores extends ConsumerWidget {
  const PantallaServidores({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final auth     = ref.watch(authProvider);
    final usuario  = auth is Autenticado ? auth.usuario : '';
    final cs       = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
        title: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text('Servidores',
                style: TextStyle(fontWeight: FontWeight.bold)),
            Text('Hola, $usuario',
                style: const TextStyle(fontSize: 12)),
          ],
        ),
      ),
      body: ListView.separated(
        itemCount:        servidoresSimulados.length,
        separatorBuilder: (_, __) => const Divider(height: 1, indent: 72),
        itemBuilder: (context, i) {
          final s = servidoresSimulados[i];
          return ListTile(
            leading: CircleAvatar(
              backgroundColor: s.ssl
                  ? Colors.green.shade50
                  : Colors.grey.shade100,
              child: Icon(Icons.dns,
                  color: s.ssl ? Colors.green.shade700 : Colors.grey),
            ),
            title:    Text(s.nombre,
                style: const TextStyle(fontWeight: FontWeight.w600)),
            subtitle: Text('${s.ip}:${s.puerto}'),
            trailing: const Icon(Icons.chevron_right),
            onTap: () => context.push('/servidores/${s.id}', extra: s),
          );
        },
      ),
    );
  }
}
```

**`lib/screens/pantalla_ajustes.dart`** — con botón de logout:

```dart
// lib/screens/pantalla_ajustes.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/auth_provider.dart';

class PantallaAjustes extends ConsumerWidget {
  const PantallaAjustes({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final auth    = ref.watch(authProvider);
    final usuario = auth is Autenticado ? auth.usuario : '—';

    return Scaffold(
      appBar: AppBar(title: const Text('Ajustes')),
      body: ListView(
        children: [
          ListTile(
            leading:  const CircleAvatar(child: Icon(Icons.person)),
            title:    Text(usuario),
            subtitle: const Text('Administrador'),
          ),
          const Divider(),
          ListTile(
            leading: const Icon(Icons.logout, color: Colors.red),
            title:   const Text('Cerrar sesion',
                style: TextStyle(color: Colors.red)),
            // logout() cambia el estado → AppMonitoreo se reconstruye
            // → routerProvider crea nuevo router → redirect manda a /login
            onTap: () => ref.read(authProvider.notifier).logout(),
          ),
        ],
      ),
    );
  }
}
```

---

## Guía rápida de imports

```dart
import 'package:go_router/go_router.dart';               // GoRouter, GoRoute, ShellRoute
import 'package:flutter_riverpod/flutter_riverpod.dart'; // para el guard

// Navegar:
context.go('/ruta');                      // reemplaza el stack
context.push('/ruta');                    // apila
context.pop();                            // retrocede
context.push('/ruta', extra: objeto);     // con datos

// Leer parámetros:
state.pathParameters['id']                // path param
state.uri.queryParameters['filtro']       // query param
state.extra as MiObjeto?                  // extra
```

---

## Cuándo usar `go` vs `push`

```
Accion                            Metodo
────────────────────────────────  ───────────────────────────────────
Navegar a una tab principal        context.go()
Abrir una pantalla de detalle      context.push()
Volver atras                       context.pop()
Navegar por nombre de ruta         context.goNamed()
Abrir un modal / bottom sheet      context.push() o showModalBottomSheet
```

---

## Ejercicios propuestos

1. **Deep link con destino** — Al redirigir al login, guarda la ruta
   original como `?destino=`. Después del login exitoso, lee ese parámetro
   con `state.uri.queryParameters['destino']` y navega al destino original
   en lugar de ir siempre a `/servidores`.

2. **Transiciones personalizadas** — Reemplaza el `builder` de las rutas
   de detalle con `pageBuilder` usando `CustomTransitionPage` para añadir
   una transición de deslizamiento horizontal. Usa `SlideTransition` con
   `Tween<Offset>(begin: const Offset(1, 0), end: Offset.zero)`.

3. **`Provider.family` con selección** — Añade una pantalla de métricas
   que muestre los datos de un servidor específico. Crea un
   `FutureProvider.family<MetricaServidor, String>` que reciba el ID
   y simule una llamada a API. Accede desde el detalle con
   `ref.watch(metricaProvider(servidor.id))`.

4. **Guard de roles** — Añade un campo `rol` al `Autenticado` state
   (`admin` o `viewer`). En el `redirect`, redirige a `/sin-permiso`
   si un `viewer` intenta acceder a `/ajustes`. Crea la pantalla
   `PantallaSinPermiso` con un botón que vuelve a `/servidores`.

---

## Resumen de la página 11

- `GoRouter` define las rutas de forma declarativa. `MaterialApp.router` con `routerConfig:` lo activa.
- `context.go()` navega reemplazando el stack. `context.push()` apila la nueva pantalla. `context.pop()` retrocede.
- Los parámetros de ruta se definen con `:nombre` en el `path` y se leen con `state.pathParameters['nombre']`. Los query params van en `state.uri.queryParameters`.
- `extra` permite pasar objetos Dart completos entre rutas sin serialización — útil para evitar buscar el objeto de nuevo.
- `ShellRoute` mantiene un widget padre (el `Scaffold` con `NavigationBar`) mientras navegan las rutas hijas — el tab bar no se reconstruye al cambiar de pantalla.
- `redirect` se ejecuta en cada navegación — es el guard centralizado. Devuelve la ruta de destino o `null` para no redirigir.
- Con Riverpod, `ref.watch(authProvider)` en el widget raíz hace que el router se regenere al cambiar el estado de autenticación, activando el guard reactivamente.
- `debugLogDiagnostics: true` imprime en la consola cada navegación — esencial para depurar rutas y guards.

---

> **Siguiente página →** Página 12: Consumo de API REST —
> `http`, `dio`, serialización JSON y manejo de errores.
