
# Curso Kotlin — Página 15
## Proyecto Integrador
### App completa: gestión de productos con Clean Architecture, Compose, Navigation, API REST y tests

---

## Descripción del proyecto

**ProductsApp** es una aplicación Android completa que integra todos
los conceptos del curso:

| Módulo del curso | Aplicado en ProductsApp |
|---|---|
| Kotlin fundamentos (pág. 1–4) | Null safety, funciones de extensión, data classes |
| OOP + sealed class (pág. 5–6) | `UiState`, `Producto`, `ProductoDto`, jerarquía de errores |
| Colecciones + lambdas (pág. 7–8) | Filtrado, mapeo, paginación |
| Coroutines + Flow (pág. 9) | `StateFlow`, `viewModelScope`, `suspend` |
| Testing (pág. 10) | Tests del ViewModel y del repositorio con MockK |
| Compose UI (pág. 11–12) | `LazyColumn`, `Scaffold`, `Card`, `TextField`, `Dialog` |
| ViewModel + Navigation (pág. 13) | Rutas tipadas, `NavHost`, `collectAsStateWithLifecycle` |
| Animaciones + Clean (pág. 14) | `AnimatedVisibility`, Use Cases, capas separadas |

---

## Estructura del proyecto

```
app/src/main/kotlin/com/ejemplo/productsapp/
├── ui/
│   ├── theme/
│   │   ├── Color.kt
│   │   ├── Theme.kt
│   │   └── Type.kt
│   ├── navigation/
│   │   └── NavGraph.kt
│   ├── productos/
│   │   ├── ListaScreen.kt
│   │   ├── ListaViewModel.kt
│   │   ├── DetalleScreen.kt
│   │   └── DetalleViewModel.kt
│   └── MainActivity.kt
├── domain/
│   ├── model/
│   │   └── Producto.kt
│   └── usecase/
│       ├── ObtenerProductosUseCase.kt
│       ├── ObtenerProductoUseCase.kt
│       └── BuscarProductosUseCase.kt
└── data/
    ├── remote/
    │   ├── ProductosApiService.kt
    │   ├── ProductoDto.kt
    │   └── RetrofitClient.kt
    └── repository/
        └── ProductosRepositoryImpl.kt

app/src/test/kotlin/com/ejemplo/productsapp/
├── domain/
│   └── usecase/
│       └── BuscarProductosUseCaseTest.kt
└── ui/
    └── productos/
        └── ListaViewModelTest.kt
```

---

## `build.gradle.kts` — dependencias completas

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
}

android {
    compileSdk = 35
    defaultConfig { minSdk = 24; targetSdk = 35 }
    buildFeatures { compose = true }
}

dependencies {
    // Compose BOM
    val composeBom = platform("androidx.compose:compose-bom:2026.03.00")
    implementation(composeBom)
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.material:material-icons-extended")
    implementation("androidx.activity:activity-compose:1.10.0")
    debugImplementation("androidx.compose.ui:ui-tooling")

    // ViewModel + Lifecycle
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.7")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.7")

    // Navigation
    implementation("androidx.navigation:navigation-compose:2.8.5")

    // Retrofit + Gson
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-gson:2.11.0")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")

    // Coil (imágenes)
    implementation("io.coil-kt:coil-compose:2.7.0")

    // Tests
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.0")
    testImplementation("io.kotest:kotest-assertions-core:5.9.1")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
    testImplementation("io.mockk:mockk:1.13.12")
}

tasks.withType<Test> { useJUnitPlatform() }
```

---

## Capa de Dominio

### `domain/model/Producto.kt`

```kotlin
package com.ejemplo.productsapp.domain.model

data class Producto(
    val id:        Int,
    val nombre:    String,
    val precio:    Double,
    val categoria: String,
    val activo:    Boolean,
    val stock:     Int,
    val urlImagen: String
) {
    val disponible: Boolean get() = activo && stock > 0
    val precioFormateado: String get() = "$${"%.2f".format(precio)}"
}
```

### `domain/usecase/ObtenerProductosUseCase.kt`

```kotlin
package com.ejemplo.productsapp.domain.usecase

