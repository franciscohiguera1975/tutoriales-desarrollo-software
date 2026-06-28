# Tutorial TypeScript — Página 3
## Control de Flujo
### Condicionales `if` y bucles `for`

---

> **Cómo usar esta página.** Cada bloque sigue el mismo ritmo:
> **1)** el concepto puro, **2)** un ejemplo aplicado real, **3)** un
> **mini-ejercicio** que se resuelve en clase en **5–8 minutos**.
> Escribe el código en el [TypeScript Playground](https://www.typescriptlang.org/play)
> o ejecútalo con `npx tsx archivo.ts`.

---

## Parte A · Condicionales `if`

Las condicionales se clasifican por **cuántos caminos** abren.
Vamos de menos a más: un camino, dos caminos, múltiples caminos y caminos enlazados.

---

### A.1 · `if` simple — un solo camino

Ejecuta un bloque **solo si** la condición es verdadera. Si es falsa, no pasa nada.

```ts
// Concepto puro
const temperatura: number = 38;

if (temperatura > 37.5) {
  console.log("Tienes fiebre");
}
// Si la temperatura fuera 36, no se imprime nada: no hay "else".
```

#### Ejemplo aplicado — alerta de stock bajo

```ts
const producto: string = "Teclado mecánico";
const unidades: number = 3;

if (unidades < 5) {
  console.log(`⚠️  Reponer "${producto}": quedan ${unidades} unidades`);
}
// ⚠️  Reponer "Teclado mecánico": quedan 3 unidades

const producto2: string = "Mouse";
const unidades2: number = 40;

if (unidades2 < 5) {
  console.log(`⚠️  Reponer "${producto2}": quedan ${unidades2} unidades`);
}
// (no imprime nada)
```

> **Mini-ejercicio (5 min).** Declara `const edad: number = 17`.
> Usa `if` simple para imprimir `"Acceso permitido"` **solo si** la edad es 18 o más.
> Pruébalo cambiando el valor a `18` y luego a `25`. ¿Cuándo aparece el mensaje?

---

### A.2 · `if` / `else` — dos caminos

Una condición, dos salidas posibles: una para verdadero, otra para falso.

```ts
// Concepto puro
const edad: number = 16;

if (edad >= 18) {
  console.log("Mayor de edad");
} else {
  console.log("Menor de edad");
}
```

El **operador ternario** comprime un `if/else` de dos caminos en una sola línea
cuando solo quieres **asignar un valor**:

```ts
// condición ? valorSiVerdadero : valorSiFalso
const estado: string = edad >= 18 ? "Mayor" : "Menor";
console.log(estado);  // Menor
```

#### Ejemplo aplicado — validación de contraseña

```ts
const password: string = "abc";
let mensajePassword: string;

if (password.length >= 8) {
  mensajePassword = "✅ Contraseña válida";
} else {
  mensajePassword = "❌ Mínimo 8 caracteres";
}
console.log(mensajePassword);  // ❌ Mínimo 8 caracteres

const password2: string = "superclave";
const mensajePassword2: string = password2.length >= 8 ? "✅ válida" : "❌ corta";
console.log(mensajePassword2);  // ✅ válida
```

> **Mini-ejercicio (6 min).** Declara `const n: number = 7`.
> Usa `if/else` para imprimir `"par"` o `"impar"` (pista: `n % 2 === 0`).
> Luego reescribe la misma decisión en **una sola línea** con el operador ternario
> y guarda el resultado en `const paridad: string`. Prueba con `4`, `7` y `0`.

---

### A.3 · `if` / `else if` / `else` — múltiples caminos

Cuando hay **más de dos** resultados posibles, encadenas condiciones.
Se evalúan **de arriba hacia abajo** y se ejecuta la **primera** que sea verdadera.

```ts
// Concepto puro
const nota: number = 75;

if (nota >= 90) {
  console.log("A");
} else if (nota >= 80) {
  console.log("B");
} else if (nota >= 70) {
  console.log("C");
} else {
  console.log("Reprobado");
}
// Imprime "C": es la primera condición verdadera.
```

Para comparar **un mismo valor** contra muchas opciones, `switch` es más legible:

```ts
const codigo: number = 404;

switch (codigo) {
  case 200:
    console.log("OK");
    break;
  case 404:
    console.log("No encontrado");
    break;
  case 500:
    console.log("Error del servidor");
    break;
  default:
    console.log("Código desconocido");
}
// ⚠️ Sin "break", la ejecución "cae" al siguiente case.
```

#### Ejemplo aplicado — clasificador de señal WiFi

```ts
// Los routers reportan la señal en dBm; cuanto más cercano a 0, mejor.
const lecturas: number[] = [-45, -62, -71, -83, -95];

for (const dbm of lecturas) {
  let clasificacion: string;

  if (dbm >= -50) {
    clasificacion = `Excelente (${dbm} dBm)`;
  } else if (dbm >= -60) {
    clasificacion = `Buena (${dbm} dBm)`;
  } else if (dbm >= -70) {
    clasificacion = `Aceptable (${dbm} dBm)`;
  } else if (dbm >= -80) {
    clasificacion = `Débil (${dbm} dBm)`;
  } else {
    clasificacion = `Sin cobertura (${dbm} dBm)`;
  }

  console.log(clasificacion);
}
```

Salida:
```
Excelente (-45 dBm)
Buena (-62 dBm)
Aceptable (-71 dBm)
Débil (-83 dBm)
Sin cobertura (-95 dBm)
```

> **Mini-ejercicio (8 min).** Declara `const bateria: number = 25`.
> Usa `if/else if` para imprimir: `"🔴 Crítica"` si < 10, `"🟠 Baja"` si < 30,
> `"🟡 Media"` si < 60, `"🟢 Buena"` si < 90, y `"⚡ Completa"` si es 90 o más.
> Prueba con varios valores. **Extra:** declara `const diaSemana: number = 3`
> y usa `switch` para imprimir el nombre del día (`1→"Lunes"`, …, `7→"Domingo"`,
> otro → `"Día inválido"`).

---

### A.4 · `if` enlazadas / anidadas — condiciones dentro de condiciones

Un `if` **dentro** de otro `if`. Útil cuando una segunda decisión solo tiene
sentido si la primera ya se cumplió.

```ts
// Concepto puro
const logueado: boolean = true;
const esAdmin: boolean = false;

if (logueado) {
  // Esta decisión solo importa si el usuario YA está logueado
  if (esAdmin) {
    console.log("Panel de administrador");
  } else {
    console.log("Panel de usuario");
  }
} else {
  console.log("Por favor inicia sesión");
}
```

#### Ejemplo aplicado — autorización de retiro en cajero

```ts
const saldo: number = 500;
const monto: number = 200;
const pinCorrecto: boolean = true;

let resultado: string;

if (pinCorrecto) {
  if (monto <= saldo) {
    if (monto % 10 === 0) {
      resultado = `✅ Entregando $${monto}. Saldo restante: $${saldo - monto}`;
    } else {
      resultado = "❌ El monto debe ser múltiplo de 10";
    }
  } else {
    resultado = "❌ Saldo insuficiente";
  }
} else {
  resultado = "❌ PIN incorrecto";
}
console.log(resultado);  // ✅ Entregando $200. Saldo restante: $300

// Otros casos:
const monto2: number = 700;
const pinCorrecto2: boolean = true;
let resultado2: string;
if (pinCorrecto2) {
  if (monto2 <= saldo) {
    resultado2 = monto2 % 10 === 0
      ? `✅ Entregando $${monto2}. Saldo restante: $${saldo - monto2}`
      : "❌ El monto debe ser múltiplo de 10";
  } else {
    resultado2 = "❌ Saldo insuficiente";
  }
} else {
  resultado2 = "❌ PIN incorrecto";
}
console.log(resultado2);  // ❌ Saldo insuficiente
```

> **Truco profesional — aplanar condiciones anidadas.** Las condiciones anidadas
> profundas se leen mal. Puedes aplanarlas usando **variables booleanas intermedias**
> o el operador `&&`, sin necesitar funciones:
> ```ts
> const pinOk: boolean = true;
> const saldoOk: boolean = monto <= saldo;
> const montoValido: boolean = monto % 10 === 0;
>
> if (!pinOk) {
>   console.log("❌ PIN incorrecto");
> } else if (!saldoOk) {
>   console.log("❌ Saldo insuficiente");
> } else if (!montoValido) {
>   console.log("❌ Múltiplo de 10");
> } else {
>   console.log(`✅ Entregando $${monto}`);
> }
> ```
> Calcular las condiciones por separado en booleanos nombrados hace el código
> más plano y fácil de leer.

> **Mini-ejercicio (8 min).** Declara `const edadC: number = 20`,
> `const tieneLicencia: boolean = true` y `const alcohol: number = 0.1`.
> Con `if` anidado, imprime: si menor de 18 → `"No tiene edad"`;
> si tiene edad pero **no** licencia → `"Le falta licencia"`;
> si tiene ambas pero `alcohol > 0.3` → `"No puede: alcohol"`;
> en otro caso → `"Puede conducir"`.
> **Extra:** reescribe la misma lógica usando variables booleanas intermedias
> para aplanar el anidamiento.

---

## Parte B · Bucles `for`

Los bucles repiten un bloque. Se clasifican por **qué recorren** y **cómo controlan**
la repetición. Veremos cinco formas + control de flujo (`break`/`continue`).

---

### B.1 · `for` clásico — cuando necesitas el índice

Tres partes: **inicio**; **condición** (mientras sea verdadera, repite); **paso**.

```ts
// Concepto puro
for (let i = 0; i < 5; i++) {
  console.log(`Iteración ${i}`);  // 0, 1, 2, 3, 4
}

// Con paso distinto
for (let i = 0; i <= 100; i += 25) {
  console.log(`Progreso: ${i}%`);  // 0, 25, 50, 75, 100
}

// Cuenta regresiva (decreciente)
for (let i = 5; i >= 1; i--) {
  console.log(`Lanzamiento en ${i}...`);
}
```

#### Ejemplo aplicado — tabla de multiplicar

```ts
const n: number = 7;

for (let i = 1; i <= 10; i++) {
  console.log(`${n} x ${i} = ${n * i}`);
}
```

> **Mini-ejercicio (5 min).** Usa un `for` clásico para sumar todos los números
> del 1 al 100 y mostrar el total (debe dar `5050`). **Extra:** suma solo los **pares**.

---

### B.2 · `for...of` — recorrer colecciones (lo más idiomático)

Recorre los **valores** de un array, string, `Set`, etc. No te preocupas por el índice.

```ts
// Concepto puro
const protocolos: string[] = ["HTTP", "HTTPS", "FTP", "SSH"];

for (const protocolo of protocolos) {
  console.log(protocolo);
}

// Sobre los caracteres de un string
for (const letra of "TS") {
  console.log(letra);  // T, S
}

// ¿Necesitas también el índice? usa entries()
for (const [indice, valor] of protocolos.entries()) {
  console.log(`${indice}: ${valor}`);  // 0: HTTP, 1: HTTPS, ...
}
```

#### Ejemplo aplicado — total de un carrito

```ts
interface Item {
  nombre: string;
  precio: number;
  cantidad: number;
}

const carrito: Item[] = [
  { nombre: "Mouse",   precio: 25, cantidad: 2 },
  { nombre: "Teclado", precio: 80, cantidad: 1 },
  { nombre: "Monitor", precio: 200, cantidad: 3 },
];

let total = 0;
for (const item of carrito) {
  const subtotal = item.precio * item.cantidad;
  console.log(`${item.nombre}: $${subtotal}`);
  total += subtotal;
}
console.log(`TOTAL: $${total}`);  // TOTAL: $730
```

> **Mini-ejercicio (7 min).** Dado `const temps = [18, 22, 25, 30, 19, 27]`,
> usa `for...of` para encontrar e imprimir la temperatura **máxima** y el
> **promedio**. Pista: arranca con `let max = temps[0]` y `let suma = 0`.

---

### B.3 · `for...in` — recorrer las claves de un objeto

Itera sobre los **nombres de las propiedades** de un objeto.

```ts
// Concepto puro
const puertos: Record<string, number> = {
  HTTP: 80,
  HTTPS: 443,
  SSH: 22,
};

for (const servicio in puertos) {
  console.log(`${servicio} → puerto ${puertos[servicio]}`);
}
// HTTP → puerto 80, HTTPS → puerto 443, SSH → puerto 22
```

> ⚠️ **Cuidado:** usa `for...in` para **objetos**, no para arrays. En un array
> te daría los **índices como texto** (`"0"`, `"1"`…), no los valores.
> Para arrays usa `for...of`.

#### Ejemplo aplicado — reporte de configuración

```ts
const config = {
  host: "localhost",
  port: 8080,
  debug: true,
  maxConexiones: 100,
};

console.log("=== Configuración activa ===");
for (const clave in config) {
  const valor = config[clave as keyof typeof config];
  console.log(`${clave.padEnd(15)}: ${valor}`);
}
```

> **Mini-ejercicio (6 min).** Dado `const notas = { mate: 85, fisica: 70,
> quimica: 95, historia: 60 }`, usa `for...in` para imprimir cada materia con su
> nota y, al final, cuántas materias están **aprobadas** (nota ≥ 70).

---

### B.4 · `forEach` / `map` — el estilo funcional

`forEach` ejecuta una función por cada elemento (no devuelve nada).
`map` **transforma** cada elemento y devuelve un **array nuevo**.

```ts
// Concepto puro
const numeros: number[] = [1, 2, 3, 4];

// forEach: para "hacer algo" con cada elemento
numeros.forEach((n) => console.log(n * 10));  // 10, 20, 30, 40

// map: para CREAR una lista transformada
const dobles: number[] = numeros.map((n) => n * 2);
console.log(dobles);  // [2, 4, 6, 8]
```

#### Ejemplo aplicado — normalizar correos

```ts
const emails: string[] = ["  ANA@MAIL.COM ", "Luis@Mail.com", " PEPE@MAIL.COM"];

const limpios: string[] = emails.map((e) => e.trim().toLowerCase());
console.log(limpios);  // ["ana@mail.com", "luis@mail.com", "pepe@mail.com"]

// forEach para reportar, map para transformar
limpios.forEach((e, i) => console.log(`Usuario ${i + 1}: ${e}`));
```

> **Mini-ejercicio (7 min).** Dado `const precios = [100, 250, 80, 500]`, usa
> `map` para crear un array con el **precio + 19% de IVA**, redondeado a 2 decimales
> (`Number(x.toFixed(2))`). Luego usa `forEach` para imprimir cada precio con IVA.

---

### B.5 · `while` y `do-while` — repetir según una condición

Cuando **no sabes de antemano** cuántas veces repetirás.
`while` comprueba **antes**; `do-while` ejecuta **al menos una vez**.

```ts
// while — comprueba la condición ANTES de cada vuelta
let buffer = 1024;       // bytes por enviar
let paquete = 0;

while (buffer > 0) {
  const tam = buffer > 256 ? 256 : buffer;
  paquete++;
  buffer -= tam;
  console.log(`Paquete ${paquete}: ${tam} bytes (quedan ${buffer})`);
}

// do-while — ejecuta AL MENOS UNA VEZ, ideal para reintentos
let intentos = 0;
let conectado = false;

do {
  intentos++;
  console.log(`Intento de conexión #${intentos}...`);
  if (intentos === 3) conectado = true;  // simula éxito al 3er intento
} while (!conectado && intentos < 5);

