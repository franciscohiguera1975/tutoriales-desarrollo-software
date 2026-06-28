# Testing E2E en React con Cypress — Página 1
## Módulo 1 · Introducción y configuración
### Qué es el testing E2E, por qué Cypress y cómo dejar tu proyecto listo para escribir tu primer test

---

## Cómo trabajar este módulo

Este módulo es **incremental**: vamos a dejar Cypress instalado y configurado
**paso a paso**, no de un solo golpe. La idea es que **construyas junto al
profesor**: ejecuta cada comando, crea cada archivo y comprueba el resultado
antes de avanzar al siguiente paso.

Para sacarle todo el jugo, ten a mano dos cosas:

- La app corriendo en `http://localhost:5173` (la **Todo App con login
  simulado** que usaremos en todo el curso).
- Una terminal lista para `npx cypress open` y ver cómo el runner reacciona a
  cada cambio.

Entre paso y paso encontrarás retos **🔧 Tu turno** que debes resolver **antes**
de mirar la solución colapsable.

```
Paso 1  →  instalar Cypress como dependencia de dev
Paso 2  →  abrir el runner y dejar que genere la carpeta cypress/
Paso 3  →  entender la estructura de carpetas
Paso 4  →  escribir cypress.config.ts (baseUrl)
Paso 5  →  scripts de npm y TypeScript para Cypress
Paso 6  →  tu primer cy.visit en verde
```

---

## ¿Qué es el testing E2E?

**E2E** significa *End-to-End* (de extremo a extremo). Un test E2E ejercita tu aplicación **como lo haría una persona real**: abre el navegador, navega a una URL, escribe en los inputs, hace click en botones y verifica que la pantalla responde como se espera.

A diferencia de otros tipos de pruebas, un test E2E **no conoce** los detalles internos de tu código. No le importa qué hook usaste, ni cómo organizaste tus componentes. Solo le importa una cosa: **¿el usuario puede completar su tarea?**

A lo largo de este curso probaremos una **Todo App con login simulado** servida en `http://localhost:5173`. Tiene dos rutas:

- `/login` — un formulario con usuario y contraseña.
- `/todos` — la lista de tareas, donde el usuario crea, completa, filtra y elimina todos.

Login válido durante todo el curso: usuario **`admin`**, contraseña **`1234`**.

---

## Unit vs Integration vs E2E

No todas las pruebas son iguales. Cada nivel tiene un propósito, un costo y una velocidad distintos.

```
            Cantidad        Velocidad      Confianza "el usuario puede usarlo"
            ────────        ─────────      ───────────────────────────────────

   ▲  E2E   pocas           lenta          ALTA
   │        ┌──────────┐    (navegador
   │        │   E2E    │     real)
   │        ├──────────┤
   │ Integ. │ Integr.  │    media          media
   │        ├──────────┤
   │  Unit  │   Unit   │    rápida         baja (prueba piezas aisladas)
   │        │          │    (sin DOM
   ▼        └──────────┘     real)
```

Esta forma de pirámide se conoce como la **pirámide de testing**: muchas pruebas unitarias en la base, algunas de integración en medio, y pocas E2E en la punta.

| Tipo | Qué prueba | Ejemplo en la Todo App | Velocidad | Aísla con mocks |
|------|-----------|------------------------|-----------|-----------------|
| **Unit** | Una función o componente aislado | `formatDate()` devuelve `"23/06/2026"` | Muy rápida | Casi todo |
| **Integration** | Varias piezas trabajando juntas | `<TodoApp/>` renderiza los todos que recibe del store | Media | Algunas dependencias |
| **E2E** | El sistema completo, como el usuario | Login → crear todo → marcarlo → eliminarlo | Lenta | Nada (o casi nada) |

> Regla práctica: usa **muchas** pruebas unitarias para la lógica, y **pocas** pruebas E2E para los flujos críticos del negocio (login, checkout, alta de datos).

### 🔧 Tu turno #1

Antes de tocar código: clasifica estas tres situaciones como **Unit**, **Integration** o **E2E**, según la tabla de arriba.

1. Compruebas que la función `validarPassword('1234')` devuelve `true`.
2. Abres el navegador, escribes usuario y contraseña, pulsas Entrar y verificas que la URL cambia a `/todos`.
3. Renderizas `<TodoList todos={[...]} />` y verificas que pinta 3 elementos.

<details><summary>✅ Solución</summary>

