# Tutorial Flutter — Página 8
## Módulo 2 · Flutter
### Material 3, tema, `Scaffold`, `AppBar`, navegación y componentes esenciales

---

## Material Design 3 en Flutter

Material 3 (M3) es el sistema de diseño de Google adoptado por Flutter 3.
Introduce un sistema de colores basado en una sola **seed color** que genera
toda la paleta automáticamente, tipografía refinada y componentes rediseñados:

```
Material 2 (antiguo)           Material 3 (actual)
──────────────────────         ──────────────────────────────────
primaryColor manual            ColorScheme.fromSeed(seedColor)
Scaffold backgroundColor       Surface, SurfaceContainer, etc.
FlatButton, RaisedButton        TextButton, ElevatedButton, FilledButton
Drawer lateral                 NavigationDrawer + NavigationRail
BottomNavigationBar             NavigationBar
```

---

## Patrones clave — de un vistazo

Referencia rápida de los patrones del módulo. Léelos ahora y vuelve
a consultarlos conforme avanzas en cada paso.

---

### `ThemeData` y `ColorScheme`

```dart
import 'package:flutter/material.dart';

MaterialApp(
  theme: ThemeData(
    // Una sola línea genera TODA la paleta de colores
    colorScheme: ColorScheme.fromSeed(
      seedColor:   const Color(0xFF1565C0),  // azul profundo
      brightness:  Brightness.light,
    ),
    useMaterial3: true,

    // Tipografía personalizada (opcional)
    textTheme: const TextTheme(
      displayLarge:  TextStyle(fontSize: 57, fontWeight: FontWeight.w400),
      headlineLarge: TextStyle(fontSize: 32, fontWeight: FontWeight.w600),
      titleLarge:    TextStyle(fontSize: 22, fontWeight: FontWeight.w600),
      bodyLarge:     TextStyle(fontSize: 16, fontWeight: FontWeight.w400),
      labelLarge:    TextStyle(fontSize: 14, fontWeight: FontWeight.w500),
    ),
  ),
)
```

---

### Los roles de color de Material 3

```dart
// Acceder a los colores del tema en cualquier widget
Widget build(BuildContext context) {
  final cs = Theme.of(context).colorScheme;

  return Column(children: [
    // Roles principales
    Container(color: cs.primary,          child: Text('primary',          style: TextStyle(color: cs.onPrimary))),
    Container(color: cs.primaryContainer, child: Text('primaryContainer', style: TextStyle(color: cs.onPrimaryContainer))),
    Container(color: cs.secondary,        child: Text('secondary',        style: TextStyle(color: cs.onSecondary))),
    Container(color: cs.tertiary,         child: Text('tertiary',         style: TextStyle(color: cs.onTertiary))),
    Container(color: cs.error,            child: Text('error',            style: TextStyle(color: cs.onError))),

    // Roles de superficie
    Container(color: cs.surface,                 child: Text('surface')),
    Container(color: cs.surfaceContainerHighest, child: Text('surfaceContainerHighest')),

    // Texto sobre superficie
    Text('onSurface',        style: TextStyle(color: cs.onSurface)),
    Text('onSurfaceVariant', style: TextStyle(color: cs.onSurfaceVariant)),
  ]);
}
```

---

### Modo oscuro

```dart
class AppRaiz extends StatefulWidget {
  const AppRaiz({super.key});
  @override
  State<AppRaiz> createState() => _AppRaizState();
}

class _AppRaizState extends State<AppRaiz> {
  ThemeMode _modoTema = ThemeMode.system;

  @override
  Widget build(BuildContext context) {
    const seedColor = Color(0xFF1565C0);

    return MaterialApp(
      themeMode: _modoTema,   // system, light o dark

      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor:  seedColor,
          brightness: Brightness.light,
        ),
        useMaterial3: true,
      ),

      darkTheme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor:  seedColor,
          brightness: Brightness.dark,  // ← diferencia clave
        ),
        useMaterial3: true,
      ),

      home: PantallaInicio(
        onToggleTema: () => setState(() {
          _modoTema = _modoTema == ThemeMode.light
              ? ThemeMode.dark
              : ThemeMode.light;
        }),
      ),
    );
  }
}
```

---

### `Scaffold` — estructura de pantalla completa

`Scaffold` implementa la estructura visual básica de Material Design:

```dart
Scaffold(
  // Barra superior
  appBar: AppBar(
    title:   const Text('Título'),
    actions: [
      IconButton(icon: const Icon(Icons.search),    onPressed: () {}),
      IconButton(icon: const Icon(Icons.more_vert), onPressed: () {}),
    ],
    leading: IconButton(icon: const Icon(Icons.menu), onPressed: () {}),
  ),

  // Contenido principal
  body: const Center(child: Text('Contenido')),

  // Botón de acción flotante
  floatingActionButton: FloatingActionButton(
    onPressed: () {},
    child:     const Icon(Icons.add),
  ),
  floatingActionButtonLocation: FloatingActionButtonLocation.endFloat,

  // Barra inferior de navegación
  bottomNavigationBar: NavigationBar(
    destinations: const [
      NavigationDestination(icon: Icon(Icons.home),     label: 'Inicio'),
      NavigationDestination(icon: Icon(Icons.settings), label: 'Ajustes'),
    ],
    selectedIndex:         0,
    onDestinationSelected: (index) {},
  ),

  // Drawer lateral
  drawer: const Drawer(child: Text('Menú lateral')),
)
```

---

### `AppBar` con variantes M3

```dart
// AppBar estándar
AppBar(
  title:           const Text('Servidores'),
  backgroundColor: Theme.of(context).colorScheme.primaryContainer,
  foregroundColor: Theme.of(context).colorScheme.onPrimaryContainer,
  elevation:       0,
  centerTitle:     false,
  actions: [
    IconButton(
      icon:      const Icon(Icons.filter_list),
      onPressed: () {},
      tooltip:   'Filtrar',
    ),
  ],
)

// SliverAppBar — colapsa al hacer scroll
CustomScrollView(
  slivers: [
    SliverAppBar.large(
      title:          const Text('Dashboard'),
      pinned:         true,    // siempre visible al menos el título
      floating:       false,
      expandedHeight: 200,
      flexibleSpace:  FlexibleSpaceBar(
        background: Container(color: Colors.indigo.shade900),
      ),
    ),
    SliverList(
      delegate: SliverChildBuilderDelegate(
        (context, i) => ListTile(title: Text('Item $i')),
        childCount: 50,
      ),
    ),
  ],
)
```

---

### Componentes de texto y botones

#### `Text` con estilo

