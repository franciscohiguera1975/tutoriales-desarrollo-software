# Curso Kotlin — Página 18
## Módulo 7 · Hardware y Sistema
### GPS, sensores del dispositivo y conectividad de red

---

## 🚀 Proyecto TechDash — Lo que construirás en esta página

En este módulo construimos **TechDash**, una app de dashboard que muestra información del hardware del dispositivo en tiempo real. Esta página añade tres módulos al proyecto:

```
TechDash
├── 📍 GPS              ← Parte 1 (esta página)
├── 📡 Sensores         ← Parte 2 (esta página)
├── 🌐 Red              ← Parte 3 (esta página)
└── 🎵 Media            ← Página 21
```

Al terminar esta página tendrás un `DashboardHardware` funcional que muestra coordenadas GPS en vivo, valores de sensores actualizados en tiempo real y un banner animado de conectividad. Construirás cada módulo de forma independiente y los ejecutarás antes de combinarlos al final.

### Archivos que crearás/modificarás en esta página

| | Archivo | Dónde crearlo |
|---|---------|---------------|
| 📄 Nuevo | `LocationRepository.kt` | paquete `ui/hardware/gps/` |
| 📄 Nuevo | `UbicacionViewModel.kt` | paquete `ui/hardware/gps/` |
| 📄 Nuevo | `PantallaGPS.kt` | paquete `ui/hardware/gps/` |
| 📄 Nuevo | `SensoresRepository.kt` | paquete `ui/hardware/sensores/` |
| 📄 Nuevo | `SensoresViewModel.kt` | paquete `ui/hardware/sensores/` |
| 📄 Nuevo | `PantallaSensores.kt` | paquete `ui/hardware/sensores/` |
| 📄 Nuevo | `ConectividadRepository.kt` | paquete `ui/hardware/red/` |
| 📄 Nuevo | `BannerConectividad.kt` | paquete `ui/hardware/red/` |
| ✏️ Modifica | `AndroidManifest.xml` | `app/src/main/` — permisos de ubicación y red |
| ✏️ Modifica | `build.gradle.kts` | módulo `app` — dependencias de Location |

---

## Dependencias y permisos

```kotlin
// build.gradle.kts
dependencies {
    // Ubicación
    implementation("com.google.android.gms:play-services-location:21.3.0")

    // Soporte de coroutines para Tasks de Play Services — necesario para .await() en lastLocation
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-play-services:1.8.1")

    // viewModel() composable — necesario para UbicacionViewModel y SensoresViewModel en Compose
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.7")

    // Google Maps Compose (para visualizar la ubicación)
    implementation("com.google.maps.android:maps-compose:6.2.0")
}
```

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<!-- Solo si necesitas ubicación en segundo plano (requiere justificación especial) -->
<!-- <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" /> -->
```

---

## Parte 1 — GPS y geolocalización

### `LocationRepository` — abstracción del GPS

📁 **Nuevo archivo → `ui/hardware/gps/LocationRepository.kt`** (contiene `UbicacionDato` + `LocationRepository`)

```kotlin
package com.ute.techdash.ui.hardware.gps

import android.annotation.SuppressLint
import android.content.Context
import android.location.Location
import android.os.Looper
import com.google.android.gms.location.FusedLocationProviderClient
import com.google.android.gms.location.LocationCallback
import com.google.android.gms.location.LocationRequest
import com.google.android.gms.location.LocationResult
import com.google.android.gms.location.LocationServices
import com.google.android.gms.location.Priority
import kotlinx.coroutines.channels.awaitClose
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.callbackFlow
import kotlinx.coroutines.tasks.await

data class UbicacionDato(
    val latitud:   Double,
    val longitud:  Double,
    val precision: Float,    // metros de precisión
    val altitud:   Double,
    val velocidad: Float,    // m/s
    val timestamp: Long
) {
    val coordenadas get() = "${"%.6f".format(latitud)}, ${"%.6f".format(longitud)}"
    val precisionTexto get() = "±${"%.0f".format(precision)}m"
}

fun Location.toUbicacionDato() = UbicacionDato(
    latitud   = latitude,
    longitud  = longitude,
    precision = accuracy,
    altitud   = altitude,
    velocidad = speed,
    timestamp = time
)

class LocationRepository(private val context: Context) {

    private val clienteUbicacion: FusedLocationProviderClient =
        LocationServices.getFusedLocationProviderClient(context)

    // Configuración de las actualizaciones
    private val peticionUbicacion = LocationRequest.Builder(
        Priority.PRIORITY_HIGH_ACCURACY,
        5_000L    // intervalo deseado: 5 segundos
    )
        .setMinUpdateIntervalMillis(2_000L)     // mínimo: 2 segundos
        .setMaxUpdateDelayMillis(10_000L)       // máximo delay: 10 segundos
        .setMinUpdateDistanceMeters(5f)         // solo si se movió 5 metros
        .build()

    // Flow de ubicaciones continuas
    @SuppressLint("MissingPermission")
    fun ubicacionFlow(): Flow<UbicacionDato> = callbackFlow {
        val callback = object : LocationCallback() {
            override fun onLocationResult(resultado: LocationResult) {
                resultado.lastLocation?.let { location ->
                    trySend(location.toUbicacionDato())
                }
            }
        }

        clienteUbicacion.requestLocationUpdates(
            peticionUbicacion,
            callback,
            Looper.getMainLooper()
        )

        // Cuando el Flow se cancela, eliminar el callback
        awaitClose {
            clienteUbicacion.removeLocationUpdates(callback)
        }
    }

    // Obtener la última ubicación conocida (instantánea)
    @SuppressLint("MissingPermission")
    suspend fun ultimaUbicacionConocida(): UbicacionDato? {
        return clienteUbicacion.lastLocation.await()?.toUbicacionDato()
    }
}
```

### `UbicacionViewModel`

📁 **Nuevo archivo → `ui/hardware/gps/UbicacionViewModel.kt`** (contiene `UbicacionState` + `UbicacionViewModel`)

```kotlin
package com.ute.techdash.ui.hardware.gps

import android.content.Context
import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.Job
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

data class UbicacionState(
    val ubicacion:      UbicacionDato? = null,
    val rastreando:     Boolean        = false,
    val error:          String?        = null,
    val historial:      List<UbicacionDato> = emptyList()
)

