# Testing en React con Vitest — Página 8
## Módulo 8 · Mocking
### Aislar dependencias con `vi.fn()`, `vi.spyOn()` y `vi.mock()`

---

## Cómo trabajar este módulo

Este módulo es **incremental**: vamos a construir nuestros mocks **paso a paso**,
desde la función falsa más simple (`vi.fn()`) hasta reemplazar el módulo de API
completo (`vi.mock()`). No copies los bloques grandes de golpe — añade **un paso
cada vez**, guarda y deja corriendo `npm run test:watch` para ver cómo cada
cambio se refleja al instante. Entre paso y paso encontrarás retos
**🔧 Tu turno** que debes resolver **antes** de mirar la solución.

```
Paso 1  →  vi.fn(): la función espía mínima
Paso 2  →  controlamos su valor de retorno
Paso 3  →  lo aplicamos a un callback de props
Paso 4  →  tú escribes un mock async (Tu turno)
Paso 5  →  vi.spyOn(): espiar lo que ya existe
Paso 6  →  vi.mock(): reemplazar el módulo de API
Paso 7  →  limpieza de mocks entre tests (Tu turno)
```

> **Nada de Jest**: en Vitest todo se hace con `vi`, no con `jest`.

---

## ¿Qué es un mock y por qué importa?

Un **mock** es una sustitución controlada de una dependencia real. Cuando pruebas
una unidad de código (un componente, un hook), quieres que el test dependa
**solo** de esa unidad, no de la red, del reloj, ni de módulos pesados. Los mocks
te permiten:

- **Aislar**: que el test no toque APIs reales ni efectos secundarios.
- **Controlar**: forzar respuestas (éxito, error) sin depender del azar.
- **Observar**: registrar cómo y cuántas veces se llamó algo.

Vitest ofrece tres herramientas principales, todas bajo el objeto `vi`:

```
   vi.fn()      → crea una función falsa desde cero (spy/stub)
   vi.spyOn()   → envuelve un método EXISTENTE para espiarlo
   vi.mock()    → reemplaza un MÓDULO entero
```

Las construiremos en ese orden, de lo más pequeño a lo más global.

---

## Paso 1 — `vi.fn()`: la función espía mínima

`vi.fn()` crea una función vacía que **registra cada llamada**: argumentos, número
de veces, valores devueltos. Es la base de todo lo demás. Crea este archivo con
**solo este caso**:

```ts
// src/examples/vi-fn.test.ts
import { describe, it, expect, vi } from 'vitest'

describe('vi.fn()', () => {
  it('registra las llamadas y sus argumentos', () => {
    // Arrange: creamos una función mock vacía.
    const mock = vi.fn()

    // Act: la llamamos dos veces con argumentos distintos.
    mock('a', 1)
    mock('b', 2)

    // Assert: el mock recuerda cuántas veces y con qué se le llamó.
    expect(mock).toHaveBeenCalledTimes(2)
    expect(mock).toHaveBeenCalledWith('a', 1)
    // último set de argumentos
    expect(mock).toHaveBeenLastCalledWith('b', 2)
  })
})
```

Guarda y mira la terminal: **1 passed**. Lo importante aquí es que `mock` no hace
nada útil, pero **lo recuerda todo**. Esa memoria es la que verificamos con los
*matchers* `toHaveBeenCalled*`.

### Aserciones sobre mocks

Estas son las aserciones que usaremos una y otra vez:

| Aserción | Verifica |
|---|---|
| `toHaveBeenCalled()` | Se llamó al menos una vez |
| `not.toHaveBeenCalled()` | Nunca se llamó |
| `toHaveBeenCalledTimes(n)` | Se llamó exactamente `n` veces |
| `toHaveBeenCalledWith(...args)` | Se llamó con esos argumentos (alguna vez) |
| `toHaveBeenLastCalledWith(...args)` | La última llamada usó esos argumentos |
| `toHaveBeenNthCalledWith(i, ...args)` | La llamada `i`-ésima usó esos argumentos |

También puedes inspeccionar el registro a mano:

```ts
const mock = vi.fn()
mock('x')
// mock.mock.calls es un array de arrays: un sub-array por llamada.
expect(mock.mock.calls).toEqual([['x']])
```

### 🔧 Tu turno #1

