# Curso Kotlin — Página 12 (Revisada)
## Módulo 6 · Jetpack Compose
### Componentes Material 3: `TextField`, `Card`, `LazyColumn`, `Scaffold` y diálogos

---

## Componentes que cubrimos

```
TextField / OutlinedTextField   ← entrada de texto con validación
Card / ElevatedCard             ← contenedor con elevación
LazyColumn / LazyRow            ← listas virtualizadas de alto rendimiento
Scaffold                        ← estructura de pantalla completa
TopAppBar                       ← barra de título
NavigationBar                   ← navegación inferior
AlertDialog / Dialog            ← diálogos y confirmaciones
```

---

## Proyecto de la página 12: Agenda de Contactos

A diferencia de la página 11 donde exploramos cada concepto en `MainActivity`,
aquí construimos **un proyecto real desde cero**, agregando cada componente
nuevo de forma incremental. Al final de la página tendrás una app funcional
con búsqueda, lista, navegación y diálogos.

### Estructura del proyecto

```
app/src/main/java/com/tuapp/contactos/
├── MainActivity.kt              ← punto de entrada (no cambia entre pasos)
├── model/
│   └── Contacto.kt              ← modelo de datos
├── ui/
│   ├── Paso01_TextField.kt      ← Paso 1: formulario de búsqueda
│   ├── Paso02_Card.kt           ← Paso 2: tarjeta de contacto
│   ├── Paso03_LazyColumn.kt     ← Paso 3: lista virtualizada
│   ├── Paso04_Scaffold.kt       ← Paso 4: estructura completa
│   ├── Paso05_NavBar.kt         ← Paso 5: navegación inferior
│   └── Paso06_Dialogos.kt       ← Paso 6: diálogos y snackbar
└── theme/
    └── Theme.kt
```

### `MainActivity.kt` — no cambia entre pasos

```kotlin
// MainActivity.kt
package com.tuapp.contactos

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.material3.MaterialTheme
import com.tuapp.contactos.ui.*

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                // ◀ CAMBIA AQUÍ para probar cada paso:
                // Paso01_TextFieldScreen()
                // Paso02_CardScreen()
                // Paso03_LazyColumnScreen()
                // Paso04_ScaffoldScreen()
                // Paso05_NavBarScreen()
                Paso06_DialogosScreen()   // ← paso activo
            }
        }
    }
}
```

### Modelo de datos compartido

**Crea:** `model/Contacto.kt`

Este archivo lo usan todos los pasos. Créalo primero antes de avanzar.

```kotlin
// model/Contacto.kt
package com.tuapp.contactos.model

data class Contacto(
    val id:       Int,
    val nombre:   String,
    val email:    String,
    val telefono: String,
    val favorito: Boolean = false
)

// Lista de muestra — la usamos en todos los pasos
val contactosDeMuestra = listOf(
    Contacto(1, "Ana García",    "ana@ejemplo.com",    "+593 99 111 2222", favorito = true),
    Contacto(2, "Luis Martínez", "luis@ejemplo.com",   "+593 99 333 4444"),
    Contacto(3, "María López",   "maria@ejemplo.com",  "+593 99 555 6666", favorito = true),
    Contacto(4, "Carlos Ruiz",   "carlos@ejemplo.com", "+593 99 777 8888"),
    Contacto(5, "Sofía Torres",  "sofia@ejemplo.com",  "+593 99 999 0000"),
    Contacto(6, "Pedro Mora",    "pedro@ejemplo.com",  "+593 98 111 2222"),
    Contacto(7, "Elena Vega",    "elena@ejemplo.com",  "+593 98 333 4444", favorito = true),
    Contacto(8, "Diego Paz",     "diego@ejemplo.com",  "+593 98 555 6666"),
)
```

> **¿Por qué un archivo de modelo separado?**
> El principio de Responsabilidad Única (SRP) de SOLID indica que cada
> archivo debe tener una sola razón para cambiar. Los datos de negocio no
> deben vivir en los composables de UI.

---

## `TextField` y `OutlinedTextField`

### Concepto

`OutlinedTextField` es la variante más usada en Material 3. A diferencia
de un `TextField` HTML, en Compose el campo es **completamente controlado**
por el estado: tú decides qué muestra y qué hace con los cambios.

```
valor actual ──▶ OutlinedTextField ──▶ onValueChange (nuevo valor)
                        ▲                       │
                        └───── setState ◀────────┘
```

Los parámetros más importantes:

| Parámetro | Propósito |
|---|---|
| `value` | El texto que muestra actualmente |
| `onValueChange` | Callback con el texto nuevo al escribir |
| `label` | Texto flotante que describe el campo |
| `placeholder` | Texto gris cuando está vacío |
| `leadingIcon` | Ícono a la izquierda |
| `trailingIcon` | Ícono a la derecha (limpiar, toggle visibilidad) |
| `isError` | Activa el estado de error (borde rojo) |
| `supportingText` | Texto de ayuda o error debajo del campo |
| `keyboardOptions` | Tipo de teclado e IME action |
| `visualTransformation` | Transforma lo que se muestra (ej: contraseñas) |

### Paso 1 — Formulario de búsqueda

**Crea:** `ui/Paso01_TextField.kt`

En este paso construimos la barra de búsqueda y un formulario de contacto
con validación en tiempo real. Todo el estado vive en el composable padre.

