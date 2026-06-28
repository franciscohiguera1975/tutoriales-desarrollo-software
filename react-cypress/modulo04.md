# Testing E2E en React con Cypress — Página 4
## Módulo 4 · Interacción con la UI
### Escribir, hacer click, marcar y enviar: simular a un usuario real sobre el login y la lista de todos

---

## Cómo trabajar este módulo

Este módulo es **incremental**: construiremos los tests de interacción
**paso a paso**, no de un copia-pega gigante. Trabaja con dos ventanas abiertas:

- La app corriendo en `http://localhost:5173` (rutas `/login` y `/todos`).
- Cypress abierto con `npx cypress open`, en modo E2E, para ver cada test correr en vivo.

Añade **un bloque por paso**, guarda y mira cómo Cypress re-ejecuta el spec al
instante. Entre paso y paso encontrarás retos **🔧 Tu turno** que debes resolver
**antes** de mirar la solución.

```
Paso 1  →  escribir en un input (.type)
Paso 2  →  enviar el login (.click)
Paso 3  →  tú envías el login con {enter}
Paso 4  →  crear un todo
Paso 5  →  marcar y eliminar un todo
Paso 6  →  tú escribes el flujo combinado
```

Recuerda: las credenciales válidas son usuario **`admin`**, contraseña **`1234`**.

---

## De observar a actuar

En los módulos anteriores **leímos** el DOM (`cy.get`, `cy.contains`) y **verificamos** estados (`.should`). Ahora vamos a **actuar**: escribir en inputs, pulsar botones, marcar checkboxes y enviar formularios, exactamente como lo haría una persona.

Los comandos de interacción de Cypress simulan eventos reales del navegador (focus, keydown, input, change, click...). Y, como ya sabes, **reintentan** hasta que el elemento esté listo para recibir la acción (visible, habilitado y no tapado por otro elemento).

| Comando | Acción | Equivalente humano |
|---------|--------|--------------------|
| `.click()` | Pulsa un elemento | Hacer click |
| `.type()` | Escribe texto | Teclear |
| `.clear()` | Vacía un input | Seleccionar todo y borrar |
| `.check()` / `.uncheck()` | Marca / desmarca | Click en un checkbox |
| `.select()` | Elige una opción de un `<select>` | Abrir el desplegable y elegir |
| `.submit()` | Envía un formulario | Pulsar Enter en el form |

---

## Paso 1 — Escribir en un input con `.type()`

`.type()` escribe carácter por carácter, disparando los eventos de teclado reales. Esto importa en React, donde cada pulsación actualiza el estado controlado del input.

Crea el archivo de test con **solo este caso**: visitamos `/login` y escribimos el usuario.

```ts
// cypress/e2e/login.cy.ts
describe('Login', () => {
  beforeEach(() => {
    // Antes de cada test partimos de la pantalla de login
    cy.visit('/login')
  })

  it('escribe el usuario en el campo', () => {
    // Escribimos y verificamos que el valor quedó registrado
    cy.get('[data-cy=username]')
      .type('admin')
      .should('have.value', 'admin')
  })
})
```

Guarda y mira Cypress: verás cómo el texto se teclea letra a letra. Desglose:

- `.type('admin')` simula pulsar cada tecla; React actualiza el estado en cada una.
- `.should('have.value', 'admin')` confirma que el input contiene exactamente ese valor.

> `.type()` exige que el elemento sea un campo editable (input, textarea o `contenteditable`). Si apuntas a un `div`, fallará.

### 🔧 Tu turno #1

Sin mirar abajo: **añade un segundo `it`** que escriba la contraseña `1234` en `[data-cy=password]` y verifique su valor. Además, prueba **vaciar** el campo después: investiga `.clear()` y comprueba que el valor vuelve a `''`.

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/login.cy.ts  (añade este it dentro del describe)
  it('escribe y luego limpia la contraseña', () => {
    cy.get('[data-cy=password]')
      .type('1234')
      .should('have.value', '1234')
      .clear()                       // vacía el input (azúcar de '{selectall}{del}')
      .should('have.value', '')
  })
