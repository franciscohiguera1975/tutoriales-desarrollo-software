# Testing E2E en React con Cypress — Página 10
## Módulo 10 · Buenas prácticas y CI
### Organizar specs, evitar anti-patrones, capturar fallos y ejecutar Cypress en integración continua

---

## Cómo trabajar este módulo

Este módulo es **incremental** y de cierre: iremos endureciendo la suite que ya
tienes **paso a paso**, sin reescribirla de golpe. Mantén la app en
`http://localhost:5173` y Cypress abierto para verificar cada cambio. Aplica
**un paso a la vez**, ejecuta `npx cypress run` cuando toque y resuelve los retos
**🔧 Tu turno** **antes** de mirar la solución.

```
Paso 1  →  organizar specs por dominio
Paso 2  →  cazar y arreglar anti-patrones
Paso 3  →  garantizar independencia (estado limpio)
Paso 4  →  capturas y vídeo automáticos
Paso 5  →  ejecución headless con cypress run
Paso 6  →  integración continua con GitHub Actions
```

Ya sabes escribir tests E2E completos (módulo 8) y de componentes (módulo 9). Lo que separa una suite que *funciona en tu máquina* de una suite **profesional** es la disciplina: cómo organizas los archivos, qué errores evitas y cómo la ejecutas automáticamente en cada *push*. Eso es lo que cierra este módulo.

```
   código limpio  +  tests independientes  +  CI verde
        │                    │                   │
        └──────── suite confiable y mantenible ──┘
```

---

## Paso 1 — Organizar specs y carpetas

Una estructura clara hace que cualquiera encuentre (y mantenga) los tests. Reorganiza tus specs así:

```
cypress/
├── e2e/
│   ├── auth/
│   │   └── login.cy.ts          # tests de autenticación
│   └── todos/
│       ├── crear-tareas.cy.ts
│       ├── filtrar-tareas.cy.ts
│       └── flujo-completo.cy.ts # recorrido del módulo 8
├── fixtures/
│   └── todos.json               # datos de ejemplo
├── support/
│   ├── commands.ts              # cy.login, etc.
│   ├── e2e.ts                   # config global de E2E
│   └── component.ts             # cy.mount para Component Testing
src/
└── components/
    ├── TodoItem.tsx
    └── TodoItem.cy.tsx          # Component Testing junto al componente
```

| Carpeta | Contiene | Patrón de archivo |
|---------|----------|-------------------|
| `cypress/e2e/` | Tests de extremo a extremo | `*.cy.ts` |
| `cypress/fixtures/` | Datos estáticos (JSON) | `*.json` |
| `cypress/support/` | Comandos y configuración global | `commands.ts`, `e2e.ts` |
| `src/components/` | Component Tests junto al código | `*.cy.tsx` |

Convenciones útiles:

- **Un spec por funcionalidad**, no un único archivo gigante.
- Nombra los specs por *qué prueban* (`filtrar-tareas.cy.ts`), no por su número.
- Agrupa por dominio (`auth/`, `todos/`) cuando la app crece.

---

## Paso 2 — Cazar y arreglar anti-patrones

Estos cuatro errores son los que más dolor causan en suites reales. Revisa tu código y corrígelos uno a uno.

### 1. Selectores frágiles

```ts
// MAL: depende de clases de CSS y estructura del DOM que cambian a menudo.
cy.get('.MuiButton-root.css-1q2w3e > span').click()

// BIEN: un atributo data-cy dedicado al testing, estable ante refactors visuales.
cy.get('[data-cy=add-button]').click()
```

Los `data-cy` son un *contrato* explícito entre el código y los tests. Sobreviven a cambios de estilos, de framework de UI y de jerarquía del DOM.

### 2. `cy.wait()` con tiempo fijo

```ts
// MAL: espera ciega; lenta si sobra, frágil ("flaky") si el backend tarda más.
cy.get('[data-cy=add-button]').click()
cy.wait(3000)
cy.get('[data-cy=todo-item]').should('have.length', 1)

// BIEN: espera dirigida a la petición real con alias (módulo 6).
cy.intercept('POST', '/api/todos').as('crearTodo')
cy.get('[data-cy=add-button]').click()
cy.wait('@crearTodo') // espera exactamente hasta que la API responde
cy.get('[data-cy=todo-item]').should('have.length', 1)
```

Y recuerda: las aserciones de Cypress ya **reintentan automáticamente**. Casi nunca necesitas un `wait` numérico.

### 🔧 Tu turno #1

Tienes este fragmento "flaky" que usa `cy.wait(2000)`. Reescríbelo para que espere a la petición real `GET /api/todos` mediante `cy.intercept` + alias, eliminando la espera fija:

```ts
// Antes (frágil):
cy.visit('/todos')
cy.wait(2000)
cy.get('[data-cy=todo-item]').should('have.length', 3)
```

<details><summary>✅ Solución</summary>

```ts
// Después (robusto): interceptamos ANTES de visitar.
cy.intercept('GET', '/api/todos').as('cargarTodos')
cy.visit('/todos')
cy.wait('@cargarTodos') // espera exactamente a que la API responda
cy.get('[data-cy=todo-item]').should('have.length', 3)
```

Clave: el `cy.intercept` debe declararse **antes** de `cy.visit`, porque la petición se dispara al cargar la página. El alias `@cargarTodos` espera lo justo, ni más ni menos.

</details>

### 3. Tests dependientes entre sí

```ts
// MAL: el test 2 asume que el test 1 ya creó la tarea. Si test 1 falla
// o se ejecuta solo, test 2 también revienta.
it('crea una tarea', () => { /* crea "Comprar pan" */ })
it('elimina la tarea creada antes', () => { /* asume que existe */ })

// BIEN: cada test parte de un estado conocido y se basta a sí mismo.
beforeEach(() => {
  cy.login('admin', '1234')
  cy.visit('/todos')
})
it('elimina una tarea', () => {
  cy.get('[data-cy=new-todo-input]').type('Comprar pan')
  cy.get('[data-cy=add-button]').click()
  cy.get('[data-cy=delete-button]').first().click()
  cy.get('[data-cy=todo-item]').should('have.length', 0)
})
```

Un test debe poder ejecutarse **solo y en cualquier orden**. Cypress incluso reinicia el estado del navegador entre tests para forzar esa independencia.

### 4. Login por la UI en cada test

```ts
// MAL: repetir el formulario de login en cada beforeEach (lento).
beforeEach(() => {
  cy.visit('/login')
  cy.get('[data-cy=username]').type('admin')
  cy.get('[data-cy=password]').type('1234')
  cy.get('[data-cy=login-button]').click()
})

// BIEN: cy.session() del módulo 8 cachea la sesión y la restaura al instante.
beforeEach(() => {
  cy.login('admin', '1234') // envuelto en cy.session()
  cy.visit('/todos')
})
```

| Anti-patrón | Síntoma | Remedio |
|-------------|---------|---------|
| Selectores frágiles | Tests rotos al cambiar el CSS | Atributos `data-cy` |
| `cy.wait(ms)` fijo | Lentitud o *flakiness* | `cy.intercept` + `cy.wait('@alias')` |
| Tests dependientes | Fallos en cascada / al cambiar el orden | `beforeEach` con estado limpio |
| Login por UI repetido | Suite lenta | `cy.session()` |

---

## Paso 3 — Garantizar independencia y estado limpio

Cypress, entre cada test, limpia cookies y `localStorage` por defecto. Para garantizar datos limpios en el backend, siembra el estado vía API en lugar de hacerlo por la UI:

```ts
// cypress/e2e/todos/crear-tareas.cy.ts
beforeEach(() => {
  // Reseteamos el backend a un estado conocido (endpoint de test).
  cy.request('POST', '/api/test/reset')
  cy.login('admin', '1234')
  cy.visit('/todos')
})
```

> Sembrar por API es más rápido y robusto que "clicar" para preparar datos. Reserva los clics para *lo que estás probando*.

### 🔧 Tu turno #2

Tienes dos tests dependientes: el primero crea "Comprar pan" y el segundo asume que ya existe para borrarla. Reescríbelos en **un único spec** donde el segundo test sea **autosuficiente** (cree su propia tarea), usando un `beforeEach` que reinicie el backend y haga login.

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/todos/gestionar.cy.ts
describe('Gestión de tareas', () => {
  beforeEach(() => {
    cy.request('POST', '/api/test/reset') // backend limpio
    cy.login('admin', '1234')             // sesión cacheada (módulo 8)
    cy.visit('/todos')
  })

  it('crea una tarea', () => {
    cy.get('[data-cy=new-todo-input]').type('Comprar pan')
    cy.get('[data-cy=add-button]').click()
    cy.get('[data-cy=todo-item]').should('have.length', 1)
  })

  it('elimina una tarea (que él mismo crea)', () => {
    // No depende del test anterior: prepara su propio estado.
    cy.get('[data-cy=new-todo-input]').type('Comprar pan')
    cy.get('[data-cy=add-button]').click()
    cy.get('[data-cy=delete-button]').first().click()
    cy.get('[data-cy=todo-item]').should('have.length', 0)
  })
})
```

Cada `it` parte de un estado limpio (reset + login) y se basta a sí mismo. Puedes ejecutarlos en cualquier orden, o uno solo con `--spec`, y siempre pasan.

</details>

---

## Paso 4 — Capturas y vídeo automáticos en fallos

En modo headless, Cypress puede capturar evidencia sin que tú hagas nada. Añade estas opciones a la configuración E2E:

```ts
// cypress.config.ts
import { defineConfig } from 'cypress'

