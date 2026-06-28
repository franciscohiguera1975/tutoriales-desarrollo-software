# Testing E2E en React con Cypress — Página 6

## Módulo 6 · Esperas y red (cy.intercept)

### Controla las peticiones HTTP: espía, mockea y simula errores sin depender del backend

---

## Cómo trabajar este módulo

Este módulo es **incremental**: construiremos los intercepts **uno por paso**,
añadiendo capacidades poco a poco. Trabaja con dos ventanas:

- La app en `http://localhost:5173` (rutas `/login` y `/todos`, login `admin` / `1234`).
- Cypress con `npx cypress open` en modo E2E. Abre el panel de red para ver las
  peticiones interceptadas en tiempo real.

Escribe **un bloque por paso**, guarda y observa cómo cambia la app cuando tú
controlas la red. Entre paso y paso hay retos **🔧 Tu turno** que resolverás
**antes** de mirar la solución.

```
Paso 1  →  por qué evitar cy.wait(tiempo)
Paso 2  →  mockear GET /api/todos
Paso 3  →  alias .as() + cy.wait('@alias')
Paso 4  →  tú verificas el POST al agregar
Paso 5  →  simular un error 500
Paso 6  →  stubbing del login
Paso 7  →  tú stubeas un login fallido (401)
```

> Nota: usamos `cy.loginRapidoMock()` para entrar logueado. Lo crearemos en el módulo 7; por ahora trátalo como "entrar a /todos ya autenticado".

---

## El modelo mental: reintento vs esperas fijas

En el módulo anterior descubriste que las aserciones de Cypress **se reintentan**. Esa propiedad cambia por completo la forma de manejar la asincronía: en Cypress casi **nunca** necesitas esperas fijas.

```
   ACCIÓN (click)  →  la app hace fetch a /api/todos  →  la UI se actualiza
                          (tarda X ms, impredecible)

   cy.get('[data-cy="todo-item"]').should('have.length', 4)
        └── reintenta cada ~50ms hasta que se cumpla o expire (4s)
```

Como las aserciones implícitas reintentan, Cypress **espera lo justo**: si la red tarda 80 ms, espera 80 ms; si tarda 900 ms, espera 900 ms. No hay tiempos mágicos que ajustar.

| Enfoque | Problema / ventaja |
| --- | --- |
| `cy.wait(3000)` | Lento si la red es rápida; **falla** si la red es lenta |
| `cy.get(...).should(...)` | Espera **solo** lo necesario; robusto |
| `cy.wait('@alias')` | Espera al evento de red exacto; determinista |

En este módulo daremos un paso más: aprenderemos a **interceptar la red** con `cy.intercept()` para espiar, mockear y simular errores.

---

## Paso 1 — Por qué evitar `cy.wait(tiempo)`

`cy.wait(numero)` introduce una **espera fija** en milisegundos. Es una mala práctica salvo casos muy puntuales (animaciones, debounce). Sus problemas:

- **Tests frágiles**: si el entorno es más lento (CI, red cargada), el tiempo se queda corto y el test falla.
- **Tests lentos**: si pones 3000 ms "por si acaso", pagas ese coste en cada ejecución.
- **Esconden bugs**: ocultan condiciones de carrera reales en lugar de exponerlas.

```ts
// cypress/e2e/anti-patron.cy.ts

// MAL: espera fija y frágil
it('agrega un todo (anti-patrón)', () => {
  cy.get('[data-cy="add-button"]').click()
  cy.wait(2000) // ❌ adivinanza de tiempo
  cy.get('[data-cy="todo-item"]').should('have.length', 4)
})

// BIEN: deja que la aserción reintente
it('agrega un todo (correcto)', () => {
  cy.get('[data-cy="add-button"]').click()
  cy.get('[data-cy="todo-item"]').should('have.length', 4) // ✓ reintenta
})
```

> Regla de oro: en lugar de **esperar tiempo**, espera un **evento** (`cy.wait('@alias')`) o un **estado** (`.should(...)`).

---

## Paso 2 — Mockear GET `/api/todos` con una respuesta fija

`cy.intercept()` es el comando estrella del módulo. Intercepta peticiones HTTP que coincidan con un patrón y permite **observarlas** (spy) o **responderlas** (mock/stub).

```ts
// Solo espiar (deja pasar la petición real, pero la observamos)
cy.intercept('GET', '/api/todos')

// Mockear (respondemos nosotros, NO llega al backend)
cy.intercept('GET', '/api/todos', { body: [...] })
```

| Modo | Llega al backend | Uso típico |
| --- | --- | --- |
| Spy | Sí | Verificar que se hizo una petición |
| Stub / Mock | No | Devolver datos fijos sin backend |

