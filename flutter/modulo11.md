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

### Dependencia

```yaml
# pubspec.yaml
dependencies:
  go_router: ^14.6.2
```

---

## Configuración básica

```dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

// El router se define FUERA del árbol de widgets
// Ideal: en un archivo separado lib/router/app_router.dart
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
    GoRoute(
      path:    '/servidores/:id',   // :id es un parámetro de ruta
      builder: (context, state) {
        final id = state.pathParameters['id']!;
        return PantallaDetalleServidor(id: id);
      },
    ),
  ],
);

void main() {
  runApp(const ProviderScope(child: AppMonitoreo()));
}

class AppMonitoreo extends StatelessWidget {
  const AppMonitoreo({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(           // .router en lugar de home:
      title:        'Monitoreo SSH',
      routerConfig: appRouter,           // pasar el router aquí
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
    );
  }
}
```

---

## Navegar entre pantallas

```dart
// En cualquier widget con acceso al BuildContext:

// go() — navega reemplazando la pantalla actual (sin stack)
// Equivale a Navigator.pushReplacement
context.go('/servidores');

// push() — apila la nueva pantalla (con botón de retroceso)
// Equivale a Navigator.push
context.push('/servidores/42');

// pop() — retrocede a la pantalla anterior
context.pop();

// goNamed() — navegar por nombre de ruta (más seguro que strings)
context.goNamed('detalle-servidor', pathParameters: {'id': '42'});

// canPop() — verificar si hay algo en el stack
if (context.canPop()) context.pop();

// Pasar datos de retorno desde pop
context.pop({'resultado': 'guardado'});
// Recibir en push:
final resultado = await context.push<Map>('/editar/42');
```

---

## Parámetros: ruta, query y extras

```dart
final appRouter = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(
      path: '/servidores',
      builder: (context, state) {
        // Query parameters: /servidores?filtro=ssl&pagina=2
        final filtro = state.uri.queryParameters['filtro'];
        final pagina = int.tryParse(
            state.uri.queryParameters['pagina'] ?? '1') ?? 1;
        return PantallaServidores(filtro: filtro, pagina: pagina);
      },
    ),
    GoRoute(
      path: '/servidores/:id',
      builder: (context, state) {
        // Path parameters: /servidores/42
        final id = state.pathParameters['id']!;
        // Extras — objetos Dart completos (no se serializan en URL)
        final servidor = state.extra as ServidorSSH?;
        return PantallaDetalleServidor(id: id, servidor: servidor);
      },
    ),
  ],
);

// Navegar con query params
context.go('/servidores?filtro=ssl&pagina=2');

// Navegar con extras
context.push(
  '/servidores/42',
  extra: servidor,   // pasa el objeto completo
);
```

---

## Rutas anidadas — `ShellRoute`

`ShellRoute` mantiene un widget padre persistente mientras navegan
las rutas hijas — ideal para el `Scaffold` con `NavigationBar`:

```dart
final appRouter = GoRouter(
  initialLocation: '/servidores',
  routes: [
    // ShellRoute — wrapper que persiste entre rutas hijas
    ShellRoute(
      builder: (context, state, child) {
        // child es la pantalla activa hija
        return ScaffoldConNavegacion(child: child);
      },
      routes: [
        GoRoute(
          path: '/servidores',
          builder: (_, __) => const PantallaServidores(),
          routes: [
            // Ruta hija de /servidores
            GoRoute(
              path:    ':id',     // ruta completa: /servidores/:id
              builder: (context, state) => PantallaDetalleServidor(
                id: state.pathParameters['id']!,
              ),
            ),
          ],
        ),
        GoRoute(
          path: '/metricas',
          builder: (_, __) => const PantallaMetricas(),
        ),
        GoRoute(
          path: '/alertas',
          builder: (_, __) => const PantallaAlertas(),
        ),
        GoRoute(
          path: '/ajustes',
          builder: (_, __) => const PantallaAjustes(),
        ),
      ],
    ),

    // Ruta fuera del shell — pantalla completa sin NavigationBar
    GoRoute(
      path: '/login',
      builder: (_, __) => const PantallaLogin(),
    ),
  ],
);

// Scaffold persistente con NavigationBar
class ScaffoldConNavegacion extends StatelessWidget {
  final Widget child;
  const ScaffoldConNavegacion({super.key, required this.child});

  // Determinar qué tab está activo según la ruta actual
  int _indiceActivo(BuildContext context) {
    final location = GoRouterState.of(context).uri.path;
    if (location.startsWith('/metricas'))  return 1;
    if (location.startsWith('/alertas'))   return 2;
    if (location.startsWith('/ajustes'))   return 3;
    return 0;  // /servidores por defecto
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: child,
      bottomNavigationBar: NavigationBar(
        selectedIndex:         _indiceActivo(context),
        onDestinationSelected: (i) {
          switch (i) {
            case 0: context.go('/servidores');
            case 1: context.go('/metricas');
            case 2: context.go('/alertas');
            case 3: context.go('/ajustes');
          }
        },
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
          NavigationDestination(
            icon:         Icon(Icons.notifications_outlined),
            selectedIcon: Icon(Icons.notifications),
            label:        'Alertas',
          ),
          NavigationDestination(
            icon:         Icon(Icons.settings_outlined),
            selectedIcon: Icon(Icons.settings),
            label:        'Ajustes',
          ),
        ],
      ),
    );
  }
}
```

---

## Guards — `redirect` para autenticación

```dart
// Provider de autenticación (Riverpod)
class AuthNotifier extends Notifier<AuthState> {
  @override
  AuthState build() => const AuthState.sinSesion();

  Future<void> login(String usuario, String password) async {
    state = const AuthState.cargando();
    await Future.delayed(const Duration(seconds: 1));

    if (usuario == 'admin' && password == 'admin123') {
      state = AuthState.autenticado(
        usuario: usuario,
        token:   'jwt-token-abc123',
      );
    } else {
      state = const AuthState.error('Credenciales incorrectas');
    }
  }

  void logout() => state = const AuthState.sinSesion();
}

final authProvider =
    NotifierProvider<AuthNotifier, AuthState>(AuthNotifier.new);

// Estados de autenticación con sealed class
sealed class AuthState { const AuthState(); }
class SinSesion   extends AuthState { const SinSesion(); }
class Cargando    extends AuthState { const Cargando(); }
class Autenticado extends AuthState {
  final String usuario;
  final String token;
  const Autenticado({required this.usuario, required this.token});
}
class ErrorAuth   extends AuthState {
  final String mensaje;
  const ErrorAuth(this.mensaje);
}

// GoRouter con guard de autenticación
// Usar ref de Riverpod requiere pasar el ProviderContainer
GoRouter crearRouter(WidgetRef ref) {
  return GoRouter(
    initialLocation: '/servidores',

    // redirect se llama en CADA navegación
    redirect: (context, state) {
      final auth     = ref.read(authProvider);
      final location = state.uri.path;
      final estaEnLogin = location == '/login';

      // No autenticado intentando acceder a ruta protegida
      if (auth is SinSesion && !estaEnLogin) {
        return '/login?destino=${Uri.encodeComponent(location)}';
      }

      // Ya autenticado intentando ir al login
      if (auth is Autenticado && estaEnLogin) {
        return '/servidores';
      }

      return null;  // null = no redirigir
    },

    // Refrescar el router cuando cambia el estado de auth
    refreshListenable: _AuthListenable(ref),

    routes: [ /* ... */ ],
  );
}

// Adaptador para que GoRouter escuche cambios de Riverpod
class _AuthListenable extends ChangeNotifier {
  _AuthListenable(WidgetRef ref) {
    ref.listenManual(authProvider, (_, __) => notifyListeners());
  }
}
```

---

## Programa completo — app con navegación, auth y Riverpod

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

// ── Modelos ───────────────────────────────────────────────────────

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

// ── Estado de autenticación ───────────────────────────────────────

sealed class AuthState { const AuthState(); }
class SinSesion    extends AuthState { const SinSesion(); }
class Cargando     extends AuthState { const Cargando(); }
class Autenticado  extends AuthState {
  final String usuario;
  const Autenticado(this.usuario);
}
class ErrorAuth    extends AuthState {
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
      state = const ErrorAuth('Usuario o contraseña incorrectos');
    }
  }

  void logout() => state = const SinSesion();
}

