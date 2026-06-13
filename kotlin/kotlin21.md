# Curso Kotlin — Página 21
## Módulo 7 · Hardware y Sistema
### Audio y video streaming con Media3/ExoPlayer

---

## 🚀 TechDash — Módulo de Media (página 21)

En la página 18 construiste el módulo de hardware (GPS, sensores, red). Ahora añades la pantalla de reproducción multimedia que completa el dashboard:

```
TechDash
├── 📍 Hardware (GPS, Sensores, Red)  ← Página 18 ✅
└── 🎵 Media (Video + Audio)          ← Esta página
```

Al terminar tendrás un reproductor de video con controles táctiles y un reproductor de audio con playlist completa. En el Checkpoint final conectarás ambos módulos con un `NavHost` para tener la app TechDash completa.

### Archivos que crearás/modificarás en esta página

| | Archivo | Dónde crearlo |
|---|---------|---------------|
| 📄 Nuevo | `ReproductorVideoUrl.kt` | paquete `ui/media/` |
| 📄 Nuevo | `ReproductorAudio.kt` | paquete `ui/media/` |
| 📄 Nuevo | `ReproduccionService.kt` | paquete `ui/media/` |
| 📄 Nuevo | `PantallaStreaming.kt` | paquete `ui/media/` |
| ✏️ Modifica | `MainActivity.kt` | raíz del paquete — añade `TechDashApp()` en `setContent` |
| ✏️ Modifica | `AndroidManifest.xml` | `app/src/main/` — permisos + declaración del Service |
| ✏️ Modifica | `build.gradle.kts` | módulo `app` — dependencias de Media3 |

---

## ¿Qué es Media3?

Media3 es la librería oficial de Google que unifica la reproducción
de medios en Android. Reemplaza a los antiguos `MediaPlayer` y
`ExoPlayer` independiente bajo un único ecosistema:

```
Media3
├── ExoPlayer          ← motor de reproducción (audio + video)
├── MediaSession       ← controles del sistema (notificación, Bluetooth)
├── UI                 ← componentes de interfaz para Compose y XML
└── Transformer        ← edición de medios (no cubierto en este curso)
```

La diferencia clave con la página 17 (donde reproducíamos un archivo
local) es que ahora usamos **URLs remotas**, gestionamos **playlists**
y la reproducción continúa **en segundo plano** con notificación de medios.

---

## Dependencias

```kotlin
// build.gradle.kts
dependencies {
    val media3 = "1.5.1"
    implementation("androidx.media3:media3-exoplayer:$media3")
    implementation("androidx.media3:media3-exoplayer-hls:$media3")   // streaming HLS
    implementation("androidx.media3:media3-exoplayer-dash:$media3")  // streaming DASH
    implementation("androidx.media3:media3-session:$media3")         // MediaSession
    implementation("androidx.media3:media3-ui:$media3")              // controles XML
    implementation("androidx.media3:media3-ui-compose:$media3")      // controles Compose
    implementation("androidx.media3:media3-datasource-okhttp:$media3") // red con OkHttp
}
```

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />

<application ...>
    <!-- Servicio de reproducción en segundo plano -->
    <service
        android:name=".media.ReproduccionService"
        android:foregroundServiceType="mediaPlayback"
        android:exported="true">
        <intent-filter>
            <action android:name="androidx.media3.session.MediaSessionService" />
        </intent-filter>
    </service>
</application>
```

---

## Parte 1 — Reproducción básica desde URL

```kotlin
import androidx.media3.common.MediaItem
import androidx.media3.common.MediaMetadata
import androidx.media3.common.Player
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.ui.compose.PlayerSurface
import androidx.media3.ui.compose.SURFACE_TYPE_SURFACE_VIEW

// 📁 Puedes declarar esto al principio de ReproductorVideoUrl.kt o en ui/media/MediaEjemplos.kt
// URLs de ejemplo para pruebas (fuentes públicas y gratuitas)
object MediaEjemplos {
    // Video MP4 directo
    const val VIDEO_MP4   =
        "https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4"

    // Video HLS (streaming adaptativo)
    const val VIDEO_HLS   =
        "https://demo.unified-streaming.com/k8s/features/stable/video/tears-of-steel/tears-of-steel.iss/.m3u8"

    // Audio MP3 directo
    const val AUDIO_MP3   =
        "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3"

    // Radio online (stream continuo)
    const val RADIO_STREAM =
        "https://stream.live.vc.bbcmedia.co.uk/bbc_world_service"
}
```

### Reproductor de video en Compose

```kotlin
import androidx.compose.runtime.*
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.platform.LocalLifecycleOwner
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.LifecycleEventObserver