```kotlin
// ui/Paso01_TextField.kt
package com.tuapp.contactos.ui

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.input.KeyboardType
import androidx.compose.ui.text.input.PasswordVisualTransformation
import androidx.compose.ui.text.input.VisualTransformation
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp

@Composable
fun Paso01_TextFieldScreen() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
            .verticalScroll(rememberScrollState()),
        verticalArrangement = Arrangement.spacedBy(24.dp)
    ) {
        Text("Paso 1 · TextField y OutlinedTextField",
             style = MaterialTheme.typography.titleMedium)
        HorizontalDivider()

        DemoBusqueda()
        HorizontalDivider()
        DemoFormularioContacto()
    }
}

// ── Demo 1: campo de búsqueda con limpiar ────────────────────────────────────
@Composable
private fun DemoBusqueda() {
    var busqueda by remember { mutableStateOf("") }

    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Text("Búsqueda con ícono y botón limpiar",
             style = MaterialTheme.typography.labelLarge,
             color = MaterialTheme.colorScheme.primary)

        OutlinedTextField(
            value         = busqueda,
            onValueChange = { busqueda = it },
            placeholder   = { Text("Buscar contacto...") },
            leadingIcon   = { Icon(Icons.Default.Search, contentDescription = null) },
            // trailingIcon solo aparece cuando hay texto — evita un ícono permanente inútil
            trailingIcon  = {
                if (busqueda.isNotEmpty()) {
                    IconButton(onClick = { busqueda = "" }) {
                        Icon(Icons.Default.Clear, contentDescription = "Limpiar búsqueda")
                    }
                }
            },
            singleLine = true,
            modifier   = Modifier.fillMaxWidth()
        )

        // El texto de ayuda refleja el estado en tiempo real
        Text(
            text  = if (busqueda.isBlank()) "Escribe para filtrar"
                    else "Buscando: \"$busqueda\"",
            style = MaterialTheme.typography.bodySmall,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
    }
}

// ── Demo 2: formulario con validación completa ───────────────────────────────
@Composable
private fun DemoFormularioContacto() {
    var nombre     by remember { mutableStateOf("") }
    var email      by remember { mutableStateOf("") }
    var telefono   by remember { mutableStateOf("") }
    var contrasena by remember { mutableStateOf("") }
    var verPass    by remember { mutableStateOf(false) }

    // Validaciones derivadas del estado — se recalculan en cada recomposición
    val nombreValido   = nombre.trim().length >= 2
    val emailValido    = email.contains("@") && email.contains(".")
    val telefonoValido = telefono.length >= 7 && telefono.all { it.isDigit() || it == '+' || it == ' ' }
    val passValida     = contrasena.length >= 8

    val formularioValido = nombreValido && emailValido && telefonoValido && passValida

    Column(verticalArrangement = Arrangement.spacedBy(12.dp)) {
        Text("Formulario nuevo contacto",
             style = MaterialTheme.typography.labelLarge,
             color = MaterialTheme.colorScheme.primary)

        // Nombre — validación básica de longitud
        OutlinedTextField(
            value           = nombre,
            onValueChange   = { nombre = it },
            label           = { Text("Nombre completo") },
            leadingIcon     = { Icon(Icons.Default.Person, contentDescription = null) },
            isError         = nombre.isNotEmpty() && !nombreValido,
            supportingText  = {
                when {
                    nombre.isNotEmpty() && !nombreValido ->
                        Text("Mínimo 2 caracteres", color = MaterialTheme.colorScheme.error)
                    nombreValido ->
                        Text("✓ Nombre válido", color = MaterialTheme.colorScheme.primary)
                    else -> Text("Requerido")
                }
            },
            // keyboardOptions configura el teclado del sistema operativo
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
            singleLine      = true,
            modifier        = Modifier.fillMaxWidth()
        )

        // Email — KeyboardType.Email activa el teclado con @ visible
        OutlinedTextField(
            value           = email,
            onValueChange   = { email = it },
            label           = { Text("Correo electrónico") },
            placeholder     = { Text("usuario@dominio.com") },
            leadingIcon     = { Icon(Icons.Default.Email, contentDescription = null) },
            isError         = email.isNotEmpty() && !emailValido,
            supportingText  = {
                if (email.isNotEmpty() && !emailValido)
                    Text("Formato inválido (requiere @ y dominio)",
                         color = MaterialTheme.colorScheme.error)
            },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Email,
                imeAction    = ImeAction.Next
            ),
            singleLine  = true,
            modifier    = Modifier.fillMaxWidth()
        )

        // Teléfono — KeyboardType.Phone activa teclado numérico
        OutlinedTextField(
            value           = telefono,
            onValueChange   = { telefono = it },
            label           = { Text("Teléfono") },
            placeholder     = { Text("+593 99 999 9999") },
            leadingIcon     = { Icon(Icons.Default.Phone, contentDescription = null) },
            isError         = telefono.isNotEmpty() && !telefonoValido,
            supportingText  = {
                if (telefono.isNotEmpty() && !telefonoValido)
                    Text("Mínimo 7 dígitos",
                         color = MaterialTheme.colorScheme.error)
            },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Phone,
                imeAction    = ImeAction.Next
            ),
            singleLine  = true,
            modifier    = Modifier.fillMaxWidth()
        )

        // Contraseña — VisualTransformation oculta el texto
        // El trailingIcon alterna entre ocultar y mostrar
        OutlinedTextField(
            value           = contrasena,
            onValueChange   = { contrasena = it },
            label           = { Text("Contraseña") },
            leadingIcon     = { Icon(Icons.Default.Lock, contentDescription = null) },
            trailingIcon    = {
                IconButton(onClick = { verPass = !verPass }) {
                    Icon(
                        imageVector        = if (verPass) Icons.Default.VisibilityOff
                                             else Icons.Default.Visibility,
                        contentDescription = if (verPass) "Ocultar contraseña"
                                             else "Mostrar contraseña"
                    )
                }
            },
            // PasswordVisualTransformation reemplaza cada char por un punto
            // VisualTransformation.None muestra el texto tal cual
            visualTransformation = if (verPass) VisualTransformation.None
                                   else PasswordVisualTransformation(),
            isError         = contrasena.isNotEmpty() && !passValida,
            supportingText  = {
                Text(
                    text  = "${contrasena.length}/8 caracteres mínimos",
                    color = if (passValida) MaterialTheme.colorScheme.primary
                            else MaterialTheme.colorScheme.onSurfaceVariant
                )
            },
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Password,
                imeAction    = ImeAction.Done
            ),
            singleLine  = true,
            modifier    = Modifier.fillMaxWidth()
        )

        Button(
            onClick  = { /* Paso 6: mostrará un diálogo de confirmación */ },
            enabled  = formularioValido,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(if (formularioValido) "Guardar contacto ✓" else "Completa todos los campos")
        }
    }
}

@Preview(showBackground = true)
@Composable
fun Paso01_Preview() {
    MaterialTheme { Paso01_TextFieldScreen() }
}
```

**▶ Ejecuta:** descomenta `Paso01_TextFieldScreen()` en `MainActivity`.
- Escribe en el campo búsqueda → el ícono X aparece solo cuando hay texto
- Escribe una letra en Nombre → aparece el error; agrega otra letra → cambia a "✓ válido"
- Prueba el toggle de contraseña — alterna entre puntos y texto visible
- Completa todos los campos correctamente → el botón se activa

**Experimenta:**
- Cambia `KeyboardType.Email` por `KeyboardType.Text` y observa cómo cambia el teclado del sistema
- Agrega un campo multilinea para "Notas": usa `maxLines = 3` y elimina `singleLine = true`

---

## `Card` y `ElevatedCard`

### Concepto

`Card` es un contenedor con superficie, borde redondeado y sombra.
En Material 3 hay tres variantes:

| Composable | Apariencia | Cuándo usarlo |
|---|---|---|
| `Card` | Sin elevación visible | Contenido informativo plano |
| `ElevatedCard` | Sombra pronunciada | Ítem interactivo destacado |
| `OutlinedCard` | Solo borde | Listas densas, formularios |

`ElevatedCard` acepta `onClick` directamente, lo que lo hace el
contenedor interactivo estándar para ítems de lista.

### Paso 2 — Tarjeta de contacto

**Crea:** `ui/Paso02_Card.kt`

Construimos la `TarjetaContacto` que reutilizaremos en todos los pasos
siguientes. Aquí también introducimos `AssistChip` para mostrar etiquetas.

