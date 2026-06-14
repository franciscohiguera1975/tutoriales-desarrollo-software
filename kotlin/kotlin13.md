# Curso Kotlin — Página 13 (Revisada)
## Módulo 6 · Jetpack Compose
### `ViewModel`, `StateFlow`, Navigation Compose y consumo de API REST

---

## ¿Por qué `ViewModel`?

Hasta ahora el estado vivía dentro de los composables con `remember`.
El problema: cuando el usuario rota la pantalla, el composable se recrea
y el estado se pierde.

```
remember { mutableStateOf(...) }   ← se pierde al rotar
ViewModel + StateFlow              ← sobrevive a rotaciones y cambios de config
```

Además, el `ViewModel` separa la lógica de negocio de la UI siguiendo
el principio de Responsabilidad Única (SRP de SOLID):

```
Composable       → describe CÓMO SE VE
ViewModel        → gestiona QUÉ SE MUESTRA y QUÉ OCURRE
Repositorio      → obtiene y guarda los datos (Red / BD)
```

Esta separación tiene beneficios concretos:
- El composable se vuelve más simple — solo recibe datos y emite eventos
- El `ViewModel` se puede probar sin UI (tests unitarios puros)
- El `Repositorio` puede cambiar la fuente de datos sin tocar la UI

---

## Proyecto de la página 13: Catálogo de Productos

Construimos una app que consume una API REST real de productos.
El proyecto evoluciona en **6 pasos**: primero el `ViewModel` con datos
locales, luego la navegación, y finalmente la integración con la API.

### Estructura del proyecto

```
app/src/main/java/com/tuapp/catalogo/
├── MainActivity.kt
├── model/
│   └── Producto.kt              ← data classes compartidas
├── network/
│   ├── ApiService.kt            ← interfaz Retrofit  (Paso 5)
│   └── RetrofitClient.kt        ← singleton Retrofit  (Paso 5)
├── repository/
│   └── ProductosRepository.kt   ← abstracción de datos (Paso 5)
├── viewmodel/
│   ├── ProductosViewModel.kt    ← ViewModel principal (Paso 1)
│   └── DetalleViewModel.kt      ← ViewModel de detalle (Paso 4)
├── ui/
│   ├── Paso01_ViewModel.kt      ← Paso 1: ViewModel básico con datos locales
│   ├── Paso02_UiState.kt        ← Paso 2: sealed class UiState + Loading/Error
│   ├── Paso03_Navigation.kt     ← Paso 3: Navigation Compose + NavHost
│   ├── Paso04_Detalle.kt        ← Paso 4: pantalla de detalle con argumentos
│   ├── Paso05_Retrofit.kt       ← Paso 5: API REST con Retrofit
│   └── Paso06_Completo.kt       ← Paso 6: app completa integrada
└── theme/
    └── Theme.kt
```

### `MainActivity.kt` — no cambia entre pasos

```kotlin
// MainActivity.kt
package com.tuapp.catalogo

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.material3.MaterialTheme
import com.tuapp.catalogo.ui.*

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                // ◀ CAMBIA AQUÍ para probar cada paso:
                // Paso01_ViewModelScreen()
                // Paso02_UiStateScreen()
                // Paso03_NavigationScreen()
                // Paso04_DetalleScreen()  ← solo para preview, la nav lo llama
                // Paso05_RetrofitScreen()
                Paso06_CompletoScreen()   // ← paso activo
            }
        }
    }
}
```

### Modelo de datos compartido

**Crea:** `model/Producto.kt`

```kotlin
// model/Producto.kt
package com.tuapp.catalogo.model

// Producto local — usado en los pasos 1 al 4
data class Producto(
    val id:        Int,
    val nombre:    String,
    val precio:    Double,
    val categoria: String,
    val stock:     Int,
    val activo:    Boolean = true
)

// Producto de la API — mapea el JSON de los pasos 5 y 6
data class ProductoApi(
    val id:            Int,
    val name:          String,
    val slug:          String,
    val price:         String,
    val stock:         Int,
    val is_active:     Boolean,
    val url_image:     String,
    val category_name: String
)

data class PaginatedResponse(
    val count:    Int,
    val next:     String?,
    val previous: String?,
    val results:  List<ProductoApi>
)

// Datos locales de muestra para los primeros pasos
val productosDeMuestra = listOf(
    Producto(1, "Teclado mecánico",  89.99,  "Periféricos", 15),
    Producto(2, "Monitor 27\"",      349.99, "Pantallas",   8),
    Producto(3, "Mouse inalámbrico", 29.99,  "Periféricos", 42, activo = false),
    Producto(4, "Auriculares BT",    149.99, "Audio",       23),
    Producto(5, "Webcam HD",         59.99,  "Cámaras",     11),
    Producto(6, "Hub USB-C",         39.99,  "Accesorios",  30),
    Producto(7, "SSD 1TB",           89.99,  "Almacenamiento", 19),
    Producto(8, "Mousepad XL",       24.99,  "Periféricos", 55),
)
```

---

## Configuración de dependencias

Agrega estas dependencias **antes de empezar** — algunas se usan desde
el Paso 1 y otras se incorporan gradualmente:

```kotlin
// build.gradle.kts (módulo app)
dependencies {
    // ── ViewModel + StateFlow ──────────────────────────────────────────────
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.7")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.7")

    // ── Navigation Compose (Paso 3) ────────────────────────────────────────
    implementation("androidx.navigation:navigation-compose:2.8.5")

    // ── Retrofit + Gson (Paso 5) ───────────────────────────────────────────
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-gson:2.11.0")

    // ── Coroutines (necesario para viewModelScope) ─────────────────────────
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")

    // ── Coil: carga de imágenes desde URL (Paso 5) ────────────────────────
    implementation("io.coil-kt:coil-compose:2.7.0")
}
```

Y en `AndroidManifest.xml`, permiso de internet para los pasos 5 y 6:

```xml
<!-- Dentro de <manifest>, fuera de <application> -->
<uses-permission android:name="android.permission.INTERNET" />
```

---

## `ViewModel` básico con `StateFlow`

### Concepto

`StateFlow` es un `Flow` de Kotlin que:
- Siempre tiene un valor actual (no está vacío)
- Emite el valor nuevo a todos los colectores cuando cambia
- Es seguro para múltiples colectores simultáneos

El patrón de encapsulación es fundamental:

```kotlin
// ✅ Correcto: estado privado mutable, público de solo lectura
private val _productos = MutableStateFlow<List<Producto>>(emptyList())
val productos: StateFlow<List<Producto>> = _productos.asStateFlow()

// ❌ Incorrecto: exponer el estado mutable permite que cualquiera lo modifique
val productos = MutableStateFlow<List<Producto>>(emptyList())
```

`viewModelScope` es un `CoroutineScope` atado al ciclo de vida del
`ViewModel`. Las coroutines lanzadas aquí se cancelan automáticamente
cuando el `ViewModel` se destruye — sin fugas de memoria.

### Paso 1 — ViewModel con datos locales

**Crea:** `viewmodel/ProductosViewModel.kt`

```kotlin
// viewmodel/ProductosViewModel.kt
package com.tuapp.catalogo.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.tuapp.catalogo.model.Producto
import com.tuapp.catalogo.model.productosDeMuestra
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

class ProductosViewModel : ViewModel() {

    // Estado privado mutable — solo el ViewModel lo modifica
    private val _productos = MutableStateFlow<List<Producto>>(emptyList())
    // Estado público de solo lectura — la UI lo observa
    val productos: StateFlow<List<Producto>> = _productos.asStateFlow()

    private val _busqueda = MutableStateFlow("")
    val busqueda: StateFlow<String> = _busqueda.asStateFlow()

    private val _cargando = MutableStateFlow(false)
    val cargando: StateFlow<Boolean> = _cargando.asStateFlow()

    // Lista completa en memoria — la fuente de verdad local
    private var todosLosProductos: List<Producto> = emptyList()

    // init se ejecuta cuando el ViewModel se crea por primera vez
    init {
        cargarProductos()
    }

    private fun cargarProductos() {
        // viewModelScope.launch: coroutine segura atada al ViewModel
        viewModelScope.launch {
            _cargando.value = true
            delay(800) // simula latencia de red — en Paso 5 será una llamada real
            todosLosProductos = productosDeMuestra
            _productos.value = todosLosProductos
            _cargando.value = false
        }
    }

    fun actualizarBusqueda(query: String) {
        _busqueda.value = query
        // Filtra la lista local sin llamar a la red
        _productos.value = if (query.isBlank()) {
            todosLosProductos
        } else {
            todosLosProductos.filter {
                it.nombre.contains(query, ignoreCase = true) ||
                it.categoria.contains(query, ignoreCase = true)
            }
        }
    }

    fun recargar() { cargarProductos() }
}
```

