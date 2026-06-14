# Tutorial Flutter — Módulo 7
## Layouts: `Column`, `Row`, `Stack`, `Container`, `Wrap` y más

---

## El sistema de layout de Flutter

Flutter usa un sistema de **constraints** (restricciones):
el padre da al hijo las restricciones de tamaño, el hijo decide
su tamaño dentro de esas restricciones, y el padre lo posiciona.

```
Padre → "Tienes entre 0 y 300px de ancho"
  ↓
Hijo  → "Voy a usar 200px"
  ↓
Padre → "Bien, te coloco en esta posición"
```

Tres reglas que evitan el 90% de los errores de layout:

```
1. Un widget acotado (bounded)      → tiene tamaño máximo definido
2. Un widget no acotado (unbounded) → puede ser infinito (ListView, Column)
3. Unconstrained dentro de unbounded → OVERFLOW ← el error más común
```

---

## Widgets de layout — de un vistazo

Antes de combinarlos en ejemplos reales, aquí está cada widget
con sus propiedades más importantes y un uso mínimo.

---

### `Container`

Caja con tamaño, padding, margin y decoración en un solo widget.

```dart
Container(
  width:   200,                              // ancho fijo
  height:  60,                               // alto fijo
  margin:  const EdgeInsets.all(8),          // espacio EXTERIOR
  padding: const EdgeInsets.all(12),         // espacio INTERIOR
  // color y decoration NO pueden usarse juntos — el color va dentro de decoration
  decoration: BoxDecoration(
    color:        Colors.blue.shade50,       // fondo
    borderRadius: BorderRadius.circular(8),  // esquinas redondeadas
    border:       Border.all(color: Colors.blue, width: 1),
    boxShadow:    [BoxShadow(color: Colors.black12, blurRadius: 6)],
  ),
  child: const Text('Contenido'),
)
```

---

### `SizedBox`

Espaciador de tamaño fijo o wrapper con dimensiones controladas.

```dart
const SizedBox(height: 16)           // espacio vertical entre widgets
const SizedBox(width: 8)             // espacio horizontal
const SizedBox(width: 120, height: 40, child: ElevatedButton(...))
const SizedBox.expand()              // ocupa todo el espacio disponible
```

---

### `Padding` · `Center` · `Align`

Añaden espacio o controlan la posición del hijo dentro del espacio disponible.

```dart
Padding(
  padding: const EdgeInsets.symmetric(horizontal: 16),
  child: const Text('Con margen'),
)

const Center(child: Text('Centrado'))          // atajo de Align(center)

Align(
  alignment: Alignment.topRight,               // .center, .bottomLeft, etc.
  child: const Icon(Icons.settings),
)
```

---

### `Column`

Apila hijos en el eje **vertical**.

```dart
Column(
  mainAxisAlignment:  MainAxisAlignment.center,    // distribución VERTICAL
  //  .start · .end · .center · .spaceBetween · .spaceAround · .spaceEvenly
  crossAxisAlignment: CrossAxisAlignment.start,    // alineación HORIZONTAL
  //  .start · .end · .center · .stretch
  mainAxisSize: MainAxisSize.min,  // .min = solo lo que ocupa · .max = toda la altura
  children: [
    const Text('Primero'),
    const SizedBox(height: 8),
    const Text('Segundo'),
  ],
)
```

---

### `Row`

Alinea hijos en el eje **horizontal** — mismos parámetros que `Column`, ejes invertidos.

```dart
Row(
  mainAxisAlignment:  MainAxisAlignment.spaceBetween,  // distribución HORIZONTAL
  crossAxisAlignment: CrossAxisAlignment.center,        // alineación VERTICAL
  children: [
    const Icon(Icons.wifi),
    const Text('Red local'),
    const Icon(Icons.more_vert),
  ],
)
```

---

### `Expanded` · `Flexible` · `Spacer`

Controlan cómo un hijo ocupa el espacio libre dentro de `Column` o `Row`.

```dart
Row(children: [
  const Icon(Icons.dns),
  Expanded(                    // ocupa TODO el espacio restante
    child: Text('Nombre largo que podría desbordar', overflow: TextOverflow.ellipsis),
  ),
  const Text('Activo'),
])

Row(children: [
  const Text('Izquierda'),
  const Spacer(),              // espacio vacío flexible — empuja lo siguiente al extremo
  const Text('Derecha'),
])
```

> `Expanded` ≡ `Flexible(fit: FlexFit.tight)`.
> `Spacer()` ≡ `Expanded(child: SizedBox.shrink())`.

---

### `Stack` · `Positioned`

`Stack` superpone widgets en capas. `Positioned` coloca un hijo en coordenadas exactas.

```dart
Stack(
  alignment:    Alignment.center,   // alineación por defecto para hijos SIN Positioned
  clipBehavior: Clip.none,          // Clip.hardEdge recorta lo que sale del Stack
  children: [
    Container(width: 60, height: 60, color: Colors.blue.shade100),   // capa base
    Positioned(
      top: -4, right: -4,           // puede salir del Stack si clipBehavior: Clip.none
      child: Container(
        width: 18, height: 18,
        decoration: const BoxDecoration(color: Colors.red, shape: BoxShape.circle),
      ),
    ),
  ],
)
```