```kotlin
// ui/Paso02_Card.kt
package com.tuapp.contactos.ui

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import com.tuapp.contactos.model.Contacto
import com.tuapp.contactos.model.contactosDeMuestra

// ── Composable reutilizable — se usará desde el Paso 3 en adelante ───────────
@Composable
fun TarjetaContacto(
    contacto:  Contacto,
    onClick:   () -> Unit = {},
    onLlamar:  () -> Unit = {},
    onFavorito: () -> Unit = {}
) {
    // ElevatedCard con onClick → toda la tarjeta es presionable
    ElevatedCard(
        onClick  = onClick,
        modifier = Modifier.fillMaxWidth()
    ) {
        Row(
            modifier          = Modifier.padding(12.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            // ── Avatar con inicial ─────────────────────────────────────────
            // Box circular con clip(CircleShape) — patrón visto en página 11
            Box(
                modifier         = Modifier
                    .size(52.dp)
                    .clip(CircleShape)
                    .background(MaterialTheme.colorScheme.primaryContainer),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    text       = contacto.nombre.first().uppercase(),
                    style      = MaterialTheme.typography.titleLarge,
                    fontWeight = FontWeight.Bold,
                    color      = MaterialTheme.colorScheme.onPrimaryContainer
                )
            }

            Spacer(Modifier.width(12.dp))

            // ── Datos del contacto ─────────────────────────────────────────
            // weight(1f) → esta Column toma el espacio sobrante del Row
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text       = contacto.nombre,
                    style      = MaterialTheme.typography.titleSmall,
                    fontWeight = FontWeight.SemiBold
                )
                Text(
                    text  = contacto.email,
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
                Spacer(Modifier.height(4.dp))
                // AssistChip — etiqueta pequeña no interactiva para información extra
                AssistChip(
                    onClick = {},
                    label   = { Text(contacto.telefono,
                                    style = MaterialTheme.typography.labelSmall) }
                )
            }

            // ── Acciones rápidas ───────────────────────────────────────────
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                IconButton(onClick = onFavorito) {
                    Icon(
                        imageVector        = if (contacto.favorito) Icons.Default.Favorite
                                             else Icons.Default.FavoriteBorder,
                        contentDescription = if (contacto.favorito) "Quitar favorito"
                                             else "Marcar favorito",
                        tint               = if (contacto.favorito)
                            MaterialTheme.colorScheme.error
                        else
                            MaterialTheme.colorScheme.onSurfaceVariant
                    )
                }
                IconButton(onClick = onLlamar) {
                    Icon(
                        imageVector        = Icons.Default.Phone,
                        contentDescription = "Llamar",
                        tint               = MaterialTheme.colorScheme.primary
                    )
                }
            }
        }
    }
}

// ── Screen del paso 2: muestra las tarjetas con datos estáticos ──────────────
@Composable
fun Paso02_CardScreen() {
    Column(
        modifier            = Modifier.fillMaxSize().padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        Text("Paso 2 · Card y ElevatedCard",
             style = MaterialTheme.typography.titleMedium)
        HorizontalDivider()

        Text("ElevatedCard — ítem interactivo",
             style = MaterialTheme.typography.labelMedium,
             color = MaterialTheme.colorScheme.primary)

        // Mostramos los primeros 3 contactos de la lista de muestra
        contactosDeMuestra.take(3).forEach { contacto ->
            TarjetaContacto(
                contacto  = contacto,
                onClick   = { /* En el Paso 6: navegar al detalle */ },
                onLlamar  = { /* En el Paso 6: mostrar snackbar */ },
                onFavorito = { /* En el Paso 3: toggle en la lista */ }
            )
        }

        HorizontalDivider()
        Text("Comparación de variantes",
             style = MaterialTheme.typography.labelMedium,
             color = MaterialTheme.colorScheme.primary)

        // Card plana
        Card(modifier = Modifier.fillMaxWidth()) {
            Text("Card — sin elevación visible",
                 Modifier.padding(16.dp))
        }

        // ElevatedCard
        ElevatedCard(modifier = Modifier.fillMaxWidth()) {
            Text("ElevatedCard — con sombra",
                 Modifier.padding(16.dp))
        }

        // OutlinedCard
        OutlinedCard(modifier = Modifier.fillMaxWidth()) {
            Text("OutlinedCard — solo borde",
                 Modifier.padding(16.dp))
        }
    }
}

@Preview(showBackground = true)
@Composable
fun Paso02_Preview() {
    MaterialTheme { Paso02_CardScreen() }
}
```

**▶ Ejecuta:** descomenta `Paso02_CardScreen()`. Presiona las tarjetas
y los íconos — observa que aún no hacen nada; los callbacks se conectarán
en pasos siguientes.

**Experimenta:**
- Cambia `ElevatedCard` por `OutlinedCard` en `TarjetaContacto` → ¿cuál prefieres visualmente?
- Agrega un `Badge` sobre el ícono de favorito que muestre "♥" cuando `contacto.favorito = true`

---

## `LazyColumn` y `LazyRow` — listas virtualizadas

### Concepto

`LazyColumn` es el equivalente de `RecyclerView` en Compose.
La palabra "Lazy" significa que **solo renderiza los elementos visibles**
en pantalla — no crea views para toda la lista de golpe.

```
┌──────────────────────┐
│  item { Cabecera }   │  ← siempre renderizado
├──────────────────────┤
│  items(lista) {      │  ← solo los visibles en pantalla
│    TarjetaContacto() │
│    TarjetaContacto() │
│    TarjetaContacto() │
├──────────────────────┤  ← scroll
│    (no renderizado)  │
│    (no renderizado)  │
└──────────────────────┘
```

Parámetros clave de `LazyColumn`:

| Parámetro | Propósito |
|---|---|
| `contentPadding` | Padding al inicio y final de la lista (no afecta el scroll) |
| `verticalArrangement` | Espaciado entre ítems |
| `key` | Identifica cada ítem — mejora rendimiento al insertar/eliminar |

> **`key` es importante:** sin él, al eliminar el ítem 2 de una lista de 10,
> Compose recompone todos los ítems 2–10. Con `key = { it.id }`, solo
> recompone el ítem eliminado.

### Paso 3 — Lista virtualizada con filtro

**Crea:** `ui/Paso03_LazyColumn.kt`

Agregamos estado mutable a la lista (favoritos y eliminación) y conectamos
los callbacks de `TarjetaContacto` del paso anterior.

```kotlin
// ui/Paso03_LazyColumn.kt
package com.tuapp.contactos.ui

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import com.tuapp.contactos.model.Contacto
import com.tuapp.contactos.model.contactosDeMuestra

@Composable
fun Paso03_LazyColumnScreen() {
    // Estado mutable de la lista — usamos mutableStateOf con una lista
    // Al reasignar la lista, Compose detecta el cambio y recompone
    var contactos by remember { mutableStateOf(contactosDeMuestra) }
    var busqueda  by remember { mutableStateOf("") }
    var filtro    by remember { mutableStateOf("Todos") }

    // Derivamos la lista filtrada — se recalcula en cada recomposición
    val contactosFiltrados = contactos
        .filter { c ->
            when (filtro) {
                "Favoritos" -> c.favorito
                else        -> true
            }
        }
        .filter { c ->
            busqueda.isBlank() ||
            c.nombre.contains(busqueda, ignoreCase = true) ||
            c.email.contains(busqueda, ignoreCase = true)
        }

    Column(modifier = Modifier.fillMaxSize()) {
        Text(
            "Paso 3 · LazyColumn + LazyRow",
            style    = MaterialTheme.typography.titleMedium,
            modifier = Modifier.padding(16.dp)
        )

        // ── Campo de búsqueda (del Paso 1) ───────────────────────────────
        OutlinedTextField(
            value         = busqueda,
            onValueChange = { busqueda = it },
            placeholder   = { Text("Buscar...") },
            leadingIcon   = { Icon(Icons.Default.Search, contentDescription = null) },
            trailingIcon  = {
                if (busqueda.isNotEmpty())
                    IconButton(onClick = { busqueda = "" }) {
                        Icon(Icons.Default.Clear, contentDescription = "Limpiar")
                    }
            },
            singleLine = true,
            modifier   = Modifier.fillMaxWidth().padding(horizontal = 16.dp)
        )

        Spacer(Modifier.height(8.dp))

        // ── LazyRow: filtros de categoría ────────────────────────────────
        // LazyRow = Row virtualizado → ideal para chips horizontales con scroll
        LazyRow(
            horizontalArrangement = Arrangement.spacedBy(8.dp),
            contentPadding        = PaddingValues(horizontal = 16.dp)
        ) {
            items(listOf("Todos", "Favoritos")) { opcion ->
                FilterChip(
                    selected = filtro == opcion,
                    onClick  = { filtro = opcion },
                    label    = { Text(opcion) },
                    // leadingIcon cambia según si está seleccionado
                    leadingIcon = if (filtro == opcion) {{
                        Icon(Icons.Default.Check,
                             contentDescription = null,
                             modifier = Modifier.size(FilterChipDefaults.IconSize))
                    }} else null
                )
            }
        }

        Spacer(Modifier.height(8.dp))

        // ── LazyColumn: lista principal ──────────────────────────────────
        if (contactosFiltrados.isEmpty()) {
            // Estado vacío — buena práctica siempre manejar lista vacía
            Box(
                modifier         = Modifier.fillMaxSize(),
                contentAlignment = Alignment.Center
            ) {
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    Icon(
                        Icons.Default.SearchOff,
                        contentDescription = null,
                        modifier = Modifier.size(56.dp),
                        tint     = MaterialTheme.colorScheme.onSurfaceVariant
                    )
                    Spacer(Modifier.height(12.dp))
                    Text("Sin resultados",
                         style = MaterialTheme.typography.bodyLarge,
                         color = MaterialTheme.colorScheme.onSurfaceVariant)
                }
            }
        } else {
            LazyColumn(
                // contentPadding: espacio en los bordes de la lista
                // No confundir con padding del Modifier — este respeta el scroll
                contentPadding      = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                // item { } → elemento único (cabecera, separador, etc.)
                item {
                    Text(
                        "${contactosFiltrados.size} contacto(s)",
                        style    = MaterialTheme.typography.labelSmall,
                        color    = MaterialTheme.colorScheme.onSurfaceVariant,
                        modifier = Modifier.padding(bottom = 4.dp)
                    )
                }

                // items(lista, key) → renderiza un composable por cada elemento
                // key = { it.id } → Compose identifica cada ítem por su id,
                // no por su posición — crítico para animaciones y rendimiento
                items(
                    items = contactosFiltrados,
                    key   = { it.id }
                ) { contacto ->
                    TarjetaContacto(
                        contacto  = contacto,
                        onLlamar  = { /* Paso 6: snackbar */ },
                        onFavorito = {
                            // Creamos nueva lista con el favorito modificado
                            // Las listas en Kotlin son inmutables por defecto —
                            // reasignamos en lugar de mutar
                            contactos = contactos.map { c ->
                                if (c.id == contacto.id) c.copy(favorito = !c.favorito)
                                else c
                            }
                        }
                    )
                }

                // item de pie — espacio para futura NavigationBar
                item { Spacer(Modifier.height(16.dp)) }
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun Paso03_Preview() {
    MaterialTheme { Paso03_LazyColumnScreen() }
}
```

