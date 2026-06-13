# Tutorial Flutter — Página 3
## Módulo 1 · Dart
### Funciones: declaración, parámetros, orden superior, closures y `typedef`

---

## ¿Qué es una función en Dart?

Una función es un bloque de código con nombre que encapsula una tarea concreta,
puede recibir datos de entrada (parámetros) y devolver un resultado (valor de retorno).

En Dart **todo es un objeto**, incluidas las funciones.
Esto significa que una función puede ser:

- Asignada a una variable
- Pasada como argumento a otra función
- Devuelta como valor de retorno

Esta característica es la base de la programación funcional en Dart
y de gran parte del sistema de widgets de Flutter.

```
┌─────────────────────────────────────────────┐
│              Anatomía de una función        │
│                                             │
│  TipoRetorno nombre(parámetros) {           │
│    // cuerpo                                │
│    return valor;                            │
│  }                                          │
│                                             │
│  • TipoRetorno → int, String, void, etc.    │
│  • nombre      → identificador camelCase    │
│  • parámetros  → pueden ser opcionales      │
│  • return      → obligatorio si no es void  │
└─────────────────────────────────────────────┘
```

---

## 1. Función básica con tipo de retorno explícito

La forma más directa: recibe parámetros posicionales y retorna un valor.

```dart
// Sintaxis completa — preferida para funciones públicas
int sumar(int a, int b) {
  return a + b;
}

// Sintaxis de flecha — cuando el cuerpo es una sola expresión
int multiplicar(int a, int b) => a * b;

// void — cuando no se devuelve nada
void imprimirSeparador(String titulo) {
  print('─── $titulo ───');
}

void main() {
  print(sumar(5, 3));          // 8
  print(multiplicar(4, 6));    // 24
  imprimirSeparador('Inicio'); // ─── Inicio ───
}
```

### Inferencia de tipo de retorno

```dart
// Dart puede inferir el tipo de retorno, pero es buena práctica declararlo
// explícitamente en funciones públicas para mejorar la legibilidad.

// Con tipo explícito — recomendado
String formatearPrecio(double precio) => '\$${precio.toStringAsFixed(2)}';

// Sin tipo — Dart infiere que retorna String
formatearPrecioSinTipo(double precio) => '\$${precio.toStringAsFixed(2)}';

void main() {
  print(formatearPrecio(1299.9));  // $1299.90
}
```

---

## 2. Parámetros posicionales opcionales `[ ]`

Los parámetros entre corchetes son opcionales.
Si no se pasan, son `null` (o el valor por defecto que se defina).

```dart
// El tercer parámetro es opcional — puede omitirse al llamar
String construirUrl(String host, String ruta, [int? puerto]) {
  if (puerto != null) {
    return 'https://$host:$puerto$ruta';
  }
  return 'https://$host$ruta';
}

// Con valor por defecto — evita el chequeo de null
String construirUrlV2(String host, String ruta, [int puerto = 443]) {
  return 'https://$host:$puerto$ruta';
}

void main() {
  print(construirUrl('api.ejemplo.com', '/usuarios'));          // https://api.ejemplo.com/usuarios
  print(construirUrl('api.ejemplo.com', '/usuarios', 8080));   // https://api.ejemplo.com:8080/usuarios
  print(construirUrlV2('api.ejemplo.com', '/productos'));       // https://api.ejemplo.com:443/productos
}
```

---

## 3. Parámetros nombrados `{ }`

Los parámetros nombrados se pasan por nombre, no por posición.
Son el estándar en Flutter — todos los widgets los usan.