**Crea:** `ui/Paso01_ViewModel.kt`

```kotlin
// ui/Paso01_ViewModel.kt
package com.tuapp.catalogo.ui

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.tuapp.catalogo.model.Producto
import com.tuapp.catalogo.viewmodel.ProductosViewModel

@Composable
fun Paso01_ViewModelScreen(
    // viewModel() crea o recupera el ViewModel del mismo ciclo de vida
    // Si el ViewModel ya existe (ej: tras rotar), devuelve el mismo
    vm: ProductosViewModel = viewModel()
) {
    // collectAsStateWithLifecycle: observa el StateFlow y recompone cuando cambia
    // Pausa la recolección cuando la UI no está activa (ahorra batería)
    val productos by vm.productos.collectAsStateWithLifecycle()
    val busqueda  by vm.busqueda.collectAsStateWithLifecycle()
    val cargando  by vm.cargando.collectAsStateWithLifecycle()

    Column(modifier = Modifier.fillMaxSize()) {
        Text(
            "Paso 1 · ViewModel + StateFlow",
            style    = MaterialTheme.typography.titleMedium,
            modifier = Modifier.padding(16.dp)
        )

        // El composable NO llama a la lógica directamente
        // Solo delega al ViewModel: vm.actualizarBusqueda(it)
        OutlinedTextField(
            value         = busqueda,
            onValueChange = { vm.actualizarBusqueda(it) },
            placeholder   = { Text("Buscar producto...") },
            leadingIcon   = { Icon(Icons.Default.Search, null) },
            trailingIcon  = {
                if (busqueda.isNotEmpty())
                    IconButton(onClick = { vm.actualizarBusqueda("") }) {
                        Icon(Icons.Default.Clear, "Limpiar")
                    }
            },
            singleLine = true,
            modifier   = Modifier
                .fillMaxWidth()
                .padding(horizontal = 16.dp, vertical = 8.dp)
        )

        // Loading state — mientras carga mostramos indicador
        if (cargando) {
            LinearProgressIndicator(modifier = Modifier.fillMaxWidth())
        }

        // Contador de resultados — derivado del StateFlow
        Text(
            "${productos.size} producto(s)",
            style    = MaterialTheme.typography.labelSmall,
            color    = MaterialTheme.colorScheme.onSurfaceVariant,
            modifier = Modifier.padding(horizontal = 16.dp, vertical = 4.dp)
        )

        LazyColumn(
            contentPadding      = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(productos, key = { it.id }) { producto ->
                TarjetaProductoSimple(producto = producto)
            }
        }
    }
}

// Composable puro — recibe datos, no accede al ViewModel directamente
// Esto facilita el testing y la reutilización
@Composable
internal fun TarjetaProductoSimple(
    producto: Producto,
    onClick:  () -> Unit = {}
) {
    ElevatedCard(
        onClick  = onClick,
        modifier = Modifier.fillMaxWidth()
    ) {
        Row(
            modifier              = Modifier.padding(16.dp),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment     = Alignment.CenterVertically
        ) {
            Column(modifier = Modifier.weight(1f)) {
                Text(producto.nombre, fontWeight = FontWeight.SemiBold,
                     style = MaterialTheme.typography.titleSmall)
                Text(producto.categoria,
                     style = MaterialTheme.typography.bodySmall,
                     color = MaterialTheme.colorScheme.onSurfaceVariant)
                Text("Stock: ${producto.stock}",
                     style = MaterialTheme.typography.bodySmall,
                     color = MaterialTheme.colorScheme.onSurfaceVariant)
            }
            Column(horizontalAlignment = Alignment.End) {
                Text(
                    "$${"%.2f".format(producto.precio)}",
                    style      = MaterialTheme.typography.titleMedium,
                    color      = MaterialTheme.colorScheme.primary,
                    fontWeight = FontWeight.Bold
                )
                if (!producto.activo) {
                    AssistChip(
                        onClick = {},
                        label   = { Text("Inactivo",
                                        style = MaterialTheme.typography.labelSmall) }
                    )
                }
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun Paso01_Preview() {
    MaterialTheme { Paso01_ViewModelScreen() }
}
```

**▶ Ejecuta:** descomenta `Paso01_ViewModelScreen()` en `MainActivity`.
- Observa el `LinearProgressIndicator` durante 800ms → desaparece al cargar
- Escribe en el buscador → la lista filtra sin llamar a la red
- **Rota el emulador** (`Ctrl+F11`) → los datos persisten. Con `remember` se habrían perdido
- Escribe algo, rota → la búsqueda persiste también (está en el ViewModel)

**Experimenta:**
- Cambia `delay(800)` por `delay(3000)` → observa 3 segundos de loading
- Agrega `fun toggleActivo(id: Int)` al ViewModel que invierta `producto.activo`
  y conéctalo a un `IconButton` en `TarjetaProductoSimple`

---

## `sealed class UiState` — modelar todos los estados posibles

### Concepto

Una operación de red puede estar en exactamente uno de tres estados:
cargando, exitosa o en error. La `sealed class` es la herramienta
perfecta para modelarlo porque el compilador garantiza que el `when`
cubra todos los casos:

```kotlin
sealed class UiState<out T> {
    object Loading                        : UiState<Nothing>()
    data class Success<T>(val data: T)    : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}
```

- `out T` — covarianza: `UiState<List<Producto>>` es un `UiState<Any>`
- `Nothing` — tipo que nunca tiene valor; perfecto para `Loading` y `Error`
  que no producen datos
- El `when` sobre una `sealed class` es **exhaustivo** — el compilador
  exige que cubras todos los casos

```kotlin
when (val estado = uiState) {
    is UiState.Loading  -> { /* mostrar spinner */ }
    is UiState.Error    -> { /* mostrar mensaje + retry */ }
    is UiState.Success  -> { /* usar estado.data */ }
    // No hace falta 'else' — el compilador sabe que ya cubriste todo
}
```

### Paso 2 — UiState completo con Loading, Error y Success

**Actualiza:** `viewmodel/ProductosViewModel.kt`

Reemplaza el contenido anterior con esta versión que usa `UiState`:

```kotlin
// viewmodel/ProductosViewModel.kt  (versión Paso 2)
package com.tuapp.catalogo.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.tuapp.catalogo.model.Producto
import com.tuapp.catalogo.model.productosDeMuestra
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

// sealed class en el mismo archivo del ViewModel — alternativa: archivo propio
sealed class UiState<out T> {
    object Loading                        : UiState<Nothing>()
    data class Success<T>(val data: T)    : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}

class ProductosViewModel : ViewModel() {

    // Un solo StateFlow con todos los estados posibles
    private val _uiState = MutableStateFlow<UiState<List<Producto>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<Producto>>> = _uiState.asStateFlow()

    private val _busqueda = MutableStateFlow("")
    val busqueda: StateFlow<String> = _busqueda.asStateFlow()

    private var todosLosProductos: List<Producto> = emptyList()

    init { cargarProductos() }

    fun cargarProductos(simularError: Boolean = false) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            delay(800)

            if (simularError) {
                // En el Paso 5 esto vendrá de una excepción real de Retrofit
                _uiState.value = UiState.Error("Error de conexión (simulado)")
                return@launch
            }

            try {
                todosLosProductos = productosDeMuestra
                _uiState.value = UiState.Success(todosLosProductos)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Error desconocido")
            }
        }
    }

    fun actualizarBusqueda(query: String) {
        _busqueda.value = query
        val actual = _uiState.value
        if (actual is UiState.Success) {
            _uiState.value = UiState.Success(
                if (query.isBlank()) todosLosProductos
                else todosLosProductos.filter {
                    it.nombre.contains(query, ignoreCase = true) ||
                    it.categoria.contains(query, ignoreCase = true)
                }
            )
        }
    }

    fun recargar() { cargarProductos() }
}
```

