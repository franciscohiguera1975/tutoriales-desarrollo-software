# Curso Kotlin — Página 11
## Módulo 6 · Jetpack Compose
### Fundamentos: composables, estado, recomposición y primeros componentes

---

## ¿Qué es Jetpack Compose?

Jetpack Compose es el toolkit moderno de Android para construir interfaces
de usuario de forma **declarativa** con Kotlin. Reemplaza al sistema de
Views XML que existía desde los inicios de Android.

La diferencia fundamental está en el **modelo mental**:

```
XML (imperativo)                     Compose (declarativo)
──────────────────────────────────   ──────────────────────────────────────
Describes CÓMO cambiar la UI         Describes CÓMO SE VE la UI dado el estado
activity.tv.text = "Hola"            Text("Hola")
if (cargando) spinner.show()         if (cargando) CircularProgressIndicator()
RecyclerView + Adapter               LazyColumn { items(lista) { ... } }
```

En Compose **la UI es una función del estado**:

```
UI = f(estado)
```

Cuando el estado cambia, Compose llama de nuevo esa función y actualiza
**solo las partes afectadas** — esto se llama **recomposición**.

### ¿Por qué Compose reemplaza a XML?

| Aspecto | Views XML | Jetpack Compose |
|---|---|---|
| Lenguaje | XML + Kotlin/Java separados | Solo Kotlin |
| Actualización de UI | Manual (`setText`, `setVisibility`) | Automática al cambiar estado |
| Reutilización | Fragments, Custom Views (verboso) | Funciones `@Composable` (simple) |
| Preview en IDE | Limitado | Instantáneo con `@Preview` |
| Animaciones | API compleja | `AnimatedVisibility`, `animate*AsState` |
| Testing | Espresso (lento) | Compose Testing (rápido) |

---

## Configuración del proyecto

Crea un proyecto nuevo en Android Studio eligiendo **"Empty Activity"**
(no "Empty Views Activity"). Android Studio configura Compose automáticamente.
Si trabajas sobre un proyecto existente, agrega estas dependencias:

```kotlin
// build.gradle.kts (módulo app)
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")  // plugin del compilador Compose
}

android {
    buildFeatures { compose = true }
}

dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2026.03.00")
    implementation(composeBom)

    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.activity:activity-compose:1.10.0")

    debugImplementation("androidx.compose.ui:ui-tooling")
}
```

> **BOM (Bill of Materials):** la línea `compose-bom` garantiza que todas
> las librerías de Compose usen versiones compatibles entre sí. No necesitas
> especificar la versión de cada una individualmente.

---

## Estructura básica: `MainActivity.kt` y el primer composable

Cuando creas un proyecto "Empty Activity", Android Studio genera este código.
Analicemos cada parte:

```kotlin
// MainActivity.kt
package com.tuapp.compose

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent          // ← extensión de Compose
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.tooling.preview.Preview

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // setContent reemplaza a setContentView(R.layout.activity_main)
        // Todo lo que esté dentro es Compose — no hay XML
        setContent {
            MaterialTheme {          // aplica el tema (colores, tipografía, formas)
                MiApp()              // composable raíz — aquí arranca el árbol de UI
            }
        }
    }
}

@Composable                          // ← esta anotación lo convierte en composable
fun MiApp() {
    Text("¡Hola, Compose!")          // dibuja texto en pantalla
}

@Preview(showBackground = true)      // ← visualiza en el editor sin ejecutar
@Composable
fun MiAppPreview() {
    MaterialTheme {
        MiApp()
    }
}
```

### ¿Qué hace cada parte?

**`setContent { }`** — reemplaza completamente a `setContentView()`. Todo el
árbol de UI de la pantalla vive dentro de este bloque.

**`MaterialTheme { }`** — aplica el sistema de diseño Material 3: colores,
tipografía y formas. Los composables internos pueden acceder a
`MaterialTheme.colorScheme`, `MaterialTheme.typography`, etc.

**`@Composable`** — anotación que transforma una función Kotlin normal en una
función que puede *dibujar* UI. Una función `@Composable`:
- No devuelve un valor (retorna `Unit`)
- Solo puede ser llamada desde otra función `@Composable`
- Puede llamar a otras funciones `@Composable`

**`@Preview`** — le dice al IDE que renderice esta función en el panel de
diseño. No afecta la app en producción.

---

## `@Composable` — la unidad básica

```kotlin
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable

// La función más simple posible: recibe datos, muestra UI
@Composable
fun Saludo(nombre: String) {
    Text(text = "Hola, $nombre!")
}
```

### Reglas fundamentales de los composables

```kotlin
// ✅ Correcto — composable llama a otros composables
@Composable
fun PantallaInicio() {
    Column {
        Saludo("Ana")
        Saludo("Luis")
    }
}

// ❌ Incorrecto — NO puedes llamar composables desde funciones normales
fun funcionNormal() {
    // Text("Error")   // ERROR DE COMPILACIÓN
}

// ✅ Correcto — los composables pueden tener lógica Kotlin normal
@Composable
fun MensajeCondicional(mostrar: Boolean) {
    if (mostrar) {
        Text("Visible")
    }
    // Si mostrar = false, no dibuja nada — no es necesario el else
}

// ✅ Correcto — composable con múltiples parámetros y valores por defecto
@Composable
fun Etiqueta(
    texto:    String,
    negrita:  Boolean = false,      // parámetro opcional
    visible:  Boolean = true
) {
    if (visible) {
        Text(
            text       = texto,
            fontWeight = if (negrita) FontWeight.Bold else FontWeight.Normal
        )
    }
}
```

> **Principio clave:** los composables deben ser **puros** — dado el mismo
> estado, siempre producen la misma UI. No deben tener efectos secundarios
> directos fuera de los efectos controlados de Compose.

---

## Ejemplos directos en `MainActivity`

Antes de organizar el código en múltiples archivos, practicamos los
conceptos directamente en `MainActivity`. Cada ejemplo reemplaza
completamente el contenido del archivo — lee el código, ejecútalo,
modifica valores y observa el resultado.

---

### Ejemplo 1 — Hello World interactivo

**Concepto:** primer composable, `Text`, `Button`, `Column` y estado básico
con `remember` y `mutableStateOf`.

Reemplaza todo el contenido de `MainActivity.kt`:

```kotlin
// MainActivity.kt — Ejemplo 1: Hello World interactivo
package com.tuapp.compose

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                EjemploHelloWorld()
            }
        }
    }
}

@Composable
fun EjemploHelloWorld() {
    // 'by remember' delega la lectura/escritura al objeto MutableState
    // 'mutableStateOf' crea estado observable: cuando cambia, Compose recompone
    var mensaje  by remember { mutableStateOf("¡Hola, Compose!") }
    var contador by remember { mutableStateOf(0) }

    // Column apila sus hijos verticalmente
    Column(
        modifier            = Modifier
            .fillMaxSize()          // ocupa todo el ancho y alto disponible
            .padding(24.dp),        // margen interior de 24dp en todos los lados
        verticalArrangement = Arrangement.Center,           // centra verticalmente
        horizontalAlignment = Alignment.CenterHorizontally  // centra horizontalmente
    ) {
        // Text muestra una cadena de texto en pantalla
        Text(
            text  = mensaje,
            style = MaterialTheme.typography.headlineMedium
        )

        Spacer(Modifier.height(16.dp))   // espacio vacío vertical

        Text(
            text  = "Botón presionado: $contador veces",
            style = MaterialTheme.typography.bodyLarge
        )

        Spacer(Modifier.height(24.dp))

        // Button ejecuta 'onClick' cuando el usuario lo presiona
        Button(onClick = {
            contador++
            mensaje = "¡Presionado $contador ${if (contador == 1) "vez" else "veces"}!"
        }) {
            Text("Presióname")
        }

        Spacer(Modifier.height(8.dp))

        // OutlinedButton: variante con borde, sin relleno
        OutlinedButton(onClick = {
            contador = 0
            mensaje  = "¡Hola, Compose!"
        }) {
            Text("Reiniciar")
        }
    }
}

@Preview(showBackground = true)
@Composable
fun EjemploHelloWorldPreview() {
    MaterialTheme { EjemploHelloWorld() }
}
```

**▶ Ejecuta y observa:**
- Presiona el botón varias veces → el texto cambia en tiempo real
- Presiona Reiniciar → vuelve al estado inicial
- Modifica `"¡Hola, Compose!"` en el código → el `@Preview` se actualiza sin ejecutar

**Experimenta:**
- Cambia `Arrangement.Center` por `Arrangement.Top` → ¿qué pasa?
- Cambia `fillMaxSize()` por `fillMaxWidth()` → ¿cuál es la diferencia?
- Agrega un tercer botón que ponga `mensaje` en mayúsculas con `.uppercase()`

---

### Ejemplo 2 — Layouts y Modifier en acción

**Concepto:** `Column`, `Row`, `Box`, `Modifier` y cómo el orden de la
cadena de modificadores importa. Sin estado aún — solo estructura visual.

Reemplaza `MainActivity.kt`:

```kotlin
// MainActivity.kt — Ejemplo 2: Layouts y Modifier
package com.tuapp.compose

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                EjemploLayouts()
            }
        }
    }
}

@Composable
fun EjemploLayouts() {
    Column(
        modifier            = Modifier
            .fillMaxSize()
            .padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(20.dp)  // espacio uniforme entre hijos
    ) {
        Text("Ejemplo 2 · Layouts y Modifier",
             style = MaterialTheme.typography.titleLarge)

        // ── COLUMN ─────────────────────────────────────────────────────────
        Text("Column — apila verticalmente:",
             style = MaterialTheme.typography.labelLarge,
             color = MaterialTheme.colorScheme.primary)

        Column(
            modifier            = Modifier
                .fillMaxWidth()
                .border(1.dp, Color.Gray, RoundedCornerShape(8.dp))
                .padding(8.dp),
            verticalArrangement = Arrangement.spacedBy(4.dp)
        ) {
            CeldaColoreada("Elemento 1", Color(0xFF90CAF9))
            CeldaColoreada("Elemento 2", Color(0xFF64B5F6))
            CeldaColoreada("Elemento 3", Color(0xFF42A5F5))
        }

        // ── ROW ────────────────────────────────────────────────────────────
        Text("Row — alinea horizontalmente (weight 1:2:1):",
             style = MaterialTheme.typography.labelLarge,
             color = MaterialTheme.colorScheme.primary)

        // weight distribuye el espacio disponible en proporciones
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .height(48.dp)
                .clip(RoundedCornerShape(8.dp))
        ) {
            Box(Modifier.weight(1f).fillMaxHeight().background(Color(0xFFCE93D8)),
                contentAlignment = Alignment.Center) { Text("1") }
            Box(Modifier.weight(2f).fillMaxHeight().background(Color(0xFFBA68C8)),
                contentAlignment = Alignment.Center) { Text("2") }
            Box(Modifier.weight(1f).fillMaxHeight().background(Color(0xFFAB47BC)),
                contentAlignment = Alignment.Center) { Text("1") }
        }

        // ── BOX ────────────────────────────────────────────────────────────
        Text("Box — superpone en capas (z-order):",
             style = MaterialTheme.typography.labelLarge,
             color = MaterialTheme.colorScheme.primary)

        Box(
            modifier         = Modifier
                .fillMaxWidth()
                .height(80.dp)
                .clip(RoundedCornerShape(12.dp))
                .background(Color(0xFF1565C0)),      // capa de fondo azul
            contentAlignment = Alignment.Center
        ) {
            // Capa 2: caja pequeña en la esquina superior
            Box(
                modifier = Modifier
                    .size(40.dp)
                    .background(Color(0xFF42A5F5))
                    .align(Alignment.TopStart)
            )
            // Capa 3: texto sobre todo
            Text("Texto en capa superior", color = Color.White,
                 style = MaterialTheme.typography.labelLarge)
        }

        // ── MODIFIER: el orden importa ──────────────────────────────────────
        Text("Modifier — clip ANTES vs DESPUÉS de background:",
             style = MaterialTheme.typography.labelLarge,
             color = MaterialTheme.colorScheme.primary)

        Row(horizontalArrangement = Arrangement.spacedBy(16.dp)) {
            // ✅ clip antes de background → fondo respeta la forma
            Box(
                modifier = Modifier
                    .size(90.dp)
                    .clip(RoundedCornerShape(16.dp))                       // 1° recorta
                    .background(MaterialTheme.colorScheme.primaryContainer), // 2° pinta
                contentAlignment = Alignment.Center
            ) {
                Text("✅ clip\nantes",
                     style = MaterialTheme.typography.labelSmall)
            }

            // ❌ background antes de clip → fondo NO se recorta
            Box(
                modifier = Modifier
                    .size(90.dp)
                    .background(Color(0xFFFFCDD2))        // 1° pinta sin recorte
                    .clip(RoundedCornerShape(16.dp)),     // 2° tarde — el bg ya se pintó
                contentAlignment = Alignment.Center
            ) {
                Text("❌ bg\nantes",
                     style = MaterialTheme.typography.labelSmall)
            }
        }
    }
}

// Composable auxiliar — reutilizable dentro del mismo archivo
@Composable
private fun CeldaColoreada(texto: String, color: Color) {
    Box(
        modifier         = Modifier
            .fillMaxWidth()
            .height(36.dp)
            .clip(RoundedCornerShape(4.dp))
            .background(color),
        contentAlignment = Alignment.Center
    ) {
        Text(texto, style = MaterialTheme.typography.labelMedium)
    }
}

@Preview(showBackground = true)
@Composable
fun EjemploLayoutsPreview() {
    MaterialTheme { EjemploLayouts() }
}
```

**▶ Ejecuta y observa:**
- El `Box` azul tiene tres capas claramente visibles
- Compara las dos cajas de Modifier: la izquierda tiene esquinas redondeadas, la derecha no — mismo código, distinto orden