Sin mirar abajo: añade un `it` que llame al mock **tres** veces (`mock(1)`,
`mock(2)`, `mock(3)`) y verifique con `toHaveBeenNthCalledWith` que la **segunda**
llamada recibió `2`. Predice qué número va primero en `toHaveBeenNthCalledWith`.

<details><summary>✅ Solución</summary>

```ts
  it('recuerda cada llamada por su posición', () => {
    const mock = vi.fn()
    mock(1)
    mock(2)
    mock(3)
    // El índice de toHaveBeenNthCalledWith empieza en 1, no en 0.
    expect(mock).toHaveBeenNthCalledWith(2, 2)
  })
```

El primer argumento (`2`) es la **posición** de la llamada (empieza en 1); el
resto son los argumentos esperados. Es un error común pensar que arranca en 0.

</details>

---

## Paso 2 — Controlar el valor de retorno

Un mock vacío devuelve `undefined`. Pero a menudo necesitamos que **devuelva
algo** para que el código bajo prueba siga su curso. Estos son los métodos clave:

| Método | Para qué sirve |
|---|---|
| `.mockReturnValue(v)` | Devuelve `v` de forma síncrona |
| `.mockReturnValueOnce(v)` | Solo en la próxima llamada |
| `.mockResolvedValue(v)` | Devuelve una promesa resuelta con `v` |
| `.mockRejectedValue(e)` | Devuelve una promesa rechazada con `e` |
| `.mockImplementation(fn)` | Reemplaza el cuerpo completo |

Añade estos casos en un archivo nuevo, uno a uno:

```ts
// src/examples/vi-fn-return.test.ts
import { describe, it, expect, vi } from 'vitest'

describe('valores de retorno de vi.fn()', () => {
  it('mockReturnValue devuelve un valor fijo', () => {
    const getNumero = vi.fn().mockReturnValue(42)
    expect(getNumero()).toBe(42)
  })

  it('mockResolvedValue devuelve una promesa resuelta', async () => {
    // Útil para simular llamadas a la API que resuelven con datos.
    const cargar = vi.fn().mockResolvedValue(['a', 'b'])
    await expect(cargar()).resolves.toEqual(['a', 'b'])
  })

  it('mockRejectedValue devuelve una promesa rechazada', async () => {
    // Útil para simular un error de red.
    const fallar = vi.fn().mockRejectedValue(new Error('boom'))
    await expect(fallar()).rejects.toThrow('boom')
  })

  it('mockImplementation reemplaza la lógica', () => {
    const doble = vi.fn().mockImplementation((n: number) => n * 2)
    expect(doble(5)).toBe(10)
  })
})
```

Fíjate en `mockResolvedValue` y `mockRejectedValue`: son atajos para promesas.
Serán los protagonistas cuando mockeemos la capa de API en el Paso 6.

---

## Paso 3 — Mockear un callback de props

Ya tenemos lo básico. Apliquémoslo al caso **más común en componentes**: pasar un
`vi.fn()` como prop para verificar que el componente lo invoca correctamente.
Recordemos el formulario que ya conoces de módulos anteriores:

```tsx
// src/components/AddTodoForm.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { describe, it, expect, vi } from 'vitest'
import { AddTodoForm } from './AddTodoForm'

describe('AddTodoForm con mock de callback', () => {
  it('invoca onAdd con el texto', async () => {
    // Arrange: un usuario y un callback mockeado como prop.
    const user = userEvent.setup()
    const onAdd = vi.fn()

    render(<AddTodoForm onAdd={onAdd} />)

    // Act: escribimos y pulsamos Enter.
    await user.type(screen.getByLabelText('Nueva tarea'), 'Pan{Enter}')

    // Assert: el componente llamó al callback con el texto, una sola vez.
    expect(onAdd).toHaveBeenCalledExactlyOnceWith('Pan')
  })
})
```

Aquí el mock no devuelve nada: lo que nos importa es **observar** que el
componente lo llamó con el argumento correcto. Esta es la razón #3 (observar) en
acción.

---

## Paso 4 — 🔧 Tu turno #2: un mock asíncrono

El componente `TodoList` recibe una prop `onLoad` que el componente invoca para
recargar tareas. Escribe tú mismo un `it` que:

1. Cree un `onLoad` con `vi.fn()` que **resuelva** con una lista de un todo.
2. Lo llame directamente (`await onLoad()`).
3. Verifique que la promesa resuelve con esa lista.

**Pistas:** usa `mockResolvedValue` y `await expect(...).resolves.toEqual(...)`.

<details><summary>✅ Solución</summary>

```ts
  it('onLoad resuelve con la lista de tareas', async () => {
    const tareas = [{ id: '1', text: 'Pan', completed: false }]
    // mockResolvedValue convierte el mock en una "promesa que resuelve".
    const onLoad = vi.fn().mockResolvedValue(tareas)

    await expect(onLoad()).resolves.toEqual(tareas)
    expect(onLoad).toHaveBeenCalledTimes(1)
  })
```

La clave es que `mockResolvedValue(x)` equivale a
`mockImplementation(() => Promise.resolve(x))`, pero más legible. Para el camino
de error usarías `mockRejectedValue(new Error(...))`.

</details>

---

## Paso 5 — `vi.spyOn()`: espiar lo que ya existe

A diferencia de `vi.fn()` (que crea algo nuevo), `vi.spyOn(objeto, 'metodo')`
**envuelve un método que ya existe**, conservando su comportamiento por defecto
pero permitiendo observarlo o reemplazarlo temporalmente.

```ts
// src/examples/vi-spyon.test.ts
import { describe, it, expect, vi, afterEach } from 'vitest'

describe('vi.spyOn()', () => {
  afterEach(() => {
    // restaura los métodos originales después de cada test
    vi.restoreAllMocks()
  })

  it('espía console.log sin perder su comportamiento', () => {
    const spy = vi.spyOn(console, 'log')

    console.log('hola')

    expect(spy).toHaveBeenCalledWith('hola')
  })

  it('puede reemplazar la implementación temporalmente', () => {
    // evita que Math.random sea impredecible en el test
    const spy = vi.spyOn(Math, 'random').mockReturnValue(0.5)

    expect(Math.random()).toBe(0.5)
    expect(spy).toHaveBeenCalled()
  })
})
```

> **`restoreAllMocks` solo afecta a los `spyOn`**: devuelve los métodos a su
> versión original. Por eso es importante con `spyOn`, donde tocas objetos reales:
> si no restauras, contaminas otros tests que usen `console` o `Math`.

```
   vi.fn()                         vi.spyOn(obj, 'm')
   --------                        ------------------
   () => undefined                 obj.m  ──wrap──▶ [spy] ──▶ obj.m original
   (lo defines tú)                 (observa y/o reemplaza, luego restaura)
```

> **En resumen**: usa un **mock** (`vi.fn`) cuando no existe nada que envolver
> (callbacks, dependencias inyectadas). Usa un **spy** (`vi.spyOn`) cuando quieres
> observar (o cambiar puntualmente) algo que **ya existe** y debe volver a su
> estado original.

---

## Paso 6 — `vi.mock()`: reemplazar el módulo de API

El salto grande: cuando una unidad importa otro módulo (por ejemplo, la capa de
API), `vi.mock()` sustituye **todo el módulo** por una versión falsa. Es ideal
para evitar llamadas de red reales.

Primero, recordemos el módulo a mockear:

```ts
// src/api/todos.ts
import type { Todo } from '../types'

export async function fetchTodos(): Promise<Todo[]> {
  const res = await fetch('/api/todos')
  if (!res.ok) throw new Error('Error al cargar tareas')
  return res.json()
}
```

Ahora el test que lo reemplaza. Empezamos con el camino de **éxito**:

```tsx
// src/components/TodoApp.test.tsx
import { render, screen } from '@testing-library/react'
import { describe, it, expect, vi, beforeEach } from 'vitest'
import type { Todo } from '../types'
import { fetchTodos } from '../api/todos'
import { TodoApp } from './TodoApp'

// vi.mock se "iza" (hoisting) al inicio del archivo: reemplaza el módulo entero.
vi.mock('../api/todos', () => ({
  fetchTodos: vi.fn(), // creamos un mock vacío que controlaremos por test
}))

// le decimos a TS que fetchTodos es ahora un mock
const fetchTodosMock = vi.mocked(fetchTodos)

describe('TodoApp con API mockeada', () => {
  beforeEach(() => {
    vi.clearAllMocks() // limpia el historial entre tests
  })

  it('muestra las tareas que devuelve la API', async () => {
    // Arrange: respuesta controlada para este test.
    const datos: Todo[] = [{ id: '1', text: 'Tarea remota', completed: false }]
    fetchTodosMock.mockResolvedValue(datos)

    render(<TodoApp />)

    // esperamos a que aparezca el dato cargado de forma asíncrona
    expect(await screen.findByText('Tarea remota')).toBeInTheDocument()
    expect(fetchTodosMock).toHaveBeenCalledTimes(1)
  })
})
```

