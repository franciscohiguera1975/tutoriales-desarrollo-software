# Curso React.js + TypeScript — Página 11
## Módulo 5 · Arquitectura
### Rendimiento: `React.memo`, patrones de optimización y code splitting

---

## ¿Cuándo re-renderiza React?

Un componente se re-renderiza cuando:

1. **Su propio estado cambia** — `useState` o `useReducer`
2. **Sus props cambian** — cualquier prop, por comparación de referencia
3. **Su contexto cambia** — cualquier valor del contexto que consume
4. **Su padre se re-renderiza** — aunque las props no hayan cambiado

El punto 4 es el más sorprendente: **por defecto, si el padre re-renderiza,
todos los hijos re-renderizan**, aunque sus props sean idénticas.

```
App (re-renderiza porque su estado cambió)
 ├── Header     ← re-renderiza aunque no tenga props del App
 ├── Sidebar    ← re-renderiza aunque no tenga props del App
 └── MainContent ← re-renderiza aunque no tenga props del App
      └── ProductList ← re-renderiza
           └── ProductRow × 100 ← 100 re-renders innecesarios
```

`React.memo` corta esta cadena.

---

## `React.memo`

`React.memo` envuelve un componente y le indica a React que **solo lo
re-renderice si sus props cambiaron**. Usa comparación superficial por defecto:
compara cada prop con `===`.

```tsx
import { memo } from 'react'

// Sin memo — re-renderiza siempre que el padre re-renderice
function UserCard({ name, email }: UserCardProps) {
  return <div>{name} — {email}</div>
}

// Con memo — solo re-renderiza si name o email cambian
const UserCard = memo(function UserCard({ name, email }: UserCardProps) {
  return <div>{name} — {email}</div>
})
```

### La trampa de las funciones como props

`React.memo` compara props con `===`. Las funciones son objetos — dos
funciones con el mismo código son referencias distintas:

```tsx
function App() {
  // Esta función se recrea en cada render de App
  function handleEdit() { console.log('edit') }

  // Aunque UserCard tenga memo, re-renderiza porque
  // handleEdit es una nueva referencia en cada render
  return <UserCard onEdit={handleEdit} />
}
```

La solución: `useCallback` para estabilizar la referencia.

---

## `src/components/UserCard.tsx`

```tsx
// src/components/UserCard.tsx

import { memo } from 'react'

interface UserCardProps {
  name:    string
  email:   string
  role:    'admin' | 'editor' | 'viewer'
  onEdit:  () => void
}

// memo — solo re-renderiza si alguna prop cambia (comparación ===)
const UserCard = memo(function UserCard({
  name,
  email,
  role,
  onEdit,
}: UserCardProps) {
  return (
    <div style={{
      display: 'flex', justifyContent: 'space-between', alignItems: 'center',
      padding: '12px 16px', border: '1px solid #e5e7eb', borderRadius: 8,
    }}>
      <div>
        <p style={{ margin: 0, fontWeight: 600 }}>{name}</p>
        <p style={{ margin: 0, fontSize: 13, color: '#6b7280' }}>{email}</p>
        <span style={{
          fontSize: 11, padding: '2px 8px', borderRadius: 10,
          background: role === 'admin' ? '#eff6ff' : '#f9fafb',
          color:      role === 'admin' ? '#1d4ed8' : '#6b7280',
          fontWeight: 600,
        }}>
          {role}
        </span>
      </div>
      <button
        onClick={onEdit}
        style={{
          padding: '6px 14px', border: '1px solid #d1d5db',
          borderRadius: 6, background: '#fff', cursor: 'pointer',
        }}
      >
        Editar
      </button>
    </div>
  )
})

export default UserCard
```

### Prueba esto

- Cambia `memo(function UserCard(...))` por `function UserCard(...)` (sin memo), añade `console.log('render UserCard')` dentro y haz clic repetidamente en el botón del padre — observa que `UserCard` se re-renderiza en cada clic aunque sus props no hayan cambiado
- Vuelve a añadir `memo` pero pasa `onEdit` como función anónima `onEdit={() => console.log('edit')}` en el padre — observa en la consola que `UserCard` sigue re-renderizando porque cada render del padre crea una función nueva
- Envuelve `onEdit` con `useCallback(() => console.log('edit'), [])` en el padre y confirma que el log de render de `UserCard` ya no aparece al hacer clic en el botón del padre
- Cambia el `role` de `"admin"` a `"viewer"` en las props — observa que `UserCard` sí se re-renderiza porque la prop `role` realmente cambió, que es el comportamiento correcto de `memo`
- Abre React DevTools, activa "Highlight updates when components render" y haz clic en el botón del padre — con `memo` + `useCallback`, `UserCard` no se resalta; sin ellos, sí

