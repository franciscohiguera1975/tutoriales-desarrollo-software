# Curso React.js + TypeScript — Página 9
## Módulo 4 · Hooks personalizados
### Crear, tipar y organizar hooks reutilizables

---

## ¿Qué es un hook personalizado?

Un **hook personalizado** es una función TypeScript cuyo nombre comienza
con `use` que puede llamar otros hooks internamente. Su único propósito es
**extraer lógica de estado y efectos** de los componentes para reutilizarla.

Ahora que conoces todos los hooks nativos, los hooks personalizados son
simplemente combinaciones de ellos encapsuladas con un nombre descriptivo.

```
useToggle       = useState + useCallback
useDebounce     = useState + useEffect (con clearTimeout)
useFetch        = useState + useEffect (con fetch + cancelled flag)
useLocalStorage = useState + useEffect (con localStorage)
useMediaQuery   = useState + useEffect (con matchMedia)
useClipboard    = useState + useCallback (con navigator.clipboard)
```

### Reglas que aplican igual que en hooks nativos

1. Solo en el nivel superior — nunca dentro de `if`, `for`, funciones anidadas.
2. Solo en componentes funcionales o en otros hooks personalizados.
3. El nombre **debe empezar con `use`** — React y ESLint lo exigen.

### Estructura de archivos

```
src/
├── hooks/            ← archivos .ts — sin JSX
│   ├── useToggle.ts
│   ├── useCounter.ts
│   ├── useLocalStorage.ts
│   ├── useDebounce.ts
│   ├── useFetch.ts
│   ├── useWindowSize.ts
│   ├── useOnlineStatus.ts
│   ├── useMediaQuery.ts
│   └── useClipboard.ts
├── components/       ← archivos .tsx — con JSX
└── App.tsx
```

---

## `src/hooks/useToggle.ts`

Estado booleano con tres funciones de control. `useCallback` garantiza
referencias estables para no causar re-renders innecesarios en hijos.

```ts
// src/hooks/useToggle.ts

import { useState, useCallback } from 'react'

export function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue)

  const toggle   = useCallback(() => setValue((v) => !v), [])
  const setTrue  = useCallback(() => setValue(true),      [])
  const setFalse = useCallback(() => setValue(false),     [])

  return { value, toggle, setTrue, setFalse }
}
```

```tsx
// Uso — renombra value al desestructurar para mayor claridad
import { useToggle } from '../hooks/useToggle'

export default function ModalDemo() {
  const { value: isOpen, toggle, setFalse } = useToggle()

  return (
    <>
      <button onClick={toggle}>Abrir modal</button>
      {isOpen && (
        <div style={{
          position: 'fixed', inset: 0,
          background: 'rgba(0,0,0,0.4)',
          display: 'flex', alignItems: 'center', justifyContent: 'center',
        }}>
          <div style={{
            background: '#fff', borderRadius: 10,
            padding: 24, minWidth: 300,
          }}>
            <h3 style={{ marginTop: 0 }}>Modal</h3>
            <p>Contenido del modal.</p>
            <button onClick={setFalse}>Cerrar</button>
          </div>
        </div>
      )}
    </>
  )
}
```

### Prueba esto

- Haz clic en "Abrir modal" y luego en "Cerrar" — observa que el overlay desaparece y el botón vuelve a estar disponible
- Cambia `useToggle()` a `useToggle(true)` — observa que el modal aparece ya abierto al cargar la página
- Añade un segundo `useToggle` para un segundo modal — comprueba que ambos estados son completamente independientes
- Reemplaza el botón "Cerrar" por `setTrue` — verifica que el modal ya no se puede cerrar (y que `setTrue` es una referencia estable que no produce re-renders)
- Haz clic en el overlay oscuro (fuera del panel blanco) y comprueba que el modal no se cierra porque no hay `onClick` en el fondo — añade `onClick={setFalse}` al div del overlay para corregirlo
- Abre las DevTools de React y observa que `toggle`, `setTrue` y `setFalse` mantienen la misma referencia entre renders gracias a `useCallback`