```

`.clear()` es azúcar sintáctico de `.type('{selectall}{del}')`. Útil al editar campos que ya tienen contenido antes de reescribirlos.

</details>

---

## Paso 2 — Enviar el login con `.click()`

`.click()` dispara un click real en el centro del elemento. Cypress comprueba primero que es **accionable**: visible, no deshabilitado y no cubierto por otro.

Ahora juntamos las piezas: rellenamos el formulario y lo enviamos pulsando el botón.

```ts
// cypress/e2e/login.cy.ts  (añade este it dentro del describe)
  it('inicia sesión con credenciales válidas', () => {
    // Rellenar el formulario
    cy.get('[data-cy=username]').type('admin')
    cy.get('[data-cy=password]').type('1234')

    // Enviar haciendo click en el botón
    cy.get('[data-cy=login-button]').click()

    // Tras el login válido, la app navega a /todos
    cy.url().should('include', '/todos')
  })
```

Variantes útiles de `.click()`:

```ts
// cypress/e2e/login.cy.ts
cy.get('[data-cy=login-button]').click()               // click normal
cy.get('[data-cy=login-button]').click({ force: true }) // fuerza, ignora chequeos (úsalo poco)
cy.get('[data-cy=todo-item]').first().dblclick()       // doble click
```

> `{ force: true }` salta las comprobaciones de accionabilidad. Es un parche: si lo necesitas a menudo, suele indicar un problema real en la UI (un elemento tapado, por ejemplo).

---

## Paso 3 — 🔧 Tu turno #2: enviar el login de tres formas

Hay tres maneras de enviar un formulario. Ya viste el `.click()` en el botón. Escribe **un test que envíe el login pulsando Enter** dentro del campo password con la secuencia `{enter}`. Luego, como reto extra, escribe **otro test que use `.submit()`** sobre el `<form>` (`[data-cy=login-form]`).

| Forma de enviar | Cómo | Cuándo |
|-----------------|------|--------|
| Click en el botón | `.get('[data-cy=login-button]').click()` | Lo más fiel al usuario |
| Tecla Enter | `.type('1234{enter}')` | Simula pulsar Enter en el campo |
| `.submit()` | `.get('[data-cy=login-form]').submit()` | Cuando quieres probar el `onSubmit` directo |

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/login.cy.ts  (dos it más dentro del describe)
  it('inicia sesión pulsando Enter en el campo password', () => {
    cy.get('[data-cy=username]').type('admin')
    // El {enter} al final del texto dispara el submit del formulario
    cy.get('[data-cy=password]').type('1234{enter}')
    cy.url().should('include', '/todos')
  })

  it('inicia sesión enviando el formulario con .submit()', () => {
    cy.get('[data-cy=username]').type('admin')
    cy.get('[data-cy=password]').type('1234')
    // .submit() dispara el evento submit del <form> directamente
    cy.get('[data-cy=login-form]').submit()
    cy.url().should('include', '/todos')
  })
```

`.type()` interpreta secuencias entre llaves como **teclas especiales**. `{enter}` es la más común. `.submit()` solo se aplica sobre un elemento `<form>`.

</details>

---

## Paso 4 — Agregar un todo

Ya estamos en `/todos`. El alta de una tarea sigue un patrón claro: escribir en `[data-cy=new-todo-input]` y pulsar `[data-cy=add-button]`.

```ts
// cypress/e2e/todos.cy.ts
describe('Crear todos', () => {
  beforeEach(() => {
    cy.visit('/todos')
  })

  it('agrega una nueva tarea', () => {
    // Escribir el texto de la tarea
    cy.get('[data-cy=new-todo-input]').type('Comprar pan')

    // Pulsar "Agregar"
    cy.get('[data-cy=add-button]').click()

    // La nueva tarea debe aparecer en la lista
    cy.contains('[data-cy=todo-item]', 'Comprar pan').should('be.visible')
  })
})
```

```
1) escribir          2) click Agregar       3) aparece en la lista
┌────────────────┐   ┌──────────┐           ┌──────────────────────┐
│ Comprar pan_   │   │ Agregar  │  ──────►   │ ☐  Comprar pan    🗑  │
└────────────────┘   └──────────┘           └──────────────────────┘
 new-todo-input       add-button             todo-item
```

### Teclas especiales: crear un todo con `{enter}`

