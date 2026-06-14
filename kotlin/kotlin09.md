# Curso Kotlin — Página 9
## Módulo 4 · Concurrencia
### Coroutines: `suspend`, `launch`, `async`, `Flow` y manejo de errores

---

## ¿Por qué coroutines?

En una aplicación Android, las operaciones lentas (red, base de datos,
archivos) no pueden ejecutarse en el hilo principal — bloquearían la UI
y el sistema mataría la app. La solución clásica era callbacks o threads
explícitos. Las coroutines ofrecen una solución más limpia:

```
Problema          Solución clásica      Solución con coroutines
──────────────    ─────────────────     ──────────────────────────────
Callback hell     Anidamiento profundo  Código secuencial con suspend
Thread management new Thread { ... }   launch { } — gestión automática
Cancelación       Flags manuales        Cancelación estructurada
Error handling    try/catch anidados    try/catch normal en suspend
```

```kotlin
// Sin coroutines — callback hell
fun cargarUsuario(id: Int, callback: (Usuario?) -> Unit) {
    thread {
        val usuario = api.obtenerUsuario(id)  // bloquea el thread
        runOnMainThread { callback(usuario) }
    }
}

// Con coroutines — código secuencial, sin bloqueos
suspend fun cargarUsuario(id: Int): Usuario? {
    return api.obtenerUsuario(id)  // suspende sin bloquear
}
```

---

## Instalación

```kotlin
// build.gradle.kts (módulo app)
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
}
```

---

## `suspend` — funciones que pueden pausarse

Una función `suspend` puede **pausarse** sin bloquear el thread.
Mientras está pausada, el thread puede ejecutar otro trabajo:

```kotlin
import kotlinx.coroutines.*

// suspend — puede pausarse y reanudarse
// Solo puede llamarse desde otra función suspend o desde una coroutine
suspend fun obtenerDato(): String {
    delay(1000)   // suspende la coroutine 1 segundo sin bloquear el thread
    return "Dato obtenido"
}

suspend fun calcular(n: Int): Int {
    delay(500)
    return n * n
}

fun main() = runBlocking {   // crea una coroutine y bloquea hasta que termina
    println("Iniciando...")
    val resultado = obtenerDato()   // pausa aquí 1 segundo
    println(resultado)              // Dato obtenido
    println("Fin")
}
```

> `runBlocking` es para tests y la función `main`. En Android
> **nunca** se usa en el hilo principal — usarás `viewModelScope.launch`.

---

## `CoroutineScope` y `launch`

`launch` inicia una coroutine que **no devuelve valor** (fire and forget).
Las coroutines viven dentro de un `CoroutineScope` que controla su ciclo de vida:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    println("Inicio en ${Thread.currentThread().name}")

    // launch — inicia una coroutine concurrente sin esperar el resultado
    val job1 = launch {
        delay(1000)
        println("Job 1 terminó en ${Thread.currentThread().name}")
    }

    val job2 = launch {
        delay(500)
        println("Job 2 terminó en ${Thread.currentThread().name}")
    }

    println("Ambos jobs lanzados — siguiendo sin esperar")

    // join() — espera a que el job termine
    job1.join()
    job2.join()

    println("Ambos jobs completados")
}
```

Salida (los jobs corren concurrentemente):
```
Inicio en main
Ambos jobs lanzados — siguiendo sin esperar
Job 2 terminó en main     ← termina primero (delay 500ms)
Job 1 terminó en main     ← termina después (delay 1000ms)
Ambos jobs completados
```

---

## `Dispatchers` — en qué thread corre la coroutine

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // Dispatchers.Default — CPU intensiva (cálculos, procesamiento)
    launch(Dispatchers.Default) {
        println("Default: ${Thread.currentThread().name}")
        val resultado = (1..1_000_000).sum()
        println("Suma: $resultado")
    }

    // Dispatchers.IO — operaciones de entrada/salida (red, archivos, BD)
    launch(Dispatchers.IO) {
        println("IO: ${Thread.currentThread().name}")
        delay(100)   // simula lectura de archivo
        println("Archivo leído")
    }

    // Dispatchers.Main — hilo principal de Android (actualizar UI)
    // Solo disponible en Android — en tests usa Dispatchers.Main.immediate
    // launch(Dispatchers.Main) { textView.text = "Actualizado" }

    // Cambiar dispatcher dentro de una coroutine
    launch {
        println("Iniciando en: ${Thread.currentThread().name}")
        val datos = withContext(Dispatchers.IO) {
            delay(200)   // simula operación IO
            "Datos de la BD"
        }
        // De vuelta al dispatcher original
        println("Procesando en: ${Thread.currentThread().name}")
        println("Datos: $datos")
    }

    delay(500)  // esperar que todos terminen
}
```