// 📁 Nuevo archivo → ui/media/ReproductorVideoUrl.kt
@Composable
fun ReproductorVideoUrl(
    url:      String,
    modifier: Modifier = Modifier
) {
    val context        = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current

    // El reproductor se recrea si cambia la URL
    val reproductor = remember(url) {
        ExoPlayer.Builder(context).build().apply {
            val item = MediaItem.Builder()
                .setUri(url)
                .setMediaMetadata(
                    MediaMetadata.Builder()
                        .setTitle("Video en streaming")
                        .build()
                )
                .build()
            setMediaItem(item)
            prepare()
            playWhenReady = true
        }
    }

    // Pausar/reanudar con el ciclo de vida
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> reproductor.play()
                Lifecycle.Event.ON_PAUSE  -> reproductor.pause()
                else -> {}
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
            reproductor.release()
        }
    }

    Box(modifier = modifier.background(Color.Black)) {
        PlayerSurface(
            player      = reproductor,
            surfaceType = SURFACE_TYPE_SURFACE_VIEW,
            modifier    = Modifier.fillMaxSize()
        )
        ControlesReproductor(reproductor = reproductor)
    }
}

@Composable
fun ControlesReproductor(reproductor: Player) {
    var reproduciendo by remember { mutableStateOf(reproductor.isPlaying) }
    var progreso      by remember { mutableFloatStateOf(0f) }
    var buffering     by remember { mutableStateOf(false) }
    var duracion      by remember { mutableLongStateOf(0L) }
    var visible       by remember { mutableStateOf(true) }

    // Ocultar controles automáticamente tras 3 segundos de inactividad
    LaunchedEffect(visible) {
        if (visible) {
            kotlinx.coroutines.delay(3_000)
            visible = false
        }
    }

    // Actualizar estado del reproductor
    LaunchedEffect(reproductor) {
        while (true) {
            reproduciendo = reproductor.isPlaying
            buffering     = reproductor.playbackState == Player.STATE_BUFFERING
            duracion      = reproductor.duration.coerceAtLeast(0)
            progreso      = if (duracion > 0)
                reproductor.currentPosition.toFloat() / duracion else 0f
            kotlinx.coroutines.delay(500)
        }
    }

    // Listener para detectar eventos del reproductor
    DisposableEffect(reproductor) {
        val listener = object : Player.Listener {
            override fun onPlaybackStateChanged(state: Int) {
                buffering = state == Player.STATE_BUFFERING
            }
            override fun onIsPlayingChanged(isPlaying: Boolean) {
                reproduciendo = isPlaying
            }
        }
        reproductor.addListener(listener)
        onDispose { reproductor.removeListener(listener) }
    }

    // Capa de controles — tap para mostrar/ocultar
    Box(
        modifier = Modifier
            .fillMaxSize()
            .clickable { visible = !visible }
    ) {
        // Spinner de buffering
        if (buffering) {
            CircularProgressIndicator(
                color    = Color.White,
                modifier = Modifier.align(Alignment.Center).size(48.dp)
            )
        }

        // Controles visibles
        AnimatedVisibility(
            visible = visible,
            enter   = fadeIn(),
            exit    = fadeOut(),
            modifier = Modifier.align(Alignment.BottomCenter)
        ) {
            Column(
                modifier = Modifier
                    .fillMaxWidth()
                    .background(
                        brush = androidx.compose.ui.graphics.Brush.verticalGradient(
                            colors = listOf(Color.Transparent, Color.Black.copy(alpha = 0.8f))
                        )
                    )
                    .padding(16.dp)
            ) {
                // Barra de progreso interactiva
                Slider(
                    value         = progreso,
                    onValueChange = { valor ->
                        reproductor.seekTo((valor * duracion).toLong())
                    },
                    colors   = SliderDefaults.colors(
                        thumbColor       = Color.White,
                        activeTrackColor = Color.White
                    ),
                    modifier = Modifier.fillMaxWidth()
                )

                Row(
                    Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.SpaceBetween,
                    verticalAlignment     = Alignment.CenterVertically
                ) {
                    // Tiempo
                    Text(
                        "${formatearTiempo(reproductor.currentPosition)} / ${formatearTiempo(duracion)}",
                        color = Color.White,
                        style = MaterialTheme.typography.labelSmall
                    )

                    // Botones centrales
                    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                        IconButton(onClick = {
                            reproductor.seekTo(
                                (reproductor.currentPosition - 10_000).coerceAtLeast(0)
                            )
                        }) {
                            Icon(Icons.Default.Replay10, null,
                                tint = Color.White, modifier = Modifier.size(28.dp))
                        }

                        IconButton(
                            onClick  = {
                                if (reproductor.isPlaying) reproductor.pause()
                                else reproductor.play()
                            },
                            modifier = Modifier
                                .size(48.dp)
                                .clip(CircleShape)
                                .background(Color.White.copy(alpha = 0.2f))
                        ) {
                            Icon(
                                if (reproduciendo) Icons.Default.Pause
                                else               Icons.Default.PlayArrow,
                                null,
                                tint     = Color.White,
                                modifier = Modifier.size(32.dp)
                            )
                        }

                        IconButton(onClick = {
                            reproductor.seekTo(
                                (reproductor.currentPosition + 10_000)
                                    .coerceAtMost(duracion)
                            )
                        }) {
                            Icon(Icons.Default.Forward10, null,
                                tint = Color.White, modifier = Modifier.size(28.dp))
                        }
                    }

                    // Silenciar
                    IconButton(onClick = {
                        reproductor.volume = if (reproductor.volume > 0f) 0f else 1f
                    }) {
                        Icon(
                            if (reproductor.volume > 0f) Icons.Default.VolumeUp
                            else Icons.Default.VolumeOff,
                            null, tint = Color.White
                        )
                    }
                }
            }
        }
    }
}

