# Testing E2E en React con Cypress — Página 9
## Módulo 9 · Component Testing en Cypress
### Montar componentes React aislados en un navegador real, espiar sus callbacks y saber cuándo conviene

---

## Cómo trabajar este módulo

Este módulo es **incremental**: configuraremos el Component Testing y construiremos
las suites de `TodoItem` y `AddTodoForm` **paso a paso**. A diferencia de los
módulos E2E, aquí **no necesitas la app completa corriendo**: Cypress levanta su
propio dev-server con Vite. Aun así, ten Cypress abierto en modo componentes
(`npx cypress open --component`) para ver cada componente montarse en un navegador
real al guardar. Añade **un bloque por paso** y resuelve los retos **🔧 Tu turno**
**antes** de mirar la solución.

```
Paso 1  →  configurar la sección component en cypress.config.ts
Paso 2  →  registrar cy.mount y el HTML contenedor
Paso 3  →  montar TodoItem y probar lo visual
Paso 4  →  espiar callbacks con cy.stub().as()
Paso 5  →  montar AddTodoForm y probar su contrato
Paso 6  →  decidir CT vs E2E vs Vitest+RTL
```

Hasta ahora Cypress arrancaba la app completa en `http://localhost:5173` y la pilotaba como lo haría un usuario. Eso es **E2E (end-to-end)**. Pero Cypress tiene un segundo modo, el **Component Testing (CT)**: monta un único componente React en un navegador real, sin levantar toda la aplicación ni el enrutado.

```
   E2E                          Component Testing
  ┌──────────────────┐        ┌──────────────────┐
  │  app completa     │        │  cy.mount(        │
  │  /login  /todos   │        │    <TodoItem />)  │
  │  router + API     │        │  componente solo  │
  │  navegador real   │        │  navegador real   │
  └──────────────────┘        └──────────────────┘
```

La clave: **ambos corren en un navegador real** (Chrome, Electron…), a diferencia de Vitest + jsdom, que simula el DOM. Eso da fidelidad de estilos y layout, pero a cambio de velocidad.

| Aspecto | E2E | Component Testing |
|---------|-----|-------------------|
| Qué prueba | La app completa en `localhost:5173` | Un componente React aislado |
| Arranque | Necesita el servidor de la app corriendo | Levanta un dev-server propio con Vite |
| Navegación | `cy.visit('/ruta')` | `cy.mount(<Componente />)` |
| Velocidad | Más lento (toda la app) | Más rápido (un componente) |
| Red / API | Real o interceptada con `cy.intercept` | Normalmente *stubeada* vía props |
| Archivos | `cypress/e2e/*.cy.ts` | junto al componente o en `cypress/component` |
| Ideal para | Recorridos de usuario, integración real | Estados visuales y contratos de props |

> Regla mental: **E2E** prueba *recorridos*; **Component Testing** prueba *componentes y sus props*.

---

## Paso 1 — Configurar la sección `component`

Cypress 14 detecta automáticamente Vite + React. Añadimos la sección `component` a la configuración existente (que ya tenía `e2e` de los módulos 1-8):

```ts
// cypress.config.ts
import { defineConfig } from 'cypress'

export default defineConfig({
  // Configuración de E2E que ya teníamos (módulos 1-8).
  e2e: {
    baseUrl: 'http://localhost:5173',
    setupNodeEvents() {},
  },

  // Nueva sección de Component Testing.
  component: {
    devServer: {
      framework: 'react', // Cypress sabe montar componentes React.
      bundler: 'vite', // Reutiliza nuestro vite.config.ts.
    },
    specPattern: 'src/**/*.cy.{ts,tsx}', // Tests junto a los componentes.
  },
})
```

Guarda. Si abres `npx cypress open`, ahora verás dos opciones: **E2E Testing** y **Component Testing**.

---

## Paso 2 — Registrar `cy.mount` y el HTML contenedor

Cypress genera, al inicializar CT, un fichero de soporte con el comando `cy.mount`. Conviene revisarlo y dejarlo así:

```ts
// cypress/support/component.ts
// Importa el comando mount oficial para React 19.
import { mount } from 'cypress/react'

// Lo registramos como comando personalizado.
Cypress.Commands.add('mount', mount)
```

Y la declaración de tipos, para que TypeScript reconozca `cy.mount`:

```ts
// cypress/support/component.d.ts
import { mount } from 'cypress/react'

declare global {
  namespace Cypress {
    interface Chainable {
      /** Monta un componente React en el navegador de Cypress. */
      mount: typeof mount
    }
  }
}
```

También necesitamos un *HTML index* mínimo que Cypress usa como contenedor:

