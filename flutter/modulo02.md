# Tutorial Flutter — Página 2
## Módulo 1 · Dart
### Control de flujo: `if/else`, `switch` y bucles

---

## `if` / `else`

La forma más básica de tomar decisiones en Dart.
Primero el concepto puro, luego lo aplicamos a algo real.

```dart
void main() {
  // Forma básica
  int temperatura = 38;

  if (temperatura > 37.5) {
    print('Fiebre');
  } else if (temperatura > 36) {
    print('Normal');
  } else {
    print('Hipotermia');
  }

  // Operador ternario — para decisiones de una línea
  // condición ? valorSiVerdadero : valorSiFalso
  String estado = temperatura > 37.5 ? 'Con fiebre' : 'Sin fiebre';
  print(estado);

  // null-aware con ternario
  String? ciudad;
  String display = ciudad != null ? ciudad.toUpperCase() : 'Sin ciudad';

  // Forma más concisa con ??
  String display2 = ciudad?.toUpperCase() ?? 'Sin ciudad';
  print(display2);  // Sin ciudad
}
```

### `if` con null safety

```dart
void main() {
  String? nombre;

  // Sin verificar — error de compilación
  // print(nombre.length);  // ERROR: nombre puede ser null

  // Forma 1 — verificación explícita
  if (nombre != null) {
    print(nombre.length);  // aquí Dart sabe que nombre es String
  }

  // Forma 2 — operador ?.
  print(nombre?.length);  // null, sin excepción

  // Forma 3 — valor por defecto
  int longitud = nombre?.length ?? 0;
  print(longitud);  // 0
}
```

### Ejemplo aplicado — validador de señal WiFi

```dart
// Los routers reportan la señal en dBm (decibeles-miliwatt)
// Cuanto más cercano a 0, mejor la señal
String clasificarSenalWifi(int dbm) {
  if (dbm >= -50) {
    return 'Excelente ($dbm dBm)';
  } else if (dbm >= -60) {
    return 'Buena ($dbm dBm)';
  } else if (dbm >= -70) {
    return 'Aceptable ($dbm dBm)';
  } else if (dbm >= -80) {
    return 'Débil ($dbm dBm)';
  } else {
    return 'Sin cobertura ($dbm dBm)';
  }
}

void main() {
  final lecturas = [-45, -62, -71, -83, -95];

  for (final dbm in lecturas) {
    print(clasificarSenalWifi(dbm));
  }
}
```

Salida:
```
Excelente (-45 dBm)
Buena (-62 dBm)
Aceptable (-71 dBm)
Débil (-83 dBm)
Sin cobertura (-95 dBm)
```

---

## `switch`

`switch` compara un valor contra múltiples casos.
En Dart 3+ existe como **sentencia** (clásica) y como **expresión** (nueva).

### Switch sentencia — forma clásica

```dart
void main() {
  String codigoHttp = '404';

  switch (codigoHttp) {
    case '200':
      print('OK');
    case '201':
      print('Creado');
    case '400':
      print('Petición incorrecta');
    case '401':
      print('No autorizado');
    case '404':
      print('No encontrado');
    case '500':
      print('Error del servidor');
    default:
      print('Código desconocido');
  }
}
```

### Switch expresión — Dart 3+ (más potente)

El switch expresión **devuelve un valor** directamente.
Equivale al `when` de Kotlin:

```dart
void main() {
  // Switch expresión — asigna el resultado a una variable
  String codigoHttp = '404';

  String descripcion = switch (codigoHttp) {
    '200' => 'OK — solicitud exitosa',
    '201' => 'Created — recurso creado',
    '204' => 'No Content — sin contenido',
    '400' => 'Bad Request — datos inválidos',
    '401' => 'Unauthorized — sin autenticación',
    '403' => 'Forbidden — sin permiso',
    '404' => 'Not Found — recurso no existe',
    '500' => 'Internal Server Error',
    '503' => 'Service Unavailable',
    _     => 'Código HTTP desconocido',  // _ es el caso por defecto
  };

  print(descripcion);  // Not Found — recurso no existe
}
```

### Switch con múltiples valores y guards

