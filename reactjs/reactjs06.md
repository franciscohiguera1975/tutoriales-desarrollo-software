# Curso React.js + TypeScript — Página 6
## Módulo 3 · Hooks nativos
### `useRef`, `useCallback` y `useMemo`

---

## Los tres hooks de esta página

| Hook | Para qué sirve |
|---|---|
| `useRef` | Referenciar nodos del DOM o guardar valores mutables sin disparar re-renders |
| `useCallback` | Memorizar funciones para que no se recreen en cada render |
| `useMemo` | Memorizar resultados de cálculos costosos |

Los tres comparten un propósito común: **controlar cuándo React hace trabajo innecesario**.
`useCallback` y `useMemo` son herramientas de optimización — no se usan por defecto en
todo el código, sino en situaciones específicas donde el costo de recalcular es real.

---

## `useRef`

`useRef` crea un objeto `{ current: T }` que persiste durante toda la vida del componente.
A diferencia de `useState`, **mutar `ref.current` no dispara un re-render**.

Tiene dos usos completamente distintos:

```
useRef
  ├── Referenciar nodos del DOM  →  ref.current = <input>, <canvas>, etc.
  └── Guardar valores mutables   →  ref.current = timer, contador, valor previo, etc.
```

### Uso 1 — Acceder a un nodo del DOM

```tsx
import { useRef, useEffect } from 'react'

// El tipo genérico <HTMLInputElement> indica qué nodo referencia
const inputRef = useRef<HTMLInputElement>(null)

useEffect(() => {
  inputRef.current?.focus()  // ?. porque current puede ser null antes del mount
}, [])

return <input ref={inputRef} />
//                ^^^ React asigna el nodo aquí después del primer render
```

El flujo es:
1. `useRef<HTMLInputElement>(null)` — la ref empieza en `null`.
2. React monta el `<input>` en el DOM.
3. React asigna el nodo real a `ref.current`.
4. `useEffect` corre — `ref.current` ya no es `null`.

---

### `src/components/AutoFocusForm.tsx`

```tsx
// src/components/AutoFocusForm.tsx

import { useRef, useEffect } from 'react'

export default function AutoFocusForm() {
  const nameRef  = useRef<HTMLInputElement>(null)
  const emailRef = useRef<HTMLInputElement>(null)

  // Foco en el primer campo al montar
  useEffect(() => {
    nameRef.current?.focus()
  }, [])

  function handleNameKeyDown(e: React.KeyboardEvent<HTMLInputElement>) {
    // Avanza al siguiente campo con Enter
    if (e.key === 'Enter') {
      e.preventDefault()
      emailRef.current?.focus()
    }
  }

  return (
    <form style={{ display: 'flex', flexDirection: 'column', gap: 10, maxWidth: 300 }}>
      <input
        ref={nameRef}
        placeholder="Nombre"
        onKeyDown={handleNameKeyDown}
        style={{ padding: '8px 12px', border: '1px solid #d1d5db', borderRadius: 6 }}
      />
      <input
        ref={emailRef}
        type="email"
        placeholder="Email"
        style={{ padding: '8px 12px', border: '1px solid #d1d5db', borderRadius: 6 }}
      />
      <button
        type="submit"
        style={{ padding: '8px', background: '#0070f3', color: '#fff', border: 'none', borderRadius: 6, cursor: 'pointer' }}
      >
        Enviar
      </button>
    </form>
  )
}
```

### Prueba esto

- Monta el componente — el cursor aparece automáticamente en el campo "Nombre" sin hacer clic; es el `useEffect` con `nameRef.current?.focus()` ejecutándose tras el primer render
- Escribe algo en "Nombre" y pulsa Enter — el foco salta al campo "Email" gracias a `emailRef.current?.focus()` dentro del `handleNameKeyDown`
- Pulsa Enter en el campo "Email" — no ocurre nada especial; `handleNameKeyDown` solo está registrado en el primer input; podrías agregar un `onKeyDown` al email que haga `.focus()` en el botón
- Agrega un tercer `useRef<HTMLInputElement>(null)` para un campo "Teléfono" y encadena los Enter: Nombre → Email → Teléfono — comprueba que el foco avanza secuencialmente
- Elimina el `ref={nameRef}` del primer `<input>` y guarda — el foco ya no funciona; `nameRef.current` sigue siendo `null` porque React no asignó ningún nodo a esa ref
- Reemplaza `emailRef.current?.focus()` por `emailRef.current?.select()` si el campo de email ya tiene texto escrito — el texto queda seleccionado en vez de solo recibir el foco

