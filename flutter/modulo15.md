# Tutorial Flutter — Página 15
## Módulo 5 · Hardware
### Permisos, cámara, galería y GPS con Riverpod

---

En los módulos anteriores construiste UIs reactivas con Riverpod y consumiste APIs REST. Ahora das el siguiente paso: interactuar con el hardware real del dispositivo. Este módulo cubre la gestión de permisos del sistema, la captura y selección de imágenes, y la obtención de posición GPS, todo integrado con Riverpod y organizado en una arquitectura limpia de servicios y notifiers.

```
Sin gestión de hardware                  Con gestión de hardware
──────────────────────────────────────   ───────────────────────────────────────────
La app solo muestra datos estáticos      Captura fotos, audio, video en tiempo real
Ubicación codificada en el código        GPS real con streaming de posición
Todos los permisos denegados             Permisos pedidos en contexto, con UX clara
Estado disperso en múltiples setState()  Estado centralizado en Notifier + Riverpod
```

Paquetes que usamos en este módulo:

```
permission_handler   → pedir y verificar permisos del sistema (cámara, ubicación, galería)
image_picker         → tomar foto con cámara o seleccionar de galería
geolocator           → obtener posición GPS una vez o como stream continuo
```

El proyecto: **HardwareApp** con dos tabs — pestaña Cámara y pestaña GPS.

---

## 6 Patrones clave

### Patrón 1 — `permission_handler`: verificar y pedir permisos

```dart
// Patrón: verificar estado de un permiso y solicitarlo si está denegado
import 'package:permission_handler/permission_handler.dart';

Future<bool> gestionarPermisoCamara() async {
  // status es síncrono si ya se conoce
  final status = await Permission.camera.status;

  if (status.isGranted)           return true;    // ya concedido
  if (status.isPermanentlyDenied) {
    // El usuario dijo "No volver a preguntar" — abrir ajustes del sistema
    await openAppSettings();
    return false;
  }

  // Solicitar al sistema — muestra el diálogo nativo
  final resultado = await Permission.camera.request();
  return resultado.isGranted;
}

// Pedir múltiples permisos a la vez
Future<void> pedirTodos() async {
  final statuses = await [
    Permission.camera,
    Permission.photos,
    Permission.locationWhenInUse,
  ].request();   // devuelve Map<Permission, PermissionStatus>

  statuses.forEach((permiso, estado) {
    print('$permiso → ${estado.name}');
  });
}
```

**Mini-ejercicio ⏱ 5 min:**
Añade una función `Future<String> estadoPermisoCamara()` que retorne:
- `'concedido'` si `isGranted`
- `'permanentemente denegado'` si `isPermanentlyDenied`
- `'pendiente'` en cualquier otro caso

---

### Patrón 2 — `ImagePicker`: tomar foto con la cámara

```dart
// Patrón: abrir la cámara y obtener la ruta de la foto tomada
import 'package:image_picker/image_picker.dart';

Future<void> tomarFoto() async {
  final picker = ImagePicker();

  // pickImage retorna XFile? (null si el usuario cancela)
  final foto = await picker.pickImage(
    source:       ImageSource.camera,    // o ImageSource.gallery
    imageQuality: 85,                    // 0-100, reduce el tamaño del archivo
    maxWidth:     1920,                  // redimensiona si es mayor
    maxHeight:    1080,
  );

  if (foto == null) return;   // usuario canceló

  // XFile tiene: path (String), name (String), length() Future<int>
  print('Ruta: ${foto.path}');
  print('Nombre: ${foto.name}');
  print('Tamaño: ${await foto.length()} bytes');

  // Mostrar con Image.file
  // Image.file(File(foto.path), fit: BoxFit.cover)
}
```

**Mini-ejercicio ⏱ 5 min:**
Modifica la función para que si `imageQuality < 50` imprima `'baja calidad'`, si está entre 50 y 80 imprima `'calidad media'`, y si es mayor imprima `'alta calidad'`. Usa un switch en Dart 3.

---

### Patrón 3 — `ImagePicker`: seleccionar múltiples imágenes de galería

```dart
// Patrón: seleccionar varias imágenes de la galería
import 'dart:io';
import 'package:image_picker/image_picker.dart';

class ServicioImagenes {
  final _picker = ImagePicker();

  // Una imagen de cámara
  Future<XFile?> tomarFoto({int calidad = 85}) =>
      _picker.pickImage(source: ImageSource.camera, imageQuality: calidad);

  // Una imagen de galería
  Future<XFile?> seleccionarDeGaleria({int calidad = 85}) =>
      _picker.pickImage(source: ImageSource.gallery, imageQuality: calidad);

  // Múltiples imágenes de galería (límite configurable)
  Future<List<XFile>> seleccionarVarias({int limite = 5}) =>
      _picker.pickMultiImage(limit: limite, imageQuality: 85);

  // Calcular tamaño total de una lista de archivos
  Future<int> tamanoTotalBytes(List<XFile> archivos) async {
    var total = 0;
    for (final f in archivos) total += await f.length();
    return total;
  }
}
```

**Mini-ejercicio ⏱ 5 min:**
Añade un método `Future<XFile?> tomarVideo({int duracionSegMax = 30})` que use `_picker.pickVideo(source: ImageSource.camera, maxDuration: Duration(seconds: duracionSegMax))`.

---

### Patrón 4 — `Geolocator`: posición única y campos de `Position`

```dart
// Patrón: obtener la posición GPS actual una sola vez
import 'package:geolocator/geolocator.dart';

Future<void> obtenerPosicion() async {
  // Verificar que el servicio de ubicación está activo
  final servicioActivo = await Geolocator.isLocationServiceEnabled();
  if (!servicioActivo) {
    print('GPS desactivado en el dispositivo');
    return;
  }

  // Obtener posición con alta precisión
  final pos = await Geolocator.getCurrentPosition(
    locationSettings: const LocationSettings(
      accuracy: LocationAccuracy.high,    // best, high, medium, low, lowest
    ),
  );

  // Campos de Position más útiles
  print('Latitud:   ${pos.latitude.toStringAsFixed(6)}');  // grados decimales
  print('Longitud:  ${pos.longitude.toStringAsFixed(6)}');
  print('Altitud:   ${pos.altitude.toStringAsFixed(1)} m');
  print('Precisión: ±${pos.accuracy.toStringAsFixed(0)} m');
  print('Velocidad: ${(pos.speed * 3.6).toStringAsFixed(1)} km/h'); // m/s → km/h
  print('Rumbo:     ${pos.heading.toStringAsFixed(1)}°');
  print('Timestamp: ${pos.timestamp}');

  // Distancia entre dos puntos (en metros)
  final distancia = Geolocator.distanceBetween(
    pos.latitude, pos.longitude,
    40.4168,      -3.7038,         // Madrid
  );
  print('Distancia a Madrid: ${(distancia / 1000).toStringAsFixed(1)} km');
}
```

