# Tutorial Flutter — Módulo 6
## Widgets: `StatelessWidget`, `StatefulWidget`, Estado y `BuildContext`

---

## ¿Qué es un Widget?

En Flutter **todo es un Widget** — texto, botones, layouts, animaciones,
incluso el tema de la app. Un widget describe cómo se ve la UI en un
momento dado. Cuando el estado cambia, Flutter reconstruye los widgets
afectados.

```
MaterialApp
└── Scaffold
    ├── AppBar → Text('Mi App')
    └── body: Column
        ├── Text('Hola')
        └── ElevatedButton → Text('Tap')
```

Comparación con Android:
```
Android/Kotlin             Flutter
───────────────────        ──────────────────────────────
View, ViewGroup       →    Widget
Activity / Fragment   →    Widget (también)
XML layout            →    Widget tree (árbol anidado)
notifyDataChanged()   →    setState() / Riverpod
```

---

## Crea el proyecto

Antes de escribir cualquier código, crea un proyecto nuevo para este módulo:

```bash
flutter create modulo06_widgets
cd modulo06_widgets
```

Abre el proyecto y limpia `lib/main.dart` — borra todo el contenido generado
y déjalo vacío. Cada paso del módulo parte desde cero en ese archivo.

```
modulo06_widgets/
├── lib/
│   └── main.dart        ← aquí trabajamos los pasos 1 al 6
├── pubspec.yaml
└── ...
```

A partir del Paso 6 (proyecto final) se crean subcarpetas dentro de `lib/`.
Por ahora solo necesitas `main.dart`.

Para correr la app:
```bash
flutter run
```

---

## Paso 1 — StatelessWidget mínimo

El widget más simple: recibe datos, los muestra, **no cambia**.
Todo en un solo archivo para arrancar rápido:

```dart
// main.dart
import 'package:flutter/material.dart';

void main() => runApp(const MaterialApp(
  home: Scaffold(body: Center(child: Saludo())),
));

class Saludo extends StatelessWidget {
  const Saludo({super.key});

  @override
  Widget build(BuildContext context) {  // describe cómo se ve
    return const Text('Hola Flutter');
  }
}
```

> Regla: si la UI **no cambia** → `StatelessWidget`.
> `build()` se llama cuando Flutter necesita dibujar o redibujar el widget.

### Prueba esto

- Agrega `style: const TextStyle(fontSize: 32, fontWeight: FontWeight.bold)` al `Text`
- Cambia `color` dentro del `TextStyle`: `Colors.indigo`, `Colors.teal`, `Colors.deepOrange`
- Agrega `letterSpacing: 6` al `TextStyle` — observa el espaciado entre letras
- Agrega `textAlign: TextAlign.center` al `Text` — ¿cambia algo? (pista: el `Text` ocupa solo el ancho de su contenido)
- Agrega un texto largo y prueba `overflow: TextOverflow.ellipsis` con `maxLines: 1`

---

## Paso 2 — StatelessWidget con parámetros

Los parámetros son siempre `final`. El `const` permite que Flutter
**reutilice** el widget si los datos no cambiaron.

### Archivo: `lib/widgets/etiqueta.dart`

```dart
import 'package:flutter/material.dart';

class Etiqueta extends StatelessWidget {
  final String texto;
  final Color  color;

  const Etiqueta({super.key, required this.texto, required this.color});

  @override
  Widget build(BuildContext context) {
    return Container(
      padding:    const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
      decoration: BoxDecoration(
        color:        color.withOpacity(0.15),
        border:       Border.all(color: color),
        borderRadius: BorderRadius.circular(20),
      ),
      child: Text(
        texto,
        style: TextStyle(color: color, fontWeight: FontWeight.w600, fontSize: 13),
      ),
    );
  }
}
```

### Cómo probarlo desde `main.dart`

