# Testing E2E en React con Cypress — Página 7

## Módulo 7 · Comandos personalizados y fixtures

### Reutiliza lógica con `Cypress.Commands.add`, tipa tus comandos y carga datos con `cy.fixture`

---

## Cómo trabajar este módulo

Este módulo es **incremental**: construiremos `cy.login()`, lo tiparemos y lo
combinaremos con fixtures **paso a paso**. Trabaja con dos ventanas:

- La app en `http://localhost:5173` (rutas `/login` y `/todos`, login `admin` / `1234`).
- Cypress con `npx cypress open` en modo E2E, para ver tus comandos correr en vivo.

Escribe **un archivo/bloque por paso** y guarda. No copies todo de golpe: cada
pieza (comando, tipo, fixture) tiene su momento. Entre paso y paso hay retos
**🔧 Tu turno** que resolverás **antes** de mirar la solución.

```
Paso 1  →  el problema: login repetido
Paso 2  →  crear cy.login()
Paso 3  →  tipar el comando (index.d.ts)
Paso 4  →  usar cy.login() en varios tests
Paso 5  →  tú creas cy.loginRapidoMock()
Paso 6  →  fixtures con cy.fixture()
Paso 7  →  combinar fixtures + intercept
Paso 8  →  tú preparas el estado en beforeEach
```

Estructura final de la carpeta `cypress/`:

```
   cypress/
   ├── e2e/
   │   └── *.cy.ts
   ├── fixtures/
   │   └── todos.json          ← datos de prueba
   └── support/
       ├── commands.ts         ← implementación de comandos
       ├── e2e.ts              ← importa commands.ts
       └── index.d.ts          ← tipos de los comandos
```

---

## Paso 1 — El problema: repetir el login en cada test

A medida que crece tu suite, empiezas a repetir los mismos pasos: hacer login, preparar datos, navegar a `/todos`. Mira este código duplicado en varios `.cy.ts`:

```ts
// Repetido una y otra vez... ❌
cy.visit('/login')
cy.get('[data-cy="username"]').type('admin')
cy.get('[data-cy="password"]').type('1234')
cy.get('[data-cy="login-button"]').click()
cy.url().should('include', '/todos')
```

Si mañana cambia un selector o el flujo de login, tendrás que editar **todos** los archivos. La solución: un comando personalizado `cy.login()`.

En este módulo eliminaremos la duplicación con dos herramientas clave: **comandos personalizados** (`Cypress.Commands.add`) y **fixtures** (`cy.fixture`). Y, como usamos TypeScript 5.9, **tiparemos** los comandos para tener autocompletado y seguridad de tipos.

---

## Paso 2 — Crear un comando personalizado: `cy.login()`

Los comandos personalizados se definen en `cypress/support/commands.ts`. Este archivo se carga automáticamente gracias a `cypress/support/e2e.ts`.

```ts
// cypress/support/commands.ts

// Comando para iniciar sesión a través de la UI
Cypress.Commands.add('login', (username = 'admin', password = '1234') => {
  cy.visit('/login')
  cy.get('[data-cy="username"]').clear().type(username)
  cy.get('[data-cy="password"]').clear().type(password)
  cy.get('[data-cy="login-button"]').click()
  // Verificamos que el login fue exitoso antes de devolver el control
  cy.url().should('include', '/todos')
})
```

Asegúrate de que `cypress/support/e2e.ts` importe el archivo de comandos:

```ts
// cypress/support/e2e.ts

// Carga los comandos personalizados en todos los specs
import './commands'
```

---

## Paso 3 — Tipar el comando en `index.d.ts`

Con TypeScript, `cy.login()` daría un error de tipo porque Cypress no lo conoce. Lo solucionamos **ampliando** la interfaz `Chainable` mediante *declaration merging*.

```ts
// cypress/support/index.d.ts

declare namespace Cypress {
  interface Chainable {
    /**
     * Inicia sesión a través de la UI.
     * @param username usuario (por defecto 'admin')
     * @param password contraseña (por defecto '1234')
     * @example cy.login()
     * @example cy.login('admin', '1234')
     */
    login(username?: string, password?: string): Chainable<void>
  }
}
```

Asegúrate de que tu `tsconfig.json` de Cypress incluya este archivo de tipos:

```json
// cypress/tsconfig.json

{
  "compilerOptions": {
    "target": "ES2021",
    "lib": ["ES2021", "DOM"],
    "types": ["cypress", "node"]
  },
  "include": ["**/*.ts", "support/index.d.ts"]
}
```