```dart
// required → el parámetro es obligatorio
// sin required → es opcional (necesita valor por defecto o ser nullable)
void configurarServidor({
  required String host,
  required int    puerto,
  bool   ssl        = true,
  int    timeoutSeg = 30,
}) {
  final protocolo = ssl ? 'https' : 'http';
  print('Conectando a $protocolo://$host:$puerto (timeout: ${timeoutSeg}s)');
}

void main() {
  // Los nombrados pueden pasarse en cualquier orden
  configurarServidor(
    host:       'db.miempresa.com',
    puerto:     5432,
    ssl:        false,
    timeoutSeg: 60,
  );

  // Solo los obligatorios — los opcionales toman su valor por defecto
  configurarServidor(
    host:   'api.miempresa.com',
    puerto: 443,
  );
}
```

Salida:
```
Conectando a http://db.miempresa.com:5432 (timeout: 60s)
Conectando a https://api.miempresa.com:443 (timeout: 30s)
```

### ¿Posicionales o nombrados?

```
┌──────────────────────┬─────────────────────────────────────────────┐
│ Posicionales         │ Nombrados                                   │
├──────────────────────┼─────────────────────────────────────────────┤
│ El orden importa     │ El orden NO importa                         │
│ Legibles si son 1-2  │ Legibles con 3+ parámetros                  │
│ sumar(3, 5)          │ crearUsuario(nombre: 'Ana', rol: 'admin')   │
│ Típico en utilidades │ Obligatorio en widgets Flutter              │
└──────────────────────┴─────────────────────────────────────────────┘
```

---

## 4. Funciones como objetos — asignación a variables

Como las funciones son objetos de primera clase, pueden almacenarse en variables.
El tipo de una función es `TipoRetorno Function(TipoParam1, TipoParam2, ...)`.

```dart
int doblar(int n)  => n * 2;
int triplicar(int n) => n * 3;

void main() {
  // La variable 'operacion' tiene tipo: int Function(int)
  int Function(int) operacion;

  operacion = doblar;
  print(operacion(5));     // 10

  operacion = triplicar;
  print(operacion(5));     // 15

  // Lista de funciones
  final transformaciones = <int Function(int)>[doblar, triplicar];
  for (final fn in transformaciones) {
    print(fn(10));         // 20, luego 30
  }
}
```

---

## 5. Funciones anónimas (lambdas)

Una función anónima no tiene nombre — se define y usa en el mismo lugar.
En Dart se llaman **lambdas** o **closures** cuando capturan el entorno.

```dart
void main() {
  // Lambda asignada a una variable
  final cuadrado = (int n) => n * n;
  print(cuadrado(7));  // 49

  // Lambda de cuerpo completo
  final calcularDescuento = (double precio, double pct) {
    final descuento = precio * (pct / 100);
    return precio - descuento;
  };
  print(calcularDescuento(100.0, 15.0));  // 85.0

  // Lambda en línea — pasada directamente como argumento
  final numeros = [3, 1, 4, 1, 5, 9, 2, 6];
  numeros.sort((a, b) => b.compareTo(a));  // orden descendente
  print(numeros);  // [9, 6, 5, 4, 3, 2, 1, 1]
}
```

---

## 6. Funciones de orden superior

Una función de orden superior **recibe** o **devuelve** otra función.
Son el núcleo de las operaciones funcionales sobre colecciones: `map`, `where`, `reduce`, `forEach`.

### `map` — transforma cada elemento

```dart
void main() {
  final precios = [29.99, 49.50, 15.00, 99.99];

  // map devuelve un Iterable con cada elemento transformado
  final preciosConIva = precios.map((p) => p * 1.15);
  print(preciosConIva.toList());
  // [34.4885, 56.925, 17.25, 114.9885]

  // map sobre Strings
  final endpoints = ['/usuarios', '/productos', '/pedidos'];
  final urls = endpoints.map((e) => 'https://api.ejemplo.com$e');
  print(urls.toList());
  // [https://api.ejemplo.com/usuarios, ...]
}
```

### `where` — filtra elementos

