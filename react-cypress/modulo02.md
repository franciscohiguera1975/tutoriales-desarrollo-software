# Testing E2E en React con Cypress — Página 2
## Módulo 2 · Anatomía de un spec
### Anatomía de un spec, visitar una página, leer el DOM y entender por qué los comandos de Cypress son una cola asíncrona

---

## Cómo trabajar este módulo

Este módulo es **incremental**: construiremos el spec de la página de login
**paso a paso**, añadiendo **un bloque por paso**. No copies el archivo final de
golpe; escribe cada `it`, guarda y mira el resultado en el runner.

Antes de empezar, ten preparado:

- La app corriendo en `http://localhost:5173` (`npm run dev`).
- Cypress abierto en modo interactivo (`npx cypress open`), con `login.cy.ts`
  seleccionado para ver **en vivo** cómo cada cambio se refleja al guardar.

Resuelve los retos **🔧 Tu turno** **antes** de abrir la solución colapsable.

```
Paso 1  →  el esqueleto del spec (describe + it)
Paso 2  →  cy.visit('/login') y beforeEach
Paso 3  →  seleccionar el DOM con cy.get
Paso 4  →  tu primera aserción .should
Paso 5  →  tú escribes el siguiente caso
```

---

## Anatomía de un spec

Un *spec* es un archivo de pruebas. En Cypress lo nombramos con la extensión `.cy.ts` y lo colocamos en `cypress/e2e/`. Su esqueleto es siempre el mismo:

```ts
// cypress/e2e/login.cy.ts
describe('Página de login', () => {
  it('muestra el formulario al entrar', () => {
    // pasos del test...
  })
})
```

| Función | Rol | Analogía |
|---------|-----|----------|
| `describe(nombre, fn)` | Agrupa tests relacionados | El "capítulo" |
| `it(nombre, fn)` | Define **un** test concreto | El "párrafo" |
| `beforeEach(fn)` | Código que corre antes de cada `it` | El "preámbulo" |

`describe` e `it` provienen del framework **Mocha**, que Cypress integra. Puedes anidar varios `describe` y tener varios `it` dentro de cada uno.

```ts
// cypress/e2e/login.cy.ts
describe('Autenticación', () => {
  describe('Formulario de login', () => {
    it('muestra el campo usuario', () => { /* ... */ })
    it('muestra el campo password', () => { /* ... */ })
    it('muestra el botón Entrar', () => { /* ... */ })
  })
})
```

> Consejo: el nombre de cada `it` debe describir el **comportamiento esperado**, no la implementación. "muestra el botón Entrar" es mejor que "test 3".

---

## Paso 1 — El esqueleto del spec

Empezamos por lo más pequeño: un spec con un solo `it` vacío. Crea el archivo con **exactamente** esto:

```ts
// cypress/e2e/login.cy.ts
describe('Página de login', () => {
  it('arranca correctamente', () => {
    // Por ahora vacío: solo comprobamos que Cypress lo lista
  })
})
```

Guarda y mira el runner: `login.cy.ts` aparece en la lista y, al ejecutarlo, el test pasa en **verde** (un test sin aserciones siempre pasa). Ya tenemos el cascarón; ahora lo llenamos.

---

## Paso 2 — `cy.visit()` y `beforeEach`

Todo test E2E empieza visitando una URL. Como definimos `baseUrl` en el módulo 1, usamos rutas relativas:

```ts
// cypress/e2e/login.cy.ts
describe('Página de login', () => {
  it('carga /login', () => {
    cy.visit('/login') // equivale a http://localhost:5173/login
  })
})
```

Para no repetir `cy.visit('/login')` en cada test, lo movemos a un `beforeEach`. Reescribe el spec así:

```ts
// cypress/e2e/login.cy.ts
describe('Página de login', () => {
  beforeEach(() => {
    // Antes de cada test, partimos siempre desde /login
    cy.visit('/login')
  })

  it('muestra el formulario', () => {
    // aquí la página ya está cargada
  })
})
```

Guarda y observa en el runner: ahora, antes de cada `it`, verás el comando `visit /login` ejecutarse y la app cargarse a la derecha.

> Cada `it` arranca en una página **limpia**: Cypress borra el estado del navegador entre tests. Por eso necesitamos volver a visitar la página en cada uno.

---

