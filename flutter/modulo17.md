# Tutorial Flutter — Página 17
## Módulo 5 · Multimedia
### Audio y video con audioplayers, video_player y Riverpod

---

En esta página construyes paso a paso un reproductor de audio y video completo usando `audioplayers`, `video_player` y `chewie`, gestionando el estado con Riverpod. Pasarás de un botón play básico a una app con navegación por pestañas, playlist y controles profesionales.

---

## ¿Qué aprenderás?

```
Característica              | audioplayers              | video_player + chewie
----------------------------|---------------------------|-------------------------------
Qué reproduce               | MP3, WAV, OGG, AAC        | MP4, MOV, HLS, DASH
Controles de UI             | Manuales (tú los creas)   | Automáticos con chewie
Soporte de plataformas      | Android, iOS, Web, Desktop| Android, iOS, Web
Audio en background         | Si (con AndroidManifest)  | No nativo
Streaming vs archivo        | URL + Asset + File        | URL + Asset + File
Integración Riverpod        | NotifierProvider directo   | NotifierProvider + controller
Progreso de reproducción    | Streams (posicion/dur.)   | _controller.value.position
Pantalla completa           | No aplica                 | ChewieController(allowFullScreen)
```

---

## Patrones clave

### Patrón 1: `AudioPlayer` básico — play / pause / stop / seek

```dart
import 'package:audioplayers/audioplayers.dart';

final AudioPlayer _player = AudioPlayer();

// Reproducir desde URL
await _player.play(UrlSource('https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3'));

// Reproducir desde asset (declarado en pubspec.yaml)
await _player.play(AssetSource('musica/cancion.mp3'));

// Control básico
await _player.pause();
await _player.resume();
await _player.stop();
await _player.seek(Duration(seconds: 30));

// Volumen (0.0 a 1.0)
await _player.setVolume(0.8);

// Liberar recursos al salir
await _player.dispose();
```

> **Mini-ejercicio (5 min):** Crea un `StatefulWidget` con tres `ElevatedButton`: "Play", "Pause" y "Stop". Al presionar Play, ejecuta `_player.play(UrlSource('https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3'))`. Después de Play imprime `_player.state` con un `debugPrint` y verifica que dice `PlayerState.playing`.

---

### Patrón 2: Streams de `AudioPlayer` — posición y duración

```dart
// Duración total del audio
_player.onDurationChanged.listen((Duration d) {
  setState(() => _duracion = d);
});

// Posición actual (progreso)
_player.onPositionChanged.listen((Duration p) {
  setState(() => _posicion = p);
});

// Estado del reproductor
_player.onPlayerStateChanged.listen((PlayerState s) {
  // PlayerState.playing / paused / stopped / completed
  setState(() => _estado = s);
});

// Conversión para Slider (valor entre 0.0 y 1.0)
double progreso = _duracion.inSeconds > 0
    ? _posicion.inSeconds / _duracion.inSeconds
    : 0.0;

// Formato mm:ss
String formatear(Duration d) {
  final min = d.inMinutes.toString().padLeft(2, '0');
  final seg = (d.inSeconds % 60).toString().padLeft(2, '0');
  return '$min:$seg';
}
```

> **Mini-ejercicio (5 min):** Conecta los streams y añade un `LinearProgressIndicator(value: progreso)` debajo del audio. Encima muestra un `Text('${formatear(_posicion)} / ${formatear(_duracion)}')`. Verifica que la barra avanza al reproducir.

---

### Patrón 3: `AudioNotifier` con Riverpod — estado centralizado

```dart
import 'dart:async';
import 'package:audioplayers/audioplayers.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Modelo inmutable de estado
class AudioState {
  final PlayerState estado;
  final Duration posicion;
  final Duration duracion;
  final double volumen;

  const AudioState({
    required this.estado,
    required this.posicion,
    required this.duracion,
    required this.volumen,
  });

  // Progreso 0.0 a 1.0 calculado desde duracion y posicion
  double get progreso =>
      duracion.inSeconds > 0 ? posicion.inSeconds / duracion.inSeconds : 0.0;

  String get posicionFormateada {
    final m = posicion.inMinutes.toString().padLeft(2, '0');
    final s = (posicion.inSeconds % 60).toString().padLeft(2, '0');
    return '$m:$s';
  }

  String get duracionFormateada {
    final m = duracion.inMinutes.toString().padLeft(2, '0');
    final s = (duracion.inSeconds % 60).toString().padLeft(2, '0');
    return '$m:$s';
  }

  AudioState copyWith({
    PlayerState? estado,
    Duration? posicion,
    Duration? duracion,
    double? volumen,
  }) =>
      AudioState(
        estado: estado ?? this.estado,
        posicion: posicion ?? this.posicion,
        duracion: duracion ?? this.duracion,
        volumen: volumen ?? this.volumen,
      );
}

class AudioNotifier extends Notifier<AudioState> {
  late final AudioPlayer _player;
  final List<StreamSubscription> _subs = [];

  @override
  AudioState build() {
    _player = AudioPlayer();
    // Cancelar suscripciones y liberar player al destruir el provider
    ref.onDispose(() async {
      for (final sub in _subs) {
        await sub.cancel();
      }
      await _player.dispose();
    });
    _escucharStreams();
    return const AudioState(
      estado: PlayerState.stopped,
      posicion: Duration.zero,
      duracion: Duration.zero,
      volumen: 1.0,
    );
  }

  void _escucharStreams() {
    _subs.add(_player.onDurationChanged.listen(
      (d) => state = state.copyWith(duracion: d),
    ));
    _subs.add(_player.onPositionChanged.listen(
      (p) => state = state.copyWith(posicion: p),
    ));
    _subs.add(_player.onPlayerStateChanged.listen(
      (s) => state = state.copyWith(estado: s),
    ));
  }

  Future<void> reproducir(String url) async {
    await _player.play(UrlSource(url));
  }

  Future<void> pausar() async => _player.pause();
  Future<void> reanudar() async => _player.resume();
  Future<void> detener() async => _player.stop();

  Future<void> buscar(Duration pos) async => _player.seek(pos);

  Future<void> cambiarVolumen(double v) async {
    await _player.setVolume(v);
    state = state.copyWith(volumen: v);
  }
}

final audioProvider =
    NotifierProvider<AudioNotifier, AudioState>(AudioNotifier.new);
```

> **Mini-ejercicio (6 min):** En un `ConsumerWidget`, consume `audioProvider`. Muestra un `IconButton` que cambia de icono según el estado: `state.estado == PlayerState.playing ? Icons.pause : Icons.play_arrow`. Al presionar llama `ref.read(audioProvider.notifier).pausar()` o `reanudar()` según corresponda.

---

### Patrón 4: `VideoPlayerController` — init + AspectRatio + listener

