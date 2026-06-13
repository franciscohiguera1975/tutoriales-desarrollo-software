# Tutorial Flutter — Página 18
## Módulo 6 · UI Avanzada
### Animaciones — construcción incremental paso a paso

---

## Crear el proyecto

```bash
flutter create animaciones_app
cd animaciones_app
```

### `pubspec.yaml`

```yaml
name: animaciones_app
description: "Demo de animaciones en Flutter"
publish_to: none
version: 1.0.0+1

environment:
  sdk: ^3.7.0

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^3.3.0
  lottie:           ^3.1.3   # animaciones Lottie (JSON)

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^5.0.0

flutter:
  uses-material-design: true
  assets:
    - assets/lottie/      # carpeta para archivos .json de Lottie
```

```bash
# Crear la carpeta de assets
mkdir -p assets/lottie
flutter pub get
```

---

## Estructura final del proyecto

```
animaciones_app/
├── assets/
│   └── lottie/
│       └── loading.json          ← descargar de lottiefiles.com
├── lib/
│   ├── main.dart
│   └── features/
│       ├── paso1_implicity.dart  ← AnimatedContainer, AnimatedOpacity, etc.
│       ├── paso2_explicit.dart   ← AnimationController + Tween
│       ├── paso3_hero.dart       ← Hero transitions
│       ├── paso4_page.dart       ← PageRouteBuilder
│       └── paso5_lottie.dart     ← Lottie
└── pubspec.yaml
```

---

## Paso 1 — App mínima

**`lib/main.dart`** — reemplaza todo:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() {
  runApp(const ProviderScope(child: AnimacionesApp()));
}

class AnimacionesApp extends StatelessWidget {
  const AnimacionesApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Animaciones',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const PantallaMenu(),
    );
  }
}

class PantallaMenu extends StatelessWidget {
  const PantallaMenu({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Animaciones')),
      body: const Center(child: Text('Paso 1 — App funcionando ✅')),
    );
  }
}
```

```bash
flutter run
# Debe mostrar "Paso 1 — App funcionando ✅"
```

---

## Paso 2 — Animaciones implícitas

Las animaciones **implícitas** son las más sencillas de usar.
Flutter interpola automáticamente entre valores cuando cambian.
No necesitas `AnimationController` ni `Ticker`:

```
AnimatedContainer    → cambia tamaño, color, decoración
AnimatedOpacity      → cambia transparencia
AnimatedPadding      → cambia espacio interior
AnimatedAlign        → cambia posición
AnimatedCrossFade    → alterna entre dos widgets
AnimatedSwitcher     → anima el cambio de cualquier widget
AnimatedDefaultTextStyle → cambia estilo de texto
```

**Crea `lib/features/paso1_implicity.dart`:**

```dart
import 'package:flutter/material.dart';

class PasoAnimacionesImplicitas extends StatefulWidget {
  const PasoAnimacionesImplicitas({super.key});

  @override
  State<PasoAnimacionesImplicitas> createState() =>
      _PasoAnimacionesImplicitasState();
}

