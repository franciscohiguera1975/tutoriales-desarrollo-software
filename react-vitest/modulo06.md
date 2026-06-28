# Testing en React con Vitest — Página 6
## Módulo 6 · Props, estado y renderizado
### Tipar props, probar renderizado condicional, listas y re-render con `rerender`

---

## Cómo trabajar este módulo

Este módulo es **incremental**: construiremos las suites de `TodoList` y
`TodoFilter` **paso a paso**, una rama de comportamiento a la vez. No copies los
archivos finales de golpe — añade **un bloque por paso**, guarda y ejecuta
`npm run test:watch` para ver al instante cómo crece la suite. Entre paso y paso hay
retos **🔧 Tu turno** que debes resolver **antes** de mirar la solución colapsable.

```
Paso 1  →  rama "vacía": estado vacío
Paso 2  →  rama "con datos": hay lista
Paso 3  →  contar items con getAllByRole
Paso 4  →  tú pruebas el emptyMessage personalizado
Paso 5  →  reaccionar a props nuevas con rerender
Paso 6  →  tú pruebas TodoFilter (aria-pressed)
```

---

## El componente como función de sus props

Un componente de React, en su forma más pura, es una función: recibe props y
devuelve UI. Probar componentes consiste en gran medida en afirmar **"dado este
conjunto de props, el DOM resultante es este"**.

```
   props  ─────▶  [ Componente ]  ─────▶  DOM renderizado
   (entrada)                              (lo que afirmamos)
```

En este módulo veremos cómo tipar props (requeridas vs opcionales), cómo probar el
renderizado condicional (estado vacío vs con datos), cómo contar elementos de una
lista y cómo verificar que un componente reacciona a un cambio de props con
`rerender`.

---

## Tipar props: requeridas vs opcionales

TypeScript distingue entre props **obligatorias** y **opcionales** mediante el
modificador `?`. Esto se traduce en garantías que el test puede explotar. Este es el
componente que probaremos:

```tsx
// src/components/TodoList.tsx
import type { Todo } from '../types'
import { TodoItem } from './TodoItem'

interface TodoListProps {
  todos: Todo[] // requerida: siempre debe pasarse
  onToggle: (id: string) => void // requerida
  onDelete: (id: string) => void // requerida
  emptyMessage?: string // opcional: con ? puede omitirse
}

export function TodoList({
  todos,
  onToggle,
  onDelete,
  emptyMessage = 'No hay tareas todavía', // valor por defecto para la opcional
}: TodoListProps) {
  if (todos.length === 0) {
    return <p role="status">{emptyMessage}</p>
  }

  return (
    <ul>
      {todos.map((todo) => (
        <TodoItem
          key={todo.id} // key estable: clave para el reconciliador
          todo={todo}
          onToggle={onToggle}
          onDelete={onDelete}
        />
      ))}
    </ul>
  )
}
```

| Modificador | Significado | En el componente |
|---|---|---|
| `todos: Todo[]` | Requerida | TS obliga a pasarla; si falta, error de compilación |
| `emptyMessage?: string` | Opcional | Puede omitirse; conviene dar valor por defecto |
| `onToggle: (id) => void` | Requerida (callback) | El padre debe proveer la lógica |

> **Ventaja para el test**: si una prop es requerida, TypeScript te avisa en tiempo
> de compilación cuando un test olvida pasarla, antes incluso de ejecutar Vitest.

---

## Paso 1 — Rama "vacía": probar el estado vacío

`TodoList` tiene dos caminos. Empezamos por el más simple: si no hay tareas, muestra
un mensaje y **ninguna** lista. Crea el archivo de test con **solo este caso**:

```tsx
// src/components/TodoList.test.tsx
import { render, screen } from '@testing-library/react'
import { describe, it, expect, vi } from 'vitest'
import type { Todo } from '../types'
import { TodoList } from './TodoList'

const noop = vi.fn() // callback vacío reutilizable

describe('TodoList — renderizado condicional', () => {
  it('muestra el estado vacío cuando no hay tareas', () => {
    render(<TodoList todos={[]} onToggle={noop} onDelete={noop} />)

    expect(screen.getByText('No hay tareas todavía')).toBeInTheDocument()
    // no debe existir ninguna lista
    expect(screen.queryByRole('list')).not.toBeInTheDocument()
  })
})
```

Guarda: **1 passed**. La clave aquí es la diferencia entre dos familias de queries:

> **`getBy` vs `queryBy`**: usa `getBy*` cuando esperas que el elemento exista (lanza
> error si no está). Usa `queryBy*` para afirmar **ausencia**, porque devuelve `null`
> en vez de lanzar.

### 🔧 Tu turno #1

