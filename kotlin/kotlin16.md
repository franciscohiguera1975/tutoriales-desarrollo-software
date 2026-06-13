# Curso Kotlin — Página 16
## Módulo 7 · Hardware y Sistema
### Ciclo de vida, Activity, Fragment y permisos en runtime

---

## 🏗️ Configuración inicial — crea el proyecto una sola vez

> ⚠️ **Un solo proyecto para todo el módulo.** Las páginas 16 a 21 construyen la misma app **TechDash**. Créalo aquí una sola vez; nunca lo borres entre páginas.

**Pasos en Android Studio:**

1. `File → New → New Project`
2. Plantilla: **Empty Activity** (con Jetpack Compose)
3. Configura:
   - **Name:** `TechDash`
   - **Package name:** `com.ute.techdash`
   - **Language:** Kotlin
   - **Minimum SDK:** API 26 (Android 8.0)
4. Pulsa **Finish** y espera a que Gradle sincronice

**Estructura de paquetes que irás creando a lo largo del módulo:**

```
app/src/main/java/com/example/techdash/
│
├── MainActivity.kt                       ← ya existe, lo modificarás
├── utils/                                ← créalo en esta página (p16)
│   └── PermisosHelper.kt
├── ui/
│   ├── permisos/                          ← p16
│   │   └── PantallaPermisos.kt
│   ├── multimedia/                        ← p17
│   │   ├── PantallaCamara.kt
│   │   ├── SelectorGaleria.kt
│   │   ├── VisorImagen.kt
│   │   └── ReproductorVideo.kt
│   ├── hardware/                          ← p18
│   │   ├── gps/
│   │   │   ├── LocationRepository.kt
│   │   │   ├── UbicacionViewModel.kt
│   │   │   └── PantallaGPS.kt
│   │   ├── sensores/
│   │   │   ├── SensoresRepository.kt
│   │   │   ├── SensoresViewModel.kt
│   │   │   └── PantallaSensores.kt
│   │   └── red/
│   │       ├── ConectividadRepository.kt
│   │       └── BannerConectividad.kt
│   └── media/                             ← p21
│       ├── ReproductorVideoUrl.kt
│       ├── ReproductorAudio.kt
│       └── PantallaStreaming.kt
└── navigation/
    └── NavGraph.kt                        ← lo actualizas al final de cada página
```

**Referencia rápida — acciones en Android Studio:**

| ¿Qué necesito? | Pasos |
|----------------|-------|
| **Crear un paquete nuevo** | Clic derecho en `techdash` → **New → Package** → escribe p. ej. `ui.permisos` |
| **Crear un archivo Kotlin** | Clic derecho en el paquete → **New → Kotlin Class/File → File** → nombre sin `.kt` |
| **Añadir permiso** | Abre `app/src/main/AndroidManifest.xml` → añade `<uses-permission .../>` dentro de `<manifest>` |
| **Añadir dependencia** | Abre `app/build.gradle.kts` → añade dentro de `dependencies { }` → sincroniza con **Sync Now** |

---

### Dependencias necesarias para esta página

Abre `app/build.gradle.kts` y verifica que estén estas líneas dentro de `dependencies { }`:

```kotlin
// Material Icons (necesario para Icons.Default.CameraAlt, Icons.Default.NoPhotography, etc.)
implementation("androidx.compose.material:material-icons-extended")
```

Luego haz **Sync Now** en la barra amarilla que aparece.

---

## 🚀 Proyecto TechDash — Fundamentos del módulo

A lo largo de este módulo construiremos **TechDash**, una app de dashboard que accede al hardware del dispositivo en tiempo real. Esta página establece los cimientos del proyecto:

```
TechDash
├── 🔐 Ciclo de vida y permisos  ← Esta página (p16)
├── 📷 Cámara y Galería          ← Página 17
├── 📍 GPS, Sensores, Red        ← Página 18
├── 💾 DataStore + Room          ← Páginas 19-20
└── 🎵 Media y Streaming         ← Página 21
```

Sin entender el ciclo de vida no sabrás cuándo liberar la cámara, el GPS o los sensores — y sin el sistema de permisos, ninguna de esas funciones podrá activarse. Aquí construyes el `PermisosHelper` y la `PantallaPermisos` que usarás en todas las páginas siguientes.

### Archivos que crearás/modificarás en esta página

| | Archivo | Dónde crearlo |
|---|---------|---------------|
| 📄 Nuevo | `PermisosHelper.kt` | paquete `utils/` |
| 📄 Nuevo | `PantallaConPermiso.kt` | paquete `ui/permisos/` |
| 📄 Nuevo | `PantallaPermisos.kt` | paquete `ui/permisos/` |
| ✏️ Modifica | `AndroidManifest.xml` | `app/src/main/` — añade los `<uses-permission>` |
| ✏️ Modifica | `MainActivity.kt` | raíz del paquete — cambia el `setContent` |

---

## ¿Por qué es importante el ciclo de vida?

