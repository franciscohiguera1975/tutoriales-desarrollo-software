
# Curso Kotlin — Página 4
## Módulo 1 · Fundamentos
### Null Safety: tipos nullable, operadores `?.` `?:` `!!` y scope functions

---

## El problema del null

En Java, cualquier variable de referencia puede ser `null` en cualquier momento.
El compilador no lo detecta — el error aparece en runtime como `NullPointerException`.

```java
// Java — compila sin problemas, explota en runtime
String nombre = null;
int longitud = nombre.length(); // NullPointerException en runtime
```

Kotlin resuelve esto en el sistema de tipos: **los tipos no nullable no pueden
contener `null`**. Si quieres permitir `null`, debes declararlo explícitamente
con el operador `?`:

```kotlin
// Kotlin — error de compilación, no de runtime
var nombre: String = "Ana"
// nombre = null  // ERROR: Null can not be a value of a non-null type String

// Para permitir null, usa String? (tipo nullable)
var nombreNullable: String? = "Ana"
nombreNullable = null  // OK — String? acepta null

// El compilador no te deja usar un String? como String sin verificar
// println(nombreNullable.length)  // ERROR: Only safe (?.) or non-null asserted (!!.) calls are allowed
```

---

## Tipos nullable vs no-nullable

```kotlin
fun main() {
    // No-nullable — garantizado nunca null
    val a: String  = "Kotlin"
    val b: Int     = 42
    val c: Boolean = true

    // Nullable — puede ser null
    val x: String?  = null
    val y: Int?     = null
    val z: Boolean? = null

    // Inferencia — Kotlin infiere el tipo más restringido posible
    val inferido1 = "texto"     // String (no-nullable)
    val inferido2 = null        // Nothing? — raramente útil solo
    val inferido3: String? = if (a.length > 3) a else null  // String?

    println("a=$a, b=$b, c=$c")
    println("x=$x, y=$y, z=$z")
}
```

---

## Operador de llamada segura `?.`

El operador `?.` ejecuta la operación **solo si el receptor no es null**.
Si es null, devuelve `null` en lugar de lanzar una excepción:

```kotlin
fun main() {
    val nombre: String? = "Ana García"
    val vacio:  String? = null

    // Sin ?.  — error de compilación porque nombre es String?
    // println(nombre.length)

    // Con ?.  — devuelve Int? (null si nombre es null)
    println(nombre?.length)   // 10
    println(vacio?.length)    // null

    // Encadenamiento — si cualquier eslabón es null, devuelve null
    val ciudad: String? = null
    println(ciudad?.uppercase()?.trim()?.length)  // null

    // Con objetos anidados
    data class Direccion(val ciudad: String?, val pais: String)
    data class Usuario(val nombre: String, val direccion: Direccion?)

    val usuario = Usuario("Ana", Direccion("Madrid", "España"))
    val usuarioSinDir = Usuario("Luis", null)

    println(usuario.direccion?.ciudad)          // Madrid
    println(usuarioSinDir.direccion?.ciudad)    // null
    println(usuario.direccion?.ciudad?.length)  // 6
}
```

---

## Operador Elvis `?:`

El operador `?:` proporciona un **valor por defecto** cuando la expresión
de la izquierda es `null`:

```kotlin
fun main() {
    val nombre: String? = null
    val ciudad: String? = "Madrid"

    // Si nombre es null, usa "Desconocido"
    val nombreSeguro = nombre ?: "Desconocido"
    println(nombreSeguro)  // Desconocido

    val ciudadSegura = ciudad ?: "Sin ciudad"
    println(ciudadSegura)  // Madrid

    // Combinación con ?.
    val longitud = nombre?.length ?: 0
    println(longitud)  // 0

    // El lado derecho puede ser cualquier expresión
    val entrada = readLine()
    val numero = entrada?.toIntOrNull() ?: -1
    println("Número: $numero")

    // throw con Elvis — para validar y fallar rápido
    fun obtenerNombre(input: String?): String {
        return input?.trim()?.takeIf { it.isNotEmpty() }
            ?: throw IllegalArgumentException("El nombre no puede estar vacío")
    }

    println(obtenerNombre("  Ana  "))  // Ana
    // obtenerNombre(null)             // lanza IllegalArgumentException
    // obtenerNombre("   ")            // lanza IllegalArgumentException
}
```

---

## Operador `!!` — Non-null assertion