### `src/App.tsx`

```tsx
// src/App.tsx

import ModalDemo from './components/ModalDemo'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  ModalDemo        — useToggle: modal con overlay                 │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

export default function App() {
  const content =
    PASO === 1 ? <ModalDemo /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 600, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

---

## `src/hooks/useCounter.ts`

Contador con límites y paso configurables. Demuestra cómo tipar
opciones con `interface` y retornar una interfaz bien definida.

```ts
// src/hooks/useCounter.ts

import { useState, useCallback } from 'react'

interface UseCounterOptions {
  initialValue?: number
  min?:          number
  max?:          number
  step?:         number
}

interface UseCounterReturn {
  count:     number
  increment: () => void
  decrement: () => void
  reset:     () => void
  set:       (value: number) => void
}

export function useCounter({
  initialValue = 0,
  min          = -Infinity,
  max          = Infinity,
  step         = 1,
}: UseCounterOptions = {}): UseCounterReturn {
  const [count, setCount] = useState(initialValue)

  const increment = useCallback(
    () => setCount((prev) => Math.min(prev + step, max)),
    [step, max]
  )
  const decrement = useCallback(
    () => setCount((prev) => Math.max(prev - step, min)),
    [step, min]
  )
  const reset = useCallback(() => setCount(initialValue), [initialValue])
  const set   = useCallback(
    (value: number) => setCount(Math.min(Math.max(value, min), max)),
    [min, max]
  )

  return { count, increment, decrement, reset, set }
}
```

```tsx
// Uso
import { useCounter } from '../hooks/useCounter'

export default function QuantitySelector() {
  const { count, increment, decrement, reset } = useCounter({
    initialValue: 1, min: 1, max: 99,
  })

  return (
    <div style={{ display: 'flex', alignItems: 'center', gap: 8 }}>
      <button
        onClick={decrement}
        disabled={count === 1}
        style={qBtn}
      >
        −
      </button>
      <span style={{ minWidth: 32, textAlign: 'center', fontWeight: 600 }}>
        {count}
      </span>
      <button
        onClick={increment}
        disabled={count === 99}
        style={qBtn}
      >
        +
      </button>
      <button onClick={reset} style={{ ...qBtn, fontSize: 11, color: '#9ca3af' }}>
        Reset
      </button>
    </div>
  )
}

const qBtn = {
  width: 30, height: 30, border: '1px solid #d1d5db',
  borderRadius: 6, background: '#f9fafb', cursor: 'pointer',
}
```

### Prueba esto

- Haz clic en "−" cuando el contador está en `1` — observa que el botón está desactivado y el valor no baja de `min`
- Haz clic en "+" hasta llegar a `99` — comprueba que el botón se desactiva automáticamente al alcanzar `max`
- Cambia `max: 99` a `max: 5` y verifica que el contador se detiene en `5`
- Cambia `step: 1` a `step: 10` — observa que cada clic incrementa el valor de diez en diez
- Haz clic en "Reset" desde cualquier valor — verifica que regresa exactamente al `initialValue` que configuraste
- Añade `initialValue: 50` y comprueba que el contador empieza en `50` en lugar de `1`
- Llama a `set(200)` desde la consola de React DevTools o un botón de prueba — observa que el valor queda limitado a `max`

### Agrega a `src/App.tsx`

```tsx
import QuantitySelector from './components/QuantitySelector'
```

```tsx
PASO === 2 ? <QuantitySelector /> :
```

Cambia `PASO = 2` y guarda.

---

## `src/hooks/useLocalStorage.ts`

Hook genérico con `<T>`. El tipo del valor se infiere del argumento inicial.
`as const` en el retorno fuerza una tupla `[T, Dispatch]` en lugar de
`(T | Dispatch)[]` — idéntico a la desestructuración de `useState`.

```ts
// src/hooks/useLocalStorage.ts