---

## Paso 4 — Usar el comando en varios tests

Ahora cualquier spec puede usar `cy.login()` con una sola línea, y con autocompletado de TypeScript.

```ts
// cypress/e2e/uso-comando-login.cy.ts
describe('Reutilizando cy.login()', () => {
  it('entra y ve la lista de todos', () => {
    cy.login() // ✓ una línea, tipado y reutilizable
    cy.get('[data-cy="todo-item"]').should('exist')
  })

  it('falla con credenciales incorrectas', () => {
    cy.visit('/login')
    cy.get('[data-cy="username"]').type('admin')
    cy.get('[data-cy="password"]').type('mala')
    cy.get('[data-cy="login-button"]').click()
    cy.contains('Credenciales inválidas').should('be.visible')
  })
})
```

### 🔧 Tu turno #1

Sin mirar abajo: **usa `cy.login('admin', '1234')` pasando argumentos explícitos** en un nuevo `it`, y verifica que tras entrar el `[data-cy="new-todo-input"]` está visible. Como reto extra, confirma en tu editor que aparece el **autocompletado** del comando (gracias al tipado del paso 3).

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/uso-comando-login.cy.ts  (añade dentro del describe)
  it('entra con credenciales explícitas y puede crear todos', () => {
    cy.login('admin', '1234') // los parámetros se documentan en index.d.ts
    cy.get('[data-cy="new-todo-input"]').should('be.visible')
  })
