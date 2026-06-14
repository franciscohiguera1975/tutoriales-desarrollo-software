
# Curso Kotlin — Página 7
## Módulo 3 · Colecciones y funciones de orden superior
### `List`, `Set`, `Map`, operaciones funcionales y `Sequence`

---

## Colecciones en Kotlin

Kotlin distingue explícitamente entre colecciones **inmutables** y **mutables**.
Esta separación es una forma de **encapsulamiento**: evita modificaciones
accidentales y hace el código más predecible.

```
Colección        Inmutable          Mutable
────────────     ─────────────      ──────────────────
Lista ordenada   List<T>            MutableList<T>
Sin duplicados   Set<T>             MutableSet<T>
Clave → Valor   Map<K,V>           MutableMap<K,V>
```

> **Regla práctica**: usa la versión inmutable por defecto.
> Cambia a mutable solo cuando necesites modificar la colección.

---

## `List` — lista ordenada

```kotlin
fun main() {
    // Inmutable — listOf()
    val frutas = listOf("manzana", "banana", "cereza", "dátil", "banana")

    println(frutas.size)          // 5
    println(frutas[0])            // manzana
    println(frutas.first())       // manzana
    println(frutas.last())        // banana
    println(frutas.get(2))        // cereza
    println(frutas.indexOf("banana"))   // 1 (primera aparición)
    println(frutas.contains("cereza"))  // true
    println("cereza" in frutas)         // true — sintaxis con in

    // Sublistas
    println(frutas.subList(1, 3))       // [banana, cereza]
    println(frutas.take(2))             // [manzana, banana]
    println(frutas.drop(3))             // [dátil, banana]
    println(frutas.takeLast(2))         // [dátil, banana]

    // Mutable — mutableListOf()
    val colores = mutableListOf("rojo", "verde", "azul")
    colores.add("amarillo")
    colores.add(0, "negro")      // insertar en posición 0
    colores.remove("verde")
    colores[1] = "blanco"        // modificar por índice
    println(colores)             // [negro, blanco, azul, amarillo]

    // ArrayDeque — lista mutable eficiente para operaciones al inicio y al final
    val deque = ArrayDeque<Int>()
    deque.addFirst(1)
    deque.addLast(2)
    deque.addFirst(0)
    println(deque)               // [0, 1, 2]
    println(deque.removeFirst()) // 0
    println(deque.removeLast())  // 2
}
```

---

## `Set` — conjunto sin duplicados

```kotlin
fun main() {
    // Set ignora los duplicados
    val numeros = setOf(1, 2, 3, 2, 1, 4)
    println(numeros)          // [1, 2, 3, 4] — sin duplicados

    // Operaciones de conjuntos
    val pares    = setOf(2, 4, 6, 8, 10)
    val multiplos3 = setOf(3, 6, 9, 12)

    println(pares union multiplos3)        // [2, 4, 6, 8, 10, 3, 9, 12]
    println(pares intersect multiplos3)   // [6] — los que están en ambos
    println(pares subtract multiplos3)    // [2, 4, 8, 10] — en pares pero no en múltiplos3

    // Mutable
    val tags = mutableSetOf("kotlin", "android", "java")
    tags.add("kotlin")    // ignorado — ya existe
    tags.add("compose")
    tags.remove("java")
    println(tags)         // [kotlin, android, compose]

    // Verificar pertenencia — O(1) en HashSet, mucho más rápido que List
    println("kotlin" in tags)   // true
    println("java" in tags)     // false
}
```

---

## `Map` — diccionario clave → valor

