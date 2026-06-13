# Tutorial Flutter — Página 17
## Módulo 5 · Multimedia
### Audio y video — construcción incremental paso a paso

---

## Crear el proyecto

```bash
flutter create media_app
cd media_app
```

### `pubspec.yaml`

```yaml
name: media_app
description: "Demo de audio y video streaming"
publish_to: none
version: 1.0.0+1

environment:
  sdk: ^3.7.0

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod:  ^3.3.0
  audioplayers:      ^6.1.0    # audio desde URL o archivo local
  video_player:      ^2.9.2    # video desde URL o archivo local
  chewie:            ^1.8.5    # controles visuales sobre video_player

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

---

## Configuración nativa

### Android — `android/app/src/main/AndroidManifest.xml`

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

### iOS — `ios/Runner/Info.plist`

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

## Estructura final del proyecto

```
media_app/
├── android/app/src/main/AndroidManifest.xml
├── ios/Runner/Info.plist
├── lib/
│   ├── main.dart
│   ├── core/
│   │   └── urls_media.dart            ← URLs de prueba públicas
│   ├── features/
│   │   ├── audio/
│   │   │   ├── servicio_audio.dart    ← wrapper de audioplayers
│   │   │   ├── audio_notifier.dart    ← estado y lógica
│   │   │   └── pantalla_audio.dart    ← UI
│   │   └── video/
│   │       ├── servicio_video.dart    ← wrapper de video_player + chewie
│   │       └── pantalla_video.dart    ← UI
└── pubspec.yaml
```

---

## Paso 1 — App mínima funcionando

**`lib/main.dart`** — reemplaza todo:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const ProviderScope(child: MediaApp()));
}

class MediaApp extends StatelessWidget {
  const MediaApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Media App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepOrange),
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
      appBar: AppBar(title: const Text('Media App')),
      body: const Center(child: Text('Paso 1 — App funcionando ✅')),
    );
  }
}
```

```bash
flutter run
# Debe mostrar "Paso 1 — App funcionando ✅"
```

---

## Paso 2 — URLs de prueba

Antes de tocar audio o video, define las URLs de recursos
públicos y gratuitos para no depender de servicios externos:

**Crea `lib/core/urls_media.dart`:**

```dart
/// URLs públicas y gratuitas para pruebas de audio y video.
/// Todas son de dominio público o Creative Commons.
abstract class UrlsMedia {

  // ── Audio ──────────────────────────────────────────────────────
  static const audioMp3_1 =
      'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3';
  static const audioMp3_2 =
      'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-2.mp3';
  static const audioMp3_3 =
      'https://www.soundhelix.com/examples/mp3/SoundHelix-Song-3.mp3';

  // Radio en vivo (stream continuo sin duración)
  static const radioStream =
      'https://stream.live.vc.bbcmedia.co.uk/bbc_world_service';

  // ── Video ──────────────────────────────────────────────────────
  static const videoMp4_360p =
      'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4';

  static const videoMp4_corto =
      'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/ForBiggerBlazes.mp4';

  // HLS — streaming adaptativo (calidad se ajusta al ancho de banda)
  static const videoHls =
      'https://flutter.github.io/assets-for-api-docs/assets/videos/bee.mp4';

  // ── Playlist de audio ──────────────────────────────────────────
  static const List<({String titulo, String artista, String url})> playlist = [
    (
      titulo:  'Song 1',
      artista: 'SoundHelix',
      url:     audioMp3_1,
    ),
    (
      titulo:  'Song 2',
      artista: 'SoundHelix',
      url:     audioMp3_2,
    ),
    (
      titulo:  'Song 3',
      artista: 'SoundHelix',
      url:     audioMp3_3,
    ),
  ];
}
```

Verifica que el archivo compila sin errores:
```bash
flutter analyze
# No debe mostrar errores
```

---

## Paso 3 — Audio básico: reproducir y pausar

Primero lo más simple: reproducir un MP3 desde URL
con un solo botón play/pause:

**Crea `lib/features/audio/servicio_audio.dart`:**

