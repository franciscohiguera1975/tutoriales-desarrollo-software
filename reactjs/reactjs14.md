# Curso React.js + TypeScript — Página 13
## Módulo 6 · Ecosistema
### Testing con Vitest y React Testing Library

---

## ¿Por qué testear componentes React?

Los tests verifican que tus componentes se comportan como el usuario
espera — no cómo están implementados internamente. React Testing Library
(RTL) impone esta filosofía: **testea lo que el usuario ve y hace**,
no los detalles del estado o las props.

```
Tests de implementación (evitar)     Tests de comportamiento (preferir)
─────────────────────────────        ─────────────────────────────────
¿El estado es { count: 1 }?          ¿La pantalla muestra "Cuenta: 1"?
¿Se llamó a setState?                 ¿El botón existe y es clickeable?
¿La función interna se ejecutó?       ¿Se mostró el mensaje de error?
```

---

## Instalación y configuración

```bash
npm install -D vitest @vitest/ui jsdom \
  @testing-library/react \
  @testing-library/jest-dom \
  @testing-library/user-event
```

### Versiones actuales

| Librería | Versión |
|---|---|
| `vitest` | ^4.1.0 |
| `@testing-library/react` | ^16.3.2 |
| `@testing-library/jest-dom` | ^6.9.1 |
| `@testing-library/user-event` | ^14.6.1 |
| `jsdom` | ^29.0.0 |

---

### `vite.config.ts` — añadir configuración de test

```ts
// vite.config.ts

import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',      // simula el DOM del navegador
    globals:     true,         // describe, it, expect disponibles sin import
    setupFiles:  ['./src/test/setup.ts'],
  },
})
```

### `src/test/setup.ts`

```ts
// src/test/setup.ts

import '@testing-library/jest-dom'
// Extiende los matchers de Vitest con los de jest-dom:
// toBeInTheDocument, toHaveTextContent, toBeVisible, etc.
```

### `tsconfig.app.json` — añadir tipos de Vitest

```json
{
  "compilerOptions": {
    "types": ["vite/client", "vitest/globals", "@testing-library/jest-dom"]
  }
}
```

### Scripts en `package.json`

```json
{
  "scripts": {
    "test":       "vitest",
    "test:run":   "vitest run",
    "test:ui":    "vitest --ui",
    "test:cover": "vitest run --coverage"
  }
}
```

### Prueba esto

- Ejecuta `npm test` en la terminal — Vitest arranca en modo watch y muestra los resultados en tiempo real; cualquier cambio en los archivos relanza los tests automáticamente
- Ejecuta `npm run test:ui` — se abre una interfaz visual en el navegador donde puedes ver el árbol de tests, filtrar por nombre y ver los errores con detalle
- Cambia `globals: true` a `globals: false` en `vite.config.ts` — los tests fallan porque `describe`, `it` y `expect` ya no están disponibles sin importarlos explícitamente
- Elimina temporalmente la línea `import '@testing-library/jest-dom'` de `setup.ts` — los matchers como `toBeInTheDocument` dejan de funcionar y TypeScript muestra errores de tipo
- Ejecuta `npm run test:cover` — se genera un informe de cobertura que muestra qué líneas de código están cubiertas por tests y cuáles no
- Añade `coverage: { reporter: ['text', 'html'] }` dentro de `test:` en `vite.config.ts` y vuelve a ejecutar `test:cover` — se genera una carpeta `coverage/` con un informe HTML navegable

---

## Estructura de archivos

```
src/
├── components/
│   ├── StatusBadge.tsx
│   ├── Counter.tsx
│   ├── SearchInput.tsx
│   └── UserCard.tsx
├── __tests__/                ← tests junto a src/
│   ├── StatusBadge.test.tsx
│   ├── Counter.test.tsx
│   ├── SearchInput.test.tsx
│   └── UserCard.test.tsx
└── test/
    └── setup.ts
```

> Convención alternativa: colocar el test junto al componente
> (`StatusBadge.test.tsx` en la misma carpeta que `StatusBadge.tsx`).
> Ambas son válidas — lo importante es ser consistente en todo el proyecto.

---

## Componentes a testear