```

Si el tipado está bien hecho, al escribir `cy.login(` el editor te muestra la firma `login(username?: string, password?: string)` y el JSDoc.

</details>

---

## Paso 5 — 🔧 Tu turno #2: crear `cy.loginRapidoMock()`

El login real puede ser lento. En módulos anteriores usamos un `cy.loginRapidoMock()` que **stubea** la red para tests más veloces y deterministas. Toca crearlo.

**Tu reto:**
1. En `commands.ts`, añade un comando `loginRapidoMock` que intercepte `POST /api/login` respondiendo `200` con un token falso (`{ token: 'fake-jwt-token', user: { username: 'admin' } }`, alias `@login`), luego visite `/login`, escriba `admin` / `1234`, pulse el botón y verifique que la URL incluye `/todos`.
2. Tipa el comando en `index.d.ts`.

<details><summary>✅ Solución</summary>

```ts
// cypress/support/commands.ts
Cypress.Commands.add('loginRapidoMock', () => {
  // Stub del login: respondemos sin backend
  cy.intercept('POST', '/api/login', {
    statusCode: 200,
    body: { token: 'fake-jwt-token', user: { username: 'admin' } },
  }).as('login')

  cy.visit('/login')
  cy.get('[data-cy="username"]').type('admin')
  cy.get('[data-cy="password"]').type('1234')
  cy.get('[data-cy="login-button"]').click()
  cy.url().should('include', '/todos')
})
```

```ts
// cypress/support/index.d.ts  (añade dentro de Chainable)
    /**
     * Inicia sesión rápido mockeando la red (sin backend real).
     * @example cy.loginRapidoMock()
     */
    loginRapidoMock(): Chainable<void>
```

| Comando | Backend | Velocidad | Uso |
| --- | --- | --- | --- |
| `cy.login()` | Real | Más lento | Verificar login real |
| `cy.loginRapidoMock()` | Stub | Rápido | Tests centrados en todos |

</details>

---

## Paso 6 — Fixtures: cargar datos con `cy.fixture()`

Una **fixture** es un archivo (normalmente JSON) con datos de prueba que vive en `cypress/fixtures/`. Es la fuente de verdad de tus mocks: separar los datos del código hace los tests más limpios y mantenibles.

Creamos `cypress/fixtures/todos.json`:

```json
// cypress/fixtures/todos.json

[
  { "id": 1, "title": "Comprar pan", "completed": false },
  { "id": 2, "title": "Pasear al perro", "completed": false },
  { "id": 3, "title": "Leer un libro", "completed": true }
]
```

Cargarla en un test es tan simple como:

```ts
// cypress/e2e/usar-fixture.cy.ts
it('carga datos desde una fixture', () => {
  cy.fixture('todos.json').then((todos) => {
    // todos es el array parseado del JSON
    expect(todos).to.have.length(3)
    expect(todos[0]).to.have.property('title', 'Comprar pan')
  })
})
```

> Cypress busca las fixtures relativas a `cypress/fixtures/`, así que basta con el nombre del archivo (`'todos.json'` o incluso `'todos'`).

---

## Paso 7 — Combinar fixtures con `cy.intercept()`

Aquí está la combinación más potente del módulo: mockear `GET /api/todos` **usando la fixture como respuesta**. Así los datos del mock están centralizados en un único archivo.

```
   cypress/fixtures/todos.json
            │  (datos)
            ▼
   cy.intercept('GET', '/api/todos', { fixture: 'todos.json' })
            │
            ▼
   La UI renderiza los 3 todos del archivo
```

Forma 1: atajo con la propiedad `fixture`.

```ts
// cypress/e2e/intercept-con-fixture.cy.ts
describe('Fixtures + intercept', () => {
  it('mockea la lista usando la fixture (atajo)', () => {
    // Cypress carga el JSON y lo usa como body de la respuesta
    cy.intercept('GET', '/api/todos', { fixture: 'todos.json' }).as('getTodos')

    cy.loginRapidoMock()
    cy.visit('/todos')
    cy.wait('@getTodos')

    cy.get('[data-cy="todo-item"]').should('have.length', 3)
    cy.get('[data-cy="todo-item"]').first().should('contain', 'Comprar pan')
  })
})
```

Forma 2: cargar primero la fixture y reutilizarla (útil si necesitas los datos en aserciones).

```ts
// cypress/e2e/intercept-con-fixture-2.cy.ts
it('mockea y luego valida contra los mismos datos', () => {
  cy.fixture('todos.json').then((todos) => {
    cy.intercept('GET', '/api/todos', { body: todos }).as('getTodos')

    cy.loginRapidoMock()
    cy.visit('/todos')
    cy.wait('@getTodos')

    // Validamos la UI contra el número real de elementos del fixture
    cy.get('[data-cy="todo-item"]').should('have.length', todos.length)
  })
})
```

> Tip: puedes tener varias fixtures (`todos-vacio.json`, `todos-muchos.json`) para probar distintos escenarios sin tocar el código del test.

---

## Paso 8 — 🔧 Tu turno #3: preparar el estado en `beforeEach`

`beforeEach` ejecuta un bloque de preparación **antes de cada test**. Es el lugar ideal para combinar comando de login + intercept + fixture, dejando cada `it` enfocado solo en su comportamiento.

**Tu reto:** crea un `describe` con un `beforeEach` que (1) mockee el GET con `{ fixture: 'todos.json' }` (alias `@getTodos`), (2) stubee el POST respondiendo `201`, (3) haga `cy.loginRapidoMock()`, visite `/todos` y espere `@getTodos`. Después escribe tres `it` cortos: uno que verifique que parte de 3 todos, otro que agregue una tarea y compruebe el POST, y otro que marque el primer todo como completado.

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/preparar-estado.cy.ts
describe('CRUD de todos con estado preparado', () => {
  beforeEach(() => {
    // 1. Mockeamos la lista inicial con la fixture
    cy.intercept('GET', '/api/todos', { fixture: 'todos.json' }).as('getTodos')

    // 2. Stub del POST para no depender del backend
    cy.intercept('POST', '/api/todos', (req) => {
      req.reply({
        statusCode: 201,
        body: { id: 99, title: req.body.title, completed: false },
      })
    }).as('createTodo')

    // 3. Login rápido y navegación
    cy.loginRapidoMock()
    cy.visit('/todos')
    cy.wait('@getTodos')
  })

  it('parte de 3 todos cargados desde la fixture', () => {
    cy.get('[data-cy="todo-item"]').should('have.length', 3)
  })

  it('agrega un nuevo todo y envía el POST', () => {
    cy.get('[data-cy="new-todo-input"]').type('Tarea nueva')
    cy.get('[data-cy="add-button"]').click()

    cy.wait('@createTodo').its('request.body').should('have.property', 'title', 'Tarea nueva')
    cy.get('[data-cy="todo-item"]').should('contain', 'Tarea nueva')
  })

  it('marca un todo como completado', () => {
    cy.get('[data-cy="todo-item"]')
      .first()
      .find('[data-cy="todo-checkbox"]')
      .check()
      .should('be.checked')
  })
})
```

Observa cómo el `beforeEach` deja cada test corto y legible: la preparación se escribe **una vez**.

| Hook | Cuándo se ejecuta |
| --- | --- |
| `before` | Una vez, antes de todos los tests del bloque |
| `beforeEach` | Antes de **cada** test |
| `afterEach` | Después de **cada** test |
| `after` | Una vez, al final del bloque |

</details>

---

## Bonus — Comandos que devuelven un valor encadenable

Un comando puede devolver un subject para seguir encadenando:

```ts
// cypress/support/commands.ts

// Ejemplo de comando con valor de retorno encadenable
Cypress.Commands.add('agregarTodo', (titulo: string) => {
  cy.get('[data-cy="new-todo-input"]').clear().type(titulo)
  cy.get('[data-cy="add-button"]').click()
  // Devolvemos el elemento recién creado para poder encadenar
  return cy.get('[data-cy="todo-item"]').contains(titulo)
})
```

Y su tipo correspondiente:

```ts
// cypress/support/index.d.ts

declare namespace Cypress {
  interface Chainable {
    login(username?: string, password?: string): Chainable<void>
    loginRapidoMock(): Chainable<void>
    /** Agrega un todo y devuelve su elemento. */
    agregarTodo(titulo: string): Chainable<JQuery<HTMLElement>>
  }
}
```

### Buenas prácticas de comandos y fixtures

- **Un comando, una responsabilidad**: `cy.login()` solo loguea; no mezcles con navegación a otras rutas.
- **Tipa siempre** tus comandos en `index.d.ts` para aprovechar el autocompletado.
- **Centraliza los datos** en fixtures; evita arrays gigantes incrustados en cada test.
- **Nombra los alias con sentido**: `@getTodos`, `@createTodo`, `@login`.
- **Usa `beforeEach`** para el estado común, pero no escondas ahí aserciones específicas de un test.

---

### Prueba esto

- Crea `cy.login()` y reemplaza el login manual en al menos dos specs anteriores.
- Tipa `cy.login()` en `index.d.ts` y verifica que aparece el autocompletado en tu editor.
- Crea `cypress/fixtures/todos.json` con 3 todos y mockea el GET con `{ fixture: 'todos.json' }`.
- Añade una segunda fixture `todos-vacio.json` (`[]`) y prueba el estado "Sin tareas".
- Mueve el login y el intercept a un `beforeEach` y comprueba que los tests siguen pasando.
- Crea un comando `cy.agregarTodo(titulo)` y úsalo para agregar dos tareas en un mismo test.

---

## Ejercicios propuestos

### Básico

**B1 — Reemplazar login manual.** Toma un spec del módulo 4 con login manual y sustitúyelo por `cy.login()`. Verifica que sigue pasando.

**B2 — Leer una fixture.** Crea `todos-vacio.json` con `[]` y escribe un test que la cargue con `cy.fixture('todos-vacio')` y afirme que su longitud es 0.

### Intermedio

**I1 — Fixture como respuesta.** Mockea el GET con `{ fixture: 'todos.json' }`, entra con `cy.loginRapidoMock()` y verifica que el primer y el último todo contienen los textos esperados.

**I2 — Comando con parámetros.** Crea `cy.agregarTodo(titulo)` (con su tipo) y úsalo dos veces seguidas para agregar "Tarea A" y "Tarea B"; comprueba con `have.length` que la lista creció en dos.

### Avanzado

**A1 — Escenarios por fixture.** Crea `todos-muchos.json` (8 todos). Escribe dos tests que, cambiando solo la fixture del intercept, verifiquen la longitud correcta (3 vs 8) **sin modificar** el cuerpo del test más allá del nombre de la fixture.

**A2 — Comando compuesto.** Crea un comando `cy.prepararTodos(fixtureName)` que combine el intercept del GET con la fixture indicada + `cy.loginRapidoMock()` + `cy.visit('/todos')` + `cy.wait('@getTodos')`. Tipa el comando y úsalo en un `beforeEach`.

---

## Resumen del módulo 7

- Los **comandos personalizados** (`Cypress.Commands.add`) eliminan la duplicación de flujos como el login.
- En TypeScript, **tipa** tus comandos en `cypress/support/index.d.ts` ampliando `Cypress.Chainable`.
- `cy.login()` y `cy.loginRapidoMock()` cubren el login real y el stubbeado respectivamente.
- Las **fixtures** (`cy.fixture`) centralizan los datos de prueba en archivos JSON.
- Combinar `cy.intercept()` con `{ fixture: 'todos.json' }` produce mocks limpios y mantenibles.
- `beforeEach` prepara el estado común y deja cada test corto y enfocado.
- Lo construiste **por pasos**: del problema del login repetido, a crear y tipar comandos, a combinarlos con fixtures que tú mismo preparaste en un `beforeEach`.

> **Siguiente página →** Módulo 8: Flujo completo (login → CRUD) — uniremos comandos, intercepts y fixtures en un recorrido end-to-end real.