---

## El trío: `memo` + `useCallback` + `useMemo`

Los tres trabajan juntos. Sin `useCallback`, `memo` no sirve de nada
cuando se pasan funciones como props.

```
memo       → evita que el hijo re-renderice por props iguales
useCallback → estabiliza funciones pasadas como props
useMemo     → estabiliza objetos/arrays calculados pasados como props
```

---

## `src/components/ProductTable.tsx`

Patrón completo: lista con `memo` en cada fila, `useCallback` en los
handlers y `useMemo` para el array filtrado.

```tsx
// src/components/ProductTable.tsx

import { useState, useCallback, useMemo, memo } from 'react'

interface Product {
  id:       number
  name:     string
  price:    number
  category: string
}

interface ProductRowProps {
  product:  Product
  onDelete: (id: number) => void
  onEdit:   (product: Product) => void
}

// memo — solo re-renderiza si product, onDelete u onEdit cambian
const ProductRow = memo(function ProductRow({
  product,
  onDelete,
  onEdit,
}: ProductRowProps) {
  return (
    <tr>
      <td style={{ padding: '10px 12px' }}>{product.name}</td>
      <td style={{ padding: '10px 12px', color: '#6b7280' }}>{product.category}</td>
      <td style={{ padding: '10px 12px', fontWeight: 600 }}>${product.price}</td>
      <td style={{ padding: '10px 12px' }}>
        <button
          onClick={() => onEdit(product)}
          style={actionBtn('#eff6ff', '#1d4ed8')}
        >
          Editar
        </button>
        <button
          onClick={() => onDelete(product.id)}
          style={actionBtn('#fef2f2', '#dc2626')}
        >
          Eliminar
        </button>
      </td>
    </tr>
  )
})

const INITIAL_PRODUCTS: Product[] = [
  { id: 1, name: 'Teclado mecánico',  price: 89,  category: 'Periféricos' },
  { id: 2, name: 'Monitor 27"',       price: 349, category: 'Pantallas'   },
  { id: 3, name: 'Mouse inalámbrico', price: 29,  category: 'Periféricos' },
  { id: 4, name: 'Webcam HD',         price: 59,  category: 'Cámaras'     },
  { id: 5, name: 'Auriculares BT',    price: 149, category: 'Audio'       },
]

export default function ProductTable() {
  const [products, setProducts] = useState<Product[]>(INITIAL_PRODUCTS)
  const [filter,   setFilter]   = useState('')

  // useCallback — handleDelete mantiene la misma referencia
  // mientras products no cambie desde fuera
  const handleDelete = useCallback((id: number) => {
    setProducts((prev) => prev.filter((p) => p.id !== id))
  }, [])

  // useCallback con dependencias vacías — onEdit no depende del estado
  const handleEdit = useCallback((product: Product) => {
    console.log('Editando producto:', product.name)
    // aquí abriría un modal, navegaría, etc.
  }, [])

  // useMemo — filtered solo recalcula cuando products o filter cambian
  const filtered = useMemo(
    () => products.filter((p) =>
      p.name.toLowerCase().includes(filter.toLowerCase()) ||
      p.category.toLowerCase().includes(filter.toLowerCase())
    ),
    [products, filter]
  )

  const totalValue = useMemo(
    () => filtered.reduce((acc, p) => acc + p.price, 0),
    [filtered]
  )

  return (
    <div style={{ maxWidth: 560 }}>
      <div style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 12 }}>
        <input
          value={filter}
          onChange={(e) => setFilter(e.target.value)}
          placeholder="Filtrar productos..."
          style={{ padding: '8px 12px', border: '1px solid #d1d5db', borderRadius: 6, flex: 1 }}
        />
        <span style={{ alignSelf: 'center', marginLeft: 16, fontSize: 14, color: '#6b7280' }}>
          Total: <strong>${totalValue}</strong>
        </span>
      </div>

      <table style={{ width: '100%', borderCollapse: 'collapse', fontSize: 14 }}>
        <thead>
          <tr style={{ background: '#f9fafb' }}>
            {['Nombre', 'Categoría', 'Precio', 'Acciones'].map((h) => (
              <th
                key={h}
                style={{
                  padding: '8px 12px', textAlign: 'left',
                  fontWeight: 600, fontSize: 12,
                  color: '#6b7280', borderBottom: '1px solid #e5e7eb',
                }}
              >
                {h}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {filtered.map((product) => (
            <ProductRow
              key={product.id}
              product={product}
              onDelete={handleDelete}
              onEdit={handleEdit}
            />
          ))}
        </tbody>
      </table>

      {filtered.length === 0 && (
        <p style={{ textAlign: 'center', color: '#9ca3af', padding: 20 }}>
          Sin resultados.
        </p>
      )}
    </div>
  )
}

function actionBtn(bg: string, color: string): React.CSSProperties {
  return {
    padding: '4px 10px', marginRight: 6,
    border: 'none', borderRadius: 4,
    background: bg, color, cursor: 'pointer',
    fontSize: 12, fontWeight: 500,
  }
}
```

