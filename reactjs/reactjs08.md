# Curso React.js + TypeScript — Página 8
## Módulo 3 · Hooks nativos
### `useContext`: estado global sin prop drilling

---

## El problema: prop drilling

Cuando un dato necesita llegar a un componente profundo en el árbol,
termina pasando por componentes intermedios que no lo usan — solo lo
retransmiten. Eso se llama **prop drilling**:

```
App                     ← tiene: user
 └── Dashboard          ← recibe user (no lo usa, solo lo pasa)
      └── Sidebar       ← recibe user (no lo usa, solo lo pasa)
           └── UserBadge ← usa user aquí
```

```tsx
// Cada nivel recibe y pasa la prop aunque no la necesite
function Dashboard({ user }: { user: User }) {
  return <Sidebar user={user} />
}

function Sidebar({ user }: { user: User }) {
  return <UserBadge user={user} />
}

function UserBadge({ user }: { user: User }) {
  return <p>{user.name}</p>  // solo aquí se usa
}
```

`useContext` resuelve esto: el dato se pone en un contexto y cualquier
componente del árbol puede leerlo directamente sin que los intermedios lo conozcan.

---

## Anatomía de un contexto

Crear y usar un contexto tiene tres pasos:

```tsx
import { createContext, useContext } from 'react'

// 1. Crear el contexto con su tipo
const MyContext = createContext<TipoDelValor | null>(null)

// 2. Proveer el valor en el árbol
function MyProvider({ children }: { children: React.ReactNode }) {
  const value = { /* datos y funciones */ }
  return (
    <MyContext value={value}>   {/* React 19 — sin .Provider */}
      {children}
    </MyContext>
  )
}

// 3. Consumir el valor en cualquier componente descendiente
function MyComponent() {
  const context = useContext(MyContext)
  // usar context aquí
}
```

### React 19 — sintaxis nueva del Provider

En React 19 ya no necesitas `.Provider`. Puedes usar el contexto
directamente como componente:

```tsx
// React 18 y anterior — todavía válido
<ThemeContext.Provider value={value}>
  {children}
</ThemeContext.Provider>

// React 19 — sintaxis simplificada (recomendada)
<ThemeContext value={value}>
  {children}
</ThemeContext>
```

Ambas sintaxis compilan sin errores. El curso usa la nueva.

---

## Tipado correcto con TypeScript

El patrón recomendado es inicializar el contexto con `null` y lanzar
un error si se usa fuera del Provider:

```tsx
// ✅ Patrón correcto — null como valor inicial + guard en el hook
const UserContext = createContext<UserContextValue | null>(null)

export function useUser(): UserContextValue {
  const context = useContext(UserContext)
  if (!context) {
    throw new Error('useUser debe usarse dentro de <UserProvider>')
  }
  return context  // TypeScript sabe que ya no es null aquí
}

// ❌ Evitar — valor por defecto falso da falsa seguridad
const UserContext = createContext<UserContextValue>({} as UserContextValue)
// TypeScript no detectará si se usa fuera del Provider
```

El guard `if (!context) throw new Error(...)` hace dos cosas:
- En desarrollo: mensaje de error claro si olvidas envolver el árbol
- En TypeScript: estrecha el tipo de `null | Value` a `Value`

---

## Estructura de archivos

Los contextos van en su propia carpeta `src/contexts/`,
en archivos `.tsx` porque contienen JSX del Provider:

```
src/
├── contexts/
│   ├── ThemeContext.tsx
│   └── AuthContext.tsx
├── components/
└── main.tsx
```

---

## `src/contexts/ThemeContext.tsx`

Contexto simple con un único estado y una función. Demuestra
el patrón base completo: crear, proveer y consumir.

