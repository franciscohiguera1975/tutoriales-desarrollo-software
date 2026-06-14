
# Curso Kotlin — Página 1
## Módulo 1 · Entorno y primer programa
### Android Studio, estructura del proyecto, REPL y fundamentos de sintaxis

---

## ¿Por qué Kotlin?

Kotlin es un lenguaje **estáticamente tipado** creado por JetBrains y adoptado
por Google como lenguaje oficial de Android en 2017. Compila a bytecode de la JVM,
JavaScript o código nativo, pero su caso de uso más común hoy es Android y backend.

Respecto a Java, el lenguaje que reemplaza progresivamente:

| Java | Kotlin |
|---|---|
| `String nombre = null;` compila y explota en runtime | `val nombre: String?` — el compilador fuerza el manejo del null |
| Clases con 50 líneas de boilerplate | `data class User(val name: String, val age: Int)` — una línea |
| No tiene funciones de extensión | `fun String.isPalindrome(): Boolean` — extensiones de cualquier tipo |
| Lambdas verbosas desde Java 8 | Lambdas concisas desde el primer día |
| `NullPointerException` en producción | Null safety garantizado por el compilador |

---

## Versiones a usar en el curso

| Herramienta | Versión |
|---|---|
| Kotlin | **2.3.0** (estable, enero 2026) |
| Android Studio | Meerkat \| 2024.3.2 o posterior |
| JVM target | 17 |
| Compose BOM | 2026.03.00 (páginas 12–15) |

---

## Configuración del entorno

### 1. Instalar Android Studio

Descarga Android Studio desde `developer.android.com/studio`.
El instalador incluye el JDK 17, el SDK de Android y el plugin de Kotlin.
No necesitas instalar Kotlin por separado.

Al abrir Android Studio por primera vez:
- Acepta el SDK setup wizard
- Elige **Standard** installation
- Deja que descargue los componentes (puede tardar varios minutos)

### 2. Crear un proyecto Kotlin puro (sin Android)

Para los módulos 1–11 trabajamos con Kotlin puro sin UI de Android.
El proyecto más limpio para esto es un **proyecto de consola**:

```
Android Studio → New Project → No Activity
    → Language: Kotlin
    → Minimum SDK: (no importa para módulos de consola)
```

Sin embargo, la forma más directa para practicar Kotlin puro es un
**archivo `.kt` con función `main`** ejecutable directamente desde el IDE.

### 3. Crear el primer archivo Kotlin

```
src/main/kotlin/Main.kt
```

En Android Studio: clic derecho sobre `src/main/kotlin` → New → Kotlin Class/File → File → `Main`

---

## Estructura de un archivo Kotlin

```kotlin
// Main.kt — el archivo puede tener cualquier nombre

// Las funciones de nivel superior van directamente en el archivo
// sin necesidad de clase contenedora (diferencia clave con Java)

fun main() {
    println("Hola, Kotlin 2.3.0")
}
```

Diferencias inmediatas con Java:
- No hay clase `public class Main { ... }` obligatoria
- No hay `public static void main(String[] args)`
- El punto y coma `;` al final de línea es **opcional** — por convención no se usa
- `println` es una función de nivel superior — no `System.out.println`

---

## El REPL de Kotlin

Android Studio incluye el REPL (Read-Eval-Print Loop) de Kotlin — un intérprete
interactivo para probar código sin compilar un proyecto completo.

```
Tools → Kotlin → Kotlin REPL
```

En el REPL puedes escribir cualquier expresión y ver el resultado inmediatamente:

```kotlin
// Escribe esto en el REPL y presiona Ctrl+Enter (⌘+Enter en Mac)
2 + 2
// res0: kotlin.Int = 4

"Hola".uppercase()
// res1: kotlin.String = HOLA

listOf(1, 2, 3).sum()
// res2: kotlin.Int = 6
```

El REPL es ideal para experimentar con la sintaxis antes de escribirla en el proyecto.

---

## Variables: `val` y `var`

Kotlin tiene dos palabras clave para declarar variables:

```kotlin
fun main() {
    // val — inmutable (equivalente a final en Java)
    val nombre = "Ana"        // tipo inferido: String
    val edad: Int = 28        // tipo explícito
    val pi = 3.14159          // tipo inferido: Double

    // var — mutable
    var contador = 0
    contador = contador + 1   // permitido
    contador++                // también permitido

    // nombre = "Luis"        // ERROR de compilación — val no se puede reasignar

    println("$nombre tiene $edad años")  // String template
}
```