Los componentes están diseñados con atributos de accesibilidad (`aria-label`,
`role`, `data-testid`) que hacen los tests más robustos y descriptivos.

### `src/components/StatusBadge.tsx`

```tsx
// src/components/StatusBadge.tsx

type BadgeStatus = 'active' | 'inactive' | 'pending'

interface StatusBadgeProps {
  status:  BadgeStatus
  label?:  string
}

const config: Record<BadgeStatus, { bg: string; color: string; text: string }> = {
  active:   { bg: '#dcfce7', color: '#166534', text: 'Activo'    },
  inactive: { bg: '#f3f4f6', color: '#6b7280', text: 'Inactivo'  },
  pending:  { bg: '#fef9c3', color: '#854d0e', text: 'Pendiente' },
}

export default function StatusBadge({ status, label }: StatusBadgeProps) {
  const { bg, color, text } = config[status]
  return (
    <span
      data-testid="status-badge"
      style={{ backgroundColor: bg, color, padding: '2px 8px', borderRadius: 10, fontSize: 12 }}
    >
      {label ?? text}
    </span>
  )
}
```

### Prueba esto

- Renderiza `<StatusBadge status="active" />` en `App.tsx` y observa el badge verde con el texto "Activo" — confirma que el componente funciona visualmente antes de escribir tests
- Añade un cuarto valor `'error'` al tipo `BadgeStatus` sin actualizar el objeto `config` — TypeScript marca un error en tiempo de compilación porque `config[status]` puede ser `undefined`
- Agrega `{ error: { bg: '#fef2f2', color: '#dc2626', text: 'Error' } }` a `config` — el nuevo estado queda disponible para los tests sin cambiar nada más en el componente
- Cambia `data-testid="status-badge"` a `data-testid="badge"` — todos los tests que usen `getByTestId('status-badge')` fallarán; actualiza el testid en los tests para que vuelvan a pasar
- Pasa `label="Conectado"` al componente — el texto "Conectado" reemplaza al texto por defecto "Activo" gracias al operador `??`
- Elimina el atributo `data-testid` completamente del `span` — los tests que usen `getByTestId` fallan; como ejercicio, reescribe esos tests usando `getByText` en su lugar

### `src/components/Counter.tsx`

```tsx
// src/components/Counter.tsx

import { useState } from 'react'

interface CounterProps {
  initialValue?:  number
  step?:          number
  onCountChange?: (count: number) => void
}

export default function Counter({
  initialValue = 0,
  step = 1,
  onCountChange,
}: CounterProps) {
  const [count, setCount] = useState(initialValue)

  function increment() {
    const next = count + step
    setCount(next)
    onCountChange?.(next)
  }

  function decrement() {
    const next = count - step
    setCount(next)
    onCountChange?.(next)
  }

  function reset() {
    setCount(initialValue)
    onCountChange?.(initialValue)
  }

  return (
    <div>
      <p data-testid="count-display">Cuenta: {count}</p>
      <button onClick={decrement}>Decrementar</button>
      <button onClick={increment}>Incrementar</button>
      <button onClick={reset}>Reset</button>
    </div>
  )
}
```

### Prueba esto

- Renderiza `<Counter initialValue={5} step={3} />` en `App.tsx` y haz clic en "Incrementar" — el contador pasa de 5 a 8, confirmando que `step` se aplica correctamente
- Añade una prop `max?: number` al componente y deshabilita el botón "Incrementar" cuando `count >= max` — los tests existentes siguen pasando; añade un nuevo test que verifique que el botón está deshabilitado al llegar al máximo
- Cambia el texto del botón de "Incrementar" a "Sumar" en el componente — el test que usa `getByRole('button', { name: 'Incrementar' })` falla inmediatamente, lo que demuestra por qué RTL busca por contenido visible
- Cambia `data-testid="count-display"` a `role="status"` en el `<p>` — actualiza los tests para usar `getByRole('status')` en lugar de `getByTestId('count-display')`
- Pasa `onCountChange={vi.fn()}` al componente y haz clic varias veces en los botones — abre la consola y observa que la función mock se llama con cada nuevo valor
- Añade un límite mínimo: impide que el contador baje de `0` modificando `decrement` — escribe un test que verifique que el contador no cambia al hacer clic en "Decrementar" cuando ya está en 0