```tsx
// src/contexts/ThemeContext.tsx

import { createContext, useContext, useState } from 'react'

type Theme = 'light' | 'dark'

interface ThemeContextValue {
  theme:       Theme
  toggleTheme: () => void
}

// null como valor inicial — el guard en el hook lo protege
const ThemeContext = createContext<ThemeContextValue | null>(null)

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light')

  function toggleTheme() {
    setTheme((prev) => (prev === 'light' ? 'dark' : 'light'))
  }

  return (
    <ThemeContext value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext>
  )
}

// Hook personalizado que encapsula el useContext + guard
export function useTheme(): ThemeContextValue {
  const context = useContext(ThemeContext)
  if (!context) throw new Error('useTheme debe usarse dentro de <ThemeProvider>')
  return context
}
```

### Prueba esto

- Cambia el estado inicial de `useState<Theme>('light')` a `'dark'` — la aplicación arranca directamente en modo oscuro sin ningún clic
- Llama a `useTheme()` en un componente fuera del `<ThemeProvider>` — React lanza `Error: useTheme debe usarse dentro de <ThemeProvider>`; el guard hace el error legible en lugar de un crash críptico
- Extiende `Theme` a `'light' | 'dark' | 'system'` — TypeScript exige actualizar `toggleTheme` para manejar el tercer valor; el compilador no deja ignorar el caso nuevo
- Añade un campo `fontSize: 'sm' | 'md' | 'lg'` a `ThemeContextValue` y actualiza el Provider — todos los consumidores de `useTheme()` pueden acceder al tamaño de fuente sin recibir ninguna prop
- Abre las DevTools de React y observa el árbol de contextos — el valor de `ThemeContext` se actualiza en tiempo real cada vez que pulsas el botón de toggle
- Mueve `<ThemeProvider>` para que envuelva solo un subárbol en lugar de toda la app — comprueba que los componentes fuera de ese subárbol reciben el error del guard

---

## `src/contexts/AuthContext.tsx`

Contexto con `useReducer` — el patrón más robusto para autenticación.
El reducer gestiona las transiciones, el contexto las distribuye.

```tsx
// src/contexts/AuthContext.tsx

import { createContext, useContext, useReducer } from 'react'

interface User {
  id:    number
  name:  string
  email: string
  role:  'admin' | 'user'
}

interface AuthState {
  user:      User | null
  isLoading: boolean
  error:     string | null
}

type AuthAction =
  | { type: 'LOGIN_START' }
  | { type: 'LOGIN_SUCCESS'; user: User }
  | { type: 'LOGIN_ERROR';   message: string }
  | { type: 'LOGOUT' }

function authReducer(state: AuthState, action: AuthAction): AuthState {
  switch (action.type) {
    case 'LOGIN_START':
      return { ...state, isLoading: true, error: null }
    case 'LOGIN_SUCCESS':
      return { user: action.user, isLoading: false, error: null }
    case 'LOGIN_ERROR':
      return { user: null, isLoading: false, error: action.message }
    case 'LOGOUT':
      return { user: null, isLoading: false, error: null }
  }
}

const INITIAL_STATE: AuthState = {
  user:      null,
  isLoading: false,
  error:     null,
}

interface AuthContextValue {
  state:  AuthState
  login:  (email: string, password: string) => Promise<void>
  logout: () => void
}

const AuthContext = createContext<AuthContextValue | null>(null)

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(authReducer, INITIAL_STATE)

  async function login(email: string, _password: string) {
    dispatch({ type: 'LOGIN_START' })

    // Simulación de llamada a API
    await new Promise((resolve) => setTimeout(resolve, 1000))

    if (email === 'error@test.com') {
      dispatch({ type: 'LOGIN_ERROR', message: 'Credenciales incorrectas' })
      return
    }

    dispatch({
      type: 'LOGIN_SUCCESS',
      user: { id: 1, name: 'Ana García', email, role: 'admin' },
    })
  }

  function logout() {
    dispatch({ type: 'LOGOUT' })
  }

  return (
    <AuthContext value={{ state, login, logout }}>
      {children}
    </AuthContext>
  )
}

export function useAuth(): AuthContextValue {
  const context = useContext(AuthContext)
  if (!context) throw new Error('useAuth debe usarse dentro de <AuthProvider>')
  return context
}
```

