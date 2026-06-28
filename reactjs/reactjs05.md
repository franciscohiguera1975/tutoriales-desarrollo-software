# Curso React.js + TypeScript — Página 5
## Módulo 2 · Estado y efectos
### `useEffect`: sincronización, ciclo de vida y efectos secundarios

---

## ¿Qué es un efecto?

En React, un **efecto secundario** es cualquier operación que ocurre fuera
del renderizado puro: llamadas a APIs, subscripciones a eventos del navegador,
timers, manipulación directa del DOM, logs, etc.

`useEffect` es el hook que permite ejecutar estos efectos de forma sincronizada
con el ciclo de vida del componente — después de que React actualiza el DOM.

```tsx
import { useEffect } from 'react'

useEffect(() => {
  // 1. El efecto — se ejecuta después del render
  return () => {
    // 2. La limpieza — se ejecuta antes del próximo efecto
    //    o cuando el componente se desmonta (opcional)
  }
}, [/* 3. Array de dependencias — controla cuándo se re-ejecuta */])
```

---

## El array de dependencias

Es el elemento más importante de `useEffect`. Controla exactamente cuándo
se ejecuta el efecto:

```tsx
import { useEffect } from 'react'

// Sin array — se ejecuta después de CADA render (rara vez es lo que quieres)
useEffect(() => {
  console.log('Ejecuta en cada render')
})

// Array vacío [] — se ejecuta UNA sola vez, al montar el componente
useEffect(() => {
  console.log('Ejecuta solo al montar')
}, [])

// Con dependencias — se ejecuta cuando alguna dependencia cambia
useEffect(() => {
  console.log('userId cambió:', userId)
}, [userId])

// Con múltiples dependencias
useEffect(() => {
  console.log('query o page cambió')
}, [query, page])
```

> **Regla de las dependencias**: incluye en el array **todo valor del componente
> que uses dentro del efecto** — estado, props, variables derivadas.
> Si omites una dependencia que usas, el efecto trabajará con un valor desactualizado
> (closure stale). ESLint con `eslint-plugin-react-hooks` detecta estos casos automáticamente.

---

## Ciclo de vida del componente con `useEffect`

```
Componente monta
      │
      ▼
   Render
      │
      ▼
React actualiza el DOM
      │
      ▼
useEffect se ejecuta   ◄──────────────────────────────────────┐
      │                                                        │
      │  (dependencia cambia → React re-renderiza)            │
      ▼                                                        │
  Re-render                                                    │
      │                                                        │
      ▼                                                        │
React actualiza el DOM                                         │
      │                                                        │
      ▼                                                        │
Función de limpieza del efecto anterior se ejecuta ───────────┘
      │
      ▼  (componente se desmonta)
Función de limpieza final se ejecuta
      │
      ▼
Componente eliminado del DOM
```

---

## Práctica — Componentes con `useEffect`

Los ejemplos van de menor a mayor complejidad. Cada uno muestra
un patrón específico del hook.

---

### `src/components/DocumentTitle.tsx`

El `useEffect` más simple: ejecutar algo al montar y limpiar al desmontar.
Array vacío `[]` — se ejecuta una sola vez.

```tsx
// src/components/DocumentTitle.tsx

import { useEffect } from 'react'

export default function DocumentTitle() {
  useEffect(() => {
    document.title = 'Mi App con React 19'

    // Limpieza: restaurar el título al desmontar
    return () => {
      document.title = 'React App'
    }
  }, [])

  return (
    <p style={{ fontSize: 14, color: '#6b7280' }}>
      El título de la pestaña cambió al montar este componente.
    </p>
  )
}
```

### `src/App.tsx`