fun formatearTiempo(ms: Long): String {
    if (ms <= 0L) return "0:00"
    val seg  = (ms / 1000) % 60
    val min  = (ms / 1000) / 60
    val hora = min / 60
    return if (hora > 0) "%d:%02d:%02d".format(hora, min % 60, seg)
    else                 "%d:%02d".format(min, seg)
}
```

---

### ▶ Checkpoint 1 — Reproduce tu primer video en streaming

1. Añade las dependencias de Media3 al `build.gradle.kts`
2. Añade el permiso `INTERNET` al `AndroidManifest.xml`
3. Coloca `ReproductorVideoUrl(url = MediaEjemplos.VIDEO_MP4)` en una pantalla
4. Ejecuta en un dispositivo con conexión a internet

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código:

```kotlin
package com.ute.techdash

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.ute.techdash.ui.media.MediaEjemplos
import com.ute.techdash.ui.media.ReproductorVideoUrl
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                ReproductorVideoUrl(url = MediaEjemplos.VIDEO_MP4)
            }
        }
    }
}
```

**¿Qué deberías ver?**
- El video de *Big Buck Bunny* comienza a reproducirse en ~2–4 segundos
- Toca la pantalla para mostrar/ocultar los controles
- Los botones ⏪10 y 10⏩ saltan hacia atrás/adelante 10 segundos
- Los controles desaparecen solos tras 3 segundos de inactividad

> 💡 Si el video no carga: verifica que el emulador/dispositivo tenga internet. Abre el browser dentro del emulador para confirmarlo.

---

> ### 🧪 Prueba esto — Cambio dinámico de fuente
>
> El reproductor usa `remember(url)`, lo que significa que se **recrea automáticamente** cuando la URL cambia.
>
> **Reto 1:** Añade dos botones debajo del reproductor para alternar entre `VIDEO_MP4` y `VIDEO_HLS`. Declara un estado `var urlActual by remember { mutableStateOf(MediaEjemplos.VIDEO_MP4) }` y pásalo a `ReproductorVideoUrl`. Observa cómo el reproductor se reinicia limpiamente al cambiar.
>
> **Reto 2:** Los controles desaparecen tras 3 segundos (`delay(3_000)`). Cambia ese valor a `1_000` ms. Luego prueba con `8_000` ms. ¿Cuál sientes más natural para un reproductor de video?
>
> **Reflexión:** El `DisposableEffect` llama a `reproductor.release()` en `onDispose`. ¿Qué pasaría si olvidaras esa línea? ¿Notarías algo inmediatamente o solo después de navegar varias veces?

---

## Parte 2 — Reproductor de audio con playlist

```kotlin
// 📁 Nuevo archivo → ui/media/ReproductorAudio.kt
data class PistaAudio(
    val id:        Int,
    val titulo:    String,
    val artista:   String,
    val urlAudio:  String,
    val urlPortada: String? = null,
    val duracion:  Long     = 0L
)