console.log(conectado ? `Conectado en ${intentos} intentos` : "Falló");
```

> **Mini-ejercicio (6 min).** Simula un dado con `Math.floor(Math.random() * 6) + 1`.
> Usa un `while` (o `do-while`) que siga "tirando" hasta sacar un **6**, contando
> cuántas tiradas fueron necesarias. Imprime el conteo final.

---

### B.6 · `break` y `continue` — controlar el bucle por dentro

- `continue` → **salta** la iteración actual y pasa a la siguiente.
- `break` → **sale** del bucle por completo.

```ts
const paquetes: number[] = [64, 128, -1, 256, 1024, -1, 32];
//                                  ↑              ↑   corruptos (negativos)

// continue: ignora los corruptos pero sigue procesando
console.log("=== con continue ===");
for (const p of paquetes) {
  if (p < 0) {
    console.log("Paquete corrupto ignorado");
    continue;  // salta al siguiente
  }
  console.log(`Procesando ${p} bytes`);
}

// break: se detiene al primer error crítico
console.log("=== con break ===");
for (const p of paquetes) {
  if (p < 0) {
    console.log("Error crítico — deteniendo");
    break;  // sale del bucle
  }
  console.log(`Procesando ${p} bytes`);
}
```

> **Mini-ejercicio (5 min).** Recorre `[3, 7, 2, 9, 11, 4, 6]`. Usa `continue`
> para saltar los **impares** e imprime solo los pares. Luego, en un segundo bucle,
> usa `break` para detenerte en cuanto encuentres un número **mayor que 8**.

---

## Ejemplo combinado — monitor de servidores

Une `if` anidado, `switch`, `for...of`, `while` y `break/continue` en un caso real:

```ts
type Estado = "ok" | "lento" | "caido";

