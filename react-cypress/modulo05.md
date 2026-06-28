# Testing E2E en React con Cypress — Página 5

## Módulo 5 · Aserciones

### Cómo verificar el estado real de tu aplicación con aserciones implícitas y explícitas

---

## Cómo trabajar este módulo

Este módulo es **incremental**: iremos sumando aserciones a un mismo spec,
**una idea por paso**. Mantén abiertas dos ventanas:

- La app en `http://localhost:5173` (rutas `/login` y `/todos`, login `admin` / `1234`).
- Cypress con `npx cypress open` en modo E2E, para ver cada aserción pasar o fallar.

Escribe **un bloque por paso**, guarda y observa el resultado. No copies el spec
final de golpe. Entre paso y paso hay retos **🔧 Tu turno** que resolverás
**antes** de mirar la solución.

```
Paso 1  →  aserción implícita .should()
Paso 2  →  encadenar con .and()
Paso 3  →  tú escribes una aserción explícita
Paso 4  →  longitud de la lista (have.length)
Paso 5  →  un todo aparece tras agregarlo
Paso 6  →  tú verificas un checkbox marcado
Paso 7  →  el reintento automático
```

> Nota: usaremos `cy.loginRapidoMock()` en algunos ejemplos sobre `/todos`. Es un comando que crearemos en el módulo 7; por ahora trátalo como "entrar logueado". Si aún no lo tienes, sustitúyelo por el login manual del módulo 4.

---

## Para qué sirven las aserciones

En los módulos anteriores aprendiste a **navegar**, **seleccionar** elementos e **interactuar** con la interfaz. Pero un test que solo hace clics no prueba nada: necesitamos **afirmar** (assert) que la aplicación se comporta como esperamos.

Una aserción es una declaración del tipo *"esto debe ser verdad"*. Si lo es, el test continúa; si no, el test falla con un mensaje claro. En Cypress hay dos grandes familias:

- **Implícitas** con `.should()` y `.and()`.
- **Explícitas** con `.then()` + `expect()`.

En este módulo dominarás ambas, conocerás el catálogo de aserciones más usadas y entenderás el **reintento automático**, una de las características que hace a Cypress tan robusto.

---

## Paso 1 — Tu primera aserción implícita: `.should()`

Es la forma más idiomática en Cypress. Se **encadena** directamente sobre el comando que obtiene el elemento. Cypress aplica la aserción al *subject* (el elemento sobre el que trabajamos).

Crea el spec con **solo este caso**: comprobar que el botón de login es visible.

```ts
// cypress/e2e/aserciones.cy.ts
describe('Aserciones sobre el login', () => {
  beforeEach(() => {
    // Visitamos la pantalla de login antes de cada test
    cy.visit('/login')
  })

  it('muestra el botón de login visible', () => {
    // Seleccionamos por data-cy y afirmamos que es visible
    cy.get('[data-cy="login-button"]').should('be.visible')
  })
})
```

La sintaxis es: `cy.get(selector).should(chainer, valor?)`.

El primer argumento (`chainer`) es una cadena que describe la aserción (`'be.visible'`, `'have.value'`, etc.). El segundo es opcional y depende del chainer.

### 🔧 Tu turno #1

Sin mirar abajo: **añade un segundo `it`** que verifique que el input de usuario (`[data-cy="username"]`) **empieza vacío**. Pista: existe un chainer `have.value` que acepta un segundo argumento.

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/aserciones.cy.ts  (añade este it dentro del describe)
  it('el input de usuario empieza vacío', () => {
    // have.value comprueba el valor de un campo de formulario
    cy.get('[data-cy="username"]').should('have.value', '')
  })
```

`have.value` con `''` confirma que el campo no tiene texto inicial.

</details>

---

## Paso 2 — Encadenar aserciones con `.and()`

Cuando quieres aplicar **varias aserciones al mismo elemento**, usa `.and()`. Es un alias de `.should()` que mejora la legibilidad: se lee como una frase.

```ts
// cypress/e2e/aserciones.cy.ts  (añade este it dentro del describe)
  it('el botón está visible y contiene el texto correcto', () => {
    cy.get('[data-cy="login-button"]')
      .should('be.visible')          // primera aserción
      .and('contain', 'Entrar')      // segunda aserción sobre el MISMO elemento
      .and('not.be.disabled')        // tercera aserción
  })