import com.ejemplo.productsapp.domain.model.Producto
import com.ejemplo.productsapp.domain.repository.ProductosRepository

class ObtenerProductosUseCase(
    private val repositorio: ProductosRepository
) {
    suspend operator fun invoke(
        pagina:   Int = 1,
        pageSize: Int = 10
    ): Result<Pair<List<Producto>, Int>> =
        repositorio.obtenerProductos(pagina, pageSize)
}
```

### `domain/usecase/BuscarProductosUseCase.kt`

```kotlin
package com.ejemplo.productsapp.domain.usecase

import com.ejemplo.productsapp.domain.model.Producto
import com.ejemplo.productsapp.domain.repository.ProductosRepository

class BuscarProductosUseCase(
    private val repositorio: ProductosRepository
) {
    suspend operator fun invoke(query: String): Result<List<Producto>> {
        if (query.length < 2) return Result.success(emptyList())
        return repositorio.buscar(query)
    }
}
```

### `domain/usecase/ObtenerProductoUseCase.kt`

```kotlin
package com.ejemplo.productsapp.domain.usecase

import com.ejemplo.productsapp.domain.model.Producto
import com.ejemplo.productsapp.domain.repository.ProductosRepository

class ObtenerProductoUseCase(
    private val repositorio: ProductosRepository
) {
    suspend operator fun invoke(id: Int): Result<Producto> =
        repositorio.obtenerPorId(id)
}
```

### `domain/repository/ProductosRepository.kt`

```kotlin
package com.ejemplo.productsapp.domain.repository

import com.ejemplo.productsapp.domain.model.Producto

interface ProductosRepository {
    suspend fun obtenerProductos(pagina: Int, pageSize: Int): Result<Pair<List<Producto>, Int>>
    suspend fun buscar(query: String): Result<List<Producto>>
    suspend fun obtenerPorId(id: Int): Result<Producto>
}
```

---

## Capa de Datos

### `data/remote/ProductoDto.kt`

```kotlin
package com.ejemplo.productsapp.data.remote

import com.ejemplo.productsapp.domain.model.Producto

data class ProductoDto(
    val id:            Int,
    val name:          String,
    val price:         String,
    val category_name: String,
    val is_active:     Boolean,
    val stock:         Int,
    val url_image:     String
) {
    fun toDomain() = Producto(
        id        = id,
        nombre    = name,
        precio    = price.toDoubleOrNull() ?: 0.0,
        categoria = category_name,
        activo    = is_active,
        stock     = stock,
        urlImagen = url_image
    )
}

data class PaginatedProductosDto(
    val count:    Int,
    val results:  List<ProductoDto>
)
```

### `data/remote/ProductosApiService.kt`

```kotlin
package com.ejemplo.productsapp.data.remote

import retrofit2.http.GET
import retrofit2.http.Path
import retrofit2.http.Query

interface ProductosApiService {
    @GET("products/")
    suspend fun obtenerProductos(
        @Query("page")      page:     Int    = 1,
        @Query("page_size") pageSize: Int    = 10,
        @Query("search")    search:   String? = null
    ): PaginatedProductosDto

    @GET("products/{id}/")
    suspend fun obtenerProducto(@Path("id") id: Int): ProductoDto
}
```

### `data/remote/RetrofitClient.kt`

```kotlin
package com.ejemplo.productsapp.data.remote

import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

object RetrofitClient {
    private const val BASE_URL =
        "https://higuera-billing-api.desarrollo-software.xyz/api/"

    val api: ProductosApiService by lazy {
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(ProductosApiService::class.java)
    }
}
```

### `data/repository/ProductosRepositoryImpl.kt`

```kotlin
package com.ejemplo.productsapp.data.repository