```kotlin
fun main() {
    // Inmutable — mapOf() con pares clave to valor
    val capitales = mapOf(
        "España"    to "Madrid",
        "Francia"   to "París",
        "Alemania"  to "Berlín",
        "Italia"    to "Roma"
    )

    // Acceso
    println(capitales["España"])           // Madrid
    println(capitales["Portugal"])         // null — clave no existe
    println(capitales.getOrDefault("Portugal", "Desconocida"))  // Desconocida
    println(capitales.getOrElse("Portugal") { "No encontrado" }) // No encontrado

    // Iteración
    for ((pais, capital) in capitales) {
        println("$pais → $capital")
    }

    println(capitales.keys)    // [España, Francia, Alemania, Italia]
    println(capitales.values)  // [Madrid, París, Berlín, Roma]
    println(capitales.entries) // [España=Madrid, ...]

    // Mutable
    val inventario = mutableMapOf("manzana" to 10, "banana" to 5)
    inventario["cereza"]  = 20           // añadir
    inventario["banana"]  = 8            // modificar
    inventario.remove("manzana")         // eliminar
    inventario.getOrPut("uva") { 15 }    // añadir solo si no existe
    println(inventario)  // {banana=8, cereza=20, uva=15}
}
```

---

## Operaciones funcionales — el núcleo de las colecciones

Kotlin tiene un conjunto rico de operaciones de orden superior para
transformar, filtrar y agregar colecciones de forma declarativa:

### `map` — transformar cada elemento

```kotlin
fun main() {
    val numeros = listOf(1, 2, 3, 4, 5)

    // Eleva al cuadrado cada número
    val cuadrados = numeros.map { it * it }
    println(cuadrados)  // [1, 4, 9, 16, 25]

    // Transforma a String
    val textos = numeros.map { "Num$it" }
    println(textos)  // [Num1, Num2, Num3, Num4, Num5]

    data class Producto(val nombre: String, val precio: Double)
    val productos = listOf(
        Producto("Teclado", 89.99),
        Producto("Mouse",   29.99),
        Producto("Monitor", 349.99)
    )

    // Extraer solo los nombres
    val nombres  = productos.map { it.nombre }
    // Aplicar descuento del 10%
    val descuento = productos.map { it.copy(precio = it.precio * 0.90) }

    println(nombres)   // [Teclado, Mouse, Monitor]
    println(descuento) // [Producto(nombre=Teclado, precio=80.99), ...]
}
```

### `filter` — conservar los que cumplen la condición

```kotlin
fun main() {
    val numeros = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    val pares    = numeros.filter { it % 2 == 0 }
    val mayores5 = numeros.filter { it > 5 }
    val paresYmayores5 = numeros.filter { it % 2 == 0 && it > 5 }

    println(pares)          // [2, 4, 6, 8, 10]
    println(mayores5)       // [6, 7, 8, 9, 10]
    println(paresYmayores5) // [6, 8, 10]

    // filterNot — el complemento de filter
    val impares = numeros.filterNot { it % 2 == 0 }
    println(impares)  // [1, 3, 5, 7, 9]

    // filterIsInstance — filtrar por tipo (polimorfismo + colecciones)
    val mezcla: List<Any> = listOf(1, "hola", 2.5, "mundo", true, 42)
    val soloStrings = mezcla.filterIsInstance<String>()
    println(soloStrings)  // [hola, mundo]
}
```

### `reduce` y `fold` — agregar a un único valor

```kotlin
fun main() {
    val numeros = listOf(1, 2, 3, 4, 5)

    // reduce — combina usando el primer elemento como acumulador inicial
    val suma     = numeros.reduce { acc, n -> acc + n }
    val producto = numeros.reduce { acc, n -> acc * n }
    println(suma)     // 15
    println(producto) // 120

    // fold — igual pero con valor inicial explícito
    val sumaConInicial  = numeros.fold(100) { acc, n -> acc + n }
    val conCadena       = numeros.fold("Números: ") { acc, n -> "$acc$n " }
    println(sumaConInicial)   // 115
    println(conCadena)         // Números: 1 2 3 4 5

    // foldRight — de derecha a izquierda
    val listaInvertida = numeros.foldRight(mutableListOf<Int>()) { n, acc ->
        acc.apply { add(n) }
    }
    println(listaInvertida)  // [5, 4, 3, 2, 1]
}
```

### `groupBy` — agrupar por criterio