### `src/App.tsx`

```tsx
// src/App.tsx

import AutoFocusForm from './components/AutoFocusForm'
// ┌──────────────────────────────────────────────────────────────────────┐
// │  1  AutoFocusForm    — useRef: foco automático y tecla Enter        │
// │  2  Stopwatch        — useRef: interval sin re-renders extra        │
// │  3  FilterableList   — useCallback: función estable para hijo       │
// │  4  ProductAnalytics — useMemo: stats y filtro memoizados           │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

export default function App() {
  const content = PASO === 1 ? <AutoFocusForm /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 500, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

---

### Uso 2 — Valor mutable sin re-render

`useRef` también sirve para guardar cualquier valor que necesite persistir entre renders
pero cuyo cambio no deba causar un nuevo render:

```tsx
import { useRef } from 'react'

// Contar renders sin causar renders adicionales
const renderCount = useRef(0)
renderCount.current += 1  // muta directamente — no dispara re-render

// Guardar el ID de un timer
const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null)
timerRef.current = setTimeout(() => {}, 1000)
clearTimeout(timerRef.current ?? undefined)

// Guardar el valor anterior de un estado
const prevValue = useRef<string>('')
// ... en el render:
// prevValue.current = currentValue  (al final)
```

---

### `src/components/Stopwatch.tsx`

Ejemplo clásico: el `intervalRef` guarda el ID del interval sin forzar re-renders.
Si usáramos `useState` para el ID, cada `setInterval`/`clearInterval` dispararía
un render innecesario.

```tsx
// src/components/Stopwatch.tsx

import { useState, useRef } from 'react'

export default function Stopwatch() {
  const [time,    setTime]    = useState(0)
  const [running, setRunning] = useState(false)

  // ReturnType<typeof setInterval> es el tipo correcto para el ID
  // del interval — funciona igual en browser y Node.js
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null)

  function handleStart() {
    if (running) return
    setRunning(true)
    intervalRef.current = setInterval(() => {
      setTime((prev) => prev + 1)
    }, 1000)
  }

  function handleStop() {
    if (intervalRef.current) {
      clearInterval(intervalRef.current)
      intervalRef.current = null
    }
    setRunning(false)
  }

  function handleReset() {
    handleStop()
    setTime(0)
  }

  const minutes = Math.floor(time / 60).toString().padStart(2, '0')
  const seconds = (time % 60).toString().padStart(2, '0')

  return (
    <div style={{ display: 'flex', flexDirection: 'column', alignItems: 'center', gap: 12 }}>
      <p style={{ fontFamily: 'monospace', fontSize: 36, margin: 0, letterSpacing: 4 }}>
        {minutes}:{seconds}
      </p>
      <div style={{ display: 'flex', gap: 8 }}>
        <button
          onClick={handleStart}
          disabled={running}
          style={btnStyle('#22c55e')}
        >
          Iniciar
        </button>
        <button
          onClick={handleStop}
          disabled={!running}
          style={btnStyle('#f59e0b')}
        >
          Pausar
        </button>
        <button
          onClick={handleReset}
          style={btnStyle('#6b7280')}
        >
          Reset
        </button>
      </div>
    </div>
  )
}