interface Servidor {
  nombre: string;
  estado: Estado;
  latenciaMs: number;
}

const servidores: Servidor[] = [
  { nombre: "web-01", estado: "ok",    latenciaMs: 25 },
  { nombre: "web-02", estado: "lento", latenciaMs: 320 },
  { nombre: "db-01",  estado: "caido", latenciaMs: 0 },
  { nombre: "cache",  estado: "ok",    latenciaMs: 8 },
];

console.log("=== Diagnóstico ===");
let caidos = 0;

for (const s of servidores) {
  if (s.estado === "caido") caidos++;

  // switch para traducir el estado a un ícono
  let icono: string;
  switch (s.estado) {
    case "ok":    icono = "🟢"; break;
    case "lento": icono = "🟡"; break;
    case "caido": icono = "🔴"; break;
    default:      icono = "⚪";
  }

  // if anidado para matizar el diagnóstico
  let diagnostico: string;
  if (s.estado === "ok") {
    if (s.latenciaMs < 50) {
      diagnostico = `${icono} ${s.nombre}: óptimo (${s.latenciaMs}ms)`;
    } else {
      diagnostico = `${icono} ${s.nombre}: aceptable (${s.latenciaMs}ms)`;
    }
  } else {
    diagnostico = `${icono} ${s.nombre}: requiere atención (${s.estado})`;
  }

  console.log(diagnostico);
}

