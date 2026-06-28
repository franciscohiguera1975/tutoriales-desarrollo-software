# Tutorial TypeScript — Página 5
## Type Aliases e Interfaces
### Modelar la forma de los datos: `type`, `interface` y composición

---

> **Cómo usar esta página.** Cada bloque sigue el mismo ritmo:
> **1)** el concepto puro, **2)** un ejemplo aplicado real, **3)** un
> **mini-ejercicio** que se resuelve en clase en **5–8 minutos**.
> Escribe el código en el [TypeScript Playground](https://www.typescriptlang.org/play)
> o ejecútalo con `npx tsx archivo.ts`.

---

## Parte A · `type` alias — ponerle nombre a cualquier tipo

---

### A.1 · `type` alias básico — nombrar objetos, uniones y primitivos

Un `type` alias crea un **nombre reutilizable** para cualquier tipo.
Puede nombrar un objeto, una unión, un primitivo o cualquier combinación.

```ts
// Concepto puro
type ID = string | number;           // unión de primitivos
type Nombre = string;                // alias de primitivo (documenta intención)
type Coordenadas = [number, number]; // alias de tupla

// Alias de objeto
type Punto = {
  x: number;
  y: number;
};

const origen: Punto = { x: 0, y: 0 };
const id: ID = 42;          // válido
const id2: ID = "usr-001";  // también válido
```

> ⚠️ Un `type` **no crea** ningún valor en tiempo de ejecución: desaparece
> después de compilar. Solo existe para el compilador.

#### Ejemplo aplicado — sistema de tickets de soporte

```ts
type TicketID = string | number;
type Prioridad = "baja" | "media" | "alta" | "critica";

type Ticket = {
  id: TicketID;
  titulo: string;
  prioridad: Prioridad;
  resuelta: boolean;
};

function imprimirTicket(t: Ticket): void {
  const estrella = t.prioridad === "critica" ? " ⚠️" : "";
  console.log(`[${t.id}] ${t.titulo} — ${t.prioridad}${estrella}`);
}

const t1: Ticket = { id: "T-001", titulo: "Error de login",  prioridad: "critica", resuelta: false };
const t2: Ticket = { id: 42,      titulo: "Ajuste de fuente", prioridad: "baja",    resuelta: true  };

imprimirTicket(t1); // [T-001] Error de login — critica ⚠️
imprimirTicket(t2); // [42] Ajuste de fuente — baja
```

> **Mini-ejercicio (5 min).** Define un `type` llamado `Moneda` que sea
> la unión `"USD" | "EUR" | "MXN"`. Luego define `type Precio = { monto: number; moneda: Moneda }`.
> Crea tres objetos `Precio` con monedas distintas y muéstralos por consola.

---

## Parte B · `interface` — la forma de los objetos

---

### B.1 · Propiedades básicas, opcionales `?` y `readonly`

Una `interface` describe la **forma exacta** que debe tener un objeto.
Puede tener propiedades obligatorias, opcionales (`?`) e inmutables (`readonly`).

```ts
// Concepto puro
interface Usuario {
  readonly id: number;      // no se puede cambiar después de crear el objeto
  nombre: string;           // obligatoria
  email: string;            // obligatoria
  avatar?: string;          // opcional: puede estar o no
}

const u: Usuario = { id: 1, nombre: "Ana", email: "ana@mail.com" };

// u.id = 99; // ERROR: no se puede asignar a 'id' porque es de solo lectura

// La propiedad opcional puede omitirse sin error:
const u2: Usuario = { id: 2, nombre: "Luis", email: "luis@mail.com", avatar: "avatar.png" };
```

> **Truco.** Usa `readonly` en `id` y fechas de creación para que nadie
> modifique datos que deben ser inmutables una vez asignados.

#### Ejemplo aplicado — perfil de producto en e-commerce

```ts
interface Producto {
  readonly sku: string;
  nombre: string;
  precio: number;
  descripcion?: string;   // texto largo, no siempre presente
  enStock: boolean;
}

function mostrarProducto(p: Producto): void {
  const desc = p.descripcion ? ` — ${p.descripcion}` : "";
  const stock = p.enStock ? "Disponible" : "Agotado";
  console.log(`[${p.sku}] ${p.nombre} $${p.precio}${desc} (${stock})`);
}

const laptop: Producto = {
  sku: "LAP-001",
  nombre: "Laptop Pro 15",
  precio: 1299,
  descripcion: "Pantalla 4K, 16 GB RAM",
  enStock: true,
};

const mouse: Producto = {
  sku: "MOU-042",
  nombre: "Mouse Inalámbrico",
  precio: 25,
  enStock: false,
};

mostrarProducto(laptop); // [LAP-001] Laptop Pro 15 $1299 — Pantalla 4K, 16 GB RAM (Disponible)
mostrarProducto(mouse);  // [MOU-042] Mouse Inalámbrico $25 (Agotado)
```

> **Mini-ejercicio (6 min).** Define una `interface Libro` con `readonly isbn: string`,
> `titulo: string`, `autor: string`, `paginas: number` y `prestado?: boolean`.
> Crea dos libros: uno con `prestado: true` y otro sin esa propiedad.
> Escribe una función `resumen(l: Libro): string` que devuelva algo como
> `"El Quijote (ISBN: 123) — Prestado"` o `"El Quijote (ISBN: 123) — Disponible"`.

---

## Parte C · `type` vs `interface` — cuándo usar cada uno

---

### C.1 · Tabla comparativa y criterios de elección

Ambos describen la forma de un objeto, pero tienen diferencias clave.

```ts
// Concepto puro — ambos producen el MISMO tipo de objeto:

type PuntoT = { x: number; y: number };

interface PuntoI {
  x: number;
  y: number;
}

const a: PuntoT = { x: 1, y: 2 };
const b: PuntoI = { x: 1, y: 2 };
```

| Capacidad | `type` | `interface` |
|---|---|---|
| Describir la forma de un objeto | Sí | Sí |
| Uniones (`A \| B`) | Sí | No |
| Intersecciones (`A & B`) | Sí | Solo via `extends` |
| Extender otra definición | Solo `&` | `extends` (más legible) |
| Declaración fusionada (merging) | No | Sí (dos `interface` con igual nombre se fusionan) |
| Alias de primitivo o tupla | Sí | No |

**Regla práctica:**
- Usa `interface` para **objetos y contratos de clase** (se puede extender con facilidad).
- Usa `type` para **uniones, tuplas, primitivos y aliases complejos**.

```ts
// type para uniones — interface no puede hacer esto:
type Resultado = "exito" | "error" | "pendiente";

// type para tupla:
type Par = [string, number];

// interface para extender con claridad (ver sección D):
interface Animal { nombre: string }
interface Perro extends Animal { raza: string }
```

> **Mini-ejercicio (5 min).** Intenta crear una interfaz llamada `Color`
> que sea la unión `"rojo" | "verde" | "azul"`. Observa el error.
> Luego corrígelo cambiando `interface` por `type`. ¿Qué mensaje de error te da TypeScript?

---

## Parte D · Extender y componer tipos

---

### D.1 · `interface extends` y composición con `&`

Las interfaces se extienden con `extends`; los `type` se componen con `&` (intersección).

```ts
// Concepto puro

// Extensión de interface:
interface Animal {
  nombre: string;
  edad: number;
}

interface Mascota extends Animal {
  duenio: string;
  vacunado: boolean;
}

// Composición de type con intersección &:
type Auditable = {
  creadoEn: Date;
  actualizadoEn: Date;
};

type ProductoAuditable = Producto & Auditable;
// Un ProductoAuditable DEBE tener TODAS las props de Producto Y de Auditable.
```

```ts
// Múltiple herencia en interface:
interface Serializable {
  toJSON(): string;
}

interface Cloneable {
  clonar(): this;
}

// Una interfaz puede extender varias a la vez:
interface Entidad extends Serializable, Cloneable {
  id: number;
}
```

#### Ejemplo aplicado — jerarquía de usuarios en un sistema

```ts
interface EntidadBase {
  readonly id: number;
  creadoEn: Date;
}

interface Usuario extends EntidadBase {
  nombre: string;
  email: string;
}

interface Administrador extends Usuario {
  permisos: string[];
  nivel: 1 | 2 | 3;
}

// Intersección con type para añadir auditoría sin herencia:
type AdminConLog = Administrador & { ultimoAcceso: Date };

const admin: AdminConLog = {
  id: 1,
  creadoEn: new Date("2024-01-01"),
  nombre: "Sandra",
  email: "sandra@empresa.com",
  permisos: ["users:read", "users:write", "reports:read"],
  nivel: 2,
  ultimoAcceso: new Date(),
};

console.log(`Admin: ${admin.nombre} (nivel ${admin.nivel})`);
console.log(`Permisos: ${admin.permisos.join(", ")}`);
```

> **Mini-ejercicio (7 min).** Define `interface Vehiculo` con `marca: string`,
> `modelo: string` y `anio: number`. Extiéndela en `interface Coche extends Vehiculo`
> añadiendo `puertas: number` y en `interface Moto extends Vehiculo` añadiendo
> `cilindrada: number`. Crea un objeto de cada tipo e imprime sus datos.
> **Extra:** usa `&` para crear un `type VehiculoElectrico = Vehiculo & { autonomiaKm: number }`.

---

## Parte E · Index signatures y `Record<K, V>`

---

### E.1 · Index signatures — objetos con claves dinámicas

Cuando no conoces los nombres de las claves en tiempo de diseño, usas
una **firma de índice** para tipar el objeto.

```ts
// Concepto puro
interface Diccionario {
  [clave: string]: number;
}

const puntuaciones: Diccionario = {
  Ana: 95,
  Luis: 87,
  Marta: 100,
};

// Cualquier clave de tipo string es válida:
puntuaciones["Pedro"] = 72;

// Equivalente más conciso con Record:
type Marcador = Record<string, number>;

const marcador: Marcador = { equipo_a: 3, equipo_b: 1 };
```

> ⚠️ Las index signatures hacen que **todas** las propiedades deban ser del
> mismo tipo de valor. Si necesitas mezclar tipos, usa `type` con propiedades explícitas.

#### Ejemplo aplicado — inventario de almacén

```ts
type Inventario = Record<string, { cantidad: number; ubicacion: string }>;

const almacen: Inventario = {
  "LAP-001": { cantidad: 12, ubicacion: "Pasillo A, Estante 3" },
  "MOU-042": { cantidad: 50, ubicacion: "Pasillo B, Estante 1" },
  "TEC-007": { cantidad: 8,  ubicacion: "Pasillo A, Estante 5" },
};

function consultarProducto(sku: string): void {
  const item = almacen[sku];
  if (item) {
    console.log(`${sku}: ${item.cantidad} unidades en ${item.ubicacion}`);
  } else {
    console.log(`${sku}: no encontrado en almacén`);
  }
}

consultarProducto("MOU-042"); // MOU-042: 50 unidades en Pasillo B, Estante 1
consultarProducto("CAM-099"); // CAM-099: no encontrado en almacén

// Recorrer todo el inventario:
for (const sku in almacen) {
  console.log(`${sku} → ${almacen[sku].cantidad} uds.`);
}
```

> **Mini-ejercicio (6 min).** Crea un `type ConteoLetras = Record<string, number>`.
> Escribe una función `contarLetras(texto: string): ConteoLetras` que devuelva
> un objeto donde cada clave es una letra y el valor es cuántas veces aparece.
> Pista: recorre `texto.toLowerCase()` con `for...of` e inicializa a 0 si
> la clave aún no existe. Prueba con `"banana"`.

---

## Parte F · Tipos de función dentro de interfaces

---

### F.1 · Firmas de método y tipos de callback

Una interface puede incluir la **firma de una función** como propiedad.
Esto es especialmente útil para tipar callbacks y manejadores de eventos.

```ts
// Concepto puro — dos sintaxis equivalentes:
interface Calculadora {
  sumar(a: number, b: number): number;        // sintaxis de método
  restar: (a: number, b: number) => number;   // sintaxis de propiedad de función
}

// Tipo de callback independiente:
type Callback<T> = (error: Error | null, resultado: T | null) => void;

// Interface para un manejador de eventos:
interface Manejador {
  onExito: (datos: string) => void;
  onError: (err: Error) => void;
  onFinalizar?: () => void;  // opcional
}
```

#### Ejemplo aplicado — sistema de notificaciones con callbacks

```ts
type NivelLog = "info" | "warn" | "error";

interface Logger {
  log(nivel: NivelLog, mensaje: string): void;
  alerta: (mensaje: string) => void;
  onError?: (err: Error) => void;
}

// Implementación concreta:
const consoleLogger: Logger = {
  log(nivel, mensaje) {
    const prefijo = nivel === "error" ? "ERROR" : nivel === "warn" ? "AVISO" : "INFO";
    console.log(`[${prefijo}] ${mensaje}`);
  },
  alerta(mensaje) {
    console.log(`🔔 ALERTA: ${mensaje}`);
  },
  onError(err) {
    console.log(`Manejando error: ${err.message}`);
  },
};

// Función que acepta cualquier Logger:
function procesarPedido(id: number, logger: Logger): void {
  logger.log("info", `Procesando pedido #${id}...`);
  if (id < 0) {
    logger.onError?.(new Error("ID de pedido inválido"));
    return;
  }
  logger.log("info", `Pedido #${id} completado`);
  logger.alerta(`Pedido #${id} listo para envío`);
}