### Prueba esto

- Añade `console.log('render ProductRow', product.name)` dentro de `ProductRow` y escribe en el filtro — observa que solo se re-renderizan las filas cuyas props cambiaron, no todas
- Quita `memo` de `ProductRow` y repite — observa en la consola que todas las filas se re-renderizan en cada pulsación de tecla aunque los datos de los productos no hayan cambiado
- Cambia `useCallback` de `handleDelete` a una función normal `function handleDelete(id: number) { ... }` declarada dentro de `ProductTable` — observa que todas las filas con `memo` vuelven a re-renderizarse al escribir en el filtro porque `handleDelete` tiene una referencia nueva en cada render
- Cambia las dependencias de `useMemo` para `filtered` de `[products, filter]` a `[]` — escribe en el filtro y observa que la lista nunca se actualiza porque `filtered` quedó congelado con el valor inicial
- Elimina `<ProductRow key={product.id} ...>` y pon `key={product.name}` — renombra mentalmente un producto añadiendo un carácter al `name` y observa que React desmonta y vuelve a montar la fila en lugar de actualizarla
- Haz clic en "Eliminar" en un producto — observa que el `totalValue` se recalcula automáticamente porque `filtered` es una dependencia de ese `useMemo`

---

## `React.memo` con comparador personalizado

Por defecto `memo` hace comparación superficial con `===`. Para arrays
u objetos complejos en props, esa comparación falla — dos arrays con el
mismo contenido son referencias distintas. El segundo argumento de `memo`
permite definir la comparación:

```tsx
// src/components/ExpensiveChart.tsx

import { memo } from 'react'

interface ExpensiveChartProps {
  data:   number[]
  label:  string
  color?: string
}

const ExpensiveChart = memo(
  function ExpensiveChart({
    data,
    label,
    color = '#0070f3',
  }: ExpensiveChartProps) {
    // cálculo costoso — simula una librería de gráficos pesada
    const max = Math.max(...data)
    const min = Math.min(...data)
    const avg = data.reduce((a, b) => a + b, 0) / data.length

    return (
      <div style={{ padding: 16, border: '1px solid #e5e7eb', borderRadius: 8 }}>
        <p style={{ margin: '0 0 8px', fontWeight: 600, color }}>{label}</p>
        <div style={{ display: 'flex', gap: 16, fontSize: 13 }}>
          <span>Mín: <strong>{min}</strong></span>
          <span>Máx: <strong>{max}</strong></span>
          <span>Prom: <strong>{avg.toFixed(1)}</strong></span>
        </div>
        {/* Barras simples */}
        <div style={{ display: 'flex', gap: 4, marginTop: 12, alignItems: 'flex-end', height: 60 }}>
          {data.map((v, i) => (
            <div
              key={i}
              style={{
                flex: 1,
                height: `${(v / max) * 100}%`,
                background: color,
                borderRadius: '2px 2px 0 0',
                opacity: 0.8,
              }}
            />
          ))}
        </div>
      </div>
    )
  },
  // Comparador personalizado — evita re-renders cuando el array
  // tiene el mismo contenido aunque sea una referencia nueva
  (prevProps, nextProps) =>
    prevProps.label === nextProps.label &&
    prevProps.color === nextProps.color &&
    prevProps.data.length === nextProps.data.length &&
    prevProps.data.every((v, i) => v === nextProps.data[i])
)

export default ExpensiveChart
```