**Experimenta:**
- En el Row de weights, cambia `weight(1f)`, `weight(2f)`, `weight(1f)` por `weight(3f)`, `weight(1f)`, `weight(1f)` → observa la redistribución proporcional
- Agrega `.padding(8.dp)` antes de `.clip(...)` en la caja ✅ → ¿el clip sigue funcionando igual?
- Cambia `Arrangement.spacedBy(20.dp)` en el Column raíz por `Arrangement.SpaceEvenly`

---

### Ejemplo 3 — Estado y contadores

**Concepto:** `remember`, `mutableStateOf`, recomposición selectiva y
state hoisting — todo con componentes básicos, sin campos de texto.

Reemplaza `MainActivity.kt`:

```kotlin
// MainActivity.kt — Ejemplo 3: Estado y contadores
package com.tuapp.compose

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                EjemploEstado()
            }
        }
    }
}

@Composable
fun EjemploEstado() {
    Column(
        modifier            = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(24.dp)
    ) {
        Text("Ejemplo 3 · Estado y recomposición",
             style = MaterialTheme.typography.titleLarge)

        DemoContador()

        HorizontalDivider()

        DemoStateHoisting()
    }
}

// ── Demo 1: Contador — ciclo básico estado → UI → evento → estado ────────────
// remember: recuerda el valor entre recomposiciones
// mutableStateOf: cuando cambia, Compose recompone los composables que lo leen
@Composable
private fun DemoContador() {
    var cuenta by remember { mutableStateOf(0) }

    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        Text("Contador básico",
             style = MaterialTheme.typography.labelLarge,
             color = MaterialTheme.colorScheme.primary)

        // Solo este Text se recompone cuando 'cuenta' cambia
        // — el resto del composable no se toca
        Text(
            text       = "$cuenta",
            style      = MaterialTheme.typography.displayLarge,
            fontWeight = FontWeight.Bold,
            color      = when {
                cuenta > 0 -> MaterialTheme.colorScheme.primary
                cuenta < 0 -> MaterialTheme.colorScheme.error
                else       -> MaterialTheme.colorScheme.onSurface
            }
        )

        // Barra de progreso visual — derivada del estado, sin estado propio
        val progreso = (cuenta.coerceIn(-10, 10) + 10) / 20f
        LinearProgressIndicator(
            progress = { progreso },
            modifier = Modifier.fillMaxWidth().height(8.dp)
                .clip(RoundedCornerShape(4.dp))
        )

        Text(
            text  = "Rango: −10 a +10",
            style = MaterialTheme.typography.labelSmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(
                onClick  = { if (cuenta > -10) cuenta-- },
                enabled  = cuenta > -10     // se deshabilita en el límite inferior
            ) { Text("−") }

            Button(
                onClick  = { if (cuenta < 10) cuenta++ },
                enabled  = cuenta < 10      // se deshabilita en el límite superior
            ) { Text("+") }

            OutlinedButton(onClick = { cuenta = 0 }) { Text("Reset") }
        }
    }
}

// ── Demo 2: State Hoisting — el estado sube al padre común ──────────────────
// Principio: el estado vive donde se necesita, no donde se muestra
@Composable
private fun DemoStateHoisting() {
    // El estado vive aquí — el padre puede usarlo para múltiples hijos
    var rojo   by remember { mutableStateOf(0) }
    var verde  by remember { mutableStateOf(0) }
    var azul   by remember { mutableStateOf(0) }

    Column(verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Text("State Hoisting — mezclador de colores",
             style = MaterialTheme.typography.labelLarge,
             color = MaterialTheme.colorScheme.primary)

        Text(
            "El padre tiene los tres estados y los combina para el color resultante.",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )

        // Caja de color resultante — usa los tres estados del padre
        Box(
            modifier         = Modifier
                .fillMaxWidth()
                .height(64.dp)
                .clip(RoundedCornerShape(12.dp))
                .background(Color(rojo, verde, azul)),
            contentAlignment = Alignment.Center
        ) {
            Text(
                "R:$rojo  G:$verde  B:$azul",
                color  = if ((rojo + verde + azul) > 382) Color.Black else Color.White,
                style  = MaterialTheme.typography.labelMedium,
                fontWeight = FontWeight.Bold
            )
        }

        // Cada ControlColor es "tonto": recibe valor y notifica cambio
        // No tiene estado propio — cumple con el principio de state hoisting
        ControlColor("Rojo",  rojo,  Color.Red)   { rojo  = it }
        ControlColor("Verde", verde, Color(0xFF4CAF50)) { verde = it }
        ControlColor("Azul",  azul,  Color.Blue)  { azul  = it }

        OutlinedButton(
            onClick  = { rojo = 0; verde = 0; azul = 0 },
            modifier = Modifier.align(Alignment.End)
        ) {
            Text("Resetear")
        }
    }
}

// Composable stateless (sin estado) — el patrón correcto de state hoisting
// 'valor'    → el padre proporciona el valor actual
// 'onChange' → el hijo notifica el cambio; el padre decide qué hacer
@Composable
private fun ControlColor(
    etiqueta: String,
    valor:    Int,
    color:    Color,
    onChange: (Int) -> Unit
) {
    Row(
        modifier          = Modifier.fillMaxWidth(),
        verticalAlignment = Alignment.CenterVertically,
        horizontalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        // Muestra del color con ancho fijo
        Box(
            modifier = Modifier
                .size(24.dp)
                .clip(RoundedCornerShape(4.dp))
                .background(color)
        )

        Text(
            text     = "$etiqueta: $valor",
            modifier = Modifier.width(90.dp),
            style    = MaterialTheme.typography.bodySmall
        )

        // Slider: componente de Compose para valores continuos
        // El Slider llama onChange en cada desplazamiento
        Slider(
            value         = valor.toFloat(),
            onValueChange = { onChange(it.toInt()) },
            valueRange    = 0f..255f,
            modifier      = Modifier.weight(1f),
            colors        = SliderDefaults.colors(
                thumbColor       = color,
                activeTrackColor = color
            )
        )
    }
}

@Preview(showBackground = true)
@Composable
fun EjemploEstadoPreview() {
    MaterialTheme { EjemploEstado() }
}
```

**▶ Ejecuta y observa:**
- El contador cambia de color según sea positivo, negativo o cero
- Los botones `+` y `−` se deshabilitan al llegar a los límites ±10
- La barra de progreso refleja la posición dentro del rango — sin estado propio
- Mueve los sliders del mezclador → la caja de color cambia en tiempo real
- Presiona Resetear → los tres estados vuelven a 0 desde el padre

**Experimenta:**
- Agrega un cuarto control `"Alfa"` que controle la transparencia con `Color(rojo, verde, azul, alfa)`
- En `DemoContador`, cambia el rango a `0..100` y el progreso a `cuenta / 100f`
- En `ControlColor`, extrae el Slider a un composable aún más pequeño llamado
  `BandaSlider` para practicar la jerarquía de composables

---

## Desglose del proyecto: de un archivo a varios

Cuando los composables crecen, mantener todo en `MainActivity.kt` se vuelve
inmanejable. La solución es separar cada pantalla o grupo lógico en su propio
archivo siguiendo el principio de Responsabilidad Única.

### Estructura del proyecto