procesarPedido(101, consoleLogger);
procesarPedido(-5,  consoleLogger);
```

> **Mini-ejercicio (7 min).** Define una `interface Validador` con:
> `validar(valor: string): boolean` y `mensajeError: string`.
> Crea dos objetos que la implementen: uno que valide que un email
> contiene `"@"`, y otro que valide que una contraseña tiene al menos 8 caracteres.
> Escribe una función `ejecutarValidacion(val: string, v: Validador): void`
> que imprima `"OK"` o el `mensajeError`.

---

## Parte G · Uniones discriminadas — el patrón más poderoso

---

### G.1 · Discriminated unions con campo `kind` y narrowing

Una **unión discriminada** es un `type` unión donde cada miembro tiene
un campo literal común (el "discriminador"). TypeScript usa ese campo
para hacer **narrowing** automático y seguro.

```ts
// Concepto puro
type Circulo = {
  kind: "circulo";
  radio: number;
};

type Rectangulo = {
  kind: "rectangulo";
  ancho: number;
  alto: number;
};

type Triangulo = {
  kind: "triangulo";
  base: number;
  altura: number;
};

type Forma = Circulo | Rectangulo | Triangulo;

function calcularArea(f: Forma): number {
  switch (f.kind) {
    case "circulo":
      return Math.PI * f.radio ** 2;     // TypeScript sabe que f es Circulo
    case "rectangulo":
      return f.ancho * f.alto;           // TypeScript sabe que f es Rectangulo
    case "triangulo":
      return (f.base * f.altura) / 2;   // TypeScript sabe que f es Triangulo
  }
}
```

> **Truco.** Agrega un `default` con un valor de tipo `never` para
> garantizar que el `switch` es **exhaustivo**: si añades una nueva
> forma al `type` y olvidas el `case`, TypeScript lanzará un error.
> ```ts
> default:
>   const _exhaustivo: never = f;
>   throw new Error(`Forma no manejada: ${JSON.stringify(_exhaustivo)}`);
> ```

#### Ejemplo aplicado — procesador de eventos de una aplicación

```ts
// Cada evento tiene un campo "tipo" como discriminador:
type EventoLogin = {
  tipo: "login";
  usuarioId: number;
  timestamp: Date;
};

