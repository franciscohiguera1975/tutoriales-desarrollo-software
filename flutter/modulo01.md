# Tutorial Flutter — Página 1
## Módulo 1 · Entorno y Dart
### Instalación, estructura del proyecto y fundamentos de Dart

---

## ¿Qué es Flutter?

Flutter es el framework de Google para construir aplicaciones nativas
desde un único código base en **Dart**. A diferencia de React Native
que usa componentes nativos del sistema, Flutter tiene su propio motor
de renderizado (**Impeller**) — dibuja cada pixel directamente en el
canvas, lo que garantiza UI idéntica en Android, iOS, web y desktop.

```
React Native          →  puente JavaScript → componentes nativos del SO
Kotlin/Compose        →  compila a bytecode JVM → componentes Android
Flutter               →  compila a código nativo → motor Impeller dibuja todo
```

### Comparativa con Kotlin (para quienes vienen del curso anterior)

| Kotlin + Compose | Flutter + Dart |
|---|---|
| `@Composable fun` | `Widget build(BuildContext)` |
| `remember { mutableStateOf() }` | `setState(() {})` / Riverpod |
| `ViewModel` + `StateFlow` | `AsyncNotifier` de Riverpod |
| `Navigation Compose` | `GoRouter` |
| `LazyColumn` | `ListView.builder` |
| `data class` | clase con `factory fromJson` |
| `suspend fun` | `async`/`await` con `Future` |

---

## Versiones del curso

| Herramienta | Versión |
|---|---|
| Flutter SDK | **3.41.2** (estable, febrero 2026) |
| Dart SDK | **3.7.0** (incluido con Flutter) |
| Android Studio | Meerkat 2024.3.2+ o VS Code |
| Riverpod | 3.3.0 (páginas 9–21) |
| GoRouter | última estable (páginas 10–21) |

---

## Parte 1 — Instalación del entorno

### Instalar Flutter SDK

```bash
# macOS con Homebrew
brew install flutter

# Windows — descargar el SDK desde flutter.dev/install
# Luego añadir al PATH:
# C:\flutter\bin

# Linux
sudo snap install flutter --classic

# Verificar instalación
flutter doctor
```

La salida de `flutter doctor` muestra el estado de cada herramienta:

```
Doctor summary (to see all details, run flutter doctor -v):
[✓] Flutter (Channel stable, 3.41.2)
[✓] Android toolchain - develop for Android devices
[✓] Chrome - develop for the web
[✓] Android Studio (version 2024.3)
[✓] VS Code (version 1.95.0)
[✓] Connected device (2 available)
[✓] Network resources
```

Resuelve cada `[✗]` antes de continuar. El error más común es
`Android licenses not accepted` — se resuelve con:

```bash
flutter doctor --android-licenses
```

### Extensiones recomendadas en VS Code

```
Flutter          (id: Dart-Code.flutter)     ← obligatoria
Dart             (id: Dart-Code.dart-code)   ← obligatoria
Error Lens       (id: usernamehw.errorlens)  ← recomendada
Pubspec Assist   (id: jeroen-meijer.pubspec-assist) ← útil
```

---

## Parte 2 — Primer proyecto

```bash
# Crear proyecto nuevo
flutter create mi_primera_app

# Entrar al directorio
cd mi_primera_app

# Ejecutar en el emulador / dispositivo conectado
flutter run

# Ejecutar en Chrome (web)
flutter run -d chrome

# Ver dispositivos disponibles
flutter devices
```

### Estructura del proyecto

```
mi_primera_app/
├── android/              ← config nativa Android
├── ios/                  ← config nativa iOS
├── lib/
│   └── main.dart         ← punto de entrada — AQUÍ trabajamos
├── test/
│   └── widget_test.dart  ← tests automáticos
├── pubspec.yaml          ← dependencias (equivale a build.gradle.kts)
├── pubspec.lock          ← versiones exactas resueltas
└── analysis_options.yaml ← reglas de linting
```

### `pubspec.yaml` — gestión de dependencias

```yaml
# pubspec.yaml — equivalente a build.gradle.kts en Android

name: mi_primera_app
description: "Mi primera app Flutter"

# Versión de Dart requerida
environment:
  sdk: ^3.7.0

dependencies:
  flutter:
    sdk: flutter

  # Añadir dependencias aquí — equivale a implementation() en Gradle
  http: ^1.2.2

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^5.0.0

flutter:
  uses-material-design: true

  # Assets (imágenes, fuentes, etc.)
  assets:
    - assets/images/

  fonts:
    - family: Inter
      fonts:
        - asset: assets/fonts/Inter-Regular.ttf
        - asset: assets/fonts/Inter-Bold.ttf
          weight: 700
```

```bash
# Instalar dependencias (equivale a Gradle sync)
flutter pub get

# Añadir una dependencia desde la terminal
flutter pub add http

# Actualizar dependencias
flutter pub upgrade
```

---

## Parte 3 — Fundamentos de Dart

Dart es el lenguaje de Flutter. Es fuertemente tipado, con null safety
y sintaxis similar a Kotlin/Java pero con sus propias particularidades.

### Variables y tipos

