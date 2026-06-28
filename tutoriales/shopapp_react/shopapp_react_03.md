# ShopApp React Web — Módulo 3

## Auth — Login, Registro, AuthStore con Zustand y rutas protegidas

**Duración estimada:** 3–4 horas
**Prerequisitos:** Módulos 1 y 2 completados, backend `shopapi_02` desplegado.

---

> **Objetivo**
> Implementar el flujo completo de autenticación: tipos TypeScript para el dominio de auth,
> funciones de API para login/registro/logout, un store Zustand que gestiona el estado de sesión
> con persistencia en localStorage, páginas de login y registro con React Hook Form + Zod,
> un componente de ruta protegida y un shell de navegación persistente con header.
>
> **Checkpoint final**
> - El usuario puede registrarse en `/register` y es redirigido a `/`.
> - El usuario puede iniciar sesión en `/login` y su sesión persiste al recargar la página.
> - Las rutas `/cart`, `/orders` y `/profile` redirigen a `/login` si el usuario no está autenticado.
> - Las rutas `/admin/*` redirigen a `/` si el usuario no tiene `is_staff = true`.
> - El header muestra el menú de usuario cuando está autenticado y el botón "Iniciar sesión" cuando no lo está.
> - Al expirar la sesión (evento `authExpired`), el store limpia el estado y el usuario es redirigido a `/login`.

---

## 3.1 Auth types

**Archivo:** `src/domain/model/auth.types.ts`

Los tipos del dominio de autenticación describen las entidades que maneja el backend sin acoplar
el modelo a ningún detalle de implementación (ni Axios, ni localStorage, ni Zustand).

```typescript
// src/domain/model/auth.types.ts

/** Usuario autenticado tal como lo devuelve el endpoint /auth/me/ */
export interface LoggedUser {
  user_id: number
  username: string
  email: string
  is_staff: boolean
}

/** Par de tokens JWT devuelto por /auth/login/ y /auth/register/ */
export interface AuthTokens {
  access: string
  refresh: string
}

/** Estado completo del módulo de autenticación en el store Zustand. */
export interface AuthState {
  /** Usuario autenticado, o null si no hay sesión activa. */
  user: LoggedUser | null
  /** Tokens JWT actuales, o null si no hay sesión. */
  tokens: AuthTokens | null
  /** true mientras hay una operación de red en curso (login, register, logout). */
  isLoading: boolean
  /** Mensaje de error del último intento fallido, null si no hay error. */
  error: string | null
}
```

Separar los tipos en `domain/model/` sigue el principio de Clean Architecture: la capa de dominio
no depende de detalles externos. Si en el futuro cambia la forma del backend, solo se actualiza
este archivo y los contratos de las capas superiores.

---

## 3.2 Auth API functions

**Archivo:** `src/data/api/auth.api.ts`

Las funciones de API son la única parte de la app que sabe cómo comunicarse con los endpoints
concretos del backend. Reciben y devuelven tipos del dominio, nunca objetos Axios sin tipar.

```typescript
// src/data/api/auth.api.ts
import { apiClient } from '@/data/api/api.client'
import { parseApiError } from '@/data/api/api.error'
import { localStore } from '@/data/storage/local.storage'
import type { LoggedUser, AuthTokens } from '@/domain/model/auth.types'

/** Respuesta del endpoint /auth/login/ */
interface LoginResponse {
  user: LoggedUser
  tokens: AuthTokens
}

/** Respuesta del endpoint /auth/register/ */
interface RegisterResponse {
  user: LoggedUser
  tokens: AuthTokens
}

/**
 * Autentica al usuario con username y password.
 * POST /auth/login/
 */
export async function loginUser(
  username: string,
  password: string,
): Promise<LoginResponse> {
  try {
    const { data } = await apiClient.post<LoginResponse>('/auth/login/', {
      username,
      password,
    })
    return data
  } catch (err) {
    throw parseApiError(err)
  }
}

/**
 * Registra un nuevo usuario.
 * POST /auth/register/
 */
export async function registerUser(
  username: string,
  email: string,
  password: string,
): Promise<RegisterResponse> {
  try {
    const { data } = await apiClient.post<RegisterResponse>('/auth/register/', {
      username,
      email,
      password,
    })
    return data
  } catch (err) {
    throw parseApiError(err)
  }
}

/**
 * Pide un nuevo access token usando el refresh token.
 * POST /auth/token/refresh/
 * Nota: el api.client ya maneja el refresh automáticamente en el interceptor.
 * Esta función se expone para usos explícitos (p. ej. forzar refresh manual).
 */
export async function refreshToken(refresh: string): Promise<{ access: string }> {
  try {
    const { data } = await apiClient.post<{ access: string }>(
      '/auth/token/refresh/',
      { refresh },
    )
    return data
  } catch (err) {
    throw parseApiError(err)
  }
}

/**
 * Invalida el refresh token en el servidor.
 * POST /auth/logout/
 * Usa el refresh token guardado en localStorage.
 */
export async function logoutUser(): Promise<void> {
  const refresh = localStore.getRefreshToken()
  if (!refresh) return
  try {
    await apiClient.post('/auth/logout/', { refresh })
  } catch {
    // Si el logout falla (token ya inválido), continuar igualmente —
    // lo importante es limpiar el estado local.
  }
}

/**
 * Obtiene el perfil del usuario autenticado actualmente.
 * GET /auth/me/
 * Úsalo para validar que la sesión sigue activa al cargar la app.
 */
export async function fetchCurrentUser(): Promise<LoggedUser> {
  try {
    const { data } = await apiClient.get<LoggedUser>('/auth/me/')
    return data
  } catch (err) {
    throw parseApiError(err)
  }
}
```