```
app/src/main/java/com/tuapp/compose/
├── MainActivity.kt              ← punto de entrada, solo coordina
├── ui/
│   ├── S01_Saludo.kt            ← @Composable básico y reglas
│   ├── S02_Text.kt              ← Text con estilos tipográficos
│   ├── S03_Button.kt            ← variantes de Button
│   ├── S04_Layout.kt            ← Column, Row, Box
│   ├── S05_Modifier.kt          ← Modifier encadenado
│   ├── S06_Estado.kt            ← remember + mutableStateOf
│   ├── S07_StateHoisting.kt     ← elevación del estado
│   └── S08_Bienvenida.kt        ← pantalla completa integradora
└── theme/
    └── Theme.kt
```

### `MainActivity.kt` — coordinator único

```kotlin
// MainActivity.kt
package com.tuapp.compose

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.material3.MaterialTheme
import com.tuapp.compose.ui.*

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                // ◀ CAMBIA AQUÍ para probar cada sección:
                // S01_SaludoScreen()
                // S02_TextScreen()
                // S03_ButtonScreen()
                // S04_LayoutScreen()
                // S05_ModifierScreen()
                // S06_EstadoScreen()
                // S07_StateHoistingScreen()
                S08_BienvenidaScreen()
            }
        }
    }
}
```

---

### Sección 1 — `@Composable`: la unidad básica

**Crea:** `ui/S01_Saludo.kt`

```kotlin
// ui/S01_Saludo.kt
package com.tuapp.compose.ui

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp

@Composable
fun Saludo(nombre: String) {
    Text(text = "Hola, $nombre!")
}

@Composable
fun S01_SaludoScreen() {
    Column(
        modifier            = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("Sección 1 · @Composable básico",
             style = MaterialTheme.typography.titleMedium)
        HorizontalDivider()

        // El mismo composable, distintos argumentos
        Saludo("Ana")
        Saludo("Luis")
        Saludo("Kotlin")

        HorizontalDivider()

        MensajeCondicional(mostrar = true)
        MensajeCondicional(mostrar = false)
    }
}

@Composable
private fun MensajeCondicional(mostrar: Boolean) {
    if (mostrar) {
        Text("✅ mostrar = true  → se dibuja")
    } else {
        // Este Text NUNCA aparece en pantalla — el else es solo para claridad
        Text("(mostrar = false → este Text existe en código pero no se dibuja)",
             color = MaterialTheme.colorScheme.outline)
    }
}

@Preview(showBackground = true)
@Composable
fun S01_Preview() {
    MaterialTheme { S01_SaludoScreen() }
}
```

**▶ En `MainActivity`:** descomenta `S01_SaludoScreen()`.

---

### Sección 2 — `Text`: estilos tipográficos

**Crea:** `ui/S02_Text.kt`

```kotlin
// ui/S02_Text.kt
package com.tuapp.compose.ui

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontStyle
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.text.style.TextDecoration
import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp

@Composable
fun S02_TextScreen() {
    Column(
        modifier            = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        Text("Sección 2 · Text con estilos",
             style = MaterialTheme.typography.titleMedium)
        HorizontalDivider()

        EtiquetaSeccion("1. Texto básico")
        Text("Texto simple sin propiedades adicionales")

        EtiquetaSeccion("2. fontSize + fontWeight + fontStyle")
        Text("Negrita 24sp",   fontSize = 24.sp, fontWeight = FontWeight.Bold)
        Text("Cursiva 18sp",   fontSize = 18.sp, fontStyle  = FontStyle.Italic)
        Text("Light 20sp",     fontSize = 20.sp, fontWeight = FontWeight.Light)

        EtiquetaSeccion("3. Color y decoración")
        Text("Texto en azul",
             color = Color(0xFF1976D2))
        Text("Subrayado",
             textDecoration = TextDecoration.Underline)
        Text("Tachado",
             textDecoration = TextDecoration.LineThrough,
             color          = MaterialTheme.colorScheme.onSurfaceVariant)

        EtiquetaSeccion("4. maxLines + TextOverflow")
        Text(
            text     = "Este texto es muy largo y definitivamente no cabe en una sola línea del dispositivo móvil moderno",
            maxLines = 1,
            overflow = TextOverflow.Ellipsis
        )
        Text(
            text     = "Este texto está limitado a dos líneas y el resto se corta con puntos suspensivos al final del segundo renglón",
            maxLines = 2,
            overflow = TextOverflow.Ellipsis
        )

        EtiquetaSeccion("5. Escala tipográfica Material 3")
        Text("headlineMedium", style = MaterialTheme.typography.headlineMedium)
        Text("titleLarge",     style = MaterialTheme.typography.titleLarge)
        Text("bodyLarge",      style = MaterialTheme.typography.bodyLarge)
        Text("bodySmall",      style = MaterialTheme.typography.bodySmall)
        Text("labelSmall",     style = MaterialTheme.typography.labelSmall)

        EtiquetaSeccion("6. TextAlign")
        Text(
            text      = "Texto centrado en todo el ancho disponible",
            textAlign = TextAlign.Center,
            modifier  = Modifier.fillMaxWidth()
        )
        Text(
            text      = "Alineado a la derecha",
            textAlign = TextAlign.End,
            modifier  = Modifier.fillMaxWidth()
        )
    }
}

// Composable de etiqueta reutilizable — se declara internal para que
// otros archivos del mismo módulo puedan usarla
@Composable
internal fun EtiquetaSeccion(texto: String) {
    Text(
        text  = texto,
        style = MaterialTheme.typography.labelMedium,
        color = MaterialTheme.colorScheme.primary
    )
}

@Preview(showBackground = true)
@Composable
fun S02_Preview() {
    MaterialTheme { S02_TextScreen() }
}
```

**▶ En `MainActivity`:** descomenta `S02_TextScreen()`.

---

### Sección 3 — `Button`: variantes

**Crea:** `ui/S03_Button.kt`