Hasta ahora hemos trabajado con `ViewModel` y `Compose` que abstraen
gran parte del ciclo de vida. Sin embargo, comprender qué ocurre
**debajo** es esencial para:

- Saber cuándo liberar la cámara, el GPS o los sensores
- Evitar fugas de memoria (memory leaks)
- Manejar correctamente los permisos de runtime
- Depurar comportamientos inesperados al rotar la pantalla

---

## Ciclo de vida de una `Activity`

```
                    ┌─────────────────────────────────┐
                    │           ACTIVITY               │
                    │                                 │
  Inicio ──────────►│  onCreate()                     │
                    │      ↓                          │
                    │  onStart()     ←── Vuelve del   │
                    │      ↓              background  │
                    │  onResume()    ←── Vuelve al    │
                    │      ↓              primer plano │
                    │  [ACTIVA — usuario interactúa]  │
                    │      ↓                          │
                    │  onPause()     ──► Otra actividad│
                    │      ↓              parcial     │
                    │  onStop()      ──► App al fondo  │
                    │      ↓                          │
                    │  onDestroy()   ──► Fin / rotación│
                    └─────────────────────────────────┘
```

📁 **`MainActivity.kt`** — ciclo de vida completo con Log:

```kotlin
package com.ute.techdash

import android.os.Bundle
import android.util.Log
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.Text
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {

    private val TAG = "CicloVida"

    // Se llama al CREAR la actividad — o tras rotación
    // Aquí: inflar la UI, inicializar el ViewModel
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        Log.d(TAG, "onCreate — savedInstanceState: ${savedInstanceState != null}")

        // Restaurar estado si la actividad fue recreada
        val textoGuardado = savedInstanceState?.getString("texto_usuario")
        Log.d(TAG, "Texto restaurado: $textoGuardado")

        setContent {
            TechDashTheme {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text("Rota la pantalla y observa Logcat")
                }
            }
        }
    }

    // La actividad se vuelve VISIBLE
    override fun onStart() {
        super.onStart()
        Log.d(TAG, "onStart — actividad visible")
    }

    // La actividad pasa al PRIMER PLANO — usuario puede interactuar
    // Aquí: reanudar animaciones, cámara, GPS, sensores
    override fun onResume() {
        super.onResume()
        Log.d(TAG, "onResume — primer plano")
    }

    // Otra actividad toma el foco PARCIALMENTE (diálogo, permiso)
    // Aquí: pausar animaciones, liberar cámara si exclusiva
    override fun onPause() {
        super.onPause()
        Log.d(TAG, "onPause — foco parcialmente perdido")
    }

    // La actividad ya NO es visible (app al fondo, otra app encima)
    // Aquí: guardar datos, detener operaciones costosas
    override fun onStop() {
        super.onStop()
        Log.d(TAG, "onStop — actividad no visible")
    }

    // Guardar estado ANTES de que la actividad sea destruida
    // Se llama antes de onStop — sobrevive a rotaciones
    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        outState.putString("texto_usuario", "dato importante")
        Log.d(TAG, "onSaveInstanceState — guardando estado")
    }

    // La actividad es DESTRUIDA definitivamente
    // Aquí: liberar recursos finales (no el ViewModel — él sobrevive)
    override fun onDestroy() {
        super.onDestroy()
        Log.d(TAG, "onDestroy — actividad destruida")
    }
}
```

### ¿Cuándo se llama cada método?

| Situación | Métodos llamados |
|---|---|
| App se abre | `onCreate` → `onStart` → `onResume` |
| Pantalla se rota | `onPause` → `onStop` → `onSaveInstanceState` → `onDestroy` → `onCreate` → `onStart` → `onResume` |
| Otra app toma el foco | `onPause` → `onStop` |
| Usuario vuelve a la app | `onStart` → `onResume` |
| Diálogo del sistema aparece | `onPause` |
| Usuario presiona Atrás | `onPause` → `onStop` → `onDestroy` |
| Usuario presiona Inicio | `onPause` → `onStop` |

---

## `ViewModel` y el ciclo de vida

El `ViewModel` **sobrevive a las rotaciones** porque su ciclo de vida
está ligado al scope de la pantalla, no a la instancia de la `Activity`:

```kotlin
// El ViewModel NO se destruye al rotar la pantalla
// Solo se destruye cuando el usuario sale definitivamente de la pantalla

class MiViewModel : ViewModel() {
    var contador = 0   // ← sobrevive a la rotación

    // Se llama cuando el ViewModel se destruye definitivamente
    override fun onCleared() {
        super.onCleared()
        // Aquí liberar recursos del ViewModel (coroutines externas, etc.)
        // viewModelScope se cancela automáticamente — no hace falta cancelarlo aquí
        println("ViewModel destruido")
    }
}

// En el Composable:
@Composable
fun MiPantalla(vm: MiViewModel = viewModel()) {
    // vm.contador sobrevive a la rotación gracias al ViewModel
    // rememberSaveable sobrevive también, pero solo para tipos primitivos
}
```

---