**Regla práctica**: usa `val` por defecto. Cambia a `var` solo cuando necesites reasignar.
El compilador te avisará si declaras un `var` que nunca cambia.

---

## Tipos básicos

En Kotlin todos los tipos son objetos — no existen los primitivos de Java
(`int`, `long`, `boolean`). El compilador los optimiza a primitivos en bytecode.

```kotlin
fun main() {
    // Números enteros
    val a: Byte   = 127
    val b: Short  = 32_767       // _ como separador visual (igual que Java)
    val c: Int    = 2_147_483_647
    val d: Long   = 9_223_372_036_854_775_807L  // sufijo L para Long

    // Números decimales
    val e: Float  = 3.14f        // sufijo f para Float
    val f: Double = 3.14159265   // Double por defecto

    // Otros
    val g: Boolean = true
    val h: Char    = 'K'         // comillas simples para Char
    val i: String  = "Kotlin"    // comillas dobles para String

    // Inferencia de tipos — Kotlin deduce el tipo del valor
    val inferido = 42            // Int
    val inferido2 = 42L          // Long por el sufijo L
    val inferido3 = 3.14         // Double
    val inferido4 = true         // Boolean

    println("Int max: $c")
    println("Tipo de inferido: ${inferido::class.simpleName}")
}
```

---

## String templates

Las plantillas de cadena son una de las características más usadas de Kotlin.
Permiten insertar expresiones dentro de un `String` con `$` y `${ }`:

```kotlin
fun main() {
    val nombre  = "Ana"
    val apellido = "García"
    val edad    = 28

    // Variable simple — solo $nombre
    println("Hola, $nombre")

    // Expresión — requiere llaves ${ }
    println("Nombre completo: ${nombre.uppercase()} ${apellido.uppercase()}")
    println("El año que viene tendrás ${edad + 1} años")
    println("¿Es mayor de edad? ${if (edad >= 18) "Sí" else "No"}")

    // String multiline — triple comillas
    val tarjeta = """
        |Nombre:  $nombre $apellido
        |Edad:    $edad
        |Acceso:  ${if (edad >= 18) "Permitido" else "Denegado"}
    """.trimMargin()   // trimMargin elimina el | y espacios al inicio

    println(tarjeta)
}
```

Salida:
```
Hola, Ana
Nombre completo: ANA GARCÍA
El año que viene tendrás 29 años
¿Es mayor de edad? Sí
Nombre:  Ana García
Edad:    28
Acceso:  Permitido
```

---

## Conversiones de tipos

Kotlin **no hace conversiones implícitas** — si quieres convertir entre tipos
numéricos tienes que llamar a la función correspondiente:

```kotlin
fun main() {
    val entero: Int = 42

    // Java: double decimal = entero;  → automático
    // Kotlin: hay que ser explícito
    val decimal: Double = entero.toDouble()
    val largo:   Long   = entero.toLong()
    val texto:   String = entero.toString()

    println(decimal)  // 42.0
    println(largo)    // 42
    println(texto)    // "42"

    // String a número
    val numero = "123".toInt()
    val numero2 = "3.14".toDouble()

    // toIntOrNull — devuelve null si no se puede convertir
    val invalido = "abc".toIntOrNull()  // null, no lanza excepción
    println(invalido)  // null

    // Conversión explícita evita bugs silenciosos
    val a: Int  = 1_000_000
    val b: Long = a.toLong() * 1_000_000L   // sin toLong() el resultado podría desbordar
    println(b)  // 1_000_000_000_000
}
```

---

## La función `main` — punto de entrada

```kotlin
// Forma estándar — sin argumentos
fun main() {
    println("Sin argumentos")
}

// Con argumentos de línea de comandos
fun main(args: Array<String>) {
    println("Argumentos recibidos: ${args.size}")
    args.forEach { println("  - $it") }
}
```

En Android Studio, ejecuta el programa haciendo clic en el ícono ▶ verde
que aparece junto a `fun main()` en el margen izquierdo.

---

## `println`, `print` y `readLine`