// while para alertar mientras haya caídos (simulado)
let alerta = caidos;
while (alerta > 0) {
  console.log(`🚨 Quedan ${alerta} servidor(es) caído(s) — notificando...`);
  alerta--;
}
console.log(`Resumen: ${caidos}/${servidores.length} caídos`);
```

---

## Reto final de clase (combina todo)

> **Validador de carrito (10–12 min, en parejas).**
> Declara:
> ```ts
> const carritoReto = [
>   { nombre: "A", precio: 50, stock: 3, pedido: 2 },
>   { nombre: "B", precio: 0,  stock: 10, pedido: 1 },
>   { nombre: "C", precio: 30, stock: 1,  pedido: 5 },
> ];
> ```
> Recorre el carrito con `for...of`. Para cada producto:
> 1. Con `if` y `continue`, **salta** los que tengan `precio <= 0` imprimiendo
>    `"Producto sin precio: X"`.
> 2. Si `pedido > stock`, imprime `"Sin stock suficiente de X"` y usa `continue`.
> 3. Si todo está bien, suma `precio * pedido` a un acumulador `let total = 0`.
>
> Al final imprime el **total a pagar**. Resultado esperado del total: `100`.

---

## Resumen de la página 3

**Condicionales (clasificadas por número de caminos):**
- **Simple** (`if`): un solo camino, se ejecuta o no.
- **Dos caminos** (`if/else`): verdadero vs falso. El ternario `cond ? a : b` lo resume en una línea para asignar valores.
- **Múltiples** (`if/else if/else` o `switch`): se evalúan en orden y gana la primera condición verdadera. `switch` es más legible cuando comparas **un mismo valor** contra muchos casos (recuerda el `break`).
- **Enlazadas/anidadas**: un `if` dentro de otro. Para evitar anidamiento profundo, usa **variables booleanas intermedias** o el operador `&&` para aplanar las condiciones.

**Bucles (clasificados por qué recorren):**
- `for` clásico → cuando necesitas el **índice** o un paso personalizado.
- `for...of` → **valores** de arrays/strings (la forma idiomática).
- `for...in` → **claves** de un objeto (¡no para arrays!).
- `forEach`/`map` → estilo funcional; `map` además **devuelve** un array transformado.
- `while`/`do-while` → cuando **no sabes** cuántas vueltas darás; `do-while` corre al menos una vez.
- `break` sale del bucle; `continue` salta a la siguiente iteración.

---

> **Siguiente página →** Página 4: Funciones en TypeScript — parámetros tipados,
> opcionales y por defecto, funciones flecha, sobrecargas y tipos de retorno.