type EventoCompra = {
  tipo: "compra";
  usuarioId: number;
  monto: number;
  productoSku: string;
};

type EventoError = {
  tipo: "error";
  codigo: number;
  mensaje: string;
  critico: boolean;
};

type EventoApp = EventoLogin | EventoCompra | EventoError;

function procesarEvento(evento: EventoApp): string {
  switch (evento.tipo) {
    case "login":
      return `Usuario ${evento.usuarioId} inició sesión a las ${evento.timestamp.toISOString()}`;

    case "compra":
      return `Usuario ${evento.usuarioId} compró ${evento.productoSku} por $${evento.monto}`;

    case "error":
      const nivel = evento.critico ? "CRITICO" : "menor";
      return `Error ${nivel} [${evento.codigo}]: ${evento.mensaje}`;
  }
}

const eventos: EventoApp[] = [
  { tipo: "login",  usuarioId: 1,  timestamp: new Date() },
  { tipo: "compra", usuarioId: 1,  monto: 299, productoSku: "LAP-001" },
  { tipo: "error",  codigo: 500,   mensaje: "DB no disponible", critico: true },
  { tipo: "compra", usuarioId: 7,  monto: 25,  productoSku: "MOU-042" },
  { tipo: "error",  codigo: 404,   mensaje: "Recurso no encontrado", critico: false },
];

