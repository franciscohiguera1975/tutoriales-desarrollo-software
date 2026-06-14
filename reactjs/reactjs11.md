# Curso React.js + TypeScript — Página 10
## Módulo 5 · Arquitectura
### React Router v7: navegación, rutas anidadas y rutas protegidas

---

## ¿Qué es React Router?

React Router es la librería estándar para manejar navegación en aplicaciones
React. Permite mostrar distintos componentes según la URL sin recargar la página
— comportamiento de **Single Page Application (SPA)**.

La versión instalada con `npm install react-router-dom` hoy es la **v7**,
que mantiene la misma API de componentes de v6 y añade soporte para
data loading y server rendering. En este módulo cubrimos la API
de componentes — la más común en proyectos SPA.

### Instalación

```bash
npm install react-router-dom
```

No necesitas instalar tipos por separado — `react-router-dom` v7
incluye sus propias definiciones TypeScript.

---

## Estructura de archivos

```
src/
├── pages/              ← un archivo por página/vista
│   ├── HomePage.tsx
│   ├── AboutPage.tsx
│   ├── ProductsPage.tsx
│   ├── ProductDetailPage.tsx
│   ├── DashboardPage.tsx
│   ├── LoginPage.tsx
│   └── NotFoundPage.tsx
├── layouts/            ← componentes con <Outlet>
│   ├── RootLayout.tsx
│   └── DashboardLayout.tsx
├── components/
└── main.tsx
```

---

## Configuración inicial en `main.tsx`

```tsx
// src/main.tsx

import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { BrowserRouter } from 'react-router-dom'
import './index.css'
import App from './App.tsx'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>,
)
```

`BrowserRouter` envuelve toda la aplicación en `main.tsx` —
no en `App.tsx` — para que todos los componentes puedan acceder
a los hooks del router.

### Prueba esto

- Mueve `<BrowserRouter>` de `main.tsx` al interior de `App()` y llama a `useNavigate()` en cualquier componente hijo — observa el error `useNavigate() may be used only in the context of a <Router> component`
- Elimina `<BrowserRouter>` por completo, guarda y navega a `/about` — observa que la URL cambia pero `<Routes>` no renderiza nada o lanza un error de contexto faltante
- Añade `<React.StrictMode>` si aún no está y observa en la consola que los efectos se ejecutan dos veces en desarrollo — comportamiento esperado y deliberado
- Cambia `BrowserRouter` por `HashRouter` (importado de `react-router-dom`) y observa que la URL pasa de `localhost:5173/about` a `localhost:5173/#/about`
- Abre las DevTools de red, recarga la página y observa que solo se descarga un archivo HTML — confirmando que React Router maneja la navegación sin peticiones al servidor

---

## Rutas básicas con `Routes` y `Route`

```tsx
// src/App.tsx

import { Routes, Route } from 'react-router-dom'
import HomePage     from './pages/HomePage'
import AboutPage    from './pages/AboutPage'
import NotFoundPage from './pages/NotFoundPage'

export default function App() {
  return (
    <Routes>
      <Route path="/"       element={<HomePage />} />
      <Route path="/about"  element={<AboutPage />} />
      <Route path="*"       element={<NotFoundPage />} />
    </Routes>
  )
}
```

- `path="/"` — coincide exactamente con la raíz.
- `path="*"` — captura cualquier ruta no definida arriba — siempre al final.
- `element` — el componente que se renderiza cuando la URL coincide.

### Prueba esto

- Escribe `/about` directamente en la barra de URL — observa que `AboutPage` se renderiza sin recargar el HTML del servidor
- Escribe `/ruta-inexistente` en la URL — observa que `NotFoundPage` se renderiza porque `path="*"` captura todo lo que no coincide
- Mueve `<Route path="*">` al principio del árbol (antes de `/about`) — observa que ahora cualquier URL renderiza `NotFoundPage` porque `*` captura primero
- Elimina `<Route path="*">` y navega a una ruta inexistente — observa que la pantalla queda en blanco porque no hay ninguna ruta que coincida
- Cambia `path="/"` por `path="/home"` y navega a `/` — observa que queda en blanco; navega a `/home` y funciona correctamente