### ▶ Checkpoint 1 — Observa el ciclo de vida en Logcat

1. Añade los `Log.d` del código anterior en tu `MainActivity`
2. Ejecuta la app en emulador o dispositivo
3. Abre **Logcat** en Android Studio, filtra por el tag `CicloVida`

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código:

```kotlin
package com.ute.techdash

import android.os.Bundle
import android.util.Log
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.Text
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        Log.d("CicloVida", "onCreate")
        setContent {
            TechDashTheme {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Text("Rota la pantalla y observa Logcat")
                }
            }
        }
    }
    override fun onStart()   { super.onStart();   Log.d("CicloVida", "onStart") }
    override fun onResume()  { super.onResume();  Log.d("CicloVida", "onResume") }
    override fun onPause()   { super.onPause();   Log.d("CicloVida", "onPause") }
    override fun onStop()    { super.onStop();    Log.d("CicloVida", "onStop") }
    override fun onDestroy() { super.onDestroy(); Log.d("CicloVida", "onDestroy") }
    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        Log.d("CicloVida", "onSaveInstanceState")
    }
}
```

**Realiza estas acciones y anota el orden de métodos:**
- Abrir la app → `onCreate → onStart → onResume`
- Rotar la pantalla → `onPause → onStop → onSaveInstanceState → onDestroy → onCreate → onStart → onResume`
- Pulsar el botón de Inicio → `onPause → onStop`
- Volver a la app → `onStart → onResume`

> 💡 Logcat está en la parte inferior de Android Studio: `View → Tool Windows → Logcat` o atajo `Alt+6`. Filtra escribiendo `CicloVida` en la barra de búsqueda.

---

> ### 🧪 Prueba esto — `remember` vs `ViewModel` al rotar
>
> **Reto 1:** Crea una pantalla con dos contadores: uno en `remember { mutableIntStateOf(0) }` y otro en un `ViewModel`. Añade botones para incrementar cada uno. Rota el dispositivo — ¿cuál se reinicia?
>
> **Reto 2:** Cambia el `remember` por `rememberSaveable`. Rota de nuevo. ¿Qué diferencia hay? ¿Por qué `rememberSaveable` funciona para `Int` pero puede fallar para un `data class` personalizado?
>
> **Reflexión:** Logcat muestra que al rotar se llama `onDestroy`. ¿Esto significa que el `ViewModel` también se destruye? Verifica añadiendo `Log.d("CicloVida", "ViewModel destruido")` dentro del `onCleared()` del ViewModel y rotando — ¿aparece el log?

---

## Ciclo de vida de un `Fragment`

Los `Fragment` tienen su propio ciclo de vida, anidado dentro del
de la `Activity`. En proyectos con Compose puro los `Fragment` se
usan menos, pero aparecen en proyectos legacy o al mezclar con XML:

```kotlin
import androidx.fragment.app.Fragment
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup

class MiFragment : Fragment() {

    // El Fragment es ADJUNTADO a la Activity
    override fun onAttach(context: android.content.Context) {
        super.onAttach(context)
        // context es la Activity — disponible desde aquí
    }

    // Se crea la VIEW del Fragment
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        // Inflar layout XML — en proyectos Compose se usa ComposeView
        return inflater.inflate(R.layout.fragment_mi, container, false)
    }

    // La VIEW está lista — aquí se inicializan los componentes visuales
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        // Configurar listeners, observar LiveData/StateFlow
    }

    // El Fragment es VISIBLE — equivale a onResume de Activity
    override fun onResume() { super.onResume() }

    // El Fragment deja de ser VISIBLE
    override fun onPause()  { super.onPause() }

    // La VIEW es destruida (Fragment se oculta pero sigue adjunto)
    override fun onDestroyView() {
        super.onDestroyView()
        // Liberar referencias a views para evitar memory leaks
    }

    // El Fragment se separa de la Activity
    override fun onDetach() { super.onDetach() }
}
```

---

## Permisos en Android

Android divide los permisos en dos categorías:

```
Permisos normales                    Permisos peligrosos
─────────────────                    ───────────────────────────────────
Se conceden automáticamente          El usuario debe aprobarlos en runtime
al instalar la app                   con un diálogo del sistema

INTERNET                             CAMERA
ACCESS_NETWORK_STATE                 ACCESS_FINE_LOCATION
VIBRATE                              ACCESS_COARSE_LOCATION
                                     READ_MEDIA_IMAGES
                                     READ_MEDIA_VIDEO
                                     RECORD_AUDIO
```

### `AndroidManifest.xml` — declarar permisos

```xml
<!-- AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Permisos normales — se conceden automáticamente -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.VIBRATE" />

    <!-- Permisos peligrosos — requieren diálogo en runtime -->
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />

    <!-- Android 13+ — acceso granular a medios -->
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
    <uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
    <uses-permission android:name="android.permission.READ_MEDIA_AUDIO" />

    <application ... >
        ...
    </application>
</manifest>
```

---

## Solicitar permisos en runtime con `ActivityResultContracts`

