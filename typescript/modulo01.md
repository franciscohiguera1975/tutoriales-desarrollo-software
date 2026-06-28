# Tutorial TypeScript — Página 1
## Fundamentos
### Qué es TypeScript, entorno y tu primer programa tipado

---

> **Cómo usar esta página.** Cada bloque sigue el mismo ritmo:
> **1)** el concepto puro, **2)** un ejemplo aplicado real, **3)** un
> **mini-ejercicio** que se resuelve en clase en **5–8 minutos**.
> Escribe el código directamente en el
> [TypeScript Playground](https://www.typescriptlang.org/play) (sin instalar nada)
> o ejecútalo localmente con `npx tsx archivo.ts`.

---

## Parte A · ¿Qué es TypeScript y por qué tipos?

TypeScript es **JavaScript con tipos estáticos**. No es un lenguaje nuevo: es un
superconjunto de JS que añade anotaciones de tipo y un compilador que detecta errores
**antes** de que el código llegue al navegador o al servidor. Todo TypeScript válido
se convierte en JavaScript al compilar.

---

### A.1 · El problema que TypeScript resuelve

```ts
// ── JavaScript puro ─────────────────────────────────────────────────────────
// Esta función calcula el precio con descuento.
// ¿Qué pasa si alguien la llama con un string?

function aplicarDescuento(precio, pct) {
  return precio - precio * (pct / 100);
}

// JS acepta esto sin quejarse en el editor:
console.log(aplicarDescuento("250", 10)); // "250" - NaN → NaN  ← bug silencioso
console.log(aplicarDescuento(250, "10")); // NaN               ← bug silencioso
```

```ts
// ── TypeScript ───────────────────────────────────────────────────────────────
// El compilador rechaza las llamadas incorrectas EN EL EDITOR, no en producción.

function aplicarDescuento(precio: number, pct: number): number {
  return precio - precio * (pct / 100);
}

// Error en tiempo de compilación — nunca llega a ejecutarse:
// aplicarDescuento("250", 10);
// Argument of type 'string' is not assignable to parameter of type 'number'.

console.log(aplicarDescuento(250, 10)); // 225  ✅
```

> ⚠️ **Errores en compilación vs. ejecución.** Un bug en compilación lo ves en
> tu editor (línea subrayada en rojo) antes de guardar el archivo. Un bug en
> ejecución aparece cuando el usuario ya está usando la app — y puede costar caro.
> TypeScript mueve los errores hacia la izquierda en el tiempo.

#### Ejemplo aplicado — API de usuarios con bug que TS atrapa

```ts
// Imagina un endpoint que recibe el ID de usuario y devuelve su nombre.
// En JS es fácil pasar accidentalmente un objeto en lugar de un número.

interface Usuario {
  id: number;
  nombre: string;
  activo: boolean;
}

const usuarios: Usuario[] = [
  { id: 1, nombre: "Ana García",   activo: true  },
  { id: 2, nombre: "Luis Pérez",   activo: false },
  { id: 3, nombre: "María Torres", activo: true  },
];

function buscarUsuario(id: number): Usuario | undefined {
  return usuarios.find((u) => u.id === id);
}

const u = buscarUsuario(2);
console.log(u?.nombre); // Luis Pérez

// TS detecta el error antes de ejecutar:
// buscarUsuario({ id: 2 });
// Argument of type '{ id: number; }' is not assignable to parameter of type 'number'.
```

> **Mini-ejercicio (5 min).** En el Playground, escribe una función JS sin tipos
> `sumar(a, b)` que devuelva `a + b`. Llámala con `sumar(5, "3")` y observa el
> resultado (`"53"` — concatenación, no suma). Luego conviértela a TypeScript
> anotando `a: number, b: number` y observa cómo el error aparece al escribir
> la llamada incorrecta.

---

## Parte B · Entorno de desarrollo

---

### B.1 · Instalación y herramientas

```ts
// ── En la terminal, dentro de tu proyecto ───────────────────────────────────

// 1. Inicializar un proyecto Node (si no existe)
//    npm init -y

// 2. Instalar TypeScript y tsx (ejecutor directo de .ts sin compilar a mano)
//    npm install --save-dev typescript tsx

// 3. Crear la configuración del compilador (solo una vez por proyecto)
//    npx tsc --init

// 4. Compilar un archivo .ts a .js
//    npx tsc archivo.ts

// 5. Ejecutar directamente sin compilar (ideal para aprender y prototipar)
//    npx tsx archivo.ts
```

> **Truco — usa el Playground para aprender.** Si estás en clase o explorando
> una idea, el [TypeScript Playground](https://www.typescriptlang.org/play) es
> la herramienta más rápida: pegas código, ves el JS generado y los errores
> en tiempo real, sin instalar nada.

#### Ejemplo aplicado — tu primer archivo TypeScript local

```ts
// archivo: hola.ts
// Ejecutar: npx tsx hola.ts

const saludo: string = "Hola, TypeScript";
const version: number = 5;

console.log(`${saludo} v${version}`);
// Hola, TypeScript v5
```

---

### B.2 · `tsconfig.json` y el modo estricto

El archivo `tsconfig.json` controla cómo se comporta el compilador.
La opción más importante para escribir código robusto es `"strict": true`.

```jsonc
// tsconfig.json generado con: npx tsc --init
{
  "compilerOptions": {
    "target": "ES2022",       // versión de JS que se genera
    "module": "commonjs",     // sistema de módulos
    "strict": true,           // ← activa TODAS las verificaciones estrictas
    "outDir": "./dist",       // carpeta donde va el JS compilado
    "rootDir": "./src"        // carpeta donde está tu TS
  }
}

// "strict": true activa, entre otros:
//   - strictNullChecks  → null y undefined no son asignables a otros tipos
//   - noImplicitAny     → no puedes dejar variables sin tipo sin querer
//   - strictFunctionTypes → funciones se verifican de forma covariante
```

> ⚠️ **Activa `strict` siempre.** Los proyectos sin `strict` acumulan deuda
> técnica: errores que TS podría atrapar pasan desapercibidos. Todos los
> ejemplos de este curso asumen `"strict": true`.

---

## Parte C · Variables: `let`, `const` y anotaciones de tipo

---

### C.1 · `const` vs `let`

```ts
// const — valor que NO cambia (preferida por defecto)
const PI: number = 3.14159;
const NOMBRE_APP: string = "InventarioApp";
const DEBUG_MODE: boolean = false;

// let — valor que SÍ puede cambiar
let contador: number = 0;
let estadoConexion: string = "desconectado";
let usuarioActivo: boolean = false;

contador++;                         // 1
estadoConexion = "conectado";       // ok
usuarioActivo = true;               // ok

// PI = 3;  // ← Error: Cannot assign to 'PI' because it is a constant.
```

> **Truco — la regla práctica.** Empieza con `const` siempre. Cambia a `let`
> solo cuando necesites reasignar. Nunca uses `var` en TypeScript moderno.

---

### C.2 · Anotación de tipo vs. inferencia

TypeScript puede **deducir** el tipo automáticamente cuando asignas un valor.
No siempre tienes que escribir el tipo explícito.

```ts
// ── Anotación explícita ───────────────────────────────────────────────────
// Tú le dices a TS cuál es el tipo.
const puerto: number = 8080;
const host: string = "localhost";
const activo: boolean = true;

// ── Inferencia de tipo ────────────────────────────────────────────────────
// TS lo deduce del valor inicial — el tipo es el mismo, pero sin escribirlo.
const puerto2 = 8080;       // TypeScript infiere: number
const host2 = "localhost";  // TypeScript infiere: string
const activo2 = true;       // TypeScript infiere: boolean

// Ambas formas producen el mismo nivel de seguridad de tipos.
// Si intentas reasignar con el tipo incorrecto, TS da error en ambos casos:
// puerto2 = "9000";  // Error: Type 'string' is not assignable to type 'number'.

// ── Cuándo anotar explícitamente ─────────────────────────────────────────
// 1. Variables declaradas sin valor inicial:
let latencia: number;      // sin inicializar — necesita anotación
latencia = 45;

// 2. Cuando quieres un tipo más amplio que el valor inicial:
let codigo: number | string = 200;  // acepta número o string
codigo = "OK";  // válido

// 3. Parámetros de funciones (TS no puede inferirlos):
function ping(host: string, intentos: number): string {
  return `Ping a ${host} — ${intentos} intento(s)`;
}
```

#### Ejemplo aplicado — dashboard de servidores

```ts
// Las variables de estado de un servidor: cuándo anotar y cuándo inferir.

const NOMBRE_SERVIDOR = "web-prod-01";   // inferido: string — claro del valor
const PUERTO_DEFAULT  = 443;             // inferido: number
const ES_PRODUCCION   = true;            // inferido: boolean

// Variables que cambian durante la vida del servidor:
let solicitudesAtendidas: number = 0;    // anotación: se inicializa en 0 pero cambia
let ultimoError: string | null = null;   // anotación: puede ser null al inicio

// Función con anotaciones completas (necesarias en parámetros):
function registrarSolicitud(ruta: string, codigoHttp: number): void {
  solicitudesAtendidas++;
  console.log(`[${NOMBRE_SERVIDOR}] ${codigoHttp} ${ruta} — total: ${solicitudesAtendidas}`);
}

registrarSolicitud("/api/usuarios", 200);
registrarSolicitud("/api/productos", 404);
// [web-prod-01] 200 /api/usuarios — total: 1
// [web-prod-01] 404 /api/productos — total: 2
```

> **Mini-ejercicio (6 min).** Declara con `const` las constantes de una tienda en
> línea: `NOMBRE_TIENDA` (string), `IVA` (number, valor 0.19), `ABIERTA` (boolean).
> Luego declara con `let` el `stockDisponible` (number, empieza en 100) y
> `categoriaActual` (string, empieza en `"electronica"`). Cambia ambas variables
> e intenta cambiar `IVA` — observa el error de TS.

---

## Parte D · Tipos primitivos — aperitivo rápido

Los tres tipos primitivos más usados. El detalle completo (arrays, tuplas, enums,
union types, `any/unknown/never`) va en la **Página 2**.

---

### D.1 · `number`, `string` y `boolean`

```ts
// number — enteros y decimales, positivos y negativos
const precio: number      = 299.99;
const puerto: number      = 8080;
const temperatura: number = -5.3;
const hexColor: number    = 0xff5733;  // también acepta hexadecimal

// string — texto, con comillas simples, dobles o backtick
const email: string    = "soporte@empresa.com";
const protocolo: string = 'HTTPS';
const ruta: string     = `/api/v2/usuarios`;

// boolean — solo true o false
const estaActivo: boolean  = true;
const requiereAuth: boolean = false;
const esAdmin: boolean      = false;

// ── Aritmética con number ─────────────────────────────────────────────────
const subtotal = 1500;
const descuento = 150;
const total = subtotal - descuento;  // 1350

// ── Métodos de string ─────────────────────────────────────────────────────
const usuario = "  admin@corp.com  ";
console.log(usuario.trim().toLowerCase()); // "admin@corp.com"
console.log(email.includes("empresa"));    // true
console.log(email.split("@")[1]);          // "empresa.com"

// ── Lógica con boolean ────────────────────────────────────────────────────
const puedeAcceder: boolean = estaActivo && !requiereAuth;
console.log(puedeAcceder); // true
```

> **Mini-ejercicio (5 min).** Declara tres variables tipadas: `latitudCiudad`
> (number), `nombreCiudad` (string) y `esCapital` (boolean). Usa los valores
> de tu ciudad o una ciudad que conozcas. Imprime una línea que combine las tres
> con un template literal.

---

## Parte E · Template literals y operaciones tipadas

---

### E.1 · Template literals con tipos

```ts
// Concepto puro — template literals (backtick ``)
const nombre: string = "Ana";
const rol: string    = "administradora";
const sesiones: number = 42;

// Interpola cualquier expresión con ${ }
const bienvenida: string = `Bienvenida, ${nombre}. Rol: ${rol}. Sesiones: ${sesiones}.`;
console.log(bienvenida);
// Bienvenida, Ana. Rol: administradora. Sesiones: 42.

// Expresiones dentro de ${ }
const precio: number = 1200;
const iva: number    = 0.19;
const total: string  = `Precio con IVA: $${(precio * (1 + iva)).toFixed(2)}`;
console.log(total);
// Precio con IVA: $1428.00

// Multi-línea — sin concatenación ni \n
const reporte: string = `
=== Reporte del sistema ===
Servidor : web-01
Estado   : activo
Uptime   : 99.9%
`;
console.log(reporte);
```

#### Ejemplo aplicado — generador de logs de servidor

```ts
// Los sistemas de backend generan miles de líneas de log por hora.
// TypeScript asegura que los campos siempre sean del tipo correcto.

type NivelLog = "INFO" | "WARN" | "ERROR";

function log(nivel: NivelLog, servicio: string, mensaje: string): string {
  const timestamp = new Date().toISOString();
  const prefijo   = nivel === "ERROR" ? "❌" : nivel === "WARN" ? "⚠️ " : "✅";
  return `[${timestamp}] ${prefijo} [${nivel}] [${servicio}] ${mensaje}`;
}

console.log(log("INFO",  "AuthService",  "Usuario 'ana' ha iniciado sesión"));
console.log(log("WARN",  "DbPool",       "Conexiones al 80% de capacidad"));
console.log(log("ERROR", "PaymentGW",    "Timeout al procesar pago #4821"));

// TS detecta el error si pasas un nivel inválido:
// log("DEBUG", "Cache", "Hit");
// Argument of type '"DEBUG"' is not assignable to parameter of type 'NivelLog'.
```

> **Mini-ejercicio (6 min).** Escribe `function formatearPrecio(producto: string,
> precio: number, moneda: string): string` que devuelva algo como
> `"Laptop → $1 299.00 USD"`. Usa un template literal. Llámala con tres productos
> distintos. **Extra:** añade un parámetro `descuentoPct: number` y muestra también
> el precio con descuento.

---

## Parte F · El compilador como red de seguridad

---

### F.1 · Por qué evitar `any`

```ts
// any — le dice a TS "no verifiques este valor"
// Es el equivalente a apagar el sistema de seguridad.

let dato: any = "hola";
dato = 42;           // ok, any acepta cualquier cosa
dato = true;         // ok
dato = { x: 1 };    // ok — TS no protesta pero tampoco ayuda

// El peligro: los errores vuelven a ser silenciosos
function duplicar(valor: any): any {
  return valor * 2;  // si valor es un string, devuelve NaN — y TS no avisa
}

console.log(duplicar(5));       // 10   ✅
console.log(duplicar("hola")); // NaN  ← bug silencioso, exactamente lo que TS evita

// ── La alternativa correcta: anotar el tipo real ───────────────────────────
function duplicarSeguro(valor: number): number {
  return valor * 2;
}
// duplicarSeguro("hola");  // Error: Argument of type 'string' is not assignable
//                          // to parameter of type 'number'.
```

> ⚠️ **`any` desactiva TypeScript.** Cada `any` que escribes es una zona sin
> protección. En proyectos profesionales, `"noImplicitAny": true` (incluido en
> `strict`) obliga a que toda variable sin valor inicial tenga un tipo explícito.
> Si no conoces el tipo todavía, usa `unknown` — es `any` pero seguro, porque
> no puedes operar con él sin verificar el tipo primero.

---

### F.2 · Strict mode en acción

```ts
// Con "strict": true en tsconfig.json, el compilador activa estas verificaciones:

// 1. strictNullChecks — null y undefined se tratan como tipos propios
let nombre: string = "Luis";
// nombre = null;  // Error: Type 'null' is not assignable to type 'string'.

let apodo: string | null = null;  // para admitir null, debes declararlo
apodo = "Lucho";  // ok

// 2. noImplicitAny — no puedes olvidar el tipo en parámetros
// function procesar(datos) { ... }  // Error: Parameter 'datos' implicitly has an 'any' type.
function procesar(datos: string): string {
  return datos.toUpperCase();  // ok, TS sabe que datos es string
}

// 3. strictPropertyInitialization — las propiedades de clase deben inicializarse
class Servidor {
  nombre: string;
  puerto: number;

  constructor(nombre: string, puerto: number) {
    this.nombre = nombre;
    this.puerto = puerto;
  }
}

const s = new Servidor("api-01", 3000);
console.log(`${s.nombre}:${s.puerto}`);  // api-01:3000
```

> **Mini-ejercicio (5 min).** En el Playground, activa `strict` (Config → Strict).
> Luego escribe `function saludar(nombre) { return "Hola " + nombre; }` y observa
> el error `"Parameter 'nombre' implicitly has an 'any' type."`. Corrígelo anotando
> el tipo. Después declara `let usuario: string = null;` y observa el segundo error.
> Corrígelo con `string | null`.

---

## Ejemplo combinado — calculadora de envíos tipada

Une todo lo visto: variables, anotaciones, inferencia, template literals
y el compilador como red de seguridad.

```ts
// Sistema de cotización de envíos para una tienda e-commerce.
// Demuestra cómo los tipos previenen errores en lógica de negocio real.

type ZonaEnvio = "local" | "nacional" | "internacional";

interface Paquete {
  descripcion: string;
  pesoKg: number;
  valorDeclarado: number;
  zona: ZonaEnvio;
}

const TARIFAS: Record<ZonaEnvio, number> = {
  local:           2.50,   // $ por kg
  nacional:        5.00,
  internacional:  12.00,
};

const SEGURO_PCT = 0.005;  // 0.5% del valor declarado

function cotizarEnvio(paquete: Paquete): string {
  const tarifaBase = TARIFAS[paquete.zona];
  const costoFlete = tarifaBase * paquete.pesoKg;
  const costoSeguro = paquete.valorDeclarado * SEGURO_PCT;
  const total = costoFlete + costoSeguro;

  return `
📦 Cotización de envío
   Descripción : ${paquete.descripcion}
   Peso        : ${paquete.pesoKg} kg
   Zona        : ${paquete.zona}
   Flete       : $${costoFlete.toFixed(2)}
   Seguro      : $${costoSeguro.toFixed(2)}
   ─────────────────────────
   TOTAL       : $${total.toFixed(2)}
  `.trim();
}

const pedido1: Paquete = {
  descripcion: "Laptop Dell XPS 15",
  pesoKg: 2.1,
  valorDeclarado: 1800,
  zona: "nacional",
};

const pedido2: Paquete = {
  descripcion: "Auriculares Sony WH-1000XM5",
  pesoKg: 0.4,
  valorDeclarado: 350,
  zona: "internacional",
};

console.log(cotizarEnvio(pedido1));
console.log("---");
console.log(cotizarEnvio(pedido2));

// TS detecta si usas una zona inválida:
// const pedido3: Paquete = { ..., zona: "express" };
// Type '"express"' is not assignable to type 'ZonaEnvio'.
```

Salida:
```
📦 Cotización de envío
   Descripción : Laptop Dell XPS 15
   Peso        : 2.1 kg
   Zona        : nacional
   Flete       : $10.50
   Seguro      : $9.00
   ─────────────────────────
   TOTAL       : $19.50
---
📦 Cotización de envío
   Descripción : Auriculares Sony WH-1000XM5
   Peso        : 0.4 kg
   Zona        : internacional
   Flete       : $4.80
   Seguro      : $1.75
   ─────────────────────────
   TOTAL       : $6.55
```

---

## Reto final de clase (combina todo)

> **Validador de registro de usuarios (10–12 min, en parejas).**
> Escribe una función `validarRegistro(nombre: string, email: string,
> edad: number, password: string): string` que aplique estas reglas:
>
> 1. Si `nombre` tiene menos de 2 caracteres → `"Nombre demasiado corto"`.
> 2. Si `email` no contiene `"@"` → `"Email inválido"`.
> 3. Si `edad` es menor de 18 → `"Debes ser mayor de edad"`.
> 4. Si `password.length` es menor de 8 → `"Contraseña debe tener mínimo 8 caracteres"`.
> 5. Si todo es válido → `` `✅ Registro exitoso: bienvenido, ${nombre}` ``.
>
> Pruébala con al menos **cuatro** llamadas que cubran cada caso.
> **Extra:** cambia el tipo de retorno a `{ ok: boolean; mensaje: string }`
> y ajusta la función — observa cómo TS te obliga a devolver siempre esa forma.

---

## Resumen de la página 1

- **TypeScript = JavaScript + tipos estáticos.** Los errores de tipo se detectan en tiempo de compilación (en el editor), no cuando el usuario usa la app.
- **Entorno:** `npm install --save-dev typescript tsx`. Para aprender, usa el [Playground](https://www.typescriptlang.org/play). Para ejecutar localmente: `npx tsx archivo.ts`. Para compilar a JS: `npx tsc archivo.ts`.
- **`tsconfig.json` con `"strict": true`** es la configuración recomendada — activa todas las verificaciones relevantes, incluyendo `strictNullChecks` y `noImplicitAny`.
- **`const`** para valores que no cambian (preferida). **`let`** para valores que cambian. Nunca `var`.
- **Anotación de tipo** (`: number`, `: string`, `: boolean`) es necesaria en parámetros de funciones y variables sin valor inicial. En los demás casos, la **inferencia** de TS suele ser suficiente.
- **`any` desactiva la protección de tipos** — evítalo. Si el tipo es desconocido, usa `unknown`.
- **Template literals** con backtick `` ` `` permiten interpolar expresiones con `${ }` y escribir texto multi-línea sin concatenación.
- El compilador es tu red de seguridad: una línea subrayada en rojo hoy evita un bug en producción mañana.

---

> **Siguiente página →** Página 2: Tipos de Datos — number, string, boolean,
> arrays, tuplas, enums, union types, any/unknown/never y type narrowing.