---

## Navegación: `Link` y `NavLink`

### `src/layouts/RootLayout.tsx`

```tsx
// src/layouts/RootLayout.tsx

import { Link, NavLink, Outlet } from 'react-router-dom'

export default function RootLayout() {
  return (
    <div style={{ fontFamily: 'sans-serif' }}>
      <header style={{
        display: 'flex', alignItems: 'center', gap: 24,
        padding: '12px 24px', borderBottom: '1px solid #e5e7eb',
      }}>
        <Link
          to="/"
          style={{ fontWeight: 700, fontSize: 18, textDecoration: 'none', color: '#111' }}
        >
          Mi App
        </Link>

        <nav style={{ display: 'flex', gap: 16 }}>
          {[
            { to: '/',        label: 'Inicio'    },
            { to: '/products', label: 'Productos' },
            { to: '/about',   label: 'Acerca de' },
          ].map(({ to, label }) => (
            <NavLink
              key={to}
              to={to}
              end={to === '/'}  // evita que "/" quede activo en todas las rutas
              style={({ isActive }) => ({
                textDecoration: 'none',
                fontWeight:   isActive ? 600   : 400,
                color:        isActive ? '#0070f3' : '#6b7280',
                borderBottom: isActive ? '2px solid #0070f3' : '2px solid transparent',
                paddingBottom: 4,
              })}
            >
              {label}
            </NavLink>
          ))}
        </nav>
      </header>

      <main style={{ maxWidth: 720, margin: '32px auto', padding: '0 16px' }}>
        <Outlet />  {/* aquí se renderiza la página activa */}
      </main>
    </div>
  )
}
```

| Componente | Diferencia |
|---|---|
| `<Link to="...">` | Navegación simple — no aplica estilos según la URL activa |
| `<NavLink to="...">` | Igual pero recibe `{ isActive }` en `style` y `className` |

> `end` en `NavLink` hace que solo esté activo en la ruta exacta,
> no en rutas hijas. Imprescindible para el enlace raíz `"/"`.

### Prueba esto

- Quita `end` del `NavLink` de `"/"` y navega a `/products` — observa que el enlace "Inicio" sigue resaltado porque `/` es prefijo de todas las rutas
- Reemplaza un `<NavLink>` por `<a href="/about">` y haz clic — observa en la pestaña de Red de DevTools que el navegador hace una petición GET completa al servidor, recargando la app
- Cambia la función de `style` en `NavLink` para que también cambie el `background` cuando está activo — observa el cambio visual al navegar entre rutas
- Navega a `/products` y luego a `/about` sin recargar — observa en el `Outlet` cómo el contenido cambia pero el `header` permanece igual, demostrando que el layout es compartido
- Abre las DevTools de React y selecciona el componente `RootLayout` — observa que no se desmonta al navegar, solo cambia lo que está dentro de `<Outlet />`

---

## `useParams` — parámetros de URL tipados

```tsx
// src/pages/ProductDetailPage.tsx

import { useParams, Link } from 'react-router-dom'

// Define el tipo de los parámetros de la URL
interface ProductParams {
  id: string   // los params siempre son string — convierte si necesitas número
}

export default function ProductDetailPage() {
  const { id } = useParams<ProductParams>()

  // Convierte a número cuando lo necesites
  const productId = Number(id)

  if (!id || isNaN(productId)) {
    return <p style={{ color: '#ef4444' }}>ID de producto inválido.</p>
  }

  return (
    <div>
      <Link
        to="/products"
        style={{ fontSize: 13, color: '#6b7280', textDecoration: 'none' }}
      >
        ← Volver a productos
      </Link>
      <h1 style={{ marginTop: 12 }}>Producto #{productId}</h1>
      <p style={{ color: '#6b7280' }}>
        Aquí iría el detalle del producto con ID {productId}.
      </p>
    </div>
  )
}
```