import com.ejemplo.productsapp.data.remote.ProductosApiService
import com.ejemplo.productsapp.domain.model.Producto
import com.ejemplo.productsapp.domain.repository.ProductosRepository

class ProductosRepositoryImpl(
    private val api: ProductosApiService
) : ProductosRepository {

    override suspend fun obtenerProductos(
        pagina: Int,
        pageSize: Int
    ): Result<Pair<List<Producto>, Int>> = try {
        val response  = api.obtenerProductos(pagina, pageSize)
        val productos = response.results.map { it.toDomain() }
        Result.success(Pair(productos, response.count))
    } catch (e: Exception) {
        Result.failure(e)
    }

    override suspend fun buscar(query: String): Result<List<Producto>> = try {
        val response = api.obtenerProductos(search = query)
        Result.success(response.results.map { it.toDomain() })
    } catch (e: Exception) {
        Result.failure(e)
    }

    override suspend fun obtenerPorId(id: Int): Result<Producto> = try {
        Result.success(api.obtenerProducto(id).toDomain())
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

---

## Capa de UI

### `ui/navigation/NavGraph.kt`

```kotlin
package com.ejemplo.productsapp.ui.navigation

import androidx.compose.runtime.Composable
import androidx.navigation.*
import androidx.navigation.compose.*
import com.ejemplo.productsapp.ui.productos.*

sealed class Ruta(val path: String) {
    object Lista   : Ruta("lista")
    object Detalle : Ruta("detalle/{id}") {
        fun conId(id: Int) = "detalle/$id"
    }
}

@Composable
fun NavGraph(navController: NavHostController) {
    NavHost(navController, startDestination = Ruta.Lista.path) {

        composable(Ruta.Lista.path) {
            ListaScreen(
                onVerDetalle = { id ->
                    navController.navigate(Ruta.Detalle.conId(id))
                }
            )
        }

        composable(
            route     = Ruta.Detalle.path,
            arguments = listOf(navArgument("id") { type = NavType.IntType })
        ) { entry ->
            val id = entry.arguments?.getInt("id") ?: return@composable
            DetalleScreen(
                productoId = id,
                onVolver   = { navController.popBackStack() }
            )
        }
    }
}
```

### `ui/productos/ListaViewModel.kt`

```kotlin
package com.ejemplo.productsapp.ui.productos

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.ejemplo.productsapp.domain.model.Producto
import com.ejemplo.productsapp.domain.usecase.*
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

data class ListaUiState(
    val productos:   List<Producto> = emptyList(),
    val cargando:    Boolean        = false,
    val error:       String?        = null,
    val busqueda:    String         = "",
    val pagina:      Int            = 1,
    val totalItems:  Int            = 0
) {
    val totalPaginas  get() = maxOf(1, (totalItems + 9) / 10)
    val hayAnterior   get() = pagina > 1
    val haySiguiente  get() = pagina < totalPaginas
}

class ListaViewModel(
    private val obtenerProductos: ObtenerProductosUseCase,
    private val buscarProductos:  BuscarProductosUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(ListaUiState())
    val state: StateFlow<ListaUiState> = _state.asStateFlow()

    init { cargar() }

    fun cargar(pagina: Int = 1) {
        viewModelScope.launch {
            _state.update { it.copy(cargando = true, error = null) }
            obtenerProductos(pagina).fold(
                onSuccess = { (lista, total) ->
                    _state.update { it.copy(
                        cargando   = false,
                        productos  = lista,
                        totalItems = total,
                        pagina     = pagina,
                        busqueda   = ""
                    )}
                },
                onFailure = { e ->
                    _state.update { it.copy(
                        cargando = false,
                        error    = e.message ?: "Error al cargar"
                    )}
                }
            )
        }
    }

    fun buscar(query: String) {
        _state.update { it.copy(busqueda = query) }
        if (query.isBlank()) { cargar(); return }
        viewModelScope.launch {
            _state.update { it.copy(cargando = true, error = null) }
            buscarProductos(query).fold(
                onSuccess = { lista ->
                    _state.update { it.copy(cargando = false, productos = lista) }
                },
                onFailure = { e ->
                    _state.update { it.copy(cargando = false, error = e.message) }
                }
            )
        }
    }

    fun paginaSiguiente() { if (_state.value.haySiguiente) cargar(_state.value.pagina + 1) }
    fun paginaAnterior()  { if (_state.value.hayAnterior)  cargar(_state.value.pagina - 1) }
}
```

### `ui/productos/ListaScreen.kt`

```kotlin
package com.ejemplo.productsapp.ui.productos

import androidx.compose.animation.*
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.*
import androidx.compose.ui.draw.clip
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import coil.compose.AsyncImage
import com.ejemplo.productsapp.domain.model.Producto

@Composable
fun ListaScreen(
    vm:          ListaViewModel = androidx.lifecycle.viewmodel.compose.viewModel(),
    onVerDetalle: (Int) -> Unit
) {
    val state    by vm.state.collectAsStateWithLifecycle()
    var busqueda by rememberSaveable { mutableStateOf("") }

    Scaffold(
        topBar = {
            TopAppBar(
                title  = { Text("Productos", fontWeight = FontWeight.Bold) },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer
                )
            )
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .padding(padding)
                .fillMaxSize()
        ) {
            // Barra de búsqueda
            OutlinedTextField(
                value         = busqueda,
                onValueChange = { busqueda = it; vm.buscar(it) },
                placeholder   = { Text("Buscar productos...") },
                leadingIcon   = { Icon(Icons.Default.Search, null) },
                trailingIcon  = {
                    AnimatedVisibility(visible = busqueda.isNotEmpty()) {
                        IconButton(onClick = { busqueda = ""; vm.buscar("") }) {
                            Icon(Icons.Default.Clear, "Limpiar")
                        }
                    }
                },
                singleLine = true,
                modifier   = Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 8.dp)
            )

            // Indicador de carga
            AnimatedVisibility(visible = state.cargando) {
                LinearProgressIndicator(modifier = Modifier.fillMaxWidth())
            }

            // Error
            state.error?.let { msg ->
                Row(
                    Modifier
                        .fillMaxWidth()
                        .padding(horizontal = 16.dp, vertical = 4.dp),
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    Text(msg, color = MaterialTheme.colorScheme.error, modifier = Modifier.weight(1f))
                    TextButton(onClick = { vm.cargar() }) { Text("Reintentar") }
                }
            }

            // Lista de productos
            LazyColumn(
                modifier            = Modifier.weight(1f),
                contentPadding      = PaddingValues(16.dp),
                verticalArrangement = Arrangement.spacedBy(10.dp)
            ) {
                items(state.productos, key = { it.id }) { producto ->
                    TarjetaProducto(
                        producto = producto,
                        onClick  = { onVerDetalle(producto.id) }
                    )
                }

                if (state.productos.isEmpty() && !state.cargando) {
                    item {
                        Box(
                            Modifier.fillMaxWidth().padding(top = 48.dp),
                            contentAlignment = Alignment.Center
                        ) {
                            Text(
                                if (busqueda.isNotEmpty()) "Sin resultados para \"$busqueda\""
                                else "No hay productos",
                                color = MaterialTheme.colorScheme.onSurfaceVariant
                            )
                        }
                    }
                }
            }

            // Paginación — solo cuando no hay búsqueda activa
            AnimatedVisibility(visible = busqueda.isBlank() && state.totalItems > 0) {
                Row(
                    Modifier
                        .fillMaxWidth()
                        .padding(horizontal = 16.dp, vertical = 8.dp),
                    horizontalArrangement = Arrangement.SpaceBetween,
                    verticalAlignment     = Alignment.CenterVertically
                ) {
                    OutlinedButton(
                        onClick  = { vm.paginaAnterior() },
                        enabled  = state.hayAnterior && !state.cargando
                    ) { Text("← Anterior") }

                    Text(
                        "${state.pagina} / ${state.totalPaginas}",
                        style = MaterialTheme.typography.labelLarge
                    )

                    Button(
                        onClick  = { vm.paginaSiguiente() },
                        enabled  = state.haySiguiente && !state.cargando
                    ) { Text("Siguiente →") }
                }
            }
        }
    }
}

@Composable
fun TarjetaProducto(producto: Producto, onClick: () -> Unit) {
    ElevatedCard(
        onClick  = onClick,
        modifier = Modifier.fillMaxWidth()
    ) {
        Row(
            modifier          = Modifier.padding(12.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            AsyncImage(
                model              = producto.urlImagen,
                contentDescription = producto.nombre,
                modifier           = Modifier
                    .size(72.dp)
                    .clip(RoundedCornerShape(8.dp)),
                contentScale       = ContentScale.Crop
            )
            Spacer(Modifier.width(12.dp))
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    producto.nombre,
                    fontWeight = FontWeight.SemiBold,
                    maxLines   = 2,
                    overflow   = TextOverflow.Ellipsis
                )
                Text(
                    producto.categoria,
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
                Spacer(Modifier.height(6.dp))
                Row(
                    verticalAlignment     = Alignment.CenterVertically,
                    horizontalArrangement = Arrangement.spacedBy(8.dp)
                ) {
                    Text(
                        producto.precioFormateado,
                        color      = MaterialTheme.colorScheme.primary,
                        fontWeight = FontWeight.Bold,
                        style      = MaterialTheme.typography.titleSmall
                    )
                    SuggestionChip(
                        onClick = {},
                        label   = {
                            Text(
                                if (producto.disponible) "En stock" else "Sin stock",
                                style = MaterialTheme.typography.labelSmall
                            )
                        }
                    )
                }
            }
            Icon(
                Icons.Default.ChevronRight,
                contentDescription = "Ver detalle",
                tint = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

### `ui/productos/DetalleViewModel.kt`

```kotlin
package com.ejemplo.productsapp.ui.productos

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.ejemplo.productsapp.domain.model.Producto
import com.ejemplo.productsapp.domain.usecase.ObtenerProductoUseCase
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

sealed class DetalleUiState {
    object Loading                          : DetalleUiState()
    data class Success(val producto: Producto) : DetalleUiState()
    data class Error(val mensaje: String)   : DetalleUiState()
}

class DetalleViewModel(
    private val obtenerProducto: ObtenerProductoUseCase
) : ViewModel() {

    private val _state = MutableStateFlow<DetalleUiState>(DetalleUiState.Loading)
    val state: StateFlow<DetalleUiState> = _state.asStateFlow()

    fun cargar(id: Int) {
        viewModelScope.launch {
            _state.value = DetalleUiState.Loading
            obtenerProducto(id).fold(
                onSuccess = { _state.value = DetalleUiState.Success(it) },
                onFailure = { _state.value = DetalleUiState.Error(it.message ?: "Error") }
            )
        }
    }
}
```

### `ui/productos/DetalleScreen.kt`

```kotlin
package com.ejemplo.productsapp.ui.productos

import androidx.compose.animation.AnimatedContent
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import coil.compose.AsyncImage

@Composable
fun DetalleScreen(
    productoId: Int,
    vm:         DetalleViewModel = androidx.lifecycle.viewmodel.compose.viewModel(),
    onVolver:   () -> Unit
) {
    val state by vm.state.collectAsStateWithLifecycle()

    LaunchedEffect(productoId) { vm.cargar(productoId) }

    Scaffold(
        topBar = {
            TopAppBar(
                title           = { Text("Detalle del producto") },
                navigationIcon  = {
                    IconButton(onClick = onVolver) {
                        Icon(Icons.Default.ArrowBack, "Volver")
                    }
                }
            )
        }
    ) { padding ->
        AnimatedContent(
            targetState = state,
            label       = "detalleState",
            modifier    = Modifier.padding(padding).fillMaxSize()
        ) { estadoActual ->
            when (estadoActual) {
                is DetalleUiState.Loading -> {
                    Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                        CircularProgressIndicator()
                    }
                }
                is DetalleUiState.Error -> {
                    Column(
                        Modifier.fillMaxSize().padding(24.dp),
                        verticalArrangement    = Arrangement.Center,
                        horizontalAlignment    = Alignment.CenterHorizontally
                    ) {
                        Icon(
                            Icons.Default.ErrorOutline,
                            contentDescription = null,
                            modifier           = Modifier.size(64.dp),
                            tint               = MaterialTheme.colorScheme.error
                        )
                        Spacer(Modifier.height(16.dp))
                        Text(estadoActual.mensaje, color = MaterialTheme.colorScheme.error)
                        Spacer(Modifier.height(12.dp))
                        Button(onClick = { vm.cargar(productoId) }) { Text("Reintentar") }
                    }
                }
                is DetalleUiState.Success -> {
                    val p = estadoActual.producto
                    Column(
                        modifier = Modifier
                            .fillMaxSize()
                            .verticalScroll(rememberScrollState())
                            .padding(16.dp),
                        verticalArrangement = Arrangement.spacedBy(16.dp)
                    ) {
                        // Imagen
                        AsyncImage(
                            model              = p.urlImagen,
                            contentDescription = p.nombre,
                            modifier           = Modifier
                                .fillMaxWidth()
                                .height(240.dp)
                                .clip(RoundedCornerShape(16.dp)),
                            contentScale       = ContentScale.Crop
                        )

                        // Info principal
                        Card(Modifier.fillMaxWidth()) {
                            Column(Modifier.padding(16.dp)) {
                                Text(p.nombre, style = MaterialTheme.typography.headlineSmall, fontWeight = FontWeight.Bold)
                                Spacer(Modifier.height(4.dp))
                                Text(p.categoria, color = MaterialTheme.colorScheme.onSurfaceVariant)
                                Spacer(Modifier.height(12.dp))
                                Row(
                                    Modifier.fillMaxWidth(),
                                    horizontalArrangement = Arrangement.SpaceBetween,
                                    verticalAlignment     = Alignment.CenterVertically
                                ) {
                                    Text(
                                        p.precioFormateado,
                                        style      = MaterialTheme.typography.headlineMedium,
                                        color      = MaterialTheme.colorScheme.primary,
                                        fontWeight = FontWeight.Bold
                                    )
                                    AssistChip(
                                        onClick = {},
                                        label   = {
                                            Text(if (p.disponible) "✓ Disponible" else "✗ Sin stock")
                                        },
                                        colors = AssistChipDefaults.assistChipColors(
                                            containerColor = if (p.disponible)
                                                MaterialTheme.colorScheme.primaryContainer
                                            else
                                                MaterialTheme.colorScheme.errorContainer
                                        )
                                    )
                                }
                            }
                        }

                        // Estadísticas
                        Card(Modifier.fillMaxWidth()) {
                            Column(Modifier.padding(16.dp)) {
                                Text("Información de stock", fontWeight = FontWeight.SemiBold, style = MaterialTheme.typography.titleSmall)
                                Spacer(Modifier.height(12.dp))
                                Row(
                                    Modifier.fillMaxWidth(),
                                    horizontalArrangement = Arrangement.SpaceEvenly
                                ) {
                                    EstadisticaItem("Stock", "${p.stock} uds.")
                                    VerticalDivider(Modifier.height(40.dp))
                                    EstadisticaItem("Estado", if (p.activo) "Activo" else "Inactivo")
                                    VerticalDivider(Modifier.height(40.dp))
                                    EstadisticaItem("ID", "#${p.id}")
                                }
                            }
                        }

                        // Botón de acción
                        Button(
                            onClick  = { /* Añadir al carrito */ },
                            enabled  = p.disponible,
                            modifier = Modifier.fillMaxWidth().height(52.dp)
                        ) {
                            Icon(Icons.Default.AddShoppingCart, null)
                            Spacer(Modifier.width(8.dp))
                            Text("Añadir al carrito", style = MaterialTheme.typography.labelLarge)
                        }
                    }
                }
            }
        }
    }
}

@Composable
private fun EstadisticaItem(label: String, valor: String) {
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Text(valor, fontWeight = FontWeight.Bold, style = MaterialTheme.typography.titleMedium)
        Text(label, style = MaterialTheme.typography.labelSmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant)
    }
}
```

---

## Tests

### `BuscarProductosUseCaseTest.kt`

```kotlin
package com.ejemplo.productsapp.domain.usecase

