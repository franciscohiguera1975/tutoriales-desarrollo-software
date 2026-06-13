# Curso Kotlin — Página 19
## Módulo 7 · Hardware y Sistema
### Almacenamiento local: `DataStore`, `Room` y estrategia offline-first

---

## 🚀 TechDash — Módulo de Persistencia

> **Mismo proyecto TechDash** — continúa en `com.ute.techdash`.
> Esta página añade dos módulos de persistencia: **preferencias con DataStore**
> (tema oscuro, nombre de usuario, tamaño de fuente) y **lista de tareas con Room**
> (base de datos local con filtros, búsqueda y deshacer). Al terminar, el dashboard
> tendrá las rutas `ajustes` y `tareas`.

### Árbol de archivos de esta página

```
app/src/main/java/com/ute/techdash/
├── utils/
│   └── PreferenciasRepository.kt    ← 📄 NUEVO (DataStore)
├── data/
│   └── db/
│       ├── TareaEntity.kt           ← 📄 NUEVO (@Entity)
│       ├── TareasDao.kt             ← 📄 NUEVO (@Dao)
│       ├── AppDatabase.kt           ← 📄 NUEVO (@Database)
│       └── TareasRepository.kt      ← 📄 NUEVO
├── ui/
│   ├── ajustes/
│   │   ├── AjustesViewModel.kt      ← 📄 NUEVO
│   │   └── PantallaAjustes.kt       ← 📄 NUEVO
│   └── tareas/
│       ├── TareasViewModel.kt       ← 📄 NUEVO
│       └── PantallaTareas.kt        ← 📄 NUEVO
└── navigation/
    └── NavGraph.kt                  ← ✏️ MODIFICA
```

### Archivos que crearás/modificarás en esta página

| Archivo | Acción | Qué hace |
|---------|--------|----------|
| `utils/PreferenciasRepository.kt` | 📄 Nuevo | DataStore: tema oscuro, nombre, tamaño de fuente |
| `ui/ajustes/AjustesViewModel.kt` | 📄 Nuevo | ViewModel de preferencias |
| `ui/ajustes/PantallaAjustes.kt` | 📄 Nuevo | Pantalla de ajustes del dashboard |
| `data/db/TareaEntity.kt` | 📄 Nuevo | Entidad Room: tabla `tareas` y `etiquetas` |
| `data/db/TareasDao.kt` | 📄 Nuevo | Consultas SQL tipadas con Flow |
| `data/db/AppDatabase.kt` | 📄 Nuevo | Singleton de la base de datos |
| `data/db/TareasRepository.kt` | 📄 Nuevo | Acceso unificado a la BD |
| `ui/tareas/TareasViewModel.kt` | 📄 Nuevo | Estado UI + filtros + búsqueda |
| `ui/tareas/PantallaTareas.kt` | 📄 Nuevo | Lista de tareas completa con Room |
| `navigation/NavGraph.kt` | ✏️ Modifica | Añade rutas `ajustes` y `tareas` |

---

## ¿Cuándo usar cada opción?

```
SharedPreferences   → Evitar. Síncrono, sin soporte coroutines.
DataStore           → Preferencias simples: tema, token, configuración.
Room                → Datos estructurados: listas, entidades relacionadas.
Archivos            → Imágenes, documentos, binarios grandes.
```

---

## Dependencias

> ⚠️ **AGP 9.x usa "Built-in Kotlin"** — el plugin KAPT es incompatible con este modo. Debemos usar **KSP** (Kotlin Symbol Processing).

Este proyecto usa **Version Catalog** (`gradle/libs.versions.toml`). La configuración requiere tocar **tres archivos** en este orden:

---

### Paso 1 — `gradle/libs.versions.toml`

Añade las líneas marcadas en `[versions]` y `[plugins]`:

```toml
[versions]
# … las versiones que ya tienes (agp, kotlin, coreKtx, etc.) …
ksp = "2.2.10-2.0.2"    # ← AÑADE esta línea

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-compose      = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
ksp                 = { id = "com.google.devtools.ksp", version.ref = "ksp" }   # ← AÑADE esta línea
```

---

### Paso 2 — `build.gradle.kts` (archivo raíz del proyecto)

```kotlin
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.compose)      apply false
    alias(libs.plugins.ksp)                 apply false   // ← AÑADE
}
```

---

### Paso 3 — `app/build.gradle.kts`

**Bloque `plugins`:**

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.ksp)                   // ← AÑADE
}
```

**Bloque `dependencies`:**

```kotlin
dependencies {
    // … las dependencias que ya tienes …

    // DataStore
    implementation("androidx.datastore:datastore-preferences:1.1.1")

    // Room  ← usa 2.7.0+; versiones anteriores fallan con KSP2 ("unexpected jvm signature V")
    val roomVersion = "2.7.0"
    implementation("androidx.room:room-runtime:$roomVersion")
    implementation("androidx.room:room-ktx:$roomVersion")
    ksp("androidx.room:room-compiler:$roomVersion")           // ← usa ksp(), no kapt()
}
```

---

### Paso 4 — `gradle.properties`

AGP 9.x tiene un conflicto con KSP por el nuevo modelo de fuentes de Kotlin. Añade esta línea al archivo `gradle.properties` (en la raíz del proyecto):

```properties
android.disallowKotlinSourceSets=false
```

> Sin esta propiedad, KSP lanza el error: `Using kotlin.sourceSets DSL to add Kotlin sources is not allowed` aunque el plugin esté correctamente configurado.

> ⚠️ **Si el error persiste después de guardar:** el proyecto tiene `org.gradle.configuration-cache=true` en `gradle.properties`, lo que puede hacer que Gradle ignore los cambios nuevos. Solución: **File → Invalidate Caches → Invalidate and Restart** en Android Studio. Después del reinicio, haz **Sync Now**.

---

Después de los cuatro cambios, guarda con **Ctrl+S** y pulsa **"Sync Now"** en el banner amarillo de Android Studio.

---

## Parte 1 — DataStore Preferences

`DataStore` es el reemplazo moderno de `SharedPreferences`.
Trabaja de forma asíncrona con coroutines y expone los datos
como `Flow` para que la UI reaccione a los cambios:

### `PreferenciasRepository`

```kotlin
// 📁 Nuevo archivo → utils/PreferenciasRepository.kt
// ──────────────────────────────────────────────────────────────────
// Capa de acceso a DataStore Preferences.
// Cada preferencia (modoOscuro, nombre, fuente) se expone como Flow<T>
// para que la UI reaccione automáticamente a los cambios.
// Las escrituras usan dataStore.edit { } — son suspend y transaccionales.
// ──────────────────────────────────────────────────────────────────
package com.ute.techdash.utils

