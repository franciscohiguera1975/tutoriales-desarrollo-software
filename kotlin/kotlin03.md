
# Curso Kotlin — Página 3
## Módulo 1 · Fundamentos
### Funciones: declaración, parámetros por defecto, extensiones e `infix`

---

## Declaración de funciones

En Kotlin las funciones se declaran con la palabra clave `fun`.
Pueden vivir en el nivel superior del archivo — sin necesidad de una clase contenedora.

```kotlin
// Forma completa — tipo de retorno explícito
fun sumar(a: Int, b: Int): Int {
    return a + b
}

// Forma expresión — para funciones de una sola expresión
// El tipo de retorno se infiere, los {} y return desaparecen
fun sumarExpr(a: Int, b: Int): Int = a + b

// Aún más concisa — tipo inferido
fun sumarInferida(a: Int, b: Int) = a + b

// Función que no devuelve valor — Unit es equivalente a void
fun saludar(nombre: String): Unit {
    println("Hola, $nombre")
}

// Unit se puede omitir
fun saludar2(nombre: String) {
    println("Hola, $nombre")
}

fun main() {
    println(sumar(3, 4))         // 7
    println(sumarExpr(3, 4))     // 7
    println(sumarInferida(3, 4)) // 7
    saludar("Ana")               // Hola, Ana
}
```

> Usa la forma expresión (`= expresión`) cuando la función tiene una sola
> instrucción. Usa la forma completa con `{}` cuando el cuerpo tiene lógica.

---

## Parámetros por defecto

Kotlin permite asignar valores por defecto a los parámetros.
Esto elimina la necesidad de sobrecargar funciones:

```kotlin
// Java necesitaría 3 sobrecargas para esto
// Kotlin lo resuelve con valores por defecto
fun crearUsuario(
    nombre:   String,
    edad:     Int    = 18,
    rol:      String = "viewer",
    activo:   Boolean = true
): String {
    return "Usuario[$nombre, edad=$edad, rol=$rol, activo=$activo]"
}

fun main() {
    // Todos los argumentos
    println(crearUsuario("Ana", 25, "admin", true))

    // Omitiendo los que tienen valor por defecto
    println(crearUsuario("Luis"))

    // Solo algunos
    println(crearUsuario("María", 30))
    println(crearUsuario("Carlos", rol = "editor"))
}
```

Salida:
```
Usuario[Ana, edad=25, rol=admin, activo=true]
Usuario[Luis, edad=18, rol=viewer, activo=true]
Usuario[María, edad=30, rol=viewer, activo=true]
Usuario[Carlos, edad=18, rol=editor, activo=true]
```

---

## Named arguments — argumentos nombrados

Los argumentos nombrados permiten pasar valores en cualquier orden
especificando el nombre del parámetro:

```kotlin
fun configurar(
    host:    String  = "localhost",
    puerto:  Int     = 8080,
    https:   Boolean = false,
    timeout: Int     = 30
) {
    val protocolo = if (https) "https" else "http"
    println("$protocolo://$host:$puerto (timeout: ${timeout}s)")
}

fun main() {
    // Sin nombres — el orden importa
    configurar("api.ejemplo.com", 443, true, 60)

    // Con nombres — el orden no importa, el código es más legible
    configurar(
        host    = "api.ejemplo.com",
        https   = true,
        puerto  = 443,
        timeout = 60
    )

    // Mezcla — los posicionales deben ir antes de los nombrados
    configurar("api.ejemplo.com", https = true)

    // Solo los que difieren del valor por defecto
    configurar(https = true, puerto = 443)
}
```

> Los named arguments son especialmente útiles cuando una función tiene
> varios parámetros del mismo tipo y el orden podría confundirse.

---

## Funciones de extensión

Las funciones de extensión añaden nuevas funciones a un tipo existente
**sin modificar su código fuente** y sin herencia.

```kotlin
// Extiende String con una función isPalindrome
fun String.isPalindrome(): Boolean {
    val limpio = this.lowercase().filter { it.isLetter() }
    return limpio == limpio.reversed()
}

// Extiende Int con una función para verificar si es par
fun Int.isEven(): Boolean = this % 2 == 0

// Extiende Double con formato de moneda
fun Double.toCurrency(symbol: String = "$"): String {
    return "$symbol%.2f".format(this)
}

// Extiende List<Int>
fun List<Int>.secondOrNull(): Int? = if (this.size >= 2) this[1] else null

fun main() {
    println("racecar".isPalindrome())     // true
    println("Anita lava la tina".isPalindrome()) // true
    println("Kotlin".isPalindrome())      // false

    println(4.isEven())                   // true
    println(7.isEven())                   // false

    println(1234.56.toCurrency())         // $1234.56
    println(99.9.toCurrency("€"))         // €99.90

    val nums = listOf(10, 20, 30)
    println(nums.secondOrNull())          // 20
    println(emptyList<Int>().secondOrNull()) // null
}
```