```tsx
// Ruta que define el parámetro
<Route path="/products/:id" element={<ProductDetailPage />} />

// Link que navega al detalle
<Link to={`/products/${product.id}`}>Ver detalle</Link>
```

### Prueba esto

- Navega a `/products/abc` — observa que `Number('abc')` produce `NaN` y se muestra el mensaje de ID inválido
- Navega a `/products/3` — observa que `id` es el string `"3"` (no el número `3`); añade un `console.log(typeof id)` para confirmarlo
- Elimina la comprobación `isNaN(productId)` y pasa un ID texto — observa que el componente renderiza `Producto #NaN` sin ningún error visible, un bug silencioso
- Añade un segundo parámetro a la ruta cambiando `path` a `/products/:id/:slug` y observa en `useParams` que ambos parámetros están disponibles como strings
- Cambia el `<Link to="/products">` del botón de volver por `navigate(-1)` y navega desde distintas páginas — observa que siempre vuelve a la página anterior en el historial, no siempre a `/products`

---

## `useNavigate` — navegación programática

```tsx
// src/pages/LoginPage.tsx

import { useState }     from 'react'
import { useNavigate }  from 'react-router-dom'

export default function LoginPage() {
  const navigate = useNavigate()
  const [email,    setEmail]    = useState('')
  const [password, setPassword] = useState('')
  const [loading,  setLoading]  = useState(false)

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    setLoading(true)

    // Simulación de login
    await new Promise((r) => setTimeout(r, 800))

    // replace: true — reemplaza la entrada en el historial
    // el usuario no puede volver al login con el botón "atrás"
    navigate('/dashboard', { replace: true })
  }

  return (
    <div style={{ maxWidth: 320, margin: '0 auto' }}>
      <h1 style={{ fontSize: 22, marginBottom: 20 }}>Iniciar sesión</h1>
      <form
        onSubmit={handleSubmit}
        style={{ display: 'flex', flexDirection: 'column', gap: 12 }}
      >
        <input
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          placeholder="Email"
          required
          style={inputStyle}
        />
        <input
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          placeholder="Contraseña"
          required
          style={inputStyle}
        />
        <button
          type="submit"
          disabled={loading}
          style={{
            padding: '10px', background: loading ? '#93c5fd' : '#0070f3',
            color: '#fff', border: 'none', borderRadius: 6,
            cursor: loading ? 'not-allowed' : 'pointer', fontWeight: 500,
          }}
        >
          {loading ? 'Entrando...' : 'Entrar'}
        </button>
      </form>
    </div>
  )
}

const inputStyle = {
  padding: '8px 12px',
  border: '1px solid #d1d5db',
  borderRadius: 6, fontSize: 14,
}
```

### Métodos de `useNavigate`

```tsx
const navigate = useNavigate()

navigate('/ruta')                   // navegar hacia adelante
navigate('/ruta', { replace: true }) // reemplaza — sin entrada en historial
navigate(-1)                         // volver atrás (como botón del navegador)
navigate(1)                          // avanzar
navigate('/ruta', { state: { from: 'login' } }) // pasar estado al destino
```

### Prueba esto

- Completa el formulario de login y haz clic en "Entrar" — observa cómo después del `await` de 800 ms la URL cambia a `/dashboard` automáticamente sin que el usuario haga clic en ningún enlace
- Llega al dashboard via login con `replace: true` y presiona el botón "Atrás" del navegador — observa que no vuelve al formulario de login sino a la página anterior al login
- Cambia `replace: true` por `replace: false` (o quita la opción) y repite — ahora el botón "Atrás" sí vuelve al login
- Añade `navigate(-1)` en un botón de "Cancelar" en el formulario — observa que navega a la página anterior en el historial del navegador
- Pasa `state: { username: email }` en el `navigate` y en `/dashboard` usa `useLocation().state` para mostrar "Bienvenido, ana@ejemplo.com" — observa cómo el estado viaja entre rutas sin query params

