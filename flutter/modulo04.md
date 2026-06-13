# Tutorial Flutter — Página 4
## Módulo 1 · Dart
### OOP: los 4 principios y cómo Dart los implementa

---

## Antes de los principios — anatomía de una clase

Una clase en Dart es una plantilla que describe los **datos** (propiedades)
y el **comportamiento** (métodos) de un tipo de objeto.

```
┌──────────────────────────────────────────────────────────┐
│                    Anatomía de una clase                 │
│                                                          │
│  class NombreClase {                                     │
│    // 1. Propiedades — datos que tiene cada objeto       │
│    String nombre;                                        │
│    int    edad;                                          │
│                                                          │
│    // 2. Constructor — cómo se crea el objeto            │
│    NombreClase({required this.nombre, required this.edad}│
│                                                          │
│    // 3. Getters — propiedades calculadas (solo lectura) │
│    bool get esMayorDeEdad => edad >= 18;                 │
│                                                          │
│    // 4. Setters — control de escritura con validación   │
│    set edad(int valor) {                                 │
│      if (valor < 0) throw ArgumentError('...');          │
│      _edad = valor;                                      │
│    }                                                     │
│                                                          │
│    // 5. Métodos — acciones que puede realizar           │
│    void saludar() => print('Hola, soy $nombre');         │
│                                                          │
│    // 6. toString — representación textual del objeto    │
│    @override                                             │
│    String toString() => 'NombreClase($nombre, $edad)';   │
│  }                                                       │
└──────────────────────────────────────────────────────────┘
```

### Clase básica completa — punto de partida

```dart
class Dispositivo {
  // 1. Propiedades
  final String id;
  final String nombre;
  String       ip;
  bool         _encendido = false;  // _ indica uso interno

  // 2. Constructor nombrado con parámetros nombrados
  Dispositivo({
    required this.id,
    required this.nombre,
    required this.ip,
  });

  // 3. Getter — propiedad derivada, solo lectura
  bool   get encendido => _encendido;
  String get estado    => _encendido ? 'activo' : 'inactivo';

  // 4. Setter — escritura controlada
  set estadoEncendido(bool valor) {
    _encendido = valor;
    print('$nombre: ${valor ? "encendido" : "apagado"}');
  }

  // 5. Métodos
  void conectar() {
    _encendido = true;
    print('$nombre conectado en $ip');
  }

  void desconectar() {
    _encendido = false;
    print('$nombre desconectado');
  }

  String resumen() => 'ID: $id | Nombre: $nombre | IP: $ip | Estado: $estado';

  // 6. toString
  @override
  String toString() => 'Dispositivo($nombre, $ip, $estado)';
}

void main() {
  // Crear una instancia
  final router = Dispositivo(
    id:     'DEV-001',
    nombre: 'router-principal',
    ip:     '192.168.1.1',
  );

  // Usar sus métodos y propiedades
  router.conectar();
  print(router.estado);       // activo
  print(router.resumen());
  print(router);              // llama toString() automáticamente

  router.estadoEncendido = false;  // usa el setter
  print(router.encendido);   // false
}
```

Salida:
```
router-principal conectado en 192.168.1.1
activo
ID: DEV-001 | Nombre: router-principal | IP: 192.168.1.1 | Estado: activo
Dispositivo(router-principal, 192.168.1.1, activo)
router-principal: apagado
false
```

### Tipos de constructores en Dart

