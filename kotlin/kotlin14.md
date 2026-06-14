# Curso Kotlin — Página 14
## Módulo 6 · Jetpack Compose
### Animaciones, tema personalizado y arquitectura Clean

---

## Animaciones en Compose

Compose tiene un sistema de animaciones integrado que funciona
de forma declarativa — describes el estado final y Compose
interpola automáticamente entre estados:

```
AnimatedVisibility     ← mostrar/ocultar con animación
animateFloatAsState    ← animar un valor numérico
animateColorAsState    ← animar un color
AnimatedContent        ← transición entre composables
animate*AsState        ← familia para cualquier tipo animable
Crossfade              ← fundido entre composables
```

---

## `AnimatedVisibility`

```kotlin
import androidx.compose.animation.*
import androidx.compose.animation.core.*
import androidx.compose.runtime.*

@Composable
fun EjemploAnimatedVisibility() {
    var visible by remember { mutableStateOf(true) }

    Column(
        Modifier.padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        Button(onClick = { visible = !visible }) {
            Text(if (visible) "Ocultar" else "Mostrar")
        }

        // Animación por defecto — fade + expand/shrink vertical
        AnimatedVisibility(visible = visible) {
            Card(Modifier.fillMaxWidth()) {
                Text("Elemento animado", Modifier.padding(16.dp))
            }
        }

        // Animación personalizada
        AnimatedVisibility(
            visible = visible,
            enter   = slideInHorizontally { -it } + fadeIn(),
            exit    = slideOutHorizontally { -it } + fadeOut()
        ) {
            Card(Modifier.fillMaxWidth()) {
                Text("Desliza desde la izquierda", Modifier.padding(16.dp))
            }
        }

        // Expandir/colapsar verticalmente
        AnimatedVisibility(
            visible = visible,
            enter   = expandVertically(expandFrom = Alignment.Top),
            exit    = shrinkVertically(shrinkTowards = Alignment.Top)
        ) {
            Card(Modifier.fillMaxWidth()) {
                Text("Expande desde arriba", Modifier.padding(16.dp))
            }
        }
    }
}
```

---

## `animate*AsState` — animar valores

```kotlin
import androidx.compose.animation.core.*
import androidx.compose.animation.animateColorAsState
import androidx.compose.animation.core.animateFloatAsState

@Composable
fun EjemploAnimateAsState() {
    var seleccionado by remember { mutableStateOf(false) }
    var progreso     by remember { mutableStateOf(0f) }

    // Animar color
    val colorFondo by animateColorAsState(
        targetValue = if (seleccionado)
            MaterialTheme.colorScheme.primaryContainer
        else
            MaterialTheme.colorScheme.surfaceVariant,
        animationSpec = tween(durationMillis = 400),
        label         = "colorFondo"
    )

    // Animar tamaño
    val tamano by animateFloatAsState(
        targetValue   = if (seleccionado) 1.1f else 1.0f,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy,
            stiffness    = Spring.StiffnessLow
        ),
        label = "tamano"
    )

    // Animar opacidad
    val opacidad by animateFloatAsState(
        targetValue = if (seleccionado) 1f else 0.5f,
        label       = "opacidad"
    )

    Column(
        Modifier.padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        // Card con color animado
        Card(
            onClick  = { seleccionado = !seleccionado },
            modifier = Modifier
                .fillMaxWidth()
                .scale(tamano),
            colors   = CardDefaults.cardColors(containerColor = colorFondo)
        ) {
            Row(
                Modifier.padding(16.dp),
                verticalAlignment = Alignment.CenterVertically,
                horizontalArrangement = Arrangement.spacedBy(12.dp)
            ) {
                Checkbox(
                    checked         = seleccionado,
                    onCheckedChange = { seleccionado = it }
                )
                Text(
                    "Seleccionable con animación",
                    modifier = Modifier.graphicsLayer(alpha = opacidad)
                )
            }
        }

        // Barra de progreso animada
        Text("Progreso: ${(progreso * 100).toInt()}%")
        LinearProgressIndicator(
            progress  = { progreso },
            modifier  = Modifier.fillMaxWidth()
        )
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { progreso = (progreso + 0.1f).coerceIn(0f, 1f) }) {
                Text("+10%")
            }
            OutlinedButton(onClick = { progreso = 0f }) {
                Text("Reset")
            }
        }
    }
}
```

---