Empecemos por lo más útil: devolver una lista de todos controlada por nosotros. Así el test es **determinista** (siempre los mismos datos) y **rápido** (sin backend).

```ts
// cypress/e2e/mock-get-todos.cy.ts
describe('Mock de GET /api/todos', () => {
  it('renderiza la lista que devuelve el mock', () => {
    // Definimos la respuesta ANTES de visitar la página
    cy.intercept('GET', '/api/todos', {
      statusCode: 200,
      body: [
        { id: 1, title: 'Tarea mockeada A', completed: false },
        { id: 2, title: 'Tarea mockeada B', completed: true },
      ],
    })

    cy.loginRapidoMock()
    cy.visit('/todos')

    // La UI debe mostrar exactamente los 2 todos del mock
    cy.get('[data-cy="todo-item"]').should('have.length', 2)
    cy.get('[data-cy="todo-item"]').first().should('contain', 'Tarea mockeada A')
    cy.get('[data-cy="todo-item"]').last().should('contain', 'Tarea mockeada B')
  })
})
```

> Importante: declara el `cy.intercept()` **antes** de la acción que dispara la petición (`cy.visit()`). Cypress registra el interceptor y luego captura la llamada.

### 🔧 Tu turno #1

Sin mirar abajo: **mockea el GET con una lista vacía** (`body: []`) y verifica que **no existe** ningún `[data-cy="todo-item"]` y que aparece el mensaje de estado vacío ("Sin tareas").

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/mock-lista-vacia.cy.ts
it('muestra el estado vacío cuando no hay todos', () => {
  cy.intercept('GET', '/api/todos', { body: [] }).as('getTodos')

  cy.loginRapidoMock()
  cy.visit('/todos')
  cy.wait('@getTodos')

  cy.get('[data-cy="todo-item"]').should('not.exist')
  cy.contains('Sin tareas').should('be.visible')
})
```

Con un `body: []` controlas por completo el escenario "lista vacía" sin tocar el backend.

</details>

---

## Paso 3 — Aliasing con `.as()` y `cy.wait('@alias')`

Para **esperar a que ocurra una petición concreta** (sin tiempos mágicos), le damos un alias con `.as()` y luego usamos `cy.wait('@alias')`.

```
   cy.intercept('GET', '/api/todos').as('getTodos')
                                         │
   cy.visit('/todos')   ──── dispara ───┘
                                         │
   cy.wait('@getTodos') ◄── espera al EVENTO exacto, no a un tiempo
```

```ts
// cypress/e2e/alias-wait.cy.ts
describe('Aliasing y cy.wait(@alias)', () => {
  it('espera a que termine GET /api/todos antes de afirmar', () => {
    // Espiamos (sin mockear) y damos alias
    cy.intercept('GET', '/api/todos').as('getTodos')

    cy.loginRapidoMock()
    cy.visit('/todos')

    // Esperamos al evento de red exacto
    cy.wait('@getTodos').its('response.statusCode').should('eq', 200)

    // Ahora afirmamos sobre la UI ya cargada
    cy.get('[data-cy="todo-item"]').should('have.length.greaterThan', 0)
  })
})
```

`cy.wait('@getTodos')` devuelve el objeto de la interceptación, sobre el que puedes hacer aserciones de `request` y `response`.

| Propiedad | Qué contiene |
| --- | --- |
| `request.method` | `GET`, `POST`, ... |
| `request.body` | Cuerpo enviado |
| `request.url` | URL completa |
| `response.statusCode` | Código HTTP de respuesta |
| `response.body` | Cuerpo de la respuesta |

---

## Paso 4 — 🔧 Tu turno #2: verificar el POST al agregar

Al pulsar "Agregar", la app debe enviar `POST /api/todos` con el nuevo título.

**Tu reto:** en un `beforeEach`, mockea el GET con `{ body: [] }` (alias `@getTodos`) y espía el POST respondiendo `201` con el recurso creado (alias `@createTodo`). Luego, en un `it`, escribe "Comprar café", pulsa agregar y **afirma sobre la petición enviada**: que el método es `POST`, que el `body` tiene `title: 'Comprar café'` y que la respuesta fue `201`. Por último, comprueba que el todo aparece en la UI.

Pista: con `cy.intercept('POST', url, (req) => { ... })` puedes inspeccionar `req.body` y responder con `req.reply(...)`.

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/verificar-post.cy.ts
describe('POST /api/todos al agregar', () => {
  beforeEach(() => {
    // Mockeamos el GET inicial para partir de un estado conocido
    cy.intercept('GET', '/api/todos', { body: [] }).as('getTodos')
    // Espiamos el POST y devolvemos el recurso creado
    cy.intercept('POST', '/api/todos', (req) => {
      // Podemos inspeccionar y responder dinámicamente
      req.reply({
        statusCode: 201,
        body: { id: 99, title: req.body.title, completed: false },
      })
    }).as('createTodo')

    cy.loginRapidoMock()
    cy.visit('/todos')
    cy.wait('@getTodos')
  })

  it('envía POST con el título escrito', () => {
    cy.get('[data-cy="new-todo-input"]').type('Comprar café')
    cy.get('[data-cy="add-button"]').click()

    // Esperamos al POST y afirmamos sobre la petición enviada
    cy.wait('@createTodo').then((interception) => {
      expect(interception.request.method).to.equal('POST')
      expect(interception.request.body).to.have.property('title', 'Comprar café')
      expect(interception.response?.statusCode).to.equal(201)
    })

    // La UI refleja el nuevo todo
    cy.get('[data-cy="todo-item"]').should('contain', 'Comprar café')
  })
})
```