```dart
import 'package:flutter/material.dart';
import 'widgets/etiqueta.dart';       // ← importar el widget

void main() => runApp(const MaterialApp(
  home: Scaffold(
    body: Center(
      child: Wrap(
        spacing: 8,
        children: [
          Etiqueta(texto: 'Activo',    color: Colors.green),
          Etiqueta(texto: 'Error',     color: Colors.red),
          Etiqueta(texto: 'En espera', color: Colors.orange),
        ],
      ),
    ),
  ),
));
```

### Prueba esto

- Cambia `BorderRadius.circular(20)` a `circular(4)` — compara la forma: píldora vs rectángulo
- Cambia `color.withOpacity(0.15)` a `0.05` y luego a `0.4` — nota la diferencia de intensidad del fondo
- En `Border.all(color: color)`, agrega `width: 2` — borde más grueso
- Agrega `boxShadow` al `BoxDecoration`:
  ```dart
  boxShadow: [BoxShadow(color: color.withOpacity(0.3), blurRadius: 8, offset: Offset(0, 2))],
  ```
- Agrega `double fontSize = 13` como parámetro y reemplaza el `13` del `TextStyle`
- Cambia `fontWeight: FontWeight.w600` a `FontWeight.w400` o `FontWeight.bold` — compara el peso

---

## Paso 3 — StatefulWidget y setState

Un `StatefulWidget` **puede cambiar** a lo largo del tiempo.
Se divide en dos clases: el widget (inmutable) y su estado (mutable).

### Archivo: `lib/widgets/contador.dart`

```dart
import 'package:flutter/material.dart';

class Contador extends StatefulWidget {
  const Contador({super.key});

  @override
  State<Contador> createState() => _ContadorState();  // ← conecta con el estado
}

class _ContadorState extends State<Contador> {
  int _valor = 0;                                      // ← variable mutable

  void _incrementar() {
    setState(() => _valor++);                          // ← notifica a Flutter
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Text('$_valor', style: const TextStyle(fontSize: 48)),
        ElevatedButton(onPressed: _incrementar, child: const Text('Sumar')),
      ],
    );
  }
}
```

### Cómo probarlo desde `main.dart`

```dart
import 'package:flutter/material.dart';
import 'widgets/contador.dart';       // ← importar el widget

void main() => runApp(const MaterialApp(
  home: Scaffold(body: Center(child: Contador())),
));
```

> `setState(() { ... })` → Flutter llama `build()` de nuevo con los nuevos valores.
> Las variables de estado viven en el `State`, no en el widget.

### Prueba esto

- Cambia `ElevatedButton` por `FilledButton` — observa la diferencia visual sin tocar la lógica
- Cambia `ElevatedButton` por `OutlinedButton` — mismo comportamiento, otro estilo
- Cambia `mainAxisAlignment: MainAxisAlignment.center` a `MainAxisAlignment.start` o `MainAxisAlignment.end`
- Agrega `crossAxisAlignment: CrossAxisAlignment.start` al `Column` — ¿cómo se alinean los hijos?
- Cambia el `fontSize: 48` del número a `72` o `24` — experimenta el impacto visual
- Cambia el `color` del número condicionalmente: `color: _valor > 5 ? Colors.red : Colors.black`

---

### Paso 3b — Acceder a parámetros desde el State

Usa `widget.` para leer los parámetros del `StatefulWidget` dentro del `State`.

### Archivo: `lib/widgets/contador_limitado.dart`

```dart
import 'package:flutter/material.dart';

class ContadorLimitado extends StatefulWidget {
  final String etiqueta;
  final int    limite;

  const ContadorLimitado({super.key, required this.etiqueta, this.limite = 10});

  @override
  State<ContadorLimitado> createState() => _ContadorLimitadoState();
}

class _ContadorLimitadoState extends State<ContadorLimitado> {
  int _valor = 0;

  @override
  Widget build(BuildContext context) {
    final enLimite = _valor >= widget.limite;    // ← widget.limite

    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Text(widget.etiqueta),                   // ← widget.etiqueta
        Text('$_valor', style: TextStyle(fontSize: 48, color: enLimite ? Colors.red : null)),
        FilledButton(
          onPressed: enLimite ? null : () => setState(() => _valor++),
          child: const Text('Sumar'),
        ),
        if (enLimite)
          const Text('Límite alcanzado', style: TextStyle(color: Colors.red)),
      ],
    );
  }
}
```