### `src/components/SearchInput.tsx`

```tsx
// src/components/SearchInput.tsx

import { useState } from 'react'

interface SearchInputProps {
  onSearch:     (query: string) => void
  placeholder?: string
  minLength?:   number
}

export default function SearchInput({
  onSearch,
  placeholder = 'Buscar...',
  minLength   = 2,
}: SearchInputProps) {
  const [query, setQuery] = useState('')
  const [error, setError] = useState<string | null>(null)

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    if (query.trim().length < minLength) {
      setError(`Mínimo ${minLength} caracteres`)
      return
    }
    setError(null)
    onSearch(query.trim())
  }

  return (
    <form onSubmit={handleSubmit} role="search">
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder={placeholder}
        aria-label="Campo de búsqueda"
      />
      <button type="submit">Buscar</button>
      {error && <p role="alert">{error}</p>}
    </form>
  )
}
```

### Prueba esto

- Renderiza `<SearchInput onSearch={(q) => console.log(q)} minLength={3} />` y escribe "ab" antes de hacer clic en "Buscar" — aparece el mensaje "Mínimo 3 caracteres" en el DOM
- Escribe una búsqueda válida y haz clic en "Buscar" — el mensaje de error desaparece porque `setError(null)` se ejecuta antes de llamar a `onSearch`
- Elimina el atributo `role="search"` del `<form>` — el test que usa `getByRole('textbox', { name: 'Campo de búsqueda' })` sigue pasando, pero ya no se puede encontrar el formulario por su rol semántico
- Cambia `role="alert"` del `<p>` de error a `className="error"` — el test que usa `getByRole('alert')` falla; demuestra que los atributos de accesibilidad también sirven como selectores de test robustos
- Añade `aria-label` dinámico al botón de búsqueda: `aria-label={query ? 'Buscar "${query}"' : 'Buscar'}` — los tests que buscan el botón por nombre deben actualizar el selector
- Prueba enviar la búsqueda con la tecla Enter escribiendo en el campo y presionando Enter con `await user.keyboard('{Enter}')` — el formulario se envía igual que con el botón porque es un `<form>` nativo

### `src/components/UserCard.tsx`

```tsx
// src/components/UserCard.tsx

interface UserCardProps {
  name:       string
  email:      string
  isVerified: boolean
  onEdit:     () => void
  onDelete:   () => void
}

export default function UserCard({
  name, email, isVerified, onEdit, onDelete,
}: UserCardProps) {
  return (
    <article aria-label={`Tarjeta de ${name}`}>
      <h3>{name}</h3>
      <p>{email}</p>
      {isVerified
        ? <span data-testid="verified-badge">✓ Verificado</span>
        : <span data-testid="unverified-badge">No verificado</span>
      }
      <button onClick={onEdit}   aria-label={`Editar ${name}`}>Editar</button>
      <button onClick={onDelete} aria-label={`Eliminar ${name}`}>Eliminar</button>
    </article>
  )
}
```

### Prueba esto

- Renderiza `<UserCard name="Ana García" email="ana@ejemplo.com" isVerified={true} onEdit={() => {}} onDelete={() => {}} />` — observa el badge "✓ Verificado" y los dos botones con sus `aria-label` dinámicos
- Cambia `isVerified` a `false` — el badge cambia a "No verificado" y el test de `verified-badge` fallaría; el de `unverified-badge` pasaría
- Añade una prop `role?: 'admin' | 'user'` y muestra una etiqueta adicional cuando `role === 'admin'` — escribe un test que verifique que la etiqueta aparece solo para administradores
- Haz clic en el botón "Editar" de la tarjeta renderizada en `App.tsx` y abre la consola — la función `onEdit` que pasaste se ejecuta, lo que confirma el flujo de callbacks
- Cambia el `aria-label` del botón de editar de `Editar ${name}` a simplemente `"Editar"` — el test `getByRole('button', { name: 'Editar Ana García' })` falla; esto muestra que los `aria-label` dinámicos hacen los tests más específicos y resistentes a colisiones
- Añade `data-testid="user-card"` al `<article>` — escribe un test que verifique que el elemento existe en el DOM y que tiene el `aria-label` correcto con `toHaveAttribute`