class _PasoAnimacionesImplicitasState
    extends State<PasoAnimacionesImplicitas> {
  // Estado que controla las animaciones
  bool _expandido   = false;
  bool _visible     = true;
  bool _alineadoDerecha = false;
  bool _destacado   = false;
  int  _colorIndice = 0;

  static const _colores = [
    Colors.teal,
    Colors.deepOrange,
    Colors.purple,
    Colors.indigo,
  ];

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(title: const Text('Animaciones implícitas')),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [

            // ── AnimatedContainer ────────────────────────────────
            _Seccion('AnimatedContainer'),
            Text(
              'Cambia tamaño, color y bordes automáticamente.',
              style: TextStyle(color: cs.onSurfaceVariant, fontSize: 13),
            ),
            const SizedBox(height: 12),
            Center(
              child: AnimatedContainer(
                // Duración de la animación — obligatoria
                duration: const Duration(milliseconds: 400),
                // Curva de interpolación (optional, default = linear)
                curve: Curves.easeInOut,

                width:  _expandido ? 250 : 100,
                height: _expandido ? 150 : 100,
                decoration: BoxDecoration(
                  color:        _colores[_colorIndice],
                  borderRadius: BorderRadius.circular(
                      _expandido ? 32 : 8),
                  boxShadow: _expandido
                      ? [BoxShadow(
                          color:     _colores[_colorIndice].withOpacity(0.4),
                          blurRadius: 20,
                          offset:    const Offset(0, 8),
                        )]
                      : [],
                ),
                child: Center(
                  child: Text(
                    _expandido ? 'Grande' : 'Pequeño',
                    style: const TextStyle(
                        color: Colors.white, fontWeight: FontWeight.bold),
                  ),
                ),
              ),
            ),
            const SizedBox(height: 12),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                FilledButton(
                  onPressed: () => setState(() => _expandido = !_expandido),
                  child: Text(_expandido ? 'Encoger' : 'Expandir'),
                ),
                const SizedBox(width: 8),
                OutlinedButton(
                  onPressed: () => setState(() =>
                      _colorIndice = (_colorIndice + 1) % _colores.length),
                  child: const Text('Cambiar color'),
                ),
              ],
            ),

            const SizedBox(height: 24),

            // ── AnimatedOpacity ──────────────────────────────────
            _Seccion('AnimatedOpacity'),
            Center(
              child: AnimatedOpacity(
                duration: const Duration(milliseconds: 500),
                opacity:  _visible ? 1.0 : 0.0,
                child: Container(
                  width: 200, height: 80,
                  decoration: BoxDecoration(
                    color:        cs.primaryContainer,
                    borderRadius: BorderRadius.circular(12),
                  ),
                  child: Center(
                    child: Text('Soy visible',
                        style: TextStyle(color: cs.onPrimaryContainer,
                            fontWeight: FontWeight.bold)),
                  ),
                ),
              ),
            ),
            const SizedBox(height: 8),
            Center(
              child: FilledButton.tonal(
                onPressed: () => setState(() => _visible = !_visible),
                child: Text(_visible ? 'Ocultar' : 'Mostrar'),
              ),
            ),

            const SizedBox(height: 24),

            // ── AnimatedAlign ────────────────────────────────────
            _Seccion('AnimatedAlign'),
            Container(
              height: 80,
              decoration: BoxDecoration(
                color:        cs.surfaceContainerHighest,
                borderRadius: BorderRadius.circular(12),
              ),
              child: AnimatedAlign(
                duration:  const Duration(milliseconds: 600),
                curve:     Curves.bounceOut,
                alignment: _alineadoDerecha
                    ? Alignment.centerRight
                    : Alignment.centerLeft,
                child: Padding(
                  padding: const EdgeInsets.all(8),
                  child: Container(
                    width: 56, height: 56,
                    decoration: BoxDecoration(
                      color:  cs.primary,
                      shape:  BoxShape.circle,
                    ),
                    child: Icon(Icons.circle, color: cs.onPrimary),
                  ),
                ),
              ),
            ),
            const SizedBox(height: 8),
            Center(
              child: FilledButton.tonal(
                onPressed: () =>
                    setState(() => _alineadoDerecha = !_alineadoDerecha),
                child: const Text('Mover'),
              ),
            ),

            const SizedBox(height: 24),

            // ── AnimatedDefaultTextStyle ──────────────────────────
            _Seccion('AnimatedDefaultTextStyle'),
            Center(
              child: AnimatedDefaultTextStyle(
                duration: const Duration(milliseconds: 300),
                style: TextStyle(
                  fontSize:   _destacado ? 28 : 16,
                  fontWeight: _destacado
                      ? FontWeight.bold
                      : FontWeight.normal,
                  color: _destacado ? cs.primary : cs.onSurface,
                ),
                child: const Text('Texto animado'),
              ),
            ),
            const SizedBox(height: 8),
            Center(
              child: FilledButton.tonal(
                onPressed: () =>
                    setState(() => _destacado = !_destacado),
                child: const Text('Destacar'),
              ),
            ),

            const SizedBox(height: 24),

            // ── AnimatedSwitcher ─────────────────────────────────
            _Seccion('AnimatedSwitcher'),
            Text(
              'Anima el cambio entre cualquier par de widgets.',
              style: TextStyle(color: cs.onSurfaceVariant, fontSize: 13),
            ),
            const SizedBox(height: 12),
            Center(
              child: AnimatedSwitcher(
                duration: const Duration(milliseconds: 400),
                // transitionBuilder personaliza el tipo de animación
                transitionBuilder: (child, animation) => FadeTransition(
                  opacity: animation,
                  child: ScaleTransition(scale: animation, child: child),
                ),
                child: _destacado
                    ? Container(
                        key: const ValueKey('grande'),
                        width: 120, height: 120,
                        decoration: BoxDecoration(
                          color:        cs.primary,
                          borderRadius: BorderRadius.circular(20),
                        ),
                        child: Icon(Icons.star,
                            color: cs.onPrimary, size: 48),
                      )
                    : Container(
                        key: const ValueKey('pequenio'),
                        width: 60, height: 60,
                        decoration: BoxDecoration(
                          color:        cs.primaryContainer,
                          borderRadius: BorderRadius.circular(10),
                        ),
                        child: Icon(Icons.star_border,
                            color: cs.primary),
                      ),
              ),
            ),
            const SizedBox(height: 80),
          ],
        ),
      ),
    );
  }
}

