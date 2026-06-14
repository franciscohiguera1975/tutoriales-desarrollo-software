# Curso Kotlin — Página 10
## Módulo 5 · Testing
### JUnit 5, Kotest, `runTest` para coroutines y MockK

---

## ¿Por qué hacer tests?

Un test es código que verifica que otro código se comporta como se espera.
En Kotlin los tests son parte natural del proyecto — no una tarea adicional:

```
Sin tests                         Con tests
─────────────────────────         ────────────────────────────────
Cambias código → rezas            Cambias código → los tests te avisan
Bug en producción a las 3am       Bug detectado antes de hacer commit
Refactoring da miedo              Refactoring con confianza
"Funciona en mi máquina"          "Funciona porque lo demuestran los tests"
```

---

## Configuración

```kotlin
// build.gradle.kts (módulo app)
dependencies {
    // JUnit 5 — framework base de testing
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.0")
    testImplementation("org.junit.jupiter:junit-jupiter-params:5.11.0")

    // Kotest — assertions más expresivas (opcional pero recomendado)
    testImplementation("io.kotest:kotest-runner-junit5:5.9.1")
    testImplementation("io.kotest:kotest-assertions-core:5.9.1")

    // Coroutines test — runTest y TestDispatcher
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")

    // MockK — mocks idiomáticos para Kotlin
    testImplementation("io.mockk:mockk:1.13.12")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

---

## Estructura de archivos de test

```
src/
├── main/kotlin/
│   ├── models/Producto.kt
│   ├── repositories/ProductoRepository.kt
│   └── services/ProductoService.kt
└── test/kotlin/
    ├── models/ProductoTest.kt
    ├── repositories/ProductoRepositoryTest.kt
    └── services/ProductoServiceTest.kt
```

El archivo de test vive en `src/test/kotlin/` con el mismo nombre
que la clase que prueba más el sufijo `Test`.

---

## JUnit 5 — tests básicos

```kotlin
// src/test/kotlin/models/ProductoTest.kt

import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.DisplayName
import org.junit.jupiter.api.Nested

// La clase bajo test
data class Producto(
    val id:     Int,
    val nombre: String,
    val precio: Double,
    val stock:  Int,
    val activo: Boolean = true
) {
    val disponible:  Boolean get() = activo && stock > 0
    val precioConIva: Double get() = precio * 1.19

    fun aplicarDescuento(pct: Double): Producto {
        require(pct in 0.0..100.0) { "Descuento debe ser 0–100" }
        return copy(precio = precio * (1 - pct / 100))
    }
}

@DisplayName("Tests de Producto")
class ProductoTest {

    // Fixture compartida — se recrea antes de cada test
    private lateinit var producto: Producto

    @BeforeEach
    fun setUp() {
        producto = Producto(1, "Teclado mecánico", 89.99, 10)
    }

    @AfterEach
    fun tearDown() {
        // Limpiar recursos si fuera necesario
    }

    // ─── Tests básicos ──────────────────────────────────────────────

    @Test
    @DisplayName("un producto con stock > 0 está disponible")
    fun `producto con stock esta disponible`() {
        assertTrue(producto.disponible)
    }

    @Test
    @DisplayName("un producto sin stock no está disponible")
    fun `producto sin stock no esta disponible`() {
        val sinStock = producto.copy(stock = 0)
        assertFalse(sinStock.disponible)
    }

    @Test
    @DisplayName("precioConIva añade el 19%")
    fun `precioConIva aplica 19 porciento`() {
        val esperado = 89.99 * 1.19
        assertEquals(esperado, producto.precioConIva, 0.001)
        //                                             ↑ tolerancia para Double
    }

    @Test
    @DisplayName("aplicarDescuento reduce el precio correctamente")
    fun `aplicarDescuento reduce precio`() {
        val conDescuento = producto.aplicarDescuento(10.0)
        assertEquals(80.991, conDescuento.precio, 0.001)
        assertEquals(producto.nombre, conDescuento.nombre)  // nombre intacto
    }

    @Test
    @DisplayName("descuento mayor que 100 lanza excepción")
    fun `descuento invalido lanza excepcion`() {
        assertThrows<IllegalArgumentException> {
            producto.aplicarDescuento(110.0)
        }
    }