📁 **Nuevo archivo → `ui/permisos/PantallaConPermiso.kt`**

```kotlin
package com.ute.techdash.ui.permisos

import android.content.pm.PackageManager
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.CameraAlt
import androidx.compose.material.icons.filled.NoPhotography
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.Button
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.core.content.ContextCompat

@Composable
fun PantallaConPermiso() {
    val context = LocalContext.current

    // Estado del permiso
    var permisoOtorgado by remember {
        mutableStateOf(
            ContextCompat.checkSelfPermission(
                context,
                android.Manifest.permission.CAMERA
            ) == PackageManager.PERMISSION_GRANTED
        )
    }
    var mostrarRacional by remember { mutableStateOf(false) }

    // Launcher para solicitar UN permiso
    val lanzadorPermiso = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestPermission()
    ) { concedido ->
        permisoOtorgado = concedido
        if (!concedido) mostrarRacional = true
    }

    Column(
        Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement   = Arrangement.Center,
        horizontalAlignment   = Alignment.CenterHorizontally
    ) {
        if (permisoOtorgado) {
            Icon(
                Icons.Default.CameraAlt,
                contentDescription = null,
                modifier = Modifier.size(64.dp),
                tint     = MaterialTheme.colorScheme.primary
            )
            Spacer(Modifier.height(16.dp))
            Text("Permiso de cámara concedido ✅")
            Spacer(Modifier.height(8.dp))
            Text(
                "Aquí iría el componente de cámara",
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        } else {
            Icon(
                Icons.Default.NoPhotography,
                contentDescription = null,
                modifier = Modifier.size(64.dp),
                tint     = MaterialTheme.colorScheme.error
            )
            Spacer(Modifier.height(16.dp))
            Text("Se necesita acceso a la cámara")
            Spacer(Modifier.height(8.dp))
            Button(onClick = {
                lanzadorPermiso.launch(android.Manifest.permission.CAMERA)
            }) {
                Text("Solicitar permiso")
            }
        }

        // Diálogo explicativo cuando el usuario deniega
        if (mostrarRacional) {
            AlertDialog(
                onDismissRequest = { mostrarRacional = false },
                title = { Text("Permiso necesario") },
                text  = {
                    Text(
                        "La cámara es necesaria para tomar fotos. " +
                        "Por favor, concede el permiso en los ajustes de la app."
                    )
                },
                confirmButton = {
                    Button(onClick = {
                        mostrarRacional = false
                        // Abrir ajustes de la app
                        val intent = android.content.Intent(
                            android.provider.Settings.ACTION_APPLICATION_DETAILS_SETTINGS,
                            android.net.Uri.fromParts("package", context.packageName, null)
                        )
                        context.startActivity(intent)
                    }) { Text("Ir a ajustes") }
                },
                dismissButton = {
                    OutlinedButton(onClick = { mostrarRacional = false }) {
                        Text("Cancelar")
                    }
                }
            )
        }
    }
}
```

---

### ▶ Checkpoint 2 — `PantallaConPermiso`: permiso de cámara en tiempo real

**Pasos para ejecutar:**

1. Abre `app/src/main/AndroidManifest.xml` y añade dentro de `<manifest>`, **antes** de `<application>`:
   ```xml
   <uses-permission android:name="android.permission.CAMERA" />
   ```
2. Crea el paquete `ui/permisos/` (clic derecho sobre `techdash` → **New → Package** → escribe `ui.permisos`)
3. Crea `PantallaConPermiso.kt` en ese paquete y pega el código de arriba
4. Reemplaza `MainActivity.kt` con el código de abajo
5. Ejecuta con **▶** (no ⚡) — haz **Build → Clean Project** primero si algo falla

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código:

```kotlin
package com.ute.techdash

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.ute.techdash.ui.permisos.PantallaConPermiso
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                PantallaConPermiso()
            }
        }
    }
}
```

**Escenarios que debes probar en orden:**

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Abrir la app por primera vez | Diálogo del sistema: "¿Permitir a TechDash tomar fotos?" |
| 2 | Pulsar **Permitir** | Ícono de cámara verde + texto "Permiso de cámara concedido ✅" |
| 3 | Resetear permiso (ver abajo) y abrir de nuevo | El diálogo del sistema aparece nuevamente |
| 4 | Pulsar **Denegar** | `AlertDialog` de la app: "Permiso necesario" + botones "Ir a ajustes" / "Cancelar" |
| 5 | Pulsar **Cancelar** en el AlertDialog | El AlertDialog se cierra, vuelve el botón "Solicitar permiso" |
| 6 | Pulsar **Solicitar permiso** de nuevo | El diálogo del sistema aparece por segunda vez |
| 7 | Volver a pulsar **Denegar** | Android ya **no muestra más** el diálogo — este es el "segundo rechazo" |

> 💡 **Cómo resetear el permiso para repetir la prueba:**
> Mantén presionado el ícono de la app → **Información de la app → Permisos → Cámara → Denegar → Aceptar**
>
> En el **emulador:** ve a **Settings → Apps → TechDash → Permissions → Camera → Don't allow**.

