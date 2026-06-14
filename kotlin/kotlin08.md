# Curso Kotlin — Página 8
## Módulo 3 · Funciones de orden superior y genéricos
### Lambdas, tipos funcionales, genéricos y `typealias`

---

## Funciones de orden superior

Una función de **orden superior** es aquella que recibe funciones como
parámetros o devuelve una función como resultado. En Kotlin las funciones
son **ciudadanos de primera clase** — se pueden almacenar en variables,
pasar como argumentos y retornar desde otras funciones:

```kotlin
// Función que recibe otra función como parámetro
fun operar(a: Int, b: Int, operacion: (Int, Int) -> Int): Int {
    return operacion(a, b)
}

fun main() {
    // Pasar una función como argumento
    val suma    = operar(10, 5) { x, y -> x + y }
    val resta   = operar(10, 5) { x, y -> x - y }
    val producto = operar(10, 5) { x, y -> x * y }

    println(suma)     // 15
    println(resta)    // 5
    println(producto) // 50

    // Almacenar una función en una variable
    val duplicar: (Int) -> Int = { n -> n * 2 }
    val saludar:  (String) -> String = { nombre -> "Hola, $nombre!" }

    println(duplicar(7))        // 14
    println(saludar("Ana"))     // Hola, Ana!

    // Pasar la referencia de una función con ::
    fun esPar(n: Int) = n % 2 == 0
    val numeros = listOf(1, 2, 3, 4, 5, 6)
    println(numeros.filter(::esPar))  // [2, 4, 6]
    println(numeros.map(::duplicar))  // [2, 4, 6, 8, 10, 12]
}
```

---

## Tipos funcionales

Los tipos funcionales describen la firma de una función:
`(TipoParam1, TipoParam2) -> TipoRetorno`

```kotlin
// Tipos funcionales comunes
val accion:      ()           -> Unit    = { println("Sin parámetros") }
val transformar: (Int)        -> String  = { n -> "Número $n" }
val combinar:    (Int, Int)   -> Int     = { a, b -> a + b }
val predicado:   (String)     -> Boolean = { it.length > 3 }

// Nullable — la función misma puede ser null
var callback: ((String) -> Unit)? = null
callback = { msg -> println("Callback: $msg") }
callback?.invoke("mensaje")   // llamada segura con ?.invoke()

// Función que devuelve una función
fun multiplicadorDe(factor: Int): (Int) -> Int {
    return { n -> n * factor }
}

fun main() {
    val doble  = multiplicadorDe(2)
    val triple = multiplicadorDe(3)

    println(doble(5))   // 10
    println(triple(5))  // 15

    val numeros = listOf(1, 2, 3, 4, 5)
    println(numeros.map(doble))   // [2, 4, 6, 8, 10]
    println(numeros.map(triple))  // [3, 6, 9, 12, 15]
}
```

---

## Lambdas en detalle

Una lambda es una función anónima definida en el lugar donde se usa:

```kotlin
fun main() {
    // Sintaxis completa
    val suma1: (Int, Int) -> Int = { a: Int, b: Int -> a + b }

    // Con tipo inferido del contexto — los tipos en la lambda son opcionales
    val suma2: (Int, Int) -> Int = { a, b -> a + b }

    // Parámetro implícito 'it' — solo cuando hay un parámetro
    val duplicar: (Int) -> Int = { it * 2 }
    val mayuscula: (String) -> String = { it.uppercase() }

    // Ignorar parámetros con _
    val siempre42: (String, Int) -> Int = { _, _ -> 42 }

    // Lambda multilínea — el valor de la última expresión es el retorno
    val procesar: (Int) -> String = { n ->
        val cuadrado = n * n
        val esPar = if (n % 2 == 0) "par" else "impar"
        "El número $n es $esPar y su cuadrado es $cuadrado"
    }

    println(procesar(4))   // El número 4 es par y su cuadrado es 16
    println(procesar(7))   // El número 7 es impar y su cuadrado es 49

    // Trailing lambda — cuando el último parámetro es una función,
    // la lambda puede ir FUERA de los paréntesis
    listOf(1, 2, 3).forEach({ n -> println(n) })  // sin trailing lambda
    listOf(1, 2, 3).forEach { n -> println(n) }   // con trailing lambda (idiomático)

    // Si es el único parámetro, los paréntesis desaparecen completamente
    listOf(1, 2, 3).forEach { println(it) }
}
```

---

## Closures — lambdas que capturan variables

Las lambdas pueden **capturar** variables del scope donde fueron definidas,
incluyendo variables mutables:

```kotlin
fun crearContador(inicio: Int = 0): () -> Int {
    var cuenta = inicio   // capturada por la lambda

    return {
        cuenta++   // modifica la variable capturada
        cuenta
    }
}

fun main() {
    val contador1 = crearContador()
    val contador2 = crearContador(10)

    println(contador1())  // 1
    println(contador1())  // 2
    println(contador1())  // 3
    println(contador2())  // 11
    println(contador2())  // 12
    println(contador1())  // 4 — sigue desde donde quedó

    // Aplicación práctica — generador de IDs únicos
    fun crearGeneradorId(prefijo: String): () -> String {
        var secuencia = 0
        return { "$prefijo-${++secuencia}" }
    }

    val generarUsuarioId  = crearGeneradorId("USR")
    val generarProductoId = crearGeneradorId("PRD")

    println(generarUsuarioId())   // USR-1
    println(generarUsuarioId())   // USR-2
    println(generarProductoId())  // PRD-1
    println(generarUsuarioId())   // USR-3
}
```

---

## Funciones inline

El modificador `inline` le indica al compilador que copie el cuerpo de la
función (y de sus lambdas) en el lugar de la llamada, eliminando la
sobrecarga de creación de objetos lambda:

```kotlin
// Sin inline — crea un objeto lambda en cada llamada
fun <T> medir(bloque: () -> T): T {
    val inicio = System.currentTimeMillis()
    val resultado = bloque()
    val fin = System.currentTimeMillis()
    println("Tiempo: ${fin - inicio}ms")
    return resultado
}

// Con inline — el compilador copia el cuerpo inline en el punto de llamada
// No crea objeto lambda — mejor rendimiento
inline fun <T> medirInline(bloque: () -> T): T {
    val inicio = System.currentTimeMillis()
    val resultado = bloque()
    val fin = System.currentTimeMillis()
    println("Tiempo: ${fin - inicio}ms")
    return resultado
}

fun main() {
    val resultado = medirInline {
        var suma = 0L
        repeat(1_000_000) { suma += it }
        suma
    }
    println("Suma: $resultado")
}
```

> Usa `inline` en funciones de orden superior que se llaman con frecuencia
> y cuyas lambdas son simples. El compilador de Kotlin usa `inline` internamente
> en `filter`, `map`, `forEach` y otras funciones de la stdlib.

---

## Genéricos

Los genéricos permiten escribir código que funciona con **cualquier tipo**
manteniendo la seguridad de tipos en tiempo de compilación:

### Funciones genéricas

```kotlin
// T es un tipo genérico — se determina al llamar la función
fun <T> primerElemento(lista: List<T>): T? = lista.firstOrNull()

fun <T> intercambiar(par: Pair<T, T>): Pair<T, T> = Pair(par.second, par.first)

fun <T : Comparable<T>> maximo(a: T, b: T): T = if (a > b) a else b

fun main() {
    println(primerElemento(listOf(1, 2, 3)))      // 1
    println(primerElemento(listOf("a", "b")))     // a
    println(primerElemento(emptyList<Int>()))      // null

    println(intercambiar(Pair(1, 2)))             // (2, 1)
    println(intercambiar(Pair("a", "b")))         // (b, a)

    // T : Comparable<T> — restricción: T debe ser comparable
    println(maximo(10, 20))      // 20
    println(maximo("Ana", "Luis"))  // Luis (orden lexicográfico)
    println(maximo(3.14, 2.71))  // 3.14
}
```

### Clases genéricas

```kotlin
// Pila genérica — funciona con cualquier tipo T
class Pila<T> {
    private val elementos = ArrayDeque<T>()

    fun apilar(elemento: T) {
        elementos.addLast(elemento)
    }

    fun desapilar(): T? = elementos.removeLastOrNull()

    fun peek(): T? = elementos.lastOrNull()

    fun estaVacia() = elementos.isEmpty()

    val tamaño: Int get() = elementos.size

    override fun toString() = elementos.toString()
}

fun main() {
    val pilaInt = Pila<Int>()
    pilaInt.apilar(1)
    pilaInt.apilar(2)
    pilaInt.apilar(3)
    println(pilaInt)            // [1, 2, 3]
    println(pilaInt.desapilar()) // 3
    println(pilaInt.peek())      // 2

    val pilaStr = Pila<String>()
    pilaStr.apilar("primero")
    pilaStr.apilar("segundo")
    println(pilaStr.desapilar()) // segundo
}
```

### Restricciones de tipo (`where` y upper bounds)