## `AnimatedContent` — transición entre contenidos

```kotlin
@Composable
fun ContadorAnimado() {
    var cuenta by remember { mutableStateOf(0) }

    Column(
        Modifier.padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        // AnimatedContent anima la transición cuando cambia el valor
        AnimatedContent(
            targetState   = cuenta,
            transitionSpec = {
                // Determina la dirección según si sube o baja
                if (targetState > initialState) {
                    slideInVertically { it } + fadeIn() togetherWith
                    slideOutVertically { -it } + fadeOut()
                } else {
                    slideInVertically { -it } + fadeIn() togetherWith
                    slideOutVertically { it } + fadeOut()
                }
            },
            label = "contador"
        ) { valor ->
            Text(
                text  = "$valor",
                style = MaterialTheme.typography.displayLarge,
                fontWeight = FontWeight.Bold,
                color = MaterialTheme.colorScheme.primary
            )
        }

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            OutlinedButton(onClick = { cuenta-- }) { Text("−") }
            Button(onClick = { cuenta++ })         { Text("+") }
        }
    }
}
```

---

## Tema personalizado — Material 3

En Compose el tema se define con `MaterialTheme` y `ColorScheme`.
Material 3 usa la paleta **Dynamic Color** en Android 12+ y una
paleta fija en versiones anteriores:

### `ui/theme/Color.kt`

```kotlin
import androidx.compose.ui.graphics.Color

// Paleta principal
val Azul80     = Color(0xFFB3C5FF)
val AzulGris80 = Color(0xFFBBC4E2)
val Verde80    = Color(0xFFBAD9BC)

val Azul40     = Color(0xFF2B5BD3)
val AzulGris40 = Color(0xFF565E75)
val Verde40    = Color(0xFF436645)

// Colores de error
val Rojo80     = Color(0xFFFFB4AB)
val Rojo40     = Color(0xFFBA1A1A)
```

### `ui/theme/Theme.kt`

```kotlin
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import android.os.Build
import androidx.compose.ui.platform.LocalContext

private val EsquemaClaroPersonalizada = lightColorScheme(
    primary          = Azul40,
    onPrimary        = Color.White,
    primaryContainer = Azul80,
    secondary        = AzulGris40,
    tertiary         = Verde40,
    error            = Rojo40
)

private val EsquemaOscuroPersonalizada = darkColorScheme(
    primary          = Azul80,
    onPrimary        = Color(0xFF00218A),
    primaryContainer = Color(0xFF0B3FBE),
    secondary        = AzulGris80,
    tertiary         = Verde80,
    error            = Rojo80
)

@Composable
fun MiAppTheme(
    modoOscuro:    Boolean = isSystemInDarkTheme(),
    dynamicColor:  Boolean = true,   // Dynamic Color en Android 12+
    content:       @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (modoOscuro) dynamicDarkColorScheme(context)
            else            dynamicLightColorScheme(context)
        }
        modoOscuro -> EsquemaOscuroPersonalizada
        else       -> EsquemaClaroPersonalizada
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography  = TipografiaPersonalizada,
        shapes      = FormasPersonalizadas,
        content     = content
    )
}
```

### `ui/theme/Type.kt`

```kotlin
import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.Font
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

// Familia tipográfica personalizada (añadir fuente en res/font/)
val Inter = FontFamily(
    Font(R.font.inter_regular,    FontWeight.Normal),
    Font(R.font.inter_medium,     FontWeight.Medium),
    Font(R.font.inter_semibold,   FontWeight.SemiBold),
    Font(R.font.inter_bold,       FontWeight.Bold),
)

val TipografiaPersonalizada = Typography(
    displayLarge  = TextStyle(fontFamily = Inter, fontWeight = FontWeight.Bold,   fontSize = 57.sp),
    headlineLarge = TextStyle(fontFamily = Inter, fontWeight = FontWeight.SemiBold, fontSize = 32.sp),
    titleLarge    = TextStyle(fontFamily = Inter, fontWeight = FontWeight.SemiBold, fontSize = 22.sp),
    titleMedium   = TextStyle(fontFamily = Inter, fontWeight = FontWeight.Medium,  fontSize = 16.sp),
    bodyLarge     = TextStyle(fontFamily = Inter, fontWeight = FontWeight.Normal,  fontSize = 16.sp),
    bodyMedium    = TextStyle(fontFamily = Inter, fontWeight = FontWeight.Normal,  fontSize = 14.sp),
    labelLarge    = TextStyle(fontFamily = Inter, fontWeight = FontWeight.Medium,  fontSize = 14.sp),
)
```