```dart
void main() {
  final temperaturas = [36.1, 37.8, 39.2, 36.5, 38.7, 35.9];

  final conFiebre = temperaturas.where((t) => t > 37.5);
  print(conFiebre.toList());  // [37.8, 39.2, 38.7]

  final normales = temperaturas.where((t) => t >= 36.0 && t <= 37.5);
  print(normales.toList());   // [36.1, 36.5]
}
```

### `reduce` y `fold` — acumulan un resultado

```dart
void main() {
  final ventas = [1500.0, 2300.0, 980.0, 3100.0, 750.0];

  // reduce — combina todos los elementos en uno
  final total = ventas.reduce((acum, venta) => acum + venta);
  print('Total: \$${total.toStringAsFixed(2)}');  // Total: $8630.00

  // fold — como reduce pero con valor inicial (más seguro con listas vacías)
  final totalFold = ventas.fold(0.0, (acum, venta) => acum + venta);
  print('Total (fold): \$${totalFold.toStringAsFixed(2)}');

  // Encontrar el máximo
  final maximo = ventas.reduce((a, b) => a > b ? a : b);
  print('Mayor venta: \$$maximo');  // Mayor venta: $3100.0
}
```

### Función que recibe otra función como parámetro

```dart
// Tipo explícito del parámetro función: bool Function(int)
List<int> filtrar(List<int> lista, bool Function(int) criterio) {
  return lista.where(criterio).toList();
}

bool esPar(int n)    => n % 2 == 0;
bool esGrande(int n) => n > 100;

void main() {
  final datos = [12, 7, 200, 4, 150, 33, 88, 301];

  print(filtrar(datos, esPar));      // [12, 200, 4, 150, 88]
  print(filtrar(datos, esGrande));   // [200, 150, 301]

  // Lambda en línea como argumento
  print(filtrar(datos, (n) => n % 3 == 0));  // [12, 33, 300] → [12, 33]
}
```

### Función que devuelve otra función

```dart
// Fábrica de multiplicadores — devuelve una función configurada
int Function(int) crearMultiplicador(int factor) {
  return (int n) => n * factor;
}

void main() {
  final doble    = crearMultiplicador(2);
  final triple   = crearMultiplicador(3);
  final decuplo  = crearMultiplicador(10);

  print(doble(5));    // 10
  print(triple(5));   // 15
  print(decuplo(5));  // 50

  // Útil para generar validadores configurables
  bool Function(double) crearValidadorPrecio(double min, double max) {
    return (precio) => precio >= min && precio <= max;
  }

  final esEconomico  = crearValidadorPrecio(0, 50);
  final esPremium    = crearValidadorPrecio(200, double.infinity);

  print(esEconomico(35.0));   // true
  print(esPremium(249.99));   // true
  print(esPremium(45.0));     // false
}
```

---

## 7. Closures — capturar el entorno

Una **closure** es una función anónima que "recuerda" las variables
del ámbito donde fue creada, incluso después de que ese ámbito haya terminado.

```dart
// Ejemplo básico — el contador "recuerda" su valor
Function crearContador() {
  int cuenta = 0;  // esta variable vive dentro de la closure

  return () {
    cuenta++;
    return cuenta;
  };
}

void main() {
  final contadorA = crearContador();
  final contadorB = crearContador();  // instancia independiente

  print(contadorA());  // 1
  print(contadorA());  // 2
  print(contadorA());  // 3
  print(contadorB());  // 1 — su propio estado, no afecta a A
  print(contadorA());  // 4
}
```

### Closure con acumulador de log

```dart
// Función que devuelve una closure de registro
List<String> Function(String) crearLogger(String modulo) {
  final registros = <String>[];

  return (String mensaje) {
    final entrada = '[${modulo.toUpperCase()}] ${DateTime.now().toIso8601String()}: $mensaje';
    registros.add(entrada);
    return List.unmodifiable(registros);
  };
}

void main() {
  final logAuth = crearLogger('auth');
  final logApi  = crearLogger('api');

  logAuth('Usuario admin inició sesión');
  logAuth('Token generado correctamente');
  logApi('GET /usuarios → 200 OK');

  final historialAuth = logAuth('Sesión cerrada');
  print('Registros de auth: ${historialAuth.length}');  // 3
}
```