```dart
// Estilos del tema
Text('Título grande',     style: Theme.of(context).textTheme.headlineLarge)
Text('Cuerpo normal',    style: Theme.of(context).textTheme.bodyMedium)
Text('Etiqueta pequeña', style: Theme.of(context).textTheme.labelSmall)

// Personalización sobre el estilo del tema
Text(
  'Texto personalizado',
  style: Theme.of(context).textTheme.bodyLarge?.copyWith(
    color:      Theme.of(context).colorScheme.primary,
    fontWeight: FontWeight.bold,
  ),
  maxLines:  2,
  overflow:  TextOverflow.ellipsis,
  textAlign: TextAlign.center,
)

// RichText — texto con múltiples estilos en la misma línea
RichText(
  text: TextSpan(
    style: DefaultTextStyle.of(context).style,
    children: [
      const TextSpan(text: 'Estado: '),
      TextSpan(
        text:  'ACTIVO',
        style: const TextStyle(color: Colors.green, fontWeight: FontWeight.bold),
      ),
      const TextSpan(text: ' · Latencia: '),
      TextSpan(
        text:  '42ms',
        style: TextStyle(color: Colors.blue.shade700),
      ),
    ],
  ),
)
```

#### Botones en Material 3

```dart
// Los 5 variantes de botón M3 — de mayor a menor énfasis
FilledButton(onPressed: () {}, child: const Text('Confirmar'))       // mayor énfasis
FilledButton.tonal(onPressed: () {}, child: const Text('Guardar'))   // énfasis medio con color secundario
ElevatedButton(onPressed: () {}, child: const Text('Continuar'))     // énfasis medio con sombra
OutlinedButton(onPressed: () {}, child: const Text('Cancelar'))      // énfasis bajo con borde
TextButton(onPressed: () {}, child: const Text('Ver más'))           // énfasis mínimo

// Con ícono
FilledButton.icon(
  onPressed: () {},
  icon:  const Icon(Icons.send),
  label: const Text('Enviar'),
)

// Deshabilitado — onPressed: null
FilledButton(onPressed: null, child: const Text('No disponible'))

// Estilo personalizado
FilledButton(
  style: FilledButton.styleFrom(
    backgroundColor: Colors.red.shade700,
    foregroundColor: Colors.white,
    minimumSize:     const Size(double.infinity, 50),
  ),
  onPressed: () {},
  child: const Text('Eliminar servidor'),
)
```

---

### `NavigationBar` — navegación inferior M3

```dart
class AppConNavegacion extends StatefulWidget {
  const AppConNavegacion({super.key});
  @override
  State<AppConNavegacion> createState() => _AppConNavegacionState();
}

class _AppConNavegacionState extends State<AppConNavegacion> {
  int _indice = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: IndexedStack(               // mantiene el estado de cada pestaña
        index: _indice,
        children: const [
          Center(child: Text('Dashboard')),
          Center(child: Text('Servidores')),
          Center(child: Text('Alertas')),
          Center(child: Text('Ajustes')),
        ],
      ),
      bottomNavigationBar: NavigationBar(
        selectedIndex:         _indice,
        onDestinationSelected: (i) => setState(() => _indice = i),
        indicatorColor: Theme.of(context).colorScheme.primaryContainer,
        destinations: const [
          NavigationDestination(
            icon:         Icon(Icons.dashboard_outlined),
            selectedIcon: Icon(Icons.dashboard),
            label:        'Dashboard',
          ),
          NavigationDestination(
            icon:         Icon(Icons.dns_outlined),
            selectedIcon: Icon(Icons.dns),
            label:        'Servidores',
          ),
          NavigationDestination(
            icon:         Badge(label: Text('3'), child: Icon(Icons.notifications_outlined)),
            selectedIcon: Badge(label: Text('3'), child: Icon(Icons.notifications)),
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

### `SnackBar` y `Dialog`

```dart
// SnackBar — mensaje temporal
void mostrarSnackBar(BuildContext context, String mensaje, {bool esError = false}) {
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(
      content: Text(mensaje),
      backgroundColor: esError
          ? Theme.of(context).colorScheme.error
          : null,
      action: SnackBarAction(
        label:    'Deshacer',
        onPressed: () {},
      ),
      behavior:  SnackBarBehavior.floating,
      shape:     RoundedRectangleBorder(borderRadius: BorderRadius.circular(8)),
      duration:  const Duration(seconds: 3),
    ),
  );
}

// AlertDialog — confirmación de acción crítica
Future<bool?> mostrarConfirmacion(BuildContext context) {
  return showDialog<bool>(
    context: context,
    builder: (ctx) => AlertDialog(
      icon:    const Icon(Icons.warning_amber, color: Colors.orange),
      title:   const Text('Eliminar servidor'),
      content: const Text('Esta acción no se puede deshacer.'),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(ctx, false),
          child:     const Text('Cancelar'),
        ),
        FilledButton(
          style: FilledButton.styleFrom(
            backgroundColor: Theme.of(ctx).colorScheme.error,
          ),
          onPressed: () => Navigator.pop(ctx, true),
          child: const Text('Eliminar'),
        ),
      ],
    ),
  );
}
```

---

## Crea el proyecto

```bash
flutter create modulo08_material3
cd modulo08_material3
```

Borra todo el contenido de `lib/main.dart` y déjalo vacío.
El Paso 1 parte desde cero en ese archivo.

```
modulo08_material3/
├── lib/
│   ├── main.dart              ← aquí trabajamos
│   ├── screens/               ← se crea al llegar al Paso 2
│   └── widgets/               ← se crea al llegar al Paso 4
├── pubspec.yaml
└── ...
```

Para correr la app:
```bash
flutter run
```

---

## Paso 1 — ThemeData básico y `Scaffold`

Escribe esto en `lib/main.dart`:

```dart
// lib/main.dart
import 'package:flutter/material.dart';

void main() => runApp(const AppMonitoreo());

class AppMonitoreo extends StatelessWidget {
  const AppMonitoreo({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
          seedColor: const Color(0xFF1565C0), // azul profundo
        ),
        useMaterial3: true,
      ),
      home: const _PantallaPrincipal(),
    );
  }
}

class _PantallaPrincipal extends StatelessWidget {
  const _PantallaPrincipal();

