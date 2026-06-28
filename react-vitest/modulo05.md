# Testing en React con Vitest — Página 5
## Módulo 5 · Interacción del usuario
### Simular clicks, escritura y teclado con `user-event` para probar formularios reales

---

## Cómo trabajar este módulo

Este módulo es **incremental**: vamos a construir las suites de interacción de
`AddTodoForm`, `TodoItem` y `LoginForm` **paso a paso**. No copies los archivos
finales de golpe — añade **un bloque por paso**, guarda y ejecuta
`npm run test:watch` para ver al instante cómo cada cambio se refleja. Entre paso y
paso encontrarás retos **🔧 Tu turno** que debes resolver **antes** de mirar la
solución colapsable.

```
Paso 1  →  setup() de user-event y primer click
Paso 2  →  escribir y enviar el formulario
Paso 3  →  por qué SIEMPRE va await
Paso 4  →  tú pruebas el caso "input vacío"
Paso 5  →  TodoItem: toggle y delete
Paso 6  →  tú pruebas LoginForm
```

---

## ¿Por qué probar la interacción?

Hasta ahora hemos renderizado componentes y consultado el DOM. Pero una aplicación
real **reacciona** a lo que hace la persona: escribe en un input, hace click en un
botón, marca un checkbox, presiona Enter. Si nuestros tests no simulan esas
acciones, estamos probando solo la mitad de la historia.

En este módulo aprenderás a disparar interacciones de forma realista y a verificar
que tus componentes responden: que `AddTodoForm` llama a `onAdd` con el texto
correcto, que `TodoItem` dispara `onToggle` y `onDelete`, y que `LoginForm` envía
las credenciales.

```
   Usuario          Test (user-event)        Componente
   --------         -----------------        ----------
   escribe   ─────▶ await user.type(...)  ─▶ onChange / setState
   click     ─────▶ await user.click(...) ─▶ onSubmit / callback
   Enter     ─────▶ await user.keyboard() ─▶ submit del form
```

---

## `fireEvent` vs `user-event`

React Testing Library expone dos formas de disparar eventos: la API de bajo nivel
`fireEvent` y la librería de alto nivel `@testing-library/user-event`.

`fireEvent` dispara **un único evento del DOM** de forma sintética. Por ejemplo,
`fireEvent.change(input, { target: { value: 'hola' } })` cambia el valor de golpe,
sin pasar por las pulsaciones individuales de teclas. Esto no se parece a cómo
escribe una persona real.

`userEvent`, en cambio, **simula la secuencia completa de eventos** que produce el
navegador. Escribir una letra dispara `keydown`, `keypress`, `input`, `keyup`, y
respeta el foco, el estado deshabilitado y demás detalles del navegador.

| Aspecto | `fireEvent` | `userEvent` |
|---|---|---|
| Nivel de abstracción | Bajo (un evento DOM) | Alto (secuencia realista) |
| Realismo | Bajo | Alto |
| `type()` dispara teclas individuales | No | Sí |
| Respeta `disabled`, foco, etc. | No | Sí |
| API | Síncrona | **Asíncrona (`await`)** |
| Recomendado por RTL | Solo casos puntuales | **Sí, por defecto** |
| Necesita `setup()` | No | Sí (v14+) |

> **Regla práctica**: usa `userEvent` para casi todo. Reserva `fireEvent` para
> eventos que `userEvent` no cubre (p. ej. `scroll` o eventos muy específicos).

---

## El componente que vamos a probar: `AddTodoForm`

Antes de testear, necesitamos el componente. Esta es la firma compartida durante
todo el curso: un input de texto y un botón; al enviar, llama a `onAdd(text)` con el
valor escrito y limpia el input.

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
    const value = text.trim()
    if (!value) return // no agregar tareas vacías
    onAdd(value)
    setText('')
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        aria-label="Nueva tarea"
        placeholder="¿Qué hay que hacer?"
        value={text}
        onChange={(e) => setText(e.target.value)}
      />
      <button type="submit">Agregar</button>
    </form>
  )
}
```

---

## Paso 1 — `userEvent.setup()` y el primer click

Desde la versión 14, **siempre** debes llamar a `userEvent.setup()` antes de
interactuar. Esto crea una instancia aislada con su propio estado (portapapeles,
temporizadores) y devuelve un objeto con los métodos de interacción. Además,
usaremos `vi.fn()` para crear un **spy**: una función falsa que registra cómo fue
llamada.

Crea el archivo de test con **solo este primer caso**:

```tsx
// src/components/AddTodoForm.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, it, expect, vi } from 'vitest'
import { AddTodoForm } from './AddTodoForm'