import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.*
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.map

// Extensión de Context — crea una sola instancia del DataStore
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "preferencias_app")

// Claves tipadas para cada preferencia
object PreferenciasKeys {
    val MODO_OSCURO      = booleanPreferencesKey("modo_oscuro")
    val NOMBRE_USUARIO   = stringPreferencesKey("nombre_usuario")
    val TAMANO_FUENTE    = intPreferencesKey("tamano_fuente")
    val PRIMERA_VEZ      = booleanPreferencesKey("primera_vez")
    val TOKEN_SESION     = stringPreferencesKey("token_sesion")
    val ULTIMA_BUSQUEDA  = stringPreferencesKey("ultima_busqueda")
}

class PreferenciasRepository(private val context: Context) {

    // ── Lectura ──────────────────────────────────────────────────────

    val modoOscuro: Flow<Boolean> = context.dataStore.data
        .catch { emit(emptyPreferences()) }
        .map { prefs -> prefs[PreferenciasKeys.MODO_OSCURO] ?: false }

    val nombreUsuario: Flow<String> = context.dataStore.data
        .catch { emit(emptyPreferences()) }
        .map { prefs -> prefs[PreferenciasKeys.NOMBRE_USUARIO] ?: "" }

    val tamanioFuente: Flow<Int> = context.dataStore.data
        .catch { emit(emptyPreferences()) }
        .map { prefs -> prefs[PreferenciasKeys.TAMANO_FUENTE] ?: 16 }

    val esPrimeraVez: Flow<Boolean> = context.dataStore.data
        .catch { emit(emptyPreferences()) }
        .map { prefs -> prefs[PreferenciasKeys.PRIMERA_VEZ] ?: true }

    val tokenSesion: Flow<String?> = context.dataStore.data
        .catch { emit(emptyPreferences()) }
        .map { prefs -> prefs[PreferenciasKeys.TOKEN_SESION] }

    // ── Escritura ────────────────────────────────────────────────────

    suspend fun setModoOscuro(activo: Boolean) {
        context.dataStore.edit { prefs ->
            prefs[PreferenciasKeys.MODO_OSCURO] = activo
        }
    }

    suspend fun setNombreUsuario(nombre: String) {
        context.dataStore.edit { prefs ->
            prefs[PreferenciasKeys.NOMBRE_USUARIO] = nombre
        }
    }

    suspend fun setTamanioFuente(tamano: Int) {
        context.dataStore.edit { prefs ->
            prefs[PreferenciasKeys.TAMANO_FUENTE] = tamano.coerceIn(12, 24)
        }
    }

    suspend fun marcarPrimeraVezCompletada() {
        context.dataStore.edit { prefs ->
            prefs[PreferenciasKeys.PRIMERA_VEZ] = false
        }
    }

    suspend fun guardarToken(token: String) {
        context.dataStore.edit { prefs ->
            prefs[PreferenciasKeys.TOKEN_SESION] = token
        }
    }

    suspend fun cerrarSesion() {
        context.dataStore.edit { prefs ->
            prefs.remove(PreferenciasKeys.TOKEN_SESION)
            prefs.remove(PreferenciasKeys.NOMBRE_USUARIO)
        }
    }

    // Limpiar todas las preferencias
    suspend fun limpiarTodo() {
        context.dataStore.edit { it.clear() }
    }
}
```

### `AjustesViewModel` y pantalla

```kotlin
// 📁 Nuevo archivo → ui/ajustes/AjustesViewModel.kt
// ──────────────────────────────────────────────────────────────────
// ViewModel de la pantalla de Ajustes.
// Expone los Flows del repositorio y lanza coroutines para las escrituras.
// La factory recibe Context para construir PreferenciasRepository.
// ──────────────────────────────────────────────────────────────────
package com.ute.techdash.ui.ajustes

import android.content.Context
import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import com.ute.techdash.utils.PreferenciasRepository
import kotlinx.coroutines.launch

