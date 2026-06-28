# Testing E2E en React con Cypress — Página 8
## Módulo 8 · Flujo completo (login → CRUD)
### Un test de extremo a extremo realista: autenticarse, gestionar tareas y acelerar con sesiones

---

## Cómo trabajar este módulo

Este módulo es **incremental**: vamos a construir el recorrido completo de la app
**paso a paso**, no de un solo golpe. Ten la app corriendo en
`http://localhost:5173` y Cypress abierto (`npx cypress open`) para ver cada
cambio reflejado al instante. Añade **un bloque por paso**, guarda y observa cómo
el test crece en el *Command Log*. Entre paso y paso hay retos **🔧 Tu turno**
que debes resolver **antes** de mirar la solución.

```
Paso 1  →  recordamos cy.login del módulo 7
Paso 2  →  primer borrador: login + crear + completar
Paso 3  →  añadimos filtros
Paso 4  →  añadimos borrado y logout
Paso 5  →  dividimos en varios it con beforeEach
Paso 6  →  aceleramos con cy.session()
```

Recordemos la app de ejemplo antes de empezar:

- Rutas `/login` y `/todos`.
- Login válido: usuario **admin**, password **1234**.
- API `/api/todos` (GET para listar, POST para crear).
- Selectores `data-cy`: `username`, `password`, `login-button`, `new-todo-input`, `add-button`, `todo-item`, `todo-checkbox`, `delete-button`, `filter-all`, `filter-active`, `filter-completed`.

```
   /login              /todos
  ┌────────┐         ┌──────────────────────────┐
  │ login  │  ────▶  │ agregar → completar →     │
  │ admin  │ redirec │ filtrar → eliminar → salir │
  └────────┘         └──────────────────────────┘
```

Un test E2E "feliz" como este es la prueba de mayor valor de toda la suite: si pasa, sabemos que la integración real entre frontend, enrutado y API funciona. Por eso vale la pena escribirlo con cuidado.

---

## Paso 1 — Recordamos el comando `cy.login` del módulo 7

En el módulo 7 creamos un comando personalizado para no repetir el formulario de login en cada test. Lo recuperamos aquí porque será la **base** del flujo. Si no lo tienes, asegúrate de que esté en tu archivo de soporte:

```ts
// cypress/support/commands.ts
// Comando reutilizable que rellena el formulario de /login y envía.
Cypress.Commands.add('login', (username = 'admin', password = '1234') => {
  cy.visit('/login')
  cy.get('[data-cy=username]').type(username)
  cy.get('[data-cy=password]').type(password)
  cy.get('[data-cy=login-button]').click()
  // Esperamos a que la redirección a /todos se complete.
  cy.url().should('include', '/todos')
})
```

Para que TypeScript reconozca el comando, declaramos su firma:

```ts
// cypress/support/index.d.ts
// Ampliamos el namespace de Cypress con nuestro comando personalizado.
declare namespace Cypress {
  interface Chainable {
    /** Inicia sesión rellenando el formulario de /login. */
    login(username?: string, password?: string): Chainable<void>
  }
}
```

Con `cy.login()` disponible, ya podemos empezar a construir el recorrido.

---

## Paso 2 — Primer borrador: login + crear + completar

Empecemos por la versión más directa y legible. Crea el spec con **un solo `it`** que, de momento, haga login, cree tareas y marque una como completada:

```ts
// cypress/e2e/flujo-completo.cy.ts
describe('Flujo completo Todo (login → CRUD)', () => {
  it('un usuario inicia sesión y gestiona tareas', () => {
    // 1. Login usando el comando del módulo 7.
    cy.login('admin', '1234')

    // 2. Verificamos la redirección y que la página de todos está lista.
    cy.url().should('include', '/todos')
    cy.get('[data-cy=new-todo-input]').should('be.visible')

    // 3. Agregamos varias tareas.
    const tareas = ['Comprar pan', 'Estudiar Cypress', 'Pasear al perro']
    tareas.forEach((texto) => {
      cy.get('[data-cy=new-todo-input]').type(texto)
      cy.get('[data-cy=add-button]').click()
    })

    // 4. La lista debe contener exactamente 3 elementos.
    cy.get('[data-cy=todo-item]').should('have.length', 3)

    // 5. Marcamos la primera tarea como completada.
    cy.get('[data-cy=todo-item]')
      .first()
      .find('[data-cy=todo-checkbox]')
      .check()
      .should('be.checked')
  })
})
```

