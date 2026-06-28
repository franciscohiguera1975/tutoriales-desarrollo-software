# Curso React.js + TypeScript — Página 9
## Módulo 3 · Hooks nativos
### `useMemo` — memoización de valores calculados

---

## ¿Qué es `useMemo`?

`useMemo` **memoriza el resultado de una función** y solo lo recalcula cuando alguna
de sus dependencias cambia. Sin él, la función se ejecuta en cada render aunque
las entradas no hayan cambiado.

```tsx
const value = useMemo(() => expensiveCalculation(a, b), [a, b])
//                                                       └── solo recalcula si a o b cambian
```

```tsx
// ❌ Sin useMemo — se ejecuta en cada render
const filtered = products.filter(p => p.active)   // recorre el array SIEMPRE

// ✅ Con useMemo — solo cuando products cambia
const filtered = useMemo(
  () => products.filter(p => p.active),
  [products]
)
```

---

## Cuándo usar `useMemo`

| Situación | Usar `useMemo` |
|---|---|
| Filtrar o ordenar una lista larga (> 100 items) | ✅ Sí |
| Calcular estadísticas derivadas de un array | ✅ Sí |
| Construir un objeto/array que se pasa a un hijo memoizado | ✅ Sí |
| Evitar que un valor costoso se recalcule por renders no relacionados | ✅ Sí |
| Suma simple `price * qty` | ❌ No — más costo que beneficio |

> **Regla práctica**: mide primero, memoiza después.
> `useMemo` tiene su propio costo (comparación de dependencias).

---

## Dependencias de `useMemo`

```tsx
// ✅ Dependencias completas — sin eslint-disable necesario
const result = useMemo(() => fn(a, b), [a, b])

// ❌ Dependencia olvidada — result no se actualiza cuando b cambia
const result = useMemo(() => fn(a, b), [a])

// ❌ Sin dependencias — se ejecuta solo en el primer render
const result = useMemo(() => fn(a), [])
```

---

## PASO 1 — `src/components/PrimeSieve.tsx`

Muestra el costo real de un cálculo sin y con `useMemo`.
Un contador independiente provoca re-renders — sin memo el cribado de primos
corre en cada pulsación; con memo solo corre cuando `limit` cambia.

```tsx
// src/components/PrimeSieve.tsx

import { useState, useMemo } from 'react'

// Criba de Eratóstenes — complejidad O(n log log n)
function sieve(n: number): number[] {
  if (n < 2) return []
  const isPrime = new Array(n + 1).fill(true)
  isPrime[0] = isPrime[1] = false
  for (let i = 2; i * i <= n; i++) {
    if (isPrime[i]) {
      for (let j = i * i; j <= n; j += i) isPrime[j] = false
    }
  }
  return isPrime.reduce<number[]>((acc, ok, i) => (ok ? [...acc, i] : acc), [])
}

export default function PrimeSieve() {
  const [limit,   setLimit]   = useState(10_000)
  const [counter, setCounter] = useState(0)

  // useMemo: el cribado solo corre cuando `limit` cambia
  const primes = useMemo(() => sieve(limit), [limit])

  return (
    <div style={{ fontFamily: 'sans-serif', maxWidth: 520, margin: '0 auto', padding: 24 }}>
      <h2 style={{ fontSize: 18, fontWeight: 700, marginBottom: 4 }}>PrimeSieve</h2>
      <p style={{ color: '#666', fontSize: 14, marginBottom: 20 }}>
        El contador provoca re-renders — el cribado solo recorre cuando cambia el límite.
      </p>

      <div style={{ display: 'flex', gap: 12, flexWrap: 'wrap', marginBottom: 20 }}>
        <label style={{ display: 'flex', flexDirection: 'column', gap: 4, fontSize: 14 }}>
          Límite (N)
          <input
            type="range"
            min={1000}
            max={100_000}
            step={1000}
            value={limit}
            onChange={e => setLimit(Number(e.target.value))}
            style={{ width: 200 }}
          />
          <span>{limit.toLocaleString()}</span>
        </label>

        <div style={{ display: 'flex', flexDirection: 'column', gap: 4, fontSize: 14 }}>
          Counter (trigger re-renders)
          <div style={{ display: 'flex', gap: 8, alignItems: 'center' }}>
            <button
              onClick={() => setCounter(c => c - 1)}
              style={{ padding: '4px 12px', cursor: 'pointer' }}
            >−</button>
            <span style={{ minWidth: 32, textAlign: 'center' }}>{counter}</span>
            <button
              onClick={() => setCounter(c => c + 1)}
              style={{ padding: '4px 12px', cursor: 'pointer' }}
            >+</button>
          </div>
        </div>
      </div>

      <div style={{
        display: 'grid',
        gridTemplateColumns: 'repeat(3, 1fr)',
        gap: 12,
        marginBottom: 20,
      }}>
        {[
          { label: 'Primos encontrados', value: primes.length.toLocaleString() },
          { label: 'Límite',             value: limit.toLocaleString() },
          { label: 'Mayor primo',        value: (primes.at(-1) ?? 0).toLocaleString() },
        ].map(({ label, value }) => (
          <div key={label} style={{ padding: 12, background: '#f5f5f5', borderRadius: 8, fontSize: 13 }}>
            <div style={{ color: '#888', marginBottom: 4 }}>{label}</div>
            <div style={{ fontWeight: 700, fontSize: 18 }}>{value}</div>
          </div>
        ))}
      </div>

      <details style={{ fontSize: 13 }}>
        <summary style={{ cursor: 'pointer', color: '#555' }}>Primeros 20 primos</summary>
        <div style={{ marginTop: 8, color: '#333', lineHeight: 1.8 }}>
          {primes.slice(0, 20).join(', ')}
        </div>
      </details>
    </div>
  )
}
```