```dart
void main() {
  // var — tipo inferido (como val en Kotlin)
  var nombre = 'Ana';           // String
  var edad   = 28;              // int
  var precio = 89.99;           // double
  var activo = true;            // bool

  // Tipo explícito
  String apellido = 'García';
  int    stock    = 100;
  double pi       = 3.14159;
  bool   visible  = false;

  // final — no se puede reasignar (como val en Kotlin)
  final ciudad = 'Madrid';
  // ciudad = 'Barcelona';  // ERROR — final no se puede reasignar

  // const — constante en tiempo de compilación (como const en Kotlin)
  const gravedad = 9.8;
  const pi2      = 3.14159;

  // Diferencia clave: final vs const
  final ahora  = DateTime.now();   // OK — se evalúa en runtime
  // const ahora = DateTime.now(); // ERROR — DateTime.now() no es constante de compilación

  print('$nombre $apellido tiene $edad años en $ciudad');
}
```

### Diferencia `var`, `final` y `const`

```dart
// var — mutable, tipo inferido
var contador = 0;
contador = 1;          // OK

// final — inmutable referencia, evaluado en runtime
final lista = [1, 2, 3];
lista.add(4);          // OK — la referencia es final, no el contenido
// lista = [5, 6];     // ERROR — no se puede reasignar la referencia

// const — inmutable profundo, evaluado en compilación
const colores = ['rojo', 'azul'];
// colores.add('verde'); // ERROR — lista const es completamente inmutable
```

### Null safety — el sistema de tipos

```dart
void main() {
  // Tipo no-nullable — NUNCA puede ser null
  String nombre = 'Ana';
  // nombre = null;       // ERROR de compilación

  // Tipo nullable — puede ser null (añadir ?)
  String? apellido = null;   // OK
  apellido = 'García';       // OK

  // Operadores de null safety
  String? ciudad;

  // ?. — safe call (igual que en Kotlin)
  print(ciudad?.length);      // null — no lanza excepción

  // ?? — operador Elvis (igual que ?: en Kotlin)
  String resultado = ciudad ?? 'Sin ciudad';
  print(resultado);           // Sin ciudad

  // ! — non-null assertion (igual que !! en Kotlin) — úsalo con precaución
  String ciudadSegura = ciudad!;  // lanza si ciudad es null

  // Null check con if
  if (apellido != null) {
    print(apellido.length);   // smart cast — ya es String aquí
  }

  // late — inicialización diferida (como lateinit en Kotlin)
  late String token;
  token = 'abc123';           // debe asignarse antes de usar
  print(token);
}
```

### Tipos de colecciones

```dart
void main() {
  // List — lista ordenada (como List en Kotlin)
  List<String> frutas   = ['manzana', 'banana', 'cereza'];
  var          numeros  = [1, 2, 3, 4, 5];       // tipo inferido: List<int>

  print(frutas[0]);         // manzana
  print(frutas.length);     // 3
  frutas.add('dátil');
  frutas.remove('banana');

  // Map — clave → valor (como Map en Kotlin)
  Map<String, int> edades = {
    'Ana':   28,
    'Luis':  34,
    'María': 25,
  };

  print(edades['Ana']);     // 28
  print(edades['Pedro']);   // null — clave no existe
  edades['Carlos'] = 40;    // añadir

  // Set — sin duplicados (como Set en Kotlin)
  Set<String> tags = {'flutter', 'dart', 'mobile'};
  tags.add('flutter');      // ignorado — ya existe
  print(tags.length);       // 3

  // Spread operator — para combinar colecciones
  var lista1 = [1, 2, 3];
  var lista2 = [4, 5, 6];
  var combinada = [...lista1, ...lista2];  // [1, 2, 3, 4, 5, 6]
  print(combinada);

  // Collection if — elementos condicionales
  bool mostrarExtra = true;
  var items = [
    'elemento1',
    'elemento2',
    if (mostrarExtra) 'elemento3',  // solo si la condición es true
  ];

  // Collection for — generar elementos
  var cuadrados = [for (var i = 1; i <= 5; i++) i * i];
  print(cuadrados);  // [1, 4, 9, 16, 25]
}
```

---

## Parte 4 — String templates e interpolación

```dart
void main() {
  final nombre = 'Ana';
  final edad   = 28;

  // Interpolación con $ (igual que en Kotlin)
  print('Hola, $nombre');                    // Hola, Ana

  // Expresión con ${ }
  print('${nombre.toUpperCase()} tiene ${edad + 1} años el próximo año');

  // String multilinea con triple comillas
  final tarjeta = '''
Nombre: $nombre
Edad:   $edad
Mayor:  ${edad >= 18 ? 'Sí' : 'No'}
  ''';
  print(tarjeta);

  // Raw string — ignora el escape y la interpolación
  final ruta = r'C:\Users\Ana\Documents';  // el \ no se interpreta
  print(ruta);

  // Concatenación (menos idiomático — preferir interpolación)
  final saludo = 'Hola, ' + nombre + '!';

  // Métodos útiles de String
  print('flutter'.toUpperCase());           // FLUTTER
  print('  Flutter  '.trim());              // Flutter
  print('Flutter'.contains('lut'));         // true
  print('Flutter'.replaceAll('t', 'T'));    // FluTTer
  print('a,b,c'.split(','));                // [a, b, c]
  print('Flutter'.substring(0, 4));         // Flut
  print('Flutter'.startsWith('Flu'));       // true
  print('abc'.padLeft(5, '0'));             // 00abc
}
```