class _Seccion extends StatelessWidget {
  final String texto;
  const _Seccion(this.texto);

  @override
  Widget build(BuildContext context) => Padding(
    padding: const EdgeInsets.only(bottom: 8),
    child: Text(texto,
        style: Theme.of(context)
            .textTheme
            .titleSmall
            ?.copyWith(
                fontWeight: FontWeight.bold,
                color: Theme.of(context).colorScheme.primary)),
  );
}
```

Agrega navegación en `PantallaMenu`:

```dart
// lib/main.dart — reemplaza PantallaMenu

import 'features/paso1_implicity.dart';

class PantallaMenu extends StatelessWidget {
  const PantallaMenu({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Animaciones')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          _ItemMenu(
            titulo: '1 — Animaciones implícitas',
            subtitulo: 'AnimatedContainer, AnimatedOpacity...',
            onTap: () => Navigator.push(context,
                MaterialPageRoute(
                    builder: (_) =>
                        const PasoAnimacionesImplicitas())),
          ),
        ],
      ),
    );
  }
}

class _ItemMenu extends StatelessWidget {
  final String       titulo;
  final String       subtitulo;
  final VoidCallback onTap;

  const _ItemMenu({
    required this.titulo,
    required this.subtitulo,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) => Card(
    margin: const EdgeInsets.only(bottom: 8),
    child: ListTile(
      title:    Text(titulo,
          style: const TextStyle(fontWeight: FontWeight.bold)),
      subtitle: Text(subtitulo),
      trailing: const Icon(Icons.arrow_forward_ios, size: 16),
      onTap:    onTap,
    ),
  );
}
```

```bash
flutter run
# Toca "1 — Animaciones implícitas"
# → Presiona "Expandir/Encoger" → el container cambia de tamaño suavemente
# → Presiona "Cambiar color" → el color cambia con interpolación
# → Presiona "Ocultar/Mostrar" → el widget se desvanece
# → Presiona "Mover" → el círculo se desliza con rebote (Curves.bounceOut)
# → Presiona "Destacar" → texto crece y el AnimatedSwitcher intercambia widgets
```

---

## Paso 3 — Animaciones explícitas

Las animaciones **explícitas** te dan control total sobre la
animación: cuándo empieza, cuándo para, cuántas veces se repite.
Requieren `AnimationController` + `Tween`:

```
AnimationController  → controla duración, dirección y estado
Tween<T>             → define el rango de valores (inicio → fin)
CurvedAnimation      → aplica una curva al controlador
AnimatedBuilder      → reconstruye el widget en cada frame
```

**Crea `lib/features/paso2_explicit.dart`:**

```dart
import 'package:flutter/material.dart';

class PasoAnimacionesExplicitas extends StatefulWidget {
  const PasoAnimacionesExplicitas({super.key});