Cada función captura el error con `parseApiError` y lo relanza ya normalizado como `ApiError`.
El llamador (el store) solo necesita hacer `catch (err)` y sabe que `err` tiene `status` y `detail`.

---

## 3.3 AuthStore

**Archivo:** `src/domain/store/auth.store.ts`

El AuthStore es el cerebro de la autenticación: orquesta las llamadas a la API, persiste los
tokens, expone el estado a toda la app mediante Zustand y reacciona al evento `authExpired` que
el cliente Axios dispara cuando el refresh token falla.

```typescript
// src/domain/store/auth.store.ts
import { create } from 'zustand'
import { localStore } from '@/data/storage/local.storage'
import {
  loginUser,
  registerUser,
  logoutUser,
  fetchCurrentUser,
} from '@/data/api/auth.api'
import { AUTH_EXPIRED_EVENT } from '@/data/api/api.client'
import type { AuthState, LoggedUser, AuthTokens } from '@/domain/model/auth.types'

// ─── Tipos del store ──────────────────────────────────────────────────────────

interface AuthActions {
  /** Inicia sesión con username y password. */
  login(username: string, password: string): Promise<void>
  /** Registra un nuevo usuario y lo autentica automáticamente. */
  register(username: string, email: string, password: string): Promise<void>
  /** Cierra la sesión: invalida el token en el servidor y limpia el estado. */
  logout(): Promise<void>
  /**
   * Lee los tokens de localStorage y valida la sesión llamando a /auth/me/.
   * Debe llamarse al montar la app (useEffect en AppRouter).
   */
  loadSession(): Promise<void>
  /** Limpia el error actual. */
  clearError(): void
  /** Limpia el estado de sesión sin llamar al servidor (usado por authExpired). */
  _clearSession(): void
}

// ─── Store ────────────────────────────────────────────────────────────────────

export const useAuthStore = create<AuthState & AuthActions>((set, get) => {
  // Suscribirse al evento authExpired que dispara el interceptor de Axios
  // cuando el refresh token ya no es válido.
  // Lo registramos aquí para que esté activo desde el inicio de la app.
  if (typeof window !== 'undefined') {
    window.addEventListener(AUTH_EXPIRED_EVENT, () => {
      get()._clearSession()
    })
  }

  return {
    // ── Estado inicial ──────────────────────────────────────────────────────
    user: null,
    tokens: null,
    isLoading: false,
    error: null,

    // ── Acciones ────────────────────────────────────────────────────────────

    async login(username, password) {
      set({ isLoading: true, error: null })
      try {
        const { user, tokens } = await loginUser(username, password)
        localStore.setTokens(tokens.access, tokens.refresh)
        set({ user, tokens, isLoading: false })
      } catch (err: unknown) {
        const apiErr = err as { detail?: string; message?: string }
        set({
          isLoading: false,
          error: apiErr.detail ?? apiErr.message ?? 'Error al iniciar sesión',
        })
        throw err
      }
    },

    async register(username, email, password) {
      set({ isLoading: true, error: null })
      try {
        const { user, tokens } = await registerUser(username, email, password)
        localStore.setTokens(tokens.access, tokens.refresh)
        set({ user, tokens, isLoading: false })
      } catch (err: unknown) {
        const apiErr = err as { detail?: string; message?: string }
        set({
          isLoading: false,
          error: apiErr.detail ?? apiErr.message ?? 'Error al registrarse',
        })
        throw err
      }
    },

    async logout() {
      set({ isLoading: true })
      await logoutUser()
      localStore.clearTokens()
      set({ user: null, tokens: null, isLoading: false, error: null })
    },

    async loadSession() {
      const saved = localStore.getTokens()
      if (!saved) {
        // No hay tokens guardados: sesión limpia
        set({ user: null, tokens: null })
        return
      }

      set({ isLoading: true })
      try {
        // Validar que el access token sigue siendo válido
        const user = await fetchCurrentUser()
        set({
          user,
          tokens: { access: saved.access, refresh: saved.refresh },
          isLoading: false,
        })
      } catch {
        // Token expirado o inválido: limpiar sesión
        localStore.clearTokens()
        set({ user: null, tokens: null, isLoading: false })
      }
    },

    clearError() {
      set({ error: null })
    },

    _clearSession() {
      localStore.clearTokens()
      set({ user: null, tokens: null, isLoading: false, error: null })
    },
  }
})

// ─── Selectores de conveniencia ───────────────────────────────────────────────

/** true si hay un usuario autenticado. */
export const selectIsAuthenticated = (state: AuthState) => state.user !== null

/** true si el usuario autenticado tiene rol de staff/admin. */
export const selectIsStaff = (state: AuthState) =>
  state.user?.is_staff === true
```