Guarda: la `TodoApp` cree que llamó al backend, pero recibió nuestros datos. Sin
red, sin azar, sin esperas reales.

### Detalles importantes de `vi.mock()`

- **Hoisting**: Vitest mueve `vi.mock()` al tope del archivo, antes de los imports.
  Por eso **no** puedes usar variables externas dentro de la factory (salvo con
  `vi.hoisted()`).
- **`vi.mocked()`**: ayuda de tipos que le dice a TypeScript que el import es un
  mock, habilitando `.mockResolvedValue`, etc.
- Mockea por **ruta de import**, igual que la usa el código bajo prueba.

```ts
// Si necesitas una variable en la factory, usa vi.hoisted()
const { datosFalsos } = vi.hoisted(() => ({ datosFalsos: [] as Todo[] }))
vi.mock('../api/todos', () => ({
  fetchTodos: vi.fn().mockResolvedValue(datosFalsos),
}))
```

### Añade el camino de error

Ahora que el éxito funciona, añade el caso de fallo en el **mismo** `describe`:

```tsx
// src/components/TodoApp.test.tsx  (añade este it dentro del describe)
  it('muestra un error si la API falla', async () => {
    // mockRejectedValue simula un fallo de red/servidor.
    fetchTodosMock.mockRejectedValue(new Error('caída'))

    render(<TodoApp />)

    expect(await screen.findByRole('alert')).toBeInTheDocument()
  })
```

Nota cómo `mockResolvedValue` y `mockRejectedValue` (del Paso 2) nos dan los dos
caminos sin tocar nada de red.

---

## Paso 7 — 🔧 Tu turno #3: limpiar mocks entre tests

Sin limpieza, el historial de llamadas se **acumula** entre tests y provoca
falsos positivos/negativos. Tu reto: **rompe** el aislamiento a propósito.

Quita el `vi.clearAllMocks()` del `beforeEach` del Paso 6 y observa qué pasa con
el `expect(fetchTodosMock).toHaveBeenCalledTimes(1)`. Luego decide cuál de los
tres métodos de limpieza necesitas para arreglarlo.

| Método | Limpia historial | Resetea implementación | Restaura original |
|---|---|---|---|
| `vi.clearAllMocks()` | Sí | No | No |
| `vi.resetAllMocks()` | Sí | Sí (a `vi.fn()` vacío) | No |
| `vi.restoreAllMocks()` | Sí | — | Sí (solo `spyOn`) |

<details><summary>✅ Solución</summary>

Al quitar la limpieza, el segundo test que llama a `fetchTodos` ve el historial
del primero. Un `toHaveBeenCalledTimes(1)` empieza a fallar porque el contador
acumula `2`, `3`… La solución correcta es **`clearAllMocks`** en `beforeEach`:
borra el historial pero conserva las implementaciones que cada test configura.

```ts
beforeEach(() => {
  // historial limpio en cada test; implementaciones intactas
  vi.clearAllMocks()
})
```