```kotlin
fun main() {
    print("Escribe tu nombre: ")    // sin salto de línea al final
    val nombre = readLine()          // lee una línea de la entrada estándar
    println("Hola, $nombre!")        // con salto de línea al final

    // readLine() devuelve String? (nullable)
    // ?: "anónimo" es el operador Elvis — lo veremos en detalle en página 5
    val nombreSeguro = readLine() ?: "anónimo"
    println("Hola, $nombreSeguro")
}
```

---

## Comentarios

```kotlin
// Comentario de una línea

/*
   Comentario
   de múltiples
   líneas
*/

/**
 * Comentario KDoc — equivalente a Javadoc
 * Documentación de funciones y clases.
 *
 * @param nombre El nombre del usuario
 * @return Un saludo personalizado
 */
fun saludar(nombre: String): String {
    return "Hola, $nombre"
}
```

---

## Operadores aritméticos

```kotlin
fun main() {
    val a = 10
    val b = 3

    println(a + b)   // 13 — suma
    println(a - b)   // 7  — resta
    println(a * b)   // 30 — multiplicación
    println(a / b)   // 3  — división entera (ambos operandos son Int)
    println(a % b)   // 1  — módulo (resto de la división)

    // División real — al menos un operando debe ser Double o Float
    println(a.toDouble() / b)   // 3.3333333333333335
    println(10.0 / 3)           // 3.3333333333333335
    println(10 / 3.0)           // 3.3333333333333335

    // Operadores de asignación compuesta
    var x = 10
    x += 5    // x = x + 5  → 15
    x -= 3    // x = x - 3  → 12
    x *= 2    // x = x * 2  → 24
    x /= 4    // x = x / 4  → 6
    x %= 4    // x = x % 4  → 2
    println(x)  // 2

    // Incremento y decremento
    var contador = 0
    contador++   // contador = 1
    contador++   // contador = 2
    contador--   // contador = 1
    println(contador)  // 1
}
```

**Truco con módulo** — `n % 2 == 0` es la forma idiomática de comprobar si un número es par.

> ⚠️ La división entre `Int` siempre produce `Int` (trunca el resultado). Este es un error frecuente en principiantes: `7 / 2` da `3`, no `3.5`. Para obtener el resultado real, convierte al menos un operando: `7.toDouble() / 2` o simplemente `7.0 / 2`.

---

## Operadores de comparación

Devuelven `Boolean` (`true` o `false`). Son la base de las condicionales que veremos en la página siguiente.

```kotlin
fun main() {
    val a = 10
    val b = 3

    println(a == b)   // false — igual a
    println(a != b)   // true  — distinto de
    println(a >  b)   // true  — mayor que
    println(a <  b)   // false — menor que
    println(a >= b)   // true  — mayor o igual que
    println(a <= b)   // false — menor o igual que

    // Comparación de Strings — == compara por valor (no por referencia como en Java)
    val s1 = "Kotlin"
    val s2 = "Kotlin"
    println(s1 == s2)          // true  — mismo contenido
    println(s1.equals(s2))     // true  — equivalente
    println(s1 === s2)         // true  — misma referencia (raramente necesario)

    // Comparación de rangos — muy útil antes de usar when/if
    val edad = 20
    println(edad in 18..65)    // true  — edad está entre 18 y 65 (inclusive)
    println(edad !in 0..17)    // true  — edad NO está en ese rango
}
```

> En Kotlin, `==` siempre compara **por valor** (llama a `equals` internamente). Para comparar referencias de objeto se usa `===`. Esto es diferente a Java, donde `==` compara referencias en objetos.

---

## Operadores lógicos

Combinan expresiones booleanas. Se usan con `if`, `while` y `when`.

```kotlin
fun main() {
    val esMayor      = true
    val tienePermiso = false
    val estaActivo   = true

    // && — AND lógico: ambas condiciones deben ser true
    println(esMayor && tienePermiso)   // false
    println(esMayor && estaActivo)     // true

    // || — OR lógico: al menos una condición debe ser true
    println(esMayor || tienePermiso)   // true
    println(tienePermiso || false)     // false

    // ! — NOT lógico: invierte el valor
    println(!esMayor)       // false
    println(!tienePermiso)  // true

    // Evaluación en cortocircuito (short-circuit)
    // Con &&: si el primer operando es false, el segundo NO se evalúa
    // Con ||: si el primer operando es true,  el segundo NO se evalúa
    val lista = listOf(1, 2, 3)
    if (lista.isNotEmpty() && lista[0] > 0) {
        println("Primer elemento positivo")   // seguro — isNotEmpty() se evalúa primero
    }

    // Combinando operadores — los paréntesis mejoran la legibilidad
    val puedeEntrar = esMayor && (tienePermiso || estaActivo)
    println(puedeEntrar)  // true
}
```