```dart
void main() {
  int codigoNumerico = 404;

  // Múltiples valores en una rama con ||
  String categoria = switch (codigoNumerico) {
    200 || 201 || 204       => 'Éxito (2xx)',
    301 || 302 || 307       => 'Redirección (3xx)',
    400 || 401 || 403 || 404 => 'Error del cliente (4xx)',
    500 || 502 || 503       => 'Error del servidor (5xx)',
    _                       => 'Desconocido',
  };

  print(categoria);  // Error del cliente (4xx)

  // Guards — condición adicional con 'when'
  double temperatura = 39.2;

  String alerta = switch (temperatura) {
    double t when t >= 40.0 => '🚨 CRÍTICO — llame a emergencias',
    double t when t >= 38.5 => '🔴 FIEBRE ALTA — consulte médico',
    double t when t >= 37.5 => '🟡 FIEBRE LEVE — descanse',
    double t when t >= 36.0 => '🟢 NORMAL',
    _                       => '🔵 HIPOTERMIA — abrígese',
  };

  print(alerta);  // 🔴 FIEBRE ALTA — consulte médico
}
```

### Switch con tipos (pattern matching)

```dart
void main() {
  // switch puede verificar el TIPO del valor
  Object respuestaApi = {'id': 1, 'nombre': 'Teclado', 'precio': 89.99};

  String resultado = switch (respuestaApi) {
    Map<String, dynamic> m when m.containsKey('error') =>
        'Error: ${m['error']}',
    Map<String, dynamic> m =>
        'Producto: ${m['nombre']} — \$${m['precio']}',
    List<dynamic> lista =>
        '${lista.length} elementos en la lista',
    String texto =>
        'Texto recibido: $texto',
    _ =>
        'Respuesta desconocida',
  };

  print(resultado);  // Producto: Teclado — $89.99
}
```

### Ejemplo aplicado — clasificador de eventos de servidor

```dart
enum NivelLog { debug, info, warn, error, fatal }

class EventoLog {
  final NivelLog nivel;
  final String  mensaje;
  final DateTime timestamp;

  EventoLog(this.nivel, this.mensaje)
      : timestamp = DateTime.now();
}

String formatearEvento(EventoLog evento) {
  final icono = switch (evento.nivel) {
    NivelLog.debug => '🔍',
    NivelLog.info  => 'ℹ️ ',
    NivelLog.warn  => '⚠️ ',
    NivelLog.error => '❌',
    NivelLog.fatal => '💀',
  };

  final hora = '${evento.timestamp.hour.toString().padLeft(2,'0')}:'
               '${evento.timestamp.minute.toString().padLeft(2,'0')}:'
               '${evento.timestamp.second.toString().padLeft(2,'0')}';

  return '[$hora] $icono ${evento.nivel.name.toUpperCase()}: ${evento.mensaje}';
}

void main() {
  final eventos = [
    EventoLog(NivelLog.info,  'Servidor iniciado en puerto 8080'),
    EventoLog(NivelLog.debug, 'Conexión entrante desde 192.168.1.5'),
    EventoLog(NivelLog.warn,  'Memoria al 85% — considere escalar'),
    EventoLog(NivelLog.error, 'Timeout en base de datos principal'),
    EventoLog(NivelLog.fatal, 'Disco lleno — sistema detenido'),
  ];

  for (final evento in eventos) {
    print(formatearEvento(evento));
  }
}
```

---

## Bucles

### `for` clásico

```dart
void main() {
  // for con índice — cuando necesitas el número de iteración
  for (int i = 0; i < 5; i++) {
    print('Iteración $i');
  }

  // for con paso distinto
  for (int i = 0; i <= 100; i += 25) {
    print('Progreso: $i%');
  }

  // for decreciente
  for (int i = 5; i >= 1; i--) {
    print('Cuenta regresiva: $i');
  }
}
```

### `for-in` — iterar sobre colecciones

```dart
void main() {
  final protocolos = ['HTTP', 'HTTPS', 'FTP', 'SSH', 'SMTP'];

  // for-in — la forma idiomática para recorrer listas
  for (final protocolo in protocolos) {
    print(protocolo);
  }

  // forEach con lambda — alternativa funcional
  protocolos.forEach((p) => print(p.toLowerCase()));

  // for-in sobre un Map
  final puertos = {'HTTP': 80, 'HTTPS': 443, 'SSH': 22, 'FTP': 21};
  for (final entrada in puertos.entries) {
    print('${entrada.key} → puerto ${entrada.value}');
  }

  // for-in sobre caracteres de un String
  for (final caracter in 'Dart') {
    print(caracter);
  }
}
```

### `while` y `do-while`

