# Curso Kotlin — Página 2
## Módulo 1 · Fundamentos
### Control de flujo: `if`, `when`, bucles y rangos

> Los ejemplos de esta página usan como hilo conductor un **sistema de gestión
> clínica** — triaje de pacientes, asignación de turnos, cálculo de tarifas por
> cobertura médica, seguimiento de citas y alertas de signos vitales. Un dominio
> con reglas de negocio claras y graduadas, ideal para ilustrar cada estructura
> de control de flujo.

---

## `if` — condicional simple

La forma más básica de `if` ejecuta un bloque solo si la condición es `true`.
Si es `false`, el programa continúa sin hacer nada.

```kotlin
fun main() {
    print("Temperatura corporal del paciente (°C): ")
    val temperatura = readLine()?.toDoubleOrNull() ?: 36.5

    if (temperatura >= 38.0) {
        println("⚠ Fiebre detectada: derivar a consulta prioritaria.")
    }

    if (temperatura >= 40.0) {
        println("🚨 Fiebre alta: atención de emergencia inmediata.")
    }

    println("Temperatura registrada: $temperatura °C")
}
```

Ejecución con `temperatura = 40.2`:
```
⚠ Fiebre detectada: derivar a consulta prioritaria.
🚨 Fiebre alta: atención de emergencia inmediata.
Temperatura registrada: 40.2 °C
```

Ejecución con `temperatura = 36.8`:
```
Temperatura registrada: 36.8 °C
```

> El bloque `{ }` es obligatorio cuando el cuerpo ocupa varias líneas.
> Si solo hay una instrucción las llaves son opcionales, pero el curso las
> usa siempre para mayor claridad.

---

### 🏋 Ejercicios — `if` simple

**E1.** El sistema registra la saturación de oxígeno de un paciente (SpO₂ en %).
Lee el valor desde la consola. Si es menor que 95, imprime
`"Alerta: saturación baja — evaluar suministro de oxígeno"`.

**E2.** Una clínica cobra recargo administrativo si la cita se cancela con menos
de 24 horas de anticipación. Lee las horas de anticipación. Si son menores que 24,
imprime `"Recargo por cancelación tardía aplicado: $15.00"`.

---

## `if` / `else` — dos caminos

Cuando la lógica tiene exactamente dos resultados posibles y mutuamente
excluyentes, se añade la rama `else`.

```kotlin
fun main() {
    print("¿El paciente tiene seguro médico activo? (s/n): ")
    val tieneSeguro = readLine()?.trim()?.lowercase() == "s"

    print("Costo base de la consulta: $")
    val costoBase = readLine()?.toDoubleOrNull() ?: 0.0

    if (tieneSeguro) {
        val cobertura = costoBase * 0.80
        println("Seguro cubre: $${"%.2f".format(cobertura)} — " +
                "Copago del paciente: $${"%.2f".format(costoBase - cobertura)}")
    } else {
        println("Pago particular: $${"%.2f".format(costoBase)}")
    }
}
```

### `if` / `else` como expresión

En Kotlin, `if` / `else` puede asignarse directamente a una variable,
eliminando la necesidad del operador ternario `? :` de otros lenguajes.

```kotlin
fun main() {
    print("Edad del paciente: ")
    val edad = readLine()?.toIntOrNull() ?: 0

    // if como expresión — produce un String directamente
    val tipoPaciente = if (edad < 18) "Pediátrico" else "Adulto"
    println("Tipo de paciente: $tipoPaciente")

    // Útil también dentro de string templates
    val descuento = if (edad >= 65) 0.30 else 0.0
    println("Descuento tercera edad: ${(descuento * 100).toInt()}%")
    println("Derivar a: ${if (edad < 18) "Consulta de Pediatría" else "Consulta General"}")
}
```

> Cuando `if` se usa como expresión, la rama `else` es **obligatoria** —
> el compilador necesita garantizar que siempre haya un valor.

---

### 🏋 Ejercicios — `if` / `else`

**E3.** Un paciente debe estar en ayunas si su examen es de tipo "laboratorio".
Lee el tipo de examen (`laboratorio` o `imagen`). Usando `if` como expresión,
asigna a una variable el mensaje de preparación correspondiente e imprímelo.

**E4.** La clínica trabaja en dos turnos: mañana (8:00–13:00) y tarde (13:00–18:00).
Lee la hora de la cita (número entero entre 8 y 17). Usando `if` como expresión,
determina el turno asignado y calcula cuántas horas faltan para el cierre de ese turno.

---

## `if` / `else if` / `else` — múltiples condiciones

Cuando hay más de dos categorías posibles, se encadenan condiciones con
`else if`. Las ramas se evalúan en orden — solo se ejecuta la primera verdadera.