### Prueba esto

- Inicia sesión con cualquier email que no sea `error@test.com` — el estado pasa de `idle` a `LOGIN_START` (isLoading: true) a `LOGIN_SUCCESS` (user: {...}); abre las DevTools para ver cada transición
- Inicia sesión con `error@test.com` — el reducer despacha `LOGIN_ERROR` y el estado queda `{ user: null, error: 'Credenciales incorrectas' }`; no hay rama `finally` porque el reducer lo gestiona todo
- Añade `| { type: 'UPDATE_ROLE'; role: User['role'] }` al tipo `AuthAction` — TypeScript exige el case en el reducer; si lo omites, la función puede retornar `undefined` y el compilador lo detecta
- Cambia el nombre del usuario en `LOGIN_SUCCESS` de `'Ana García'` a `email.split('@')[0]` — el badge muestra las iniciales del alias del email en lugar del nombre fijo
- Despacha `LOGOUT` inmediatamente después de un login exitoso — el estado vuelve a `INITIAL_STATE`; observa que `isLoading` vuelve a `false` y `error` a `null` en una sola acción
- Observa que `_password` tiene el guion bajo — TypeScript no genera warning de variable no usada gracias a esa convención; en producción real lo usarías para llamar a la API

---

## Componentes que consumen los contextos

### `src/components/ThemeToggle.tsx`

```tsx
// src/components/ThemeToggle.tsx
// No recibe ninguna prop — lee el contexto directamente

import { useTheme } from '../contexts/ThemeContext'

export default function ThemeToggle() {
  const { theme, toggleTheme } = useTheme()

  return (
    <button
      onClick={toggleTheme}
      style={{
        padding: '8px 16px',
        borderRadius: 20,
        border: '1px solid #d1d5db',
        background: theme === 'dark' ? '#1f2937' : '#f9fafb',
        color:      theme === 'dark' ? '#f9fafb' : '#1f2937',
        cursor: 'pointer',
        fontWeight: 500,
        fontSize: 14,
      }}
    >
      {theme === 'light' ? '🌙 Modo oscuro' : '☀️ Modo claro'}
    </button>
  )
}
```

### `src/App.tsx`

```tsx
// src/App.tsx

import { useAuth }  from './contexts/AuthContext'
import ThemeToggle  from './components/ThemeToggle'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  ThemeToggle   — botón que alterna el tema desde el contexto     │
// │  2  UserBadge     — badge de usuario autenticado con logout         │
// │  3  LoginForm     — formulario de login conectado a AuthContext      │
// │  4  AppHeader     — header con dos contextos simultáneos            │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

export default function App() {
  const { state } = useAuth()

  const content =
    PASO === 1 ? <ThemeToggle /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 600, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {PASO === 4 ? content : (
        <>
          {state.user && (
            <p style={{ marginBottom: 16, fontSize: 14, color: '#6b7280' }}>
              Sesión activa: <strong>{state.user.name}</strong>
            </p>
          )}
          {content}
        </>
      )}
    </main>
  )
}
```

### Prueba esto

- Pulsa el botón — el fondo y el texto del botón cambian de claro a oscuro; nota que el componente no tiene estado propio, solo lee el contexto con `useTheme()`
- Añade un segundo `<ThemeToggle />` en otro lugar de la app — ambos botones reflejan el mismo tema y al pulsar cualquiera cambia los dos; es la misma referencia de contexto
- Cambia el texto del botón a un span con un icono SVG en lugar de emoji — el comportamiento de toggle no cambia porque depende solo del valor `theme` del contexto
- Inspecciona el componente con React DevTools — aparece como consumidor del `ThemeContext`; cuando el tema cambia, solo re-renderizan los componentes suscritos a ese contexto
- Coloca `<ThemeToggle />` fuera del `<ThemeProvider>` temporalmente — el guard de `useTheme()` lanza un error claro; muévelo de vuelta para resolverlo
- Añade `transition: 'background 0.3s, color 0.3s'` al estilo del botón — el cambio de colores se anima suavemente sin necesidad de estado adicional