---

> ### 🧪 Prueba esto — El segundo rechazo
>
> Tras el paso 7 de la tabla, el botón "Solicitar permiso" ya no lanza el diálogo del sistema. Actualmente la app no hace nada en ese caso.
>
> **Reto:** Detecta cuándo estás ante el segundo rechazo. `ActivityCompat.shouldShowRequestPermissionRationale(activity, permiso)` devuelve `true` después del primer rechazo y `false` después del segundo. Muestra automáticamente el `AlertDialog` de "Ir a ajustes" cuando el usuario pulsa "Solicitar permiso" y ya no hay diálogo disponible.
>
> **Pista:** `shouldShowRequestPermissionRationale` también devuelve `false` *antes* del primer intento — diferéncialos con `rememberSaveable { mutableStateOf(false) }` que marcas `true` cuando el launcher se lanza por primera vez.
>
> **Reflexión:** En la página 18 el módulo GPS necesita el permiso de ubicación. ¿Qué es mejor: bloquear el acceso al módulo GPS si el permiso está denegado, o mostrar la pantalla con un mensaje de error explicativo? ¿Cómo afecta eso al `onTodosConcedidos` de `PantallaPermisos`?

---

## Solicitar múltiples permisos

> Este es un ejemplo educativo que muestra el mecanismo. La pantalla completa y lista para usar es `PantallaPermisos` más abajo.

```kotlin
import android.content.pm.PackageManager
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Cancel
import androidx.compose.material.icons.filled.CheckCircle
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.HorizontalDivider
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.core.content.ContextCompat

@Composable
fun SolicitarPermisosMultiples() {
    val context = LocalContext.current

    val permisos = arrayOf(
        android.Manifest.permission.CAMERA,
        android.Manifest.permission.ACCESS_FINE_LOCATION,
        android.Manifest.permission.RECORD_AUDIO
    )

    // Estado de cada permiso
    var estadoPermisos by remember {
        mutableStateOf(
            permisos.associateWith { permiso ->
                ContextCompat.checkSelfPermission(context, permiso) ==
                    PackageManager.PERMISSION_GRANTED
            }
        )
    }

    // Launcher para múltiples permisos
    val lanzador = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestMultiplePermissions()
    ) { resultados ->
        estadoPermisos = resultados
    }

    val nombresPermisos = mapOf(
        android.Manifest.permission.CAMERA             to "Cámara",
        android.Manifest.permission.ACCESS_FINE_LOCATION to "Ubicación precisa",
        android.Manifest.permission.RECORD_AUDIO        to "Micrófono"
    )

    Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Text("Estado de permisos", style = MaterialTheme.typography.titleLarge,
            fontWeight = FontWeight.Bold)

        estadoPermisos.forEach { (permiso, concedido) ->
            Row(
                Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment     = Alignment.CenterVertically
            ) {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Icon(
                        imageVector = if (concedido) Icons.Default.CheckCircle
                                      else Icons.Default.Cancel,
                        contentDescription = null,
                        tint = if (concedido) MaterialTheme.colorScheme.primary
                               else MaterialTheme.colorScheme.error,
                        modifier = Modifier.size(20.dp)
                    )
                    Spacer(Modifier.width(8.dp))
                    Text(nombresPermisos[permiso] ?: permiso.substringAfterLast("."))
                }
                Text(
                    if (concedido) "Concedido" else "Denegado",
                    style = MaterialTheme.typography.bodySmall,
                    color = if (concedido) MaterialTheme.colorScheme.primary
                            else MaterialTheme.colorScheme.error
                )
            }
            HorizontalDivider()
        }

        val todosConcedidos = estadoPermisos.values.all { it }
        if (!todosConcedidos) {
            Button(
                onClick  = { lanzador.launch(permisos) },
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Solicitar permisos pendientes")
            }
        } else {
            Card(
                colors   = CardDefaults.cardColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer
                ),
                modifier = Modifier.fillMaxWidth()
            ) {
                Text(
                    "✅ Todos los permisos concedidos",
                    modifier = Modifier.padding(16.dp),
                    fontWeight = FontWeight.SemiBold
                )
            }
        }
    }
}
```

---

## Helper reutilizable de permisos

📁 **Nuevo archivo → `utils/PermisosHelper.kt`**

```kotlin
package com.ute.techdash.utils

import android.content.Context
import android.content.pm.PackageManager
import androidx.core.content.ContextCompat

object PermisosHelper {

    fun tienePermiso(context: Context, permiso: String): Boolean =
        ContextCompat.checkSelfPermission(context, permiso) ==
            PackageManager.PERMISSION_GRANTED

    fun tieneTodosLosPermisos(context: Context, permisos: Array<String>): Boolean =
        permisos.all { tienePermiso(context, it) }

    fun permisosNecesarios(
        context:  Context,
        permisos: Array<String>
    ): Array<String> = permisos.filter { !tienePermiso(context, it) }.toTypedArray()
}
```