```dart
class Servidor {
  final String hostname;
  final String ip;
  final int    puerto;
  final bool   usaSsl;

  // Constructor principal
  Servidor({
    required this.hostname,
    required this.ip,
    required this.puerto,
    this.usaSsl = false,
  });

  // Constructor nombrado — alternativa de creación con configuración predefinida
  Servidor.local()
      : hostname = 'localhost',
        ip       = '127.0.0.1',
        puerto   = 8080,
        usaSsl   = false;

  Servidor.produccion({required this.hostname, required this.ip})
      : puerto  = 443,
        usaSsl  = true;

  // Constructor factory — lógica de creación más compleja
  factory Servidor.desdeUrl(String url) {
    // Analiza una URL y extrae sus partes
    final uri = Uri.parse(url);
    return Servidor(
      hostname: uri.host,
      ip:       uri.host,        // simplificado para el ejemplo
      puerto:   uri.port != 0 ? uri.port : (uri.scheme == 'https' ? 443 : 80),
      usaSsl:   uri.scheme == 'https',
    );
  }

  @override
  String toString() =>
      '${usaSsl ? "https" : "http"}://$hostname:$puerto';
}

void main() {
  final s1 = Servidor(hostname: 'api.mi-app.com', ip: '10.0.1.5', puerto: 3000);
  final s2 = Servidor.local();
  final s3 = Servidor.produccion(hostname: 'api.mi-app.com', ip: '10.0.1.5');
  final s4 = Servidor.desdeUrl('https://pagos.mi-app.com:8443/v1');

  print(s1);  // http://api.mi-app.com:3000
  print(s2);  // http://localhost:8080
  print(s3);  // https://api.mi-app.com:443
  print(s4);  // https://pagos.mi-app.com:8443
}
```

---

## Los 4 principios de la OOP

La **Programación Orientada a Objetos** se basa en cuatro principios.
Antes de ver código, es fundamental entender qué problema resuelve cada uno:

| Principio | Pregunta que responde | Mecanismo en Dart |
|---|---|---|
| **Abstracción** | ¿Qué puede hacer este objeto? | `abstract class`, interfaces, métodos públicos |
| **Encapsulamiento** | ¿Quién puede acceder a sus datos? | `_privado`, getters/setters, constructores nombrados |
| **Herencia** | ¿Puede reutilizar el comportamiento de otro? | `extends`, `super`, `@override` |
| **Polimorfismo** | ¿Puede comportarse diferente según el contexto? | `@override`, `is`, `sealed class` |

Esta página cubre los cuatro principios aplicados a un dominio
de **infraestructura de red** — servidores, dispositivos y servicios.

---

## Principio 1 — Abstracción

> **"Muestra solo lo que el usuario necesita. Oculta cómo funciona por dentro."**

La abstracción separa el **qué** (interfaz pública) del **cómo** (implementación).
El código que usa la clase no necesita saber los detalles internos.

### Ejercicio 1a — Lo más simple: forma y área

```dart
// abstract class define el contrato — QUÉ puede hacer cualquier Forma
abstract class Forma {
  String get nombre;
  double calcularArea();     // cada forma lo implementa a su manera
  double calcularPerimetro();

  // Método concreto construido sobre la abstracción
  void describir() {
    print('$nombre — área: ${calcularArea().toStringAsFixed(2)}, '
          'perímetro: ${calcularPerimetro().toStringAsFixed(2)}');
  }
}

// Implementaciones concretas — el CÓMO es específico de cada clase
class Circulo extends Forma {
  final double radio;
  Circulo(this.radio);

  @override String get nombre => 'Círculo (r=$radio)';
  @override double calcularArea()      => 3.1416 * radio * radio;
  @override double calcularPerimetro() => 2 * 3.1416 * radio;
}

class Rectangulo extends Forma {
  final double ancho, alto;
  Rectangulo(this.ancho, this.alto);

  @override String get nombre => 'Rectángulo (${ancho}x$alto)';
  @override double calcularArea()      => ancho * alto;
  @override double calcularPerimetro() => 2 * (ancho + alto);
}

void main() {
  final formas = <Forma>[Circulo(5), Rectangulo(4, 7)];
  for (final f in formas) {
    f.describir();  // no importa qué tipo de Forma es
  }
}
```

Salida:
```
Círculo (r=5.0) — área: 78.54, perímetro: 31.42
Rectángulo (4.0x7.0) — área: 28.00, perímetro: 22.00
```

### Ejercicio 1b — Aplicado: dispositivos de red con abstracción real