**▶ Ejecuta:** descomenta `Paso03_LazyColumnScreen()`.
- Toca el corazón en cualquier tarjeta → cambia de estado inmediatamente
- Escribe en el buscador → la lista filtra en tiempo real
- Selecciona "Favoritos" en el chip → solo muestra los marcados
- Combina búsqueda + filtro de favoritos

**Experimenta:**
- Elimina el parámetro `key = { it.id }` en `items(...)` → observa en el
  logcat cómo Compose marca más recomposiciones (visible con Layout Inspector)
- Agrega un tercer chip "Sin favoritos" con la lógica `!c.favorito`

---

## `Scaffold` — estructura completa de pantalla

### Concepto

`Scaffold` implementa el layout estándar de Material 3. Gestiona
automáticamente los `insets` del sistema (barra de estado, barra de
navegación del sistema) para que tu contenido no quede oculto.

```
┌─────────────────────────────────┐
│  topBar: TopAppBar              │ ← slot opcional
├─────────────────────────────────┤
│                                 │
│  content (paddingValues)        │ ← obligatorio
│                                 │   paddingValues SIEMPRE debe aplicarse
│                         [FAB]   │ ← floatingActionButton: slot opcional
├─────────────────────────────────┤
│  bottomBar: NavigationBar       │ ← slot opcional (Paso 5)
└─────────────────────────────────┘
```

> **Regla crítica:** el lambda `content` recibe `paddingValues`. Debes
> aplicarlo con `Modifier.padding(paddingValues)` en el contenido raíz.
> Si no lo haces, el contenido quedará debajo de la `TopAppBar`.

### Paso 4 — Scaffold con TopAppBar y FAB

**Crea:** `ui/Paso04_Scaffold.kt`

Envolvemos la lista del Paso 3 en un `Scaffold` con `TopAppBar` y `FAB`.
El estado de búsqueda se eleva aquí para que la `TopAppBar` también pueda leerlo.

```kotlin
// ui/Paso04_Scaffold.kt
package com.tuapp.contactos.ui

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import com.tuapp.contactos.model.contactosDeMuestra

@Composable
fun Paso04_ScaffoldScreen() {
    var contactos  by remember { mutableStateOf(contactosDeMuestra) }
    var busqueda   by remember { mutableStateOf("") }
    var filtro     by remember { mutableStateOf("Todos") }
    var mostrarFab by remember { mutableStateOf(false) } // feedback del FAB

    val contactosFiltrados = contactos
        .filter { c -> if (filtro == "Favoritos") c.favorito else true }
        .filter { c ->
            busqueda.isBlank() ||
            c.nombre.contains(busqueda, ignoreCase = true)
        }

    Scaffold(
        // ── TopAppBar ──────────────────────────────────────────────────────
        topBar = {
            TopAppBar(
                title = {
                    // El título muestra el conteo — estado elevado al Scaffold
                    Text(
                        "Contactos (${contactos.size})",
                        fontWeight = FontWeight.Bold
                    )
                },
                // actions: iconos a la derecha del título
                actions = {
                    // Ícono de favoritos como acceso rápido
                    IconButton(onClick = {
                        filtro = if (filtro == "Favoritos") "Todos" else "Favoritos"
                    }) {
                        Icon(
                            imageVector        = if (filtro == "Favoritos")
                                Icons.Default.Favorite else Icons.Default.FavoriteBorder,
                            contentDescription = "Filtrar favoritos",
                            tint               = if (filtro == "Favoritos")
                                MaterialTheme.colorScheme.error
                            else
                                MaterialTheme.colorScheme.onSurface
                        )
                    }
                },
                // colors: personaliza los colores de la barra
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.onPrimaryContainer
                )
            )
        },

        // ── FloatingActionButton ───────────────────────────────────────────
        floatingActionButton = {
            FloatingActionButton(
                onClick = { mostrarFab = true }
            ) {
                Icon(Icons.Default.PersonAdd, contentDescription = "Nuevo contacto")
            }
        }

    // ── Content ────────────────────────────────────────────────────────────
    // paddingValues contiene el espacio que ocupa topBar y bottomBar
    // SIEMPRE aplícalo al contenido — de lo contrario queda bajo la TopAppBar
    ) { paddingValues ->
        Column(
            modifier = Modifier
                .padding(paddingValues)  // ← CRÍTICO: aplica el padding del Scaffold
                .fillMaxSize()
        ) {
            // Búsqueda
            OutlinedTextField(
                value         = busqueda,
                onValueChange = { busqueda = it },
                placeholder   = { Text("Buscar contacto...") },
                leadingIcon   = { Icon(Icons.Default.Search, null) },
                trailingIcon  = {
                    if (busqueda.isNotEmpty())
                        IconButton(onClick = { busqueda = "" }) {
                            Icon(Icons.Default.Clear, "Limpiar")
                        }
                },
                singleLine = true,
                modifier   = Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 8.dp)
            )

            // Chips de filtro
            LazyRow(
                horizontalArrangement = Arrangement.spacedBy(8.dp),
                contentPadding        = PaddingValues(horizontal = 16.dp)
            ) {
                items(listOf("Todos", "Favoritos")) { opcion ->
                    FilterChip(
                        selected = filtro == opcion,
                        onClick  = { filtro = opcion },
                        label    = { Text(opcion) },
                        leadingIcon = if (filtro == opcion) {{
                            Icon(Icons.Default.Check, null,
                                 Modifier.size(FilterChipDefaults.IconSize))
                        }} else null
                    )
                }
            }

            Spacer(Modifier.height(4.dp))

            // Lista
            LazyColumn(
                contentPadding      = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                item {
                    Text("${contactosFiltrados.size} resultado(s)",
                         style    = MaterialTheme.typography.labelSmall,
                         color    = MaterialTheme.colorScheme.onSurfaceVariant,
                         modifier = Modifier.padding(bottom = 4.dp))
                }
                items(contactosFiltrados, key = { it.id }) { contacto ->
                    TarjetaContacto(
                        contacto  = contacto,
                        onFavorito = {
                            contactos = contactos.map { c ->
                                if (c.id == contacto.id) c.copy(favorito = !c.favorito) else c
                            }
                        }
                    )
                }
                item { Spacer(Modifier.height(80.dp)) } // espacio para FAB
            }
        }
    }

    // Feedback temporal del FAB — en el Paso 6 se reemplaza por un diálogo real
    if (mostrarFab) {
        AlertDialog(
            onDismissRequest = { mostrarFab = false },
            title   = { Text("Nuevo contacto") },
            text    = { Text("Esta función se conectará con el formulario del Paso 1 en el Paso 6.") },
            confirmButton = {
                TextButton(onClick = { mostrarFab = false }) { Text("OK") }
            }
        )
    }
}

@Preview(showBackground = true)
@Composable
fun Paso04_Preview() {
    MaterialTheme { Paso04_ScaffoldScreen() }
}
```