@Composable
fun ReproductorAudio(
    playlist: List<PistaAudio>,
    modifier: Modifier = Modifier
) {
    val context = LocalContext.current

    // Reproductor con toda la playlist cargada
    val reproductor = remember {
        ExoPlayer.Builder(context).build().apply {
            val items = playlist.map { pista ->
                MediaItem.Builder()
                    .setUri(pista.urlAudio)
                    .setMediaMetadata(
                        MediaMetadata.Builder()
                            .setTitle(pista.titulo)
                            .setArtist(pista.artista)
                            .setArtworkUri(
                                pista.urlPortada?.let { android.net.Uri.parse(it) }
                            )
                            .build()
                    )
                    .build()
            }
            setMediaItems(items)
            prepare()
        }
    }

    DisposableEffect(Unit) {
        onDispose { reproductor.release() }
    }

    // Estado reactivo
    var indiceActual     by remember { mutableIntStateOf(0) }
    var reproduciendo    by remember { mutableStateOf(false) }
    var progreso         by remember { mutableFloatStateOf(0f) }
    var duracion         by remember { mutableLongStateOf(0L) }
    var modoRepeticion   by remember { mutableStateOf(Player.REPEAT_MODE_OFF) }
    var mezclado         by remember { mutableStateOf(false) }

    LaunchedEffect(reproductor) {
        while (true) {
            reproduciendo  = reproductor.isPlaying
            progreso       = if (reproductor.duration > 0)
                reproductor.currentPosition.toFloat() / reproductor.duration else 0f
            duracion       = reproductor.duration.coerceAtLeast(0)
            indiceActual   = reproductor.currentMediaItemIndex
            kotlinx.coroutines.delay(500)
        }
    }

    val pistaActual = playlist.getOrNull(indiceActual)

    Column(modifier = modifier.fillMaxWidth().padding(16.dp)) {

        // Portada
        Box(
            modifier = Modifier
                .fillMaxWidth()
                .height(240.dp)
                .clip(RoundedCornerShape(16.dp))
                .background(MaterialTheme.colorScheme.surfaceVariant),
            contentAlignment = Alignment.Center
        ) {
            if (pistaActual?.urlPortada != null) {
                AsyncImage(
                    model              = pistaActual.urlPortada,
                    contentDescription = "Portada",
                    contentScale       = ContentScale.Crop,
                    modifier           = Modifier.fillMaxSize()
                )
            } else {
                Icon(
                    Icons.Default.MusicNote,
                    contentDescription = null,
                    modifier = Modifier.size(80.dp),
                    tint     = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
        }

        Spacer(Modifier.height(24.dp))

        // Título y artista
        Text(
            pistaActual?.titulo  ?: "Sin título",
            style      = MaterialTheme.typography.titleLarge,
            fontWeight = FontWeight.Bold,
            maxLines   = 1,
            overflow   = TextOverflow.Ellipsis
        )
        Text(
            pistaActual?.artista ?: "Artista desconocido",
            style = MaterialTheme.typography.bodyMedium,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )

        Spacer(Modifier.height(16.dp))

        // Barra de progreso
        Slider(
            value         = progreso,
            onValueChange = { reproductor.seekTo((it * duracion).toLong()) },
            modifier      = Modifier.fillMaxWidth()
        )
        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
            Text(formatearTiempo(reproductor.currentPosition),
                style = MaterialTheme.typography.labelSmall,
                color = MaterialTheme.colorScheme.onSurfaceVariant)
            Text(formatearTiempo(duracion),
                style = MaterialTheme.typography.labelSmall,
                color = MaterialTheme.colorScheme.onSurfaceVariant)
        }

        Spacer(Modifier.height(8.dp))

        // Controles de reproducción
        Row(
            Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceEvenly,
            verticalAlignment     = Alignment.CenterVertically
        ) {
            // Mezclar
            IconButton(onClick = {
                mezclado = !mezclado
                reproductor.shuffleModeEnabled = mezclado
            }) {
                Icon(
                    Icons.Default.Shuffle,
                    null,
                    tint = if (mezclado) MaterialTheme.colorScheme.primary
                           else MaterialTheme.colorScheme.onSurfaceVariant
                )
            }

            // Anterior
            IconButton(
                onClick  = { reproductor.seekToPreviousMediaItem() },
                enabled  = reproductor.hasPreviousMediaItem()
            ) {
                Icon(Icons.Default.SkipPrevious, null, modifier = Modifier.size(36.dp))
            }

            // Play / Pause
            FilledIconButton(
                onClick  = {
                    if (reproductor.isPlaying) reproductor.pause()
                    else reproductor.play()
                },
                modifier = Modifier.size(64.dp)
            ) {
                Icon(
                    if (reproduciendo) Icons.Default.Pause else Icons.Default.PlayArrow,
                    null,
                    modifier = Modifier.size(36.dp)
                )
            }

            // Siguiente
            IconButton(
                onClick  = { reproductor.seekToNextMediaItem() },
                enabled  = reproductor.hasNextMediaItem()
            ) {
                Icon(Icons.Default.SkipNext, null, modifier = Modifier.size(36.dp))
            }

            // Repetir
            IconButton(onClick = {
                modoRepeticion = when (modoRepeticion) {
                    Player.REPEAT_MODE_OFF  -> Player.REPEAT_MODE_ONE
                    Player.REPEAT_MODE_ONE  -> Player.REPEAT_MODE_ALL
                    else                   -> Player.REPEAT_MODE_OFF
                }
                reproductor.repeatMode = modoRepeticion
            }) {
                Icon(
                    when (modoRepeticion) {
                        Player.REPEAT_MODE_ONE -> Icons.Default.RepeatOne
                        Player.REPEAT_MODE_ALL -> Icons.Default.Repeat
                        else                   -> Icons.Default.Repeat
                    },
                    null,
                    tint = if (modoRepeticion != Player.REPEAT_MODE_OFF)
                        MaterialTheme.colorScheme.primary
                    else MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
        }

        Spacer(Modifier.height(16.dp))

        // Lista de pistas
        Text("Lista de reproducción",
            style = MaterialTheme.typography.labelLarge,
            fontWeight = FontWeight.Bold)
        Spacer(Modifier.height(8.dp))

        playlist.forEachIndexed { index, pista ->
            val esActual = index == indiceActual
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .clip(RoundedCornerShape(8.dp))
                    .background(
                        if (esActual) MaterialTheme.colorScheme.primaryContainer
                        else Color.Transparent
                    )
                    .clickable { reproductor.seekToMediaItem(index) }
                    .padding(horizontal = 12.dp, vertical = 10.dp),
                verticalAlignment = Alignment.CenterVertically,
                horizontalArrangement = Arrangement.spacedBy(12.dp)
            ) {
                if (esActual && reproduciendo) {
                    Icon(Icons.Default.Equalizer, null,
                        tint     = MaterialTheme.colorScheme.primary,
                        modifier = Modifier.size(20.dp))
                } else {
                    Text(
                        "${index + 1}",
                        style = MaterialTheme.typography.labelMedium,
                        color = MaterialTheme.colorScheme.onSurfaceVariant,
                        modifier = Modifier.width(20.dp)
                    )
                }
                Column(Modifier.weight(1f)) {
                    Text(pista.titulo,
                        fontWeight = if (esActual) FontWeight.Bold else FontWeight.Normal,
                        maxLines   = 1, overflow = TextOverflow.Ellipsis,
                        color      = if (esActual) MaterialTheme.colorScheme.primary
                                     else MaterialTheme.colorScheme.onSurface)
                    Text(pista.artista,
                        style = MaterialTheme.typography.bodySmall,
                        color = MaterialTheme.colorScheme.onSurfaceVariant)
                }
            }
        }
    }
}
```

---

### ▶ Checkpoint 2 — Navega por la playlist de audio

1. Crea la lista de `PistaAudio` con las 3 canciones del ejemplo
2. Coloca `ReproductorAudio(playlist = miPlaylist)` en una pantalla
3. Ejecuta la app

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código:

```kotlin
package com.ute.techdash

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.runtime.remember
import com.ute.techdash.ui.media.PistaAudio
import com.ute.techdash.ui.media.ReproductorAudio
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                val playlist = remember {
                    listOf(
                        PistaAudio("Sinfonía de prueba", "Beethoven",
                            "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3"),
                        PistaAudio("Jazz tranquilo", "Artista 2",
                            "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-2.mp3"),
                        PistaAudio("Pop suave", "Artista 3",
                            "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-3.mp3")
                    )
                }
                ReproductorAudio(playlist = playlist)
            }
        }
    }
}
```

**¿Qué deberías ver?**
- La pista activa se resalta con fondo `primaryContainer` y el ícono de ecualizador
- Los botones ⏮ y ⏭ navegan entre pistas; ⏮ al inicio deshabilita si no hay anterior
- El modo mezcla (`shuffle`) reordena la lista aleatoriamente
- El modo repetición cicla: sin repetir → repetir una → repetir todas

> 💡 La playlist se carga completa en el `ExoPlayer` con `setMediaItems(items)`. La app puede saltar entre pistas sin re-crear el reproductor.

---

> ### 🧪 Prueba esto — Portada y quinta pista
>
> El campo `urlPortada` acepta cualquier URL de imagen PNG/JPG. Si es `null`, se muestra el ícono de nota musical.
>
> **Reto 1:** Añade una cuarta pista a la playlist con una URL de imagen como portada (puede ser cualquier URL pública de imagen). Verifica que la imagen aparece en el área de portada mientras esa pista está activa.
>
> **Reto 2:** La playlist usa `remember { ... }` sin key, lo que significa que **no se recrea** al recomponer. ¿Qué pasa si rotaras el dispositivo? ¿El audio se interrumpe? Pruébalo.
>
> **Reflexión:** En `ReproductorAudio` usamos `remember` sin key, pero en `ReproductorVideoUrl` usamos `remember(url)`. ¿Por qué la diferencia? ¿Qué pasaría si `ReproductorAudio` usara `remember(playlist)` y la playlist cambiara?

---

## Parte 3 — Reproducción en segundo plano con `MediaSessionService`

Para que el audio/video continúe en segundo plano y aparezca en la
notificación de medios del sistema, necesitamos un `Service`:

```kotlin
import androidx.media3.common.AudioAttributes
import androidx.media3.common.C
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.session.MediaSession
import androidx.media3.session.MediaSessionService