El listener de `authExpired` se registra una sola vez cuando Zustand crea el store, que ocurre
antes del primer render. De este modo, cualquier petición que falle por sesión expirada —incluso
antes de que haya un componente montado— limpia el estado correctamente.

---

## 3.4 Login page

**Archivo:** `src/presentation/pages/auth/LoginPage.tsx`

Reemplaza el placeholder del módulo 1 con la implementación completa.

```tsx
// src/presentation/pages/auth/LoginPage.tsx
import { useEffect } from 'react'
import { useNavigate, useLocation, Link } from 'react-router-dom'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { Loader2, ShoppingBag } from 'lucide-react'

import { useAuthStore } from '@/domain/store/auth.store'
import { Button } from '@/presentation/components/ui/button'
import { Input } from '@/presentation/components/ui/input'
import { Label } from '@/presentation/components/ui/label'
import {
  Card,
  CardContent,
  CardDescription,
  CardFooter,
  CardHeader,
  CardTitle,
} from '@/presentation/components/ui/card'

// ─── Schema de validación ─────────────────────────────────────────────────────

const loginSchema = z.object({
  username: z
    .string()
    .min(3, 'El usuario debe tener al menos 3 caracteres'),
  password: z
    .string()
    .min(6, 'La contraseña debe tener al menos 6 caracteres'),
})

type LoginFormData = z.infer<typeof loginSchema>

// ─── Componente ───────────────────────────────────────────────────────────────

export default function LoginPage() {
  const navigate = useNavigate()
  const location = useLocation()

  // Destino al que redirigir tras login (si vinieron de una ruta protegida)
  const from = (location.state as { from?: Location })?.from?.pathname ?? '/'

  const { login, isLoading, error, clearError, user } = useAuthStore()

  // Si ya está autenticado, redirigir directamente
  useEffect(() => {
    if (user) navigate(from, { replace: true })
  }, [user, from, navigate])

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
  })

  async function onSubmit(data: LoginFormData) {
    clearError()
    try {
      await login(data.username, data.password)
      navigate(from, { replace: true })
    } catch {
      // El error ya se guardó en el store; no necesitamos hacer nada aquí
    }
  }

  return (
    <div className="flex min-h-screen items-center justify-center bg-muted/40 px-4">
      <Card className="w-full max-w-sm">
        <CardHeader className="text-center">
          {/* Logo / marca */}
          <div className="flex justify-center mb-2">
            <div className="flex items-center gap-2 rounded-full bg-primary p-3 text-primary-foreground">
              <ShoppingBag className="h-6 w-6" />
            </div>
          </div>
          <CardTitle className="text-2xl">ShopApp</CardTitle>
          <CardDescription>Inicia sesión para continuar</CardDescription>
        </CardHeader>

        <form onSubmit={handleSubmit(onSubmit)} noValidate>
          <CardContent className="space-y-4">
            {/* Error global de la API */}
            {error && (
              <div className="rounded-md bg-destructive/10 px-4 py-3 text-sm text-destructive">
                {error}
              </div>
            )}

            {/* Campo: username */}
            <div className="space-y-1">
              <Label htmlFor="username">Usuario</Label>
              <Input
                id="username"
                type="text"
                autoComplete="username"
                placeholder="tu_usuario"
                aria-invalid={!!errors.username}
                {...register('username')}
              />
              {errors.username && (
                <p className="text-xs text-destructive">{errors.username.message}</p>
              )}
            </div>

            {/* Campo: password */}
            <div className="space-y-1">
              <Label htmlFor="password">Contraseña</Label>
              <Input
                id="password"
                type="password"
                autoComplete="current-password"
                placeholder="••••••••"
                aria-invalid={!!errors.password}
                {...register('password')}
              />
              {errors.password && (
                <p className="text-xs text-destructive">{errors.password.message}</p>
              )}
            </div>
          </CardContent>

          <CardFooter className="flex flex-col gap-4">
            <Button type="submit" className="w-full" disabled={isLoading}>
              {isLoading ? (
                <>
                  <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                  Iniciando sesión…
                </>
              ) : (
                'Iniciar sesión'
              )}
            </Button>

            <p className="text-center text-sm text-muted-foreground">
              ¿No tienes cuenta?{' '}
              <Link
                to="/register"
                className="font-medium text-primary hover:underline"
              >
                Regístrate
              </Link>
            </p>
          </CardFooter>
        </form>
      </Card>
    </div>
  )
}
```