**Crea:** `ui/Paso02_UiState.kt`

```kotlin
// ui/Paso02_UiState.kt
package com.tuapp.catalogo.ui

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.tuapp.catalogo.viewmodel.ProductosViewModel
import com.tuapp.catalogo.viewmodel.UiState

@Composable
fun Paso02_UiStateScreen(vm: ProductosViewModel = viewModel()) {
    val uiState  by vm.uiState.collectAsStateWithLifecycle()
    val busqueda by vm.busqueda.collectAsStateWithLifecycle()

    // Estado local solo para el botón de simulación de error
    var simularError by remember { mutableStateOf(false) }

    Column(modifier = Modifier.fillMaxSize()) {
        Text(
            "Paso 2 · sealed class UiState",
            style    = MaterialTheme.typography.titleMedium,
            modifier = Modifier.padding(16.dp)
        )

        // Botones para probar los tres estados manualmente
        Row(
            modifier              = Modifier
                .fillMaxWidth()
                .padding(horizontal = 16.dp),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            OutlinedButton(
                onClick  = { simularError = false; vm.recargar() },
                modifier = Modifier.weight(1f)
            ) { Text("✓ Éxito") }

            OutlinedButton(
                onClick  = { simularError = true; vm.cargarProductos(simularError = true) },
                modifier = Modifier.weight(1f),
                colors   = ButtonDefaults.outlinedButtonColors(
                    contentColor = MaterialTheme.colorScheme.error
                )
            ) { Text("✗ Error") }
        }

        Spacer(Modifier.height(8.dp))

        OutlinedTextField(
            value         = busqueda,
            onValueChange = { vm.actualizarBusqueda(it) },
            placeholder   = { Text("Buscar...") },
            leadingIcon   = { Icon(Icons.Default.Search, null) },
            trailingIcon  = {
                if (busqueda.isNotEmpty())
                    IconButton(onClick = { vm.actualizarBusqueda("") }) {
                        Icon(Icons.Default.Clear, "Limpiar")
                    }
            },
            singleLine = true,
            modifier   = Modifier
                .fillMaxWidth()
                .padding(horizontal = 16.dp)
        )

        Spacer(Modifier.height(8.dp))

        // when sobre sealed class — el compilador garantiza exhaustividad
        when (val estado = uiState) {

            // ── Loading ────────────────────────────────────────────────────
            is UiState.Loading -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                        CircularProgressIndicator()
                        Spacer(Modifier.height(12.dp))
                        Text("Cargando productos...",
                             style = MaterialTheme.typography.bodyMedium,
                             color = MaterialTheme.colorScheme.onSurfaceVariant)
                    }
                }
            }

            // ── Error ──────────────────────────────────────────────────────
            is UiState.Error -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Column(
                        horizontalAlignment = Alignment.CenterHorizontally,
                        verticalArrangement = Arrangement.spacedBy(12.dp),
                        modifier            = Modifier.padding(32.dp)
                    ) {
                        Icon(Icons.Default.WifiOff, null,
                             Modifier.size(56.dp),
                             tint = MaterialTheme.colorScheme.error)
                        Text(
                            // estado.message está disponible porque el compilador
                            // sabe que 'estado' es UiState.Error aquí
                            text  = estado.message,
                            style = MaterialTheme.typography.bodyLarge,
                            color = MaterialTheme.colorScheme.error
                        )
                        Button(onClick = { vm.recargar() }) {
                            Icon(Icons.Default.Refresh, null)
                            Spacer(Modifier.width(8.dp))
                            Text("Reintentar")
                        }
                    }
                }
            }

            // ── Success ────────────────────────────────────────────────────
            is UiState.Success -> {
                // estado.data está disponible aquí como List<Producto>
                Text(
                    "${estado.data.size} producto(s)",
                    style    = MaterialTheme.typography.labelSmall,
                    color    = MaterialTheme.colorScheme.onSurfaceVariant,
                    modifier = Modifier.padding(horizontal = 16.dp, vertical = 4.dp)
                )
                LazyColumn(
                    contentPadding      = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
                    verticalArrangement = Arrangement.spacedBy(8.dp)
                ) {
                    items(estado.data, key = { it.id }) { producto ->
                        TarjetaProductoSimple(producto = producto)
                    }
                }
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun Paso02_Preview() {
    MaterialTheme { Paso02_UiStateScreen() }
}
```

**▶ Ejecuta:** descomenta `Paso02_UiStateScreen()`.
- Presiona "✗ Error" → aparece el estado de error con ícono y botón Reintentar
- Presiona Reintentar o "✓ Éxito" → carga correctamente
- Rota mientras carga → el estado de loading persiste (ViewModel sobrevive)

**Experimenta:**
- Agrega un estado `Empty` a `UiState` para cuando la búsqueda no da resultados
- Cambia `CircularProgressIndicator` por `LinearProgressIndicator` → ¿cuál UX prefieres?

---

## Navigation Compose

### Concepto

Navigation Compose gestiona el stack de pantallas con un grafo declarativo.
Los tres elementos clave son:

```
NavController   → controla la navegación (navegar, volver, etc.)
NavHost         → define el grafo completo de rutas
composable { } → declara cada pantalla dentro del NavHost
```

Las rutas son strings, pero las envolvemos en una `sealed class` para
evitar typos y centralizar todas las rutas en un solo lugar:

```kotlin
sealed class Ruta(val path: String) {
    object Lista   : Ruta("productos")
    object Detalle : Ruta("productos/{id}") {
        fun conId(id: Int) = "productos/$id"  // genera "productos/42"
    }
}
```

**Parámetros de navegación** — se declaran en la ruta con `{nombre}` y
se extraen del `backStackEntry.arguments`:

```kotlin
composable(
    route     = Ruta.Detalle.path,        // "productos/{id}"
    arguments = listOf(navArgument("id") { type = NavType.IntType })
) { backStackEntry ->
    val id = backStackEntry.arguments?.getInt("id") ?: return@composable
    PantallaDetalle(id = id)
}
```

**Opciones de navegación en `NavigationBar`** — evitan duplicar pantallas
en el back stack:

```kotlin
navController.navigate(ruta) {
    popUpTo(navController.graph.startDestinationId) { saveState = true }
    launchSingleTop = true   // no crea nueva instancia si ya está en el top
    restoreState    = true   // restaura el scroll y estado de la pantalla
}
```

### Paso 3 — Navigation Compose con NavHost

**Crea:** `ui/Paso03_Navigation.kt`

