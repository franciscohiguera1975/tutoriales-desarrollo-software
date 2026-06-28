# Testing en React con Vitest — Página 7
## Módulo 7 · Hooks personalizados
### Probar lógica reutilizable con `renderHook`, `act` y `result.current`

---

## Cómo trabajar este módulo

Este módulo es **incremental**: construiremos las suites de `useTodos` y `useAuth`
**paso a paso**, una operación de estado a la vez. No copies los archivos finales de
golpe — añade **un bloque por paso**, guarda y ejecuta `npm run test:watch` para ver
al instante cómo cada `act` mueve el estado. Entre paso y paso hay retos
**🔧 Tu turno** que debes resolver **antes** de mirar la solución colapsable.

```
Paso 1  →  renderHook y el estado inicial
Paso 2  →  act() para agregar (add)
Paso 3  →  toggle y remove
Paso 4  →  tú pruebas el filtro derivado (visibleTodos)
Paso 5  →  useAuth: login y logout
Paso 6  →  tú pruebas hooks con efectos (localStorage)
```

---

## ¿Por qué probar los hooks por separado?

Un hook personalizado encapsula lógica de estado y efectos sin UI. Si pruebas esa
lógica **solo** a través de componentes, mezclas dos preocupaciones: el render y la
lógica. Probar el hook de forma aislada te da tests rápidos, enfocados y fáciles de
mantener.

`@testing-library/react` incluye `renderHook`, que monta el hook dentro de un
componente mínimo y te da acceso a su valor de retorno a través de `result.current`.

```
   renderHook(() => useTodos())
        │
        ▼
   [ componente invisible que llama al hook ]
        │
        ▼
   result.current  ◀── el valor que devuelve el hook ahora mismo
```

---

## El hook bajo prueba: `useTodos`

Antes de testear, necesitamos el hook. Esta es la firma compartida del curso:

```tsx
// src/hooks/useTodos.ts
import { useState, useMemo, useCallback } from 'react'
import type { Todo, Filter } from '../types'

let nextId = 0
function genId(): string {
  // id incremental simple para el ejemplo
  return `todo-${++nextId}`
}

export function useTodos(initial: Todo[] = []) {
  const [todos, setTodos] = useState<Todo[]>(initial)
  const [filter, setFilter] = useState<Filter>('all')

  const add = useCallback((text: string) => {
    setTodos((prev) => [...prev, { id: genId(), text, completed: false }])
  }, [])

  const toggle = useCallback((id: string) => {
    setTodos((prev) =>
      prev.map((t) => (t.id === id ? { ...t, completed: !t.completed } : t)),
    )
  }, [])

  const remove = useCallback((id: string) => {
    setTodos((prev) => prev.filter((t) => t.id !== id))
  }, [])

  // lista derivada según el filtro activo
  const visibleTodos = useMemo(() => {
    if (filter === 'active') return todos.filter((t) => !t.completed)
    if (filter === 'completed') return todos.filter((t) => t.completed)
    return todos
  }, [todos, filter])

  return { todos, visibleTodos, filter, setFilter, add, toggle, remove }
}
```

---

## Paso 1 — `renderHook` y el estado inicial

`renderHook(() => useTodos())` monta el hook y expone su retorno en `result.current`.
Empezamos por lo más simple: comprobar el estado de arranque. Crea el archivo con
**solo estos dos casos**:

```tsx
// src/hooks/useTodos.test.tsx
import { renderHook } from '@testing-library/react'
import { describe, it, expect } from 'vitest'
import type { Todo } from '../types'
import { useTodos } from './useTodos'

describe('useTodos', () => {
  it('arranca con lista vacía y filtro "all"', () => {
    const { result } = renderHook(() => useTodos())
    //      ^^^^^^ objeto con .current (snapshot del retorno del hook)

    // result.current contiene exactamente lo que el hook retorna
    expect(result.current.todos).toEqual([])
    expect(result.current.filter).toBe('all')
  })

  it('respeta el estado inicial provisto', () => {
    const inicial: Todo[] = [{ id: 'x', text: 'Pre-cargada', completed: false }]
    const { result } = renderHook(() => useTodos(inicial))

    expect(result.current.todos).toHaveLength(1)
    expect(result.current.todos[0].text).toBe('Pre-cargada')
  })
})
```

Guarda: **2 passed**. `renderHook` devuelve más de lo que usamos aquí:

| Propiedad | Qué es |
|---|---|
| `result.current` | El valor que el hook retorna **ahora mismo** |
| `rerender(props)` | Vuelve a ejecutar el hook (útil para hooks con argumentos) |
| `unmount()` | Desmonta el componente (dispara cleanups de efectos) |