1. **Unit** — una función pura y aislada.
2. **E2E** — el flujo completo en un navegador real, como el usuario.
3. **Integration** — varias piezas (lista + items) trabajando juntas, pero sin navegador real.

En este curso nos centraremos en el caso 2: los flujos E2E críticos.

</details>

---

## ¿Qué es Cypress y cómo funciona?

**Cypress** es una herramienta de testing que ejecuta tus pruebas **dentro de un navegador real** (Chrome, Edge, Firefox o Electron). No simula el DOM con una librería: abre una ventana de verdad y controla la aplicación desde dentro.

Esto tiene consecuencias muy concretas:

- Ves la aplicación **renderizándose en vivo** mientras el test corre.
- Cypress puede hacer **time-travel**: retroceder a cualquier paso del test y ver cómo estaba la pantalla en ese instante.
- Las pruebas se ejecutan en el mismo *event loop* que tu app, lo que las hace rápidas y deterministas.

```
┌─────────────────────────────────────────────┐
│              Navegador real                    │
│                                                │
│   ┌──────────────┐      ┌──────────────────┐  │
│   │   Tu app      │◄────►│  Test de Cypress │  │
│   │  (React)      │      │  (login.cy.ts)   │  │
│   └──────────────┘      └──────────────────┘  │
│         ▲                                       │
└─────────┼───────────────────────────────────┘
          │  controla / observa
          ▼
   Node.js (proceso de Cypress: servidor, plugins, red)
```

El test y la app conviven en la misma pestaña. Por eso Cypress puede esperar inteligentemente a que un elemento aparezca, en lugar de fallar al instante.

---

## Cypress vs Playwright (honesto y breve)

Ambas son excelentes. La elección depende del equipo y del proyecto.

| Criterio | Cypress | Playwright |
|----------|---------|------------|
| Curva de aprendizaje | Suave, muy buena DX | Algo más técnica |
| Ejecución | En el navegador, mismo loop que la app | Fuera del navegador, vía protocolo |
| Multi-pestaña / multi-origen | Limitado (mejoró, pero incómodo) | Nativo y cómodo |
| Navegadores | Chromium, Firefox, WebKit (experimental) | Chromium, Firefox, WebKit completos |
| Time-travel visual | Excelente, integrado | Vía trace viewer |
| Lenguaje de tests | JS / TS | JS, TS, Python, Java, .NET |
| Velocidad en CI | Buena | Generalmente más rápida |

> Resumen honesto: si tu app es una SPA de un solo origen y valoras una DX visual excelente, **Cypress** brilla. Si necesitas multi-pestaña, multi-dominio o varios lenguajes, **Playwright** encaja mejor. En este curso usamos Cypress.

---

## Paso 1 — Instalar Cypress

Asumimos un proyecto **React 19.2** creado con **Vite 8** y **TypeScript 5.9**. Abre una terminal **dentro de la carpeta del proyecto** y ejecuta:

```bash
# Instalar Cypress como dependencia de desarrollo (versión ^14)
npm i -D cypress
```

Cypress se instala como binario **local** del proyecto. **No** lo instales globalmente: así cada proyecto fija su propia versión (en este curso, `^14`).

Verifica la versión instalada antes de seguir:

```bash
# Mostrar la versión de Cypress del proyecto
npx cypress version
```

Si ves un número de versión `14.x`, vas bien. Si Cypress no se reconoce, revisa que estás dentro de la carpeta del proyecto y que `node_modules` existe.

---

## Paso 2 — Abrir el runner y generar `cypress/`

La primera vez que abres Cypress, él mismo crea la estructura de carpetas por ti. Ejecútalo:

```bash
# Abre la interfaz interactiva (Launchpad)
npx cypress open
```

Se abrirá el **Launchpad**, donde eliges entre:

- **E2E Testing** — pruebas de extremo a extremo (lo nuestro en módulos 1-8 y 10).
- **Component Testing** — pruebas de componentes aislados (módulo 9).

Elige **E2E Testing**. Cypress te ofrecerá crear los archivos de configuración por ti: **acepta**. A continuación elige el navegador y verás una pantalla para crear tu primer spec.

> No cierres la ventana: la dejaremos abierta para ver en vivo cada cambio de los siguientes pasos.

---

## Paso 3 — Entender la estructura que genera

Tras el primer arranque, tu proyecto tendrá una carpeta `cypress/`. Tómate un momento para ubicar cada pieza:

```
mi-todo-app/
├─ cypress/
│  ├─ e2e/              ← aquí viven tus tests  (*.cy.ts)
│  ├─ fixtures/         ← datos de prueba en JSON (ej. todos.json)
│  │  └─ example.json
│  └─ support/
│     ├─ commands.ts    ← comandos personalizados (módulo 7)
│     └─ e2e.ts         ← se ejecuta antes de cada spec
├─ cypress.config.ts    ← configuración global
├─ src/
├─ package.json
└─ tsconfig.json
```

| Carpeta / archivo | Para qué sirve |
|-------------------|----------------|
| `cypress/e2e/` | Tus archivos de test, con extensión `.cy.ts` |
| `cypress/fixtures/` | Datos estáticos (JSON) que reutilizas en los tests |
| `cypress/support/commands.ts` | Comandos personalizados como `cy.login()` |
| `cypress/support/e2e.ts` | Código que corre **antes de cada** spec (imports globales) |
| `cypress.config.ts` | Configuración: `baseUrl`, viewport, timeouts, etc. |

### 🔧 Tu turno #2

Sin mirar abajo: si quieres crear un test llamado `login.cy.ts`, **¿en qué carpeta** debe ir? Y si quieres guardar una lista de tareas de ejemplo en un JSON para reutilizarla, **¿dónde** la pondrías?

<details><summary>✅ Solución</summary>

- El test `login.cy.ts` va en **`cypress/e2e/`** (es donde Cypress busca specs por defecto, según el `specPattern`).
- El JSON de datos (por ejemplo `todos.json`) va en **`cypress/fixtures/`**, y luego lo cargas con `cy.fixture('todos')`.

Si pusieras el test fuera de `cypress/e2e/`, Cypress no lo encontraría a menos que cambies el `specPattern` (lo configuramos en el paso 4).

</details>

---

## Paso 4 — Escribir `cypress.config.ts`

Este archivo es el corazón de la configuración. Lo más importante para empezar es definir el `baseUrl`, así podrás escribir `cy.visit('/login')` en lugar de la URL completa. Abre el `cypress.config.ts` que generó Cypress y déjalo así:

```ts
// cypress.config.ts
import { defineConfig } from 'cypress'

export default defineConfig({
  e2e: {
    // URL base de la app servida por Vite
    baseUrl: 'http://localhost:5173',

    // Patrón de los archivos de test
    specPattern: 'cypress/e2e/**/*.cy.{ts,tsx}',

    // Tamaño de la ventana del navegador durante los tests
    viewportWidth: 1280,
    viewportHeight: 720,

    setupNodeEvents(on, config) {
      // Aquí registraremos plugins y tareas más adelante
      return config
    },
  },
})
```

Con `baseUrl` definido, estas dos líneas son equivalentes:

```ts
// cypress/e2e/demo.cy.ts
cy.visit('http://localhost:5173/login') // sin baseUrl
cy.visit('/login')                        // con baseUrl (recomendado)
```

> Si cambias el puerto de Vite, solo actualizas el `baseUrl` en un único lugar.

---

## Paso 5 — Scripts de npm y TypeScript para Cypress

Para no escribir los comandos largos a mano, añade dos scripts a tu `package.json`:

```json
// package.json (fragmento)
{
  "scripts": {
    "dev": "vite",
    "cypress:open": "cypress open",
    "cypress:run": "cypress run"
  }
}
```

| Script | Qué hace | Cuándo usarlo |
|--------|----------|---------------|
| `npm run cypress:open` | Abre la interfaz interactiva con time-travel | Desarrollo, depurar tests |
| `npm run cypress:run` | Ejecuta todos los tests en modo *headless* (sin ventana) | CI, ejecución completa |

Flujo típico de trabajo (¡recuérdalo, lo usarás todo el curso!):

```bash
# Terminal 1: levanta la app (debe responder en localhost:5173)
npm run dev

# Terminal 2: abre Cypress en modo interactivo
npm run cypress:open
```

> Cypress **no** levanta tu app por ti. La app debe estar corriendo **antes** de visitar su URL.

Ahora, para que el editor reconozca los tipos de `cy`, `Cypress`, `describe` e `it`, crea un `tsconfig.json` **dentro** de la carpeta `cypress/`:

```json
// cypress/tsconfig.json
{
  "compilerOptions": {
    "target": "ES2021",
    "lib": ["ES2021", "DOM"],
    "types": ["cypress", "node"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "skipLibCheck": true,
    "isolatedModules": false
  },
  "include": ["**/*.ts"]
}
```