class AjustesViewModel(
    private val repositorio: PreferenciasRepository
) : ViewModel() {

    val modoOscuro    by lazy { repositorio.modoOscuro }
    val nombreUsuario by lazy { repositorio.nombreUsuario }
    val tamanioFuente by lazy { repositorio.tamanioFuente }

    fun toggleModoOscuro(activo: Boolean) {
        viewModelScope.launch { repositorio.setModoOscuro(activo) }
    }

    fun actualizarNombre(nombre: String) {
        viewModelScope.launch { repositorio.setNombreUsuario(nombre) }
    }

    fun actualizarFuente(tamano: Int) {
        viewModelScope.launch { repositorio.setTamanioFuente(tamano) }
    }

    fun cerrarSesion() {
        viewModelScope.launch { repositorio.cerrarSesion() }
    }

    companion object {
        fun factory(context: Context): ViewModelProvider.Factory = object : ViewModelProvider.Factory {
            @Suppress("UNCHECKED_CAST")
            override fun <T : ViewModel> create(modelClass: Class<T>): T =
                AjustesViewModel(PreferenciasRepository(context)) as T
        }
    }
}
```

```kotlin
// 📁 Nuevo archivo → ui/ajustes/PantallaAjustes.kt
// ──────────────────────────────────────────────────────────────────
// Pantalla de ajustes: toggle de tema oscuro, slider de fuente y nombre.
// Observa los Flows del ViewModel con collectAsStateWithLifecycle.
// Los cambios se persisten en DataStore y sobreviven al cierre de la app.
// ──────────────────────────────────────────────────────────────────
package com.ute.techdash.ui.ajustes

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.DarkMode
import androidx.compose.material.icons.filled.LightMode
import androidx.compose.material.icons.filled.Logout
import androidx.compose.material.icons.filled.Person
import androidx.compose.material3.Button
import androidx.compose.material3.ButtonDefaults
import androidx.compose.material3.ElevatedCard
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.HorizontalDivider
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Slider
import androidx.compose.material3.Switch
import androidx.compose.material3.Text
import androidx.compose.material3.TopAppBar
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
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.SnackbarHost
import androidx.compose.material3.SnackbarHostState
import androidx.compose.runtime.rememberCoroutineScope
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import kotlinx.coroutines.launch

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PantallaAjustes() {
    val context = LocalContext.current
    val vm: AjustesViewModel = viewModel(factory = AjustesViewModel.factory(context))
    val modoOscuro    by vm.modoOscuro.collectAsStateWithLifecycle(false)
    val nombre        by vm.nombreUsuario.collectAsStateWithLifecycle("")
    val fuente        by vm.tamanioFuente.collectAsStateWithLifecycle(16)
    var nombreEdit    by remember(nombre) { mutableStateOf(nombre) }
    val snackbarState = remember { SnackbarHostState() }
    val scope         = rememberCoroutineScope()

    Scaffold(
        topBar       = { TopAppBar(title = { Text("Ajustes", fontWeight = FontWeight.Bold) }) },
        snackbarHost = { SnackbarHost(snackbarState) }
    ) { padding ->
        LazyColumn(
            modifier        = Modifier.padding(padding).fillMaxSize(),
            contentPadding  = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            // Apariencia
            item {
                Text("Apariencia", style = MaterialTheme.typography.labelLarge,
                    color = MaterialTheme.colorScheme.primary,
                    modifier = Modifier.padding(bottom = 4.dp))

                ElevatedCard(Modifier.fillMaxWidth()) {
                    Column(Modifier.padding(16.dp)) {
                        // Toggle modo oscuro
                        Row(
                            Modifier.fillMaxWidth(),
                            horizontalArrangement = Arrangement.SpaceBetween,
                            verticalAlignment     = Alignment.CenterVertically
                        ) {
                            Row(verticalAlignment = Alignment.CenterVertically) {
                                Icon(
                                    if (modoOscuro) Icons.Default.DarkMode else Icons.Default.LightMode,
                                    null,
                                    tint = MaterialTheme.colorScheme.primary
                                )
                                Spacer(Modifier.width(12.dp))
                                Column {
                                    Text("Modo oscuro", fontWeight = FontWeight.Medium)
                                    Text("Cambia el tema visual de la app",
                                        style = MaterialTheme.typography.bodySmall,
                                        color = MaterialTheme.colorScheme.onSurfaceVariant)
                                }
                            }
                            Switch(
                                checked         = modoOscuro,
                                onCheckedChange = { vm.toggleModoOscuro(it) }
                            )
                        }

                        HorizontalDivider(Modifier.padding(vertical = 12.dp))

                        // Tamaño de fuente
                        Text("Tamaño de fuente: ${fuente}sp",
                            fontWeight = FontWeight.Medium)
                        Spacer(Modifier.height(8.dp))
                        Slider(
                            value         = fuente.toFloat(),
                            onValueChange = { vm.actualizarFuente(it.toInt()) },
                            valueRange    = 12f..24f,
                            steps         = 5,
                            modifier      = Modifier.fillMaxWidth()
                        )
                        Row(Modifier.fillMaxWidth(),
                            horizontalArrangement = Arrangement.SpaceBetween) {
                            Text("12sp", style = MaterialTheme.typography.labelSmall)
                            Text("24sp", style = MaterialTheme.typography.labelSmall)
                        }
                    }
                }
            }

            // Perfil
            item {
                Text("Perfil", style = MaterialTheme.typography.labelLarge,
                    color = MaterialTheme.colorScheme.primary,
                    modifier = Modifier.padding(bottom = 4.dp))

                ElevatedCard(Modifier.fillMaxWidth()) {
                    Column(Modifier.padding(16.dp)) {
                        OutlinedTextField(
                            value         = nombreEdit,
                            onValueChange = { nombreEdit = it },
                            label         = { Text("Nombre de usuario") },
                            leadingIcon   = { Icon(Icons.Default.Person, null) },
                            singleLine    = true,
                            modifier      = Modifier.fillMaxWidth()
                        )
                        Spacer(Modifier.height(8.dp))
                        Button(
                            onClick  = { vm.actualizarNombre(nombreEdit) },
                            enabled  = nombreEdit.isNotBlank() && nombreEdit != nombre,
                            modifier = Modifier.align(Alignment.End)
                        ) { Text("Guardar") }
                    }
                }
            }

            // Sesión
            item {
                OutlinedButton(
                    onClick  = {
                        vm.cerrarSesion()
                        scope.launch {
                            snackbarState.showSnackbar("Sesión cerrada — nombre y token eliminados")
                        }
                    },
                    modifier = Modifier.fillMaxWidth(),
                    colors   = ButtonDefaults.outlinedButtonColors(
                        contentColor = MaterialTheme.colorScheme.error
                    )
                ) {
                    Icon(Icons.Default.Logout, null)
                    Spacer(Modifier.width(8.dp))
                    Text("Cerrar sesión")
                }
            }
        }
    }
}
```

---

### ▶ Checkpoint 1 — DataStore: ajustes que persisten entre sesiones

**Conecta los archivos al proyecto:**

1. Añade la dependencia DataStore en `build.gradle.kts` (módulo `app`).
2. En Android Studio, crea el paquete `ui/ajustes/`.
3. Copia `PreferenciasRepository.kt` en `utils/`.
4. Copia `AjustesViewModel.kt` y `PantallaAjustes.kt` en `ui/ajustes/`.
5. En `NavGraph.kt`, añade dentro del bloque `NavHost`:
   ```kotlin
   composable("ajustes") { PantallaAjustes() }
   ```
6. En `PantallaDashboard`, añade un botón `⚙️ Ajustes` que llame a `navController.navigate("ajustes")`.
7. Ejecuta en el emulador.

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código:

```kotlin
package com.ute.techdash

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.ute.techdash.ui.TechDashNavGraph
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                TechDashNavGraph()
            }
        }
    }
}
```

📁 **Nuevo archivo → `ui/PantallaDashboard.kt`**

```kotlin
package com.ute.techdash.ui

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.lazy.grid.GridCells
import androidx.compose.foundation.lazy.grid.LazyVerticalGrid
import androidx.compose.foundation.lazy.grid.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Dashboard
import androidx.compose.material.icons.filled.LocationOn
import androidx.compose.material.icons.filled.Memory
import androidx.compose.material.icons.filled.Sensors
import androidx.compose.material.icons.filled.Settings
import androidx.compose.material3.ElevatedCard
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.material3.TopAppBar
import androidx.compose.material3.TopAppBarDefaults
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.navigation.NavController