La página usa `location.state.from` para guardar la ruta de origen cuando React Router redirige
al login desde una ruta protegida, permitiendo volver a esa ruta al autenticarse correctamente.
El `useEffect` redirige directamente si el usuario ya tiene sesión activa, evitando que alguien
navegue al login estando ya autenticado.

---

## 3.5 Register page

**Archivo:** `src/presentation/pages/auth/RegisterPage.tsx`

```tsx
// src/presentation/pages/auth/RegisterPage.tsx
import { useEffect } from 'react'
import { useNavigate, Link } from 'react-router-dom'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { Loader2, ShoppingBag } from 'lucide-react'

import { useAuthStore } from '@/domain/store/auth.store'
import { Button } from '@/presentation/components/ui/button'
import { Input } from '@/presentation/components/ui/input'
import { Label } from '@/presentation/components/ui/label'
import {
  Card,
  CardContent,
  CardDescription,
  CardFooter,
  CardHeader,
  CardTitle,
} from '@/presentation/components/ui/card'

// ─── Schema de validación ─────────────────────────────────────────────────────

const registerSchema = z
  .object({
    username: z
      .string()
      .min(3, 'El usuario debe tener al menos 3 caracteres')
      .max(150, 'El usuario es demasiado largo')
      .regex(
        /^[\w.@+-]+$/,
        'Solo letras, números y los caracteres @ . + - _',
      ),
    email: z
      .string()
      .min(1, 'El email es obligatorio')
      .email('Introduce un email válido'),
    password: z
      .string()
      .min(6, 'La contraseña debe tener al menos 6 caracteres'),
    confirmPassword: z
      .string()
      .min(1, 'Confirma tu contraseña'),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: 'Las contraseñas no coinciden',
    path: ['confirmPassword'],
  })

type RegisterFormData = z.infer<typeof registerSchema>

// ─── Componente ───────────────────────────────────────────────────────────────

export default function RegisterPage() {
  const navigate = useNavigate()
  const { register: registerUser, isLoading, error, clearError, user } = useAuthStore()

  // Si ya está autenticado, ir al inicio
  useEffect(() => {
    if (user) navigate('/', { replace: true })
  }, [user, navigate])

  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<RegisterFormData>({
    resolver: zodResolver(registerSchema),
  })

  async function onSubmit(data: RegisterFormData) {
    clearError()
    try {
      await registerUser(data.username, data.email, data.password)
      navigate('/', { replace: true })
    } catch {
      // El error ya está en el store
    }
  }

  return (
    <div className="flex min-h-screen items-center justify-center bg-muted/40 px-4">
      <Card className="w-full max-w-sm">
        <CardHeader className="text-center">
          <div className="flex justify-center mb-2">
            <div className="flex items-center gap-2 rounded-full bg-primary p-3 text-primary-foreground">
              <ShoppingBag className="h-6 w-6" />
            </div>
          </div>
          <CardTitle className="text-2xl">Crear cuenta</CardTitle>
          <CardDescription>Únete a ShopApp hoy</CardDescription>
        </CardHeader>

        <form onSubmit={handleSubmit(onSubmit)} noValidate>
          <CardContent className="space-y-4">
            {/* Error global de la API */}
            {error && (
              <div className="rounded-md bg-destructive/10 px-4 py-3 text-sm text-destructive">
                {error}
              </div>
            )}

            {/* Campo: username */}
            <div className="space-y-1">
              <Label htmlFor="username">Usuario</Label>
              <Input
                id="username"
                type="text"
                autoComplete="username"
                placeholder="tu_usuario"
                aria-invalid={!!errors.username}
                {...register('username')}
              />
              {errors.username && (
                <p className="text-xs text-destructive">{errors.username.message}</p>
              )}
            </div>

            {/* Campo: email */}
            <div className="space-y-1">
              <Label htmlFor="email">Email</Label>
              <Input
                id="email"
                type="email"
                autoComplete="email"
                placeholder="tuemail@ejemplo.com"
                aria-invalid={!!errors.email}
                {...register('email')}
              />
              {errors.email && (
                <p className="text-xs text-destructive">{errors.email.message}</p>
              )}
            </div>

            {/* Campo: password */}
            <div className="space-y-1">
              <Label htmlFor="password">Contraseña</Label>
              <Input
                id="password"
                type="password"
                autoComplete="new-password"
                placeholder="••••••••"
                aria-invalid={!!errors.password}
                {...register('password')}
              />
              {errors.password && (
                <p className="text-xs text-destructive">{errors.password.message}</p>
              )}
            </div>

            {/* Campo: confirmPassword */}
            <div className="space-y-1">
              <Label htmlFor="confirmPassword">Confirmar contraseña</Label>
              <Input
                id="confirmPassword"
                type="password"
                autoComplete="new-password"
                placeholder="••••••••"
                aria-invalid={!!errors.confirmPassword}
                {...register('confirmPassword')}
              />
              {errors.confirmPassword && (
                <p className="text-xs text-destructive">
                  {errors.confirmPassword.message}
                </p>
              )}
            </div>
          </CardContent>

          <CardFooter className="flex flex-col gap-4">
            <Button type="submit" className="w-full" disabled={isLoading}>
              {isLoading ? (
                <>
                  <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                  Creando cuenta…
                </>
              ) : (
                'Crear cuenta'
              )}
            </Button>

            <p className="text-center text-sm text-muted-foreground">
              ¿Ya tienes cuenta?{' '}
              <Link
                to="/login"
                className="font-medium text-primary hover:underline"
              >
                Inicia sesión
              </Link>
            </p>
          </CardFooter>
        </form>
      </Card>
    </div>
  )
}
```