for (const e of eventos) {
  console.log(procesarEvento(e));
}
```

Salida:
```
Usuario 1 inició sesión a las 2024-06-23T...
Usuario 1 compró LAP-001 por $299
Error CRITICO [500]: DB no disponible
Usuario 7 compró MOU-042 por $25
Error menor [404]: Recurso no encontrado
```

> **Mini-ejercicio (8 min).** Modela los **medios de pago** de una tienda
> con una unión discriminada. Define tres tipos con campo `metodo`:
> `"tarjeta"` (con `ultimos4: string`, `marca: "visa" | "mastercard"`),
> `"transferencia"` (con `banco: string`, `referencia: string`) y
> `"efectivo"` (con `cambioRequerido: number`).
> Une en `type MedioPago`. Escribe `function confirmarPago(m: MedioPago): string`
> que use `switch` sobre `metodo` y devuelva un mensaje distinto por cada caso.
> Prueba con un objeto de cada tipo.

---

## Parte H · `readonly` y propiedades inmutables

---

### H.1 · `readonly` en interfaces, types y arrays

`readonly` impide **reasignar** una propiedad después de la inicialización.
También existe `ReadonlyArray<T>` (o `readonly T[]`) para arrays que no deben mutar.

```ts
// Concepto puro
interface Configuracion {
  readonly host: string;
  readonly puerto: number;
  readonly secreto: string;
  reintentos: number;        // esta SÍ puede cambiar
}

