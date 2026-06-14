
# Curso Kotlin — Página 6
## Módulo 2 · Programación Orientada a Objetos
### Herencia, polimorfismo, interfaces y `sealed class`

---

## Los principios que cubrimos aquí

En la página 5 vimos **Abstracción** y **Encapsulamiento** — cómo
diseñar una clase individualmente bien construida.

Esta página cubre los principios que conectan clases entre sí:

| Principio | Qué permite | Mecanismo principal |
|---|---|---|
| **Herencia** | Una clase reutiliza y especializa el comportamiento de otra | `open class`, `: ClasePadre()`, `override` |
| **Polimorfismo** | Un mismo código funciona con diferentes tipos | `override`, `interface`, `sealed class`, `is` |

---

## Principio 3 — Herencia

> **"Una clase puede reutilizar y especializar el comportamiento de otra,
> sin repetir código."**

En Kotlin todas las clases son **`final` por defecto** — no se pueden
heredar a menos que se marquen con `open`. Esto obliga a diseñar
la jerarquía de forma consciente:

```kotlin
// Sin open — no se puede heredar (protección por defecto)
class Animal(val nombre: String)
// class Perro : Animal("Rex")  // ERROR — Animal es final

// Con open — la jerarquía está diseñada para ello
open class Animal(val nombre: String, val sonido: String) {
    // open — la subclase PUEDE sobreescribir
    open fun hacerSonido() = println("$nombre dice: $sonido")
    open fun descripcion() = "Soy $nombre"

    // Sin open — la subclase NO puede sobreescribir
    fun respirar() = println("$nombre respira")
}

// HERENCIA: Perro reutiliza todo de Animal y especializa hacerSonido
class Perro(nombre: String) : Animal(nombre, "Guau") {
    override fun hacerSonido() {
        super.hacerSonido()          // reutiliza la implementación del padre
        println("(mueve la cola)")   // añade comportamiento propio
    }
    override fun descripcion() = "${super.descripcion()}, un perro"
}

class Gato(nombre: String, val interior: Boolean) : Animal(nombre, "Miau") {
    override fun descripcion() =
        "${super.descripcion()}, un gato ${if (interior) "de interior" else "callejero"}"
}

fun main() {
    val perro = Perro("Rex")
    perro.hacerSonido()
    // Rex dice: Guau
    // (mueve la cola)

    val gato = Gato("Misi", true)
    println(gato.descripcion())  // Soy Misi, un gato de interior

    // Herencia — Perro y Gato tienen todo lo de Animal más lo propio
    perro.respirar()  // Rex respira — heredado de Animal
}
```

### `override` y `final override`

```kotlin
open class Vehiculo(val marca: String) {
    open fun arrancar() = println("$marca: arrancando...")
}

open class Coche(marca: String, val modelo: String) : Vehiculo(marca) {
    // final override — sobreescribe aquí PERO bloquea más abajo
    final override fun arrancar() = println("$marca $modelo: motor encendido 🔑")
}

class CocheElectrico(marca: String, modelo: String) : Coche(marca, modelo) {
    // override fun arrancar() = ...  // ERROR — arrancar es final en Coche
}
```

---

## Principio 4 — Polimorfismo

> **"El mismo código puede trabajar con objetos de distintos tipos,
> ejecutando el comportamiento correcto para cada uno."**

El polimorfismo tiene dos formas en Kotlin:

### Polimorfismo por herencia — `override`

```kotlin
// Una sola lista, tres tipos distintos, tres comportamientos distintos
val animales: List<Animal> = listOf(
    Perro("Rex"),
    Gato("Misi", true),
    Gato("Rayos", false)
)

// forEach llama el método correcto según el tipo real de cada objeto
animales.forEach { animal ->
    animal.hacerSonido()   // Polimorfismo: cada animal hace SU sonido
}
// Rex dice: Guau / (mueve la cola)
// Misi dice: Miau
// Rayos dice: Miau
```

### Polimorfismo por interfaz