---

### `Wrap`

Flujo automático — cuando los hijos no caben en una fila, salta a la siguiente.

```dart
Wrap(
  spacing:    8,              // espacio horizontal entre hijos
  runSpacing: 8,              // espacio entre filas
  direction:  Axis.horizontal,// Axis.vertical para fluir en columnas
  alignment:  WrapAlignment.start,
  children: [
    Chip(label: Text('nginx')),
    Chip(label: Text('TLS 1.3')),
    Chip(label: Text('HTTP/2')),
    Chip(label: Text('IPv6')),
  ],
)
```

---

## Crea el proyecto

```bash
flutter create modulo07_layouts
cd modulo07_layouts
```

Borra todo el contenido de `lib/main.dart` y déjalo vacío.
El Paso 1 parte desde cero en ese archivo.

```
modulo07_layouts/
├── lib/
│   ├── main.dart          ← aquí trabajamos
│   └── widgets/           ← se crea al llegar al Paso 2
├── pubspec.yaml
└── ...
```

Para correr la app:
```bash
flutter run
```

---

## Paso 1 — Container: la caja universal

Escribe esto en `lib/main.dart`:

```dart
// lib/main.dart
import 'package:flutter/material.dart';

void main() => runApp(MaterialApp(
  home: Scaffold(
    body: Center(
      child: Container(
        width:   220,
        height:  80,
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
        decoration: BoxDecoration(
          color:        Colors.indigo.shade50,
          borderRadius: BorderRadius.circular(12),
          border:       Border.all(color: Colors.indigo, width: 1.5),
          boxShadow: [
            BoxShadow(
              color:      Colors.black.withOpacity(0.08),
              blurRadius: 8,
              offset:     const Offset(0, 2),
            ),
          ],
        ),
        child: const Text('Servidor web-01',
            style: TextStyle(fontWeight: FontWeight.bold)),
      ),
    ),
  ),
));
```

`Container` combina padding, margin, color y decoración en un solo widget.

> **Nota:** `color` y `decoration` no pueden usarse juntos.
> Si usas `BoxDecoration`, el color va dentro: `decoration: BoxDecoration(color: ...)`.

Referencia de `EdgeInsets`:
```dart
const EdgeInsets.all(16)
const EdgeInsets.symmetric(horizontal: 24, vertical: 12)
const EdgeInsets.only(top: 8, bottom: 16, left: 12)
```

### Prueba esto

- Cambia `width: 220` a `double.infinity` — el Container ocupa todo el ancho disponible
- Cambia `BorderRadius.circular(12)` a `circular(0)` (esquinas rectas) y a `circular(40)` (cápsula)
- Cambia `Colors.black.withOpacity(0.08)` a `0.3` — la sombra se intensifica
- Agrega `margin: const EdgeInsets.all(24)` antes de `padding` — nota que `margin` es espacio exterior al widget
- Reemplaza `Border.all(...)` por `Border(left: BorderSide(color: Colors.indigo, width: 4))` — borde solo izquierdo

---

## Convierte `main.dart` en selector de pasos

Antes de avanzar al Paso 2, actualiza `main.dart` **una sola vez**.
A partir de aquí solo cambias el número de `paso` para navegar entre ejemplos
— el Paso 1 sigue mostrando el mismo Container con `paso = 1`:

```dart
// lib/main.dart
import 'package:flutter/material.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1  Container — decoración y espaciado                  │
// │  2  Paso 2  Column — TarjetaLog                                 │
// │  3  Paso 3  Row + Expanded + Spacer — FilaEstado                │
// │  4  Paso 4  Stack + Positioned — AvatarBadge                   │
// │  5  Paso 5  SizedBox, Padding, Align, Wrap                      │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 1;

void main() => runApp(MaterialApp(
  debugShowCheckedModeBanner: false,
  home: switch (paso) {
    1 => _paso1(),
    _ => Scaffold(body: Center(child: Text('Paso $paso: crea el widget primero'))),
  },
));

Widget _paso1() => Scaffold(
  body: Center(
    child: Container(
      width:   220,
      height:  80,
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
      decoration: BoxDecoration(
        color:        Colors.indigo.shade50,
        borderRadius: BorderRadius.circular(12),
        border:       Border.all(color: Colors.indigo, width: 1.5),
        boxShadow: [
          BoxShadow(
            color:      Colors.black.withOpacity(0.08),
            blurRadius: 8,
            offset:     const Offset(0, 2),
          ),
        ],
      ),
      child: const Text('Servidor web-01',
          style: TextStyle(fontWeight: FontWeight.bold)),
    ),
  ),
);
```

Guarda y verifica que la app sigue mostrando el Container. Listo — de ahora
en adelante cada paso nuevo solo requiere agregar un `import` y un `case` al switch.

---

## Paso 2 — Column: apilar verticalmente

`Column` apila sus hijos en el eje vertical.

```
mainAxisAlignment  → cómo distribuir espacio VERTICAL
                     .start · .center · .end · .spaceBetween · .spaceAround · .spaceEvenly

crossAxisAlignment → cómo alinear hijos HORIZONTALMENTE
                     .start · .end · .center · .stretch

mainAxisSize       → .max (toda la altura, por defecto) · .min (solo lo necesario)
```