---

## 8. `typedef` — nombrar tipos de funciones

`typedef` crea un alias para un tipo de función complejo,
mejorando la legibilidad y reutilizando el tipo en múltiples lugares.

```dart
// Sin typedef — verboso y difícil de leer
void procesarEventos(List<Map<String, dynamic>> eventos,
    bool Function(Map<String, dynamic>) filtro,
    void Function(Map<String, dynamic>) accion) { /* ... */ }

// Con typedef — claro y reutilizable
typedef Evento      = Map<String, dynamic>;
typedef FiltroEvento = bool Function(Evento);
typedef AccionEvento = void Function(Evento);

void procesarEventos2(
  List<Evento>  eventos,
  FiltroEvento  filtro,
  AccionEvento  accion,
) {
  for (final evento in eventos) {
    if (filtro(evento)) {
      accion(evento);
    }
  }
}

void main() {
  final eventos = [
    {'tipo': 'click', 'elemento': 'botonPagar',  'usuario': 'u001'},
    {'tipo': 'scroll', 'elemento': 'listaProductos', 'usuario': 'u002'},
    {'tipo': 'click', 'elemento': 'menuHamburguesa', 'usuario': 'u001'},
  ];

  // Solo procesar los clicks
  procesarEventos2(
    eventos,
    (e) => e['tipo'] == 'click',
    (e) => print('Click en: ${e['elemento']} por usuario ${e['usuario']}'),
  );
}
```

Salida:
```
Click en: botonPagar por usuario u001
Click en: menuHamburguesa por usuario u001
```

---

## 9. Funciones recursivas

Una función recursiva se llama a sí misma para resolver un problema
dividiéndolo en versiones más pequeñas del mismo problema.

```dart
// Factorial — caso base y caso recursivo
int factorial(int n) {
  if (n <= 1) return 1;       // caso base
  return n * factorial(n - 1); // llamada recursiva
}

// Fibonacci — árbol de llamadas
int fibonacci(int n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

// Búsqueda en árbol de directorios (simulado)
int contarArchivos(Map<String, dynamic> directorio) {
  int total = 0;
  for (final entrada in directorio.entries) {
    if (entrada.value is Map) {
      // Es un subdirectorio — llamada recursiva
      total += contarArchivos(entrada.value as Map<String, dynamic>);
    } else {
      total++;  // Es un archivo
    }
  }
  return total;
}

void main() {
  print(factorial(6));    // 720
  print(fibonacci(10));   // 55

  final sistemaArchivos = {
    'src': {
      'controllers': {'user_controller.dart': true, 'auth_controller.dart': true},
      'models':      {'user.dart': true},
      'services':    {'api_service.dart': true, 'cache_service.dart': true},
    },
    'test': {'widget_test.dart': true},
    'pubspec.yaml': true,
    'README.md':    true,
  };

  print('Total de archivos: ${contarArchivos(sistemaArchivos)}');  // 9
}
```

---

## 10. Funciones `async` — vista previa

Las funciones asíncronas retornan un `Future<T>` y permiten usar `await`.
Se estudian en profundidad en el módulo de asincronía, pero es útil
reconocer su sintaxis desde ahora.

```dart
import 'dart:io';

// async → la función retorna Future<String>
Future<String> obtenerIpPublica() async {
  // await suspende la ejecución hasta que el Future se resuelva
  await Future.delayed(Duration(milliseconds: 200));  // simula latencia
  return '203.0.113.42';
}

// void main también puede ser async
void main() async {
  print('Consultando IP...');
  final ip = await obtenerIpPublica();
  print('IP pública: $ip');
  print('Consulta completada');
}
```

---