```dart
import 'package:audioplayers/audioplayers.dart';

/// Wrapper de AudioPlayer que expone la API que necesita la app.
/// Un ServicioAudio = un reproductor = una pista a la vez.
class ServicioAudio {
  final _player = AudioPlayer();

  // Streams que exponen el estado interno del reproductor
  Stream<PlayerState>  get estadoStream    => _player.onPlayerStateChanged;
  Stream<Duration>     get posicionStream  => _player.onPositionChanged;
  Stream<Duration>     get duracionStream  => _player.onDurationChanged;
  Stream<void>         get completoStream  => _player.onPlayerComplete;

  PlayerState get estado   => _player.state;
  bool        get tocando  => estado == PlayerState.playing;

  // Reproducir desde URL
  Future<void> reproducir(String url) async {
    await _player.play(UrlSource(url));
  }

  // Reproducir desde assets locales (para sonidos de la app)
  Future<void> reproducirAsset(String ruta) async {
    await _player.play(AssetSource(ruta));
  }

  Future<void> pausar()   => _player.pause();
  Future<void> reanudar() => _player.resume();
  Future<void> detener()  => _player.stop();

  Future<void> buscar(Duration posicion) => _player.seek(posicion);

  // Volumen: 0.0 a 1.0
  Future<void> setVolumen(double volumen) =>
      _player.setVolume(volumen.clamp(0.0, 1.0));

  // Velocidad: 0.5 a 2.0
  Future<void> setVelocidad(double velocidad) =>
      _player.setPlaybackRate(velocidad.clamp(0.5, 2.0));

  // Liberar recursos — llamar en dispose()
  Future<void> liberar() => _player.dispose();
}
```

Prueba directamente en `PantallaRaiz`:

```dart
// lib/main.dart — reemplaza PantallaRaiz

import 'package:audioplayers/audioplayers.dart';
import 'features/audio/servicio_audio.dart';
import 'core/urls_media.dart';

class PantallaRaiz extends StatefulWidget {
  const PantallaRaiz({super.key});

  @override
  State<PantallaRaiz> createState() => _PantallaRaizState();
}

class _PantallaRaizState extends State<PantallaRaiz> {
  final _audio = ServicioAudio();
  bool  _tocando = false;
  String _estado = 'Detenido';

  @override
  void initState() {
    super.initState();
    _audio.estadoStream.listen((estado) {
      setState(() {
        _tocando = estado == PlayerState.playing;
        _estado  = switch (estado) {
          PlayerState.playing   => '▶️ Reproduciendo',
          PlayerState.paused    => '⏸️ Pausado',
          PlayerState.stopped   => '⏹️ Detenido',
          PlayerState.completed => '✅ Completado',
          _                     => 'Desconocido',
        };
      });
    });
  }

  @override
  void dispose() {
    _audio.liberar();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Paso 3 — Audio básico')),
      body: Padding(
        padding: const EdgeInsets.all(32),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Text(
              _estado,
              textAlign: TextAlign.center,
              style: const TextStyle(fontSize: 22),
            ),
            const SizedBox(height: 32),
            FilledButton.icon(
              onPressed: () async {
                if (_tocando) {
                  await _audio.pausar();
                } else {
                  await _audio.reproducir(UrlsMedia.audioMp3_1);
                }
              },
              icon:  Icon(_tocando ? Icons.pause : Icons.play_arrow),
              label: Text(_tocando ? 'Pausar' : 'Reproducir'),
            ),
            const SizedBox(height: 12),
            OutlinedButton.icon(
              onPressed: () => _audio.detener(),
              icon:  const Icon(Icons.stop),
              label: const Text('Detener'),
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
# Presiona "Reproducir" → empieza el audio desde la URL
# El estado cambia a "▶️ Reproduciendo"
# Presiona "Pausar" → el audio para, estado "⏸️ Pausado"
# Presiona "Reproducir" de nuevo → reanuda desde donde paró
# Presiona "Detener" → regresa al inicio
```

---

## Paso 4 — Audio con barra de progreso y control de posición