---

## `useLocation` y `useSearchParams`

```tsx
// src/pages/ProductsPage.tsx

import { useState, useMemo }  from 'react'
import { Link, useSearchParams } from 'react-router-dom'

interface Product {
  id:       number
  name:     string
  category: string
  price:    number
}

const PRODUCTS: Product[] = [
  { id: 1, name: 'Teclado mecánico',  category: 'periféricos',  price: 89  },
  { id: 2, name: 'Monitor 27"',       category: 'pantallas',    price: 349 },
  { id: 3, name: 'Mouse inalámbrico', category: 'periféricos',  price: 29  },
  { id: 4, name: 'Webcam HD',         category: 'cámaras',      price: 59  },
  { id: 5, name: 'Auriculares BT',    category: 'audio',        price: 149 },
]

export default function ProductsPage() {
  // useSearchParams sincroniza filtros con la URL
  // ?q=teclado&category=periféricos queda en la barra del navegador
  const [searchParams, setSearchParams] = useSearchParams()

  const query    = searchParams.get('q')        ?? ''
  const category = searchParams.get('category') ?? ''

  function handleQueryChange(value: string) {
    setSearchParams(
      (prev) => { prev.set('q', value); return prev },
      { replace: true }
    )
  }

  function handleCategoryChange(value: string) {
    setSearchParams(
      (prev) => {
        if (value) prev.set('category', value)
        else       prev.delete('category')
        return prev
      },
      { replace: true }
    )
  }

  const filtered = useMemo(() =>
    PRODUCTS
      .filter((p) => p.name.toLowerCase().includes(query.toLowerCase()))
      .filter((p) => !category || p.category === category),
    [query, category]
  )

  const categories = [...new Set(PRODUCTS.map((p) => p.category))]

  return (
    <div>
      <h1 style={{ fontSize: 22, marginBottom: 16 }}>Productos</h1>

      <div style={{ display: 'flex', gap: 8, marginBottom: 16 }}>
        <input
          value={query}
          onChange={(e) => handleQueryChange(e.target.value)}
          placeholder="Buscar..."
          style={{ flex: 1, padding: '8px 12px', border: '1px solid #d1d5db', borderRadius: 6 }}
        />
        <select
          value={category}
          onChange={(e) => handleCategoryChange(e.target.value)}
          style={{ padding: '8px 12px', border: '1px solid #d1d5db', borderRadius: 6 }}
        >
          <option value="">Todas las categorías</option>
          {categories.map((cat) => (
            <option key={cat} value={cat}>{cat}</option>
          ))}
        </select>
      </div>

      <div style={{ display: 'flex', flexDirection: 'column', gap: 8 }}>
        {filtered.map((product) => (
          <Link
            key={product.id}
            to={`/products/${product.id}`}
            style={{ textDecoration: 'none', color: 'inherit' }}
          >
            <div style={{
              display: 'flex', justifyContent: 'space-between', alignItems: 'center',
              padding: '12px 16px', border: '1px solid #e5e7eb', borderRadius: 8,
            }}>
              <div>
                <p style={{ margin: 0, fontWeight: 500 }}>{product.name}</p>
                <p style={{ margin: 0, fontSize: 12, color: '#9ca3af' }}>{product.category}</p>
              </div>
              <span style={{ fontWeight: 600 }}>${product.price}</span>
            </div>
          </Link>
        ))}
        {filtered.length === 0 && (
          <p style={{ color: '#9ca3af' }}>Sin resultados.</p>
        )}
      </div>
    </div>
  )
}
```

### Prueba esto