```kotlin
data class Empleado(val nombre: String, val depto: String, val salario: Double)

fun main() {
    val empleados = listOf(
        Empleado("Ana",    "IT",        75000.0),
        Empleado("Luis",   "IT",        68000.0),
        Empleado("María",  "RRHH",      55000.0),
        Empleado("Carlos", "IT",        82000.0),
        Empleado("Sofía",  "RRHH",      61000.0),
        Empleado("Pedro",  "Marketing", 58000.0)
    )

    // Agrupar por departamento — devuelve Map<String, List<Empleado>>
    val porDepto = empleados.groupBy { it.depto }
    porDepto.forEach { (depto, lista) ->
        val salarioMedio = lista.map { it.salario }.average()
        println("$depto (${lista.size} personas): salario medio $${"%.0f".format(salarioMedio)}")
    }
    // IT (3 personas): salario medio $75000
    // RRHH (2 personas): salario medio $58000
    // Marketing (1 personas): salario medio $58000

    // groupBy + transformValue
    val nombresPorDepto = empleados.groupBy(
        keySelector   = { it.depto },
        valueTransform = { it.nombre }
    )
    println(nombresPorDepto)
    // {IT=[Ana, Luis, Carlos], RRHH=[María, Sofía], Marketing=[Pedro]}
}
```

### Otras operaciones útiles

```kotlin
fun main() {
    val numeros = listOf(3, 1, 4, 1, 5, 9, 2, 6, 5, 3)

    // Ordenación
    println(numeros.sorted())                    // [1, 1, 2, 3, 3, 4, 5, 5, 6, 9]
    println(numeros.sortedDescending())          // [9, 6, 5, 5, 4, 3, 3, 2, 1, 1]
    println(numeros.sortedBy { -it })            // descendente por expresión

    // Agregación
    println(numeros.sum())                       // 39
    println(numeros.average())                   // 3.9
    println(numeros.min())                       // 1
    println(numeros.max())                       // 9
    println(numeros.count { it > 4 })            // 4

    // Búsqueda
    println(numeros.find { it > 4 })             // 5 — primer elemento que cumple
    println(numeros.findLast { it > 4 })         // 3? no — 6 — último que cumple
    println(numeros.any { it > 8 })              // true — alguno cumple
    println(numeros.all { it > 0 })              // true — todos cumplen
    println(numeros.none { it > 10 })            // true — ninguno cumple

    // Transformaciones
    println(numeros.distinct())                  // [3, 1, 4, 5, 9, 2, 6] — sin duplicados
    println(numeros.chunked(3))                  // [[3,1,4],[1,5,9],[2,6,5],[3]]
    println(numeros.windowed(3))                 // [[3,1,4],[1,4,1],[4,1,5],...]
    println(numeros.zip(numeros.drop(1)))        // [(3,1),(1,4),(4,1),...]

    // Aplanar listas de listas
    val matriz = listOf(listOf(1,2,3), listOf(4,5,6), listOf(7,8,9))
    println(matriz.flatten())                    // [1,2,3,4,5,6,7,8,9]

    // flatMap — map + flatten en una sola operación
    val palabras = listOf("hola mundo", "kotlin es genial")
    val letras = palabras.flatMap { it.split(" ") }
    println(letras)  // [hola, mundo, kotlin, es, genial]
}
```

---

## Encadenar operaciones

La potencia real está en encadenar operaciones para crear pipelines
de transformación de datos declarativos:

```kotlin
data class Pedido(
    val id:       Int,
    val cliente:  String,
    val total:    Double,
    val pagado:   Boolean,
    val items:    List<String>
)

fun main() {
    val pedidos = listOf(
        Pedido(1, "Ana",    150.0, true,  listOf("Teclado", "Mouse")),
        Pedido(2, "Luis",   89.0,  false, listOf("Webcam")),
        Pedido(3, "María",  320.0, true,  listOf("Monitor", "Hub")),
        Pedido(4, "Carlos", 45.0,  true,  listOf("Cable")),
        Pedido(5, "Sofía",  200.0, false, listOf("Auriculares", "Micrófono"))
    )

    // ¿Cuánto suma lo pagado?
    val totalPagado = pedidos
        .filter { it.pagado }
        .sumOf { it.total }
    println("Total pagado: $$totalPagado")   // Total pagado: $515.0

    // ¿Qué clientes tienen pedidos pendientes?
    val clientesPendientes = pedidos
        .filterNot { it.pagado }
        .map { it.cliente }
    println("Pendientes: $clientesPendientes")  // [Luis, Sofía]

    // ¿Cuáles son todos los artículos pedidos (sin duplicados)?
    val todosLosArticulos = pedidos
        .flatMap { it.items }
        .distinct()
        .sorted()
    println("Artículos: $todosLosArticulos")

    // Top 2 pedidos más caros entre los pagados
    val top2 = pedidos
        .filter { it.pagado }
        .sortedByDescending { it.total }
        .take(2)
        .map { "${it.cliente}: $${it.total}" }
    println("Top 2: $top2")  // [María: $320.0, Ana: $150.0]
}
```

---

## `Sequence` — evaluación perezosa

Las operaciones sobre `List` se evalúan **eagerly** (inmediatamente):
cada operación crea una lista intermedia. Para colecciones grandes,
`Sequence` evalúa **lazily** (solo cuando se necesita el resultado):

```kotlin
fun main() {
    val numeros = (1..1_000_000).toList()

    // CON List — crea 3 listas intermedias (cara en memoria)
    val resultadoLista = numeros
        .filter { it % 2 == 0 }          // crea lista de 500_000 elementos
        .map { it * it }                  // crea otra lista de 500_000 elementos
        .take(5)                          // por fin toma 5
    println(resultadoLista)

    // CON Sequence — procesa elemento por elemento, se detiene al tener 5
    val resultadoSeq = numeros
        .asSequence()                     // convierte a Sequence
        .filter { it % 2 == 0 }           // lazy — no ejecuta nada aún
        .map { it * it }                  // lazy — no ejecuta nada aún
        .take(5)                          // lazy — no ejecuta nada aún
        .toList()                         // AQUÍ se evalúa todo — solo procesa ~10 elementos
    println(resultadoSeq)  // [4, 16, 36, 64, 100]

    // generateSequence — secuencias infinitas
    val fibonacci = generateSequence(Pair(0L, 1L)) { (a, b) -> Pair(b, a + b) }
        .map { it.first }
        .take(10)
        .toList()
    println(fibonacci)  // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

    // Números primos perezosamente
    val primos = generateSequence(2) { n ->
        var siguiente = n + 1
        while ((2 until siguiente).any { siguiente % it == 0 }) siguiente++
        siguiente
    }.take(10).toList()
    println(primos)  // [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
}
```

### Cuándo usar `Sequence`

```
Usa List cuando...                 Usa Sequence cuando...
─────────────────                  ──────────────────────────────
Colección pequeña (< 1000 elem)    Colección grande o potencialmente infinita
Acceso aleatorio por índice        Procesamiento de un extremo al otro
Pocas operaciones encadenadas      Muchas operaciones encadenadas
Necesitas el resultado completo    Solo necesitas los primeros N elementos
```

---

## Programa completo — análisis de ventas