```kotlin
// ui/S03_Button.kt
package com.tuapp.compose.ui

import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Add
import androidx.compose.material.icons.filled.Delete
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp

@Composable
fun S03_ButtonScreen() {
    // Estado para mostrar cuál botón fue presionado
    var ultimoClick by remember { mutableStateOf("(ninguno)") }

    Column(
        modifier            = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.spacedBy(10.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("Sección 3 · Variantes de Button",
             style = MaterialTheme.typography.titleMedium)
        HorizontalDivider()

        // Panel de feedback — muestra el último click
        Surface(
            color    = MaterialTheme.colorScheme.surfaceVariant,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(
                text     = "Último click: $ultimoClick",
                modifier = Modifier.padding(12.dp),
                style    = MaterialTheme.typography.bodyMedium
            )
        }

        Spacer(Modifier.height(4.dp))

        Button(
            onClick  = { ultimoClick = "Button (Primary)" },
            modifier = Modifier.fillMaxWidth()
        ) { Text("Button — Primary") }

        // Button con ícono dentro del slot de contenido
        Button(
            onClick  = { ultimoClick = "Button con ícono" },
            modifier = Modifier.fillMaxWidth()
        ) {
            Icon(
                imageVector        = Icons.Default.Add,
                contentDescription = null,
                modifier           = Modifier.size(18.dp)
            )
            Spacer(Modifier.width(8.dp))
            Text("Button con ícono")
        }

        OutlinedButton(
            onClick  = { ultimoClick = "OutlinedButton" },
            modifier = Modifier.fillMaxWidth()
        ) { Text("OutlinedButton") }

        TextButton(
            onClick  = { ultimoClick = "TextButton" },
            modifier = Modifier.fillMaxWidth()
        ) { Text("TextButton") }

        ElevatedButton(
            onClick  = { ultimoClick = "ElevatedButton" },
            modifier = Modifier.fillMaxWidth()
        ) { Text("ElevatedButton") }

        FilledTonalButton(
            onClick  = { ultimoClick = "FilledTonalButton" },
            modifier = Modifier.fillMaxWidth()
        ) { Text("FilledTonalButton") }

        // enabled = false → el botón no dispara onClick, apariencia atenuada
        Button(
            onClick  = { },
            enabled  = false,
            modifier = Modifier.fillMaxWidth()
        ) { Text("Deshabilitado (enabled = false)") }

        HorizontalDivider()

        // IconButton — solo ícono, sin texto
        EtiquetaSeccion("IconButton")
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            IconButton(onClick = { ultimoClick = "IconButton Add" }) {
                Icon(Icons.Default.Add, contentDescription = "Agregar")
            }
            IconButton(onClick = { ultimoClick = "IconButton Delete" }) {
                Icon(Icons.Default.Delete, contentDescription = "Eliminar",
                     tint = MaterialTheme.colorScheme.error)
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun S03_Preview() {
    MaterialTheme { S03_ButtonScreen() }
}
```

**▶ En `MainActivity`:** descomenta `S03_ButtonScreen()`.

---

### Sección 4 — `Column`, `Row` y `Box`: layouts

**Crea:** `ui/S04_Layout.kt`

```kotlin
// ui/S04_Layout.kt
package com.tuapp.compose.ui

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp

@Composable
fun S04_LayoutScreen() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
            .verticalScroll(rememberScrollState()),  // scroll para ver todo el contenido
        verticalArrangement = Arrangement.spacedBy(20.dp)
    ) {
        Text("Sección 4 · Column · Row · Box",
             style = MaterialTheme.typography.titleMedium)
        HorizontalDivider()

        // ── COLUMN ─────────────────────────────────────────────────────────
        EtiquetaSeccion("Column — apila verticalmente")
        Column(
            modifier            = Modifier
                .fillMaxWidth()
                .background(Color(0xFFE3F2FD))
                .padding(12.dp),
            verticalArrangement = Arrangement.spacedBy(4.dp),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            CeldaLayout("Elemento 1", Color(0xFF90CAF9))
            CeldaLayout("Elemento 2", Color(0xFF64B5F6))
            CeldaLayout("Elemento 3", Color(0xFF42A5F5))
        }

        // ── ROW con Arrangement ────────────────────────────────────────────
        EtiquetaSeccion("Row — SpaceBetween")
        Row(
            modifier              = Modifier
                .fillMaxWidth()
                .background(Color(0xFFF3E5F5))
                .padding(12.dp),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment     = Alignment.CenterVertically
        ) {
            Text("Izquierda")
            Text("Centro")
            Text("Derecha")
        }

        EtiquetaSeccion("Row — SpaceEvenly")
        Row(
            modifier              = Modifier
                .fillMaxWidth()
                .background(Color(0xFFE8F5E9))
                .padding(12.dp),
            horizontalArrangement = Arrangement.SpaceEvenly
        ) {
            Text("A"); Text("B"); Text("C"); Text("D")
        }

        // ── ROW con weight ─────────────────────────────────────────────────
        EtiquetaSeccion("Row + weight (distribución proporcional 1:2:1)")
        Row(Modifier.fillMaxWidth().height(50.dp)) {
            Box(Modifier.weight(1f).fillMaxHeight().background(Color(0xFFEF9A9A)),
                contentAlignment = Alignment.Center) { Text("1") }
            Box(Modifier.weight(2f).fillMaxHeight().background(Color(0xFFE57373)),
                contentAlignment = Alignment.Center) { Text("2") }
            Box(Modifier.weight(1f).fillMaxHeight().background(Color(0xFFEF5350)),
                contentAlignment = Alignment.Center) { Text("1") }
        }

        // ── BOX ────────────────────────────────────────────────────────────
        EtiquetaSeccion("Box — superpone capas con align por capa")
        Box(
            modifier         = Modifier
                .fillMaxWidth()
                .height(120.dp)
                .background(Color(0xFF1565C0)),
            contentAlignment = Alignment.Center   // alineación por defecto
        ) {
            // Capa 1: esquina superior izquierda
            Box(Modifier.size(40.dp).background(Color(0xFF42A5F5))
                    .align(Alignment.TopStart))
            // Capa 2: esquina inferior derecha
            Box(Modifier.size(40.dp).background(Color(0xFF1976D2))
                    .align(Alignment.BottomEnd))
            // Capa 3: centrada (por defecto del Box padre)
            Text("Texto encima de todo",
                 color = Color.White,
                 style = MaterialTheme.typography.labelLarge)
        }
    }
}

@Composable
private fun CeldaLayout(label: String, color: Color) {
    Box(
        modifier         = Modifier.fillMaxWidth().height(36.dp).background(color),
        contentAlignment = Alignment.Center
    ) { Text(label, style = MaterialTheme.typography.labelMedium) }
}

@Preview(showBackground = true)
@Composable
fun S04_Preview() {
    MaterialTheme { S04_LayoutScreen() }
}
```

**▶ En `MainActivity`:** descomenta `S04_LayoutScreen()`.

---

### Sección 5 — `Modifier`: el orden importa

**Crea:** `ui/S05_Modifier.kt`