class UbicacionViewModel(
    private val repositorio: LocationRepository
) : ViewModel() {

    private val _state = MutableStateFlow(UbicacionState())
    val state: StateFlow<UbicacionState> = _state.asStateFlow()

    private var jobRastreo: Job? = null

    fun iniciarRastreo() {
        if (_state.value.rastreando) return
        _state.update { it.copy(rastreando = true, error = null) }

        jobRastreo = viewModelScope.launch {
            try {
                repositorio.ubicacionFlow()
                    .catch { e ->
                        _state.update { it.copy(error = e.message, rastreando = false) }
                    }
                    .collect { ubicacion ->
                        _state.update { estado ->
                            estado.copy(
                                ubicacion  = ubicacion,
                                historial  = (estado.historial + ubicacion).takeLast(50)
                            )
                        }
                    }
            } catch (e: Exception) {
                _state.update { it.copy(error = e.message, rastreando = false) }
            }
        }
    }

    fun detenerRastreo() {
        jobRastreo?.cancel()
        jobRastreo = null
        _state.update { it.copy(rastreando = false) }
    }

    fun limpiarHistorial() {
        _state.update { it.copy(historial = emptyList()) }
    }

    override fun onCleared() {
        super.onCleared()
        detenerRastreo()
    }

    companion object {
        fun factory(context: Context) = object : ViewModelProvider.Factory {
            @Suppress("UNCHECKED_CAST")
            override fun <T : ViewModel> create(modelClass: Class<T>): T =
                UbicacionViewModel(LocationRepository(context)) as T
        }
    }
}
```

### `PantallaGPS` — UI completa con permisos

📁 **Nuevo archivo → `ui/hardware/gps/PantallaGPS.kt`**

```kotlin
package com.ute.techdash.ui.hardware.gps

import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
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
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.DeleteSweep
import androidx.compose.material.icons.filled.LocationOff
import androidx.compose.material.icons.filled.LocationOn
import androidx.compose.material.icons.filled.PlayArrow
import androidx.compose.material.icons.filled.Stop
import androidx.compose.material3.Button
import androidx.compose.material3.ButtonDefaults
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.ElevatedCard
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.material3.MaterialTheme
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
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.ute.techdash.utils.PermisosHelper

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PantallaGPS() {
    val context = LocalContext.current
    val vm: UbicacionViewModel = viewModel(factory = UbicacionViewModel.factory(context))
    val state   by vm.state.collectAsStateWithLifecycle()

    // Verificar permiso de ubicación
    var tienePermiso by remember {
        mutableStateOf(
            PermisosHelper.tienePermiso(
                context, android.Manifest.permission.ACCESS_FINE_LOCATION
            )
        )
    }

    val lanzadorPermiso = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { resultados ->
        tienePermiso = resultados[android.Manifest.permission.ACCESS_FINE_LOCATION] == true ||
                       resultados[android.Manifest.permission.ACCESS_COARSE_LOCATION] == true
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("GPS y Ubicación", fontWeight = FontWeight.Bold) },
                actions = {
                    if (state.historial.isNotEmpty()) {
                        IconButton(onClick = { vm.limpiarHistorial() }) {
                            Icon(Icons.Default.DeleteSweep, "Limpiar historial")
                        }
                    }
                }
            )
        }
    ) { padding ->
        Column(
            modifier            = Modifier
                .padding(padding)
                .fillMaxSize()
                .padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            // Sin permiso
            if (!tienePermiso) {
                Card(
                    colors   = CardDefaults.cardColors(
                        containerColor = MaterialTheme.colorScheme.errorContainer
                    )
                ) {
                    Column(Modifier.padding(16.dp)) {
                        Text(
                            "Permiso de ubicación requerido",
                            fontWeight = FontWeight.SemiBold,
                            color      = MaterialTheme.colorScheme.onErrorContainer
                        )
                        Spacer(Modifier.height(8.dp))
                        Button(onClick = {
                            lanzadorPermiso.launch(arrayOf(
                                android.Manifest.permission.ACCESS_FINE_LOCATION,
                                android.Manifest.permission.ACCESS_COARSE_LOCATION
                            ))
                        }) { Text("Conceder permiso") }
                    }
                }
                return@Column
            }

            // Tarjeta de ubicación actual
            ElevatedCard(Modifier.fillMaxWidth()) {
                Column(Modifier.padding(16.dp)) {
                    Row(
                        Modifier.fillMaxWidth(),
                        horizontalArrangement = Arrangement.SpaceBetween,
                        verticalAlignment     = Alignment.CenterVertically
                    ) {
                        Text("Ubicación actual", style = MaterialTheme.typography.titleMedium,
                            fontWeight = FontWeight.Bold)
                        Icon(
                            if (state.rastreando) Icons.Default.LocationOn
                            else                  Icons.Default.LocationOff,
                            contentDescription = null,
                            tint = if (state.rastreando) MaterialTheme.colorScheme.primary
                                   else MaterialTheme.colorScheme.onSurfaceVariant
                        )
                    }

                    Spacer(Modifier.height(12.dp))

                    if (state.ubicacion != null) {
                        val ub = state.ubicacion!!
                        InfoRow("Coordenadas", ub.coordenadas)
                        InfoRow("Precisión",   ub.precisionTexto)
                        InfoRow("Altitud",     "${"%.1f".format(ub.altitud)}m")
                        InfoRow("Velocidad",   "${"%.1f".format(ub.velocidad * 3.6)} km/h")
                    } else {
                        Text(
                            if (state.rastreando) "Obteniendo ubicación..."
                            else                  "Sin datos de ubicación",
                            color = MaterialTheme.colorScheme.onSurfaceVariant
                        )
                    }
                }
            }

            // Error
            state.error?.let { msg ->
                Card(colors = CardDefaults.cardColors(
                    containerColor = MaterialTheme.colorScheme.errorContainer)
                ) {
                    Text(msg, Modifier.padding(12.dp),
                        color = MaterialTheme.colorScheme.onErrorContainer)
                }
            }

            // Botón de rastreo
            Button(
                onClick  = {
                    if (state.rastreando) vm.detenerRastreo()
                    else                  vm.iniciarRastreo()
                },
                modifier = Modifier.fillMaxWidth().height(50.dp),
                colors   = if (state.rastreando)
                    ButtonDefaults.buttonColors(containerColor = MaterialTheme.colorScheme.error)
                else
                    ButtonDefaults.buttonColors()
            ) {
                Icon(
                    if (state.rastreando) Icons.Default.Stop else Icons.Default.PlayArrow,
                    null
                )
                Spacer(Modifier.width(8.dp))
                Text(if (state.rastreando) "Detener rastreo" else "Iniciar rastreo")
            }

            // Historial
            if (state.historial.isNotEmpty()) {
                Text(
                    "Historial (${state.historial.size} puntos)",
                    style = MaterialTheme.typography.titleSmall,
                    fontWeight = FontWeight.SemiBold
                )
                LazyColumn(
                    modifier            = Modifier.weight(1f),
                    verticalArrangement = Arrangement.spacedBy(4.dp)
                ) {
                    items(state.historial.reversed()) { ub ->
                        Card(Modifier.fillMaxWidth()) {
                            Row(
                                Modifier.padding(horizontal = 12.dp, vertical = 8.dp),
                                horizontalArrangement = Arrangement.SpaceBetween
                            ) {
                                Text(ub.coordenadas,
                                    style = MaterialTheme.typography.bodySmall)
                                Text(ub.precisionTexto,
                                    style = MaterialTheme.typography.bodySmall,
                                    color = MaterialTheme.colorScheme.primary)
                            }
                        }
                    }
                }
            }
        }
    }
}