**Tabla de verdad:**

| `a` | `b` | `a && b` | `a \|\| b` | `!a` |
|-----|-----|----------|------------|------|
| true | true | true | true | false |
| true | false | false | true | false |
| false | true | false | true | true |
| false | false | false | false | true |

---

## Precedencia de operadores

Cuando hay varios operadores en una expresión, Kotlin los evalúa en este orden (de mayor a menor precedencia):

| Prioridad | Operadores | Ejemplo |
|-----------|-----------|---------|
| 1 (más alta) | `++`, `--`, unario `-`, `!` | `!true`, `-5` |
| 2 | `*`, `/`, `%` | `2 * 3 + 1` → `7` |
| 3 | `+`, `-` | `2 + 3 * 4` → `14` |
| 4 | `..`, `..<` | rangos |
| 5 | `in`, `!in`, `is`, `!is` | `x in 1..10` |
| 6 | `<`, `>`, `<=`, `>=` | comparaciones |
| 7 | `==`, `!=` | igualdad |
| 8 | `&&` | AND lógico |
| 9 (más baja) | `\|\|` | OR lógico |

```kotlin
fun main() {
    // Sin paréntesis — sigue la precedencia
    println(2 + 3 * 4)               // 14 (no 20) — * antes que +
    println(10 - 2 + 3)              // 11 — asociatividad izquierda
    println(true || false && false)   // true — && antes que ||

    // Con paréntesis — siempre más claro
    println((2 + 3) * 4)             // 20
    println(true || (false && false)) // true — lo mismo pero explícito

    // Ejemplo real con combinación de operadores
    val x = 8
    val resultado = x > 5 && x % 2 == 0  // true: x es mayor que 5 Y es par
    println(resultado)  // true
}
```

**Regla práctica**: usa paréntesis cuando combines `&&` y `||` en la misma expresión. Mejora la legibilidad y evita bugs silenciosos.

---

## Operadores de rango

Los rangos son objetos de primera clase en Kotlin. Se usan con `in`, en bucles `for` y en expresiones `when`.

```kotlin
fun main() {
    // Rango cerrado: incluye ambos extremos
    val rango = 1..10           // IntRange: 1, 2, 3, ..., 10

    // Rango abierto por la derecha: excluye el extremo superior
    val rangoAbierto = 1..<10   // 1, 2, 3, ..., 9  (equivale a 1 until 10)

    // Rango descendente
    val descendente = 10 downTo 1   // 10, 9, 8, ..., 1

    // Con paso personalizado
    val pares   = 0..20 step 2      // 0, 2, 4, ..., 20
    val impares = 1..20 step 2      // 1, 3, 5, ..., 19

    // Comprobar pertenencia con el operador in
    val edad = 25
    println(edad in 18..65)          // true
    println(edad !in 0..17)          // true
    println('a' in 'a'..'z')         // true — funciona con Char también
    println("mango" in "apple".."orange")  // true — orden lexicográfico

    // Recorrer rangos con for (los bucles se estudian en la página 2)
    for (i in 1..5) print("$i ")     // 1 2 3 4 5
    println()
    for (i in pares) print("$i ")    // 0 2 4 6 8 10 12 14 16 18 20
    println()
    for (i in 10 downTo 1 step 3) print("$i ")  // 10 7 4 1
    println()
}
```

**Resumen de constructores de rango:**

| Expresión | Resultado |
|-----------|-----------|
| `1..10` | 1, 2, 3, …, 10 (cerrado) |
| `1..<10` | 1, 2, 3, …, 9 (abierto por la derecha) |
| `1 until 10` | igual que `1..<10` |
| `10 downTo 1` | 10, 9, 8, …, 1 |
| `1..10 step 2` | 1, 3, 5, 7, 9 |
| `10 downTo 1 step 3` | 10, 7, 4, 1 |

---

## Operador `is` — comprobación de tipo en tiempo de ejecución

Permite preguntar si un objeto es de un tipo determinado. Es especialmente útil antes de acceder a propiedades o métodos específicos del tipo.