El operador `!!` convierte `T?` en `T` y **lanza `NullPointerException`** si
el valor es null. Es la "salida de emergencia" del sistema de null safety:

```kotlin
fun main() {
    val nombre: String? = "Ana"
    val vacio: String?  = null

    // !! lanza NPE si el valor es null
    val longitud = nombre!!.length   // OK — nombre no es null
    println(longitud)  // 3

    // vacio!!.length   // NullPointerException en runtime

    // Cuándo usar !!:
    // Solo cuando SABES con certeza que el valor no es null
    // pero el compilador no puede demostrarlo por contexto externo
    //
    // En la práctica, !! es una señal de que algo se puede mejorar.
    // Si lo ves en código propio, pregúntate si puedes usar ?: o ?.let
}
```

> **Regla práctica**: evita `!!`. Úsalo solo cuando llames a código Java
> que puede devolver null pero estás seguro de que no lo hará.
> Si lo usas en código Kotlin propio, hay casi siempre una forma mejor.

---

## Smart cast

Después de una comprobación de null (o de tipo), el compilador
"recuerda" el resultado y trata la variable como el tipo restringido:

```kotlin
fun main() {
    val texto: String? = "Kotlin"

    // Sin smart cast — texto sigue siendo String?
    // println(texto.length)  // ERROR

    // Después de la comprobación, texto es tratado como String
    if (texto != null) {
        println(texto.length)        // OK — String, no String?
        println(texto.uppercase())   // OK
    }

    // when también aplica smart cast
    val valor: Any = 42
    when (valor) {
        is String -> println("Texto: ${valor.uppercase()}")   // valor es String aquí
        is Int    -> println("Número al cuadrado: ${valor * valor}") // valor es Int aquí
        is List<*> -> println("Lista con ${valor.size} elementos")
    }

    // Smart cast NO funciona si la variable puede cambiar entre
    // la comprobación y el uso (var mutable o propiedad abierta)
    var mutable: String? = "valor"
    // if (mutable != null) { mutable = null; mutable.length }  // ERROR — puede cambiar
    
    // Solución: asignar a un val local
    val inmutable = mutable
    if (inmutable != null) {
        println(inmutable.length)  // OK — inmutable no puede cambiar
    }
}
```

---

## Scope functions: `let`, `run`, `also`, `apply`, `with`

Las scope functions son funciones de la stdlib que ejecutan un bloque
de código en el contexto de un objeto. La diferencia entre ellas está
en cómo se accede al objeto dentro del bloque y qué devuelven.

```
Función  │ Objeto dentro   │ Retorna        │ Uso principal
─────────┼─────────────────┼────────────────┼──────────────────────────────
let      │ it              │ resultado lambda│ Transformar nullable, encadenar
run      │ this            │ resultado lambda│ Inicializar y calcular resultado
also     │ it              │ el objeto mismo │ Efectos secundarios (logging)
apply    │ this            │ el objeto mismo │ Configurar un objeto
with     │ this (no ext.)  │ resultado lambda│ Operar sobre un objeto
```

### `let` — transformar y encadenar

```kotlin
fun main() {
    // Uso principal: ejecutar código solo si el valor no es null
    val nombre: String? = "  ana garcía  "

    val resultado = nombre?.let { n ->
        n.trim().capitalizar()
    }
    println(resultado)  // Ana García (o null si nombre era null)

    // Encadenamiento de transformaciones
    val numero: Int? = "42".toIntOrNull()
    val cuadrado = numero?.let { it * it }
    println(cuadrado)  // 1764

    // Sin nombre explícito — 'it' es el parámetro implícito
    val longitud = "Kotlin".let { it.length * 2 }
    println(longitud)  // 12

    // Encadenar let para pipeline de transformaciones
    val procesado = "  HOLA MUNDO  "
        .let { it.trim() }
        .let { it.lowercase() }
        .let { it.split(" ") }
        .let { it.joinToString("-") }
    println(procesado)  // hola-mundo
}

fun String.capitalizar(): String =
    split(" ").joinToString(" ") { word ->
        word.lowercase().replaceFirstChar { it.uppercase() }
    }
```

### `apply` — configurar un objeto

