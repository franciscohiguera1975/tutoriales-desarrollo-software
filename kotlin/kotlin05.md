# Curso Kotlin — Página 5
## Módulo 2 · Programación Orientada a Objetos
### Los 4 principios OOP y cómo Kotlin los implementa

---

## Los 4 principios de la OOP

La **Programación Orientada a Objetos** se basa en cuatro principios
fundamentales. Antes de escribir código, es importante entender qué
problema resuelve cada uno:

| Principio | Pregunta que responde | Mecanismo en Kotlin |
|---|---|---|
| **Abstracción** | ¿Qué puede hacer este objeto? | `abstract class`, `interface` |
| **Encapsulamiento** | ¿Quién puede acceder a sus datos? | `private`, `val`, getters/setters |
| **Herencia** | ¿Puede reutilizar comportamiento de otro? | `open class`, `: ClasePadre()` |
| **Polimorfismo** | ¿Puede comportarse diferente según el contexto? | `override`, `interface`, `sealed class` |

Esta página cubre **Abstracción** y **Encapsulamiento** — los dos
principios que se aplican al diseñar una clase bien construida.
La página 6 cubre **Herencia** y **Polimorfismo**.

---

## Principio 1 — Abstracción

> **"Muestra solo lo que el usuario necesita. Oculta cómo funciona por dentro."**

La abstracción separa el **qué** (la interfaz pública) del **cómo**
(la implementación interna). El código que usa la clase no necesita
saber cómo está implementada — solo qué puede pedirle.

En Kotlin, una clase simple ya aplica abstracción al exponer solo
lo que tiene sentido exponer:

```kotlin
// El usuario de esta clase solo sabe QUÉ puede hacer con un Producto
// No necesita saber cómo se calcula precioConIva ni cómo funciona disponible
class Producto(
    val id:       Int,
    val nombre:   String,
    val precio:   Double,
    private val stock: Int      // privado — el usuario no manipula el stock directamente
) {
    val precioConIva: Double    // interfaz pública — qué puede consultar
        get() = precio * 1.19

    val disponible: Boolean
        get() = stock > 0

    override fun toString() = "$nombre ($${"%.2f".format(precio)})"
}

fun main() {
    val teclado = Producto(1, "Teclado mecánico", 89.99, 15)

    // El código externo usa la interfaz pública — no sabe el detalle interno
    println(teclado.disponible)   // true
    println(teclado.precioConIva) // 106.99
    // teclado.stock = 0           // ERROR — privado, protegido por diseño
}
```

---

## Principio 2 — Encapsulamiento

> **"Los datos de una clase solo se modifican a través de sus propios métodos."**

El encapsulamiento protege el estado interno de una clase.
En Kotlin se logra con modificadores de visibilidad y propiedades
con getter/setter personalizados:

### Modificadores de visibilidad

```kotlin
class CuentaBancaria(titular: String, saldoInicial: Double) {

    val titular: String = titular       // público — cualquiera puede leer

    private var saldo: Double = saldoInicial  // privado — solo esta clase lo modifica

    internal val numeroCuenta: String =        // internal — visible en el mismo módulo
        "ES${(100000..999999).random()}"

    protected open fun calcularInteres(): Double = saldo * 0.02  // protected — visible en subclases

    // El saldo solo cambia a través de estos métodos — NUNCA directamente
    fun depositar(monto: Double) {
        require(monto > 0) { "El monto debe ser positivo" }
        saldo += monto
        println("Depositado: $${"%.2f".format(monto)} | Nuevo saldo: ${consultarSaldo()}")
    }

    fun retirar(monto: Double): Boolean {
        require(monto > 0) { "El monto debe ser positivo" }
        if (monto > saldo) {
            println("Fondos insuficientes")
            return false
        }
        saldo -= monto
        println("Retirado: $${"%.2f".format(monto)} | Nuevo saldo: ${consultarSaldo()}")
        return true
    }

    fun consultarSaldo(): String = "$${"%.2f".format(saldo)}"
}

fun main() {
    val cuenta = CuentaBancaria("Ana García", 1000.0)

    cuenta.depositar(500.0)    // Depositado: $500.00 | Nuevo saldo: $1500.00
    cuenta.retirar(200.0)      // Retirado: $200.00 | Nuevo saldo: $1300.00
    cuenta.retirar(2000.0)     // Fondos insuficientes

    println(cuenta.titular)         // Ana García — acceso público permitido
    println(cuenta.consultarSaldo()) // $1300.00
    // cuenta.saldo = 999999.0       // ERROR — saldo es privado
}
```

### Resumen de visibilidades

