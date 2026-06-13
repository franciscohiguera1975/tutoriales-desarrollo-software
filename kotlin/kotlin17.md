# Curso Kotlin — Página 17
## Módulo 7 · Hardware y Sistema
### Cámara con CameraX, galería con PhotoPicker y reproducción de video

---

## 🚀 TechDash — Módulo de cámara y galería (página 17)

En la página 16 construiste los fundamentos (ciclo de vida y permisos). Ahora añades el módulo multimedia local de TechDash:

```
TechDash
├── 🔐 Permisos y ciclo de vida  ← Página 16 ✅
├── 📷 Cámara y Galería          ← Esta página
├── 📍 GPS, Sensores, Red        ← Página 18
└── 🎵 Media y Streaming         ← Página 21
```

Al terminar tendrás `PantallaMultimedia` integrada: el usuario puede tomar fotos con CameraX, seleccionar imágenes y videos con PhotoPicker, y reproducir videos locales con ExoPlayer.

### Archivos que crearás/modificarás en esta página

| | Archivo | Dónde crearlo |
|---|---------|---------------|
| 📄 Nuevo | `PantallaCamara.kt` | paquete `ui/multimedia/` |
| 📄 Nuevo | `SelectorGaleria.kt` | paquete `ui/multimedia/` |
| 📄 Nuevo | `VisorImagen.kt` | paquete `ui/multimedia/` |
| 📄 Nuevo | `ReproductorVideo.kt` | paquete `ui/multimedia/` |
| 📄 Nuevo | `PantallaMultimedia.kt` | paquete `ui/multimedia/` |
| 📄 Nuevo | `file_paths.xml` | `app/src/main/res/xml/` — crea la carpeta `xml` si no existe |
| ✏️ Modifica | `AndroidManifest.xml` | `app/src/main/` — permisos + FileProvider |
| ✏️ Modifica | `build.gradle.kts` | módulo `app` — dependencias de CameraX, Media3 y Coil |

---

## Dependencias

```kotlin
// build.gradle.kts
dependencies {
    // CameraX
    val cameraxVersion = "1.4.1"
    implementation("androidx.camera:camera-core:$cameraxVersion")
    implementation("androidx.camera:camera-camera2:$cameraxVersion")
    implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
    implementation("androidx.camera:camera-view:$cameraxVersion")
    implementation("androidx.camera:camera-video:$cameraxVersion")

    // Media3 / ExoPlayer para reproducción de video
    val media3Version = "1.5.1"
    implementation("androidx.media3:media3-exoplayer:$media3Version")
    implementation("androidx.media3:media3-ui:$media3Version")

    // Coil para mostrar imágenes
    implementation("io.coil-kt:coil-compose:2.7.0")
}
```

### `AndroidManifest.xml`

> ⚠️ `tools:ignore` requiere `xmlns:tools="http://schemas.android.com/tools"` en la etiqueta raíz `<manifest ...>` de tu archivo.

```xml
<uses-permission android:name="android.permission.CAMERA" />
<!-- Android 13+ (API 33+) — sin permiso de almacenamiento gracias a scoped storage -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
<!-- Android 9–12 (API 28–32) — galería legacy y acceso a archivos de video -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
    android:maxSdkVersion="28"
    tools:ignore="ScopedStorage" />

<application ...>
    <!-- FileProvider — para compartir URIs de archivos privados -->
    <provider
        android:name="androidx.core.content.FileProvider"
        android:authorities="${applicationId}.fileprovider"
        android:exported="false"
        android:grantUriPermissions="true">
        <meta-data
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/file_paths" />
    </provider>
</application>
```

### `res/xml/file_paths.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <cache-path name="imagenes_cache" path="images/" />
    <cache-path name="videos_cache"   path="videos/" />
</paths>
```

---

## Parte 1 — Fotografía con CameraX

CameraX es la librería oficial de Google para acceder a la cámara.
Abstrae las diferencias entre fabricantes y versiones de Android:

📁 **Nuevo archivo → `ui/multimedia/PantallaCamara.kt`**

```kotlin
package com.ute.techdash.ui.multimedia

import android.net.Uri
import android.util.Log
import androidx.camera.core.CameraSelector
import androidx.camera.core.ImageCapture
import androidx.camera.core.ImageCaptureException
import androidx.camera.core.Preview
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.camera.view.PreviewView
import androidx.compose.foundation.background
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Cameraswitch
import androidx.compose.material.icons.filled.Close
import androidx.compose.material.icons.filled.FlashOff
import androidx.compose.material.icons.filled.FlashOn
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.compose.ui.viewinterop.AndroidView
import androidx.core.content.ContextCompat
import androidx.lifecycle.compose.LocalLifecycleOwner
import java.io.File
import java.text.SimpleDateFormat
import java.util.Locale