### Cómo probarlo desde `main.dart`

```dart
import 'package:flutter/material.dart';
import 'widgets/contador_limitado.dart';

void main() => runApp(const MaterialApp(
  home: Scaffold(
    body: Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          ContadorLimitado(etiqueta: 'Intentos login', limite: 3),
          SizedBox(height: 32),
          ContadorLimitado(etiqueta: 'Conexiones',     limite: 10),
        ],
      ),
    ),
  ),
));
```

### Prueba esto

- Cambia `this.limite = 10` a `this.limite = 3` desde `main.dart` — sin tocar el widget, el comportamiento cambia
- Agrega `Color colorBoton = Colors.blue` como parámetro y pásalo a `FilledButton` mediante `style:`
- Agrega `String textoBoton = 'Sumar'` y úsalo en el `child` del botón — prueba `limite: 3, textoBoton: 'Intentar'`
- Prueba `limite: 1` — el botón se deshabilita después del primer toque. ¿por qué?
- Cambia `FilledButton` por `ElevatedButton` — la lógica de `widget.limite` no cambia

---

## Paso 4 — Ciclo de vida

```dart
class _MiWidgetState extends State<MiWidget> {

  // 1. UNA VEZ al crear — inicializar timers, cargar datos, suscripciones
  @override
  void initState() {
    super.initState();
    // Equivale a onCreate() en Android
  }

  // 2. Cada vez que cambia el estado (setState) o el padre reconstruye
  @override
  Widget build(BuildContext context) {
    return const Text('widget');
  }

  // 3. Al destruir — OBLIGATORIO liberar recursos
  @override
  void dispose() {
    // timer.cancel()       ← evita fugas de memoria
    // controller.dispose()
    // subscription.cancel()
    super.dispose();        // siempre al final
  }
}
```

**Flujo:**
```
createState → initState → build → [setState → build] → dispose
```

Flujo completo con todos los métodos:
```
createState()
     ↓
initState()             ← UNA vez — inicializar recursos
     ↓
didChangeDependencies() ← tras initState y cuando cambia el tema/locale
     ↓
build()                 ← cada rebuild (setState o padre cambia)
     ↓
didUpdateWidget()       ← cuando el padre cambia los parámetros
     ↓
deactivate()            ← widget quitado del árbol temporalmente
     ↓
dispose()               ← limpieza final
```

### Ciclo de vida con Timer

### Archivo: `lib/widgets/reloj.dart`

```dart
import 'dart:async';
import 'package:flutter/material.dart';

class Reloj extends StatefulWidget {
  const Reloj({super.key});

  @override
  State<Reloj> createState() => _RelojState();
}

class _RelojState extends State<Reloj> {
  late Timer _timer;
  int        _segundos = 0;

  @override
  void initState() {
    super.initState();
    _timer = Timer.periodic(const Duration(seconds: 1), (_) {
      setState(() => _segundos++);
    });
  }

  @override
  void dispose() {
    _timer.cancel();   // ← SIEMPRE cancelar en dispose()
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final h = _segundos ~/ 3600;
    final m = (_segundos % 3600) ~/ 60;
    final s = _segundos % 60;
    return Text(
      '$h:${m.toString().padLeft(2, '0')}:${s.toString().padLeft(2, '0')}',
      style: const TextStyle(fontSize: 32, fontFamily: 'monospace'),
    );
  }
}
```

### Cómo probarlo desde `main.dart`

```dart
import 'package:flutter/material.dart';
import 'widgets/reloj.dart';

void main() => runApp(const MaterialApp(
  home: Scaffold(
    appBar: AppBar(title: Text('Cronómetro')),
    body: Center(child: Reloj()),
  ),
));
```