## Paso 3 — `cy.get()`: seleccionar elementos del DOM

`cy.get()` busca elementos usando un selector CSS, igual que `document.querySelector`, pero con superpoderes: **reintenta** hasta que el elemento aparece o se agota el timeout.

```ts
// cypress/e2e/login.cy.ts
cy.get('[data-cy=username]')      // input de usuario
cy.get('[data-cy=password]')      // input de contraseña
cy.get('[data-cy=login-button]')  // botón "Entrar"
```

Nuestra app expone atributos `data-cy` precisamente para esto. Los selectores de login que usaremos en todo el curso son:

| Elemento | Selector |
|----------|----------|
| Input usuario | `[data-cy=username]` |
| Input contraseña | `[data-cy=password]` |
| Botón Entrar | `[data-cy=login-button]` |

También existe `cy.contains()`, que localiza un elemento por el **texto** que muestra:

```ts
// cypress/e2e/login.cy.ts
cy.contains('Entrar')              // cualquier elemento con el texto "Entrar"
cy.contains('button', 'Entrar')    // un <button> que contenga "Entrar"
```

| Comando | Busca por | Úsalo cuando |
|---------|-----------|--------------|
| `cy.get()` | Selector CSS / `data-cy` | Quieres precisión y estabilidad |
| `cy.contains()` | Texto visible | El texto es parte del contrato visual |

> En el módulo 3 veremos por qué, para selección estable, **preferimos `data-cy`** sobre el texto.

### 🔧 Tu turno #1

Añade dentro del `it('muestra el formulario', ...)` una sola línea que **seleccione** el input de usuario por su `data-cy`. Todavía **no** hace falta aserción: solo el `cy.get`. Predice qué verás en el Command Log del runner.

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/login.cy.ts
it('muestra el formulario', () => {
  cy.get('[data-cy=username]')
})
```

En el Command Log verás `get [data-cy=username]` y, al pasar el cursor por encima, el runner **resaltará** el input en la app de la derecha. El test sigue en verde porque `cy.get` falla solo si el elemento no existe tras el timeout. En el siguiente paso le añadimos una aserción de verdad.

</details>

---

## Paso 4 — Tu primera aserción: `.should()` y `.and()`

Una **aserción** verifica que algo es cierto. Sin aserciones, un test solo "hace clicks" pero nunca confirma nada. En Cypress las aserciones más comunes se escriben con `.should()`. Completa el primer test para que compruebe que los tres controles del formulario están visibles:

```ts
// cypress/e2e/login.cy.ts
describe('Página de login', () => {
  beforeEach(() => {
    cy.visit('/login')
  })

  it('muestra el formulario de login completo', () => {
    // El input de usuario debe ser visible
    cy.get('[data-cy=username]').should('be.visible')

    // El input de contraseña debe ser visible
    cy.get('[data-cy=password]').should('be.visible')

    // El botón debe ser visible y contener el texto "Entrar"
    cy.get('[data-cy=login-button]')
      .should('be.visible')
      .and('contain', 'Entrar')
  })
})
```

`.and()` es un alias de `.should()` que se lee mejor cuando encadenas varias aserciones sobre el **mismo** elemento:

```ts
// cypress/e2e/login.cy.ts
cy.get('[data-cy=login-button]')
  .should('be.visible')   // primera aserción
  .and('be.enabled')      // segunda aserción, mismo elemento
  .and('contain', 'Entrar')
```

| Aserción | Significado |
|----------|-------------|
| `be.visible` | El elemento se ve en pantalla |
| `exist` | El elemento está en el DOM (visible o no) |
| `contain` | Su texto incluye el valor dado |
| `have.value` | El input tiene ese valor |
| `be.enabled` / `be.disabled` | Estado del control |
| `have.length` | Cantidad de elementos seleccionados |

Guarda: ahora el test hace algo real. En el Command Log cada `should` aparece con un ✔ verde.

---

## Lo más importante: los comandos son una **cola asíncrona**

Este es el concepto que más confunde al venir de otras librerías. En Cypress, cuando escribes:

```ts
// cypress/e2e/login.cy.ts
cy.visit('/login')
cy.get('[data-cy=username]')
```

**no estás ejecutando** esos comandos inmediatamente. Los estás **encolando**. Cypress lee todo el `it`, arma una cola de comandos y luego la ejecuta uno a uno, esperando a que cada paso termine antes del siguiente.

```
Tú escribes:                  Cypress encola:               Luego ejecuta:
─────────────                 ───────────────               ──────────────
cy.visit('/login')      →     [ visit, get, should ]   →    1) visit  ✔
cy.get('[data-cy=...]')                                      2) get    ✔ (reintenta)
  .should('be.visible')                                      3) should ✔