```kotlin
fun main() {
    print("Presión arterial sistólica (mmHg): ")
    val sistolica = readLine()?.toIntOrNull() ?: 0

    val clasificacion = if (sistolica < 90) {
        "Hipotensión — monitorear y valorar causa"
    } else if (sistolica <= 119) {
        "Normal — continuar controles periódicos"
    } else if (sistolica <= 129) {
        "Elevada — recomendar cambios en estilo de vida"
    } else if (sistolica <= 139) {
        "Hipertensión grado 1 — iniciar seguimiento mensual"
    } else if (sistolica <= 179) {
        "Hipertensión grado 2 — iniciar tratamiento farmacológico"
    } else {
        "Crisis hipertensiva — atención de urgencia inmediata"
    }

    println("Clasificación: $clasificacion")
}
```

Ejecución con `sistolica = 145`:
```
Clasificación: Hipertensión grado 2 — iniciar tratamiento farmacológico
```

---

### 🏋 Ejercicios — `if` / `else if` / `else`

**E5.** El índice de masa corporal (IMC) clasifica el estado nutricional de un
paciente. Lee peso (kg) y altura (m), calcula el IMC (`peso / altura²`) y
clasifícalo: menor de 16 → `"Desnutrición severa"`, 16–18.4 → `"Bajo peso"`,
18.5–24.9 → `"Normal"`, 25–29.9 → `"Sobrepeso"`, 30–34.9 → `"Obesidad grado I"`,
35–39.9 → `"Obesidad grado II"`, 40 o más → `"Obesidad mórbida"`.

**E6.** Una clínica asigna tipo de consulta según el tiempo de espera en sala:
0–10 min → `"Flujo normal"`, 11–30 min → `"Espera moderada — informar al paciente"`,
31–60 min → `"Espera prolongada — ofrecer reagendamiento"`,
más de 60 min → `"Alerta de gestión — notificar a coordinación"`.
Lee el tiempo de espera e imprime el mensaje correspondiente.

---

## `if` anidado

Un `if` dentro de otro permite evaluar condiciones que solo tienen sentido
cuando una condición exterior ya se cumplió.

```kotlin
fun main() {
    print("¿El paciente tiene antecedentes cardíacos? (s/n): ")
    val antecedenteCardiaco = readLine()?.trim()?.lowercase() == "s"

    print("Frecuencia cardíaca (lpm): ")
    val frecuencia = readLine()?.toIntOrNull() ?: 0

    if (antecedenteCardiaco) {
        println("Paciente con antecedentes cardíacos — protocolo de monitoreo activo.")
        if (frecuencia < 50) {
            println("🚨 Bradicardia severa: derivar a cardiología de urgencia.")
        } else if (frecuencia > 100) {
            println("🚨 Taquicardia: solicitar ECG y valorar medicación.")
        } else {
            println("✅ Frecuencia dentro del rango esperado para su condición.")
        }
    } else {
        println("Paciente sin antecedentes cardíacos registrados.")
        if (frecuencia < 60 || frecuencia > 100) {
            println("⚠ Frecuencia fuera del rango normal — registrar en historia clínica.")
        } else {
            println("✅ Frecuencia cardíaca normal.")
        }
    }
}
```

Ejecución con `antecedenteCardiaco = s`, `frecuencia = 112`:
```
Paciente con antecedentes cardíacos — protocolo de monitoreo activo.
🚨 Taquicardia: solicitar ECG y valorar medicación.
```

> El `if` anidado es útil para condiciones dependientes, pero más de dos
> niveles suele indicar que `when` es una alternativa más legible.

---

### 🏋 Ejercicios — `if` anidado

**E7.** El sistema registra si un paciente es diabético y su nivel de glucosa
en sangre (mg/dL). Si es diabético: menor de 70 → `"Hipoglucemia — administrar glucosa"`,
70–180 → `"Dentro del rango objetivo"`, mayor de 180 → `"Hiperglucemia — ajustar dosis"`.
Si no es diabético: menor de 70 → `"Hipoglucemia"`, 70–99 → `"Normal"`,
100–125 → `"Prediabetes — derivar a endocrinología"`, mayor de 125 → `"Posible diabetes — solicitar confirmación"`.

**E8.** Una consulta puede estar en estado `"agendada"`, `"en_curso"` o `"finalizada"`.
Si está en curso, comprueba además si lleva más de 30 minutos (el usuario ingresa
los minutos transcurridos). Si supera ese tiempo, imprime
`"Consulta extendida — notificar a la sala de espera sobre el retraso"`.

---

## `when` — clasificación sin cascadas de `else if`

