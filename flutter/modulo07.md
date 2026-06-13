# Tutorial Flutter — Página 7
## Módulo 2 · Flutter
### Layouts y componentes básicos: `Column`, `Row`, `Stack`, `Container` y más

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
1. Un widget acotado (bounded)    → tiene tamaño máximo definido
2. Un widget no acotado (unbounded) → puede ser infinito (ListView, Column)
3. Unconstrained dentro de unbounded → OVERFLOW ← el error más común
```

---

## `Container` — la caja universal

`Container` combina padding, margin, color, decoración y tamaño:

```dart
// Container mínimo — solo color
Container(
  color: Colors.blue,
  child: Text('Hola'),
)

// Container completo
Container(
  width:   200,
  height:  100,
  margin:  const EdgeInsets.all(8),
  padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
  decoration: BoxDecoration(
    color:        Colors.indigo.shade50,
    borderRadius: BorderRadius.circular(12),
    border: Border.all(color: Colors.indigo, width: 1.5),
    boxShadow: [
      BoxShadow(
        color:   Colors.black.withOpacity(0.08),
        blurRadius:   8,
        offset:  const Offset(0, 2),
      ),
    ],
  ),
  child: const Text('Contenedor decorado'),
)
```

> **Nota:** `color` y `decoration` no pueden usarse juntos.
> Si usas `BoxDecoration`, el color va dentro: `decoration: BoxDecoration(color: ...)`.

### `EdgeInsets` — espaciado

```dart
// Todos los lados iguales
const EdgeInsets.all(16)

// Horizontal y vertical por separado
const EdgeInsets.symmetric(horizontal: 24, vertical: 12)

// Lados individuales
const EdgeInsets.only(top: 8, bottom: 16, left: 12)

// Usando valores del sistema (safe area, barra de estado)
EdgeInsets.fromWindowPadding(...)   // raro
MediaQuery.paddingOf(context)       // safe area del dispositivo
```

---

## `Column` — apila verticalmente

```dart
Column(
  // mainAxisAlignment — cómo distribuir los hijos en el EJE PRINCIPAL (vertical)
  mainAxisAlignment: MainAxisAlignment.start,     // por defecto
  // .center, .end, .spaceBetween, .spaceAround, .spaceEvenly

  // crossAxisAlignment — alineación en el EJE CRUZADO (horizontal)
  crossAxisAlignment: CrossAxisAlignment.stretch, // ocupa todo el ancho
  // .start, .end, .center, .baseline

  // mainAxisSize — cuánto espacio ocupa la Column en su eje
  mainAxisSize: MainAxisSize.min,  // solo lo necesario
  // .max — toda la altura disponible (por defecto)

  children: [
    const Text('Elemento 1'),
    const SizedBox(height: 8),   // espaciador
    const Text('Elemento 2'),
    const Text('Elemento 3'),
  ],
)
```

### Ejemplo aplicado — tarjeta de log de evento

```dart
class TarjetaLogEvento extends StatelessWidget {
  final String  nivel;     // DEBUG, INFO, WARN, ERROR
  final String  componente;
  final String  mensaje;
  final DateTime timestamp;