### Archivo: `lib/widgets/tarjeta_log.dart`

Crea la carpeta `lib/widgets/` y dentro el archivo:

```dart
import 'package:flutter/material.dart';

class TarjetaLog extends StatelessWidget {
  final String   nivel;        // DEBUG, INFO, WARN, ERROR
  final String   componente;
  final String   mensaje;
  final DateTime timestamp;

  const TarjetaLog({
    super.key,
    required this.nivel,
    required this.componente,
    required this.mensaje,
    required this.timestamp,
  });

  Color get _colorNivel => switch (nivel) {
    'DEBUG' => Colors.grey,
    'INFO'  => Colors.blue,
    'WARN'  => Colors.orange,
    'ERROR' => Colors.red,
    _       => Colors.grey,
  };

  @override
  Widget build(BuildContext context) {
    return Container(
      margin:  const EdgeInsets.symmetric(horizontal: 16, vertical: 4),
      padding: const EdgeInsets.all(12),
      decoration: BoxDecoration(
        color:        _colorNivel.withOpacity(0.05),
        borderRadius: BorderRadius.circular(8),
        border:       Border(left: BorderSide(color: _colorNivel, width: 3)),
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        mainAxisSize:       MainAxisSize.min,
        children: [
          // Fila superior: nivel + componente + hora
          Row(
            children: [
              Container(
                padding:    const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
                decoration: BoxDecoration(
                  color: _colorNivel, borderRadius: BorderRadius.circular(4)),
                child: Text(nivel,
                    style: const TextStyle(
                        color: Colors.white, fontSize: 10, fontWeight: FontWeight.bold)),
              ),
              const SizedBox(width: 8),
              Text(componente,
                  style: TextStyle(
                      fontSize: 12, color: Colors.grey.shade700, fontWeight: FontWeight.w600)),
              const Spacer(),
              Text(
                '${timestamp.hour.toString().padLeft(2, '0')}:'
                '${timestamp.minute.toString().padLeft(2, '0')}:'
                '${timestamp.second.toString().padLeft(2, '0')}',
                style: TextStyle(fontSize: 11, color: Colors.grey.shade500),
              ),
            ],
          ),
          const SizedBox(height: 6),
          Text(mensaje, style: TextStyle(fontSize: 13, color: Colors.grey.shade800)),
        ],
      ),
    );
  }
}
```

### Agrega al `main.dart`

1. Import al inicio:
```dart
import 'widgets/tarjeta_log.dart';
```

2. Nuevo case en el switch:
```dart
2 => Scaffold(
      body: ListView(
        children: [
          TarjetaLog(nivel: 'ERROR', componente: 'auth-service',
              mensaje:   'Token expirado — usuario forzado a re-login',
              timestamp: DateTime.now()),
          TarjetaLog(nivel: 'WARN',  componente: 'db-pool',
              mensaje:   'Conexiones disponibles: 2 / 10',
              timestamp: DateTime.now().subtract(const Duration(minutes: 2))),
          TarjetaLog(nivel: 'INFO',  componente: 'scheduler',
              mensaje:   'Tarea de backup completada',
              timestamp: DateTime.now().subtract(const Duration(minutes: 5))),
          TarjetaLog(nivel: 'DEBUG', componente: 'http-client',
              mensaje:   'GET /api/status → 200 OK (38ms)',
              timestamp: DateTime.now().subtract(const Duration(minutes: 8))),
        ],
      ),
    ),
```

3. Cambia `const int paso = 2;` y guarda.

### Prueba esto

- Cambia `crossAxisAlignment: CrossAxisAlignment.start` a `CrossAxisAlignment.center` en la `Column` — el mensaje se centra
- Cambia `mainAxisSize: MainAxisSize.min` a `MainAxisSize.max` — la tarjeta ocupa toda la altura disponible
- Cambia `_colorNivel.withOpacity(0.05)` a `0.15` — más contraste de fondo
- Cambia `BorderSide(color: _colorNivel, width: 3)` a `width: 6` — acento izquierdo más grueso
- Añade `nivel: 'DEBUG'` a un `TarjetaLog` para ver el color gris

---

## Paso 3 — Row, Expanded y Spacer

`Row` alinea sus hijos en el eje horizontal (mismos parámetros que `Column`, ejes invertidos).

```dart
// Expanded — ocupa TODO el espacio restante del eje
// Sin Expanded, un texto largo desborda la Row

Row(children: [
  Icon(Icons.wifi),
  Expanded(child: Text('Nombre de red muy largo...')),  // ← necesario
  Text('WPA3'),
])

// Spacer — espacio vacío flexible que empuja los demás al extremo
Row(children: [
  Text('Servidor'),
  Spacer(),                  // ← empuja lo siguiente al borde derecho
  Icon(Icons.circle, color: Colors.green, size: 12),
])
```

### Archivo: `lib/widgets/fila_estado.dart`