@Composable
fun PantallaCamara(
    onFotoTomada: (Uri) -> Unit,
    onCerrar:     () -> Unit
) {
    val context        = LocalContext.current
    val lifecycleOwner = LocalLifecycleOwner.current

    var capturaImagen     by remember { mutableStateOf<ImageCapture?>(null) }
    var usarCamaraFrontal by remember { mutableStateOf(false) }
    var flashActivo       by remember { mutableStateOf(false) }
    var tomandoFoto       by remember { mutableStateOf(false) }

    // PreviewView se crea una sola vez y se reutiliza — evita recrear la superficie al cambiar cámara
    val vistaPreviaView = remember {
        PreviewView(context).apply {
            scaleType = PreviewView.ScaleType.FILL_CENTER
        }
    }

    // LaunchedEffect se ejecuta cuando cambia usarCamaraFrontal o flashActivo
    // — garantiza que CameraX se reinicia con los nuevos valores capturados correctamente
    LaunchedEffect(usarCamaraFrontal, flashActivo) {
        val proveedorFuturo = ProcessCameraProvider.getInstance(context)
        proveedorFuturo.addListener({
            try {
                val proveedor = proveedorFuturo.get()

                val selectorCamara = if (usarCamaraFrontal)
                    CameraSelector.DEFAULT_FRONT_CAMERA
                else
                    CameraSelector.DEFAULT_BACK_CAMERA

                val preview = Preview.Builder().build().also {
                    it.surfaceProvider = vistaPreviaView.surfaceProvider
                }

                val imageCapture = ImageCapture.Builder()
                    .setFlashMode(
                        if (flashActivo) ImageCapture.FLASH_MODE_ON
                        else             ImageCapture.FLASH_MODE_OFF
                    )
                    .setCaptureMode(ImageCapture.CAPTURE_MODE_MAXIMIZE_QUALITY)
                    .build()

                capturaImagen = imageCapture

                proveedor.unbindAll()

                // Verificar que el dispositivo tiene esa cámara antes de hacer bind
                if (proveedor.hasCamera(selectorCamara)) {
                    proveedor.bindToLifecycle(
                        lifecycleOwner, selectorCamara, preview, imageCapture
                    )
                } else {
                    Log.e("TechDash", "Cámara no disponible en este dispositivo")
                    if (usarCamaraFrontal) usarCamaraFrontal = false
                }
            } catch (e: Exception) {
                Log.e("TechDash", "Error al iniciar cámara", e)
            }
        }, ContextCompat.getMainExecutor(context))
    }

    fun tomarFoto() {
        val captura = capturaImagen ?: return
        tomandoFoto = true

        val archivo = File(
            context.cacheDir.resolve("images").also { it.mkdirs() },
            SimpleDateFormat("yyyyMMdd_HHmmss", Locale.US).format(System.currentTimeMillis()) + ".jpg"
        )

        captura.takePicture(
            ImageCapture.OutputFileOptions.Builder(archivo).build(),
            ContextCompat.getMainExecutor(context),
            object : ImageCapture.OnImageSavedCallback {
                override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                    tomandoFoto = false
                    val uri = androidx.core.content.FileProvider.getUriForFile(
                        context, "${context.packageName}.fileprovider", archivo
                    )
                    onFotoTomada(uri)
                }
                override fun onError(exc: ImageCaptureException) {
                    tomandoFoto = false
                    Log.e("TechDash", "Error al guardar foto", exc)
                }
            }
        )
    }

    Box(modifier = Modifier.fillMaxSize().background(Color.Black)) {

        // AndroidView con factory simple — la vista ya está inicializada en remember
        AndroidView(
            factory  = { vistaPreviaView },
            modifier = Modifier.fillMaxSize()
        )

        // Controles superpuestos
        Column(
            modifier            = Modifier.align(Alignment.BottomCenter).padding(32.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.spacedBy(16.dp)
        ) {
            Row(
                Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween
            ) {
                // Cerrar
                IconButton(
                    onClick  = onCerrar,
                    modifier = Modifier.size(48.dp).clip(CircleShape)
                        .background(Color.Black.copy(alpha = 0.5f))
                ) {
                    Icon(Icons.Default.Close, "Cerrar", tint = Color.White)
                }

                // Flash
                IconButton(
                    onClick  = { flashActivo = !flashActivo },
                    modifier = Modifier.size(48.dp).clip(CircleShape)
                        .background(Color.Black.copy(alpha = 0.5f))
                ) {
                    Icon(
                        if (flashActivo) Icons.Default.FlashOn else Icons.Default.FlashOff,
                        "Flash",
                        tint = if (flashActivo) Color.Yellow else Color.White
                    )
                }
            }

            // Disparador
            Box(contentAlignment = Alignment.Center) {
                if (tomandoFoto) {
                    CircularProgressIndicator(color = Color.White, modifier = Modifier.size(80.dp))
                } else {
                    Box(
                        modifier = Modifier
                            .size(80.dp)
                            .clip(CircleShape)
                            .background(Color.White)
                            .clickable { tomarFoto() }
                    )
                }
            }

            // Cambiar cámara — al cambiar el estado, LaunchedEffect reinicia CameraX
            TextButton(onClick = { usarCamaraFrontal = !usarCamaraFrontal }) {
                Icon(Icons.Default.Cameraswitch, null, tint = Color.White)
                Spacer(Modifier.width(4.dp))
                Text(
                    if (usarCamaraFrontal) "Frontal" else "Trasera",
                    color = Color.White
                )
            }
        }
    }
}
```

---

### ▶ Checkpoint 1 — Toma tu primera foto con CameraX

**Pasos para ejecutar:**

1. Crea el paquete `ui/multimedia/` (clic derecho sobre `techdash` → **New → Package** → escribe `ui.multimedia`)
2. Crea `PantallaCamara.kt` en ese paquete y pega el código de arriba
3. Verifica que `res/xml/file_paths.xml` existe con el contenido de la sección de configuración
4. Verifica que `AndroidManifest.xml` tiene el permiso de cámara y el bloque `<provider>` del FileProvider
5. Reemplaza `MainActivity.kt` con el código de abajo
6. Ejecuta con **▶** — preferiblemente en un **dispositivo físico** (la cámara del emulador es simulada)

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código:

```kotlin
package com.ute.techdash