> Usa el comparador personalizado con cuidado. Si devuelve `true`
> (props iguales) cuando en realidad cambiaron, el componente mostrará
> datos desactualizados. El comparador incorrecto es peor que no tener `memo`.

### Prueba esto

- Pasa `data={[42, 78, 35, 91, 63, 57, 84]}` como array literal en el JSX del padre (sin `useMemo`) — observa que el comparador personalizado evalúa `every` y evita el re-render aunque sea un array nuevo en cada render del padre
- Quita el comparador personalizado (solo pasa un argumento a `memo`) y pasa el array como literal — observa que ahora `ExpensiveChart` se re-renderiza en cada render del padre porque `===` compara referencias y el array literal es nuevo cada vez
- Modifica el comparador para que siempre devuelva `true` — cambia un valor en el `data` y observa que el gráfico no se actualiza aunque los datos hayan cambiado; este es el bug que produce un comparador incorrecto
- Cambia `prevProps.data.length === nextProps.data.length` en el comparador por una comparación falsa como `prevProps.data.length !== nextProps.data.length` — observa que ahora el componente siempre re-renderiza aunque los datos sean iguales
- Añade `console.log('render ExpensiveChart')` dentro del componente y provoca re-renders del padre — con el comparador correcto, el log no debe aparecer si los datos y el label son iguales

---

## Medir renders con `useRef`

Antes de optimizar, verifica que el problema existe. Un contador de renders
simple con `useRef` muestra cuántas veces se ejecuta cada componente:

```tsx
// src/components/RenderCounter.tsx
// Solo para desarrollo — quita antes de producción

import { useRef } from 'react'

interface RenderCounterProps {
  label: string
}

export default function RenderCounter({ label }: RenderCounterProps) {
  const renderCount = useRef(0)
  renderCount.current++

  return (
    <span style={{
      fontSize: 11, padding: '1px 6px',
      background: renderCount.current > 3 ? '#fef2f2' : '#f0fdf4',
      color:      renderCount.current > 3 ? '#dc2626' : '#16a34a',
      borderRadius: 4, fontFamily: 'monospace',
    }}>
      {label}: {renderCount.current} renders
    </span>
  )
}
```

```tsx
// Usar temporalmente en el componente que sospechas
function ProductRow({ product, onDelete, onEdit }: ProductRowProps) {
  return (
    <tr>
      <td><RenderCounter label="ProductRow" /></td>
      <td>{product.name}</td>
      ...
    </tr>
  )
}
```

### Prueba esto

- Añade `<RenderCounter label="App" />` en `App` y `<RenderCounter label="UserCard" />` en `UserCard` — haz clic en el botón "Re-renderizar App" varias veces y observa que el contador de `App` sube pero el de `UserCard` se mantiene en 1 gracias a `memo`
- Quita `memo` de `UserCard` temporalmente y observa cómo el contador de `UserCard` ahora sincroniza con el de `App`
- Observa el cambio de color del badge: después de 3 renders pasa de verde a rojo — usa esto para identificar qué componentes están re-renderizando excesivamente
- Añade `<RenderCounter label="ProductRow" />` dentro de `ProductRow` y escribe rápido en el filtro — observa cuántos renders se producen por pulsación de tecla con y sin `memo`
- Comprueba que `renderCount.current++` no provoca re-renders adicionales — a diferencia de `useState`, actualizar un `ref` es silencioso para React

---

## Code splitting con `lazy` y `Suspense`

`lazy` permite cargar un componente solo cuando se necesita —
el bundle de ese componente se descarga en ese momento, no al inicio.

En producción esto reduce el tamaño del bundle inicial significativamente.

```tsx
// src/App.tsx — lazy loading de páginas completas

import { lazy, Suspense } from 'react'
import { Routes, Route }  from 'react-router-dom'
import RootLayout         from './layouts/RootLayout'

// Las páginas se cargan solo cuando el usuario navega a esa ruta
const HomePage          = lazy(() => import('./pages/HomePage'))
const ProductsPage      = lazy(() => import('./pages/ProductsPage'))
const ProductDetailPage = lazy(() => import('./pages/ProductDetailPage'))
const DashboardPage     = lazy(() => import('./pages/DashboardPage'))
const AboutPage         = lazy(() => import('./pages/AboutPage'))
const NotFoundPage      = lazy(() => import('./pages/NotFoundPage'))

function PageLoader() {
  return (
    <div style={{ display: 'flex', justifyContent: 'center', padding: 48 }}>
      <p style={{ color: '#9ca3af' }}>Cargando...</p>
    </div>
  )
}

export default function App() {
  return (
    // Suspense muestra el fallback mientras el chunk se descarga
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route element={<RootLayout />}>
          <Route index                element={<HomePage />} />
          <Route path="products"      element={<ProductsPage />} />
          <Route path="products/:id"  element={<ProductDetailPage />} />
          <Route path="dashboard"     element={<DashboardPage />} />
          <Route path="about"         element={<AboutPage />} />
          <Route path="*"             element={<NotFoundPage />} />
        </Route>
      </Routes>
    </Suspense>
  )
}
```