function btnStyle(bg: string) {
  return {
    padding: '8px 16px',
    background: bg,
    color: '#fff',
    border: 'none',
    borderRadius: 6,
    cursor: 'pointer',
    fontWeight: 500,
  }
}
```

### Prueba esto

- Pulsa "Iniciar" y luego "Pausar" varias veces — el tiempo se detiene y continúa desde donde quedó; el ID del interval se guarda en `intervalRef.current` sin causar re-renders adicionales
- Pulsa "Iniciar" dos veces seguidas — el segundo clic no hace nada gracias a `if (running) return`; sin esa guarda se crearían dos intervals corriendo simultáneamente, haciendo que el tiempo avance al doble de velocidad
- Reemplaza `const intervalRef = useRef<...>(null)` por `const [intervalId, setIntervalId] = useState<...>(null)` y actualiza los usos — el cronómetro funciona igual pero cada `setInterval`/`clearInterval` genera un render adicional innecesario; confirma en DevTools → React Profiler
- Pulsa "Iniciar", deja correr hasta 65 segundos — el display cambia de `01:04` a `01:05`; verifica que el cálculo con `Math.floor(time / 60)` y `time % 60` funciona correctamente para minutos
- Cambia `setTime((prev) => prev + 1)` por `setTime(time + 1)` — el cronómetro puede estancarse o comportarse incorrectamente porque `time` en el closure siempre tiene el valor del render donde se creó el interval; el updater funcional `(prev) => prev + 1` siempre opera sobre el valor más reciente
- Agrega un botón "Vuelta" que guarde el tiempo actual en un array de `useState<number[]>([])` y muestre la lista de vueltas debajo — el array puede actualizarse con `setLaps(prev => [...prev, time])`

### Agrega a `src/App.tsx`

```tsx
import Stopwatch from './components/Stopwatch'
```

```tsx
PASO === 2 ? <Stopwatch /> :
```

Cambia `PASO = 2` y guarda.

---

### `useRef` vs `useState` — cuándo usar cada uno

| Necesito... | Usar |
|---|---|
| Acceder a un nodo del DOM | `useRef` |
| Guardar un valor que NO debe causar re-render al cambiar | `useRef` |
| Guardar el ID de un timer o interval | `useRef` |
| Guardar un valor que SÍ debe causar re-render al cambiar | `useState` |
| Mostrar el valor en el JSX | `useState` — `ref.current` no actualiza la UI |

> Si muestras `ref.current` en el JSX y lo mutas, la UI no se actualizará.
> `useRef` es para valores que React no necesita observar.

---

## `useCallback`

`useCallback` memoriza una función y solo la recrea cuando sus dependencias cambian.

```tsx
import { useCallback } from 'react'

// Sin useCallback — nueva función en cada render
function handleDelete(id: number) { ... }

// Con useCallback — misma referencia mientras las dependencias no cambien
const handleDelete = useCallback((id: number) => { ... }, [/* dependencias */])
```

### ¿Por qué importa que sea la misma referencia?

En JavaScript dos funciones nunca son iguales aunque hagan lo mismo:

```ts
const a = () => {}
const b = () => {}
a === b  // false — referencias distintas
```

React usa comparación por referencia para decidir si un prop cambió.
Si pasas una función como prop a un componente hijo y esa función se recrea
en cada render del padre, el hijo recibe un prop "nuevo" en cada render
aunque la lógica sea idéntica — causando re-renders innecesarios.

---

### `src/components/FilterableList.tsx`

```tsx
// src/components/FilterableList.tsx

import { useState, useCallback } from 'react'

interface Item {
  id:       number
  name:     string
  category: string
}

interface ItemRowProps {
  item:     Item
  onDelete: (id: number) => void
}

// Componente hijo — recibe onDelete como prop
function ItemRow({ item, onDelete }: ItemRowProps) {
  return (
    <div style={{
      display: 'flex', justifyContent: 'space-between',
      alignItems: 'center', padding: '8px 0',
      borderBottom: '1px solid #e5e7eb',
    }}>
      <span>
        <strong style={{ fontSize: 14 }}>{item.name}</strong>
        <span style={{ marginLeft: 8, fontSize: 12, color: '#9ca3af' }}>
          {item.category}
        </span>
      </span>
      <button
        onClick={() => onDelete(item.id)}
        style={{ background: 'none', border: 'none', cursor: 'pointer', color: '#ef4444', fontSize: 16 }}
      >
        ✕
      </button>
    </div>
  )
}

const INITIAL_ITEMS: Item[] = [
  { id: 1, name: 'Teclado mecánico',   category: 'Periféricos' },
  { id: 2, name: 'Monitor ultrawide',  category: 'Pantallas'   },
  { id: 3, name: 'Mouse ergonómico',   category: 'Periféricos' },
  { id: 4, name: 'Webcam 4K',          category: 'Cámaras'     },
]