```kotlin
// La interfaz define el contrato — QUÉ puede hacer
// Las implementaciones definen el CÓMO
interface Pagable {
    fun procesar(monto: Double): Boolean
    val nombre: String
}

class TarjetaCredito(val numero: String) : Pagable {
    override val nombre = "Tarjeta de crédito"
    override fun procesar(monto: Double): Boolean {
        println("💳 Cargando $${"%.2f".format(monto)} a $numero")
        return true
    }
}

class PayPal(val email: String) : Pagable {
    override val nombre = "PayPal"
    override fun procesar(monto: Double): Boolean {
        println("🅿️ Enviando $${"%.2f".format(monto)} a $email")
        return true
    }
}

class Efectivo : Pagable {
    override val nombre = "Efectivo"
    override fun procesar(monto: Double): Boolean {
        println("💵 Recibiendo $${"%.2f".format(monto)} en efectivo")
        return true
    }
}

// Esta función no sabe ni le importa qué tipo de pago es
// Solo sabe que recibe algo que implementa Pagable — POLIMORFISMO
fun cobrar(monto: Double, metodoPago: Pagable) {
    println("Procesando pago con ${metodoPago.nombre}...")
    val exito = metodoPago.procesar(monto)
    println(if (exito) "✅ Pago exitoso" else "❌ Pago fallido")
}

fun main() {
    val metodos: List<Pagable> = listOf(
        TarjetaCredito("**** **** **** 1234"),
        PayPal("ana@test.com"),
        Efectivo()
    )

    // Misma función — comportamiento distinto según el tipo
    metodos.forEach { cobrar(99.99, it) }
}
```

---

## Clases `abstract` — herencia obligatoria

Una clase `abstract` aplica **ambos principios juntos**:
- **Abstracción**: define la forma sin implementar el detalle
- **Herencia**: las subclases completan lo que falta

```kotlin
abstract class Figura(val nombre: String) {
    // abstract — las subclases DEBEN implementar esto (herencia forzada)
    abstract val area: Double
    abstract val perimetro: Double
    abstract fun descripcion(): String

    // concreto — disponible en todas las subclases (reutilización)
    fun comparar(otra: Figura): String = when {
        area > otra.area -> "$nombre es más grande que ${otra.nombre}"
        area < otra.area -> "$nombre es más pequeña que ${otra.nombre}"
        else             -> "$nombre y ${otra.nombre} tienen la misma área"
    }

    // Polimorfismo: toString usa area y descripcion que son polimórficas
    override fun toString() = "${descripcion()} | Área: ${"%.2f".format(area)}"
}

class Circulo(val radio: Double) : Figura("Círculo") {
    override val area:       Double get() = Math.PI * radio * radio
    override val perimetro:  Double get() = 2 * Math.PI * radio
    override fun descripcion() = "Círculo de radio $radio"
}

class Rectangulo(val ancho: Double, val alto: Double) : Figura("Rectángulo") {
    override val area:       Double get() = ancho * alto
    override val perimetro:  Double get() = 2 * (ancho + alto)
    override fun descripcion() = "Rectángulo de ${ancho}x${alto}"
}

class TrianguloEquilatero(val lado: Double) : Figura("Triángulo") {
    override val area:       Double get() = (Math.sqrt(3.0) / 4) * lado * lado
    override val perimetro:  Double get() = 3 * lado
    override fun descripcion() = "Triángulo equilátero de lado $lado"
}

fun main() {
    // POLIMORFISMO: la lista acepta cualquier Figura
    val figuras: List<Figura> = listOf(
        Circulo(5.0),
        Rectangulo(4.0, 6.0),
        TrianguloEquilatero(8.0)
    )

    figuras.forEach { println(it) }  // toString polimórfico

    val mayor = figuras.maxByOrNull { it.area }
    println("\nFigura más grande: ${mayor?.nombre}")

    println(figuras[0].comparar(figuras[1]))
}
```

---

## Interfaces con implementación por defecto

Las interfaces en Kotlin pueden tener **implementaciones por defecto**
y propiedades. Una clase puede implementar múltiples interfaces:

```kotlin
interface Serializable {
    val id: String                    // abstracta — debe implementarse
    fun serializar(): String          // abstracta — debe implementarse
    val version: Int get() = 1        // con default — puede sobreescribirse
}

interface Validable {
    val errores: List<String>
    val esValido: Boolean get() = errores.isEmpty()

    fun validar(): Boolean
    fun imprimirErrores() {                // implementación por defecto
        if (errores.isEmpty()) println("Sin errores")
        else errores.forEach { println("  ❌ $it") }
    }
}

// POLIMORFISMO: Pedido puede usarse donde se espere Serializable O Validable
data class Pedido(
    override val id: String,
    val cliente:     String,
    val items:       List<String>,
    val total:       Double
) : Serializable, Validable {

    override fun serializar() =
        "$id|$cliente|${items.joinToString(",")}|$total"

    override val errores: List<String> get() = buildList {
        if (cliente.isBlank()) add("El cliente no puede estar vacío")
        if (items.isEmpty())   add("El pedido debe tener al menos un item")
        if (total <= 0)        add("El total debe ser mayor que cero")
    }

    override fun validar() = esValido
}

fun main() {
    val pedido1 = Pedido("P001", "Ana", listOf("Teclado", "Mouse"), 119.98)
    val pedido2 = Pedido("P002", "",    emptyList(),                -5.0)

    // Polimorfismo por interfaz
    fun procesarSerializable(s: Serializable) = println("→ ${s.serializar()}")
    fun procesarValidable(v: Validable) {
        println("Válido: ${v.esValido}")
        v.imprimirErrores()
    }

    procesarSerializable(pedido1)   // → P001|Ana|Teclado,Mouse|119.98
    procesarValidable(pedido1)      // Válido: true / Sin errores
    procesarValidable(pedido2)      // Válido: false / ❌ ...
}
```

### Resolución de conflictos entre interfaces

```kotlin
interface A { fun saludar() = println("Hola desde A") }
interface B { fun saludar() = println("Hola desde B") }

class C : A, B {
    // Obligatorio cuando ambas tienen implementación del mismo método
    override fun saludar() {
        super<A>.saludar()
        super<B>.saludar()
        println("Y desde C")
    }
}
```

---

## `sealed class` — polimorfismo cerrado y seguro

Una `sealed class` es **polimorfismo controlado**: el compilador conoce
exactamente todos los tipos posibles, lo que permite un `when` exhaustivo
sin `else` y detección de casos no manejados en tiempo de compilación:

```kotlin
// Sin sealed — si alguien añade un subclase, el when podría quedar incompleto
// Con sealed — el compilador avisa si falta una rama en el when

sealed class Resultado<out T> {
    data class Exito<T>(val datos: T)       : Resultado<T>()
    data class Error(val mensaje: String,
                     val codigo: Int = 0)   : Resultado<Nothing>()
    object Cargando                         : Resultado<Nothing>()
}

fun obtenerUsuario(id: Int): Resultado<String> = when (id) {
    0    -> Resultado.Error("ID inválido", 400)
    1    -> Resultado.Exito("Ana García")
    else -> Resultado.Cargando
}

fun procesarResultado(resultado: Resultado<String>) {
    // POLIMORFISMO + EXHAUSTIVIDAD: sin else — el compilador garantiza que
    // todos los casos están cubiertos. Si añades una subclase, el compilador
    // avisa en todos los when que no la tengan
    when (resultado) {
        is Resultado.Exito    -> println("✅ Usuario: ${resultado.datos}")
        is Resultado.Error    -> println("❌ Error ${resultado.codigo}: ${resultado.mensaje}")
        is Resultado.Cargando -> println("⏳ Cargando...")
    }
}

fun main() {
    for (id in 0..2) {
        procesarResultado(obtenerUsuario(id))
    }
    // ❌ Error 400: ID inválido
    // ✅ Usuario: Ana García
    // ⏳ Cargando...
}
```

### `sealed class` con estados ricos