> `lazy` solo funciona con `export default`. Si el componente usa
> named export, necesitas un archivo intermedio o adaptas la importación:
>
> ```tsx
> // Componente con named export
> const MyPage = lazy(() =>
>   import('./pages/MyPage').then((m) => ({ default: m.MyPage }))
> )
> ```

### Prueba esto

- Abre las DevTools de red, ve a la pestaña "JS" y navega por primera vez a `/products` — observa que se descarga un chunk separado (algo como `ProductsPage-[hash].js`) solo en ese momento
- Navega de `/` a `/dashboard` — el chunk del dashboard se descarga solo cuando llegas allí; observa en la red el nuevo archivo `.js`
- Simula una conexión lenta con el throttling de DevTools (3G lento) y navega a una ruta perezosa — observa el `<p>Cargando...</p>` del `PageLoader` durante la descarga del chunk
- Elimina `<Suspense>` manteniendo `lazy` y navega a una ruta perezosa — observa el error en consola: React exige un `Suspense` ancestro para cualquier componente `lazy`
- Cambia `lazy(() => import('./pages/HomePage'))` a `import('./pages/HomePage')` (sin `lazy`) y navega — el componente se carga de inmediato pero el import dinámico sin `lazy` no integra con `Suspense` y React lo trata como una promesa normal

---

## `src/App.tsx` — todo junto

```tsx
// src/App.tsx

import { useState, useCallback } from 'react'
import UserCard       from './components/UserCard'
import ProductTable   from './components/ProductTable'
import ExpensiveChart from './components/ExpensiveChart'
import RenderCounter  from './components/RenderCounter'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  UserCard      — memo + useCallback, re-renders del padre        │
// │  2  ProductTable  — memo + useCallback + useMemo en tabla           │
// │  3  ExpensiveChart — memo con comparador personalizado              │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

const CHART_DATA = [42, 78, 35, 91, 63, 57, 84]

function Step1() {
  const [count, setCount] = useState(0)

  // useCallback — referencia estable para UserCard memo
  const handleEditUser = useCallback(() => {
    console.log('Editando usuario')
  }, [])

  return (
    <div>
      <div style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 12 }}>
        <h2 style={{ fontSize: 15, margin: 0 }}>React.memo — UserCard</h2>
        <RenderCounter label="App" />
      </div>

      <button
        onClick={() => setCount((c) => c + 1)}
        style={{
          padding: '8px 16px', background: '#6366f1', color: '#fff',
          border: 'none', borderRadius: 6, cursor: 'pointer', marginBottom: 12,
        }}
      >
        Re-renderizar App ({count} veces)
      </button>

      <p style={{ fontSize: 13, color: '#6b7280', marginBottom: 12 }}>
        UserCard tiene memo + onEdit con useCallback — no re-renderiza al hacer clic en el botón.
      </p>

      <UserCard
        name="Ana García"
        email="ana@ejemplo.com"
        role="admin"
        onEdit={handleEditUser}
      />
    </div>
  )
}

function Step2() {
  return (
    <div>
      <h2 style={{ fontSize: 15, marginBottom: 12 }}>
        memo + useCallback + useMemo — ProductTable
      </h2>
      <ProductTable />
    </div>
  )
}

function Step3() {
  return (
    <div>
      <h2 style={{ fontSize: 15, marginBottom: 12 }}>
        memo con comparador personalizado — ExpensiveChart
      </h2>
      <ExpensiveChart
        data={CHART_DATA}
        label="Ventas semanales"
        color="#0070f3"
      />
    </div>
  )
}

export default function App() {
  const content =
    PASO === 1 ? <Step1 /> :
    PASO === 2 ? <Step2 /> :
    PASO === 3 ? <Step3 /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 580, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

### Prueba esto

- Cambia `PASO` a `1` y guarda — observa `UserCard` con el botón de re-renderizar; el contador de App sube pero `UserCard` no se re-renderiza
- Cambia `PASO` a `2` y guarda — interactúa con el filtro y con los botones de eliminar; observa el total recalculándose con `useMemo`
- Cambia `PASO` a `3` y guarda — abre la consola y provoca un re-render del padre; el gráfico no se re-renderiza gracias al comparador personalizado
- Cambia `PASO` a `99` y guarda — observa el mensaje de error en rojo indicando que el paso no existe
- En `PASO === 1`, quita `useCallback` de `handleEditUser` y añade `<RenderCounter label="UserCard" />` dentro de `UserCard` — observa que el contador sube con cada clic en el botón aunque el usuario nunca interactuó con la tarjeta

---

## Cuándo optimizar — y cuándo no

| Optimiza cuando... | No optimizes si... |
|---|---|
| Tienes listas largas con muchos re-renders medibles | El componente es simple y rápido |
| Un cálculo tarda visiblemente | Solo tienes 1–2 props simples |
| Pasas funciones a hijos con `memo` | El valor memoizado se invalida en cada render de todas formas |
| Puedes medir el problema con DevTools | "Puede que sea lento en el futuro" |

> **Regla de oro**: primero mide, luego optimiza.
> Añadir `memo`, `useCallback` y `useMemo` en todo el código tiene
> un costo real — cada hook necesita almacenar el valor anterior y
> comparar dependencias en cada render. La optimización prematura
> puede hacer el código más lento, no más rápido.

---

## Errores comunes con `memo`

```tsx
// ❌ memo no sirve si pasas un objeto literal como prop
// Se crea un objeto nuevo en cada render → memo siempre falla
<UserCard style={{ color: 'red' }} />