---

### `src/components/UserBadge.tsx`

```tsx
// src/components/UserBadge.tsx

import { useAuth } from '../contexts/AuthContext'

export default function UserBadge() {
  const { state, logout } = useAuth()

  if (!state.user) {
    return (
      <span style={{ fontSize: 13, color: '#9ca3af' }}>
        No autenticado
      </span>
    )
  }

  const initials = state.user.name
    .split(' ')
    .map((w) => w[0])
    .join('')
    .toUpperCase()

  return (
    <div style={{ display: 'flex', alignItems: 'center', gap: 10 }}>
      <div style={{
        width: 34, height: 34, borderRadius: '50%',
        background: '#6366f1', color: '#fff',
        display: 'flex', alignItems: 'center', justifyContent: 'center',
        fontWeight: 700, fontSize: 13,
      }}>
        {initials}
      </div>
      <div>
        <p style={{ margin: 0, fontSize: 14, fontWeight: 600 }}>
          {state.user.name}
        </p>
        <p style={{ margin: 0, fontSize: 12, color: '#9ca3af' }}>
          {state.user.role}
        </p>
      </div>
      <button
        onClick={logout}
        style={{
          marginLeft: 8, padding: '4px 10px',
          background: 'none', border: '1px solid #d1d5db',
          borderRadius: 6, cursor: 'pointer',
          fontSize: 12, color: '#6b7280',
        }}
      >
        Salir
      </button>
    </div>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import UserBadge from './components/UserBadge'
```

```tsx
PASO === 2 ? <UserBadge /> :
```

Cambia `PASO = 2` y guarda.

### Prueba esto

- Observa el badge cuando no hay usuario autenticado — muestra el texto "No autenticado" con color gris; no lanza ningún error gracias al guard `if (!state.user) return ...`
- Inicia sesión y observa las iniciales — se calculan con `.split(' ').map((w) => w[0]).join('').toUpperCase()`; prueba un nombre de tres palabras para ver tres iniciales
- Cambia el color del avatar de `#6366f1` a `#0070f3` — el círculo de iniciales adopta el nuevo color sin tocar la lógica del componente
- Pulsa "Salir" — el badge vuelve inmediatamente al estado "No autenticado" porque `logout` despacha `LOGOUT` al contexto y todos los consumidores se actualizan
- Añade el campo `email` debajo del nombre en el badge — accede a `state.user.email` directamente desde el contexto sin necesidad de pasar props
- Cambia el rol de `'admin'` a `'user'` en el dispatch de `LOGIN_SUCCESS` dentro de `AuthContext` — el badge muestra `user` en lugar de `admin`; el tipo `Role` garantiza que no puedes poner un valor no permitido

---

### `src/components/LoginForm.tsx`

```tsx
// src/components/LoginForm.tsx

import { useState } from 'react'
import { useAuth }  from '../contexts/AuthContext'

export default function LoginForm() {
  const { state, login } = useAuth()
  const [email,    setEmail]    = useState('')
  const [password, setPassword] = useState('')

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    await login(email, password)
  }

  return (
    <form
      onSubmit={handleSubmit}
      style={{ display: 'flex', flexDirection: 'column', gap: 10, maxWidth: 300 }}
    >
      <input
        type="email"
        value={email}
        onChange={(e) => setEmail(e.target.value)}
        placeholder="Correo electrónico"
        disabled={state.isLoading}
        style={inputStyle}
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Contraseña"
        disabled={state.isLoading}
        style={inputStyle}
      />

      {state.error && (
        <p style={{ margin: 0, fontSize: 13, color: '#ef4444' }}>
          {state.error}
        </p>
      )}

      <button
        type="submit"
        disabled={state.isLoading || !email || !password}
        style={{
          padding: '10px',
          background: state.isLoading ? '#93c5fd' : '#0070f3',
          color: '#fff', border: 'none', borderRadius: 6,
          cursor: state.isLoading ? 'not-allowed' : 'pointer',
          fontWeight: 500,
        }}
      >
        {state.isLoading ? 'Entrando...' : 'Iniciar sesión'}
      </button>

      <p style={{ margin: 0, fontSize: 12, color: '#9ca3af' }}>
        Prueba con error@test.com para ver el manejo de errores
      </p>
    </form>
  )
}

const inputStyle = {
  padding: '8px 12px',
  border: '1px solid #d1d5db',
  borderRadius: 6,
  fontSize: 14,
}
```