---

## Los tests — todos verificados y pasando (20/20)

### `src/__tests__/StatusBadge.test.tsx`

```tsx
// src/__tests__/StatusBadge.test.tsx

import { render, screen } from '@testing-library/react'
import StatusBadge         from '../components/StatusBadge'

describe('StatusBadge', () => {
  it('muestra el texto por defecto según el status', () => {
    render(<StatusBadge status="active" />)
    expect(screen.getByTestId('status-badge')).toHaveTextContent('Activo')
  })

  it('muestra el label personalizado cuando se proporciona', () => {
    render(<StatusBadge status="active" label="En línea" />)
    expect(screen.getByTestId('status-badge')).toHaveTextContent('En línea')
  })

  it('renderiza correctamente para cada status', () => {
    const casos: Array<{ status: 'active' | 'inactive' | 'pending'; texto: string }> = [
      { status: 'active',   texto: 'Activo'    },
      { status: 'inactive', texto: 'Inactivo'  },
      { status: 'pending',  texto: 'Pendiente' },
    ]

    for (const { status, texto } of casos) {
      const { unmount } = render(<StatusBadge status={status} />)
      expect(screen.getByTestId('status-badge')).toHaveTextContent(texto)
      unmount()  // limpia el DOM entre iteraciones
    }
  })
})
```

### Prueba esto

- Ejecuta `npm test` con estos tests escritos — los tres tests pasan y Vitest muestra el árbol "StatusBadge > muestra el texto por defecto..." en verde
- Cambia `toHaveTextContent('Activo')` a `toHaveTextContent('Activos')` — el test falla con un mensaje claro: "Expected element to have text content: Activos, Received: Activo"
- Añade un cuarto test para el label personalizado con `status="pending"` y `label="En revisión"` — ejecuta los tests y observa que el nuevo test aparece en el árbol con el nombre que le diste en `it(...)`
- Elimina `unmount()` del bucle del tercer test — el test lanza un error porque hay múltiples elementos con `data-testid="status-badge"` y `getByTestId` solo espera encontrar uno
- Cambia `getByTestId('status-badge')` por `getByText('Activo')` en el primer test — el test sigue pasando; observa que `getByText` es menos específico pero igualmente válido para este caso
- Añade `it.skip('test pendiente', () => { ... })` para un cuarto status `'error'` aún no implementado — Vitest muestra el test como "skipped" en amarillo sin fallar la suite

### `src/__tests__/Counter.test.tsx`

```tsx
// src/__tests__/Counter.test.tsx

import { render, screen } from '@testing-library/react'
import userEvent            from '@testing-library/user-event'
import Counter              from '../components/Counter'

describe('Counter', () => {
  it('muestra el valor inicial por defecto (0)', () => {
    render(<Counter />)
    expect(screen.getByTestId('count-display')).toHaveTextContent('Cuenta: 0')
  })

  it('respeta el valor inicial personalizado', () => {
    render(<Counter initialValue={10} />)
    expect(screen.getByTestId('count-display')).toHaveTextContent('Cuenta: 10')
  })

  it('incrementa al hacer clic en Incrementar', async () => {
    const user = userEvent.setup()
    render(<Counter />)
    await user.click(screen.getByRole('button', { name: 'Incrementar' }))
    expect(screen.getByTestId('count-display')).toHaveTextContent('Cuenta: 1')
  })

  it('decrementa al hacer clic en Decrementar', async () => {
    const user = userEvent.setup()
    render(<Counter initialValue={5} />)
    await user.click(screen.getByRole('button', { name: 'Decrementar' }))
    expect(screen.getByTestId('count-display')).toHaveTextContent('Cuenta: 4')
  })

  it('respeta el step personalizado', async () => {
    const user = userEvent.setup()
    render(<Counter step={5} />)
    await user.click(screen.getByRole('button', { name: 'Incrementar' }))
    expect(screen.getByTestId('count-display')).toHaveTextContent('Cuenta: 5')
  })

  it('resetea al valor inicial al hacer clic en Reset', async () => {
    const user = userEvent.setup()
    render(<Counter initialValue={3} />)
    await user.click(screen.getByRole('button', { name: 'Incrementar' }))
    await user.click(screen.getByRole('button', { name: 'Incrementar' }))
    await user.click(screen.getByRole('button', { name: 'Reset' }))
    expect(screen.getByTestId('count-display')).toHaveTextContent('Cuenta: 3')
  })

  it('llama a onCountChange con el nuevo valor', async () => {
    const user     = userEvent.setup()
    const onChange = vi.fn()
    render(<Counter onCountChange={onChange} />)
    await user.click(screen.getByRole('button', { name: 'Incrementar' }))
    expect(onChange).toHaveBeenCalledWith(1)
    expect(onChange).toHaveBeenCalledTimes(1)
  })
})
```