```dart
import 'package:flutter/material.dart';

class FilaEstado extends StatelessWidget {
  final String nombre;
  final String detalle;
  final bool   activo;

  const FilaEstado({
    super.key,
    required this.nombre,
    required this.detalle,
    required this.activo,
  });

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
      child: Row(
        children: [
          // Ícono de estado
          Icon(
            activo ? Icons.check_circle : Icons.cancel,
            color: activo ? Colors.green : Colors.red,
            size:  20,
          ),
          const SizedBox(width: 12),

          // Expanded — el Column ocupa todo el espacio restante
          // Sin Expanded, un nombre largo desbordaría la Row
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              mainAxisSize:       MainAxisSize.min,
              children: [
                Text(nombre,
                    style:    const TextStyle(fontWeight: FontWeight.w600),
                    overflow: TextOverflow.ellipsis),
                Text(detalle,
                    style: TextStyle(fontSize: 12, color: Colors.grey.shade600)),
              ],
            ),
          ),

          const SizedBox(width: 8),

          // Chip de estado — queda pegado al borde derecho gracias a Expanded
          Container(
            padding:    const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
            decoration: BoxDecoration(
              color:        (activo ? Colors.green : Colors.red).withOpacity(0.1),
              borderRadius: BorderRadius.circular(12),
            ),
            child: Text(
              activo ? 'Activo' : 'Caído',
              style: TextStyle(
                fontSize:   11,
                color:      activo ? Colors.green.shade700 : Colors.red.shade700,
                fontWeight: FontWeight.w600,
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
import 'widgets/fila_estado.dart';
```

```dart
3 => const Scaffold(
      body: Column(
        children: [
          FilaEstado(nombre: 'nginx-proxy',   detalle: '10.0.0.5 · 45ms',          activo: true),
          Divider(height: 1),
          FilaEstado(nombre: 'db-primary',    detalle: '10.0.0.12 · 8ms',           activo: true),
          Divider(height: 1),
          FilaEstado(nombre: 'backup-worker', detalle: '10.0.0.30 · sin respuesta', activo: false),
          Divider(height: 1),
          FilaEstado(nombre: 'api-gateway-produccion-region-us-east',
                     detalle: '10.0.0.8 · 12ms', activo: true),
        ],
      ),
    ),
```

Cambia `paso = 3` y guarda.

### Prueba esto

- Quita el `Expanded` que envuelve el `Column` — con el último nombre largo verás el overflow
- Reemplaza `Expanded(child: Column(...))` por solo `Text(nombre)` y agrega `const Spacer()` antes del chip — compara `Spacer` vs `Expanded`
- Cambia `Icons.check_circle` a `Icons.circle` — mismo ícono pero sin relleno, más sutil
- Agrega `mainAxisAlignment: MainAxisAlignment.spaceBetween` a la `Row` — ¿cambia algo con `Expanded` ya presente?
- Cambia `crossAxisAlignment: CrossAxisAlignment.start` en el `Column` a `CrossAxisAlignment.end` — ¿el detalle se desplaza?

---

## Paso 4 — Stack y Positioned: capas superpuestas

`Stack` coloca widgets uno encima del otro.
`Positioned` ubica un hijo en coordenadas exactas dentro del `Stack`.

```dart
Stack(
  alignment: Alignment.center,  // alineación por defecto para hijos sin Positioned
  children: [
    Container(width: 60, height: 60, color: Colors.blue.shade100),  // capa inferior
    Positioned(
      top: 4, right: 4,
      child: Container(
        padding:    const EdgeInsets.all(4),
        decoration: const BoxDecoration(color: Colors.red, shape: BoxShape.circle),
        child: const Text('3', style: TextStyle(color: Colors.white, fontSize: 10)),
      ),
    ),
  ],
)
```

### Archivo: `lib/widgets/avatar_badge.dart`

```dart
import 'package:flutter/material.dart';

class AvatarBadge extends StatelessWidget {
  final String nombre;
  final int    alertas;
  final bool   activo;

  const AvatarBadge({
    super.key,
    required this.nombre,
    required this.alertas,
    required this.activo,
  });

  @override
  Widget build(BuildContext context) {
    return Stack(
      clipBehavior: Clip.none,   // permite que el badge salga del Stack
      children: [
        // Avatar — capa inferior
        Container(
          width:  56,
          height: 56,
          decoration: BoxDecoration(
            color:        activo ? Colors.indigo.shade100 : Colors.grey.shade200,
            borderRadius: BorderRadius.circular(12),
          ),
          child: Center(
            child: Text(
              nombre.substring(0, 2).toUpperCase(),
              style: TextStyle(
                fontWeight: FontWeight.bold,
                fontSize:   18,
                color:      activo ? Colors.indigo : Colors.grey,
              ),
            ),
          ),
        ),

        // Punto de estado — esquina inferior derecha
        Positioned(
          bottom: 0, right: 0,
          child: Container(
            width:  14,
            height: 14,
            decoration: BoxDecoration(
              color:  activo ? Colors.green : Colors.red,
              shape:  BoxShape.circle,
              border: Border.all(color: Colors.white, width: 2),
            ),
          ),
        ),

        // Badge de alertas — capa superior, solo si las hay
        if (alertas > 0)
          Positioned(
            top: -4, right: -4,
            child: Container(
              padding:     const EdgeInsets.all(4),
              decoration:  const BoxDecoration(color: Colors.orange, shape: BoxShape.circle),
              constraints: const BoxConstraints(minWidth: 18, minHeight: 18),
              child: Text(
                alertas > 9 ? '9+' : '$alertas',
                style: const TextStyle(
                    color: Colors.white, fontSize: 10, fontWeight: FontWeight.bold),
                textAlign: TextAlign.center,
              ),
            ),
          ),
      ],
    );
  }
}
```