```kotlin
// ui/Paso03_Navigation.kt
package com.tuapp.catalogo.ui

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material.icons.outlined.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.navigation.*
import androidx.navigation.compose.*
import com.tuapp.catalogo.model.Producto
import com.tuapp.catalogo.viewmodel.ProductosViewModel
import com.tuapp.catalogo.viewmodel.UiState

// ── Definición de rutas ──────────────────────────────────────────────────────
sealed class Ruta(val path: String) {
    object Lista   : Ruta("productos")
    object Detalle : Ruta("productos/{id}") {
        // Función helper que genera la ruta con el id real
        fun conId(id: Int) = "productos/$id"
    }
    object Perfil  : Ruta("perfil")
}

// Modelo del ítem de NavigationBar — separar datos de presentación
data class ItemNav(
    val ruta:          Ruta,
    val etiqueta:      String,
    val iconoActivo:   ImageVector,
    val iconoInactivo: ImageVector
)

// ── Composable raíz del paso 3 ───────────────────────────────────────────────
@Composable
fun Paso03_NavigationScreen() {
    // rememberNavController crea el controlador atado a este composable
    val navController = rememberNavController()

    // currentBackStackEntryAsState: State que se actualiza en cada navegación
    val backStackEntry by navController.currentBackStackEntryAsState()
    val rutaActual     = backStackEntry?.destination?.route

    val itemsNav = listOf(
        ItemNav(Ruta.Lista,  "Productos", Icons.Filled.Inventory,    Icons.Outlined.Inventory),
        ItemNav(Ruta.Perfil, "Perfil",    Icons.Filled.AccountCircle, Icons.Outlined.AccountCircle),
    )

    // Solo mostramos la NavigationBar en pantallas de nivel raíz
    val rutasConBar = listOf(Ruta.Lista.path, Ruta.Perfil.path)

    Scaffold(
        topBar = {
            TopAppBar(
                title = {
                    // El título cambia según la ruta actual
                    Text(
                        when (rutaActual) {
                            Ruta.Lista.path   -> "Catálogo"
                            Ruta.Perfil.path  -> "Perfil"
                            else              -> "Detalle"
                        },
                        fontWeight = FontWeight.Bold
                    )
                },
                // Botón atrás — solo en pantallas de detalle
                navigationIcon = {
                    if (rutaActual !in rutasConBar) {
                        IconButton(onClick = { navController.popBackStack() }) {
                            Icon(Icons.Default.ArrowBack, "Volver")
                        }
                    }
                },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor    = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.onPrimaryContainer
                )
            )
        },
        bottomBar = {
            // NavigationBar visible solo en pantallas raíz
            if (rutaActual in rutasConBar) {
                NavigationBar {
                    itemsNav.forEach { item ->
                        val sel = rutaActual == item.ruta.path
                        NavigationBarItem(
                            selected = sel,
                            onClick  = {
                                navController.navigate(item.ruta.path) {
                                    // Evita acumular la misma pantalla en el stack
                                    popUpTo(navController.graph.startDestinationId) {
                                        saveState = true
                                    }
                                    launchSingleTop = true
                                    restoreState    = true
                                }
                            },
                            icon  = {
                                Icon(if (sel) item.iconoActivo else item.iconoInactivo,
                                     item.etiqueta)
                            },
                            label = { Text(item.etiqueta) }
                        )
                    }
                }
            }
        }
    ) { paddingValues ->
        // NavHost define el grafo completo de navegación
        NavHost(
            navController    = navController,
            startDestination = Ruta.Lista.path,
            modifier         = Modifier.padding(paddingValues)
        ) {
            // ── Ruta: lista de productos ───────────────────────────────────
            composable(Ruta.Lista.path) {
                // El ViewModel se comparte si se instancia con viewModel()
                // desde el mismo scope — aquí lo pasamos como parámetro
                val vm: ProductosViewModel = viewModel()
                val uiState  by vm.uiState.collectAsStateWithLifecycle()
                val busqueda by vm.busqueda.collectAsStateWithLifecycle()

                ListaProductosContent(
                    uiState       = uiState,
                    busqueda      = busqueda,
                    onBusqueda    = { vm.actualizarBusqueda(it) },
                    onRecargar    = { vm.recargar() },
                    onVerDetalle  = { id ->
                        // Navegamos pasando el id en la ruta
                        navController.navigate(Ruta.Detalle.conId(id))
                    }
                )
            }

            // ── Ruta: detalle — recibe 'id' como argumento ─────────────────
            composable(
                route     = Ruta.Detalle.path,
                arguments = listOf(
                    navArgument("id") { type = NavType.IntType }
                )
            ) { backStackEntry ->
                // Extraemos el argumento del back stack
                val id = backStackEntry.arguments?.getInt("id") ?: return@composable
                DetalleProductoSimple(
                    productoId = id,
                    onVolver   = { navController.popBackStack() }
                )
            }

            // ── Ruta: perfil ───────────────────────────────────────────────
            composable(Ruta.Perfil.path) {
                PantallaPerfilSimple()
            }
        }
    }
}

// ── Pantallas internas del Paso 3 ────────────────────────────────────────────

@Composable
private fun ListaProductosContent(
    uiState:      UiState<List<Producto>>,
    busqueda:     String,
    onBusqueda:   (String) -> Unit,
    onRecargar:   () -> Unit,
    onVerDetalle: (Int) -> Unit
) {
    Column(modifier = Modifier.fillMaxSize()) {
        OutlinedTextField(
            value         = busqueda,
            onValueChange = onBusqueda,
            placeholder   = { Text("Buscar...") },
            leadingIcon   = { Icon(Icons.Default.Search, null) },
            trailingIcon  = {
                if (busqueda.isNotEmpty())
                    IconButton(onClick = { onBusqueda("") }) {
                        Icon(Icons.Default.Clear, "Limpiar")
                    }
            },
            singleLine = true,
            modifier   = Modifier.fillMaxWidth().padding(16.dp)
        )

        when (val estado = uiState) {
            is UiState.Loading -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
            is UiState.Error -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                        Text(estado.message, color = MaterialTheme.colorScheme.error)
                        Spacer(Modifier.height(8.dp))
                        Button(onClick = onRecargar) { Text("Reintentar") }
                    }
                }
            }
            is UiState.Success -> {
                LazyColumn(
                    contentPadding      = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
                    verticalArrangement = Arrangement.spacedBy(8.dp)
                ) {
                    items(estado.data, key = { it.id }) { producto ->
                        TarjetaProductoSimple(
                            producto = producto,
                            // Al hacer click, navegamos al detalle con el id
                            onClick  = { onVerDetalle(producto.id) }
                        )
                    }
                }
            }
        }
    }
}

@Composable
private fun DetalleProductoSimple(productoId: Int, onVolver: () -> Unit) {
    // En el Paso 4 esto vendrá de un DetalleViewModel
    val producto = com.tuapp.catalogo.model.productosDeMuestra
        .find { it.id == productoId }

    Box(Modifier.fillMaxSize().padding(24.dp), contentAlignment = Alignment.Center) {
        if (producto == null) {
            Text("Producto #$productoId no encontrado")
        } else {
            Column(verticalArrangement = Arrangement.spacedBy(12.dp)) {
                Text(producto.nombre, style = MaterialTheme.typography.headlineSmall,
                     fontWeight = FontWeight.Bold)
                Text("Categoría: ${producto.categoria}")
                Text("Precio: $${"%.2f".format(producto.precio)}",
                     style = MaterialTheme.typography.titleLarge,
                     color = MaterialTheme.colorScheme.primary)
                Text("Stock: ${producto.stock}")
                OutlinedButton(onClick = onVolver, modifier = Modifier.fillMaxWidth()) {
                    Icon(Icons.Default.ArrowBack, null)
                    Spacer(Modifier.width(8.dp))
                    Text("Volver a la lista")
                }
            }
        }
    }
}

@Composable
private fun PantallaPerfilSimple() {
    Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        Column(horizontalAlignment = Alignment.CenterHorizontally,
               verticalArrangement = Arrangement.spacedBy(8.dp)) {
            Icon(Icons.Default.AccountCircle, null, Modifier.size(80.dp),
                 tint = MaterialTheme.colorScheme.primary)
            Text("Mi Perfil", style = MaterialTheme.typography.titleLarge,
                 fontWeight = FontWeight.Bold)
            Text("Usuario de ejemplo", color = MaterialTheme.colorScheme.onSurfaceVariant)
        }
    }
}

@Preview(showBackground = true)
@Composable
fun Paso03_Preview() {
    MaterialTheme { Paso03_NavigationScreen() }
}
```

**▶ Ejecuta:** descomenta `Paso03_NavigationScreen()`.
- Presiona cualquier tarjeta → navega al detalle con el id correcto
- Presiona la flecha atrás en la `TopAppBar` → regresa a la lista
- Navega a "Perfil" y vuelve → el scroll de la lista se conserva (`restoreState = true`)
- Rota en la pantalla de detalle → el argumento `id` persiste

**Experimenta:**
- Agrega una ruta `object Crear : Ruta("productos/crear")` y un FAB en la lista
  que navegue a ella
- Cambia `launchSingleTop = false` → presiona "Productos" varias veces → el
  botón Atrás requiere varios clicks para salir (se acumulan instancias)

---

## `ViewModel` de detalle con argumentos

### Concepto

Cada pantalla del grafo de navegación puede tener su propio `ViewModel`.
Para pasar el `id` del producto al `ViewModel` de detalle usamos
`SavedStateHandle` — un mapa persistente que Navigation rellena automáticamente
con los argumentos de la ruta:

```kotlin
class DetalleViewModel(
    savedStateHandle: SavedStateHandle   // inyectado por el framework
) : ViewModel() {
    // Lee el argumento 'id' de la ruta "productos/{id}"
    private val productoId: Int = checkNotNull(savedStateHandle["id"])
}
```

### Paso 4 — DetalleViewModel con SavedStateHandle

**Crea:** `viewmodel/DetalleViewModel.kt`

```kotlin
// viewmodel/DetalleViewModel.kt
package com.tuapp.catalogo.viewmodel

import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.tuapp.catalogo.model.Producto
import com.tuapp.catalogo.model.productosDeMuestra
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

class DetalleViewModel(
    // SavedStateHandle: mapa persistente que Navigation llena con los
    // argumentos de la ruta. Sobrevive a rotaciones y process death.
    savedStateHandle: SavedStateHandle
) : ViewModel() {

    // checkNotNull lanza excepción si el argumento no existe en la ruta
    // — falla rápido en lugar de comportamiento impredecible
    private val productoId: Int = checkNotNull(savedStateHandle["id"])

    private val _uiState = MutableStateFlow<UiState<Producto>>(UiState.Loading)
    val uiState: StateFlow<UiState<Producto>> = _uiState.asStateFlow()

    init { cargarDetalle() }

    private fun cargarDetalle() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            delay(400) // simula latencia
            val producto = productosDeMuestra.find { it.id == productoId }
            _uiState.value = if (producto != null) {
                UiState.Success(producto)
            } else {
                UiState.Error("Producto #$productoId no encontrado")
            }
        }
    }
}
```

**Crea:** `ui/Paso04_Detalle.kt`

```kotlin
// ui/Paso04_Detalle.kt
package com.tuapp.catalogo.ui

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.tuapp.catalogo.model.Producto
import com.tuapp.catalogo.viewmodel.DetalleViewModel
import com.tuapp.catalogo.viewmodel.UiState

// PantallaDetalle: el NavHost llama a este composable con el id
// El viewModel() aquí crea el DetalleViewModel con SavedStateHandle automático
@Composable
fun PantallaDetalle(
    onVolver: () -> Unit = {},
    vm:       DetalleViewModel = viewModel()
) {
    val uiState by vm.uiState.collectAsStateWithLifecycle()

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Detalle de producto") },
                navigationIcon = {
                    IconButton(onClick = onVolver) {
                        Icon(Icons.Default.ArrowBack, "Volver")
                    }
                },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor    = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.onPrimaryContainer
                )
            )
        }
    ) { paddingValues ->
        Box(
            modifier         = Modifier.padding(paddingValues).fillMaxSize(),
            contentAlignment = Alignment.Center
        ) {
            when (val estado = uiState) {
                is UiState.Loading -> CircularProgressIndicator()

                is UiState.Error -> Column(
                    horizontalAlignment = Alignment.CenterHorizontally,
                    verticalArrangement = Arrangement.spacedBy(12.dp)
                ) {
                    Icon(Icons.Default.Error, null, Modifier.size(48.dp),
                         tint = MaterialTheme.colorScheme.error)
                    Text(estado.message, color = MaterialTheme.colorScheme.error)
                    OutlinedButton(onClick = onVolver) { Text("Volver") }
                }

                is UiState.Success -> ContenidoDetalle(
                    producto = estado.data,
                    onVolver = onVolver
                )
            }
        }
    }
}

@Composable
private fun ContenidoDetalle(producto: Producto, onVolver: () -> Unit) {
    Column(
        modifier            = Modifier
            .fillMaxWidth()
            .padding(24.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        // Avatar grande con inicial
        Box(
            modifier         = Modifier
                .size(100.dp)
                .align(Alignment.CenterHorizontally),
            contentAlignment = Alignment.Center
        ) {
            Surface(
                shape  = RoundedCornerShape(16.dp),
                color  = MaterialTheme.colorScheme.primaryContainer,
                modifier = Modifier.fillMaxSize()
            ) {
                Box(contentAlignment = Alignment.Center) {
                    Text(
                        producto.nombre.first().uppercase(),
                        style      = MaterialTheme.typography.displayMedium,
                        fontWeight = FontWeight.Bold,
                        color      = MaterialTheme.colorScheme.onPrimaryContainer
                    )
                }
            }
        }

        Text(
            producto.nombre,
            style      = MaterialTheme.typography.headlineSmall,
            fontWeight = FontWeight.Bold,
            modifier   = Modifier.align(Alignment.CenterHorizontally)
        )

        HorizontalDivider()

        FilaDetalle("Categoría",  producto.categoria)
        FilaDetalle("Precio",     "$${"%.2f".format(producto.precio)}")
        FilaDetalle("Stock",      "${producto.stock} unidades")
        FilaDetalle("Estado",     if (producto.activo) "Activo ✓" else "Inactivo")

        Spacer(Modifier.height(8.dp))

        Button(
            onClick  = onVolver,
            modifier = Modifier.fillMaxWidth()
        ) {
            Icon(Icons.Default.ArrowBack, null)
            Spacer(Modifier.width(8.dp))
            Text("Volver a la lista")
        }
    }
}

@Composable
private fun FilaDetalle(etiqueta: String, valor: String) {
    Row(
        modifier              = Modifier.fillMaxWidth(),
        horizontalArrangement = Arrangement.SpaceBetween
    ) {
        Text(etiqueta,
             style = MaterialTheme.typography.bodyMedium,
             color = MaterialTheme.colorScheme.onSurfaceVariant)
        Text(valor,
             style      = MaterialTheme.typography.bodyMedium,
             fontWeight = FontWeight.SemiBold)
    }
}

// Preview independiente — usa id=1 de los datos de muestra
@Preview(showBackground = true)
@Composable
fun Paso04_Preview() {
    MaterialTheme {
        // No podemos previsualizar directamente PantallaDetalle porque necesita
        // SavedStateHandle — previewamos el contenido directamente
        ContenidoDetalle(
            producto = com.tuapp.catalogo.model.productosDeMuestra.first(),
            onVolver = {}
        )
    }
}
```

Para conectar `PantallaDetalle` al grafo de navegación, actualiza el
`composable(Ruta.Detalle.path)` en `Paso03_Navigation.kt`:

```kotlin
// Reemplaza DetalleProductoSimple(...) por:
composable(
    route     = Ruta.Detalle.path,
    arguments = listOf(navArgument("id") { type = NavType.IntType })
) {
    // Navigation pasa el SavedStateHandle automáticamente al DetalleViewModel
    PantallaDetalle(onVolver = { navController.popBackStack() })
}
```

**▶ Ejecuta:** activa `Paso03_NavigationScreen()` con esta actualización.
- Presiona una tarjeta → navega a la pantalla de detalle con `DetalleViewModel`
- Rota en la pantalla de detalle → el ViewModel persiste, no recarga

---

## Consumo de API REST con Retrofit

### Concepto

Retrofit transforma interfaces Kotlin en clientes HTTP. El flujo completo:

```
API REST (JSON)
     │
     ▼
RetrofitClient         ← singleton que construye el cliente HTTP
     │
     ▼
ApiService (interfaz)  ← declara los endpoints con anotaciones
     │
     ▼
Repository             ← abstrae la fuente de datos con Result<T>
     │
     ▼
ViewModel              ← consume el repositorio en viewModelScope
     │
     ▼
Composable             ← observa el StateFlow del ViewModel
```

`Result<T>` de Kotlin es el mecanismo para devolver éxito o fallo sin
lanzar excepciones hasta la UI:

```kotlin
// En el repositorio: capturamos la excepción y la envolvemos
Result.success(datos)       // éxito
Result.failure(exception)   // fallo

// En el ViewModel: desempaquetamos con fold
resultado.fold(
    onSuccess = { datos -> /* usar datos */ },
    onFailure = { error -> /* manejar error */ }
)
```

### Paso 5 — Retrofit + Repository + ViewModel con API real

**Crea:** `network/ApiService.kt`