  @override
  State<PasoAnimacionesExplicitas> createState() =>
      _PasoAnimacionesExplicitasState();
}

class _PasoAnimacionesExplicitasState
    extends State<PasoAnimacionesExplicitas>
    with SingleTickerProviderStateMixin {
  // SingleTickerProviderStateMixin provee el "ticker" que
  // sincroniza la animación con los frames de la pantalla

  late final AnimationController _ctrl;

  // Tweens — rangos de valores para cada propiedad animada
  late final Animation<double>  _escala;
  late final Animation<double>  _rotacion;
  late final Animation<Color?>  _color;
  late final Animation<double>  _opacidad;

  @override
  void initState() {
    super.initState();

    _ctrl = AnimationController(
      vsync:    this,          // this es el TickerProvider
      duration: const Duration(milliseconds: 1200),
    );

    // CurvedAnimation aplica una curva al controlador (0.0 → 1.0)
    final curveAnim = CurvedAnimation(
      parent: _ctrl,
      curve:  Curves.elasticOut,
    );

    // Cada Tween interpola entre dos valores
    _escala = Tween<double>(begin: 0.5, end: 1.0).animate(curveAnim);

    _rotacion = Tween<double>(
      begin: 0.0,
      end:   2 * 3.14159,    // 360 grados en radianes
    ).animate(CurvedAnimation(
      parent: _ctrl,
      curve:  Curves.easeInOut,
    ));

    _color = ColorTween(
      begin: Colors.teal,
      end:   Colors.deepOrange,
    ).animate(CurvedAnimation(
      parent: _ctrl,
      curve:  Curves.easeIn,
    ));

    _opacidad = Tween<double>(begin: 0.0, end: 1.0)
        .animate(CurvedAnimation(
          parent: _ctrl,
          // Interval: esta animación solo ocurre entre 0% y 50% del total
          curve:  const Interval(0.0, 0.5, curve: Curves.easeIn),
        ));
  }

  @override
  void dispose() {
    _ctrl.dispose();   // SIEMPRE liberar el controller
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Animaciones explícitas')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [

            // AnimatedBuilder reconstruye SOLO este subtree
            // en cada frame — más eficiente que setState
            AnimatedBuilder(
              animation: _ctrl,
              builder: (context, child) {
                return Opacity(
                  opacity: _opacidad.value,
                  child: Transform.scale(
                    scale: _escala.value,
                    child: Transform.rotate(
                      angle: _rotacion.value,
                      child: Container(
                        width:  120,
                        height: 120,
                        decoration: BoxDecoration(
                          color:        _color.value,
                          borderRadius: BorderRadius.circular(20),
                          boxShadow: [
                            BoxShadow(
                              color:      (_color.value ?? Colors.teal)
                                  .withOpacity(0.4),
                              blurRadius: 20,
                              offset:     const Offset(0, 8),
                            ),
                          ],
                        ),
                        // child estático — no se reconstruye en cada frame
                        child: child,
                      ),
                    ),
                  ),
                );
              },
              // El child estático se pasa aquí — se construye una sola vez
              child: const Icon(Icons.flutter_dash,
                  color: Colors.white, size: 60),
            ),

            const SizedBox(height: 40),

            // Controles de la animación
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                FilledButton(
                  onPressed: () => _ctrl.forward(from: 0.0),
                  child: const Text('Reproducir'),
                ),
                const SizedBox(width: 8),
                OutlinedButton(
                  onPressed: () => _ctrl.reverse(),
                  child: const Text('Reversa'),
                ),
                const SizedBox(width: 8),
                FilledButton.tonal(
                  onPressed: () => _ctrl.repeat(reverse: true),
                  child: const Text('Loop'),
                ),
              ],
            ),
            const SizedBox(height: 12),
            OutlinedButton(
              onPressed: () => _ctrl.stop(),
              child: const Text('Detener'),
            ),

            const SizedBox(height: 24),

