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

## `ThemeData` y `ColorScheme`

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

    // Tipografía personalizada
    textTheme: const TextTheme(
      displayLarge:  TextStyle(fontSize: 57, fontWeight: FontWeight.w400),
      headlineLarge: TextStyle(fontSize: 32, fontWeight: FontWeight.w600),
      titleLarge:    TextStyle(fontSize: 22, fontWeight: FontWeight.w600),
      titleMedium:   TextStyle(fontSize: 16, fontWeight: FontWeight.w500),
      bodyLarge:     TextStyle(fontSize: 16, fontWeight: FontWeight.w400),
      bodyMedium:    TextStyle(fontSize: 14, fontWeight: FontWeight.w400),
      labelLarge:    TextStyle(fontSize: 14, fontWeight: FontWeight.w500),
      labelSmall:    TextStyle(fontSize: 11, fontWeight: FontWeight.w500),
    ),

    // Forma global de los componentes
    shape: RoundedRectangleBorder(
      borderRadius: BorderRadius.circular(12),
    ),
  ),
)
```

### Los roles de color de Material 3

```dart
// Acceder a los colores del tema en cualquier widget
Widget build(BuildContext context) {
  final cs = Theme.of(context).colorScheme;

  return Column(children: [
    // Roles principales
    Container(color: cs.primary,           child: Text('primary',           style: TextStyle(color: cs.onPrimary))),
    Container(color: cs.primaryContainer,  child: Text('primaryContainer',  style: TextStyle(color: cs.onPrimaryContainer))),
    Container(color: cs.secondary,         child: Text('secondary',         style: TextStyle(color: cs.onSecondary))),
    Container(color: cs.tertiary,          child: Text('tertiary',          style: TextStyle(color: cs.onTertiary))),
    Container(color: cs.error,             child: Text('error',             style: TextStyle(color: cs.onError))),

    // Roles de superficie
    Container(color: cs.surface,                    child: Text('surface')),
    Container(color: cs.surfaceContainerHighest,    child: Text('surfaceContainerHighest')),
    Container(color: cs.surfaceContainerLow,        child: Text('surfaceContainerLow')),

    // Texto sobre superficie
    Text('onSurface',        style: TextStyle(color: cs.onSurface)),
    Text('onSurfaceVariant', style: TextStyle(color: cs.onSurfaceVariant)),
  ]);
}
```

---

## Modo oscuro

```dart
class AppRaiz extends StatefulWidget {
  const AppRaiz({super.key});

  @override
  State<AppRaiz> createState() => _AppRaizState();
}

class _AppRaizState extends State<AppRaiz> {
  ThemeMode _modoTema = ThemeMode.system;

  void _toggleTema() {
    setState(() {
      _modoTema = _modoTema == ThemeMode.light
          ? ThemeMode.dark
          : ThemeMode.light;
    });
  }

  @override
  Widget build(BuildContext context) {
    const seedColor = Color(0xFF1565C0);

    return MaterialApp(
      title:     'Mi App',
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
          brightness: Brightness.dark,   // ← diferencia clave
        ),
        useMaterial3: true,
      ),