import android.os.Bundle
import android.util.Log
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.ute.techdash.ui.multimedia.PantallaCamara
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                PantallaCamara(
                    onFotoTomada = { uri -> Log.d("TechDash", "Foto guardada: $uri") },
                    onCerrar     = {}
                )
            }
        }
    }
}
```

**Escenarios que debes probar:**

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Abrir la app | Diálogo del sistema: "¿Permitir a TechDash tomar fotos?" |
| 2 | Conceder el permiso | El visor de la cámara ocupa toda la pantalla |
| 3 | Pulsar el botón circular blanco | Se toma la foto — en **Logcat** filtra `TechDash` y verás `Foto guardada: content://...` |
| 4 | Pulsar el botón de flash ⚡ | Alterna entre encendido/apagado (el ícono cambia) |
| 5 | Pulsar el botón de cambio de cámara 🔄 | Cambia entre frontal y trasera **sin reiniciar** la app |
| 6 | Rotar el dispositivo | La cámara sigue funcionando sin reiniciar la sesión |

> ⚠️ **CameraX requiere dispositivo físico** — El emulador puede cerrar la app al iniciar la cámara porque la virtualización del hardware no es completa. Prueba siempre en un teléfono real. Si insistes con el emulador activa primero la cámara virtual en `Extended controls (···) → Camera → Virtual scene`.
>
> Si el permiso de cámara fue denegado: mantén presionado el ícono de la app → **Información de la app → Permisos → Cámara → Permitir**.

---

> ### 🧪 Prueba esto — Temporizador de disparo
>
> La cámara dispara inmediatamente al pulsar el botón. Muchas apps tienen un temporizador.
>
> **Reto:** Añade un `IconButton` que cicle entre `Off → 3s → 10s`. Al pulsar el disparador con temporizador activo, muestra la cuenta regresiva en pantalla con un `Text` grande superpuesto usando `delay()` en un coroutine lanzado desde `rememberCoroutineScope()`. Al llegar a 0, llama a `tomarFoto()`.
>
> **Reflexión:** La implementación usa `LaunchedEffect(usarCamaraFrontal, flashActivo)` para reiniciar CameraX cuando cambia el estado, y el `PreviewView` se crea con `remember { PreviewView(context) }` fuera del `AndroidView`. ¿Por qué no funciona poner la lógica de inicio de CameraX directamente en el lambda `update` del `AndroidView`? Pista: `update` captura los valores en el momento de la recomposición, pero los callbacks de `ProcessCameraProvider.addListener` se ejecutan de forma asíncrona en otro hilo — cuando el callback llega, los valores de `usarCamaraFrontal` y `flashActivo` que capturó `update` pueden ser ya obsoletos (stale).

---

## Parte 2 — Galería con PhotoPicker

`PhotoPicker` es la API moderna de Android para seleccionar
imágenes y videos sin necesitar permisos de lectura en Android 13+:

📁 **Nuevo archivo → `ui/multimedia/SelectorGaleria.kt`**