### `ui/theme/Shape.kt`

```kotlin
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.Shapes
import androidx.compose.ui.unit.dp

val FormasPersonalizadas = Shapes(
    extraSmall = RoundedCornerShape(4.dp),
    small      = RoundedCornerShape(8.dp),
    medium     = RoundedCornerShape(12.dp),
    large      = RoundedCornerShape(16.dp),
    extraLarge = RoundedCornerShape(24.dp)
)
```

---

## Toggle de modo oscuro desde la UI

```kotlin
@Composable
fun AppConToggleTema() {
    var modoOscuro by rememberSaveable { mutableStateOf(false) }

    MiAppTheme(modoOscuro = modoOscuro) {
        Scaffold(
            topBar = {
                TopAppBar(
                    title = { Text("Mi App") },
                    actions = {
                        IconButton(onClick = { modoOscuro = !modoOscuro }) {
                            Icon(
                                imageVector = if (modoOscuro)
                                    Icons.Default.LightMode
                                else
                                    Icons.Default.DarkMode,
                                contentDescription = "Cambiar tema"
                            )
                        }
                    }
                )
            }
        ) { padding ->
            // El tema se aplica automáticamente a todos los composables hijos
            Column(Modifier.padding(padding).padding(16.dp)) {
                Text("Título", style = MaterialTheme.typography.headlineLarge)
                Text("Cuerpo", style = MaterialTheme.typography.bodyLarge)
                Button(onClick = {}) { Text("Botón primario") }
            }
        }
    }
}
```

---

## Arquitectura Clean en Android con Compose

La arquitectura limpia separa el código en capas con dependencias
en una sola dirección: UI → ViewModel → Domain → Data

```
┌─────────────────────────────────────────────┐
│  UI Layer (Compose)                         │
│  Composables + ViewModel                    │
├─────────────────────────────────────────────┤
│  Domain Layer                               │
│  Use Cases + Modelos de dominio             │
├─────────────────────────────────────────────┤
│  Data Layer                                 │
│  Repositories + DataSources (API / BD)      │
└─────────────────────────────────────────────┘
```

### Estructura de paquetes

```
app/
└── src/main/kotlin/com/ejemplo/app/
    ├── ui/
    │   ├── theme/            ← Color.kt, Theme.kt, Type.kt
    │   ├── productos/
    │   │   ├── ProductosScreen.kt
    │   │   └── ProductosViewModel.kt
    │   └── MainActivity.kt
    ├── domain/
    │   ├── model/
    │   │   └── Producto.kt   ← modelo de dominio (puro Kotlin)
    │   └── usecase/
    │       ├── ObtenerProductosUseCase.kt
    │       └── BuscarProductosUseCase.kt
    └── data/
        ├── remote/
        │   ├── ProductosApiService.kt
        │   ├── ProductoDto.kt  ← datos tal como llegan de la API
        │   └── RetrofitClient.kt
        └── repository/
            └── ProductosRepositoryImpl.kt
```

### Modelos separados por capa

```kotlin
// domain/model/Producto.kt — modelo limpio, sin dependencias de Android ni Retrofit
data class Producto(
    val id:        Int,
    val nombre:    String,
    val precio:    Double,
    val categoria: String,
    val activo:    Boolean,
    val stock:     Int
)

// data/remote/ProductoDto.kt — refleja exactamente el JSON de la API
data class ProductoDto(
    val id:            Int,
    val name:          String,
    val price:         String,    // String porque así llega de la API
    val category_name: String,
    val is_active:     Boolean,
    val stock:         Int,
    val url_image:     String
) {
    // Función de mapeo: DTO → Modelo de dominio
    fun toDomain() = Producto(
        id        = id,
        nombre    = name,
        precio    = price.toDoubleOrNull() ?: 0.0,
        categoria = category_name,
        activo    = is_active,
        stock     = stock
    )
}
```

### Interfaz del repositorio (domain) e implementación (data)