```

### No uses `async/await`

Como los comandos se encolan, **no necesitas** (ni debes) usar `async/await` ni `.then()` para encadenarlos. Esto es **incorrecto**:

```ts
// cypress/e2e/login.cy.ts
// MAL: async/await no aplica a la cola de Cypress
it('mal ejemplo', async () => {
  const input = await cy.get('[data-cy=username]') // NO funciona como esperas
  input.type('admin')                               // 'input' no es un elemento DOM
})
```

Lo correcto es simplemente **encadenar**:

```ts
// cypress/e2e/login.cy.ts
// BIEN: encadenamos comandos; Cypress los serializa por ti
it('buen ejemplo', () => {
  cy.get('[data-cy=username]')
    .should('be.visible')
    .type('admin')
})
```

> Regla de oro: lo que devuelve `cy.get()` **no es** un elemento del DOM, es un objeto *Chainable* de Cypress. Por eso no se asigna a variables como si fuera un nodo HTML.

### Reintentos automáticos

Gracias a la cola, Cypress espera de forma inteligente. Si el formulario tarda 300 ms en aparecer, `cy.get()` reintenta hasta encontrarlo (hasta agotar el `defaultCommandTimeout`, 4 s por defecto). Esto **elimina** la necesidad de `sleep` o esperas manuales en la mayoría de los casos.

```
cy.get('[data-cy=username]')
   │
   ├─ ¿existe? no → reintenta (50 ms)
   ├─ ¿existe? no → reintenta (50 ms)
   ├─ ¿existe? sí → continúa
   └─ pasa al siguiente comando
```

---

## Paso 5 — 🔧 Tu turno #2: escribe el siguiente caso

Dentro del mismo `describe`, añade un **segundo `it`** que verifique que el botón Entrar **está habilitado** (no deshabilitado) cuando el formulario está vacío. No borres el test anterior.

**Pistas:**
- El `beforeEach` ya visita `/login`, así que no repitas `cy.visit`.
- Selecciona `[data-cy=login-button]` y usa la aserción `be.enabled`.

Inténtalo antes de mirar.

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/login.cy.ts  (añade este it dentro del describe)
it('el botón Entrar está habilitado al cargar', () => {
  cy.get('[data-cy=login-button]').should('be.enabled')
})
```

Ahora la suite tiene **2 tests en verde**. En el Command Log verás que el `beforeEach` (`visit /login`) se ejecuta otra vez antes de este segundo `it`: cada test parte de una página limpia. Si por error escribiste `be.disabled`, el test fallará y el mensaje te dirá que el elemento sí estaba habilitado.

</details>

---

## El archivo completo (referencia)

Tras los cinco pasos, tu spec debería parecerse a esto. Úsalo solo para **comparar**, no para copiar:

```ts
// cypress/e2e/login.cy.ts
describe('Página de login', () => {
  beforeEach(() => {
    cy.visit('/login')
  })

  it('muestra el formulario de login completo', () => {
    cy.get('[data-cy=username]').should('be.visible')
    cy.get('[data-cy=password]').should('be.visible')
    cy.get('[data-cy=login-button]')
      .should('be.visible')
      .and('contain', 'Entrar')
  })

  it('el botón Entrar está habilitado al cargar', () => {
    cy.get('[data-cy=login-button]').should('be.enabled')
  })
})
```

---

## Ejecutar el test

Con la app corriendo y Cypress abierto en modo interactivo:

```bash
# Terminal 1: la app debe estar viva
npm run dev

# Terminal 2: abre Cypress
npm run cypress:open
```

En el **Test Runner**, elige el navegador y haz click en `login.cy.ts`. Verás cómo Cypress abre la app, ejecuta cada comando y marca el test en **verde** si todas las aserciones pasan.

Para correrlo sin ventana (como en CI):

```bash
# Ejecuta todos los specs en headless
npm run cypress:run
```

---

## El Test Runner y el time-travel

