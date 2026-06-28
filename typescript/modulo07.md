# Tutorial TypeScript — Página 7
## Genéricos
### Código reutilizable y con tipos seguros: `<T>`, restricciones y utility types

---

> **Cómo usar esta página.** Cada bloque sigue el mismo ritmo:
> **1)** el concepto puro, **2)** un ejemplo aplicado real, **3)** un
> **mini-ejercicio** que se resuelve en clase en **5–8 minutos**.
> Escribe el código en el [TypeScript Playground](https://www.typescriptlang.org/play)
> o ejecútalo con `npx tsx archivo.ts`.

---

## Parte A · El problema que resuelven los genéricos

Sin genéricos, el código que reutiliza lógica tiene dos salidas malas:
**duplicar funciones** para cada tipo, o usar `any` y perder toda seguridad de tipos.
Los genéricos son la tercera salida: una sola función que acepta cualquier tipo
y **recuerda cuál usaste**.

---

### A.1 · Motivación — duplicar código vs. `any` vs. genérico

```ts
// ❌ Opción 1: duplicar la misma lógica para cada tipo
function primerNumero(arr: number[]): number {
  return arr[0];
}
function primerTexto(arr: string[]): string {
  return arr[0];
}

// ❌ Opción 2: usar "any" — TypeScript ya no puede ayudarte
function primeroAny(arr: any[]): any {
  return arr[0];    // el valor de retorno es "any": pierdes autocompletado y errores
}

const resultado = primeroAny([1, 2, 3]);
resultado.toUpperCase(); // no error en compilación, pero falla en runtime ❌

// ✅ Opción 3: genérico — una sola función, tipo preservado
function primero<T>(arr: T[]): T {
  return arr[0];
}

const n = primero([10, 20, 30]);   // TypeScript infiere T = number
const s = primero(["a", "b"]);    // TypeScript infiere T = string
// n.toUpperCase();  // ← Error de compilación: number no tiene toUpperCase ✅
```

La letra `T` es solo un nombre convencional (de *Type*). Puedes usar cualquier
identificador válido; `T`, `U`, `V`, `Item`, `Valor` son nombres comunes.

#### Ejemplo aplicado — función `ultimo` genérica

```ts
function ultimo<T>(arr: T[]): T | undefined {
  return arr.length > 0 ? arr[arr.length - 1] : undefined;
}

console.log(ultimo([1, 2, 3]));          // 3  (tipo: number | undefined)
console.log(ultimo(["x", "y", "z"]));   // "z" (tipo: string | undefined)
console.log(ultimo([]));                 // undefined

// La función trabaja con booleanos, objetos o cualquier tipo sin cambios
console.log(ultimo([true, false, true])); // true
```

> **Mini-ejercicio (5 min).** Escribe una función genérica `envolverArray<T>(valor: T): T[]`
> que reciba un valor y lo devuelva dentro de un array de un solo elemento.
> Pruébala con un número, un string y un objeto `{ id: 1 }`.
> Verifica que TypeScript infiere correctamente el tipo del array resultante.

---

## Parte B · Función genérica y múltiples parámetros de tipo

### B.1 · Función genérica — inferencia del tipo al llamar

TypeScript infiere el tipo de `T` a partir de los argumentos que pasas.
No siempre tienes que escribirlo explícitamente, aunque puedes hacerlo.

```ts
// TypeScript infiere T por los argumentos
function identidad<T>(valor: T): T {
  return valor;
}

// Inferencia automática — no hay que escribir <number>
const a = identidad(42);        // T = number
const b = identidad("hola");   // T = string
const c = identidad(true);     // T = boolean

// Anotación explícita — útil cuando la inferencia no es obvia
const d = identidad<number[]>([1, 2, 3]);

// Función flecha genérica (sintaxis con coma en .tsx para no confundir con JSX)
const copiar = <T,>(arr: T[]): T[] => [...arr];
```

#### Ejemplo aplicado — función `repetir` genérica

```ts
function repetir<T>(valor: T, veces: number): T[] {
  return Array.from({ length: veces }, () => valor);
}

console.log(repetir("eco", 3));         // ["eco", "eco", "eco"]
console.log(repetir(0, 5));             // [0, 0, 0, 0, 0]
console.log(repetir({ ok: true }, 2));  // [{ ok: true }, { ok: true }]
```

> **Mini-ejercicio (6 min).** Escribe `function intercambiar<T>(arr: T[], i: number, j: number): T[]`
> que devuelva una copia del array con los elementos en posiciones `i` y `j` intercambiados.
> Pruébala con `intercambiar([1, 2, 3, 4], 0, 3)` (debe dar `[4, 2, 3, 1]`).

---

### B.2 · Múltiples parámetros de tipo `<K, V>`

Cuando la función relaciona **dos tipos distintos**, declaras dos parámetros.

```ts
// Concepto puro — par de valores de tipos distintos
function crearPar<K, V>(clave: K, valor: V): [K, V] {
  return [clave, valor];
}

const par1 = crearPar("edad", 30);          // [string, number]
const par2 = crearPar(1, true);             // [number, boolean]
const par3 = crearPar("activo", { ok: 1 }); // [string, { ok: number }]

// Función que construye un objeto a partir de dos arrays (zip)
function zipear<K extends string, V>(claves: K[], valores: V[]): Record<K, V> {
  const resultado = {} as Record<K, V>;
  claves.forEach((k, i) => {
    resultado[k] = valores[i];
  });
  return resultado;
}

const obj = zipear(["a", "b", "c"], [1, 2, 3]);
console.log(obj); // { a: 1, b: 2, c: 3 }
```

#### Ejemplo aplicado — mapa de configuraciones

```ts
function crearMapa<K extends string, V>(
  entradas: Array<[K, V]>
): Map<K, V> {
  return new Map(entradas);
}

const roles = crearMapa<string, string[]>([
  ["admin",   ["leer", "escribir", "borrar"]],
  ["editor",  ["leer", "escribir"]],
  ["lector",  ["leer"]],
]);

console.log(roles.get("editor")); // ["leer", "escribir"]
```

> **Mini-ejercicio (7 min).** Escribe `function transformarMapa<K extends string, V, W>(
> mapa: Record<K, V>, fn: (v: V) => W): Record<K, W>` que aplique `fn` a cada valor
> del objeto. Pruébala convirtiendo `{ a: 1, b: 2, c: 3 }` a `{ a: "1€", b: "2€", c: "3€" }`
> usando `fn = (n) => n + "€"`.

---

## Parte C · Restricciones con `extends`

### C.1 · Constraints — `<T extends { id: number }>`

A veces no quieres aceptar *cualquier* tipo, sino solo los que tengan
ciertas propiedades. Las restricciones con `extends` lo garantizan en tiempo de compilación.

```ts
// Concepto puro
// Sin restricción: T puede ser cualquier cosa
// Con restricción: T DEBE tener al menos la propiedad "id: number"
function buscarPorId<T extends { id: number }>(lista: T[], id: number): T | undefined {
  return lista.find((item) => item.id === id);
}

// ✅ Funciona: el tipo tiene "id: number"
const usuarios = [
  { id: 1, nombre: "Ana" },
  { id: 2, nombre: "Luis" },
];
console.log(buscarPorId(usuarios, 2)); // { id: 2, nombre: "Luis" }

// ✅ También funciona con tipos que tengan más campos
const productos = [
  { id: 10, nombre: "Mouse", precio: 25 },
  { id: 11, nombre: "Teclado", precio: 80 },
];
console.log(buscarPorId(productos, 11)); // { id: 11, nombre: "Teclado", precio: 80 }

// ❌ Error de compilación: string[] no tiene propiedad "id"
// buscarPorId(["a", "b"], 0);
```

> ⚠️ `extends` en genéricos no significa herencia de clases. Significa
> "el tipo pasado debe ser **compatible con** (o más específico que) este tipo".
> `T extends { id: number }` acepta cualquier objeto que tenga al menos `id: number`.

#### Ejemplo aplicado — ordenar por cualquier campo numérico

```ts
function ordenarPor<T extends Record<string, unknown>>(
  lista: T[],
  campo: keyof T
): T[] {
  return [...lista].sort((a, b) => {
    const va = a[campo] as number;
    const vb = b[campo] as number;
    return va - vb;
  });
}

const empleados = [
  { nombre: "Carlos", salario: 3000, antiguedad: 5 },
  { nombre: "Marta",  salario: 4500, antiguedad: 2 },
  { nombre: "Pedro",  salario: 2800, antiguedad: 8 },
];

console.log(ordenarPor(empleados, "salario").map((e) => e.nombre));
// ["Pedro", "Carlos", "Marta"]
```

> **Mini-ejercicio (7 min).** Escribe `function fusionar<T extends object>(base: T, extra: Partial<T>): T`
> que devuelva un nuevo objeto con las propiedades de `base` sobreescritas por las de `extra`.
> Pruébala con `base = { nombre: "Ana", edad: 30, activa: true }` y `extra = { edad: 31 }`.
> Resultado esperado: `{ nombre: "Ana", edad: 31, activa: true }`.

---

### C.2 · `keyof` y acceso por clave genérico

`keyof T` produce un tipo unión con todos los nombres de propiedades de `T`.
Combinado con genéricos, garantiza que solo puedas acceder a claves que realmente existen.

```ts
// Concepto puro
function prop<T, K extends keyof T>(obj: T, k: K): T[K] {
  return obj[k];
}

const persona = { nombre: "Ana", edad: 30, activa: true };

const nombre = prop(persona, "nombre");  // tipo: string
const edad   = prop(persona, "edad");    // tipo: number
const activa = prop(persona, "activa"); // tipo: boolean

// ❌ Error de compilación: "telefono" no existe en el tipo
// prop(persona, "telefono");

// Todos los nombres de clave de persona como tipo unión
type ClavesPersona = keyof typeof persona;  // "nombre" | "edad" | "activa"
```

#### Ejemplo aplicado — extractor de columnas (tipo pick manual)

```ts
function seleccionarColumnas<T, K extends keyof T>(
  lista: T[],
  ...campos: K[]
): Pick<T, K>[] {
  return lista.map((item) => {
    const resultado = {} as Pick<T, K>;
    campos.forEach((campo) => {
      resultado[campo] = item[campo];
    });
    return resultado;
  });
}

const inventario = [
  { id: 1, nombre: "Monitor",  precio: 300, stock: 12 },
  { id: 2, nombre: "Teclado",  precio: 80,  stock: 50 },
  { id: 3, nombre: "Webcam",   precio: 120, stock: 8  },
];

const resumen = seleccionarColumnas(inventario, "nombre", "precio");
console.log(resumen);
// [{ nombre: "Monitor", precio: 300 }, { nombre: "Teclado", precio: 80 }, ...]
```

> **Mini-ejercicio (8 min).** Escribe `function agruparPor<T, K extends keyof T>(
> lista: T[], campo: K): Map<T[K], T[]>` que agrupe los elementos de un array
> por el valor del campo indicado.
> Pruébala con una lista de `{ ciudad: string; nombre: string }` y `campo = "ciudad"`.

---

## Parte D · Interfaces y clases genéricas

### D.1 · Interface genérica `Caja<T>`

Las interfaces también aceptan parámetros de tipo. Son ideales para modelar
contenedores, resultados de API o estructuras que envuelven datos de tipo variable.

```ts
// Concepto puro
interface Caja<T> {
  valor: T;
  etiqueta: string;
  creada: Date;
}

const cajaNum: Caja<number> = {
  valor: 42,
  etiqueta: "respuesta",
  creada: new Date(),
};

const cajaStr: Caja<string> = {
  valor: "hola mundo",
  etiqueta: "mensaje",
  creada: new Date(),
};

// Interface genérica para respuestas de API
interface RespuestaApi<T> {
  datos: T;
  exito: boolean;
  mensaje: string;
  total?: number;
}

interface Usuario { id: number; nombre: string; }

const respuesta: RespuestaApi<Usuario[]> = {
  datos: [{ id: 1, nombre: "Ana" }, { id: 2, nombre: "Luis" }],
  exito: true,
  mensaje: "Usuarios cargados",
  total: 2,
};
```

#### Ejemplo aplicado — resultado con error tipado

```ts
interface Resultado<T, E = string> {
  ok: boolean;
  valor?: T;
  error?: E;
}

function dividir(a: number, b: number): Resultado<number> {
  if (b === 0) return { ok: false, error: "División por cero" };
  return { ok: true, valor: a / b };
}

const r1 = dividir(10, 2);
const r2 = dividir(5, 0);

if (r1.ok) console.log(`Resultado: ${r1.valor}`); // Resultado: 5
if (!r2.ok) console.log(`Error: ${r2.error}`);    // Error: División por cero
```

> **Mini-ejercicio (6 min).** Define `interface Paginado<T>` con campos
> `items: T[]`, `pagina: number`, `porPagina: number` y `total: number`.
> Escribe `function paginar<T>(lista: T[], pagina: number, porPagina: number): Paginado<T>`
> que devuelva el subconjunto correcto. Pruébala con un array de 10 números y `porPagina = 3`.

---

### D.2 · Clase genérica `Pila<T>` — stack con push / pop

```ts
// Concepto puro
class Pila<T> {
  private elementos: T[] = [];

  push(item: T): void {
    this.elementos.push(item);
  }

  pop(): T | undefined {
    return this.elementos.pop();
  }

  peek(): T | undefined {
    return this.elementos[this.elementos.length - 1];
  }

  get tamano(): number {
    return this.elementos.length;
  }

  estaVacia(): boolean {
    return this.elementos.length === 0;
  }
}

// Pila de números
const pilaNum = new Pila<number>();
pilaNum.push(1);
pilaNum.push(2);
pilaNum.push(3);
console.log(pilaNum.peek());   // 3
console.log(pilaNum.pop());    // 3
console.log(pilaNum.tamano);   // 2

// Pila de strings — la misma clase, otro tipo
const pilaStr = new Pila<string>();
pilaStr.push("primero");
pilaStr.push("segundo");
console.log(pilaStr.pop());    // "segundo"
```

#### Ejemplo aplicado — historial de navegación

```ts
interface Pagina {
  url: string;
  titulo: string;
  visitada: Date;
}

class HistorialNavegacion {
  private pila = new Pila<Pagina>();

  visitar(url: string, titulo: string): void {
    this.pila.push({ url, titulo, visitada: new Date() });
    console.log(`Visitando: ${titulo} (${url})`);
  }

  atras(): Pagina | undefined {
    const actual = this.pila.pop();
    if (actual) console.log(`Volviendo desde: ${actual.titulo}`);
    return this.pila.peek();
  }

  paginaActual(): Pagina | undefined {
    return this.pila.peek();
  }
}

const historial = new HistorialNavegacion();
historial.visitar("/",        "Inicio");
historial.visitar("/docs",    "Documentación");
historial.visitar("/docs/ts", "TypeScript");

historial.atras(); // Volviendo desde: TypeScript
console.log(historial.paginaActual()?.titulo); // Documentación
```

> **Mini-ejercicio (8 min).** Crea una clase genérica `Cola<T>` (queue FIFO)
> con métodos `encolar(item: T): void`, `desencolar(): T | undefined` y `frente(): T | undefined`.
> Una `Cola<string>` gestiona mensajes de chat: encola tres mensajes y desencólalos
> en orden FIFO (el primero en entrar es el primero en salir).

---

## Parte E · Valores por defecto y utility types

### E.1 · Valor por defecto de tipo `<T = string>`

Cuando tienes un tipo genérico que, en la mayoría de casos, siempre usas con
el mismo tipo, puedes declarar un **valor por defecto** para evitar repetición.

```ts
// Concepto puro
interface Respuesta<T = string> {
  datos: T;
  codigo: number;
}

// Sin anotar el tipo: T = string por defecto
const r1: Respuesta = { datos: "Ok", codigo: 200 };

// Anotando explícitamente para sobreescribir el default
const r2: Respuesta<number[]> = { datos: [1, 2, 3], codigo: 200 };

// En funciones genéricas también funciona
function crearContenedor<T = boolean>(valor: T, etiqueta: string) {
  return { valor, etiqueta };
}

const c1 = crearContenedor(true, "flag");           // T = boolean (inferido)
const c2 = crearContenedor<number>(99, "contador"); // T = number (explícito)
```

> **Truco.** El valor por defecto solo aplica cuando TypeScript **no puede inferir**
> el tipo. Si pasas un argumento, la inferencia gana sobre el default.

#### Ejemplo aplicado — generador de eventos con payload opcional

```ts
interface Evento<Payload = null> {
  tipo: string;
  payload: Payload;
  timestamp: number;
}

function crearEvento<P = null>(tipo: string, payload: P): Evento<P> {
  return { tipo, payload, timestamp: Date.now() };
}

// Sin payload: usa null por defecto
const evLogin  = crearEvento("login", null);

// Con payload tipado
const evCompra = crearEvento("compra", { productoId: 5, monto: 150 });
console.log(evCompra.payload.monto); // 150 — TypeScript conoce el tipo
```

> **Mini-ejercicio (5 min).** Define `interface Cache<V = string>` con campos
> `clave: string`, `valor: V` y `expiraEn: number` (timestamp Unix).
> Crea instancias con `V = string`, `V = number` y `V = boolean[]`
> para demostrar que el default se aplica solo cuando no se indica el tipo.

---

### E.2 · Utility types comunes

TypeScript incluye tipos genéricos de utilidad en su librería estándar.
Son genéricos predefinidos que transforman otros tipos. Los más usados:

---

#### `Partial<T>` — hace todas las propiedades opcionales

```ts
interface Config {
  host: string;
  puerto: number;
  debug: boolean;
  timeout: number;
}

// Partial<Config> = { host?: string; puerto?: number; debug?: boolean; timeout?: number }
function actualizarConfig(base: Config, cambios: Partial<Config>): Config {
  return { ...base, ...cambios };
}

const cfg: Config = { host: "localhost", puerto: 8080, debug: false, timeout: 3000 };
const nueva = actualizarConfig(cfg, { debug: true, puerto: 9090 });
console.log(nueva);
// { host: "localhost", puerto: 9090, debug: true, timeout: 3000 }
```

---

#### `Required<T>` — hace todas las propiedades obligatorias

```ts
interface OpcionesBusqueda {
  query?: string;
  pagina?: number;
  limite?: number;
}

// Required<OpcionesBusqueda> = { query: string; pagina: number; limite: number }
function buscar(opciones: Required<OpcionesBusqueda>): void {
  console.log(`Buscando "${opciones.query}" — p.${opciones.pagina}, lím.${opciones.limite}`);
}

buscar({ query: "typescript", pagina: 1, limite: 10 });
// buscar({ query: "ts" }); // ❌ Error: faltan pagina y limite
```

---

#### `Pick<T, K>` — selecciona un subconjunto de propiedades

```ts
interface Producto {
  id: number;
  nombre: string;
  precio: number;
  stock: number;
  proveedor: string;
}

// Solo las propiedades necesarias para mostrar en un listado
type ProductoResumen = Pick<Producto, "id" | "nombre" | "precio">;

const resumen: ProductoResumen = { id: 1, nombre: "Mouse", precio: 25 };
// resumen.stock;  // ❌ Error: 'stock' no existe en ProductoResumen
```

---

#### `Omit<T, K>` — excluye propiedades (inverso de Pick)

```ts
// Tipo para crear un producto (sin id, que lo genera la base de datos)
type ProductoNuevo = Omit<Producto, "id">;

const nuevo: ProductoNuevo = {
  nombre: "Webcam",
  precio: 120,
  stock: 15,
  proveedor: "TechCorp",
};

// También puedes omitir varias con unión
type ProductoPublico = Omit<Producto, "stock" | "proveedor">;
```

---

#### `Record<K, V>` — objeto con claves de tipo K y valores de tipo V

```ts
type RolUsuario = "admin" | "editor" | "lector";

// Record<RolUsuario, string[]> garantiza que existan exactamente esas tres claves
const permisos: Record<RolUsuario, string[]> = {
  admin:   ["leer", "escribir", "borrar", "gestionar"],
  editor:  ["leer", "escribir"],
  lector:  ["leer"],
};

// Record con clave string genérica (útil para diccionarios dinámicos)
const etiquetas: Record<string, number> = {};
etiquetas["typescript"] = 42;
etiquetas["genéricos"]  = 17;
```

---

#### `Readonly<T>` — impide modificar las propiedades

```ts
interface Punto {
  x: number;
  y: number;
}

const origen: Readonly<Punto> = { x: 0, y: 0 };
// origen.x = 5;  // ❌ Error de compilación: no se puede asignar a propiedad de solo lectura

// Muy útil para constantes de configuración o valores que no deben mutar
const CONFIG: Readonly<Config> = {
  host: "api.ejemplo.com",
  puerto: 443,
  debug: false,
  timeout: 5000,
};
```

> **Mini-ejercicio (8 min).** Tienes `interface Empleado { id: number; nombre: string;
> departamento: string; salario: number; fechaIngreso: string }`.
> 1. Crea `EmpleadoBorrador = Partial<Empleado>` y úsalo para ir construyendo un empleado campo a campo.
> 2. Crea `FichaPublica = Omit<Empleado, "salario">` y muestra que no puedes acceder al salario.
> 3. Crea `DirectorioDepto = Record<string, Pick<Empleado, "nombre" | "departamento">[]>`
>    y llénalo con datos de al menos dos departamentos.

---

## Ejemplo combinado

Une genéricos, restricciones, `keyof`, clases genéricas y utility types
en un sistema de caché genérico con expiración y soporte de consultas tipadas:

```ts
// --- Tipos e interfaces ---

interface EntradaCache<T> {
  valor: T;
  expiraEn: number;  // timestamp Unix en ms
}

interface Entidad {
  id: number;
}

// --- Clase genérica: Cache con restricción y acceso tipado ---

class CacheConExpiracion<T extends Entidad> {
  private almacen = new Map<number, EntradaCache<T>>();

  guardar(item: T, ttlMs: number): void {
    this.almacen.set(item.id, {
      valor: item,
      expiraEn: Date.now() + ttlMs,
    });
  }

  obtener(id: number): T | undefined {
    const entrada = this.almacen.get(id);
    if (!entrada) return undefined;
    if (Date.now() > entrada.expiraEn) {
      this.almacen.delete(id);  // evict expirado
      return undefined;
    }
    return entrada.valor;
  }

  // keyof: devuelve el valor de un campo específico del item cacheado
  campo<K extends keyof T>(id: number, clave: K): T[K] | undefined {
    return this.obtener(id)?.[clave];
  }

  // Utility: resumen de todos los items válidos (sin los expirados)
  resumenActivo(): Pick<T, "id">[] {
    const resultado: Pick<T, "id">[] = [];
    for (const [id, entrada] of this.almacen) {
      if (Date.now() <= entrada.expiraEn) {
        resultado.push({ id } as Pick<T, "id">);
      }
    }
    return resultado;
  }
}

// --- Uso con tipos concretos ---

interface Sesion extends Entidad {
  usuario: string;
  rol: "admin" | "editor" | "lector";
  ip: string;
}

const cacheSesiones = new CacheConExpiracion<Sesion>();

cacheSesiones.guardar({ id: 1, usuario: "ana", rol: "admin", ip: "10.0.0.1" }, 60_000);
cacheSesiones.guardar({ id: 2, usuario: "luis", rol: "editor", ip: "10.0.0.2" }, 60_000);

console.log(cacheSesiones.obtener(1)?.usuario);   // "ana"
console.log(cacheSesiones.campo(2, "rol"));        // "editor"
// cacheSesiones.campo(2, "clave_inexistente");     // ❌ Error de compilación

// Partial para actualizar solo algunos campos
type ActualizacionSesion = Partial<Omit<Sesion, "id">>;
function aplicarCambios(base: Sesion, cambios: ActualizacionSesion): Sesion {
  return { ...base, ...cambios };
}

const sesionActualizada = aplicarCambios(
  cacheSesiones.obtener(1)!,
  { rol: "lector", ip: "10.0.0.99" }
);
console.log(sesionActualizada);
// { id: 1, usuario: "ana", rol: "lector", ip: "10.0.0.99" }

console.log("Sesiones activas:", cacheSesiones.resumenActivo());
// [{ id: 1 }, { id: 2 }]
```

---

## Resumen de la página 7

**El problema que resuelven los genéricos:**
- Evitan duplicar código para cada tipo y eliminan la necesidad de `any`.
- Un tipo genérico `<T>` es un "marcador de posición" que TypeScript reemplaza con el tipo real al usarlo.

**Función y clase genérica:**
- `function primero<T>(arr: T[]): T` — TypeScript infiere `T` de los argumentos.
- Múltiples parámetros `<K, V>` cuando la función relaciona dos tipos distintos.
- Las clases genéricas (`class Pila<T>`) crean estructuras de datos reutilizables y con tipos seguros.

**Restricciones y acceso tipado:**
- `<T extends { id: number }>` limita `T` a tipos que tengan al menos esa forma.
- `keyof T` produce la unión de nombres de propiedad; `T[K]` accede al tipo del valor.
- `function prop<T, K extends keyof T>(obj: T, k: K): T[K]` es el patrón canónico de acceso seguro.

**Valores por defecto:**
- `<T = string>` define el tipo que se usará cuando no se anote ni se pueda inferir.

**Utility types predefinidos (los más usados):**
- `Partial<T>` — todas las propiedades opcionales.
- `Required<T>` — todas las propiedades obligatorias.
- `Pick<T, K>` — subconjunto de propiedades.
- `Omit<T, K>` — excluye propiedades (inverso de Pick).
- `Record<K, V>` — objeto tipado con claves `K` y valores `V`.
- `Readonly<T>` — impide la mutación de propiedades.

---

> **Fin del curso 🎉** Repasa las 7 páginas y construye un proyecto pequeño
> aplicando tipos, funciones, interfaces, clases y genéricos.