```kotlin
package com.ute.techdash.ui.multimedia

import android.net.Uri
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.PickVisualMediaRequest
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.width
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Collections
import androidx.compose.material.icons.filled.PhotoLibrary
import androidx.compose.material.icons.filled.VideoLibrary
import androidx.compose.material3.Button
import androidx.compose.material3.Icon
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun SelectorGaleria(
    onImagenSeleccionada: (Uri) -> Unit
) {
    // PickVisualMedia — una sola imagen
    val launcherUnaImagen = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.PickVisualMedia()
    ) { uri ->
        uri?.let { onImagenSeleccionada(it) }
    }

    // PickMultipleVisualMedia — hasta 5 imágenes
    val launcherMultiple = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.PickMultipleVisualMedia(maxItems = 5)
    ) { uris ->
        uris.forEach { onImagenSeleccionada(it) }
    }

    Column(
        Modifier.padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        // Solo imágenes
        OutlinedButton(
            onClick  = {
                launcherUnaImagen.launch(
                    PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
                )
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Icon(Icons.Default.PhotoLibrary, null)
            Spacer(Modifier.width(8.dp))
            Text("Seleccionar una imagen")
        }

        // Imágenes y videos
        OutlinedButton(
            onClick  = {
                launcherUnaImagen.launch(
                    PickVisualMediaRequest(
                        ActivityResultContracts.PickVisualMedia.ImageAndVideo
                    )
                )
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Icon(Icons.Default.VideoLibrary, null)
            Spacer(Modifier.width(8.dp))
            Text("Seleccionar imagen o video")
        }

        // Múltiples imágenes
        Button(
            onClick  = {
                launcherMultiple.launch(
                    PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
                )
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Icon(Icons.Default.Collections, null)
            Spacer(Modifier.width(8.dp))
            Text("Seleccionar hasta 5 imágenes")
        }
    }
}
```

---

## Parte 3 — Mostrar imágenes con Coil

📁 **Nuevo archivo → `ui/multimedia/VisorImagen.kt`**

```kotlin
package com.ute.techdash.ui.multimedia

import android.net.Uri
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.size
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.ImageNotSupported
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import coil.compose.AsyncImage
import coil.request.ImageRequest

@Composable
fun VisorImagen(
    uri:      Uri?,
    modifier: Modifier = Modifier
) {
    if (uri == null) {
        // Placeholder
        Box(
            modifier         = modifier
                .background(MaterialTheme.colorScheme.surfaceVariant),
            contentAlignment = Alignment.Center
        ) {
            Icon(
                Icons.Default.ImageNotSupported,
                contentDescription = "Sin imagen",
                tint     = MaterialTheme.colorScheme.onSurfaceVariant,
                modifier = Modifier.size(48.dp)
            )
        }
        return
    }

    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(uri)
            .crossfade(true)        // animación de fade al cargar
            .build(),
        contentDescription = "Imagen seleccionada",
        contentScale       = ContentScale.Crop,
        modifier           = modifier
    )
}
```

---

### ▶ Checkpoint 2 — Selecciona imágenes de la galería

**Pasos para ejecutar:**

1. Crea `SelectorGaleria.kt` en el paquete `ui/multimedia/` y pega el código de la Parte 2
2. Crea `VisorImagen.kt` en el mismo paquete y pega el código de la Parte 3
3. Reemplaza `MainActivity.kt` con el código de abajo
4. Ejecuta con **▶**

> 💡 **API 28 y anteriores:** `PhotoPicker` se retrocompatibiliza automáticamente vía Google Play Services. Si el emulador no tiene Play Services (imagen de tipo "Google APIs" o "AOSP"), usa un emulador con API 29+ o un dispositivo físico.

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código:

```kotlin
package com.ute.techdash

import android.net.Uri
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.Column
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import com.ute.techdash.ui.multimedia.SelectorGaleria
import com.ute.techdash.ui.multimedia.VisorImagen
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                var uriActual by remember { mutableStateOf<Uri?>(null) }
                Column {
                    SelectorGaleria(onImagenSeleccionada = { uriActual = it })
                    uriActual?.let { VisorImagen(uri = it) }
                }
            }
        }
    }
}
```

**Escenarios que debes probar:**

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Pulsar "Seleccionar una imagen" | Se abre el `PhotoPicker` nativo del sistema |
| 2 | Elegir una foto | La imagen aparece en `VisorImagen` con animación `crossfade` |
| 3 | Pulsar "Seleccionar imagen o video" | El picker muestra imágenes y videos juntos |
| 4 | Pulsar "Seleccionar hasta 5 imágenes" | Puedes marcar varias antes de confirmar con ✔ |
| 5 | Seleccionar 3 imágenes a la vez | Cada una llama `onImagenSeleccionada` — `VisorImagen` muestra la última |
| 6 | Rotar el dispositivo | El `uri` persiste gracias a `remember` |

> 💡 **Emulador vacío:** Añade fotos con `adb push foto.jpg /sdcard/Pictures/` y luego `adb shell am broadcast -a android.intent.action.MEDIA_SCANNER_SCAN_FILE -d file:///sdcard/Pictures/foto.jpg`

---

> ### 🧪 Prueba esto — Pinch-to-zoom y permisos en versiones antiguas
>
> `AsyncImage` muestra la imagen a tamaño fijo, sin posibilidad de zoom.
>
> **Reto 1:** Implementa pinch-to-zoom en `VisorImagen` usando `rememberTransformableState` y el modifier `.transformable(state)`. Declara `var escala by remember { mutableFloatStateOf(1f) }` y aplícala con `.graphicsLayer { scaleX = escala; scaleY = escala }`. Limita con `coerceIn(1f, 5f)`.
>
> **Reto 2:** `PhotoPicker` **no** requiere permiso `READ_MEDIA_IMAGES` en Android 13+ (API 33). ¿Por qué entonces lo declaramos en el manifest? Prueba a quitarlo y ejecuta en un emulador con API 33. ¿Funciona? ¿Y en uno con API 32?
>
> **Reflexión:** `PhotoPicker` devuelve URIs temporales que expiran cuando la app se cierra. ¿Qué pasaría si guardaras esa URI en `DataStore` (página 19) e intentaras mostrar la imagen la próxima vez que abras la app?