export default defineConfig({
  e2e: {
    baseUrl: 'http://localhost:5173',
    // Graba un vídeo de cada spec (útil para depurar en CI).
    video: true,
    // Captura un screenshot automáticamente cuando un test falla.
    screenshotOnRunFailure: true,
    setupNodeEvents() {},
  },
})
```

- Las **capturas de fallo** se guardan en `cypress/screenshots/`.
- Los **vídeos** se guardan en `cypress/videos/`.
- En CI, sube ambos como *artifacts* para inspeccionar fallos sin reproducirlos localmente.

También puedes capturar manualmente en un punto concreto:

```ts
// Captura puntual con nombre, útil para estados intermedios.
cy.screenshot('lista-con-tres-tareas')
```

---

## Paso 5 — Ejecución headless con `cypress run`

`cypress open` abre el runner interactivo (para desarrollar tests). En CI usamos `cypress run`, que ejecuta **sin interfaz**, más rápido, y produce los códigos de salida que el pipeline necesita.

```bash
# Ejecuta TODOS los specs de E2E en modo headless.
npx cypress run

# Solo un spec concreto.
npx cypress run --spec "cypress/e2e/todos/flujo-completo.cy.ts"

# En un navegador específico.
npx cypress run --browser chrome

# Component Testing en headless.
npx cypress run --component
```

Conviene añadir scripts a `package.json`:

```json
// package.json (fragmento)
{
  "scripts": {
    "dev": "vite",
    "cy:open": "cypress open",
    "cy:run": "cypress run",
    "cy:component": "cypress run --component"
  }
}
```

---

## Paso 6 — Integración en CI con GitHub Actions

La acción oficial `cypress-io/github-action` instala dependencias, arranca tu app, espera a que esté lista y ejecuta los tests. Crea este workflow para que se ejecute en cada *push* y *pull request*:

```yaml
# .github/workflows/cypress.yml
name: Tests E2E con Cypress

on:
  push:
    branches: [main]
  pull_request:

jobs:
  cypress-run:
    runs-on: ubuntu-latest
    steps:
      # 1. Clonamos el repositorio.
      - name: Checkout
        uses: actions/checkout@v4

      # 2. Preparamos Node 20.
      - name: Configurar Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      # 3. La acción oficial instala deps, levanta la app y corre Cypress.
      - name: Ejecutar Cypress
        uses: cypress-io/github-action@v6
        with:
          # Arranca el dev-server de Vite…
          start: npm run dev
          # …y espera a que responda antes de lanzar los tests.
          wait-on: 'http://localhost:5173'
          wait-on-timeout: 120
          browser: chrome

      # 4. Si algo falla, subimos capturas y vídeos como evidencia.
      - name: Subir capturas en fallo
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: cypress-screenshots
          path: cypress/screenshots

      - name: Subir vídeos
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: cypress-videos
          path: cypress/videos
```

Flujo del pipeline:

```
push ──▶ checkout ──▶ setup-node ──▶ instala deps
        ──▶ npm run dev ──▶ wait-on :5173 ──▶ cypress run
        ──▶ ¿falló? ──▶ sube screenshots + vídeos