- Escribe "teclado" en el input y observa cómo la URL cambia a `?q=teclado` — copia esa URL, abre una nueva pestaña y pégala — el filtro se restaura automáticamente
- Filtra por categoría "periféricos" y luego recarga la página — observa que el filtro sigue activo porque está en la URL (`?category=periféricos`), no en estado de React
- Elimina `{ replace: true }` de `setSearchParams` y teclea varias letras en el buscador — luego presiona "Atrás" del navegador y observa que retrocede letra a letra en el historial
- Combina ambos filtros: escribe "mouse" y selecciona "periféricos" — observa en la URL que los dos params coexisten: `?q=mouse&category=periféricos`
- Llama a `setSearchParams` directamente con `new URLSearchParams()` vacío para limpiar todos los filtros y observa que la lista vuelve a mostrar todos los productos

---

## Rutas anidadas con `Outlet`

```tsx
// src/layouts/DashboardLayout.tsx

import { NavLink, Outlet } from 'react-router-dom'

const NAV_ITEMS = [
  { to: '',           label: 'Resumen'       },
  { to: 'analytics',  label: 'Analítica'     },
  { to: 'settings',   label: 'Configuración' },
]

export default function DashboardLayout() {
  return (
    <div style={{ display: 'grid', gridTemplateColumns: '200px 1fr', gap: 24 }}>
      <aside style={{ borderRight: '1px solid #e5e7eb', paddingRight: 16 }}>
        <p style={{ fontSize: 12, fontWeight: 600, color: '#9ca3af', marginBottom: 8 }}>
          DASHBOARD
        </p>
        <nav style={{ display: 'flex', flexDirection: 'column', gap: 4 }}>
          {NAV_ITEMS.map(({ to, label }) => (
            <NavLink
              key={label}
              to={to}
              end
              style={({ isActive }) => ({
                padding: '6px 10px', borderRadius: 6,
                textDecoration: 'none', fontSize: 14,
                background: isActive ? '#eff6ff' : 'transparent',
                color:      isActive ? '#1d4ed8' : '#374151',
                fontWeight: isActive ? 600 : 400,
              })}
            >
              {label}
            </NavLink>
          ))}
        </nav>
      </aside>

      <section>
        <Outlet />  {/* renderiza la sub-ruta activa aquí */}
      </section>
    </div>
  )
}
```

### Prueba esto

- Navega a `/dashboard` — observa que el `Outlet` renderiza `<Overview>` (la sub-ruta `index`) y el sidebar muestra "Resumen" resaltado
- Navega a `/dashboard/analytics` — observa que el sidebar actualiza el resaltado a "Analítica" sin recargar la página ni desmontar `DashboardLayout`
- Abre las DevTools de React, selecciona `DashboardLayout` y navega entre sub-rutas — confirma que `DashboardLayout` nunca se desmonta; solo cambia el contenido del `Outlet`
- Quita `end` de los `NavLink` del sidebar y navega a `/dashboard/analytics` — observa que "Resumen" (ruta vacía `""`) también aparece resaltado al mismo tiempo
- Añade una nueva entrada `{ to: 'team', label: 'Equipo' }` al array `NAV_ITEMS` sin crear la ruta en `App.tsx` — navega a ella y observa que `NotFoundPage` se renderiza dentro del `Outlet` del dashboard

```tsx
// src/App.tsx — rutas anidadas
import { Routes, Route } from 'react-router-dom'
import RootLayout       from './layouts/RootLayout'
import DashboardLayout  from './layouts/DashboardLayout'
import HomePage         from './pages/HomePage'
import ProductsPage     from './pages/ProductsPage'
import ProductDetailPage from './pages/ProductDetailPage'
import AboutPage        from './pages/AboutPage'
import LoginPage        from './pages/LoginPage'
import NotFoundPage     from './pages/NotFoundPage'

// Páginas del dashboard (simples por ahora)
function Overview()   { return <h2>Resumen</h2> }
function Analytics()  { return <h2>Analítica</h2> }
function SettingsPage(){ return <h2>Configuración</h2> }

export default function App() {
  return (
    <Routes>
      {/* Layout raíz — todas las páginas comparten header */}
      <Route element={<RootLayout />}>
        <Route index          element={<HomePage />} />
        <Route path="products" element={<ProductsPage />} />
        <Route path="products/:id" element={<ProductDetailPage />} />
        <Route path="about"   element={<AboutPage />} />
        <Route path="login"   element={<LoginPage />} />

        {/* Dashboard con su propio layout anidado */}
        <Route path="dashboard" element={<DashboardLayout />}>
          <Route index          element={<Overview />} />
          <Route path="analytics"  element={<Analytics />} />
          <Route path="settings"   element={<SettingsPage />} />
        </Route>

        <Route path="*" element={<NotFoundPage />} />
      </Route>
    </Routes>
  )
}
```