---

## Parte 4 — Reproducción de video con Media3/ExoPlayer

📁 **Nuevo archivo → `ui/multimedia/ReproductorVideo.kt`**

```kotlin
package com.ute.techdash.ui.multimedia

import android.net.Uri
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Forward10
import androidx.compose.material.icons.filled.Pause
import androidx.compose.material.icons.filled.PlayArrow
import androidx.compose.material.icons.filled.Replay10
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.material3.LinearProgressIndicator
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.DisposableEffect
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableFloatStateOf
import androidx.compose.runtime.mutableLongStateOf
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.LocalLifecycleOwner
import android.view.TextureView
import androidx.compose.ui.viewinterop.AndroidView
import androidx.media3.common.MediaItem
import androidx.media3.common.Player
import androidx.media3.exoplayer.ExoPlayer

@Composable
fun ReproductorVideo(
    uri:      Uri,
    modifier: Modifier = Modifier
) {
    val context = LocalContext.current

    // Crear y recordar el reproductor
    val reproductor = remember(uri) {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(uri))
            prepare()
            playWhenReady = true   // reproducir automáticamente
        }
    }

    // Limpiar el reproductor cuando el composable sale de la composición
    DisposableEffect(reproductor) {
        onDispose {
            reproductor.release()  // IMPORTANTE — liberar recursos
        }
    }

    // Pausar/reanudar según el ciclo de vida
    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner) {
        val observer = androidx.lifecycle.LifecycleEventObserver { _, event ->
            when (event) {
                androidx.lifecycle.Lifecycle.Event.ON_PAUSE  -> reproductor.pause()
                androidx.lifecycle.Lifecycle.Event.ON_RESUME -> reproductor.play()
                else -> {}
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }

    Box(modifier = modifier.background(Color.Black)) {
        // Superficie de video
        // TextureView no tiene problemas de z-order en Compose (funciona en Android 9-11+)
        // SurfaceView (que usa PlayerView internamente) puede mostrar negro en versiones antiguas
        AndroidView(
            factory  = { ctx ->
                TextureView(ctx).also { tv ->
                    reproductor.setVideoTextureView(tv)
                }
            },
            modifier = Modifier.fillMaxSize()
        )

        // Controles personalizados
        ControlesVideo(reproductor = reproductor)
    }
}

@Composable
fun ControlesVideo(reproductor: Player) {
    var reproduciendo by remember { mutableStateOf(reproductor.isPlaying) }
    var progreso      by remember { mutableFloatStateOf(0f) }
    var duracion      by remember { mutableLongStateOf(0L) }

    // Actualizar progreso periódicamente
    LaunchedEffect(reproductor) {
        while (true) {
            reproduciendo = reproductor.isPlaying
            progreso      = if (reproductor.duration > 0)
                reproductor.currentPosition.toFloat() / reproductor.duration
            else 0f
            duracion = reproductor.duration
            kotlinx.coroutines.delay(500)
        }
    }

    Box(
        modifier         = Modifier
            .fillMaxSize()
            .padding(16.dp),
        contentAlignment = Alignment.BottomCenter
    ) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .clip(RoundedCornerShape(12.dp))
                .background(Color.Black.copy(alpha = 0.6f))
                .padding(12.dp)
        ) {
            // Barra de progreso
            LinearProgressIndicator(
                progress = { progreso },
                modifier = Modifier.fillMaxWidth(),
                color    = Color.White
            )

            Spacer(Modifier.height(8.dp))

            Row(
                Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment     = Alignment.CenterVertically
            ) {
                // Tiempo actual / duración
                Text(
                    "${formatearTiempo(reproductor.currentPosition)} / ${formatearTiempo(duracion)}",
                    color = Color.White,
                    style = MaterialTheme.typography.labelMedium
                )

                // Controles
                Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                    // Retroceder 10s
                    IconButton(onClick = {
                        reproductor.seekTo((reproductor.currentPosition - 10_000).coerceAtLeast(0))
                    }) {
                        Icon(Icons.Default.Replay10, null, tint = Color.White)
                    }

                    // Play / Pause
                    IconButton(onClick = {
                        if (reproductor.isPlaying) reproductor.pause()
                        else reproductor.play()
                    }) {
                        Icon(
                            if (reproduciendo) Icons.Default.Pause else Icons.Default.PlayArrow,
                            null,
                            tint     = Color.White,
                            modifier = Modifier.size(32.dp)
                        )
                    }

                    // Avanzar 10s
                    IconButton(onClick = {
                        reproductor.seekTo(
                            (reproductor.currentPosition + 10_000)
                                .coerceAtMost(reproductor.duration)
                        )
                    }) {
                        Icon(Icons.Default.Forward10, null, tint = Color.White)
                    }
                }
            }
        }
    }
}

fun formatearTiempo(ms: Long): String {
    if (ms <= 0) return "0:00"
    val seg  = (ms / 1000) % 60
    val min  = (ms / 1000) / 60
    val hora = min / 60
    return if (hora > 0) "%d:%02d:%02d".format(hora, min % 60, seg)
    else                 "%d:%02d".format(min, seg)
}
```

