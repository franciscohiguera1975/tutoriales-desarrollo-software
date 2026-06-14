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

## Patrones clave — de un vistazo

Referencia rápida de los patrones del módulo. Léelos ahora y vuelve
a consultarlos conforme avanzas en cada paso.

---

### `StatelessWidget` — estructura completa

```dart
class MiWidget extends StatelessWidget {
  // Parámetros: siempre final, definidos en el constructor
  final String  titulo;           // requerido
  final int     valor;            // requerido
  final Color   color;            // requerido
  final String? subtitulo;        // opcional — null por defecto
  final double  fontSize;         // opcional con valor por defecto
  final VoidCallback? onTap;      // callback opcional

  const MiWidget({                // const si todos los params son const
    super.key,
    required this.titulo,
    required this.valor,
    required this.color,
    this.subtitulo,               // no lleva required — es opcional
    this.fontSize   = 14,         // valor por defecto
    this.onTap,
  });

  @override
  Widget build(BuildContext context) {   // se llama cada vez que Flutter dibuja
    return Text(titulo, style: TextStyle(fontSize: fontSize, color: color));
  }
}

// Uso con const → Flutter reutiliza el widget si los datos no cambiaron
const MiWidget(titulo: 'Servidor', valor: 3, color: Colors.green)
```

---

### `StatefulWidget` + `State` — estructura y separación

```dart
class MiWidget extends StatefulWidget {
  // Datos inmutables — van en el StatefulWidget
  final String etiqueta;
  final int    limite;

  const MiWidget({super.key, required this.etiqueta, this.limite = 10});

  @override
  State<MiWidget> createState() => _MiWidgetState();
}

class _MiWidgetState extends State<MiWidget> {
  // Variables mutables — van en el State, cambian con setState()
  int    _contador = 0;
  bool   _activo   = true;
  String _mensaje  = '';

  // Acceder al StatefulWidget padre: widget.etiqueta, widget.limite
  // Solo lectura — no se puede modificar desde el State

  @override
  Widget build(BuildContext context) {
    return Text('${widget.etiqueta}: $_contador / ${widget.limite}');
  }
}
```

---

### `setState()` — qué hace y qué va adentro

```dart
void _accion() {
  setState(() {
    // Todo lo que cambia va DENTRO del bloque
    _contador++;
    _activo   = !_activo;
    _mensaje  = _activo ? 'Activo' : 'Inactivo';
  });
  // Flutter llama build() de nuevo con estos nuevos valores
}

// ✗ INCORRECTO — cambia el valor pero NO actualiza la UI:
_contador++;                 // sin setState, el widget no se redibuja

// ✗ INCORRECTO — lógica costosa dentro de setState:
setState(() {
  _datos = calcularTodo();   // cálculos pesados fuera, setState solo asigna
});

// ✓ CORRECTO — lógica fuera, solo asignación dentro:
final resultado = calcularTodo();
setState(() => _datos = resultado);
```

---

### Cambio de estatus en la UI

El estado (`_activo`, `_valor`, `_cargando`) **controla directamente** cómo se ve la UI.
Estos son los patrones más usados:

```dart
// ── 1. Color condicional ──────────────────────────────────────────
color: _activo ? Colors.green : Colors.red

// ── 2. Ícono condicional ──────────────────────────────────────────
Icon(_activo ? Icons.check_circle : Icons.cancel,
     color: _activo ? Colors.green : Colors.red)

// ── 3. Texto condicional ──────────────────────────────────────────
Text(_activo ? 'En línea' : 'Fuera de línea')

// ── 4. Tres o más estados con switch ─────────────────────────────
Text(switch (_estado) {
  'ok'       => 'Funcionando',
  'warning'  => 'Degradado',
  'error'    => 'Caído',
  _          => 'Desconocido',
})

// ── 5. Widget que aparece / desaparece ───────────────────────────
if (!_activo)                     // solo se muestra si la condición es true
  const Text('Requiere atención', style: TextStyle(color: Colors.red))

// ── 6. Botón desactivado — onPressed: null lo deshabilita ────────
FilledButton(
  onPressed: _bloqueado ? null : _ejecutar,   // null = sin toque, gris visual
  child: const Text('Ejecutar'),
)

// ── 7. Opacidad condicional ───────────────────────────────────────
Opacity(
  opacity: _activo ? 1.0 : 0.4,   // 1.0 = visible · 0.0 = invisible
  child: Icon(Icons.settings),
)
```

---

### Ciclo de vida de `StatefulWidget`

```dart
class _MiState extends State<MiWidget> {

  @override
  void initState() {
    super.initState();            // ← siempre PRIMERO
    // UNA sola vez al crear el State
    // → inicializar Timer, cargar datos, crear AnimationController
    // NO llamar setState aquí — la UI aún no existe
    _timer = Timer.periodic(Duration(seconds: 1), _tick);
  }

  @override
  void didUpdateWidget(MiWidget old) {
    super.didUpdateWidget(old);
    // El padre cambió los parámetros del widget
    if (old.limite != widget.limite) { /* reaccionar al cambio */ }
  }

  @override
  Widget build(BuildContext context) { /* ... */ }

  @override
  void dispose() {
    _timer.cancel();              // ← liberar TODOS los recursos
    _controller.dispose();        //   si no se liberan → fuga de memoria
    super.dispose();              // ← siempre AL FINAL
  }
}

// Proteger setState en callbacks asíncronos o timers:
void _tick(_) {
  if (!mounted) return;           // el widget ya fue destruido, no hacer nada
  setState(() => _segundos++);
}
```

Flujo completo:
```
createState → initState → build → [setState → build] → dispose
```

---

### `BuildContext` — accesos más comunes

```dart
// Dentro de build(BuildContext context):

// Tema — colores y tipografía del MaterialApp
final tema    = Theme.of(context);
final colores = tema.colorScheme;        // primary, secondary, surface, error…
final fuentes = tema.textTheme;          // bodyLarge, titleMedium, labelSmall…

colores.primary           // color principal
colores.primaryContainer  // variante más suave del principal
colores.surface           // fondo de tarjetas
colores.onSurface         // texto sobre surface

// Tamaño y orientación de pantalla
final ancho   = MediaQuery.sizeOf(context).width;
final alto    = MediaQuery.sizeOf(context).height;
final esMovil = ancho < 600;
final esRetrato = MediaQuery.orientationOf(context) == Orientation.portrait;

// Navegación
Navigator.of(context).push(MaterialPageRoute(builder: (_) => OtraPantalla()));
Navigator.of(context).pop();

// ⚠ No usar context después de un await sin verificar mounted:
// await algo();
// if (!context.mounted) return;   // ← necesario en Dart 3+
// Theme.of(context)...
```

---

## Crea el proyecto

```bash
flutter create modulo06_widgets
cd modulo06_widgets
```

Abre el proyecto, borra todo el contenido de `lib/main.dart` y déjalo vacío.
El Paso 1 parte desde cero en ese archivo.

```
modulo06_widgets/
├── lib/
│   ├── main.dart              ← aquí trabajamos
│   ├── widgets/               ← se crea al llegar al Paso 1b
│   └── screens/               ← se crea al llegar al Paso 5
├── pubspec.yaml
└── ...
```

Para correr la app:
```bash
flutter run
```

---

## Paso 1 — StatelessWidget mínimo

El widget más simple: recibe datos, los muestra, **no cambia**.
Escribe esto en `lib/main.dart`:

```dart
// lib/main.dart
import 'package:flutter/material.dart';

void main() => runApp(const MaterialApp(
  home: Scaffold(body: Center(child: Saludo())),
));

class Saludo extends StatelessWidget {
  const Saludo({super.key});

  @override
  Widget build(BuildContext context) {   // describe cómo se ve
    return const Text('Hola Flutter', style: TextStyle(fontSize: 32));
  }
}
```