private data class Destino(
    val ruta: String,
    val titulo: String,
    val descripcion: String,
    val icono: ImageVector
)

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PantallaDashboard(navController: NavController) {
    val destinos = listOf(
        Destino("gps",      "GPS",      "Coordenadas en vivo",   Icons.Default.LocationOn),
        Destino("sensores", "Sensores", "Acelerómetro y luz",    Icons.Default.Sensors),
        Destino("hardware", "Hardware", "Dashboard unificado",   Icons.Default.Dashboard),
        Destino("ajustes",  "Ajustes",  "Tema, nombre y fuente", Icons.Default.Settings)
        // ← Checkpoint 2 añade "tareas" aquí
    )

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("TechDash", fontWeight = FontWeight.Bold) },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.onPrimaryContainer
                )
            )
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(horizontal = 20.dp, vertical = 24.dp),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            Surface(
                shape = MaterialTheme.shapes.extraLarge,
                color = MaterialTheme.colorScheme.primaryContainer,
                modifier = Modifier.size(72.dp)
            ) {
                Box(contentAlignment = Alignment.Center) {
                    Icon(
                        Icons.Default.Memory,
                        contentDescription = null,
                        modifier = Modifier.size(40.dp),
                        tint = MaterialTheme.colorScheme.onPrimaryContainer
                    )
                }
            }
            Spacer(Modifier.height(16.dp))
            Text(
                "Módulos del curso",
                style = MaterialTheme.typography.headlineSmall,
                fontWeight = FontWeight.Bold,
                textAlign = TextAlign.Center
            )
            Text(
                "Páginas 18 y 19",
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant,
                textAlign = TextAlign.Center
            )
            Spacer(Modifier.height(28.dp))
            LazyVerticalGrid(
                columns = GridCells.Fixed(2),
                horizontalArrangement = Arrangement.spacedBy(12.dp),
                verticalArrangement   = Arrangement.spacedBy(12.dp)
            ) {
                items(destinos) { d ->
                    ElevatedCard(
                        onClick = { navController.navigate(d.ruta) },
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Column(
                            modifier = Modifier.padding(18.dp).fillMaxWidth(),
                            horizontalAlignment = Alignment.CenterHorizontally,
                            verticalArrangement = Arrangement.spacedBy(8.dp)
                        ) {
                            Surface(
                                shape = MaterialTheme.shapes.large,
                                color = MaterialTheme.colorScheme.secondaryContainer,
                                modifier = Modifier.size(48.dp)
                            ) {
                                Box(contentAlignment = Alignment.Center) {
                                    Icon(
                                        d.icono,
                                        contentDescription = null,
                                        modifier = Modifier.size(26.dp),
                                        tint = MaterialTheme.colorScheme.onSecondaryContainer
                                    )
                                }
                            }
                            Text(d.titulo, fontWeight = FontWeight.SemiBold,
                                style = MaterialTheme.typography.titleSmall)
                            Text(d.descripcion,
                                style = MaterialTheme.typography.bodySmall,
                                color = MaterialTheme.colorScheme.onSurfaceVariant,
                                textAlign = TextAlign.Center)
                        }
                    }
                }
            }
        }
    }
}
```

📁 **Crea o reemplaza `ui/NavGraph.kt`** con este código:

```kotlin
package com.ute.techdash.ui

import androidx.compose.runtime.Composable
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.ute.techdash.ui.ajustes.PantallaAjustes
import com.ute.techdash.ui.hardware.DashboardHardware
import com.ute.techdash.ui.hardware.gps.PantallaGPS
import com.ute.techdash.ui.hardware.sensores.PantallaSensores

@Composable
fun TechDashNavGraph() {
    val navController = rememberNavController()
    NavHost(navController, startDestination = "dashboard") {
        composable("dashboard") { PantallaDashboard(navController) }
        composable("gps")       { PantallaGPS() }
        composable("sensores")  { PantallaSensores() }
        composable("hardware")  { DashboardHardware() }
        composable("ajustes")   { PantallaAjustes() }    // ← nuevo (p19)
    }
}
```

**¿Qué deberías ver?**
- ⚙️ La pantalla de Ajustes abre desde el dashboard.
- 🌙 El switch de modo oscuro **cambia de posición** y al volver a entrar a la pantalla **mantiene su estado** (DataStore lo persiste).
- ✏️ Escribe un nombre y pulsa **Guardar**; mata la app completamente (botón cuadrado → desliza hacia arriba) y vuélvela a abrir — el nombre debe seguir en el campo.
- 🔤 Mueve el slider y cierra la app — al regresar debe tener el mismo valor.
- 🚪 Al pulsar **Cerrar sesión** aparece un Snackbar de confirmación y el campo de nombre se limpia.

> ⚠️ **El switch no cambia el color de la app** — eso es normal en este punto. DataStore **guarda** el valor correctamente, pero conectarlo al `MaterialTheme` de `MainActivity` requiere observar el Flow desde fuera del NavGraph. Eso es exactamente lo que propone el **Reto 2** al final del checkpoint. Verifica que el valor **persiste**: activa el switch, mata la app, vuelve a entrar — el switch debe seguir activado aunque el tema no cambie aún.

> 💡 Si el switch no persiste: revisa que `val Context.dataStore` se declara como propiedad de nivel superior en `PreferenciasRepository.kt` (fuera de la clase) y que el `ViewModel` recibe la instancia correcta del repositorio.

### 🧪 Prueba esto

> **Reto 1 — Preferencia nueva:** Añade la clave `IDIOMA` (`stringPreferencesKey`) con valores `"es"` y `"en"`. Añade un `DropdownMenu` en `PantallaAjustes` para elegir el idioma y guárdalo con `dataStore.edit`. Cierra y reabre la app — ¿el idioma seleccionado persiste?
>
> **Reto 2 — Tema real:** Conecta la preferencia `modoOscuro` al `MaterialTheme` de `MainActivity`: cuando sea `true`, pasa `darkColorScheme()` al tema; cuando sea `false`, `lightColorScheme()`. Observa cómo **toda la app** cambia de tema al activar el switch desde `PantallaAjustes`.
>
> **Reflexión:** `DataStore` expone los datos como `Flow`, no como un valor que lees una sola vez. ¿Qué ventaja tiene esto para la UI? ¿Qué pasaría si dos pantallas distintas observaran la misma preferencia y una la modificara — se actualizaría la otra automáticamente?

---

## Parte 2 — Room: base de datos local

Room es el ORM de Android sobre SQLite. Trabaja con
coroutines y expone consultas como `Flow` para que la UI
se actualice automáticamente al cambiar los datos:

```
@Entity     → Tabla en la base de datos
@Dao        → Interfaz con las consultas SQL
@Database   → Configuración de la BD y punto de entrada
```

### Entidades

```kotlin
// 📁 Nuevo archivo → data/db/TareaEntity.kt
// ──────────────────────────────────────────────────────────────────
// Define las tablas de la base de datos Room.
// @Entity "tareas" con prioridad (1=baja, 2=media, 3=alta) y etiqueta.
// TareaConEtiqueta es una proyección de JOIN — no es una tabla.
// ──────────────────────────────────────────────────────────────────
package com.ute.techdash.data.db

import androidx.room.Embedded
import androidx.room.Entity
import androidx.room.PrimaryKey
import androidx.room.Relation