Guarda y ejecuta el spec desde Cypress. Verás el recorrido completo en el *Command Log* y podrás "viajar en el tiempo" por cada comando. Este borrador ya prueba login + creación + completado, pero el recorrido aún no incluye filtros, borrado ni logout. Vamos a añadirlos.

---

## Paso 3 — Añadimos los filtros (all / active / completed)

La app tiene tres botones de filtro. Tras marcar una tarea como completada, podemos verificar que cada filtro muestra el subconjunto correcto. Añade este fragmento **dentro del mismo `it`**, justo después del paso 5:

```ts
// cypress/e2e/flujo-completo.cy.ts (continúa dentro del it)
// 6. Filtro "active": solo deben quedar las tareas NO completadas (2).
cy.get('[data-cy=filter-active]').click()
cy.get('[data-cy=todo-item]').should('have.length', 2)

// 7. Filtro "completed": solo la tarea completada (1).
cy.get('[data-cy=filter-completed]').click()
cy.get('[data-cy=todo-item]').should('have.length', 1)
cy.get('[data-cy=todo-item]')
  .first()
  .should('contain.text', 'Comprar pan')

// 8. Filtro "all": vuelven a verse las 3.
cy.get('[data-cy=filter-all]').click()
cy.get('[data-cy=todo-item]').should('have.length', 3)
```

| Filtro | Selector | Resultado esperado (con 3 tareas, 1 completada) |
|--------|----------|--------------------------------------------------|
| Todas | `filter-all` | 3 elementos |
| Activas | `filter-active` | 2 elementos |
| Completadas | `filter-completed` | 1 elemento |

> Fíjate en que **no afirmamos índices fijos** del DOM, sino la *cantidad* y el *contenido*. Es más robusto frente a reordenamientos.

### 🔧 Tu turno #1

Antes de seguir con el borrado, **escribe tú la aserción del filtro "active"** verificando no solo la cantidad, sino que la tarea completada ("Comprar pan") **NO aparece** mientras el filtro "active" está activo. Pista: usa `cy.contains('[data-cy=todo-item]', 'Comprar pan').should('not.exist')`.

<details><summary>✅ Solución</summary>

```ts
// Tras hacer clic en filter-active:
cy.get('[data-cy=filter-active]').click()
cy.get('[data-cy=todo-item]').should('have.length', 2)
// La tarea completada no debe verse en el filtro de activas.
cy.contains('[data-cy=todo-item]', 'Comprar pan').should('not.exist')
```

`should('not.exist')` reintenta hasta que el elemento desaparece del DOM. Es la forma robusta de afirmar ausencia, mucho mejor que un `cy.wait()` ciego.

</details>

---

## Paso 4 — Eliminar una tarea y cerrar sesión

Cada `todo-item` incluye un botón `delete-button`. Borramos una y verificamos que la lista decrece. Añade este fragmento al final del `it`, asegurándote de estar en el filtro "all":

```ts
// cypress/e2e/flujo-completo.cy.ts (continúa dentro del it)
// 9. Volvemos al filtro "all" y eliminamos la primera tarea.
cy.get('[data-cy=filter-all]').click()
cy.get('[data-cy=todo-item]').should('have.length', 3)

cy.get('[data-cy=todo-item]')
  .first()
  .find('[data-cy=delete-button]')
  .click()

// 10. Ahora deben quedar 2 tareas y "Comprar pan" ya no debe existir.
cy.get('[data-cy=todo-item]').should('have.length', 2)
cy.contains('[data-cy=todo-item]', 'Comprar pan').should('not.exist')
```