```kotlin
// network/ApiService.kt
package com.tuapp.catalogo.network

import com.tuapp.catalogo.model.PaginatedResponse
import retrofit2.http.GET
import retrofit2.http.Query

interface ProductosApiService {
    // suspend: función suspendible — Retrofit la ejecuta en un hilo de I/O
    // @GET: método HTTP y ruta relativa a la BASE_URL
    // @Query: parámetros opcionales de query string (?page=1&search=teclado)
    @GET("products/")
    suspend fun obtenerProductos(
        @Query("page")      page:     Int     = 1,
        @Query("page_size") pageSize: Int     = 10,
        @Query("search")    search:   String? = null
    ): PaginatedResponse
}
```

**Crea:** `network/RetrofitClient.kt`

```kotlin
// network/RetrofitClient.kt
package com.tuapp.catalogo.network

import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

// object: singleton en Kotlin — una sola instancia en toda la app
object RetrofitClient {
    private const val BASE_URL =
        "https://higuera-billing-api.desarrollo-software.xyz/api/"

    // by lazy: se inicializa solo cuando se accede por primera vez
    val productosApi: ProductosApiService by lazy {
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            // GsonConverterFactory: convierte JSON ↔ data classes automáticamente
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(ProductosApiService::class.java)
    }
}
```

**Crea:** `repository/ProductosRepository.kt`

```kotlin
// repository/ProductosRepository.kt
package com.tuapp.catalogo.repository

import com.tuapp.catalogo.model.PaginatedResponse
import com.tuapp.catalogo.network.RetrofitClient

class ProductosRepository {
    // Result<T>: devuelve éxito o fallo sin propagar excepciones
    // El ViewModel decide qué hacer con cada caso
    suspend fun obtenerProductos(
        page:     Int     = 1,
        pageSize: Int     = 10,
        search:   String? = null
    ): Result<PaginatedResponse> = try {
        val respuesta = RetrofitClient.productosApi
            .obtenerProductos(page, pageSize, search)
        Result.success(respuesta)
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

**Crea:** `viewmodel/ProductosApiViewModel.kt`

```kotlin
// viewmodel/ProductosApiViewModel.kt
package com.tuapp.catalogo.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.tuapp.catalogo.model.ProductoApi
import com.tuapp.catalogo.repository.ProductosRepository
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

// Data class de estado — agrupa todos los campos en un solo objeto
// Facilita actualizaciones atómicas con .update { }
data class EstadoProductos(
    val productos:    List<ProductoApi> = emptyList(),
    val cargando:     Boolean           = false,
    val error:        String?           = null,
    val paginaActual: Int               = 1,
    val totalItems:   Int               = 0,
    val busqueda:     String            = ""
) {
    // Propiedades derivadas — calculadas a partir del estado
    val totalPaginas  get() = if (totalItems == 0) 1 else (totalItems + 9) / 10
    val hayMasPaginas get() = paginaActual < totalPaginas
}

class ProductosApiViewModel(
    private val repositorio: ProductosRepository = ProductosRepository()
) : ViewModel() {

    private val _estado = MutableStateFlow(EstadoProductos())
    val estado: StateFlow<EstadoProductos> = _estado.asStateFlow()

    init { cargar() }

    fun cargar(
        pagina:   Int    = 1,
        busqueda: String = _estado.value.busqueda
    ) {
        viewModelScope.launch {
            // .update { } modifica el StateFlow de forma atómica
            // Equivale a: _estado.value = _estado.value.copy(cargando = true, ...)
            _estado.update { it.copy(cargando = true, error = null) }

            repositorio.obtenerProductos(
                page   = pagina,
                search = busqueda.ifBlank { null }
            ).fold(
                onSuccess = { respuesta ->
                    _estado.update { estado ->
                        estado.copy(
                            cargando     = false,
                            productos    = respuesta.results,
                            totalItems   = respuesta.count,
                            paginaActual = pagina,
                            busqueda     = busqueda
                        )
                    }
                },
                onFailure = { error ->
                    _estado.update {
                        it.copy(
                            cargando = false,
                            error    = error.message ?: "Error al conectar con el servidor"
                        )
                    }
                }
            )
        }
    }

    fun buscar(query: String) { cargar(pagina = 1, busqueda = query) }
    fun paginaSiguiente() { if (_estado.value.hayMasPaginas) cargar(_estado.value.paginaActual + 1) }
    fun paginaAnterior()  { if (_estado.value.paginaActual > 1) cargar(_estado.value.paginaActual - 1) }
    fun recargar()        { cargar(pagina = 1) }
}
```

**Crea:** `ui/Paso05_Retrofit.kt`

```kotlin
// ui/Paso05_Retrofit.kt
package com.tuapp.catalogo.ui

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import coil.compose.AsyncImage
import com.tuapp.catalogo.model.ProductoApi
import com.tuapp.catalogo.viewmodel.ProductosApiViewModel

@Composable
fun Paso05_RetrofitScreen(vm: ProductosApiViewModel = viewModel()) {
    val estado   by vm.estado.collectAsStateWithLifecycle()
    // rememberSaveable para que la búsqueda sobreviva rotaciones en la UI
    var busqueda by rememberSaveable { mutableStateOf("") }

    Scaffold(
        topBar = {
            TopAppBar(
                title  = { Text("Catálogo API", fontWeight = FontWeight.Bold) },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor    = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.onPrimaryContainer
                )
            )
        }
    ) { paddingValues ->
        Column(
            modifier = Modifier
                .padding(paddingValues)
                .fillMaxSize()
        ) {
            // ── Buscador ──────────────────────────────────────────────────
            OutlinedTextField(
                value         = busqueda,
                onValueChange = { busqueda = it; vm.buscar(it) },
                placeholder   = { Text("Buscar productos...") },
                leadingIcon   = { Icon(Icons.Default.Search, null) },
                trailingIcon  = {
                    if (busqueda.isNotEmpty())
                        IconButton(onClick = { busqueda = ""; vm.buscar("") }) {
                            Icon(Icons.Default.Clear, null)
                        }
                },
                singleLine = true,
                modifier   = Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 8.dp)
            )

            // ── Error inline (no bloquea la lista) ───────────────────────
            estado.error?.let { mensajeError ->
                Row(
                    modifier          = Modifier
                        .fillMaxWidth()
                        .padding(horizontal = 16.dp),
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    Icon(Icons.Default.WifiOff, null,
                         tint     = MaterialTheme.colorScheme.error,
                         modifier = Modifier.size(20.dp))
                    Spacer(Modifier.width(8.dp))
                    Text(mensajeError,
                         color    = MaterialTheme.colorScheme.error,
                         style    = MaterialTheme.typography.bodySmall,
                         modifier = Modifier.weight(1f))
                    TextButton(onClick = { vm.recargar() }) { Text("Reintentar") }
                }
            }

            // ── LinearProgressIndicator: no bloquea el contenido ─────────
            if (estado.cargando) {
                LinearProgressIndicator(modifier = Modifier.fillMaxWidth())
            }

            // ── Lista de productos de la API ──────────────────────────────
            LazyColumn(
                modifier            = Modifier.weight(1f),
                contentPadding      = PaddingValues(16.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                items(estado.productos, key = { it.id }) { producto ->
                    TarjetaProductoApi(producto = producto)
                }
            }

            // ── Paginación ────────────────────────────────────────────────
            if (estado.totalItems > 0) {
                Row(
                    modifier              = Modifier
                        .fillMaxWidth()
                        .padding(horizontal = 16.dp, vertical = 8.dp),
                    horizontalArrangement = Arrangement.SpaceBetween,
                    verticalAlignment     = Alignment.CenterVertically
                ) {
                    OutlinedButton(
                        onClick  = { vm.paginaAnterior() },
                        enabled  = estado.paginaActual > 1 && !estado.cargando
                    ) { Text("← Anterior") }

                    Text(
                        "${estado.paginaActual} / ${estado.totalPaginas}  " +
                        "(${estado.totalItems} total)",
                        style = MaterialTheme.typography.bodySmall
                    )

                    Button(
                        onClick  = { vm.paginaSiguiente() },
                        enabled  = estado.hayMasPaginas && !estado.cargando
                    ) { Text("Siguiente →") }
                }
            }
        }
    }
}