final authProvider =
    NotifierProvider<AuthNotifier, AuthState>(AuthNotifier.new);

// ── Providers de datos ────────────────────────────────────────────

final servidoresProvider = Provider<List<ServidorSSH>>((ref) => const [
  ServidorSSH(id: '1', nombre: 'prod-web-01', ip: '10.0.2.10', puerto: 22,   ssl: true),
  ServidorSSH(id: '2', nombre: 'prod-db-01',  ip: '10.0.2.20', puerto: 22,   ssl: true),
  ServidorSSH(id: '3', nombre: 'staging-api', ip: '10.0.3.10', puerto: 2222, ssl: false),
]);

final servidorPorIdProvider = Provider.family<ServidorSSH?, String>((ref, id) {
  return ref.watch(servidoresProvider)
      .where((s) => s.id == id)
      .firstOrNull;
});

// ── Router ────────────────────────────────────────────────────────

class _AuthListenable extends ChangeNotifier {
  _AuthListenable(Ref ref) {
    ref.listen(authProvider, (_, __) => notifyListeners());
  }
}

GoRouter buildRouter(Ref ref) => GoRouter(
  initialLocation: '/servidores',
  refreshListenable: _AuthListenable(ref),

  redirect: (context, state) {
    final auth        = ref.read(authProvider);
    final enLogin     = state.uri.path == '/login';
    final autenticado = auth is Autenticado;

    if (!autenticado && !enLogin) {
      final destino = Uri.encodeComponent(state.uri.toString());
      return '/login?destino=$destino';
    }
    if (autenticado && enLogin) return '/servidores';
    return null;
  },

  routes: [
    // Ruta de login — fuera del shell
    GoRoute(
      path:    '/login',
      name:    'login',
      builder: (_, __) => const PantallaLogin(),
    ),

    // Shell con NavigationBar persistente
    ShellRoute(
      builder: (context, state, child) =>
          _ScaffoldConNav(child: child),
      routes: [
        GoRoute(
          path:    '/servidores',
          name:    'servidores',
          builder: (_, __) => const PantallaServidores(),
          routes: [
            GoRoute(
              path:    ':id',
              name:    'detalle-servidor',
              builder: (context, state) => PantallaDetalleServidor(
                id: state.pathParameters['id']!,
              ),
            ),
          ],
        ),
        GoRoute(
          path:    '/ajustes',
          name:    'ajustes',
          builder: (_, __) => const PantallaAjustes(),
        ),
      ],
    ),
  ],
);

// Provider del router — necesita ref para el guard
final routerProvider = Provider<GoRouter>((ref) => buildRouter(ref));

// ── App ───────────────────────────────────────────────────────────

void main() => runApp(const ProviderScope(child: _App()));

class _App extends ConsumerWidget {
  const _App();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);
    return MaterialApp.router(
      title:        'SSH Manager',
      routerConfig: router,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
    );
  }
}

// ── Scaffold con NavigationBar ────────────────────────────────────

class _ScaffoldConNav extends StatelessWidget {
  final Widget child;
  const _ScaffoldConNav({required this.child});

  int _indiceActivo(String path) {
    if (path.startsWith('/ajustes')) return 1;
    return 0;
  }

  @override
  Widget build(BuildContext context) {
    final path = GoRouterState.of(context).uri.path;
    return Scaffold(
      body: child,
      bottomNavigationBar: NavigationBar(
        selectedIndex: _indiceActivo(path),
        onDestinationSelected: (i) {
          switch (i) {
            case 0: context.go('/servidores');
            case 1: context.go('/ajustes');
          }
        },
        destinations: const [
          NavigationDestination(
            icon:         Icon(Icons.dns_outlined),
            selectedIcon: Icon(Icons.dns),
            label:        'Servidores',
          ),
          NavigationDestination(
            icon:         Icon(Icons.settings_outlined),
            selectedIcon: Icon(Icons.settings),
            label:        'Ajustes',
          ),
        ],
      ),
    );
  }
}