**Mini-ejercicio ⏱ 5 min:**
Escribe una función `String clasificarVelocidad(double speedMs)` que retorne:
- `'estático'` si speed < 0.5 m/s
- `'caminando'` si < 3 m/s
- `'corriendo'` si < 10 m/s
- `'en vehículo'` en cualquier otro caso

---

### Patrón 5 — `Geolocator.getPositionStream()` + `StreamSubscription`

```dart
// Patrón: rastrear posición continua con stream + cancelación limpia
import 'dart:async';
import 'package:geolocator/geolocator.dart';

class RastreadorGps {
  StreamSubscription<Position>? _suscripcion;
  bool get rastreando => _suscripcion != null;

  void iniciar({int metrosMinimos = 5}) {
    if (rastreando) return;    // ya está activo

    final stream = Geolocator.getPositionStream(
      locationSettings: LocationSettings(
        accuracy:       LocationAccuracy.high,
        distanceFilter: metrosMinimos,   // solo emite si se movió N metros
      ),
    );

    _suscripcion = stream.listen(
      (pos) {
        print('Nueva posición: ${pos.latitude}, ${pos.longitude}');
      },
      onError:       (e) => print('Error GPS: $e'),
      cancelOnError: false,    // continuar aunque haya un error
    );
  }

  void detener() {
    _suscripcion?.cancel();
    _suscripcion = null;
  }
}

// ── En un Riverpod Notifier — SIEMPRE cancelar en ref.onDispose() ────
// @override
// UbicacionState build() {
//   ref.onDispose(() => _suscripcion?.cancel());  // limpieza automática
//   return const UbicacionState();
// }
```

**Mini-ejercicio ⏱ 5 min:**
Añade un campo `List<Position> historial = []` a `RastreadorGps` que acumule las últimas 50 posiciones en el `stream.listen`. Añade también un método `limpiarHistorial()` que vacíe la lista.

---

### Patrón 6 — `ref.onDispose()` en Notifier para limpiar streams

```dart
// Patrón: gestionar el ciclo de vida de StreamSubscription dentro de Riverpod
import 'dart:async';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:geolocator/geolocator.dart';

class UbicacionState {
  final Position?      posicion;
  final bool           rastreando;
  final bool           tienePermiso;
  final List<Position> historial;

  const UbicacionState({
    this.posicion     = null,
    this.rastreando   = false,
    this.tienePermiso = false,
    this.historial    = const [],
  });

  UbicacionState copyWith({
    Position?       posicion,
    bool?           rastreando,
    bool?           tienePermiso,
    List<Position>? historial,
  }) => UbicacionState(
    posicion:     posicion     ?? this.posicion,
    rastreando:   rastreando   ?? this.rastreando,
    tienePermiso: tienePermiso ?? this.tienePermiso,
    historial:    historial    ?? this.historial,
  );
}

class UbicacionNotifier extends Notifier<UbicacionState> {
  StreamSubscription<Position>? _sub;

  @override
  UbicacionState build() {
    // ref.onDispose() se llama cuando el provider se destruye
    // Garantiza que el stream se cancela aunque el usuario cambie de pantalla
    ref.onDispose(() => _sub?.cancel());
    return const UbicacionState();
  }

  void iniciarRastreo() {
    if (state.rastreando) return;
    state = state.copyWith(rastreando: true);

    _sub = Geolocator.getPositionStream(
      locationSettings: const LocationSettings(
        accuracy:       LocationAccuracy.high,
        distanceFilter: 3,
      ),
    ).listen((pos) {
      state = state.copyWith(
        posicion:  pos,
        historial: [...state.historial, pos].take(50).toList(),
      );
    });
  }

  void detenerRastreo() {
    _sub?.cancel();
    _sub = null;
    state = state.copyWith(rastreando: false);
  }
}

final ubicacionProvider =
    NotifierProvider<UbicacionNotifier, UbicacionState>(UbicacionNotifier.new);
```

**Mini-ejercicio ⏱ 5 min:**
Añade un método `void limpiarHistorial()` al notifier que setee `state = state.copyWith(historial: [])`. Llámalo desde la UI con un botón "Limpiar recorrido".

---

## Crea el proyecto

```bash
flutter create modulo15_hardware --empty
cd modulo15_hardware
```

**`pubspec.yaml`:**

```yaml
name: modulo15_hardware
description: Cámara, galería y GPS — módulo 15

environment:
  sdk: '>=3.4.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod:   ^2.6.1
  permission_handler: ^11.3.1
  image_picker:       ^1.1.2
  geolocator:         ^13.0.1
```

```bash
flutter pub get
```

**Permisos nativos — configurar ANTES de correr la app:**

`android/app/src/main/AndroidManifest.xml` — dentro de `<manifest>`:

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

`ios/Runner/Info.plist` — dentro de `<dict>`:

```xml
<key>NSCameraUsageDescription</key>
<string>Para tomar fotos con la app</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>Para seleccionar imágenes de la galería</string>
<key>NSLocationWhenInUseUsageDescription</key>
<string>Para mostrar tu ubicación actual en el mapa</string>
```

**Estructura del proyecto:**

```
modulo15_hardware/
├── lib/
│   ├── main.dart
│   ├── core/
│   │   └── permisos_helper.dart           ← helper centralizado
│   └── features/
│       ├── camara/
│       │   ├── servicio_imagenes.dart      ← abstracción image_picker
│       │   ├── media_notifier.dart         ← Riverpod Notifier
│       │   └── pantalla_media.dart         ← UI cámara/galería
│       └── gps/
│           ├── servicio_ubicacion.dart     ← abstracción geolocator
│           ├── ubicacion_notifier.dart     ← Riverpod Notifier + stream
│           └── pantalla_gps.dart           ← UI GPS
```