// Tabla: tareas
@Entity(tableName = "tareas")
data class TareaEntity(
    @PrimaryKey(autoGenerate = true)
    val id:          Int     = 0,
    val titulo:      String,
    val descripcion: String  = "",
    val completada:  Boolean = false,
    val prioridad:   Int     = 1,        // 1=baja, 2=media, 3=alta
    val creadaEn:    Long    = System.currentTimeMillis(),
    val etiqueta:    String  = ""
)

// Tabla: etiquetas
@Entity(tableName = "etiquetas")
data class EtiquetaEntity(
    @PrimaryKey(autoGenerate = true)
    val id:     Int    = 0,
    val nombre: String,
    val color:  String = "#1976D2"
)

// Relación: tarea con su etiqueta (join)
data class TareaConEtiqueta(
    @Embedded val tarea:     TareaEntity,
    @Relation(
        parentColumn = "etiqueta",
        entityColumn = "nombre"
    )
    val etiquetaObj: EtiquetaEntity?
)
```

### `@Dao` — consultas

```kotlin
// 📁 Nuevo archivo → data/db/TareasDao.kt
// ──────────────────────────────────────────────────────────────────
// Interfaz DAO con todas las consultas SQL de la tabla "tareas".
// Las funciones que retornan Flow se actualizan solas al cambiar los datos.
// Las funciones suspend son operaciones puntuales (insertar, actualizar, etc.).
// ──────────────────────────────────────────────────────────────────
package com.ute.techdash.data.db

import androidx.room.Dao
import androidx.room.Delete
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query
import androidx.room.Update
import kotlinx.coroutines.flow.Flow

@Dao
interface TareasDao {

    // Flow — la UI se actualiza sola cuando cambian los datos
    @Query("SELECT * FROM tareas ORDER BY prioridad DESC, creadaEn DESC")
    fun obtenerTodas(): Flow<List<TareaEntity>>

    @Query("SELECT * FROM tareas WHERE completada = :completada ORDER BY prioridad DESC")
    fun obtenerPorEstado(completada: Boolean): Flow<List<TareaEntity>>

    @Query("SELECT * FROM tareas WHERE titulo LIKE '%' || :busqueda || '%'")
    fun buscar(busqueda: String): Flow<List<TareaEntity>>

    @Query("SELECT COUNT(*) FROM tareas WHERE completada = 0")
    fun contarPendientes(): Flow<Int>

    // Suspend — operación puntual
    @Query("SELECT * FROM tareas WHERE id = :id")
    suspend fun obtenerPorId(id: Int): TareaEntity?

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertar(tarea: TareaEntity): Long

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertarTodas(tareas: List<TareaEntity>)

    @Update
    suspend fun actualizar(tarea: TareaEntity)

    @Query("UPDATE tareas SET completada = :completada WHERE id = :id")
    suspend fun toggleCompletada(id: Int, completada: Boolean)

    @Delete
    suspend fun eliminar(tarea: TareaEntity)

    @Query("DELETE FROM tareas WHERE completada = 1")
    suspend fun eliminarCompletadas()

    @Query("DELETE FROM tareas")
    suspend fun eliminarTodas()
}
```

### `@Database`

```kotlin
// 📁 Nuevo archivo → data/db/AppDatabase.kt
// ──────────────────────────────────────────────────────────────────
// Punto de entrada único a la base de datos SQLite via Room.
// Singleton con @Volatile para seguridad en hilos concurrentes.
// fallbackToDestructiveMigration() → durante desarrollo borra y recrea la BD.
// ──────────────────────────────────────────────────────────────────
package com.ute.techdash.data.db

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase
import androidx.sqlite.db.SupportSQLiteDatabase

@Database(
    entities  = [TareaEntity::class, EtiquetaEntity::class],
    version   = 1,
    exportSchema = false
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun tareasDao(): TareasDao

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val db = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                )
                    .addCallback(object : RoomDatabase.Callback() {
                        override fun onCreate(db: SupportSQLiteDatabase) {
                            super.onCreate(db)
                            // Datos iniciales (seed)
                        }
                    })
                    // Manejo de migraciones
                    .fallbackToDestructiveMigration()
                    .build()
                    .also { INSTANCE = it }
                db
            }
        }
    }
}
```

### `TareasRepository`

```kotlin
// 📁 Nuevo archivo → data/db/TareasRepository.kt
// ──────────────────────────────────────────────────────────────────
// Capa intermedia entre el DAO y el ViewModel.
// Abstrae el origen de los datos — el ViewModel no conoce Room directamente.
// Expone Flows públicos y operaciones suspend para la UI.
// ──────────────────────────────────────────────────────────────────
package com.ute.techdash.data.db

import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map

class TareasRepository(private val dao: TareasDao) {

    // Flows públicos
    val todasLasTareas: Flow<List<TareaEntity>>  = dao.obtenerTodas()
    val tareasPendientes: Flow<List<TareaEntity>> = dao.obtenerPorEstado(false)
    val contadorPendientes: Flow<Int>             = dao.contarPendientes()

    fun buscar(query: String): Flow<List<TareaEntity>> = dao.buscar(query)

    // Operaciones suspendidas
    suspend fun agregar(titulo: String, descripcion: String = "", prioridad: Int = 1): Long =
        dao.insertar(TareaEntity(titulo = titulo, descripcion = descripcion, prioridad = prioridad))

    suspend fun toggleCompletada(tarea: TareaEntity) =
        dao.toggleCompletada(tarea.id, !tarea.completada)

    suspend fun actualizar(tarea: TareaEntity) = dao.actualizar(tarea)

    suspend fun eliminar(tarea: TareaEntity) = dao.eliminar(tarea)

    suspend fun eliminarCompletadas() = dao.eliminarCompletadas()
}
```

### `TareasViewModel`

```kotlin
// 📁 Nuevo archivo → ui/tareas/TareasViewModel.kt  (incluye TareasUiState, FiltroTareas y TareasViewModel)
// ──────────────────────────────────────────────────────────────────
// ViewModel de la lista de tareas. Combina filtro + búsqueda + contador
// en un único StateFlow<TareasUiState> usando combine + flatMapLatest.
// La factory construye TareasRepository a partir de AppDatabase.
// ──────────────────────────────────────────────────────────────────
package com.ute.techdash.ui.tareas

import android.content.Context
import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import com.ute.techdash.data.db.AppDatabase
import com.ute.techdash.data.db.TareaEntity
import com.ute.techdash.data.db.TareasRepository
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.flatMapLatest
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

data class TareasUiState(
    val tareas:     List<TareaEntity> = emptyList(),
    val pendientes: Int               = 0,
    val busqueda:   String            = "",
    val filtro:     FiltroTareas      = FiltroTareas.TODAS
)

enum class FiltroTareas { TODAS, PENDIENTES, COMPLETADAS }