// ── Pantalla Login ────────────────────────────────────────────────

class PantallaLogin extends ConsumerStatefulWidget {
  const PantallaLogin({super.key});

  @override
  ConsumerState<PantallaLogin> createState() => _PantallaLoginState();
}

class _PantallaLoginState extends ConsumerState<PantallaLogin> {
  final _formKey  = GlobalKey<FormState>();
  final _ctrlUser = TextEditingController();
  final _ctrlPass = TextEditingController();
  bool  _verPass  = false;

  @override
  void dispose() {
    _ctrlUser.dispose();
    _ctrlPass.dispose();
    super.dispose();
  }

  Future<void> _login() async {
    if (!_formKey.currentState!.validate()) return;
    await ref.read(authProvider.notifier)
        .login(_ctrlUser.text, _ctrlPass.text);
  }

  @override
  Widget build(BuildContext context) {
    final auth = ref.watch(authProvider);
    final cs   = Theme.of(context).colorScheme;

    return Scaffold(
      body: Center(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(32),
          child: ConstrainedBox(
            constraints: const BoxConstraints(maxWidth: 400),
            child: Form(
              key: _formKey,
              child: Column(
                children: [
                  Icon(Icons.terminal,
                      size: 64, color: cs.primary),
                  const SizedBox(height: 8),
                  Text('SSH Manager',
                      style: Theme.of(context).textTheme.headlineMedium
                          ?.copyWith(fontWeight: FontWeight.bold)),
                  const SizedBox(height: 32),

                  // Error de auth
                  if (auth is ErrorAuth)
                    Container(
                      width:   double.infinity,
                      padding: const EdgeInsets.all(12),
                      margin:  const EdgeInsets.only(bottom: 16),
                      decoration: BoxDecoration(
                        color:        cs.errorContainer,
                        borderRadius: BorderRadius.circular(8),
                      ),
                      child: Text(
                        (auth as ErrorAuth).mensaje,
                        style: TextStyle(color: cs.onErrorContainer),
                      ),
                    ),

                  TextFormField(
                    controller:  _ctrlUser,
                    decoration:  const InputDecoration(
                      labelText:  'Usuario',
                      prefixIcon: Icon(Icons.person_outline),
                      border:     OutlineInputBorder(),
                    ),
                    textInputAction: TextInputAction.next,
                    validator: (v) =>
                        v == null || v.isEmpty ? 'Campo obligatorio' : null,
                  ),
                  const SizedBox(height: 16),

                  TextFormField(
                    controller:  _ctrlPass,
                    obscureText: !_verPass,
                    decoration:  InputDecoration(
                      labelText:  'Contraseña',
                      prefixIcon: const Icon(Icons.lock_outline),
                      border:     const OutlineInputBorder(),
                      suffixIcon: IconButton(
                        icon: Icon(_verPass
                            ? Icons.visibility_off
                            : Icons.visibility),
                        onPressed: () =>
                            setState(() => _verPass = !_verPass),
                      ),
                    ),
                    textInputAction: TextInputAction.done,
                    onFieldSubmitted: (_) => _login(),
                    validator: (v) =>
                        v == null || v.isEmpty ? 'Campo obligatorio' : null,
                  ),
                  const SizedBox(height: 24),

                  SizedBox(
                    width: double.infinity,
                    child: FilledButton.icon(
                      onPressed: auth is Cargando ? null : _login,
                      icon: auth is Cargando
                          ? const SizedBox(
                              width: 16, height: 16,
                              child: CircularProgressIndicator(
                                  strokeWidth: 2, color: Colors.white))
                          : const Icon(Icons.login),
                      label: Text(
                          auth is Cargando ? 'Iniciando...' : 'Iniciar sesión'),
                    ),
                  ),

                  const SizedBox(height: 16),
                  Text(
                    'Credenciales de prueba: admin / admin123',
                    style: TextStyle(
                        fontSize: 12,
                        color: cs.onSurfaceVariant),
                  ),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }
}

// ── Pantalla Servidores ───────────────────────────────────────────

class PantallaServidores extends ConsumerWidget {
  const PantallaServidores({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final servidores = ref.watch(servidoresProvider);
    final auth       = ref.watch(authProvider);
    final usuario    = auth is Autenticado ? auth.usuario : '';

    return Scaffold(
      appBar: AppBar(
        title: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const Text('Servidores',
                style: TextStyle(fontWeight: FontWeight.bold)),
            Text('Hola, $usuario',
                style: const TextStyle(fontSize: 12)),
          ],
        ),
        actions: [
          IconButton(
            icon:      const Icon(Icons.logout),
            onPressed: () {
              ref.read(authProvider.notifier).logout();
              // El guard redirige automáticamente a /login
            },
            tooltip: 'Cerrar sesión',
          ),
        ],
      ),
      body: ListView.separated(
        itemCount:        servidores.length,
        separatorBuilder: (_, __) => const Divider(height: 1, indent: 72),
        itemBuilder: (context, i) {
          final s = servidores[i];
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
            onTap: () => context.push(
              '/servidores/${s.id}',
              extra: s,
            ),
          );
        },
      ),
    );
  }
}

// ── Pantalla Detalle ──────────────────────────────────────────────

class PantallaDetalleServidor extends ConsumerWidget {
  final String id;
  const PantallaDetalleServidor({super.key, required this.id});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final servidor = ref.watch(servidorPorIdProvider(id));

    if (servidor == null) {
      return Scaffold(
        appBar: AppBar(title: const Text('Servidor no encontrado')),
        body: Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.error_outline, size: 56),
              const SizedBox(height: 12),
              Text('Servidor con id "$id" no existe'),
              const SizedBox(height: 12),
              FilledButton(
                onPressed: () => context.go('/servidores'),
                child: const Text('Volver'),
              ),
            ],
          ),
        ),
      );
    }