> Regla: si la UI **no cambia** → `StatelessWidget`.
> `build()` se llama cuando Flutter necesita dibujar o redibujar el widget.

### Prueba esto

- Agrega `fontWeight: FontWeight.bold` y `letterSpacing: 4` al `TextStyle`
- Cambia `color`: `Colors.indigo`, `Colors.teal`, `Colors.deepOrange`
- Agrega `textAlign: TextAlign.center` — ¿cambia algo? El `Text` solo ocupa el ancho de su contenido; envuélvelo en `SizedBox(width: double.infinity)` y observa la diferencia
- Prueba `overflow: TextOverflow.ellipsis` con `maxLines: 1` y un texto muy largo
- Agrega `shadows: [Shadow(color: Colors.black26, blurRadius: 4, offset: Offset(2,2))]` dentro del `TextStyle`
- Reemplaza `Text` por `SelectableText` — el texto se vuelve seleccionable con doble toque

---

## Convierte `main.dart` en selector de pasos

Antes de avanzar al Paso 2, actualiza `main.dart` **una sola vez**.
A partir de aquí solo cambias el número de `paso` para navegar entre ejemplos
— el Paso 1 sigue funcionando igual con `paso = 1`:

```dart
// lib/main.dart
import 'package:flutter/material.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1   StatelessWidget mínimo                             │
// │  2  Paso 1b  Widgets básicos — catálogo                        │
// │  3  Paso 2   StatelessWidget con parámetros                     │
// │  4  Paso 3   StatefulWidget / setState / cambio de estatus      │
// │  5  Paso 3b  Parámetros en StatefulWidget                       │
// │  6  Paso 4   Ciclo de vida con Timer                            │
// │  7  Paso 5   BuildContext                                        │
// │  8  Paso 6   Composición de widgets                             │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 1;

void main() => runApp(MaterialApp(
  debugShowCheckedModeBanner: false,
  home: switch (paso) {
    1 => const Scaffold(body: Center(child: Saludo())),
    _ => Scaffold(body: Center(child: Text('Paso $paso: crea el widget primero'))),
  },
));

class Saludo extends StatelessWidget {
  const Saludo({super.key});
  @override
  Widget build(BuildContext context) =>
      const Text('Hola Flutter', style: TextStyle(fontSize: 32));
}
```

Guarda y verifica que la app sigue mostrando "Hola Flutter". Listo — de ahora
en adelante cada paso nuevo solo requiere agregar un `import` y un `case` al switch.

---

## Paso 1b — Widgets básicos

Antes de crear widgets propios, explora los de presentación e interacción más usados.
Crea un único archivo y agrega cada bloque al avanzar en cada sección.

### Archivo: `lib/widgets/catalogo_basicos.dart`

Crea la carpeta `lib/widgets/` y dentro el archivo con esta estructura vacía:

```dart
import 'package:flutter/material.dart';

class CatalogoBasicos extends StatelessWidget {
  const CatalogoBasicos({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Widgets básicos')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          // ← pega aquí cada bloque al avanzar
        ],
      ),
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'widgets/catalogo_basicos.dart';
```

```dart
2 => const CatalogoBasicos(),
```

Cambia `paso = 2` y guarda — la app muestra la pantalla vacía con el AppBar.

---

### Bloque 1 — `Text` y `TextStyle`

`Text` muestra una cadena. Todo el estilo vive en `TextStyle`.
Pega esto en `children: [ ... ]` y recarga:

```dart
// ── Text básico ───────────────────────────────────────────────────────
const Text(
  'nginx-proxy: En línea',
  style: TextStyle(
    fontSize:      20,
    fontWeight:    FontWeight.bold,    // .w100–.w900  ·  .bold = .w700
    color:         Colors.green,
    letterSpacing: 0.5,
    fontStyle:     FontStyle.normal,   // .italic
    decoration:    TextDecoration.none,
    //             .underline  .lineThrough  .overline
  ),
),
const SizedBox(height: 8),

// ── Alineación y desbordamiento ───────────────────────────────────────
SizedBox(
  width: double.infinity,
  child: Text(
    'api-gateway-produccion-region-us-east → sin respuesta',
    textAlign: TextAlign.center,        // .left  .right  .justify  .start  .end
    maxLines:  1,
    overflow:  TextOverflow.ellipsis,   // .clip  .fade  .visible
  ),
),
const SizedBox(height: 8),

// ── Text.rich — estilos distintos en un solo widget ───────────────────
const Text.rich(
  TextSpan(children: [
    TextSpan(text: 'Estado: ',
        style: TextStyle(fontWeight: FontWeight.w600)),
    TextSpan(text: 'CRÍTICO',
        style: TextStyle(color: Colors.red, fontWeight: FontWeight.bold)),
    TextSpan(text: ' — última revisión hace 5 min',
        style: TextStyle(color: Colors.grey, fontSize: 12)),
  ]),
),
const SizedBox(height: 8),

// ── SelectableText — el usuario puede seleccionar y copiar ───────────
const SelectableText(
  '10.0.0.12:5432',
  style: TextStyle(fontFamily: 'monospace', fontSize: 14),
),
const Divider(height: 32),
```

#### Prueba esto
- Cambia `overflow: TextOverflow.ellipsis` a `.fade` y `.clip` — efectos distintos de truncamiento
- Agrega `shadows: [Shadow(color: Colors.black26, blurRadius: 4, offset: Offset(2,2))]` al `TextStyle` del primer `Text`
- Cambia `maxLines: 1` a `maxLines: 2` y alarga el texto — aparece la segunda línea
- Agrega `softWrap: false` al segundo `Text` — el texto no salta de línea
- Prueba `TextDecoration.underline` y `TextDecoration.lineThrough` en `decoration`
- Cambia `textAlign: TextAlign.justify` en el segundo `Text` — la justificación completa se ve en textos largos

---

### Bloque 2 — `Icon`

```dart
// Agrega a children: [ ... ]

Row(
  mainAxisAlignment: MainAxisAlignment.spaceEvenly,
  children: const [
    Icon(Icons.check_circle,  size: 40, color: Colors.green),
    Icon(Icons.cancel,        size: 40, color: Colors.red),
    Icon(Icons.warning_amber, size: 40, color: Colors.orange),
    Icon(Icons.dns,           size: 40, color: Colors.indigo),
    Icon(Icons.wifi_off,      size: 40, color: Colors.grey),
  ],
),
const SizedBox(height: 8),
const Icon(Icons.settings,
    size:          24,
    color:         Colors.blueGrey,
    semanticLabel: 'Configuración'),   // leído por lectores de pantalla
const Divider(height: 32),
```