```kotlin
fun main() {
    val valor: Any = "Hola, Kotlin"   // Any es la superclase de todos los tipos en Kotlin

    // is — comprueba el tipo
    println(valor is String)    // true
    println(valor is Int)       // false
    println(valor !is Int)      // true

    // Smart cast — después de comprobar con is, Kotlin "recuerda" el tipo
    // y no hay que hacer un cast explícito
    if (valor is String) {
        // Aquí, 'valor' ya es tratado como String automáticamente
        println(valor.length)          // 12 — sin (valor as String).length
        println(valor.uppercase())     // HOLA, KOTLIN
    }

    // as — cast explícito (lanza ClassCastException si el tipo no coincide)
    val texto = valor as String
    println(texto.reversed())   // niltoK ,aloH

    // as? — cast seguro (devuelve null si falla, no lanza excepción)
    val numero = valor as? Int  // null — valor no es Int
    println(numero)             // null

    // Ejemplo práctico: función que acepta cualquier tipo
    fun describir(obj: Any): String {
        return when (obj) {
            is Int    -> "Número entero: $obj"
            is Double -> "Número decimal: $obj"
            is String -> "Texto de ${obj.length} caracteres: $obj"
            is Boolean -> "Booleano: $obj"
            else      -> "Tipo desconocido"
        }
    }

    println(describir(42))          // Número entero: 42
    println(describir(3.14))        // Número decimal: 3.14
    println(describir("Kotlin"))    // Texto de 6 caracteres: Kotlin
    println(describir(true))        // Booleano: true
}
```

> El **smart cast** es una de las características más cómodas de Kotlin: una vez que el compilador sabe que `valor is String`, trata `valor` como `String` dentro de ese bloque sin necesidad de ningún cast explícito. Lo veremos mucho en combinación con `when` (página 2).

---

## Expresiones vs. sentencias

En Kotlin, muchas construcciones son **expresiones** — producen un valor y pueden usarse directamente en el lado derecho de una asignación. Esto contrasta con lenguajes como Java donde la mayoría son solo sentencias.

```kotlin
fun main() {
    val a = 10
    val b = 3

    // Las operaciones aritméticas y lógicas son expresiones
    val suma     = a + b          // expresión aritmética → Int
    val esIgual  = a == b         // expresión de comparación → Boolean
    val esPar    = a % 2 == 0     // combinación de expresiones → Boolean

    println(suma)     // 13
    println(esIgual)  // false
    println(esPar)    // true

    // if como expresión — produce un valor directamente
    val resultado = if (a > b) "mayor" else "menor"
    println(resultado)   // mayor

    // Uso directo en string templates
    println("$a ${if (a > b) ">" else "<"} $b")   // 10 > 3
    println("${a} es ${if (a % 2 == 0) "par" else "impar"}")  // 10 es par

    // La asignación en sí es una sentencia — no produce valor
    // (a diferencia de Java, donde int x = (y = 5) está permitido)
    // Esto evita el clásico bug: if (x = 5) en vez de if (x == 5)
}
```

> El `if` como expresión es uno de los rasgos más idiomáticos de Kotlin — lo exploraremos en profundidad en la página 2, junto con `when`, que lo complementa de manera aún más poderosa.

---

## Primer programa completo

```kotlin
// src/main/kotlin/Main.kt

fun main() {
    println("=== Calculadora simple ===")

    print("Primer número: ")
    val a = readLine()?.toDoubleOrNull() ?: 0.0

    print("Segundo número: ")
    val b = readLine()?.toDoubleOrNull() ?: 0.0

    val suma       = a + b
    val diferencia = a - b
    val producto   = a * b
    val cociente   = if (b != 0.0) a / b else "indefinido"

    // Operadores adicionales demostrados
    val aInt = a.toInt()
    val bInt = if (b.toInt() != 0) b.toInt() else 1
    val modulo = aInt % bInt
    val esPar  = aInt % 2 == 0

    println("""
        |Resultados para $a y $b:
        |  Suma:        $suma
        |  Diferencia:  $diferencia
        |  Producto:    $producto
        |  División:    $cociente
        |  Módulo ($aInt % $bInt): $modulo
        |  ¿$aInt es par? $esPar
        |  ¿$aInt está en 1..100? ${aInt in 1..100}
    """.trimMargin())
}
```