---

## Parte 5 — Tipos y conversiones

```dart
void main() {
  // Conversiones numéricas
  int    entero  = 42;
  double decimal = entero.toDouble();   // 42.0
  String texto   = entero.toString();   // "42"

  // String → número
  int    num1 = int.parse('123');       // 123
  double num2 = double.parse('3.14');   // 3.14

  // Conversión segura (no lanza excepción)
  int?    num3 = int.tryParse('abc');   // null
  double? num4 = double.tryParse('99'); // 99.0

  // Verificar tipo con is (como en Kotlin)
  Object valor = 'texto';
  if (valor is String) {
    print(valor.length);  // smart cast — ya es String
  }

  // Cast explícito con as
  Object obj = 'Hola';
  String str = obj as String;

  // Comprobar nulabilidad
  String? nullable = null;
  int longitud = nullable?.length ?? 0;
  print(longitud);  // 0

  // Números especiales
  print(double.infinity);     // Infinity
  print(double.nan);          // NaN
  print(double.maxFinite);    // 1.7976931348623157e+308
}
```

---

## El archivo `main.dart` explicado

```dart
// lib/main.dart — punto de entrada de la app

import 'package:flutter/material.dart';  // importa el framework Material

// Punto de entrada — equivale a fun main() en Kotlin
void main() {
  runApp(const MiApp());  // arranca Flutter con el widget raíz
}

// Widget sin estado — equivale a @Composable fun sin remember
class MiApp extends StatelessWidget {
  const MiApp({super.key});  // constructor con key para optimización

  @override
  Widget build(BuildContext context) {
    // build() devuelve el árbol de widgets
    // se llama cuando Flutter necesita (re)construir la UI
    return MaterialApp(
      title:   'Mi App',
      theme:   ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.blue),
        useMaterial3: true,
      ),
      home:    const PantallaInicio(),
    );
  }
}

class PantallaInicio extends StatelessWidget {
  const PantallaInicio({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Mi primera app Flutter'),
      ),
      body: const Center(
        child: Text(
          '¡Hola, Flutter 3.41!',
          style: TextStyle(fontSize: 24),
        ),
      ),
    );
  }
}
```

---

## Hot Reload vs Hot Restart

```
Hot Reload  (⌘R / Ctrl+F5)    →  recarga solo los cambios de widget
                                   El estado se preserva
                                   Instantáneo — ideal para UI

Hot Restart (⌘⇧R / Ctrl+⇧F5) →  reinicia la app desde cero
                                   El estado se pierde
                                   Necesario para cambios en main()
                                   o en initState()
```

---

## Ejercicios propuestos

1. **Explorar `flutter doctor`** — Ejecuta `flutter doctor -v` y resuelve
   todos los warnings hasta que todos los items estén con `[✓]`.
   Documenta qué pasos seguiste para resolver cada uno.

2. **Tipos Dart** — En `main.dart` crea variables de cada tipo (`int`,
   `double`, `String`, `bool`, `List`, `Map`, `Set`) con valor inicial,
   una versión `final` y una versión nullable. Imprímelas todas con
   interpolación de strings.

3. **Null safety** — Escribe una función `String describir(String? valor)`
   que devuelva `"Vacío"` si el valor es null o está en blanco, el valor
   en mayúsculas si tiene más de 5 caracteres, o el valor tal cual
   en los demás casos. Usa `?.`, `??` y el operador ternario.

4. **Collection operations** — Crea una `List<Map<String, dynamic>>` con
   5 productos (nombre, precio, activo). Usa `collection if`, `collection for`
   y el spread operator para construir listas filtradas. Imprime el total
   de los productos activos con precio mayor a 50.

---

## Resumen de la página 1

- Flutter 3.41.2 usa Dart 3.7.0 con null safety obligatorio — todas las variables no-nullable están garantizadas por el compilador.
- `var` infiere el tipo; `final` es inmutable en referencia; `const` es inmutable profundo en tiempo de compilación.
- El operador `?` declara tipos nullable (`String?`). Los operadores `?.`, `??` y `!` manejan valores potencialmente nulos — idénticos a Kotlin.
- `pubspec.yaml` gestiona las dependencias como `build.gradle.kts` en Android. `flutter pub get` las instala.
- `main()` llama a `runApp()` con el widget raíz. Todo en Flutter es un widget — la UI es un árbol de widgets.
- `StatelessWidget` describe UI estática. `build(BuildContext)` devuelve el árbol de widgets y se llama cuando Flutter necesita redibujar.
- Hot Reload recarga sin perder estado — ideal para iterar en la UI. Hot Restart reinicia completamente.

---

> **Siguiente página →** Página 2: Control de flujo, funciones,
> parámetros nombrados y programación asíncrona con `Future`/`async`/`await`.