`when` evalúa un valor contra múltiples ramas y ejecuta solo la primera que
coincide. Es más legible que encadenar `else if` cuando hay muchos casos.

### Forma básica — valor exacto

```kotlin
fun main() {
    print("Código de especialidad médica (1-7): ")
    val codigo = readLine()?.toIntOrNull() ?: 0

    val especialidad = when (codigo) {
        1    -> "Medicina General"
        2    -> "Pediatría"
        3    -> "Cardiología"
        4    -> "Ginecología"
        5    -> "Traumatología"
        6    -> "Neurología"
        7    -> "Dermatología"
        else -> "Especialidad no registrada en el sistema"
    }

    println("Especialidad: $especialidad")
}
```

---

### 🏋 Ejercicios — `when` básico

**E9.** El sistema registra el tipo de muestra de laboratorio con un código:
`1` → Sangre venosa, `2` → Orina, `3` → Heces, `4` → Hisopado nasofaríngeo,
`5` → Biopsia. Usa `when` para imprimir el tipo de muestra y el tiempo
estimado de procesamiento: sangre (4 h), orina (2 h), heces (24 h),
hisopado (6 h), biopsia (72 h).

**E10.** El resultado de un examen de imágenes tiene cuatro conclusiones posibles
codificadas con letra: `N` → Normal, `L` → Hallazgo leve, `M` → Hallazgo moderado,
`G` → Hallazgo grave. Usa `when` para imprimir la conclusión y la acción clínica
recomendada para cada caso.

---

### `when` con condiciones arbitrarias y rangos

Cuando `when` no recibe argumento, cada rama es una condición booleana
independiente. Las ramas se evalúan de arriba a abajo.

```kotlin
fun calcularCopago(edadPaciente: Int, tieneSeguro: Boolean, nivelSeguro: String): Double {
    return when {
        !tieneSeguro && edadPaciente < 18  -> 0.0    // menores sin seguro: exentos
        !tieneSeguro && edadPaciente >= 65 -> 15.0   // adultos mayores sin seguro: tarifa reducida
        !tieneSeguro                       -> 45.0   // particular adulto
        nivelSeguro == "BASICO"            -> 20.0
        nivelSeguro == "INTERMEDIO"        -> 10.0
        nivelSeguro == "PREMIUM"           -> 0.0
        else                               -> 30.0   // seguro no reconocido
    }
}

fun main() {
    print("Edad del paciente: ")
    val edad = readLine()?.toIntOrNull() ?: 0

    print("¿Tiene seguro médico? (s/n): ")
    val tieneSeguro = readLine()?.trim()?.lowercase() == "s"

    val nivel = if (tieneSeguro) {
        print("Nivel del seguro (BASICO / INTERMEDIO / PREMIUM): ")
        readLine()?.trim()?.uppercase() ?: ""
    } else ""

    val copago = calcularCopago(edad, tieneSeguro, nivel)
    println("Copago aplicable: $${"%.2f".format(copago)}")
}
```

---

### 🏋 Ejercicios — `when` con condiciones

**E11.** El sistema de triaje clasifica la prioridad de atención según los síntomas
reportados. Usa `when` sin argumento con las siguientes reglas (en orden de prioridad):
dolor de pecho + dificultad respiratoria → `"P1 — Emergencia"`,
dolor de pecho solo → `"P2 — Urgente"`,
temperatura ≥ 39.5 → `"P2 — Urgente"`,
temperatura 38–39.4 → `"P3 — Prioritario"`,
resto → `"P4 — Consulta general"`.
Lee los síntomas desde la consola con preguntas de sí/no.

**E12.** Una farmacia de la clínica cobra el medicamento según el tipo de paciente
y la cantidad de unidades. Usa `when` para determinar el precio unitario:
paciente pediátrico (< 18 años): jarabe (≤ 100 ml) $4.50, tabletas $1.20;
adulto con seguro PREMIUM: jarabe $3.00, tabletas $0.80;
adulto particular o con seguro básico: jarabe $6.00, tabletas $1.80.
Lee todos los datos desde la consola e imprime el total a pagar.

---

### `when` con comprobación de tipo (`is`)