export default function FilterableList() {
  const [items,  setItems]  = useState<Item[]>(INITIAL_ITEMS)
  const [filter, setFilter] = useState('')

  // useCallback — handleDelete mantiene la misma referencia
  // mientras items no cambie, evitando re-renders de ItemRow
  const handleDelete = useCallback((id: number) => {
    setItems((prev) => prev.filter((item) => item.id !== id))
  }, [])

  const filteredItems = items.filter((item) =>
    item.name.toLowerCase().includes(filter.toLowerCase()) ||
    item.category.toLowerCase().includes(filter.toLowerCase())
  )

  return (
    <div style={{ maxWidth: 400 }}>
      <input
        value={filter}
        onChange={(e) => setFilter(e.target.value)}
        placeholder="Filtrar por nombre o categoría..."
        style={{
          width: '100%', padding: '8px 12px',
          border: '1px solid #d1d5db', borderRadius: 6,
          marginBottom: 12, boxSizing: 'border-box',
        }}
      />

      {filteredItems.length === 0
        ? <p style={{ color: '#9ca3af', fontSize: 14 }}>Sin resultados.</p>
        : filteredItems.map((item) => (
            <ItemRow
              key={item.id}
              item={item}
              onDelete={handleDelete}
            />
          ))
      }
    </div>
  )
}
```

### Prueba esto

- Escribe "Peri" en el filtro — solo aparecen "Teclado mecánico" y "Mouse ergonómico" porque ambos pertenecen a la categoría "Periféricos"; el filtro busca tanto en nombre como en categoría
- Haz clic en ✕ de un ítem — desaparece de la lista; el `handleDelete` usa el updater funcional `prev.filter(...)` para no depender del closure de `items`
- Quita el array `[]` del `useCallback` y agrega `[items]` como dependencia — `handleDelete` ahora se recrea cada vez que cambia `items`; si `ItemRow` usara `React.memo`, los hijos se re-renderizarían con cada eliminación
- Reemplaza `useCallback` con una función normal `function handleDelete(id: number) { ... }` — la funcionalidad es idéntica; la diferencia solo sería observable con `React.memo` en `ItemRow` y miles de filas
- Escribe "cam" en el filtro — aparecen "Webcam 4K" (por nombre) y ningún otro; borra el filtro y todos los ítems vuelven porque el filtrado es solo visual, no elimina del estado
- Agrega un campo `price: number` a la interfaz `Item` y a `INITIAL_ITEMS`, y muestra el precio en `ItemRow` con `$${item.price}` — observa que TypeScript exige que todos los objetos del array incluyan el nuevo campo

### Agrega a `src/App.tsx`

```tsx
import FilterableList from './components/FilterableList'
```

```tsx
PASO === 3 ? <FilterableList /> :
```

Cambia `PASO = 3` y guarda.

---

## `useMemo`

`useMemo` memoriza el **resultado** de un cálculo. Solo recalcula cuando
sus dependencias cambian.

```tsx
import { useMemo } from 'react'

// Sin useMemo — recalcula en cada render aunque products no haya cambiado
const total = products.reduce((acc, p) => acc + p.price, 0)

// Con useMemo — solo recalcula cuando products cambia
const total = useMemo(
  () => products.reduce((acc, p) => acc + p.price, 0),
  [products]
)
```

### `src/components/ProductAnalytics.tsx`

Dos `useMemo` encadenados: el primero calcula estadísticas del catálogo completo,
el segundo filtra sin recalcular las estadísticas.

```tsx
// src/components/ProductAnalytics.tsx

import { useState, useMemo } from 'react'

interface Product {
  id:       number
  name:     string
  price:    number
  category: string
  stock:    number
}

const CATALOG: Product[] = [
  { id: 1, name: 'Laptop Pro',       price: 1299, category: 'Computadoras', stock: 5  },
  { id: 2, name: 'Teclado mecánico', price: 89,   category: 'Periféricos',  stock: 20 },
  { id: 3, name: 'Monitor 27"',      price: 349,  category: 'Pantallas',    stock: 8  },
  { id: 4, name: 'Mouse inalámbrico',price: 29,   category: 'Periféricos',  stock: 35 },
  { id: 5, name: 'Webcam HD',        price: 59,   category: 'Cámaras',      stock: 12 },
  { id: 6, name: 'Auriculares BT',   price: 149,  category: 'Audio',        stock: 0  },
]