---

## `async` y `await` — concurrencia con valor de retorno

`async` inicia una coroutine que **devuelve un valor** a través de un `Deferred<T>`.
`await()` suspende hasta tener el resultado:

```kotlin
import kotlinx.coroutines.*

suspend fun obtenerNombre(id: Int): String {
    delay(1000)   // simula llamada a API
    return when (id) {
        1 -> "Ana García"
        2 -> "Luis Martínez"
        else -> "Desconocido"
    }
}

suspend fun obtenerEmail(id: Int): String {
    delay(800)    // simula llamada a BD
    return "usuario$id@test.com"
}

fun main() = runBlocking {

    // ❌ SECUENCIAL — espera 1000ms + 800ms = 1800ms
    val tiempoSecuencial = measureTimeMillis {
        val nombre = obtenerNombre(1)
        val email  = obtenerEmail(1)
        println("$nombre — $email")
    }
    println("Tiempo secuencial: ${tiempoSecuencial}ms")

    // ✅ CONCURRENTE con async — espera max(1000, 800) = ~1000ms
    val tiempoConcurrente = measureTimeMillis {
        val nombreDeferred = async { obtenerNombre(2) }
        val emailDeferred  = async { obtenerEmail(2) }

        // await() suspende hasta tener el resultado
        val nombre = nombreDeferred.await()
        val email  = emailDeferred.await()
        println("$nombre — $email")
    }
    println("Tiempo concurrente: ${tiempoConcurrente}ms")
}
```

### `awaitAll` — esperar múltiples resultados

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val ids = listOf(1, 2, 3, 4, 5)

    // Lanzar todas las peticiones concurrentemente
    val deferreds = ids.map { id ->
        async { obtenerNombre(id) }
    }

    // Esperar todos los resultados a la vez
    val nombres = deferreds.awaitAll()
    println(nombres)
    // [Ana García, Luis Martínez, Desconocido, Desconocido, Desconocido]
}
```

---

## Cancelación estructurada

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch {
        repeat(10) { i ->
            println("Iteración $i")
            delay(500)
        }
    }

    delay(1700)   // deja correr 3 iteraciones y media
    println("Cancelando...")
    job.cancel()  // cancela la coroutine
    job.join()    // espera a que complete la cancelación
    println("Cancelado")
}
// Iteración 0, 1, 2, 3 → Cancelando → Cancelado
```

### Verificar cancelación con `isActive`

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val job = launch(Dispatchers.Default) {
        var i = 0
        // isActive es false cuando la coroutine fue cancelada
        while (isActive) {
            i++
            if (i % 100_000 == 0) println("Progreso: $i")
        }
        println("Loop terminado por cancelación")
    }

    delay(50)
    job.cancel()
    job.join()
    println("Job cancelado")
}
```

---

## Manejo de errores

```kotlin
import kotlinx.coroutines.*