El árbol de rutas anidadas genera este comportamiento:

```
URL: /                    → RootLayout > HomePage
URL: /products            → RootLayout > ProductsPage
URL: /products/3          → RootLayout > ProductDetailPage (id="3")
URL: /dashboard           → RootLayout > DashboardLayout > Overview
URL: /dashboard/analytics → RootLayout > DashboardLayout > Analytics
```

### Prueba esto

- Navega a `/products/3` directamente en la barra de URL — observa que `RootLayout` (con el header) se mantiene y solo cambia el contenido principal
- Cambia `<Route index element={<Overview />} />` por `<Route path="" element={<Overview />} />` y verifica que el comportamiento es idéntico — `index` es azúcar sintáctico para `path=""`
- Añade `console.log('mount DashboardLayout')` en un `useEffect(() => { ... }, [])` dentro de `DashboardLayout` y navega entre `/dashboard`, `/dashboard/analytics` y `/dashboard/settings` — observa que el log solo aparece una vez, al entrar al dashboard por primera vez
- Navega a `/dashboard/rutaghosts` y observa que `NotFoundPage` se renderiza en el lugar del `Outlet` de `RootLayout`, no dentro de `DashboardLayout`
- Mueve la ruta `path="*"` dentro del bloque de `dashboard` — observa que ahora `/dashboard/inexistente` muestra `NotFoundPage` en el panel derecho del dashboard

---

## Rutas protegidas

```tsx
// src/components/ProtectedRoute.tsx

import { Navigate, useLocation } from 'react-router-dom'

interface ProtectedRouteProps {
  isAuthenticated: boolean
  children:        React.ReactNode
}

export default function ProtectedRoute({
  isAuthenticated,
  children,
}: ProtectedRouteProps) {
  const location = useLocation()

  if (!isAuthenticated) {
    // Guarda la ruta actual para redirigir después del login
    return <Navigate to="/login" state={{ from: location.pathname }} replace />
  }

  return <>{children}</>
}
```

```tsx
// Uso en App.tsx
<Route
  path="dashboard"
  element={
    <ProtectedRoute isAuthenticated={isAuthenticated}>
      <DashboardLayout />
    </ProtectedRoute>
  }
>
  <Route index element={<Overview />} />
</Route>
```

```tsx
// En LoginPage — recuperar la ruta de origen y redirigir allí
import { useLocation, useNavigate } from 'react-router-dom'

interface LocationState {
  from?: string
}

function LoginPage() {
  const navigate = useNavigate()
  const location = useLocation()

  const from = (location.state as LocationState)?.from ?? '/dashboard'

  async function handleLogin() {
    // ...autenticación
    navigate(from, { replace: true })  // vuelve a la ruta original
  }
}
```

### Prueba esto

- Pasa `isAuthenticated={false}` a `ProtectedRoute` y navega directamente a `/dashboard` — observa que la URL cambia a `/login` y en `location.state` está la ruta de origen
- Tras el login exitoso con `navigate(from, { replace: true })`, presiona "Atrás" — observa que no vuelves al login sino a la página anterior al intento de entrar al dashboard
- Pasa `isAuthenticated={true}` y navega a `/dashboard` — observa que `ProtectedRoute` renderiza `children` directamente sin ninguna redirección
- Cambia `replace` a `false` en el `<Navigate>` dentro de `ProtectedRoute` y navega a `/dashboard` no autenticado — luego inicia sesión y presiona "Atrás"; observas que vuelves al `/login` (que ya no debería ser accesible)
- Elimina `state={{ from: location.pathname }}` del `<Navigate>` y completa el login — observa que siempre redirige a `/dashboard` (el fallback `??`), perdiendo la ruta de origen
- Añade un `console.log(location.state)` en `LoginPage` para inspeccionar el objeto de estado que llega desde `ProtectedRoute`