```dart
void main() {
  // while — comprueba la condición ANTES de ejecutar
  int paquetes = 0;
  int buffer   = 1024;  // bytes disponibles

  while (buffer > 0) {
    final tamano = buffer > 256 ? 256 : buffer;
    paquetes++;
    buffer -= tamano;
    print('Paquete $paquetes: $tamano bytes (restante: $buffer)');
  }

  // do-while — ejecuta AL MENOS UNA VEZ antes de comprobar
  int reintentos = 0;
  bool conexionEstablecida = false;

  do {
    reintentos++;
    print('Intento de conexión #$reintentos...');
    // Simular que conecta en el 3er intento
    if (reintentos == 3) conexionEstablecida = true;
  } while (!conexionEstablecida && reintentos < 5);

  print(conexionEstablecida
      ? 'Conectado tras $reintentos intentos'
      : 'No se pudo conectar');
}
```

### `break` y `continue`

```dart
void main() {
  final paquetesRed = [64, 128, 512, -1, 256, 1024, -1, 32];
  //                              ↑         ↑
  //                         paquete malo  paquete malo

  // continue — salta el paquete corrupto y continúa
  print('=== Procesando con continue ===');
  for (final paquete in paquetesRed) {
    if (paquete < 0) {
      print('Paquete corrupto ignorado');
      continue;  // salta al siguiente
    }
    print('Procesando paquete de $paquete bytes');
  }

  // break — detiene completamente al encontrar error crítico
  print('\n=== Procesando con break ===');
  for (final paquete in paquetesRed) {
    if (paquete < 0) {
      print('Error crítico — deteniendo procesamiento');
      break;  // sale del bucle
    }
    print('Procesando paquete de $paquete bytes');
  }
}
```

### `break` con etiquetas — bucles anidados

```dart
void main() {
  // Escanear una matriz de servidores buscando uno disponible
  final datacenters = ['EU-WEST', 'EU-EAST', 'US-WEST'];
  final racks       = ['RACK-A', 'RACK-B', 'RACK-C'];

  String? servidorEncontrado;

  // La etiqueta marca el bucle exterior
  busqueda:
  for (final dc in datacenters) {
    for (final rack in racks) {
      final servidor = '$dc/$rack';
      print('Probando $servidor...');

      // Simular que EU-EAST/RACK-B está disponible
      if (dc == 'EU-EAST' && rack == 'RACK-B') {
        servidorEncontrado = servidor;
        break busqueda;  // sale de AMBOS bucles
      }
    }
  }

  print('Servidor asignado: $servidorEncontrado');
}
```

---

## Ejemplo combinado — monitor de conectividad

Este ejemplo usa `if`, `switch`, `for` y `while` juntos
para resolver un problema real:

```dart
enum EstadoConexion { conectado, desconectado, limitado, sinDns }

class InterfazRed {
  final String        nombre;      // eth0, wlan0, lo
  final String        ip;
  final EstadoConexion estado;
  final int           latenciaMs;
  final double        perdidaPct;  // % de paquetes perdidos

  const InterfazRed({
    required this.nombre,
    required this.ip,
    required this.estado,
    required this.latenciaMs,
    required this.perdidaPct,
  });
}

// Analiza una interfaz de red y emite diagnóstico
String diagnosticar(InterfazRed iface) {
  // switch para el estado base
  if (iface.estado == EstadoConexion.desconectado) {
    return '❌ ${iface.nombre}: Cable desconectado o interfaz caída';
  }

  // Construir lista de problemas detectados
  final problemas = <String>[];

  if (iface.perdidaPct > 10) {
    problemas.add('pérdida de paquetes ${iface.perdidaPct.toStringAsFixed(1)}%');
  }

  if (iface.latenciaMs > 200) {
    problemas.add('latencia alta ${iface.latenciaMs}ms');
  }

  // switch expresión para calidad de la conexión
  final calidad = switch (iface.estado) {
    EstadoConexion.conectado when iface.latenciaMs < 20 && iface.perdidaPct < 1
        => '🟢 Excelente',
    EstadoConexion.conectado when iface.latenciaMs < 80
        => '🟡 Aceptable',
    EstadoConexion.conectado
        => '🔴 Degradada',
    EstadoConexion.limitado
        => '🟠 Limitada (sin internet)',
    EstadoConexion.sinDns
        => '🟠 Sin DNS',
    EstadoConexion.desconectado
        => '❌ Sin conexión',
  };

  final resumen = problemas.isEmpty
      ? 'sin problemas detectados'
      : 'problemas: ${problemas.join(', ')}';

  return '$calidad | ${iface.nombre} (${iface.ip}) — $resumen';
}

void main() {
  final interfaces = [
    const InterfazRed(
      nombre: 'eth0', ip: '192.168.1.100',
      estado: EstadoConexion.conectado,
      latenciaMs: 12, perdidaPct: 0.0,
    ),
    const InterfazRed(
      nombre: 'wlan0', ip: '192.168.1.105',
      estado: EstadoConexion.conectado,
      latenciaMs: 245, perdidaPct: 15.3,
    ),
    const InterfazRed(
      nombre: 'vpn0', ip: '10.0.0.1',
      estado: EstadoConexion.limitado,
      latenciaMs: 80, perdidaPct: 0.5,
    ),
    const InterfazRed(
      nombre: 'eth1', ip: '0.0.0.0',
      estado: EstadoConexion.desconectado,
      latenciaMs: 0, perdidaPct: 100.0,
    ),
  ];

  print('=== Diagnóstico de red ===\n');

  // for-in para recorrer todas las interfaces
  for (final iface in interfaces) {
    print(diagnosticar(iface));
  }

  // Contar cuántas están en buen estado
  int buenas = 0;
  for (final iface in interfaces) {
    if (iface.estado == EstadoConexion.conectado &&
        iface.latenciaMs < 80 &&
        iface.perdidaPct < 5) {
      buenas++;
    }
  }

  print('\nInterfaces en buen estado: $buenas/${interfaces.length}');

  // while para reintentar la interfaz desconectada
  final desconectadas = interfaces
      .where((i) => i.estado == EstadoConexion.desconectado)
      .toList();

  if (desconectadas.isNotEmpty) {
    print('\n=== Reintentando interfaces caídas ===');
    int intento = 0;
    while (intento < 3) {
      intento++;
      print('Intento $intento/3: reconectando ${desconectadas.first.nombre}...');
      // En producción aquí iría la lógica real de reconexión
    }
    print('Sin éxito tras 3 intentos — revise el hardware');
  }
}
```