// try/catch funciona normalmente con coroutines
fun main() = runBlocking {

    // En launch — el error se propaga al scope padre
    val handler = CoroutineExceptionHandler { _, excepcion ->
        println("Excepción capturada: ${excepcion.message}")
    }

    val job = launch(handler) {
        println("Iniciando trabajo arriesgado")
        delay(500)
        throw RuntimeException("Algo salió mal")
        println("Esto nunca se ejecuta")
    }
    job.join()
    println("Continuando después del error")

    // En async — el error se lanza cuando llamas a await()
    val deferred = async {
        delay(300)
        throw IllegalStateException("Error en async")
        42
    }

    try {
        val resultado = deferred.await()
        println("Resultado: $resultado")
    } catch (e: IllegalStateException) {
        println("Capturado en await: ${e.message}")
    }

    // supervisorScope — los hijos fallan independientemente
    supervisorScope {
        val hijo1 = launch {
            delay(100)
            throw RuntimeException("Hijo 1 falló")
        }
        val hijo2 = launch {
            delay(200)
            println("Hijo 2 completado — a pesar de que hijo 1 falló")
        }
        hijo1.join()
        hijo2.join()
    }
}
```

---

## `Flow` — flujo de datos asíncronos

`Flow<T>` es la versión asíncrona de `Sequence<T>`.
Emite múltiples valores a lo largo del tiempo de forma reactiva:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// flow { } construye un flujo
fun numerosFlow(): Flow<Int> = flow {
    for (i in 1..5) {
        delay(300)
        emit(i)         // emite un valor — equivale a yield en Sequence
        println("Emitido: $i")
    }
}

fun main() = runBlocking {
    numerosFlow()
        .collect { valor ->   // collect — suscribe y recibe cada valor
            println("Recibido: $valor")
        }
}
```

### Operadores de `Flow`

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    val flow = flow {
        for (i in 1..10) {
            emit(i)
            delay(100)
        }
    }

    // Los mismos operadores que las colecciones — pero asíncronos
    flow
        .filter { it % 2 == 0 }
        .map    { it * it }
        .take   (3)
        .collect { println(it) }
    // 4, 16, 36

    // flowOf — flow de valores fijos
    flowOf("Ana", "Luis", "María")
        .map    { it.uppercase() }
        .collect { println(it) }

    // toList() — materializa el flow en una lista
    val lista = flow { repeat(5) { emit(it) } }.toList()
    println(lista)  // [0, 1, 2, 3, 4]
}
```

---

## `StateFlow` — estado reactivo

`StateFlow<T>` es un `Flow` especializado para representar **estado** que
persiste y siempre tiene un valor actual. Es el mecanismo principal de
Jetpack Compose para reaccionar a cambios de estado:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

class ContadorViewModel {
    // MutableStateFlow — mutable internamente
    private val _cuenta = MutableStateFlow(0)
    // StateFlow — expuesto como solo lectura
    val cuenta: StateFlow<Int> = _cuenta.asStateFlow()

    fun incrementar() { _cuenta.value++ }
    fun decrementar() { _cuenta.value-- }
    fun resetear()    { _cuenta.value = 0 }
}

fun main() = runBlocking {
    val viewModel = ContadorViewModel()

    // Observar — se ejecuta cada vez que cambia el valor
    val job = launch {
        viewModel.cuenta.collect { valor ->
            println("Cuenta actualizada: $valor")
        }
    }

    delay(100)
    viewModel.incrementar()  // Cuenta actualizada: 1
    viewModel.incrementar()  // Cuenta actualizada: 2
    viewModel.decrementar()  // Cuenta actualizada: 1
    viewModel.resetear()     // Cuenta actualizada: 0

    job.cancel()
}
```

---

## Patrón completo — repositorio con coroutines

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Modelo
data class Producto(val id: Int, val nombre: String, val precio: Double)

// Simulación de API remota
object ApiProductos {
    private val productos = listOf(
        Producto(1, "Teclado", 89.99),
        Producto(2, "Mouse",   29.99),
        Producto(3, "Monitor", 349.99),
        Producto(4, "Webcam",  59.99),
    )

    suspend fun obtenerProductos(): List<Producto> {
        delay(800)   // simula latencia de red
        return productos
    }

    suspend fun obtenerProducto(id: Int): Producto? {
        delay(300)
        return productos.find { it.id == id }
    }
}

// Repositorio — abstrae el origen de los datos
class RepositorioProductos {

    // Flow que emite el estado de carga
    fun obtenerProductosFlow(): Flow<Result<List<Producto>>> = flow {
        emit(Result.Loading)
        try {
            val productos = ApiProductos.obtenerProductos()
            emit(Result.Exito(productos))
        } catch (e: Exception) {
            emit(Result.Error(e.message ?: "Error desconocido"))
        }
    }
}

