# Testing en React con Vitest — Página 1
## Módulo 1 · Introducción y configuración
### Por qué testear el frontend, la pirámide de tests y cómo poner Vitest a funcionar

---

## Cómo trabajar este módulo

Este módulo es **incremental** y empezamos **desde cero**: primero creamos el
proyecto React + TypeScript con Vite y luego le añadimos el testing **paso a
paso** — instalar dependencias, configurar Vite, crear el `setup.ts` y, por fin,
lograr que pase nuestro primer test. Tras cada paso conviene **guardar y
ejecutar** para ver el efecto inmediato. En desarrollo deja siempre abierto:

```bash
npm run test:watch
```

así Vitest re-ejecuta lo afectado al instante. Entre paso y paso encontrarás retos
**🔧 Tu turno** que debes intentar resolver **antes** de abrir la solución.

```
Paso 0  →  crear el proyecto React + TS con Vite (npm create vite)
Paso 1  →  instalar las dependencias de testing
Paso 2  →  configurar vite.config.ts (entorno jsdom)
Paso 3  →  crear src/test/setup.ts (matchers jest-dom)
Paso 4  →  scripts npm + primer test que pasa (smoke test)
```

---

## ¿Por qué testear el frontend?

Durante años se consideró que las pruebas automatizadas eran cosa del backend. El frontend, decían, "cambia demasiado" y "es visual, hay que verlo con los ojos". Esa idea quedó obsoleta. Hoy una aplicación React puede tener decenas de miles de líneas, varios desarrolladores tocando los mismos componentes y reglas de negocio que viven en el cliente.

Sin pruebas, cada cambio es una apuesta. Testear el frontend nos da:

- **Confianza para refactorizar.** Cambias la implementación interna de un componente y, si las pruebas siguen en verde, sabes que el comportamiento visible no se rompió.
- **Documentación viva.** Un test bien escrito describe *qué hace* el componente mejor que cualquier comentario.
- **Detección temprana.** Un bug atrapado en CI cuesta minutos; el mismo bug en producción cuesta horas y reputación.
- **Diseño más limpio.** Lo que es difícil de testear suele estar mal diseñado. El test es un primer "usuario" de tu código.

> La pregunta no es "¿tengo tiempo de escribir tests?", sino "¿tengo tiempo de depurar en producción lo que un test habría evitado?".

---

## La pirámide de tests

No todas las pruebas son iguales. La **pirámide de tests** ordena los tipos según su número ideal, su velocidad y su coste de mantenimiento.

```
            /\
           /  \        E2E  (pocos, lentos, caros)
          /____\       Cypress / Playwright
         /      \
        /        \     Integración (varios, medios)
       /__________\    RTL: varios componentes juntos
      /            \
     /              \  Unitarios (muchos, rápidos, baratos)
    /________________\ Vitest: funciones, hooks, 1 componente
```

| Nivel | Qué prueba | Velocidad | Cantidad ideal |
|-------|------------|-----------|----------------|
| **Unitario** | Una función, un hook o un componente aislado | Milisegundos | Muchos |
| **Integración** | Varios componentes colaborando (formulario + lista) | Decenas de ms | Algunos |
| **E2E** | La app completa en un navegador real | Segundos | Pocos |

La regla práctica: **muchos tests unitarios, algunos de integración y unos pocos E2E**. En este curso nos centramos en los dos niveles inferiores con **Vitest** y **Testing Library**, que es donde está la mejor relación coste/beneficio para una app React.

---

## ¿Qué es Vitest y por qué usarlo?

**Vitest** es un *test runner* moderno construido sobre Vite. Si tu proyecto ya usa Vite (como el nuestro), Vitest reutiliza tu mismo `vite.config.ts`, tus *plugins*, tus *aliases* y tu transformación de TypeScript/JSX. No hay que mantener dos configuraciones paralelas.

Sus ventajas principales:

- **Integración nativa con Vite.** Cero duplicación de configuración.
- **Velocidad.** Usa *esbuild* para transformar y ejecuta en paralelo con un *watcher* inteligente que solo re-ejecuta lo afectado.
- **API compatible con Jest.** `describe`, `it`, `expect`, `vi.fn()`... Si vienes de Jest, te sentirás en casa.
- **Soporte de TypeScript y ESM de primera clase**, sin transpiladores extra.
- **UI integrada** (`vitest --ui`) para ver los tests en el navegador.

---

## Vitest vs Jest

| Aspecto | Vitest | Jest |
|---------|--------|------|
| Configuración | Comparte `vite.config.ts` | Requiere `jest.config` propio |
| ESM / TypeScript | Nativo | Necesita `babel`/`ts-jest` |
| Velocidad en proyectos Vite | Muy alta | Media (transpila aparte) |
| API de aserciones | `expect`, `vi` | `expect`, `jest` |
| Mocks | `vi.fn()`, `vi.mock()` | `jest.fn()`, `jest.mock()` |
| UI de tests | `vitest --ui` incluida | Requiere terceros |
| Watch mode | Instantáneo (HMR de tests) | Más lento |

La diferencia clave para nuestro día a día: en Vitest el objeto global de utilidades de mocking es **`vi`**, no `jest`. A lo largo del curso usaremos siempre `vi`.

---

## El proyecto de ejemplo: "Todo App con login simulado"

Todo el curso girará en torno a una misma aplicación: una lista de tareas con un login simulado. Mantener una sola app nos permite ver cómo evolucionan las pruebas a medida que crece la complejidad.

Estos son los tipos compartidos que usaremos en todos los módulos:

```ts
// src/types.ts
// Tipos compartidos por toda la aplicación.

// Una tarea individual.
export interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

// Filtros disponibles para la lista.
export type Filter = 'all' | 'active' | 'completed';

// Usuario autenticado (login simulado).
export interface User {
  id: string;
  name: string;
}
```

La estructura de carpetas será:

```
src/
├── api/
│   └── todos.ts          // fetchTodos(): Promise<Todo[]>
├── components/
│   ├── LoginForm.tsx
│   ├── AddTodoForm.tsx
│   ├── TodoItem.tsx
│   ├── TodoList.tsx
│   ├── TodoFilter.tsx
│   └── TodoApp.tsx
├── context/
│   └── AuthContext.tsx   // { user, login, logout }
├── hooks/
│   ├── useTodos.ts
│   └── useAuth.ts
├── test/
│   └── setup.ts          // configuración global de tests
└── types.ts
```

> No crearemos todos estos archivos hoy. El proyecto **nace casi vacío** y crece
> módulo a módulo: cada página añade el componente o hook que toca y sus tests.

### Mapa de construcción progresiva

Esta tabla es tu hoja de ruta: muestra **qué añades al proyecto** y **qué tests escribes** en cada módulo. Así nunca pierdes de vista cómo encaja cada pieza.

| Módulo | Añades al proyecto | Tests que escribes |
|--------|--------------------|--------------------|
| **1** | Proyecto base + config de testing + `Hello` | *smoke test* mínimo |
| **2** | `TodoItem` | render: texto, checkbox, botón |
| **3** | `TodoList` | queries `getBy`/`queryBy`/`findBy` |
| **4** | — | *matchers* de jest-dom sobre `TodoItem` |
| **5** | `AddTodoForm` | interacción con `user-event` |
| **6** | `TodoFilter` | props, estado y renderizado de listas |
| **7** | `useTodos`, `useAuth` | hooks con `renderHook` |
| **8** | `api/todos.ts` | *mocking* con `vi.fn`/`vi.mock` |
| **9** | *fetch* en `TodoApp` | asíncrono + MSW |
| **10** | `AuthContext` + router | Context + `MemoryRouter` |
| **11** | — | cobertura y buenas prácticas |

---

## El stack de testing

Para este curso usamos versiones modernas:

- **React 19.2** + **Vite 8** + **TypeScript 5.9** (modo `strict`, `verbatimModuleSyntax`).
- **Vitest 3.x** como *runner*.
- **@testing-library/react ^16** para renderizar y consultar componentes.
- **@testing-library/jest-dom ^6** para *matchers* de DOM.
- **@testing-library/user-event ^14** para simular interacción realista.
- **jsdom ^25** como entorno de DOM en Node.
- **msw ^2** para *mockear* la red (lo veremos en el módulo 9).

> Como usamos `verbatimModuleSyntax`, importaremos los **tipos** con `import type`. Esto no es opcional: TypeScript dará error si importas un tipo sin `type`.

---

## Paso 0 — Crear el proyecto React + TypeScript con Vite

Si ya tienes un proyecto Vite + React + TS, salta al Paso 1. Si partes de cero,
créalo ahora — será el proyecto que iremos llenando de componentes y tests
durante todo el curso.

```bash
# Crea el proyecto con el template oficial react + typescript
npm create vite@latest todo-app -- --template react-ts

# Entra e instala las dependencias base (react, vite, typescript...)
cd todo-app
npm install

# Arranca el servidor de desarrollo para comprobar que todo funciona
npm run dev
```

Abre `http://localhost:5173`: deberías ver la página de bienvenida de Vite. Con
eso confirmas que React, TypeScript y Vite están operativos. Detén el servidor
con `Ctrl+C` cuando quieras.

El template genera esta base (resumida):

```
todo-app/
├── src/
│   ├── App.tsx          ← componente raíz (lo limpiaremos)
│   ├── main.tsx         ← punto de entrada
│   └── index.css
├── index.html
├── package.json
├── tsconfig.json
├── tsconfig.app.json
└── vite.config.ts       ← aquí añadiremos el bloque test en el Paso 2
```

Limpia el `App.tsx` de demostración y déjalo mínimo; lo iremos ampliando módulo
a módulo según el **mapa de construcción progresiva**:

```tsx
// src/App.tsx
export default function App() {
  return (
    <main>
      <h1>Todo App</h1>
      {/* Iremos montando aquí TodoApp, LoginForm, etc. en los próximos módulos */}
    </main>
  );
}
```

Crea también el archivo de **tipos compartidos** que ya vimos, porque casi todos
los módulos lo importarán:

```ts
// src/types.ts
export interface Todo {
  id: string;
  text: string;
  completed: boolean;
}

export type Filter = 'all' | 'active' | 'completed';

export interface User {
  id: string;
  name: string;
}
```

> El template `react-ts` ya trae los `tsconfig` en modo `strict` y con
> `verbatimModuleSyntax` activado — exactamente lo que necesitamos.

### 🔧 Tu turno #0

Antes de instalar nada de testing: abre el `package.json` recién creado y localiza
las versiones de `react`, `vite` y `typescript`. ¿En qué sección está cada una:
`dependencies` o `devDependencies`? ¿Por qué `react` va en una y `vite` en otra?

<details><summary>✅ Solución</summary>

`react` y `react-dom` van en **`dependencies`** porque forman parte del código que
se ejecuta en producción (viajan al *bundle*). `vite`, `typescript` y `@types/*`
van en **`devDependencies`** porque solo se usan al desarrollar y compilar, no en
el navegador del usuario final. Las dependencias de testing del Paso 1 también
serán `devDependencies`, por la misma razón.

</details>

---

## Paso 1 — Instalar las dependencias

Empezamos por lo mínimo: traer el *runner* y las utilidades de testing. Todas son dependencias de **desarrollo** (`-D`), porque no viajan al bundle de producción:

```bash
# Instala el runner y las utilidades de testing
npm i -D vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
```

Más adelante (módulo 9) añadiremos MSW; **todavía no lo necesitamos**:

```bash
# Solo cuando lleguemos al módulo de red
npm i -D msw
```

Tras instalar, abre `package.json` y comprueba que las dependencias aparecen bajo `devDependencies`. Si están ahí, el Paso 1 está listo.

### 🔧 Tu turno #1

Sin mirar abajo: **predice** qué pasa si intentas ejecutar `npx vitest run` **ahora mismo**, sin haber creado ningún archivo `.test.tsx` ni configurado nada. ¿Error? ¿Pasa? ¿Cuántos tests encuentra?