// 📁 Nuevo archivo → ui/media/ReproduccionService.kt
class ReproduccionService : MediaSessionService() {

    private var mediaSession: MediaSession? = null
    private lateinit var reproductor: ExoPlayer

    override fun onCreate() {
        super.onCreate()

        // Configurar atributos de audio — tipo MUSIC para ducking automático
        val atributosAudio = AudioAttributes.Builder()
            .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
            .setUsage(C.USAGE_MEDIA)
            .build()

        reproductor = ExoPlayer.Builder(this)
            .setAudioAttributes(atributosAudio, /* handleAudioFocus= */ true)
            .setHandleAudioBecomingNoisy(true)  // pausa al desconectar auriculares
            .build()

        // MediaSession vincula el reproductor con el sistema
        mediaSession = MediaSession.Builder(this, reproductor)
            .build()
    }

    // Retornar la sesión activa al cliente (la app)
    override fun onGetSession(
        controllerInfo: MediaSession.ControllerInfo
    ): MediaSession? = mediaSession

    override fun onDestroy() {
        mediaSession?.run {
            reproductor.release()
            release()
            mediaSession = null
        }
        super.onDestroy()
    }
}
```

### `MediaController` — conectar la UI al servicio

```kotlin
import androidx.media3.session.MediaController
import androidx.media3.session.SessionToken
import com.google.common.util.concurrent.MoreExecutors