`.type()` interpreta secuencias entre llaves como teclas. Con `{enter}` creas un todo sin tocar el botón:

```ts
// cypress/e2e/todos.cy.ts
cy.get('[data-cy=new-todo-input]')
  .type('Lavar el coche{enter}') // escribe y pulsa Enter

cy.contains('[data-cy=todo-item]', 'Lavar el coche').should('be.visible')
```

Otras teclas y modificadores útiles:

| Secuencia | Efecto |
|-----------|--------|
| `{enter}` | Pulsar Enter |
| `{esc}` | Pulsar Escape |
| `{backspace}` | Borrar el carácter anterior |
| `{selectall}` | Seleccionar todo el texto |
| `{del}` | Suprimir |
| `{leftarrow}` `{rightarrow}` | Mover el cursor |

> Las llaves son literales de Cypress. Para escribir un `{` real, duplícalo: `.type('{{}')`.

---

## Paso 5 — Marcar y eliminar un todo

Para los checkboxes, usa `.check()` (marcar) y `.uncheck()` (desmarcar). Son más expresivos que `.click()` porque también **verifican** que el estado quedó como se pide.

```ts
// cypress/e2e/todos.cy.ts
it('marca una tarea como completada', () => {
  cy.visit('/todos')

  // Crear la tarea
  cy.get('[data-cy=new-todo-input]').type('Estudiar Cypress{enter}')

  // Localizarla y marcar su checkbox
  cy.contains('[data-cy=todo-item]', 'Estudiar Cypress').within(() => {
    cy.get('[data-cy=todo-checkbox]').check()
    // El checkbox debe quedar marcado
    cy.get('[data-cy=todo-checkbox]').should('be.checked')
  })
})
```

Para desmarcarla:

```ts
// cypress/e2e/todos.cy.ts
cy.get('[data-cy=todo-checkbox]').uncheck().should('not.be.checked')
```

> `.check()` y `.uncheck()` solo funcionan sobre `<input type="checkbox">` o `<input type="radio">`.

El borrado combina localizar el item y pulsar su `[data-cy=delete-button]`. Usamos `.within()` para asegurarnos de borrar **el correcto**.

```ts
// cypress/e2e/todos.cy.ts
it('elimina una tarea', () => {
  cy.visit('/todos')

  cy.get('[data-cy=new-todo-input]').type('Tarea temporal{enter}')
  cy.contains('[data-cy=todo-item]', 'Tarea temporal').should('exist')

  // Localizar el item y, dentro de él, su botón eliminar
  cy.contains('[data-cy=todo-item]', 'Tarea temporal').within(() => {
    cy.get('[data-cy=delete-button]').click()
  })

  // La tarea ya no debe existir en el DOM
  cy.contains('[data-cy=todo-item]', 'Tarea temporal').should('not.exist')
})
```

> `should('not.exist')` es la aserción correcta para confirmar que algo desapareció del DOM, distinta de `not.be.visible` (que sigue existiendo pero oculto).

---

## Paso 6 — 🔧 Tu turno #3: el flujo combinado

Ahora encadena todo lo aprendido en **un único test legible**: haz login en el `beforeEach`, luego dentro de un `it` **crea** un todo con `{enter}`, **márcalo** como completado y por último **elimínalo**, verificando que ya no existe. Inténtalo antes de mirar.

**Pistas:**
- En el `beforeEach`, repite el login (visit + type + type + click) y verifica la URL.
- Usa `.within()` para operar sobre el item correcto al marcar y al eliminar.

<details><summary>✅ Solución</summary>

```ts
// cypress/e2e/interaccion.cy.ts
describe('Interacción con la Todo App', () => {
  beforeEach(() => {
    // Login y entrada a /todos
    cy.visit('/login')
    cy.get('[data-cy=username]').type('admin')
    cy.get('[data-cy=password]').type('1234')
    cy.get('[data-cy=login-button]').click()
    cy.url().should('include', '/todos')
  })

  it('crea, completa y elimina una tarea', () => {
    // Crear con Enter
    cy.get('[data-cy=new-todo-input]').type('Pasear al perro{enter}')

    // Completar
    cy.contains('[data-cy=todo-item]', 'Pasear al perro').within(() => {
      cy.get('[data-cy=todo-checkbox]').check().should('be.checked')
    })

    // Eliminar
    cy.contains('[data-cy=todo-item]', 'Pasear al perro').within(() => {
      cy.get('[data-cy=delete-button]').click()
    })

    cy.contains('[data-cy=todo-item]', 'Pasear al perro').should('not.exist')
  })
})
```