            // Mostrar el valor actual del controlador
            AnimatedBuilder(
              animation: _ctrl,
              builder: (_, __) => Text(
                'Progreso: ${(_ctrl.value * 100).toStringAsFixed(0)}%',
                style: const TextStyle(fontSize: 16),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

Añade la entrada al menú en `main.dart`:

```dart
// lib/main.dart — añadir dentro de ListView de PantallaMenu

import 'features/paso2_explicit.dart';

// Añadir después del primer _ItemMenu:
_ItemMenu(
  titulo:   '2 — Animaciones explícitas',
  subtitulo: 'AnimationController, Tween, CurvedAnimation',
  onTap: () => Navigator.push(context,
      MaterialPageRoute(
          builder: (_) =>
              const PasoAnimacionesExplicitas())),
),
```

```bash
flutter run
# → Presiona "Reproducir" → el ícono aparece rotando y escalando
# → Presiona "Reversa" → vuelve al estado inicial
# → Presiona "Loop" → cicla hacia adelante y atrás indefinidamente
# → Presiona "Detener" → para en el frame actual
# → El porcentaje de progreso muestra el valor del controller en tiempo real
```

---

## Paso 4 — Animaciones Hero

`Hero` anima la transición de un widget entre dos rutas.
El sistema busca el `Hero` con el mismo `tag` en ambas pantallas
y anima la transición del tamaño y posición automáticamente:

**Crea `lib/features/paso3_hero.dart`:**

```dart
import 'package:flutter/material.dart';

// Datos de ejemplo — servidores con "avatar" que hace Hero
class _ServidorItem {
  final String id;
  final String nombre;
  final String ip;
  final Color  color;
  final IconData icono;

  const _ServidorItem({
    required this.id,
    required this.nombre,
    required this.ip,
    required this.color,
    required this.icono,
  });
}

const _servidores = [
  _ServidorItem(id: '1', nombre: 'prod-web-01',
      ip: '10.0.2.10', color: Colors.teal,       icono: Icons.web),
  _ServidorItem(id: '2', nombre: 'prod-db-01',
      ip: '10.0.2.20', color: Colors.deepOrange,  icono: Icons.storage),
  _ServidorItem(id: '3', nombre: 'cache-redis',
      ip: '10.0.2.30', color: Colors.purple,      icono: Icons.memory),
];

// Pantalla de lista
class PasoHeroLista extends StatelessWidget {
  const PasoHeroLista({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Hero — Lista')),
      body: ListView.builder(
        padding:     const EdgeInsets.all(16),
        itemCount:   _servidores.length,
        itemBuilder: (_, i) {
          final s = _servidores[i];
          return Card(
            margin: const EdgeInsets.only(bottom: 12),
            child: ListTile(
              contentPadding: const EdgeInsets.all(12),
              // El Hero envuelve el avatar — debe tener el mismo tag en destino
              leading: Hero(
                tag: 'avatar-${s.id}',    // tag único por elemento
                child: CircleAvatar(
                  radius:          28,
                  backgroundColor: s.color,
                  child: Icon(s.icono, color: Colors.white, size: 24),
                ),
              ),
              title:    Text(s.nombre,
                  style: const TextStyle(fontWeight: FontWeight.bold)),
              subtitle: Text(s.ip),
              trailing: const Icon(Icons.arrow_forward_ios, size: 16),
              onTap: () => Navigator.push(
                context,
                MaterialPageRoute(
                  builder: (_) => PasoHeroDetalle(servidor: s),
                ),
              ),
            ),
          );
        },
      ),
    );
  }
}

// Pantalla de detalle
class PasoHeroDetalle extends StatelessWidget {
  final _ServidorItem servidor;
  const PasoHeroDetalle({super.key, required this.servidor});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(servidor.nombre)),
      body: Column(
        children: [
          // Hero en destino — mismo tag, tamaño mayor
          Hero(
            tag: 'avatar-${servidor.id}',
            child: Container(
              width:  double.infinity,
              height: 220,
              color:  servidor.color,
              child: Icon(servidor.icono,
                  color: Colors.white.withOpacity(0.8), size: 100),
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(24),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(servidor.nombre,
                    style: Theme.of(context).textTheme.headlineSmall
                        ?.copyWith(fontWeight: FontWeight.bold)),
                const SizedBox(height: 8),
                Text('IP: ${servidor.ip}',
                    style: Theme.of(context).textTheme.bodyLarge),
                const SizedBox(height: 24),
                FilledButton.icon(
                  onPressed: () => Navigator.pop(context),
                  icon:  const Icon(Icons.arrow_back),
                  label: const Text('Volver'),
                  style: FilledButton.styleFrom(
                      backgroundColor: servidor.color),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

Añade al menú:

```dart
// lib/main.dart — añadir a PantallaMenu

import 'features/paso3_hero.dart';

_ItemMenu(
  titulo:   '3 — Hero transitions',
  subtitulo: 'Animación compartida entre pantallas',
  onTap: () => Navigator.push(context,
      MaterialPageRoute(builder: (_) => const PasoHeroLista())),
),
```

```bash
flutter run
# → Toca cualquier servidor de la lista
# → El avatar circular vuela hasta convertirse en el banner del detalle
# → Presiona "Volver" → el banner encoge de vuelta al avatar circular
# El efecto es automático — Flutter solo necesita el mismo tag en ambas pantallas
```

---

## Paso 5 — Transiciones de página personalizadas

```dart
// Añadir al menú en main.dart — no necesita archivo separado

// PageRouteBuilder permite customizar la transición de navegación
void _navegar(BuildContext context, Widget pantalla,
    {String tipo = 'fade'}) {
  Navigator.push(
    context,
    PageRouteBuilder(
      pageBuilder: (_, __, ___) => pantalla,
      transitionDuration: const Duration(milliseconds: 500),
      transitionsBuilder: (_, animation, __, child) {
        return switch (tipo) {
          'slide' => SlideTransition(
              position: Tween<Offset>(
                begin: const Offset(1, 0),
                end:   Offset.zero,
              ).animate(CurvedAnimation(
                  parent: animation, curve: Curves.easeOut)),
              child: child,
            ),
          'scale' => ScaleTransition(
              scale: CurvedAnimation(
                  parent: animation, curve: Curves.elasticOut),
              child: child,
            ),
          'fade' => FadeTransition(opacity: animation, child: child),
          _ => child,
        };
      },
    ),
  );
}

// Añadir estas entradas al ListView de PantallaMenu:
_ItemMenu(
  titulo:   '4a — Transición slide',
  subtitulo: 'Desliza desde la derecha',
  onTap: () => _navegar(context,
      const _PantallaDemo(titulo: 'Slide', color: Colors.indigo),
      tipo: 'slide'),
),
_ItemMenu(
  titulo:   '4b — Transición scale',
  subtitulo: 'Aparece escalando desde el centro',
  onTap: () => _navegar(context,
      const _PantallaDemo(titulo: 'Scale', color: Colors.deepOrange),
      tipo: 'scale'),
),
_ItemMenu(
  titulo:   '4c — Transición fade',
  subtitulo: 'Fundido de entrada',
  onTap: () => _navegar(context,
      const _PantallaDemo(titulo: 'Fade', color: Colors.teal),
      tipo: 'fade'),
),

// Widget de pantalla de demo reutilizable
class _PantallaDemo extends StatelessWidget {
  final String titulo;
  final Color  color;
  const _PantallaDemo({required this.titulo, required this.color});

  @override
  Widget build(BuildContext context) => Scaffold(
    backgroundColor: color,
    body: Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text(titulo,
              style: const TextStyle(
                  color: Colors.white, fontSize: 40,
                  fontWeight: FontWeight.bold)),
          const SizedBox(height: 24),
          FilledButton(
            style: FilledButton.styleFrom(
                backgroundColor: Colors.white,
                foregroundColor: color),
            onPressed: () => Navigator.pop(context),
            child: const Text('Volver'),
          ),
        ],
      ),
    ),
  );
}
```

```bash
flutter run
# → Toca "Transición slide" → pantalla entra deslizando desde la derecha
# → Toca "Transición scale" → pantalla aparece creciendo con efecto elástico
# → Toca "Transición fade" → pantalla aparece con fundido
```

---

## Paso 6 — Lottie

Lottie reproduce animaciones vectoriales complejas exportadas
desde After Effects como JSON. Son escalables y ligeras:

Descarga un archivo `.json` de prueba:
```bash
# Descargar un archivo Lottie de ejemplo (requiere curl)
curl -o assets/lottie/loading.json \
  "https://assets10.lottiefiles.com/packages/lf20_p8bfn5to.json"