### Prueba esto

- Ejecuta `npm test` y observa que los 7 tests de `Counter` pasan — el árbol muestra cada descripción del `it(...)` como una fila verde independiente
- Cambia `toHaveTextContent('Cuenta: 1')` a `toHaveTextContent('Cuenta: 2')` en el test de incremento — el test falla con el diff exacto: "Expected: Cuenta: 2 / Received: Cuenta: 1"
- Elimina el `await` antes de `user.click(...)` en el test de incremento — el test puede pasar o fallar de forma no determinista (falso positivo); añade el `await` de vuelta para entender por qué es obligatorio
- Añade un test que haga clic en "Incrementar" tres veces seguidas y verifique que el contador muestra "Cuenta: 3" — usa tres `await user.click(...)` consecutivos en el mismo test
- Cambia `vi.fn()` por `vi.fn().mockReturnValue(undefined)` en el test del callback — el comportamiento es idéntico; observa que `mockReturnValue` es útil cuando el callback necesita retornar un valor específico
- Añade `expect(onChange).not.toHaveBeenCalledWith(0)` al test del callback después de incrementar — el test pasa porque el callback se llamó con `1`, no con `0`
- Escribe un test que haga clic en "Decrementar" desde `initialValue={0}` y verifique que el contador muestra "Cuenta: -1" — ejecuta el test para confirmar que el componente actual permite valores negativos

### `src/__tests__/SearchInput.test.tsx`

```tsx
// src/__tests__/SearchInput.test.tsx

import { render, screen } from '@testing-library/react'
import userEvent            from '@testing-library/user-event'
import SearchInput          from '../components/SearchInput'

describe('SearchInput', () => {
  it('renderiza el input y el botón', () => {
    render(<SearchInput onSearch={vi.fn()} />)
    expect(screen.getByRole('textbox', { name: 'Campo de búsqueda' })).toBeInTheDocument()
    expect(screen.getByRole('button', { name: 'Buscar' })).toBeInTheDocument()
  })

  it('muestra error si el query es menor al mínimo', async () => {
    const user = userEvent.setup()
    render(<SearchInput onSearch={vi.fn()} minLength={3} />)
    await user.type(screen.getByRole('textbox'), 'ab')
    await user.click(screen.getByRole('button', { name: 'Buscar' }))
    expect(screen.getByRole('alert')).toHaveTextContent('Mínimo 3 caracteres')
  })

  it('llama a onSearch con el valor correcto', async () => {
    const user     = userEvent.setup()
    const onSearch = vi.fn()
    render(<SearchInput onSearch={onSearch} />)
    await user.type(screen.getByRole('textbox'), 'react')
    await user.click(screen.getByRole('button', { name: 'Buscar' }))
    expect(onSearch).toHaveBeenCalledWith('react')
    expect(onSearch).toHaveBeenCalledTimes(1)
  })

  it('no muestra error si el query es válido', async () => {
    const user = userEvent.setup()
    render(<SearchInput onSearch={vi.fn()} />)
    await user.type(screen.getByRole('textbox'), 'typescript')
    await user.click(screen.getByRole('button', { name: 'Buscar' }))
    expect(screen.queryByRole('alert')).not.toBeInTheDocument()
  })

  it('usa el placeholder proporcionado', () => {
    render(<SearchInput onSearch={vi.fn()} placeholder="Buscar productos..." />)
    expect(screen.getByPlaceholderText('Buscar productos...')).toBeInTheDocument()
  })
})
```

### Prueba esto