```kotlin
// ui/S05_Modifier.kt
package com.tuapp.compose.ui

import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp

@Composable
fun S05_ModifierScreen() {
    var ultimoClick by remember { mutableStateOf("Toca algún elemento") }

    Column(
        modifier            = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        Text("Sección 5 · Modifier",
             style = MaterialTheme.typography.titleMedium)
        HorizontalDivider()

        // Panel de feedback
        Surface(
            color    = MaterialTheme.colorScheme.surfaceVariant,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(ultimoClick, Modifier.padding(12.dp),
                 style = MaterialTheme.typography.bodySmall)
        }

        EtiquetaSeccion("1. clip ANTES de background (correcto)")
        Box(
            modifier = Modifier
                .size(130.dp)
                .clip(RoundedCornerShape(16.dp))                       // 1° recorta
                .background(MaterialTheme.colorScheme.primaryContainer) // 2° pinta dentro
                .border(2.dp, MaterialTheme.colorScheme.primary, RoundedCornerShape(16.dp))
                .padding(12.dp)
                .clickable { ultimoClick = "Click en Box ✅" },
            contentAlignment = Alignment.Center
        ) {
            Text("clip\nantes de\nbackground ✅",
                 style = MaterialTheme.typography.labelSmall,
                 color = MaterialTheme.colorScheme.onPrimaryContainer)
        }

        EtiquetaSeccion("2. background ANTES de clip (error común)")
        Box(
            modifier = Modifier
                .size(130.dp)
                .background(Color(0xFFFFCDD2))    // 1° pinta (sin recorte aún)
                .clip(RoundedCornerShape(16.dp))  // 2° recorta — tarde para el fondo
                .padding(12.dp),
            contentAlignment = Alignment.Center
        ) {
            Text("background\nantes de\nclip ❌",
                 style = MaterialTheme.typography.labelSmall)
        }

        EtiquetaSeccion("3. CircleShape + clickable")
        Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
            listOf("A" to Color(0xFF1976D2), "B" to Color(0xFF388E3C), "C" to Color(0xFFF57C00))
                .forEach { (letra, color) ->
                    Box(
                        modifier = Modifier
                            .size(56.dp)
                            .clip(CircleShape)
                            .background(color)
                            .clickable { ultimoClick = "Avatar $letra presionado" },
                        contentAlignment = Alignment.Center
                    ) {
                        Text(letra, color = Color.White,
                             style = MaterialTheme.typography.titleMedium)
                    }
                }
        }

        EtiquetaSeccion("4. fillMaxWidth + padding asimétrico")
        Text(
            text     = "horizontal: 32dp, vertical: 8dp",
            modifier = Modifier
                .fillMaxWidth()
                .background(Color(0xFFE8F5E9))
                .padding(horizontal = 32.dp, vertical = 8.dp)
        )

        EtiquetaSeccion("5. size fijo vs fillMaxWidth")
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Box(Modifier.size(60.dp).background(Color(0xFFBBDEFB)),
                contentAlignment = Alignment.Center) { Text("60dp") }
            Box(Modifier.weight(1f).height(60.dp).background(Color(0xFFB3E5FC)),
                contentAlignment = Alignment.Center) { Text("weight(1f)") }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun S05_Preview() {
    MaterialTheme { S05_ModifierScreen() }
}
```

**▶ En `MainActivity`:** descomenta `S05_ModifierScreen()`.

---

### Sección 6 — Estado y recomposición

**Crea:** `ui/S06_Estado.kt`

```kotlin
// ui/S06_Estado.kt
package com.tuapp.compose.ui

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp

@Composable
fun S06_EstadoScreen() {
    Column(
        modifier            = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(24.dp)
    ) {
        Text("Sección 6 · Estado y recomposición",
             style = MaterialTheme.typography.titleMedium)
        HorizontalDivider()

        DemoContadorS6()
        HorizontalDivider()
        DemoEstadoDerivado()
    }
}

// ── Demo 1: Contador clásico ─────────────────────────────────────────────────
@Composable
private fun DemoContadorS6() {
    var cuenta by remember { mutableStateOf(0) }

    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        EtiquetaSeccion("Contador — remember + mutableStateOf")

        // Solo este Text se recompone cuando 'cuenta' cambia
        Text(
            text       = "$cuenta",
            style      = MaterialTheme.typography.displayMedium,
            fontWeight = FontWeight.Bold
        )

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { cuenta-- }) { Text("−") }
            Button(onClick = { cuenta++ }) { Text("+") }
            OutlinedButton(onClick = { cuenta = 0 }) { Text("Reset") }
        }

        Text(
            "Solo el número se recompone al hacer click",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
    }
}

// ── Demo 2: Estado derivado — calculado del estado principal ─────────────────
// El estado derivado NO necesita su propio mutableStateOf
// Se recalcula automáticamente en cada recomposición
@Composable
private fun DemoEstadoDerivado() {
    var nivel by remember { mutableStateOf(0) }
    val max   = 5

    // Estado derivado: calculado del estado 'nivel'
    // No usa remember ni mutableStateOf propio
    val porcentaje = nivel.toFloat() / max
    val etiquetaNivel = when {
        nivel == 0    -> "Sin nivel"
        nivel <= 2    -> "Principiante"
        nivel <= 4    -> "Intermedio"
        else          -> "Avanzado"
    }

    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        EtiquetaSeccion("Estado derivado — calculado del estado principal")

        Text(
            "$etiquetaNivel (nivel $nivel/$max)",
            style      = MaterialTheme.typography.titleMedium,
            fontWeight = FontWeight.SemiBold
        )

        LinearProgressIndicator(
            progress = { porcentaje },
            modifier = Modifier
                .fillMaxWidth()
                .height(12.dp)
                .clip(RoundedCornerShape(6.dp))
        )

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            OutlinedButton(
                onClick  = { if (nivel > 0) nivel-- },
                enabled  = nivel > 0
            ) { Text("Bajar nivel") }

            Button(
                onClick  = { if (nivel < max) nivel++ },
                enabled  = nivel < max
            ) { Text("Subir nivel") }
        }

        Text(
            "porcentaje = ${"%.0f".format(porcentaje * 100)}% " +
            "— derivado de nivel, sin estado propio",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
    }
}

@Preview(showBackground = true)
@Composable
fun S06_Preview() {
    MaterialTheme { S06_EstadoScreen() }
}
```

**▶ En `MainActivity`:** descomenta `S06_EstadoScreen()`.
- Sube y baja el nivel → el porcentaje y la etiqueta cambian sin estado propio
- Los botones se deshabilitan en los límites 0 y 5

---

### Sección 7 — State Hoisting: elevación del estado

**Crea:** `ui/S07_StateHoisting.kt`