// Estado sellado para representar los posibles estados
sealed class Result<out T> {
    object Loading                           : Result<Nothing>()
    data class Exito<T>(val datos: T)        : Result<T>()
    data class Error(val mensaje: String)    : Result<Nothing>()
}

// ViewModel simulado
class ProductosViewModel(
    private val repositorio: RepositorioProductos = RepositorioProductos()
) {
    private val _estado = MutableStateFlow<Result<List<Producto>>>(Result.Loading)
    val estado: StateFlow<Result<List<Producto>>> = _estado.asStateFlow()

    // En Android esto sería viewModelScope.launch
    fun cargar(scope: CoroutineScope) {
        scope.launch {
            repositorio.obtenerProductosFlow().collect { resultado ->
                _estado.value = resultado
            }
        }
    }
}

fun main() = runBlocking {
    val viewModel = ProductosViewModel()
    viewModel.cargar(this)

    // Observar el estado
    viewModel.estado.collect { estado ->
        when (estado) {
            is Result.Loading      -> println("⏳ Cargando productos...")
            is Result.Exito        -> {
                println("✅ ${estado.datos.size} productos cargados:")
                estado.datos.forEach { println("  ${it.nombre}: $${it.precio}") }
                cancel()   // cancela el scope para salir del collect
            }
            is Result.Error        -> println("❌ Error: ${estado.mensaje}")
        }
    }
}
```

---

## Ejercicios propuestos

1. **Descarga concurrente** — Simula la descarga de 5 archivos con tiempos
   distintos usando `async`. Muestra cuánto tardaría de forma secuencial vs
   concurrente. Usa `measureTimeMillis` para comparar.

2. **Flow de sensor** — Crea un `Flow<Double>` que emita lecturas de temperatura
   simuladas cada 500ms con `generateSequence`. Aplica `filter` para conservar
   solo lecturas anómalas (fuera de 15..30°C) y `take(3)` para recoger las
   primeras 3. Cancela el flow después de 10 segundos.

3. **`StateFlow` con historial** — Extiende `ContadorViewModel` para que mantenga
   un `historial: StateFlow<List<Int>>` con los últimos 5 valores. Cada cambio
   en `cuenta` actualiza el historial.

4. **Repositorio con caché** — Modifica `RepositorioProductos` para que guarde
   el resultado en un `Map` en memoria. En llamadas posteriores, emite los datos
   del caché inmediatamente antes de hacer la llamada remota (patrón
   emit-caché → emit-red).

---

## Resumen de la página 9

- `suspend` marca una función que puede pausarse sin bloquear el thread. Solo puede llamarse desde otra `suspend` o desde una coroutine.
- `runBlocking` crea un scope bloqueante — solo para `main()` y tests. En Android se usa `viewModelScope` o `lifecycleScope`.
- `launch` inicia una coroutine sin valor de retorno. `async` devuelve un `Deferred<T>` — llama `await()` para obtener el resultado.
- `async` + `await` permite ejecutar operaciones independientes **concurrentemente** en lugar de esperar una a una.
- `Dispatchers.IO` para red y archivos, `Dispatchers.Default` para cálculo intensivo, `Dispatchers.Main` para actualizar la UI en Android.
- `withContext(Dispatcher)` cambia el dispatcher dentro de una coroutine sin crear una nueva.
- La cancelación es **estructurada** — cancelar un scope cancela todos sus hijos. `isActive` permite responder a la cancelación.
- `supervisorScope` aísla los fallos — un hijo que falla no cancela a los demás.
- `Flow<T>` emite múltiples valores asíncronos. Soporta los mismos operadores que las colecciones: `filter`, `map`, `take`.
- `StateFlow<T>` es el estado reactivo — siempre tiene un valor actual y notifica a sus colectores cuando cambia.

---

> **Siguiente página →** Página 10: Testing — JUnit 5, Kotest,
> `runTest` para coroutines y MockK.