const cfg: Configuracion = {
  host: "db.empresa.com",
  puerto: 5432,
  secreto: "abc-xyz-123",
  reintentos: 3,
};

cfg.reintentos = 5;   // OK
// cfg.host = "otro"; // ERROR: no se puede asignar a 'host' (readonly)

// Arrays inmutables:
const ESTADOS_PERMITIDOS: readonly string[] = ["activo", "inactivo", "suspendido"];
// ESTADOS_PERMITIDOS.push("eliminado"); // ERROR
// ESTADOS_PERMITIDOS[0] = "otro";       // ERROR
```

> ⚠️ `readonly` es una garantía **en tiempo de compilación**, no en
> tiempo de ejecución. No es lo mismo que `Object.freeze()`.
> Si necesitas inmutabilidad real en runtime, usa `Object.freeze()` también.

#### Ejemplo aplicado — constantes de configuración de una API

```ts
type MetodoHTTP = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";

interface ConfigAPI {
  readonly baseUrl: string;
  readonly version: string;
  readonly metodosPermitidos: readonly MetodoHTTP[];
  timeoutMs: number;   // ajustable en runtime
}

const apiConfig: ConfigAPI = {
  baseUrl: "https://api.empresa.com",
  version: "v2",
  metodosPermitidos: ["GET", "POST", "PUT", "DELETE"],
  timeoutMs: 5000,
};