<details><summary>✅ Solución / qué observar</summary>

Vitest arranca, no encuentra archivos de test y termina con un mensaje del tipo
`No test files found, exiting with code 1`. No es un fallo de configuración: es
que aún no hay nada que ejecutar. Esto confirma que el binario `vitest` quedó
instalado correctamente. En los próximos pasos le daremos un entorno y un test.

</details>

---

## Paso 2 — Configurar `vite.config.ts` (entorno jsdom)

Vitest lee la configuración del bloque `test` dentro de tu `vite.config.ts`. Vamos a añadir ese bloque **una línea relevante a la vez** para entender cada opción.

> **Importante — el orden importa.** Edita el `vite.config.ts` **después** de haber
> hecho el Paso 1 (`npm i -D vitest ...`). Si lo editas antes, el editor marcará
> `Cannot find module 'vitest/config'` porque el paquete aún no está instalado.

La clave del bloque `test` es de **dónde** importas `defineConfig`. El campo `test`
**no existe** en la configuración estándar de Vite; lo aporta Vitest. Por eso
importamos `defineConfig` desde **`vitest/config`** (no desde `vite`):

```ts
// vite.config.ts
import { defineConfig } from 'vitest/config'; // ← desde vitest/config, NO desde 'vite'
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    // Simula un navegador (document, window) dentro de Node.
    environment: 'jsdom',
  },
});
```

> **¿Por qué `vitest/config` y no `vite`?** Si importas `defineConfig` desde
> `'vite'`, TypeScript usa el tipo `UserConfig` de Vite, que **no tiene** la
> propiedad `test`, y obtienes el error:
> `Object literal may only specify known properties, and 'test' does not exist...`.
> `vitest/config` reexporta `defineConfig` con el tipo **extendido** que sí
> incluye `test`. (Existe también la variante con `/// <reference types="vitest/config" />`
> + import desde `'vite'`, pero es más frágil; preferimos el import directo.)

Ahora le sumamos comodidad y predictibilidad: `globals`, `setupFiles` e `include`.

```ts
// vite.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    // Simula un navegador (document, window) dentro de Node.
    environment: 'jsdom',
    // Permite usar describe/it/expect sin importarlos en cada archivo.
    globals: true,
    // Archivo que se ejecuta una vez antes de toda la suite.
    setupFiles: './src/test/setup.ts',
    // Vitest incluye por defecto los **/*.test.tsx; lo dejamos explícito.
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
  },
});
```

Puntos importantes:

- **`environment: 'jsdom'`** crea un DOM falso en memoria. Sin esto, `document` no existe y RTL falla.
- **`globals: true`** evita tener que escribir `import { describe, it, expect } from 'vitest'` en cada archivo. Es cómodo, pero requiere un ajuste en `tsconfig` (ver abajo).
- **`setupFiles`** apunta a un archivo que se ejecuta antes de los tests para registrar configuraciones globales (lo creamos en el Paso 3).

Si usas `globals: true`, añade los tipos globales en tu `tsconfig.json` para que TypeScript reconozca `describe`/`it`/`expect` y los matchers:

```json
// tsconfig.json (fragmento)
{
  "compilerOptions": {
    "types": ["vitest/globals", "@testing-library/jest-dom"]
  }
}
```

### 🔧 Tu turno #2

`setupFiles` apunta a `./src/test/setup.ts`, **un archivo que aún no existe**.
Predice: si guardas ahora y ejecutas `npx vitest run`, ¿qué tipo de error verás?
¿Será sobre los tests o sobre el arranque?

<details><summary>✅ Solución / qué observar</summary>

El error ocurre **en el arranque**, antes de ejecutar ningún test: Vitest no
puede cargar el `setupFiles` porque la ruta no resuelve (algo como
`Failed to load ... setup.ts` / `Cannot find module`). La lección: el
`setupFiles` es un requisito de configuración, no un test. Lo arreglamos en el
Paso 3 creando precisamente ese archivo.

</details>

---