```

O descarga manualmente desde https://lottiefiles.com y guárdalo
como `assets/lottie/loading.json`.

**Crea `lib/features/paso5_lottie.dart`:**

```dart
import 'package:flutter/material.dart';
import 'package:lottie/lottie.dart';

class PasoLottie extends StatefulWidget {
  const PasoLottie({super.key});

  @override
  State<PasoLottie> createState() => _PasoLottieState();
}

class _PasoLottieState extends State<PasoLottie>
    with SingleTickerProviderStateMixin {
  late final AnimationController _ctrl;
  double _velocidad = 1.0;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(vsync: this);
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Lottie')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [

            // Lottie desde asset local
            Lottie.asset(
              'assets/lottie/loading.json',
              controller:  _ctrl,
              width:       200,
              height:      200,
              // onLoaded se llama cuando el archivo está listo
              onLoaded:    (composition) {
                _ctrl
                  ..duration = composition.duration
                  ..repeat();           // reproducir en loop
              },
            ),

            const SizedBox(height: 24),

            // Lottie desde URL — sin controller, reproduce solo
            const Text('Desde URL (sin controller):',
                style: TextStyle(fontWeight: FontWeight.bold)),
            const SizedBox(height: 8),
            Lottie.network(
              'https://assets10.lottiefiles.com/packages/lf20_jR229r.json',
              width:    120,
              height:   120,
              repeat:   true,
              errorBuilder: (_, e, __) => const Icon(
                  Icons.broken_image, size: 60),
            ),

            const SizedBox(height: 24),

            // Control de velocidad
            Row(
              children: [
                Text('Velocidad: ${_velocidad.toStringAsFixed(1)}x',
                    style: const TextStyle(fontSize: 14)),
                Expanded(
                  child: Slider(
                    value:     _velocidad,
                    min:       0.25,
                    max:       3.0,
                    divisions: 11,
                    onChanged: (v) {
                      setState(() => _velocidad = v);
                      _ctrl.animateTo(
                        _ctrl.value,
                        duration: Duration(
                          milliseconds:
                              (_ctrl.duration!.inMilliseconds / v).toInt(),
                        ),
                      );
                      _ctrl.repeat();
                    },
                  ),
                ),
              ],
            ),

            // Controles
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                FilledButton(
                  onPressed: () => _ctrl.repeat(),
                  child: const Text('Loop'),
                ),
                const SizedBox(width: 8),
                OutlinedButton(
                  onPressed: () => _ctrl.stop(),
                  child: const Text('Detener'),
                ),
                const SizedBox(width: 8),
                FilledButton.tonal(
                  onPressed: () => _ctrl.forward(from: 0),
                  child: const Text('Una vez'),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

Añade al menú:

```dart
// lib/main.dart — añadir a PantallaMenu

import 'features/paso5_lottie.dart';

_ItemMenu(
  titulo:   '5 — Lottie',
  subtitulo: 'Animaciones vectoriales desde JSON',
  onTap: () => Navigator.push(context,
      MaterialPageRoute(builder: (_) => const PasoLottie())),
),
```

```bash
flutter run
# → Abre la pantalla de Lottie
# → La animación desde asset reproduce en loop
# → La animación desde URL carga y reproduce (requiere internet)
# → El slider cambia la velocidad de reproducción
# → Botones Loop / Detener / Una vez controlan el playback
```

---

## `lib/main.dart` final completo

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'features/paso1_implicity.dart';
import 'features/paso2_explicit.dart';
import 'features/paso3_hero.dart';
import 'features/paso5_lottie.dart';

void main() {
  runApp(const ProviderScope(child: AnimacionesApp()));
}

class AnimacionesApp extends StatelessWidget {
  const AnimacionesApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Animaciones',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const PantallaMenu(),
    );
  }
}