```dart
// ABSTRACCIÓN: el usuario de DispositivoRed solo sabe QUÉ puede pedirle.
// No necesita saber cómo se implementa internamente cada dispositivo.
abstract class DispositivoRed {
  final String id;
  final String nombre;
  final String ip;

  DispositivoRed({
    required this.id,
    required this.nombre,
    required this.ip,
  });

  // Interfaz pública — QUÉ puede hacer cualquier dispositivo
  void encender();
  void apagar();
  Map<String, dynamic> obtenerMetricas();

  // Método concreto compartido — construido sobre la abstracción
  void verificarSalud() {
    final metricas = obtenerMetricas();  // llama al método abstracto
    print('[$nombre] Métricas: $metricas');
  }

  bool get activo;
  String get estado => activo ? 'en línea' : 'fuera de línea';

  @override
  String toString() => '$nombre ($ip) — $estado';
}

class Switch extends DispositivoRed {
  final int puertos;
  int  paquetesPorSeg;
  bool _activo = false;

  Switch({
    required super.id,
    required super.nombre,
    required super.ip,
    required this.puertos,
    this.paquetesPorSeg = 0,
  });

  @override bool get activo => _activo;
  @override void encender() { _activo = true;  print('$nombre: encendido ✅'); }
  @override void apagar()   { _activo = false; print('$nombre: apagado ❌');   }

  @override
  Map<String, dynamic> obtenerMetricas() => {
    'puertos':      puertos,
    'paquetes_seg': paquetesPorSeg,
  };
}

class PuntoAccesoWifi extends DispositivoRed {
  final String ssid;
  int  clientesConectados;
  bool _activo = false;

  PuntoAccesoWifi({
    required super.id,
    required super.nombre,
    required super.ip,
    required this.ssid,
    this.clientesConectados = 0,
  });

  @override bool get activo => _activo;
  @override void encender() { _activo = true;  print('$nombre: transmitiendo SSID "$ssid" ✅'); }
  @override void apagar()   { _activo = false; print('$nombre: SSID "$ssid" apagado ❌');        }

  @override
  Map<String, dynamic> obtenerMetricas() => {
    'ssid':     ssid,
    'clientes': clientesConectados,
  };
}

void main() {
  final sw = Switch(
    id: 'SW-001', nombre: 'core-switch',
    ip: '10.0.0.1', puertos: 48, paquetesPorSeg: 42000,
  );
  final ap = PuntoAccesoWifi(
    id: 'AP-001', nombre: 'sala-wifi',
    ip: '10.0.0.5', ssid: 'Oficina-5G', clientesConectados: 12,
  );

  sw.encender();
  ap.encender();

  // verificarSalud no sabe qué tipo de dispositivo es — solo usa la abstracción
  for (final dispositivo in [sw, ap]) {
    dispositivo.verificarSalud();
  }
}
```

---

## Principio 2 — Encapsulamiento

> **"Los datos internos solo se modifican a través de los métodos de la propia clase."**

El encapsulamiento protege el estado interno con visibilidad controlada.
En Dart los miembros privados empiezan con `_`.

### Ejercicio 2a — Lo más simple: cuenta bancaria

```dart
class CuentaBancaria {
  final String titular;
  double _saldo;  // privado — nadie lo modifica directamente

  CuentaBancaria(this.titular, double saldoInicial)
      : _saldo = saldoInicial;

  // Getter — lectura permitida, escritura no
  double get saldo => _saldo;

  // Los únicos caminos para modificar _saldo
  void depositar(double monto) {
    if (monto <= 0) throw ArgumentError('El monto debe ser positivo');
    _saldo += monto;
    print('Depósito de \$$monto. Nuevo saldo: \$$_saldo');
  }

  void retirar(double monto) {
    if (monto <= 0)      throw ArgumentError('El monto debe ser positivo');
    if (monto > _saldo)  throw StateError('Saldo insuficiente');
    _saldo -= monto;
    print('Retiro de \$$monto. Nuevo saldo: \$$_saldo');
  }
}

void main() {
  final cuenta = CuentaBancaria('Ana López', 500.0);

  cuenta.depositar(200.0);  // Depósito de $200.0. Nuevo saldo: $700.0
  cuenta.retirar(150.0);    // Retiro de $150.0.  Nuevo saldo: $550.0
  print(cuenta.saldo);      // 550.0

  // cuenta._saldo = 999999;  // ERROR — privado, Dart no lo permite
}
```

### Ejercicio 2b — Aplicado: servidor de métricas con historial y validación