**▶ Ejecuta:** descomenta `Paso04_ScaffoldScreen()`.
- Observa que el contenido **no** queda debajo de la `TopAppBar` — gracias al `paddingValues`
- El ícono de corazón en la barra alterna el filtro de favoritos
- Presiona el FAB → aparece el diálogo temporal

**Experimenta:**
- Comenta la línea `.padding(paddingValues)` del Column → el contenido
  queda oculto debajo de la TopAppBar
- Cambia `containerColor` de la `TopAppBar` por `MaterialTheme.colorScheme.surface`

---

## `NavigationBar` — barra de navegación inferior

### Concepto

`NavigationBar` implementa la barra inferior de Material 3. Cada destino
usa `NavigationBarItem` con un ícono activo (relleno) y uno inactivo (contorno) —
convención visual de Material 3 para indicar cuál pestaña está seleccionada.

```kotlin
// Patrón estándar de NavigationBarItem
NavigationBarItem(
    selected = destinoActual == "productos",
    onClick  = { destinoActual = "productos" },
    icon     = {
        Icon(
            // Ícono relleno si seleccionado, contorno si no
            imageVector = if (seleccionado) Icons.Filled.Inventory
                          else Icons.Outlined.Inventory,
            ...
        )
    },
    label = { Text("Productos") }
)
```

### Paso 5 — Scaffold con NavigationBar

**Crea:** `ui/Paso05_NavBar.kt`

Agregamos navegación inferior con tres pestañas. Cada pestaña muestra
una pantalla diferente — por ahora texto simple; en la página 13 usaremos
Navigation Compose para hacerlo escalable.

```kotlin
// ui/Paso05_NavBar.kt
package com.tuapp.contactos.ui

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material.icons.outlined.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import com.tuapp.contactos.model.Contacto
import com.tuapp.contactos.model.contactosDeMuestra

// Modelo del destino de navegación
// Separar datos de presentación es buena práctica (SRP)
data class DestinoNav(
    val ruta:          String,
    val etiqueta:      String,
    val iconoActivo:   ImageVector,
    val iconoInactivo: ImageVector
)

@Composable
fun Paso05_NavBarScreen() {
    var destinoActual by remember { mutableStateOf("contactos") }
    var contactos     by remember { mutableStateOf(contactosDeMuestra) }

    val destinos = listOf(
        DestinoNav("contactos", "Contactos", Icons.Filled.People,       Icons.Outlined.People),
        DestinoNav("favoritos", "Favoritos", Icons.Filled.Favorite,     Icons.Outlined.FavoriteBorder),
        DestinoNav("perfil",    "Perfil",    Icons.Filled.AccountCircle, Icons.Outlined.AccountCircle),
    )

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Agenda", fontWeight = FontWeight.Bold) },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor    = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.onPrimaryContainer
                )
            )
        },

        // ── NavigationBar ──────────────────────────────────────────────────
        bottomBar = {
            NavigationBar {
                destinos.forEach { destino ->
                    val seleccionado = destinoActual == destino.ruta
                    NavigationBarItem(
                        selected = seleccionado,
                        onClick  = { destinoActual = destino.ruta },
                        icon     = {
                            // Convención M3: relleno = activo, contorno = inactivo
                            Icon(
                                imageVector        = if (seleccionado) destino.iconoActivo
                                                     else destino.iconoInactivo,
                                contentDescription = destino.etiqueta
                            )
                        },
                        label = { Text(destino.etiqueta) }
                    )
                }
            }
        },

        floatingActionButton = {
            // FAB solo visible en la pestaña de contactos
            if (destinoActual == "contactos") {
                FloatingActionButton(onClick = { /* Paso 6 */ }) {
                    Icon(Icons.Default.PersonAdd, "Nuevo contacto")
                }
            }
        }

    ) { paddingValues ->
        // when actúa como router simple — en pág. 13 usaremos NavHost
        when (destinoActual) {
            "contactos" -> PantallaContactosContent(
                contactos  = contactos,
                onFavorito = { id ->
                    contactos = contactos.map { c ->
                        if (c.id == id) c.copy(favorito = !c.favorito) else c
                    }
                },
                modifier   = Modifier.padding(paddingValues)
            )
            "favoritos" -> PantallaFavoritosContent(
                favoritos = contactos.filter { it.favorito },
                modifier  = Modifier.padding(paddingValues)
            )
            "perfil"    -> PantallaPerfilContent(
                modifier  = Modifier.padding(paddingValues)
            )
        }
    }
}

// ── Contenido de cada pestaña ────────────────────────────────────────────────

@Composable
private fun PantallaContactosContent(
    contactos:  List<Contacto>,
    onFavorito: (Int) -> Unit,
    modifier:   Modifier = Modifier
) {
    LazyColumn(
        modifier            = modifier,
        contentPadding      = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(contactos, key = { it.id }) { contacto ->
            TarjetaContacto(
                contacto   = contacto,
                onFavorito = { onFavorito(contacto.id) }
            )
        }
        item { Spacer(Modifier.height(80.dp)) }
    }
}

@Composable
private fun PantallaFavoritosContent(
    favoritos: List<Contacto>,
    modifier:  Modifier = Modifier
) {
    if (favoritos.isEmpty()) {
        Box(modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                Icon(Icons.Default.FavoriteBorder, null,
                     Modifier.size(56.dp), tint = MaterialTheme.colorScheme.onSurfaceVariant)
                Spacer(Modifier.height(12.dp))
                Text("Sin favoritos aún",
                     style = MaterialTheme.typography.bodyLarge,
                     color = MaterialTheme.colorScheme.onSurfaceVariant)
                Text("Toca el corazón en un contacto",
                     style = MaterialTheme.typography.bodySmall,
                     color = MaterialTheme.colorScheme.onSurfaceVariant)
            }
        }
    } else {
        LazyColumn(
            modifier            = modifier,
            contentPadding      = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(favoritos, key = { it.id }) { contacto ->
                TarjetaContacto(contacto = contacto)
            }
        }
    }
}

@Composable
private fun PantallaPerfilContent(modifier: Modifier = Modifier) {
    Box(modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Icon(Icons.Default.AccountCircle, null, Modifier.size(80.dp),
                 tint = MaterialTheme.colorScheme.primary)
            Spacer(Modifier.height(12.dp))
            Text("Mi Perfil", style = MaterialTheme.typography.titleLarge,
                 fontWeight = FontWeight.Bold)
            Text("Próximamente...",
                 style = MaterialTheme.typography.bodyMedium,
                 color = MaterialTheme.colorScheme.onSurfaceVariant)
        }
    }
}

@Preview(showBackground = true)
@Composable
fun Paso05_Preview() {
    MaterialTheme { Paso05_NavBarScreen() }
}
```