---

## `src/main.tsx` completo

```tsx
// src/main.tsx

import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { BrowserRouter } from 'react-router-dom'
import './index.css'
import App from './App.tsx'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>,
)
```

---

## Hooks del router — resumen rápido

| Hook | Retorna | Uso típico |
|---|---|---|
| `useNavigate()` | función `navigate` | Redirigir programáticamente |
| `useParams<T>()` | objeto con params de URL | Leer `:id`, `:slug`, etc. |
| `useLocation()` | objeto `location` | Leer `pathname`, `search`, `state` |
| `useSearchParams()` | `[params, setParams]` | Leer y escribir query strings `?q=...` |

---

## Errores comunes con React Router

```tsx
// ❌ BrowserRouter dentro de App — los hooks no funcionarán
// fuera del árbol de BrowserRouter si lo colocas en App
function App() {
  return (
    <BrowserRouter>   {/* ← muévelo a main.tsx */}
      <Routes>...</Routes>
    </BrowserRouter>
  )
}

// ❌ Usar <a href="..."> en lugar de <Link> — recarga la página
<a href="/about">Acerca de</a>

// ✅ Siempre Link/NavLink para navegación interna
<Link to="/about">Acerca de</Link>

// ❌ Olvidar end en NavLink para "/"
<NavLink to="/">Inicio</NavLink>  // queda activo en /about, /products, etc.

// ✅ end hace que solo esté activo en la ruta exacta
<NavLink to="/" end>Inicio</NavLink>

// ❌ No convertir el param a número cuando se necesita
const { id } = useParams<{ id: string }>()
fetch(`/api/products/${id}`)  // ok — id es string y fetch acepta string

const productId = id   // ❌ si lo usas donde esperan number
const productId = Number(id)  // ✅ conversión explícita
```

---

## Ejercicios propuestos

1. **Breadcrumbs** — crea un componente `Breadcrumbs` que use `useLocation`
   para dividir `pathname` por `/` y mostrar la ruta actual como migas de pan.
   Cada segmento debe ser un `<Link>` a la ruta parcial correspondiente.

2. **Tabs con URL** — crea un componente con 3 pestañas (Perfil, Seguridad, Notificaciones).
   Cada pestaña actualiza un search param `?tab=profile` con `useSearchParams`.
   Al recargar la página, la pestaña activa se preserva.

3. **404 personalizado** — crea una `NotFoundPage` que use `useLocation`
   para mostrar la ruta que no se encontró y un botón que llame a `navigate(-1)`.

---

## Resumen de la página 10

- `BrowserRouter` va en `main.tsx` — envuelve toda la aplicación.
- `Routes` + `Route` definen el árbol de rutas. `path="*"` captura rutas no definidas.
- `Link` navega sin recargar. `NavLink` recibe `{ isActive }` para estilos condicionales.
- `end` en `NavLink` es imprescindible para el enlace raíz `/`.
- `useParams<{ id: string }>()` — los params siempre son `string`, convierte si necesitas otro tipo.
- `useNavigate()` para redirección programática. `replace: true` evita que la ruta anterior quede en el historial.
- `useSearchParams()` sincroniza filtros con la URL — los usuarios pueden compartir o marcar la búsqueda.
- Las rutas anidadas con `<Outlet />` permiten layouts compartidos sin duplicar código.
- `ProtectedRoute` redirige con `<Navigate>` y guarda la ruta de origen en `location.state`.

---

> **Siguiente página →** Rendimiento: `React.memo`, patrones de optimización
> y herramientas de profiling para detectar re-renders innecesarios.