class PantallaMenu extends StatelessWidget {
  const PantallaMenu({super.key});

  void _navegar(BuildContext context, Widget pantalla,
      {String tipo = 'fade'}) {
    Navigator.push(
      context,
      PageRouteBuilder(
        pageBuilder: (_, __, ___) => pantalla,
        transitionDuration: const Duration(milliseconds: 500),
        transitionsBuilder: (_, animation, __, child) {
          return switch (tipo) {
            'slide' => SlideTransition(
                position: Tween<Offset>(
                  begin: const Offset(1, 0),
                  end:   Offset.zero,
                ).animate(CurvedAnimation(
                    parent: animation, curve: Curves.easeOut)),
                child: child,
              ),
            'scale' => ScaleTransition(
                scale: CurvedAnimation(
                    parent: animation, curve: Curves.elasticOut),
                child: child,
              ),
            _ => FadeTransition(opacity: animation, child: child),
          };
        },
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Animaciones Flutter'),
        backgroundColor:
            Theme.of(context).colorScheme.primaryContainer,
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          _ItemMenu(
            titulo:   '1 — Implícitas',
            subtitulo: 'AnimatedContainer, AnimatedOpacity...',
            onTap: () => Navigator.push(context,
                MaterialPageRoute(
                    builder: (_) =>
                        const PasoAnimacionesImplicitas())),
          ),
          _ItemMenu(
            titulo:   '2 — Explícitas',
            subtitulo: 'AnimationController + Tween',
            onTap: () => Navigator.push(context,
                MaterialPageRoute(
                    builder: (_) =>
                        const PasoAnimacionesExplicitas())),
          ),
          _ItemMenu(
            titulo:   '3 — Hero transitions',
            subtitulo: 'Elemento compartido entre pantallas',
            onTap: () => Navigator.push(context,
                MaterialPageRoute(
                    builder: (_) => const PasoHeroLista())),
          ),
          _ItemMenu(
            titulo:   '4a — Slide',
            subtitulo: 'PageRouteBuilder — deslizar',
            onTap: () => _navegar(context,
                const _PantallaDemo(
                    titulo: 'Slide', color: Colors.indigo),
                tipo: 'slide'),
          ),
          _ItemMenu(
            titulo:   '4b — Scale',
            subtitulo: 'PageRouteBuilder — escalar',
            onTap: () => _navegar(context,
                const _PantallaDemo(
                    titulo: 'Scale', color: Colors.deepOrange),
                tipo: 'scale'),
          ),
          _ItemMenu(
            titulo:   '4c — Fade',
            subtitulo: 'PageRouteBuilder — fundido',
            onTap: () => _navegar(context,
                const _PantallaDemo(
                    titulo: 'Fade', color: Colors.teal),
                tipo: 'fade'),
          ),
          _ItemMenu(
            titulo:   '5 — Lottie',
            subtitulo: 'Animaciones vectoriales JSON',
            onTap: () => Navigator.push(context,
                MaterialPageRoute(
                    builder: (_) => const PasoLottie())),
          ),
        ],
      ),
    );
  }
}