---

### ▶ Checkpoint 3 — Pantalla multimedia completa con reproductor de video

**Pasos para ejecutar:**

1. Crea `ReproductorVideo.kt` en el paquete `ui/multimedia/` y pega el código de la Parte 4
2. Crea `PantallaMultimedia.kt` en el mismo paquete y pega el código del "Programa completo"
3. Reemplaza `MainActivity.kt` con el código de abajo
4. Ejecuta con **▶**

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código:

```kotlin
package com.ute.techdash

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.ute.techdash.ui.multimedia.PantallaMultimedia
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                PantallaMultimedia()
            }
        }
    }
}
```

**Escenarios que debes probar:**

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Abrir la app | Pantalla vacía con 3 botones: Cámara, Galería, Video |
| 2 | Pulsar "Cámara" (sin permiso) | Diálogo del sistema para conceder permiso de cámara |
| 3 | Conceder permiso y volver | Se abre `PantallaCamara` a pantalla completa |
| 4 | Tomar una foto | La foto aparece en el grid y en el visor superior |
| 5 | Pulsar "Galería" | PhotoPicker abre — selecciona hasta 10 imágenes |
| 6 | Pulsar "Video" | PhotoPicker filtra solo videos |
| 7 | Pulsar un video en el grid | Se abre `ReproductorVideo` con play automático |
| 8 | Pulsar ⏪10 y 10⏩ | Saltan 10 segundos atrás/adelante |
| 9 | Rotar el dispositivo | El video pausa en `onPause` y reanuda en `onResume` sin perder posición |

> 💡 **Emulador sin videos:** Añade con `adb push video.mp4 /sdcard/Movies/` y luego `adb shell am broadcast -a android.intent.action.MEDIA_SCANNER_SCAN_FILE -d file:///sdcard/Movies/video.mp4`

> ⚠️ **Error "Conflicting overloads"** — Si el IDE marca `ReproductorVideo`, `ControlesVideo` o `formatearTiempo` con el error *"Conflicting overloads: fun ... is already defined in the root package"*, significa que tu archivo `PantallaMultimedia.kt` contiene esas funciones duplicadas.
>
> **Causa:** copiaste el contenido de `ReproductorVideo.kt` dentro de `PantallaMultimedia.kt` (o tu archivo tiene una versión anterior).
>
> **Solución:** abre `PantallaMultimedia.kt` en tu proyecto y deja **solo** estas dos cosas:
> 1. `sealed class ItemMedia { ... }` — el modelo de datos
> 2. `fun PantallaMultimedia() { ... }` — el composable principal
>
> Borra cualquier definición de `fun ReproductorVideo`, `fun ControlesVideo` o `fun formatearTiempo` que encuentres en ese archivo.

---

> ### 🧪 Prueba esto — Seekbar interactiva
>
> `ControlesVideo` usa `LinearProgressIndicator` que solo muestra progreso pero no permite arrastrar para saltar.
>
> **Reto:** Reemplaza el `LinearProgressIndicator` por un `Slider`. En `onValueChange`, llama a `reproductor.seekTo((valor * duracion).toLong())`. El `LaunchedEffect` seguirá actualizando el valor del `Slider` automáticamente cada 500ms.
>
> **Reflexión:** El `LaunchedEffect` actualiza `progreso` cada 500ms. Mientras el usuario arrastra el slider, el LaunchedEffect podría sobreescribir el valor del arrastre. ¿Cómo lo resolverías? Pista: guarda `var estaDraggeando by remember { mutableStateOf(false) }` y actualiza `progreso` solo cuando `!estaDraggeando` (úsalo en `onValueChangeFinished` del Slider).

---

## Programa completo — pantalla multimedia integrada

📁 **Nuevo archivo → `ui/multimedia/PantallaMultimedia.kt`**