import com.ejemplo.productsapp.domain.model.Producto
import com.ejemplo.productsapp.domain.repository.ProductosRepository
import io.kotest.matchers.shouldBe
import io.kotest.matchers.collections.shouldBeEmpty
import io.mockk.*
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test

class BuscarProductosUseCaseTest {

    private val repositorio = mockk<ProductosRepository>()
    private val useCase     = BuscarProductosUseCase(repositorio)

    @Test
    fun `query menor a 2 caracteres devuelve lista vacía sin llamar al repositorio`() = runTest {
        val resultado = useCase("")
        resultado.getOrThrow().shouldBeEmpty()
        coVerify(exactly = 0) { repositorio.buscar(any()) }
    }

    @Test
    fun `query de 1 caracter devuelve lista vacía`() = runTest {
        val resultado = useCase("a")
        resultado.getOrThrow().shouldBeEmpty()
        coVerify(exactly = 0) { repositorio.buscar(any()) }
    }

    @Test
    fun `query válida delega al repositorio y devuelve resultados`() = runTest {
        val productosEsperados = listOf(
            Producto(1, "Teclado", 89.99, "Periféricos", true, 10, "")
        )
        coEvery { repositorio.buscar("Teclado") } returns Result.success(productosEsperados)

        val resultado = useCase("Teclado")

        resultado.isSuccess shouldBe true
        resultado.getOrThrow() shouldBe productosEsperados
        coVerify(exactly = 1) { repositorio.buscar("Teclado") }
    }