@Composable
fun rememberMediaController(): MediaController? {
    val context = LocalContext.current
    var controlador by remember { mutableStateOf<MediaController?>(null) }

    DisposableEffect(context) {
        val token = SessionToken(
            context,
            android.content.ComponentName(context, ReproduccionService::class.java)
        )

        val futuro = MediaController.Builder(context, token)
            .buildAsync()

        futuro.addListener(
            { controlador = futuro.get() },
            MoreExecutors.directExecutor()
        )

        onDispose {
            futuro.cancel(true)
            controlador?.release()
            controlador = null
        }
    }

    return controlador
}
```

---

## Parte 4 — Pantalla completa de streaming

```kotlin
// 📁 Nuevo archivo → ui/media/PantallaStreaming.kt
@Composable
fun PantallaStreaming() {
    var modo by remember { mutableStateOf<ModoMedia>("video") }

    // Playlists de ejemplo
    val playlistAudio = remember {
        listOf(
            PistaAudio(
                id       = 1,
                titulo   = "Song 1",
                artista  = "SoundHelix",
                urlAudio = "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-1.mp3"
            ),
            PistaAudio(
                id       = 2,
                titulo   = "Song 2",
                artista  = "SoundHelix",
                urlAudio = "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-2.mp3"
            ),
            PistaAudio(
                id       = 3,
                titulo   = "Song 3",
                artista  = "SoundHelix",
                urlAudio = "https://www.soundhelix.com/examples/mp3/SoundHelix-Song-3.mp3"
            )
        )
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Streaming Media", fontWeight = FontWeight.Bold) }
            )
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .padding(padding)
                .fillMaxSize()
        ) {
            // Selector de modo
            TabRow(selectedTabIndex = if (modo == "video") 0 else 1) {
                Tab(
                    selected = modo == "video",
                    onClick  = { modo = "video" },
                    text     = { Text("Video") },
                    icon     = { Icon(Icons.Default.VideoLibrary, null) }
                )
                Tab(
                    selected = modo == "audio",
                    onClick  = { modo = "audio" },
                    text     = { Text("Audio") },
                    icon     = { Icon(Icons.Default.LibraryMusic, null) }
                )
            }

            when (modo) {
                "video" -> {
                    Column(modifier = Modifier.fillMaxSize()) {
                        // Reproductor de video
                        ReproductorVideoUrl(
                            url      = MediaEjemplos.VIDEO_MP4,
                            modifier = Modifier
                                .fillMaxWidth()
                                .aspectRatio(16f / 9f)
                        )

                        Spacer(Modifier.height(16.dp))

                        // Información del video
                        Column(Modifier.padding(16.dp)) {
                            Text("Big Buck Bunny",
                                style      = MaterialTheme.typography.titleLarge,
                                fontWeight = FontWeight.Bold)
                            Text("Pelicula animada de código abierto · Blender Foundation",
                                style = MaterialTheme.typography.bodyMedium,
                                color = MaterialTheme.colorScheme.onSurfaceVariant)
                            Spacer(Modifier.height(8.dp))
                            Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                                AssistChip(onClick = {}, label = { Text("HD") })
                                AssistChip(onClick = {}, label = { Text("Open Source") })
                            }
                        }
                    }
                }

                "audio" -> {
                    Column(
                        modifier = Modifier
                            .fillMaxSize()
                            .verticalScroll(rememberScrollState())
                    ) {
                        ReproductorAudio(
                            playlist = playlistAudio
                        )
                    }
                }
            }
        }
    }
}