```dart
class ServidorMetricas {
  // ENCAPSULAMIENTO: datos privados — nadie puede modificarlos directamente
  final String _hostname;
  double _cargaCpu     = 0.0;
  double _usoRamGb     = 0.0;
  int    _solicitudes  = 0;
  final  _historialCpu = <double>[];

  ServidorMetricas(this._hostname);

  // Getters — acceso de lectura controlado
  String get hostname    => _hostname;
  double get cargaCpu    => _cargaCpu;
  double get usoRamGb    => _usoRamGb;
  int    get solicitudes => _solicitudes;

  // Propiedad calculada — abstracción sobre los datos internos
  double get promedioCpu =>
      _historialCpu.isEmpty
          ? 0
          : _historialCpu.reduce((a, b) => a + b) / _historialCpu.length;

  bool get estaSaturado => _cargaCpu > 90 || _usoRamGb > 28;

  // Setter con validación — ENCAPSULAMIENTO en acción
  set cargaCpu(double valor) {
    if (valor < 0 || valor > 100) {
      throw RangeError('CPU debe estar entre 0 y 100: $valor');
    }
    _cargaCpu = valor;
    _historialCpu.add(valor);
    // Solo guardamos los últimos 60 valores
    if (_historialCpu.length > 60) _historialCpu.removeAt(0);
  }

  set usoRamGb(double valor) {
    if (valor < 0) throw ArgumentError('RAM no puede ser negativa');
    _usoRamGb = valor;
  }

  // Método controlado — la única forma de incrementar solicitudes
  void registrarSolicitud() => _solicitudes++;

  void resetearContador() {
    _solicitudes = 0;
    print('$_hostname: contador reseteado');
  }

  @override
  String toString() =>
      '$_hostname | CPU: ${_cargaCpu.toStringAsFixed(1)}% '
      'RAM: ${_usoRamGb.toStringAsFixed(1)}GB '
      'Req: $_solicitudes';
}

void main() {
  final servidor = ServidorMetricas('prod-api-01');

  // Solo podemos modificar el estado a través de los métodos
  servidor.cargaCpu  = 72.5;
  servidor.usoRamGb  = 18.3;
  servidor.registrarSolicitud();
  servidor.registrarSolicitud();
  servidor.registrarSolicitud();

  print(servidor);
  print('Promedio CPU: ${servidor.promedioCpu.toStringAsFixed(1)}%');
  print('Saturado: ${servidor.estaSaturado}');

  try {
    servidor.cargaCpu = 150;  // lanza RangeError
  } catch (e) {
    print('Error capturado: $e');
  }
}
```

---

## Principio 3 — Herencia

> **"Una clase reutiliza y especializa el comportamiento de otra sin repetir código."**

### Ejercicio 3a — Lo más simple: animales y sonido

```dart
// Clase base — comportamiento y datos comunes
class Animal {
  final String nombre;
  final int    edadAnios;

  Animal(this.nombre, this.edadAnios);

  // Método que cada subclase debe especializar
  String hacerSonido() => '...';

  // Método común — reutilizado sin cambios por todas las subclases
  void presentarse() {
    print('Soy $nombre, tengo $edadAnios años y hago: ${hacerSonido()}');
  }
}

// HERENCIA: Perro y Gato reutilizan Animal y lo especializan
class Perro extends Animal {
  Perro(super.nombre, super.edadAnios);

  @override
  String hacerSonido() => '¡Guau!';

  void buscarPelota() => print('$nombre busca la pelota 🎾');
}

class Gato extends Animal {
  Gato(super.nombre, super.edadAnios);

  @override
  String hacerSonido() => '¡Miau!';

  void trepar() => print('$nombre trepa al árbol 🌳');
}

void main() {
  final perro = Perro('Rex', 3);
  final gato  = Gato('Misu', 5);

  perro.presentarse();  // Soy Rex, tengo 3 años y hago: ¡Guau!
  gato.presentarse();   // Soy Misu, tengo 5 años y hago: ¡Miau!

  perro.buscarPelota();
  gato.trepar();
}
```

### Ejercicio 3b — Aplicado: jerarquía de nodos de red con `super`