```kotlin
// Upper bound — T debe implementar Comparable
fun <T : Comparable<T>> ordenar(lista: List<T>): List<T> = lista.sorted()

// Múltiples restricciones con where
interface Imprimible { fun imprimir() }
interface Guardable  { fun guardar() }

fun <T> procesarYGuardar(item: T) where T : Imprimible, T : Guardable {
    item.imprimir()
    item.guardar()
}

// Reified — permite inspeccionar el tipo en runtime dentro de funciones inline
inline fun <reified T> esInstanciaDe(valor: Any): Boolean = valor is T

fun main() {
    println(ordenar(listOf(3, 1, 4, 1, 5, 9)))     // [1, 1, 3, 4, 5, 9]
    println(ordenar(listOf("banana", "manzana", "cereza")))  // [banana, cereza, manzana]

    println(esInstanciaDe<String>("hola"))  // true
    println(esInstanciaDe<Int>("hola"))     // false
    println(esInstanciaDe<List<*>>(listOf(1, 2, 3)))  // true
}
```

### Varianza: `out` (covariante) e `in` (contravariante)

```kotlin
// out T — solo produce T (puede leerse), equivale a <? extends T> en Java
class Caja<out T>(val contenido: T) {
    fun obtener(): T = contenido
    // fun guardar(item: T) { ... }  // ERROR — out no puede recibir T
}

// in T — solo consume T (puede escribirse), equivale a <? super T> en Java
class Receptor<in T> {
    fun recibir(item: T) { println("Recibido: $item") }
    // fun obtener(): T { ... }  // ERROR — in no puede devolver T
}

fun main() {
    // out — puedes asignar Caja<String> donde se espera Caja<Any>
    val cajaStr: Caja<String>  = Caja("Hola")
    val cajaAny: Caja<Any>     = cajaStr  // OK gracias a 'out'
    println(cajaAny.obtener())  // Hola

    // in — puedes asignar Receptor<Any> donde se espera Receptor<String>
    val receptorAny: Receptor<Any>    = Receptor()
    val receptorStr: Receptor<String> = receptorAny  // OK gracias a 'in'
    receptorStr.recibir("texto")      // Recibido: texto
}
```

---

## `typealias` — alias de tipos

`typealias` crea un nombre alternativo para un tipo existente,
mejorando la legibilidad sin crear un nuevo tipo:

```kotlin
// Tipos funcionales complejos se vuelven legibles
typealias Predicado<T>   = (T) -> Boolean
typealias Transformacion<T, R> = (T) -> R
typealias Callback       = () -> Unit
typealias Handler        = (String) -> Unit

// Tipos de colecciones con nombre de dominio
typealias Inventario     = Map<String, Int>
typealias ListaProductos = List<String>
typealias MatrizDouble   = List<List<Double>>

// Uso — el código es mucho más legible
fun filtrarLista(items: List<String>, criterio: Predicado<String>): List<String> =
    items.filter(criterio)

fun procesarEventos(eventos: List<String>, manejador: Handler) {
    eventos.forEach(manejador)
}

fun componerTransformaciones(
    primera:  Transformacion<String, Int>,
    segunda:  Transformacion<Int, String>
): Transformacion<String, String> = { s -> segunda(primera(s)) }

fun main() {
    val esPalabra  = filtrarLista(listOf("hola", "", "mundo", ""), { it.isNotEmpty() })
    println(esPalabra)  // [hola, mundo]

    procesarEventos(listOf("click", "scroll", "resize")) { evento ->
        println("Evento: $evento")
    }

    val longitudATexto = componerTransformaciones(
        { s: String -> s.length },
        { n: Int    -> "longitud=$n" }
    )
    println(longitudATexto("Kotlin"))  // longitud=6
}
```

---

## Programa completo — motor de reglas de negocio