---

> **Selector de pasos** — cambia `const int paso = N` en `main.dart`:
>
> ```
> Paso 1 → App mínima compila y corre
> Paso 2 → Permisos: diálogo del sistema y resultado
> Paso 3 → Cámara: tomar foto y mostrarla
> Paso 4 → Galería: múltiples fotos en GridView con Riverpod
> Paso 5 → GPS: posición única con coordenadas reales
> Paso 6 → GPS: stream continuo + historial con Riverpod
> Paso 7 → App final: NavigationBar con tab Cámara + tab GPS
> ```

---

## Paso 1 — App mínima funcionando

Antes de cualquier hardware, arranca la app con solo un `Scaffold`. Verifica que compila sin errores.

**`lib/main.dart` (Paso 1):**

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

const int paso = 1;

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const ProviderScope(child: HardwareApp()));
}

class HardwareApp extends StatelessWidget {
  const HardwareApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Hardware App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const _Paso1(),
    );
  }
}

class _Paso1 extends StatelessWidget {
  const _Paso1();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Hardware App — Paso 1')),
      body: const Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            Icon(Icons.phone_android, size: 80, color: Colors.indigo),
            SizedBox(height: 16),
            Text('App funcionando', style: TextStyle(fontSize: 20)),
            SizedBox(height: 8),
            Text('Paso 1 completado',
                style: TextStyle(color: Colors.grey)),
          ],
        ),
      ),
    );
  }
}
```

**Prueba esto:**

```bash
flutter run
```

- La app muestra el ícono y el texto "App funcionando".
- No hay errores de compilación.

---

## Paso 2 — Permisos con `permission_handler`

**`lib/core/permisos_helper.dart`:**

```dart
// lib/core/permisos_helper.dart
import 'package:geolocator/geolocator.dart';
import 'package:permission_handler/permission_handler.dart';

class PermisosHelper {
  // Cámara
  static Future<bool> solicitarCamara() async {
    final estado = await Permission.camera.request();
    return estado.isGranted;
  }

  // Galería / fotos
  static Future<bool> solicitarGaleria() async {
    final estado = await Permission.photos.request();
    return estado.isGranted;
  }

  // Ubicación (verifica servicio GPS antes de pedir permiso)
  static Future<bool> solicitarUbicacion() async {
    final servicioActivo = await Geolocator.isLocationServiceEnabled();
    if (!servicioActivo) return false;

    var estado = await Permission.locationWhenInUse.status;
    if (estado.isDenied) {
      estado = await Permission.locationWhenInUse.request();
    }
    return estado.isGranted;
  }
}
```

Pantalla del Paso 2 — tres botones que piden permisos:

```dart
// Cambia const int paso = 2 y reemplaza home: por _Paso2()

class _Paso2 extends StatefulWidget {
  const _Paso2();

  @override
  State<_Paso2> createState() => _Paso2State();
}

class _Paso2State extends State<_Paso2> {
  String _resultado = 'Sin verificar';

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Hardware App — Paso 2 (Permisos)')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment:  MainAxisAlignment.center,
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: Text(
                  _resultado,
                  textAlign: TextAlign.center,
                  style: const TextStyle(fontSize: 18),
                ),
              ),
            ),
            const SizedBox(height: 32),
            FilledButton.icon(
              icon:      const Icon(Icons.camera_alt),
              label:     const Text('Pedir permiso de cámara'),
              onPressed: () async {
                final ok = await PermisosHelper.solicitarCamara();
                setState(() => _resultado =
                    ok ? 'Cámara: CONCEDIDO' : 'Cámara: DENEGADO');
              },
            ),
            const SizedBox(height: 12),
            FilledButton.icon(
              icon:      const Icon(Icons.photo_library),
              label:     const Text('Pedir permiso de galería'),
              onPressed: () async {
                final ok = await PermisosHelper.solicitarGaleria();
                setState(() => _resultado =
                    ok ? 'Galería: CONCEDIDO' : 'Galería: DENEGADO');
              },
            ),
            const SizedBox(height: 12),
            FilledButton.icon(
              icon:      const Icon(Icons.location_on),
              label:     const Text('Pedir permiso de ubicación'),
              onPressed: () async {
                final ok = await PermisosHelper.solicitarUbicacion();
                setState(() => _resultado =
                    ok ? 'Ubicación: CONCEDIDO' : 'Ubicación: DENEGADO');
              },
            ),
          ],
        ),
      ),
    );
  }
}
```

**Prueba esto:**

```bash
flutter run
```

- Pulsa cada botón — aparece el diálogo nativo del sistema.
- El texto en la Card cambia según lo que respondas.
- Pulsa "Pedir permiso de cámara" → acepta → aparece `Cámara: CONCEDIDO`.

---

## Paso 3 — Cámara: tomar foto y mostrarla

**`lib/features/camara/servicio_imagenes.dart`:**

```dart
// lib/features/camara/servicio_imagenes.dart
import 'package:image_picker/image_picker.dart';

class ServicioImagenes {
  final _picker = ImagePicker();

  Future<XFile?> tomarFoto({int calidad = 85}) =>
      _picker.pickImage(source: ImageSource.camera, imageQuality: calidad);

  Future<XFile?> seleccionarDeGaleria({int calidad = 85}) =>
      _picker.pickImage(source: ImageSource.gallery, imageQuality: calidad);

  Future<List<XFile>> seleccionarVarias({int limite = 5}) =>
      _picker.pickMultiImage(limit: limite, imageQuality: 85);
}
```

Pantalla del Paso 3 — foto con vista previa:

```dart
// Cambia const int paso = 3 y usa _Paso3 como home

import 'dart:io';
import 'package:flutter/material.dart';

class _Paso3 extends StatefulWidget {
  const _Paso3();

  @override
  State<_Paso3> createState() => _Paso3State();
}

class _Paso3State extends State<_Paso3> {
  final _servicio = ServicioImagenes();
  XFile?  _foto;
  String  _mensaje = 'Toca un botón para comenzar';

  Future<void> _tomarFoto() async {
    final ok = await PermisosHelper.solicitarCamara();
    if (!ok) {
      setState(() => _mensaje = 'Sin permiso de cámara');
      return;
    }
    final foto = await _servicio.tomarFoto();
    setState(() {
      _foto    = foto;
      _mensaje = foto != null ? 'Foto tomada' : 'Cancelado';
    });
  }