describe('AddTodoForm', () => {
  it('llama a onAdd con el texto al enviar el formulario', async () => {
    // Arrange: setup() ANTES de interactuar + spy que registra llamadas
    const user = userEvent.setup()
    const onAdd = vi.fn()
    render(<AddTodoForm onAdd={onAdd} />)

    // Act: escribimos y hacemos click en el botón
    const input = screen.getByLabelText('Nueva tarea')
    await user.type(input, 'Comprar pan')
    await user.click(screen.getByRole('button', { name: /agregar/i }))

    // Assert: onAdd recibió exactamente el texto escrito, una sola vez
    expect(onAdd).toHaveBeenCalledTimes(1)
    expect(onAdd).toHaveBeenCalledWith('Comprar pan')
  })
})
```

Guarda y mira la terminal: **1 passed**. Desglose:

- `userEvent.setup()` se llama **dentro** del test, antes del `render`.
- `await user.type(input, 'Comprar pan')` escribe carácter a carácter, disparando
  los `onChange` reales del input.
- `vi.fn()` registra cada llamada; `toHaveBeenCalledWith` afirma los argumentos.

> **Importante**: no compartas la misma instancia `user` entre tests. Llama a
> `setup()` en cada test para evitar estado contaminado.

### 🔧 Tu turno #1

Sin mirar abajo: **rompe el test a propósito**. Borra el `await` de la línea
`await user.click(...)` (déjala como `user.click(...)`). Antes de guardar, predice
qué pasará con las aserciones.

<details><summary>✅ Solución / qué observar</summary>

Al quitar el `await`, la promesa del click queda pendiente y las aserciones corren
**antes** de que el evento termine. El test puede fallar de forma **intermitente**
(`expected "spy" to be called once`) o pasar por casualidad. Este es **el error
número uno** al empezar con `user-event`. Vuelve a poner el `await` y el test pasa
de forma estable.

</details>

---

## Paso 2 — Enviar con Enter y limpiar el input

`.type()` interpreta llaves `{}` como **teclas especiales**: `{Enter}`,
`{Backspace}`, `{ArrowLeft}`, etc. Dentro de un input, `{Enter}` dispara el submit
del formulario. Añade estos dos casos dentro del mismo `describe` (no borres el
anterior):

```tsx
// src/components/AddTodoForm.test.tsx  (añade dentro del describe)
  it('también envía al presionar Enter dentro del input', async () => {
    const user = userEvent.setup()
    const onAdd = vi.fn()
    render(<AddTodoForm onAdd={onAdd} />)

    // {Enter} al final del texto dispara el submit del form
    await user.type(screen.getByLabelText('Nueva tarea'), 'Estudiar Vitest{Enter}')

    expect(onAdd).toHaveBeenCalledWith('Estudiar Vitest')
  })

  it('limpia el input después de agregar', async () => {
    const user = userEvent.setup()
    render(<AddTodoForm onAdd={vi.fn()} />)

    const input = screen.getByLabelText<HTMLInputElement>('Nueva tarea')
    await user.type(input, 'Tarea temporal{Enter}')

    // tras enviar, el input vuelve a estar vacío (setText(''))
    expect(input.value).toBe('')
  })
```

Guarda: **3 passed**. Fíjate en `getByLabelText<HTMLInputElement>`: el genérico nos
da el tipo correcto para leer `.value`.

---

## Paso 3 — Por qué `userEvent` es asíncrono

Todos los métodos de `userEvent` devuelven una promesa. Refuerza el concepto del
Paso 1 con esta comparación mental — **no hace falta que la escribas**, solo
entiéndela:

```tsx
// ❌ MAL: sin await, la aserción corre antes de tiempo
it('falla silenciosamente', () => {
  const user = userEvent.setup()
  user.click(boton) // promesa ignorada
  expect(spy).toHaveBeenCalled() // puede fallar de forma intermitente
})