    return Scaffold(
      appBar: AppBar(title: Text(servidor.nombre)),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            _FilaDetalle('ID',       servidor.id),
            _FilaDetalle('Hostname', servidor.nombre),
            _FilaDetalle('IP',       servidor.ip),
            _FilaDetalle('Puerto',   '${servidor.puerto}'),
            _FilaDetalle('SSL/TLS',  servidor.ssl ? '✅ Activo' : '❌ Inactivo'),
            const SizedBox(height: 24),
            SizedBox(
              width: double.infinity,
              child: FilledButton.icon(
                onPressed: () {
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(
                      content: Text(
                          'Conectando a ${servidor.nombre}...'),
                      behavior: SnackBarBehavior.floating,
                    ),
                  );
                },
                icon:  const Icon(Icons.terminal),
                label: Text('SSH a ${servidor.nombre}'),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _FilaDetalle extends StatelessWidget {
  final String etiqueta;
  final String valor;
  const _FilaDetalle(this.etiqueta, this.valor);

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(vertical: 8),
      child: Row(children: [
        SizedBox(
          width: 80,
          child: Text(etiqueta,
              style: TextStyle(
                  color: Theme.of(context).colorScheme.onSurfaceVariant,
                  fontWeight: FontWeight.w500)),
        ),
        Expanded(child: Text(valor,
            style: const TextStyle(fontWeight: FontWeight.w600))),
      ]),
    );
  }
}

// ── Pantalla Ajustes ──────────────────────────────────────────────

class PantallaAjustes extends ConsumerWidget {
  const PantallaAjustes({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final auth = ref.watch(authProvider);
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
            leading:  const Icon(Icons.logout, color: Colors.red),
            title:    const Text('Cerrar sesión',
                style: TextStyle(color: Colors.red)),
            onTap: () => ref.read(authProvider.notifier).logout(),
          ),
        ],
      ),
    );
  }
}
```

---

## Ejercicios propuestos

1. **Deep link con destino** — Al redirigir al login, se guarda la ruta
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
- `refreshListenable` recibe un `Listenable` que notifica al router cuando debe re-evaluar el `redirect` — permite que Riverpod controle los guards.
- `Provider.family` crea un provider parametrizado — `servidorPorIdProvider(id)` devuelve un provider diferente por cada ID.

---

> **Siguiente página →** Página 12: Consumo de API REST —
> `http`, modelos con `fromJson`/`toJson`, `FutureProvider` y manejo de errores.