```dart
// lib/main.dart — reemplaza PantallaRaiz

class PantallaRaiz extends StatefulWidget {
  const PantallaRaiz({super.key});

  @override
  State<PantallaRaiz> createState() => _PantallaRaizState();
}

class _PantallaRaizState extends State<PantallaRaiz> {
  final _audio   = ServicioAudio();
  bool     _tocando  = false;
  Duration _posicion = Duration.zero;
  Duration _duracion = Duration.zero;

  @override
  void initState() {
    super.initState();

    _audio.estadoStream.listen((s) {
      if (mounted) setState(() => _tocando = s == PlayerState.playing);
    });

    // Actualizar barra de progreso
    _audio.posicionStream.listen((pos) {
      if (mounted) setState(() => _posicion = pos);
    });

    // Obtener la duración total cuando está disponible
    _audio.duracionStream.listen((dur) {
      if (mounted) setState(() => _duracion = dur);
    });
  }

  @override
  void dispose() {
    _audio.liberar();
    super.dispose();
  }

  String _formatear(Duration d) {
    final min = d.inMinutes.remainder(60).toString().padLeft(2, '0');
    final seg = d.inSeconds.remainder(60).toString().padLeft(2, '0');
    return '$min:$seg';
  }

  @override
  Widget build(BuildContext context) {
    final progreso = _duracion.inSeconds > 0
        ? _posicion.inSeconds / _duracion.inSeconds
        : 0.0;

    return Scaffold(
      appBar: AppBar(title: const Text('Paso 4 — Barra de progreso')),
      body: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [

            // Ícono grande
            Icon(
              Icons.music_note,
              size:  80,
              color: Theme.of(context).colorScheme.primary,
            ),
            const SizedBox(height: 16),

            Text('SoundHelix — Song 1',
                style: const TextStyle(
                    fontWeight: FontWeight.bold, fontSize: 18)),
            const SizedBox(height: 4),
            Text('soundhelix.com',
                style: TextStyle(color: Colors.grey.shade600)),

            const SizedBox(height: 24),

            // Barra de progreso interactiva
            Slider(
              value:    progreso.clamp(0.0, 1.0),
              onChanged: (v) {
                final nuevaPos = Duration(
                    seconds: (v * _duracion.inSeconds).toInt());
                _audio.buscar(nuevaPos);
              },
            ),

            // Tiempo
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text(_formatear(_posicion),
                    style: const TextStyle(fontSize: 12)),
                Text(_formatear(_duracion),
                    style: const TextStyle(fontSize: 12)),
              ],
            ),

            const SizedBox(height: 24),

            // Controles
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                // Retroceder 10s
                IconButton(
                  icon:     const Icon(Icons.replay_10),
                  iconSize: 36,
                  onPressed: () {
                    final nueva = _posicion - const Duration(seconds: 10);
                    _audio.buscar(nueva < Duration.zero
                        ? Duration.zero : nueva);
                  },
                ),

                // Play / Pause
                FilledIconButton(
                  onPressed: () async {
                    if (_tocando) {
                      await _audio.pausar();
                    } else {
                      if (_posicion == Duration.zero) {
                        await _audio.reproducir(UrlsMedia.audioMp3_1);
                      } else {
                        await _audio.reanudar();
                      }
                    }
                  },
                  style: FilledIconButton.styleFrom(
                      minimumSize: const Size(56, 56)),
                  child: Icon(
                    _tocando ? Icons.pause : Icons.play_arrow,
                    size: 32,
                  ),
                ),

                // Avanzar 10s
                IconButton(
                  icon:     const Icon(Icons.forward_10),
                  iconSize: 36,
                  onPressed: () {
                    final nueva = _posicion + const Duration(seconds: 10);
                    _audio.buscar(
                        nueva > _duracion ? _duracion : nueva);
                  },
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

```bash
flutter run
# El Slider avanza en tiempo real mientras suena
# Arrastra el slider para saltar a cualquier posición
# Los botones -10s y +10s saltan en la pista
```

---

## Paso 5 — Playlist con Riverpod

Mueve el estado al provider y añade navegación entre pistas:

**Crea `lib/features/audio/audio_notifier.dart`:**

```dart
import 'package:audioplayers/audioplayers.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'servicio_audio.dart';
import '../../core/urls_media.dart';

class AudioState {
  final int      indice;
  final bool     tocando;
  final bool     cargando;
  final Duration posicion;
  final Duration duracion;
  final double   volumen;
  final double   velocidad;

  const AudioState({
    this.indice   = 0,
    this.tocando  = false,
    this.cargando = false,
    this.posicion = Duration.zero,
    this.duracion = Duration.zero,
    this.volumen  = 1.0,
    this.velocidad = 1.0,
  });

  // Pista actual de la playlist
  ({String titulo, String artista, String url}) get pistaActual =>
      UrlsMedia.playlist[indice];

  double get progreso => duracion.inMilliseconds > 0
      ? posicion.inMilliseconds / duracion.inMilliseconds
      : 0.0;

  bool get hayAnterior => indice > 0;
  bool get haySiguiente => indice < UrlsMedia.playlist.length - 1;

