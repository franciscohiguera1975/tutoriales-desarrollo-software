# Tutorial Flutter — Página 15
## Módulo 5 · Hardware
### Cámara y GPS — construcción incremental paso a paso

---

## Crear el proyecto

```bash
flutter create hardware_app
cd hardware_app
```

Agrega las dependencias en `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod:   ^3.3.0
  permission_handler: ^11.3.1
  image_picker:       ^1.1.2
  geolocator:         ^13.0.1
```

```bash
flutter pub get
```

Permisos nativos — agregar antes de continuar:

**`android/app/src/main/AndroidManifest.xml`** — dentro de `<manifest>`:
```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

**`ios/Runner/Info.plist`** — dentro de `<dict>`:
```xml
<key>NSCameraUsageDescription</key>
<string>Para tomar fotos con la app</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>Para seleccionar imágenes</string>
<key>NSLocationWhenInUseUsageDescription</key>
<string>Para mostrar tu ubicación actual</string>
```

---

## Paso 1 — App mínima funcionando

Antes de cualquier hardware, arranca la app con solo un `Scaffold`.
Verifica que compila y corre sin errores.

**`lib/main.dart`** — reemplaza todo el contenido:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

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

class PantallaRaiz extends StatelessWidget {
  const PantallaRaiz({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Hardware App')),
      body: const Center(child: Text('Paso 1 — App funcionando ✅')),
    );
  }
}
```

```bash
flutter run
# Debe mostrar la pantalla con el texto "Paso 1 — App funcionando ✅"
```

---

## Paso 2 — Verificar y pedir permisos

Crea el archivo de helper de permisos. Todavía no hay cámara ni GPS —
solo el código que pregunta al sistema si tiene los permisos:

**Crea `lib/core/permisos_helper.dart`:**

```dart
import 'package:permission_handler/permission_handler.dart';

class PermisosHelper {
  static Future<bool> solicitarCamara() async {
    final estado = await Permission.camera.request();
    return estado.isGranted;
  }

  static Future<bool> solicitarGaleria() async {
    final estado = await Permission.photos.request();
    return estado.isGranted;
  }

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

Agrega el import de geolocator que falta arriba del archivo:

```dart
import 'package:geolocator/geolocator.dart';
```

Ahora prueba el helper directo desde `main.dart`. Modifica
`PantallaRaiz` para mostrar botones que soliciten permisos:

```dart
// lib/main.dart — reemplaza PantallaRaiz

class PantallaRaiz extends StatefulWidget {
  const PantallaRaiz({super.key});

  @override
  State<PantallaRaiz> createState() => _PantallaRaizState();
}

class _PantallaRaizState extends State<PantallaRaiz> {
  String _resultado = 'Sin verificar';

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 2 — Permisos')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Text(
              _resultado,
              textAlign: TextAlign.center,
              style: const TextStyle(fontSize: 18),
            ),
            const SizedBox(height: 32),
            FilledButton.icon(
              icon:  const Icon(Icons.camera_alt),
              label: const Text('Pedir permiso de cámara'),
              onPressed: () async {
                final ok = await PermisosHelper.solicitarCamara();
                setState(() => _resultado =
                    ok ? '📸 Cámara: CONCEDIDO' : '❌ Cámara: DENEGADO');
              },
            ),
            const SizedBox(height: 12),
            FilledButton.icon(
              icon:  const Icon(Icons.photo_library),
              label: const Text('Pedir permiso de galería'),
              onPressed: () async {
                final ok = await PermisosHelper.solicitarGaleria();
                setState(() => _resultado =
                    ok ? '🖼️ Galería: CONCEDIDO' : '❌ Galería: DENEGADO');
              },
            ),
            const SizedBox(height: 12),
            FilledButton.icon(
              icon:  const Icon(Icons.location_on),
              label: const Text('Pedir permiso de ubicación'),
              onPressed: () async {
                final ok = await PermisosHelper.solicitarUbicacion();
                setState(() => _resultado =
                    ok ? '📍 Ubicación: CONCEDIDO' : '❌ Ubicación: DENEGADO');
              },
            ),
          ],
        ),
      ),
    );
  }
}
```

```bash
flutter run
# Presiona cada botón — debe aparecer el diálogo de permiso del sistema
# El texto en pantalla cambia según lo que respondas
```

---

## Paso 3 — Tomar una foto y mostrarla

Con los permisos funcionando, ahora agrega la cámara.
Crea el servicio y pruébalo de inmediato en la misma pantalla:

**Crea `lib/features/camara/servicio_imagenes.dart`:**

```dart
import 'package:image_picker/image_picker.dart';

