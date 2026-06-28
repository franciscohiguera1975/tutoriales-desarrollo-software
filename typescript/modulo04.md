# Tutorial TypeScript — Página 4
## Funciones
### Parámetros tipados, opcionales, flecha, orden superior y sobrecargas

---

> **Cómo usar esta página.** Cada bloque sigue el mismo ritmo:
> **1)** el concepto puro, **2)** un ejemplo aplicado real, **3)** un
> **mini-ejercicio** que se resuelve en clase en **5–8 minutos**.
> Escribe el código en el [TypeScript Playground](https://www.typescriptlang.org/play)
> o ejecútalo con `npx tsx archivo.ts`.

---

## Parte A · Declaración y tipado básico

Las funciones son el bloque de construcción más fundamental de cualquier programa.
En TypeScript, cada parámetro y el valor de retorno pueden —y deben— tener un tipo explícito.

---

### A.1 · Declaración con parámetros tipados y tipo de retorno explícito

Cada parámetro lleva `: Tipo` y, tras el paréntesis de cierre, `: TipoRetorno` declara
qué devolverá la función. Si hay contradicción, TypeScript lo señala en tiempo de compilación.

```ts
// Concepto puro
function suma(a: number, b: number): number {
  return a + b;
}

function saludar(nombre: string): string {
  return `Hola, ${nombre}`;
}

// TypeScript verifica el argumento Y el retorno
console.log(suma(3, 4));        // 7
console.log(saludar("Sofía")); // Hola, Sofía

// Error de compilación — a propósito (descoméntalo para verlo):
// suma("3", 4);  // Argument of type 'string' is not assignable to 'number'
```

> ⚠️ **Tipo de retorno explícito vs inferido.** TypeScript puede inferir el retorno
> en la mayoría de los casos, pero declararlo explícitamente actúa como un contrato:
> si olvidas un `return` o devuelves el tipo incorrecto, el error aparece en la
> **definición** de la función, no en quien la llama.

#### Ejemplo aplicado — calculadora de descuento

```ts
function calcularDescuento(precio: number, porcentaje: number): number {
  const descuento = precio * (porcentaje / 100);
  return Number((precio - descuento).toFixed(2));
}

function resumenCompra(producto: string, precio: number, descuento: number): string {
  const final = calcularDescuento(precio, descuento);
  return `${producto}: $${precio} → $${final} (${descuento}% off)`;
}

console.log(resumenCompra("Teclado", 120, 15));   // Teclado: $120 → $102 (15% off)
console.log(resumenCompra("Monitor", 350, 20));   // Monitor: $350 → $280 (20% off)
console.log(resumenCompra("Mouse", 45, 0));       // Mouse: $45 → $45 (0% off)
```

> **Mini-ejercicio (5 min).** Escribe `function areaRectangulo(base: number,
> altura: number): number` y `function perimetroRectangulo(base: number,
> altura: number): number`. Pruébalas con base `8` y altura `5`.
> El área debe dar `40` y el perímetro `26`.

---

### A.2 · Funciones flecha (arrow functions)

Las funciones flecha (`=>`) tienen una sintaxis más compacta. Cuando el cuerpo
es una sola expresión, las llaves y el `return` son opcionales (retorno implícito).

```ts
// Concepto puro

// Declaración tradicional
function cuadrado(n: number): number {
  return n * n;
}

// Flecha equivalente — con cuerpo explícito
const cuadradoFlecha = (n: number): number => {
  return n * n;
};

// Flecha con retorno implícito (una expresión, sin llaves)
const cuadradoCorto = (n: number): number => n * n;

// Sin parámetros
const ahora = (): string => new Date().toLocaleTimeString();

// Un solo parámetro (paréntesis opcionales, pero recomendados en TS)
const doble = (n: number): number => n * 2;

console.log(cuadrado(5));       // 25
console.log(cuadradoFlecha(5)); // 25
console.log(cuadradoCorto(5));  // 25
console.log(doble(7));          // 14
console.log(ahora());           // e.g. "10:34:22"
```

> **Truco.** Prefiere **declaración tradicional** para funciones con nombre en el
> módulo principal (permiten hoisting y son más legibles en stack traces).
> Usa **funciones flecha** para callbacks, métodos de array y cuando quieras
> capturar el `this` del contexto léxico.

#### Ejemplo aplicado — pipeline de transformación de texto

```ts
const trim      = (s: string): string => s.trim();
const aMinusculas = (s: string): string => s.toLowerCase();
const capitalizar = (s: string): string =>
  s.charAt(0).toUpperCase() + s.slice(1);
const quitarEspacios = (s: string): string => s.replace(/\s+/g, "_");

// Encadenar transformaciones manualmente
function normalizarUsuario(nombre: string): string {
  return quitarEspacios(capitalizar(aMinusculas(trim(nombre))));
}

const entradas = ["  ANA GARCÍA  ", " luis rodríguez", "PEDRO  LÓPEZ "];
entradas.forEach((e) => console.log(normalizarUsuario(e)));
// Ana_garcía
// Luis_rodríguez
// Pedro__lópez  (doble espacio interno → doble guión)
```

> **Mini-ejercicio (6 min).** Escribe tres funciones flecha con retorno implícito:
> `esPar`, `esPositivo` y `enCelsius` (convierte Fahrenheit → Celsius con
> `(f - 32) * 5/9`, redondeado a 1 decimal). Pruébalas con `4`, `-3` y `98.6`.
> Resultado esperado de temperatura: `37.0`.

---

## Parte B · Parámetros avanzados

---

### B.1 · Parámetros opcionales `?` y parámetros por defecto

Un parámetro **opcional** (`nombre?`) puede omitirse; dentro de la función su tipo es
`T | undefined`. Un parámetro **por defecto** usa `= valor` y garantiza que siempre
habrá un valor; nunca es `undefined`.

```ts
// Concepto puro

// Opcional: el parámetro puede no llegarse a pasar
function crearEtiqueta(texto: string, mayusculas?: boolean): string {
  // Dentro, mayusculas es boolean | undefined
  if (mayusculas) {
    return `[${texto.toUpperCase()}]`;
  }
  return `[${texto}]`;
}

console.log(crearEtiqueta("info"));          // [info]
console.log(crearEtiqueta("alerta", true)); // [ALERTA]

// Por defecto: si no se pasa, usa el valor indicado
function repetir(texto: string, veces: number = 3): string {
  return texto.repeat(veces);
}

console.log(repetir("ha"));    // hahaha  (usa el default 3)
console.log(repetir("ha", 5)); // hahahahaha
```

> ⚠️ **Regla de orden.** Los parámetros opcionales y los que tienen valor por
> defecto deben ir **después** de los obligatorios. TypeScript te lo advertirá
> si los pones antes.

#### Ejemplo aplicado — constructor de mensajes de log

```ts
type Nivel = "info" | "warn" | "error";

function log(
  mensaje: string,
  nivel: Nivel = "info",
  timestamp?: boolean
): string {
  const prefijos: Record<Nivel, string> = {
    info:  "ℹ️  INFO ",
    warn:  "⚠️  WARN ",
    error: "❌ ERROR",
  };

  const hora = timestamp ? ` [${new Date().toISOString()}]` : "";
  return `${prefijos[nivel]}${hora}: ${mensaje}`;
}

console.log(log("Servidor iniciado"));
// ℹ️  INFO : Servidor iniciado

console.log(log("Memoria alta", "warn"));
// ⚠️  WARN : Memoria alta

console.log(log("Conexión perdida", "error", true));
// ❌ ERROR [2026-06-23T...]: Conexión perdida
```

> **Mini-ejercicio (6 min).** Escribe `function formatearPrecio(monto: number,
> moneda: string = "USD", decimales: number = 2): string` que devuelva algo como
> `"USD 1,234.50"`. Pruébala con `(1234.5)`, `(99, "EUR")` y `(50.125, "MXN", 1)`.
> Pista: usa `monto.toFixed(decimales)`.

---

### B.2 · Rest parameters `...nums: number[]`

Un rest parameter recopila **todos los argumentos restantes** en un array.
Permite llamar a una función con un número variable de argumentos sin usar `arguments`.

```ts
// Concepto puro

// El rest parameter SIEMPRE es el último
function sumarTodos(...nums: number[]): number {
  return nums.reduce((acc, n) => acc + n, 0);
}

console.log(sumarTodos(1, 2, 3));          // 6
console.log(sumarTodos(10, 20, 30, 40));   // 100
console.log(sumarTodos());                 // 0

// Combinado con parámetros normales
function construirRuta(base: string, ...segmentos: string[]): string {
  return [base, ...segmentos].join("/");
}

console.log(construirRuta("https://api.ejemplo.com", "v1", "usuarios", "42"));
// https://api.ejemplo.com/v1/usuarios/42
```

#### Ejemplo aplicado — sistema de registro de eventos

```ts
function registrarEvento(tipo: string, ...detalles: string[]): void {
  const timestamp = new Date().toLocaleTimeString();
  const cuerpo = detalles.length > 0 ? ` | ${detalles.join(" · ")}` : "";
  console.log(`[${timestamp}] ${tipo.toUpperCase()}${cuerpo}`);
}

registrarEvento("inicio");
// [10:05:01] INICIO

registrarEvento("login", "usuario: ana", "ip: 192.168.1.10");
// [10:05:02] LOGIN | usuario: ana · ip: 192.168.1.10

registrarEvento("error", "módulo: pagos", "código: 503", "reintento: sí");
// [10:05:03] ERROR | módulo: pagos · código: 503 · reintento: sí
```

> **Mini-ejercicio (5 min).** Escribe `function maximo(primero: number,
> ...resto: number[]): number` que devuelva el número más grande entre todos
> los argumentos. Pruébala con `(3, 1, 4, 1, 5, 9, 2, 6)` → debe dar `9`.
> **Extra:** escribe `minimo` de la misma forma.

---

## Parte C · Tipos de retorno especiales e inferencia

---

### C.1 · `void` vs `never` e inferencia de retorno

- `void`: la función **no devuelve** un valor útil (efectos secundarios como `console.log`).
- `never`: la función **nunca termina** normalmente (lanza excepción o bucle infinito).
- Inferencia: TypeScript deduce el tipo de retorno por el `return`; lo explícito actúa como contrato.

```ts
// Concepto puro

// void — no hay valor de retorno significativo
function imprimirLinea(texto: string): void {
  console.log(texto);
  // No hay return, o hay un "return;" vacío
}

// never — la función nunca retorna
function lanzarError(mensaje: string): never {
  throw new Error(mensaje);
  // TypeScript sabe que el código tras throw es inalcanzable
}

function bucleInfinito(): never {
  while (true) {
    // proceso eterno de un worker, por ejemplo
  }
}

// Inferencia — TypeScript deduce "number"
function multiplicar(a: number, b: number) {
  return a * b; // tipo inferido: number
}

// Pero el retorno explícito actúa de contrato:
function dividir(a: number, b: number): number {
  if (b === 0) lanzarError("División por cero"); // never encaja en cualquier tipo
  return a / b;
}
```

#### Ejemplo aplicado — manejador de errores de API

```ts
type CodigoHTTP = 200 | 400 | 401 | 403 | 404 | 500;

function manejarRespuesta(codigo: CodigoHTTP, datos?: string): void {
  if (codigo === 200) {
    console.log(`Éxito: ${datos ?? "sin datos"}`);
    return; // return vacío en void
  }
  procesarError(codigo); // never — el flujo no sigue
}

function procesarError(codigo: CodigoHTTP): never {
  const mensajes: Partial<Record<CodigoHTTP, string>> = {
    400: "Solicitud inválida",
    401: "No autenticado",
    403: "Sin permisos",
    404: "Recurso no encontrado",
    500: "Error interno del servidor",
  };
  throw new Error(`HTTP ${codigo}: ${mensajes[codigo] ?? "error desconocido"}`);
}

manejarRespuesta(200, "usuario cargado");  // Éxito: usuario cargado
// manejarRespuesta(404);                 // Lanza Error: HTTP 404: Recurso no encontrado
```

> **Mini-ejercicio (5 min).** Escribe `function asegurar(condicion: boolean,
> mensaje: string): void` que llame a una función `fallar(mensaje: string): never`
> si la condición es falsa (usando `throw new Error(mensaje)`). Úsala para
> comprobar que `asegurar(2 + 2 === 4, "Matematicas rotas")` no lanza,
> pero `asegurar(1 === 2, "Uno no es dos")` sí lo hace.

---

## Parte D · Funciones de orden superior

Las funciones de orden superior **reciben funciones como argumento** o
**devuelven funciones**. Son el corazón de la programación funcional en TypeScript.

---

### D.1 · Tipos de función y callbacks tipados

Un tipo de función se escribe `(param: Tipo) => TipoRetorno`. Puedes nombrarlo
con `type` para reutilizarlo.

```ts
// Concepto puro

// Tipo de función nombrado
type Transformador = (x: number) => number;
type Predicado     = (x: number) => boolean;

// Función que RECIBE una función (orden superior)
function aplicar(n: number, fn: Transformador): number {
  return fn(n);
}

// Función que DEVUELVE una función
function multiplicadorDe(factor: number): Transformador {
  return (x) => x * factor;
}

// Uso
const triple = multiplicadorDe(3);
const cuadrado: Transformador = (x) => x * x;

console.log(aplicar(5, triple));    // 15
console.log(aplicar(5, cuadrado)); // 25
console.log(aplicar(5, (x) => x + 10)); // 15 (lambda inline)

// Filtrar con un predicado tipado
function filtrar(nums: number[], condicion: Predicado): number[] {
  return nums.filter(condicion);
}

const nums = [1, 2, 3, 4, 5, 6, 7, 8];
console.log(filtrar(nums, (n) => n % 2 === 0)); // [2, 4, 6, 8]
console.log(filtrar(nums, (n) => n > 5));       // [6, 7, 8]
```

#### Ejemplo aplicado — pipeline de procesamiento de pedidos

```ts
type Pedido = { id: number; total: number; cliente: string };
type ProcesadorPedido = (pedido: Pedido) => Pedido;

// Funciones que transforman un pedido
const aplicarIVA: ProcesadorPedido = (p) => ({
  ...p,
  total: Number((p.total * 1.19).toFixed(2)),
});

const aplicarDescuentoVIP = (descuento: number): ProcesadorPedido =>
  (p) => ({ ...p, total: Number((p.total * (1 - descuento)).toFixed(2)) });

// Pipeline: aplica una lista de procesadores en orden
function procesarPedido(pedido: Pedido, pasos: ProcesadorPedido[]): Pedido {
  return pasos.reduce((p, fn) => fn(p), pedido);
}

const pedido: Pedido = { id: 101, total: 100, cliente: "Ana" };

const resultado = procesarPedido(pedido, [
  aplicarDescuentoVIP(0.10),  // 10% descuento VIP → $90
  aplicarIVA,                 // + 19% IVA         → $107.10
]);

console.log(resultado);
// { id: 101, total: 107.1, cliente: 'Ana' }
```

> **Mini-ejercicio (8 min).** Escribe `function componer<T>(
> ...fns: Array<(x: T) => T>): (x: T) => T` que devuelva una nueva función
> que aplica todas las funciones dadas de **derecha a izquierda** (igual que
> `compose` matemático). Pruébala con tres funciones sobre `string`:
> `trim`, `toLowerCase` y `capitalizar`. El resultado de
> `componer(capitalizar, toLowerCase, trim)("  HOLA MUNDO  ")`
> debe ser `"Hola mundo"`.

---

## Parte E · Sobrecargas de función

Las sobrecargas permiten que **una misma función** acepte distintas combinaciones
de tipos de parámetros, con retornos distintos según la firma usada.
Se declaran varias **firmas de sobrecarga** y luego **una sola implementación**.

---

### E.1 · Function overloads — firmas múltiples, una implementación

```ts
// Concepto puro

// 1. Firmas de sobrecarga (solo tipos, sin cuerpo)
function formatear(valor: number): string;
function formatear(valor: string): string;
function formatear(valor: boolean): string;

// 2. Implementación (el tipo real debe ser compatible con todas las firmas)
function formatear(valor: number | string | boolean): string {
  if (typeof valor === "number") {
    return valor.toLocaleString("es-MX", { minimumFractionDigits: 2 });
  }
  if (typeof valor === "boolean") {
    return valor ? "Sí" : "No";
  }
  return `"${valor}"`;
}

// TypeScript conoce el retorno exacto según la firma usada
console.log(formatear(1234567.5)); // 1,234,567.50
console.log(formatear(true));      // Sí
console.log(formatear("activo"));  // "activo"
```

> ⚠️ **La implementación no es una sobrecarga.** La firma de implementación
> NO es visible para quien llama a la función. Solo las firmas declaradas
> antes del cuerpo son las que se exponen.

#### Ejemplo aplicado — buscador polimórfico

```ts
// Una función que busca por ID (number) o por nombre (string)
type Producto = { id: number; nombre: string; precio: number };

const catalogo: Producto[] = [
  { id: 1, nombre: "Laptop",  precio: 1200 },
  { id: 2, nombre: "Teclado", precio: 80   },
  { id: 3, nombre: "Monitor", precio: 350  },
];

// Sobrecargas
function buscar(id: number): Producto | undefined;
function buscar(nombre: string): Producto[];
// Implementación
function buscar(criterio: number | string): Producto | Producto[] | undefined {
  if (typeof criterio === "number") {
    return catalogo.find((p) => p.id === criterio);
  }
  const termino = criterio.toLowerCase();
  return catalogo.filter((p) => p.nombre.toLowerCase().includes(termino));
}

// TypeScript sabe el retorno exacto por la firma elegida
const porId   = buscar(2);             // Producto | undefined
const porNombre = buscar("o");         // Producto[]

console.log(porId);
// { id: 2, nombre: 'Teclado', precio: 80 }

console.log(porNombre);
// [ { id: 2, nombre: 'Teclado', ... }, { id: 3, nombre: 'Monitor', ... } ]
```

> **Mini-ejercicio (8 min).** Escribe una función sobrecargada `convertir` con
> dos firmas: `convertir(valor: number, a: "binario" | "hex"): string` y
> `convertir(valor: string, desde: "binario" | "hex"): number`.
> La implementación convierte un número a cadena en la base indicada
> (`n.toString(2)`, `n.toString(16)`) o parsea una cadena a entero
> (`parseInt(s, 2)`, `parseInt(s, 16)`).
> Prueba: `convertir(255, "hex")` → `"ff"`, `convertir("ff", "hex")` → `255`.

---

## Parte F · `this` y closures

---

### F.1 · Closures — funciones que recuerdan su entorno

Un closure es una función que **captura variables** del ámbito donde fue creada,
incluso cuando ese ámbito ya no está activo.

```ts
// Concepto puro — fábrica de contadores independientes
function crearContador(inicio: number = 0) {
  let cuenta = inicio; // variable capturada

  return {
    incrementar: () => ++cuenta,
    decrementar: () => --cuenta,
    valor:       () => cuenta,
    resetear:    () => { cuenta = inicio; },
  };
}

const c1 = crearContador();
const c2 = crearContador(10); // contador independiente

c1.incrementar();
c1.incrementar();
c1.incrementar();
c2.incrementar();

console.log(c1.valor()); // 3   (c1 y c2 tienen su propia "cuenta")
console.log(c2.valor()); // 11

c1.resetear();
console.log(c1.valor()); // 0
```

#### Ejemplo aplicado — caché con memoización

```ts
// Memoize: guarda resultados de llamadas previas en un closure
function memoize<T, R>(fn: (arg: T) => R): (arg: T) => R {
  const cache = new Map<T, R>(); // capturada por el closure

  return (arg: T): R => {
    if (cache.has(arg)) {
      console.log(`Cache hit: ${arg}`);
      return cache.get(arg)!;
    }
    const resultado = fn(arg);
    cache.set(arg, resultado);
    return resultado;
  };
}

// Fibonacci costoso
function fibLento(n: number): number {
  if (n <= 1) return n;
  return fibLento(n - 1) + fibLento(n - 2);
}

const fibRapido = memoize(fibLento);

console.log(fibRapido(10)); // calcula
console.log(fibRapido(10)); // Cache hit: 10 — retorna al instante
console.log(fibRapido(20)); // calcula
```

> **Mini-ejercicio (7 min).** Usa `crearContador` (del concepto puro de arriba)
> para modelar dos semáforos independientes: `semR` (rojo) y `semV` (verde).
> Haz que `semR` cuente 3 veces y `semV` cuente 1 vez. Verifica que los valores
> son independientes. **Extra:** agrega a la fábrica un método
> `historial: () => number[]` que devuelva todos los valores que tomó el contador.

---

## Ejemplo combinado

Une todos los conceptos de la página en un sistema real de procesamiento de tareas:

```ts
// ─── Tipos y sobrecargas ───────────────────────────────────────────────────
type Prioridad = "alta" | "media" | "baja";
type Tarea = { id: number; titulo: string; prioridad: Prioridad; horas: number };

// Sobrecarga: buscar por id (number) o por prioridad (Prioridad)
function buscarTarea(id: number): Tarea | undefined;
function buscarTarea(prioridad: Prioridad): Tarea[];
function buscarTarea(criterio: number | Prioridad): Tarea | Tarea[] | undefined {
  if (typeof criterio === "number") {
    return tareas.find((t) => t.id === criterio);
  }
  return tareas.filter((t) => t.prioridad === criterio);
}

// ─── Rest + valor por defecto ──────────────────────────────────────────────
function crearTareas(prioridad: Prioridad = "media", ...titulos: string[]): Tarea[] {
  return titulos.map((titulo, i) => ({
    id: Date.now() + i,
    titulo,
    prioridad,
    horas: 0,
  }));
}

// ─── Orden superior: tipo de función ──────────────────────────────────────
type AnalizadorTarea = (t: Tarea) => string;

const formatearTarea: AnalizadorTarea = (t) =>
  `[${t.prioridad.toUpperCase().padEnd(5)}] #${t.id} "${t.titulo}" (${t.horas}h)`;

function generarReporte(lista: Tarea[], analizar: AnalizadorTarea): void {
  console.log("=== Reporte de Tareas ===");
  lista.forEach((t) => console.log(analizar(t)));
  const totalHoras = lista.reduce((acc, t) => acc + t.horas, 0);
  console.log(`Total: ${lista.length} tareas · ${totalHoras}h estimadas`);
}

// ─── Closure: acumulador de horas ─────────────────────────────────────────
function crearAcumulador() {
  let total = 0;
  return {
    agregar: (horas: number): void => { total += horas; },
    obtener: (): number => total,
  };
}

// ─── void y never ─────────────────────────────────────────────────────────
function validarTarea(t: Tarea): void {
  if (t.titulo.trim() === "") lanzarValidacion("El título no puede estar vacío");
  if (t.horas < 0)            lanzarValidacion("Las horas no pueden ser negativas");
}

function lanzarValidacion(msg: string): never {
  throw new Error(`Validación fallida: ${msg}`);
}

// ─── Ejecución ────────────────────────────────────────────────────────────
const tareas: Tarea[] = [
  { id: 1, titulo: "Diseñar API",      prioridad: "alta",  horas: 8 },
  { id: 2, titulo: "Escribir pruebas", prioridad: "media", horas: 4 },
  { id: 3, titulo: "Actualizar docs",  prioridad: "baja",  horas: 2 },
  { id: 4, titulo: "Code review",      prioridad: "alta",  horas: 3 },
];

const acum = crearAcumulador();
tareas.forEach((t) => {
  validarTarea(t);
  acum.agregar(t.horas);
});

generarReporte(tareas, formatearTarea);
console.log(`Horas totales (closure): ${acum.obtener()}h`);

// Sobrecargas en acción
const alta = buscarTarea("alta");   // Tarea[]
const t1   = buscarTarea(1);        // Tarea | undefined

console.log("Tareas de alta prioridad:", alta.map((t) => t.titulo));
console.log("Tarea #1:", t1?.titulo);
```

Salida esperada:
```
=== Reporte de Tareas ===
[ALTA ] #1 "Diseñar API" (8h)
[MEDIA] #2 "Escribir pruebas" (4h)
[BAJA ] #3 "Actualizar docs" (2h)
[ALTA ] #4 "Code review" (3h)
Total: 4 tareas · 17h estimadas
Horas totales (closure): 17h
Tareas de alta prioridad: [ 'Diseñar API', 'Code review' ]
Tarea #1: Diseñar API
```

---

## Resumen de la página 4

**Declaración y tipado básico:**
- Añade `: Tipo` a cada parámetro y `: TipoRetorno` tras el paréntesis de cierre para que TypeScript verifique contratos en tiempo de compilación.
- El tipo de retorno explícito protege la definición; la inferencia es cómoda pero menos descriptiva.

**Funciones flecha:**
- Sintaxis `(param: Tipo): RetType => expresión` permite retorno implícito en una línea.
- Prefiere declaración tradicional para funciones de módulo; usa flechas para callbacks y para preservar el `this` léxico.

**Parámetros opcionales, por defecto y rest:**
- `param?` → tipo `T | undefined`; siempre después de los obligatorios.
- `param = valor` → garantiza un valor, nunca `undefined`, y simplifica el cuerpo.
- `...rest: T[]` → agrupa argumentos extra en un array; debe ser el último parámetro.

**Tipos de retorno especiales:**
- `void` → función sin valor de retorno útil (efectos secundarios).
- `never` → función que lanza o nunca termina; encaja en cualquier tipo de retorno.

**Funciones de orden superior:**
- Escribe el tipo de una función como `(x: T) => R` o nómbralo con `type`.
- Las funciones que devuelven funciones (fábricas) y los callbacks tipados son el patrón central de la programación funcional en TypeScript.

**Sobrecargas:**
- Declara varias firmas (sin cuerpo) seguidas de una implementación cuya firma interna cubre todos los casos.
- La implementación no es visible para quien llama; solo las firmas de sobrecarga lo son.

**Closures:**
- Una función captura variables del ámbito donde fue creada; cada llamada a la fábrica genera un estado privado independiente.
- Patrones clásicos: contadores, memoización, acumuladores.

---

> **Siguiente página →** Página 5: Type Aliases e Interfaces — `type` vs
> `interface`, extends, intersecciones y cuándo usar cada uno.