> **`result.current` es una "foto"**: después de cada actualización de estado debes
> volver a leerlo. No guardes una referencia vieja en una variable.

---

## Paso 2 — `act` para agregar una tarea

Cuando llamas a una función que actualiza el estado del hook (como `add`), React
procesa el re-render. Para que el test vea ese cambio de forma determinista, la
llamada debe ir dentro de `act`. Añade el import de `act` y este caso:

```tsx
// src/hooks/useTodos.test.tsx  (actualiza el import y añade el it)
import { act, renderHook } from '@testing-library/react'

  it('add agrega una tarea no completada', () => {
    const { result } = renderHook(() => useTodos())

    // act() agrupa la actualización de estado y aplica el re-render
    act(() => {
      result.current.add('Estudiar Vitest')
    })

    // tras act, result.current refleja el nuevo estado
    expect(result.current.todos).toHaveLength(1)
    expect(result.current.todos[0]).toMatchObject({
      text: 'Estudiar Vitest',
      completed: false,
    })
  })
```

Guarda: **3 passed**.

### 🔧 Tu turno #1

Sin mirar abajo: **quita el `act`** y llama a `result.current.add('Estudiar Vitest')`
directamente. Ejecuta y observa la consola de Vitest. ¿Qué cambia?

<details><summary>✅ Solución / qué observar</summary>

Vitest imprime un warning: *"An update to TestComponent inside a test was not wrapped
in act(...)"*. Según el timing, la aserción puede incluso fallar porque
`result.current` todavía apunta a la "foto" anterior al re-render. La lección:
**toda actualización de estado va dentro de `act`**. Para actualizaciones asíncronas
se usa `await act(async () => { ... })`. Vuelve a envolver la llamada y el warning
desaparece.

</details>

---

## Paso 3 — `toggle` y `remove`

Ahora las otras dos operaciones. `toggle` alterna `completed`; `remove` elimina por
`id`. Añade ambos casos dentro del `describe`:

```tsx
// src/hooks/useTodos.test.tsx  (añade dentro del describe)
  it('toggle alterna el campo completed', () => {
    const inicial: Todo[] = [{ id: 'a', text: 'Tarea', completed: false }]
    const { result } = renderHook(() => useTodos(inicial))

    act(() => result.current.toggle('a'))
    expect(result.current.todos[0].completed).toBe(true)

    // segundo toggle vuelve al estado original
    act(() => result.current.toggle('a'))
    expect(result.current.todos[0].completed).toBe(false)
  })

  it('remove elimina la tarea por id', () => {
    const inicial: Todo[] = [
      { id: 'a', text: 'Una', completed: false },
      { id: 'b', text: 'Dos', completed: false },
    ]
    const { result } = renderHook(() => useTodos(inicial))

    act(() => result.current.remove('a'))

    expect(result.current.todos).toHaveLength(1)
    expect(result.current.todos[0].id).toBe('b')
  })
```

Guarda: **5 passed**. Nota cómo en `toggle` leemos `result.current` **dos veces**,
una tras cada `act`: es la "foto" actualizada, no una referencia guardada.

---

## Paso 4 — 🔧 Tu turno #2: el filtro derivado

`visibleTodos` se recalcula con `useMemo` según `filter`. Escribe tú mismo un nuevo
`describe('useTodos — filtro', ...)` con un helper `setup()` y **tres** casos:
filtro `'all'` muestra las 2 tareas, `'active'` muestra solo la no completada, y
`'completed'` solo la completada.

**Pistas:**
- El `setup()` debe devolver `renderHook(() => useTodos(inicial))` con una tarea
  activa y otra completada.
- Cambia el filtro con `act(() => result.current.setFilter('active'))`.
- Afirma sobre `result.current.visibleTodos`.

<details><summary>✅ Solución</summary>

```tsx
// src/hooks/useTodos.test.tsx  (nuevo describe)
describe('useTodos — filtro', () => {
  function setup() {
    const inicial: Todo[] = [
      { id: '1', text: 'Activa', completed: false },
      { id: '2', text: 'Hecha', completed: true },
    ]
    return renderHook(() => useTodos(inicial))
  }

  it('filtro "all" muestra todas', () => {
    const { result } = setup()
    expect(result.current.visibleTodos).toHaveLength(2)
  })

  it('filtro "active" muestra solo las no completadas', () => {
    const { result } = setup()

    act(() => result.current.setFilter('active'))

    expect(result.current.visibleTodos).toHaveLength(1)
    expect(result.current.visibleTodos[0].text).toBe('Activa')
  })

  it('filtro "completed" muestra solo las completadas', () => {
    const { result } = setup()

    act(() => result.current.setFilter('completed'))

    expect(result.current.visibleTodos).toHaveLength(1)
    expect(result.current.visibleTodos[0].text).toBe('Hecha')
  })
})
```