    @Test
    fun `error del repositorio se propaga como Result failure`() = runTest {
        coEvery { repositorio.buscar(any()) } returns
            Result.failure(RuntimeException("Sin conexión"))

        val resultado = useCase("Monitor")

        resultado.isFailure shouldBe true
        resultado.exceptionOrNull()?.message shouldBe "Sin conexión"
    }
}
```

### `ListaViewModelTest.kt`

```kotlin
package com.ejemplo.productsapp.ui.productos

import com.ejemplo.productsapp.domain.model.Producto
import com.ejemplo.productsapp.domain.usecase.BuscarProductosUseCase
import com.ejemplo.productsapp.domain.usecase.ObtenerProductosUseCase
import io.kotest.matchers.shouldBe
import io.kotest.matchers.nulls.shouldBeNull
import io.kotest.matchers.nulls.shouldNotBeNull
import io.mockk.*
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.jupiter.api.*

@OptIn(ExperimentalCoroutinesApi::class)
class ListaViewModelTest {

    private val obtenerProductos = mockk<ObtenerProductosUseCase>()
    private val buscarProductos  = mockk<BuscarProductosUseCase>()

    private val productosEjemplo = listOf(
        Producto(1, "Teclado", 89.99, "Periféricos", true, 10, ""),
        Producto(2, "Monitor", 349.99, "Pantallas",  true,  5, "")
    )