class ServicioImagenes {
  final _picker = ImagePicker();

  Future<XFile?> tomarFoto({int calidad = 85}) {
    return _picker.pickImage(
      source:       ImageSource.camera,
      imageQuality: calidad,
    );
  }

  Future<XFile?> seleccionarDeGaleria({int calidad = 85}) {
    return _picker.pickImage(
      source:       ImageSource.gallery,
      imageQuality: calidad,
    );
  }

  Future<List<XFile>> seleccionarVarias({int limite = 5}) {
    return _picker.pickMultiImage(
      limit:        limite,
      imageQuality: 85,
    );
  }
}
```

Ahora modifica `PantallaRaiz` para probar la cámara:

```dart
// lib/main.dart — reemplaza PantallaRaiz

import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:image_picker/image_picker.dart';
import 'core/permisos_helper.dart';
import 'features/camara/servicio_imagenes.dart';

class PantallaRaiz extends StatefulWidget {
  const PantallaRaiz({super.key});

  @override
  State<PantallaRaiz> createState() => _PantallaRaizState();
}

class _PantallaRaizState extends State<PantallaRaiz> {
  final _servicio = ServicioImagenes();
  XFile? _foto;
  String _mensaje = 'Toca un botón para comenzar';

  Future<void> _tomarFoto() async {
    final ok = await PermisosHelper.solicitarCamara();
    if (!ok) {
      setState(() => _mensaje = '❌ Sin permiso de cámara');
      return;
    }
    final foto = await _servicio.tomarFoto();
    setState(() {
      _foto    = foto;
      _mensaje = foto != null ? '✅ Foto tomada' : 'Cancelado';
    });
  }