```kotlin
sealed class EstadoPedido {
    object Nuevo                                          : EstadoPedido()
    data class EnProceso(val operador: String)            : EstadoPedido()
    data class Enviado(val tracking: String,
                       val empresa: String)               : EstadoPedido()
    data class Entregado(val fecha: String)               : EstadoPedido()
    data class Cancelado(val motivo: String,
                         val reembolso: Boolean = false)  : EstadoPedido()
}

// POLIMORFISMO: una sola función maneja todos los estados posibles
fun descripcionEstado(estado: EstadoPedido): String = when (estado) {
    is EstadoPedido.Nuevo      -> "Pedido recibido"
    is EstadoPedido.EnProceso  -> "Procesado por ${estado.operador}"
    is EstadoPedido.Enviado    -> "Enviado con ${estado.empresa} — ${estado.tracking}"
    is EstadoPedido.Entregado  -> "Entregado el ${estado.fecha}"
    is EstadoPedido.Cancelado  -> "Cancelado: ${estado.motivo}" +
                                  if (estado.reembolso) " (con reembolso)" else ""
}

fun main() {
    listOf(
        EstadoPedido.Nuevo,
        EstadoPedido.EnProceso("Ana"),
        EstadoPedido.Enviado("ES123456789", "Correos"),
        EstadoPedido.Cancelado("Sin stock", reembolso = true)
    ).forEach { println(descripcionEstado(it)) }
}
```

---

## Delegación — composición sobre herencia

Kotlin favorece la **composición** como alternativa a la herencia
cuando se quiere reutilizar comportamiento sin crear jerarquías rígidas:

```kotlin
interface Imprimible {
    fun imprimir()
}

class ImpresoraRapida : Imprimible {
    override fun imprimir() = println("Imprimiendo a 300ppm ⚡")
}

class ImpresoraEco : Imprimible {
    override fun imprimir() = println("Imprimiendo en modo ahorro 🍃")
}

// HERENCIA vs COMPOSICIÓN:
// DocumentoOficial NO hereda de Impresora — delega con 'by'
// Ventaja: puede cambiar la impresora en tiempo de ejecución
class DocumentoOficial(
    val titulo:    String,
    val impresora: Imprimible
) : Imprimible by impresora {   // delega la interfaz al objeto recibido

    fun imprimirCopias(n: Int) {
        println("=== $titulo ($n copias) ===")
        repeat(n) { imprimir() }  // usa la implementación delegada
    }
}

fun main() {
    val docRapido = DocumentoOficial("Contrato", ImpresoraRapida())
    val docEco    = DocumentoOficial("Factura",  ImpresoraEco())

    docRapido.imprimirCopias(2)
    docEco.imprimirCopias(1)
}
```

---

## Programa completo — sistema de notificaciones

Todos los principios OOP juntos:

```kotlin
// ABSTRACCIÓN: sealed class define los tipos posibles de notificación
sealed class Notificacion(val titulo: String, val mensaje: String) {
    abstract fun formatear(): String  // cada tipo formatea de forma distinta

    data class Email(
        val destinatario: String,
        val asunto:       String,
        val cuerpo:       String
    ) : Notificacion(asunto, cuerpo) {
        override fun formatear() =
            "📧 Email → $destinatario\n   Asunto: $titulo\n   ${mensaje.take(50)}..."
    }

    data class Push(val dispositivo: String, val icono: String = "🔔")
        : Notificacion("Push", "") {
        override fun formatear() = "$icono Push → $dispositivo: $titulo"
    }

    data class Sms(val telefono: String, val texto: String)
        : Notificacion("SMS", texto) {
        override fun formatear() = "📱 SMS → $telefono: ${texto.take(160)}"
    }

    object Silenciosa : Notificacion("", "") {
        override fun formatear() = "🔕 Notificación silenciosa"
    }
}

// ABSTRACCIÓN + POLIMORFISMO: interfaz con contrato genérico
interface EnviadorNotificacion {
    val nombre: String
    fun enviar(notificacion: Notificacion): Boolean
}

// HERENCIA: implementaciones concretas del mismo contrato
class ServicioEmail : EnviadorNotificacion {
    override val nombre = "Email"
    override fun enviar(n: Notificacion): Boolean {
        if (n !is Notificacion.Email) return false
        println("  [EMAIL] → ${n.destinatario}")
        return true
    }
}

class ServicioPush : EnviadorNotificacion {
    override val nombre = "Push"
    override fun enviar(n: Notificacion): Boolean {
        if (n !is Notificacion.Push) return false
        println("  [PUSH] → ${n.dispositivo}")
        return true
    }
}

// ENCAPSULAMIENTO: la lista de servicios es privada
class Dispatcher(private val servicios: List<EnviadorNotificacion>) {

    fun enviar(notificacion: Notificacion) {
        println(notificacion.formatear())  // POLIMORFISMO: cada tipo formatea distinto
        val exito = servicios.any { it.enviar(notificacion) }
        if (!exito) println("  ⚠️ Sin servicio disponible")
        println()
    }
}

fun main() {
    val dispatcher = Dispatcher(listOf(ServicioEmail(), ServicioPush()))

    listOf(
        Notificacion.Email("ana@test.com", "Bienvenida", "Gracias por registrarte."),
        Notificacion.Push("iPhone-Ana"),
        Notificacion.Sms("+34600000000", "Tu código es 1234"),
        Notificacion.Silenciosa
    ).forEach { dispatcher.enviar(it) }
}
```