> Si haces `setState()` desde un callback asíncrono, verifica
> `if (!mounted) return;` antes de llamarlo.

### Prueba esto

- Cambia `Duration(seconds: 1)` a `Duration(milliseconds: 500)` — el reloj avanza al doble de velocidad
- Cambia `Duration(seconds: 1)` a `Duration(seconds: 2)` — el contador salta de 2 en 2
- Cambia el `fontSize: 32` a `48` y agrega `fontWeight: FontWeight.bold` al `TextStyle`
- Agrega `color:` al `TextStyle` condicionalmente según el tiempo:
  ```dart
  color: _segundos > 60 ? Colors.red : _segundos > 30 ? Colors.orange : Colors.green,
  ```
- Agrega `bool _pausado = false` en el `State`. En `initState`, guarda el timer en `_timer`.
  Agrega un método `_togglePausa()` que llame `_timer.cancel()` cuando pausa y cree un nuevo `Timer.periodic` al reanudar

---

## Paso 5 — BuildContext

`BuildContext` es la referencia del widget en el árbol.
Permite acceder al tema, tamaño de pantalla y datos de ancestros:

### Archivo: `lib/screens/pantalla_contexto.dart`

```dart
import 'package:flutter/material.dart';

class PantallaContexto extends StatelessWidget {
  const PantallaContexto({super.key});

  @override
  Widget build(BuildContext context) {
    final tema    = Theme.of(context);           // colores, fuentes
    final colores = tema.colorScheme;
    final tamanio = MediaQuery.sizeOf(context);  // ancho y alto en px
    final esMovil = tamanio.width < 600;

    return Scaffold(
      backgroundColor: colores.surface,
      appBar: AppBar(
        backgroundColor: colores.primaryContainer,
        title: Text(
          'Pantalla ${esMovil ? "móvil" : "tablet"}',
          style: tema.textTheme.titleLarge,
        ),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'Tamaño: ${tamanio.width.toStringAsFixed(0)} × ${tamanio.height.toStringAsFixed(0)}',
              style: tema.textTheme.bodyLarge,
            ),
            Text('Orientación: ${MediaQuery.orientationOf(context).name}'),
            const SizedBox(height: 16),
            Container(
              padding:    const EdgeInsets.all(16),
              decoration: BoxDecoration(
                color:        colores.primaryContainer,
                borderRadius: BorderRadius.circular(12),
              ),
              child: Text(
                'Color del tema aplicado',
                style: TextStyle(color: colores.onPrimaryContainer),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

### Cómo probarlo desde `main.dart`

```dart
import 'package:flutter/material.dart';
import 'screens/pantalla_contexto.dart';

void main() => runApp(const MaterialApp(
  home: PantallaContexto(),    // ← la pantalla ya tiene Scaffold
));
```

Para probar con un tema diferente, cambia `seedColor`:
```dart
void main() => runApp(MaterialApp(
  theme: ThemeData(
    colorScheme:  ColorScheme.fromSeed(seedColor: Colors.teal),  // ← cambia esto
    useMaterial3: true,
  ),
  home: const PantallaContexto(),
));
```

Accesos comunes de `BuildContext`:
```dart
Theme.of(context).colorScheme.primary     // color principal del tema
Theme.of(context).textTheme.bodyLarge     // estilo de texto
MediaQuery.sizeOf(context).width          // ancho de pantalla
MediaQuery.orientationOf(context)         // portrait / landscape
Navigator.of(context).push(...)           // navegación
```

### Prueba esto

- Cambia `seedColor: Colors.indigo` a `Colors.deepPurple`, `Colors.teal` o `Colors.deepOrange` — observa cómo cambia toda la UI automáticamente
- Cambia `colores.primaryContainer` por `colores.secondaryContainer` o `colores.tertiaryContainer`
- En el `AppBar`, cambia `backgroundColor: colores.primaryContainer` a `null` — usa el color por defecto del tema
- Agrega `brightness: Brightness.dark` dentro de `ColorScheme.fromSeed(...)` para probar modo oscuro
- Cambia `colores.surface` del `Scaffold` a `colores.surfaceVariant` — nota la diferencia de contraste

---

## Paso 6 — Composición de widgets

En Flutter la UI se construye **componiendo widgets pequeños**.
Cada widget hace **una sola cosa** (principio de responsabilidad única).

### Archivo: `lib/widgets/indicador.dart`

```dart
import 'package:flutter/material.dart';