  const TarjetaLogEvento({
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
                padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
                decoration: BoxDecoration(
                  color:        _colorNivel,
                  borderRadius: BorderRadius.circular(4),
                ),
                child: Text(
                  nivel,
                  style: const TextStyle(
                    color:      Colors.white,
                    fontSize:   10,
                    fontWeight: FontWeight.bold,
                  ),
                ),
              ),
              const SizedBox(width: 8),
              Text(
                componente,
                style: TextStyle(
                  fontSize: 12,
                  color:    Colors.grey.shade700,
                  fontWeight: FontWeight.w600,
                ),
              ),
              const Spacer(),
              Text(
                '${timestamp.hour.toString().padLeft(2,'0')}:'
                '${timestamp.minute.toString().padLeft(2,'0')}:'
                '${timestamp.second.toString().padLeft(2,'0')}',
                style: TextStyle(fontSize: 11, color: Colors.grey.shade500),
              ),
            ],
          ),
          const SizedBox(height: 6),
          // Mensaje
          Text(
            mensaje,
            style: TextStyle(
              fontSize: 13,
              color:    Colors.grey.shade800,
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## `Row` — alinea horizontalmente

```dart
Row(
  mainAxisAlignment:  MainAxisAlignment.spaceBetween,
  crossAxisAlignment: CrossAxisAlignment.center,
  children: [
    const Icon(Icons.wifi, color: Colors.green),
    const Text('192.168.1.1'),
    const Chip(label: Text('Conectado')),
  ],
)
```

### `Expanded` y `Flexible` — distribuir espacio disponible

```dart
// Expanded — ocupa TODO el espacio restante
Row(
  children: [
    const Icon(Icons.signal_wifi_4_bar),
    const SizedBox(width: 8),
    Expanded(
      // Sin Expanded, el texto podría desbordar la Row
      child: Text(
        'Nombre de red muy largo que no cabe en una línea',
        overflow: TextOverflow.ellipsis,
      ),
    ),
    const Text('WPA3'),
  ],
)

// Flexible — ocupa el espacio que necesita, hasta el máximo disponible
Row(
  children: [
    Flexible(flex: 2, child: Container(color: Colors.red,   height: 40)),
    Flexible(flex: 1, child: Container(color: Colors.green, height: 40)),
    Flexible(flex: 1, child: Container(color: Colors.blue,  height: 40)),
  ],
)
// flex: 2 ocupa el doble que flex: 1
```

### `Spacer` — espacio flexible vacío

```dart
Row(
  children: [
    const Text('Servidor'),
    const Spacer(),           // empuja los siguientes al borde derecho
    const Icon(Icons.circle, color: Colors.green, size: 12),
    const SizedBox(width: 4),
    const Text('Activo'),
  ],
)
```

### Ejemplo aplicado — fila de dispositivo de red

```dart
class FilaDispositivoRed extends StatelessWidget {
  final String   nombre;
  final String   ip;
  final String   mac;
  final bool     conectado;
  final int      senialDbm;
  final VoidCallback? onTap;

  const FilaDispositivoRed({
    super.key,
    required this.nombre,
    required this.ip,
    required this.mac,
    required this.conectado,
    required this.senialDbm,
    this.onTap,
  });

  IconData get _iconoSenial {
    if (!conectado) return Icons.wifi_off;
    if (senialDbm >= -50) return Icons.signal_wifi_4_bar;
    if (senialDbm >= -70) return Icons.network_wifi_3_bar;
    return Icons.network_wifi_1_bar;
  }

  @override
  Widget build(BuildContext context) {
    return InkWell(
      onTap: onTap,
      child: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
        child: Row(
          children: [
            // Ícono de señal
            Icon(
              _iconoSenial,
              color: conectado ? Colors.green : Colors.grey,
              size:  28,
            ),
            const SizedBox(width: 12),

            // Datos del dispositivo — Expanded para texto largo
            Expanded(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                mainAxisSize:       MainAxisSize.min,
                children: [
                  Text(
                    nombre,
                    style: const TextStyle(fontWeight: FontWeight.w600),
                    overflow: TextOverflow.ellipsis,
                  ),
                  Text(
                    '$ip · $mac',
                    style: TextStyle(
                      fontSize: 12,
                      color:    Colors.grey.shade600,
                    ),
                  ),
                ],
              ),
            ),

            const SizedBox(width: 8),

            // Señal en dBm
            Text(
              '$senialDbm dBm',
              style: TextStyle(
                fontSize: 12,
                color:    Colors.grey.shade600,
              ),
            ),

            const SizedBox(width: 8),

            // Indicador de estado
            Container(
              padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 4),
              decoration: BoxDecoration(
                color:        (conectado ? Colors.green : Colors.grey)
                              .withOpacity(0.15),
                borderRadius: BorderRadius.circular(12),
              ),
              child: Text(
                conectado ? 'Conectado' : 'Inactivo',
                style: TextStyle(
                  fontSize:   11,
                  color:      conectado ? Colors.green.shade700 : Colors.grey,
                  fontWeight: FontWeight.w600,
                ),
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

## `Stack` — superponer widgets en capas

```dart
// Stack coloca widgets uno encima del otro
// Useful para overlays, badges, imágenes con texto

Stack(
  alignment: Alignment.center,  // alineación por defecto para hijos no posicionados
  children: [
    // Capa inferior
    Container(width: 200, height: 200, color: Colors.blue.shade100),

    // Capa media
    const Icon(Icons.cloud, size: 80, color: Colors.blue),

    // Capa superior — posicionada con Positioned
    Positioned(
      top:   8,
      right: 8,
      child: Container(
        padding:    const EdgeInsets.all(4),
        decoration: const BoxDecoration(
          color: Colors.red,
          shape: BoxShape.circle,
        ),
        child: const Text('3', style: TextStyle(color: Colors.white, fontSize: 10)),
      ),
    ),
  ],
)
```

### Ejemplo aplicado — miniatura de servidor con badge de alertas

```dart
class AvatarServidorConBadge extends StatelessWidget {
  final String nombre;
  final int    alertas;
  final bool   activo;

  const AvatarServidorConBadge({
    super.key,
    required this.nombre,
    required this.alertas,
    required this.activo,
  });

  @override
  Widget build(BuildContext context) {
    return Stack(
      clipBehavior: Clip.none,
      children: [
        // Avatar principal
        Container(
          width:  56,
          height: 56,
          decoration: BoxDecoration(
            color:        activo
                ? Colors.indigo.shade100
                : Colors.grey.shade200,
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

        // Indicador de estado (punto verde/rojo)
        Positioned(
          bottom: 0,
          right:  0,
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

        // Badge de alertas — solo si hay alertas
        if (alertas > 0)
          Positioned(
            top:   -4,
            right: -4,
            child: Container(
              padding: const EdgeInsets.all(4),
              decoration: const BoxDecoration(
                color: Colors.orange,
                shape: BoxShape.circle,
              ),
              constraints: const BoxConstraints(minWidth: 18, minHeight: 18),
              child: Text(
                alertas > 9 ? '9+' : '$alertas',
                style: const TextStyle(
                  color:    Colors.white,
                  fontSize: 10,
                  fontWeight: FontWeight.bold,
                ),
                textAlign: TextAlign.center,
              ),
            ),
          ),
      ],
    );
  }
}
```

---

## `SizedBox`, `Padding` y `Align`

```dart
// SizedBox — espaciador o tamaño fijo
const SizedBox(height: 16)          // espaciador vertical
const SizedBox(width: 8)            // espaciador horizontal
const SizedBox(width: 200, height: 50)  // tamaño fijo
const SizedBox.expand()             // ocupa todo el espacio disponible
const SizedBox.shrink()             // sin tamaño (invisible)

// Padding — añade espacio alrededor del hijo
Padding(
  padding: const EdgeInsets.all(16),
  child:   Text('Con padding'),
)

// Align — posiciona el hijo dentro del espacio disponible
Align(
  alignment: Alignment.topRight,
  child: const Icon(Icons.settings),
)
// Otros: Alignment.center, topLeft, bottomCenter, bottomRight...

// Center — atajo para Align(alignment: Alignment.center)
Center(child: CircularProgressIndicator())

// FractionallySizedBox — porcentaje del padre
FractionallySizedBox(
  widthFactor:  0.8,   // 80% del ancho del padre
  child: ElevatedButton(onPressed: () {}, child: Text('80% de ancho')),
)
```

---

## `Wrap` — flujo automático de elementos

```dart
// Wrap coloca elementos en fila y salta a la siguiente línea automáticamente
// Útil para tags, chips, filtros

Wrap(
  spacing:     8,   // espacio horizontal entre elementos
  runSpacing:  8,   // espacio entre filas
  children: [
    Chip(label: Text('Flutter')),
    Chip(label: Text('Dart')),
    Chip(label: Text('Mobile')),
    Chip(label: Text('Cross-platform')),
    Chip(label: Text('Material Design 3')),
  ],
)
```

---

## Programa completo — pantalla de topología de red

```dart
import 'package:flutter/material.dart';

void main() => runApp(const AppTopologia());

class AppTopologia extends StatelessWidget {
  const AppTopologia({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Topología de Red',
      theme: ThemeData(
        colorScheme:  ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const PantallaTopologia(),
    );
  }
}

class InfoDispositivo {
  final String   nombre;
  final String   tipo;      // 'router', 'switch', 'server', 'endpoint'
  final String   ip;
  final bool     activo;
  final int      alertas;
  final List<String> etiquetas;

  const InfoDispositivo({
    required this.nombre,
    required this.tipo,
    required this.ip,
    required this.activo,
    this.alertas  = 0,
    this.etiquetas = const [],
  });
}

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

    return Scaffold(
      appBar: AppBar(
        title: const Text('Topología de Red'),
        actions: [
          IconButton(icon: const Icon(Icons.filter_list), onPressed: () {}),
          IconButton(icon: const Icon(Icons.refresh),    onPressed: () {}),
        ],
      ),
      body: Column(
        children: [
          // Resumen superior — Row con métricas
          Container(
            color:   Theme.of(context).colorScheme.surfaceContainerHighest,
            padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
            child: Row(
              children: [
                _ChipResumen(
                  icono:  Icons.hub,
                  texto:  '${dispositivos.length} dispositivos',
                  color:  Colors.indigo,
                ),
                const SizedBox(width: 8),
                _ChipResumen(
                  icono:  Icons.circle,
                  texto:  '${dispositivos.where((d) => d.activo).length} activos',
                  color:  Colors.green,
                ),
                const SizedBox(width: 8),
                _ChipResumen(
                  icono:  Icons.warning_amber,
                  texto:  '${dispositivos.fold(0, (s, d) => s + d.alertas)} alertas',
                  color:  Colors.orange,
                ),
              ],
            ),
          ),

          // Lista de dispositivos
          Expanded(
            child: ListView.separated(
              padding:       const EdgeInsets.symmetric(vertical: 8),
              itemCount:     dispositivos.length,
              separatorBuilder: (_, __) => const Divider(height: 1),
              itemBuilder: (context, index) {
                final d = dispositivos[index];
                return _FilaDispositivo(dispositivo: d);
              },
            ),
          ),
        ],
      ),
    );
  }
}

// Widget auxiliar — chip de resumen en la cabecera
class _ChipResumen extends StatelessWidget {
  final IconData icono;
  final String   texto;
  final Color    color;

  const _ChipResumen({
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
        Text(
          texto,
          style: TextStyle(
            fontSize:   12,
            color:      color,
            fontWeight: FontWeight.w600,
          ),
        ),
      ],
    );
  }
}

// Widget de fila — combina Row, Column, Stack y Wrap
class _FilaDispositivo extends StatelessWidget {
  final InfoDispositivo dispositivo;

  const _FilaDispositivo({required this.dispositivo});

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

          // Avatar con badge usando Stack
          AvatarServidorConBadge(
            nombre:  dispositivo.nombre,
            alertas: dispositivo.alertas,
            activo:  dispositivo.activo,
          ),

          const SizedBox(width: 12),

          // Datos del dispositivo usando Column + Wrap
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                // Nombre + tipo en Row
                Row(
                  children: [
                    Expanded(
                      child: Text(
                        dispositivo.nombre,
                        style: const TextStyle(fontWeight: FontWeight.bold),
                        overflow: TextOverflow.ellipsis,
                      ),
                    ),
                    Icon(_icono,
                        size:  16,
                        color: Colors.grey.shade500),
                  ],
                ),
                Text(
                  dispositivo.ip,
                  style: TextStyle(
                    fontSize: 12,
                    color:    Colors.grey.shade600,
                  ),
                ),
                const SizedBox(height: 6),

                // Etiquetas usando Wrap (flujo automático)
                Wrap(
                  spacing:    4,
                  runSpacing: 4,
                  children: dispositivo.etiquetas.map((tag) =>
                    Container(
                      padding: const EdgeInsets.symmetric(
                          horizontal: 6, vertical: 2),
                      decoration: BoxDecoration(
                        color:        Colors.indigo.shade50,
                        borderRadius: BorderRadius.circular(4),
                        border:       Border.all(
                            color: Colors.indigo.shade200),
                      ),
                      child: Text(
                        tag,
                        style: TextStyle(
                          fontSize:   10,
                          color:      Colors.indigo.shade700,
                          fontWeight: FontWeight.w500,
                        ),
                      ),
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

## Ejercicios propuestos

1. **Tarjeta de métrica de red** — Crea un widget `TarjetaTrafico` con `Stack`.
   El fondo es un `LinearProgressIndicator` vertical que muestra el uso del ancho
   de banda (0–100%). Encima, centrado, muestra el porcentaje en texto grande
   y el nombre de la interfaz (`eth0`, `wlan0`) en texto pequeño debajo.

2. **Fila de estadísticas** — Crea una `Row` con tres secciones separadas
   por `VerticalDivider`. Cada sección tiene el valor en grande y la etiqueta
   en pequeño. Usa `Expanded(flex: 1)` para que las tres sean del mismo ancho.
   Las métricas: `Latencia / 45ms`, `Paquetes / 12.4k`, `Uptime / 99.8%`.

3. **Grid de puertos** — Usa `Wrap` para mostrar 24 puertos de un switch.
   Cada puerto es un `Container` de 32×32 con bordes redondeados. El color
   indica el estado: verde = activo, gris = libre, naranja = error, rojo = bloqueado.
   Usa una lista de estados y `List.generate`.

4. **Layout responsivo** — Crea una pantalla que use `LayoutBuilder` para
   detectar el ancho disponible y mostrar: si ancho < 600px → `ListView` de tarjetas
   en una columna; si ancho ≥ 600px → `GridView` de 2 columnas.
   Pruébalo en emulador de teléfono y tablet.

---

## Resumen de la página 7

- Flutter usa un sistema de **constraints**: el padre da restricciones, el hijo decide su tamaño dentro de ellas.
- `Container` combina padding, margin, color y decoración. `color` y `decoration` no pueden usarse juntos — el color va dentro de `BoxDecoration`.
- `Column` apila verticalmente. `mainAxisAlignment` controla la distribución en el eje principal. `crossAxisAlignment` controla la alineación en el eje cruzado.
- `Row` alinea horizontalmente con los mismos ejes pero intercambiados. `Spacer()` es un elemento flexible vacío que empuja los demás al extremo.
- `Expanded` ocupa todo el espacio restante en su eje. `Flexible` ocupa solo lo que necesita hasta el máximo. `flex:` define la proporción entre elementos.
- `Stack` superpone widgets en capas. `Positioned` coloca un hijo en coordenadas exactas dentro del Stack.
- `SizedBox` crea espaciadores o tamaños fijos. `Padding` añade espacio alrededor. `Align` posiciona el hijo dentro del espacio disponible.
- `Wrap` fluye elementos en filas y salta automáticamente cuando no caben — ideal para tags y chips.
- `Expanded` dentro de una `Column` o `Row` sin altura/ancho acotado causa **overflow** — siempre asegúrate de que el eje principal esté acotado.

---

> **Siguiente página →** Página 8: Material 3 y tema — `MaterialApp`,
> `ThemeData`, `ColorScheme`, modo oscuro y componentes de scaffold.