### Dónde poner las funciones de extensión

Las funciones de extensión se colocan típicamente en archivos dedicados
o junto al código que las usa:

```
src/main/kotlin/
├── extensions/
│   ├── StringExtensions.kt
│   ├── IntExtensions.kt
│   └── CollectionExtensions.kt
└── Main.kt
```

---

## Funciones locales

Kotlin permite declarar funciones dentro de otras funciones.
Son útiles para encapsular lógica auxiliar que no necesita ser visible fuera:

```kotlin
fun procesarPedido(items: List<String>, descuento: Double): Double {
    // Función local — solo visible dentro de procesarPedido
    fun calcularImpuesto(precio: Double): Double = precio * 0.19

    // También puede acceder a las variables del scope exterior
    fun aplicarDescuento(precio: Double): Double = precio * (1 - descuento)

    val precioBase = items.size * 10.0
    val conDescuento = aplicarDescuento(precioBase)
    val conImpuesto  = conDescuento + calcularImpuesto(conDescuento)

    return conImpuesto
}

fun main() {
    val total = procesarPedido(listOf("libro", "cuaderno", "lápiz"), 0.10)
    println("Total: $${"%.2f".format(total)}")  // Total: $32.13
}
```

---

## Funciones `infix`

Una función `infix` se puede llamar sin el punto y los paréntesis,
lo que permite crear DSLs y expresiones más legibles:

```kotlin
// infix requiere: ser función de extensión o método de clase
// y tener exactamente un parámetro
infix fun Int.elevadoA(exponente: Int): Long {
    var resultado = 1L
    repeat(exponente) { resultado *= this }
    return resultado
}

infix fun String.contiene(subcadena: String): Boolean =
    this.contains(subcadena, ignoreCase = true)

// Pair se construye con infix 'to' — ya viene en la stdlib
// "clave" to "valor"  →  Pair("clave", "valor")

fun main() {
    println(2 elevadoA 10)          // 1024
    println(3 elevadoA 5)           // 243

    println("Kotlin es genial" contiene "Kotlin")  // true
    println("Kotlin es genial" contiene "Java")    // false

    // infix 'to' de la stdlib para crear Pair y Map
    val par      = "nombre" to "Ana"
    val mapa     = mapOf("a" to 1, "b" to 2, "c" to 3)
    println(par)   // (nombre, Ana)
    println(mapa)  // {a=1, b=2, c=3}
}
```

---

## Funciones `tailrec`

El modificador `tailrec` optimiza funciones recursivas en las que la
llamada recursiva es la **última operación**. El compilador las convierte
en un bucle, evitando el desbordamiento de pila (`StackOverflowError`):

```kotlin
// Sin tailrec — riesgo de StackOverflow con n grande
fun factorialNormal(n: Int): Long {
    return if (n <= 1) 1L else n * factorialNormal(n - 1)
    //                              ↑ la multiplicación ocurre DESPUÉS — no es tailrec
}

// Con tailrec — el compilador lo convierte en un bucle
tailrec fun factorialTailRec(n: Int, acumulador: Long = 1L): Long {
    return if (n <= 1) acumulador
    else factorialTailRec(n - 1, n * acumulador)
    //   ↑ la llamada recursiva es la ÚLTIMA operación — sí es tailrec
}

tailrec fun fibonacci(n: Int, a: Long = 0L, b: Long = 1L): Long =
    if (n == 0) a else fibonacci(n - 1, b, a + b)

fun main() {
    println(factorialTailRec(10))   // 3628800
    println(factorialTailRec(20))   // 2432902008176640000
    println(fibonacci(50))          // 12586269025
    println(fibonacci(70))          // 190392490709135
}
```

---

## `Nothing` — funciones que nunca retornan

`Nothing` es el tipo de retorno de funciones que siempre lanzan una excepción
o entran en un bucle infinito. El compilador lo usa para análisis de flujo:

```kotlin
fun error(mensaje: String): Nothing {
    throw IllegalStateException(mensaje)
}

fun main() {
    val entrada = readLine()

    // El compilador sabe que si entrada == null, error() nunca retorna
    // Por eso no hace falta el ?.let o el ?: en las líneas siguientes
    val texto: String = entrada ?: error("La entrada no puede ser null")
    println(texto.uppercase())  // texto es String, no String?
}
```

---

## Programa completo — utilidades de texto