@Composable
private fun InfoRow(etiqueta: String, valor: String) {
    Row(
        Modifier
            .fillMaxWidth()
            .padding(vertical = 3.dp),
        horizontalArrangement = Arrangement.SpaceBetween
    ) {
        Text(etiqueta, color = MaterialTheme.colorScheme.onSurfaceVariant,
            style = MaterialTheme.typography.bodySmall)
        Text(valor, fontWeight = FontWeight.Medium,
            style = MaterialTheme.typography.bodySmall)
    }
}
```

---

### ▶ Checkpoint 1 — Ejecuta la app con GPS

1. Añade `LocationRepository` y `UbicacionViewModel` al proyecto
2. Añade `PantallaGPS()` como pantalla en tu `NavHost` (o directamente como contenido del `Activity`)
3. Ejecuta en un **dispositivo físico** o en el emulador con ubicación simulada

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código:

```kotlin
package com.ute.techdash

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.ute.techdash.ui.hardware.gps.PantallaGPS
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                PantallaGPS()
            }
        }
    }
}
```

**¿Qué deberías ver?**
- Solicitud de permiso de ubicación al abrir la app
- Al pulsar "Iniciar rastreo": en 2–5 segundos aparecen coordenadas reales
- Precisión en metros (`±Xm`) que mejora mientras el GPS se fija
- Historial acumulando filas cada ~5 segundos

> 💡 **Emulador:** ve a `Extended controls (···) → Location → Send` para enviar una ubicación simulada. Pulsa `Routes` para simular movimiento continuo.

---

> ### 🧪 Prueba esto — Intervalo de actualización
>
> El intervalo de actualización está configurado en `LocationRepository`:
>
> ```kotlin
> LocationRequest.Builder(
>     Priority.PRIORITY_HIGH_ACCURACY,
>     5_000L    // ← intervalo deseado en milisegundos
> )
>     .setMinUpdateDistanceMeters(5f)   // ← solo actualiza si se movió 5m
> ```
>
> **Reto 1:** Cambia el intervalo a `2_000L` y el umbral de distancia a `1f`. Ejecuta la app y observa si el historial crece más rápido.
>
> **Reto 2:** Ahora ponlo en `200L`. ¿La UI se congela? ¿Por qué no o sí? (Pista: el `collect` corre en un coroutine del `viewModelScope`, no en el hilo principal)
>
> **Reflexión:** ¿Por qué `setMinUpdateDistanceMeters` puede ser más importante que el intervalo de tiempo cuando el usuario está quieto? ¿Qué impacto tiene en la batería?

---

## Parte 2 — Sensores del dispositivo

📁 **Nuevo archivo → `ui/hardware/sensores/SensoresRepository.kt`** (contiene `DatoAcelerometro`, `DatoGiroscopio`, `DatoLuz` y `SensoresRepository`)

```kotlin
package com.ute.techdash.ui.hardware.sensores

import android.content.Context
import android.hardware.Sensor
import android.hardware.SensorEvent
import android.hardware.SensorEventListener
import android.hardware.SensorManager
import kotlinx.coroutines.channels.awaitClose
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.callbackFlow
import kotlinx.coroutines.flow.map

// Datos de cada sensor
data class DatoAcelerometro(val x: Float, val y: Float, val z: Float) {
    val magnitud get() = Math.sqrt((x*x + y*y + z*z).toDouble()).toFloat()
    val inclinacion get() = Math.toDegrees(Math.atan2(y.toDouble(), x.toDouble())).toFloat()
}

data class DatoGiroscopio(val x: Float, val y: Float, val z: Float)

data class DatoLuz(val lux: Float) {
    val descripcion get() = when {
        lux < 10    -> "Muy oscuro"
        lux < 100   -> "Interior tenue"
        lux < 1000  -> "Interior iluminado"
        lux < 10000 -> "Nublado exterior"
        else        -> "Luz solar directa"
    }
}

// Repositorio genérico de sensores
class SensoresRepository(context: Context) {

    private val sensorManager =
        context.getSystemService(Context.SENSOR_SERVICE) as SensorManager

    // Flow para cualquier tipo de sensor
    private fun sensorFlow(tipoSensor: Int): Flow<FloatArray> = callbackFlow {
        val sensor = sensorManager.getDefaultSensor(tipoSensor)
            ?: run { close(Exception("Sensor $tipoSensor no disponible")); return@callbackFlow }

        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent) {
                trySend(event.values.clone())
            }
            override fun onAccuracyChanged(sensor: Sensor, accuracy: Int) {}
        }

        sensorManager.registerListener(
            listener,
            sensor,
            SensorManager.SENSOR_DELAY_UI   // ~60ms — suficiente para UI
        )

        awaitClose {
            sensorManager.unregisterListener(listener)
        }
    }

    fun acelerometroFlow(): Flow<DatoAcelerometro> =
        sensorFlow(Sensor.TYPE_ACCELEROMETER).map { v ->
            DatoAcelerometro(v[0], v[1], v[2])
        }

    fun giroscopioFlow(): Flow<DatoGiroscopio> =
        sensorFlow(Sensor.TYPE_GYROSCOPE).map { v ->
            DatoGiroscopio(v[0], v[1], v[2])
        }

    fun luzAmbienteFlow(): Flow<DatoLuz> =
        sensorFlow(Sensor.TYPE_LIGHT).map { v -> DatoLuz(v[0]) }

    fun sensorDisponible(tipo: Int): Boolean =
        sensorManager.getDefaultSensor(tipo) != null
}
```

### `SensoresViewModel` y pantalla

📁 **Nuevo archivo → `ui/hardware/sensores/SensoresViewModel.kt`** (contiene `SensoresState` + `SensoresViewModel`)

```kotlin
package com.ute.techdash.ui.hardware.sensores