### Agrega a `src/App.tsx`

```tsx
import LoginForm from './components/LoginForm'
```

```tsx
PASO === 3 ? <LoginForm /> :
```

Cambia `PASO = 3` y guarda.

### Prueba esto

- Deja el campo email vacío e intenta pulsar el botón — está deshabilitado porque `!email || !password`; el botón solo se activa cuando ambos tienen contenido
- Escribe cualquier email y contraseña válidos y envía — el botón cambia a "Entrando..." durante 1 s y luego el contexto actualiza el estado a `LOGIN_SUCCESS`
- Escribe `error@test.com` como email — el formulario muestra el mensaje de error que viene del contexto (`state.error`); observa que el error lo gestiona el reducer, no el estado local del formulario
- Cambia la simulación de 1000 ms a 3000 ms en `AuthContext` — el estado `isLoading` permanece visible más tiempo; los inputs quedan deshabilitados durante todo ese periodo
- Añade un `console.log(email, password)` dentro de `handleSubmit` antes de llamar a `login` — observa en la consola que los valores locales del formulario están disponibles sin salir del componente
- Comprueba que `password` tiene el tipo `string` aunque el input sea `type="password"` — el tipo del input solo afecta la visualización, el valor siempre es texto plano

---

### `src/components/AppHeader.tsx`

Componente que usa **dos contextos** al mismo tiempo.
Demuestra que múltiples contextos conviven sin conflicto.

```tsx
// src/components/AppHeader.tsx

import { useTheme } from '../contexts/ThemeContext'
import { useAuth }  from '../contexts/AuthContext'
import ThemeToggle  from './ThemeToggle'
import UserBadge    from './UserBadge'

export default function AppHeader() {
  const { theme }        = useTheme()
  const { state: auth }  = useAuth()

  return (
    <header style={{
      display: 'flex', justifyContent: 'space-between', alignItems: 'center',
      padding: '12px 24px',
      background: theme === 'dark' ? '#111827' : '#fff',
      borderBottom: '1px solid #e5e7eb',
    }}>
      <div>
        <h1 style={{ margin: 0, fontSize: 18, fontWeight: 700 }}>
          Mi App
        </h1>
        {auth.user && (
          <p style={{ margin: 0, fontSize: 12, color: '#9ca3af' }}>
            Panel de {auth.user.role}
          </p>
        )}
      </div>

      <div style={{ display: 'flex', alignItems: 'center', gap: 16 }}>
        <ThemeToggle />
        <UserBadge />
      </div>
    </header>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import AppHeader from './components/AppHeader'
```

```tsx
PASO === 4 ? <AppHeader /> :
```

Cambia `PASO = 4` y guarda. Desde aquí puedes volver a cualquier paso anterior cambiando la constante.

### Prueba esto

- Pulsa el toggle de tema — el fondo del header cambia de blanco a `#111827`; observa que `AppHeader` consume dos contextos independientes y ambos afectan la vista
- Inicia sesión y observa el subtítulo "Panel de admin" — aparece debajo del título solo cuando `auth.user` es truthy; cambia `auth.user.role` a `'user'` para ver el texto diferente
- Inspecciona el componente con React DevTools — verás que está suscrito tanto a `ThemeContext` como a `AuthContext`; cuando cualquiera de los dos cambia, el header re-renderiza
- Mueve `<ThemeToggle />` y `<UserBadge />` a la derecha del header y añade una barra de navegación en el centro — ningún cambio de layout requiere tocar los contextos
- Añade `color: theme === 'dark' ? '#f9fafb' : '#111827'` al estilo del `<h1>` — el texto también cambia de color con el tema, completando el modo oscuro del header
- Comenta la línea `<UserBadge />` — el header sigue funcionando y el ThemeToggle sigue operativo; los contextos son independientes y no se afectan entre sí