Alternativamente, actívalo de forma global en la config:

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'jsdom',
    clearMocks: true, // equivale a vi.clearAllMocks() antes de cada test
  },
})
```

> **Regla práctica**: `clearAllMocks` en `beforeEach` para el historial;
> `resetAllMocks` si además quieres olvidar implementaciones; `restoreAllMocks`
> (en `afterEach`) cuando usas `spyOn` sobre objetos reales.

</details>

---

## Mock vs Spy: ¿cuándo cada uno?

Con todo construido, esta es la tabla de decisión que te llevarás del módulo:

| Aspecto | Mock (`vi.fn`) | Spy (`vi.spyOn`) |
|---|---|---|
| Qué crea | Una función nueva desde cero | Un envoltorio sobre un método existente |
| Comportamiento por defecto | Ninguno (devuelve `undefined`) | El original (a menos que lo reemplaces) |
| Caso típico | Callbacks de props, dependencias inyectadas | Espiar `console`, `Math`, métodos de un objeto real |
| Restauración | No aplica (es nuevo) | `vi.restoreAllMocks()` devuelve el original |
| Riesgo | Bajo | Si no restauras, contaminas otros tests |

---

### Prueba esto

- Crea un `vi.fn()` con `.mockReturnValueOnce(1).mockReturnValue(0)` y comprueba que
  la primera llamada devuelve 1 y las siguientes 0.
- Usa `vi.spyOn(console, 'error')` para verificar que un componente registra un
  error, y restáuralo con `vi.restoreAllMocks()`.
- Mockea `../api/todos` y prueba tanto el camino de éxito (`mockResolvedValue`)
  como el de error (`mockRejectedValue`).
- Quita el `vi.clearAllMocks()` del `beforeEach` y observa cómo un
  `toHaveBeenCalledTimes(1)` empieza a fallar al acumularse llamadas.
- Compara `vi.mock` con `vi.hoisted`: intenta usar una variable externa en la
  factory sin `vi.hoisted` y lee el error.
- Espía `Date.now` con `vi.spyOn` y `mockReturnValue` para tener un reloj
  determinista en un test.

---

## Ejercicios propuestos

### Básico

**B1 — Contador de clics.** Crea un `vi.fn()` llamado `onClick`, llámalo cuatro
veces y escribe un `it` que verifique con `toHaveBeenCalledTimes` que se llamó
exactamente 4 veces.

**B2 — Retorno fijo.** Crea un mock `getSaludo` con `mockReturnValue('Hola')` y
verifica que al invocarlo dos veces devuelve `'Hola'` ambas veces.

### Intermedio

**I1 — `onToggle` de TodoItem.** Renderiza `TodoItem` con un `onToggle` mockeado,
haz clic en el checkbox con `user-event` y verifica que `onToggle` fue llamado
con el `id` de la tarea.

**I2 — Reloj determinista.** Usa `vi.spyOn(Date, 'now').mockReturnValue(0)` para
fijar el tiempo y verifica que una función que genera ids basados en `Date.now()`
produce un id predecible. Restaura con `vi.restoreAllMocks()` en `afterEach`.

### Avanzado

**A1 — Mock parcial con `importActual`.** Mockea `../api/todos` pero conserva
todas sus exportaciones reales salvo `fetchTodos`, usando
`vi.importActual` dentro de una factory `async`. Verifica que las demás
exportaciones siguen funcionando.

**A2 — Secuencia de respuestas.** Usa `mockResolvedValueOnce` encadenado para que
`fetchTodos` devuelva una lista en la primera llamada y otra distinta en la
segunda. Renderiza un componente que recargue datos y verifica que se ven ambos
resultados en orden.

---

## Resumen del módulo 8

- `vi.fn()` crea funciones mock que registran llamadas; controla su salida con
  `mockReturnValue`, `mockResolvedValue`, `mockRejectedValue`, `mockImplementation`.
- Aserciones clave: `toHaveBeenCalled`, `toHaveBeenCalledWith`,
  `toHaveBeenCalledTimes` (y variantes `Last`/`Nth`).
- Construimos los mocks **por pasos**: de la función espía mínima, al valor de
  retorno, al callback de props, al mock async, al spy y al módulo completo.
- `vi.spyOn(obj, 'metodo')` envuelve métodos existentes para observarlos o
  reemplazarlos temporalmente; restaura con `vi.restoreAllMocks()`.
- `vi.mock('ruta')` reemplaza módulos completos (ideal para la capa de API); se
  iza al inicio del archivo y se tipa con `vi.mocked()`.
- Limpia el estado en `beforeEach`: `clearAllMocks` (historial), `resetAllMocks`
  (historial + implementación), `restoreAllMocks` (originales de `spyOn`).
- Mock = función nueva (callbacks/módulos); Spy = envoltorio de algo existente que
  hay que restaurar.

> **Siguiente página →** Módulo 9: testing asíncrono y peticiones `fetch` interceptadas con MSW (Mock Service Worker).