El `useMemo` recalcula `visibleTodos` en cuanto cambia `filter`; como el cambio va en
`act`, la "foto" ya está actualizada cuando afirmamos.

</details>

---

## Paso 5 — `useAuth`: login y logout

Cambiamos de hook. `useAuth` maneja el inicio y cierre de sesión: `login` setea el
usuario; `logout` lo limpia.

```tsx
// src/hooks/useAuth.ts
import { useState, useCallback } from 'react'
import type { User } from '../types'

export function useAuth() {
  const [user, setUser] = useState<User | null>(null)

  const login = useCallback((name: string) => {
    // simulación: generamos un usuario a partir del nombre
    setUser({ id: crypto.randomUUID(), name })
  }, [])

  const logout = useCallback(() => {
    setUser(null)
  }, [])

  const isAuthenticated = user !== null

  return { user, isAuthenticated, login, logout }
}
```

Crea la suite paso a paso. Primero el estado inicial y el login:

```tsx
// src/hooks/useAuth.test.tsx
import { act, renderHook } from '@testing-library/react'
import { describe, it, expect } from 'vitest'
import { useAuth } from './useAuth'

describe('useAuth', () => {
  it('arranca sin usuario autenticado', () => {
    const { result } = renderHook(() => useAuth())

    expect(result.current.user).toBeNull()
    expect(result.current.isAuthenticated).toBe(false)
  })

  it('login setea el usuario y marca como autenticado', () => {
    const { result } = renderHook(() => useAuth())

    act(() => result.current.login('Ada'))

    expect(result.current.user).not.toBeNull()
    expect(result.current.user?.name).toBe('Ada')
    expect(result.current.isAuthenticated).toBe(true)
  })

  it('logout limpia el usuario', () => {
    const { result } = renderHook(() => useAuth())

    act(() => result.current.login('Ada'))
    expect(result.current.isAuthenticated).toBe(true)

    act(() => result.current.logout())

    expect(result.current.user).toBeNull()
    expect(result.current.isAuthenticated).toBe(false)
  })
})
```

Guarda: **3 passed**. El tercer caso encadena dos `act` (login, luego logout) y
verifica el estado tras cada uno.

---

## Paso 6 — 🔧 Tu turno #3: hooks con efectos (`useEffect`)

Cuando un hook tiene efectos, `renderHook` los ejecuta tras montar. Este hook
persiste el filtro en `localStorage` (jsdom incluye un `localStorage` funcional):

```tsx
// src/hooks/usePersistedFilter.ts
import { useState, useEffect } from 'react'
import type { Filter } from '../types'

export function usePersistedFilter() {
  const [filter, setFilter] = useState<Filter>(
    () => (localStorage.getItem('filter') as Filter) ?? 'all',
  )

  // efecto: cada cambio de filtro se guarda en localStorage
  useEffect(() => {
    localStorage.setItem('filter', filter)
  }, [filter])

  return { filter, setFilter }
}
```

Escribe tú mismo la suite con **tres** casos: lee el valor inicial desde
`localStorage`, persiste el filtro al cambiarlo, y no lanza al desmontar.

**Pistas:**
- Usa `beforeEach(() => localStorage.clear())` para aislar los tests.
- Para el caso inicial, haz `localStorage.setItem('filter', 'completed')` **antes** de
  `renderHook`.
- Cambia el filtro con `act(() => result.current.setFilter('active'))` y luego lee
  `localStorage.getItem('filter')`.
- Para el cleanup, usa `const { unmount } = renderHook(...)` y
  `expect(() => unmount()).not.toThrow()`.

<details><summary>✅ Solución</summary>