</details>

---

## Paso 5 — Mockear un error 500 y verificar el mensaje en la UI

Probar el "camino feliz" no basta: una buena suite también prueba **errores**. Con `cy.intercept()` simulamos un fallo del servidor y comprobamos que la UI muestra un mensaje adecuado.

```
   GET /api/todos  ──► [intercept]  ──►  responde 500
                                              │
                                              ▼
                          La app muestra: "Error al cargar las tareas"
```

```ts
// cypress/e2e/error-500.cy.ts
describe('Manejo de error del servidor', () => {
  it('muestra un mensaje de error cuando el GET falla', () => {
    // Forzamos un fallo del servidor
    cy.intercept('GET', '/api/todos', {
      statusCode: 500,
      body: { message: 'Internal Server Error' },
    }).as('getTodosError')

    cy.loginRapidoMock()
    cy.visit('/todos')

    cy.wait('@getTodosError')

    // La lista NO debe renderizarse
    cy.get('[data-cy="todo-item"]').should('not.exist')

    // Y debe aparecer un mensaje de error visible
    cy.contains('Error al cargar las tareas').should('be.visible')
  })

  it('muestra error cuando el POST falla', () => {
    cy.intercept('GET', '/api/todos', { body: [] }).as('getTodos')
    cy.intercept('POST', '/api/todos', {
      statusCode: 500,
      body: { message: 'No se pudo guardar' },
    }).as('createError')

    cy.loginRapidoMock()
    cy.visit('/todos')
    cy.wait('@getTodos')

    cy.get('[data-cy="new-todo-input"]').type('Tarea que falla')
    cy.get('[data-cy="add-button"]').click()

    cy.wait('@createError')
    cy.contains('No se pudo guardar la tarea').should('be.visible')
  })
})
```

> Consejo: añade `forceNetworkError: true` en lugar de un status para simular **caída total de red** (sin respuesta del servidor).

```ts
// Simula que la red falla por completo (sin respuesta)
cy.intercept('GET', '/api/todos', { forceNetworkError: true }).as('netDown')
```

---

## Paso 6 — Stubbing del login

El login real puede ser lento o requerir un backend. Para tests que se centran en los todos, conviene **stubbear** la autenticación y entrar directamente en estado "logueado".

```ts
// cypress/e2e/stub-login.cy.ts
describe('Login stubbeado', () => {
  it('autentica con respuesta mockeada y navega a /todos', () => {
    // Interceptamos la llamada de autenticación
    cy.intercept('POST', '/api/login', {
      statusCode: 200,
      body: { token: 'fake-jwt-token', user: { username: 'admin' } },
    }).as('login')

    // También mockeamos los todos para un entorno controlado
    cy.intercept('GET', '/api/todos', { body: [] }).as('getTodos')

    cy.visit('/login')
    cy.get('[data-cy="username"]').type('admin')
    cy.get('[data-cy="password"]').type('1234')
    cy.get('[data-cy="login-button"]').click()

    // Verificamos la petición de login y la navegación
    cy.wait('@login').its('request.body').should('deep.include', {
      username: 'admin',
      password: '1234',
    })

    cy.url().should('include', '/todos')
    cy.wait('@getTodos')
  })
})
```

> En el módulo 7 convertiremos este login stubbeado en un **comando personalizado** reutilizable (`cy.login()`).

---

## Paso 7 — 🔧 Tu turno #3: login fallido (401)