El esquema Zod usa `.refine()` al nivel del objeto (no de un campo individual) para comparar
`password` con `confirmPassword`. El `path: ['confirmPassword']` hace que React Hook Form sepa
a qué campo asignar el error de validación para mostrarlo debajo del campo correcto.

---

## 3.6 Protected route component

**Archivo:** `src/presentation/router/ProtectedRoute.tsx`

El componente `ProtectedRoute` actúa como guardia declarativa: cualquier ruta que necesite
autenticación o permisos de staff simplemente se envuelve con este componente en el router,
manteniendo la lógica de autorización centralizada y fuera de las páginas individuales.

```tsx
// src/presentation/router/ProtectedRoute.tsx
import { type ReactNode } from 'react'
import { Navigate, useLocation } from 'react-router-dom'
import { useAuthStore } from '@/domain/store/auth.store'

interface ProtectedRouteProps {
  /** Contenido a renderizar si la autorización es correcta. */
  children: ReactNode
  /**
   * Si es true, además de estar autenticado el usuario debe tener is_staff = true.
   * Los usuarios normales autenticados son redirigidos a /.
   */
  requireStaff?: boolean
}

/**
 * Ruta protegida por autenticación (y opcionalmente por rol de staff).
 *
 * Comportamiento:
 * - No autenticado → redirige a /login guardando la ruta actual en location.state.from
 * - Autenticado + requireStaff=true + user.is_staff=false → redirige a /
 * - Autenticado (y staff si se requiere) → renderiza children
 */
export default function ProtectedRoute({
  children,
  requireStaff = false,
}: ProtectedRouteProps) {
  const location = useLocation()
  const user = useAuthStore((state) => state.user)
  const isLoading = useAuthStore((state) => state.isLoading)

  // Mientras loadSession() está en curso, no redirigir aún
  // (evita un flash de redirect al recargar con sesión válida)
  if (isLoading) {
    return (
      <div className="flex min-h-screen items-center justify-center">
        <div className="h-8 w-8 animate-spin rounded-full border-4 border-primary border-t-transparent" />
      </div>
    )
  }

  // Sin autenticar: ir al login guardando la ruta de origen
  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />
  }

  // Autenticado pero sin permisos de staff
  if (requireStaff && !user.is_staff) {
    return <Navigate to="/" replace />
  }

  return <>{children}</>
}
```

El spinner durante `isLoading` es crítico para evitar un "flash of unauthenticated content":
cuando la app arranca y `loadSession()` está validando el token contra el servidor, el estado
inicial es `user: null`, lo que sin este guard haría un redirect inmediato a `/login` aunque
la sesión sea válida.

---

## 3.7 AppRouter actualizado

**Archivo:** `src/presentation/router/AppRouter.tsx`

Versión completa que integra `ProtectedRoute`, el `AppShell` y la carga inicial de sesión.