| Modificador | Visible desde |
|---|---|
| `public` (defecto) | En cualquier lugar |
| `private` | Solo dentro de la clase |
| `protected` | Dentro de la clase y sus subclases |
| `internal` | Dentro del mismo módulo (proyecto/librería) |

---

## Clases en Kotlin

Con los principios claros, veamos la sintaxis completa.

```kotlin
// El constructor primario integra la declaración de propiedades
class Persona(val nombre: String, val edad: Int)

// Con cuerpo adicional
class Persona2(val nombre: String, val edad: Int) {
    fun presentarse() = "Soy $nombre y tengo $edad años"
    fun esMayorDeEdad() = edad >= 18
}

fun main() {
    val p = Persona("Ana", 28)
    println(p.nombre)   // Ana
    println(p.edad)     // 28

    val p2 = Persona2("Luis", 17)
    println(p2.presentarse())     // Soy Luis y tengo 17 años
    println(p2.esMayorDeEdad())   // false
}
```

### `val` vs `var` en el constructor primario

```kotlin
class PuntoInmutable(val x: Double, val y: Double)   // solo lectura

class Contador(var valor: Int = 0) {                  // lectura y escritura
    fun incrementar() { valor++ }
    fun resetear()    { valor = 0 }
}

// Sin val/var — parámetro del constructor, NO propiedad
// Solo accesible dentro del bloque init
class Temporal(nombre: String) {
    val nombreUpper = nombre.uppercase()
    // nombre no existe fuera de aquí
}
```

---

## Bloque `init`

El bloque `init` es parte del constructor primario.
Es el lugar correcto para validar y calcular propiedades derivadas:

```kotlin
class Usuario(val nombre: String, val email: String) {
    val nombreNormalizado: String
    val dominioEmail: String

    init {
        // Encapsulamiento en acción: validamos antes de construir
        require(nombre.isNotBlank()) { "El nombre no puede estar vacío" }
        require(email.contains("@")) { "Email inválido: $email" }

        nombreNormalizado = nombre.trim().lowercase()
        dominioEmail      = email.substringAfter("@")
    }
}

fun main() {
    val u = Usuario("  Ana García  ", "ana@kotlin.dev")
    println(u.nombreNormalizado)  // ana garcía
    println(u.dominioEmail)       // kotlin.dev

    // Usuario("", "invalido")   // IllegalArgumentException — require falla
}
```

---

## Propiedades con getter y setter personalizados

Las propiedades con getter/setter son **encapsulamiento en su forma
más expresiva** — el código externo accede con sintaxis simple pero
la lógica de validación y cálculo está protegida dentro:

```kotlin
class Temperatura(celsius: Double) {

    // ENCAPSULAMIENTO: el setter valida antes de asignar
    var celsius: Double = celsius
        set(value) {
            require(value >= -273.15) { "Temperatura bajo el cero absoluto" }
            field = value  // 'field' es el backing field
        }

    // ABSTRACCIÓN: el usuario consulta fahrenheit sin saber la fórmula
    val fahrenheit: Double
        get() = celsius * 9.0 / 5.0 + 32.0

    val kelvin: Double
        get() = celsius + 273.15

    val descripcion: String
        get() = when {
            celsius < 0  -> "Bajo cero"
            celsius < 15 -> "Frío"
            celsius < 25 -> "Templado"
            celsius < 35 -> "Caluroso"
            else         -> "Muy caluroso"
        }
}

fun main() {
    val temp = Temperatura(20.0)
    println("${temp.celsius}°C = ${temp.fahrenheit}°F = ${temp.kelvin}K")
    println(temp.descripcion)  // Templado

    temp.celsius = -5.0
    println("${temp.celsius}°C → ${temp.descripcion}")  // Bajo cero

    // temp.celsius = -300.0  // IllegalArgumentException
}
```

---

## Constructores secundarios

En la mayoría de casos los parámetros por defecto son suficientes.
Los constructores secundarios se usan cuando necesitas lógica de
construcción más compleja:

```kotlin
class Rectangulo(val ancho: Double, val alto: Double) {
    val area:      Double get() = ancho * alto
    val perimetro: Double get() = 2 * (ancho + alto)

    // Siempre llaman al constructor primario con this(...)
    constructor(lado: Double) : this(lado, lado)
    constructor(ancho: Int, alto: Int) : this(ancho.toDouble(), alto.toDouble())

    override fun toString() = "Rectángulo(${ancho}x${alto}) | área=${area}"
}

fun main() {
    val r1 = Rectangulo(5.0, 3.0)
    val r2 = Rectangulo(4.0)        // cuadrado
    val r3 = Rectangulo(6, 2)       // con Int

    println(r1)  // Rectángulo(5.0x3.0) | área=15.0
    println(r2)  // Rectángulo(4.0x4.0) | área=16.0
}
```