```dart
// Clase base — define el comportamiento común de todos los nodos
class Nodo {
  final String id;
  final String nombre;
  bool _activo = false;

  Nodo({required this.id, required this.nombre});

  bool   get activo => _activo;
  String get estado => _activo ? 'operativo' : 'fuera de servicio';

  void encender() {
    _activo = true;
    print('$nombre [$id]: encendido');
  }

  void apagar() {
    _activo = false;
    print('$nombre [$id]: apagado');
  }

  @override
  String toString() => '$nombre — $estado';
}

// HERENCIA: Servidor reutiliza todo de Nodo y lo especializa
class Servidor extends Nodo {
  final String sistemaOperativo;
  final int    cpuCores;
  final double ramGb;
  final _servicios = <String>[];

  Servidor({
    required super.id,
    required super.nombre,
    required this.sistemaOperativo,
    required this.cpuCores,
    required this.ramGb,
  });

  @override
  void encender() {
    super.encender();           // reutiliza el código del padre
    print('  → $sistemaOperativo arrancando...');
    print('  → $cpuCores cores / ${ramGb}GB RAM disponibles');
  }

  void instalarServicio(String nombre) {
    _servicios.add(nombre);
    print('$nombre instalado en ${this.nombre}');
  }

  List<String> get servicios => List.unmodifiable(_servicios);

  @override
  String toString() =>
      'Servidor(${super.toString()}, '
      'SO: $sistemaOperativo, '
      'Servicios: ${_servicios.length})';
}

// HERENCIA: Enrutador especializa Nodo de una forma diferente
class Enrutador extends Nodo {
  final List<String> interfaces;
  int paquetesPorSeg = 0;

  Enrutador({
    required super.id,
    required super.nombre,
    required this.interfaces,
  });

  void registrarTrafico(int pps) {
    paquetesPorSeg = pps;
    if (pps > 100000) {
      print('⚠️ ${nombre}: tráfico alto ($pps pps)');
    }
  }

  @override
  String toString() =>
      'Enrutador(${super.toString()}, '
      '${interfaces.length} interfaces, '
      '$paquetesPorSeg pps)';
}

void main() {
  final servidor = Servidor(
    id: 'SRV-01', nombre: 'prod-web',
    sistemaOperativo: 'Ubuntu 24.04',
    cpuCores: 16, ramGb: 64,
  );

  final router = Enrutador(
    id: 'RTR-01', nombre: 'core-router',
    interfaces: ['eth0', 'eth1', 'eth2', 'eth3'],
  );

  servidor.encender();
  servidor.instalarServicio('nginx');
  servidor.instalarServicio('postgresql');

  router.encender();
  router.registrarTrafico(125000);

  print('\n$servidor');
  print(router);
}
```

---

## Principio 4 — Polimorfismo

> **"El mismo código trabaja con objetos de distintos tipos,
> ejecutando el comportamiento correcto para cada uno."**

### Ejercicio 4a — Lo más simple: lista de figuras con área distinta

```dart
// Reusamos la jerarquía de Forma del ejercicio 1a
abstract class Figura {
  String get nombre;
  double calcularArea();
}

class Cuadrado extends Figura {
  final double lado;
  Cuadrado(this.lado);
  @override String get nombre => 'Cuadrado';
  @override double calcularArea() => lado * lado;
}

class TrianguloRectangulo extends Figura {
  final double base, altura;
  TrianguloRectangulo(this.base, this.altura);
  @override String get nombre => 'Triángulo';
  @override double calcularArea() => (base * altura) / 2;
}

class CirculoPoli extends Figura {
  final double radio;
  CirculoPoli(this.radio);
  @override String get nombre => 'Círculo';
  @override double calcularArea() => 3.1416 * radio * radio;
}

// POLIMORFISMO: una sola función trabaja con cualquier Figura
void imprimirArea(Figura figura) {
  print('${figura.nombre}: ${figura.calcularArea().toStringAsFixed(2)} u²');
}

void main() {
  final figuras = <Figura>[
    Cuadrado(4),
    TrianguloRectangulo(6, 3),
    CirculoPoli(5),
  ];

  // Misma llamada — comportamiento diferente según el tipo real
  for (final f in figuras) {
    imprimirArea(f);
  }

  // Figura con mayor área — POLIMORFISMO con reduce
  final mayor = figuras.reduce((a, b) => a.calcularArea() > b.calcularArea() ? a : b);
  print('\nFigura más grande: ${mayor.nombre}');
}
```

Salida:
```
Cuadrado: 16.00 u²
Triángulo: 9.00 u²
Círculo: 78.54 u²

Figura más grande: Círculo
```