```html
<!-- cypress/support/component-index.html -->
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="utf-8" />
    <title>Cypress Component Testing</title>
  </head>
  <body>
    <!-- Cypress montará aquí cada componente. -->
    <div data-cy-root></div>
  </body>
</html>
```

Para abrir el modo de componentes:

```bash
# Abre el runner interactivo en modo Component Testing.
npx cypress open --component

# O headless en CI:
npx cypress run --component
```

Antes de testear, recordemos las firmas de los dos componentes de la app (idénticas a las del resto del curso):

```tsx
// src/components/TodoItem.tsx
export interface Todo {
  id: number
  text: string
  completed: boolean
}

interface TodoItemProps {
  todo: Todo
  onToggle: (id: number) => void
  onDelete: (id: number) => void
}

export function TodoItem({ todo, onToggle, onDelete }: TodoItemProps) {
  return (
    <li data-cy="todo-item" className={todo.completed ? 'completed' : ''}>
      <input
        type="checkbox"
        data-cy="todo-checkbox"
        checked={todo.completed}
        onChange={() => onToggle(todo.id)}
      />
      <span>{todo.text}</span>
      <button data-cy="delete-button" onClick={() => onDelete(todo.id)}>
        Eliminar
      </button>
    </li>
  )
}
```

```tsx
// src/components/AddTodoForm.tsx
import { useState } from 'react'

interface AddTodoFormProps {
  onAdd: (text: string) => void
}

export function AddTodoForm({ onAdd }: AddTodoFormProps) {
  const [text, setText] = useState('')

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    if (!text.trim()) return // No agregamos tareas vacías.
    onAdd(text.trim())
    setText('')
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        data-cy="new-todo-input"
        value={text}
        onChange={(e) => setText(e.target.value)}
      />
      <button data-cy="add-button" type="submit">
        Agregar
      </button>
    </form>
  )
}
```

---

## Paso 3 — Montar `TodoItem` y probar lo visual

La gran ventaja del Component Testing es que **pasamos las props directamente**. No hay API ni router de por medio: controlamos cada entrada del componente. Empieza con el caso más simple, comprobar que muestra el texto. Crea el spec con **solo este `it`**:

```tsx
// src/components/TodoItem.cy.tsx
import { TodoItem, type Todo } from './TodoItem'

describe('<TodoItem />', () => {
  const todoBase: Todo = { id: 1, text: 'Comprar pan', completed: false }

  it('muestra el texto de la tarea', () => {
    // Arrange + Act: montamos el componente con props directas.
    cy.mount(<TodoItem todo={todoBase} onToggle={() => {}} onDelete={() => {}} />)
    // Assert: el item contiene el texto.
    cy.get('[data-cy=todo-item]').should('contain.text', 'Comprar pan')
  })
})
```

Guarda y observa cómo el componente se monta en el navegador de Cypress. Ahora añade un segundo `it` que verifique el **estado completado**:

```tsx
// src/components/TodoItem.cy.tsx (añade este it dentro del describe)
  it('aplica la clase "completed" cuando la tarea está completa', () => {
    const completada: Todo = { ...todoBase, completed: true }
    cy.mount(<TodoItem todo={completada} onToggle={() => {}} onDelete={() => {}} />)
    cy.get('[data-cy=todo-item]').should('have.class', 'completed')
    cy.get('[data-cy=todo-checkbox]').should('be.checked')
  })
```

### 🔧 Tu turno #1

Escribe tú mismo un tercer `it` que verifique el caso contrario: cuando `completed` es `false`, el checkbox **NO** debe estar marcado y el item **NO** debe tener la clase `completed`. Usa `should('not.be.checked')` y `should('not.have.class', 'completed')`.

<details><summary>✅ Solución</summary>

```tsx
  it('NO marca el checkbox cuando la tarea está activa', () => {
    cy.mount(<TodoItem todo={todoBase} onToggle={() => {}} onDelete={() => {}} />)
    cy.get('[data-cy=todo-checkbox]').should('not.be.checked')
    cy.get('[data-cy=todo-item]').should('not.have.class', 'completed')
  })
```

`todoBase` ya tiene `completed: false`, así que aprovechamos ese estado base. Las aserciones negativas (`not.be.checked`, `not.have.class`) confirman el estado por defecto del componente.

</details>

---

## Paso 4 — Espiar callbacks con `cy.stub().as()`

Lo visual está cubierto. Falta el **comportamiento**: verificar que al interactuar se invocan los callbacks correctos. Para ello creamos un *stub*, le damos un alias y luego afirmamos sobre él. Añade este `it`:

```tsx
// src/components/TodoItem.cy.tsx (fragmento)
it('llama a onToggle con el id al marcar el checkbox', () => {
  // 1. Creamos un stub y le ponemos alias para referirnos luego con @.
  const onToggle = cy.stub().as('onToggle')

  cy.mount(<TodoItem todo={todoBase} onToggle={onToggle} onDelete={() => {}} />)

  // 2. Interactuamos con el componente como un usuario.
  cy.get('[data-cy=todo-checkbox]').check()

  // 3. Afirmamos que el stub fue llamado una vez con el id esperado.
  cy.get('@onToggle').should('have.been.calledOnceWith', 1)
})
```

> Las aserciones `have.been.calledOnceWith` vienen de **Sinon**, integrado en Cypress vía `chai-sinon`. Funcionan sobre cualquier `cy.stub()` o `cy.spy()` con alias.

### 🔧 Tu turno #2

Por simetría, falta probar el botón **Eliminar**. Escribe un `it` que cree un stub `onDelete` con alias, monte `TodoItem`, haga clic en `[data-cy=delete-button]` y verifique que `onDelete` se llamó una vez con el `id` correcto (`1`).

<details><summary>✅ Solución</summary>

```tsx
it('llama a onDelete con el id al pulsar Eliminar', () => {
  const onDelete = cy.stub().as('onDelete')

  cy.mount(<TodoItem todo={todoBase} onToggle={() => {}} onDelete={onDelete} />)

  cy.get('[data-cy=delete-button]').click()
  cy.get('@onDelete').should('have.been.calledOnceWith', 1)
})
```

Mismo patrón que `onToggle`: stub con alias, interacción y aserción Sinon. El componente pasa `todo.id` (que es `1`) al callback, por eso afirmamos `calledOnceWith', 1`.

</details>

---

## Paso 5 — Montar y testear `AddTodoForm`

`AddTodoForm` tiene estado interno (`useState`) y un callback `onAdd`. Probamos tres cosas: que envía el texto, que limpia el input y que ignora entradas vacías. Construye el spec:

```tsx
// src/components/AddTodoForm.cy.tsx
import { AddTodoForm } from './AddTodoForm'

describe('<AddTodoForm />', () => {
  it('llama a onAdd con el texto escrito y limpia el input', () => {
    const onAdd = cy.stub().as('onAdd')
    cy.mount(<AddTodoForm onAdd={onAdd} />)

    cy.get('[data-cy=new-todo-input]').type('Estudiar Cypress')
    cy.get('[data-cy=add-button]').click()

    // Se invocó con el texto exacto…
    cy.get('@onAdd').should('have.been.calledOnceWith', 'Estudiar Cypress')
    // …y el input quedó vacío tras enviar.
    cy.get('[data-cy=new-todo-input]').should('have.value', '')
  })

  it('no llama a onAdd si el input está vacío o solo tiene espacios', () => {
    const onAdd = cy.stub().as('onAdd')
    cy.mount(<AddTodoForm onAdd={onAdd} />)

    // Solo espacios → submit con Enter.
    cy.get('[data-cy=new-todo-input]').type('   {enter}')
    cy.get('@onAdd').should('not.have.been.called')
  })

  it('recorta espacios alrededor del texto', () => {
    const onAdd = cy.stub().as('onAdd')
    cy.mount(<AddTodoForm onAdd={onAdd} />)

    cy.get('[data-cy=new-todo-input]').type('  Pasear al perro  {enter}')
    cy.get('@onAdd').should('have.been.calledOnceWith', 'Pasear al perro')
  })
})
```

Observa cómo, sin tocar la API ni el enrutado, hemos cubierto el **contrato del componente**: qué hace con cada entrada y qué emite hacia su padre. Fíjate en el tercer test: envía con la tecla **`{enter}`** en lugar del botón, lo que también prueba el `onSubmit` del formulario.

---

## Paso 6 — ¿Component Testing, E2E o Vitest + RTL?

Las tres herramientas se solapan, pero cada una brilla en un nicho. Esta tabla es deliberadamente honesta:

| Criterio | Cypress E2E | Cypress Component | Vitest + RTL |
|----------|-------------|-------------------|--------------|
| Entorno | Navegador real, app completa | Navegador real, 1 componente | jsdom (DOM simulado) |
| Velocidad | Lenta (segundos) | Media | Muy rápida (ms) |
| Fidelidad visual (CSS real) | Alta | Alta | Baja (no renderiza estilos) |
| Cobertura de integración | Total (router, API, sesión) | Solo el componente | Componentes + hooks |
| Coste de mantenimiento | Alto | Medio | Bajo |
| Depuración | Time-travel + DevTools | Time-travel + DevTools | Logs / `--ui` |
| Cantidad ideal | Pocos | Algunos | Muchos |