### `src/App.tsx`

```tsx
// src/App.tsx

import PrimeSieve from './components/PrimeSieve'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  PrimeSieve       — useMemo para cálculo costoso (criba primos)  │
// │  2  FilteredCatalog  — dos useMemo encadenados: filtrar → ordenar   │
// │  3  OrderMetrics     — múltiples useMemo derivados de un filtro     │
// │  4  MultiTagFilter   — filtro AND por tags con conteos memoizados   │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

export default function App() {
  const content =
    PASO === 1 ? <PrimeSieve /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 620, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

### Prueba esto

- Arrastra el slider hacia la derecha (N = 100 000) — observa la primera carga (el cribado corre una vez)
- Pulsa `+` varias veces seguidas — el counter cambia pero los primos **no** se recalculan
- Mueve el slider de nuevo — ahora sí el cribado vuelve a ejecutarse (`limit` cambió)
- Comenta `useMemo` y reemplaza por `const primes = sieve(limit)` — pulsa `+` con N alto y nota la lentitud
- Inspeciona que `[limit]` es la única dep — si la omites, los primos nunca se actualizan

---

## PASO 2 — `src/components/FilteredCatalog.tsx`

Filtra y ordena un catálogo de productos con dos `useMemo` encadenados.
El segundo depende de la salida del primero.

```tsx
// src/components/FilteredCatalog.tsx

import { useState, useMemo } from 'react'

interface Product {
  id:       number
  name:     string
  category: string
  price:    number
  active:   boolean
  stock:    number
}

const CATALOG: Product[] = [
  { id:  1, name: 'Teclado mecánico',     category: 'Periféricos',    price:  89.99, active: true,  stock: 15 },
  { id:  2, name: 'Monitor 27"',          category: 'Pantallas',      price: 349.99, active: true,  stock:  8 },
  { id:  3, name: 'Mouse inalámbrico',    category: 'Periféricos',    price:  29.99, active: false, stock:  0 },
  { id:  4, name: 'Webcam HD',            category: 'Cámaras',        price:  59.99, active: true,  stock: 22 },
  { id:  5, name: 'Auriculares BT',       category: 'Audio',          price: 149.99, active: true,  stock:  6 },
  { id:  6, name: 'Micrófono condensador',category: 'Audio',          price: 199.99, active: true,  stock:  3 },
  { id:  7, name: 'Hub USB-C 7 puertos',  category: 'Periféricos',    price:  44.99, active: false, stock:  0 },
  { id:  8, name: 'Monitor 32" 4K',       category: 'Pantallas',      price: 699.99, active: true,  stock:  4 },
  { id:  9, name: 'SSD NVMe 2TB',         category: 'Almacenamiento', price: 149.99, active: true,  stock: 11 },
  { id: 10, name: 'Cámara mirrorless',    category: 'Cámaras',        price: 899.99, active: true,  stock:  2 },
]