  Future<void> _abrirGaleria() async {
    final ok = await PermisosHelper.solicitarGaleria();
    if (!ok) {
      setState(() => _mensaje = 'Sin permiso de galería');
      return;
    }
    final foto = await _servicio.seleccionarDeGaleria();
    setState(() {
      _foto    = foto;
      _mensaje = foto != null ? 'Imagen seleccionada' : 'Cancelado';
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Hardware App — Paso 3 (Cámara)')),
      body: Column(
        children: [
          Expanded(
            child: _foto != null
                ? Image.file(File(_foto!.path), fit: BoxFit.contain)
                : Center(
                    child: Column(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        const Icon(Icons.camera_alt_outlined,
                            size: 64, color: Colors.grey),
                        const SizedBox(height: 12),
                        Text(_mensaje,
                            style: const TextStyle(color: Colors.grey)),
                      ],
                    ),
                  ),
          ),
          Padding(
            padding: const EdgeInsets.all(16),
            child: Row(
              children: [
                Expanded(
                  child: FilledButton.icon(
                    onPressed: _tomarFoto,
                    icon:      const Icon(Icons.camera_alt),
                    label:     const Text('Cámara'),
                  ),
                ),
                const SizedBox(width: 8),
                Expanded(
                  child: OutlinedButton.icon(
                    onPressed: _abrirGaleria,
                    icon:      const Icon(Icons.photo_library),
                    label:     const Text('Galería'),
                  ),
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

**Prueba esto:**

```bash
flutter run
```

- Pulsa "Cámara" → se abre la cámara del dispositivo → toma la foto → aparece en pantalla.
- Pulsa "Galería" → selecciona una imagen → reemplaza la vista previa.

---

## Paso 4 — Múltiples fotos con Riverpod

**`lib/features/camara/media_notifier.dart`:**

```dart
// lib/features/camara/media_notifier.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:image_picker/image_picker.dart';
import '../../core/permisos_helper.dart';
import 'servicio_imagenes.dart';

class MediaNotifier extends Notifier<List<XFile>> {
  late final ServicioImagenes _servicio;

  @override
  List<XFile> build() {
    _servicio = ServicioImagenes();
    return [];
  }

  Future<void> tomarFoto() async {
    final ok = await PermisosHelper.solicitarCamara();
    if (!ok) return;
    final foto = await _servicio.tomarFoto();
    if (foto != null) state = [...state, foto];
  }

  Future<void> seleccionarDeGaleria() async {
    final ok = await PermisosHelper.solicitarGaleria();
    if (!ok) return;
    final seleccionadas = await _servicio.seleccionarVarias();
    if (seleccionadas.isNotEmpty) state = [...state, ...seleccionadas];
  }

  void eliminar(String path) {
    state = state.where((f) => f.path != path).toList();
  }

  void limpiarTodo() => state = [];
}

final mediaProvider = NotifierProvider<MediaNotifier, List<XFile>>(
  MediaNotifier.new,
);
```

Pantalla del Paso 4 — GridView con múltiples fotos:

```dart
// Cambia const int paso = 4 y usa _Paso4 como home

import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'features/camara/media_notifier.dart';

class _Paso4 extends ConsumerWidget {
  const _Paso4();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final fotos = ref.watch(mediaProvider);

    return Scaffold(
      appBar: AppBar(
        title: Text('Paso 4 — Fotos (${fotos.length})'),
        actions: [
          if (fotos.isNotEmpty)
            IconButton(
              icon:      const Icon(Icons.delete_sweep),
              onPressed: () =>
                  ref.read(mediaProvider.notifier).limpiarTodo(),
            ),
        ],
      ),
      body: fotos.isEmpty
          ? const Center(child: Text('Sin fotos aún'))
          : GridView.builder(
              padding: const EdgeInsets.all(8),
              gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount:   3,
                crossAxisSpacing: 4,
                mainAxisSpacing:  4,
              ),
              itemCount: fotos.length,
              itemBuilder: (_, i) => Stack(
                fit: StackFit.expand,
                children: [
                  ClipRRect(
                    borderRadius: BorderRadius.circular(6),
                    child: Image.file(File(fotos[i].path), fit: BoxFit.cover),
                  ),
                  Positioned(
                    top: 4, right: 4,
                    child: GestureDetector(
                      onTap: () => ref
                          .read(mediaProvider.notifier)
                          .eliminar(fotos[i].path),
                      child: Container(
                        padding: const EdgeInsets.all(4),
                        decoration: const BoxDecoration(
                          color: Colors.black54,
                          shape: BoxShape.circle,
                        ),
                        child: const Icon(Icons.close,
                            color: Colors.white, size: 14),
                      ),
                    ),
                  ),
                ],
              ),
            ),
      floatingActionButton: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          FloatingActionButton.small(
            heroTag:   'galeria',
            onPressed: () => ref
                .read(mediaProvider.notifier)
                .seleccionarDeGaleria(),
            child: const Icon(Icons.photo_library),
          ),
          const SizedBox(height: 8),
          FloatingActionButton(
            heroTag:   'camara',
            onPressed: () => ref.read(mediaProvider.notifier).tomarFoto(),
            child: const Icon(Icons.camera_alt),
          ),
        ],
      ),
    );
  }
}
```

**Prueba esto:**

```bash
flutter run
```

- Pulsa el FAB de cámara → toma 3 fotos → aparecen en el grid 3x3.
- Pulsa el FAB de galería → selecciona varias → se suman al grid.
- Toca la X de una miniatura → se elimina solo esa foto.
- Pulsa el icono de basura en el AppBar → borra todas.

---

## Paso 5 — GPS: posición única

**`lib/features/gps/servicio_ubicacion.dart`:**

```dart
// lib/features/gps/servicio_ubicacion.dart
import 'package:geolocator/geolocator.dart';

class ServicioUbicacion {
  Future<Position> obtenerPosicion() =>
      Geolocator.getCurrentPosition(
        locationSettings: const LocationSettings(
          accuracy: LocationAccuracy.high,
        ),
      );

  Stream<Position> streamPosiciones({int metrosMinimos = 5}) =>
      Geolocator.getPositionStream(
        locationSettings: LocationSettings(
          accuracy:       LocationAccuracy.high,
          distanceFilter: metrosMinimos,
        ),
      );

  double distanciaMetros({
    required double latA, required double lonA,
    required double latB, required double lonB,
  }) => Geolocator.distanceBetween(latA, lonA, latB, lonB);
}
```

Pantalla del Paso 5 — lectura única:

```dart
// Cambia const int paso = 5 y usa _Paso5 como home

import 'package:geolocator/geolocator.dart';
import 'core/permisos_helper.dart';
import 'features/gps/servicio_ubicacion.dart';

class _Paso5 extends StatefulWidget {
  const _Paso5();

  @override
  State<_Paso5> createState() => _Paso5State();
}

class _Paso5State extends State<_Paso5> {
  final _servicio = ServicioUbicacion();
  Position? _pos;
  bool      _cargando = false;

  Future<void> _leerUbicacion() async {
    final ok = await PermisosHelper.solicitarUbicacion();
    if (!ok) return;

    setState(() => _cargando = true);
    try {
      final pos = await _servicio.obtenerPosicion();
      setState(() {
        _pos      = pos;
        _cargando = false;
      });
    } catch (e) {
      setState(() => _cargando = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Hardware App — Paso 5 (GPS)')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment:  MainAxisAlignment.center,
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: _cargando
                    ? const Center(child: CircularProgressIndicator())
                    : _pos == null
                        ? const Text('Sin ubicación — pulsa el botón',
                            textAlign: TextAlign.center)
                        : Column(
                            crossAxisAlignment: CrossAxisAlignment.start,
                            children: [
                              _Fila('Latitud',   _pos!.latitude.toStringAsFixed(6)),
                              _Fila('Longitud',  _pos!.longitude.toStringAsFixed(6)),
                              _Fila('Altitud',   '${_pos!.altitude.toStringAsFixed(1)} m'),
                              _Fila('Precisión', '±${_pos!.accuracy.toStringAsFixed(0)} m'),
                              _Fila('Velocidad', '${(_pos!.speed * 3.6).toStringAsFixed(1)} km/h'),
                            ],
                          ),
              ),
            ),
            const SizedBox(height: 24),
            FilledButton.icon(
              onPressed: _cargando ? null : _leerUbicacion,
              icon:      const Icon(Icons.my_location),
              label:     const Text('Obtener mi ubicación'),
            ),
          ],
        ),
      ),
    );
  }
}

class _Fila extends StatelessWidget {
  final String etiqueta, valor;
  const _Fila(this.etiqueta, this.valor);

  @override
  Widget build(BuildContext context) => Padding(
    padding: const EdgeInsets.symmetric(vertical: 4),
    child: Row(
      children: [
        SizedBox(
          width: 90,
          child: Text(etiqueta,
              style: TextStyle(color: Colors.grey.shade600)),
        ),
        Text(valor,
            style: const TextStyle(fontWeight: FontWeight.w600)),
      ],
    ),
  );
}
```

**Prueba esto:**

```bash
flutter run
```

- Pulsa "Obtener mi ubicación" → aparece el diálogo del sistema.
- Acepta → se muestran las coordenadas reales del dispositivo.
- El spinner desaparece cuando llega la posición.

---

## Paso 6 — GPS: stream continuo con Riverpod

**`lib/features/gps/ubicacion_notifier.dart`:**

```dart
// lib/features/gps/ubicacion_notifier.dart
import 'dart:async';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:geolocator/geolocator.dart';
import '../../core/permisos_helper.dart';
import 'servicio_ubicacion.dart';

class UbicacionState {
  final Position?      posicion;
  final bool           rastreando;
  final bool           tienePermiso;
  final List<Position> historial;

  const UbicacionState({
    this.posicion     = null,
    this.rastreando   = false,
    this.tienePermiso = false,
    this.historial    = const [],
  });

  UbicacionState copyWith({
    Position?       posicion,
    bool?           rastreando,
    bool?           tienePermiso,
    List<Position>? historial,
  }) => UbicacionState(
    posicion:     posicion     ?? this.posicion,
    rastreando:   rastreando   ?? this.rastreando,
    tienePermiso: tienePermiso ?? this.tienePermiso,
    historial:    historial    ?? this.historial,
  );
}

class UbicacionNotifier extends Notifier<UbicacionState> {
  late final ServicioUbicacion _servicio;
  StreamSubscription<Position>? _sub;

  @override
  UbicacionState build() {
    _servicio = ServicioUbicacion();
    // SIEMPRE cancelar el stream cuando el provider se destruye
    ref.onDispose(() => _sub?.cancel());
    return const UbicacionState();
  }

  Future<void> solicitarPermiso() async {
    final ok = await PermisosHelper.solicitarUbicacion();
    state = state.copyWith(tienePermiso: ok);
  }

  Future<void> obtenerUnaVez() async {
    if (!state.tienePermiso) await solicitarPermiso();
    if (!state.tienePermiso) return;
    final pos = await _servicio.obtenerPosicion();
    state = state.copyWith(posicion: pos);
  }

  void iniciarRastreo() {
    if (state.rastreando || !state.tienePermiso) return;
    state = state.copyWith(rastreando: true);

    _sub = _servicio.streamPosiciones(metrosMinimos: 3).listen((pos) {
      state = state.copyWith(
        posicion:  pos,
        historial: [...state.historial, pos].take(50).toList(),
      );
    });
  }

  void detenerRastreo() {
    _sub?.cancel();
    _sub = null;
    state = state.copyWith(rastreando: false);
  }

  void limpiarHistorial() => state = state.copyWith(historial: []);
}

final ubicacionProvider =
    NotifierProvider<UbicacionNotifier, UbicacionState>(UbicacionNotifier.new);
```

Pantalla del Paso 6 — rastreo en tiempo real:

```dart
// Cambia const int paso = 6 y usa _Paso6 como home
import 'features/gps/ubicacion_notifier.dart';

class _Paso6 extends ConsumerWidget {
  const _Paso6();

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final estado = ref.watch(ubicacionProvider);
    final pos    = estado.posicion;
    final cs     = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: Text('Paso 6 — GPS${estado.rastreando ? " · EN VIVO" : ""}'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: pos == null
                    ? Text(
                        estado.tienePermiso
                            ? 'Presiona un botón para obtener ubicación'
                            : 'Solicita el permiso de ubicación primero',
                        style: TextStyle(color: cs.onSurfaceVariant),
                      )
                    : Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          _Fila('Latitud',   pos.latitude.toStringAsFixed(6)),
                          _Fila('Longitud',  pos.longitude.toStringAsFixed(6)),
                          _Fila('Precisión', '±${pos.accuracy.toStringAsFixed(0)} m'),
                          _Fila('Velocidad', '${(pos.speed * 3.6).toStringAsFixed(1)} km/h'),
                          _Fila('Historial', '${estado.historial.length} pts'),
                        ],
                      ),
              ),
            ),
            const SizedBox(height: 16),
            if (!estado.tienePermiso)
              FilledButton.icon(
                onPressed: () => ref
                    .read(ubicacionProvider.notifier)
                    .solicitarPermiso(),
                icon:  const Icon(Icons.security),
                label: const Text('Solicitar permiso'),
              )
            else ...[
              Row(children: [
                Expanded(
                  child: FilledButton.icon(
                    onPressed: () => ref
                        .read(ubicacionProvider.notifier)
                        .obtenerUnaVez(),
                    icon:  const Icon(Icons.my_location),
                    label: const Text('Una vez'),
                  ),
                ),
                const SizedBox(width: 8),
                Expanded(
                  child: estado.rastreando
                      ? OutlinedButton.icon(
                          onPressed: () => ref
                              .read(ubicacionProvider.notifier)
                              .detenerRastreo(),
                          icon:  const Icon(Icons.stop),
                          label: const Text('Detener'),
                        )
                      : FilledButton.tonal(
                          onPressed: () => ref
                              .read(ubicacionProvider.notifier)
                              .iniciarRastreo(),
                          icon:  const Icon(Icons.play_arrow),
                          label: const Text('Rastrear'),
                        ),
                ),
              ]),
              if (estado.historial.isNotEmpty) ...[
                const SizedBox(height: 8),
                TextButton(
                  onPressed: () => ref
                      .read(ubicacionProvider.notifier)
                      .limpiarHistorial(),
                  child: Text(
                      'Limpiar historial (${estado.historial.length} pts)'),
                ),
              ],
            ],
          ],
        ),
      ),
    );
  }
}
```

**Prueba esto:**

```bash
flutter run
```

- Pulsa "Solicitar permiso" → acepta en el diálogo.
- Pulsa "Una vez" → aparecen las coordenadas.
- Pulsa "Rastrear" → el contador de historial sube al moverte.
- Pulsa "Detener" → el rastreo para.
- El AppBar muestra "EN VIVO" mientras rastrea.

---

## Paso 7 — App final: `NavigationBar` con dos tabs

**`lib/main.dart` (Paso 7 — final):**

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'features/camara/pantalla_media.dart';
import 'features/gps/pantalla_gps.dart';

const int paso = 7;

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const ProviderScope(child: HardwareApp()));
}

class HardwareApp extends StatelessWidget {
  const HardwareApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Hardware App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.indigo),
        useMaterial3: true,
      ),
      home: const PantallaRaiz(),
    );
  }
}

final _tabProvider = StateProvider<int>((ref) => 0);

class PantallaRaiz extends ConsumerWidget {
  const PantallaRaiz({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final indice = ref.watch(_tabProvider);

    return Scaffold(
      body: switch (indice) {
        0 => const PantallaMedia(),
        1 => const PantallaGps(),
        _ => const PantallaMedia(),
      },
      bottomNavigationBar: NavigationBar(
        selectedIndex:         indice,
        onDestinationSelected: (i) =>
            ref.read(_tabProvider.notifier).state = i,
        destinations: const [
          NavigationDestination(
            icon:         Icon(Icons.camera_alt_outlined),
            selectedIcon: Icon(Icons.camera_alt),
            label:        'Cámara',
          ),
          NavigationDestination(
            icon:         Icon(Icons.location_on_outlined),
            selectedIcon: Icon(Icons.location_on),
            label:        'GPS',
          ),
        ],
      ),
    );
  }
}
```

**`lib/features/camara/pantalla_media.dart`:**

```dart
// lib/features/camara/pantalla_media.dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'media_notifier.dart';

class PantallaMedia extends ConsumerWidget {
  const PantallaMedia({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final fotos = ref.watch(mediaProvider);

    return Scaffold(
      appBar: AppBar(
        title: Text('Cámara · ${fotos.length} fotos'),
        actions: [
          if (fotos.isNotEmpty)
            IconButton(
              icon:      const Icon(Icons.delete_sweep),
              onPressed: () =>
                  ref.read(mediaProvider.notifier).limpiarTodo(),
            ),
        ],
      ),
      body: fotos.isEmpty
          ? const Center(
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Icon(Icons.camera_alt_outlined,
                      size: 64, color: Colors.grey),
                  SizedBox(height: 12),
                  Text('Toma una foto o abre la galería',
                      style: TextStyle(color: Colors.grey)),
                ],
              ),
            )
          : GridView.builder(
              padding: const EdgeInsets.all(8),
              gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount:   3,
                crossAxisSpacing: 4,
                mainAxisSpacing:  4,
              ),
              itemCount: fotos.length,
              itemBuilder: (_, i) => Stack(
                fit: StackFit.expand,
                children: [
                  ClipRRect(
                    borderRadius: BorderRadius.circular(6),
                    child: Image.file(File(fotos[i].path), fit: BoxFit.cover),
                  ),
                  Positioned(
                    top: 4, right: 4,
                    child: GestureDetector(
                      onTap: () => ref
                          .read(mediaProvider.notifier)
                          .eliminar(fotos[i].path),
                      child: Container(
                        padding: const EdgeInsets.all(4),
                        decoration: const BoxDecoration(
                          color: Colors.black54,
                          shape: BoxShape.circle,
                        ),
                        child: const Icon(Icons.close,
                            color: Colors.white, size: 14),
                      ),
                    ),
                  ),
                ],
              ),
            ),
      floatingActionButton: Column(
        mainAxisSize: MainAxisSize.min,
        children: [
          FloatingActionButton.small(
            heroTag:   'galeria',
            onPressed: () => ref
                .read(mediaProvider.notifier)
                .seleccionarDeGaleria(),
            child: const Icon(Icons.photo_library),
          ),
          const SizedBox(height: 8),
          FloatingActionButton(
            heroTag:   'camara',
            onPressed: () => ref.read(mediaProvider.notifier).tomarFoto(),
            child: const Icon(Icons.camera_alt),
          ),
        ],
      ),
    );
  }
}
```

**`lib/features/gps/pantalla_gps.dart`:**

```dart
// lib/features/gps/pantalla_gps.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'ubicacion_notifier.dart';

class PantallaGps extends ConsumerWidget {
  const PantallaGps({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final estado = ref.watch(ubicacionProvider);
    final pos    = estado.posicion;
    final cs     = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: Row(children: [
          const Text('GPS'),
          if (estado.rastreando) ...[
            const SizedBox(width: 8),
            Container(
              padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
              decoration: BoxDecoration(
                color:        cs.primaryContainer,
                borderRadius: BorderRadius.circular(12),
              ),
              child: Text('EN VIVO',
                  style: TextStyle(
                    color:      cs.primary,
                    fontSize:   11,
                    fontWeight: FontWeight.bold,
                  )),
            ),
          ],
        ]),
        actions: [
          if (estado.historial.isNotEmpty)
            IconButton(
              icon:      const Icon(Icons.delete_sweep),
              onPressed: () => ref
                  .read(ubicacionProvider.notifier)
                  .limpiarHistorial(),
            ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: pos == null
                    ? Text(
                        estado.tienePermiso
                            ? 'Presiona un botón para obtener ubicación'
                            : 'Solicita el permiso de ubicación primero',
                        style: TextStyle(color: cs.onSurfaceVariant),
                      )
                    : Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          _Fila('Latitud',   pos.latitude.toStringAsFixed(6)),
                          _Fila('Longitud',  pos.longitude.toStringAsFixed(6)),
                          _Fila('Precisión', '±${pos.accuracy.toStringAsFixed(0)} m'),
                          _Fila('Velocidad', '${(pos.speed * 3.6).toStringAsFixed(1)} km/h'),
                          if (estado.historial.isNotEmpty)
                            _Fila('Historial',
                                '${estado.historial.length} puntos'),
                        ],
                      ),
              ),
            ),
            const SizedBox(height: 16),
            if (!estado.tienePermiso)
              FilledButton.icon(
                onPressed: () => ref
                    .read(ubicacionProvider.notifier)
                    .solicitarPermiso(),
                icon:  const Icon(Icons.security),
                label: const Text('Solicitar permiso'),
              )
            else ...[
              Row(children: [
                Expanded(
                  child: FilledButton.icon(
                    onPressed: () => ref
                        .read(ubicacionProvider.notifier)
                        .obtenerUnaVez(),
                    icon:  const Icon(Icons.my_location),
                    label: const Text('Mi ubicación'),
                  ),
                ),
                const SizedBox(width: 8),
                Expanded(
                  child: estado.rastreando
                      ? OutlinedButton.icon(
                          onPressed: () => ref
                              .read(ubicacionProvider.notifier)
                              .detenerRastreo(),
                          icon:  const Icon(Icons.stop),
                          label: const Text('Detener'),
                        )
                      : FilledButton.tonal(
                          onPressed: () => ref
                              .read(ubicacionProvider.notifier)
                              .iniciarRastreo(),
                          icon:  const Icon(Icons.play_arrow),
                          label: const Text('Rastrear'),
                        ),
                ),
              ]),
            ],
            if (estado.historial.isNotEmpty) ...[
              const SizedBox(height: 16),
              Text('Últimas posiciones',
                  style: Theme.of(context)
                      .textTheme
                      .labelLarge
                      ?.copyWith(color: cs.primary)),
              const SizedBox(height: 8),
              Expanded(
                child: ListView.separated(
                  itemCount: estado.historial.length,
                  separatorBuilder: (_, __) => const Divider(height: 1),
                  itemBuilder: (_, i) {
                    final p = estado.historial.reversed.toList()[i];
                    return ListTile(
                      dense:   true,
                      leading: Icon(Icons.location_on,
                          size: 16, color: cs.primary),
                      title: Text(
                        '${p.latitude.toStringAsFixed(5)}, '
                        '${p.longitude.toStringAsFixed(5)}',
                        style: const TextStyle(fontSize: 13),
                      ),
                      subtitle: Text('±${p.accuracy.toStringAsFixed(0)} m',
                          style: const TextStyle(fontSize: 11)),
                    );
                  },
                ),
              ),
            ],
          ],
        ),
      ),
    );
  }
}