```tsx
// src/presentation/router/AppRouter.tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom'
import { Suspense, lazy, useEffect } from 'react'
import { useAuthStore } from '@/domain/store/auth.store'
import ProtectedRoute from './ProtectedRoute'
import AppShell from '@/presentation/components/AppShell'

// ─── Lazy imports ─────────────────────────────────────────────────────────────

// Auth (sin shell)
const LoginPage = lazy(() => import('../pages/auth/LoginPage'))
const RegisterPage = lazy(() => import('../pages/auth/RegisterPage'))

// Catalog
const CatalogPage = lazy(() => import('../pages/catalog/CatalogPage'))
const ProductDetailPage = lazy(() => import('../pages/catalog/ProductDetailPage'))

// Cart
const CartPage = lazy(() => import('../pages/cart/CartPage'))

// Orders
const OrdersPage = lazy(() => import('../pages/orders/OrdersPage'))
const OrderDetailPage = lazy(() => import('../pages/orders/OrderDetailPage'))

// Profile
const ProfilePage = lazy(() => import('../pages/profile/ProfilePage'))

// Admin
const AdminDashboardPage = lazy(() => import('../pages/admin/AdminDashboardPage'))
const AdminCategoriesPage = lazy(() => import('../pages/admin/AdminCategoriesPage'))
const AdminProductsPage = lazy(() => import('../pages/admin/AdminProductsPage'))
const AdminOrdersPage = lazy(() => import('../pages/admin/AdminOrdersPage'))
const AdminUsersPage = lazy(() => import('../pages/admin/AdminUsersPage'))

// ─── Loader global ────────────────────────────────────────────────────────────

function PageLoader() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="h-8 w-8 animate-spin rounded-full border-4 border-primary border-t-transparent" />
    </div>
  )
}

// ─── Router ───────────────────────────────────────────────────────────────────

export default function AppRouter() {
  const loadSession = useAuthStore((state) => state.loadSession)

  // Cargar la sesión guardada al iniciar la app.
  // loadSession() lee localStorage y valida el token con /auth/me/
  useEffect(() => {
    loadSession()
  }, [loadSession])

  return (
    <BrowserRouter>
      <Suspense fallback={<PageLoader />}>
        <Routes>
          {/* ── Rutas de autenticación (sin AppShell) ── */}
          <Route path="/login" element={<LoginPage />} />
          <Route path="/register" element={<RegisterPage />} />

          {/* ── Rutas con AppShell ── */}
          <Route element={<AppShell />}>
            {/* Públicas */}
            <Route path="/" element={<CatalogPage />} />
            <Route path="/catalog" element={<CatalogPage />} />
            <Route path="/products/:id" element={<ProductDetailPage />} />

            {/* Requieren autenticación */}
            <Route
              path="/cart"
              element={
                <ProtectedRoute>
                  <CartPage />
                </ProtectedRoute>
              }
            />
            <Route
              path="/orders"
              element={
                <ProtectedRoute>
                  <OrdersPage />
                </ProtectedRoute>
              }
            />
            <Route
              path="/orders/:id"
              element={
                <ProtectedRoute>
                  <OrderDetailPage />
                </ProtectedRoute>
              }
            />
            <Route
              path="/profile"
              element={
                <ProtectedRoute>
                  <ProfilePage />
                </ProtectedRoute>
              }
            />

            {/* Requieren autenticación + rol staff */}
            <Route
              path="/admin"
              element={
                <ProtectedRoute requireStaff>
                  <AdminDashboardPage />
                </ProtectedRoute>
              }
            />
            <Route
              path="/admin/categories"
              element={
                <ProtectedRoute requireStaff>
                  <AdminCategoriesPage />
                </ProtectedRoute>
              }
            />
            <Route
              path="/admin/products"
              element={
                <ProtectedRoute requireStaff>
                  <AdminProductsPage />
                </ProtectedRoute>
              }
            />
            <Route
              path="/admin/orders"
              element={
                <ProtectedRoute requireStaff>
                  <AdminOrdersPage />
                </ProtectedRoute>
              }
            />
            <Route
              path="/admin/users"
              element={
                <ProtectedRoute requireStaff>
                  <AdminUsersPage />
                </ProtectedRoute>
              }
            />
          </Route>

          {/* Fallback */}
          <Route path="*" element={<Navigate to="/" replace />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  )
}
```

Usar `<Route element={<AppShell />}>` como layout route de React Router v6 permite que todas las
rutas hijas compartan el mismo header sin tener que importar y usar `AppShell` en cada página.
`AppShell` debe renderizar `<Outlet />` para que React Router inyecte el contenido de la ruta hija.

---

## 3.8 App shell con header

**Archivo:** `src/presentation/components/AppShell.tsx`

El AppShell es el layout persistente de la aplicación: contiene el header con navegación, el
menú de usuario y el área de contenido principal. Se usa como layout route en React Router v6.