```kotlin
// ui/S07_StateHoisting.kt
package com.tuapp.compose.ui

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp

@Composable
fun S07_StateHoistingScreen() {
    Column(
        modifier            = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(20.dp)
    ) {
        Text("Sección 7 · State Hoisting",
             style = MaterialTheme.typography.titleMedium)
        HorizontalDivider()

        DemoEstadoAtrapado()
        HorizontalDivider()
        DemoEstadoElevado()
    }
}

// ── ❌ Anti-patrón: estado atrapado dentro del componente ────────────────────
@Composable
private fun DemoEstadoAtrapado() {
    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        EtiquetaSeccion("❌ Estado atrapado — el padre no puede leerlo")

        Text(
            "El estado vive dentro del botón. El padre no sabe cuántas veces " +
            "fue presionado ni puede usarlo para nada.",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )

        // El estado está atrapado — nadie más puede acceder a él
        BotonAtrapado()

        // El padre intenta mostrar el conteo pero no puede
        Text(
            "El padre no puede mostrar el conteo aquí ❌",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.error
        )
    }
}

@Composable
private fun BotonAtrapado() {
    // 'cuenta' está encerrado aquí — el padre no tiene acceso
    var cuenta by remember { mutableStateOf(0) }
    Button(onClick = { cuenta++ }) {
        Text("Presionado $cuenta veces (estado atrapado)")
    }
}

// ── ✅ Patrón correcto: estado elevado al padre ──────────────────────────────
@Composable
private fun DemoEstadoElevado() {
    // El estado vive aquí — el padre puede usarlo para múltiples propósitos
    var seleccion by remember { mutableStateOf<String?>(null) }
    var historial by remember { mutableStateOf(listOf<String>()) }

    val opciones = listOf("🔴 Rojo", "🟢 Verde", "🔵 Azul", "🟡 Amarillo")

    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        EtiquetaSeccion("✅ Estado elevado — el padre coordina todo")

        Text(
            "El hijo solo notifica qué fue seleccionado. " +
            "El padre actualiza la selección Y el historial.",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )

        // El padre pasa el valor actual y un callback
        // El hijo es "tonto" — no decide qué hacer con el click
        SelectorOpciones(
            opciones   = opciones,
            seleccion  = seleccion,
            onSeleccion = { opcion ->
                // El padre decide qué hacer con el evento:
                seleccion = opcion
                historial = (historial + opcion).takeLast(4)  // máximo 4 entradas
            }
        )

        // El padre usa el mismo estado para dos cosas distintas
        seleccion?.let { sel ->
            val color = when {
                "Rojo"     in sel -> Color(0xFFFFCDD2)
                "Verde"    in sel -> Color(0xFFC8E6C9)
                "Azul"     in sel -> Color(0xFFBBDEFB)
                "Amarillo" in sel -> Color(0xFFFFF9C4)
                else              -> Color.Transparent
            }
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .height(48.dp)
                    .clip(RoundedCornerShape(8.dp))
                    .background(color),
                contentAlignment = Alignment.Center
            ) {
                Text("Seleccionado: $sel",
                     style = MaterialTheme.typography.labelLarge)
            }
        }

        if (historial.isNotEmpty()) {
            Text(
                "Historial: ${historial.joinToString(" → ")}",
                style = MaterialTheme.typography.bodySmall,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}

// Composable stateless — recibe todo por parámetros, no tiene estado propio
// Fácil de testear: solo necesitas pasarle datos y lambdas
@Composable
private fun SelectorOpciones(
    opciones:    List<String>,
    seleccion:   String?,
    onSeleccion: (String) -> Unit
) {
    Column(verticalArrangement = Arrangement.spacedBy(4.dp)) {
        opciones.forEach { opcion ->
            val estaSeleccionado = seleccion == opcion
            Button(
                onClick  = { onSeleccion(opcion) },
                modifier = Modifier.fillMaxWidth(),
                colors   = if (estaSeleccionado)
                    ButtonDefaults.buttonColors()
                else
                    ButtonDefaults.outlinedButtonColors()
            ) {
                Text(opcion)
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun S07_Preview() {
    MaterialTheme { S07_StateHoistingScreen() }
}
```

**▶ En `MainActivity`:** descomenta `S07_StateHoistingScreen()`.
- Compara el botón atrapado (arriba) vs los botones elevados (abajo)
- Presiona opciones → la caja de color y el historial se actualizan
- El historial muestra hasta 4 entradas — lógica del padre, no del hijo

---

### Sección 8 — Pantalla completa: todos los conceptos integrados

**Crea:** `ui/S08_Bienvenida.kt`

Integra `@Composable`, `Text`, `Button`, `Column`/`Box`, `Modifier`,
`remember`, `mutableStateOf` y state hoisting en una pantalla cohesiva.
Sin componentes de la página 12 — todo con lo aprendido aquí.

```kotlin
// ui/S08_Bienvenida.kt
package com.tuapp.compose.ui

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp

@Composable
fun S08_BienvenidaScreen() {
    // Estado elevado a la pantalla completa
    var paso by remember { mutableStateOf(1) }  // 1, 2 o 3

    Column(
        modifier            = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        when (paso) {
            1 -> PasoUno(onSiguiente = { paso = 2 })
            2 -> PasoDos(onSiguiente = { paso = 3 }, onVolver = { paso = 1 })
            3 -> PasoTres(onReiniciar = { paso = 1 })
        }
    }
}

// ── Paso 1: elige un tema de color ───────────────────────────────────────────
@Composable
private fun PasoUno(onSiguiente: () -> Unit) {
    var temaElegido by remember { mutableStateOf<String?>(null) }

    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        // Indicador de progreso derivado — sin estado propio
        IndicadorPasos(pasoActual = 1, totalPasos = 3)

        Spacer(Modifier.height(8.dp))

        Text("Elige un tema",
             style      = MaterialTheme.typography.headlineSmall,
             fontWeight = FontWeight.Bold)
        Text("Selecciona el color que más te guste",
             style = MaterialTheme.typography.bodyMedium,
             color = MaterialTheme.colorScheme.onSurfaceVariant)

        Spacer(Modifier.height(8.dp))

        // Selector de tema — state hoisting: el estado vive en PasoUno
        listOf("🔵 Azul", "🟢 Verde", "🟣 Morado").forEach { tema ->
            val seleccionado = temaElegido == tema
            Button(
                onClick  = { temaElegido = tema },
                modifier = Modifier.fillMaxWidth(),
                colors   = if (seleccionado) ButtonDefaults.buttonColors()
                            else ButtonDefaults.outlinedButtonColors()
            ) {
                Text(tema)
                if (seleccionado) {
                    Spacer(Modifier.width(8.dp))
                    Text("✓")
                }
            }
        }

        Spacer(Modifier.height(8.dp))

        Button(
            onClick  = onSiguiente,
            enabled  = temaElegido != null,  // solo activo si eligió algo
            modifier = Modifier.fillMaxWidth().height(50.dp),
            shape    = RoundedCornerShape(12.dp)
        ) {
            Text("Siguiente →")
        }
    }
}

// ── Paso 2: ajusta el nivel de experiencia ───────────────────────────────────
@Composable
private fun PasoDos(onSiguiente: () -> Unit, onVolver: () -> Unit) {
    var nivel by remember { mutableStateOf(1) }
    val max = 5

    val descripcion = when (nivel) {
        1    -> "Recién comienzo con Android"
        2    -> "Conozco algo de XML"
        3    -> "Tengo experiencia con Views"
        4    -> "He usado Compose antes"
        else -> "Experto en Compose"
    }

    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        IndicadorPasos(pasoActual = 2, totalPasos = 3)

        Spacer(Modifier.height(8.dp))

        Text("Tu nivel de experiencia",
             style      = MaterialTheme.typography.headlineSmall,
             fontWeight = FontWeight.Bold)

        Text(
            "Nivel $nivel de $max",
            style      = MaterialTheme.typography.displaySmall,
            fontWeight = FontWeight.Bold,
            color      = MaterialTheme.colorScheme.primary
        )

        Text(descripcion,
             style = MaterialTheme.typography.bodyLarge,
             color = MaterialTheme.colorScheme.onSurfaceVariant)

        // Estado derivado — sin mutableStateOf propio
        LinearProgressIndicator(
            progress = { nivel.toFloat() / max },
            modifier = Modifier.fillMaxWidth().height(8.dp)
                .clip(RoundedCornerShape(4.dp))
        )

        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            OutlinedButton(
                onClick  = { if (nivel > 1) nivel-- },
                enabled  = nivel > 1
            ) { Text("−") }
            Button(
                onClick  = { if (nivel < max) nivel++ },
                enabled  = nivel < max
            ) { Text("+") }
        }

        Spacer(Modifier.height(8.dp))

        Row(
            modifier              = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            OutlinedButton(onClick = onVolver, modifier = Modifier.weight(1f)) {
                Text("← Volver")
            }
            Button(onClick = onSiguiente, modifier = Modifier.weight(1f),
                   shape = RoundedCornerShape(12.dp)) {
                Text("Siguiente →")
            }
        }
    }
}

// ── Paso 3: pantalla de confirmación ─────────────────────────────────────────
@Composable
private fun PasoTres(onReiniciar: () -> Unit) {
    Column(
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        IndicadorPasos(pasoActual = 3, totalPasos = 3)

        Spacer(Modifier.height(8.dp))

        // Avatar con ícono de check
        Box(
            modifier         = Modifier
                .size(88.dp)
                .clip(CircleShape)
                .background(MaterialTheme.colorScheme.primaryContainer),
            contentAlignment = Alignment.Center
        ) {
            Text("✓",
                 style      = MaterialTheme.typography.displaySmall,
                 fontWeight = FontWeight.Bold,
                 color      = MaterialTheme.colorScheme.onPrimaryContainer)
        }

        Text("¡Configuración lista!",
             style      = MaterialTheme.typography.headlineSmall,
             fontWeight = FontWeight.Bold)

        Text(
            "Has completado los 3 pasos del onboarding.\n" +
            "En la página 12 aprenderemos TextField, Card, LazyColumn y más.",
            style     = MaterialTheme.typography.bodyMedium,
            color     = MaterialTheme.colorScheme.onSurfaceVariant
        )

        Spacer(Modifier.height(8.dp))

        OutlinedButton(
            onClick  = onReiniciar,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("↺ Empezar de nuevo")
        }
    }
}

// ── Composable de indicador de pasos — stateless ─────────────────────────────
@Composable
private fun IndicadorPasos(pasoActual: Int, totalPasos: Int) {
    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
        (1..totalPasos).forEach { paso ->
            Box(
                modifier = Modifier
                    .size(if (paso == pasoActual) 12.dp else 8.dp)
                    .clip(CircleShape)
                    .background(
                        if (paso <= pasoActual) MaterialTheme.colorScheme.primary
                        else MaterialTheme.colorScheme.surfaceVariant
                    )
            )
        }
    }
}

@Preview(showBackground = true)
@Composable
fun S08_Preview() {
    MaterialTheme { S08_BienvenidaScreen() }
}
```