class _Fila extends StatelessWidget {
  final String etiqueta, valor;
  const _Fila(this.etiqueta, this.valor);

  @override
  Widget build(BuildContext context) => Padding(
    padding: const EdgeInsets.symmetric(vertical: 3),
    child: Row(
      children: [
        SizedBox(
          width: 80,
          child: Text(etiqueta,
              style: TextStyle(
                  color: Colors.grey.shade600, fontSize: 13)),
        ),
        Text(valor,
            style: const TextStyle(
                fontWeight: FontWeight.w600, fontSize: 13)),
      ],
    ),
  );
}
```

**Prueba esto:**

```bash
flutter run
```

- Navega entre los dos tabs con la `NavigationBar` inferior.
- **Tab Cámara:** toma fotos, ábrelas de la galería, elimínalas individualmente o todas.
- **Tab GPS:** solicita permiso, obtén ubicación, activa el rastreo en tiempo real.
- El historial de posiciones se lista debajo de los controles GPS.

---

## main.dart completo — referencia (Paso 7)

El `main.dart` del Paso 7 es la versión definitiva. Usa `_tabProvider` como `StateProvider<int>` para controlar la navegación entre los dos tabs sin un `StatefulWidget`. Cada pantalla vive en su propio archivo (`pantalla_media.dart`, `pantalla_gps.dart`) y gestiona su propio estado con `mediaProvider` y `ubicacionProvider`.

La clave de esta arquitectura es la separación de responsabilidades: `main.dart` solo conoce las dos pantallas raíz y el provider del tab activo. No sabe nada de cámaras, GPS ni permisos. Todo eso vive encapsulado en sus respectivas carpetas de `features/`.

---

## Proyecto final — estructura de archivos

```
lib/
├── main.dart                              ← HardwareApp + PantallaRaiz + _tabProvider
├── core/
│   └── permisos_helper.dart              ← PermisosHelper (estático, sin estado)
└── features/
    ├── camara/
    │   ├── servicio_imagenes.dart         ← ServicioImagenes (sin estado)
    │   ├── media_notifier.dart            ← MediaNotifier + mediaProvider
    │   └── pantalla_media.dart            ← PantallaMedia (ConsumerWidget)
    └── gps/
        ├── servicio_ubicacion.dart        ← ServicioUbicacion (sin estado)
        ├── ubicacion_notifier.dart        ← UbicacionNotifier + ubicacionProvider
        └── pantalla_gps.dart             ← PantallaGps (ConsumerWidget)