### Ejercicio 4b — Aplicado: polimorfismo con interfaces, `sealed class` y `is`

```dart
// Interfaz — define el CONTRATO sin implementación
abstract class Monitoreable {
  Map<String, double> leerMetricas();
  bool estaEnEstadoCritico();
  String get nombreServicio;
}

// POLIMORFISMO: tres clases distintas implementan la misma interfaz
class MonitorCpu implements Monitoreable {
  double _uso = 0;
  void simularCarga(double pct) => _uso = pct;

  @override String get nombreServicio => 'CPU';
  @override Map<String, double> leerMetricas() => {
    'uso_pct':    _uso,
    'temp_c':     45 + _uso * 0.4,
    'frecuencia': 3200 - (100 - _uso) * 5,
  };
  @override bool estaEnEstadoCritico() => _uso > 95;
}

class MonitorRed implements Monitoreable {
  double _mbps = 0;
  void simularTrafico(double mbps) => _mbps = mbps;

  @override String get nombreServicio => 'Red';
  @override Map<String, double> leerMetricas() => {
    'mbps_entrada': _mbps * 0.6,
    'mbps_salida':  _mbps * 0.4,
    'latencia_ms':  2 + _mbps * 0.01,
  };
  @override bool estaEnEstadoCritico() => _mbps > 900;
}

class MonitorDisco implements Monitoreable {
  double _usadoGb = 0;
  final double _totalGb = 500;
  void simularUso(double gb) => _usadoGb = gb;

  @override String get nombreServicio => 'Disco';
  @override Map<String, double> leerMetricas() => {
    'usado_gb':  _usadoGb,
    'libre_gb':  _totalGb - _usadoGb,
    'uso_pct':   (_usadoGb / _totalGb) * 100,
  };
  @override bool estaEnEstadoCritico() => (_usadoGb / _totalGb) > 0.95;
}

// Esta función no sabe ni le importa qué tipo de monitor recibe
void verificarMonitor(Monitoreable monitor) {
  final critico = monitor.estaEnEstadoCritico();
  final prefijo = critico ? '🔴 CRÍTICO' : '🟢 OK';
  print('$prefijo — ${monitor.nombreServicio}:');
  monitor.leerMetricas().forEach(
    (k, v) => print('  $k: ${v.toStringAsFixed(2)}'),
  );
}

// POLIMORFISMO con sealed class — switch exhaustivo y seguro
sealed class EstadoServicio { const EstadoServicio(); }
class Iniciando     extends EstadoServicio { const Iniciando(); }
class Activo        extends EstadoServicio {
  final int conexiones;
  const Activo(this.conexiones);
}
class Degradado     extends EstadoServicio {
  final String motivo;
  const Degradado(this.motivo);
}
class Caido         extends EstadoServicio {
  final String error;
  Caido(this.error);
}
class Mantenimiento extends EstadoServicio {
  final String responsable;
  const Mantenimiento(this.responsable);
}

String describirEstado(EstadoServicio estado) => switch (estado) {
  Iniciando()                              => '🔄 Iniciando...',
  Activo(conexiones: final c)              => '✅ Activo ($c conexiones)',
  Degradado(motivo: final m)               => '⚠️ Degradado: $m',
  Caido(error: final e)                    => '❌ Caído: $e',
  Mantenimiento(responsable: final r)      => '🔧 En mantenimiento por $r',
};

void main() {
  // POLIMORFISMO con interfaz común
  final cpu   = MonitorCpu()..simularCarga(97.5);
  final red   = MonitorRed()..simularTrafico(450);
  final disco = MonitorDisco()..simularUso(480);

  for (final m in <Monitoreable>[cpu, red, disco]) {
    verificarMonitor(m);
    print('');
  }

  // POLIMORFISMO con sealed class
  final estados = <EstadoServicio>[
    const Iniciando(),
    const Activo(342),
    const Degradado('Latencia alta en BD'),
    Caido('OutOfMemoryError'),
    const Mantenimiento('ops-team@empresa.com'),
  ];

  print('=== Estados de servicio ===');
  for (final e in estados) {
    print(describirEstado(e));
  }
}
```

---

## Mixins — reutilización horizontal