  @override
  Widget build(BuildContext context) {
    final cs   = Theme.of(context).colorScheme;
    final text = Theme.of(context).textTheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Sistema de Monitoreo'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
        actions: [
          IconButton(icon: const Icon(Icons.refresh), onPressed: () {}),
        ],
      ),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(Icons.dns, size: 64, color: cs.primary),
            const SizedBox(height: 16),
            Text(
              'Servidor web-01',
              style: text.headlineMedium?.copyWith(fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            Text(
              '10.0.2.10 · Ubuntu 24.04',
              style: text.bodyMedium?.copyWith(color: cs.onSurfaceVariant),
            ),
            const SizedBox(height: 24),
            FilledButton.icon(
              onPressed: () {},
              icon:  const Icon(Icons.terminal),
              label: const Text('Conectar SSH'),
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {},
        tooltip: 'Agregar servidor',
        child:   const Icon(Icons.add),
      ),
    );
  }
}
```

`ColorScheme.fromSeed` genera automáticamente toda la paleta — primario,
secundario, terciario, error y superficies. Accede a los colores siempre
con `Theme.of(context).colorScheme`.

> **Nunca** uses `Colors.blue` directamente en producción: rompe el tema
> y el modo oscuro. Siempre usa `cs.primary`, `cs.error`, etc.

### Prueba esto

- Cambia `seedColor: const Color(0xFF1565C0)` a `Colors.green` — toda la paleta
  cambia automáticamente sin tocar ningún widget
- Cambia `cs.primaryContainer` a `cs.tertiaryContainer` en el AppBar —
  observa el contraste automático en `foregroundColor`
- Agrega `centerTitle: true` en el AppBar — el título se centra
- Cambia `size: 64` del Icon a `size: 96` y `Icons.dns` a `Icons.cloud`

---

## Convierte `main.dart` en selector de pasos

Antes de avanzar al Paso 2, actualiza `main.dart` **una sola vez**.
A partir de aquí solo cambias el número de `paso` para navegar entre ejemplos.

La raíz es un `StatefulWidget` porque en Paso 2 necesitamos cambiar el `ThemeMode`
desde dentro de la app. El Paso 1 sigue mostrando la misma pantalla con `paso = 1`:

```dart
// lib/main.dart
import 'package:flutter/material.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1  ThemeData + Scaffold básico                         │
// │  2  Paso 2  Modo oscuro — ThemeMode dinámico                    │
// │  3  Paso 3  AppBar variantes y SliverAppBar                     │
// │  4  Paso 4  Botones Material 3                                  │
// │  5  Paso 5  NavigationBar con 4 pestañas                        │
// │  6  Paso 6  SnackBar y AlertDialog                              │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 1;

void main() => runApp(const AppMonitoreo());

class AppMonitoreo extends StatefulWidget {
  const AppMonitoreo({super.key});
  @override
  State<AppMonitoreo> createState() => _AppMonitoreoState();
}

class _AppMonitoreoState extends State<AppMonitoreo> {
  ThemeMode _themeMode = ThemeMode.system;

  @override
  Widget build(BuildContext context) {
    const seedColor = Color(0xFF1565C0);

    return MaterialApp(
      debugShowCheckedModeBanner: false,
      themeMode: _themeMode,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
            seedColor: seedColor, brightness: Brightness.light),
        useMaterial3: true,
      ),
      darkTheme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
            seedColor: seedColor, brightness: Brightness.dark),
        useMaterial3: true,
      ),
      home: switch (paso) {
        1 => const _Paso1(),
        _ => Scaffold(
            body: Center(child: Text('Paso $paso: crea el widget primero'))),
      },
    );
  }
}

// ─── Paso 1 — vive en main.dart ────────────────────────────────────────
class _Paso1 extends StatelessWidget {
  const _Paso1();