> Busca íconos en [fonts.google.com/icons](https://fonts.google.com/icons) →
> copia el nombre y úsalo como `Icons.nombre`.

#### Prueba esto
- Cambia `size: 40` a `size: 80` y a `size: 14` — escala con un parámetro
- Reemplaza `color: Colors.green` por `color: Theme.of(context).colorScheme.primary` — usa el color del tema (quita `const` del `Row`)
- Prueba `Icons.check_circle_outline` vs `Icons.check_circle` — contorno vs relleno
- Envuelve un `Icon` en `Tooltip(message: 'Servidor activo', child: Icon(...))` — aparece tooltip al mantener pulsado

---

### Bloque 3 — Botones

Flutter tiene cuatro botones y sus variantes `.icon`:

```dart
// Agrega a children: [ ... ]

// ── Cuatro variantes ──────────────────────────────────────────────────
Wrap(
  spacing: 8, runSpacing: 8,
  children: [
    ElevatedButton(onPressed: () {}, child: const Text('ElevatedButton')),
    FilledButton(  onPressed: () {}, child: const Text('FilledButton')),
    OutlinedButton(onPressed: () {}, child: const Text('OutlinedButton')),
    TextButton(    onPressed: () {}, child: const Text('TextButton')),
    ElevatedButton(onPressed: null,  child: const Text('Desactivado')),
    //             ↑ onPressed: null → desactiva el botón visualmente
  ],
),
const SizedBox(height: 12),

// ── Variantes .icon ───────────────────────────────────────────────────
Wrap(
  spacing: 8, runSpacing: 8,
  children: [
    ElevatedButton.icon(
      onPressed: () {},
      icon:  const Icon(Icons.refresh, size: 18),
      label: const Text('Reiniciar'),
    ),
    FilledButton.icon(
      onPressed: () {},
      icon:  const Icon(Icons.stop, size: 18),
      label: const Text('Detener'),
    ),
    IconButton(
      onPressed: () {},
      icon:     const Icon(Icons.settings),
      color:    Colors.indigo,
      iconSize: 28,
    ),
  ],
),
const SizedBox(height: 12),

// ── Botón con estilo personalizado ────────────────────────────────────
ElevatedButton(
  onPressed: () {},
  style: ElevatedButton.styleFrom(
    backgroundColor: Colors.red.shade600,
    foregroundColor: Colors.white,
    padding:     const EdgeInsets.symmetric(horizontal: 32, vertical: 14),
    shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(8)),
    elevation:   4,
    minimumSize: const Size(double.infinity, 0),  // ocupa todo el ancho
  ),
  child: const Text('Acción crítica',
      style: TextStyle(fontWeight: FontWeight.bold)),
),
const Divider(height: 32),
```

#### Prueba esto
- Cambia `elevation: 4` a `elevation: 0` (plano) y a `elevation: 12` (muy elevado)
- Cambia `shape` a `const StadiumBorder()` — el botón se vuelve píldora
- Quita `minimumSize` — el botón vuelve a su ancho mínimo
- Prueba `TextButton.icon` y `OutlinedButton.icon` con el mismo patrón que `FilledButton.icon`
- Agrega `tooltip: 'Detiene todos los servicios'` al `IconButton`
- Cambia `onPressed: null` del "Desactivado" a `onPressed: () {}` — el botón se activa

---

### Bloque 4 — `Card` y `ListTile`

`Card` es una tarjeta con sombra. `ListTile` estructura una fila en tres zonas:
`leading` (izquierda), contenido central y `trailing` (derecha).

```dart
// Agrega a children: [ ... ]

Card(
  elevation: 3,
  margin: const EdgeInsets.only(bottom: 8),
  shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
  child: ListTile(
    leading:  const Icon(Icons.dns, color: Colors.indigo),
    title:    const Text('nginx-proxy'),
    subtitle: const Text('10.0.0.5 · 45ms'),
    trailing: const Icon(Icons.circle, color: Colors.green, size: 12),
    onTap:    () {},           // toda la fila queda tocable
  ),
),
Card(
  elevation: 1,
  child: ListTile(
    leading: CircleAvatar(
      backgroundColor: Colors.red.shade100,
      child: const Icon(Icons.cancel, color: Colors.red, size: 20),
    ),
    title:    const Text('backup-worker'),
    subtitle: const Text('sin respuesta · 10.0.0.30'),
    trailing: TextButton(onPressed: () {}, child: const Text('Ver')),
  ),
),
const Divider(height: 32),
```

#### Prueba esto
- Cambia `elevation: 3` a `elevation: 0` — tarjeta plana; a `elevation: 12` — sombra pronunciada
- Agrega `color: Colors.red.shade50` al `Card` — fondo coloreado
- Agrega `contentPadding: const EdgeInsets.symmetric(horizontal: 20, vertical: 12)` al `ListTile`
- En `ListTile` agrega `isThreeLine: true` y un `subtitle` largo — el tile crece verticalmente
- Agrega `Card(child: SwitchListTile(value: false, onChanged: (_){}, title: const Text('Modo mantenimiento')))` — variante con Switch integrado

---

### Bloque 5 — `Chip`

```dart
// Agrega a children: [ ... ]

Wrap(
  spacing: 8, runSpacing: 8,
  children: [
    const Chip(label: Text('nginx')),
    const Chip(
      avatar:          Icon(Icons.check, size: 16, color: Colors.white),
      label:           Text('TLS 1.3'),
      backgroundColor: Colors.green,
      labelStyle:      TextStyle(color: Colors.white, fontSize: 12),
    ),
    FilterChip(
      label:      const Text('HTTP/2'),
      selected:   true,
      onSelected: (_) {},
    ),
    ActionChip(
      label:     const Text('Ver logs'),
      avatar:    const Icon(Icons.open_in_new, size: 16),
      onPressed: () {},
    ),
  ],
),
const Divider(height: 32),
```

#### Prueba esto
- Cambia `selected: true` a `false` en `FilterChip` — el estilo de selección desaparece
- Agrega `deleteIcon: const Icon(Icons.close, size: 16)` y `onDeleted: () {}` al primer `Chip` — aparece la X
- Agrega `padding: const EdgeInsets.all(8)` al `Chip` — más espacio interior
- Cambia `backgroundColor: Colors.green` del segundo `Chip` a `Colors.blue`
- Prueba `InputChip` — combina `avatar`, `label`, `onDeleted` y `selected` en un solo widget

---

### Bloque 6 — Indicadores de progreso

```dart
// Agrega a children: [ ... ]

// ── Circular ──────────────────────────────────────────────────────────
Row(
  mainAxisAlignment: MainAxisAlignment.spaceEvenly,
  children: const [
    SizedBox(width: 48, height: 48,
      child: CircularProgressIndicator()),           // value: null → animación continua
    SizedBox(width: 48, height: 48,
      child: CircularProgressIndicator(
        value:       0.7,           // 70 %
        color:       Colors.green,
        strokeWidth: 6,
      )),
    SizedBox(width: 48, height: 48,
      child: CircularProgressIndicator(
        value:       0.3,
        color:       Colors.red,
        strokeWidth: 3,
        strokeCap:   StrokeCap.round,   // puntas redondeadas
      )),
  ],
),
const SizedBox(height: 16),

// ── Lineal ────────────────────────────────────────────────────────────
const LinearProgressIndicator(),                                  // indeterminado
const SizedBox(height: 8),
const LinearProgressIndicator(value: 0.6, color: Colors.indigo), // 60 %
const SizedBox(height: 8),
const LinearProgressIndicator(
  value:     1.0,
  color:     Colors.green,
  minHeight: 6,                     // barra más gruesa (default: 4)
),
const Divider(height: 32),
```

#### Prueba esto
- Cambia `value: 0.7` a `null` — pasa a animación continua
- Agrega `backgroundColor: Colors.grey.shade200` al `CircularProgressIndicator` — la pista se vuelve visible
- Cambia `minHeight: 6` a `minHeight: 12` en el `LinearProgressIndicator`
- Envuelve un `CircularProgressIndicator` en `Transform.scale(scale: 0.5, child: ...)` — escala sin cambiar `strokeWidth`

---

### Archivo completo: `lib/widgets/catalogo_basicos.dart`

Al terminar todos los bloques el archivo queda así:

```dart
import 'package:flutter/material.dart';

class CatalogoBasicos extends StatelessWidget {
  const CatalogoBasicos({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Widgets básicos')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [

          // ── Bloque 1: Text ────────────────────────────────────────────
          const Text(
            'nginx-proxy: En línea',
            style: TextStyle(
              fontSize:      20,
              fontWeight:    FontWeight.bold,
              color:         Colors.green,
              letterSpacing: 0.5,
            ),
          ),
          const SizedBox(height: 8),
          SizedBox(
            width: double.infinity,
            child: Text(
              'api-gateway-produccion-region-us-east → sin respuesta',
              textAlign: TextAlign.center,
              maxLines:  1,
              overflow:  TextOverflow.ellipsis,
            ),
          ),
          const SizedBox(height: 8),
          const Text.rich(
            TextSpan(children: [
              TextSpan(text: 'Estado: ',
                  style: TextStyle(fontWeight: FontWeight.w600)),
              TextSpan(text: 'CRÍTICO',
                  style: TextStyle(color: Colors.red, fontWeight: FontWeight.bold)),
              TextSpan(text: ' — última revisión hace 5 min',
                  style: TextStyle(color: Colors.grey, fontSize: 12)),
            ]),
          ),
          const SizedBox(height: 8),
          const SelectableText('10.0.0.12:5432',
              style: TextStyle(fontFamily: 'monospace', fontSize: 14)),
          const Divider(height: 32),

          // ── Bloque 2: Icon ────────────────────────────────────────────
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceEvenly,
            children: const [
              Icon(Icons.check_circle,  size: 40, color: Colors.green),
              Icon(Icons.cancel,        size: 40, color: Colors.red),
              Icon(Icons.warning_amber, size: 40, color: Colors.orange),
              Icon(Icons.dns,           size: 40, color: Colors.indigo),
              Icon(Icons.wifi_off,      size: 40, color: Colors.grey),
            ],
          ),
          const SizedBox(height: 8),
          const Icon(Icons.settings, size: 24, color: Colors.blueGrey,
              semanticLabel: 'Configuración'),
          const Divider(height: 32),

          // ── Bloque 3: Botones ─────────────────────────────────────────
          Wrap(
            spacing: 8, runSpacing: 8,
            children: [
              ElevatedButton(onPressed: () {}, child: const Text('ElevatedButton')),
              FilledButton(  onPressed: () {}, child: const Text('FilledButton')),
              OutlinedButton(onPressed: () {}, child: const Text('OutlinedButton')),
              TextButton(    onPressed: () {}, child: const Text('TextButton')),
              ElevatedButton(onPressed: null,  child: const Text('Desactivado')),
            ],
          ),
          const SizedBox(height: 12),
          Wrap(
            spacing: 8, runSpacing: 8,
            children: [
              ElevatedButton.icon(
                onPressed: () {},
                icon:  const Icon(Icons.refresh, size: 18),
                label: const Text('Reiniciar'),
              ),
              FilledButton.icon(
                onPressed: () {},
                icon:  const Icon(Icons.stop, size: 18),
                label: const Text('Detener'),
              ),
              IconButton(
                onPressed: () {},
                icon:     const Icon(Icons.settings),
                color:    Colors.indigo,
                iconSize: 28,
              ),
            ],
          ),
          const SizedBox(height: 12),
          ElevatedButton(
            onPressed: () {},
            style: ElevatedButton.styleFrom(
              backgroundColor: Colors.red.shade600,
              foregroundColor: Colors.white,
              padding:     const EdgeInsets.symmetric(horizontal: 32, vertical: 14),
              shape: RoundedRectangleBorder(
                  borderRadius: BorderRadius.circular(8)),
              elevation:   4,
              minimumSize: const Size(double.infinity, 0),
            ),
            child: const Text('Acción crítica',
                style: TextStyle(fontWeight: FontWeight.bold)),
          ),
          const Divider(height: 32),

          // ── Bloque 4: Card y ListTile ─────────────────────────────────
          Card(
            elevation: 3,
            margin: const EdgeInsets.only(bottom: 8),
            shape: RoundedRectangleBorder(
                borderRadius: BorderRadius.circular(12)),
            child: ListTile(
              leading:  const Icon(Icons.dns, color: Colors.indigo),
              title:    const Text('nginx-proxy'),
              subtitle: const Text('10.0.0.5 · 45ms'),
              trailing: const Icon(Icons.circle, color: Colors.green, size: 12),
              onTap:    () {},
            ),
          ),
          Card(
            elevation: 1,
            child: ListTile(
              leading: CircleAvatar(
                backgroundColor: Colors.red.shade100,
                child: const Icon(Icons.cancel, color: Colors.red, size: 20),
              ),
              title:    const Text('backup-worker'),
              subtitle: const Text('sin respuesta · 10.0.0.30'),
              trailing: TextButton(
                  onPressed: () {}, child: const Text('Ver')),
            ),
          ),
          const Divider(height: 32),

          // ── Bloque 5: Chip ────────────────────────────────────────────
          Wrap(
            spacing: 8, runSpacing: 8,
            children: [
              const Chip(label: Text('nginx')),
              const Chip(
                avatar:          Icon(Icons.check, size: 16, color: Colors.white),
                label:           Text('TLS 1.3'),
                backgroundColor: Colors.green,
                labelStyle: TextStyle(color: Colors.white, fontSize: 12),
              ),
              FilterChip(
                label: const Text('HTTP/2'),
                selected: true, onSelected: (_) {},
              ),
              ActionChip(
                label:     const Text('Ver logs'),
                avatar:    const Icon(Icons.open_in_new, size: 16),
                onPressed: () {},
              ),
            ],
          ),
          const Divider(height: 32),

          // ── Bloque 6: Indicadores de progreso ────────────────────────
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceEvenly,
            children: const [
              SizedBox(width: 48, height: 48,
                  child: CircularProgressIndicator()),
              SizedBox(width: 48, height: 48,
                  child: CircularProgressIndicator(
                    value: 0.7, color: Colors.green, strokeWidth: 6)),
              SizedBox(width: 48, height: 48,
                  child: CircularProgressIndicator(
                    value: 0.3, color: Colors.red,
                    strokeWidth: 3, strokeCap: StrokeCap.round)),
            ],
          ),
          const SizedBox(height: 16),
          const LinearProgressIndicator(),
          const SizedBox(height: 8),
          const LinearProgressIndicator(value: 0.6, color: Colors.indigo),
          const SizedBox(height: 8),
          const LinearProgressIndicator(
              value: 1.0, color: Colors.green, minHeight: 6),
        ],
      ),
    );
  }
}
```

---

## Paso 2 — StatelessWidget con parámetros

Los parámetros son siempre `final`. El `const` permite que Flutter
**reutilice** el widget si los datos no cambiaron.

### Archivo: `lib/widgets/etiqueta.dart`

Crea el archivo (la carpeta ya existe del Paso 1b):

```dart
import 'package:flutter/material.dart';

class Etiqueta extends StatelessWidget {
  final String  texto;
  final Color   color;
  final double  fontSize;        // parámetro con valor por defecto
  final bool    relleno;         // controla si el fondo tiene opacidad alta

  const Etiqueta({
    super.key,
    required this.texto,
    required this.color,
    this.fontSize = 13,          // opcional — no necesita required
    this.relleno  = false,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
      decoration: BoxDecoration(
        color:        color.withOpacity(relleno ? 0.3 : 0.12),
        border:       Border.all(color: color, width: 1.5),
        borderRadius: BorderRadius.circular(20),
        boxShadow: [
          BoxShadow(
            color:      color.withOpacity(0.2),
            blurRadius: 6,
            offset:     const Offset(0, 2),
          ),
        ],
      ),
      child: Text(
        texto,
        style: TextStyle(
          color:      color,
          fontWeight: FontWeight.w600,
          fontSize:   fontSize,
        ),
      ),
    );
  }
}
```

### Agrega al `main.dart`

1. Import al inicio:
```dart
import 'widgets/etiqueta.dart';
```

2. Nuevo case en el switch:
```dart
3 => const Scaffold(
      body: Center(
        child: Wrap(
          spacing:    12,
          runSpacing: 8,
          children: [
            Etiqueta(texto: 'Activo',    color: Colors.green),
            Etiqueta(texto: 'Error',     color: Colors.red,    relleno: true),
            Etiqueta(texto: 'En espera', color: Colors.orange),
            Etiqueta(texto: 'Crítico',   color: Colors.red,    fontSize: 16, relleno: true),
            Etiqueta(texto: 'Info',      color: Colors.blue,   fontSize: 11),
          ],
        ),
      ),
    ),
```

3. Cambia `const int paso = 3;` y guarda.

### Prueba esto

- Cambia `BorderRadius.circular(20)` a `circular(4)` — de píldora a rectángulo; prueba `circular(0)` para esquinas rectas
- Agrega `shape: BoxShape.rectangle` (ya es el valor por defecto). Prueba solo con `shape: BoxShape.circle` y un tamaño fijo `width: 36, height: 36` — ¿qué pasa con el texto?
- Cambia el `Border.all` por `Border.only(bottom: BorderSide(color: color, width: 2))` — subrayado inferior
- Agrega `gradient` al `BoxDecoration` (reemplaza `color`):
  ```dart
  gradient: LinearGradient(colors: [color.withOpacity(0.1), color.withOpacity(0.25)]),
  ```
- Prueba pasar `relleno: true` a todas las etiquetas — compara la diferencia visual con `false`
- Agrega `IconData? icono` como parámetro y muéstralo antes del texto con `Icon(icono, size: fontSize, color: color)`

---

## Paso 3 — StatefulWidget, setState y cambio de estatus

Un `StatefulWidget` **puede cambiar** a lo largo del tiempo.
El estado (`_activo`, `_reinicios`) controla directamente cómo se ve la UI.

### Archivo: `lib/widgets/servicio_estado.dart`

Este widget muestra los seis patrones de cambio de estatus en un solo lugar:

```dart
import 'package:flutter/material.dart';

class ServicioEstado extends StatefulWidget {
  final String nombre;
  const ServicioEstado({super.key, required this.nombre});

  @override
  State<ServicioEstado> createState() => _ServicioEstadoState();
}

class _ServicioEstadoState extends State<ServicioEstado> {
  bool _activo    = true;
  int  _reinicios = 0;

  static const int _maxReinicios = 3;

  void _toggle() {
    setState(() {              // notifica a Flutter → rebuild
      _activo = !_activo;
      if (_activo) _reinicios++;
    });
  }

  @override
  Widget build(BuildContext context) {
    final enLimite = _reinicios >= _maxReinicios;

    return Padding(
      padding: const EdgeInsets.all(24),
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [

          // ── Patrón 1: Ícono + color condicional ─────────────────
          Icon(
            _activo ? Icons.check_circle : Icons.cancel,
            size:  72,
            color: _activo ? Colors.green : Colors.red,
          ),
          const SizedBox(height: 8),

          Text(widget.nombre,
              style: const TextStyle(fontWeight: FontWeight.bold, fontSize: 20)),

          // ── Patrón 2: Texto condicional ──────────────────────────
          Text(
            _activo ? 'En línea' : 'Fuera de línea',
            style: TextStyle(
              fontSize:   15,
              fontWeight: FontWeight.w600,
              color:      _activo ? Colors.green.shade700 : Colors.red.shade700,
            ),
          ),
          const SizedBox(height: 16),

          // ── Patrón 3: Widget que aparece / desaparece ────────────
          if (!_activo)
            Container(
              margin:     const EdgeInsets.only(bottom: 16),
              padding:    const EdgeInsets.symmetric(horizontal: 14, vertical: 8),
              decoration: BoxDecoration(
                color:        Colors.red.shade50,
                borderRadius: BorderRadius.circular(8),
                border:       Border.all(color: Colors.red.shade300),
              ),
              child: const Row(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Icon(Icons.warning_amber, color: Colors.red, size: 16),
                  SizedBox(width: 6),
                  Text('Requiere atención',
                      style: TextStyle(color: Colors.red, fontSize: 13)),
                ],
              ),
            ),

          // ── Patrón 4: Botón con texto, color y estado dinámicos ──
          FilledButton.icon(
            onPressed: enLimite ? null : _toggle,    // null = desactivado
            icon: Icon(_activo ? Icons.stop : Icons.play_arrow),
            label: Text(_activo ? 'Detener servicio' : 'Iniciar servicio'),
            style: FilledButton.styleFrom(
              backgroundColor: _activo ? Colors.red.shade600 : Colors.green.shade600,
            ),
          ),
          const SizedBox(height: 12),

          // ── Patrón 5: Opacidad condicional ───────────────────────
          Opacity(
            opacity: enLimite ? 0.4 : 1.0,
            child: Text(
              'Reinicios: $_reinicios / $_maxReinicios',
              style: TextStyle(
                fontSize: 13,
                color:    enLimite ? Colors.red : Colors.grey.shade600,
              ),
            ),
          ),

          // ── Patrón 6: Widget condicional por otro estado ─────────
          if (enLimite)
            Padding(
              padding: const EdgeInsets.only(top: 8),
              child: Text(
                'Límite de reinicios alcanzado',
                style: TextStyle(
                    fontSize: 12, color: Colors.red.shade700, fontWeight: FontWeight.bold),
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
import 'widgets/servicio_estado.dart';
```

```dart
4 => const Scaffold(
      body: Center(
        child: ServicioEstado(nombre: 'nginx-proxy'),
      ),
    ),
```

Cambia `paso = 4` y guarda.

> `setState(() { ... })` → Flutter llama `build()` de nuevo con los nuevos valores.
> Las variables de estado viven en el `State`, no en el widget.

### Prueba esto

- Cambia `Icons.check_circle` por `Icons.wifi` y `Icons.wifi_off` — misma lógica, diferente semántica visual
- Agrega `fontStyle: _activo ? FontStyle.normal : FontStyle.italic` al texto "Fuera de línea"
- Cambia el `_maxReinicios` de `3` a `1` — el botón se bloquea al primer reinicio
- Modifica `Opacity(opacity: enLimite ? 0.4 : 1.0, ...)` a `0.1` cuando está en límite — casi invisible
- Agrega un tercer estado `String _nivel = 'normal'` que cambie a `'warning'` después de 1 reinicio y a `'critico'` después de 2, y muestra un color diferente en el ícono para cada nivel
- Agrega un botón `TextButton` que llame `setState(() { _activo = true; _reinicios = 0; })` para reiniciar todo
- Cambia `FilledButton.icon` por `ElevatedButton.icon` — la lógica de `enLimite ? null : _toggle` no cambia

---

## Paso 3b — Acceder a parámetros desde el State

Usa `widget.` para leer los parámetros del `StatefulWidget` dentro del `State`.
En el `main.dart` este paso usa el número `4`.

### Archivo: `lib/widgets/contador_limitado.dart`

```dart
import 'package:flutter/material.dart';

class ContadorLimitado extends StatefulWidget {
  final String       etiqueta;
  final int          limite;
  final Color        color;          // parámetro extra para demostrar widget.param
  final VoidCallback? onLimite;      // callback opcional — se llama al alcanzar el límite

  const ContadorLimitado({
    super.key,
    required this.etiqueta,
    this.limite  = 10,
    this.color   = Colors.indigo,
    this.onLimite,
  });

  @override
  State<ContadorLimitado> createState() => _ContadorLimitadoState();
}

class _ContadorLimitadoState extends State<ContadorLimitado> {
  int _valor = 0;

  void _incrementar() {
    if (_valor >= widget.limite) return;    // defensa extra
    setState(() => _valor++);
    if (_valor == widget.limite) {
      widget.onLimite?.call();              // notifica al padre si registró un callback
    }
  }

  @override
  Widget build(BuildContext context) {
    final enLimite  = _valor >= widget.limite;
    final progreso  = _valor / widget.limite;   // 0.0 → 1.0

    return Column(
      mainAxisSize: MainAxisSize.min,
      children: [
        Text(widget.etiqueta,                               // ← widget.etiqueta
            style: TextStyle(color: widget.color, fontWeight: FontWeight.w600)),

        const SizedBox(height: 4),

        // Barra de progreso que refleja el estado
        LinearProgressIndicator(
          value:           progreso,
          color:           enLimite ? Colors.red : widget.color,   // ← widget.color
          backgroundColor: widget.color.withOpacity(0.15),
        ),

        const SizedBox(height: 4),

        Text(
          '$_valor / ${widget.limite}',                      // ← widget.limite
          style: TextStyle(
            fontSize:   28,
            fontWeight: FontWeight.bold,
            color:      enLimite ? Colors.red : widget.color,
          ),
        ),

        Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            FilledButton(
              onPressed: enLimite ? null : _incrementar,    // null = desactivado
              child: const Text('Sumar'),
            ),
            const SizedBox(width: 8),
            TextButton(
              onPressed: () => setState(() => _valor = 0),  // reiniciar
              child: const Text('Reset'),
            ),
          ],
        ),

        if (enLimite)
          Text('Límite alcanzado',
              style: TextStyle(fontSize: 12, color: Colors.red.shade700)),
      ],
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'widgets/contador_limitado.dart';
```

```dart
5 => Scaffold(                               // Paso 3b
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ContadorLimitado(
              etiqueta: 'Intentos de login',
              limite:   3,
              color:    Colors.red,
              onLimite: () => debugPrint('¡Cuenta bloqueada!'),
            ),
            const SizedBox(height: 40),
            ContadorLimitado(
              etiqueta: 'Conexiones activas',
              limite:   10,
              color:    Colors.indigo,
            ),
          ],
        ),
      ),
    ),
```

Cambia `paso = 5` y guarda.

### Prueba esto

- Cambia `limite: 3` a `limite: 1` — el botón se bloquea al primer toque. `widget.limite` controla el comportamiento sin cambiar el widget
- Agrega `String textoBoton = 'Sumar'` como parámetro y úsalo en el `child` del botón — pasa `textoBoton: 'Intentar'` desde el case
- Cambia `color: Colors.red` a `color: Colors.deepPurple` — `widget.color` se propaga al texto, la barra y el botón
- Agrega `int pasoIncremento = 1` como parámetro y cambia `_valor++` por `_valor += widget.pasoIncremento`
- Prueba `onLimite: () => debugPrint('¡Cuenta bloqueada!')` y observa la consola al alcanzar el límite

---

## Paso 4 — Ciclo de vida con Timer

`initState` → `build` → `[setState → build]*` → `dispose`.
Los recursos creados en `initState` **deben liberarse** en `dispose`.

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
  late Timer _timer;      // late — se asigna en initState, antes no existe
  int  _segundos = 0;
  bool _pausado  = false;

  @override
  void initState() {
    super.initState();    // ← siempre primero
    _iniciarTimer();
  }

  void _iniciarTimer() {
    _timer = Timer.periodic(const Duration(seconds: 1), (_) {
      if (!mounted) return;   // ← protege setState en callbacks
      setState(() => _segundos++);
    });
  }

  void _togglePausa() {
    setState(() {
      _pausado = !_pausado;
      if (_pausado) {
        _timer.cancel();      // pausa: cancela el timer actual
      } else {
        _iniciarTimer();      // reanuda: crea un timer nuevo
      }
    });
  }

  @override
  void dispose() {
    _timer.cancel();          // ← SIEMPRE liberar en dispose
    super.dispose();          // ← siempre al final
  }

  String get _formato {
    final h = _segundos ~/ 3600;
    final m = (_segundos % 3600) ~/ 60;
    final s = _segundos % 60;
    return '$h:${m.toString().padLeft(2, '0')}:${s.toString().padLeft(2, '0')}';
  }

  // Color cambia según el tiempo transcurrido
  Color get _colorTiempo {
    if (_segundos > 60) return Colors.red;
    if (_segundos > 30) return Colors.orange;
    return Colors.green;
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Text(
          _formato,
          style: TextStyle(
            fontSize:   40,
            fontFamily: 'monospace',
            fontWeight: FontWeight.bold,
            color:      _colorTiempo,         // cambia automáticamente con el tiempo
          ),
        ),
        const SizedBox(height: 16),
        Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            FilledButton.icon(
              onPressed: _togglePausa,
              icon:  Icon(_pausado ? Icons.play_arrow : Icons.pause),
              label: Text(_pausado ? 'Reanudar' : 'Pausar'),
            ),
            const SizedBox(width: 8),
            TextButton(
              onPressed: () => setState(() {
                _timer.cancel();
                _segundos = 0;
                _pausado  = false;
                _iniciarTimer();
              }),
              child: const Text('Reiniciar'),
            ),
          ],
        ),
        const SizedBox(height: 8),
        Text(
          _pausado ? 'Pausado' : 'Corriendo',
          style: TextStyle(fontSize: 12, color: Colors.grey.shade600),
        ),
      ],
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'widgets/reloj.dart';
```

```dart
6 => Scaffold(                              // Paso 4
      appBar: AppBar(title: const Text('Cronómetro')),
      body: const Center(child: Reloj()),
    ),