Un `mixin` añade capacidades a una clase sin herencia.
No responde a "es un", sino a "puede hacer":

```dart
mixin Loggable {
  String get logTag => runtimeType.toString();

  void logInfo(String msg)  => print('ℹ️  [$logTag] $msg');
  void logWarn(String msg)  => print('⚠️  [$logTag] $msg');
  void logError(String msg) => print('❌ [$logTag] $msg');
}

mixin Alertable {
  final _alertas = <String>[];

  void emitirAlerta(String msg) {
    _alertas.add('[${DateTime.now().toIso8601String()}] $msg');
    print('🚨 ALERTA: $msg');
  }

  List<String> get historialAlertas => List.unmodifiable(_alertas);
}

// Mixin con restricción — solo para subclases de Nodo
mixin Reiniciable on Nodo {
  int _reinicios = 0;
  int get reinicios => _reinicios;

  Future<void> reiniciar({String motivo = 'manual'}) async {
    _reinicios++;
    apagar();
    print('  Esperando antes de reiniciar ($motivo)...');
    await Future.delayed(const Duration(milliseconds: 100));
    encender();
    print('  Reinicio #$_reinicios completado');
  }
}

// Combina herencia + tres mixins
class ServicioBalanceador extends Nodo with Loggable, Alertable, Reiniciable {
  final List<String> backends;
  int _solicitudesTotal = 0;
  int _erroresTotal     = 0;

  ServicioBalanceador({
    required super.id,
    required super.nombre,
    required this.backends,
  });

  Future<String> enrutar(String peticion) async {
    _solicitudesTotal++;
    final backend = backends[_solicitudesTotal % backends.length];
    logInfo('Enrutando "$peticion" → $backend');

    if (_solicitudesTotal % 5 == 0) {
      _erroresTotal++;
      emitirAlerta('Backend $backend no responde');
      return 'ERROR';
    }
    return 'OK → $backend';
  }

  double get tasaError =>
      _solicitudesTotal == 0 ? 0 : _erroresTotal / _solicitudesTotal * 100;
}

void main() async {
  final balanceador = ServicioBalanceador(
    id: 'LB-001', nombre: 'nginx-lb',
    backends: ['app-01:8080', 'app-02:8080', 'app-03:8080'],
  );

  balanceador.encender();
  balanceador.logInfo('Balanceador listo con ${balanceador.backends.length} backends');

  for (var i = 1; i <= 7; i++) {
    final resp = await balanceador.enrutar('/api/productos');
    print('  Respuesta: $resp');
  }

  if (balanceador.tasaError > 10) {
    balanceador.logWarn('Tasa de error: ${balanceador.tasaError.toStringAsFixed(1)}%');
    await balanceador.reiniciar(motivo: 'tasa de error > 10%');
  }

  print('\n=== Historial de alertas ===');
  balanceador.historialAlertas.forEach(print);
  print('Reinicios totales: ${balanceador.reinicios}');
}
```

---

## Resumen visual — los 4 principios