      home: PantallaInicio(onToggleTema: _toggleTema),
    );
  }
}
```

---

## `Scaffold` — estructura de pantalla completa

`Scaffold` implementa la estructura visual básica de Material Design:

```dart
Scaffold(
  // Barra superior
  appBar: AppBar(
    title:   const Text('Título'),
    actions: [
      IconButton(icon: const Icon(Icons.search),   onPressed: () {}),
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
      NavigationDestination(icon: Icon(Icons.home),    label: 'Inicio'),
      NavigationDestination(icon: Icon(Icons.settings), label: 'Ajustes'),
    ],
    selectedIndex:  0,
    onDestinationSelected: (index) {},
  ),

  // Drawer lateral
  drawer: const Drawer(child: Text('Menú lateral')),

  // Snackbar host
  // Se muestra con ScaffoldMessenger.of(context).showSnackBar(...)
)
```

### `AppBar` con variantes M3

```dart
// AppBar estándar
AppBar(
  title:            const Text('Servidores'),
  backgroundColor:  Theme.of(context).colorScheme.primaryContainer,
  foregroundColor:  Theme.of(context).colorScheme.onPrimaryContainer,
  elevation:        0,
  centerTitle:      false,
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
      title:      const Text('Dashboard'),
      pinned:     true,    // siempre visible al menos el título
      floating:   false,
      expandedHeight: 200,
      flexibleSpace: FlexibleSpaceBar(
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

## Componentes de texto y botones

### `Text` con estilo

```dart
// Estilos del tema
Text('Título grande',    style: Theme.of(context).textTheme.headlineLarge)
Text('Cuerpo normal',   style: Theme.of(context).textTheme.bodyMedium)
Text('Etiqueta pequeña', style: Theme.of(context).textTheme.labelSmall)

// Personalización sobre el estilo del tema
Text(
  'Texto personalizado',
  style: Theme.of(context).textTheme.bodyLarge?.copyWith(
    color:      Theme.of(context).colorScheme.primary,
    fontWeight: FontWeight.bold,
    decoration: TextDecoration.underline,
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
        style: const TextStyle(
          color:      Colors.green,
          fontWeight: FontWeight.bold,
        ),
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

### Botones en Material 3

```dart
// Los 5 variantes de botón M3
FilledButton(          // botón principal — mayor énfasis
  onPressed: () {},
  child: const Text('Confirmar'),
)

FilledButton.tonal(    // énfasis medio con color secundario
  onPressed: () {},
  child: const Text('Guardar borrador'),
)

ElevatedButton(        // énfasis medio con sombra
  onPressed: () {},
  child: const Text('Continuar'),
)

OutlinedButton(        // énfasis bajo con borde
  onPressed: () {},
  child: const Text('Cancelar'),
)

TextButton(            // énfasis mínimo
  onPressed: () {},
  child: const Text('Ver más'),
)

// Con icono
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
    shape: RoundedRectangleBorder(
      borderRadius: BorderRadius.circular(8),
    ),
  ),
  onPressed: () {},
  child: const Text('Eliminar servidor'),
)
```

---

## `NavigationBar` — navegación inferior M3

```dart
class AppConNavegacion extends StatefulWidget {
  const AppConNavegacion({super.key});

  @override
  State<AppConNavegacion> createState() => _AppConNavegacionState();
}

class _AppConNavegacionState extends State<AppConNavegacion> {
  int _indiceActual = 0;