function construirUrl(config: ConfigAPI, ruta: string): string {
  return `${config.baseUrl}/${config.version}/${ruta.replace(/^\//, "")}`;
}

function validarMetodo(config: ConfigAPI, metodo: MetodoHTTP): boolean {
  return (config.metodosPermitidos as MetodoHTTP[]).includes(metodo);
}

apiConfig.timeoutMs = 10000;  // OK — puede cambiar
// apiConfig.baseUrl = "otro"; // ERROR

console.log(construirUrl(apiConfig, "/usuarios"));  // https://api.empresa.com/v2/usuarios
console.log(validarMetodo(apiConfig, "PATCH"));     // false
console.log(validarMetodo(apiConfig, "GET"));       // true
```

> **Mini-ejercicio (5 min).** Define un `type PlanSuscripcion` con
> `readonly nombre: string`, `readonly precioMensual: number`,
> `readonly caracteristicas: readonly string[]` y `activo: boolean`.
> Crea dos planes (`"Básico"` y `"Pro"`). Intenta cambiar `precioMensual`
> y observa el error. Luego cambia `activo` a `false` en uno de ellos (debe funcionar).

---

## Ejemplo combinado

Une `type`, `interface`, unión discriminada, extensión, index signature y `readonly`
en un mini-sistema de gestión de empleados:

```ts
// --- Tipos base ---
type EmpleadoID = string;
type Departamento = "ingenieria" | "marketing" | "ventas" | "rrhh";

interface EntidadBase {
  readonly id: EmpleadoID;
  readonly ingresadoEn: Date;
}

// --- Roles como unión discriminada ---
type RolDesarrollador = {
  rol: "desarrollador";
  lenguajes: string[];
  nivel: "junior" | "mid" | "senior";
};

type RolManager = {
  rol: "manager";
  equipoSize: number;
  presupuesto: number;
};

type RolPracticante = {
  rol: "practicante";
  tutorId: EmpleadoID;
  fechaFin: Date;
};

type RolEmpleado = RolDesarrollador | RolManager | RolPracticante;

// --- Empleado completo ---
interface Empleado extends EntidadBase {
  nombre: string;
  email: string;
  departamento: Departamento;
  salario: number;
  info: RolEmpleado;
}

// --- Índice de empleados por departamento ---
type DirectorioEmpleados = Record<Departamento, Empleado[]>;

// --- Función polimórfica usando narrowing ---
function descripcionRol(info: RolEmpleado): string {
  switch (info.rol) {
    case "desarrollador":
      return `Dev ${info.nivel} — ${info.lenguajes.join(", ")}`;
    case "manager":
      return `Manager de equipo (${info.equipoSize} personas, presupuesto $${info.presupuesto})`;
    case "practicante":
      return `Practicante hasta ${info.fechaFin.toLocaleDateString()} — tutor: ${info.tutorId}`;
  }
}

function reporteEmpleado(emp: Empleado): void {
  console.log(`-- ${emp.nombre} (${emp.id}) --`);
  console.log(`   Dept: ${emp.departamento} | Salario: $${emp.salario}`);
  console.log(`   Rol: ${descripcionRol(emp.info)}`);
}