type SortKey = 'name' | 'price' | 'stock'

export default function FilteredCatalog() {
  const [search,     setSearch]     = useState('')
  const [onlyActive, setOnlyActive] = useState(true)
  const [category,   setCategory]   = useState('Todas')
  const [sortBy,     setSortBy]     = useState<SortKey>('name')

  // useMemo 1 — filtrar
  const filtered = useMemo(() => {
    const q = search.toLowerCase()
    return CATALOG.filter(p =>
      (!onlyActive || p.active) &&
      (category === 'Todas' || p.category === category) &&
      (p.name.toLowerCase().includes(q) || p.category.toLowerCase().includes(q))
    )
  }, [search, onlyActive, category])

  // useMemo 2 — ordenar (depende de filtered)
  const sorted = useMemo(
    () => [...filtered].sort((a, b) =>
      sortBy === 'name'  ? a.name.localeCompare(b.name) :
      sortBy === 'price' ? a.price - b.price :
                           b.stock - a.stock
    ),
    [filtered, sortBy]
  )

  const categories = useMemo(
    () => ['Todas', ...new Set(CATALOG.map(p => p.category))],
    []
  )

  return (
    <div style={{ fontFamily: 'sans-serif', maxWidth: 600, margin: '0 auto', padding: 24 }}>
      <h2 style={{ fontSize: 18, fontWeight: 700, marginBottom: 4 }}>FilteredCatalog</h2>
      <p style={{ color: '#666', fontSize: 14, marginBottom: 20 }}>
        Dos <code>useMemo</code> encadenados: filtrar → ordenar.
      </p>

      <div style={{ display: 'flex', gap: 12, flexWrap: 'wrap', marginBottom: 16 }}>
        <input
          type="text"
          placeholder="Buscar..."
          value={search}
          onChange={e => setSearch(e.target.value)}
          style={{ flex: 1, minWidth: 140, padding: '6px 10px', border: '1px solid #ccc', borderRadius: 6 }}
        />
        <select
          value={category}
          onChange={e => setCategory(e.target.value)}
          style={{ padding: '6px 10px', border: '1px solid #ccc', borderRadius: 6 }}
        >
          {categories.map(c => <option key={c}>{c}</option>)}
        </select>
        <select
          value={sortBy}
          onChange={e => setSortBy(e.target.value as SortKey)}
          style={{ padding: '6px 10px', border: '1px solid #ccc', borderRadius: 6 }}
        >
          <option value="name">A–Z</option>
          <option value="price">Precio ↑</option>
          <option value="stock">Stock ↓</option>
        </select>
        <label style={{ display: 'flex', alignItems: 'center', gap: 6, fontSize: 14, cursor: 'pointer' }}>
          <input type="checkbox" checked={onlyActive} onChange={e => setOnlyActive(e.target.checked)} />
          Solo activos
        </label>
      </div>

      <p style={{ fontSize: 13, color: '#888', marginBottom: 12 }}>
        {sorted.length} de {CATALOG.length} productos
      </p>

      <div style={{ display: 'flex', flexDirection: 'column', gap: 8 }}>
        {sorted.map(p => (
          <div key={p.id} style={{
            display: 'flex', justifyContent: 'space-between', alignItems: 'center',
            padding: '10px 14px', background: p.active ? '#f9f9f9' : '#f0f0f0',
            borderRadius: 8, border: '1px solid #e5e5e5', opacity: p.active ? 1 : 0.6,
          }}>
            <div>
              <span style={{ fontWeight: 600, fontSize: 14 }}>{p.name}</span>
              <span style={{ marginLeft: 8, fontSize: 12, color: '#888' }}>{p.category}</span>
            </div>
            <div style={{ textAlign: 'right', fontSize: 13 }}>
              <div style={{ fontWeight: 700 }}>${p.price.toFixed(2)}</div>
              <div style={{ color: p.stock < 5 ? '#e00' : '#888' }}>Stock: {p.stock}</div>
            </div>
          </div>
        ))}
        {sorted.length === 0 && (
          <p style={{ textAlign: 'center', color: '#aaa', padding: 24 }}>Sin resultados.</p>
        )}
      </div>
    </div>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import FilteredCatalog from './components/FilteredCatalog'
```

```tsx
PASO === 2 ? <FilteredCatalog /> :
```

Cambia `PASO = 2` y guarda.

### Prueba esto

- Escribe "monitor" en el buscador — solo `filtered` se recalcula; `sorted` recalcula porque `filtered` cambió
- Cambia la categoría a "Audio" — filtra a los 2 productos de esa categoría
- Desmarca "Solo activos" — aparecen los productos con `active: false` (opacidad reducida)
- Cambia el orden a "Precio ↑" — `sorted` recalcula por la nueva dep `sortBy`
- Nota que `categories` usa `[]` — las categorías solo se derivan una vez porque `CATALOG` es constante

---

## PASO 3 — `src/components/OrderMetrics.tsx`

Deriva múltiples métricas de un array de pedidos.
Cada `useMemo` es independiente — solo recalcula la métrica que depende del estado que cambió.

```tsx
// src/components/OrderMetrics.tsx

import { useState, useMemo } from 'react'

interface Order {
  id:        number
  customer:  string
  amount:    number
  status:    'pending' | 'paid' | 'refunded'
  createdAt: string
}

const ORDERS: Order[] = [
  { id: 1, customer: 'Ana García',    amount:  120.00, status: 'paid',     createdAt: '2024-01-15' },
  { id: 2, customer: 'Luis Pérez',    amount:  340.50, status: 'paid',     createdAt: '2024-01-18' },
  { id: 3, customer: 'María López',   amount:   89.99, status: 'pending',  createdAt: '2024-01-20' },
  { id: 4, customer: 'Carlos Ruiz',   amount:  560.00, status: 'refunded', createdAt: '2024-01-22' },
  { id: 5, customer: 'Ana García',    amount:  210.00, status: 'paid',     createdAt: '2024-02-01' },
  { id: 6, customer: 'Sofía Torres',  amount:   75.00, status: 'pending',  createdAt: '2024-02-05' },
  { id: 7, customer: 'Luis Pérez',    amount: 1100.00, status: 'paid',     createdAt: '2024-02-08' },
  { id: 8, customer: 'Elena Díaz',    amount:  290.00, status: 'paid',     createdAt: '2024-02-10' },
]

export default function OrderMetrics() {
  const [statusFilter, setStatusFilter] = useState<Order['status'] | 'all'>('all')
  const [minAmount,    setMinAmount]    = useState(0)

  const visibleOrders = useMemo(
    () => ORDERS.filter(o =>
      (statusFilter === 'all' || o.status === statusFilter) &&
      o.amount >= minAmount
    ),
    [statusFilter, minAmount]
  )

  const total    = useMemo(() => visibleOrders.reduce((s, o) => s + o.amount, 0), [visibleOrders])
  const average  = useMemo(() => visibleOrders.length ? total / visibleOrders.length : 0, [total, visibleOrders.length])
  const maxOrder = useMemo(
    () => visibleOrders.reduce<Order | null>((max, o) => (!max || o.amount > max.amount) ? o : max, null),
    [visibleOrders]
  )
  const byStatus = useMemo(
    () => ({
      paid:     visibleOrders.filter(o => o.status === 'paid').length,
      pending:  visibleOrders.filter(o => o.status === 'pending').length,
      refunded: visibleOrders.filter(o => o.status === 'refunded').length,
    }),
    [visibleOrders]
  )

  const STATUS_COLORS: Record<Order['status'], string> = {
    paid: '#15803d', pending: '#b45309', refunded: '#9333ea',
  }

  return (
    <div style={{ fontFamily: 'sans-serif', maxWidth: 580, margin: '0 auto', padding: 24 }}>
      <h2 style={{ fontSize: 18, fontWeight: 700, marginBottom: 4 }}>OrderMetrics</h2>
      <p style={{ color: '#666', fontSize: 14, marginBottom: 20 }}>
        Múltiples <code>useMemo</code> independientes derivados de un filtro base.
      </p>

      <div style={{ display: 'flex', gap: 16, flexWrap: 'wrap', marginBottom: 24 }}>
        <label style={{ fontSize: 14, display: 'flex', flexDirection: 'column', gap: 4 }}>
          Estado
          <select
            value={statusFilter}
            onChange={e => setStatusFilter(e.target.value as typeof statusFilter)}
            style={{ padding: '6px 10px', border: '1px solid #ccc', borderRadius: 6 }}
          >
            <option value="all">Todos</option>
            <option value="paid">Pagado</option>
            <option value="pending">Pendiente</option>
            <option value="refunded">Reembolsado</option>
          </select>
        </label>
        <label style={{ fontSize: 14, display: 'flex', flexDirection: 'column', gap: 4 }}>
          Importe mínimo: ${minAmount}
          <input
            type="range" min={0} max={500} step={50} value={minAmount}
            onChange={e => setMinAmount(Number(e.target.value))}
            style={{ width: 160 }}
          />
        </label>
      </div>

      <div style={{ display: 'grid', gridTemplateColumns: 'repeat(2, 1fr)', gap: 12, marginBottom: 24 }}>
        {[
          { label: 'Pedidos visibles', value: visibleOrders.length },
          { label: 'Total',            value: `$${total.toFixed(2)}` },
          { label: 'Promedio',         value: `$${average.toFixed(2)}` },
          { label: 'Mayor pedido',     value: maxOrder ? `$${maxOrder.amount.toFixed(2)} (${maxOrder.customer})` : '—' },
        ].map(({ label, value }) => (
          <div key={label} style={{ padding: 14, background: '#f5f5f5', borderRadius: 10, fontSize: 13 }}>
            <div style={{ color: '#888', marginBottom: 4 }}>{label}</div>
            <div style={{ fontWeight: 700, fontSize: 16 }}>{value}</div>
          </div>
        ))}
      </div>

      <div style={{ display: 'flex', gap: 8, marginBottom: 24 }}>
        {(Object.entries(byStatus) as [Order['status'], number][]).map(([status, count]) => (
          <span key={status} style={{
            padding: '4px 12px', borderRadius: 999, fontSize: 12, fontWeight: 600,
            background: `${STATUS_COLORS[status]}20`, color: STATUS_COLORS[status],
          }}>
            {status}: {count}
          </span>
        ))}
      </div>

      <table style={{ width: '100%', borderCollapse: 'collapse', fontSize: 13 }}>
        <thead>
          <tr>
            {['#', 'Cliente', 'Importe', 'Estado', 'Fecha'].map(h => (
              <th key={h} style={{
                textAlign: 'left', padding: '6px 8px',
                borderBottom: '2px solid #e5e5e5', color: '#666', fontWeight: 600,
              }}>{h}</th>
            ))}
          </tr>
        </thead>
        <tbody>
          {visibleOrders.map(o => (
            <tr key={o.id}>
              <td style={{ padding: '6px 8px', color: '#aaa' }}>{o.id}</td>
              <td style={{ padding: '6px 8px', fontWeight: 500 }}>{o.customer}</td>
              <td style={{ padding: '6px 8px', fontWeight: 700 }}>${o.amount.toFixed(2)}</td>
              <td style={{ padding: '6px 8px' }}>
                <span style={{
                  padding: '2px 8px', borderRadius: 999, fontSize: 11, fontWeight: 700,
                  background: `${STATUS_COLORS[o.status]}20`, color: STATUS_COLORS[o.status],
                }}>
                  {o.status}
                </span>
              </td>
              <td style={{ padding: '6px 8px', color: '#888' }}>{o.createdAt}</td>
            </tr>
          ))}
          {visibleOrders.length === 0 && (
            <tr><td colSpan={5} style={{ textAlign: 'center', padding: 24, color: '#aaa' }}>Sin pedidos.</td></tr>
          )}
        </tbody>
      </table>
    </div>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import OrderMetrics from './components/OrderMetrics'
```

```tsx
PASO === 3 ? <OrderMetrics /> :
```

Cambia `PASO = 3` y guarda.

### Prueba esto

- Cambia el filtro a "Pagado" — `visibleOrders` recalcula y en cascada: `total`, `average`, `maxOrder`, `byStatus`
- Sube el importe mínimo a $200 — filtra los pedidos pequeños; el promedio sube
- Fíjate que `average` depende de `[total, visibleOrders.length]` — reutiliza el `total` del memo anterior
- Pon filtro "Todos" e importe mínimo $500 — solo un pedido pasa; el promedio es igual al total

---

## PASO 4 — `src/components/MultiTagFilter.tsx`

Filtrado por múltiples tags con conteos por tag calculados con `useMemo`.
Demuestra `useMemo` para estabilizar un objeto que se pasaría a un componente hijo.

```tsx
// src/components/MultiTagFilter.tsx

import { useState, useMemo } from 'react'

interface Article {
  id:    number
  title: string
  tags:  string[]
  views: number
}

const ARTICLES: Article[] = [
  { id: 1, title: 'Introducción a React Hooks',         tags: ['react', 'hooks', 'tutorial'],    views: 4200 },
  { id: 2, title: 'TypeScript con React: guía práctica',tags: ['typescript', 'react', 'guía'],  views: 3100 },
  { id: 3, title: 'useMemo y useCallback explicados',   tags: ['react', 'hooks', 'performance'], views: 2800 },
  { id: 4, title: 'CSS Modules vs Styled Components',   tags: ['css', 'estilos', 'react'],       views: 1900 },
  { id: 5, title: 'TanStack Query desde cero',          tags: ['react', 'fetch', 'tutorial'],    views: 5100 },
  { id: 6, title: 'Testing con Vitest y RTL',           tags: ['testing', 'react', 'tutorial'],  views: 2200 },
  { id: 7, title: 'Performance en React: técnicas clave',tags:['react', 'performance', 'hooks'], views: 3600 },
  { id: 8, title: 'TypeScript strict mode explicado',   tags: ['typescript', 'guía'],            views: 1500 },
]

export default function MultiTagFilter() {
  const [activeTags,  setActiveTags]  = useState<Set<string>>(new Set())
  const [sortByViews, setSortByViews] = useState(false)

  const tagCounts = useMemo(() => {
    const counts: Record<string, number> = {}
    ARTICLES.forEach(a => a.tags.forEach(t => { counts[t] = (counts[t] ?? 0) + 1 }))
    return counts
  }, [])

  const filtered = useMemo(() => {
    if (activeTags.size === 0) return ARTICLES
    return ARTICLES.filter(a => [...activeTags].every(t => a.tags.includes(t)))
  }, [activeTags])

  const sorted = useMemo(
    () => sortByViews ? [...filtered].sort((a, b) => b.views - a.views) : filtered,
    [filtered, sortByViews]
  )

  function toggleTag(tag: string) {
    setActiveTags(prev => {
      const next = new Set(prev)
      next.has(tag) ? next.delete(tag) : next.add(tag)
      return next
    })
  }

  return (
    <div style={{ fontFamily: 'sans-serif', maxWidth: 580, margin: '0 auto', padding: 24 }}>
      <h2 style={{ fontSize: 18, fontWeight: 700, marginBottom: 4 }}>MultiTagFilter</h2>
      <p style={{ color: '#666', fontSize: 14, marginBottom: 20 }}>
        Tags múltiples con filtro AND. Conteos memoizados — se calculan una sola vez.
      </p>

      <div style={{ display: 'flex', flexWrap: 'wrap', gap: 8, marginBottom: 16 }}>
        {Object.entries(tagCounts).map(([tag, count]) => {
          const active = activeTags.has(tag)
          return (
            <button
              key={tag}
              onClick={() => toggleTag(tag)}
              style={{
                padding: '4px 12px', borderRadius: 999, border: '1px solid',
                borderColor: active ? '#0070f3' : '#ddd',
                background:  active ? '#0070f3' : 'white',
                color:       active ? 'white'    : '#555',
                fontSize: 13, cursor: 'pointer', fontWeight: active ? 700 : 400,
              }}
            >
              {tag} <span style={{ opacity: 0.7 }}>({count})</span>
            </button>
          )
        })}
      </div>

      <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: 16 }}>
        <span style={{ fontSize: 13, color: '#888' }}>
          {sorted.length} artículo{sorted.length !== 1 ? 's' : ''}
          {activeTags.size > 0 && ` (filtrado: ${[...activeTags].join(' + ')})`}
        </span>
        <label style={{ display: 'flex', alignItems: 'center', gap: 6, fontSize: 13, cursor: 'pointer' }}>
          <input type="checkbox" checked={sortByViews} onChange={e => setSortByViews(e.target.checked)} />
          Ordenar por visitas
        </label>
      </div>

      <div style={{ display: 'flex', flexDirection: 'column', gap: 10 }}>
        {sorted.map(a => (
          <div key={a.id} style={{
            padding: '12px 16px', background: '#f9f9f9', borderRadius: 10, border: '1px solid #eee',
          }}>
            <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'flex-start' }}>
              <span style={{ fontWeight: 600, fontSize: 14 }}>{a.title}</span>
              <span style={{ fontSize: 12, color: '#888', whiteSpace: 'nowrap', marginLeft: 12 }}>
                {a.views.toLocaleString()} vistas
              </span>
            </div>
            <div style={{ display: 'flex', gap: 6, marginTop: 8, flexWrap: 'wrap' }}>
              {a.tags.map(t => (
                <span
                  key={t}
                  onClick={() => toggleTag(t)}
                  style={{
                    padding: '2px 8px', borderRadius: 999, fontSize: 11, cursor: 'pointer',
                    background: activeTags.has(t) ? '#0070f320' : '#f0f0f0',
                    color:      activeTags.has(t) ? '#0070f3'   : '#666',
                    fontWeight: activeTags.has(t) ? 700 : 400,
                  }}
                >
                  {t}
                </span>
              ))}
            </div>
          </div>
        ))}
      </div>

      {activeTags.size > 0 && (
        <button
          onClick={() => setActiveTags(new Set())}
          style={{
            marginTop: 16, padding: '6px 16px', borderRadius: 6,
            border: '1px solid #ddd', cursor: 'pointer', fontSize: 13, background: 'white',
          }}
        >
          Limpiar filtros
        </button>
      )}
    </div>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import MultiTagFilter from './components/MultiTagFilter'
```

```tsx
PASO === 4 ? <MultiTagFilter /> :
```

Cambia `PASO = 4` y guarda. Desde aquí puedes volver a cualquier paso anterior cambiando la constante.

### Prueba esto

- Haz clic en el tag `react` — filtra los artículos que lo tienen
- Agrega `hooks` — filtro AND: solo artículos con `react` AND `hooks`
- Activa "Ordenar por visitas" — `sorted` recalcula; `filtered` no recalcula
- Haz clic en un tag dentro de un artículo — lo activa sin usar los botones del encabezado
- Cambia `PASO = 1` para volver a `PrimeSieve` — pulsa el counter y recuerda el contraste con y sin memo

---

## Resumen de la página 9

- `useMemo` memoriza el resultado de una función; solo recalcula cuando cambian sus dependencias.
- El array de dependencias vacío `[]` indica que el valor se calcula una sola vez (útil para datos estáticos).
- Los `useMemo` pueden encadenarse: el segundo depende de la salida del primero.
- Cada `useMemo` es independiente — solo recalcula la pieza que depende del estado que cambió.
- No memoices cálculos triviales: el costo de comparar dependencias puede superar el de recalcular.
- `useMemo` estabiliza referencias de objetos/arrays; útil para evitar re-renders de hijos memoizados.

---

> **Siguiente página →** Página 10: `useCallback` — memoización de funciones.
> **Páginas anteriores →** Página 8: `useContext` · Página 6: intro a `useRef`, `useCallback`, `useMemo`.