### Agrega al `main.dart`

```dart
import 'widgets/avatar_badge.dart';
```

```dart
4 => const Scaffold(
      body: Center(
        child: Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            AvatarBadge(nombre: 'web-01', alertas: 2,  activo: true),
            SizedBox(width: 24),
            AvatarBadge(nombre: 'db-01',  alertas: 0,  activo: true),
            SizedBox(width: 24),
            AvatarBadge(nombre: 'worker', alertas: 0,  activo: false),
            SizedBox(width: 24),
            AvatarBadge(nombre: 'cache',  alertas: 11, activo: true),
          ],
        ),
      ),
    ),
```

Cambia `paso = 4` y guarda.

### Prueba esto

- Cambia `clipBehavior: Clip.none` a `Clip.hardEdge` — el badge de alertas queda recortado por el borde del Stack
- Cambia `Positioned(top: -4, right: -4, ...)` a `Positioned(top: 0, right: 0, ...)` — el badge queda dentro del avatar
- Cambia `shape: BoxShape.circle` del punto de estado a `BoxShape.rectangle` — se convierte en un cuadrado
- Prueba `alertas: 15` en el último avatar — deberías ver el texto `9+`
- Cambia `Border.all(color: Colors.white, width: 2)` a `width: 4` — borde del punto más grueso

---

## Paso 5 — SizedBox, Padding, Align y Wrap

Todo el código de este paso vive directamente en `main.dart`.
No necesitas crear ningún archivo extra.

### Agrega al `main.dart`

```dart
5 => Scaffold(
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          // SizedBox — espaciado fijo
          const Text('SizedBox', style: TextStyle(fontWeight: FontWeight.bold)),
          const SizedBox(height: 8),
          const Text('Primer elemento'),
          const SizedBox(height: 32),          // ← espacio fijo de 32px
          const Text('Segundo elemento (después de 32px)'),

          const Divider(height: 32),

          // Padding — espacio alrededor de un hijo
          const Text('Padding', style: TextStyle(fontWeight: FontWeight.bold)),
          const SizedBox(height: 8),
          Container(
            color: Colors.indigo.shade50,
            child: const Padding(
              padding: EdgeInsets.only(left: 24),    // ← sangría izquierda
              child:   Text('Texto con Padding izquierdo'),
            ),
          ),

          const Divider(height: 32),

          // Align — posicionar dentro del espacio disponible
          const Text('Align', style: TextStyle(fontWeight: FontWeight.bold)),
          const SizedBox(height: 8),
          const Align(
            alignment: Alignment.centerRight,        // ← borde derecho
            child: Icon(Icons.settings, color: Colors.indigo),
          ),

          const Divider(height: 32),

          // Wrap — flujo automático de elementos
          const Text('Wrap', style: TextStyle(fontWeight: FontWeight.bold)),
          const SizedBox(height: 8),
          Wrap(
            spacing:    8,
            runSpacing: 8,
            children: ['nginx', 'TLS 1.3', 'HTTP/2', 'IPv6', 'Load Balancer', 'CDN', 'WAF']
                .map((t) => Chip(label: Text(t)))
                .toList(),
          ),
        ],
      ),
    ),
```

Cambia `paso = 5` y guarda.

### Prueba esto

- Cambia `SizedBox(height: 32)` a `height: 4` y a `height: 100` — controla el espacio
- Cambia `Alignment.centerRight` a `Alignment.centerLeft`, `Alignment.topRight`, `Alignment.bottomLeft`
- Cambia `spacing: 8` del `Wrap` a `spacing: 24` — más espacio horizontal entre chips
- Agrega `direction: Axis.vertical` al `Wrap` — los chips fluyen verticalmente
- Reemplaza los `Chip` por `Container(width: 40, height: 40, color: Colors.primaries[i % 18])` con `List.generate` — crea una paleta de colores

---

## `main.dart` completo — referencia

Al terminar todos los pasos, el archivo queda así.
Cambia `paso` en cualquier momento para revisar cualquier ejemplo:

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'widgets/tarjeta_log.dart';
import 'widgets/fila_estado.dart';
import 'widgets/avatar_badge.dart';

// ┌──────────────────────────────────────────────────────────────────┐
// │  Cambia este número y guarda (Ctrl+S) para navegar entre pasos. │
// │  1  Paso 1  Container — decoración y espaciado                  │
// │  2  Paso 2  Column — TarjetaLog                                 │
// │  3  Paso 3  Row + Expanded + Spacer — FilaEstado                │
// │  4  Paso 4  Stack + Positioned — AvatarBadge                   │
// │  5  Paso 5  SizedBox, Padding, Align, Wrap                      │
// └──────────────────────────────────────────────────────────────────┘
const int paso = 1;