```tsx
// src/presentation/components/AppShell.tsx
import { Outlet, Link, useNavigate, NavLink } from 'react-router-dom'
import { ShoppingBag, ShoppingCart, Package, User, LogOut, LayoutDashboard } from 'lucide-react'
import { useAuthStore } from '@/domain/store/auth.store'
import { Button } from '@/presentation/components/ui/button'
import { Badge } from '@/presentation/components/ui/badge'
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuLabel,
  DropdownMenuSeparator,
  DropdownMenuTrigger,
} from '@/presentation/components/ui/dropdown-menu'
import { Avatar, AvatarFallback } from '@/presentation/components/ui/avatar'
import { Separator } from '@/presentation/components/ui/separator'

// ─── Helpers ──────────────────────────────────────────────────────────────────

/** Obtiene las iniciales del username para el avatar. */
function getInitials(username: string): string {
  return username.slice(0, 2).toUpperCase()
}

/** Clases para los enlaces de navegación activos/inactivos. */
function navLinkClass({ isActive }: { isActive: boolean }) {
  return [
    'text-sm font-medium transition-colors hover:text-primary',
    isActive ? 'text-primary' : 'text-muted-foreground',
  ].join(' ')
}

// ─── Componente ───────────────────────────────────────────────────────────────

export default function AppShell() {
  const navigate = useNavigate()
  const { user, logout } = useAuthStore()

  // En módulos siguientes esto vendrá del CartStore
  const cartItemCount = 0

  async function handleLogout() {
    await logout()
    navigate('/login', { replace: true })
  }

  return (
    <div className="flex min-h-screen flex-col">
      {/* ── Header ─────────────────────────────────────────────────────────── */}
      <header className="sticky top-0 z-50 w-full border-b bg-background/95 backdrop-blur supports-[backdrop-filter]:bg-background/60">
        <div className="mx-auto flex h-16 max-w-7xl items-center gap-6 px-4">
          {/* Logo / marca */}
          <Link
            to="/"
            className="flex items-center gap-2 font-bold text-primary"
          >
            <ShoppingBag className="h-5 w-5" />
            <span>ShopApp</span>
          </Link>

          <Separator orientation="vertical" className="h-6" />

          {/* Navegación principal */}
          <nav className="flex items-center gap-4">
            <NavLink to="/catalog" className={navLinkClass}>
              Catálogo
            </NavLink>
            {user && (
              <>
                <NavLink to="/orders" className={navLinkClass}>
                  Pedidos
                </NavLink>
                <NavLink to="/profile" className={navLinkClass}>
                  Perfil
                </NavLink>
              </>
            )}
            {user?.is_staff && (
              <NavLink to="/admin" className={navLinkClass}>
                Admin
              </NavLink>
            )}
          </nav>

          {/* Espacio flexible */}
          <div className="flex-1" />

          {/* Acciones del lado derecho */}
          <div className="flex items-center gap-2">
            {/* Carrito */}
            {user && (
              <Button
                variant="ghost"
                size="icon"
                asChild
                className="relative"
                aria-label="Carrito de compras"
              >
                <Link to="/cart">
                  <ShoppingCart className="h-5 w-5" />
                  {cartItemCount > 0 && (
                    <Badge
                      variant="destructive"
                      className="absolute -right-1 -top-1 flex h-5 w-5 items-center justify-center rounded-full p-0 text-xs"
                    >
                      {cartItemCount > 99 ? '99+' : cartItemCount}
                    </Badge>
                  )}
                </Link>
              </Button>
            )}

            {/* Usuario autenticado → menú desplegable */}
            {user ? (
              <DropdownMenu>
                <DropdownMenuTrigger asChild>
                  <Button
                    variant="ghost"
                    className="relative h-9 w-9 rounded-full"
                    aria-label="Menú de usuario"
                  >
                    <Avatar className="h-9 w-9">
                      <AvatarFallback className="bg-primary text-primary-foreground text-sm">
                        {getInitials(user.username)}
                      </AvatarFallback>
                    </Avatar>
                  </Button>
                </DropdownMenuTrigger>

                <DropdownMenuContent align="end" className="w-48">
                  <DropdownMenuLabel className="font-normal">
                    <div className="flex flex-col gap-1">
                      <p className="text-sm font-medium">{user.username}</p>
                      <p className="text-xs text-muted-foreground truncate">
                        {user.email}
                      </p>
                    </div>
                  </DropdownMenuLabel>

                  <DropdownMenuSeparator />

                  <DropdownMenuItem asChild>
                    <Link to="/profile" className="flex items-center gap-2 cursor-pointer">
                      <User className="h-4 w-4" />
                      Mi perfil
                    </Link>
                  </DropdownMenuItem>

                  <DropdownMenuItem asChild>
                    <Link to="/orders" className="flex items-center gap-2 cursor-pointer">
                      <Package className="h-4 w-4" />
                      Mis pedidos
                    </Link>
                  </DropdownMenuItem>

                  {user.is_staff && (
                    <>
                      <DropdownMenuSeparator />
                      <DropdownMenuItem asChild>
                        <Link to="/admin" className="flex items-center gap-2 cursor-pointer">
                          <LayoutDashboard className="h-4 w-4" />
                          Panel Admin
                        </Link>
                      </DropdownMenuItem>
                    </>
                  )}

                  <DropdownMenuSeparator />

                  <DropdownMenuItem
                    onClick={handleLogout}
                    className="flex items-center gap-2 cursor-pointer text-destructive focus:text-destructive"
                  >
                    <LogOut className="h-4 w-4" />
                    Cerrar sesión
                  </DropdownMenuItem>
                </DropdownMenuContent>
              </DropdownMenu>
            ) : (
              /* Usuario no autenticado → botón de login */
              <Button asChild size="sm">
                <Link to="/login">Iniciar sesión</Link>
              </Button>
            )}
          </div>
        </div>
      </header>

      {/* ── Contenido principal ─────────────────────────────────────────────── */}
      <main className="flex-1">
        <div className="mx-auto max-w-7xl px-4 py-6">
          {/* React Router inyecta aquí el componente de la ruta activa */}
          <Outlet />
        </div>
      </main>

      {/* ── Footer mínimo ───────────────────────────────────────────────────── */}
      <footer className="border-t py-4 text-center text-sm text-muted-foreground">
        ShopApp &copy; {new Date().getFullYear()}
      </footer>
    </div>
  )
}
```