// ✅ BIEN: await en cada interacción
it('espera a que el evento termine', async () => {
  const user = userEvent.setup()
  await user.click(boton)
  expect(spy).toHaveBeenCalled()
})
```

### Los métodos más usados

| Método | Qué hace | Ejemplo |
|---|---|---|
| `.click(el)` | Click izquierdo | `await user.click(boton)` |
| `.dblClick(el)` | Doble click | `await user.dblClick(fila)` |
| `.type(el, txt)` | Escribe carácter a carácter | `await user.type(input, 'hola')` |
| `.clear(el)` | Vacía un input | `await user.clear(input)` |
| `.selectOptions(sel, v)` | Elige opciones de un `<select>` | `await user.selectOptions(sel, 'active')` |
| `.keyboard(seq)` | Pulsa teclas / atajos | `await user.keyboard('{Enter}')` |
| `.tab()` | Mueve el foco con Tab | `await user.tab()` |
| `.hover(el)` | Pasa el ratón por encima | `await user.hover(icono)` |

```tsx
// Escribe, borra el último carácter y corrige
await user.type(input, 'Holaa{Backspace}')
// resultado en el input: "Hola"

// Mantiene Shift mientras pulsa Tab (Shift+Tab)
await user.keyboard('{Shift>}{Tab}{/Shift}')
```

---

## Paso 4 — 🔧 Tu turno #2: el caso "texto vacío"

El componente **no** debe llamar a `onAdd` si el texto está vacío o son solo
espacios (mira el `if (!value) return` del componente). Escribe tú mismo un cuarto
`it` que lo verifique.

**Pistas:**
- Escribe `'   '` (tres espacios) en el input.
- Haz click en el botón "Agregar".
- Afirma con `expect(onAdd).not.toHaveBeenCalled()`.

Inténtalo antes de mirar.

<details><summary>✅ Solución</summary>

```tsx
  it('no llama a onAdd si el texto está vacío o son espacios', async () => {
    const user = userEvent.setup()
    const onAdd = vi.fn()
    render(<AddTodoForm onAdd={onAdd} />)

    await user.type(screen.getByLabelText('Nueva tarea'), '   ')
    await user.click(screen.getByRole('button', { name: /agregar/i }))

    expect(onAdd).not.toHaveBeenCalled()
  })
```

Con esto la suite de `AddTodoForm` llega a **4 passed**. El `.trim()` del componente
convierte `'   '` en `''`, y el `if (!value) return` corta antes de llamar a `onAdd`.

</details>

---

## Paso 5 — `TodoItem`: probar checkbox y botón eliminar

Ahora cambiamos de componente. `TodoItem` dispara `onToggle` al marcar el checkbox y
`onDelete` al pulsar el botón ✕.

```tsx
// src/components/TodoItem.tsx
import type { Todo } from '../types'

interface TodoItemProps {
  todo: Todo
  onToggle: (id: string) => void
  onDelete: (id: string) => void
}

export function TodoItem({ todo, onToggle, onDelete }: TodoItemProps) {
  return (
    <li>
      <input
        type="checkbox"
        aria-label={`Completar ${todo.text}`}
        checked={todo.completed}
        onChange={() => onToggle(todo.id)}
      />
      <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
        {todo.text}
      </span>
      <button aria-label={`Eliminar ${todo.text}`} onClick={() => onDelete(todo.id)}>
        ✕
      </button>
    </li>
  )
}
```

Crea el test con un dato base reutilizable y dos interacciones:

```tsx
// src/components/TodoItem.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, it, expect, vi } from 'vitest'
import type { Todo } from '../types'
import { TodoItem } from './TodoItem'

// Tarea base reutilizable en los tests
const todo: Todo = { id: 'a1', text: 'Leer documentación', completed: false }