Sin mirar abajo: **rompe el test a propósito**. Cambia la segunda aserción de
`queryByRole('list')` a `getByRole('list')` (manteniendo el `.not.toBeInTheDocument()`).
Predice qué error verás.

<details><summary>✅ Solución / qué observar</summary>

El test falla **dentro de la propia query**, no en el matcher: `getByRole('list')`
lanza `Unable to find an accessible element with the role "list"` antes de llegar a
`.not.toBeInTheDocument()`. Esa es la lección: para afirmar **ausencia** se usa
`queryBy*`, que devuelve `null` en lugar de lanzar. Vuelve a `queryByRole` y el test
pasa.

</details>

---

## Paso 2 — Rama "con datos": probar que aparece la lista

Ahora el otro camino: cuando hay tareas, aparece la `<ul>` y desaparece el estado
vacío. Añade este caso dentro del mismo `describe`:

```tsx
// src/components/TodoList.test.tsx  (añade dentro del describe)
  it('muestra la lista cuando hay tareas y oculta el estado vacío', () => {
    const todos: Todo[] = [{ id: '1', text: 'Tarea uno', completed: false }]
    render(<TodoList todos={todos} onToggle={noop} onDelete={noop} />)

    expect(screen.getByRole('list')).toBeInTheDocument()
    expect(screen.queryByText(/no hay tareas/i)).not.toBeInTheDocument()
  })
```

Guarda: **2 passed**. Hemos cubierto **ambas ramas** del condicional. Probar las dos
caras de un `if` es una de las disciplinas más rentables del testing.

---

## Paso 3 — Contar items con `getAllByRole`

Cuando renderizamos una lista, queremos verificar **cuántos** elementos aparecen.
Cada `<li>` tiene el rol `listitem`, así que `getAllByRole('listitem')` devuelve un
array que podemos medir. Añade un segundo `describe` para los tests de lista:

```tsx
// src/components/TodoList.test.tsx  (añade un nuevo describe)
describe('TodoList — renderizado de listas', () => {
  const todos: Todo[] = [
    { id: '1', text: 'Comprar pan', completed: false },
    { id: '2', text: 'Lavar ropa', completed: true },
    { id: '3', text: 'Llamar al banco', completed: false },
  ]

  it('renderiza un listitem por cada tarea', () => {
    render(<TodoList todos={todos} onToggle={noop} onDelete={noop} />)

    // getAllByRole devuelve TODOS los elementos con ese rol
    const items = screen.getAllByRole('listitem')
    expect(items).toHaveLength(3)
  })

  it('muestra el texto de cada tarea', () => {
    render(<TodoList todos={todos} onToggle={noop} onDelete={noop} />)

    expect(screen.getByText('Comprar pan')).toBeInTheDocument()
    expect(screen.getByText('Lavar ropa')).toBeInTheDocument()
    expect(screen.getByText('Llamar al banco')).toBeInTheDocument()
  })
})
```

Guarda: **4 passed**. Estas son las queries por rol más útiles:

| Elemento HTML | Rol implícito | Query típica |
|---|---|---|
| `<ul>` / `<ol>` | `list` | `getByRole('list')` |
| `<li>` | `listitem` | `getAllByRole('listitem')` |
| `<button>` | `button` | `getByRole('button', { name })` |
| `<input type="checkbox">` | `checkbox` | `getByRole('checkbox')` |
| `<a href>` | `link` | `getByRole('link')` |
| `<input type="text">` | `textbox` | `getByRole('textbox')` |

---

## La importancia de `key`

Cuando renderizas una lista con `.map()`, React necesita una `key` **estable y
única** por elemento. La `key` no es un atributo del DOM: es la pista que usa el
reconciliador para saber qué elemento corresponde a qué dato entre renders.

```
   Antes              Después (se borra "B")
   ------             ----------------------
   key="a" → A        key="a" → A   (se reutiliza)
   key="b" → B        key="c" → C   (se reutiliza, NO se recrea)
   key="c" → C
```

Si usaras el **índice** como `key`, al borrar un elemento intermedio React podría
confundir qué fila cambió, provocando bugs sutiles (estado de inputs que "salta" de
fila). Por eso usamos `todo.id`:

```tsx
{todos.map((todo) => (
  <TodoItem key={todo.id} todo={todo} onToggle={onToggle} onDelete={onDelete} />
  //         ^^^^^^^^^^^^ id estable, NO el índice del array
))}
```

> **Regla**: usa un identificador estable del dato (`id`) como `key`, no el índice
> del array, salvo en listas que nunca cambian de orden ni se filtran.

---

## Paso 4 — 🔧 Tu turno #2: el `emptyMessage` personalizado