```kotlin
// Extensiones para String
fun String.palabras(): List<String> =
    this.trim().split(Regex("\\s+")).filter { it.isNotEmpty() }

fun String.contarVocales(): Int =
    this.count { it.lowercase() in "aeiouáéíóúü" }

fun String.capitalizar(): String =
    this.palabras().joinToString(" ") { palabra ->
        palabra.lowercase().replaceFirstChar { it.uppercase() }
    }

fun String.esPalindromo(): Boolean {
    val solo = this.lowercase().filter { it.isLetter() }
    return solo == solo.reversed()
}

// Extensión con parámetro por defecto
fun String.truncar(maxLongitud: Int = 50, sufijo: String = "..."): String =
    if (this.length <= maxLongitud) this
    else this.take(maxLongitud - sufijo.length) + sufijo

fun main() {
    val frase = "  el rápido zorro marrón salta sobre el perro perezoso  "

    println(frase.palabras())           // [el, rápido, zorro, ...]
    println(frase.palabras().size)      // 9
    println(frase.contarVocales())      // 20
    println(frase.capitalizar())        // El Rápido Zorro Marrón...
    println(frase.truncar(30))          // el rápido zorro marrón...

    val candidatos = listOf("racecar", "Kotlin", "Anita lava la tina", "Java")
    for (c in candidatos) {
        println("\"$c\" → palindromo: ${c.esPalindromo()}")
    }
}
```

---

## Programa completo — calculadora con funciones nombradas

```kotlin
fun sumar(a: Double, b: Double)      = a + b
fun restar(a: Double, b: Double)     = a - b
fun multiplicar(a: Double, b: Double) = a * b
fun dividir(a: Double, b: Double): Double {
    require(b != 0.0) { "No se puede dividir entre cero" }
    return a / b
}
fun potencia(base: Double, exp: Int): Double {
    require(exp >= 0) { "El exponente debe ser no negativo" }
    return (1..exp).fold(1.0) { acc, _ -> acc * base }
}

fun calcular(a: Double, op: String, b: Double): Double = when (op) {
    "+"  -> sumar(a, b)
    "-"  -> restar(a, b)
    "*"  -> multiplicar(a, b)
    "/"  -> dividir(a, b)
    "**" -> potencia(a, b.toInt())
    else -> throw IllegalArgumentException("Operador desconocido: '$op'")
}

fun main() {
    val operaciones = listOf(
        Triple(10.0, "+",  5.0),
        Triple(10.0, "-",  3.0),
        Triple(6.0,  "*",  7.0),
        Triple(15.0, "/",  4.0),
        Triple(2.0,  "**", 10.0),
    )

    for ((a, op, b) in operaciones) {
        val resultado = calcular(a, op, b)
        println("$a $op $b = $resultado")
    }
}
```

Salida:
```
10.0 + 5.0 = 15.0
10.0 - 3.0 = 7.0
6.0 * 7.0 = 42.0
15.0 / 4.0 = 3.75
2.0 ** 10.0 = 1024.0
```

---

## Ejercicios propuestos

1. **Extensión `Int.factorial()`** — implementa `fun Int.factorial(): Long` como
   función de extensión usando `tailrec`. Úsala para calcular `10.factorial()`.

2. **Función de validación** — escribe `fun validarContrasena(pass: String): List<String>`
   que devuelva una lista de errores (vacía si es válida). Reglas: mínimo 8 caracteres,
   al menos una mayúscula, al menos un número, al menos un símbolo.
   Usa funciones locales para cada regla.

3. **Infix `entre`** — crea la función `infix fun Int.entre(rango: IntRange): Boolean`
   que permita escribir `5 entre 1..10`. Usa named arguments y valores por defecto
   en una función que formatee el resultado.

4. **DSL de texto** — crea una función `fun construirParrafo(vararg lineas: String): String`
   que una las líneas con saltos de línea y aplique `capitalizar()` a cada una
   usando las extensiones de String del ejemplo anterior.

---

## Resumen de la página 3

- Las funciones en Kotlin se declaran con `fun` y pueden vivir en el nivel superior del archivo — sin clase contenedora.
- La forma expresión `fun f() = expresión` es la forma idiomática para funciones de una sola expresión.
- Los parámetros por defecto eliminan la necesidad de sobrecargar funciones — úsalos en lugar de múltiples versiones de la misma función.
- Los named arguments `f(param = valor)` mejoran la legibilidad cuando hay varios parámetros del mismo tipo o muchos opcionales.
- Las funciones de extensión añaden comportamiento a tipos existentes sin modificarlos ni heredar de ellos — `fun Tipo.nuevaFuncion()`.
- Las funciones locales encapsulan lógica auxiliar dentro del scope donde se necesita.
- `infix` permite llamadas sin punto ni paréntesis — útil para DSLs y operaciones que se leen como lenguaje natural.
- `tailrec` convierte recursión de cola en un bucle — necesario para recursión profunda sin riesgo de `StackOverflowError`.
- `Nothing` es el tipo de retorno de funciones que nunca completan normalmente — el compilador lo usa en análisis de flujo.

---

> **Siguiente página →** Página 4: Null safety — tipos nullable, operadores
> `?.`, `?:`, `!!`, scope functions y smart cast.