```tsx
// src/App.tsx

import DocumentTitle from './components/DocumentTitle'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  DocumentTitle    — useEffect con array vacío, limpia al desmontar│
// │  2  OnlineStatus     — subscripción a eventos online/offline         │
// │  3  WindowSize       — evento resize con estado objeto tipado        │
// │  4  LiveClock        — setInterval con inicializador perezoso        │
// │  5  SearchWithEffect — efecto con dependencia, búsqueda sincronizada │
// │  6  DebounceSearch   — setTimeout/clearTimeout, patrón debounce      │
// │  7  FetchUser        — fetch real, loading/error, flag cancelled      │
// │  8  AutoFocusInput   — useRef + useEffect para foco imperativo       │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

export default function App() {
  const content =
    PASO === 1 ? <DocumentTitle /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 600, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

### Prueba esto

- Monta el componente y mira la pestaña del navegador — el título cambia a "Mi App con React 19" justo después del primer render
- Cambia `'Mi App con React 19'` por tu nombre y guarda — la pestaña muestra el nuevo título inmediatamente
- Elimina la función de retorno (la limpieza) y navega a otro componente en el `App.tsx` — el título no se restaura a "React App" porque ya no existe la limpieza
- Agrega un `console.log('efecto ejecutado')` dentro del `useEffect` y un `console.log('limpieza ejecutada')` en el return — abre DevTools y observa el orden de ejecución al montar y desmontar
- Cambia `[]` por `[Math.random()]` — el efecto se ejecuta en cada render porque la dependencia siempre cambia
- Mueve la línea `document.title = '...'` fuera del `useEffect`, directamente en el cuerpo del componente — funciona igual aquí, pero nota que sin `useEffect` no puedes limpiar al desmontar

---

### `src/components/OnlineStatus.tsx`

Subscripción a eventos globales del navegador. La limpieza es crítica:
sin `removeEventListener` el listener persiste aunque el componente
ya no exista en el DOM.

```tsx
// src/components/OnlineStatus.tsx

import { useState, useEffect } from 'react'