export default function ProductAnalytics() {
  const [search,      setSearch]      = useState('')
  const [onlyInStock, setOnlyInStock] = useState(false)

  // Recalcula solo cuando CATALOG cambia (nunca, es constante)
  // En un caso real cambiaría cuando lleguen datos del servidor
  const stats = useMemo(() => {
    const total    = CATALOG.reduce((acc, p) => acc + p.price, 0)
    const average  = total / CATALOG.length
    const inStock  = CATALOG.filter((p) => p.stock > 0).length
    return { total, average, inStock, count: CATALOG.length }
  }, [])

  // Recalcula cuando search u onlyInStock cambian — no cuando cambia el estado del otro
  const filtered = useMemo(() => {
    return CATALOG
      .filter((p) => p.name.toLowerCase().includes(search.toLowerCase()))
      .filter((p) => !onlyInStock || p.stock > 0)
  }, [search, onlyInStock])

  return (
    <div style={{ maxWidth: 440 }}>

      {/* Stats — calculadas una sola vez */}
      <div style={{
        display: 'grid', gridTemplateColumns: 'repeat(4, 1fr)',
        gap: 8, marginBottom: 16,
      }}>
        {[
          { label: 'Productos', value: stats.count },
          { label: 'En stock',  value: stats.inStock },
          { label: 'Promedio',  value: `$${stats.average.toFixed(0)}` },
          { label: 'Total inv.',value: `$${stats.total.toFixed(0)}` },
        ].map((s) => (
          <div key={s.label} style={{
            padding: '10px 8px', background: '#f9fafb',
            borderRadius: 8, textAlign: 'center',
          }}>
            <p style={{ margin: 0, fontSize: 18, fontWeight: 700 }}>{s.value}</p>
            <p style={{ margin: 0, fontSize: 11, color: '#9ca3af' }}>{s.label}</p>
          </div>
        ))}
      </div>

      {/* Controles */}
      <div style={{ display: 'flex', gap: 8, marginBottom: 12 }}>
        <input
          value={search}
          onChange={(e) => setSearch(e.target.value)}
          placeholder="Buscar producto..."
          style={{ flex: 1, padding: '8px 12px', border: '1px solid #d1d5db', borderRadius: 6 }}
        />
        <label style={{ display: 'flex', alignItems: 'center', gap: 6, fontSize: 13, cursor: 'pointer' }}>
          <input
            type="checkbox"
            checked={onlyInStock}
            onChange={(e) => setOnlyInStock(e.target.checked)}
          />
          Solo en stock
        </label>
      </div>

      {/* Lista filtrada */}
      <ul style={{ listStyle: 'none', padding: 0 }}>
        {filtered.map((product) => (
          <li key={product.id} style={{
            display: 'flex', justifyContent: 'space-between',
            alignItems: 'center', padding: '8px 0',
            borderBottom: '1px solid #e5e7eb',
            opacity: product.stock === 0 ? 0.4 : 1,
          }}>
            <div>
              <span style={{ fontSize: 14, fontWeight: 500 }}>{product.name}</span>
              <span style={{ marginLeft: 8, fontSize: 12, color: '#9ca3af' }}>
                {product.category}
              </span>
            </div>
            <div style={{ textAlign: 'right' }}>
              <p style={{ margin: 0, fontWeight: 600 }}>${product.price}</p>
              <p style={{ margin: 0, fontSize: 11, color: product.stock > 0 ? '#22c55e' : '#ef4444' }}>
                {product.stock > 0 ? `${product.stock} uds.` : 'Sin stock'}
              </p>
            </div>
          </li>
        ))}
      </ul>

      {filtered.length === 0 && (
        <p style={{ color: '#9ca3af', fontSize: 14 }}>Sin resultados.</p>
      )}
    </div>
  )
}
```

### Prueba esto

- Escribe "tec" en el buscador — solo aparece "Teclado mecánico"; cambia a "a" y aparecen todos los que tienen "a" en el nombre; el `useMemo` de `filtered` recalcula solo cuando `search` u `onlyInStock` cambian
- Activa el checkbox "Solo en stock" — "Auriculares BT" desaparece porque su `stock` es 0; el `useMemo` de `stats` no se recalcula porque sus dependencias son solo `[]`
- Agrega un `console.log('recalculando stats')` dentro del primer `useMemo` y escribe en el buscador — el log NO aparece porque `stats` tiene dependencia `[]` y no se recalcula cuando cambia `search`
- Agrega un `console.log('recalculando filtered')` dentro del segundo `useMemo` y alterna el checkbox — el log aparece cada vez que cambia `onlyInStock` o `search`, confirmando que las dependencias están correctas
- Reemplaza el primer `useMemo` con `[]` por una constante fuera del componente: `const STATS = { total: ..., ... }` calculada una vez al importar el módulo — para datos que nunca cambian es incluso mejor que `useMemo`, que tiene un pequeño overhead de comparación
- Agrega un tercer `useMemo` que calcule el producto más caro del resultado filtrado: `const mostExpensive = useMemo(() => filtered.reduce((a, b) => a.price > b.price ? a : b, filtered[0]), [filtered])` — muestra su nombre encima de la lista; observa que depende de `filtered`, que ya es un valor memorizado
- Desde aquí puedes volver a cualquier PASO anterior cambiando el número y guardando.

### Agrega a `src/App.tsx`

```tsx
import ProductAnalytics from './components/ProductAnalytics'
```

```tsx
PASO === 4 ? <ProductAnalytics /> :
```

Cambia `PASO = 4` y guarda.

---

## `useCallback` vs `useMemo` — la diferencia exacta

```tsx
import { useCallback, useMemo } from 'react'