import { useState, useEffect } from 'react'

export function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = localStorage.getItem(key)
      return item ? (JSON.parse(item) as T) : initialValue
    } catch {
      return initialValue
    }
  })

  useEffect(() => {
    try {
      localStorage.setItem(key, JSON.stringify(storedValue))
    } catch {
      console.warn(`useLocalStorage: no se pudo guardar "${key}"`)
    }
  }, [key, storedValue])

  return [storedValue, setStoredValue] as const
}
```

```tsx
// Uso — API idéntica a useState, pero persiste entre recargas
import { useLocalStorage } from '../hooks/useLocalStorage'

export default function ThemeSelector() {
  const [theme, setTheme] = useLocalStorage<'light' | 'dark'>('theme', 'light')

  return (
    <div style={{ display: 'flex', gap: 8 }}>
      {(['light', 'dark'] as const).map((t) => (
        <button
          key={t}
          onClick={() => setTheme(t)}
          style={{
            padding: '6px 14px', borderRadius: 6,
            border: '1px solid #d1d5db',
            background: theme === t ? '#0070f3' : '#fff',
            color:      theme === t ? '#fff'    : '#333',
            cursor: 'pointer',
          }}
        >
          {t === 'light' ? '☀️ Claro' : '🌙 Oscuro'}
        </button>
      ))}
    </div>
  )
}
```

### Prueba esto

- Selecciona "Oscuro", recarga la página (F5) — observa que el tema persiste sin perder el valor gracias a `localStorage`
- Abre las DevTools del navegador → Application → Local Storage → `localhost` — comprueba que la clave `"theme"` aparece con el valor `"dark"` o `"light"`
- Elimina manualmente la clave `"theme"` en DevTools y recarga — verifica que el hook vuelve al `initialValue` (`'light'`)
- Cambia la clave de `'theme'` a `'app-theme'` — observa que el hook crea una nueva entrada en localStorage y la antigua queda huérfana
- Usa el hook con un tipo distinto, por ejemplo `useLocalStorage<number>('contador', 0)`, y comprueba que TypeScript infiere el tipo correcto en `setStoredValue`
- Abre dos pestañas del mismo origen y cambia el tema en una — observa que la otra pestaña no se actualiza en tiempo real (el hook no escucha `storage` events por defecto)

### Agrega a `src/App.tsx`

```tsx
import ThemeSelector from './components/ThemeSelector'
```

```tsx
PASO === 3 ? <ThemeSelector /> :
```

Cambia `PASO = 3` y guarda.

---

## `src/hooks/useDebounce.ts`

Genérico `<T>` que aplica debounce a cualquier valor.
Encapsula el patrón `setTimeout` + `clearTimeout` de página 5.

```ts
// src/hooks/useDebounce.ts

import { useState, useEffect } from 'react'

export function useDebounce<T>(value: T, delay = 500): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(timer)
  }, [value, delay])

  return debouncedValue
}
```

```tsx
// Uso — el componente queda libre de gestionar el timer
import { useState }    from 'react'
import { useDebounce } from '../hooks/useDebounce'