La clave es `"types": ["cypress", "node"]`: le dice a TypeScript que cargue las definiciones globales de Cypress. Así, al escribir `cy.` en el editor obtendrás autocompletado y verificación de tipos.

---

## Paso 6 — Tu primer `cy.visit` en verde

Con todo configurado, vamos a comprobar que el entorno funciona de punta a punta. Crea un spec mínimo:

```ts
// cypress/e2e/smoke.cy.ts
describe('Smoke test', () => {
  it('carga la página de login', () => {
    // Visita /login usando el baseUrl configurado en el paso 4
    cy.visit('/login')

    // El botón de login debe ser visible: si lo es, todo el setup funciona
    cy.get('[data-cy=login-button]').should('be.visible')
  })
})
```

Con la app corriendo (`npm run dev`) y Cypress abierto (`npm run cypress:open`), haz click en `smoke.cy.ts` dentro del runner. Si el test corre en **verde**, tu entorno está listo para el resto del curso.

> Aún no entendemos cada comando (`cy.get`, `should`); eso es el módulo 2. Por ahora solo confirmamos que la tubería completa funciona.

---

### Prueba esto

- Crea un proyecto React + Vite + TypeScript y ejecuta `npm i -D cypress`.
- Lanza `npx cypress open`, elige **E2E Testing** y deja que Cypress genere la carpeta `cypress/`.
- Abre `cypress.config.ts` y confirma que `baseUrl` apunta a `http://localhost:5173`.
- Añade los scripts `cypress:open` y `cypress:run` a tu `package.json`.
- Crea `cypress/tsconfig.json` con `"types": ["cypress", "node"]` y comprueba que el editor autocompleta `cy.`.
- Levanta la app con `npm run dev` y verifica que abre en el puerto `5173`.
- Cambia temporalmente `baseUrl` a un puerto incorrecto (`5174`) y observa cómo `smoke.cy.ts` falla al no poder visitar la app; luego restáuralo.

---

## Ejercicios propuestos

### Básico

**B1 — Cambia el viewport.** Modifica `viewportWidth`/`viewportHeight` en `cypress.config.ts` a `375 × 667` (tamaño móvil), reabre el runner y observa cómo la app se renderiza en formato móvil.

**B2 — Segundo smoke.** Crea `cypress/e2e/home.cy.ts` que haga `cy.visit('/todos')` y compruebe que el input `[data-cy=new-todo-input]` es visible.

### Intermedio

**I1 — `specPattern` propio.** Cambia el `specPattern` para que Cypress también busque tests en una carpeta `cypress/integration/**/*.cy.ts`. Crea allí un spec y verifica que el runner lo lista.

**I2 — Variable de entorno.** Investiga `env` en `defineConfig` y define `env: { user: 'admin' }`. Léela en un test con `Cypress.env('user')` e imprímela con `cy.log()`.

### Avanzado

**A1 — Levantar app y tests con un comando.** Investiga el paquete `start-server-and-test` y crea un script `e2e` en `package.json` que levante `npm run dev` y, cuando `localhost:5173` responda, lance `cypress run` automáticamente.

**A2 — Dos navegadores.** Ejecuta `npm run cypress:run --browser chrome` y luego `--browser electron`. Compara los tiempos y anota qué diferencias notas en la salida de consola.

---

## Resumen del módulo 1

- El **testing E2E** prueba la app como un usuario real, dentro de un **navegador real**.
- La **pirámide de testing**: muchas unitarias, algunas de integración, pocas E2E.
- **Cypress** ejecuta los tests en el mismo loop que la app y ofrece **time-travel** visual.
- Frente a **Playwright**, Cypress destaca en DX para SPAs de un solo origen.
- **Paso 1:** se instala con `npm i -D cypress` (local, versión `^14`).
- **Paso 2:** `npx cypress open` genera `cypress/e2e`, `cypress/fixtures` y `cypress/support`.
- **Paso 4:** en `cypress.config.ts` definimos `baseUrl: 'http://localhost:5173'` y el `specPattern`.
- **Paso 5:** añadimos scripts `cypress:open` (interactivo) y `cypress:run` (headless/CI), y habilitamos TypeScript con `cypress/tsconfig.json` y `"types": ["cypress"]`.
- **Paso 6:** un `smoke.cy.ts` con `cy.visit('/login')` en verde confirma que el entorno está listo.

> **Siguiente página →** Módulo 2: Tu primer test E2E — anatomía de un spec, `cy.visit`, `cy.get`, `cy.contains` y el Test Runner con time-travel.