class TareasViewModel(
    private val repositorio: TareasRepository
) : ViewModel() {

    private val _filtro   = MutableStateFlow(FiltroTareas.TODAS)
    private val _busqueda = MutableStateFlow("")

    // Combinar filtro + búsqueda en un solo Flow de estado
    val uiState: StateFlow<TareasUiState> = combine(
        _filtro,
        _busqueda,
        repositorio.contadorPendientes
    ) { filtro, busqueda, pendientes ->
        Triple(filtro, busqueda, pendientes)
    }.flatMapLatest { (filtro, busqueda, pendientes) ->
        val flujoDatos = when {
            busqueda.isNotBlank() -> repositorio.buscar(busqueda)
            filtro == FiltroTareas.PENDIENTES  ->
                repositorio.tareasPendientes
            filtro == FiltroTareas.COMPLETADAS ->
                repositorio.todasLasTareas.map { it.filter { t -> t.completada } }
            else -> repositorio.todasLasTareas
        }
        flujoDatos.map { tareas ->
            TareasUiState(
                tareas     = tareas,
                pendientes = pendientes,
                busqueda   = busqueda,
                filtro     = filtro
            )
        }
    }.stateIn(
        scope            = viewModelScope,
        started          = SharingStarted.WhileSubscribed(5_000),
        initialValue     = TareasUiState()
    )

    fun setBusqueda(q: String)       { _busqueda.value = q }
    fun setFiltro(f: FiltroTareas)   { _filtro.value   = f }

    fun agregar(titulo: String, descripcion: String = "", prioridad: Int = 1) {
        if (titulo.isBlank()) return
        viewModelScope.launch { repositorio.agregar(titulo, descripcion, prioridad) }
    }

    fun toggleCompletada(tarea: TareaEntity) {
        viewModelScope.launch { repositorio.toggleCompletada(tarea) }
    }

    fun eliminar(tarea: TareaEntity) {
        viewModelScope.launch { repositorio.eliminar(tarea) }
    }

    fun eliminarCompletadas() {
        viewModelScope.launch { repositorio.eliminarCompletadas() }
    }

    companion object {
        fun factory(context: Context): ViewModelProvider.Factory = object : ViewModelProvider.Factory {
            @Suppress("UNCHECKED_CAST")
            override fun <T : ViewModel> create(modelClass: Class<T>): T {
                val db = AppDatabase.getInstance(context)
                return TareasViewModel(TareasRepository(db.tareasDao())) as T
            }
        }
    }
}
```

---

## Parte 3 — Estrategia offline-first

La estrategia offline-first muestra datos locales inmediatamente
y sincroniza con la API en segundo plano:

```kotlin
// ⚠️ PATRÓN ILUSTRATIVO — no es un archivo que debes copiar literalmente.
// Muestra la estructura conceptual; ProductosApiService es un placeholder
// de cualquier cliente Retrofit que ya tengas en tu proyecto.