```

> Nota mental: `.should()` y `.and()` operan sobre el **mismo subject**. Si en medio cambias de elemento (con otro `.get()`), las aserciones se aplican al nuevo subject.

---

## El catálogo de aserciones comunes

Esta tabla es tu chuleta de referencia. Todos estos chainers funcionan con `.should()` y `.and()`.

| Chainer | Qué comprueba | Ejemplo |
| --- | --- | --- |
| `exist` | El elemento existe en el DOM | `.should('exist')` |
| `not.exist` | El elemento NO está en el DOM | `.should('not.exist')` |
| `be.visible` | Es visible para el usuario | `.should('be.visible')` |
| `have.text` | El texto exacto del elemento | `.should('have.text', 'Hola')` |
| `contain` | El elemento contiene un texto | `.should('contain', 'Hola')` |
| `have.value` | El valor de un input | `.should('have.value', 'admin')` |
| `have.length` | Cantidad de elementos | `.should('have.length', 3)` |
| `be.checked` | Un checkbox está marcado | `.should('be.checked')` |
| `not.be.checked` | Un checkbox NO está marcado | `.should('not.be.checked')` |
| `have.class` | Tiene una clase CSS | `.should('have.class', 'done')` |
| `have.attr` | Tiene un atributo (con valor opcional) | `.should('have.attr', 'href', '/todos')` |
| `be.disabled` | Está deshabilitado | `.should('be.disabled')` |

> Diferencia clave: `have.text` exige el texto **completo y exacto**; `contain` solo exige que el texto **incluya** el valor.

```ts
// cypress/e2e/have-text-vs-contain.cy.ts
it('have.text vs contain', () => {
  cy.visit('/login')

  // have.text: el texto debe coincidir EXACTAMENTE (sin espacios extra)
  cy.get('[data-cy="login-button"]').should('have.text', 'Entrar')

  // contain: basta con que el texto esté contenido
  cy.get('[data-cy="login-button"]').should('contain', 'Entr')
})
```

---

## Paso 3 — 🔧 Tu turno #2: aserciones explícitas con `.then()` + `expect()`

A veces necesitas lógica intermedia: extraer un valor, transformarlo, compararlo. Para eso usamos `.then()` para acceder al subject y `expect()` (sintaxis Chai) para afirmar.

**Tu reto:** escribe un `it` que obtenga `[data-cy="login-button"]`, extraiga su texto con `.then(($boton) => ...)`, lo recorte con `.trim()` y afirme con `expect()` que es exactamente `'Entrar'` y que su longitud es mayor que 0.

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/aserciones-explicitas.cy.ts
it('extrae el texto y lo valida manualmente', () => {
  cy.visit('/login')
  cy.get('[data-cy="login-button"]').then(($boton) => {
    // $boton es un objeto jQuery: accedemos al texto con .text()
    const texto = $boton.text().trim()
    // expect() es una aserción explícita de Chai
    expect(texto).to.equal('Entrar')
    expect(texto.length).to.be.greaterThan(0)
  })
})
```

### ¿Cuándo usar cada una?

| Situación | Recomendación |
| --- | --- |
| Comprobar estado visible/texto/valor directo | `.should()` (implícita) |
| Encadenar varias aserciones al mismo elemento | `.should().and()` |
| Necesitas extraer/transformar un valor | `.then()` + `expect()` |
| Comparar dos elementos o variables | `.then()` + `expect()` |
| Quieres reintento automático | `.should()` (¡siempre que puedas!) |

</details>

---

## Paso 4 — La lista de todos tiene N elementos

Una de las aserciones más útiles en una app Todo es comprobar **cuántos elementos** hay en la lista. Usamos `have.length`.

```
   Lista de todos (data-cy="todo-item")
   ┌───────────────────────────────┐
   │ [ ] Comprar pan               │  ← elemento 0
   │ [ ] Pasear al perro           │  ← elemento 1
   │ [x] Leer un libro             │  ← elemento 2
   └───────────────────────────────┘
            have.length === 3
```

```ts
// cypress/e2e/lista-longitud.cy.ts
describe('Longitud de la lista de todos', () => {
  beforeEach(() => {
    cy.loginRapidoMock()
    cy.visit('/todos')
  })

  it('la lista contiene exactamente 3 elementos al inicio', () => {
    // have.length verifica el número de elementos seleccionados
    cy.get('[data-cy="todo-item"]').should('have.length', 3)
  })

  it('la lista no está vacía', () => {
    // have.length.greaterThan permite comparaciones numéricas
    cy.get('[data-cy="todo-item"]').should('have.length.greaterThan', 0)
  })
})
```

Variantes numéricas útiles:

| Chainer | Significado |
| --- | --- |
| `have.length`, n | Exactamente n |
| `have.length.greaterThan`, n | Más de n |
| `have.length.lessThan`, n | Menos de n |
| `have.length.at.least`, n | n o más |

---

## Paso 5 — Un todo aparece tras agregarlo

Este es el flujo más representativo del CRUD. Escribimos en el input, pulsamos "Agregar" y afirmamos que el nuevo todo **aparece** y que la lista **creció**.

```ts
// cypress/e2e/agregar-todo.cy.ts
describe('Agregar un todo', () => {
  beforeEach(() => {
    cy.loginRapidoMock()
    cy.visit('/todos')
  })

  it('muestra el nuevo todo tras agregarlo', () => {
    const nuevoTexto = 'Estudiar Cypress'

    // Estado inicial: contamos los elementos existentes
    cy.get('[data-cy="todo-item"]').should('have.length', 3)

    // Acción: escribimos y agregamos
    cy.get('[data-cy="new-todo-input"]').type(nuevoTexto)
    cy.get('[data-cy="add-button"]').click()

    // Aserción 1: la lista ahora tiene un elemento más
    cy.get('[data-cy="todo-item"]').should('have.length', 4)

    // Aserción 2: el nuevo texto aparece en la lista
    cy.get('[data-cy="todo-item"]')
      .should('contain', nuevoTexto)

    // Aserción 3 (más estricta): el ÚLTIMO elemento contiene el texto
    cy.get('[data-cy="todo-item"]')
      .last()
      .should('be.visible')
      .and('contain', nuevoTexto)
  })
})
```

Observa cómo combinamos **acción** + **aserciones encadenadas**. El patrón *Arrange-Act-Assert* (Preparar-Actuar-Afirmar) queda muy claro.

---

## Paso 6 — 🔧 Tu turno #3: verificar un checkbox marcado

Marcar un todo como completado deja el checkbox `checked` y suele aplicar una clase CSS al item (`completed`).

**Tu reto:** escribe un `it` que, sobre el **primer** `[data-cy="todo-item"]`:
1. Verifique que su `[data-cy="todo-checkbox"]` **no** está marcado al inicio.
2. Lo marque con `.check()`.
3. Afirme que ahora **sí** está marcado.
4. Afirme que el item recibió la clase `completed`.

Pista: usa `.first().find('[data-cy="todo-checkbox"]')` para llegar al checkbox del primer item.

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/completar-todo.cy.ts
describe('Completar un todo', () => {
  beforeEach(() => {
    cy.loginRapidoMock()
    cy.visit('/todos')
  })

  it('marca el primer todo como completado', () => {
    // El checkbox empieza sin marcar
    cy.get('[data-cy="todo-item"]')
      .first()
      .find('[data-cy="todo-checkbox"]')
      .should('not.be.checked')

    // Lo marcamos con .check() (específico de checkboxes)
    cy.get('[data-cy="todo-item"]')
      .first()
      .find('[data-cy="todo-checkbox"]')
      .check()

    // Aserción: ahora está marcado
    cy.get('[data-cy="todo-item"]')
      .first()
      .find('[data-cy="todo-checkbox"]')
      .should('be.checked')

    // Aserción adicional: el item recibe una clase visual "completed"
    cy.get('[data-cy="todo-item"]')
      .first()
      .should('have.class', 'completed')
  })
})
```

> Tip: si repites `cy.get(...).first().find(...)` muchas veces, considera guardar el alias con `.as('primerTodo')` y reusar con `cy.get('@primerTodo')`.

</details>

---

## Paso 7 — El reintento automático: la magia de Cypress

Esta es la idea **más importante** del módulo. Las aserciones implícitas (`.should()` / `.and()`) **se reintentan automáticamente** hasta que pasan o hasta agotar el timeout (4 segundos por defecto).

```
   cy.get('[data-cy="todo-item"]').should('have.length', 4)

   t=0ms   ┌─> ¿hay 4 items?  → NO (hay 3, el POST aún no terminó)
           │   reintenta...
   t=50ms  ├─> ¿hay 4 items?  → NO
   t=120ms ├─> ¿hay 4 items?  → NO
   t=300ms └─> ¿hay 4 items?  → SÍ  ✓  el test continúa
           ─────────────────────────────────────────────
           (si tras 4000ms sigue sin cumplirse → FALLA)