    // ─── Tests anidados — agrupa por escenario ──────────────────────

    @Nested
    @DisplayName("cuando el producto está inactivo")
    inner class ProductoInactivo {

        private lateinit var inactivo: Producto

        @BeforeEach
        fun setUp() {
            inactivo = producto.copy(activo = false)
        }

        @Test
        fun `no esta disponible aunque tenga stock`() {
            assertFalse(inactivo.disponible)
        }

        @Test
        fun `sigue teniendo precio con IVA calculable`() {
            assertEquals(producto.precioConIva, inactivo.precioConIva, 0.001)
        }
    }
}
```

---

## Tests parametrizados

```kotlin
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.CsvSource
import org.junit.jupiter.params.provider.ValueSource
import org.junit.jupiter.params.provider.MethodSource
import org.junit.jupiter.api.Assertions.*

class CalculadoraTest {

    fun sumar(a: Double, b: Double) = a + b
    fun esPrimo(n: Int): Boolean {
        if (n < 2) return false
        return (2 until n).none { n % it == 0 }
    }

    // @ValueSource — un parámetro
    @ParameterizedTest(name = "{0} es primo")
    @ValueSource(ints = [2, 3, 5, 7, 11, 13])
    fun `numeros primos conocidos`(n: Int) {
        assertTrue(esPrimo(n))
    }

    @ParameterizedTest(name = "{0} NO es primo")
    @ValueSource(ints = [0, 1, 4, 6, 8, 9, 10])
    fun `numeros no primos conocidos`(n: Int) {
        assertFalse(esPrimo(n))
    }

    // @CsvSource — múltiples parámetros
    @ParameterizedTest(name = "{0} + {1} = {2}")
    @CsvSource(
        "1.0,  2.0,  3.0",
        "0.0,  0.0,  0.0",
        "-1.0, 1.0,  0.0",
        "10.0, 5.5,  15.5"
    )
    fun `suma correcta`(a: Double, b: Double, esperado: Double) {
        assertEquals(esperado, sumar(a, b), 0.001)
    }

    // @MethodSource — fuente de datos desde una función
    companion object {
        @JvmStatic
        fun datosPalindromos() = listOf(
            org.junit.jupiter.params.provider.Arguments.of("racecar",          true),
            org.junit.jupiter.params.provider.Arguments.of("Kotlin",           false),
            org.junit.jupiter.params.provider.Arguments.of("anita lava tina",  true),
        )
    }

    @ParameterizedTest(name = "\"{0}\" palindromo: {1}")
    @MethodSource("datosPalindromos")
    fun `detectar palindromos`(texto: String, esperado: Boolean) {
        val limpio = texto.lowercase().filter { it.isLetter() }
        assertEquals(esperado, limpio == limpio.reversed())
    }
}
```

---

## Kotest — assertions más expresivas

Kotest ofrece una API fluida que hace los tests más legibles:

```kotlin
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import io.kotest.matchers.doubles.plusOrMinus
import io.kotest.matchers.collections.*
import io.kotest.matchers.string.*
import io.kotest.assertions.throwables.shouldThrow
import io.kotest.assertions.throwables.shouldThrowExactly
import org.junit.jupiter.api.Test

class ProductoKotestTest {

    private val producto = Producto(1, "Teclado", 89.99, 10)

    @Test
    fun `assertions con Kotest son mas legibles`() {
        // shouldBe — equivale a assertEquals
        producto.nombre   shouldBe "Teclado"
        producto.stock    shouldBe 10
        producto.disponible shouldBe true

        // shouldNotBe
        producto.precio shouldNotBe 0.0

        // Double con tolerancia
        producto.precioConIva shouldBe (107.09 plusOrMinus 0.01)

        // Colecciones
        val nombres = listOf("Ana", "Luis", "María")
        nombres shouldHaveSize 3
        nombres shouldContain "Ana"
        nombres shouldNotContain "Carlos"
        nombres.shouldBeSorted()

        // Strings
        "Kotlin es genial" shouldContain "genial"
        "Kotlin" shouldStartWith "Ko"
        "Kotlin" shouldEndWith "lin"
        "Kotlin" shouldHaveLength 6
        "KOTLIN".shouldBeUpperCase()
    }