  // Pantallas correspondientes a cada destino
  static const _pantallas = [
    _PantallaDashboard(),
    _PantallaServidores(),
    _PantallaAlertas(),
    _PantallaAjustes(),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: _pantallas[_indiceActual],

      bottomNavigationBar: NavigationBar(
        selectedIndex:         _indiceActual,
        onDestinationSelected: (i) => setState(() => _indiceActual = i),
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

// Pantallas placeholder
class _PantallaDashboard  extends StatelessWidget {
  const _PantallaDashboard();
  @override
  Widget build(BuildContext context) => const Center(child: Text('Dashboard'));
}
class _PantallaServidores extends StatelessWidget {
  const _PantallaServidores();
  @override
  Widget build(BuildContext context) => const Center(child: Text('Servidores'));
}
class _PantallaAlertas    extends StatelessWidget {
  const _PantallaAlertas();
  @override
  Widget build(BuildContext context) => const Center(child: Text('Alertas'));
}
class _PantallaAjustes    extends StatelessWidget {
  const _PantallaAjustes();
  @override
  Widget build(BuildContext context) => const Center(child: Text('Ajustes'));
}
```

---

## `SnackBar` y `Dialog`

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
      shape:     RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(8),
      ),
      duration: const Duration(seconds: 3),
    ),
  );
}

// AlertDialog — confirmación de acción crítica
Future<bool?> mostrarConfirmacion(
  BuildContext context, {
  required String titulo,
  required String mensaje,
  String textoConfirmar = 'Confirmar',
  String textoCancelar  = 'Cancelar',
  bool   esDestructivo  = false,
}) {
  return showDialog<bool>(
    context: context,
    builder: (ctx) => AlertDialog(
      icon:    esDestructivo
          ? const Icon(Icons.warning_amber, color: Colors.orange)
          : null,
      title:   Text(titulo),
      content: Text(mensaje),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(ctx, false),
          child:     Text(textoCancelar),
        ),
        FilledButton(
          style: esDestructivo
              ? FilledButton.styleFrom(
                  backgroundColor: Theme.of(ctx).colorScheme.error,
                )
              : null,
          onPressed: () => Navigator.pop(ctx, true),
          child: Text(textoConfirmar),
        ),
      ],
    ),
  );
}
```

---

## Programa completo — pantalla de ajustes con tema dinámico

```dart
import 'package:flutter/material.dart';

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
      theme:     ThemeData(
        colorScheme:  ColorScheme.fromSeed(
            seedColor: _seedColor, brightness: Brightness.light),
        useMaterial3: true,
      ),
      darkTheme: ThemeData(
        colorScheme:  ColorScheme.fromSeed(
            seedColor: _seedColor, brightness: Brightness.dark),
        useMaterial3: true,
      ),
      home: _PantallaAjustesCompleta(
        themeMode:    _themeMode,
        seedColor:    _seedColor,
        onThemeMode:  (mode)  => setState(() => _themeMode = mode),
        onSeedColor:  (color) => setState(() => _seedColor = color),
        paletas:      _paletas,
      ),
    );
  }
}

class _PantallaAjustesCompleta extends StatelessWidget {
  final ThemeMode _themeMode;
  final Color     _seedColor;
  final void Function(ThemeMode)  onThemeMode;
  final void Function(Color)      onSeedColor;
  final List<({String nombre, Color color})> paletas;

  const _PantallaAjustesCompleta({
    required ThemeMode themeMode,
    required Color     seedColor,
    required this.onThemeMode,
    required this.onSeedColor,
    required this.paletas,
  })  : _themeMode = themeMode,
        _seedColor  = seedColor;

  @override
  Widget build(BuildContext context) {
    final cs   = Theme.of(context).colorScheme;
    final text = Theme.of(context).textTheme;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Apariencia'),
        backgroundColor: cs.surfaceContainerHighest,
      ),
      body: ListView(
        children: [

          // ── Sección: Tema ───────────────────────────────────────
          _SeccionTitulo('Tema'),
          ...[
            (label: 'Sistema', mode: ThemeMode.system,  icon: Icons.brightness_auto),
            (label: 'Claro',   mode: ThemeMode.light,   icon: Icons.light_mode),
            (label: 'Oscuro',  mode: ThemeMode.dark,    icon: Icons.dark_mode),
          ].map((opcion) => RadioListTile<ThemeMode>(
            title:    Text(opcion.label),
            secondary: Icon(opcion.icon),
            value:    opcion.mode,
            groupValue: _themeMode,
            onChanged: (v) => onThemeMode(v!),
          )),

          const Divider(),

          // ── Sección: Color de acento ────────────────────────────
          _SeccionTitulo('Color de acento'),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
            child: Wrap(
              spacing: 12,
              children: paletas.map((p) {
                final seleccionado = _seedColor == p.color;
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

          // ── Sección: Vista previa ───────────────────────────────
          _SeccionTitulo('Vista previa'),
          Padding(
            padding: const EdgeInsets.all(16),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                // Muestra los colores activos del tema
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
                            child:     const Text('Métricas'),
                          ),
                          const SizedBox(width: 8),
                          TextButton(
                            onPressed: () {},
                            child:     const Text('Logs'),
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
                  _CirculoColor(color: cs.primary,           label: 'P'),
                  _CirculoColor(color: cs.secondary,         label: 'S'),
                  _CirculoColor(color: cs.tertiary,          label: 'T'),
                  _CirculoColor(color: cs.error,             label: 'E'),
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
          color:    color.computeLuminance() > 0.5
              ? Colors.black
              : Colors.white,
          fontSize: 10,
          fontWeight: FontWeight.bold,
        ),
      ),
    ),
  );
}
```

---

## Ejercicios propuestos

1. **Tema dinámico con DataStore** — Extiende el programa completo para que
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

4. **Dialog de formulario** — Crea un `showDialog` que contenga un
   formulario con `TextFormField` para nombre y IP de un servidor nuevo.
   Valida que la IP tenga formato correcto antes de cerrar el dialog con
   el resultado. Usa `Form` y `GlobalKey<FormState>`.

---

## Resumen de la página 8

- `ColorScheme.fromSeed(seedColor: color)` genera automáticamente toda la paleta — primario, secundario, terciario, error y superficies. Solo cambia el `seedColor`.
- `themeMode: ThemeMode.system/light/dark` y `darkTheme:` permiten modo oscuro automático sin cambiar los widgets individuales.
- `Scaffold` provee la estructura estándar: `appBar`, `body`, `floatingActionButton`, `bottomNavigationBar` y `drawer`.
- Los colores siempre se obtienen del tema: `Theme.of(context).colorScheme.primary`. Nunca usar `Colors.blue` directamente en código de producción.
- Los 5 botones de M3 van de mayor a menor énfasis: `FilledButton` → `FilledButton.tonal` → `ElevatedButton` → `OutlinedButton` → `TextButton`.
- `NavigationBar` es el componente M3 para navegación inferior. Los íconos rellenos/contorno para el estado activo/inactivo son el estándar de M3.
- `ScaffoldMessenger.of(context).showSnackBar()` muestra mensajes temporales. `showDialog()` muestra diálogos modales que devuelven un `Future<T>`.
- `RichText` con `TextSpan` permite múltiples estilos en la misma línea de texto.
- `SliverAppBar.large` colapsa al hacer scroll — `pinned: true` mantiene el título visible.

---

> **Siguiente página →** Página 9: Formularios y listas —
> `TextField`, `Form`, `ListView`, `GridView` y `ListTile`.