export default function LiveSearch() {
  const [query, setQuery]  = useState('')
  const debouncedQuery     = useDebounce(query, 400)

  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: 8, maxWidth: 320 }}>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Buscar..."
        style={{ padding: '8px 12px', border: '1px solid #d1d5db', borderRadius: 6 }}
      />
      <p style={{ margin: 0, fontSize: 13, color: '#6b7280' }}>
        Query activa (400ms): <strong>{debouncedQuery || '—'}</strong>
      </p>
    </div>
  )
}
```

### Prueba esto

- Escribe rápidamente varias letras en el input — observa que `debouncedQuery` solo se actualiza cuando dejas de escribir durante 400ms
- Cambia el delay de `400` a `2000` — comprueba que ahora tienes que esperar dos segundos de inactividad para que la query se active
- Cambia el delay a `0` — verifica que `debouncedQuery` se actualiza en cada tecla sin retraso, comportándose como un `useState` normal
- Escribe algo, espera a que `debouncedQuery` se actualice, y luego borra todo el input — observa que `debouncedQuery` vuelve a `''` y muestra `'—'`
- Añade un segundo párrafo que muestre `query` (sin debounce) junto a `debouncedQuery` — compara ambos valores mientras escribes para ver la diferencia
- Aplica `useDebounce` a un número en lugar de un string: `useDebounce<number>(count, 300)` — comprueba que el genérico `<T>` funciona con cualquier tipo

### Agrega a `src/App.tsx`

```tsx
import LiveSearch from './components/LiveSearch'
```

```tsx
PASO === 4 ? <LiveSearch /> :
```

Cambia `PASO = 4` y guarda.

---

## `src/hooks/useFetch.ts`

Fetch genérico con estado completo y cancelación anti-race-condition.
Extrae toda la complejidad de página 5 en un hook de una línea de uso.

```ts
// src/hooks/useFetch.ts

import { useState, useEffect } from 'react'

interface FetchState<T> {
  data:    T | null
  loading: boolean
  error:   string | null
}

export function useFetch<T>(url: string) {
  const [state, setState] = useState<FetchState<T>>({
    data: null, loading: true, error: null,
  })

  useEffect(() => {
    let cancelled = false

    async function fetchData() {
      setState((prev) => ({ ...prev, loading: true, error: null }))
      try {
        const res = await fetch(url)
        if (!res.ok) throw new Error(`HTTP ${res.status}`)
        const data: T = await res.json()
        if (!cancelled) setState({ data, loading: false, error: null })
      } catch (err) {
        if (!cancelled) setState({
          data:    null,
          loading: false,
          error:   err instanceof Error ? err.message : 'Error desconocido',
        })
      }
    }

    fetchData()
    return () => { cancelled = true }
  }, [url])

  return state
}
```

```tsx
// Uso — toda la lógica de fetch en una línea
import { useFetch } from '../hooks/useFetch'

interface Post { id: number; title: string; body: string }

export default function PostList() {
  const { data: posts, loading, error } = useFetch<Post[]>(
    'https://jsonplaceholder.typicode.com/posts?_limit=5'
  )

  if (loading) return <p style={{ color: '#6b7280' }}>Cargando...</p>
  if (error)   return <p style={{ color: '#ef4444' }}>Error: {error}</p>

  return (
    <ul style={{ listStyle: 'none', padding: 0, display: 'flex', flexDirection: 'column', gap: 10 }}>
      {posts?.map((post) => (
        <li key={post.id} style={{ padding: 14, border: '1px solid #e5e7eb', borderRadius: 8 }}>
          <p style={{ margin: '0 0 4px', fontWeight: 600, fontSize: 14 }}>{post.title}</p>
          <p style={{ margin: 0, fontSize: 13, color: '#6b7280' }}>
            {post.body.slice(0, 80)}...
          </p>
        </li>
      ))}
    </ul>
  )
}
```

### Prueba esto

- Cambia la URL a `'https://jsonplaceholder.typicode.com/posts/99999'` — observa cómo el estado `error` recibe `"HTTP 404"` y el componente muestra el mensaje de error
- Cambia la URL a `'https://url-que-no-existe.xyz/posts'` — verifica que el error de red (fallo de conexión) también queda capturado en el estado `error`
- Cambia `?_limit=5` a `?_limit=20` y observa que la lista crece sin modificar nada más en el componente
- Pasa la URL como prop `url: string` al componente en lugar de tenerla hardcodeada — comprueba que cambiar la prop desde el padre desencadena un nuevo fetch gracias a `[url]` en las dependencias del `useEffect`
- Agrega `console.log('fetchData called')` dentro de `fetchData` y monta/desmonta el componente varias veces — verifica que el flag `cancelled` evita que se actualice el estado tras el desmontaje
- Cambia el tipo genérico a `useFetch<Post>` (singular) con una URL de post individual (`/posts/1`) — TypeScript debe inferir que `data` es `Post | null`

### Agrega a `src/App.tsx`

```tsx
import PostList from './components/PostList'
```

```tsx
PASO === 5 ? <PostList /> :
```

Cambia `PASO = 5` y guarda.

---

## `src/hooks/useWindowSize.ts`

```ts
// src/hooks/useWindowSize.ts