**▶ Ejecuta:** descomenta `Paso05_NavBarScreen()`.
- Navega entre las tres pestañas → el ícono cambia entre relleno y contorno
- Marca favoritos en la pestaña "Contactos" → aparecen en "Favoritos"
- El FAB desaparece cuando estás en "Favoritos" o "Perfil"

**Experimenta:**
- Agrega una cuarta pestaña "Configuración" con `Icons.Filled.Settings` / `Icons.Outlined.Settings`
- Mueve el FAB al centro de la `NavigationBar` usando `floatingActionButtonPosition = FabPosition.Center`

---

## `AlertDialog`, `Dialog` y `SnackbarHost`

### Concepto

Material 3 ofrece tres niveles de feedback al usuario:

| Componente | Cuándo usarlo |
|---|---|
| `AlertDialog` | Confirmaciones destructivas, decisiones importantes |
| `Dialog` | Contenido personalizado (formularios, selectors) |
| `Snackbar` | Feedback breve no bloqueante de acciones completadas |

`AlertDialog` estándar:
```kotlin
AlertDialog(
    onDismissRequest = { /* al tocar fuera o presionar atrás */ },
    icon    = { Icon(...) },          // opcional
    title   = { Text("Título") },
    text    = { Text("Mensaje") },
    confirmButton = { Button(...) },   // acción principal
    dismissButton = { OutlinedButton(...) }  // cancelar (opcional)
)
```

Para mostrar un `Snackbar`, necesitas dos piezas:
- `SnackbarHostState` — el estado que controla qué mensaje mostrar
- `LaunchedEffect` — efecto de Compose para llamar `showSnackbar()` (es una
  función suspendida que debe ejecutarse en una coroutine)

```kotlin
val snackbarHostState = remember { SnackbarHostState() }

// Reacciona cuando 'mensajeSnack' cambia
LaunchedEffect(mensajeSnack) {
    mensajeSnack?.let {
        snackbarHostState.showSnackbar(it)
        mensajeSnack = null   // limpia después de mostrar
    }
}

Scaffold(
    snackbarHost = { SnackbarHost(snackbarHostState) }
) { ... }
```

> **`LaunchedEffect`** es el primer "efecto" de Compose que vemos. Ejecuta
> un bloque de código cuando una de sus claves cambia. En la página 13
> profundizaremos en los efectos de Compose.

### Paso 6 — App completa con diálogos y snackbar

**Crea:** `ui/Paso06_Dialogos.kt`

Integramos todo: el formulario del Paso 1 en un diálogo, confirmación de
eliminación, y snackbar para feedback de acciones.