```tsx
// src/hooks/usePersistedFilter.test.tsx
import { act, renderHook } from '@testing-library/react'
import { describe, it, expect, beforeEach } from 'vitest'
import { usePersistedFilter } from './usePersistedFilter'

describe('usePersistedFilter', () => {
  beforeEach(() => {
    // jsdom incluye localStorage; lo limpiamos entre tests
    localStorage.clear()
  })

  it('lee el valor inicial desde localStorage', () => {
    localStorage.setItem('filter', 'completed')

    const { result } = renderHook(() => usePersistedFilter())

    expect(result.current.filter).toBe('completed')
  })

  it('persiste el filtro cuando cambia (efecto)', () => {
    const { result } = renderHook(() => usePersistedFilter())

    act(() => result.current.setFilter('active'))

    // el useEffect ya corrió tras el re-render dentro de act
    expect(localStorage.getItem('filter')).toBe('active')
  })

  it('ejecuta cleanups al desmontar', () => {
    const { unmount } = renderHook(() => usePersistedFilter())
    // unmount dispara los cleanups de useEffect (si los hubiera)
    expect(() => unmount()).not.toThrow()
  })
})
```

El `useEffect` corre tras el re-render **dentro** de `act`, así que `localStorage` ya
está actualizado cuando afirmamos. El `beforeEach` evita que un test contamine al
siguiente.

</details>

---

## `renderHook` con argumentos cambiantes

Si el hook recibe argumentos, usa la opción `initialProps` y `rerender` para
simular cambios:

```tsx
// ejemplo ilustrativo
const { result, rerender } = renderHook(
  ({ initial }) => useTodos(initial),
  { initialProps: { initial: [] as Todo[] } },
)

// re-ejecuta el hook con otros argumentos
rerender({ initial: [{ id: 'z', text: 'Nueva', completed: false }] })
```

---

### Prueba esto

- Quita el `act` de un test de `add` y observa el warning de React en la consola de
  Vitest.
- Añade un método `clearCompleted` a `useTodos` y escribe su test (debe eliminar solo
  las completadas).
- Verifica que las funciones `add`, `toggle`, `remove` mantienen la misma referencia
  entre renders (gracias a `useCallback`).
- Prueba `useAuth` haciendo login dos veces seguidas y comprobando que el segundo
  usuario reemplaza al primero.
- Usa `initialProps` y `rerender` para probar `useTodos` con distintos estados
  iniciales.
- Escribe un test que use `unmount()` y confirme que un efecto de limpieza se
  ejecutó (por ejemplo, un `vi.fn()` registrado como cleanup).

---

## Ejercicios propuestos

### Básico

**B1 — Add múltiple.** Escribe un `it` que llame a `add` tres veces (cada una en su
propio `act`) y verifique que `result.current.todos` tiene longitud 3 y que los
textos coinciden en orden.

**B2 — isAuthenticated tras logout.** Escribe un `it` para `useAuth` que haga login,
afirme `isAuthenticated === true`, haga logout y afirme `user === null` y
`isAuthenticated === false`.

### Intermedio

**I1 — `clearCompleted`.** Diseña (en papel) un método `clearCompleted` para
`useTodos` que elimine solo las tareas completadas. Escribe un test que parta de dos
tareas (una completada, una activa), lo llame dentro de `act` y verifique que queda
solo la activa.

**I2 — Referencias estables.** Escribe un test que guarde `result.current.add` en una
variable, fuerce un re-render (por ejemplo con un `act` que cambie el filtro) y
verifique con `toBe` que `result.current.add` sigue siendo **la misma** referencia
gracias a `useCallback`.

### Avanzado

**A1 — Filtro y operaciones combinados.** Escribe un test que, sobre `useTodos`,
agregue tres tareas, marque una como completada con `toggle`, ponga el filtro en
`'completed'` y verifique que `visibleTodos` contiene exactamente esa tarea.

**A2 — `initialProps` y `rerender`.** Usa `renderHook(({ initial }) => useTodos(initial), { initialProps })`
para arrancar con lista vacía, luego `rerender` con un estado inicial distinto y
razona (con un comentario en el test) por qué `todos` **no** cambia: `useState` solo
usa el argumento inicial en el primer render.

---

## Resumen del módulo 7

- `renderHook` monta un hook en un componente invisible y expone su retorno en
  `result.current`.
- `result.current` es una foto del estado actual: vuelve a leerlo tras cada
  actualización, no guardes referencias viejas.
- Envuelve toda actualización de estado en `act(() => {})`; para asíncrono usa
  `await act(async () => {})`.
- Construimos las suites **por pasos**: estado inicial, `add`, `toggle`/`remove`, el
  filtro derivado que escribiste tú, `useAuth` (login/logout) y un hook con efectos.
- Los efectos (`useEffect`) corren tras montar y dentro de `act`; `unmount()` dispara
  sus cleanups.
- Para hooks con argumentos, usa `initialProps` y `rerender`.

> **Siguiente página →** Módulo 8: mocking con `vi.fn()`, `vi.spyOn()` y `vi.mock()` para aislar dependencias.