## Ejemplo de intensidad media — Pipeline de procesamiento de datos

Combina funciones de orden superior, closures, `typedef` y funciones puras
para construir un pipeline de transformación y validación de registros.

```dart
// ─── Tipos ───────────────────────────────────────────────────────────────────

typedef Registro     = Map<String, dynamic>;
typedef Transformador = Registro Function(Registro);
typedef Validador     = bool Function(Registro);
typedef Reporte       = Map<String, int>;

// ─── Transformadores — funciones puras ────────────────────────────────────────

Registro normalizarNombre(Registro r) => {
  ...r,
  'nombre': (r['nombre'] as String).trim().toLowerCase(),
};

Registro aplicarIva(Registro r) => {
  ...r,
  'precio_final': ((r['precio'] as num) * 1.15).toStringAsFixed(2),
};

Registro agregarTimestamp(Registro r) => {
  ...r,
  'procesado_en': DateTime.now().toIso8601String().substring(0, 19),
};

// ─── Validadores ──────────────────────────────────────────────────────────────

bool tieneNombreValido(Registro r) =>
    r.containsKey('nombre') && (r['nombre'] as String).isNotEmpty;

bool tienePrecioPositivo(Registro r) =>
    r.containsKey('precio') && (r['precio'] as num) > 0;

// ─── Pipeline ─────────────────────────────────────────────────────────────────

// Aplica todos los transformadores en secuencia
Registro aplicarPipeline(Registro registro, List<Transformador> pasos) =>
    pasos.fold(registro, (actual, transformar) => transformar(actual));

// Ejecuta el procesamiento completo y devuelve un reporte
Reporte procesarLote({
  required List<Registro>     registros,
  required List<Validador>    validadores,
  required List<Transformador> transformadores,
  required void Function(Registro) onExito,
  required void Function(Registro, String) onError,
}) {
  int exitosos = 0;
  int fallidos  = 0;

  for (final registro in registros) {
    // Validar con todos los validadores
    final errorValidacion = _encontrarError(registro, validadores);

    if (errorValidacion != null) {
      onError(registro, errorValidacion);
      fallidos++;
      continue;
    }

    // Transformar y emitir
    final procesado = aplicarPipeline(registro, transformadores);
    onExito(procesado);
    exitosos++;
  }

  return {'exitosos': exitosos, 'fallidos': fallidos, 'total': registros.length};
}

// Aux — devuelve el primer error de validación o null si todo está bien
String? _encontrarError(Registro r, List<Validador> validadores) {
  if (!tieneNombreValido(r))    return 'Nombre inválido o ausente';
  if (!tienePrecioPositivo(r))  return 'Precio inválido o negativo';
  return null;
}

// ─── Programa principal ───────────────────────────────────────────────────────

void main() {
  final productos = [
    {'nombre': '  Teclado Mecánico  ', 'precio': 89.99,  'categoria': 'periféricos'},
    {'nombre': 'Monitor 4K',           'precio': 0,       'categoria': 'monitores'},   // inválido
    {'nombre': 'Mouse Inalámbrico',    'precio': 35.50,   'categoria': 'periféricos'},
    {'nombre': '',                     'precio': 120.00,  'categoria': 'laptops'},      // inválido
    {'nombre': 'Webcam HD',            'precio': 55.00,   'categoria': 'accesorios'},
  ];

  final reporte = procesarLote(
    registros:      productos,
    validadores:    [tieneNombreValido, tienePrecioPositivo],
    transformadores: [normalizarNombre, aplicarIva, agregarTimestamp],
    onExito: (r) => print('✅ ${r['nombre']} → \$${r['precio_final']}'),
    onError: (r, err) => print('❌ ${r['nombre'] ?? 'sin nombre'}: $err'),
  );

  print('\n─── Reporte de procesamiento ───');
  print('Total:     ${reporte['total']}');
  print('Exitosos:  ${reporte['exitosos']}');
  print('Fallidos:  ${reporte['fallidos']}');
}
```