    @Test
    fun `shouldThrow con Kotest`() {
        // shouldThrow devuelve la excepción para hacer más assertions
        val excepcion = shouldThrow<IllegalArgumentException> {
            producto.aplicarDescuento(150.0)
        }
        excepcion.message shouldContain "Descuento debe ser"
    }

    @Test
    fun `assertions multiples con assertSoftly`() {
        // Sin assertSoftly: falla en el primer assert
        // Con assertSoftly: ejecuta TODOS y reporta todos los fallos juntos
        io.kotest.assertions.assertSoftly(producto) {
            nombre   shouldBe "Teclado"
            precio   shouldBe (89.99 plusOrMinus 0.01)
            stock    shouldBe 10
            activo   shouldBe true
            disponible shouldBe true
        }
    }
}
```

---

## Testing de coroutines con `runTest`

```kotlin
import kotlinx.coroutines.test.*
import kotlinx.coroutines.*
import org.junit.jupiter.api.Test
import io.kotest.matchers.shouldBe
import io.kotest.matchers.collections.shouldHaveSize

// Clase a testear
class ProductoRepository {
    private val cache = mutableListOf<Producto>()

    suspend fun cargar(): List<Producto> {
        delay(1000)   // simula latencia de red — en tests se maneja con TestCoroutineScheduler
        return listOf(
            Producto(1, "Teclado", 89.99, 10),
            Producto(2, "Mouse",   29.99, 5)
        )
    }

    suspend fun guardar(producto: Producto) {
        delay(200)
        cache.add(producto)
    }

    fun contarEnCache() = cache.size
}

class ProductoRepositoryTest {

    private val repositorio = ProductoRepository()

    @Test
    fun `cargar devuelve productos`() = runTest {
        // runTest ejecuta el bloque en un TestCoroutineScheduler
        // delay() en el código bajo test se adelanta virtualmente — no espera 1 segundo real
        val productos = repositorio.cargar()

        productos shouldHaveSize 2
        productos[0].nombre shouldBe "Teclado"
        productos[1].nombre shouldBe "Mouse"
    }

    @Test
    fun `guardar agrega al cache`() = runTest {
        val nuevo = Producto(3, "Monitor", 349.99, 3)
        repositorio.guardar(nuevo)
        repositorio.contarEnCache() shouldBe 1
    }

    @Test
    fun `multiples operaciones concurrentes`() = runTest {
        // async dentro de runTest funciona correctamente
        val deferred1 = async { repositorio.cargar() }
        val deferred2 = async { repositorio.cargar() }

        val resultado1 = deferred1.await()
        val resultado2 = deferred2.await()

        resultado1 shouldHaveSize 2
        resultado2 shouldHaveSize 2
    }
}
```

### Testing de `StateFlow`

```kotlin
import kotlinx.coroutines.test.*
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import org.junit.jupiter.api.Test
import io.kotest.matchers.shouldBe

class ContadorViewModel {
    private val _cuenta = MutableStateFlow(0)
    val cuenta: StateFlow<Int> = _cuenta.asStateFlow()

    fun incrementar() { _cuenta.value++ }
    fun decrementar() { if (_cuenta.value > 0) _cuenta.value-- }
    fun resetear()    { _cuenta.value = 0 }
}

class ContadorViewModelTest {

    @Test
    fun `incrementar aumenta la cuenta`() = runTest {
        val vm = ContadorViewModel()
        vm.cuenta.value shouldBe 0
        vm.incrementar()
        vm.cuenta.value shouldBe 1
        vm.incrementar()
        vm.cuenta.value shouldBe 2
    }

    @Test
    fun `decrementar no baja de cero`() = runTest {
        val vm = ContadorViewModel()
        vm.decrementar()
        vm.cuenta.value shouldBe 0   // no es -1
    }

    @Test
    fun `resetear vuelve a cero`() = runTest {
        val vm = ContadorViewModel()
        repeat(5) { vm.incrementar() }
        vm.cuenta.value shouldBe 5
        vm.resetear()
        vm.cuenta.value shouldBe 0
    }