// ✅ Extrae el objeto fuera del componente o usa useMemo
const cardStyle = { color: 'red' }  // fuera del componente
<UserCard style={cardStyle} />

// ❌ memo no sirve si pasas una función sin useCallback
<ProductRow onDelete={() => setItems(...)} />

// ✅ useCallback estabiliza la referencia
const handleDelete = useCallback((id: number) => {
  setItems((prev) => prev.filter((i) => i.id !== id))
}, [])
<ProductRow onDelete={handleDelete} />

// ❌ Comparador personalizado que siempre retorna true
const BadMemo = memo(Component, () => true)
// El componente nunca se actualiza aunque las props cambien

// ❌ Comparador que no compara todos los campos
const PartialMemo = memo(Component, (prev, next) =>
  prev.name === next.name  // ignora email, role — bug silencioso
)
```

---

## Ejercicios propuestos

1. **Medir antes de optimizar** — crea una lista de 50 `UserCard` en un componente
   con un input de búsqueda. Añade `RenderCounter` a cada `UserCard`.
   Verifica cuántos re-renders ocurren al escribir en el input.
   Luego aplica `memo` + `useCallback` y compara.

2. **InfiniteList** — crea una lista que empieza con 20 items y tiene un botón
   "Cargar más" que añade 20 más. Usa `memo` en el componente de cada fila
   para que los items ya cargados no se re-rendericen al cargar más.

3. **LazyDashboard** — crea un dashboard con 3 pestañas pesadas. Cada pestaña
   es un componente separado cargado con `lazy`. Solo se descarga el componente
   de la pestaña que el usuario activa.

---

## Resumen de la página 11

- React re-renderiza un componente cuando cambia su estado, props, contexto o cuando su padre re-renderiza.
- `React.memo` evita re-renders del hijo cuando sus props no cambian — usa comparación `===` por defecto.
- `memo` sin `useCallback` es inútil cuando se pasan funciones como props — las funciones son objetos y crean referencias nuevas en cada render.
- El segundo argumento de `memo` permite un comparador personalizado — útil para arrays y objetos en props.
- `useMemo` estabiliza arrays y objetos calculados que se pasan como props a componentes con `memo`.
- `useRef` para contar renders es la forma más rápida de detectar re-renders innecesarios en desarrollo.
- `lazy` + `Suspense` divide el bundle por rutas — el código de cada página se descarga solo cuando el usuario navega a ella.
- `lazy` requiere `export default`. Para named exports usa `.then((m) => ({ default: m.Component }))`.
- Regla de oro: mide primero, optimiza después. La memoización tiene un costo propio.

---

> **Siguiente página →** Formularios y validación: formularios controlados,
> tipado de eventos, validación manual y con Zod.