```kotlin
data class Configuracion(
    var host:    String  = "localhost",
    var puerto:  Int     = 8080,
    var https:   Boolean = false,
    var timeout: Int     = 30
)

fun main() {
    // Sin apply — repetición del nombre
    val config1 = Configuracion()
    config1.host    = "api.ejemplo.com"
    config1.puerto  = 443
    config1.https   = true
    config1.timeout = 60

    // Con apply — más limpio, devuelve el objeto configurado
    val config2 = Configuracion().apply {
        host    = "api.ejemplo.com"  // this.host = ... (this implícito)
        puerto  = 443
        https   = true
        timeout = 60
    }

    println(config2)
    // Configuracion(host=api.ejemplo.com, puerto=443, https=true, timeout=60)

    // apply es ideal para builders y configuraciones
    val sb = StringBuilder().apply {
        append("Hola")
        append(", ")
        append("Kotlin")
        append("!")
    }
    println(sb.toString())  // Hola, Kotlin!
}
```

### `also` — efectos secundarios sin romper la cadena

```kotlin
fun procesarDatos(entrada: String?): String? {
    return entrada
        ?.trim()
        .also { println("Después de trim: '$it'") }
        ?.uppercase()
        .also { println("Después de uppercase: '$it'") }
        ?.take(10)
        .also { println("Resultado final: '$it'") }
}

fun main() {
    procesarDatos("  hola mundo desde kotlin  ")
}
```

Salida:
```
Después de trim: 'hola mundo desde kotlin'
Después de uppercase: 'HOLA MUNDO DESDE KOTLIN'
Resultado final: 'HOLA MUNDO D'
```

### `run` — calcular un resultado en el contexto del objeto

```kotlin
data class Circulo(val radio: Double) {
    val area:         Double get() = Math.PI * radio * radio
    val circunferencia: Double get() = 2 * Math.PI * radio
}

fun main() {
    val resultado = Circulo(5.0).run {
        // this es el Circulo dentro del bloque
        val areaFormateada = "%.2f".format(area)
        val circFormateada = "%.2f".format(circunferencia)
        "Circulo(r=$radio): área=$areaFormateada, circ=$circFormateada"
    }
    println(resultado)
    // Circulo(r=5.0): área=78.54, circ=31.42

    // run sin receptor — simplemente ejecuta un bloque y devuelve su resultado
    val inicializado = run {
        val x = 10
        val y = 20
        x + y  // valor retornado
    }
    println(inicializado)  // 30
}
```

### `with` — operar sobre un objeto (no extensión)

```kotlin
data class Reporte(
    val titulo:     String,
    val autor:      String,
    val paginas:    Int,
    val publicado:  Int
)

fun main() {
    val reporte = Reporte("Kotlin en Producción", "Ana García", 320, 2025)

    // with — igual que run pero recibe el objeto como argumento, no receptor
    val resumen = with(reporte) {
        """
        |Título:    $titulo
        |Autor:     $autor
        |Páginas:   $paginas
        |Año:       $publicado
        |Lectura:   ~${paginas / 50} horas
        """.trimMargin()
    }
    println(resumen)
}
```

---

## `lateinit` y `lazy`

Dos formas de diferir la inicialización de una propiedad:

```kotlin
class Servicio {
    // lateinit — se inicializa más tarde, antes de usarse
    // Solo para var, solo para tipos de referencia (no Int, Double, etc.)
    lateinit var conexion: String

    fun inicializar(host: String) {
        conexion = host
    }

    fun conectar() {
        // isInitialized verifica si ya fue inicializado
        if (::conexion.isInitialized) {
            println("Conectando a $conexion")
        } else {
            println("No inicializado")
        }
    }
}

// lazy — se inicializa la PRIMERA VEZ que se accede
// Solo para val, hilo-seguro por defecto
val configuracion: String by lazy {
    println("Calculando configuración...")
    "configuracion-calculada"
}

fun main() {
    // lazy — solo se calcula al acceder por primera vez
    println("Antes de acceder")
    println(configuracion)  // Calculando configuración... (imprime una vez)
    println(configuracion)  // No recalcula
    println(configuracion)  // No recalcula

    // lateinit
    val servicio = Servicio()
    servicio.conectar()     // No inicializado
    servicio.inicializar("api.ejemplo.com")
    servicio.conectar()     // Conectando a api.ejemplo.com
}
```

---

## Programa completo — procesador de entradas del usuario