describe('TodoItem', () => {
  it('dispara onToggle con el id al hacer click en el checkbox', async () => {
    const user = userEvent.setup()
    const onToggle = vi.fn()
    render(<TodoItem todo={todo} onToggle={onToggle} onDelete={vi.fn()} />)

    await user.click(screen.getByRole('checkbox'))

    expect(onToggle).toHaveBeenCalledTimes(1)
    expect(onToggle).toHaveBeenCalledWith('a1')
  })

  it('dispara onDelete con el id al hacer click en eliminar', async () => {
    const user = userEvent.setup()
    const onDelete = vi.fn()
    render(<TodoItem todo={todo} onToggle={vi.fn()} onDelete={onDelete} />)

    await user.click(screen.getByRole('button', { name: /eliminar/i }))

    expect(onDelete).toHaveBeenCalledWith('a1')
  })

  it('muestra el checkbox marcado cuando la tarea está completada', () => {
    const completada: Todo = { ...todo, completed: true }
    render(<TodoItem todo={completada} onToggle={vi.fn()} onDelete={vi.fn()} />)

    // este caso NO interactúa: por eso no necesita async/await
    expect(screen.getByRole('checkbox')).toBeChecked()
  })
})
```

> Nota cómo el tercer caso **no** usa `user` ni `await`: solo verifica el render
> inicial. Solo necesitas `async`/`await` cuando hay una interacción.

---

## Paso 6 — 🔧 Tu turno #3: probar `LoginForm`

`LoginForm` recibe `onLogin(username, password)` y tiene campos de usuario y
contraseña. Este es el componente:

```tsx
// src/components/LoginForm.tsx
import { useState } from 'react'

interface LoginFormProps {
  onLogin: (username: string, password: string) => void
}

export function LoginForm({ onLogin }: LoginFormProps) {
  const [username, setUsername] = useState('')
  const [password, setPassword] = useState('')

  function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    onLogin(username, password)
  }

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Usuario
        <input value={username} onChange={(e) => setUsername(e.target.value)} />
      </label>
      <label>
        Contraseña
        <input
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
        />
      </label>
      <button type="submit">Entrar</button>
    </form>
  )
}
```

Escribe tú mismo un test que rellene **ambos** campos con `'ada'` y `'s3cret'`, pulse
"Entrar" y verifique que `onLogin` recibió ambos argumentos.

**Pistas:**
- Localiza los inputs con `getByLabelText('Usuario')` y `getByLabelText('Contraseña')`.
- El botón se filtra con `getByRole('button', { name: /entrar/i })`.
- Afirma con `toHaveBeenCalledWith('ada', 's3cret')`.

<details><summary>✅ Solución</summary>

```tsx
// src/components/LoginForm.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, it, expect, vi } from 'vitest'
import { LoginForm } from './LoginForm'

describe('LoginForm', () => {
  it('envía usuario y contraseña al hacer login', async () => {
    const user = userEvent.setup()
    const onLogin = vi.fn()
    render(<LoginForm onLogin={onLogin} />)

    await user.type(screen.getByLabelText('Usuario'), 'ada')
    await user.type(screen.getByLabelText('Contraseña'), 's3cret')
    await user.click(screen.getByRole('button', { name: /entrar/i }))

    expect(onLogin).toHaveBeenCalledWith('ada', 's3cret')
  })

  it('permite corregir el campo usuario con clear()', async () => {
    const user = userEvent.setup()
    const onLogin = vi.fn()
    render(<LoginForm onLogin={onLogin} />)

    const usuario = screen.getByLabelText('Usuario')
    await user.type(usuario, 'aada')
    await user.clear(usuario) // vacía el input por completo
    await user.type(usuario, 'ada')
    await user.click(screen.getByRole('button', { name: /entrar/i }))

    expect(onLogin).toHaveBeenCalledWith('ada', '')
  })
})
```

El segundo caso muestra `.clear()`: vacía el input por completo antes de reescribir.
Como nunca tocamos la contraseña, llega vacía (`''`).

</details>

---

## Bonus: `.selectOptions()` con un `<select>`

Aunque nuestro `TodoFilter` puede ser de botones, si lo implementaras como
`<select>`, así se prueba:

```tsx
// src/components/SelectFilter.test.tsx (ejemplo ilustrativo)
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { it, expect, vi } from 'vitest'