---

## `data class` — abstracción para modelos de datos

Las `data class` aplican **abstracción** — representan datos de forma
limpia sin que el código externo tenga que gestionar `equals`,
`hashCode` o `toString` manualmente:

```kotlin
data class Producto(
    val id:        Int,
    val nombre:    String,
    val precio:    Double,
    val categoria: String,
    val activo:    Boolean = true
)

fun main() {
    val p1 = Producto(1, "Teclado mecánico", 89.99, "Periféricos")
    val p2 = Producto(1, "Teclado mecánico", 89.99, "Periféricos")
    val p3 = Producto(2, "Monitor 27\"",     349.99, "Pantallas")

    // toString() automático
    println(p1)  // Producto(id=1, nombre=Teclado mecánico, ...)

    // equals() por valor
    println(p1 == p2)   // true
    println(p1 == p3)   // false

    // copy() — nuevo objeto con cambios puntuales
    val barato   = p1.copy(precio = 59.99)
    val inactivo = p1.copy(activo = false)

    // Desestructuración
    val (id, nombre, precio) = p1
    println("$id: $nombre — $$precio")

    // En bucles
    listOf(p1, p3).forEach { (id2, nombre2, precio2) ->
        println("[$id2] $nombre2: $$precio2")
    }
}
```

---

## `object` — Singleton con encapsulamiento

Un `object` es un **singleton encapsulado**: estado y comportamiento
en una única instancia protegida:

```kotlin
object Configuracion {
    val host:    String = "api.ejemplo.com"
    val puerto:  Int    = 443
    private val apiKey: String = "sk-secreto-123"   // privado — nunca expuesto

    fun baseUrl() = "https://$host:$puerto"
    fun headers() = mapOf("Authorization" to "Bearer $apiKey")
}

class Usuario private constructor(val id: Int, val nombre: String) {
    companion object {
        private var contadorId = 0

        // Factory function — encapsulamiento del constructor
        fun crear(nombre: String, email: String): Usuario? {
            if (nombre.isBlank() || !email.contains("@")) return null
            return Usuario(++contadorId, nombre.trim())
        }

        const val ROL_DEFECTO = "viewer"
    }
}

fun main() {
    println(Configuracion.baseUrl())  // https://api.ejemplo.com:443
    // Configuracion.apiKey            // ERROR — privado

    val u = Usuario.crear("Ana", "ana@test.com")
    println(u)  // Usuario(id=1, nombre=Ana García)
}
```

---

## `enum class` — abstracción de estados

Los `enum class` abstraen conjuntos de estados o categorías,
eliminando el uso de cadenas o números mágicos:

```kotlin
enum class Estado(val descripcion: String, val esTerminal: Boolean) {
    PENDIENTE  ("Esperando procesamiento", false),
    EN_PROCESO ("Siendo procesado",        false),
    COMPLETADO ("Finalizado con éxito",    true),
    FALLIDO    ("Finalizado con error",    true),
    CANCELADO  ("Cancelado por usuario",   true);

    fun puedeTransicionarA(siguiente: Estado): Boolean = when (this) {
        PENDIENTE  -> siguiente == EN_PROCESO || siguiente == CANCELADO
        EN_PROCESO -> siguiente == COMPLETADO || siguiente == FALLIDO
        else       -> false
    }
}

fun main() {
    val estado = Estado.EN_PROCESO
    println(estado.descripcion)  // Siendo procesado
    println(estado.esTerminal)   // false

    // when exhaustivo — sin else porque el compilador conoce todos los casos
    val icono = when (estado) {
        Estado.PENDIENTE   -> "⏰"
        Estado.EN_PROCESO  -> "⏳"
        Estado.COMPLETADO  -> "✅"
        Estado.FALLIDO     -> "❌"
        Estado.CANCELADO   -> "🚫"
    }
    println(icono)  // ⏳

    println(estado.puedeTransicionarA(Estado.COMPLETADO))  // true
}
```

---

## Programa completo — sistema de productos