Salida:
```
✅ teclado mecánico → $103.49
❌ Monitor 4K: Precio inválido o negativo
✅ mouse inalámbrico → $40.83
❌ sin nombre: Nombre inválido o ausente
✅ webcam hd → $63.25

─── Reporte de procesamiento ───
Total:     5
Exitosos:  3
Fallidos:  2
```

---

## Ejemplo de intensidad media — Enrutador funcional HTTP

Simula el mecanismo de enrutamiento de un framework web
usando funciones de orden superior y un mapa de handlers.

```dart
// ─── Tipos ───────────────────────────────────────────────────────────────────

typedef Solicitud  = ({String metodo, String ruta, Map<String, String> headers});
typedef Respuesta  = ({int codigo, String cuerpo});
typedef Handler    = Respuesta Function(Solicitud);
typedef Middleware = Handler Function(Handler);

// ─── Helpers para construir respuestas ────────────────────────────────────────

Respuesta ok(String cuerpo)             => (codigo: 200, cuerpo: cuerpo);
Respuesta creado(String cuerpo)         => (codigo: 201, cuerpo: cuerpo);
Respuesta noEncontrado(String recurso)  => (codigo: 404, cuerpo: 'No encontrado: $recurso');
Respuesta noAutorizado()                => (codigo: 401, cuerpo: 'Autenticación requerida');

// ─── Middlewares — envuelven al handler con lógica transversal ────────────────

// Middleware de logging
Middleware conLogging(String etiqueta) {
  return (Handler siguiente) {
    return (Solicitud req) {
      print('[LOG] ${req.metodo} ${req.ruta}');
      final res = siguiente(req);
      print('[LOG] → ${res.codigo}');
      return res;
    };
  };
}

// Middleware de autenticación
Middleware conAuth() {
  return (Handler siguiente) {
    return (Solicitud req) {
      final token = req.headers['Authorization'];
      if (token == null || !token.startsWith('Bearer ')) {
        return noAutorizado();
      }
      return siguiente(req);
    };
  };
}

// Aplica una lista de middlewares de derecha a izquierda
Handler aplicarMiddlewares(Handler handler, List<Middleware> middlewares) =>
    middlewares.reversed.fold(handler, (h, mw) => mw(h));

// ─── Handlers de ruta ─────────────────────────────────────────────────────────

Respuesta handlerUsuarios(Solicitud req) =>
    ok('[{"id":1,"nombre":"Ana"},{"id":2,"nombre":"Luis"}]');

Respuesta handlerProductos(Solicitud req) =>
    ok('[{"id":10,"nombre":"Laptop"},{"id":11,"nombre":"Monitor"}]]');

Respuesta handlerCrearPedido(Solicitud req) =>
    creado('{"pedido_id": 9901, "estado": "pendiente"}');

// ─── Registro de rutas ────────────────────────────────────────────────────────

final rutas = <String, Handler>{
  'GET /api/usuarios':  aplicarMiddlewares(handlerUsuarios,   [conLogging('usuarios'), conAuth()]),
  'GET /api/productos': aplicarMiddlewares(handlerProductos,  [conLogging('productos')]),
  'POST /api/pedidos':  aplicarMiddlewares(handlerCrearPedido,[conLogging('pedidos'), conAuth()]),
};

// ─── Despacho de solicitudes ──────────────────────────────────────────────────

Respuesta despachar(Solicitud solicitud) {
  final clave = '${solicitud.metodo} ${solicitud.ruta}';
  final handler = rutas[clave];
  return handler != null ? handler(solicitud) : noEncontrado(solicitud.ruta);
}

// ─── Main ─────────────────────────────────────────────────────────────────────

void main() {
  final solicitudes = <Solicitud>[
    (metodo: 'GET',  ruta: '/api/productos', headers: {}),
    (metodo: 'GET',  ruta: '/api/usuarios',  headers: {}),  // sin token
    (metodo: 'GET',  ruta: '/api/usuarios',  headers: {'Authorization': 'Bearer tok_abc123'}),
    (metodo: 'POST', ruta: '/api/pedidos',   headers: {'Authorization': 'Bearer tok_xyz'}),
    (metodo: 'GET',  ruta: '/api/facturas',  headers: {}),  // ruta inexistente
  ];

  print('═══ Servidor simulado ═══\n');
  for (final req in solicitudes) {
    final res = despachar(req);
    print('  ${res.codigo} ${res.cuerpo.substring(0, res.cuerpo.length.clamp(0, 50))}');
    print('');
  }
}
```