import android.content.Context
import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.Job
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

data class SensoresState(
    val acelerometro: DatoAcelerometro? = null,
    val giroscopio:   DatoGiroscopio?   = null,
    val luz:          DatoLuz?           = null,
    val activo:       Boolean            = false
)

class SensoresViewModel(
    private val repositorio: SensoresRepository
) : ViewModel() {

    private val _state = MutableStateFlow(SensoresState())
    val state: StateFlow<SensoresState> = _state.asStateFlow()

    private var jobs = listOf<Job>()

    fun iniciar() {
        if (_state.value.activo) return
        _state.update { it.copy(activo = true) }

        jobs = listOf(
            viewModelScope.launch {
                repositorio.acelerometroFlow()
                    .catch { }
                    .collect { _state.update { s -> s.copy(acelerometro = it) } }
            },
            viewModelScope.launch {
                repositorio.giroscopioFlow()
                    .catch { }
                    .collect { _state.update { s -> s.copy(giroscopio = it) } }
            },
            viewModelScope.launch {
                repositorio.luzAmbienteFlow()
                    .catch { }
                    .collect { _state.update { s -> s.copy(luz = it) } }
            }
        )
    }

    fun detener() {
        jobs.forEach { it.cancel() }
        jobs = emptyList()
        _state.update { it.copy(activo = false) }
    }

    override fun onCleared() { super.onCleared(); detener() }

    companion object {
        fun factory(context: Context) = object : ViewModelProvider.Factory {
            @Suppress("UNCHECKED_CAST")
            override fun <T : ViewModel> create(modelClass: Class<T>): T =
                SensoresViewModel(SensoresRepository(context)) as T
        }
    }
}
```

📁 **Nuevo archivo → `ui/hardware/sensores/PantallaSensores.kt`**

```kotlin
package com.ute.techdash.ui.hardware.sensores

import androidx.compose.foundation.background
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
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.layout.ColumnScope
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Autorenew
import androidx.compose.material.icons.filled.LightMode
import androidx.compose.material.icons.filled.ScreenRotation
import androidx.compose.material3.ElevatedCard
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Icon
import androidx.compose.material3.LinearProgressIndicator
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.material3.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.runtime.DisposableEffect
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PantallaSensores() {
    val context = LocalContext.current
    val vm: SensoresViewModel = viewModel(factory = SensoresViewModel.factory(context))
    val state by vm.state.collectAsStateWithLifecycle()

    // Iniciar/detener con el ciclo de vida del composable
    DisposableEffect(Unit) {
        vm.iniciar()
        onDispose { vm.detener() }
    }

    Scaffold(
        topBar = { TopAppBar(title = { Text("Sensores", fontWeight = FontWeight.Bold) }) }
    ) { padding ->
        LazyColumn(
            modifier        = Modifier.padding(padding).fillMaxSize(),
            contentPadding  = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            // Acelerómetro
            item {
                TarjetaSensor(
                    titulo  = "Acelerómetro",
                    icono   = Icons.Default.ScreenRotation,
                    activo  = state.activo
                ) {
                    state.acelerometro?.let { a ->
                        FilaValorSensor("X", a.x, "m/s²")
                        FilaValorSensor("Y", a.y, "m/s²")
                        FilaValorSensor("Z", a.z, "m/s²")
                        Spacer(Modifier.height(4.dp))
                        Text("Magnitud: ${"%.2f".format(a.magnitud)} m/s²",
                            style = MaterialTheme.typography.bodySmall,
                            color = MaterialTheme.colorScheme.primary)
                    } ?: Text("Sin datos", color = MaterialTheme.colorScheme.onSurfaceVariant)
                }
            }

            // Giroscopio
            item {
                TarjetaSensor(
                    titulo = "Giroscopio",
                    icono  = Icons.Default.Autorenew,
                    activo = state.activo
                ) {
                    state.giroscopio?.let { g ->
                        FilaValorSensor("Yaw   (Z)", g.z, "rad/s")
                        FilaValorSensor("Pitch (X)", g.x, "rad/s")
                        FilaValorSensor("Roll  (Y)", g.y, "rad/s")
                    } ?: Text("Sin datos", color = MaterialTheme.colorScheme.onSurfaceVariant)
                }
            }

            // Luz ambiente
            item {
                TarjetaSensor(
                    titulo = "Luz ambiente",
                    icono  = Icons.Default.LightMode,
                    activo = state.activo
                ) {
                    state.luz?.let { l ->
                        Text(
                            "${"%.1f".format(l.lux)} lux",
                            style      = MaterialTheme.typography.headlineSmall,
                            fontWeight = FontWeight.Bold,
                            color      = MaterialTheme.colorScheme.primary
                        )
                        Text(l.descripcion,
                            style = MaterialTheme.typography.bodyMedium,
                            color = MaterialTheme.colorScheme.onSurfaceVariant)
                        Spacer(Modifier.height(8.dp))
                        LinearProgressIndicator(
                            progress = { (l.lux / 100_000f).coerceIn(0f, 1f) },
                            modifier = Modifier.fillMaxWidth()
                        )
                    } ?: Text("Sin datos", color = MaterialTheme.colorScheme.onSurfaceVariant)
                }
            }
        }
    }
}

@Composable
private fun TarjetaSensor(
    titulo: String,
    icono:  ImageVector,
    activo: Boolean,
    contenido: @Composable ColumnScope.() -> Unit
) {
    ElevatedCard(Modifier.fillMaxWidth()) {
        Column(Modifier.padding(16.dp)) {
            Row(
                Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment     = Alignment.CenterVertically
            ) {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    Icon(icono, null, tint = MaterialTheme.colorScheme.primary)
                    Spacer(Modifier.width(8.dp))
                    Text(titulo, fontWeight = FontWeight.SemiBold)
                }
                if (activo) {
                    Box(
                        modifier = Modifier
                            .size(10.dp)
                            .clip(CircleShape)
                            .background(MaterialTheme.colorScheme.primary)
                    )
                }
            }
            Spacer(Modifier.height(12.dp))
            contenido()
        }
    }
}