**Uso típico en un Composable** — añade esto dentro del archivo que necesite verificar un permiso:

```kotlin
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.platform.LocalContext
import com.ute.techdash.utils.PermisosHelper

@Composable
fun VerificarPermisoCamara(
    contenidoConPermiso:  @Composable () -> Unit,
    contenidoSinPermiso:  @Composable (onSolicitar: () -> Unit) -> Unit
) {
    val context = LocalContext.current
    var permisoConcedido by remember {
        mutableStateOf(
            PermisosHelper.tienePermiso(context, android.Manifest.permission.CAMERA)
        )
    }

    val launcher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { permisoConcedido = it }

    if (permisoConcedido) {
        contenidoConPermiso()
    } else {
        contenidoSinPermiso {
            launcher.launch(android.Manifest.permission.CAMERA)
        }
    }
}
```

---

## Programa completo — pantalla de configuración de permisos

📁 **Nuevo archivo → `ui/permisos/PantallaPermisos.kt`**

```kotlin
package com.ute.techdash.ui.permisos

import android.content.pm.PackageManager
import android.os.Build
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.animation.AnimatedVisibility
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.*
import androidx.compose.ui.draw.clip
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import com.ute.techdash.utils.PermisosHelper

data class ConfigPermiso(
    val permiso:      String,
    val nombre:       String,
    val descripcion:  String,
    val icono:        androidx.compose.ui.graphics.vector.ImageVector,
    val obligatorio:  Boolean = true
)

// READ_MEDIA_IMAGES existe solo en Android 13+ (API 33).
// En versiones anteriores se usa READ_EXTERNAL_STORAGE.
private val PERMISO_FOTOS =
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU)
        android.Manifest.permission.READ_MEDIA_IMAGES
    else
        android.Manifest.permission.READ_EXTERNAL_STORAGE

@Composable
fun PantallaPermisos(onTodosConcedidos: () -> Unit = {}) {
    val context = LocalContext.current

    val configuraciones = remember {
        listOf(
            ConfigPermiso(
                android.Manifest.permission.CAMERA,
                "Cámara",
                "Necesaria para tomar fotos y escanear códigos QR.",
                Icons.Default.CameraAlt,
                obligatorio = true
            ),
            ConfigPermiso(
                android.Manifest.permission.ACCESS_FINE_LOCATION,
                "Ubicación precisa",
                "Para mostrar tu posición en el mapa con GPS.",
                Icons.Default.LocationOn,
                obligatorio = true
            ),
            ConfigPermiso(
                android.Manifest.permission.RECORD_AUDIO,
                "Micrófono",
                "Para grabar notas de voz y videollamadas.",
                Icons.Default.Mic,
                obligatorio = false
            ),
            ConfigPermiso(
                PERMISO_FOTOS,                 // ← API 33+: READ_MEDIA_IMAGES / anterior: READ_EXTERNAL_STORAGE
                "Fotos",
                "Para adjuntar imágenes desde la galería.",
                Icons.Default.Photo,
                obligatorio = false
            )
        )
    }

    var estadoPermisos by remember {
        mutableStateOf(
            configuraciones.associate { cfg ->
                cfg.permiso to PermisosHelper.tienePermiso(context, cfg.permiso)
            }
        )
    }

    val lanzador = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { resultados ->
        estadoPermisos = estadoPermisos + resultados
    }

    val obligatoriosConcedidos = configuraciones
        .filter { it.obligatorio }
        .all { estadoPermisos[it.permiso] == true }

    LaunchedEffect(obligatoriosConcedidos) {
        if (obligatoriosConcedidos) onTodosConcedidos()
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Permisos de la app", fontWeight = FontWeight.Bold) }
            )
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .padding(padding)
                .fillMaxSize()
        ) {
            LazyColumn(
                modifier            = Modifier.weight(1f),
                contentPadding      = PaddingValues(16.dp),
                verticalArrangement = Arrangement.spacedBy(10.dp)
            ) {
                item {
                    Text(
                        "La app necesita los siguientes permisos para funcionar correctamente.",
                        style = MaterialTheme.typography.bodyMedium,
                        color = MaterialTheme.colorScheme.onSurfaceVariant,
                        modifier = Modifier.padding(bottom = 8.dp)
                    )
                }

                items(configuraciones) { cfg ->
                    val concedido = estadoPermisos[cfg.permiso] == true

                    ElevatedCard(
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Row(
                            Modifier.padding(16.dp),
                            verticalAlignment = Alignment.CenterVertically
                        ) {
                            // Icono con color según estado
                            Box(
                                modifier = Modifier
                                    .size(48.dp)
                                    .clip(CircleShape)
                                    .background(
                                        if (concedido)
                                            MaterialTheme.colorScheme.primaryContainer
                                        else
                                            MaterialTheme.colorScheme.surfaceVariant
                                    ),
                                contentAlignment = Alignment.Center
                            ) {
                                Icon(
                                    cfg.icono,
                                    contentDescription = null,
                                    tint = if (concedido)
                                        MaterialTheme.colorScheme.onPrimaryContainer
                                    else
                                        MaterialTheme.colorScheme.onSurfaceVariant
                                )
                            }

                            Spacer(Modifier.width(12.dp))

                            Column(modifier = Modifier.weight(1f)) {
                                Row(verticalAlignment = Alignment.CenterVertically) {
                                    Text(cfg.nombre, fontWeight = FontWeight.SemiBold)
                                    if (cfg.obligatorio) {
                                        Spacer(Modifier.width(4.dp))
                                        Text(
                                            "*",
                                            color = MaterialTheme.colorScheme.error,
                                            fontWeight = FontWeight.Bold
                                        )
                                    }
                                }
                                Text(
                                    cfg.descripcion,
                                    style = MaterialTheme.typography.bodySmall,
                                    color = MaterialTheme.colorScheme.onSurfaceVariant
                                )
                            }

                            Spacer(Modifier.width(8.dp))

                            // Estado visual
                            if (concedido) {
                                Icon(
                                    Icons.Default.CheckCircle,
                                    contentDescription = "Concedido",
                                    tint     = MaterialTheme.colorScheme.primary,
                                    modifier = Modifier.size(24.dp)
                                )
                            } else {
                                OutlinedButton(
                                    onClick = {
                                        lanzador.launch(arrayOf(cfg.permiso))
                                    },
                                    contentPadding = PaddingValues(horizontal = 12.dp, vertical = 4.dp)
                                ) {
                                    Text("Permitir", style = MaterialTheme.typography.labelSmall)
                                }
                            }
                        }
                    }
                }
            }

            // Panel inferior — acción principal
            Surface(
                tonalElevation = 3.dp,
                modifier       = Modifier.fillMaxWidth()
            ) {
                Column(Modifier.padding(16.dp)) {
                    Text(
                        "* Permisos obligatorios",
                        style = MaterialTheme.typography.labelSmall,
                        color = MaterialTheme.colorScheme.error
                    )
                    Spacer(Modifier.height(8.dp))

                    val pendientes = configuraciones.filter {
                        estadoPermisos[it.permiso] != true
                    }

                    AnimatedVisibility(visible = pendientes.isNotEmpty()) {
                        Button(
                            onClick  = {
                                lanzador.launch(
                                    pendientes.map { it.permiso }.toTypedArray()
                                )
                            },
                            modifier = Modifier.fillMaxWidth().height(50.dp)
                        ) {
                            Text("Conceder ${pendientes.size} permiso(s) pendiente(s)")
                        }
                    }

                    AnimatedVisibility(visible = obligatoriosConcedidos) {
                        Button(
                            onClick  = onTodosConcedidos,
                            modifier = Modifier.fillMaxWidth().height(50.dp)
                        ) {
                            Text("Continuar a la app →")
                        }
                    }
                }
            }
        }
    }
}
```