---

## Los 4 principios — resumen visual

```
┌─────────────────────────────────────────────────────────────────┐
│  ABSTRACCIÓN          │  ENCAPSULAMIENTO                        │
│  abstract class       │  private / protected / internal         │
│  interface            │  getter/setter personalizados           │
│  data class           │  constructor privado + factory          │
│  sealed class         │  object / companion object              │
├─────────────────────────────────────────────────────────────────┤
│  HERENCIA             │  POLIMORFISMO                           │
│  open class           │  override                               │
│  : ClasePadre()       │  List<Animal> con Perro, Gato...        │
│  abstract class       │  fun cobrar(p: Pagable) — cualquier impl│
│  super.metodo()       │  when (sealed) — exhaustivo             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Ejercicios propuestos

1. **Los 4 principios juntos** — Diseña un sistema de vehículos:
   - `abstract class Vehiculo` con `abstract val velocidadMax`, `abstract fun mover()`
   - Subclases `Coche`, `Moto`, `Bicicleta`
   - Interfaz `Electrico` con `fun cargar()`
   - `CocheElectrico` que hereda de `Coche` e implementa `Electrico`
   - Lista `List<Vehiculo>` con todos y un `forEach { it.mover() }` polimórfico

2. **`sealed class` Result** — Implementa `Result<T>` con `Success<T>`,
   `Failure(val exception: Exception)` y `Loading`. Escribe `dividirSeguro(a, b)`.
   Usa `when` exhaustivo en todos los puntos de consumo.

3. **Interfaz `Comparable` manual** — Crea `interface Ordenable<T>` con
   `fun compararCon(otro: T): Int`. Impleméntala en `Temperatura` y `Producto`
   (por precio). Úsalas en `sortedWith` pasando el comparador polimórfico.

4. **Delegación vs herencia** — Crea `interface Logger` con `fun log(msg: String)`.
   Implementa `ConsoleLogger` y `FileLogger`. Crea `ServicioUsuarios` que
   delega `Logger` con `by` y puede cambiar el logger en el constructor.

---

## Resumen de la página 6

- **Herencia**: las clases son `final` por defecto — `open` marca explícitamente qué puede heredarse. `override` es obligatorio. `super` llama la implementación del padre.
- **Polimorfismo**: el mismo código trabaja con distintos tipos. Se logra por herencia (`override`) o por interfaz — `fun cobrar(p: Pagable)` funciona con `TarjetaCredito`, `PayPal` y `Efectivo`.
- Las clases `abstract` combinan herencia obligatoria (las subclases completan los miembros `abstract`) con reutilización (los métodos concretos están disponibles en todas).
- Las interfaces pueden tener implementaciones por defecto. Una clase puede implementar múltiples interfaces. Los conflictos se resuelven con `super<Interfaz>.metodo()`.
- `sealed class` es polimorfismo cerrado — el `when` puede ser exhaustivo sin `else`. Si añades una subclase, el compilador avisa en todos los `when` incompletos.
- La delegación con `by` es la alternativa de Kotlin a la herencia rígida — composición con polimorfismo sin acoplamiento de jerarquía.

---

> **Siguiente página →** Página 7: Colecciones — `List`, `Set`, `Map`,
> operaciones funcionales y `Sequence`.