```

Cambia `paso = 6` y guarda.

> **`mounted`** — si el widget se destruye mientras un `Timer` corre,
> `setState()` sobre un widget sin árbol lanza un error. `if (!mounted) return`
> evita ese caso. En Dart 3+ también existe `context.mounted`.

### Prueba esto

- Cambia `Duration(seconds: 1)` a `Duration(milliseconds: 100)` — el reloj corre 10× más rápido; observa que pausa/reanuda sigue funcionando igual
- Agrega un segundo color condicional: `_segundos > 120 ? Colors.deepPurple : _colorTiempo`
- Prueba comentar `_timer.cancel()` en `dispose()` — ejecuta en modo debug y observa el warning de fuga de memoria en consola
- Agrega `int _vueltas = 0` y un botón "Vuelta" que guarde `_segundos` en una lista `_tiemposVuelta` y muestre la última vuelta debajo del reloj
- Cambia `late Timer _timer` por `Timer? _timer` y ajusta el código para que sea nullable — practica el manejo de null safety

---

## Paso 5 — BuildContext

`BuildContext` es la referencia del widget en el árbol.
Permite acceder al tema, tamaño de pantalla y datos de ancestros.

### Archivo: `lib/screens/pantalla_contexto.dart`

Crea la carpeta `lib/screens/` y dentro el archivo:

```dart
import 'package:flutter/material.dart';