---

### ▶ Checkpoint 3 — `PantallaPermisos`: gestión de múltiples permisos

**Pasos para ejecutar:**

1. Crea el paquete `utils/` si no existe (clic derecho sobre `techdash` → **New → Package** → escribe `utils`)
2. Crea `PermisosHelper.kt` en `utils/` con el código de la sección anterior
3. Crea `PantallaPermisos.kt` en `ui/permisos/` con el código de arriba
4. Añade en `AndroidManifest.xml` los permisos que usa la pantalla:
   ```xml
   <uses-permission android:name="android.permission.CAMERA" />
   <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
   <uses-permission android:name="android.permission.RECORD_AUDIO" />

   <!-- Fotos: Android 13+ usa READ_MEDIA_IMAGES, versiones anteriores READ_EXTERNAL_STORAGE -->
   <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
   <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
       android:maxSdkVersion="32" />
   ```
5. Reemplaza `MainActivity.kt` con el código de abajo
6. Ejecuta con **▶** en un dispositivo físico o emulador

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código:

```kotlin
package com.ute.techdash

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.ute.techdash.ui.permisos.PantallaPermisos
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                PantallaPermisos(
                    onTodosConcedidos = { /* en páginas siguientes aquí irá navController.navigate("dashboard") */ }
                )
            }
        }
    }
}
```

**Escenarios que debes probar en orden:**

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Abrir la app | Lista de 4 tarjetas — Cámara y Ubicación con `*` (obligatorios), Micrófono y Fotos sin `*` |
| 2 | Pulsar **Permitir** en la tarjeta "Cámara" | Solo aparece el diálogo de Cámara → al aceptar, el ícono de esa tarjeta cambia a ✅ verde |
| 3 | Pulsar **Permitir** en "Ubicación precisa" | Diálogo de ubicación → al aceptar, ✅ verde en esa tarjeta |
| 4 | Verificar el botón inferior | Con los 2 obligatorios concedidos → aparece con animación el botón **"Continuar a la app →"** |
| 5 | Pulsar **"Conceder X permiso(s) pendiente(s)"** (si no usaste el paso 2-3) | El sistema lanza los diálogos restantes en secuencia |
| 6 | Dejar Micrófono y Fotos sin conceder | El botón **"Continuar"** aparece igual — son opcionales |
| 7 | Pulsar **"Continuar a la app →"** | Se llama a `onTodosConcedidos` — por ahora no hace nada visible |