```kotlin
// domain/repository/ProductosRepository.kt — contrato (interfaz)
// La capa de dominio no sabe cómo se implementa, solo qué puede pedir
interface ProductosRepository {
    suspend fun obtenerProductos(pagina: Int, pageSize: Int): Result<List<Producto>>
    suspend fun buscar(query: String): Result<List<Producto>>
}

// data/repository/ProductosRepositoryImpl.kt — implementación concreta
class ProductosRepositoryImpl(
    private val api: ProductosApiService
) : ProductosRepository {

    override suspend fun obtenerProductos(
        pagina: Int,
        pageSize: Int
    ): Result<List<Producto>> = try {
        val response = api.obtenerProductos(pagina, pageSize)
        Result.success(response.results.map { it.toDomain() })
    } catch (e: Exception) {
        Result.failure(e)
    }

    override suspend fun buscar(query: String): Result<List<Producto>> = try {
        val response = api.obtenerProductos(search = query)
        Result.success(response.results.map { it.toDomain() })
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

### Use Cases — lógica de negocio

```kotlin
// domain/usecase/ObtenerProductosUseCase.kt
// Un Use Case = una sola responsabilidad
class ObtenerProductosUseCase(
    private val repositorio: ProductosRepository
) {
    suspend operator fun invoke(
        pagina:   Int = 1,
        pageSize: Int = 10
    ): Result<List<Producto>> {
        return repositorio.obtenerProductos(pagina, pageSize)
    }
}

// domain/usecase/BuscarProductosUseCase.kt
class BuscarProductosUseCase(
    private val repositorio: ProductosRepository
) {
    suspend operator fun invoke(query: String): Result<List<Producto>> {
        if (query.length < 2) return Result.success(emptyList())
        return repositorio.buscar(query)
    }
}
```

### `ViewModel` con Use Cases

```kotlin
// ui/productos/ProductosViewModel.kt
class ProductosViewModel(
    private val obtenerProductos: ObtenerProductosUseCase,
    private val buscarProductos:  BuscarProductosUseCase
) : ViewModel() {

    private val _estado = MutableStateFlow(EstadoProductos())
    val estado: StateFlow<EstadoProductos> = _estado.asStateFlow()

    init { cargarPagina(1) }

    fun cargarPagina(pagina: Int) {
        viewModelScope.launch {
            _estado.update { it.copy(cargando = true, error = null) }
            obtenerProductos(pagina).fold(
                onSuccess = { lista ->
                    _estado.update { it.copy(
                        cargando  = false,
                        productos = lista,
                        pagina    = pagina
                    )}
                },
                onFailure = { e ->
                    _estado.update { it.copy(
                        cargando = false,
                        error    = e.message
                    )}
                }
            )
        }
    }

    fun buscar(query: String) {
        viewModelScope.launch {
            _estado.update { it.copy(cargando = true) }
            buscarProductos(query).fold(
                onSuccess = { lista ->
                    _estado.update { it.copy(cargando = false, productos = lista) }
                },
                onFailure = { e ->
                    _estado.update { it.copy(cargando = false, error = e.message) }
                }
            )
        }
    }
}