class _ItemMenu extends StatelessWidget {
  final String       titulo;
  final String       subtitulo;
  final VoidCallback onTap;

  const _ItemMenu({
    required this.titulo,
    required this.subtitulo,
    required this.onTap,
  });

  @override
  Widget build(BuildContext context) => Card(
    margin: const EdgeInsets.only(bottom: 8),
    child: ListTile(
      title:    Text(titulo,
          style: const TextStyle(fontWeight: FontWeight.bold)),
      subtitle: Text(subtitulo),
      trailing: const Icon(Icons.arrow_forward_ios, size: 16),
      onTap:    onTap,
    ),
  );
}

class _PantallaDemo extends StatelessWidget {
  final String titulo;
  final Color  color;
  const _PantallaDemo({required this.titulo, required this.color});

  @override
  Widget build(BuildContext context) => Scaffold(
    backgroundColor: color,
    body: Center(
      child: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          Text(titulo,
              style: const TextStyle(
                  color: Colors.white, fontSize: 40,
                  fontWeight: FontWeight.bold)),
          const SizedBox(height: 24),
          FilledButton(
            style: FilledButton.styleFrom(
                backgroundColor: Colors.white,
                foregroundColor: color),
            onPressed: () => Navigator.pop(context),
            child: const Text('Volver'),
          ),
        ],
      ),
    ),
  );
}
```

---

## Estado final del proyecto

```
lib/
├── main.dart                         ← menú con todos los pasos
└── features/
    ├── paso1_implicity.dart          ← paso 2
    ├── paso2_explicit.dart           ← paso 3
    ├── paso3_hero.dart               ← paso 4
    └── paso5_lottie.dart             ← paso 6
    (paso4 vive en main.dart)
```

---

## Ejercicios propuestos

1. **Indicador de carga animado** — Crea un widget `LoadingDots` con
   tres puntos que aparecen y desaparecen en secuencia usando
   `AnimationController` con `Interval` para escalonar cada punto.
   Úsalo como placeholder mientras carga la lista de servidores de
   la app del catálogo (página 12).

2. **Card con flip 3D** — Usa `AnimatedBuilder` con `Transform` y
   `Matrix4.rotationY(angulo)` para crear una tarjeta que gira 180°
   al tocarla, mostrando información diferente en cada cara.
   Recuerda invertir la escala en la segunda cara para que no se vea espejada.

3. **Staggered list** — Al entrar en `PasoAnimacionesImplicitas`, los
   elementos de la lista deben aparecer uno por uno con un retardo
   progresivo (0ms, 100ms, 200ms...). Usa `FutureBuilder` o
   `Future.delayed` con `AnimatedOpacity` en cada elemento.

4. **Hero con FlightShuttleBuilder** — Personaliza la animación Hero
   de la pantalla de servidores. Usando `flightShuttleBuilder`, muestra
   un widget de transición diferente durante el vuelo (ej: un círculo
   que se expande). El widget de origen y destino permanecen iguales.

---

## Resumen de la página 18

- Las **animaciones implícitas** (`AnimatedContainer`, `AnimatedOpacity`, etc.) son la primera opción — solo cambia el valor y Flutter interpola automáticamente. No necesitan `AnimationController`.
- `AnimationController` necesita un `TickerProvider` — se obtiene con el mixin `SingleTickerProviderStateMixin`. Siempre llamar `_ctrl.dispose()` en `dispose()`.
- `Tween<T>.animate(controller)` crea la animación tipada. `CurvedAnimation` aplica una curva de interpolación. `Interval(inicio, fin)` segmenta cuándo ocurre cada animación dentro del total.
- `AnimatedBuilder(animation, builder)` reconstruye solo el subtree afectado en cada frame. El parámetro `child` evita reconstruir widgets estáticos dentro del builder.
- `Hero` requiere el mismo `tag` en origen y destino. El tag debe ser único en toda la pantalla. No anida Heroes.
- `PageRouteBuilder` con `transitionsBuilder` personaliza la animación de navegación. `SlideTransition`, `FadeTransition` y `ScaleTransition` son los más usados.
- `Lottie.asset` carga desde `pubspec.yaml` assets. `Lottie.network` carga desde URL. `onLoaded` proporciona la duración real de la animación para configurar el controller.
- `Curves.bounceOut`, `Curves.elasticOut` y `Curves.easeInOut` son las curvas más expresivas. `Curves.linear` es la opción por defecto y la menos atractiva visualmente.

---

> **Siguiente página →** Página 19: Clean Architecture con Riverpod —
> capas domain/data/ui, repositorios, use cases y proyecto integrador.