- Ejecuta `npm test` — los 5 tests de `SearchInput` pasan; observa en la salida que el test de error de validación lleva más tiempo porque `userEvent.type` simula cada pulsación de tecla
- Cambia `minLength={3}` a `minLength={5}` en el test de error de validación pero mantén `'ab'` como entrada — el error cambia a "Mínimo 5 caracteres"; actualiza la aserción para que el test vuelva a pasar
- En el test de `onSearch`, cambia `'react'` por `'  react  '` (con espacios) como entrada de `user.type` — observa que `onSearch` se llama con `'react'` sin espacios porque `handleSubmit` llama a `query.trim()` antes de pasar el valor
- Añade un test que verifique que `onSearch` no se llama si el query está vacío y se hace clic en "Buscar" — usa `expect(onSearch).not.toHaveBeenCalled()` para la aserción
- Cambia `screen.queryByRole('alert')` a `screen.getByRole('alert')` en el test de ausencia de error — el test lanza una excepción en lugar de fallar limpiamente; esto ilustra cuándo usar `queryBy*` frente a `getBy*`
- Escribe un test que verifique que el mensaje de error desaparece al escribir una búsqueda válida después de haber recibido un error: envía un query corto, observa el error, escribe uno válido y vuelve a enviar, luego verifica con `queryByRole('alert')` que el error ya no está

### `src/__tests__/UserCard.test.tsx`

```tsx
// src/__tests__/UserCard.test.tsx

import { render, screen } from '@testing-library/react'
import userEvent            from '@testing-library/user-event'
import UserCard             from '../components/UserCard'

// Props por defecto reutilizables en todos los tests
const DEFAULT_PROPS = {
  name:       'Ana García',
  email:      'ana@ejemplo.com',
  isVerified: true,
  onEdit:     vi.fn(),
  onDelete:   vi.fn(),
}

describe('UserCard', () => {
  it('muestra nombre y email', () => {
    render(<UserCard {...DEFAULT_PROPS} />)
    expect(screen.getByRole('heading', { name: 'Ana García' })).toBeInTheDocument()
    expect(screen.getByText('ana@ejemplo.com')).toBeInTheDocument()
  })

  it('muestra badge verificado cuando isVerified es true', () => {
    render(<UserCard {...DEFAULT_PROPS} isVerified={true} />)
    expect(screen.getByTestId('verified-badge')).toBeInTheDocument()
    expect(screen.queryByTestId('unverified-badge')).not.toBeInTheDocument()
  })

  it('muestra badge no verificado cuando isVerified es false', () => {
    render(<UserCard {...DEFAULT_PROPS} isVerified={false} />)
    expect(screen.getByTestId('unverified-badge')).toBeInTheDocument()
    expect(screen.queryByTestId('verified-badge')).not.toBeInTheDocument()
  })

  it('llama a onEdit al hacer clic en Editar', async () => {
    const user   = userEvent.setup()
    const onEdit = vi.fn()
    render(<UserCard {...DEFAULT_PROPS} onEdit={onEdit} />)
    await user.click(screen.getByRole('button', { name: 'Editar Ana García' }))
    expect(onEdit).toHaveBeenCalledTimes(1)
  })

  it('llama a onDelete al hacer clic en Eliminar', async () => {
    const user     = userEvent.setup()
    const onDelete = vi.fn()
    render(<UserCard {...DEFAULT_PROPS} onDelete={onDelete} />)
    await user.click(screen.getByRole('button', { name: 'Eliminar Ana García' }))
    expect(onDelete).toHaveBeenCalledTimes(1)
  })
})
```

### Prueba esto