data class EstadoProductos(
    val productos: List<Producto> = emptyList(),
    val cargando:  Boolean        = false,
    val error:     String?        = null,
    val pagina:    Int            = 1
)
```

---

## Programa completo — pantalla animada con tema

```kotlin
@Composable
fun PantallaAnimada() {
    var seccionExpandida by remember { mutableStateOf<Int?>(null) }

    val secciones = listOf(
        Triple(1, "¿Qué es Kotlin?",    "Kotlin es un lenguaje moderno, conciso y seguro para la JVM y Android."),
        Triple(2, "¿Qué es Compose?",   "Jetpack Compose es el toolkit declarativo de UI para Android con Kotlin."),
        Triple(3, "¿Qué son coroutines?","Las coroutines permiten programación asíncrona sin bloquear hilos.")
    )

    LazyColumn(
        modifier            = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        item {
            Text(
                "Preguntas frecuentes",
                style      = MaterialTheme.typography.headlineMedium,
                fontWeight = FontWeight.Bold,
                modifier   = Modifier.padding(bottom = 8.dp)
            )
        }

        items(secciones, key = { it.first }) { (id, titulo, contenido) ->
            val expandido = seccionExpandida == id

            val elevacion by animateDpAsState(
                targetValue   = if (expandido) 8.dp else 1.dp,
                animationSpec = tween(200),
                label         = "elevacion"
            )

            ElevatedCard(
                onClick   = {
                    seccionExpandida = if (expandido) null else id
                },
                modifier  = Modifier.fillMaxWidth(),
                elevation = CardDefaults.elevatedCardElevation(defaultElevation = elevacion)
            ) {
                Column(Modifier.padding(16.dp)) {
                    Row(
                        Modifier.fillMaxWidth(),
                        horizontalArrangement = Arrangement.SpaceBetween,
                        verticalAlignment     = Alignment.CenterVertically
                    ) {
                        Text(titulo, fontWeight = FontWeight.SemiBold,
                            style = MaterialTheme.typography.titleMedium)

                        // Icono que rota según el estado
                        val rotacion by animateFloatAsState(
                            targetValue   = if (expandido) 180f else 0f,
                            animationSpec = tween(200),
                            label         = "rotacion"
                        )
                        Icon(
                            Icons.Default.ExpandMore,
                            contentDescription = null,
                            modifier = Modifier.rotate(rotacion)
                        )
                    }

                    // Contenido con animación de expansión
                    AnimatedVisibility(
                        visible = expandido,
                        enter   = expandVertically() + fadeIn(),
                        exit    = shrinkVertically() + fadeOut()
                    ) {
                        Column {
                            Spacer(Modifier.height(8.dp))
                            HorizontalDivider()
                            Spacer(Modifier.height(8.dp))
                            Text(
                                contenido,
                                style = MaterialTheme.typography.bodyMedium,
                                color = MaterialTheme.colorScheme.onSurfaceVariant
                            )
                        }
                    }
                }
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun PantallaAnimadaPreview() {
    MiAppTheme {
        PantallaAnimada()
    }
}
```

---

## Ejercicios propuestos

1. **Skeleton loading** — Implementa un efecto de esqueleto de carga.
   Crea un composable `SkeletonBox(modifier)` que muestre un `Box` con
   color gris que parpadea usando `infiniteTransition` + `animateFloat`.
   Úsalo en `TarjetaProductoApi` mientras `cargando == true`.

2. **Tema dinámico persistente** — Guarda la preferencia de modo oscuro
   en `DataStore` y léela con `collectAsStateWithLifecycle`. Crea un
   `TemaViewModel` que exponga `modoOscuro: StateFlow<Boolean>` y una
   función `toggleTema()`.

3. **Arquitectura Clean completa** — Reorganiza el proyecto de la
   página 13 siguiendo la estructura de paquetes Clean: separa `Producto`
   (dominio), `ProductoDto` (datos), crea `ObtenerProductosUseCase` y
   conecta todo en el `ViewModel`. Verifica que la capa de dominio no
   tiene imports de Android.

4. **Animación compartida** — Investiga `sharedElement()` en Compose
   para animar la transición de una tarjeta de producto a la pantalla
   de detalle. La imagen debe "volar" desde la lista hasta ocupar la
   parte superior del detalle.

---

## Resumen de la página 14

- `AnimatedVisibility` anima la aparición/desaparición con `enter`/`exit` personalizables — `slideIn`, `fadeIn`, `expandVertically` y sus contrapartes.
- `animate*AsState` interpola cualquier valor animable entre el estado actual y el nuevo. `label` es requerido para el profiler.
- `animationSpec` define la curva: `tween(durationMillis)` para duración fija, `spring(dampingRatio, stiffness)` para rebote físico.
- `AnimatedContent` anima la transición cuando cambia el `targetState` — `transitionSpec` permite diferente animación según la dirección del cambio.
- El tema Material 3 se define con `lightColorScheme`/`darkColorScheme` en `Theme.kt`. Dynamic Color en Android 12+ usa los colores del fondo de pantalla del usuario.
- La tipografía y las formas se personalizan con `Typography` y `Shapes` pasados a `MaterialTheme`.
- La arquitectura Clean separa en tres capas: UI (Compose + ViewModel), Domain (Use Cases + modelos), Data (API + BD + Repositorio).
- Los **DTO** (Data Transfer Objects) pertenecen a la capa de datos y tienen una función `toDomain()` que los convierte al modelo de dominio.
- La interfaz del repositorio vive en el dominio — el ViewModel depende de la abstracción, no de la implementación concreta.
- Los **Use Cases** con `operator fun invoke()` permiten llamarlos como si fueran funciones: `obtenerProductos()` en lugar de `obtenerProductos.execute()`.

---

> **Siguiente página →** Página 15: Proyecto integrador — app completa de
> gestión de productos con arquitectura Clean, Compose, Navigation, API REST
> y tests.