// AsyncImage: carga imágenes desde URL de forma asíncrona
// Maneja automáticamente: descarga, caché en memoria y disco, y error
@Composable
internal fun TarjetaProductoApi(
    producto: ProductoApi,
    onClick:  () -> Unit = {}
) {
    ElevatedCard(onClick = onClick, modifier = Modifier.fillMaxWidth()) {
        Row(
            modifier          = Modifier.padding(12.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            // AsyncImage de Coil — requiere: implementation("io.coil-kt:coil-compose:2.7.0")
            AsyncImage(
                model              = producto.url_image,
                contentDescription = producto.name,
                modifier           = Modifier
                    .size(72.dp)
                    .clip(RoundedCornerShape(8.dp)),
                contentScale       = ContentScale.Crop
            )

            Spacer(Modifier.width(12.dp))

            Column(modifier = Modifier.weight(1f)) {
                Text(producto.name,
                     fontWeight = FontWeight.SemiBold,
                     maxLines   = 2,
                     overflow   = TextOverflow.Ellipsis,
                     style      = MaterialTheme.typography.titleSmall)
                Text(producto.category_name,
                     style = MaterialTheme.typography.bodySmall,
                     color = MaterialTheme.colorScheme.onSurfaceVariant)
                Spacer(Modifier.height(4.dp))
                Row(
                    verticalAlignment     = Alignment.CenterVertically,
                    horizontalArrangement = Arrangement.spacedBy(8.dp)
                ) {
                    Text("$${producto.price}",
                         style      = MaterialTheme.typography.titleSmall,
                         color      = MaterialTheme.colorScheme.primary,
                         fontWeight = FontWeight.Bold)
                    Text("Stock: ${producto.stock}",
                         style = MaterialTheme.typography.labelSmall,
                         color = MaterialTheme.colorScheme.onSurfaceVariant)
                    AssistChip(
                        onClick = {},
                        label   = {
                            Text(if (producto.is_active) "Activo" else "Inactivo",
                                 style = MaterialTheme.typography.labelSmall)
                        }
                    )
                }
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun Paso05_Preview() {
    MaterialTheme { Paso05_RetrofitScreen() }
}
```

**▶ Ejecuta:** descomenta `Paso05_RetrofitScreen()`.
- La app carga productos reales de la API (requiere internet)
- Busca "teclado" → llama a la API con el parámetro `search`
- Navega entre páginas con los botones Anterior/Siguiente
- Desactiva el wifi → aparece el error inline con botón Reintentar

---

## App completa integrada

### Paso 6 — Todo integrado: Navigation + ViewModel + API

**Crea:** `ui/Paso06_Completo.kt`

Integramos la navegación del Paso 3/4 con la API real del Paso 5.

```kotlin
// ui/Paso06_Completo.kt
package com.tuapp.catalogo.ui

import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material.icons.outlined.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.navigation.*
import androidx.navigation.compose.*
import com.tuapp.catalogo.viewmodel.ProductosApiViewModel

// Rutas finales — misma definición que en el Paso 3
sealed class RutaFinal(val path: String) {
    object Catalogo : RutaFinal("catalogo")
    object Detalle  : RutaFinal("catalogo/{productoId}") {
        fun conId(id: Int) = "catalogo/$id"
    }
    object Perfil   : RutaFinal("perfil")
}

data class ItemNavFinal(
    val ruta:          RutaFinal,
    val etiqueta:      String,
    val iconoActivo:   ImageVector,
    val iconoInactivo: ImageVector
)

@Composable
fun Paso06_CompletoScreen() {
    val navController  = rememberNavController()
    val backStackEntry by navController.currentBackStackEntryAsState()
    val rutaActual     = backStackEntry?.destination?.route
    val rutasRaiz      = listOf(RutaFinal.Catalogo.path, RutaFinal.Perfil.path)

    val itemsNav = listOf(
        ItemNavFinal(RutaFinal.Catalogo, "Catálogo",
            Icons.Filled.Inventory,     Icons.Outlined.Inventory),
        ItemNavFinal(RutaFinal.Perfil,   "Perfil",
            Icons.Filled.AccountCircle, Icons.Outlined.AccountCircle),
    )

    Scaffold(
        topBar = {
            TopAppBar(
                title = {
                    Text(
                        when {
                            rutaActual == RutaFinal.Catalogo.path -> "Catálogo de Productos"
                            rutaActual == RutaFinal.Perfil.path   -> "Mi Perfil"
                            else                                  -> "Detalle"
                        },
                        fontWeight = FontWeight.Bold
                    )
                },
                navigationIcon = {
                    if (rutaActual !in rutasRaiz) {
                        IconButton(onClick = { navController.popBackStack() }) {
                            Icon(Icons.Default.ArrowBack, "Volver")
                        }
                    }
                },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor    = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.onPrimaryContainer
                )
            )
        },
        bottomBar = {
            if (rutaActual in rutasRaiz) {
                NavigationBar {
                    itemsNav.forEach { item ->
                        val sel = rutaActual == item.ruta.path
                        NavigationBarItem(
                            selected = sel,
                            onClick  = {
                                navController.navigate(item.ruta.path) {
                                    popUpTo(navController.graph.startDestinationId) {
                                        saveState = true
                                    }
                                    launchSingleTop = true
                                    restoreState    = true
                                }
                            },
                            icon  = {
                                Icon(if (sel) item.iconoActivo else item.iconoInactivo,
                                     item.etiqueta)
                            },
                            label = { Text(item.etiqueta) }
                        )
                    }
                }
            }
        }
    ) { paddingValues ->
        NavHost(
            navController    = navController,
            startDestination = RutaFinal.Catalogo.path,
            modifier         = Modifier.padding(paddingValues)
        ) {
            // Catálogo — ViewModel de API
            composable(RutaFinal.Catalogo.path) {
                val vm: ProductosApiViewModel = viewModel()
                val estado   by vm.estado.collectAsStateWithLifecycle()
                var busqueda by rememberSaveable { mutableStateOf("") }

                CatalogoContent(
                    estado       = estado,
                    busqueda     = busqueda,
                    onBusqueda   = { busqueda = it; vm.buscar(it) },
                    onAnterior   = { vm.paginaAnterior() },
                    onSiguiente  = { vm.paginaSiguiente() },
                    onRecargar   = { vm.recargar() },
                    onVerDetalle = { id ->
                        navController.navigate(RutaFinal.Detalle.conId(id))
                    }
                )
            }

            // Detalle — recibe productoId de la ruta
            composable(
                route     = RutaFinal.Detalle.path,
                arguments = listOf(navArgument("productoId") { type = NavType.IntType })
            ) { backStackEntry ->
                val productoId = backStackEntry.arguments?.getInt("productoId")
                    ?: return@composable
                // Buscamos el producto en el estado actual del ViewModel del catálogo
                // En producción, DetalleViewModel haría su propia llamada a la API
                val vm: ProductosApiViewModel = viewModel(
                    viewModelStoreOwner = backStackEntry
                )
                val estado by vm.estado.collectAsStateWithLifecycle()
                val producto = estado.productos.find { it.id == productoId }

                if (producto != null) {
                    DetalleApiContent(
                        producto = producto,
                        onVolver = { navController.popBackStack() }
                    )
                } else {
                    Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                        Column(horizontalAlignment = Alignment.CenterHorizontally) {
                            CircularProgressIndicator()
                            Spacer(Modifier.height(8.dp))
                            Text("Cargando detalle...")
                        }
                    }
                }
            }

            // Perfil
            composable(RutaFinal.Perfil.path) {
                PantallaPerfilSimple()
            }
        }
    }
}

@Composable
private fun CatalogoContent(
    estado:       com.tuapp.catalogo.viewmodel.EstadoProductos,
    busqueda:     String,
    onBusqueda:   (String) -> Unit,
    onAnterior:   () -> Unit,
    onSiguiente:  () -> Unit,
    onRecargar:   () -> Unit,
    onVerDetalle: (Int) -> Unit
) {
    Column(modifier = Modifier.fillMaxSize()) {
        OutlinedTextField(
            value         = busqueda,
            onValueChange = onBusqueda,
            placeholder   = { Text("Buscar productos...") },
            leadingIcon   = { Icon(Icons.Default.Search, null) },
            trailingIcon  = {
                if (busqueda.isNotEmpty())
                    IconButton(onClick = { onBusqueda("") }) {
                        Icon(Icons.Default.Clear, null)
                    }
            },
            singleLine = true,
            modifier   = Modifier.fillMaxWidth().padding(horizontal = 16.dp, vertical = 8.dp)
        )

        estado.error?.let { error ->
            Row(
                Modifier.fillMaxWidth().padding(horizontal = 16.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text(error, color = MaterialTheme.colorScheme.error,
                     modifier = Modifier.weight(1f),
                     style    = MaterialTheme.typography.bodySmall)
                TextButton(onClick = onRecargar) { Text("Reintentar") }
            }
        }

        if (estado.cargando) LinearProgressIndicator(Modifier.fillMaxWidth())

        androidx.compose.foundation.lazy.LazyColumn(
            modifier            = Modifier.weight(1f),
            contentPadding      = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            androidx.compose.foundation.lazy.items(
                estado.productos, key = { it.id }
            ) { producto ->
                TarjetaProductoApi(
                    producto = producto,
                    onClick  = { onVerDetalle(producto.id) }
                )
            }
        }

        if (estado.totalItems > 0) {
            Row(
                modifier              = Modifier.fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 8.dp),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment     = Alignment.CenterVertically
            ) {
                OutlinedButton(
                    onClick  = onAnterior,
                    enabled  = estado.paginaActual > 1 && !estado.cargando
                ) { Text("← Anterior") }

                Text("${estado.paginaActual}/${estado.totalPaginas}",
                     style = MaterialTheme.typography.bodySmall)

                Button(
                    onClick  = onSiguiente,
                    enabled  = estado.hayMasPaginas && !estado.cargando
                ) { Text("Siguiente →") }
            }
        }
    }
}

@Composable
private fun DetalleApiContent(
    producto: com.tuapp.catalogo.model.ProductoApi,
    onVolver: () -> Unit
) {
    Column(
        modifier            = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        AsyncImageDetalle(url = producto.url_image, nombre = producto.name)
        Text(producto.name, style = MaterialTheme.typography.headlineSmall,
             fontWeight = FontWeight.Bold)
        HorizontalDivider()
        FilaDetalleApi("Categoría",  producto.category_name)
        FilaDetalleApi("Precio",     "$${producto.price}")
        FilaDetalleApi("Stock",      "${producto.stock} unidades")
        FilaDetalleApi("Estado",     if (producto.is_active) "Activo ✓" else "Inactivo")
        Spacer(Modifier.weight(1f))
        Button(onClick = onVolver, modifier = Modifier.fillMaxWidth()) {
            Icon(Icons.Default.ArrowBack, null)
            Spacer(Modifier.width(8.dp))
            Text("Volver al catálogo")
        }
    }
}

@Composable
private fun AsyncImageDetalle(url: String, nombre: String) {
    coil.compose.AsyncImage(
        model              = url,
        contentDescription = nombre,
        modifier           = Modifier
            .fillMaxWidth()
            .height(200.dp)
            .androidx.compose.ui.draw.clip(
                androidx.compose.foundation.shape.RoundedCornerShape(12.dp)
            ),
        contentScale       = androidx.compose.ui.layout.ContentScale.Crop
    )
}

@Composable
private fun FilaDetalleApi(etiqueta: String, valor: String) {
    Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
        Text(etiqueta, color = MaterialTheme.colorScheme.onSurfaceVariant,
             style = MaterialTheme.typography.bodyMedium)
        Text(valor, fontWeight = FontWeight.SemiBold,
             style = MaterialTheme.typography.bodyMedium)
    }
}

@Preview(showBackground = true)
@Composable
fun Paso06_Preview() {
    MaterialTheme { Paso06_CompletoScreen() }
}
```

**▶ Ejecuta:** `Paso06_CompletoScreen()` ya está activo en `MainActivity`.
- Carga productos reales de la API con imágenes
- Presiona una tarjeta → navega al detalle con imagen grande
- Busca un producto → la búsqueda llama a la API con `?search=`
- Navega a Perfil y vuelve → el catálogo conserva su scroll y página
- Rota el dispositivo en cualquier pantalla → el estado persiste

---

## Referencia rápida — qué descomentar en `MainActivity`

| Paso | Archivo | Descomentar | Qué introduce |
|---|---|---|---|
| 1 | `Paso01_ViewModel.kt` | `Paso01_ViewModelScreen()` | `ViewModel`, `StateFlow`, `viewModel()`, `collectAsStateWithLifecycle` |
| 2 | `Paso02_UiState.kt` | `Paso02_UiStateScreen()` | `sealed class UiState`, estados Loading/Error/Success |
| 3 | `Paso03_Navigation.kt` | `Paso03_NavigationScreen()` | `NavHost`, `NavController`, rutas, argumentos |
| 4 | `Paso04_Detalle.kt` | actualiza Paso03 | `SavedStateHandle`, `DetalleViewModel` |
| 5 | `Paso05_Retrofit.kt` | `Paso05_RetrofitScreen()` | Retrofit, `ApiService`, `Repository`, `Result<T>`, `AsyncImage` |
| 6 | `Paso06_Completo.kt` | `Paso06_CompletoScreen()` | Integración total: Navigation + ViewModel + API |

---

## Ejercicios propuestos

**Ejercicio 1 — Detalle desde API** (`viewmodel/DetalleApiViewModel.kt`)
Crea un `DetalleApiViewModel(savedStateHandle)` que llame a un nuevo
endpoint `@GET("products/{id}/")` en `ApiService`. Úsalo en el composable
de detalle del Paso 6 para cargar el producto directamente por ID en lugar
de buscarlo en el estado del catálogo.

**Ejercicio 2 — Pull to refresh**
Investiga `PullToRefreshBox` de Material 3 y envuelve la `LazyColumn`
de `Paso05_RetrofitScreen`. El gesto de deslizar hacia abajo debe llamar
a `vm.recargar()`.

**Ejercicio 3 — Búsqueda con debounce**
En `ProductosApiViewModel`, reemplaza la llamada directa a la API por
un `Flow` con `debounce(500)`:
```kotlin
private val _queryFlow = MutableStateFlow("")
init {
    viewModelScope.launch {
        _queryFlow.debounce(500).collect { query -> cargar(busqueda = query) }
    }
}
fun actualizarQuery(q: String) { _queryFlow.value = q }
```
Conecta `actualizarQuery` en lugar de `buscar` desde el `OutlinedTextField`.

**Ejercicio 4 — Caché local**
Agrega un `Flow<List<ProductoApi>>` en `ProductosRepository` que emita
inmediatamente los datos cacheados antes de hacer la llamada a la red.
El `ViewModel` debe mostrar los datos del caché mientras carga los nuevos,
y reemplazarlos al completar la respuesta.

---

## Resumen de la página 13

- `ViewModel` sobrevive a rotaciones y cambios de configuración. El estado con `remember` no.
- El patrón `private MutableStateFlow` + `public StateFlow` encapsula la mutabilidad dentro del `ViewModel`.
- `collectAsStateWithLifecycle()` es la forma recomendada — pausa la recolección cuando la UI no está activa y ahorra batería.
- `sealed class UiState` con `Loading`, `Success` y `Error` cubre los tres estados posibles de cualquier operación de red. El compilador garantiza exhaustividad en el `when`.
- `viewModelScope.launch` lanza coroutines que se cancelan automáticamente cuando el `ViewModel` se destruye.
- `_estado.update { }` modifica el `StateFlow` de forma atómica — útil cuando hay múltiples campos que actualizar.
- `SavedStateHandle` en el `ViewModel` de detalle recibe los argumentos de la ruta automáticamente — sin pasarlos manualmente.
- Navigation Compose usa rutas como strings. Los argumentos se definen con `{nombre}` y se declaran en `navArgument`.
- `launchSingleTop = true` + `restoreState = true` evitan crear múltiples instancias en la `NavigationBar`.
- Retrofit convierte JSON a data classes con `GsonConverterFactory`. La interfaz `ApiService` usa `suspend fun` para integración nativa con coroutines.
- El `Repository` usa `Result<T>` para devolver éxito o fallo sin propagar excepciones hasta la UI.

---

> **Siguiente página →** Página 14: Jetpack Compose — animaciones,
> temas personalizados y patrones de arquitectura Clean.