// Alias de tipo para el modo de media
private typealias ModoMedia = String
```

---

## Manejo de errores en streaming

```kotlin
@Composable
fun ReproductorConManejadorErrores(url: String) {
    val context = LocalContext.current
    var estadoError by remember { mutableStateOf<String?>(null) }
    var intentos    by remember { mutableIntStateOf(0) }

    val reproductor = remember(url, intentos) {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(url))
            prepare()
            playWhenReady = true
        }
    }

    DisposableEffect(reproductor) {
        val listener = object : Player.Listener {
            override fun onPlayerError(error: androidx.media3.common.PlaybackException) {
                estadoError = when (error.errorCode) {
                    androidx.media3.common.PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED ->
                        "Sin conexión a internet"
                    androidx.media3.common.PlaybackException.ERROR_CODE_IO_FILE_NOT_FOUND ->
                        "Archivo no encontrado"
                    androidx.media3.common.PlaybackException.ERROR_CODE_TIMEOUT ->
                        "Tiempo de espera agotado"
                    else ->
                        "Error de reproducción: ${error.message}"
                }
            }
        }
        reproductor.addListener(listener)
        onDispose {
            reproductor.removeListener(listener)
            reproductor.release()
        }
    }

    if (estadoError != null) {
        Column(
            Modifier.fillMaxWidth().padding(16.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            Icon(Icons.Default.ErrorOutline, null,
                modifier = Modifier.size(48.dp),
                tint     = MaterialTheme.colorScheme.error)
            Text(estadoError!!, color = MaterialTheme.colorScheme.error)
            Button(onClick = {
                estadoError = null
                intentos++   // fuerza la recreación del reproductor vía remember(url, intentos)
            }) {
                Icon(Icons.Default.Refresh, null)
                Spacer(Modifier.width(4.dp))
                Text("Reintentar")
            }
        }
    } else {
        Box(
            modifier = Modifier
                .fillMaxWidth()
                .aspectRatio(16f / 9f)
        ) {
            PlayerSurface(
                player      = reproductor,
                surfaceType = SURFACE_TYPE_SURFACE_VIEW,
                modifier    = Modifier.fillMaxSize()
            )
            ControlesReproductor(reproductor)
        }
    }
}
```

---

## 🏁 TechDash — Integración final (páginas 18 + 21)

Con esta página y la página 18 completas, TechDash tiene todos sus módulos. Conecta las pantallas con un `NavHost`:

```kotlin
// MainActivity.kt
@Composable
fun TechDashApp() {
    val navController = rememberNavController()
    val contexto      = LocalContext.current
    val repositorioRed = remember { ConectividadRepository(contexto) }

    // El banner de red envuelve toda la app
    Column {
        BannerConectividad(repositorioRed)

        NavHost(navController, startDestination = "dashboard") {
            composable("dashboard") { PantallaDashboard(navController) }
            composable("gps")       { PantallaGPS() }
            composable("sensores")  { PantallaSensores() }
            composable("streaming") { PantallaStreaming() }
        }
    }
}

@Composable
fun PantallaDashboard(navController: NavController) {
    Scaffold(
        topBar = {
            TopAppBar(title = { Text("TechDash", fontWeight = FontWeight.Bold) })
        }
    ) { padding ->
        Column(
            modifier            = Modifier.padding(padding).fillMaxSize().padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            TarjetaModulo(
                titulo   = "GPS y Ubicación",
                subtitulo = "Coordenadas en tiempo real",
                icono    = Icons.Default.LocationOn,
                color    = MaterialTheme.colorScheme.primaryContainer,
                onClick  = { navController.navigate("gps") }
            )
            TarjetaModulo(
                titulo   = "Sensores del dispositivo",
                subtitulo = "Acelerómetro, giroscopio y luz",
                icono    = Icons.Default.Sensors,
                color    = MaterialTheme.colorScheme.secondaryContainer,
                onClick  = { navController.navigate("sensores") }
            )
            TarjetaModulo(
                titulo   = "Media y Streaming",
                subtitulo = "Video y audio online",
                icono    = Icons.Default.PlayCircle,
                color    = MaterialTheme.colorScheme.tertiaryContainer,
                onClick  = { navController.navigate("streaming") }
            )
        }
    }
}