@Composable
private fun FilaValorSensor(eje: String, valor: Float, unidad: String) {
    Row(
        Modifier.fillMaxWidth().padding(vertical = 2.dp),
        horizontalArrangement = Arrangement.SpaceBetween
    ) {
        Text(eje, color = MaterialTheme.colorScheme.onSurfaceVariant,
            style = MaterialTheme.typography.bodySmall,
            modifier = Modifier.width(80.dp))
        Text(
            "${"%.3f".format(valor)} $unidad",
            style      = MaterialTheme.typography.bodySmall,
            fontWeight = FontWeight.Medium
        )
    }
}
```

---

### ▶ Checkpoint 2 — Ejecuta la app con sensores

1. Añade `SensoresRepository` y `SensoresViewModel` al proyecto
2. Coloca `PantallaSensores()` como pantalla (puede ser en una tab o directamente)
3. Ejecuta en un **dispositivo físico** para ver datos reales

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código:

```kotlin
package com.ute.techdash

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.ute.techdash.ui.hardware.sensores.PantallaSensores
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                PantallaSensores()
            }
        }
    }
}
```

**¿Qué deberías ver?**
- Tres tarjetas con un punto verde indicando que los datos están llegando
- **Acelerómetro:** inclina el dispositivo y los ejes X/Y/Z cambian en tiempo real
- **Giroscopio:** gira el dispositivo y observa la velocidad angular
- **Luz:** cubre la cámara trasera con el dedo y el valor de lux baja drásticamente

> 💡 **Emulador:** los sensores están simulados. Ve a `Extended controls → Virtual sensors` para ajustar manualmente los valores de acelerómetro y rotación.

---

> ### 🧪 Prueba esto — Delay de sensores y un sensor nuevo
>
> El `SensoresRepository` usa `SensorManager.SENSOR_DELAY_UI` (~60ms). Existen otras opciones:
>
> | Constante | Frecuencia aprox. |
> |-----------|------------------|
> | `SENSOR_DELAY_UI` | ~60 ms |
> | `SENSOR_DELAY_GAME` | ~20 ms |
> | `SENSOR_DELAY_FASTEST` | ~5 ms |
>
> **Reto 1:** Cambia el delay a `SENSOR_DELAY_GAME` y observa si la UI se vuelve más fluida o empieza a tener problemas de rendimiento.
>
> **Reto 2:** Añade un cuarto sensor — el **barómetro** (`Sensor.TYPE_PRESSURE`). Muestra la presión en hectopascales (hPa). Si el dispositivo no tiene barómetro, el `sensorFlow` cierra con excepción — maneja ese caso mostrando "Sensor no disponible" en lugar de crashear.
>
> **Pista:** En `SensoresRepository` ya tienes `sensorDisponible(tipo)` para comprobarlo antes de suscribirte.

---

## Parte 3 — Conectividad de red

📁 **Nuevo archivo → `ui/hardware/red/ConectividadRepository.kt`** (contiene `EstadoRed` + `ConectividadRepository`)

```kotlin
package com.ute.techdash.ui.hardware.red

import android.content.Context
import android.net.ConnectivityManager
import android.net.Network
import android.net.NetworkCapabilities
import android.net.NetworkRequest
import kotlinx.coroutines.channels.awaitClose
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.callbackFlow
import kotlinx.coroutines.flow.distinctUntilChanged

enum class EstadoRed { CONECTADO_WIFI, CONECTADO_DATOS, SIN_CONEXION }

class ConectividadRepository(context: Context) {

    private val conectividadManager =
        context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager

    // Estado actual (instantáneo)
    fun estadoActual(): EstadoRed {
        val red     = conectividadManager.activeNetwork ?: return EstadoRed.SIN_CONEXION
        val caps    = conectividadManager.getNetworkCapabilities(red) ?: return EstadoRed.SIN_CONEXION
        return when {
            caps.hasTransport(NetworkCapabilities.TRANSPORT_WIFI)     -> EstadoRed.CONECTADO_WIFI
            caps.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR) -> EstadoRed.CONECTADO_DATOS
            else -> EstadoRed.SIN_CONEXION
        }
    }

    // Flow reactivo — emite cuando cambia la conectividad
    fun conectividadFlow(): Flow<EstadoRed> = callbackFlow {
        // Enviar estado inicial
        trySend(estadoActual())

        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                val caps = conectividadManager.getNetworkCapabilities(network)
                val estado = when {
                    caps?.hasTransport(NetworkCapabilities.TRANSPORT_WIFI)     == true
                        -> EstadoRed.CONECTADO_WIFI
                    caps?.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR) == true
                        -> EstadoRed.CONECTADO_DATOS
                    else -> EstadoRed.CONECTADO_DATOS
                }
                trySend(estado)
            }

            override fun onLost(network: Network) {
                trySend(EstadoRed.SIN_CONEXION)
            }

            override fun onCapabilitiesChanged(
                network:              Network,
                networkCapabilities:  NetworkCapabilities
            ) {
                val estado = when {
                    networkCapabilities.hasTransport(NetworkCapabilities.TRANSPORT_WIFI)     ->
                        EstadoRed.CONECTADO_WIFI
                    networkCapabilities.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR) ->
                        EstadoRed.CONECTADO_DATOS
                    else -> EstadoRed.SIN_CONEXION
                }
                trySend(estado)
            }
        }

        val peticion = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()

        conectividadManager.registerNetworkCallback(peticion, callback)

        awaitClose {
            conectividadManager.unregisterNetworkCallback(callback)
        }
    }.distinctUntilChanged()   // solo emite cuando realmente cambia
}
```

### Banner de conectividad — componente reutilizable

📁 **Nuevo archivo → `ui/hardware/red/BannerConectividad.kt`** (contiene `BannerConectividad`)

```kotlin
package com.ute.techdash.ui.hardware.red

import androidx.compose.animation.AnimatedVisibility
import androidx.compose.animation.fadeIn
import androidx.compose.animation.fadeOut
import androidx.compose.animation.slideInVertically
import androidx.compose.animation.slideOutVertically
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Wifi
import androidx.compose.material.icons.filled.WifiOff
import androidx.compose.material3.Icon
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle

@Composable
fun BannerConectividad(
    repositorio: ConectividadRepository
) {
    val estadoRed by repositorio.conectividadFlow()
        .collectAsStateWithLifecycle(
            initialValue = repositorio.estadoActual()
        )

    AnimatedVisibility(
        visible = estadoRed == EstadoRed.SIN_CONEXION,
        enter   = slideInVertically { -it } + fadeIn(),
        exit    = slideOutVertically { -it } + fadeOut()
    ) {
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .background(MaterialTheme.colorScheme.errorContainer)
                .padding(horizontal = 16.dp, vertical = 8.dp),
            verticalAlignment     = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.Center
        ) {
            Icon(
                Icons.Default.WifiOff,
                contentDescription = null,
                tint     = MaterialTheme.colorScheme.onErrorContainer,
                modifier = Modifier.size(18.dp)
            )
            Spacer(Modifier.width(8.dp))
            Text(
                "Sin conexión a internet",
                color = MaterialTheme.colorScheme.onErrorContainer,
                style = MaterialTheme.typography.labelLarge
            )
        }
    }

    // Indicador positivo cuando recupera conexión
    var mostrarConectado by remember { mutableStateOf(false) }
    var ultimoEstado by remember { mutableStateOf(estadoRed) }

    LaunchedEffect(estadoRed) {
        if (ultimoEstado == EstadoRed.SIN_CONEXION && estadoRed != EstadoRed.SIN_CONEXION) {
            mostrarConectado = true
            kotlinx.coroutines.delay(2000)
            mostrarConectado = false
        }
        ultimoEstado = estadoRed
    }

    AnimatedVisibility(
        visible = mostrarConectado,
        enter   = slideInVertically { -it } + fadeIn(),
        exit    = slideOutVertically { -it } + fadeOut()
    ) {
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .background(MaterialTheme.colorScheme.primaryContainer)
                .padding(horizontal = 16.dp, vertical = 8.dp),
            verticalAlignment     = Alignment.CenterVertically,
            horizontalArrangement = Arrangement.Center
        ) {
            Icon(
                Icons.Default.Wifi,
                contentDescription = null,
                tint     = MaterialTheme.colorScheme.onPrimaryContainer,
                modifier = Modifier.size(18.dp)
            )
            Spacer(Modifier.width(8.dp))
            Text(
                "Conexión restaurada",
                color = MaterialTheme.colorScheme.onPrimaryContainer,
                style = MaterialTheme.typography.labelLarge
            )
        }
    }
}

```

> 💡 **Uso:** envuelve el contenido de cualquier pantalla con `BannerConectividad` dentro de una `Column`:
> ```kotlin
> Column {
>     BannerConectividad(repositorioRed)
>     Scaffold(/* ... */) { paddingValues ->
>         /* contenido de la pantalla */
>     }
> }
> ```

---

### ▶ Checkpoint 3 — Ejecuta la app con el banner de red

1. Añade el permiso `ACCESS_NETWORK_STATE` al `AndroidManifest.xml`
2. Crea una instancia de `ConectividadRepository` con `LocalContext.current` y pásala a `BannerConectividad` dentro de una `Column` junto a tu `Scaffold`
3. Ejecuta en un dispositivo físico

📁 **Reemplaza TODO el contenido de `MainActivity.kt`** con este código:

```kotlin
package com.ute.techdash

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.Column
import androidx.compose.runtime.remember
import androidx.compose.ui.platform.LocalContext
import com.ute.techdash.ui.hardware.red.BannerConectividad
import com.ute.techdash.ui.hardware.red.ConectividadRepository
import com.ute.techdash.ui.hardware.sensores.PantallaSensores
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                val context = LocalContext.current
                val repositorioRed = remember { ConectividadRepository(context) }
                Column {
                    BannerConectividad(repositorioRed)
                    PantallaSensores()    // sustituye por PantallaGPS() si prefieres
                }
            }
        }
    }
}
```

**¿Qué deberías ver?**
- Banner rojo deslizándose desde arriba cuando desactivas el WiFi (o activas el modo avión)
- Banner verde aparece 2 segundos cuando la conexión se restaura, y luego desaparece solo
- Ambas animaciones son fluidas gracias a `slideInVertically`

---

> ### 🧪 Prueba esto — Distinguir WiFi de datos móviles
>
> El `BannerConectividad` actual solo reacciona al estado `SIN_CONEXION`. Muchas apps también cambian el comportamiento cuando el usuario pasa de WiFi a datos móviles (para ahorrar batería y datos).
>
> **Reto:** Modifica `BannerConectividad` para que, cuando `estadoRed == EstadoRed.CONECTADO_DATOS`, muestre un banner informativo de color `tertiaryContainer` con el ícono `Icons.Default.SignalCellularAlt` y el texto "Usando datos móviles".
>
> El banner de datos debe ser **persistente** (no desaparece solo) y tiene menos urgencia visual que el banner de sin conexión.
>
> **Reflexión:** ¿Por qué usamos `.distinctUntilChanged()` al final del `conectividadFlow()`? Prueba a quitarlo y activa/desactiva el WiFi rápidamente varias veces — ¿qué ves diferente?

---

## Programa completo — dashboard de hardware

```kotlin
// 📄 Programa completo → ui/hardware/DashboardHardware.kt
package com.ute.techdash.ui.hardware