// useCallback — memoriza la FUNCIÓN
const handleClick = useCallback(() => {
  doSomething(value)
}, [value])
// equivale a:
const handleClick = useMemo(() => () => {
  doSomething(value)
}, [value])

// useMemo — memoriza el RESULTADO de ejecutar la función
const total = useMemo(() => {
  return items.reduce((acc, item) => acc + item.price, 0)
}, [items])
```

`useCallback(fn, deps)` es exactamente `useMemo(() => fn, deps)`.
Existen los dos porque son semánticamente distintos y tener los dos nombres
hace el código más legible.

---

## Cuándo NO usar `useCallback` y `useMemo`

La memorización tiene un costo: React necesita guardar el valor anterior
y comparar las dependencias en cada render. Si el cálculo es barato,
memorizar puede ser más lento que recalcular.

```tsx
// ❌ No vale la pena — suma de dos números es instantánea
const sum = useMemo(() => a + b, [a, b])

// ❌ No vale la pena — componente que se re-renderiza igual de seguido
const handleClick = useCallback(() => setCount(c => c + 1), [])

// ✅ Vale la pena — cálculo sobre arrays grandes
const stats = useMemo(() => computeHeavyStats(dataset), [dataset])

// ✅ Vale la pena — función pasada a un hijo que usa React.memo
const handleDelete = useCallback((id: number) => {
  setItems(prev => prev.filter(i => i.id !== id))
}, [])
```

> Regla práctica: **primero escribe sin `useCallback`/`useMemo`**.
> Añádelos solo cuando puedas medir un problema real de rendimiento
> o cuando pases funciones a componentes hijos optimizados con `React.memo`.

---

## Ejercicios propuestos

1. **`ScrollTracker`** — usa `useRef` para referenciar un `<div>` con scroll
   y muestra la posición actual con un event listener en el elemento.

2. **`PreviousValue`** — crea un componente que muestre el valor actual de un input
   y el valor anterior (antes del último cambio) usando `useRef` para guardar el previo.

3. **`SortableTable`** — recibe un array de objetos, permite ordenar por columna.
   Usa `useMemo` para el array ordenado y `useCallback` para los handlers de clic
   en encabezados.

4. **`WordFrequency`** — recibe un texto largo como prop y usa `useMemo` para calcular
   las 10 palabras más frecuentes. Debe recalcular solo cuando el texto cambia,
   no cuando cambia un filtro de visualización separado.

---

## Resumen de la página 6

- `useRef` crea un objeto `{ current }` que persiste sin causar re-renders al mutar.
  Tiene dos usos: referenciar nodos del DOM y guardar valores mutables internos.
- `useRef<HTMLInputElement>(null)` — el tipo genérico es el elemento HTML que referenciará.
- `ReturnType<typeof setInterval>` es el tipo correcto para IDs de timers en TypeScript.
- `useCallback` memoriza funciones — útil cuando se pasan como props a hijos optimizados.
- `useMemo` memoriza resultados de cálculos — útil para transformaciones costosas de datos.
- `useCallback(fn, deps)` es equivalente a `useMemo(() => fn, deps)`.
- No memorizar por defecto — añadir `useCallback`/`useMemo` solo cuando haya un problema
  de rendimiento medible o al pasar funciones a componentes con `React.memo`.

---

> **Siguiente página →** `useReducer`: gestión de estado complejo
> con acciones tipadas — alternativa a `useState` cuando el estado
> tiene múltiples transiciones relacionadas.