// Widget reutilizable — muestra un número con etiqueta de color
class Indicador extends StatelessWidget {
  final String label;
  final int    valor;
  final Color  color;

  const Indicador({super.key, required this.label, required this.valor, required this.color});

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(
          '$valor',
          style: TextStyle(fontSize: 28, fontWeight: FontWeight.bold, color: color),
        ),
        Text(label, style: TextStyle(fontSize: 12, color: Colors.grey.shade600)),
      ],
    );
  }
}
```

### Cómo probarlo desde `main.dart`

```dart
import 'package:flutter/material.dart';
import 'widgets/indicador.dart';

void main() => runApp(const MaterialApp(
  home: Scaffold(
    body: Center(
      child: Row(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Indicador(label: 'Activas',  valor: 8,  color: Colors.green),
          SizedBox(width: 32),
          Indicador(label: 'Cerradas', valor: 3,  color: Colors.grey),
          SizedBox(width: 32),
          Indicador(label: 'Errores',  valor: 1,  color: Colors.red),
        ],
      ),
    ),
  ),
));
```

### Prueba esto

- Cambia `fontSize: 28` a `36` o `20` — observa el espacio que ocupa en el `Row`
- Cambia `fontWeight: FontWeight.bold` a `FontWeight.w300` — ¿sigue siendo legible?
- Cambia `crossAxisAlignment: CrossAxisAlignment.start` a `CrossAxisAlignment.center` en el `Column`
- Agrega `String? subtitulo` como parámetro opcional y muéstralo debajo del `label`:
  ```dart
  if (subtitulo != null)
    Text(subtitulo!, style: TextStyle(fontSize: 10, color: Colors.grey.shade400)),
  ```
- Cambia `Colors.grey.shade600` del `label` a `Colors.grey.shade400` o `Colors.grey.shade900` — compara el contraste

> Widgets con `_` al inicio del nombre son **privados al archivo** — convención para
> widgets auxiliares que no se reutilizan fuera de ese archivo.

---

## Proyecto — Monitor de Infraestructura

Proyecto que combina todos los pasos en una app real con múltiples archivos.

```
monitor_infraestructura/
└── lib/
    ├── main.dart                          ← entrada
    ├── models/
    │   └── servidor.dart                 ← modelo de datos (sin UI)
    ├── widgets/
    │   ├── etiqueta.dart                 ← Paso 2
    │   ├── indicador.dart                ← Paso 6
    │   ├── tarjeta_metrica.dart          ← composición
    │   └── tarjeta_servidor.dart         ← composición + widget privado
    └── screens/
        └── pantalla_dashboard.dart       ← StatefulWidget + Timer (Pasos 3-5)
```

---

### `lib/models/servidor.dart`

Modelo de datos puro — sin UI, sin Flutter:

```dart
class Servidor {
  final String nombre;
  final double cpu;
  final double ram;
  final int    conexiones;
  final bool   activo;

  const Servidor({
    required this.nombre,
    required this.cpu,
    required this.ram,
    required this.conexiones,
    required this.activo,
  });

  // Getter derivado — no almacena estado, lo calcula
  bool get esCritico => cpu > 85 || ram > 90;
}
```

---

### `lib/widgets/tarjeta_metrica.dart`

Composición: ícono + valor + título:

```dart
import 'package:flutter/material.dart';