// --- Datos de ejemplo ---
const empleados: Empleado[] = [
  {
    id: "EMP-001",
    ingresadoEn: new Date("2022-03-15"),
    nombre: "Laura Ruiz",
    email: "laura@empresa.com",
    departamento: "ingenieria",
    salario: 4800,
    info: { rol: "desarrollador", lenguajes: ["TypeScript", "Rust"], nivel: "senior" },
  },
  {
    id: "EMP-002",
    ingresadoEn: new Date("2020-01-10"),
    nombre: "Carlos Vega",
    email: "carlos@empresa.com",
    departamento: "ingenieria",
    salario: 6500,
    info: { rol: "manager", equipoSize: 8, presupuesto: 120000 },
  },
  {
    id: "EMP-003",
    ingresadoEn: new Date("2024-02-01"),
    nombre: "Sofía Torres",
    email: "sofia@empresa.com",
    departamento: "marketing",
    salario: 1200,
    info: { rol: "practicante", tutorId: "EMP-002", fechaFin: new Date("2024-08-01") },
  },
];

// --- Construir directorio por departamento ---
const directorio: DirectorioEmpleados = {
  ingenieria: [],
  marketing: [],
  ventas: [],
  rrhh: [],
};

for (const emp of empleados) {
  directorio[emp.departamento].push(emp);
}

// --- Reporte final ---
console.log("=== REPORTE DE EMPLEADOS ===\n");
for (const emp of empleados) {
  reporteEmpleado(emp);
}

console.log("\n=== DIRECTORIO POR DEPARTAMENTO ===");
for (const dept in directorio) {
  const lista = directorio[dept as Departamento];
  if (lista.length > 0) {
    console.log(`${dept}: ${lista.map((e) => e.nombre).join(", ")}`);
  }
}
```

Salida:
```
=== REPORTE DE EMPLEADOS ===

-- Laura Ruiz (EMP-001) --
   Dept: ingenieria | Salario: $4800
   Rol: Dev senior — TypeScript, Rust
-- Carlos Vega (EMP-002) --
   Dept: ingenieria | Salario: $6500
   Rol: Manager de equipo (8 personas, presupuesto $120000)
-- Sofía Torres (EMP-003) --
   Dept: marketing | Salario: $1200
   Rol: Practicante hasta 01/08/2024 — tutor: EMP-002

=== DIRECTORIO POR DEPARTAMENTO ===
ingenieria: Laura Ruiz, Carlos Vega
marketing: Sofía Torres
```

---

## Resumen de la página 5

- **`type` alias** da nombre a cualquier tipo: objetos, uniones, primitivos, tuplas. Es la única opción para uniones y aliases de primitivo.
- **`interface`** describe la forma de un objeto con propiedades obligatorias, opcionales (`?`) y de solo lectura (`readonly`). Es preferible para objetos y contratos de clase.
- **`type` vs `interface`**: usa `interface` para objetos y jerarquías; usa `type` para uniones, tuplas y aliases complejos.
- **Extensión y composición**: `interface extends` hereda propiedades; `type &` forma una intersección. Ambos pueden combinar múltiples fuentes.
- **Index signatures** `{ [clave: string]: V }` y `Record<K, V>` modelan objetos con claves dinámicas desconocidas en tiempo de diseño.
- **Tipos de función en interfaces**: las firmas de método y las propiedades de tipo función permiten tipar callbacks y manejadores de forma precisa.
- **Uniones discriminadas**: campo literal común (`kind`/`tipo`) + `switch` exhaustivo = narrowing seguro y código sin `as` ni `!`.
- **`readonly`**: garantía estática de inmutabilidad en propiedades y arrays; complementar con `Object.freeze()` si se necesita en runtime.

---

> **Siguiente página →** Página 6: Programación Orientada a Objetos — clases,
> modificadores de acceso, herencia, clases abstractas y polimorfismo.