Ejecución de ejemplo:
```
=== Calculadora simple ===
Primer número: 10
Segundo número: 3
Resultados para 10.0 y 3.0:
  Suma:        13.0
  Diferencia:  7.0
  Producto:    30.0
  División:    3.3333333333333335
  Módulo (10 % 3): 1
  ¿10 es par? true
  ¿10 está en 1..100? true
```

---

## Ejercicios propuestos

1. **Presentación** — Declara variables `val` para nombre, edad y ciudad.
   Imprime una presentación usando string templates con al menos una expresión
   dentro de `${ }`.

2. **Conversiones** — Lee un número desde la consola con `readLine()`.
   Conviértelo a `Int`, `Double` y `Long`. Imprime los tres valores.
   Maneja el caso en que el usuario escriba texto no numérico usando `toIntOrNull()`.

3. **REPL** — Abre el REPL de Kotlin y experimenta con:
   - `"kotlin 2.3.0".split(" ")`
   - `(1..10).filter { it % 2 == 0 }`
   - `mapOf("a" to 1, "b" to 2).values.sum()`

4. **Operadores** — Declara dos variables `val x: Int` e `val y: Int` con los
   valores que quieras. Calcula e imprime: suma, diferencia, producto, cociente
   real (con `toDouble()`) y módulo. Luego imprime:
   - Si `x` es par (usando `%`)
   - Si `x` está en el rango `1..100` (usando `in`)
   - Si `x > y && x % 2 == 0`

5. **Precedencia** — Evalúa mentalmente cada expresión y luego comprueba el
   resultado en el REPL:
   - `2 + 3 * 4`
   - `(2 + 3) * 4`
   - `10 / 3`
   - `10.0 / 3`
   - `true || false && false`
   - `(true || false) && false`

6. **Tipo en tiempo de ejecución** — Crea una lista `listOf(1, "dos", 3.0, true, 'K')`.
   Recórrela con `forEach` e imprime una descripción de cada elemento usando
   `is` (sin `when` por ahora — solo con `if / else if`).

---

## Convenciones del curso

A lo largo del curso usamos estas convenciones:

```kotlin
// ✅ Correcto — lo que el curso recomienda
val nombre = "Ana"

// ❌ Incorrecto — lo que hay que evitar
var nombre = "Ana"   // innecesariamente mutable
```

- `val` por defecto, `var` solo cuando es necesario
- Nombres de variables y funciones en `camelCase`
- Nombres de clases en `PascalCase`
- Constantes en `UPPER_SNAKE_CASE` dentro de `companion object`
- Indentación de 4 espacios (el formatter de Android Studio lo aplica automáticamente)
- Punto y coma `;` nunca al final de línea
- Tipo explícito en declaraciones públicas de funciones, implícito en variables locales

---

## Resumen de la página 1

- Kotlin compila a JVM bytecode — toda la librería de Java está disponible, más las APIs propias de Kotlin.
- `val` declara una referencia inmutable — equivalente a `final` en Java. Úsalo por defecto.
- `var` declara una referencia mutable — solo cuando el valor necesita cambiar.
- Los tipos se infieren del valor asignado — no siempre es necesario escribirlos explícitamente.
- Los String templates con `$variable` y `${expresión}` evitan la concatenación con `+`.
- Kotlin **no hace conversiones numéricas implícitas** — hay que llamar a `.toInt()`, `.toDouble()`, etc.
- `readLine()` devuelve `String?` (nullable) — lo gestionamos con `?.toInt()` y el operador `?:`.
- El REPL de Android Studio permite probar expresiones sin compilar el proyecto.
- La división entre `Int` siempre produce `Int` (trunca) — para división real, al menos un operando debe ser `Double`.
- `&&` y `||` usan **evaluación en cortocircuito** — el segundo operando no se evalúa si el primero ya determina el resultado.
- Los rangos (`1..10`, `downTo`, `step`) son objetos de primera clase y se pueden usar con `in` para comprobar pertenencia.
- El operador `is` activa el **smart cast** — después de comprobar el tipo, no hace falta un cast explícito.
- Muchas construcciones en Kotlin son **expresiones** que producen un valor — `if` incluido.

---

> **Siguiente página →** Página 2: Control de flujo — `if` como expresión,
> `when` exhaustivo, bucles y rangos.