```dart
import 'package:video_player/video_player.dart';

late VideoPlayerController _controller;

@override
void initState() {
  super.initState();
  _controller = VideoPlayerController.networkUrl(
    Uri.parse('https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4'),
  );
  // Añadir listener ANTES de initialize para capturar cambios de estado
  _controller.addListener(() => setState(() {}));
  _controller.initialize().then((_) => setState(() {}));
}

@override
void dispose() {
  _controller.dispose();
  super.dispose();
}

@override
Widget build(BuildContext context) {
  if (!_controller.value.isInitialized) {
    return const Center(child: CircularProgressIndicator());
  }
  return Column(
    children: [
      AspectRatio(
        aspectRatio: _controller.value.aspectRatio,
        child: VideoPlayer(_controller),
      ),
      VideoProgressIndicator(_controller, allowScrubbing: true),
      Row(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          IconButton(
            icon: Icon(_controller.value.isPlaying
                ? Icons.pause
                : Icons.play_arrow),
            onPressed: () => _controller.value.isPlaying
                ? _controller.pause()
                : _controller.play(),
          ),
          IconButton(
            icon: const Icon(Icons.replay_10),
            onPressed: () => _controller.seekTo(
              _controller.value.position - const Duration(seconds: 10),
            ),
          ),
        ],
      ),
    ],
  );
}
```

> **Mini-ejercicio (5 min):** Debajo del `VideoPlayer`, añade `VideoProgressIndicator(_controller, allowScrubbing: true)`. Verifica que puedes arrastrar la barra para saltar en el video. Imprime `_controller.value.position` al presionar un botón "Posición actual".

---

### Patrón 5: `ChewieController` + `Chewie` — controles completos

```dart
import 'package:chewie/chewie.dart';

late ChewieController _chewieController;

// Llamar después de que _controller.initialize() complete
void _inicializarChewie() {
  _chewieController = ChewieController(
    videoPlayerController: _controller,
    aspectRatio: 16 / 9,
    autoPlay: false,
    looping: false,
    allowFullScreen: true,
    allowMuting: true,
    showControls: true,
    autoInitialize: true,
    // Mostrar indicador mientras carga
    placeholder: const Center(child: CircularProgressIndicator()),
  );
}

// Widget en build()
Chewie(controller: _chewieController)

// IMPORTANTE: dispose en orden correcto
@override
void dispose() {
  _chewieController.dispose();  // primero chewie
  _controller.dispose();         // luego video_player
  super.dispose();
}
```

> **Mini-ejercicio (5 min):** Configura `looping: true` en el `ChewieController`. Añade un `IconButton` fuera del widget `Chewie` que alterne `looping` llamando `_chewieController.setLooping(!_chewieController.isLooping)`. Verifica que el video repite al terminar.

---

### Patrón 6: Lista de reproducción con Riverpod — `VideoNotifier`

```dart
class VideoState {
  final int indiceActual;
  final List<String> playlist;
  final bool inicializado;

  const VideoState({
    required this.indiceActual,
    required this.playlist,
    required this.inicializado,
  });

  // URL del video activo
  String get urlActual => playlist[indiceActual];

  // Para mostrar "2 / 3"
  String get progresoPista =>
      '${indiceActual + 1} / ${playlist.length}';

  VideoState copyWith({
    int? indiceActual,
    List<String>? playlist,
    bool? inicializado,
  }) =>
      VideoState(
        indiceActual: indiceActual ?? this.indiceActual,
        playlist: playlist ?? this.playlist,
        inicializado: inicializado ?? this.inicializado,
      );
}

class VideoNotifier extends Notifier<VideoState> {
  @override
  VideoState build() => const VideoState(
        indiceActual: 0,
        playlist: [
          'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4',
          'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/ElephantsDream.mp4',
        ],
        inicializado: false,
      );

  void siguiente() {
    final siguiente = (state.indiceActual + 1) % state.playlist.length;
    state = state.copyWith(indiceActual: siguiente, inicializado: false);
  }

  void anterior() {
    final anterior = (state.indiceActual - 1 + state.playlist.length) %
        state.playlist.length;
    state = state.copyWith(indiceActual: anterior, inicializado: false);
  }

  void irA(int i) =>
      state = state.copyWith(indiceActual: i, inicializado: false);

  void marcarInicializado() =>
      state = state.copyWith(inicializado: true);
}

final videoProvider =
    NotifierProvider<VideoNotifier, VideoState>(VideoNotifier.new);
```

> **Mini-ejercicio (5 min):** En un `ConsumerWidget`, consume `videoProvider` y muestra `Text(state.progresoPista)` (p. ej. "1 / 2"). Añade botones "Anterior" y "Siguiente" que llamen `ref.read(videoProvider.notifier).anterior()` y `.siguiente()`. Verifica que el texto cambia al presionar.

---

## Crea el proyecto

```bash
flutter create modulo17_media
cd modulo17_media
```

### `pubspec.yaml`

```yaml
name: modulo17_media
description: "Demo completo de audio y video con Riverpod"
publish_to: none
version: 1.0.0+1

environment:
  sdk: ^3.7.0

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^3.3.0
  audioplayers:     ^6.1.0    # audio desde URL o archivo local
  video_player:     ^2.9.2    # video desde URL o archivo local
  chewie:           ^1.8.5    # controles visuales sobre video_player

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^5.0.0

flutter:
  uses-material-design: true
```

```bash
flutter pub get
```

### Configuración nativa

#### Android — `android/app/src/main/AndroidManifest.xml`

Añadir dentro de `<manifest>`:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

Para reproducción en segundo plano, añadir dentro de `<application>`:

```xml
<service
    android:name="xyz.luan.audioplayers.services.AudioService"
    android:exported="false"
    android:foregroundServiceType="mediaPlayback"/>
```

#### iOS — `ios/Runner/Info.plist`

Añadir dentro de `<dict>`:

```xml
<key>UIBackgroundModes</key>
<array>
    <string>audio</string>
</array>
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

---

## Selector de pasos

Cambia este valor para compilar solo el paso que quieres probar:

```dart
const int paso = 1;
```

En `main.dart` usa un `switch` o `if/else` sobre `paso` para decidir qué pantalla mostrar en `home`. Así no necesitas comentar y descomentar código al avanzar.

---

## Paso 1 — `AudioPlayer` básico (sin Riverpod)

```dart
// lib/main.dart
import 'package:audioplayers/audioplayers.dart';
import 'package:flutter/material.dart';

void main() => runApp(const MaterialApp(
      debugShowCheckedModeBanner: false,
      home: PasoUno(),
    ));

const String _urlAudio =
    'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3';

class PasoUno extends StatefulWidget {
  const PasoUno({super.key});
  @override
  State<PasoUno> createState() => _PasoUnoState();
}

class _PasoUnoState extends State<PasoUno> {
  final AudioPlayer _player = AudioPlayer();
  PlayerState _estado = PlayerState.stopped;
  bool _cargando = false;

  @override
  void initState() {
    super.initState();
    // Escuchar cambios de estado para actualizar la UI
    _player.onPlayerStateChanged.listen((s) {
      setState(() {
        _estado = s;
        _cargando = false;
      });
    });
  }

  @override
  void dispose() {
    _player.dispose();
    super.dispose();
  }

  Future<void> _reproducir() async {
    setState(() => _cargando = true);
    await _player.play(UrlSource(_urlAudio));
  }

  Future<void> _pausar() async => _player.pause();
  Future<void> _detener() async => _player.stop();