```kotlin
data class Venta(
    val id:        Int,
    val producto:  String,
    val categoria: String,
    val cantidad:  Int,
    val precio:    Double,
    val vendedor:  String
) {
    val total: Double get() = cantidad * precio
}

fun analizarVentas(ventas: List<Venta>) {
    println("=== ANÁLISIS DE VENTAS ===\n")

    // Total general
    val totalGeneral = ventas.sumOf { it.total }
    println("Total facturado: $${"%.2f".format(totalGeneral)}")
    println("Número de ventas: ${ventas.size}\n")

    // Por categoría
    println("── Por categoría ──")
    ventas
        .groupBy { it.categoria }
        .mapValues { (_, ventas) -> ventas.sumOf { it.total } }
        .entries
        .sortedByDescending { it.value }
        .forEach { (cat, total) ->
            println("  $cat: $${"%.2f".format(total)}")
        }

    // Top 3 productos
    println("\n── Top 3 productos ──")
    ventas
        .groupBy { it.producto }
        .mapValues { (_, v) -> v.sumOf { it.total } }
        .entries
        .sortedByDescending { it.value }
        .take(3)
        .forEachIndexed { i, (producto, total) ->
            println("  ${i + 1}. $producto: $${"%.2f".format(total)}")
        }

    // Rendimiento por vendedor
    println("\n── Por vendedor ──")
    ventas
        .groupBy { it.vendedor }
        .mapValues { (_, v) ->
            mapOf(
                "total"  to v.sumOf { it.total },
                "ventas" to v.size.toDouble()
            )
        }
        .entries
        .sortedByDescending { it.value["total"] }
        .forEach { (vendedor, stats) ->
            println("  $vendedor: $${"%.2f".format(stats["total"])} " +
                    "(${stats["ventas"]!!.toInt()} ventas)")
        }
}

fun main() {
    val ventas = listOf(
        Venta(1, "Teclado",     "Periféricos", 3, 89.99,  "Ana"),
        Venta(2, "Monitor",     "Pantallas",   1, 349.99, "Luis"),
        Venta(3, "Mouse",       "Periféricos", 5, 29.99,  "Ana"),
        Venta(4, "Auriculares", "Audio",       2, 149.99, "María"),
        Venta(5, "Teclado",     "Periféricos", 2, 89.99,  "Luis"),
        Venta(6, "Webcam",      "Cámaras",     1, 59.99,  "María"),
        Venta(7, "Monitor",     "Pantallas",   2, 349.99, "Ana"),
        Venta(8, "Mouse",       "Periféricos", 3, 29.99,  "Luis"),
    )

    analizarVentas(ventas)
}
```

---

## Ejercicios propuestos

1. **Estadísticas de texto** — Lee una cadena larga y calcula:
   frecuencia de cada carácter (con `groupBy`), las 5 palabras más
   repetidas (con `groupBy` + `sortedByDescending` + `take`),
   y el porcentaje de vocales sobre el total de letras.

2. **Agenda de contactos** — Modela `Contacto(nombre, email, telefono, etiquetas)`
   donde `etiquetas` es un `Set<String>`. Implementa:
   buscar por nombre parcial (`filter`), filtrar por etiqueta (`any`),
   agrupar por primera letra (`groupBy`), y exportar a `Map<String, String>`
   de nombre → email (`associate`).

3. **Pipeline de pedidos** — Dado un `List<Pedido>` (del ejemplo anterior),
   usa una `Sequence` para encontrar los primeros 3 pedidos pagados que
   superen $100, ordenados por total descendente, sin crear listas intermedias.

4. **`generateSequence` collatz** — La secuencia de Collatz empieza en N:
   si es par divide entre 2, si es impar multiplica por 3 y suma 1, repite
   hasta llegar a 1. Usa `generateSequence` para generar la secuencia completa
   de cualquier N y encuentra el N entre 1 y 100 que genera la secuencia más larga.

---

## Resumen de la página 7

- Kotlin distingue colecciones **inmutables** (`List`, `Set`, `Map`) de **mutables** (`MutableList`, etc.) — usa la inmutable por defecto.
- `Set` ignora duplicados y sus operaciones `union`, `intersect`, `subtract` modelan álgebra de conjuntos.
- `Map` almacena pares clave→valor. `getOrDefault` y `getOrPut` evitan los null checks manuales.
- `map` transforma, `filter` filtra, `reduce`/`fold` agregan, `groupBy` agrupa en un `Map<K, List<V>>`.
- `flatMap` es `map` + `flatten` en un paso — útil para listas de listas.
- Las operaciones encadenadas crean **pipelines declarativos** que expresan el qué sin detallar el cómo.
- `Sequence` evalúa perezosamente — ideal para colecciones grandes o infinitas y pipelines con `take(N)`.
- `generateSequence` crea secuencias potencialmente infinitas procesadas bajo demanda.

---

> **Siguiente página →** Página 8: Funciones de orden superior, lambdas,
> genéricos y `typealias`.