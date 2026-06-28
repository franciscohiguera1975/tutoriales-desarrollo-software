# Tutorial Flutter — Página 18
## Módulo 6 · UI Avanzada
### Animaciones: implícitas, explícitas, Hero y Lottie

---

Flutter trata las animaciones como ciudadanos de primera clase: desde un simple cambio de color
hasta transiciones entre pantallas con elemento compartido, todo sigue el mismo modelo basado en
`Animation<T>` y `AnimationController`. En esta página construirás cinco demos progresivos que
cubren las cuatro familias de animación que usarás en producción.

---

## ¿Qué aprenderás?

```
Tipo                        | Clase / Widget principal          | Cuando usar
----------------------------|-----------------------------------|-----------------------------------------
Animacion implicita simple  | AnimatedContainer                 | Cambia width/height/color/radius con dur.
Animacion implicita opac.   | AnimatedOpacity                   | Fade in/out de cualquier widget
Transicion entre widgets    | AnimatedSwitcher                  | Swap de dos widgets distintos (con key)
Animacion explicita         | AnimationController + Tween       | Control total: play, pause, reverse
Elemento compartido         | Hero                              | Transicion visual entre dos pantallas
Transicion de pagina        | PageRouteBuilder                  | Slide/fade/scale al navegar entre rutas
Animaciones JSON            | Lottie.asset / Lottie.network     | Ilustraciones animadas de LottieFiles
Multi-controlador           | TickerProviderStateMixin          | Cuando se usan 2+ AnimationControllers
Animacion en loop           | _ctrl.repeat() / Lottie repeat    | Indicadores de carga o fondos animados
Animacion por gesto         | _ctrl.value = gesture.delta       | Drag-to-reveal, pull-to-refresh custom
```

---

## Patrones clave

### Patron 1 — Animaciones implícitas: `AnimatedContainer` y `AnimatedOpacity`

```dart
// AnimatedContainer cambia propiedades suavemente al cambiar el estado
AnimatedContainer(
  duration: const Duration(milliseconds: 400),  // obligatorio — sin esto Flutter lanza error
  curve: Curves.easeInOut,
  width: _expandido ? 200.0 : 80.0,
  height: _expandido ? 200.0 : 80.0,
  decoration: BoxDecoration(
    color: _expandido ? Colors.blue : Colors.red,
    borderRadius: BorderRadius.circular(_expandido ? 32 : 8),
  ),
)

// AnimatedOpacity — fade in / fade out de cualquier widget hijo
AnimatedOpacity(
  duration: const Duration(milliseconds: 300),
  opacity: _visible ? 1.0 : 0.0,
  child: const Text('Aparezco y desaparezco'),
)
```

> **Mini-ejercicio (5 min):** Agrega un `FloatingActionButton` que alterne `_expandido`.
> Cambia la curva a `Curves.bounceOut` y verifica el efecto de rebote al expandirse.
> Luego agrega un `AnimatedPadding` que envuelva un icono con padding `8.0` en reposo y `32.0`
> al expandirse. Observa cómo el padding se anima sin escribir una sola linea de lógica
> de interpolacion.

---

### Patron 2 — `AnimatedSwitcher`: transición entre widgets

```dart
// ScaleTransition personalizado al cambiar el valor del contador
AnimatedSwitcher(
  duration: const Duration(milliseconds: 300),
  transitionBuilder: (child, animation) => ScaleTransition(
    scale: animation,
    child: child,
  ),
  // KEY es obligatorio para que Flutter detecte que el widget cambio
  child: Text(
    '$_contador',
    key: ValueKey<int>(_contador),
    style: const TextStyle(fontSize: 48),
  ),
)

// FadeTransition (comportamiento por defecto si se omite transitionBuilder)
AnimatedSwitcher(
  duration: const Duration(milliseconds: 500),
  child: _mostrarA
      ? const WidgetA(key: ValueKey('a'))
      : const WidgetB(key: ValueKey('b')),
)
```

> **Mini-ejercicio (5 min):** Reemplaza el `Text` del contador por un `AnimatedSwitcher` con
> `ScaleTransition`. Observa que sin `ValueKey` la animacion no dispara aunque cambie el numero.
> Agrega `ValueKey<String>('A')` y `ValueKey<String>('B')` a los hijos de un segundo
> `AnimatedSwitcher` que alterne entre dos `Container` de colores distintos; verifica que la
> animacion se activa correctamente.

---

### Patron 3 — `AnimationController` + `Tween` + `AnimatedBuilder`: animaciones explícitas

```dart
// El mixin SingleTickerProviderStateMixin provee el vsync necesario
class _MiWidgetState extends State<MiWidget> with SingleTickerProviderStateMixin {
  late AnimationController _ctrl;
  late Animation<double> _rotacion;
  late Animation<Color?> _color;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,                              // referencia al State actual
      duration: const Duration(seconds: 2),
    );
    _rotacion = Tween<double>(begin: 0, end: 2 * 3.14159).animate(
      CurvedAnimation(parent: _ctrl, curve: Curves.linear),
    );
    _color = ColorTween(begin: Colors.blue, end: Colors.red).animate(_ctrl);

    // Escuchar cuando termina la animacion
    _ctrl.addStatusListener((status) {
      if (status == AnimationStatus.completed) debugPrint('Termino!');
    });
  }

  @override
  void dispose() {
    _ctrl.dispose();    // siempre liberar recursos
    super.dispose();
  }
}

// En build() — AnimatedBuilder reconstruye solo el subarbol afectado
AnimatedBuilder(
  animation: _ctrl,
  builder: (context, child) => Transform.rotate(
    angle: _rotacion.value,
    child: Container(color: _color.value, width: 80, height: 80),
  ),
)

// Metodos de control
_ctrl.forward();              // reproducir hacia adelante (0.0 → 1.0)
_ctrl.reverse();              // reproducir hacia atras (1.0 → 0.0)
_ctrl.repeat(reverse: true);  // ping-pong continuo
_ctrl.stop();                 // detener en el frame actual
```