Cerrar la sesión nos devuelve a `/login`. Si tu app tiene un botón `logout-button`, el cierre del flujo es:

```ts
// cypress/e2e/flujo-completo.cy.ts (continúa dentro del it)
// 11. Logout: debe regresar a /login.
cy.get('[data-cy=logout-button]').click()
cy.url().should('include', '/login')
cy.get('[data-cy=login-button]').should('be.visible')
```

Con esto el recorrido es completo: **login → crear → completar → filtrar → eliminar → logout**.

---

## Paso 5 — Dividir el spec en varios `it` con `beforeEach`

Mantener todo en un único `it` enorme funciona, pero dificulta localizar fallos: si el test rompe en el paso 9, Cypress solo te dice que "el test falló", no *qué parte*. Dividir en varios `it` con un `beforeEach` común mejora la legibilidad y aísla los fallos.

```ts
// cypress/e2e/flujo-completo.cy.ts
describe('Gestión de tareas autenticada', () => {
  beforeEach(() => {
    // Antes de cada test partimos de una sesión iniciada en /todos.
    cy.login('admin', '1234')
    cy.url().should('include', '/todos')
  })

  it('agrega varias tareas', () => {
    const tareas = ['Comprar pan', 'Estudiar Cypress', 'Pasear al perro']
    tareas.forEach((texto) => {
      cy.get('[data-cy=new-todo-input]').type(texto)
      cy.get('[data-cy=add-button]').click()
    })
    cy.get('[data-cy=todo-item]').should('have.length', 3)
  })

  it('marca una tarea como completada', () => {
    cy.get('[data-cy=new-todo-input]').type('Comprar pan')
    cy.get('[data-cy=add-button]').click()
    cy.get('[data-cy=todo-checkbox]').first().check().should('be.checked')
  })

  it('filtra entre activas y completadas', () => {
    cy.get('[data-cy=new-todo-input]').type('Tarea activa')
    cy.get('[data-cy=add-button]').click()
    cy.get('[data-cy=new-todo-input]').type('Tarea completada')
    cy.get('[data-cy=add-button]').click()
    cy.get('[data-cy=todo-checkbox]').last().check()

    cy.get('[data-cy=filter-active]').click()
    cy.get('[data-cy=todo-item]').should('have.length', 1)

    cy.get('[data-cy=filter-completed]').click()
    cy.get('[data-cy=todo-item]').should('have.length', 1)
  })
})
```

Cada test es **independiente**: no depende del estado dejado por el anterior. Esa independencia es una de las reglas de oro que veremos a fondo en el módulo 10.

### 🔧 Tu turno #2

Añade un cuarto `it` a este `describe` que pruebe el **borrado**: crea una tarea "Tarea a borrar", elimínala y verifica que la lista queda vacía (`have.length', 0`). Recuerda que el `beforeEach` ya te deja autenticado en `/todos`.

<details><summary>✅ Solución</summary>

```ts
  it('elimina una tarea recién creada', () => {
    // Arrange: creamos una tarea.
    cy.get('[data-cy=new-todo-input]').type('Tarea a borrar')
    cy.get('[data-cy=add-button]').click()
    cy.get('[data-cy=todo-item]').should('have.length', 1)

    // Act: la eliminamos.
    cy.get('[data-cy=todo-item]')
      .first()
      .find('[data-cy=delete-button]')
      .click()

    // Assert: la lista queda vacía.
    cy.get('[data-cy=todo-item]').should('have.length', 0)
  })
```

Observa que este test **no depende** de los anteriores: crea su propia tarea y la borra. Podría ejecutarse solo y seguiría pasando.

</details>

---

## Paso 6 — El problema del login repetido y la solución `cy.session()`

Con `beforeEach`, **cada** test vuelve a visitar `/login`, escribe usuario y contraseña, espera la redirección... y todo eso cuesta segundos. Con 3 tests no se nota; con 50, tu suite tarda minutos solo en autenticarse una y otra vez.