class TarjetaMetrica extends StatelessWidget {
  final String   titulo;
  final String   valor;
  final IconData icono;
  final Color    colorIcono;
  final String?  subtitulo;

  const TarjetaMetrica({
    super.key,
    required this.titulo,
    required this.valor,
    required this.icono,
    required this.colorIcono,
    this.subtitulo,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Row(
          children: [
            Container(
              padding:    const EdgeInsets.all(10),
              decoration: BoxDecoration(
                color:        colorIcono.withOpacity(0.1),
                borderRadius: BorderRadius.circular(10),
              ),
              child: Icon(icono, color: colorIcono, size: 24),
            ),
            const SizedBox(width: 12),
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(valor,  style: const TextStyle(fontSize: 22, fontWeight: FontWeight.bold)),
                  Text(titulo, style: TextStyle(color: Colors.grey.shade600, fontSize: 13)),
                  if (subtitulo != null)
                    Text(subtitulo!, style: TextStyle(color: Colors.grey.shade500, fontSize: 11)),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

### `lib/widgets/tarjeta_servidor.dart`

`_Barra` es un widget privado — solo se usa en este archivo:

```dart
import 'package:flutter/material.dart';
import '../models/servidor.dart';

class TarjetaServidor extends StatelessWidget {
  final Servidor servidor;

  const TarjetaServidor({super.key, required this.servidor});

  @override
  Widget build(BuildContext context) {
    final color = servidor.activo
        ? (servidor.esCritico ? Colors.orange : Colors.green)
        : Colors.red;

    return Card(
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Row(
              children: [
                Icon(servidor.activo ? Icons.circle : Icons.circle_outlined, color: color, size: 12),
                const SizedBox(width: 6),
                Expanded(
                  child: Text(servidor.nombre,
                    style: const TextStyle(fontWeight: FontWeight.bold),
                    overflow: TextOverflow.ellipsis),
                ),
                if (servidor.esCritico && servidor.activo)
                  const Icon(Icons.warning_amber, color: Colors.orange, size: 16),
              ],
            ),
            const SizedBox(height: 8),
            _Barra(label: 'CPU', valor: servidor.cpu, critico: servidor.cpu > 85),
            const SizedBox(height: 4),
            _Barra(label: 'RAM', valor: servidor.ram, critico: servidor.ram > 90),
            const SizedBox(height: 6),
            Text('${servidor.conexiones} conexiones',
              style: TextStyle(fontSize: 11, color: Colors.grey.shade600)),
          ],
        ),
      ),
    );
  }
}

// Widget privado — no importable desde otros archivos
class _Barra extends StatelessWidget {
  final String label;
  final double valor;
  final bool   critico;

  const _Barra({required this.label, required this.valor, required this.critico});

  @override
  Widget build(BuildContext context) {
    final color = critico ? Colors.orange : Colors.green;
    return Row(
      children: [
        SizedBox(width: 30, child: Text(label, style: const TextStyle(fontSize: 11))),
        Expanded(
          child: LinearProgressIndicator(
            value:           valor / 100,
            backgroundColor: Colors.grey.shade200,
            color:           color,
          ),
        ),
        const SizedBox(width: 6),
        SizedBox(
          width: 36,
          child: Text('${valor.toStringAsFixed(0)}%',
            style: TextStyle(fontSize: 11, color: color, fontWeight: FontWeight.w600),
            textAlign: TextAlign.right),
        ),
      ],
    );
  }
}
```

---

### `lib/screens/pantalla_dashboard.dart`

`StatefulWidget` con `Timer` — aplica Pasos 3, 4 y 5:

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import '../models/servidor.dart';
import '../widgets/tarjeta_metrica.dart';
import '../widgets/tarjeta_servidor.dart';

class PantallaDashboard extends StatefulWidget {
  const PantallaDashboard({super.key});

  @override
  State<PantallaDashboard> createState() => _PantallaDashboardState();
}

class _PantallaDashboardState extends State<PantallaDashboard> {
  late Timer         _timer;
  List<Servidor>     _servidores = [];
  bool               _cargando   = true;
  int                _ciclo      = 0;

  @override
  void initState() {
    super.initState();
    _cargarDatos();
    _timer = Timer.periodic(const Duration(seconds: 3), (_) => _cargarDatos());
  }

  @override
  void dispose() {
    _timer.cancel();   // ← cancelar timer — evita fugas de memoria
    super.dispose();
  }

  void _cargarDatos() {
    Future.delayed(const Duration(milliseconds: 400), () {
      if (!mounted) return;   // ← protege el setState asíncrono
      setState(() {
        _ciclo++;
        _cargando   = false;
        _servidores = [
          Servidor(nombre: 'web-01',      cpu: 45 + (_ciclo * 3 % 30), ram: 62, conexiones: 230,  activo: true),
          Servidor(nombre: 'web-02',      cpu: 38 + (_ciclo * 7 % 20), ram: 55, conexiones: 180,  activo: true),
          Servidor(nombre: 'api-primary', cpu: 72 + (_ciclo * 5 % 25), ram: 78, conexiones: 450,  activo: true),
          Servidor(nombre: 'db-primary',  cpu: 88 + (_ciclo % 10),     ram: 91, conexiones: 80,   activo: true),
          Servidor(nombre: 'cache',       cpu: 15,                     ram: 40, conexiones: 1200, activo: true),
          Servidor(nombre: 'worker-01',   cpu: 0,                      ram: 0,  conexiones: 0,    activo: false),
        ];
      });
    });
  }

  @override
  Widget build(BuildContext context) {
    final activos = _servidores.where((s) => s.activo).length;
    final alertas = _servidores.where((s) => s.esCritico).length;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Monitor de Infraestructura'),
        actions: [
          if (_cargando)
            const Padding(
              padding: EdgeInsets.all(12),
              child: SizedBox(width: 20, height: 20,
                child: CircularProgressIndicator(strokeWidth: 2)),
            ),
          IconButton(icon: const Icon(Icons.refresh), onPressed: _cargarDatos),
        ],
      ),
      body: _cargando && _servidores.isEmpty
          ? const Center(child: CircularProgressIndicator())
          : Column(
              children: [
                // Resumen — StatelessWidgets reciben datos del State
                Padding(
                  padding: const EdgeInsets.all(12),
                  child: Row(
                    children: [
                      Expanded(child: TarjetaMetrica(
                        titulo: 'Activos', valor: '$activos/${_servidores.length}',
                        icono: Icons.dns, colorIcono: Colors.green,
                      )),
                      const SizedBox(width: 8),
                      Expanded(child: TarjetaMetrica(
                        titulo: 'Alertas', valor: '$alertas',
                        icono: Icons.warning_amber,
                        colorIcono: alertas > 0 ? Colors.orange : Colors.green,
                        subtitulo:  alertas > 0 ? 'Requieren atención' : null,
                      )),
                    ],
                  ),
                ),

                // Grid de servidores
                Expanded(
                  child: GridView.builder(
                    padding: const EdgeInsets.symmetric(horizontal: 12),
                    gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                      crossAxisCount: 2, childAspectRatio: 1.3,
                      crossAxisSpacing: 8, mainAxisSpacing: 8,
                    ),
                    itemCount:   _servidores.length,
                    itemBuilder: (_, i) => TarjetaServidor(servidor: _servidores[i]),
                  ),
                ),

                Padding(
                  padding: const EdgeInsets.all(8),
                  child: Text('Ciclo #$_ciclo · actualiza cada 3 s',
                    style: TextStyle(fontSize: 11, color: Colors.grey.shade500)),
                ),
              ],
            ),
    );
  }
}
```

---

### `lib/main.dart`

Punto de entrada — importa la pantalla principal:

```dart
import 'package:flutter/material.dart';
import 'screens/pantalla_dashboard.dart';

void main() => runApp(const AppMonitor());

class AppMonitor extends StatelessWidget {
  const AppMonitor({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title:                      'Monitor Infraestructura',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme:  ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const PantallaDashboard(),
    );
  }
}
```

Para probar un widget específico sin la pantalla completa,
cambia solo el `home:`:

```dart
// Probar TarjetaMetrica aislada
import 'widgets/tarjeta_metrica.dart';

home: const Scaffold(
  body: Center(
    child: Padding(
      padding: EdgeInsets.all(16),
      child: TarjetaMetrica(
        titulo: 'Servidores activos', valor: '5/6',
        icono: Icons.dns, colorIcono: Colors.green,
      ),
    ),
  ),
),
```

---

## Guía rápida de imports

```dart
// Widgets de la misma carpeta (lib/widgets/ → lib/widgets/)
import 'etiqueta.dart';

// Widgets desde screens (lib/screens/ → lib/widgets/)
import '../widgets/tarjeta_metrica.dart';

// Widgets desde screens (lib/screens/ → lib/models/)
import '../models/servidor.dart';

// Desde main.dart (lib/ → lib/widgets/ o lib/screens/)
import 'widgets/etiqueta.dart';
import 'screens/pantalla_dashboard.dart';
```

---

## Cuándo usar cada widget

```
StatelessWidget                      StatefulWidget
─────────────────────────────        ────────────────────────────────
Solo muestra datos recibidos         Necesita reaccionar al usuario
No hay timers ni animaciones         Tiene campos, switches, checkboxes
Los datos vienen del padre           Tiene timers o animaciones propias
  o de un estado externo             Tiene datos que cambia internamente
```

---

## Ejercicios

1. **Semáforo interactivo** — `StatefulWidget` con tres círculos (rojo, amarillo, verde).
   Al tocar un botón avanza cíclicamente. El activo brilla, los demás están opacos.
   Texto asociado: "STOP", "PRECAUCIÓN", "GO". Crea `lib/widgets/semaforo.dart` e impórtalo en `main.dart`.

2. **Cronómetro de sesión** — Basado en `Reloj` del Paso 4. Agrega: formato `HH:MM:SS`,
   botón pausa/reanudación, color que cambia por tramos de tiempo.

3. **Tarjeta expandible** — Nombre de dispositivo de red. Al tocar → se expande
   mostrando IP, MAC, estado y latencia. Al tocar de nuevo → colapsa.
   Usa `bool _expandido` y `AnimatedCrossFade`. Crea `lib/widgets/tarjeta_dispositivo.dart`.

4. **Layout responsivo** — Con `MediaQuery.orientationOf(context)`:
   `TarjetaMetrica` en columna en retrato, en fila en paisaje.
   Verifica rotando el emulador.

---

## Resumen

- En Flutter **todo es un widget** — texto, layouts, botones, tema. La UI es un árbol de widgets anidados.
- `StatelessWidget` solo muestra datos recibidos. Parámetros siempre `final`. Usa `const` cuando sea posible.
- `StatefulWidget` = widget (inmutable, configura) + `State` (mutable, almacena datos que cambian).
- `setState(() { ... })` notifica a Flutter — llama `build()` de nuevo con los nuevos valores.
- `initState()` → **una vez** al crear. Inicializar timers, streams, carga de datos.
- `dispose()` → **obligatorio** para cancelar `Timer`, cerrar `StreamSubscription`, liberar `AnimationController`.
- `mounted` protege `setState()` en callbacks asíncronos: `if (!mounted) return;`.
- `BuildContext` da acceso al tema (`Theme.of(context)`) y tamaño (`MediaQuery.sizeOf(context)`).
- La composición de widgets pequeños y reutilizables es el patrón fundamental de Flutter.

---

> **Siguiente →** Módulo 7: Layouts — `Column`, `Row`, `Stack`, `Container`, `Expanded` y más.