import { useState, useEffect } from 'react'

interface WindowSize {
  width:  number
  height: number
}

export function useWindowSize(): WindowSize {
  const [size, setSize] = useState<WindowSize>({
    width:  window.innerWidth,
    height: window.innerHeight,
  })

  useEffect(() => {
    function handleResize() {
      setSize({ width: window.innerWidth, height: window.innerHeight })
    }
    window.addEventListener('resize', handleResize)
    return () => window.removeEventListener('resize', handleResize)
  }, [])

  return size
}
```

### Prueba esto

- Redimensiona la ventana del navegador arrastrando su borde — observa que `width` y `height` se actualizan en tiempo real
- Usa las DevTools del navegador (F12) y activa la vista de dispositivo móvil — comprueba que `width` refleja el ancho del viewport emulado
- Monta el hook en dos componentes distintos — verifica que cada uno tiene su propio listener y ambos valores se sincronizan al mismo tiempo
- Desmonta el componente (navega a otro paso en el App) y redimensiona — confirma que no aparecen errores de "state update on unmounted component" gracias al cleanup del `useEffect`
- Añade `console.log('resize')` dentro de `handleResize` y redimensiona rápidamente — observa que cada pixel de cambio dispara el evento; considera añadir un debounce para optimizar

### Agrega a `src/App.tsx`

```tsx
import ResponsiveLayout from './components/ResponsiveLayout'
```

```tsx
PASO === 6 ? <ResponsiveLayout /> :
```

Cambia `PASO = 6` y guarda.

---

## `src/hooks/useOnlineStatus.ts`

```ts
// src/hooks/useOnlineStatus.ts

import { useState, useEffect } from 'react'

export function useOnlineStatus(): boolean {
  const [isOnline, setIsOnline] = useState(navigator.onLine)

  useEffect(() => {
    function handleOnline()  { setIsOnline(true)  }
    function handleOffline() { setIsOnline(false) }

    window.addEventListener('online',  handleOnline)
    window.addEventListener('offline', handleOffline)

    return () => {
      window.removeEventListener('online',  handleOnline)
      window.removeEventListener('offline', handleOffline)
    }
  }, [])

  return isOnline
}
```

### Prueba esto

- Abre las DevTools del navegador → Network → activa "Offline" — observa que `isOnline` cambia a `false` inmediatamente
- Vuelve a activar la conexión en DevTools → Network → "Online" — verifica que `isOnline` regresa a `true` sin recargar la página
- Renderiza el valor en pantalla como un badge: verde si `true`, rojo si `false` — comprueba el cambio de color al simular offline
- Monta el componente, desconecta la red y vuelve a montar — verifica que el estado inicial lee `navigator.onLine` correctamente en el momento del montaje
- Añade una notificación toast que aparezca solo cuando se pasa de `true` a `false` usando un `useEffect` que observe `isOnline`

---

## `src/hooks/useMediaQuery.ts`

Encapsula `window.matchMedia`. El genérico no es necesario aquí — siempre retorna `boolean`.
Útil para layout responsivo sin CSS media queries en JSX.

```ts
// src/hooks/useMediaQuery.ts

import { useState, useEffect } from 'react'