```kotlin
package com.ute.techdash.ui.multimedia

import android.net.Uri
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.PickVisualMediaRequest
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.animation.AnimatedVisibility
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.aspectRatio
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.lazy.grid.GridCells
import androidx.compose.foundation.lazy.grid.LazyVerticalGrid
import androidx.compose.foundation.lazy.grid.items
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.CameraAlt
import androidx.compose.material.icons.filled.Close
import androidx.compose.material.icons.filled.PermMedia
import androidx.compose.material.icons.filled.PhotoLibrary
import androidx.compose.material.icons.filled.PlayCircle
import androidx.compose.material.icons.filled.VideoLibrary
import androidx.compose.material3.Button
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.material3.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import coil.compose.AsyncImage
import com.ute.techdash.utils.PermisosHelper

// Llamamos ItemMedia (no MediaItem) para evitar conflicto de nombre con
// androidx.media3.common.MediaItem que se importa en ReproductorVideo.kt
sealed class ItemMedia {
    data class Imagen(val uri: Uri) : ItemMedia()
    data class Video(val uri: Uri)  : ItemMedia()
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PantallaMultimedia() {
    val context = LocalContext.current

    var mostrarCamara   by remember { mutableStateOf(false) }
    var mediaSeleccionado by remember { mutableStateOf<ItemMedia?>(null) }
    var galeriaItems    by remember { mutableStateOf<List<ItemMedia>>(emptyList()) }

    // Permisos necesarios
    var permisoCamara by remember {
        mutableStateOf(
            PermisosHelper.tienePermiso(context, android.Manifest.permission.CAMERA)
        )
    }
    val launcherPermisoCamara = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { permisoCamara = it }

    // PhotoPicker — imágenes
    val launcherImagen = rememberLauncherForActivityResult(
        ActivityResultContracts.PickMultipleVisualMedia(maxItems = 10)
    ) { uris ->
        galeriaItems = galeriaItems + uris.map { ItemMedia.Imagen(it) }
    }

    // PhotoPicker — videos
    val launcherVideo = rememberLauncherForActivityResult(
        ActivityResultContracts.PickVisualMedia()
    ) { uri ->
        uri?.let { galeriaItems = galeriaItems + ItemMedia.Video(it) }
    }

    // Si se está mostrando la cámara, ocupa toda la pantalla
    if (mostrarCamara) {
        PantallaCamara(
            onFotoTomada = { uri ->
                galeriaItems    = galeriaItems + ItemMedia.Imagen(uri)
                mediaSeleccionado = ItemMedia.Imagen(uri)
                mostrarCamara   = false
            },
            onCerrar = { mostrarCamara = false }
        )
        return
    }

    // Vista principal
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Multimedia", fontWeight = FontWeight.Bold) }
            )
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .padding(padding)
                .fillMaxSize()
        ) {
            // Botones de acción
            Row(
                Modifier
                    .fillMaxWidth()
                    .padding(16.dp),
                horizontalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                // Cámara
                Button(
                    onClick = {
                        if (permisoCamara) mostrarCamara = true
                        else launcherPermisoCamara.launch(
                            android.Manifest.permission.CAMERA
                        )
                    },
                    modifier = Modifier.weight(1f)
                ) {
                    Icon(Icons.Default.CameraAlt, null)
                    Spacer(Modifier.width(4.dp))
                    Text("Cámara")
                }

                // Galería imágenes
                OutlinedButton(
                    onClick  = {
                        launcherImagen.launch(
                            PickVisualMediaRequest(
                                ActivityResultContracts.PickVisualMedia.ImageOnly
                            )
                        )
                    },
                    modifier = Modifier.weight(1f)
                ) {
                    Icon(Icons.Default.PhotoLibrary, null)
                    Spacer(Modifier.width(4.dp))
                    Text("Galería")
                }

                // Video
                OutlinedButton(
                    onClick  = {
                        launcherVideo.launch(
                            PickVisualMediaRequest(
                                ActivityResultContracts.PickVisualMedia.VideoOnly
                            )
                        )
                    },
                    modifier = Modifier.weight(1f)
                ) {
                    Icon(Icons.Default.VideoLibrary, null)
                    Spacer(Modifier.width(4.dp))
                    Text("Video")
                }
            }

            // Visor del item seleccionado
            AnimatedVisibility(visible = mediaSeleccionado != null) {
                Box(
                    modifier = Modifier
                        .fillMaxWidth()
                        .height(260.dp)
                        .padding(horizontal = 16.dp)
                        .clip(RoundedCornerShape(16.dp))
                ) {
                    when (val item = mediaSeleccionado) {
                        is ItemMedia.Imagen -> VisorImagen(
                            uri      = item.uri,
                            modifier = Modifier.fillMaxSize()
                        )
                        is ItemMedia.Video  -> ReproductorVideo(
                            uri      = item.uri,
                            modifier = Modifier.fillMaxSize()
                        )
                        null -> {}
                    }

                    // Botón cerrar visor
                    IconButton(
                        onClick  = { mediaSeleccionado = null },
                        modifier = Modifier
                            .align(Alignment.TopEnd)
                            .padding(4.dp)
                            .size(36.dp)
                            .clip(CircleShape)
                            .background(Color.Black.copy(alpha = 0.5f))
                    ) {
                        Icon(Icons.Default.Close, "Cerrar", tint = Color.White, modifier = Modifier.size(18.dp))
                    }
                }
            }

            Spacer(Modifier.height(8.dp))

            // Grid de miniaturas
            if (galeriaItems.isEmpty()) {
                Box(
                    modifier         = Modifier.fillMaxSize(),
                    contentAlignment = Alignment.Center
                ) {
                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                        Icon(
                            Icons.Default.PermMedia,
                            contentDescription = null,
                            modifier = Modifier.size(64.dp),
                            tint     = MaterialTheme.colorScheme.onSurfaceVariant
                        )
                        Spacer(Modifier.height(8.dp))
                        Text(
                            "Toma una foto o selecciona de la galería",
                            color = MaterialTheme.colorScheme.onSurfaceVariant,
                            style = MaterialTheme.typography.bodyMedium
                        )
                    }
                }
            } else {
                LazyVerticalGrid(
                    columns         = GridCells.Fixed(3),
                    modifier        = Modifier.fillMaxSize(),
                    contentPadding  = PaddingValues(horizontal = 16.dp),
                    horizontalArrangement = Arrangement.spacedBy(4.dp),
                    verticalArrangement   = Arrangement.spacedBy(4.dp)
                ) {
                    items(galeriaItems) { item ->
                        Box(
                            modifier = Modifier
                                .aspectRatio(1f)
                                .clip(RoundedCornerShape(8.dp))
                                .clickable { mediaSeleccionado = item }
                        ) {
                            when (item) {
                                is ItemMedia.Imagen -> AsyncImage(
                                    model              = item.uri,
                                    contentDescription = null,
                                    contentScale       = ContentScale.Crop,
                                    modifier           = Modifier.fillMaxSize()
                                )
                                is ItemMedia.Video  -> {
                                    Box(
                                        Modifier
                                            .fillMaxSize()
                                            .background(Color.Black),
                                        contentAlignment = Alignment.Center
                                    ) {
                                        Icon(
                                            Icons.Default.PlayCircle,
                                            contentDescription = "Video",
                                            tint     = Color.White,
                                            modifier = Modifier.size(40.dp)
                                        )
                                    }
                                }
                            }

                            // Indicador de seleccionado
                            if (mediaSeleccionado == item) {
                                Box(
                                    modifier = Modifier
                                        .fillMaxSize()
                                        .border(
                                            3.dp,
                                            MaterialTheme.colorScheme.primary,
                                            RoundedCornerShape(8.dp)
                                        )
                                )
                            }
                        }
                    }
                }
            }
        }
    }
}
```