    @BeforeEach
    fun setUp() {
        Dispatchers.setMain(UnconfinedTestDispatcher())
    }

    @AfterEach
    fun tearDown() {
        Dispatchers.resetMain()
    }

    @Test
    fun `init carga los productos automáticamente`() = runTest {
        coEvery { obtenerProductos(1) } returns
            Result.success(Pair(productosEjemplo, 2))

        val vm = ListaViewModel(obtenerProductos, buscarProductos)

        vm.state.value.productos shouldBe productosEjemplo
        vm.state.value.cargando  shouldBe false
        vm.state.value.error.shouldBeNull()
    }

    @Test
    fun `error de red actualiza el estado con mensaje de error`() = runTest {
        coEvery { obtenerProductos(1) } returns
            Result.failure(RuntimeException("Sin conexión"))

        val vm = ListaViewModel(obtenerProductos, buscarProductos)

        vm.state.value.error.shouldNotBeNull()
        vm.state.value.error shouldBe "Sin conexión"
        vm.state.value.productos shouldBe emptyList()
    }

    @Test
    fun `buscar actualiza los productos con los resultados de búsqueda`() = runTest {
        coEvery { obtenerProductos(1) } returns
            Result.success(Pair(productosEjemplo, 2))
        val soloTeclado = productosEjemplo.filter { it.nombre == "Teclado" }
        coEvery { buscarProductos("Teclado") } returns Result.success(soloTeclado)

        val vm = ListaViewModel(obtenerProductos, buscarProductos)
        vm.buscar("Teclado")

        vm.state.value.productos shouldBe soloTeclado
        vm.state.value.busqueda  shouldBe "Teclado"
    }