**Tu reto:** stubea `POST /api/login` para que responda `401` con `{ message: 'Credenciales inválidas' }` (alias `@loginFail`). Escribe usuario `admin` y password `clave-mala`, pulsa el botón y verifica que: aparece el mensaje "Credenciales inválidas" visible **y** la URL **sigue** en `/login` (no navega).

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/stub-login-fail.cy.ts
it('muestra error con credenciales inválidas (mock 401)', () => {
  cy.intercept('POST', '/api/login', {
    statusCode: 401,
    body: { message: 'Credenciales inválidas' },
  }).as('loginFail')

  cy.visit('/login')
  cy.get('[data-cy="username"]').type('admin')
  cy.get('[data-cy="password"]').type('clave-mala')
  cy.get('[data-cy="login-button"]').click()

  cy.wait('@loginFail')
  cy.contains('Credenciales inválidas').should('be.visible')
  cy.url().should('include', '/login') // no navega
})
```

Al controlar el status de la respuesta puedes probar la rama de error sin depender de un backend que devuelva 401.

</details>

---

## Patrones de URL en `cy.intercept()`

`cy.intercept()` acepta varias formas de coincidir con URLs. Aquí las más usadas:

| Patrón | Coincide con |
| --- | --- |
| `'/api/todos'` | Ruta exacta |
| `'**/api/todos'` | Cualquier host + ruta (glob) |
| `'/api/todos/*'` | `/api/todos/1`, `/api/todos/2`... |
| `{ method: 'POST', url: '/api/todos' }` | Objeto de configuración |
| `/\/api\/todos\/\d+/` | Expresión regular |

```ts
// cypress/e2e/patrones-url.cy.ts

// Objeto de configuración (legible cuando hay varias opciones)
cy.intercept({ method: 'GET', url: '**/api/todos' }, { body: [] }).as('getTodos')

// Comodín para IDs dinámicos (eliminar un todo concreto)
cy.intercept('DELETE', '/api/todos/*', { statusCode: 204 }).as('deleteTodo')
```

---

### Prueba esto

- Reemplaza un `cy.wait(2000)` por `cy.wait('@getTodos')` y compara la velocidad de la suite.
- Mockea `GET /api/todos` con una lista vacía y verifica que aparece el estado "Sin tareas".
- Inspecciona `interception.request.body` tras un POST y afirma que el título coincide.
- Simula un error 500 en el POST y comprueba que el mensaje de error es visible.
- Usa `forceNetworkError: true` y observa cómo reacciona la app a una caída total.
- Stubbea el login con un token falso y navega directo a `/todos` sin backend real.

---

## Ejercicios propuestos

### Básico

**B1 — Spy puro.** Espía `GET /api/todos` sin mockear (`cy.intercept('GET', '/api/todos').as('getTodos')`), visita `/todos` y verifica con `cy.wait('@getTodos')` que el `response.statusCode` es 200.

**B2 — Mock de dos todos.** Mockea el GET con exactamente dos todos y verifica con `have.length` que la UI muestra dos `[data-cy="todo-item"]`.

### Intermedio

**I1 — Verificar el body del POST.** Agrega un todo "Llamar al banco" y, esperando `@createTodo`, afirma que `request.body.title` es exactamente ese texto.

**I2 — Caída de red.** Usa `forceNetworkError: true` en el GET y verifica que la lista `not.exist` y que aparece un mensaje de error.

### Avanzado

**A1 — DELETE con comodín.** Mockea `DELETE /api/todos/*` con `204`, parte de una lista de 3 todos (GET mockeado), elimina uno y verifica con `cy.wait('@deleteTodo')` que la petición salió y que la lista baja a 2.

**A2 — Latencia controlada.** Investiga `delay` en la respuesta de `cy.intercept` (`{ body: [...], delay: 1000 }`). Mockea el GET con 1 s de retardo y demuestra que `.should('have.length', n)` **igual pasa** sin ningún `cy.wait(tiempo)` gracias al reintento.

---

## Resumen del módulo 6

- Gracias al **reintento de aserciones**, Cypress espera lo justo: evita `cy.wait(tiempo)`.
- Las esperas fijas hacen los tests **lentos y frágiles**; espera **eventos** o **estados**.
- `cy.intercept()` permite **espiar** (spy) y **mockear** (stub) peticiones HTTP.
- Con `.as('alias')` + `cy.wait('@alias')` esperas a una petición concreta de forma determinista.
- Puedes verificar el **método**, el **body** y el **status** de cada petición interceptada.
- Mockear **errores 500** y stubbear el **login** te dan tests rápidos, controlados y completos.
- Lo construiste **por pasos**: de mockear un GET, a verificar un POST, a stubbear un login fallido que tú mismo escribiste.

> **Siguiente página →** Módulo 7: Comandos personalizados y fixtures — crea `cy.login()`, tipa tus comandos y carga datos con `cy.fixture()`.