  Future<void> _abrirGaleria() async {
    final ok = await PermisosHelper.solicitarGaleria();
    if (!ok) {
      setState(() => _mensaje = '❌ Sin permiso de galería');
      return;
    }
    final foto = await _servicio.seleccionarDeGaleria();
    setState(() {
      _foto    = foto;
      _mensaje = foto != null ? '✅ Imagen seleccionada' : 'Cancelado';
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 3 — Cámara')),
      body: Column(
        children: [
          // Vista previa de la foto
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

          // Botones
          Padding(
            padding: const EdgeInsets.all(16),
            child: Row(
              children: [
                Expanded(
                  child: FilledButton.icon(
                    onPressed: _tomarFoto,
                    icon:  const Icon(Icons.camera_alt),
                    label: const Text('Cámara'),
                  ),
                ),
                const SizedBox(width: 8),
                Expanded(
                  child: OutlinedButton.icon(
                    onPressed: _abrirGaleria,
                    icon:  const Icon(Icons.photo_library),
                    label: const Text('Galería'),
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

```bash
flutter run
# Presiona "Cámara" — se abre la cámara del dispositivo
# Toma la foto — aparece en pantalla
# Presiona "Galería" — se abre la galería del sistema
```

---

## Paso 4 — Múltiples fotos con Riverpod

Ahora mueve el estado de la pantalla a un `NotifierProvider`.
La UI no cambia visualmente, pero el estado ya está en Riverpod:

**Crea `lib/features/camara/media_notifier.dart`:**

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:image_picker/image_picker.dart';
import 'servicio_imagenes.dart';
import '../../core/permisos_helper.dart';

class MediaNotifier extends Notifier<List<XFile>> {
  late final ServicioImagenes _servicio;

  @override
  List<XFile> build() {
    _servicio = ServicioImagenes();
    return [];   // lista vacía al iniciar
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

Actualiza `PantallaRaiz` para usar el provider y mostrar el grid:

```dart
// lib/main.dart — reemplaza PantallaRaiz

import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'features/camara/media_notifier.dart';

class PantallaRaiz extends ConsumerWidget {
  const PantallaRaiz({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final fotos = ref.watch(mediaProvider);
    final cs    = Theme.of(context).colorScheme;

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
              padding:     const EdgeInsets.all(8),
              gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount:   3,
                crossAxisSpacing: 4,
                mainAxisSpacing:  4,
              ),
              itemCount:   fotos.length,
              itemBuilder: (_, i) => Stack(
                fit: StackFit.expand,
                children: [
                  ClipRRect(
                    borderRadius: BorderRadius.circular(6),
                    child: Image.file(File(fotos[i].path),
                        fit: BoxFit.cover),
                  ),
                  Positioned(
                    top: 4, right: 4,
                    child: GestureDetector(
                      onTap: () => ref
                          .read(mediaProvider.notifier)
                          .eliminar(fotos[i].path),
                      child: Container(
                        padding:    const EdgeInsets.all(4),
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
            onPressed: () =>
                ref.read(mediaProvider.notifier).tomarFoto(),
            child: const Icon(Icons.camera_alt),
          ),
        ],
      ),
    );
  }
}
```

```bash
flutter run
# Toma varias fotos — aparecen en el grid
# Toca la X de una foto — se elimina
# Toca el ícono de la basura en el AppBar — borra todas
```

---

## Paso 5 — GPS: leer la posición

El módulo GPS se construye igual: primero solo leer la posición
sin stream ni Riverpod, para verificar que funciona:

**Crea `lib/features/gps/servicio_ubicacion.dart`:**

```dart
import 'package:geolocator/geolocator.dart';

class ServicioUbicacion {
  // Leer la posición una sola vez
  Future<Position> obtenerPosicion() {
    return Geolocator.getCurrentPosition(
      locationSettings: const LocationSettings(
        accuracy: LocationAccuracy.high,
      ),
    );
  }

  // Stream de posiciones continuas
  Stream<Position> streamPosiciones({int metrosMinimos = 5}) {
    return Geolocator.getPositionStream(
      locationSettings: LocationSettings(
        accuracy:       LocationAccuracy.high,
        distanceFilter: metrosMinimos,
      ),
    );
  }

  double distanciaMetros({
    required double latA, required double lonA,
    required double latB, required double lonB,
  }) => Geolocator.distanceBetween(latA, lonA, latB, lonB);
}
```

Prueba el GPS directamente en `PantallaRaiz` — cambia `ConsumerWidget`
a `StatefulWidget` para este paso:

```dart
// lib/main.dart — reemplaza solo PantallaRaiz

import 'package:geolocator/geolocator.dart';
import 'core/permisos_helper.dart';
import 'features/gps/servicio_ubicacion.dart';

class PantallaRaiz extends StatefulWidget {
  const PantallaRaiz({super.key});
  @override
  State<PantallaRaiz> createState() => _PantallaRaizState();
}

class _PantallaRaizState extends State<PantallaRaiz> {
  final _servicio = ServicioUbicacion();
  Position? _pos;
  bool _cargando = false;

  Future<void> _leerUbicacion() async {
    final ok = await PermisosHelper.solicitarUbicacion();
    if (!ok) {
      setState(() => _pos = null);
      return;
    }

    setState(() => _cargando = true);
    try {
      final pos = await _servicio.obtenerPosicion();
      setState(() { _pos = pos; _cargando = false; });
    } catch (e) {
      setState(() => _cargando = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 5 — GPS')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            if (_cargando)
              const Center(child: CircularProgressIndicator())
            else if (_pos != null) ...[
              _Fila('Latitud',    _pos!.latitude.toStringAsFixed(6)),
              _Fila('Longitud',   _pos!.longitude.toStringAsFixed(6)),
              _Fila('Altitud',    '${_pos!.altitude.toStringAsFixed(1)} m'),
              _Fila('Precisión',  '±${_pos!.accuracy.toStringAsFixed(0)} m'),
              _Fila('Velocidad',  '${(_pos!.speed * 3.6).toStringAsFixed(1)} km/h'),
            ] else
              const Center(
                child: Text('Presiona el botón para obtener ubicación',
                    textAlign: TextAlign.center),
              ),
            const SizedBox(height: 32),
            FilledButton.icon(
              onPressed: _cargando ? null : _leerUbicacion,
              icon:  const Icon(Icons.my_location),
              label: const Text('Obtener mi ubicación'),
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
        Text(valor, style: const TextStyle(fontWeight: FontWeight.w600)),
      ],
    ),
  );
}
```

```bash
flutter run
# Presiona "Obtener mi ubicación"
# Aparece el diálogo de permiso del sistema
# Al aceptar, se muestran las coordenadas reales
```

---

## Paso 6 — GPS con stream y Riverpod

Igual que con la cámara, mueve el GPS a un provider.
Añade rastreo continuo con stream:

**Crea `lib/features/gps/ubicacion_notifier.dart`:**

```dart
import 'dart:async';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:geolocator/geolocator.dart';
import 'servicio_ubicacion.dart';
import '../../core/permisos_helper.dart';

class UbicacionState {
  final Position? posicion;
  final bool      rastreando;
  final bool      tienePermiso;
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
    ref.onDispose(() => _sub?.cancel()); // limpieza automática
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
    NotifierProvider<UbicacionNotifier, UbicacionState>(
  UbicacionNotifier.new,
);
```

Prueba el GPS con Riverpod actualizando `PantallaRaiz`:

```dart
// lib/main.dart — reemplaza PantallaRaiz

import 'features/gps/ubicacion_notifier.dart';

class PantallaRaiz extends ConsumerWidget {
  const PantallaRaiz({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final estado = ref.watch(ubicacionProvider);
    final pos    = estado.posicion;
    final cs     = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: Text('Paso 6 — GPS'
            '${estado.rastreando ? " 🔴 EN VIVO" : ""}'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // Datos de posición
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16),
                child: pos == null
                    ? const Text('Sin ubicación — presiona un botón')
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

            // Botones
            if (!estado.tienePermiso)
              FilledButton.icon(
                onPressed: () => ref
                    .read(ubicacionProvider.notifier)
                    .solicitarPermiso(),
                icon:  const Icon(Icons.security),
                label: const Text('Solicitar permiso'),
              )
            else ...[
              FilledButton.icon(
                onPressed: () => ref
                    .read(ubicacionProvider.notifier)
                    .obtenerUnaVez(),
                icon:  const Icon(Icons.my_location),
                label: const Text('Ubicación actual'),
              ),
              const SizedBox(height: 8),
              estado.rastreando
                  ? OutlinedButton.icon(
                      onPressed: () => ref
                          .read(ubicacionProvider.notifier)
                          .detenerRastreo(),
                      icon:  const Icon(Icons.stop),
                      label: const Text('Detener rastreo'),
                    )
                  : FilledButton.tonal(
                      onPressed: () => ref
                          .read(ubicacionProvider.notifier)
                          .iniciarRastreo(),
                      icon:  const Icon(Icons.play_arrow),
                      label: const Text('Iniciar rastreo'),
                    ),
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
              style: TextStyle(color: Colors.grey.shade600,
                  fontSize: 13)),
        ),
        Text(valor, style: const TextStyle(
            fontWeight: FontWeight.w600, fontSize: 13)),
      ],
    ),
  );
}
```

```bash
flutter run
# Presiona "Solicitar permiso" → acepta en el diálogo
# Presiona "Ubicación actual" → aparecen las coordenadas
# Presiona "Iniciar rastreo" → los datos se actualizan al moverte
# El título muestra "🔴 EN VIVO" mientras rastrea
# Presiona "Detener rastreo" para parar
```

---

## Paso 7 — App final con NavigationBar

Ahora junta los dos módulos en la app definitiva con dos tabs.
Este es el único paso donde tocas `main.dart` estructuralmente:

**`lib/main.dart` final — reemplaza todo:**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'features/camara/pantalla_media.dart';
import 'features/gps/pantalla_gps.dart';

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

final _tabProvider = StateProvider<int>((ref) => 0);
```

Ahora crea las dos pantallas finales como archivos separados.

**Crea `lib/features/camara/pantalla_media.dart`:**

```dart
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
        title: Text('Cámara · ${fotos.length}'),
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
          ? const Center(child: Text('Toma una foto o abre la galería'))
          : GridView.builder(
              padding:     const EdgeInsets.all(8),
              gridDelegate:
                  const SliverGridDelegateWithFixedCrossAxisCount(
                crossAxisCount:   3,
                crossAxisSpacing: 4,
                mainAxisSpacing:  4,
              ),
              itemCount:   fotos.length,
              itemBuilder: (_, i) => Stack(
                fit: StackFit.expand,
                children: [
                  ClipRRect(
                    borderRadius: BorderRadius.circular(6),
                    child: Image.file(File(fotos[i].path),
                        fit: BoxFit.cover),
                  ),
                  Positioned(
                    top: 4, right: 4,
                    child: GestureDetector(
                      onTap: () => ref
                          .read(mediaProvider.notifier)
                          .eliminar(fotos[i].path),
                      child: Container(
                        padding:    const EdgeInsets.all(4),
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
            onPressed: () =>
                ref.read(mediaProvider.notifier).tomarFoto(),
            child: const Icon(Icons.camera_alt),
          ),
        ],
      ),
    );
  }
}
```

**Crea `lib/features/gps/pantalla_gps.dart`:**

```dart
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
        title: Row(
          children: [
            const Text('GPS'),
            if (estado.rastreando) ...[
              const SizedBox(width: 8),
              Container(
                padding: const EdgeInsets.symmetric(
                    horizontal: 8, vertical: 2),
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
          ],
        ),
        actions: [
          if (estado.historial.isNotEmpty)
            IconButton(
              icon:      const Icon(Icons.delete_sweep),
              onPressed: () =>
                  ref.read(ubicacionProvider.notifier).limpiarHistorial(),
            ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [

            // Tarjeta de datos
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
                          _Fila('Latitud',
                              pos.latitude.toStringAsFixed(6)),
                          _Fila('Longitud',
                              pos.longitude.toStringAsFixed(6)),
                          _Fila('Precisión',
                              '±${pos.accuracy.toStringAsFixed(0)} m'),
                          _Fila('Velocidad',
                              '${(pos.speed * 3.6).toStringAsFixed(1)} km/h'),
                          if (estado.historial.isNotEmpty)
                            _Fila('Historial',
                                '${estado.historial.length} puntos'),
                        ],
                      ),
              ),
            ),

            const SizedBox(height: 16),

            // Controles
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

            // Historial
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
                  separatorBuilder: (_, __) =>
                      const Divider(height: 1),
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
                      subtitle: Text(
                        '±${p.accuracy.toStringAsFixed(0)} m',
                        style: const TextStyle(fontSize: 11),
                      ),
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

```bash
flutter run
# Navega entre los dos tabs con la NavigationBar inferior
# Tab Cámara: toma fotos, ábrelas de la galería, elimínalas
# Tab GPS: solicita permiso, obtén ubicación, activa el rastreo
```

---

## Estado final del proyecto

```
lib/
├── main.dart                              ← modificado en cada paso
├── core/
│   └── permisos_helper.dart              ← paso 2
├── features/
│   ├── camara/
│   │   ├── servicio_imagenes.dart        ← paso 3
│   │   ├── media_notifier.dart           ← paso 4
│   │   └── pantalla_media.dart           ← paso 7
│   └── gps/
│       ├── servicio_ubicacion.dart       ← paso 5
│       ├── ubicacion_notifier.dart       ← paso 6
│       └── pantalla_gps.dart             ← paso 7
```

---

## Ejercicios propuestos

1. **Visor a pantalla completa** — Crea `lib/features/camara/pantalla_visor.dart`.
   En `pantalla_media.dart`, envuelve cada miniatura con `GestureDetector`
   y navega a `PantallaVisor(path: fotos[i].path)` usando `Navigator.push`.
   Muestra la imagen con `InteractiveViewer` para zoom y pan.

2. **Punto de referencia** — En `UbicacionState` añade `posicionReferencia: Position?`.
   En `UbicacionNotifier` crea `guardarReferencia()` que asigne `posicion` actual
   a `posicionReferencia`. En `pantalla_gps.dart` muestra la distancia en metros
   usando `ServicioUbicacion.distanciaMetros` cuando hay referencia guardada.

3. **Historial con timestamp** — El `Position` de geolocator tiene
   `timestamp: DateTime?`. Modifica `pantalla_gps.dart` para mostrar la hora
   de cada punto del historial formateada como `HH:MM:SS`. Añade también
   la diferencia de tiempo entre el punto más antiguo y el más reciente.

4. **Badge de contador en el tab** — Usa `Badge` de Material 3 en la
   `NavigationDestination` de Cámara para mostrar el número de fotos acumuladas.
   El badge debe desaparecer cuando no hay fotos. Lee el valor con
   `ref.watch(mediaProvider).length` directamente en `PantallaRaiz`.

---

## Resumen de la página 15

- `WidgetsFlutterBinding.ensureInitialized()` va siempre al inicio de `main()` cuando hay operaciones async antes de `runApp`.
- Construir en pasos pequeños permite verificar cada pieza antes de agregar la siguiente — si algo falla sabes exactamente dónde está el error.
- `PermisosHelper` centraliza toda la lógica de permisos. Los servicios y notifiers lo llaman pero nunca tocan `permission_handler` directamente.
- `ServicioImagenes` y `ServicioUbicacion` son clases puras sin estado — solo abstraen la librería externa. El estado vive en los notifiers.
- `ref.onDispose(() => _sub?.cancel())` en el `build()` del notifier garantiza que el stream GPS se cancela cuando el provider se destruye. Nunca canceles el stream en el `dispose()` del widget.
- `[...lista, elemento].take(n).toList()` es el patrón para añadir al historial con límite sin modificar el estado directamente.
- Separar `servicio / notifier / pantalla` en archivos distintos hace que cada pieza sea testeable independientemente y reemplazable sin tocar el resto.

---

> **Siguiente página →** Página 16: Notificaciones locales y push con FCM.