import androidx.compose.foundation.layout.Arrangement
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
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.SignalCellularAlt
import androidx.compose.material.icons.filled.Wifi
import androidx.compose.material.icons.filled.WifiOff
import androidx.compose.material3.ElevatedCard
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Icon
import androidx.compose.material3.LinearProgressIndicator
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.material3.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.remember
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import com.ute.techdash.ui.hardware.red.BannerConectividad
import com.ute.techdash.ui.hardware.red.ConectividadRepository
import com.ute.techdash.ui.hardware.red.EstadoRed
import com.ute.techdash.ui.hardware.sensores.DatoAcelerometro
import com.ute.techdash.ui.hardware.sensores.DatoLuz
import com.ute.techdash.ui.hardware.sensores.SensoresRepository
import kotlinx.coroutines.flow.catch

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DashboardHardware() {
    val context = LocalContext.current
    val repositorioRed      = remember { ConectividadRepository(context) }
    val repositorioSensores = remember { SensoresRepository(context) }

    val estadoRed by repositorioRed.conectividadFlow()
        .collectAsStateWithLifecycle(initialValue = repositorioRed.estadoActual())

    val acelerometro by repositorioSensores.acelerometroFlow()
        .catch { emit(DatoAcelerometro(0f, 0f, 0f)) }
        .collectAsStateWithLifecycle(initialValue = null)

    val luz by repositorioSensores.luzAmbienteFlow()
        .catch { emit(DatoLuz(0f)) }
        .collectAsStateWithLifecycle(initialValue = null)

    Scaffold(
        topBar = {
            Column {
                BannerConectividad(repositorioRed)
                TopAppBar(
                    title = { Text("Dashboard Hardware", fontWeight = FontWeight.Bold) }
                )
            }
        }
    ) { padding ->
        LazyColumn(
            modifier        = Modifier.padding(padding).fillMaxSize(),
            contentPadding  = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            // Red
            item {
                ElevatedCard(Modifier.fillMaxWidth()) {
                    Row(
                        Modifier.padding(16.dp),
                        verticalAlignment     = Alignment.CenterVertically,
                        horizontalArrangement = Arrangement.spacedBy(12.dp)
                    ) {
                        Icon(
                            when (estadoRed) {
                                EstadoRed.CONECTADO_WIFI  -> Icons.Default.Wifi
                                EstadoRed.CONECTADO_DATOS -> Icons.Default.SignalCellularAlt
                                EstadoRed.SIN_CONEXION    -> Icons.Default.WifiOff
                            },
                            contentDescription = null,
                            tint = when (estadoRed) {
                                EstadoRed.SIN_CONEXION -> MaterialTheme.colorScheme.error
                                else                   -> MaterialTheme.colorScheme.primary
                            },
                            modifier = Modifier.size(36.dp)
                        )
                        Column {
                            Text("Red", fontWeight = FontWeight.SemiBold)
                            Text(
                                when (estadoRed) {
                                    EstadoRed.CONECTADO_WIFI  -> "WiFi conectado"
                                    EstadoRed.CONECTADO_DATOS -> "Datos móviles"
                                    EstadoRed.SIN_CONEXION    -> "Sin conexión"
                                },
                                style = MaterialTheme.typography.bodySmall,
                                color = MaterialTheme.colorScheme.onSurfaceVariant
                            )
                        }
                    }
                }
            }

            // Movimiento (acelerómetro simplificado)
            item {
                ElevatedCard(Modifier.fillMaxWidth()) {
                    Column(Modifier.padding(16.dp)) {
                        Text("Movimiento", fontWeight = FontWeight.SemiBold)
                        Spacer(Modifier.height(8.dp))
                        acelerometro?.let { a ->
                            val magnitud = a.magnitud
                            val descripcion = when {
                                magnitud < 1  -> "Quieto"
                                magnitud < 5  -> "Ligero movimiento"
                                magnitud < 15 -> "En movimiento"
                                else          -> "Movimiento intenso"
                            }
                            Text(descripcion, color = MaterialTheme.colorScheme.primary,
                                style = MaterialTheme.typography.titleMedium)
                            Text("${"%.2f".format(magnitud)} m/s²",
                                style = MaterialTheme.typography.bodySmall,
                                color = MaterialTheme.colorScheme.onSurfaceVariant)
                        } ?: Text("Cargando...",
                            color = MaterialTheme.colorScheme.onSurfaceVariant)
                    }
                }
            }

            // Luz
            item {
                ElevatedCard(Modifier.fillMaxWidth()) {
                    Column(Modifier.padding(16.dp)) {
                        Text("Luz ambiente", fontWeight = FontWeight.SemiBold)
                        Spacer(Modifier.height(8.dp))
                        luz?.let { l ->
                            Text(l.descripcion, color = MaterialTheme.colorScheme.primary,
                                style = MaterialTheme.typography.titleMedium)
                            Text("${"%.1f".format(l.lux)} lux",
                                style = MaterialTheme.typography.bodySmall,
                                color = MaterialTheme.colorScheme.onSurfaceVariant)
                            Spacer(Modifier.height(6.dp))
                            LinearProgressIndicator(
                                progress = { (l.lux / 100_000f).coerceIn(0f, 1f) },
                                modifier = Modifier.fillMaxWidth()
                            )
                        } ?: Text("Cargando...",
                            color = MaterialTheme.colorScheme.onSurfaceVariant)
                    }
                }
            }
        }
    }
}
```

---

## 🏁 Integración final — Menú de navegación

Conecta todas las pantallas de este módulo con un menú principal elegante que usa **Navigation Compose**.

### Dependencia (si no la tienes aún)

En `build.gradle.kts` (app):

```kotlin
implementation("androidx.navigation:navigation-compose:2.7.7")
```

---

### 📁 Nuevo archivo → `ui/hardware/PantallaMenu.kt`

```kotlin
package com.ute.techdash.ui.hardware

import androidx.compose.animation.core.Spring
import androidx.compose.animation.core.animateFloatAsState
import androidx.compose.animation.core.spring
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
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.scale
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.navigation.NavController