```
Sin sesión cacheada:
test 1  [login 1.2s][trabajo 0.8s]
test 2  [login 1.2s][trabajo 0.6s]
test 3  [login 1.2s][trabajo 0.9s]
        └── 3.6s desperdiciados en login ──┘
```

La solución es **`cy.session()`**. Cachea el estado del navegador (cookies, `localStorage`, `sessionStorage`) bajo una clave. La primera vez ejecuta el bloque de login real; las siguientes **restaura** el estado guardado sin volver a pasar por la UI.

```ts
// cypress/support/commands.ts
// Versión optimizada: envuelve el login real dentro de cy.session().
Cypress.Commands.add('login', (username = 'admin', password = '1234') => {
  cy.session(
    // 1. Clave de caché: distinta sesión por usuario.
    [username, password],
    // 2. Bloque "setup": solo se ejecuta si la sesión no está cacheada.
    () => {
      cy.visit('/login')
      cy.get('[data-cy=username]').type(username)
      cy.get('[data-cy=password]').type(password)
      cy.get('[data-cy=login-button]').click()
      cy.url().should('include', '/todos')
    },
    {
      // 3. validate: comprueba que la sesión restaurada sigue siendo válida.
      validate() {
        cy.window().its('localStorage.token').should('exist')
      },
      cacheAcrossSpecs: true, // Reutiliza la sesión entre archivos .cy.ts.
    },
  )
})
```

Ahora, en el spec, tras `cy.login()` debemos **visitar la página protegida explícitamente**, porque `cy.session()` no navega por sí mismo:

```ts
// cypress/e2e/flujo-completo.cy.ts
describe('Gestión de tareas autenticada (con sesión)', () => {
  beforeEach(() => {
    cy.login('admin', '1234')
    // cy.session() restaura cookies/localStorage, pero NO navega:
    cy.visit('/todos')
  })

  it('parte de /todos sin repetir el formulario de login', () => {
    cy.url().should('include', '/todos')
    cy.get('[data-cy=new-todo-input]').should('be.visible')
  })
})
```

```
Con cy.session():
test 1  [login real 1.2s][trabajo 0.8s]   ← cachea
test 2  [restore 0.05s][trabajo 0.6s]
test 3  [restore 0.05s][trabajo 0.9s]
        └── login real solo UNA vez ──┘
```

| Enfoque | Login por test | Velocidad | Cuándo usarlo |
|---------|----------------|-----------|---------------|
| `cy.login()` por UI en cada `beforeEach` | Sí, vía formulario | Lenta | Pocos tests, o cuando *quieres* probar el login |
| `cy.login()` + `cy.session()` | Solo la primera vez | Rápida | La mayoría de suites autenticadas |
| Sembrar `localStorage` directamente | Nunca pasa por UI | Muy rápida | Cuando el login no es lo que pruebas |

> Mantén **al menos un test** que ejercite el login real por la UI (sin sesión cacheada). Así garantizas que el formulario de autenticación sigue funcionando.

---

## El spec final, ordenado

Juntando todo, un spec limpio y rápido queda así. Úsalo solo para **comparar** con lo que construiste, no para copiarlo de golpe:

```ts
// cypress/e2e/flujo-completo.cy.ts
describe('Todo App · recorrido autenticado completo', () => {
  beforeEach(() => {
    cy.login('admin', '1234') // cacheado con cy.session()
    cy.visit('/todos')
  })

  it('crea, completa, filtra y elimina tareas', () => {
    // Crear
    ;['Comprar pan', 'Estudiar Cypress', 'Pasear al perro'].forEach((t) => {
      cy.get('[data-cy=new-todo-input]').type(t)
      cy.get('[data-cy=add-button]').click()
    })
    cy.get('[data-cy=todo-item]').should('have.length', 3)

    // Completar la primera
    cy.get('[data-cy=todo-checkbox]').first().check().should('be.checked')

    // Filtrar
    cy.get('[data-cy=filter-active]').click()
    cy.get('[data-cy=todo-item]').should('have.length', 2)
    cy.get('[data-cy=filter-all]').click()

    // Eliminar la primera
    cy.get('[data-cy=todo-item]').first().find('[data-cy=delete-button]').click()
    cy.get('[data-cy=todo-item]').should('have.length', 2)
  })

  it('mantiene el formulario de alta tras recargar', () => {
    cy.reload()
    cy.get('[data-cy=new-todo-input]').should('be.visible')
  })
})
```