```kotlin
data class Categoria(val id: Int, val nombre: String)

data class Producto(
    val id:        Int,
    val nombre:    String,
    val precio:    Double,
    val stock:     Int,
    val categoria: Categoria,
    val activo:    Boolean = true
) {
    // ABSTRACCIÓN: el usuario consulta disponible sin saber la lógica
    val disponible: Boolean get() = activo && stock > 0
    val precioConIva: Double get() = precio * 1.19

    // Devuelve una copia — inmutabilidad como forma de encapsulamiento
    fun aplicarDescuento(porcentaje: Double): Producto {
        require(porcentaje in 0.0..100.0) { "Descuento debe ser entre 0 y 100" }
        return copy(precio = precio * (1 - porcentaje / 100))
    }
}

// ENCAPSULAMIENTO: el estado del catálogo es privado y mutable internamente
object CatalogoProductos {
    private val categorias = mutableListOf(
        Categoria(1, "Periféricos"),
        Categoria(2, "Pantallas"),
        Categoria(3, "Audio")
    )
    private val productos   = mutableListOf<Producto>()
    private var siguienteId = 1

    fun agregarProducto(nombre: String, precio: Double, stock: Int, categoriaId: Int): Producto? {
        val categoria = categorias.find { it.id == categoriaId } ?: return null
        val producto  = Producto(siguienteId++, nombre, precio, stock, categoria)
        productos.add(producto)
        return producto
    }

    // ABSTRACCIÓN: interfaz pública limpia — solo lectura de listas
    fun listar(): List<Producto>              = productos.toList()
    fun disponibles(): List<Producto>         = productos.filter { it.disponible }
    fun porCategoria(id: Int): List<Producto> = productos.filter { it.categoria.id == id }
    fun buscar(query: String): List<Producto> =
        productos.filter { it.nombre.contains(query, ignoreCase = true) }
}

fun main() {
    CatalogoProductos.agregarProducto("Teclado mecánico",   89.99, 15, 1)
    CatalogoProductos.agregarProducto("Mouse inalámbrico",  29.99,  0, 1)
    CatalogoProductos.agregarProducto("Monitor 27\"",      349.99,  5, 2)
    CatalogoProductos.agregarProducto("Auriculares BT",    149.99,  8, 3)

    println("=== Todos los productos ===")
    CatalogoProductos.listar().forEach { p ->
        val estado = if (p.disponible) "✅" else "❌"
        println("$estado ${p.nombre} — ${"%.2f".format(p.precioConIva)} (con IVA)")
    }

    println("\n=== Disponibles con 10% descuento ===")
    CatalogoProductos.disponibles()
        .map { it.aplicarDescuento(10.0) }
        .forEach { println("  ${it.nombre}: ${"%.2f".format(it.precio)}") }
}
```

---

## Ejercicios propuestos

1. **Encapsulamiento — `CuentaBancaria`** — Extiende la clase del ejemplo con
   `historial: List<String>` privado. Cada `depositar` y `retirar` agrega
   una entrada al historial. Expón solo `verHistorial(): List<String>` en la
   interfaz pública.

2. **Abstracción — `data class` Punto** — Crea `data class Punto(val x: Double, val y: Double)`.
   Añade funciones de extensión: `distanciaA(otro: Punto)`, `trasladar(dx, dy)`, `escalar(factor)`.
   Desestructura en un `for` sobre una lista de puntos.

3. **`enum class` Semáforo** — Crea `enum class Semaforo { ROJO, AMARILLO, VERDE }`.
   Añade `siguiente(): Semaforo` que cicla entre estados. Simula 9 cambios con `repeat`.

4. **`companion object` Factory** — Crea `class Conexion` con constructor privado,
   propiedades `host`, `puerto` y `activa`. El `companion object` expone
   `crearHttp(host)` y `crearHttps(host)` con puertos por defecto.

---

## Resumen de la página 5

Los 4 principios OOP en Kotlin:

- **Abstracción** — `abstract class`, `interface`, `data class` y propiedades calculadas ocultan la implementación y exponen solo la interfaz necesaria.
- **Encapsulamiento** — `private`/`protected`/`internal` controlan el acceso. Los getters/setters personalizados protegen las invariantes. El constructor privado con factory function es encapsulamiento del proceso de creación.

Conceptos técnicos:
- El constructor primario integra `val`/`var` para declarar propiedades directamente.
- `init` valida con `require()` — lanza `IllegalArgumentException` si la condición falla.
- Las propiedades calculadas con `get()` no tienen backing field — se recalculan cada vez.
- `field` en un setter personalizado referencia el valor almacenado.
- `data class` genera `equals`, `hashCode`, `toString`, `copy` y desestructuración automáticamente.
- `copy()` crea una nueva instancia con cambios puntuales — el original queda intacto.
- `object` es singleton. `companion object` agrupa miembros de clase (equivalente a `static`).

---

> **Siguiente página →** Página 6: Herencia y polimorfismo — `open`, `abstract`,
> interfaces, `sealed class` y delegación.