## Paso 3 — Crear `src/test/setup.ts` (matchers jest-dom)

Este archivo registra los *matchers* de **jest-dom** (como `toBeInTheDocument`) y se ejecuta una sola vez, antes de toda la suite. Importamos la variante específica para Vitest:

```ts
// src/test/setup.ts
// Registra los matchers de jest-dom (toBeInTheDocument, toBeVisible, etc.)
// en el `expect` de Vitest. Se ejecuta una vez antes de toda la suite.
import '@testing-library/jest-dom/vitest';
```

> Ojo: usamos `@testing-library/jest-dom/vitest`, **no** `@testing-library/jest-dom`. La variante `/vitest` extiende el `expect` correcto y configura el *cleanup* automático tras cada test.

Con esto, el flujo de arranque queda completo:

```
vitest
  │
  ▼
lee vite.config.ts (bloque test)
  │
  ▼
crea entorno jsdom  ──►  globals (describe/it/expect)
  │
  ▼
ejecuta setupFiles (setup.ts) ──► registra matchers jest-dom
  │
  ▼
ejecuta los archivos *.test.tsx
```

Ahora que el archivo existe, el error del *Tu turno #2* desaparece: Vitest ya
puede cargar el `setupFiles`.

---

## Paso 4 — Scripts npm y el primer test que pasa

Antes de escribir el test, añade estos *scripts* a tu `package.json` para tener atajos cómodos:

```json
// package.json (fragmento)
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:ui": "vitest --ui",
    "coverage": "vitest run --coverage"
  }
}
```

| Script | Comando | Cuándo usarlo |
|--------|---------|---------------|
| `npm test` | `vitest run` | Una sola pasada (ideal para CI) |
| `npm run test:watch` | `vitest` | Desarrollo: re-ejecuta al guardar |
| `npm run test:ui` | `vitest --ui` | Inspeccionar tests en el navegador |
| `npm run coverage` | `vitest run --coverage` | Medir cobertura (módulo 11) |

> Por defecto `vitest` sin argumentos entra en modo *watch*; `vitest run` hace una sola pasada y termina. Esta diferencia confunde a muchos al principio.

**Convención de archivos de test.** Los archivos de prueba viven **junto al código que prueban** y terminan en `.test.tsx` (o `.test.ts` para lógica sin JSX):

```
src/components/
├── TodoItem.tsx
├── TodoItem.test.tsx      ← test del componente
├── AddTodoForm.tsx
└── AddTodoForm.test.tsx
```

Esta colocación tiene ventajas: al abrir un componente ves su test al lado, y al moverlo arrastras ambos. La alternativa (`__tests__/`) también es válida, pero preferimos la cercanía.

Ahora, el componente más trivial posible para verificar que **todo el andamiaje funciona**. Todavía **no** escribimos tests reales de comportamiento (eso es el módulo 2); solo comprobamos que importar y renderizar no explota:

```tsx
// src/components/Hello.tsx
// Componente trivial usado únicamente para verificar la configuración.
interface HelloProps {
  name: string;
}

export function Hello({ name }: HelloProps) {
  return <p>Hola, {name}</p>;
}
```

Y su *smoke test*, lo más básico posible:

```tsx
// src/components/Hello.test.tsx
import { describe, it, expect } from 'vitest';
import { Hello } from './Hello';

describe('Hello', () => {
  // El test mínimo: el componente existe y es importable.
  it('debería estar definido', () => {
    expect(Hello).toBeDefined();
  });
});
```

Ejecuta una sola pasada:

```bash
# Confirma que el setup completo funciona
npm test
```

Si ves `1 passed`, ¡la configuración de los cuatro pasos es correcta!

```
 ✓ src/components/Hello.test.tsx (1 test) 3ms
   ✓ Hello > debería estar definido

 Test Files  1 passed (1)
      Tests  1 passed (1)
```

En el módulo 2 daremos el salto a renderizar de verdad con Testing Library.

---

### Prueba esto