  @override
  Widget build(BuildContext context) {
    final cs   = Theme.of(context).colorScheme;
    final text = Theme.of(context).textTheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Sistema de Monitoreo'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
        actions: [
          IconButton(icon: const Icon(Icons.refresh), onPressed: () {}),
        ],
      ),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(Icons.dns, size: 64, color: cs.primary),
            const SizedBox(height: 16),
            Text(
              'Servidor web-01',
              style: text.headlineMedium?.copyWith(fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            Text(
              '10.0.2.10 · Ubuntu 24.04',
              style: text.bodyMedium?.copyWith(color: cs.onSurfaceVariant),
            ),
            const SizedBox(height: 24),
            FilledButton.icon(
              onPressed: () {},
              icon:  const Icon(Icons.terminal),
              label: const Text('Conectar SSH'),
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {},
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

Guarda y verifica que la app sigue mostrando el servidor. Listo — de ahora
en adelante cada paso nuevo solo requiere agregar un `import` y un `case` al switch.

---

## Paso 2 — Modo oscuro: `ThemeMode` dinámico

`ThemeMode` controla qué tema aplica la app: `system` sigue la preferencia
del dispositivo, `light` fuerza claro, `dark` fuerza oscuro.

### Archivo: `lib/screens/pantalla_tema.dart`

Crea la carpeta `lib/screens/` y dentro el archivo:

```dart
// lib/screens/pantalla_tema.dart
import 'package:flutter/material.dart';

class PantallaTema extends StatelessWidget {
  final ThemeMode themeMode;
  final void Function(ThemeMode) onToggle;

  const PantallaTema({
    super.key,
    required this.themeMode,
    required this.onToggle,
  });

  @override
  Widget build(BuildContext context) {
    final cs   = Theme.of(context).colorScheme;
    final text = Theme.of(context).textTheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Modo de tema'),
        backgroundColor: cs.surfaceContainerHighest,
      ),
      body: ListView(
        children: [

          // ── Sección: Selección de tema ────────────────────────────
          Padding(
            padding: const EdgeInsets.fromLTRB(16, 16, 16, 4),
            child: Text(
              'Apariencia',
              style: text.labelLarge?.copyWith(color: cs.primary),
            ),
          ),
          ...[
            (label: 'Sistema', mode: ThemeMode.system, icon: Icons.brightness_auto),
            (label: 'Claro',   mode: ThemeMode.light,  icon: Icons.light_mode),
            (label: 'Oscuro',  mode: ThemeMode.dark,   icon: Icons.dark_mode),
          ].map((opcion) => RadioListTile<ThemeMode>(
            title:     Text(opcion.label),
            secondary: Icon(opcion.icon),
            value:     opcion.mode,
            groupValue: themeMode,
            onChanged:  (v) => onToggle(v!),
          )),

          const Divider(),

          // ── Sección: Vista previa de la paleta activa ─────────────
          Padding(
            padding: const EdgeInsets.all(16),
            child: Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text('Vista previa',
                        style: text.labelLarge?.copyWith(color: cs.primary)),
                    const SizedBox(height: 12),
                    // Paleta activa
                    Row(children: [
                      _CirculoPreview(color: cs.primary,                label: 'P'),
                      _CirculoPreview(color: cs.secondary,              label: 'S'),
                      _CirculoPreview(color: cs.tertiary,               label: 'T'),
                      _CirculoPreview(color: cs.error,                  label: 'E'),
                      _CirculoPreview(color: cs.surfaceContainerHighest, label: 'Sf'),
                    ]),
                    const SizedBox(height: 12),
                    // Ejemplo de componentes con el tema actual
                    Row(children: [
                      FilledButton.icon(
                        onPressed: () {},
                        icon:  const Icon(Icons.terminal, size: 16),
                        label: const Text('SSH'),
                      ),
                      const SizedBox(width: 8),
                      OutlinedButton(onPressed: () {}, child: const Text('Logs')),
                    ]),
                  ],
                ),
              ),
            ),
          ),
        ],
      ),
    );
  }
}

class _CirculoPreview extends StatelessWidget {
  final Color  color;
  final String label;
  const _CirculoPreview({required this.color, required this.label});

  @override
  Widget build(BuildContext context) => Padding(
    padding: const EdgeInsets.only(right: 8),
    child: CircleAvatar(
      radius:          16,
      backgroundColor: color,
      child: Text(
        label,
        style: TextStyle(
          color: color.computeLuminance() > 0.5 ? Colors.black : Colors.white,
          fontSize:   10,
          fontWeight: FontWeight.bold,
        ),
      ),
    ),
  );
}
```

### Agrega al `main.dart`

```dart
import 'screens/pantalla_tema.dart';
```

```dart
2 => PantallaTema(
       themeMode: _themeMode,
       onToggle:  (mode) => setState(() => _themeMode = mode),
     ),
```

Cambia `paso = 2` y guarda.

### Prueba esto

- Selecciona **Oscuro** — toda la app cambia de tema al instante sin recargar
- Selecciona **Sistema** y cambia el modo del dispositivo/emulador — la app sigue
- Observa cómo los círculos de vista previa cambian de color al cambiar el tema:
  el azul claro del tema claro se convierte en azul oscuro en modo oscuro
- Vuelve a `paso = 1` — la pantalla del servidor también respeta el modo oscuro

---

## Paso 3 — `AppBar` variantes y `SliverAppBar`

`SliverAppBar.large` colapsa al hacer scroll. `pinned: true` mantiene el
título visible. `flexibleSpace` es el área que aparece cuando el AppBar está expandido.

### Archivo: `lib/screens/pantalla_appbar.dart`

```dart
// lib/screens/pantalla_appbar.dart
import 'package:flutter/material.dart';

class PantallaAppBar extends StatelessWidget {
  const PantallaAppBar({super.key});

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      body: CustomScrollView(
        slivers: [
          // SliverAppBar — colapsa al hacer scroll
          SliverAppBar.large(
            title:           const Text('Servidores'),
            pinned:          true,
            backgroundColor: cs.primaryContainer,
            foregroundColor: cs.onPrimaryContainer,
            actions: [
              IconButton(
                icon:      const Icon(Icons.filter_list),
                onPressed: () {},
                tooltip:   'Filtrar',
              ),
              IconButton(
                icon:      const Icon(Icons.search),
                onPressed: () {},
                tooltip:   'Buscar',
              ),
            ],
            flexibleSpace: FlexibleSpaceBar(
              background: Container(
                color: cs.primaryContainer,
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    const SizedBox(height: 56),
                    Icon(Icons.dns, size: 48, color: cs.onPrimaryContainer),
                    const SizedBox(height: 8),
                    Text(
                      '8 servidores activos',
                      style: TextStyle(color: cs.onPrimaryContainer),
                    ),
                  ],
                ),
              ),
            ),
          ),

          // Lista de servidores
          SliverPadding(
            padding: const EdgeInsets.all(8),
            sliver: SliverList(
              delegate: SliverChildBuilderDelegate(
                (context, i) => Card(
                  child: ListTile(
                    leading:  Icon(Icons.dns, color: cs.primary),
                    title:    Text('prod-web-0${i + 1}'),
                    subtitle: Text('10.0.2.${i + 10} · Activo'),
                    trailing: Chip(
                      label:           const Text('OK'),
                      backgroundColor: cs.primaryContainer,
                      labelStyle:      TextStyle(color: cs.onPrimaryContainer),
                    ),
                    onTap: () {},
                  ),
                ),
                childCount: 10,
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'screens/pantalla_appbar.dart';
```

```dart
3 => const PantallaAppBar(),
```

Cambia `paso = 3` y guarda.

### Prueba esto

- Haz scroll hacia abajo — el AppBar colapsa y solo queda el título visible
- Haz scroll hacia arriba — el AppBar se expande mostrando el icono y el contador
- Cambia `SliverAppBar.large` a `SliverAppBar.medium` — la altura colapsada es menor
- Cambia `pinned: true` a `pinned: false` — el AppBar desaparece completamente al hacer scroll
- Cambia `backgroundColor: cs.primaryContainer` a `cs.tertiaryContainer` en el SliverAppBar

---

## Paso 4 — Botones Material 3

M3 tiene 5 variantes de botón ordenadas de mayor a menor énfasis visual.
La regla de oro: solo un `FilledButton` por pantalla como acción principal.

### Archivo: `lib/widgets/catalogo_botones.dart`

```dart
// lib/widgets/catalogo_botones.dart
import 'package:flutter/material.dart';

class CatalogoBotones extends StatelessWidget {
  const CatalogoBotones({super.key});

  @override
  Widget build(BuildContext context) {
    final cs   = Theme.of(context).colorScheme;
    final text = Theme.of(context).textTheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Botones Material 3'),
        backgroundColor: cs.surfaceContainerHighest,
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [

          // ── Los 5 variantes ──────────────────────────────────────
          Text('Variantes — de mayor a menor énfasis',
              style: text.labelLarge?.copyWith(color: cs.primary)),
          const SizedBox(height: 12),
          FilledButton(
            onPressed: () {},
            child: const Text('FilledButton — acción principal'),
          ),
          const SizedBox(height: 8),
          FilledButton.tonal(
            onPressed: () {},
            child: const Text('FilledButton.tonal — acción secundaria'),
          ),
          const SizedBox(height: 8),
          ElevatedButton(
            onPressed: () {},
            child: const Text('ElevatedButton — acción con sombra'),
          ),
          const SizedBox(height: 8),
          OutlinedButton(
            onPressed: () {},
            child: const Text('OutlinedButton — acción con borde'),
          ),
          const SizedBox(height: 8),
          TextButton(
            onPressed: () {},
            child: const Text('TextButton — acción mínima'),
          ),

          const Divider(height: 32),

          // ── Con ícono ────────────────────────────────────────────
          Text('Con ícono',
              style: text.labelLarge?.copyWith(color: cs.primary)),
          const SizedBox(height: 12),
          FilledButton.icon(
            onPressed: () {},
            icon:  const Icon(Icons.send),
            label: const Text('Enviar reporte'),
          ),
          const SizedBox(height: 8),
          OutlinedButton.icon(
            onPressed: () {},
            icon:  const Icon(Icons.download),
            label: const Text('Exportar logs'),
          ),
          const SizedBox(height: 8),
          TextButton.icon(
            onPressed: () {},
            icon:  const Icon(Icons.open_in_new),
            label: const Text('Ver documentación'),
          ),

          const Divider(height: 32),

          // ── Estados y personalización ────────────────────────────
          Text('Estados y personalización',
              style: text.labelLarge?.copyWith(color: cs.primary)),
          const SizedBox(height: 12),
          FilledButton(
            onPressed: null,               // null = deshabilitado
            child: const Text('No disponible'),
          ),
          const SizedBox(height: 8),
          FilledButton(
            style: FilledButton.styleFrom(
              backgroundColor: cs.error,
              foregroundColor: cs.onError,
              minimumSize:     const Size(double.infinity, 48),
            ),
            onPressed: () {},
            child: const Text('Eliminar servidor'),
          ),
          const SizedBox(height: 8),
          // Fila de botones compactos
          Row(children: [
            Expanded(
              child: OutlinedButton(onPressed: () {}, child: const Text('Reiniciar')),
            ),
            const SizedBox(width: 8),
            Expanded(
              child: FilledButton(onPressed: () {}, child: const Text('Apagar')),
            ),
          ]),
        ],
      ),
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'widgets/catalogo_botones.dart';
```

```dart
4 => const CatalogoBotones(),
```

Cambia `paso = 4` y guarda.

### Prueba esto

- Desactiva un `FilledButton` pasando `onPressed: null` — se vuelve gris automáticamente
- Cambia el `FilledButton` destructivo para usar `cs.errorContainer` / `cs.onErrorContainer` —
  variante más suave del rojo
- Convierte un `OutlinedButton` en `FilledButton.tonal` — observa el cambio de énfasis
- Prueba `IconButton(icon: Icon(Icons.delete), onPressed: () {})` — botón solo con ícono

---

## Paso 5 — `NavigationBar` con 4 pestañas

`NavigationBar` es el componente M3 para navegación inferior.
`IndexedStack` mantiene el estado de cada pantalla al cambiar de pestaña.

### Archivo: `lib/screens/pantalla_navegacion.dart`

```dart
// lib/screens/pantalla_navegacion.dart
import 'package:flutter/material.dart';

class PantallaNavegacion extends StatefulWidget {
  const PantallaNavegacion({super.key});

  @override
  State<PantallaNavegacion> createState() => _PantallaNavegacionState();
}

class _PantallaNavegacionState extends State<PantallaNavegacion> {
  int _indice = 0;

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Sistema de Monitoreo'),
        backgroundColor: cs.surfaceContainerHighest,
      ),
      body: IndexedStack(
        index: _indice,
        children: const [
          _PantallaDashboard(),
          _PantallaServidores(),
          _PantallaAlertas(),
          _PantallaAjustes(),
        ],
      ),
      bottomNavigationBar: NavigationBar(
        selectedIndex:         _indice,
        onDestinationSelected: (i) => setState(() => _indice = i),
        indicatorColor: cs.primaryContainer,
        destinations: const [
          NavigationDestination(
            icon:         Icon(Icons.dashboard_outlined),
            selectedIcon: Icon(Icons.dashboard),
            label:        'Dashboard',
          ),
          NavigationDestination(
            icon:         Icon(Icons.dns_outlined),
            selectedIcon: Icon(Icons.dns),
            label:        'Servidores',
          ),
          NavigationDestination(
            icon:         Badge(label: Text('3'), child: Icon(Icons.notifications_outlined)),
            selectedIcon: Badge(label: Text('3'), child: Icon(Icons.notifications)),
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

// ─── Pantallas de cada pestaña ────────────────────────────────────────────

class _PantallaDashboard extends StatelessWidget {
  const _PantallaDashboard();

  @override
  Widget build(BuildContext context) {
    final cs   = Theme.of(context).colorScheme;
    final text = Theme.of(context).textTheme;

    return ListView(
      padding: const EdgeInsets.all(16),
      children: [
        Text('Resumen', style: text.headlineSmall),
        const SizedBox(height: 16),
        // Tarjetas de métricas
        Row(children: [
          Expanded(child: _TarjetaMetrica(titulo: 'Servidores', valor: '8',  icono: Icons.dns,          color: cs.primaryContainer)),
          const SizedBox(width: 8),
          Expanded(child: _TarjetaMetrica(titulo: 'Alertas',    valor: '3',  icono: Icons.notifications, color: cs.errorContainer)),
        ]),
        const SizedBox(height: 8),
        Row(children: [
          Expanded(child: _TarjetaMetrica(titulo: 'Uptime',   valor: '99.8%', icono: Icons.trending_up, color: cs.tertiaryContainer)),
          const SizedBox(width: 8),
          Expanded(child: _TarjetaMetrica(titulo: 'Tráfico',  valor: '4.2 GB', icono: Icons.wifi,       color: cs.secondaryContainer)),
        ]),
      ],
    );
  }
}

class _TarjetaMetrica extends StatelessWidget {
  final String titulo;
  final String valor;
  final IconData icono;
  final Color    color;

  const _TarjetaMetrica({
    required this.titulo,
    required this.valor,
    required this.icono,
    required this.color,
  });

  @override
  Widget build(BuildContext context) {
    final text = Theme.of(context).textTheme;

    return Card(
      color: color,
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Icon(icono, size: 28),
            const SizedBox(height: 8),
            Text(valor,  style: text.headlineMedium?.copyWith(fontWeight: FontWeight.bold)),
            Text(titulo, style: text.bodySmall),
          ],
        ),
      ),
    );
  }
}

class _PantallaServidores extends StatelessWidget {
  const _PantallaServidores();

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return ListView.builder(
      padding: const EdgeInsets.all(8),
      itemCount: 6,
      itemBuilder: (ctx, i) => Card(
        child: ListTile(
          leading:  Icon(Icons.dns, color: cs.primary),
          title:    Text('prod-web-0${i + 1}'),
          subtitle: Text('10.0.2.${i + 10} · Activo'),
          trailing: Icon(Icons.chevron_right, color: cs.onSurfaceVariant),
          onTap: () {},
        ),
      ),
    );
  }
}

class _PantallaAlertas extends StatelessWidget {
  const _PantallaAlertas();

  @override
  Widget build(BuildContext context) {
    final cs   = Theme.of(context).colorScheme;
    final text = Theme.of(context).textTheme;

    const alertas = [
      (servidor: 'prod-db-01',  mensaje: 'CPU > 90%',         nivel: 'CRÍTICO'),
      (servidor: 'prod-web-03', mensaje: 'Disco al 85%',      nivel: 'AVISO'),
      (servidor: 'prod-api-02', mensaje: 'Reinicio inesperado', nivel: 'CRÍTICO'),
    ];

    return ListView.builder(
      padding: const EdgeInsets.all(8),
      itemCount: alertas.length,
      itemBuilder: (ctx, i) {
        final alerta = alertas[i];
        final esCritico = alerta.nivel == 'CRÍTICO';

        return Card(
          color: esCritico ? cs.errorContainer : cs.tertiaryContainer,
          child: ListTile(
            leading: Icon(
              esCritico ? Icons.error : Icons.warning,
              color: esCritico ? cs.onErrorContainer : cs.onTertiaryContainer,
            ),
            title: Text(alerta.servidor,
                style: text.titleMedium?.copyWith(fontWeight: FontWeight.bold)),
            subtitle: Text(alerta.mensaje),
            trailing: Chip(
              label: Text(alerta.nivel, style: const TextStyle(fontSize: 11)),
              backgroundColor: esCritico ? cs.error : cs.tertiary,
              labelStyle: TextStyle(
                color: esCritico ? cs.onError : cs.onTertiary,
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
        );
      },
    );
  }
}

class _PantallaAjustes extends StatelessWidget {
  const _PantallaAjustes();

  @override
  Widget build(BuildContext context) {
    return ListView(
      children: const [
        ListTile(
          leading: Icon(Icons.notifications_outlined),
          title:   Text('Notificaciones'),
          trailing: Icon(Icons.chevron_right),
        ),
        ListTile(
          leading: Icon(Icons.security_outlined),
          title:   Text('Seguridad'),
          trailing: Icon(Icons.chevron_right),
        ),
        ListTile(
          leading: Icon(Icons.info_outline),
          title:   Text('Acerca de'),
          trailing: Icon(Icons.chevron_right),
        ),
      ],
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'screens/pantalla_navegacion.dart';
```

```dart
5 => const PantallaNavegacion(),
```

Cambia `paso = 5` y guarda.

### Prueba esto

- Navega entre las 4 pestañas — observa el indicador animado bajo el ícono activo
- Vuelve a Dashboard y navega a Servidores: el Dashboard mantiene su posición de scroll
  gracias a `IndexedStack` (en vez de `Stack` que destruiría el estado)
- Toca una alerta CRÍTICA y una de AVISO — observa cómo el color de la tarjeta varía
  automáticamente con `cs.errorContainer` vs `cs.tertiaryContainer`
- Cambia `indicatorColor: cs.primaryContainer` a `cs.tertiaryContainer` —
  el indicador de la pestaña activa cambia de color

---

## Paso 6 — `SnackBar` y `AlertDialog`

`ScaffoldMessenger.of(context).showSnackBar()` muestra mensajes temporales.
`showDialog()` muestra diálogos modales y devuelve un `Future<T>`.

### Archivo: `lib/screens/pantalla_dialogs.dart`

```dart
// lib/screens/pantalla_dialogs.dart
import 'package:flutter/material.dart';

class PantallaDialogs extends StatelessWidget {
  const PantallaDialogs({super.key});

  void _mostrarSnackBar(BuildContext context, {bool esError = false}) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text(esError
            ? 'Error: no se pudo conectar al servidor'
            : 'Conexión SSH establecida correctamente'),
        backgroundColor: esError
            ? Theme.of(context).colorScheme.error
            : null,
        action: SnackBarAction(
          label:    'Deshacer',
          onPressed: () {},
        ),
        behavior:  SnackBarBehavior.floating,
        shape:     RoundedRectangleBorder(borderRadius: BorderRadius.circular(8)),
        duration:  const Duration(seconds: 3),
      ),
    );
  }

  Future<void> _mostrarConfirmacion(BuildContext context) async {
    final confirmar = await showDialog<bool>(
      context: context,
      builder: (ctx) => AlertDialog(
        icon:    const Icon(Icons.warning_amber, color: Colors.orange),
        title:   const Text('Eliminar servidor'),
        content: const Text(
          '¿Estás seguro de que deseas eliminar prod-web-01?\n'
          'Esta acción no se puede deshacer.',
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(ctx, false),
            child:     const Text('Cancelar'),
          ),
          FilledButton(
            style: FilledButton.styleFrom(
              backgroundColor: Theme.of(ctx).colorScheme.error,
            ),
            onPressed: () => Navigator.pop(ctx, true),
            child: const Text('Eliminar'),
          ),
        ],
      ),
    );

    // Verificar que el widget sigue montado antes de usar el contexto
    if (!context.mounted) return;

    if (confirmar == true) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Servidor eliminado correctamente')),
      );
    }
  }

  Future<void> _mostrarFormulario(BuildContext context) async {
    final formKey = GlobalKey<FormState>();
    final ctrlNombre = TextEditingController();
    final ctrlIp     = TextEditingController();

    await showDialog<void>(
      context: context,
      builder: (ctx) => AlertDialog(
        title: const Text('Agregar servidor'),
        content: Form(
          key: formKey,
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              TextFormField(
                controller:  ctrlNombre,
                decoration:  const InputDecoration(labelText: 'Nombre'),
                validator:   (v) => v == null || v.isEmpty ? 'Campo requerido' : null,
              ),
              const SizedBox(height: 8),
              TextFormField(
                controller: ctrlIp,
                decoration: const InputDecoration(labelText: 'Dirección IP'),
                validator:  (v) {
                  if (v == null || v.isEmpty) return 'Campo requerido';
                  final partes = v.split('.');
                  if (partes.length != 4) return 'Formato: 192.168.1.1';
                  return null;
                },
              ),
            ],
          ),
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(ctx),
            child:     const Text('Cancelar'),
          ),
          FilledButton(
            onPressed: () {
              if (formKey.currentState!.validate()) {
                Navigator.pop(ctx);
              }
            },
            child: const Text('Agregar'),
          ),
        ],
      ),
    );

    if (!context.mounted) return;
    if (ctrlNombre.text.isNotEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Servidor "${ctrlNombre.text}" agregado')),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    final cs   = Theme.of(context).colorScheme;
    final text = Theme.of(context).textTheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('SnackBar y Dialog'),
        backgroundColor: cs.surfaceContainerHighest,
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [

          // ── SnackBar ──────────────────────────────────────────────
          Text('SnackBar', style: text.labelLarge?.copyWith(color: cs.primary)),
          const SizedBox(height: 12),
          FilledButton.icon(
            onPressed: () => _mostrarSnackBar(context),
            icon:  const Icon(Icons.check_circle_outline),
            label: const Text('SnackBar de éxito'),
          ),
          const SizedBox(height: 8),
          FilledButton.icon(
            style: FilledButton.styleFrom(
              backgroundColor: cs.error,
              foregroundColor: cs.onError,
            ),
            onPressed: () => _mostrarSnackBar(context, esError: true),
            icon:  const Icon(Icons.error_outline),
            label: const Text('SnackBar de error'),
          ),

          const Divider(height: 32),

          // ── AlertDialog ───────────────────────────────────────────
          Text('AlertDialog', style: text.labelLarge?.copyWith(color: cs.primary)),
          const SizedBox(height: 12),
          OutlinedButton.icon(
            style: OutlinedButton.styleFrom(
              foregroundColor: cs.error,
              side: BorderSide(color: cs.error),
            ),
            onPressed: () => _mostrarConfirmacion(context),
            icon:  const Icon(Icons.delete_outline),
            label: const Text('Eliminar servidor (confirmación)'),
          ),
          const SizedBox(height: 8),
          FilledButton.tonal(
            onPressed: () => _mostrarFormulario(context),
            child: const Text('Agregar servidor (formulario)'),
          ),
        ],
      ),
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'screens/pantalla_dialogs.dart';
```

```dart
6 => const PantallaDialogs(),
```

Cambia `paso = 6` y guarda.

### Prueba esto

- Presiona **SnackBar de éxito** — aparece flotante con botón "Deshacer"
- Presiona **SnackBar de error** — el fondo cambia a `cs.error` (rojo del tema)
- Presiona **Eliminar servidor** → **Cancelar** — no aparece ningún SnackBar
- Presiona **Eliminar servidor** → **Eliminar** — aparece SnackBar de confirmación
- Presiona **Agregar servidor** → deja los campos vacíos → **Agregar** — aparece validación
- En el formulario, escribe una IP con formato incorrecto (p. ej. `192.168`) — valida el formato

---

## `main.dart` completo — referencia

Al terminar todos los pasos, el archivo queda así.
Cambia `paso` en cualquier momento para revisar cualquier ejemplo:

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'screens/pantalla_tema.dart';
import 'screens/pantalla_appbar.dart';
import 'widgets/catalogo_botones.dart';
import 'screens/pantalla_navegacion.dart';
import 'screens/pantalla_dialogs.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1  ThemeData + Scaffold básico                         │
// │  2  Paso 2  Modo oscuro — ThemeMode dinámico                    │
// │  3  Paso 3  AppBar variantes y SliverAppBar                     │
// │  4  Paso 4  Botones Material 3                                  │
// │  5  Paso 5  NavigationBar con 4 pestañas                        │
// │  6  Paso 6  SnackBar y AlertDialog                              │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 1;

void main() => runApp(const AppMonitoreo());

class AppMonitoreo extends StatefulWidget {
  const AppMonitoreo({super.key});
  @override
  State<AppMonitoreo> createState() => _AppMonitoreoState();
}

class _AppMonitoreoState extends State<AppMonitoreo> {
  ThemeMode _themeMode = ThemeMode.system;

  @override
  Widget build(BuildContext context) {
    const seedColor = Color(0xFF1565C0);

    return MaterialApp(
      debugShowCheckedModeBanner: false,
      themeMode: _themeMode,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
            seedColor: seedColor, brightness: Brightness.light),
        useMaterial3: true,
      ),
      darkTheme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
            seedColor: seedColor, brightness: Brightness.dark),
        useMaterial3: true,
      ),
      home: switch (paso) {
        1 => const _Paso1(),
        2 => PantallaTema(
               themeMode: _themeMode,
               onToggle:  (mode) => setState(() => _themeMode = mode),
             ),
        3 => const PantallaAppBar(),
        4 => const CatalogoBotones(),
        5 => const PantallaNavegacion(),
        6 => const PantallaDialogs(),
        _ => Scaffold(body: Center(child: Text('Paso $paso no definido'))),
      },
    );
  }
}