```

> Para acelerar suites grandes, `cypress-io/github-action` soporta **paralelización** y *grabación* en Cypress Cloud con la opción `record: true` y una `record-key`.

---

## Cypress vs Playwright

Ambos son excelentes para E2E. Esta comparación te ayuda a elegir con criterio:

| Criterio | Cypress | Playwright |
|----------|---------|------------|
| Modelo de ejecución | Dentro del navegador (mismo runloop) | Fuera del navegador (protocolo DevTools) |
| Lenguajes | JS / TS | JS/TS, Python, Java, C# |
| Multi-pestaña / multi-origen | Limitado | Nativo y robusto |
| Navegadores | Chrome, Edge, Firefox, Electron | Chromium, Firefox, WebKit (Safari) |
| Runner interactivo / time-travel | Excelente, muy visual | Buen *trace viewer* (post-mortem) |
| Component Testing | Sí (módulo 9) | Sí (experimental/estable según versión) |
| Paralelización | Vía Cypress Cloud o sharding | Integrada y gratuita |
| Curva de aprendizaje | Suave, muy didáctica | Algo mayor, más flexible |

Cuándo elegir cada uno:

- **Cypress**: equipos centrados en JS/TS que valoran un *developer experience* excepcional, depuración visual y una curva suave. Ideal si tu app es de un solo origen.
- **Playwright**: cuando necesitas **WebKit/Safari**, escenarios **multi-pestaña o multi-dominio**, paralelización gratuita potente o tests en varios lenguajes.

> No hay una respuesta universal. Para la mayoría de apps React de un solo dominio, Cypress es una elección sobresaliente; Playwright gana en escenarios de navegador más complejos.

---

### Prueba esto

- Reorganiza tus specs en subcarpetas por dominio (`auth/`, `todos/`) y verifica que `cypress run` los sigue encontrando.
- Busca en tu suite cualquier `cy.wait(número)` y reemplázalo por `cy.intercept` + `cy.wait('@alias')`.
- Ejecuta un único test con `--spec` para comprobar que es **independiente** y pasa por sí solo.
- Añade el workflow de GitHub Actions, haz un *push* y observa el job en la pestaña Actions.
- Provoca un fallo a propósito y descarga el *artifact* de screenshots desde la ejecución de CI.
- Activa `video: true`, corre `cypress run` localmente y revisa el vídeo generado en `cypress/videos/`.

---

## Ejercicios propuestos

### Básico

**B1 — Scripts npm.** Añade a `package.json` un script `cy:run:chrome` que ejecute `cypress run --browser chrome` y otro `cy:spec` que reciba un spec concreto. Pruébalos desde la terminal.

**B2 — Renombrar por dominio.** Toma un spec mal nombrado (p. ej. `test1.cy.ts`) y renómbralo según *qué prueba* (`filtrar-tareas.cy.ts`), moviéndolo a la subcarpeta de dominio que corresponda.

### Intermedio

**I1 — Cazar el wait fijo.** Busca en tu suite un `cy.wait(ms)` numérico, reemplázalo por una intercepción con alias y mide la diferencia de tiempo en `cypress run`. Anota en un comentario por qué la versión nueva es más robusta.

**I2 — Estado limpio por API.** Añade un `beforeEach` que use `cy.request('POST', '/api/test/reset')` antes del login en uno de tus specs de `todos/` y verifica que el test arranca siempre con la lista vacía, sin importar el orden de ejecución.

### Avanzado

**A1 — Workflow con Component Testing.** Amplía el workflow de GitHub Actions para que, además del E2E, ejecute el Component Testing (`npx cypress run --component`) como un *step* o *job* separado. Asegúrate de que ambos suben sus artifacts.

**A2 — Matriz de navegadores.** Configura una `strategy.matrix` en GitHub Actions para correr la suite E2E en `chrome` y en `firefox`, y comprueba que ambos jobs aparecen en la pestaña Actions tras un push.

---

## Resumen del módulo 10

- Organiza los specs **por funcionalidad y dominio**, con nombres descriptivos y comandos compartidos en `support/`.
- Evita los cuatro anti-patrones clave: **selectores frágiles**, **`cy.wait` fijo**, **tests dependientes** y **login por UI repetido**.
- Cada test debe ser **independiente**: parte de un estado limpio (siembra por API con `cy.request`) y pasa en cualquier orden.
- Configura **`screenshotOnRunFailure`** y **`video`** para tener evidencia automática de los fallos.
- Usa **`cypress run`** (headless) en CI y `cypress open` para desarrollar; añade scripts a `package.json`.
- Integra en CI con **`cypress-io/github-action@v6`**, usando `start` + `wait-on` y subiendo capturas/vídeos como *artifacts*.
- **Cypress vs Playwright**: Cypress brilla en *developer experience* y apps de un solo origen; Playwright en multi-navegador (Safari), multi-pestaña y paralelización gratuita.

---

> **Fin del tutorial.** A lo largo de estos diez módulos has recorrido Cypress de principio a fin: desde la introducción y el primer test (01-02), pasando por selectores, interacción y aserciones (03-05), esperas y red (06), comandos y fixtures (07), hasta un flujo completo de login → CRUD (08), Component Testing con `cy.mount()` (09) y las buenas prácticas y CI de este módulo (10). Ahora tienes las herramientas para escribir suites E2E rápidas, robustas y mantenibles.
>
> **Un último consejo: Cypress no trabaja solo.** Los tests E2E son pocos por diseño —cubren los recorridos críticos—, pero el grueso de la confianza diaria viene de los tests unitarios y de integración. Por eso te recomendamos **combinar este tutorial con el de Vitest** que encontrarás en la carpeta **`react-vitest/`**: usa **Vitest + Testing Library** para la base de la pirámide (componentes, hooks y lógica, rápidos y baratos) y **Cypress** para los pocos recorridos E2E de extremo a extremo y el Component Testing visual. Juntos cubren toda la pirámide de tests y te dan una red de seguridad completa para tu aplicación React. ¡A testear!