export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(
    () => window.matchMedia(query).matches
  )

  useEffect(() => {
    const mediaQuery = window.matchMedia(query)

    function handleChange(e: MediaQueryListEvent) {
      setMatches(e.matches)
    }

    mediaQuery.addEventListener('change', handleChange)
    return () => mediaQuery.removeEventListener('change', handleChange)
  }, [query])

  return matches
}
```

```tsx
// Uso — layout responsivo en el componente
import { useMediaQuery } from '../hooks/useMediaQuery'
import { useWindowSize } from '../hooks/useWindowSize'

export default function ResponsiveLayout() {
  const isMobile  = useMediaQuery('(max-width: 768px)')
  const isTablet  = useMediaQuery('(max-width: 1024px)')
  const { width } = useWindowSize()

  return (
    <div style={{
      padding: isMobile ? 12 : 24,
      display: 'grid',
      gridTemplateColumns: isMobile ? '1fr' : isTablet ? '1fr 1fr' : '1fr 1fr 1fr',
      gap: 12,
    }}>
      <div style={{ padding: 16, background: '#f9fafb', borderRadius: 8 }}>
        <p style={{ margin: 0, fontSize: 13, fontWeight: 600 }}>Vista actual</p>
        <p style={{ margin: '4px 0 0', fontSize: 12, color: '#6b7280' }}>
          {isMobile ? 'Móvil' : isTablet ? 'Tablet' : 'Escritorio'} — {width}px
        </p>
      </div>
      <div style={{ padding: 16, background: '#f9fafb', borderRadius: 8 }}>
        <p style={{ margin: 0, fontSize: 13, fontWeight: 600 }}>Columnas</p>
        <p style={{ margin: '4px 0 0', fontSize: 12, color: '#6b7280' }}>
          {isMobile ? 1 : isTablet ? 2 : 3}
        </p>
      </div>
      {!isMobile && (
        <div style={{ padding: 16, background: '#f9fafb', borderRadius: 8 }}>
          <p style={{ margin: 0, fontSize: 13, fontWeight: 600 }}>Extra</p>
          <p style={{ margin: '4px 0 0', fontSize: 12, color: '#6b7280' }}>
            Solo visible en tablet/escritorio
          </p>
        </div>
      )}
    </div>
  )
}
```

### Prueba esto

- Redimensiona la ventana del navegador a menos de 768px — observa que el layout cambia a una sola columna y la tarjeta "Extra" desaparece
- Redimensiona entre 769px y 1024px — verifica que el layout pasa a dos columnas (modo tablet)
- Usa las DevTools de Chrome con emulación de iPhone — comprueba que `isMobile` es `true` y `padding` cambia de `24` a `12`
- Cambia el breakpoint de `useMediaQuery('(max-width: 768px)')` a `(max-width: 1200px)` — observa que el umbral de "móvil" se amplía
- Añade un tercer `useMediaQuery('(prefers-color-scheme: dark)')` — cambia el esquema de color del sistema en las preferencias del SO y observa que el hook reacciona sin recargar la página
- Pasa una media query inválida como `'not-a-query'` al hook — observa si el navegador lanza un error o simplemente devuelve `false`

---

## `src/hooks/useClipboard.ts`

Copia texto al portapapeles y expone un estado `copied` que vuelve a
`false` automáticamente tras el delay. `resetDelay` es configurable.

```ts
// src/hooks/useClipboard.ts

import { useState, useCallback } from 'react'

interface UseClipboardReturn {
  copy:   (text: string) => Promise<void>
  copied: boolean
}

export function useClipboard(resetDelay = 2000): UseClipboardReturn {
  const [copied, setCopied] = useState(false)

  const copy = useCallback(async (text: string) => {
    try {
      await navigator.clipboard.writeText(text)
      setCopied(true)
      setTimeout(() => setCopied(false), resetDelay)
    } catch {
      console.warn('useClipboard: no se pudo copiar al portapapeles')
    }
  }, [resetDelay])

  return { copy, copied }
}
```

```tsx
// Uso — botón de copiar código con feedback visual
import { useClipboard } from '../hooks/useClipboard'

interface CodeBlockProps {
  code: string
  language?: string
}