```kotlin
data class Perfil(
    val nombre:   String,
    val email:    String,
    val edad:     Int,
    val website:  String?
)

fun parsearPerfil(datos: Map<String, String?>): Perfil? {
    val nombre  = datos["nombre"]?.trim()?.takeIf { it.isNotEmpty() } ?: return null
    val email   = datos["email"]?.trim()?.takeIf { it.contains("@") } ?: return null
    val edadStr = datos["edad"]?.trim()
    val edad    = edadStr?.toIntOrNull()?.takeIf { it in 1..120 } ?: return null
    val website = datos["website"]?.trim()?.takeIf { it.startsWith("http") }

    return Perfil(nombre, email, edad, website)
}

fun main() {
    val entradas = listOf(
        mapOf("nombre" to "Ana García", "email" to "ana@test.com", "edad" to "28",   "website" to "https://ana.dev"),
        mapOf("nombre" to "",           "email" to "sin@nombre",   "edad" to "25",   "website" to null),
        mapOf("nombre" to "Luis",       "email" to "invalido",     "edad" to "30",   "website" to null),
        mapOf("nombre" to "María",      "email" to "m@test.com",   "edad" to "abc",  "website" to null),
        mapOf("nombre" to "Carlos",     "email" to "c@test.com",   "edad" to "35",   "website" to "no-es-url"),
    )

    for ((indice, datos) in entradas.withIndex()) {
        val perfil = parsearPerfil(datos)
        if (perfil != null) {
            println("✅ Perfil $indice: ${perfil.nombre} (${perfil.email})")
            perfil.website?.let { println("   Web: $it") }
        } else {
            println("❌ Perfil $indice inválido: ${datos["nombre"] ?: "(sin nombre)"}")
        }
    }
}
```

Salida:
```
✅ Perfil 0: Ana García (ana@test.com)
   Web: https://ana.dev
❌ Perfil 1 inválido: (sin nombre)
❌ Perfil 2 inválido: Luis
❌ Perfil 3 inválido: María
✅ Perfil 4: Carlos (c@test.com)
```

---

## Ejercicios propuestos

1. **Cadena de transformaciones** — Escribe una función
   `fun procesarTexto(entrada: String?): String` que use `?.let` y `?:`
   para: hacer trim, verificar que no esté vacío, convertir a mayúsculas
   y truncar a 20 caracteres. Si la entrada es null o vacía, devuelve `"[vacío]"`.

2. **Config builder con `apply`** — Crea una `data class` `ConexionBD` con
   propiedades como `host`, `puerto`, `usuario`, `contrasena`, `baseDatos`.
   Usa `apply` para construir dos configuraciones distintas (desarrollo y producción).

3. **Pipeline con `also`** — Escribe una función que procese una lista de números
   como String (`"1,2,abc,4,5,xyz"`), use `also` para loguear cada paso:
   split, conversión a Int ignorando inválidos, filtrar pares, calcular suma.

4. **`lazy` vs `lateinit`** — Crea una clase `CacheServicio` que tenga:
   - Una propiedad `lateinit var cliente: String` que se inicializa con `conectar(host)`
   - Una propiedad `val configuracion by lazy { cargarConfiguracion() }`
   - Una función `cargarConfiguracion()` que imprima "Cargando..." y devuelva un String

---

## Resumen de la página 4

- Los tipos no-nullable (`String`) **nunca** pueden ser `null`. Los nullable (`String?`) sí — y el compilador obliga a manejarlos.
- `?.` (safe call) ejecuta la operación solo si el receptor no es null, devolviendo null si lo es.
- `?:` (Elvis) proporciona un valor por defecto cuando la expresión izquierda es null. El lado derecho puede ser `throw`.
- `!!` convierte `T?` en `T` lanzando NPE si es null — evítalo en código Kotlin propio.
- El smart cast permite usar una variable como tipo restringido después de una comprobación, sin cast explícito.
- `let` — ejecuta un bloque con el objeto como `it`, devuelve el resultado del bloque. Ideal para nullable y pipelines.
- `apply` — configura un objeto con `this` implícito, devuelve el objeto. Ideal para constructores y builders.
- `also` — efectos secundarios con `it`, devuelve el objeto. Ideal para logging sin romper una cadena.
- `run` — calcula un resultado con `this` implícito, devuelve el resultado del bloque.
- `with` — igual que `run` pero recibe el objeto como argumento en lugar de ser extensión.
- `lateinit var` difiere la inicialización de referencias mutables. `val by lazy` inicializa al primer acceso.

---

> **Siguiente página →** Página 5: OOP — clases, `data class`, `object`,
> constructores y propiedades.