> 💡 **Diferencia entre obligatorio y opcional en el código:**
> ```kotlin
> ConfigPermiso(..., obligatorio = true)   // bloquea "Continuar" hasta que se conceda
> ConfigPermiso(..., obligatorio = false)  // permite continuar aunque esté denegado
> ```
>
> **Resetear todos los permisos:** Mantén presionado el ícono de la app → **Información de la app → Permisos** → deniega cada uno uno a uno.

---

> ### 🧪 Prueba esto — Estado que no se actualiza
>
> `PantallaPermisos` lee el estado de los permisos al montarse (dentro de `remember { ... }`). Sal de la app con el botón de inicio, ve a **Ajustes → Apps → TechDash → Permisos** y concede el Micrófono manualmente. Vuelve a la app.
>
> **Reto 1:** ¿Se actualiza la tarjeta de Micrófono automáticamente? ¿Por qué no? Añade un `LaunchedEffect` que vuelva a leer los permisos cuando la pantalla regresa al primer plano. Pista: observa `LocalLifecycleOwner.current.lifecycle` con `DisposableEffect` para detectar el evento `ON_RESUME`.
>
> **Reto 2:** El botón individual "Permitir" llama a `lanzador.launch(arrayOf(cfg.permiso))`. Esto lanza el launcher con **un** permiso. El botón "Conceder X pendiente(s)" lanza **todos** los pendientes a la vez. Prueba ambos caminos. ¿Cuál da mejor experiencia de usuario? ¿Por qué el lanzador recibe un `Array<String>` aunque solo tenga un elemento?
>
> **Reflexión:** `PantallaPermisos` está pensada como pantalla de bienvenida que bloquea el acceso hasta tener los permisos críticos. ¿Qué pasa si el usuario toca "Atrás" en esta pantalla? ¿La app debería cerrase, o simplemente volver a mostrar la misma pantalla cuando el usuario intente acceder a una función que necesita esos permisos?

---

## Ejercicios propuestos

1. **Observar el ciclo de vida** — Añade `Log.d` en todos los métodos del
   ciclo de vida de `MainActivity`. Ejecuta en un emulador, rota la pantalla,
   minimiza y vuelve a la app. Anota el orden exacto de llamadas en cada
   escenario en un cuadro comparativo.

2. **Estado que sobrevive a la rotación** — Crea una pantalla con un
   `TextField` para escribir texto. Usa primero `remember` y rota — el texto
   se pierde. Cambia a `rememberSaveable` y vuelve a rotar. Explica por qué
   sobrevive esta vez pero `remember` no.

3. **Permiso con racional** — Implementa `VerificarPermisoCamara` del
   helper. Si el usuario deniega el permiso por segunda vez, muestra un
   `AlertDialog` con botón "Ir a ajustes" que abra la pantalla de permisos
   de la app en los ajustes del sistema usando `Settings.ACTION_APPLICATION_DETAILS_SETTINGS`.

4. **ViewModel + ciclo de vida** — Crea un `CronometroViewModel` con un
   `StateFlow<Int>` que cuente segundos usando `viewModelScope.launch`.
   Verifica que el contador NO se reinicia al rotar la pantalla pero SÍ
   se detiene cuando el usuario sale definitivamente de la pantalla.

---

## Resumen de la página 16

- El ciclo de vida de una `Activity` define cuándo la app está activa, visible, en segundo plano o destruida. `onCreate` es para inicializar; `onResume`/`onPause` para recursos exclusivos como la cámara.
- `onSaveInstanceState` guarda estado antes de una rotación o muerte del proceso. Se restaura en `onCreate` a través del `Bundle`.
- El `ViewModel` sobrevive a las rotaciones — su ciclo de vida está ligado al scope de la pantalla, no a la instancia de `Activity`.
- Los `Fragment` tienen su propio ciclo de vida anidado. `onViewCreated` es el lugar correcto para inicializar componentes visuales.
- Los permisos **normales** se conceden automáticamente al instalar. Los **peligrosos** (cámara, GPS, micrófono) requieren solicitud explícita al usuario en runtime.
- `ActivityResultContracts.RequestPermission()` y `RequestMultiplePermissions()` son la forma moderna y recomendada de solicitar permisos en Compose.
- `rememberLauncherForActivityResult` registra el callback del resultado — siempre debe llamarse en el nivel del composable, nunca dentro de un `onClick`.
- Si el usuario deniega el permiso, mostrar un diálogo explicativo con opción de ir a los ajustes del sistema es la práctica recomendada por Google.

---

> **Siguiente página →** Página 17: Cámara con CameraX, galería con
> PhotoPicker y reproducción de video en Compose.