---

### `src/main.tsx`

Los Providers se anidan en `main.tsx` — envuelven toda la aplicación
para que cualquier componente pueda acceder a los contextos.

```tsx
// src/main.tsx

import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import App from './App.tsx'
import { ThemeProvider } from './contexts/ThemeContext.tsx'
import { AuthProvider }  from './contexts/AuthContext.tsx'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <ThemeProvider>
      <AuthProvider>
        <App />
      </AuthProvider>
    </ThemeProvider>
  </StrictMode>,
)
```

> El orden de los Providers importa cuando un Provider necesita
> datos de otro contexto. En este caso son independientes, así que
> el orden no afecta el resultado.

---

## El flujo completo sin prop drilling

```
main.tsx
 └── ThemeProvider     ← provee: theme, toggleTheme
      └── AuthProvider ← provee: state, login, logout
           └── App
                ├── AppHeader
                │    ├── ThemeToggle  ← usa useTheme() directamente
                │    └── UserBadge   ← usa useAuth() directamente
                └── LoginForm        ← usa useAuth() directamente
```

Ningún componente intermedio recibe props que no necesita.
Cada componente toma exactamente lo que necesita del contexto correspondiente.

---

## Cuándo usar Context — y cuándo no

| Usa Context para... | No uses Context para... |
|---|---|
| Tema visual (light/dark) | Estado local de un componente |
| Usuario autenticado | Datos que solo necesitan 1–2 niveles de profundidad |
| Idioma / localización | Estado que cambia muy frecuentemente |
| Configuración global | Datos del servidor — usa React Query o SWR |

> Context re-renderiza **todos** los componentes que lo consumen
> cada vez que el valor cambia. Para estado que cambia frecuentemente
> (cada pulsación de tecla, cada frame de animación), Context no es
> la herramienta correcta — puede causar problemas de rendimiento.

---

## Ejercicios propuestos

1. **`LanguageContext`** — contexto de idioma con `'es' | 'en' | 'fr'`.
   Crea un hook `useTranslation()` que retorne una función `t(key: string)`
   que busca la traducción en un objeto de strings por idioma.

2. **`NotificationsContext`** — gestiona una lista de notificaciones globales.
   Acciones: `ADD_NOTIFICATION`, `REMOVE_NOTIFICATION`, `CLEAR_ALL`.
   Cada notificación tiene `id`, `message` y `type: 'info' | 'success' | 'error'`.

3. **`CartContext`** — extrae el carrito de la página 7 a un contexto.
   El reducer y las acciones quedan iguales, pero ahora cualquier componente
   puede acceder al carrito con `useCart()` sin recibir props.

---

## Resumen de la página 8

- `useContext` resuelve el prop drilling — datos accesibles en cualquier nivel del árbol sin pasar props por componentes intermedios.
- En React 19 el Provider se escribe como `<MiContexto value={...}>` sin `.Provider`.
- El patrón correcto: `createContext<Tipo | null>(null)` + hook personalizado con guard `if (!context) throw new Error(...)`.
- El guard estrecha el tipo de `null | Valor` a `Valor` — TypeScript lo garantiza.
- Los Providers se anidan en `main.tsx` para envolver toda la aplicación.
- Context + `useReducer` es el patrón más robusto para estado global complejo.
- Context no es adecuado para estado que cambia frecuentemente — puede generar re-renders en todo el árbol.

---

> **Siguiente página →** Hooks personalizados: ahora que conoces todos los hooks
> nativos, aprenderás a combinarlos en hooks reutilizables con tipado genérico.