export default function CodeBlock({ code, language = 'tsx' }: CodeBlockProps) {
  const { copy, copied } = useClipboard(1500)

  return (
    <div style={{ position: 'relative', borderRadius: 8, overflow: 'hidden' }}>
      <div style={{
        display: 'flex', justifyContent: 'space-between', alignItems: 'center',
        padding: '6px 12px', background: '#1e293b',
      }}>
        <span style={{ fontSize: 12, color: '#94a3b8' }}>{language}</span>
        <button
          onClick={() => copy(code)}
          style={{
            padding: '3px 10px', borderRadius: 4,
            border: '1px solid #334155',
            background: copied ? '#166534' : '#1e293b',
            color:      copied ? '#bbf7d0' : '#94a3b8',
            cursor: 'pointer', fontSize: 12,
            transition: 'background 0.2s, color 0.2s',
          }}
        >
          {copied ? '✓ Copiado' : 'Copiar'}
        </button>
      </div>
      <pre style={{
        margin: 0, padding: '12px 16px',
        background: '#0f172a', color: '#e2e8f0',
        fontSize: 13, overflowX: 'auto',
      }}>
        <code>{code}</code>
      </pre>
    </div>
  )
}
```

### Prueba esto

- Haz clic en "Copiar" y pega en un editor de texto — verifica que el contenido copiado coincide exactamente con el código mostrado
- Observa que el botón cambia a "✓ Copiado" (fondo verde) durante 1500ms y luego vuelve a "Copiar" automáticamente
- Cambia `resetDelay` de `1500` a `5000` — comprueba que el botón tarda más en volver al estado inicial
- Pasa `code=""` (string vacío) al componente — verifica que el portapapeles recibe un string vacío y el botón sigue funcionando sin errores
- Prueba el componente en un contexto sin HTTPS (como `http://localhost` pero con `http` explícito en otro origen) — observa si `navigator.clipboard.writeText` falla y comprueba que el `catch` muestra el warning en la consola sin romper la UI
- Renderiza dos `CodeBlock` con distintos códigos — comprueba que hacer clic en "Copiar" en uno no afecta al estado `copied` del otro
- Desde aquí puedes volver a cualquier PASO anterior cambiando el número y guardando.

### Agrega a `src/App.tsx`

```tsx
import CodeBlock from './components/CodeBlock'
```

```tsx
PASO === 7 ? <CodeBlock code={EXAMPLE_CODE} language="tsx" /> :
```

Cambia `PASO = 7` y guarda.

---

## Cuándo extraer un hook personalizado

| Extrae a hook cuando... | No extraigas cuando... |
|---|---|
| La misma lógica aparece en 2+ componentes | Solo se usa en un componente y es breve |
| Un componente tiene demasiada lógica mezclada | Mover JSX a otro lugar — eso es un componente |
| Quieres testear la lógica sin montar un componente | La extracción haría el código más difícil de seguir |
| La lógica involucra hooks y es reutilizable | La lógica es pura JavaScript — usa una función normal |

---

## Resumen de la página 9

- Los hooks personalizados son combinaciones de hooks nativos encapsuladas con un nombre descriptivo.
- Van en archivos `.ts` dentro de `src/hooks/` — sin JSX, sin extensión `.tsx`.
- El genérico `<T>` permite que el mismo hook funcione con cualquier tipo — TypeScript infiere `T` del argumento.
- `as const` en el retorno de una tupla fuerza el tipo correcto para desestructurar igual que `useState`.
- `useCallback` en los retornos de funciones garantiza referencias estables.
- `useMediaQuery` recibe el string de la media query y usa `MediaQueryListEvent` para el listener — tipo específico del DOM.
- La regla de extracción: misma lógica en 2+ componentes, o componente con demasiada lógica mezclada.

---

> **Siguiente página →** React Router v6: navegación entre páginas,
> rutas anidadas, parámetros de URL y rutas protegidas con TypeScript.