> **Mini-ejercicio (6 min):** Cambia la curva de `_rotacion` a `Curves.elasticOut`.
> Agrega `_ctrl.repeat(reverse: true)` en `initState` y observa el efecto ping-pong.
> Luego agrega `_ctrl.addStatusListener` que imprima `'Termino'` en consola cuando la
> animacion complete. Usa el metodo `_ctrl.stop()` en un boton adicional para pausarla.

---

### Patron 4 — `Hero`: transición con elemento compartido entre pantallas

```dart
// Pantalla ORIGEN — el Hero envuelve la imagen pequena
Hero(
  tag: 'imagen_${producto.id}',   // tag unico por elemento en pantalla
  child: Image.network(producto.imagenUrl, width: 80),
)

// Pantalla DESTINO — mismo tag, imagen mas grande
Hero(
  tag: 'imagen_${producto.id}',
  child: Image.network(producto.imagenUrl, width: double.infinity),
)

// La animacion Hero se activa automaticamente al hacer push/pop
// No se necesita codigo adicional — Flutter la detecta por el tag coincidente
Navigator.push(
  context,
  MaterialPageRoute(builder: (_) => PantallaDetalle(producto: producto)),
);

// IMPORTANTE: el mismo tag dos veces en la misma pantalla provoca assertion error
```

> **Mini-ejercicio (5 min):** Crea una lista de 3 cajas de colores distintos, cada una con
> un `Hero` de tag `'caja_$i'`. Al tocar una caja navega a una pantalla de detalle que
> muestra la misma caja ampliada a 200x200. Observa la animacion de expansion.
> Luego agrega un segundo `Hero` con tag `'nombre_$i'` para un `Text` debajo de cada caja
> y verifica que el nombre tambien viaja a la pantalla de detalle.

---

### Patron 5 — `PageRouteBuilder`: transición de página personalizada

```dart
Navigator.push(
  context,
  PageRouteBuilder(
    transitionDuration: const Duration(milliseconds: 500),
    reverseTransitionDuration: const Duration(milliseconds: 300),  // duracion al volver
    pageBuilder: (context, animation, secondaryAnimation) =>
        const PantallaDestino(),
    transitionsBuilder: (context, animation, secondaryAnimation, child) {
      // Slide desde abajo
      final tween = Tween(begin: const Offset(0, 1), end: Offset.zero)
          .chain(CurveTween(curve: Curves.easeOut));
      return SlideTransition(
        position: animation.drive(tween),
        child: child,
      );
    },
  ),
);

// Alternativa: FadeTransition
FadeTransition(opacity: animation, child: child)

// Alternativa: ScaleTransition desde el centro
ScaleTransition(scale: animation, child: child)

// Combinacion slide + fade
return SlideTransition(
  position: animation.drive(tween),
  child: FadeTransition(opacity: animation, child: child),
);
```

> **Mini-ejercicio (5 min):** Cambia `Offset(0, 1)` a `Offset(1, 0)` y observa el slide
> desde la derecha. Luego combina `SlideTransition` con `FadeTransition` envolviendo el hijo:
> el resultado debe deslizarse Y fundirse al mismo tiempo. Ajusta `transitionDuration` a 800ms
> para ver el efecto con mas detalle.

---

### Patron 6 — `Lottie`: animaciones JSON desde LottieFiles

```dart
import 'package:lottie/lottie.dart';

// Desde asset (declarado en pubspec.yaml bajo flutter.assets)
Lottie.asset('assets/lottie/loading.json')

// Desde URL — no requiere declaracion en pubspec
Lottie.network('https://assets5.lottiefiles.com/packages/lf20_usmfx6bp.json')

// Con AnimationController para control manual
LottieBuilder.asset(
  'assets/lottie/success.json',
  controller: _ctrl,
  onLoaded: (composition) {
    // Establecer la duracion real del JSON antes de reproducir
    _ctrl.duration = composition.duration;
    _ctrl.forward();
  },
)

// Opciones comunes
Lottie.asset(
  'assets/lottie/loading.json',
  width: 200,
  height: 200,
  fit: BoxFit.contain,
  repeat: true,    // loop automatico
  reverse: false,
)
```

> **Mini-ejercicio (6 min):** Usa `Lottie.network` con una URL de LottieFiles para mostrar
> una animacion sin necesidad de assets locales. Agrega `repeat: false` y un boton que llame
> `_ctrl.forward()` para reproducir desde el inicio al pulsar. Observa la diferencia entre
> `Lottie.asset` (requiere pubspec) y `Lottie.network` (no requiere).

---

## Crea el proyecto

```bash
flutter create modulo18_animaciones
cd modulo18_animaciones
mkdir -p assets/lottie
# Descarga un .json de https://lottiefiles.com/featured y colócalo en assets/lottie/
flutter pub get
```

### `pubspec.yaml`

```yaml
name: modulo18_animaciones
description: "Demo de animaciones en Flutter — Modulo 18"
publish_to: none
version: 1.0.0+1

environment:
  sdk: ^3.7.0

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^3.3.0
  lottie: ^3.1.3

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^5.0.0

flutter:
  uses-material-design: true
  assets:
    - assets/lottie/
```

### Configuración de assets

```
modulo18_animaciones/
├── assets/
│   └── lottie/
│       ├── loading.json    ← descargar de lottiefiles.com (ej: "loading spinner")
│       └── success.json    ← descargar de lottiefiles.com (ej: "checkmark success")
└── pubspec.yaml
```

Para descargar assets Lottie: ir a `https://lottiefiles.com/featured`, elegir una animacion,
pulsar "Download" → "Lottie JSON" y colocar el archivo en `assets/lottie/`.