// Repositorio offline-first que combina Room + API
class ProductosOfflineRepository(
    private val dao:    TareasDao,           // Room como fuente de verdad
    private val api:    ProductosApiService  // API como fuente de datos remotos
) {
    // 1. Siempre emite desde Room (instantáneo)
    // 2. Lanza sincronización con la API en background
    // 3. Room actualiza → Flow emite → UI se actualiza sola
    fun obtenerProductos(): Flow<List<TareaEntity>> {
        return dao.obtenerTodas()
            .onStart {
                // Sincronizar en background sin bloquear el Flow
                kotlinx.coroutines.GlobalScope.launch {
                    sincronizarDesdeApi()
                }
            }
    }

    private suspend fun sincronizarDesdeApi() {
        try {
            val remotos = api.obtenerProductos()
            // Insertar/actualizar en Room — el Flow se actualiza automáticamente
            dao.insertarTodas(remotos.results.map { dto ->
                TareaEntity(
                    id     = dto.id,
                    titulo = dto.name,
                    descripcion = dto.category_name
                )
            })
        } catch (e: Exception) {
            // Error de red — los datos locales siguen disponibles
            e.printStackTrace()
        }
    }
}
```

---

## Programa completo — app de tareas con Room y DataStore

```kotlin
// 📁 Nuevo archivo → ui/tareas/PantallaTareas.kt  (incluye PantallaTareas e ItemTarea)
// ──────────────────────────────────────────────────────────────────
// Lista de tareas con búsqueda en tiempo real, filtros por estado y prioridad.
// FAB abre un AlertDialog para crear tareas; eliminar muestra Snackbar de deshacer.
// ItemTarea colorea la tarjeta según prioridad (gris/azul/rojo).
// ──────────────────────────────────────────────────────────────────
package com.ute.techdash.ui.tareas

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Add
import androidx.compose.material.icons.filled.Clear
import androidx.compose.material.icons.filled.Delete
import androidx.compose.material.icons.filled.DoneAll
import androidx.compose.material.icons.filled.Search
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.Checkbox
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.FilterChip
import androidx.compose.material3.FloatingActionButton
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Scaffold
import androidx.compose.material3.SnackbarDuration
import androidx.compose.material3.SnackbarHost
import androidx.compose.material3.SnackbarHostState
import androidx.compose.material3.SnackbarResult
import androidx.compose.material3.Text
import androidx.compose.material3.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.ute.techdash.data.db.TareaEntity

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PantallaTareas() {
    val context = LocalContext.current
    val vm: TareasViewModel = viewModel(factory = TareasViewModel.factory(context))
    val uiState     by vm.uiState.collectAsStateWithLifecycle()
    var mostrarForm by remember { mutableStateOf(false) }
    var tituloNueva by remember { mutableStateOf("") }
    var prioridadNueva by remember { mutableStateOf(1) }

    val snackbarState = remember { SnackbarHostState() }
    var ultimaEliminada by remember { mutableStateOf<TareaEntity?>(null) }

    Scaffold(
        topBar = {
            TopAppBar(
                title = {
                    Column {
                        Text("Mis tareas", fontWeight = FontWeight.Bold)
                        if (uiState.pendientes > 0) {
                            Text(
                                "${uiState.pendientes} pendiente(s)",
                                style = MaterialTheme.typography.labelSmall,
                                color = MaterialTheme.colorScheme.onSurfaceVariant
                            )
                        }
                    }
                },
                actions = {
                    if (uiState.tareas.any { it.completada }) {
                        IconButton(onClick = { vm.eliminarCompletadas() }) {
                            Icon(Icons.Default.DoneAll, "Eliminar completadas")
                        }
                    }
                }
            )
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { mostrarForm = true }) {
                Icon(Icons.Default.Add, "Nueva tarea")
            }
        },
        snackbarHost = { SnackbarHost(snackbarState) }
    ) { padding ->
        Column(modifier = Modifier.padding(padding).fillMaxSize()) {

            // Búsqueda
            OutlinedTextField(
                value         = uiState.busqueda,
                onValueChange = { vm.setBusqueda(it) },
                placeholder   = { Text("Buscar tarea...") },
                leadingIcon   = { Icon(Icons.Default.Search, null) },
                trailingIcon  = {
                    if (uiState.busqueda.isNotEmpty())
                        IconButton(onClick = { vm.setBusqueda("") }) {
                            Icon(Icons.Default.Clear, null)
                        }
                },
                singleLine = true,
                modifier   = Modifier.fillMaxWidth().padding(horizontal = 16.dp, vertical = 8.dp)
            )

            // Filtros
            LazyRow(
                horizontalArrangement = Arrangement.spacedBy(8.dp),
                contentPadding        = PaddingValues(horizontal = 16.dp)
            ) {
                items(FiltroTareas.entries) { filtro ->
                    FilterChip(
                        selected = uiState.filtro == filtro,
                        onClick  = { vm.setFiltro(filtro) },
                        label    = {
                            Text(when (filtro) {
                                FiltroTareas.TODAS      -> "Todas"
                                FiltroTareas.PENDIENTES -> "Pendientes"
                                FiltroTareas.COMPLETADAS -> "Completadas"
                            })
                        }
                    )
                }
            }

            Spacer(Modifier.height(8.dp))

            // Lista de tareas
            LazyColumn(
                modifier            = Modifier.weight(1f),
                contentPadding      = PaddingValues(horizontal = 16.dp, vertical = 4.dp),
                verticalArrangement = Arrangement.spacedBy(6.dp)
            ) {
                if (uiState.tareas.isEmpty()) {
                    item {
                        Box(
                            Modifier.fillMaxWidth().padding(top = 48.dp),
                            contentAlignment = Alignment.Center
                        ) {
                            Text(
                                if (uiState.busqueda.isNotEmpty())
                                    "Sin resultados para \"${uiState.busqueda}\""
                                else "¡Sin tareas! Añade una con el botón +",
                                color = MaterialTheme.colorScheme.onSurfaceVariant,
                                style = MaterialTheme.typography.bodyMedium
                            )
                        }
                    }
                }

                items(uiState.tareas, key = { it.id }) { tarea ->
                    ItemTarea(
                        tarea        = tarea,
                        onToggle     = { vm.toggleCompletada(tarea) },
                        onEliminar   = {
                            ultimaEliminada = tarea
                            vm.eliminar(tarea)
                        }
                    )
                }
            }
        }
    }

    // Snackbar de deshacer eliminación
    LaunchedEffect(ultimaEliminada) {
        ultimaEliminada?.let { tarea ->
            val resultado = snackbarState.showSnackbar(
                message     = "\"${tarea.titulo}\" eliminada",
                actionLabel = "Deshacer",
                duration    = SnackbarDuration.Short
            )
            if (resultado == SnackbarResult.ActionPerformed) {
                vm.agregar(tarea.titulo, tarea.descripcion, tarea.prioridad)
            }
            ultimaEliminada = null
        }
    }

    // Diálogo para nueva tarea
    if (mostrarForm) {
        AlertDialog(
            onDismissRequest = { mostrarForm = false; tituloNueva = "" },
            title   = { Text("Nueva tarea") },
            text    = {
                Column(verticalArrangement = Arrangement.spacedBy(12.dp)) {
                    OutlinedTextField(
                        value         = tituloNueva,
                        onValueChange = { tituloNueva = it },
                        label         = { Text("Título de la tarea") },
                        singleLine    = true,
                        modifier      = Modifier.fillMaxWidth()
                    )
                    Text("Prioridad", style = MaterialTheme.typography.labelMedium)
                    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                        listOf(1 to "Baja", 2 to "Media", 3 to "Alta").forEach { (valor, label) ->
                            FilterChip(
                                selected = prioridadNueva == valor,
                                onClick  = { prioridadNueva = valor },
                                label    = { Text(label) }
                            )
                        }
                    }
                }
            },
            confirmButton = {
                Button(
                    onClick  = {
                        vm.agregar(tituloNueva, prioridad = prioridadNueva)
                        tituloNueva    = ""
                        prioridadNueva = 1
                        mostrarForm    = false
                    },
                    enabled = tituloNueva.isNotBlank()
                ) { Text("Agregar") }
            },
            dismissButton = {
                OutlinedButton(onClick = { mostrarForm = false; tituloNueva = "" }) {
                    Text("Cancelar")
                }
            }
        )
    }
}

@Composable
fun ItemTarea(
    tarea:      TareaEntity,
    onToggle:   () -> Unit,
    onEliminar: () -> Unit
) {
    val coloresPrioridad = mapOf(
        1 to MaterialTheme.colorScheme.surfaceVariant,
        2 to MaterialTheme.colorScheme.secondaryContainer,
        3 to MaterialTheme.colorScheme.errorContainer
    )

    Card(
        modifier = Modifier.fillMaxWidth(),
        colors   = CardDefaults.cardColors(
            containerColor = coloresPrioridad[tarea.prioridad]
                ?: MaterialTheme.colorScheme.surface
        )
    ) {
        Row(
            Modifier.padding(horizontal = 12.dp, vertical = 8.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Checkbox(
                checked         = tarea.completada,
                onCheckedChange = { onToggle() }
            )
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    tarea.titulo,
                    fontWeight    = if (tarea.completada) FontWeight.Normal else FontWeight.SemiBold,
                    style         = MaterialTheme.typography.bodyMedium,
                    textDecoration = if (tarea.completada)
                        androidx.compose.ui.text.style.TextDecoration.LineThrough
                    else null,
                    color = if (tarea.completada)
                        MaterialTheme.colorScheme.onSurfaceVariant
                    else
                        MaterialTheme.colorScheme.onSurface
                )
                if (tarea.descripcion.isNotBlank()) {
                    Text(
                        tarea.descripcion,
                        style = MaterialTheme.typography.bodySmall,
                        color = MaterialTheme.colorScheme.onSurfaceVariant,
                        maxLines = 1,
                        overflow = TextOverflow.Ellipsis
                    )
                }
            }
            IconButton(onClick = onEliminar) {
                Icon(Icons.Default.Delete, "Eliminar",
                    tint = MaterialTheme.colorScheme.error,
                    modifier = Modifier.size(20.dp))
            }
        }
    }
}
```

---

### ▶ Checkpoint 2 — Room: lista de tareas persistente

**Conecta los archivos al proyecto:**

1. Asegúrate de haber completado los **4 pasos de dependencias** al inicio de esta página (KSP ya configurado).
2. Crea el paquete `data/db/`.
3. Copia `TareaEntity.kt`, `TareasDao.kt`, `AppDatabase.kt` y `TareasRepository.kt` en `data/db/`.
4. Crea el paquete `ui/tareas/` y copia `TareasViewModel.kt` y `PantallaTareas.kt`.
5. En `NavGraph.kt`, añade dentro del bloque `NavHost`:
   ```kotlin
   composable("tareas") { PantallaTareas() }
   ```
6. En `ui/PantallaDashboard.kt` añade el import y la línea en la lista `destinos`:
   ```kotlin
   // Import (junto a los demás de material.icons.filled):
   import androidx.compose.material.icons.filled.CheckBox

   // Dentro de val destinos = listOf(...), añade al final antes del cierre ):
   Destino("tareas", "Tareas", "Lista de tareas con Room", Icons.Default.CheckBox)
   ```
7. Ejecuta en el emulador.

📁 **Reemplaza TODO el contenido de `ui/NavGraph.kt`** con este código:

```kotlin
package com.ute.techdash.ui