- Ejecuta `npm test` — los 5 tests de `UserCard` pasan; observa que los tests de `onEdit` y `onDelete` usan `vi.fn()` local en lugar del `DEFAULT_PROPS.onEdit` para poder verificar llamadas de forma aislada
- Cambia `name: 'Ana García'` en `DEFAULT_PROPS` a `name: 'Carlos López'` — todos los tests fallan porque los `aria-label` dinámicos de los botones cambian a "Editar Carlos López" y "Eliminar Carlos López"
- Haz clic dos veces en el botón "Editar" dentro del test `it('llama a onEdit...')` añadiendo otro `await user.click(...)` — cambia la aserción a `toHaveBeenCalledTimes(2)` para ver que el mock registra múltiples llamadas
- Añade `beforeEach(() => { vi.clearAllMocks() })` al `describe` de `UserCard` — esto reinicia los contadores de los mocks entre tests para evitar que llamadas de un test contaminen las aserciones del siguiente
- Añade un test que verifique que el artículo tiene el `aria-label` correcto: `expect(screen.getByRole('article')).toHaveAttribute('aria-label', 'Tarjeta de Ana García')` — el test pasa porque el componente usa `aria-label={Tarjeta de ${name}}`
- Escribe un test que renderice dos `UserCard` con nombres diferentes y verifique que puedes seleccionar cada botón de forma independiente usando `getByRole('button', { name: 'Editar Ana García' })` y `getByRole('button', { name: 'Editar Carlos López' })`

---

## Conceptos clave de RTL

### Queries — cómo buscar elementos

```tsx
// getBy* — lanza si no existe o hay más de uno
screen.getByRole('button', { name: 'Enviar' })
screen.getByText('Hola mundo')
screen.getByTestId('count-display')
screen.getByPlaceholderText('Buscar...')
screen.getByLabelText('Email')

// queryBy* — retorna null si no existe (no lanza)
// Útil para verificar que algo NO está en el DOM
screen.queryByRole('alert')       // null si no hay alerta
screen.queryByTestId('error-msg') // null si no hay error

// findBy* — async, espera a que aparezca
await screen.findByText('Cargando completo')
await screen.findByRole('status')
```

### Prioridad de queries — de más a menos preferida

```
1. getByRole           ← accesibilidad real del elemento
2. getByLabelText      ← inputs asociados a su label
3. getByPlaceholderText
4. getByText           ← contenido visible
5. getByDisplayValue   ← valor actual de inputs/selects
6. getByTestId         ← último recurso — añade data-testid al componente
```

> `getByRole` es la query más robusta: usa el árbol de accesibilidad
> real del navegador. Si tu componente es accesible, los tests con
> `getByRole` también pasan con lectores de pantalla.

### `userEvent` vs `fireEvent`

```tsx
// fireEvent — simula el evento de DOM directamente (simple)
import { fireEvent } from '@testing-library/react'
fireEvent.click(button)
fireEvent.change(input, { target: { value: 'texto' } })

// userEvent — simula el comportamiento real del usuario (preferido)
// Dispara todos los eventos intermedios (focus, keydown, keyup, etc.)
const user = userEvent.setup()
await user.click(button)
await user.type(input, 'texto')     // dispara cada keystroke
await user.keyboard('{Enter}')
await user.selectOptions(select, 'opción')
await user.clear(input)
```

> Usa `userEvent` por defecto. Es más fiel al comportamiento real.
> `fireEvent` sirve para casos donde necesitas más control.

### Matchers de `jest-dom`

```tsx
expect(element).toBeInTheDocument()
expect(element).not.toBeInTheDocument()
expect(element).toBeVisible()
expect(element).toBeDisabled()
expect(element).toBeEnabled()
expect(element).toHaveTextContent('texto')
expect(element).toHaveValue('valor')
expect(element).toHaveAttribute('type', 'email')
expect(element).toHaveClass('mi-clase')
expect(element).toHaveFocus()
expect(element).toBeChecked()
```

---

## Testear callbacks con `vi.fn()`

`vi.fn()` crea una función mock — registra cada llamada para que puedas
verificar cuántas veces fue llamada y con qué argumentos:

```tsx
it('llama a onSearch con el término correcto', async () => {
  const user     = userEvent.setup()
  const onSearch = vi.fn()          // función mock

  render(<SearchInput onSearch={onSearch} />)
  await user.type(screen.getByRole('textbox'), 'react')
  await user.click(screen.getByRole('button', { name: 'Buscar' }))

  expect(onSearch).toHaveBeenCalledTimes(1)        // se llamó exactamente una vez
  expect(onSearch).toHaveBeenCalledWith('react')   // con este argumento
  expect(onSearch).not.toHaveBeenCalledWith('React') // no con este otro
})
```

### Prueba esto