it('selecciona la opción "active"', async () => {
  const user = userEvent.setup()
  const onChange = vi.fn()
  render(
    <select aria-label="Filtro" onChange={(e) => onChange(e.target.value)}>
      <option value="all">Todas</option>
      <option value="active">Activas</option>
      <option value="completed">Completadas</option>
    </select>,
  )

  // selecciona por value (también acepta el texto visible)
  await user.selectOptions(screen.getByLabelText('Filtro'), 'active')

  expect(onChange).toHaveBeenCalled()
})
```

---

### Prueba esto

- Cambia `await user.type(input, 'Comprar pan')` por `fireEvent.change(...)` y
  observa qué eventos deja de disparar y por qué `.type()` es más realista.
- Quita un `await` de una interacción y ejecuta el test varias veces: nota cómo se
  vuelve intermitente (a veces pasa, a veces no).
- Añade un test que escriba texto, lo borre con `{Backspace}` repetido y verifique
  el valor final del input.
- Usa `user.tab()` para mover el foco entre los campos de `LoginForm` y comprueba
  `toHaveFocus()` en cada paso.
- Prueba que el botón "Agregar" no rompe nada si se hace doble click rápido con
  `user.dblClick()`: ¿cuántas veces se llama `onAdd`?
- Crea un helper `renderWithUser(ui)` que devuelva `{ user: userEvent.setup(), ...render(ui) }`
  y reescribe un test para usarlo.

---

## Ejercicios propuestos

### Básico

**B1 — Botón deshabilitado.** Modifica mentalmente `AddTodoForm` para que el botón
tenga `disabled={!text.trim()}` y escribe un `it` que verifique que, con el input
vacío, el botón está deshabilitado usando `toBeDisabled()`.

**B2 — Placeholder.** Escribe un `it` que localice el input por su placeholder
(`getByPlaceholderText('¿Qué hay que hacer?')`) y verifique que está en el documento.

### Intermedio

**I1 — Doble envío.** Escribe un test que agregue dos tareas seguidas con el mismo
formulario (escribir + Enter, escribir + Enter) y verifique que `onAdd` fue llamado
**dos veces** con los textos correctos, usando `toHaveBeenNthCalledWith`.

**I2 — Foco con Tab.** En `LoginForm`, usa `user.tab()` partiendo desde el body y
verifica con `toHaveFocus()` que el foco viaja en orden: Usuario → Contraseña →
botón "Entrar".

### Avanzado

**A1 — Helper compartido.** Crea `src/test/setup-user.tsx` con `renderWithUser(ui)`
que devuelva `{ user, ...render(ui) }`. Reescribe las tres suites (`AddTodoForm`,
`TodoItem`, `LoginForm`) para usarlo y elimina las llamadas repetidas a `setup()`.

**A2 — Borrado interactivo.** Escribe un test que escriba `'Comprar pannn'`, luego
use `user.keyboard('{Backspace}{Backspace}')` para corregir a `'Comprar pan'`, envíe
con `{Enter}` y verifique que `onAdd` recibió exactamente `'Comprar pan'`.

---

## Resumen del módulo 5

- `userEvent` simula interacciones realistas (secuencias completas de eventos);
  `fireEvent` dispara un único evento de bajo nivel.
- Desde la v14 hay que llamar a `userEvent.setup()` en cada test antes de
  interactuar; no compartas la instancia entre tests.
- **Todos los métodos de `userEvent` son asíncronos**: usa siempre `await`. Olvidarlo
  vuelve los tests intermitentes.
- Métodos clave: `.click()`, `.type()`, `.clear()`, `.selectOptions()`,
  `.keyboard()`; `.type()` interpreta teclas especiales como `{Enter}`.
- `vi.fn()` crea spies para verificar callbacks: `toHaveBeenCalledTimes`,
  `toHaveBeenCalledWith`, `not.toHaveBeenCalled`.
- Construimos las suites **por pasos**: del primer click, a escribir y enviar, hasta
  que tú mismo probaste el caso vacío y el `LoginForm` completo.

> **Siguiente página →** Módulo 6: props requeridas vs opcionales, renderizado condicional, listas con `getAllByRole` y re-render con `rerender`.