// ─── Paso 1 — vive en main.dart ────────────────────────────────────────
class _Paso1 extends StatelessWidget {
  const _Paso1();

  @override
  Widget build(BuildContext context) {
    final cs   = Theme.of(context).colorScheme;
    final text = Theme.of(context).textTheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Sistema de Monitoreo'),
        backgroundColor: cs.primaryContainer,
        foregroundColor: cs.onPrimaryContainer,
        actions: [
          IconButton(icon: const Icon(Icons.refresh), onPressed: () {}),
        ],
      ),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(Icons.dns, size: 64, color: cs.primary),
            const SizedBox(height: 16),
            Text(
              'Servidor web-01',
              style: text.headlineMedium?.copyWith(fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 8),
            Text(
              '10.0.2.10 · Ubuntu 24.04',
              style: text.bodyMedium?.copyWith(color: cs.onSurfaceVariant),
            ),
            const SizedBox(height: 24),
            FilledButton.icon(
              onPressed: () {},
              icon:  const Icon(Icons.terminal),
              label: const Text('Conectar SSH'),
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {},
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

---

## Proyecto — Pantalla de Ajustes con tema dinámico

Proyecto separado que combina tema dinámico, selector de seed color y
vista previa en tiempo real. Reemplaza el contenido de `lib/main.dart`
con el del proyecto (o crea uno nuevo con `flutter create`).

```
modulo08_material3/
├── lib/
│   ├── main.dart
│   └── screens/
│       └── pantalla_ajustes.dart
```

### `lib/screens/pantalla_ajustes.dart`

```dart
// lib/screens/pantalla_ajustes.dart
import 'package:flutter/material.dart';

class PantallaAjustes extends StatelessWidget {
  final ThemeMode themeMode;
  final Color     seedColor;
  final void Function(ThemeMode) onThemeMode;
  final void Function(Color)     onSeedColor;
  final List<({String nombre, Color color})> paletas;

  const PantallaAjustes({
    super.key,
    required this.themeMode,
    required this.seedColor,
    required this.onThemeMode,
    required this.onSeedColor,
    required this.paletas,
  });

  @override
  Widget build(BuildContext context) {
    final cs   = Theme.of(context).colorScheme;
    final text = Theme.of(context).textTheme;

    return Scaffold(
      appBar: AppBar(
        title:           const Text('Apariencia'),
        backgroundColor: cs.surfaceContainerHighest,
      ),
      body: ListView(
        children: [

          // ── Sección: Tema ─────────────────────────────────────────
          _SeccionTitulo('Tema'),
          ...[
            (label: 'Sistema', mode: ThemeMode.system, icon: Icons.brightness_auto),
            (label: 'Claro',   mode: ThemeMode.light,  icon: Icons.light_mode),
            (label: 'Oscuro',  mode: ThemeMode.dark,   icon: Icons.dark_mode),
          ].map((opcion) => RadioListTile<ThemeMode>(
            title:     Text(opcion.label),
            secondary: Icon(opcion.icon),
            value:     opcion.mode,
            groupValue: themeMode,
            onChanged:  (v) => onThemeMode(v!),
          )),

          const Divider(),

          // ── Sección: Color de acento ──────────────────────────────
          _SeccionTitulo('Color de acento'),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: Wrap(
              spacing: 12,
              children: paletas.map((p) {
                final seleccionado = seedColor == p.color;
                return GestureDetector(
                  onTap: () => onSeedColor(p.color),
                  child: Column(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      AnimatedContainer(
                        duration: const Duration(milliseconds: 200),
                        width:  48,
                        height: 48,
                        decoration: BoxDecoration(
                          color:  p.color,
                          shape:  BoxShape.circle,
                          border: seleccionado
                              ? Border.all(color: cs.onSurface, width: 3)
                              : null,
                          boxShadow: seleccionado
                              ? [BoxShadow(
                                  color:      p.color.withOpacity(0.5),
                                  blurRadius: 8,
                                )]
                              : null,
                        ),
                        child: seleccionado
                            ? const Icon(Icons.check, color: Colors.white)
                            : null,
                      ),
                      const SizedBox(height: 4),
                      Text(p.nombre, style: text.labelSmall),
                    ],
                  ),
                );
              }).toList(),
            ),
          ),

          const Divider(),

          // ── Sección: Vista previa ─────────────────────────────────
          _SeccionTitulo('Vista previa'),
          Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                // Tarjeta de servidor con botones
                Card(
                  child: Padding(
                    padding: const EdgeInsets.all(16),
                    child: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        Text('prod-web-01',
                            style: text.titleMedium?.copyWith(
                                fontWeight: FontWeight.bold)),
                        Text('10.0.2.10 · Ubuntu 24.04',
                            style: text.bodySmall?.copyWith(
                                color: cs.onSurfaceVariant)),
                        const SizedBox(height: 12),
                        Row(children: [
                          FilledButton.icon(
                            onPressed: () {},
                            icon:  const Icon(Icons.terminal, size: 16),
                            label: const Text('SSH'),
                          ),
                          const SizedBox(width: 8),
                          OutlinedButton(
                            onPressed: () {},
                            child: const Text('Métricas'),
                          ),
                          const SizedBox(width: 8),
                          TextButton(
                            onPressed: () {},
                            child: const Text('Logs'),
                          ),
                        ]),
                      ],
                    ),
                  ),
                ),
                const SizedBox(height: 12),
                // Paleta activa
                Text('Colores activos:',
                    style: text.labelMedium?.copyWith(
                        color: cs.onSurfaceVariant)),
                const SizedBox(height: 6),
                Row(children: [
                  _CirculoColor(color: cs.primary,                label: 'P'),
                  _CirculoColor(color: cs.secondary,              label: 'S'),
                  _CirculoColor(color: cs.tertiary,               label: 'T'),
                  _CirculoColor(color: cs.error,                  label: 'E'),
                  _CirculoColor(color: cs.surfaceContainerHighest, label: 'Sf'),
                ]),
              ],
            ),
          ),

          const SizedBox(height: 32),
        ],
      ),
    );
  }
}

class _SeccionTitulo extends StatelessWidget {
  final String texto;
  const _SeccionTitulo(this.texto);

  @override
  Widget build(BuildContext context) => Padding(
    padding: const EdgeInsets.fromLTRB(16, 16, 16, 4),
    child: Text(
      texto,
      style: Theme.of(context).textTheme.labelLarge?.copyWith(
        color: Theme.of(context).colorScheme.primary,
      ),
    ),
  );
}

class _CirculoColor extends StatelessWidget {
  final Color  color;
  final String label;
  const _CirculoColor({required this.color, required this.label});

  @override
  Widget build(BuildContext context) => Padding(
    padding: const EdgeInsets.only(right: 8),
    child: CircleAvatar(
      radius:          16,
      backgroundColor: color,
      child: Text(
        label,
        style: TextStyle(
          color: color.computeLuminance() > 0.5 ? Colors.black : Colors.white,
          fontSize:   10,
          fontWeight: FontWeight.bold,
        ),
      ),
    ),
  );
}
```

### `lib/main.dart` (proyecto)

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'screens/pantalla_ajustes.dart';

void main() => runApp(const AppAjustes());

class AppAjustes extends StatefulWidget {
  const AppAjustes({super.key});
  @override
  State<AppAjustes> createState() => _AppAjustesState();
}

class _AppAjustesState extends State<AppAjustes> {
  ThemeMode _themeMode = ThemeMode.system;
  Color     _seedColor = const Color(0xFF1565C0);

  static const _paletas = [
    (nombre: 'Azul',    color: Color(0xFF1565C0)),
    (nombre: 'Verde',   color: Color(0xFF2E7D32)),
    (nombre: 'Morado',  color: Color(0xFF6A1B9A)),
    (nombre: 'Naranja', color: Color(0xFFE65100)),
    (nombre: 'Rojo',    color: Color(0xFFC62828)),
  ];

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title:     'Sistema de Monitoreo',
      themeMode: _themeMode,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
            seedColor: _seedColor, brightness: Brightness.light),
        useMaterial3: true,
      ),
      darkTheme: ThemeData(
        colorScheme: ColorScheme.fromSeed(
            seedColor: _seedColor, brightness: Brightness.dark),
        useMaterial3: true,
      ),
      home: PantallaAjustes(
        themeMode:   _themeMode,
        seedColor:   _seedColor,
        onThemeMode: (mode)  => setState(() => _themeMode = mode),
        onSeedColor: (color) => setState(() => _seedColor = color),
        paletas:     _paletas,
      ),
    );
  }
}
```

---

## Guía rápida de imports

```dart
import 'package:flutter/material.dart';  // todo Material 3