- Copia este test a tu archivo de tests, ejecútalo con `npm test` — pasa; modifica el argumento de `toHaveBeenCalledWith` a `'React'` con mayúscula — falla con el mensaje "Expected: React / Received: react"
- Añade `expect(onSearch).toHaveBeenCalledTimes(2)` al final del test sin hacer un segundo clic — el test falla con "Expected number of calls: 2 / Received: 1", que es el error más común con mocks
- Reemplaza `vi.fn()` por `vi.fn().mockImplementation((q) => console.log('Buscando:', q))` — el mock ahora ejecuta código real; útil para depurar qué argumentos recibe sin cambiar el test
- Llama a `onSearch.mockReset()` antes de la segunda parte del test si necesitas verificar llamadas desde cero — esto reinicia `toHaveBeenCalledTimes` a 0 sin necesidad de crear un nuevo mock
- Añade `expect(onSearch).toHaveBeenLastCalledWith('react')` — esta aserción verifica solo la última llamada, útil cuando el mock se llama varias veces y solo te interesa el resultado final
- Verifica el argumento por posición con `expect(onSearch.mock.calls[0][0]).toBe('react')` — accede directamente al registro de llamadas para casos donde los matchers estándar no son suficientes

---

## Errores comunes en tests RTL

```tsx
// ❌ Buscar por clase CSS — frágil, rompe si cambias el estilo
screen.getByClass('btn-primary')

// ❌ Buscar por selector CSS — frágil
document.querySelector('.card > p')

// ✅ Buscar por rol accesible — robusto
screen.getByRole('button', { name: 'Enviar' })

// ❌ Olvidar await en userEvent — el test pasa aunque falle
user.click(button)
expect(screen.getByText('Resultado')).toBeInTheDocument() // puede ser falso positivo

// ✅ Siempre await con userEvent
await user.click(button)
expect(screen.getByText('Resultado')).toBeInTheDocument()

// ❌ Usar getBy* cuando el elemento puede no existir
screen.getByText('Error')   // lanza si no hay error — falso negativo

// ✅ queryBy* para verificar ausencia
expect(screen.queryByText('Error')).not.toBeInTheDocument()

// ❌ Testear implementación interna
expect(component.state.count).toBe(1)   // accede al estado directamente

// ✅ Testear lo que el usuario ve
expect(screen.getByTestId('count-display')).toHaveTextContent('Cuenta: 1')
```

---

## Ejercicios propuestos

1. **Testear `StatusBadge` de página 2** — añade `data-testid="status-badge"` al componente
   `StatusBadge` que creaste en la página 2 y escribe tests para verificar
   que cada variante (`active`, `inactive`, `pending`, `error`) muestra el texto correcto.

2. **Testear `ContactForm` de página 12** — escribe tests para verificar que:
   - El formulario muestra errores si se envía vacío.
   - El error desaparece al escribir en el campo.
   - El mensaje de éxito aparece tras un envío correcto (usa `findByText` para esperar async).

3. **Testear `useCounter`** — instala `@testing-library/react-hooks` o usa `renderHook`
   de RTL para testear el hook directamente sin montar un componente completo.

---

## Resumen de la página 13

- Vitest es el test runner — compatible con la configuración de Vite, sin configuración extra.
- RTL testea comportamiento del usuario, no implementación interna.
- La configuración mínima: `vite.config.ts` con `test: { environment: 'jsdom', globals: true }` + `setup.ts` con `import '@testing-library/jest-dom'`.
- `getByRole` es la query más robusta — usa el árbol de accesibilidad real.
- `queryBy*` retorna `null` si no existe — úsalo para verificar ausencia con `.not.toBeInTheDocument()`.
- `findBy*` es async — úsalo cuando el elemento aparece después de una operación async.
- `userEvent.setup()` + `await user.click/type/...` simula el comportamiento real del usuario.
- `vi.fn()` crea mocks de funciones — verifica llamadas con `toHaveBeenCalledWith` y `toHaveBeenCalledTimes`.
- `data-testid` es el último recurso — primero intenta con `getByRole`, `getByLabelText` o `getByText`.

---

> **Siguiente página →** Fetching con TanStack Query (React Query):
> caché automático, estados de carga, refetching y mutaciones tipadas.