```kotlin
fun procesarRegistro(dato: Any): String {
    return when (dato) {
        is Int    -> "Valor numérico entero registrado: $dato"
        is Double -> "Medición clínica: ${"%.2f".format(dato)}"
        is String -> if (dato.isBlank())
                         "⚠ Campo de texto vacío — no registrado"
                     else
                         "Texto registrado: \"$dato\" (${dato.length} caracteres)"
        is Boolean -> if (dato) "✅ Condición confirmada" else "❌ Condición negada"
        is List<*> -> "Lista de ${dato.size} registros adjuntos"
        else       -> "Tipo de dato no soportado por el sistema"
    }
}

fun main() {
    val registros: List<Any> = listOf(
        120,                          // presión sistólica
        98.6,                         // temperatura en °F
        "Paciente refiere dolor torácico irradiado a brazo izquierdo",
        true,                         // consentimiento informado firmado
        listOf("HCL-001", "HCL-002", "HCL-003"),  // historias clínicas adjuntas
        ""                            // campo vacío
    )

    for (r in registros) println(procesarRegistro(r))
}
```

Salida:
```
Valor numérico entero registrado: 120
Medición clínica: 98.60
Texto registrado: "Paciente refiere dolor torácico irradiado a brazo izquierdo" (57 caracteres)
✅ Condición confirmada
Lista de 3 registros adjuntos
⚠ Campo de texto vacío — no registrado
```

> Dentro de cada rama `is Tipo`, Kotlin aplica **smart cast** automáticamente —
> la variable ya es tratada como ese tipo sin necesidad de ningún cast explícito.

---

### `when` con bloque de código

Cuando una rama requiere más de una instrucción, se usa un bloque `{ }`.

```kotlin
fun gestionarAlerta(nivel: String, paciente: String) {
    when (nivel.trim().uppercase()) {
        "CRITICO" -> {
            println("🚨 ALERTA CRÍTICA — Paciente: $paciente")
            println("   Acción inmediata: llamar a médico de guardia.")
            println("   Registrar hora de activación de protocolo.")
        }
        "URGENTE" -> {
            println("⚠ URGENTE — Paciente: $paciente")
            println("   Priorizar en sala de espera.")
            println("   Reevaluar en 15 minutos.")
        }
        "MODERADO" -> println("📋 Moderado — $paciente: registrar y monitorear.")
        "LEVE"     -> println("ℹ Leve — $paciente: continuar en lista de espera normal.")
        else       -> println("Nivel de alerta '$nivel' no reconocido en el protocolo.")
    }
}

fun main() {
    print("Nombre del paciente: ")
    val paciente = readLine() ?: "Sin identificar"

    print("Nivel de alerta (CRITICO / URGENTE / MODERADO / LEVE): ")
    val nivel = readLine() ?: ""

    gestionarAlerta(nivel, paciente)
}
```

---

## Rangos

Los rangos representan secuencias de valores y se combinan naturalmente con
`for`, `when` e `in`.