Fíjate cómo el `beforeEach` saca el login del test y deja el `it` enfocado solo en el flujo crear → completar → eliminar.

</details>

---

## `.select()` — desplegables (bonus)

Aunque nuestra Todo App usa botones para filtrar, es habitual encontrar `<select>` en otras apps. `.select()` elige una opción por su **valor** o por su **texto**:

```ts
// cypress/e2e/ejemplo-select.cy.ts
// Por el texto visible de la opción
cy.get('[data-cy=filtro]').select('Completadas')

// Por el atributo value
cy.get('[data-cy=filtro]').select('completed')

// Selección múltiple
cy.get('[data-cy=etiquetas]').select(['urgente', 'casa'])
```

---

### Prueba esto

- Rellena el login con `.type('admin')` y `.type('1234')` y envíalo con `.click()` en `[data-cy=login-button]`.
- Repite el login pero enviándolo con `.type('1234{enter}')` en el campo password.
- Crea un todo escribiendo en `[data-cy=new-todo-input]` y pulsando `[data-cy=add-button]`.
- Crea otro todo usando solo `{enter}` y verifica que aparece como `[data-cy=todo-item]`.
- Marca un todo con `.check()` y comprueba `should('be.checked')`; luego desmárcalo con `.uncheck()`.
- Elimina un todo con `[data-cy=delete-button]` dentro de un `.within()` y verifica `should('not.exist')`.

---

## Ejercicios propuestos

### Básico

**B1 — Login fallido.** Escribe un test que escriba `admin` / `clave-mala`, pulse el botón y verifique que la URL **sigue** en `/login` (no navega).

**B2 — Dos todos seguidos.** En `/todos`, crea dos tareas distintas con `{enter}` y verifica que ambas aparecen como `[data-cy=todo-item]` visibles.

### Intermedio

**I1 — Editar antes de enviar.** Escribe `admon` en `[data-cy=username]`, luego usa `.clear()` y reescribe `admin`. Verifica el `value` final antes de enviar el login.

**I2 — Marcar y desmarcar.** Crea un todo, márcalo con `.check().should('be.checked')` y en la misma cadena desmárcalo con `.uncheck().should('not.be.checked')`.

### Avanzado

**A1 — Borrar el correcto.** Crea tres todos con textos distintos, elimina **solo el del medio** usando `.within()` y verifica con `have.length` que quedan dos y que el eliminado ya `not.exist`.

**A2 — Teclas especiales.** En `[data-cy=new-todo-input]`, escribe `Tarea con error`, usa `{backspace}` cinco veces para borrar "error", escribe `final` y crea la tarea con `{enter}`. Verifica que aparece "Tarea con final".

---

## Resumen del módulo 4

- Los comandos de interacción simulan eventos reales y **esperan** a que el elemento sea accionable.
- `.type()` escribe carácter a carácter; `.clear()` vacía un input antes de reescribirlo.
- `.click()` pulsa el centro del elemento; `{ force: true }` salta los chequeos (úsalo con cuidado).
- Un formulario se envía con `.click()` en su botón, con `{enter}` en un campo o con `.submit()`.
- En la Todo App: crear con `[data-cy=new-todo-input]` + `[data-cy=add-button]` (o `{enter}`).
- `.check()` / `.uncheck()` operan checkboxes y verifican el estado resultante.
- Eliminar combina `.within()` + `[data-cy=delete-button]`, y se confirma con `should('not.exist')`.
- `{enter}` y otras teclas especiales se escriben entre llaves dentro de `.type()`.
- Construimos los tests **por pasos**: de escribir en un input al flujo combinado que tú mismo redactaste.

> **Siguiente página →** Módulo 5: Aserciones — `.should()` con BDD y TDD, aserciones implícitas vs explícitas, `expect`, y cómo verificar texto, valores, longitudes, clases y estado de los todos.