**▶ En `MainActivity`:** `S08_BienvenidaScreen()` ya está activo.
- Navega los 3 pasos con los botones
- Elige un tema en el paso 1 → el botón "Siguiente" se activa
- Ajusta el nivel en el paso 2 con los límites
- Presiona "Empezar de nuevo" en el paso 3 → vuelve al inicio
- Rota el emulador en cualquier paso → el paso actual se pierde
  (en la página 13 lo resolveremos con `ViewModel`)

---

## Referencia rápida — qué descomentar en `MainActivity`

| Sección | Archivo | Descomentar | Componentes usados |
|---|---|---|---|
| 1 · @Composable | `S01_Saludo.kt` | `S01_SaludoScreen()` | `Text`, condicionales |
| 2 · Text | `S02_Text.kt` | `S02_TextScreen()` | `Text` con todos sus parámetros |
| 3 · Button | `S03_Button.kt` | `S03_ButtonScreen()` | Todas las variantes de `Button`, `IconButton` |
| 4 · Layouts | `S04_Layout.kt` | `S04_LayoutScreen()` | `Column`, `Row`, `Box`, `weight` |
| 5 · Modifier | `S05_Modifier.kt` | `S05_ModifierScreen()` | Cadenas de `Modifier`, `clip`, `clickable` |
| 6 · Estado | `S06_Estado.kt` | `S06_EstadoScreen()` | `remember`, `mutableStateOf`, estado derivado |
| 7 · Hoisting | `S07_StateHoisting.kt` | `S07_StateHoistingScreen()` | State hoisting, composables stateless |
| 8 · Integrado | `S08_Bienvenida.kt` | `S08_BienvenidaScreen()` | Todo lo anterior combinado |

---

## Ejercicios propuestos

**Ejercicio 1 — Contador con historial** (`ui/E01_ContadorHistorial.kt`)
Extiende `DemoContadorS6` con un `var historial by remember { mutableStateOf(listOf<Int>()) }`.
Cada click en `+` o `−` agrega el valor resultante al historial.
Muestra los últimos 5 valores en una `Row` con `Text` separados por `→`.
El botón Reset también limpia el historial.

**Ejercicio 2 — Semáforo interactivo** (`ui/E02_Semaforo.kt`)
Crea un composable que muestre tres círculos (`Box` con `CircleShape`) en
rojo, amarillo y verde. Un botón "Cambiar" avanza al siguiente estado
cíclicamente. Solo el círculo activo tiene opacidad completa
(`.alpha(1f)`), los otros usan `.alpha(0.3f)`. Muestra el nombre del
estado actual debajo.

**Ejercicio 3 — Votación** (`ui/E03_Votacion.kt`)
Crea una pantalla con 4 opciones (texto a elección). Cada opción tiene un
`Button` de "Votar". El padre acumula los votos en un `Map<String, Int>`
con `remember`. Debajo de cada opción muestra una barra de progreso
(`LinearProgressIndicator`) cuyo progreso es `votos[opcion] / totalVotos`.
El composable de cada opción recibe `(texto, votos, total, onVotar)` — sin
estado propio.

**Ejercicio 4 — Onboarding extendido** (`ui/E04_Onboarding.kt`)
Extiende `S08_BienvenidaScreen` a 5 pasos. Agrega un paso de "Selección
de idioma" (3 botones: Español, English, Português) y un paso de
"Notificaciones" (3 `Switch` independientes: Marketing, Novedades, Alertas).
El indicador de puntos debe actualizarse correctamente. Todos los estados
viven en el composable padre — ningún paso tiene estado propio.

---

## Resumen de la página 11

- Compose es **declarativo**: `UI = f(estado)`. Describes cómo se ve, no cómo cambiarla paso a paso.
- `@Composable` marca funciones que dibujan UI. Solo pueden llamarse desde otras funciones `@Composable`.
- `remember { mutableStateOf(valor) }` crea estado que sobrevive recomposiciones dentro del mismo composable.
- El **estado derivado** se calcula directamente del estado principal sin necesitar su propio `mutableStateOf` — se recomputa en cada recomposición.
- Cuando el estado cambia, Compose **recompone solo los composables** que leen ese estado.
- **State hoisting**: el estado sube al padre común para que varios composables puedan usarlo y para hacer los hijos fácilmente testeables.
- `Modifier` es la cadena de transformaciones visuales. El **orden importa**: `clip` antes de `background` para que el fondo respete la forma recortada.
- `Column` apila verticalmente, `Row` alinea horizontalmente, `Box` superpone en capas. `weight` distribuye el espacio en proporciones.
- `@Preview` visualiza composables en el editor sin ejecutar la app.

---

> **Siguiente página →** Página 12: Jetpack Compose — componentes de
> Material 3: `TextField`, `Card`, `LazyColumn`, `Scaffold` y diálogos.