---

## Ejercicios propuestos

1. **Guardar foto permanentemente** — Modifica `PantallaCamara` para que
   al tomar la foto, además de devolverla como URI, la guarde en la carpeta
   `Pictures` del almacenamiento externo usando `MediaStore.Images.Media`
   para que aparezca en la galería del sistema.

2. **Miniatura de video** — Usa `MediaMetadataRetriever` para extraer el
   primer fotograma de un video seleccionado con el `PhotoPicker` y
   mostrarlo como miniatura en lugar del ícono de play. Convierte el
   `Bitmap` a `ImageBitmap` con `asImageBitmap()` para mostrarlo en Compose.

3. **Controles con gestos** — Añade al `ReproductorVideo` la capacidad
   de hacer doble tap para avanzar/retroceder 10 segundos (derecha/izquierda)
   usando `detectTapGestures` del módulo `pointerInput` de Compose.

4. **Grid editable** — Añade a `PantallaMultimedia` la posibilidad de
   eliminar elementos del grid con un long press que muestre un `AlertDialog`
   de confirmación. Usa `combinedClickable` de `foundation` para detectar
   tanto el tap normal (abrir) como el long press (eliminar).

---

## Resumen de la página 17

- CameraX abstrae las diferencias entre fabricantes con una API unificada. `Preview`, `ImageCapture` y `VideoCapture` son los tres casos de uso principales.
- `ProcessCameraProvider.bindToLifecycle` vincula la cámara al ciclo de vida — se libera automáticamente en `onStop` sin código manual.
- `AndroidView` es el puente entre componentes Android clásicos (como `PreviewView`) y Compose. El lambda `update` se llama en cada recomposición.
- `PhotoPicker` (`PickVisualMedia`) no requiere permisos de lectura en Android 13+. Es la forma recomendada de acceder a la galería.
- `DisposableEffect` es el mecanismo de Compose para registrar/desregistrar recursos con ciclo de vida — ideal para el `ExoPlayer` y los observers.
- El `ExoPlayer` **debe liberarse** en `DisposableEffect { onDispose { reproductor.release() } }` para evitar fugas de memoria.
- `TextureView` dentro de `AndroidView` es la forma más compatible de renderizar video de ExoPlayer en Compose; evita el problema de z-order de `SurfaceView` en Android 9–11 (`reproductor.setVideoTextureView(tv)`).
- `LazyVerticalGrid` con `GridCells.Fixed(n)` crea grids de columnas fijas. `aspectRatio(1f)` garantiza celdas cuadradas.

---

> **Siguiente página →** Página 18: GPS, sensores del dispositivo
> y monitoreo de conectividad de red.