    @Test
    fun `StateFlow emite valores correctamente`() = runTest {
        val vm = ContadorViewModel()
        val valores = mutableListOf<Int>()

        // turbine-style: recoger emisiones durante el test
        val job = launch {
            vm.cuenta.take(4).toList(valores)
        }

        vm.incrementar()
        vm.incrementar()
        vm.incrementar()

        job.join()
        valores shouldBe listOf(0, 1, 2, 3)
    }
}
```

---

## MockK — mocks idiomáticos para Kotlin

`MockK` permite crear objetos falsos que simulan dependencias:

```kotlin
import io.mockk.*
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test
import io.kotest.matchers.shouldBe
import io.kotest.matchers.collections.shouldHaveSize

// Dependencias — interfaces para facilitar el mocking
interface ApiProductos {
    suspend fun obtenerTodos(): List<Producto>
    suspend fun obtener(id: Int): Producto?
    suspend fun crear(producto: Producto): Producto
}

interface Logger {
    fun log(mensaje: String)
}

// Clase bajo test — depende de ApiProductos y Logger
class ProductoService(
    private val api:    ApiProductos,
    private val logger: Logger
) {
    suspend fun obtenerDisponibles(): List<Producto> {
        logger.log("Obteniendo productos disponibles")
        return api.obtenerTodos().filter { it.disponible }
    }

    suspend fun obtenerPorId(id: Int): Producto? {
        val producto = api.obtener(id)
        if (producto == null) logger.log("Producto $id no encontrado")
        return producto
    }

    suspend fun crearProducto(nombre: String, precio: Double, stock: Int): Producto {
        require(nombre.isNotBlank()) { "Nombre vacío" }
        require(precio > 0)          { "Precio inválido" }
        val nuevo = Producto(0, nombre, precio, stock)
        val creado = api.crear(nuevo)
        logger.log("Producto creado: ${creado.id}")
        return creado
    }
}

class ProductoServiceTest {

    // mockk() crea un mock de la interfaz
    private val apiMock    = mockk<ApiProductos>()
    private val loggerMock = mockk<Logger>(relaxed = true)  // relaxed: ignora llamadas no configuradas
    private val service    = ProductoService(apiMock, loggerMock)

    private val productosFixture = listOf(
        Producto(1, "Teclado", 89.99, 10, activo = true),
        Producto(2, "Mouse",   29.99,  0, activo = true),   // sin stock
        Producto(3, "Monitor", 349.99, 5, activo = false),  // inactivo
    )

    @Test
    fun `obtenerDisponibles filtra correctamente`() = runTest {
        // given — configurar el mock
        coEvery { apiMock.obtenerTodos() } returns productosFixture

        // when — ejecutar
        val resultado = service.obtenerDisponibles()

        // then — verificar resultado
        resultado shouldHaveSize 1
        resultado[0].nombre shouldBe "Teclado"

        // verificar que se llamó al logger
        verify { loggerMock.log("Obteniendo productos disponibles") }

        // verificar que se llamó a la API exactamente una vez
        coVerify(exactly = 1) { apiMock.obtenerTodos() }
    }

    @Test
    fun `obtenerPorId existente devuelve producto`() = runTest {
        val esperado = productosFixture[0]
        coEvery { apiMock.obtener(1) } returns esperado

        val resultado = service.obtenerPorId(1)

        resultado shouldBe esperado
        verify(exactly = 0) { loggerMock.log(any()) }  // sin log si se encontró
    }

    @Test
    fun `obtenerPorId inexistente devuelve null y loguea`() = runTest {
        coEvery { apiMock.obtener(99) } returns null

        val resultado = service.obtenerPorId(99)

        resultado shouldBe null
        verify { loggerMock.log("Producto 99 no encontrado") }
    }

    @Test
    fun `crearProducto llama a la api y loguea`() = runTest {
        val creado = Producto(10, "Webcam", 59.99, 8)
        coEvery { apiMock.crear(any()) } returns creado

        val resultado = service.crearProducto("Webcam", 59.99, 8)

        resultado shouldBe creado
        coVerify { apiMock.crear(match { it.nombre == "Webcam" }) }
        verify { loggerMock.log("Producto creado: 10") }
    }