```kotlin
fun main() {
    // Franjas horarias de los turnos de consulta
    val turnoMañana  = 8..12
    val turnoTarde   = 13..17
    val turnoNoche   = 18..22

    print("Hora de la cita (8-22): ")
    val hora = readLine()?.toIntOrNull() ?: 0

    val turno = when (hora) {
        in turnoMañana -> "Turno mañana (8:00–12:59)"
        in turnoTarde  -> "Turno tarde (13:00–17:59)"
        in turnoNoche  -> "Turno noche (18:00–22:59)"
        else           -> "Fuera del horario de atención"
    }
    println("Turno asignado: $turno")

    // Rangos de edad para tarifas pediátricas
    val edades = listOf(2, 7, 14, 18, 35, 67)
    for (edad in edades) {
        val tarifa = when (edad) {
            in 0..2   -> "Neonatal — tarifa diferenciada $25.00"
            in 3..11  -> "Pediátrico — tarifa $30.00"
            in 12..17 -> "Adolescente — tarifa $35.00"
            in 18..64 -> "Adulto — tarifa $45.00"
            else      -> "Adulto mayor — tarifa $30.00 (descuento 33%)"
        }
        println("$edad años → $tarifa")
    }
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

### 🏋 Ejercicios — rangos

**E13.** Una clínica tiene consultorios numerados del 1 al 20. Los consultorios
1–5 son de emergencia, 6–12 de especialidades y 13–20 de medicina general.
Lee el número de consultorio e imprime el área correspondiente y si requiere
código de acceso (solo emergencia lo requiere).

**E14.** El sistema asigna número de turno de sala de espera del 1 al 60.
Turnos 1–20 → Ventanilla A, 21–40 → Ventanilla B, 41–60 → Ventanilla C.
Lee el número de turno usando `in` con rangos e imprime la ventanilla asignada
y cuántos turnos faltan para llegar al límite de esa ventanilla.

---

## Bucle `for`

```kotlin
fun main() {
    // Revisar signos vitales de los pacientes en sala de observación
    val pacientes = listOf(
        Triple("García, M.", 37.2, 98),    // nombre, temperatura, SpO2
        Triple("López, R.",  39.1, 94),
        Triple("Torres, A.", 36.8, 99),
        Triple("Mora, K.",   40.3, 91)
    )

    println("=== Revisión de sala de observación ===")
    for ((posicion, paciente) in pacientes.withIndex()) {
        val (nombre, temperatura, spo2) = paciente
        val alertaTemp = if (temperatura >= 38.0) "⚠ FIEBRE" else "✅ Normal"
        val alertaSpo2 = if (spo2 < 95) "⚠ SpO₂ BAJA" else "✅ Normal"
        println("Cama ${posicion + 1} — $nombre | Temp: $temperatura°C $alertaTemp | SpO₂: $spo2% $alertaSpo2")
    }

    // Generar agenda de control de turno (cada 2 horas de 8 a 18)
    println("\n=== Controles programados — turno mañana ===")
    for (hora in 8..18 step 2) {
        println("Control a las ${hora}:00 h")
    }
}
```

---

### 🏋 Ejercicios — `for`

**E15.** Una clínica tiene `n` pacientes en lista de espera (lee `n` desde la
consola). Para cada uno, lee el nombre y el tiempo de espera en minutos.
Usa `for` con `withIndex` para imprimir la lista numerada y al final indica
cuántos pacientes llevan más de 30 minutos esperando.

**E16.** Genera el horario semanal de un médico: trabaja de lunes a viernes,
con citas cada 20 minutos desde las 8:00 hasta las 12:00 (solo mañana).
Usa `for` con `step` para imprimir cada hora y minuto de las citas disponibles
en formato `"08:00 — disponible"`.

---

## Bucles `while` y `do-while`

```kotlin
fun main() {
    // while — monitorear la saturación de un paciente hasta que se estabilice
    var spo2 = 88
    var minutos = 0

    println("=== Monitoreo de SpO₂ post-procedimiento ===")
    while (spo2 < 95) {
        minutos++
        spo2 += 2   // simulación: la saturación sube 2% por minuto con oxígeno
        println("Minuto $minutos — SpO₂: $spo2%")
    }
    println("✅ Paciente estabilizado en $minutos minutos.")

    // do-while — registrar peso del paciente con validación
    var peso: Double?
    do {
        print("\nPeso del paciente en kg (entre 1.0 y 300.0): ")
        peso = readLine()?.toDoubleOrNull()
        when {
            peso == null    -> println("  ⚠ Escribe un número válido.")
            peso < 1.0      -> println("  ⚠ El peso ingresado es demasiado bajo.")
            peso > 300.0    -> println("  ⚠ El peso ingresado supera el límite del sistema.")
        }
    } while (peso == null || peso < 1.0 || peso > 300.0)

    println("Peso registrado: $peso kg")
}
```

---

### 🏋 Ejercicios — `while` / `do-while`

**E17.** Un paciente debe tomar una pastilla cada 8 horas durante 7 días.
Usa `while` para imprimir el horario completo de tomas (día y hora), comenzando
desde las 8:00 del día 1. Cuenta cuántas tomas se realizan en total.

**E18.** Con `do-while`, implementa el ingreso de un paciente: pide el número
de cédula (exactamente 10 dígitos numéricos), el nombre completo (mínimo
5 caracteres y que contenga un espacio), y la fecha de nacimiento en formato
`DD/MM/AAAA` (valida que tenga exactamente 10 caracteres y dos barras `/`).
Valida cada campo antes de pasar al siguiente.

---

## `break`, `continue` y etiquetas

```kotlin
fun main() {
    val resultados = listOf(
        Pair("LAB-001",  85.0),    // glucosa en mg/dL — normal
        Pair("LAB-002", -5.0),     // valor inválido
        Pair("LAB-003", 310.0),    // valor crítico
        Pair("LAB-004",  72.0),
        Pair("LAB-005", 102.0)
    )

    println("=== Procesamiento de resultados de laboratorio ===")
    for ((codigo, valor) in resultados) {
        if (valor < 0) {
            println("$codigo → ERROR: valor negativo ($valor). Detener procesamiento del lote.")
            break   // dato incoherente: detiene toda la revisión
        }
        if (valor > 250.0) {
            println("$codigo → OMITIDO: valor crítico ($valor mg/dL) — requiere revisión manual.")
            continue   // salta este resultado, continúa con los demás
        }
        val estado = if (valor in 70.0..99.0) "Normal" else "Fuera de rango"
        println("$codigo → $valor mg/dL — $estado")
    }

    // Etiquetas — buscar el primer médico disponible por especialidad y turno
    println("\n=== Búsqueda de médico disponible ===")
    val especialidades = listOf("Cardiología", "Neurología", "Traumatología")
    val turnos = listOf("Mañana", "Tarde", "Noche")
    var asignado = false

    asignacion@ for (especialidad in especialidades) {
        for (turno in turnos) {
            val disponible = (especialidad == "Neurología" && turno == "Tarde")
            if (disponible) {
                println("Médico encontrado: $especialidad — Turno $turno")
                asignado = true
                break@asignacion
            }
        }
    }
    if (!asignado) println("No hay médicos disponibles con los criterios indicados.")
}
```

---

### 🏋 Ejercicios — `break` / `continue`

**E19.** Lee resultados de glucosa de pacientes en un bucle. Si el valor es
negativo, ignóralo con `continue`. Si algún valor supera 400 mg/dL, imprime
`"Valor crítico detectado — detener revisión automatizada"` y sal con `break`.
Al final muestra cuántos resultados válidos se procesaron y el promedio.

**E20.** Una clínica tiene 4 pisos con 6 habitaciones cada uno. Usa dos `for`
anidados con etiqueta para encontrar la primera habitación disponible cuyo
número combinado (`piso * 6 + habitacion`) sea divisible entre 5 y entre 3
al mismo tiempo. Imprime el piso y habitación encontrados.

---

## `repeat`

Alternativa concisa a `for` cuando solo necesitas ejecutar un bloque
un número fijo de veces.

```kotlin
fun main() {
    print("¿Cuántas pulsaciones tomar para calcular frecuencia cardíaca? ")
    val mediciones = readLine()?.toIntOrNull() ?: 3

    var totalPulsaciones = 0
    repeat(mediciones) { i ->
        print("  Medición ${i + 1} (pulsos en 15 seg): ")
        val pulsos = readLine()?.toIntOrNull() ?: 0
        totalPulsaciones += pulsos * 4   // extrapolar a 60 segundos
    }

    val promedio = totalPulsaciones / mediciones
    println("\nFrecuencia cardíaca promedio: $promedio lpm")
    println("Clasificación: ${
        when {
            promedio < 60  -> "Bradicardia"
            promedio <= 100 -> "Normal"
            else            -> "Taquicardia"
        }
    }")
}
```

---

### 🏋 Ejercicios — `repeat`

**E21.** Un médico debe registrar la temperatura de un paciente cada 4 horas
durante 24 horas (6 mediciones). Usa `repeat` para pedir cada valor, detecta
si alguna supera 38.5°C e imprime al final el promedio y si hubo fiebre sostenida
(más de 2 mediciones con fiebre).

**E22.** Una clínica realiza 5 pruebas de presión arterial seguidas con 1 minuto
de pausa entre ellas (`Thread.sleep(1000)` para simular). Usa `repeat` para
pedir sistólica y diastólica en cada toma, y al final imprime el promedio de
ambas y si el paciente está en rango normal (sistólica ≤ 120 y diastólica ≤ 80).

---

## Programas finales — combinando controles de flujo

### Programa 1 — Sistema de triaje automatizado

Evalúa a varios pacientes en cola, clasifica su prioridad con `when` y
signos vitales, valida cada dato con `do-while` y genera el reporte del turno.

```kotlin
fun nivelTriaje(temperatura: Double, spo2: Int, frecCardiaca: Int): String {
    return when {
        spo2 < 90 || frecCardiaca > 150 || temperatura >= 40.5 -> "P1 — EMERGENCIA"
        spo2 < 95 || frecCardiaca > 120 || temperatura >= 39.0 -> "P2 — URGENTE"
        temperatura >= 38.0 || frecCardiaca !in 60..100        -> "P3 — PRIORITARIO"
        else                                                    -> "P4 — GENERAL"
    }
}