---

## Selector de pasos

Cada paso es un `main.dart` completo e independiente. Para probar un paso especifico,
cambia el valor de la constante en la parte superior del archivo:

```dart
// Cambia este valor (1–5) para probar cada demo
const int paso = 1;
```

Esto permite ejecutar `flutter run` y ver cada concepto de forma aislada antes de
ensamblar el proyecto final.

---

## Paso 1 — Animaciones implícitas

```dart
// lib/main.dart
import 'package:flutter/material.dart';

// Cambia este valor (1–5) para probar cada demo
const int paso = 1;

void main() => runApp(const MaterialApp(
      debugShowCheckedModeBanner: false,
      home: PasoUno(),
    ));

class PasoUno extends StatefulWidget {
  const PasoUno({super.key});
  @override
  State<PasoUno> createState() => _PasoUnoState();
}

class _PasoUnoState extends State<PasoUno> {
  bool _expandido = false;
  bool _visible = true;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 1 — Animaciones implicitas')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // AnimatedContainer: cambia tamaño, color y radio suavemente
            GestureDetector(
              onTap: () => setState(() => _expandido = !_expandido),
              child: AnimatedContainer(
                duration: const Duration(milliseconds: 400),
                curve: Curves.easeInOut,
                width: _expandido ? 200.0 : 80.0,
                height: _expandido ? 200.0 : 80.0,
                decoration: BoxDecoration(
                  color: _expandido ? Colors.blue : Colors.red,
                  borderRadius: BorderRadius.circular(_expandido ? 32 : 8),
                ),
                child: const Icon(Icons.star, color: Colors.white),
              ),
            ),
            const SizedBox(height: 16),
            const Text(
              'Toca la caja para expandir',
              style: TextStyle(color: Colors.grey),
            ),

            const SizedBox(height: 40),

            // AnimatedOpacity: fade in/out
            AnimatedOpacity(
              duration: const Duration(milliseconds: 400),
              curve: Curves.elasticOut,
              opacity: _visible ? 1.0 : 0.0,
              child: Container(
                padding: const EdgeInsets.all(16),
                decoration: BoxDecoration(
                  color: Colors.green.shade100,
                  borderRadius: BorderRadius.circular(12),
                ),
                child: const Text(
                  'Aparezco y desaparezco',
                  style: TextStyle(fontSize: 18),
                ),
              ),
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: () => setState(() => _visible = !_visible),
              child: Text(_visible ? 'Ocultar' : 'Mostrar'),
            ),

            const SizedBox(height: 40),

            // AnimatedPadding: padding animado
            AnimatedPadding(
              duration: const Duration(milliseconds: 400),
              curve: Curves.easeInOut,
              padding: EdgeInsets.all(_expandido ? 32.0 : 8.0),
              child: Container(
                color: Colors.orange.shade200,
                child: const Icon(Icons.favorite, size: 32),
              ),
            ),
            const Text(
              'AnimatedPadding — toca la caja de arriba',
              style: TextStyle(color: Colors.grey, fontSize: 12),
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => setState(() => _expandido = !_expandido),
        child: Icon(_expandido ? Icons.compress : Icons.expand),
      ),
    );
  }
}
```

> **Prueba esto:** Toca la caja roja — debe crecer, volverse azul y redondear sus esquinas
> suavemente con efecto `easeInOut`. Toca el boton "Ocultar" — el texto verde debe fundirse.
> Observa como `AnimatedPadding` reacciona al mismo estado `_expandido` sin codigo extra.

Salida esperada: la caja pasa de 80x80 (roja, esquinas rectas) a 200x200 (azul, esquinas muy
redondeadas) en 400ms. El texto verde hace fade out en 400ms con curva `elasticOut`.

---

## Paso 2 — `AnimatedSwitcher` con transiciones

```dart
// lib/main.dart
import 'package:flutter/material.dart';

void main() => runApp(const MaterialApp(
      debugShowCheckedModeBanner: false,
      home: PasoDos(),
    ));

class PasoDos extends StatefulWidget {
  const PasoDos({super.key});
  @override
  State<PasoDos> createState() => _PasosDosState();
}

class _PasosDosState extends State<PasoDos> {
  int _contador = 0;
  bool _mostrarA = true;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 2 — AnimatedSwitcher')),
      body: GestureDetector(
        // Toca en cualquier lugar para incrementar el contador
        onTap: () => setState(() => _contador++),
        behavior: HitTestBehavior.opaque,
        child: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Text(
                'Toca la pantalla para contar',
                style: TextStyle(color: Colors.grey),
              ),
              const SizedBox(height: 24),

              // ScaleTransition personalizado — KEY obligatoria para detectar cambio
              AnimatedSwitcher(
                duration: const Duration(milliseconds: 300),
                transitionBuilder: (child, animation) => ScaleTransition(
                  scale: animation,
                  child: child,
                ),
                child: Text(
                  '$_contador',
                  // Sin ValueKey la animacion NO dispara aunque cambie el numero
                  key: ValueKey<int>(_contador),
                  style: const TextStyle(
                    fontSize: 72,
                    fontWeight: FontWeight.bold,
                    color: Colors.deepPurple,
                  ),
                ),
              ),

              const SizedBox(height: 48),
              const Divider(),
              const SizedBox(height: 24),

              const Text(
                'Alternancia entre dos widgets',
                style: TextStyle(fontWeight: FontWeight.bold),
              ),
              const SizedBox(height: 16),

              // FadeTransition (default) entre dos Containers de colores
              AnimatedSwitcher(
                duration: const Duration(milliseconds: 500),
                child: _mostrarA
                    ? Container(
                        key: const ValueKey<String>('A'),
                        width: 120,
                        height: 120,
                        color: Colors.teal,
                        child: const Center(
                          child: Text(
                            'Widget A',
                            style: TextStyle(color: Colors.white, fontSize: 18),
                          ),
                        ),
                      )
                    : Container(
                        key: const ValueKey<String>('B'),
                        width: 120,
                        height: 120,
                        color: Colors.orange,
                        child: const Center(
                          child: Text(
                            'Widget B',
                            style: TextStyle(color: Colors.white, fontSize: 18),
                          ),
                        ),
                      ),
              ),
              const SizedBox(height: 16),
              ElevatedButton(
                onPressed: () => setState(() => _mostrarA = !_mostrarA),
                child: Text(_mostrarA ? 'Mostrar Widget B' : 'Mostrar Widget A'),
              ),
              const SizedBox(height: 16),
              const Text(
                'Sin ValueKey la animacion no se activa\n'
                'aunque el widget cambie de tipo',
                textAlign: TextAlign.center,
                style: TextStyle(color: Colors.grey, fontSize: 12),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

> **Prueba esto:** Toca la pantalla varias veces rapidamente — cada numero debe aparecer con
> un efecto de escala desde 0 hasta 1. Pulsa "Mostrar Widget B" — el contenedor teal debe
> fundirse y aparecer el naranja.

Salida esperada: el contador salta con animacion de escala en cada toque. El `AnimatedSwitcher`
inferior hace crossfade entre los dos contenedores. Si remueves las `ValueKey`, el contador
cambia de numero pero sin animacion — ese es el comportamiento erroneo a evitar.

---

## Paso 3 — Animación explícita con `AnimationController`

```dart
// lib/main.dart
import 'dart:math';
import 'package:flutter/material.dart';