    @Test
    fun `crearProducto con nombre vacio lanza excepcion`() = runTest {
        val excepcion = io.kotest.assertions.throwables.shouldThrow<IllegalArgumentException> {
            service.crearProducto("", 50.0, 5)
        }
        excepcion.message shouldBe "Nombre vacío"
        coVerify(exactly = 0) { apiMock.crear(any()) }   // nunca llamó a la API
    }
}
```

---

## Estructura recomendada de un test — patrón AAA

```kotlin
@Test
fun `descripcion clara de lo que testea`() = runTest {
    // ARRANGE — preparar el estado inicial y los mocks
    val producto = Producto(1, "Teclado", 89.99, 10)
    coEvery { apiMock.obtener(1) } returns producto

    // ACT — ejecutar la acción bajo test
    val resultado = service.obtenerPorId(1)

    // ASSERT — verificar el resultado
    resultado shouldBe producto
}
```

Reglas para nombres de tests:
```kotlin
// ✅ Nombres descriptivos con backticks — describen el comportamiento
fun `producto sin stock no esta disponible`() { }
fun `descuento mayor que 100 lanza excepcion`() { }
fun `cargar devuelve lista vacia si la API falla`() { }

// ❌ Nombres crípticos que no dicen qué testan
fun testDisponible() { }
fun test1() { }
fun checkDescuento() { }
```

---

## Ejercicios propuestos

1. **Tests de `Temperatura`** — Escribe tests para la clase `Temperatura`
   de la página 5. Cubre: conversión a Fahrenheit y Kelvin, setter con
   valor inválido (bajo −273.15°C), y que `descripcion` devuelve el texto
   correcto para cada rango.

2. **Tests parametrizados de `FizzBuzz`** — Implementa `fizzBuzz(n: Int): String`
   y escribe tests parametrizados con `@CsvSource` para los casos: múltiplos
   de 3, de 5, de 15 y valores normales.

3. **Tests del `ContadorViewModel`** — Añade tests para un `ViewModel` con
   `decrement` que no permite negativos, `reset` y el historial de los
   últimos 5 valores. Usa `StateFlow.take(n).toList()` para verificar las emisiones.

4. **MockK completo** — Crea una interfaz `RepositorioUsuarios` con
   `suspend fun buscar(nombre: String): List<Usuario>` y `suspend fun guardar(u: Usuario)`.
   Implementa `ServicioUsuarios` que la use. Escribe tests con MockK que
   verifiquen: que `buscar` llama al repositorio, que un resultado vacío
   devuelve un mensaje específico, y que `guardar` no se llama si la
   validación falla.

---

## Resumen de la página 10

- Los tests en Kotlin viven en `src/test/kotlin/` con la misma estructura de paquetes que el código principal.
- `@Test`, `@BeforeEach`, `@AfterEach` y `@Nested` son las anotaciones más usadas de JUnit 5.
- `@ParameterizedTest` con `@ValueSource`, `@CsvSource` y `@MethodSource` elimina la repetición de tests similares.
- Kotest ofrece una API fluida: `valor shouldBe esperado`, `shouldThrow<Tipo> { }`, `assertSoftly` para reportar todos los fallos juntos.
- `runTest` es el equivalente de `runBlocking` para tests — adelanta los `delay()` virtualmente sin esperar tiempo real.
- Para testear `StateFlow`: accede a `.value` directamente o usa `take(n).toList()` para verificar secuencias de emisiones.
- `mockk<Interfaz>()` crea un mock. `coEvery { mock.funcion() } returns valor` configura respuestas de funciones `suspend`.
- `verify { mock.funcion() }` comprueba que se llamó. `coVerify(exactly = 0)` verifica que **no** se llamó.
- El patrón AAA (Arrange, Act, Assert) hace los tests predecibles y fáciles de leer.
- Los nombres de test con backticks describen el comportamiento esperado como una frase completa.

---

> **Siguiente página →** Página 11: Jetpack Compose — fundamentos,
> composables, `remember`, `mutableStateOf` y los primeros componentes.