export default function OnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine)

  useEffect(() => {
    function handleOnline()  { setIsOnline(true)  }
    function handleOffline() { setIsOnline(false) }

    window.addEventListener('online',  handleOnline)
    window.addEventListener('offline', handleOffline)

    // Limpieza — se ejecuta al desmontar
    return () => {
      window.removeEventListener('online',  handleOnline)
      window.removeEventListener('offline', handleOffline)
    }
  }, [])

  return (
    <p style={{ color: isOnline ? '#166534' : '#991b1b', fontWeight: 500 }}>
      {isOnline ? '🟢 Conectado a Internet' : '🔴 Sin conexión'}
    </p>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import OnlineStatus from './components/OnlineStatus'
```

```tsx
PASO === 2 ? <OnlineStatus /> :
```

Cambia `PASO = 2` y guarda.

### Prueba esto

- Abre DevTools → Network → Throttling y selecciona "Offline" — el texto cambia a "Sin conexión" en tiempo real sin recargar la página
- Vuelve a "Online" en DevTools — el estado regresa a "Conectado" inmediatamente gracias al event listener
- Elimina la función de retorno y desmonta el componente cambiando `PASO` en App.tsx — en DevTools verás que el listener `offline` sigue activo (Network tab → Events); los event listeners acumulados causan memory leaks
- Inicializa el estado con `useState(false)` en lugar de `useState(navigator.onLine)` — si el usuario ya está offline al montar, el estado inicial sería incorrecto hasta que ocurra el próximo evento de red
- Agrega un tercer evento: `window.addEventListener('visibilitychange', () => console.log(document.hidden))` dentro del mismo `useEffect` y observa que se limpia junto con los otros al desmontar
- Cambia `window.addEventListener` por `document.addEventListener` — funciona igual; los eventos `online`/`offline` se propagan hasta `document` también

---

### `src/components/WindowSize.tsx`

El mismo patrón de subscripción, aplicado al evento `resize`.
El estado es un objeto tipado con `interface`.

```tsx
// src/components/WindowSize.tsx

import { useState, useEffect } from 'react'

interface WindowDimensions {
  width:  number
  height: number
}

export default function WindowSize() {
  const [dimensions, setDimensions] = useState<WindowDimensions>({
    width:  window.innerWidth,
    height: window.innerHeight,
  })

  useEffect(() => {
    function handleResize() {
      setDimensions({
        width:  window.innerWidth,
        height: window.innerHeight,
      })
    }

    window.addEventListener('resize', handleResize)
    return () => window.removeEventListener('resize', handleResize)
  }, [])

  return (
    <p style={{ fontFamily: 'monospace', fontSize: 14, color: '#374151' }}>
      Ventana: {dimensions.width} × {dimensions.height} px
    </p>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import WindowSize from './components/WindowSize'
```

```tsx
PASO === 3 ? <WindowSize /> :
```

Cambia `PASO = 3` y guarda.

### Prueba esto

- Arrastra el borde de la ventana del navegador para cambiar su tamaño — los números `width` y `height` se actualizan en tiempo real con cada píxel de cambio
- Reemplaza `useState<WindowDimensions>({ width: window.innerWidth, height: window.innerHeight })` por `useState<WindowDimensions>({ width: 0, height: 0 })` — el componente muestra `0 × 0` al montar hasta el primer `resize`; observa por qué el valor inicial importa
- Agrega `console.log('resize detectado')` dentro de `handleResize` y redimensiona rápido — ves cuántos eventos dispara el navegador por segundo; es la razón por la que en producción se debouncea este handler
- Cambia `setDimensions({ width: window.innerWidth, height: window.innerHeight })` por `setDimensions((prev) => ({ ...prev, width: window.innerWidth }))` — el alto no se actualiza; muestra la diferencia entre actualización parcial y completa del estado objeto
- Añade `devicePixelRatio: window.devicePixelRatio` a la interfaz `WindowDimensions` y muéstralo — observa el valor en pantallas de alta densidad (Retina, 4K)
- Envuelve la actualización en un throttle manual con `setTimeout`: guarda el timer en un `useRef` y cancélalo si llega otro evento antes de 100ms — observa cómo reduce la cantidad de renders

---

### `src/components/LiveClock.tsx`

`setInterval` con `useEffect`. La limpieza con `clearInterval` es obligatoria —
sin ella el interval sigue corriendo en memoria aunque el componente se desmonte.

El estado se inicializa con una función `() => new Date()` para evitar llamar
`new Date()` en cada render (inicializador perezoso).

```tsx
// src/components/LiveClock.tsx

import { useState, useEffect } from 'react'

export default function LiveClock() {
  // Inicializador perezoso — new Date() se llama una sola vez
  const [time, setTime] = useState(() => new Date())

  useEffect(() => {
    const interval = setInterval(() => {
      setTime(new Date())
    }, 1000)

    // Limpieza obligatoria — detiene el interval al desmontar
    return () => clearInterval(interval)
  }, [])

  return (
    <p style={{ fontFamily: 'monospace', fontSize: 28, margin: 0, letterSpacing: 2 }}>
      {time.toLocaleTimeString('es-ES')}
    </p>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import LiveClock from './components/LiveClock'
```

```tsx
PASO === 4 ? <LiveClock /> :
```

Cambia `PASO = 4` y guarda.

### Prueba esto

- Monta el componente y observa que el reloj empieza a correr desde el primer segundo sin esperar — gracias al inicializador perezoso `() => new Date()` que lee la hora actual en el primer render
- Reemplaza `useState(() => new Date())` por `useState(new Date())` — funciona igual aquí, pero la diferencia es que `new Date()` se evaluaría en cada re-render del padre si usaras el componente dentro de uno; el inicializador perezoso solo corre una vez
- Elimina el `return () => clearInterval(interval)` y navega a otro componente — abre DevTools → Performance → Memory y observa que el interval sigue ejecutándose; después de varios montajes y desmontajes acumulas múltiples intervals corriendo simultáneamente
- Cambia `1000` por `100` — el reloj se actualiza 10 veces por segundo; observa que el UI se actualiza más seguido pero los segundos siguen mostrándose correctamente porque `toLocaleTimeString` formatea con segundo de precisión
- Reemplaza `time.toLocaleTimeString('es-ES')` por `time.toLocaleTimeString('en-US', { hour12: true })` — el formato cambia de 24h a 12h con AM/PM
- Cambia `setTime(new Date())` dentro del interval por `setTime((prev) => new Date(prev.getTime() + 1000))` — el reloj avanza exactamente 1 segundo por tick sin depender del reloj del sistema; nota que puede desincronizarse con el tiempo real si hay latencia en el tab

---

### `src/components/SearchWithEffect.tsx`

Efecto con dependencia: se re-ejecuta cada vez que `query` cambia.
Muestra cómo sincronizar estado derivado con otro estado.

```tsx
// src/components/SearchWithEffect.tsx

import { useState, useEffect } from 'react'

const MOCK_DB: Record<string, string> = {
  react:      'Biblioteca para construir interfaces de usuario.',
  typescript: 'JavaScript con tipos estáticos.',
  vite:       'Herramienta de desarrollo frontend ultrarrápida.',
  hooks:      'Funciones que permiten usar estado y efectos en componentes funcionales.',
}

export default function SearchWithEffect() {
  const [query,  setQuery]  = useState('')
  const [result, setResult] = useState<string | null>(null)

  useEffect(() => {
    const normalized = query.toLowerCase().trim()

    if (!normalized) {
      setResult(null)
      return
    }

    const found = MOCK_DB[normalized]
    setResult(found ?? 'Sin resultados para esa búsqueda.')
  }, [query])

  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: 8, maxWidth: 340 }}>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Busca: react, typescript, vite, hooks..."
        style={{
          padding: '8px 12px',
          border: '1px solid #d1d5db',
          borderRadius: 6,
          fontSize: 14,
        }}
      />
      {result && (
        <p style={{ margin: 0, fontSize: 14, color: '#374151', padding: '8px 12px', background: '#f9fafb', borderRadius: 6 }}>
          {result}
        </p>
      )}
    </div>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import SearchWithEffect from './components/SearchWithEffect'
```

```tsx
PASO === 5 ? <SearchWithEffect /> :
```

Cambia `PASO = 5` y guarda.

### Prueba esto

- Escribe "react" en el input — el resultado aparece instantáneamente porque el efecto corre sincronizado con cada cambio de `query`
- Escribe "REACT" o "React" con mayúsculas — el resultado es el mismo gracias al `.toLowerCase()` en la normalización
- Escribe "vue" — aparece "Sin resultados para esa búsqueda." porque la clave no existe en `MOCK_DB`
- Borra todo el texto del input — el resultado desaparece porque `if (!normalized)` llama `setResult(null)` y retorna antes de buscar
- Elimina la condición `if (!normalized)` y borra todo el texto — `MOCK_DB['']` devuelve `undefined`, y `undefined ?? 'Sin resultados...'` muestra el mensaje de error incluso con input vacío
- Quita `[query]` del array de dependencias (usa `[]` en su lugar) — el efecto solo corre al montar y nunca reacciona a cambios del input; es un ejemplo visual de closure stale
- Agrega una nueva entrada al `MOCK_DB`, como `jsx: 'Extensión de sintaxis de JavaScript para describir UI.'` — escribe "jsx" y verifica que aparece el resultado sin cambiar el componente

---

### `src/components/DebounceSearch.tsx`

`setTimeout` + limpieza con `clearTimeout`. Patrón clásico de debounce:
si el usuario sigue escribiendo, el timer se cancela y reinicia.
Cada vez que `input` cambia, la limpieza cancela el timer anterior.

```tsx
// src/components/DebounceSearch.tsx

import { useState, useEffect } from 'react'

export default function DebounceSearch() {
  const [input,          setInput]          = useState('')
  const [debouncedValue, setDebouncedValue] = useState('')

  useEffect(() => {
    // Se ejecuta 500ms después de que el usuario dejó de escribir
    const timer = setTimeout(() => {
      setDebouncedValue(input)
    }, 500)

    // La limpieza cancela el timer si input vuelve a cambiar
    // antes de que pasen los 500ms
    return () => clearTimeout(timer)
  }, [input])

  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: 8, maxWidth: 320 }}>
      <input
        value={input}
        onChange={(e) => setInput(e.target.value)}
        placeholder="Escribe algo..."
        style={{
          padding: '8px 12px',
          border: '1px solid #d1d5db',
          borderRadius: 6,
          fontSize: 14,
        }}
      />
      <p style={{ margin: 0, fontSize: 13, color: '#6b7280' }}>
        Valor debounced (500ms): <strong>{debouncedValue || '—'}</strong>
      </p>
      <p style={{ margin: 0, fontSize: 12, color: '#9ca3af' }}>
        Útil para evitar llamadas a API en cada pulsación de tecla.
      </p>
    </div>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import DebounceSearch from './components/DebounceSearch'
```

```tsx
PASO === 6 ? <DebounceSearch /> :
```

Cambia `PASO = 6` y guarda.

### Prueba esto

- Escribe rápido varias letras seguidas — el valor "debounced" solo se actualiza cuando paras de escribir por 500ms; cada pulsación reinicia el timer
- Cambia `500` por `2000` — ahora hay un retraso de 2 segundos; es más fácil ver cómo el valor debounced permanece detrás del valor del input
- Cambia `500` por `0` — el valor debounced sigue igual al valor del input pero con un render extra por tick; observa que aún así hay un ligero retraso porque el `setTimeout(..., 0)` pone la tarea en la cola de eventos
- Elimina el `return () => clearTimeout(timer)` y escribe rápido 5 letras seguidas — el valor debounced se actualiza 5 veces en vez de 1 porque los timers anteriores no se cancelan
- Agrega un `console.log('timer creado')` dentro del efecto y `console.log('timer cancelado')` en la limpieza — escribe "hola" y observa en DevTools que se crean y cancelan 4 timers antes de que el 5to se complete
- Reemplaza la dependencia `[input]` por `[]` — el timer se crea solo al montar; escribir en el input no tiene efecto en `debouncedValue`

---

### `src/components/FetchUser.tsx`

Fetch a una API real con estados de carga, error y datos. El patrón de
`cancelled = true` en la limpieza evita actualizar el estado de un componente
ya desmontado — uno de los errores más comunes con `useEffect` y async.

```tsx
// src/components/FetchUser.tsx

import { useState, useEffect } from 'react'

interface User {
  id:       number
  name:     string
  email:    string
  username: string
}

export default function FetchUser() {
  const [userId,  setUserId]  = useState(1)
  const [user,    setUser]    = useState<User | null>(null)
  const [loading, setLoading] = useState(false)
  const [error,   setError]   = useState<string | null>(null)

  useEffect(() => {
    // Flag de cancelación — evita race conditions y
    // actualizaciones de estado en componentes desmontados
    let cancelled = false

    async function fetchUser() {
      setLoading(true)
      setError(null)

      try {
        const res = await fetch(
          `https://jsonplaceholder.typicode.com/users/${userId}`
        )
        if (!res.ok) throw new Error(`Error HTTP ${res.status}`)

        const data: User = await res.json()

        // Solo actualiza si el componente sigue montado
        if (!cancelled) setUser(data)
      } catch (err) {
        if (!cancelled) {
          setError(err instanceof Error ? err.message : 'Error desconocido')
        }
      } finally {
        if (!cancelled) setLoading(false)
      }
    }

    fetchUser()

    return () => { cancelled = true }
  }, [userId])

  return (
    <div style={{ maxWidth: 360 }}>
      <div style={{ display: 'flex', gap: 8, marginBottom: 12 }}>
        {[1, 2, 3].map((id) => (
          <button
            key={id}
            onClick={() => setUserId(id)}
            style={{
              padding: '6px 14px',
              borderRadius: 6,
              border: '1px solid #d1d5db',
              background: userId === id ? '#0070f3' : '#fff',
              color:      userId === id ? '#fff'    : '#333',
              cursor: 'pointer',
              fontWeight: userId === id ? 600 : 400,
            }}
          >
            Usuario {id}
          </button>
        ))}
      </div>

      {loading && (
        <p style={{ color: '#6b7280', fontSize: 14 }}>Cargando...</p>
      )}
      {error && (
        <p style={{ color: '#991b1b', fontSize: 14 }}>Error: {error}</p>
      )}
      {user && !loading && (
        <div style={{ padding: 14, border: '1px solid #e5e7eb', borderRadius: 8 }}>
          <p style={{ margin: '0 0 4px', fontWeight: 600 }}>{user.name}</p>
          <p style={{ margin: '0 0 4px', fontSize: 13, color: '#6b7280' }}>
            @{user.username}
          </p>
          <p style={{ margin: 0, fontSize: 13, color: '#6b7280' }}>
            {user.email}
          </p>
        </div>
      )}
    </div>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import FetchUser from './components/FetchUser'
```

```tsx
PASO === 7 ? <FetchUser /> :
```

Cambia `PASO = 7` y guarda.

### Prueba esto

- Haz clic en "Usuario 1", luego "Usuario 2" rápidamente antes de que cargue el 1 — el estado de carga aparece brevemente y el resultado final muestra al usuario 2, no al 1; el flag `cancelled = true` descarta la respuesta más lenta
- Cambia `[1, 2, 3]` por `[1, 2, 3, 4, 5]` para agregar más botones — la API de JSONPlaceholder tiene 10 usuarios; prueba hasta el id 10
- Elimina el `return () => { cancelled = true }` y haz clics rápidos entre usuarios — en DevTools → Network ves que todas las respuestas intentan actualizar el estado; con el flag, solo la última request que completó tiene efecto
- Cambia `useState(1)` por `useState(999)` — la API devuelve un 404 para ids que no existen; el bloque `catch` captura el error `'Error HTTP 404'` y lo muestra en la UI
- Añade `phone: string` a la interfaz `User` y muéstralo en la tarjeta — JSON Placeholder incluye el campo `phone` en cada usuario
- Reemplaza la condición `if (!res.ok) throw new Error(...)` por comentarla — con un id inválido la app mostraría datos `undefined` sin mensaje de error; observa por qué la verificación de `res.ok` es importante

---

### `src/components/AutoFocusInput.tsx`

Introduce `useRef` junto a `useEffect`. `useRef` crea una referencia al nodo
del DOM real — sin `useRef` no hay forma de llamar `.focus()` desde React.

```tsx
// src/components/AutoFocusInput.tsx

import { useEffect, useRef } from 'react'

export default function AutoFocusInput() {
  // useRef<HTMLInputElement>(null) — la referencia empieza en null
  // y se asigna automáticamente cuando React monta el <input>
  const inputRef = useRef<HTMLInputElement>(null)

  useEffect(() => {
    // inputRef.current puede ser null si el componente se desmontó
    // El operador ?. lo maneja de forma segura
    inputRef.current?.focus()
  }, [])

  return (
    <input
      ref={inputRef}
      placeholder="Este input recibe foco automáticamente al montar"
      style={{
        padding: '8px 12px',
        border: '1px solid #d1d5db',
        borderRadius: 6,
        width: '100%',
        fontSize: 14,
      }}
    />
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import AutoFocusInput from './components/AutoFocusInput'
```

```tsx
PASO === 8 ? <AutoFocusInput /> :
```

Cambia `PASO = 8` y guarda. Desde aquí puedes volver a cualquier paso anterior cambiando la constante.

### Prueba esto

- Monta el componente — el cursor aparece en el input sin que el usuario haga clic; es el comportamiento esperado del `.focus()` ejecutado en el `useEffect`
- Elimina el `?.` del `inputRef.current?.focus()` — TypeScript muestra un error porque `current` es `HTMLInputElement | null`; el operador `?.` es necesario para satisfacer el sistema de tipos
- Elimina el `ref={inputRef}` del `<input>` y guarda — el `useEffect` corre pero `inputRef.current` sigue siendo `null` porque React nunca asignó el nodo; no hay error en tiempo de ejecución gracias al `?.`, pero el foco no ocurre
- Cambia `useRef<HTMLInputElement>(null)` por `useRef<HTMLTextAreaElement>(null)` y el `<input>` por `<textarea>` — el auto-foco funciona igual con cualquier elemento focusable
- Agrega un segundo `useRef<HTMLInputElement>(null)` llamado `buttonRef` y asócialo a un `<button>` — en el `useEffect` llama `.focus()` en el botón en vez del input; observa que el botón queda con el foco activo y se puede presionar con Enter
- Reemplaza `useEffect(() => { inputRef.current?.focus() }, [])` por llamar `inputRef.current?.focus()` directamente en el cuerpo del componente (sin `useEffect`) — obtendrás un error porque `inputRef.current` todavía es `null` en el momento del render, antes de que React monte el DOM

---

## Errores comunes con `useEffect`

### 1. Olvidar limpiar timers y listeners

```tsx
// ❌ El interval sigue corriendo aunque el componente se desmonte
useEffect(() => {
  setInterval(() => setTime(new Date()), 1000)
}, [])

// ✅ Siempre retorna la limpieza
useEffect(() => {
  const interval = setInterval(() => setTime(new Date()), 1000)
  return () => clearInterval(interval)
}, [])
```

### 2. Olvidar dependencias (closure stale)

```tsx
// ❌ El efecto siempre ve el valor inicial de count (closure stale)
useEffect(() => {
  const id = setInterval(() => {
    console.log(count) // siempre imprime 0
  }, 1000)
  return () => clearInterval(id)
}, []) // falta count en dependencias

// ✅ Con count en dependencias — reinicia el interval cuando count cambia
useEffect(() => {
  const id = setInterval(() => {
    console.log(count)
  }, 1000)
  return () => clearInterval(id)
}, [count])
```

### 3. Función async directamente en el callback

```tsx
// ❌ useEffect no acepta una función async como callback
// porque async retorna una Promise, no una función de limpieza
useEffect(async () => {
  const data = await fetch('/api/data')
}, [])

// ✅ Define la función async adentro y llámala
useEffect(() => {
  async function loadData() {
    const data = await fetch('/api/data')
  }
  loadData()
}, [])
```

### 4. Race condition sin flag de cancelación

```tsx
// ❌ Si userId cambia rápido, respuestas anteriores pueden
// sobreescribir la respuesta más reciente
useEffect(() => {
  fetch(`/api/users/${userId}`)
    .then((r) => r.json())
    .then((data) => setUser(data)) // puede llegar tarde y pisar datos nuevos
}, [userId])

// ✅ Flag de cancelación — solo aplica la última respuesta
useEffect(() => {
  let cancelled = false
  fetch(`/api/users/${userId}`)
    .then((r) => r.json())
    .then((data) => { if (!cancelled) setUser(data) })
  return () => { cancelled = true }
}, [userId])
```

---

## Resumen de la página 5

- `useEffect` sincroniza el componente con el mundo exterior después de que React actualiza el DOM.
- El **array de dependencias** controla cuándo se re-ejecuta: `[]` = solo al montar, `[dep]` = cuando `dep` cambia, sin array = en cada render.
- La **función de limpieza** es obligatoria cuando el efecto abre recursos: event listeners, timers, conexiones, subscripciones.
- `useEffect` no acepta callbacks `async` — define la función async adentro y llámala.
- El flag `cancelled = true` en la limpieza evita race conditions y actualizaciones de estado en componentes ya desmontados.
- `useRef<T>(null)` permite acceder al nodo del DOM real — se combina con `useEffect` para operaciones imperativas como `.focus()`.

---

> **Siguiente página →** Hooks personalizados: extraer lógica repetida en hooks reutilizables,
> tipado de hooks con genéricos y convención `use*`.