class PantallaContexto extends StatelessWidget {
  const PantallaContexto({super.key});

  @override
  Widget build(BuildContext context) {
    // ── Tema ──────────────────────────────────────────────────────
    final tema    = Theme.of(context);
    final colores = tema.colorScheme;

    // ── Pantalla ──────────────────────────────────────────────────
    final tamanio   = MediaQuery.sizeOf(context);
    final esMovil   = tamanio.width < 600;
    final esRetrato = MediaQuery.orientationOf(context) == Orientation.portrait;

    return Scaffold(
      backgroundColor: colores.surface,
      appBar: AppBar(
        backgroundColor: colores.primaryContainer,
        foregroundColor: colores.onPrimaryContainer,
        title: Text(
          'Pantalla ${esMovil ? "móvil" : "tablet"} · ${esRetrato ? "retrato" : "paisaje"}',
          style: tema.textTheme.titleMedium,
        ),
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          // ── Información de pantalla ────────────────────────────
          _Seccion(
            titulo: 'Pantalla',
            items: [
              'Ancho:        ${tamanio.width.toStringAsFixed(0)} px',
              'Alto:         ${tamanio.height.toStringAsFixed(0)} px',
              'Pixel ratio:  ${MediaQuery.devicePixelRatioOf(context).toStringAsFixed(1)}',
              'Orientación:  ${MediaQuery.orientationOf(context).name}',
            ],
          ),
          const SizedBox(height: 16),

          // ── Colores del tema ───────────────────────────────────
          _Seccion(titulo: 'colorScheme', items: []),
          Wrap(
            spacing: 8, runSpacing: 8,
            children: [
              _ChipColor(nombre: 'primary',          color: colores.primary),
              _ChipColor(nombre: 'primaryContainer', color: colores.primaryContainer),
              _ChipColor(nombre: 'secondary',        color: colores.secondary),
              _ChipColor(nombre: 'surface',          color: colores.surface),
              _ChipColor(nombre: 'error',            color: colores.error),
            ],
          ),
          const SizedBox(height: 16),

          // ── Tipografía del tema ────────────────────────────────
          _Seccion(titulo: 'textTheme', items: []),
          Text('displaySmall',  style: tema.textTheme.displaySmall),
          Text('headlineMedium', style: tema.textTheme.headlineMedium),
          Text('titleLarge',    style: tema.textTheme.titleLarge),
          Text('bodyLarge',     style: tema.textTheme.bodyLarge),
          Text('bodyMedium',    style: tema.textTheme.bodyMedium),
          Text('labelSmall',    style: tema.textTheme.labelSmall),
        ],
      ),
    );
  }
}