  AudioState copyWith({
    int?      indice,
    bool?     tocando,
    bool?     cargando,
    Duration? posicion,
    Duration? duracion,
    double?   volumen,
    double?   velocidad,
  }) => AudioState(
    indice:    indice    ?? this.indice,
    tocando:   tocando   ?? this.tocando,
    cargando:  cargando  ?? this.cargando,
    posicion:  posicion  ?? this.posicion,
    duracion:  duracion  ?? this.duracion,
    volumen:   volumen   ?? this.volumen,
    velocidad: velocidad ?? this.velocidad,
  );
}

class AudioNotifier extends Notifier<AudioState> {
  late final ServicioAudio _svc;

  @override
  AudioState build() {
    _svc = ServicioAudio();

    // Escuchar streams del reproductor y actualizar el estado
    _svc.estadoStream.listen((s) {
      state = state.copyWith(
        tocando:  s == PlayerState.playing,
        cargando: false,
      );
    });

    _svc.posicionStream.listen((pos) {
      state = state.copyWith(posicion: pos);
    });

    _svc.duracionStream.listen((dur) {
      state = state.copyWith(duracion: dur);
    });

    // Al completar, avanzar automáticamente a la siguiente pista
    _svc.completoStream.listen((_) {
      if (state.haySiguiente) {
        siguiente();
      } else {
        state = state.copyWith(tocando: false,
            posicion: Duration.zero);
      }
    });

    // Liberar el reproductor cuando el provider se destruye
    ref.onDispose(() => _svc.liberar());

    return const AudioState();
  }

  Future<void> reproducirPista(int indice) async {
    state = state.copyWith(
      indice:   indice,
      cargando: true,
      posicion: Duration.zero,
      duracion: Duration.zero,
    );
    final url = UrlsMedia.playlist[indice].url;
    await _svc.reproducir(url);
  }

  Future<void> togglePlay() async {
    if (state.tocando) {
      await _svc.pausar();
    } else if (state.posicion > Duration.zero) {
      await _svc.reanudar();
    } else {
      await reproducirPista(state.indice);
    }
  }

  Future<void> anterior() async {
    if (state.hayAnterior) await reproducirPista(state.indice - 1);
  }

  Future<void> siguiente() async {
    if (state.haySiguiente) await reproducirPista(state.indice + 1);
  }

  Future<void> buscar(double progreso) async {
    final pos = Duration(
        milliseconds: (progreso * state.duracion.inMilliseconds).toInt());
    await _svc.buscar(pos);
  }

  Future<void> setVolumen(double v) async {
    await _svc.setVolumen(v);
    state = state.copyWith(volumen: v);
  }

  Future<void> setVelocidad(double v) async {
    await _svc.setVelocidad(v);
    state = state.copyWith(velocidad: v);
  }
}

final audioProvider = NotifierProvider<AudioNotifier, AudioState>(
  AudioNotifier.new,
);
```

Prueba el notifier desde `PantallaRaiz`:

```dart
// lib/main.dart — reemplaza PantallaRaiz

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'features/audio/audio_notifier.dart';
import 'core/urls_media.dart';

class PantallaRaiz extends ConsumerWidget {
  const PantallaRaiz({super.key});