@Composable
private fun TarjetaModulo(
    titulo:    String,
    subtitulo: String,
    icono:     ImageVector,
    color:     androidx.compose.ui.graphics.Color,
    onClick:   () -> Unit
) {
    ElevatedCard(
        onClick  = onClick,
        modifier = Modifier.fillMaxWidth()
    ) {
        Row(
            modifier          = Modifier.padding(20.dp),
            verticalAlignment = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.spacedBy(16.dp)
        ) {
            Box(
                modifier         = Modifier
                    .size(52.dp)
                    .clip(CircleShape)
                    .background(color),
                contentAlignment = Alignment.Center
            ) {
                Icon(icono, null, modifier = Modifier.size(28.dp))
            }
            Column {
                Text(titulo, fontWeight = FontWeight.Bold,
                    style = MaterialTheme.typography.titleMedium)
                Text(subtitulo,
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant)
            }
        }
    }
}
```

### ▶ Checkpoint final — TechDash completo

1. Añade `TechDashApp()` como contenido de tu `MainActivity`
2. Asegúrate de que `BannerConectividad` está por fuera del `NavHost`
3. Ejecuta la app

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código (versión final):

```kotlin
package com.ute.techdash

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.ute.techdash.ui.TechDashApp
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                TechDashApp()   // NavHost completo + BannerConectividad
            }
        }
    }
}
```

**La app está completa cuando:**
- [ ] La pantalla de inicio muestra las tres tarjetas de módulo
- [ ] Pulsas "GPS y Ubicación" → ves coordenadas en tiempo real
- [ ] Pulsas "Sensores" → los valores cambian al mover el dispositivo
- [ ] Pulsas "Media" → el video se reproduce y la playlist de audio funciona
- [ ] Desactivas el WiFi → el banner rojo aparece en **cualquier** pantalla donde estés

> ### 🧪 Prueba esto — Mejora el dashboard
>
> El `PantallaDashboard` muestra siempre las mismas tres tarjetas estáticas.
>
> **Reto:** Haz que la tarjeta de GPS muestre las últimas coordenadas conocidas directamente en el subtítulo del dashboard. Para ello necesitas que el `UbicacionViewModel` sobreviva a la navegación — decláralo en el nivel del `Activity` o usa un `ViewModelStore` compartido.
>
> **Pista:** `val vm: UbicacionViewModel = viewModel(viewModelStoreOwner = LocalContext.current as ViewModelStoreOwner)` o inyéctalo con Hilt/Koin a nivel de `Activity`.

---

## Ejercicios propuestos

1. **Velocidad de reproducción** — Añade al `ControlesReproductor` un
   menú desplegable con velocidades `0.5x`, `0.75x`, `1x`, `1.25x`, `1.5x`,
   `2x`. Usa `reproductor.setPlaybackSpeed(velocidad)`. Muestra la velocidad
   actual junto al tiempo de reproducción.

2. **Guardar posición** — Cuando el usuario abandona la pantalla, guarda
   la posición actual con `DataStore` (página 19). Al volver, reanuda desde
   ese punto con `reproductor.seekTo(posicionGuardada)`. Borra la posición
   guardada cuando la reproducción llega al final.

3. **Subtítulos / captions** — Investiga `MediaItem.Builder().setSubtitleConfigurations()`
   de Media3 para añadir un archivo `.vtt` de subtítulos a un video.
   Añade un botón de toggle de subtítulos en los controles del reproductor.

4. **Modo picture-in-picture** — Implementa el modo PiP para el reproductor
   de video. Cuando el usuario presiona el botón de inicio del sistema,
   la app debe entrar en PiP con `enterPictureInPictureMode()`. Añade
   las acciones de play/pause en el PiP con `RemoteAction`.

---

## Resumen de la página 21

- Media3 unifica `ExoPlayer`, `MediaSession` y la UI de medios bajo un mismo ecosistema. Usa la versión `1.5.1` junto al BOM de Compose.
- `MediaItem.Builder()` con `.setUri()` y `.setMediaMetadata()` crea un elemento de media con metadatos que aparecen en la notificación del sistema.
- `remember(url)` o `remember(url, intentos)` garantiza que el reproductor se recrea cuando cambian sus dependencias.
- `DisposableEffect` es el mecanismo para liberar `reproductor.release()` — nunca olvides llamarlo o habrá fugas de memoria.
- El reproductor se pausa automáticamente en `Lifecycle.Event.ON_PAUSE` y reanuda en `ON_RESUME` — necesario para liberar el audio correctamente.
- `setMediaItems(lista)` carga una playlist completa. `seekToMediaItem(index)`, `seekToNextMediaItem()` y `seekToPreviousMediaItem()` navegan entre pistas.
- `shuffleModeEnabled` y `repeatMode` (`REPEAT_MODE_OFF`, `REPEAT_MODE_ONE`, `REPEAT_MODE_ALL`) controlan la reproducción aleatoria y la repetición.
- `MediaSessionService` + `MediaController` permiten reproducción en segundo plano con notificación de medios nativa del sistema operativo.
- `setHandleAudioBecomingNoisy(true)` pausa la reproducción automáticamente al desconectar auriculares — comportamiento esperado por el usuario.
- `Player.Listener.onPlayerError()` expone códigos de error tipados para distinguir entre error de red, archivo no encontrado y timeout.

---

> **Siguiente página →** Página 22: Empaquetado APK/AAB firmado,
> keystore, Google Play Console y publicación final.