Recomendación práctica:

- **Vitest + RTL** para el grueso de los tests de componentes y hooks: son baratos, rápidos y cubren la lógica. Es lo que más vas a tener.
- **Cypress Component Testing** cuando el CSS o el comportamiento del navegador real importan (medidas, scroll, focus, overflow) y jsdom se queda corto.
- **Cypress E2E** para unos pocos recorridos críticos de extremo a extremo (login → CRUD → logout), como el del módulo 8.

```
        Vitest + RTL  ███████████  (muchos, rápidos)
   Cypress Component  ████         (algunos, visuales)
         Cypress E2E  ██           (pocos, recorridos)
```

> No es "Cypress *contra* Vitest", sino "Cypress *con* Vitest". Cada nivel de la pirámide aporta una confianza distinta.

---

### Prueba esto

- Inicializa Component Testing con `npx cypress open --component` y deja que Cypress genere `support/component.ts` y el `component-index.html`.
- Escribe un test de `TodoItem` que verifique que el checkbox NO está marcado cuando `completed` es `false`.
- Cambia `cy.stub()` por `cy.spy()` en un callback real y observa la diferencia (el spy deja pasar la llamada original).
- Añade a `AddTodoForm.cy.tsx` un test que envíe el formulario con la tecla Enter (`{enter}`) en lugar del botón.
- Compara el tiempo de un mismo test de `TodoItem` en Cypress CT frente a Vitest + RTL.
- Monta `TodoItem` con un texto muy largo y comprueba visualmente (captura) cómo se comporta el layout real.

---

## Ejercicios propuestos

### Básico

**B1 — Texto distinto.** Monta `TodoItem` con `text: 'Leer un libro'` y verifica con `contain.text` que ese texto aparece. Cambia el id a `42` y confirma que el render no se rompe.

**B2 — Botón presente.** Escribe un `it` que monte `AddTodoForm` y verifique que existen tanto `[data-cy=new-todo-input]` como `[data-cy=add-button]` (`should('exist')`), sin escribir nada todavía.

### Intermedio

**I1 — Doble toggle.** Monta `TodoItem` con un stub `onToggle`, marca y desmarca el checkbox dos veces, y verifica con `should('have.been.calledTwice')` que el callback se invocó dos veces.

**I2 — Texto preservado al fallar el envío.** Monta `AddTodoForm`, escribe solo espacios y pulsa el botón. Verifica que `onAdd` NO se llamó y que el input conserva exactamente lo escrito (`have.value`), comprobando que el componente no limpia ante entradas inválidas.

### Avanzado

**A1 — Spy sobre callback real.** Crea una pequeña función real `manejarToggle(id)` que haga `console.log`, envuélvela con `cy.spy(obj, 'manejarToggle').as('toggle')`, móntala como prop de `TodoItem` y verifica que el spy registró la llamada **y** dejó pasar la lógica original.

**A2 — Wrapper con estado.** Crea un componente de prueba que envuelva `AddTodoForm` y mantenga una lista en `useState`, agregando cada texto recibido por `onAdd`. Móntalo con `cy.mount`, escribe dos tareas y verifica en el DOM del wrapper que ambas aparecen renderizadas.

---

## Resumen del módulo 9

- Cypress ofrece **dos modos**: E2E (app completa) y **Component Testing** (un componente aislado), ambos en navegador real.
- Se configura **por pasos**: la sección **`component`** en `cypress.config.ts` (`framework: 'react'`, `bundler: 'vite'`), el comando `cy.mount` en `support/component.ts` y el `component-index.html`.
- Los componentes se renderizan con **`cy.mount(<Componente ... />)`**, pasándole las props directamente: primero lo visual, luego el comportamiento.
- Espiamos callbacks con **`cy.stub().as('nombre')`** y afirmamos con `cy.get('@nombre').should('have.been.calledOnceWith', ...)` (Sinon).
- Probamos `TodoItem` (toggle y delete) y `AddTodoForm` (envío, limpieza del input y rechazo de vacíos) sin tocar API ni router.
- La decisión **CT vs E2E vs Vitest+RTL** depende de fidelidad visual, velocidad y nivel de integración: muchos Vitest, algunos CT, pocos E2E.

> **Siguiente página →** Módulo 10: Buenas prácticas y CI — organización de specs, anti-patrones, independencia de tests, capturas y vídeo en fallos, ejecución headless y GitHub Actions con `cypress-io/github-action`.