```

¿Por qué importa? Porque en una app real la UI tarda en actualizarse tras una petición de red. Gracias al reintento **no necesitas `cy.wait(1000)`**: Cypress espera por ti, justo lo necesario, ni más ni menos.

| Tipo de aserción | ¿Se reintenta? |
| --- | --- |
| `.should(...)` / `.and(...)` | **Sí**, automáticamente |
| `.then(($el) => expect(...))` | **No**, se evalúa una sola vez |

Por eso, cuando puedas, **prefiere `.should()`**. Usa `.then()` + `expect()` solo cuando necesites lógica que no cabe en un chainer.

```ts
// cypress/e2e/reintento.cy.ts
it('demuestra el reintento sin esperas fijas', () => {
  cy.loginRapidoMock()
  cy.visit('/todos')

  cy.get('[data-cy="new-todo-input"]').type('Tarea con red lenta')
  cy.get('[data-cy="add-button"]').click()

  // Aunque el guardado tarde, .should() reintenta hasta verlo
  cy.get('[data-cy="todo-item"]').should('contain', 'Tarea con red lenta')
})
```

### Aserciones negativas: un aviso importante

Las aserciones negativas (`not.exist`, `not.be.visible`) también se reintentan, pero ten cuidado: pueden pasar "demasiado pronto" si el elemento aún no ha aparecido.

```ts
// cypress/e2e/negativas.cy.ts
it('verifica que un todo desaparece tras eliminarlo', () => {
  cy.loginRapidoMock()
  cy.visit('/todos')

  // Lo eliminamos
  cy.get('[data-cy="todo-item"]')
    .first()
    .find('[data-cy="delete-button"]')
    .click()

  // Aserción positiva primero (la lista bajó a 2), luego la negativa
  cy.get('[data-cy="todo-item"]').should('have.length', 2)
})
```

> Buena práctica: combina una aserción **positiva** (que algo cambió) antes de una **negativa** (que algo ya no existe). Así evitas falsos positivos.

---

### Prueba esto

- Cambia `have.text` por `contain` en el botón de login y observa la diferencia cuando hay espacios extra.
- Agrega dos todos seguidos y verifica con `have.length` que la lista crece de 3 a 5.
- Marca y desmarca un checkbox encadenando `.check().should('be.checked').uncheck().should('not.be.checked')`.
- Usa `.then(($items) => expect($items).to.have.length(3))` y compáralo con `.should('have.length', 3)`: ¿cuál reintenta?
- Comprueba con `have.attr` que el enlace a `/todos` tiene el atributo `href` correcto.
- Provoca un fallo a propósito (`have.length`, 99) y lee el mensaje de error que produce Cypress.

---

## Ejercicios propuestos

### Básico

**B1 — Visible y habilitado.** Sobre `/login`, encadena en un solo `cy.get` que el botón está `be.visible` y `not.be.disabled`.

**B2 — Lista no vacía.** En `/todos`, verifica con `have.length.greaterThan` que hay al menos un `[data-cy="todo-item"]`.

### Intermedio

**I1 — El último todo.** Agrega un todo "Reciente" y verifica con `.last().and('contain', 'Reciente')` que aparece como último elemento de la lista.

**I2 — Clase tras completar.** Marca el segundo todo y verifica que ese item tiene la clase `completed` mientras el primero **no** la tiene.

### Avanzado

**A1 — Implícita vs explícita.** Escribe el mismo control de longitud de dos formas: con `.should('have.length', 3)` y con `.then(($i) => expect($i).to.have.length(3))`. Documenta en un comentario cuál reintenta y por qué.

**A2 — Atributos del input.** Verifica que `[data-cy="new-todo-input"]` tiene el atributo `placeholder` esperado usando `have.attr`, y que está vacío (`have.value`, `''`) al cargar la página.

---

## Resumen del módulo 5

- Las aserciones afirman que la app está en el estado esperado; sin ellas un test no prueba nada.
- Las aserciones **implícitas** (`.should()` / `.and()`) se encadenan al elemento y se **reintentan automáticamente**.
- Las aserciones **explícitas** (`.then()` + `expect()`) sirven para lógica intermedia, pero **no reintentan**.
- Conoces el catálogo: `exist`, `be.visible`, `have.text`, `contain`, `have.value`, `have.length`, `be.checked`, `have.class`, `have.attr`.
- Construiste, paso a paso, aserciones sobre el login, la longitud de la lista, la aparición de un todo y el estado de un checkbox.
- El **reintento automático** elimina la necesidad de esperas fijas y hace tus tests robustos.

> **Siguiente página →** Módulo 6: Esperas y red con `cy.intercept()` — espiar, mockear y controlar las peticiones HTTP de tu app.