fun main() {
    print("Número de pacientes en triaje: ")
    val totalPacientes = readLine()?.toIntOrNull() ?: 0

    val contadorNivel = mutableMapOf("P1 — EMERGENCIA" to 0, "P2 — URGENTE" to 0,
                                     "P3 — PRIORITARIO" to 0, "P4 — GENERAL" to 0)

    for (n in 1..totalPacientes) {
        println("\n--- Paciente $n de $totalPacientes ---")

        print("  Nombre: ")
        val nombre = readLine()?.ifBlank { "Sin identificar" } ?: "Sin identificar"

        var temperatura: Double?
        do {
            print("  Temperatura (°C): ")
            temperatura = readLine()?.toDoubleOrNull()
            if (temperatura == null || temperatura !in 30.0..45.0)
                println("  ⚠ Temperatura fuera de rango válido (30–45 °C).")
        } while (temperatura == null || temperatura !in 30.0..45.0)

        var spo2: Int?
        do {
            print("  SpO₂ (%): ")
            spo2 = readLine()?.toIntOrNull()
            if (spo2 == null || spo2 !in 50..100)
                println("  ⚠ SpO₂ debe estar entre 50 y 100.")
        } while (spo2 == null || spo2 !in 50..100)

        var frecCardiaca: Int?
        do {
            print("  Frecuencia cardíaca (lpm): ")
            frecCardiaca = readLine()?.toIntOrNull()
            if (frecCardiaca == null || frecCardiaca !in 20..250)
                println("  ⚠ Frecuencia debe estar entre 20 y 250 lpm.")
        } while (frecCardiaca == null || frecCardiaca !in 20..250)

        val nivel = nivelTriaje(temperatura, spo2, frecCardiaca)
        contadorNivel[nivel] = (contadorNivel[nivel] ?: 0) + 1
        println("  → $nombre clasificado como: $nivel")
    }

    println("\n=== Resumen del turno de triaje ===")
    for ((nivel, cantidad) in contadorNivel) {
        if (cantidad > 0) println("  $nivel: $cantidad paciente(s)")
    }
}
```

---

### Programa 2 — Gestión de agenda de consultas

El recepcionista registra y consulta citas en una sesión de consola.
Combina `while`, `when`, `continue` y `break`.

```kotlin
fun main() {
    // Agenda del día: hora (8-17) → nombre del paciente o null si está libre
    val agenda = mutableMapOf<Int, String?>()
    for (hora in 8..17) agenda[hora] = null   // inicializar todas las horas como libres

    println("=== Agenda de Consultas — Recepción ===")
    println("Escribe una opción o 'salir' para cerrar.\n")

    while (true) {
        println("1. Agendar cita  |  2. Ver agenda  |  3. Cancelar cita  |  0. Salir")
        print("Opción: ")

        val opcion = readLine()?.trim() ?: continue

        when (opcion) {
            "0" -> {
                println("Cerrando recepción. Hasta mañana.")
                break
            }
            "1" -> {
                print("Hora de la cita (8-17): ")
                val hora = readLine()?.toIntOrNull()
                if (hora == null || hora !in 8..17) {
                    println("  ⚠ Hora inválida.\n")
                    continue
                }
                if (agenda[hora] != null) {
                    println("  ⚠ La hora ${hora}:00 ya está ocupada por ${agenda[hora]}.\n")
                    continue
                }
                print("Nombre del paciente: ")
                val nombre = readLine()?.ifBlank { null }
                if (nombre == null) {
                    println("  ⚠ El nombre no puede estar vacío.\n")
                    continue
                }
                agenda[hora] = nombre
                println("  ✅ Cita agendada: ${hora}:00 — $nombre\n")
            }
            "2" -> {
                println("\n  Hora  | Paciente")
                println("  ------|-----------------------------")
                for (hora in 8..17) {
                    val estado = agenda[hora] ?: "— disponible"
                    println("  ${hora}:00  | $estado")
                }
                println()
            }
            "3" -> {
                print("Hora a cancelar (8-17): ")
                val hora = readLine()?.toIntOrNull()
                if (hora == null || hora !in 8..17) {
                    println("  ⚠ Hora inválida.\n")
                    continue
                }
                if (agenda[hora] == null) {
                    println("  ⚠ No hay cita registrada a las ${hora}:00.\n")
                    continue
                }
                println("  Cita de ${agenda[hora]} a las ${hora}:00 cancelada.")
                agenda[hora] = null
                println()
            }
            else -> println("  ⚠ Opción no válida.\n")
        }
    }
}
```

---

### Programa 3 — Monitor de signos vitales por turno

Lee los signos vitales de varios pacientes en sala de observación, detecta
alertas con `when` y genera un informe al cierre del turno. Combina `repeat`,
`do-while`, `for` y rangos.

```kotlin
data class Paciente(
    val nombre: String,
    val temperatura: Double,
    val spo2: Int,
    val presionSistolica: Int
)