  @override
  Widget build(BuildContext context) {
    final reproduciendo = _estado == PlayerState.playing;
    final detenido = _estado == PlayerState.stopped;

    return Scaffold(
      appBar: AppBar(title: const Text('Paso 1 — AudioPlayer básico')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Indicador de carga
            if (_cargando)
              const CircularProgressIndicator()
            else
              Icon(
                reproduciendo ? Icons.music_note : Icons.music_off,
                size: 64,
                color: reproduciendo ? Colors.green : Colors.grey,
              ),
            const SizedBox(height: 24),
            // Texto de estado actual
            Text(
              'Estado: ${_estado.name}',
              style: const TextStyle(fontSize: 18),
            ),
            const SizedBox(height: 32),
            // Botones de control
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                ElevatedButton.icon(
                  onPressed: detenido || _estado == PlayerState.completed
                      ? _reproducir
                      : null,
                  icon: const Icon(Icons.play_arrow),
                  label: const Text('Play'),
                ),
                const SizedBox(width: 12),
                ElevatedButton.icon(
                  onPressed: reproduciendo ? _pausar : null,
                  icon: const Icon(Icons.pause),
                  label: const Text('Pausa'),
                ),
                const SizedBox(width: 12),
                ElevatedButton.icon(
                  onPressed: !detenido ? _detener : null,
                  icon: const Icon(Icons.stop),
                  label: const Text('Stop'),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

> **Prueba esto:** Presiona Play y observa el ícono cambiar a `music_note` verde. El texto debe decir "Estado: playing". Presiona Pausa — debe decir "Estado: paused". Presiona Stop — debe volver a "Estado: stopped".

Salida esperada en consola con `debugPrint(_player.state.toString())`:
```
PlayerState.playing
PlayerState.paused
PlayerState.stopped
```

---

## Paso 2 — Slider de progreso y formato `mm:ss`

```dart
// lib/main.dart
import 'dart:async';
import 'package:audioplayers/audioplayers.dart';
import 'package:flutter/material.dart';

void main() => runApp(const MaterialApp(
      debugShowCheckedModeBanner: false,
      home: PasoDos(),
    ));

const String _urlAudio =
    'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3';

class PasoDos extends StatefulWidget {
  const PasoDos({super.key});
  @override
  State<PasoDos> createState() => _PasosDosState();
}

class _PasosDosState extends State<PasoDos> {
  final AudioPlayer _player = AudioPlayer();
  PlayerState _estado = PlayerState.stopped;
  Duration _posicion = Duration.zero;
  Duration _duracion = Duration.zero;

  // Guardamos las suscripciones para cancelarlas en dispose
  final List<StreamSubscription> _subs = [];

  @override
  void initState() {
    super.initState();
    _subs.add(_player.onPlayerStateChanged.listen(
      (s) => setState(() => _estado = s),
    ));
    _subs.add(_player.onPositionChanged.listen(
      (p) => setState(() => _posicion = p),
    ));
    _subs.add(_player.onDurationChanged.listen(
      (d) => setState(() => _duracion = d),
    ));
  }

  @override
  void dispose() {
    for (final sub in _subs) {
      sub.cancel();
    }
    _player.dispose();
    super.dispose();
  }

  // Formato mm:ss
  String _formato(Duration d) {
    final m = d.inMinutes.toString().padLeft(2, '0');
    final s = (d.inSeconds % 60).toString().padLeft(2, '0');
    return '$m:$s';
  }

  double get _progreso => _duracion.inSeconds > 0
      ? _posicion.inSeconds / _duracion.inSeconds
      : 0.0;

  @override
  Widget build(BuildContext context) {
    final reproduciendo = _estado == PlayerState.playing;

    return Scaffold(
      appBar: AppBar(title: const Text('Paso 2 — Progreso y tiempo')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Tiempo actual / total
            Text(
              '${_formato(_posicion)} / ${_formato(_duracion)}',
              style: const TextStyle(fontSize: 22, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 16),
            // Barra de progreso deslizable
            Slider(
              value: _progreso.clamp(0.0, 1.0),
              onChanged: _duracion.inSeconds > 0
                  ? (v) {
                      final nuevaPos =
                          Duration(seconds: (v * _duracion.inSeconds).round());
                      _player.seek(nuevaPos);
                    }
                  : null,
            ),
            // Barra de progreso secundaria (visual)
            LinearProgressIndicator(value: _progreso.clamp(0.0, 1.0)),
            const SizedBox(height: 32),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                ElevatedButton.icon(
                  onPressed: () =>
                      _player.play(UrlSource(_urlAudio)),
                  icon: const Icon(Icons.play_arrow),
                  label: const Text('Play'),
                ),
                const SizedBox(width: 12),
                ElevatedButton.icon(
                  onPressed:
                      reproduciendo ? () => _player.pause() : null,
                  icon: const Icon(Icons.pause),
                  label: const Text('Pausa'),
                ),
                const SizedBox(width: 12),
                ElevatedButton.icon(
                  onPressed: () => _player.stop(),
                  icon: const Icon(Icons.stop),
                  label: const Text('Stop'),
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

> **Prueba esto:** Inicia Play y observa el texto `mm:ss / mm:ss` actualizarse cada segundo. Arrastra el `Slider` hacia la mitad y verifica que la posición salta al punto correspondiente. El `LinearProgressIndicator` debe avanzar de forma continua.

Salida esperada (en texto): `00:15 / 03:42` mientras el audio lleva 15 segundos de 3 minutos 42 segundos.

---

## Paso 3 — Riverpod: `AudioState` + `AudioNotifier` + `ConsumerWidget`

```dart
// lib/main.dart
import 'dart:async';
import 'package:audioplayers/audioplayers.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() => runApp(
      const ProviderScope(
        child: MaterialApp(
          debugShowCheckedModeBanner: false,
          home: PasoTres(),
        ),
      ),
    );

// --------------- MODELO ---------------
class AudioState {
  final PlayerState estado;
  final Duration posicion;
  final Duration duracion;
  final double volumen;

  const AudioState({
    required this.estado,
    required this.posicion,
    required this.duracion,
    required this.volumen,
  });

  double get progreso =>
      duracion.inSeconds > 0 ? posicion.inSeconds / duracion.inSeconds : 0.0;

  String _fmt(Duration d) {
    final m = d.inMinutes.toString().padLeft(2, '0');
    final s = (d.inSeconds % 60).toString().padLeft(2, '0');
    return '$m:$s';
  }

  String get posicionFmt => _fmt(posicion);
  String get duracionFmt => _fmt(duracion);

  AudioState copyWith({
    PlayerState? estado,
    Duration? posicion,
    Duration? duracion,
    double? volumen,
  }) =>
      AudioState(
        estado: estado ?? this.estado,
        posicion: posicion ?? this.posicion,
        duracion: duracion ?? this.duracion,
        volumen: volumen ?? this.volumen,
      );
}

// --------------- NOTIFIER ---------------
class AudioNotifier extends Notifier<AudioState> {
  late final AudioPlayer _player;
  final List<StreamSubscription> _subs = [];

  static const String _urlDemo =
      'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3';

  @override
  AudioState build() {
    _player = AudioPlayer();
    ref.onDispose(() async {
      for (final sub in _subs) {
        await sub.cancel();
      }
      await _player.dispose();
    });
    _subs.add(_player.onPlayerStateChanged.listen(
      (s) => state = state.copyWith(estado: s),
    ));
    _subs.add(_player.onPositionChanged.listen(
      (p) => state = state.copyWith(posicion: p),
    ));
    _subs.add(_player.onDurationChanged.listen(
      (d) => state = state.copyWith(duracion: d),
    ));
    return const AudioState(
      estado: PlayerState.stopped,
      posicion: Duration.zero,
      duracion: Duration.zero,
      volumen: 1.0,
    );
  }

  Future<void> reproducir() async =>
      _player.play(UrlSource(_urlDemo));
  Future<void> pausar() async => _player.pause();
  Future<void> reanudar() async => _player.resume();
  Future<void> detener() async => _player.stop();
  Future<void> buscar(Duration pos) async => _player.seek(pos);
  Future<void> cambiarVolumen(double v) async {
    await _player.setVolume(v);
    state = state.copyWith(volumen: v);
  }
}

final audioProvider =
    NotifierProvider<AudioNotifier, AudioState>(AudioNotifier.new);

// --------------- PANTALLA ---------------
class PasoTres extends ConsumerWidget {
  const PasoTres({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final audio = ref.watch(audioProvider);
    final notifier = ref.read(audioProvider.notifier);
    final reproduciendo = audio.estado == PlayerState.playing;

    return Scaffold(
      appBar: AppBar(title: const Text('Paso 3 — Riverpod AudioNotifier')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Tiempo y progreso
            Text(
              '${audio.posicionFmt} / ${audio.duracionFmt}',
              style: const TextStyle(fontSize: 22, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 12),
            Slider(
              value: audio.progreso.clamp(0.0, 1.0),
              onChanged: audio.duracion.inSeconds > 0
                  ? (v) {
                      final pos = Duration(
                          seconds: (v * audio.duracion.inSeconds).round());
                      notifier.buscar(pos);
                    }
                  : null,
            ),
            const SizedBox(height: 24),
            // Controles play / pause / stop
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                IconButton(
                  iconSize: 48,
                  icon: const Icon(Icons.stop),
                  onPressed: audio.estado != PlayerState.stopped
                      ? notifier.detener
                      : null,
                ),
                IconButton(
                  iconSize: 64,
                  icon: Icon(reproduciendo ? Icons.pause : Icons.play_arrow),
                  onPressed: reproduciendo
                      ? notifier.pausar
                      : audio.estado == PlayerState.paused
                          ? notifier.reanudar
                          : notifier.reproducir,
                ),
              ],
            ),
            const SizedBox(height: 32),
            // Control de volumen
            Row(
              children: [
                Icon(
                  audio.volumen == 0 ? Icons.volume_off : Icons.volume_up,
                ),
                Expanded(
                  child: Slider(
                    value: audio.volumen,
                    onChanged: notifier.cambiarVolumen,
                  ),
                ),
                Text('${(audio.volumen * 100).round()}%'),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

> **Prueba esto:** Presiona play y arrastra el slider de volumen a 0. El ícono debe cambiar a `Icons.volume_off`. Sube el volumen y verifica que el audio vuelve. El slider de progreso debe reflejar la posición en tiempo real sin tocar `setState` en la UI.

Salida esperada: la UI se reconstruye automáticamente con cada cambio de `state` en el notifier, sin un solo `setState` en la pantalla.

---

## Paso 4 — `VideoPlayerController` + `Chewie`

```dart
// lib/main.dart
import 'package:chewie/chewie.dart';
import 'package:flutter/material.dart';
import 'package:video_player/video_player.dart';

void main() => runApp(const MaterialApp(
      debugShowCheckedModeBanner: false,
      home: PasoCuatro(),
    ));

const String _urlVideo =
    'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4';

class PasoCuatro extends StatefulWidget {
  const PasoCuatro({super.key});
  @override
  State<PasoCuatro> createState() => _PasoCuatroState();
}

class _PasoCuatroState extends State<PasoCuatro> {
  late VideoPlayerController _videoController;
  ChewieController? _chewieController;

  @override
  void initState() {
    super.initState();
    _inicializar();
  }

  Future<void> _inicializar() async {
    _videoController = VideoPlayerController.networkUrl(
      Uri.parse(_urlVideo),
    );
    // Listener para reconstruir UI cuando cambia el estado
    _videoController.addListener(() => setState(() {}));

    await _videoController.initialize();

    _chewieController = ChewieController(
      videoPlayerController: _videoController,
      aspectRatio: 16 / 9,
      autoPlay: false,
      looping: false,
      allowFullScreen: true,
      allowMuting: true,
      showControls: true,
      placeholder: const Center(child: CircularProgressIndicator()),
    );
    // Reconstruir para mostrar el video ya inicializado
    setState(() {});
  }

  @override
  void dispose() {
    // ORDEN IMPORTANTE: primero chewie, luego video_player
    _chewieController?.dispose();
    _videoController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final listo = _chewieController != null &&
        _videoController.value.isInitialized;

    return Scaffold(
      appBar: AppBar(title: const Text('Paso 4 — Video con Chewie')),
      body: Center(
        child: listo
            ? AspectRatio(
                aspectRatio: 16 / 9,
                child: Chewie(controller: _chewieController!),
              )
            : const Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  CircularProgressIndicator(),
                  SizedBox(height: 16),
                  Text('Cargando video...'),
                ],
              ),
      ),
    );
  }
}
```

> **Prueba esto:** Al iniciar, aparece "Cargando video..." durante la inicialización. Una vez listo, aparece el reproductor con controles: botón play/pausa, barra de progreso, volumen y botón de pantalla completa. Gira el dispositivo (o el simulador) y verifica que los controles siguen funcionando en landscape.

Salida esperada: reproductor con controles de `chewie` totalmente funcionales, pantalla completa disponible al presionar el ícono de expansión en la esquina inferior derecha.

---

## Paso 5 — App completa con `NavigationBar` (Tab Audio + Tab Video)

```dart
// lib/main.dart
import 'dart:async';
import 'package:audioplayers/audioplayers.dart';
import 'package:chewie/chewie.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:video_player/video_player.dart';

void main() => runApp(
      const ProviderScope(
        child: MaterialApp(
          debugShowCheckedModeBanner: false,
          home: AppPrincipal(),
        ),
      ),
    );

// ================================================================
// MODELOS
// ================================================================

class AudioState {
  final PlayerState estado;
  final Duration posicion;
  final Duration duracion;
  final double volumen;

  const AudioState({
    required this.estado,
    required this.posicion,
    required this.duracion,
    required this.volumen,
  });

  double get progreso =>
      duracion.inSeconds > 0 ? posicion.inSeconds / duracion.inSeconds : 0.0;

  String _fmt(Duration d) {
    final m = d.inMinutes.toString().padLeft(2, '0');
    final s = (d.inSeconds % 60).toString().padLeft(2, '0');
    return '$m:$s';
  }

  String get posicionFmt => _fmt(posicion);
  String get duracionFmt => _fmt(duracion);

  AudioState copyWith({
    PlayerState? estado,
    Duration? posicion,
    Duration? duracion,
    double? volumen,
  }) =>
      AudioState(
        estado: estado ?? this.estado,
        posicion: posicion ?? this.posicion,
        duracion: duracion ?? this.duracion,
        volumen: volumen ?? this.volumen,
      );
}

class VideoState {
  final int indiceActual;
  final List<String> playlist;
  final bool inicializado;

  const VideoState({
    required this.indiceActual,
    required this.playlist,
    required this.inicializado,
  });

  String get urlActual => playlist[indiceActual];
  String get progresoPista => '${indiceActual + 1} / ${playlist.length}';

  VideoState copyWith({
    int? indiceActual,
    List<String>? playlist,
    bool? inicializado,
  }) =>
      VideoState(
        indiceActual: indiceActual ?? this.indiceActual,
        playlist: playlist ?? this.playlist,
        inicializado: inicializado ?? this.inicializado,
      );
}

// ================================================================
// PROVIDERS
// ================================================================

class AudioNotifier extends Notifier<AudioState> {
  late final AudioPlayer _player;
  final List<StreamSubscription> _subs = [];

  @override
  AudioState build() {
    _player = AudioPlayer();
    ref.onDispose(() async {
      for (final sub in _subs) {
        await sub.cancel();
      }
      await _player.dispose();
    });
    _subs.add(_player.onPlayerStateChanged.listen(
      (s) => state = state.copyWith(estado: s),
    ));
    _subs.add(_player.onPositionChanged.listen(
      (p) => state = state.copyWith(posicion: p),
    ));
    _subs.add(_player.onDurationChanged.listen(
      (d) => state = state.copyWith(duracion: d),
    ));
    return const AudioState(
      estado: PlayerState.stopped,
      posicion: Duration.zero,
      duracion: Duration.zero,
      volumen: 1.0,
    );
  }

  Future<void> reproducir(String url) async =>
      _player.play(UrlSource(url));
  Future<void> pausar() async => _player.pause();
  Future<void> reanudar() async => _player.resume();
  Future<void> detener() async => _player.stop();
  Future<void> buscar(Duration pos) async => _player.seek(pos);
  Future<void> cambiarVolumen(double v) async {
    await _player.setVolume(v);
    state = state.copyWith(volumen: v);
  }
}

final audioProvider =
    NotifierProvider<AudioNotifier, AudioState>(AudioNotifier.new);

class VideoNotifier extends Notifier<VideoState> {
  @override
  VideoState build() => const VideoState(
        indiceActual: 0,
        playlist: [
          'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4',
          'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/ElephantsDream.mp4',
        ],
        inicializado: false,
      );

  void siguiente() {
    final sig = (state.indiceActual + 1) % state.playlist.length;
    state = state.copyWith(indiceActual: sig, inicializado: false);
  }

  void anterior() {
    final ant = (state.indiceActual - 1 + state.playlist.length) %
        state.playlist.length;
    state = state.copyWith(indiceActual: ant, inicializado: false);
  }

  void irA(int i) =>
      state = state.copyWith(indiceActual: i, inicializado: false);
}

final videoProvider =
    NotifierProvider<VideoNotifier, VideoState>(VideoNotifier.new);

// ================================================================
// APP PRINCIPAL CON NAVEGACION
// ================================================================

class AppPrincipal extends StatefulWidget {
  const AppPrincipal({super.key});
  @override
  State<AppPrincipal> createState() => _AppPrincipalState();
}

class _AppPrincipalState extends State<AppPrincipal> {
  int _tabActual = 0;

  static const List<Widget> _pantallas = [
    PantallaAudio(),
    PantallaVideo(),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: _pantallas[_tabActual],
      bottomNavigationBar: NavigationBar(
        selectedIndex: _tabActual,
        onDestinationSelected: (i) => setState(() => _tabActual = i),
        destinations: const [
          NavigationDestination(
            icon: Icon(Icons.music_note),
            label: 'Audio',
          ),
          NavigationDestination(
            icon: Icon(Icons.videocam),
            label: 'Video',
          ),
        ],
      ),
    );
  }
}

// ================================================================
// PANTALLA AUDIO
// ================================================================

const String _urlAudio =
    'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3';

class PantallaAudio extends ConsumerWidget {
  const PantallaAudio({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final audio = ref.watch(audioProvider);
    final notifier = ref.read(audioProvider.notifier);
    final reproduciendo = audio.estado == PlayerState.playing;
    final pausado = audio.estado == PlayerState.paused;

    return Scaffold(
      appBar: AppBar(title: const Text('Reproductor de Audio')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            // Ícono animado
            Icon(
              reproduciendo ? Icons.music_note : Icons.music_off,
              size: 80,
              color: reproduciendo ? Colors.deepPurple : Colors.grey,
            ),
            const SizedBox(height: 24),
            // Tiempo
            Text(
              '${audio.posicionFmt} / ${audio.duracionFmt}',
              style: const TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 12),
            // Slider de progreso
            Slider(
              value: audio.progreso.clamp(0.0, 1.0),
              onChanged: audio.duracion.inSeconds > 0
                  ? (v) {
                      final pos = Duration(
                          seconds: (v * audio.duracion.inSeconds).round());
                      notifier.buscar(pos);
                    }
                  : null,
            ),
            const SizedBox(height: 24),
            // Controles principales
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                // Retroceder 10 segundos
                IconButton(
                  iconSize: 36,
                  icon: const Icon(Icons.replay_10),
                  onPressed: reproduciendo || pausado
                      ? () => notifier.buscar(
                            audio.posicion - const Duration(seconds: 10),
                          )
                      : null,
                ),
                // Play / Pausa central
                FloatingActionButton(
                  onPressed: reproduciendo
                      ? notifier.pausar
                      : pausado
                          ? notifier.reanudar
                          : () => notifier.reproducir(_urlAudio),
                  child: Icon(
                    reproduciendo ? Icons.pause : Icons.play_arrow,
                  ),
                ),
                // Detener
                IconButton(
                  iconSize: 36,
                  icon: const Icon(Icons.stop),
                  onPressed: !reproduciendo && !pausado
                      ? null
                      : notifier.detener,
                ),
              ],
            ),
            const SizedBox(height: 32),
            // Control de volumen
            Row(
              children: [
                GestureDetector(
                  // Silenciar / reactivar al tocar el ícono
                  onTap: () => notifier.cambiarVolumen(
                    audio.volumen > 0 ? 0.0 : 1.0,
                  ),
                  child: Icon(
                    audio.volumen == 0 ? Icons.volume_off : Icons.volume_up,
                    color: audio.volumen == 0 ? Colors.red : null,
                  ),
                ),
                Expanded(
                  child: Slider(
                    value: audio.volumen,
                    onChanged: notifier.cambiarVolumen,
                  ),
                ),
                Text('${(audio.volumen * 100).round()}%'),
              ],
            ),
          ],
        ),
      ),
    );
  }
}

// ================================================================
// PANTALLA VIDEO
// ================================================================

const String _urlVideo1 =
    'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4';

class PantallaVideo extends ConsumerStatefulWidget {
  const PantallaVideo({super.key});
  @override
  ConsumerState<PantallaVideo> createState() => _PantallaVideoState();
}

class _PantallaVideoState extends ConsumerState<PantallaVideo> {
  VideoPlayerController? _videoController;
  ChewieController? _chewieController;
  int _indiceAnterior = -1;

  @override
  void dispose() {
    _chewieController?.dispose();
    _videoController?.dispose();
    super.dispose();
  }

  Future<void> _cargarVideo(String url) async {
    // Liberar reproductores anteriores antes de crear nuevos
    _chewieController?.dispose();
    await _videoController?.dispose();

    final ctrl = VideoPlayerController.networkUrl(Uri.parse(url));
    ctrl.addListener(() => setState(() {}));
    await ctrl.initialize();

    if (!mounted) return;

    setState(() {
      _videoController = ctrl;
      _chewieController = ChewieController(
        videoPlayerController: ctrl,
        aspectRatio: 16 / 9,
        autoPlay: true,
        looping: false,
        allowFullScreen: true,
        showControls: true,
        placeholder: const Center(child: CircularProgressIndicator()),
      );
    });
  }

  @override
  Widget build(BuildContext context) {
    final videoState = ref.watch(videoProvider);
    final notifier = ref.read(videoProvider.notifier);

    // Cargar nuevo video cuando cambia el índice
    if (_indiceAnterior != videoState.indiceActual) {
      _indiceAnterior = videoState.indiceActual;
      WidgetsBinding.instance.addPostFrameCallback((_) {
        _cargarVideo(videoState.urlActual);
      });
    }

    final listo =
        _chewieController != null && _videoController?.value.isInitialized == true;

    return Scaffold(
      appBar: AppBar(
        title: Text('Video — ${videoState.progresoPista}'),
      ),
      body: Column(
        children: [
          // Reproductor
          AspectRatio(
            aspectRatio: 16 / 9,
            child: listo
                ? Chewie(controller: _chewieController!)
                : const Center(child: CircularProgressIndicator()),
          ),
          const SizedBox(height: 16),
          // Controles de playlist
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 24),
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                ElevatedButton.icon(
                  onPressed: notifier.anterior,
                  icon: const Icon(Icons.skip_previous),
                  label: const Text('Anterior'),
                ),
                Text(
                  videoState.progresoPista,
                  style: const TextStyle(
                    fontSize: 18,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                ElevatedButton.icon(
                  onPressed: notifier.siguiente,
                  icon: const Icon(Icons.skip_next),
                  label: const Text('Siguiente'),
                ),
              ],
            ),
          ),
          const SizedBox(height: 16),
          // Lista de videos
          Expanded(
            child: ListView.builder(
              itemCount: videoState.playlist.length,
              itemBuilder: (context, i) {
                final esActual = i == videoState.indiceActual;
                return ListTile(
                  leading: Icon(
                    esActual ? Icons.play_circle : Icons.video_file,
                    color: esActual ? Colors.deepPurple : null,
                  ),
                  title: Text('Video ${i + 1}'),
                  subtitle: Text(
                    videoState.playlist[i],
                    maxLines: 1,
                    overflow: TextOverflow.ellipsis,
                  ),
                  selected: esActual,
                  onTap: () => notifier.irA(i),
                );
              },
            ),
          ),
        ],
      ),
    );
  }
}
```

> **Prueba esto:** Navega entre las pestañas "Audio" y "Video". En Video, presiona "Siguiente" para pasar al segundo video — el título del AppBar debe cambiar a "Video — 2 / 2". Selecciona un video de la lista para cargarlo directamente. El audio de la pestaña Audio sigue su estado aunque cambies de pestaña.

Salida esperada: Tab Audio muestra reproductor con slider funcional. Tab Video muestra `Chewie` con controles y la lista de reproducción debajo. Cambiar de video en la lista carga y reproduce automáticamente el nuevo video.

---

## main.dart completo — referencia

El `main.dart` del Paso 5 es la versión completa de referencia. Incluye todos los modelos, providers y pantallas en un solo archivo para facilitar la clase. En el proyecto final se separan en archivos individuales (ver sección siguiente).

---

## Proyecto final

### Estructura

```
modulo17_media/
├── lib/
│   ├── main.dart
│   ├── models/
│   │   ├── audio_state.dart
│   │   └── video_state.dart
│   ├── providers/
│   │   ├── audio_provider.dart
│   │   └── video_provider.dart
│   └── screens/
│       ├── pantalla_audio.dart
│       └── pantalla_video.dart
├── pubspec.yaml
└── android/
    └── app/src/main/AndroidManifest.xml
```

### Archivos clave

#### `lib/models/audio_state.dart`

```dart
import 'package:audioplayers/audioplayers.dart';

class AudioState {
  final PlayerState estado;
  final Duration posicion;
  final Duration duracion;
  final double volumen;

  const AudioState({
    required this.estado,
    required this.posicion,
    required this.duracion,
    required this.volumen,
  });

  double get progreso =>
      duracion.inSeconds > 0 ? posicion.inSeconds / duracion.inSeconds : 0.0;

  String _fmt(Duration d) {
    final m = d.inMinutes.toString().padLeft(2, '0');
    final s = (d.inSeconds % 60).toString().padLeft(2, '0');
    return '$m:$s';
  }

  String get posicionFmt => _fmt(posicion);
  String get duracionFmt => _fmt(duracion);

  AudioState copyWith({
    PlayerState? estado,
    Duration? posicion,
    Duration? duracion,
    double? volumen,
  }) =>
      AudioState(
        estado: estado ?? this.estado,
        posicion: posicion ?? this.posicion,
        duracion: duracion ?? this.duracion,
        volumen: volumen ?? this.volumen,
      );
}
```

#### `lib/models/video_state.dart`

```dart
class VideoState {
  final int indiceActual;
  final List<String> playlist;
  final bool inicializado;

  const VideoState({
    required this.indiceActual,
    required this.playlist,
    required this.inicializado,
  });

  String get urlActual => playlist[indiceActual];
  String get progresoPista => '${indiceActual + 1} / ${playlist.length}';

  VideoState copyWith({
    int? indiceActual,
    List<String>? playlist,
    bool? inicializado,
  }) =>
      VideoState(
        indiceActual: indiceActual ?? this.indiceActual,
        playlist: playlist ?? this.playlist,
        inicializado: inicializado ?? this.inicializado,
      );
}
```

#### `lib/providers/audio_provider.dart`

```dart
import 'dart:async';
import 'package:audioplayers/audioplayers.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/audio_state.dart';

class AudioNotifier extends Notifier<AudioState> {
  late final AudioPlayer _player;
  final List<StreamSubscription> _subs = [];

  @override
  AudioState build() {
    _player = AudioPlayer();
    ref.onDispose(() async {
      for (final sub in _subs) {
        await sub.cancel();
      }
      await _player.dispose();
    });
    _subs.add(_player.onPlayerStateChanged.listen(
      (s) => state = state.copyWith(estado: s),
    ));
    _subs.add(_player.onPositionChanged.listen(
      (p) => state = state.copyWith(posicion: p),
    ));
    _subs.add(_player.onDurationChanged.listen(
      (d) => state = state.copyWith(duracion: d),
    ));
    return const AudioState(
      estado: PlayerState.stopped,
      posicion: Duration.zero,
      duracion: Duration.zero,
      volumen: 1.0,
    );
  }

  Future<void> reproducir(String url) async =>
      _player.play(UrlSource(url));
  Future<void> pausar() async => _player.pause();
  Future<void> reanudar() async => _player.resume();
  Future<void> detener() async => _player.stop();
  Future<void> buscar(Duration pos) async => _player.seek(pos);
  Future<void> cambiarVolumen(double v) async {
    await _player.setVolume(v);
    state = state.copyWith(volumen: v);
  }
}

final audioProvider =
    NotifierProvider<AudioNotifier, AudioState>(AudioNotifier.new);
```

#### `lib/providers/video_provider.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../models/video_state.dart';

class VideoNotifier extends Notifier<VideoState> {
  @override
  VideoState build() => const VideoState(
        indiceActual: 0,
        playlist: [
          'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4',
          'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/ElephantsDream.mp4',
        ],
        inicializado: false,
      );

  void siguiente() {
    final sig = (state.indiceActual + 1) % state.playlist.length;
    state = state.copyWith(indiceActual: sig, inicializado: false);
  }

  void anterior() {
    final ant =
        (state.indiceActual - 1 + state.playlist.length) % state.playlist.length;
    state = state.copyWith(indiceActual: ant, inicializado: false);
  }

  void irA(int i) =>
      state = state.copyWith(indiceActual: i, inicializado: false);
}

final videoProvider =
    NotifierProvider<VideoNotifier, VideoState>(VideoNotifier.new);
```

#### `lib/screens/pantalla_audio.dart`

```dart
import 'package:audioplayers/audioplayers.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/audio_provider.dart';

const String _urlAudio =
    'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3';

class PantallaAudio extends ConsumerWidget {
  const PantallaAudio({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final audio = ref.watch(audioProvider);
    final notifier = ref.read(audioProvider.notifier);
    final reproduciendo = audio.estado == PlayerState.playing;
    final pausado = audio.estado == PlayerState.paused;

    return Scaffold(
      appBar: AppBar(title: const Text('Reproductor de Audio')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Icon(
              reproduciendo ? Icons.music_note : Icons.music_off,
              size: 80,
              color: reproduciendo ? Colors.deepPurple : Colors.grey,
            ),
            const SizedBox(height: 24),
            Text(
              '${audio.posicionFmt} / ${audio.duracionFmt}',
              style: const TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
            ),
            const SizedBox(height: 12),
            Slider(
              value: audio.progreso.clamp(0.0, 1.0),
              onChanged: audio.duracion.inSeconds > 0
                  ? (v) {
                      final pos = Duration(
                          seconds: (v * audio.duracion.inSeconds).round());
                      notifier.buscar(pos);
                    }
                  : null,
            ),
            const SizedBox(height: 24),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                IconButton(
                  iconSize: 36,
                  icon: const Icon(Icons.replay_10),
                  onPressed: reproduciendo || pausado
                      ? () => notifier.buscar(
                            audio.posicion - const Duration(seconds: 10),
                          )
                      : null,
                ),
                FloatingActionButton(
                  onPressed: reproduciendo
                      ? notifier.pausar
                      : pausado
                          ? notifier.reanudar
                          : () => notifier.reproducir(_urlAudio),
                  child: Icon(
                    reproduciendo ? Icons.pause : Icons.play_arrow,
                  ),
                ),
                IconButton(
                  iconSize: 36,
                  icon: const Icon(Icons.stop),
                  onPressed:
                      !reproduciendo && !pausado ? null : notifier.detener,
                ),
              ],
            ),
            const SizedBox(height: 32),
            Row(
              children: [
                GestureDetector(
                  onTap: () =>
                      notifier.cambiarVolumen(audio.volumen > 0 ? 0.0 : 1.0),
                  child: Icon(
                    audio.volumen == 0 ? Icons.volume_off : Icons.volume_up,
                    color: audio.volumen == 0 ? Colors.red : null,
                  ),
                ),
                Expanded(
                  child: Slider(
                    value: audio.volumen,
                    onChanged: notifier.cambiarVolumen,
                  ),
                ),
                Text('${(audio.volumen * 100).round()}%'),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

#### `lib/screens/pantalla_video.dart`

```dart
import 'package:chewie/chewie.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:video_player/video_player.dart';
import '../providers/video_provider.dart';

class PantallaVideo extends ConsumerStatefulWidget {
  const PantallaVideo({super.key});
  @override
  ConsumerState<PantallaVideo> createState() => _PantallaVideoState();
}

class _PantallaVideoState extends ConsumerState<PantallaVideo> {
  VideoPlayerController? _videoController;
  ChewieController? _chewieController;
  int _indiceAnterior = -1;

  @override
  void dispose() {
    _chewieController?.dispose();
    _videoController?.dispose();
    super.dispose();
  }

  Future<void> _cargarVideo(String url) async {
    _chewieController?.dispose();
    await _videoController?.dispose();

    final ctrl = VideoPlayerController.networkUrl(Uri.parse(url));
    ctrl.addListener(() => setState(() {}));
    await ctrl.initialize();

    if (!mounted) return;

    setState(() {
      _videoController = ctrl;
      _chewieController = ChewieController(
        videoPlayerController: ctrl,
        aspectRatio: 16 / 9,
        autoPlay: true,
        looping: false,
        allowFullScreen: true,
        showControls: true,
        placeholder: const Center(child: CircularProgressIndicator()),
      );
    });
  }

  @override
  Widget build(BuildContext context) {
    final videoState = ref.watch(videoProvider);
    final notifier = ref.read(videoProvider.notifier);

    if (_indiceAnterior != videoState.indiceActual) {
      _indiceAnterior = videoState.indiceActual;
      WidgetsBinding.instance.addPostFrameCallback((_) {
        _cargarVideo(videoState.urlActual);
      });
    }

    final listo = _chewieController != null &&
        _videoController?.value.isInitialized == true;

    return Scaffold(
      appBar: AppBar(
        title: Text('Video — ${videoState.progresoPista}'),
      ),
      body: Column(
        children: [
          AspectRatio(
            aspectRatio: 16 / 9,
            child: listo
                ? Chewie(controller: _chewieController!)
                : const Center(child: CircularProgressIndicator()),
          ),
          const SizedBox(height: 16),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 24),
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                ElevatedButton.icon(
                  onPressed: notifier.anterior,
                  icon: const Icon(Icons.skip_previous),
                  label: const Text('Anterior'),
                ),
                Text(
                  videoState.progresoPista,
                  style: const TextStyle(
                    fontSize: 18,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                ElevatedButton.icon(
                  onPressed: notifier.siguiente,
                  icon: const Icon(Icons.skip_next),
                  label: const Text('Siguiente'),
                ),
              ],
            ),
          ),
          const SizedBox(height: 16),
          Expanded(
            child: ListView.builder(
              itemCount: videoState.playlist.length,
              itemBuilder: (context, i) {
                final esActual = i == videoState.indiceActual;
                return ListTile(
                  leading: Icon(
                    esActual ? Icons.play_circle : Icons.video_file,
                    color: esActual ? Colors.deepPurple : null,
                  ),
                  title: Text('Video ${i + 1}'),
                  subtitle: Text(
                    videoState.playlist[i],
                    maxLines: 1,
                    overflow: TextOverflow.ellipsis,
                  ),
                  selected: esActual,
                  onTap: () => notifier.irA(i),
                );
              },
            ),
          ),
        ],
      ),
    );
  }
}
```

#### `lib/main.dart` (proyecto final)

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'screens/pantalla_audio.dart';
import 'screens/pantalla_video.dart';

void main() => runApp(
      const ProviderScope(
        child: MaterialApp(
          debugShowCheckedModeBanner: false,
          title: 'Módulo 17 — Media',
          home: AppPrincipal(),
        ),
      ),
    );

class AppPrincipal extends StatefulWidget {
  const AppPrincipal({super.key});
  @override
  State<AppPrincipal> createState() => _AppPrincipalState();
}

class _AppPrincipalState extends State<AppPrincipal> {
  int _tabActual = 0;

  static const List<Widget> _pantallas = [
    PantallaAudio(),
    PantallaVideo(),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: _pantallas[_tabActual],
      bottomNavigationBar: NavigationBar(
        selectedIndex: _tabActual,
        onDestinationSelected: (i) => setState(() => _tabActual = i),
        destinations: const [
          NavigationDestination(
            icon: Icon(Icons.music_note),
            label: 'Audio',
          ),
          NavigationDestination(
            icon: Icon(Icons.videocam),
            label: 'Video',
          ),
        ],
      ),
    );
  }
}
```

---

## Guía rápida de imports

```dart
// Audio
import 'package:audioplayers/audioplayers.dart';
// Clases clave: AudioPlayer, UrlSource, AssetSource, FileSource,
//               PlayerState, ReleaseMode

// Video
import 'package:video_player/video_player.dart';
// Clases clave: VideoPlayerController, VideoPlayer, VideoProgressIndicator

// Chewie (controles visuales sobre video_player)
import 'package:chewie/chewie.dart';
// Clases clave: ChewieController, Chewie

// Riverpod
import 'package:flutter_riverpod/flutter_riverpod.dart';
// Clases clave: ProviderScope, NotifierProvider, Notifier,
//               ConsumerWidget, ConsumerStatefulWidget, ConsumerState, WidgetRef

// Dart
import 'dart:async';
// Clases clave: StreamSubscription (para cancelar streams en dispose)
```

---

## Cuándo usar qué

```
Situacion                              | Solucion
---------------------------------------|-----------------------------------------------
Solo audio desde URL                   | AudioPlayer + UrlSource('https://...')
Audio local (asset)                    | AudioPlayer + AssetSource('audio/file.mp3')
Audio en background (Android)          | AudioPlayer + servicio en AndroidManifest.xml
Video desde URL sin controles propios  | VideoPlayerController + VideoPlayer widget
Video con controles completos          | VideoPlayerController + ChewieController + Chewie
Video fullscreen y landscape           | ChewieController(allowFullScreen: true)
Progreso de audio (posicion y tiempo)  | streams onPositionChanged + onDurationChanged
Lista de reproduccion (audio o video)  | AudioNotifier / VideoNotifier con indice en estado
Estado reactivo sin setState en UI     | NotifierProvider<Notifier, State> + ConsumerWidget
Evitar memory leaks en providers       | ref.onDispose() + _player.dispose() / _controller.dispose()
Cancelar streams al destruir widget    | List<StreamSubscription> + sub.cancel() en dispose
Cambiar velocidad de reproduccion      | _controller.setPlaybackSpeed(double speed)
```

---

## Ejercicios propuestos

1. **Playlist de audio:** Extiende `AudioNotifier` con una lista de 3 URLs. Añade los métodos `siguientePista()` y `pistaAnterior()` que carguen la URL correspondiente llamando a `reproducir(url)`. Muestra un `Text('${indiceActual + 1} / 3')` en la pantalla que se actualice al cambiar de pista.

2. **Reproduccion en background:** Verifica que el servicio de `audioplayers` esté configurado en `AndroidManifest.xml` (ver sección Configuración nativa). Añade `await _player.setReleaseMode(ReleaseMode.keep)` antes de reproducir. Prueba que el audio continúa sonando al presionar el botón Home del dispositivo y volver a la app.

3. **Miniatura de video:** Antes de que `VideoPlayerController` termine de inicializarse, muestra una imagen de portada desde URL usando `Image.network('https://...')` como placeholder. Cuando `_controller.value.isInitialized` sea `true`, reemplázala con el widget `Chewie`. Usa un `AnimatedSwitcher` para la transición.

4. **Velocidad de reproduccion:** Añade `setPlaybackSpeed(double speed)` al `VideoNotifier` que llame `_controller.setPlaybackSpeed(speed)`. Crea un `DropdownButton<double>` con opciones `[0.5, 1.0, 1.5, 2.0]` en la `PantallaVideo`. Al seleccionar una opción, llama al notifier. Muestra la velocidad activa junto al texto de progreso de pista.

---

## Resumen

- `AudioPlayer` de `audioplayers` acepta `UrlSource`, `AssetSource` y `FileSource`; los tres streams clave son `onPositionChanged`, `onDurationChanged` y `onPlayerStateChanged`.
- Para evitar memory leaks en streams, guarda cada `StreamSubscription` en una lista y cancélalos en `dispose` o en `ref.onDispose`.
- `AudioNotifier extends Notifier<AudioState>` centraliza el estado del reproductor; la UI solo llama a métodos del notifier y escucha con `ref.watch`.
- `VideoPlayerController.networkUrl(Uri.parse('...'))` reemplaza al deprecado `VideoPlayerController.network()`; siempre añade un `addListener(() => setState((){}))` antes de `initialize()`.
- `ChewieController` envuelve al `VideoPlayerController` y provee controles de UI automáticos (play, pausa, slider, pantalla completa); siempre llama `_chewieController.dispose()` antes de `_videoController.dispose()`.
- `VideoNotifier` gestiona la playlist con un índice y métodos `siguiente()`, `anterior()` e `irA(int)`; la `PantallaVideo` es `ConsumerStatefulWidget` para combinar Riverpod con el ciclo de vida de los controladores de video.
- Las URLs de prueba recomendadas son `soundhelix.com` para audio y el bucket público de Google para video (`gtv-videos-bucket`); no requieren autenticación ni CORS especial.
- El patrón `ConsumerStatefulWidget` + `ConsumerState` es necesario cuando necesitas tanto `ref` de Riverpod como `initState`/`dispose` del ciclo de vida del widget.

> **Siguiente página →** Página 18: Internacionalización (i18n) y localización (l10n).