import androidx.compose.runtime.Composable
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.ute.techdash.ui.ajustes.PantallaAjustes
import com.ute.techdash.ui.hardware.DashboardHardware
import com.ute.techdash.ui.hardware.gps.PantallaGPS
import com.ute.techdash.ui.hardware.sensores.PantallaSensores
import com.ute.techdash.ui.tareas.PantallaTareas

@Composable
fun TechDashNavGraph() {
    val navController = rememberNavController()
    NavHost(navController, startDestination = "dashboard") {
        composable("dashboard") { PantallaDashboard(navController) }
        composable("gps")       { PantallaGPS() }
        composable("sensores")  { PantallaSensores() }
        composable("hardware")  { DashboardHardware() }
        composable("ajustes")   { PantallaAjustes() }
        composable("tareas")    { PantallaTareas() }    // ← nuevo (p19)
    }
}
```

> `MainActivity.kt` no cambia — sigue llamando a `TechDashNavGraph()`.

**¿Qué deberías ver?**
- ➕ El botón flotante abre el diálogo — escribe un título, elige prioridad y pulsa **Agregar**; la tarea aparece al instante.
- 🔍 La barra de búsqueda filtra las tareas en tiempo real mientras escribes.
- ☑️ Al marcar una tarea como completada queda tachada; mata la app y vuelve a abrirla — **sigue marcada** (Room la persiste).
- 🗑️ Al eliminar aparece el Snackbar "Deshacer"; pulsarlo restaura la tarea.
- 🎨 Las tarjetas se colorean según la prioridad: gris (baja), azul (media), rojo (alta).

> 💡 Si KSP falla al compilar: asegúrate de que la versión del plugin KSP coincide exactamente con la versión de Kotlin del proyecto.
> Si `AppDatabase` lanza error de migración: usa `.fallbackToDestructiveMigration()` durante el desarrollo (elimina y recrea la BD).

### 🧪 Prueba esto

> **Reto 1 — Campo fecha de vencimiento:** Añade el campo `fechaVencimiento: Long? = null` a `TareaEntity`. Incrementa `version` a 2 y añade una `Migration(1, 2)` con `ALTER TABLE tareas ADD COLUMN fechaVencimiento INTEGER`. Muestra la fecha en `ItemTarea` si no es nula. Verifica que al migrar, las tareas antiguas siguen intactas.
>
> **Reto 2 — Filtro combinado:** Añade `FiltroTareas.ALTA` al enum y su caso en el `flatMapLatest` del ViewModel. Muestra el chip "Alta" en la `LazyRow` de filtros. El filtro debe combinarse con la búsqueda: si el usuario filtra por "Alta" y además escribe en la búsqueda, ¿qué debería pasar? Impleméntalo.
>
> **Reflexión:** `combine + flatMapLatest + stateIn` es una cadena de tres operadores. ¿Por qué no es suficiente un simple `if` encadenado? ¿Qué problema habría si cambias el filtro mientras hay una búsqueda activa y no usas `flatMapLatest`?

---

## Ejercicios propuestos

1. **Migración de Room** — Añade el campo `fechaVencimiento: Long?` a
   `TareaEntity`. Incrementa `version` a 2 y crea una `Migration(1, 2)`
   con `ALTER TABLE tareas ADD COLUMN fechaVencimiento INTEGER`.
   Verifica que los datos existentes se preservan.

2. **DataStore con Proto** — Investiga `DataStore<UserPreferences>` con
   Protocol Buffers. Crea un `.proto` con campos `nombre`, `modoOscuro` y
   `idioma`. Compara ventajas sobre Preferences DataStore (tipado fuerte,
   sin claves string).

3. **Sincronización offline** — Extiende `TareasViewModel` con una función
   `sincronizar()` que lea las tareas de la API de la página 13, las inserte
   en Room y muestre un `Snackbar` con cuántas tareas nuevas se añadieron.
   La pantalla debe seguir funcionando sin conexión.

4. **Búsqueda FTS** — Room soporta Full-Text Search con `@Fts4`. Crea una
   entidad virtual `TareaFts` que indexe `titulo` y `descripcion`. Compara
   el rendimiento de `LIKE '%query%'` vs `MATCH 'query*'` con 1000 tareas.

---

## Resumen de la página 19

- `DataStore` reemplaza a `SharedPreferences` — es asíncrono, seguro con coroutines y expone datos como `Flow`.
- Las claves de `DataStore` se declaran con `booleanPreferencesKey`, `stringPreferencesKey`, etc. — tipado garantizado en compilación.
- `context.dataStore.edit { prefs -> }` es la única forma de escribir en DataStore — transaccional y seguro en concurrencia.
- Room divide responsabilidades en tres anotaciones: `@Entity` define la tabla, `@Dao` define las consultas, `@Database` conecta todo.
- Las consultas `@Query` que devuelven `Flow` hacen que la UI se actualice automáticamente cuando cambian los datos — sin necesidad de recargar manualmente.
- `combine + flatMapLatest + stateIn` es el patrón para combinar múltiples Flows en un único `StateFlow` de estado de UI.
- `SharingStarted.WhileSubscribed(5_000)` mantiene el Flow activo 5 segundos después de que la UI lo deje de observar — evita recargar al rotar la pantalla.
- La estrategia offline-first usa Room como fuente de verdad: emite datos locales inmediatamente y sincroniza con la API en background.
- El `Snackbar` con `actionLabel = "Deshacer"` y `SnackbarResult.ActionPerformed` implementa la operación de deshacer sin estado adicional.

---

> **Siguiente página →** Página 20: Notificaciones push, audio con
> Media3, empaquetado APK/AAB y publicación en Google Play.