---

## Ejercicios propuestos

1. **Semáforo de carga de batería** — Escribe `String estadoBateria(int porcentaje, bool cargando)`
   que use `if/else` para devolver: `'🔴 Crítica'` si < 10%, `'🟠 Baja'` si < 30%,
   `'🟡 Media'` si < 60%, `'🟢 Buena'` si < 90%, `'⚡ Completa'` si >= 90%.
   Si está cargando, añade `' (cargando)'` al final con el operador ternario.

2. **Decodificador de teclas** — Usando `switch` expresión, convierte códigos
   de tecla numéricos a su acción: `27 → 'Escape'`, `13 → 'Enter'`, `32 → 'Espacio'`,
   `8 → 'Retroceso'`, `9 → 'Tab'`, `127 → 'Suprimir'`, cualquier otro entre
   32..126 → `'Carácter: ${String.fromCharCode(codigo)}'`, `_ → 'Tecla especial'`.

3. **Tabla de multiplicar invertida** — Usa bucles `for` anidados para imprimir
   una tabla de multiplicar del 1 al 5, pero en orden **descendente** (de 5×5 hasta 1×1).
   Usa `continue` para saltar las combinaciones donde ambos factores sean iguales.

4. **Escáner de puertos** — Simula un escáner de puertos con una lista de puertos conocidos
   `{22: 'SSH', 80: 'HTTP', 443: 'HTTPS', 3306: 'MySQL', 5432: 'PostgreSQL', 6379: 'Redis'}`.
   Itera sobre una lista de puertos a escanear `[80, 443, 8080, 3306, 9200, 6379]`.
   Usa `if` para detectar si el puerto está en la lista conocida, `switch` expresión
   para determinar si es `'seguro'`, `'inseguro'` o `'desconocido'`, y `continue`
   para saltar los menores de 1024 sin mostrarlos.

---

## Resumen de la página 2

- `if/else` evalúa condiciones en cascada. El operador ternario `condición ? a : b` asigna en una línea. `??` proporciona valor por defecto para nulos.
- `switch` sentencia es la forma clásica. `switch` expresión (Dart 3+) devuelve un valor con `=>` — equivalente al `when` de Kotlin. `_` es el caso por defecto.
- Las guards `when condición` en el `switch` expresión añaden filtros adicionales al patrón — muy útiles para rangos numéricos.
- `for (int i = 0; i < n; i++)` cuando necesitas el índice. `for (final item in coleccion)` para recorrer elementos directamente.
- `forEach` con lambda es la alternativa funcional a `for-in` — útil para cadenas de transformación.
- `while` comprueba antes de ejecutar. `do-while` ejecuta al menos una vez — ideal para reintentos.
- `continue` salta la iteración actual y continúa. `break` sale del bucle completamente.
- Las etiquetas `nombre:` en bucles permiten `break etiqueta` y `continue etiqueta` para controlar bucles anidados desde el interior.

---

> **Siguiente página →** Página 3: Funciones en Dart — declaración,
> parámetros nombrados, funciones de orden superior, closures y `typedef`.