```kotlin
// typealias para claridad
typealias Regla<T>     = (T) -> Boolean
typealias Transformador<T> = (T) -> T

data class Pedido(
    val id:      Int,
    val cliente: String,
    val total:   Double,
    val items:   List<String>,
    val pagado:  Boolean = false
)

// Motor genérico de validación
class ValidadorPedidos(private val reglas: List<Regla<Pedido>>) {
    fun validar(pedido: Pedido): List<String> = buildList {
        if (pedido.cliente.isBlank())  add("Cliente vacío")
        if (pedido.total <= 0)         add("Total inválido")
        if (pedido.items.isEmpty())    add("Sin artículos")
        // aplicar reglas personalizadas
        reglas.forEachIndexed { i, regla ->
            if (!regla(pedido)) add("Regla $i incumplida")
        }
    }
    fun esValido(pedido: Pedido) = validar(pedido).isEmpty()
}

// Pipeline de transformaciones con funciones de orden superior
fun <T> pipeline(vararg transformadores: Transformador<T>): Transformador<T> {
    return { entrada ->
        transformadores.fold(entrada) { acc, transformador -> transformador(acc) }
    }
}

fun main() {
    val validador = ValidadorPedidos(listOf(
        { pedido -> pedido.total <= 10_000 },           // límite de compra
        { pedido -> pedido.items.size <= 20 },          // máximo 20 artículos
        { pedido -> !pedido.cliente.contains("test") }  // no cuentas de prueba
    ))

    // Pipeline de procesamiento de un pedido
    val procesarPedido = pipeline<Pedido>(
        { p -> p.copy(cliente = p.cliente.trim().lowercase()) },
        { p -> p.copy(total   = Math.round(p.total * 100) / 100.0) },
        { p -> if (p.total > 0) p.copy(pagado = true) else p }
    )

    val pedidos = listOf(
        Pedido(1, "  Ana García  ", 149.999, listOf("Teclado", "Mouse")),
        Pedido(2, "test-user",     50.0,    listOf("Webcam")),
        Pedido(3, "Luis",          15_000.0, listOf("Servidor")),
        Pedido(4, "María",         0.0,     listOf("Regalo"))
    )

    pedidos.forEach { pedidoOriginal ->
        val pedido = procesarPedido(pedidoOriginal)
        val errores = validador.validar(pedido)
        if (errores.isEmpty()) {
            println("✅ Pedido ${pedido.id} de ${pedido.cliente}: $${pedido.total} — pagado: ${pedido.pagado}")
        } else {
            println("❌ Pedido ${pedido.id} rechazado: ${errores.joinToString(", ")}")
        }
    }
}
```

Salida:
```
✅ Pedido 1 de ana garcía: $150.0 — pagado: true
❌ Pedido 2 rechazado: Regla 2 incumplida
❌ Pedido 3 rechazado: Regla 0 incumplida
❌ Pedido 4 rechazado: Total inválido, Regla 2 incumplida? no — Total inválido
```

---

## Ejercicios propuestos

1. **Composición de funciones** — Implementa `fun <A, B, C> componer(f: (A) -> B, g: (B) -> C): (A) -> C`
   que devuelve la composición matemática `g(f(x))`. Crea un pipeline de
   transformaciones de texto: `limpiar`, `capitalizar`, `truncar(20)` y
   compón las tres en una sola función.

2. **Pila genérica con historial** — Extiende `Pila<T>` para que registre
   todas las operaciones en un `historial: List<String>`. Cada `apilar` y
   `desapilar` añade una entrada. Agrega `deshacerUltimo()` que revierta
   la última operación.

3. **Motor de filtros** — Implementa `class FiltroCompuesto<T>(vararg filtros: (T) -> Boolean)`
   con operadores `AND` (`todos deben cumplirse`) y `OR` (`al menos uno`).
   Aplícalo a una lista de `Producto` con filtros de precio, categoría y stock.

4. **`reified` + genéricos** — Implementa `inline fun <reified T> List<Any>.obtenerPrimero(): T?`
   que devuelva el primer elemento de tipo `T` de una lista heterogénea.
   Pruébalo con `listOf(1, "hola", 2.5, "mundo", true)`.

---

## Resumen de la página 8

- Una función de orden superior recibe o devuelve funciones. En Kotlin las funciones son ciudadanos de primera clase — se almacenan en variables y se pasan como argumentos.
- El tipo funcional `(Param) -> Retorno` describe la firma. `() -> Unit` es una acción sin parámetros ni retorno.
- `it` es el parámetro implícito de las lambdas de un solo parámetro. Se puede nombrar explícitamente con `{ nombre -> ... }`.
- El **trailing lambda** — cuando el último argumento es una función, la lambda va fuera de los paréntesis. Si es el único argumento, los paréntesis desaparecen.
- Las **closures** capturan variables del scope exterior, incluyendo `var` mutables.
- `inline` elimina la sobrecarga de creación de objetos lambda copiando el cuerpo en el punto de llamada.
- Los **genéricos** `<T>` permiten código reutilizable con seguridad de tipos. `T : Comparable<T>` restringe el tipo.
- `out T` (covarianza) — solo produce `T`. `in T` (contravarianza) — solo consume `T`.
- `reified` permite inspeccionar el tipo genérico en runtime dentro de funciones `inline`.
- `typealias` nombra tipos complejos para mejorar la legibilidad sin crear un tipo nuevo.

---

> **Siguiente página →** Página 9: Coroutines — `suspend`, `launch`,
> `async`, `Flow` y manejo de errores asíncronos.