    @Test
    fun `buscar con string vacío recarga la lista paginada`() = runTest {
        coEvery { obtenerProductos(1) } returns
            Result.success(Pair(productosEjemplo, 2))

        val vm = ListaViewModel(obtenerProductos, buscarProductos)
        vm.buscar("")

        coVerify(atLeast = 1) { obtenerProductos(1) }
        coVerify(exactly  = 0) { buscarProductos(any()) }
    }

    @Test
    fun `paginaSiguiente no avanza si ya estamos en la última página`() = runTest {
        coEvery { obtenerProductos(1) } returns
            Result.success(Pair(productosEjemplo, 2))  // 2 items, pageSize=10 → 1 página

        val vm = ListaViewModel(obtenerProductos, buscarProductos)
        vm.paginaSiguiente()

        coVerify(exactly = 1) { obtenerProductos(1) }  // solo la llamada inicial
    }
}
```

---

## `MainActivity.kt`

```kotlin
package com.ejemplo.productsapp.ui

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.navigation.compose.rememberNavController
import com.ejemplo.productsapp.ui.navigation.NavGraph
import com.ejemplo.productsapp.ui.theme.MiAppTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MiAppTheme {
                val navController = rememberNavController()
                NavGraph(navController = navController)
            }
        }
    }
}
```

---

## Flujo completo de la aplicación

```
MainActivity
└── MiAppTheme
    └── NavGraph
        ├── ListaScreen (ruta "lista")
        │   ├── ListaViewModel
        │   │   ├── ObtenerProductosUseCase → ProductosRepository → API
        │   │   └── BuscarProductosUseCase  → ProductosRepository → API
        │   └── TarjetaProducto × N → navController.navigate("detalle/{id}")
        │
        └── DetalleScreen (ruta "detalle/{id}")
            └── DetalleViewModel
                └── ObtenerProductoUseCase → ProductosRepository → API
```

---

## Resumen del curso

Con esta página concluyen las **15 páginas** del curso de Kotlin.

| Módulo | Páginas | Contenido |
|---|---|---|
| **1 — Fundamentos** | 1–4 | Entorno, sintaxis, tipos, control de flujo, funciones, null safety |
| **2 — OOP** | 5–6 | Los 4 principios, clases, data class, herencia, interfaces, sealed class |
| **3 — Colecciones y funciones** | 7–8 | List/Set/Map, operaciones funcionales, Sequence, lambdas, genéricos |
| **4 — Concurrencia** | 9 | Coroutines, suspend, launch, async, Flow, StateFlow |
| **5 — Testing** | 10 | JUnit 5, Kotest, runTest, MockK |
| **6 — Jetpack Compose** | 11–15 | Fundamentos, componentes Material 3, ViewModel, Navigation, API REST, animaciones, Clean Architecture, proyecto integrador |