```
┌─────────────────────────────────────────────────────────────────┐
│  ABSTRACCIÓN                │  ENCAPSULAMIENTO                  │
│  abstract class             │  _privado (guión bajo)            │
│  interfaces (implements)    │  getters y setters                │
│  métodos públicos           │  constructor privado + factory    │
│  mixins                     │  validación en setter             │
├─────────────────────────────────────────────────────────────────┤
│  HERENCIA                   │  POLIMORFISMO                     │
│  extends + super            │  @override                        │
│  super.metodo()             │  List<TipoBase> con subtipos      │
│  @override                  │  implements (interfaz común)      │
│  mixin on ClaseBase         │  switch (sealed class)            │
│                             │  is para verificar tipo           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Ejercicios propuestos

### Abstracción

**Ejercicio A1 — Básico** Crea `abstract class Vehiculo` con
`String get tipo`, `double calcularConsumo(double km)` y un método concreto
`void reportarConsumo(double km)` que imprima el resultado.
Implementa `Automovil` (8 L/100km) y `Motocicleta` (4 L/100km).

**Ejercicio A2 — Intermedio** Diseña `abstract class Exportador` con
`String get formato` y `String exportar(List<Map<String,dynamic>> datos)`.
Implementa `ExportadorCsv`, `ExportadorJson` y `ExportadorHtml`.
Escribe una función `void generarReporte(Exportador e, List<Map<String,dynamic>> datos)`
que use la abstracción para generar el archivo sin saber el formato de destino.

---

### Encapsulamiento

**Ejercicio E1 — Básico** Crea `class Termostato` con `_temperatura` privada,
un getter `temperatura` y un setter que solo permita valores entre 15 y 30°C
(lanza `RangeError` si no). Añade `void subir(int grados)` y `void bajar(int grados)`
como únicos métodos de modificación.

**Ejercicio E2 — Intermedio** Implementa `class CarritoDeCompras` con una lista
privada `_items` de tipo `Map<String, dynamic>`. Expón `int get cantidadItems`,
`double get total` y `List<Map<String,dynamic>> get items` como solo lectura.
Añade `void agregar(String nombre, double precio, int cantidad)` con validación,
`bool eliminar(String nombre)` y `void aplicarDescuento(double porcentaje)`
que valide que el descuento esté entre 0 y 100.

---

### Herencia

**Ejercicio H1 — Básico** Crea `class Empleado` con `nombre`, `salarioBase` y
`double calcularPago()`. Extiéndela con `EmpleadoFijo` (pago = salarioBase) y
`EmpleadoComision` (pago = salarioBase + ventas × 0.10). Usa `super.nombre`
en el `toString()` de cada subclase. Muestra los pagos de tres empleados en una lista.

**Ejercicio H2 — Intermedio** Construye la jerarquía `Notificacion → NotificacionEmail`
y `Notificacion → NotificacionSms`. La clase base tiene `destinatario`, `asunto`,
`_enviada` (privado) y `void enviar()` que marca `_enviada = true` e imprime un log.
Cada subclase sobreescribe `enviar()` llamando a `super.enviar()` y añadiendo detalles
específicos (el email agrega `servidorSmtp` y puerto; el SMS agrega `codigoPais`).
Añade `bool get enviada` y protege el reenvío lanzando `StateError` si ya fue enviada.

---

### Polimorfismo

**Ejercicio P1 — Básico** Crea una lista `List<Animal>` con instancias de
`Perro`, `Gato` y `Pajaro` (del ejercicio 3a ampliado con Pajaro).
Escribe `void hacerRuido(Animal a)` e itera la lista llamando a esa función.
Filtra con `is` solo los que son `Perro` y llama a `buscarPelota()`.

**Ejercicio P2 — Intermedio** Define `sealed class ResultadoPago` con
`Aprobado(String codigoTransaccion, double monto)`,
`Rechazado(String motivo, int codigoError)` y
`Pendiente(String referencia, Duration tiempoEstimado)`.
Escribe `String interpretarResultado(ResultadoPago r)` usando `switch` exhaustivo.
Simula una lista de 5 pagos con distintos resultados y muestra el resumen de cada uno.

---

## Resumen de la página 4

- Una clase en Dart tiene propiedades, constructor, getters, setters, métodos y `toString`. Los miembros con `_` son privados al archivo. Los constructores nombrados y `factory` encapsulan variantes de creación.
- **Abstracción** — `abstract class` e interfaces definen el *qué* sin el *cómo*. El código que usa la clase solo conoce la interfaz pública, no los detalles internos.
- **Encapsulamiento** — `_privado` protege el estado. Getters y setters controlan el acceso con validación. Los constructores `factory` privados encapsulan la lógica de creación.
- **Herencia** — `extends` reutiliza código del padre. `super.metodo()` invoca la implementación del padre. `@override` personaliza el comportamiento en la subclase. `super.campo` en el constructor pasa argumentos al padre.
- **Polimorfismo** — Una lista `List<Nodo>` acepta `Servidor`, `Enrutador` y cualquier subclase. El mismo código produce comportamiento diferente según el tipo real. `is` hace smart cast automático. `sealed class` garantiza exhaustividad en `switch`.
- Los `mixin` añaden capacidades horizontales sin jerarquías de herencia. `mixin on ClaseBase` restringe el mixin a subclases específicas.

---

> **Siguiente página →** Página 5: `Future`, `async`/`await` y `Stream` —
> programación asíncrona y flujos de datos en Dart.