```

Flujo de dependencias:

```
PermisosHelper
  ├── MediaNotifier.tomarFoto()
  ├── MediaNotifier.seleccionarDeGaleria()
  └── UbicacionNotifier.solicitarPermiso()

ServicioImagenes  → MediaNotifier     → mediaProvider     → PantallaMedia
ServicioUbicacion → UbicacionNotifier → ubicacionProvider → PantallaGps
```

Cada capa tiene una responsabilidad única: los servicios abstraen la librería externa, los notifiers gestionan el estado y disparan las operaciones, y las pantallas solo leen el estado y llaman métodos del notifier. Esta separación permite reemplazar cualquier librería (por ejemplo, cambiar `geolocator` por otra) sin tocar la UI.

---

## Guía rápida de imports

```dart
// Permisos
import 'package:permission_handler/permission_handler.dart';
// Uso: await Permission.camera.request()

// Cámara / galería
import 'package:image_picker/image_picker.dart';
// Uso: ImagePicker().pickImage(source: ImageSource.camera)

// Mostrar foto (necesita dart:io)
import 'dart:io';
// Uso: Image.file(File(foto.path))

// GPS
import 'package:geolocator/geolocator.dart';
// Uso: Geolocator.getCurrentPosition(...)

// Stream GPS
import 'dart:async';
// Uso: StreamSubscription<Position>? _sub = ...