void main() => runApp(const MaterialApp(
      debugShowCheckedModeBanner: false,
      home: PasoTres(),
    ));

class PasoTres extends StatefulWidget {
  const PasoTres({super.key});
  @override
  State<PasoTres> createState() => _PasoTresState();
}

// SingleTickerProviderStateMixin provee el vsync para UN AnimationController
class _PasoTresState extends State<PasoTres> with SingleTickerProviderStateMixin {
  late AnimationController _ctrl;
  late Animation<double> _rotacion;
  late Animation<Color?> _color;
  String _estadoTexto = 'Detenido';

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 2),
    );

    // Tween de rotacion: 0 a 2π (una vuelta completa)
    _rotacion = Tween<double>(begin: 0, end: 2 * pi).animate(
      CurvedAnimation(parent: _ctrl, curve: Curves.linear),
    );

    // ColorTween: azul a rojo
    _color = ColorTween(begin: Colors.blue, end: Colors.red).animate(_ctrl);

    // Escuchar cambios de estado para actualizar el texto
    _ctrl.addStatusListener((status) {
      setState(() {
        switch (status) {
          case AnimationStatus.forward:
            _estadoTexto = 'Reproduciendo...';
          case AnimationStatus.reverse:
            _estadoTexto = 'Regresando...';
          case AnimationStatus.completed:
            _estadoTexto = 'Completado';
          case AnimationStatus.dismissed:
            _estadoTexto = 'Detenido';
        }
      });
    });
  }

  @override
  void dispose() {
    _ctrl.dispose();    // liberar el ticker — obligatorio
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 3 — Animacion Explicita')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // AnimatedBuilder reconstruye solo este subarbol en cada frame
            AnimatedBuilder(
              animation: _ctrl,
              builder: (context, child) {
                return Column(
                  children: [
                    Transform.rotate(
                      angle: _rotacion.value,
                      child: Container(
                        width: 120,
                        height: 120,
                        decoration: BoxDecoration(
                          color: _color.value,
                          borderRadius: BorderRadius.circular(16),
                        ),
                        child: const Icon(
                          Icons.settings,
                          size: 64,
                          color: Colors.white,
                        ),
                      ),
                    ),
                    const SizedBox(height: 16),
                    // Progreso numerico del controlador (0.0 a 1.0)
                    Text(
                      '${(_ctrl.value * 100).round()}%',
                      style: const TextStyle(fontSize: 32, fontWeight: FontWeight.bold),
                    ),
                  ],
                );
              },
            ),

            const SizedBox(height: 8),
            Text(
              _estadoTexto,
              style: const TextStyle(color: Colors.grey),
            ),
            const SizedBox(height: 32),

            // Botones de control
            Wrap(
              spacing: 8,
              runSpacing: 8,
              alignment: WrapAlignment.center,
              children: [
                ElevatedButton(
                  onPressed: () => _ctrl.forward(),
                  child: const Text('Play Forward'),
                ),
                ElevatedButton(
                  onPressed: () => _ctrl.reverse(),
                  child: const Text('Reverse'),
                ),
                ElevatedButton(
                  onPressed: () => _ctrl.repeat(reverse: true),
                  child: const Text('Ping-Pong'),
                ),
                ElevatedButton(
                  onPressed: () => _ctrl.stop(),
                  style: ElevatedButton.styleFrom(backgroundColor: Colors.red),
                  child: const Text(
                    'Stop',
                    style: TextStyle(color: Colors.white),
                  ),
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

> **Prueba esto:** Pulsa "Play Forward" — el engranaje debe rotar y cambiar de azul a rojo
> mostrando el porcentaje. Pulsa "Ping-Pong" para animacion continua de ida y vuelta.
> Pulsa "Stop" para congelar en el frame actual.

Salida esperada: el icono `Icons.settings` rota 360° en 2 segundos cambiando de azul a rojo.
El texto de porcentaje va de `0%` a `100%`. El `addStatusListener` actualiza `_estadoTexto`
en cada cambio de estado del controlador.

---

## Paso 4 — Hero: transición entre pantallas

```dart
// lib/main.dart
import 'package:flutter/material.dart';

void main() => runApp(const MaterialApp(
      debugShowCheckedModeBanner: false,
      home: PasosCuatro(),
    ));

// Datos de las tarjetas
const List<Map<String, dynamic>> _tarjetas = [
  {'id': 0, 'color': Colors.red, 'nombre': 'Roja'},
  {'id': 1, 'color': Colors.green, 'nombre': 'Verde'},
  {'id': 2, 'color': Colors.blue, 'nombre': 'Azul'},
  {'id': 3, 'color': Colors.orange, 'nombre': 'Naranja'},
  {'id': 4, 'color': Colors.purple, 'nombre': 'Purpura'},
  {'id': 5, 'color': Colors.teal, 'nombre': 'Teal'},
];

class PasosCuatro extends StatelessWidget {
  const PasosCuatro({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 4 — Hero Transitions')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: GridView.builder(
          gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: 2,
            crossAxisSpacing: 16,
            mainAxisSpacing: 16,
          ),
          itemCount: _tarjetas.length,
          itemBuilder: (context, i) {
            final tarjeta = _tarjetas[i];
            return GestureDetector(
              onTap: () {
                Navigator.push(
                  context,
                  MaterialPageRoute(
                    builder: (_) => PantallaDetalle(tarjeta: tarjeta),
                  ),
                );
              },
              child: Hero(
                // tag unico por elemento — mismo tag en origen y destino
                tag: 'card_${tarjeta['id']}',
                child: Container(
                  decoration: BoxDecoration(
                    color: tarjeta['color'] as Color,
                    borderRadius: BorderRadius.circular(16),
                  ),
                  child: Column(
                    mainAxisAlignment: MainAxisAlignment.center,
                    children: [
                      const Icon(Icons.palette, color: Colors.white, size: 32),
                      const SizedBox(height: 8),
                      // El Hero del nombre viaja independientemente
                      Hero(
                        tag: 'nombre_${tarjeta['id']}',
                        child: Material(
                          color: Colors.transparent,
                          child: Text(
                            tarjeta['nombre'] as String,
                            style: const TextStyle(
                              color: Colors.white,
                              fontSize: 16,
                              fontWeight: FontWeight.bold,
                            ),
                          ),
                        ),
                      ),
                    ],
                  ),
                ),
              ),
            );
          },
        ),
      ),
    );
  }
}

class PantallaDetalle extends StatelessWidget {
  final Map<String, dynamic> tarjeta;
  const PantallaDetalle({super.key, required this.tarjeta});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Detalle — ${tarjeta['nombre']}'),
        backgroundColor: tarjeta['color'] as Color,
        foregroundColor: Colors.white,
      ),
      body: Column(
        children: [
          // Hero con el mismo tag expande la tarjeta a pantalla completa
          Hero(
            tag: 'card_${tarjeta['id']}',
            child: Container(
              width: double.infinity,
              height: 300,
              color: tarjeta['color'] as Color,
              child: const Icon(Icons.palette, color: Colors.white, size: 80),
            ),
          ),
          const SizedBox(height: 24),
          Hero(
            tag: 'nombre_${tarjeta['id']}',
            child: Material(
              child: Text(
                tarjeta['nombre'] as String,
                style: const TextStyle(fontSize: 32, fontWeight: FontWeight.bold),
              ),
            ),
          ),
          const SizedBox(height: 16),
          const Text(
            'Pulsa Atras para volver con animacion Hero inversa',
            style: TextStyle(color: Colors.grey),
          ),
        ],
      ),
    );
  }
}
```

> **Prueba esto:** Toca cualquier tarjeta de la grilla — la caja de color debe "volar" desde
> su posicion hasta ocupar la parte superior de la pantalla de detalle. Al pulsar Atras, el
> elemento regresa a su lugar original. Observa que el nombre tambien viaja con su propio Hero.

Salida esperada: al tocar una tarjeta, la caja de color se expande fluidamente desde 160px en
la grilla hasta 300px de alto en la pantalla de detalle. No se necesita codigo de animacion
adicional — Flutter calcula la interpolacion automaticamente por el tag coincidente.

---

## Paso 5 — App completa con 4 pestañas

```dart
// lib/main.dart
import 'dart:math';
import 'package:flutter/material.dart';
import 'package:lottie/lottie.dart';

void main() => runApp(const MaterialApp(
      debugShowCheckedModeBanner: false,
      home: PasosCinco(),
    ));

// -----------------------------------------------------------------------
// APP PRINCIPAL CON NavigationBar
// -----------------------------------------------------------------------
class PasosCinco extends StatefulWidget {
  const PasosCinco({super.key});
  @override
  State<PasosCinco> createState() => _PasosCincoState();
}

class _PasosCincoState extends State<PasosCinco> {
  int _paginaActual = 0;

  static const List<Widget> _paginas = [
    TabImplicitas(),
    TabSwitcher(),
    TabExplicita(),
    TabHero(),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: _paginas[_paginaActual],
      bottomNavigationBar: NavigationBar(
        selectedIndex: _paginaActual,
        onDestinationSelected: (i) => setState(() => _paginaActual = i),
        destinations: const [
          NavigationDestination(icon: Icon(Icons.auto_awesome), label: 'Implicitas'),
          NavigationDestination(icon: Icon(Icons.swap_horiz), label: 'Switcher'),
          NavigationDestination(icon: Icon(Icons.settings), label: 'Explicita'),
          NavigationDestination(icon: Icon(Icons.grid_view), label: 'Hero'),
        ],
      ),
    );
  }
}

// -----------------------------------------------------------------------
// TAB 0 — Animaciones implicitas
// -----------------------------------------------------------------------
class TabImplicitas extends StatefulWidget {
  const TabImplicitas({super.key});
  @override
  State<TabImplicitas> createState() => _TabImplicitasState();
}

class _TabImplicitasState extends State<TabImplicitas> {
  bool _grande = false;
  bool _visible = true;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Implicitas')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            AnimatedContainer(
              duration: const Duration(milliseconds: 500),
              curve: Curves.easeInOut,
              width: _grande ? 200 : 80,
              height: _grande ? 200 : 80,
              decoration: BoxDecoration(
                color: _grande ? Colors.deepPurple : Colors.amber,
                borderRadius: BorderRadius.circular(_grande ? 40 : 8),
              ),
              child: const Icon(Icons.star, color: Colors.white, size: 32),
            ),
            const SizedBox(height: 24),
            AnimatedOpacity(
              duration: const Duration(milliseconds: 400),
              opacity: _visible ? 1.0 : 0.0,
              child: const Chip(label: Text('Soy visible')),
            ),
            const SizedBox(height: 24),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                ElevatedButton(
                  onPressed: () => setState(() => _grande = !_grande),
                  child: const Text('Toggle tamaño'),
                ),
                const SizedBox(width: 8),
                ElevatedButton(
                  onPressed: () => setState(() => _visible = !_visible),
                  child: const Text('Toggle opacidad'),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}

// -----------------------------------------------------------------------
// TAB 1 — AnimatedSwitcher contador
// -----------------------------------------------------------------------
class TabSwitcher extends StatefulWidget {
  const TabSwitcher({super.key});
  @override
  State<TabSwitcher> createState() => _TabSwitcherState();
}

class _TabSwitcherState extends State<TabSwitcher> {
  int _n = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('AnimatedSwitcher')),
      body: GestureDetector(
        onTap: () => setState(() => _n++),
        behavior: HitTestBehavior.opaque,
        child: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              AnimatedSwitcher(
                duration: const Duration(milliseconds: 300),
                transitionBuilder: (child, anim) =>
                    ScaleTransition(scale: anim, child: child),
                child: Text(
                  '$_n',
                  key: ValueKey<int>(_n),
                  style: const TextStyle(fontSize: 80, fontWeight: FontWeight.bold),
                ),
              ),
              const SizedBox(height: 16),
              const Text('Toca en cualquier lugar', style: TextStyle(color: Colors.grey)),
            ],
          ),
        ),
      ),
    );
  }
}

// -----------------------------------------------------------------------
// TAB 2 — Animacion explicita con engranaje
// -----------------------------------------------------------------------
class TabExplicita extends StatefulWidget {
  const TabExplicita({super.key});
  @override
  State<TabExplicita> createState() => _TabExplicitaState();
}

class _TabExplicitaState extends State<TabExplicita>
    with SingleTickerProviderStateMixin {
  late AnimationController _ctrl;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 3),
    );
  }

  @override
  void dispose() {
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Explicita — Engranaje')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            AnimatedBuilder(
              animation: _ctrl,
              builder: (_, __) => Transform.rotate(
                angle: _ctrl.value * 2 * pi,
                child: Icon(Icons.settings, size: 120, color: Colors.indigo.shade400),
              ),
            ),
            const SizedBox(height: 32),
            Wrap(
              spacing: 8,
              runSpacing: 8,
              alignment: WrapAlignment.center,
              children: [
                ElevatedButton(
                  onPressed: () => _ctrl.forward(from: 0),
                  child: const Text('Play'),
                ),
                ElevatedButton(
                  onPressed: () => _ctrl.repeat(reverse: true),
                  child: const Text('Ping-Pong'),
                ),
                ElevatedButton(
                  onPressed: () => _ctrl.stop(),
                  style: ElevatedButton.styleFrom(backgroundColor: Colors.red),
                  child: const Text('Stop', style: TextStyle(color: Colors.white)),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}

// -----------------------------------------------------------------------
// TAB 3 — Hero grid + FAB con Lottie
// -----------------------------------------------------------------------
const List<Color> _colores = [
  Colors.red, Colors.green, Colors.blue,
  Colors.orange, Colors.purple, Colors.teal,
];

class TabHero extends StatelessWidget {
  const TabHero({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Hero Grid')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: GridView.builder(
          gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
            crossAxisCount: 3,
            crossAxisSpacing: 12,
            mainAxisSpacing: 12,
          ),
          itemCount: _colores.length,
          itemBuilder: (context, i) => GestureDetector(
            onTap: () => Navigator.push(
              context,
              MaterialPageRoute(
                builder: (_) => DetalleHero(indice: i, color: _colores[i]),
              ),
            ),
            child: Hero(
              tag: 'color_$i',
              child: Container(
                decoration: BoxDecoration(
                  color: _colores[i],
                  borderRadius: BorderRadius.circular(12),
                ),
              ),
            ),
          ),
        ),
      ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: () => Navigator.push(
          context,
          PageRouteBuilder(
            transitionDuration: const Duration(milliseconds: 600),
            pageBuilder: (_, __, ___) => const PantallaLottie(),
            transitionsBuilder: (_, anim, __, child) =>
                FadeTransition(opacity: anim, child: child),
          ),
        ),
        icon: const Icon(Icons.celebration),
        label: const Text('Lottie'),
      ),
    );
  }
}

class DetalleHero extends StatelessWidget {
  final int indice;
  final Color color;
  const DetalleHero({super.key, required this.indice, required this.color});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        backgroundColor: color,
        foregroundColor: Colors.white,
        title: Text('Color $indice'),
      ),
      body: Column(
        children: [
          Hero(
            tag: 'color_$indice',
            child: Container(width: double.infinity, height: 280, color: color),
          ),
          const SizedBox(height: 24),
          Text('Tarjeta $indice',
              style: const TextStyle(fontSize: 28, fontWeight: FontWeight.bold)),
          const SizedBox(height: 8),
          const Text('Pulsa Atras para ver la animacion Hero inversa',
              style: TextStyle(color: Colors.grey)),
        ],
      ),
    );
  }
}

// -----------------------------------------------------------------------
// PANTALLA LOTTIE — accesible desde el FAB del Tab 3
// -----------------------------------------------------------------------
class PantallaLottie extends StatefulWidget {
  const PantallaLottie({super.key});
  @override
  State<PantallaLottie> createState() => _PantallaLottieState();
}

class _PantallaLottieState extends State<PantallaLottie>
    with SingleTickerProviderStateMixin {
  late AnimationController _ctrl;

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
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Lottie.network no requiere assets en pubspec
            Lottie.network(
              'https://assets5.lottiefiles.com/packages/lf20_usmfx6bp.json',
              width: 200,
              height: 200,
              fit: BoxFit.contain,
              repeat: true,
              errorBuilder: (_, __, ___) => const Column(
                children: [
                  Icon(Icons.celebration, size: 80, color: Colors.amber),
                  Text('(Lottie requiere internet)'),
                ],
              ),
            ),
            const SizedBox(height: 16),
            const Text('Lottie desde URL', style: TextStyle(fontSize: 18)),
            const SizedBox(height: 8),
            const Text(
              'Para Lottie.asset: coloca el .json\nen assets/lottie/ y declara en pubspec',
              textAlign: TextAlign.center,
              style: TextStyle(color: Colors.grey, fontSize: 13),
            ),
            const SizedBox(height: 24),
            ElevatedButton(
              onPressed: () => Navigator.pop(context),
              child: const Text('Volver'),
            ),
          ],
        ),
      ),
    );
  }
}
```

> **Prueba esto:** Navega entre las 4 pestanas. En Tab 3 (Hero), toca una tarjeta — la caja
> de color debe expandirse con animacion Hero. Pulsa el FAB "Lottie" — debe navegar con
> `FadeTransition` a la pantalla de la animacion JSON.

Salida esperada: la app muestra 4 pestanas funcionales. El contador en Tab 1 anima con escala
en cada toque. El engranaje en Tab 2 rota con control fino. Las tarjetas en Tab 3 vuelan con
Hero. El FAB navega con fade a la pantalla Lottie.

---

## main.dart completo — referencia

El archivo del Paso 5 es el `main.dart` de referencia final. Incluye las 4 pestanas y la
pantalla Lottie en un solo archivo para facilitar la experimentacion en clase. Para el proyecto
final, cada pantalla se separa en su propio archivo dentro de `lib/screens/`.

Puntos clave del codigo completo:

- `NavigationBar` con `selectedIndex` controlado por estado del widget padre
- Cada tab es un `StatefulWidget` independiente con su propio ciclo de vida
- `SingleTickerProviderStateMixin` en cada State que usa un `AnimationController`
- `PageRouteBuilder` con `FadeTransition` para la navegacion al Lottie
- `errorBuilder` en `Lottie.network` para manejar falta de conexion

---

## Proyecto final

### Estructura

```
modulo18_animaciones/
├── assets/
│   └── lottie/
│       ├── loading.json      <- descargar de lottiefiles.com
│       └── success.json      <- descargar de lottiefiles.com
├── lib/
│   ├── main.dart             <- punto de entrada, NavigationBar con 4 tabs
│   └── screens/
│       ├── pantalla_implicitas.dart   <- AnimatedContainer, AnimatedOpacity, AnimatedSwitcher
│       ├── pantalla_explicita.dart    <- AnimationController, Tween, AnimatedBuilder
│       ├── pantalla_hero.dart         <- Hero grid + PantallaDetalle
│       └── pantalla_lottie.dart       <- Lottie + PageRouteBuilder
└── pubspec.yaml
```

### Archivos clave

**`lib/main.dart`** — configura `MaterialApp` y el `NavigationBar` principal con los 4 tabs.
Importa cada pantalla de `screens/` y las ensambla en la lista de paginas.

**`lib/screens/pantalla_implicitas.dart`** — `StatefulWidget` con `_expandido` y `_visible`.
Contiene `AnimatedContainer`, `AnimatedOpacity`, `AnimatedPadding` y `AnimatedSwitcher` con
`ScaleTransition` para el contador.

**`lib/screens/pantalla_explicita.dart`** — `StatefulWidget` con `SingleTickerProviderStateMixin`.
Contiene `AnimationController`, `Tween<double>` para rotacion, `ColorTween` para color,
`AnimatedBuilder` para reconstruccion eficiente, y botones Play/Reverse/PingPong/Stop.

**`lib/screens/pantalla_hero.dart`** — `StatelessWidget` con `GridView` de 6 tarjetas de color.
Cada tarjeta tiene `Hero(tag: 'color_$i')`. La clase `PantallaDetalle` recibe el color e indice
y muestra el Hero expandido.

**`lib/screens/pantalla_lottie.dart`** — `StatefulWidget` con `SingleTickerProviderStateMixin`
para el `AnimationController` de `LottieBuilder.asset`. Muestra la animacion JSON controlada
por botones. Incluye `PageRouteBuilder` con `FadeTransition` como ejemplo de navegacion.

---

## Guía rápida de imports

```dart
// Animaciones implicitas — incluidas en Flutter SDK, no requiere import adicional
import 'package:flutter/material.dart';
// AnimatedContainer, AnimatedOpacity, AnimatedPadding, AnimatedAlign,
// AnimatedSwitcher, AnimatedCrossFade, AnimatedPositioned, AnimatedDefaultTextStyle

// Animaciones explicitas — incluidas en Flutter SDK
import 'package:flutter/material.dart';
// AnimationController, Tween, ColorTween, CurvedAnimation, AnimatedBuilder,
// SlideTransition, FadeTransition, ScaleTransition, Transform.rotate

// Hero y PageRouteBuilder — incluidas en Flutter SDK
import 'package:flutter/material.dart';
// Hero, Navigator, MaterialPageRoute, PageRouteBuilder

// Lottie — paquete externo, requiere pubspec.yaml
import 'package:lottie/lottie.dart';
// Lottie.asset, Lottie.network, LottieBuilder.asset

// Para calculos de pi en Tween de rotacion
import 'dart:math';
// pi (= 3.14159265...)
```

---

## Cuándo usar qué

```
Situacion                                      | Solucion recomendada
-----------------------------------------------|-----------------------------------------------
Propiedad simple que cambia (color, tamaño)    | AnimatedContainer / AnimatedOpacity
Animacion de padding o alineacion              | AnimatedPadding / AnimatedAlign
Transicion entre dos widgets distintos         | AnimatedSwitcher (con ValueKey obligatoria)
Crossfade entre dos estados de un widget       | AnimatedCrossFade (crossFadeState)
Control preciso (play, pause, reversa)         | AnimationController + Tween + AnimatedBuilder
Elemento compartido entre dos pantallas        | Hero (mismo tag en origen y destino)
Transicion de pagina personalizada             | PageRouteBuilder (Slide/Fade/Scale)
Animaciones complejas prediseñadas             | Lottie.asset / Lottie.network
Dos o mas AnimationControllers en el mismo     | TickerProviderStateMixin (no Single...)
  widget                                       |
Animacion en loop automatico                   | _ctrl.repeat() o Lottie(repeat: true)
Animacion que sigue el gesto del usuario       | _ctrl.value = gesto.delta (0.0 a 1.0)
Lottie desde internet sin assets locales       | Lottie.network('https://...') sin pubspec
Reproducir Lottie una sola vez al abrir       | LottieBuilder.asset + ctrl.forward()
```

---

## Ejercicios propuestos

**Ejercicio 1 — Tarjeta expandible con `AnimatedCrossFade`**

Crea un `Card` con un titulo siempre visible y un cuerpo de texto oculto. Al pulsar la tarjeta,
usa `AnimatedCrossFade` para alternar entre un `SizedBox.shrink()` y el cuerpo con texto
completo. Usa `CrossFadeState.showFirst` para ocultar y `CrossFadeState.showSecond` para
mostrar. El cuerpo debe tener al menos 3 lineas de texto de descripcion.

```dart
AnimatedCrossFade(
  duration: const Duration(milliseconds: 300),
  crossFadeState: _expandido ? CrossFadeState.showSecond : CrossFadeState.showFirst,
  firstChild: const SizedBox.shrink(),
  secondChild: const Text('Contenido del cuerpo...'),
)
```

**Ejercicio 2 — Indicador de carga circular**

Crea un widget que use `AnimationController` con `repeat()` para rotar un icono `Icons.refresh`
continuamente. Agrega un `Tween<double>(begin: 0, end: 2 * pi)` para la rotacion completa.
Coloca el icono rotante dentro de un `CircleAvatar` de 64px con fondo azul.
Añade un boton "Detener" que llame `_ctrl.stop()` y un boton "Reanudar" que llame `_ctrl.repeat()`.

**Ejercicio 3 — Galeria Hero con imagenes de internet**

Muestra una grilla de 9 imagenes usando `Image.network` con URLs de `https://picsum.photos/200`.
Cada imagen debe tener un `Hero` con tag unico `'foto_$i'`. Al tocar una imagen se abre una
pantalla de detalle a pantalla completa con la imagen en `Image.network` del mismo URL pero
con dimensiones grandes. Agrega `GestureDetector(onTap: () => Navigator.pop(context))` en la
pantalla de detalle para volver con animacion Hero inversa.

**Ejercicio 4 — Lottie controlado con formulario**

Descarga un Lottie de LottieFiles (por ejemplo un checkmark de exito).
Crea una pantalla con un `TextFormField` y un boton "Enviar". Al pulsar "Enviar", valida que
el campo no este vacio. Si la validacion pasa, navega con `PageRouteBuilder` y `FadeTransition`
a una pantalla de exito que muestra el Lottie con `LottieBuilder.asset` y un `AnimationController`
que llama `_ctrl.forward()` en `onLoaded`. Configura `repeat: false` para que se reproduzca
solo una vez.

---

## Resumen

- Las **animaciones implicitas** (`AnimatedContainer`, `AnimatedOpacity`, `AnimatedPadding`)
  son la forma mas rapida de animar: solo cambia el estado y Flutter interpola automaticamente.
  Siempre requieren el parametro `duration`.
- **`AnimatedSwitcher`** permite transiciones entre widgets distintos. El `ValueKey` es
  obligatorio en los hijos para que Flutter detecte el cambio; sin el, la animacion no dispara.
- **`AnimationController` + `Tween` + `AnimatedBuilder`** dan control total sobre la animacion:
  inicio, pausa, reversa, ping-pong y seguimiento de gestos. Siempre usar con
  `SingleTickerProviderStateMixin` y llamar `dispose()`.
- **`Hero`** crea transiciones de elemento compartido entre pantallas con cero codigo adicional:
  solo el mismo `tag` en origen y destino. El tag debe ser unico en cada pantalla.
- **`PageRouteBuilder`** permite personalizar la transicion de pagina con `SlideTransition`,
  `FadeTransition`, `ScaleTransition` o combinaciones. Usar `reverseTransitionDuration` para
  controlar la duracion al volver.
- **Lottie** reproduce animaciones JSON de LottieFiles. `Lottie.network` no requiere pubspec;
  `Lottie.asset` requiere declarar la carpeta en `flutter.assets`. `LottieBuilder.asset` con
  `AnimationController` da control manual de reproduccion.
- Para **multiples `AnimationController`** en el mismo widget usar `TickerProviderStateMixin`
  en lugar de `Single...`. Para animaciones en loop: `_ctrl.repeat()` o `Lottie(repeat: true)`.
- El patron de **5 pasos con `const int paso = 1`** permite explorar cada concepto de forma
  aislada antes de ensamblar el proyecto final con `NavigationBar`.

---

> **Siguiente pagina →** Pagina 19: Temas, tipografia y diseno adaptativo.