// ── Widgets auxiliares privados — solo usados en este archivo ────────

class _Seccion extends StatelessWidget {
  final String       titulo;
  final List<String> items;
  const _Seccion({required this.titulo, required this.items});

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(titulo,
            style: Theme.of(context).textTheme.labelLarge?.copyWith(
                color: Theme.of(context).colorScheme.primary)),
        const Divider(),
        for (final item in items)
          Padding(
            padding: const EdgeInsets.symmetric(vertical: 2),
            child: Text(item, style: Theme.of(context).textTheme.bodyMedium),
          ),
      ],
    );
  }
}

class _ChipColor extends StatelessWidget {
  final String nombre;
  final Color  color;
  const _ChipColor({required this.nombre, required this.color});

  @override
  Widget build(BuildContext context) {
    final luminancia = color.computeLuminance();
    final textoColor = luminancia > 0.4 ? Colors.black87 : Colors.white;
    return Container(
      padding:    const EdgeInsets.symmetric(horizontal: 10, vertical: 6),
      decoration: BoxDecoration(color: color, borderRadius: BorderRadius.circular(8)),
      child: Text(nombre, style: TextStyle(color: textoColor, fontSize: 11, fontWeight: FontWeight.w600)),
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'screens/pantalla_contexto.dart';
```

```dart
7 => const PantallaContexto(),    // Paso 5 — ya tiene su propio Scaffold
```

Cambia `paso = 7` y guarda.

Para probar con tema diferente, modifica el `MaterialApp` en `main.dart`:
```dart
void main() => runApp(MaterialApp(
  debugShowCheckedModeBanner: false,
  theme: ThemeData(
    colorScheme:  ColorScheme.fromSeed(
      seedColor:  Colors.teal,          // ← cambia aquí
      brightness: Brightness.light,     // ← Brightness.dark para modo oscuro
    ),
    useMaterial3: true,
  ),
  home: switch (paso) { ... },
));
```

### Prueba esto

- Cambia `seedColor` a `Colors.deepPurple`, `Colors.teal`, `Colors.deepOrange` — toda la paleta de colores se regenera automáticamente
- Cambia `Brightness.light` a `Brightness.dark` — modo oscuro sin cambiar ningún widget
- Reemplaza `colores.primaryContainer` por `colores.secondaryContainer` en el `AppBar` — otra variante del tema
- Agrega `MediaQuery.paddingOf(context).top` a los datos mostrados — muestra el padding del notch/statusbar
- Cambia `MediaQuery.devicePixelRatioOf(context)` — en emuladores suele ser 2.0 o 3.0; en un dispositivo real puede variar
- Agrega `Scaffold.of(context).showSnackBar(...)` en un botón para ver un acceso más de context

---

## Paso 6 — Composición de widgets

En Flutter la UI se construye **componiendo widgets pequeños reutilizables**.
Cada widget hace **una sola cosa** bien.

### Archivo: `lib/widgets/indicador.dart`

```dart
import 'package:flutter/material.dart';

class Indicador extends StatelessWidget {
  final String  label;
  final String  valor;          // String para mayor flexibilidad: '8', '4.2 GB', '99%'
  final Color   color;
  final String? subtitulo;      // línea adicional opcional
  final IconData? icono;        // ícono opcional antes del valor

  const Indicador({
    super.key,
    required this.label,
    required this.valor,
    required this.color,
    this.subtitulo,
    this.icono,
  });

  @override
  Widget build(BuildContext context) {
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      mainAxisSize:       MainAxisSize.min,
      children: [
        // Valor principal — con ícono opcional
        Row(
          mainAxisSize: MainAxisSize.min,
          children: [
            if (icono != null) ...[
              Icon(icono, size: 22, color: color),
              const SizedBox(width: 4),
            ],
            Text(
              valor,
              style: TextStyle(
                  fontSize: 28, fontWeight: FontWeight.bold, color: color),
            ),
          ],
        ),
        Text(label,
            style: TextStyle(fontSize: 12, color: Colors.grey.shade600)),
        if (subtitulo != null)
          Text(subtitulo!,
              style: TextStyle(fontSize: 10, color: Colors.grey.shade400)),
      ],
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'widgets/indicador.dart';
```

```dart
8 => Scaffold(                             // Paso 6
      body: Center(
        child: Wrap(
          spacing:    32,
          runSpacing: 24,
          alignment:  WrapAlignment.center,
          children: const [
            Indicador(label: 'Servidores activos', valor: '8',
                      color: Colors.green, icono: Icons.dns),
            Indicador(label: 'Alertas críticas',   valor: '2',
                      color: Colors.red,   icono: Icons.warning_amber,
                      subtitulo: 'Requieren atención'),
            Indicador(label: 'Tráfico',            valor: '4.2 GB',
                      color: Colors.indigo),
            Indicador(label: 'Uptime',             valor: '99.8%',
                      color: Colors.teal, subtitulo: 'Últimos 30 días'),
          ],
        ),
      ),
    ),
```

Cambia `paso = 8` y guarda.

> Widgets con `_` al inicio (`_Seccion`, `_ChipColor`) son **privados al archivo**.
> No se pueden importar desde otros archivos — convención para widgets auxiliares.

### Prueba esto

- Cambia `valor` de `String` a `int` y observa dónde rompe la llamada — luego vuelve a `String` y nota la flexibilidad (`'4.2 GB'`, `'99%'`, `'8'` con un solo tipo)
- Agrega `double opacidad = 1.0` como parámetro y envuelve el `Column` en `Opacity(opacity: opacidad, ...)`
- Pasa `icono: Icons.cloud` a uno de los `Indicador` — el ícono aparece sin cambiar el widget
- Agrega un segundo `Indicador` con el mismo `label` pero diferente `color` — Flutter los trata como widgets independientes
- Cambia `crossAxisAlignment: CrossAxisAlignment.start` a `CrossAxisAlignment.center` — el valor y la etiqueta se centran juntos

---

## `main.dart` completo — referencia

Al terminar todos los pasos, el archivo queda así.
Cambia `paso` en cualquier momento para revisar cualquier ejemplo:

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'widgets/catalogo_basicos.dart';
import 'widgets/etiqueta.dart';
import 'widgets/servicio_estado.dart';
import 'widgets/contador_limitado.dart';
import 'widgets/reloj.dart';
import 'widgets/indicador.dart';
import 'screens/pantalla_contexto.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1   StatelessWidget mínimo                             │
// │  2  Paso 1b  Widgets básicos — catálogo                        │
// │  3  Paso 2   StatelessWidget con parámetros                     │
// │  4  Paso 3   StatefulWidget / setState / cambio de estatus      │
// │  5  Paso 3b  Parámetros en StatefulWidget                       │
// │  6  Paso 4   Ciclo de vida con Timer                            │
// │  7  Paso 5   BuildContext                                        │
// │  8  Paso 6   Composición de widgets                             │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 1;

void main() => runApp(MaterialApp(
  debugShowCheckedModeBanner: false,
  theme: ThemeData(
    colorScheme:  ColorScheme.fromSeed(seedColor: Colors.indigo),
    useMaterial3: true,
  ),
  home: switch (paso) {
    1 => const Scaffold(body: Center(child: Saludo())),
    2 => const CatalogoBasicos(),                       // Paso 1b
    3 => const Scaffold(
          body: Center(
            child: Wrap(
              spacing: 12, runSpacing: 8,
              children: [
                Etiqueta(texto: 'Activo',    color: Colors.green),
                Etiqueta(texto: 'Error',     color: Colors.red,    relleno: true),
                Etiqueta(texto: 'En espera', color: Colors.orange),
                Etiqueta(texto: 'Crítico',   color: Colors.red,    fontSize: 16, relleno: true),
                Etiqueta(texto: 'Info',      color: Colors.blue,   fontSize: 11),
              ],
            ),
          ),
        ),
    4 => const Scaffold(
          body: Center(child: ServicioEstado(nombre: 'nginx-proxy')),
        ),
    5 => Scaffold(                                     // Paso 3b
          body: Center(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: const [
                ContadorLimitado(
                  etiqueta: 'Intentos de login',
                  limite:   3,
                  color:    Colors.red,
                ),
                SizedBox(height: 40),
                ContadorLimitado(
                  etiqueta: 'Conexiones activas',
                  limite:   10,
                  color:    Colors.indigo,
                ),
              ],
            ),
          ),
        ),
    6 => Scaffold(                                     // Paso 4
          appBar: AppBar(title: const Text('Cronómetro')),
          body: const Center(child: Reloj()),
        ),
    7 => const PantallaContexto(),                    // Paso 5 — ya tiene Scaffold
    8 => Scaffold(                                     // Paso 6
          body: Center(
            child: Wrap(
              spacing: 32, runSpacing: 24,
              alignment: WrapAlignment.center,
              children: const [
                Indicador(label: 'Servidores activos', valor: '8',
                          color: Colors.green, icono: Icons.dns),
                Indicador(label: 'Alertas críticas',   valor: '2',
                          color: Colors.red,   icono: Icons.warning_amber,
                          subtitulo: 'Requieren atención'),
                Indicador(label: 'Tráfico',            valor: '4.2 GB',
                          color: Colors.indigo),
                Indicador(label: 'Uptime',             valor: '99.8%',
                          color: Colors.teal, subtitulo: 'Últimos 30 días'),
              ],
            ),
          ),
        ),
    _ => Scaffold(body: Center(child: Text('Paso $paso no definido'))),
  },
));

// ─── Paso 1 — vive en main.dart ──────────────────────────────────────
class Saludo extends StatelessWidget {
  const Saludo({super.key});
  @override
  Widget build(BuildContext context) =>
      const Text('Hola Flutter', style: TextStyle(fontSize: 32));
}
```

---

## Proyecto — Monitor de Infraestructura

Proyecto separado que combina todos los pasos en una app real.
Reemplaza el contenido de `lib/main.dart` con el del proyecto.

```
modulo06_widgets/
└── lib/
    ├── main.dart                          ← entrada del proyecto
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

  bool get esCritico => cpu > 85 || ram > 90;
}
```

---

### `lib/widgets/tarjeta_metrica.dart`

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
                Icon(servidor.activo ? Icons.circle : Icons.circle_outlined,
                    color: color, size: 12),
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
  late Timer     _timer;
  List<Servidor> _servidores = [];
  bool           _cargando   = true;
  int            _ciclo      = 0;

  @override
  void initState() {
    super.initState();
    _cargarDatos();
    _timer = Timer.periodic(const Duration(seconds: 3), (_) => _cargarDatos());
  }

  @override
  void dispose() {
    _timer.cancel();
    super.dispose();
  }

  void _cargarDatos() {
    Future.delayed(const Duration(milliseconds: 400), () {
      if (!mounted) return;
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
                        titulo:    'Alertas', valor: '$alertas',
                        icono:     Icons.warning_amber,
                        colorIcono: alertas > 0 ? Colors.orange : Colors.green,
                        subtitulo:  alertas > 0 ? 'Requieren atención' : null,
                      )),
                    ],
                  ),
                ),
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

### `lib/main.dart` (proyecto)

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

---

## Guía rápida de imports

```dart
// Desde lib/main.dart → lib/widgets/
import 'widgets/etiqueta.dart';
import 'widgets/servicio_estado.dart';
import 'widgets/contador_limitado.dart';
import 'widgets/reloj.dart';
import 'widgets/indicador.dart';

// Desde lib/main.dart → lib/screens/
import 'screens/pantalla_contexto.dart';

// Desde lib/screens/ → lib/widgets/
import '../widgets/tarjeta_metrica.dart';

// Desde lib/screens/ o lib/widgets/ → lib/models/
import '../models/servidor.dart';
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
   Al tocar un botón avanza cíclicamente. El activo brilla (`Opacity(opacity: 1.0)`),
   los demás están opacos (`0.25`). Texto asociado: "STOP", "PRECAUCIÓN", "GO".
   Crea `lib/widgets/semaforo.dart` e impórtalo en `main.dart`.

2. **Cronómetro de sesión** — Basado en `Reloj` del Paso 4. Agrega: registro de vueltas
   (lista de tiempos), botón "Vuelta" que guarda el tiempo actual, y color que cambia
   en 3 rangos de tiempo.

3. **Tarjeta expandible** — Nombre de dispositivo de red. Al tocar → se expande
   mostrando IP, MAC, estado y latencia. Al tocar de nuevo → colapsa.
   Usa `bool _expandido` y `AnimatedCrossFade`. Crea `lib/widgets/tarjeta_dispositivo.dart`.

4. **Layout responsivo** — Con `MediaQuery.orientationOf(context)`:
   `TarjetaMetrica` en columna en retrato, en fila en paisaje.

---

## Resumen

- En Flutter **todo es un widget** — texto, layouts, botones, tema. La UI es un árbol de widgets anidados.
- `StatelessWidget` solo muestra datos recibidos. Parámetros siempre `final`. Usa `const` cuando sea posible.
- `StatefulWidget` = widget (inmutable, configura) + `State` (mutable, almacena datos que cambian).
- `setState(() { ... })` notifica a Flutter — llama `build()` de nuevo con los nuevos valores.
- El **cambio de estatus** se expresa con operadores ternarios (`_activo ? A : B`), `if` dentro de `children`, `onPressed: null` para deshabilitar, y `Opacity` para suavizar transiciones.
- `initState()` → **una vez** al crear. Inicializar timers, streams, carga de datos.
- `dispose()` → **obligatorio** para cancelar `Timer`, cerrar `StreamSubscription`, liberar `AnimationController`.
- `mounted` protege `setState()` en callbacks asíncronos: `if (!mounted) return;`.
- `BuildContext` da acceso al tema (`Theme.of(context)`), tamaño (`MediaQuery.sizeOf(context)`) y navegación.
- La composición de widgets pequeños y reutilizables es el patrón fundamental de Flutter.

---

> **Siguiente →** Módulo 7: Layouts — `Column`, `Row`, `Stack`, `Container`, `Expanded` y más.