```kotlin
// ui/Paso06_Dialogos.kt
package com.tuapp.contactos.ui

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material.icons.outlined.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.input.KeyboardType
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.compose.ui.window.Dialog
import com.tuapp.contactos.model.Contacto
import com.tuapp.contactos.model.contactosDeMuestra

@Composable
fun Paso06_DialogosScreen() {
    var contactos        by remember { mutableStateOf(contactosDeMuestra) }
    var busqueda         by remember { mutableStateOf("") }
    var filtro           by remember { mutableStateOf("Todos") }
    var destinoActual    by remember { mutableStateOf("contactos") }

    // Estado de los diálogos — cada booleano controla visibilidad de un dialog
    var mostrarNuevo     by remember { mutableStateOf(false) }
    var contactoAEliminar by remember { mutableStateOf<Contacto?>(null) }

    // Snackbar: un String nullable — null = nada que mostrar
    var mensajeSnack     by remember { mutableStateOf<String?>(null) }
    val snackbarHostState = remember { SnackbarHostState() }

    // LaunchedEffect: ejecuta el bloque cada vez que 'mensajeSnack' cambia
    // Es una coroutine — showSnackbar() suspende hasta que el snackbar desaparece
    LaunchedEffect(mensajeSnack) {
        mensajeSnack?.let {
            snackbarHostState.showSnackbar(it)
            mensajeSnack = null
        }
    }

    val contactosFiltrados = contactos
        .filter { c -> if (filtro == "Favoritos") c.favorito else true }
        .filter { c -> busqueda.isBlank() || c.nombre.contains(busqueda, ignoreCase = true) }

    val destinos = listOf(
        DestinoNav("contactos", "Contactos", Icons.Filled.People,       Icons.Outlined.People),
        DestinoNav("favoritos", "Favoritos", Icons.Filled.Favorite,     Icons.Outlined.FavoriteBorder),
        DestinoNav("perfil",    "Perfil",    Icons.Filled.AccountCircle, Icons.Outlined.AccountCircle),
    )

    Scaffold(
        topBar = {
            TopAppBar(
                title = {
                    Text("Agenda (${contactos.size})", fontWeight = FontWeight.Bold)
                },
                actions = {
                    IconButton(onClick = {
                        filtro = if (filtro == "Favoritos") "Todos" else "Favoritos"
                    }) {
                        Icon(
                            imageVector = if (filtro == "Favoritos")
                                Icons.Default.Favorite else Icons.Default.FavoriteBorder,
                            contentDescription = "Filtrar favoritos",
                            tint = if (filtro == "Favoritos")
                                MaterialTheme.colorScheme.error
                            else
                                MaterialTheme.colorScheme.onPrimaryContainer
                        )
                    }
                },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor    = MaterialTheme.colorScheme.primaryContainer,
                    titleContentColor = MaterialTheme.colorScheme.onPrimaryContainer
                )
            )
        },
        bottomBar = {
            NavigationBar {
                destinos.forEach { destino ->
                    val sel = destinoActual == destino.ruta
                    NavigationBarItem(
                        selected = sel,
                        onClick  = { destinoActual = destino.ruta },
                        icon     = {
                            Icon(if (sel) destino.iconoActivo else destino.iconoInactivo,
                                 destino.etiqueta)
                        },
                        label = { Text(destino.etiqueta) }
                    )
                }
            }
        },
        floatingActionButton = {
            if (destinoActual == "contactos") {
                FloatingActionButton(onClick = { mostrarNuevo = true }) {
                    Icon(Icons.Default.PersonAdd, "Nuevo contacto")
                }
            }
        },
        // snackbarHost conecta el SnackbarHostState con el Scaffold
        snackbarHost = { SnackbarHost(snackbarHostState) }

    ) { paddingValues ->
        when (destinoActual) {
            "contactos" -> ContenidoContactos(
                contactos    = contactosFiltrados,
                busqueda     = busqueda,
                filtro       = filtro,
                onBusqueda   = { busqueda = it },
                onFiltro     = { filtro = it },
                onFavorito   = { id ->
                    contactos = contactos.map { c ->
                        if (c.id == id) c.copy(favorito = !c.favorito) else c
                    }
                },
                onLlamar     = { nombre -> mensajeSnack = "📞 Llamando a $nombre..." },
                onEliminar   = { contacto -> contactoAEliminar = contacto },
                modifier     = Modifier.padding(paddingValues)
            )
            "favoritos" -> PantallaFavoritosContent(
                favoritos = contactos.filter { it.favorito },
                modifier  = Modifier.padding(paddingValues)
            )
            "perfil"    -> PantallaPerfilContent(
                modifier  = Modifier.padding(paddingValues)
            )
        }
    }

    // ── Diálogo 1: Nuevo contacto (Dialog personalizado) ────────────────────
    // Usamos Dialog (no AlertDialog) para tener control total del contenido
    if (mostrarNuevo) {
        DialogNuevoContacto(
            onDismiss = { mostrarNuevo = false },
            onGuardar = { nuevo ->
                contactos    = contactos + nuevo
                mostrarNuevo = false
                mensajeSnack = "✅ ${nuevo.nombre} agregado"
            }
        )
    }

    // ── Diálogo 2: Confirmar eliminación (AlertDialog estándar) ─────────────
    // contactoAEliminar != null → mostrar dialog; null → ocultarlo
    contactoAEliminar?.let { contacto ->
        AlertDialog(
            onDismissRequest = { contactoAEliminar = null },
            icon    = {
                Icon(Icons.Default.Warning, null,
                     tint = MaterialTheme.colorScheme.error)
            },
            title   = { Text("Eliminar contacto") },
            text    = {
                Text("¿Seguro que quieres eliminar a ${contacto.nombre}? " +
                     "Esta acción no se puede deshacer.")
            },
            confirmButton = {
                Button(
                    onClick = {
                        contactos         = contactos.filter { it.id != contacto.id }
                        mensajeSnack      = "🗑 ${contacto.nombre} eliminado"
                        contactoAEliminar = null
                    },
                    colors = ButtonDefaults.buttonColors(
                        containerColor = MaterialTheme.colorScheme.error
                    )
                ) { Text("Eliminar") }
            },
            dismissButton = {
                OutlinedButton(onClick = { contactoAEliminar = null }) {
                    Text("Cancelar")
                }
            }
        )
    }
}

// ── Contenido de la pestaña Contactos ───────────────────────────────────────

@Composable
private fun ContenidoContactos(
    contactos:  List<Contacto>,
    busqueda:   String,
    filtro:     String,
    onBusqueda: (String) -> Unit,
    onFiltro:   (String) -> Unit,
    onFavorito: (Int) -> Unit,
    onLlamar:   (String) -> Unit,
    onEliminar: (Contacto) -> Unit,
    modifier:   Modifier = Modifier
) {
    Column(modifier = modifier.fillMaxSize()) {
        OutlinedTextField(
            value         = busqueda,
            onValueChange = onBusqueda,
            placeholder   = { Text("Buscar contacto...") },
            leadingIcon   = { Icon(Icons.Default.Search, null) },
            trailingIcon  = {
                if (busqueda.isNotEmpty())
                    IconButton(onClick = { onBusqueda("") }) {
                        Icon(Icons.Default.Clear, "Limpiar")
                    }
            },
            singleLine = true,
            modifier   = Modifier.fillMaxWidth().padding(horizontal = 16.dp, vertical = 8.dp)
        )

        LazyRow(
            horizontalArrangement = Arrangement.spacedBy(8.dp),
            contentPadding        = PaddingValues(horizontal = 16.dp)
        ) {
            items(listOf("Todos", "Favoritos")) { opcion ->
                FilterChip(
                    selected    = filtro == opcion,
                    onClick     = { onFiltro(opcion) },
                    label       = { Text(opcion) },
                    leadingIcon = if (filtro == opcion) {{
                        Icon(Icons.Default.Check, null,
                             Modifier.size(FilterChipDefaults.IconSize))
                    }} else null
                )
            }
        }

        Spacer(Modifier.height(4.dp))

        if (contactos.isEmpty()) {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    Icon(Icons.Default.SearchOff, null, Modifier.size(56.dp),
                         tint = MaterialTheme.colorScheme.onSurfaceVariant)
                    Spacer(Modifier.height(8.dp))
                    Text("Sin resultados",
                         style = MaterialTheme.typography.bodyLarge,
                         color = MaterialTheme.colorScheme.onSurfaceVariant)
                }
            }
        } else {
            LazyColumn(
                contentPadding      = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                item {
                    Text("${contactos.size} contacto(s)",
                         style    = MaterialTheme.typography.labelSmall,
                         color    = MaterialTheme.colorScheme.onSurfaceVariant,
                         modifier = Modifier.padding(bottom = 4.dp))
                }
                items(contactos, key = { it.id }) { contacto ->
                    // TarjetaContacto ahora con TODOS los callbacks conectados
                    TarjetaContactoCompleta(
                        contacto   = contacto,
                        onFavorito = { onFavorito(contacto.id) },
                        onLlamar   = { onLlamar(contacto.nombre) },
                        onEliminar = { onEliminar(contacto) }
                    )
                }
                item { Spacer(Modifier.height(100.dp)) }
            }
        }
    }
}

// TarjetaContacto extendida con botón de eliminar
@Composable
private fun TarjetaContactoCompleta(
    contacto:  Contacto,
    onFavorito: () -> Unit,
    onLlamar:  () -> Unit,
    onEliminar: () -> Unit
) {
    ElevatedCard(modifier = Modifier.fillMaxWidth()) {
        Row(
            modifier          = Modifier.padding(12.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            androidx.compose.foundation.layout.Box(
                modifier = Modifier
                    .size(52.dp)
                    .androidx.compose.ui.draw.clip(androidx.compose.foundation.shape.CircleShape)
                    .androidx.compose.foundation.background(MaterialTheme.colorScheme.primaryContainer),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    contacto.nombre.first().uppercase(),
                    style      = MaterialTheme.typography.titleLarge,
                    fontWeight = FontWeight.Bold,
                    color      = MaterialTheme.colorScheme.onPrimaryContainer
                )
            }
            Spacer(Modifier.width(12.dp))
            Column(Modifier.weight(1f)) {
                Text(contacto.nombre, fontWeight = FontWeight.SemiBold,
                     style = MaterialTheme.typography.titleSmall)
                Text(contacto.email,
                     style = MaterialTheme.typography.bodySmall,
                     color = MaterialTheme.colorScheme.onSurfaceVariant)
                Text(contacto.telefono,
                     style = MaterialTheme.typography.bodySmall,
                     color = MaterialTheme.colorScheme.onSurfaceVariant)
            }
            IconButton(onClick = onFavorito) {
                Icon(
                    if (contacto.favorito) Icons.Default.Favorite else Icons.Default.FavoriteBorder,
                    null,
                    tint = if (contacto.favorito) MaterialTheme.colorScheme.error
                           else MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
            IconButton(onClick = onLlamar) {
                Icon(Icons.Default.Phone, null,
                     tint = MaterialTheme.colorScheme.primary)
            }
            IconButton(onClick = onEliminar) {
                Icon(Icons.Default.Delete, null,
                     tint = MaterialTheme.colorScheme.error)
            }
        }
    }
}

// ── Diálogo personalizado: formulario de nuevo contacto ──────────────────────
@Composable
private fun DialogNuevoContacto(
    onDismiss: () -> Unit,
    onGuardar: (Contacto) -> Unit
) {
    var nombre   by remember { mutableStateOf("") }
    var email    by remember { mutableStateOf("") }
    var telefono by remember { mutableStateOf("") }

    val nombreValido   = nombre.trim().length >= 2
    val emailValido    = email.contains("@") && email.contains(".")
    val telefonoValido = telefono.length >= 7
    val valido         = nombreValido && emailValido && telefonoValido

    // Dialog (no AlertDialog) → contenido completamente personalizado
    Dialog(onDismissRequest = onDismiss) {
        Card(modifier = Modifier.fillMaxWidth()) {
            Column(
                modifier            = Modifier.padding(24.dp),
                verticalArrangement = Arrangement.spacedBy(12.dp)
            ) {
                Text("Nuevo contacto",
                     style      = MaterialTheme.typography.titleLarge,
                     fontWeight = FontWeight.Bold)

                OutlinedTextField(
                    value           = nombre,
                    onValueChange   = { nombre = it },
                    label           = { Text("Nombre") },
                    leadingIcon     = { Icon(Icons.Default.Person, null) },
                    isError         = nombre.isNotEmpty() && !nombreValido,
                    singleLine      = true,
                    modifier        = Modifier.fillMaxWidth(),
                    keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next)
                )

                OutlinedTextField(
                    value           = email,
                    onValueChange   = { email = it },
                    label           = { Text("Email") },
                    leadingIcon     = { Icon(Icons.Default.Email, null) },
                    isError         = email.isNotEmpty() && !emailValido,
                    singleLine      = true,
                    modifier        = Modifier.fillMaxWidth(),
                    keyboardOptions = KeyboardOptions(
                        keyboardType = KeyboardType.Email,
                        imeAction    = ImeAction.Next
                    )
                )

                OutlinedTextField(
                    value           = telefono,
                    onValueChange   = { telefono = it },
                    label           = { Text("Teléfono") },
                    leadingIcon     = { Icon(Icons.Default.Phone, null) },
                    isError         = telefono.isNotEmpty() && !telefonoValido,
                    singleLine      = true,
                    modifier        = Modifier.fillMaxWidth(),
                    keyboardOptions = KeyboardOptions(
                        keyboardType = KeyboardType.Phone,
                        imeAction    = ImeAction.Done
                    )
                )

                Row(
                    modifier              = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.End,
                    verticalAlignment     = Alignment.CenterVertically
                ) {
                    TextButton(onClick = onDismiss) { Text("Cancelar") }
                    Spacer(Modifier.width(8.dp))
                    Button(
                        onClick  = {
                            onGuardar(
                                Contacto(
                                    id       = System.currentTimeMillis().toInt(),
                                    nombre   = nombre.trim(),
                                    email    = email.trim(),
                                    telefono = telefono.trim()
                                )
                            )
                        },
                        enabled  = valido
                    ) { Text("Guardar") }
                }
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun Paso06_Preview() {
    MaterialTheme { Paso06_DialogosScreen() }
}
```