void main() => runApp(MaterialApp(
  debugShowCheckedModeBanner: false,
  home: switch (paso) {
    1 => _paso1(),
    2 => Scaffold(
          body: ListView(
            children: [
              TarjetaLog(nivel: 'ERROR', componente: 'auth-service',
                  mensaje:   'Token expirado — usuario forzado a re-login',
                  timestamp: DateTime.now()),
              TarjetaLog(nivel: 'WARN',  componente: 'db-pool',
                  mensaje:   'Conexiones disponibles: 2 / 10',
                  timestamp: DateTime.now().subtract(const Duration(minutes: 2))),
              TarjetaLog(nivel: 'INFO',  componente: 'scheduler',
                  mensaje:   'Tarea de backup completada',
                  timestamp: DateTime.now().subtract(const Duration(minutes: 5))),
              TarjetaLog(nivel: 'DEBUG', componente: 'http-client',
                  mensaje:   'GET /api/status → 200 OK (38ms)',
                  timestamp: DateTime.now().subtract(const Duration(minutes: 8))),
            ],
          ),
        ),
    3 => const Scaffold(
          body: Column(
            children: [
              FilaEstado(nombre: 'nginx-proxy',   detalle: '10.0.0.5 · 45ms',          activo: true),
              Divider(height: 1),
              FilaEstado(nombre: 'db-primary',    detalle: '10.0.0.12 · 8ms',           activo: true),
              Divider(height: 1),
              FilaEstado(nombre: 'backup-worker', detalle: '10.0.0.30 · sin respuesta', activo: false),
              Divider(height: 1),
              FilaEstado(nombre: 'api-gateway-produccion-region-us-east',
                         detalle: '10.0.0.8 · 12ms', activo: true),
            ],
          ),
        ),
    4 => const Scaffold(
          body: Center(
            child: Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                AvatarBadge(nombre: 'web-01', alertas: 2,  activo: true),
                SizedBox(width: 24),
                AvatarBadge(nombre: 'db-01',  alertas: 0,  activo: true),
                SizedBox(width: 24),
                AvatarBadge(nombre: 'worker', alertas: 0,  activo: false),
                SizedBox(width: 24),
                AvatarBadge(nombre: 'cache',  alertas: 11, activo: true),
              ],
            ),
          ),
        ),
    5 => Scaffold(
          body: ListView(
            padding: const EdgeInsets.all(16),
            children: [
              const Text('SizedBox', style: TextStyle(fontWeight: FontWeight.bold)),
              const SizedBox(height: 8),
              const Text('Primer elemento'),
              const SizedBox(height: 32),
              const Text('Segundo elemento (después de 32px)'),
              const Divider(height: 32),
              const Text('Padding', style: TextStyle(fontWeight: FontWeight.bold)),
              const SizedBox(height: 8),
              Container(
                color: Colors.indigo.shade50,
                child: const Padding(
                  padding: EdgeInsets.only(left: 24),
                  child:   Text('Texto con Padding izquierdo'),
                ),
              ),
              const Divider(height: 32),
              const Text('Align', style: TextStyle(fontWeight: FontWeight.bold)),
              const SizedBox(height: 8),
              const Align(
                alignment: Alignment.centerRight,
                child: Icon(Icons.settings, color: Colors.indigo),
              ),
              const Divider(height: 32),
              const Text('Wrap', style: TextStyle(fontWeight: FontWeight.bold)),
              const SizedBox(height: 8),
              Wrap(
                spacing: 8, runSpacing: 8,
                children: ['nginx', 'TLS 1.3', 'HTTP/2', 'IPv6', 'Load Balancer', 'CDN', 'WAF']
                    .map((t) => Chip(label: Text(t)))
                    .toList(),
              ),
            ],
          ),
        ),
    _ => Scaffold(body: Center(child: Text('Paso $paso no definido'))),
  },
));

Widget _paso1() => Scaffold(
  body: Center(
    child: Container(
      width:   220,
      height:  80,
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
      decoration: BoxDecoration(
        color:        Colors.indigo.shade50,
        borderRadius: BorderRadius.circular(12),
        border:       Border.all(color: Colors.indigo, width: 1.5),
        boxShadow: [
          BoxShadow(
            color:      Colors.black.withOpacity(0.08),
            blurRadius: 8,
            offset:     const Offset(0, 2),
          ),
        ],
      ),
      child: const Text('Servidor web-01',
          style: TextStyle(fontWeight: FontWeight.bold)),
    ),
  ),
);
```

---

## Proyecto — Pantalla de Topología de Red

Combina todos los pasos en una app completa con múltiples archivos.

```
modulo07_layouts/
└── lib/
    ├── main.dart                          ← entrada del proyecto
    ├── models/
    │   └── dispositivo.dart              ← modelo de datos (sin UI)
    ├── widgets/
    │   ├── chip_resumen.dart             ← Row pequeño (Paso 3)
    │   ├── avatar_badge.dart             ← Stack (Paso 4 — mismo archivo)
    │   └── fila_dispositivo.dart         ← Row + Column + Wrap (Pasos 2-3-5)
    └── screens/
        └── pantalla_topologia.dart       ← pantalla completa
```

> Para el proyecto, reemplaza el contenido de `lib/main.dart` con el que se muestra al final.

---

### `lib/models/dispositivo.dart`

```dart
class InfoDispositivo {
  final String       nombre;
  final String       tipo;       // 'router', 'switch', 'server', 'endpoint'
  final String       ip;
  final bool         activo;
  final int          alertas;
  final List<String> etiquetas;