El header usa `position: sticky` con `backdrop-blur` para que permanezca visible mientras el
usuario hace scroll sin ocultar el contenido. `NavLink` de React Router añade automáticamente
la clase `active` que la función `navLinkClass` aprovecha para cambiar el color del enlace activo.

---

## 3.9 Checkpoint y verificación

Con todos los archivos del módulo en su lugar, verifica que todo funciona correctamente:

### Verificar TypeScript

```bash
npx tsc --noEmit
```

No debe haber errores de tipos.

### Flujo de registro

1. Navega a `http://localhost:5173/register`
2. Completa el formulario con un usuario nuevo
3. Deberías ser redirigido a `/` (catálogo)
4. El header debe mostrar tu avatar con las iniciales del usuario

### Flujo de login

1. Cierra sesión desde el menú desplegable del header
2. Navega a `http://localhost:5173/login`
3. Ingresa las credenciales del usuario registrado
4. Deberías ser redirigido a `/`

### Persistencia de sesión

1. Recarga la página con `F5` o `Cmd+R`
2. El usuario debe seguir autenticado (sin volver al login)
3. Verifica en DevTools → Application → Local Storage que existen las claves `shopapp_access` y `shopapp_refresh`

### Rutas protegidas

1. Cierra sesión
2. Navega directamente a `http://localhost:5173/cart`
3. Debes ser redirigido a `/login`
4. Inicia sesión y debes ser redirigido de vuelta a `/cart`

### Rutas de admin

1. Inicia sesión con un usuario normal (no staff)
2. Navega a `http://localhost:5173/admin`
3. Debes ser redirigido a `/`
4. Inicia sesión con un usuario staff — el enlace "Admin" debe aparecer en el header

---

## Resumen

| Archivo | Responsabilidad |
|---|---|
| `src/domain/model/auth.types.ts` | Interfaces `LoggedUser`, `AuthTokens` y `AuthState` — el contrato del dominio de autenticación |
| `src/data/api/auth.api.ts` | Funciones `loginUser`, `registerUser`, `refreshToken`, `logoutUser` y `fetchCurrentUser` contra la API Django |
| `src/domain/store/auth.store.ts` | Store Zustand con `login`, `register`, `logout`, `loadSession`; escucha el evento `authExpired` |
| `src/presentation/pages/auth/LoginPage.tsx` | Formulario de login con React Hook Form + Zod, Card de shadcn/ui, redirect inteligente |
| `src/presentation/pages/auth/RegisterPage.tsx` | Formulario de registro con validación de confirmación de contraseña mediante `.refine()` |
| `src/presentation/router/ProtectedRoute.tsx` | Guardia declarativa de rutas: redirige a `/login` si no autenticado, a `/` si no tiene permisos de staff |
| `src/presentation/router/AppRouter.tsx` | Router actualizado con layout routes, `ProtectedRoute` y `loadSession()` en el montaje |
| `src/presentation/components/AppShell.tsx` | Header sticky con logo, navegación, badge del carrito, avatar y menú desplegable del usuario |
