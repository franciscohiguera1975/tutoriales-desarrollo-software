# Tutorial TypeScript — Página 2
## Tipos de Datos
### Primitivos, arrays, tuplas, enums, uniones y narrowing

---

> **Cómo usar esta página.** Cada bloque sigue el mismo ritmo:
> **1)** el concepto puro, **2)** un ejemplo aplicado real, **3)** un
> **mini-ejercicio** que se resuelve en clase en **5–8 minutos**.
> Escribe el código en el [TypeScript Playground](https://www.typescriptlang.org/play)
> o ejecútalo con `npx tsx archivo.ts`.

---

## Parte A · Tipos primitivos

Los tipos primitivos son los bloques más básicos del sistema de tipos de TypeScript.
TypeScript los infiere automáticamente, pero anotarlos explícitamente hace el código
más legible y seguro.

---

### A.1 · `number` — enteros, decimales, hex y binario

TypeScript usa un único tipo `number` para todos los números (no hay `int` ni `float`
separados, como en otros lenguajes).

```ts
// Concepto puro
const entero: number = 42;
const decimal: number = 3.14;
const negativo: number = -100;
const hexadecimal: number = 0xff;   // 255 en base 16
const binario: number = 0b1010;     // 10 en base 2
const octal: number = 0o17;         // 15 en base 8
const grande: number = 1_000_000;   // _ como separador visual (ES2021)

console.log(hexadecimal); // 255
console.log(binario);     // 10
console.log(grande);      // 1000000

// Constantes especiales de number
console.log(Number.MAX_SAFE_INTEGER); // 9007199254740991
console.log(Number.isFinite(1 / 0)); // false (Infinity no es finito)
console.log(Number.isNaN(0 / 0));    // true
```

#### Ejemplo aplicado — calculadora de descuento

```ts
const precio: number = 199.99;
const porcentaje: number = 15;

const descuento: number = precio * (porcentaje / 100);
const precioFinal: number = Number((precio - descuento).toFixed(2));
console.log(precioFinal); // 169.99

const precio2: number = 50;
const precioFinal2: number = Number((precio2 - precio2 * (10 / 100)).toFixed(2));
console.log(precioFinal2); // 45

const precio3: number = 1299;
const precioFinal3: number = Number((precio3 - precio3 * (20 / 100)).toFixed(2));
console.log(precioFinal3); // 1039.2
```

> **Mini-ejercicio (5 min).** Declara `const celsius: number` con el valor `0`, `100`
> y `37` (uno a la vez o tres variables). Para cada uno, calcula la conversión a
> Fahrenheit con la fórmula `(celsius * 9/5) + 32`, redondea a 1 decimal con
> `Number(resultado.toFixed(1))` e imprímelo. Resultados esperados: `32`, `212` y `98.6`.

---

### A.2 · `string` — texto y template literals

Los strings se definen con comillas simples, dobles o backticks. Los backticks
permiten **template literals**: interpolación y strings multilínea.

```ts
// Concepto puro
const simple: string = "Hola TypeScript";
const doble: string = 'También funciona';
const template: string = `Hola ${"mundo"}`; // template literal

const nombre: string = "Ana";
const edad: number = 28;

// Interpolación: embebe expresiones dentro de ${}
const saludo: string = `Hola, ${nombre}. Tienes ${edad} años.`;
const mayoria: string = `Eres ${edad >= 18 ? "mayor" : "menor"} de edad.`;

// Multilínea sin caracteres de escape
const mensaje: string = `
  Línea 1
  Línea 2
  Línea 3
`.trim();

// Métodos comunes (tipados, el editor autocompleta)
console.log("  hola  ".trim());         // "hola"
console.log("hola".toUpperCase());      // "HOLA"
console.log("2024-06-15".split("-"));   // ["2024", "06", "15"]
console.log("error: fallo".includes("error")); // true
console.log("archivo.ts".endsWith(".ts"));     // true
```

#### Ejemplo aplicado — generador de slugs para URLs

```ts
const titulo1: string = "Introducción a TypeScript";
const slug1: string = titulo1
  .toLowerCase()
  .trim()
  .replace(/[áéíóú]/g, (c) => ({ á:"a", é:"e", í:"i", ó:"o", ú:"u" }[c] ?? c))
  .replace(/[^a-z0-9\s-]/g, "")
  .replace(/\s+/g, "-");
console.log(slug1); // introduccion-a-typescript

const titulo2: string = "  Hola Mundo  ";
const slug2: string = titulo2
  .toLowerCase()
  .trim()
  .replace(/[áéíóú]/g, (c) => ({ á:"a", é:"e", í:"i", ó:"o", ú:"u" }[c] ?? c))
  .replace(/[^a-z0-9\s-]/g, "")
  .replace(/\s+/g, "-");
console.log(slug2); // hola-mundo

const titulo3: string = "10 Trucos para Node.js";
const slug3: string = titulo3
  .toLowerCase()
  .trim()
  .replace(/[áéíóú]/g, (c) => ({ á:"a", é:"e", í:"i", ó:"o", ú:"u" }[c] ?? c))
  .replace(/[^a-z0-9\s-]/g, "")
  .replace(/\s+/g, "-");
console.log(slug3); // 10-trucos-para-nodejs
```

> **Mini-ejercicio (6 min).** Declara `const nombre: string = "ANA"` y
> `const apellido: string = "LOPEZ"`. Construye el nombre completo formateado con
> la primera letra en mayúscula y el resto en minúscula para cada parte, usando
> `charAt(0).toUpperCase() + slice(1).toLowerCase()`. Combínalos en una sola variable
> e imprímela. Resultado esperado: `"Ana Lopez"`.

---

### A.3 · `boolean` — verdadero o falso

```ts
// Concepto puro
const activo: boolean = true;
const eliminado: boolean = false;

// Se infiere sin anotación explícita
const esMayor = 25 >= 18;    // boolean inferido → true
const tieneStock = 0 > 0;    // boolean inferido → false

// Valores "falsy" en TypeScript/JavaScript (importantes para narrowing)
// false, 0, "", null, undefined, NaN → todos se comportan como false en un if
if (!tieneStock) {
  console.log("Sin stock disponible");
}
```

> ⚠️ **Cuidado con la comparación.** Usa `===` (igualdad estricta) y no `==`
> (igualdad débil). Con `==`, `0 == false` es `true`; con `===` es `false`.
> TypeScript favorece `===` y lo verás en todo el código idiomático.

#### Ejemplo aplicado — validador de formulario de registro

```ts
const email: string = "ana@mail.com";
const password: string = "secret123";
const aceptaTerminos: boolean = true;

const emailValido: boolean = email.includes("@") && email.includes(".");
const passSegura: boolean = password.length >= 8;

let resultado: string;
if (!aceptaTerminos) {
  resultado = "Debes aceptar los términos";
} else if (!emailValido) {
  resultado = "Email no válido";
} else if (!passSegura) {
  resultado = "La contraseña debe tener al menos 8 caracteres";
} else {
  resultado = "Registro exitoso";
}
console.log(resultado); // Registro exitoso

// Caso: contraseña corta
const password2: string = "abc";
const passSegura2: boolean = password2.length >= 8;
let resultado2: string;
if (!aceptaTerminos) {
  resultado2 = "Debes aceptar los términos";
} else if (!emailValido) {
  resultado2 = "Email no válido";
} else if (!passSegura2) {
  resultado2 = "La contraseña debe tener al menos 8 caracteres";
} else {
  resultado2 = "Registro exitoso";
}
console.log(resultado2); // La contraseña debe tener al menos 8 caracteres

// Caso: no acepta términos
const aceptaTerminos3: boolean = false;
let resultado3: string;
if (!aceptaTerminos3) {
  resultado3 = "Debes aceptar los términos";
} else {
  resultado3 = "Registro exitoso";
}
console.log(resultado3); // Debes aceptar los términos
```

> **Mini-ejercicio (5 min).** Declara `const anio: number` con el valor `2000`, `1900`,
> `2024` y `2023` (una variable a la vez). Para cada uno, calcula si es bisiesto con
> la regla: divisible entre 4, excepto los siglos (divisibles entre 100) a menos que
> también sean divisibles entre 400. Guarda el resultado en `const esBisiesto: boolean`
> e imprímelo. Resultados esperados: `true`, `false`, `true`, `false`.

---

### A.4 · `null` y `undefined`

`undefined` significa que una variable existe pero **no tiene valor asignado**.
`null` es una asignación **intencional** de "sin valor". TypeScript los distingue.

```ts
// Concepto puro
let sinAsignar: undefined = undefined;
let sinValor: null = null;

// En la práctica: resultado de una búsqueda que puede no encontrar nada
const idBuscado: number = 5;
const usuarioEncontrado: string | null = idBuscado === 1 ? "Ana" : null;

// Operador de coalescencia nula ?? (devuelve el lado derecho si el izquierdo es null/undefined)
const nombre: string = usuarioEncontrado ?? "Invitado";
console.log(nombre); // "Invitado"

// Encadenamiento opcional ?. (no lanza error si algo es null/undefined)
const longitud: number | undefined = usuarioEncontrado?.length;
console.log(longitud); // undefined (no lanza error)
```

> **Truco** Con `strictNullChecks: true` en `tsconfig.json` (activado por defecto
> en proyectos modernos), TypeScript te **obliga** a manejar `null` y `undefined`
> antes de usar un valor. Eso previene el famoso `Cannot read properties of null`.

---

## Parte B · Arrays

---

### B.1 · Sintaxis y métodos básicos tipados

Hay dos formas equivalentes de anotar un array. La primera es más común en código
TypeScript moderno.

```ts
// Concepto puro — dos sintaxis equivalentes
const numeros: number[] = [1, 2, 3, 4, 5];
const textos: Array<string> = ["a", "b", "c"];   // forma genérica

// TypeScript infiere el tipo del array si lo inicializas
const inferido = [10, 20, 30]; // number[] inferido

// Métodos tipados: el compilador conoce el tipo del elemento
const dobles: number[] = numeros.map((n) => n * 2);       // [2, 4, 6, 8, 10]
const pares: number[] = numeros.filter((n) => n % 2 === 0); // [2, 4]
const suma: number = numeros.reduce((acc, n) => acc + n, 0); // 15

// Mutación (cambia el array original)
numeros.push(6);       // agrega al final
numeros.unshift(0);    // agrega al inicio
const ultimo = numeros.pop();   // elimina y devuelve el último
const primero = numeros.shift(); // elimina y devuelve el primero

// Búsqueda
const existe: boolean = numeros.includes(3);       // true
const indice: number = numeros.indexOf(3);         // posición o -1
const encontrado: number | undefined = numeros.find((n) => n > 4); // 5
```

#### Ejemplo aplicado — procesador de calificaciones

```ts
const calificaciones: number[] = [85, 92, 70, 55, 98, 63, 78];

const aprobados: number[] = calificaciones.filter((n) => n >= 70);
const reprobados: number[] = calificaciones.filter((n) => n < 70);
const promedio: number = Number(
  (calificaciones.reduce((acc, n) => acc + n, 0) / calificaciones.length).toFixed(1)
);
const maxima: number = Math.max(...calificaciones);
const minima: number = Math.min(...calificaciones);

console.log(`Aprobados: ${aprobados.length} | Reprobados: ${reprobados.length}`);
console.log(`Promedio: ${promedio} | Máxima: ${maxima} | Mínima: ${minima}`);
// Aprobados: 5 | Reprobados: 2
// Promedio: 77.3 | Máxima: 98 | Mínima: 55
```

> **Mini-ejercicio (7 min).** Dado `const precios: number[] = [120, 35, 450, 89, 210, 15]`,
> usa métodos de array para: (1) filtrar solo los que cuestan más de 100, (2) aplicar
> un descuento del 10% a cada uno de esos con `map`, (3) calcular el total con `reduce`.
> Imprime la lista con descuento y el total.

---

### B.2 · Arrays de objetos

Los arrays más comunes en aplicaciones reales contienen objetos. TypeScript tipa
cada propiedad y autocompletará al acceder a ellas.

```ts
// Concepto puro
type Producto = {
  id: number;
  nombre: string;
  precio: number;
  disponible: boolean;
};

const catalogo: Producto[] = [
  { id: 1, nombre: "Laptop",  precio: 999,  disponible: true },
  { id: 2, nombre: "Mouse",   precio: 25,   disponible: true },
  { id: 3, nombre: "Monitor", precio: 350,  disponible: false },
];

// TypeScript sabe que cada "p" es de tipo Producto
const disponibles: Producto[] = catalogo.filter((p) => p.disponible);
const nombres: string[] = catalogo.map((p) => p.nombre);
const masBarato: Producto | undefined = catalogo.reduce((min, p) =>
  p.precio < min.precio ? p : min
);

console.log(nombres);                  // ["Laptop", "Mouse", "Monitor"]
console.log(masBarato?.nombre);       // "Mouse"
console.log(disponibles.length);      // 2
```

> **Mini-ejercicio (8 min).** Crea `const alumnos: { nombre: string; nota: number }[]`
> con 5 alumnos. Luego, con código corrido, calcula y declara: `const aprobados` (nota >= 60),
> `const reprobados`, `const promedio` y `const notaMayor` (nombre del alumno con la
> nota más alta). Imprime los cuatro valores en consola.

---

## Parte C · Tuplas

---

### C.1 · Tuplas básicas y con nombre

Una **tupla** es un array con un número **fijo** de elementos y **tipos definidos
por posición**. A diferencia de un array normal, cada posición tiene un tipo distinto.

```ts
// Concepto puro
type Coordenada = [number, number];           // [x, y]
type RGB = [number, number, number];          // [rojo, verde, azul]
type Entrada = [string, number];              // [clave, valor]

const punto: Coordenada = [10.5, -3.2];
const color: RGB = [255, 128, 0];            // naranja
const par: Entrada = ["temperatura", 36.6];

// Desestructuración (la forma más cómoda de usar tuplas)
const [x, y] = punto;
const [rojo, verde, azul] = color;
const [clave, valor] = par;

console.log(`Punto: x=${x}, y=${y}`);         // Punto: x=10.5, y=-3.2
console.log(`Color: rgb(${rojo},${verde},${azul})`); // Color: rgb(255,128,0)

// Tuplas con nombre (TS 4.0+) — mejoran la legibilidad
type Rango = [inicio: number, fin: number];
const horario: Rango = [9, 18];              // de 9:00 a 18:00
```

> ⚠️ **Tuplas vs objetos.** Usa tuplas para datos **posicionales** breves (coordenadas,
> rangos, pares clave-valor de una API). Si tienes 3 o más campos con significado
> propio, es mejor un objeto `{ x, y, z }` que una tupla `[x, y, z]`.

#### Ejemplo aplicado — retorno múltiple con tupla

Uno de los casos de uso más prácticos de las tuplas es representar **múltiples valores**
relacionados sin crear un objeto intermedio.

```ts
// División segura expresada como código corrido con tupla
const a: number = 10;
const b: number = 2;

const divisionResultado: [number, string] = b === 0
  ? [0, "Error: división por cero"]
  : [a / b, "ok"];

const [resultado, estado] = divisionResultado;
console.log(`${estado}: ${resultado}`);  // ok: 5

// Caso de división por cero
const c: number = 5;
const d: number = 0;

const divisionResultado2: [number, string] = d === 0
  ? [0, "Error: división por cero"]
  : [c / d, "ok"];

const [res2, estado2] = divisionResultado2;
console.log(`${estado2}: ${res2}`);      // Error: división por cero: 0

// Patrón común en React: [valor, setter] (como useState)
type EstadoTupla = [string, (v: string) => void];
```

> **Mini-ejercicio (6 min).** Dado `const nums: number[] = [5, 3, 9, 1, 7]`, calcula
> el mínimo y el máximo con `Math.min(...nums)` y `Math.max(...nums)`. Guárdalos en
> una tupla `const minMax: [number, number] = [min, max]`, desestructúrala en
> `const [min, max]` e imprime ambos valores. Resultado esperado: `1` y `9`.

---

## Parte D · Enums

---

### D.1 · Enum numérico y de string

Un `enum` agrupa constantes relacionadas bajo un mismo nombre, evitando "magic strings"
o números sueltos en el código.

```ts
// Enum numérico (los valores son 0, 1, 2, … por defecto)
enum Direccion {
  Norte,  // 0
  Sur,    // 1
  Este,   // 2
  Oeste,  // 3
}

const rumbo: Direccion = Direccion.Norte;
console.log(rumbo);           // 0
console.log(Direccion[0]);    // "Norte" (mapeo inverso automático)

// Enum numérico con valor de inicio personalizado
enum CodigoHTTP {
  OK = 200,
  NoEncontrado = 404,
  Error = 500,
}

// Enum de string (recomendado: los valores son legibles en logs y redes)
enum Rol {
  Admin    = "ADMIN",
  Editor   = "EDITOR",
  Lector   = "READER",
}

const miRol: Rol = Rol.Editor;
console.log(miRol); // "EDITOR"
```

### D.2 · Union de literales — la alternativa moderna

Para muchos casos, una **union de literales** es más simple y liviana que un `enum`.
TypeScript la recomienda cuando no necesitas el objeto enum en tiempo de ejecución.

```ts
// Union de literales: más simple, cero código JS generado
type Estado = "pendiente" | "procesando" | "completado" | "error";
type Prioridad = "baja" | "media" | "alta";

const idPedido: number = 1;
const estadoPedido: Estado = "procesando";
console.log(`Pedido #${idPedido}: ${estadoPedido}`); // Pedido #1: procesando

// El tipo garantiza que solo se usan valores válidos en tiempo de compilación:
// const estadoInvalido: Estado = "cancelado"; // Error de compilación
```

> **Truco** Prefiere **union de literales** cuando los valores son strings conocidos
> y no necesitas iterar sobre ellos. Usa `enum` cuando necesites el mapeo inverso
> numérico o cuando tu equipo viene de lenguajes como C# o Java.

#### Ejemplo aplicado — sistema de tickets de soporte

```ts
type PrioridadTicket = "baja" | "media" | "alta" | "critica";

interface Ticket {
  id: number;
  titulo: string;
  prioridad: PrioridadTicket;
  resuelto: boolean;
}

const prefijos: Record<PrioridadTicket, string> = {
  baja:    "⚪",
  media:   "🟡",
  alta:    "🟠",
  critica: "🔴",
};

const tickets: Ticket[] = [
  { id: 1, titulo: "Botón no funciona",  prioridad: "baja",    resuelto: true  },
  { id: 2, titulo: "Pago falla",         prioridad: "critica", resuelto: false },
  { id: 3, titulo: "Lentitud en carga",  prioridad: "media",   resuelto: false },
];

for (const t of tickets) {
  const estado = t.resuelto ? "✅" : "⏳";
  const etiqueta = `${estado} ${prefijos[t.prioridad]} [#${t.id}] ${t.titulo}`;
  console.log(etiqueta);
}
// ✅ ⚪ [#1] Botón no funciona
// ⏳ 🔴 [#2] Pago falla
// ⏳ 🟡 [#3] Lentitud en carga
```

> **Mini-ejercicio (6 min).** Define `type Semaforo = "rojo" | "amarillo" | "verde"`.
> Crea un array `const semaforos: Semaforo[] = ["rojo", "verde", "amarillo", "verde"]`
> y recórrelo con `for...of`. Dentro del bucle, usa `if/else` para imprimir
> `"Detente"`, `"Precaución"` o `"Avanza"` según el valor de cada elemento.

---

## Parte E · Union types y Literal types

---

### E.1 · Union types — un valor de varios tipos posibles

El operador `|` permite que una variable o parámetro acepte **más de un tipo**.

```ts
// Concepto puro
type Id = string | number;     // puede ser "abc-123" o 42

const idNumerico: Id = 42;
const idTexto: Id = "abc-123";
console.log(`Buscando con id: ${idNumerico} (tipo: ${typeof idNumerico})`);
// Buscando con id: 42 (tipo: number)
console.log(`Buscando con id: ${idTexto} (tipo: ${typeof idTexto})`);
// Buscando con id: abc-123 (tipo: string)

// Union con tipos compuestos
type Respuesta =
  | { exito: true;  datos: string[] }
  | { exito: false; error: string  };

const respuestaOk: Respuesta = { exito: true, datos: ["item1", "item2"] };
const respuestaError: Respuesta = { exito: false, error: "No se pudo conectar" };
```

### E.2 · Literal types — valores exactos como tipos

Un **literal type** restringe una variable a un valor específico (no solo un tipo).

```ts
// Concepto puro
type Direccion = "izquierda" | "derecha" | "arriba" | "abajo";
type Dado = 1 | 2 | 3 | 4 | 5 | 6;
type Activado = true; // solo puede ser true

let movimiento: Direccion = "arriba";   // ok
// movimiento = "diagonal";             // Error: no es un Direccion válido

let tirada: Dado = 4;                  // ok
// tirada = 7;                         // Error: 7 no es un Dado válido

// Uso directo con template literal
const dir: Direccion = "izquierda";
const pasos: number = 3;
console.log(`Moviéndose ${pasos} paso(s) hacia ${dir}`);
// Moviéndose 3 paso(s) hacia izquierda
```

> **Mini-ejercicio (5 min).** Define `type Talla = "XS" | "S" | "M" | "L" | "XL"`.
> Declara variables `const talla1: Talla = "XS"`, `talla2 = "S"`, `talla3 = "M"`,
> `talla4 = "L"`, `talla5 = "XL"`. Para cada una, calcula `const extra: number`
> usando `if/else` (0 para S/M, 5 para XS/L, 10 para XL) e imprímelo.

---

## Parte F · `any`, `unknown` y `never`

---

### F.1 · Los tres casos extremos del sistema de tipos

```ts
// ─── any ───────────────────────────────────────────────────────────────────
// Desactiva el chequeo de tipos. Es el "apagado de emergencia" de TypeScript.
// Evítalo salvo al migrar código JavaScript o al trabajar con librerías sin tipos.
let cualquierCosa: any = "texto";
cualquierCosa = 42;       // ok, any acepta cualquier valor
cualquierCosa = true;     // ok
cualquierCosa.metodoFalso(); // NO da error en compilación, pero falla en runtime

// ─── unknown ───────────────────────────────────────────────────────────────
// Como any, acepta cualquier valor. Pero NO puedes usarlo sin comprobar el tipo.
// Es la versión SEGURA de any para datos de fuentes externas (APIs, JSON, input).
let dato: unknown = "hola";
dato = 42;                 // ok, acepta cualquier valor

// console.log(dato.toUpperCase()); // Error: Object is of type 'unknown'
if (typeof dato === "string") {
  console.log(dato.toUpperCase()); // ok: TypeScript sabe que es string aquí
}

// ─── never ─────────────────────────────────────────────────────────────────
// Representa algo que NUNCA ocurre: una rama de código inalcanzable.
// Útil para verificar exhaustividad en switches.
// Una expresión de tipo never se usa así en tiempo de compilación:
type SinValor = string & number; // never: no existe valor que sea string Y number
```

#### Ejemplo aplicado — parseador seguro de datos externos

```ts
// unknown es perfecto para datos que vienen de una API o de JSON.parse()
// Caso 1: el valor es un número
const valorDesconocido1: unknown = 25;
let edadParsada1: number;
if (typeof valorDesconocido1 === "number" && !Number.isNaN(valorDesconocido1)) {
  edadParsada1 = valorDesconocido1;
} else if (typeof valorDesconocido1 === "string") {
  const n = Number(valorDesconocido1);
  edadParsada1 = !Number.isNaN(n) ? n : 0;
} else {
  edadParsada1 = 0;
}
console.log(edadParsada1); // 25

// Caso 2: el valor es un string numérico
const valorDesconocido2: unknown = "30";
let edadParsada2: number;
if (typeof valorDesconocido2 === "number" && !Number.isNaN(valorDesconocido2)) {
  edadParsada2 = valorDesconocido2;
} else if (typeof valorDesconocido2 === "string") {
  const n = Number(valorDesconocido2);
  edadParsada2 = !Number.isNaN(n) ? n : 0;
} else {
  edadParsada2 = 0;
}
console.log(edadParsada2); // 30

// never para verificar que un switch sea exhaustivo
type Forma = "circulo" | "cuadrado" | "triangulo";

const forma: Forma = "circulo";
const medida: number = 5;
let areaForma: number;

switch (forma) {
  case "circulo":   areaForma = Math.PI * medida ** 2; break;
  case "cuadrado":  areaForma = medida ** 2; break;
  case "triangulo": areaForma = (Math.sqrt(3) / 4) * medida ** 2; break;
  default:
    // Si agregamos una nueva Forma y olvidamos el case, TypeScript da error aquí:
    const _agotado: never = forma;
    areaForma = 0;
}
console.log(areaForma.toFixed(2)); // 78.54
```

> **Mini-ejercicio (7 min).** Declara `const v1: unknown = 42`, `const v2: unknown = "hola"`,
> `const v3: unknown = true`, `const v4: unknown = null`, `const v5: unknown = [1, 2, 3]`.
> Para cada uno, usa `if/else if` con `typeof` para imprimir: `"Número: X"` si es number,
> `"Texto de N caracteres"` si es string, `"Booleano: X"` si es boolean,
> o `"Tipo desconocido"` en cualquier otro caso.

---

## Parte G · Type Narrowing (estrechamiento de tipos)

El **narrowing** es el proceso por el que TypeScript **reduce** el tipo de una variable
dentro de un bloque gracias a comprobaciones en tiempo de compilación.

---

### G.1 · Técnicas de narrowing

```ts
// ─── typeof ────────────────────────────────────────────────────────────────
let valorFormateado: string | number = "hola mundo";
let resultadoFormato: string;
if (typeof valorFormateado === "string") {
  resultadoFormato = valorFormateado.toUpperCase(); // TypeScript sabe que es string
} else {
  resultadoFormato = valorFormateado.toFixed(2);   // TypeScript sabe que es number
}
console.log(resultadoFormato); // "HOLA MUNDO"

// ─── truthiness ────────────────────────────────────────────────────────────
const nombreOpcional: string | null = null;
const saludo: string = nombreOpcional
  ? `Hola, ${nombreOpcional}`  // nombreOpcional es string (no null)
  : "Hola, invitado";
console.log(saludo); // "Hola, invitado"

// ─── igualdad (===) ────────────────────────────────────────────────────────
const valorA: string | number = 42;
const valorB: string | number = 42;
const comparacion: string = valorA === valorB ? `Iguales: ${valorA}` : "Distintos";
console.log(comparacion); // "Iguales: 42"

// ─── Array.isArray ─────────────────────────────────────────────────────────
const datoArray: string | string[] = ["uno", "dos", "tres"];
const conteo: number = Array.isArray(datoArray) ? datoArray.length : 1;
console.log(conteo); // 3

// ─── operador "in" — verifica si una propiedad existe en un objeto ─────────
type Perro = { nombre: string; raza: string };
type Gato  = { nombre: string; vidas: number };

const animalA: Perro | Gato = { nombre: "Rex",   raza: "Labrador" };
const animalB: Perro | Gato = { nombre: "Mishi", vidas: 9 };

const descAnimalA: string = "raza" in animalA
  ? `Perro: ${animalA.nombre}, raza ${(animalA as Perro).raza}`
  : `Gato: ${animalA.nombre}, ${(animalA as Gato).vidas} vidas`;

const descAnimalB: string = "raza" in animalB
  ? `Perro: ${animalB.nombre}, raza ${(animalB as Perro).raza}`
  : `Gato: ${animalB.nombre}, ${(animalB as Gato).vidas} vidas`;

console.log(descAnimalA); // Perro: Rex, raza Labrador
console.log(descAnimalB); // Gato: Mishi, 9 vidas
```

#### Ejemplo aplicado — procesador de pagos multi-método

```ts
type PagoTarjeta  = { metodo: "tarjeta";  numero: string; cvv: number };
type PagoTransfer = { metodo: "transferencia"; banco: string; cuenta: string };
type PagoEfectivo = { metodo: "efectivo"; moneda: string };

type Pago = PagoTarjeta | PagoTransfer | PagoEfectivo;

const pagos: Array<{ pago: Pago; monto: number }> = [
  { pago: { metodo: "tarjeta", numero: "4111111111111234", cvv: 123 }, monto: 500 },
  { pago: { metodo: "transferencia", banco: "BBVA", cuenta: "ES12345" }, monto: 200 },
  { pago: { metodo: "efectivo", moneda: "MXN" }, monto: 150 },
];

for (const { pago, monto } of pagos) {
  let descripcion: string;
  switch (pago.metodo) {
    case "tarjeta":
      descripcion = `Cobrando $${monto} a tarjeta ****${pago.numero.slice(-4)}`;
      break;
    case "transferencia":
      descripcion = `Transfiriendo $${monto} via ${pago.banco} a ${pago.cuenta}`;
      break;
    case "efectivo":
      descripcion = `Recibiendo $${monto} en ${pago.moneda}`;
      break;
  }
  console.log(descripcion);
}
// Cobrando $500 a tarjeta ****1234
// Transfiriendo $200 via BBVA a ES12345
// Recibiendo $150 en MXN
```

> **Mini-ejercicio (8 min).** Define `type Figura = { tipo: "circulo"; radio: number }
> | { tipo: "rectangulo"; base: number; altura: number }`.
> Declara `const figuras: Figura[]` con un círculo de radio 7 y un rectángulo de 10×5.
> Recorre el array con `for...of` y usa narrowing con `switch` (o `if` sobre `tipo`) para
> calcular e imprimir el área de cada figura.

---

## Parte H · Objetos tipados inline y propiedades opcionales

---

### H.1 · Objeto tipado inline y propiedades opcionales `?`

TypeScript permite tipar objetos directamente en la anotación (tipo inline) sin
declarar un `type` o `interface` separado. Útil para objetos simples y de un solo uso.

```ts
// Concepto puro — objeto tipado inline
let usuario: { nombre: string; edad: number; email: string };
usuario = { nombre: "Ana", edad: 28, email: "ana@mail.com" };

// Propiedades opcionales con ?
// La propiedad puede estar presente o ser undefined
let config: {
  host: string;
  puerto: number;
  debug?: boolean;      // opcional
  timeout?: number;     // opcional
};

config = { host: "localhost", puerto: 3000 };             // ok, sin opcionales
config = { host: "api.com", puerto: 443, debug: true };   // ok, con debug

// Propiedades de solo lectura con readonly
const constante: { readonly id: number; valor: string } = { id: 1, valor: "a" };
// constante.id = 2; // Error: no se puede asignar a una propiedad readonly

// Objetos anidados inline
let pedido: {
  id: number;
  cliente: { nombre: string; email: string };
  total: number;
};

pedido = {
  id: 101,
  cliente: { nombre: "Luis", email: "luis@mail.com" },
  total: 250,
};
```

> ⚠️ **Inline vs `type` vs `interface`.** Para objetos que se reutilizan en varios
> lugares, declara un `type` o `interface` con nombre. Los tipos inline son cómodos
> para variables locales o parámetros de función de uso único.

#### Ejemplo aplicado — perfil de usuario con campos opcionales

```ts
type Perfil = {
  readonly id: number;
  nombre: string;
  apellido: string;
  email: string;
  telefono?: string;        // no todos los usuarios lo dan
  avatar?: string;          // URL de foto de perfil
  fechaNacimiento?: string; // ISO 8601: "1995-08-20"
};

const perfil1: Perfil = {
  id: 1,
  nombre: "Carlos",
  apellido: "Ruiz",
  email: "carlos@mail.com",
  telefono: "+52 55 1234 5678",
};

console.log(`[#${perfil1.id}] ${perfil1.nombre} ${perfil1.apellido} — ${perfil1.email}`);
if (perfil1.telefono) console.log(`  Tel: ${perfil1.telefono}`);
if (perfil1.avatar)   console.log(`  Avatar: ${perfil1.avatar}`);

const perfil2: Perfil = {
  id: 2,
  nombre: "Sara",
  apellido: "López",
  email: "sara@mail.com",
  fechaNacimiento: "1998-03-12",
};

console.log(`[#${perfil2.id}] ${perfil2.nombre} ${perfil2.apellido} — ${perfil2.email}`);
if (perfil2.telefono) console.log(`  Tel: ${perfil2.telefono}`);
if (perfil2.fechaNacimiento) {
  const anios = new Date().getFullYear() - new Date(perfil2.fechaNacimiento).getFullYear();
  console.log(`  Edad aprox.: ${anios} años`);
}
```

> **Mini-ejercicio (8 min).** Define un tipo `Producto` inline con `id: number`,
> `nombre: string`, `precio: number`, `descuento?: number` y `tags?: string[]`.
> Crea `const productos: Producto[]` con 3 productos (algunos sin descuento, algunos
> sin tags). Recorre el array con `for...of` y, para cada uno, calcula el precio final
> con código corrido: si tiene `descuento`, aplícalo; si no, usa el precio tal cual.
> Imprime el nombre y el precio final de cada producto.

---

## Ejemplo combinado — inventario de tienda online

Une primitivos, arrays, tuplas, enums, unions, narrowing y objetos opcionales en
un caso real de gestión de inventario:

```ts
// ── Tipos y enums ──────────────────────────────────────────────────────────
type EstadoProducto = "activo" | "agotado" | "descontinuado";
type Categoria = "electronica" | "ropa" | "hogar" | "deportes";

type Dimension = [ancho: number, alto: number, profundidad: number]; // tupla con nombre

interface ProductoInventario {
  readonly id: number;
  nombre: string;
  precio: number;
  stock: number;
  categoria: Categoria;
  estado: EstadoProducto;
  dimensiones?: Dimension;  // opcional: no todos los productos tienen medidas
  tags?: string[];
}

// ── Datos ──────────────────────────────────────────────────────────────────
const inventario: ProductoInventario[] = [
  {
    id: 1, nombre: "Laptop Pro",    precio: 1299, stock: 15,
    categoria: "electronica", estado: "activo",
    dimensiones: [35, 25, 2], tags: ["trabajo", "portatil"],
  },
  {
    id: 2, nombre: "Camiseta Slim", precio: 29,   stock: 0,
    categoria: "ropa",         estado: "agotado",
  },
  {
    id: 3, nombre: "Lámpara LED",   precio: 49,   stock: 40,
    categoria: "hogar",        estado: "activo",
    tags: ["iluminacion", "ahorro"],
  },
  {
    id: 4, nombre: "Bicicleta MTB", precio: 499,  stock: 5,
    categoria: "deportes",     estado: "activo",
    dimensiones: [180, 100, 60],
  },
];

// ── Íconos de estado por narrowing con switch ──────────────────────────────
const iconosEstado: Record<EstadoProducto, string> = {
  activo:        "✅",
  agotado:       "❌",
  descontinuado: "🚫",
};

// ── Procesamiento: mostrar inventario ──────────────────────────────────────
console.log("=== INVENTARIO ===");
for (const p of inventario) {
  const icono = iconosEstado[p.estado];
  const dim = p.dimensiones
    ? ` | ${p.dimensiones[0]}×${p.dimensiones[1]}×${p.dimensiones[2]} cm`
    : "";
  const tags = p.tags ? ` [${p.tags.join(", ")}]` : "";
  console.log(`${icono} [${p.categoria.toUpperCase()}] ${p.nombre} — $${p.precio} (${p.stock} uds.)${dim}${tags}`);
}

// ── Resumen ────────────────────────────────────────────────────────────────
const activos: ProductoInventario[] = inventario.filter((p) => p.estado === "activo");
const valorTotal: number = activos.reduce((acc, p) => acc + p.precio * p.stock, 0);
const masBarato: ProductoInventario = activos.reduce((min, p) => p.precio < min.precio ? p : min);
const masCaro: ProductoInventario   = activos.reduce((max, p) => p.precio > max.precio ? p : max);

console.log(`\n=== RESUMEN ===`);
console.log(`Activos: ${activos.length} | Valor total en stock: $${valorTotal.toLocaleString()}`);
console.log(`Más barato: ${masBarato.nombre} ($${masBarato.precio})`);
console.log(`Más caro:   ${masCaro.nombre} ($${masCaro.precio})`);
```

Salida:
```
=== INVENTARIO ===
✅ [ELECTRONICA] Laptop Pro — $1299 (15 uds.) | 35×25×2 cm [trabajo, portatil]
❌ [ROPA] Camiseta Slim — $29 (0 uds.)
✅ [HOGAR] Lámpara LED — $49 (40 uds.) [iluminacion, ahorro]
✅ [DEPORTES] Bicicleta MTB — $499 (5 uds.) | 180×100×60 cm

=== RESUMEN ===
Activos: 3 | Valor total en stock: $23,420
Más barato: Lámpara LED ($49)
Más caro:   Laptop Pro ($1299)
```

---

## Resumen de la página 2

**Tipos primitivos:**
- `number` cubre enteros, decimales, hex (`0xff`) y binario (`0b1010`). Usa `_` como separador visual en números grandes.
- `string`: prefiere **template literals** (backticks) para interpolación y strings multilínea.
- `boolean`: usa `===` para comparaciones; conoce los valores falsy (`false`, `0`, `""`, `null`, `undefined`, `NaN`).
- `null` intencional vs `undefined` sin asignar. Los operadores `??` y `?.` simplifican su manejo.

**Arrays y tuplas:**
- Anota arrays con `tipo[]` (preferida) o `Array<tipo>` (genérica).
- Arrays de objetos: TypeScript tipa cada propiedad y habilita autocompletado en métodos (`map`, `filter`, `reduce`).
- Tuplas: array de longitud fija con tipos por posición. Ideales para datos posicionales breves y pares de valores relacionados.

**Enums y unions:**
- `enum` numérico para mapeos con índice; `enum` de string cuando los valores deben ser legibles.
- Prefiere **union de literales** (`"a" | "b" | "c"`) para conjuntos de strings conocidos: más simple y sin código JS generado.

**Casos especiales:**
- `any`: desactiva el chequeo. Evitar salvo en migraciones.
- `unknown`: acepta todo, pero **obliga** a comprobar el tipo antes de usar. Es la versión segura de `any` para datos externos.
- `never`: representa lo imposible; útil para exhaustividad en `switch`.

**Narrowing:**
- `typeof` distingue primitivos. `Array.isArray` distingue arrays. `in` comprueba propiedades de objetos. `switch` sobre un campo discriminador (`tipo`, `metodo`) es la técnica más idiomática.

**Objetos tipados:**
- Inline para variables locales de uso único; `type`/`interface` para reutilizar.
- La propiedad opcional `?` permite que el campo esté ausente (es `undefined`). `readonly` protege campos que no deben cambiar tras la inicialización.

---

> **Siguiente página →** Página 3: Control de Flujo — condicionales `if`
> (simple, dos caminos, múltiples, enlazadas) y bucles `for`.