  const InfoDispositivo({
    required this.nombre,
    required this.tipo,
    required this.ip,
    required this.activo,
    this.alertas   = 0,
    this.etiquetas = const [],
  });
}
```

---

### `lib/widgets/chip_resumen.dart`

```dart
import 'package:flutter/material.dart';

class ChipResumen extends StatelessWidget {
  final IconData icono;
  final String   texto;
  final Color    color;

  const ChipResumen({
    super.key,
    required this.icono,
    required this.texto,
    required this.color,
  });

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisSize: MainAxisSize.min,
      children: [
        Icon(icono, size: 14, color: color),
        const SizedBox(width: 4),
        Text(texto,
            style: TextStyle(fontSize: 12, color: color, fontWeight: FontWeight.w600)),
      ],
    );
  }
}
```

---

### `lib/widgets/avatar_badge.dart`

El mismo archivo del Paso 4 — no se cambia nada.

---

### `lib/widgets/fila_dispositivo.dart`

Composición: `Row` externo + `Column` de datos + `Wrap` de etiquetas + `AvatarBadge`:

```dart
import 'package:flutter/material.dart';
import '../models/dispositivo.dart';
import 'avatar_badge.dart';

class FilaDispositivo extends StatelessWidget {
  final InfoDispositivo dispositivo;

  const FilaDispositivo({super.key, required this.dispositivo});