`emptyMessage` es una prop **opcional** con valor por defecto. Escribe tú mismo un
`it` (dentro del primer `describe`, "renderizado condicional") que pase un
`emptyMessage="¡Todo limpio!"` con una lista vacía y verifique que ese texto
personalizado aparece.

**Pistas:**
- Pasa `todos={[]}` y `emptyMessage="¡Todo limpio!"`.
- Usa `getByText('¡Todo limpio!')`.

<details><summary>✅ Solución</summary>

```tsx
  it('usa el emptyMessage personalizado si se provee', () => {
    render(
      <TodoList
        todos={[]}
        onToggle={noop}
        onDelete={noop}
        emptyMessage="¡Todo limpio!"
      />,
    )

    expect(screen.getByText('¡Todo limpio!')).toBeInTheDocument()
  })
```

Como `emptyMessage` es opcional, TypeScript te deja omitirla (usa el valor por
defecto) o pasarla. Aquí probamos que, cuando se provee, **gana** sobre el
predeterminado.

</details>

---

## Paso 5 — Reaccionar a props nuevas con `rerender`

`render()` devuelve una función `rerender` que vuelve a montar el **mismo**
componente con props distintas, conservando el árbol. Es la forma correcta de probar
que un componente reacciona a cambios de props. Añade estos casos:

```tsx
// src/components/TodoList.test.tsx  (añade donde corresponda)
it('actualiza la cantidad de items al cambiar las props', () => {
  const base: Todo[] = [{ id: '1', text: 'Una', completed: false }]

  const { rerender } = render(
    <TodoList todos={base} onToggle={noop} onDelete={noop} />,
  )
  expect(screen.getAllByRole('listitem')).toHaveLength(1)

  // mismas claves de props, nuevos datos
  const masTodos: Todo[] = [...base, { id: '2', text: 'Dos', completed: false }]
  rerender(<TodoList todos={masTodos} onToggle={noop} onDelete={noop} />)

  expect(screen.getAllByRole('listitem')).toHaveLength(2)
})

it('pasa de tener items a estado vacío con rerender', () => {
  const { rerender } = render(
    <TodoList
      todos={[{ id: '1', text: 'Una', completed: false }]}
      onToggle={noop}
      onDelete={noop}
    />,
  )
  expect(screen.getByRole('list')).toBeInTheDocument()

  rerender(<TodoList todos={[]} onToggle={noop} onDelete={noop} />)

  expect(screen.queryByRole('list')).not.toBeInTheDocument()
  expect(screen.getByRole('status')).toBeInTheDocument()
})
```

> **No uses `render()` dos veces** para simular un cambio de props: eso monta dos
> árboles independientes. `rerender` actualiza el árbol existente, igual que React en
> producción.

---

## Paso 6 — 🔧 Tu turno #3: probar `TodoFilter`

`TodoFilter` recibe el filtro actual (`current`) y un callback `onChange`. Debe
**resaltar** el botón del filtro activo vía `aria-pressed`. Este es el componente:

```tsx
// src/components/TodoFilter.tsx
import type { Filter } from '../types'

interface TodoFilterProps {
  current: Filter
  onChange: (filter: Filter) => void
}

const FILTERS: { value: Filter; label: string }[] = [
  { value: 'all', label: 'Todas' },
  { value: 'active', label: 'Activas' },
  { value: 'completed', label: 'Completadas' },
]

export function TodoFilter({ current, onChange }: TodoFilterProps) {
  return (
    <div role="group" aria-label="Filtros">
      {FILTERS.map(({ value, label }) => (
        <button
          key={value}
          onClick={() => onChange(value)}
          // aria-pressed indica visual y semánticamente el filtro activo
          aria-pressed={current === value}
        >
          {label}
        </button>
      ))}
    </div>
  )
}
```

Escribe tú mismo **dos** `it` para `TodoFilter`:
1. Con `current="active"`, el botón "Activas" tiene `aria-pressed="true"` y "Todas"
   lo tiene en `"false"`.
2. Al hacer click en "Completadas", `onChange` se llama con `'completed'`.

**Pistas:**
- Verifica el atributo con `toHaveAttribute('aria-pressed', 'true')`.
- Para el segundo, recuerda `userEvent.setup()` + `await user.click(...)`.

<details><summary>✅ Solución</summary>