  String _fmt(Duration d) {
    final m = d.inMinutes.remainder(60).toString().padLeft(2, '0');
    final s = d.inSeconds.remainder(60).toString().padLeft(2, '0');
    return '$m:$s';
  }

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(audioProvider);
    final notif = ref.read(audioProvider.notifier);
    final cs    = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(title: const Text('Paso 5 — Playlist')),
      body: Column(
        children: [

          // Pista actual
          Padding(
            padding: const EdgeInsets.all(24),
            child: Column(
              children: [
                Container(
                  width: 160, height: 160,
                  decoration: BoxDecoration(
                    color:        cs.primaryContainer,
                    borderRadius: BorderRadius.circular(16),
                  ),
                  child: Icon(Icons.music_note,
                      size: 80, color: cs.primary),
                ),
                const SizedBox(height: 16),
                Text(state.pistaActual.titulo,
                    style: const TextStyle(
                        fontWeight: FontWeight.bold, fontSize: 20)),
                Text(state.pistaActual.artista,
                    style: TextStyle(color: cs.onSurfaceVariant)),
              ],
            ),
          ),

          // Barra de progreso
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 24),
            child: Column(
              children: [
                Slider(
                  value: state.progreso.clamp(0.0, 1.0),
                  onChanged: (v) => notif.buscar(v),
                ),
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    Text(_fmt(state.posicion),
                        style: const TextStyle(fontSize: 12)),
                    Text(_fmt(state.duracion),
                        style: const TextStyle(fontSize: 12)),
                  ],
                ),
              ],
            ),
          ),

          const SizedBox(height: 8),

          // Controles principales
          Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              IconButton(
                icon:     const Icon(Icons.skip_previous),
                iconSize: 36,
                onPressed: state.hayAnterior ? notif.anterior : null,
              ),
              const SizedBox(width: 8),
              FilledIconButton(
                onPressed: notif.togglePlay,
                style: FilledIconButton.styleFrom(
                    minimumSize: const Size(60, 60)),
                child: state.cargando
                    ? const SizedBox(
                        width: 24, height: 24,
                        child: CircularProgressIndicator(
                            color: Colors.white, strokeWidth: 2))
                    : Icon(
                        state.tocando ? Icons.pause : Icons.play_arrow,
                        size: 32),
              ),
              const SizedBox(width: 8),
              IconButton(
                icon:     const Icon(Icons.skip_next),
                iconSize: 36,
                onPressed: state.haySiguiente ? notif.siguiente : null,
              ),
            ],
          ),

          const SizedBox(height: 16),

          // Lista de pistas
          Expanded(
            child: ListView.builder(
              itemCount: UrlsMedia.playlist.length,
              itemBuilder: (_, i) {
                final pista   = UrlsMedia.playlist[i];
                final activa  = state.indice == i;
                return ListTile(
                  leading: CircleAvatar(
                    backgroundColor: activa
                        ? cs.primaryContainer
                        : cs.surfaceContainerHighest,
                    child: activa && state.tocando
                        ? Icon(Icons.equalizer, color: cs.primary)
                        : Text('${i + 1}',
                            style: TextStyle(
                                color: activa ? cs.primary : null)),
                  ),
                  title: Text(pista.titulo,
                      style: TextStyle(
                          fontWeight: activa
                              ? FontWeight.bold
                              : FontWeight.normal,
                          color: activa ? cs.primary : null)),
                  subtitle: Text(pista.artista),
                  onTap: () => notif.reproducirPista(i),
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

```bash
flutter run
# Toca una pista de la lista → empieza a reproducir
# Toca siguiente/anterior → cambia de pista automáticamente
# Al terminar una pista → pasa a la siguiente sola
# El spinner aparece brevemente mientras carga la nueva URL
```

---

## Paso 6 — Video con `video_player` y `chewie`

**Crea `lib/features/video/servicio_video.dart`:**

```dart
import 'package:chewie/chewie.dart';
import 'package:flutter/material.dart';
import 'package:video_player/video_player.dart';

/// Wrapper que combina VideoPlayerController (motor) con
/// ChewieController (controles visuales).
///
/// Flujo de uso:
///   1. Crear instancia
///   2. await inicializar(url)
///   3. Usar widget() en el árbol de widgets
///   4. llamar liberar() en dispose()
class ServicioVideo {
  VideoPlayerController? _videoCtrl;
  ChewieController?      _chewieCtrl;

  bool get inicializado =>
      _videoCtrl?.value.isInitialized ?? false;

  /// Inicializar el reproductor con una URL.
  /// Devuelve false si la inicialización falla.
  Future<bool> inicializar(String url) async {
    try {
      // Liberar el anterior si existe
      await liberar();

      _videoCtrl = VideoPlayerController.networkUrl(Uri.parse(url));
      await _videoCtrl!.initialize();

      _chewieCtrl = ChewieController(
        videoPlayerController: _videoCtrl!,
        autoPlay:             true,
        looping:              false,
        allowFullScreen:      true,
        allowMuting:          true,
        showControlsOnInitialize: true,
        // Relación de aspecto del video
        aspectRatio: _videoCtrl!.value.aspectRatio,
        placeholder: const Center(
          child: CircularProgressIndicator(),
        ),
        errorBuilder: (context, errorMessage) => Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.error_outline,
                  color: Colors.red, size: 48),
              const SizedBox(height: 8),
              Text(errorMessage,
                  style: const TextStyle(color: Colors.red)),
            ],
          ),
        ),
      );

      return true;
    } catch (e) {
      return false;
    }
  }

  /// Widget que muestra el video con los controles de Chewie.
  Widget widget() {
    if (_chewieCtrl == null) {
      return const Center(child: CircularProgressIndicator());
    }
    return Chewie(controller: _chewieCtrl!);
  }

  Future<void> liberar() async {
    _chewieCtrl?.dispose();
    _chewieCtrl = null;
    await _videoCtrl?.dispose();
    _videoCtrl = null;
  }
}
```

**Crea `lib/features/video/pantalla_video.dart`:**

```dart
import 'package:flutter/material.dart';
import '../../core/urls_media.dart';
import 'servicio_video.dart';

class PantallaVideo extends StatefulWidget {
  const PantallaVideo({super.key});

  @override
  State<PantallaVideo> createState() => _PantallaVideoState();
}

class _PantallaVideoState extends State<PantallaVideo> {
  final _svc    = ServicioVideo();
  bool  _cargando = false;
  bool  _error    = false;

  // URLs disponibles para probar
  static const _videos = [
    (nombre: 'Big Buck Bunny (MP4)',  url: UrlsMedia.videoMp4_360p),
    (nombre: 'For Bigger Blazes',     url: UrlsMedia.videoMp4_corto),
    (nombre: 'Bee (flutter.dev)',     url: UrlsMedia.videoHls),
  ];

  String? _urlActual;

  Future<void> _cargar(String url) async {
    setState(() { _cargando = true; _error = false; });
    final ok = await _svc.inicializar(url);
    if (mounted) {
      setState(() {
        _cargando  = false;
        _error     = !ok;
        _urlActual = ok ? url : null;
      });
    }
  }

  @override
  void dispose() {
    _svc.liberar();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Video'),
        backgroundColor: cs.primaryContainer,
      ),
      body: Column(
        children: [

          // Reproductor de video
          Container(
            width:  double.infinity,
            height: 220,
            color:  Colors.black,
            child: _cargando
                ? const Center(
                    child: CircularProgressIndicator(
                        color: Colors.white))
                : _error
                    ? const Center(
                        child: Column(
                          mainAxisSize: MainAxisSize.min,
                          children: [
                            Icon(Icons.error_outline,
                                color: Colors.white54, size: 48),
                            SizedBox(height: 8),
                            Text('Error al cargar el video',
                                style: TextStyle(
                                    color: Colors.white54)),
                          ],
                        ),
                      )
                    : _urlActual == null
                        ? const Center(
                            child: Text('Selecciona un video',
                                style: TextStyle(
                                    color: Colors.white54)))
                        : _svc.widget(),
          ),

          // Lista de videos disponibles
          Padding(
            padding: const EdgeInsets.fromLTRB(16, 12, 16, 4),
            child: Align(
              alignment: Alignment.centerLeft,
              child: Text('Videos de prueba',
                  style: Theme.of(context)
                      .textTheme
                      .labelLarge
                      ?.copyWith(color: cs.primary)),
            ),
          ),

          Expanded(
            child: ListView.separated(
              itemCount: _videos.length,
              separatorBuilder: (_, __) =>
                  const Divider(height: 1),
              itemBuilder: (_, i) {
                final v        = _videos[i];
                final esActual = _urlActual == v.url;
                return ListTile(
                  leading: Icon(
                    Icons.play_circle_outline,
                    color: esActual ? cs.primary : cs.onSurfaceVariant,
                  ),
                  title: Text(v.nombre,
                      style: TextStyle(
                          fontWeight: esActual
                              ? FontWeight.bold
                              : FontWeight.normal,
                          color: esActual ? cs.primary : null)),
                  trailing: esActual
                      ? Icon(Icons.check_circle,
                          color: cs.primary, size: 20)
                      : null,
                  onTap: () => _cargar(v.url),
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

---

## Paso 7 — App final con NavigationBar

**`lib/features/audio/pantalla_audio.dart`** — crea el archivo
con el reproductor completo (es el mismo código de Paso 5 pero en su archivo):

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'audio_notifier.dart';
import '../../core/urls_media.dart';

class PantallaAudio extends ConsumerWidget {
  const PantallaAudio({super.key});

  String _fmt(Duration d) {
    final m = d.inMinutes.remainder(60).toString().padLeft(2, '0');
    final s = d.inSeconds.remainder(60).toString().padLeft(2, '0');
    return '$m:$s';
  }

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(audioProvider);
    final notif = ref.read(audioProvider.notifier);
    final cs    = Theme.of(context).colorScheme;

    return Scaffold(
      appBar: AppBar(
        title: const Text('Audio'),
        backgroundColor: cs.primaryContainer,
      ),
      body: Column(
        children: [

          // Portada y datos de la pista
          Padding(
            padding: const EdgeInsets.symmetric(
                horizontal: 24, vertical: 20),
            child: Row(
              children: [
                Container(
                  width: 80, height: 80,
                  decoration: BoxDecoration(
                    color:        cs.primaryContainer,
                    borderRadius: BorderRadius.circular(12),
                  ),
                  child: Icon(Icons.music_note,
                      size: 40, color: cs.primary),
                ),
                const SizedBox(width: 16),
                Expanded(
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text(state.pistaActual.titulo,
                          style: const TextStyle(
                              fontWeight: FontWeight.bold,
                              fontSize: 17),
                          overflow: TextOverflow.ellipsis),
                      Text(state.pistaActual.artista,
                          style: TextStyle(
                              color: cs.onSurfaceVariant)),
                      const SizedBox(height: 4),
                      Text(
                        '${state.indice + 1} / '
                        '${UrlsMedia.playlist.length}',
                        style: TextStyle(
                            fontSize: 12,
                            color: cs.onSurfaceVariant),
                      ),
                    ],
                  ),
                ),
              ],
            ),
          ),

          // Barra de progreso
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            child: Column(
              children: [
                Slider(
                  value: state.progreso.clamp(0.0, 1.0),
                  onChanged: notif.buscar,
                ),
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    Text(_fmt(state.posicion),
                        style: const TextStyle(fontSize: 12)),
                    Text(_fmt(state.duracion),
                        style: const TextStyle(fontSize: 12)),
                  ],
                ),
              ],
            ),
          ),

          const SizedBox(height: 8),

          // Controles
          Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              IconButton(
                icon: const Icon(Icons.skip_previous),
                iconSize: 36,
                onPressed: state.hayAnterior ? notif.anterior : null,
              ),
              const SizedBox(width: 4),
              FilledIconButton(
                onPressed: notif.togglePlay,
                style: FilledIconButton.styleFrom(
                    minimumSize: const Size(56, 56)),
                child: state.cargando
                    ? const SizedBox(
                        width: 22, height: 22,
                        child: CircularProgressIndicator(
                            color: Colors.white, strokeWidth: 2))
                    : Icon(
                        state.tocando
                            ? Icons.pause
                            : Icons.play_arrow,
                        size: 30),
              ),
              const SizedBox(width: 4),
              IconButton(
                icon: const Icon(Icons.skip_next),
                iconSize: 36,
                onPressed: state.haySiguiente ? notif.siguiente : null,
              ),
            ],
          ),

          const SizedBox(height: 12),

          // Velocidad
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 16),
            child: Row(
              children: [
                Text('Velocidad: ${state.velocidad.toStringAsFixed(1)}x',
                    style: const TextStyle(fontSize: 13)),
                Expanded(
                  child: Slider(
                    value:     state.velocidad,
                    min:       0.5,
                    max:       2.0,
                    divisions: 6,
                    onChanged: notif.setVelocidad,
                  ),
                ),
              ],
            ),
          ),

          // Lista de pistas
          const Divider(),
          Expanded(
            child: ListView.builder(
              itemCount: UrlsMedia.playlist.length,
              itemBuilder: (_, i) {
                final pista  = UrlsMedia.playlist[i];
                final activa = state.indice == i;
                return ListTile(
                  dense:   true,
                  leading: CircleAvatar(
                    radius: 18,
                    backgroundColor: activa
                        ? cs.primaryContainer
                        : cs.surfaceContainerHighest,
                    child: activa && state.tocando
                        ? Icon(Icons.equalizer,
                            color: cs.primary, size: 16)
                        : Text('${i + 1}',
                            style: TextStyle(
                                fontSize: 12,
                                color: activa ? cs.primary : null)),
                  ),
                  title: Text(pista.titulo,
                      style: TextStyle(
                          fontWeight: activa
                              ? FontWeight.bold
                              : FontWeight.normal,
                          color: activa ? cs.primary : null,
                          fontSize: 14)),
                  subtitle: Text(pista.artista,
                      style: const TextStyle(fontSize: 12)),
                  onTap: () => notif.reproducirPista(i),
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

**`lib/main.dart` final** — une los dos módulos:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'features/audio/pantalla_audio.dart';
import 'features/video/pantalla_video.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const ProviderScope(child: MediaApp()));
}

class MediaApp extends StatelessWidget {
  const MediaApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Media App',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepOrange),
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
        0 => const PantallaAudio(),
        1 => const PantallaVideo(),
        _ => const PantallaAudio(),
      },
      bottomNavigationBar: NavigationBar(
        selectedIndex:         indice,
        onDestinationSelected: (i) =>
            ref.read(_tabProvider.notifier).state = i,
        destinations: const [
          NavigationDestination(
            icon:         Icon(Icons.music_note_outlined),
            selectedIcon: Icon(Icons.music_note),
            label:        'Audio',
          ),
          NavigationDestination(
            icon:         Icon(Icons.video_library_outlined),
            selectedIcon: Icon(Icons.video_library),
            label:        'Video',
          ),
        ],
      ),
    );
  }
}

final _tabProvider = StateProvider<int>((ref) => 0);
```

```bash
flutter run
# Tab Audio:
#   → Toca una pista de la lista → reproduce
#   → Botones anterior/siguiente → cambia de pista
#   → Slider de velocidad → cambia la velocidad de reproducción
#   → Al terminar una pista → avanza automáticamente

# Tab Video:
#   → Toca cualquier video de la lista → se carga y reproduce
#   → Chewie muestra controles automáticos (play, volumen, fullscreen)
#   → Toca pantalla completa para expandir
```

---

## Estado final del proyecto

```
lib/
├── main.dart                              ← actualizado en cada paso
├── core/
│   └── urls_media.dart                   ← paso 2
├── features/
│   ├── audio/
│   │   ├── servicio_audio.dart           ← paso 3
│   │   ├── audio_notifier.dart           ← paso 5
│   │   └── pantalla_audio.dart           ← paso 7
│   └── video/
│       ├── servicio_video.dart           ← paso 6
│       └── pantalla_video.dart           ← paso 6
```

---

## Ejercicios propuestos

1. **Modo aleatorio** — Añade a `AudioState` un campo `bool mezclado`.
   En `AudioNotifier.siguiente()`, si `mezclado` es true elige un índice
   aleatorio con `Random().nextInt(UrlsMedia.playlist.length)`.
   Añade un `IconButton` de shuffle en `PantallaAudio` que togglee el campo.

2. **Persistir progreso** — Al pausar o cerrar la app, guarda en
   `SharedPreferences` el índice de la pista actual y la posición en
   milisegundos. En `AudioNotifier.build()`, lee esos valores y llama
   `_svc.buscar(posicionGuardada)` después de cargar la pista.

3. **Notificación de medios** — Instala `audio_service` e integra
   `AudioNotifier` como handler. Al reproducir, aparecerá una notificación
   persistente en la barra del sistema con los botones play/pause/anterior/siguiente.
   El audio seguirá sonando con la pantalla apagada.

4. **Visor de letras simulado** — Añade a cada entrada de `UrlsMedia.playlist`
   un campo `List<({int ms, String texto})> letras`. Usando `posicionStream`,
   compara la posición actual con los timestamps y muestra la línea activa
   resaltada en un `AnimatedSwitcher`. Simula 3-4 líneas por canción.

---

## Resumen de la página 17

- `audioplayers` expone `AudioPlayer` con streams reactivos: `onPlayerStateChanged`, `onPositionChanged`, `onDurationChanged`, `onPlayerComplete`.
- Llamar `liberar()` / `dispose()` en el `dispose()` del widget o en `ref.onDispose()` del notifier es **obligatorio** — libera la sesión de audio del SO.
- `_svc.completoStream.listen((_) { siguiente(); })` en el notifier implementa avance automático sin código en la UI.
- `ChewieController` envuelve `VideoPlayerController` y añade controles de pantalla completa, volumen y seek automáticamente.
- `VideoPlayerController.networkUrl(Uri.parse(url))` carga video desde internet. Siempre hay que llamar `await initialize()` antes de usar el controller.
- `chewie` necesita que `VideoPlayerController` esté inicializado antes de crear `ChewieController` — de lo contrario lanza una excepción.
- El estado de audio en Riverpod con `ref.onDispose` garantiza que el reproductor se libera aunque el usuario cambie de tab abruptamente.
- `setPlaybackRate` de audioplayers acepta valores entre 0.5 y 2.0. Valores fuera de ese rango producen comportamiento indefinido según el dispositivo.

---

> **Siguiente página →** Página 18: Animaciones — `AnimatedContainer`,
> `AnimatedOpacity`, `Hero`, `AnimationController` y `Lottie`.