// Acceso al tema:
final cs   = Theme.of(context).colorScheme;   // colores
final text = Theme.of(context).textTheme;     // tipografía

// Diálogos:
ScaffoldMessenger.of(context).showSnackBar(SnackBar(...));
showDialog<T>(context: context, builder: (ctx) => AlertDialog(...));
Navigator.pop(context, resultado);
```

---

## Cuándo usar cada componente M3

```
Botón           Cuándo usarlo
────────────    ────────────────────────────────────────────────────
FilledButton    Acción principal de la pantalla — solo uno por vista
FilledButton.tonal  Acción importante pero no la principal
ElevatedButton  Acción sobre fondo plano que necesita diferenciarse
OutlinedButton  Acción secundaria con igual importancia visual
TextButton      Acciones en diálogos, cards o junto a un FilledButton

Navegación      Cuándo usarlo
────────────    ────────────────────────────────────────────────────
NavigationBar   2–5 destinos principales en móvil
NavigationRail  2–7 destinos en tablet/pantallas anchas (> 600px)
Drawer          Muchos destinos (> 5) o navegación de segundo nivel
```

---

## Ejercicios propuestos

1. **Tema dinámico con DataStore** — Extiende el proyecto para que
   la preferencia de tema (claro/oscuro/sistema) y el color de acento se
   guarden en `SharedPreferences`. Al reiniciar la app, los valores se
   restauran. Usa `FutureBuilder` para esperar la carga inicial.

2. **AppBar con búsqueda** — Crea un `AppBar` que tenga un `IconButton` de
   búsqueda. Al presionarlo, el título se reemplaza por un `TextField` con
   foco automático. Al borrar o presionar escape, vuelve al título original.
   Usa `AnimatedSwitcher` para la transición.

3. **NavigationRail** — Para pantallas con ancho > 600px, reemplaza la
   `NavigationBar` inferior por un `NavigationRail` lateral usando
   `LayoutBuilder`. El rail debe mostrar iconos y etiquetas cuando el ancho
   supera 800px y solo iconos entre 600 y 800px.

4. **RichText en las alertas** — En `_PantallaAlertas` del Paso 5, reemplaza
   el `subtitle` de cada `ListTile` por un `RichText` que muestre el nombre
   del servidor en negrita y el mensaje en color `cs.onSurfaceVariant`.

---

## Resumen de la página 8

- `ColorScheme.fromSeed(seedColor: color)` genera automáticamente toda la paleta — primario, secundario, terciario, error y superficies. Solo cambia el `seedColor`.
- `themeMode: ThemeMode.system/light/dark` y `darkTheme:` permiten modo oscuro automático sin cambiar los widgets individuales.
- `Scaffold` provee la estructura estándar: `appBar`, `body`, `floatingActionButton`, `bottomNavigationBar` y `drawer`.
- Los colores siempre se obtienen del tema: `Theme.of(context).colorScheme.primary`. Nunca usar `Colors.blue` directamente en código de producción.
- Los 5 botones de M3 van de mayor a menor énfasis: `FilledButton` → `FilledButton.tonal` → `ElevatedButton` → `OutlinedButton` → `TextButton`.
- `NavigationBar` es el componente M3 para navegación inferior. `IndexedStack` mantiene el estado de cada pestaña activa.
- `ScaffoldMessenger.of(context).showSnackBar()` muestra mensajes temporales. `showDialog()` muestra diálogos modales que devuelven un `Future<T>`.
- Verifica `context.mounted` antes de usar el `BuildContext` después de un `await`.
- `SliverAppBar.large` colapsa al hacer scroll — `pinned: true` mantiene el título visible.

---

> **Siguiente página →** Página 9: Formularios y listas —
> `TextField`, `Form`, `ListView`, `GridView` y `ListTile`.