fun evaluarPaciente(p: Paciente): List<String> {
    val alertas = mutableListOf<String>()
    if (p.temperatura >= 38.0) alertas.add("Fiebre (${p.temperatura}°C)")
    if (p.spo2 < 95) alertas.add("SpO₂ baja (${p.spo2}%)")
    when {
        p.presionSistolica >= 180 -> alertas.add("Crisis hipertensiva (${p.presionSistolica} mmHg)")
        p.presionSistolica >= 140 -> alertas.add("HTA grado 2 (${p.presionSistolica} mmHg)")
        p.presionSistolica < 90   -> alertas.add("Hipotensión (${p.presionSistolica} mmHg)")
    }
    return alertas
}

fun main() {
    print("Número de pacientes en sala de observación: ")
    val numPacientes = readLine()?.toIntOrNull() ?: 0

    val pacientes = mutableListOf<Paciente>()

    repeat(numPacientes) { i ->
        println("\n--- Registro paciente ${i + 1} ---")

        print("  Nombre: ")
        val nombre = readLine()?.ifBlank { "Paciente ${i + 1}" } ?: "Paciente ${i + 1}"

        var temp: Double?
        do {
            print("  Temperatura (°C): ")
            temp = readLine()?.toDoubleOrNull()
            if (temp == null || temp !in 30.0..45.0)
                println("  ⚠ Valor fuera de rango.")
        } while (temp == null || temp !in 30.0..45.0)

        var spo2: Int?
        do {
            print("  SpO₂ (%): ")
            spo2 = readLine()?.toIntOrNull()
            if (spo2 == null || spo2 !in 50..100)
                println("  ⚠ Valor fuera de rango.")
        } while (spo2 == null || spo2 !in 50..100)

        var presion: Int?
        do {
            print("  Presión sistólica (mmHg): ")
            presion = readLine()?.toIntOrNull()
            if (presion == null || presion !in 50..250)
                println("  ⚠ Valor fuera de rango.")
        } while (presion == null || presion !in 50..250)

        pacientes.add(Paciente(nombre, temp, spo2, presion))
    }

    println("\n=== Informe de sala de observación ===")
    var conAlertas = 0

    for ((indice, paciente) in pacientes.withIndex()) {
        val alertas = evaluarPaciente(paciente)
        print("Cama ${indice + 1} — ${paciente.nombre}: ")
        if (alertas.isEmpty()) {
            println("✅ Sin alertas")
        } else {
            conAlertas++
            println("⚠ ${alertas.joinToString(", ")}")
        }
    }

    println("\nPacientes sin alertas : ${pacientes.size - conAlertas}")
    println("Pacientes con alertas : $conAlertas")
}
```

---

## Ejercicios finales — combinando controles de flujo

**EF1. Módulo de facturación clínica** — Implementa un menú de consola con
las siguientes opciones:

- `1` Registrar servicio — elige entre consulta, examen de laboratorio o imagen;
  pide el tipo de paciente y nivel de seguro; calcula el copago con `when`
- `2` Ver resumen de facturación — total de servicios, monto bruto y monto neto
  después de coberturas
- `3` Aplicar descuento especial — solicita código de descuento y porcentaje
- `0` Cerrar turno e imprimir factura final

El menú se repite con `while` hasta elegir `0`. Cada entrada se valida
con `do-while` + las funciones de conversión segura (`toIntOrNull`, etc.).

**EF2. Planificador de turnos médicos** — Una clínica tiene 3 médicos generales
y cada uno puede atender hasta 8 pacientes por turno (8:00–16:00, citas de 1 hora).
Lee los pacientes con su nombre y prioridad (`P1`, `P2`, `P3`, `P4`).
Asigna primero los de mayor prioridad al médico con menos pacientes en ese momento.
Si los tres médicos están llenos, imprime `"Lista de espera para próximo turno"`.
Al final imprime la agenda de cada médico con sus pacientes y el nivel de
ocupación en porcentaje.

---

## Resumen de la página 2

- `if` simple ejecuta un bloque solo si la condición es `true` — si es `false`, el programa continúa sin hacer nada.
- `if` / `else` cubre dos caminos mutuamente excluyentes. Como expresión, la rama `else` es obligatoria.
- `if` / `else if` / `else` evalúa condiciones en orden — solo se ejecuta la primera que sea `true`.
- El `if` anidado es útil para condiciones dependientes; más de dos niveles suele indicar que `when` es más legible.
- `when` reemplaza y supera al `switch`: acepta rangos, condiciones arbitrarias y comprobación de tipo con `is`.
- Los rangos `1..10`, `1 until 10`, `10 downTo 1` y `step` permiten expresar secuencias de forma concisa.
- El operador `in` comprueba pertenencia a un rango o colección — `!in` es su negación.
- `for (i in rango)` y `for (elem in coleccion)` son las formas idiomáticas de iterar; `withIndex()` da acceso al índice.
- `while` comprueba la condición antes de ejecutar; `do-while` garantiza al menos una ejecución — ideal para validar entradas interactivas.
- `break` y `continue` con etiquetas `@` controlan bucles anidados de forma explícita.
- `repeat(n) { }` es la alternativa concisa a `for` cuando no se necesita el índice.

---

> **Siguiente página →** Página 3: Funciones — declaración, parámetros por
> defecto, named arguments, funciones de extensión y funciones de orden superior.