  IconData get _icono => switch (dispositivo.tipo) {
    'router'   => Icons.router,
    'switch'   => Icons.device_hub,
    'server'   => Icons.dns,
    'endpoint' => Icons.computer,
    _          => Icons.devices,
  };

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 10),
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [

          // AvatarBadge — Stack del Paso 4
          AvatarBadge(
            nombre:  dispositivo.nombre,
            alertas: dispositivo.alertas,
            activo:  dispositivo.activo,
          ),

          const SizedBox(width: 12),

          // Column: nombre, IP y Wrap de etiquetas
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Row(
                  children: [
                    Expanded(
                      child: Text(
                        dispositivo.nombre,
                        style:    const TextStyle(fontWeight: FontWeight.bold),
                        overflow: TextOverflow.ellipsis,
                      ),
                    ),
                    Icon(_icono, size: 16, color: Colors.grey.shade500),
                  ],
                ),
                Text(dispositivo.ip,
                    style: TextStyle(fontSize: 12, color: Colors.grey.shade600)),
                const SizedBox(height: 6),
                Wrap(
                  spacing: 4, runSpacing: 4,
                  children: dispositivo.etiquetas.map((tag) =>
                    Container(
                      padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
                      decoration: BoxDecoration(
                        color:        Colors.indigo.shade50,
                        borderRadius: BorderRadius.circular(4),
                        border:       Border.all(color: Colors.indigo.shade200),
                      ),
                      child: Text(tag,
                          style: TextStyle(
                              fontSize:   10,
                              color:      Colors.indigo.shade700,
                              fontWeight: FontWeight.w500)),
                    ),
                  ).toList(),
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

---

### `lib/screens/pantalla_topologia.dart`

```dart
import 'package:flutter/material.dart';
import '../models/dispositivo.dart';
import '../widgets/chip_resumen.dart';
import '../widgets/fila_dispositivo.dart';

class PantallaTopologia extends StatelessWidget {
  const PantallaTopologia({super.key});

  @override
  Widget build(BuildContext context) {
    final dispositivos = [
      const InfoDispositivo(
        nombre: 'core-router', tipo: 'router',
        ip: '10.0.0.1', activo: true, alertas: 2,
        etiquetas: ['BGP', 'OSPF', 'Gateway'],
      ),
      const InfoDispositivo(
        nombre: 'sw-distribucion', tipo: 'switch',
        ip: '10.0.1.1', activo: true, alertas: 0,
        etiquetas: ['L3', 'VLAN 10', 'VLAN 20'],
      ),
      const InfoDispositivo(
        nombre: 'prod-web-01', tipo: 'server',
        ip: '10.0.2.10', activo: true, alertas: 1,
        etiquetas: ['nginx', 'TLS'],
      ),
      const InfoDispositivo(
        nombre: 'prod-db-01', tipo: 'server',
        ip: '10.0.2.20', activo: true, alertas: 3,
        etiquetas: ['PostgreSQL', 'Primary'],
      ),
      const InfoDispositivo(
        nombre: 'backup-srv', tipo: 'server',
        ip: '10.0.3.5', activo: false, alertas: 0,
        etiquetas: ['Backup', 'Offsite'],
      ),
    ];

    final totalAlertas = dispositivos.fold(0, (s, d) => s + d.alertas);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Topología de Red'),
        actions: [
          IconButton(icon: const Icon(Icons.filter_list), onPressed: () {}),
          IconButton(icon: const Icon(Icons.refresh),     onPressed: () {}),
        ],
      ),
      body: Column(
        children: [
          // Cabecera — Container con Row de ChipResumen (Pasos 1 + 3)
          Container(
            color:   Theme.of(context).colorScheme.surfaceContainerHighest,
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
            child: Row(
              children: [
                ChipResumen(
                  icono: Icons.hub,
                  texto: '${dispositivos.length} dispositivos',
                  color: Colors.indigo,
                ),
                const SizedBox(width: 16),
                ChipResumen(
                  icono: Icons.circle,
                  texto: '${dispositivos.where((d) => d.activo).length} activos',
                  color: Colors.green,
                ),
                const SizedBox(width: 16),
                ChipResumen(
                  icono: Icons.warning_amber,
                  texto: '$totalAlertas alertas',
                  color: Colors.orange,
                ),
              ],
            ),
          ),

          // Lista — Expanded para que ocupe el espacio restante (Paso 3)
          Expanded(
            child: ListView.separated(
              padding:          const EdgeInsets.symmetric(vertical: 8),
              itemCount:        dispositivos.length,
              separatorBuilder: (_, __) => const Divider(height: 1),
              itemBuilder:      (_, i)  => FilaDispositivo(dispositivo: dispositivos[i]),
            ),
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
import 'screens/pantalla_topologia.dart';

void main() => runApp(const AppTopologia());

class AppTopologia extends StatelessWidget {
  const AppTopologia({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title:                      'Topología de Red',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme:  ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const PantallaTopologia(),
    );
  }
}
```

---

## Guía rápida de imports

```dart
// Desde lib/main.dart → lib/widgets/
import 'widgets/tarjeta_log.dart';
import 'widgets/fila_estado.dart';
import 'widgets/avatar_badge.dart';

// Desde lib/widgets/ → lib/models/
import '../models/dispositivo.dart';

// Desde lib/widgets/ → lib/widgets/
import 'avatar_badge.dart';

// Desde lib/screens/ → lib/widgets/
import '../widgets/chip_resumen.dart';
import '../widgets/fila_dispositivo.dart';

// Desde lib/screens/ → lib/models/
import '../models/dispositivo.dart';
```

---

## Cuándo usar qué widget de layout

```
Container   → decoración (color, borde, sombra, radio), padding y margin en uno solo
Column      → apilar widgets verticalmente
Row         → alinear widgets horizontalmente
Expanded    → que un hijo ocupe todo el espacio restante del eje (dentro de Column/Row)
Spacer      → espacio vacío flexible que empuja los demás al extremo
Stack       → superponer widgets en capas
Positioned  → coordenadas exactas dentro de un Stack
SizedBox    → espaciador fijo o tamaño controlado
Padding     → espacio alrededor de un hijo
Align       → posicionar un hijo dentro del espacio disponible
Wrap        → flujo automático — salta a la siguiente fila cuando no caben los hijos
```

---

## Ejercicios propuestos

1. **Tarjeta de métrica de red** — Crea `lib/widgets/tarjeta_trafico.dart`.
   Usa `Stack`: el fondo es un `LinearProgressIndicator` vertical que muestra
   el uso del ancho de banda (0–100%). Encima, centrado, muestra el porcentaje
   en texto grande y el nombre de la interfaz (`eth0`, `wlan0`) en texto pequeño.

2. **Fila de estadísticas** — Crea una `Row` con tres secciones separadas por
   `VerticalDivider`. Cada sección tiene el valor en grande y la etiqueta en
   pequeño. Usa `Expanded(flex: 1)` para que las tres sean del mismo ancho.
   Métricas: `Latencia / 45ms`, `Paquetes / 12.4k`, `Uptime / 99.8%`.

3. **Grid de puertos** — Usa `Wrap` para mostrar 24 puertos de un switch.
   Cada puerto es un `Container` de 32×32 con bordes redondeados. El color
   indica estado: verde = activo, gris = libre, naranja = error, rojo = bloqueado.
   Usa `List.generate` con una lista de estados.

4. **Layout responsivo** — Usa `LayoutBuilder` para detectar el ancho disponible:
   si ancho < 600px → `ListView` de tarjetas en una columna;
   si ancho ≥ 600px → `GridView` de 2 columnas.
   Pruébalo en emulador de teléfono y tablet.

---

## Resumen del módulo 7

- Flutter usa un sistema de **constraints**: el padre da restricciones, el hijo decide su tamaño dentro de ellas.
- `Container` combina padding, margin, color y decoración. `color` y `decoration` no pueden usarse juntos — el color va dentro de `BoxDecoration`.
- `Column` apila verticalmente. `mainAxisAlignment` controla la distribución en el eje principal. `crossAxisAlignment` controla la alineación en el eje cruzado.
- `Row` alinea horizontalmente con los mismos ejes pero intercambiados.
- `Expanded` ocupa todo el espacio restante en su eje. Sin él, un texto largo puede **desbordarse** (overflow). `Spacer()` es espacio vacío flexible.
- `Stack` superpone widgets en capas. `Positioned` coloca un hijo en coordenadas exactas. `clipBehavior: Clip.none` permite que los hijos salgan del área del Stack.
- `SizedBox` crea espaciadores o tamaños fijos. `Padding` añade espacio alrededor. `Align` posiciona el hijo dentro del espacio disponible.
- `Wrap` fluye elementos en filas y salta automáticamente cuando no caben — ideal para tags y chips.
- `Expanded` dentro de `Column`/`Row` sin altura/ancho acotado puede causar **overflow** — asegúrate siempre de que el eje principal esté acotado.

---

> **Siguiente →** Módulo 8: Material 3 y tema — `MaterialApp`, `ThemeData`, `ColorScheme`, modo oscuro y componentes de scaffold.