```tsx
// src/components/TodoFilter.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, it, expect, vi } from 'vitest'
import type { Filter } from '../types'
import { TodoFilter } from './TodoFilter'

describe('TodoFilter', () => {
  it('marca como activo el filtro indicado por current', () => {
    render(<TodoFilter current="active" onChange={vi.fn()} />)

    expect(screen.getByRole('button', { name: 'Activas' })).toHaveAttribute(
      'aria-pressed',
      'true',
    )
    // los otros no están presionados
    expect(screen.getByRole('button', { name: 'Todas' })).toHaveAttribute(
      'aria-pressed',
      'false',
    )
  })

  it('llama a onChange con el filtro elegido', async () => {
    const user = userEvent.setup()
    const onChange = vi.fn()
    render(<TodoFilter current="all" onChange={onChange} />)

    await user.click(screen.getByRole('button', { name: 'Completadas' }))

    expect(onChange).toHaveBeenCalledWith<[Filter]>('completed')
  })

  it('cambia el resaltado al re-renderizar con otro current', () => {
    const { rerender } = render(<TodoFilter current="all" onChange={vi.fn()} />)
    expect(screen.getByRole('button', { name: 'Todas' })).toHaveAttribute(
      'aria-pressed',
      'true',
    )

    rerender(<TodoFilter current="completed" onChange={vi.fn()} />)

    expect(screen.getByRole('button', { name: 'Completadas' })).toHaveAttribute(
      'aria-pressed',
      'true',
    )
    expect(screen.getByRole('button', { name: 'Todas' })).toHaveAttribute(
      'aria-pressed',
      'false',
    )
  })
})
```

El tercer caso, de regalo, combina lo aprendido en el Paso 5: usa `rerender` para
cambiar `current` y verificar que el resaltado se mueve.

</details>

---

### Prueba esto

- Elimina una prop requerida al llamar a `<TodoList />` en un test y observa el error
  de TypeScript antes de ejecutar nada.
- Cambia `key={todo.id}` por `key={index}` y crea un test donde se borra un item
  intermedio; reflexiona sobre por qué la `key` estable importa.
- Escribe un test que use `getAllByRole('listitem')` y luego verifique el texto de un
  item concreto con `within(items[1])`.
- Usa `rerender` para pasar `TodoFilter` por los tres valores de `Filter` y afirma el
  resaltado en cada caso.
- Prueba el `emptyMessage` por defecto y luego uno personalizado en `TodoList`.
- Añade una prop opcional `title?: string` a `TodoList` y testea tanto su presencia
  como su ausencia.

---

## Ejercicios propuestos

### Básico

**B1 — Contar checkboxes.** Escribe un `it` que renderice `TodoList` con tres tareas
y verifique con `getAllByRole('checkbox')` y `toHaveLength(3)` que hay tantos
checkboxes como tareas.

**B2 — Estado vacío por defecto.** Escribe un `it` que renderice `TodoList` con
`todos={[]}` **sin** pasar `emptyMessage` y verifique que aparece el texto por
defecto `'No hay tareas todavía'`.

### Intermedio

**I1 — `within` para un item concreto.** Renderiza tres tareas, obtén
`getAllByRole('listitem')` y usa `within(items[1])` para afirmar que el segundo item
contiene el texto de la segunda tarea y su propio checkbox.

**I2 — Todos los filtros con `rerender`.** Escribe un `it` que renderice `TodoFilter`
y, con un solo `rerender` por valor, recorra `'all'`, `'active'` y `'completed'`,
afirmando en cada paso que solo el botón correspondiente tiene `aria-pressed="true"`.

### Avanzado

**A1 — Prop opcional nueva.** Diseña (en papel) una prop opcional
`heading?: string` para `TodoList` que renderice un `<h2>{heading}</h2>` cuando se
provee. Escribe dos tests: uno que verifique su presencia con
`getByRole('heading')` y otro que confirme su ausencia con `queryByRole('heading')`
cuando se omite.

**A2 — Lista filtrada combinada.** Renderiza un `TodoList` con tareas mixtas
(completadas y activas) y un `TodoFilter` en el mismo árbol. Sin lógica de estado
real, usa `rerender` para simular el cambio de `todos` que produciría cada filtro y
verifica el número de `listitem` resultante en cada caso.

---

## Resumen del módulo 6

- Las props se tipan con interfaces; `?` marca opcionales, y conviene darles valor
  por defecto. TS verifica las requeridas en compilación.
- El renderizado condicional (vacío vs con datos) exige probar **ambas ramas**; usa
  `getBy*` para presencia y `queryBy*` para ausencia.
- `getAllByRole('listitem')` cuenta elementos de una lista; combínalo con
  `toHaveLength`.
- La `key` debe ser un identificador estable (`id`), no el índice, para que React
  reconcilie correctamente.
- `rerender` vuelve a renderizar el mismo árbol con props nuevas: es la forma de
  probar reacción a cambios de props.
- Construimos las suites **por pasos**: rama vacía, rama con datos, conteo, el
  `emptyMessage` que escribiste tú, `rerender` y finalmente `TodoFilter` con
  `aria-pressed`.

> **Siguiente página →** Módulo 7: probar hooks personalizados (`useTodos`, `useAuth`) con `renderHook` y `act`.