// Riverpod
import 'package:flutter_riverpod/flutter_riverpod.dart';
```

---

## Cuándo usar qué

```
Situación                                          Herramienta
────────────────────────────────────────────────────────────────────────────────
Pedir permiso de cámara                            Permission.camera.request()
Pedir permiso de galería (Android 13+)             Permission.photos.request()
Pedir permiso de ubicación                         Permission.locationWhenInUse.request()
Redirigir al usuario a Ajustes del sistema         openAppSettings()
Tomar foto con cámara                              ImagePicker.pickImage(source: camera)
Seleccionar foto de galería                        ImagePicker.pickImage(source: gallery)
Seleccionar múltiples fotos                        ImagePicker.pickMultiImage()
Mostrar foto local en pantalla                     Image.file(File(xfile.path))
Obtener posición GPS una sola vez                  Geolocator.getCurrentPosition()
Rastrear posición en tiempo real                   Geolocator.getPositionStream()
Calcular distancia entre dos coordenadas           Geolocator.distanceBetween()
Cancelar stream GPS al destruir el provider        ref.onDispose(() => _sub?.cancel())
```

---

## Ejercicios propuestos

**1. Visor a pantalla completa**

Crea `lib/features/camara/pantalla_visor.dart` con un `ConsumerWidget` que reciba un `path: String`. Muestra la imagen con `InteractiveViewer(child: Image.file(File(path)))` para permitir zoom y pan. En `pantalla_media.dart`, envuelve cada miniatura con `GestureDetector` y navega a `PantallaVisor(path: fotos[i].path)` usando `Navigator.push(context, MaterialPageRoute(...))`.

**2. Punto de referencia GPS**

En `UbicacionState` añade `posicionReferencia: Position?`. En `UbicacionNotifier` crea `void guardarReferencia()` que asigne `posicion` a `posicionReferencia`. En `PantallaGps`, cuando hay referencia guardada, muestra la distancia con:

```dart
final dist = ServicioUbicacion().distanciaMetros(
  latA: pos.latitude,  lonA: pos.longitude,
  latB: ref.latitude,  lonB: ref.longitude,
);
```

Formatea como `'${dist.toStringAsFixed(0)} m'` o `'${(dist/1000).toStringAsFixed(2)} km'` si supera 1000 m.

**3. Historial GPS con timestamps**

`Position.timestamp` es un `DateTime?`. Modifica `PantallaGps` para mostrar en cada item del historial la hora formateada como `HH:MM:SS`. Añade también un `ListTile` al final que muestre la duración total del recorrido (diferencia entre el timestamp más reciente y el más antiguo).

**4. Badge en el tab de Cámara**

Usa `Badge` de Material 3 en la `NavigationDestination` de la pestaña Cámara:

```dart
NavigationDestination(
  icon: Badge(
    isLabelVisible: ref.watch(mediaProvider).isNotEmpty,
    label: Text('${ref.watch(mediaProvider).length}'),
    child: const Icon(Icons.camera_alt_outlined),
  ),
  label: 'Cámara',
),
```

El badge debe desaparecer cuando no hay fotos (`isLabelVisible: fotos.isNotEmpty`).

---

## Resumen

- `WidgetsFlutterBinding.ensureInitialized()` va siempre al inicio de `main()` cuando hay operaciones async antes de `runApp`.
- Construir en pasos pequeños (1 al 7) permite verificar cada pieza antes de agregar la siguiente — si algo falla, sabes exactamente dónde está el error.
- `PermisosHelper` centraliza toda la lógica de permisos. Los servicios y notifiers lo llaman pero nunca tocan `permission_handler` directamente.
- `ServicioImagenes` y `ServicioUbicacion` son clases puras sin estado — solo abstraen la librería externa. El estado vive en los notifiers.
- `ref.onDispose(() => _sub?.cancel())` en el `build()` del notifier garantiza que el stream GPS se cancela cuando el provider se destruye. Nunca canceles en el `dispose()` del widget.
- `[...lista, elemento].take(n).toList()` es el patrón para añadir al historial con límite máximo sin modificar el estado directamente.
- Separar `servicio / notifier / pantalla` en archivos distintos hace que cada pieza sea testeable independientemente y reemplazable sin tocar el resto.

---

> **Siguiente página →** Página 16: Notificaciones locales y push con FCM.