---

### Prueba esto

- Convierte tu test monolítico del primer borrador en varios `it` independientes y comprueba que el reporte de fallos es más claro.
- Añade `cy.session()` a `cy.login` y mide el tiempo total de la suite antes y después con `cypress run`.
- Activa la *Command Log* en el modo interactivo (`cypress open`) y observa cómo la primera ejecución dice "session created" y las siguientes "session restored".
- Crea un test que intercepte `POST /api/todos` con `cy.intercept` y verifique que el cuerpo enviado contiene el texto de la tarea.
- Escribe un test de logout que confirme que, al volver a `/todos` sin sesión, la app redirige a `/login`.
- Rompe a propósito el `validate()` (afirma un token inexistente) y observa cómo Cypress vuelve a ejecutar el login real.

---

## Ejercicios propuestos

### Básico

**B1 — Tarea única.** Escribe un `it` que, partiendo del `beforeEach` con sesión, cree una sola tarea "Leer documentación" y verifique con `contain.text` que aparece en la lista.

**B2 — Contador tras crear.** Crea dos tareas y afirma que `[data-cy=todo-item]` tiene `have.length', 2`. Luego marca una y confirma que el total sigue siendo 2 (completar no borra).

### Intermedio

**I1 — Filtro completas vacío.** Crea tres tareas sin completar ninguna, pulsa `filter-completed` y verifica que la lista queda en `have.length', 0`. Después pulsa `filter-all` y confirma que vuelven las 3.

**I2 — Borrado por contenido.** Crea las tareas "A", "B" y "C". Usando `cy.contains('[data-cy=todo-item]', 'B').find('[data-cy=delete-button]').click()`, borra solo "B" y verifica que "A" y "C" siguen existiendo.

### Avanzado

**A1 — Login real más sesión.** Escribe **un** spec con dos bloques: uno con un `it` que haga login por la UI real (sin `cy.session`) verificando la pantalla de login, y otro `describe` con `beforeEach` usando `cy.login()` cacheado. Comprueba que la cobertura del formulario se mantiene y la suite sigue siendo rápida.

**A2 — Persistencia entre recargas.** Con `cy.session()` activo, crea una tarea, ejecuta `cy.reload()` y verifica si la tarea persiste o no según cómo la app guarde el estado. Documenta en comentarios qué comportamiento observas y por qué.

---

## Resumen del módulo 8

- Un **test de flujo completo** recorre login → CRUD → logout y es la prueba de mayor valor de la suite; lo construimos **por pasos** en lugar de un bloque gigante.
- Reutilizamos el comando **`cy.login`** del módulo 7 para no repetir el formulario en cada spec.
- Verificamos la **redirección** con `cy.url().should('include', '/todos')` antes de operar.
- Los **filtros** se prueban afirmando *cantidad* y *contenido*, no índices frágiles del DOM; la ausencia con `should('not.exist')`.
- Dividir el recorrido en varios `it` con un **`beforeEach`** común mejora la legibilidad y aísla los fallos.
- **`cy.session()`** cachea cookies y `localStorage`, ejecuta el login real una sola vez y acelera enormemente la suite; recuerda **`cy.visit()`** después, porque la sesión no navega.
- Conserva siempre **un test que ejercite el login real por la UI** para no dejar de cubrir esa pantalla.

> **Siguiente página →** Módulo 9: Component Testing en Cypress — montar componentes React aislados con `cy.mount()`, espiar callbacks con `cy.stub()` y decidir entre Component Testing, E2E y Vitest+RTL.