- Cambia `globals: true` a `false` y observa el error al usar `describe` sin importarlo; luego añade el `import { describe, it, expect } from 'vitest'`.
- Borra temporalmente `environment: 'jsdom'` y mira qué error aparece cuando un test intente usar `document`.
- Renombra `Hello.test.tsx` a `Hello.spec.tsx` y comprueba que sigue ejecutándose gracias al patrón `include`.
- Ejecuta `npm run test:watch`, edita el texto del componente y observa cómo Vitest re-ejecuta al instante.
- Lanza `npm run test:ui` y explora la interfaz en el navegador.
- Quita el `import '@testing-library/jest-dom/vitest'` del `setup.ts` y verás que más adelante `toBeInTheDocument` deja de existir.

---

## Ejercicios propuestos

### Básico

**B1 — Segundo smoke test.** Crea un componente trivial `Goodbye` (devuelve `<p>Adiós, {name}</p>`) y su test `debería estar definido`. Comprueba que la salida pasa a `2 passed` repartidos en dos archivos.

**B2 — Renombrado de patrón.** Cambia el `include` para que solo acepte `*.test.tsx` (sin `spec`). Renombra `Hello.test.tsx` a `Hello.spec.tsx` y verifica que **ya no se ejecuta**. Vuelve a dejarlo como estaba.

### Intermedio

**I1 — Sin `globals`.** Pon `globals: false` en `vite.config.ts`, quita los tipos `vitest/globals` del `tsconfig.json` y arregla `Hello.test.tsx` importando explícitamente `describe`, `it` y `expect` desde `'vitest'`. Reflexiona sobre la ventaja y el coste de cada enfoque.

**I2 — Setup verificable.** Añade en `src/test/setup.ts` un `console.log('setup cargado')` y ejecuta `npm test`. Localiza el mensaje en la salida y razona **cuántas veces** aparece aunque haya varios archivos de test.

### Avanzado

**A1 — Cobertura inicial.** Ejecuta `npm run coverage` (puede pedirte instalar `@vitest/coverage-v8`). Interpreta la tabla de cobertura: ¿por qué `Hello.tsx` puede aparecer con baja cobertura pese a tener un test? Relaciona la respuesta con que el *smoke test* no llega a renderizar.

**A2 — Dos entornos.** Investiga el comentario `// @vitest-environment node` al inicio de un archivo de test. Crea un test de una función pura (sin DOM) que se ejecute en entorno `node` mientras el resto sigue en `jsdom`. Explica cuándo conviene cambiar de entorno por archivo.

---

## Resumen del módulo 1

- Testear el frontend da confianza para refactorizar, sirve de documentación viva y atrapa bugs temprano.
- La **pirámide de tests** recomienda muchos unitarios, algunos de integración y pocos E2E.
- **Vitest** se integra de forma nativa con Vite, es rápido y ofrece una API compatible con Jest, pero su objeto de utilidades es **`vi`**, no `jest`.
- Partimos **desde cero**: en el **Paso 0** creamos el proyecto con `npm create vite@latest todo-app -- --template react-ts`, lo limpiamos y añadimos `src/types.ts`. El proyecto nace casi vacío y crece módulo a módulo (ver el **mapa de construcción progresiva**).
- Construimos la configuración **por pasos**: (1) instalar `vitest`, `@testing-library/*` y `jsdom`; (2) configurar el bloque `test` en `vite.config.ts` importando `defineConfig` desde **`vitest/config`** (no desde `'vite'`) y ajustando `environment: 'jsdom'`, `globals: true`, `setupFiles` e `include`; (3) crear `src/test/setup.ts` que importa `@testing-library/jest-dom/vitest`; (4) añadir los scripts npm y hacer pasar un *smoke test*.
- Los tests viven junto al código en archivos `.test.tsx` y verificamos el setup con un *smoke test* mínimo (`toBeDefined`).
- Los scripts npm `test`, `test:watch`, `test:ui` y `coverage` cubren CI, desarrollo, inspección visual y cobertura.

> **Siguiente página →** Módulo 2: Anatomía de un test — `describe`/`it`/`expect`, el patrón AAA y nuestro primer test real de `TodoItem`.