El Test Runner es la joya de Cypress. A la izquierda ves la **lista de comandos** ejecutados; a la derecha, la **app en vivo**.

```
┌───────────────────────────┬───────────────────────────────┐
│  Command Log              │     App preview                │
│  ───────────              │     ───────────                │
│  ✔ visit  /login          │   ┌─────────────────────────┐  │
│  ✔ get    [data-cy=user.] │   │  Usuario: [_________]   │  │
│  ✔ should be.visible      │   │  Pass:    [_________]   │  │
│  ✔ get    [login-button]  │   │         [  Entrar  ]    │  │
│  ✔ should contain 'Entrar'│   └─────────────────────────┘  │
└───────────────────────────┴───────────────────────────────┘
```

El **time-travel** es lo más útil: pasa el cursor por cualquier comando del log y la vista de la derecha muestra **cómo estaba la pantalla en ese instante exacto**. Si un test falla, puedes "viajar" al paso anterior y ver qué pasó justo antes del error.

> Time-travel convierte la depuración de "¿por qué falló?" en "déjame ver el momento exacto del fallo".

---

### Prueba esto

- Crea `cypress/e2e/login.cy.ts` con un `describe` y un `beforeEach` que visite `/login`.
- Escribe un `it` que verifique que `[data-cy=username]`, `[data-cy=password]` y `[data-cy=login-button]` son visibles.
- Sustituye una de las aserciones por `cy.contains('Entrar')` y observa que también pasa.
- Intenta (a propósito) escribir el test con `async/await` y comprueba que no se comporta como esperas; luego corrígelo encadenando.
- Cambia un selector a uno inexistente y observa cómo Cypress reintenta hasta fallar por timeout.
- En el Test Runner, pasa el cursor por cada comando del log y observa el **time-travel**.

---

## Ejercicios propuestos

### Básico

**B1 — Aserción de placeholder.** Añade un `it` que verifique que el input `[data-cy=username]` tiene un atributo `placeholder` con la aserción `should('have.attr', 'placeholder')`.

**B2 — Tres `it` por control.** Reorganiza el test grande en tres `it` independientes: uno por cada control (`username`, `password`, `login-button`). Observa cómo se ven los tres casos en la salida.

### Intermedio

**I1 — `describe` anidado.** Envuelve los tests en `describe('Autenticación', ...)` y dentro `describe('Formulario', ...)`. Comprueba cómo el runner muestra la jerarquía anidada.

**I2 — `cy.url()`.** Añade un `it` que tras `cy.visit('/login')` verifique que la URL incluye `/login` usando `cy.url().should('include', '/login')`.

### Avanzado

**A1 — Demostrar el reintento.** Crea un test que visite una ruta y haga `cy.get('[data-cy=login-button]', { timeout: 1000 })` sobre un selector inexistente; mide cuánto tarda en fallar y compáralo con el timeout por defecto de 4 s.

**A2 — `cy.contains` vs `cy.get`.** Escribe dos tests equivalentes que verifiquen el botón Entrar, uno con `cy.contains('button', 'Entrar')` y otro con `cy.get('[data-cy=login-button]')`. Cambia el texto del botón en la app a "Acceder" y observa cuál de los dos se rompe.

---

## Resumen del módulo 2

- Un **spec** se organiza con `describe` (grupo) e `it` (test); `beforeEach` prepara cada test.
- Construimos el spec **por pasos**: del esqueleto vacío, a `cy.visit`, al `cy.get`, a la primera aserción, hasta escribir tú mismo el segundo caso.
- `cy.visit('/login')` abre la página usando el `baseUrl` configurado.
- `cy.get()` selecciona por CSS / `data-cy`; `cy.contains()` selecciona por texto.
- Las **aserciones** se escriben con `.should()` y se encadenan con `.and()`.
- Los comandos de Cypress se **encolan** y ejecutan en orden: **no uses `async/await`**.
- `cy.get()` devuelve un *Chainable*, no un nodo del DOM, y **reintenta** automáticamente.
- El **Test Runner** muestra los comandos y la app en vivo; el **time-travel** revela el estado en cada paso.

> **Siguiente página →** Módulo 3: Selectores y buenas prácticas — por qué `data-cy` es el estándar, la tabla de prioridad de selectores y comandos como `.within()`, `.find()`, `.first()` y `.eq()`.