**▶ Ejecuta:** `Paso06_DialogosScreen()` ya está activo en `MainActivity`.
- Presiona el FAB → abre el formulario en un `Dialog` personalizado
- Completa los 3 campos → el botón "Guardar" se activa; guarda y aparece el snackbar
- Presiona el ícono de eliminar en un contacto → aparece `AlertDialog` de confirmación
- Toca "Eliminar" → el contacto desaparece y aparece snackbar de confirmación
- Navega entre las tres pestañas con todos los datos actualizados

---

## Referencia rápida — qué descomentar en `MainActivity`

| Paso | Archivo | Descomentar | Qué introduce |
|---|---|---|---|
| 1 | `Paso01_TextField.kt` | `Paso01_TextFieldScreen()` | `OutlinedTextField`, validación, teclado |
| 2 | `Paso02_Card.kt` | `Paso02_CardScreen()` | `ElevatedCard`, `AssistChip` |
| 3 | `Paso03_LazyColumn.kt` | `Paso03_LazyColumnScreen()` | `LazyColumn`, `LazyRow`, `key`, filtros |
| 4 | `Paso04_Scaffold.kt` | `Paso04_ScaffoldScreen()` | `Scaffold`, `TopAppBar`, `FAB`, `paddingValues` |
| 5 | `Paso05_NavBar.kt` | `Paso05_NavBarScreen()` | `NavigationBar`, `NavigationBarItem` |
| 6 | `Paso06_Dialogos.kt` | `Paso06_DialogosScreen()` | `AlertDialog`, `Dialog`, `Snackbar`, `LaunchedEffect` |

---

## Ejercicios propuestos

**Ejercicio 1 — Lista de tareas** (`ui/E01_Tareas.kt`)
Crea una pantalla con `Scaffold` y `LazyColumn` que muestre tareas.
Cada tarea: `texto: String`, `completada: Boolean`, `prioridad: Int` (1–3).
Un `Checkbox` en cada ítem alterna `completada`. El FAB abre un `Dialog`
con `OutlinedTextField` para agregar. Agrega `FilterChip` para filtrar
por `"Todas"`, `"Pendientes"` y `"Completadas"`.

**Ejercicio 2 — Perfil con formulario** (`ui/E02_Perfil.kt`)
En la pestaña "Perfil" del Paso 5, agrega un formulario con `nombre`,
`bio` (multilinea, `maxLines = 4`) y `website`. Valida URL con
`startsWith("https://")`. Botón "Guardar" deshabilitado hasta validación.
Muestra `Snackbar` de éxito al guardar.

**Ejercicio 3 — Carrito de compras** (`ui/E03_Carrito.kt`)
Crea una `NavigationBar` con "Catálogo" y "Carrito". En "Catálogo",
una `LazyColumn` de productos con `FilledTonalButton` para agregar.
En "Carrito", lista de ítems con cantidades y total calculado. Usa
`mutableStateListOf` para el estado del carrito.

**Ejercicio 4 — Swipe to dismiss**
Investiga `SwipeToDismissBox` (Material 3) y agrégalo a la lista del
Paso 6. Al deslizar izquierda, muestra fondo rojo con `Icons.Default.Delete`.
Al soltar, ejecuta el mismo flujo de confirmación del `AlertDialog`.

---

## Resumen de la página 12

- `OutlinedTextField` es completamente controlado: recibe `value` y notifica cambios con `onValueChange`. `keyboardOptions`, `visualTransformation` e `isError` + `supportingText` cubren la mayoría de casos de formularios.
- `ElevatedCard` con `onClick` es la forma estándar de ítem interactivo en Material 3. Las variantes `Card` y `OutlinedCard` sirven para contenido plano o listas densas.
- `LazyColumn` virtualiza la lista — solo renderiza elementos visibles. El parámetro `key` es crítico para el rendimiento al insertar o eliminar ítems.
- `contentPadding` en `LazyColumn` añade espacio al inicio y final sin bloquear el scroll.
- `Scaffold` gestiona los insets del sistema. El `paddingValues` que devuelve **siempre** debe aplicarse al contenido para evitar solapamiento con la `TopAppBar` o `NavigationBar`.
- `TopAppBar` con `actions` acepta `IconButton` para acciones secundarias.
- `NavigationBar` usa íconos rellenos para el destino activo y contorno para los inactivos — convención estándar de Material 3.
- `AlertDialog` para confirmaciones destructivas; `Dialog` para contenido personalizado.
- `SnackbarHostState` + `LaunchedEffect` es el patrón para mostrar snackbars desde eventos de estado. `LaunchedEffect` ejecuta código en una coroutine cuando su clave cambia.

---

> **Siguiente página →** Página 13: Jetpack Compose — estado con `ViewModel`,
> `StateFlow`, navegación con Navigation Compose y consumo de API REST.