private data class MenuDestino(
    val ruta: String,
    val titulo: String,
    val descripcion: String,
    val icono: ImageVector
)

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PantallaMenu(navController: NavController) {
    val destinos = listOf(
        MenuDestino("gps",       "GPS",        "Coordenadas y altitud en vivo",  Icons.Default.LocationOn),
        MenuDestino("sensores",  "Sensores",   "Acelerómetro y luz ambiente",    Icons.Default.Sensors),
        MenuDestino("dashboard", "Dashboard",  "Vista unificada de hardware",    Icons.Default.Dashboard)
    )

    Scaffold(
        topBar = {
            TopAppBar(
                title = {
                    Text("TechDash", fontWeight = FontWeight.Bold)
                },
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
            // ── Encabezado ──────────────────────────────────────────
            Surface(
                shape = MaterialTheme.shapes.extraLarge,
                color = MaterialTheme.colorScheme.primaryContainer,
                modifier = Modifier.size(80.dp)
            ) {
                Box(contentAlignment = Alignment.Center) {
                    Icon(
                        Icons.Default.Memory,
                        contentDescription = null,
                        modifier = Modifier.size(44.dp),
                        tint = MaterialTheme.colorScheme.onPrimaryContainer
                    )
                }
            }

            Spacer(Modifier.height(16.dp))

            Text(
                "Hardware en tiempo real",
                style = MaterialTheme.typography.headlineSmall,
                fontWeight = FontWeight.Bold,
                textAlign = TextAlign.Center
            )
            Text(
                "Selecciona un módulo para explorar",
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant,
                textAlign = TextAlign.Center
            )

            Spacer(Modifier.height(32.dp))

            // ── Cuadrícula de destinos ───────────────────────────────
            LazyVerticalGrid(
                columns = GridCells.Fixed(2),
                horizontalArrangement = Arrangement.spacedBy(12.dp),
                verticalArrangement   = Arrangement.spacedBy(12.dp)
            ) {
                items(destinos) { destino ->
                    TarjetaMenu(destino = destino) { navController.navigate(destino.ruta) }
                }
            }
        }
    }
}

@Composable
private fun TarjetaMenu(destino: MenuDestino, onClick: () -> Unit) {
    var presionado by remember { mutableStateOf(false) }
    val escala by animateFloatAsState(
        targetValue    = if (presionado) 0.93f else 1f,
        animationSpec  = spring(dampingRatio = Spring.DampingRatioMediumBouncy),
        label          = "escala_tarjeta"
    )

    ElevatedCard(
        onClick = {
            presionado = true
            onClick()
        },
        modifier = Modifier
            .fillMaxWidth()
            .scale(escala)
    ) {
        Column(
            modifier             = Modifier.padding(20.dp).fillMaxWidth(),
            horizontalAlignment  = Alignment.CenterHorizontally,
            verticalArrangement  = Arrangement.spacedBy(10.dp)
        ) {
            Surface(
                shape    = MaterialTheme.shapes.large,
                color    = MaterialTheme.colorScheme.secondaryContainer,
                modifier = Modifier.size(56.dp)
            ) {
                Box(contentAlignment = Alignment.Center) {
                    Icon(
                        destino.icono,
                        contentDescription = null,
                        modifier = Modifier.size(30.dp),
                        tint = MaterialTheme.colorScheme.onSecondaryContainer
                    )
                }
            }
            Text(
                destino.titulo,
                fontWeight = FontWeight.SemiBold,
                style      = MaterialTheme.typography.titleMedium
            )
            Text(
                destino.descripcion,
                style     = MaterialTheme.typography.bodySmall,
                color     = MaterialTheme.colorScheme.onSurfaceVariant,
                textAlign = TextAlign.Center
            )
        }
    }
}
```

---

### 📁 Reemplaza `MainActivity.kt` con navegación completa

```kotlin
package com.ute.techdash

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import com.ute.techdash.ui.hardware.DashboardHardware
import com.ute.techdash.ui.hardware.PantallaMenu
import com.ute.techdash.ui.hardware.gps.PantallaGPS
import com.ute.techdash.ui.hardware.sensores.PantallaSensores
import com.ute.techdash.ui.theme.TechDashTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TechDashTheme {
                val navController = rememberNavController()
                NavHost(navController = navController, startDestination = "menu") {
                    composable("menu")      { PantallaMenu(navController) }
                    composable("gps")       { PantallaGPS() }
                    composable("sensores")  { PantallaSensores() }
                    composable("dashboard") { DashboardHardware() }
                }
            }
        }
    }
}
```

### ▶ Checkpoint final — Ejecuta la app con navegación

1. Crea `PantallaMenu.kt` en `ui/hardware/`
2. Reemplaza `MainActivity.kt`
3. Ejecuta → verás el menú con tres tarjetas
4. Toca cada tarjeta y confirma que navega a la pantalla correcta
5. Presiona el botón atrás del sistema para volver al menú

**¿Qué deberías ver?**
- Menú con ícono central grande y cuadrícula 2×2 de tarjetas elevadas
- Al tocar "GPS" → `PantallaGPS` con coordenadas en vivo
- Al tocar "Sensores" → `PantallaSensores` con acelerómetro y luz
- Al tocar "Dashboard" → `DashboardHardware` con banner de red + cards combinados

> 🧪 **Prueba esto**
>
> **Reto 1:** Añade una cuarta tarjeta "Red" que navegue a una pantalla que solo muestre `BannerConectividad` + un `Text` con el estado actual. ¿Qué ruta necesitas agregar al `NavHost`?
>
> **Reto 2:** Añade un `AnimatedContentTransition` al `NavHost` para que las pantallas entren con `fadeIn() + slideInHorizontally()`. Busca `enterTransition` en la documentación de `composable()`.
>
> **Reflexión:** ¿Por qué `presionado` en `TarjetaMenu` no regresa a `false` después de navegar? ¿Cómo lo corregirías?

---

## Ejercicios propuestos

1. **Detector de sacudida** — Usando el `acelerometroFlow`, detecta cuando la
   magnitud supera 25 m/s² durante más de 100ms. Cuenta las sacudidas y muestra
   el total en pantalla. Usa `debounce(300)` para evitar múltiples conteos
   de la misma sacudida.

2. **Historial de ubicaciones en mapa** — Instala `maps-compose` y dibuja la
   ruta del historial del `UbicacionViewModel` como una `Polyline` sobre
   `GoogleMap`. Añade un marcador en la posición actual.

3. **Modo offline** — Integra `BannerConectividad` con el `ProductosViewModel`
   de la página 13. Cuando `estadoRed == SIN_CONEXION`, muestra los datos
   cacheados en `StateFlow` y deshabilita el botón de recarga con un `Tooltip`
   que explique que no hay conexión.

4. **Brújula con giroscopio** — Usa el flujo del giroscopio para acumular la
   rotación en el eje Z y dibuja una aguja que apunte al norte. Usa
   `Canvas` de Compose con `drawLine` y `rotate` para renderizar la aguja.

---

## Resumen de la página 18

- `FusedLocationProviderClient` es la API recomendada para GPS — combina WiFi, red celular y GPS físico para dar la mejor ubicación disponible.
- `callbackFlow` convierte callbacks basados en listeners en `Flow` — patrón reutilizable para sensores, ubicación y red.
- `awaitClose { }` en un `callbackFlow` garantiza que el listener se desregistra cuando el Flow se cancela — evita fugas de memoria y batería.
- `SensorManager.SENSOR_DELAY_UI` (~60ms) es suficiente para actualizar la UI. `SENSOR_DELAY_FASTEST` es para algoritmos que necesitan máxima frecuencia.
- Los sensores deben desregistrarse cuando la app va al fondo — usa `DisposableEffect` o vincula el ViewModel al ciclo de vida con `iniciar`/`detener`.
- `ConnectivityManager.NetworkCallback` es la API moderna para detectar cambios de red — reemplaza a `BroadcastReceiver` con `CONNECTIVITY_ACTION`.
- `distinctUntilChanged()` en el Flow de conectividad evita emitir el mismo estado dos veces seguidas — importante para no mostrar el banner dos veces.
- El `BannerConectividad` con `AnimatedVisibility` + `slideInVertically` es el patrón estándar de Android para comunicar el estado de red al usuario.

---

> **Siguiente página →** Página 19: Almacenamiento local — `DataStore`,
> `Room` con coroutines y estrategia offline-first.