---

## Ejercicios propuestos

1. **Conversor de unidades configurable** — Escribe `double Function(double)`
   `crearConversor(String de, String a)` que devuelva una closure capaz de convertir
   entre: `km↔millas`, `celsius↔fahrenheit`, `kg↔libras`.
   Úsala para crear `kmAMillas`, `celsiusAFahrenheit` y `kgALibras`
   como variables independientes.

2. **Pipeline de validación de contraseñas** — Define el tipo
   `typedef ReglaPassword = ({bool valido, String mensaje}) Function(String)`.
   Crea al menos 4 reglas: longitud mínima, mayúscula, número, carácter especial.
   Escribe `List<String> validarPassword(String password, List<ReglaPassword> reglas)`
   que devuelva la lista de mensajes de error.
   Pruébala con contraseñas válidas e inválidas.

3. **Memoización genérica** — Implementa `T Function(K) memoizar<K, T>(T Function(K) fn)`
   usando un `Map<K, T>` como caché. Aplícala a `fibonacci` y compara cuántas
   llamadas se hacen con y sin memoización para `fibonacci(35)`.

4. **Compositor de transformaciones de texto** — Define `typedef TransformTexto = String Function(String)`.
   Crea: `aMayusculas`, `quitarEspacios`, `invertir`, `truncarEn(int n)`.
   Escribe `TransformTexto componer(List<TransformTexto> fns)` que aplique
   todas las transformaciones en secuencia y devuelva la función compuesta.
   Combínalas para crear un formateador de slugs de URL.

---

## Resumen de la página 3

- Una función en Dart tiene tipo de retorno, nombre, parámetros y cuerpo. La sintaxis de flecha `=>` simplifica funciones de una sola expresión.
- Los parámetros posicionales opcionales `[param]` permiten omitirlos. Los nombrados `{param}` se pasan por nombre en cualquier orden — son el estándar de Flutter. `required` hace que un parámetro nombrado sea obligatorio.
- Las funciones son objetos de primera clase: se almacenan en variables con tipo `TipoRetorno Function(Params...)`, se pasan como argumentos y se devuelven como resultado.
- Las funciones anónimas (lambdas) se definen sin nombre, directamente donde se usan, con `(params) => expresion` o `(params) { cuerpo; }`.
- Las funciones de orden superior reciben o devuelven funciones. `map`, `where`, `reduce` y `fold` son ejemplos integrados en las colecciones de Dart.
- Una closure captura y mantiene el estado del ámbito donde fue creada, incluso cuando ese ámbito ya no existe. Es la base de contadores, loggers y fábricas de funciones.
- `typedef` asigna un nombre a un tipo de función complejo, mejorando la legibilidad y permitiendo reutilizarlo en múltiples firmas.
- Las funciones recursivas se resuelven dividiéndose en casos base y casos recursivos. Úsalas con cautela en listas grandes — pueden ser costosas sin memoización.
- Las funciones `async/await` retornan `Future<T>` y se estudian en profundidad en el módulo de asincronía.

---

> **Siguiente página →** Página 4: Clases y POO en Dart — constructores,
> getters/setters, herencia, mixins e interfaces.